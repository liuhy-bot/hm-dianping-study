# 03 用 Redis 分布式锁解决集群一人多单

接 02 的结论：锁的边界必须从 JVM 内部搬到 JVM 之外，所有实例都能访问的地方。

本篇记录三次迭代：`synchronized` 失效 → 简单 Redis 锁 → Lua 补上原子性。

## 一、问题起点：synchronized 在集群下失效

```java
synchronized (userId.toString().intern()) {
    // 查订单数 → 判断 → 扣库存 → 插订单
}
```

`intern()` 让相同 userId 字符串指向同一个常量池对象，单机下这把锁是有效的。

但两个实例 = 两个 JVM = 两个互相看不见的堆。8081 的锁对象和 8082 的锁对象是两个无关的内存地址，互不阻塞。

实测（20 并发，nginx 前置 8080）：**2 单成功**，订单 `660017915117961221`、`660017915117961222`，库存 100 → 98。

刚好 2 条，恰好反证了两件事：nginx 轮询把请求对半分给两台，每台内部的锁只放行 1 个。

## 二、第二步：用 Redis 实现分布式锁

### 加锁

一条命令同时完成「占位 + 设过期」：

```java
Boolean success = stringRedisTemplate.opsForValue()
        .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
```

底层是 `SET lock:order:5 <threadId> NX EX 1200`。

- `NX`：key 不存在才写入，实现互斥
- `EX`：写入时带过期时间，持有者崩溃后锁自动释放，避免死锁
- 原子：`SETNX` 和 `EXPIRE` 在同一条命令里，不存在「抢到锁但没设过期」的窗口

### 持有者标识

```java
private static final String ID_PREFIX = UUID.randomUUID().toString();
String threadId = ID_PREFIX + Thread.currentThread().getId();
```

`static` 保证每个 JVM 一个 UUID，后缀区分线程。

为什么两个都要：两个 JVM 上线程 ID 完全可能相同（都是 1、2、3……），只靠线程 ID 无法区分；加上 JVM 级 UUID 前缀才唯一。

### 验收

清库后 20 并发：

| | synchronized 版 | Redis 锁版 |
| --- | --- | --- |
| 成功数 | 2 | **1** |
| 重复订单 | 2 | **1**（正常） |
| 库存变化 | 100 → 98 | 100 → **99** |

```
订单 id: 660055620635852808  (user_id=5, voucher_id=10)
dup_count = 1
stock = 99
```

19 个请求返回「不允许重复下单」，一人一单在集群下守住了。

### 谁先抢到锁是不确定的

20 个线程用 `Barrier` 同时放出，只是「同时开始发」，不是「同时到达」。中间还隔着：

- 操作系统线程调度顺序
- 各自 TCP 连接建立耗时
- nginx 分给 8081 还是 8082
- Tomcat 线程池接单顺序
- Redis 单线程执行的先后

任何一环都能翻转结果。所以秒杀里不需要关心「谁先」，只需要保证「只有一个」。

两种错误信息也对应不同阶段：

- **不允许重复下单** = 没抢到锁
- **不能重复购买** = 抢到了锁，但数据库里已有订单

## 三、遗留问题：`unlock` 不是原子的

```java
String id = redis.get(key);      // ① 一次网络往返
if (threadId.equals(id)) {       // ② 本地内存判断
    redis.delete(key);           // ③ 又一次网络往返
}
```

问题不在于「分两步」，而在于 **「判断是不是自己的锁」和「删除锁」不在同一个不可分割的执行单元里**。

①③ 之间是两次独立的网络往返，中间 Redis 一直在处理其他客户端的命令。按 0.1~1ms 的往返估算，一个每秒能处理十万条以上命令的实例，在这个窗口里能跑上百条命令——包括锁到期被自动删除、另一个线程 `SET NX` 成功抢锁。

此时 ③ 删掉的就是**别人的锁**。

⚠️ 锁超时只是触发条件，原子性缺失才是根本原因。

### 为什么 MULTI/EXEC 解决不了

Redis 事务保证「这批命令连续执行、中间不插入其他客户端」，但**不支持条件执行**——写不出「如果值等于 X 才 DELETE」。

`WATCH` 能做条件判断（乐观锁），但冲突后要整体重试，竞争激烈时表现很差，`DEL` 本身也依然是无条件的。

### Lua 才是正解

```lua
if(redis.call('get', KEYS[1]) == ARGV[1]) then
    return redis.call('del', KEYS[1])
end
return 0
```

```java
private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;

static {
    UNLOCK_SCRIPT = new DefaultRedisScript<>();
    UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
    UNLOCK_SCRIPT.setResultType(Long.class);
}

@Override
public void unlock() {
    stringRedisTemplate.execute(
            UNLOCK_SCRIPT,
            Collections.singletonList(KEY_PREFIX + name),   // KEYS[1]
            ID_PREFIX + Thread.currentThread().getId()      // ARGV[1]
    );
}
```

三个决定性区别：

| | Java 两步 | Lua |
| --- | --- | --- |
| 网络往返 | 2 次 | 1 次 |
| 中间能否插入其他命令 | 能 | 不能 |
| 条件判断在哪执行 | 客户端内存 | Redis 内部 |

关键在于第三行：判断发生在 Redis 内部，和删除共享同一段不可打断的执行。客户端不再依赖自己看到的旧快照。

返回值含义：`1` = 删除成功（锁是自己的），`0` = 锁不存在或不属于自己。

### `execute` 的三个参数

```java
<T> T execute(RedisScript<T> script, List<K> keys, Object... args)
```

对应原生命令：

```
EVAL script numkeys key [key ...] arg [arg ...]
```

keys 和 args 必须分开传：集群模式下 Redis 要先扫 `numkeys` 确定槽位、路由到正确节点，并校验同一脚本内的 key 是否同 slot。混在 args 里就没法路由。

## 四、三次迭代对照

| 阶段 | 实现 | 互斥范围 | 并发结果 |
| --- | --- | --- | --- |
| 1 | `synchronized (userId.intern())` | 单个 JVM | 2 单 |
| 2 | `SET NX EX` + `get/delete` | 所有实例 | 1 单（但删锁不安全） |
| 3 | `SET NX EX` + Lua | 所有实例 | 1 单（删锁安全） |

## 五、结论

> 互斥失效的根因是：「这锁是不是我的」这个判断，和「删掉它」这个动作，不在同一个不可分割的执行单元里。

超时和 Lua 解决的是两件事，缺一不可：

- **`EX` 超时** → 防死锁。持有者宕机，锁能自动释放
- **Lua 原子释放** → 防误删。业务超时被别人抢走后，不会删错

## 六、待办：Redisson 与看门狗

手写的 `SimpleRedisLock` 是为了理解原理。它还有两个问题：

1. **锁的超时时间该设多大？** 设小了业务没跑完就过期，设大了崩溃后恢复慢
2. **不可重入**：同一线程再次尝试加锁会被自己挡住

Redisson 的看门狗解决了第一个：默认 30 秒，业务没结束就每 10 秒自动续期一次。

## 测试数据清理

```sql
DELETE FROM tb_voucher_order WHERE user_id = 5 AND voucher_id = 10;
UPDATE tb_seckill_voucher SET stock = 100 WHERE voucher_id = 10;
```

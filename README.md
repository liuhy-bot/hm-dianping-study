# hm-dianping-study

黑马点评（Redis 实战课程）学习项目与笔记。

## 关于

本项目跟随黑马程序员《Redis 实战》课程实现，是一个点评类应用，包含商户缓存、优惠券秒杀、分布式锁等模块。
用于个人学习与知识记录，非原创作品。

## 学习进度

| 阶段 | 内容 | 状态 |
| --- | --- | --- |
| 01 | nginx 反向代理 + 双实例集群搭建 | 已完成 |
| 02 | 集群模式下「一人一单」并发问题复现 | 已完成 |
| 03 | Redis 分布式锁（SETNX + Lua 脚本） | 进行中 |
| 04 | 秒杀异步优化与消息队列 | 待开始 |

## 目录结构

```
hm-dianping/
├── src/            项目源码
├── notes/          学习笔记（按章节编号）
├── pom.xml         Maven 构建配置
└── README.md
```

## 运行环境

| 组件 | 版本 / 配置 |
| --- | --- |
| JDK | 1.8 |
| MySQL | 8.0，库名 `hmdp` |
| Redis | 6.x，独立部署 |
| nginx | 1.18 |
| 后端实例 | 8081、8082 两个实例 |
| 入口 | nginx 监听 8080，`/api` 前缀反向代理到 `upstream backend` |

启动顺序：MySQL、Redis → 两个 Spring Boot 实例（8081/8082）→ nginx（8080）。

## 配置说明

仓库里**不包含**真实的 `application.yaml`（该文件含数据库与 Redis 密码，已被 `.gitignore` 排除）。

克隆后自行创建：

```bash
cp src/main/resources/application.yaml.example src/main/resources/application.yaml
```

然后填入本机实际的密码，或通过环境变量注入：

| 环境变量 | 含义 |
| --- | --- |
| `MYSQL_PASSWORD` | MySQL 密码 |
| `REDIS_HOST` | Redis 地址 |
| `REDIS_PASSWORD` | Redis 密码 |

## 笔记索引

见 [notes/README.md](notes/README.md)

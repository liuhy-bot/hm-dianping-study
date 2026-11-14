# hm-dianping-study

黑马点评（Redis 实战课程）学习项目与笔记。

## 关于

本项目跟随黑马程序员《Redis 实战》课程实现，是一个点评类应用，包含短信登录、商户缓存、优惠券秒杀、分布式锁等模块。
用于个人学习与知识记录，非原创作品。

## 学习进度

按课程顺序排列，与 `notes/` 目录下的笔记编号相互独立。

| 阶段 | 内容 | 状态 |
| --- | --- | --- |
| 01 | 项目搭建与 nginx 反向代理、双实例集群 | 已完成 |
| 02 | 短信验证码登录：session 共享问题、Redis 实现共享 session、token 刷新 | 已完成 |
| 03 | 商户查询缓存：缓存更新策略、缓存穿透 / 雪崩 / 击穿、缓存工具封装 | 已完成 |
| 04 | 优惠券秒杀：一人一单、集群下的并发超卖问题 | 已完成 |
| 05 | Redis 分布式锁：SETNX 互斥、超时释放、Lua 原子释放 | 已完成 |
| 06 | 秒杀异步优化与消息队列 | 待开始 |

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

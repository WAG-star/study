# Redis 学习笔记

> 归入数据库/ 目录 · 数据结构 + 命令大全 + 综合评估
> 学习方式：概念讲解 → 5 题/组 → 逐题批改（10 分制）

## 1. Redis 定位

- **定义**：以内存为主要存储介质、提供丰富数据结构的键值数据库
- **光谱位置**：MySQL（B+Tree 磁盘）← RocksDB（LSM-Tree）← Redis（全内存+可选持久化）
- 💡 **类比**：MySQL 像图书馆（书在书架/磁盘），Redis 像摊在桌上的便签（伸手即写）

## 2. 为什么快（架构）

- **全内存操作**：无磁盘寻道，一次读写几十纳秒
- **单线程无锁**：不存在竞争，不需要 mutex
- **epoll 多路复用**：一个线程监听成千上万 socket，有数据才处理
- 注：Redis 6.0+ 引入多线程 I/O（网络读写），命令执行仍单线程

## 3. 核心概念

| 概念 | 要点 |
|------|------|
| SCAN 游标 | 不是页码/索引/offset，是哈希表**遍历到的位置编码**；回到 0 = 扫完一圈结束 |
| RDB vs AOF | 快照 vs 日志（存储层设计取舍） |
| TTL 语义 | `-1` 永不过期 / `-2` 不存在 / `>0` 剩余秒数 |
| HINCRBY 返回值 | 返回**操作后的新值**，不是增量 |
| ZRANK 索引 | 从 0 开始（最低分 = 0） |

## 4. 命令速查（已掌握 53 条核心）

### 通用键操作
`KEYS`（生产禁用）、`DEL`（返回实际删除数）、`EXISTS`（返回存在数量）、`EXPIRE`（1=成功 0=不存在）、`TTL`、`TYPE`、`RENAME`、`UNLINK`（异步删除，大 key 专用）

### SCAN 系列
`SCAN cursor [MATCH pat] [COUNT n]` / `SSCAN` / `HSCAN` / `ZSCAN` — 游标从 0 开始，回 0 结束；COUNT 是建议值，MATCH 是后过滤

### String
`SET key val [EX n] [NX|XX]`、`GET`、`GETDEL`（取值+删除）、`INCR`（**只接受 1 个参数**）、`INCRBY`、`MSET/MGET`（不存在的 key 占位 nil 不跳过）

### Hash
`HSET/HSET`（字段无独立 TTL）、`HGET`（不存在返回 `(nil)`）、`HGETALL`（**field-val 交替返回**）、`HINCRBY`（整数）、`HEXISTS`、`HDEL`、`HMGET`

### List
`LPUSH/RPUSH`、`LPOP/RPOP`、`LRANGE key 0 -1`、`LTRIM key start stop`（**保留范围其余全删**，不可逆）、`LLEN`、`BLPOP key timeout`（超时返回 `(nil)`）

### Set
`SADD`、`SCARD`、`SINTER`（交集）、`SUNION`、`SDIFF`

### Sorted Set
`ZADD`、`ZRANK`（从 0 起）、`ZREVRANK`、`ZRANGE [WITHSCORES]`、`ZSCORE`

## 5. 数据结构选型速查（场景题）

| 场景 | 最佳结构 |
|------|:--:|
| 共同好友列表 | Set（SINTER） |
| API 调用计数 | **String（INCR）** |
| 最新 50 条行为日志 | List（LPUSH + LTRIM 固定容量） |
| 实时排行榜前 100 | **Sorted Set** |
| 会话 token 10 分钟过期 | String（SET EX） |
| 用户资料（昵称/头像/邮箱） | Hash |

## 6. 经典实战模式

1. **缓存**：String + EX 过期
2. **分布式锁**：`SET lock key NX EX 10`
3. **排行榜**：Sorted Set + ZREVRANGE
4. **固定容量队列**：`LPUSH timeline "新动态"` + `LTRIM timeline 0 99`
5. **短信验证码**：`SET sms:138... 8842 EX 300`（验证码是单个值，用 String 不用 Hash）

## 7. 易错点清单（综合评估 63/100 错题记录）

- [x] `INCR` 只接受 1 个参数，`INCR a 5` 直接报错
- [x] `INCR page` 对非整数值（"views"）报错
- [x] `HGET` 不存在的字段返回 `(nil)` 不是 0
- [x] `HGETALL` 返回 field-val 交替的**多个元素**
- [x] `HINCRBY` 返回新值（100→120 返回 120）
- [x] `LTRIM` 是原地裁切不可逆
- [x] `ZRANGE 0 0` 只返回第一个元素
- [x] ZSet 同分时按字典序排列

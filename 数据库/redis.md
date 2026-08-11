# Redis 学习笔记

> 归入数据库/ 目录 · 定位 + 架构 + 底层编码 + 持久化 + 53 命令 + 错题
> 学习方式：概念讲解 → 5 题/组 → 逐题批改（10 分制）→ 综合评估 63/100

## 1. Redis 定位

- **定义**：以内存为主要存储介质、提供丰富数据结构的键值数据库
- **光谱位置**：MySQL（B+Tree 磁盘）← RocksDB（LSM-Tree）← Redis（全内存+可选持久化）
- 💡 **类比**：MySQL 像图书馆（书在书架/磁盘），Redis 像摊在桌上的便签（伸手即写）

## 2. 为什么快（架构）

- **全内存操作**：无磁盘寻道，一次读写几十纳秒
- **单线程无锁**：不存在竞争，不需要 mutex（命令执行仍单线程，6.0+ 网络 I/O 多线程）
- **epoll 多路复用**：一个线程监听成千上万 socket，有数据才处理
- 💡 **类比**：咖啡师一杯一杯做（动作极快），切换代价比并发更大

## 3. 底层数据结构（数据库学习路线衔接点）

| 对外类型 | 底层编码 | 切换条件 |
|---------|---------|---------|
| String | `int` → `embstr` → `raw` | 整数 → ≤44字节 → 长串 |
| Hash | `ziplist` → `hashtable` | 字段少用压缩列表，多了转字典 |
| List | `quicklist` | 双向链表 + 压缩列表混合体 |
| Set | `intset` → `hashtable` | 全整数且少 |
| Sorted Set | `ziplist` → `skiplist + dict` | 少元素 → 跳表+字典 |

### SDS（Simple Dynamic String）
为什么不用 C 字符串：`strlen` O(n)、拼接易溢出、`\0` 截断二进制。
SDS = `len`(O(1)取长度) + `alloc`(容量，防溢出) + `buf`(二进制安全)。

### 跳表（skiplist）
- 概率平衡，比红黑树实现简单；O(log n) 查找
- Redis ZSet、RocksDB/LevelDB MemTable 都用
- **对你学习路线的价值**：学跳表 = 后面看 LSM-Tree 源码直接对上

## 4. 持久化（存储层设计取舍）

| | RDB 快照 | AOF 追加日志 |
|---|---------|-------------|
| 原理 | fork 子进程 COW 写 dump.rdb | 每条写命令追加 aof 文件 |
| 优 | 恢复快、文件紧凑 | 丢数据少（everysec 最多丢 1 秒） |
| 劣 | 两次快照之间数据丢 | 文件大、恢复慢（重放命令） |
| 学习价值 | COW fork = OS×DB 交叉点 | 追加写 = LSM WAL 同思路 |

混合持久化（4.0+）：RDB 快照 + 增量 AOF 一个文件。

## 5. 命令速查（已掌握 53 条）

### 通用键操作
| 命令 | 说明 |
|------|------|
| `KEYS pattern` | 按模式匹配，**生产禁用**（阻塞） |
| `DEL key...` | 删除，返回实际删除数 |
| `EXISTS key...` | 判断存在，返回存在数量 |
| `EXPIRE key s` | 设过期（秒），1=成功 0=不存在 |
| `TTL key` | >0 剩余 / -1 永不过期 / -2 不存在 |
| `TYPE key` | 返回数据类型 |
| `RENAME old new` | 重命名 |
| `UNLINK key` | 异步删除（非阻塞），大 key 专用 |

### SCAN 系列（4 条）
`SCAN cursor [MATCH pat] [COUNT n]` / `SSCAN` / `HSCAN` / `ZSCAN` — 游标从 0 开始，回 0 结束；COUNT 是建议值，MATCH 是后过滤。

**游标本质**：不是页码/索引/offset，是哈希表**遍历到的位置编码**。类比：页码翻书 vs "第3章第2节第5段"的结构位置。两次 SCAN 17 可能返回不同 key（中间数据变了）。

### String
`SET key val [EX s] [NX|XX] [GET]`（NX=不存在才设=锁，XX=存在才设）、`GET`、`INCR`（**只接受 1 个参数**）、`DECR`、`INCRBY key N`、`DECRBY`、`MSET/MGET`（不存在占位 nil 不跳过）、`GETDEL`（取值+删除）、`SETNX/SETEX`（已过时）

### Hash
`HSET`（支持多字段，**不支持 EX/NX/XX**）、`HGET`（不存在返回 `(nil)`）、`HMGET`、`HGETALL`（**field-val 交替，元素数=字段数×2**）、`HDEL`、`HEXISTS`、`HLEN`、`HINCRBY`（**只能整数，返回操作后的新值**）、`HKEYS/HVALS`

### List
`LPUSH/RPUSH`（最后推的在最左/顺序）、`LPOP/RPOP [count]`、`LRANGE key start stop`（负数从尾倒数）、`LLEN`、`LTRIM key start stop`（**保留范围其余全删，不可逆**）、`BLPOP/BRPOP key timeout`（**返回 `[key名, 值]`**，0=永久等）

### Set
`SADD`（自动去重，返回新增数）、`SREM`、`SMEMBERS`（大集合慎用）、`SISMEMBER`、`SCARD`、`SINTER`（交集）、`SUNION`（并集）、`SDIFF`（第一个有后面都没有）、`SPOP [count]`（随机弹删）、`SRANDMEMBER`（随机抽不删）、`SMOVE src dst m`（原子迁移）

### Sorted Set
**规则：分数在前，成员在后。**
`ZADD key score member`（已存在=更新分数）、`ZREM`、`ZRANGE`（升序）、`ZREVRANGE`（降序）、`ZRANK`（**排名索引 0-based，不是分数**）、`ZREVRANK`（0=最高分）、`ZSCORE`、`ZINCRBY`（返回新值）、`ZRANGEBYSCORE key min max`（`(90`=开区间）、`ZCOUNT`、`ZCARD`

## 6. 数据结构选型速查

| 场景 | 最佳结构 |
|------|:--:|
| 计数器/自增ID/限流 | **String（INCR）** |
| 用户信息/配置项 | Hash |
| 消息队列/最新动态 | List（LRANGE+LTRIM） |
| 排行榜/TopN | **Sorted Set（ZREVRANGE）** |
| 标签/共同好友/去重 | Set（SINTER/SADD） |
| 临时 token+过期 | String（SET EX） |
| 大 key 安全遍历 | SCAN |
| 短信验证码 | String（SET EX，**不是 Hash**——字段无独立 TTL） |

## 7. 实战模式

1. **缓存**：`SET key val EX 3600`
2. **分布式锁**：`SET lock uuid NX EX 30`
3. **排行榜**：ZSet + `ZREVRANGE 0 9`
4. **固定容量队列**：`LPUSH timeline "新"` + `LTRIM timeline 0 99`
5. **消息队列**：生产者 `RPUSH` + 消费者 `BLPOP key 0`
6. **投票防重**：`SADD votes:article user` + `SISMEMBER` 判重

## 8. 易错点清单（63/100 错题记录）

- [x] `INCR` 只接受 1 个参数，`INCR a 5` 直接报错（要用 INCRBY）
- [x] `INCR` 对非整数值（如 "views"）报错，值不变
- [x] `HGET` 不存在的字段返回 `(nil)` 不是 0
- [x] `HGETALL` 返回 field-val 交替的**多个元素**（字段数×2）
- [x] `HINCRBY` 返回新值（100+20=120 返回 120），不是增量
- [x] `LTRIM` 是**保留**范围其余删，不是取出；不可逆
- [x] `BLPOP` 超时返回 `(nil)`；成功返回 `[key名, 值]`
- [x] `ZRANK` 返回排名索引（0-based），不是分数
- [x] `ZRANGE 0 0` 只返回第一个元素（闭区间）
- [x] ZSet 同分按成员**字典序**排列
- [x] 场景选型：API 计数→String 不是 List；日志→List 不是 Hash；排行榜→ZSet 不是 List
- [x] `SET` 支持 EX/NX/XX/GET 后缀；`HSET` 一律不支持

## 9. 核心公式速记

```
命名规律:   {类型首字母}{操作动词}   H=Hash L/R=List S=Set Z=ZSet
过期三件套: SET EX → EXPIRE → TTL(>0/-1/-2)
计数器:     INCR/DECR/INCRBY（不存在从0开始，只操作整数）
List 方向:  RPUSH+LPOP=队列(FIFO)  LPUSH+LPOP=栈(LIFO)  BLPOP+RPUSH=阻塞消费
Set 集合:   SINTER=∩  SUNION=∪  SDIFF=-
ZSet 规则:  分数在前成员后 | ZRANGE升序 ZREVRANGE降序 | 排名0起 | 同分字典序
写命令:     key 不存在自动创建；数值命令不存在从 0 开始，非整数报错
```

# 04 Redis（redis-py）

> 内存级键值数据库。原理侧（底层编码 / 持久化 / 53 命令）见 [`../数据库/redis.md`](../数据库/redis.md)。

## 1. 安装与连接

```bash
pip install redis
```

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, db=0, decode_responses=True)
# decode_responses=True：返回 str 而非 bytes（Python3 必开，省去解码）
```

## 2. 字符串（最常用）

```python
r.set("key", "value")                  # 写
r.get("key")                           # 读
r.set("key", "value", ex=60)           # 60 秒过期
r.setnx("key", "value")                # 不存在才写（分布式锁基础）
r.incr("counter")                      # 自增（计数器）
r.mset({"a": 1, "b": 2})               # 批量写
```

## 3. 哈希（≈ 对象）

```python
r.hset("user:1", mapping={"name": "Alice", "age": 25})
r.hget("user:1", "name")               # 单字段
r.hgetall("user:1")                    # 全部字段
r.hincrby("user:1", "age", 1)          # 字段自增
```

## 4. 列表（队列）

```python
r.rpush("queue", "task1")              # 右入队
r.lpush("queue", "task0")              # 左入队
r.lpop("queue")                        # 左出队（FIFO）
r.rpop("queue")                        # 右出队（LIFO）
r.llen("queue")                        # 长度
r.lrange("queue", 0, -1)               # 全部元素
```

## 5. 集合 / 有序集合

```python
# Set：去重 + 集合运算
r.sadd("tags", "python", "爬虫")
r.smembers("tags")
r.sismember("tags", "python")          # 是否成员

# ZSet：带分数排序（排行榜）
r.zadd("ranking", {"user1": 100, "user2": 90})
r.zrevrange("ranking", 0, 2, withscores=True)  # 前三名
r.zincrby("ranking", 5, "user1")       # 加分
```

## 6. 过期与生存时间

```python
r.expire("key", 60)                    # 设置过期秒数
r.ttl("key")                           # 剩余时间（-1 永不过期）
r.persist("key")                       # 取消过期
```

## 7. 管道（Pipeline：减少网络往返）

```python
pipe = r.pipeline()
pipe.set("a", 1)
pipe.incr("counter")
pipe.execute()                         # 一次性发给服务器
```

## 8. 连接池（高并发必用）

```python
pool = redis.ConnectionPool(host="127.0.0.1", port=6379, db=0, max_connections=50)
r = redis.Redis(connection_pool=pool)
```

## 9. 实战模式

```python
# 爬虫去重
if r.sadd("seen:urls", url):          # 返回 1 = 首次见
    process(url)

# 简单限流
if r.setnx(f"rate:{user}", 1):
    r.expire(f"rate:{user}", 60)
else:
    print("请求过于频繁")

# 分布式锁
import uuid
token = str(uuid.uuid4())
if r.set("lock:order", token, nx=True, ex=30):
    try:
        do_work()
    finally:
        # 只释放自己的锁（防误删）
        if r.get("lock:order") == token:
            r.delete("lock:order")
```

## 10. 踩坑记录

| 坑 | 解决 |
|----|------|
| 返回 bytes 而非 str | `decode_responses=True` |
| key 太多没前缀 | 命名规范：`业务:对象:id`（如 `user:1`） |
| 大 key 阻塞 | 避免超大 hash/list，必要时拆分 |
| 数据全在内存，重启丢失 | 理解 RDB/AOF 持久化策略（见原理笔记） |

## 记忆点

> redis-py = `redis.Redis(decode_responses=True)` 一把梭；数据结构对应：字符串=缓存/计数器，哈希=对象，列表=队列，ZSet=排行榜。

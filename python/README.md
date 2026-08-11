# Python 数据库操作笔记

> Python 侧操作数据库的知识整理：SQLite / MySQL / MongoDB / Redis / SQLAlchemy ORM。
> 原理侧见 [`数据库/`](../数据库/README.md)（内核/存储系统）与 [`mysql/`](../mysql/README.md)。

## 学习内容

| 文件 | 内容 | 状态 |
|------|------|:--:|
| [01-SQLite.md](01-SQLite.md) | 内置数据库：sqlite3 CRUD / 事务 / 上下文管理 | ✅ 已整理 |
| [02-MySQL.md](02-MySQL.md) | pymysql：连接 / 参数化查询 / 事务 / 批量写入 | ✅ 已整理 |
| [03-MongoDB.md](03-MongoDB.md) | pymongo：CRUD / 条件查询 / 聚合 / 索引 / 与 pandas 结合 | ✅ 已整理 |
| [04-Redis.md](04-Redis.md) | redis-py：连接池 / 数据类型操作 / 过期 / 管道 | ✅ 已整理 |
| [05-ORM-SQLAlchemy.md](05-ORM-SQLAlchemy.md) | ORM：模型定义 / 查询 / 关系 / 迁移 / N+1 陷阱 | ✅ 已整理 |

## 选型速查

| 场景 | 选择 | 理由 |
|------|------|------|
| 本地单机 / 原型 / 学习 | SQLite | Python 内置，零配置 |
| Web 生产 / 关系型 | MySQL + SQLAlchemy | 生态成熟，ORM 防注入 |
| 爬虫中间存储 / 结构多变 | MongoDB | 文档型，字段随意长 |
| 缓存 / 计数器 / 队列 | Redis | 内存级速度，丰富数据结构 |

## 一句话主线

> Python 操作数据库 = 连接 → 建游标/集合 → CRUD → 关连接；SQL 注入靠参数化，N+1 靠预加载，性能靠索引。

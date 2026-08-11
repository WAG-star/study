# 01 SQLite（Python 内置数据库）

> 零依赖、单文件、免安装——学习数据库概念和本地原型的最佳起点。

## 1. 连接与建表

```python
import sqlite3

conn = sqlite3.connect("test.db")          # 文件不存在会自动创建
cur = conn.cursor()                         # 游标：执行 SQL 的"手"

cur.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        age  INTEGER
    )
""")
conn.commit()                               # 写操作必须提交才生效
conn.close()                                # 用完关闭
```

## 2. CRUD

```python
# 增（参数化，防 SQL 注入）
cur.execute("INSERT INTO users (name, age) VALUES (?, ?)", ("Alice", 25))
conn.commit()

# 查
cur.execute("SELECT * FROM users WHERE age > ?", (20,))
rows = cur.fetchall()                       # [(1, 'Alice', 25), ...]
row  = cur.fetchone()                       # 单行
# 查字典形式（带列名）
conn.row_factory = sqlite3.Row
rows = [dict(r) for r in cur.fetchall()]

# 改 / 删
cur.execute("UPDATE users SET age = ? WHERE name = ?", (26, "Alice"))
cur.execute("DELETE FROM users WHERE name = ?", ("Bob",))
conn.commit()
```

## 3. 事务与上下文管理（推荐写法）

```python
with sqlite3.connect("test.db") as conn:
    # with 块内自动提交；异常自动回滚
    conn.execute("INSERT INTO users (name, age) VALUES (?, ?)", ("Eve", 30))
```

## 4. 批量写入（executemany）

```python
data = [("u1", 20), ("u2", 21), ("u3", 22)]
conn.executemany("INSERT INTO users (name, age) VALUES (?, ?)", data)
conn.commit()
```

## 5. 踩坑记录

| 坑 | 解决 |
|----|------|
| 忘记 `commit()`，数据没写入 | 写操作后显式 commit，或用 with 块 |
| 字符串拼接 SQL 被注入 / 引号报错 | 一律用 `?` 占位参数化 |
| 多线程并发写报 `database is locked` | 加 `timeout` 参数，或单写者模式 |

## 记忆点

> SQLite 三连：`connect → cursor → commit`；参数化用 `?`，防注入靠占位符。

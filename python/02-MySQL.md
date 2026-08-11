# 02 MySQL（pymysql）

> 生产级关系型数据库。原理侧（B+Tree / 事务 / 锁）见 [`../mysql/README.md`](../mysql/README.md) 与 [`../数据库/`](../数据库/README.md)。

## 1. 安装与连接

```bash
pip install pymysql
```

```python
import pymysql

conn = pymysql.connect(
    host="127.0.0.1",
    port=3306,
    user="root",
    password="your_password",
    database="test_db",
    charset="utf8mb4",          # 中文必须，否则乱码
    cursorclass=pymysql.cursors.DictCursor,   # 返回字典而非元组
)
cur = conn.cursor()
```

## 2. CRUD（参数化防注入）

```python
# 增
sql = "INSERT INTO users (name, age) VALUES (%s, %s)"
cur.execute(sql, ("Alice", 25))
conn.commit()

# 查
cur.execute("SELECT * FROM users WHERE age > %s", (20,))
rows = cur.fetchall()           # 字典列表 [{...}, ...]
row  = cur.fetchone()

# 改 / 删
cur.execute("UPDATE users SET age = %s WHERE name = %s", (26, "Alice"))
cur.execute("DELETE FROM users WHERE name = %s", ("Bob",))
conn.commit()
```

**注意**：MySQL 占位符是 `%s`（不是 SQLite 的 `?`）。

## 3. 批量写入（executemany）

```python
data = [("u1", 20), ("u2", 21), ("u3", 22)]
cur.executemany("INSERT INTO users (name, age) VALUES (%s, %s)", data)
conn.commit()
```

## 4. 事务控制

```python
try:
    cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    cur.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
    conn.commit()               # 全部成功才提交
except Exception:
    conn.rollback()             # 任一失败整体回滚
    raise
```

## 5. 爬虫落库完整模式

```python
import pymysql
from contextlib import closing

def save_to_mysql(items):
    with closing(pymysql.connect(**DB_CONFIG)) as conn:
        with conn.cursor() as cur:
            cur.executemany(
                "INSERT INTO data (name, mobile, address) VALUES (%s, %s, %s)",
                items
            )
        conn.commit()
    print(f"✅ 写入 {len(items)} 条")
```

## 6. 踩坑记录

| 坑 | 解决 |
|----|------|
| 中文乱码 | `charset="utf8mb4"` + 表字段也 utf8mb4 |
| SQL 注入 / 引号报错 | 全部用 `%s` 参数化，禁止 f-string 拼 SQL |
| 忘记 commit，数据丢失 | 写操作后 commit，或用 with 管理连接 |
| 连接数耗尽 | 用完 `conn.close()`；高并发用连接池（见 DBUtils / SQLAlchemy pool） |

## 记忆点

> pymysql 四件套：`connect(charset=utf8mb4) → cursor(DictCursor) → %s 参数化 → commit`。

# 05 SQLAlchemy ORM

> Python 事实标准的 ORM：用对象操作数据库，跨 SQLite/MySQL/PostgreSQL 无缝切换。
> Flask 集成实战见 [`../python/README.md`](README.md)（04-Web开发）。

## 1. 安装

```bash
pip install sqlalchemy
# Flask 集成时：pip install flask-sqlalchemy
```

## 2. 引擎与连接

```python
from sqlalchemy import create_engine

# SQLite
engine = create_engine("sqlite:///test.db")
# MySQL（需 pymysql）
engine = create_engine("mysql+pymysql://root:password@127.0.0.1:3306/test_db?charset=utf8mb4")
# PostgreSQL
engine = create_engine("postgresql://user:password@localhost:5432/db")
```

> 迁移数据库 = 只改连接串，代码不动。

## 3. 模型定义（Declarative Base）

```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean, Text, ForeignKey
from sqlalchemy.orm import declarative_base, relationship
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(200), nullable=False)
    age = Column(Integer)
    # 一对多关系
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    slug = Column(String(200), unique=True, nullable=False, index=True)
    body = Column(Text, nullable=False)
    is_published = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    user_id = Column(Integer, ForeignKey("users.id"))
    author = relationship("User", back_populates="posts")

Base.metadata.create_all(engine)       # 建表（生产用迁移，见 §7）
```

## 4. Session（会话）

```python
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(bind=engine)
session = Session()
```

## 5. CRUD

```python
# 增
u = User(name="Alice", age=25)
session.add(u)
session.commit()

# 查（Query API）
users = session.query(User).filter(User.age > 20).all()
user  = session.query(User).filter_by(name="Alice").first()
session.query(User).order_by(User.age.desc()).limit(5).all()

# 改
u.age = 26
session.commit()

# 删
session.delete(u)
session.commit()
```

## 6. 关系查询与 N+1 陷阱（重点）

```python
# 错误：N+1 —— 每篇文章都触发一次额外查询作者
posts = session.query(Post).all()
for p in posts:
    print(p.author.name)              # 100 篇 → 1 + 100 次 SQL

# 正确：预加载（Eager Loading）
from sqlalchemy.orm import joinedload
posts = session.query(Post).options(joinedload(Post.author)).all()
# 或 selectinload(Post.author)
```

## 7. 迁移（Alembic / Flask-Migrate）

```python
# Flask 项目推荐
# pip install flask-migrate
from flask_migrate import Migrate

migrate = Migrate(app, db)            # 初始化后：
# flask db init      # 首次
# flask db migrate   # 生成迁移脚本（每次改模型后）
# flask db upgrade   # 应用迁移
```

> 禁止用 `db.create_all()` 覆盖生产表——会丢数据。结构变更走迁移脚本。

## 8. 与原生 SQL 对比

| 能力 | 裸 SQL (pymysql) | SQLAlchemy ORM |
|------|:--:|:--:|
| 手写 SQL | ✅ | 自动生成 |
| 防注入 | 参数化 | 内置 |
| 跨数据库切换 | 改 SQL 方言 | 改连接串 |
| 复杂报表查询 | ✅ 灵活 | 可用 `text()` 原生 SQL 兜底 |
| 性能调优 | 可控 | 需注意 N+1 / 懒加载 |

## 9. 踩坑记录

| 坑 | 解决 |
|----|------|
| N+1 查询性能暴跌 | `joinedload` / `selectinload` 预加载 |
| 忘记 `session.commit()` | 写操作后显式 commit，或用 contextmanager |
| 懒加载在 session 关闭后报错 | 预加载或保持 session 存活 |
| 模型改了表没变 | 迁移脚本（migrate + upgrade） |

## 记忆点

> ORM 三件套：`declarative_base 定义模型 → sessionmaker 建会话 → query().filter()` 查对象；查关系必加 `joinedload` 防 N+1。

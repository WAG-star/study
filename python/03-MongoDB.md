# 03 MongoDB（pymongo）

> 文档型 NoSQL。字段可动态增减，适合爬虫中间存储、结构多变的业务数据。

## 1. 安装与连接

```bash
pip install pymongo
```

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")   # 本地
client = MongoClient("mongodb+srv://<user>:<pwd>@cluster.mongodb.net/")  # Atlas 云
db = client["test_db"]                    # 选库（不存在则创建）
collection = db["users"]                  # 选集合（≈ SQL 的表）
```

## 2. CRUD

```python
# ── 增 ──
collection.insert_one({"name": "Alice", "age": 25})
collection.insert_many([
    {"name": "Bob", "age": 30},
    {"name": "Charlie", "age": 22},
])

# ── 查 ──
collection.find_one({"name": "Alice"})                 # 单条
for doc in collection.find({"age": {"$gte": 25}}):     # 多条（游标）
    print(doc)
for doc in collection.find({}, {"_id": 0, "name": 1}): # 投影：只要 name
    print(doc)
for doc in collection.find().sort("age", -1).limit(3): # 排序+限量
    print(doc)

# ── 改 ──
collection.update_one({"name": "Alice"}, {"$set": {"age": 26}})
collection.update_many({"age": {"$lt": 30}}, {"$inc": {"age": 1}})

# ── 删 ──
collection.delete_one({"name": "Alice"})
collection.delete_many({"age": {"$gt": 30}})

# ── 统计 ──
collection.count_documents({})
```

## 3. 条件查询（核心）

```python
# 比较
{"age": {"$eq": 25}}   {"age": {"$ne": 25}}
{"age": {"$gt": 25}}   {"age": {"$gte": 25}}
{"age": {"$lt": 25}}   {"age": {"$lte": 25}}
{"age": {"$gte": 20, "$lt": 30}}          # 区间可连写

# 集合
{"city": {"$in": ["Beijing", "Shanghai"]}}    # IN
{"city": {"$nin": ["Beijing"]}}               # NOT IN

# 逻辑组合
{"age": {"$gt": 25}, "city": "Beijing"}       # 隐式 AND
{"$and": [{"age": {"$gt": 25}}, {"city": "Beijing"}]}
{"$or": [{"age": {"$lt": 18}}, {"city": "Shanghai"}]}

# 正则（模糊）
{"name": {"$regex": "^A"}}                     # 以 A 开头
{"name": {"$regex": "li", "$options": "i"}}   # 包含 li，忽略大小写

# 数组字段
{"skills": "Python"}                          # 包含某值
{"skills": {"$all": ["Python", "SQL"]}}       # 包含全部值
{"skills.0": "Python"}                        # 按索引

# 日期范围
from datetime import datetime
{"order_date": {"$gte": datetime(2023,1,1), "$lt": datetime(2024,1,1)}}

# 字段存在性
{"field": {"$exists": True}}
```

## 4. 索引

```python
collection.create_index("name")                 # 单字段
collection.create_index([("field1", 1), ("field2", -1)])  # 复合
print(collection.index_information())           # 查看索引
```

## 5. 聚合管道（≈ SQL GROUP BY）

```python
pipeline = [
    {"$match": {"age": {"$gte": 25}}},          # WHERE
    {"$group": {"_id": "$city", "avg_age": {"$avg": "$age"}}},  # GROUP BY
    {"$sort": {"avg_age": -1}},
    {"$limit": 5},
]
for result in collection.aggregate(pipeline):
    print(result)
```

常用阶段：`$match / $group / $sort / $project / $limit / $skip`

## 6. 分页

```python
page_size, page_number = 5, 2
results = collection.find().skip((page_number-1)*page_size).limit(page_size)
```

> 大数据集分页建议用**游标分页**（基于上次 `_id` 或排序键），`skip` 越深越慢。

## 7. 与 pandas 结合（爬虫常用）

```python
import pandas as pd

query = {
    "$and": [
        {"age": {"$gte": 25, "$lte": 40}},
        {"city": {"$in": ["Beijing", "Shanghai"]}},
    ]
}
projection = {"_id": 0, "name": 1, "age": 1, "city": 1}
df = pd.DataFrame(list(collection.find(query, projection)))
```

## 8. 命令行导入导出

```bash
mongoexport -d test_db -c users -o users.json
mongoimport -d test_db -c users --file users.json
```

## 9. 踩坑记录

| 坑 | 解决 |
|----|------|
| 忘记投影 `_id`，数据里多个字段 | 查询加 `{"_id": 0, ...}` |
| 日期用字符串比较 | 必须用 `datetime` 对象 |
| 大结果集内存爆炸 | `.limit(n)` / `.batch_size(n)` 分批 |
| 凭据写死在代码 | 环境变量存放，最小权限账号 |

## 记忆点

> MongoDB 查询 = `find(条件, 投影)`；条件靠 `$` 操作符，投影 `1` 返回 `0` 隐藏（`_id` 默认返回要排掉）。

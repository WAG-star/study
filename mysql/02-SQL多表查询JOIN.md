# SQL 多表查询：JOIN 与 NULL（SQLBolt L6-L8）

> 来源：SQLBolt 交互式教程（已学完）https://sqlbolt.com/

## 1. 数据库规范化（L6 背景）

真实数据被拆到多张正交表中存储，称为**规范化（Normalization）**：

- ✅ 好处：单表重复数据最小化；数据可独立增长（引擎类型表独立于车型表）
- ⚠️ 代价：查询变复杂；多张大表 join 有性能问题

**主键（Primary Key）**：跨库唯一标识实体的键，常用自增整数（空间高效），也可用字符串/哈希。

## 2. INNER JOIN（L6）

匹配两表中 key 相同的行，合并列为结果行。

```sql
SELECT column, another_table_column, ...
FROM mytable
INNER JOIN another_table
    ON mytable.id = another_table.id
WHERE condition(s)
ORDER BY column, ... ASC/DESC
LIMIT num_limit OFFSET num_offset;
```

- `ON` 定义匹配条件（两表 key 相等）
- join 之后，之前学的 WHERE/ORDER BY/LIMIT 照常应用
- `INNER JOIN` 与 `JOIN` 等价

**关键特性**：结果只包含两表都有的行（对称数据 OK，非对称数据会丢行）。

## 3. OUTER JOIN（L7）

当两表数据**不对称**（如新楼还没员工）时，INNER JOIN 会漏数据，需用外连接：

```sql
SELECT column, another_column, ...
FROM mytable
INNER/LEFT/RIGHT/FULL JOIN another_table
    ON mytable.id = another_table.matching_id
WHERE condition(s)
ORDER BY column, ... ASC/DESC
LIMIT num_limit OFFSET num_offset;
```

| 连接类型 | 保留行 | 说明 |
|---------|--------|------|
| `LEFT JOIN` | A 表全部行 | B 无匹配则 NULL 填充（最常用） |
| `RIGHT JOIN` | B 表全部行 | A 无匹配则 NULL 填充 |
| `FULL JOIN` | 两表全部行 | 无匹配侧 NULL 填充 |

- `LEFT/RIGHT/FULL OUTER JOIN` 与省略 OUTER 等价（OUTER 仅为 SQL-92 兼容）
- 外连接结果会出现 NULL，需额外处理（见下节）

## 4. NULL 处理（L8）

**为什么尽量少用 NULL**：查询、约束、函数处理结果时都需要特殊照顾。

替代方案：
- 数值列用默认值 `0`
- 文本列用空字符串 `''`
- 但若默认值会扭曲统计（如求平均值），存 NULL 更合适
- 外连接不对称数据时 NULL 不可避免

```sql
-- 测试 NULL
SELECT column, another_column, ...
FROM mytable
WHERE column IS NULL AND/OR another_condition;

SELECT column, ...
FROM mytable
WHERE column IS NOT NULL AND/OR another_condition;
```

> 判断 NULL 必须用 `IS NULL` / `IS NOT NULL`，不能写 `= NULL`（NULL 不等于任何值包括它自己）。

## 5. 实战示例（Pixar 库）

```sql
-- 每部电影的票房（Movies 1:1 BoxOffice）
SELECT m.title, b.rating, b.sales_in_millions
FROM movies m
INNER JOIN boxoffice b ON m.id = b.movie_id;

-- 找出还没有办公室的员工（LEFT JOIN + NULL）
SELECT e.name, b.building_name
FROM employees e
LEFT JOIN buildings b ON e.building_id = b.id
WHERE b.id IS NULL;
```

## 记忆点

> 连接四选一：**INNER 只要交集，LEFT 保左表，RIGHT 保右表，FULL 保全部**；外连接查"没匹配上的" = `WHERE 对方主键 IS NULL`。

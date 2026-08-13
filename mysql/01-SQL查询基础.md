# SQL 基础：查询入门（SQLBolt L1-L5）

> 来源：SQLBolt 交互式教程（已学完）https://sqlbolt.com/
> 范围：SELECT 查询 / WHERE 约束 / 排序分页 / DISTINCT

## 1. SELECT 查询（L1）

查询的本质：**声明要找什么数据、从哪张表找、如何变换后再返回**。

```sql
-- 选指定列
SELECT column, another_column, ...
FROM mytable;

-- 选全部列（查看表结构最快方式）
SELECT * FROM mytable;
```

表格心智模型：表 = 实体类型（如 Movies），行 = 实例，列 = 共同属性。

## 2. WHERE 约束（L2-L3）

过滤行，避免全表扫描百万行数据。

```sql
SELECT column, another_column, ...
FROM mytable
WHERE condition AND/OR another_condition;
```

### 数值操作符

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `= != < <= > >=` | 标准数值比较 | `col_name != 4` |
| `BETWEEN ... AND ...` | 在范围内（含边界） | `col BETWEEN 1.5 AND 10.5` |
| `NOT BETWEEN ... AND ...` | 不在范围内 | `col NOT BETWEEN 1 AND 10` |
| `IN (...)` | 在列表中 | `col IN (2, 4, 6)` |
| `NOT IN (...)` | 不在列表中 | `col NOT IN (1, 3, 5)` |

### 文本操作符

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `=` | 大小写敏感精确匹配 | `col = "abc"` |
| `!=` 或 `<>` | 大小写敏感不等 | `col != "abcd"` |
| `LIKE` | 大小写不敏感匹配 | `col LIKE "ABC"` |
| `NOT LIKE` | 大小写不敏感不等 | `col NOT LIKE "ABCD"` |
| `%` | 任意 0+ 字符（仅配 LIKE） | `col LIKE "%AT%"` |
| `_` | 单个字符（仅配 LIKE） | `col LIKE "AN_"` |
| `IN (...)` / `NOT IN (...)` | 字符串列表 | `col IN ("A","B","C")` |

> 字符串必须加引号，否则解析器分不清字符串与关键字。
> 全文搜索（LIKE '%xx%'）数据量大时效率低，应用 Lucene/Sphinx 等专用库。

## 3. DISTINCT 去重（L4）

```sql
SELECT DISTINCT column, another_column, ...
FROM mytable
WHERE condition(s);
```

> 只按整行去重；按特定列去重后续用 GROUP BY。

## 4. ORDER BY 排序（L4）

```sql
SELECT column, ...
FROM mytable
WHERE condition(s)
ORDER BY column ASC/DESC;
```

- 按指定列字母/数值序排列
- 不指定时 ASC 升序
- 真实数据库数据通常乱序插入，排序对阅读大结果集很重要

## 5. LIMIT / OFFSET 分页（L4）

```sql
SELECT column, ...
FROM mytable
WHERE condition(s)
ORDER BY column ASC/DESC
LIMIT num_limit OFFSET num_offset;
```

- `LIMIT`：返回行数上限
- `OFFSET`：跳过前 N 行（分页起点）
- 网站翻页 = 不同 OFFSET 的查询（Reddit/Pinterest 模式）
- 执行时机：在所有其他子句之后（详见 L12）

## 6. 完整查询骨架（L5 复习）

```sql
SELECT column, another_column, ...
FROM mytable
WHERE condition(s)
ORDER BY column ASC/DESC
LIMIT num_limit OFFSET num_offset;
```

## 记忆点

> 查询五件套：**SELECT → FROM → WHERE → ORDER BY → LIMIT/OFFSET**；过滤用 WHERE，去重用 DISTINCT，排序用 ORDER BY，分页用 LIMIT OFFSET。

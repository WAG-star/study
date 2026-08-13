# SQL 表达式、聚合与分组（SQLBolt L9-L11）

> 来源：SQLBolt 交互式教程（已学完）https://sqlbolt.com/

## 1. 查询表达式（L9）

在查询中对列值做数学/字符串/日期变换，节省事后处理：

```sql
-- 数学表达式 + AS 别名
SELECT particle_speed / 2.0 AS half_particle_speed
FROM physics_data
WHERE ABS(particle_position) * 10.0 > 500;

-- 列与表都能取别名
SELECT column AS better_column_name, ...
FROM a_long_widgets_table_name AS mywidgets
INNER JOIN widget_sales
    ON mywidgets.id = widget_sales.widget_id;
```

> 表达式会降低查询可读性 → SELECT 中的表达式务必用 `AS` 给描述性别名。

## 2. 聚合函数（L10）

对一组行做汇总，返回单个值：

| 函数 | 说明 |
|------|------|
| `COUNT(*)` | 组内行数；`COUNT(column)` 只数非 NULL 值 |
| `MIN(column)` | 组内最小值 |
| `MAX(column)` | 组内最大值 |
| `AVG(column)` | 组内平均值 |
| `SUM(column)` | 组内数值总和 |

```sql
-- 全表聚合（不加 GROUP BY → 整个结果集算一个组）
SELECT AGG_FUNC(column_or_expression) AS aggregate_description, ...
FROM mytable
WHERE constraint_expression;
```

> 与表达式一样，聚合函数加别名便于阅读处理。

## 3. GROUP BY 分组聚合（L10）

按某列相同值分组，每组各自聚合：

```sql
SELECT AGG_FUNC(column_or_expression) AS aggregate_description, ...
FROM mytable
WHERE constraint_expression
GROUP BY column;
```

- 结果行数 = 该列去重后的组数
- 经典问题："Pixar 每年票房最高的电影" = 按 year 分组后 MAX

## 4. HAVING 过滤分组（L11）

WHERE 在分组**前**过滤行；HAVING 在分组**后**过滤组：

```sql
SELECT group_by_column, AGG_FUNC(column_expression) AS aggregate_result_alias, ...
FROM mytable
WHERE condition            -- 先过滤行
GROUP BY column            -- 再分组
HAVING group_condition;    -- 最后过滤组
```

- HAVING 写法同 WHERE，但作用于分组后的结果
- 没写 GROUP BY 时，普通 WHERE 就够用

## 5. 实战示例

```sql
-- 每个团队的人数
SELECT team, COUNT(*) AS member_count
FROM employees
GROUP BY team;

-- 有 2 人以上的团队（过滤组）
SELECT team, COUNT(*) AS member_count
FROM employees
GROUP BY team
HAVING COUNT(*) > 2;

-- 每年上映电影数与最高票房（Pixar）
SELECT year, COUNT(*) AS movies, MAX(grossing) AS top_gross
FROM movies
GROUP BY year;
```

## 记忆点

> 聚合五函数：**COUNT/MIN/MAX/AVG/SUM**；WHERE 过滤行、HAVING 过滤组、GROUP BY 按列分组——三者顺序别搞混。

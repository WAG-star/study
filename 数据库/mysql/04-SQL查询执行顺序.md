# SQL 查询执行顺序（SQLBolt L12）

> 来源：SQLBolt 交互式教程（已学完）https://sqlbolt.com/
> 本节是理解"为什么 WHERE 不能用别名、HAVING 能过滤聚合"的关键。

## 完整 SELECT 语句

```sql
SELECT DISTINCT column, AGG_FUNC(column_or_expression), ...
FROM mytable
    JOIN another_table ON mytable.column = another_table.column
WHERE constraint_expression
GROUP BY column
HAVING constraint_expression
ORDER BY column ASC/DESC
LIMIT count OFFSET COUNT;
```

## 执行顺序（8 步）

| 顺序 | 子句 | 作用 | 关键限制 |
|:--:|------|------|---------|
| 1 | `FROM` + `JOIN` | 确定总工作集（含子查询，可能建临时表） | 决定后续能访问哪些列 |
| 2 | `WHERE` | 逐行过滤，丢弃不满足的行 | **只能访问 FROM 中的列，SELECT 别名不可用** |
| 3 | `GROUP BY` | 按列值分组 | 结果行数 = 唯一值个数；有聚合函数时才需要 |
| 4 | `HAVING` | 过滤分组后的组 | **同样不能用 SELECT 别名** |
| 5 | `SELECT` | 计算选择表达式（聚合、数学变换） | 到这里才真正算表达式 |
| 6 | `DISTINCT` | 丢弃重复行 | 作用于 SELECT 结果 |
| 7 | `ORDER BY` | 排序 | **可以引用 SELECT 别名**（表达式已算完） |
| 8 | `LIMIT / OFFSET` | 截取范围，返回最终结果 | 最后执行 |

## 为什么要懂顺序

1. **WHERE 不能用别名**：别名可能依赖尚未执行的表达式（步骤 5 才计算）
2. **HAVING 不能引用 SELECT 别名**：同样因为聚合/表达式还没算
3. **ORDER BY 可以用别名**：执行时表达式已完成
4. **LIMIT/OFFSET 最后执行**：数据库能先处理完过滤排序再取子集，因此高效

## 记忆点

> 执行顺序口诀：**FROM-JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT**。
> "别名晚于 WHERE/HAVING 生效，早于 ORDER BY 生效"——这是最容易踩的坑。

# SQL 数据修改与表结构（SQLBolt L13-L18）

> 来源：SQLBolt 交互式教程（已学完）https://sqlbolt.com/
> 范围：INSERT / UPDATE / DELETE / CREATE TABLE / ALTER TABLE / DROP TABLE

## 1. Schema 概念（L13）

数据库 schema = 每张表的结构描述：列名 + 数据类型 + 约束 + 默认值。

- 固定结构让数据库在存储数百万行时仍高效一致
- 例：Movies 表的 Year 列必须是 Integer，Title 必须是 String

## 2. INSERT 插入行（L13）

```sql
-- 全列插入（值顺序必须与表列顺序一致）
INSERT INTO mytable
VALUES (value_or_expr, another_value_or_expr, ...),
       (value_2, another_value_2, ...);

-- 指定列插入（推荐：前向兼容）
INSERT INTO mytable (column, another_column, ...)
VALUES (value_or_expr, another_value_or_expr, ...),
       (value_2, another_value_2, ...);

-- 值可用表达式
INSERT INTO boxoffice (movie_id, rating, sales_in_millions)
VALUES (1, 9.9, 283742034 / 1000000);
```

指定列插入的好处：
- 表新增带默认值的列时，已写的 INSERT 不用改
- 数量必须匹配：值的个数 = 指定的列数

## 3. UPDATE 更新行（L14）⚠️ 高危操作

```sql
UPDATE mytable
SET column = value_or_expr, other_column = another_value_or_expr, ...
WHERE condition;
```

**安全铁律**（官方强调）：
1. **先写 WHERE 再写 SET**——不写 WHERE 会更新所有行！
2. 先跑 SELECT 验证 WHERE 选中的行正确，再执行 UPDATE
3. 更新值必须匹配列的数据类型

```sql
-- 安全流程：先查
SELECT * FROM movies WHERE title = 'Toy Story';
-- 确认无误再改
UPDATE movies SET director = 'John Lasseter' WHERE title = 'Toy Story';
```

## 4. DELETE 删除行（L15）⚠️ 更危险

```sql
-- 按条件删
DELETE FROM mytable WHERE condition;

-- 不带 WHERE = 清空整张表（若有意为之才用）
DELETE FROM mytable;
```

**安全铁律**：
1. 同样先 SELECT 验证 WHERE 选中的行
2. 没有备份/测试库时，误删不可恢复——**读两遍再执行一次**

## 5. CREATE TABLE 建表（L16）

```sql
CREATE TABLE IF NOT EXISTS mytable (
    column DataType TableConstraint DEFAULT default_value,
    another_column DataType TableConstraint DEFAULT default_value,
    ...
);
```

### 常见数据类型

| 类型 | 说明 |
|------|------|
| `INTEGER, BOOLEAN` | 整数（布尔在部分实现是 0/1） |
| `FLOAT, DOUBLE, REAL` | 浮点数（按精度需求选） |
| `CHARACTER(n), VARCHAR(n), TEXT` | 字符串；CHAR/VARCHAR 限最大字符数更高效 |
| `DATE, DATETIME` | 日期时间（跨时区处理较麻烦） |
| `BLOB` | 二进制（数据库不解析，需自带元数据） |

### 常见表约束

| 约束 | 说明 |
|------|------|
| `PRIMARY KEY` | 值唯一，可标识单行 |
| `AUTOINCREMENT` | 整数自动递增 |
| `UNIQUE` | 值唯一 |
| `NOT NULL` | 不可为空 |
| `CHECK (expression)` | 值需满足表达式 |
| `FOREIGN KEY` | 引用其他表主键 |

`IF NOT EXISTS`：表已存在时跳过创建而非报错。

## 6. ALTER TABLE 改表结构（L17）

```sql
-- 加列（需指定类型/约束/默认值）
ALTER TABLE mytable ADD column DataType OptionalTableConstraint DEFAULT default_value;

-- 删列（部分数据库如 SQLite 不支持，需建新表迁移数据）
ALTER TABLE mytable DROP column_to_be_deleted;

-- 改表名
ALTER TABLE mytable RENAME TO new_table_name;
```

- MySQL 可用 `FIRST` / `AFTER` 指定新列位置（非标准）
- 不同数据库支持差异大，操作前查对应文档

## 7. DROP TABLE 删表（L18）

```sql
DROP TABLE IF EXISTS mytable;
```

- **与 DELETE 的区别**：DROP 连 schema 一起删，表结构彻底消失；DELETE 只删数据
- 有 FOREIGN KEY 依赖时，需先处理依赖表
- `IF EXISTS`：表不存在时跳过而非报错

## 8. 安全操作对比

| 操作 | 风险 | 防护 |
|------|------|------|
| UPDATE 不带 WHERE | 全表改 | 先 SELECT 验证 WHERE |
| DELETE 不带 WHERE | 全表清空 | 先 SELECT 验证 WHERE |
| DROP TABLE | 表和结构全没 | 先备份 |
| ALTER DROP COLUMN | 数据丢失 | 部分库不支持，先迁移 |

## 记忆点

> 改数据三件套：**INSERT 加行、UPDATE 改行、DELETE 删行**；改表结构三件套：**CREATE 建表、ALTER 加删列、DROP 删表**。
> 高危操作统一口诀：**先 SELECT 验证，再执行修改；WHERE 永远不能忘**。

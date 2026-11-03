	# SQL 条件判断与聚合统计

使用 `CASE WHEN` 为每一行计算条件结果，再用 `COUNT` 或 `SUM` 汇总，可以在一次查询中得到总数和多种条件的数量。本笔记的示例适用于 MySQL 8.0 及支持公共表表达式的 SQLite；实际结果还应结合所用数据库及版本验证。

## 原理

### CASE WHEN：按条件生成值

```sql
CASE
    WHEN condition_1 THEN result_1
    WHEN condition_2 THEN result_2
    ELSE default_result
END
```

- 按顺序判断条件，返回第一个为真的分支结果；没有命中时返回 `ELSE`，省略 `ELSE` 时返回 `NULL`。
- 可以类比 Java 的 `if-else` 或三元表达式，但 SQL 条件还存在 `UNKNOWN`：例如 `NULL = 'PAID'` 的结果不是 `TRUE`。判断空值应使用 `IS NULL` 或 `IS NOT NULL`。
- `CASE` 是表达式，可以用在 `SELECT` 中生成展示值，也可以作为聚合函数的输入。

### COUNT 与 SUM：是否忽略 NULL

下表中的“空输入”指 `WHERE` 过滤后没有任何行，且查询不含 `GROUP BY` 的聚合结果；有行时，`column` 是每行的某个字段。

| 表达式 | 作用 | 空输入结果 |
| --- | --- | --- |
| `COUNT(*)` | 统计输入行数，包括字段值为 `NULL` 的行 | `0` |
| `COUNT(1)` / `COUNT(0)` | 统计常量表达式非 `NULL` 的行；常量 `1`、`0` 均非 `NULL`，因此统计行数 | `0` |
| `COUNT(column)` | 仅统计 `column` 非 `NULL` 的行，不等同于去重计数 | `0` |
| `SUM(1)` | 每行累加 `1`，有输入行时结果为行数 | `NULL` |
| `SUM(0)` | 每行累加 `0`，有输入行时结果为 `0` | `NULL` |
| `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` | 每行满足条件记 `1`，否则记 `0`，再求和 | `NULL` |

`COUNT` 统计行或非 `NULL` 表达式的数量；`SUM` 对非 `NULL` 数值求和，没有可求和的非 `NULL` 值时返回 `NULL`。`COUNT(*)` 清楚表达“统计行数”；`COUNT(1)` 通常不会带来有意义的性能优势，具体执行方式和性能仍取决于数据库、版本、存储引擎、索引及执行计划，应以实际测试为准。

## 示例

以下自包含示例使用虚构订单数据，分别计算总数、非空金额数和已支付订单数：

```sql
WITH orders (order_id, status, amount) AS (
    SELECT 1, 'PAID', 100
    UNION ALL SELECT 2, 'PENDING', NULL
    UNION ALL SELECT 3, 'PAID', 50
    UNION ALL SELECT 4, NULL, 20
)
SELECT
    COUNT(*) AS total_count,
    COUNT(1) AS count_one,
    COUNT(0) AS count_zero,
    COUNT(amount) AS amount_count,
    SUM(1) AS sum_one,
    SUM(0) AS sum_zero,
    SUM(CASE WHEN status = 'PAID' THEN 1 ELSE 0 END) AS paid_count,
    COUNT(CASE WHEN status = 'PAID' THEN 1 END) AS paid_count_by_count
FROM orders;
```

预期依次返回 `4、4、4、3、4、0、2、2`。最后一种写法中，未命中条件时 `CASE` 返回 `NULL`，因此不被 `COUNT` 计入。

### 无匹配数据与 COALESCE

```sql
WITH orders (order_id, status) AS (
    SELECT 1, 'PAID'
)
SELECT
    COUNT(*) AS total_count,
    SUM(1) AS sum_one,
    SUM(0) AS sum_zero,
    SUM(CASE WHEN status = 'PAID' THEN 1 ELSE 0 END) AS paid_count,
    COALESCE(SUM(CASE WHEN status = 'PAID' THEN 1 ELSE 0 END), 0) AS paid_count_or_zero
FROM orders
WHERE order_id < 0;
```

预期依次返回 `0、NULL、NULL、NULL、0`。`COALESCE` 返回参数中第一个非 `NULL` 的值，可按业务需要将空聚合结果转换为 `0`。

## 注意事项

- **无匹配条件不等于无输入行。** 有输入行但没有 `PAID` 记录时，上述带 `ELSE 0` 的条件求和返回 `0`；输入行为空时返回 `NULL`。`COALESCE` 可以统一为 `0`，但只有业务允许把“无数据”视为零时才这样处理。
- **省略 `ELSE 0` 会改变求和结果。** `SUM(CASE WHEN condition THEN 1 END)` 在没有任何行满足条件时，即使存在输入行，也会因所有结果为 `NULL` 而返回 `NULL`。
- **分组时不存在的组不会自动生成。** `GROUP BY` 后，每个实际存在的组各自聚合；如果过滤后没有任何行，通常没有结果行，`COALESCE` 不能凭空生成缺失的组。需要展示零值组时，应考虑使用完整维度数据及外连接，并注意外连接产生的空扩展行可能被 `COUNT(*)` 计入。
- **`COUNT(column)` 统计非空，不统计不同值。** 如需不同非空值数量，应按需求使用 `COUNT(DISTINCT column)`。
- **过滤与条件统计的范围不同。** `WHERE` 先限定参与聚合的输入行；`CASE` 在这些输入行内分别计算不同指标。不要为了统计某一个状态而在 `WHERE` 中排除其他指标所需的数据。

## 实际开发应用

在订单看板中，可以用一次聚合查询同时计算订单总数、已支付数和待支付数，再根据需要按日期或渠道分组。先明确每个指标的过滤范围、`NULL` 和空数据语义；数据量较大时，结合查询条件、索引、执行计划和实际数据量评估性能，不根据 `COUNT(*)` 与 `COUNT(1)` 的写法直接推断快慢。

## 资料来源

- 用户提供的 SQL 条件判断与聚合素材；上面的边界说明和示例为整理补充。
- MySQL 8.4 参考手册：[聚合函数](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html)、[流程控制函数](https://dev.mysql.com/doc/refman/8.4/en/flow-control-functions.html) 和 [比较函数与运算符](https://dev.mysql.com/doc/refman/8.4/en/comparison-operators.html)。本次官方文档访问服务不可用，链接内容尚未在线核验。

返回 [[数据库知识索引]]。

---
title: 慢查询治理：让 EXPLAIN 说话
date: 2023-03-14 14:00:00
tags:
  - Career
categories:
  - Career
description: 慢查询优化不是玄学，是让 EXPLAIN 执行计划说话。type、key、rows、Extra 怎么看，真实案例拆解。
---

前面讲了慢查询监控怎么搭。监控搭好之后，每周从 Top 慢查询里挑几条出来优化——这就是"慢查询治理"。

做了半年之后我发现一件事：大多数慢查询的根因就那几类。索引没建、索引建了但没用上、SQL 写法导致优化器选错执行计划。你不需要是什么 MySQL 源码级专家，只要能读懂 `EXPLAIN` 的输出，就能解决 80% 的慢查询。

这篇讲怎么读 EXPLAIN，以及我们治过的几种典型慢查询。

## EXPLAIN 怎么读

随便拿一条 SQL：

```sql
SELECT * FROM t_xxx_orders
WHERE user_id = 12345 AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

`EXPLAIN` 一下：

```sql
EXPLAIN SELECT * FROM t_xxx_orders
WHERE user_id = 12345 AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

输出大概长这样：

```
+----+-------------+-------------+------+---------------+------+---------+------+----------+-----------------------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows     | Extra                       |
+----+-------------+-------------+------+---------------+------+---------+------+----------+-----------------------------+
|  1 | SIMPLE      | t_xxx_orders| ALL  | NULL          | NULL | NULL    | NULL | 8500000  | Using where; Using filesort |
+----+-------------+-------------+------+---------------+------+---------+------+----------+-----------------------------+
```

这条 SQL 是典型的慢查询。逐列讲：

### type 列：访问类型

这是最重要的列。从好到差：

- `system` / `const`：通过主键或唯一索引等值查询，最快。
- `eq_ref`：JOIN 时用主键或唯一索引匹配，一行对一行。
- `ref`：通过普通索引等值查询。
- `range`：索引范围扫描（`>`, `<`, `BETWEEN`, `IN`）。
- `index`：扫描整个索引树。
- `ALL`：全表扫描，最差。

上面那个例子 `type = ALL`，说明没走任何索引，扫了 850 万行。这就是问题所在。

看到 `type = ALL` 基本就可以判定索引有问题了。

### key 列：实际用的索引

`possible_keys` 列出"可能用到的索引"，`key` 是"实际用了哪个"。如果 `possible_keys` 有值但 `key` 是 NULL，说明优化器选了全表扫描——这时候得想想为什么。

### rows 列：预估扫描行数

这个数字是优化器的估算值，越小越好。一条查询如果 rows 是百万级，不管别的列好不好看，基本都是慢查询。

### Extra 列：附加信息

这里面藏的信息很关键。常见的：

- `Using where`：用了 WHERE 条件过滤（正常）。
- `Using index`：覆盖索引，不用回表，好东西。
- `Using filesort`：需要额外的排序操作，通常意味着 ORDER BY 没走索引。
- `Using temporary`：用了临时表，常见于 GROUP BY、DISTINCT，要警惕。

上面那个例子 `Extra = Using where; Using filesort`，说明既有全表扫描又有额外排序，双重打击。

## 治过的几种典型慢查询

### 类型一：没建索引

就是上面那条 SQL。解决方法很简单：

```sql
CREATE INDEX idx_uid_status_created ON t_xxx_orders(user_id, status, created_at);
```

联合索引把 `WHERE` 条件的 `user_id` 和 `status` 放前面，`ORDER BY` 的 `created_at` 放最后。加完之后再看 EXPLAIN：

```
+----+-------------+-------------+-------+----------------------+----------------------+---------+------+------+--------------------------+
| id | select_type | table       | type  | possible_keys        | key                  | key_len | ref  | rows | Extra                    |
+----+-------------+-------------+-------+----------------------+----------------------+---------+------+------+--------------------------+
|  1 | SIMPLE      | t_xxx_orders| ref   | idx_uid_status_created| idx_uid_status_created| 12      | const|  120 | Using where; Using index |
+----+-------------+-------------+-------+----------------------+----------------------+---------+------+------+--------------------------+
```

`type` 从 ALL 变成 `ref`，`rows` 从 850 万降到 120，`Extra` 出现了 `Using index`（覆盖索引）。扫描行数降了 7 万倍，查询从 2 秒变成 2 毫秒。

加一个索引解决 99% 的性能问题，这句话不夸张。

### 类型二：索引建了但没用上

这条 SQL：

```sql
SELECT * FROM t_xxx_orders WHERE DATE(created_at) = '2023-03-01';
```

表上有 `created_at` 的索引，但 EXPLAIN 显示走了全表扫描。为什么？

因为对索引列用了函数。`DATE(created_at)` 让优化器无法直接走索引。改成：

```sql
SELECT * FROM t_xxx_orders
WHERE created_at >= '2023-03-01' AND created_at < '2023-03-02';
```

立刻走索引了。

类似地，这些写法都会让索引失效：

```sql
-- 用了函数
WHERE LEFT(name, 3) = 'abc'
-- 用了运算
WHERE id + 1 = 100
-- 用了隐式类型转换（user_id 是 VARCHAR，传入 INT）
WHERE user_id = 12345
-- 用了 LIKE 前缀通配
WHERE name LIKE '%abc'
```

### 类型三：深分页

```sql
SELECT * FROM t_xxx_orders ORDER BY id LIMIT 1000000, 20;
```

MySQL 要先扫描前 100 万行再丢弃，然后返回 20 行。越往后翻越慢。

解决方法是**游标分页**（也叫 keyset pagination）：

```sql
-- 上一页最后一条的 id 是 1000099
SELECT * FROM t_xxx_orders
WHERE id > 1000099
ORDER BY id
LIMIT 20;
```

走主键索引，直接定位，不管翻到第几页都一样快。

### 类型四：GROUP BY 慢

```sql
SELECT user_id, COUNT(*)
FROM t_xxx_orders
WHERE created_at >= '2023-03-01'
GROUP BY user_id;
```

EXPLAIN 里出现 `Using temporary; Using filesort`。GROUP BY 默认会排序，如果不需要排序，加上 `ORDER BY NULL`（MySQL 8.0 之前有效）或者干脆给 GROUP BY 的列加索引。

不过老实说，大数据量的 GROUP BY 不应该放在 MySQL 里做。这种统计类查询更适合扔到数仓或者用预聚合表。我们后来把这类需求推到了 OLAP 平台，MySQL 只管行存业务数据。

## 治理节奏

慢查询治理不是一次性运动，而是持续的日常。

我们的节奏：

- 每周一从监控里拉出 Top 20 慢查询。
- DAL 小组认领，逐条 EXPLAIN 分析。
- 能加索引的加索引，能改 SQL 的推动业务方改。
- 改完之后观察一周，确认生效。

慢查询治理最怕的不是"难"，而是"放着不管"。

每条慢查询其实都不难治，难的是建立机制持续去做。一旦停下来不管，慢查询会越积越多，最后积重难返。

---
title: Slow-Query Tuning: Let EXPLAIN Do the Talking
date: 2023-03-14 14:00:00
tags:
  - Career
categories:
  - Career
description: Slow-query optimization isn't mysticism — it's letting the EXPLAIN execution plan speak. How to read type, key, rows, and Extra, with real examples.
lang: en
---

Earlier I covered how we built slow-query monitoring. After monitoring was in place, every week we'd pull a few entries from the Top slow queries and optimize them. That's "slow-query governance."

Half a year in, I noticed something: the root causes of most slow queries fall into just a few categories. Missing indexes, indexes that exist but aren't being used, SQL patterns that mislead the optimizer. You don't need to be a MySQL source-code expert. If you can read `EXPLAIN` output, you can fix 80% of slow queries.

This post is about reading EXPLAIN, and the typical slow-query patterns we've treated.

## How to read EXPLAIN

Take a random SQL:

```sql
SELECT * FROM t_xxx_orders
WHERE user_id = 12345 AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

Run `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM t_xxx_orders
WHERE user_id = 12345 AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

The output looks roughly like this:

```
+----+-------------+-------------+------+---------------+------+---------+------+----------+-----------------------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows     | Extra                       |
+----+-------------+-------------+------+---------------+------+---------+------+----------+-----------------------------+
|  1 | SIMPLE      | t_xxx_orders| ALL  | NULL          | NULL | NULL    | NULL | 8500000  | Using where; Using filesort |
+----+-------------+-------------+------+---------------+------+---------+------+----------+-----------------------------+
```

This SQL is a textbook slow query. Column by column:

### The type column: access type

This is the most important column. From best to worst:

- `system` / `const`: equality lookup on primary key or unique index. Fastest.
- `eq_ref`: JOIN matching on primary key or unique index, one-to-one.
- `ref`: equality lookup on a non-unique index.
- `range`: index range scan (`>`, `<`, `BETWEEN`, `IN`).
- `index`: scanning the entire index tree.
- `ALL`: full table scan. Worst.

In the example above, `type = ALL` means no index is being used, so 8.5 million rows are scanned. That's the problem.

> Seeing `type = ALL` is almost always a sign that something is wrong with indexing.

### The key column: actual index used

`possible_keys` lists "indexes that could be used"; `key` is "which one was actually used." If `possible_keys` has values but `key` is NULL, the optimizer chose a full table scan, and you need to figure out why.

### The rows column: estimated rows scanned

This is the optimizer's estimate. Smaller is better. If `rows` is in the millions, regardless of what the other columns look like, it's almost certainly a slow query.

### The Extra column: additional info

A lot of detail hides here. Common values:

- `Using where`: filtering with a WHERE clause (normal).
- `Using index`: covering index, no table lookup needed. Good news.
- `Using filesort`: extra sort operation needed, usually means ORDER BY isn't using an index.
- `Using temporary`: a temporary table was used. Common with GROUP BY and DISTINCT. Be alert.

In the example above, `Extra = Using where; Using filesort` means both a full table scan and an extra sort, a double blow.

## Typical slow queries we've treated

### Type 1: no index

That SQL above. The fix is straightforward:

```sql
CREATE INDEX idx_uid_status_created ON t_xxx_orders(user_id, status, created_at);
```

The composite index puts the `WHERE` columns (`user_id`, `status`) first and the `ORDER BY` column (`created_at`) last. After adding it, EXPLAIN looks like:

```
+----+-------------+-------------+-------+----------------------+----------------------+---------+------+------+--------------------------+
| id | select_type | table       | type  | possible_keys        | key                  | key_len | ref  | rows | Extra                    |
+----+-------------+-------------+-------+----------------------+----------------------+---------+------+------+--------------------------+
|  1 | SIMPLE      | t_xxx_orders| ref   | idx_uid_status_created| idx_uid_status_created| 12      | const|  120 | Using where; Using index |
+----+-------------+-------------+-------+----------------------+----------------------+---------+------+------+--------------------------+
```

`type` went from ALL to `ref`. `rows` went from 8.5 million to 120. `Extra` now shows `Using index` (covering index). Scanned rows dropped by a factor of 70,000. The query went from 2 seconds to 2 milliseconds.

> "Adding one index solves 99% of performance problems" is not an exaggeration.

### Type 2: index exists but isn't being used

This SQL:

```sql
SELECT * FROM t_xxx_orders WHERE DATE(created_at) = '2023-03-01';
```

There's an index on `created_at`, but EXPLAIN shows a full table scan. Why?

Because the column was wrapped in a function. `DATE(created_at)` prevents the optimizer from using the index directly. Rewrite as:

```sql
SELECT * FROM t_xxx_orders
WHERE created_at >= '2023-03-01' AND created_at < '2023-03-02';
```

Now it uses the index.

Similarly, these patterns all disable indexes:

```sql
-- Function on column
WHERE LEFT(name, 3) = 'abc'
-- Arithmetic on column
WHERE id + 1 = 100
-- Implicit type conversion (user_id is VARCHAR, INT passed in)
WHERE user_id = 12345
-- LIKE with leading wildcard
WHERE name LIKE '%abc'
```

### Type 3: deep pagination

```sql
SELECT * FROM t_xxx_orders ORDER BY id LIMIT 1000000, 20;
```

MySQL has to scan the first 1 million rows, discard them, then return 20. The further you page, the slower it gets.

The fix is cursor pagination (also called keyset pagination):

```sql
-- The id of the last row on the previous page is 1000099
SELECT * FROM t_xxx_orders
WHERE id > 1000099
ORDER BY id
LIMIT 20;
```

Uses the primary-key index, positions directly. Equally fast no matter how deep you page.

### Type 4: slow GROUP BY

```sql
SELECT user_id, COUNT(*)
FROM t_xxx_orders
WHERE created_at >= '2023-03-01'
GROUP BY user_id;
```

EXPLAIN shows `Using temporary; Using filesort`. GROUP BY sorts by default; if you don't need sorting, add `ORDER BY NULL` (effective before MySQL 8.0) or just add an index on the GROUP BY column.

Honestly though, large-data-volume GROUP BY shouldn't be done in MySQL at all. Statistical queries like this belong in a data warehouse or a pre-aggregated table. We eventually pushed such requirements to an OLAP platform; MySQL only handles row-store business data.

## Governance rhythm

Slow-query governance isn't a one-off campaign. It's a sustained routine.

Our rhythm:

- Every Monday, pull the Top 20 slow queries from monitoring.
- The DAL team claims them and analyzes each with EXPLAIN.
- Add indexes where possible; push business teams to rewrite SQL where needed.
- Observe for a week after the fix to confirm it worked.

> The hardest thing about slow-query governance isn't technical difficulty. It's the discipline of not ignoring them.

Each slow query is individually easy to fix. The hard part is building a mechanism that keeps doing it. Once you stop, they accumulate and eventually become an unmanageable mess.

---
title: A DDL/DML Review Checklist
date: 2022-02-15 14:00:00
tags:
  - Career
categories:
  - Career
description: What does the DAL team actually look at when reviewing DDL and DML? One checklist covering indexes, locks, and migration risks.
lang: en
---

After enough time doing DAL team reviews, I realized that most SQL problems fall into just a handful of categories. Wrong indexes, underestimated locks, migration consequences nobody thought through. New reviewers would flail around, glancing here and there, missing the key points.

So I consolidated everything into a checklist and walked through it on every review. This post lays that checklist out.

## The DDL Checklist

### 1. Index Design

This is the main event. 80% of slow queries trace back to bad indexes.

**What to check:**

- Will the new column show up in WHERE clauses? If yes, consider indexing it.
- Is the index's cardinality high enough? A status column with only 0/1 values makes for a nearly useless standalone index.
- Can a **composite index** replace several single-column ones? Column order matters: equality conditions first, range conditions after.

```sql
-- Anti-pattern: standalone index on a low-cardinality column, basically useless
CREATE INDEX idx_status ON t_xxx_orders(status);

-- Better: composite index, high cardinality first
CREATE INDEX idx_uid_status ON t_xxx_orders(user_id, status);
```

- Are there redundant indexes? If `(user_id, status)` already exists, a separate `(user_id)` index is wasted space.

### 2. Column Types

Picking the wrong type is a long-term headache. I've seen far too many people shove a status value into `VARCHAR(255)` when `TINYINT` would do.

- Status and enum values: `TINYINT` or `SMALLINT`. Not strings.
- Money: `DECIMAL`. Not `FLOAT`. Floating-point precision issues will ruin your day.
- Timestamps: Think carefully about `DATETIME` vs `TIMESTAMP`. `TIMESTAMP` has the 2038 problem but takes half the space.

### 3. Lock Impact

DDL in MySQL is not lock-free.

- Before MySQL 5.6, `ALTER TABLE` locked the entire table. Both reads and writes were blocked.
- 5.6 onward introduced Online DDL, so most operations can be done "online". But not all operations are supported. Adding a `NOT NULL` column to a large existing table, for example, still locks.
- Large tables (millions of rows and up): always use pt-online-schema-change or gh-ost, never naked DDL.

```bash
# gh-ost is more modern than pt-osc — it swaps triggers for binlog subscription
gh-ost \
  --alter "ADD COLUMN remark VARCHAR(200)" \
  --database=t_xxx --table=orders \
  --execute
```

### 4. Migration Risk

This is the part newcomers ignore most often. After you add a column, will the old code break?

Example: an API used to return an order as JSON. Now there's an extra field, and the downstream service's deserialization POJO doesn't have that field. Most of the time Jackson ignores unknown fields, but if `FAIL_ON_UNKNOWN_PROPERTIES` is configured, it blows up.

So before DDL goes live, confirm:

- Does the new column have a default value?
- Will old code error when it reads the new column?
- Does the downstream service handle backward compatibility?
- What's the rollback plan: drop the column, or leave it alone?

## The DML Checklist

### 1. WHERE Clauses on UPDATE / DELETE

**This is the highest-incident zone.** I've watched someone write `UPDATE t_xxx_orders SET status = 1;` and forget the WHERE clause. One second later, the whole table is updated. gg.

Must-check during review:

- Is there a WHERE clause at all? (Sounds obvious, but people forget.)
- Does the WHERE clause hit an index or trigger a full table scan? `EXPLAIN` tells you.
- How many rows are affected? If `EXPLAIN` shows the `rows` column in the millions, stop and reconsider.
- Large batch updates must be done in chunks, not all at once.

```sql
-- Large updates in batches, 1000 rows per batch
UPDATE t_xxx_orders
SET status = 2
WHERE status = 1 AND id <= 1000000
LIMIT 1000;
```

### 2. Transaction Size

Updating too many rows in a single transaction causes:

- Undo log bloat, eating disk space.
- Long transactions blocking other queries, with lock wait times spiking.
- Replication lag ballooning.

Rule of thumb: no more than 10,000 rows updated per transaction. If you exceed that, split it.

### 3. Batch INSERT vs Single-Row

Batch `INSERT` is dozens of times faster than looping single `INSERT`s. No need to over-explain this one.

```sql
-- Anti-pattern: looping single inserts
INSERT INTO t_xxx_log(user_id, action) VALUES (1, 'login');
INSERT INTO t_xxx_log(user_id, action) VALUES (2, 'login');

-- Better: batch insert
INSERT INTO t_xxx_log(user_id, action) VALUES (1, 'login'), (2, 'login');
```

## One Table to Sum It Up

| Check | DDL | DML | Severity |
|-------|-----|-----|----------|
| WHERE hits an index? | - | Always | Fatal |
| Lock risk? | Always | Sometimes | High |
| Column type sensible? | Always | - | Medium |
| Affected rows controllable? | Always | Always | High |
| Rollback plan? | Always | Always | Fatal |
| Downstream compatibility? | Always | Sometimes | High |

## One Last Thing

The whole point of a checklist is using it every time, not cherry-picking.

> **Review isn't about nitpicking. It's about turning "should be fine" into "confirmed fine."**

People assume "this SQL is simple, it'll be fine", and then watch it blow up. The simple stuff is what catches you off guard, because you let your guard down. Build the habit: run the checklist every single review, no matter what the SQL looks like.

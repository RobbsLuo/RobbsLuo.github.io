---
title: Sharding Research: Why We Didn't Adopt a Middleware
date: 2023-11-13 15:00:00
tags:
  - Career
categories:
  - Career
description: We spent half a year researching and piloting sharding. The conclusion: no middleware for now — self-built read/write split plus Redis caching is enough. Here's the cost/benefit/risk reasoning.
lang: en
---

Around 2023, business volume was climbing fast, and several core tables were heading toward the hundreds-of-millions range. Business teams got anxious: should we shard? Should we adopt ShardingSphere?

The DAL team spent about six months on research and a small-scale pilot. The conclusion we delivered: no sharding middleware at this stage. Self-built read/write split plus Redis caching is enough.

This post lays out the research process and the decision logic. It's not a universal answer to "should you shard." It's how we weighed it at the time.

## First, understand the problem

The first step of research isn't looking at solutions. It's figuring out whether there's actually a problem.

Business teams said "the data is getting big." But "big" doesn't mean "problematic." How much data can a single MySQL table handle? The answer depends on a lot of factors:

- Whether the schema design is sound
- Whether indexes are well-designed
- Whether the query pattern is point-lookups or range scans, OLTP or OLAP
- Hardware configuration

We pulled the data at the time:

```sql
-- Row counts for the core tables
SELECT table_name, table_rows, ROUND(data_length/1024/1024, 2) AS data_mb,
       ROUND(index_length/1024/1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 't_xxx';
```

Result: the largest table had 120 million rows, 45 GB of data, 18 GB of indexes. B+ tree height of 3-4 levels. A point lookup via index sat under 10ms latency.

Honestly, MySQL handles this volume fine on a single table. 100 million rows isn't a big deal for InnoDB, as long as indexes and query patterns are reasonable.

> A lot of the time "the data is getting big" actually means "slow queries are getting many." The root cause isn't the data volume. It's bad indexes and bad SQL.

## The cost of sharding

But since we were researching, we had to think through "what happens if we do shard." The cost of sharding is far higher than most people imagine.

### 1. Transactions are gone

MySQL doesn't natively support cross-database transactions. You either use XA distributed transactions (poor performance, complex), or go with TCC / Saga application-layer compensation (heavy development).

Our core transaction pipeline depends heavily on transactions. If we sharded, a single order touching the orders DB, inventory DB, and accounts DB, originally handled with a single `BEGIN ... COMMIT`, would require a pile of compensation logic. Development and maintenance costs multiply several times over.

### 2. JOINs are gone

Cross-database JOINs are basically impossible. Data you used to get with one SQL statement now requires either multiple queries assembled at the application layer, or data redundancy.

Data redundancy means data-consistency maintenance overhead. When data in DB A is updated, how does the redundant copy in DB B sync? Now you're into binlog subscription, async syncing, eventual consistency. That whole stack.

### 3. Operations complexity skyrockets

- Backup and recovery: from one DB to coordinated multi-DB backups.
- DDL changes: one `ALTER TABLE` has to execute across all shards.
- Data migration: scaling out, scaling in, rebalancing. Each step is a major operation.
- Monitoring: you have to monitor the health of every shard.

These aren't things that just hiring a few more DBAs solves. The entire operations system has to be rebuilt.

### 4. ShardingSphere pitfalls

We did a small-scale pilot with ShardingSphere-JDBC. Powerful, yes. We hit a few pitfalls:

- Complex SQL parsing occasionally misbehaved. Nested subqueries beyond three levels sometimes produced incorrect routing.
- Operations tooling was immature. ShardingSphere-Scaling for data migration wasn't very stable at the time.
- High learning curve. Few people on the team could really run it; troubleshooting when things broke was painful.

## If we don't shard, how do we hold up?

Having decided not to shard, how do we hold up business growth? Our answer was a three-part toolkit: optimize existing queries, read/write split, and Redis caching.

### Tool 1: SQL and index optimization

Covered in the slow-query governance post. EXPLAIN + adding indexes eliminated a huge number of full-table-scan slow queries. This step alone halved database pressure.

A lot of business teams assume "if the DB is slow, we should shard." In reality, 80% of slowness is just bad SQL. Fix the SQL and a single table easily handles 200-300 million rows.

### Tool 2: read/write split

Writes to master, reads to slave, covered earlier. The bulk of read traffic is reports and list queries, which go to the slave. Master pressure drops by half immediately.

### Tool 3: Redis caching

Hot data goes in Redis. MySQL only gets hit on cache misses.

```python
# Classic cache-aside pattern
def get_order(order_id):
    # Check Redis first
    cached = redis.get(f"order:{order_id}")
    if cached:
        return json.loads(cached)
    # Cache miss, query MySQL
    order = db.query(Order).filter_by(id=order_id).first()
    if order:
        redis.setex(f"order:{order_id}", 300, json.dumps(order.to_dict()))
    return order
```

Lessons from our caching strategy:

- Add random jitter to expiry times to prevent cache avalanches.
- Protect hot keys individually with a local-cache fallback layer.
- For cache-database consistency, use binlog subscription plus active deletion; don't attempt complex in-place updates.

> Caching is trading memory for CPU and IO. Memory is more expensive than disk, but far cheaper than sharding.

## When should you actually shard?

We didn't say "never shard." We defined a clear trigger condition:

- Single-table data exceeds 500 million rows, and the three-part toolkit (SQL optimization + read/write split + caching) can no longer meet performance requirements.
- A clear vertical split boundary emerges: some business modules have fundamentally different access patterns from the rest and can be extracted into their own database.

Neither condition has been triggered yet. When they are, we'll re-evaluate.

## The decision logic

Putting the decision logic in a table:

| Option | Cost | Benefit | Risk |
|--------|------|---------|------|
| Sharding + middleware | High (dev + ops double) | Theoretical linear scaling | Lost transactions/JOINs, ops complexity |
| SQL optimization + read/write split + caching | Low (within existing system) | Sufficient, handles 200-300M rows | Cache consistency, slave lag |

> Don't reach for the advanced solution. If what you have is enough, don't add complexity.

There's an unwritten rule in software engineering: complexity is debt, so borrow as little as possible. Sharding borrows a large chunk of complexity debt. Don't borrow it unless you absolutely have to.

## Conclusion

The research report went up. The CTO signed off: no sharding for now. The DAL team would keep deepening read/write split and caching, and keep pushing slow-query governance.

In hindsight, the call was right. Over a year later, the business kept growing, but optimization and caching held the line, with no sharding-induced ops nightmare.

The biggest takeaway from my DAL research work: pick the solution whose cost and benefit best match your current stage, not the one that sounds more advanced.

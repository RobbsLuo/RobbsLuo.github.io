---
title: Two and a Half Years on DAL: Building a Database Governance System
date: 2024-04-15 10:00:00
tags:
  - Career
categories:
  - Career
description: Two and a half years on the DAL team — from change review to cross-language SDKs to slow-query governance. A retrospective on how the database governance system was built, step by step.
lang: en
---

In the spring of 2024, I was putting together a work summary for the DAL team. From November 2021 to that point, two and a half years, this team had built a database governance system from scratch. This post is a retrospective, not to show off what we did, but to lay out the thread of "how it was built", as a reference for those who come after and a bookmark for myself.

## Starting Point: A Tangled Mess

When I joined the DAL team at the end of 2021, the state of database access at the company could be summed up in four words: every service for itself.

- A dozen-plus Java services with wildly varying pool configs: some HikariCP, some ancient c3p0, some connecting raw.
- Python services were even more chaotic, some SQLAlchemy, some raw pymysql.
- No unified slow-query monitoring. Incidents were caught by user complaints and DBAs watching processlist.
- No DDL change-review pipeline. Anyone could change anything, whenever, nearly causing disasters several times.
- Multi-tenant isolation depended on business-code discipline, with the constant risk of cross-tenant data leakage.

> Governance isn't icing on the cake, it's firefighting. Not "nice to have" but "without it, things blow up."

## The Order of Building

Looking back over those two and a half years, the work follows a clear sequence. This order wasn't arbitrary, it was dictated by dependencies.

### Step 1: Change Review (the Foundation)

Timing: late 2021 to early 2022.

The reasoning was simple: as long as changes run uncontrolled, incidents won't stop. Without change review first, no matter how good the SDK is, a single `ALTER TABLE` can take production down.

The specifics are in the earlier post "The DAL Team: Reviewing Database Changes." Core idea: tickets + review meetings + execution follow-up.

> Stop the bleeding first, then treat the disease. That's the iron rule of building any system.

### Step 2: The Java SDK (Mainstream Consolidation)

Timing: first half of 2022.

Change review pinned down the "reckless changes" problem; next was the "reckless connections" problem.

Java came first because it was the primary language. HikariCP for pooling, self-built read/write split, automatic slow-query reporting. In six months, every Java service was on the dal-java SDK.

The value of this step was making infrastructure upgrades land across all services at once. Previously, changing a pool parameter meant begging a dozen teams. Now, one change to the SDK config center and everything takes effect.

### Step 3: The Python SDK (Covering the Long Tail)

Timing: second half of 2022.

Once Java was consolidated, the "raw connections" in Python services stood out. Fewer Python services, but the same needs: pooling, read/write split, slow-query monitoring.

Built independently on SQLAlchemy, with behavior aligned to the Java edition. The hard part wasn't writing the code, it was keeping the behavioral contract consistent across both ends: identical read/write rules, identical slow-query thresholds, identical config sources.

### Step 4: Slow-Query Governance (a Sustained Campaign)

Timing: throughout 2023, continuing ever since.

Once the SDK had collection and reporting in place, the remaining work was "continuous optimization." Every week, pull a few entries from the Top slow queries, EXPLAIN them, add indexes, rewrite SQL.

No tricks here, just persistence. After half a year, the count of P99 slow queries in production dropped by an order of magnitude.

### Step 5: Multi-Tenant Isolation Solidified (Plugging Holes)

Timing: mid-2023.

The multi-tenant isolation scheme had existed since the system's early days, but it relied on business-code discipline. The DAL team pushed it down into the SDK layer: automatic `tenant_id` injection, error if no context set. Upgrading "convention" into "enforcement."

### Step 6: Sharding Research (Defining the Boundary)

Timing: second half of 2023.

As business volume grew, "should we shard?" came up again and again. We spent six months on research and piloting. The conclusion: no middleware. Optimization plus read/write split plus caching is enough. The reasoning is in the earlier post on sharding.

## The Thread of the System

Chaining those six steps together gives the thread of this database governance system:

```
Change review (stop the bleeding)
    ↓
Java SDK (mainstream consolidation)
    ↓
Python SDK (cover the long tail)
    ↓
Read/write split + slow-query monitoring (built-in SDK capabilities)
    ↓
Slow-query governance (a sustained campaign)
    ↓
Multi-tenant isolation solidified (plug the holes)
    ↓
Sharding research (define the boundary)
```

> A system isn't built in a day. It's layered on, step by step. Each step paves the way for the next.

## What We Got Right

Looking back, a few decisions I think were right:

1. Change review before the SDK. The other way around and production would have been taken down by reckless DDL before the SDK was even done.

2. Java first, Python second. Consolidate the main language first, scout ahead, then do Python. Clear sequence, controllable risk.

3. Self-built read/write split, no middleware. Saved operational cost, avoided single-point risk, and the same logic worked for Python.

4. No sharding. The most questioned decision, but right in hindsight. Complexity debt: borrow as little as possible.

## What Could Have Been Better

There are regrets too:

1. Python started too late. Java was done, then we dragged for over half a year before starting Python. During that gap, Python services were still raw-connecting, essentially six extra months of risk. Ideally, Python should have started the moment Java stabilized.

2. Slow-query governance was intermittent. During a stretch when people were busy elsewhere, the Top-slow-query optimization paused for two weeks. Twenty-plus slow queries piled up, and it took a month to clear them. A sustained campaign is expensive to restart once it stops.

3. Alert thresholds weren't refined. Initially the slow-query threshold was uniformly 200ms; we later realized different businesses needed different thresholds. Refinement came too late.

## The Essence of a System

After these two and a half years, my understanding of the word "system" has changed.

I used to think a system was just a pile of tools and processes. Now I think the essence of a system is "making the right things easy and the wrong things hard."

- Change review: makes "reckless DDL" hard.
- SDK consolidation: makes "reckless connections" hard.
- Multi-tenant isolation pushed down: makes "forgetting tenant_id" hard.
- Slow-query monitoring: makes "ignoring performance problems" hard.

> A good system doesn't rely on people watching. It relies on mechanisms that block mistakes.

People get tired, forget, move on. Mechanisms don't. Team members rotate, but the system stays, and newcomers just follow it.

## In Closing

The DAL team's system isn't cutting-edge. No original architecture, it's all combinations of mature industry solutions. Its value lies in completeness and continuity: from changes to SDK to monitoring to governance, one unbroken thread.

> The value of a technical system isn't brilliance at any single point, it's coordination across the whole.

Two and a half years, from a tangled mess to a system that runs. What I learned in that process outweighs anything I've read in a book.

---
title: The DAL Team: Reviewing Database Changes
date: 2021-12-14 10:00:00
tags:
  - Career
categories:
  - Career
description: Database changes can never go naked. How the DAL team built a change-review pipeline and the traps we hit along the way.
lang: en
---

## Why Review at All

Databases. Anyone who has touched them knows. One bad `ALTER TABLE` on production and the whole business freezes. Getting paged at 3 a.m. to roll back is a rite of passage for most backend devs who have been around a few years.

At Supernova we had a DAL team (Database Access Layer), formally spun up at the end of 2021. When I joined, the very first thing we did was **get a database change-review pipeline running**.

> **A bug in code breaks one feature. A bad SQL change takes the whole database down with it.**

That's a line I kept repeating to the new folks. It sounds dramatic, but it's the truth.

## What the Pipeline Looks Like

We didn't start with anything fancy. Just a **ticket + review meeting** combo.

### 1. File a Ticket

Any DDL or DML against a production database had to go through a ticket. The ticket had to include:

- The actual SQL (the raw statement, not "I'll describe it verbally")
- An impact assessment: estimated rows affected, any full-table scans
- A rollback plan: how to save it if it blows up
- An execution window: usually a low-traffic night slot

```yaml
# Example ticket (sanitized)
ticket:
  db: t_xxx_orders
  type: DDL
  sql: "ALTER TABLE t_xxx_orders ADD COLUMN refund_status TINYINT DEFAULT 0"
  affected_rows: ~
  rollback: "ALTER TABLE t_xxx_orders DROP COLUMN refund_status"
  window: "2021-12-10 02:00-04:00"
```

### 2. DAL Team Pre-review

Once a ticket came in, someone on the DAL side took a first look. The main things we checked:

- Are the right indexes in place?
- Is the column type appropriate (`TINYINT` vs `INT` and so on)?
- Is there a `LIMIT` as a safety net to prevent wiping the whole table?

If pre-review didn't pass, we bounced it straight back, no need to bring it to the meeting.

### 3. The Review Meeting

Tickets that cleared pre-review went to a fixed weekly meeting where we walked through the batch accumulated that week. **The meeting was not a rubber stamp.** People argued. Someone would say the index was unnecessary, someone else would insist the `ALTER` had to go through pt-online-schema-change. It got hashed out.

> **The value of review is not the signature. It's making a second and third person actually look at your SQL.**

Very often the author can't see the problem in their own SQL, but a fresh pair of eyes immediately spots it: oh, missing an index here; oh, the `WHERE` clause on this `UPDATE` is doing a full table scan.

### 4. Execution and Follow-up

After approval, a DBA or someone on the DAL team executed the change. Then there was a follow-up check: confirm the change took effect, the business wasn't throwing errors, and nothing weird had suddenly appeared in the slow-query log.

## Traps We Hit

Once the pipeline was running, the biggest resistance wasn't technical. It was **people**.

**Business teams thought it was slow.** Before, if you wanted to add a column you just pinged the DBA and it went live. Now you had to file a ticket, wait for the review, queue up. There were complaints at first. But after two or three months, production incidents dropped noticeably, and everyone got on board.

**Review meetings drift into formalism.** If reviewers don't actually read the SQL, the meeting is theater. So our rule was: reviewers must be experienced backend devs or DBAs, and they must actually `EXPLAIN` the query, not just glance at it and say "approved."

Another trap: **big-table DDL.** Early on, someone filed an `ALTER TABLE` to add a column on a table with 80 million rows, and ran it directly. It blew up the binlog. After that we mandated: any table over 1 million rows, the DDL must go through pt-online-schema-change or gh-ost, never naked.

```bash
# Adding an index on a big table — use pt-osc, properly
pt-online-schema-change \
  --alter "ADD INDEX idx_user_id (user_id)" \
  D=t_xxx,t=orders \
  --execute
```

## The Outcome

By around mid-2022, production incidents caused by SQL changes had essentially dropped to zero. The one or two that slipped through were recovered quickly using the rollback plans filed in the tickets.

My personal take: **the hard part of change review isn't designing the process, it's the discipline of execution.** A pipeline looks great in a slide deck. What's hard is making every single change actually go through it, with no shortcuts.

> **A process isn't for when things are quiet. It's for when things blow up.**

The DAL team later went on to build cross-language SDKs, do slow-query governance, and work on multi-tenant isolation. But change review was always the foundation. If the foundation is shaky, nothing you build on top of it matters.

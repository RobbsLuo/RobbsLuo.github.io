---
title: Cross-Language SDK Part 1: Java with HikariCP
date: 2022-05-17 11:00:00
tags:
  - Career
categories:
  - Career
description: The DAL cross-language SDK started with the Java edition. We used HikariCP to build a solid connection-pool foundation, paving the way for behavior parity with the Python edition.
lang: en
---

Supernova's backend was primarily Java, so when it came to building a cross-language DAL SDK, the Java edition came first. In hindsight that was the right call: lock down the main language's foundation, then tackle Python. The rhythm was clean.

This post covers how the Java SDK was built. The next one covers how we independently built the Python edition, and how we kept behavior consistent across both.

## Why Build an SDK at All

The company's data access layer was a zoo. Some services connected via raw JDBC with no connection pool; some used MyBatis; legacy projects still ran on c3p0. Pool configs were all over the place, nobody monitored slow queries, and read/write split didn't exist.

After the DAL team was formed, we decided to unify: build one SDK and route all Java database access through it. Clear goals:

- Unified connection pool management (HikariCP)
- Built-in read/write split
- Automatic slow-query reporting
- Centralized configuration

> Unification is what lets infrastructure upgrades land across all services in one shot.

Without it, even changing a pool parameter meant begging a dozen teams to each make the change. Unmovable.

## Why HikariCP

Around 2022, there were really only two serious Java connection pool choices: HikariCP and Druid.

- **Druid**, open-sourced by Alibaba, packed with features — built-in SQL parsing, a monitoring dashboard, popular domestically.
- **HikariCP**, the Spring Boot default. High performance, lean code, active community.

We picked HikariCP. The reasoning was simple: we were going to build monitoring and read/write split ourselves, so we didn't need the pool to double as a Swiss army knife. HikariCP does one thing, pooling, and does it to the extreme, and stays out of the way. That made it easier to stack things on top.

Also, HikariCP's codebase is small (the core is a few thousand lines). When something goes wrong, you can actually read the source. Druid has several times the code; debugging drags you through all kinds of tangential logic.

## What the SDK Looks Like

Three layers:

```
Application
  ↓
DAL SDK (connection management + read/write split + slow-query reporting)
  ↓
HikariCP (pool)
  ↓
MySQL (master + replica)
```

### Pool Configuration

Each DataSource was wrapped in HikariCP, with config pulled centrally.

```yaml
# dal-sdk default config (sanitized)
dal:
  datasource:
    master:
      jdbc-url: jdbc:mysql://master-host:3306/t_xxx
      username: xxx
      password: xxx
      pool:
        maximum-pool-size: 20
        minimum-idle: 5
        connection-timeout: 3000
        idle-timeout: 600000
        max-lifetime: 1800000
    slave:
      jdbc-url: jdbc:mysql://slave-host:3306/t_xxx
      pool:
        maximum-pool-size: 20
```

A few parameters worth highlighting:

- `maximum-pool-size`: default 20. Don't start at 100. An oversized pool is more likely to overwhelm the database. 20 per service × 100 services = 2000 connections, and MySQL's default `max_connections` is just 151.
- `connection-timeout`: if a connection can't be obtained within 3 seconds, throw. Better to fail fast than to hang.
- `max-lifetime`: recycle connections every 30 minutes. MySQL's `wait_timeout` defaults to 8 hours, but if a firewall or NAT drops the connection in between, it becomes a dead socket. 30-minute recycling is safer.

### Read/Write Split

This was implemented in the SDK layer ourselves, not via a middleware like ShardingSphere. The reasoning gets its own post later; here's the implementation.

The approach isn't complicated: maintain a master and a slave HikariDataSource, route by SQL type.

```java
public class DalRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // Writes go to master
        if (DalContext.isWriteOperation()) {
            return "master";
        }
        // Reads go to slave
        return "slave";
    }
}
```

Read/write detection is simple: `INSERT/UPDATE/DELETE` to master, `SELECT` to slave. Queries inside a transaction are forced to master (to avoid reading stale data due to replication lag).

### Slow-Query Reporting

After each SQL execution, the SDK records the elapsed time. Anything above the threshold (default 200ms) gets pushed to the monitoring platform.

```java
@Around("execution(* javax.sql.DataSource.getConnection(..))")
public Object trackSql(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.nanoTime();
    try {
        return pjp.proceed();
    } finally {
        long costMs = (System.nanoTime() - start) / 1_000_000;
        if (costMs > SLOW_THRESHOLD_MS) {
            Metrics.report("dal.slow_query", costMs, currentSql());
        }
    }
}
```

This is illustrative. In practice we intercepted at the JDBC `PreparedStatement` execution layer, where we could capture the full SQL text and bound parameters.

## Paving the Way for the Python Edition

Once the Java SDK was done, we had a clear picture: what capabilities a DAL SDK should have. That capability list became the requirements doc for the Python edition:

- Unified connection pool
- Read/write split (writes to master, reads to slave, transactions to master)
- Slow-query reporting
- Config pulled from a central source

> The Java edition going first was scouting ahead for the Python edition.

What kind of mines did it scout? For example, we initially put the read/write routing decision in a ThreadLocal on the business thread. Later we realized Python doesn't have ThreadLocal — we'd need a different approach. More on that in the Python post.

## Rollout Rhythm

The Java SDK wasn't pushed in one shot. We piloted on a new service, ran it for two weeks without issues, then migrated legacy services one by one. Migration surfaced all kinds of weird legacy configs — one service had its pool set to 200. After migration we cut it to 20, and the business owner said "performance dropped." Investigation revealed that of those 200 connections, 150 had been sitting idle; real concurrency was around a dozen. Dialing it down actually reduced database pressure.

That kind of communication overhead is real, but it's worth it. Unification is painful at the start and glorious once you're past it.

The next post covers how we built the Python edition independently on SQLAlchemy, and how we kept behavior consistent across both ends.

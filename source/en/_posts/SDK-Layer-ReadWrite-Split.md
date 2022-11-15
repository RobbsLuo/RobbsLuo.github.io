---
title: Read/Write Split and Slow-Query Monitoring in the SDK
date: 2022-11-15 10:30:00
tags:
  - Career
categories:
  - Career
description: Read/write split and slow-query monitoring, built entirely in the SDK layer with no middleware. How we did it and why.
lang: en
---

The previous two posts covered building the Java and Python SDKs. This one pulls read/write split and slow-query monitoring out on their own, because these two are the core capabilities of the entire SDK, and we implemented both ourselves in the SDK layer, without touching any middleware.

## Why Not a Middleware

First, why not ShardingSphere, MyCat, ProxySQL, and their ilk.

Around 2022, read/write split solutions broadly fell into two camps:

- **Middleware layer**: ShardingSphere-JDBC / ShardingSphere-Proxy, MyCat, ProxySQL, MaxScale. SQL flows through the middleware first, which decides whether to route to master or slave.
- **SDK layer**: route inside the application yourself.

The appeal of middleware is "transparent to the application": no business code changes, plug in the middleware and you have read/write split. Sounds lovely. We didn't use it. Reasons:

1. Operational cost is too high.

ShardingSphere-Proxy and MyCat are independent processes. They need deployment, monitoring, and high availability. Our DBA team was stretched thin. Introducing another middleware cluster was just creating more work for ourselves.

2. The failure domain grows.

If the middleware goes down, every service routing through it goes down. We didn't want that single-point risk. Routing in the SDK layer means one service going down affects only that service, a much smaller blast radius.

3. SQL compatibility.

ShardingSphere and MyCat have limited support for complex SQL. Subqueries, stored procedures, certain JOIN patterns cause issues. Our business SQL was diverse; we didn't want to step on that minefield.

> Middleware fits the "I don't want to change anything" scenario. The SDK layer fits the "I want full control" scenario. We were the latter.

4. Python couldn't use it anyway.

ShardingSphere-JDBC is a Java library. Python services can't use it. If read/write split lived in the middleware, Python would need a separate solution. Doing it in the SDK layer lets Java and Python each have their own implementation, with behavior aligned.

## How Read/Write Split Works

### Java Edition

The core is Spring's `AbstractRoutingDataSource`. It's essentially a "data source router": every time you grab a connection, it asks you: master or slave?

```java
public class DalRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // Write operation or inside a transaction — go to master
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            return "master";
        }
        if (DalContext.isWriteOperation()) {
            return "master";
        }
        return "slave";
    }
}
```

How we detect "is this a write": intercept the SQL, check whether it starts with `INSERT/UPDATE/DELETE/REPLACE`. A simple regex:

```java
private static final Pattern WRITE_PATTERN =
    Pattern.compile("^\\s*(insert|update|delete|replace|create|alter|drop|truncate)", Pattern.CASE_INSENSITIVE);

public static boolean isWriteOperation(String sql) {
    return WRITE_PATTERN.matcher(sql).find();
}
```

A detail many people miss: queries inside a transaction must go to master. Why? Replication lag. You just wrote a row; reading it from the slave immediately might miss it. Queries inside a transaction typically depend on writes made earlier in that same transaction. Routing them to slave is a bug waiting to happen.

### Python Edition

The Python approach was covered in the previous post: maintain a master and a slave Engine, route at the Session level. The rule is identical to Java:

```python
class DalManager:
    def session(self, write=False):
        engine = self.master if write else self.slave
        return sessionmaker(bind=engine)()

    @contextmanager
    def transaction(self):
        # Inside a transaction, force master
        session = self.session(write=True)
        ...
```

> Identical rules across both ends matter more than identical-looking code.

### What About Replication Lag

You can't talk about read/write split without addressing slave lag.

We didn't build anything fancy like "auto-promote-to-master on lag detection." The approach was plain:

1. Strong-consistency scenarios go to master. Queries inside transactions, or reads that immediately follow a write, all go to master. The SDK decides this via the transaction context.
2. Scenarios that tolerate lag go to slave. Reports, list queries, historical data. A few hundred milliseconds of lag doesn't matter.
3. Monitor slave lag. Percona's `pt-heartbeat` running; alert above 1 second; auto-route reads back to master above 5 seconds.

This strategy was enough. Most read requests don't actually care about that lag.

## Slow-Query Monitoring

Slow-query monitoring has two parts: collection and reporting.

### Collection

Intercept the execution time of every SQL at the SDK layer.

The Java edition instruments through a JDBC `PreparedStatement` wrapper:

```java
public class DalPreparedStatement extends PreparedStatementWrapper {
    @Override
    public ResultSet executeQuery() throws SQLException {
        long start = System.nanoTime();
        try {
            return super.executeQuery();
        } finally {
            reportIfSlow("query", System.nanoTime() - start);
        }
    }

    private void reportIfSlow(String op, long elapsedNs) {
        long elapsedMs = elapsedNs / 1_000_000;
        if (elapsedMs > SLOW_THRESHOLD_MS) {
            SlowQueryReporter.report(sql, elapsedMs, op);
        }
    }
}
```

The Python edition uses SQLAlchemy event hooks; code was in the previous post, not repeating it here.

### Reporting

Slow-query data flows to two destinations:

1. Monitoring platform (Prometheus / internal metrics): aggregated trend views, P99 and P999 slow-query counts.
2. Log system (ELK): raw SQL plus call stacks, for troubleshooting.

```yaml
# Slow-query reporting config
dal:
  slow-query:
    threshold-ms: 200      # over 200ms counts as slow
    sample-rate: 1.0       # full sampling
    report:
      - type: metrics
        endpoint: prometheus-pushgateway:9091
      - type: log
        endpoint: elasticsearch:9200
```

The 200ms threshold was an initial gut call, later tuned with data. Different businesses have different tolerances. Report-style services can go up to 1 second; trading-style services tighten down to 100ms.

## Outcomes

After read/write split landed, read pressure on the master dropped by more than 60%. During peak hours, master CPU used to sit at 80%; after split it settled around 30%.

Slow-query monitoring had even bigger value. Before, we found out about slow queries when users complained. After monitoring, we caught them before they got worse. Every week we'd pull the Top 10 slow queries and optimize them one by one, which is the topic of the next post on slow-query governance.

> The point of monitoring is driving action, not pretty dashboards.

If monitoring only produces charts and nobody optimizes anything, it's wasted effort. Slow-query monitoring has to be paired with a "optimize Top N every week" routine to actually generate value.

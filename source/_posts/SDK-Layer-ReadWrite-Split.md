---
title: 读写分离与慢查询监控：在 SDK 层自己实现
date: 2022-11-15 10:30:00
tags:
  - Career
categories:
  - Career
description: 读写分离和慢查询监控没用任何中间件，全在 SDK 层自己实现。怎么做、为什么这么做。
---

前面两篇讲了 Java 版和 Python 版 SDK 的搭建。这一篇单独把读写分离和慢查询监控拎出来讲，因为这两个东西是整个 SDK 最核心的能力，而且都是我们自己在 SDK 层实现的，没碰任何中间件。

## 为什么不用中间件

先说为什么不选 ShardingSphere、MyCat、ProxySQL 这些。

2022 年那会儿，读写分离的方案大概分两类：

- 中间件层：ShardingSphere-JDBC / ShardingSphere-Proxy、MyCat、ProxySQL、MaxScale。SQL 先过中间件，中间件决定路由到主还是从。
- SDK 层：自己在应用里做路由。

中间件的优点是"对应用透明"，业务代码不用改，接上中间件就有读写分离。听起来很美，但我们最后没用。原因：

第一，运维成本太高。ShardingSphere-Proxy 和 MyCat 是独立进程，得部署、得监控、得高可用。我们当时 DBA 资源紧张，再引入一套中间件集群等于给自己加活。

第二，故障域变大。中间件一旦挂了，所有走它的服务全挂。这种单点风险我们不愿意背。SDK 层做路由，挂的只是一个服务，影响面小得多。

第三，SQL 兼容性。ShardingSphere 和 MyCat 对复杂 SQL 的支持有限，子查询、存储过程、某些 JOIN 场景会出问题。我们的业务 SQL 五花八门，不想踩这个坑。

我们是要完全可控，不是"啥都不想改"，所以选了 SDK 层。

第四，Python 那边没法用。

ShardingSphere-JDBC 是 Java 库，Python 服务用不了。如果读写分离做在中间件层，Python 服务还得另搞一套。SDK 层做的话，Java 和 Python 各自实现一份，行为对齐就行。

## 读写分离怎么做

### Java 版

核心是 Spring 的 `AbstractRoutingDataSource`。它本质上是个"数据源路由器"，每次拿连接的时候问你：你要主还是从？

```java
public class DalRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // 写操作或事务内，走主库
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

判断"是不是写操作"的方法：拦截 SQL，看开头是不是 `INSERT/UPDATE/DELETE/REPLACE`。我们用了一个简单的正则：

```java
private static final Pattern WRITE_PATTERN =
    Pattern.compile("^\\s*(insert|update|delete|replace|create|alter|drop|truncate)", Pattern.CASE_INSENSITIVE);

public static boolean isWriteOperation(String sql) {
    return WRITE_PATTERN.matcher(sql).find();
}
```

这里有个细节很多人会漏：事务里的查询必须走主库。为什么？因为主从有延迟。你刚写了一条数据，马上在从库查，可能查不到。事务内的查询通常依赖于事务内的写入，走从库就会出 bug。

### Python 版

Python 那边的做法在上一篇讲过：维护主从两个 Engine，在 Session 级别路由。规则和 Java 完全一致：

```python
class DalManager:
    def session(self, write=False):
        engine = self.master if write else self.slave
        return sessionmaker(bind=engine)()

    @contextmanager
    def transaction(self):
        # 事务内强制走主库
        session = self.session(write=True)
        ...
```

两端规则一致比代码长得一样重要。

### 主从延迟怎么办

读写分离绕不开这个问题：从库延迟。

我们没有搞什么"延迟检测自动切主"的花活。做法很朴素：

1. 强一致场景走主库。事务内的查询、刚写入马上要读的，统统走主库。SDK 层用事务上下文判断。
2. 业务能容忍延迟的走从库。报表、列表查询、历史数据这些，延迟几百毫秒无所谓。
3. 监控从库延迟。Percona 的 `pt-heartbeat` 跑着，延迟超过 1 秒报警，超过 5 秒自动把读流量切回主库。

这套策略够用了。大部分业务的读请求其实不在乎那点延迟。

## 慢查询监控

慢查询监控分两部分：采集和上报。

### 采集

在 SDK 层拦截每一条 SQL 的执行时间。

Java 版通过 JDBC 的 `PreparedStatement` 包装类埋点：

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

Python 版用 SQLAlchemy 的事件钩子，上一篇文章里有代码，这里不重复了。

### 上报

慢查询数据报到两个地方：

1. 监控平台（Prometheus / 内部 metrics 系统）：聚合看趋势，P99、P999 慢查询数量。
2. 日志系统（ELK）：存原始 SQL 和调用栈，排查用。

```yaml
# 慢查询上报配置
dal:
  slow-query:
    threshold-ms: 200      # 超过 200ms 算慢
    sample-rate: 1.0       # 全量采样
    report:
      - type: metrics
        endpoint: prometheus-pushgateway:9091
      - type: log
        endpoint: elasticsearch:9200
```

阈值 200ms 是拍脑袋定的，后来根据数据调整过。实际上不同业务的容忍度不一样，报表类服务可以放到 1 秒，交易类服务收紧到 100ms。

## 效果

读写分离上了之后，主库的读压力直接降了 60% 以上。原来高峰期 CPU 经常飙到 80%，分离后稳在 30% 左右。

慢查询监控的价值更大。以前慢查询是用户投诉了才知道，上了监控之后，还没变慢就能先发现。每周从监控里捞 Top 10 慢查询，逐一优化，这就是后面"慢查询治理"那篇文章要讲的事。

监控的目的不是为了看数据好看，而是驱动行动。如果监控只是看个图，不做优化，那等于白装。慢查询监控一定要配上"每周优化 Top N"的机制，才能真正产生价值。

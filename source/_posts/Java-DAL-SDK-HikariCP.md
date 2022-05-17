---
title: 跨语言 SDK（一）：Java 版用 HikariCP 把底子打好
date: 2022-05-17 11:00:00
tags:
  - Career
categories:
  - Career
description: DAL 跨语言 SDK 先从 Java 版做起，用 HikariCP 把连接池底子打牢，为后续 Python 版的行为一致铺路。
---

Supernova 的服务端以 Java 为主力，所以 DAL 跨语言 SDK 这件事，Java 版是第一个做的。这事后来看是对的，先把主力语言的底子打牢，再去啃 Python 那边，节奏清楚得多。

这一篇讲 Java SDK 怎么搭，下一篇讲 Python 版怎么独立搭起来，以及两端怎么做到行为一致。

## 为什么要自建 SDK

公司原来的数据访问层五花八门。有的服务直接拿 JDBC 连，连接池也不管；有的用 MyBatis；有的老项目还在用 c3p0。连接池配置各搞各的，慢查询没人监控，读写分离更是没有。

DAL 小组成立后，我们决定统一收口：做一套 SDK，所有 Java 服务都走它来访问数据库。目标很明确：

- 统一连接池管理（用 HikariCP）
- 内置读写分离
- 慢查询自动上报
- 配置统一管理

收口的目的是让基础设施的升级能一次推到所有服务。如果不收口，你改个连接池参数都得求着十几个团队各自改一遍，推不动。

## 为什么选 HikariCP

2022 年那会儿，Java 连接池其实就两个正经选择：HikariCP 和 Druid。

- Druid 是阿里开源的，功能多，自带 SQL 解析和监控页面，国内用的人多。
- HikariCP 是 Spring Boot 默认带的，性能好，代码精简，社区活跃。

我们最后选了 HikariCP。原因很简单：我们要自己做监控和读写分离，不需要连接池自带这些。HikariCP 就是把连接池这件事做到极致，其他不掺和，反而更适合我们往上叠东西。

而且 HikariCP 的代码量很小（据说核心就几千行），出了问题翻源码也翻得动。Druid 功能虽多，但代码量大了好几倍，排查问题的时候会被各种旁枝逻辑带跑。

## SDK 长什么样

整套 SDK 分三层：

```
应用层
  ↓
DAL SDK（连接管理 + 读写分离 + 慢查询上报）
  ↓
HikariCP（连接池）
  ↓
MySQL（主从）
```

### 连接池配置

我们给每个 DataSource 套了一层 HikariCP，配置统一从配置中心拉。

```yaml
# dal-sdk 默认配置（脱敏）
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

几个参数我重点说一下：

- `maximum-pool-size`：默认 20。别一上来就开 100，连接池开太大反而会让数据库被打满。一个服务 20 个连接，100 个服务就是 2000 个连接，MySQL 默认 `max_connections` 才 151。
- `connection-timeout`：拿不到连接时最多等 3 秒。超过 3 秒直接抛异常，比卡死强。
- `max-lifetime`：连接最多活 30 分钟。MySQL 的 `wait_timeout` 默认 8 小时，但中间如果有防火墙或 NAT 断连，连接就废了。30 分钟回收重建更保险。

### 读写分离

这一块是在 SDK 层自己实现的，没用 ShardingSphere 之类的中间件。原因后面那篇会专门讲，这里先说实现。

做法其实不复杂：维护一主一从两个 HikariDataSource，根据 SQL 类型分流。

```java
public class DalRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // 写操作走主库
        if (DalContext.isWriteOperation()) {
            return "master";
        }
        // 读操作走从库
        return "slave";
    }
}
```

判断读写的方法很简单：`INSERT/UPDATE/DELETE` 走主库，`SELECT` 走从库。事务内的查询强制走主库（避免主从延迟读到旧数据）。

### 慢查询上报

每个 SQL 执行完后，SDK 记录耗时。超过阈值（默认 200ms）的，打到监控平台。

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

这段是示意。实际上我们是在 JDBC 的 `PreparedStatement` 执行层做拦截，能拿到完整 SQL 文本和参数。

## 为 Python 版铺路

Java 版 SDK 做完之后，我们心里大概有了谱：一个 DAL SDK 该有哪些能力。这个"能力清单"后来直接成了 Python 版的需求文档：

- 统一连接池
- 读写分离（主写从读，事务走主）
- 滞查询上报
- 配置从中心拉取

Java 版先走通，本质上是给 Python 版趟了一遍雷。哪些雷呢？比如一开始我们把读写分离的判断逻辑放在了业务线程的 ThreadLocal 里，后来发现 Python 那边没有 ThreadLocal 这东西，得用别的方案。这些后面 Python 那篇再细讲。

## 上线节奏

Java SDK 不是一次性推的。我们先在一个新服务上试点，跑了两周没问题，再逐个迁移老服务。迁移过程中遇到不少奇葩配置，比如有的服务原来连接池开到 200，迁移后只给 20，业务方说"性能降了"。一查发现是原来 200 个连接里有 150 个是空跑的，真正并发的也就十几个。降下来之后数据库的压力反而小了。

这种沟通成本很高，但值得。收口这件事，一开始就是费劲，推过去之后爽得不行。

下一篇讲 Python 版怎么用 SQLAlchemy 独立搭起来，以及两端怎么保证行为一致。

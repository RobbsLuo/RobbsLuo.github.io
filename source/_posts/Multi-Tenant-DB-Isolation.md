---
title: SaaS 多租户：数据库隔离这套方案我们一直用
date: 2023-07-18 11:00:00
tags:
  - Career
categories:
  - Career
description: 多租户数据库隔离不是某个时间点新建的方案，而是从 DAL 小组成立之前就一直存在的做法，贯穿整个系统。
---

这篇文章我想讲一件容易被误解的事：SaaS 多租户的数据库隔离不是一个"项目"，不是某个时间点立项、设计、上线的东西，而是一套从系统诞生之初就存在的方案，贯穿至今。

DAL 小组 2021 年底成立的时候，这套隔离方案已经在跑了。我们做的事是把它在 SDK 层固化下来，让它更规范、更不容易出错，而不是从零设计。

## 多租户隔离的几种路子

先说背景。SaaS 多租户的数据隔离，业界通常三种模式：

1. 独立数据库：每个租户一个库。隔离最好，成本最高。
2. 共享数据库、独立 Schema：一个库里多个 Schema，每个租户一个 Schema。隔离中等，管理复杂。
3. 共享数据库、共享表：所有租户的数据在同一张表里，用 `tenant_id` 区分。成本最低，隔离靠应用层保证。

我们公司走的是第三种，共享表 + `tenant_id`。原因很实际：客户量大，独立数据库扛不住成本。一个库一个库地维护、备份、监控，光是 DBA 的工时就吃不消。

隔离方案不是"哪个最好"的问题，而是"哪个你养得起"的问题。

## 方案一直存在

我想强调的是：这套共享表 + tenant_id 的方案，在 DAL 小组成立之前就一直在用了。不是我们发明的，是整个系统从早期就奠定的架构选择。

具体怎么做：

- 每张业务表都有一个 `tenant_id` 字段。
- 所有查询必须带 `tenant_id` 条件。不带 `tenant_id` 的查询就是 bug。
- 应用层在每次请求的上下文里带上当前租户 ID，SDK 在生成 SQL 时自动追加 `tenant_id` 条件。

```sql
-- 应用层写的 SQL（业务代码）
SELECT * FROM t_xxx_orders WHERE user_id = 12345;

-- SDK 实际执行的 SQL（自动追加了 tenant_id）
SELECT * FROM t_xxx_orders
WHERE user_id = 12345 AND tenant_id = 'current_tenant';
```

这种"SDK 层强制注入 tenant_id"的做法，是 DAL 小组的贡献。在这之前，tenant_id 是靠业务代码自觉加的，而"自觉"这种东西，迟早会出漏子。

## Job 按客户隔离

这里有个关键细节：离线任务（Job）按客户访问隔离数据库。

什么意思？SaaS 系统除了在线请求，还有大量后台任务，报表生成、数据清洗、定时推送等等。这些任务如果混在一起跑，一个大客户的报表任务占满了数据库连接，小客户的在线请求就会被拖慢。

我们的做法是：Job 执行时，按客户维度隔离数据库访问。

```yaml
# Job 执行框架的租户隔离配置（脱敏）
job:
  isolation:
    mode: per-tenant
    pool:
      per-tenant-max-connections: 5   # 每个租户最多 5 个数据库连接
    schedule:
      stagger: true                    # 不同租户的 Job 错峰执行
```

效果：一个租户的 Job 不管怎么跑，最多占 5 个连接，不会把别的租户的在线请求挤死。

资源隔离是多租户系统活下去的根本。不隔离，一个大客户就能把所有人都拖下水。

这种按租户限流的思路，跟在线请求的限流是一脉相承的。在线请求按租户限流，离线 Job 也按租户限流，逻辑一致。

## SDK 层怎么固化

DAL 小组做的核心改进，是把这套隔离方案从"靠自觉"变成"靠框架"。

### Java SDK 层

在 SQL 执行前拦截，自动注入 `tenant_id`：

```java
public class TenantInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        String tenantId = TenantContext.get();
        if (tenantId == null) {
            throw new DalException("Tenant context not set");
        }
        // 拿到原始 SQL，追加 tenant_id 条件
        String sql = invocation.getSql();
        String rewritten = TenantSqlRewriter.rewrite(sql, tenantId);
        invocation.setSql(rewritten);
        return invocation.proceed();
    }
}
```

如果 TenantContext 没有设值，直接抛异常。这是强制的，宁可报错，也不允许"忘了带 tenant_id"的查询混过去。

### Python SDK 层

Python 版用 SQLAlchemy 的 Session 事件做同样的事：

```python
@event.listens_for(Session, "do_orm_execute")
def _add_tenant_filter(execute_state):
    tenant_id = TenantContext.get()
    if tenant_id is None:
        raise DalException("Tenant context not set")
    execute_state.statement = execute_state.statement.where(
        execute_state.statement.column_descriptions[0]["entity"].tenant_id == tenant_id
    )
```

两端的行为一致：没设 tenant 上下文就报错，设了就自动追加过滤。

## 踩过的坑

### 坑一：跨租户查询

有些后台管理功能需要查所有租户的数据（比如运营看全局数据）。这时候 `tenant_id` 过滤反而成了障碍。

我们的解决方法：给运营后台开一个"超级管理员"模式，可以绕过 tenant 过滤。但这个模式有严格的权限控制和审计日志，不是谁都能用。

```java
// 只有特定角色能跳过 tenant 过滤
if (CurrentUser.isSuperAdmin()) {
    // 不追加 tenant_id
} else {
    // 强制追加
}
```

### 坑二：JOIN 忘了带 tenant_id

```sql
-- 单表查询 SDK 能自动加 tenant_id
-- 但 JOIN 的时候，如果只给主表加了，关联表忘了加，就会泄露其他租户的数据
SELECT * FROM t_xxx_orders o
JOIN t_xxx_items i ON o.id = i.order_id
WHERE o.user_id = 12345;
```

SDK 的 SQL 重写器必须处理 JOIN 场景，给所有涉及的业务表都追加 `tenant_id` 条件。这一块的实现比单表复杂得多，我们踩了几次"数据泄露"的虚惊之后才把逻辑补全。

多租户系统的数据泄露，往往不是因为黑客攻击，而是因为 JOIN 忘了带条件。

## 这套方案的价值

这套方案的价值不在"技术多先进"，而在"一直被执行"。

很多公司的多租户隔离是写在文档里的，"请大家务必带 tenant_id"，然后某天某个新人忘了，数据就串了。我们把它下沉到 SDK 层之后，这种事基本绝迹了。你不带 tenant_id，代码直接跑不通，根本到不了生产。

好的架构不是靠人记住的，而是让犯错变得不可能。

这套隔离方案从系统早期一直用到现在，DAL 小组只是把它从"约定"升级成了"强制"。这个升级看似不性感，但它消除了多租户系统里最危险的一类风险，也就是人为失误导致的数据串号。

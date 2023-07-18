---
title: Multi-Tenant Database Isolation: A Scheme We've Used All Along
date: 2023-07-18 11:00:00
tags:
  - Career
categories:
  - Career
description: Multi-tenant database isolation isn't a "project" or something designed at a point in time — it's a scheme that's existed since the system's early days and runs throughout.
lang: en
---

This post is about something that's easy to misunderstand: SaaS multi-tenant database isolation is not a "project." It wasn't scoped, designed, and launched at some specific date. It's a scheme that's existed since the system was born, and it runs through everything.

When the DAL team was formed at the end of 2021, this isolation scheme was already running. What we did was solidify it at the SDK layer, making it more standardized and harder to get wrong, not design it from scratch.

## The three paths for multi-tenant isolation

Background first. Multi-tenant data isolation in SaaS generally falls into three patterns:

1. Database per tenant: each tenant gets its own database. Best isolation, highest cost.
2. Shared database, separate schemas: one database, one schema per tenant. Moderate isolation, complex management.
3. Shared database, shared tables: all tenants' data sits in the same tables, differentiated by `tenant_id`. Lowest cost, isolation guaranteed by the application layer.

We went with the third option: shared tables with `tenant_id`. The reason is pragmatic: a large customer count makes per-tenant databases financially untenable. Maintaining, backing up, and monitoring database after database would consume the DBA team's entire capacity.

> The right isolation scheme is the one you can afford to run, not the "best" one on paper.

## The scheme has always been there

What I want to stress: this shared-tables-plus-tenant_id scheme was already in use before the DAL team existed. We didn't invent it; it was an architectural choice laid down in the system's early days.

How it works concretely:

- Every business table has a `tenant_id` column.
- Every query must carry a `tenant_id` condition. A query without `tenant_id` is a bug.
- The application carries the current tenant ID in each request's context, and the SDK automatically appends the `tenant_id` condition when generating SQL.

```sql
-- SQL as written by the business code
SELECT * FROM t_xxx_orders WHERE user_id = 12345;

-- SQL actually executed by the SDK (tenant_id automatically appended)
SELECT * FROM t_xxx_orders
WHERE user_id = 12345 AND tenant_id = 'current_tenant';
```

This "SDK-layer forced tenant_id injection" is what the DAL team contributed. Before us, tenant_id was added by business code on a "be careful to remember" basis, and "be careful" always fails eventually.

## Jobs isolated per tenant

Here's a key detail: background jobs access the database isolated per customer.

What does that mean? A SaaS system has online requests plus a lot of background work, report generation, data cleansing, scheduled pushes, and so on. If those jobs run mixed together, one big customer's report task can hog all the database connections and slow down the online requests of smaller customers.

Our approach: when a job runs, database access is isolated per tenant.

```yaml
# Job framework's tenant isolation config (sanitized)
job:
  isolation:
    mode: per-tenant
    pool:
      per-tenant-max-connections: 5   # at most 5 DB connections per tenant
    schedule:
      stagger: true                    # jobs for different tenants run on staggered schedules
```

Effect: no matter how hard one tenant's job runs, it can occupy at most 5 connections, so it can't suffocate another tenant's online requests.

> Resource isolation is what keeps a multi-tenant system alive. Without it, one big customer can drag everyone down.

This per-tenant throttling logic is consistent with how online requests are throttled. Online requests are rate-limited per tenant; background jobs are rate-limited per tenant. The logic is the same.

## How we solidified it at the SDK layer

The core improvement the DAL team made was turning this scheme from "dependent on discipline" to "enforced by framework."

### Java SDK Layer

Intercept SQL before execution and inject `tenant_id` automatically:

```java
public class TenantInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        String tenantId = TenantContext.get();
        if (tenantId == null) {
            throw new DalException("Tenant context not set");
        }
        // Get original SQL, append tenant_id condition
        String sql = invocation.getSql();
        String rewritten = TenantSqlRewriter.rewrite(sql, tenantId);
        invocation.setSql(rewritten);
        return invocation.proceed();
    }
}
```

If TenantContext has no value, throw an exception. That's mandatory. Better to error than let a "forgot the tenant_id" query slip through.

### Python SDK Layer

The Python edition does the same thing via SQLAlchemy's Session events:

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

Behavior is consistent across both ends: no tenant context means an exception; with it, the filter is appended automatically.

## Traps we hit

### Trap 1: cross-tenant queries

Some back-office features need to query data across all tenants (operations looking at global data, for example). In that case, the `tenant_id` filter becomes an obstacle.

Our solution: open a "super admin" mode for the operations backend that can bypass tenant filtering. But this mode has strict access control and audit logging, so not just anyone can use it.

```java
// Only specific roles can skip tenant filtering
if (CurrentUser.isSuperAdmin()) {
    // Don't append tenant_id
} else {
    // Force the append
}
```

### Trap 2: JOIN forgetting tenant_id

```sql
-- Single-table queries: SDK can auto-append tenant_id
-- But on JOINs, if only the main table gets it and the joined table is forgotten, other tenants' data can leak
SELECT * FROM t_xxx_orders o
JOIN t_xxx_items i ON o.id = i.order_id
WHERE o.user_id = 12345;
```

The SDK's SQL rewriter has to handle JOIN scenarios and append the `tenant_id` condition to every business table involved. This was far more complex than the single-table case. We had a few near-misses with "data leakage" before we got the logic complete.

> Data leakage in multi-tenant systems usually isn't a hacker attack. It's a JOIN that forgot a condition.

## The value of this scheme

The value isn't in "how advanced the tech is." It's that it's been consistently enforced the whole time.

At many companies, multi-tenant isolation lives in a document ("please remember to include tenant_id"), and then one day a new hire forgets, and data crosses tenants. After we pushed it down to the SDK layer, this kind of incident basically vanished. If you don't include tenant_id, the code doesn't even run. It never reaches production.

> Good architecture doesn't rely on people remembering. It makes mistakes impossible.

This isolation scheme has been in use from the system's early days to today. The DAL team only upgraded it from "convention" to "enforcement." It's an unsexy upgrade, but it eliminates the most dangerous category of risk in a multi-tenant system: human error causing cross-tenant data leakage.

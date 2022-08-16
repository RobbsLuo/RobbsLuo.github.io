---
title: Cross-Language SDK Part 2: Python with SQLAlchemy
date: 2022-08-16 15:00:00
tags:
  - Career
categories:
  - Career
description: After the Java SDK was running, how did we build the Python edition independently on SQLAlchemy — keeping behavior consistent while respecting Python's own conventions?
lang: en
---

The previous post covered how the Java SDK set the foundation with HikariCP. This one is about the Python edition.

Bottom line up front: the Python edition was built independently, not as a "translation" of the Java one. Python has its own ecosystem and conventions; mechanically copying the Java approach would have backfired. But behavior across the two ends must be consistent. The same SQL must follow the same read/write routing logic, hit the same slow-query threshold, and use the same pool strategy, whether it's issued from Java or Python.

## Why Python Came Second

Not because it was lower priority, but because it was genuinely harder.

After the Java edition was done, we had a clear capability list: connection pool, read/write split, slow-query reporting, config center. Just transcribe it into Python, right?

No. The reasons:

- Java used Spring's `AbstractRoutingDataSource` for read/write split. Python had no equivalent.
- Java used ThreadLocal to pass context (whether the current operation is read or write). Python's concurrency model is fundamentally different.
- Java services run inside the JVM with a stable process model. Python had web services, cron jobs, and data scripts running inside Airflow — the process models were all over the place.

So the Python edition had to be designed from scratch. Not ported.

## Choosing SQLAlchemy

In the Python ecosystem, the main database access options were:

- **Raw DB-API (pymysql / mysqlclient)**: low-level, flexible, but you'd have to build pooling and ORM yourself. High maintenance cost.
- **SQLAlchemy**: the de facto Python ORM standard. Built-in pool, Session management, mature ecosystem.
- **Django ORM**: too tightly coupled to Django, and not all our services were Django.

We picked SQLAlchemy. The reasoning was the same as choosing HikariCP for Java: it does the things it should (pool, Session) and stays out of the way when you stack things on top.

> Framework selection follows the same logic: solid foundation, flexible on top. Java uses HikariCP; Python uses SQLAlchemy.

## Architecture

The Python SDK has three layers, mirroring the Java edition:

```
Application
  ↓
dal-python (Session management + read/write split + slow-query reporting)
  ↓
SQLAlchemy (pool)
  ↓
MySQL (master + replica)
```

### Connection Pool: QueuePool

SQLAlchemy ships with `QueuePool`. No third-party dependency needed.

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "mysql+pymysql://user:pass@slave-host:3306/t_xxx",
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=10,
    pool_timeout=3,      # wait 3 seconds for a connection
    pool_recycle=1800,   # recycle after 30 minutes
)
```

Parameters map one-to-one with the Java edition:

| Parameter | Java (HikariCP) | Python (SQLAlchemy) | Meaning |
|-----------|-----------------|---------------------|---------|
| Max connections | `maximum-pool-size=20` | `pool_size + max_overflow` | Same |
| Connection timeout | `connection-timeout=3000` | `pool_timeout=3` | Same (3 sec) |
| Connection recycle | `max-lifetime=1800000` | `pool_recycle=1800` | Same (30 min) |

This is the first layer of "behavior consistency": parameter semantics aligned.

### Read/Write Split: Session-Level Routing

Java did routing at the DataSource layer via `AbstractRoutingDataSource`. On the Python side I took a different approach: route at the Session level.

Why: Python's concurrency model differs from Java's. Java services are thread-based, one request, one thread, ThreadLocal for context is natural. Python leans heavily on coroutines (gevent, asyncio), where thread-local variables are unreliable.

My approach: maintain a master and a slave Engine, pick the Engine when creating the Session based on operation type.

```python
class DalManager:
    def __init__(self, master_url, slave_url):
        self.master = create_engine(master_url, poolclass=QueuePool, ...)
        self.slave = create_engine(slave_url, poolclass=QueuePool, ...)

    def session(self, write=False):
        """Pick engine based on operation type"""
        engine = self.master if write else self.slave
        return sessionmaker(bind=engine)()

    @contextmanager
    def transaction(self):
        """Transactions always go to master"""
        session = self.session(write=True)
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
```

Business code reads cleanly:

```python
# Read operation, goes to slave
with dal.session() as s:
    orders = s.query(Order).filter_by(user_id=uid).all()

# Write operation or transaction, goes to master
with dal.transaction() as s:
    order = Order(user_id=uid, amount=100)
    s.add(order)
```

This is the second layer of "behavior consistency": the read/write routing rule is identical across both ends. Java's rule is "writes to master, reads to slave, transactions to master." Python's is exactly the same.

### Slow-Query Reporting

SQLAlchemy exposes `before_cursor_execute` and `after_cursor_execute` event hooks, perfect for instrumentation.

```python
from sqlalchemy import event
import time

@event.listens_for(engine, "before_cursor_execute")
def _before(conn, cursor, statement, parameters, context, executemany):
    context._query_start = time.monotonic()

@event.listens_for(engine, "after_cursor_execute")
def _after(conn, cursor, statement, parameters, context, executemany):
    cost_ms = (time.monotonic() - context._query_start) * 1000
    if cost_ms > 200:  # same threshold as the Java edition
        metrics.report("dal.slow_query", cost_ms, statement)
```

## How We Guaranteed Cross-Language Consistency

This is the hardest part of a cross-language SDK. Both ends must process the same SQL with identical logic.

I made sure of four things. Key pool parameters take the same values and carry the same meaning across Java and Python (see the table above). The read/write routing rule is the same on both ends: writes to master, reads to slave, transactions to master, with no special-casing on either side. The slow-query threshold is 200ms on both, and reports land under the same monitoring metric and tag. Both ends pull pool config from the same config center, so nobody drifts.

> The real cost of a cross-language SDK is maintaining two behavioral contracts, not writing two codebases.

Case in point: once, the Java team changed `max-lifetime` from 30 minutes to 15, and forgot to sync Python. Java services recycled connections faster; Python services held on to stale ones. When MySQL did a connection reset, Python services threw a wave of connection errors while Java was unaffected.

After that incident, the rule went straight into the change-management process: any config change must land on both ends together.

## Rollout

The Python edition rolled out more smoothly than Java. Partly because Java had already proved the concept, so people trusted it. Partly because there were fewer Python services — smaller migration surface.

But there were traps. The biggest was async framework compatibility. Some services used asyncio, and a synchronous SQLAlchemy Engine blocks the event loop. We ended up providing a separate `AsyncEngine` + `AsyncSession` path for async services. The read/write split logic stayed the same; only the Engine implementation changed.

Next post: the concrete details of read/write split and slow-query monitoring, all built in the SDK layer, with no middleware dependency.

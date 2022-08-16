---
title: 跨语言 SDK（二）：Python 版用 SQLAlchemy 独立搭起来
date: 2022-08-16 15:00:00
tags:
  - Career
categories:
  - Career
description: Java 版 SDK 跑通后，Python 版怎么独立搭起来？用 SQLAlchemy，既要行为一致又要尊重 Python 的玩法。
---

上一篇讲了 Java 版 SDK 怎么用 HikariCP 把底子打好。这一篇讲 Python 版。

先说结论：Python 版是独立搭建的，不是 Java 版的"翻译"。Python 有自己的生态和习惯，硬搬 Java 那套会水土不服。但两端的行为必须一致，同样的 SQL，在 Java 和 Python 里走的是同样的读写分离逻辑、同样的慢查询阈值、同样的连接池策略。

## 为什么 Python 版后做

不是优先级低，是确实更难。

Java 版做完后，我们手里有了一份明确的能力清单：连接池、读写分离、慢查询上报、配置中心。照着抄一份 Python 版不就行了？

不行。原因是：

- Java 那套用 Spring 的 `AbstractRoutingDataSource` 做读写分离，Python 没有对应的机制。
- Java 用 ThreadLocal 传递上下文（当前是读还是写），Python 的线程模型完全不一样。
- Java 服务跑在 JVM 里，进程模型稳定；Python 那边有的是 Web 服务，有的是定时任务，还有跑在 Airflow 里的数据脚本，进程模型五花八门。

所以 Python 版必须从头设计，不能照搬。

## 选 SQLAlchemy

Python 生态里，数据库访问主要有几个选择：

- 原生 DB-API（pymysql / mysqlclient）：底层、灵活，但连接池、ORM 啥都得自己造。维护成本高。
- SQLAlchemy：Python ORM 的事实标准，自带连接池、Session 管理，生态成熟。
- Django ORM：和 Django 绑定太死，我们有的服务不是 Django。

最后选了 SQLAlchemy。原因和 Java 选 HikariCP 类似：它把该做的事做了（连接池、Session），但不限制你怎么往上叠东西。

选框架的逻辑两端是一致的，底层扎实，上层灵活。Java 用 HikariCP，Python 用 SQLAlchemy。

## 架构设计

Python SDK 分三层，跟 Java 版结构对齐：

```
应用层
  ↓
dal-python（Session 管理 + 读写分离 + 慢查询上报）
  ↓
SQLAlchemy（连接池）
  ↓
MySQL（主从）
```

### 连接池：QueuePool

SQLAlchemy 自带的 `QueuePool` 够用，不用再引入第三方。

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "mysql+pymysql://user:pass@slave-host:3306/t_xxx",
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=10,
    pool_timeout=3,      # 拿不到连接等 3 秒
    pool_recycle=1800,   # 30 分钟回收
)
```

参数和 Java 版一一对应：

| 参数 | Java (HikariCP) | Python (SQLAlchemy) | 含义 |
|------|-----------------|---------------------|------|
| 最大连接数 | `maximum-pool-size=20` | `pool_size + max_overflow` | 同 |
| 连接超时 | `connection-timeout=3000` | `pool_timeout=3` | 同（3 秒） |
| 连接回收 | `max-lifetime=1800000` | `pool_recycle=1800` | 同（30 分钟） |

这就是"行为一致"的第一层：参数语义对齐。

### 读写分离：Session 级别路由

Java 用 `AbstractRoutingDataSource` 在数据源层做路由。Python 这边我换了个思路：在 Session 级别做。

原因：Python 的并发模型跟 Java 不一样。Java 服务是线程模型，一个请求一个线程，ThreadLocal 传上下文很自然。Python 大量用协程（gevent、asyncio），线程本地变量不靠谱。

我的做法是维护主从两个 Engine，根据操作类型选 Engine 创建 Session：

```python
class DalManager:
    def __init__(self, master_url, slave_url):
        self.master = create_engine(master_url, poolclass=QueuePool, ...)
        self.slave = create_engine(slave_url, poolclass=QueuePool, ...)

    def session(self, write=False):
        """根据操作类型选 Engine"""
        engine = self.master if write else self.slave
        return sessionmaker(bind=engine)()

    @contextmanager
    def transaction(self):
        """事务强制走主库"""
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

业务代码用起来很清爽：

```python
# 读操作，走从库
with dal.session() as s:
    orders = s.query(Order).filter_by(user_id=uid).all()

# 写操作或事务，走主库
with dal.transaction() as s:
    order = Order(user_id=uid, amount=100)
    s.add(order)
```

这就是"行为一致"的第二层：读写分离的规则两端一致。Java 那边是"写走主、读走从、事务走主"，Python 这边一模一样。

### 慢查询上报

SQLAlchemy 有个 `before_cursor_execute` 和 `after_cursor_execute` 的事件钩子，正好用来埋点。

```python
from sqlalchemy import event
import time

@event.listens_for(engine, "before_cursor_execute")
def _before(conn, cursor, statement, parameters, context, executemany):
    context._query_start = time.monotonic()

@event.listens_for(engine, "after_cursor_execute")
def _after(conn, cursor, statement, parameters, context, executemany):
    cost_ms = (time.monotonic() - context._query_start) * 1000
    if cost_ms > 200:  # 和 Java 版同样的阈值
        metrics.report("dal.slow_query", cost_ms, statement)
```

## 两端怎么保证行为一致

这是跨语言 SDK 最难的部分。代码能跑只是起步，两端对同一个 SQL 的处理逻辑必须一样。

我做了几件事：

1. 参数语义对齐：连接池的关键参数，Java 和 Python 两端取同样的值，含义一致（见上面那张表）。
2. 读写分离规则一致：都是"写走主、读走从、事务走主"，不允许某一端有特殊行为。
3. 慢查询阈值一致：两端都是 200ms，上报到同一个监控指标的同一个 tag 下。
4. 配置来源统一：两端的连接池配置都从同一个配置中心拉，避免各改各的。

跨语言 SDK 的真正成本不是写两份代码，而是维护两份行为契约。

举个例子：有一次 Java 版调了连接池的 `max-lifetime` 从 30 分钟改成 15 分钟，Python 那边忘了同步。结果 Java 服务连接回收更快，Python 服务还留着老连接，碰上一次 MySQL 侧的连接重置，Python 那边报了一波连接错误，Java 没事。

从那以后，配置改动必须两端一起走，写进了变更流程。

## 上线

Python 版的推广比 Java 版顺一些。一是因为 Java 版已经趟过坑了，大家知道这套东西有用；二是 Python 服务相对少，迁移面小。

但也有坑。最大的一个是异步框架的兼容。有的服务用 asyncio，SQLAlchemy 的同步 Engine 在协程里会阻塞事件循环。后来我们对异步服务单独提供了 `AsyncEngine` + `AsyncSession` 的方案，读写分离逻辑还是同一套，只是 Engine 换了个异步实现。

下一篇讲读写分离和慢查询监控的具体实现细节，都在 SDK 层自己做，没依赖任何中间件。

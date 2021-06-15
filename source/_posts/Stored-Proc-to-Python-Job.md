---
title: 把抵押品计算从存储过程搬到 Python Job
date: 2021-06-15 10:00:00
tags:
  - Career
  - Python
  - Refactor
categories:
  - Career
description: 抵押品价值计算原来全塞在数据库存储过程里，跑一次能把库卡死。重构为 Python Job 后，计算从天级降到小时级，数据库也终于喘上气了。
---

## 这事儿的起因

2021 年中，我在 Collateral（抵押品管理）项目组接了个烫手活。

系统干的事说起来不复杂：维护贷款抵押品的价值和贷款数据，生成报表、做风险预警。但问题出在抵押品价值怎么算上：所有计算逻辑全压在数据库的存储过程里。

你说存储过程有什么不好？其实本身没什么不好。但在我们这个场景下，它就是那个把数据库拖垮的元凶。

## 存储过程为什么拖垮了数据库

先把锅扣明白。存储过程跑在数据库进程里，和数据库的查询、写入共用 CPU、内存、连接池。当计算量一上来，会发生什么？计算把数据库的资源吃光了，正常的业务读写全排队等着。

这就像你让厨师一边炒菜一边算账，算到一半客人点单都没人理。数据库卡死的直接后果：

- 业务接口超时，上游系统疯狂告警
- 报表跑一次要跑一整天，业务方第二天才看得到数据
- DBA 每天提心吊胆，生怕锁表把生产搞挂

把计算和存储混在一起，等于让数据库一个人干两个人的活，最后两个活都干不好。

我当时判断很简单：问题的根不在 SQL 写得好不好，而在架构分层错了。计算逻辑不该住在数据库里。

## 重构方案：Python Job 接棒

重构的核心思路就一句话：把计算逻辑从存储过程里搬出来，用独立的 Python Job 跑。

数据库只管存数据，计算交给应用层。这样资源天然隔离，数据库不再被计算任务绑架。

具体怎么做？我分了三步。

### 第一步：任务拆分

原来的存储过程是一个大 blob，几千行，里面什么都有：取数、计算、写回、出报表。我先按职责把它拆开：

```python
# 任务拆分后的模块结构（脱敏示意）
from collateral.jobs import (
    FetchLoanData,        # 拉取贷款数据
    FetchCollateralData,  # 拉取抵押品数据
    CalculateValue,       # 价值计算（核心，口径脱敏）
    WriteReport,          # 写回报表
    NotifyDownstream,     # 通知下游
)

# 每个 Step 是独立可测试的单元
class CollateralJobPipeline:
    steps = [
        FetchLoanData(),
        FetchCollateralData(),
        CalculateValue(),
        WriteReport(),
        NotifyDownstream(),
    ]

    def run(self, run_date):
        ctx = {"run_date": run_date}
        for step in self.steps:
            ctx = step.execute(ctx)
        return ctx
```

拆分的好处太明显了：每一步可以单独测试、单独重跑。哪一步出了问题，定位到那个 step 就行，不用在几千行存储过程里大海捞针。

### 第二步：调度

存储过程以前靠 crontab 或者数据库自带的 job scheduler 触发，时间一到就闷头跑。重构后我用 Airflow 来编排：

```yaml
# Airflow DAG 简化示意
collateral_daily_calc:
  schedule: "0 22 * * *"       # 每晚 22 点触发
  tasks:
    - fetch_loan_data
    - fetch_collateral_data
    - calculate_value:
        depends_on: [fetch_loan_data, fetch_collateral_data]
    - write_report:
        depends_on: [calculate_value]
    - notify_downstream:
        depends_on: [write_report]
```

Airflow 给我的不只是定时，还有依赖管理和可视化。哪个任务卡住了、跑了多久、重试了几次，DAG 页面上一目了然。这在以前是完全没有的：存储过程跑在数据库里，外面根本看不见状态。

### 第三步：资源隔离

这是最关键的一步。Python Job 跑在独立的计算节点上，和数据库物理隔离。计算过程中的中间结果写在本地或者对象存储里，只有最终结果才写回数据库。

```python
# 计算节点独立执行，不占数据库资源
class CalculateValue:
    def execute(self, ctx):
        loans = ctx["loans"]
        collaterals = ctx["collaterals"]

        # 在计算节点内存里跑，不碰数据库
        results = self._compute_in_memory(loans, collaterals)

        # 中间结果落本地 parquet，方便排查
        self._save_checkpoint(results, ctx["run_date"])
        ctx["results"] = results
        return ctx
```

这样一来，数据库只负责最终的写入，计算的压力完全转移到计算节点。想加机器加机器，想横向扩展横向扩展，不再受限于数据库那台机器的配置。

## 效果：天级降到小时级

重构上线后，效果是实打实的：

| 指标 | 重构前（存储过程） | 重构后（Python Job） |
|---|---|---|
| 单次计算耗时 | 天级（8h+） | 小时级（~2h） |
| 计算期间数据库负载 | 打满，业务接口超时 | 几乎无影响 |
| 故障定位 | 几千行 SP 里翻 | DAG 页面 + 日志，分钟级 |
| 可重跑 | 基本不行（状态丢了） | 支持按 step 重跑 |

计算从天级降到小时级，不是因为我把算法优化了多少倍，算法口径完全没变。变的是架构：把计算从数据库里解放出来。

这个项目让我想通一件事：很多时候性能问题的根不在"代码写得快不快"，而在东西放对了位置没有。存储过程不是不能用，但它不该承担重量级的计算职责。让数据库专注存储，让应用层专注计算，各司其职，这才是正道。

后来这套 Python Job 的模式在团队里推开了，其他几个还在用存储过程跑批的模块也陆续迁了过来。我觉得这比单纯解决一个性能 bug 有价值得多，它改变的是团队对架构分层的认知。

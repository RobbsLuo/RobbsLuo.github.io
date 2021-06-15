---
title: Moving Collateral Calculation Out of Stored Procedures
date: 2021-06-15 10:00:00
tags:
  - Career
  - Python
  - Refactor
categories:
  - Career
lang: en
description: The collateral valuation calculation used to live entirely in database stored procedures — and running it once could lock up the whole database. After refactoring to a standalone Python Job, compute time dropped from days to hours, and the database could finally breathe.
---

## How This Started

Mid-2021, I picked up a thorny assignment on the Collateral Management team.

The system's job sounds simple enough: maintain collateral values and loan data, generate reports, and fire off risk alerts. The real problem was **how the collateral valuation got computed**: every bit of the calculation logic lived inside database stored procedures.

Is there something inherently wrong with stored procedures? No. But in our scenario, they were the **single biggest reason the database kept falling over**.

## Why Stored Procedures Killed the Database

A stored procedure runs inside the database process. It shares CPU, memory, and connection pools with every regular query and write on that same database. When the calculation load spikes, here's what happens:

**The computation eats all the database resources, and normal business reads and writes queue up behind it.**

It's like asking a chef to do the accounting while cooking. Halfway through the spreadsheet, a customer walks in and nobody takes their order. The direct consequences of a database locked up by computation:

- Business APIs start timing out, and upstream systems fire off alerts
- Reports take an entire day to run — stakeholders don't see numbers until the next morning
- The DBA lives in constant fear that a table lock will take down production

> **Mixing computation and storage in the same process means the database ends up doing two jobs badly instead of one job well.**

My diagnosis was straightforward: **the architectural layering was wrong**. Heavy computation has no business living inside the database.

## The Refactor: Python Job Takes Over

The core idea was a single sentence: **move the calculation logic out of stored procedures into a standalone Python Job**.

The database stores data. The application layer computes. Resources are naturally isolated, and the database is no longer held hostage by batch jobs.

Here's how I broke it down.

### Step One: Task Decomposition

The old stored procedure was one giant blob. Thousands of lines covering fetching data, computing, writing back, generating reports. I split it by responsibility:

```python
# Pipeline structure after decomposition (sanitized)
from collateral.jobs import (
    FetchLoanData,        # pull loan data
    FetchCollateralData,  # pull collateral data
    CalculateValue,       # valuation calc (core, logic sanitized)
    WriteReport,          # persist report
    NotifyDownstream,     # notify downstream consumers
)

# Each Step is an independently testable unit
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

The payoff is obvious: every step can be **tested and re-run independently**. When something breaks, you pinpoint the failing step instead of trawling through thousands of lines of stored procedure code.

### Step Two: Scheduling

The stored procedure used to be triggered by crontab or the database's built-in job scheduler. Fire and forget. After the refactor, I brought in Airflow for orchestration:

```yaml
# Simplified Airflow DAG
collateral_daily_calc:
  schedule: "0 22 * * *"       # every night at 22:00
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

Airflow gave me more than just cron. It gave me **dependency management** and **visibility**. Which task is stuck, how long it's been running, how many times it retried, all right there on the DAG page. We had none of that before. A stored procedure running inside the database is essentially a black box from the outside.

### Step Three: Resource Isolation

This was the most critical piece. The Python Job runs on a dedicated compute node, physically separate from the database. Intermediate results are written to local disk or object storage. Only the final output gets written back to the database.

```python
# Runs on the compute node — zero database load
class CalculateValue:
    def execute(self, ctx):
        loans = ctx["loans"]
        collaterals = ctx["collaterals"]

        # Compute in memory on the worker, don't touch the DB
        results = self._compute_in_memory(loans, collaterals)

        # Checkpoint to local parquet for debugging
        self._save_checkpoint(results, ctx["run_date"])
        ctx["results"] = results
        return ctx
```

Now the **database only handles the final write. All the compute pressure shifts to the worker.** Need more horsepower? Add machines. Need to scale horizontally? Go ahead. You're no longer constrained by the specs of the database server.

## Results: Days to Hours

After the refactor shipped, the numbers spoke for themselves:

| Metric | Before (Stored Proc) | After (Python Job) |
|---|---|---|
| Single run time | Day-level (8h+) | Hour-level (~2h) |
| DB load during computation | Maxed out, API timeouts | Negligible impact |
| Failure diagnosis | Sifting through SP code | DAG view + logs, minutes |
| Re-run capability | Essentially impossible (state lost) | Per-step re-run supported |

> **Compute time dropped from days to hours, not because I optimized the algorithm. The calculation logic is identical. What changed is the architecture: the computation was freed from the database.**

This project clarified something for me: more often than not, **the root of a performance problem is whether things are placed in the right position**, not how fast the code runs. Stored procedures aren't inherently bad. They just shouldn't carry heavy computation. Let the database store. Let the application layer compute. Each does what it's good at.

That Python Job pattern later spread across the team. Other modules still running batch logic in stored procedures migrated over one by one. I think that's far more valuable than fixing a single performance bug. **It shifted the team's mental model of architectural layering.**

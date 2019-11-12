---
title: Cleaning and Normalizing Metrics with AirFlow
date: 2019-11-12 15:00:00
tags:
  - Career
  - AirFlow
  - Data Cleaning
categories:
  - Career
description: Data in MySQL isn't ready to use yet. This post covers how we organized the AirFlow DAGs to run the full pipeline — cleaning, denoising, and normalization.
lang: en
---

Once data is in MySQL, you can't just start computing metrics on it. **Raw data is dirtier than you'd think**: lockfiles changing by thousands of lines, auto-generated code, duplicate diffs from merge commits, "+1" reviews, and the inevitable outlier who pushes 80 times in a day. Skip the cleanup and the profile output will publicly embarrass you.

This post is about how we used AirFlow to run cleaning, denoising, and normalization as one coherent pipeline.

## Why cleaning is non-negotiable

A few real "dirty data" scenarios, verbatim:

- Someone racks up 10,000+ added_lines in a week. Turns out it's `yarn.lock` being committed 12 times.
- A repo's `proto/generated/*.go` changes by thousands of lines on every build, all auto-generated.
- Someone leaves 200+ review comments, every single one is "👍", "+1", or "LGTM".
- Someone pushes 80 commits in a single day because a script batch-formatted import order.

Leave these alone and every metric we defined in the first post goes sideways. **Cleaning is about removing anything that would mislead a decision.**

## How the DAG is organized

Our DAG design follows one principle: **collection, cleaning, normalization, aggregation. Four layers, chained by dependencies, parallel tasks within a layer.**

```
┌─────────────┐   ┌─────────────┐
│ Collection   │ (external trigger, outside this DAG)
└──────┬──────┘
       │
       v
┌──────────────────────────────────────┐
│ Layer 1: Prepare data (time window)  │
└──────┬───────────────────────────────┘
       │
       v
┌──────────────────────────────────────────────┐
│ Layer 2: Denoise (parallel cleaning tasks)   │
│  ├─ Filter ignored paths                     │
│  ├─ Filter auto-generated commits            │
│  ├─ Strip noisy review comments              │
│  └─ Clip outliers                            │
└──────┬───────────────────────────────────────┘
       │
       v
┌──────────────────────────────────────┐
│ Layer 3: Normalize (language / role)  │
└──────┬───────────────────────────────┘
       │
       v
┌──────────────────────────────────────┐
│ Layer 4: Aggregate into profile table │
└──────────────────────────────────────┘
```

In AirFlow code it looks roughly like this:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime, timedelta

default_args = {
    'owner': 'coreteam',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    dag_id='metrics_compute_daily',
    default_args=default_args,
    schedule_interval='0 2 * * *',  # 2 AM daily
    start_date=datetime(2019, 6, 1),
    catchup=False,
) as dag:

    prepare = PythonOperator(
        task_id='prepare_window',
        python_callable=load_time_window,
        op_kwargs={'days': 7},
    )

    with TaskGroup('clean') as clean_group:
        clean_paths = PythonOperator(
            task_id='clean_ignore_paths',
            python_callable=filter_ignore_paths,
        )
        clean_generated = PythonOperator(
            task_id='clean_generated_commits',
            python_callable=filter_auto_generated,
        )
        clean_reviews = PythonOperator(
            task_id='clean_review_comments',
            python_callable=strip_noise_reviews,
        )
        clip_outliers = PythonOperator(
            task_id='clip_outliers',
            python_callable=clip_metric_outliers,
        )
        # Parallel within the layer
        [clean_paths, clean_generated, clean_reviews, clip_outliers]

    normalize = PythonOperator(
        task_id='normalize_metrics',
        python_callable=apply_normalization,
    )

    aggregate = PythonOperator(
        task_id='aggregate_profile',
        python_callable=write_to_profile_table,
    )

    prepare >> clean_group >> normalize >> aggregate
```

`TaskGroup` is an AirFlow 2.x nicety. It collapses a group of tasks in the graph view, which keeps things readable. We were on 1.10 at the time and had to fake it with subDAGs, which was painful enough to be one of the reasons we eventually upgraded.

## Denoising: one function per category of noise

The denoise layer is the part most worth expanding on. I'll walk through each of the four cleaning tasks.

### 1. Ignored-path filter

The collection layer already filters these, but config gets added retroactively and old data lingers. So the cleaning layer scans again:

```python
IGNORE_PATTERNS = [
    '**/node_modules/**', '**/vendor/**', '**/dist/**',
    '**/*.lock', '**/package-lock.json', '**/yarn.lock',
    '**/generated/**', '**/*.pb.go', '**/*.gen.ts',
]

def filter_ignore_paths():
    # Mark matching rows in git_commit_file with is_noise=1
    sql = """
        UPDATE git_commit_file
        SET is_noise = 1, noise_reason = 'ignore_path'
        WHERE is_noise = 0
          AND (file_path LIKE CONCAT('%/', %s, '/%') OR ...)
    """
    # Real implementation joins against a rules table; simplified here
```

Note: **we mark, not delete.** Raw data is never deleted. That's the baseline. If rules change, we can recompute from source.

### 2. Auto-generated commit detection

This one's trickier. We use a few heuristics:

- Commit message keyword match: `generate`, `auto`, `swagger`, `protobuf`, `migration`.
- Single commit touches more than a threshold number of files (say 50), and most of them share a common prefix (classic batch generation signature).
- Author email is in the CI/CD bot list.

```python
BOT_AUTHORS = {
    'ci-bot@coreteam', 'dependabot[bot]@users.noreply.github.com',
    'renovate[bot]@users.noreply.github.com',
}

def filter_auto_generated():
    # Mark entire commits from bots as noise
    mark_bot_commits(BOT_AUTHORS)
    # Mark commits whose messages match keywords
    mark_commits_by_message_keyword(['generate', 'auto-gen', 'protobuf'])
```

### 3. Review comment denoising

Regex to the rescue:

```python
NOISE_PATTERNS = [
    r'^\s*\+1\s*$', r'^\s*lgtm\s*$', r'^\s*👍\s*$',
    r'^\s*(nice|cool|good)\s*$', r'^:\w+:\s*$',  # GitHub short-code emoji
]

def strip_noise_reviews():
    update_sql = """
        UPDATE review_comment
        SET is_noise = 1
        WHERE is_noise = 0
          AND REGEXP_LIKE(body, %s)
    """
    # Combine patterns with OR
```

One nuance: **we don't delete noisy comments, we just exclude them from `effective_comment_count`.** The original reviews stay, so we can still see them when analyzing collaboration patterns later.

### 4. Outlier clipping

Even after the first three steps, you get outliers. Someone really did push 80 times today. We don't just delete those; we clip statistically.

The naive approach is the **3-sigma rule**: for each person, compute the mean and stddev of their metrics over the past N weeks, and flag any day that exceeds μ + 3σ.

```python
def clip_metric_outliers(metric='daily_commits', window_days=90):
    sql = f"""
        WITH stats AS (
            SELECT author_email,
                   AVG({metric}) AS mu,
                   STDDEV({metric}) AS sigma
            FROM daily_metric
            WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL %s DAY)
            GROUP BY author_email
        )
        UPDATE daily_metric d
        JOIN stats s USING (author_email)
        SET d.is_outlier = 1
        WHERE d.{metric} > s.mu + 3 * s.sigma
    """
    run(sql, (window_days,))
```

We ended up **computing distributions per person**, not for the whole company. Reason is simple: people work at different rhythms. A frontend engineer and an SRE aren't on the same commit-frequency scale. A global mean would false-positive normal work.

## Normalization: put different contexts on the same ruler

After denoising, there's still one issue you can't dodge: **code volume across languages isn't directly comparable.**

A Hello World in Java is dozens of lines. In Python it's three. Without normalization, the Java dev always looks "more productive."

We maintain a per-language weight table (desensitized):

| Language | Weight |
|---|---|
| Java | 0.6 |
| Go | 0.8 |
| Python | 1.0 |
| JavaScript / TypeScript | 0.9 |
| PHP | 0.8 |
| SQL | 1.2 |

There's no absolutely "correct" weight, **but having one beats not having one**. Normalization is just a multiplication:

```python
def apply_normalization():
    sql = """
        UPDATE git_commit_file f
        JOIN dim_language_weight w ON f.file_ext = w.file_ext
        SET f.weighted_added = f.added_lines * w.weight,
            f.weighted_removed = f.removed_lines * w.weight
        WHERE f.is_noise = 0
    """
    run(sql)
```

Beyond language weights, there's a **role baseline**: the frontend team mean, the backend team mean, the SRE team mean, each computed separately. The dashboard shows "your position within your team," not a raw absolute. More on that in the next post when we cover the Vue dashboard.

## One closing thought

For data cleaning, my feeling is: **the hard part is deciding what counts as noise.**

Every cleaning rule is, at heart, a judgment call: "this kind of behavior doesn't count as contribution." Make the call too loose, and people game the metrics. Make it too tight, and you kill legitimate work. Our approach was to **err on the loose side, but expose the judgment itself**: every `is_noise` / `is_outlier` flag carries a `noise_reason`, so the dashboard can show "this person's metrics are low because 30% of their work was flagged as noise," rather than a black-box number.

Collection and cleaning both run now. The last post covers how Vue 2 turns this data into pictures.

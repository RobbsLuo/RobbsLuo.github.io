---
title: 指标计算：用 AirFlow 把清洗和归一跑顺
date: 2019-11-12 15:00:00
tags:
  - Career
  - AirFlow
  - 数据清洗
categories:
  - Career
description: 数据进了 MySQL 不等于能直接用。这一篇讲我们怎么用 AirFlow 把 DAG 组织起来，跑通清洗、去噪、归一这一整套流程。
---

数据进了 MySQL，不代表就能直接算了。原始数据脏得超乎想象：lockfile 几千行一改、自动生成的代码、合并提交的重复 diff、"+1" 评审、还有那种一天提交 80 次的异常值，这些不处理掉，画像系统出来的结果会让人当场社死。

这一篇就讲我们怎么用 AirFlow 把清洗、去噪、归一这条链路跑顺。

## 为什么必须清洗

我直接列几个真实的"脏数据"场景：

- 某同学一周内 `added_lines` 上万，细看是 `yarn.lock` 提交了 12 次
- 某仓库的 `proto/generated/*.go` 一改就是几千行，全是自动生成的
- 某同学 PR 评审意见 200 多条，打开一看全是 "👍"、"+1"、"LGTM"
- 某同学某天突然提交了 80 次，因为脚本批量修改了 import 顺序

这些不处理，前面定义的所有指标都会失真。清洗的目的就是把会误导决策的噪音去掉。

## AirFlow DAG 怎么组织

我们的整体 DAG 设计遵循一个原则：采集、清洗、归一、汇总，分四层，层与层之间靠依赖串起来，层内可以并行。

```
┌─────────────┐   ┌─────────────┐
│ 采集完成事件 │ (外部触发，不在本 DAG)
└──────┬──────┘
       │
       v
┌──────────────────────────────────────┐
│ Layer 1: 数据准备 (拉取时间窗)         │
└──────┬───────────────────────────────┘
       │
       v
┌──────────────────────────────────────────────┐
│ Layer 2: 去噪 (并行多个清洗任务)              │
│  ├─ 过滤忽略路径                              │
│  ├─ 过滤自动生成提交                          │
│  ├─ 评审意见去噪                              │
│  └─ 异常值裁剪                                │
└──────┬───────────────────────────────────────┘
       │
       v
┌──────────────────────────────────────┐
│ Layer 3: 归一化 (按语言 / 角色系数)    │
└──────┬───────────────────────────────┘
       │
       v
┌──────────────────────────────────────┐
│ Layer 4: 汇总到画像层 (写最终指标表)   │
└──────────────────────────────────────┘
```

落到 AirFlow 代码里大概长这样：

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
    schedule_interval='0 2 * * *',  # 每天凌晨 2 点
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
        # 同层内并行
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

`TaskGroup` 是 AirFlow 2.x 之后才有的好东西，能把一组任务在图上折起来，看起来清爽很多。我们用的是 AirFlow 1.10，当时只能靠 subDAG 模拟，体验差不少，这也是后来升级的一个动力。

## 去噪逻辑：每一类噪音一个清洗函数

去噪这一层是最值得展开的，我把四个清洗任务的逻辑分别说一下。

### 1. 忽略路径过滤

采集层虽然已经过滤了一遍，但有时候配置是后加的，老数据还在。所以清洗层会再扫一遍：

```python
IGNORE_PATTERNS = [
    '**/node_modules/**', '**/vendor/**', '**/dist/**',
    '**/*.lock', '**/package-lock.json', '**/yarn.lock',
    '**/generated/**', '**/*.pb.go', '**/*.gen.ts',
]

def filter_ignore_paths():
    # 用 LIKE 配合路径前缀，把命中的 git_commit_file 行打上 is_noise=1
    sql = """
        UPDATE git_commit_file
        SET is_noise = 1, noise_reason = 'ignore_path'
        WHERE is_noise = 0
          AND (file_path LIKE CONCAT('%/', %s, '/%') OR ...)
    """
    # 实际是用一份规则表 join，这里简化
```

注意，这一步是打标记而不是删除。原始数据永远不删，这是底线，万一规则改了，还能回溯重算。

### 2. 自动生成提交识别

这个稍微 tricky 一点。我们用了几种启发式：

- commit message 匹配关键词：`generate`、`auto`、`swagger`、`protobuf`、`migration`
- 单次 commit 影响文件数 > 阈值（比如 50）且改动行数集中在前缀相同的文件（典型的批量生成）
- 作者邮箱匹配 CI/CD bot 列表

```python
BOT_AUTHORS = {
    'ci-bot@coreteam', 'dependabot[bot]@users.noreply.github.com',
    'renovate[bot]@users.noreply.github.com',
}

def filter_auto_generated():
    # bot 提交直接整条标为噪音
    mark_bot_commits(BOT_AUTHORS)
    # message 命中关键词的提交单独打标
    mark_commits_by_message_keyword(['generate', 'auto-gen', 'protobuf'])
```

### 3. 评审意见去噪

这个直接上正则：

```python
NOISE_PATTERNS = [
    r'^\s*\+1\s*$', r'^\s*lgtm\s*$', r'^\s*👍\s*$',
    r'^\s*(nice|cool|good)\s*$', r'^:\w+:\s*$',  # GitHub 短代码表情
]

def strip_noise_reviews():
    # 把命中噪音模式的评审意见从 "有效意见数" 中扣除
    update_sql = """
        UPDATE review_comment
        SET is_noise = 1
        WHERE is_noise = 0
          AND REGEXP_LIKE(body, %s)
    """
    # 用 OR 把所有 pattern 拼起来
```

这里有个细节：噪音意见不会真的删掉，只是在算 `effective_comment_count` 的时候排除掉。原始评审数据保留，方便后面做协作模式分析时还能看到。

### 4. 异常值裁剪

即便前面三步都做了，还是会有异常值，比如某同学某天真的就提交了 80 次。这种我们不能简单删，得用统计方法裁剪。

最朴素的做法是 3σ 原则：算每个人过去 N 周指标的均值和标准差，超过 μ+3σ 的那天单独标出来。

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

我们最后用的是每个人独立算分布，而不是全公司一刀切。理由很简单：不同人的工作节奏差异很大，前端同学和 SRE 同学的提交频率不在一个量级，强行用全公司均值裁剪会把正常值误杀。

## 归一化：把不同语境拉到同一标尺

去噪之后，还有个绕不开的问题：不同语言的代码量根本不可比。

写 Java 一个 Hello World 几十行，写 Python 三行结束。要是不归一，Java 选手的"代码量"天然占优。

我们维护了一份语言系数表（脱敏）：

| 语言 | 系数 |
|---|---|
| Java | 0.6 |
| Go | 0.8 |
| Python | 1.0 |
| JavaScript / TypeScript | 0.9 |
| PHP | 0.8 |
| SQL | 1.2 |

系数本身没有绝对正确的值，但有它就比没有强。归一化时直接乘上去：

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

除了语言系数，还有一个维度是角色基准：前端组的均值、后端组的均值、SRE 组的均值各算各的，画像看板上看的是"相对自己组内的位置"而不是绝对值。这一块下一篇讲 Vue 看板的时候会再提。

## 一点感受

我感觉数据清洗这件事，真正的难点是判断什么算噪音，而不是写代码。

每一条清洗规则，本质上都是一个判断："这种行为不算贡献"。这个判断下得太松，画像会被刷分；下得太紧，会把正常工作误杀。我们的做法是宁可松一点，也要把判断本身暴露出来：所有 `is_noise` / `is_outlier` 的标记都带 `noise_reason` 字段，看板上能直接看到"这个人指标低，是因为 30% 被标成噪音了"，而不是一个黑盒数字。

采集和清洗都跑通了，最后一篇讲 Vue 2 怎么把这些数据画出来。

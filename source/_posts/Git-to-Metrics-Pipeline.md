---
title: 数据管道：从 Git 仓库到 MySQL
date: 2019-08-13 14:00:00
tags:
  - Career
  - Git
  - 数据管道
categories:
  - Career
description: 上一篇把指标定义聊完了，这一篇讲数据怎么进来——Git 仓库怎么解析、字段怎么落库、表结构怎么设计，以及我们踩过的几个坑。
---

上一篇把指标定义聊完了，这一篇往下走一层：数据怎么进来。

我们这套画像系统的数据源大头就是 Git 仓库，几百万行代码、几十个仓库、几十号人的提交记录。如果解析方式选错，后面所有清洗和归一都会跟着崩。所以这一篇专门讲 Git 解析和入库 MySQL 的工程做法。

## 先把范围说清楚

我们要解决的不是"看一份 Git 报表"，而是把多个仓库的提交行为结构化地落到 MySQL 里，供后面的 AirFlow 任务算指标。

也就是说，这一层只负责把 Git 数据"原样"搬进数据库，不做任何业务计算。这是刻意的，采集层和计算层必须分开，否则一改指标定义就得重跑一遍 Git 解析，那 Git 仓库早就不知多大去了。

采集层只回答一个问题：这个 commit 里到底发生了什么。剩下的交给下游。

## 解析 Git 的几种方式

最直观的当然是直接调 `git log`：

```bash
# 最朴素的写法
git log --pretty=format:'%H|%an|%ae|%aI|%s' \
       --numstat \
       --no-merges \
       --since='2019-01-01'
```

输出长这样：

```
a1b2c3...|罗某|luo@example.com|2019-05-12T14:03:22+08:00|修复登录bug
3       1       src/login.py
0       5       src/old_session.py
```

第一行是 commit 元信息，后面每行是 `added \t removed \t file_path`。看起来够用对吧？

但实战中至少有几个问题它处理不了：

1. **重命名**会被 `--numstat` 报成"删一个文件 + 加一个文件"，代码量直接翻倍。
2. **二进制文件**会显示 `-\t-\timage.png`，需要单独处理。
3. **大仓库下重复执行**特别慢，每次都要扫一遍完整历史。

所以最后我们没用裸 `git log`，而是用 [PyDriller](https://github.com/ishepypy/pydriller) 做了一层封装。PyDriller 内部会处理好上面这些边角，并且能拿到结构化的 commit 对象，省掉一堆字符串解析的活。

```python
from pydriller import Repository

for commit in Repository(
    path_to_repo='https://github.com/org/repo.git',
    since=datetime(2019, 1, 1, tzinfo=timezone.utc),
).traverse_commits():
    for mod in commit.modified_files:
        # 落库
        save_commit_file(
            sha=commit.hash,
            author_email=commit.author.email,
            author_date=commit.author_date,
            file_path=mod.new_path or mod.old_path,
            added=mod.added_lines,
            removed=mod.deleted_lines,
            is_binary=(mod.added_lines is None),
        )
```

PyDriller 不是没缺点，它对超大的 monorepo 会有点慢，但对我们几十个中等规模仓库来说够用，而且改起来快。

## 表结构怎么设计

采集层的表，我建议尽量贴近 Git 原始结构，不要过早抽象。

我们的核心就两张表（字段脱敏后大概长这样）：

```sql
-- commit 元信息
CREATE TABLE git_commit (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  sha           CHAR(40) NOT NULL,
  repo_id       INT NOT NULL,
  author_email  VARCHAR(255) NOT NULL,
  author_name   VARCHAR(128) NOT NULL,
  author_date   DATETIME NOT NULL,
  committer_date DATETIME NOT NULL,
  is_merge      TINYINT(1) NOT NULL DEFAULT 0,
  message       MEDIUMTEXT,
  UNIQUE KEY uk_repo_sha (repo_id, sha),
  KEY idx_author_date (author_email, author_date)
);

-- 每次改动到文件的粒度
CREATE TABLE git_commit_file (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  commit_id    BIGINT NOT NULL,
  repo_id      INT NOT NULL,
  sha          CHAR(40) NOT NULL,
  file_path    VARCHAR(512) NOT NULL,
  file_ext     VARCHAR(16) DEFAULT NULL,
  added_lines  INT DEFAULT NULL,
  removed_lines INT DEFAULT NULL,
  is_binary    TINYINT(1) NOT NULL DEFAULT 0,
  is_rename    TINYINT(1) NOT NULL DEFAULT 0,
  KEY idx_commit (commit_id),
  KEY idx_sha_path (sha, file_path)
);
```

几个我觉得比较关键的判断：

- 以 `(repo_id, sha)` 作为唯一键，不要用全局 sha。不同仓库的 sha 是可能"撞"的（虽然概率极低），加上 repo_id 才绝对稳。
- author 和 committer 都留。后期会经常遇到"提交是 A 写的，但 B 合并进来的"这种情况，author_date 才反映真实编码时间。
- 文件粒度的改动单独成表。不要把改动塞进 commit 表的一个 JSON 字段，后面按文件维度做聚合（比如"谁动这个文件最多"）会痛不欲生。
- file_ext 提前抽出来。算指标时经常要按语言过滤（只看 `.py` / `.js`），与其每次 `SUBSTRING_INDEX(file_path, '.', -1)`，不如入库时就抽好。

作者维度单独建一张映射表，把邮箱归并到"人"：

```sql
CREATE TABLE dim_developer (
  id           INT PRIMARY KEY AUTO_INCREMENT,
  canonical_email VARCHAR(255) NOT NULL,
  display_name VARCHAR(128) NOT NULL,
  team         VARCHAR(64),
  UNIQUE KEY uk_email (canonical_email)
);

-- 邮箱别名表：同一个人可能有公司邮箱、私人邮箱、noreply 邮箱
CREATE TABLE dim_developer_email (
  developer_id INT NOT NULL,
  email        VARCHAR(255) NOT NULL,
  PRIMARY KEY (developer_id, email),
  UNIQUE KEY uk_email (email)
);
```

这一步看着不起眼，但它是后面所有指标算得准不准的前提。同一个人在 GitHub 上用的是个人邮箱，公司 GitLab 上用的是企业邮箱，不归并的话一个人会被算成两个，提交量直接腰斩。

## 增量入库：用 sha 当水位

第一次跑是全量，从仓库第一个 commit 一路扫到现在。后续就只跑增量。

增量逻辑很简单：

```python
# 伪代码
last_sha = get_last_imported_sha(repo_id)
if last_sha:
    # 用 git rev-list 拿到 last_sha 之后的所有 commit
    new_shas = run(['git', 'rev-list', f'{last_sha}..HEAD', '--no-merges'])
else:
    new_shas = run(['git', 'rev-list', 'HEAD', '--no-merges'])

for sha in new_shas:
    parse_and_insert(sha)
```

实际工程上更稳的做法是不依赖 sha 顺序，而是用 author_date 做时间窗：拉最近 7 天的 commit，跟数据库里已有的 sha 集合做差。`(repo_id, sha)` 这个唯一键就是兜底，重复了直接 `INSERT IGNORE`。

不要相信 Git 的 commit 顺序是稳定的物理顺序。force push、rebase、cherry-pick 都会打乱它。我吃过亏。

## 几个踩过的坑

按时间顺序列几个印象深的：

1. merge commit 必须过滤掉。merge commit 自带的 diff 是空的或者重复的，混进指标会把代码量算飞。`--no-merges` 是默认动作。
2. vendor 目录、lockfile、生成代码要白名单过滤。`package-lock.json` 一改几千行，算进去就是噪音。我们在采集层就维护了一份"忽略路径"配置：
   ```yaml
   ignore_paths:
     - '**/node_modules/**'
     - '**/vendor/**'
     - '**/*.lock'
     - '**/package-lock.json'
     - '**/dist/**'
   ```
3. submodule 的提交不要算进外层仓库。PyDriller 默认不会展开 submodule，但有些工具会，需要注意。
4. 作者邮箱归并是长期工作。每隔一段时间就会发现新的别名：某某同学换电脑了、换公司邮箱了、用 GitHub noreply 了。映射表得有一个维护入口，最好接 SSO 当 source of truth。

## 一点感受

我感觉数据管道这种东西，业务逻辑越少越好。采集层就老老实实把数据搬进来，字段尽量贴近原始结构，任何"我觉得这个字段没用"的判断都先别下。后面做归一化的时候你会发现，当初没采的字段就是现在最缺的那个。

指标定义上一篇已经讲完了，数据也进 MySQL 了。下一篇讲 AirFlow 怎么把这些原始数据跑成清洗过、归一化过的指标。

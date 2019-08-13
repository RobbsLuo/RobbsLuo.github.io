---
title: From Git Repos to a Metrics Store
date: 2019-08-13 14:00:00
tags:
  - Career
  - Git
  - Data Pipeline
categories:
  - Career
description: With the metric definitions out of the way, this post walks through how the data actually flows in — how we parse Git repos, shape the fields, design the MySQL schema, and the traps we hit along the way.
lang: en
---

In the last post we settled the metric definitions. This one moves one layer down: **how the data actually arrives**.

The bulk of our profile system's data sources are Git repos. Millions of lines of code, dozens of repositories, commits from dozens of developers. Get the parsing wrong here, and every downstream cleaning and normalization step topples with it. So this post is specifically about the engineering of Git parsing and loading into MySQL.

## Scope first

We're not building "a nicer Git report." We're **structuring commits across many repos into MySQL, so downstream AirFlow tasks can compute metrics from them**.

Which means this layer's only job is to move Git data into the database as-is, with zero business logic. That's deliberate. **The collection layer and the computation layer must stay separate.** Otherwise every metric-definition change means re-parsing Git from scratch, and Git history only ever grows.

The collection layer answers exactly one question: what actually happened in this commit? Everything else is downstream.

## Ways to parse Git

The obvious move is to just call `git log`:

```bash
# Naive version
git log --pretty=format:'%H|%an|%ae|%aI|%s' \
       --numstat \
       --no-merges \
       --since='2019-01-01'
```

Output looks like:

```
a1b2c3...|Luo|luo@example.com|2019-05-12T14:03:22+08:00|fix login bug
3       1       src/login.py
0       5       src/old_session.py
```

First line is commit metadata, every line after is `added \t removed \t file_path`. Seems usable, right?

In practice, there are at least three things it can't handle:

1. **Renames** get reported by `--numstat` as "delete one file + add one file," doubling the apparent code change.
2. **Binary files** show up as `-\t-\timage.png` and need special handling.
3. **Re-running on large repos** is painfully slow: every run walks the full history again.

So we didn't end up using bare `git log`. We wrapped [PyDriller](https://github.com/ishepypy/pydriller), which handles those edge cases internally and hands back structured commit objects, saving us a lot of string parsing.

```python
from pydriller import Repository

for commit in Repository(
    path_to_repo='https://github.com/org/repo.git',
    since=datetime(2019, 1, 1, tzinfo=timezone.utc),
).traverse_commits():
    for mod in commit.modified_files:
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

PyDriller isn't perfect. It gets sluggish on giant monorepos, but for our dozens of mid-sized repos it was good enough, and **it was fast to modify**.

## Schema design

For the collection layer, my advice is to **stay close to Git's native shape, and resist premature abstraction**.

We have essentially two core tables (desensitized):

```sql
-- Commit metadata
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

-- Per-file change in each commit
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

A few decisions I'd defend:

- **Use `(repo_id, sha)` as the unique key**, not a global sha. SHAs from different repos can theoretically collide; prefixing with repo_id makes it bulletproof.
- **Keep both author and committer.** Sooner or later you'll hit "A wrote it, B merged it," and author_date is what reflects real coding time.
- **Per-file changes get their own table.** Don't cram changes into a JSON column on the commit row. Aggregations like "who touched this file the most" will become miserable.
- **Pre-extract file_ext.** Metrics frequently filter by language (only `.py` / `.js`). Doing this at insert time beats running `SUBSTRING_INDEX(file_path, '.', -1)` on every query.

Authors get a separate mapping table that collapses emails to "people":

```sql
CREATE TABLE dim_developer (
  id           INT PRIMARY KEY AUTO_INCREMENT,
  canonical_email VARCHAR(255) NOT NULL,
  display_name VARCHAR(128) NOT NULL,
  team         VARCHAR(64),
  UNIQUE KEY uk_email (canonical_email)
);

-- Aliases: one person may have a work email, a personal email, and a noreply
CREATE TABLE dim_developer_email (
  developer_id INT NOT NULL,
  email        VARCHAR(255) NOT NULL,
  PRIMARY KEY (developer_id, email),
  UNIQUE KEY uk_email (email)
);
```

This step looks unremarkable, but **it's the foundation for every downstream metric being accurate**. Someone using their personal email on GitHub and their work email on GitLab will otherwise get split in two, commit count halved overnight.

## Incremental load: use SHA as the watermark

The first run is full: walk the repo from its very first commit to now. After that, only increments.

The increment logic is simple:

```python
# Pseudocode
last_sha = get_last_imported_sha(repo_id)
if last_sha:
    # All commits after last_sha
    new_shas = run(['git', 'rev-list', f'{last_sha}..HEAD', '--no-merges'])
else:
    new_shas = run(['git', 'rev-list', 'HEAD', '--no-merges'])

for sha in new_shas:
    parse_and_insert(sha)
```

In practice the more robust version **doesn't trust commit order, but uses author_date as a time window**: pull the last 7 days of commits, diff against the SHA set already in the database. The `(repo_id, sha)` unique key is the safety net; duplicates just `INSERT IGNORE`.

> Don't trust Git's commit order to be a stable physical order. Force pushes, rebases, and cherry-picks all scramble it. Learned that the hard way.

## Traps we hit

In rough chronological order:

1. **Merge commits must be filtered out.** Their diff is empty or duplicated; let them through and code-change totals explode. `--no-merges` is the default action.
2. **Vendor dirs, lockfiles, generated code need a whitelist.** A `package-lock.json` change is thousands of lines of pure noise. We keep an ignore-path config at the collection layer:
   ```yaml
   ignore_paths:
     - '**/node_modules/**'
     - '**/vendor/**'
     - '**/*.lock'
     - '**/package-lock.json'
     - '**/dist/**'
   ```
3. **Submodule commits don't belong to the outer repo.** PyDriller doesn't expand submodules by default, but some tools do, so be aware.
4. **Email reconciliation is ongoing work.** Every so often a new alias shows up: someone got a new laptop, switched to a work email, started using GitHub's noreply. The mapping table needs a maintenance entry point, ideally sourced from SSO.

## One closing thought

For data pipelines, my feeling is: **the less business logic, the better**. The collection layer should just move data in, with fields that stay close to the source. Resist every "I don't think we need this field" instinct. When you get to normalization, the field you skipped is the one you'll be missing most.

Metric definitions are done, the data is in MySQL. Next post: how AirFlow turns that raw data into cleaned, normalized metrics.

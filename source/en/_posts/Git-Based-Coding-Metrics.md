---
title: Defining Coding Metrics for a Developer Profile
date: 2019-05-14 10:00:00
tags:
  - Career
  - Developer Profile
  - Git
categories:
  - Career
description: Building a developer profile system? The first real hurdle is deciding what "coding metrics" even means. This post walks through the dimensions we settled on in CoreTeam — after breaking the first version.
lang: en
---

## Why start with metric definitions

A developer profile system sounds fancy, but the moment you actually start building one, you hit a wall: **how do you define the metrics?**

The CoreTeam profile system was originally meant to quantify engineering capability: pull behavioral data out of Git repos, the ticketing system, and code review tools, then analyze each person's coding ability, collaboration habits, and growth potential across multiple dimensions.

And the thing about quantification is: **get the metrics wrong and the whole system is garbage**. That's the most direct lesson I took away from it.

For example, when we looked at open-source engineering-insight tools early on, almost all of them tracked exactly two things: commit count and lines of code. Sounds reasonable, right? In practice, those two are the textbook example of anti-KPIs.

> Measure only lines of code, and the guy who writes fluffy code will happily hand you three thousand lines a day.

Measure only commit count, and it gets worse. Some people get addicted to splitting commits, breaking one feature into 20 commits just to look busy.

So my take is: **before a metric gets a column in the database, you have to be clear about what it actually reflects**. The dimensions below are what we ended up with after the first version got torn apart in review.

## Combine metrics so no single one can be gamed

When we landed the system, we split coding metrics into four dimensions:

- **Commit dimension**: count, frequency, regularity
- **Code dimension**: added lines, removed lines, net change, files touched
- **Collaboration dimension**: review participation, times reviewed, comment density
- **Output dimension**: requirements closed, bugs fixed, defect density

These four **constrain each other**. Any one of them in isolation invites gaming; only together do they paint a realistic picture.

## Commit dimension: steadiness matters more than volume

For commit count itself, we eventually stopped caring much about the absolute number. **What we really want to see is rhythm.**

```python
# Simplified commit-rhythm feature extraction
def commit_pattern(commits):
    counts_by_day = group_by_day(commits)
    return {
        "total": len(commits),
        "active_days": len(counts_by_day),
        "stddev": stddev(list(counts_by_day.values())),
        "burst_ratio": max(counts_by_day.values()) / mean(counts_by_day.values()),
    }
```

Specifically, we look at:

- **Active days**: how many days in a period actually had commits. This tells you more than total count. Someone active 5 days out of 30 is clearly in a different mode from someone active every day.
- **burst_ratio**: peak-day commit count divided by the daily mean. A very high value usually signals a "cram style": slacking off until right before a release, then panic-pushing.
- **Rhythm stddev**: low variance means work is planned; high variance usually means firefighting.

This whole group exists to filter out the "spike performers." **Our belief: steady output over the long haul is worth more than short bursts.**

## Code dimension: net change beats raw line count

Line count is the easiest trap to fall into.

We went back and forth on whether to even track "lines added" and "lines removed." In the end we decided to track both, but not weight them the same.

```sql
-- Simplified code-change aggregation (desensitized pseudo-SQL)
SELECT
  author_id,
  SUM(added_lines)   AS added,
  SUM(removed_lines) AS removed,
  SUM(added_lines - removed_lines) AS net_change,
  SUM(files_touched) AS files_touched,
  -- change volume / commit count, so one giant commit can't dominate
  SUM(added_lines + removed_lines) / COUNT(DISTINCT commit_id) AS per_commit_load
FROM git_commit_diff
GROUP BY author_id;
```

A few key calls:

- **Deleting code is also contribution.** People who delete are usually the ones actually refactoring and paying down tech debt. People who only add code can be dangerous.
- **Net change** is more honest than raw lines. Someone who adds 1000 lines and removes 800 is not in the same league as someone who adds 200, but "lines added" alone gives the first person a higher score.
- **per_commit_load** (average change per commit) guards against commit-splitting. High commit count with very low per_commit_load almost always means noise.

> If someone else solves in one line what you couldn't crack in thirty, I don't want that "effort" counted as productivity.

## Collaboration dimension: reviews are where it's at

This is the dimension I personally care most about.

**Writing code is individual skill. Doing reviews is team contribution.** If only one or two people on a team do reviews, that team's code quality will suffer, guaranteed.

We collect a handful of fields:

- **Active review count**: how many PRs/MRs by others you commented on
- **Times reviewed**: how many people looked at your code
- **Comment density**: average comments left per PR
- **Review response time**: gap from PR submission to first review comment

Here's a counter-intuitive one: **high comment density is a good sign.** It means reviewers are actually reading the code and digging into details, not just dropping an LGTM and walking away.

You do have to strip noise, though. Pure "+1" or emoji-only comments get filtered out at the cleaning layer (more on that in the next post, when we talk about the data pipeline).

## Output dimension: tie it back to the business

Code alone isn't enough. **An engineer's value ultimately has to land on business outcomes.**

We pulled data from the requirement system (Jira-like tools) too:

- **Requirements closed**: number of requirements/tasks closed in a period
- **Bugs fixed**: how many bugs handled
- **Defect density**: bugs introduced by your own code, divided by your code volume
- **Delivery cycle**: average time from a requirement being claimed to being closed

This group is the hardest to keep clean, because requirement-system data quality is bad to begin with. Some people create a ticket for every little thing; others do a ton of work without creating any ticket at all. So during normalization we treat the output dimension as **a reference, not a ranking**. Look for anomalies, don't rank people.

> Rankings push people to game the data. Looking for anomalies actually surfaces problems. That's what we learned after shipping v1.

## The summary table

Roughly (desensitized):

| Dimension | Fields | What it reflects |
|---|---|---|
| Commit | active days, burst_ratio, rhythm stddev | Planned vs. chaotic |
| Code | added/removed/net, per_commit_load | Real workload |
| Collaboration | review count, comment density, response time | Team contribution |
| Output | requirements, bugs, defect density | Business outcome |

Four dimensions, about forty-plus fields in total. **Not a huge number, but every single one was argued over.**

## One closing thought

My feeling is that the hardest part of a developer profile system isn't the technology. **It's deciding what "good" means.**

The stack is almost a side issue. Git parsing, AirFlow, a Vue dashboard, those are all engineering problems, and engineering problems have solutions. Metric definition is a design problem. **It directly determines whether your system is incentivizing the right behavior or just forcing people to perform.**

So that's why this post exists. The next three will cover:

1. The data pipeline: how we parse Git repos and load them
2. Metric computation: how AirFlow handles cleaning and normalization
3. Visualization: how the Vue 2 dashboard surfaces anomalies

Once the fields are settled, the next step is getting the data in. See you in the next post.

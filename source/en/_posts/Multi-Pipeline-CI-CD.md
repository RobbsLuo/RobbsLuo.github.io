---
title: Running Jenkins, GitLab CI and GitHub Actions Together
date: 2020-07-14 15:00:00
tags: [Career, Jenkins, GitLab CI, GitHub Actions, DevOps]
categories:
  - Career
description: In 2020 CoreTeam was running Jenkins, GitLab CI and GitHub Actions simultaneously. It was not a tooling mistake — it was a pragmatic choice under different business scenarios. This post explains why all three coexisted and where the boundaries were.
lang: en
---

## First, admit it sounds absurd

Mid-2020, someone asked what CI/CD we used. I said "Jenkins, GitLab CI, and GitHub Actions, all three running." Their expression shifted, and the subtext was clear: "What kind of ops team can't even unify their tools?"

Guilty as charged; on the surface it is not elegant. **But the reality of internal systems is that the cost of unifying often outweighs the cost of living with the split.**

> **Unification is not the goal, getting work done is.** Forcing a migration to "look tidy" hurts the business.

How we ended up with three running in parallel for over a year is what follows.

## Jenkins: the workhorse carrying the heaviest load

Jenkins was our earliest CI/CD, running since 2018. The previous post on Jenkins + ECS covers it in detail. By 2020 it was still carrying **the release of every production service**, about 60-plus core business services, all going through Jenkins pipelines into ECS.

Why did we not migrate away?

1. **Mature, stable pipelines.** These services' Jenkinsfiles had been polished for two years; every stage had reasoning behind it. Migrating meant rewriting all of that.
2. **Deep plugin dependencies.** Plugins (AWS ECS plugin, specific Slack notifier, etc.) were tuned in Jenkins; moving meant re-stepping traps.
3. **Business teams knew it.** Forcing a dozen teams to learn a new CI syntax was a huge training cost.

Jenkins's role was clear: **the last mile of production release**. Anything going to production had to go through Jenkins pipelines and approvals.

```
Jenkins boundaries:
✅ Production release (ECS deploy)
✅ Cross-service integration tests
✅ Scheduled database backup jobs
❌ PR checks (handled by GHA)
❌ Internal tool builds (handled by GitLab CI)
```

## GitLab CI: the new favorite for internal services

Early 2020 the company brought in GitLab as an internal code host (some teams migrated over), and with it came GitLab CI.

Why did some teams move? Because GitLab integrates hosting and CI, **and for internal business teams, one fewer system means one fewer thing to maintain**. A few GitLab CI features impressed us at the time:

- **`.gitlab-ci.yml` is cleaner than Jenkinsfile.** YAML has a lower barrier than Groovy; business teams are more willing to edit it themselves.
- **Runner registration is simple.** Install gitlab-runner, register, done.
- **Environment isolation built in.** `tags` route jobs to different runners; dev jobs run on dev hosts, prod jobs on prod hosts.
- **Built-in artifact management.** No need to set up Nexus or Artifactory separately.

A typical `.gitlab-ci.yml` looks like:

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - package

variables:
  DOCKER_IMAGE: registry.coreteam.internal/user-service

unit_test:
  stage: test
  image: node:14-alpine
  script:
    - npm ci
    - npm test -- --coverage
  coverage: '/All files\s*\|\s*([\d\.]+)/'
  artifacts:
    reports:
      junit: test-results.xml

build:
  stage: build
  image: docker:19.03
  services:
    - docker:19.03-dind
  script:
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
  only:
    - main
    - tags
```

GitLab CI's role here: **CI and build for internal business systems**. Characteristics of these services:

- Not customer-facing (or used only in internal-facing environments).
- Team-autonomous, no need for platform involvement.
- Fast iteration, want a build the moment a PR merges.

> **The core question in tool selection is which best fits this scenario, not which is strongest.** GitLab CI beat Jenkins handily on "team-autonomous internal release."

## GitHub Actions: the light cavalry for open source and tooling

The third one is GitHub Actions. This sounds even stranger: if you have GitLab, why is GitHub still around?

A few reasons:

1. **Open source projects live on GitHub.** The company had a few open-sourced tool libraries that had to stay on GitHub.
2. **Internal CLI tools, scaffolds, lightweight projects** were on GitHub private repos because the team preferred GitHub's DX.
3. **Cross-repo workflows.** Things like "tag push auto-publishes an npm package" or "PR auto-runs lint" were smoothest in GHA.

GHA's killer feature is the **marketplace**. Many things do not need to be written from scratch: to publish an npm package, use `actions/setup-node` + a one-line `npm publish`; to scan code, drop in the codeql-action.

A common GHA workflow of ours:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '14'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

GHA's role here: **open source projects + lightweight toolchain automation**.

## Why not unify?

This is the most-asked question. The honest answer is that **unification does not pay off**.

Let's do the math:

- Migrating 60-plus Jenkins services to GitLab CI: conservatively 3 months, during which business teams need to relearn, platform needs to rebuild, pipelines need to be re-tuned. **Incident probability doubles during those 3 months.**
- Migrating internal GitLab services back to Jenkins: business teams would revolt.
- Migrating open-source GitHub projects to GitLab: open-source collaboration breaks.

> **The hidden cost of "unification" is the stability risk and team learning cost during migration.** Those two bills are far higher than the maintenance cost of running three.

But **not unifying is not the same as not managing**. We did several things to mitigate the pain of fragmentation:

### 1. Shared artifact registry

No matter which CI builds it, images go to ECR, npm packages go to an internal Nexus. **The artifact store is unified, the tools do not have to be.**

### 2. Shared deploy scripts

The deploy scripts called from Jenkins, GitLab CI and GHA are the same (`ecs-deploy.sh` from the earlier post). **Deploy logic has a single source of truth**, avoiding divergent outcomes from different CIs.

### 3. Shared notification channels

Failure notifications from all three CIs go to the same Slack channel. Business teams do not care which CI failed; they care that "my build broke."

### 4. Documentation with explicit boundaries

In the docs we hand to new hires, **there is a diagram showing which CI handles what**. This is the most effective way to eliminate confusion: clarify which tool to use when, rather than forcing the tools themselves to merge.

```
┌──────────────────────────────────────────────┐
│           Which CI should I use?              │
├──────────────────────────────────────────────┤
│ Production services  → Jenkins               │
│ Internal business    → GitLab CI             │
│ OSS / tools / bots   → GitHub Actions        │
└──────────────────────────────────────────────┘
```

## The real cost of running three

No sugar-coating. **There is a real cost.**

1. **Ops overhead.** Jenkins needs plugin upgrades, GitLab Runners need capacity planning, GHA self-hosted runners (if used) need management. All three need attention.
2. **Learning curve.** New hires have to learn three syntaxes. We mitigate with docs and mentorship, but the first two weeks are genuinely painful.
3. **Cross-toolchain coordination is hard.** A microservice built as a base image from an OSS repo, referenced by an internal GitLab repo, finally released via Jenkins. This chain is genuinely complex to monitor and trace.

But these costs, compared to "forced migration risk", are acceptable.

## Looking back

This mid-2020 moment, three running in parallel, was the **most pragmatic answer** we could find.

**I do not regret not forcing unification.** In hindsight, companies that pushed hard for "one toolchain" either spent a fortune migrating, or gave up halfway and ended up with four. We acknowledged reality, did isolation and sharing well, and walked steadily.

> **Internal systems are not about picking the single best tool. They are about using the most fitting tool per scenario, and making them collaborate.** Tool diversity itself is not the problem; lack of collaboration is.

When we re-evaluated in 2021, we noticed Jenkins's share was naturally declining: new services went straight to GitLab CI, and legacy Jenkins services decreased as businesses wound down. **Unification is better left to time, rather than forced.**

Next post: going deeper on Terraform for AWS. The further you go on infrastructure as code, the deeper the water.

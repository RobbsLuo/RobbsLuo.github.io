---
title: CI/CD 三套流水线并存的日子
date: 2020-07-14 15:00:00
tags:
  - Career
  - Jenkins
  - GitLab CI
  - GitHub Actions
  - DevOps
categories:
  - Career
description: 2020 年 CoreTeam 同时跑着 Jenkins、GitLab CI 和 GitHub Actions。这不是技术选型失误，而是不同业务场景下的现实选择。这篇说清楚三套并存的原因和边界。
---

## 先承认这件事听起来很荒谬

2020 年中，有人问我们公司用什么 CI/CD，我说"Jenkins、GitLab CI、GitHub Actions 三个都在跑"。对方眼神一下子变了，那意思大概是，"你们运维是怎么当的，连工具都统一不了？"

这事我承认，表面看确实不优雅。但内部系统的现实是，统一工具的代价往往比维持分裂更高。

统一不是目的，把活干好才是。为了"看起来整洁"而强推迁移，最后吃亏的是业务。

我们这一年多是怎么走到三套并存的，挨个说清楚。

## Jenkins：老牌主力，扛着最重的活

Jenkins 是我们最早的 CI/CD，2018 年那会儿就在跑，前面那篇 Jenkins + ECS 的文章说得很细。到 2020 年它仍然扛着所有生产服务的发布，大概 60 多个核心业务，全部走 Jenkins pipeline 发布到 ECS。

为什么没动它？

1. 流水线成熟稳定。这些服务的 Jenkinsfile 都经过两年打磨，每一个 stage 都有讲究，迁移到别的工具等于重写一遍。
2. 插件依赖深。某些插件（比如 AWS ECS 插件、特定的 Slack 通知插件）在 Jenkins 里调好了，迁过去要重新踩坑。
3. 业务方熟悉。让十几个业务团队重新学一套 CI 语法，培训成本巨大。

Jenkins 的定位很明确：生产发布的最后一公里。任何上生产的动作，必须经过 Jenkins 的流水线和审批。

```
Jenkins 的边界：
✅ 生产发布（ECS 部署）
✅ 跨服务集成测试
✅ 定时数据库备份任务
❌ PR 检查（交给 GHA）
❌ 内部工具构建（交给 GitLab CI）
```

## GitLab CI：内部业务的新欢

2020 年初公司引入了 GitLab 作为内部代码仓库（部分团队迁过去），随之而来的就是 GitLab CI。

为什么有些团队迁过去了？因为 GitLab 把代码托管和 CI 整合在一起，对于内部业务团队来说，少一个系统就少一份维护成本。GitLab CI 的几个特性当时让我们眼前一亮：

- `.gitlab-ci.yml` 写起来比 Jenkinsfile 干净。YAML 比 Groovy 门槛低，业务方更愿意自己改。
- Runner 注册简单。装个 gitlab-runner，注册一下就能跑。
- 环境隔离天然支持。`tags` 把 job 路由到不同 runner，dev 跑 dev 机器，prod 跑 prod 机器。
- 制品管理内置。不需要单独配 Nexus 或 Artifactory。

一个典型的 `.gitlab-ci.yml` 长这样：

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

GitLab CI 在我们这里的定位：内部业务系统的 CI 和构建。这些服务的特点是：

- 不上生产（或者上的是内部使用的环境）。
- 团队自治，不需要平台组介入。
- 迭代快，希望 PR 一合并就有制品。

工具选型的核心问题，不是"哪个最强"，而是哪个最贴合这个场景。GitLab CI 在"团队自治 + 内部发布"这个场景下完胜 Jenkins。

## GitHub Actions：开源和工具链的轻骑兵

第三套是 GitHub Actions。这事儿听起来更怪，既然公司有 GitLab，为什么还有 GitHub？

原因有几个：

1. 开源项目托管在 GitHub。公司有几个对外开源的工具库，必须留在 GitHub 上。
2. 内部 CLI 工具、脚手架这类轻量项目，托管在 GitHub 私有 repo 上，团队偏好 GitHub 的开发体验。
3. 跨 repo 工作流。比如"打 tag 自动发 npm 包"、"PR 自动跑 lint"，这类小自动化用 GHA 最顺手。

GitHub Actions 的杀手锏是 marketplace。很多事情不用自己写，想发 npm 包，用 `actions/setup-node` + 一行 `npm publish`；想做 code scan，装个 codeql-action 就行。

一个我们常用的 GHA workflow：

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

GHA 在我们这里的定位：开源项目 + 轻量工具链自动化。

## 为什么不统一？

这是被问最多的问题。答案不是"不能统一"，而是统一不划算。

算一笔账：

- 把 60 多个 Jenkins 服务迁到 GitLab CI：保守估计 3 个月，期间业务团队要重新学、平台组要重新搭、流水线要重新调。这 3 个月里出事故的概率翻倍。
- 把 GitLab 内部业务迁回 Jenkins：业务团队集体抗议。
- 把 GitHub 开源项目迁到 GitLab：开源协作断了。

"统一"的隐性成本是迁移期间的稳定性风险和团队学习成本。这两笔账，比"维持三套"的维护成本高得多。

但不统一不等于不管。我们做了几件事来缓解分裂的痛苦：

### 1. 共享制品库

不管哪套 CI 构建，镜像统一推到 ECR，npm 包统一推到内部 Nexus。制品库是统一的，工具不必统一。

### 2. 共享部署脚本

Jenkins、GitLab CI、GHA 三套里调用的部署脚本，是同一份（前面那篇提到的 `ecs-deploy.sh`）。部署逻辑只有一份真理来源，避免不同 CI 部署出不一样的结果。

### 3. 共享通知渠道

三套 CI 的失败通知，全部进同一个 Slack channel。业务方不关心是哪个 CI 失败的，只关心"我的构建挂了"。

### 4. 文档明确定位

新人入职我们给的文档里，有一张图说明三种 CI 各自负责什么。这是消除混乱最有效的办法，消除的不是工具多样性，而是认知混乱。

```
┌──────────────────────────────────────────────┐
│             该用哪个 CI？                      │
├──────────────────────────────────────────────┤
│ 上生产的服务      → Jenkins                   │
│ 内部业务系统       → GitLab CI                │
│ 开源/工具/自动化   → GitHub Actions           │
└──────────────────────────────────────────────┘
```

## 三套并存的真实代价

不卖关子，确实有代价：

1. 运维成本。Jenkins 要维护插件升级、GitLab Runner 要扩容、GHA 的 self-hosted runner（如果用）要管。三套都要人盯。
2. 学习曲线。新人进来要学三套语法。我们靠文档和 mentor 缓解，但前两周确实痛苦。
3. 跨工具链协作难。一个微服务从开源 repo 构建出基础镜像，被内部 GitLab repo 引用，最后通过 Jenkins 发布，这种链路在监控和追溯上确实复杂。

但这些代价，比起"强行迁移导致的事故风险"，是可接受的。

## 现在回看

2020 年这个时间点，三套并存是我们能找到的最务实的解。

我不后悔没强推统一。回头看，那些年强推"统一工具链"的公司，要么花了大价钱迁移，要么迁到一半放弃变成四套。我们承认现实、做好隔离和共享，反而走得稳。

内部系统的路，从来不是"选一个最好的工具"，而是在不同场景下用最合适的工具，并让它们能协作。工具多样性本身不是问题，缺乏协作才是。

后来到 2021 年我们再评估这件事，发现 Jenkins 的份额在自然下降，新服务直接走 GitLab CI，老的 Jenkins 服务随着业务下线也在减少。统一这件事，与其强推，不如让时间慢慢消化。

下一篇写 Terraform 在 AWS 上的深化，基建即代码这条路，越往深走水越深。

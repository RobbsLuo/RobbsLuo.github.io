---
title: 服务发布这件事，我们交给 Jenkins 和 ECS
date: 2018-08-20 10:00:00
tags:
  - Career
  - Jenkins
  - AWS ECS
  - Docker
  - DevOps
categories:
  - Career
description: 资源管起来之后，下一个绕不开的坎是"发布"。这篇记我们怎么把 Jenkins 流水线、Docker 镜像和 AWS ECS 拼到一起，做出业务团队能自己用的自助发布。
---

## 发布这件事，从来都是个麻烦

上一篇聊到我们把 AWS 资源盘点清楚了。地基有了，下一个问题马上来，服务怎么发上去？

听起来很简单对吧，git push 完事。但实际跑过线上的都知道，发布这件事涉及的环节多得很：

- 代码从哪儿拉、哪个分支、哪个 commit
- 怎么打包，打到哪儿
- 配置怎么注入，密钥怎么办
- 怎么不中断老版本
- 出问题怎么回滚
- 业务方自己能不能操作

2018 年那会儿 CoreTeam 内部业务团队十几个，每个团队发版节奏都不一样。最早是手动 SSH 上 EC2，`git pull` 然后 `service restart`。这种发布方式最大的问题不是慢，是不可重复，你永远不知道上次发的是哪个 commit，谁发的，发完之后改过什么。

不可重复的发布，本质上是事故的温床。不是"会不会出事"的问题，是"什么时候出事"的问题。

所以我们定了一个目标：让业务团队能自己点一下把服务发出去，不用找运维。这就是所谓的"自助发布"。

## 选型：Jenkins + ECS + Docker

技术栈其实没什么悬念。

- Docker：镜像把运行时环境打包进去，构建一次到处跑。这是摆脱"环境差异"的根本手段。
- AWS ECS：当时公司主力在 AWS，K8s 还在早期，ECS 是更稳的选择。Fargate 那时刚出还不成熟，我们走的是 EC2 launch type。
- Jenkins：CI/CD 老牌选手，插件生态无敌，团队里大多数人都用过。

也有人提过 Spinnaker，但那东西部署本身就不轻，对一个十几人的内部团队来说太重了。工具选型有一条铁律：选团队能 hold 住的，而不是"听起来最先进"的。

## 整体流水线设计

我们最终落地的发布流水线长这样：

```
开发 push 代码
   ↓
Jenkins 触发构建
   ↓
构建 Docker 镜像 → 推到 ECR
   ↓
生成新的 ECS Task Definition
   ↓
更新 ECS Service（滚动发布）
   ↓
健康检查通过 → 老任务下线
```

听着挺顺的，但每个环节都有讲究。下面挨个说。

### 1. Dockerfile 的统一约定

我们不强迫每个业务团队从零写 Dockerfile。平台组维护了一套基础镜像，业务方只需要在末尾加自己的代码：

```dockerfile
# 平台提供的基础镜像
FROM coreteam/base-node:10-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

# 业务方只需关心这里
CMD ["node", "server.js"]
```

这样有几个好处：

- 基础镜像里的监控 agent、日志收集、安全补丁统一升级，业务无感。
- Dockerfile 极简，业务团队不犯错。
- 镜像体积可控（alpine 基础上几十 MB）。

统一的 base image 是多团队 Docker 化的关键。让每个团队自己折腾基础镜像，最后一定是一地鸡毛。

### 2. Jenkinsfile：把流水线写成代码

Jenkins 2.0 之后有了 Pipeline as Code，所有构建逻辑写到 `Jenkinsfile` 里跟着仓库走。我们定了一份模板：

```groovy
// Jenkinsfile
pipeline {
  agent any
  environment {
    AWS_ACCOUNT_ID = '123456789012'
    ECR_REPO       = 'coreteam/user-service'
    IMAGE_TAG      = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(8)}"
  }
  stages {
    stage('Build') {
      steps {
        sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
      }
    }
    stage('Push') {
      steps {
        sh "aws ecr get-login --region us-east-1 | sh"
        sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
      }
    }
    stage('Deploy') {
      steps {
        // 调用一个共享的 deploy 脚本
        sh "./scripts/ecs-deploy.sh ${ECR_REPO}:${IMAGE_TAG} user-service prod"
      }
    }
  }
  post {
    failure {
      slackSend channel: '#alerts', message: "发布失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
  }
}
```

注意几个细节：

- IMAGE_TAG 带 BUILD_NUMBER 和 GIT_COMMIT，永远不重复，不要用 latest 这种坑爹标签。
- 失败自动通知 Slack，不用人去盯。
- deploy 脚本是平台组统一维护的，业务方不直接碰 ECS API。

### 3. ECS Task Definition 怎么生成

ECS 的核心概念其实就两个：

- Task Definition：一个容器长什么样（镜像、CPU/内存、端口、环境变量、日志驱动）
- Service：这个 Task 跑几份，挂在哪个 ALB 后面

每次发布要做的就是用新镜像生成一份新的 Task Definition revision，然后让 Service 用新版本。

```bash
#!/usr/bin/env bash
# scripts/ecs-deploy.sh (简化版)
IMAGE=$1
SERVICE=$2
CLUSTER=$3
FAMILY="coreteam-${SERVICE}"

# 基于模板生成新的 task definition
TASK_DEF=$(aws ecs describe-task-definition \
  --task-definition ${FAMILY} \
  --query taskDefinition)

# 替换镜像
NEW_TASK=$(echo "$TASK_DEF" | \
  jq --arg IMG "$IMAGE" \
  '.containerDefinitions[0].image = $IMG | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities)')

# 注册新版本
NEW_REVISION=$(echo "$NEW_TASK" | \
  aws ecs register-task-definition --cli-input-json file:///dev/stdin \
  --query 'taskDefinition.taskDefinitionArn' --output text)

# 更新 service
aws ecs update-service \
  --cluster ${CLUSTER} \
  --service ${SERVICE} \
  --task-definition ${NEW_REVISION}
```

这段脚本后来成了整个公司发布的"标准动作"。

### 4. docker-compose：本地和生产的对齐

有个坑必须提，本地能跑、上了 ECS 就跪。原因通常是：本地用 docker-compose，多个容器能互相访问；但 ECS 上网络模型不同，服务发现机制也不一样。

我们的解法是让 docker-compose 配置尽量贴近 ECS Task Definition。一个服务如果 Task Definition 里挂了 Redis sidecar，本地 docker-compose 也得有对应的 Redis 服务：

```yaml
# docker-compose.yml
version: "3"
services:
  app:
    image: coreteam/user-service:latest
    build: .
    ports:
      - "3000:3000"
    environment:
      - REDIS_URL=redis://cache:6379
    depends_on:
      - cache
  cache:
    image: redis:5-alpine
```

本地环境和生产环境的拓扑对齐，是减少"在我机器上能跑"的根本手段。这事看着小，但能省下大量排查时间。

## 滚动发布：最小可用性优先

ECS 自带 rolling update，但默认参数不一定够用。我们调过几个关键参数：

```hcl
# Service 的部署配置
deployment_controller {
  type = "ECS"
}

# 关键：老任务下线前，新任务必须先健康
minimum_healthy_percent = 100   # 发布期间至少保持原数量
maximum_percent         = 200   # 最多扩到两倍
```

`minimum_healthy_percent = 100` 意味着发布期间至少要保持现有的容量，先起新的再下老的。代价是发布期间会用双倍资源，但这钱花得值，为了省那点资源而牺牲可用性，是本末倒置。

## 这套东西跑起来之后

到 2018 年下半年，业务团队发版是这样操作的：

1. 在自己仓库写 `Jenkinsfile`（套模板）。
2. 写一个 `Dockerfile`（基于 base image）。
3. push 到 master 分支。
4. Jenkins 自动构建、推镜像、更新 ECS。
5. 5 分钟内新版本上线，健康检查通过，老版本下线。

整个过程不需要运维介入。运维只在新服务第一次接入时做一次配置，之后全是业务自服务。

内部工具的终极目标不是"运维很强"，而是"业务方不需要找运维"。你把路修好了，人家自己会走。

## 回头看

这一套 Jenkins + ECS + Docker 的组合，今天看可能不那么时髦，很多人会说"为什么不上 K8s"。但 2018 年的 ECS 是相当务实的选择，稳定、AWS 原生支持、运维成本低。我们用这套东西扛过了从十几个服务到上百个服务的增长。

真正的问题不在于用了什么工具，而在于有没有把发布这件事标准化、自助化、可观测化。后面 2020 年我们接入 GitLab CI 和 GitHub Actions 的时候，底层 ECS 这套依然在用，发布的"最后一公里"没怎么变。

下一篇写监控。日志那摊事，比发布更脏，但更躲不开。

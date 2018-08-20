---
title: Self-Service Release with Jenkins and ECS
date: 2018-08-20 10:00:00
tags: [Career, Jenkins, AWS ECS, Docker, DevOps]
categories:
  - Career
description: Once resources were under control, the next unavoidable question was "how do we ship services?" This is how we stitched together Jenkins pipelines, Docker images and AWS ECS to give business teams a release flow they could actually run themselves.
lang: en
---

## Release is always a pain

In the previous post I covered how we got AWS resources inventoried. Foundation in place. The next question arrived immediately: **how do we get services deployed onto it?**

Sounds trivial, right? Just git push. But anyone who has run production knows release involves a lot of moving parts:

- Where does the code come from, which branch, which commit
- How do you package it, and where to
- How do you inject config and secrets
- How do you avoid downtime
- How do you roll back when it breaks
- Can the business side do it themselves

Back in 2018 CoreTeam had a dozen-plus internal business teams, each shipping at its own cadence. Originally release meant SSHing onto an EC2, `git pull`, then `service restart`. **The biggest problem with this approach is that it is non-repeatable.** Slowness you can live with; never knowing which commit was deployed last time, by whom, and what got tweaked afterward, you cannot.

> **A non-repeatable release is, by nature, an accident waiting to happen.** Not a question of "if", but "when."

So we set a goal: **let business teams release their own services with a single click, without filing a ticket with ops.** Self-service release.

## Stack: Jenkins + ECS + Docker

There was not much suspense here.

- **Docker**: images bundle the runtime, build once run anywhere. The fundamental way out of "works on my machine".
- **AWS ECS**: the company was on AWS, Kubernetes was still early, ECS was the steady choice. Fargate had just launched and was not mature yet, so we went with the EC2 launch type.
- **Jenkins**: the old guard of CI/CD, unbeatable plugin ecosystem, and most of the team already knew it.

Someone floated Spinnaker, but deploying Spinnaker itself is not light, and it was too heavy for a small internal platform team. **One iron rule of tool selection: pick what the team can actually hold, not what "sounds most advanced."**

## The pipeline design

The pipeline we landed on looked like this:

```
dev pushes code
   ↓
Jenkins triggers a build
   ↓
build Docker image → push to ECR
   ↓
generate new ECS Task Definition
   ↓
update ECS Service (rolling deploy)
   ↓
health checks pass → old tasks drain
```

Sounds clean, but every step has its own subtleties.

### 1. A unified Dockerfile convention

We did not force every business team to write Dockerfiles from scratch. Platform maintained a set of **base images**, and business teams only had to add their code at the end:

```dockerfile
# Base image provided by platform
FROM coreteam/base-node:10-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

# Business team only cares about this
CMD ["node", "server.js"]
```

A few benefits:

- Monitoring agents, log shippers, security patches in the base image get upgraded centrally, transparently.
- Dockerfiles stay minimal, business teams can't shoot themselves in the foot.
- Image size is predictable (tens of MB on alpine).

**A shared base image is the key to multi-team Dockerization.** Let every team fight their own base image and you get a junkyard.

### 2. Jenkinsfile: pipeline as code

Jenkins 2.0 brought Pipeline as Code. All build logic lives in a `Jenkinsfile` checked into the repo. We provided a template:

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
        sh "./scripts/ecs-deploy.sh ${ECR_REPO}:${IMAGE_TAG} user-service prod"
      }
    }
  }
  post {
    failure {
      slackSend channel: '#alerts', message: "Deploy failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
  }
}
```

A few things to notice:

- **IMAGE_TAG** carries BUILD_NUMBER and GIT_COMMIT, always unique. **Never use tags like `latest`**; that is a footgun.
- **Failures ping Slack automatically**, no humans staring at Jenkins.
- The deploy script is maintained by platform. Business teams do not touch the ECS API directly.

### 3. How the ECS Task Definition is generated

ECS has two core concepts:

- **Task Definition**: what a container looks like (image, CPU/memory, ports, env vars, log driver)
- **Service**: how many copies of this task run, and which ALB they sit behind

Each release is: **generate a new Task Definition revision with the new image, then point the Service at it**.

```bash
#!/usr/bin/env bash
# scripts/ecs-deploy.sh (simplified)
IMAGE=$1
SERVICE=$2
CLUSTER=$3
FAMILY="coreteam-${SERVICE}"

# Pull the current task definition
TASK_DEF=$(aws ecs describe-task-definition \
  --task-definition ${FAMILY} \
  --query taskDefinition)

# Swap the image
NEW_TASK=$(echo "$TASK_DEF" | \
  jq --arg IMG "$IMAGE" \
  '.containerDefinitions[0].image = $IMG | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities)')

# Register the new revision
NEW_REVISION=$(echo "$NEW_TASK" | \
  aws ecs register-task-definition --cli-input-json file:///dev/stdin \
  --query 'taskDefinition.taskDefinitionArn' --output text)

# Update the service
aws ecs update-service \
  --cluster ${CLUSTER} \
  --service ${SERVICE} \
  --task-definition ${NEW_REVISION}
```

This script eventually became the "standard gesture" for releases across the company.

### 4. docker-compose: aligning local and prod

One trap deserves a mention: **runs locally, dies on ECS**. Usually because locally you use docker-compose, where containers see each other; ECS networking is different, and so is service discovery.

Our fix was to keep docker-compose topology as close to the ECS Task Definition as possible. If the task definition mounts a Redis sidecar, the local docker-compose gets the same Redis service:

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

**Aligning local topology with production topology is the real cure for "works on my machine."** Small thing, huge time saver.

## Rolling deploy: availability first

ECS does rolling updates natively, but **the defaults are not always enough**. We tuned a few critical parameters:

```hcl
deployment_controller {
  type = "ECS"
}

# Critical: new tasks must be healthy before old ones drain
minimum_healthy_percent = 100   # keep at least current capacity during release
maximum_percent         = 200   # allow up to double briefly
```

`minimum_healthy_percent = 100` means during release we maintain **at least the current capacity**: bring up new first, then drain old. The cost is double resources briefly during deploy, but it is money well spent. **Sacrificing availability to save those resources is putting the cart before the horse.**

## Once it was up

By the second half of 2018, releasing a service looked like this:

1. Write a `Jenkinsfile` in your repo (from a template).
2. Write a `Dockerfile` on top of the base image.
3. Push to master.
4. Jenkins builds, pushes the image, updates ECS automatically.
5. New version is live in 5 minutes, health checks pass, old version drains.

The whole thing **requires no ops involvement**. Ops only does the initial onboarding for a new service; after that it is fully self-service.

> **The endgame for internal tooling is "business teams don't need ops."** Build the road, people will walk it.

## Looking back

This Jenkins + ECS + Docker stack may not look trendy today; many would ask "why not K8s." But **ECS in 2018 was an extremely pragmatic choice**, stable, native to AWS, low operational overhead. It carried us from a dozen services to over a hundred.

The question was never which tool, but whether release was **standardized and self-service**. When we added GitLab CI and GitHub Actions in 2020, the underlying ECS layer barely changed. The "last mile" of release stayed put.

Next up, monitoring. Logging is even dirtier than release, but even harder to skip.

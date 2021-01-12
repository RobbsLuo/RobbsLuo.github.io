---
title: Going Deeper on ECS: Task Orchestration and Release Strategy
date: 2021-01-12 14:00:00
tags: [Career, AWS ECS, Docker, docker-compose, DevOps]
categories:
  - Career
description: "From \"runs\" to \"runs reliably\" is still a distance. This post covers ECS task orchestration details — sidecar design, health checks, blue-green vs rolling, capacity reservation and service discovery."
lang: en
---

## "It runs" and "it runs reliably" are not the same thing

The earlier Jenkins + ECS post covered how to "deploy" a service. But anyone who has used ECS knows: **getting a container up is just step one. The hard part is keeping it stable in production.**

By end of 2020 we had over a hundred services running on ECS clusters. Day-to-day issues we hit:

- A service occasionally returns 502 right after release, then self-heals in a few minutes.
- During a traffic peak, ALB sends requests to a still-starting container and connections fail.
- During blue-green cut-over, DB connections are not released cleanly, and old + new versions write to the DB simultaneously.
- A sidecar dies and takes the main service down with it.

The common thread: **none of these are "code bugs", they are "deployment architecture issues".** This post is about lessons at that layer.

## Task Definition: think through the container composition

In ECS, **the smallest scheduling unit is not a single container, it is a Task.** A Task can contain multiple containers that share a network namespace and storage volumes. This is almost identical to Kubernetes' Pod concept.

A few principles when designing a Task Definition:

### 1. Main container + sidecar composition

**The main container runs the business logic; sidecars run auxiliary functions.** Log shippers, monitoring agents, service mesh proxies: these belong in sidecars, not baked into the main container.

```json
{
  "family": "user-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["EC2"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "coreteam.dkr.ecr.us-east-1.amazonaws.com/user-service:v1.2.3",
      "essential": true,
      "portMappings": [{ "containerPort": 3000 }],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "LOG_LEVEL", "value": "info" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/user-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "app"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 30
      }
    },
    {
      "name": "log-router",
      "image": "coreteam.dkr.ecr.us-east-1.amazonaws.com/fluent-bit:1.8",
      "essential": false,
      "memoryReservation": 64,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": { "awslogs-group": "/ecs/user-service-sidecar" }
      }
    }
  ]
}
```

A few details:

- **`essential: true`** is only on the main container. A sidecar dying should not bring down the whole Task, so mark it `false` and it restarts independently.
- **`healthCheck`** must be configured. ECS uses it to decide if a container is ready. Without it, you are relying on ALB health checks as an indirect proxy.
- **`startPeriod: 30`** gives the application a grace period to start. Otherwise slow starters like Java get falsely marked unhealthy.

### 2. Fine-grained CPU and memory allocation

**ECS resource limits are hard.** Go over and you get OOM-killed with no negotiation. So you must configure based on real usage, not a hand-waved "big enough" number.

```json
{
  "cpu": "256",
  "memory": "512",
  "memoryReservation": "256"
}
```

`memory` is the hard limit: exceed it and you are killed. `memoryReservation` is a soft limit telling Docker "try not to let it use more than this". **The combination of soft + hard limits is more flexible than a single hard limit**; containers stay lean normally and can borrow from neighbors in bursts.

### 3. Use awsvpc network mode

ECS has three network modes: bridge, host, awsvpc. **In production we always use awsvpc.**

- bridge: Docker's default NAT mode. Port mapping is messy; multiple Tasks fight over ports.
- host: uses the host network directly. Best performance, weak isolation.
- awsvpc: each Task gets an ENI with its own private IP; security groups can be applied precisely.

```hcl
resource "aws_security_group" "user_service" {
  name = "user-service-sg"
  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Under awsvpc, every Task can have precise inbound/outbound rules via security groups.** Other modes cannot do this.

## ALB integration: controlled traffic shifting

The ECS Service sits behind an ALB. During release, the ALB's health check drives traffic shifting.

```hcl
resource "aws_ecs_service" "user_service" {
  name            = "user-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.user_service.arn
  desired_count   = 3
  launch_type     = "EC2"

  load_balancer {
    target_group_arn = aws_lb_target_group.user_service.id
    container_name   = "app"
    container_port   = 3000
  }

  deployment_maximum_percent         = 200
  deployment_minimum_healthy_percent = 100

  health_check_grace_period = 60
}
```

**`health_check_grace_period` is an often-overlooked but critical parameter.** It tells the ALB "do not health-check this container for the first 60 seconds after start." Otherwise the application is not ready yet, the ALB marks it unhealthy, and the new Task never comes up.

### Designing the health check endpoint

Applications must provide a `/health` endpoint. **But a simple 200 OK is not enough.**

```javascript
const healthy = {
  db: false,
  cache: false,
};

app.get('/health', (req, res) => {
  if (dbConnected && cacheConnected) {
    return res.json({ status: 'ok' });
  }
  return res.status(503).json({ status: 'degraded', details: healthy });
});

app.get('/ready', (req, res) => {
  if (dbConnected && cacheConnected) {
    return res.json({ ready: true });
  }
  return res.status(503).json({ ready: false });
});
```

`/health` is for ALB health checks: if it can accept requests, OK.
`/ready` is for release decisions, only ready when all dependencies (DB, cache) are connected.

> **The health check endpoint is the only window the ECS scheduler has into your application.** If you half-ass it, the scheduler half-asses you back.

## Release strategy: rolling vs blue-green

ECS supports two main release strategies. **Which one to use depends on the scenario, not on "which is more advanced".**

### Rolling (default)

```
old old old  →  old old new  →  old new new  →  new new new
```

Pros: stable resource usage (at most doubled briefly), simple to implement.
Cons: during old + new coexistence, an incompatible DB schema change will break things.

**Best for**: routine iterations, backward-compatible changes.

### Blue-green

```
[Blue] old old old    [Green] new new new (standby)
        ↓ traffic cut ↓
[Blue] standby        [Green] new new new (live)
```

Pros: instant traffic cut, no old/new coexistence; instant rollback on issues.
Cons: requires double resources; DB connections drop during the cut.

**Best for**: major version upgrades, incompatible changes, scenarios needing a visible "pre-release" window.

We use CodeDeploy for blue-green:

```hcl
resource "aws_ecs_service" "user_service" {
  deployment_controller {
    type = "CODE_DEPLOY"
  }
}

resource "aws_codedeploy_app" "user_service" {
  name             = "user-service"
  compute_platform = "ECS"
}

resource "aws_codedeploy_deployment_group" "user_service" {
  app_name               = aws_codedeploy_app.user_service.name
  deployment_group_name  = "user-service-prod"
  service_role_arn       = aws_iam_role.codedeploy.arn
  deployment_config_name = "CodeDeployDefault.ECSAllAtOnce"

  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }

  alarm_configuration {
    alarms             = ["user-service-5xx-rate"]
    enabled            = true
    ignore_poll_alarm_failure = false
  }
}
```

**`auto_rollback_configuration` is the safety rope for blue-green.** Any deploy failure or CloudWatch alarm triggers automatic rollback to the old version, with no human needed. This single setting is what makes blue-green "dare to use".

## Service discovery: do not call by IP

Services calling each other **should not depend on IPs**. Containers are ephemeral; IPs change.

We use AWS Cloud Map + ECS Service Discovery:

```hcl
resource "aws_service_discovery_private_dns_namespace" "internal" {
  name = "internal.coreteam"
  vpc  = aws_vpc.main.id
}

resource "aws_service_discovery_service" "user_service" {
  name = "user-service"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.internal.id
    dns_records {
      type  = "A"
      ttl   = 10
    }
  }

  health_check_custom_config {
    failure_threshold = 1
  }
}

resource "aws_ecs_service" "user_service" {
  service_registries {
    registry_arn = aws_service_discovery_service.user_service.arn
    container_name = "app"
    container_port = 3000
  }
}
```

Effect: any service can reach user-service via the DNS name `user-service.internal.coreteam`. **DNS follows Task IP changes automatically; callers are unaware.**

## Capacity planning: cluster autoscaling

Finally, a chronic pain point: **capacity management of the ECS cluster itself**.

Under EC2 mode, the cluster has a finite number of EC2 instances. If all EC2s are full, new Tasks stay PENDING forever.

Our fix: **make the cluster autoscale**:

```hcl
resource "aws_ecs_capacity_provider" "main" {
  name = "main"

  auto_scaling_group_provider {
    auto_scaling_group_arn         = aws_autoscaling_group.ecs.arn
    managed_termination_protection = "ENABLED"

    managed_scaling {
      status                    = "ENABLED"
      target_capacity           = 75
      maximum_scaling_step_size = 10
      minimum_scaling_step_size = 1
    }
  }
}
```

**`target_capacity = 75`** means the cluster maintains 75% utilization; once EC2s hit 75%, scale out. The 25% buffer lets new Tasks schedule immediately without waiting for ASG to bring up new machines.

## Where docker-compose fits in all this

A word on docker-compose: it remains the **standard for local development** here, but it must align with the production ECS config.

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    image: user-service:latest
    build: .
    ports:
      - "3000:3000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
    environment:
      - NODE_ENV=development
      - LOG_LEVEL=debug
    depends_on:
      log-router:
        condition: service_started
  log-router:
    image: fluent-bit:1.8
```

This compose file and the Task Definition above are **aligned** on container list, ports, and health checks. After local debugging, deploying to ECS is just swapping the image tag and env vars, with no "works locally, dies in prod" awkwardness.

> **The value of docker-compose is not "replacing ECS", it is "making local look like production".** The more they resemble each other, the fewer surprises.

## Where this left us

By early 2021, CoreTeam's ECS stack was mature:

- Over a hundred services on a unified cluster, with capacity autoscaling.
- Rolling deploy by default, blue-green for major changes, auto-rollback as the safety net.
- Service discovery via Cloud Map, callers are blind to IP changes.
- Health check endpoints standardized, used by both ALB and ECS.

The biggest realization at this stage: **the "advanced skills" of container orchestration are often just understanding the basics well.** Task boundaries, network mode, health checks, capacity headroom; each is simple alone, and their composition is what makes a stable deployment system.

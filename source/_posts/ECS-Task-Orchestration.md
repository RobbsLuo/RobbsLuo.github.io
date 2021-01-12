---
title: 容器化部署深化：ECS 任务编排与发布策略
date: 2021-01-12 14:00:00
tags:
  - Career
  - AWS ECS
  - Docker
  - docker-compose
  - DevOps
categories:
  - Career
description: 从"能跑起来"到"跑得稳"还有一段距离。这篇聊 ECS 上 task 编排的细节——sidecar 设计、健康检查、蓝绿与滚动发布的选择、容量预留与服务发现。
---

## "能跑"和"跑得稳"是两回事

前面那篇 Jenkins + ECS 写的是怎么把服务"发上去"。但用过 ECS 的人都知道，把容器发上去只是第一步，真正难的是让它在生产里稳稳地跑。

到 2020 年底，我们 ECS 集群上跑着上百个服务。日常会遇到这些问题：

- 某个服务发布之后偶发性 502，过几分钟自己好了。
- 流量高峰时 ALB 把请求打到正在启动的容器上，连不上。
- 蓝绿发布切流量时数据库连接没释放干净，新旧版本同时写库出问题。
- 一个 sidecar 挂了导致主业务挂掉。

这些问题的共同点是，都不是"代码 bug"，而是部署架构没设计好。这篇聊的就是这一层的经验。

## Task Definition：把容器组合想清楚

ECS 里最小的调度单位不是单个容器，是 Task。一个 Task 可以包含多个容器，它们共享网络 namespace 和 storage volume。这跟 K8s 的 Pod 概念几乎一模一样。

设计 Task Definition 的时候，几个原则：

### 1. 主容器 + sidecar 的组合

主容器跑业务，sidecar 跑辅助功能。日志收集、监控 agent、服务网格 proxy 这类东西，应该放 sidecar，而不是塞进主容器。

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

注意几个细节：

- `essential: true` 只标在主容器上。sidecar 挂了不应该让整个 Task 一起挂，标 `false` 让它独立重启。
- `healthCheck` 一定要配。ECS 用这个判断容器是不是 ready，没配的话只能靠 ALB 的 health check 间接判断。
- `startPeriod: 30` 给应用启动留缓冲期，否则 Java 这种启动慢的应用会被误判为 unhealthy。

### 2. CPU 和内存的精细分配

ECS 的资源分配是硬限制。超了就被 OOM kill，没有商量。所以必须根据实际用量来配，而不是拍脑袋给个"够大"的值。

```json
{
  "cpu": "256",          // 0.25 vCPU
  "memory": "512",       // 512 MB
  "memoryReservation": "256"   // 软限制，给 docker --memory-reservation
}
```

`memory` 是硬限制，超了直接被杀；`memoryReservation` 是软限制，告诉 Docker "尽量别让它用超过这个"。软限制+硬限制的组合比单一硬限制更灵活，容器平时省着用，突发时能借用邻居的资源。

### 3. 网络模式选 awsvpc

ECS 有三种网络模式：bridge、host、awsvpc。生产环境我们一律用 awsvpc。

- bridge：Docker 默认的 NAT 模式，端口映射混乱，多 Task 抢端口。
- host：直接用宿主机网络，性能最好但隔离差。
- awsvpc：每个 Task 拿到一个 ENI，有独立内网 IP，安全组可以精确控制。

```hcl
resource "aws_security_group" "user_service" {
  name = "user-service-sg"
  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # 只让 ALB 访问
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

awsvpc 模式下，每个 Task 都能用安全组精细控制出入站规则，这是其他模式做不到的。

## ALB 集成：让流量切换可控

ECS Service 挂在 ALB 后面，发布时通过 ALB 的健康检查来决定流量切换。

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

  health_check_grace_period = 60   # 给应用启动留时间
}
```

`health_check_grace_period` 是一个被忽略但极重要的参数。它告诉 ALB："容器启动后这 60 秒内别做健康检查"。否则应用还没 ready，ALB 一检查就判定 unhealthy，导致新 Task 一直起不来。

### 健康检查端点的设计

应用必须提供一个 `/health` 端点。但简单的 200 OK 是不够的。

```javascript
// Node.js 健康检查的正确姿势
const healthy = {
  db: false,
  cache: false,
};

app.get('/health', (req, res) => {
  // 检查依赖
  if (dbConnected && cacheConnected) {
    return res.json({ status: 'ok' });
  }
  return res.status(503).json({ status: 'degraded', details: healthy });
});

// readiness check（更严格）
app.get('/ready', (req, res) => {
  if (dbConnected && cacheConnected) {
    return res.json({ ready: true });
  }
  return res.status(503).json({ ready: false });
});
```

`/health` 用于 ALB 健康检查，能接请求就算 OK。
`/ready` 用于发布判断，所有依赖（DB、cache）都连上才算 ready。

健康检查端点是 ECS 调度器了解你的应用的唯一窗口。它写得糊弄，调度器就糊弄你。

## 发布策略：滚动 vs 蓝绿

ECS 支持两种主要发布策略，选哪个看场景，不是看哪个"更先进"。

### 滚动发布（默认）

```
旧旧旧  →  旧旧新  →  旧新新  →  新新新
```

优点：资源占用稳定（最多 double 期间），实现简单。
缺点：新旧版本并存期间，如果有不兼容的数据库 schema 变更，会出事。

适合场景：常规迭代、向后兼容的变更。

### 蓝绿发布

```
[蓝组] 旧旧旧    [绿组] 新新新（待命）
        ↓ 流量切换 ↓
[蓝组] 待命      [绿组] 新新新（在线）
```

优点：流量瞬间切换，没有新旧并存；出问题瞬间回滚。
缺点：需要双倍资源，切流量期间数据库连接会断。

适合场景：重大版本升级、不兼容变更、需要可观测的"准发布"窗口。

我们用 CodeDeploy 来做蓝绿：

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

`auto_rollback_configuration` 是蓝绿发布的保险绳。任何部署失败或 CloudWatch 告警触发，自动回滚到旧版本，不需要人介入。这一条让蓝绿发布变成"敢用"的策略。

## 服务发现：别用 IP 直接调

服务之间互相调用，不应该依赖 IP。容器是临时的，IP 会变。

我们用 AWS Cloud Map + ECS Service Discovery：

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

效果是，任何服务都能通过 `user-service.internal.coreteam` 这个 DNS 名访问到 user-service。DNS 自动跟随 Task 的 IP 变化，调用方无感。

## 容量规划：cluster autoscaling

最后说一个痛点，ECS 集群本身的容量管理。

EC2 模式下，cluster 里的 EC2 实例数量是有限的。如果所有 EC2 都满了，新 Task 一直处于 PENDING 状态，永远起不来。

我们的解法是让 cluster 自动扩容：

```hcl
# ECS cluster capacity provider
resource "aws_ecs_capacity_provider" "main" {
  name = "main"

  auto_scaling_group_provider {
    auto_scaling_group_arn         = aws_autoscaling_group.ecs.arn
    managed_termination_protection = "ENABLED"

    managed_scaling {
      status                    = "ENABLED"
      target_capacity           = 75   # 维持 75% 容量水位
      maximum_scaling_step_size = 10
      minimum_scaling_step_size = 1
    }
  }
}
```

`target_capacity = 75` 意思是 cluster 维持 75% 的水位，EC2 用到 75% 就触发扩容。留 25% buffer 让新 Task 能立即调度，不用等 ASG 拉起新机器。

## docker-compose 在这之中的角色

最后说一下 docker-compose，它在这套体系里仍然是本地开发的标配，但和生产 ECS 配置需要对齐。

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

这个 compose 文件和上面的 Task Definition 在容器列表、端口、健康检查上保持一致。开发本地调通后，部署到 ECS 只需要换 image tag 和环境变量，不会出"本地能跑生产跑不了"的尴尬。

docker-compose 的价值不是替代 ECS，而是让本地环境长得像生产环境。越像，问题越少。

## 走到这里

到 2021 年初，CoreTeam 的 ECS 这套已经相当成熟：

- 上百个服务跑在统一的 cluster 上，容量自动伸缩。
- 滚动发布是默认，蓝绿用于重大变更，自动回滚兜底。
- 服务发现通过 Cloud Map，调用方零感知 IP 变化。
- 健康检查 endpoint 是标准约定，ALB 和 ECS 都能用上。

这一阶段最大的体会是，容器编排的"高级技巧"，往往是把基本概念理解到位。Task 边界、网络模式、健康检查、容量水位，每一项单看都不复杂，组合起来才是稳定的部署体系。

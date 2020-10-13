---
title: 基建即代码：用 Terraform 编排 AWS
date: 2020-10-13 10:30:00
tags:
  - Career
  - Terraform
  - AWS
  - DevOps
  - IaC
categories:
  - Career
description: 2018 年用 Terraform 只是管住 VPC 和 IAM，到 2020 年这条路越走越深——module 切分、多环境隔离、state drift 检测、workspaces 实战。这篇是两年 IaC 实践的复盘。
---

## 从"管住"到"编排"

前面写过 2018 年我们用 Terraform 把 AWS 资源统一管理起来的事。那篇文章发出去之后，有人留言问，不就是写几个 .tf 文件吗，有这么玄乎？

我理解这种印象。Terraform 入门确实不难，照着官方教程两小时就能拉起一个 VPC。但真正难的不是"能跑起来"，而是两年后还能不能跑得稳。

到 2020 年下半年，我们的 Terraform 仓库规模大概是：

- 8 个 AWS 账号（dev/staging/prod 各一套 + 沙盒 + 安全审计）。
- 上百个 module 实例。
- state 文件几十个，加起来代表了几千个 AWS 资源。

到这个规模，问题就从"会不会写 HCL"变成怎么让这些东西不失控。这篇聊的就是这一层。

## Module 切分：粒度是个艺术

Terraform module 这件事，粒度太粗等于没切，粒度太细等于自虐。

我们试过几种粒度，最后沉淀下来的原则是：

### 1. 资源生命周期一致的，放一个 module

比如 VPC + 子网 + 路由表 + IGW + NAT Gateway，这一组东西生命周期完全绑定，要么一起创建，要么一起销毁。把它们放一个 module，调用方一次调用就拿到一个完整的网络栈。

```hcl
# modules/vpc/variables.tf
variable "cidr"              { type = string }
variable "project"           { type = string }
variable "env"               { type = string }
variable "azs"               { type = list(string) }
variable "public_subnets"    { type = list(string) }
variable "private_subnets"   { type = list(string) }
variable "enable_nat"        { type = bool, default = true }

# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${var.project}-${var.env}-vpc" })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(local.common_tags, { Name = "${var.project}-${var.env}-igw" })
}

resource "aws_nat_gateway" "this" {
  count               = var.enable_nat ? length(var.azs) : 0
  allocation_id       = aws_eip.nat[count.index].id
  subnet_id           = aws_subnet.public[count.index].id
  tags                = merge(local.common_tags, { Name = "${var.project}-${var.env}-nat-${count.index}" })
  depends_on          = [aws_internet_gateway.this]
}
```

### 2. 业务方不直接调底层 module

业务团队不应该自己组装 VPC + RDS + ECS 这种组合。我们提供"服务级 module"，一个 module 拉起一整套业务所需的基础设施：

```hcl
# 业务方写这样就够了
module "user_service" {
  source = "git::https://gitlab.internal/infra/modules//service-stack?ref=v2.1.0"

  service_name = "user-service"
  env          = "prod"
  port         = 3000
  desired      = 3
  db_required  = true
  db_size      = "db.t3.medium"
}
```

这个 `service-stack` module 内部组合了 VPC 引用、ALB listener rule、ECS service、CloudWatch alarm、IAM role，业务方完全不需要知道这些细节。

好的 module 抽象让业务方不需要懂就能用对，坏的抽象让业务方必须懂才能用，那就是没抽象。

### 3. 别把所有东西塞一个 module

我们见过一些团队把整个生产环境塞到一个巨型 module 里，几百个 resource，`terraform plan` 跑五分钟。这种做法的问题：

- plan 输出几千行，没人能 review。
- 一个小改动可能引发连锁反应。
- state 文件巨大，操作慢且容易出错。

module 的边界应该沿着独立的部署单元切。一个微服务一个目录，一个共享平台一个目录。

## 多环境：用 workspaces 还是目录分开？

Terraform 多环境管理是个老话题。主流方案两种：

**方案 A：terraform workspaces**
```bash
terraform workspace new dev
terraform workspace new prod
terraform apply -var-file="environments/$(terraform workspace show).tfvars"
```

**方案 B：目录物理分开**
```
environments/
├── dev/
│   └── main.tf
├── staging/
│   └── main.tf
└── prod/
    └── main.tf
```

我们最后选了**方案 B 的变体——目录分开，但共享 module**：

```
infra/
├── modules/                    # 共享 module
│   ├── vpc/
│   ├── service-stack/
│   └── ...
└── environments/
    ├── dev/
    │   ├── main.tf             # 调用 module
    │   ├── backend.tf          # 独立 state
    │   └── terraform.tfvars
    ├── staging/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf
        └── terraform.tfvars
```

为什么不用 workspaces？因为 workspaces 让多个环境共享同一份 .tf 配置，看似 DRY，实则危险：

- workspace 切错了一个 `terraform destroy` 就是灾难。
- 不同环境的资源拓扑往往不同（dev 不需要 NAT，prod 需要），共享配置会塞满 `count = var.env == "prod" ? 1 : 0` 这种丑陋代码。
- state 隔离不彻底，初学者容易踩坑。

目录分开的好处是 dev 的 .tf 文件和 prod 的 .tf 文件物理隔离，误操作的半径被限制在当前目录。代价是有一些重复代码，但通过 module 复用已经把重复降到很低。

```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"

  cidr            = "10.10.0.0/16"
  project         = "coreteam"
  env             = "prod"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnets  = ["10.10.1.0/24", "10.10.2.0/24", "10.10.3.0/24"]
  private_subnets = ["10.10.101.0/24", "10.10.102.0/24", "10.10.103.0/24"]
  enable_nat      = true
}
```

prod 这个目录里只有"prod 专属的配置"，逻辑全在 module 里。

## State 管理：drift 检测是必修课

跑了一两年 Terraform 之后，state drift 几乎一定会发生。原因无非几种：

- 有人在 Console 手动改了东西（违反规矩但就是会发生）。
- 某个 AWS 自动修复改了资源属性。
- Terraform 升级版本时 provider 行为变化。

drift 不检测，下次 `terraform apply` 可能就是惊喜，而且不是好的那种。

我们用一套定时任务做 drift 检测：

```bash
#!/usr/bin/env bash
# scripts/drift-check.sh
set -euo pipefail

ENV=$1
WORKDIR="environments/${ENV}"

cd "$WORKDIR"
terraform init -backend=false

# plan 到一个文件，不实际改动
terraform plan -detailed-exitcode -out=/tmp/plan.bin > /tmp/plan.txt
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
  echo "✅ ${ENV}: no drift"
elif [ $EXIT_CODE -eq 1 ]; then
  echo "❌ ${ENV}: plan error"
  cat /tmp/plan.txt
  exit 1
elif [ $EXIT_CODE -eq 2 ]; then
  echo "⚠️  ${ENV}: drift detected"
  cat /tmp/plan.txt
  # 发 Slack 告警
  curl -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"Terraform drift in ${ENV}!\"}"
fi
```

`-detailed-exitcode` 这个 flag 是关键：返回 0 表示无变更，2 表示有变更（即 drift），1 表示出错。这个脚本每天跑一次，任何 drift 当天就能发现。

发现 drift 之后怎么办？我的原则是先搞清楚为什么 drift，而不是马上 apply 抹平。如果是有人手动改的，要么补回 Terraform，要么把这次改动纳入 Terraform（写进 .tf 文件）。

state drift 的本质是现实与代码不一致。解决它有两步：先纠正现实（或代码），再让两者对齐。盲目 apply 是赌博。

## 多账号策略

最后聊一下多账号。dev 和 prod 共享一个 AWS 账号是灾难，dev 一个失控的脚本能把 prod 的数据库删了。

我们用 AWS Organizations + Terraform 管理多账号：

```hcl
# modules/organization/main.tf (简化)
resource "aws_organizations_account" "dev" {
  name      = "coreteam-dev"
  email     = "aws+dev@coreteam.internal"
  parent_id = aws_organizations_organizational_unit.platform.id
}

resource "aws_organizations_account" "prod" {
  name      = "coreteam-prod"
  email     = "aws+prod@coreteam.internal"
  parent_id = aws_organizations_organizational_unit.platform.id
}

# 用 SCP 限制 dev 账号不能创建某些危险资源
resource "aws_organizations_policy" "dev_guardrails" {
  name = "dev-guardrails"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Deny"
        Action   = ["iam:DeleteRole", "rds:DeleteDBCluster"]
        Resource = "*"
      }
    ]
  })
}
```

每个账号有独立的 state、独立的 IAM、独立的 CloudTrail。物理隔离加上 SCP 兜底，比任何策略逻辑都靠谱。

## 一些血泪经验

1. **Terraform 版本锁定**。`.terraform-version` 文件写死版本，团队所有人用同一个版本。版本不一致导致 state format 不兼容的事我们踩过。
2. provider 版本也要锁。`version = "~> 3.0"` 比 `version = ">= 3.0"` 安全。
3. state 加密必须开。`encrypt = true` 在 backend 配置里，没开的话 state 里的密钥是明文存 S3。
4. 大改动分步走。一次 PR 别动几十个 resource，分几个 PR 逐步推进，每个 PR 都能独立 plan/apply。
5. 永远先 plan 再 apply。生产环境禁止 `terraform apply -auto-approve`，必须人眼 review plan 输出。

## 走到这一步

2020 年底回头看，Terraform 在我们这里已经不是"管 AWS 资源的工具"，而是整个基建的真理来源。任何 AWS 资源的状态，都以 Terraform 为准；任何变更，都必须通过 Terraform PR。

这条路走通的关键不是我们用了 Terraform，而是我们把 Terraform 当成基建的宪法来执行。工具有，执行力才是分水岭。

下一篇继续 ECS 的话题，任务编排和发布策略，越往深走越有意思。

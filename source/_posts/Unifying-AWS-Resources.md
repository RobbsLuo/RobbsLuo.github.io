---
title: 运维平台起步：把 AWS 资源统一管起来
date: 2018-07-09 09:30:00
tags:
  - Career
  - AWS
  - Terraform
  - DevOps
categories:
  - Career
description: 2018 年刚到 CoreTeam 时面对的是一摊没人说得清的 AWS 资源，这篇记录我们是怎么把账上的 EC2、RDS、S3 一件件盘点清楚，并用 Terraform 拉起第一版基建的。
---

## 接手时的烂摊子

2018 年年中我到 CoreTeam 接手内部运维平台那块。第一天大佬带我过环境，打开 AWS Console 一看，我当时人就麻了。

光 EC2 实例就两百多台，散在四五个 Region 里。有的挂着两年前的标签写着"测试用，可删"，结果一停业务那边就报警。RDS 实例大得离谱的也有，跑着低得不能再低的连接数。S3 bucket 命名风格五花八门，有的是 `coreteam-test`，有的是 `coreteam-prod-backup-2017`，还有直接拿同事英文名命名的。

所谓的"云上自由"，本质上是没人兜底的失控。每个团队都能随手开资源，开完没人管，账单月底一拉全员沉默。

更麻烦的是没人能给出一份完整的资源清单。这事儿不是某个人失职，是组织层面的痼疾，业务团队来回换人，谁开的资源、为什么开的、跑的什么业务，早就成了一笔糊涂账。

## 第一步：盘点，而不是重构

很多人遇到这种局面第一反应是"重构一把"，上 K8s，全容器化，搞个牛逼的平台。我建议压住这种冲动。

资源都没盘清，谈什么平台化？你不知道哪些是僵尸、哪些是关键路径，重构就是赌命。

我们花了两周时间做了一件极其枯燥的事，资源盘点。具体做了几件事：

1. 给所有 EC2 打标签（`Owner`、`Project`、`Environment`、`CostCenter`），没标签的全网通报一周后停机。
2. 把所有 RDS 实例的连接数和 IO 拉了一周的数据，看哪些是空跑的。
3. S3 bucket 一律加 lifecycle policy，老 bucket 按生命周期转 Glacier。
4. 跑一遍 AWS Cost Explorer，按 tag 拆账单，看钱花哪儿了。

打标签这事听着 low，但没有标签就没有可观测性，后面做什么都是瞎的。

AWS 里给资源打标签的命令大概长这样：

```bash
# 批量给 EC2 打标签
aws ec2 create-tags \
  --resources i-0abc123def456 \
  --tags \
    Key=Project,Value=CoreTeam \
    Key=Owner,Value=platform \
    Key=Environment,Value=prod

# 查某个 Region 下所有没打 Project 标签的实例
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?!Tags[?Key==`Project`]]' \
  --output table
```

就这种命令，我们写了一堆脚本循环跑。两周之后，账单能按团队拆出大概了，僵尸资源清理掉一批，月度账单直接下来一截。

## 第二步：选 Terraform，不是 CloudFormation

盘点完，下一步就是怎么防止这种事再发生。

其实 AWS 自己有 CloudFormation，写 JSON/YAML 就能把资源管理起来。但我们选了 Terraform，原因很实在：

1. 跨云中性。当时公司虽然主要在 AWS，但已经在谈 GCP 的合作。CloudFormation 只认 AWS，未来切云就废一半。
2. 语法顺眼。HCL 比 CloudFormation 那一坨 JSON 友好太多，团队上手快。
3. 社区 module 多，AWS provider 覆盖几乎全部资源。
4. state 文件在外部，可以集中管理，多人协作。

工具选型这件事，我有一个朴素的原则：选"够用且未来不会被绑死"的，而不是"看起来最强大"的。CloudFormation 功能没问题，但绑死 AWS 这一点就足以否决它。

第一版 Terraform 目录结构大概是这样：

```
infra/
├── providers.tf          # AWS provider 配置
├── backend.tf            # state 存到 S3 + DynamoDB 加锁
├── vpc/                  # 网络层
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── ec2/                  # 计算层
├── rds/                  # 数据库
├── s3/                   # 对象存储
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

state 后端必须放在 S3 上，并且一定要配 DynamoDB 加锁，不然两个人同时 `terraform apply` 就能炸给你看：

```hcl
terraform {
  backend "s3" {
    bucket         = "coreteam-terraform-state"
    key            = "infra/global.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

## 第一个 module：VPC

我们没有上来就堆 module，第一个落地的就是 VPC，因为这是所有东西的底座，复用价值最高。

一个标准的 VPC module 长这样（简化版）：

```hcl
# vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project}-${var.env}"
    Project     = var.project
    Environment = var.env
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index]
  tags = { Name = "${var.project}-${var.env}-public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]
  tags = { Name = "${var.project}-${var.env}-private-${count.index}" }
}
```

这个 module 后来在整个团队里被复用了几十次，每开一个新业务直接 `module "vpc"` 就能拉起一套标准化网络。

## 踩过的坑

不是说用了 Terraform 就万事大吉。第一个月我们踩了一个挺常见的坑：state drift。

有人为了赶时间，在 Console 里手动改了一个安全组的端口范围，没走 Terraform。下次 `terraform plan` 一跑，说要销毁这个安全组重建，吓得那人赶紧跑来找我。

我们后来的规矩是生产环境禁止任何 Console 修改，所有变更必须走 Terraform PR review。这是流程问题，不是工具问题。工具管不住手贱的人，流程能。

基建即代码的核心从来不是"能不能用代码定义资源"，而是敢不敢让人随便改。Terraform 给你的是信心，不是免疫。

## 这一步走完之后

到 2018 年 7 月底，我们做到了几件事：

- AWS 资源清单 100% 可枚举，标签覆盖 95% 以上。
- VPC、IAM、S3 这些公共底座全部 Terraform 化。
- 月度账单能按团队/项目拆出来给老板看。
- 任何新增资源必须走 Terraform，否则审计能查到。

这只是个开始。资源管起来，下一步就是怎么把服务跑上去，这是后面 Jenkins + ECS 那条线要解决的问题。

回头看，这一步其实没做什么"牛逼"的事，全是脏活累活。但内部系统的路就是这样，没有起点就没有方向。把地基打扎实了，后面 ECS、CI/CD、监控才有得谈。

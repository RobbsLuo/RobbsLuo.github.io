---
title: Unifying AWS Resources at Scale
date: 2018-07-09 09:30:00
tags: [Career, AWS, Terraform, DevOps]
categories:
  - Career
description: When I joined CoreTeam in mid-2018, I inherited a pile of AWS resources nobody could fully account for. This is how we inventoried every EC2, RDS and S3 bucket and brought up the first version of our infrastructure with Terraform.
lang: en
---

## The mess I inherited

Mid-2018, I joined CoreTeam to take over the internal ops platform. Day one, my lead walked me through the environment. He opened the AWS Console and I just went cold.

More than two hundred EC2 instances, scattered across four or five regions. Some carried tags from two years ago saying "test, can be deleted". Stop one, and the business side would immediately start screaming. There were RDS instances absurdly oversized, sitting there with connection counts you could count on one hand. S3 buckets had naming styles all over the map: `coreteam-test`, `coreteam-prod-backup-2017`, and even a few named after coworkers.

> **The so-called "freedom of the cloud" is really just ungoverned chaos. Every team can spin up resources on a whim, but nobody owns them, and the bill at month-end leaves everyone silent.**

Worse, nobody could produce a complete inventory. Not because one particular person was negligent; it was organizational rot. Teams rotated, people left, and the reasoning behind every resource became folklore.

## Step one: inventory, not rewrite

The first reaction of many engineers in this situation is to "rewrite everything": go full Kubernetes, containerize the world, build a flashy platform. I would push back on that impulse.

**If you don't even know what your resources are, what platform are you building?** You don't know which are zombies and which are load-bearing. Rewriting under those conditions is just gambling.

So we spent two weeks doing something deeply tedious: a full inventory. Concretely:

1. Tag every EC2 (`Owner`, `Project`, `Environment`, `CostCenter`). Anything untagged got announced company-wide, then shut down a week later.
2. Pull a week of connection and IO metrics for every RDS instance, looking for the idle ones.
3. Apply lifecycle policies to every S3 bucket. Legacy buckets transitioned to Glacier on a schedule.
4. Run AWS Cost Explorer, split the bill by tag, see where the money was going.

Tagging sounds low-brow, but **without tags there is no observability, and without observability everything after that is blind**.

Tagging resources in AWS looks roughly like this:

```bash
# Tag an EC2 instance in bulk
aws ec2 create-tags \
  --resources i-0abc123def456 \
  --tags \
    Key=Project,Value=CoreTeam \
    Key=Owner,Value=platform \
    Key=Environment,Value=prod

# List every instance in a region missing the Project tag
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?!Tags[?Key==`Project`]]' \
  --output table
```

We wrote a pile of scripts around commands like these and ran them in loops. After two weeks, the bill was attributable to teams, a chunk of zombies had been cleaned up, and the monthly number visibly dropped.

## Step two: Terraform, not CloudFormation

With the inventory done, the next question was how to prevent the rot from coming back.

AWS has CloudFormation: write JSON or YAML and it manages resources for you. **We picked Terraform instead**, for very practical reasons:

1. **Cloud-agnostic.** The company was mostly on AWS, but GCP conversations were already happening. CloudFormation only speaks AWS; switching clouds later would throw half of it away.
2. **Pleasant syntax.** HCL is far friendlier than CloudFormation's wall of JSON. The team got up to speed fast.
3. **Mature ecosystem.** Community modules everywhere, and the AWS provider covers nearly every resource type.
4. **External state.** State lives where you want it. You can centralize and share it across the team.

> **On tool selection I have a plain rule: pick "good enough and not locking you in", not "looks the most powerful."** CloudFormation works fine functionally, but the AWS lock-in alone was enough to veto it.

Our first Terraform layout looked roughly like this:

```
infra/
├── providers.tf          # AWS provider config
├── backend.tf            # state to S3 + DynamoDB lock
├── vpc/                  # networking
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── ec2/                  # compute
├── rds/                  # databases
├── s3/                   # object storage
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

The state backend has to live on S3, and **you must enable DynamoDB locking**, otherwise two people running `terraform apply` simultaneously will blow things up:

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

## The first module: VPC

We did not rush into piling up modules. The first one to land was VPC, because it is the foundation of everything, and reuse value was highest.

A standard VPC module looks roughly like (simplified):

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

This module got reused dozens of times across teams. Spinning up a new stack for a new service was just `module "vpc"` away.

## Pitfalls

Terraform is not magic. The first month we hit a classic: **state drift**.

Someone in a hurry changed a security group port range directly in the Console, bypassing Terraform. Next `terraform plan` cheerfully announced it would destroy and recreate that security group. That person ran over to me in a panic.

The rule we eventually enforced: **no Console changes in production. Every change goes through a Terraform PR with review.** This is a process problem, not a tooling problem. Tools cannot stop someone determined to click. Process can.

> **The hardest part of infrastructure as code is trusting people to change resources freely without losing control.** Terraform gives you confidence, not immunity.

## Where this left us

By the end of July 2018 we had:

- 100% enumerable AWS resources, with 95%+ tag coverage.
- Common foundations (VPC, IAM, S3) fully Terraformed.
- A monthly bill that could be split by team and project for leadership.
- A rule that any new resource had to go through Terraform, or audit would catch it.

That was just the start. **Resources are tamed; the next question is how to run services on top of them**, which is what the Jenkins + ECS track tackled next.

In hindsight, nothing we did here was flashy. It was all grunt work. But that is how internal systems go: **without a starting point there is no direction**. Get the foundation solid, and everything afterward, whether ECS, CI/CD, or monitoring, has a place to stand.

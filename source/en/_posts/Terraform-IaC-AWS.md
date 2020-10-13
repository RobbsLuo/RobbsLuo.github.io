---
title: Infrastructure as Code with Terraform on AWS
date: 2020-10-13 10:30:00
tags: [Career, Terraform, AWS, DevOps, IaC]
categories:
  - Career
description: In 2018 Terraform just managed our VPC and IAM. By 2020 the road had gone much deeper — module partitioning, multi-environment isolation, state drift detection, workspaces in practice. This is a retrospective on two years of IaC.
lang: en
---

## From "managing" to "orchestrating"

Earlier I wrote about how we used Terraform in 2018 to unify AWS resource management. Someone commented on that post: **"Isn't it just writing a few .tf files? What's the big deal?"**

I get that reaction. Terraform basics are genuinely easy. Following the official tutorial you can stand up a VPC in two hours. But **the hard part is not "can it run", it is "can it still run cleanly two years later".**

By the second half of 2020, our Terraform repo had roughly:

- 8 AWS accounts (dev/staging/prod sets + sandbox + security audit).
- Hundreds of module instances.
- Dozens of state files, collectively representing thousands of AWS resources.

At this scale the problem shifts from "do you know HCL" to "**how do you keep this from spiraling out of control**". That is what this post is about.

## Module partitioning: granularity is an art

With Terraform modules, **too coarse means no partitioning at all, too fine means self-torture**.

We tried several granularities. The principles we settled on:

### 1. Resources with aligned lifecycles go in one module

VPC + subnets + route tables + IGW + NAT Gateway: these are **lifecycle-bound**. You create them together or destroy them together. Putting them in one module lets the caller spin up a whole network stack with one invocation.

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

### 2. Business teams do not call low-level modules directly

Business teams should not assemble VPC + RDS + ECS combinations themselves. **We provide "service-level modules"** that bring up an entire stack a service needs:

```hcl
# This is all the business team writes
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

This `service-stack` module composes VPC references, ALB listener rules, ECS service, CloudWatch alarms, IAM roles internally, so the business team does not need to know any of it.

> **A module is fundamentally an abstraction. A good abstraction lets the user "not need to understand" and still use it correctly. A bad abstraction forces the user to "understand before using", which means it is not an abstraction at all.**

### 3. Do not stuff everything into one module

We have seen teams cram an entire production environment into one giant module with hundreds of resources, where `terraform plan` takes five minutes. Problems with this:

- Plan output runs to thousands of lines, impossible to review.
- A small change can trigger cascading effects.
- The state file is huge, slow to operate on, and error-prone.

**Module boundaries should follow "independent deployable units".** One microservice per directory, one shared platform per directory.

## Multi-environment: workspaces or split directories?

Multi-environment management is an old Terraform topic. Two mainstream approaches:

**Approach A: terraform workspaces**
```bash
terraform workspace new dev
terraform workspace new prod
terraform apply -var-file="environments/$(terraform workspace show).tfvars"
```

**Approach B: physical directory split**
```
environments/
├── dev/
│   └── main.tf
├── staging/
│   └── main.tf
└── prod/
    └── main.tf
```

We went with a **variant of Approach B: split directories, shared modules**:

```
infra/
├── modules/                    # shared modules
│   ├── vpc/
│   ├── service-stack/
│   └── ...
└── environments/
    ├── dev/
    │   ├── main.tf             # calls modules
    │   ├── backend.tf          # independent state
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

Why not workspaces? Because **workspaces have multiple environments share the same .tf config, which looks DRY but is dangerous**:

- Switch to the wrong workspace and `terraform destroy` is a catastrophe.
- Resource topology differs across environments (dev does not need NAT, prod does); shared config fills up with ugly `count = var.env == "prod" ? 1 : 0`.
- State isolation is not thorough; beginners step on landmines.

**Directory split benefits**: dev's .tf files and prod's .tf files are physically isolated, **and the blast radius of a mistake is contained to the current directory**. The cost is some duplication, but module reuse keeps it minimal.

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

The prod directory contains only "prod-specific config"; all the logic lives in modules.

## State: drift detection is mandatory

After running Terraform for a year or two, **state drift will happen, almost certainly**. Causes are typically:

- Someone changed something manually in the Console (against the rules, but it happens).
- An AWS auto-remediation changed a resource attribute.
- Provider behavior changed on a Terraform version upgrade.

Without drift detection, the next `terraform apply` is a surprise, and not a good one.

We run drift detection on a schedule:

```bash
#!/usr/bin/env bash
# scripts/drift-check.sh
set -euo pipefail

ENV=$1
WORKDIR="environments/${ENV}"

cd "$WORKDIR"
terraform init -backend=false

# plan to a file, do not actually apply
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
  curl -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"Terraform drift in ${ENV}!\"}"
fi
```

The `-detailed-exitcode` flag is the key: 0 means no change, 2 means change (drift), 1 means error. This script runs daily, and **any drift is caught the same day**.

What to do after detecting drift? **Do not blindly apply to smooth it over; first figure out why the drift happened.** If someone changed it manually, either revert or fold the change into Terraform (write it into the .tf files).

> **State drift is, in essence, "reality disagreeing with code". Fixing it has two steps: first correct reality (or code), then bring them back into alignment. Blindly applying is gambling.**

## Multi-account strategy

Finally, multi-account. **Sharing one AWS account between dev and prod is a disaster**: an out-of-control dev script can take down the prod database.

We use AWS Organizations + Terraform for multi-account management:

```hcl
# modules/organization/main.tf (simplified)
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

# SCP restricts dev account from certain destructive actions
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

Each account has independent state, IAM, and CloudTrail. **Physical isolation + SCP as a guardrail** beats any policy-logic isolation.

## Lessons learned the hard way

1. **Pin Terraform versions**. A `.terraform-version` file locks the version for the whole team. We have been bitten by state-format incompatibility from version mismatch.
2. **Pin provider versions too**. `version = "~> 3.0"` is safer than `version = ">= 3.0"`.
3. **State encryption must be on**. `encrypt = true` in the backend config; without it, secrets in state are plaintext on S3.
4. **Take big changes in steps**. One PR should not touch dozens of resources. Split into several PRs, each independently plan-able and apply-able.
5. **Always plan before apply**. `terraform apply -auto-approve` is banned in production; plan output must be human-reviewed.

## Where this left us

Looking back at end of 2020, Terraform was no longer "a tool that manages AWS resources"; it was **the source of truth for the entire infrastructure**. Any AWS resource's state is defined by Terraform; any change must go through a Terraform PR.

What made this work was treating Terraform as the constitution of the infrastructure, and enforcing it. The tool exists; execution is the dividing line.

Next post: back to ECS, task orchestration and release strategy. The deeper you go, the more interesting it gets.

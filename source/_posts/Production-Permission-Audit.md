---
title: 权限审计：谁能动生产环境
date: 2019-02-12 11:00:00
tags:
  - Career
  - AWS
  - Security
  - IAM
categories:
  - Career
description: 运维平台上线半年后，我们发现能登生产的人比想象中多得多。这篇记一次彻底的权限审计——IAM 策略梳理、SSH 通道收紧、操作可追溯的全过程。
---

## 一次有惊无险的发现

2019 年初，发生了一件让我后背发凉的事。

某天一个离职同事的账号还活着，被外部扫描器扫到了，差点酿成事故。事后我们做了个统计，有 write 权限访问生产 AWS 资源的人，比团队花名册还多。离职的、调岗的、临时帮忙的、外包的，权限给出去容易收回来难，时间一长就成了一个没人说得清的大杂烩。

权限管理真正的风险不在授权，而在不回收。给出去的时候都有正当理由，回收的时候没人记得。

我们立刻决定：做一次彻底的权限审计，逐人、逐账号、逐策略地过。

## 审计的三个维度

权限这事不是只看"谁能登录"，至少涉及三个层面：

1. **AWS 资源层**：谁能动 EC2、RDS、S3、Lambda 这些。
2. **系统访问层**：谁能 SSH 上服务器。
3. **操作可追溯**：万一出事，能不能查到谁干的。

下面挨个说我们是怎么做的。

## AWS IAM：最小权限原则

先说 AWS 这一层。审计之前的状态：

- 有几个共享的 IAM User，密钥到处分发。
- 不少 `AdministratorAccess` 策略，理由是"图方便"。
- Root 账号的 access key 居然还活着。

整改第一刀就砍这三件事。

### 1. 砍掉所有共享账号

共享 IAM User 是反模式。任何"方便"换来的代价都是审计断裂，你永远不知道某个 API 调用是谁触发的。

我们要求：每个人用自己的 IAM User，开启 MFA（强制），密钥三个月轮换一次。共享场景一律走 IAM Role + STS 临时凭证。

```bash
# 强制 MFA 的策略片段
aws iam create-account-alias --account-alias coreteam

# 给用户挂上"必须用 MFA 才能操作"的策略
aws iam put-user-policy \
  --user-name robbs \
  --policy-name RequireMFA \
  --policy-document file://require-mfa.json
```

`require-mfa.json` 内容：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

这条策略的效果是：不挂 MFA 啥都干不了。强制每个人配 MFA，没有例外。

### 2. 干掉 AdministratorAccess

`AdministratorAccess` 是个甜蜜的陷阱。开发说"我需要动 S3"，运维图省事直接给 admin。这事儿短期方便，长期是定时炸弹。

我们把权限按角色拆开：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::coreteam-*",
        "arn:aws:s3:::coreteam-*/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "ap-southeast-1"]
        }
      }
    }
  ]
}
```

几个原则：

- **只能动自己负责的资源**：用 `coreteam-${project}-*` 这样的资源命名约束。
- **限制 Region**：防止误操作开到不认识的 Region。
- **只能读，除非明确需要写**：默认 read-only，写权限单独申请。
- **环境隔离**：dev/staging 账号和 prod 账号物理分开，不靠策略逻辑隔离。

### 3. Root 账号关进笼子

Root 账号的 access key 直接删掉。Root 账号的密码放进密码管理器，日常谁都不用，只在紧急情况下走流程取用。MFA 绑到硬件 token 上，锁柜子里。

Root 账号是个核武器。它存在的意义不是被使用，是被保管。

## SSH 通道：别让人直连服务器

EC2 上的 SSH 访问是另一个重灾区。最早大家用同一个 PEM key，几十台机器用同一把钥匙，谁能登录全靠口口相传。

整改方案：

### 1. SSH key 个人化

每个人用自己的 SSH key，不再共享 PEM。离职的时候只要删掉他的 key 就行，不用换全网。

```bash
# 给 EC2 注入用户 SSH key 的标准做法
# 通过 user-data 或 SSM 实现
#!/bin/bash
# 拉取当前团队成员的公钥
aws s3 cp s3://coreteam-ssh-keys/robbs.pub /tmp/
useradd -m -s /bin/bash robbs
mkdir -p /home/robbs/.ssh
mv /tmp/robbs.pub /home/robbs/.ssh/authorized_keys
chown -R robbs:robbs /home/robbs/.ssh
chmod 700 /home/robbs/.ssh
chmod 600 /home/robbs/.ssh/authorized_keys
```

公钥统一存在 S3 上，人员变动时新机器自动拉取最新的 key 列表。

### 2. 用 Session Manager 替代 SSH

更好的方案是根本不开放 22 端口。AWS Systems Manager Session Manager 提供了不依赖 SSH 的 shell 访问，所有操作走 IAM 鉴权，所有命令被 CloudTrail 记录。

```bash
# 通过 Session Manager 登录 EC2
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartPortForwardingSession
```

这个方案的好处：

- 安全组里直接关掉 22 端口，外部攻击面归零。
- 不需要管 PEM key，IAM 权限说了算。
- 所有会话被记录，事后可追溯。

但实话说，Session Manager 当时响应速度比直连 SSH 慢半秒到一秒，开发体验差一点。我们作为生产运维通道推，开发日常调试仍然允许 SSH，但必须个人 key + 跳板机。

### 3. 生产跳板机（bastion）

直接登生产 EC2 这件事被收紧了。所有访问必须通过一台跳板机，跳板机上：

- 强制 MFA 二次验证。
- 记录所有命令（用 `tlog` 或 auditd）。
- 限制可访问的目标机器白名单。

```hcl
# Terraform: 跳板机的安全组只放给特定网段
resource "aws_security_group" "bastion" {
  name = "bastion-sg"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]   # 只能从内网访问
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## 操作可追溯：CloudTrail 是底线

权限收紧之后，下一步是确保任何操作都能被追溯。AWS CloudTrail 是这事的基础。

```hcl
# Terraform: 启用 CloudTrail，记录所有 API 调用
resource "aws_cloudtrail" "main" {
  name                          = "coreteam-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_logging                = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }

  depends_on = [aws_s3_bucket_policy.cloudtrail]
}
```

关键配置：

- multi-region trail：不要只记一个 Region，否则 attacker 在别的 Region 操作你看不到。
- log 文件做 immutable：S3 bucket 上 bucket policy 禁止删除，只能追加。
- logs 进 ELK：CloudTrail 默认是 S3 上的 JSON 文件，业务方根本不看。我们用 Lambda 把它送进 ELK，做成可视化。

后来还接了一些告警规则：root 账号一旦被使用、新 IAM user 被创建、安全组放开了 0.0.0.0/0，这些事件立即触发告警。

## 一些不那么技术但更重要的东西

权限审计最大阻力，从来不是技术，是人。

业务团队会问："为什么我现在不能登录生产了？" "为什么我要走这么多审批？" 你会发现"图方便"是组织里最顽固的惯性。

我们的做法：

1. 先立规矩，再改工具。发布一份《生产环境访问规范》，明确谁能申请、什么场景、什么权限、怎么回收。没有规矩，工具改了也是白改。
2. 流程尽量自动化。审批不要邮件来回，写一个内部工具：填表 → 经理审批 → 自动开权限 → 24 小时过期。临时权限过期自动回收这一条最重要。
3. 给业务方台阶。不是简单"不让你用"，而是提供替代，比如只读账号随时申请，写权限需要 PR review。

安全策略的落地，本质上是和组织惯性的对抗。你不能只做"堵"，必须做"疏"，给出合规且不难用的替代方案，人家才会配合。

## 审计跑完之后

到 2019 年 3 月，我们的状态：

- 生产 AWS 账号上 IAM User 数量从 80+ 降到 30，每个都能对到花名册上具体的人。
- AdministratorAccess 持有者从 12 个降到 3 个（平台组核心成员）。
- 所有 EC2 的 SSH key 个人化，共享 PEM 全部作废。
- CloudTrail 全 Region 启用，关键事件实时告警。
- 一份成文的《生产访问规范》，新员工入职就签。

权限审计这事没有终点。每季度我们都会跑一次 review，看看有没有 drift。安全这件事，永远是持续运动，不是一次性项目。

回头看，这次审计比任何工具升级都重要。权限是事故的最后一道闸门，出了事你可以补救，但如果连"谁干的"都不知道，那就真的是裸奔了。

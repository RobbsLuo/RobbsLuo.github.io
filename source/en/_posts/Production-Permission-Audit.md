---
title: Who Can Touch Production
date: 2019-02-12 11:00:00
tags: [Career, AWS, Security, IAM]
categories:
  - Career
description: Half a year after the ops platform went live, we found far more people could reach production than we thought. This is the story of a full permission audit — IAM policy review, tightened SSH access, and making every action traceable.
lang: en
---

## A close call

Early 2019, something happened that gave me chills.

One day a departed colleague's account was still alive, got picked up by an external scanner, and nearly turned into an incident. After the dust settled we ran the numbers: more people had write access to production AWS resources than were on the team roster. Departures, transfers, temps who helped out, contractors. Permissions were easy to grant and hard to revoke, and over time it became a soup nobody could explain.

> **The biggest risk in access management is not "granting", it is "not revoking."** Every grant had a legitimate reason at the time; nobody remembered to clean up afterward.

We decided immediately: a thorough permission audit. Not a checkbox exercise. Person by person, account by account, policy by policy.

## Three dimensions of auditing

Access is not just "who can log in." There are at least three layers:

1. **AWS resource layer**: who can touch EC2, RDS, S3, Lambda.
2. **System access layer**: who can SSH onto hosts.
3. **Traceability**: if something breaks, can you find out who did it.

Let's walk through each.

## AWS IAM: least privilege

The state before the audit:

- A few shared IAM Users, with access keys handed around.
- Plenty of `AdministratorAccess` policies, "for convenience".
- Root account access keys still alive.

The first cuts targeted exactly these.

### 1. Kill shared accounts

A shared IAM User is an anti-pattern. Every "convenience" you trade here costs you audit trail later; you never know which person triggered an API call.

We required: every person uses their own IAM User, MFA mandatory, keys rotated every three months. Shared use cases go through IAM Roles + STS temporary credentials.

```bash
# Force MFA policy fragment
aws iam create-account-alias --account-alias coreteam

# Attach a "MFA required for anything" policy to a user
aws iam put-user-policy \
  --user-name robbs \
  --policy-name RequireMFA \
  --policy-document file://require-mfa.json
```

`require-mfa.json`:

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

**Effect: without MFA, you can do nothing.** That is the whole point.

### 2. Strip AdministratorAccess

`AdministratorAccess` is a sweet trap. A developer says "I need to touch S3", ops grants admin to save time. Convenient short-term, ticking bomb long-term.

We split permissions by role:

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

Principles:

- **Only touch resources you own**: enforced via resource naming like `coreteam-${project}-*`.
- **Limit region**: prevents accidents in unfamiliar regions.
- **Read by default, write only when explicitly needed**: write access is a separate request.
- **Environment isolation**: dev/staging accounts are physically separate from prod, not just policy-isolated.

### 3. Cage the root account

Root account access keys were deleted outright. The root password lives in a password manager, used by nobody in day-to-day work, retrieved only via a documented process in genuine emergencies. MFA is bound to a hardware token, locked in a cabinet.

> **The root account is a nuclear weapon.** Its job is to be guarded, not used.

## SSH: do not let people hit hosts directly

SSH access to EC2 was another disaster zone. Originally everyone shared one PEM key, dozens of machines using the same key, and "who can log in" was tribal knowledge.

The remediation:

### 1. Personalize SSH keys

Every person uses their own SSH key. No more shared PEM. When someone leaves, just delete their key — no need to re-key the whole fleet.

```bash
# Standard way to inject personal SSH keys into EC2
# via user-data or SSM
#!/bin/bash
aws s3 cp s3://coreteam-ssh-keys/robbs.pub /tmp/
useradd -m -s /bin/bash robbs
mkdir -p /home/robbs/.ssh
mv /tmp/robbs.pub /home/robbs/.ssh/authorized_keys
chown -R robbs:robbs /home/robbs/.ssh
chmod 700 /home/robbs/.ssh
chmod 600 /home/robbs/.ssh/authorized_keys
```

Public keys are centralized in S3, so new machines pull the latest key list automatically.

### 2. Use Session Manager instead of SSH

A better approach is to not expose port 22 at all. AWS Systems Manager Session Manager gives you shell access without SSH, all auth via IAM, all commands recorded by CloudTrail.

```bash
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartPortForwardingSession
```

Benefits:

- Security group closes port 22 entirely; external attack surface drops to zero.
- No PEM keys to manage; IAM permissions are the gate.
- Every session is recorded and traceable.

**Honest caveat: Session Manager was about half a second to a second slower than direct SSH, a worse dev experience.** We pushed it as the production ops channel, while still allowing SSH for daily debugging, but always with personal keys plus a bastion.

### 3. Production bastion

Direct SSH into prod EC2 was tightened. All access goes through a bastion host, which:

- Enforces a second MFA check.
- Records every command (`tlog` or auditd).
- Whitelists reachable target machines.

```hcl
# Terraform: bastion's security group only allows internal CIDRs
resource "aws_security_group" "bastion" {
  name = "bastion-sg"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Traceability: CloudTrail as the baseline

With access tightened, the next step is to make sure every action can be traced. AWS CloudTrail is the foundation.

```hcl
# Terraform: enable CloudTrail, record all API calls
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

Key settings:

- **Multi-region trail**: do not log just one region, or attackers operating elsewhere are invisible.
- **Immutable log files**: S3 bucket policy forbids deletes, allows appends only.
- **Pipes into ELK**: CloudTrail defaults to JSON files on S3 that business teams never read. We used Lambda to ship them into ELK for visualization.

Later we added alerts: root account usage, new IAM user creation, a security group opening 0.0.0.0/0. Each of these triggers an immediate alert.

## Things less technical but more important

**The biggest resistance to permission audits is never technical. It is human.**

Business teams ask: "Why can't I log into production anymore?" "Why all this approval overhead?" You discover that "convenience" is the most stubborn inertia in any organization.

What we did:

1. **Set the rule first, change tools second**. Publish a "Production Access Policy" that specifies who can apply, under what scenarios, what permissions, how they get revoked. Without a rule, tooling changes are wasted.
2. **Automate the process**. No back-and-forth email approvals. We built an internal tool: form then manager approval then auto-grant then 24-hour expiry. **Temporary permissions auto-expiring** is the single most important rule.
3. **Give people a path**. Not just "no", but a compliant alternative: read-only access available any time, write access via PR review.

> **Rolling out a security policy is largely a fight against organizational inertia.** You cannot only block; you have to channel. Provide a compliant path that is not painful, and people will cooperate.

## After the audit

By March 2019:

- IAM User count on the production account dropped from 80+ to 30, each mappable to a specific person on the roster.
- AdministratorAccess holders dropped from 12 to 3 (core platform members).
- All EC2 SSH keys personalized, shared PEMs invalidated.
- CloudTrail enabled across all regions, with real-time alerts on key events.
- A written "Production Access Policy", signed at onboarding.

Permission audits have no end state. Every quarter we run a review to check for drift. Security is a continuous motion, not a one-off project.

In hindsight, this audit mattered more than any tooling upgrade. Access is the last gate before an incident. You can recover from a mistake, but if you cannot even tell "who did it", you are running naked.

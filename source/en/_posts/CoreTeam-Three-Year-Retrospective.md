---
title: Three Years on CoreTeam: Building Internal Systems
date: 2021-04-13 16:00:00
tags: [Career, Retrospective, DevOps, Reflection]
categories:
  - Career
description: From inheriting an unexplainable pile of AWS resources in 2018 to a smoothly running ops platform by 2021. A retrospective on what we got right, what we got wrong, and what "the road of internal systems" really means.
lang: en
---

## At the three-year mark

In spring 2021, I formally handed over ownership of CoreTeam's internal ops platform to my successor. From taking it over in May 2018 to handing it off in April 2021, it was just shy of three years.

Three years is a useful chunk of time, enough to turn a "mess" into "a system that runs". The previous seven posts covered the specifics: resource inventory, CI/CD, ELK, permission audit, Terraform, ECS orchestration. This post is reflection and retrospective.

> Looking back after three years, what really determined outcomes was the choices, not the tools.

## One: What we got right

### 1. Inventory first, refactor later

The first impulse on inheriting the mess was "go K8s, containerize everything". We did not do that. We spent two weeks doing inventory, cleaning zombies, applying tags.

In hindsight this was one of the best calls we made. If we had refactored directly we would have moved a pile of resources nobody understood, and incidents were guaranteed. The precondition for refactoring is understanding the present, otherwise it is gambling.

I tell new hires this repeatedly: in a new environment, do not think "what do I change" for the first three months. Think "can I actually understand what's here". Understanding precedes changing.

### 2. Business teams self-serve, ops steps back

The biggest value of that 2018 Jenkins + ECS pipeline was not "we can release". It was that business teams could release themselves.

This drew the boundary of the entire platform. If ops stays the "release executor", team size is always the bottleneck. More business means more ops pain and more error surface. Letting business self-serve is what moves ops from grunt work to building roads for other teams.

GitLab CI and GitHub Actions later followed the same philosophy: the road builder does not manage who drives on it, just keeps the road flat.

### 3. Treat the tool as a constitution

At the end of the Terraform post I wrote that "the tool exists, execution is the dividing line". Three years in, this hits harder.

Any tool's "power" comes from the organization's acceptance of it as the single source of truth. Terraform worked for us not because the .tf files were pretty, but because consensus formed:

- Every AWS change goes through a Terraform PR.
- Console edits are illegal; drift detection catches them.
- `terraform apply -auto-approve` is banned in production.

Building this consensus is process and culture work, not technical work. Once it forms, the tool's power gets released.

### 4. Three CIs in parallel, no forced unification

Covered earlier: in 2020 we had Jenkins + GitLab CI + GitHub Actions running. We took a lot of "why not unify" pressure.

Looking back, not forcing unification was the right call. The hidden cost of forced migration (incident risk + training cost) far exceeds the cost of running multiple. What matters is collaboration, things like shared artifact stores, shared deploy scripts, and notification channels.

> Diversity itself is not the problem; lack of collaboration is. This applies beyond CI to the entire tech stack.

### 5. Logs before metrics

This ordering drew questions: "why not both at once?"

The honest answer: resources are finite, priorities must be set. Logs answer 80% of business questions ("why did this request fail"), metrics only tell you "the system is slow". Get logs solid first, business teams see value immediately, ROI is obvious. Metrics can be filled in later without blocking anything.

> Resources are never enough, so "what to do first" matters more than "what to do". Pick the highest ROI.

## Two: What we got wrong, or regret

Cannot only tell the good parts, otherwise this is a puff piece.

### 1. Severe underinvestment in docs early on

For the first six months we were heads-down building, but did not record the decision process. A year later a new hire would ask "why awsvpc instead of bridge for ECS", and nobody could answer.

We paid a lot to backfill this: every key decision now requires an ADR (Architecture Decision Record), even if just a few lines:

```markdown
# ADR-007: ECS network mode = awsvpc

Date: 2019-03-15
Status: Accepted

## Context
Three ECS network modes: bridge, host, awsvpc.

## Decision
Standardize on awsvpc.

## Rationale
- Security groups down to the Task level
- No port conflicts
- Future-compatible with Fargate

## Cost
- ENI quota could become a bottleneck (resolved via subnet tuning)
```

An ADR takes five minutes to write and saves hours of "why did we decide this" meetings. Should have done it from day one.

### 2. Over-reliance on "hero employees"

A core system was maintained by one colleague. When he left we scrambled. Single-point knowledge becomes a single-point of failure, especially in ops.

We later mandated at least two people familiar with every core system, and code reviews must include someone outside the team. But that lesson cost us.

### 3. Alerts were too coarse early on

After ELK went live we thought "logs are enough". But logs are passive queries; alerts are active notifications. One night the database connection count saturated, and we only found out the next morning. The logs were full of errors, but nobody was watching.

Later we added CloudWatch alarms + Slack integration. Key metrics (5xx rate, DB connection count, ECS task restart count) trigger immediate alerts when thresholds are crossed. This should have shipped alongside ELK, not after the fact.

### 4. Cost was an afterthought in early containerization

Initially we used fairly large base images (Ubuntu + a pile of tools), and ECR storage costs and image pull times were ugly. Switching to alpine base images cut storage by 80% and doubled pull speed.

Containerization cost optimization was a late realization. Doing it earlier would have saved real money and real release time.

## Three: On "the road of internal systems"

With that out of the way, I want to talk about the bigger picture: what "internal systems" actually is.

Many people misunderstand "internal systems" as "tools for internal use, low technical content". This is wrong.

The essence of internal systems is "amplifier". A good internal tool lets a team of ten perform like fifty; a bad one (or none at all) makes fifty perform like ten.

Everything CoreTeam did in three years (resource management, CI/CD, logging, permissions, IaC, ECS orchestration) was building "amplifiers". Each step appears to solve a technical problem, but really solves an organizational efficiency problem:

- Inventory → transparent billing, evidence-based decisions.
- Self-service release → business autonomy, ops not a bottleneck.
- ELK → debugging time from hours to minutes.
- Permission audit → traceable incidents.
- Terraform → controllable, reproducible changes.
- ECS orchestration → scaling without humans watching.

> The value of internal systems is never "how fancy the tech is". It is "how many people work better because of it".

## Four: A few words for those who come after

If I had one piece of advice for someone doing internal systems, it would be:

Do not chase new tech; chase the business team's real pain points.

Is K8s impressive? Yes. But if your scale does not demand it, ECS is plenty. Is a service mesh advanced? Sure. But with a dozen services, its complexity is a burden.

A tool's value is relative to the problem it solves. Talking tools without talking business is technical self-indulgence.

Second: take docs and training seriously; do not be an "invisible hero".

The success metric for an internal system is not "it runs". It is "people use it". An internal tool nobody uses is a failure no matter how impressive. Docs, training, internal sessions, these "non-technical" things often matter more than the code itself.

Third: accept that you will not be there forever; make systems that can run without you.

At handoff, what made me proudest was not "how powerful the system is", but "it keeps running after I leave". That is the highest state of an internal system: it does not depend on any one person.

## Five: In closing

These three years at CoreTeam were an important stretch of my career.

Technically I learned concrete things: Terraform, ECS, ELK, IAM, Cloud Map. But more importantly, I learned how to actually build something from nothing inside a real organization, how to weigh trade-offs, how to collaborate, how to compromise, how to hold the line.

> The road of internal systems has no endpoint. Every stage's "done" is the next stage's "start".

What I handed off was not a perfect system, but it was a system someone can take over, modify, and keep running. That is enough.

I will carry these lessons into the next scenario. Hope that road also walks steadily.

Thanks, CoreTeam. These three years were worth it.

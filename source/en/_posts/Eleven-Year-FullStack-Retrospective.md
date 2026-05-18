---
title: 11 Years, One Company - A Full-Stack Retrospective
date: 2026-05-18 14:00:00
tags:
  - Career
  - Full-Stack
  - Retrospective
categories:
  - Career
description: 2014 to 2026, eleven years at one company. Always full-stack — only the stack changed, from MEAN.JS and Rails to Vue and React; from CoreTeam to Collateral to DAL to the fintech suite, from hand-written code to AI coding. This is my technical-system retrospective, and the closing of a journey.
lang: en
---

In July 2014, I joined Changsha Honey Badger Information Technology Co., Ltd.

In May 2026, I left.

Eleven years. One company.

Before that, I also ran my own venture. From 2011 to 2014, I built a conference-information platform. Those three years are a different story; I will not expand on them here, but they are the real starting point of my technical career. All together, my time in engineering is these eleven years plus the three years of entrepreneurship before them.

This is not a resume. Everything on the resume, I will not repeat. What this post is about is how my technical system grew, layer by layer, over those eleven years, and what I learned at each stage.

## The Beginning: Backend Grunt Work

After joining in 2014, I spent my first year at Baishanghao, a marketing platform for department stores and supermarkets, built on the MEAN.JS full-stack framework, frontend and backend alike. It was my first real project.

In 2015 I moved to HomePartners, building a data-computation and investment-management system for real-estate investment, with a Ruby on Rails backend. I stayed there three years.

Nothing flashy in that phase. It was writing one endpoint at a time, fixing one bug at a time. But two things from that period I only later realized were valuable.

First, a respect for data. Investment returns, asset distribution, risk metrics: the numbers cannot be wrong. One decimal point off and every downstream report collapses. I built a habit that I carried all the way into the fintech tools, that any calculation logic has to be reproducible and traceable.

Second, a sense of the whole project. I did not just write backend code. I touched deployment, monitoring, the release pipeline. Not because the division of labor let me, but because I wanted to understand. That habit later became the foundation for my architectural judgment.

> Being full-stack is not "knowing how to do everything." It is having a feel for the whole project. You do not have to be expert in every piece, but you have to know what every piece is doing.

## CoreTeam: The First Taste of a "Global View"

In 2018, I moved to CoreTeam, building the user-profile system and the ops management platform.

This was the first time I started from "designing a system" rather than from "writing an endpoint."

The user-profile system had to ingest coding activity, compute capability metrics, and do multi-dimensional analysis. The ops platform had to manage resources, releases, and monitoring. What these systems had in common: there was no ready-made path. You had to define the process yourself.

The core ability I picked up in this phase was abstraction: turning scattered business requirements into reusable computation nodes and composable functional modules. That abstraction ability transferred directly when I started building fintech tools.

It was also in CoreTeam that I started leading people and doing planning. Technical ability answers "can it be done." Management ability answers "can it be done sustainably."

## Supernova: End-to-End on the Fintech Suite

In 2021, I moved to Supernova and took on the fintech tools product line. This was the heaviest stretch of the eleven years.

The fintech suite has 13 tools, a wealth calculator, a mortgage calculator, a refinance tool, a home-affordability tool, and so on. Each is a standalone calculator, but they all share the same underlying financial logic.

What I did on this line spanned a wide range:

- Product planning: which tools to build first, which later, how the tools relate to each other.
- System architecture: frontend Vue 2.7 later migrated to React 19, backend API design, state-management approach.
- Core algorithms: confirming the semantics and implementing the financial calculations (specific algorithms aside, the numbers for every tool have to reconcile).
- Delivery: a three-person team, 13 tools, AI-coding-driven.
- Quality: unit tests, e2e, lint, release pipeline, the full set.

At the same time, I also ran Collateral Management and the DAL group.

Collateral taught me performance optimization: how to compress computation time under large data volumes, how to design indexes, how to tune SQL.

DAL taught me cross-language engineering: maintaining Java and Python versions of the database SDK with aligned behavior. Connection management, read-write splitting, slow-query monitoring had to match across both. Maintaining two codebases is not copy-paste. It is using engineering conventions to guarantee cross-language consistency.

## AI Coding: The Biggest Variable in These Eleven Years

If the first ten years of my growth followed a "conventional" path, full-stack throughout, only the stack shifting from MEAN.JS and Rails to Vue and React, execution to architecture, individual to team, then AI coding, starting in 2025, was the biggest variable.

Once Claude Code came out, I made a call: this is not a question of "whether to use AI." It is a question of "sooner or later."

The fintech-suite migration then became the proving ground for AI coding. A month and a half, 13 tools, Vue to React, run in parallel by a migration agent.

The story after that is covered in the previous posts, Skill distillation, CLAUDE.md across the team, Subagent orchestration. By early 2026, AI coding had become the standard way our team works.

> What AI coding changes is not "how code is written." It is "where an engineer's attention should go." Time that used to go into writing repetitive code now goes into designing rules, orchestrating workflows, and making architectural calls.

## What the Technical System Looks Like

After eleven years, if I had to draw a picture of my "technical system," it would look roughly like this:

```
                    ┌─────────────────┐
                    │  Engineering    │  Abstraction,
                    │  Mindset        │  reuse, automation
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───────┐ ┌───▼────────┐ ┌───▼────────┐
     │   Frontend     │ │  Backend   │ │   Data     │
     │ React/Vue/Vite │ │ API/Arch   │ │ SQL/Tuning │
     │ State/CSS      │ │ Perf/Deploy│ │ Sharding   │
     └────────────────┘ └────────────┘ └────────────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │   AI Coding     │  Skill / CLAUDE.md / Agent
                    └─────────────────┘
```

At the base: three tech stacks, frontend, backend, data. In the middle: the engineering mindset that connects them. On top: the AI coding system that multiplies the output of the whole thing.

This system was not built in a day. It is eleven years of layers, each one with its own pitfalls and tuition paid.

## One Line for a New Hire

If I had to give someone just starting out one line, it would be:

> Do not rush after new technologies. Go deep on one thing first. Once you have gone deep on one thing, the second one goes much faster.

When I was writing Rails at HomePartners, building investment calculations, I had no idea I would later be doing React. But the respect for data, the sense of the project, the abstraction ability I built in those years all came back when I was doing React.

Tech stacks change. The underlying engineering thinking and problem-solving ability do not. Those are the most valuable things from these eleven years.

## Closing

Eleven years. One company.

People have asked me why I stayed so long. The answer is not complicated: every two or three years brought a new challenge. The tech stack kept changing, the business kept changing, the role kept changing. That was enough change; I did not need to job-hop for novelty.

But this is a pivot point. The AI-coding road is just beginning. The fintech-suite system is in shape. The remaining upside needs new terrain to prove itself on.

> Eleven years is not the destination. It is the delivery of one phase.

Writing this post is the period at the end of that journey. Where I go next, what I do next, that is for later.

But whatever I do, the things these eleven years built, respect for data, a sense of the project, an understanding of engineering, judgment about AI, they all come with me.

And that is enough.

---
title: Claude Code for the Whole Team - CLAUDE.md and Agent Orchestration
date: 2025-09-15 15:00:00
tags:
  - Career
  - Claude Code
  - CLAUDE.md
categories:
  - Career
description: After the migration proved out, I pushed Claude Code from a personal tool to a team-wide standard. This post covers how to write team-level AI coding rules into CLAUDE.md, and how to orchestrate Subagents for parallel multi-person work.
lang: en
---

After the fintech-suite migration landed, I did one more thing: I pushed Claude Code from a tool I used personally into the way the whole team works.

During the migration's month and a half, it was basically me and Claude Code (or rather, me plus a pile of agents) doing the running. But after it was done, everyone else had to get on board, otherwise these Skills and conventions stayed mine alone, and the moment someone else touched them they fell apart.

This post is about two things: how to write team rules into CLAUDE.md, and how to orchestrate Subagents for multi-person parallel work.

## Why Verbal Conventions Are Not Enough

Once the migration was done, people on the team started using Claude Code on their own. But a problem surfaced quickly: everyone's output style was different.

Engineer A asked Claude to write a component, organizing files as one-directory-per-component. Engineer B asked Claude and got everything dumped in a single file. Engineer C had thorough test coverage; Engineer D never asked Claude to write tests at all.

You can say "we need to standardize," but it amounts to nothing. In the AI-coding era, verbal conventions have almost zero binding power, because every person's conversation with the AI is different, and the AI only follows the context of the current conversation.

> A human team can scrape by with wikis and verbal handoffs for conventions. An AI team has to write them into a place the machine can actually read.

That is the point of CLAUDE.md.

## What CLAUDE.md Is

CLAUDE.md is a config file that Claude Code automatically reads from the project root. Write your team conventions into it, and Claude Code reads it every time it starts, then follows the rules.

Think of it as a CONTRIBUTING.md written for the AI. A new human reads the wiki; a new AI instance (that is, every Claude Code launch) reads CLAUDE.md.

## What Our CLAUDE.md Looks Like

Here is the core, sanitized:

```markdown
# Project Engineering Conventions (CLAUDE.md)

## Tech Stack
- React 19 + TypeScript 5.x
- Vite 6 build
- Zustand state management
- CSS Modules styling
- pnpm monorepo

## Code Conventions
- Function components only, no class components
- File naming: components PascalCase, utility functions camelCase
- One directory per component: `Button/index.tsx` + `Button.module.css` + `types.ts`
- Cross-component state goes through Zustand stores, component-local state via useState
- No `any`; types must be explicit

## Testing Requirements
- Every new component must have at least a main-path unit test
- Business tools must have e2e coverage on core calculation flows
- Test files live in `__tests__`, named `xxx.test.ts`

## PR Conventions
- Commit messages follow conventional commits
- PR description must include: what changed, why, how it was tested
- No direct pushes to master, PRs only

## Forbidden
- No new dependencies added without a stated reason in the PR
- No `// @ts-ignore` to skip type checks
- No committing console.log
```

It is not anything fancy. It is just team conventions written in a format the AI can parse. But the effect is dramatic, once it was in place, everyone's Claude Code output started converging on the same style.

## How to Organize CLAUDE.md

Our CLAUDE.md is not one giant file. It is layered:

```
project-root/
├── CLAUDE.md              <- global rules (tech stack, code style, PR flow)
├── packages/
│   ├── shared/
│   │   └── CLAUDE.md      <- shared-library-specific rules
│   └── tools/
│       ├── mortgage-calculator/
│       │   └── CLAUDE.md  <- tool-specific rules (business logic constraints)
│       └── ...
└── .claude/
    └── skills/            <- reusable Skills
        ├── vue-to-react-mapping.md
        ├── code-review-checklist.md
        └── ...
```

Claude Code merges these files by layer: the global rules from the root, plus the specific rules from the current working directory. That way, when you are working inside one tool, Claude knows both the global conventions and that tool's special constraints.

> The layered design of CLAUDE.md matters. Global rules manage consistency; local rules manage specificity. Do not cram everything into one file.

## How Subagents Are Orchestrated

CLAUDE.md solves the "uniform conventions" problem. Subagents solve the "parallel throughput" problem.

During the migration I used Subagents to run 13 tools in parallel. After the migration, everyday development also started using the Subagent pattern.

A real scenario: three tools each need a new feature at the same time. The old way: three people each do their own thing, none aware of the others. The Subagent way:

```
Main Agent (me)
├── Subagent A: add early-repayment to the mortgage tool
├── Subagent B: add a new rate scenario to the refinance tool
└── Subagent C: add tax-and-fee calc to the home-affordability tool
```

Each Subagent carries the same CLAUDE.md and the relevant Skills, running independently. When done, results roll up to the main agent for an integration check.

The key: the main agent's job is not to write code. It is to orchestrate and act as the safety net.

- Task allocation (who does what)
- Conflict detection (did two agents touch the same shared file?)
- Integration verification (with all three features added, does the whole thing run?)

> In the era of humans writing code, managers allocated tasks. In the era of AI writing code, humans design the orchestration. The center of gravity of the role shifts from execution to orchestration.

## What Actually Changed

About two months after the rollout, a few clear shifts:

- PR style unified: no longer a different style per person; "formatting issues" in code review dropped by 80% or more.
- Onboarding got faster: a new engineer comes in, reads CLAUDE.md, glances at the Skills, and the code Claude Code produces naturally conforms.
- Test coverage went up: CLAUDE.md specifies testing requirements, so Claude Code proactively writes tests every time, without anyone prompting.

But there is a cost: maintaining CLAUDE.md is not cheap. When the tech stack changes or conventions shift, it has to be updated in sync. I put CLAUDE.md updates on the team's regular iteration checklist, otherwise it goes stale fast.

## The Most Important Takeaway

After a full round of rollout, the most important thing I learned is this:

> Rolling out AI coding to a team is not about teaching everyone to write prompts. It is about making conventions machine-readable.

Prompts are an individual skill; teaching them does not control how each person uses the tool. CLAUDE.md and Skills are engineering infrastructure; once written, everyone benefits automatically.

This is exactly the same logic as DevOps used to be. You do not teach every new hire how to configure CI/CD. You set up CI/CD and let the pipeline run. Standardizing AI coding is the same. It is not about teaching tricks; it is about building infrastructure.

The next post is the full retrospective on AI coding for the fintech suite: the two-year arc from "one person using AI" to "team-wide engineering."

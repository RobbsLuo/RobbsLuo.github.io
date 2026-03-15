---
title: Two Years on the Fintech Suite - Codifying AI Engineering
date: 2026-03-15 10:00:00
tags:
  - Career
  - Claude Code
  - AI Engineering
categories:
  - Career
description: From first touching Claude Code in early 2025 to rolling AI coding engineering out at the team level by early 2026. This is the two-year retrospective for the fintech suite: how the system from Prompt to Skill to CLAUDE.md to Subagent grew, layer by layer.
lang: en
---

In February 2025, when Claude Code shipped its research preview, I started using it. The fintech suite was just about to undertake the big Vue 2.7 to React 19 migration.

By March 2026, when I am writing this, three people on the fintech suite have used AI coding to deliver the full migration of 13 tools, plus continuous iteration and new features. After two years, AI coding here is no longer a tool. It is an engineering system.

This is the full retrospective on how that system came together, layer by layer.

## Timeline

First, the key milestones:

```
2025-02  Started using Claude Code (research preview)
2025-03  Wrote first prompts, hand-migrated pilot tools
2025-04  Extracted the first batch of Skills, codified migration rules
2025-05  UI migration agent took shape, 13 tools migrated in parallel
2025-06  Skill library reached 10+, started reusing them
2025-07  Switched Webpack to Vite, build-tool migration done
2025-08  Migration wrapped up, started pushing to the team
2025-09  CLAUDE.md + Subagent mode entered the team
2025-12  Team AI coding conventions stabilized
2026-03  Wrote this retrospective
```

Notice the rhythm. The system was not designed up front. It grew layer by layer, pushed forward by real needs.

## The Four Layers of Evolution

Looking back over those two years, the evolution of AI coding engineering splits roughly into four layers.

### Layer 1: Prompt (early 2025)

It started with writing prompts. I would manually paste Vue code into Claude and ask it to translate to React.

The problem was obvious: every time, I had to re-explain the conventions, the constraints, the acceptance criteria. Prompts kept getting longer, and switching to a new file meant starting over.

> The prompt layer solved "can the AI do this work at all." It did not solve "will it do it the same way every time."

### Layer 2: Skill (spring 2025)

After writing enough prompts, I noticed how much was being repeated. I extracted the conventions, the mapping rules, and the checklists into Skills that Claude Code loaded automatically.

This layer solved consistency: the reason all 13 tools came out with a uniform style was the Skills, not the prompts.

```markdown
# Skeleton of a Skill (sanitized)
## When to use
Use this Skill when xxx.

## Rules
- Rule 1
- Rule 2

## Forbidden
- No xxx

## Acceptance criteria
- tsc clean
- ESLint zero errors
```

Skills took me from "teach the AI how to do it, every single time" to "teach it once and reuse it automatically from here on."

### Layer 3: CLAUDE.md (autumn 2025)

Skills solved "consistency within a single task," but across the team, every person's output still looked different. Because the global engineering conventions (tech stack, PR process, testing requirements) were not in the Skills, and they were not consistently in everyone's head either.

CLAUDE.md writes the team-level conventions into the project root, and Claude Code reads it every time it launches. This layer solved team consistency, not just one person's code being uniform, but everyone's.

### Layer 4: Subagent Orchestration (late 2025)

The first three layers were all about "how one AI does one task well." At layer four, we started running multiple agents simultaneously to handle parallel work.

Subagent orchestration solved parallel throughput: three people pushing three features at once, relying not on each grinding alone, but on the main agent for orchestration and safety-net checks.

```
Orchestration layer (main agent)
  ├── Task A (Subagent)
  ├── Task B (Subagent)
  └── Task C (Subagent)
       │
       └─ each carries CLAUDE.md + Skills
```

## What Worked and What Did Not

Two years in, some judgments I am fairly confident about:

What worked:

- Rules first. No matter what you are doing, write the rules into a Skill or CLAUDE.md before letting the AI run. The AI works fast, but without rules, faster just means more errors, faster.
- Layered design. Skill manages single tasks, CLAUDE.md manages the global picture, Subagent manages orchestration. Each owns its own lane, no mixing.
- Automated verification. AI-produced code must pass lint + type checks + tests. AI coding without an automated review safety net is just gambling.

What did not help much:

- Teaching the team to write prompts. The ROI is low; everyone uses the tool differently and you cannot enforce it. Better to spend that energy on Skills and CLAUDE.md.
- Chasing prompt "perfection." A prompt that is 80 percent good is enough. The remaining 20 percent gets covered by Skills and the verification pipeline. Chasing perfect prompts is over-optimization.
- Letting AI do everything. Some things, the pitfalls of a build-tool migration, diagnosing a production incident, the AI cannot answer for you. Those still depend on human experience.

> AI coding is not "using AI to replace humans." It is "using AI for the repetitive parts so humans can focus on the parts that need judgment."

## How Quality Is Guaranteed

The most common question: you use AI to write code, how do you guarantee quality?

My answer: the quality of AI-produced code depends on how solid your engineering infrastructure is, not on how smart the AI is.

Our quality safety net:

1. CLAUDE.md sets conventions, so the AI produces code by the rules.
2. Skills set mapping and constraints, so the AI works within the frame.
3. ESLint + tsc auto-check, so formatting and type issues are caught automatically.
4. e2e covers core paths, so business-logic regressions surface automatically.
5. Human review, for architecture decisions and edge cases.

Of those five layers, the first three are automatic, the fourth is semi-automatic, and only the fifth involves humans. Human attention concentrates where judgment is actually needed, rather than on formatting and syntax.

## If I Started Over

If I did it again, I would adjust two things about the pacing:

- Push CLAUDE.md earlier. I rolled it out after the migration was done. It should have come alongside the first Skill, so the rest of the team could get involved sooner.
- Set up Subagent orchestration earlier. We used Subagents during the migration but were late to adopt them for everyday development. For any multi-tool parallel scenario, orchestration mode pays off the earlier you start.

Overall though, the two-year arc was reasonable. From Prompt to Skill to CLAUDE.md to Subagent, each layer solved a problem the previous layer could not. It was not designed up front; it was pushed into shape by need.

> AI coding engineering is like all engineering, there is no silver bullet, only layer after layer of problem-solving. How many layers you can stack determines how reliable the AI's output gets.

The fintech-suite AI coding system is more or less in shape as of 2026. But I know this is only a beginning. The tools and paradigms of AI coding shift so fast that this year's system may need rebuilding next year. Staying in learning mode and ready to adjust is probably the only constant in this system.

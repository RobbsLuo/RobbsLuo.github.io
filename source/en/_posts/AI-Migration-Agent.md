---
title: A UI Migration Agent - Moving 13 Tools at Once with Claude Code
date: 2025-05-14 10:00:00
tags:
  - Career
  - Claude Code
  - Agent
categories:
  - Career
description: Migrating 13 Vue 2.7 tools in the fintech suite to React 19 - I did not hand-migrate them. Instead I built a UI migration agent on top of Claude Code and ran all 13 tools in parallel, shipping in about a month and a half.
lang: en
---

The fintech suite had 13 tools, all in Vue 2.7, and they all needed to move to React 19.

By the traditional hand-migration math, one tool takes about three days, so that works out to nearly two months; three people hand-migrating in parallel, with all the pitfalls and rework, meant three months minimum.

I ended up spending roughly a month and a half, and it was not me grinding alone. It was three people spreading across all 13 tools at the same time.

How? The core idea came down to one thing: I did not migrate the code myself. I built an agent that migrates code.

## The Conclusion First

> Do not ask AI to migrate your code. Ask AI to be a "migration engineer."

These sound similar but they are very different. The former means you sit there feeding Claude one file at a time, asking it to translate into React. The latter means you design a workflow where the agent scans, decides, generates, and verifies on its own, and you only step in for the edges it cannot handle.

## Why Migrate All 13 at Once

The fintech tools look like this: a wealth calculator, a mortgage calculator, a refinance tool, a home-affordability tool, a debt-consolidation tool, and so on, about a dozen of them. They are business-independent, but the tech stack is identical: Vue 2.7 + Pinia + Element-Plus + Webpack.

That is actually a scenario made for parallelism. If the migration rules can be distilled into a "spec," then 13 tools are just 13 applications of the same rules, not 13 brand-new problems.

So I treated them as one set of migration rules plus 13 executions, not 13 independent migration tasks. That shift in framing matters.

## How the Agent Was Designed

I built a UI migration agent on top of Claude Code with a four-step workflow:

```
┌─────────────┐   ┌─────────────┐   ┌──────────────┐   ┌─────────────┐
│  1. SCAN    │──▶│  2. MAP     │──▶│ 3. GENERATE  │──▶│  4. VERIFY  │
│ parse SFC   │   │ rule map    │   │ emit React   │   │ lint+types  │
└─────────────┘   └─────────────┘   └──────────────┘   └─────────────┘
                                                                 │
                                                        on fail ▼
                                                     back to GENERATE
```

**Step 1, SCAN**: The agent reads each Vue single-file component, splits out the `<template>`, `<script setup>`, and `<style>` blocks, and structures the metadata (props, emits, computed properties, refs) into a JSON description. This gives the mapping step real context, not raw string substitution.

**Step 2, MAP**: The core of the whole agent. I wrote a mapping ruleset (later extracted into a Skill, more in the next post) that looks roughly like this:

```yaml
# vue-to-react-mapping (excerpt, sanitized)
template:
  v-if:     "-> conditional render {cond && <Comp />}"
  v-for:    "-> Array.map, prefer business id as key"
  v-model:  "-> useState + onChange two-way binding"
  @click:   "-> onClick"
  :class:   "-> clsx()"
  slot:     "-> children / render prop"

script:
  ref / reactive:  "-> useState"
  computed:        "-> useMemo"
  watch:           "-> useEffect (with deps)"
  onMounted:       "-> useEffect(() => {}, [])"
  defineProps:     "-> interface Props + destructured defaults"
  defineEmits:     "-> callback props (onXxx)"

style:
  scoped css:  "-> CSS Modules (*.module.css)"
```

**Step 3, GENERATE**: The agent emits React 19 `.tsx` + `.module.css` files following the rules, with state management unified on Zustand (why Zustand is in the next post; short version: lowest migration cost and the mental model is closest to the Composition API).

**Step 4, VERIFY**: Automatically runs ESLint + `tsc` type-checking + a smoke e2e over the critical paths. If anything fails, it loops back to GENERATE and the agent fixes it itself, without pulling in a human. This is what turns "human watching AI" into "AI watching AI."

## The Key Design Decision: Rules and Execution Stay Separate

Here is a judgement call I think matters a lot:

> Migration rules are decided by humans. Migration execution is done by AI. Never mix the two.

If you tell the agent to "just figure it out," tossing it a Vue file and saying "convert to React," you get something that runs but has no consistent style. Thirteen tools written thirteen different ways is a maintenance nightmare.

My approach was to nail the rules down first and then let the agent work inside that frame. The rules covered:

- Naming conventions (components PascalCase, hooks prefixed with `use`, utilities camelCase)
- File organization (one directory per component: `index.tsx` + `*.module.css` + `types.ts`)
- State boundaries (cross-component state goes through Zustand stores, component-local state via useState, no mixing)
- Styling (CSS Modules everywhere, no styled-components)
- Testing requirements (each tool must have e2e coverage on its main path, and it has to pass before the migration counts as done)

The agent could only operate inside that frame. It could not invent its own conventions.

## Running 13 Tools in Parallel

Claude Code supports a Subagent mode that lets you split a large task into independent subtasks running in parallel. My orchestration looked roughly like this:

```bash
# Simplified for illustration; actual run uses Claude Code Task orchestration
tools=(cash-reserve mortgage refinance home-affordability \
       debt-consolidation retirement liquidity tax-payment ...)

for tool in "${tools[@]}"; do
  # one independent agent per tool, carrying the same ruleset
  claude run migration-agent --target "$tool" --rules ./skills/ &
done
wait
echo "all done, start manual cleanup"
```

Each tool gets its own agent instance running the full SCAN -> MAP -> GENERATE -> VERIFY flow. All 13 tools spin at once. The machines are grinding while the humans go look at the failures the machines cannot resolve.

In practice, most tools reached a passing VERIFY on the first run. A few got stuck on complex computed properties or on places that leaned heavily on Element-Plus components (certain table components with complicated slot setups, for instance) and needed manual intervention. But the intervention volume was a fraction of what pure hand-migration would have produced.

## Where the Month and a Half Actually Went

To be honest, the month and a half was not the agent running for a month and a half. The agent probably ran for about two weeks. The rest went into:

- Writing rules (the most expensive part): For the first two weeks I was almost entirely writing and tuning the mapping rules, running three or four pilot tools through them repeatedly until the rules stabilized.
- Manual cleanup: Some Vue tricks (`$attrs` pass-through, dynamic components, render functions, `v-html`) did not convert cleanly and had to be hand-fixed.
- e2e backfill: Migration being done does not mean the business logic is correct. Fintech tools have data semantics that cannot drift, so verification took real time.
- Integration and staged rollout: Rolling out 13 tools together is not cheap on the integration side, and staged rollout had to happen step by step.

## The Decision That Was Worth the Most

Looking back, the most valuable thing was not using AI. It was deciding to build the rules before starting work.

If we had just dove in and migrated file by file, AI would have made things messier over time: every file coming out different, nobody able to pick up someone else's work. Spending two weeks stabilizing the rules and Skills up front meant the remaining 13 tools felt almost like copy-paste.

> The bottleneck of AI-driven migration is never the AI's capability. It is whether you have explained the rules clearly enough.

Get the rules right and running 13 tools in parallel falls out naturally. Get the rules wrong and even one tool is a gamble.

The next post breaks down how those ten-plus migration Skills were written, and why I think Skills are the most underrated thing in AI coding right now.

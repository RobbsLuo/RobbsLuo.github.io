---
title: Migration Skills - 10+ Reusable Artifacts
date: 2025-06-17 14:00:00
tags:
  - Career
  - Claude Code
  - Skill
categories:
  - Career
description: After migrating 13 tools, I extracted the rules I kept reusing into 10+ Claude Code Skills. This post covers what a Skill actually is, how to write one, and why I think it is the most underrated thing in AI coding right now.
lang: en
---

In the last post I mentioned using a UI migration agent to move 13 tools from Vue 2.7 to React 19 in about a month and a half.

What a lot of people actually zeroed in on was the line after that: "and then I distilled out 10+ reusable Skills."

So this post is about exactly that: what a Skill is, how to write one, and why I think it is the most underrated thing in AI coding today.

## What a Skill Actually Is

A Skill in Claude Code, put simply, is a reusable instruction template. You write down the way something should be done, along with its constraints and acceptance criteria, as structured markdown. Claude Code then loads it automatically in the right context and follows your rules.

Think of it this way: a prompt is a one-time conversation; a Skill is engineering spec that has been allowed to settle.

The analogy: the first time you teach a new hire how your team files PRs, you type out a long message in Slack. The second time, you type it again. By the third time you are annoyed, so you write it into the wiki, right? A Skill is that wiki, except the AI goes and reads it on its own when it needs to, without you reminding it.

## What a Skill Looks Like

Take `vue-to-react-mapping`, the most-used Skill during the migration (sanitized):

```markdown
# vue-to-react-mapping

> Map common Vue 2.7 SFC patterns to React 19 so all 13 tools migrate with consistent style.

## When to use
When migrating a Vue single-file component into a React function component.

## Template mapping rules
| Vue pattern | React output | Notes |
|---|---|---|
| `v-if` | `{cond && <Comp />}` | short-circuit over ternary |
| `v-for` | `arr.map(item => ...)` | prefer business id as key |
| `v-model` | `useState + onChange` | controlled component |
| `:class` | `clsx(...)` | no string concatenation |
| `slot` | `children` / render prop | named slots via object props |

## Script mapping rules
- `ref(x)` / `reactive({})` -> `useState`
- `computed(() => ...)` -> `useMemo(() => ..., [deps])`
- `watch(src, cb)` -> `useEffect(() => cb(), [deps])`
- `onMounted` -> `useEffect(..., [])`
- `defineProps` -> `interface Props` + destructure + defaults

## Forbidden
- No class components, function components only
- Do not use `dangerouslySetInnerHTML` for `v-html`, mark as TODO instead
- Do not invent state-management approaches, cross-component state goes through Zustand

## Acceptance criteria
- `tsc --noEmit` passes clean
- ESLint zero errors
- Main-path e2e passes
```

Notice it is not a vague prompt. It is an engineering document with a scope, a rule table, a forbidden list, and acceptance criteria. That is what makes the AI's output consistent across all 13 tools.

## The Skills I Ended Up With

After the migration wrapped, I counted. The actual output broke down roughly like this, in a few categories:

**Mapping** (migration core)
- `vue-to-react-mapping` - Vue syntax to React mapping rules
- `element-to-component-lib` - Element-Plus components to the target UI library
- `pinia-to-zustand` - state management migration rules

**Convention** (consistency)
- `react-naming-convention` - naming rules
- `file-structure` - the one-directory-per-component layout
- `css-modules-style` - styling convention

**Quality** (safety net)
- `migration-verify` - post-migration self-check list (types, lint, tests)
- `test-coverage-check` - coverage verification
- `a11y-checklist` - accessibility checks (fintech tools have real a11y requirements)

**Process** (team coordination)
- `migration-handoff` - handoff checklist after a tool is migrated
- `code-review-checklist` - review points for migration-related PRs

That is 10-plus in total. None of them is long, but each one nails down a specific thing that would otherwise keep going wrong if it were not written down.

## Why Zustand Instead of Redux

Quickly closing the thread from the last post. I landed on Zustand for state management, and the reasons are very pragmatic:

- Lowest migration cost: Pinia's `defineStore` maps almost one-to-one onto Zustand's `create`. The agent converts it cleanly on the first pass with almost no manual cleanup.
- No boilerplate: Redux Toolkit is already lean, but Zustand is lighter still. For a calculator tool, I do not want a pile of slices and reducers just to hold a few numbers.
- TypeScript-friendly: inference is natural, no extra type ceremony.

```ts
// Pinia (before)
export const useCalcStore = defineStore('calc', () => {
  const amount = ref(0)
  const setAmount = (v: number) => { amount.value = v }
  return { amount, setAmount }
})

// Zustand (after)
interface CalcState {
  amount: number
  setAmount: (v: number) => void
}
export const useCalcStore = create<CalcState>((set) => ({
  amount: 0,
  setAmount: (v) => set({ amount: v }),
}))
```

See how direct that mapping is. The agent gets it right on the first pass, no tuning needed.

> **The criterion for picking a tech stack is not "which one is better." It is "which one is least likely to go wrong when AI does the migration in this specific context."**

## Why Skills Are Underrated

I think the reason people get stuck with AI coding right now is this: they obsess over prompts and ignore Skills.

A prompt is one conversation with the AI, gone once it ends. A Skill is the part of that conversation worth keeping, frozen and automatically reused next time.

The analogy again: a prompt is you explaining something to a new hire once. A Skill is you writing that explanation into the team playbook. The former repeats every time; the latter you explain once.

The migration confirmed something for me: the ceiling of AI coding is set by the prompt, but the floor and the consistency are set by the Skill. Running 13 tools in parallel and getting consistent results was not thanks to a great prompt. It was thanks to the Skills nailing the conventions down.

If your team is on Claude Code, I strongly recommend extracting the conventions, constraints, and checklists that keep coming up into Skills. It costs some time up front, and it pays back every single time after.

Next post: the pitfalls of moving the build tooling from Webpack to Vite. That one was more painful than the component migration.

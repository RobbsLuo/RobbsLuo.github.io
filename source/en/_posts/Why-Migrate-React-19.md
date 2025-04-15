---
title: Why We Decided to Migrate Vue to React 19
date: 2025-04-15 14:00:00
tags:
 - Career
 - React 19
 - Vue
 - Tech Selection
categories:
 - Career
lang: en
description: Vue 2.7 has hit EOL. React 19 just shipped. This is not a "which framework is better" post — it is a grounded account of why, in April 2025, a specific financial-tool product line decided to migrate.
---

This post may offend some Vue loyalists. Let me be clear up front: this is not a "React is better than Vue" article. It is a grounded account of one business scenario, at one specific moment in April 2025, making a deliberate, opinionated selection.

If you disagree after reading it, come argue.

## Setting the scene

By early 2025 our state was:

- Frontend: 13 tools, all Vue 2.7 + Webpack
- Component library: 15+ components, built on Vue 2.7
- pnpm monorepo + Turbo parallel builds, humming along
- Test coverage: 93% on core modules

Everything looked fine. But two shadows kept growing.

## The Vue 2.7 countdown

Vue 2.7 is the last minor of Vue 2.x, released in July 2022. Its purpose was to give Vue 2 projects a transition path to Composition API. But it has an EOL: December 31, 2023, when the entire Vue 2 line stopped receiving maintenance.

> By 2025, we were running a framework that had been unmaintained for over a year.

What does that mean?

- Security patches are gone. A CVE shows up, and you are forking a branch to patch it yourself.
- The ecosystem is leaving. `vue-router@3`, `vuex@3`, and companion libraries are no longer maintained either. Vue Router and Pinia target Vue 3.
- Hiring gets harder. Resumes saying "expert in Vue 2" are increasingly rare. Younger frontend engineers learn React or Vue 3; nobody proactively learns an EOL framework.

Someone will say: "Just upgrade to Vue 3."

Right, Vue 3 is an option. After serious evaluation we concluded React 19 fits our scenario better. Here is why.

## Vue 3 upgrade vs React 19 migration

These were the two options on the table. Compared point by point.

### Ecosystem: React has deeper roots in financial tooling

Our tools are not content sites. They are heavy-interaction data tools: tables, forms, charts, drag-and-drop, real-time calculation. React's ecosystem advantage in this domain is concrete:

- ag-Grid / TanStack Table: the de facto standard for financial reporting. The React version is a full step ahead of the Vue version.
- TanStack Query: server-state management that pairs beautifully with our Spring Boot API.
- Recharts / Nivo: the data-visualization ecosystem is far richer than Vue's.
- react-hook-form: the complex-form solution; on the Vue side `vee-validate` works, but the ecosystem gap is obvious.

> Choosing a framework is choosing an ecosystem, not just a language.

Vue 3 has an ecosystem too, but in the specific domain of "financial data tools", React's depth is something Vue cannot match. This is not subjective preference, it is an objective gap.

### Type system: TS + React feels better

Vue 2.7's TypeScript support is "works" but not "comfortable". `defineComponent` type inference breaks at complex boundaries, and type-checking inside templates leans hard on the VS Code plugin.

Vue 3 improved types significantly, but React + TypeScript is type-first by design from the ground up:

```tsx
// Defining props in React — clean and direct
interface AmountInputProps {
  value: number;
  onChange: (value: number) => void;
  max?: number;
  currency?: 'CNY' | 'USD';
}

function AmountInput({ value, onChange, max, currency = 'CNY' }: AmountInputProps) {
  // ...
}
```

To get the same typing experience in Vue, you need `defineProps` + generics + macros, which is heavier cognitive load.

### The pull of React 19

React 19 shipped in December 2024. By April 2025 when we made the call, it had been stable for four months with ample community feedback. Several things were particularly compelling:

Server Components: even though we cannot use RSC right away, React 19's architectural direction makes "load on demand" a framework-level capability.

Actions and `useFormStatus`: form-submission state management finally stops needing hand-rolled scaffolding. Financial tools have tons of forms; this feature deletes a whole category of boilerplate.

The `use()` hook: consuming async data gets more intuitive. Paired with Suspense, loading-state handling is much cleaner.

React Compiler: still experimental, but its direction is "stop worrying about `useMemo` and `useCallback`". That is revolutionary for large-app performance optimization.

> React 19 is not a patch release. It is the React team's response to five years of community pain points.

### The team factor

The last point, and the most pragmatic one: a small team has to lean on the hiring market.

This is not purely a technical decision, it is a people decision. We are a three-person group (including frontend and QA), and we will keep hiring. After migrating to React 19, onboarding new hires is cheaper: there are several times more React engineers than Vue engineers on the market.

The Vue 3 path would mean the whole team re-learning Composition API best practices, Pinia patterns, a new reactivity mental model. For a small team that needs to keep growing, the React path simply has a wider hiring pool.

## My verdict

Distilling the comparison into one line:

> At this moment in April 2025, for a heavy-interaction financial-tool product line, migrating to React 19 is the more rational choice than upgrading to Vue 3.

I acknowledge Vue 3 is a good framework. If this were a content website, marketing pages, or a mid-size admin panel, Vue 3 would be more than enough, even more comfortable. But our scenario, 13 heavy-interaction data tools, financial-grade type rigor, a small team that needs a wider hiring market, every signal points to React.

Selection is about picking the framework that best fits the current scenario, not "the best framework" in the abstract. If anyone takes this post as proof that "React is better than Vue", that is a misreading. I am describing a specific business decision, not a universal conclusion.

## Migration strategy

The decision is made; how to migrate is the next question. 13 tools cannot flip overnight. Our strategy:

1. New tools are written in React 19 directly; no more Vue for new tools.
2. Legacy tools migrate gradually by traffic priority; high-frequency tools first.
3. Component library goes dual-track: the Vue version keeps getting maintenance, a React version is developed in parallel. Both share design tokens and styles.
4. Micro-frontend as a transition: React tools and Vue tools coexist inside one shell via module federation.

This process will take 6-9 months. Not fast, but stable.

## For readers wrestling with the same choice

If you are also torn between Vue and React, my advice:

- Look at the scenario first: content-oriented, Vue; interaction-oriented, React.
- Then look at the team: use what the team knows; do not swim against the current.
- Finally look at the ecosystem: which libraries does your scenario depend on? Check their maturity in both frameworks.

> The framework debate has no standard answer. But your business scenario does.

This post is the close of the tool line's first half and the start of the second half. Future posts will cover concrete technical decisions from the migration: how the micro-frontend is wired, how the component library goes dual-track, how legacy code migrates incrementally. Stay tuned.

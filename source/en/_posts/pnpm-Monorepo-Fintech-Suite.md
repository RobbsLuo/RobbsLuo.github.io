---
title: One Repo for 13 Tools: Building a pnpm Monorepo
date: 2024-06-17 10:00:00
tags:
 - Career
 - Monorepo
 - pnpm
 - Frontend Engineering
categories:
 - Career
lang: en
description: Thirteen financial calculation tools in a single repo. Here is how pnpm workspace keeps dependencies unified, builds fast, and the workspace layout, dependency management and parallel builds actually work.
---

Recently I was handed a chunky task: build the frontend engineering foundation for an entire wealth-tools product line from scratch. Not one or two tools, thirteen of them. Wealth calculation, retirement planning, tax optimization... different shapes, but they all share underlying data, UI conventions and a calculation engine.

> Thirteen tools in thirteen repos is not engineering. It is a disaster.

Day one we made the call: go monorepo. This post walks through how we made pnpm workspace actually work, and the traps we stepped into along the way.

## Why pnpm, not yarn or lerna

The selection did not take long. The reasons are blunt:

- Hard links save disk. Thirteen tools each installing the full Vue + Webpack stack balloons to tens of GB under yarn/npm. pnpm's global store means the same package occupies disk exactly once.
- Strict dependency isolation. yarn workspace hoisting lets a sub-package "accidentally" reach a root-level dependency. pnpm is strict by default; if it is not declared in `package.json`, it cannot be imported. This kills a whole class of "works on my machine" ghosts.
- Speed. Cold install on CI is roughly twice as fast as yarn.

> The first principle of a monorepo: dependencies can be shared, but they must never "bleed".

Lerna has been handed off to the nx team since 2023, and I am not betting its future on a new project. pnpm's built-in `workspace` protocol plus the `-r` recursive flag covers about 90% of what we need.

## How the workspace is layered

We started from a layering principle and built the directory after it:

```
fintech-suite/
├── apps/                  # 13 independently deployable tools
│   ├── wealth-calc/       # a wealth calculation tool
│   ├── retirement-plan/   # a retirement planning tool
│   ├── tax-optimize/      # a tax optimization tool
│   └── ...                # the other 10
├── packages/              # shared internal packages
│   ├── ui-components/     # component library (separate post)
│   ├── calc-engine/       # JS binding layer for the calc engine
│   ├── api-client/        # unified HTTP client + interceptors
│   ├── shared-types/      # TS type definitions
│   └── eslint-config/     # shared ESLint config
├── tools/                 # build/script helpers
│   └── release-cli/
├── pnpm-workspace.yaml
├── package.json
└── .npmrc
```

The key is the three layers: `apps` (final artifacts), `packages` (internal dependencies), `tools` (build-time helpers). Once the layers are set, dependency direction is one-way: `apps` depend on `packages`, packages may depend on each other, but never the reverse.

`pnpm-workspace.yaml` is just a few lines:

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'tools/*'
```

It looks mundane, but it defines the topological backbone of the entire repo: every recursive command, dependency resolution and build order derives from it.

## Dependency management: catalog locks the versions

All 13 tools must run the exact same Vue version, otherwise built artifacts drift in behavior. Early on we relied on manual review of each sub-package's `package.json`, and things slipped through constantly. The `catalog:` protocol introduced in pnpm 9 fixes exactly this: declare versions centrally in the root `pnpm-workspace.yaml`:

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'tools/*'

catalog:
  vue: 2.7.16
  vue-router: 3.6.5
  webpack: 5.91.0
  sass: 1.77.0
  '@vitest/runner': ^1.6.0
```

Sub-packages then reference them like this:

```json
{
  "dependencies": {
    "vue": "catalog:",
    "vue-router": "catalog:"
  }
}
```

> `catalog:` is the right way to manage dependency versions in a monorepo: declare once, apply everywhere.

Bumping a Vue minor only touches the root; every sub-package follows automatically. This matters enormously in finance: dependency versions must be controllable and traceable. Downstream audits need to see "the global Vue version is X.Y.Z", not thirteen packages each drifting on their own.

A few key entries in `.npmrc`:

```ini
# strict isolation, no hoisting backdoors
strict-peer-dependencies=true
# unified lockfile at root
shared-workspace-lockfile=true
# save time on CI
prefer-frozen-lockfile=true
```

## Parallel builds: turbo shrinks the pipeline

A tidy workspace is not the finish line. Thirteen tools built serially takes over ten minutes on CI, and nobody tolerates that. We ended up picking Turbo (the Vercel one), and the payoff was immediate: incremental plus parallel execution dropped average CI from about 12 minutes to just over 3.

> For a thirteen-tool repo, this is the difference between "merge in ten minutes" and "stuck for half a day".

Root `turbo.json`:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {}
  }
}
```

The `^` in `^build` is the key: it means "build my upstream dependencies first, then build me". If `wealth-calc` depends on `@suite/ui-components`, turbo automatically builds `ui-components` before `wealth-calc`, and tools with no dependency on each other run in parallel.

Full build:

```bash
pnpm turbo run build
```

Far more useful is selective builds: only the packages affected by a git diff and their downstream.

```bash
# only build what changed since HEAD^
pnpm turbo run build --filter=...[HEAD^]
```

This `--filter` is a lifesaver during PR review: if a PR only touches the retirement planning tool, CI builds just that tool and its dependencies. The other twelve are not touched at all.

## Traps we hit

Trap one: reversed dependency direction. Early on we placed a piece of shared logic inside one of the apps. Two other apps started depending on it, and the build order broke down. Turbo reported a circular dependency and it took ages to locate. Lesson: anything shared goes up into `packages`; apps never depend on each other.

Trap two: catalog bumps require full regression. Once we bumped a vue-router major and only ran the affected tool's tests. Two other tools' e2e suites collapsed. The iron rule since then: any catalog bump touching a runtime dependency triggers a full e2e run, no shortcuts.

Trap three: cross-workspace references must use an explicit protocol. pnpm refuses to let you write a bare version number for a workspace-internal package. You must write `workspace:*` or `workspace:^1.0.0`:

```json
{
  "dependencies": {
    "@suite/ui-components": "workspace:*",
    "@suite/calc-engine": "workspace:^1.0.0"
  }
}
```

It felt verbose at first, but it is actually a good thing. It forces you to be explicit every time about "is this internal or external", and removes a lot of sneaky dependencies.

## Wrapping up

With monorepos, once the structure is right, everything downstream flows. pnpm's workspace + catalog + strict isolation, combined with Turbo's parallel builds, is the most comfortable combination I have found at the thirteen-tool scale.

The next post covers the backend: how thirteen tools share a single Spring Boot API surface. Stay tuned.

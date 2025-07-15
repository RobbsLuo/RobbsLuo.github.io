---
title: Migration Retrospective - Switching Webpack to Vite
date: 2025-07-15 11:00:00
tags:
  - Career
  - Vite
  - Webpack
categories:
  - Career
description: Alongside migrating 13 tools from Vue 2.7 to React 19, the build tooling also moved from Webpack to Vite. This post covers the pitfalls I hit: dependency compatibility, HMR, path aliases, build output, CSS Modules, and more.
lang: en
---

The previous two posts covered component migration and Skill distillation. There is one thread of the whole migration I have not mentioned yet: the build tooling also moved from Webpack to Vite.

Component migration was handled by the AI agent, but the build tool switch could not be fully delegated to AI, because every pitfall was tied to a specific dependency or a specific environment that the AI could not guess. This part I basically walked through one hole at a time.

This post records those holes so that whoever migrates next can skip some of them.

## Why Drop Webpack

The reasons are not complicated:

- Dev startup was too slow: a monorepo of 13 tools meant a Webpack cold start of over 40 seconds, and HMR response kept getting slower.
- Config was too heavy: a pile of loaders and plugins interdependent on each other, where touching one thing risked breaking another.
- The React 19 ecosystem is friendlier to the newer build tools: lots of new libraries default to Vite in their docs, and the Webpack adapter had become extra work.

> The payoff of switching build tools is not "a bit faster." Vite's dev server starts in about a second, and once you feel that gap you cannot go back.

But the cost is real too. Here are the pitfalls I actually hit.

## Pitfall 1: CommonJS Dependency Pre-Bundling

Vite's dev server is based on ESM, but many packages in node_modules are still CommonJS. Vite automatically pre-bundles these with esbuild, but some packages break after pre-bundling.

The classic case: a dependency under CommonJS exports `{ default: xxx, ...namedExports }`. After esbuild pre-bundles it, `default` cannot be resolved and at runtime you get `xxx is not a function`.

The fix is to add such packages explicitly to `optimizeDeps.include` to force Vite to handle them:

```ts
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: [
      // these are CJS; without an explicit declaration they throw at runtime
      'some-cjs-lib',
      '@some/legacy-package',
    ],
  },
})
```

What is worse is that some packages **only blow up during the production build** (dev is fine, but at build time Rollup handles CJS differently from esbuild). The only way to catch those is to run `vite build` once before shipping.

## Pitfall 2: Path Aliases

Webpack's `resolve.alias` and Vite's `resolve.alias` are configured differently. They share a name but behave differently, which is the most deceptive part.

```ts
// Webpack style
resolve: {
  alias: {
    '@': path.resolve(__dirname, 'src'),
  },
}

// Vite style (note the different format)
resolve: {
  alias: {
    '@': path.resolve(__dirname, 'src'),
    // trailing slash matters; omit it and matching breaks
    '@/': path.resolve(__dirname, 'src') + '/',
  },
}
```

What I actually hit: one tool mixed `@/components/Button` and `@components/Button`. Webpack resolved both; Vite only accepted the first. Unifying alias usage has to happen before the migration, otherwise you get `module not found` everywhere afterward.

Also, tsconfig.json's `paths` need to be updated at the same time, or the IDE's type hints drift.

## Pitfall 3: CSS Modules Output Differences

We standardized on CSS Modules, as decided in the migration Skill. But Webpack and Vite handle CSS Modules differently:

```css
/* Button.module.css */
.title { color: red; }
```

- Webpack: the generated class name is `Button_title__xxxxx` (prefixed with the file name).
- Vite: default is `_title_xxxxx` (no file-name prefix).

Sounds harmless? It explodes if your e2e tests use class names as selectors. Some of our Playwright tests fell back to class names where `data-testid` was missing, and they all failed after the migration.

The fix is to explicitly configure Vite's CSS Modules output rule:

```ts
css: {
  modules: {
    generateScopedName: '[name]_[local]__[hash:base64:5]',
  },
},
```

That aligns the output format with Webpack. But I realized later that tests should not depend on implementation details (class names are an implementation detail), so I swapped every class-name selector in the e2e suite for `data-testid`. That is the right way.

## Pitfall 4: HMR Behavior Differences

Vite's HMR is based on native ESM and is much faster than Webpack, but the behavior differs:

- Webpack's HMR preserves component state (paired with react-refresh).
- Vite + react-refresh is the same in most cases, but when you edit a store file, Vite does a full page reload whereas Webpack might only refresh the component.

This really affects the feel during debugging. You change one Zustand store and the whole page reloads, losing the form data you had just filled in. Our answer was to split stores finer: editing one store should not affect state on unrelated pages. That is a better architecture practice in its own right.

## Pitfall 5: Environment Variables

Webpack injects env vars via `DefinePlugin`; Vite uses `import.meta.env`. The migration agent can help convert part of this, but the way values are read at runtime changes, so you have to grep globally:

```ts
// Webpack
const apiBase = process.env.API_BASE

// Vite
const apiBase = import.meta.env.VITE_API_BASE
```

Note that Vite's env vars must start with `VITE_` to be exposed to the frontend. Several of our env vars were not renamed, resolved to `undefined` in production, and took a while to track down.

```bash
# .env file
VITE_API_BASE=https://api.example.com  # correct
API_BASE=https://api.example.com        # wrong, frontend cannot see it
```

## Pitfall 6: Production Build Output Structure

Vite by default puts everything into `dist/assets/`, with hashed file names. Webpack is similar, but the chunk-splitting strategy differs.

Webpack's `splitChunks` config maps to Vite's `build.rollupOptions.output.manualChunks`. If you had manually configured chunk splitting before, it has to be rewritten. Our approach: do not manually split at first; run Vite's defaults, look at the actual output, then tune. In most cases the defaults are good enough; do not over-optimize.

## Retrospective: How to Migrate Smoothly

Summarizing, if I were to do it again, I would follow this order:

1. Unify aliases and path conventions first - things that do not depend on the build tool.
2. Remove implementation-detail dependencies from e2e (swap class-name selectors for data-testid).
3. Get Vite running in dev first, then run a build, then compare output.
4. Unify env vars to the `VITE_` prefix, and grep through the whole codebase.
5. Staged rollout: switch one tool to Vite first, let it stabilize, then fan out.

> The hard part of build-tool migration is not the config itself. It is that you think you are done, and some edge case only surfaces in production. Dev passing does not mean it works. Run a full build plus e2e before shipping.

Plenty of pitfalls in this one, but each was real. Build-tool migration, unlike component migration, cannot be handed off to AI in bulk. It is more about the human's understanding of and judgment about the toolchain. AI can write the config for you, but the experience of hitting these pitfalls has to be earned yourself.

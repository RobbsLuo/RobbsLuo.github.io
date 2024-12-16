---
title: A Component Library: UMD and Package at Once
date: 2024-12-16 15:00:00
tags:
 - Career
 - Component Library
 - UMD
 - Cross-Tool Reuse
categories:
 - Career
lang: en
description: Thirteen tools need to share one set of UI components, but integration differs — some embed via UMD, others import as an npm package. 10+ components, two kinds of artifacts, one source tree.
---

Halfway through the tool line, an unavoidable question surfaced: the UI components across all 13 tools have to be unified.

The amount input in the wealth calculation tool, the year picker in the retirement planning tool, the rate display card in the tax tool. If every tool writes its own version, visual inconsistency is the small problem. Behavioral inconsistency is the real danger. In a financial context, the formatting logic of an amount input must be byte-for-byte identical across all 13 tools.

So we built an internal component library: 10+ components, one source tree, emitting two kinds of artifacts simultaneously, a UMD bundle and an npm package. Here is how.

## Why two artifacts

We could not pick just one.

Only an npm package is not enough. Several tools are legacy projects whose build config cannot be easily changed, so they cannot `import` directly. What they need is a UMD artifact dropped in via a `<script>` tag.

Only UMD is not enough. New tools in the monorepo use the ES module system, and UMD's tree-shaking is poor, which bloats the bundle.

> One source tree, two artifacts. It looks like "we want both," but reality forced it on us.

## Directory layout of the library

The library itself is a package inside the monorepo:

```
packages/ui-components/
├── src/
│   ├── components/
│   │   ├── AmountInput/
│   │   ├── YearPicker/
│   │   ├── RateCard/
│   │   ├── ResultTable/
│   │   └── ...
│   ├── styles/
│   │   ├── tokens.scss          # design tokens (colors, spacing)
│   │   └── theme.scss           # theme variables
│   ├── utils/
│   │   └── format.ts            # formatting utilities
│   └── index.ts                 # unified entry
├── vite.config.ts               # build config (the crux)
├── package.json
└── tsconfig.json
```

Each component gets its own directory containing `index.vue` and `index.test.ts`. This structure makes per-component exports easy; more on that later.

## Vite Library Mode: two artifacts in one build

The core is `vite.config.ts`. Vite's library mode supports emitting multiple formats at once:

```typescript
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import dts from 'vite-plugin-dts';
import { resolve } from 'path';

export default defineConfig({
  plugins: [
    vue(),
    dts({ insertTypesEntry: true }),
  ],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'SuiteUI',                // UMD global variable
      formats: ['es', 'umd'],         // emit both ES and UMD
      fileName: (format) => `suite-ui.${format}.js`,
    },
    rollupOptions: {
      external: ['vue'],              // do not bundle Vue
      output: {
        globals: { vue: 'Vue' },
        assetFileNames: 'suite-ui.[ext]',
      },
    },
  },
});
```

**The crux is `formats: ['es', 'umd']`**: one build emits:

- `suite-ui.es.js`: ES module for tools inside the monorepo to `import`
- `suite-ui.umd.js`: UMD bundle for legacy projects to include via `<script>`
- `suite-ui.css`: standalone stylesheet (UMD mode requires CSS to be exported separately)

## The exports field in package.json

The `exports` field lets each consumption style automatically pick the right artifact:

```json
{
  "name": "@suite/ui-components",
  "version": "1.4.0",
  "main": "./dist/suite-ui.umd.js",
  "module": "./dist/suite-ui.es.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/suite-ui.es.js",
      "require": "./dist/suite-ui.umd.js",
      "types": "./dist/index.d.ts"
    },
    "./styles.css": "./dist/suite-ui.css",
    "./package.json": "./package.json"
  },
  "sideEffects": ["*.css", "*.scss"],
  "files": ["dist/"]
}
```

> The `sideEffects` field must be set; otherwise tree-shaking in consumers will treat CSS as side-effect code and strip it all out.

Tools inside the monorepo consume it like this:

```json
{
  "dependencies": {
    "@suite/ui-components": "workspace:*"
  }
}
```

```typescript
// ES module style (new tools)
import { AmountInput, RateCard } from '@suite/ui-components';
import '@suite/ui-components/styles.css';
```

Legacy projects do this:

```html
<!-- UMD style (legacy) -->
<script src="/vendor/vue.runtime.js"></script>
<script src="/vendor/suite-ui.umd.js"></script>
<link rel="stylesheet" href="/vendor/suite-ui.css" />

<script>
  const { AmountInput } = window.SuiteUI;
  // register into Vue...
</script>
```

## On-demand imports: subpath exports

Once the component count grew past 15, full imports became too heavy. We added subpath exports so consumers only pull what they need:

```json
{
  "exports": {
    ".": {
      "import": "./dist/suite-ui.es.js",
      "require": "./dist/suite-ui.umd.js"
    },
    "./AmountInput": {
      "import": "./dist/components/AmountInput/index.es.js",
      "require": "./dist/components/AmountInput/index.umd.js"
    },
    "./RateCard": {
      "import": "./dist/components/RateCard/index.es.js",
      "require": "./dist/components/RateCard/index.umd.js"
    }
  }
}
```

```typescript
// import only the amount input, not the whole library
import AmountInput from '@suite/ui-components/AmountInput';
```

This means the build config needs multiple entries; update Vite:

```typescript
build: {
  lib: {
    entry: {
      index: resolve(__dirname, 'src/index.ts'),
      'components/AmountInput/index': resolve(__dirname, 'src/components/AmountInput/index.ts'),
      'components/RateCard/index': resolve(__dirname, 'src/components/RateCard/index.ts'),
      // ... other components
    },
    formats: ['es', 'umd'],
    fileName: (format, entryName) => `${entryName}.${format}.js`,
  },
}
```

## Design tokens: the root of visual consistency

All 13 tools must not drift visually. We define a set of design tokens in `tokens.scss`:

```scss
// semantic colors
$color-primary: #1677ff;
$color-success: #52c41a;
$color-warning: #faad14;
$color-danger: #ff4d4f;

// amount-specific colors (finance needs to distinguish positive/negative)
$color-amount-positive: #52c41a;
$color-amount-negative: #ff4d4f;

// spacing
$spacing-unit: 4px;
$spacing-sm: $spacing-unit * 2;
$spacing-md: $spacing-unit * 4;
$spacing-lg: $spacing-unit * 6;

// border radius
$radius-sm: 2px;
$radius-md: 4px;
$radius-lg: 8px;
```

These tokens compile into CSS variables, and consumers can override the theme at runtime:

```scss
:root {
  --suite-color-primary: #{$color-primary};
  --suite-color-amount-positive: #{$color-amount-positive};
}
```

```css
/* consumer overrides theme */
:root {
  --suite-color-primary: #722ed1;
}
```

## Traps we hit

Trap one: UMD mode did not auto-import CSS. On the ES module side, `import '@suite/ui-components/styles.css'` works. On the UMD side, users kept forgetting to include the CSS, and the page rendered blank. We added an iron rule in the docs: UMD consumers must include the CSS file or layout will break.

Trap two: Vue version mismatch caused runtime errors. The library externalizes Vue, but if the consumer's Vue version does not match what the library was developed against, runtime crashes happen. We locked the range in peerDependencies:

```json
{
  "peerDependencies": {
    "vue": "^2.7.0"
  }
}
```

Trap three: per-component bundle size went up instead of down. Early on, multi-entry builds duplicated the shared utils and styles in every component. Only after using Vite's `manualChunks` to extract shared dependencies did the size actually drop.

## Wrapping up

One source tree emitting both UMD and an npm package comes down to three things:

- Vite library mode multi-format output (`formats: ['es', 'umd']`)
- package.json exports for precise routing (consumers automatically get the right artifact)
- Design tokens for unified visuals (CSS variables support theme overrides)

A component library is an engineering delivery problem more than a writing components problem. Next post: the quality baseline, how we pushed test coverage to 90%.

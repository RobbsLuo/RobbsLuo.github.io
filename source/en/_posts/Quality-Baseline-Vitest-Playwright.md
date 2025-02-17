---
title: Quality Baseline: Vitest, Playwright, ESLint at 90% Coverage
date: 2025-02-17 10:30:00
tags:
 - Career
 - Vitest
 - Playwright
 - ESLint
 - Test Coverage
categories:
 - Career
lang: en
description: Thirteen financial tools cannot survive without a test baseline. This post covers how we use Vitest for unit tests, Playwright for e2e, and ESLint for style — pushing core-module coverage above 90%.
---

Eight months into the tool line, one number made me sit up: unit test coverage on core modules was only 47%.

Thirteen financial calculation tools, less than half covered. More than half the codebase could be changed with zero visibility into collateral damage. In a financial context that is a ticking bomb. You cannot rely on manual review to guarantee correctness of financial calculations.

So we spent three months building a quality baseline: Vitest for unit tests, Playwright for e2e, ESLint for style. By year-end, core-module coverage hit 93%. Here is how.

## Why Vitest, not Jest

With Vue 2.7 + Webpack + TypeScript, the Jest-versus-Vitest decision took some deliberation. Vitest won for these reasons:

- Zero config with the Vite ecosystem. Our component library and new tools all use Vite. Vitest reuses the Vite config, so we avoid maintaining a separate jest.config.
- Speed. Vitest uses esbuild for transpilation by default, far faster than Jest's babel pipeline. After migrating, full-suite runtime dropped from 90 seconds to 35.
- API almost identical to Jest. `describe`, `it`, `expect`, `vi.mock`; migration cost is minimal.

> If you are already on Vite, Vitest is essentially the only reasonable answer.

## Vitest configuration

Root-level `vitest.config.ts`, managed at the workspace level:

```typescript
import { defineConfig } from 'vitest/config';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    coverage: {
      provider: 'v8',              // v8 is much faster than istanbul
      reporter: ['text', 'lcov', 'html'],
      reportsDirectory: './coverage',
      // this is the threshold — 90% for core modules
      thresholds: {
        statements: 90,
        branches: 85,
        functions: 90,
        lines: 90,
      },
      include: ['src/**/*.{ts,vue,tsx}'],
      exclude: ['src/**/*.d.ts', 'src/**/__mocks__/**'],
    },
    setupFiles: ['./test/setup.ts'],
  },
});
```

**`thresholds` is the crux**: it is not a suggestion, it is a hard gate. If coverage falls below these numbers, `vitest --coverage` exits non-zero and CI fails.

> Coverage thresholds are "fail to merge if you miss," not "try to hit."

## Unit test example: a financial component

Take the `AmountInput` component. It looks simple, but in a financial context it has a pile of edge cases:

```typescript
// src/components/AmountInput/AmountInput.test.ts
import { mount } from '@vue/test-utils';
import { describe, it, expect } from 'vitest';
import AmountInput from './AmountInput.vue';

describe('AmountInput', () => {
  it('formats input with thousands separators', async () => {
    const wrapper = mount(AmountInput);
    await wrapper.find('input').setValue('1234567.89');
    expect(wrapper.find('input').element.value).toBe('1,234,567.89');
  });

  it('rejects non-numeric input', async () => {
    const wrapper = mount(AmountInput);
    await wrapper.find('input').setValue('abc');
    expect(wrapper.emitted('update:modelValue')?.[0]).toEqual(['']);
  });

  it('marks negative amounts in red', async () => {
    const wrapper = mount(AmountInput);
    await wrapper.find('input').setValue('-500');
    expect(wrapper.classes()).toContain('amount-input--negative');
  });

  it('emits error event when exceeding max', async () => {
    const wrapper = mount(AmountInput, {
      props: { max: 1000000 },
    });
    await wrapper.find('input').setValue('2000000');
    expect(wrapper.emitted('error')).toBeTruthy();
  });
});
```

Note that these cases are not imagined from thin air; every one corresponds to a real financial business constraint. Thousands-separator formatting, negative highlighting, ceiling validation: if any of these breaks in a tool, the client sees wrong numbers.

## Playwright: end-to-end testing

Unit tests cover component-level logic, but the full flow across all 13 tools (login → pick client → fill form → click calculate → read report) needs e2e tests.

Playwright config lives at the repo root:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: process.env.CI ? [['github'], ['html']] : 'list',
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

A typical e2e: the full flow of the wealth calculation tool.

```typescript
// e2e/wealth-calc.spec.ts
import { test, expect } from '@playwright/test';

test('wealth calculation full flow', async ({ page }) => {
  await page.goto('/wealth-calc');

  // pick a client
  await page.click('[data-testid="client-select"]');
  await page.click('[data-testid="client-option-0"]');

  // fill calculation params (desensitized: no real business fields)
  await page.fill('[data-testid="input-amount"]', '500000');
  await page.fill('[data-testid="input-years"]', '20');

  // hit calculate
  await page.click('[data-testid="calc-button"]');

  // wait for the result to appear
  await expect(page.locator('[data-testid="result-table"]')).toBeVisible();

  // verify the table has data rows
  const rows = page.locator('[data-testid="result-table"] tbody tr');
  await expect(rows).toHaveCount(20);   // 20 years = 20 rows

  // verify the export button is enabled
  await expect(page.locator('[data-testid="export-button"]')).toBeEnabled();
});
```

**The soul of e2e tests is `data-testid`.** We set a rule: every element needing interaction testing must carry a `data-testid`. CSS class names may change during refactors, but testids must not, otherwise tests collapse en masse every refactor.

## How coverage gates are enforced

Configuration alone is not enough; someone has to hold the line. We do it in two layers:

Layer one: pre-commit hook. Runs lint plus the affected package's unit tests; blocks commit on failure:

```bash
#!/bin/bash
# .husky/pre-commit
pnpm lint-staged
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,vue,tsx}": ["eslint --fix", "vitest related --run"]
  }
}
```

Layer two: CI pipeline. Full unit tests + coverage check + e2e; any failure blocks the PR:

```yaml
# .github/workflows/quality.yml
name: Quality Gate
on: [pull_request]

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --coverage
      # threshold miss = automatic failure

  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps
      - run: pnpm e2e
```

Coverage is a number, but behind the number is discipline. Without CI enforcement, pretty configs mean nothing.

## ESLint: codify the rules into the machine

Human code review misses things. Machines do not. Our ESLint config lives in a shared package:

```javascript
// packages/eslint-config/index.js
module.exports = {
  extends: [
    'eslint:recommended',
    '@vue/eslint-config-typescript',
    '@vue/eslint-config-typescript/strict',
  ],
  rules: {
    // no leftover console.log in financial code
    'no-console': ['error', { allow: ['warn', 'error'] }],
    // ban any — financial types must be explicit
    '@typescript-eslint/no-explicit-any': 'error',
    // ban parseFloat/parseInt — amounts must use Decimal
    'no-restricted-globals': ['error', { name: 'parseFloat', message: 'Use BigDecimal or decimal.js for amounts' }],
  },
};
```

The last `no-restricted-globals` rule is purpose-built for finance. It forcibly prevents `parseFloat` from appearing anywhere in the codebase, because floating-point precision errors are lethal in financial contexts.

## Wrapping up

A quality baseline has no silver bullet. It comes down to three things done properly:

- Vitest for unit tests, with coverage thresholds locked (core modules 90%+)
- Playwright for e2e, covering every critical flow
- ESLint codifying conventions into CI (what humans miss, machines must catch)

> Tests are not "write the feature, then add tests." They are "write the test alongside the feature," or even write the test first.

This post wraps up the first half of the tool line, the architecture foundation. The next post is a turning point: why I decided to migrate this entire Vue 2.7 system to React 19.

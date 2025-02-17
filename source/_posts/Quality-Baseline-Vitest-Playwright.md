---
title: 质量基线：Vitest + Playwright + ESLint 把覆盖率拉到 90%
date: 2025-02-17 10:30:00
tags:
 - Career
 - Vitest
 - Playwright
 - ESLint
 - 测试覆盖率
categories:
 - Career
description: 13 个金融工具没有测试基线是不行的。这篇讲我们怎么用 Vitest 跑单测、Playwright 跑 e2e、ESLint 守风格，核心模块覆盖率拉到 90%+。
---

工具线跑到第 8 个月的时候，有个数字让我坐不住了：核心模块的单元测试覆盖率只有 47%。

13 个金融计算工具，覆盖率不到一半。换句话说，超过一半的代码改了之后根本不知道有没有把别的东西改坏。这在金融场景里是定时炸弹——你不可能靠人工 review 保证金融计算的正确性。

于是我们花了三个月建了一套质量基线：Vitest 跑单测、Playwright 跑 e2e、ESLint 守代码风格。到年底，核心模块覆盖率拉到 93%。这篇讲怎么做的。

## 为什么是 Vitest，不是 Jest

Vue 2.7 + Webpack + TypeScript 这套技术栈，选 Jest 还是 Vitest，我纠结过一阵。最后选 Vitest 的原因：

- 和 Vite 生态零配置：我们的组件库和新工具都在用 Vite，Vitest 直接复用 Vite 的配置，不用单独维护一份 jest.config。
- 速度：Vitest 默认用 esbuild 做转译，比 Jest 的 babel 快一大截。我们的测试套件从 Jest 迁过来后，跑完全量从 90 秒降到 35 秒。
- API 几乎兼容 Jest：`describe`、`it`、`expect`、`vi.mock` 这些写法迁移成本极低。

如果你已经在用 Vite，选 Vitest 几乎是唯一合理的答案。

## Vitest 配置

根目录的 `vitest.config.ts`，workspace 级别统一管理：

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
      provider: 'v8',              // 用 v8 比 istanbul 快很多
      reporter: ['text', 'lcov', 'html'],
      reportsDirectory: './coverage',
      // 这就是覆盖率门槛——核心模块 90%
      thresholds: {
        statements: 90,
        branches: 85,
        functions: 90,
        lines: 90,
      },
      // 只统计 src/ 下的代码，不算测试文件和配置
      include: ['src/**/*.{ts,vue,tsx}'],
      exclude: ['src/**/*.d.ts', 'src/**/__mocks__/**'],
    },
    setupFiles: ['./test/setup.ts'],
  },
});
```

`thresholds` 这块是关键——它不是建议，是硬性门槛。覆盖率低于这个数字，`vitest --coverage` 直接退出码非零，CI 会挂掉。

覆盖率门槛不是"尽量达到"，是"达不到就别想合并"。

## 单元测试示例：金融组件

拿金额输入框 `AmountInput` 举例。这个组件看似简单，但金融场景下有一堆边界情况要覆盖：

```typescript
// src/components/AmountInput/AmountInput.test.ts
import { mount } from '@vue/test-utils';
import { describe, it, expect } from 'vitest';
import AmountInput from './AmountInput.vue';

describe('AmountInput', () => {
  it('格式化输入为千分位', async () => {
    const wrapper = mount(AmountInput);
    await wrapper.find('input').setValue('1234567.89');
    expect(wrapper.find('input').element.value).toBe('1,234,567.89');
  });

  it('拒绝非数字输入', async () => {
    const wrapper = mount(AmountInput);
    await wrapper.find('input').setValue('abc');
    expect(wrapper.emitted('update:modelValue')?.[0]).toEqual(['']);
  });

  it('负数金额标红', async () => {
    const wrapper = mount(AmountInput);
    await wrapper.find('input').setValue('-500');
    expect(wrapper.classes()).toContain('amount-input--negative');
  });

  it('超过最大值时触发 error 事件', async () => {
    const wrapper = mount(AmountInput, {
      props: { max: 1000000 },
    });
    await wrapper.find('input').setValue('2000000');
    expect(wrapper.emitted('error')).toBeTruthy();
  });
});
```

注意这些测试用例不是拍脑袋想的——每一个都对应一个真实的金融业务约束。千分位格式化、负数标红、上限校验，这些逻辑如果在某个工具里坏了，客户看到的数字就是错的。

## Playwright：端到端测试

单测覆盖了组件级别的逻辑，但 13 个工具端到端的流程（用户登录 → 选客户 → 填表 → 点计算 → 看报表）需要 e2e 测试来守。

Playwright 的配置放在仓库根目录：

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

一个典型的 e2e 测试——理财计算工具的完整流程：

```typescript
// e2e/wealth-calc.spec.ts
import { test, expect } from '@playwright/test';

test('理财计算完整流程', async ({ page }) => {
  await page.goto('/wealth-calc');

  // 选客户
  await page.click('[data-testid="client-select"]');
  await page.click('[data-testid="client-option-0"]');

  // 填入计算参数（脱敏：不写真实业务字段）
  await page.fill('[data-testid="input-amount"]', '500000');
  await page.fill('[data-testid="input-years"]', '20');

  // 点击计算
  await page.click('[data-testid="calc-button"]');

  // 等待结果出现
  await expect(page.locator('[data-testid="result-table"]')).toBeVisible();

  // 验证结果表格有数据行
  const rows = page.locator('[data-testid="result-table"] tbody tr');
  await expect(rows).toHaveCount(20);   // 20 年 = 20 行

  // 验证导出按钮可点击
  await expect(page.locator('[data-testid="export-button"]')).toBeEnabled();
});
```

e2e 测试的灵魂在于 `data-testid`。我们定了一个规范：所有需要测试交互的元素必须有 `data-testid` 属性，CSS 类名可以变，但 testid 不能变。这样重构时测试不会大面积失效。

## 覆盖率门槛怎么执行

光有配置不够，得有人守。我们的做法是分两层：

第一层是 pre-commit hook。跑 lint + 受影响包的单测，不通过不让提交：

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

第二层是 CI 流水线。跑全量单测 + 覆盖率检查 + e2e，任何一项挂了 PR 不能合并：

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
      # threshold 不过会自动失败

  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps
      - run: pnpm e2e
```

覆盖率是个数字，但数字背后是纪律。没有 CI 卡门，配置写得再漂亮也没用。

## ESLint：把规范写进机器里

人的 code review 会遗漏，机器不会。我们的 ESLint 配置在共享包里：

```javascript
// packages/eslint-config/index.js
module.exports = {
  extends: [
    'eslint:recommended',
    '@vue/eslint-config-typescript',
    '@vue/eslint-config-typescript/strict',
  ],
  rules: {
    // 金融场景禁止 console.log 残留
    'no-console': ['error', { allow: ['warn', 'error'] }],
    // 禁止 any，金融类型必须明确
    '@typescript-eslint/no-explicit-any': 'error',
    // 禁止 parseFloat/parseInt，金额必须用 Decimal
    'no-restricted-globals': ['error', { name: 'parseFloat', message: '金额计算请使用 BigDecimal 或 decimal.js' }],
  },
};
```

最后那条 `no-restricted-globals` 是专门为金融场景加的——强制阻止 `parseFloat` 出现在代码里，因为浮点精度问题在金融场景下是致命的。

## 小结

质量基线这件事，没有银弹，就是三件事做到位：

- Vitest 跑单测，覆盖率门槛卡死（核心模块 90%+）
- Playwright 跑 e2e，关键流程全覆盖
- ESLint 把规范写进 CI（人能漏的机器不能漏）

测试不是"写完功能再补"，是和功能一起写，甚至先写测试再写实现。

到这一篇为止，工具线的前半段（架构搭建）基本讲完了。下一篇是一个转折——为什么我决定把这套 Vue 2.7 的体系迁到 React 19。

---
title: 组件库：10+ 组件怎么同时支持 UMD 和组件包
date: 2024-12-16 15:00:00
tags:
 - Career
 - 组件库
 - UMD
 - 跨工具复用
categories:
 - Career
description: 13 个工具要复用一套 UI 组件，但集成方式不同——有的要 UMD 直接嵌，有的要走 npm 包 import。10+ 个组件，两种产物，一套源码。
---

工具线做到中期，一个绕不开的问题冒出来了：13 个工具的 UI 组件得统一。

理财计算工具里的金额输入框、退休规划工具里的年份选择器、税务工具里的税率展示卡片……这些东西如果每个工具各写一套，视觉不一致是小事，行为不一致才是大事。金融场景里，一个金额输入框的格式化逻辑必须 13 个工具完全一致。

所以我们做了一个内部组件库，10+ 个组件，一套源码，同时输出两种产物：UMD 包和 npm 包。这篇讲怎么做到的。

## 为什么是两种产物

先说清楚为什么不能只选一种。

只有 npm 包不行——因为有几个工具是早期遗留项目，构建配置改不动，没法直接 import。它们需要的是往 HTML 里丢一个 `<script>` 标签就能用的 UMD 产物。

只有 UMD 不行——因为新工具走的是 monorepo 里的 ES 模块体系，UMD 的 tree-shaking 效果差，包体积会膨胀。

一套源码，两种产物，看起来是"都要"，其实是被现实逼的。

## 组件库的目录结构

组件库本身也是 monorepo 里的一个 package：

```
packages/ui-components/
├── src/
│   ├── components/
│   │   ├── AmountInput/         # 金额输入（金融专用）
│   │   ├── YearPicker/          # 年份选择
│   │   ├── RateCard/            # 比率展示卡片
│   │   ├── ResultTable/         # 结果表格
│   │   └── ...
│   ├── styles/
│   │   ├── tokens.scss          # 设计 token（颜色、间距）
│   │   └── theme.scss           # 主题变量
│   ├── utils/
│   │   └── format.ts            # 格式化工具（金额、日期等）
│   └── index.ts                 # 统一出口
├── vite.config.ts               # 构建配置（关键！）
├── package.json
└── tsconfig.json
```

每个组件一个目录，内含 `index.vue`（组件实现）和 `index.test.ts`（单元测试）。这种结构方便单独导出——后面讲。

## Vite Library Mode：一次构建两种产物

配置重心在 `vite.config.ts`。Vite 的 library mode 支持同时输出多种格式：

```typescript
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import dts from 'vite-plugin-dts';
import { resolve } from 'path';

export default defineConfig({
  plugins: [
    vue(),
    dts({ insertTypesEntry: true }),  // 自动生成 .d.ts
  ],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'SuiteUI',                // UMD 全局变量名
      formats: ['es', 'umd'],         // 同时输出 ES 和 UMD
      fileName: (format) => `suite-ui.${format}.js`,
    },
    rollupOptions: {
      // 外部化 Vue，不打包进去
      external: ['vue'],
      output: {
        globals: { vue: 'Vue' },
        // UMD 需要导出 CSS
        assetFileNames: 'suite-ui.[ext]',
      },
    },
  },
});
```

关键是 `formats: ['es', 'umd']` 这一行——一次构建，同时生成：

- `suite-ui.es.js`：给 monorepo 内部工具 import 的 ES 模块
- `suite-ui.umd.js`：给遗留项目 `<script>` 标签引入的 UMD 包
- `suite-ui.css`：样式文件（UMD 模式必须单独导出 CSS）

## package.json 的 exports 字段

`package.json` 里用 `exports` 字段让两种消费方式自动选对产物：

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

`sideEffects` 这个字段必须配，否则消费者的 tree-shaking 会把 CSS 当副作用代码全部删掉。

monorepo 里的工具这样引用：

```json
{
  "dependencies": {
    "@suite/ui-components": "workspace:*"
  }
}
```

```typescript
// ES 模块方式（新工具）
import { AmountInput, RateCard } from '@suite/ui-components';
import '@suite/ui-components/styles.css';
```

遗留项目这样用：

```html
<!-- UMD 方式（老项目） -->
<script src="/vendor/vue.runtime.js"></script>
<script src="/vendor/suite-ui.umd.js"></script>
<link rel="stylesheet" href="/vendor/suite-ui.css" />

<script>
  const { AmountInput } = window.SuiteUI;
  // 注册到 Vue...
</script>
```

## 按需引入：子路径导出

到后面组件多了（15+ 个），全量引入太重。我们做了子路径导出，让消费者只引需要的组件：

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
// 只引金额输入框，不打包整个库
import AmountInput from '@suite/ui-components/AmountInput';
```

这意味着构建配置要做多入口——Vite 配置改一下：

```typescript
build: {
  lib: {
    entry: {
      index: resolve(__dirname, 'src/index.ts'),
      'components/AmountInput/index': resolve(__dirname, 'src/components/AmountInput/index.ts'),
      'components/RateCard/index': resolve(__dirname, 'src/components/RateCard/index.ts'),
      // ... 其他组件
    },
    formats: ['es', 'umd'],
    fileName: (format, entryName) => `${entryName}.${format}.js`,
  },
}
```

## 设计 Token：样式一致性的底子

13 个工具的视觉不能飘。我们在 `tokens.scss` 里定义了一套设计 token：

```scss
// 语义色
$color-primary: #1677ff;
$color-success: #52c41a;
$color-warning: #faad14;
$color-danger: #ff4d4f;

// 金额专用色（金融场景区分正负）
$color-amount-positive: #52c41a;
$color-amount-negative: #ff4d4f;

// 间距
$spacing-unit: 4px;
$spacing-sm: $spacing-unit * 2;
$spacing-md: $spacing-unit * 4;
$spacing-lg: $spacing-unit * 6;

// 圆角
$radius-sm: 2px;
$radius-md: 4px;
$radius-lg: 8px;
```

这些 token 编译进 CSS 变量，消费者可以在运行时覆盖主题：

```scss
:root {
  --suite-color-primary: #{$color-primary};
  --suite-color-amount-positive: #{$color-amount-positive};
  // ...
}
```

```css
/* 消费者覆盖主题 */
:root {
  --suite-color-primary: #722ed1;  /* 换成紫色 */
}
```

## 踩过的坑

坑一：UMD 模式下 CSS 没自动引入。ES 模块那边 `import '@suite/ui-components/styles.css'` 就行了，但 UMD 那边用户经常忘了引 CSS，页面全白。后来我们在文档里加了一句铁律：UMD 引入必须同时引 CSS，否则布局必崩。

坑二：Vue 版本不一致导致运行时报错。组件库 external 掉了 Vue，但如果消费者用的 Vue 版本和库开发时的不一致，运行时可能炸。我们在 peerDependencies 里锁死了范围：

```json
{
  "peerDependencies": {
    "vue": "^2.7.0"
  }
}
```

坑三：按需引入的产物体积没降反升。早期多入口构建时，每个组件都重复打包了 utils 和 styles 的公共部分。后来用 Vite 的 `manualChunks` 把公共依赖抽出来，体积才真正降下去。

## 小结

一套源码同时出 UMD 和 npm 包，核心就三件事：

- Vite library mode 多格式输出（`formats: ['es', 'umd']`）
- package.json exports 精准路由（让消费者自动选对产物）
- 设计 token 统一视觉（CSS 变量支持主题覆盖）

组件库不是"写组件"的问题，是"工程化交付"的问题。下一篇讲质量基线，怎么把测试覆盖率拉到 90%。

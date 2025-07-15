---
title: 迁移复盘：从 Webpack 切到 Vite 踩过的坑
date: 2025-07-15 11:00:00
tags:
  - Career
  - Vite
  - Webpack
categories:
  - Career
description: 13 个工具从 Vue 2.7 迁到 React 19 的同时，构建工具也从 Webpack 切到了 Vite。这篇复盘踩过的坑：依赖兼容、HMR、别名、构建产物、CSS Modules 等。
---

前面两篇讲了组件迁移和 Skill 沉淀，其实整个迁移还有一条线没怎么提——构建工具从 Webpack 切到了 Vite。

组件迁移是 AI Agent 跑的，但构建工具的切换没法让 AI 全包，因为每个坑都跟具体依赖、具体环境相关，AI 猜不出来。这部分基本是我一个坑一个坑踩出来的。

这篇把我踩过的坑记一下，别人要迁的时候能少走点弯路就行。

## 为什么要换掉 Webpack

原因其实不复杂：

- dev 启动太慢：13 个工具的 monorepo，Webpack cold start 要 40 秒以上，HMR 响应也越来越慢。
- 配置太重：一堆 loader 和 plugin 互相依赖，改一个地方怕牵一发动全身。
- React 19 生态对新构建工具更友好：很多新库的文档默认按 Vite 写，Webpack 的适配反而成了额外工作。

换构建工具的收益不是"快了一点"，是整个开发体验的质变。Vite 的 dev server 秒开，这个体感差距是回不去的。

但代价也不小。下面是真实踩到的坑。

## 坑一：CommonJS 依赖的预构建

Vite 的 dev server 基于 ESM，但 node_modules 里很多包还是 CommonJS。Vite 会自动用 esbuild 预构建这些依赖，但有些包预构建后会出问题。

典型的：某个依赖在 CommonJS 下导出 `{ default: xxx, ...namedExports }`，esbuild 预构建后 `default` 取不到，运行时报 `xxx is not a function`。

解决办法是把这类包显式加到 `optimizeDeps.include`，让 Vite 强制处理：

```ts
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: [
      // 这些包是 CJS，不显式声明会出运行时错误
      'some-cjs-lib',
      '@some/legacy-package',
    ],
  },
})
```

更坑的是有些包只有 production build 才报错（dev 没问题，build 时 Rollup 处理 CJS 的方式和 esbuild 不一样）。这种只能上线前跑一遍 `vite build` 才能发现。

## 坑二：路径别名

Webpack 的 `resolve.alias` 和 Vite 的 `resolve.alias` 配置方式不一样，名字一样但行为不同，这个最坑。

```ts
// Webpack 写法
resolve: {
  alias: {
    '@': path.resolve(__dirname, 'src'),
  },
}

// Vite 写法（注意格式不同）
resolve: {
  alias: {
    '@': path.resolve(__dirname, 'src'),
    // 尾斜杠很关键，不写会匹配错
    '@/': path.resolve(__dirname, 'src') + '/',
  },
}
```

实际踩到的：有个工具里 `@/components/Button` 和 `@components/Button` 两种写法混用，Webpack 都能解析，Vite 只认第一种。统一别名写法这件事得在迁移前先做，不然迁完到处报 `module not found`。

另外 tsconfig.json 的 `paths` 也得同步改，不然 IDE 的类型提示全飘了。

## 坑三：CSS Modules 的产物差异

我们统一用 CSS Modules，这个在迁移 Skill 里定了。但 Webpack 和 Vite 对 CSS Modules 的处理有差异：

```css
/* Button.module.css */
.title { color: red; }
```

- Webpack：生成的类名是 `Button_title__xxxxx`（带文件名前缀）
- Vite：默认是 `_title_xxxxx`（不带文件名前缀）

看起来无所谓？但你如果有 e2e 测试用类名做选择器就炸了。我们的 Playwright 测试里有一些 `data-testid` 不够的地方用了类名兜底，迁完全挂。

解决办法是显式配置 Vite 的 CSS Modules 生成规则：

```ts
css: {
  modules: {
    generateScopedName: '[name]_[local]__[hash:base64:5]',
  },
},
```

让产物格式和 Webpack 对齐。不过我后来想通了，测试不应该依赖实现细节（类名是实现细节），所以把 e2e 的类名选择器全换成了 `data-testid`，这才是正道。

## 坑四：HMR 行为不一样

Vite 的 HMR 是基于原生 ESM 的，比 Webpack 快很多，但行为有差异：

- Webpack 的 HMR 会保留组件的 state（配合 react-refresh）。
- Vite + react-refresh 大部分场景一样，但修改 store 文件时，Vite 会整页 reload，而 Webpack 可能只刷组件。

这在调试的时候很影响体感。你改一个 Zustand store，页面整个刷新，之前填的表单数据没了。最后我们的做法是把 store 拆得更细——改一个 store 不应该影响无关页面的 state，这本身也是更好的架构实践。

## 坑五：环境变量

Webpack 用 `DefinePlugin` 注环境变量，Vite 用 `import.meta.env`。这个迁移 Agent 能帮忙转一部分，但运行时取值方式变了，需要全局搜一遍：

```ts
// Webpack
const apiBase = process.env.API_BASE

// Vite
const apiBase = import.meta.env.VITE_API_BASE
```

注意 Vite 的环境变量必须以 `VITE_` 开头才会暴露给前端代码。我们有好几个环境变量没改前缀，上线后取到 `undefined`，排查了半天。

```bash
# .env 文件
VITE_API_BASE=https://api.example.com  # 对
API_BASE=https://api.example.com        # 错，前端拿不到
```

## 坑六：production build 产物结构

Vite 默认把所有东西打到 `dist/assets/`，文件名带 hash。WebPack 也是类似，但 chunk 拆分策略不同。

Webpack 的 `splitChunks` 配置在 Vite 里对应 `build.rollupOptions.output.manualChunks`。如果你之前手动配过 chunk 拆分，迁过来要重写。我们的做法是先不手动拆，用 Vite 默认策略跑一轮，看实际产物再调。大多数场景默认策略够用，别过度优化。

## 复盘：怎么平滑迁移

总结一下，如果重来一遍，我会按这个顺序做：

1. 先统一别名和路径规范——不依赖构建工具的事先做了。
2. 把 e2e 里的实现细节依赖去掉（类名选择器换成 data-testid）。
3. Vite 配置先跑 dev，再跑 build，最后对比产物。
4. 环境变量统一改 `VITE_` 前缀，全局搜一遍。
5. 灰度上线，先切一个工具到 Vite，稳定后再铺开。

构建工具迁移的难点不在配置本身，在于"你以为迁好了但某个边角在生产环境才暴露"。dev 能跑不等于没问题，一定要 build + e2e 全过再上线。

这一篇坑比较多，但每个都是实打实遇到的。构建工具的迁移不像组件迁移能让 AI 大包大揽，这里更多是人对工具链的理解和判断。AI 能帮你写配置，但踩坑的经验得自己攒。

---
title: 13 个工具一个仓库：pnpm Monorepo 怎么搭
date: 2024-06-17 10:00:00
tags:
 - Career
 - Monorepo
 - pnpm
 - 前端工程化
categories:
 - Career
description: 13 个金融计算工具塞进一个仓库，pnpm workspace 是怎么把依赖收敛、构建提速的——讲清楚 workspace 结构、依赖管理和多工具并行构建。
---

最近我接手了一个蛮有体量的活儿：给理财工具产品线从零搭一套前端工程化底座。不是一两个工具，是 13 个。理财计算、退休规划、税务优化……形态各不相同，但底层数据、UI 风格、计算引擎都得复用。

13 个工具开 13 个仓库，那不叫工程化，那叫灾难。

第一天我们就拍板：上 monorepo。这篇讲清楚我们怎么用 pnpm workspace 把这事做扎实，顺带把踩过的坑也摊出来。

## 为什么是 pnpm，不是 yarn / lerna

选型的时候我没纠结太久。理由很直接：

- 硬链接省磁盘：13 个工具都装 Vue + Webpack 一整套，yarn/npm 装出来几十 GB；pnpm 的全局 store 让同一个包只占一份磁盘空间。
- 严格的依赖隔离：yarn workspace 的 hoisting 会让子包"意外"用上根目录的依赖。pnpm 默认 strict，没在 `package.json` 里声明的依赖就是 import 不到，避免了一大堆"在我机器上能跑"的玄学问题。
- 速度：CI 上 cold install 比 yarn 快将近一倍。

monorepo 的第一原则：依赖能复用，但绝不能"串味儿"。

Lerna 2023 年起已经交给 nx 团队维护了，新项目我就不赌它的未来。pnpm 自带的 `workspace` 协议加上 `-r` 递归命令，能覆盖我们 90% 的场景。

## workspace 怎么分层

我们先定了一个分层原则，照着搭目录：

```
fintech-suite/
├── apps/                  # 13 个可独立部署的工具
│   ├── wealth-calc/       # 某理财计算工具
│   ├── retirement-plan/   # 某退休规划工具
│   ├── tax-optimize/      # 某税务优化工具
│   └── ...                # 其余 10 个
├── packages/              # 多工具共享的内部包
│   ├── ui-components/     # 组件库（后面单独写一篇）
│   ├── calc-engine/       # 计算引擎的 JS 绑定层
│   ├── api-client/        # 统一 HTTP 客户端 + 拦截器
│   ├── shared-types/      # TS 类型定义
│   └── eslint-config/     # 共享 ESLint 配置
├── tools/                 # 构建/脚本工具
│   └── release-cli/
├── pnpm-workspace.yaml
├── package.json
└── .npmrc
```

核心是分了三层：`apps`（最终产物）、`packages`（内部依赖）、`tools`（开发期辅助）。这层分好之后，依赖方向是单向的——`apps` 依赖 `packages`，`packages` 之间可以互相依赖，但绝不能反向。

`pnpm-workspace.yaml` 就这么几行：

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'tools/*'
```

这一步看着平淡，但它定义了整个仓库的"拓扑骨架"——后面所有递归命令、依赖解析、构建顺序都从这里推出来。

## 依赖管理：catalog 把版本统死

13 个工具的 Vue 版本必须一致，不然构建出来的行为飘忽不定。早期我们靠人工 review 各子包的 `package.json`，特别容易漏掉一两个。pnpm 9+ 引入的 `catalog:` 协议正好治这个病——在根 `pnpm-workspace.yaml` 里集中声明版本：

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

子包里这样引用：

```json
{
  "dependencies": {
    "vue": "catalog:",
    "vue-router": "catalog:"
  }
}
```

`catalog:` 是 monorepo 里管依赖版本的正确姿势——一次声明，处处生效。

升级 Vue 小版本时只改根目录一处，所有子包自动跟着走。这在金融业务里尤其重要：金融工具的依赖版本必须可控可追溯，下游审计要的是"全局 Vue 版本是 X.Y.Z"，不是 13 个包各自飘各自的。

`.npmrc` 也要配几个关键项：

```ini
# 严格隔离，禁止子包通过 hoisting 偷用未声明的依赖
strict-peer-dependencies=true
# 锁文件统一在根目录
shared-workspace-lockfile=true
# CI 环境节省时间
prefer-frozen-lockfile=true
```

## 多工具并行构建：turbo 把流水线压短

光搭好 workspace 还不算完。13 个工具串行 build，CI 上要跑十几分钟，谁也忍不了。我们最后选了 Turbo（Vercel 那个），效果立竿见影：增量构建 + 并行执行，CI 平均从 12 分钟压到 3 分多。

对 13 个工具的仓库来说，这是"十分钟合入主干"和"卡半天"的区别。

根目录 `turbo.json`：

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

`^build` 里那个 `^` 是关键——它表示"先构建完我的上游依赖包，再构建我"。比如 `wealth-calc` 依赖 `@suite/ui-components`，turbo 会自动先 build `ui-components`，再 build `wealth-calc`，而且彼此没有依赖关系的工具之间是并行跑的。

跑全量构建：

```bash
pnpm turbo run build
```

更实用的是按需构建——只跑 git diff 涉及的工具及其下游：

```bash
# 只构建 HEAD^ 之后变更影响到的包
pnpm turbo run build --filter=...[HEAD^]
```

这个 `--filter` 在 PR review 时是神器：一个 PR 只动了退休规划工具，CI 就只构建它和它的依赖，其他 12 个工具完全不碰。

## 踩过的坑

坑一：依赖方向反过来。早期我们把一段共享逻辑写在了某个 app 里，结果另外两个 app 开始依赖它，构建顺序就乱了。turbo 报循环依赖，找了半天才定位。教训：共享的东西永远往上提到 `packages`，`apps` 之间不互相依赖。

坑二：catalog 升级要全量回归。有次升级 vue-router 大版本，只跑了改动的那一个工具的测试，结果另外两个工具的 e2e 挂了。后来定了个铁律——catalog 涉及运行时依赖的升级，强制走全量 e2e，不能省。

坑三：跨 workspace 引用要显式声明协议。pnpm 默认不允许你写个裸版本号去引用 workspace 内的包，必须显式写 `workspace:*` 或 `workspace:^1.0.0`：

```json
{
  "dependencies": {
    "@suite/ui-components": "workspace:*",
    "@suite/calc-engine": "workspace:^1.0.0"
  }
}
```

一开始觉得啰嗦，后来发现这是好事——它逼着你每次都明确"这是内部包还是外部包"，少了很多浑水摸鱼的依赖。

## 小结

monorepo 这事，结构定了，后面所有事都顺。pnpm 的 workspace + catalog + 严格依赖隔离，加上 Turbo 的并行构建，是 13 个工具规模下我觉得最舒服的组合。

下一篇我会写工具线后端——13 个工具共用一套 Spring Boot API 是怎么收敛的，敬请期待。

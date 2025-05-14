---
title: UI 迁移 Agent：用 Claude Code 把 13 个工具一起迁了
date: 2025-05-14 10:00:00
tags:
  - Career
  - Claude Code
  - Agent
categories:
  - Career
description: 把理财线 13 个 Vue 2.7 工具整体迁到 React 19，我没有手迁，而是用 Claude Code 做了一个 UI 迁移 Agent，让 13 个工具并行跑完，一个半月交付。
---

理财线有 13 个工具，全是 Vue 2.7 的，要整体迁到 React 19。

按传统手迁的路子，一个工具按 3 天算，串下来快两个月；3 个人手迁，算上踩坑和返工，三个月起步。

我最后花了大概一个半月，而且不是一个人闷头干，是 3 个人同时铺开 13 个工具。

怎么做到的？核心就一件事：我没有去迁代码，我做了一个迁代码的 Agent。

## 先说结论

不要让 AI 帮你迁代码，让 AI 去当一个"迁移工程师"。

这两件事听起来像，其实差很远。前者是你坐在那里，一个文件一个文件喂给 Claude，让它给你翻成 React；后者是你设计一套工作流，让 Agent 自己扫描、自己决策、自己产出、自己验证，你只负责处理它搞不定的边角。

## 为什么是 13 个一起迁

理财线的工具长这样：某理财计算器、某贷款计算器、某再融资工具、某购房能力评估、某债务合并工具……十几个，业务上互相独立，但技术栈完全一样：Vue 2.7 + Pinia + Element-Plus + Webpack。

这其实是个非常适合并行的场景。如果迁移规则能沉淀成一份"规范"，那 13 个工具就是 13 次同一套规则的应用，不是 13 个全新的问题。

所以我把它们当成 1 个迁移规则 + 13 次执行，而不是 13 个独立的迁移任务。这个视角的转换很重要。

## Agent 怎么设计的

我用 Claude Code 做了一个 UI 迁移 Agent，工作流分四步：

```
┌─────────────┐   ┌─────────────┐   ┌──────────────┐   ┌─────────────┐
│  1. SCAN    │──▶│  2. MAP     │──▶│ 3. GENERATE  │──▶│  4. VERIFY  │
│ 扫 Vue SFC  │   │ 规则映射    │   │ 产出 React   │   │ lint+类型   │
└─────────────┘   └─────────────┘   └──────────────┘   └─────────────┘
                                                                 │
                                                        不通过 ▼
                                                       回 GENERATE 重跑
```

**第一步 SCAN**：让 Agent 读 Vue 单文件组件，把 `<template>`、`<script setup>`、`<style>` 三段拆出来，同时把 props、emits、computed、refs 这些元信息结构化，输出一个 JSON 描述。这步是为了后面映射有"上下文"，不是裸字符串替换。

**第二步 MAP**：整个 Agent 的核心。我写了一份映射规则（后面抽成了 Skill，下篇细讲），规则长这样：

```yaml
# vue-to-react-mapping（节选，脱敏）
template:
  v-if:     "→ 条件渲染 {cond && <Comp />}"
  v-for:    "→ Array.map，key 优先用业务 id"
  v-model:  "→ useState + onChange 双向绑定"
  @click:   "→ onClick"
  :class:   "→ clsx() 拼接"
  slot:     "→ children / render prop"

script:
  ref / reactive:  "→ useState"
  computed:        "→ useMemo"
  watch:           "→ useEffect（带依赖）"
  onMounted:       "→ useEffect(() => {}, [])"
  defineProps:     "→ interface Props + 解构默认值"
  defineEmits:     "→ 回调 props（onXxx）"

style:
  scoped css:  "→ CSS Modules（*.module.css）"
```

**第三步 GENERATE**：Agent 按映射规则产出 React 19 的 `.tsx` + `.module.css`，状态管理统一走 Zustand（为什么是 Zustand 下篇讲，简单说就是迁移成本最低、心智模型最接近 Composition API）。

**第四步 VERIFY**：自动跑 ESLint + `tsc` 类型检查 + 关键路径的 e2e 冒烟。挂了就回退到 GENERATE 让 Agent 自己修，尽量不让人介入。这一步把"人盯着 AI"变成了"AI 盯着 AI"。

## 关键设计：规则和执行分离

这里有个我认为很关键的判断：

迁移规则是人定的，迁移执行是 AI 干的，千万别混在一起。

如果你让 Agent "看着办"，把一个 Vue 文件丢给他说"转成 React"，他会给你一个能跑但风格全不一样的结果。13 个工具 13 种写法，后面维护就是灾难。

我的做法是先把规则钉死，再让 Agent 在规则的框架里干活。规则覆盖了：

- 命名规范（组件 PascalCase、hooks 用 `use` 前缀、工具函数 camelCase）
- 文件组织（一个组件一个目录，`index.tsx` + `*.module.css` + `types.ts`）
- 状态边界（跨组件走 Zustand store，组件内状态走 useState，不混用）
- 样式方案（统一 CSS Modules，不用 styled-components）
- 测试要求（每个工具至少覆盖主路径的 e2e，迁移后跑通才算完）

Agent 只能在这个框架里发挥，不能自己发明规范。

## 怎么并行 13 个工具

Claude Code 支持 Subagent（子代理）模式，可以把一个大任务拆成多个独立的子任务并行跑。我的编排大概是：

```bash
# 简化示意，实际走 Claude Code 的 Task 编排
tools=(wealth-calc-1 wealth-calc-2 wealth-calc-3 wealth-calc-4 \
       wealth-calc-5 wealth-calc-6 wealth-calc-7 wealth-calc-8 ...)

for tool in "${tools[@]}"; do
  # 每个工具一个独立 Agent，带同一份迁移规则
  claude run migration-agent --target "$tool" --rules ./skills/ &
done
wait
echo "全部跑完，开始人工兜底"
```

每个工具起一个独立的 Agent 实例，跑完整的 SCAN → MAP → GENERATE → VERIFY 流程。13 个工具同时转，机器在跑，人去看它搞不定的报错。

实际跑下来，大部分工具第一轮就能跑到 VERIFY 通过。少数卡在复杂计算属性或者深度依赖 Element-Plus 组件的地方（比如某些带复杂 slot 的表格组件），需要人工介入。但介入量比纯手迁少太多了。

## 一个半月到底花在哪了

坦白讲，一个半月里 Agent 真正跑的时间可能就两周。剩下的时间花在：

- 写规则（最耗时）：前两周几乎全在写和调映射规则，跑了三四个试点工具反复打磨，直到规则稳定。
- 人工兜底：有些 Vue 里的奇技淫巧（`$attrs` 透传、动态组件、render 函数、`v-html`）Agent 转不好，要手改。
- e2e 补测：迁完不是结束，要保证业务逻辑没回归，理财工具的数据口径不能错，验证花了不少时间。
- 联调和灰度：13 个工具一起上线，联调成本不低，灰度也要一步步来。

## 最值的一个判断

回头看，最值的判断不是用了 AI，而是"先建规则再开工"。

如果一上来就开干，一个文件一个文件地迁，AI 会越用越乱：每个文件的迁移结果都不一样，后面接手的人完全看不懂。先花两周把规则和 Skill 沉淀好，后面 13 个工具几乎是"复制粘贴"的体验。

AI 迁移的瓶颈从来不是 AI 的能力，而是你有没有把规则讲清楚。

规则讲清楚了，13 个工具并行就是水到渠成的事。规则没讲清楚，迁 1 个工具都是赌。

下一篇我会展开讲那 10 多个迁移 Skill 是怎么写的，以及为什么我觉得 Skill 是目前 AI 编码里最被低估的东西。

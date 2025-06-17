---
title: 迁移 Skill：沉淀出 10+ 个能复用的东西
date: 2025-06-17 14:00:00
tags:
  - Career
  - Claude Code
  - Skill
categories:
  - Career
description: 13 个工具迁完后，我把迁移过程中反复用到的规则抽成了 10+ 个 Claude Code Skill。这篇讲 Skill 到底是什么、怎么写、为什么我觉得它是最被低估的 AI 编码工具。
---

上篇说到，我用一个 UI 迁移 Agent 把 13 个工具从 Vue 2.7 迁到了 React 19，花了一个半月。

很多人关心的其实是那句话后面的一句，"后面沉淀出了 10+ 个能复用的 Skill"。

这篇就展开讲：Skill 到底是什么，怎么写，以及为什么我觉得它是目前 AI 编码里最被低估的东西。

## 先说 Skill 是什么

Claude Code 里的 Skill，简单说就是一段可复用的指令模板。你把一件事情的做法、约束、检查标准写成结构化的 markdown，Claude Code 在合适的场景自动加载它，按你的规矩来干活。

你可以理解为：Prompt 是一次性的对话，Skill 是沉淀下来的工程规范。

打个比方：你第一次教一个新人"我们这边 PR 怎么提"，你会在 Slack 上打一大段话；第二次又来一个新人，你又打一遍；打到第三次你就烦了，写进 wiki 对吧？Skill 就是那个 wiki，只不过 AI 会自动在需要的时候去读它，不需要你提醒。

## 一个 Skill 长什么样

拿迁移里最常用的 `vue-to-react-mapping` 来说（脱敏后）：

```markdown
# vue-to-react-mapping

> 把 Vue 2.7 SFC 的常见写法映射到 React 19，保证 13 个工具迁移风格一致。

## 适用场景
当需要把 Vue 单文件组件迁移为 React 函数组件时使用。

## 模板映射规则
| Vue 写法 | React 产出 | 备注 |
|---|---|---|
| `v-if` | `{cond && <Comp />}` | 不用三元，短路更清晰 |
| `v-for` | `arr.map(item => ...)` | key 优先用业务 id |
| `v-model` | `useState + onChange` | 受控组件 |
| `:class` | `clsx(...)` | 不拼字符串 |
| `slot` | `children` / render prop | 具名 slot 用对象传 |

## Script 映射规则
- `ref(x)` / `reactive({})` → `useState`
- `computed(() => ...)` → `useMemo(() => ..., [deps])`
- `watch(src, cb)` → `useEffect(() => cb(), [deps])`
- `onMounted` → `useEffect(..., [])`
- `defineProps` → `interface Props` + 解构 + 默认值

## 禁止项
- 禁止产出 Class 组件，一律函数组件
- 禁止用 `dangerouslySetInnerHTML` 替代 `v-html`，先标记 TODO
- 禁止自创状态管理方案，跨组件状态走 Zustand store

## 验收标准
- `tsc --noEmit` 零报错
- ESLint 零 error
- 主路径 e2e 通过
```

你看，它不是一段模糊的 Prompt，而是有适用场景、有规则表、有禁止项、有验收标准的工程文档。AI 拿到这个东西，迁出来的代码才会一致。

## 我沉淀了哪些 Skill

迁移跑完，我数了一下，实际产出的大概是这些（分几类）：

映射类（迁移核心）
- `vue-to-react-mapping` — Vue 语法到 React 的映射规则
- `element-to-component-lib` — Element-Plus 组件到目标 UI 库的对应关系
- `pinia-to-zustand` — 状态管理迁移规则

规范类（保证一致性）
- `react-naming-convention` — 命名规范
- `file-structure` — 一个组件一个目录的组织方式
- `css-modules-style` — 样式方案规范

质量类（兜底）
- `migration-verify` — 迁移后的自检清单（类型、lint、测试）
- `test-coverage-check` — 测试覆盖检查
- `a11y-checklist` — 可访问性检查（理财工具对 a11y 有要求）

流程类（团队协作）
- `migration-handoff` — 迁移完一个工具后的交接检查
- `code-review-checklist` — 迁移相关 PR 的 review 要点

加起来 10+ 个，每个都不长，但每个都在解决一个"如果不写下来就会重复出问题"的点。

## 为什么是 Zustand 不是 Redux

顺便回答上篇留的坑。状态管理我最后选了 Zustand，原因很实在：

- 迁移成本最低：Pinia 的 `defineStore` 到 Zustand 的 `create`，心智模型几乎一对一。Agent 迁起来很顺，几乎不需要人介入。
- 没有 boilerplate：Redux Toolkit 已经够精简了，但 Zustand 更轻。对于一个计算器工具来说，我不想为了状态管理写一堆 slice、reducer。
- TypeScript 友好：类型推导自然，不用额外写类型模板。

```ts
// Pinia（迁移前）
export const useCalcStore = defineStore('calc', () => {
  const amount = ref(0)
  const setAmount = (v: number) => { amount.value = v }
  return { amount, setAmount }
})

// Zustand（迁移后）
interface CalcState {
  amount: number
  setAmount: (v: number) => void
}
export const useCalcStore = create<CalcState>((set) => ({
  amount: 0,
  setAmount: (v) => set({ amount: v }),
}))
```

你看这个映射多直接。Agent 第一轮就能转对，不用反复调。

选技术栈的判断标准不是"哪个更好"，而是哪个在当前场景下 AI 迁起来最不容易出错。

## Skill 为什么被低估

我觉得大家用 AI 编码，目前卡在一个误区里：太关注 Prompt，不关注 Skill。

Prompt 是你和 AI 的一次对话，聊完就没了。Skill 是你把这次对话里值得沉淀的部分固化下来，下次自动复用。

打个比方：Prompt 是你给新人讲了一遍怎么做，Skill 是你把这个做法写进了团队规范。前者每次都要重来，后者只需要讲一次。

迁移这件事让我确认了一个判断：AI 编码的上限由 Prompt 决定，但 AI 编码的下限和一致性由 Skill 决定。13 个工具能并行跑出一致的结果，靠的不是 Prompt 写得多好，是 Skill 把规范钉死了。

如果你团队在用 Claude Code，我强烈建议把那些重复出现的规范、约束、检查标准都抽成 Skill。前期花点时间写，后面每次都用上，这笔账非常划算。

下一篇讲 Webpack 切到 Vite 的那些坑，构建工具的迁移比组件迁移坑多了。

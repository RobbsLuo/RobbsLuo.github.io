---
title: Claude Code 进团队：CLAUDE.md 和 Agent 怎么编排
date: 2025-09-15 15:00:00
tags:
  - Career
  - Claude Code
  - CLAUDE.md
categories:
  - Career
description: 迁移跑通后，我把 Claude Code 的使用从"个人工具"推到了"团队规范"。这篇讲 CLAUDE.md 怎么写团队级的 AI 编码规范，Subagent 怎么编排多人并行。
---

理财线迁移跑通之后，我做了一件事：把 Claude Code 从我自己用的工具，推成了整个团队的工作方式。

迁移那一个半月，主要是我和 Claude Code 两个人（应该说一个人加一堆 Agent）在跑。但跑完之后，其他人也得用起来，不然这些 Skill 和规范就是我一个人的东西，换个人就废了。

这篇讲两件事：CLAUDE.md 怎么写团队规范，Subagent 怎么编排多人并行。

## 先说为什么不能只靠口头规范

迁移完了之后，团队里有人开始自己用 Claude Code。但很快出现一个问题：每个人用出来的风格不一样。

A 同学让 Claude 写组件，文件组织是一个目录一个组件；B 同学让 Claude 写，全堆在一个文件里。C 同学的测试覆盖很全，D 同学压根没让 Claude 写测试。

你去说"我们要统一规范"，说了等于没说。口头规范在 AI 编码时代几乎等于零约束力，因为每个人跟 AI 的对话都不一样，AI 只按当前对话的上下文来。

人类团队的规范靠 wiki 和口头传达还能凑合，AI 团队的规范得写进机器读得到的地方。

这就是 CLAUDE.md 存在的意义。

## CLAUDE.md 是什么

CLAUDE.md 是 Claude Code 在项目根目录自动读取的一个配置文件。你把团队规范写进去，Claude Code 每次启动都会读它，然后按规矩办事。

你可以把它理解成给 AI 看的 CONTRIBUTING.md。人类的新人读 wiki，AI 的新人（也就是每次 Claude Code 启动）读 CLAUDE.md。

## 我们的 CLAUDE.md 长什么样

脱敏后的核心片段：

```markdown
# 项目工程规范（CLAUDE.md）

## 技术栈
- React 19 + TypeScript 5.x
- Vite 6 构建
- Zustand 状态管理
- CSS Modules 样式
- pnpm monorepo

## 代码规范
- 组件用函数组件，禁止 Class 组件
- 文件命名：组件 PascalCase，工具函数 camelCase
- 一个组件一个目录：`Button/index.tsx` + `Button.module.css` + `types.ts`
- 跨组件状态走 Zustand store，组件内状态走 useState
- 禁止 `any`，必须显式标类型

## 测试要求
- 每个新组件至少有主路径的单元测试
- 业务工具必须有 e2e 覆盖核心计算流程
- 测试文件放 `__tests__` 目录，命名 `xxx.test.ts`

## PR 规范
- commit message 用 conventional commits 格式
- PR 描述必须包含：改了什么、为什么改、怎么测的
- 禁止直接 push master，必须走 PR

## 禁止项
- 禁止引入新依赖而不在 PR 里说明理由
- 禁止用 `// @ts-ignore` 跳过类型检查
- 禁止提交 console.log
```

你看，它不是什么高深的东西，就是把团队约定的事实写成了 AI 能读的格式。但效果差别巨大：写进去之后，每个人用 Claude Code 产出的代码风格开始趋同了。

## 怎么组织 CLAUDE.md

我们的 CLAUDE.md 不是一个大文件，而是分层的：

```
项目根/
├── CLAUDE.md              ← 全局规范（技术栈、代码风格、PR 流程）
├── packages/
│   ├── shared/
│   │   └── CLAUDE.md      ← 共享库的特定规范
│   └── tools/
│       ├── mortgage-calculator/
│       │   └── CLAUDE.md  ← 这个工具的特定规范（业务逻辑约束）
│       └── ...
└── .claude/
    └── skills/            ← 可复用 Skill
        ├── vue-to-react-mapping.md
        ├── code-review-checklist.md
        └── ...
```

Claude Code 会按层级合并这些文件：根目录的全局规范 + 当前工作目录的特定规范。这样你在一个工具里干活时，Claude 既知道全局规矩，也知道这个工具的特殊约束。

CLAUDE.md 的分层设计很重要。全局规范管一致性，局部规范管特殊性，不要全堆一个文件里。

## Subagent 怎么编排

CLAUDE.md 解决的是"规范统一"的问题，Subagent 解决的是"并行效率"的问题。

迁移的时候我用了 Subagent 来并行跑 13 个工具。迁移完了之后，日常开发也开始用 Subagent 模式。

举一个真实的场景：同时有三个工具要加新功能。传统做法是三个人各干各的，互相不知道对方在干嘛。用 Subagent 的做法是：

```
主 Agent（我）
├── Subagent A：给某贷款计算器加提前还款功能
├── Subagent B：给某再融资工具加新利率场景
└── Subagent C：给某购房能力评估加税费计算
```

每个 Subagent 带着同一份 CLAUDE.md 和相关 Skill，独立跑。跑完汇总到主 Agent 做集成检查。

关键是主 Agent 的职责不是写代码，而是编排和兜底：

- 分配任务（谁干什么）
- 检查冲突（改了同一个共享文件？）
- 集成验证（三个功能加完，整体能跑吗？）

人类写代码的时代，管理者分配任务；AI 写代码的时代，人类设计编排方式。角色的重心从"执行"转向了"编排"。

## 实际效果

推了大概两个月，几个明显的变化：

- PR 风格统一了：不再是每个人一个风格，code review 的"格式问题"少了 80% 以上。
- 新人上手快了：新同学进来，CLAUDE.md 一读，Skill 一看，用 Claude Code 产出的代码天然符合规范。
- 测试覆盖上去了：CLAUDE.md 里写了测试要求，Claude Code 每次都会主动补测试，不用人提醒。

但也有代价：CLAUDE.md 的维护成本不低。技术栈变了、规范调整了，都得同步改。我把 CLAUDE.md 的更新列进了团队的常规迭代项，不然它很快就会过时。

## 我觉得最重要的一点

推了一轮下来，我觉得最重要的一点是：

AI 编码进团队，核心不是教大家怎么写 Prompt，而是把规范"机器可读化"。

Prompt 是个人技巧，你教了也管不住每个人怎么用。CLAUDE.md 和 Skill 是工程基建，写好了所有人自动受益。

这跟以前做 DevOps 是一个道理：你不会去教每个新人怎么配 CI/CD，你把 CI/CD 配好，让流程自动跑。AI 编码的规范化也是一样，建基建比教技巧更有用。

下一篇是理财线 AI 编码的总复盘，讲这两年从"个人用 AI"到"团队工程化"的完整脉络。

# OpenCode 内置 Agent 笔记：`plan`

一句话定位：规划模式；默认禁止改代码（只允许写/改 plan 文件），并通过注入 system reminder 强约束“只读”。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:90`

## Prompt / Reminder

- `plan` 自身没有内置专属 prompt（`agent.ts` 未设置 `prompt` 字段）。
- plan mode 的 system reminder 文本：`opencode/packages/opencode/src/session/prompt/plan.txt`
- reminder 注入逻辑：`opencode/packages/opencode/src/session/prompt.ts:1231`

## 权限/工具（默认）

- 基础默认权限：`opencode/packages/opencode/src/agent/agent.ts:53`
- `plan` 在默认基础上额外开启：
  - `question: allow`
  - `plan_exit: allow`
  - `external_directory` 允许访问 data plans 目录
  - `edit` 默认 deny，但允许写/改 plan 文件：
    - `.opencode/plans/*.md`
    - `path.relative(Instance.worktree, path.join(Global.Path.data, "plans", "*.md"))`（global data plans 的相对路径）
  - 来源：`opencode/packages/opencode/src/agent/agent.ts:94`

## 相关机制

- 进入/退出 plan 的交互入口：`plan_enter` / `plan_exit`（`opencode/packages/opencode/src/tool/plan.ts`）
- 注意：plan mode 的“只读”并不完全依赖权限；session 会把 plan reminder 作为 synthetic user part 注入，提示 LLM 必须只读（`session/prompt.ts`）。

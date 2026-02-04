# OpenCode 内置 Agent 报告 (plan)

一句话定位：plan mode 的 primary agent；通过权限与 reminder 注入实现“默认只读”。

定义位置：`opencode/packages/opencode/src/agent/agent.ts:90`

## 1. 身份与系统提示词 (Identity & Prompt)

`plan` **没有**独立的 `agent.prompt` 文本；它与 `build/general` 一样走 provider system prompt。

### system prompt 的真实来源（与 build 相同）

- 拼接逻辑：`opencode/packages/opencode/src/session/llm.ts:67`
- provider 选择：`opencode/packages/opencode/src/session/system.ts:13`
- 文本文件：
  - `opencode/packages/opencode/src/session/prompt/codex_header.txt`
  - `opencode/packages/opencode/src/session/prompt/beast.txt`
  - `opencode/packages/opencode/src/session/prompt/gemini.txt`
  - `opencode/packages/opencode/src/session/prompt/anthropic.txt`
  - `opencode/packages/opencode/src/session/prompt/qwen.txt`

补充：对于 Codex 会话（OpenAI OAuth），provider prompt 不一定进 system content，而是通过 `options.instructions` 发送。

- `options.instructions = SystemPrompt.instructions()`
- 来源：`opencode/packages/opencode/src/session/llm.ts:114`, `opencode/packages/opencode/src/session/system.ts:13`

补充：`input.system` 通常由 session 层提供（环境信息 + 指令文件）：

- `system: [...(await SystemPrompt.environment(model)), ...(await InstructionPrompt.system())]`
- 来源：`opencode/packages/opencode/src/session/prompt.ts:619`

### plan mode reminder（synthetic user part；不是 system prompt）

plan 的“只读”除了权限外，还依赖 reminder 注入。

#### 旧逻辑（flag 关闭）

- 注入点：`opencode/packages/opencode/src/session/prompt.ts:1235`
- 注入文本（原文）：`opencode/packages/opencode/src/session/prompt/plan.txt`

#### 新逻辑（flag 开启）

- 注入点：`opencode/packages/opencode/src/session/prompt.ts:1261`
- 注入文本：内联模板（包含 plan file 路径与分阶段工作流），不在单独的 .txt 文件里。

## 2. 工具系统（权限差异）

来源：`opencode/packages/opencode/src/agent/agent.ts:94`

- `question: allow`
- `plan_exit: allow`
- `external_directory`：允许访问 global data 下的 `plans/*`（absolute path pattern）
- `edit`：默认 deny，仅允许写/改 plan 文件：
  - `.opencode/plans/*.md`
  - global data plans 的相对路径 pattern（`path.relative(Instance.worktree, ...)`）

说明：最终权限仍会 merge 用户配置（`cfg.permission` + `cfg.agent.plan.permission`）。

## 3. 相关机制

- plan/build 切换提示注入：`opencode/packages/opencode/src/session/prompt.ts:1231`
- `plan_enter` / `plan_exit` 是工具层面的 mode 切换入口（文本指引在 `opencode/packages/opencode/src/tool/plan-enter.txt` / `opencode/packages/opencode/src/tool/plan-exit.txt`）。

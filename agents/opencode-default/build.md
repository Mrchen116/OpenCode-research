# OpenCode 内置 Agent 报告 (build)

一句话定位：默认 primary agent；当没有显式指定 agent（或 `default_agent` 配置无效）时通常会落到 `build`。

定义位置：`opencode/packages/opencode/src/agent/agent.ts:75`

## 1. 身份与系统提示词 (Identity & Prompt)

`build` **没有**独立的 `agent.prompt` 文本；因此它的“系统提示词”来自 OpenCode 的 provider system prompt 选择逻辑。

### system prompt 拼接规则（权威）

来源：`opencode/packages/opencode/src/session/llm.ts:67`

1) 如果 `input.agent.prompt` 存在：用它。
2) 否则：用 `SystemPrompt.provider(input.model)`。
3) 然后再拼上 `input.system` 与 `input.user.system`。

对 `build` 而言：因为 `agent.prompt` 不存在，所以会走第 2 条。

补充：对于 Codex 会话（OpenAI OAuth），provider prompt 不一定进 system content，而是通过 `options.instructions` 发送。

- `options.instructions = SystemPrompt.instructions()`
- 来源：`opencode/packages/opencode/src/session/llm.ts:114`, `opencode/packages/opencode/src/session/system.ts:13`

补充：`input.system` 通常由 session 层提供（环境信息 + 指令文件），不是写死在 agent 内。

- `system: [...(await SystemPrompt.environment(model)), ...(await InstructionPrompt.system())]`
- 来源：`opencode/packages/opencode/src/session/prompt.ts:619`

### provider system prompt 的真实来源

来源：`opencode/packages/opencode/src/session/system.ts:13`

- 选择逻辑在 `SystemPrompt.provider(model)`。
- 真实文本在这些文件（按模型选择其一）：
  - `opencode/packages/opencode/src/session/prompt/codex_header.txt`
  - `opencode/packages/opencode/src/session/prompt/beast.txt`
  - `opencode/packages/opencode/src/session/prompt/gemini.txt`
  - `opencode/packages/opencode/src/session/prompt/anthropic.txt`
  - `opencode/packages/opencode/src/session/prompt/qwen.txt`

### plan → build 切换提示（synthetic user part）

从 plan 切回 build 时，会在最后一条 user message 里追加提醒文本（不是 system prompt）。

- 注入逻辑：`opencode/packages/opencode/src/session/prompt.ts:1247`
- 文本来源：`opencode/packages/opencode/src/session/prompt/build-switch.txt`

## 2. 工具系统（权限基线 + build 差异）

### 默认权限基线（所有 agent merge 的起点）

来源：`opencode/packages/opencode/src/agent/agent.ts:53`

- `* : allow`
- `question: deny`（默认禁止提问，避免无意义追问）
- `plan_enter/plan_exit: deny`
- `read` 对 `.env*`：默认 `ask`（降低意外泄露风险）
- `external_directory`：默认 `ask`，但 truncation 输出目录默认 allow

### build 的权限差异

来源：`opencode/packages/opencode/src/agent/agent.ts:79`

- `question: allow`
- `plan_enter: allow`

说明：最终权限还会 merge 用户配置（`cfg.permission` + `cfg.agent.build.permission`）。

## 3. 相关机制

- `default_agent` 配置项与校验：`opencode/packages/opencode/src/agent/agent.ts:268`
- 默认 agent 选择：
  - 指定了 `cfg.default_agent`：必须存在且不能是 subagent/hidden
  - 未指定：选择第一个“primary 且可见”的 agent
  - 来源：`opencode/packages/opencode/src/agent/agent.ts:264`

# OpenCode 内置 Agents（默认）

一句话定位：OpenCode core 自带的 agent 定义与权限/提示词来源索引；用于回答“默认有哪些 agent、系统提示词到底来自哪、plan/build 切换怎么生效”。

## 1. 定义位置（权威来源）

- Agent 列表与权限规则：`opencode/packages/opencode/src/agent/agent.ts`
- System prompt 选择与环境信息：`opencode/packages/opencode/src/session/system.ts`
- system 片段来源（环境 + 指令文件）：`opencode/packages/opencode/src/session/prompt.ts:619`
- System prompt 拼接（provider/agent prompt + system 片段合并）：`opencode/packages/opencode/src/session/llm.ts:67`
- Plan/build reminder 注入（synthetic user part）：`opencode/packages/opencode/src/session/prompt.ts:1231`

## 2. 内置 agent 列表（与源码一致）

来源：`opencode/packages/opencode/src/agent/agent.ts:74`

| name | mode | hidden | prompt（agent.prompt） |
| --- | --- | --- | --- |
| `build` | primary | no | 无（走 provider system prompt） |
| `plan` | primary | no | 无（走 provider system prompt + plan reminder 注入） |
| `general` | subagent | no | 无（走 provider system prompt） |
| `explore` | subagent | no | `opencode/packages/opencode/src/agent/prompt/explore.txt` |
| `compaction` | primary | yes | `opencode/packages/opencode/src/agent/prompt/compaction.txt` |
| `title` | primary | yes | `opencode/packages/opencode/src/agent/prompt/title.txt` |
| `summary` | primary | yes | `opencode/packages/opencode/src/agent/prompt/summary.txt` |

## 3. “真实系统提示词”到底从哪来？

OpenCode 发给模型的 system 内容不是“每个 agent 一份固定文本”，而是按以下规则动态拼接：

### 3.1 system 拼接顺序

来源：`opencode/packages/opencode/src/session/llm.ts:67`

1) **agent prompt 优先**：如果 `input.agent.prompt` 存在，用它；否则用 `SystemPrompt.provider(input.model)`。
2) `...input.system`：调用栈/插件追加的 system 片段。
3) `...(input.user.system)`：最后一条 user message 自带的 system 片段。

结论：只有 `explore/compaction/title/summary` 这种显式 `agent.prompt` 的 agent 才有“固定的 agent prompt 文本”。
`build/plan/general` 没有 agent.prompt，它们的“系统提示词”来自 **provider prompt**。

补充：对于 Codex 会话（OpenAI OAuth），provider prompt 不一定进 system content，而是通过 `options.instructions` 发送。

- `options.instructions = SystemPrompt.instructions()`
- 来源：`opencode/packages/opencode/src/session/llm.ts:114`, `opencode/packages/opencode/src/session/system.ts:13`

补充：`input.system` 不是凭空来的，通常由 session 层传入（环境信息 + 指令文件拼接）：

- `system: [...(await SystemPrompt.environment(model)), ...(await InstructionPrompt.system())]`
- 来源：`opencode/packages/opencode/src/session/prompt.ts:619`

### 3.2 provider system prompt 的选择逻辑

来源：`opencode/packages/opencode/src/session/system.ts:13`

- `SystemPrompt.provider(model)` 会按模型选择下列文件之一（或之一组）：
  - `opencode/packages/opencode/src/session/prompt/codex_header.txt`（gpt-5 系列；也用于 Codex instructions）
  - `opencode/packages/opencode/src/session/prompt/beast.txt`（多数 gpt-/o1/o3）
  - `opencode/packages/opencode/src/session/prompt/gemini.txt`
  - `opencode/packages/opencode/src/session/prompt/anthropic.txt`
  - `opencode/packages/opencode/src/session/prompt/qwen.txt`（fallback，命名虽是 qwen 但作为默认）

### 3.3 “plan mode 提醒”不是 system prompt

plan/build 切换相关的提醒是通过 **在最后一条 user message 里追加 synthetic text part** 实现的。

- 旧逻辑（flag 关闭）：`opencode/packages/opencode/src/session/prompt.ts:1235`
  - plan：追加 `opencode/packages/opencode/src/session/prompt/plan.txt`
  - 从 plan 切回 build：追加 `opencode/packages/opencode/src/session/prompt/build-switch.txt`
- 新逻辑（flag 开启）：`opencode/packages/opencode/src/session/prompt.ts:1261`
  - 进入 plan：注入一个内联 `<system-reminder>`（包含 plan file 路径与工作流）
  - 从 plan 切回 build：注入 `build-switch.txt` + plan file 存在提示

## 4. 权限模型（简要）

- 默认权限基线（所有 agent 都会 merge）：`opencode/packages/opencode/src/agent/agent.ts:53`
  - `* : allow`
  - `question/plan_enter/plan_exit` 默认 deny（只对特定 agent 放开）
  - `read` 对 `.env*` 默认 ask（避免泄露）
  - `external_directory` 默认 ask，但 truncation 目录默认 allow
- 最终权限 = defaults + agent-specific overrides + user cfg.permission + user cfg.agent[agent].permission。

## 5. 详细笔记（逐 agent）

- `opencode-research/agents/opencode-default/build.md`
- `opencode-research/agents/opencode-default/plan.md`
- `opencode-research/agents/opencode-default/general.md`
- `opencode-research/agents/opencode-default/explore.md`
- `opencode-research/agents/opencode-default/compaction.md`
- `opencode-research/agents/opencode-default/title.md`
- `opencode-research/agents/opencode-default/summary.md`

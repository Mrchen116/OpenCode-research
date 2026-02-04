# OpenCode 内置 Agent 报告 (general)

一句话定位：通用 subagent；用于分发子任务（并行/多单元工作）但默认禁用 todo 工具，避免污染主会话的 todo 状态。

定义位置：`opencode/packages/opencode/src/agent/agent.ts:113`

## 1. 身份与系统提示词 (Identity & Prompt)

`general` **没有**独立的 `agent.prompt` 文本；它走 provider system prompt。

- 拼接逻辑：`opencode/packages/opencode/src/session/llm.ts:67`
- provider 选择：`opencode/packages/opencode/src/session/system.ts:13`

补充：对于 Codex 会话（OpenAI OAuth），provider prompt 不一定进 system content，而是通过 `options.instructions` 发送。

- `options.instructions = SystemPrompt.instructions()`
- 来源：`opencode/packages/opencode/src/session/llm.ts:114`, `opencode/packages/opencode/src/session/system.ts:13`

补充：`input.system` 通常由 session 层提供（环境信息 + 指令文件）：

- `system: [...(await SystemPrompt.environment(model)), ...(await InstructionPrompt.system())]`
- 来源：`opencode/packages/opencode/src/session/prompt.ts:619`

## 2. 工具系统（权限差异）

来源：`opencode/packages/opencode/src/agent/agent.ts:116`

- `todoread: deny`
- `todowrite: deny`

其余权限来自 defaults + 用户配置 merge（见 `opencode/packages/opencode/src/agent/agent.ts:53`）。

## 3. 相关机制

- `general` 是 subagent（`mode: subagent`），因此不能被设置为 `default_agent`：`opencode/packages/opencode/src/agent/agent.ts:271`

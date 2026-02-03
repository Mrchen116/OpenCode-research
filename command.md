# Command 笔记

## command 的 agent/model/subtask 行为
- agent 选择顺序：`command.agent` > `input.agent` > 默认 agent。（`opencode/packages/opencode/src/session/prompt.ts`）
- model 选择顺序：`command.model` > `command.agent.model` > `input.model` > 上一次使用的模型。（`opencode/packages/opencode/src/session/prompt.ts`）
- subtask 判定：`isSubtask = (agent.mode === "subagent" && command.subtask !== false) || command.subtask === true`。（`opencode/packages/opencode/src/session/prompt.ts`）

## 当用户在 A agent，对应 command 配置 agent = B 时会发生什么
- 若 B 是普通 agent（非 subagent）：这条 command 响应由 B 生成。
  - system prompt 用 B 的 prompt（`LLM.stream` 使用 `input.agent.prompt`）。（`opencode/packages/opencode/src/session/llm.ts`）
  - 可用工具会按 B 的权限过滤（`resolveTools`）。（`opencode/packages/opencode/src/session/llm.ts`）
  - 仍在同一 session 历史中，仅这条 assistant 消息的 agent 标记为 B。（`opencode/packages/opencode/src/session/prompt.ts`）
  - UI 的当前 agent 不会因为这条 command 自动切换；只会在“切换 session”时同步。（`opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`）
- 若 B 是 subagent（或 command 显式 `subtask: true`）：会创建 subtask，主对话仍由 A 继续。（`opencode/packages/opencode/src/session/prompt.ts`）

## 实用建议
- 想让 command 用不同 agent 执行、但不影响主对话：用 subagent（或 `subtask: true`）。
- 想让 command 的回复直接以另一个主 agent 身份输出：`command.agent` 设为该 agent，`subtask` 保持未设或 false。

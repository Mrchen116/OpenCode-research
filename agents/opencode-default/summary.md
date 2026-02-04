# OpenCode 内置 Agent 笔记：`summary`

一句话定位：内部总结 agent（hidden），用于生成对话摘要。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:186`

## Prompt

- `opencode/packages/opencode/src/agent/prompt/summary.txt`
- 绑定位置：`opencode/packages/opencode/src/agent/agent.ts:199`

## 权限/工具（默认）

- `* : deny`（不允许任何工具调用）
- 来源：`opencode/packages/opencode/src/agent/agent.ts:192`

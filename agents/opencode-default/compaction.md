# OpenCode 内置 Agent 笔记：`compaction`

一句话定位：会话压缩用的内部 agent（hidden），用于把历史压缩成更短的上下文。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:155`

## Prompt

- `opencode/packages/opencode/src/agent/prompt/compaction.txt`
- 绑定位置：`opencode/packages/opencode/src/agent/agent.ts:160`

## 权限/工具（默认）

- `* : deny`（不允许任何工具调用）
- 来源：`opencode/packages/opencode/src/agent/agent.ts:161`

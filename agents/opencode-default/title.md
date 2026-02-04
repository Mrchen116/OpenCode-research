# OpenCode 内置 Agent 笔记：`title`

一句话定位：内部标题生成 agent（hidden），只输出标题。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:70`

## Prompt

- `opencode/packages/opencode/src/agent/prompt/title.txt`
- 绑定位置：`opencode/packages/opencode/src/agent/agent.ts:184`

## 权限/工具（默认）

- `* : deny`（不允许任何工具调用）
- 来源：`opencode/packages/opencode/src/agent/agent.ts:177`

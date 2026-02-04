# OpenCode 内置 Agent 笔记：`general`

一句话定位：通用 subagent；用于执行子任务（task/subtask），但默认禁用 todo 工具，避免把主线程的 todo 状态带入子任务。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:113`

## Prompt

- `general` 没有内置专属 prompt（`agent.ts` 未设置 `prompt` 字段）。

## 权限/工具（默认）

- 基础默认权限：`opencode/packages/opencode/src/agent/agent.ts:53`
- `general` 在默认基础上额外禁用：
  - `todoread: deny`
  - `todowrite: deny`
  - 来源：`opencode/packages/opencode/src/agent/agent.ts:116`

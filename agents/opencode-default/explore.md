# OpenCode 内置 Agent 笔记：`explore`

一句话定位：代码库探索/搜索 subagent；默认只开放“找文件/搜内容/读文件/查外部资料”的工具集合。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:128`

## Prompt

- `opencode/packages/opencode/src/agent/prompt/explore.txt`
- `agent.ts` 将其作为 `prompt: PROMPT_EXPLORE` 注入：`opencode/packages/opencode/src/agent/agent.ts:149`

## 权限/工具（默认）

- `explore` 是“白名单模式”：先 `* : deny`，再逐项 allow。
- 允许：
  - `grep`, `glob`, `list`, `bash`, `webfetch`, `websearch`, `codesearch`, `read`
  - `external_directory`（用于截断输出文件目录等）
  - 来源：`opencode/packages/opencode/src/agent/agent.ts:130`

## 备注

- 虽然权限里允许 `bash`，但 `explore.txt` 明确要求“不要用 bash 做任何会修改系统状态的事情”。

# OpenCode 内置 Agent 笔记：`build`

一句话定位：默认执行 agent；当用户不指定 agent 时（或配置无效时）通常会落到它。

## 定义位置

- `opencode/packages/opencode/src/agent/agent.ts:75`

## Prompt

- `build` 没有内置专属 prompt（`agent.ts` 未设置 `prompt` 字段）。
- plan/build 切换时的提示注入在 `opencode/packages/opencode/src/session/prompt.ts:1231`。

## 权限/工具（默认）

- 基础默认权限：`opencode/packages/opencode/src/agent/agent.ts:53`
- `build` 在默认基础上额外开启：
  - `question: allow`
  - `plan_enter: allow`
  - 来源：`opencode/packages/opencode/src/agent/agent.ts:79`

## 相关机制

- `default_agent`：`opencode/packages/opencode/src/config/config.ts:967`
- 选择默认 agent 的逻辑：
  - 读取配置并校验（不能是 subagent、不能 hidden）：`opencode/packages/opencode/src/agent/agent.ts:264`

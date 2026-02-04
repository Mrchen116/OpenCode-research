# OpenCode 内置 Agents（默认）

定义位置：`opencode/packages/opencode/src/agent/agent.ts`

OpenCode 默认内置了这些 agent：

| name | mode | hidden | 主要用途 |
| --- | --- | --- | --- |
| `build` | primary | no | 默认执行 agent（能做大多数工作） |
| `plan` | primary | no | 规划/只读模式（主要靠权限 + plan 提醒约束） |
| `general` | subagent | no | 通用子任务执行（默认禁用 todo 工具） |
| `explore` | subagent | no | 代码库探索/搜索（强约束只给搜索相关工具） |
| `compaction` | primary | yes | 会话压缩（内部用） |
| `title` | primary | yes | 生成会话标题（内部用） |
| `summary` | primary | yes | 总结会话（内部用） |

## 关键点

- 对外 README 里主要提到的内置 agent：
  - `build` / `plan`（可 Tab 切换）
  - `general`（可 `@general` 调用）
  - 来源：`opencode/README.md:94`
- `default_agent` 配置项：`opencode/packages/opencode/src/config/config.ts:967`
- plan mode 的“只读”主要由两部分共同实现：
  - 权限：`opencode/packages/opencode/src/agent/agent.ts:90`
  - user message 注入的 plan 提醒：`opencode/packages/opencode/src/session/prompt.ts:1231`

## 详细笔记

- `opencode-research/agents/opencode-default/build.md`
- `opencode-research/agents/opencode-default/plan.md`
- `opencode-research/agents/opencode-default/general.md`
- `opencode-research/agents/opencode-default/explore.md`
- `opencode-research/agents/opencode-default/compaction.md`
- `opencode-research/agents/opencode-default/title.md`
- `opencode-research/agents/opencode-default/summary.md`

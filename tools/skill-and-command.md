# Tools Category: Skill & Command

技能加载与命令模板工具；同名 `skill` 在 opencode 与 oh-my-opencode 中存在不同实现。

## `skill` (opencode)
功能: 加载技能指令文本。
描述摘要: 返回技能正文与基础目录。
来源文件: `opencode/packages/opencode/src/tool/skill.ts`
入参: `name: string`
出参: `{ title, output, metadata }`，`output` 为技能内容文本。

## `skill` (oh-my-opencode)
功能: 加载技能并可附 MCP 能力清单。
描述摘要: 输出包含技能说明、目录与可用 MCP 能力。
来源文件: `oh-my-opencode/src/tools/skill/tools.ts`
入参: `name: string`
出参: `string`（格式化的技能内容文本）。

## `skill_mcp`
功能: 调用技能内 MCP 工具/资源/提示词。
描述摘要: 需指定 `tool_name`/`resource_name`/`prompt_name` 之一。
来源文件: `oh-my-opencode/src/tools/skill-mcp/tools.ts`
入参: `mcp_name: string`, `tool_name?: string`, `resource_name?: string`, `prompt_name?: string`, `arguments?: string|object`, `grep?: string`
出参: `string`（JSON 字符串，可按 grep 过滤）。

## `slashcommand`
功能: 执行或展开斜杠命令。
描述摘要: 支持命令模糊匹配与参数替换。
来源文件: `oh-my-opencode/src/tools/slashcommand/tools.ts`
入参: `command: string`, `user_message?: string`
出参: `string`（命令内容或候选列表）。

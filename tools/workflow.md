# Tools Category: Workflow

流程控制与批量执行相关工具。

## `plan_enter`
功能: 建议切换至 plan agent。
描述摘要: 通过提问确认是否进入规划模式。
来源文件: `opencode/packages/opencode/src/tool/plan.ts`
入参: 无
出参: `{ title, output, metadata }`

## `plan_exit`
功能: 规划完成后切换至 build agent。
描述摘要: 通过提问确认是否进入实现模式。
来源文件: `opencode/packages/opencode/src/tool/plan.ts`
入参: 无
出参: `{ title, output, metadata }`

## `batch`
功能: 并行执行多工具调用。
描述摘要: 最多 25 个工具调用，部分失败不影响其他调用。
来源文件: `opencode/packages/opencode/src/tool/batch.ts`
入参: `tool_calls: { tool: string, parameters: object }[]`
出参: `{ title, output, metadata, attachments? }`，`metadata` 含成功/失败统计。

## `question`
功能: 向用户提问并收集答案。
描述摘要: 返回结构化回答并可用于继续执行。
来源文件: `opencode/packages/opencode/src/tool/question.ts`
入参: `questions: Question.Info[]`
出参: `{ title, output, metadata }`，`metadata.answers` 为回答数组。

## `invalid`
功能: 表示无效工具调用。
描述摘要: 仅用于错误回退与诊断提示。
来源文件: `opencode/packages/opencode/src/tool/invalid.ts`
入参: `tool: string`, `error: string`
出参: `{ title, output, metadata }`

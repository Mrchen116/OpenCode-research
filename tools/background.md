# Tools Category: Background

后台任务管理工具。

## `background_task`
功能: 启动后台子任务。
描述摘要: 定义存在，但默认注册列表中未启用（仅 `background_output`/`background_cancel` 被注册）。
来源文件: `oh-my-opencode/src/tools/background-task/tools.ts`
入参: `description: string`, `prompt: string`, `agent: string`
出参: `string`（任务创建状态与 Task ID）。

## `background_output`
功能: 获取后台任务输出。
描述摘要: 支持等待完成与返回完整会话内容。
来源文件: `oh-my-opencode/src/tools/background-task/tools.ts`
入参: `task_id: string`, `block?: boolean`, `timeout?: number`, `full_session?: boolean`, `include_thinking?: boolean`, `message_limit?: number`, `since_message_id?: string`, `include_tool_results?: boolean`, `thinking_max_chars?: number`
出参: `string`（任务状态、结果或全量会话文本）。

## `background_cancel`
功能: 取消后台任务。
描述摘要: 支持指定任务或取消全部。
来源文件: `oh-my-opencode/src/tools/background-task/tools.ts`
入参: `taskId?: string`, `all?: boolean`
出参: `string`（取消结果或错误提示）。

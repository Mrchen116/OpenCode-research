# Tools Category: Agent Delegation

子代理与任务委派工具。

## `task` (opencode)
功能: 启动 opencode 内建子代理会话并桥接提示词执行。
描述摘要: 以 `subagent_type` 选择内建 subagent，自动创建/复用会话并跟踪工具部件状态，返回 `<task_metadata>`（含 session_id、summary）。
来源文件: `opencode/packages/opencode/src/tool/task.ts`
入参: `description: string`, `prompt: string`, `subagent_type: string`, `session_id?: string`, `command?: string`
出参: `{ title, output, metadata }`，`output` 包含 `<task_metadata>`。

## `delegate_task`
功能: 在 oh-my-opencode 中按分类或指定代理委派任务。
描述摘要: `category`/`subagent_type` 二选一（互斥）；`category`=Sisyphus-Junior + 类别 prompt 追加，`subagent_type`=直接使用指定 agent 的完整 system prompt。`load_skills` 会先解析并**注入到 system prompt**（不是给子代理后续再调用），支持后台运行、task_id 返回与 session 续跑。
来源文件: `oh-my-opencode/src/tools/delegate-task/tools.ts`
入参: `load_skills: string[]`, `description: string`, `prompt: string`, `run_in_background: boolean`, `category?: string`, `subagent_type?: string`, `session_id?: string`, `command?: string`
出参: `string`（任务状态/错误/结果提示）。

## `call_omo_agent`
功能: 直接调用 Explore/Librarian 专用子代理。
描述摘要: 仅限 `explore`/`librarian`，可后台或同步执行；同步模式返回结果文本并附 `<task_metadata>`，后台模式返回任务状态。独立出来的目的是让“执行型子代理”还能启动 Explore/Librarian 做检索。
来源文件: `oh-my-opencode/src/tools/call-omo-agent/tools.ts`
入参: `description: string`, `prompt: string`, `subagent_type: string`, `run_in_background: boolean`, `session_id?: string`
出参: `string`（任务状态或同步结果，含 `<task_metadata>`）。

# Tools Category: Session

会话历史、检索与元信息查询工具。

## `session_list`
功能: 列出会话。
描述摘要: 支持日期范围与路径过滤。
来源文件: `oh-my-opencode/src/tools/session-manager/tools.ts`
入参: `limit?: number`, `from_date?: string`, `to_date?: string`, `project_path?: string`
出参: `string`（格式化会话列表或 `Error: ...`）。

## `session_read`
功能: 读取会话消息。
描述摘要: 可选择包含 Todo 与 transcript。
来源文件: `oh-my-opencode/src/tools/session-manager/tools.ts`
入参: `session_id: string`, `include_todos?: boolean`, `include_transcript?: boolean`, `limit?: number`
出参: `string`（格式化消息或 `Error: ...`）。

## `session_search`
功能: 搜索会话内容。
描述摘要: 可指定会话与大小写敏感。
来源文件: `oh-my-opencode/src/tools/session-manager/tools.ts`
入参: `query: string`, `session_id?: string`, `case_sensitive?: boolean`, `limit?: number`
出参: `string`（格式化搜索结果或 `Error: ...`）。

## `session_info`
功能: 获取会话元信息。
描述摘要: 返回会话统计与时间范围信息。
来源文件: `oh-my-opencode/src/tools/session-manager/tools.ts`
入参: `session_id: string`
出参: `string`（格式化会话信息或 `Error: ...`）。

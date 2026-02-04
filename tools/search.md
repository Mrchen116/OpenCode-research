# Tools Category: Search

覆盖本地与结构化搜索的工具集合；部分同名工具在 oh-my-opencode 中会覆盖默认实现。

## `grep` (opencode)
功能: 正则搜索文件内容。
描述摘要: 输出按修改时间排序并按文件分组展示匹配行。
来源文件: `opencode/packages/opencode/src/tool/grep.ts`
入参: `pattern: string`, `path?: string`, `include?: string`
出参: `{ title, output, metadata }`，`metadata` 含 `matches` 与 `truncated`。

## `glob` (opencode)
功能: 按 glob 模式匹配文件路径。
描述摘要: 返回按修改时间排序的文件路径列表。
来源文件: `opencode/packages/opencode/src/tool/glob.ts`
入参: `pattern: string`, `path?: string`
出参: `{ title, output, metadata }`，`metadata` 含 `count` 与 `truncated`。

## `grep` (oh-my-opencode)
功能: 正则搜索文件内容。
描述摘要: 60s 超时与 10MB 输出上限，结果按修改时间排序。
来源文件: `oh-my-opencode/src/tools/grep/tools.ts`
入参: `pattern: string`, `include?: string`, `path?: string`
出参: `string`（格式化结果或 `Error: ...`）。

## `glob` (oh-my-opencode)
功能: 按 glob 模式匹配文件路径。
描述摘要: 60s 超时与 100 文件上限，按修改时间排序。
来源文件: `oh-my-opencode/src/tools/glob/tools.ts`
入参: `pattern: string`, `path?: string`
出参: `string`（格式化结果或 `Error: ...`）。

## `ast_grep_search`
功能: AST 结构化搜索。
描述摘要: 使用 AST 模式与元变量匹配代码结构。
来源文件: `oh-my-opencode/src/tools/ast-grep/tools.ts`
入参: `pattern: string`, `lang: enum`, `paths?: string[]`, `globs?: string[]`, `context?: number`
出参: `string`（格式化结果，空结果可附提示，或 `Error: ...`）。

## `ast_grep_replace`
功能: AST 结构化替换。
描述摘要: 支持 dry-run 默认预览与变量保留替换。
来源文件: `oh-my-opencode/src/tools/ast-grep/tools.ts`
入参: `pattern: string`, `rewrite: string`, `lang: enum`, `paths?: string[]`, `globs?: string[]`, `dryRun?: boolean`
出参: `string`（格式化结果或 `Error: ...`）。

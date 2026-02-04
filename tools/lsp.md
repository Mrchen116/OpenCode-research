# Tools Category: LSP

代码智能相关工具；包含 opencode 的统一 LSP 工具和 oh-my-opencode 的拆分版工具。

## `lsp` (opencode)
功能: 统一入口的 LSP 操作。
描述摘要: 支持定义/引用/悬浮/符号/实现/调用层级等多种操作。
来源文件: `opencode/packages/opencode/src/tool/lsp.ts`
入参: `operation: enum`, `filePath: string`, `line: number`, `character: number`
出参: `{ title, output, metadata }`，`output` 为 JSON 字符串或无结果提示。

## `lsp_goto_definition`
功能: 跳转到定义。
描述摘要: 返回格式化位置列表或无结果提示。
来源文件: `oh-my-opencode/src/tools/lsp/tools.ts`
入参: `filePath: string`, `line: number`, `character: number`
出参: `string`（位置列表、无结果提示或 `Error: ...`）。

## `lsp_find_references`
功能: 查找引用。
描述摘要: 返回引用列表，必要时截断并提示。
来源文件: `oh-my-opencode/src/tools/lsp/tools.ts`
入参: `filePath: string`, `line: number`, `character: number`, `includeDeclaration?: boolean`
出参: `string`（引用列表、无结果提示或 `Error: ...`）。

## `lsp_symbols`
功能: 文档/工作区符号检索。
描述摘要: 支持 document/workspace 两种范围与结果截断。
来源文件: `oh-my-opencode/src/tools/lsp/tools.ts`
入参: `filePath: string`, `scope?: "document"|"workspace"`, `query?: string`, `limit?: number`
出参: `string`（符号列表、无结果提示或 `Error: ...`）。

## `lsp_diagnostics`
功能: LSP 诊断查询。
描述摘要: 可按严重级别过滤诊断输出。
来源文件: `oh-my-opencode/src/tools/lsp/tools.ts`
入参: `filePath: string`, `severity?: "error"|"warning"|"information"|"hint"|"all"`
出参: `string`（诊断列表或无结果提示）。

## `lsp_prepare_rename`
功能: 重命名前置校验。
描述摘要: 检查是否允许重命名并返回可编辑范围。
来源文件: `oh-my-opencode/src/tools/lsp/tools.ts`
入参: `filePath: string`, `line: number`, `character: number`
出参: `string`（格式化结果或 `Error: ...`）。

## `lsp_rename`
功能: 跨文件重命名。
描述摘要: 执行重命名并返回应用结果摘要。
来源文件: `oh-my-opencode/src/tools/lsp/tools.ts`
入参: `filePath: string`, `line: number`, `character: number`, `newName: string`
出参: `string`（应用结果或 `Error: ...`）。

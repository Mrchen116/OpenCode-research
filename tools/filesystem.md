# Tools Category: Filesystem

面向本地文件读写与编辑的工具。

## `read`
功能: 读取本地文件内容或媒体文件。
描述摘要: 按行读取文本；图像/PDF 会以附件返回并给出简短说明。
来源文件: `opencode/packages/opencode/src/tool/read.ts`
入参: `filePath: string`, `offset?: number`, `limit?: number`
出参: `{ title, output, metadata, attachments? }`，`output` 为 `<file>...` 文本或媒体读取提示。

## `write`
功能: 写入或覆盖文件内容。
描述摘要: 写入后触发文件变更事件并汇报 LSP 诊断。
来源文件: `opencode/packages/opencode/src/tool/write.ts`
入参: `content: string`, `filePath: string`
出参: `{ title, output, metadata }`，`output` 含成功提示及可选诊断块。

## `edit`
功能: 精确字符串替换式编辑。
描述摘要: 基于 oldString/newString 执行替换，失败会报错并提示原因。
来源文件: `opencode/packages/opencode/src/tool/edit.ts`
入参: `filePath: string`, `oldString: string`, `newString: string`, `replaceAll?: boolean`
出参: `{ title, output, metadata }`，`metadata` 包含 diff 与诊断信息。

## `multiedit`
功能: 对同一文件执行多次编辑。
描述摘要: 顺序执行多组 edit，输出采用最后一次 edit 的结果。
来源文件: `opencode/packages/opencode/src/tool/multiedit.ts`
入参: `filePath: string`, `edits: { oldString: string, newString: string, replaceAll?: boolean }[]`
出参: `{ title, output, metadata }`，`metadata.results` 汇总每次 edit 的元数据。

## `apply_patch`
功能: 以补丁格式批量增删改文件。
描述摘要: 解析补丁并生成多文件变更，校验权限后执行。
来源文件: `opencode/packages/opencode/src/tool/apply_patch.ts`
入参: `patchText: string`
出参: `{ title, output, metadata }`，`output` 汇总 A/M/D 并附 LSP 诊断。

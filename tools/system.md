# Tools Category: System

系统交互与环境类工具。

## `bash`
功能: 执行非交互式 shell 命令。
描述摘要: 支持超时与工作目录，输出合并 stdout/stderr。
来源文件: `opencode/packages/opencode/src/tool/bash.ts`
入参: `command: string`, `timeout?: number`, `workdir?: string`, `description: string`
出参: `{ title, output, metadata }`，`metadata` 含 `output` 与 `exit`。

## `interactive_bash`
功能: 执行 tmux 子命令。
描述摘要: 用于管理交互式 tmux，会阻止部分命令并给出替代建议。
来源文件: `oh-my-opencode/src/tools/interactive-bash/tools.ts`
入参: `tmux_command: string`
出参: `string`（stdout 或错误提示）。

## `look_at`
功能: 多模态文件分析。
描述摘要: 通过 multimodal-looker 提取 PDF/图片/音视频信息。
来源文件: `oh-my-opencode/src/tools/look-at/tools.ts`
入参: `file_path: string`, `goal: string`
出参: `string`（提取结果或错误提示）。

## `list`
功能: 列出目录结构。
描述摘要: 输出树形结构并支持忽略模式。
来源文件: `opencode/packages/opencode/src/tool/ls.ts`
入参: `path?: string`, `ignore?: string[]`
出参: `{ title, output, metadata }`，`output` 为树状目录列表。

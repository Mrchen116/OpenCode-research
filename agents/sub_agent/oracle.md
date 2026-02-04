# OhMyOpenCode 主 Agent 报告 (Oracle)

一句话定位：只读高智商顾问，用于复杂架构决策、疑难调试与高难度推理；产出可直接执行的建议，而非实现。

## 1. 身份与系统提示词 (Identity & Prompt)

**Role & Scope（只读顾问）**
Oracle 被定义为“战略技术顾问”，以独立咨询的方式回答复杂问题；偏好最小复杂度、复用现有模式、给出单一路径建议，并按 Quick/Short/Medium/Large 估算投入。

**Use When / Avoid When（触发条件）**
内置元数据明确 Oracle 适用于架构权衡、重要实现后的自审、2+ 次失败后的硬调试等；避免用于简单文件操作或第一次尝试的常规修复。

来源定位：`oh-my-opencode/src/agents/oracle.ts:8`, `oh-my-opencode/src/agents/oracle.ts:34`
系统提示词来源：`oh-my-opencode/src/agents/oracle.ts:34`

## 2. 工具系统 (可调用工具)

**权限差异 (Oracle 专属)**
- 明确禁止：`write`, `edit`, `task`, `delegate_task`
- 来源：`oh-my-opencode/src/agents/oracle.ts:100`

**权限合并与裁决规则 (重要，决定“能不能用”)**
- 权限不是“文档列了就能用”，而是由多层规则合并后裁决。
- 规则来源：Agent 自身权限 + 会话权限（用户/项目配置）+ 用户工具禁用项。
- 运行时会按合并结果剔除工具（被 deny 的工具不会出现在可用工具列表里）。
- 未命中任何规则时默认是 `ask`（调用时弹权限请求）。
- `edit` 权限覆盖 `edit/write/patch/multiedit`。
- 来源：`opencode/packages/opencode/src/permission/next.ts:63`, `opencode/packages/opencode/src/permission/next.ts:231`, `opencode/packages/opencode/src/session/llm.ts:268`

**工具清单与用途 (目录清单；是否可用由权限模型决定)**

LSP
- `lsp_goto_definition`：跳转到定义
- `lsp_find_references`：查找引用
- `lsp_symbols`：符号检索
- `lsp_diagnostics`：诊断错误/警告
- `lsp_prepare_rename`：校验重命名可行性
- `lsp_rename`：跨文件重命名

AST-Grep
- `ast_grep_search`：按语法树模式搜索
- `ast_grep_replace`：按语法树模式替换

Search
- `grep`：内容正则搜索
- `glob`：文件路径匹配

Session
- `session_list`：列出会话
- `session_read`：读取会话消息
- `session_search`：搜索会话内容
- `session_info`：会话元数据

Agent
- `delegate_task`：委托子代理或按 category 生成 Sisyphus-Junior
- `call_omo_agent`：直接调用指定代理

Background
- `background_output`：获取后台任务结果
- `background_cancel`：取消后台任务

System
- `interactive_bash`：tmux 交互式会话
- `look_at`：分析图像/PDF 等多模态文件

Skill
- `skill`：加载并执行技能
- `skill_mcp`：调用技能内 MCP
- `slashcommand`：运行斜杠命令

Core
- `bash`：执行命令
- `read`：读取文件
- `write`：创建文件
- `edit`：修改文件
- `todowrite`：写入 Todo 列表
- `todoread`：读取 Todo 列表
- `question`：向用户提问
- `webfetch`：抓取网页内容
- `websearch`：网页搜索
- `task`：任务状态工具

工具来源：`oh-my-opencode/src/tools/AGENTS.md:1`

## 3. 可调用 Sub-agent (含用途)

Oracle 自身是只读子代理；由于 `delegate_task` 被禁用，不能再向下分发任务。

来源：`oh-my-opencode/src/agents/oracle.ts:100`

## 4. Hook (生命周期钩子与作用)

Oracle 作为子代理会受到通用质量/恢复钩子的影响（例如输出截断、注释约束、子代理提问拦截等）。

来源：`oh-my-opencode/src/hooks/AGENTS.md:62`

## 5. Skills (可加载技能与作用)

Oracle 为只读顾问，通常不需要技能；技能清单由系统统一提供。

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`

## 6. 关键差异小结

- Oracle 是“只读高推理顾问”：给结论与行动路径，不写实现。
- Metis/Momus 也是只读，但分别面向“规划前分析 / 计划审查”；Oracle 更偏“决策与调试推理”。

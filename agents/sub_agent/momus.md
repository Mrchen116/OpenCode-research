# OhMyOpenCode 主 Agent 报告 (Momus)

一句话定位：计划审阅者（Plan Reviewer），只读验证“计划是否可执行、引用是否有效”，以阻塞点为中心做高强度挑错。

## 1. 身份与系统提示词 (Identity & Prompt)

**Role & Job（只查阻塞点）**
Momus 的系统提示词将其定位为“务实的计划审阅者”：只回答“开发者能否不被卡住地执行这份计划”。它只检查引用是否存在/相关、任务是否有足够上下文开始；默认倾向 OKAY，只有真阻塞才 REJECT，并限制最多列 3 条阻塞问题。

**Input Extraction（从输入里提取单一计划路径）**
第一规则要求从输入中提取唯一的 `.sisyphus/plans/*.md` 路径并读取；若没有或存在多个路径则拒绝。

来源定位：`oh-my-opencode/src/agents/momus.ts:8`, `oh-my-opencode/src/agents/momus.ts:22`, `oh-my-opencode/src/agents/momus.ts:191`
系统提示词来源：`oh-my-opencode/src/agents/momus.ts:22`

## 2. 工具系统 (可调用工具)

**权限差异 (Momus 专属)**
- 明确禁止：`write`, `edit`, `task`, `delegate_task`
- 来源：`oh-my-opencode/src/agents/momus.ts:191`

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

Momus 自身是只读子代理；由于 `delegate_task` 被禁用，不能再向下分发任务。

来源：`oh-my-opencode/src/agents/momus.ts:191`

## 4. Hook (生命周期钩子与作用)

Momus 通常由 Prometheus 的“计划审阅选项”或主编排器触发；运行期仍受通用钩子体系影响（例如输出截断与恢复机制）。

来源：`oh-my-opencode/src/hooks/AGENTS.md:62`

## 5. Skills (可加载技能与作用)

Momus 为只读审阅，不需要技能；技能由执行侧使用。

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`

## 6. 关键差异小结

- Momus 是“计划审阅”：只查计划可执行性/引用有效性，默认通过。
- Metis 是“规划前分析”：在计划生成前找缺口。
- Oracle 是“推理顾问”：做架构/调试的高智商咨询。

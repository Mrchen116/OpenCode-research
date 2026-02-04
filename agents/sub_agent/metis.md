# OhMyOpenCode 主 Agent 报告 (Metis)

一句话定位：规划前顾问（Pre‑Planning Consultant），专门在进入规划/执行前识别隐含意图、歧义与 AI 失败点，并产出可执行的澄清问题与指令。

## 1. 身份与系统提示词 (Identity & Prompt)

**Role & Mission（规划前分析）**
Metis 被定义为“在规划之前运行”的分析代理：先做意图分类（Refactor/Build/Mid-sized/Collaborative/Architecture/Research），再按类别给出风险、澄清问题、对 Prometheus 的硬约束（MUST/MUST NOT）和可执行验收标准指令。

**Guardrails（防 AI Slop）**
系统提示词内反复强调：先探索再提问、明确 Must Not Have、验收标准必须可由 agent 执行（curl/bun/playwright），禁止依赖用户手工验证。

来源定位：`oh-my-opencode/src/agents/metis.ts:7`, `oh-my-opencode/src/agents/metis.ts:21`, `oh-my-opencode/src/agents/metis.ts:313`
系统提示词来源：`oh-my-opencode/src/agents/metis.ts:21`

## 2. 工具系统 (可调用工具)

**权限差异 (Metis 专属)**
- 明确禁止：`write`, `edit`, `task`, `delegate_task`
- 来源：`oh-my-opencode/src/agents/metis.ts:306`

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

Metis 自身是只读子代理；由于 `delegate_task` 被禁用，不能再向下分发任务。

来源：`oh-my-opencode/src/agents/metis.ts:306`

## 4. Hook (生命周期钩子与作用)

Metis 通常被主编排器在规划前调用；运行期仍受通用钩子体系影响（例如输出截断与恢复机制）。

来源：`oh-my-opencode/src/hooks/AGENTS.md:62`

## 5. Skills (可加载技能与作用)

Metis 为只读分析顾问，通常不直接使用技能；技能由主编排器在执行阶段选择。

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`

## 6. 关键差异小结

- Metis 是“规划前分析”：识别歧义/隐含需求/失败点，产出问题与指令。
- Prometheus 是“规划器”：负责访谈与产出计划文件。
- Momus 是“计划审阅”：专查计划是否可执行、引用是否有效。

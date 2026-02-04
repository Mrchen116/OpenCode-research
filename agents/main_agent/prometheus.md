# OhMyOpenCode 主 Agent 报告 (Prometheus)

一句话定位：规划专用主代理，负责访谈、研究与生成工作计划；不执行代码，只产出 `.md` 计划文件。

## 1. 身份与系统提示词 (Identity & Prompt)

**Critical Identity & Request Interpretation**
开头 CRITICAL IDENTITY 强制定义 Prometheus 是规划者，不是实现者，绝不写代码或执行任务；REQUEST INTERPRETATION 明确用户说“做/修/实现”全部解释为“生成计划”。

**Identity Constraints / Forbidden / Outputs**
随后给出 Identity Constraints 表，明确“你是什么/你不是什么”，并列出 FORBIDDEN ACTIONS（写代码、改源码、运行实现命令、写非 .md）和 ONLY OUTPUTS（澄清问题、研究结果、计划文件、草稿文件）。当用户要求直接实现时，提示词要求拒绝并解释规划价值。

**Absolute Constraints（Interview → Plan）**
进入 ABSOLUTE CONSTRAINTS：默认 Interview Mode 先访谈澄清；每次对话后做 Clearance Checklist（目标清晰、范围明确、无关键歧义、技术路径已定、测试策略已定、无阻塞问题）以决定是否自动进入计划生成。

**Markdown‑only / Plan Output / Single Plan**
强制 Markdown‑only，并限定计划输出路径为 `.sisyphus/plans/*.md`；SINGLE PLAN MANDATE 要求无论任务多大都只能生成一份计划，禁止拆分多计划。

**Draft as Memory**
访谈期持续写 `.sisyphus/drafts/*.md`，并给出草稿结构，用作长期记忆与事实记录。

**Turn Termination Rules**
每轮必须以提问/等待/转计划结束，不允许无收束的结尾。

来源定位：`oh-my-opencode/src/agents/prometheus/identity-constraints.ts:11`, `oh-my-opencode/src/agents/prometheus/identity-constraints.ts:42`, `oh-my-opencode/src/agents/prometheus/identity-constraints.ts:80`, `oh-my-opencode/src/agents/prometheus/identity-constraints.ts:109`, `oh-my-opencode/src/agents/prometheus/identity-constraints.ts:117`, `oh-my-opencode/src/agents/prometheus/identity-constraints.ts:140`, `oh-my-opencode/src/agents/prometheus/identity-constraints.ts:190`
系统提示词来源：`oh-my-opencode/src/agents/prometheus/index.ts:30`

## 2. 工具系统 (可调用工具)

**权限差异 (Prometheus 专属)**
- PROMETHEUS_PERMISSION：允许 `edit`, `bash`, `webfetch`, `question`
- 写入被 `prometheus-md-only` 钩子限制为 `.md`
- 运行时权限补充：允许 `delegate_task`, `question`, `task_*`, `teammate`，禁止 `call_omo_agent`
- 来源：`oh-my-opencode/src/agents/prometheus/index.ts:42`, `oh-my-opencode/src/plugin-handlers/config-handler.ts:433`

**权限合并与裁决规则 (重要，决定“能不能用”)**
- 权限不是“文档列了就能用”，而是由多层规则合并后裁决。
- 规则来源：Agent 自身权限 + 会话权限（用户/项目配置）+ 用户工具禁用项。
- 运行时会按合并结果剔除工具（被 deny 的工具不会出现在可用工具列表里）。
- 未命中任何规则时默认是 `ask`（调用时弹权限请求）。
- `edit` 权限覆盖 `edit/write/patch/multiedit`。
- 来源：`opencode/packages/opencode/src/permission/next.ts:63`, `opencode/packages/opencode/src/permission/next.ts:231`, `opencode/packages/opencode/src/session/llm.ts:268`

**工具清单与用途 (全部列出即全部解释)**

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
- `call_omo_agent`：直接调用指定代理（Prometheus 被禁用）

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
- `edit`：修改文件（限 `.md`）
- `todowrite`：写入 Todo 列表
- `todoread`：读取 Todo 列表
- `question`：向用户提问
- `webfetch`：抓取网页内容
- `websearch`：网页搜索
- `codesearch`：代码搜索
- `task`：任务状态工具

工具来源：`oh-my-opencode/src/tools/AGENTS.md:1`, `opencode-research/main_agent.md:28`

## 3. 可调用 Sub-agent (含用途)

Prometheus 在规划阶段会调用研究与审阅子代理。

- `Librarian`：外部文档与开源检索
- `Explore`：代码库快速定位
- `Metis`：规划缺口与边界分析
- `Momus`：高精度计划审查
- `Oracle`：架构/疑难推理顾问（只读）

来源：`oh-my-opencode/src/agents/AGENTS.md:7`

## 4. Hook (生命周期钩子与作用)

- `prometheus-md-only`：限制仅写入 `.md`
- `auto-slash-command`：自动识别 `/start-work` 等命令
- `directory-agents-injector`：注入目录级 AGENTS.md
- `category-skill-reminder`：提醒选择 category + skills
- `subagent-question-blocker`：阻止子代理直接提问用户

来源：`oh-my-opencode/src/hooks/AGENTS.md:16`

## 5. Skills (可加载技能与作用)

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `agent-browser`：浏览器自动化（CLI 方式，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`, `oh-my-opencode/src/features/builtin-skills/*/SKILL.md:1`

## 6. 关键差异小结

- Prometheus 只产出计划，不执行实现。
- Sisyphus 负责理解用户并调度执行。
- Atlas 负责按计划编排执行。
- Hephaestus 负责深度实现。

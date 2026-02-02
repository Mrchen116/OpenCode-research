# OhMyOpenCode 主 Agent 报告 (Prometheus)

一句话定位：规划专用主代理，负责访谈、研究与生成工作计划；不执行代码，只产出 `.md` 计划文件。

## 1. 身份与系统提示词 (Identity & Prompt)

**身份设定 (核心区别)**
- 角色：Strategic Planning Consultant（规划顾问）。
- 核心约束：永远只做规划，不实现代码。
- 用户说“做/修/实现”：统一解释为“生成工作计划”。

**提示词摘要 (你能从中预期什么)**
- 先访谈再规划：先问清需求，再生成计划。
- 规划前会调用 Metis 做缺口扫描。
- 可选高精度模式：交给 Momus 反复审查计划。
- 只能写 `.md`，且输出路径固定为 `.sisyphus/plans/*.md`。

**系统提示词来源**
- 系统提示词拼装：`oh-my-opencode/src/agents/prometheus/index.ts:30`
- 身份与约束：`oh-my-opencode/src/agents/prometheus/identity-constraints.ts:8`
- 访谈模式：`oh-my-opencode/src/agents/prometheus/interview-mode.ts:8`
- 计划生成：`oh-my-opencode/src/agents/prometheus/plan-generation.ts:8`
- 高精度审核：`oh-my-opencode/src/agents/prometheus/high-accuracy-mode.ts:7`
- 计划模板：`oh-my-opencode/src/agents/prometheus/plan-template.ts:8`
- 行为总结：`oh-my-opencode/src/agents/prometheus/behavioral-summary.ts:7`

## 2. 工具系统 (可调用工具)

**权限差异 (Prometheus 专属)**
- PROMETHEUS_PERMISSION：允许 `edit`, `bash`, `webfetch`, `question`
- 写入被 `prometheus-md-only` 钩子限制为 `.md`
- 运行时权限补充：允许 `delegate_task`, `question`, `task_*`, `teammate`，禁止 `call_omo_agent`
- 来源：`oh-my-opencode/src/agents/prometheus/index.ts:42`, `oh-my-opencode/src/plugin-handlers/config-handler.ts:433`

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

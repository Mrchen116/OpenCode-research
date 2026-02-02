# OhMyOpenCode 主 Agent 报告 (Sisyphus)

一句话定位：默认主编排器，负责理解用户意图、拆解任务、调度子代理并验证输出；面向“从请求到交付”的完整闭环。

## 1. 身份与系统提示词 (Identity & Prompt)

**身份设定 (核心区别)**
- 角色：Primary Orchestrator（顶层编排，直接对用户交互）。
- 性格与标准：SF Bay Area engineer；“Work, delegate, verify, ship. No AI slop.”
- 关键能力：意图分流、代码库成熟度评估、强制并行、强制验证。

**提示词摘要 (你能从中预期什么)**
- 先分类再行动：Trivial/Explicit/Exploratory/Open-ended/Ambiguous。
- 先探索后执行：优先用 Explore/Librarian 并行补上下文。
- 默认委托：能委托就委托，只有超简单才自己做。
- 强制验证：改动后必须诊断、构建、测试（如适用）。

**系统提示词来源**
- 动态提示词入口：`oh-my-opencode/src/agents/sisyphus.ts:26`
- 身份与风格：`oh-my-opencode/src/agents/sisyphus.ts:42`
- 动态拼装器：`oh-my-opencode/src/agents/dynamic-agent-prompt-builder.ts:65`

## 2. 工具系统 (可调用工具)

**权限差异 (Sisyphus 专属)**
- 允许：`delegate_task`, `question`, `task_*`, `teammate`
- 禁止：`call_omo_agent`
- 来源：`oh-my-opencode/src/plugin-handlers/config-handler.ts:425`

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
- `call_omo_agent`：直接调用指定代理（Sisyphus 被禁用）

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
- `codesearch`：代码搜索
- `task`：任务状态工具

工具来源：`oh-my-opencode/src/tools/AGENTS.md:1`, `opencode-research/main_agent.md:28`

## 3. 可调用 Sub-agent (含用途)

Sisyphus 使用 `delegate_task` 调用子代理或按 category 生成 Sisyphus-Junior。

- `Hephaestus`：深度自治执行（复杂实现）
- `Oracle`：架构/疑难推理顾问（只读）
- `Librarian`：外部文档与开源检索
- `Explore`：代码库快速定位
- `Multimodal-Looker`：图片/PDF 分析
- `Metis`：规划前的漏洞与边界分析
- `Momus`：规划审查与挑错
- `Sisyphus-Junior`：具体实现执行者（按 category 生成）

来源：`oh-my-opencode/src/agents/AGENTS.md:7`

## 4. Hook (生命周期钩子与作用)

- `atlas`：全局编排与任务状态维护
- `todo-continuation-enforcer`：强制 Todo 连续执行
- `start-work`：启动 Sisyphus 工作会话
- `category-skill-reminder`：提醒选择 category + skills
- `subagent-question-blocker`：阻止子代理直接提问用户
- `sisyphus-junior-notepad`：为 Junior 注入记事本
- `tool-output-truncator`：截断过长工具输出
- `directory-agents-injector`：注入目录级 AGENTS.md

来源：`oh-my-opencode/src/hooks/AGENTS.md:16`

## 5. Skills (可加载技能与作用)

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `agent-browser`：浏览器自动化（CLI 方式，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`, `oh-my-opencode/src/features/builtin-skills/*/SKILL.md:1`

## 6. 关键差异小结

- Sisyphus 是“默认主入口”：面向用户、做意图分流与调度。
- Atlas 是“执行编排器”：专注计划执行与验证，不直接写代码。
- Prometheus 是“规划器”：访谈、生成计划，不执行实现。
- Hephaestus 是“深度执行者”：自主完成复杂实现。

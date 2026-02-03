# OhMyOpenCode 主 Agent 报告 (Atlas)

一句话定位：计划执行编排器，专门把“工作计划”拆解为可执行任务、并行分发并强制验证；自身不写代码。

## 1. 身份与系统提示词 (Identity & Prompt)

**Identity & Mission**
Identity 段把 Atlas 定义为 Master Orchestrator，强调“指挥不演奏”，只负责编排、协调与验证，绝不自己写代码。Mission 明确目标是“Complete ALL tasks in a work plan”，每次委托只做一个任务，可并行则并行，所有结果必须验证。

**Delegation System（两种互斥模式）**
`delegate_task` 只有两种互斥模式：Category 模式（`delegate_task(category=..., load_skills=[...])`）会生成 `Sisyphus-Junior-{category}`，适合领域任务；Agent 模式（`delegate_task(subagent_type=..., load_skills=[...])`）直接调用专精子代理（如 oracle、librarian）。

**6‑Section Prompt Structure（委托提示词模板）**
委托提示词必须包含六段：
1) TASK：引用计划里的 checkbox；
2) EXPECTED OUTCOME：文件清单/行为/验证命令；
3) REQUIRED TOOLS：需要用的工具及用途；
4) MUST DO：必须遵循的模式/测试/记事本更新；
5) MUST NOT DO：禁止范围/禁止改动/禁止跳过验证；
6) CONTEXT：notepad 路径/继承经验/依赖关系。提示词过短视为失败。

**Workflow（执行编排顺序）**
先用 TodoWrite 注册“完成全部任务”，再解析计划并构建并行/依赖图，随后初始化 notepad 目录结构，为后续委托提供上下文。

来源定位：`oh-my-opencode/src/agents/atlas.ts:125`, `oh-my-opencode/src/agents/atlas.ts:141`, `oh-my-opencode/src/agents/atlas.ts:173`, `oh-my-opencode/src/agents/atlas.ts:217`, `oh-my-opencode/src/agents/atlas.ts:246`
系统提示词来源：`oh-my-opencode/src/agents/atlas.ts:125`

## 2. 工具系统 (可调用工具)

**权限差异 (Atlas 专属)**
- 允许：`delegate_task`, `task_*`, `teammate`
- 禁止：`task`, `call_omo_agent`
- 来源：`oh-my-opencode/src/plugin-handlers/config-handler.ts:421`

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
- `call_omo_agent`：直接调用指定代理（Atlas 被禁用）

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

Atlas 通过 `delegate_task` 调用子代理或按 category 生成 Sisyphus-Junior。

- `Hephaestus`：深度自治执行
- `Oracle`：架构/疑难推理顾问（只读）
- `Librarian`：外部文档与开源检索
- `Explore`：代码库快速定位
- `Multimodal-Looker`：图片/PDF 分析
- `Metis`：规划前的漏洞与边界分析
- `Momus`：规划审查与挑错
- `Sisyphus-Junior`：具体实现执行者（按 category 生成）

来源：`oh-my-opencode/src/agents/AGENTS.md:7`

## 4. Hook (生命周期钩子与作用)

- `atlas`：主编排钩子，维护全局状态与 Todo
- `todo-continuation-enforcer`：强制 Todo 连续执行
- `delegate-task-retry`：失败任务自动重试
- `task-resume-info`：记录可恢复信息
- `tool-output-truncator`：截断过长工具输出
- `start-work`：启动 Sisyphus 工作会话

来源：`oh-my-opencode/src/hooks/AGENTS.md:16`

## 5. Skills (可加载技能与作用)

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `agent-browser`：浏览器自动化（CLI 方式，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`, `oh-my-opencode/src/features/builtin-skills/*/SKILL.md:1`

## 6. 关键差异小结

- Atlas 专注“计划执行编排”，不是对话入口。
- Sisyphus 负责理解用户、拆解任务。
- Prometheus 只负责规划。
- Hephaestus 负责深度执行。

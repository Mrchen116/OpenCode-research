# OhMyOpenCode 主 Agent 报告 (Hephaestus)

一句话定位：深度自治执行者，专注“从需求到实现”的独立完成与强验证，适合复杂实现和大块工作。

## 1. 身份与系统提示词 (Identity & Prompt)

**身份设定 (核心区别)**
- 角色：Autonomous Deep Worker（深度执行者）。
- 等级：Senior Staff Engineer 级别的自主执行。
- 行为：先探索、再决策、最后闭环验证；不猜测，不提前停。

**提示词摘要 (你能从中预期什么)**
- 默认探索优先：先并行 explore/librarian 再执行。
- 任务必须闭环：诊断、构建、测试证据齐全才算完成。
- 复杂任务更倾向独立完成，不频繁与用户交互。

**系统提示词来源**
- Prompt 入口：`oh-my-opencode/src/agents/hephaestus.ts:33`
- 身份与能力描述：`oh-my-opencode/src/agents/hephaestus.ts:49`
- 强约束与完成标准：`oh-my-opencode/src/agents/hephaestus.ts:67`
- 动态拼装器：`oh-my-opencode/src/agents/dynamic-agent-prompt-builder.ts:65`

## 2. 工具系统 (可调用工具)

**权限差异 (Hephaestus 专属)**
- 允许：`delegate_task`, `question`
- 禁止：`call_omo_agent`
- 来源：`oh-my-opencode/src/plugin-handlers/config-handler.ts:429`

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
- `call_omo_agent`：直接调用指定代理（Hephaestus 被禁用）

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

Hephaestus 可通过 `delegate_task` 调用子代理。

- `Oracle`：架构/疑难推理顾问（只读）
- `Librarian`：外部文档与开源检索
- `Explore`：代码库快速定位
- `Multimodal-Looker`：图片/PDF 分析
- `Metis`：规划前的漏洞与边界分析
- `Momus`：规划审查与挑错
- `Sisyphus-Junior`：具体实现执行者（按 category 生成）

来源：`oh-my-opencode/src/agents/AGENTS.md:7`

## 4. Hook (生命周期钩子与作用)

- `todo-continuation-enforcer`：强制 Todo 连续执行
- `category-skill-reminder`：提醒选择 category + skills
- `subagent-question-blocker`：阻止子代理直接提问用户
- `tool-output-truncator`：截断过长工具输出
- `comment-checker`：限制多余注释
- `edit-error-recovery`：修复 edit 失败

来源：`oh-my-opencode/src/hooks/AGENTS.md:16`

## 5. Skills (可加载技能与作用)

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `agent-browser`：浏览器自动化（CLI 方式，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`, `oh-my-opencode/src/features/builtin-skills/*/SKILL.md:1`

## 6. 关键差异小结

- Hephaestus 是“执行型深度工作者”：更少对话，更多闭环执行。
- Sisyphus 是“对话型主编排器”：负责理解与调度。
- Atlas 是“计划执行编排器”：专注计划完成与验证。
- Prometheus 是“规划器”：只产出计划，不写实现。

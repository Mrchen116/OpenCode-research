# OhMyOpenCode 主 Agent 报告 (Explore)

一句话定位：代码库极速探索员，负责并行搜索与结构化结果输出，回答“在哪里/哪段代码/如何关联”的问题。

## 1. 身份与系统提示词 (Identity & Prompt)

**Role & Mission**
Explore 被定义为“代码库搜索专家”，目标是快速定位文件与实现细节，并给出可直接行动的结果。

**Intent Analysis（先理解再搜索）**
每次请求必须先写出“字面请求/真实需求/成功标准”，避免只回答表面问题。

**Parallel Execution（首轮 3+ 并行工具）**
第一次行动必须并行调用 3 个以上工具；只有在结果依赖前置输出时才允许串行。

**Structured Results（强制结构化输出）**
输出必须包含 `<results>` 块，内含 `<files>`（绝对路径 + 相关性理由）、`<answer>`（直接回答需求）、`<next_steps>`（下一步行动）。

**Success / Failure Criteria**
成功标准包括：路径必须是绝对路径、覆盖所有相关匹配、可让调用者直接继续；未满足则视为失败。

**Tool Strategy**
语义用 LSP，结构用 AST-Grep，文本用 grep，文件模式用 glob，历史演进用 git。

**Trigger & Use When**
触发器：涉及 2 个以上模块即后台调用；适用于多搜索角度、不熟悉模块结构、跨层模式发现。

来源定位：`oh-my-opencode/src/agents/explore.ts:7`, `oh-my-opencode/src/agents/explore.ts:36`, `oh-my-opencode/src/agents/explore.ts:43`, `oh-my-opencode/src/agents/explore.ts:52`, `oh-my-opencode/src/agents/explore.ts:65`, `oh-my-opencode/src/agents/explore.ts:88`, `oh-my-opencode/src/agents/explore.ts:112`
系统提示词来源：`oh-my-opencode/src/agents/explore.ts:43`

## 2. 工具系统 (可调用工具)

**权限差异 (Explore 专属)**
- 明确禁止：`write`, `edit`, `task`, `delegate_task`, `call_omo_agent`
- 其余工具：由“全局权限规则 + 会话权限”决定，未命中规则时默认是 `ask`
- 来源：`oh-my-opencode/src/agents/explore.ts:27`, `oh-my-opencode/src/agents/AGENTS.md:54`

**权限合并与裁决规则 (重要，决定“能不能用”)**
- 权限不是“文档列了就能用”，而是由多层规则合并后裁决。
- 规则来源：Agent 自身权限 + 会话权限（用户/项目配置）+ 用户工具禁用项。
- 运行时会按合并结果剔除工具（被 deny 的工具不会出现在可用工具列表里）。
- 未命中任何规则时默认是 `ask`（调用时弹权限请求）。
- `edit` 权限覆盖 `edit/write/patch/multiedit`。
- 来源：`opencode/packages/opencode/src/permission/next.ts:63`, `opencode/packages/opencode/src/permission/next.ts:231`, `opencode/packages/opencode/src/session/llm.ts:268`

**重要说明：这里的“工具清单”是完整目录，不等于“全部可直接用”**
- Explore 的“显式禁止”只有上面五项；其他工具是否可用取决于权限规则合并结果。
- 运行时会移除被禁用的工具：`session/llm.ts` 会按权限剔除工具。
- 参考：`opencode/packages/opencode/src/session/llm.ts:268`

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
- `call_omo_agent`：直接调用指定代理（Explore 被禁用）

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

Explore 不能调用子代理（`delegate_task` 被禁用）。

来源：`oh-my-opencode/src/agents/explore.ts:27`, `oh-my-opencode/src/agents/AGENTS.md:54`

## 4. Hook (生命周期钩子与作用)

- `subagent-question-blocker`：阻止子代理直接提问用户
- `tool-output-truncator`：截断过长工具输出
- `comment-checker`：限制多余注释
- `directory-agents-injector`：注入目录级 AGENTS.md

来源：`oh-my-opencode/src/hooks/AGENTS.md:22`, `oh-my-opencode/src/hooks/AGENTS.md:37`, `oh-my-opencode/src/hooks/AGENTS.md:49`, `oh-my-opencode/src/hooks/AGENTS.md:25`

## 5. Skills (可加载技能与作用)

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `agent-browser`：浏览器自动化（CLI 方式，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`, `oh-my-opencode/src/features/builtin-skills/*/SKILL.md:1`

## 6. 关键差异小结

- Explore 面向“内部代码探索”，强调覆盖与可执行线索。
- Librarian 面向“外部资料检索”，强调引用与证据化输出。
- Oracle 用于复杂推理与架构咨询，Explore 不做推理，只给定位与上下文。

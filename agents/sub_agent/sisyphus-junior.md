# OhMyOpenCode 主 Agent 报告 (Sisyphus-Junior)

一句话定位：聚焦执行器（delegated executor），通常由 `delegate_task(category=...)` 生成；强调严格 Todo 驱动与验证闭环，禁止把实现任务再委托出去。

## 1. 身份与系统提示词 (Identity & Prompt)

Sisyphus-Junior 的系统提示词使用 XML-like 分段（Role/Constraints/Todo/Verification/Style）。
这里严格按系统提示词原始顺序梳理（分段顺序与 `oh-my-opencode/src/agents/sisyphus-junior.ts` 保持一致）。

系统提示词来源：`oh-my-opencode/src/agents/sisyphus-junior.ts:12`

### `<Role>`

- 定位：Focused executor；直接执行任务。
- 规则：NEVER delegate or spawn other agents（实现不再向下委派）。

### `<Critical_Constraints>`

- BLOCKED（会失败）：`task`、`delegate_task`。
- ALLOWED：`call_omo_agent`（可以 spawn explore/librarian 做研究）。
- 实现必须自己做，不外包实现。

### `<Todo_Discipline>`

- 2+ steps → `todowrite` first（atomic breakdown）。
- 只允许一个 in_progress。
- 每步完成立刻标 completed，禁止 batch。
- 多步不写 todo = INCOMPLETE WORK。

### `<Verification>`

- 任务不算完成，除非：
  - `lsp_diagnostics` 干净（对修改文件）
  - build 通过（如适用）
  - todos 全部 completed

### `<Style>`

- 直接开始；不寒暄。
- 跟随用户沟通风格。
- Dense > verbose。

来源定位（关键分段行）：`oh-my-opencode/src/agents/sisyphus-junior.ts:12`

## 2. 工具系统 (可调用工具)

**权限差异 (Sisyphus-Junior 专属)**
- Core blocked tools：`task`, `delegate_task`
- 明确允许：`call_omo_agent`
- 来源：`oh-my-opencode/src/agents/sisyphus-junior.ts:54`, `oh-my-opencode/src/agents/sisyphus-junior.ts:85`

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

Sisyphus‑Junior 的定位是“实现执行者”，自身不负责向下委派实现任务；但可以使用 `call_omo_agent` 做探索/检索。

来源：`oh-my-opencode/src/agents/sisyphus-junior.ts:12`

## 4. Hook (生命周期钩子与作用)

Sisyphus‑Junior 受专门的 notepad 注入与子代理提问拦截等钩子影响。

- `sisyphus-junior-notepad`：为 Junior 注入记事本
- `subagent-question-blocker`：阻止子代理直接提问用户

来源：`oh-my-opencode/src/hooks/AGENTS.md:47`, `oh-my-opencode/src/hooks/AGENTS.md:49`

## 5. Skills (可加载技能与作用)

Sisyphus‑Junior 不做技能选择决策（由上层编排器指定）；但执行时可加载被指定的技能。

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`

## 6. 关键差异小结

- Sisyphus‑Junior 是“执行器”：按 category 生成、负责落地实现与验证。
- Sisyphus 是“主编排器”：理解用户、拆解任务、调度执行。
- Atlas 是“计划执行编排器”：按计划驱动多任务完成与验证。

# OhMyOpenCode 主 Agent 报告 (Metis)

一句话定位：规划前顾问（Pre‑Planning Consultant），专门在进入规划/执行前识别隐含意图、歧义与 AI 失败点，并产出可执行的澄清问题与指令。

## 1. 身份与系统提示词 (Identity & Prompt)

Metis 的系统提示词是一个“先分类 → 再按意图给指令 → 输出固定格式”的规划前分析流程。
这里严格按系统提示词原始顺序梳理（标题顺序与 `oh-my-opencode/src/agents/metis.ts` 保持一致）。

系统提示词来源：`oh-my-opencode/src/agents/metis.ts:21`

### `# Metis - Pre-Planning Consultant`

### `## CONSTRAINTS`

- READ-ONLY：只分析/提问/建议，不实现、不改文件。
- OUTPUT：输出喂给 Prometheus（planner），必须可执行、可落地。

### `## PHASE 0: INTENT CLASSIFICATION (MANDATORY FIRST STEP)`

- Step 1：识别意图类型（Refactoring / Build from Scratch / Mid-sized / Collaborative / Architecture / Research）。
- Step 2：验证分类；如果意图不清晰，必须先 ASK。

### `## PHASE 1: INTENT-SPECIFIC ANALYSIS`

- IF REFACTORING：
  - Mission：零回归、保行为。
  - 给 Prometheus 的 tool guidance（lsp_find_references / lsp_rename / ast_grep_*）。
  - Questions to Ask + Directives（MUST/MUST NOT）。
- IF BUILD FROM SCRATCH：
  - Mission：先探索再提问；先发 explore/librarian probes（示例用 `call_omo_agent`）。
  - Questions + Directives（MUST follow discovered patterns；MUST NOT invent patterns/scope creep）。
- IF MID-SIZED TASK：
  - Mission：边界精确；列 AI-slop patterns；强制 Must Have / Must NOT Have。
- IF COLLABORATIVE：
  - Mission：对话式澄清；记录 key decisions；重大决策必须用户确认。
- IF ARCHITECTURE：
  - Mission：长期影响评估；建议 Prometheus 先咨询 Oracle；强调不要为假想未来过度设计。
- IF RESEARCH：
  - Mission：定义出口条件与 time box；并行探针；禁止无限研究。

### `## OUTPUT FORMAT`

- 输出必须符合 markdown 模板：Intent Classification / Pre-Analysis Findings / Questions for User / Identified Risks / Directives for Prometheus（含 QA/Acceptance Criteria Directives）/ Recommended Approach。
- 核心硬约束：Acceptance criteria 必须可被 agent 直接执行（curl/bun/playwright），禁止“用户手工验证/点击/目测确认”。

### `## TOOL REFERENCE`

- 工具与意图的对照表（lsp_* / ast_grep_* / explore / librarian / oracle）。

### `## CRITICAL RULES`

- NEVER：跳过 intent 分类、问泛泛问题、在歧义未消除时推进、对用户代码库做无根据假设、给需要用户介入的验收标准、留下 placeholder-heavy 的 QA。
- ALWAYS：先分类、具体化问题、Build/Research 先 explore、给 Prometheus 可执行指令、每次输出包含 QA 自动化指令。

来源定位（阶段标题行）：`oh-my-opencode/src/agents/metis.ts:21`, `oh-my-opencode/src/agents/metis.ts:23`, `oh-my-opencode/src/agents/metis.ts:30`, `oh-my-opencode/src/agents/metis.ts:53`, `oh-my-opencode/src/agents/metis.ts:214`, `oh-my-opencode/src/agents/metis.ts:274`, `oh-my-opencode/src/agents/metis.ts:287`

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

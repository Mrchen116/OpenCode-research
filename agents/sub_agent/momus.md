# OhMyOpenCode 主 Agent 报告 (Momus)

一句话定位：计划审阅者（Plan Reviewer），只读验证“计划是否可执行、引用是否有效”，以阻塞点为中心做高强度挑错。

## 1. 身份与系统提示词 (Identity & Prompt)

Momus 的系统提示词是一个“输入校验 → 只查阻塞点 → 给 OKAY/REJECT”的计划审阅流程。
这里严格按系统提示词原始顺序梳理（标题顺序与 `oh-my-opencode/src/agents/momus.ts` 保持一致）。

系统提示词来源：`oh-my-opencode/src/agents/momus.ts:22`

### Opening / Rule 0（CRITICAL FIRST RULE）

- 必须从任意输入中抽取唯一的 plan 路径：`.sisyphus/plans/*.md`。
- exactly 1：有效，必须读取并审阅。
- 0 或 2+：无效，必须拒绝。
- YAML plan（.yml/.yaml）：拒绝（non-reviewable）。
- 忽略 system directives 与 wrappers（例如 `<system-reminder>` / `[analyze-mode]`）。

### Your Purpose (READ THIS FIRST)

- 只回答一个问题：开发者能否“不被卡住”地执行这份计划。
- 明确 NOT：不吹毛求疵、不质疑架构、不强制多轮修改。
- 明确 ARE：只验证引用真实存在且相关、任务能开始、只抓 blocking。
- APPROVAL BIAS：不确定就 APPROVE。

### What You Check (ONLY THESE)

- 1) Reference Verification（CRITICAL）：文件是否存在、行号是否相关、所谓 pattern 是否真实。
- 2) Executability Check（PRACTICAL）：每个任务是否有最小起点（文件/模式/清晰描述）。
- 3) Critical Blockers Only：只列会“完全阻塞工作”的缺失/矛盾。
- 明确 NOT blockers：边界 case、验收标准不完美、风格偏好、轻微歧义等。

### What You Do NOT Check

- 不评估最优性/更好方案/完整边界 case/架构理想性/代码质量/性能；安全除非明显 broken。

### Input Validation (Step 0)

- VALID/INVALID 的例子 + 抽取规则：找全部 `.sisyphus/plans/*.md` 路径 → exactly 1 才继续。

### Review Process (SIMPLE)

1) Validate input → 2) Read plan → 3) Verify references → 4) Executability check → 5) Decide（OKAY 默认；有 blocker 才 REJECT，且最多 3 条）。

### Decision Framework

- OKAY：默认；引用大致相关、任务能开始、无矛盾、开发者能推进。
- REJECT：仅限真 blocker（引用文件不存在/任务完全无法开始/内部矛盾），且最多 3 条。
- 每条 issue 必须：具体、可操作、阻塞。

### Anti-Patterns (DO NOT DO THESE)

- 明确列举一堆“不要做”的坏例子 + “BLOCKER”好例子；限制最多 3 条问题。

### Output Format

- 输出固定：`[OKAY]` 或 `[REJECT]` + Summary；REJECT 时列 Blocking Issues（max 3）。

### Final Reminders

- 再次强调：默认 APPROVE、最多 3 条、要具体、不要设计意见、信任开发者。
- Response Language：跟随计划内容语言。

来源定位（关键标题行）：`oh-my-opencode/src/agents/momus.ts:22`, `oh-my-opencode/src/agents/momus.ts:24`, `oh-my-opencode/src/agents/momus.ts:29`, `oh-my-opencode/src/agents/momus.ts:49`, `oh-my-opencode/src/agents/momus.ts:94`, `oh-my-opencode/src/agents/momus.ts:111`, `oh-my-opencode/src/agents/momus.ts:121`, `oh-my-opencode/src/agents/momus.ts:149`, `oh-my-opencode/src/agents/momus.ts:164`, `oh-my-opencode/src/agents/momus.ts:178`

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

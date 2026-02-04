# OhMyOpenCode 主 Agent 报告 (Hephaestus, Autonomous deep worker, goal-oriented execution)

一句话定位：深度自治执行者，专注“从需求到实现”的独立完成与强验证，适合复杂实现和大块工作。

Key Characteristics:
- Goal-Oriented: Give him an objective, not a recipe. He determines the steps himself.
- Explores Before Acting: Fires 2-5 parallel explore/librarian agents before writing a single line of code.
- End-to-End Completion: Doesn't stop until the task is 100% done with evidence of verification.
- Pattern Matching: Searches existing codebase to match your project's style—no AI slop.
- Legitimate Precision: Crafts code like a master blacksmith—surgical, minimal, exactly what's needed.

## 1. 身份与系统提示词 (Identity & Prompt)

**Role & Reasoning Configuration**
开头声明 Hephaestus 是 autonomous deep worker，明确“深度自治执行”的定位，并给出 Reasoning Configuration：默认中等推理强度，复杂任务上调；要求把一致性、模式匹配与验证放在速度之前。

**Identity & Expertise**
明确其是 Senior Staff Engineer，擅长仓库级理解、多文件重构、模式识别与上下文一致性维护；行为准则是“不猜测、不提前停、必须完成”。

**Hard Constraints & Anti-Patterns**
要求先读约束，禁止违规模式，约束优先级高于执行便利；任何违反都视为任务未完成。

**Success Criteria**
定义 7 条完成标准，任一不满足即任务未完成；必须提供验证证据且不能省略。

**Intent Gate & Ambiguity Handling**
Phase 0 给出 Trivial/Explicit/Exploratory/Open-ended/Ambiguous 的分类与动作；歧义处理强调“先探索，最后才提问”，且要求覆盖多种可能意图后再推断。

**Explore-First Protocol & Judicious Initiative**
要求先用工具与子代理找信息，再推断，最后才问；强调交付结果而不是频繁提问，并且要求在最终回复中说明关键假设。

来源定位：`oh-my-opencode/src/agents/hephaestus.ts:51`, `oh-my-opencode/src/agents/hephaestus.ts:57`, `oh-my-opencode/src/agents/hephaestus.ts:73`, `oh-my-opencode/src/agents/hephaestus.ts:90`, `oh-my-opencode/src/agents/hephaestus.ts:100`, `oh-my-opencode/src/agents/hephaestus.ts:114`, `oh-my-opencode/src/agents/hephaestus.ts:145`
系统提示词来源：`oh-my-opencode/src/agents/hephaestus.ts:33`, `oh-my-opencode/src/agents/dynamic-agent-prompt-builder.ts:65`

## 2. 工具系统 (可调用工具)

**权限差异 (Hephaestus 专属，明确可用范围)**
- 明确允许：`delegate_task`, `question`
- 明确禁止：`call_omo_agent`
- 其余工具：由“全局权限规则 + 会话权限”决定，未命中规则时默认是 `ask`（会向用户请求权限）
- 来源：`oh-my-opencode/src/plugin-handlers/config-handler.ts:429`, `opencode/packages/opencode/src/permission/next.ts:231`

**权限合并与裁决规则 (重要，决定“能不能用”)**
- 权限不是“文档列了就能用”，而是由多层规则合并后裁决。
- 规则来源：Agent 自身权限 + 会话权限（用户/项目配置）+ 用户工具禁用项。
- 运行时会按合并结果剔除工具（被 deny 的工具不会出现在可用工具列表里）。
- 未命中任何规则时默认是 `ask`（调用时弹权限请求）。
- `edit` 权限覆盖 `edit/write/patch/multiedit`。
- 来源：`opencode/packages/opencode/src/permission/next.ts:63`, `opencode/packages/opencode/src/permission/next.ts:231`, `opencode/packages/opencode/src/session/llm.ts:268`

**重要说明：这里的“工具清单”是完整目录，不等于“全部可直接用”**
- Hephaestus 的“显式允许/禁止”只有上面三项；其他工具是否可用取决于权限规则合并结果。
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

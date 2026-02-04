# OhMyOpenCode 主 Agent 报告 (Sisyphus) —— 不正确

在集成 `@oh-my-opencode` 插件后，OpenCode 的主 Agent 能力得到了显著增强，从单一的对话模型转变为一个多模型编排系统。

## 1. 主 Agent 身份与提示词 (Identity & Prompt)

主 Agent 被赋予了 **"Sisyphus"** 的身份，其提示词是**动态构建**的，根据当前可用的工具、Agent 和技能进行实时组装。

*   **身份设定**: 一位来自旧金山湾区的资深工程师（SF Bay Area engineer）。工作风格：**工作、委托、验证、交付**。拒绝 AI 废话（No AI slop）。
*   **核心特质**:
    *   **解析隐式需求**: Sisyphus 不仅仅是执行命令，它通过 `Phase 0 - Intent Gate` 协议，将用户请求分为 `Trivial` (琐碎)、`Explicit` (显式)、`Exploratory` (探索)、`Open-ended` (开放) 、 `Ambiguous `(模糊)、等 5 类。它通过 **Key Triggers** (如 "Look into" 隐式包含 "fix/PR") 识别深层意图。对于复杂任务，它会通过 **Metis (Analyzer)** 进行预计划分析，识别潜在的 "AI-Slop" (如过度工程、范围膨胀)。
    *   **适应代码库成熟度**: 在执行开放任务前，Sisyphus 会进入 `Phase 1 - Codebase Assessment`。它会扫描项目配置 (Linter/TSConfig)、采样 2-3 个相似文件，并将代码库分类为：
        *   **Disciplined** (严谨): 严格遵循既有模式。
        *   **Transitional** (过渡): 询问用户使用哪种模式。
        *   **Legacy/Chaotic** (混乱): 主动提议现代最佳实践，而不是盲目跟随。
        *   **Greenfield** (新项目): 应用全套现代工程标准。
    *   **优先委托**: 绝不单打独斗，优先将专业任务分发给子 Agent。
    *   **并行处理**: 利用并行执行（`run_in_background`）最大化处理能力。
*   **动态提示词构建**: 
    *   **按需组装**: 由 `oh-my-opencode/src/agents/sisyphus.ts` 中的 `buildDynamicSisyphusPrompt` 函数生成。
    *   **元数据驱动**: 提示词不是静态的，而是根据当前**已安装的工具** (LSP, AST-Grep 等)、**已注册的子 Agent** (从 `agentMetadata` 读取 Cost, Triggers) 和 **可用的 Skill** 动态生成的。
    *   **环境注入**: 自动注入当前时间、时区、本地化设置以及当前目录的 `AGENTS.md` 知识，确保 Agent 具备地理/时间感知能力。

## 2. 工具系统 (Tool System)

集成后，主 Agent 同时拥有 OpenCode 原生工具和 `@oh-my-opencode` 增强工具。

### 2.1 增强工具 (Enhanced Tools)
由 `@oh-my-opencode` 提供，通过插件接口注入。这些工具通常具有更高的超时控制、更好的并发支持或特定的高级功能。

| 类别 | 工具名称 | 功能描述 | 源代码参考 (oh-my-opencode) |
| :--- | :--- | :--- | :--- |
| **LSP (语言服务)** | `lsp_goto_definition`, `lsp_find_references`, `lsp_symbols`, `lsp_diagnostics`, `lsp_prepare_rename`, `lsp_rename` | 提供精准的定义跳转、引用查找、符号搜索、诊断和代码重构。 | `src/tools/lsp/tools.ts` |
| **高级搜索** | `ast_grep_search`, `ast_grep_replace` | 基于 AST 的多语言搜索/替换（支持 25 种语言）。 | `src/tools/ast-grep/tools.ts` |
| **增强搜索** | `grep`, `glob` | **替换原生**: 带 60s 超时控制和结果限制的高性能搜索。 | `src/tools/grep/tools.ts`, `src/tools/glob/tools.ts` |
| **会话管理** | `session_list`, `session_read`, `session_search`, `session_info` | 跨 Agent 管理和检索会话历史，支持 `resume` 参数实现 70%+ 的 Token 节省。 | `src/tools/session-manager/tools.ts` |
| **Agent 协作** | `delegate_task`, `call_omo_agent` | **核心**: 将任务分发给特定子 Agent 或根据 Category 动态分配。支持后台异步执行。 | `src/tools/delegate-task/tools.ts`, `src/tools/call-omo-agent/tools.ts` |
| **系统操作** | `interactive_bash`, `look_at` | 管理交互式 Tmux 会话，以及多模态文件（PDF、图像、UI）分析。 | `src/tools/interactive-bash/tools.ts`, `src/tools/look-at/tools.ts` |
| **后台任务** | `background_output`, `background_cancel` | 监控和取消正在运行的异步后台任务。 | `src/tools/background-task/tools.ts` |
| **技能扩展** | `skill`, `skill_mcp`, `slashcommand` | 执行复杂 Skill 流程，调用 MCP 服务或斜杠命令（如 `/refactor`, `/review`）。 | `src/tools/skill/tools.ts`, `src/tools/skill-mcp/tools.ts`, `src/tools/slashcommand/tools.ts` |

### 2.2 原生工具 (Native Opencode Tools)
这些是 OpenCode 核心提供的基础工具。

| 工具名称 | 功能描述 | 状态与关系 | 源代码参考 (opencode) |
| :--- | :--- | :--- | :--- |
| `bash` | 执行非交互式 Shell 命令。 | 保留。 | `packages/opencode/src/tool/bash.ts` |
| `read` | 读取文件内容。 | 保留。 | `packages/opencode/src/tool/read.ts` |
| `write`, `edit` | 写入或编辑文件。 | 保留。 | `packages/opencode/src/tool/write.ts`, `packages/opencode/src/tool/edit.ts` |
| `glob`, `grep` | 基本的文件搜索。 | **被增强工具覆盖**。Sisyphus 会优先使用 OMO 版本的 `grep`/`glob` 以获得更好的性能。 | `packages/opencode/src/tool/glob.ts` |
| `todowrite`, `todoread` | 管理任务列表。 | 保留，作为 OMO 编排系统的底层存储。 | `packages/opencode/src/tool/todo.ts` |
| `websearch`, `codesearch` | 外部搜索。 | 保留。 | `packages/opencode/src/tool/websearch.ts` |
| `webfetch` | 抓取网页内容。 | 保留。 | `packages/opencode/src/tool/webfetch.ts` |
| `task` | 标记任务状态。 | 保留。 | `packages/opencode/src/tool/task.ts` |
| `question` | 向用户提问。 | 保留，但 OMO 钩子会限制子 Agent 使用。 | `packages/opencode/src/tool/question.ts` |

**工具共存说明**: 当 `oh-my-opencode` 加载时，它注册的同名工具（如 `grep`, `glob`）会覆盖原生工具的调用。主 Agent `Sisyphus` 的提示词明确指导其使用增强版工具以实现更复杂的编排模式。

## 3. 子 Agent 体系 (Subagents)

系统由 **Native (原生)** 和 **Enhanced (增强)** 两类子 Agent 组成。

### 3.1 增强专家 Agent (OhMyOpenCode)
这些是专为复杂工程任务设计的专家，通过 `delegate_task` 调用。

| Agent | 身份/职责 | 模型参考 | 源代码 (oh-my-opencode) |
| :--- | :--- | :--- | :--- |
| **Sisyphus (Primary)** | 主编排器，负责顶层分类和策略。 | Claude Opus 4.5 | `src/agents/sisyphus.ts` |
| **Atlas** | 主编排钩子，持有全局 Todo 列表，管理任务全生命周期。 | Claude Opus 4.5 | `src/agents/atlas.ts` |
| **oracle (Advisor)** | 战略顾问，负责复杂逻辑、架构分析和调试（只读）。 | GPT-5.2 | `src/agents/oracle.ts` |
| **librarian (Researcher)** | 负责多仓库研究、GitHub 搜索和外部文档检索。 | Big Pickle | `src/agents/librarian.ts` |
| **explore (Code Finder)** | 快速代码库 Grep 搜索，用于上下文发现。 | GPT-5 Nano | `src/agents/explore.ts` |
| **multimodal-looker** | 分析图表、PDF、图片和 UI 截图。 | Gemini 3 Flash | `src/agents/multimodal-looker.ts` |
| **Prometheus (Planner)** | 负责制定宏大的战略计划（只读）。 | Claude Opus 4.5 | `src/agents/prometheus-prompt.ts` |
| **Metis (Analyzer)** | 预计划分析，识别任务中的潜在缺口。 | Claude Sonnet 4.5 | `src/agents/metis.ts` |
| **Momus (Reviewer)** | 计划审计员，寻找计划漏洞。 | Claude Sonnet 4.5 | `src/agents/momus.ts` |
| **Sisyphus-Junior** | 具体的任务执行者，由主 Agent 根据 Category 动态生成。 | Claude Sonnet 4.5 | `src/agents/sisyphus-junior.ts` |

### 3.2 原生 Agent (Native Opencode)
这些是 OpenCode 自带的 Agent，虽然功能较基础，但在特定模式下仍可调用。

| Agent | 职责描述 | 源代码 (opencode) |
| :--- | :--- | :--- |
| **build** | 默认 Agent，执行基础工具。 | `packages/opencode/src/agent/agent.ts` (第 74 行) |
| **plan** | 计划模式，禁止所有编辑工具。 | `packages/opencode/src/agent/agent.ts` (第 89 行) |
| **general** | 通用子 Agent，用于简单多步任务并行。 | `packages/opencode/src/agent/agent.ts` (第 112 行) |
| **explore (Native)** | 原生探索 Agent。 | `packages/opencode/src/agent/agent.ts` (第 127 行) |

**注**: 在 OMO 环境下，`Sisyphus` 作为 Primary Agent，其提示词会引导它使用 OMO 版本的 `explore` 和 `prometheus` 等专家 Agent，而不是原生的 `build` 或 `plan`。原生 Agent 被视为底层能力而非首选编排对象。

## 4. 周期钩子 (Lifecycle Hooks)

`@oh-my-opencode` 实现了 **32 个** 生命周期钩子，深度干预 Agent 的行为，确保稳健性。

| 钩子名称 | 功能描述 | 核心事件 | 源代码参考 (oh-my-opencode) |
| :--- | :--- | :--- | :--- |
| **atlas** | 核心编排钩子，维护全局状态和 Todo 列表。 | `PreToolUse`, `PostToolUse`, `event` | `src/hooks/atlas/` |
| **todo-continuation-enforcer** | 强制执行 Todo 列表，防止任务丢失或过早结束。 | `Stop`, `UserPromptSubmit` | `src/hooks/todo-continuation-enforcer.ts` |
| **session-recovery** | 自动处理网络超时或错误并恢复对话。 | `event` (session.error) | `src/hooks/session-recovery/` |
| **think-mode** | 动态调整模型的思考模式（Reasoning Effort）和 Token 预算。 | `chat.params` | `src/hooks/think-mode/` |
| **comment-checker** | 检查并防止 AI 在代码中生成过多废话注释。 | `PreToolUse`, `PostToolUse` | `src/hooks/comment-checker/` |
| **edit-error-recovery** | 自动修复 `edit` 失败的代码块（如 Linter 报错）。 | `PostToolUse` | `src/hooks/edit-error-recovery/` |
| **rules-injector** | 自动注入 `.cursorrules` 或 `.opencode/rules`。 | `PreToolUse`, `PostToolUse` | `src/hooks/rules-injector/` |
| **context-window-monitor** | 监控 Token 使用情况，提供头部剩余空间提醒。 | `PostToolUse`, `event` | `src/hooks/context-window-monitor.ts` |
| **directory-agents-injector** | 自动注入当前目录下的 `AGENTS.md` 知识库。 | `PreToolUse`, `PostToolUse` | `src/hooks/directory-agents-injector/` |
| **claude-code-hooks** | 提供与 Claude Code 兼容的钩子层。 | `PreToolUse`, `PostToolUse`, `event` | `src/hooks/claude-code-hooks/` |
| **keyword-detector** | 检测 `ultrawork`, `search`, `analyze` 等触发关键字。 | `UserPromptSubmit` | `src/hooks/keyword-detector/` |
| **auto-slash-command** | 自动检测并执行斜杠命令模式。 | `UserPromptSubmit` | `src/hooks/auto-slash-command/` |
| **ralph-loop** | 管理自引用开发循环（Ralph Loop）。 | `event`, `chat.message` | `src/hooks/ralph-loop/` |
| **thinking-block-validator** | 确保 `<thinking>` 块格式正确。 | `messages.transform` | `src/hooks/thinking-block-validator/` |
| **anthropic-context-limit** | 处理 Anthropic 上下文窗口限制后的自动恢复/压缩。 | `event` | `src/hooks/anthropic-.../` |
| **interactive-bash-session** | 管理交互式 Tmux 会话的状态同步。 | `PostToolUse`, `event` | `src/hooks/interactive-bash-.../` |
| **non-interactive-env** | 为非 TTY 环境自动调整工具输入。 | `PreToolUse` | `src/hooks/non-interactive-env/` |
| **prometheus-md-only** | 强制 Prometheus 规划器处于只读模式。 | `PreToolUse` | `src/hooks/prometheus-md-only/` |
| **sisyphus-junior-notepad** | 为 Sisyphus Junior 提供专用的记事本上下文。 | `PreToolUse` | `src/hooks/sisyphus-junior-notepad/` |
| **delegate-task-retry** | 自动重试失败的任务分发。 | `PostToolUse` | `src/hooks/delegate-task-retry/` |
| **agent-usage-reminder** | 为特定 Agent（如 Oracle）提供使用建议。 | `PostToolUse`, `event` | `src/hooks/agent-usage-reminder/` |
| **compaction-context-injector** | 在会话压缩（Compaction）时注入关键上下文。 | `onSummarize` | `src/hooks/compaction-.../` |
| **directory-readme-injector** | 自动注入当前目录下的 `README.md`。 | `PreToolUse`, `PostToolUse` | `src/hooks/directory-readme-injector/` |
| **empty-task-response-detector** | 检测并处理 Agent 的空响应。 | `PostToolUse` | `src/hooks/empty-task-response-detector.ts` |
| **background-notification** | 通过系统通知推送异步任务结果。 | `event` | `src/hooks/background-notification/` |
| **auto-update-checker** | 检查插件更新。 | `event` | `src/hooks/auto-update-checker/` |
| **category-skill-reminder** | 提醒 Agent 使用合适的 Category 技能。 | `PostToolUse`, `event` | `src/hooks/category-skill-reminder/` |
| **start-work** | 初始化 Sisyphus 的工作会话。 | `UserPromptSubmit` | `src/hooks/start-work/` |
| **task-resume-info** | 为取消的任务记录恢复信息。 | `PostToolUse` | `src/hooks/task-resume-info/` |
| **tool-output-truncator** | 自动截断过长的工具输出，防止上下文爆炸。 | `PostToolUse` | `src/hooks/tool-output-truncator.ts` |
| **question-label-truncator** | 自动截断提问标签。 | `PreToolUse` | `src/hooks/question-label-truncator/` |
| **subagent-question-blocker** | 阻止子 Agent 直接向用户提问，强制其报告给主 Agent。 | `PreToolUse` | `src/hooks/subagent-question-blocker/` |

## 5. 可用的技能 (Skills)

Skill 是复合任务流，通常由 `delegate_task` 加载：

| 技能名称 | 核心用途 | 源代码参考 (oh-my-opencode) |
| :--- | :--- | :--- |
| **`playwright`** | 浏览器自动化。UI 测试、截图任务必备。 | `src/features/builtin-skills/skills.ts` (第 4 行) |
| **`frontend-ui-ux`** | 前端专家。由 Vercel v0 级别的 Agent 驱动，将模糊构思转化为精美 UI。 | `src/features/builtin-skills/skills.ts` (第 315 行) |
| **`git-master`** | Git 专家。处理提交、变基、冲突解决。 | `src/features/builtin-skills/skills.ts` (第 393 行) |
| **`dev-browser`** | 开发者专用浏览器，优化了文档抓取。 | `src/features/builtin-skills/skills.ts` (第 1499 行) |
| **`agent-browser`** | 通用网页浏览技能。 | `src/features/builtin-skills/skills.ts` (第 18 行) |
# hook 使用地图（业务层，硬条件版）

说明：每个 hook 只写三件事。
- 触发事件：在哪个生命周期触发
- 判定条件：代码里明确的 if 条件
- 动作：命中后改了什么

## 提示词 / 上下文注入

### keyword-detector
高层作用：把用户意图中的“工作模式信号”转成可执行的前置指令（ultrawork/search/analyze）。
- 触发事件：`chat.message`
- 判定条件：
  1. 从 `output.parts` 提取出的文本命中 `ultrawork/search/analyze` 关键词正则
  2. 不是 system directive（`[SYSTEM DIRECTIVE: OH-MY-OPENCODE...`）
  3. 不是后台子会话（`!subagentSessions.has(sessionID)`）
  4. 非主会话时只保留 `ultrawork`（`search/analyze` 被过滤）
- 动作：把 mode 指令前置到原消息：`{mode}\n\n---\n\n{original}`
- 代码：`oh-my-opencode/src/hooks/keyword-detector/index.ts`

### rules-injector
高层作用：按文件上下文自动加载“该文件应遵守的规则”，减少风格和流程偏差。
- 触发事件：`tool.execute.after`
- 判定条件：
  1. `input.tool.toLowerCase()` 在 `read/write/edit/multiedit`
  2. 能从工具输出解析到目标文件路径
  3. 找到匹配规则文件（frontmatter 条件通过），且路径/内容未重复注入
- 动作：在 `output.output` 追加规则文本块（`[Rule: ...][Match: ...]`）
- 清理：`session.deleted` / `session.compacted` 清会话缓存
- 代码：`oh-my-opencode/src/hooks/rules-injector/index.ts`

### directory-agents-injector
高层作用：自动补齐子目录 `AGENTS.md` 的约束与约定，让 agent 按目录规范执行。
- 触发事件：`tool.execute.after`
- 判定条件：
  1. `input.tool.toLowerCase() === "read"`
  2. 从读取文件目录向上找到 `AGENTS.md`（到 workspace 根）
  3. 同目录尚未注入（session cache 去重）
- 动作：在 `output.output` 追加 `[Directory Context: ...]` + 文件内容
- 清理：`session.deleted` / `session.compacted`
- 代码：`oh-my-opencode/src/hooks/directory-agents-injector/index.ts`

### directory-readme-injector
高层作用：自动补齐目录级 README 背景，避免 agent 脱离模块语义做修改。
- 触发事件：`tool.execute.after`
- 判定条件：
  1. `input.tool.toLowerCase() === "read"`
  2. 从读取文件目录向上找到 `README.md`
  3. 同目录尚未注入（session cache 去重）
- 动作：在 `output.output` 追加 `[Project README: ...]` + 文件内容
- 清理：`session.deleted` / `session.compacted`
- 代码：`oh-my-opencode/src/hooks/directory-readme-injector/index.ts`

## 工具执行防护

### subagent-question-blocker
高层作用：禁止子 agent 直接打断用户，强制“先完成子任务，再回报主 agent”。
- 触发事件：`tool.execute.before`
- 判定条件：
  1. `input.tool` 是 `question` 或 `askuserquestion`
  2. `subagentSessions.has(input.sessionID) === true`
- 动作：`throw new Error(...)`，直接阻断本次提问工具调用
- 说明：这就是“子 agent 想问你”的判定，完全由工具名 + 会话类型决定
- 代码：`oh-my-opencode/src/hooks/subagent-question-blocker/index.ts`

### question-label-truncator
高层作用：防止提问选项文本过长导致交互界面/协议表现异常。
- 触发事件：`tool.execute.before`
- 判定条件：
  1. `input.tool` 是 `askuserquestion` 或 `ask_user_question`
  2. `args.questions[].options[].label` 存在
- 动作：`label.length > 30` 时改成 `label.slice(0, 27) + "..."`
- 代码：`oh-my-opencode/src/hooks/question-label-truncator/index.ts`

### comment-checker
高层作用：在写代码后自动审查注释质量，抑制“AI 噪音注释”污染代码。
- 触发事件：`tool.execute.before` + `tool.execute.after`
- 判定条件：
  1. before：工具是 `write/edit/multiedit`，先记录 filePath/content 到 `pendingCalls`
  2. after：按 callID 找回 pendingCall；工具输出不是明显失败（包含 `error:/failed/could not` 会跳过）
  3. comment-checker CLI 可用（存在可执行路径）
- 动作：运行 checker；若有问题，追加提示到 `output.output`
- 代码：`oh-my-opencode/src/hooks/comment-checker/index.ts`

### prometheus-md-only
高层作用：把规划 agent 锁在“计划文档工作区”，防止越权改业务代码。
- 触发事件：`tool.execute.before`
- 判定条件：当前 session 的 agent 被识别为 Prometheus/planner
- 动作：限制可用工具与可写路径（主要只允许规划相关 `.sisyphus` 范围）；不符合则阻断/提示
- 代码：`oh-my-opencode/src/hooks/prometheus-md-only/index.ts`

## 恢复与稳态

### session-recovery
高层作用：会话异常时自动修复消息结构并恢复流程，降低人工救火频率。
- 触发事件：`event(session.error)`（由 `src/index.ts` 调用 `handleSessionRecovery`）
- 判定条件：
  1. 出错消息是 assistant 消息，且有 `error`
  2. `detectErrorType(error)` 命中：`tool_result_missing` / `thinking_block_order` / `thinking_disabled_violation`
- 动作：按错误类型执行修复（补 tool_result、修 thinking 顺序、剥 thinking），可选自动 resume
- 代码：`oh-my-opencode/src/hooks/session-recovery/index.ts`

### delegate-task-retry
高层作用：在委派参数错误时给出可直接复制的纠错重试指导，避免卡死在报错回合。
- 触发事件：`tool.execute.after`
- 判定条件：
  1. `input.tool.toLowerCase() === "delegate_task"`
  2. `output.output` 含 `[ERROR]` 或 `Invalid arguments`
  3. 命中内置参数错误模式（如缺 `run_in_background`、category/subagent 冲突）
- 动作：在输出后追加“立即重试”的参数修复模板
- 代码：`oh-my-opencode/src/hooks/delegate-task-retry/index.ts`

### stop-continuation-guard
高层作用：提供“停止自动续跑”的会话级闸门，防止系统在你叫停后继续推进。
- 触发事件：`chat.message` + `event(session.deleted)` + 外部调用 `stop()`
- 判定条件：
  1. `stop(sessionID)` 后会把 session 标记进 `stoppedSessions`
  2. 新 `chat.message` 到来且该 session 已被标记
- 动作：
  1. `isStopped(sessionID)` 对外提供“是否停止自动继续”的硬开关
  2. 新用户消息/会话删除时清理 stop 状态
- 代码：`oh-my-opencode/src/hooks/stop-continuation-guard/index.ts`

## 上下文体积控制

### tool-output-truncator
高层作用：主动压缩高噪声工具输出，控制上下文成本并保持主任务可持续推进。
- 触发事件：`tool.execute.after`
- 判定条件：
  1. 若 `truncate_all_tool_outputs=false`，仅处理白名单工具：`grep/glob/lsp_diagnostics/ast_grep_search/interactive_bash/skill_mcp/webfetch`
  2. `output.output` 必须是字符串
- 动作（明确算法）：
  1. token 估算：`tokens = ceil(chars / 4)`
  2. 目标上限：`min(remainingContext * 0.5, targetMaxTokens)`
  3. `targetMaxTokens` 默认 50000，`webfetch` 专用 10000
  4. 截断时固定保留前 3 行（`preserveHeaderLines=3`）
  5. 后续内容按“逐行累加 token 不超限”保留
  6. 末尾追加 `[N more lines truncated ...]`
- 代码：`oh-my-opencode/src/hooks/tool-output-truncator.ts`, `oh-my-opencode/src/shared/dynamic-truncator.ts`

### context-window-monitor
高层作用：监控上下文消耗并在逼近阈值时提醒，防止中后期因窗口压力失真。
- 触发事件：`tool.execute.after`
- 判定条件：
  1. 该 session 尚未提醒过
  2. 最近 assistant 消息 provider 是 `anthropic`
  3. 使用率 `>= 70%`（按 `input + cache.read` 计算）
- 动作：在 `output.output` 末尾追加 context 状态提醒
- 清理：`session.deleted` 时清提醒状态
- 代码：`oh-my-opencode/src/hooks/context-window-monitor.ts`

### compaction-context-injector
高层作用：在压缩前强制摘要结构化，确保压缩后仍能无缝继续业务执行。
- 触发事件：`experimental.session.compacting`（压缩前）
- 判定条件：被注册即执行
- 动作：注入固定摘要模板，强制 compaction summary 包含“已完成/未完成/约束/验收状态”等段落
- 代码：`oh-my-opencode/src/hooks/compaction-context-injector/index.ts`

## 编排 / 任务系统

### atlas
高层作用：作为编排中枢，强制“先委派、后验证、再推进计划”的执行纪律。
- 触发事件：`tool.execute.before`
- 判定条件 A（写改拦截提醒）：
  1. `isCallerOrchestrator(sessionID) === true`
  2. `input.tool` 在 `Write/Edit`
  3. `filePath` 不在 `.sisyphus/`
- 动作 A：在 `output.message` 注入“必须 delegate，不应直接实现”强提醒

- 触发事件：`tool.execute.before`
- 判定条件 B（delegate_task 单任务约束）：
  1. `input.tool === "delegate_task"`
  2. `args.prompt` 存在且不含 system directive 前缀
- 动作 B：在 `args.prompt` 前插入 `SINGLE_TASK_DIRECTIVE`

- 触发事件：`tool.execute.after`
- 判定条件 C（delegate_task 后处理）：
  1. `input.tool === "delegate_task"`
  2. 输出不是 “Background task launched/continued”
- 动作 C：改写输出，附 git diff 统计 + 验证清单 + 后续步骤提醒

- 触发事件：`event(session.idle)`
- 判定条件 D（自动 continuation）：
  1. 会话是主会话 / 后台会话 / boulder 会话之一
  2. 最近不是 abort error
  3. 没有运行中的后台任务
  4. 存在 active boulder 且计划未完成
  5. 最近调用者是 orchestrator（Atlas）
  6. 不在 cooldown（5 秒）
- 动作 D：自动注入 boulder continuation prompt
- 代码：`oh-my-opencode/src/hooks/atlas/index.ts`

### todo-continuation-enforcer
高层作用：检测未完成 todo 并自动续跑，避免任务在“看似完成”时提前停止。
- 触发事件：`event(session.idle)`
- 判定条件：
  1. 会话是主会话或后台会话
  2. 不在 recovering
  3. 不在 abort 窗口（事件与 API 双重检测）
  4. 无运行中的后台任务
  5. todo 列表存在且有未完成项
  6. 当前 agent 不在 skip 列表（默认 `prometheus`, `compaction`）
  7. `isContinuationStopped(sessionID) !== true`
- 动作：先倒计时 2 秒 toast，再注入 continuation prompt 让 agent 继续完成 todos
- 取消条件：`message.updated` / `message.part.updated` / `tool.execute.before/after` / `session.error` 会取消倒计时
- 代码：`oh-my-opencode/src/hooks/todo-continuation-enforcer.ts`

### start-work
高层作用：把“开始干活”请求转成可执行工作态（选计划、建状态、切编排 agent）。
- 触发事件：`chat.message`
- 判定条件：消息文本包含 `<session-context>`（只处理真实 start-work 模板）
- 动作：
  1. `updateSessionAgent(sessionID, "atlas")`
  2. 读取/创建 boulder state（支持 `<user-request>` 指定计划名）
  3. 把选中的 plan、进度、下一步说明追加到当前消息
- 代码：`oh-my-opencode/src/hooks/start-work/index.ts`

### ralph-loop
高层作用：提供可控的迭代续跑循环，直到满足完成承诺或达到上限。
- 触发事件：`event(session.idle)`
- 判定条件：
  1. 有 active loop state，且 session 匹配
  2. 尚未检测到 `<promise>{completion_promise}</promise>`
  3. 当前迭代未超过 `max_iterations`
- 动作：
  1. `iteration + 1`
  2. 注入 continuation prompt 继续执行
  3. 若 `ultrawork=true`，continuation 前自动加 `ultrawork`
- 终止条件：检测到 promise / 达到最大迭代 / session 删除 / abort error
- 代码：`oh-my-opencode/src/hooks/ralph-loop/index.ts`

## 兼容层

### claude-code-hooks
高层作用：把 Claude Code 的 hook 生态接到当前流水线上，复用其拦截与注入能力。
- 触发事件：
  1. `chat.message`（UserPromptSubmit）
  2. `tool.execute.before`（PreToolUse）
  3. `tool.execute.after`（PostToolUse）
  4. `experimental.session.compacting`（PreCompact）
  5. `event`（Stop 等）
- 判定条件：对应阶段未被 `isHookDisabled(...)` 禁用
- 动作：执行 Claude Code 风格 hooks；可阻断、改参数、注入上下文
- 代码：`oh-my-opencode/src/hooks/claude-code-hooks/index.ts`

## 入口顺序（你排查行为时最有用）
- 统一编排：`oh-my-opencode/src/index.ts`
- 重点顺序：
  1. `chat.message`: `keyword-detector -> claude-code-hooks -> auto-slash-command -> start-work`
  2. `tool.execute.before`: `subagent-question-blocker -> question-label-truncator -> ... -> prometheus-md-only -> sisyphus-junior-notepad -> atlas`
  3. `tool.execute.after`: `... -> tool-output-truncator -> context-window-monitor -> ... -> delegate-task-retry -> atlas -> task-resume-info`

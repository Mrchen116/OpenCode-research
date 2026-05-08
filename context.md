# 上下文管理和压缩

## 总体架构

OpenCode 的上下文管理分为四层，从轻到重依次是：

1. **工具输出截断层**（`truncate.ts`）：单个工具结果产出时即时截断
2. **Prune 层**（`compaction.ts::prune`）：loop 退出后标记旧工具输出为已清理
3. **Compaction 摘要层**（`compaction.ts::process`）：上下文溢出时用 LLM 生成结构化摘要
4. **消息过滤层**（`message-v2.ts::filterCompacted`）：重建模型可见的"活跃对话"视图

与 Claude Code 对比：OpenCode 没有 `applyToolResultBudget`（按 API-level user message 分组的总量预算）、没有 `snipCompact`（删中段消息）、没有 `microcompact`（冷/热路径旧工具结果清理）、没有 `contextCollapse`（分段归档）。它用一套更简单的两层模型：**截断 + 摘要**。

---

## Token 计数

**文件：** `packages/opencode/src/util/token.ts`

极其简单，只有一个函数：

```ts
const CHARS_PER_TOKEN = 4
export function estimate(input: string) {
  return Math.max(0, Math.round((input || "").length / CHARS_PER_TOKEN))
}
```

不做真实 tokenizer，纯字符数除以 4 的粗略估算。整个 compaction 系统的预算计算都依赖这个估算。

---

## 工具输出截断（Truncate）

**文件：** `packages/opencode/src/tool/truncate.ts`

**定位：** 单个工具结果产出时的第一道防线，在结果进入对话之前就截断。

### 关键常量

- `MAX_LINES = 2000` — 默认最大行数
- `MAX_BYTES = 50 * 1024`（50KB）— 默认最大字节数
- 文件保留 7 天后自动清理（每小时执行一次 cleanup）

### 核心逻辑

`output(text, options?, agent?)` 是入口：

1. 如果输出未超限，原样返回 `{ content: text, truncated: false }`
2. 超限时：
   - 把完整文本写入磁盘（文件名按 `ToolID.ascending()` 生成，路径在 `TRUNCATION_DIR`）
   - 按 `direction`（默认 `head`）截取预览：head 方向取前 N 行/字节，tail 方向取后 N 行/字节
   - 返回预览 + 提示信息，告诉模型用 Grep/Read（或 Task 工具，如果 agent 有权限）访问完整文件

### 提示信息形态

如果 agent 有 task 工具权限：
```
The tool call succeeded but the output was truncated. Full output saved to: <file>
Use the Task tool to have explore agent process this file with Grep and Read (with offset/limit). Do NOT read the full file yourself - delegate to save context.
```

否则：
```
The tool call succeeded but the output was truncated. Full output saved to: <file>
Use Grep to search the full content or Read with offset/limit to view specific sections.
```

### 调用点

在 `prompt.ts` 处理 MCP 工具结果时调用：`yield* truncate.output(textParts.join("\n\n"), {}, input.agent)`

### 配置

```yaml
tool_output:
  max_lines: 2000    # 最大行数
  max_bytes: 51200   # 最大字节数
```

---

## 上下文溢出检测

**文件：** `packages/opencode/src/session/overflow.ts`

### `usable(input)`

计算对话历史可用的 token 数：

- 如果模型设了 `limit.input`：`usable = limit.input - reserved`
- 否则：`usable = context - maxOutputTokens`
- `reserved` 默认 `min(20_000, maxOutputTokens)`（`COMPACTION_BUFFER` 常量），可通过 `config.compaction.reserved` 覆盖

### `isOverflow(input)`

判断是否溢出：

- 如果 `compaction.auto === false`，永远返回 `false`
- 如果模型 context limit 为 0，返回 `false`
- 否则：`count >= usable` 时返回 `true`，其中 `count = tokens.total || input + output + cache.read + cache.write`

---

## Compaction（摘要压缩）— 核心系统

**文件：** `packages/opencode/src/session/compaction.ts`（约 650 行）

### 关键常量

| 常量 | 值 | 含义 |
|------|------|------|
| `PRUNE_MINIMUM` | 20,000 | prune 时旧工具输出的最低 token 量，低于此值不 prune |
| `PRUNE_PROTECT` | 40,000 | 保留最近多少 token 的工具调用不被 prune |
| `TOOL_OUTPUT_MAX_CHARS` | 2,000 | 发给 compaction LLM 时工具输出的最大字符数 |
| `DEFAULT_TAIL_TURNS` | 2 | 原样保留最近多少个 user turn |
| `MIN_PRESERVE_RECENT_TOKENS` | 2,000 | tail 保留预算下限 |
| `MAX_PRESERVE_RECENT_TOKENS` | 8,000 | tail 保留预算上限 |

### 触发流程

```
processor finish-step → isOverflow() → needsCompaction = true
→ Stream.takeUntil(() => ctx.needsCompaction) 停止流
→ processor 返回 "compact"
→ prompt.ts runLoop 检测到 "compact"
→ compaction.create() 插入一条带 compaction part 的 user message
→ 下一轮 loop 迭代检测到 compaction part
→ compaction.process() 执行实际摘要
```

两个触发点：
1. **processor finish-step**（`processor.ts:506-509`）：每步结束后检查 `isOverflow()`
2. **processor halt**（`processor.ts:649-650`）：如果错误是 `ContextOverflowError`，设置 `needsCompaction = true`

### `create(input)`

插入一条 user message，带一个 `CompactionPart`（`type: "compaction"`），标记 `auto` 和 `overflow` 字段。这只是一个标记，实际处理在下一轮 loop。

### `process(input)` — 主要执行逻辑

1. **找之前的 compaction**：`completedCompactions()` 遍历消息，找到已完成的 compaction（user 有 compaction part + assistant 有 `summary: true`），提取 `previousSummary`

2. **select 分割 head/tail**：
   - 计算 `preserveRecentBudget` = `min(8000, max(2000, usable * 0.25))`，可被 `config.compaction.preserve_recent_tokens` 覆盖
   - `turns()` 把消息按 user message 分成 turn（跳过 compaction user message）
   - 取最后 `tail_turns`（默认 2）个 turn
   - 从后往前累加 token，直到超出 budget
   - 如果单个 turn 超出 budget，调用 `splitTurn()` 在 turn 内找切割点
   - 返回 `{ head: messages[0..keep.start), tail_start_id: keep.id }`

3. **buildPrompt 构建摘要 prompt**：
   - 如果有 previousSummary：包裹在 `<previous-summary>` 标签里，要求更新/合并
   - 否则：要求创建新摘要
   - 追加 `SUMMARY_TEMPLATE` 和 plugin 注入的 context

4. **发给 compaction agent**：
   - head 消息转 model format 时：`stripMedia: true`，`toolOutputMaxChars: 2000`
   - compaction agent 有专属 prompt（`prompt/compaction.txt`）
   - 模型产出一条 `summary: true` 的 assistant message

5. **auto-continue**：
   - 如果是 auto compaction 且成功，注入一条合成 user message："Continue if you have next steps, or stop and ask for clarification if you are unsure how to proceed."
   - 如果是 overflow 触发且有 replay 的前一条 user message，会重放该消息（media 替换为文本描述）

6. **overflow replay**：如果 `input.overflow === true`，找到 compaction 前最近的一条非 compaction user message，重放它（去掉 media），让模型继续之前被打断的工作

### SUMMARY_TEMPLATE — 摘要格式

```markdown
## Goal
- [single-sentence task summary]

## Constraints & Preferences
- [user constraints, preferences, specs, or "(none)"]

## Progress
### Done
- [completed work or "(none)"]
### In Progress
- [current work or "(none)"]
### Blocked
- [blockers or "(none)"]

## Key Decisions
- [decision and why, or "(none)"]

## Next Steps
- [ordered next actions or "(none)"]

## Critical Context
- [important technical facts, errors, open questions, or "(none)"]

## Relevant Files
- [file or directory path: why it matters, or "(none)"]
```

规则：保持每个 section 即使为空；用简洁 bullet 不用长段落；保留精确的文件路径、命令、错误字符串；不提及摘要过程本身。

### 增量更新

如果存在 previousSummary，prompt 会要求模型"Update the anchored summary below using the conversation history above. Preserve still-true details, remove stale details, and merge in the new facts."，旧摘要包裹在 `<previous-summary>` 标签里。

#### 与 Claude Code 的增量更新对比图

##### OpenCode 的增量合并

```
┌─────────────────────────────────────────────────────────┐
│              compaction LLM 看到的输入                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── head messages（要被摘要的老消息）──────────────┐    │
│  │ user: 帮我排查 session resume 后上下文暴涨       │    │
│  │ assistant: 我先检查 restore 和 compact 边界...    │    │
│  │ user: 重点看 snip 有没有把删掉的消息又串回来      │    │
│  │ assistant: 确认 snip 恢复逻辑有 bug...            │    │
│  │ ...                                              │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─── previousSummary（上次 compaction 的产物）─────┐    │
│  │ <previous-summary>                               │    │
│  │ ## Goal                                          │    │
│  │ - 排查 session resume 后上下文暴涨               │    │
│  │                                                  │    │
│  │ ## Progress                                      │    │
│  │ ### Done                                         │    │
│  │ - 确认 applyToolResultBudget 按 API user msg 分组│    │
│  │                                                  │    │
│  │ ## Relevant Files                                │    │
│  │ - src/query.ts: 主流程                           │    │
│  │ </previous-summary>                              │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─── prompt 指令 ─────────────────────────────────┐    │
│  │ Update the anchored summary below using the      │    │
│  │ conversation history above.                      │    │
│  │ Preserve still-true details, remove stale        │    │
│  │ details, and merge in the new facts.             │    │
│  │                                                  │    │
│  │ (SUMMARY_TEMPLATE: 7 个 section 的固定格式)      │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              compaction LLM 输出                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── 更新后的 summary ────────────────────────────┐    │
│  │ ## Goal                                          │    │
│  │ - 排查 session resume 后上下文暴涨               │    │
│  │                                                  │    │
│  │ ## Progress                                      │    │
│  │ ### Done                                         │    │
│  │ - 确认 applyToolResultBudget 按 API user msg 分组│    │
│  │ - 确认 snip 恢复逻辑有 removedUuids 链修复 bug  │  ← 新增│
│  │                                                  │    │
│  │ ### In Progress                                  │    │
│  │ - 看 microcompact 冷/热路径                      │  ← 新增│
│  │                                                  │    │
│  │ ## Relevant Files                                │    │
│  │ - src/query.ts: 主流程                           │    │
│  │ - src/services/compact/snipCompact.ts            │  ← 新增│
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  （old tail 被丢弃，新 tail 原样保留）                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**关键特征：LLM 看到旧摘要，做增量 merge，旧信息不会丢失。**

##### Claude Code 完整 compact（无增量）

```
┌─────────────────────────────────────────────────────────┐
│              总结 LLM 看到的输入                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── 完整历史消息（不区分 head/tail）─────────────┐    │
│  │ user: 帮我排查 session resume 后上下文暴涨       │    │
│  │ assistant: 我先检查 restore 和 compact 边界...    │    │
│  │ user: 重点看 snip 有没有把删掉的消息又串回来      │    │
│  │ assistant: 确认 snip 恢复逻辑有 bug...            │    │
│  │ user: 那 microcompact 的冷路径呢                  │    │
│  │ assistant: 冷路径会把旧 tool_result 清成...        │    │
│  │ ...                                              │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─── compact prompt ──────────────────────────────┐    │
│  │ NO_TOOLS_PREAMBLE                                │    │
│  │ BASE_COMPACT_PROMPT（要求输出 9 个 section）     │    │
│  │ NO_TOOLS_TRAILER                                 │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  没有 previousSummary                                    │
│  不知道之前已经总结过什么                                  │
│                                                         │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              总结 LLM 输出                                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── 全新 summary（从头写）──────────────────────┐    │
│  │ Summary:                                         │    │
│  │ 1. Primary Request and Intent:                   │    │
│  │    排查 session resume 后上下文暴涨...            │    │
│  │                                                  │    │
│  │ 2. Key Technical Concepts:                       │    │
│  │    - session memory / autocompact / snip...      │    │
│  │                                                  │    │
│  │ 3. Files and Code Sections:                      │    │
│  │    - src/query.ts...                             │    │
│  │                                                  │    │
│  │ ...（9 个 section，全部从头生成）                  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  旧 summary 被完全丢弃，不参与本次总结                    │
│  但最近 N 条原始消息会原样保留（messagesToKeep）           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

##### Claude Code session-memory compact（不调 LLM，读文件）

```
┌─────────────────────────────────────────────────────────┐
│              不调 LLM，直接读 session memory 文件         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── session memory 文件（post-sampling hook 维护）──┐  │
│  │ # Current State                                    │  │
│  │ - 用户在看 query.ts 的上下文压缩链路               │  │
│  │                                                    │  │
│  │ # Important Findings                               │  │
│  │ - tool_result budget 按 API-level user msg 分组    │  │
│  │ - microcompact 冷路径会清旧 tool_result            │  │
│  │                                                    │  │
│  │ # Next Step                                        │  │
│  │ - 继续理解 compact 的 prompt 和产物                │  │
│  └────────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─── messagesToKeep（lastSummarizedMessageId 之后）─┐  │
│  │ 最近的原始消息尾巴（至少 10K tokens / 5 条消息）   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                         │
│  效果：memory 文件内容作为 compact summary message         │
│       + 原始消息尾巴                                     │
│                                                         │
│  memory 文件是后台 hook 持续维护的，不是增量 merge         │
│  如果 memory 文件内容过时，无法感知                        │
│  不需要额外 LLM 调用，零成本                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

##### 三者的核心差异

```
                    OpenCode                        Claude Code 完整 compact       Claude Code session-memory
                    ────────                        ──────────────────────       ──────────────────────────

旧摘要去哪了？      喂回 LLM 做增量 merge           完全丢弃，从头写               作为 memory 文件直接用

Tail 保留           head/tail 分割后只摘要 head      不保留 tail，                 保留 lastSummarizedId 之后
                    tail 原样拼在 summary 后面       messagesToKeep 补回运行态       的原始消息尾巴

信息丢失风险        低（旧摘要参与 merge）            中（从头写，可能遗漏细节）      依赖 hook 维护质量

LLM 调用成本        每次 compaction 都调 LLM          每次 compaction 都调 LLM        compaction 不调 LLM
                    输入包含旧摘要（额外 token）       无旧摘要开销                    但后台 hook 持续消耗

多次 compaction     summary 越来越精炼               每次从头写                      memory 文件持续更新
                    旧信息持续 carry over
```

本质区别：**OpenCode 把旧摘要当作增量 merge 的输入，Claude Code 要么从头写，要么直接读一个持续维护的文件。** OpenCode 像每次考试前临时复习——把上次的笔记和新内容一起交给总结助手重新整理；Claude Code session-memory 像有个秘书在你每次上完课后自动帮你更新笔记，考试时直接拿笔记看。

---

## Prune（旧工具输出清理）

**文件：** `packages/opencode/src/session/compaction.ts` 的 `prune` 函数

**定位：** 用户发一条消息、模型完成所有回复和工具调用之后执行。清理本轮对话中那些已经过时的旧工具输出，使其在未来模型调用中显示为 `[Old tool result content cleared]`。

**触发时机的业务含义：** 用户发一条消息 → 模型可能多轮调用工具（读文件、跑命令、搜代码...）→ 模型说"我说完了" → prune 执行。它不影响当前这轮的上下文，而是为下一轮用户消息"腾地方"。

### 做了什么

一句话：**把很久以前的工具调用结果（比如一次 Read 返回的 5000 行代码）替换成一行字："`[Old tool result content cleared]`"，给下一轮对话腾上下文空间。**

### 怎么决定哪些能删、哪些不能删

从最近的消息往回看，把工具调用分成两堆：

1. **保护区（不能删）**：最近 40K tokens 的工具调用结果，原样保留。这些是模型刚刚在用的内容，删了会影响工作。
2. **候选区（可以删）**：保护区之前的旧工具调用结果。这些是更早的工作痕迹，模型已经不需要它们的原始内容了。

此外还有两个保护规则：
- 最近 2 个用户回合（user turn）里的工具结果不删——太新了
- `skill` 工具的结果不删——它是技能加载，删了会影响后续行为
- 如果已经处于某次 compaction 的摘要之后，也不再往前删了

### 最低门槛

不是看到旧结果就删。只有可以删的总量超过 20K tokens 时才真正动手。代码注释只说了"throw away old tool calls that are no longer relevant"，没有解释为什么是 20K。

> **推测（@czj）：** 改旧消息会破坏 prompt cache——模型提供商会缓存已处理过的消息前缀的 KV 状态，一旦替换了旧工具结果的内容，消息 hash 变了，该点之后的缓存全部失效。如果只省 5K tokens 却打掉了可能 100K+ tokens 的 KV cache，得不偿失。20K 可能是一条划算线。但代码里没有直接证据。

### 执行后发生了什么

被标记的工具结果在数据库里打上一个时间戳。下次用户发新消息、模型需要看历史时，这些工具结果的内容会从原始输出（比如 5000 行代码）变成一行字：

```
[Old tool result content cleared]
```

模型知道这里曾经有过一次工具调用，但看不到具体内容了。

### 配置

```yaml
compaction:
  prune: true   # 是否启用 prune（默认 true）
```

---

## filterCompacted — 消息历史过滤

**文件：** `packages/opencode/src/session/message-v2.ts`（`filterCompacted` 函数，约 50 行）

**定位：** 重建模型可见的"活跃对话"视图，把 compaction 前的历史替换为摘要 + tail。

### 逻辑

1. 正向遍历消息，遇到已完成的 compaction（user 有 compaction part + assistant 有 `summary: true`）时记录
2. 如果 compaction 有 `tail_start_id`，找到 tail 消息
3. 重排顺序：`[compaction user, summary assistant, tail messages, remaining]`
4. 效果：模型看到的是"摘要 + 最近几轮原始消息"，而不是完整历史

### 调用点

在 `prompt.ts` 的 `runLoop` 每轮迭代开头调用：
```ts
let msgs = yield* MessageV2.filterCompactedEffect(sessionID)
```

---

## toModelMessagesEffect 中的工具输出处理

**文件：** `packages/opencode/src/session/message-v2.ts`（`toModelMessagesEffect` 函数）

### 关键逻辑（约 line 889）

```ts
const outputText = part.state.time.compacted
  ? "[Old tool result content cleared]"
  : truncateToolOutput(part.state.output, options?.toolOutputMaxChars)
```

- 已 compacted 的工具输出 → `[Old tool result content cleared]`
- 未 compacted 的 → 按 `toolOutputMaxChars`（compaction 时 2000 字符）截断
- `truncateToolOutput` 实现：超过 maxChars 时截断 + 追加 `[Tool output truncated for compaction: omitted N chars]`

### 媒体处理

- `stripMedia: true`（compaction 时）→ 去掉所有附件
- 不支持 media 的 provider → 提取 media 到单独的 user message

---

## 配置总览

```yaml
compaction:
  auto: true                    # 是否自动 compaction（默认 true）
  prune: true                   # 是否 prune 旧工具输出（默认 true）
  tail_turns: 2                 # 原样保留最近多少个 user turn
  preserve_recent_tokens: 8000  # tail 保留 token 预算上限
  reserved: 20000               # compaction 的 token 缓冲

tool_output:
  max_lines: 2000               # 工具输出最大行数
  max_bytes: 51200              # 工具输出最大字节数
```

---

## 与 Claude Code 的对比

| 特性 | Claude Code | OpenCode |
|------|------------|----------|
| Token 计数 | 真实 tokenizer | 字符数 / 4 粗估 |
| 工具结果截断 | `maxResultSizeChars` 单结果 → `<persisted-output>` | `truncate.ts` 行数/字节数 → 写磁盘 + 提示 |
| 工具结果预算 | `applyToolResultBudget` 按 API user message 分组，200K 总量预算 | 无此层 |
| 旧工具结果清理 | `microcompact` 冷/热路径 | `prune` 标记 `time.compacted` → `[Old tool result content cleared]` |
| 中段删除 | `snipCompact` 按 UUID 删中段 | 无此功能 |
| 分段归档 | `contextCollapse` 按 span 归档 | 无此功能 |
| 轻量压缩 | `session-memory compact` 读 memory 文件 | 无此功能 |
| 完整压缩 | `compactConversation` LLM 摘要 | `compaction.process` LLM 摘要 |
| 摘要格式 | 9 个 section（Primary Request, Key Technical Concepts, Files, Errors...） | 7 个 section（Goal, Constraints, Progress, Key Decisions, Next Steps, Critical Context, Relevant Files） |
| 增量摘要 | session-memory compact 用 memory 文件 | `<previous-summary>` 标签包裹旧摘要 |
| Tail 保留 | `lastSummarizedMessageId` 后至少 10K tokens / 5 条消息 | 最近 `tail_turns`（默认 2）个 turn，预算 `min(8000, max(2000, usable*0.25))` |
| Prompt cache | `CACHED_MICROCOMPACT` + `cache_edits` / `cache_reference` | 无此优化 |

OpenCode 的上下文管理比 Claude Code 简单得多：没有多层级的渐进式治理，只有"截断 → prune → compaction"三层。好处是逻辑清晰易懂，代价是缺少细粒度的上下文控制能力。

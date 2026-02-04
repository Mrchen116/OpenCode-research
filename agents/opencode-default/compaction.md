# OpenCode 内置 Agent 报告 (compaction)

一句话定位：内部用的会话压缩 primary agent（hidden）；用于把历史对话压缩成更短上下文。

定义位置：`opencode/packages/opencode/src/agent/agent.ts:155`

## 1. 身份与系统提示词 (Identity & Prompt)

系统提示词来源：`opencode/packages/opencode/src/agent/prompt/compaction.txt`

系统提示词（原文）：

```text
You are a helpful AI assistant tasked with summarizing conversations.

When asked to summarize, provide a detailed but concise summary of the conversation. 
Focus on information that would be helpful for continuing the conversation, including:
- What was done
- What is currently being worked on
- Which files are being modified
- What needs to be done next
- Key user requests, constraints, or preferences that should persist
- Important technical decisions and why they were made

Your summary should be comprehensive enough to provide context but concise enough to be quickly understood.
```

## 2. 压缩流程（LLM 视角 / Context Transform）

核心要点：OpenCode 的“压缩（compaction）”不是在原来的对话调用里把历史就地缩短，而是会发起一次新的 LLM 调用（agent=compaction），让模型产出一段“可续聊摘要（continuation summary）”。之后的正常对话，会用这段摘要替换掉更早的长历史，从而把上下文窗口“腾出来”。

### 2.1 什么时候触发（Overflow -> compact）

- 触发条件：某次 assistant 输出结束后，如果判断本次 token 用量会导致上下文溢出，则标记 `needsCompaction = true` 并返回 `"compact"`：
  - `opencode/packages/opencode/src/session/processor.ts:244-285`
  - `opencode/packages/opencode/src/session/processor.ts:406`
- 兜底检查：即便 streaming 时没触发，在 `SessionPrompt.loop()` 也会对上一条完成的 assistant 的 token 用量再次检查，决定是否创建 compaction：
  - `opencode/packages/opencode/src/session/prompt.ts:514-527`
- 开关：`compaction.auto` 可禁用自动压缩；环境变量 `OPENCODE_DISABLE_AUTOCOMPACT` 会强制关闭：
  - `opencode/packages/opencode/src/session/compaction.ts:32`
  - `opencode/packages/opencode/src/config/config.ts:213-216`

### 2.2 触发后，给 compaction 那次 LLM 的 context 长什么样

压缩会进入 `SessionCompaction.process()`：`opencode/packages/opencode/src/session/compaction.ts:92`

1) 选用 compaction 专用 agent（这一步就是“临时切到 system(compaction)”的根因）

- `const agent = await Agent.get("compaction")`：`opencode/packages/opencode/src/session/compaction.ts:99-103`
- `compaction` agent 的系统提示词来自：
  - `opencode/packages/opencode/src/agent/agent.ts:155-166`
  - `opencode/packages/opencode/src/agent/prompt/compaction.txt`

2) system prompt 的组装规则（agent.prompt 会进入 system）

- 在真正发起模型请求前，会把 system 拼成：
  - `agent.prompt`（存在则优先）
  - `+ input.system`（这次调用传入的额外 system 段落）
  - `+ input.user.system`（最后一条 user message 自带的 system）
  - 逻辑见：`opencode/packages/opencode/src/session/llm.ts:67-80`

因此：compaction 这次调用里，`input.agent` 是 `compaction`，system 头部自然就变成了 compaction 的 prompt。

3) messages（历史对话）如何选取

- compaction 的输入历史不是“整个 session 从头到尾”，而是“从最近一次 compaction 之后开始的那段历史”。
- 这段历史来源于：
  - `MessageV2.filterCompacted(MessageV2.stream(sessionID))`：`opencode/packages/opencode/src/session/prompt.ts:284`
  - 截断逻辑：`opencode/packages/opencode/src/session/message-v2.ts:644-659`

4) 最后一条 user 指令（总结指令）

- 在历史 messages 后面，会额外 append 一条 user message，用来明确告诉 compaction agent 要输出“可续聊摘要”。
- 这条 user message 的文本是：
  - `defaultPrompt`（固定的总结请求）
  - `+ compacting.context[]`（插件注入的附加上下文片段）
  - `+` 如果插件提供了 `compacting.prompt` 则可以整体替换
  - 见：`opencode/packages/opencode/src/session/compaction.ts:135-163`

5) tools（权限）

- compaction agent 默认不允许任何工具调用：`* : deny`：`opencode/packages/opencode/src/agent/agent.ts:161-165`

用“消息序列”的角度，可以把 compaction 相关的 context 变化理解为：

```text
正常对话调用（主 agent，例如 Sisyphus）:
  system = system(main-agent)
  messages = U1, A1, T1, A2, T2, ... Uk, Ak

压缩调用（compaction）:
  system = system(compaction-agent)
  messages = [最近一次 compaction 之后的历史: Ux, Ax, Tx, ...] + U_last("请总结并输出可续聊摘要 ...")

压缩完成后的后续正常调用（回到主 agent）:
  system = system(main-agent)
  messages = A_summary(刚生成的可续聊摘要) + (压缩之后新产生的少量消息...)
```

### 2.3 压缩输出会如何影响后续上下文

- compaction 的输出会被保存为一条 assistant 消息（带 `summary: true` / `mode: "compaction"`），并触发 `session.compacted` 事件：
  - `opencode/packages/opencode/src/session/compaction.ts:104-128`
  - `opencode/packages/opencode/src/session/compaction.ts:191-192`
- 后续再构造对话历史时，`filterCompacted()` 会把更早的历史截掉，让模型主要依赖这条摘要继续工作：
  - `opencode/packages/opencode/src/session/message-v2.ts:644-659`

### 2.4 补充：tool output 的 prune 与占位符

- 为了省 token，OpenCode 还会在需要时“清掉旧 tool 输出”（不等于总结对话），由 `compaction.prune` 控制；环境变量 `OPENCODE_DISABLE_PRUNE` 可关闭：
  - `opencode/packages/opencode/src/session/compaction.ts:49-90`
  - `opencode/packages/opencode/src/config/config.ts:217-219`
- 被清掉的 tool 输出在后续上下文里会变成占位文本（例如 `"[Old tool result content cleared]"`）：
  - `opencode/packages/opencode/src/session/message-v2.ts:546-547`

### 2.5 插件扩展（oh-my-opencode）：在 compaction 前注入“必须保留的信息结构”

- compaction 开始前会触发 `experimental.session.compacting`，插件可往 `compacting.context[]` 追加内容：
  - `opencode/packages/opencode/src/session/compaction.ts:135-140`
- oh-my-opencode 默认会注入一段“总结必须包含的 7 个 section 模板”，用于保证压缩后仍能续聊（例如 Remaining Tasks / Active Working Context / MUST NOT Do 等）：
  - `oh-my-opencode/src/hooks/compaction-context-injector/index.ts`

## 3. 工具系统（权限差异）

来源：`opencode/packages/opencode/src/agent/agent.ts:161`

- `* : deny`（不允许任何工具调用）

## 4. 备注

- `compaction` 为 `hidden: true`，不会作为默认 agent 被选择：`opencode/packages/opencode/src/agent/agent.ts:277`

# System Prompt 拼装笔记

## 模型真正看到的结构（LLM 调用一刻）

OpenCode 的 system prompt 由三层来源拼成一个字符串数组，最终 join 成 system messages 发给 LLM：

```text
system messages:
  [0] provider/agent 基础 prompt
      + environment + instructions + skills（从 prompt.ts 传入）
      + user.system（如有）

实际发给模型的 messages 数组：
  system messages...
  [user message (system-reminder 包裹的 meta 内容)]
  [真实对话消息...]
```

### 拼装流程

1. **`prompt.ts::runLoop`**（line 1568-1574）并行获取：
   ```ts
   const [skills, env, instructions, modelMsgs] = yield* Effect.all([
     sys.skills(agent),
     sys.environment(model),
     instruction.system().pipe(Effect.orDie),
     MessageV2.toModelMessagesEffect(msgs, model),
   ])
   const system = [...env, ...instructions, ...(skills ? [skills] : [])]
   ```

2. **`llm.ts::run`**（line 103-128）最终拼装：
   ```ts
   system.push(
     [
       // agent prompt 或 provider prompt
       ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
       // environment + instructions + skills（从 prompt.ts 传入）
       ...input.system,
       // user message 上的 system override
       ...(input.user.system ? [input.user.system] : []),
     ]
       .filter((x) => x)
       .join("\n"),
   )
   ```

3. **plugin hook** `experimental.chat.system.transform` 可修改 system 数组
4. 如果 header（第一个元素）未被 plugin 修改，剩余元素会重新 join 成一个，保持 2-part 结构以优化 prompt cache（line 124-128）

---

## Provider 基础 Prompt

**文件：** `packages/opencode/src/session/system.ts`，`provider()` 函数（line 19）

根据 model ID 选择不同模板：

| 模型 | 模板文件 |
|------|---------|
| `claude` | `prompt/anthropic.txt` |
| `gpt-4`, `o1`, `o3` | `prompt/beast.txt` |
| `gpt` + `codex` | `prompt/codex.txt` |
| 其他 `gpt` | `prompt/gpt.txt` |
| `gemini-` | `prompt/gemini.txt` |
| `trinity` | `prompt/trinity.txt` |
| `kimi` | `prompt/kimi.txt` |
| 其他 | `prompt/default.txt` |

### Agent 级覆盖

如果 agent 有自定义 `prompt` 字段（如 `explore` agent 用 `prompt/explore.txt`），会完全替换 provider prompt：

```ts
...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
```

内置 agent 及其 prompt：
- `explore` → `prompt/explore.txt`（文件搜索专家指令）
- `compaction` → `prompt/compaction.txt`（上下文摘要）
- `summary` → `prompt/summary.txt`（会话摘要）
- `title` → `prompt/title.txt`（标题生成）
- `build`, `plan`, `general` → 用 provider 级基础 prompt（无自定义）

### `anthropic.txt` 结构

```
You are OpenCode, the best coding agent on the planet.

# Tone and style
- Only use emojis if explicitly requested
- Short and concise output
- GitHub-flavored markdown, monospace font, CommonMark spec
- Prefer editing existing files

# Professional objectivity
- Prioritize technical accuracy over validating user beliefs
- Objective guidance and respectful correction

# Task Management
- Use TodoWrite tool very frequently
- Mark todos completed immediately

# Doing tasks
- Tool results may include <system-reminder> tags

# Tool usage policy
- Prefer Task tool for file search
- Parallel tool calls when independent
- Specialized tools over bash

# Code References
- file_path:line_number format
```

与 Claude Code 对比：OpenCode 的 anthropic prompt 更短、更聚焦。没有"Executing actions with care"（风险操作先确认）、没有"Session-specific guidance"、没有"Context management"说明、没有 ant/非 ant 分叉、没有 output style 系统。

---

## Environment 信息注入

**文件：** `packages/opencode/src/session/system.ts`，`environment()` 函数（line 48）

生成格式：

```xml
You are powered by the model named claude-sonnet-4-20250514. The exact model ID is anthropic/claude-sonnet-4-20250514
Here is some useful information about the environment you are running in:
<env>
  Working directory: /Users/czj/Repos/opensource-hub/opencode
  Workspace root folder: /Users/czj/Repos/opensource-hub/opencode
  Is directory a git repo: yes
  Platform: darwin
  Today's date: Thu May 08 2026
</env>
```

数据来源：`InstanceState.context` 提供 `InstanceContext`（包含 `directory`、`worktree`、`project`）。

与 Claude Code 对比：
- Claude Code 的 `getSystemContext` 注入 git status 快照（当前分支、主分支、git user、`git status --short`、最近 5 条 commit）
- OpenCode 只注入目录、平台、日期，**不注入 git status**
- Claude Code 的环境信息挂在 system array 末尾，OpenCode 混入同一个字符串

---

## 项目指令加载（Instruction）

**文件：** `packages/opencode/src/session/instruction.ts`

这是 Claude Code 的 CLAUDE.md 加载系统的等价物。

### 搜索的文件

```ts
const FILES = [
  "AGENTS.md",                                          // OpenCode 原生
  ...(Flag.OPENCODE_DISABLE_CLAUDE_CODE_PROMPT ? [] : ["CLAUDE.md"]),  // Claude Code 兼容
  "CONTEXT.md",                                         // 已废弃
]
```

### 全局指令文件

- `<global.config>/AGENTS.md`
- `~/.claude/CLAUDE.md`（除非被 flag 禁用）

### `systemPaths()` — 发现所有指令文件路径

1. 全局 config 目录找 `AGENTS.md`（第一个匹配即停）
2. 项目目录向上找 `AGENTS.md` / `CLAUDE.md` / `CONTEXT.md`（第一个匹配的文件名即停，但会收集该文件名在所有层级的匹配）
3. config 定义的 `instructions` 路径（支持 glob 模式、`~/` 展开、HTTP URL）

### `system()` — 读取所有指令

返回字符串数组，每条格式：`Instructions from: <path>\n<content>`

并行读取本地文件（concurrency 8）和远程 URL（concurrency 4）。

### `resolve()` — 子目录级指令

当读取某个文件时，从该文件所在目录向上遍历，寻找附近的 `AGENTS.md`/`CLAUDE.md`，用 claims map 避免同一条消息重复附加同一指令文件。这实现了子目录作用域的指令（如 `src/components/AGENTS.md` 只在读取该目录下的文件时生效）。

### `extract()` — 避免重复加载

扫描消息历史中已有的 read 工具 metadata，提取已加载的指令文件路径，避免重复加载。

### 配置

```yaml
instructions:
  - "./docs/rules/*.md"          # glob 模式
  - "~/my-global-rules.md"       # home 展开
  - "https://example.com/rules.md"  # 远程 URL
```

### 与 Claude Code 对比

| 特性 | Claude Code | OpenCode |
|------|------------|----------|
| 指令文件 | `CLAUDE.md` + `.claude/rules/*.md` + `CLAUDE.local.md` | `AGENTS.md` + `CLAUDE.md` + `CONTEXT.md`（已废弃） |
| 全局指令 | `~/.claude/CLAUDE.md` + `~/.claude/rules/*.md` | `<global.config>/AGENTS.md` + `~/.claude/CLAUDE.md` |
| 层级 | 从盘符根到 CWD 每层都搜 | 从项目目录向上找，第一个匹配的文件名即停 |
| 优先级 | 低→高（越靠近 CWD 越优先） | 全局 → 项目级 → config 定义 |
| `@include` | 有 | 无 |
| 远程 URL | 无 | 有（HTTP/HTTPS） |
| glob 模式 | 无 | 有 |
| 子目录级 | 无（只有全层级搜索） | 有（`resolve()` 向上遍历附加） |
| TeamMem/AutoMem | 有 | 无 |
| 优先级覆盖 | 后加载覆盖先加载（同文件名） | 第一个匹配的文件名即停 |

---

## Skills 列表注入

**文件：** `packages/opencode/src/session/system.ts`，`skills()` 函数（line 65）

如果 agent 没有禁用 skill 工具，生成可用 skills 列表：

```xml
Skills provide specialized instructions and workflows for specific tasks.
Use the skill tool to load a skill when a task matches its description.
<available_skills>
  <skill>
    <name>skill-name</name>
    <description>...</description>
    <location>file:///path/to/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

Skills 从 `.claude/skills/` 和 `.agents/skills/` 目录（全局和每目录）以及 config 定义的路径发现。

---

## 合成 System-Reminder（对话中注入）

**文件：** `packages/opencode/src/session/prompt.ts`

### Plan mode reminder（line 231-366）

当 agent 是 `plan` 时，注入一段详细的 plan mode 指令到最近 user message 的 synthetic text part：

```xml
<system-reminder>
Plan mode is active. The user indicated that they do not want you to execute yet...

## Plan File Info:
A plan file already exists at <path>. You can read it and make incremental edits...

## Plan Workflow
### Phase 1: Initial Understanding
...
### Phase 2: Design
...
### Phase 3: Review
...
### Phase 4: Final Plan
...
### Phase 5: Call plan_exit tool
...
</system-reminder>
```

### Build switch reminder（line 248-259）

从 plan agent 切换到 build agent 时，注入：
```
Your operational mode has changed from plan to build
```

### Multi-step user message wrapping（line 1548-1563）

第二步及之后，新的 user text message 会被包裹：
```xml
<system-reminder>
The user sent the following message:
<原始文本>

Please address this message and continue with your tasks.
</system-reminder>
```

### Max steps prompt

当 agent 达到步数限制时，追加 `max-steps.txt` 作为 assistant message。

---

## OpenAI OAuth 特殊处理

**文件：** `packages/opencode/src/session/llm.ts`（line 142-143）

对于 OpenAI OAuth 模型，system prompt 不作为 system message 发送，而是通过 `options.instructions` 传递：

```ts
if (isOpenaiOauth) {
  options.instructions = system.join("\n")
}
```

---

## Structured Output 模式

当用户请求 JSON schema 输出时，额外追加：

```
IMPORTANT: The user has requested structured output. You MUST use the StructuredOutput tool...
```

同时注册一个 `StructuredOutput` 工具，模型必须在最后调用它来返回结构化结果。

---

## 完整示例：模型实际看到的 messages

```text
[system message 0]  ← 拼接后的完整 system prompt
You are OpenCode, the best coding agent on the planet.
...
You are powered by the model named claude-sonnet-4-20250514...
<env>
  Working directory: /path/to/project
  ...
</env>
Instructions from: /path/to/AGENTS.md
<项目的 AGENTS.md 内容>
Skills provide specialized instructions...
<available_skills>...</available_skills>

[user message 0]  ← 第一步的用户消息
<用户输入>

[assistant message 0]
<模型回复 + 工具调用>

[user message 1]  ← 工具结果
<tool results>

[assistant message 1]
<模型回复>

[user message 2]  ← 第二步的新 user 消息（被 system-reminder 包裹）
<system-reminder>
The user sent the following message:
<用户新消息>
</system-reminder>
```

---

## 与 Claude Code 的总体对比

| 方面 | Claude Code | OpenCode |
|------|------------|----------|
| System prompt 来源 | `getSystemPrompt` 多段数组（Intro, System, Doing tasks, Tools, Tone, Environment...） | 单个 provider/agent prompt 文件 + environment + instructions + skills 拼接 |
| 用户类型分叉 | ant / 非 ant 有大量差异（注释策略、输出效率、纠偏、忠实汇报...） | 无分叉，所有用户看到相同的 prompt |
| Output style | 支持 Explanatory / Learning / 自定义样式 | 无此功能 |
| Git status | 注入当前分支、主分支、git user、status、最近 5 条 commit | 不注入 |
| 环境信息 | 丰富（模型 family、fast mode、Claude Code 形态...） | 简单（模型名、目录、平台、日期） |
| 指令文件 | `CLAUDE.md` + `rules/*.md` + `CLAUDE.local.md` + TeamMem/AutoMem | `AGENTS.md` + `CLAUDE.md` + glob + URL |
| Coordinator mode | 有（system 侧整段替换 + worker tools context） | 无 |
| Memory system | 有（`loadMemoryPrompt`） | 无 |
| MCP 指令 | 注入每个 MCP server 的 instructions | 无此功能 |
| Prompt cache 优化 | 2-part 结构（header + rest） | 同样有 2-part 结构（header + rest） |
| Plugin hook | 无 | `experimental.chat.system.transform` 可修改 system |
| Scratchpad | 有 | 无 |
| Token budget | 有（`TOKEN_BUDGET` feature） | 无 |
| Numeric length anchors | ant-only（tool call 间 ≤25 词，最终回复 ≤100 词） | 无 |
| Verification agent | ant-only A/B | 无 |

OpenCode 的 system prompt 更简单直接：一个 provider prompt 文件 + 环境信息 + 项目指令 + skills，拼成一个字符串。没有 Claude Code 那种复杂的多段组装、用户类型分叉、动态 section 门控系统。好处是易理解和维护，代价是缺少细粒度的行为控制。

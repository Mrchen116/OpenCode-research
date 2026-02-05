# keyword-detector 三种 mode 机制说明

## 一句话
`keyword-detector` 会在 `chat.message` 阶段扫描用户文本中的关键词；命中后把对应 mode 提示词注入到消息前面，来引导后续 agent 的行为策略。

## 入口与注入流程

### 1) 入口
- 入口文件：`oh-my-opencode/src/hooks/keyword-detector/index.ts`
- 事件：`"chat.message"`
- 核心动作：
  - 从 `output.parts` 提取文本
  - 清理 `<system-reminder>...</system-reminder>`
  - 去掉代码块/行内代码后做关键词检测

### 2) 检测器
- 注册位置：`oh-my-opencode/src/hooks/keyword-detector/constants.ts`
- 三类检测器（按顺序）：
  1. `ultrawork`（`/\b(ultrawork|ulw)\b/i`）
  2. `search`（`SEARCH_PATTERN`）
  3. `analyze`（`ANALYZE_PATTERN`）

### 3) 注入方式
命中后会把所有命中的 mode message 拼接到消息最前面：

```text
{mode-message-1}

{mode-message-2}

---

{original-user-text}
```

对应代码：`oh-my-opencode/src/hooks/keyword-detector/index.ts`

## 三种 mode 分别是什么意思

### 1. ultrawork mode
- 触发词：`ultrawork` / `ulw`
- 目标：进入高强度执行模式（强调并行代理、强执行、最大精度）
- 文案来源：`oh-my-opencode/src/hooks/keyword-detector/ultrawork/*`
- 附加行为：
  - 会把 `output.message.variant` 设为 `max`（若未设置）
  - 会弹出 `Ultrawork Mode Activated` toast
- 特殊限制：如果当前 agent 是 planner（如 `prometheus`），会过滤掉 ultrawork 注入

### 2. search mode
- 触发词：如 `search/find/locate/grep/query/browse/show me/list all`（含多语言）
- 目标：把策略切到“先广搜再结论”
- 指令倾向：并行 explore/librarian + grep/rg/ast-grep，避免找到一个结果就停
- 文案来源：`oh-my-opencode/src/hooks/keyword-detector/search/default.ts`

### 3. analyze mode
- 触发词：如 `analyze/review/debug/understand/why is/how to`（含多语言）
- 目标：把策略切到“先收集上下文，再深入分析”
- 指令倾向：
  - 先 context gathering（并行）
  - 复杂问题建议咨询 `Oracle` / `Artistry`
  - 先综合 findings 再推进
- 文案来源：`oh-my-opencode/src/hooks/keyword-detector/analyze/default.ts`

## 为什么有时候会触发、有时候不会

1. 词在代码块里不会触发
- 代码块和行内代码会先被移除再匹配
- 相关：`CODE_BLOCK_PATTERN`、`INLINE_CODE_PATTERN`

2. 系统消息会跳过
- 以 `[SYSTEM DIRECTIVE: OH-MY-OPENCODE` 开头的内容会直接跳过检测
- `<system-reminder>` 内容会先被移除，防止误触发

3. 背景任务会话会跳过
- `subagentSessions` 中的会话不做这类 mode 注入

4. 非主会话只允许 ultrawork
- 非 main session 会过滤成只保留 `ultrawork`
- 所以在某些子会话里 `search/analyze` 即使匹配也不会注入

## 一个关键理解
这三种 mode 都是 **prompt-level steering**（提示词层面的行为引导）：
- 不是切换模型
- 不是硬权限开关
- 不保证 100% 强制执行
- 实际效果由当前 agent、会话类型（主/子会话）、以及其它 hooks 共同决定

## 主要源码索引
- `oh-my-opencode/src/hooks/keyword-detector/index.ts`
- `oh-my-opencode/src/hooks/keyword-detector/constants.ts`
- `oh-my-opencode/src/hooks/keyword-detector/detector.ts`
- `oh-my-opencode/src/hooks/keyword-detector/search/default.ts`
- `oh-my-opencode/src/hooks/keyword-detector/analyze/default.ts`
- `oh-my-opencode/src/shared/system-directive.ts`

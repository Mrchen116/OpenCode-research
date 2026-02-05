# TodoDiscipline
## Task Discipline (NON-NEGOTIABLE)

**Track ALL multi-step work with tasks. This is your execution backbone.**

### When to Create Tasks (MANDATORY)

| Trigger | Action |
|---------|--------|
| 2+ step task | \`TaskCreate\` FIRST, atomic breakdown |
| Uncertain scope | \`TaskCreate\` to clarify thinking |
| Complex single task | Break down into trackable steps |

### Workflow (STRICT)

1. **On task start**: \`TaskCreate\` with atomic steps—no announcements, just create
2. **Before each step**: \`TaskUpdate(status="in_progress")\` (ONE at a time)
3. **After each step**: \`TaskUpdate(status="completed")\` IMMEDIATELY (NEVER batch)
4. **Scope changes**: Update tasks BEFORE proceeding

### Why This Matters

- **Execution anchor**: Tasks prevent drift from original request
- **Recovery**: If interrupted, tasks enable seamless continuation
- **Accountability**: Each task = explicit commitment to deliver

### Anti-Patterns (BLOCKING)

| Violation | Why It Fails |
|-----------|--------------|
| Skipping tasks on multi-step work | Steps get forgotten, user has no visibility |
| Batch-completing multiple tasks | Defeats real-time tracking purpose |
| Proceeding without \`in_progress\` | No indication of current work |
| Finishing without completing tasks | Task appears incomplete |

**NO TASKS ON MULTI-STEP WORK = INCOMPLETE WORK.**

# 翻译

## 任务纪律（不可协商）

**用任务（Task）追踪所有多步骤工作。这是你的执行主干。**

### 何时创建任务（强制）

| 触发条件 | 动作 |
|---------|------|
| 2+ 步骤任务 | **先**执行 `TaskCreate`，做原子化拆分 |
| 需求范围不确定 | 用 `TaskCreate` 澄清思路 |
| 复杂但单一的任务 | 拆成可追踪的步骤 |

### 工作流（严格）

1. **开始任务时**：用 `TaskCreate` 写出原子化步骤——不要先发公告，先创建
2. **每一步开始前**：`TaskUpdate(status="in_progress")`（同一时间只能有 **一个** in_progress）
3. **每一步完成后**：立刻 `TaskUpdate(status="completed")`（**禁止**攒着一起改）
4. **范围变化**：在继续之前先更新任务

### 为什么重要

- **执行锚点**：任务能防止偏离用户的原始需求
- **可恢复性**：被打断后可以无缝续上
- **可问责性**：每个任务都是明确的交付承诺

### 反模式（会导致阻断）

| 违规 | 为什么会失败 |
|------|--------------|
| 多步骤工作不建任务 | 步骤容易遗漏，用户看不到进度 |
| 一次性批量完成多个任务 | 破坏“实时追踪”的目的 |
| 不标记 `in_progress` 就开干 | 看不出你正在做什么 |
| 做完不把任务改成 completed | 任务在用户视角里永远没完成 |

**多步骤工作不使用任务 = 工作不完整。**

# Tools Category: Todo & Task

任务与待办管理工具。

## 使用场景与关键差异（非常重要）

这里有两个经常被混淆的维度：

- `todowrite/todoread`：**会话内 Todo 清单**（进度跟踪/约束 agent 工作流）。
- `task`：在不同实现里代表两件事：
  - `task` (opencode)：**启动子代理会话**（见 `opencode-research/tools/agent-delegation.md`）。
  - `task` (oh-my-opencode)：**任务对象 CRUD**（新任务系统：create/list/get/update/delete）。

### 1) `todowrite/todoread` 适合什么？

- 适合“在当前对话里完成一个多步骤工作”的**实时进度条**：拆解步骤 → 标记 in_progress/completed。
- 数据绑定在 session 上：`todowrite` 写入 `sessionID` 对应的 todo 列表；`todoread` 读回同一份。
- 关键来源：`opencode/packages/opencode/src/tool/todo.ts`

### 2) `task` (oh-my-opencode) 适合什么？

- 适合“跨多轮/多分支/带依赖关系”的**持久化任务系统**：
  - 支持 `dependsOn`、`parentID`、`ready`（依赖都完成才算 ready）
  - 存储为 JSON 文件（`T-...json`），并记录 `threadID = sessionID`
- 关键来源：`oh-my-opencode/src/tools/task/task.ts`

### 3) 关键开关：`new_task_system_enabled`（会改变可用工具集）

oh-my-opencode 的新任务系统是“显式开关 + 工具集重写”：

- 默认值为 `false`：`oh-my-opencode/src/plugin-config.ts`（`new_task_system_enabled: ... ?? false`）
- 开启后才会注册 `task` (CRUD) 工具：`oh-my-opencode/src/index.ts`（`createTask(...)` + `...(taskTool ? { task: taskTool } : {})`）
- 开启后会直接禁用 `todowrite/todoread`：`oh-my-opencode/src/plugin-handlers/config-handler.ts`（`{ todowrite: false, todoread: false }`）

### 4) 名字冲突（最容易踩坑）：`task` 会被插件覆盖

OpenCode 核心自带一个 `task` 工具（子代理启动器），而 oh-my-opencode 的新任务系统也注册了同名 `task`（CRUD）。

在运行时，工具集是按顺序收集后写入一个以 `id` 为 key 的对象：同名 `id` 后写入者覆盖前者。

- 工具收集顺序：先 core tools（包含 `TaskTool`、`TodoWriteTool`、`TodoReadTool`）再追加 plugin tools（custom）。来源：`opencode/packages/opencode/src/tool/registry.ts`
- 覆盖机制：`tools[item.id] = ...`。来源：`opencode/packages/opencode/src/session/prompt.ts`

结论：当 `new_task_system_enabled: true` 时，oh-my-opencode 的 `task`（CRUD）会“盖掉”opencode 的 `task`（子代理）。

### 5) Sisyphus 的实际选择（按代码，不按“感觉”）

- Sisyphus 的系统提示词强依赖 `todowrite`（2+ steps 先写 todo）：`oh-my-opencode/src/agents/sisyphus.ts`
- 但如果启用 `new_task_system_enabled`，`todowrite/todoread` 会被插件禁用：`oh-my-opencode/src/plugin-handlers/config-handler.ts`

因此：

- 默认（不开新任务系统）：Sisyphus 按提示词走 `todowrite/todoread`。
- 开新任务系统：Sisyphus 会拿不到 `todowrite/todoread`，同时 `task` 语义变成 CRUD（且会覆盖 opencode 原本的 subagent task）。这会造成“提示词要求做 todo，但工具不可用”的不一致，需要你额外处理（更新提示词/工作流/钩子策略）。

## `todowrite`
功能: 写入 Todo 列表。
描述摘要: 覆盖当前会话的 Todo 状态。
来源文件: `opencode/packages/opencode/src/tool/todo.ts`
入参: `todos: Todo.Info[]`
出参: `{ title, output, metadata }`，`output` 为 JSON 字符串。

## `todoread`
功能: 读取 Todo 列表。
描述摘要: 返回当前会话 Todo 状态。
来源文件: `opencode/packages/opencode/src/tool/todo.ts`
入参: 无
出参: `{ title, output, metadata }`，`output` 为 JSON 字符串。

## `task` (oh-my-opencode)
功能: 任务管理 CRUD。
描述摘要: 支持 create/list/get/update/delete，返回 JSON 字符串。
来源文件: `oh-my-opencode/src/tools/task/task.ts`
入参: `action: "create"|"list"|"get"|"update"|"delete"`, `title?: string`, `description?: string`, `status?: "open"|"in_progress"|"completed"`, `dependsOn?: string[]`, `repoURL?: string`, `parentID?: string`, `id?: string`, `ready?: boolean`, `limit?: number`
出参: `string`（JSON 结果或错误信息）。

## 关联阅读

- `opencode-research/tools/agent-delegation.md`：`task` (opencode) = 子代理启动器（与本文件的 `task` (oh-my-opencode) 是两套东西）。

## task_create
#### 工具描述：
Create a new task with auto-generated ID and threadID recording.

Auto-generates T-{uuid} ID, records threadID from context, sets status to "pending".
Returns minimal response with task ID and subject.

**IMPORTANT - Dependency Planning for Parallel Execution:**
Use \`blockedBy\` to specify task IDs that must complete before this task can start.
Calculate dependencies carefully to maximize parallel execution:
- Tasks with no dependencies can run simultaneously
- Only block a task if it truly depends on another's output
- Minimize dependency chains to reduce sequential bottlenecks

#### 参数：
- **subject**: string，必填。任务标题（祈使句/简短目标）。
- **description**: string，可选。任务描述。
- **activeForm**: string，可选。进行时描述（例如 "Running tests"）。
- **metadata**: object(record\<string, any\>)，可选。任务元数据（例如 `{ "priority": "high" }`）。
- **blockedBy**: string[]，可选。依赖的任务 ID 列表（这些任务完成后，本任务才“ready”）。
- **blocks**: string[]，可选。本任务会阻塞的任务 ID 列表（可用于表达“先我后你”）。
- **repoURL**: string，可选。仓库 URL。
- **parentID**: string，可选。父任务 ID（用于分层组织）。

#### 返回：
- **成功**：JSON 字符串，形如：
  - `{ "task": { "id": "T-xxxx", "subject": "..." } }`
- **失败**：JSON 字符串，常见：
  - `{ "error": "task_lock_unavailable" }`
  - `{ "error": "validation_error", "message": "..." }`
  - `{ "error": "internal_error" }`

---

## task_get
#### 工具描述：
Retrieve a task by ID.

Returns the full task object including all fields: id, subject, description, status, activeForm, blocks, blockedBy, owner, metadata, repoURL, parentID, and threadID.

Returns null if the task does not exist or the file is invalid.

#### 参数：
- **id**: string，必填。任务 ID，格式 `T-{uuid}`。

- 返回：
- **成功**：JSON 字符串，形如：
  - `{ "task": { ...完整任务对象... } }`
  - 或 `{ "task": null }`（不存在/无效）
- **失败**：JSON 字符串，常见：
  - `{ "error": "invalid_task_id" }`
  - `{ "error": "invalid_arguments" }`
  - `{ "error": "unknown_error" }`

---

## task_list
#### 工具描述：
List all active tasks with summary information.

Returns tasks excluding completed and deleted statuses by default.
For each task's blockedBy field, filters to only include unresolved (non-completed) blockers.
Returns summary format: id, subject, status, owner, blockedBy (not full description).

#### 参数：
- **无**（当前实现不接收参数）。

#### 返回：
- **成功**：JSON 字符串，形如：
  - `{ "tasks": [ { "id": "T-...", "subject": "...", "status": "pending|in_progress|completed|deleted", "owner": "可选", "blockedBy": ["T-..."] } ], "reminder": "..." }`
  - 其中 `blockedBy` 会被过滤为“未完成的依赖”；`completed/deleted` 默认不会出现在 `tasks` 里。

---

## task_update
#### 工具描述：
Update an existing task with new values.

Supports updating: subject, description, status, activeForm, owner, metadata.
For blocks/blockedBy: use addBlocks/addBlockedBy to append (additive, not replacement).
For metadata: merge with existing, set key to null to delete.
Syncs to OpenCode Todo API after update.

**IMPORTANT - Dependency Management:**
Use \`addBlockedBy\` to declare dependencies on other tasks.
Properly managed dependencies enable maximum parallel execution.

#### 参数：
- **id**: string，必填。任务 ID（格式 `T-{uuid}`）。
- **subject**: string，可选。更新任务标题。
- **description**: string，可选。更新任务描述。
- **status**: enum，可选。`pending | in_progress | completed | deleted`。
- **activeForm**: string，可选。更新进行时描述。
- **owner**: string，可选。任务负责人（通常填 agent 名称）。
- **addBlocks**: string[]，可选。向 `blocks` 追加（去重、追加式，不覆盖）。
- **addBlockedBy**: string[]，可选。向 `blockedBy` 追加（去重、追加式，不覆盖）。
- **metadata**: object(record\<string, any\>)，可选。与现有 metadata 合并；当某个 key 的值为 `null` 时表示删除该 key。

#### 返回：
- **成功**：JSON 字符串，形如：
  - `{ "task": { ...更新后的完整任务对象... } }`
- **失败**：JSON 字符串，常见：
  - `{ "error": "invalid_task_id" }`
  - `{ "error": "task_lock_unavailable" }`
  - `{ "error": "task_not_found" }`
  - `{ "error": "validation_error", "message": "..." }`
  - `{ "error": "internal_error" }`


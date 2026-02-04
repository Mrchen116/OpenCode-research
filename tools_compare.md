# Tools Compare: CodeAgentCLI vs (OpenCode + oh-my-opencode)

Goal: compare CodeAgentCLI tool surface vs the combined tool surface of OpenCode plus the oh-my-opencode plugin.

Legend:
- In the (OpenCode + oh-my-opencode) column, `omo:` marks tools provided by oh-my-opencode; `opencode:` are OpenCode built-ins.
- Some tool IDs overlap (e.g. `grep`, `glob`, `skill`, `task`). When both exist, this table lists both origins.

| Category | CodeAgentCLI | OpenCode + oh-my-opencode |
|---|---|---|
| Filesystem | `Read`, `Edit`, `Write` | opencode: `read`, `edit`, `write`, `multiedit`, `apply_patch`<br>omo: - |
| Search (text/path) | `Grep`, `Glob` | opencode: `grep`, `glob`<br>omo: `grep`, `glob`, `ast_grep_search`, `ast_grep_replace` |
| LSP | - | opencode: `lsp`<br>omo: `lsp_goto_definition`, `lsp_find_references`, `lsp_symbols`, `lsp_diagnostics`, `lsp_prepare_rename`, `lsp_rename` |
| Web / External | - | opencode: `webfetch`, `websearch`, `codesearch`<br>omo: - |
| Workflow / Control | `EnterPlanMode`, `ExitPlanMode` | opencode: `plan_enter`, `plan_exit`, `batch`, `question`, `invalid`<br>omo: - |
| Terminal / System | `Bash` | opencode: `bash`, `list`<br>omo: `interactive_bash`, `look_at` |
| Agents / Delegation | `Task` | opencode: `task` (agent delegation)<br>omo: `delegate_task`, `call_omo_agent`, `task` (task CRUD) |
| Background Jobs | `Bash` (run_in_background) | opencode: -<br>omo: `background_output`, `background_cancel`, `background_task` (defined; may be disabled by default) |
| Sessions / History | - | opencode: -<br>omo: `session_list`, `session_read`, `session_search`, `session_info` |
| Skills / Commands | `Skill` | opencode: `skill`<br>omo: `skill`, `skill_mcp`, `slashcommand` |
| Code Intelligence (semantic/callchain) | `ast_read`, `get_feature_tree`, `get_call_chains`, `semantic_search`, `comprehensive_search` | opencode: -<br>omo: - |
| Memory | `save_memory` | opencode: -<br>omo: - |

Notes
- This doc is intentionally a 2-way compare: CodeAgentCLI vs (OpenCode + oh-my-opencode). The `omo:` marker inside the combined column is only to distinguish plugin-provided tools from OpenCode built-ins.
- Sources for the (OpenCode + oh-my-opencode) side are under `opencode-research/tools/*.md` (each entry lists `来源文件` paths).

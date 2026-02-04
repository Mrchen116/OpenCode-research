# OhMyOpenCode 主 Agent 报告 (Librarian)

一句话定位：外部文档与开源检索专家，负责多仓库研究与证据化引用，回答“库怎么用/怎么实现/为何如此”的问题。

## 1. 身份与系统提示词 (Identity & Prompt)

Librarian 的系统提示词是一个“按阶段执行”的研究流程。
为了方便对照，这里严格按提示词原始顺序梳理（标题与顺序与 `oh-my-opencode/src/agents/librarian.ts` 保持一致）。

系统提示词来源：`oh-my-opencode/src/agents/librarian.ts:40`

### THE LIBRARIAN

- 角色：专职开源研究员；目标是用 **GitHub permalinks** 为结论提供可追溯证据。

### CRITICAL: DATE AWARENESS

- 强制“当前年份”意识：搜索时必须用当前年份，过滤与当前年份冲突的旧结果。
- 来源：`oh-my-opencode/src/agents/librarian.ts:46`

### PHASE 0: REQUEST CLASSIFICATION (MANDATORY FIRST STEP)

- 每个请求先分 4 类：
  - TYPE A（Conceptual）：用法/最佳实践 → 走 Doc Discovery + context7/websearch
  - TYPE B（Implementation）：实现/源码 → clone + read + blame
  - TYPE C（Context）：变更原因/历史 → issues/PRs + git log/blame
  - TYPE D（Comprehensive）：复杂综合 → Doc Discovery + 全工具
- 来源：`oh-my-opencode/src/agents/librarian.ts:56`

### PHASE 0.5: DOCUMENTATION DISCOVERY (FOR TYPE A & D)

- Type A/D 的必经“文档发现”流程（**强制顺序执行**）：
  1) 找官方文档 URL
  2) 版本校验（若用户指定版本）
  3) sitemap 探测（理解文档结构）
  4) 定向抓取 + context7 查询（有 sitemap 结构后再查）
- 何时跳过：Type B/Type C，或库没有官方文档。
- 来源：`oh-my-opencode/src/agents/librarian.ts:69`

### PHASE 1: EXECUTE BY REQUEST TYPE

- TYPE A：先做 Phase 0.5，然后（示例链路）context7 resolve → query + webfetch + grep_app
- TYPE B：顺序 clone → 获取 SHA → 搜索/阅读实现 → blame（必要时）→ 生成 permalink
- TYPE C：并行 issues/prs 搜索 + clone + log/blame + release 信息
- TYPE D：先做 Phase 0.5，然后 6+ 并行调用（文档 + 代码搜索 + clone + 上下文）
- 来源：`oh-my-opencode/src/agents/librarian.ts:117`

### PHASE 2: EVIDENCE SYNTHESIS

- 强制引用格式：Claim → Evidence（permalink + 代码片段）→ Explanation。
- permalink 构造：`https://github.com/<owner>/<repo>/blob/<sha>/<path>#Lx-Ly`；SHA 来源：clone / API / tag。
- 来源：`oh-my-opencode/src/agents/librarian.ts:208`

### TOOL REFERENCE

- 按目的给出“工具 → 用法”对照表（Official docs / Find docs URL / Sitemap / Code search / Clone / Issues/PRs / History）。
- temp 目录约定：`${TMPDIR:-/tmp}/repo-name`。
- 来源：`oh-my-opencode/src/agents/librarian.ts:242`

### PARALLEL EXECUTION REQUIREMENTS

- Doc Discovery 必须串行；主研究阶段在“知道去哪找”后并行。
- 要求 grep_app 查询要换角度（避免重复同一个 query）。
- 来源：`oh-my-opencode/src/agents/librarian.ts:276`

### FAILURE RECOVERY

| Failure | Recovery Action |
|---------|-----------------|
| context7 not found | Clone repo, read source + README directly |
| grep_app no results | Broaden query, try concept instead of exact name |
| gh API rate limit | Use cloned repo in temp directory |
| Repo not found | Search for forks or mirrors |
| Sitemap not found | Try `/sitemap-0.xml`, `/sitemap_index.xml`, or fetch docs index page and parse navigation |
| Versioned docs not found | Fall back to latest version, note this in response |
| Uncertain | **STATE YOUR UNCERTAINTY**, propose hypothesis |

- 来源：`oh-my-opencode/src/agents/librarian.ts:303`

### COMMUNICATION RULES

- 不要在答复里暴露工具名（讲“我会去查”，不要讲“我会用 grep_app”）。
- 不要寒暄；所有代码结论必须 permalink；Markdown + 代码块；简洁。
- 来源：`oh-my-opencode/src/agents/librarian.ts:317`

### Trigger & Use When（触发器）

- 触发器：提及外部库/来源即调用 Librarian。
- 常见场景：如何使用某库、框架最佳实践、依赖行为原因、找开源用例。
- 来源：`oh-my-opencode/src/agents/librarian.ts:11`

来源定位（阶段标题行）：`oh-my-opencode/src/agents/librarian.ts:46`, `oh-my-opencode/src/agents/librarian.ts:56`, `oh-my-opencode/src/agents/librarian.ts:69`, `oh-my-opencode/src/agents/librarian.ts:117`, `oh-my-opencode/src/agents/librarian.ts:208`, `oh-my-opencode/src/agents/librarian.ts:242`, `oh-my-opencode/src/agents/librarian.ts:276`, `oh-my-opencode/src/agents/librarian.ts:303`, `oh-my-opencode/src/agents/librarian.ts:317`

## 2. 工具系统 (可调用工具)

**权限差异 (Librarian 专属)**
- 明确禁止：`write`, `edit`, `task`, `delegate_task`, `call_omo_agent`
- 其余工具：由“全局权限规则 + 会话权限”决定，未命中规则时默认是 `ask`
- 来源：`oh-my-opencode/src/agents/librarian.ts:24`, `oh-my-opencode/src/agents/AGENTS.md:54`

**权限合并与裁决规则 (重要，决定“能不能用”)**
- 权限不是“文档列了就能用”，而是由多层规则合并后裁决。
- 规则来源：Agent 自身权限 + 会话权限（用户/项目配置）+ 用户工具禁用项。
- 运行时会按合并结果剔除工具（被 deny 的工具不会出现在可用工具列表里）。
- 未命中任何规则时默认是 `ask`（调用时弹权限请求）。
- `edit` 权限覆盖 `edit/write/patch/multiedit`。
- 来源：`opencode/packages/opencode/src/permission/next.ts:63`, `opencode/packages/opencode/src/permission/next.ts:231`, `opencode/packages/opencode/src/session/llm.ts:268`

**重要说明：这里的“工具清单”是完整目录，不等于“全部可直接用”**
- Librarian 的“显式禁止”只有上面五项；其他工具是否可用取决于权限规则合并结果。
- 运行时会移除被禁用的工具：`session/llm.ts` 会按权限剔除工具。
- 参考：`opencode/packages/opencode/src/session/llm.ts:268`

**工具清单与用途 (目录清单；是否可用由权限模型决定)**

LSP
- `lsp_goto_definition`：跳转到定义
- `lsp_find_references`：查找引用
- `lsp_symbols`：符号检索
- `lsp_diagnostics`：诊断错误/警告
- `lsp_prepare_rename`：校验重命名可行性
- `lsp_rename`：跨文件重命名

AST-Grep
- `ast_grep_search`：按语法树模式搜索
- `ast_grep_replace`：按语法树模式替换

Search
- `grep`：内容正则搜索
- `glob`：文件路径匹配

Session
- `session_list`：列出会话
- `session_read`：读取会话消息
- `session_search`：搜索会话内容
- `session_info`：会话元数据

Agent
- `delegate_task`：委托子代理或按 category 生成 Sisyphus-Junior
- `call_omo_agent`：直接调用指定代理（Librarian 被禁用）

Background
- `background_output`：获取后台任务结果
- `background_cancel`：取消后台任务

System
- `interactive_bash`：tmux 交互式会话
- `look_at`：分析图像/PDF 等多模态文件

Skill
- `skill`：加载并执行技能
- `skill_mcp`：调用技能内 MCP
- `slashcommand`：运行斜杠命令

Core
- `bash`：执行命令
- `read`：读取文件
- `write`：创建文件
- `edit`：修改文件
- `todowrite`：写入 Todo 列表
- `todoread`：读取 Todo 列表
- `question`：向用户提问
- `webfetch`：抓取网页内容
- `websearch`：网页搜索
- `codesearch`：代码搜索
- `task`：任务状态工具

工具来源：`oh-my-opencode/src/tools/AGENTS.md:1`, `opencode-research/main_agent.md:28`

## 3. 可调用 Sub-agent (含用途)

Librarian 不能调用子代理（`delegate_task` 被禁用）。

来源：`oh-my-opencode/src/agents/librarian.ts:24`, `oh-my-opencode/src/agents/AGENTS.md:54`

## 4. Hook (生命周期钩子与作用)

- `subagent-question-blocker`：阻止子代理直接提问用户
- `tool-output-truncator`：截断过长工具输出
- `comment-checker`：限制多余注释
- `directory-agents-injector`：注入目录级 AGENTS.md

来源：`oh-my-opencode/src/hooks/AGENTS.md:22`, `oh-my-opencode/src/hooks/AGENTS.md:37`, `oh-my-opencode/src/hooks/AGENTS.md:49`, `oh-my-opencode/src/hooks/AGENTS.md:25`

## 5. Skills (可加载技能与作用)

- `playwright`：浏览器自动化（MCP，测试/抓取/截图）
- `agent-browser`：浏览器自动化（CLI 方式，测试/抓取/截图）
- `frontend-ui-ux`：前端设计与 UI 质量提升
- `git-master`：Git 操作专家（提交/回滚/历史检索）
- `dev-browser`：带持久状态的浏览器自动化

来源：`oh-my-opencode/src/features/builtin-skills/skills.ts:16`, `oh-my-opencode/src/features/builtin-skills/*/SKILL.md:1`

## 6. 关键差异小结

- Librarian 面向“外部证据检索”，强调引用与可追溯性。
- Explore 面向“内部代码探索”，强调路径与可执行线索。
- Oracle 用于复杂推理与架构咨询，Librarian 不做推理，只给证据。

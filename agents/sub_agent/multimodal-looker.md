# OhMyOpenCode 主 Agent 报告 (Multimodal Looker)

一句话定位：多模态文件分析员，只读读取图像/PDF 等文件并按目标提取信息；用于“需要解释/抽取信息”的场景，而不是原样抄文本。

## 1. 身份与系统提示词 (Identity & Prompt)

Multimodal Looker 的系统提示词是一个简单的“用途说明 + 工作步骤 + 输出规则”。
这里严格按系统提示词原始顺序梳理（顺序与 `oh-my-opencode/src/agents/multimodal-looker.ts` 保持一致）。

系统提示词来源：`oh-my-opencode/src/agents/multimodal-looker.ts:24`

### Role

- 定位：解释无法当作纯文本阅读的媒体文件。
- 目标：只提取被请求的内容（ONLY what was requested）。

### When to use you

- Read 工具无法“解释”的媒体文件。
- 需要从文档里提取信息/摘要。
- 描述图片或图表/架构图内容。
- 需要“分析后的信息”，而不是原始文本。

### When NOT to use you

- 源码/纯文本需要逐字内容（用 Read）。
- 之后需要编辑文件（需要先获取 literal content）。
- 简单读取无需解释。

### How you work

1) 接收 file path + goal。
2) 深度阅读并分析。
3) 只返回与 goal 相关的提取信息。
4) 主 agent 不处理 raw file（节省 tokens）。

### Modality hints

- PDFs：抽取文本/结构/表格/指定章节数据。
- Images：描述布局、UI 元素、文本、图形、图表。
- Diagrams：解释关系/流程/架构。

### Response rules

- 直接返回提取信息（no preamble）。
- 找不到就明确说明缺失。
- 跟随请求语言。
- 对 goal 彻底，其余尽量简洁。
- 输出直达主 agent。

来源定位（关键段落行）：`oh-my-opencode/src/agents/multimodal-looker.ts:24`

## 2. 工具系统 (可调用工具)

**权限差异 (Multimodal Looker 专属)**
- Allowlist：仅允许 `read`（其余默认 deny）
- 来源：`oh-my-opencode/src/agents/multimodal-looker.ts:15`

**权限合并与裁决规则 (重要，决定“能不能用”)**
- 权限不是“文档列了就能用”，而是由多层规则合并后裁决。
- 规则来源：Agent 自身权限 + 会话权限（用户/项目配置）+ 用户工具禁用项。
- 运行时会按合并结果剔除工具（被 deny 的工具不会出现在可用工具列表里）。
- 未命中任何规则时默认是 `ask`（调用时弹权限请求）。
- 来源：`opencode/packages/opencode/src/permission/next.ts:63`, `opencode/packages/opencode/src/permission/next.ts:231`, `opencode/packages/opencode/src/session/llm.ts:268`

工具来源：`oh-my-opencode/src/tools/AGENTS.md:1`

## 3. 可调用 Sub-agent (含用途)

Multimodal Looker 是只读工具型子代理，不负责分发任务。

来源：`oh-my-opencode/src/agents/multimodal-looker.ts:15`

## 4. Hook (生命周期钩子与作用)

作为子代理，它仍会被通用钩子体系包裹（如输出截断、稳定性恢复）。

来源：`oh-my-opencode/src/hooks/AGENTS.md:62`

## 5. Skills (可加载技能与作用)

该代理本身就是“多模态能力封装”，不依赖额外技能。

来源：`oh-my-opencode/src/agents/multimodal-looker.ts:24`

## 6. 关键差异小结

- Multimodal Looker 是“读取并解释媒体文件”的子代理：只读 + 目标导向提取。
- Explore/Librarian 是“搜索定位”子代理：前者偏内部代码，后者偏外部资料。

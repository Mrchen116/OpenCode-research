opencode的最大两个优点：
- 可拓展性强，任何东西几乎都可以通过插件实现
- provider和产品分离

细节：
- Oh-my-opencode是 OpenCode 的插件，而“Claude Code 兼容层”指的是 OpenCode 读取 Claude Code 的配置与目录结构（commands/skills/agents/MCP/hooks 等），不是让 Claude Code 去加载 oh-my-opencode。
- command可以自定义，hook可以自定义，工具可以自定义
- hook可以订阅的event非常多，而且hook是通过js代码实现，可以非常自由。内置大量hook对llm全程看护。
- 采用TUI和agent前后端分离，http通信的设计 https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/可以多端使用
[img](https://cefboud.com/assets/opencode-arch.png)
- 探索代码仓，默认多个subagent，同时后台进行
- agent的系统提示词更像workflow，一步步写清楚他要做的事情。
# Tools Category: Web & Code Search

面向外部内容检索与抓取的工具。

## `webfetch`
功能: 抓取指定 URL 内容。
描述摘要: 支持 text/markdown/html 格式并限制响应大小。
来源文件: `opencode/packages/opencode/src/tool/webfetch.ts`
入参: `url: string`, `format?: "text"|"markdown"|"html"`, `timeout?: number`
出参: `{ title, output, metadata }`，`output` 为抓取内容。

## `websearch`
功能: 外部网页搜索。
描述摘要: 基于 Exa MCP，支持搜索类型与实时抓取模式。
来源文件: `opencode/packages/opencode/src/tool/websearch.ts`
入参: `query: string`, `numResults?: number`, `livecrawl?: "fallback"|"preferred"`, `type?: "auto"|"fast"|"deep"`, `contextMaxCharacters?: number`
出参: `{ title, output, metadata }`，`output` 为搜索结果文本或提示。

## `codesearch`
功能: 代码与文档上下文搜索。
描述摘要: 基于 Exa Code API 返回高质量上下文。
来源文件: `opencode/packages/opencode/src/tool/codesearch.ts`
入参: `query: string`, `tokensNum?: number`
出参: `{ title, output, metadata }`，`output` 为检索到的上下文文本。

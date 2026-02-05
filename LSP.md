# oh-my-opencode LSP 工具实现笔记

## 1. LSP 实现机制与多语言支持原理

`oh-my-opencode` 采用了 **客户端-服务器架构**，通过标准的 LSP 协议与各种编程语言的语言服务器进行通信。

### 核心实现组件
- **`LSPClient` (`client.ts`)**: 核心客户端类。
    - **进程管理**: 使用 Bun 的 `spawn` 启动外部 LSP 进程（如 `rust-analyzer`, `pyright`）。
    - **RPC 通信**: 使用 `vscode-jsonrpc` 通过 stdin/stdout 进行 JSON-RPC 通信。
    - **状态同步**: 通过 `textDocument/didOpen` 和 `textDocument/didChange` 实时同步内存中的代码改动。
- **`LSPServerManager` (`client.ts`)**: 单例管理类，维护连接池，支持自动清理闲置 5 分钟以上的服务器进程。

### 多语言支持原理
- **配置驱动**: 在 `constants.ts` 中预定义了 20 多种语言的 `BUILTIN_SERVERS` 配置，包含启动命令和关联的文件后缀。
- **动态匹配**: Agent 操作文件时，根据后缀名动态查找匹配的服务器配置。
- **协议标准化**: 只要服务器遵循 LSP 标准，`LSPClient` 就能用统一的逻辑（如 `textDocument/definition`）进行交互。

## 2. 找不到服务器时的提示机制

当用户尝试对某种语言使用 LSP 功能但本地未安装对应的服务器时，`oh-my-opencode` 会触发提示机制：

- **查找流程**: `findServerForExtension` 函数会检查命令是否在系统的 `PATH` 中。
- **提示内容**: 
    - 如果是内置支持的语言，会从 `LSP_INSTALL_HINTS` 中提取具体的安装建议（例如 Python 会提示 `npm install -g pyright` 或 `pip install ruff`）。
    - 如果没有预设提示，则默认提示 `Install '<command>' and ensure it's in your PATH`。
- **触发方式**: 
    - **主动触发**: 用户运行 `omo doctor` 时，会列出所有未安装但支持的服务器及其安装命令。
    - **被动触发**: Agent 调用工具失败时，会将安装建议作为错误信息返回给 Agent，Agent 随后会告知用户。

## 3. 与 IDE 交互 vs LSP 的区别

这是一个关键的澄清：**`oh-my-opencode` 与 IDE 的交互并不是为了替代 LSP，而是为了获取“实时上下文”。**

| 功能维度 | IDE 伴侣 (vscode-ide-companion) | 独立 LSP 工具 |
| :--- | :--- | :--- |
| **核心目的** | 获取用户当前的“视觉”上下文 | 执行深度代码分析和重构 |
| **交互对象** | 正在运行的 VS Code 实例 | 独立的二进制进程 (如 `pyright-langserver`) |
| **获取内容** | 当前打开的文件、光标位置、选中的代码、编辑器报错 | 函数定义位置、所有引用、符号列表、全局重命名 |
| **依赖关系** | 需要安装 VS Code 扩展 | 需要在系统中安装对应的 LSP 服务器 |
| **使用场景** | Agent 问：“我正在看的这段代码有什么问题？” | Agent 说：“帮我把这个函数在全工程重命名。” |

**结论**: `oh-my-opencode` 并不“傻”到非要你自己装 LSP，它能通过 IDE 伴侣感知你的环境；但为了实现**脱离 IDE 也能运行**的强大重构能力，它选择自己维护一套独立的 LSP 客户端。

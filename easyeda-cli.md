# 嘉立创EDA专业版命令行客户端 / EasyEDA Pro Command-Line Client

**[English](#english)** | **[中文](#中文)**

---

## English

Besides the graphical interface, the EasyEDA Pro desktop client ships with a command-line client (CLI) and a local bridge service. They let you control a running editor from a script, or let an AI coding assistant drive the editor directly.

> The CLI and the extension API are still evolving. Command names, options, and API signatures may change between client versions. This document therefore stays short: it explains the concepts and how to discover commands at runtime. Whenever help output differs from this document, **trust the help output of the installed version**.

### Requirements

| Item | Description |
| --- | --- |
| Product | EasyEDA Pro **desktop client** v4.1.60 or above. The browser / online edition does not support sessions or the bridge |
| Executable | `easyeda-pro` (international) or `lceda-pro` (mainland China); the same binary starts the GUI and serves the CLI |
| Installation | The CLI is built into the client and needs no separate installation. Download from the [client download page](https://easyeda.com/page/download); specific versions are available from the [client release history](https://prodocs.easyeda.com/en/faq/client-version.html) |

The installer normally adds the executable directory to `PATH`. If the command is not found, add the directory to `PATH` or use the full path to the executable.

### Capabilities

- Open and create projects, and manage multiple editor windows (sessions);
- Execute JavaScript inside the editor, and therefore call any [extension API](https://prodocs.easyeda.com/en/api/reference/pro-api.html);
- Query the extension API reference and the [design file format reference](./easyeda-format.md);
- Return results as uniform JSON, which is convenient for scripts to parse;
- Run as an [MCP](https://modelcontextprotocol.io/) server, and therefore serve directly as a tool for an AI coding assistant.

### Basic Concepts

The CLI treats a running editor as a programmable backend:

| Concept | Description |
| --- | --- |
| Bridge service | The local service provided by the client process; every CLI command communicates with the editor through it |
| Session | One editor window. A session is created by `open`, which returns the session ID; later commands need that ID |
| Invocation | A piece of JavaScript executed inside a given session through `invoke` |
| Response envelope | Every command returns the same JSON structure. Only the top-level `ok` field needs to be checked; on failure, read the `error` field |

### Discovering Commands

Do not rely on memorized commands. The installed executable documents itself:

1. **List all commands** — run `easyeda-pro --help`. The header prints the exact product version, followed by the full command list.
2. **Search by keyword** — when you are unsure of a name, run `easyeda-pro --search <keyword>`. Keywords are English (for example `session`, `invoke`, `doc`).
3. **Read a command's usage** — every command and subcommand supports `--help`, for example `easyeda-pro <command> --help` or `easyeda-pro <subcommand> --help`. The output gives the usage line, options, defaults, and accepted values.

The built-in `doc` command family serves the editor reference: it can list the API classes, the methods of a class, the full signature of a method, enumerations, and the design-file format field definitions. Start with `easyeda-pro doc --help` and follow the subcommand help from there.

### MCP Server Mode

The CLI can run as an MCP server so that an AI coding assistant can use it directly as a tool. Both the transport options and their parameters are shown by the help output; see `easyeda-pro --mcp --help` for the current syntax.

### Related Documents

- [For AI: Command-Line Client](./cli-for-ai.md) — for AI coding assistants: how to discover commands and read help at runtime;
- [Design File Format](./easyeda-format.md) — the V3 design file format behind `.eprj3` / `.epro2` projects.

---

## 中文

嘉立创 EDA 专业版客户端除了提供图形界面，还内置了命令行客户端（CLI）与本地桥接服务，允许你以脚本方式控制正在运行的编辑器，或由 AI 编程助手直接驱动编辑器完成设计工作。

> CLI 与扩展 API 仍在演进中，命令名称、选项和 API 签名可能随客户端版本变化。因此本文只介绍概念与「如何查找命令」的方法。若帮助输出与本文不一致，**以所安装版本的帮助输出为准**。

### 版本要求

| 项目 | 说明 |
| --- | --- |
| 产品 | 嘉立创 EDA 专业版**桌面客户端** v4.1.60 及以上。浏览器 / 在线版本不支持会话与桥接功能 |
| 可执行文件 | 国际版为 `easyeda-pro`，国内版为 `lceda-pro`；同一可执行文件既用于启动图形界面，也用于执行命令行操作 |
| 安装 | 命令行客户端内置于客户端，无需单独安装。请前往[客户端下载页面](https://lceda.cn/page/download)下载；特定版本可前往[客户端历史版本](https://prodocs.lceda.cn/faq/client-version.html)页面获取 |

安装程序通常会将可执行文件目录加入 `PATH`。若提示命令不存在，请将该目录加入 `PATH`，或直接使用可执行文件的完整路径。

### 能力

- 打开、新建工程，并在多个编辑器窗口（会话）之间进行管理；
- 在编辑器内执行 JavaScript 代码，从而调用任意[扩展 API](https://prodocs.lceda.cn/api/reference/pro-api.html)；
- 查询扩展 API 参考文档与[设计文件格式规范](./easyeda-format.md)；
- 以统一的 JSON 格式返回执行结果，便于脚本解析；
- 以 [MCP](https://modelcontextprotocol.io/) 服务器模式运行，直接作为 AI 编程助手的工具。

### 基本概念

命令行客户端将正在运行的编辑器视为一个可编程的后端：

| 概念 | 说明 |
| --- | --- |
| 桥接服务 | 由客户端进程提供的本地服务，所有 CLI 命令均通过它与编辑器通信 |
| 会话 | 一个编辑器窗口对应一个会话，由 `open` 创建并获得会话 ID；后续命令需要该 ID |
| 调用 | 通过 `invoke` 在指定会话内执行一段 JavaScript 代码 |
| 返回值 | 所有命令返回统一结构的 JSON，只需检查顶层 `ok` 字段；失败时读取 `error` 字段 |

### 查找命令

不要依赖记忆中的命令。已安装的可执行文件自带完整帮助：

1. **列出全部命令** — 执行 `lceda-pro --help`，帮助头部会打印确切的产品版本，随后是完整命令列表。
2. **按关键词搜索** — 不确定命令名时执行 `lceda-pro --search <关键词>`，关键词使用英文（如 `session`、`invoke`、`doc`）。
3. **查看命令用法** — 所有命令与子命令均支持 `--help`，例如 `lceda-pro <命令> --help` 或 `lceda-pro <子命令> --help`，输出包含用法、选项、默认值与可选值。

内置的 `doc` 命令族提供编辑器参考文档：可列出 API 类、某个类的方法、某个方法的完整签名、枚举，以及设计文件格式的字段定义。先执行 `lceda-pro doc --help`，再按子命令的帮助逐级深入。

### MCP 服务器模式

CLI 可以以 MCP 服务器模式运行，直接作为 AI 编程助手的工具。传输方式及其参数均以帮助输出为准，当前语法见 `lceda-pro --mcp --help`。

### 相关文档

- [For AI：命令行客户端](./cli-for-ai.md) — 面向 AI 编程助手：如何在运行时查找命令与读取帮助；
- [设计文件格式](./easyeda-format.md) — `.eprj3` / `.epro2` 工程背后的 V3 设计文件格式。

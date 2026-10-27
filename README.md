# easyeda-client-cli

[![English](https://img.shields.io/badge/docs-English-blue)](#english) [![中文](https://img.shields.io/badge/文档-中文-red)](#中文)

EasyEDA Pro (嘉立创 EDA 专业版) desktop client ships with a command-line client (CLI) and a local bridge service. They let you control a running editor from a script, or let an AI coding assistant drive the editor directly — from automated design all the way to exporting manufacturing files.

嘉立创 EDA 专业版客户端内置了命令行客户端（CLI）与本地桥接服务，允许你以脚本方式控制正在运行的编辑器，或由 AI 编程助手直接驱动编辑器完成设计工作，实现从自动化设计到导出制造文件的全流程。

## English

### What the CLI provides

- Opening and creating projects, and managing multiple editor windows (sessions);
- Executing JavaScript inside the editor, and therefore calling any [extension API](https://prodocs.easyeda.com/en/api/reference/pro-api.html);
- Querying the extension API reference and the design file format reference (`.eprj3`);
- Returning results as uniform JSON, which is convenient for scripts and AI to parse;
- Running as an [MCP](https://modelcontextprotocol.io/) server, serving directly as a tool for an AI coding assistant.

### Documentation

| Document | Description |
| --- | --- |
| [Command-Line Client](./easyeda-cli.md) | Bilingual (English / 中文) overview: requirements, capabilities, core concepts, and how to discover commands and read help at runtime — no hard-coded command examples |
| [For AI: Command-Line Client](./cli-for-ai.md) | Written for AI coding assistants: how to discover commands, search by keyword, read `--help` usage, and query the built-in API / file-format reference at runtime — no hard-coded command or API examples, since the CLI is still evolving |
| [Design File Format](./easyeda-format.md) | The V3 design file format behind `.eprj3` / `.epro2` projects: log documents, record structure, eventual consistency, and the unit system (schematic `10 mil`, PCB `mil`) |

> If you are using an AI coding assistant (Claude Code, OpenCode, QwenCode, GitHub Copilot, etc.) to drive EasyEDA Pro, point the AI at [cli-for-ai.md](./cli-for-ai.md) — it is designed to be followed directly by agents.

### Quick start

> **Note:** The CLI requires EasyEDA Pro client **V4.1.60** or above.

```shell
# The CLI is built into the desktop client; verify the runtime environment first
easyeda-pro doctor

# Open or create a project and get a session ID
easyeda-pro session open

# Execute JavaScript inside the editor
easyeda-pro invoke --ext-uuid eda "eda.sys_Message.showMessage('Hello EasyEDA!')"
```

See [Command-Line Client](./easyeda-cli.md) for the complete guide.

## 中文

### CLI 提供的能力

- 打开、新建工程，并在多个编辑器窗口（会话）之间进行管理；
- 在编辑器内执行 JavaScript 代码，从而调用任意 [扩展 API](https://prodocs.lceda.cn/api/reference/pro-api.html)；
- 查询扩展 API 参考文档与设计文件格式规范（`.eprj3`）;
- 以统一的 JSON 格式返回执行结果，便于脚本与 AI 解析；
- 以 [MCP](https://modelcontextprotocol.io/) 服务器模式运行，直接作为 AI 编程助手的工具。

### 文档

| 文档 | 说明 |
| --- | --- |
| [命令行客户端](./easyeda-cli.md) | 双语（英文 / 中文）概览：版本要求、能力、核心概念，以及在运行时查找命令与读取帮助的方法——不提供写死的命令范例 |
| [For AI：命令行客户端](./cli-for-ai.md) | 面向 AI 编程助手编写：教会 AI 在运行时查找命令、按关键词搜索、读取 `--help` 用法，以及查询内置 API / 文件格式参考——不提供写死的命令与 API 范例，因为 CLI 仍在演进 |
| [设计文件格式](./easyeda-format.md) | `.eprj3` / `.epro2` 工程背后的 V3 设计文件格式：日志文档、记录结构、最终一致性，以及单位体系（原理图 `10 mil`，PCB `mil`） |

> 如果你正使用 AI 编程助手（Claude Code、OpenCode、QwenCode、GitHub Copilot 等）驱动嘉立创 EDA 专业版，请让 AI 遵循 [cli-for-ai.md](./cli-for-ai.md) 执行——该文档专为 Agent 直接执行而编写。

### 快速开始

> **注意：** 命令行客户端要求客户端版本在 **V4.1.60** 及以上。

```shell
# CLI 内置于客户端，先确认运行环境
lceda-pro doctor

# 打开或新建工程并获得会话 ID
lceda-pro session open

# 在编辑器内执行 JavaScript 代码
lceda-pro invoke --ext-uuid eda "eda.sys_Message.showMessage('Hello EasyEDA!')"
```

完整指南请阅读[命令行客户端](./easyeda-cli.md)。

## Notes / 说明

- **Version requirement / 版本要求：the CLI requires EasyEDA Pro client `V4.1.60` or above; older versions do not include the command-line client. / 命令行客户端要求客户端版本在 `V4.1.60` 及以上，更早版本的客户端不包含该功能。**
- The CLI requires the **desktop client** of EasyEDA Pro; the browser / online edition does not support sessions and the bridge.
- 命令行客户端要求使用**客户端版本**的嘉立创 EDA 专业版，浏览器 / 在线版本不支持会话与桥接功能。
- The executable name differs between distributions: `easyeda-pro` (international) and `lceda-pro` (China). The documentation above matches each distribution accordingly.
- 可执行文件名因发行渠道不同而异：国际版为 `easyeda-pro`，国内版为 `lceda-pro`。上文各语言文档与对应发行渠道保持一致。

## License

Apache-2.0

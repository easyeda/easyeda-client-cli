# 命令行客户端 <Badge type="tip" text="CLI" />

::: tip 面向 AI 编程助手：命令行客户端

如果你正使用 AI 编程助手（如 Claude Code、OpenCode、QwenCode、GitHub Copilot 等）驱动嘉立创 EDA 专业版完成设计工作，请让 AI 遵循 [For AI 命令行客户端文档](https://prodocs.lceda.cn/storage/texts/api/guide/cli-for-ai.md) 执行。该文档面向 AI 编程助手编写，去除了冗余叙述，并提供了可直接执行的命令、硬性约束与校验步骤。

你也可以直接将下面的提示词复制给 AI 编程助手：

```text
请协助我通过嘉立创 EDA 专业版客户端的命令行（lceda-pro）完成设计任务，要求如下：
1. 严格遵循 https://prodocs.lceda.cn/storage/texts/api/guide/cli-for-ai.md 中的步骤执行；
2. 动手前先通过 lceda-pro doctor 确认端点连接状态，并向我确认工程名称等信息；
3. 关键命令执行后需检查返回值的 ok 字段与磁盘产物，完成后汇报工程目录、导出产物路径与会话关闭情况。
```

:::

嘉立创 EDA 专业版客户端除了提供图形界面，还内置了一套命令行客户端（CLI）与本地桥接服务，允许你以脚本方式控制正在运行的编辑器，或由 AI 编程助手直接驱动编辑器完成设计工作。

命令行客户端提供了以下能力：

- 打开、新建工程，并在多个编辑器窗口（会话）之间进行管理；
- 在编辑器内执行 JavaScript 代码，从而调用任意 [扩展 API](./index)；
- 查询扩展 API 参考文档与设计文件格式规范；
- 以统一的 JSON 格式返回执行结果，便于脚本与 AI 解析；
- 以 [MCP](https://modelcontextprotocol.io/) 服务器模式运行，直接作为 AI 编程助手的工具。

## 基本概念

命令行客户端将正在运行的编辑器视为一个可编程的后端，客户端与编辑器之间存在以下三层关系：

| 概念       | 说明                                                                   |
| ---------- | ---------------------------------------------------------------------- |
| 会话       | 一个编辑器窗口对应一个会话，由 `open` 创建并获得会话 ID                |
| 调用       | 通过 `invoke` 在指定会话内执行一段 JavaScript 代码                     |
| 桥接服务   | 由客户端进程提供的本地服务，`lceda-pro` 命令即通过它与编辑器通信       |

命令行客户端与服务均由 `lceda-pro` 可执行文件提供，因此该可执行文件既用于启动图形界面，也用于执行命令行操作。

## 准备工作

### I. 获取客户端

命令行客户端内置于嘉立创 EDA 专业版客户端中，无需单独安装。请前往 [客户端下载页面](https://lceda.cn/page/download) 下载并安装适用于你系统的嘉立创 EDA 专业版客户端。

安装完成后，`lceda-pro` 可执行文件通常位于以下位置（以 Windows x64 版本为例）：

```
C:\Program Files\lceda-pro\lceda-pro.exe
```

安装程序会在安装过程中将该目录加入系统 `PATH`，因此你一般可以直接在终端内使用 `lceda-pro` 命令。若提示命令不存在，请检查安装目录是否已加入 `PATH`，或直接使用该可执行文件的完整路径。

你也可以前往 [客户端历史版本](../../faq/client-version) 页面获取特定版本的安装包。

::: warning

命令行客户端要求使用客户端版本的嘉立创 EDA 专业版，浏览器 / 在线版本不支持本文档描述的会话与桥接功能。

:::

### II. 确认运行环境

在终端内执行以下命令，客户端将会探测本地编辑器端点并报告其状态：

```shell
lceda-pro doctor
```

```json
{
	"ok": true,
	"value": {
		"endpoint": "JLCEDAProf126dbc1",
		"connected": false,
		"bridgeVersion": null,
		"versionMatch": null,
		"resultChannel": null
	},
	"logs": [],
	"durationMs": 207
}
```

`connected` 为 `false` 表示当前没有正在运行的编辑器，打开任意一个编辑器窗口后该字段将变为 `true`。

你同样可以在脚本内查询当前运行环境的属性，例如：

```javascript
return {
	isClient: eda.sys_Environment.isClient(),
	isHalfOfflineMode: eda.sys_Environment.isHalfOfflineMode(),
	version: eda.sys_Environment.getEditorCurrentVersion()
};
```

其中 [SYS_Environment.isClient()](../reference/pro-api.sys_environment.isclient) 用于判断当前是否为客户端环境，[SYS_Environment.isHalfOfflineMode()](../reference/pro-api.sys_environment.ishalfofflinemode) 用于判断是否为半离线模式，[SYS_Environment.getEditorCurrentVersion()](../reference/pro-api.sys_environment.geteditorcurrentversion) 用于获取当前编辑器版本号。

### III. 工程存储目录

在半离线 / 全离线模式下，客户端的数据目录位于：

```
%USERPROFILE%\Documents\LCEDA-Pro
```

该目录下的 `config.json` 文件记录了各类路径配置，其中 `APP_PROJECT_DIR` 为工程目录列表：

```json
"APP_PROJECT_DIR": [
	"C:\\Users\\username\\Documents\\LCEDA-Pro\\projects",
	"C:\\Users\\username\\Documents\\LCEDA-Pro\\example-projects"
]
```

[SYS_FileSystem](../reference/pro-api.sys_filesystem) 系列接口的读写范围即为上述目录及其子目录，你也可以通过 [SYS_FileSystem.getProjectsPaths()](../reference/pro-api.sys_filesystem.getprojectspaths) 在脚本内获取该路径列表。将导出产物写入 `projects\<工程名>\` 目录下是较为省事的做法。

### IV. 返回值结构

所有命令行客户端命令都返回同一种 JSON 结构：

```json
{
	"ok": true,
	"value": {},
	"logs": [],
	"durationMs": 12
}
```

当执行失败时，返回结构如下：

```json
{
	"ok": false,
	"error": {
		"code": "EXECUTION_ERROR",
		"message": "..."
	}
}
```

在脚本内处理返回值时，仅需判断最外层的 `ok` 字段即可。

## 命令总览

### I. 查看帮助

执行 `--help` 可以查看命令行客户端的全部命令：

```shell
lceda-pro --help
```

```
JLCEDA Pro 4.1.x.abcdef01

Usage:
  lceda-pro <command> [options]
  lceda-pro [global flags]

Commands:
  activate          Bring the server window up and focus it
  open              Open a project and return its session id; .eprj3 folder projects
                    stay in two-way sync with the editor; without --path open a bare editor
  stop              Gracefully stop the server
  doctor            Probe the endpoint and report its identity

  session/          Manage sessions
    list            List sessions (origin cli/user, project path, window title)
    close           Close a session this CLI opened
    abort           Force-kill a session (last resort)

  invoke            Invoke a registered function by extension id + name
  functions         List extensions known to the session

  doc/              Query the editor API reference (eda / external / format)
    api             Query the eda namespace (static editor API)
    external        Query the external namespace (extension-injected API)
    format          Query the format namespace (design format reference)

Global flags:
  --help              Show help for any command:  lceda-pro <command> --help
  --search <query>    Fuzzy-search commands when unsure of the name
  --mcp stdio|http    Start MCP server
```

### II. 搜索命令

若不确定命令名称，可以使用 `--search` 进行模糊搜索：

```shell
lceda-pro --search session
```

```
Matches:
  session.list    List sessions (each with origin cli/user, the project path and the window title)
  session.close   Close a session this CLI opened ...
  session.abort   Force-kill a session ...
  open            Open a project and return its session id ...
```

搜索关键词可以为 `session` `invoke` `doc` `permission` `BoardOutline` 等英文词汇。

### III. 查看子命令帮助

任意命令均可通过 `--help` 查看其用法：

```shell
lceda-pro open --help
```

```
open

Open a project and return its session id; .eprj3 folder projects stay in two-way sync
with the editor (disk edits reload automatically, editor saves land on disk automatically);
without --path open a bare editor (visible window; --headless for a hidden render)

Usage:
  lceda-pro open [--path <undefined>] [--headless <boolean>]

Options:
  --path       Project file path (.eprj/.eprj2/.eprj3/.elib); omit to open a bare editor
  --headless   Open without a visible window (hidden render); default opens a GUI window
```

```shell
lceda-pro invoke --help
```

```
invoke

Invoke a registered function by extension id + name (source runs on the editor command
space; omit extUuid to run with independent script permission)

Usage:
  lceda-pro invoke --session <undefined> [--code <undefined>] [--fn <undefined>]
                   [--args <undefined>] [--ext-uuid <undefined>] [--timeout <undefined>]

Options:
  --session   Session id (string, or positive integer)
  --code      Source snippet (string; async function body), reads args via __CLI__.args
  --fn        Function name (default: run)
  --args      Call arguments as JSON text, exposed to the source as __CLI__.args
  --ext-uuid  Extension id (optional; omit to run with independent script permission)
  --timeout   Invocation budget in ms (integer, 1-1800000; default 60000)
```

其余子命令的帮助信息也可以按同样的方式查看，例如 `lceda-pro session list --help`、`lceda-pro doc api --help`、`lceda-pro --mcp --help`。

### IV. 管理会话

执行 `open` 命令将会打开一个新的编辑器窗口，并返回其会话 ID：

```shell
lceda-pro open
```

```json
{
	"ok": true,
	"value": {
		"sessionId": "9aca2242-79bb-4e2b-9d03-754316927f96",
		"renderId": 1,
		"path": "",
		"headless": false
	},
	"logs": [],
	"durationMs": 1381
}
```

你也可以在打开时指定工程路径，或使用 `--headless` 以隐藏窗口的方式打开：

```shell
# 打开指定工程（.eprj3 文件夹工程会自动双向同步）
lceda-pro open --path "C:\Users\username\Documents\LCEDA-Pro\projects\MyBoard\MyBoard.eprj3"
```

```shell
# 隐藏窗口（无 GUI 渲染）
lceda-pro open --headless true
```

使用 `session list` 可以查看当前存在的所有会话：

```shell
lceda-pro session list
```

```json
{
	"ok": true,
	"value": [
		{
			"sessionId": "9aca…",
			"renderId": 1,
			"status": "alive",
			"headless": false,
			"origin": "cli",
			"path": null,
			"title": "嘉立创EDA(专业版) - V4.1.x"
		},
		{
			"sessionId": "3c66…",
			"renderId": 4,
			"status": "alive",
			"headless": false,
			"origin": "cli",
			"path": "C:\\…\\FMT_Test\\FMT_Test.eprj3",
			"title": "FMT_Test | 嘉立创EDA(专业版)"
		}
	]
}
```

使用 `session close` 可以关闭由命令行客户端打开的会话，附加 `--destroy` 参数将会连同窗口一并关闭：

```shell
lceda-pro session close --session 9aca2242-79bb-4e2b-9d03-754316927f96 --destroy
```

::: tip

建议在脚本内将 `open` 创建的会话使用完毕后，统一执行 `session close --destroy` 关闭，以免窗口堆积。

:::

### V. 作为 MCP 服务器运行

命令行客户端可以以 MCP 服务器模式运行，从而直接作为 AI 编程助手的工具调用：

```shell
# 本地客户端（Claude Code / Cursor 等）
lceda-pro --mcp stdio
```

```shell
# 远程 / 浏览器端 Agent
lceda-pro --mcp http --port 3030 --token <token>
```

## 执行 JavaScript

`invoke` 命令用于在指定的会话内执行一段 JavaScript 代码，这是命令行客户端最核心的能力，通过它可以调用全部扩展 API。

### I. 基本用法

最简单的调用如下，代码的返回值将会被序列化为 JSON 并回传：

```shell
lceda-pro invoke --session <SESSION_ID> --code "return 1 + 1"
```

```json
{
	"ok": true,
	"value": 2,
	"logs": [],
	"durationMs": 4
}
```

### II. 使用代码文件

当代码较长时，将其直接写在命令行内难以维护，建议将其保存为 `.js` 文件后再通过管道传入：

```shell
cat > probe.js <<'EOF'
const out = {};
out.client = eda.sys_Environment.isClient();
out.version = eda.sys_Environment.getEditorCurrentVersion();
out.projectCount = (await eda.dmt_Project.getAllProjectsUuid()).length;
return out;
EOF

lceda-pro invoke --session $S --ext-uuid eda --code "$(cat probe.js)"
```

### III. 常用选项

| 选项              | 说明                                                                                              |
| ----------------- | ------------------------------------------------------------------------------------------------- |
| `--ext-uuid eda`  | 以编辑器内置扩展的身份运行，可获得完整的外部交互权限（读写本地文件、访问系统接口等），推荐始终附加 |
| `--timeout <ms>`  | 单次调用的超时时间，默认为 `60000`，最大为 `1800000`，批量操作或自动布线等耗时任务应适当调大       |

### IV. 传递参数

通过 `--args` 可以向代码内传递 JSON 参数，代码内可通过 `__CLI__.args` 访问：

```shell
lceda-pro invoke --session $S --ext-uuid eda \
  --args '{"designator":"R1","x":1000,"y":800}' \
  --code 'const a = __CLI__.args; return a.designator + "@" + a.x + "," + a.y;'
```

```json
{
	"ok": true,
	"value": "R1@1000,800",
	"logs": [],
	"durationMs": 6
}
```

### V. 代码约定

传入的代码会被视为一个异步函数体，因此需要遵循以下约定：

- 使用 `return` 返回结果，返回值将会被序列化为 JSON；
- 可以直接使用 `await`；
- 建议路径统一使用正斜杠（`C:/Users/...`），以避免在不同终端之间传递时出现转义问题。

```javascript
// 推荐的路径写法
const DIR = 'C:/Users/username/Documents/LCEDA-Pro/projects/MyBoard/';
```

### VI. 获取文件

部分导出接口会返回 `File` 或 `Blob` 对象，例如截图、BOM、STEP、网表等。若需要将这些内容通过 JSON 传回，可以先将其转换为 Base64 字符串：

```javascript
const toB64 = (b) => new Promise((res) => {
	const fr = new FileReader();
	fr.onload = () => res(String(fr.result).split(',')[1] || '');
	fr.readAsDataURL(b);
});
```

以下示例获取原理图的 PNG 文件并转换为 Base64：

```javascript
const f = await eda.sch_ManufactureData.getPngFile('MySchematic', { width: 1400 });
return { name: f.name, size: f.size, b64: await toB64(f) };
```

### VII. 写入文件

更为省事的做法是直接使用 [SYS_FileSystem](../reference/pro-api.sys_filesystem) 系列接口将文件写入工程目录：

```javascript
const f = await eda.pcb_ManufactureData.get3DFile('MyBoard', 'step', ['Component Model'], 'Outfit', true);
await eda.sys_FileSystem.saveFileToFileSystem(
	'C:/Users/username/Documents/LCEDA-Pro/projects/MyBoard/MyBoard.step',
	f,
	undefined,
	true
); // 第 4 个参数 force = true 表示覆盖
```

常用的文件系统接口如下：

| 接口                                                                             | 说明                                                                |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| [saveFileToFileSystem(uri, file, fileName?, force?)](../reference/pro-api.sys_filesystem.savefiletofilesystem) | 写入文件，`uri` 以 `/` 结尾表示为目录，否则表示完整的文件名 |
| [readFileFromFileSystem(uri)](../reference/pro-api.sys_filesystem.readfilefromfilesystem) | 读取文件，返回 `File` 对象                                |
| [listFilesOfFileSystem(folderPath, recursive?)](../reference/pro-api.sys_filesystem.listfilesoffilesystem) | 列出目录内容                                      |
| [deleteFileInFileSystem(uri, force?)](../reference/pro-api.sys_filesystem.deletefileinfilesystem) | 删除文件                                                |
| [getProjectsPaths()](../reference/pro-api.sys_filesystem.getprojectspaths) / [getEdaPath()](../reference/pro-api.sys_filesystem.getedapath) / [getLibrariesPaths()](../reference/pro-api.sys_filesystem.getlibrariespaths) | 获取各类根目录 |

## 查询 API 文档

扩展 API 拥有数百个类，其文档采用渐进式披露的方式组织：先列出全部类，再查看某个类的方法，最后查看单个方法的完整签名。

### I. 列出全部类

```shell
lceda-pro doc api
```

该命令返回 `{ name, comment }` 形式的数组。类名以领域前缀分组：

| 前缀   | 领域                                                                       |
| ------ | -------------------------------------------------------------------------- |
| `DMT_` | 文档树 / 工程 / 板 / 原理图 / PCB / 编辑器控制                             |
| `SCH_` | 原理图图元（Component / Wire / Pin / Text / Drc / Netlist / ManufactureData） |
| `PCB_` | PCB 图元（Component / Pad / Line / Via / Polyline / Pour / Drc / ManufactureData / Layer / Net） |
| `LIB_` | 器件库（Device / Symbol / Footprint / Classification）                     |
| `SYS_` | 系统（FileSystem / Dialog / Window / Setting / Environment / MessageBus）  |
| `E*_`  | 枚举                                                                       |

### II. 查看类的方法

```shell
lceda-pro doc api --class-name DMT_Project
```

```json
{
	"ok": true,
	"value": {
		"title": "文档树 / 工程管理类",
		"methods": [
			{ "name": "createProject", "comment": "创建工程" },
			{ "name": "openProject", "comment": "打开工程" },
			{ "name": "getCurrentProjectInfo", "comment": "获取当前工程的详细属性" }
		],
		"callPath": "dmt_Project"
	}
}
```

返回值中的 `callPath` 即该类在 `eda` 对象上的属性名，因此 `DMT_Project` 类的方法调用形式为 `eda.dmt_Project.createProject(...)`。

### III. 查看方法的完整签名

```shell
lceda-pro doc api --class-name DMT_Project --method-name createProject
```

```json
{
	"ok": true,
	"value": {
		"title": "创建工程",
		"params": [
			{ "name": "projectFriendlyName", "type": "string", "description": "工程友好名称", "required": true },
			{ "name": "projectName", "type": "string", "description": "工程名称，不可重复，仅支持字母/数字/中划线", "required": false },
			{ "name": "teamUuid", "type": "string" },
			{ "name": "fileFormat", "type": "EDMT_ProjectFileFormat", "description": "工程文件格式，默认为 EPRJ3；仅在客户端半离线/全离线模式生效" }
		],
		"returns": { "type": "string | undefined", "description": "工程 UUID，如若为 undefined 则创建失败" },
		"isAsync": true,
		"example": "...(可直接运行的示例代码)..."
	}
}
```

每个方法均附带可直接运行的示例代码，这是了解方法用法的最高效途径。

### IV. 查看枚举

```shell
lceda-pro doc api --class-name EPCB_LayerId
```

```json
{
	"ok": true,
	"value": {
		"name": "EPCB_LayerId",
		"comment": "图层 ID",
		"values": [
			{ "name": "TOP", "value": 1, "description": "顶层" },
			{ "name": "BOARD_OUTLINE", "value": 11, "description": "板框层" }
		]
	}
}
```

### V. 查询设计文件格式

设计文件格式规范通过 `doc format` 子命令查询，其用法详见 [直接生成工程文件](#直接生成工程文件) 章节：

```shell
lceda-pro doc format
```

```shell
lceda-pro doc format --class-name TMSchComponent
```

## 通过扩展 API 绘制原理图

以下示例通过扩展 API 从零绘制一个分压与滤波电路，并导出网表、BOM 与截图。

```text
VCC ──┬── R1(1kΩ) ──┬── MID ── R2(10kΩ) ──┬── GND
      │             │                     │
      └── R3(1kΩ) ──┘        C1(100nF) ───┘   (C1 与 R2 并联)
```

### I. 建立会话

```shell
lceda-pro open
export S=9aca2242-79bb-4e2b-9d03-754316927f96
```

### II. 新建工程

[`DMT_Project.createProject()`](../reference/pro-api.dmt_project.createproject) 用于创建工程，随后通过 [`DMT_Project.openProject()`](../reference/pro-api.dmt_project.openproject) 将其打开：

```javascript
// step1_create_proj.js
const uuid = await eda.dmt_Project.createProject(
	'MyBoard', // 友好名称
	'my-board', // 工程名（字母 / 数字 / 中划线）
	undefined, // 团队，留空为个人
	undefined, // 文件夹，留空为根目录
	'教程示例工程', // 描述
	undefined, // 协作模式
	EDMT_ProjectFileFormat.EPRJ3 // 文件夹工程格式
);
await eda.dmt_Project.openProject(uuid);
await new Promise((r) => setTimeout(r, 3000));

const p = await eda.dmt_Project.getCurrentProjectInfo();
return {
	uuid: p.uuid,
	board: p.data[0].name,
	schematic: p.data[0].schematic.name,
	page: p.data[0].schematic.page[0].uuid,
	pcb: p.data[0].pcb.uuid
};
```

```shell
lceda-pro invoke --session $S --ext-uuid eda --code "$(cat step1_create_proj.js)"
```

```json
{
	"ok": true,
	"value": {
		"uuid": "5d3ca5e2…",
		"board": "Board1",
		"schematic": "Schematic1",
		"page": "429e4af1f4a665c3",
		"pcb": "509258b8ebfffb42"
	},
	"logs": [],
	"durationMs": 2611
}
```

创建完成后，磁盘上将会出现以下文件：

```
projects/MyBoard/
├── MyBoard.eprj3
├── sch/Schematic1/P1.esch2
├── sch/Schematic1/Schematic1.ecfg
├── pcb/PCB1.epcb2
└── panel/Panel1.epan2
```

### III. 搜索器件

[`LIB_LibrariesList.getSystemLibraryUuid()`](../reference/pro-api.lib_librarieslist.getsystemlibraryuuid) 用于获取系统库 UUID，[`LIB_Device.search()`](../reference/pro-api.lib_device.search) 用于在库内搜索器件：

```javascript
// step2_search.js
const sysLib = await eda.lib_LibrariesList.getSystemLibraryUuid();
const items = await eda.lib_Device.search('0603WAF1001T5E');
return {
	systemLibrary: sysLib,
	first: {
		name: items[0].name,
		uuid: items[0].uuid,
		lcsc: items[0].supplierId,
		libraryUuid: items[0].libraryUuid
	}
};
```

```json
{
	"ok": true,
	"value": {
		"systemLibrary": "0819f05c4eef4c71ace90d822a990e87",
		"first": {
			"name": "0603WAF1001T5E",
			"uuid": "18db11499a04403ba8c98e2f7a687fbc",
			"lcsc": "C21190",
			"libraryUuid": "0819f05c4eef4c71ace90d822a990e87"
		}
	},
	"durationMs": 7282
}
```

::: tip

在半离线模式下系统库是完整可用的，其中包含大量真实器件，并带有立创商城编号、封装与 3D 模型，`search()` 的返回项可以直接用于放置接口。

:::

本示例所使用的器件如下：

| 器件               | 立创商城编号 | 说明       |
| ------------------ | ------------ | ---------- |
| `0603WAF1001T5E`   | C21190       | 1kΩ 0603   |
| `0603WAF1002T5E`   | C25804       | 10kΩ 0603  |
| `CC0603KRX7R9BB104` | C14663      | 100nF 0603 |

### IV. 放置器件

首先通过 [`DMT_EditorControl.openDocument()`](../reference/pro-api.dmt_editorcontrol.opendocument) 打开原理图图页，然后使用 [`SCH_PrimitiveComponent.create()`](../reference/pro-api.sch_primitivecomponent.create) 放置器件。

推荐的做法是在 `create()` 之后先读取器件的属性快照，再将 `designator` 与 `otherProperty` 在同一次 [`SCH_PrimitiveComponent.modify()`](../reference/pro-api.sch_primitivecomponent.modify) 调用中一并写回，这样位号与全部器件属性可以一次成型：

```javascript
// step3_place.js
const pageUuid = '429e4af1f4a665c3'; // 上一步返回的 page
await eda.dmt_EditorControl.openDocument(pageUuid);
await new Promise((r) => setTimeout(r, 2500));

const LIB = '0819f05c4eef4c71ace90d822a990e87';
const R1K = { libraryType: '3', libraryUuid: LIB, uuid: '18db11499a04403ba8c98e2f7a687fbc' };
const R10K = { libraryType: '3', libraryUuid: LIB, uuid: 'b948db94476e4027ac8953235755ec96' };
const C100 = { libraryType: '3', libraryUuid: LIB, uuid: '96b39256cc3f4d80bd3b503deb4f3328' };

const out = [];
for (const [ref, lib, x, y] of [
	['R1', R1K, 400, 610],
	['R2', R10K, 400, 510],
	['C1', C100, 650, 510],
	['R3', R1K, 150, 610]
]) {
	// 1. 放置器件（坐标单位 10mil，rotation 为 API 角度）
	const c = await eda.sch_PrimitiveComponent.create(lib, x, y, undefined, 90, false, true, true);
	const id = c.getState_PrimitiveId();

	// 2. 先读取属性快照
	const pre = (await eda.sch_PrimitiveComponent.get(id)).getState_OtherProperty();

	// 3. 将位号与属性一并写回
	await eda.sch_PrimitiveComponent.modify(id, { designator: ref, otherProperty: pre });

	// 4. 读取引脚绝对坐标，供后续布线使用
	const pins = await eda.sch_PrimitiveComponent.getAllPinsByPrimitiveId(id);
	out.push({
		ref,
		id,
		value: pre.Value,
		pins: pins.map((p) => ({ n: p.pinNumber, x: p.x, y: p.y }))
	});
}
return out;
```

```json
{
	"ok": true,
	"value": [
		{
			"ref": "R1",
			"id": "4dc7d7c4967a383f",
			"value": "1kΩ",
			"pins": [
				{ "n": "2", "x": 400, "y": 590 },
				{ "n": "1", "x": 400, "y": 630 }
			]
		},
		{
			"ref": "R2",
			"id": "054e84ce1932ae79",
			"value": "10kΩ",
			"pins": [
				{ "n": "2", "x": 400, "y": 490 },
				{ "n": "1", "x": 400, "y": 530 }
			]
		},
		{
			"ref": "C1",
			"id": "6942169cfa200a6f",
			"value": "100nF",
			"pins": [
				{ "n": "1", "x": 650, "y": 530 },
				{ "n": "2", "x": 650, "y": 490 }
			]
		},
		{
			"ref": "R3",
			"id": "3acfabe56d603ec8",
			"value": "1kΩ",
			"pins": [
				{ "n": "2", "x": 150, "y": 590 },
				{ "n": "1", "x": 150, "y": 630 }
			]
		}
	]
}
```

::: tip

每个器件都应先 `create()` 再通过 [`SCH_PrimitiveComponent.getAllPinsByPrimitiveId()`](../reference/pro-api.sch_primitivecomponent.getallpinsbyprimitiveid) 读回引脚坐标，不要自行推算，引脚坐标是后续布线的唯一依据。

:::

### V. 放置网络标识

[`SCH_PrimitiveComponent.createNetFlag()`](../reference/pro-api.sch_primitivecomponent.createnetflag) 用于放置网络标识，其支持的类型为 `Power` `Ground` `AnalogGround` `ProtectGround`：

```javascript
// step4_flags.js
const vcc = await eda.sch_PrimitiveComponent.createNetFlag('Power', 'VCC', 400, 660, 0, false);
const gnd = await eda.sch_PrimitiveComponent.createNetFlag('Ground', 'GND', 650, 410, 0, false);
return {
	vcc: vcc.getState_PrimitiveId(),
	gnd: gnd.getState_PrimitiveId(),
	vccNet: vcc.getState_Net(),
	gndNet: gnd.getState_Net()
};
```

```json
{
	"ok": true,
	"value": {
		"vcc": "e6fc4f9698692111",
		"gnd": "2535215168c2bb20",
		"vccNet": "VCC",
		"gndNet": "GND"
	}
}
```

### VI. 绘制导线

[`SCH_PrimitiveWire.create()`](../reference/pro-api.sch_primitivewire.create) 用于绘制导线。导线为正交折线，一次 `create()` 调用可以绘制多段，但段与段之间必须首尾相连，且每一段都必须是水平的或垂直的：

```javascript
// step5_wire.js
const plan = [
	['VCC->R1.1', [[400, 660, 400, 630]]],
	['R1.2->R2.1', [[400, 590, 400, 530]]],
	['C1.1->MID', [[650, 530, 400, 530]]],
	['C1.2->GND', [[650, 490, 650, 410]]],
	['R2.2->GND', [[400, 490, 400, 410, 650, 410]]],
	['R3.1->VCC', [[150, 630, 150, 660, 400, 660]]],
	['R3.2->GND', [[150, 590, 150, 410, 400, 410]]]
];
const made = [];
for (const [tag, line] of plan) {
	const w = await eda.sch_PrimitiveWire.create(line);
	made.push({ tag, net: w.getState_Net(), id: w.getState_PrimitiveId() });
}
return made;
```

```json
{
	"ok": true,
	"value": [
		{ "tag": "VCC->R1.1", "id": "11b3075f364b1790" },
		{ "tag": "R1.2->R2.1", "id": "2d6fcb2792dae43f" },
		{ "tag": "C1.1->MID", "id": "2d6fcb2792dae43f" }
	]
}
```

::: tip

相邻的线段会被编辑器自动合并为一个 `WIRE` 图元组，这是正常行为。只要坐标精确落在引脚上，导线即可正确连通。

:::

### VII. 执行 DRC 检查

[`SCH_Drc.check()`](../reference/pro-api.sch_drc.check) 用于执行设计规则检查，其参数含义如下：

| 参数                 | 说明                                                             |
| -------------------- | ---------------------------------------------------------------- |
| `strict`             | 严格模式，当前统一传入 `true`                                    |
| `userInterface`      | 是否呼出底部 DRC 面板，脚本内应传入 `false`                       |
| `includeVerboseError` | 传入 `true` 返回违规明细数组，传入 `false` 仅返回是否全部通过     |

```javascript
// step6_drc.js
await eda.sch_Document.save();
await new Promise((r) => setTimeout(r, 8000)); // 等待编辑器刷新

const verbose = await eda.sch_Drc.check(true, false, true); // 返回违规明细数组
const passed = await eda.sch_Drc.check(true, false, false); // 返回是否全部通过
return {
	violationCount: verbose.length,
	passed,
	violations: verbose.map((v) => v.rule + '(' + v.type + '):' + v.primitives.map((p) => p.name).join('|'))
};
```

```json
{
	"ok": true,
	"value": {
		"violationCount": 0,
		"passed": true,
		"violations": []
	}
}
```

::: tip

建议在执行 `save()` 后等待 8 至 10 秒再执行 DRC 检查，以便编辑器完成拓扑计算，从而获得最准确的结果。

:::

### VIII. 导出网表

[`SCH_ManufactureData.getNetlistFile()`](../reference/pro-api.sch_manufacturedata.getnetlistfile) 用于导出网表文件：

```javascript
// step7_netlist.js
const f = await eda.sch_ManufactureData.getNetlistFile('MyBoard_NET', 'Protel2');
const txt = await f.text();
return { name: f.name, size: f.size, nets: txt.split('\n').slice(-24).join('\n') };
```

```json
{
	"ok": true,
	"value": {
		"name": "MyBoard_NET.net",
		"size": 4206,
		"nets": "(\nGND \nR2-2 0603WAF1002T5E-2 Input   \nC1-2 CC0603KRX7R9BB104-2 Passive \nR3-2 0603WAF1001T5E-2 Input   \n)\n(\n$1N2 \nR2-1 …\n)\n(\nVCC \nR3-1 …\nR1-1 …\n)"
	}
}
```

导出结果中的网表与设计意图完全一致，因此连接关系可以通过程序进行断言校验。

### IX. 导出 BOM

[`SCH_ManufactureData.getBomFile()`](../reference/pro-api.sch_manufacturedata.getbomfile) 用于导出 BOM，支持 `csv` 与 `xlsx` 两种格式：

```javascript
// step8_bom.js
const DIR = 'C:/Users/username/Documents/LCEDA-Pro/projects/MyBoard/';

// CSV（文本，可直接读回校验）
const csv = await eda.sch_ManufactureData.getBomFile('BOM', 'csv');
const csvText = await csv.text();
await eda.sys_FileSystem.saveFileToFileSystem(DIR + 'BOM.csv', new Blob([csvText], { type: 'text/csv' }), undefined, true);

// XLSX（二进制，直接落盘）
const xlsx = await eda.sch_ManufactureData.getBomFile('BOM', 'xlsx');
const saved = await eda.sys_FileSystem.saveFileToFileSystem(DIR + 'BOM.xlsx', xlsx, undefined, true);

return { csvText, xlsxSize: xlsx.size, saved };
```

```json
{
	"ok": true,
	"value": {
		"csvText": "No.\tQuantity\tComment\tDesignator\tFootprint\tValue\tManufacturer Part\tManufacturer\tSupplier Part\tSupplier\n1\t1\t100nF\tC1\tC0603\t100nF\tCC0603KRX7R9BB104\tYAGEO(国巨)\tC14663\tLCSC\n2\t2\t1kΩ\tR1,R3\tR0603\t1kΩ\t0603WAF1001T5E\tUNI-ROYAL(厚声)\tC21190\tLCSC\n3\t1\t10kΩ\tR2\tR0603\t10kΩ\t0603WAF1002T5E\tUNI-ROYAL(厚声)\tC25804\tLCSC\n",
		"xlsxSize": 6893,
		"saved": true
	}
}
```

同型号的 R1 与 R3 会被自动归并至同一行，数量统计为 2。

### X. 导出截图

[`DMT_EditorControl.getCurrentRenderedAreaImage()`](../reference/pro-api.dmt_editorcontrol.getcurrentrenderedareaimage) 用于获取当前渲染区域的图像：

```javascript
// step9_shot.js
const tabId = await eda.dmt_EditorControl.openDocument('429e4af1f4a665c3');
await eda.dmt_EditorControl.zoomToAllPrimitives();
await new Promise((r) => setTimeout(r, 2000));

const blob = await eda.dmt_EditorControl.getCurrentRenderedAreaImage(tabId);
await eda.sys_FileSystem.saveFileToFileSystem('C:/Users/username/Documents/LCEDA-Pro/projects/MyBoard/sch.png', blob, undefined, true);
return { saved: true, size: blob.size };
```

若需要导出整张图页，可以使用 [`SCH_ManufactureData.getPngFile()`](../reference/pro-api.sch_manufacturedata.getpngfile)，并指定分辨率：

```javascript
const png = await eda.sch_ManufactureData.getPngFile('MySchematic', { width: 1400 });
await eda.sys_FileSystem.saveFileToFileSystem(DIR + png.name, png, undefined, true);
```

### XI. 保存文档

原理图与 PCB 文档分别通过对应的 `Document` 类保存：

```javascript
await eda.sch_Document.save(); // 原理图
await eda.pcb_Document.save(); // PCB
```

`.eprj3` 是文件夹工程格式，与编辑器之间保持双向同步：编辑器保存后磁盘上的 `.esch2` / `.epcb2` 文件会立即更新；反之，直接修改磁盘上的文件，编辑器也会很快重新读取。

### XII. 设计 PCB

PCB 侧的流程与原理图类似：绘制板框、从原理图导入变更、自动布局、自动布线。

```javascript
// step10_pcb.js
const pcbUuid = '509258b8ebfffb42';

// 1. 打开 PCB
await eda.dmt_EditorControl.openDocument(pcbUuid);
await new Promise((r) => setTimeout(r, 2500));

// 2. 绘制板框：板框层 EPCB_LayerId.BOARD_OUTLINE = 11
//    使用折线图元配合显式顶点，最不容易出错
const poly = eda.pcb_MathPolygon.createPolygon([40, -340, 'L', 400, -340, 400, -120, 40, -120, 40, -340]);
const outline = await eda.pcb_PrimitivePolyline.create('', 11, poly, 10, false);

// 3. 从原理图导入变更（相当于 GUI 内的「设计 - 导入变更」）
await eda.pcb_Document.importChanges();
await new Promise((r) => setTimeout(r, 5000));

// 4. 自动布局
const layout = await eda.pcb_Document.autoLayout();

// 5. 自动布线
const route = await eda.pcb_Document.autoRouting();

// 6. 缩放至板框并截图
await eda.pcb_Document.zoomToBoardOutline();
const tabId = '509258b8ebfffb42@<项目uuid>';
const img = await eda.dmt_EditorControl.getCurrentRenderedAreaImage(tabId);
await eda.sys_FileSystem.saveFileToFileSystem(DIR + 'pcb.png', img, undefined, true);

await eda.pcb_Document.save();
return { outline: outline.getState_PrimitiveId(), layout, route, imgSize: img.size };
```

```json
{
	"ok": true,
	"value": {
		"outline": "24275a39b9fc526f",
		"layout": {
			"success": true,
			"totalComponentsCount": 4,
			"successComponentsCount": 4,
			"failedComponents": [],
			"duration": 974
		},
		"route": {
			"success": true,
			"totalNetsCount": 3,
			"successNetsCount": 3,
			"failedNets": [],
			"duration": 1405
		},
		"imgSize": 124914
	}
}
```

[`PCB_Document.autoRouting()`](../reference/pro-api.pcb_document.autorouting) 支持通过参数指定需要布线的网络：

```javascript
await eda.pcb_Document.autoRouting({ RoutingNets: ['VCC'], optimization: 1 });
```

布线完成后，可以执行 [`PCB_Drc.check()`](../reference/pro-api.pcb_drc.check) 确认连通性：

```javascript
const passed = await eda.pcb_Drc.check(true, false, false); // true 表示全部通过
```

`DMT_Board` `PCB_Drc` `PCB_ManufactureData` 等类中还提供了网络类、差分对、等长组、铺铜、Gerber、坐标文件、IPC-2581、ODB++、交互式 BOM 导出等大量能力，可以通过 `doc api` 逐个查阅。

### XIII. 实践建议

1. 执行 `invoke` 时始终附加 `--ext-uuid eda`，以确保获得完整的权限；
2. 路径统一使用正斜杠，写入前确认目标位于 `APP_PROJECT_DIR` 目录内；
3. 放置器件时遵循「`create()` → 读取属性快照 → `modify(designator + otherProperty)`」的步骤，使位号与属性一次成型；
4. 引脚坐标应通过 `getAllPinsByPrimitiveId()` 读取，不要自行推算；
5. 器件数量较多时，建议分批执行 `invoke`，每批 20 至 25 个，为超时留出余量；
6. 修改完成后等待 5 至 10 秒再查询状态（DRC / 网表 / 渲染），以获取最终结果；
7. 建议将每一步的关键返回值保存为 JSON 留档；
8. 收尾时执行 `save()` 并在磁盘上核对产物；
9. 会话使用完毕后执行 `session close --destroy` 关闭。

## 直接生成工程文件

除通过扩展 API 操作编辑器外，你也可以直接编写 `.eprj3` 工程文件。两种方式可以混用：使用 API 创建工程骨架，再通过文件生成内容。

### I. 工程文件结构

`.eprj3` 是文件夹化工程格式，由一个目录与若干 JSON / 纯文本文件组成，对 Git 与脚本均较为友好：

```
MyBoard/
├── MyBoard.eprj3                 # 工程索引与元数据（创建工程时唯一生成的文件）
├── sch/                          # 原理图，每张原理图一个文件夹
│   └── <原理图名>/
│       ├── <图页名>.esch2         # 图页源码
│       ├── <原理图名>.ecfg        # 该原理图的设计规则与配置
│       └── <原理图名>.evar        # 装配变量
├── pcb/
│   └── <PCB名>.epcb2             # PCB 源码
└── panel/
    └── <面板名>.epan2            # 面板源码
```

文件名即文档名。

### II. 记录格式

`.esch2` `.epcb2` `.epan2` 采用统一的记录格式，每个文件由多行记录组成，每行一条 JSON 记录，其形式如下：

```
{"type":"<类型>","ticket":<逻辑时钟>,"id":"<16位小写hex>"}||{<载荷>}|
```

- 外壳与载荷之间使用 `||` 分隔；
- 行尾必须为 `|` 加换行符（`LF`，而非 `CRLF`）；
- `type` 决定该条记录的含义。

常见类型如下：

| 文件   | 类型                                                                    |
| ------ | ----------------------------------------------------------------------- |
| 原理图 | `DOCHEAD` `META` `COMPONENT` `ATTR` `WIRE` `LINE` `TEXT`                 |
| PCB    | `DOCHEAD` `META` `COMPONENT` `ATTR` `PAD_NET` `NET` `POLY` `LINE` `LAYER` `RULE` |

### III. 多文档结构

`.esch2` 文件并非对应单张图页，而是由一串文档组成，每段以 `DOCHEAD` 开头：

```
DOCHEAD  docType=SYMBOL    uuid=<图框符号>       ← 图框（Drawing-Symbol_A4）
DOCHEAD  docType=SYMBOL    uuid=<器件符号1>      ← 1kΩ 电阻符号
DOCHEAD  docType=SYMBOL    uuid=<器件符号2>
DOCHEAD  docType=FOOTPRINT uuid=<封装1>          ← R0603
DOCHEAD  docType=DEVICE    uuid=<器件1>          ← 仅包含 DOCHEAD 与 META
...
DOCHEAD  docType=BLOB      uuid=BLOB
DOCHEAD  docType=SCH_PAGE  uuid=<图页uuid>       ← 图页的实际内容
```

器件库（符号、封装、器件）内嵌于同一个文件之中，`.eprj3` 不存在独立的工程库，器件均为文件内的独立文档。

书写顺序建议与编辑器实际输出的顺序保持一致：`SYMBOL` → `FOOTPRINT` → `DEVICE` → `BLOB` → `SCH_PAGE`。

### IV. 查询格式规范

使用 `doc format` 可以查询各类设计文件记录的字段定义：

```shell
lceda-pro doc format
lceda-pro doc format --class-name TMSchComponent
lceda-pro doc format --class-name TSchAttr
lceda-pro doc format --class-name TWire
lceda-pro doc format --class-name TSchLine
```

```json
{
	"ok": true,
	"value": {
		"name": "TMSchComponent",
		"comment": "元件（COMPONENT 图元的线格式）",
		"fields": [
			{ "name": "partId", "type": "TPartId", "required": true },
			{ "name": "x", "type": "number", "required": true },
			{ "name": "y", "type": "number", "required": true },
			{ "name": "rotation", "type": "number", "required": true },
			{ "name": "isMirror", "type": "boolean", "required": true },
			{
				"name": "attrs",
				"type": "{ DeviceName: string; Devices: string; FootprintName: string; Footprints: string; SymbolName: string; Symbols: string }",
				"required": true
			},
			{ "name": "zIndex", "type": "null | number", "required": true }
		]
	}
}
```

::: tip

`doc format` 的字段说明较为详尽，写文件前建议逐字段阅读，这比从真机样本反推更为准确。

:::

常用类型对照如下：

| 需要编写的内容             | 对应类型                                                                 |
| -------------------------- | ------------------------------------------------------------------------ |
| 原理图器件实例             | `TMSchComponent`                                                         |
| 原理图属性（位号 / 值 / 网络名等） | `TSchAttr`                                                        |
| 原理图导线组 / 单线段      | `TWire` / `TSchLine`                                                     |
| PCB 器件实例               | `TMPcbComponent`                                                         |
| PCB 属性                   | `TPcbAttr`                                                               |
| PCB 焊盘网络映射           | `TPadNetWire`                                                            |
| PCB 走线 / 板框折线        | `TPcbLine` / `TPcbPoly` + `TPcbBoard`                                    |
| 图元 id 的各种形态         | `TElementId` / `TSingletonElementId` / `TKeyedElementId` / `TCompositeElementId` |
| PART id                    | `TPartId`                                                                |

### V. 编写图页内容

以下示例将前述分压电路直接书写为 `P1.esch2`。

#### 1. 使用 API 创建工程骨架

首先通过 API 创建工程并打开，以获得正确的目录结构，省去手写索引文件的工作：

```shell
lceda-pro open
lceda-pro invoke --session $S --ext-uuid eda --code "$(cat step1_create_proj.js)"
```

此时 `P1.esch2` 内包含 A4 图框的 `SYMBOL` 文档、`DEVICE` 文档、`BLOB` 文档以及空的 `SCH_PAGE` 文档。

#### 2. 采集库文档

符号文档包含 `PART` `PIN` `LINE` `POLY` `ATTR` 等一整套图形定义，手写并不现实，必须来自编辑器的实际导出。

推荐的做法是：先使用 API 放置一遍器件并保存，随后读取磁盘上的 `.esch2` 文件，将其中 `SYMBOL` `FOOTPRINT` `DEVICE` 文档段原样取出备用。

```shell
# 使用 API 放置器件并保存
lceda-pro invoke --session $S --ext-uuid eda --code "$(cat place_and_save.js)"
# 保存后文件立即更新，可直接读取磁盘
```

#### 3. 编写 SCH_PAGE 文档段

`SCH_PAGE` 文档段中只编写业务记录，即器件实例、属性与导线：

```python
# COMPONENT：partId 指向 SYMBOL 文档中 PART 行的 id
#            ATTR 的 Symbol 指向 SYMBOL 文档 uuid，Device 指向 DEVICE 文档 uuid
comp = {
    "type": "COMPONENT", "ticket": tk, "id": cid,
    "body": {
        "partId": "0603WAF1001T5E.1",           # 符号文档中 PART 行的 id
        "x": 400, "y": 610,
        "rotation": 270,                         # 文件角度 = (360 - API 角度) % 360
        "isMirror": False,
        "attrs": {"Footprints": "[]", "Devices": "[]", "DeviceName": None,
                  "FootprintName": None, "pinClass": {}, "differentialPairClass": {},
                  "Symbols": "[]"},
        "zIndex": 4, "yAxisDirection": "up"
    }
}

# ATTR：一行一个键值对，parentId 指回元件
def attr(cid, key, value, x=None, y=None, z=0):
    return {"type": "ATTR", "ticket": tk, "id": new_id(), "body": {
        "x": x, "y": y, "rotation": 0, "color": None, "fontFamily": None,
        "fontSize": None, "fontWeight": None, "italic": None, "underline": None,
        "align": None, "value": value, "keyVisible": None, "valueVisible": None,
        "key": key, "fillColor": None, "parentId": cid, "zIndex": z,
        "yAxisDirection": "up"}}

attrs = [
    attr(cid, "Symbol", "c828663d1111a8cf"),          # → SYMBOL 文档 uuid
    attr(cid, "Device", "e16440afb3ce936d"),          # → DEVICE 文档 uuid
    attr(cid, "Designator", "R1", x=410, y=610, z=2),
    attr(cid, "Name", "={Value}", x=410, y=600, z=30),
    attr(cid, "Manufacturer", "UNI-ROYAL(厚声)"),
    attr(cid, "Manufacturer Part", "0603WAF1001T5E"),
    attr(cid, "Supplier", "LCSC"),
    attr(cid, "Supplier Part", "C21190"),
    attr(cid, "Value", "1kΩ"),
    # 其余属性略
]

# WIRE（导线组）+ LINE（单线段）
wire = {"type": "WIRE", "ticket": tk, "id": wid, "body": {"zIndex": 10}}
line = {"type": "LINE", "ticket": tk, "id": new_id(), "body": {
    "fillColor": None, "fillStyle": None, "strokeColor": None, "strokeStyle": None,
    "strokeWidth": None, "startX": 400, "startY": 630, "endX": 400, "endY": 660,
    "lineGroup": wid, "yAxisDirection": "up"}}
```

编写时需遵循以下三条规则：

1. `partId` 必须是其所引用的 `SYMBOL` 文档中 `PART` 行的 id（形如 `0603WAF1001T5E.1`）；
2. `id` 使用随机的 16 位小写十六进制字符串，且在文档内唯一；
3. `ticket` 在文档内唯一，且组件的 `ticket` 应小于其自身属性的 `ticket`。

#### 4. 校验引用完整性

写入磁盘前，应先进行静态校验：

```python
# 每个 COMPONENT 的 partId 必须能在某个 SYMBOL 文档的 PART 行中找到
part_ids = {rec.id for doc in docs if doc.docType == 'SYMBOL'
            for rec in doc.recs if rec.type == 'PART'}
doc_uuids = {d.uuid for d in docs}

for c in components:
    assert c.partId in part_ids or c.partId.startswith('pid'), c
    a = attrs_of(c)
    assert a['Symbol'] in doc_uuids
    assert a['Device'] in doc_uuids

# 导线：每段必须正交，lineGroup 必须指向已存在的 WIRE
for l in lines:
    assert l.startX == l.endX or l.startY == l.endY
    assert l.lineGroup in wire_ids
```

#### 5. 写入与复核

将文件写入工程目录后，等待编辑器重新加载，再进行复核：

```shell
cp build/P1.esch2 "C:\Users\username\Documents\LCEDA-Pro\projects\MyBoard\sch\Schematic1\P1.esch2"
```

```javascript
// 复核脚本：对象数量 + DRC + 截图
const comps = (await eda.sch_PrimitiveComponent.getAll()).map((c) => c.getState_Designator() || c.getState_ComponentType());
const wires = await eda.sch_PrimitiveWire.getAllPrimitiveId();
const drc = await eda.sch_Drc.check(true, false, false);
await eda.sch_ManufactureData.getPngFile('check', { width: 1400 }); // 顺带出图
return { comps, wireCount: wires.length, drcPassed: drc };
```

```json
{
	"ok": true,
	"value": {
		"comps": ["sheet", "netflag", "netflag", "R2", "R3", "C1", "R1"],
		"wireCount": 3,
		"drcPassed": true
	}
}
```

对象数量符合预期、DRC 通过、截图目视正常，三者均满足方可视为编写完成。

::: tip

手写文件后务必复核对象数量。若发现数量不符，可将文件重新写入一次，编辑器会重新完整加载。

:::

### VI. 编写 PCB 文件

`.epcb2` 相比 `.esch2` 多了一层焊盘网络映射，其结构如下：

```
SYMBOL × N        ← 器件符号（渲染用）
FOOTPRINT × N     ← 封装（焊盘定义）
DEVICE × N        ← 器件
PCB               ← 板子文档
   ├─ 板级样板：LAYER(60) / LAYER_PHYS / RULE / PREFERENCE / PRIMITIVE / PANELIZE / D3_ATTRIBUTE …
   └─ 业务记录：NET / COMPONENT / ATTR / PAD_NET / POLY(板框) / LINE(走线)
```

业务记录的编写方式如下：

```python
# NET：一个网络一条记录
{"type": "NET", "id": '["NET","VCC"]', "body": {"netType": None, "specialColor": None,
  "retLine": True, "differentialName": None, "isPositiveNet": False, "equalLengthGroupName": None}}

# COMPONENT：器件实例（角度使用 PCB 约定）
{"type": "COMPONENT", "id": cid, "body": {
  "partitionId": "", "groupId": 0, "layerId": 1,            # layerId 1 = 顶层
  "x": 307.15, "y": -222.71, "angle": 270,
  "attrs": {"Reuse Block": "", "Group ID": "", "Channel ID": "$1I5", "Unique ID": "gge1",
           "DeviceName": '{"name":"0603WAF1001T5E","source":"","uuid":"<DEVICE uuid>"}'},
  "locked": False, "zIndex": -1, "pinSwap": False,
  "pinSwapInfo": {cid + "e7": {"pinClass": "", "differentialPairClass": ""},
                  cid + "e8": {"pinClass": "", "differentialPairClass": ""}},
  "footprintPrimitives": True}}

# ATTR：复合 id = 元件 id + 局部 id
{"type": "ATTR", "id": cid + "e15", "body": {"parentId": cid, "layerId": 3, "key": "Footprint",
  "value": "<FOOTPRINT uuid>"}}
{"type": "ATTR", "id": cid + "e16", "body": {"parentId": cid, "layerId": 3, "key": "Designator",
  "value": "R1", "x": 341.16, "y": -165.18, "angle": 270}}

# PAD_NET：焊盘到网络的映射（id 为数组串）
{"type": "PAD_NET", "id": '["PAD_NET","<cid>","2","e7"]',
 "body": {"partitionId": "", "padNet": "VCC", "padLen": None, "propagationDelay": None, "attrsMap": {}}}

# POLY：板框（layerId = 11 = BOARD_OUTLINE）
{"type": "POLY", "id": pid, "body": {"partitionId": "", "groupId": 0, "netName": "",
  "layerId": 11, "width": 10, "path": [40, -340, "L", 400, -340, 400, -120, 40, -120, 40, -340],
  "locked": False, "zIndex": -1, "polyType": "NORMAL"}}

# LINE：走线
{"type": "LINE", "id": lid, "body": {"partitionId": "", "groupId": 0, "netName": "VCC",
  "layerId": 1, "startX": 307.15, "startY": -193.05, "endX": 339.49, "endY": -225.39,
  "width": 10, "locked": False, "zIndex": -1}}
```

各类 id 之间的对应关系如下：

- `PAD_NET` 的 id 为 `["PAD_NET", <元件id>, <焊盘号>, <焊盘局部id>]`，其中最后一个字段必须与 `FOOTPRINT` 文档中对应 `PAD` 行的 id 一致；
- `pinSwapInfo` 的键为 `<元件id><焊盘局部id>`；
- `ATTR` 的 id 为 `<元件id><局部id>`，局部 id 沿用编辑器的 `e0` `e15` `e16` 等命名；
- 板框 `POLY` 使用 `layerId: 11`。

编写 PCB 文件的建议如下：

1. 板级样板（`LAYER` `RULE` `PREFERENCE` `PRIMITIVE` 等）沿用编辑器生成的记录，不要手写；
2. 库文档（`SYMBOL` `FOOTPRINT` `DEVICE`）从编辑器导出中采集；
3. 业务记录（`NET` `COMPONENT` `ATTR` `PAD_NET` `POLY` `LINE`）手写，并进行引用完整性校验；
4. 写入后复核器件数量、走线条数、[`PCB_Net.getAllNetsName()`](../reference/pro-api.pcb_net.getallnetsname)、PCB DRC 与渲染截图。

[`PCB_Drc.check()`](../reference/pro-api.pcb_drc.check) 除几何规则外，还会执行原理图与 PCB 之间的网表比对，只要保证 PCB 上的器件与焊盘网络和原理图一致，比对即可通过。

### VII. 实践建议

1. 工程骨架使用 API 创建（`createProject()` + `openProject()`），避免手写索引文件；
2. 库文档从编辑器导出中采集，业务记录手写；
3. 编写文件前先通过 `doc format` 逐字段阅读相关类型；
4. 遵循写盘三条规则：`partId` 指向 `PART` 行、`id` 使用随机 16 位十六进制、`ticket` 在文档内唯一；
5. 行尾使用 `|` 加 `LF`，外壳与载荷之间使用 `||` 分隔；
6. 写入前先进行静态校验（引用完整性、正交性、id 与 ticket 的唯一性），再落盘；
7. 落盘后等待 10 至 15 秒，复核对象数量、DRC 与截图，数量不符时可将文件重新写入一次；
8. `.eprj3` 的双向同步是即时的，编辑器保存后磁盘文件会立即更新，可以直接读回作为标准答案进行比对。

## 两种方式的对比

|              | 通过扩展 API 操作                   | 直接编写 `.eprj3` 文件            |
| ------------ | ----------------------------------- | --------------------------------- |
| 上手难度     | 低                                  | 中（需先了解格式规范）            |
| 适用场景     | 交互式操作、校验、导出、自动化测试  | 批量生成工程、代码评审、Git 管理与 AI 直接产出工程 |
| 库文档       | 编辑器自带，可直接搜索              | 需从导出中采集                    |
| 版本管理     | 二进制 / SQLite 格式不友好          | 纯文本，便于 diff                |

两种方式可以混用，推荐流程如下：

```text
新建工程(API) → 采集库文档(API + 读盘) → 生成业务记录(脚本 / 程序) → 落盘
      ↓                                                                  ↓
   校验(API: DRC / 网表 / BOM / 截图)  ←──────────────────────────  等待重载
```

## 速查表

### 命令

| 目的         | 命令                                                                                          |
| ------------ | --------------------------------------------------------------------------------------------- |
| 查看总览     | `lceda-pro --help`                                                                            |
| 模糊搜索命令 | `lceda-pro --search <关键词>`                                                                 |
| 查看命令用法 | `lceda-pro <command> --help`                                                                  |
| 探测端点     | `lceda-pro doctor`                                                                            |
| 打开窗口     | `lceda-pro open [--path <工程>] [--headless <bool>]`                                          |
| 查看会话     | `lceda-pro session list`                                                                      |
| 关闭会话     | `lceda-pro session close --session <id> --destroy`                                            |
| 列出类       | `lceda-pro doc api`                                                                           |
| 列出方法     | `lceda-pro doc api --class-name <类>`                                                         |
| 查看方法     | `lceda-pro doc api --class-name <类> --method-name <方法>`                                    |
| 查看枚举     | `lceda-pro doc api --class-name <E枚举>`                                                      |
| 查看格式     | `lceda-pro doc format [--class-name <类型>]`                                                  |
| 执行 JS      | `lceda-pro invoke --session <id> --ext-uuid eda --code "…" [--args '<json>'] [--timeout <ms>]` |
| 作为 MCP     | `lceda-pro --mcp stdio` / `--mcp http --port 3030`                                            |

### 常用 API

| 领域         | 类                                                                     | 常用方法                                                                   |
| ------------ | ---------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| 工程         | [DMT_Project](../reference/pro-api.dmt_project)                         | `createProject` `openProject` `getCurrentProjectInfo`                      |
| 板 / 图      | [DMT_Board](../reference/pro-api.dmt_board) [DMT_Schematic](../reference/pro-api.dmt_schematic) [DMT_Pcb](../reference/pro-api.dmt_pcb) | `createBoard` `createSchematic` `getAllBoardsInfo`        |
| 编辑器       | [DMT_EditorControl](../reference/pro-api.dmt_editorcontrol)             | `openDocument` `zoomToAllPrimitives` `getCurrentRenderedAreaImage`         |
| 器件库       | [LIB_LibrariesList](../reference/pro-api.lib_librarieslist) [LIB_Device](../reference/pro-api.lib_device) | `getSystemLibraryUuid` `search`                              |
| 原理图图元   | [SCH_PrimitiveComponent](../reference/pro-api.sch_primitivecomponent)   | `create` `createNetFlag` `modify` `getAllPinsByPrimitiveId`                |
|              | [SCH_PrimitiveWire](../reference/pro-api.sch_primitivewire)             | `create` `getAll`                                                          |
| 原理图检查   | [SCH_Drc](../reference/pro-api.sch_drc) [SCH_Netlist](../reference/pro-api.sch_netlist) [SCH_ManufactureData](../reference/pro-api.sch_manufacturedata) | `check` `getNetlistFile` `getBomFile` `getPngFile` |
| 原理图文档   | [SCH_Document](../reference/pro-api.sch_document)                       | `save` `getPrimitiveAtPoint` `getPrimitivesInRegion`                       |
| PCB 图元     | [PCB_PrimitiveComponent](../reference/pro-api.pcb_primitivecomponent) [PCB_PrimitiveLine](../reference/pro-api.pcb_primitiveline) [PCB_PrimitivePolyline](../reference/pro-api.pcb_primitivepolyline) | `create` 等                                   |
| PCB 文档     | [PCB_Document](../reference/pro-api.pcb_document)                       | `save` `importChanges` `autoLayout` `autoRouting` `zoomToBoardOutline`     |
| PCB 检查     | [PCB_Drc](../reference/pro-api.pcb_drc) [PCB_ManufactureData](../reference/pro-api.pcb_manufacturedata) [PCB_Net](../reference/pro-api.pcb_net) | `check` `getGerberFile` `get3DFile` `getAllNetsName`      |
| 文件         | [SYS_FileSystem](../reference/pro-api.sys_filesystem)                   | `saveFileToFileSystem` `readFileFromFileSystem` `listFilesOfFileSystem`    |
| 环境         | [SYS_Environment](../reference/pro-api.sys_environment)                 | `isClient` `getEditorCurrentVersion` `getUserInfo`                         |

### 坐标系与单位

| 领域   | 约定                                                                                          |
| ------ | --------------------------------------------------------------------------------------------- |
| 原理图 | 单位为 10mil，原点位于图纸左下角，Y 轴向上，A4 尺寸为 1170 × 825                              |
| PCB    | 单位为 mil，`layerId` 为 `1` 表示顶层，`11` 表示板框层                                        |
| 角度   | API 使用逆时针为正，原理图文件内 `COMPONENT.rotation` 为 `(360 - API 角度) % 360`，PCB 侧 `COMPONENT.angle` 与 API 角度同向 |

## 完整脚本示例

以下脚本将上述流程串联为一个可直接运行的完整示例：

```bash
#!/usr/bin/env bash
set -e
L="lceda-pro"

# 1. 建立会话
S=$($L open | python -c "import sys,json;print(json.load(sys.stdin)['value']['sessionId'])")
echo "session = $S"

# 2. 新建工程
$L invoke --session $S --ext-uuid eda --code "$(cat step1_create_proj.js)"

# 3. 放置器件、网络标识与导线
$L invoke --session $S --ext-uuid eda --code "$(cat step3_place.js)"
$L invoke --session $S --ext-uuid eda --code "$(cat step4_flags.js)"
$L invoke --session $S --ext-uuid eda --code "$(cat step5_wire.js)"

# 4. 保存、等待刷新、执行 DRC
$L invoke --session $S --ext-uuid eda --code "await eda.sch_Document.save(); return true;"
sleep 10
$L invoke --session $S --ext-uuid eda --code "$(cat step6_drc.js)"

# 5. 导出网表、BOM 与截图
$L invoke --session $S --ext-uuid eda --code "$(cat step7_netlist.js)"
$L invoke --session $S --ext-uuid eda --code "$(cat step8_bom.js)"
$L invoke --session $S --ext-uuid eda --code "$(cat step9_shot.js)"

# 6. 关闭会话
$L session close --session $S --destroy
```

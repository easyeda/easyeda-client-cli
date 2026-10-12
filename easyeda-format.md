# 嘉立创EDA专业版文件格式 / EasyEDA Pro File Format

**[English](#english)** | **[中文](#中文)**

---

## English

The EasyEDA Pro file format describes design data such as projects, schematics, PCBs, footprints, and symbols. It is organized as a log document: each line consists of outer consistency metadata and inner atomic data, supporting incremental storage, eventual consistency, and version evolution.

Project formats:

- **epro2**: Project archive format of EasyEDA Pro V3 and above, a zip archive.
- **eprj2**: Default project storage format of the offline client — a SQLite file, storing data in the V3 format. Do not modify the file directly without going through the client.
- **eprj**: Project format of offline clients V2 and below, stored as a SQLite file in the legacy V2 format. It is automatically upgraded to eprj2 when opened in the V3 client.
- **eprj3**: The unpacked counterpart of epro2 — the project is split into individual files stored directly on disk, making it easy for Git and third-party tools to read and write. This is the primary format going forward.

### V3 Format Key Points

- All project data is stored in **log documents**. The first record of each document is a `DOCTYPE` header that identifies the document type (e.g. `SCH_PAGE`, `PCB`, `SYMBOL`, `FOOTPRINT`, `DEVICE`), as a block-level element in the project log.
- Each line consists of **two JSON objects** concatenated by the `||` delimiter: the outer object `{ "type", "id", "ticket" }` is used by the eventual consistency framework, and the inner object is the atomic key-value data:

  ```json
  { "type": "TYPE", "id": "UUID", "ticket": 1 }||{ "key": "value" }|
  ```

- **Eventual consistency**: create/update/delete operations all append records to the log. When `type` and `id` are the same, the record with the larger `ticket` is kept; when `ticket` is also the same, the record with the smaller `client` value in the document header is kept.
- **Deletion** sets the inner data to an empty string (`||""`), or appends a `DELETE_DOC` marker for a whole document. Deleted records are retained in the log (cloning the project removes them).
- Keys use camelCase; all primitives carry a unique ID within the document; rotation is counter-clockwise positive in degrees; coordinates/lengths use `10 mil` (`0.01 inch`) in schematic documents (e.g. `SCH_PAGE`, `SYMBOL`) and `mil` in PCB documents (e.g. `PCB`, `FOOTPRINT`) unless otherwise specified; colors are `"#RRGGBB"` (`""` = fully transparent); boolean properties use `1`/`0`.

### Documentation

- Latest file format specification: [easyeda-format-skill](https://github.com/easyeda/easyeda-format-skill)
- eprj3 folder-based project format: [easyeda-eprj3-skill](https://github.com/easyeda/easyeda-eprj3-skill)

---

## 中文

嘉立创EDA专业版文件格式用于描述工程、原理图、PCB、封装、符号等设计数据。该格式以日志文档的形式组织：每一行由外层一致性元数据和内层原子数据组成，支持增量存储、最终一致性和版本演进。

工程格式：

- **epro2**：嘉立创EDA专业版 V3 及以上版本的工程归档格式，是一个 zip 压缩包。
- **eprj2**：离线客户端默认的工程存储格式，SQLite 文件，使用 V3 格式存储数据。请勿越过客户端直接修改文件内容。
- **eprj**：V2 及以下离线客户端工程格式，使用旧版 V2 格式的 SQLite 文件格式存储。在 V3 客户端打开时将自动升级到 eprj2。
- **eprj3**：epro2 的拆分（解包）格式，将 zip 包拆分为独立的文件直接存储在磁盘上，便于 Git 管理和第三方工具读写。后续将主推该格式。

### V3 格式关键说明

- 工程的所有数据存储在**日志文档**里。每个文档的首条记录是 `DOCTYPE` 文件头，标识文档类型（如 `SCH_PAGE`、`PCB`、`SYMBOL`、`FOOTPRINT`、`DEVICE`），作为块级元素存在于工程日志中。
- 每一行由 `||` 分隔符拼接的**两个 JSON 对象**组成：外层 `{ "type", "id", "ticket" }` 用于最终一致性框架，内层是原子 key-value 数据：

  ```json
  { "type": "TYPE", "id": "UUID", "ticket": 1 }||{ "key": "value" }|
  ```

- **最终一致性**：数据的增删改都是往日志里追加记录。`type` 和 `id` 相同时保留 `ticket` 更大的记录；`ticket` 也相同时，保留文档头 `client` 更小的记录。
- **删除机制**：删除是将内层数据置为空字符串（`||""`），整文档删除则追加 `DELETE_DOC` 标识。已删除的记录会保留在日志里（克隆工程可去除）。
- 键名采用驼峰命名；所有图元携带文件内唯一编号；旋转角度逆时针为正、角度制；坐标和长度单位为：原理图文档（如 `SCH_PAGE`、`SYMBOL`）使用 `10 mil`（`0.01 inch`），PCB 文档（如 `PCB`、`FOOTPRINT`）使用 `mil`；颜色使用 `"#RRGGBB"`（`""` 表示全透明）；布尔属性使用 `1`/`0`。

### 相关文档

- 最新文件格式说明：[easyeda-format-skill](https://github.com/easyeda/easyeda-format-skill)
- eprj3 文件夹工程格式：[easyeda-eprj3-skill](https://github.com/easyeda/easyeda-eprj3-skill)


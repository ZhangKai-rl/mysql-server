# JSON 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 JSON 数据类型、二进制存储格式（JSONB）、DOM 内存表示、JSON Path、函数体系（含实现算法）、JSON_TABLE、索引方案（生成列/函数索引/多值索引）、比较排序、部分更新全链路、崩溃恢复、以及一条 UPDATE 的全链路串联。
>
> 相关独立文档：[`innodb/lob.md`](../../innodb/lob.md)（LOB 物理层）、[`server/gis.md`](gis.md)（GeoJSON 与 JSON 的边界）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [一、JSON 类型的定位与字段实现](#一json-类型的定位与字段实现)
- [二、文本解析与 DOM 内存表示](#二文本解析与-dom-内存表示)
- [三、二进制存储格式 JSONB](#三二进制存储格式-jsonb)
- [四、Json_wrapper：DOM 与二进制的统一视图](#四json_wrapperdom-与二进制的统一视图)
- [五、JSON Path](#五json-path)
- [六、JSON 函数体系](#六json-函数体系)
- [七、JSON_TABLE](#七json_table)
- [八、JSON 的索引方案](#八json-的索引方案)
- [九、比较、排序与类型转换](#九比较排序与类型转换)
- [十、部分更新（Partial Update）全链路](#十部分更新partial-update全链路)
- [十一、InnoDB 存储层：JSON 就是 BLOB](#十一innodb-存储层json-就是-blob)
- [十二、redo 与崩溃恢复中的 JSON / LOB](#十二redo-与崩溃恢复中的-json--lob)
- [十三、全链路串联：一条 UPDATE 的一生](#十三全链路串联一条-update-的一生)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

MySQL 的 `JSON` 是一种**原生 SQL 数据类型**（`MYSQL_TYPE_JSON`），语义遵循 RFC 7159 / ECMA-404。它不是 `TEXT` 的别名：写入时由解析器（rapidjson）解析成 DOM，再序列化为一种**可随机访问的二进制格式（JSONB）**落盘；读取时优先在二进制上直接导航，只在必要时才物化 DOM。

```
文本 JSON ──rapidjson──> Json_dom（内存树）──json_binary::serialize──> JSONB（磁盘/内存二进制）
                              ▲                                              │
                              └──────────── Json_dom::parse ─────────────────┘
```

### 用途

解决"半结构化数据"的三种诉求：

1. **灵活 schema**：字段可增删，无需 DDL；适合稀疏属性、配置、日志扩展字段。
2. **高效元素访问**：文本 JSON 每次读一个 key 都要全量解析；JSONB 把 object 拆成"定长目录 + 变长数据区"，可 O(log n) 二分定位 key。
3. **就地修改**：大文档的局部修改不必重写整个文档（部分更新 + LOB 部分写），这是 MySQL JSON 相对 MongoDB/PostgreSQL 最独特的工程点。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7.6 | **生成列（generated column）** —— 比 JSON 类型早两个小版本，是后来 JSON 索引方案的地基（见 8.1） |
| 5.7.8 | 引入原生 JSON 类型、JSONB 二进制格式、首批 JSON 函数（`JSON_OBJECT`、`JSON_ARRAY`、`JSON_MERGE`、`JSON_CONTAINS`、`JSON_INSERT/SET/REPLACE/REMOVE`、`JSON_QUOTE/UNQUOTE` 等） |
| 5.7.9 | `->` 操作符（`JSON_EXTRACT` 同义词） |
| 5.7.13 | `->>` 操作符（unquoting extract） |
| 5.7.22 | `JSON_PRETTY`、`JSON_STORAGE_SIZE`、`JSON_STORAGE_FREE`、聚合函数 `JSON_ARRAYAGG` / `JSON_OBJECTAGG` |
| 8.0.3 | **JSON 部分更新**（server 层 diff + InnoDB LOB 部分写）、新增 `binlog_row_value_options=PARTIAL_JSON` |
| 8.0.4 | `JSON_TABLE`（把 JSON 展开成关系表） |
| 8.0.12 | LOB 小改动优化（WL#11328）：改动 ≤100 字节时用 undo 记录 Lob diff 并原地改页，官方称 TPS 最高约 3 倍提升 |
| **8.0.13** | **函数索引**（functional key parts）—— 不必手写生成列即可索引 JSON 表达式，本质是隐藏的 VIRTUAL 生成列（见 8.2） |
| 8.0.17 | **多值索引**（multi-valued index）、`MEMBER OF`、`JSON_SCHEMA_VALID` / `JSON_SCHEMA_VALIDATION_REPORT`、`JSON_OVERLAPS` |
| 8.0.21 | `JSON_VALUE`（EXTRACT + UNQUOTE + CAST 三合一） |
| 8.3 | **与 JSON 数据类型无关**：新增 `explain_json_format_version`（`EXPLAIN FORMAT=JSON` 输出格式 v1/v2，WL#15684）+ 两个 JSON bug 修复（`NULLIF/COALESCE` 错误处理；`JOIN`/`GROUP BY` 对 JSON 值处理不一致）。已核对 8.3.0 release notes：**无 JSONB 格式变更**；8.0 的 JSONB 在这条版本线上**始终没有版本号字段** |
| 9.7 | JSON Duality Views 支持完整 DML（见 cloud/ 与版本谱系笔记） |

---

## 理论基础

### 设计模式

| 模式 | 在代码中的体现 |
|------|----------------|
| **Wrapper / 适配器 + 惰性物化** | `Json_wrapper`（`json_dom.h:1161`）用 union 包裹 DOM 指针或 `json_binary::Value`，`to_dom()`（`:1287`）惰性把二进制转成 DOM。绝大多数读路径（取一个 key、取一个数组元素）**不建 DOM** |
| **访问者 / SAX Handler** | 文本解析用 rapidjson 的 Handler 概念（`json_dom.cc:353-558` 匿名 namespace 内的 handler），边解析边建 DOM，不产生中间文本表示 |
| **组合模式（Composite）** | `Json_dom` → `Json_scalar` / `Json_container` → `Json_object` / `Json_array` 的树形结构（`json_dom.h:151-172`） |
| **享元 / 别名（Alias）** | `Json_wrapper::set_alias()`（`:1219`）标记"不拥有 DOM"，避免拷贝 |
| **迭代器** | `Json_wrapper_object_iterator`（`:1780`）屏蔽 DOM（std::map 迭代器）与 binary（下标）两种遍历方式 |
| **策略 / 双分派** | `Json_wrapper::lookup()`（`json_dom.cc:1768`）在 DOM 与 binary 两条路径上分派查找 |
| **影子副本 + 增量登记（Shadow Copy + Diff）** | 部分更新：在 `record[0]`（新行）上就地改，同时把改动区间登记为 `Binary_diff`（`table.h:1319`），交给引擎做增量落盘 |
| **写时复制（COW）** | LOB 大改动时新分配数据页 + 新 index entry，旧版本靠 `lob_version` 保留给 MVCC（`lob0update.cc:268`） |

### 相关论文

- **RFC 7159 / ECMA-404**：JSON 文本格式标准，MySQL 的输入语法依据。
- **RFC 7396**：JSON Merge Patch —— `Json_object::merge_patch()`（`json_dom.h:514`，实现 `json_dom.cc:885`）即 `JSON_MERGE_PATCH` 的规范实现（`JSON_MERGE_PRESERVE` 是 MySQL 自有的、RFC 7386 的"补丁不删值"语义）。
- **JSON Schema Draft 4**：`JSON_SCHEMA_VALID` 基于 rapidjson 的 schema validator（只支持 draft 4）。
- **SQL/JSON 标准（SQL:2016 Part 6）**：`JSON_TABLE`、`JSON_EXTRACT`、`JSON_VALUE`、`JSON_EXISTS` 的名称与语义来源，但 MySQL 的 **path 语法是 ECMAScript 风格，不是 SQL/JSON 的 lax/strict 模式语法**，这是 MySQL 与标准最大的分歧点。

### 算法与数据结构

| 结构/算法 | 位置 | 说明 |
|-----------|------|------|
| **有序数组 + 二分查找** | `Value::lookup_index`（`json_binary.cc:1183`） | object 的 key entry 定长且按 **(长度, 字节序)** 排序 → 先比长度再 `memcmp`，O(log n) |
| **std::map（红黑树）** | `Json_object_map`（`json_dom.h:365`） | DOM 侧 object 容器，`Malloc_allocator` 分配；比较器 `Json_key_comparator` 同样"先长度后 memcmp"（`json_dom.cc:926`） |
| **变长整数（LEB128 变体，7bit + 0x80 续行）** | `append_variable_length`（`json_binary.cc:252`）/ `read_variable_length`（`:285`） | 存字符串/opaque 长度，最多 5 字节 |
| **显式栈替代递归** | `Json_dom::parse`（`json_dom.cc:637`） | 用 `Prealloced_array`（16 深度内联）做 DFS，避免深文档爆栈 |
| **Small/Large 自动升级重试** | `serialize_json_value`（`json_binary.cc:710`） | 先按 small 序列化，`VALUE_TOO_BIG` 则回退缓冲、改 type 字节重试 large；父节点也随之长大 |
| **内联（inline）小标量** | `inlined_type`（`json_binary.cc:398`）/ `attempt_inline_value`（`:451`） | literal / int16 / uint16（large 下还可 int32/uint32）直接塞进 value entry 的 offset 字段，不占数据区 |
| **空洞复用（hole reuse）** | `Value::has_space`（`json_binary.cc:1410`） | 部分更新时寻找 value 区里足够大的空洞；找不到就放弃部分更新 |

**复杂度**：

- 取 object 成员：binary O(log n)（不建 DOM）；DOM O(log n)。
- 取 array 元素：O(1)（定长 value entry 直接算偏移）。
- 全文档扫描：无 DOM 化开销；`JSON_EXTRACT` 单路径是 O(path 深度 × log n)，不是 O(文档大小)。
- 部分更新：O(路径深度 + diff 字节数)，**与文档总长无关**。

### 类似实现对比

| 系统 | 格式 | 关键差异 |
|------|------|----------|
| **PostgreSQL `jsonb`** | JSONB（与 MySQL 完全不同的自研格式） | PG 的 jsonb 是每个 value 一个 `JsonbValue` + 4 字节长度头的**线性流式布局**，object 内部也是有序但**无目录区**，查找是线性/二分混合；支持 GIN 索引（整文档倒排），MySQL 无对应能力。PG 无"部分更新"能力（改一个字段要重写整行整列） |
| **MongoDB BSON** | BSON | 也是长度前缀 + 线性扫描，无随机访问目录；BSON 有 `$set` 的部分更新但粒度仍是文档级重写 |
| **MySQL JSONB** | 自研 | 最大特色是**目录区（key entry / value entry）与数据区分离**，天然支持：①O(log n) 取 key；②**原地部分更新**（目录区定长，改一个 value 只动数据区若干字节 + 一个 entry）；③`JSON_STORAGE_FREE` 能报出文档里"浪费的空间" |
| **Oracle** | OSON | 类似思路（二进制 + 目录），但闭源 |

### 历史背景

MySQL 5.7 之前 JSON 只能存 TEXT，任何访问都要 `LIKE`/正则/应用层解析。5.7 引入原生类型的核心动机是"**让 JSON 成为一等公民，而不是字符串**"，为此必须解决两个工程问题：

1. **读放大**：文本 JSON 取一个字段要全量解析 → 必须设计可随机访问的二进制格式（这是 JSONB 目录区设计的根本原因，也是 PostgreSQL jsonb 与 MySQL JSONB 共同的出发点）。
2. **写放大**：大 JSON 改一个字段要重写整列（整行 + 整 LOB）→ 必然推导出"部分更新"。而部分更新要成立，又反过来要求：
   - 二进制格式里 object/array 的**头必须定长**（否则改一个值要挪动整个头部）；
   - 引擎必须支持 **LOB 的部分写**（8.0 LOB 重构，引入 first page + data page + index entry + lob_version，即 WL#11328 之前的那次 LOB 重写）；
   - undo 不能记全量 → 小改动时记 Lob diff，大改动时靠版本化保留旧页。

这三条约束在源码里都能直接看到：定长头（`KEY_ENTRY_SIZE_SMALL=4` 等常量）、`lob::update()`、`trx_undo_report_blob_update()`。

---

## 一、JSON 类型的定位与字段实现

### 1.1 Field_json

`sql/field.h:3986` —— **继承自 `Field_blob`**，底层字符集 `my_charset_bin`：

```cpp
class Field_json : public Field_blob {
 public:
  Field_json(uchar *ptr_arg, ...)
      : Field_blob(ptr_arg, ..., &my_charset_bin) {}
```

| 成员 | 行号 | 说明 |
|------|------|------|
| `type()` → `MYSQL_TYPE_JSON` | `field.h:4002` | |
| `charset()` → `&my_charset_utf8mb4_bin` | `field.h:4009` | **对外伪装成 utf8mb4**，让字符串函数按非二进制处理 |
| `sort_charset()` | `field.h:4016` | 排序仍用二进制 |
| `has_charset()` → `false` | `field.h:4022` | SHOW CREATE TABLE 不附加 CHARACTER SET |
| `pack_diff` / `unpack_diff` | `field.h:4032` / `:4070` | 部分更新专用（实现 `field.cc:7858` / `:7983`） |
| `make_sort_key` / `make_hash_key` | `field.cc:8046` / `:8054` | ORDER BY / DISTINCT |
| `cmp_binary` | `field.cc:8035` | 按 JSON 语义比较，非字节比较 |

pack length 与 LONGBLOB 相同：`4 + portable_sizeof_char_ptr`（`field.cc:9241`）。

### 1.2 写入路径与校验

`Field_json::store` —— `sql/field.cc:7633`：

```cpp
type_conversion_status Field_json::store(const char *from, size_t length,
                                         const CHARSET_INFO *cs) {
  reset();
  String v(from, length, cs);
  if (ensure_utf8mb4(v, &value, &s, &ss, true)) return TYPE_ERR_BAD_VALUE;  // ① 转 utf8mb4
  std::unique_ptr<Json_dom> dom(Json_dom::parse(                            // ② 解析成 DOM
      s, ss,
      [..](const char *parse_err, size_t err_offset) {
        my_error(ER_INVALID_JSON_TEXT, MYF(0), parse_err, err_offset, ...); // ③ 非法文本报错
      },
      JsonDocumentDefaultDepthHandler));
  if (dom.get() == nullptr) return TYPE_ERR_BAD_VALUE;
  if (json_binary::serialize(current_thd, dom.get(), &value)) return TYPE_ERR_BAD_VALUE; // ④ 序列化
  return store_binary(value.ptr(), value.length());                          // ⑤ 落入 Field_blob
}
```

重写 `store()` 而不是 `store_internal()` 的原因（`field.cc:7616-7626`）：`Field_blob::store()` 有绕过 `store_internal()` 的分支（空串、GROUP_CONCAT 的 ORDER BY/DISTINCT），必须在此拦截保证语法校验。

非字符串类型（int/decimal/time）一律拒绝 → `unsupported_conversion()`（`field.cc:7681`）→ `ER_INVALID_JSON_TEXT` "not a JSON text, may need CAST"。

### 1.3 长度限制（三层）

| 层 | 限制 | 位置 |
|----|------|------|
| 硬上限 | > `UINT_MAX32`（4GB）→ `ER_JSON_VALUE_TOO_BIG` | `field.cc:7699`、`json_binary.cc:232` |
| 软上限 | 序列化后长度 > `max_allowed_packet` → `ER_WARN_ALLOWED_PACKET_OVERFLOWED` | `json_binary.cc:857` |
| key 长度 | object 的 key > 65535 → `ER_JSON_KEY_TOO_BIG` | `json_binary.cc:373` |

### 1.4 字符集

`ensure_utf8mb4`（`item_json_func.cc:81`）：

- `my_charset_bin` 输入 → `ER_INVALID_JSON_CHARSET`（不能用二进制串构造 JSON）；
- utf8mb4 / utf8mb3 / ascii → 直接使用，零拷贝；
- 其它字符集 → 转换到 `my_charset_utf8mb4_bin`。

错误码体系：

| 错误码 | 场景 |
|--------|------|
| `ER_INVALID_JSON_TEXT` | 写入 JSON **列**解析失败 |
| `ER_INVALID_JSON_TEXT_IN_PARAM` | **函数参数**解析失败 |
| `ER_INVALID_JSON_CHARSET` | 二进制串构造 JSON |
| `ER_INVALID_JSON_BINARY_DATA` | 二进制表示损坏 |
| `ER_JSON_VALUE_TOO_BIG` | >4GB |
| `ER_JSON_USED_AS_KEY` | JSON 列直接建索引（`sql_table.cc:4927`） |

---

## 二、文本解析与 DOM 内存表示

### 2.1 类层次

`sql-common/json_dom.h:151-172` 的官方层次图：

```
Json_dom (abstract)
 Json_scalar (abstract)
   Json_string
   Json_number (abstract)
     Json_decimal / Json_int / Json_uint / Json_double
   Json_boolean / Json_null / Json_datetime / Json_opaque
 Json_container (abstract)
   Json_object / Json_array
```

关键类行号（`json_dom.h`）：`Json_dom`:173、`Json_container`:330、`Json_object`:373、`Json_array`:520、`Json_scalar`:684、`Json_string`:694、`Json_number`:729、`Json_decimal`:737、`Json_double`:827、`Json_int`:850、`Json_uint`:882、`Json_null`:914、`Json_datetime`:926、`Json_opaque`:1003、`Json_boolean`:1045。

> 注意：`Json_datetime` 一个类表示 DATE/TIME/DATETIME/TIMESTAMP 四种（`m_field_type` 区分，`json_dom.h:929`），`PACKED_SIZE = 8`（`:991`）。

### 2.2 Json_object 的容器与比较器

`sql-common/json_dom.h:365`：

```cpp
using Json_object_map =
    std::map<std::string, Json_dom_ptr, Json_key_comparator,
             Malloc_allocator<std::pair<const std::string, Json_dom_ptr>>>;
```

比较器 `Json_key_comparator`（`json_dom.h:343`）—— **先比长度，再 memcmp**（`json_dom.cc:926`）：

```cpp
static bool json_key_less(const char *key1, size_t length1,
                          const char *key2, size_t length2) {
  if (length1 != length2) return length1 < length2;
  return memcmp(key1, key2, length1) < 0;
}
```

- `is_transparent = void` → `find()` 可直接接受 `MYSQL_LEX_CSTRING`，不构造临时 `std::string`。
- 复杂度 O(log n) 次比较；"先比长度"让绝大多数比较只需一次长度比较，避免整串 memcmp。
- DOM 与 binary 使用**同一套 key 排序**，两边查找逻辑同构，转换时 key 顺序天然一致。

### 2.3 内存管理

- `Json_dom` 重载类级 `operator new/delete`，走 `my_malloc(key_memory_JSON)` / `my_free`（`json_dom.cc:122-142`）——**不用 MEM_ROOT**，因为 DOM 可能长期存活（如 prepared statement、缓存的常量）。
- 子容器用 `Malloc_allocator`（同为 `key_memory_JSON`）：`Json_object` 构造 `json_dom.cc:349`、`Json_array` 构造 `:965`。
- 所有权：`using Json_dom_ptr = std::unique_ptr<Json_dom>`（`json_dom.h:64`），**独占语义**；`Json_dom::m_parent` 是裸反向指针，不参与生命周期。
- 深拷贝：虚 `clone()`（`json_dom.h:252`），`Json_object::clone()`（`json_dom.cc:873`）递归 `add_clone`。
- 对比：`Json_path` **用 MEM_ROOT**（`json_path.h:363`），因为 path leg 短生命周期、可整体 `ClearForReuse()`。

### 2.4 DOM ↔ binary

| 方向 | 入口 | 位置 |
|------|------|------|
| 文本 → DOM | `Json_dom::parse(text, len, err_handler, depth_handler)` | `json_dom.cc:562`（内部用 rapidjson handler，`json_dom.cc:353-558`） |
| DOM → binary | `json_binary::serialize(thd, dom, dest)` | `json_binary.cc:132` |
| binary → DOM | `Json_dom::parse(const json_binary::Value&)` | `json_dom.cc:637`（显式栈 DFS） |
| wrapper 惰性 | `Json_wrapper::to_dom()` / `to_binary()` | `json_dom.cc:1387` / `:1408` |

---

## 三、二进制存储格式 JSONB

### 3.0 先搞清楚：JSONB 属于哪一层？是"物理文件格式"吗？

**结论：JSONB 是 server 层定义的「数据编码格式」，不是 InnoDB 的存储格式，也不是独立的文件。**

MySQL 是分层架构，JSON 的语义全部在上面一层：

```
┌──────────────────────────────────────────────┐
│ server 层（sql/、sql-common/）                │
│  · 解析 SQL、执行 JSON 函数                   │
│  · 定义 JSON 类型的语义                       │
│  · ★ 定义 JSONB 二进制格式（本章要讲的）      │
│  · Field_json / Json_dom / 序列化反序列化     │
└─────────────────┬────────────────────────────┘
                  │ handler 接口：存/取"一行"
┌─────────────────▼────────────────────────────┐
│ 存储引擎层（storage/innobase/）               │
│  · 只知道：这一列是 BLOB，长度 N 字节         │
│  · 把字节流放进页（大了就 off-page 到 LOB）   │
│  · ★ 完全不知道 JSONB 是什么                  │
└──────────────────────────────────────────────┘
```

三条硬证据：

1. **目录位置**：格式定义在 `sql-common/json_binary.h` / `.cc` —— `sql-common` 是"server 与 client 共用"的代码，**不在 `storage/` 下**。
2. **InnoDB 侧只有一行**：`ha_innodb.cc:7996` `case MYSQL_TYPE_JSON: return (DATA_BLOB);`，注释直接写着 `// JSON fields are stored as BLOBs`。整个 InnoDB 没有一行 JSON 解析代码。
3. **分工**：server 决定"这段字节表示什么 JSON"，InnoDB 决定"这段字节放在哪个页"。

**那它到底是不是"物理文件"？**

- JSONB 的内容**最终确实会被写进 `.ibd` 文件**（作为 BLOB 列的值，超限则进 off-page LOB 页，见 `innodb/lob.md`）
- 但它是**逻辑/编码格式**，不是 InnoDB 定义的存储结构

类比一下就清楚了：

| | 类比 | 谁定义 | 描述什么 |
|---|------|--------|----------|
| **JSONB** | Word 文档的文件格式 | server 层 | 一个 JSON 文档怎么编码成字节 |
| **InnoDB 页格式** | 文件系统的块/扇区 | InnoDB | 一个 16KB 页怎么组织（页头、行目录槽、记录） |

Word 不关心数据落在哪个磁盘块，磁盘也不认识 Word 文档——**JSONB 和 InnoDB 页格式就是这种关系**。

顺带对比：**FTS 恰恰相反**（见 [`innodb/fts.md`](../../innodb/fts.md)），它是 InnoDB 自己维护的倒排索引，由 11 张辅助表承载，InnoDB **完全知道**里面每个字节的含义。这就是"JSON 当 BLOB 存"和"FTS 自建索引"的本质差别。

格式文法注释在 `sql-common/json_binary.h:59-141`：

```
doc    ::= type value
object ::= element-count size key-entry* value-entry* key* value*
array  ::= element-count size value-entry* value*
```

### 3.1 类型字节（type tag）

`json_binary.cc:54-71`：

| tag | 含义 | tag | 含义 |
|-----|------|-----|------|
| `0x0` | small object | `0x8` | uint32 |
| `0x1` | large object | `0x9` | int64 |
| `0x2` | small array  | `0xA` | uint64 |
| `0x3` | large array  | `0xB` | double |
| `0x4` | literal（null/true/false） | `0xC` | string |
| `0x5` | int16 | `0xF` | opaque |
| `0x6` | uint16 | `0xD`/`0xE` | **保留未用** |
| `0x7` | int32 | | |

> 8.0 的 JSONB **没有版本字节**：`serialize` 只预留 1 字节 type（`json_binary.cc:138`），`parse_binary` 直接把 `data[0]` 当 tag（`json_binary.cc:1080`）。tag 空间只剩 0xD/0xE，这是未来引入版本化的唯一入口（本树与 8.3.0 release notes 均未见版本化改动）。

> **易混淆**：large/small 的区分写在上表的 **tag 最低位**（0x0 vs 0x1），**不是** `0x80` 标记位。`0x80` 是变长长度的**续行位**（`json_binary.cc:263` 写、`:297` 读）。

### 3.2 Object 布局

```
[type][count:2|4][size:2|4][key-entry × N][value-entry × N][key data × N][value data × N]
```

- key entry：`offset(2|4) + length(2)` → small 4 字节 / large 6 字节；
- value entry：`type(1) + offset(2|4)` → small 3 字节 / large 5 字节；
  常量见 `json_binary.cc:77-96`。
- **为什么要分离目录与数据**：①定长 entry → `element(pos)` 直接乘算偏移，O(1) 随机访问；②定长 + 有序 → `lookup_index` 二分 O(log n)；③值区可被**原地更新**（`has_space` / `update_in_shadow` 依赖固定大小头）。

序列化：`serialize_json_object`（`json_binary.cc:564`）、`append_key_entries`（`:350`）。
布局校验：`Value::is_valid()`（`json_binary.cc:876`）会验证 key 有序。

### 3.3 标量编码

| 类型 | 编码 | 位置（写 / 读） |
|------|------|-----------------|
| literal | tag 0x4 + 1 字节（0=null,1=true,2=false） | `:819` / `:922` |
| int16/32/64、uint16/32/64 | 小端定长 | `:773-808` / `:934` |
| double | `float8store` 8 字节（平台无关） | `:809` / `:952` |
| string | varint 长度 + utf8mb4 数据 | `:763` / `:956` |
| opaque | **1 字节 `enum_field_types`** + varint 长度 + 原始数据 | `:644` / `:964` |
| DECIMAL | 包装成 opaque(`MYSQL_TYPE_NEWDECIMAL`)，≤67 字节 | `serialize_decimal :663` |
| DATE/TIME/DATETIME/TIMESTAMP | 包装成 opaque，8 字节 packed | `serialize_datetime :681` |

**opaque 与 string 的唯一实质区别**：opaque 多 1 字节 `field_type`，且不参与字符集校验。

**inline 优化**：literal / int16 / uint16（large 下还有 int32/uint32）可直接塞进 value entry 的 offset 字段，不占数据区 —— `inlined_type`（`:398`）、`attempt_inline_value`（`:451`）；读取端 `Value::element`（`:1101`）。

### 3.4 逐字节实例：`{"a":1,"b":"x"}` 落盘长什么样

前面都是常量表，这里给一个能自己验算的完整例子。先看两条最容易搞错的规则：

1. **doc = 1 字节 type + payload**，`size` 字段记的是 **payload 的字节数**（不含 type 字节）；
2. **所有 offset 都相对 payload 起点**（即跳过第 1 个 type 字节）——因为 `parse_binary` 传的是 `data + 1`（`json_binary.cc:1080`），`Value::m_data` 指向 payload 起点；
3. 整数字段是**小端**。

**例子**：

```sql
SELECT JSON_STORAGE_SIZE('{"a":1,"b":"x"}');   -- 23
```

**序列化推演**（对应 `serialize_json_object`，`json_binary.cc:564`）：

| 步骤 | 动作 |
|------|------|
| ① | DOM 里 key 排序：长度都是 1，`memcmp("a","b")<0` → 顺序 `a`、`b` |
| ② | 写 `type = 0x00`（small object） |
| ③ | 写 `count = 2`，`size` 先占 4 字节坑（`:585` 记录 `size_pos`） |
| ④ | 算 `first_key_offset = 18`（= 2+2 + 2×4 + 2×3，即 count+size+key entry×2+value entry×2） |
| ⑤ | 写 2 条 key entry（`offset`、`length`） |
| ⑥ | 给 2 条 value entry 占位 6 字节（`:607`） |
| ⑦ | 写 key 数据 `a`、`b`（payload 偏移 18、19） |
| ⑧ | 写 value 数据，并回填对应 entry（`:616-627`） |
| ⑨ | 回填 `size = 22`（`:630-632`） |

**最终 23 个字节**：

```
绝对  payload  字节              含义
────  ───────  ────────────────  ─────────────────────────────────────────
0     -        00                type = 0x00   small object
1     0        02 00             element-count = 2
3     2        16 00             size = 22     (payload 字节数)
5     4        12 00 01 00       key entry 0 : offset=18  length=1   → "a"
9     8        13 00 01 00       key entry 1 : offset=19  length=1   → "b"
13    12       05 01 00          value entry 0: type=0x05(int16)  **inlined value = 1**
16    15       0c 14 00          value entry 1: type=0x0c(string) offset=20
19    18       61                key   "a"
20    19       62                key   "b"
21    20       01 78             value: data-length=1, "x"
```

三个必须注意的点：

- **值 `1` 完全不占数据区** —— 它被 inline 进了 value entry（`05 01 00`）。这是 small 格式省空间的关键；`attempt_inline_value`（`:451`）只 inline literal / int16 / uint16（large 下还有 int32/uint32）。
- **string 的 type 字节在 entry 里，数据区只有 `[长度][数据]`** —— 因为 `serialize_json_object` 调 `serialize_json_value` 时传的 `type_pos = entry_pos`（`:623`），type 被写回 entry，数据追加到末尾。
- **`1` 的类型是 `0x05`(int16) 而不是 `0x06`(uint16)** —— 因为 rapidjson 的 `Uint(unsigned)` 回调构造的也是 **`Json_int`**（`json_dom.cc:462`），只有 `Uint64` 才构造 `Json_uint`（`:472`）。

**第二个例子，数组 `[1,"x"]`**（13 字节）：

```
绝对  payload  字节              含义
────  ───────  ────────────────  ─────────────────────────────────────────
0     -        02                type = 0x02   small array
1     0        02 00             element-count = 2
3     2        0c 00             size = 12
5     4        05 01 00          value entry 0: int16, inlined = 1
8     7        0c 0a 00          value entry 1: string, offset = 10
11    10       01 78             value: data-length=1, "x"
```

数组没有 key entry，header 就是 `count + size + value-entry × N`。

**small vs large 的字节开销对比**：

| | small | large |
|---|-------|-------|
| `count` / `size` | 2 + 2 | 4 + 4 |
| object 每个 key entry | 4（off 2 + len 2） | 6（off 4 + len 2） |
| 每个 value entry | 3（type 1 + off 2） | 5（type 1 + off 4） |
| 可 inline 的整数 | int16 / uint16 | 还多 int32 / uint32 |

> 文档超过 `UINT_MAX16`（65535）字节就必须升到 large（`is_too_big_for_json`，`json_binary.cc:330`），升级时**父节点也要一起长大**，所以 8.0 的做法是"先按 small 试，返回 `VALUE_TOO_BIG` 就回退缓冲重来"（`:719-762`）。

**怎么自己验证推算对不对**：`JSON_STORAGE_SIZE()` 返回的就是二进制表示的总字节数（含 type 字节）。上例 23 字节，可以拿它和推演结果对账。

**交叉验证**（用公开实测数据反推，检验上面三条规则）：

| JSON | `JSON_STORAGE_SIZE` 实测 | 按本节规则推算 |
|------|--------------------------|----------------|
| `"abc"` | **5** | 1(type) + 1(data-length) + 3 = **5** ✓ |
| `[42, "xy", "abc"]` | **21** | 1 + payload；payload = 2(count) + 2(size) + 9(value-entry×3) + 3 + 4 = **20** → `size` 字段 = 0x14 ✓ |
| `{"b": 42, "a": "xy"}` | **24** | 1 + payload；payload = 2+2+8(key-entry×2)+6(value-entry×2)+2(keys)+3 = **23** ✓（key 排序后是 `a`、`b`；`42` 被 inline 不占数据区） |

第二例的 `size = 20`（而不是 21）直接印证了**规则 1：`size` 不含 type 字节**；第三例里 `42` 不占数据区则印证了**inline 规则**。三例全部对得上，说明本节的三条规则是可靠的。

> 格式 BNF 的权威出处是 **WL#8132**（MySQL 官方 worklog），头文件 `json_binary.h:59-141` 里的注释就是它的副本。上表实测数据引自公开文章（掘金《mysql 中 json 数据类型的底层实现（源码解析）》、nullwy.me《MySQL 5.7 的 JSON 类型》），已用源码逐条核对。

### 3.5 解析入口链

```
parse_binary (json_binary.cc:1068)
  → parse_value (:1053)           按 tag 分派
      → parse_array_or_object (:1012)   读 count / size / header_size
      → parse_scalar (:920)
```

`parse_binary` 对空文档（如 IGNORE 插入、非严格模式下 NULL 进 NOT NULL 列）返回 JSON null（`json_binary.cc:1076`）。

---

## 四、Json_wrapper：DOM 与二进制的统一视图

定义 `sql-common/json_dom.h:1161`，设计意图见 `:1145-1160`：*"allow uniform access for callers... without necessarily building a DOM"*。

```cpp
union {
  struct { Json_dom *m_value; bool m_alias; } m_dom;  // DOM 表示
  json_binary::Value m_value;                          // 二进制表示
};
bool m_is_dom;
```

| 方法 | 行号（声明 / 实现） | 说明 |
|------|---------------------|------|
| `to_dom()` | `:1287` / `json_dom.cc:1387` | 惰性物化 DOM 并翻转 `m_is_dom` |
| `to_binary()` | `:1324` / `:1408` | binary 侧直接 `raw_binary` 拷贝，零转换 |
| `lookup(key)` | `:1421` / `:1768` | DOM 与 binary 双路分派 |
| `seek()` | `:1558` | 路径查找 |
| `compare()` | `:1582` | 比较 |
| `make_sort_key()` / `make_hash_key()` | `:1690` / `:1698` | 排序键 / 哈希键 |
| `attempt_binary_update()` | `:1735` / `json_dom.cc:3439` | **原地**修改二进制，部分更新的核心 |
| `binary_remove()` | `:1761` | binary 侧删除 |

`Json_wrapper_object_iterator`（`:1780`）统一遍历语义；`Json_object_wrapper`（`:1862`）提供 `begin()/end()` 供 range-for。

**实践意义**：`SELECT j->'$.a' FROM t` 只解析路径命中的那几个字节，不产生任何 DOM 节点、不分配 `my_malloc`。这就是为什么 MySQL 的 `->>` 在大文档上远快于"读出来再在应用层解析"。

---

## 五、JSON Path

### 5.1 leg 类型

`sql-common/json_path.h:52-92`，共 6 种：

| 类型 | 语法 | 说明 |
|------|------|------|
| `jpl_member` | `.name` | 对象成员 |
| `jpl_array_cell` | `[n]` | 数组下标 |
| `jpl_array_range` | `[m to n]` | 数组范围（**MySQL 扩展**） |
| `jpl_member_wildcard` | `.*` | 匹配所有成员 |
| `jpl_array_cell_wildcard` | `[*]` | 匹配所有元素 |
| `jpl_ellipsis` | `**` | 递归下降（**MySQL 扩展**） |

### 5.2 语法 EBNF

`sql-common/json_path.h:323-356`：

```
pathExpression ::= scope pathLeg (pathLeg)*
scope          ::= dollarSign
pathLeg        ::= member | arrayLocation | doubleAsterisk
member         ::= period (keyName | asterisk)
arrayLocation  ::= leftBracket (arrayIndex | arrayRange | asterisk) rightBracket
arrayIndex     ::= non-negative-integer | last [ minus non-negative-integer ]
arrayRange     ::= arrayIndex to arrayIndex
keyName        ::= ECMAScript-identifier | ECMAScript-string-literal
```

### 5.3 解析函数

| 功能 | 位置 |
|------|------|
| 顶层工厂 `parse_path` | `json_path.cc:258`（流版本 `:286`） |
| leg 分发 | `:319` |
| `**` 解析 | `:341`（禁止 `***`、禁止作为最后一个 leg） |
| `[...]` 全语法 | `:428`（`[*]` 在 `:436`，range 在 `:453`） |
| `last` 关键字 | `parse_array_index :380` |
| 成员名 + ECMAScript 校验 | `parse_member_leg :581`、`is_ecmascript_identifier :717` |

### 5.4 MySQL 的四个非标准扩展

1. **`last` / `last-N`**：`from_end` 标志位（`json_path.h:118` 的 `Json_array_index`），越界自动 clamp。
2. **`[m to n]`**：闭区间，两端可各自是 `last` 形式；`" to "` **必须两侧有空白**（`json_path.cc:456`）；静态剪枝掉"同侧且起点在终点之后"的永假路径（`:471`）。
3. **`**` ellipsis**：递归下降，求值走 `seek_ellipsis()`（`json_dom.cc:2052`）+ `json_binary::for_each_node()`。
4. **auto-wrapping**：对非数组值应用 `[0]`/`[last]` 时隐式包成单元素数组（`is_autowrap()` `json_path.cc:126`）。副作用：ellipsis + autowrap 会产生重复匹配，需 `path_gives_duplicates()` 判断后走 DOM 去重（`json_dom.cc:2151`）。

### 5.5 seek 求值

| leg | 函数 | 行号（`json_dom.cc`） |
|-----|------|----------------------|
| member | `seek_member` | 1960 |
| member wildcard | `seek_member_wildcard` | 1977 |
| array cell | `seek_array_cell` | 1999 |
| array range / `[*]` | `seek_array_range` | 2021 |
| ellipsis | `seek_ellipsis` | 2052 |
| 路径末尾 | `seek_end` | 2071 |

---

## 六、JSON 函数体系

基类 `Item_json_func`（`sql/item_json_func.h:151`），持有 `m_path_cache`（常量路径缓存）、`m_partial_update_column`。

| 类别 | 代表函数 | 类（行号） |
|------|----------|-----------|
| 构造 | `JSON_OBJECT` / `JSON_ARRAY` / `JSON_QUOTE` | `:728` / `:704` / `:868` |
| 取值 | `JSON_EXTRACT` / `->` / `->>` / `JSON_UNQUOTE` / `JSON_VALUE` | `:567` / `:896` / `:1089` |
| 修改 | `JSON_INSERT` / `JSON_SET` / `JSON_REPLACE` / `JSON_REMOVE` / `JSON_ARRAY_APPEND` / `JSON_ARRAY_INSERT` | `:634` / `:680` / `:692` / `:795` / `:621` / `:647` |
| 合并 | `JSON_MERGE_PRESERVE` / `JSON_MERGE_PATCH` | `:819` / `:849` |
| 查询 | `JSON_CONTAINS` / `JSON_CONTAINS_PATH` / `JSON_SEARCH` / `JSON_OVERLAPS` / `MEMBER OF` | `:391` / `:426` / `:754` / `:1044` / `:1059` |
| 元信息 | `JSON_TYPE` / `JSON_VALID` / `JSON_LENGTH` / `JSON_DEPTH` / `JSON_KEYS` / `JSON_STORAGE_SIZE` / `JSON_STORAGE_FREE` / `JSON_PRETTY` | `:463` / `:311` / `:501` / `:522` / `:542` / `:941` / `:960` / `:923` |
| 校验 | `JSON_SCHEMA_VALID` / `JSON_SCHEMA_VALIDATION_REPORT` | `:333` / `:359`（基于 rapidjson，常量 schema 会预编译缓存 `item_json_func.cc:611`） |
| 转换 | `CAST(x AS JSON)` / `CAST(x AS t ARRAY)` | `:479` / `:978` |
| 聚合 | `JSON_ARRAYAGG` / `JSON_OBJECTAGG` | `item_sum.h:1256` / `:1275` |

`JSON_STORAGE_SIZE` / `JSON_STORAGE_FREE` 是 JSONB 格式特有的能力：**能直接读出"文档占了多大 / 里面有多少空洞"**，这是目录区设计的红利，也是判断"是否该重写文档"的依据。

### 6.1 JSON_EXTRACT / `->` / `->>`

实现 `item_json_func.cc:1823`。对每个路径调 `w.seek(path, ..., &v, auto_wrap=true, only_need_one=false)`（`:1856`）收集**全部**匹配：

- 命中 0 个 → NULL（`:1860`）；
- 单路径且无通配符 → `std::move(v[0])`，**不复制**（`:1873`）；
- 多路径 / 含通配符 → 建 `Json_array` 逐个 `append_clone`（`:1865-1872`）。

**`->` 与 `->>` 的差别不在 `val_json` 里，而在语法层**（`sql_yacc.yy:10585` vs `:10598`）：

```
->   →  Item_func_json_extract
->>  →  Item_func_json_unquote( Item_func_json_extract(...) )
```

所以 `->` 在字符串上下文输出带引号（`val_string_from_json` 用 `json_quoted=true`，`item_json_func.cc:1275`），`->>` 不带（`:3263`）。

### 6.2 JSON_CONTAINS：递归 `contains_wr()`

`static bool contains_wr(...)`，`item_json_func.cc:785`，三种情形：

| doc 类型 | 语义 | 行号 |
|----------|------|------|
| **object** | containee 必须也是 object 且成员数不超过；遍历 containee 每个 key 在 doc 里 `lookup`，**递归**比较；全部匹配才 true | `:787` |
| **array** | containee 非数组先包成单元素数组；两边 `sort_and_remove_dups`（`:753`）排序去重后扫描；元素是容器则递归，标量则 `compare()==0` | `:818` |
| **标量** | `compare()==0` | `:915` |

代价：**必须物化命中值 + 排序 + 递归比较**。带路径参数时用 `forbid_wildcards=true`（`:950`）。

### 6.3 JSON_CONTAINS_PATH：只判路径，不取值

`item_json_func.cc:994`。比 CONTAINS 快在两个地方：

1. `seek(..., only_need_one=true)`（`:1042`）—— `is_seek_done()` 命中即停，不再遍历剩余候选；
2. **完全不做值比较**：不排序、不 clone、不 `compare`。

此外它**允许通配符**（`:1033` `forbid_wildcards=false`），与 CONTAINS 正相反。`one` 命中即短路；`all` 一个不中即整体 0（`:1043-1053`）。

### 6.4 JSON_SEARCH：递归 + LIKE

`find_matches()`，`item_json_func.cc:2762`：

- **只搜字符串标量**，其他类型直接跳过（`:2825`）；
- 匹配委托给 `Item_func_like`：在 `fix_fields` 里就造好 LIKE 节点（`:2702`），最终走 `my_wildcmp`（`item_cmpfunc.cc:6291`）。`%` / `_` / `ESCAPE` 全部由它实现，无 ESCAPE 时默认 `\`；
- 路径字符串由 `Json_path_leg::to_string`（`json_path.cc:91`）拼接；**路径含通配符**时必须先 `to_dom()`，再用 `subdocument->get_location()`（`:2931`）取真实路径；
- 返回：0 命中 → NULL；**1 命中 → 标量字符串（不是数组）**；多命中才是数组。

### 6.5 JSON_LENGTH / JSON_DEPTH

- `JSON_LENGTH`（`item_json_func.cc:1716`）→ `Json_wrapper::length()`（`json_dom.cc:2171`）：array/object 返回元素数，**标量恒为 1**。
- `JSON_DEPTH`（`:1741`）—— 注意它**强制物化 DOM**：`wrapper.to_dom()->depth()`。递归定义 `Json_object::depth()`（`json_dom.cc:863`）= `1 + max(children)`，标量 = 1。

### 6.6 JSON_KEYS

`item_json_func.cc:1762`：路径命中数必须**恰好 1**（多命中返回 NULL），目标必须是 object，返回 key 组成的数组。**空 object 返回空数组 `[]`，不是 NULL**（`:1796-1820`）。

### 6.7 MERGE_PATCH vs MERGE_PRESERVE

| | `JSON_MERGE_PATCH`（RFC 7396） | `JSON_MERGE_PRESERVE` |
|---|---|---|
| 核心函数 | `Json_object::merge_patch` `json_dom.cc:885` | `merge_doms` `json_dom.cc:102` |
| 同 key 冲突 | **覆盖**（非 object 直接替换） | **递归合并**（值再走 merge_doms） |
| 数组 | 当普通值 → 覆盖 | **拼接** |
| 标量 | 覆盖 | 自动包成数组后拼接 |
| JSON null | **删除该 key**（`:888` `remove`） | 普通值，参与合并 |
| patch 非 object | 结果 = patch 本身（`:3462`） | 包成数组拼接 |

`MERGE_PRESERVE` 是**左折叠**逐参数 `merge_doms`（`:3157`）；重复 key 的递归发生在 `Json_object::consume`（`json_dom.cc:807`）。

### 6.8 JSON_QUOTE / JSON_UNQUOTE

转义规则在 **`json_dom.cc`**（不是 `sql_string.cc`）：

- `double_quote`（`json_dom.cc:1132`）：先用 `std::find_if` 批量跳过无需转义的字符，只在遇到 `<= 0x1f`、`"`、`\` 时才调 `escape_character`（`:1104`，`\b\t\n\f\r` + `\u00XX`）；
- `JSON_UNQUOTE`（`item_json_func.cc:3246`）两条分支：
  - 参数已是 JSON 类型 → `to_string(quoted=false)` 直接取内部串，**不解析**（这条就是 `->>` 的路径）；
  - 参数是字符串 → 首尾无引号则原样返回；否则 **rapidjson 解析成 DOM 再取内部表示**（没有手写反转义函数）。

### 6.9 JSON_VALID 为什么不建 DOM

`item_json_func.cc:527` 给 `json_is_valid` 传 `dom = nullptr` → `parse_json`（`:146`）走 `is_valid_json_syntax`（`json_syntax_check.cc:62`），用 rapidjson 的 `Syntax_check_handler`（`json_syntax_check.h:65`）—— 只实现 `Start/EndObject/Array` 做**深度计数**，不构造节点。快在这里。

对比 `CAST(x AS JSON)`（`Item_typecast_json::val_json` `:1660`）**必须建 DOM**。

### 6.10 聚合函数 JSON_ARRAYAGG / JSON_OBJECTAGG

- 累积方式：`Item_sum_json_array::add()`（`item_sum.cc:6002`）逐个 `append_alias(wrapper.to_dom())` 累积到一个 `Json_array`；对象版 `:6109` 用 `add_alias(key, value)`。
- **序列化时机**：`val_json`（`:5805`）返回 DOM 的**克隆**（`val_*` 可能被调用多次，直接返回会被破坏）；`val_str`（`:5779`）才真正 `to_string`。
- **无显式条数/字节上限**：累积阶段只受内存约束，上限在最终序列化时才由 `is_too_big_for_json` 触发 `ER_JSON_VALUE_TOO_BIG`。
- **窗口函数**：数组逆操作是 `arr->remove(0)`（`:5994`）；对象逆操作用 `m_key_map` 计数减到 0 才删（`:6066-6097`），若窗口 ORDER BY 正好是 key 则走 `m_optimize` 快路径（`:5957`）。
- 影响优化器：`JOIN::with_json_agg`（`sql_optimizer.h:611`）改变 GROUP BY 的索引/排序选择。

### 6.11 解析深度上限

- 常量 **`JSON_DOCUMENT_MAX_DEPTH = 100`**，定义在 `sql-common/json_syntax_check.cc:85`（文件静态常量，未在头文件导出）。
- 判深函数 `check_json_depth()`（`json_syntax_check.h:49`），共 7 处调用：语法校验 2 处（`:40`/`:50`）、DOM 解析（`json_dom.cc:538`）、转字符串 2 处（`:1527`/`:1598`）、二进制序列化 2 处（`json_binary.cc:511`/`:573`）。
- 超限回调 `JsonDocumentDefaultDepthHandler`（`json_error_handler.cc:36`）→ `ER_JSON_DOCUMENT_TOO_DEEP`（22032）。

---

## 七、JSON_TABLE

### 7.1 它解决什么问题

JSON 是"一个值一棵树"，SQL 是"一张表若干行"。**JSON_TABLE 就是把 JSON 数组展开成关系行的桥梁**（术语上叫 flatten / unnest / shredding）。

没有它的时候，要把数组元素当行处理，只能写死下标：

```sql
-- 数组长度不定就抓瞎
SELECT JSON_EXTRACT(doc, '$.items[0].sku'),
       JSON_EXTRACT(doc, '$.items[1].sku')   -- 有第三个元素就漏了
FROM orders;
```

有了 JSON_TABLE，数组有几个元素就出几行，还能直接 JOIN、聚合、INSERT 到规范化表。

### 7.2 一个完整例子

```sql
CREATE TABLE orders (
  id  INT PRIMARY KEY,
  doc JSON
);

INSERT INTO orders VALUES (1, '{
  "customer": "张三",
  "items": [
    {"sku": "A1", "qty": 2, "price": 9.90, "tags": ["hot", "new"]},
    {"sku": "B2", "qty": 1, "price": 20.00, "tags": ["sale"]}
  ]
}');
```

```sql
SELECT o.id, jt.*
FROM orders o,
     JSON_TABLE(o.doc, '$.items[*]'
       COLUMNS (
         rn    FOR ORDINALITY,
         sku   VARCHAR(16)   PATH '$.sku',
         qty   INT           PATH '$.qty',
         price DECIMAL(8, 2) PATH '$.price'
       )
     ) AS jt;
```

结果：

| id | rn | sku | qty | price |
|----|----|-----|-----|-------|
| 1  | 1  | A1  | 2   | 9.90  |
| 1  | 2  | B2  | 1   | 20.00 |

**一行 JSON → 两行关系数据。** 这就是它的全部核心语义。

#### `rn FOR ORDINALITY` 到底是什么？

一句话：**它就是"这是数组的第几个元素"，一个自动生成行号的列。**

看看去掉它会怎样 —— 对比一下最直观：

| 带 `rn FOR ORDINALITY` | 去掉它 |
|---|---|
| rn \| sku<br>1 \| A1<br>2 \| B2 | sku<br>A1<br>B2 |

**它不影响行数，只是多出一列递增的序号，从 1 开始。**

为什么需要它？因为 JSON 数组 `["A1","B2"]` 里**并没有存"第几个"这个信息**（它只有顺序）。展开成表以后，如果你想知道"这是第几项"，就得让 JSON_TABLE 帮你数出来。

三条必须记住的规则：

1. **它不是 JSON 里的字段** —— 上面例子里 JSON 根本没有 `"rn"` 这个 key，这一列是凭空生成的
2. **不需要写 `PATH`** —— 其他列都要 `PATH '$.xxx'`，只有它不用，语法就固定是 `列名 FOR ORDINALITY`
3. **不能指定类型** —— 固定是 UNSIGNED INT

什么时候真的需要它：

```sql
-- ① 保持 JSON 里的原始顺序（关系表本身是无序的！）
SELECT sku FROM ..., JSON_TABLE(...) AS jt ORDER BY jt.rn;

-- ② 取第 N 个元素
SELECT sku FROM ..., JSON_TABLE(...) AS jt WHERE jt.rn = 2;

-- ③ 嵌套展开时区分"第几组的第几个"
```

> 注意 ① 很关键：**SQL 表理论上是无序集合**，没有 ORDER BY 就不保证顺序。想按 JSON 里的顺序输出，就必须靠 `FOR ORDINALITY` 记下来再排序。

### 7.3 语法拆解

完整模板（`[ ]` 表示可省略）：

```sql
JSON_TABLE( <数据源>, <行路径>
  COLUMNS (
      <列定义> [, <列定义> ...]
  )
) [AS] <别名>

<列定义>:
    <列名> FOR ORDINALITY                                          -- 序号列
  | <列名> <SQL类型> PATH '<列路径>' [<on_empty>] [<on_error>]       -- 普通列
  | <列名> <SQL类型> EXISTS PATH '<列路径>'                          -- 存在性列
  | NESTED PATH '<子行路径>' COLUMNS ( <列定义> [, ...] )            -- 嵌套展开

<on_empty> / <on_error>:   { NULL | DEFAULT '<json值>' | ERROR } ON { EMPTY | ERROR }
```

| 部分 | 作用 | 说明 |
|------|------|------|
| **数据源** `o.doc` | 待展开的 JSON 文档 | 可以是 JSON 列、JSON 函数结果、字符串字面量。**可以引用前面表的列**（见 7.5 的 LATERAL 语义） |
| **行路径** `'$.items[*]'` | **决定"出几行"** | `$` = 整个文档；`$.items[*]` = items 数组的每个元素一行。路径匹配不到 → 0 行 |
| **COLUMNS** | **决定"每行有哪些列"** | 每列有自己的路径，路径**相对当前行**求值（见下面的"两个坐标系"） |
| 别名 `AS jt` | 必须 | 它是一张**表**，不是标量函数 |

#### ⚠ 最容易搞混的一点：存在两个不同的"坐标系"

**行路径在"整个文档"上求值，列路径在"当前这一行"上求值。** 两者不是同一棵树：

```
o.doc = { "customer": "张三",
          "items": [ {"sku":"A1","qty":2}, {"sku":"B2","qty":1} ] }

行路径 '$.items[*]'   ← 在【整个文档】上求值，匹配到 2 个对象：
                          ├─ 第 1 行 的"当前行" = {"sku":"A1","qty":2}
                          └─ 第 2 行 的"当前行" = {"sku":"B2","qty":1}

列路径 PATH '$.sku'   ← 在【当前这一行】上求值（不是整个文档！）：
                          第 1 行 → {"sku":"A1","qty":2}.sku = "A1"
                          第 2 行 → {"sku":"B2","qty":1}.sku = "B2"
```

所以两个常见疑问的答案：

- **`'$.items[*]'` 是指 `o.doc` 的 items 全部元素吗？** → **是**。`$` 是文档根，`.items` 取字段，`[*]` 是数组通配符（每个元素）。匹配到几个元素就出几行。
- **`PATH '$.sku'` 是提取 `o.doc` 的 sku 吗？** → **不是**。它提取的是**当前这一行那个元素**的 sku，也就是 `$.items[i].sku`。之所以不用写 `$.items[0].sku` 这种全路径，是因为 `$.items[*]` 已经把起点移到元素上了。

**反例自证**：在 `$.items[*]` 这一层写 `PATH '$.customer'`，取的其实是 `items[i].customer` → 不存在 → 得到 NULL，**而不是**顶层的 `"张三"`。列路径不能向上引用父级。

**想同时取外层字段，就把行路径提到 `$`，用 NESTED PATH 展开内层数组**（这也是 NESTED PATH 最常见的真实用途）：

```sql
SELECT jt.cust, jt.sku, jt.qty
FROM orders o,
     JSON_TABLE(o.doc, '$'                        -- ← 行路径改为文档根：全文只 1 行
       COLUMNS (
         cust VARCHAR(20) PATH '$.customer',      -- ← 顶层字段
         NESTED PATH '$.items[*]' COLUMNS (       -- ← 内层数组展开
           sku VARCHAR(16) PATH '$.sku',
           qty INT         PATH '$.qty'
         )
       )
     ) AS jt;
```

| cust | sku | qty |
|------|-----|-----|
| 张三 | A1  | 2   |
| 张三 | B2  | 1   |

> 记忆法：**行路径 = 把镜头推到哪一层（决定行数）；列路径 = 在当前这一层的画面里取哪个字段（决定列）。**

COLUMNS 里四种列：

```sql
COLUMNS (
  rn        FOR ORDINALITY,                    -- ① 行号，从 1 开始
  sku       VARCHAR(16) PATH '$.sku',          -- ② 普通列：取值 + 转 SQL 类型
  has_note  INT EXISTS PATH '$.note',          -- ③ 存在性：路径存在=1，不存在=0
  NESTED PATH '$.tags[*]' COLUMNS (            -- ④ 嵌套展开，见 7.4
    tag VARCHAR(16) PATH '$'
  )
)
```

**类型与缺省处理**（这是 JSON_TABLE 把半结构化数据"钉死"成关系模式的关键）：

```sql
qty INT PATH '$.qty'
    DEFAULT '0'   ON EMPTY     -- 路径不存在
    DEFAULT '-1'  ON ERROR     -- 值无法转成 INT（如 "abc"）
```

- `ON EMPTY` / `ON ERROR` 各有三种：省略时默认 `NULL`；`DEFAULT '值'`；`ERROR`（抛错）。
- 所以 JSON 的"字段可能缺失、类型可能不对"在 JSON_TABLE 这一层被显式收敛成 SQL 的确定性行为。

### 7.4 NESTED PATH：多层展开

上面例子里 `tags` 本身也是数组，用 `NESTED PATH` 再展开一层：

```sql
SELECT jt.sku, jt.tag
FROM orders o,
     JSON_TABLE(o.doc, '$.items[*]'
       COLUMNS (
         sku VARCHAR(16) PATH '$.sku',
         NESTED PATH '$.tags[*]' COLUMNS (
           tag VARCHAR(16) PATH '$'      -- '$' = 当前这个标量元素本身
         )
       )
     ) AS jt;
```

| sku | tag  |
|-----|------|
| A1  | hot  |
| A1  | new  |
| B2  | sale |

A1 有两个 tag → 产生两行（**父子做笛卡尔组合**）。嵌套上限 16 层（`MAX_NESTED_PATH`，`table_function.h:318`）。

### 7.5 两个必须知道的语义

1. **它是隐式的 LATERAL JOIN**：`JSON_TABLE(o.doc, ...)` 里引用了左表 `o.doc`，因此**左表每一行都会重新计算一次 JSON_TABLE**。SQL 里不用写 `LATERAL` 关键字，行为天然如此。
2. **路径匹配不到就是 0 行**（不是 1 行 NULL）。配合 `LEFT JOIN ... ON TRUE` 才能保留左表行：

```sql
SELECT o.id, jt.sku
FROM orders o
LEFT JOIN JSON_TABLE(o.doc, '$.items[*]' COLUMNS (sku VARCHAR(16) PATH '$.sku')) AS jt
       ON TRUE;
```

### 7.6 典型用途

| 场景 | 写法 |
|------|------|
| 数组展开做聚合 | `SELECT jt.sku, SUM(jt.qty) ... GROUP BY jt.sku` |
| JSON 与维度表 JOIN | `JOIN products p ON p.sku = jt.sku` |
| **JSON 导入规范化** | `INSERT INTO order_items SELECT ... FROM staging, JSON_TABLE(...)` |
| 替代 EAV 属性表 | 稀疏属性存 JSON，查询时展开 |

### 7.7 实现：物化成临时表

实现类 **`Table_function_json`**（`sql/table_function.h:320`），注意它是 **Table_function（表函数）而不是 Item_（标量函数）**——这正对应"它产出的是一张表"：

```cpp
class Table_function_json final : public Table_function {
  std::array<JT_data_source, MAX_NESTED_PATH> m_jds;  // 每层 NESTED PATH（上限 16，:318）
  List<Json_table_column> m_vt_list;
  Mem_root_array<Json_table_column *> m_all_columns;
  const char *m_table_alias;
  Item *source;                                       // 数据源表达式
```

执行方式：**建一张临时表 → 逐行写入 → 再当普通表扫描**。

```1965:1975:sql/iterators/composite_iterators.cc
bool MaterializedTableFunctionIterator::Init() {
  if (!table()->materialized) {
    if (table()->pos_in_table_list->create_materialized_table(thd())) return true;
  }
  if (m_table_function->fill_result_table()) return true;   // ← 真正执行展开
  return m_table_iterator->Init();
}
```

- 临时表由 `create_tmp_table_from_fields()` 创建（`table_function.cc:61`），Heap 引擎，**放不下自动转磁盘**。
- 挂接点：`PT_table_factor_function::contextualize()`（`parse_tree_nodes.cc:1282`）→ `Table_ident` → `Table_ref::table_function`（`sql_lex.h:299`）。

**由此推出的三个性能事实**：

1. **不缓存**：每次 `Init()` 都重新物化一遍。若 JSON_TABLE 处在大表的嵌套循环内侧，会被反复重建（`composite_iterators.h:603` 的 TODO 明确说这是非最优设计）。
2. **无行数估算**：优化器不知道会展开出几行，`table.cc:6546` 返回 arbitrary 常量（源码有 FIXME），所以后续 JOIN 顺序、成本估算都可能是错的。
3. **全量展开**：即使外层只需要 1 行，也会把整个数组展开完写进临时表。

> 实践建议：JSON_TABLE 适合"一次性展开 + 聚合 / 导入"，不适合放在高频查询的大循环内侧；数组很长时尤其要先在左表侧用其它条件把行数压下来。

---

## 八、JSON 的索引方案

JSON 列**不能直接建索引**（`sql_table.cc:4927` → `ER_JSON_USED_AS_KEY`）。三条路：

### 8.1 生成列（VIRTUAL / STORED）+ 索引

```sql
gc INT AS (JSON_EXTRACT(j, '$.id')) VIRTUAL,  INDEX(gc);   -- 虚拟列，不占聚簇索引
gc2 INT AS (JSON_EXTRACT(j, '$.id')) STORED,  INDEX(gc2);  -- 存储列，落盘
```

优化器通过 `Item_func::gc_subst_transformer()`（`sql/item_func.cc:1300`）把谓词中的表达式**等价替换**为生成列字段 `Item_field`（`substitute_gc_expression` `:1123`，靠 `func->eq(expr, bin_cmp)` 做 Item 树等价比较），之后走普通 range 优化。

**源码表示**（8.0 里旧的 `enum generated_type` 已被取代）：

| 概念 | 位置 | 说明 |
|------|------|------|
| `Field::gcol_info` | `field.h:811` | 非 NULL 即"是生成列" |
| `Field::stored_in_db` | `field.h:817` | VIRTUAL / STORED 的唯一判别位 |
| `is_gcol()` / `is_virtual_gcol()` | `field.h:823` / `:824` | |
| `Value_generator` | `field.h:483` | 持有 `expr_item`（`:493`）、`base_columns_map`（`:515`）、`stored_in_db`（`:570`） |
| `Value_generator_source` | `field.h:468` | 区分生成列 / 默认值表达式 / 检查约束 |

**VIRTUAL 的实际存储**（常见误解：虚拟列完全不占空间 —— 错）：

- **聚簇索引不占位置**：`innobase_vcol_build_templ()`（`ha_innodb.cc:6705`）把 `templ->clust_rec_field_no = ULINT_UNDEFINED`；只有"聚簇 + INSERT"的 tuple 才额外挂 v-fields（`row0row.cc:59`）。
- **二级索引里会物化值**：`index->type |= DICT_VIRTUAL`（`ha_innodb.cc:12113`），列标记 `DATA_VIRTUAL`（`data0type.h:223`）。所以"虚拟列 + 索引"省不掉索引存储。
- 索引标记位：`HA_GENERATED_KEY`（`my_base.h:518`）/ `HA_VIRTUAL_GEN_KEY`（`:552`）。

**求值时机**（写/读共用唯一内核）：

```7101:7145:sql/table.cc
static bool update_generated_columns(TABLE *table, const MY_BITMAP *columns,
                                     bool virtual_only, MY_BITMAP *updated_columns) {
  for (Field **field_ptr = table->vfield; *field_ptr != nullptr; ++field_ptr) {
    Field *field = *field_ptr;
    if (virtual_only && !field->is_virtual_gcol()) continue;
    if (!bitmap_is_set(columns, field->field_index())) continue;
    if (field->handle_old_value()) { /* 虚拟 BLOB / typed array 保留旧值供引擎删索引项 */ }
    type_conversion_status status =
        field->gcol_info->expr_item->save_in_field(field, false);   // ← 真正求值
```

- 写路径 `update_generated_write_fields()`（`:7225`），触发点：INSERT `sql_base.cc:9569`（`fill_record` 末尾）、UPDATE `sql_update.cc:2718`、触发器后 `:9877`、从库 apply `log_event.cc:8091`。
- 读路径 `update_generated_read_fields()`（`:7163`），由 `handler.cc` 里各扫描函数的 `m_update_generated_read_fields` 延迟标志驱动（覆盖索引时跳过 `:7167`）。
- 位图维护 `mark_gcol_in_maps()`（`table.cc:7231`）：生成列进 `write_set`、依赖基列进 `read_set`；**typed array 还要额外进 read_set**（`:7252`，内部有转换字段）。

### 8.2 函数索引 = 隐藏的 VIRTUAL 生成列（8.0.13+）

```sql
INDEX((CAST(j->'$.id' AS UNSIGNED)))
```

判别函数 `is_field_for_functional_index()`（`field.h:875`）= `HT_HIDDEN_SQL` + `gcol_info != nullptr`。

创建路径 `add_functional_index_to_create_list()`（`sql_table.cc:7763`）—— **就是"加一个 VIRTUAL 隐藏生成列 + 在该列上建普通索引"**：

```7855:7862:sql/sql_table.cc
  cr->hidden = dd::Column::enum_hidden_type::HT_HIDDEN_SQL;
  cr->stored_in_db = false;                                   // ← 一定是 VIRTUAL
  Value_generator *gcol_info = new (thd->mem_root) Value_generator();
  gcol_info->expr_item = kp->get_expression();
  gcol_info->set_field_stored(false);
  cr->gcol_info = gcol_info;
```

- 列名由索引名生成（`sql_table.cc:7825`），错误会被 `Functional_index_error_handler`（`error_handler.cc:230`）改写成 `ER_FUNCTIONAL_INDEX_*`。
- **binlog：隐藏的函数索引列不进 row image**（`rpl_record.cc:275` 用 `ColumnFilterOutboundFunctionalIndexes` 过滤），因此**从库必须自己重算**（`log_event.cc:8012-8095`）。虚拟列也不参与 BI/AI 比较（`log_event.cc:8691`）。
- 多值索引走的**就是这条路径**，只是表达式换成 `CAST(... AS ... ARRAY)`；`CAST AS ARRAY` **禁止出现在函数索引表达式之外**（`item_json_func.cc:3623`）。

### 8.3 多值索引（Multi-Valued Index，8.0.17+）

```sql
INDEX idx ((CAST(j->'$.arr' AS UNSIGNED ARRAY)))
```

- 载体：`Field_typed_array : Field_json`（`field.h:4162`），`is_array()` → true；表达式是 `Item_func_array_cast`（`item_json_func.h:978`）。
- **一个数组 → N 条二级索引记录**：`row_ins_sec_index_multi_value_entry`（`row0ins.cc:3260`）用 `Multi_value_entry_builder`（`row0row.h:318`，`next()` `:353`）**复用同一个 dtuple、只替换多值字段的数据指针**，逐条插入。
- 存储层转换：`innobase_store_multi_value_low`（`ha_innodb.cc:8795`）把 JSON array 元素转成 InnoDB 内部格式，存进 `multi_value_data`（`data0data.h:384`）；空数组 → `UNIV_NO_INDEX_VALUE`（不入索引）。
- UPDATE：先删全部旧值再插全部新值（`row0upd.cc:2157`）。
- **可用谓词**：仅 `MEMBER OF`、`JSON_CONTAINS`、`JSON_OVERLAPS`。
  - `MEMBER OF` → 1 个 EQ range（`range_analysis.cc:621`）；
  - `JSON_CONTAINS` / `JSON_OVERLAPS` → N 个 EQ range 的 OR（`get_func_mm_tree_from_json_overlaps_contains` `:463`）。
  - 生成列替换：`gc_subst_overlaps_contains`（`item_func.cc:1207`）。
- **去重**：同一行多值匹配会返回重复 rowid → server 层 `Unique_on_insert` 过滤（`handler::filter_dup_records` `handler.cc:8483`）。
- **限制**：每个索引最多 1 个多值键部（`sql_table.cc:4837`）；不支持显式 ASC/DESC；不支持排序（`ha_innodb.cc:6393` 清掉 `HA_READ_ORDER`）；`key_cmp` 恒返回 -1（`field.h:4232`）；元素为 JSON null 报错；容量受 undo 页大小约束（`ha_mv_key_capacity` `ha_innodb.cc:23862`，超限报 `ER_EXCEEDED_MV_KEYS_NUM`/`SPACE`）。
- 字符集仅支持 binary 与 `utf8mb4_0900_bin`（`item_create.cc:1865`），因为 JSON 区分 `"abc"` 与 `"abc "`，padding 字符集会导致键不一致。

### 8.4 生成列与 JSON 部分更新的交互

- **没有"存在生成列就禁用部分更新"的开关**。部分更新发生时 `record[0]` 里的 JSON 值仍被就地改成**完整的新值**（`json_diff.cc:447-463`），之后生成列照常走 `update_generated_columns()` 重算 —— 多值索引的 typed array 列拿到的是全量正确数组。
- 虚拟 BLOB / typed array 生成列在计算前会 `keep_old_value()`（`table.cc:7123-7130`），供引擎删除旧索引项。
- 冲突的消解方式是"**diff 收集/应用失败 → `disable_binary_diffs_for_current_row()`（`table.cc:7510`）→ 该行回退全量更新**"，而不是提前禁用。
- 引擎侧：`upd_t::is_partially_updated()`（`row0upd.cc:3502`）；多值索引的更新走独立路径 `row_upd_multi_sec_index_entry()`（`row0upd.cc:2157`）。

### 8.5 三者对比

| 维度 | 生成列 / 函数索引 | 多值索引 |
|------|------------------|----------|
| 一行产出 | 1 条索引记录 | N 条（数组元素数） |
| 可用谓词 | 所有常规 range 谓词 | 仅 MEMBER OF / CONTAINS / OVERLAPS |
| 排序 | 支持 | 不支持 |
| 重复行 | 无 | 有，需去重 |
| 元素类型 | 标量 | 数组元素（VARCHAR/TIME/DATETIME/DATE/INT/DECIMAL） |

---

## 九、比较、排序与类型转换

### 9.1 类型优先级

`enum_json_type` 的定义顺序**本身承载排序语义**（`json_dom.h:100-107`）：

```
null < number(DECIMAL/INT/UINT/DOUBLE) < string < object < array
     < boolean < date < time < datetime/timestamp < opaque
```

查表矩阵 `type_comparison[15][15]` —— `json_dom.cc:2417-2433`，主函数 `Json_wrapper::compare()` `:2436`：

- array：逐元素比较，前缀相等则短者小；
- object：先比元素个数，再比 key，再比 value；
- number 四种类型互相为 0（需按数值比较，用 `compare_json_decimal_double` 等辅助函数 `:2206-2391`）。

### 9.2 排序键

`Json_wrapper::make_sort_key()`（`json_dom.cc:3258`），类型标签常量 `:3117-3129`：

```
NULL=0x00, NUMBER_NEG=0x01, NUMBER_ZERO=0x02, NUMBER_POS=0x03,
STRING=0x04, OBJECT=0x05, ARRAY=0x06, FALSE=0x07, TRUE=0x08,
DATE=0x09, TIME=0x0A, DATETIME=0x0B, OPAQUE=0x0C
```

number 拆成 NEG/ZERO/POS 三段保证负数 < 0 < 正数。**object/array 只编码长度，不做深度排序**，并 push `ER_NOT_SUPPORTED_YET` 警告（`:3313`）。

Field 层：`Field_json::make_sort_key`（`field.cc:8046`）、`make_hash_key`（`:8054`，DISTINCT 用）、`cmp_binary`（`:8035`）。

### 9.3 SQL 层比较器

只要一侧是 JSON 就用 JSON 比较器（`item_cmpfunc.cc:1140` → `Arg_comparator::compare_json` `:1695`）。JSON 与数字隐式比较会告警 `ER_IMPLICIT_COMPARISON_FOR_JSON`（`:671`）。

### 9.4 类型转换

- `CAST(x AS JSON)`：`Item_typecast_json`（`item_json_func.h:479`，实现 `item_json_func.cc:1660`）—— 把参数当文本解析，合法则包成 DOM，非法报错。
- JSON → 标量：`JSON_UNQUOTE` / `coerce_int|real|decimal`。
- 协议输出：`item.cc:7441` 对 `MYSQL_TYPE_JSON` 直接 `val_json()` 再 `to_string()`。
- JSON **不参与直方图**（`histogram.cc:173` → `Value_map_type::INVALID`）。

---

## 十、部分更新（Partial Update）全链路

这是 MySQL JSON 最有工程价值的部分，分四个层次。

### 10.1 Server 层：判定与 diff 收集

**触发条件**（`prepare_partial_update` `sql/sql_update.cc:1346`）：

- 列类型必须是 `MYSQL_TYPE_JSON`；
- 引擎必须声明 `HA_BLOB_PARTIAL_UPDATE`（`handler.h:482`，InnoDB 在 `ha_innodb.cc:2982`）；
- 表达式必须是 `json_col = JSON_SET|JSON_REPLACE|JSON_REMOVE(json_col, path, ...)`：
  - `supports_partial_update()`（`item_json_func.cc:2332`）要求**源列 == 目标列**，允许嵌套（如 `JSON_SET(JSON_SET(col,...),...)`）；
  - 只有 `Item_func_json_set_replace`（`item_json_func.h:664`）和 `Item_func_json_remove`（`:797`）返回 `can_use_in_partial_update() == true`。**`JSON_INSERT` / `JSON_ARRAY_APPEND` / `JSON_MERGE_PATCH` 不支持**；
- 批量更新（will_batch）禁用；多表 UPDATE 只有首表启用（`sql_update.cc:2273`）。

**两种 diff**（`TABLE::setup_partial_update` `sql/table.cc:7560`）：

| diff | 结构 | 用途 | 开关 |
|------|------|------|------|
| **Binary diff** | `Binary_diff{offset, length}`（`table.h:1319`） | 交给引擎做物理增量写 | 引擎支持即可，与 binlog 无关 |
| **Logical diff** | `Json_diff{operation, path, value}`（`json_diff.h:53`） | 写 binlog | `binlog_row_value_options=PARTIAL_JSON` + binlog 打开 + 非 v1 row events + 当前语句 ROW 格式 |

`Binary_diff` **没有操作枚举**：只记"某段字节被替换"，新旧数据从 `Field_json` 的两个行缓冲现算（`new_data()` `table.cc:7686` 取 `record[0]`，`old_data()` `:7695` 取 `record[1]`）。合并/保序由 `TABLE::add_binary_diff()`（`:7626`）二分插入完成。

**核心计算**：`Json_wrapper::attempt_binary_update()`（`json_dom.cc:3439`）：

```cpp
if (path.leg_count() == 0) { *partially_updated = false; ... }        // 整篇替换 → 不做
seek_no_dup_elimination(...)                                          // 定位父容器
switch (parent.type()) { ... }                                        // 定位 element_pos
if (space_needed(thd, new_value, parent.large_format(), &needed)) ... // :3543 新值需要多少字节
if (needed > 0 && !parent.has_space(element_pos, needed, &data_offset))
  { *partially_updated = false; ... }                                 // :3547 放不下 → 放弃
parent.update_in_shadow(field, element_pos, new_value, data_offset, needed,
                        original, destination, &changed)               // :3573 影子副本上就地改
```

`update_in_shadow`（`json_binary.cc:1727`，含 20 行示例注释 `:1599-1726`）只在 `memcmp` 发现真变了时才 `add_binary_diff()`（数据区一条 + value entry 一条）。删除走 `remove_in_shadow`（`:1914`），产生 2~3 条 diff。

**放弃部分更新的条件**（重要，直接决定性能）：

1. 路径为空（整篇替换）；
2. 父容器是标量；
3. **成员/下标不存在**（即需要新增）—— REPLACE 可 no-op，**SET 必须回退全量**；
4. **新值放不进现有空洞**（`has_space` 失败）—— 部分更新**永远不能让文档变长**；
5. 结果为 NULL / 路径为 NULL；
6. 引擎不支持 / 批量更新 / 多表非首表。

> 因此可以反推最佳实践：**JSON 文档要"预留空间"或"定长化"**，用 `JSON_REPLACE` 而不是 `JSON_SET` 更新已存在的 key，才能让部分更新稳定生效。`JSON_STORAGE_FREE` 就是看这个的。

### 10.2 InnoDB 层：LOB 部分写

```
ha_innobase::update_row
 → calc_row_difference (ha_innodb.cc:9808)
 → row_upd (row0umod.cc:160)
 → btr_cur_optimistic_update / btr_cur_pessimistic_update
 → lob::btr_store_big_rec_extern_fields(..., OPCODE_UPDATE) (lob0lob.cc:410)
     判定 :481/:550  必须是 off-page 大 LOB 且 upd->is_partially_updated(field_no)
     → lob::update (lob0update.cc:96)     成功则 do_insert = false，不再全量重写
     → 否则 lob::insert (:577)            全量重写
```

`lob::update()`（`lob0update.cc:96-163`）：

```cpp
const ulint bytes_changed = upd_t::get_total_modified_bytes(*bdiff_vector);
const bool small_change = (bytes_changed <= ref_t::LOB_SMALL_CHANGE_THRESHOLD);  // 阈值 100，lob0lob.h:210
if (small_change) lob_version = first_page.get_lob_version();   // 原地改，版本不变
else              lob_version = first_page.incr_lob_version();  // COW，版本+1
for (Binary_diff_vector::const_iterator iter = bdiff_vector->begin(); ...) {
  if (small_change) replace_inline(...)   // :501 → mlog_write_string 直接覆盖页内字节
  else              replace(...)          // :268 → 新分配数据页（COW）+ 新 index entry
}
```

- 大改动走 `first_page_t::replace()`（`lob0first.cc:60`）/ `data_page_t::replace()`（`lob0pages.cc:59`），**只动受影响的 1~2 个 LOB 数据页**，其余页面完全不动，旧版本靠 `lob_version` 保留给 purge。

### 10.3 Undo 层

`trx_undo_report_blob_update()`（`trx0rec.cc:1000`，调用点 `:1451`）：

- 外部列本身在 undo 里只存**本地前缀 + 20 字节 BLOB ref**，从来不是全量 JSON；
- **小改动（≤100 字节）**：额外记 `offset / length / old_data` + 受影响的 LOB index entry 旧值（`lob::get_affected_index_entries`），回滚时用 `Lob_diff`（`row0upd.h:392`）apply；
- **非小改动**：diff 向量长度写 0，undo **完全不记 LOB 旧数据**，靠 LOB 版本化 + purge 回收。

> 8.0.12 的 WL#11328 就是这个优化（<https://yq.aliyun.com/articles/647327>）。官方博客称最高约 3 倍 TPS 提升。副作用：升级到 8.0.12+ 后**无法降级**。

### 10.4 Binlog 与复制

**默认行为（`binlog_row_value_options=''`）：JSON 更新在 binlog 里是全量记录，与 InnoDB 层的部分更新优化完全无关。** 原因链：

1. JSON 列底层是 BLOB（`Field_json : Field_blob`），row 格式下 BLOB 值**只能全量打包**——`pack_field()`（`rpl_record.cc:101`）里只有 `UPDATE_AI` 且 `pack_diff` 成功才走 diff，否则 fallthrough 到 `pack_with_metadata_bytes` 写全量；
2. 默认 `binlog_row_image=FULL`（`table.cc:5729-5731` 直接 `bitmap_set_all(read_set/write_set)`）：一条 UPDATE 的 BI 和 AI 都含整个 JSON 文档 → 约 **2× 文档大小**进 binlog；MINIMAL 下被修改的 JSON 列的 BI/AI 也仍是全量；NOBLOB 只是"不修改就不记 BLOB"；
3. binlog 是 server 层**逻辑日志**，必须自包含可重放，从库没有主库的物理页，无法享受 `lob::update()` 那种"只改 1~2 个 LOB 页"的物理增量。所以 redo/undo 因部分更新大幅缩小，binlog 默认不缩小——这是"物理优化"与"逻辑复制"的天然鸿沟。

**量化示例**：1 MB 的 off-page JSON，改其中 50 字节——redo 侧只有 ~60 B（`MLOG_WRITE_STRING`），而 binlog 默认要写 ~2 MB（BI + AI）。开 `binlog_row_value_options=PARTIAL_JSON` 后 AI 只写 diff（`operation + path + value`，可能只有几十字节），配合 `binlog_row_image=MINIMAL/NOBLOB` 收益最大。

- 事件类型：`PARTIAL_UPDATE_ROWS_EVENT = 39`（`libbinlogevents/include/binlog_event.h:354`），选择逻辑 `log_event.cc:12533`（只要 `binlog_row_value_options != 0` 就用 39，哪怕本行没有一列真正写成 diff）。
- **binlog 里存的是逻辑 JSON diff，不是物理 Binary_diff**，且是逐列决定的混合格式：diff 不比全量小就写全量（`Field_json::get_diff_vector_and_length` `field.cc:7956`）。**PARTIAL_JSON 只作用于 AI（前像 BI 不受影响）**——`pack_field()` 仅在 `row_image_type == UPDATE_AI` 时才尝试 `pack_diff`（`rpl_record.cc:110`），且值未变时写"空 diff 向量"（`field.cc:7947-7953`）。
- 行内布局（`rpl_record.cc:304-331`）：先写 `value_options`（无 diff 则写 0），再写 partial_bits（每个 JSON 列 1 bit）。
- diff 二进制格式（`json_diff.cc:148`）：`1 字节 operation | length-encoded path | length-encoded JSON binary value`（REMOVE 不写 value）。
- **从库 apply**：`unpack_row`（`rpl_record.cc:508-568`）→ 标记 JSON 列可部分更新 → `Field_json::unpack_diff()`（`field.cc:7983`）→ `apply_json_diffs()`（`json_diff.cc:403`）→ **优先再走一次 `attempt_binary_update()` 就地改并收集 Binary_diff**，失败才转 DOM 全量。因此**从库的 InnoDB 同样能走 LOB 部分更新**。
- diff 应用不上 → `ER_COULD_NOT_APPLY_JSON_DIFF`（通常意味着主从不同步）；格式损坏 → `ER_CORRUPTED_JSON_DIFF`。
- 注意：**Table_map 的 optional metadata 里没有 PARTIAL_JSON 项**（`rows_event.h:547-571`），信息由新事件类型 + value_options + partial_bits 承载。

---

## 十一、InnoDB 存储层：JSON 就是 BLOB

`get_innobase_type_from_mysql_type`（`ha_innodb.cc:7996`）：

```cpp
case MYSQL_TYPE_JSON:  // JSON fields are stored as BLOBs
  return (DATA_BLOB);
```

即：**InnoDB 完全不知道 JSON 的存在**，只知道这是一个 BLOB。因此：

- 超过 ~8KB（页半满规则）会 off-page 存储，聚簇索引行里只留 20 字节 BLOB ref（REDUNDANT/COMPACT 下还留 768 字节本地前缀）；
- 所有 JSON 语义（解析、路径、类型、比较）都在 server 层完成；
- 部分更新是唯一打通两层的机制：server 把 `Binary_diff` 塞进 `upd_t`，InnoDB 照着改 LOB 页。

> **off-page 之后的物理实现（first page / data page / index entry / lob_version / purge 回收）属于 InnoDB LOB 子系统，已独立到 [`innodb/lob.md`](../../innodb/lob.md)。** 这里只保留与 JSON 直接相关的结论。

> **GeoJSON / 空间类型与 JSON 共享解析器但存储独立，已独立到 [`server/gis.md`](gis.md)。**

---

## 十二、redo 与崩溃恢复中的 JSON / LOB

JSON 部分更新引入了"字节区间级的部分写 + 页级 COW + undo 记增量"，崩溃后如何保证一致性？结论是：**靠 mtr 原子性 + 事务回滚/purge，没有任何 LOB 专属的恢复逻辑。**

### 12.1 分叉点：内联 JSON 与 off-page JSON 的 redo 完全不同

全枚举 `mlog_id_t` 在 `mtr0types.h:63-274`，**没有 `MLOG_LOB_*`**。要搞清楚 JSON 更新产生什么 redo，先要过一个闸门 —— `row_upd_changes_field_size_or_external()`（`row0upd.cc:407`，判定在 `:476`）：

```cpp
// row0upd.cc:476
if (dfield_is_ext(new_val) || old_len != new_len ||
    rec_offs_nth_extern(index, offsets, upd_field->field_no)) {
  return true;      // ← 不能走 in-place
}
```

三个条件任一成立就不能原地更新。再叠加"记录里有外部列时乐观更新直接放弃"（`btr0cur.cc:3736` 返回 `DB_OVERFLOW`），于是有两条路：

| | **情形 A：小 JSON，内联在聚簇记录里** | **情形 B：off-page JSON（LOB）** |
|---|---|---|
| 判定 | 新旧长度**相同** → 可 in-place | 必走**悲观 delete + insert** |
| 聚簇记录 redo | `MLOG_REC_UPDATE_IN_PLACE`（**70**，`mtr0types.h:264`） | `MLOG_REC_DELETE`（69）+ `MLOG_REC_INSERT`（67） |
| 写入函数 | `btr_cur_update_in_place_log`（`btr0cur.cc:3305`） | `page_cur_delete_rec`（`page0cur.cc:2245`）/ `page_cur_insert_rec_write_log`（`:977`） |
| 变长时 | **值变长 ⇒ `old_len != new_len` ⇒ 也退化成 delete+insert** | 同左 |
| 大数据 | —— | LOB 页：`MLOG_WRITE_STRING`；ref 的 version：`MLOG_4BYTES` |

**两个很容易踩的坑**：

1. **JSON 值变长，即使还在"内联"范围内，也必须 delete+insert**，不是 in-place。这对短 JSON 的高频更新影响很大。
2. **`MLOG_REC_UPDATE_IN_PLACE` 记录的是字段新值全量，不是 diff** —— `row_upd_index_write_log`（`row0upd.cc:644`）逐字段写 `field_no / len / 新值`。所以 in-place 路径下 redo 量仍与 JSON 值大小成正比。

> ⚠️ **纠错（本轮发现）**：8.0.30 之后**写入侧已不再用 `MLOG_COMP_REC_*_8027`**。那批常量（`mtr0types.h:171/182/185`）全库只在两处出现：定义处和恢复解析 `log0recv.cc:2021/4142`，**写入路径（`btr/`、`page/`、`lob/`）零引用**。现在的写入侧类型是 `MLOG_REC_INSERT/DELETE/UPDATE_IN_PLACE`（67/69/70），因为引入了 `mlog_open_and_write_index()` 把 index 元信息内联进 redo 记录，不再需要区分 COMPACT/REDUNDANT 两套类型。

### 12.2 LOB 侧的 redo：全是通用类型

| LOB 动作 | redo 类型 | 位置 |
|----------|-----------|------|
| `replace_inline` 原地改写 | `MLOG_WRITE_STRING`(30) | `lob0pages.cc:45`、`lob0first.cc:43` |
| `replace`（COW） | 新页分配 + **3 条** `MLOG_WRITE_STRING`（前缀/新数据/后缀） | `lob0pages.cc:83/91/101` |
| `lob_version` 递增 | `MLOG_4BYTES`(4) | `lob0first.cc:392` |
| 行内 ref 的 version 字段 | `MLOG_4BYTES` | `lob0lob.h:449` `ref_t::set_offset()` |
| 页类型 / `data_len` | `MLOG_2BYTES`(2) / `MLOG_4BYTES` | `lob0pages.h:79` |
| 6 字节 trx_id | `mlog_log_string` → 也是 `MLOG_WRITE_STRING` | `lob0index.h:312` |

恢复分发总入口 `recv_parse_or_apply_log_rec_body()`（`log0recv.cc:1582`，`switch` 在 `:1730`）：
`MLOG_WRITE_STRING` → `:2276`，`MLOG_4BYTES` → `:1736`，`MLOG_2BYTES` → `:1805`，`MLOG_UNDO_INSERT` → `:2177`。

### 12.3 redo 记录的二进制格式

公共头由 `mlog_write_initial_log_record_low()`（`mtr0log.ic:173`）写入：

```
[type:1][space_id:1~5][page_no:1~5]      上界常量 REDO_LOG_INITIAL_INFO_SIZE = 11 (mtr0log.h:61)
```

（`space_id`/`page_no` 用 `mach_write_compressed`：<0x80→1B，<0x4000→2B，…）

| 函数 | 布局 | 总长 |
|------|------|------|
| `mlog_write_ulint`（`mtr0log.cc:256`） | `[type][space][page_no][page_offset:2][val:1~5]` | 6~18 B |
| `mlog_write_string` / `mlog_log_string`（`mtr0log.cc:327/342`） | `[type][space][page_no][offset:2][len:2]` + **数据体** | 头 7~15 B（典型 9）+ N |

注意 `mlog_log_string` 里数据体是 `mlog_catenate_string()`（`:366`）追加到 dyn buffer 的，**不占** `mlog_open(mtr, 30, ...)` 那 30 字节预算。

### 12.4 部分更新到底省了多少 redo（量化）

LOB 页有效载荷（`lob0pages.h:97`）：

```cpp
static ulint payload() { return (UNIV_PAGE_SIZE - LOB_PAGE_DATA - FIL_PAGE_DATA_END); }
//                             = 16384 - 49 - 8 = 16327 字节
```

所以 **1 MB 的 JSON ≈ 65 个 LOB 页**。以"改其中 50 字节"为例：

| 方案 | 新分配 LOB 页 | redo 量级 | 相对量 |
|------|--------------|-----------|--------|
| **部分更新（≤100B，inline）** | **0** | LOB 侧 **~60 B**（1 条 `MLOG_WRITE_STRING` = 9 + 50）；含聚簇记录与 undo 合计约 1 KB | **1×** |
| 部分更新（>100B，COW 碰 1 页） | 1 | ~16 KB（整页拷贝 3 段） | ~270× |
| **全量重写（1MB）** | **65** | **~1.05 MB**（每页 `mlog_write_string` 一次） | **~17,000×** |

**结论：部分更新的收益本质是"不重写 LOB 页"，redo 量因此与文档大小解耦。** 这也是为什么官方/阿里云实测 TPS 能有几倍提升 —— 瓶颈从"写 1MB redo + 分配 65 个页"变成"写 60 字节 redo"。

全量重写时 `lob::insert()`（`lob0impl.cc:929`）还会**每 4 页提交一次 mtr**（`:1048` `commit_freq`），65 页 ≈ 17 次 mtr 提交 —— 这也解释了为什么它比部分更新慢得多（每次提交都要走 log_buffer_reserve + 刷脏链）。

### 12.5 redo 与 undo 如何分工

| | redo | undo |
|---|---|---|
| 内容 | **物理/页级**：哪些页的哪些字节变成什么 | **逻辑**：怎么撤销（列旧值 / LOB diff 的 `old_data`） |
| 目标 | 把页重放到崩溃那一刻（**已提交和未提交的修改都在**） | 把未提交事务的修改回滚掉 |
| JSON/LOB | LOB 页按字节写 + ref 用 `MLOG_4BYTES` | 小改动存 `old_data`；大改动**不记**（`N=0`），靠 LOB 多版本 |
| 大小 | 与改动字节数成正比 | 外部列在 undo 里只有本地前缀 + 20B ref，**1MB 的 JSON 绝不会进 undo** |

一个容易忽略的点：**undo 页本身也是 Buffer Pool 里的页，写它同样要记 redo** —— `MLOG_UNDO_INSERT`(20，`mtr0types.h:112`)，写入函数 `trx_undof_page_add_undo_rec_log()`（`trx0rec.cc:69`）。所以"记 undo"这件事本身也有 redo 开销。

### 12.6 原子性：一次 UPDATE 的所有 LOB 修改落在同一个 mtr

`lob::update()` 用的 mtr 就是调用方传下来的 **`btr_mtr`**（`lob0update.cc:100` `ctx.get_mtr()`）。所有 `Binary_diff`（可跨多个 LOB 数据页，每个 diff 最多跨 2 个 index entry）的写入，加上最后 `blobref.set_offset(lob_version, mtr)`（`:160`），**全在同一个 mtr 内**。

`BtrContext::check_redolog_normal()` 会打断 mtr，但在 UPDATE 路径只在进入循环前调用一次（`lob0lob.cc:443`），diff 循环里不调用 —— 所以不会因为 redo 量而中途提交。

崩溃时 redo 按 mtr（以 `MLOG_MULTI_REC_END` / `MLOG_SINGLE_REC_FLAG` 定界）整体 apply，**要么全做要么全不做**。

### 12.7 恢复顺序（`srv0start.cc`）

| 步骤 | 行号 |
|------|------|
| redo 扫描 | `srv0start.cc:1988` |
| redo apply | `:2036` |
| 重建事务 + purge 队列 | `:2186` |
| DD 事务先回滚 | `:2384-2394` |
| 普通未提交事务回滚（后台线程） | `:2534-2545` → `trx_recovery_rollback_thread` |
| purge 线程启动 | `:2443` |

**回滚入口在 `trx0roll.cc` 而不是 `trx0trx.cc`**：

- `trx_recovery_rollback_thread()` `trx0roll.cc:851`
- `trx_rollback_or_clean_recovered(bool all)` `trx0roll.cc:712`
- `trx_rollback_or_clean_resurrected(trx, all)` `trx0roll.cc:765`

> `trx_rollback_or_clean_all_recovered()` **不是真实符号**，只在注释里出现。

### 12.8 回滚时如何还原 LOB

```
row_undo_step (row0undo.cc:347)
 └─ row_undo_mod (row0umod.cc:1272)
     └─ row_undo_mod_parse_undo_rec (:1202)
         └─ trx_undo_update_rec_get_update (trx0rec.cc:1740)   ← 传 lob_undo = nullptr (:1249)
             └─ trx_undo_read_blob_update (trx0rec.cc:904)     ← 门闸 is_lob_undo() && is_lob_updated() (:1883)
     └─ row_undo_mod_clust_low (:84)
         └─ btr_cur_pessimistic_update (btr0cur.cc:3942)
             ├─ lob::mark_not_partially_updatable (btr0cur.cc:4089)
             └─ BtrContext::free_updated_extern_fields (lob0lob.cc:957)
                 └─ lob::purge (lob0lob.cc:1009) → rollback (lob0purge.cc:65)
                     ├─ 有 diff → rollback_from_undolog (:73) → apply_undolog (lob0update.cc:600)
                     └─ 无 diff → make_old_version_current (lob0index.cc:47)
```

关键点：

1. **回滚路径不构造 `undo_vers_t`**（`row0umod.cc:1251` 传 `nullptr`），因此 `trx_undo_read_blob_update` 走"跳过 old_data"分支（`trx0rec.cc:952`），只记 offset/length/**指针**，不拷贝 —— 因为回滚是就地写回，不需要把旧值搬到内存。
2. **分叉判据是"undo 里有没有 diff 向量"**（`lob0purge.cc:73`），**不是重新计算字节数**。阈值 100 只在正向（写 undo + 执行更新）时判定一次，结果被固化进 undo 记录。
3. 外部列更新一定走 **pessimistic**（`btr_cur_optimistic_update` 遇到 extern 直接返回 `DB_OVERFLOW`，`btr0cur.cc:3727`）。
4. **大改动回滚不回退 `lob_version` 计数器**（`apply_undolog` 只还原 `last_trx_id`/`last_undo_no`，`lob0update.cc:623-624`），靠 index entry 的 versions 链恢复可见性。
5. **LOB 回滚本身不是原子的**：`rollback()` 用 `local_mtr` 每轮提交，靠 `ref.set_length(0)`（长度置 0）做"部分删除"哨兵，源码预留 3 个 crash 注入点（`lob0purge.cc:139/171/183`）。这是**幂等/可重入**设计。

### 12.9 `TRX_UNDO_MODIFY_BLOB` 标志

```cpp
// include/trx0rec.h:312-320
constexpr uint32_t TRX_UNDO_CMPL_INFO_MULT = 16;
constexpr uint32_t TRX_UNDO_MODIFY_BLOB   = 64;   // bit6：undo 支持 BLOB 部分更新格式
constexpr uint32_t TRX_UNDO_UPD_EXTERN    = 128;  // bit7：undo 涉及外部存储列
```

⚠️ 在 8.0.39 中 `TRX_UNDO_MODIFY_BLOB` 是**无条件置位**的（`trx0rec.cc:1244`），它只表示"这条 undo 用了新格式"，**不代表 undo 里真有 Lob diff**。真正判据是 `uf->lob_diffs != nullptr && size() > 0`。

进入 `trx_undo_read_blob_update()` 的门闸是 `is_lob_undo() && is_lob_updated()`（`trx0rec.cc:1883`）。

### 12.10 binlog / XA 恢复：没有 JSON 特殊处理

- `Binlog_recovery::recover()`（`sql/binlog/recovery.cc:64-91`）只处理 `QUERY_EVENT` / `XID_EVENT` / `XA_PREPARE_LOG_EVENT`，**携带 JSON diff 的 ROWS 事件直接 `default: break` 忽略**；
- `MYSQL_BIN_LOG::recover()`（`binlog.cc:7883`）只做 XID 收集、`ha_recover()`、按 `valid_pos` 截断 binlog，**从不把 binlog 事件回放到 InnoDB**；
- InnoDB 侧 `innobase_xa_recover()`（`ha_innodb.cc:20102`）→ `trx_recover_for_mysql()`（`trx0trx.cc:3253`）只收集 `TRX_STATE_PREPARED` 的事务，随后 commit/rollback 走的就是普通路径（即 12.4 那条栈）。

三种情形：binlog 有 XID → 提交；有 PREPARE 无 XID → 回滚（LOB diff 反向 apply 或 versions 链扶正）；binlog 无记录 → 回滚。**均无 JSON 分支。**

### 12.11 恢复后没有 LOB 校验

`log0recv.cc` / `srv0start.cc` 中无任何 LOB 校验；`lob/` 下所有 `validate*` 都是 `UNIV_DEBUG`/`ut_ad` 级别，release 构建不生效。唯一带修复语义的 LOB 处理在 **IMPORT TABLESPACE**（`row0import.cc:2565-2584`）。

**一致性完全依赖 mtr 原子性 + 事务回滚/purge，而非事后校验。**

---

## 十三、全链路串联：一条 UPDATE 的一生

用一条语句把所有模块串起来（这也是检验理解是否闭环的最好方式）：

```sql
UPDATE t SET j = JSON_REPLACE(j, '$.a', 12345) WHERE id = 7;
-- 假设 j 是 off-page 大 JSON，改动 ≤100 字节
```

### 阶段 1：解析与优化（server / SQL 层）

```
mysql_parse → mysql_execute_command → Sql_cmd_update::execute
 → prepare_partial_update (sql_update.cc:1346)          ← 判定能否部分更新
     ├─ 列必须是 MYSQL_TYPE_JSON
     ├─ 引擎必须 HA_BLOB_PARTIAL_UPDATE (ha_innodb.cc:2982)
     └─ Item_func_json_replace::supports_partial_update (item_json_func.cc:2332)
          → can_use_in_partial_update() == true（只有 SET/REPLACE/REMOVE）
          → 且源列 == 目标列
 → TABLE::mark_column_for_partial_update (table.cc:7495)
 → TABLE::setup_partial_update (table.cc:7560)          ← 打开 diff 收集
```

### 阶段 2：执行，逐行计算并收集 diff

```
Sql_cmd_update::update_single_table → ... → 求值新值
 → Item_func_json_set_replace::val_json (item_json_func.cc:2368)
     → Json_wrapper::attempt_binary_update (json_dom.cc:3439)
         → space_needed (json_binary.cc:1538)      新值要占多少字节
         → Value::has_space (:1410)                现有空洞够不够
         → Value::update_in_shadow (:1727)         在 record[0] 上就地改
             → TABLE::add_binary_diff (table.cc:7626)  登记 {offset, length}
     → 若 partially_updated == false → disable_binary_diffs (回退全量)
 → update_generated_write_fields (table.cc:7225)   重算依赖的生成列（含多值索引列）
 → Field_json::store_binary (field.cc:7699)         写入 record[0]
```

### 阶段 3：交给引擎

```
ha_innobase::update_row → calc_row_difference (ha_innodb.cc:9808)
 → row_upd (row0umod.cc:160)
 → btr_cur_pessimistic_update (row0upd.cc:2943)     ← 有 extern 列必走 pessimistic
 → lob::btr_store_big_rec_extern_fields (lob0lob.cc:410)
     → upd->is_partially_updated(field_no)? → lob::update (lob0update.cc:96)
         → 小改动 → replace_inline (:501)  mlog_write_string 原地覆盖
         → blobref.set_offset(lob_version, mtr) (:160)
```

### 阶段 4：记 undo

```
trx_undo_page_report_modify → trx_undo_report_blob_update (trx0rec.cc:1000)
 → 小改动：写 N 个 Lob diff（offset / length / old_data + 受影响的 index entry 旧 modifier）
 → 大改动：直接写 N = 0，什么都不记（靠 lob_version 保留旧页）
 → 置 TRX_UNDO_MODIFY_BLOB + TRX_UNDO_UPD_EXTERN
```

### 阶段 5：提交与 binlog

```
ha_commit_trans
 → prepare (InnoDB prepare)
 → MYSQL_BIN_LOG::ordered_commit (binlog.cc:8853)      ← 见 server/replication/binlog.md
     → flush/sync 阶段写 Rows_log_event
         → Field_json::pack_diff (field.cc:7858)        ← 若 binlog_row_value_options=PARTIAL_JSON
             写逻辑 diff（operation + path + value）
             否则 pack_field 写全量
 → commit (InnoDB commit)
```

### 阶段 6：读（另一个事务，可能读旧版本）

```
row_search_mvcc → row_sel_build_prev_vers_for_mysql (row0sel.cc:3071)
 → trx_undo_prev_version_build (trx0rec.cc:2469)
     → trx_undo_read_blob_update (:904)                构造 undo_vers_t（拷贝 old_data）
 → row_sel_store_mysql_field (:2725)
     → lob::read (lob0impl.cc:1074)                    按 lob_version 选 entry/页
     → lob_undo->apply (:2818) → undo_data_t::apply (lob0undo.cc:41)  打旧字节补丁
```

### 阶段 7：purge 回收旧版本

```
row_purge_record_func (row0purge.cc:1096)
 → lob::purge (lob0purge.cc:414)
     → 逐 versions 链 → can_be_purged (lob0index.h:178) → purge_version (lob0index.cc:111)
     → first page 延迟到 purge batch 末尾释放 (trx0purge.cc:2561)
```

### 阶段 8：从库回放（若开启复制）

```
Rows_log_event::do_apply_event → unpack_row (rpl_record.cc:508)
 → Field_json::unpack_diff (field.cc:7983) → apply_json_diffs (json_diff.cc:403)
     → 再走一次 attempt_binary_update → 重新收集 Binary_diff
 → ha_update_row → 回到阶段 3（从库也享受 LOB 部分更新）
```

### 阶段 9：崩溃（若在第 3~4 步之间）

```
重启 → recv_recovery_from_checkpoint_start (srv0start.cc:1988)
     → recv_apply_hashed_log_recs (:2036)       按 mtr 整体重放（原子）
     → trx_recovery_rollback_thread (:2534)     未提交事务回滚
         → row_undo_mod → lob::rollback → apply_undolog / make_old_version_current
```

---

## 核心调用栈

**写入（文本 → JSONB → BLOB）**

```
Field_json::store (field.cc:7633)
  → ensure_utf8mb4 (item_json_func.cc:81)
  → Json_dom::parse (json_dom.cc:562)           rapidjson SAX → DOM
  → json_binary::serialize (json_binary.cc:132)
      → serialize_json_value (:710)
          → serialize_json_object (:564) / serialize_json_array (:501)
          → serialize_opaque (:644) / serialize_decimal (:663) / serialize_datetime (:681)
  → Field_json::store_binary (field.cc:7699)
  → Field_blob::store
```

**读取（字段 → Json_wrapper ≈ 零拷贝）**

```
Field_json::val_json (field.cc:7788)
  → json_binary::parse_binary (json_binary.cc:1068)
      → parse_value (:1053) → parse_array_or_object (:1012) / parse_scalar (:920)
  → Json_wrapper(json_binary::Value)             不建 DOM
```

**路径取值**

```
Item_func_json_extract::val_json (item_json_func.cc)
  → parse_path (item_json_func.cc:355)
  → Json_wrapper::seek (json_dom.cc:1558)
      → seek_no_dup_elimination (:1949)
          → seek_member (:1960) / seek_array_cell (:1999)
          → seek_array_range (:2021) / seek_ellipsis (:2052) / seek_end (:2071)
```

**部分更新（主库）**

```
prepare_partial_update (sql_update.cc:1346)          ← 能力判定
  → Item_json_func::supports_partial_update (item_json_func.cc:2332)
  → mark_for_partial_update (:2315) / TABLE::mark_column_for_partial_update (table.cc:7495)
TABLE::setup_partial_update (table.cc:7560)          ← 打开收集
Item_func_json_set_replace::val_json (item_json_func.cc:2368)
  → Json_wrapper::attempt_binary_update (json_dom.cc:3439)
      → space_needed (json_binary.cc:1538)
      → Value::has_space (:1410)
      → Value::update_in_shadow (:1727)  → TABLE::add_binary_diff (table.cc:7626)
ha_innobase::update_row
  → calc_row_difference (ha_innodb.cc:9808)
  → lob::btr_store_big_rec_extern_fields (lob0lob.cc:410)
      → lob::update (lob0update.cc:96)
          → replace_inline (:501) / replace (:268)
  → trx_undo_report_blob_update (trx0rec.cc:1000)
```

**部分更新（binlog → 从库）**

```
Rows_log_event::do_apply_event
  → unpack_current_row (log_event.cc:7985)
  → unpack_row (rpl_record.cc:508)                 标记 JSON 列可部分更新 + setup_partial_update
  → unpack_field (rpl_record.cc:171)
      → Field_json::unpack_diff (field.cc:7983)
          → Json_diff_vector::read_binary (json_diff.cc:282)
          → apply_json_diffs (json_diff.cc:403)
              → Json_wrapper::attempt_binary_update (json_dom.cc:3439)  再次收集 Binary_diff
              → 失败则 Json_dom 全量
  → ha_update_row → (同主库) lob::update
```

**多值索引写入**

```
Field_typed_array::store_array (field.cc:9823)
  → coerce_json_value (field.h:4277) → remove_duplicates
  → ha_mv_key_capacity (ha_innodb.cc:23862)        容量校验
  → store_json
  → innobase_get_multi_value (ha_innodb.cc:8972)
      → innobase_store_multi_value_low (:8795)     数组 → multi_value_data
  → row_ins_sec_index_multi_value_entry (row0ins.cc:3260)
      → Multi_value_entry_builder::next (row0row.h:353)   循环 N 次
          → row_ins_sec_index_entry
```

**多值索引查询**

```
Item_func::gc_subst_transformer (item_func.cc:1300)
  → gc_subst_overlaps_contains (:1207) / substitute_gc_expression (:1123)
  → get_func_mm_tree (range_analysis.cc:553)
      → get_func_mm_tree_from_json_overlaps_contains (:463)   N 个 EQ 的 OR
      → MEMBER OF (:621)                                      1 个 EQ
  → get_mm_parts → range scan
  → handler::filter_dup_records (handler.cc:8483)            去重
```

**JSON_TABLE**

```
PT_table_factor_function::contextualize (parse_tree_nodes.cc:1282)
  → new Table_function_json (table_function.h:320)
GetTableAccessPath (sql_executor.cc:1753)
  → NewMaterializedTableFunctionAccessPath
MaterializedTableFunctionIterator::Init (composite_iterators.cc:1965)
  → create_materialized_table (sql_derived.cc:1636)
  → fill_result_table (table_function.h:397)
  → 临时表扫描
```

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `binlog_row_value_options` | `''`（可设 `PARTIAL_JSON`） | Global/Session | binlog 中 JSON 列只记 diff。要求 binlog 打开 + ROW 格式 + 非 v1 row events，否则只告警不报错（`sys_vars.cc:7099`）。定义 `sys_vars.cc:7162`，常量 `PARTIAL_JSON_UPDATES=1`（`system_variables.h:74`） |
| `binlog_row_image` | FULL | Global/Session | 设为 MINIMAL/NOBLOB 才能与 PARTIAL_JSON 叠加出最大收益（`sys_vars.cc:1467` 提示） |
| `binlog_format` | ROW(8.0) | Global/Session | STATEMENT 下 PARTIAL_JSON 无效并告警（`sys_vars.cc:1402`） |
| `max_allowed_packet` | 64MB | Global/Session | JSON 文档的实际软上限，超限报 `ER_WARN_ALLOWED_PACKET_OVERFLOWED`（`json_binary.cc:857`） |
| `log_bin_use_v1_row_events` | OFF | Global | ON 时禁用 PARTIAL_JSON |
| `character_set_client/connection/results` | utf8mb4 | Global/Session | 影响 JSON 文本的字符集转换与非 ASCII 处理 |
| `end_markers_in_json` | OFF | Global/Session | 只影响 `EXPLAIN FORMAT=JSON` 与 optimizer trace 输出，与 JSON 数据类型无关（`sys_vars.cc:3535`） |

### 状态变量

**没有 JSON 专用的状态计数器**（无 `Com_json_*`）。观察 JSON 负载只能间接通过：

| 变量 | 说明 |
|------|------|
| `Com_update` / `Com_select` | JSON 相关语句计数走通用 SQL command 计数 |
| `Innodb_row_lock_*` / `Handler_*` | 部分更新不减少 handler 层计数，只看得出 row 级行为 |
| `Bytes_sent` | 主库侧**包含 dump 线程发给从库的字节数**，开 PARTIAL_JSON 后会下降（因为发的是 diff 不是全量） |

> 排查部分更新是否生效，最实用的三个办法：①对比 `JSON_STORAGE_SIZE`（是否稳定）与 `JSON_STORAGE_FREE`（是否下降）；②用 `SHOW BINARY LOGS` 对比更新前后的 binlog 文件增长量（开/关 `binlog_row_value_options` 各跑一次）；③源码层面看 `TABLE::add_binary_diff`（`table.cc:7626`）是否被调用。
>
> 注意：`Bytes_sent` **不能直接当 binlog 体积的指标**（它是发给所有客户端的总字节数），看 binlog 体积应当直接看 binlog 文件大小。

---

## Misc

### JSON 与 FTS 的存储形态对比

两者都处理"非结构化内容"，但存储路线**完全相反**，这是最容易搞混的地方：

| 维度 | **JSON** | **FTS** |
|------|----------|---------|
| 载体 | **列值本身**（JSONB 编码后存在 BLOB 列里） | **11 张独立辅助表** |
| 格式谁定义 | server 层（`sql-common/json_binary.h`） | InnoDB（`storage/innobase/fts/`） |
| InnoDB 理解内容吗 | **不理解**，当 BLOB | **理解**，自己维护倒排表 |
| 有"辅助表"吗 | **没有** | 有：6 张 `index_N` + 5 张公共表 |
| 有内存缓存吗 | 没有（读就是读页） | 有：`fts_cache`（每表一份内存倒排表） |
| 有后台线程吗 | 没有 | 有：`fts_optimize_thread`（sync + optimize） |
| 写入立即落盘吗 | 是（随事务写 BLOB） | **否**（先进 cache，后台 sync） |
| 删除立即生效吗 | 是 | **否**（写墓碑表，OPTIMIZE 才真删） |
| 崩溃后怎么恢复 | 就是普通 BLOB，靠 redo | 靠 CONFIG 的 `synced_doc_id` 懒重建 cache |
| 系统变量 | `binlog_row_value_options` 等（见上节） | `innodb_ft_*` 13 个 + `ft_*` 5 个 |
| 状态变量 | **无** | **无**（都靠 I_S 表看） |
| 可观测手段 | `JSON_STORAGE_SIZE` / `JSON_STORAGE_FREE` | `INNODB_FT_INDEX_TABLE/CACHE/DELETED/CONFIG` 等 6 张 |
| 支持部分更新吗 | **支持**（Binary diff + LOB 部分写） | 不支持（改一次 = 删旧文档 + 插新文档） |

一句话概括：**JSON 是"我把内容塞进 BLOB，引擎别管"；FTS 是"引擎自己建一套索引表来管"**。

### 术语表

| 术语 | 含义 |
|------|------|
| **JSONB** | MySQL 内部的 JSON 二进制表示（注意：与 PostgreSQL 的 jsonb 不是同一格式） |
| **DOM** | Document Object Model，JSON 的内存树形表示（`Json_dom` 子类） |
| **Binary diff** | 物理层增量：`{offset, length}`，发给引擎 |
| **Logical diff / Json_diff** | 逻辑层增量：`{REPLACE/INSERT/REMOVE, path, value}`，写进 binlog |
| **LOB** | Large Object，InnoDB 8.0 重构后的大对象存储（first page + data page + index entry）—— 完整内容见 [`innodb/lob.md`](../../innodb/lob.md) |
| **lob_version** | LOB 首页上的版本号，MVCC 用它区分同一 LOB 的不同版本 |
| **multi-valued index** | 多值索引，一个 JSON 数组展开成 N 条二级索引记录 |
| **auto-wrapping** | 对非数组值应用 `[0]` 时隐式包成单元素数组（MySQL 扩展） |
| **opaque** | JSONB 中的"自定义二进制"类型，带 1 字节 `enum_field_types`；DECIMAL/DATETIME 都包装成它 |
| **inline value** | 小标量直接塞进 value entry 的 offset 字段，不占数据区 |

### 易混淆概念对比

| 对比项 | 说明 |
|--------|------|
| **JSON 的 charset() vs sort_charset()** | `charset()` 返回 utf8mb4_bin（对外表现为字符串），`sort_charset()` 返回 binary（排序不做字符集转换） |
| **`->` vs `->>`** | `->` = `JSON_EXTRACT`（返回 JSON 类型，带引号）；`->>` = `JSON_UNQUOTE(JSON_EXTRACT(...))`（返回字符串） |
| **`JSON_MERGE` vs `JSON_MERGE_PRESERVE` vs `JSON_MERGE_PATCH`** | `JSON_MERGE` 是 `JSON_MERGE_PRESERVE` 的废弃别名；PRESERVE 保留重复 key（数组化），PATCH 遵循 RFC 7396（后者覆盖前者，值为 null 表示删除） |
| **`JSON_STORAGE_SIZE` vs `JSON_STORAGE_FREE`** | 前者 = 文档当前占用字节；后者 = 文档中被"空洞"浪费的字节（部分更新后可能变小）。两者都是 JSONB 目录区设计的产物 |
| **Binary diff vs Logical diff** | 前者是物理字节区间（引擎用），后者是路径+值（binlog 用）。**主库发给引擎的一定是 Binary diff；binlog 里的一定是 Logical diff** |
| **部分更新"能省"的前提** | **文档总长不变** —— 部分更新只复用已有空间，从不重新分配。因此：新增 key / 值变大到**超过原 value 占的空间** / 值区没有足够空洞 → 自动退化为全量重写，SQL 不报错但性能回退。注意"值变大"本身**不必然**退化，只要还在原 value 的空间内就可以 |
| **多值索引 vs 生成列索引** | 一行对 N 条 vs 一行对 1 条；前者只能 MEMBER OF/CONTAINS/OVERLAPS 且不支持排序 |
| **`0x80` 位** | 是变长长度的续行位，**不是** large object 标记位。large/small 写在 type tag 最低位（0x0 vs 0x1） |

### FAQ

**Q：为什么 MySQL 的 JSONB 没有版本号，而 PostgreSQL 有？**
A：MySQL 8.0 的 type tag 空间是 0x0~0xF，只预留了 0xD/0xE。`serialize` 不写版本字节（`json_binary.cc:138`），`parse_binary` 也不读（`:1080`）。**注意一个常见误传：8.3 并没有改 JSON 数据类型的格式** —— 8.3 的 `explain_json_format_version`（WL#15684）改的是 `EXPLAIN FORMAT=JSON` 的**输出**格式，与 JSONB 无关；8.3.0 release notes 中 JSON 相关条目只有两个 bug 修复。因此在这条版本线上，JSONB 格式是稳定的；将来要引入版本化只能复用 0xD/0xE 两个保留 tag。

**Q：部分更新为什么只支持 JSON_SET / JSON_REPLACE / JSON_REMOVE？**
A：因为只有这三个函数的语义能映射成"已有位置的值替换 / 已有位置的值删除"，且**不需要改变文档结构**（不新增成员、不新增数组元素）。`JSON_INSERT` 要新增成员、`JSON_ARRAY_APPEND` 要改变数组长度，都会打破"定长目录 + 数据区"的布局，必须重写。

**Q：怎么判断一条 UPDATE 是否真的走了部分更新？**
A：三条路径 —— ①看 `JSON_STORAGE_SIZE` 是否稳定、`JSON_STORAGE_FREE` 是否下降；②开 `binlog_row_value_options=PARTIAL_JSON` 后看 binlog 体积；③源码层面看 `TABLE::add_binary_diff` 是否被调用（`table.cc:7626`）。

**Q：多值索引的重复行是怎么产生的？**
A：`[1,2,3]` 会产生 3 条二级索引记录，都指向同一个 rowid。`MEMBER OF` 命中其中 1 条，只返回 1 次；但 `JSON_OVERLAPS` 展开成多个 EQ range 的 OR 时，同一个 rowid 可能被多个 range 命中 → 需要 `Unique_on_insert` 去重（`handler.cc:8483`）。

**Q：JSON 列能不能建普通索引？**
A：不能，直接报错 `ER_JSON_USED_AS_KEY`（`sql_table.cc:4927`）。必须走生成列 / 函数索引 / 多值索引。

**Q：JSON 的更新是不是会产生大量 binlog？**
A：**默认会**。JSON 列是 BLOB，row 格式下 binlog 对 BLOB 只能全量打包（`rpl_record.cc:101` 的 `pack_field`），且默认 `binlog_row_image=FULL` 使 UPDATE 的 BI+AI 各含一份完整文档（约 2× 文档大小）。InnoDB 侧的部分更新优化（undo/redo 只记增量）救不了 binlog——binlog 是 server 层逻辑日志，必须自包含可重放。两条缓解手段：①`binlog_row_value_options=PARTIAL_JSON`（8.0.3+，WL#10497），AI 里 JSON 列改记 logical diff（`operation + path + value`），且"diff 不比全量小就自动回退全量"（`field.cc:7956`），对**大文档小改动**收益巨大；②`binlog_row_image=MINIMAL/NOBLOB` 压缩未修改列/未修改 BLOB 的 BI。注意 PARTIAL_JSON 只作用于 AI，BI 仍按 row image 规则走；STATEMENT 格式下则只记语句本身。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/field.h:3986` | `Field_json` 定义（继承 `Field_blob`） |
| `sql/field.cc:7633` | `Field_json::store`（解析 + 序列化 + 落库） |
| `sql/field.cc:7699` | `store_binary`（4GB 硬上限） |
| `sql/field.cc:7788` / `:7825` | `val_json` / `val_str` |
| `sql/field.cc:8035` / `:8046` / `:8054` | `cmp_binary` / `make_sort_key` / `make_hash_key` |
| `sql/field.h:4162` | `Field_typed_array`（多值索引载体） |
| `sql-common/json_binary.h:59-141` | **JSONB 格式文法注释** |
| `sql-common/json_binary.cc:54-71` | type tag 常量 |
| `sql-common/json_binary.cc:77-96` | entry / offset 尺寸常量 |
| `sql-common/json_binary.cc:132` | `serialize` 入口 |
| `sql-common/json_binary.cc:252` / `:285` | 变长长度写 / 读（0x80 续行位） |
| `sql-common/json_binary.cc:350` | `append_key_entries`（key 排序约束） |
| `sql-common/json_binary.cc:398` / `:451` | `inlined_type` / `attempt_inline_value` |
| `sql-common/json_binary.cc:564` / `:501` | `serialize_json_object` / `serialize_json_array` |
| `sql-common/json_binary.cc:644` / `:663` / `:681` | opaque / decimal / datetime 序列化 |
| `sql-common/json_binary.cc:920` / `:1012` / `:1053` / `:1068` | `parse_scalar` / `parse_array_or_object` / `parse_value` / `parse_binary` |
| `sql-common/json_binary.cc:1183` | `Value::lookup_index`（二分查找 key） |
| `sql-common/json_binary.cc:1410` / `:1538` / `:1727` / `:1914` | `has_space` / `space_needed` / `update_in_shadow` / `remove_in_shadow` |
| `sql-common/json_dom.h:151-172` | DOM 类层次图 |
| `sql-common/json_dom.h:343` / `:365` | `Json_key_comparator` / `Json_object_map` |
| `sql-common/json_dom.h:109-125` | `enum_json_type`（排序优先级） |
| `sql-common/json_dom.h:1161` | `Json_wrapper`（DOM/binary 统一视图） |
| `sql-common/json_dom.cc:122` | `Json_dom::operator new`（`key_memory_JSON`） |
| `sql-common/json_dom.cc:562` / `:637` | 文本→DOM / binary→DOM |
| `sql-common/json_dom.cc:1387` / `:1408` | `to_dom` / `to_binary` |
| `sql-common/json_dom.cc:2417` / `:2436` | `type_comparison` 矩阵 / `compare` |
| `sql-common/json_dom.cc:3117` / `:3258` / `:3365` | sort key 常量 / `make_sort_key` / `make_hash_key` |
| `sql-common/json_dom.cc:3439` | **`attempt_binary_update`（部分更新核心）** |
| `sql-common/json_path.h:52-92` | `enum_json_path_leg_type`（6 种） |
| `sql-common/json_path.h:323-356` | path 语法 EBNF |
| `sql-common/json_path.cc:258` / `:319` / `:341` / `:380` / `:428` / `:581` | 各解析函数 |
| `sql-common/json_path.cc:126` | `is_autowrap` |
| `sql/json_diff.h:53` | `enum_json_diff_operation`（REPLACE/INSERT/REMOVE） |
| `sql/json_diff.cc:148` / `:260` / `:282` / `:403` | diff 写/读/应用 |
| `sql/table.h:1319` | `Binary_diff` |
| `sql/table.cc:7560` / `:7626` / `:7686` | `setup_partial_update` / `add_binary_diff` / `new_data` |
| `sql/sql_update.cc:1346` | `prepare_partial_update`（触发判定） |
| `sql/item_json_func.h:151` | `Item_json_func` 基类 |
| `sql/item_json_func.cc:81` | `ensure_utf8mb4` |
| `sql/item_json_func.cc:2332` | `supports_partial_update` |
| `sql/item_json_func.cc:2368` | `Item_func_json_set_replace::val_json` |
| `sql/table_function.h:320` | `Table_function_json`（JSON_TABLE） |
| `sql/iterators/composite_iterators.cc:1965` | `MaterializedTableFunctionIterator::Init` |
| `sql/item_func.cc:1300` / `:1123` / `:1207` | 生成列替换（多值索引优化入口） |
| `sql/range_optimizer/range_analysis.cc:463` / `:621` | CONTAINS/OVERLAPS / MEMBER OF 的 range 构造 |
| `sql/handler.cc:8483` | `filter_dup_records`（多值索引去重） |
| `storage/innobase/handler/ha_innodb.cc:7996` | JSON → `DATA_BLOB` |
| `storage/innobase/handler/ha_innodb.cc:8795` | `innobase_store_multi_value_low` |
| `storage/innobase/row/row0ins.cc:3260` | `row_ins_sec_index_multi_value_entry` |
| `storage/innobase/include/row0row.h:318` / `:353` | `Multi_value_entry_builder` |
| `storage/innobase/lob/lob0lob.cc:410` / `:481` / `:550` | `btr_store_big_rec_extern_fields` 与部分更新判定 |
| `storage/innobase/lob/lob0update.cc:96` / `:268` / `:501` | `lob::update` / `replace` / `replace_inline` |
| `storage/innobase/lob/lob0lob.h:210` | `LOB_SMALL_CHANGE_THRESHOLD = 100` |
| `storage/innobase/trx/trx0rec.cc:1000` | `trx_undo_report_blob_update` |
| `sql/rpl_record.cc:304` / `:508` | binlog 行内 partial 布局 / 从库 apply 准备 |
| `sql/field.cc:7858` / `:7890` / `:7983` | `pack_diff` / `get_diff_vector_and_length` / `unpack_diff` |
| `libbinlogevents/include/binlog_event.h:354` | `PARTIAL_UPDATE_ROWS_EVENT = 39` |
| `unittest/gunit/json_binary-t.cc` / `json_dom-t.cc` / `json_path-t.cc` | 三个单元测试，格式细节的最佳参考 |

**函数实现（第六章）**

| 位置 | 说明 |
|------|------|
| `sql/item_json_func.cc:1823` | `JSON_EXTRACT::val_json`（`->`/`->>` 在 yacc 层区分） |
| `sql/item_json_func.cc:785` | `contains_wr`（JSON_CONTAINS 递归判定） |
| `sql/item_json_func.cc:994` | `JSON_CONTAINS_PATH`（`only_need_one`，不取值比较，故快） |
| `sql/item_json_func.cc:2762` | `find_matches`（JSON_SEARCH 递归 + LIKE） |
| `sql/item_json_func.cc:1716` / `:1741` | `JSON_LENGTH` / `JSON_DEPTH`（后者强制建 DOM） |
| `sql/item_json_func.cc:1762` | `JSON_KEYS`（空 object → `[]`） |
| `sql-common/json_dom.cc:885` / `:102` | `merge_patch`（RFC 7396）/ `merge_doms`（PRESERVE） |
| `sql-common/json_dom.cc:1104` / `:1132` | `escape_character` / `double_quote`（JSON 转义） |
| `sql/item_json_func.cc:527` + `json_syntax_check.cc:62` | `JSON_VALID`（不建 DOM） |
| `sql/item_json_func.cc:1660` | `Item_typecast_json::val_json`（CAST AS JSON，建 DOM） |
| `sql/item_sum.cc:6002` / `:6043` | `JSON_ARRAYAGG::add` / `JSON_OBJECTAGG::add` |
| `sql-common/json_syntax_check.cc:85` | `JSON_DOCUMENT_MAX_DEPTH = 100` |

**崩溃恢复（第十二章）**

| 位置 | 说明 |
|------|------|
| `storage/innobase/srv/srv0start.cc:1988` / `:2036` / `:2534` | redo 扫描 / apply / 回滚线程 |
| `storage/innobase/trx/trx0roll.cc:712` / `:765` / `:851` | 回滚入口三件套 |
| `storage/innobase/trx/trx0rec.cc:904` / `:1883` | `trx_undo_read_blob_update` / 进入门闸 |
| `storage/innobase/lob/lob0purge.cc:65` / `:414` | LOB 回滚 / purge 总入口 |
| `storage/innobase/lob/lob0update.cc:600` | `apply_undolog`（小改动回滚） |
| `storage/innobase/lob/lob0index.h:169` / `:178` | `can_rollback` / `can_be_purged` |
| `storage/innobase/include/trx0rec.h:317` / `:320` | `TRX_UNDO_MODIFY_BLOB` / `TRX_UNDO_UPD_EXTERN` |
| `storage/innobase/log/log0recv.cc:1730` / `:2276` | redo 类型分发 / `MLOG_WRITE_STRING` |
| `storage/innobase/row/row0upd.cc:407` / `:476` | `row_upd_changes_field_size_or_external`（in-place vs delete+insert 的闸门） |
| `storage/innobase/btr/btr0cur.cc:3305` / `:3500` / `:3736` | `btr_cur_update_in_place_log` / in-place 更新 / extern 列放弃乐观更新 |
| `storage/innobase/include/mtr0types.h:264` / `:263` / `:261` | `MLOG_REC_UPDATE_IN_PLACE`(70) / DELETE(69) / INSERT(67) |
| `storage/innobase/mtr/mtr0log.cc:256` / `:327` / `:342` | `mlog_write_ulint` / `mlog_write_string` / `mlog_log_string` |
| `storage/innobase/include/lob0pages.h:97` | LOB 页有效载荷 `payload() = 16327` |
| `storage/innobase/lob/lob0impl.cc:929` / `:1048` | `lob::insert`（全量重写）/ 每 4 页提交一次 mtr |

> LOB 更完整的速查见 [`innodb/lob.md`](../../innodb/lob.md)。

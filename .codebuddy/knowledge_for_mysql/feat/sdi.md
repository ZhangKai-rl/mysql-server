# SDI（Serialized Dictionary Information）

> **边界**：本篇**横跨 server 与 InnoDB 两层**。归位 `feat/` 是因为 SDI 与 `fts.md` / `partitioning.md` 同性质——**8.0 引入的跨层端到端特性**，且特性自身的机制（JSON 格式、SDI B-tree、`ibd2sdi`）需要深剖（类比 `fts.md` 深剖倒排列表与 VLC 编码）。这与 `feat/i_s.md`（通用**框架**，server 提供框架、引擎填充）性质不同。
> - **server 侧**（`sql/dd/impl/sdi.cc`、`sdi_api.cc`、`sdi_file.cc`）：序列化格式、`serialize`/`deserialize`、两个 context、三层版本号、`.sdi` 文件载体
> - **InnoDB 侧**（`dict0sdi.cc`、`ibd2sdi.cc`、`row0import.cc`）：SDI B-tree 存储、根页号、zlib 压缩与 LOB 溢出、离线解析工具
> - **与 DD 的关系**：SDI 是 DD 对象的**冗余序列化副本**，不是独立数据源。DD 的表体系 / 对象模型 / 缓存 / 自举 / 升级 / 建表往返见 [`../server/dd/dd.md`](../server/dd/dd.md)；I_S 的同款跨层机制见 [`i_s.md`](../infra/observability/i_s.md)。本篇只讲 SDI 自身。

**一句话机制**：DD 对象树被序列化成**紧凑 JSON**，写进**表空间内的 SDI B-tree**（或退化为独立 `.sdi` 文件）。⚠️ 服务器自己目前**不读**它——`dict_sdi_get` / `dict_sdi_drop` 整个函数体被 `#if 0` 关掉（见 §10），真实读场景只有 `IMPORT TABLESPACE ... FROM .sdi` 与 `ibd2sdi` 离线工具。

## 目录

- [1. 是什么：紧凑 JSON](#1-是什么紧凑-json)（顶层 5 键 / `dd_object` 骨架 / **枚举写成数值** / 三处偏离对象树）
- [2. 什么时候写、哪些表空间有](#2-什么时候写哪些表空间有)
- [3. `ibd2sdi`：独立实现的页级解析器（77 KB）](#3-ibd2sdi独立实现的页级解析器77-kb)（★ 为什么不能走 `dict_sdi_get` / 命令行 / 七步流程 / 校验三层 / ★输出信封是手写的）
- [4. SDI 与 DD 的关系](#4-sdi-与-dd-的关系)
- [5. `.sdi` 文件：SDI 的备用载体](#5-sdi-文件sdi-的备用载体)（★ 按 SE 能力分派，不是按 `innodb_read_only`）
- [6. serialize / deserialize 与两个"反向 pimpl" context](#6-serialize--deserialize-与两个反向-pimpl-context)
- [7. 引用怎么表示：三种形式](#7-引用怎么表示三种形式)
- [8. 三层版本号与兼容检查](#8-三层版本号与兼容检查)
- [9. 存储：SDI 是一棵"伪装的 InnoDB B-tree"](#9-存储sdi-是一棵伪装的-innodb-b-tree)
- [10. 读写路径：SDI 实际上"只写不读"](#10-读写路径sdi-实际上只写不读)
- [11. 已知 bug 与限制](#11-已知-bug-与限制)
- [12. 反序列化（读）那一半](#12-反序列化读那一半)（★ `resolve_hton` 在**写**侧不在读侧 / ★ `lookup_schema_ref` 其实不查找 / opx 回查 / `buf_handle` 复用缓冲）
- [13. store / drop 家族：按引擎能力分派](#13-store--drop-家族api-层与按引擎能力分派)（★ `drop_after_update` 为什么必须"后删" / 文件名前 16 字符）
- [14. ★ 两条 IMPORT 是两回事](#14--两条-import-是两回事物理搬运-vs-逻辑重建)（`FROM .sdi` 逻辑重建 vs 传统 `.cfg` 物理搬运，后者不解析 JSON）
- [15. 5.7 升级与 SDI](#15-57-升级与-sdi为什么升级期禁写)（★ 升级期 SDI 写失败**不让升级失败**——该阶段只能向前滚动）
- [16. 其他引擎：no-op 是一种"显式不做"](#16-其他引擎怎么处理-sdino-op-是一种显式不做)（★ NDB 与 PFS 的手法）
- [17. SDI 页族 layout 全景（精确偏移）](#17-sdi-页族-layout-全景精确偏移)（★ 三种页 / 索引页与 `FIL_PAGE_INDEX` **字节级相同** / 记录定长偏移 / ★ 两种溢出页布局**完全不同** / ★ page 0 偏移是**算出来的**=10505 / 根页与段 / redo 只有建页是专属的）
  - [17.8 端到端字节追踪](#178--端到端字节追踪从-ibd-文件到一条-json)（★ 八步串起全部偏移：page 0 → 根页 → 叶子 → 记录 → BLOB 引用 → 溢出页 → 解压 → JSON）
- [Misc](#misc) / [参考](#参考)

---

## 1. 是什么：**紧凑 JSON**

**SDI 就是 JSON 文本**——由 RapidJSON 的 `Writer`（**不是** `PrettyWriter`）生成，UTF-8、无缩进无换行。头文件里直接把类型定义为字符串：

```cpp
typedef String_type Sdi_type;
/**
  Serialize a Table object.
  @return sdi (as json string).
*/
```

⚠️ `PrettyWriter` 只是**可选开关**，注释明说会显著变大：

```cpp
/**
  Alias for the rapidjson Writer type to use in serialization.
  Can be changeed to RJ_PrettyWriter to get human-readable (but
  significatly larger) sdis.
*/
using Sdi_writer = RJ_Writer;
```

⇒ **`ibd2sdi` 默认输出的多行漂亮 JSON 是它自己再过一遍 PrettyWriter 的结果，不是 SDI 原样**（`--pretty` **默认开**；`--skip-pretty` 才是原样 `fwrite`）。写进表空间的始终是紧凑单行（输出信封的完整细节见 3.5）。

**顶层恰好 5 个键，顺序固定**（生成代码就是这段）：

```cpp
  w.StartObject();
  w.String(STRING_WITH_LEN("mysqld_version_id"));   w.Uint64(MYSQL_VERSION_ID);
  w.String(STRING_WITH_LEN("dd_version"));          w.Uint(Dictionary_impl::get_target_dd_version());
  w.String(STRING_WITH_LEN("sdi_version"));         w.Uint64(SDI_VERSION);
  w.String(STRING_WITH_LEN("dd_object_type"));      w.String(dd_object_type, ...);
  w.String(STRING_WITH_LEN("dd_object"));           dd_obj.serialize(&wctx, &w);
  w.EndObject();
```

`dd_object_type` 只有两种字面量：`"Table"` 或 `"Tablespace"`。

#### `dd_object` 的层次

序列化**没有独立的 schema 描述层**（没有 XSD / protobuf / IDL）——就是**每个 DD 对象 impl 的成员函数** `void X_impl::serialize(Sdi_wcontext*, Sdi_writer*) const` 递归拼出来的。

- **继承**展开到**同一个 JSON object**（`Table_impl::serialize` 先调 `Abstract_table_impl::serialize` 再写自己的，所以 `name`/`columns`/`schema_ref` 与 `engine`/`indexes` 是**平级**键，不是 `"base": {...}`）
- **组合**（表→列/索引/分区）是 JSON **array**
- **子分区是递归嵌套**（`Partition_impl` 里 `serialize_each(wctx, w, m_subpartitions, ...)`）

一个表级 SDI 的骨架（由各 `serialize()` 直接推出）：

```
{
  "mysqld_version_id": 80039, "dd_version": 80023, "sdi_version": 80019,
  "dd_object_type": "Table",
  "dd_object": {
    "name": "t1", "mysql_version_id": 80039, "created": ..., "last_altered": ...,
    "hidden": 1, "options": "avg_row_length=0;...",
    "columns": [ { "name":"id", "type":4, "ordinal_position":1,
                   "default_value":"<base64>", "collation_id":255,
                   "elements":[], "se_private_data":"...", ... } ],
    "schema_ref": "db1",
    "se_private_id": 1234, "engine": "InnoDB", "comment": "",
    "se_private_data": "...", "row_format": 2, "partition_type": 0,
    "indexes": [ { "name":"PRIMARY", "hidden":false, "type":1, "algorithm":2,
                   "elements":[{"ordinal_position":1, "length":4294967295,
                                "order":0, "column_opx":0}], ... } ],
    "foreign_keys": [...], "check_constraints": [...],
    "partitions": [ { "name":"p0", "values":[...], "indexes":[...],
                      "subpartitions":[...], "tablespace_ref":"db1/t1" } ],
    "collation_id": 255, "tablespace_ref": "db1/t1"
  }
}
```

#### ★ 枚举一律写成**数值**，不是字符串

```cpp
template <typename W, typename ENUM_T>
void write_enum(W *w, ENUM_T enum_val, const char *key, size_t keysz) {
  write(w, static_cast<ulonglong>(enum_val), key, keysz);
}
```

C++ 侧本来就是 `enum class`，`write_enum` 只是 `static_cast<ulonglong>` 直转——**没有任何"枚举→名字"映射表**，所以数值是唯一无成本的写法。（`dd::enum_column_types` 里 `LONG = 4`，所以 `"type": 4` 就是 LONG 列。）

⚠️ 代价是 **SDI 不可自解释**：不看源码无法知道 `4` 是什么意思。这与 SDI"离线可读"的定位形成张力，也是 `ibd2sdi` 只能原样输出数值的原因。

#### 三处系统性偏离（SDI ≠ 对象树镜像）

1. **不写 `id`**——`Entity_object_impl::serialize` 只写 `name`。整棵树里没有任何对象 id 落进 SDI ⇒ **IMPORT 时对象会被重新分配 id**
2. **引用被"解引用"成名字或序号**（见 4.7）
3. **二进制/属性被编码成字符串**：`default_value` 走 **base64**；`options` / `se_private_data`（`dd::Properties`）写成**裸的 `key=value;...` 字符串**（`write_properties` 直接 `p.raw_string()`）

（页类型：`FIL_PAGE_SDI = 17853`，另有 `FIL_PAGE_SDI_BLOB = 18` / `FIL_PAGE_SDI_ZBLOB = 19`；根页号存在 page 0，见 4.9。）

## 2. 什么时候写、哪些表空间有

写入链：`Storage_adapter::store()` 写完 DD 表后立即 `sdi::store()`（`storage_adapter.cc:342-347`）→ `dd::sdi::store()` → `hton->sdi_set()` → InnoDB `dict_sdi_set()`（`dict0sdi.cc:307`，复用当前 DDL 事务）。

★ **不是所有表空间都有**（`dict0sdi.cc:321-369` 的跳过条件）：

- discarded 表空间、尚未分配 `se_private_id` 的表、无 `DD_SPACE_ID` 的表空间 → 跳过；
- ★ **`fsp_is_undo_tablespace() || fsp_is_system_temporary()` → 直接 return false**（undo 与临时表空间**不写 SDI**）。

**★ `dict_sdi_set()` 的完整实现**（`storage/innobase/dict/dict0sdi.cc`）——注意**五道跳过条件全部 `return false`（伪装成功，不报错）**：

```cpp
bool dict_sdi_set(handlerton *hton, const dd::Tablespace &tablespace,
                  const dd::Table *table, const sdi_key_t *sdi_key,
                  const void *sdi, uint64_t sdi_len) {
  const char *operation = "set";

  DBUG_EXECUTE_IF("skip_sdi", return (false););

  if (dd_tablespace_is_discarded(&tablespace)) {
    /* Claim success. */
    return (false);                                     // ① discarded 表空间
  }

  /* Check if dd:Table has valid se_private_id. In case of partitions,
     all partitions should have valid se_private_id. If not, we cannot
     proceed with storing SDI as the tablespace is not created yet. */
  if (table && (table->se_private_id() == dd::INVALID_OBJECT_ID) &&
      std::all_of(table->leaf_partitions().begin(),
                  table->leaf_partitions().end(), [](const dd::Partition *lp) {
                    return (lp->se_private_id() == dd::INVALID_OBJECT_ID);
                  })) {
    /* This is a preliminary store of the object - before SE has
       added SE-specific data. Cannot, and should not, store sdi at
       this point. We should not throw error here. There will be SDI
       store again with valid se_private_id/data */
    return (false);                                     // ② SE 还没回填私有数据（"预写"阶段）
  }

  if (!tablespace.se_private_data().exists(dd_space_key_strings[DD_SPACE_ID])) {
    /* Claim success, there will be one more sdi_set()
       after the tablespace is created. */
    return (false);                                     // ③ 表空间还没建
  }

  uint32_t space_id;
  dberr_t err = dict_sdi_exists(tablespace, &space_id);
  if (err != DB_SUCCESS) {
    if (err == DB_TABLESPACE_NOT_FOUND) {
      return (false);                                   // ④ 表空间不存在
    } else {
      dict_sdi_report_error(operation, table, tablespace);
      ut_d(ut_error);
      ut_o(return (true));
    }
  }

  if (fsp_is_undo_tablespace(space_id) || fsp_is_system_temporary(space_id)) {
    return (false);                                     // ⑤ undo / 临时表空间
  }

  trx_t *trx = check_trx_exists(current_thd);
  trx_start_if_not_started(trx, true, UT_LOCATION_HERE);
  innobase_register_trx(hton, current_thd, trx);        // ★ 复用当前 DDL 事务

  ib_sdi_key_t ib_sdi_key;
  ib_sdi_key.sdi_key = sdi_key;

  Sdi_Compressor compressor(static_cast<uint32_t>(sdi_len), sdi);
  compressor.compress();

  err = ib_sdi_set(space_id, &ib_sdi_key, static_cast<uint32_t>(sdi_len),
                   compressor.get_comp_len(), compressor.get_data(), trx);

  if (err == DB_INTERRUPTED) {
    my_error(ER_QUERY_INTERRUPTED, MYF(0));
    return (true);
  } else if (err != DB_SUCCESS) {
    dict_sdi_report_error(operation, table, tablespace);
    ut_d(ut_error);
    ut_o(return (true));
  } else {
    return (false);
  }
}
```

★ 第 ⑤ 条在 `dict_sdi_exists()` 里**已经判过一次**，这里又判一次（防御性冗余）：

```cpp
static dberr_t dict_sdi_exists(const dd::Tablespace &dd_space, uint32_t *space_id) {
  if (dd_space.se_private_data().get(dd_space_key_strings[DD_SPACE_ID], space_id)) {
    ut_d(ut_error);
    ut_o(return (DB_ERROR));
  }
  ut_ad(check_trx_exists(current_thd) != nullptr);

  if (fsp_is_undo_tablespace(*space_id) || fsp_is_system_temporary(*space_id)) {
    /* Claim Success */
    return (DB_SUCCESS);
  }
  return (fsp_has_sdi(*space_id));       // ★ 最终判据是 FSP_SPACE_FLAGS 的 SDI 标志位
}
```

所以：系统表空间 ibdata1、mysql.ibd、每个 file-per-table 的 .ibd、每个通用表空间都有 SDI 索引。

SQL 层的闸门在 `Storage_adapter::store()`——**"先写 DD 表，后写 SDI"**，且有两个额外条件：

```cpp
  if (ctx.otx.open_tables() || object->impl()->store(&ctx.otx)) {
    return true;                                    // ① 先写 mysql.* DD 表
  }
  ...
  // Do not create SDIs for tablespaces and tables while creating
  // dictionary entry during upgrade.
  if (bootstrap::DD_bootstrap_ctx::instance().get_stage() >
          bootstrap::Stage::CREATED_TABLES &&
      dd::upgrade_57::allow_sdi_creation() && sdi::store(thd, object))
    return true;                                    // ② 后写 SDI
```

> ★ 对应的 **drop 侧顺序相反**：`storage_adapter.cc` 里是 `if (sdi::drop(thd, object)) return true;` 出现在删 DD 行**之前**——即"先删 SDI"。因为 SDI 是冗余副本，先删它不影响 DD 的权威数据。

**按 SE 能力分派**（注意 store 用 `sdi_set`、drop 用 `sdi_delete`，**两个不同的函数指针**）：

```cpp
bool store(THD *thd, const Table *tp) {
  const Table &t = ptr_as_cref(tp);
  handlerton *hton = resolve_hton(thd, t);

  return with_schema(thd, t.schema_id(), [&](const Schema &s) {
    dd::Sdi_type sdi = serialize(thd, t, s.name());
    if (sdi.empty()) return checked_return(true);
    return checked_return(
        hton->sdi_set ? sdi_tablespace::store_tbl_sdi(thd, hton, sdi, t, s)
                      : sdi_file::store_tbl_sdi(sdi, t, s));   // ★ 三元分派
  });
}

bool drop(THD *thd, const Table *tp) {
  const Table &t = ptr_as_cref(tp);
  const handlerton &hton = ptr_as_cref(resolve_hton(thd, t));
  return with_schema(thd, t.schema_id(), [&](const Schema &s) {
    return checked_return(hton.sdi_delete
                              ? sdi_tablespace::drop_tbl_sdi(thd, hton, t, s)
                              : sdi_file::drop_tbl_sdi(t, s));  // ★ sdi_delete
  });
}
```

而 Tablespace 的分派是**早退**——它没有 `.sdi` 文件这个退路：

```cpp
bool store(THD *thd, const Tablespace *ts) {
  handlerton *hton = resolve_hton(thd, *ts);
  if (hton->sdi_set == nullptr) {
    return false;  // SDI api not supported
  }
  ...
}
```

> ⚠️ **判据是函数指针，不是能力标志**：`sql/handler.h` 里 `HTON_SUPPORTS_` 共 16 处命中，**没有任何一个是 SDI 相关的**。

**`.sdi` 文件为什么没有 fsync**——三条硬证据（`sdi_file.cc`）：

```cpp
bool write_sdi_file(const dd::String_type &fname, const dd::Sdi_type &sdi) {
  File sdif = mysql_file_create(key_file_sdi, fname.c_str(), CREATE_MODE,
                                O_WRONLY | O_TRUNC, MYF(MY_FAE));   // ① 无 O_SYNC
  ...
  size_t bw = mysql_file_write(sdif, (const uchar *)sdi.c_str(),
                               sdi.length(), MYF(MY_FNABP));        // ② 无同步语义
  ...
  return checked_return(mysql_file_close(sdif, MYF(MY_FAE)));       // ③ 无 my_sync()
}
```

grep `fsync|MY_SYNC|my_sync|O_SYNC|MY_FULL_IO` 于 `sql/dd` → **0 命中**。设计意图就是文件头注释那句：

> *"Storage and retrieval of SDIs to/from files. Default for SEs which do not have the ability to store SDIs in tablespaces. **File storage is not transactional.**"*

文件名的构造（**16 是"字符"不是"字节"**，多字节表名按 `my_mbcharlen()` 推进）：

```cpp
String_type sdi_filename(Object_id id, const String_type &entity_name,
                         const String_type &schema) {
  const CHARIT begin = entity_name.begin();
  CHARIT i = begin;
  size_t count = 0;
  while (i != end && count < dd::sdi_file::FILENAME_PREFIX_CHARS) {
    size_t charlen = my_mbcharlen(system_charset_info, static_cast<uchar>(*i));
    i += charlen;
    ++count;
  }
  Stringstream_type fnamestr;
  fnamestr << String_type(begin, i) << "_" << id;      // <表名前16字符>_<dd_object_id>
  ...
  build_table_filename(path, sizeof(path) - 1, schema.c_str(),
                       fnamestr.str().c_str(), EXT.c_str(), 0, &was_truncated);
  return String_type(path);
}
```

## 3. `ibd2sdi`：独立实现的页级解析器（77 KB）

> 这是 SDI 唯一一条**不依赖活着的服务器**的读路径。`utilities/ibd2sdi.cc` 有 **77 KB**，比 server 侧 `sdi.cc`（22 KB）+ InnoDB 侧 `dict0sdi.cc` 加起来还大——因为它在服务器不启动的前提下，**自己重新实现了页解析、B-tree 下潜、LOB 拼接与 zlib 解压**。

### 3.1 ★ 为什么它不能走 `dict_sdi_get`

`dict_sdi_get` 需要一个活的 InnoDB 实例（`dict_sys`、事务、btr cursor），而 `ibd2sdi` **根本没有**：

```
MYSQL_ADD_EXECUTABLE(ibd2sdi ibd2sdi.cc COMPONENT Server DEPENDENCIES GenError
  LINK_LIBRARIES mysys innodb_zipdecompress ext::lz4 extra::rapidjson)
TARGET_COMPILE_DEFINITIONS(ibd2sdi PRIVATE UNIV_NO_ERR_MSGS UNIV_LIBRARY DISABLE_PSI_MEMORY)
```

只链 `mysys` + `innodb_zipdecompress`（压缩页解压）+ lz4 + rapidjson——**没有 innobase、没有 sql/DD**。

它甚至**伪造了 InnoDB 的全局符号**，只为复用"页格式头文件/工具函数"：

```c
/** Replaces declaration in srv0srv.c */
ulong srv_page_size;
ulong srv_page_size_shift;
page_size_t univ_page_size(0, 0, false);
```

并补了自己的桩：`ut_dbg_assertion_failed()`（打印后 `abort()`）、`Fil_page_header::print()`（注释 *"A dummy implementation. Actual implementation available in fil0fil.cc"*）、以及完整重写的 `namespace ib { info/warn/error/fatal }`。

`main()` 只有 `ut_crc32_init(); MY_INIT(argv[0]);` ——**没有 `srv_start`、没有 data dictionary、没有 trx/buf pool**。它 include 的是 `page0page.h / fsp0fsp.h / mach0data.h / lob0lob.h / dict0sdi-decompress.h` 这类"纯页结构/纯计算"头文件。

### 3.2 命令行接口

```c
static struct my_option ibd2sdi_options[] = {
    {"dump-file", 'd', "Dump the tablespace SDI into the file passed by user."
                       " Without the filename, it will default to stdout", ...},
    {"skip-data", 's', "Skip retrieving data from SDI records. Retrieve only id and type.", ...},
    {"id", 'i', "Retrieve the SDI record matching the id passed by user.", ...},
    {"type", 't', "Retrieve the SDI records matching the type passed by user.", ...},
    {"strict-check", 'c', "Specify the strict checksum algorithm by the user."
                          " Allowed values are innodb, crc32, none.", ...},
    {"no-check", 'n', "Ignore the checksum verification.", ...},
    {"pretty", 'p', "Pretty format the SDI output. If false, SDI would be not human"
                    " readable but it will be of less size", ...OPT_ARG, 1 /*默认开*/},
};
```

（`--strict-check` 三值对应 `SRV_CHECKSUM_ALGORITHM_STRICT_CRC32` / `STRICT_INNODB` / `STRICT_NONE`；man 页给出默认：`id`=0、`type`=0、`strict-check`=**crc32**、`pretty`=**TRUE**，用 `--skip-pretty` 关。）

**选项组合的语义**（`main()` 里）：

- `--no-check` 与 `--strict-check` **互斥**，同时给直接报错退出
- **`--id` + `--type` 同时给**才走"精确匹配一条"（`if (opts.is_sdi_id && opts.is_sdi_type) opts.is_sdi_rec = true;`）；只给 `--id` → `dump_matching_ids()`；只给 `--type` → `dump_matching_types()`；都不给 → 全量
- `--dump-file` 的实现是**先写临时文件再改名**（`create_tmp_file` → `my_rename`），跨分区失败则 `my_copy`，注释：*"Rename file can fail if the source and destination are across partitions."*

### 3.3 完整解析流程：七步

**① 建"表空间"对象**——先只读 1024 字节拿头部，取 `space_id` 与 `first_page_num`。

**② ★ 定页大小**（page 0 损坏时的兜底最有意思）：先走 `fsp_header_get_flags()`；若 flags 非法则打印 *"Page 0 corruption detected"* 并转 `determine_page_size()`——**穷举** logical(4k…64k) × physical 组合，每种读最多 `MAX_PAGES_TO_SCAN = 60` 页算"损坏率"，取最小的那个（全 0 页不计入）。

多文件（ibdata\*）还有连续性约束：后续文件 `space_id` 必须相同、`first_page_num` 必须 = 上一文件最后一页 +1。

**③ 拿 SDI 根页号**——`fsp_header_get_sdi_offset()` 算出偏移后：

```c
  ulint sdi_offset = fsp_header_get_sdi_offset(m_page_size);
  uint32_t version = mach_read_from_4(buf + sdi_offset);
  if (version != SDI_VERSION) ib::warn() << "Unexpected SDI version. Expected: ... Got: ...";
  return mach_read_from_4(buf + sdi_offset + 4);
```

即 **SDI 偏移处先 4 字节版本号、再 4 字节根页号**（与 9. 的 `fsp_sdi_write_root_to_page()` 对称）。⚠️ 版本不匹配**只 warn 不失败**。

**④ 下潜到最左叶子**——每读一页先校验页类型：

```c
  ulint page_type = fil_page_get_type(buf);
  if (page_type != FIL_PAGE_SDI) { ib::error() << "Unexpected page type: ... Expected page type:" << FIL_PAGE_SDI;
                                   return UINT64_MAX; }
```

从 INFIMUM 出发，`child_page_num` 的读法是**跳过 type+id 两个定长列**后取 4 字节；要求 `>= SDI_BLOB_ALLOWED (4)`，且 level 必须**恰好减 1**，否则失败。记录数为 0 → 打印 *"SDI is empty"* 并成功返回。

**⑤ 遍历叶子层**——`memcmp(buf + PAGE_NEW_INFIMUM, "infimum", 7)` 校验伪记录；delete-marked 递归跳过；遇 supremum 后按 `FIL_PAGE_NEXT` 翻页，`FIL_NULL` 结束。

⚠️ **防死循环**：`inc_num_of_recs_on_page()` 超过 `m_max_recs_per_page` 时报 *"Record Corruption detected. Too many records or infinite loop detected"*。

**⑥ 解析记录字段**——文件头常量给出了精确的定长列布局：

```c
/** Length of ID field in record of SDI Index. */         REC_DATA_ID_LEN = 8;
/** Length of TYPE field in record of SDI Index. */       REC_DATA_TYPE_LEN = 4;
/** Length of UNCOMPRESSED_LEN field ... */               REC_DATA_UNCOMP_LEN = 4;
/** Length of COMPRESSED_LEN field ... */                 REC_DATA_COMP_LEN = 4;
/** SDI BLOB not expected before the following page number.
0 (tablespace header), 1 (tabespace bitmap), 2 (ibuf bitmap), 3 (SDI Index root page) */
                                                          SDI_BLOB_ALLOWED = 4;
```

`--skip-data` 就在读完 type/id 后**立即 return**，不读 blob、不解压。

变长长度带两个标志位：`0x80` = 两字节长度；`0x40` = **外部存储且带 768 字节内联前缀**（`is_rec_data_external = true`，总长从 `lob::BTR_EXTERN_LEN` 读 8 字节）。这里有个 TODO：*"This 768 check should be removed when this tool supports only REC_FORMAT_DYNAMIC"*。

完整性校验：`if (rec_data_length != sdi_comp_len) { ib::error() << "Record corruption"; return DB_CORRUPTION; }`

**⑦ 解压 + ★ 拼 LOB（顺序很关键：先拼块、后 zlib）**

- 非压缩溢出页 → `copy_uncompressed_blob()`：逐页校验必须是 `FIL_PAGE_SDI_BLOB`，按 `lob::LOB_HDR_PART_LEN` 拷数据，下一页从 `lob::LOB_HDR_NEXT_PAGE_NO` 取，直到 `FIL_NULL`
- 压缩溢出页 → `copy_compressed_blob()`：每页必须是 `FIL_PAGE_SDI_ZBLOB`，用 **zlib 流式 inflate**（`inflateInit` → 循环 `inflate(Z_NO_FLUSH)` → 末页 `Z_FINISH` → `inflateEnd`），分配器 `mem_heap_create(40000)` + `page_zip_set_alloc`（注释：*"Zlib inflate needs 32KB for the default window size, plus a few KB for small objects"*）
- **拼完之后才对 SDI 本体解压**（用的是 InnoDB 的 `dict0sdi-decompress.h`）：

```c
  byte *uncompressed_sdi = static_cast<byte *>(calloc(sdi_uncomp_len + 1, 1));
  Sdi_Decompressor decompressor(uncompressed_sdi, sdi_uncomp_len + 1, str, sdi_comp_len);
  decompressor.decompress();
```

`Sdi_Decompressor::decompress()` 内部就是 `uncompress()`，**失败直接 `ib::fatal`**（abort 进程，不可恢复）。

### 3.4 校验：每一页都做，且有三层

```c
static bool is_page_corrupted(const byte *buf, const page_size_t &page_size) {
  BlockReporter reporter(false, buf, page_size, opts.strict_check);
  return reporter.is_corrupted();
}
```

复用 InnoDB 的 `BlockReporter`；`fetch_page()` 里**每页（含 BLOB 页）都校验**，`--no-check` 才跳过。非严格模式下同时接受 innodb/crc32/none 三种。

三层校验：① 校验和（每页）② 页类型必须是 `FIL_PAGE_SDI` / `FIL_PAGE_SDI_BLOB` / `FIL_PAGE_SDI_ZBLOB` ③ 结构完整性（记录数上限、`child_page_no`/`next_blob_page_no` 不得 `<= SDI_BLOB_ALLOWED`、`rec_data_length == sdi_comp_len`、infimum/supremum payload 文本）。

### 3.5 ★ 输出形态：手工信封 + 条件性 PrettyWriter

容易看错的一点——**外层是 `fprintf` 手写的 JSON 数组，第一个元素是字符串字面量**：

```c
fprintf(out_stream, "%s", "[\"ibd2sdi\"\n");
...
fprintf(out_stream, "{\n");
fprintf(out_stream, "\t\"type\": " UINT64PF ",\n", sdi_type);
fprintf(out_stream, "\t\"id\": " UINT64PF, sdi_id);
if (!m_skip_data) {
  fprintf(out_stream, "\t\"object\":\n\t\t");
  if (opts.pretty) {  rapidjson::Document d; ... PrettyWriter writer(_b);
                      d.Accept(writer); fprintf(out_stream, "%s", _b.GetString()); }
  else { if (fwrite(sdi_data, 1, sdi_data_len - 1, out_stream) != sdi_data_len - 1) ...; }
}
fprintf(out_stream, "\n}\n");
```

要点：

- **`--pretty` 默认开** ⇒ 平时看到的"多行 JSON"是**重新格式化过的**；`--skip-pretty` 时是**原样 `fwrite`** 解压出的原始字节（长度 `-1` 是因为 `calloc(sdi_uncomp_len + 1)` 多留了结尾 `\0`）
- 所以 `ibd2sdi x.ibd | jq '.[1]["object"]'` 的下标**从 1 起**（第 0 个元素是 `"ibd2sdi"`）
- pretty 模式下若 JSON 解析失败**直接 `exit(1)`**，不是返回错误码
- `--skip-data` 时不输出 `object` 键（man 页示例：`["ibd2sdi",{"type":1,"id":330},{"type":2,"id":7}]`）

**`--pretty` 默认开的实证**（`my_option` 的第 9 个字段 `def_value = 1`，且用 `OPT_ARG` 所以能显式写 `--pretty=0`）：

```cpp
    {"pretty", 'p',
     "Pretty format the SDI output."
     "If false, SDI would be not human readable but it will be of less size",
     &opts.pretty, &opts.pretty, nullptr, GET_BOOL, OPT_ARG, 1, 0, 0, nullptr,
     0, nullptr},
```

**从根页下降到叶子层**——它直接用普通索引页的宏（`PAGE_LEVEL` / `PAGE_N_RECS` / `PAGE_NEW_INFIMUM`），这本身就是"SDI 页 ≡ 索引页"的证据：

```cpp
uint64_t ibd2sdi::read_page_and_return_level(ib_tablespace *ts, uint32_t buf_len,
                                             byte *buf, page_no_t page_num) {
  if (fetch_page(ts, page_num, buf_len, buf) == IB_ERROR) { ... }

  ulint page_type = fil_page_get_type(buf);
  if (page_type != FIL_PAGE_SDI) {
    ib::error() << "Unexpected page type: " << page_type
                << ". Expected page type:" << FIL_PAGE_SDI;
    return UINT64_MAX;
  }
  ulint page_level = mach_read_from_2(buf + FIL_PAGE_DATA + PAGE_LEVEL);
  return page_level;
}

err_t ibd2sdi::reach_to_leftmost_leaf_level(ib_tablespace *ts, uint32_t buf_len,
                                            byte *buf, page_no_t root_page_num) {
  uint64_t page_level = read_page_and_return_level(ts, buf_len, buf, root_page_num);
  if (page_level == UINT64_MAX) return FALIURE;

  ulint num_of_recs = mach_read_from_2(buf + FIL_PAGE_DATA + PAGE_N_RECS);
  if (num_of_recs == 0) { ib::dbug() << "Number of records is zero. Nothing to process";
                          return NO_RECORDS; }
  if (page_level == 0) return SUCCESS;

  while (page_level != 0 && page_level != UINT64_MAX) {
    rec_type_byte = *(buf + PAGE_NEW_INFIMUM - REC_OFF_TYPE);
    rec_type = rec_type_byte & 0x7;                       /* ★ 取低 3 位判记录类型 */
    if (rec_type != REC_STATUS_INFIMUM) { ... break; }

    ulint next_rec_off_t = mach_read_from_2(buf + PAGE_NEW_INFIMUM - REC_OFF_NEXT);
    page_no_t child_page_num =
        mach_read_from_4(buf + PAGE_NEW_INFIMUM + next_rec_off_t +
                         REC_DATA_TYPE_LEN + REC_DATA_ID_LEN);
    /* ★ node pointer 的 child page no 就在 type(4)+id(8) 之后 —— 与普通 FIL_PAGE_INDEX 一致 */
    ...
  }
  if (page_level != 0) return FALIURE;
  return SUCCESS;
}
```

**解析一条记录**（含 off-page 判定与解压）——`is_rec_data_external` 由**变长长度字节的最高位**决定：

```cpp
dberr_t ibd2sdi::parse_fields_in_rec(ib_tablespace *ts, byte *rec,
                                     uint64_t *sdi_type, uint64_t *sdi_id,
                                     byte **sdi_data, uint64_t *sdi_data_len) {
  *sdi_type = mach_read_from_4(rec + REC_OFF_DATA_TYPE);          // +0
  *sdi_id   = mach_read_from_8(rec + REC_OFF_DATA_ID);            // +4
  uint32_t sdi_uncomp_len = mach_read_from_4(rec + REC_OFF_DATA_UNCOMP_LEN);  // +25
  uint32_t sdi_comp_len   = mach_read_from_4(rec + REC_OFF_DATA_COMP_LEN);    // +29

  byte rec_data_len_partial = *(rec - REC_MIN_HEADER_SIZE - 1);

  if (rec_data_len_partial & 0x80) {
    /* Rec length is store in two bytes. Read next byte and calculate the total length. */
    rec_data_in_page_len = (rec_data_len_partial & 0x3f) << 8;
    if (rec_data_len_partial & 0x40) {
      is_rec_data_external = true;                       /* ★ bit 6 = off-page 标志 */
      rec_data_length =
          mach_read_from_8(rec + REC_OFF_DATA_VARCHAR + rec_data_in_page_len +
                           lob::BTR_EXTERN_LEN);
      rec_data_length += rec_data_in_page_len;
    } else {
      rec_data_length = *(rec - REC_MIN_HEADER_SIZE - 2);
      rec_data_length += rec_data_in_page_len;
    }
  } else {
    rec_data_length = rec_data_len_partial;              /* 长度 <= 127，单字节 */
  }
  ...
  if (is_rec_data_external) {
    page_no_t first_blob_page_num =
        mach_read_from_4(rec + REC_OFF_DATA_VARCHAR + rec_data_in_page_len +
                         lob::BTR_EXTERN_PAGE_NO);
    if (page_size.is_compressed())
      blob_len_retrieved = copy_compressed_blob(ts, first_blob_page_num, ...);
    else
      blob_len_retrieved = copy_uncompressed_blob(ts, first_blob_page_num, ...);
  } else {
    memcpy(str, rec_data_origin, rec_data_length);
  }

  ut_ad(rec_data_length == sdi_comp_len);                /* ★ 完整性校验 */
  if (rec_data_length != sdi_comp_len) { free(str); return DB_CORRUPTION; }

  byte *uncompressed_sdi = static_cast<byte *>(calloc(sdi_uncomp_len + 1, 1));
  Sdi_Decompressor decompressor(uncompressed_sdi, sdi_uncomp_len + 1, str, sdi_comp_len);
  decompressor.decompress();                             /* ★ zlib inflate */
  ...
}
```
- 精确匹配时有提前终止优化：`else if (sdi_type > m_specific_type && sdi_id > m_specific_id) explicit_sdi_rec_found = true;`

### 3.6 已知限制（有硬证据的才写）

来自 man 页：

1. **不支持临时表空间与 undo 表空间**（*"not supported for use with temporary tablespaces or undo tablespaces"*）
2. ★ **是"未提交读"**：*"ibd2sdi performs an uncommitted read of SDI from the specified tablespace. Redo logs and undo logs are not accessed."* —— 且 DDL/ROLLBACK/purge 期间有短暂窗口会读失败
3. **一次只能一个表空间**（多文件表空间可以，但必须是同一个 space，且文件按页号升序给出）
4. **默认值可能是补零填充的假象**：*"a default value may be present and non-empty in ibd2sdi output ... even if it is not defined using DEFAULT"*

⚠️ **两点不要写错**：

- **压缩表空间支持**（代码明确处理了 `FIL_PAGE_SDI_ZBLOB` 的 inflate 与压缩页的 `page_zip_decompress_low`）——不是限制
- **加密表空间**：`ibd2sdi.cc` 全文搜 `encrypt|keyring` **0 命中**，源码**无直接证据**，不能断言"不支持"

## 4. SDI 与 DD 的关系

**SDI 是 DD 对象序列化后的冗余副本，不是独立数据源**。权威永远是 mysql.ibd 里的 DD 表。用途是**离线/自描述**：`ibd2sdi` 导出、`.ibd` 单独迁移 / IMPORT、崩溃场景的元数据恢复。二者一致性由"DD 写 + SDI 写在同一个事务里"保证。

---

## 5. `.sdi` 文件：SDI 的备用载体

上面四节讲的都是**写进表空间**的 SDI。但 SDI 还有第二条路：**落成独立的 `.sdi` 文件**。文件头注释就写明了定位：

```cpp
/**
  Storage and retrieval of SDIs to/from files. Default for SEs which do
  not have the ability to store SDIs in tablespaces. File storage is
  not transactional.
*/
```

#### ★ 分派条件：按 SE 能力，不是按 `innodb_read_only`

```cpp
    return checked_return(
        hton->sdi_set ? sdi_tablespace::store_tbl_sdi(thd, hton, sdi, t, s)
                      : sdi_file::store_tbl_sdi(sdi, t, s));
```

⚠️ **纠正一个常见说法**：这条路的分派判据是 **`hton->sdi_set == nullptr`（引擎有没有实现 SDI 接口）**，源码中**没有**按 `innodb_read_only` 分派的逻辑。即：

- **能往表空间写 SDI 的引擎**（InnoDB）→ 写表空间
- **不能的引擎**（MyISAM / Archive / CSV / Blackhole 等）→ 写 `.sdi` 文件

对应地，**表空间 SDI 根本没有文件版**——`store(THD*, const Tablespace*)` 里 `if (hton->sdi_set == nullptr) return false;`。

#### 文件名：`<表名前 16 字符>_<id>.sdi`

```cpp
/** Number of character (not bytes) of a tablename which will contrbute to the
    sdi file name. The whole name is not needed as the Object_id is added so
    that uniqueness is ensured */
const size_t FILENAME_PREFIX_CHARS = 16;
```

改名/改 schema 时的清理逻辑也印证了这一点——**前 16 字符没变则文件名不变，不必删旧文件**：

```cpp
  if ((old_t.schema_id() == new_t.schema_id() &&
       equal_prefix_chars_name(old_t, new_t, sdi_file::FILENAME_PREFIX_CHARS))
      || old_t.engine() == "innodb") {
    return false;
  }
```

#### 各符号做什么

| 符号 | 作用 |
|---|---|
| `write_sdi_file` | `mysql_file_create` → `write` → `close`（`MY_FAE` / `MY_FNABP`） |
| `sdi_file_exists` | POSIX 用 `my_access`；⚠️ Windows 上 `my_access` 不能用来判"文件不存在"，改用 `GetFileAttributesEx` |
| `make_dir_pattern_tuple` | 把通配拆成 `dir` + 文件名模式，并算 `m_is_inside_datadir` |
| `expand_sdi_pattern` | 展开成实际路径列表：**受 `secure_file_priv` 限制**、`my_wildcmp` 匹配、**必须以 `.sdi` 结尾**，一个都没匹配则报 `ER_IMP_NO_FILES_MATCHED` |
| `sdi_filename` | 生成文件名（16 字符前缀 + `_` + id） |
| `store_tbl_sdi` / `drop_tbl_sdi` | 写/删（不存在不报错） |
| **`load`** | `fstat` + 一次整读，得到 **SDI 的 JSON 文本** |
| `check_data_files_exist` | 检查 `.MYD` / `.MYI` 是否都在（IMPORT 用） |

`load()` 只负责读字节，反序列化在 `sdi_api.cc`：

```cpp
bool Import_target::load(THD *thd, String_type *shared_buffer) {
  if (dd::sdi_file::load(thd, m_path, shared_buffer)) return true;
  if (dd::deserialize(thd, *shared_buffer, m_table_object.get(), &m_schema_name_in_sdi))
    return true;
```

即：**`.sdi` 文件里就是 SDI 的 JSON 文本**，`load` → `deserialize` → 检查数据文件 → `dc.store()`，这就是 `IMPORT TABLESPACE` 能从 `.ibd`/`.MYD` + `.sdi` 恢复表定义的机制。

#### ⚠️ 为什么没有 fsync

**源码未给出解释。** 写路径全程只有 `mysql_file_create(MY_FAE)` + `mysql_file_write(MY_FNABP)` + `mysql_file_close(MY_FAE)`，**没有任何 `my_sync` / `fdatasync`**。唯一的说明是文件头那句 *"File storage is not transactional."*。可以如实记为：**文件存储被明确声明为非事务性，但未解释省略 fsync 的理由**。

（对比：写进表空间的 SDI 由"DD 写 + SDI 写在同一事务"保证一致性——见 4.4。文件版没有这个保证。）

## 6. ★ serialize / deserialize 与两个"反向 pimpl" context

**写**：`dd::Table` 对象树 → RapidJSON `Writer` **流式**写进 `RJ_StringBuffer` → 取字符串。**不经 DOM、不自写 JSON**。

```cpp
  dd::Sdi_wcontext wctx(thd, schema_name);
  dd::RJ_StringBuffer buf;
  dd::Sdi_writer w(buf);
  ... // 5 个顶层键；dd_object 由 dd_obj.serialize(&wctx, &w) 递归写
  return (wctx.error() ? empty_ : String_type(buf.GetString(), buf.GetSize()));
```

**读**：JSON → RapidJSON **DOM**（`doc.Parse<0>`）→ 版本检查 → `dst->deserialize(&rctx, dd_object_val)` 递归。

★ **读写不对称**：写用流式 Writer，读必须用 DOM——因为要先读顶层版本键做兼容检查，还要按名字随机访问成员（`gv.HasMember(key)`）。

⚠️ **顶层各键的存在性/类型全是 `assert`，不是错误检查**（`generic_deserialize` 里 6 处 `assert`）——前提是"SDI 由本版本自己写的"。release 下损坏/伪造的 SDI 可能走到未定义行为。

#### ★ `Sdi_wcontext` / `Sdi_rcontext`：为什么是"空壳 + 友元自由函数"

源码专门写了注释解释：

```cpp
/**
  Declarations of internal functions which operate on Sdi_context
  objects. Conceptually these are member functions of Sdi_context,
  but declaring them as such would mean that the definition of
  Sdi_context would have had to be made available in every
  translation unit where these functions get called (most data
  dictionary object implementation files).

  This is essentially a modification of the pimpl idiom where we avoid
  the need to create separate api and implementation classes ...
*/
```

⇒ **反向 pimpl**：类是空壳（前向声明 + `friend`），成员函数全部实现为 `sdi.cc` 里的自由函数。目的是让 40 多个 `sql/dd/impl/types/*_impl.cc` **不必 include RapidJSON 与 THD 的重型定义**（这就是 §8 讲的"编译防火墙"）。

**`Sdi_wcontext`（写）四个职责**：

| 成员 | 干什么 |
|---|---|
| `Byte_buffer m_buf` | **可复用临时缓冲**，主要给 base64 编码（`write_binary`） |
| `THD *m_thd` | 让序列化能**回查 DD**（拿 tablespace 名）并**拿 MDL** |
| `const String_type *m_schema_name` | 写 `schema_ref` 用的名字 |
| `bool m_error` | 错误累积标志；`generic_serialize` 最后据此返回**空串** |

⚠️ **序列化一个表可能触发 MDL 等待/死锁**——`lookup_tablespace_name` 在把 `tablespace_id` 翻成名字时会 `acquire_uncached_uncommitted` + `mdl_lock(MDL_key::TABLESPACE, ..., MDL_INTENTION_EXCLUSIVE)`。

**`Sdi_rcontext`（读）**多了两个关键成员：

```cpp
  /** Column objects created during deserialization */
  dd_vector<Column *> m_column_object_opx;
  /** Index objects created during deserialization */
  dd_vector<Index *> m_index_object_opx;
```

即 **opx → 对象指针的解析表**，用来重建"索引元素→列"、"分区索引→索引"的内存指针引用（见 4.7）。

★ **一个值得记的发现**：`rctx` 里存了 `m_target_dd_version` 与 `m_sdi_version`，但**全代码库没有任何读取处**——只有声明、构造初始化、构造 rctx 三处。即：**"按版本做兼容反序列化"的钩子目前是死代码**，版本兼容实际只在 `CheckDefaultCompatibility` 里做"全等拒绝"。

## 7. 引用怎么表示：三种形式

**SDI 里没有对象图，只有引用**。按被引用对象的性质分三种：

| 引用目标 | SDI 里长什么样 | 读回时怎么解析 |
|---|---|---|
| **Column / Index**（来自 Index_element、Foreign_key、Partition_index） | `"column_opx": N` / `"index_opx": N`（**0-based**，即 `ordinal_position - 1`） | 查 `rctx` 的 opx 表拿**内存指针**，不查 DD |
| **Schema** | `"schema_ref": "db1"`（名字） | **不解析**——id 置 `INVALID_OBJECT_ID`，由 IMPORT 后续修正 |
| **Tablespace** | `"tablespace_ref": "db1/t1"`（名字，**可能整个键缺失**） | 拿 MDL + 查当前 DD 得 id |
| **Collation / SRS** | `"collation_id": 255`（**裸 id**） | 直接当 id 用，**不校验** |

**opx 的注册与解析**：

```cpp
template <typename T>
void generic_track_object(dd_vector<T *> *tvp, T *t) {
  assert(t->ordinal_position() > 0);
  uint opx = t->ordinal_position() - 1;
  ...
  tv[opx] = t;
}
```

★ 这带来一条**硬性顺序约束**，`table_impl.cc` 有专门注释：

```cpp
  // Note! Deserialization of ordinal position cross-referenced
  // objects (i.e. Index and Column) must happen before deserializing
  // objects which reference these objects:
  // Foreign_key_element -> Column, Foreign_key -> Index,
  // Index_element -> Column, Partition_index -> Index
  // Otherwise the cross-references will not be deserialized correctly
```

⇒ 即 **`columns` 先于 `indexes` 先于 `foreign_keys` / `partitions`**——这也是顶层键顺序的由来。

⚠️ **最脆弱的一环是 `collation_id`**：它是 DD 内部自增主键，**跨实例不保证一致**。SDI 存的是 id 而非名字，所以"SDI 可跨实例移植"这一说法**对 collation 不成立**。

## 8. ★ 三层版本号与兼容检查

| 版本 | 含义 | 谁递增 |
|---|---|---|
| `mysqld_version_id` | 服务器二进制版本（`MYSQL_VERSION_ID`） | 随发版自动 |
| `dd_version` | **DD 表结构版本**（当前 80023） | 改 `mysql.*` DD 表定义时 |
| `sdi_version` | **SDI JSON 格式版本**（当前 80019） | 改"DD 对象 → JSON 的转换方式"时（**DD 表结构变更不递增**） |

`sdi_version` 的注释把递增责任说得很清楚：

```cpp
/**
  This number is bumpred only when there is a change in how the data
  dictionary information is converted to json (e.g. adding a forgotten
  member). It does not need to be bumped when the data dictionary schema
  structure is changed, as this is covered by the DD_VERSION variable.
*/
```

（"adding a forgotten member"正是下面两个 bug 的类型。）

#### 版本不匹配：**拒绝，不是警告**

```cpp
bool CheckDefaultCompatibility(const RJ_Document &doc) {
  if (mysqld_version_id.GetUint64() > MYSQL_VERSION_ID) {
    // Cannot deserialize SDIs from newer versions.
    my_error(ER_IMP_INCOMPATIBLE_MYSQLD_VERSION, ...); return true;
  }
  if (dd_version != Dictionary_impl::get_target_dd_version()) {
    my_error(ER_IMP_INCOMPATIBLE_DD_VERSION, ...); return true;
  }
  if (sdi_version_ != SDI_VERSION) {
    my_error(ER_IMP_INCOMPATIBLE_SDI_VERSION, ...); return true;
  }
  return false;
}
```

三条规则**语义不同**：

- `mysqld_version_id`：**`<=` 当前**（单向宽松）——**老版本写的能读，新版本写的不能**
- `dd_version`：必须**全等**
- `sdi_version`：必须**全等**

⇒ **`sdi_version` 必须全等，导致跨 minor 版本导入基本不可行**（8.0.15→80016→80019 各变过一次）。

架构上预留了可插拔 checker（`using SdiCompatibilityChecker = std::function<bool(const RJ_Document&)>;`），但 **代码库中没有任何调用方传入自定义 checker**——全部走默认版。

#### 版本演进（源码注释原文）

`sdi_version`：`1`（8.0.15）→ `80016`（8.0.16，修 **Bug#29210646**：`Index_impl::m_hidden` 漏进 SDI）→ `80019`（当前，修 **Bug#30326020**：子分区没进 SDI）。

**`SDI_VERSION` 的定义与那段版本历史注释的原文**（`sql/dd/impl/sdi.h`）：

```cpp
/**
  The version of the current SDI Json format.

  This version number is stored inside SDIs. This number is bumpred only when
  there is a change in how the data dictionary information is converted to json
  (e.g. adding a forgotten member). It does not need to be bumped when the data
  dictionary schema structure is changed, as this is covered by the DD_VERSION
  variable.
  ...
  Historical version number published in the data dictionary, (note that 1 is a
  legacy number from before SDI version numbers were mapped to server versions):

  1: Published in 8.0.15
  ----------------------------------------------------------------------------
  Initial version.

  80016: Published in 8.0.16
  ----------------------------------------------------------------------------
  Changes from version 1:

  - Bug#29210646: DD::INDEX_IMPL::M_HIDDEN NOT INCLUDED IN SDI

  80019: Current
  ----------------------------------------------------------------------------
  Changes from version 80016:

  - Bug#30326020: SUBPARTITIONING NOT REFLECTED IN SDI

  800XX: Next SDI version number after the previous is public. ...
  ----------------------------------------------------------------------------
  Changes from current version:

  - No changes, this version number is not active yet.

  If a new SDI version is published in a MRU, it will not
  be possible to import this version into previous MRUs within the same GA.
*/
constexpr const std::uint64_t SDI_VERSION = 80019;
```

> ★ 最后那段是**前瞻性风险说明**：在 MRU（小版本）里 bump `sdi_version`，会导致同一 GA 内的旧 MRU 无法导入新版 SDI。这是"版本号三个层级"设计里最容易被忽视的约束。

**⚠️ 两个同名不同义的 `SDI_VERSION`（极易混淆）**：

```cpp
/* storage/innobase/include/dict0dict.h —— InnoDB 页头版，恒为 1 */
/** SDI version. Written on Page 1 & 2 at FIL_PAGE_FILE_FLUSH_LSN offset. */
const uint32_t SDI_VERSION = 1;
```

| | 值 | 写在哪 |
|---|---|---|
| `dd::SDI_VERSION` | **80019** | SDI **JSON** 里的 `sdi_version` 键 |
| `dict0dict.h::SDI_VERSION` | **1** | page 0 的 SDI 元数据区（`fsp_sdi_write_root_to_page`） |

**兼容检查 `CheckDefaultCompatibility()`——三条规则不对称**：

```cpp
bool CheckDefaultCompatibility(const RJ_Document &doc) {
  const RJ_Value &mysqld_version_id = doc["mysqld_version_id"];
  if (mysqld_version_id.GetUint64() > std::uint64_t(MYSQL_VERSION_ID)) {
    // Cannot deserialize SDIs from newer versions.
    my_error(ER_IMP_INCOMPATIBLE_MYSQLD_VERSION, MYF(0),
             mysqld_version_id.GetUint64(), std::uint64_t(MYSQL_VERSION_ID));
    return true;
  }                                                   // ★ ① mysqld_version：<=

  const RJ_Value &dd_version_val = doc["dd_version"];
  uint dd_version = dd_version_val.GetUint();
  if (dd_version != Dictionary_impl::get_target_dd_version()) {
    my_error(ER_IMP_INCOMPATIBLE_DD_VERSION, MYF(0), dd_version,
             Dictionary_impl::get_target_dd_version());
    return true;
  }                                                   // ★ ② dd_version：严格 ==

  const RJ_Value &sdi_version_val = doc["sdi_version"];
  std::uint64_t sdi_version_ = sdi_version_val.GetUint64();
  if (sdi_version_ != SDI_VERSION) {
    my_error(ER_IMP_INCOMPATIBLE_SDI_VERSION, MYF(0), sdi_version_, SDI_VERSION);
    return true;
  }                                                   // ★ ③ sdi_version：严格 ==
  return false;
}
```

⇒ **只有 `mysqld_version_id` 允许"从旧版本导入"**（`<=`）；两个结构版本号必须**全等**，所以跨 minor 版本导入基本不可行。

三个版本号的取值来源：

```cpp
  w.String(STRING_WITH_LEN("mysqld_version_id"));
  w.Uint64(MYSQL_VERSION_ID);                                    // 80039，CMake 注入
  w.String(STRING_WITH_LEN("dd_version"));
  w.Uint(Dictionary_impl::get_target_dd_version());               // = dd::DD_VERSION = 80023
  w.String(STRING_WITH_LEN("sdi_version"));
  w.Uint64(SDI_VERSION);                                          // = 80019
```

（`DD_VERSION` 亦带 `static_assert(DD_VERSION <= MYSQL_VERSION_ID, "This release can not use a version number from the future");`）

★ 有意思的对照：**8.0.39 的 `dd_version` 是 80023、`sdi_version` 是 80019，但 `mysqld_version_id` 是 80039** —— 说明自 8.0.23 以来 DD 表结构与 SDI JSON 格式**都再没变过**。

## 9. 存储：SDI 是一棵"伪装的 InnoDB B-tree"

**SDI 不是自造格式，而是伪装成系统表的 InnoDB 聚簇索引**——定义在 `dict_sdi_create_idx_in_mem`：

```cpp
  snprintf(table_name, sizeof(table_name), "SDI_" SPACE_ID_PF, space);
  dict_mem_table_add_col(table, heap, "type",            DATA_INT,  ..., 4, true);
  dict_mem_table_add_col(table, heap, "id",              DATA_INT,  ..., 8, true);
  dict_mem_table_add_col(table, heap, "compressed_len",  DATA_INT,  ..., 4, true);
  dict_mem_table_add_col(table, heap, "uncompressed_len",DATA_INT,  ..., 4, true);
  dict_mem_table_add_col(table, heap, "data",            DATA_BLOB, ..., 0, true);
  const char *index_name = "CLUST_IND_SDI";
  dict_index_t *temp_index = dict_mem_index_create(table_name, index_name, space,
                                DICT_CLUSTERED | DICT_UNIQUE | DICT_SDI, 2);
```

即：每个表空间有一张**虚拟内存表** `SDI_<space_id>`，主键 **`(type, id)`**（2 字段聚簇唯一）。两种 type：

```cpp
/** Id for identifying Table SDIs */      constexpr uint32 SDI_TYPE_TABLE = 1;
/** Id for identifying Tablespace SDIs */ constexpr uint32 SDI_TYPE_TABLESPACE = 2;
```

**"多个表/分区的 SDI 是否在同一棵 B-tree"**：
- 同一表空间内的多个 SDI 记录**确实在同一棵 B-tree**
- 但**一个表只写一份**——写到 `fetch_first_tablespace_id()` 找到的**第一个**表空间（分区表：表级 SDI 只进第一个分区的表空间）

**记录是标准 InnoDB COMPACT/DYNAMIC 记录**（带 6 字节 trx_id + 7 字节 roll_ptr），`ibd2sdi.cc` 把偏移写死了（这是最权威的布局文档）：

```cpp
static const uint32_t REC_OFF_DATA_TYPE = REC_ORIGIN;                          // 0
static const uint32_t REC_OFF_DATA_ID   = REC_OFF_DATA_TYPE + REC_DATA_TYPE_LEN; // 4
static const uint32_t REC_OFF_DATA_TRX_ID = REC_OFF_DATA_ID + REC_DATA_ID_LEN;   // 12
static const uint32_t REC_OFF_DATA_ROLL_PTR = REC_OFF_DATA_TRX_ID + DATA_TRX_ID_LEN; // 18
static const uint32_t REC_OFF_DATA_UNCOMP_LEN = ... + DATA_ROLL_PTR_LEN;        // 25
static const uint32_t REC_OFF_DATA_COMP_LEN  = ... + REC_DATA_UNCOMP_LEN;       // 29
static const uint32_t REC_OFF_DATA_VARCHAR   = ... + REC_DATA_COMP_LEN;         // 33
```

⇒ **带 trx_id/roll_ptr 正是 SDI 能参与 MVCC 与回滚的物理基础**。

#### 根页号在 page 0 的**动态偏移**

```cpp
inline ulint fsp_header_get_sdi_offset(const page_size_t &page_size) {
  ulint offset = XDES_ARR_OFFSET + XDES_SIZE * xdes_arr_size(page_size) +
                 Encryption::INFO_MAX_SIZE;
  return (offset);
}
```

⚠️ **没有常量偏移**——紧跟 XDES 数组与加密信息之后，随 page size 变化。8 字节 = 4 字节 SDI version + 4 字节 root page no。读取时还会校验 `sdi_ver != SDI_VERSION` 并打 warning。

（页号布局：0 header、1 bitmap、2 ibuf bitmap、**3 SDI 根页**，所以 `ut_ad(root > 2)`、`SDI_BLOB_ALLOWED = 4`。）

#### 大表定义怎么办：三层机制

1. **先 zlib 压缩**（level 6）——`Sdi_Compressor::compress()`，⚠️ 失败（`Z_MEM_ERROR` 等）是 **`ib::fatal`**，直接崩实例
2. **BLOB 列走 off-page**（LOB），溢出页类型 `FIL_PAGE_SDI_BLOB` / `FIL_PAGE_SDI_ZBLOB`；压缩表的 SDI 溢出页**每页独立 inflate**
3. 多条记录时 B-tree 正常分裂

> ⚠️ **纠错**：第 1 条的"先 zlib 压缩"不是"大表才压"——**8.0.39 是无条件压缩**，代码里**没有任何 `if (sdi_len > 阈值)` 判断**：

```cpp
  Sdi_Compressor compressor(static_cast<uint32_t>(sdi_len), sdi);
  compressor.compress();                       // ★ 无条件，不看 sdi_len 大小

  err = ib_sdi_set(space_id, &ib_sdi_key, static_cast<uint32_t>(sdi_len),
                   compressor.get_comp_len(), compressor.get_data(), trx);
```

所以"三层机制"的准确表述是：① **无条件** zlib 压缩（level 6），原长与压后长度**都**存进记录；② 第 5 列是 `DATA_BLOB`，放不下时由 InnoDB **通用 off-page 机制**自动溢出，SDI 代码里**没有专门分支**；③ 记录落进 SDI B-tree，多了就正常分裂。

**`Sdi_Compressor`**（`storage/innobase/include/dict0sdi.h`）——注意任何 zlib 错误都是 `ib::fatal`（**崩实例**，不是返回错误码）：

```cpp
class Sdi_Compressor {
 public:
  Sdi_Compressor(uint32_t src_len, const void *sdi)
      : m_src_len(src_len), m_comp_len(), m_sdi(sdi), m_comp_sdi() {}
  ~Sdi_Compressor() { ut::free(m_comp_sdi); }

  inline void compress() {
    uLongf zlen = compressBound(static_cast<uLong>(m_src_len));
    auto src = reinterpret_cast<const Bytef *>(m_sdi);

    m_comp_sdi = static_cast<byte *>(
        ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, zlen));

    switch (compress2(m_comp_sdi, &zlen, src, static_cast<uLong>(m_src_len), 6)) {
      case Z_BUF_ERROR:    ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_FAILED_SDI_Z_BUF_ERROR); break;
      case Z_MEM_ERROR:    ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_FAILED_SDI_Z_MEM_ERROR); break;
      case Z_STREAM_ERROR: ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_SDI_Z_STREAM_ERROR);     break;
      case Z_OK:           m_comp_len = zlen; break;
      default:             ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_SDI_Z_UNKNOWN_ERROR);    break;
    }
  }
  const byte *get_data() const { return (m_comp_sdi); }
  uint32_t get_src_len() const { return (m_src_len); }
  uint32_t get_comp_len() const { return (m_comp_len); }
 private:
  uint32_t m_src_len;    // 未压缩长度
  uint32_t m_comp_len;   // 压缩后长度
  const void *m_sdi;
  byte *m_comp_sdi;
};
```

**一条 SDI 记录就是 5 个字段的元组**（`ib_sdi_create_insert_tuple()`）：

```cpp
static ib_tpl_t ib_sdi_create_insert_tuple(ib_crsr_t ib_crsr,
                                           const sdi_key_t *sdi_key,
                                           uint32_t uncomp_len, uint32_t comp_len,
                                           const void *sdi) {
  ut_ad(ib_crsr->prebuilt->index->get_field(0)->fixed_len == SDI_TYPE_LEN);   // 4
  ut_ad(ib_crsr->prebuilt->index->get_field(1)->fixed_len == SDI_KEY_LEN);    // 8

  ib_tpl_t tuple = ib_clust_read_tuple_create(ib_crsr);
  ib_col_set_value(tuple, 0, &sdi_key->type, SDI_TYPE_LEN, false);   // type    (4)
  ib_col_set_value(tuple, 1, &sdi_key->id, SDI_KEY_LEN, false);      // id      (8)
  ib_col_set_value(tuple, 2, &uncomp_len, 4, false);                 // uncomp_len(4)
  ib_col_set_value(tuple, 3, &comp_len, 4, false);                   // comp_len(4)
  ib_col_set_value(tuple, 4, sdi, comp_len, false);                  // data (压缩后)
  return (tuple);
}
```

**内部表的列定义**（`dict_sdi_create_idx_in_mem()`）——表名 `SDI_<space_id>`，第 5 列是 `DATA_BLOB`：

```cpp
  /* 18 = strlen(SDI) + Max digits of 4 byte spaceid (10) + 1 */
  char table_name[18];
  snprintf(table_name, sizeof(table_name), "SDI_" SPACE_ID_PF, space);

  constexpr size_t n_cols_sdi = 5;
  dict_table_t *table = dict_mem_table_create(table_name, space, n_cols_sdi, 0,
                                              0, table_flags, 0);

  dict_mem_table_add_col(table, heap, "type", DATA_INT,
                         DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "id", DATA_INT,
                         DATA_NOT_NULL | DATA_UNSIGNED, 8, true);
  dict_mem_table_add_col(table, heap, "compressed_len", DATA_INT,
                         DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "uncompressed_len", DATA_INT,
                         DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "data", DATA_BLOB, DATA_NOT_NULL, 0, true);
```

> ⚠️ **别把两个长度的列序写反**：`ib_col_set_value` 里是 uncomp 在**第 2 列**、comp 在**第 3 列**；而 `dict_mem_table_add_col` 的顺序是 `compressed_len`(第 3)、`uncompressed_len`(第 4)。

⚠️ **源码没有任何显式的 SDI 大小上限常量或注释**。理论上限来自 4 字节长度字段（≈4 GiB）。

#### 写的事务性：**先写 DD 表，后写 SDI，同一事务**

```cpp
  if (ctx.otx.open_tables() || object->impl()->store(&ctx.otx)) {
    return true;                                  // ① 先写 DD 表
  }
  ...
  if (... && sdi::store(thd, object)) return true; // ② 后写 SDI
```

SDI 写复用**当前 THD 的 InnoDB 事务**（`dict_sdi_set` 里 `trx_start_if_not_started` + `innobase_register_trx`），写入本身是普通 InnoDB DML，**没有独立的补偿逻辑**——失败则整个 DDL 事务回滚（这正是 4.4"一致性由同一事务保证"的具体机制）。

★ 一个优化：新旧 SDI 完全相同时**不产生更新**（`upd->n_fields == 0` 直接 `DB_SUCCESS`），避免无谓的 undo/redo。

## 10. ★ 读写路径：SDI 实际上"只写不读"

**这是一个反直觉但证据确凿的发现**：`hton->sdi_get` 在整个 `sql/` 下**没有任何调用点**；而 InnoDB 的 `dict_sdi_get` **整个函数体被 `#if 0` 掉了**：

```cpp
bool dict_sdi_get(const dd::Tablespace &, const sdi_key_t *, void *, uint64_t *) {
#if 0 /* TODO: Enable in WL#9761 */
        ... 完整实现 ...
#endif /* TODO: Enable in WL#9761 */
  ut_d(ut_error);
  ut_o(return (false));
}
```

→ **调用它在 debug 下直接 `ut_error`，release 下静默返回 false。这条读路径是死的。**（`dict_sdi_drop` 同样被 `#if 0`，注释说 *"This API should be used only when SDI is corrupted"*。）

**两段被关掉代码的完整原文**——注意它们的**参数全部无名**，所以 `#if 0` 里的代码一旦打开也编译不过，是"冻结的草稿"：

```cpp
bool dict_sdi_drop(dd::Tablespace *) {
#if 0  /* TODO: Enable in WL#9761 */
        uint32  space_id;
        if (dict_sdi_exists(tablespace, &space_id) != DB_SUCCESS) {
                return(true);
        }
        dberr_t err = ib_sdi_drop(space_id);
        return(err != DB_SUCCESS);
#endif /* TODO: Enable in WL#9761 */
  ut_d(ut_error);
  ut_o(return (false));
}
```

```cpp
bool dict_sdi_get(const dd::Tablespace &, const sdi_key_t *, void *, uint64_t *) {
#if 0 /* TODO: Enable in WL#9761 */
        ...
        uint32_t        uncompressed_sdi_len;
        uint32_t        compressed_sdi_len = static_cast<uint32_t>(*sdi_len);
        byte*           compressed_sdi = static_cast<byte*>(
                ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, compressed_sdi_len));

        dberr_t err = ib_sdi_get(space_id, &ib_sdi_key,
                                 compressed_sdi, &compressed_sdi_len,
                                 &uncompressed_sdi_len, trx);

        if (err == DB_OUT_OF_MEMORY) {
                *sdi_len = uncompressed_sdi_len;   /* ★ 缓冲不够：回写"需要的长度"让上层重来 */
        } else if (err != DB_SUCCESS) {
                *sdi_len = UINT64_MAX;
        } else {
                *sdi_len = uncompressed_sdi_len;
                /* Decompress the data */
                Sdi_Decompressor decompressor(static_cast<byte*>(sdi),
                                              uncompressed_sdi_len,
                                              compressed_sdi, compressed_sdi_len);
                decompressor.decompress();
        }
        ut::free(compressed_sdi);
        return(err != DB_SUCCESS);
#endif /* TODO: Enable in WL#9761 */
  ut_d(ut_error);
  ut_o(return (false));
}
```

> ★ 但这两个函数**仍然被挂到 handlerton 上**（`ha_innodb.cc:5211-5216`）：
>
> ```cpp
>   innobase_hton->sdi_create = dict_sdi_create;
>   innobase_hton->sdi_drop = dict_sdi_drop;
>   innobase_hton->sdi_get_keys = dict_sdi_get_keys;
>   innobase_hton->sdi_get = dict_sdi_get;
>   innobase_hton->sdi_set = dict_sdi_set;
>   innobase_hton->sdi_delete = dict_sdi_delete;
> ```
>
> 所以"只写不读"的准确表述是：`sdi_set` / `sdi_delete` / `sdi_get_keys` / `sdi_create` 是**活的**；`sdi_get` 与 `sdi_drop` 是**死的**。真正做删除的是 `dict_sdi_delete()`（**没有** `#if 0`）。

**所以真实的读场景只有三条**：

1. **`IMPORT TABLE FROM <pattern,...>`** —— 读的是**磁盘上的 `.sdi` 文件**，不是表空间里的 SDI
   > ⚠️ **语法纠错**：不存在 `ALTER TABLE ... IMPORT TABLESPACE FROM .sdi` 这种写法。8.0 的"逻辑重建"是**独立语句** `IMPORT TABLE FROM ...`（`SQLCOM_IMPORT` → `Sql_cmd_import_table`），与 `ALTER TABLE ... IMPORT TABLESPACE`（走 `.cfg` 物理搬运）是**两条完全不同的路径**。详见 §14。
2. **`ibd2sdi` 离线工具** —— 独立页级解析器
3. **InnoDB `row0import.cc`**（传统 IMPORT，8.0 已废弃路径）—— 只把 SDI 当作**需要校验/搬运的索引**，**不解析 JSON**（检查 `FSP_FLAGS_HAS_SDI`、调 `btr_root_adjust_on_import` 修 B-tree 段头）

★ **崩溃恢复不读 SDI**——redo 恢复里 SDI 只作为一个页类型出现（`MLOG_PAGE_CREATE_SDI` → `page_parse_create(..., FIL_PAGE_SDI)`），只做页重建，不做元数据解析。

⇒ **结论**：SDI 在 8.0.39 是**"写多读少"甚至"只写不读"**的。它的真实价值是**给外部工具（`ibd2sdi`）和未来的 IMPORT 实现留出自描述元数据**。

#### `ibd2sdi` 为什么必须是独立页级解析器

⚠️ 源码**没有直接陈述理由的注释**，但代码事实给出三条有据的原因：

1. **它必须能在服务器不运行时工作**——`utilities/` 下的独立可执行文件，只链接 InnoDB 的**页/文件层**（`fil0fil.h`、`fsp0fsp.h`、`page0page.h`、`lob0lob.h`），**没有 DD、没有 THD、没有 MDL、没有插件系统**。而 `dict_sdi_get` 签名要求 `const dd::Tablespace&`，内部还要 `check_trx_exists(current_thd)` —— 无法在无 server 环境调用
2. **`dict_sdi_get` 本身就是 `#if 0` 的死代码**——想走也没得走
3. **要处理损坏场景**——自带 `--no-checksum` / `--strict-check`，能绕过校验和；走正常 B-tree 路径遇到坏页会直接报错

⚠️ **代价**：`ibd2sdi` 必须自己重新实现记录布局解析、B-tree 下潜、LOB 页链遍历、zlib 解压、RapidJSON 解析与 pretty-print——**格式耦合被复制了一份**，InnoDB 改 SDI 物理布局必须同步改 `ibd2sdi.cc`。

## 11. 已知 bug 与限制

#### 两个真实 SDI bug（均已修，但代价是 `sdi_version` 递增）

| Bug | 内容 | 代价 |
|---|---|---|
| **Bug#29210646** | `Index_impl::m_hidden` **漏进 SDI** | `sdi_version` `1` → `80016` |
| **Bug#30326020** | **子分区**没反映进 SDI | `sdi_version` `80016` → `80019` |

★ 第二个 bug 还留着"后遗症处理"——`drop_tbl_sdi` 专门为"8.0.16 之前、尚未做过升级检查的子分区表"装了容错 handler，因为那些老表空间里**根本没有 SDI 记录**，删除会报 `ER_SDI_OPERATION_FAILED_MISSING_RECORD`，需要降级为 warning：

```cpp
  // Sub-partitioned tables from older versions which have not yet
  // been checked for upgrade may not have an SDI record in the SDI
  // index in the tablespace, so we need to install an error handler ...
```

#### ★ 两个 bug 的共同模式（值得记住）

SDI 是**手写**的字段清单（每个 `serialize()` 里一行一个 `write(...)`），**没有任何自动化的"字段完整性"校验**。加了一个 DD 成员而忘了加对应的 `write()`，SDI 就会**静默丢失**该字段。这正是 `SDI_VERSION` 注释里那句 *"adding a forgotten member"* 的来源。

#### 其它限制

- 顶层键的存在性/类型全是 `assert`，非错误检查（损坏 SDI 可能 UB）
- **`ER_IMP_INCOMPATIBLE_SDI_VERSION` 的格式串与实参类型不匹配**（`%llu` vs `%u`，一处真实瑕疵）
- DD 对象 id 不进 SDI ⇒ IMPORT 必然重新分配 id
- `collation_id` 裸 id ⇒ **跨实例不可移植**
- `sdi_version` 必须全等 ⇒ **跨 minor 版本导入基本不可行**
- `rctx` 的版本字段是死代码 ⇒ 兼容反序列化能力预留但未实现

---

## 12. 反序列化（读）那一半

**两个 context 是不对称的**——写侧 `Sdi_wcontext` 的 schema 名是**外面传进来的指针**，读侧 `Sdi_rcontext` 得自己存：

```cpp
class Sdi_wcontext {
  Byte_buffer m_buf;                       // 复用缓冲（base64 编码用）
  THD *m_thd;
  const String_type *m_schema_name;        // ★ 外部传入的指针
  bool m_error;
 public:
  const String_type &lookup_schema_name() const { return *m_schema_name; }   // 直接解引用
};

class Sdi_rcontext {
  Byte_buffer buf;
  dd_vector<Column *> m_column_object_opx;   // ★ opx → Column* 的映射表
  dd_vector<Index *> m_index_object_opx;     // ★ opx → Index* 的映射表
  THD *m_thd;
  uint m_target_dd_version;
  std::uint64_t m_sdi_version;
  bool m_error;
 public:
  String_type m_schema_name;               // ★ public：反序列化过程中被填进来
};
```

**`lookup_schema_ref()` 确实"不查找"**（实证）——对比 `lookup_tablespace_ref()` 就能看出差异：

```cpp
bool lookup_schema_ref(Sdi_rcontext *sdictx, const String_type &name,
                       dd::Object_id *idp) {
  sdictx->m_schema_name = name;   /* 唯一副作用：把 schema 名记到 rctx 上 */
  *idp = INVALID_OBJECT_ID;       /* ★ 不查 DD，不给真实 id */
  return false;                   /* 永远成功 */
}

bool lookup_tablespace_ref(Sdi_rcontext *sdictx, const String_type &name,
                           Object_id *idp) {
  return generic_lookup_ref<Tablespace>(sdictx->m_thd, MDL_key::TABLESPACE,
                                        name, idp);   /* ★ 这个是真查 DD，且会加 MDL */
}
```

⇒ 后果：反序列化出来的 `dd::Table` **没有 schema_id**，必须由调用方补——`Import_target::store_in_dd()` 里那句 `m_table_object->set_schema_id(schema->id())` 注释写得很直白：

> *"Since deserialization does no longer does this automatically, we must do it here to make the table object valid."*

而 `generic_deserialize()` 把 schema 名交回调用方的方式是 `std::move(rctx.m_schema_name)`：

```cpp
  if (dst->deserialize(&rctx, dd_object_val)) return checked_return(true);
  if (schema_name_from_sdi != nullptr) {
    *schema_name_from_sdi = std::move(rctx.m_schema_name);
  }
```

**opx 回查机制**（`column_opx` / `index_opx` 是怎么变回指针的）：

```cpp
template <typename W, typename PP>
void write_opx_reference(W *w, const PP &p, const char *key, size_t keysz) {
  uint opx = 0;
  if (p) {
    assert(p->ordinal_position() > 0);
    opx = p->ordinal_position() - 1;        /* ★ opx = ordinal_position - 1（0-based） */
    write(w, opx, key, keysz);
  }
}

template <typename PP, typename GV>
bool read_opx_reference(dd::Sdi_rcontext *rctx, PP *p, const GV &gv, const char *key) {
  uint opx = 0;
  if (read(&opx, gv, key)) return true;
  *p = get_by_opx(rctx, *p, opx);           /* ★ 用 *p 的静态类型做重载决议 */
  return false;
}
```

注册点只有两处，都在各自 `deserialize()` 的**最后一行**：

```cpp
  track_object(rctx, this);      // Column_impl::deserialize / Index_impl::deserialize 末尾
```

```cpp
template <typename T>
void generic_track_object(dd_vector<T *> *tvp, T *t) {
  assert(t->ordinal_position() > 0);
  uint opx = t->ordinal_position() - 1;
  if (opx >= tvp->size()) tvp->resize(opx + 1);
  (*tvp)[opx] = t;
}
```

> ★ **顺序依赖**：`columns` 数组在 `indexes` 之前，而 `indexes[].elements[].column_opx` 才引用列——所以引用方解析时被引用的 Column 一定已 `track_object`。这个顺序是 SDI 格式的一部分，不能随意调整。

前面 6. 讲的是 `serialize` 这一侧。读侧同样有一批函数，且**读比写更容易踩坑**——因为读进来的对象是不完整的，需要调用方补。

### 12.1 `generic_deserialize()` 骨架

```c
template <class Dd_type>
bool generic_deserialize(THD *thd, const Sdi_type &sdi, const String_type &object_type_name,
                         Dd_type *dst, const SdiCompatibilityChecker &comp_checker,
                         String_type *schema_name_from_sdi) {
  RJ_Document doc;
  doc.Parse<0>(sdi.c_str());
  if (doc.HasParseError()) { my_error(ER_INVALID_JSON_DATA, ...); return true; }
  if (comp_checker(doc)) return checked_return(true);
  ...
  Sdi_rcontext rctx(thd, doc["dd_version"].GetUint(), doc["sdi_version"].GetUint());
  if (dst->deserialize(&rctx, dd_object_val)) return checked_return(true);
  if (schema_name_from_sdi != nullptr) *schema_name_from_sdi = std::move(rctx.m_schema_name);
  return false;
}
```

即：**解析 → 兼容性检查（可插拔 checker）→ 构造 `rctx`（把两个版本号传进去供版本分支用）→ 交给对象自己的 `deserialize()` → 回传 schema 名**。

**真实调用点只有两处**：`sdi_api.cc`（IMPORT）与 `ndb_dd_sdi.cc`（NDB）。（加上单测 `dd_sdi-t.cc`。）

### 12.2 `CheckDefaultCompatibility()` 与"可插拔"

文档注释即定义：

```
/** Default checker which implements the traditional (strict) compatibility check:
    MYSQL_VERSION less than or equal, dd_version equal, and sdi_version equal. */
```

⚠️ **它是可替换的**——NDB 用自己的 `check_sdi_compatibility()`，注释明说：

```c
  // Skip dd_version and sdi_version checks to ensure compatibility during upgrades
```

即 NDB 为了升级期兼容，**故意放宽了 `dd_version`/`sdi_version` 两项**。

### 12.3 ★ `resolve_hton()` 在写侧，不在读侧

一个容易搞反的点：**反序列化根本不解析引擎插件**。`resolve_hton()` 只被 `store(Table)` / `store(Tablespace)` / `drop()` / `drop_after_update()` / `drop_all_impl()` 调用——全是写侧。

```c
template <typename DDT>
static handlerton *resolve_hton(THD *thd, const DDT &ddt) {
  plugin_ref pr = ha_resolve_by_name_raw(thd, lex_cstring_handle(ddt.engine()));
  if (pr) return plugin_data<handlerton *>(pr);
  return nullptr;
}
```

读侧拿到 `dd_object.engine` 只是**一个字符串字段**被原样读回（`ibd2sdi` 输出里能看到 `"engine": "InnoDB"`）。插件真正被解析是在之后落盘时。

（代码里还有个单测规避注释：*"Hack to avoid calling resolve_hton() during unit tests // reslove_hton() will crash in unit tests because the plugin_LOCK mutex has not been initialized."*）

### 12.4 引用解析：`lookup_schema_ref` 其实不查找

```c
bool lookup_schema_ref(Sdi_rcontext *sdictx, const String_type &name, dd::Object_id *idp) {
  sdictx->m_schema_name = name;
  *idp = INVALID_OBJECT_ID;
  return false;
}
```

⚠️ **它只把名字记下来，id 一律填 `INVALID_OBJECT_ID`**。所以反序列化出来的 Table 的 `schema_id` 是**无效的**，必须由调用方补——硬证据在 `sdi_api.cc`：

```c
  // Since deserialization does no longer does this automatically, we must do it here
  // to make the table object valid.
  m_table_object->set_schema_id(schema->id());
```

而 `lookup_tablespace_ref()` 是**真查 DD**（走 `generic_lookup_ref<T>()`）：先 `mdl_lock(... MDL_INTENTION_EXCLUSIVE)`，再 `dc->acquire(name, &p)`，**查不到就报错**（与 schema 的处理完全不同）。

### 12.5 opx 怎么解析回对象

`track_object()` / `get_by_opx()` 是 `Sdi_rcontext` 的 friend 函数：

```c
template <typename T> void generic_track_object(dd_vector<T *> *tvp, T *t) {
  uint opx = t->ordinal_position() - 1;
  if (opx >= tv.size()) tv.resize(opx + 1);
  tv[opx] = t;
}
```

即**建对象时按 `ordinal_position()-1` 登记进数组，之后 SDI 里出现的 opx 直接按下标取回指针**。⚠️ 只对 **Column / Index** 两种做跟踪。

### 12.6 `buf_handle()`：两个 context 的可复用缓冲

```c
char *generic_buf_handle(Byte_buffer *buf, size_t sz) {
  if (buf->reserve(sz)) { assert(false); return nullptr; }
  return &(*(buf->begin()));
}
```

⚠️ 注意它返回的是**缓冲区首地址而不是尾部**——每次调用复用同一块内存，调用方必须在下次 `buf_handle()` 之前消费完（写侧是 rapidjson 立即写出，读侧是立即解码）。写侧注释 *"A reusable byte buffer for e.g. base64 encoding"*，读侧是 decoding。

---

## 13. `store` / `drop` 家族：API 层与"按引擎能力分派"

这套函数在 `sdi.cc` 末尾的 `namespace dd { namespace sdi { ... } }` 里（⚠️ **不在 `sdi_api.cc`**——那个文件是 IMPORT 的 `Import_target`，见 14.）。

它把"序列化 + 按引擎能力分派到两种载体"封装成一个动作：

```c
bool store(THD *thd, const Table *tp) {
  handlerton *hton = resolve_hton(thd, t);
  return with_schema(thd, t.schema_id(), [&](const Schema &s) {
    dd::Sdi_type sdi = serialize(thd, t, s.name());
    if (sdi.empty()) return checked_return(true);
    return checked_return(hton->sdi_set ? sdi_tablespace::store_tbl_sdi(thd, hton, sdi, t, s)
                                        : sdi_file::store_tbl_sdi(sdi, t, s));
  });
}
```

⚠️ **Tablespace 没有文件回退**：

```c
bool store(THD *thd, const Tablespace *ts) {
  ...
  if (hton->sdi_set == nullptr) return false;  // SDI api not supported
```

### 13.1 ★ `drop_after_update` 为什么必须"after update"

`sdi.h` 的文档注释给出了场景：*DD 的 store 若是一次 update，会写一份新的 SDI；若 SDI 存在文件里且 update 改了表名，就必须在……之后删掉旧文件*。

`dictionary_client.cc` 的调用点注释是完整答案：

```c
  /*
    Remove the old sdi after store() has successfully written the
    new one. Note that this is a noop unless we are writing the SDI
    to file and the name of the new object is different. (If the
    names are identical the old file will be over-written by
    store(). If we are storing the SDI in a tablespace the key
    does not depend on the name and the store is a transactional
    update).
  */
```

**完整的 `drop_after_update()`**——三道早退决定"到底要不要删"：

```cpp
bool drop_after_update(THD *thd, const Table *old_tp, const Table *new_tp) {
  const Table &old_t = ptr_as_cref(old_tp);
  const Table &new_t = ptr_as_cref(new_tp);

  if ((old_t.schema_id() == new_t.schema_id() &&
       equal_prefix_chars_name(old_t, new_t, sdi_file::FILENAME_PREFIX_CHARS))
      // Hack to avoid calling resolve_hton() during unit tests
      // reslove_hton() will crash in unit tests because the
      // plugin_LOCK mutex has not been initialized.
      || old_t.engine() == "innodb") {
    return false;                                  // ① 文件名没变（原地覆盖）或本来就走表空间
  }

  const handlerton &old_hton = ptr_as_cref(resolve_hton(thd, old_t));
  if (old_hton.sdi_set) {
    return false;                                  // ② 老引擎走表空间 → key 与名字无关
  }

  return with_schema(thd, old_t.schema_id(), [&](const Schema &s) {
    return checked_return(sdi_file::drop_tbl_sdi(old_t, s));   // ③ 文件型 SE + 改名/跨 schema
  });
}
```

> ★ 判"文件名是否变化"用的是**前 16 字符前缀 + 同 schema**，不是全名比较——因为 `.sdi` 文件名里被裁掉的尾部不影响路径：

```cpp
/** Number of character (not bytes) of a tablename which will
    contrubute to the sdi file name. The whole name is not needed as
    the Object_id is added so that uniqueness is ensured */
const size_t FILENAME_PREFIX_CHARS = 16;
```

（`equal_prefix_chars()` 按 `my_mbcharlen()` 逐字符推进，所以是**字符数**不是字节数。）

**家族清单**（8.0.39 实际只有 6 个，且**没有** `store(Schema)` / `drop(Tablespace)`）：

```cpp
  // Note that there is NO drop() overload for Tablespace. Dropping a
  // tablespace implies that all sdis in it are dropped also.
```

| 函数 | 载体 | 说明 |
|---|---|---|
| `store(Table)` / `store(Tablespace)` | `hton->sdi_set` ? 表空间 : 文件 | Tablespace 无文件回退 |
| `drop(Table)` | `hton.sdi_delete` ? 表空间 : 文件 | — |
| `drop_after_update(Table, Table)` | **只处理文件侧** | 改名后删旧文件 |
| `drop_all_for_table` / `drop_all_for_part` | 表空间 | 遍历所有 tablespace，`sdi_get_keys` 枚举后逐个删 |

拆开是三条：

1. **顺序**：新 SDI 必须先写成功——旧 SDI 是此刻唯一可用的副本，先删旧的再写新的一半崩了就两个都没了
2. **为什么不能在 update 前删**：判断"文件名是否变了"需要 `old_object` 与 `new_object` 的**对比结果**，而这只能在 store 之后拿到
3. **什么时候真的需要删**：`.sdi` 文件名 = 表名前 **16 个字符** + Object_id（`FILENAME_PREFIX_CHARS = 16`）。`RENAME TABLE` 改了名字 → 新文件是新名字、旧的变孤儿；若前 16 字符没变，新 store 直接覆盖，无需删

代码里的 noop 条件：

```c
if ((old_t.schema_id() == new_t.schema_id() &&
     equal_prefix_chars_name(old_t, new_t, sdi_file::FILENAME_PREFIX_CHARS))
    || old_t.engine() == "innodb") return false;
const handlerton &old_hton = ptr_as_cref(resolve_hton(thd, old_t));
if (old_hton.sdi_set) return false;   // 载体是表空间：key 与名字无关，事务性就地更新
return with_schema(thd, old_t.schema_id(), [&](const Schema &s) {
  return checked_return(sdi_file::drop_tbl_sdi(old_t, s));   // 只有 .sdi 文件载体才真删
});
```

顺带解释了 `equal_prefix_chars()` / `equal_prefix_chars_name()` / `with_schema()` 三个函数：**前两个是多字节安全的前缀比较**（用 `my_mbcharlen` 逐字符推进），唯一业务用途就是上面这个"前 16 字符是否相同"的判断；`with_schema()` 是"按 schema_id 取 Schema 对象再执行闭包"的便捷封装。

---

## 14. ★ 两条 IMPORT 是两回事（物理搬运 vs 逻辑重建）

这是 SDI 最容易混淆的地方——**名字都叫 IMPORT，机制完全不同**。

### 14.1 `IMPORT TABLE FROM <pattern,...>`：逻辑重建

> ⚠️ **真名纠错**：常被写成 `ALTER TABLE ... IMPORT TABLESPACE FROM .sdi`，但**语法里没有这个东西**。8.0 的"逻辑重建"是一条**独立语句**（`sql_yacc.yy`）：

```yacc
import_stmt:
          IMPORT TABLE_SYM FROM TEXT_STRING_sys_list
          {
            LEX *lex= Lex;
            lex->m_sql_cmd=
              new (YYTHD->mem_root) Sql_cmd_import_table($4);
            if (lex->m_sql_cmd == NULL)
              MYSQL_YYABORT;
            lex->sql_command= SQLCOM_IMPORT;
          }
```

⇒ 它与 §14.2 的 `ALTER TABLE ... IMPORT TABLESPACE` **在语法层面就是两条路**，只是名字像。

**结论先行：这条路径只读 `.sdi` 文件，从不从表空间读 SDI。**（整个 `sql/` 树里 `hton->sdi_get` **没有任何调用点**，见 10.）

**"逻辑重建"到底重建了什么**——`Import_target::store_in_dd()` 的核心只有三件事：查 schema、补 `schema_id`、存进 DD：

```cpp
bool Import_target::store_in_dd(THD *thd) const {
  dd::cache::Dictionary_client &dc = *thd->dd_client();
  const Schema *schema = nullptr;
  if (dc.acquire(m_schema_name_in_sdi, &schema)) return true;
  if (schema == nullptr) {
    my_error(ER_IMP_SCHEMA_DOES_NOT_EXIST, MYF(0), m_schema_name_in_sdi.c_str());
    return true;
  }
  // Since deserialization does no longer does this automatically, we
  // must do it here to make the table object valid.
  m_table_object->set_schema_id(schema->id());      /* ★ 补 schema_id（见 12.4） */
  ...
  if (dd::sdi_file::check_data_files_exist(m_schema_name_in_sdi,
                                           m_table_object->name()))
    return true;                                     /* ★ 只查 .MYD / .MYI */
  if (dc.store(m_table_object.get())) return true;
  ...
}
```

> ★ `check_data_files_exist()` 只检查 **`.MYD` / `.MYI`** —— 这直接证明 `IMPORT TABLE FROM` **面向的是非 InnoDB 引擎**（MyISAM 那类"数据文件 + .sdi 文件"的组合），InnoDB 表走的是 §14.2 的 `.cfg` 物理搬运。

```cpp
bool check_data_files_exist(const dd::String_type &schema_name,
                            const dd::String_type &table_name) {
  build_table_filename(path, sizeof(path) - 1, schema_name.c_str(),
                       table_name.c_str(), ".MYD", 0, &was_truncated);
  MY_STAT sa;
  if (nullptr == my_stat(path, &sa, MYF(0))) { ... return true; }
  path[plen - 3] = 0;
  strcat(path, "MYI");
  ...
}
```

调用链：`sql_import.cc` → `dd::sdi::Import_target`（`sdi_api.cc`）→ `sdi_file::load()` + `dd::deserialize()`。

**① 装两道回滚守卫**（`Sql_cmd_import_table::execute()`）：

```c
  auto rbgrd = dd::sdi_utils::make_guard(thd, [&](THD *) { trans_rollback_stmt(thd); trans_rollback(thd); });
  auto tgtgrd = dd::sdi_utils::make_guard(thd, [&](THD *) { for (auto &tgt : targets) tgt.rollback(); });
```

**② 权限与锁**：`check_access(FILE_ACL)`；`sdi_file::expand_pattern()` 展开成 `(path, in_datadir)` 列表；每个表 `MDL_EXCLUSIVE`、涉及的 schema `MDL_INTENTION_EXCLUSIVE`，再加 BACKUP_LOCK 与 GLOBAL 锁。

**③ 真正读出并重建**（`Import_target::load()`）：

```c
  if (dd::sdi_file::load(thd, m_path, shared_buffer)) return true;
  if (dd::deserialize(thd, *shared_buffer, m_table_object.get(), &m_schema_name_in_sdi)) return true;
  if (!dd::has_primary_key(*m_table_object) && thd->variables.sql_require_primary_key) { ... }
  if (!m_in_datadir) return false;
  dd::String_type tmpname{m_path};
  tmpname.insert(tmpname.length() - 4, "_import");     // t1.sdi → t1_import.sdi
  if (mysql_file_rename(key_file_sdi, m_path.c_str(), tmpname.c_str(), MYF(MY_FAE))) return true;
```

⚠️ 那个 `t1.sdi → t1_import.sdi` 的改名很关键——避免后续 `open_tables` / `check_data_files_exist` 把它当成"已存在的表定义"或留下孤儿。

**④ 写入 DD**：`dc.acquire(m_schema_name_in_sdi, &schema)`（不存在 → `ER_IMP_SCHEMA_DOES_NOT_EXIST`）→ **补 `set_schema_id()`**（见 12.4）→ 检查表是否已存在 → `check_data_files_exist()` → `dc.store()` → `update_referencing_views_metadata()` → `open_tables()`。

⚠️ `open_tables()` 的**错误被降级为 warning**，注释：*"Downgrade all errors from open to warnings as we don't want to fail import here, but rather give the user a chance to repair the table."* —— 即导入成功但表打不开是允许的，留给用户修。

**⑤ 提交/回滚**：成功则 `tgt.commit()`（删掉 `*_import.sdi`，删除失败**也降级为 warning**）；失败则 `rbgrd` 回滚 DD 事务（SDI 与 DD 在同一事务，一起回滚），`tgtgrd` 把 `*_import.sdi` 改回原名。

### 14.2 传统 `ALTER TABLE ... IMPORT TABLESPACE`（走 `.cfg`）：物理搬运

**完全不解析 SDI JSON，只把 SDI 当作"多出来的一个 B-tree"来搬运**。`row0import.cc` 的硬证据：

- `bool m_has_sdi; /*!< true if tablespace has SDI */`；`cfg->m_has_sdi = FSP_FLAGS_HAS_SDI(m_space_flags);`
- 扫索引时跳过 SDI：*"Since there are SDI Indexes before normal indexes, we check for FIL_PAGE_INDEX type."*
- 索引数量对账要**额外算上 SDI 那一个**：`UT_LIST_GET_LEN(m_table->indexes) + (m_has_sdi ? 1 : 0) != m_n_indexes`
- `PageConverter` 显式处理 SDI 页族：`case FIL_PAGE_SDI:` 要先解压；`case FIL_PAGE_SDI_BLOB: case FIL_PAGE_SDI_ZBLOB:` 与 BLOB 一起处理
- 根页调整：`if (FSP_FLAGS_HAS_SDI(space->flags)) { ... btr_root_adjust_on_import(sdi_index); }`

★ **从 5.7 的 `.ibd` 导入时会现建一个 SDI 索引**：

```c
  if (!FSP_FLAGS_HAS_SDI(space_flags_from_disk)) {
    /* This is IMPORT from 5.7 .ibd file or pre 8.0.1 */
    dict_sdi_remove_from_cache(table->space, nullptr, true);
    btr_sdi_create_index(table->space, true);
    if (upgrade_space_version(table->space, false)) return row_import_error(...);
  }
```

**一句话对照**：传统 IMPORT 是**物理搬运 + 索引根页重定位**（SDI 只是被照搬的一个 B-tree，没有就现建一个空的）；`IMPORT ... FROM .sdi` 是**逻辑重建**（从 `.sdi` 反序列化出 `dd::Table` 再写进 DD）。

---

## 15. 5.7 升级与 SDI：为什么升级期禁写

### 15.1 守卫机制

`upgrade_57/upgrade.cc` 有个全局开关与访问器：

```c
/*
  This variable is used to skip creation of SDI files for tables
  and InnoDB tablespaces when creating entry in dictionary. SDI entry
  These entries are created as last phase of upgrade.
*/
static bool create_sdi_flag = true;
bool allow_sdi_creation() { return create_sdi_flag; }
```

真正的守卫点在 `storage_adapter.cc`：

```c
  // Do not create SDIs for tablespaces and tables while creating
  // dictionary entry during upgrade.
  if (bootstrap::DD_bootstrap_ctx::instance().get_stage() > bootstrap::Stage::CREATED_TABLES &&
      dd::upgrade_57::allow_sdi_creation() && sdi::store(thd, object)) return true;
```

开关的时机：`do_pre_checks_and_initialize_dd()` 里 `set_allow_sdi_creation(false)`（紧接在 `ha_migrate_tablespaces` 前）；`fill_dd_and_finalize()` 里同样关掉（紧接 `migrate_all_frm_to_dd()` 前），之后 `set_allow_sdi_creation(true)`；最后在 `fill_dd_and_finalize()` 尾部 `add_sdi_info(thd)` 批量补写。

⚠️ **关于"为什么"要如实写**：注释给出的理由只有"SDI 条目作为升级的最后阶段统一创建"，**没有更细的 rationale**（不要额外猜测性能动机）。

### 15.2 批量补写 `add_sdi_info()`：两轮，每对象一个事务

- **第一轮 tablespaces**：`fetch_global_component_names<Tablespace>()` → 逐个 `ha_resolve_by_name_raw` 拿 hton → `upgrade_space_version()` → **`hton->sdi_create(ts)`（只有实现了 `sdi_create` 的引擎才写）** → `dc->update()` → 每个表空间**单独事务**提交
- **第二轮 tables**：`fetch_global_component_ids<Table>()` → 逐个 `dd::sdi::store(thd, table)` → 每个表单独事务

**`add_sdi_info()` 的完整实现**（`sql/dd/upgrade_57/upgrade.cc`）——注意注释里三轮都明说"出错也继续"：

```cpp
bool add_sdi_info(THD *thd) {
  // Fetch list of tablespaces. We will ignore error in storing SDI info
  // as upgrade can only roll forward in this stage. Use Error handler to avoid
  // any error calls in dd::sdi::store()
  std::vector<dd::String_type> tablespace_names;
  Dummy_error_handler error_handler;
  ...
  thd->push_internal_handler(&error_handler);
  for (dd::String_type &tsc : tablespace_names) {
    ...
    // In case of error, we will continue with upgrade.
    if (hton && hton->upgrade_space_version(ts))
      LogErr(ERROR_LEVEL, ER_DD_UPGRADE_FAILED_TO_UPDATE_VER_NO_IN_TABLESPACE, ...);

    if (hton && hton->sdi_create) {
      // Error handling not possible at this stage, upgrade should complete.
      if (hton->sdi_create(ts))
        LogErr(ERROR_LEVEL, ER_FAILED_TO_STORE_SDI_FOR_TABLESPACE, ts->name().c_str());

      if (thd->dd_client()->update(ts)) {
        trans_rollback_stmt(thd);
        LogErr(ERROR_LEVEL, ER_FAILED_TO_STORE_SDI_FOR_TABLESPACE, ...);
      }
      trans_commit_stmt(thd);
      trans_commit(thd);                 // ★ 每个 tablespace 一个独立事务
    }
    mem_root.ClearForReuse();
  }
  thd->pop_internal_handler();

  // Fetch list of tables from dictionary
  std::vector<dd::Object_id> table_ids;
  ...
  thd->push_internal_handler(&error_handler);
  for (dd::Object_id &table_id : table_ids) {
    ...
    if (dd::sdi::store(thd, table)) {
      LogErr(ERROR_LEVEL, ER_UNKNOWN_TABLE_IN_UPGRADE, table->name().c_str());
      trans_rollback_stmt(thd);
    }
    trans_commit_stmt(thd);
    trans_commit(thd);                   // ★ 每个 table 一个独立事务
    mem_root.ClearForReuse();
  }
  thd->pop_internal_handler();

  if (Upgrade_status().update(Upgrade_status::enum_stage::SDI_INFO_UPDATED))
    return true;
  ...
}
```

**"升级期 SDI 写失败不让升级失败"的两处实证**：

```cpp
  /*
    Add status to mark creation and initialization of dictionary.
    We will be modifying innodb tablespaces now.
    After this step, upgrade process can only roll forward.
  */
  if (Upgrade_status().update(Upgrade_status::enum_stage::USER_TABLE_UPGRADED))
    return true;

  // Add SDI information to all tablespaces
  if (add_sdi_info(thd))
    LogErr(ERROR_LEVEL, ER_DD_UPGRADE_SDI_INFO_UPDATE_FAILED);   // ★ 只打日志，没有 return！
```

```cpp
      case Upgrade_status::enum_stage::USER_TABLE_UPGRADED: {
        LogErr(INFORMATION_LEVEL, ER_DD_UPGRADE_FOUND_PARTIALLY_UPGRADED_DD_CONTINUE);
        if (restart_dictionary(thd)) return true;
        // Ignore error in this stage and continue with server restart.
        (void)add_sdi_info(thd);                                  // ★ 显式丢弃返回值
        (void)migrate_stats(thd);
        finalize_upgrade(thd);
        return false;
      }
```

⇒ 原因就在那段注释：**此时 InnoDB 表空间已被改写，回滚不回去了**，只能向前滚动。SDI 只是冗余副本，写失败可以事后补——`dict_sdi_create()` 就是专门给"无 SDI 的老表空间"补建 SDI 索引的入口。

**失败处理（★ 关键）**：

```c
  // Fetch list of tablespaces. We will ignore error in storing SDI info
  // as upgrade can only roll forward in this stage. Use Error handler to avoid
  // any error calls in dd::sdi::store()
```

除了两处 `fetch_global_component_*` 失败会返回 true（且调用方**也只是打日志不终止升级**），其余**全部是 `LogErr(ERROR_LEVEL, ...) + continue`**，并用 `Dummy_error_handler` 吞掉 `store()` 里的 `my_error()`。注释：*"Error handling not possible at this stage, upgrade should complete."*

即：**升级期 SDI 写失败不会让升级失败**——因为该阶段**只能向前滚动**。

**另一条触发路径**：部分升级后重启时 `(void)add_sdi_info(thd);`，注释 *"Ignore error in this stage and continue with server restart."*

**擦屁股逻辑**：升级失败回滚时 `drop_dd_table()` 会删掉 datadir 里已生成的 `.SDI` 文件。

---

## 16. 其他引擎怎么处理 SDI：no-op 是一种"显式不做"

### 16.1 handlerton 回调与默认行为

**handlerton 上只有 6 个 SDI 回调**（`sql/handler.h`）——**没有** `sdi_get_filename`，文件名是 SQL 层的 `dd::sdi_file::sdi_filename()`：

```cpp
  /*
    APIs for retrieving Serialized Dictionary Information by tablespace id
  */
  sdi_create_t sdi_create;        // 建 SDI 索引（升级 / SDI 损坏后重建）
  sdi_drop_t sdi_drop;            // 删整个 SDI 索引（仅 SDI 损坏时用）
  sdi_get_keys_t sdi_get_keys;    // 枚举一个表空间里的所有 SDI key
  sdi_get_t sdi_get;              // 按 key 读 SDI
  sdi_set_t sdi_set;              // 按 key 写/更新 SDI
  sdi_delete_t sdi_delete;        // ★ 按 key 删一条 SDI（日常删除走这个，不是 sdi_drop）
```

全仓 grep `sdi_(set|get|drop|create|delete|get_keys)\s*=` 在 `storage/` 下**只命中 4 个文件**：`ha_innodb.cc`、`ha_ndbcluster.cc`、`ha_ndbinfo.cc`、`ha_perfschema.cc`。其它引擎（Archive / Blackhole / CSV / Heap / MyISAM / TempTable…）**全部留 nullptr** ⇒ 走 `.sdi` 文件回退。

**★ NDB 的一个反直觉细节**——它自己反序列化时**故意跳过 dd_version / sdi_version 检查**：

```cpp
static bool check_sdi_compatibility(const dd::RJ_Document &doc) {
  const dd::RJ_Value &mysqld_version_id = doc["mysqld_version_id"];
  if (mysqld_version_id.GetUint64() > std::uint64_t(MYSQL_VERSION_ID)) {
    my_error(ER_IMP_INCOMPATIBLE_MYSQLD_VERSION, MYF(0), ...);
    return true;
  }
  // Skip dd_version and sdi_version checks to ensure compatibility during
  // upgrades
  return false;
}
```

⇒ 与 `dd::CheckDefaultCompatibility()` 的"严格三查"形成鲜明对照：NDB 把 SDI 当作**跨集群复制的元数据**用，不能因版本号不等就拒收。这是 `SdiCompatibilityChecker` 设计成**可插拔函数**的实际收益——`generic_deserialize()` 接受它作参数。

`sdi_create` / `sdi_drop` / `sdi_get` / `sdi_set` / `sdi_delete` / `sdi_get_keys` 都是 `sql/handler.h` 里的函数指针。规则来自 13. 的分派：

| 引擎实现情况 | 表 | 表空间 |
|---|---|---|
| `sdi_set == nullptr` | **回退写 `.sdi` 文件**（表名前 16 字符 + id + `.sdi`） | **不写**（`store(Tablespace)` 直接 return false） |
| `sdi_set != nullptr` | 写表空间 | 写表空间 |

`drop_all_impl()` 里还有 `assert(hton.sdi_get != nullptr && hton.sdi_delete != nullptr);`——不实现这两个的引擎走这条路径会在 debug 下断言。

### 16.2 ★ NDB：既不写文件也不写表空间

（⚠️ 纠正一个望文生义：`ndb_dd_sdi.cc` **不是回调实现**，它只有 `prettify` / `serialize` / `deserialize` 三个辅助函数。真正的回调在 `ndb_dummy_ts.h`。）

```c
// These are dummy callback implementations as there is no
// point in keeping .SDI files for each table stored in
// the data directory of MySQL Server. The tables and
// their metadata is safely stored in the transactional
// dictionary of NDB
static bool sdi_set(...)  { return false; }   // 无 assert，正常路径会走到
static bool sdi_create(dd::Tablespace *) { assert(false); /* Never called */ return false; }
```

★ **机制很精妙**：`sdi_set` 是**非空的 no-op** ⇒ `store()` 走"写表空间"分支 ⇒ 但 `fetch_first_tablespace_id(table)` 对 NDB 表返回 `INVALID_OBJECT_ID` ⇒ `store_tbl_sdi()` 里 `if (res.value == nullptr) return false;` 直接返回，**什么都不发生**。同时因为 `sdi_set != nullptr`，服务器**不会**落到 `.sdi` 文件回退分支。

⇒ **这是个通用手法**：想"既不写文件也不写表空间"，就把 `sdi_set` 实现成 no-op；想"写文件"，就留 `nullptr`。

### 16.3 Performance Schema：运行时动态切换

同一手法的另一个例子，而且是**运行时开关**：

```c
void pfs_sdi_disable() {   // 全装 no-op → 不产生 .sdi 文件
  pfs_hton->sdi_set = pfs_sdi_set_ignored; ...
}
void pfs_sdi_enable() {    // 全部置 nullptr → 走 .sdi 文件回退
  pfs_hton->sdi_set = nullptr; ...
}
```

调用者在 `log0pfs.cc` 的 `log_pfs_create_tables()`：创建 redo log 的 PFS 表时**先 disable、建完再 enable**（失败路径也会恢复）。

---

## 17. SDI 页族 layout 全景（精确偏移）

> SDI 是**独立的页类型**——而且是**三种**（索引页 + 两种溢出页）。本节给出每种页的字节级布局，偏移量都标注来源常量，可对着图算出每个字节在哪。

### 17.1 三种页速查

| 页类型 | 值 | 数据起点 | 页内私有头 | 页间串链 | 存在于 |
|---|---|---|---|---|---|
| **`FIL_PAGE_SDI`** | **17853** | 记录区 94 / 首条用户记录 120 | 56B PAGE_HEADER + 2×10B FSEG header（仅根页） | `FIL_PAGE_PREV`(8) / `FIL_PAGE_NEXT`(12) 同层双向链 | 所有含 SDI 的表空间 |
| **`FIL_PAGE_SDI_BLOB`** | **18** | `FIL_PAGE_DATA + LOB_HDR_SIZE` = **46** | 8B：`LOB_HDR_PART_LEN`(38) + `LOB_HDR_NEXT_PAGE_NO`(42) | 页内 offset **42** 的 4B next | COMPACT / DYNAMIC |
| **`FIL_PAGE_SDI_ZBLOB`** | **19** | `FIL_PAGE_DATA` = **38** | **无** | `FIL_PAGE_NEXT`(**12**)；`FIL_PAGE_FILE_FLUSH_LSN`(26) 复用为 owner 反查指针 | 仅 COMPRESSED |

**17853 的来历**：`FIL_PAGE_INDEX = 17855`、`FIL_PAGE_RTREE = 17854`、`FIL_PAGE_SDI = 17853`——**连续递减三个整数**（0x45BF / 0x45BE / 0x45BD），R-tree 与 SDI 是后来从 INDEX 往下分配的号段。除这三个外**所有其他页类型都在 0..29** 的小数值区间。
⚠️ **源码没有任何注释解释为什么是 17853**（`i_s.cc` 只提到"大数值不能当数组下标"），不要编造"魔数校验"之类的说法。

**定义处**（注意：在 `fil0fil.h`，**不是** `fil0types.h`；后者只有页内偏移如 `FIL_PAGE_DATA=38`）：

```cpp
/* storage/innobase/include/fil0fil.h */
/** B-tree node */
constexpr page_type_t FIL_PAGE_INDEX = 17855;
/** R-tree node */
constexpr page_type_t FIL_PAGE_RTREE = 17854;
/** Tablespace SDI Index page */
constexpr page_type_t FIL_PAGE_SDI = 17853;
```

```cpp
/** Uncompressed SDI BLOB page */
constexpr page_type_t FIL_PAGE_SDI_BLOB = 18;
/** Compressed SDI BLOB page */
constexpr page_type_t FIL_PAGE_SDI_ZBLOB = 19;
```

> ⚠️ 18/19 是"小号段"，与普通的 `FIL_PAGE_TYPE_BLOB=10` / `ZBLOB=11` 不是同一段。

**★ page 0 的 SDI 根页号偏移 = 10505 是"算出来的"，不是常量**——`fsp_header_get_sdi_offset()`：

```cpp
inline ulint fsp_header_get_sdi_offset(const page_size_t &page_size) {
  ulint offset = XDES_ARR_OFFSET + XDES_SIZE * xdes_arr_size(page_size) +
                 Encryption::INFO_MAX_SIZE;
#ifdef UNIV_DEBUG
  ulint left_size =
      page_size.physical() - FSP_HEADER_OFFSET - offset - FIL_PAGE_DATA_END;
  ut_ad(left_size >= FSP_SDI_HEADER_LEN);
#endif
  return (offset);
}
```

代入 16 KiB 页逐项展开：

| 项 | 计算 | 值 |
|---|---|---|
| `FSP_HEADER_OFFSET` | = `FIL_PAGE_DATA` | 38 |
| `FSP_HEADER_SIZE` | `32 + 5 × FLST_BASE_NODE_SIZE(16)` | 112 |
| `XDES_ARR_OFFSET` | `38 + 112` | **150** |
| `XDES_SIZE` | `XDES_BITMAP(24) + UT_BITS_IN_BYTES(64×2)=16` | 40 |
| `xdes_arr_size(16K)` | `16384 / FSP_EXTENT_SIZE(64)` | 256 |
| `XDES_SIZE × n` | `40 × 256` | **10240** |
| `Encryption::INFO_MAX_SIZE` | `3+4+64+36+4 +4` | **115** |
| **合计** | `150 + 10240 + 115` | **10505** ✅ |

> ★ 这个偏移**随页大小变化**（`xdes_arr_size` 依赖 `page_size.physical()`），所以 10505 只对 16 KiB 页成立——这正是它必须写成函数而不是宏的原因。相邻字段也印证位置：`fsp_header_get_encryption_progress_offset()` = `fsp_header_get_sdi_offset() + FSP_SDI_HEADER_LEN`。

**读写 page 0 的那两个字节组**（前 4 字节是 SDI_VERSION，后 4 字节是根页号）：

```cpp
page_no_t fsp_sdi_get_root_page_num(space_id_t space, const page_size_t &page_size,
                                    mtr_t *mtr) {
  buf_block_t *block = buf_page_get(page_id_t(space, 0), page_size, RW_S_LATCH,
                                    UT_LOCATION_HERE, mtr);
  page_t *page = buf_block_get_frame(block);

  ulint sdi_offset = fsp_header_get_sdi_offset(page_size);
  uint32_t sdi_ver = mach_read_from_4(page + sdi_offset);

  if (sdi_ver != SDI_VERSION) {
    ib::warn(ER_IB_MSG_426) << "SDI version mismatch. Expected: " << SDI_VERSION
                            << " Current version: " << sdi_ver;
  }
  ut_ad(sdi_ver == SDI_VERSION);

  page_no_t root = mach_read_from_4(page + sdi_offset + 4);
  ut_ad(root > 2);
  return (root);
}

void fsp_sdi_write_root_to_page(page_t *page, const page_size_t &page_size,
                                page_no_t root_page_num, mtr_t *mtr) {
  ulint sdi_offset = fsp_header_get_sdi_offset(page_size);
  /* Write SDI version here. */
  mlog_write_ulint(page + sdi_offset, SDI_VERSION, MLOG_4BYTES, mtr);
  /* Write SDI root page number */
  mlog_write_ulint(page + sdi_offset + 4, root_page_num, MLOG_4BYTES, mtr);
}
```

> ⚠️ **这里读写的 `SDI_VERSION` 是 InnoDB 页头版（=1），不是 JSON 里的 80019** —— 见 §8 的"两个同名 SDI_VERSION"。

**"有没有 SDI"还有一个信号：page 0 的 space flags**

```cpp
/** Return the value of the SDI field */
constexpr uint32_t FSP_FLAGS_HAS_SDI(uint32_t flags) {
  return (flags & FSP_FLAGS_MASK_SDI) >> FSP_FLAGS_POS_SDI;
}
dberr_t fsp_has_sdi(space_id_t space_id) {
  ...
  return (FSP_FLAGS_HAS_SDI(space->flags) ? DB_SUCCESS : DB_ERROR);
}
```

### 17.2 ★ `FIL_PAGE_SDI` 索引页：与 `FIL_PAGE_INDEX` **字节级完全相同**

三条硬证据：

```c
/** Check whether the page type is index (Btree or Rtree or SDI) type */
inline bool fil_page_type_is_index(page_type_t page_type) {
  return page_type == FIL_PAGE_INDEX || page_type == FIL_PAGE_SDI ||
         page_type == FIL_PAGE_RTREE;
}
```

1. 上面这个函数把 SDI 与 B-tree/R-tree 归为**同一族**
2. 建页走的是**同一个函数** `page_create_low()`，只在 `fil_page_set_type()` 上区分三者（`ut_ad(page_type == FIL_PAGE_INDEX || FIL_PAGE_RTREE || FIL_PAGE_SDI)`）
3. 记录级 redo 恢复时统一用 `fil_page_type_is_index()` 放行（如 `case MLOG_REC_INSERT: ut_ad(!page || fil_page_type_is_index(page_type));`）

⇒ **FIL header、PAGE_HEADER、infimum/supremum、记录堆、page directory、FIL trailer 与 `FIL_PAGE_INDEX` 逐字节一致；SDI 页没有自己的页头结构。**

**通用布局不在本篇展开**——FIL header(38B) / PAGE_HEADER(56B) / infimum@99 / supremum@112 / 首条用户记录@120 / page directory / FIL trailer(8B) 的**完整逐字节图见 [`innodb/physical/page_structure.md`](../innodb/physical/page_structure.md)「① INDEX 页」**，SDI 页可直接套用（该篇「完整清单（30 种）」也含 SDI 的三种页类型）。本篇只列 **SDI 专属的三处差异**：

| 位置 | `FIL_PAGE_INDEX` | `FIL_PAGE_SDI` |
|---|---|---|
| `FIL_PAGE_TYPE`（offset 24, 2B） | 17855 | **17853** |
| `PAGE_INDEX_ID`（offset 66, 8B） | 真实 index id | **恒 `0xFFFFFFFFFFFFFFFF`**（`dict_sdi_get_index_id()` 返回 `UINT64_MAX`）——可作为识别 SDI 页的**第二指纹** |
| 建页 redo | `MLOG_COMP_PAGE_CREATE` | **`MLOG_COMP_PAGE_CREATE_SDI`**（见 17.7） |

其余全部沿用：`PAGE_LEVEL`(64)、`PAGE_N_RECS`(54)、`PAGE_BTR_SEG_LEAF`(74) / `PAGE_BTR_SEG_TOP`(84) 的 10B FSEG header（仅根页）、page directory（slot n 地址 `16384 − 8 − 2*(n+1)`，每 slot 管辖 4~8 条）、`FIL_PAGE_PREV`(8) / `FIL_PAGE_NEXT`(12) 同层双向链。

### 17.3 ★ 记录 layout（定长偏移 + 变长编码）

**权威列定义**（`dict_sdi_create_idx_in_mem()`，5 列）：

```c
  dict_mem_table_add_col(table, heap, "type", DATA_INT, DATA_NOT_NULL|DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "id",   DATA_INT, DATA_NOT_NULL|DATA_UNSIGNED, 8, true);
  dict_mem_table_add_col(table, heap, "compressed_len",   DATA_INT, ..., 4, true);
  dict_mem_table_add_col(table, heap, "uncompressed_len", DATA_INT, ..., 4, true);
  dict_mem_table_add_col(table, heap, "data", DATA_BLOB, DATA_NOT_NULL, 0, true);
  ...
  const char *index_name = "CLUST_IND_SDI";   // DICT_CLUSTERED | DICT_UNIQUE | DICT_SDI, 2 字段
  temp_index->add_field("type", 0, true);
  temp_index->add_field("id", 0, true);
```

主键 `(type, id)`；`DB_TRX_ID`(6) / `DB_ROLL_PTR`(7) 由 `dict_table_add_system_columns()` 追加（因为索引唯一，**不**加 `DB_ROW_ID`）。

**聚簇索引的物理字段顺序**：

| # | 字段 | 长度 | 来源 |
|---|---|---|---|
| 0 | `type` | 4 | PK 第 1 列 |
| 1 | `id` | 8 | PK 第 2 列 |
| 2 | `DB_TRX_ID` | 6 | 系统列 |
| 3 | `DB_ROLL_PTR` | 7 | 系统列 |
| 4 | 定长长度列 #1 | 4 | 表列 |
| 5 | 定长长度列 #2 | 4 | 表列 |
| 6 | `data` | 变长 | 表列 |

**记录内字节布局**（相对记录原点 `rec`；`REC_OFF_*` 常量链）：

```
 高地址 ──────────────────────────────────────────────────► 低地址

 rec-7  [可选] 变长长度低 8 位（2 字节编码时出现）
 rec-6  变长长度字节：1B 编码 = 长度；2B 编码 = 0x80|0x40?(|len>>8)
 rec-5  4 bits info-bits(delete-mark/min-rec) + 4 bits n_owned
 rec-4  ┐
 rec-3  ┘ 13 bits heap_no + 3 bits record type
 rec-2  ┐
 rec-1  ┘ 2 bytes next-record offset（页内相对，有符号）
═══════════════ ORIGIN (rec) ═══════════════════════════════════
 rec+0   type               4B   REC_OFF_DATA_TYPE        big-endian
 rec+4   id                 8B   REC_OFF_DATA_ID          big-endian
 rec+12  DB_TRX_ID          6B   REC_OFF_DATA_TRX_ID
 rec+18  DB_ROLL_PTR        7B   REC_OFF_DATA_ROLL_PTR
 rec+25  定长长度列 #1      4B   REC_OFF_DATA_UNCOMP_LEN  (=25)
 rec+29  定长长度列 #2      4B   REC_OFF_DATA_COMP_LEN    (=29)
 rec+33  data（变长）            REC_OFF_DATA_VARCHAR     (=33)
         ├─ 内联前缀：0B (DYNAMIC/COMPRESSED) 或 768B (COMPACT)
         └─ 20B BLOB 引用（BTR_EXTERN_FIELD_REF_SIZE）
```

⚠️ **源码里一处命名冲突，务必注意**：`dict0crea.cc` 把列命名为 `compressed_len`（第 3 列）/`uncompressed_len`（第 4 列），但 **`api0api.cc` 与 `ibd2sdi.cc` 实际存取的语义正好相反**——offset 25 是**未压缩**长度、offset 29 是**压缩后**长度：

```c
  ib_col_set_value(tuple, 2, &uncomp_len, 4, false);   // 列 #2 = 未压缩长度
  ib_col_set_value(tuple, 3, &comp_len, 4, false);     // 列 #3 = 压缩后长度
```

`ibd2sdi.cc` 的用法与之自洽：`ut_ad(rec_data_length == sdi_comp_len);`（页内取出的字节数 == 压缩长度）。⇒ **磁盘格式不受影响**，是 `dict0crea.cc` 的一个命名 bug。

**变长长度编码**（`rem0rec.cc`，`lens` 指针从 `rec-6` 向下递减）：

```c
      } else if (dfield_is_ext(field)) {
        *lens = (byte)(len >> 8) | 0xc0;   // 外部存储：0x80|0x40
        lens--;  *lens = (byte)len;  lens--;
      } else {
        if (len < 128 || !DATA_BIG_LEN_MTYPE(...)) { *lens = (byte)len; lens--; }
        else { *lens = (byte)(len >> 8) | 0x80;  lens--;  *lens = (byte)len;  lens--; }
      }
```

| 位 | 含义 |
|---|---|
| `rec-6` bit7 (`0x80`) | **2 字节长度**标志；为 0 → 只有 1 字节，长度 0..127 |
| `rec-6` bit6 (`0x40`) | **外部存储**标志（仅当 `0x80` 置位时有意义） |
| `rec-6` bit5..0 + `rec-7` | 长度（最大 `0x3FFF` = 16383，`ut_ad(len < 16384)`） |

⚠️ **字节序是"高位靠原点"**：`rec-6` = 高字节、`rec-7` = 低字节。

★ **外部存储时长度字段存的不是真实数据长度**，而是「内联前缀长度 + 20」：

| 行格式 | `len` 字段值 | `rec-6` | `rec-7` | 内联前缀 | 真实总长来源 |
|---|---|---|---|---|---|
| DYNAMIC / COMPRESSED | 0 + 20 = **20** | `0xC0` | `0x14` | **0 字节** | BLOB 引用的 `BTR_EXTERN_LEN`(8B) |
| COMPACT / REDUNDANT | 768 + 20 = **788** | `0xC3` | `0x14` | **768 字节** | 同上 |

**行格式由表空间 flags 决定，不是硬编码**：

```c
  if (zip_ssize > 0)          rec_format = REC_FORMAT_COMPRESSED;
  else if (atomic_blobs)      rec_format = REC_FORMAT_DYNAMIC;
  else                        rec_format = REC_FORMAT_COMPACT;
```

8.0 新建表空间 `atomic_blobs` 置位 ⇒ **DYNAMIC**（零前缀）。

**20 字节 BLOB 引用**（`lob0lob.h`，`BTR_EXTERN_FIELD_REF_SIZE = 20`）：`SPACE_ID`(0) / `PAGE_NO`(4) / `OFFSET`(8) / `LEN`(12, 8B，高 2 位是标志)。

### 17.4 ★ 两种溢出页布局**完全不同**（最容易搞混）

**① `FIL_PAGE_SDI_BLOB`（18）—— 未压缩**

```c
/** Offset within header of LOB length on this page. */
const ulint LOB_HDR_PART_LEN = 0;
/** Offset within header of next BLOB part page no. FIL_NULL if none */
const ulint LOB_HDR_NEXT_PAGE_NO = 4;
/** Size of an uncompressed LOB page header, in bytes */
const ulint LOB_HDR_SIZE = 8;
```

**16 KiB 页的完整布局**：

```
偏移     内容
─────────────────────────────────────────────────────────────────┐
0x0000  ┌─ FIL Header (38B) ────────────────────────────────────┐│
0x0000  │ [+0,+4)  FIL_PAGE_SPACE_OR_CHKSUM  校验和              ││
0x0004  │ [+4,+8)  FIL_PAGE_OFFSET           页号                ││
0x0008  │ [+8,+12) FIL_PAGE_PREV            ★ 本类页不用于串链    ││
0x000C  │ [+12,+16)FIL_PAGE_NEXT            ★ 本类页不用于串链    ││
0x0010  │ [+16,+24)FIL_PAGE_LSN                                  ││
0x0018  │ [+24,+26)FIL_PAGE_TYPE            ★★ = 18              ││
0x001A  │ [+26,+34)FIL_PAGE_FILE_FLUSH_LSN                       ││
0x0022  │ [+34,+38)FIL_PAGE_SPACE_ID                             ││
0x0026  ├───────────────────────────────────────────────────────┤│
0x0026  ┌─ LOB Header (8B) ─────────────────────────────────────┐│
0x0026  │ [+0,+4) 本页数据字节数              LOB_HDR_PART_LEN    ││
0x002A  │ [+4,+8) 下一页页号 / FIL_NULL       LOB_HDR_NEXT_PAGE_NO││
0x002E  ├───────────────────────────────────────────────────────┤│
0x002E  │ payload：本页 LOB 数据（最多 16330B）                   ││
        │ …… 未用完的部分内容未定义 ……                            ││
0x3FF8  ├───────────────────────────────────────────────────────┤│
0x3FF8  │ FIL Trailer (8B)                                      ││
0x4000  └───────────────────────────────────────────────────────┘│
─────────────────────────────────────────────────────────────────┘
```

⚠️ 注意：串链**不是**用 FIL header 的 `PREV`/`NEXT`（那两个是给同层 B-tree 页用的），而是用 LOB Header 里的 `LOB_HDR_NEXT_PAGE_NO`（页内 offset 42）。这是 BLOB 页与索引页最容易搞混的一点。

每页容量 = `physical − FIL_PAGE_DATA − LOB_HDR_SIZE − FIL_PAGE_DATA_END`，16 KiB 页 = **16330 字节**。第一页入口：BLOB 引用里 `BTR_EXTERN_OFFSET` 写的是 `FIL_PAGE_DATA`。

> ⚠️ 本版本**没有 `LOB_HDR_DATA_LEN` 这个常量**（8.0.39 LOB 重构后已移除），未压缩 LOB 头只有上述两个 4B 字段。

**② `FIL_PAGE_SDI_ZBLOB`（19）—— 压缩，布局和串链方式都不一样**

```c
  void log_page_type(page_t *blob_page, ulint nth_blob_page) {
    if (is_index_sdi())      page_type = FIL_PAGE_SDI_ZBLOB;
    else if (nth_blob_page == 0) page_type = FIL_PAGE_TYPE_ZBLOB;
    else                     page_type = FIL_PAGE_TYPE_ZBLOB2;
```

👉 普通压缩 LOB 第一页是 `ZBLOB`、后续是 `ZBLOB2`；**SDI 压缩 LOB 每一页都是 `FIL_PAGE_SDI_ZBLOB`（19）**，没有 ZBLOB2 变体。

**完整布局**（`P` = `page_zip_get_size()`，压缩表空间的物理页大小，如 8K = 8192）：

```
偏移     内容
─────────────────────────────────────────────────────────────────┐
0x0000  ┌─ FIL Header (38B) ────────────────────────────────────┐│
0x0000  │ [+0,+4)  FIL_PAGE_SPACE_OR_CHKSUM                      ││
0x0004  │ [+4,+8)  FIL_PAGE_OFFSET           页号                ││
0x0008  │ [+8,+12) FIL_PAGE_PREV           ★★ 写死 FIL_NULL      ││
0x000C  │ [+12,+16)FIL_PAGE_NEXT           ★★ 这就是链表指针      ││
0x0010  │ [+16,+24)FIL_PAGE_LSN                                  ││
0x0018  │ [+24,+26)FIL_PAGE_TYPE           ★★ = 19               ││
0x001A  │ [+26,+30)FIL_PAGE_FILE_FLUSH_LSN ★ 复用 = owner space_id││
0x001E  │ [+30,+34)          +4            ★ 复用 = owner 记录所在页号││
0x0022  │ [+34,+38)FIL_PAGE_SPACE_ID                             ││
0x0026  ├───────────────────────────────────────────────────────┤│
0x0026  │ ★★★ zlib 数据流直接开始——没有 LOB_HDR、没有长度字段 ★★★││
        │ …… 一直写到 physical 页尾（**不减** FIL_PAGE_DATA_END）…││
        │ 未用完的部分：memset 0                                  ││
   P    └───────────────────────────────────────────────────────┘│
─────────────────────────────────────────────────────────────────┘
```

⚠️ **ZBLOB 页没有 FIL trailer**——zlib 数据一直写到页尾。这正是容量公式用 `physical − FIL_PAGE_DATA`（不减 `FIL_PAGE_DATA_END`）的原因，也是它与 BLOB 页（有 8B trailer）的关键区别。

⚠️ 另一个反直觉点：**串链用的是 FIL header 的 `FIL_PAGE_NEXT`(12)**，而不是 BLOB 页那种页内私有字段。所以 `FIL_PAGE_PREV` 被显式写死成 `FIL_NULL`（源码注释：*"Initialize the unused 'prev page' pointer"*），只留 `NEXT` 单向串。

每页容量 = `physical − FIL_PAGE_DATA`（**不减** `FIL_PAGE_DATA_END`）。串链用 `FIL_PAGE_NEXT`(12) 而非 `LOB_HDR_NEXT_PAGE_NO`。

★ **不是每页独立压缩，是跨页的单一 zlib 流**：`zInserter::prepare()` 只 `deflateInit2` 一次，每换页只重设 `next_out/avail_out` 继续 `deflate(&m_stream, Z_FINISH)`，直到 `Z_STREAM_END`。读侧同理一条 `m_stream` 逐页 `inflate(Z_NO_FLUSH)`。

⇒ **页头没有任何"本页解压后长度"字段**：

| 信息 | 存放位置 |
|---|---|
| 解压后总长 | 记录定长列（offset 25） |
| 压缩后总长 | 记录定长列（offset 29），= BLOB 引用 `BTR_EXTERN_LEN` |
| 本页字节数 | **没有**——只能靠 `FIL_PAGE_NEXT` 是否 `FIL_NULL` 判断是否最后一页 |
| redo | **整页** `mlog_log_string` 全量字节串（未压缩版是字段级 `mlog_write_ulint`/`mlog_write_string`） |

#### ★ payload 里装的是什么：两层压缩 + 没有 index 页

**① payload 是纯字节流，没有内部结构**

未压缩页的本页长度在 `LOB_HDR_PART_LEN`；压缩页**连长度字段都没有**（zlib 流自描述，只能靠 `FIL_PAGE_NEXT == FIL_NULL` 判断是否最后一页）。

**② ★ payload 里装的是"压缩后的 SDI"，不是 JSON 明文**

写入侧（`api0api.cc`）：

```c
  Sdi_Compressor compressor(*sdi_len, sdi);
  err = ib_sdi_set(tablespace_id, &sk, *sdi_len, compressor.get_comp_len(), ...);
```

⇒ JSON 先经 `Sdi_Compressor`（zlib）压成 **C 字节**，**C 字节才是 `data` 列的内容**，也才是溢出页 payload 里装的东西。

**③ ★ 压缩表空间下是"双重 zlib"**

| 层 | 谁做的 | 作用 |
|---|---|---|
| 外层（**仅压缩表空间**） | `zInserter` 的 `deflate`（跨页单一流） | 把 **C 字节再压一遍**写进 `FIL_PAGE_SDI_ZBLOB` |
| 内层（**总是**） | `Sdi_Compressor` | 把 JSON 压成 C 字节 |

读侧顺序（`ibd2sdi.cc`）：

```c
  if (page_size.is_compressed())  copy_compressed_blob(...);    // ① 页级 inflate → C 字节
  else                            copy_uncompressed_blob(...);  // ① 直接拷     → C 字节
  ut_ad(rec_data_length == sdi_comp_len);
  Sdi_Decompressor decompressor(uncompressed_sdi, sdi_uncomp_len + 1,
                                str, sdi_comp_len);             // ② uncompress → JSON
```

⇒ 未压缩表空间：`JSON --zlib--> C --存--> SDI_BLOB 页`（**一层**）
⇒ 压缩表空间：`JSON --zlib--> C --zlib--> SDI_ZBLOB 页`（**两层**）

**④ ★ SDI 的 LOB 没有 index 页——源码硬编码跳过**

8.0 的新 LOB 是 `LOB_FIRST` / `LOB_DATA` / **`LOB_INDEX`** 三件套（index 页用于 O(1) 定位第 n 个 chunk）。但 SDI **强制走无 index 路径**：

```c
  if (page_zip != nullptr) {
    DBUG_EXECUTE_IF("lob_insert_single_zstream", { goto insert_single_zstream; });
    if (dict_index_is_sdi(index)) {
      goto insert_single_zstream;      // 压缩：单一 zstream + 无 index
    }
  } else {
    /* Uncompressed LOB */
    DBUG_EXECUTE_IF("lob_insert_noindex", { goto insert_noindex; });
    if (dict_index_is_sdi(index)) {
      goto insert_noindex;             // 未压缩：无 index
    }
  }
```

⇒ 所以 SDI 溢出**虽然用的是新 LOB 的代码路径，语义却退化成老式 BLOB 链表**：只会出现 `FIL_PAGE_SDI_BLOB`(18) 与 `FIL_PAGE_SDI_ZBLOB`(19)，**永远不会出现 `FIL_PAGE_TYPE_LOB_INDEX`(22) / `LOB_DATA`(23) / `LOB_FIRST`(24)**。这正是 `ibd2sdi` 能用简单的"逐页拼 + next 指针"读完的原因——代价是不能随机访问某个 chunk，但 SDI 本来就整块读。

页类型也是同一批代码特判出来的：

```c
    if (is_index_sdi())      page_type = FIL_PAGE_SDI_BLOB;   // 非 SDI 才是 FIL_PAGE_TYPE_BLOB
    ...
    if (is_index_sdi())      page_type = FIL_PAGE_SDI_ZBLOB;  // 非 SDI 才分 ZBLOB / ZBLOB2
```

### 17.5 ★ page 0 的 SDI 元数据：偏移量是**算出来的**

```c
inline ulint fsp_header_get_sdi_offset(const page_size_t &page_size) {
  ulint offset = XDES_ARR_OFFSET + XDES_SIZE * xdes_arr_size(page_size) +
                 Encryption::INFO_MAX_SIZE;
  return (offset);
}
static inline ulint xdes_arr_size(const page_size_t &page_size) {
  return (page_size.physical() / FSP_EXTENT_SIZE);
}
```

16 KiB 页逐项代入：

| 量 | 值 |
|---|---|
| `FSP_HEADER_OFFSET` = `FIL_PAGE_DATA` | 38 |
| `FSP_HEADER_SIZE` = `32 + 5*16` | 112 |
| `XDES_ARR_OFFSET` = 38 + 112 | **150** |
| `XDES_SIZE` | **40** |
| `xdes_arr_size()` = 16384 / 64 | **256** |
| `Encryption::INFO_MAX_SIZE` | **115** |

```
fsp_header_get_sdi_offset(16K) = 150 + 40*256 + 115 = 10505
```

```
10505  ┌ SDI Header (FSP_SDI_HEADER_LEN = 8) ─┐
10505  │ [+0,+4) SDI_VERSION = 1              │
10509  │ [+4,+8) SDI 根页号                   │
10513  └──────────────────────────────────────┘
10513  ┌ Encryption Progress（1B + 4B）       │
       ⋯⋯  FIL trailer（最后 8B）⋯⋯
```

写入顺序（低地址 4B = 版本，高地址 4B = 根页号）：

```c
  ulint sdi_offset = fsp_header_get_sdi_offset(page_size);
  mlog_write_ulint(page + sdi_offset, SDI_VERSION, MLOG_4BYTES, mtr);
  mlog_write_ulint(page + sdi_offset + 4, root_page_num, MLOG_4BYTES, mtr);
```

> ⚠️ **纠错**：有些资料（含本文早前引用的一处批注）把 SDI Header 写成 `≈10654`，并称加密信息"约 260~264 字节"。**按源码公式应为 10505**，加密信息 `INFO_MAX_SIZE` 是 **115 字节**。
> 另注：`dict0dict.h` 里 `SDI_VERSION` 的注释说 *"Written on Page 1 & 2"* 是**陈旧注释**，实际写的是 page 0。

**FSP flags 里的 SDI 标志位 = bit 14**：

```
bit:  0        1..4       5         6..9      10    11    12   13   14   15
    +--------+---------+---------+----------+-----+-----+----+----+----+----+
    | POST_  | ZIP_    | ATOMIC_ | PAGE_    |DATA |SHARE|TEMP|ENC |SDI |SM4 |
    | ANTE.. | SSIZE   | BLOBS   | SSIZE    |_DIR | D   |    |    |    |    |
    +--------+---------+---------+----------+-----+-----+----+----+----+----+
```

`FSP_FLAGS_POS_SDI = 14`、`FSP_FLAGS_MASK_SDI = 0x4000`、`FSP_FLAGS_HAS_SDI(flags) = (flags & MASK) >> 14`。

### 17.6 根页、段与分裂

- **有完整的段结构**——`btr_sdi_create()` 调的是通用 `btr_create()`，走 `fseg_create()` 分配 leaf/top 两个 file segment，根页上 `PAGE_BTR_SEG_LEAF`(74) 与 `PAGE_BTR_SEG_TOP`(84) 各 10 字节 FSEG header 都写上了
- **初始 1 页**。新 `.ibd` 是 **7 页**（`FIL_IBD_FILE_INITIAL_SIZE`），布局注释：

```
0: file space header   1: insert buffer bitmap   2: inode page
3: sdi index page      4: index page             5,6: freshly allocated
```

SDI 根页按惯例落在 **page 3**，但**这不是硬编码**——根页号由 `fseg_create()` 分配，必须从 page 0 读（`fsp_sdi_get_root_page_num()` 里只有 `ut_ad(root > 2)`）。5.7 老文件是 4 页（无 SDI），`fil0fil.cc` 用 SDI flag 区分这两种初始大小。

- **分裂无专属逻辑**：走通用 `btr_page_split_and_insert()` / `btr_root_raise_and_insert()`，新页类型由 `dict_index_is_sdi(index) → FIL_PAGE_SDI` 决定
- **识别 SDI 页的两个指纹**：`FIL_PAGE_TYPE` = 17853；`PAGE_INDEX_ID`(66) = `0xFFFFFFFFFFFFFFFF`（`dict_sdi_get_index_id()` 返回 `UINT64_MAX`）

### 17.7 redo：只有"建页"是专属的

```c
  /** create a SDI index page */   MLOG_PAGE_CREATE_SDI = 63,
  /** create a SDI compact page */ MLOG_COMP_PAGE_CREATE_SDI = 64,
```

```c
static inline void page_create_write_log(buf_frame_t *frame, mtr_t *mtr,
                                         bool comp, page_type_t page_type) {
  switch (page_type) {
    case FIL_PAGE_INDEX: type = comp ? MLOG_COMP_PAGE_CREATE : MLOG_PAGE_CREATE; break;
    case FIL_PAGE_RTREE: type = comp ? MLOG_COMP_PAGE_CREATE_RTREE : MLOG_PAGE_CREATE_RTREE; break;
    case FIL_PAGE_SDI:   type = comp ? MLOG_COMP_PAGE_CREATE_SDI : MLOG_PAGE_CREATE_SDI; break;
    default: ut_error;
  }
  mlog_write_initial_log_record(frame, type, mtr);
}
```

⇒ **没有任何 payload**（只有 type 1B + space_id + page_no，上限 11B）。恢复时是 **physiological 日志**：`page_parse_create(block, comp, FIL_PAGE_SDI)` → `page_create_low()` **从零重建一个空页**（清零 PAGE_HEADER 前 26B、写 `PAGE_N_DIR_SLOTS=2`、置 `PAGE_N_HEAP = 0x8000|2`、写 infimum/supremum 模板与两个 directory slot、空闲区 memset 0），随后由后续 `MLOG_COMP_REC_INSERT` 等把记录插回去。

**其余全部复用通用类型**：记录插入/删除/更新用 `MLOG_COMP_REC_*`（恢复时用 `fil_page_type_is_index()` 放行）；BLOB 页类型字段用 `mlog_write_ulint`；ZBLOB 页体用 `mlog_log_string` 整页全量。

⇒ **崩溃恢复不解析 SDI 语义**——SDI 只作为一个页类型出现，只做页重建（与 10. 的"服务器只写不读"一致）。

### 17.8 ★ 端到端字节追踪：从 `.ibd` 文件到一条 JSON

前面几节是"零件图"，这里把它们串成**一条可执行的路径**。以 16 KiB 页、DYNAMIC 行格式、单表 `t1`（`space_id = S`）为例，追踪读出一个 SDI 记录的全过程——**每一步都给出要读的偏移量**：

**① 读 page 0**（文件偏移 `0`，长 16384）

| 偏移 | 读什么 | 期望 |
|---|---|---|
| 24 (2B) | `FIL_PAGE_TYPE` | `8`（`FSP_HDR`） |
| 34 (4B) | `FIL_PAGE_SPACE_ID` | `S` |
| 38+ | `FSP header` → `FSP_SPACE_FLAGS` | 检查 **bit 14**（`FSP_FLAGS_HAS_SDI`）== 1 ⇒ 该表空间"声称"有 SDI |
| **10505** (4B) | `SDI_VERSION` | `1`（不符**只 warn 不失败**） |
| **10509** (4B) | SDI 根页号 `R` | `ut_ad(R > 2)`，惯例 `= 3` |

**② 读根页 `R`**（偏移 `R × 16384`）

- 偏移 24 (2B)：`FIL_PAGE_TYPE == 17853` ✓ ← **页类型校验**
- 偏移 66 (8B)：`PAGE_INDEX_ID == 0xFFFFFFFFFFFFFFFF` ✓ ← **第二指纹**
- 偏移 64 (2B)：`PAGE_LEVEL`
  - `> 0` → 非叶页：从 infimum(99) 出发，`next` 偏移在 `99-2`，**跳过 type(4)+id(8) 后的 4B 就是子页号**；要求子页号 `>= SDI_BLOB_ALLOWED(4)` 且 level **恰好减 1**
  - `= 0` → 已是叶子，进 ③

**③ 在叶子页取第一条用户记录**

- 偏移 54 (2B)：`PAGE_N_RECS == 0` ⇒ 打印 "SDI is empty" 并成功返回
- `memcmp(99, "infimum", 7)` 校验伪记录；第一条用户记录从 **120** 起
- `next` 偏移在 `rec-2`；沿链表走，遇 supremum 则按 `FIL_PAGE_NEXT`(12) 翻页，`FIL_NULL` 结束

**④ 解析一条记录**（设记录原点 `rec`）

```
rec+0   (4B) type        → 1 = Table, 2 = Tablespace
rec+4   (8B) id
rec+12  (6B) DB_TRX_ID
rec+18  (7B) DB_ROLL_PTR
rec+25  (4B) 未压缩长度 U   ★ 注意 dict0crea.cc 的列名与此相反
rec+29  (4B) 压缩长度 C
rec+33       data（变长）
```

`data` 的长度先看 **`rec-6`**：

- `& 0x80 == 0` → 1 字节编码，长度就是 `rec-6` 的值（≤127）
- `& 0x80 != 0` → 2 字节：`((rec-6) & 0x3F) << 8 | rec-7`
  - `& 0x40 != 0` → **外部存储**，此时长度字段存的是「内联前缀 + 20」而非真实长度：
    - **DYNAMIC**：`0 + 20 = 20` ⇒ 前缀 **0 字节**，紧跟 20B BLOB 引用
    - **COMPACT**：`768 + 20 = 788` ⇒ 前缀 **768 字节**，再跟 20B 引用

**⑤ 若外部存储，从 BLOB 引用拿到第一页**（引用相对 `rec+33`）

| 引用内偏移 | 含义 |
|---|---|
| +0 (4B) | space_id |
| **+4 (4B)** | 第一页页号 `B` |
| +8 (4B) | 页内偏移，恒 = `FIL_PAGE_DATA`(38) |
| **+12 (8B)** | 外部总长，应 == `C` |

**⑥ 逐页拼接溢出数据**

- `FIL_PAGE_SDI_BLOB`(18)：页内 **38**(4B) = 本页字节数、**42**(4B) = 下一页；数据从 **46** 起
- `FIL_PAGE_SDI_ZBLOB`(19)：**无 LOB_HDR**，数据从 **38** 起；下一页在 `FIL_PAGE_NEXT`(**12**)

每页都要校验页类型，直到 next == `FIL_NULL`。拼完得到 **C 字节**。

**⑦ 解压 → 得到 JSON**

```
C 字节 --Sdi_Decompressor(zlib uncompress)--> U 字节 = 紧凑单行 JSON
```

⚠️ 解压失败是 `ib::fatal`，**直接 abort 进程**（不可恢复）。

**⑧ 输出**（`ibd2sdi` 侧）

```
["ibd2sdi",                                  ← fprintf 手写的信封，第 0 个元素
 {"type": 1, "id": 330, "object": {...}},     ← --pretty 默认开 ⇒ PrettyWriter 重排
 {"type": 2, "id": 7,   "object": {...}}
]
```

⇒ 所以 `jq '.[1]["object"]["dd_object"]["columns"]'` 的下标**从 1 起**。

---

## Misc

### 工程技法

1. **紧凑 JSON + 顶层元数据信封**——`mysqld_version_id` / `dd_version` / `sdi_version` / `dd_object_type` / `dd_object` 五键固定，读侧先查版本再决定能否解析（读必须用 DOM，写用流式 Writer，读写不对称）
2. **反向 pimpl 当编译防火墙**——`Sdi_wcontext` / `Sdi_rcontext` 是空壳 + `friend` 自由函数，让 40 多个 `*_impl.cc` 不必 include RapidJSON 与 THD（见 §6）
3. **存储完全复用 InnoDB**——SDI 不发明新格式，而是伪装成一张内存表 `SDI_<space_id>`（5 列、主键 `(type,id)`），从而**免费获得** MVCC、事务回滚、zlib 压缩、LOB 溢出、B-tree 分裂（见 §9）。代价是根页号偏移依赖 XDES 数组与加密信息，只能运行时算
4. **二进制与属性降级为字符串**——`default_value` 走 base64、`options`/`se_private_data` 走裸 `key=value;...` 串，避免为它们发明 JSON 子结构
5. **引用不解引用成对象**——只写 opx 序号 / 名字 / 裸 id 三种形式（见 §7），从而保持"一次序列化一棵树"的单一遍历

### ★★★ 真名勘误与数字核实

| 常见说法 | 8.0.39 实际 |
|---|---|
| `ALTER TABLE ... IMPORT TABLESPACE FROM .sdi` | **该语法不存在**。逻辑重建是独立语句 **`IMPORT TABLE FROM <pattern,...>`**（`SQLCOM_IMPORT` → `Sql_cmd_import_table`，`sql_yacc.yy:18126`） |
| zlib 压缩有大小阈值 | **无条件压缩**。`dict_sdi_set()` 里直接 `compressor.compress()`，没有任何 `if (sdi_len > N)` |
| `FIL_PAGE_SDI=17853` 在 `fil0types.h` | 在 **`fil0fil.h`**；`fil0types.h` 只有页内偏移（`FIL_PAGE_DATA=38` 等） |
| page 0 的 SDI 偏移是常量 10505 | 是**算出来的**：`XDES_ARR_OFFSET(150) + XDES_SIZE(40)×xdes_arr_size(256) + Encryption::INFO_MAX_SIZE(115)`，**随页大小变化**（`fsp_header_get_sdi_offset()`） |
| `SDI_VERSION` 只有一个 | **两个同名不同义**：`dd::SDI_VERSION = 80019`（JSON 里）、`dict0dict.h::SDI_VERSION = 1`（page 0 页头） |
| `sdi::remove_all_sdi()` | **不存在**（全树 grep 0 命中）。是 `sdi::drop_all_for_table` / `drop_all_for_part` → `sdi_tablespace::drop_all_sdi` |
| handlerton 有 `sdi_get_filename` | **不存在**。只有 6 个 SDI 回调；文件名由 SQL 层 `sdi_file::sdi_filename()` 生成 |
| 按 `HTON_SUPPORTS_*` 标志分派 | **没有任何 SDI 能力标志**。分派全靠函数指针 `sdi_set` / `sdi_delete` |
| `sql/dd/impl/storage_adapter.cc` | 实际路径含 `cache/`：`sql/dd/impl/cache/storage_adapter.cc` |
| `DATA_SDI_*` 记录偏移宏 | **不存在**。记录布局由 `dict_sdi_create_idx_in_mem()` 的 5 列定义决定；`ibd2sdi` 自己有一套 `REC_OFF_DATA_*` |
| 日常删 SDI 走 `sdi_drop` | 走 **`sdi_delete`**（按 key 删一条）；`sdi_drop` 是"删整个 SDI 索引"，只在 SDI 损坏时用，且 InnoDB 侧是 `#if 0` |

### ★ 机制纠错（三条，均已在上文就近更正）

1. **`ALTER TABLE ... IMPORT TABLESPACE FROM .sdi` 这条语法不存在**——逻辑重建是独立的 `IMPORT TABLE FROM`。两者在语法层面就是两条路。
2. **"大表定义才压缩"是错的**——无条件 zlib 压缩（level 6）；"三层机制"的第三层其实是 InnoDB 通用 BLOB off-page，SDI 代码里没有专门分支。
3. **"只写不读"要限定**——`sdi_get` 与 `sdi_drop` 是死的（`#if 0`），但 `sdi_set` / `sdi_delete` / `sdi_get_keys` / `sdi_create` 是活的，而且这 6 个回调**全都挂在 handlerton 上**。

### 易混淆点

| 混淆 | 澄清 |
|---|---|
| `ibd2sdi` 输出的多行 JSON 就是 SDI 原样？ | ❌ SDI 是**紧凑单行**；多行是 `ibd2sdi` 再过一遍 `PrettyWriter`（**`--pretty` 默认开**，`--skip-pretty` 才是原样）。且外层 `["ibd2sdi", {...}]` 信封是 `fprintf` **手写**的，第 0 个元素是字符串 `"ibd2sdi"` ⇒ `jq '.[1]'` 下标从 1 起 |
| 所有 DD 对象都能回退写 `.sdi` 文件？ | ❌ **只有 `dd::Table` 有文件回退**；`store(Tablespace)` 在 `sdi_set == nullptr` 时**直接 return false**（见 13.） |
| 想"既不写文件也不写表空间"就留回调为 `nullptr`？ | ❌ 留 `nullptr` ⇒ **回退写 `.sdi` 文件**。要"什么都不写"得把 `sdi_set` 实现成 **no-op**（NDB 与 PFS 都这么干，见 16.） |
| SDI 里能拿到对象 id？ | ❌ **不写 id**，只写 name ⇒ IMPORT 必然重新分配 id |
| SDI 可跨实例移植？ | ⚠️ **部分不成立**——`collation_id` 是裸 id，DD 内部自增主键跨实例不保证一致 |
| 跨 minor 版本能导入？ | ❌ `sdi_version` 必须**全等** |
| 服务器会读表空间里的 SDI？ | ❌ `dict_sdi_get` / `_drop` 被 `#if 0`，debug 下 `ut_error`（见 §10） |
| `.sdi` 文件是 `innodb_read_only` 时才写？ | ❌ 按 **`hton->sdi_set == nullptr`（SE 能力）**分派，与该变量无关（见 §5） |
| SDI 是独立数据源？ | ❌ 是 DD 的**冗余副本**，一致性由"DD 写 + SDI 写同一事务"保证（见 §4） |
| SDI 有大小上限？ | 源码**无显式常量**；理论上限来自 4 字节长度字段；压缩失败是 `ib::fatal`（崩实例） |

---

## 参考

- MySQL 8.0 Reference Manual → *Data Dictionary* / *Serialized Dictionary Information (SDI)* / *`ibd2sdi`*
- MySQL 8.0 Source Code：
  - server 侧 —— `sql/dd/impl/sdi.cc`（serialize / deserialize / 两个 context）、`sql/dd/impl/sdi_api.cc`、`sql/dd/impl/sdi_file.cc`（`.sdi` 文件载体）、`sql/dd/impl/sdi_tablespace.cc`
  - InnoDB 侧 —— `storage/innobase/dict/dict0sdi.cc`（B-tree 存储、根页号）、`storage/innobase/ibd2sdi/ibd2sdi.cc`（离线解析）、`storage/innobase/row/row0import.cc`（传统 IMPORT 只搬运不解析）
  - 各 DD 对象的 `serialize()` —— `sql/dd/impl/types/*_impl.cc`
- [`../server/dd/dd.md`](../server/dd/dd.md)（DD 表体系 / 对象模型 / 缓存 / 自举 / 升级 / 建表往返 §2.6）
- [`i_s.md`](../infra/observability/i_s.md)（同为跨层的 server ↔ 引擎机制，可作组织方式对照）
- [`../innodb/io.md`](../innodb/io.md)（表空间与文件 IO，SDI 索引的宿主）


# 索引的元数据：DD ↔ `dict_index_t` ↔ SDI

> 基于 MySQL 8.0.39 源码。本篇讲 **"索引作为一个对象"是怎么被表示和存储的**——server 层数据字典（`dd::Index`）、引擎侧内存对象（`dict_index_t`）、两者之间的桥（`se_private_data`）、以及 SDI 里的副本。
>
> **边界**：索引的**类型语义**（用户可见的主键/二级/唯一/降序/前缀/功能/多值/全文/空间/不可见）见 [`types.md`](types.md)；索引的**物理结构**见 [`physical_storage.md`](physical_storage.md)、[`btr.md`](btr.md)；DD 系统的整体设计见 [`../server/dd/dd.md`](../server/dd/dd.md)、引擎侧字典见 [`../server/dd/innodb_dict.md`](../server/dd/innodb_dict.md)——本篇只取"索引对象"这一视角。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [两套字典的分工与桥：`se_private_data`](#两套字典的分工与桥se_private_data)
    - [`dict_index_t`：完整对象模型](#dict_index_t完整对象模型)
    - [`n_uniq` 的三种情况与 `n_unique_in_tree`](#n_uniq-的三种情况与-n_unique_in_tree)
    - [索引名规范与 ID 分配](#索引名规范与-id-分配)
  - 加载路径
    - [开表：从 DD 一次性构造全部索引](#开表从-dd-一次性构造全部索引)
    - [缓存与淘汰：没有索引级 LRU](#缓存与淘汰没有索引级-lru)
  - 写路径（DDL 时序）
    - [建索引的三步：拿 ID → 建树 → 回填 DD](#建索引的三步拿-id--建树--回填-dd)
    - [SDI 里的索引副本](#sdi-里的索引副本)
  - 特殊索引的元数据
    - [功能/多值、全文、空间、不可见索引](#功能多值全文空间不可见索引)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引的元数据在 8.0 里有**两份表示**：

| | 位置 | 性质 |
|---|---|---|
| `dd::Index` + `dd::Index_element` | `mysql.indexes` / `mysql.index_column_usage` / `mysql.index_partitions` 表 | **权威持久化**（DD 表，InnoDB 存储） |
| `dict_index_t` + `dict_field_t` | InnoDB 内存（`dict_sys`） | **开表时从 DD 重建的缓存**，不做独立持久化 |

两者通过 `dd::Index::se_private_data()` 里的五个键桥接：`id`、`space_id`、`table_id`、`root`、`trx_id`。

### 用途

- 开表时重建索引对象（拿到 root page no 才能定位 B 树）；
- DDL 时同步两侧（原子 DDL 的时序核心）；
- SDI 让 `.ibd` 自带元数据副本（崩溃/导入时重建 DD）。

### 版本演进

- **5.7 及以前**：InnoDB 自己维护 `SYS_INDEXES` / `SYS_FIELDS` 系统表（`DICT_HDR`），server 层另有一份 `.frm`——**双写双源**；
- **8.0**：DD 统一为权威源，InnoDB 的 `SYS_INDEXES` 只在**升级路径**读（`dict_load_index_low` 的 `srv_is_upgrade_mode` 分支），正常启动走 `se_private_data`。

---

## 理论基础

### 设计思想与权衡

**1. 单向投影，不做双向同步。** DD 是唯一真相；`dict_index_t` 是投影。**没有**"dict 改了写回 DD"的通用机制——只有 DDL 时显式调 `dd_write_index()` 回填。

**2. 物理信息藏在 `se_private_data`。** DD 是**通用**字典（要服务所有引擎），不能为 InnoDB 加 root page 列。解法是给每个 `dd::Index` 挂一个 MEDIUMTEXT 的 `se_private_data` 属性包，引擎自己往里塞键值——这是"通用 schema + SE 私有扩展"的经典模式。

**3. 加载粒度的取舍：一次性全量。** 开表时把表上**所有**索引一次性构造完。代价是开一个有很多索引的表较慢；收益是运行期不需要"索引级 miss 回填"的复杂逻辑（对比：列的 instant 元数据才有延迟语义）。

**4. 元数据与物理的分离点：root page no。** DD 只存一个页号，树的实际内容全在页里。这带来一个后果——**root page no 在索引生命周期内恒定**（见 [`physical_storage.md`](physical_storage.md)），否则每次树长高都要改 DD。

### ★ 为什么必须有"两份"：三个约束

与 [`construction.md`](construction.md) 讲的"SQL 语义 vs 引擎存储四处不对等"是**同一个约束的另一面**——那里是 SQL 层与引擎层的落差，这里是**通用字典与引擎私有**的落差：

| # | 约束 | 说明 |
|---|---|---|
| 1 | **DD 必须引擎无关** | `dd::Index` 要服务所有存储引擎，不能为 InnoDB 加 `root_page_no` / `space_id` 这类专属列。解法只能是属性包（`se_private_data`）——这是"通用 schema + SE 私有扩展"的经典模式，代价是引擎私有信息变成**无类型的字符串键值** |
| 2 | **运行期需要可直接使用的内存结构** | `dict_index_t` 里除了元数据还挂着锁、统计（`stat_n_diff_key_vals`）、在线 DDL 状态、页面缓存引用——这些是**运行态**，从 DD 对象（需解析 MEDIUMTEXT、查属性表）每次现取是不可能的。所以必须有一份内存投影 |
| 3 | **两者生命周期不同** | DD 是事务性持久化（原子 DDL），`dict_index_t` 是内存缓存（开表构建、关表释放，可随时丢弃重建） |

**为什么不能只留一份**：

- 只留 DD：运行期每访问一次索引都要解析 DD 对象与属性包——性能上不可接受，且运行态（锁、统计）无处安放；
- 只留 `dict_index_t`：没有持久化（崩溃丢失），且引擎私有内存格式无法被通用工具消费——**升级、SDI 导出、崩溃恢复时的 DD 重建都依赖那份通用表示**。

★ 所以"两份"不是冗余，而是"**一份管持久化与跨引擎通用性，一份管运行期效率与运行态**"的分工。代价就是本篇大量篇幅在讲的**映射与同步**（`se_private_data` 五键、加载路径、DDL 时序）。

### ★ 为什么 root page no 必须恒定（深化）

DD 里存的不是"树的形状"而是一个页号。这意味着：**索引树的 root 页一旦分配就不再变**（长高不换根，见 [`physical_storage.md`](physical_storage.md)）。

为什么必须这样：若树长高/重组就要改 root page no，则每次 SMO 都要**改 DD**——而 DD 的修改是**事务性 DDL 操作**（要走 DD 事务、写 redo、可能触发原子 DDL 的整套流程），把高频的物理操作升级成高频的元数据事务，代价不可接受。所以"root 恒定"是"把元数据变更从热路径上摘掉"的必然设计。

### ★ 为什么还要 SDI 副本（元数据的第三份）

**先说一个反直觉的事实**：SDI **不在** mysql.ibd 里，而是**每个表空间一份**。源码证据是 `fsp_sdi_write_root_to_page()` 写在**该表空间 page 0**（`fsp0fsp.cc:3938`）：

```cpp
void fsp_sdi_write_root_to_page(page_t *page, const page_size_t &page_size,
                                page_no_t root_page_num, mtr_t *mtr) {
  ut_ad(page_get_page_no(page) == 0);        // ★★ 就是本表空间的第 0 页

  ulint sdi_offset = fsp_header_get_sdi_offset(page_size);
  mlog_write_ulint(page + sdi_offset, SDI_VERSION, MLOG_4BYTES, mtr);
  mlog_write_ulint(page + sdi_offset + 4, root_page_num, MLOG_4BYTES, mtr);
}
```

**三份元数据的分工**（源码归纳）：

| 份 | 载体 | 谁读 | 什么时候必须 |
|---|---|---|---|
| ① DD（`mysql.*` InnoDB 表） | `mysql.indexes` / `mysql.index_column_usage` 行的 `se_private_data` | 正常 SQL 路径 | 每次开表 |
| ② InnoDB 内存字典（`dict_index_t`） | 内存 + 物理页（`PAGE_INDEX_ID` / root page） | InnoDB 内部 | 每次 DML |
| ③ SDI（每表空间一个 B-tree） | zlib 压缩的 JSON，页类型 `FIL_PAGE_SDI` | `ibd2sdi` / IMPORT / 升级 / 崩溃后裸表 | **DD 不可用**时 |

> ★★ 关键区别：**DD 是"SQL 引擎当前相信的事实"，SDI 是"这个 .ibd 文件自己声明的事实"**。前者可能被误删/损坏/版本不匹配，后者随数据文件走。所以：
> - `ibd2sdi file.ibd` 不需要 mysqld，不需要 DD —— 直接读 page 0 找 root，扫 SDI B-tree
> - `IMPORT TABLESPACE` / `DISCARD` 之后重挂，靠 SDI 重建
> - 崩溃后 `dict_table_open_on_id()` 走到 `dict_table_is_sdi(table_id)` 分支时，SDI 表**纯内存构造**（`dict_sdi_create_idx_in_mem`），不查 DD

**为什么 SDI 必须压缩**（`dict0sdi.h:40`）：SDI 是完整表定义的 JSON（含所有列、索引、分区、字符集、注释），单表可达几十 KB。不压缩会让每个 .ibd 头部膨胀出可观空间，且 BLOB 列会溢出页。`Sdi_Compressor` 固定 `compress2(..., 6)`，压缩后长度与原长都存进 SDI 表的 `compressed_len` / `uncompressed_len` 两列——**因为解压前必须知道要分配多大的目标缓冲**。

**为什么 SDI 索引禁用 AHI**（`dict0crea.cc:721`）：

```cpp
  /* Disable AHI on SDI tables */
  temp_index->disable_ahi = true;
```

> ★ 源码注释原文：*"Currently limited to intrinsic temporary table and SDI table as **index id is not unique** for such table which is one of the validation criterion for ahi."* —— 所有 SDI 索引共用 `dict_sdi_get_index_id() == UINT64_MAX`，AHI 用 `(space_id, index_id)` 做键会冲突。

**为什么 SDI 表的 table_id 用 `0xFFFFFFFF00000000 | space_id`**（`dict0dict.ic:1330`）：把 SDI 表强制挤到 table_id 空间的**最高段**，从而 `dict_table_is_sdi(table_id)` 只需一次 `>=` 比较就能判定，且永远不会与真实用户表 ID 冲突（真实 table_id 从 `DICT_MAX_DD_TABLES = 1024` 起，且 `dict_hdr_get_new_id()` 在 `id >= dict_sdi_get_table_id(0)` 时直接 `ib::fatal "running out of table_ids"`）。

**SDI 的 key 只有两个 type**（`sql/handler.h:119`）：`SDI_TYPE_TABLE = 1`（表定义）、`SDI_TYPE_TABLESPACE = 2`（表空间定义）。一个 .ibd 里可以有**多个** table SDI（分区表：每个分区一份）+ 一份 tablespace SDI。`dict_sdi_get_keys()` 就是全表扫描 SDI 聚簇索引列出所有 key。

`.ibd` 里还存了一份 SDI（Serialized Dictionary Information）。为什么 DD 已经有了还要副本：

- 约束：**`.ibd` 文件会被搬走**（导入到另一个实例、恢复到别的机器）。搬走时目标实例的 DD 里没有这张表的元数据——SDI 让文件**自描述**，导入时可从文件重建 DD 条目；
- 约束：崩溃恢复场景下，若 `mysql.ibd`（DD 所在）本身损坏，SDI 是最后的元数据来源。

代价：SDI 占用表空间页（每表一份），且是**快照**（可能与 DD 不一致，需 `IMPORT TABLESPACE` 时校验）。

---

## 核心实现

### 主线与基础构件

#### 两套字典的分工与桥：`se_private_data`

⚠️ **勘误**：root page no 的键名是 **`"root"`**，不是 `"page"` / `"root_page"`；索引的 **`space_id` 只写不读**（读的是 `dd::Tablespace::se_private_data()["id"]`）。

```cpp
// include/dict0dd.h:246 —— InnoDB 私有 key 的枚举与字符串数组
enum dd_index_keys {
  /** Index identifier */
  DD_INDEX_ID,
  /** Space id */
  DD_INDEX_SPACE_ID,
  /** Table id */
  DD_TABLE_ID,
  /** Root page number */
  DD_INDEX_ROOT,
  /** Creating transaction ID */
  DD_INDEX_TRX_ID,
  /** Sentinel */
  DD_INDEX__LAST
};

const char *const dd_index_key_strings[DD_INDEX__LAST] = {
    "id", "space_id", "table_id", "root", "trx_id"};
```

> ★ 这就是 `mysql.indexes` 表 `se_private_data` 列里那串 JSON 的全部键。可以 `SELECT name, se_private_data FROM mysql.indexes WHERE table_id=...` 直接看到。

**另外三组 key**（`dict0dd.h:216-244`）：

```cpp
const char *const dd_space_key_strings[DD_SPACE__LAST] = {
    "flags", "id", "discard", "server_version", "space_version", "state"};
    //                              ↑ 表空间的 "id" 才是 dict_index_t::space 的真实来源

const char *const dd_table_key_strings[DD_TABLE__LAST] = {
    "autoinc", "data_directory", "version", "discard", "instant_col"};

const char *const dd_column_key_strings[DD_COLUMN__LAST] = {
    "default", "default_null", "version_added", "version_dropped", "physical_pos"};

const char *const dd_partition_key_strings[DD_PARTITION__LAST] = {
    "format", "instant_col", "discard"};
```

**InnoDB → DD 的唯一回填点**（`dd_write_index()`，`dict0dd.cc:2615`）：

```cpp
template <typename Index>
static void dd_write_index(dd::Object_id dd_space_id, Index *dd_index,
                           const dict_index_t *index) {
  ut_ad(index->id != 0);
  ut_ad(index->page >= FSP_FIRST_INODE_PAGE_NO);

  dd_index->set_tablespace_id(dd_space_id);     // ★ dd::Tablespace 的 Object_id，不是 InnoDB space_id

  dd::Properties &p = dd_index->se_private_data();
  p.set(dd_index_key_strings[DD_INDEX_ID], index->id);            // "id"
  p.set(dd_index_key_strings[DD_INDEX_SPACE_ID], index->space);   // "space_id"
  p.set(dd_index_key_strings[DD_TABLE_ID], index->table->id);     // "table_id"
  p.set(dd_index_key_strings[DD_INDEX_ROOT], index->page);        // ★★ "root" —— 两份字典的接缝
  p.set(dd_index_key_strings[DD_INDEX_TRX_ID], index->trx_id);    // "trx_id"
}
```

**反向对账**（`dd_index_match()`，`dict0dd.cc:193`）——缓存命中时也要逐个索引比对：

```cpp
static bool dd_index_match(const dict_index_t *index, const Index *dd_index) {
  bool match = true;

  /* Don't check the name for primary index, since internal index
  name could be variant */
  if (my_strcasecmp(system_charset_info, index->name(),
                    dd_index->name().c_str()) != 0 &&
      strcmp(dd_index->name().c_str(), "PRIMARY") != 0) {
    ib::warn(ER_IB_MSG_162) << "Index name in InnoDB is " << index->name()
                            << " while index name in global DD is "
                            << dd_index->name();
    match = false;
  }

  const dd::Properties &p = dd_index->se_private_data();
  p.get(dd_index_key_strings[DD_INDEX_ID], &id);
  if (id != index->id) { ...  match = false; }          // ★ id 不符

  p.get(dd_index_key_strings[DD_INDEX_ROOT], &root);
  if (root != index->page) { ...  match = false; }      // ★★ root 不符 —— 最常见的"表打不开"原因

  p.get(dd_index_key_strings[DD_INDEX_TRX_ID], &trx_id);
  if (trx_id != 0 && index->trx_id != 0 && trx_id != index->trx_id) {
    ... match = false;
  }
  return match;
}
```

**按名字反查，不依赖顺序**（`dd_find_index()`，`dict0dd.cc:2872`）：

```cpp
template <typename Index>
const dict_index_t *dd_find_index(const dict_table_t *table, Index *dd_index) {
  /* If the name is PRIMARY, return the first index directly,
  because the internal index name could be 'GEN_CLUST_INDEX'. */
  if (dd_index->name() == "PRIMARY") {
    return table->first_index();
  }

  /* The order could be different because all unique dd::Index(es)
  would be in front of other indexes. */
  //  ★★ DD 里 UNIQUE 排前面，InnoDB 链表按 key_info 顺序 ⇒ 必须按名字找，不能按序号
  const dict_index_t *index;
  for (index = table->first_index();
       (index != nullptr &&
        (dd_index->name() != index->name() || !index->is_committed()));
       index = index->next()) {
  }
  ut_ad(index != nullptr);
  return index;
}
```

**DD 侧对象**（`sql/dd/types/index.h:52` / `index_element.h:45`）——关键成员：

| `dd::Index` 成员 | 对应 InnoDB |
|---|---|
| `se_private_data()` | `id` / `space_id` / `table_id` / `root` / `trx_id` |
| `name()` | `dict_index_t::name`（PRIMARY 除外，InnoDB 内部可能是 `GEN_CLUST_INDEX`） |
| `type()`（`IT_PRIMARY/IT_UNIQUE/IT_MULTIPLE/IT_FULLTEXT/IT_SPATIAL`） | **不直接对应**；InnoDB 的 `type` 位由 `KEY::flags` 推导 |
| `algorithm()`（`IA_SE_SPECIFIC/IA_BTREE/IA_RTREE/IA_HASH/IA_FULLTEXT`） | **无对应**，InnoDB 只有 BTree/RTree/FTS，由 `type` 位隐含 |
| `is_hidden()` | `dict_index_t::hidden`（FTS_DOC_ID_INDEX） |
| `is_visible()` | `KEY::is_visible`（不可见索引；**InnoDB dict 侧无此标志**） |
| `comment()` | `MERGE_THRESHOLD=` 藏在这里 → `dict_index_t::merge_threshold` |
| `tablespace_id()` | 经 `dd::Tablespace::se_private_data()["id"]` → `dict_index_t::space` |
| `elements()` | `dict_field_t[]` |
| `options()` / `engine_attribute()` | 开表时不用（只用于 EXCHANGE PARTITION 互换） |

```cpp
// sql/dd/types/index_element.h:45
class Index_element : virtual public Weak_object {
 public:
  typedef tables::Index_column_usage DD_table;   // ★ 落库表：mysql.index_column_usage
  enum enum_index_element_order { ORDER_UNDEF = 1, ORDER_ASC, ORDER_DESC };

  virtual const Column &column() const = 0;      // ★ SRID / is_se_hidden 从这里取
  virtual uint ordinal_position() const = 0;     // key part 序号（1-based）
  virtual uint length() const = 0;               // ★ 前缀长度；NULL 表示整列
  virtual bool is_length_null() const = 0;
  virtual enum_index_element_order order() const = 0;   // ★ ASC/DESC ↔ dict_field_t::is_ascending
  virtual bool is_hidden() const = 0;            // ★ 隐藏 key part（PK 尾随列）
  virtual bool is_prefix() const = 0;
};
```

> ⚠️ **索引类型换算的真名在 SQL 层，不在 InnoDB**：`dd_get_new_index_type()`（`sql/dd/dd_table.cc:809`）、`dd_get_new_index_algorithm_type()`（`dd_table.cc:782`）、反向 `dd_get_old_index_algorithm_type()`（`sql/dd_table_share.cc:201`）。**不存在 `innobase_get_index_type()` / `get_dd_index_type()`**。
>
> ★ 两套 ID 别混：`dict_sys_t::s_dict_space_id = 0xFFFFFFFE`（InnoDB space_id）vs `dict_sys_t::s_dd_dict_space_id = 1`（`dd::Tablespace` 的 Object_id）。`dd_open_table_one()` 里两者都要用。

```cpp
// storage/innobase/include/dict0dd.h:246-265
enum dd_index_keys {
  DD_INDEX_ID,        /** Index identifier */
  DD_INDEX_SPACE_ID,  /** Space id */
  DD_TABLE_ID,        /** Table id */
  DD_INDEX_ROOT,      /** Root page number */
  DD_INDEX_TRX_ID,    /** Creating transaction ID */
  DD_INDEX__LAST
};
const char *const dd_index_key_strings[DD_INDEX__LAST] =
    {"id", "space_id", "table_id", "root", "trx_id"};
```

写入（`dd_write_index`，`dict0dd.cc:2619-2633`）：

```cpp
static void dd_write_index(dd::Object_id dd_space_id, Index *dd_index,
                           const dict_index_t *index) {
  ut_ad(index->id != 0);
  ut_ad(index->page >= FSP_FIRST_INODE_PAGE_NO);
  dd_index->set_tablespace_id(dd_space_id);
  dd::Properties &p = dd_index->se_private_data();
  p.set(dd_index_key_strings[DD_INDEX_ID], index->id);
  p.set(dd_index_key_strings[DD_INDEX_SPACE_ID], index->space);
  p.set(dd_index_key_strings[DD_TABLE_ID], index->table->id);
  p.set(dd_index_key_strings[DD_INDEX_ROOT], index->page);
  p.set(dd_index_key_strings[DD_INDEX_TRX_ID], index->trx_id);
}
```

**server 侧 `dd::Index` 的属性全表**（`sql/dd/types/index.h:81-223`）：`is_generated`（GIPK 等隐式生成）、`is_hidden`（SE 内部索引：`FTS_DOC_ID_INDEX`、`DB_ROW_ID` 主键）、`type`（`IT_PRIMARY`/`IT_UNIQUE`/`IT_MULTIPLE`/`IT_FULLTEXT`/`IT_SPATIAL`）、`algorithm`（`IA_SE_SPECIFIC`/`IA_BTREE`/`IA_RTREE`/`IA_HASH`/`IA_FULLTEXT`）、`is_visible`、`ordinal_position`（1-based，由 `Collection::push_back` 统一赋值）、`options`（`flags`/`block_size`/`parser_name`）、`engine_attribute`。

**`dd::Index_element`**（`sql/dd/types/index_element.h:45-135`）四个关键成员：

| 成员 | 语义 |
|---|---|
| `length` | 前缀**字节数**；整列索引也写实际长度（**不是 0**）；"未设置"用 `(uint)-1` 表示 |
| `order` | `ORDER_UNDEF/ORDER_ASC/ORDER_DESC`——降序索引即 `ORDER_DESC` |
| `is_hidden` | **只有一种情况为 true**：InnoDB 把 PK 列追加进二级索引时，追加的元素被标 hidden |
| `ordinal_position` | 键内序号（1-based） |

四种 DD 表：`mysql.indexes`（`UNIQUE KEY(table_id, name)`，name 列 collation 为 `utf8mb3_tolower_ci` → **同表内索引名大小写不敏感唯一**）、`mysql.index_column_usage`（`length` 可空、`order`、`hidden`）、`mysql.index_partitions`（分区表每索引每分区一行）、`mysql.index_stats`（DD 的"动态统计"，**不是** InnoDB 持久统计的落点——后者写 `mysql.innodb_index_stats`）。

#### `dict_index_t`：完整对象模型

⚠️ **勘误**：
- **`fields` 是裸数组，没有 `m_fields`**（`m_fields` 属于 `ddl::Index_def`）
- **`magic_n` 仅在 `UNIV_DEBUG` 下存在**
- **`is_primary()` / `is_unique()` / `has_virtual()` / `get_n_unique_in_tree()` 都不是成员函数**——是自由函数 `dict_index_is_auto_gen_clust()` / `dict_index_is_unique()` / `dict_index_has_virtual()` / `dict_index_get_n_unique_in_tree()`
- **不存在 `n_uniq_non_null` / `n_core_fields` / `n_definition_fields`**（对应的是 `nulls_equal`、`n_instant_nullable`、`n_user_defined_cols`）

**`dict_field_t`**（`dict0mem.h:912`）——只有 4 个数据成员：

```cpp
struct dict_field_t {
  dict_field_t() : col(nullptr), prefix_len(0), fixed_len(0), is_ascending(0) {}

  dict_col_t *col;           /*!< pointer to the table column */
  id_name_t name;            /*!< name of the column */
  unsigned prefix_len : 12;  /*!< 0 or the length of the column prefix in bytes
                             ... NOTE that in the UTF-8 charset, MySQL sets
                             this to (mbmaxlen * the prefix len) in UTF-8 chars */
  unsigned fixed_len : 10;   /*!< 0 or the fixed length of the column if
                             smaller than DICT_ANTELOPE_MAX_INDEX_COL_LEN */
  unsigned is_ascending : 1; /*!< 0=DESC, 1=ASC */
};
```

> ★ `prefix_len` 12 位上限受 `MAX_KEY_LENGTH_BITS = 12` 约束；`is_ascending` 就 1 位——**降序信息在这里，不在 `index->type` 里**。

**`dict_index_t` 头部与位域区**（`dict0mem.h:1062`）：

```cpp
struct dict_index_t {
  space_index_t id;          // ★ 类型 space_index_t = ib_id_t，"表空间内唯一"
  mem_heap_t *heap;
  id_name_t name;
  const char *table_name;
  dict_table_t *table;
  unsigned space : 32;
  unsigned page : 32;        // ★★ root page no —— 两份字典的耦合点
  unsigned merge_threshold : 6;
  unsigned type : DICT_IT_BITS;          // DICT_CLUSTERED/DICT_UNIQUE/DICT_IBUF/...
  unsigned trx_id_offset : MAX_KEY_LENGTH_BITS;
  unsigned n_user_defined_cols : 10;
  unsigned allow_duplicates : 1;
  unsigned nulls_equal : 1;              // ★ SQL NULL == SQL NULL
  unsigned disable_ahi : 1;              // ★ 目前只给 intrinsic 临时表与 SDI 表
  unsigned n_uniq : 10;                  // ★★ 计算出来的（见下节）
  unsigned n_def : 10;                   // 已定义的字段数
  unsigned n_fields : 10;                // 内部表示的字段总数（含系统列）
  unsigned n_total_fields : 10;          // 含 INSTANT DROP 列
  unsigned n_nullable : 10;
  unsigned n_instant_nullable : 10;
  unsigned cached : 1;                   // ★ 是否已在字典缓存中
  unsigned to_be_dropped : 1;
  unsigned online_status : 2;
  unsigned uncommitted : 1;
  unsigned instant_cols : 1;
  unsigned row_versions : 1;

  uint32_t srid;                         // ★ 空间索引
  bool srid_is_valid;
  std::unique_ptr<dd::Spatial_reference_system> rtr_srs;
#ifdef UNIV_DEBUG
  uint32_t magic_n;                      // ★ = DICT_INDEX_MAGIC_N = 76789786
#endif
  dict_field_t *fields;                  // ★★ 裸数组，不是 m_fields
  std::vector<uint16_t> fields_array;    // 仅 INSTANT 场景：按物理位置排序
  uint32_t nullables[MAX_ROW_VERSION + 1] = {0};
  st_mysql_ftparser *parser;             // FTS
  bool is_ngram;
  bool has_new_v_col;
  bool hidden;
```

**链表节点 / 统计信息**（`dict0mem.h:1202`）：

```cpp
  UT_LIST_NODE_T(dict_index_t) indexes;   // ★ 挂在 dict_table_t::indexes 双向链表上
  btr_search_t *search_info;
  row_log_t *online_log;                  // ★ 仅 online_status == ONLINE_INDEX_CREATION 时有效

  /*----------------------*/
  /** Statistics for query optimization */
  uint64_t *stat_n_diff_key_vals;         // 长度 = n_uniq
  uint64_t *stat_n_sample_sizes;          // 长度 = n_uniq
  uint64_t *stat_n_non_null_key_vals;     // 长度 = n_uniq（innodb_stats_method=nulls_ignored）
  ulint stat_index_size;
  ulint stat_n_leaf_pages;
  /** @} */

  last_ops_cur_t *last_ins_cur;
  last_ops_cur_t *last_sel_cur;
  rec_cache_t rec_cache;
  rtr_ssn_t rtr_ssn;                      // R-Tree node sequence number
  rtr_info_track_t *rtr_track;
  trx_id_t trx_id;                        // ★ 创建该索引的事务（会写进 se_private_data["trx_id"]）
  zip_pad_info_t zip_pad;
  rw_lock_t lock;                         // ★ 保护索引树上层
  bool fill_dd;
```

> ★ 三个 `uint64_t*` 统计数组长度都是 `n_uniq`，在 `dict0dict.cc:2541-2551` 用 `mem_heap_zalloc` 分配；`stat_index_size` / `stat_n_leaf_pages` 初值均为 1。

**`DICT_*` 类型位标志（完整）**（`dict0mem.h:91`）：

```cpp
/** Type flags of an index: OR'ing of the flags is allowed to define a
combination of types */
constexpr uint32_t DICT_CLUSTERED = 1;
constexpr uint32_t DICT_UNIQUE = 2;
constexpr uint32_t DICT_IBUF = 8;          // ★ 缺 4，历史遗留
constexpr uint32_t DICT_CORRUPT = 16;
constexpr uint32_t DICT_FTS = 32;          // can't be combined with the other flags
constexpr uint32_t DICT_SPATIAL = 64;      // can't be combined with the other flags
constexpr uint32_t DICT_VIRTUAL = 128;
constexpr uint32_t DICT_SDI = 256;         // 只在内存索引结构上
constexpr uint32_t DICT_MULTI_VALUE = 512;
constexpr uint32_t DICT_IT_BITS = 10;      // ★ 位域宽度
```

> ★★ `DICT_CLUSTERED` **单独出现**（`type == 1`）= 自动生成的 `GEN_CLUST_INDEX`（无主键）；显式主键是 `DICT_CLUSTERED | DICT_UNIQUE = 3`。这正是 `dict_index_is_auto_gen_clust()` 用 `==` 而不是 `&` 的原因。

**创建与填充**（⚠️ **不存在 `dict_index_create()`**）：

```cpp
// dict/mem.cc:277
dict_index_t *dict_mem_index_create(const char *table_name, const char *index_name,
                                    ulint space, ulint type, ulint n_fields) {
  heap = mem_heap_create(DICT_HEAP_SIZE, UT_LOCATION_HERE);
  index = static_cast<dict_index_t *>(mem_heap_zalloc(heap, sizeof(*index)));
  //  ★★ 整个 dict_index_t 由 mem_heap_zalloc 零初始化

  new (&index->fields_array)(decltype(index->fields_array))();
  //  ★ 手工 placement-new fields_array（std::vector）—— 因为对象从没调过构造函数
  dict_mem_fill_index_struct(index, heap, table_name, index_name, space, type, n_fields);
  ...
  if (type & DICT_SPATIAL) { mutex_create(LATCH_ID_RTR_SSN_MUTEX, &index->rtr_ssn.mutex); ... }
  return (index);
}
```

```cpp
// include/dict0mem.ic:38
static inline void dict_mem_fill_index_struct(...) {
  if (heap) {
    index->heap = heap;
    index->name = mem_heap_strdup(heap, index_name);
    index->fields = (dict_field_t *)mem_heap_alloc(
        heap, 1 + n_fields * sizeof(dict_field_t));
    //                  ★ 多分配一个槽位："The '1 +' prevents allocation of an empty mem block"
  } else { ... }
  index->type = type;
  index->space = (unsigned int)space;
  index->page = FIL_NULL;                                    // ★ 初始未分配
  index->merge_threshold = DICT_INDEX_MERGE_THRESHOLD_DEFAULT;  // = 50
  index->table_name = table_name;
  index->n_fields = (unsigned int)n_fields;
  index->n_total_fields = (unsigned int)n_fields;
  index->allow_duplicates = false;
  index->nulls_equal = false;
  index->disable_ahi = false;
  ...
}
```

> ⚠️ **这里不设置 `id`**——`id` 由调用方从 DD / dict_hdr 读取后赋值。

**`dict_index_add_col()`（`dict/dict.cc:44`）——只做 4 件事，`n_uniq` 完全不在这里决定**：

```cpp
void dict_index_add_col(dict_index_t *index, const dict_table_t *table,
                        dict_col_t *col, ulint prefix_len, bool is_ascending) {
  ...  // 虚拟列：登记到 v_col->v_indexes
  index->add_field(col_name, prefix_len, is_ascending);   // n_def++ + 写 name/prefix_len/is_ascending
  field = index->get_field(index->n_def - 1);
  field->col = col;                                        // ★ 回填 col 指针

  /* DATA_POINT is a special type ... */
  if (dict_index_is_spatial(index) && DATA_POINT_MTYPE(col->mtype) && index->n_def == 1) {
    field->fixed_len = DATA_MBR_LEN;
  } else {
    field->fixed_len = static_cast<unsigned int>(col->get_fixed_size(dict_table_is_comp(table)));
  }
  if (prefix_len && field->fixed_len > prefix_len) field->fixed_len = (unsigned int)prefix_len;

  /* Long fixed-length fields that need external storage are treated as
  variable-length fields, so that the extern flag can be embedded in the
  length word. */
  if (field->fixed_len > DICT_MAX_FIXED_COL_LEN) field->fixed_len = 0;
  static_assert(DICT_MAX_FIXED_COL_LEN == 768, "DICT_MAX_FIXED_COL_LEN != 768");

  /* Skip INSTANT DROP column */
  if (col->is_nullable() && !col->is_instant_dropped()) index->n_nullable++;
}
```

```cpp
// dict0mem.h:1496 —— add_field 只 3 行
  void add_field(const char *name_arg, ulint prefix_len, bool is_ascending) {
    n_def++;
    field = get_field(n_def - 1);
    field->name = name_arg;
    field->prefix_len = (unsigned int)prefix_len;
    field->is_ascending = is_ascending;
  }
```

> ⚠️ **不存在 `dict_index_add_field()`**（那是 `ddl::Index_def` 的）。

**索引长度常量**（`dict0mem.h:879`）：

```cpp
constexpr uint32_t DICT_ANTELOPE_MAX_INDEX_COL_LEN = REC_ANTELOPE_MAX_INDEX_COL_LEN;  // = 768
#define DICT_MAX_FIELD_LEN_BY_FORMAT(table)                              \
  (dict_table_has_atomic_blobs(table) ? REC_VERSION_56_MAX_INDEX_COL_LEN \
                                      : REC_ANTELOPE_MAX_INDEX_COL_LEN - 1)
static inline uint32_t DICT_MAX_FIELD_LEN_BY_FORMAT_FLAG(uint32_t flags) {
  if (DICT_TF_HAS_ATOMIC_BLOBS(flags)) return REC_VERSION_56_MAX_INDEX_COL_LEN;  // 3072
  return REC_ANTELOPE_MAX_INDEX_COL_LEN - 1;                                     // 767
}
constexpr uint32_t DICT_MAX_FIXED_COL_LEN = DICT_ANTELOPE_MAX_INDEX_COL_LEN;
```

> ⚠️ **不存在 `DICT_INDEX_NAME_LEN` / `MAX_NUMBER_OF_INDEX_COLS`**——名字长度由 SQL 层 `NAME_LEN = 64` 约束；列数由 `prefix_len` 的 12 位与 `n_fields` 的 10 位天然限制。
>
> ⚠️ `dict_table_t::indexes` 是 **`UT_LIST_BASE_NODE_T` 双向链表**，不是 vector；`first_index()` 返回链表头，InnoDB 约定**聚簇索引必须排在第一个**（`dict0dict.cc:2453` 有 `ut_a(!index->is_clustered() || UT_LIST_GET_LEN(table->indexes) == 0)`）。
>
> 🧹 工作区污染提示：本副本若干文件（`include/dict0dict.h`、`include/dict0mem.h:2717`、`ddl/ddl0loader.cc`、`handler0alter.cc:6355`、`ddl/ddl0ctx.cc:511`）夹有后来添加的中英文批注，非上游原文。

**标识与定位**

| 成员 | 说明 |
|---|---|
| `id` | 索引 ID，来自 `DICT_HDR_INDEX_ID` 计数器 |
| `space` / `page` | space id / **root page no**（加载时从 `se_private_data["root"]` 回填） |
| `name` | 索引名（InnoDB 内主键可能叫 `GEN_CLUST_INDEX`） |
| `table` / `table_name` | 回指表 |

**类型标志体系**（`dict0mem.h:93-115`，`DICT_IT_BITS = 10`）

```
DICT_CLUSTERED=1   聚簇（非自动生成的主键同时带 DICT_UNIQUE）
DICT_UNIQUE=2      唯一
DICT_IBUF=8        change buffer 树
DICT_CORRUPT=16    SYS_INDEXES.TYPE 里的损坏标志位
DICT_FTS=32        全文（不能与其他组合）
DICT_SPATIAL=64    空间（不能与其他组合）
DICT_VIRTUAL=128   含虚拟列
DICT_SDI=256       表空间字典索引（仅内存置位）
DICT_MULTI_VALUE=512  多值索引
```

（值 `4` 在 8.0 未定义；加载时只接受 `CLUSTERED|UNIQUE|CORRUPT|FTS|SPATIAL|VIRTUAL` 组合。）

**计数与状态**

- `n_uniq` / `n_fields` / `n_def` / `n_user_defined_cols`——见下节；
- `n_nullable` / `n_instant_nullable`（instant add column 前后的可空数）；
- `cached`（已在 dict cache——这是 `dict_index_get_n_unique*()` 的前置断言）；
- `to_be_dropped`、`online_status`（`ONLINE_INDEX_COMPLETE/CREATION/ABORTED/ABORTED_DROPPED`）、`uncommitted`（未提交到 DD）；
- `merge_threshold`（6 位位域，默认 50）；
- `trx_id_offset`（若前导字段定长，缓存 DB_TRX_ID 的列偏移）；
- `allow_duplicates` / `nulls_equal` / `disable_ahi`（各 1 bit）。

**特殊类型附加**：`srid` + `srid_is_valid` + `rtr_srs`（空间）、`rtr_ssn` / `rtr_track`（R-tree）、`parser` / `is_ngram`（全文）、`hidden`（**SE 内部隐藏**，≠ SQL 的 INVISIBLE）、`has_new_v_col`。

**统计与锁**：`stat_n_diff_key_vals[]` / `stat_n_sample_sizes[]` / `stat_index_size` / `stat_n_leaf_pages`（见 [`stats.md`](stats.md)）、`rw_lock_t lock`（索引树锁，见 [`btr.md`](btr.md)「索引锁」）、`search_info`（AHI）。

> `dict_index_t` **没有** `charset` 成员（字符集在 `dtype_t` 上），也**没有** `is_visible`。

**`dict_field_t`**（`dict0mem.h:913-938`）：

```cpp
struct dict_field_t {
  dict_col_t *col;
  id_name_t name;
  unsigned prefix_len : 12;   // 0 或前缀字节数（UTF-8 下 = mbmaxlen × 前缀字符数）
  unsigned fixed_len : 10;    // 0 或定长列长度
  unsigned is_ascending : 1;  // 0=DESC, 1=ASC
};
```

**整列索引的 `prefix_len == 0`**（不是"全长度值"）——这是与 DD 侧 `length`（总写实际字节数）的语义差异，容易混淆。

#### `n_uniq` 的三种情况与 `n_unique_in_tree`

⚠️ **勘误**：`dict_index_get_n_unique_in_tree()` 只有 **2 个分支，不是 3 个**；"无主键 → row_id"是通过聚簇分支里 `n_uniq = 1 + n_def` 间接体现的。而且 **`n_uniq` 不是调用方传入的，是在 `dict_index_build_internal_*()` 里计算出来的**。

**决定点 ①：聚簇索引**（`dict0dict.cc:2957`）：

```cpp
static dict_index_t *dict_index_build_internal_clust(const dict_table_t *table,
                                                    dict_index_t *index) {
  ...
  dict_index_t *new_index =
      dict_mem_index_create(table->name.m_name, index->name, table->space,
                            index->type, index->n_fields + total_cols_in_table);
  new_index->n_user_defined_cols = index->n_fields;
  new_index->id = index->id;
  dict_index_copy(new_index, index, table, 0, index->n_fields);

  if (dict_index_is_unique(index)) {
    /* Only the fields defined so far are needed to identify
    the index entry uniquely */
    new_index->n_uniq = new_index->n_def;              // ★ 显式主键 = 主键列数
  } else {
    /* Also the row id is needed to identify the entry */
    new_index->n_uniq = 1 + new_index->n_def;          // ★★ 无主键 = DB_ROW_ID + 0 = 1
  }
  ...
  /* Add ROW ID */
  if (!dict_index_is_unique(index)) {
    dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROW_ID), 0, true);
    set_phy_pos(table->get_sys_col(DATA_ROW_ID));
    trx_id_pos++;
  }
  /* Add TRX ID */
  dict_index_add_col(new_index, table, table->get_sys_col(DATA_TRX_ID), 0, true);
  ...
  /* Add ROLL PTR. UNDO logging is turned-off for intrinsic table and so
  DATA_ROLL_PTR system columns are not added as default system columns. */
  if (!table->is_intrinsic()) {
    dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROLL_PTR), 0, true);
  }
  ...
  /* Add to new_index non-system columns of table not yet included there */
  ...
  ut_ad(UT_LIST_GET_LEN(table->indexes) == 0);   // ★ 聚簇索引必须是第一个
  new_index->cached = true;
  return (new_index);
}
```

> ★ `trx_id_offset` 的计算（`:3057-3088`）：只有 DB_TRX_ID 之前**所有列都是定长且无前缀**时才非 0，是 B-tree 定位 DB_TRX_ID 的快速偏移。

**决定点 ②：二级索引**（`dict0dict.cc:3160`）：

```cpp
  clust_index = UT_LIST_GET_FIRST(table->indexes);   // ★ 聚簇索引必须是第一个
  ut_ad(clust_index->is_clustered());

  new_index = dict_mem_index_create(table->name.m_name, index->name, index->space,
                                    index->type,
                                    index->n_fields + 1 + clust_index->n_uniq);
  dict_index_copy(new_index, index, table, 0, index->n_fields);

  /* Add to new_index the columns necessary to determine the clustered
  index entry uniquely */
  for (i = 0; i < clust_index->n_uniq; i++) {
    field = clust_index->get_field(i);
    if (!indexed[field->col->ind]) {
      dict_index_add_col(new_index, table, field->col, field->prefix_len,
                         field->is_ascending);
      //  ★★★ 把聚簇索引前 n_uniq 个 field（= 主键列；无主键时就是 DB_ROW_ID）
      //       去重后追加到二级索引尾部 —— "二级索引叶子记录隐含包含主键"的物理实现
    } else if (dict_index_is_spatial(index)) {
      /* For spatial index, we still need to add the field to index. */
      dict_index_add_col(new_index, table, field->col, field->prefix_len,
                         field->is_ascending);
    }
  }

  if (dict_index_is_unique(index)) {
    new_index->n_uniq = index->n_fields;     // ★ 只算用户定义的列，不含追加的主键列
  } else {
    new_index->n_uniq = new_index->n_def;    // ★ 用户列 + 追加的主键列
  }

  new_index->n_fields = new_index->n_def;
  new_index->cached = true;
```

**FTS 索引**（`dict0dict.cc:3257`）：`new_index->n_uniq = 0;` —— 完全不走 B-tree 唯一性语义。

**两个"n_unique"的区别**（`include/dict0dict.ic:664`）：

```cpp
/** Gets the number of fields in the internal representation of an index
 that uniquely determine the position of an index entry in the index, if
 we do not take multiversioning into account: in the B-tree use the value
 returned by dict_index_get_n_unique_in_tree. */
static inline ulint dict_index_get_n_unique(const dict_index_t *index) {
  ut_ad(index->cached);
  return static_cast<uint16_t>(index->n_uniq);
}

/** ... if we also take multiversioning into account. */
static inline ulint dict_index_get_n_unique_in_tree(const dict_index_t *index) {
  ut_ad(index->cached);

  if (index->is_clustered()) {
    return (dict_index_get_n_unique(index));        // ★ 聚簇 → n_uniq
  }
  return (static_cast<uint16_t>(dict_index_get_n_fields(index)));  // ★ 二级 → n_fields
}
```

> ★★ **关键**：二级索引的 `n_unique_in_tree` 用的是 `n_fields`（**物理字段总数**）而不是 `n_uniq`——因为 UNIQUE 二级索引的 `n_uniq < n_fields`，B-tree 里的记录必须带上追加的主键列才能唯一定位。
>
> ★ 两个函数都断言 `index->cached`——**只有进了字典缓存的"内部表示"才可调用**。这也是 `mtr0log.cc` / `zipdecompress.cc` 里那些 dummy index 要显式绕开它的原因。

**三种情况总结**：

| 索引 | `n_uniq` | `n_fields` | `n_unique_in_tree` |
|---|---|---|---|
| 显式主键聚簇 | 主键列数 | 主键列 + DB_TRX_ID + DB_ROLL_PTR + 其余所有列 | `n_uniq` |
| 无主键 GEN_CLUST | **1**（DB_ROW_ID） | DB_ROW_ID + DB_TRX_ID + DB_ROLL_PTR + 所有列 | `n_uniq` = 1 |
| UNIQUE 二级 | 用户列数 | 用户列 + 追加的主键列 | **`n_fields`** |
| 非 UNIQUE 二级 | 用户列 + 主键列 | 同上 | `n_fields` |
| FTS | **0** | 用户列 | — |

**"用户表示 → 内部表示"的总入口**（`dict_index_add_to_cache_w_vcol()`，`dict0dict.cc:2435`）：

```cpp
  ut_a(!index->is_clustered() || UT_LIST_GET_LEN(table->indexes) == 0);

  if (!dict_index_find_and_set_cols(table, index, add_v)) { ... return DB_CORRUPTION; }

  if (index->type == DICT_FTS) {
    new_index = dict_index_build_internal_fts(table, index);
  } else if (index->is_clustered()) {
    new_index = dict_index_build_internal_clust(table, index);
  } else {
    new_index = dict_index_build_internal_non_clust(table, index);
  }
  ...
  n_ord = new_index->n_uniq;
  /* Flag the ordering columns and also set column max_prefix */
  for (i = 0; i < n_ord; i++) {
    const dict_field_t *field = new_index->get_field(i);
    if (field->col->ord_part == 0) {
      field->col->max_prefix = field->prefix_len;
      field->col->ord_part = 1;
    } else if (field->prefix_len == 0) {
      field->col->max_prefix = 0;
    } else if (field->col->max_prefix != 0 &&
               field->prefix_len > field->col->max_prefix) {
      field->col->max_prefix = field->prefix_len;
    }
    ut_ad(field->col->ord_part == 1);
  }

  /* 三个 stat_* 数组按 n_uniq 长度分配 */
  new_index->stat_n_diff_key_vals = ...dict_index_get_n_unique(new_index)...;
  new_index->stat_index_size = 1;
  new_index->stat_n_leaf_pages = 1;

  new_index->table = table;
  new_index->search_info = btr_search_info_create(new_index->heap);
  new_index->page = page_no;                 // ★ 开表路径上这里是 0，稍后回填
  rw_lock_create(index_tree_rw_lock_key, &new_index->lock, LATCH_ID_INDEX_TREE);

  dict_sys_mutex_enter();
  UT_LIST_ADD_LAST(table->indexes, new_index);   // ★★ 只挂链表，没有全局 hash / LRU
  dict_sys->size += mem_heap_get_size(new_index->heap);
  dict_sys_mutex_exit();
  ...
  dict_mem_index_free(index);                // ★ 用户表示用完即弃
```

> ⚠️ **`dict_sys_t` 里没有索引级缓存，也没有索引级 LRU**——只有 `table_hash`（按名）/ `table_id_hash`（按 id）/ `table_LRU` / `table_non_LRU`（`dict0dict.h:1023-1052`）。索引的生命周期完全跟随 `dict_table_t`，查找只有两种：链表遍历 + 按名 `dict_table_get_index_on_name()`。
> `dict_index_remove_from_cache_low()` 的 `lru_evict` 参数指的是"**表**是否被 LRU 淘汰"，不是索引自己有 LRU。

`n_uniq` 在**加入 dict cache 时**由 `dict_index_build_internal_*()` 确定：

| 索引类型 | `n_uniq` | 理由 |
|---|---|---|
| 聚簇（有显式 PK） | `n_def` = PK 字段数 | PK 本身唯一 |
| 聚簇（`GEN_CLUST_INDEX`） | `1 + n_def` | 加 `DB_ROW_ID` |
| 唯一二级索引 | `index->n_fields`（用户字段数） | 用户字段已保证唯一，**不**计追加的 PK |
| 普通二级索引 | `n_def` = 用户字段 + **追加的 PK** | 必须靠 PK 才能唯一定位一条索引项 |
| 全文索引 | `0` | `ut_ad(!!(type & DICT_FTS) == (n_uniq == 0))` |

**追加 PK 的实现**（`dict0dict.cc:3222-3237`）：把 `clust_index->n_uniq` 个主键列加到二级索引尾部（已在索引中且非前缀的列跳过；SPATIAL 强制再追加一次）。

三个访问器的差别：

```cpp
// dict0dict.ic:664-716
dict_index_get_n_fields()         → n_fields（内部表示全部字段）
dict_index_get_n_unique()         → n_uniq
dict_index_get_n_unique_in_tree() → 聚簇 = n_uniq；二级 = n_fields  ★
```

`n_unique_in_tree` 是**B 树搜索/比较真正用的字段数**（`btr0sea.cc:421`、`row0row.cc:91`）。对二级索引它等于全部字段——因为 node pointer 必须含 PK 才能唯一定位子节点。

#### 索引名规范与 ID 分配

**索引 ID 的唯一分配点**（⚠️ **`dict_sys_t` 里没有 index_id 分配器**，只有 `row_id`）：

```cpp
// dict/dict0boot.cc:64
void dict_hdr_get_new_id(table_id_t *table_id, space_index_t *index_id,
                         space_id_t *space_id, const dict_table_t *table,
                         bool disable_redo) {
  mtr_start(&mtr);
  if (table) {
    dict_disable_redo_if_temporary(table, &mtr);
  } else if (disable_redo) {
    /* In non-read-only mode we need to ensure that space-id header
    page is written to disk ... */
    mtr_set_log_mode(&mtr, (srv_read_only_mode ? MTR_LOG_NONE : MTR_LOG_NO_REDO));
  }
  dict_hdr = dict_hdr_get(&mtr);        // ★ x-latch page 7 of space 0

  if (index_id) {
    id = mach_read_from_8(dict_hdr + DICT_HDR_INDEX_ID);
    id++;
    mlog_write_ull(dict_hdr + DICT_HDR_INDEX_ID, id, &mtr);   // ★★ 每次都写 redo，无 margin
    *index_id = id;
  }
  if (table_id) { /* id >= dict_sdi_get_table_id(0) → ib::fatal "running out of table_ids" */ }
  if (space_id) { /* fil_assign_new_space_id(space_id) + 写回 DICT_HDR_MAX_SPACE_ID */ }
  mtr_commit(&mtr);
}
```

```cpp
// dict0boot.cc:48 —— DICT_HDR 页 = space 0 的 page 7
dict_hdr_t *dict_hdr_get(mtr_t *mtr) {
  block = buf_page_get(page_id_t(DICT_HDR_SPACE, DICT_HDR_PAGE_NO),
                       univ_page_size, RW_X_LATCH, UT_LOCATION_HERE, mtr);
  header = DICT_HDR + buf_block_get_frame(block);
  buf_block_dbg_add_level(block, SYNC_DICT_HEADER);
  return (header);
}
```

**DICT_HDR 页结构**（`include/dict0boot.h:94`）：

| 偏移 | 常量 | 宽 | 说明 |
|---|---|---|---|
| 0 | `DICT_HDR_ROW_ID` | 8B | 最近分配 row id（**批量回刷，margin=256**） |
| 8 | `DICT_HDR_TABLE_ID` | 8B | 最近 table id，初值 `DICT_MAX_DD_TABLES`=1024 |
| 16 | `DICT_HDR_INDEX_ID` | 8B | **最近 index id，初值 0** |
| 24 | `DICT_HDR_MAX_SPACE_ID` | 4B | 最近 space id |
| 28 | `DICT_HDR_MIX_ID_LOW` | 4B | 废弃 |
| 32/36/40/44/48 | `DICT_HDR_TABLES/TABLE_IDS/COLUMNS/INDEXES/FIELDS` | 各 4B | 旧 SYS_* 表根页号（**仅升级路径用**） |
| 56 | `DICT_HDR_FSEG_HEADER` | — | 段头 |

```cpp
constexpr uint32_t DICT_TABLES_ID = 1;
constexpr uint32_t DICT_COLUMNS_ID = 2;
constexpr uint32_t DICT_INDEXES_ID = 3;
constexpr uint32_t DICT_FIELDS_ID = 4;
constexpr uint32_t DICT_TABLE_IDS_ID = 5;      // SYS_TABLES 的二级索引
constexpr uint32_t DICT_HDR_FIRST_ID = 10;     // ★ 前 10 个留给系统表自身
constexpr uint32_t DICT_HDR = FSEG_PAGE_DATA;
```

> ★★ `DICT_HDR_INDEX_ID` 初值 0，而 `dict_hdr_get_new_id()` **先 `++` 再返回** ⇒ **第一个被分配的用户索引 ID 是 1**；1..5 预留给 SYS_* 自身索引。
>
> ★ 三个 ID 共用同一个局部 `ib_id_t id`，**一次 mtr 可同时分配 table_id + index_id + space_id**。

**row id 走另一套（内存计数器 + margin）**：

```cpp
// include/dict0boot.ic:34
static inline row_id_t dict_sys_get_new_row_id(void) {
  dict_sys_mutex_enter();
  id = dict_sys->row_id;
  if (0 == (id % DICT_HDR_ROW_ID_WRITE_MARGIN)) {
    dict_hdr_flush_row_id();        // ★ 每 256 个才持久化一次
  }
  dict_sys->row_id++;
  dict_sys_mutex_exit();
  return (id);
}
```

```cpp
// include/dict0boot.h:331
constexpr uint32_t DICT_HDR_ROW_ID_WRITE_MARGIN = 256;
```

```cpp
// dict0boot.cc:218 —— 启动时 +256 再向上对齐，防止与已用过的 row id 重叠
  dict_sys->row_id =
      DICT_HDR_ROW_ID_WRITE_MARGIN +
      ut_uint64_align_up(mach_read_from_8(dict_hdr + DICT_HDR_ROW_ID),
                         DICT_HDR_ROW_ID_WRITE_MARGIN);
```

> ★★ **对比**：index id 每次分配都写 redo（无 margin，无空洞）；row id 每 256 个才落一次盘（用 ID 空洞换性能，启动时 `+256` 补偿）。

**索引名常量**：

```cpp
// dict0mem.h:1046 —— InnoDB 唯一的保留索引名常量
const char innobase_index_reserve_name[] = "GEN_CLUST_INDEX";

// fts0fts.cc:126
const char *FTS_PREFIX = "fts_";
const char *FTS_PREFIX_5_7 = "FTS_";

// fts0fts.h:60
#define FTS_DOC_ID_INDEX_NAME "FTS_DOC_ID_INDEX"
#define FTS_DOC_ID_INDEX_NAME_LEN 16
#define FTS_INDEX_TABLE_IND_NAME "FTS_INDEX_TABLE_IND"
#define FTS_COMMON_TABLE_IND_NAME "FTS_COMMON_TABLE_IND"
```

> ⚠️ **`PRIMARY` 不是 InnoDB 常量，是字面量**——且只在**升级路径**才把 `GEN_CLUST_INDEX` 改写成 `"PRIMARY"`（`dict0load.cc:443-446`）。
>
> ⚠️ **`DB_FOREIGN_` 不是索引名前缀**——`DB_FOREIGN_DUPLICATE_KEY`（`db0err.h:103`）、`DB_FOREIGN_EXCEED_MAX_CASCADE`（`db0err.h:118`）是 `dberr_t` **错误码**。外键索引名在 8.0 由 DD 管理，InnoDB 侧无专门前缀。
>
> ★ `dict_index_t::name` 类型是 `id_name_t`，**InnoDB 侧无独立长度宏**（不存在 `DICT_INDEX_NAME_LEN`）——长度由 SQL 层 `NAME_LEN = 64` 约束。双向校验在 `dd_index_match()` 用 `my_strcasecmp` 比对，但 **PRIMARY 豁免**（因为内部名可能是 `GEN_CLUST_INDEX`）。
>
> ★ 真正的"长度"校验针对 key part 字节长度：`validate_index_len()` → `DICT_MAX_FIELD_LEN_BY_FORMAT`（767 或 3072），超限则 `dict_set_corrupted(index)`。

- **主键必须叫 `PRIMARY`**：`const char *primary_key_name = "PRIMARY";`（`sql/sql_table.cc:210`）。`dd_get_new_index_type()` 靠 `key->name == primary_key_name` 判 `IT_PRIMARY`；InnoDB 侧有双向断言（`ha_innodb.cc:14800`）。
- **无主键时**：InnoDB 在 `get_extra_columns_and_keys()` 里补一个 `add_first_index()` 名为 `PRIMARY` 的 hidden UNIQUE 索引（列 `DB_ROW_ID`），`dict_index_t` 侧名叫 `GEN_CLUST_INDEX`；升级路径里还会把 `GEN_CLUST_INDEX` 改名为 `PRIMARY`（`dict0load.cc:442-447`）。
- **FTS 辅助表名的 5.7 大写兼容**：`fts_common_tables_5_7[]`、`fts_index_selector_5_7[]`（`fts0fts.cc:144-158`），前缀常量 `FTS_PREFIX="fts_"` / `FTS_PREFIX_5_7="FTS_"`。

**ID 分配**（`dict0boot.cc:70-147`）：`DICT_HDR` 页头的 `DICT_HDR_INDEX_ID` 计数器自增并写 redo：

```cpp
id = mach_read_from_8(dict_hdr + DICT_HDR_INDEX_ID);
id++;
mlog_write_ull(dict_hdr + DICT_HDR_INDEX_ID, id, &mtr);
```

唯一分配点是 `dict_create_index()` → `dict_hdr_get_new_id(nullptr, &index->id, ...)`。

---

### 加载路径

#### 开表：从 DD 一次性构造全部索引

⚠️ **勘误**：
- **`ha_innobase::open()` 里没有任何 `indexes()` 循环**——它只做缓存查找 + 一行委托 `dd_open_table()`（`ha_innodb.cc:7276`）
- **不存在 `dict_build_index()` / `dict_index_build_from_dd()` / `dd_load_index()` / `dd_open_table_one_index()`**。真名是 **`dd_fill_one_dict_index()`**，由 **`dd_fill_dict_index()`** 调度
- **构造是"两段式"**：先建结构（page 传 0），后回填物理坐标

**真实调用链**：

```
ha_innobase::open()                          ha_innodb.cc:7177
  └─ dict_table_check_if_in_cache_low()      命中 → dd_table_match() 对账后复用
  └─ dd_open_table<dd::Table>()              dict0dd.cc:5448   ← ha_innodb.cc:7276
       └─ dd_open_table_one<Table>()         dict0dd.cc:5075
            ├─ dd_fill_dict_table()          dict0dd.cc:3808   → m_table->id = se_private_id()
            ├─ dd_fill_dict_index()          dict0dd.cc:3200   → 建索引骨架（page 全 0）
            ├─ 【第二段】for (dd_index : dd_table->indexes())   dict0dd.cc:5151
            │     → index->space / ->page / ->id / ->trx_id   ← se_private_data
            └─ dict_table_add_to_cache()     dict0dict.cc:1191
```

**★ `dd_open_table_one()` 的第二段——"一次性构造"的物理坐标回填**（`dict0dd.cc:5151`）：

```cpp
  /* Now fill the space ID and Root page number for each index */
  dict_index_t *index = m_table->first_index();
  for (const auto dd_index : dd_table->indexes()) {
    ut_ad(index != nullptr);

    const dd::Properties &se_private_data = dd_index->se_private_data();
    uint64_t id = 0;
    uint32_t root = 0;
    uint32_t sid = 0;
    uint64_t trx_id = 0;

    /* ---- space_id：来自 dd::Tablespace，不是 dd::Index ---- */
    if (dd_table->tablespace_id() == dict_sys_t::s_dd_dict_space_id) {
      sid = dict_sys_t::s_dict_space_id;                 // mysql.ibd → 0xFFFFFFFE
    } else if (dd_table->tablespace_id() == dict_sys_t::s_dd_temp_space_id) {
      sid = dict_sys_t::s_temp_space_id;                 // innodb_temporary → 0xFFFFFFFD
    } else {
      client->acquire_uncached_uncommitted<dd::Tablespace>(index_space_id, &index_space);
      index_space->se_private_data().get(dd_space_key_strings[DD_SPACE_ID], &sid);
      //                                          ★★ dd::Tablespace 的 "id"
    }

    if (first_index) {
      m_table->space = sid;
      m_table->dd_space_id = index_space_id;
      ...
      dd_load_tablespace(dd_table, m_table, heap, DICT_ERR_IGNORE_RECOVER_LOCK, dd_fsp_flags);
      first_index = false;
    }

    if (se_private_data.get(dd_index_key_strings[DD_INDEX_ID], &id) ||
        se_private_data.get(dd_index_key_strings[DD_INDEX_ROOT], &root) ||
        se_private_data.get(dd_index_key_strings[DD_INDEX_TRX_ID], &trx_id)) {
      fail = true;
      break;
    }

    ut_ad(root > 1);
    ut_ad(id != 0);
    index->page = root;      // ★★ root page no 落点
    index->space = sid;      // ★ space id 落点
    index->id = id;          // ★ index id 落点
    index->trx_id = trx_id;

    if (dict_index_is_spatial(index)) {
      index->rtr_srs.reset(fetch_srs(index->srid));
      /* 持 dict mutex 时不能开表读 DD，所以必须在这之前完成 */
    }

    index = index->next();   // ★ 与 m_table->indexes 链表同步步进
  }
```

**★ 第一段：`dd_fill_dict_index()`（`dict0dd.cc:3194`）——聚簇索引必须第一个建**：

```cpp
inline int dd_fill_dict_index(const dd::Table &dd_table, const TABLE *m_form,
                              dict_table_t *m_table, THD *m_thd) {
  if (m_form->s->keys == 0 || m_form->s->primary_key == MAX_KEY) {
    /* Create an index which is used as the clustered index;
    order the rows by the hidden InnoDB column DB_ROW_ID. */
    /*  GEN_CLUST_INDEX 聚簇索引的定义. n_uniq=1 = db_row_id
          [0] GEN_CLUST_INDEX (CLUSTERED, hidden)
          Fields: DB_ROW_ID, DB_TRX_ID, DB_ROLL_PTR, name, age.
          n_uniq: 1 (DB_ROW_ID) */
    dict_index_t *index = dict_mem_index_create(
        m_table->name.m_name, "GEN_CLUST_INDEX", 0, DICT_CLUSTERED, 0);
    index->n_uniq = 0;
    dict_index_add_to_cache(m_table, index, index->page, false);
  } else {
    /* In InnoDB, the clustered index must always be created first. */
    error = dd_fill_one_dict_index(dd_table.indexes()[m_form->s->primary_key],
                                   m_table, m_form->s, m_form->s->primary_key);
  }

  for (uint i = !m_form->s->primary_key; i < m_form->s->keys; i++) {
    ulint dd_index_num = i + ((m_form->s->primary_key == MAX_KEY) ? 1 : 0);
    //  ★ 无 PK 时偏移 +1（GEN_CLUST_INDEX 占位）
    error = dd_fill_one_dict_index(dd_table.indexes()[dd_index_num], m_table,
                                   m_form->s, i);
  }
  ...
  /* FTS：若无 FTS_DOC_ID_INDEX 则补建一个 hidden 的 */
  case FTS_NOT_EXIST_DOC_ID_INDEX:
    doc_id_index = dict_mem_index_create(m_table->name.m_name,
                                         FTS_DOC_ID_INDEX_NAME, 0, DICT_UNIQUE, 1);
    doc_id_index->add_field(FTS_DOC_ID_COL_NAME, 0, true);
    dict_index_add_to_cache(m_table, doc_id_index, doc_id_index->page, false);
    doc_id_index = UT_LIST_GET_LAST(m_table->indexes);
    doc_id_index->hidden = true;      // ★ 对应 dd::Index::is_hidden()
```

**★ 单个索引的构造：`dd_fill_one_dict_index()`（`dict0dd.cc:2945`）**：

```cpp
[[nodiscard]] static int dd_fill_one_dict_index(const dd::Index *dd_index,
                                                dict_table_t *table,
                                                const TABLE_SHARE *form,
                                                uint key_num) {
  const KEY &key = form->key_info[key_num];     // ★★ 字段来源主要是 TABLE_SHARE::key_info，不是 dd::Index！
  unsigned n_fields = key.user_defined_key_parts;
  unsigned n_uniq = n_fields;

  if (key.flags & HA_SPATIAL) {
    type = DICT_SPATIAL;
    ut_ad(n_fields == 1);
  } else if (key.flags & HA_FULLTEXT) {
    type = DICT_FTS;
    n_uniq = 0;
  } else if (key_num == form->primary_key) {
    ut_ad(key.flags & HA_NOSAME);
    type = DICT_CLUSTERED | DICT_UNIQUE;        // ★ PRIMARY
  } else {
    type = (key.flags & HA_NOSAME) ? DICT_UNIQUE : 0;   // ★ HA_NOSAME → DICT_UNIQUE
  }

  dict_index_t *index =
      dict_mem_index_create(table->name.m_name, key.name, 0, type, n_fields);
  index->n_uniq = n_uniq;

  for (unsigned i = 0; i < key.user_defined_key_parts; i++) {
    const KEY_PART_INFO *key_part = &key.key_part[i];
    const Field *field = key_part->field;

    if (field->is_virtual_gcol()) {
      index->type |= DICT_VIRTUAL;
      if ((field->gcol_info->expr_item && field->gcol_info->expr_item->returns_array()) ||
          field->is_array()) {
        index->type |= DICT_MULTI_VALUE;        // ★ 多值索引
      }
    }

    bool is_asc = true;
    if (key_part->key_part_flag & HA_REVERSE_SORT) is_asc = false;   // ★ 降序

    const auto prefix_len = get_index_prefix_len(key, key_part);    // ★ 前缀长度
    ...
    dict_index_add_col(index, table, col, prefix_len, is_asc);
  }

  index->n_user_defined_cols = key.user_defined_key_parts;

  if (dict_index_add_to_cache(table, index, 0, false) != DB_SUCCESS) {
    //                                         ★★ page_no 传 0！
    return HA_ERR_GENERIC;
  }
  ...
  if (dict_index_is_spatial(index)) {
    /* SRID 只能从 dd::Index_element → dd::Column 拿，TABLE_SHARE 里没有 */
    const dd::Column &col = dd_index->elements()[geom_col_idx]->column();
    index->fill_srid_value(col.srs_id().value_or(0), col.srs_id().has_value());
  }
  return 0;
}
```

**缓存命中也要对账**（`ha_innodb.cc:7229`）：

```cpp
      } else {
        cached = true;
        if (!dd_table_match(ib_table, table_def)) {
          dict_set_corrupted(ib_table->first_index());
          dict_table_remove_from_cache(ib_table);
          ib_table = nullptr;                   // ★ 对不上就当作没缓存
        } else {
          ib_table->acquire_with_lock();
        }
      }
```

```cpp
// dict0dd.cc:243 —— 逐个索引对账
template <typename Table>
bool dd_table_match(const dict_table_t *table, const Table *dd_table) {
  if (dd_table == nullptr || table->is_temporary()) return true;
  bool match = true;
  if (dd_table->se_private_id() != table->id) { ...  match = false; }
  if (dict_table_is_discarded(table)) return match;
  for (const auto dd_index : dd_table->indexes()) {
    const dict_index_t *index = dd_find_index(table, dd_index);
    ut_ad(index != nullptr);
    if (!dd_index_match(index, dd_index)) match = false;   // ★ id / root / trx_id 逐个比
  }
  return match;
}
```

**后台/恢复路径：`dd_table_open_on_id()`**（`dict0dd.cc:683`）——先查 `table_id_hash`，未命中走 DD：

```cpp
  HASH_SEARCH(id_hash, dict_sys->table_id_hash, hash_value, dict_table_t *,
              ib_table, ut_ad(ib_table->cached), ib_table->id == table_id);
reopen:
  if (ib_table == nullptr) {
    if (dict_table_is_sdi(table_id)) {
      /* The table is SDI table —— 不在 DD 里，纯内存构造 */
      dict_index_t *sdi_index = dict_sdi_create_idx_in_mem(space_id, false, 0, false);
      ib_table = sdi_index->table;
    } else {
      ib_table = dd_table_open_on_id_low(thd, mdl, table_id);
    }
  }
```

> ⚠️ **不存在 `dd_table_acquire()`**——等价物是 `dd_table_open_on_id/name()`（缓存 + 回源）与 `dd_table_open_on_*_in_mem()`（纯缓存，`dict0dd.ic:143`）。

入口 `dd_open_table_one()`（`dict0dd.cc:5075`）→ `dd_fill_dict_index()`：

1. **聚簇索引必须先建**（`dict_index_add_to_cache_w_vcol` 里 `ut_a(!index->is_clustered() || 链表为空)`）；无主键则造 `GEN_CLUST_INDEX`；
2. 循环 `m_form->s->keys` 构造剩余索引；
3. **构造时 root page 还未填**——`dict_index_add_to_cache(table, index, /*page_no=*/0, false)`；
4. 最后逐个索引从 `se_private_data` 回填：

```cpp
// dict0dd.cc:5149-5251（节选）
for (const auto dd_index : dd_table->indexes()) {
  const dd::Properties &se_private_data = dd_index->se_private_data();
  if (se_private_data.get(dd_index_key_strings[DD_INDEX_ID], &id) ||
      se_private_data.get(dd_index_key_strings[DD_INDEX_ROOT], &root) ||
      se_private_data.get(dd_index_key_strings[DD_INDEX_TRX_ID], &trx_id)) { fail = true; break; }
  ut_ad(root > 1);
  index->page = root;  index->space = sid;  index->id = id;  index->trx_id = trx_id;
  if (dict_index_is_spatial(index)) index->rtr_srs.reset(fetch_srs(index->srid));
  index = index->next();
}
```

一个反直觉的细节：**`dd_fill_one_dict_index()` 主要从 server 的 `form->key_info[]`（`KEY`）取类型与字段，而不是逐个读 `dd::Index_element`**——`dd::Index` 只用于取 SRID 与隐藏列判断。降序由 `key_part->key_part_flag & HA_REVERSE_SORT` 转成 `is_asc`。

`online_status` 在加载路径**不设置**（`dict_mem_index_create()` 整体置零）→ 恒为 `ONLINE_INDEX_COMPLETE`；在线建索引只在 DDL 期间于内存内设为 `ONLINE_INDEX_CREATION`。

> **SYS_INDEXES 旧路径**仍在代码里（`dict_load_index_low`，`dict0load.cc:312-485`），但只在 `srv_is_upgrade_mode` 下走。8.0 正常启动**不走**这条路。

#### 缓存与淘汰：没有索引级 LRU

- 索引挂在 `dict_table_t::indexes`（`UT_LIST_BASE_NODE_T`），**生命周期随表**；
- `dict_sys` 只有**表级** LRU（`dict_sys->table_LRU` / `table_non_LRU`），淘汰入口 `dict_make_room_in_cache()`；被淘汰的表走 `dict_table_remove_from_cache()` → 逐索引 `dict_index_remove_from_cache_low()`；
- 遍历接口（8.0 已无 `dict_table_get_indexes()` 宏，源码零命中）：`table->first_index()` + `index->next()`；按名查找 `dict_table_get_index_on_name()`。

---

### 写路径（DDL 时序）

#### 建索引的三步：拿 ID → 建树 → 回填 DD

⚠️ **勘误（8.0.39 大重构）**：
- **`row0merge.cc` / `row_merge_create_index()` / `row_merge_build_indexes()` 全部不存在**——8.0 已迁移到 `storage/innobase/ddl/`：`ddl::create_index()`（`ddl0ddl.cc:184`）、`ddl::Context::build()`（`ddl0ctx.cc:512`）、`Loader::build_all()` / `scan_and_build_indexes()`（`ddl0loader.cc:484/402`）
- **`ha_innobase::add_index()` / `drop_index()` 不存在**——ADD INDEX 一律走 prepare/inplace/commit 三阶段
- **不存在 `dict_create_index_tree_in_tablespace()`**（真名 `dict_create_index_tree_in_mem()`）

| 步骤 | 阶段 | 位置 |
|---|---|---|
| ① 拿 ID + 建 `dict_index_t` + 建 B-tree 根页 | prepare | `ddl::create_index()`（`ddl0ddl.cc:184`） |
| ② 扫表 + 排序 + 批量建树 | inplace | `ddl::Context::build()` → `Loader::scan_and_build_indexes()` |
| ③ 回填 DD（root / id / trx_id） | commit | `dd_commit_inplace_alter_table()` → `dd_write_table()` → `dd_write_index()` |

**① `ddl::create_index()` 全文**（`ddl0ddl.cc:184`）：

```cpp
dict_index_t *create_index(trx_t *trx, dict_table_t *table,
                           const Index_defn *index_def,
                           const dict_add_v_col_t *add_v) noexcept {
  const size_t n_fields = index_def->m_n_fields;

  /* Create the index prototype, using the passed in def, this is not
  a persistent operation. We pass 0 as the space id, and determine at
  a lower level the space id where to row the table. */
  auto index = dict_mem_index_create(table->name.m_name, index_def->m_name, 0,
                                     index_def->m_ind_type, n_fields);
  ut_a(index);
  index->set_committed(index_def->m_rebuild);

  for (size_t i = 0; i < n_fields; i++) {
    ...  // 虚拟列名解析
    index->add_field(name, ifield->m_prefix_len, ifield->m_is_ascending);
    //  ★ prefix_len + 升降序在这里落进 dict_field_t
  }

  /* Create B-tree */
  dict_sys_mutex_exit();

  dict_build_index_def(table, index, trx);
  //  ★★ 分配 index->id（dict_hdr_get_new_id）+ index->space = table->space

  auto err = dict_index_add_to_cache_w_vcol(table, index, add_v, index->page,
                                            trx_is_strict(trx));
  if (err != DB_SUCCESS) { trx->error_state = err; return nullptr; }

  index = dict_table_get_index_on_name(table, index_def->m_name, index_def->m_rebuild);
  ut_ad(index != nullptr);

  err = dict_create_index_tree_in_mem(index, trx);
  //  ★★★ btr_create() → index->page = root page no

  dict_sys_mutex_enter();
  if (err != DB_SUCCESS) { ... return nullptr; }

  if (dict_index_is_spatial(index)) {
    index->fill_srid_value(index_def->m_srid, index_def->m_srid_is_valid);
    index->rtr_srs.reset(fetch_srs(index->srid));
  }

  index->parser = index_def->m_parser;      // FTS parser
  index->is_ngram = index_def->m_is_ngram;
  index->has_new_v_col = has_new_v_col;

  /* Note the id of the transaction that created this index, we use it to
  restrict readers from accessing this index, to ensure read consistency. */
  ut_ad(index->trx_id == trx->id);
  index->table->def_trx_id = trx->id;
  return index;
}
```

**★ ID 分配：`dict_build_index_def()`**（`dict0crea.cc:354`）：

```cpp
void dict_build_index_def(const dict_table_t *table, dict_index_t *index, trx_t *trx) {
  ut_ad((UT_LIST_GET_LEN(table->indexes) > 0) || index->is_clustered());

  if (!table->is_intrinsic()) {
    if (srv_is_upgrade_mode) {
      index->id = dd_upgrade_indexes_num++;      // ★ 升级路径用内存计数器
    } else {
      dict_hdr_get_new_id(nullptr, &index->id, nullptr, table, false);
      //                          ★★ 正常路径：从 DICT_HDR_INDEX_ID 分配
    }
  } else {
    /* Index are re-loaded in process of creation using id. If same-id is used
    for all indexes only first index will always be retrieved ... */
    index->id = (UT_LIST_GET_LEN(table->indexes) > 0)
                    ? UT_LIST_GET_LAST(table->indexes)->id + 1 : 1;
  }

  /* Inherit the space id from the table; we store all indexes of a table
  in the same tablespace */
  index->space = table->space;

  /* Note that the index was created by this transaction. */
  index->trx_id = trx->id;      // ★ 会写进 se_private_data["trx_id"]
}
```

**★ root page no 的诞生：`dict_create_index_tree_in_mem()`**（`dict0crea.cc:390`）：

```cpp
dberr_t dict_create_index_tree_in_mem(dict_index_t *index, trx_t *trx) {
  if (index->type == DICT_FTS) {
    /* FTS index does not need an index tree */   // ★ FTS 数据在 aux 表里
    return (DB_SUCCESS);
  }

  const bool missing =
      index->table->ibd_file_missing || dict_table_is_discarded(index->table);
  if (missing) {
    index->page = FIL_NULL;
    index->trx_id = trx->id;
    return (DB_SUCCESS);
  }

  mtr_start(&mtr);
  if (index->table->is_temporary()) mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);

  page_no = btr_create(index->type, index->space, index->id, index, &mtr);
  //                                            ★★ 根页号在此产生

  index->page = page_no;        // ★★★ 两份字典的耦合点
  index->trx_id = trx->id;
  mtr_commit(&mtr);

  if (page_no == FIL_NULL) {
    err = DB_OUT_OF_FILE_SPACE;
  } else if (log_ddl != nullptr) {
    err = log_ddl->write_free_tree_log(trx, index, false);
  }
  return (err);
}
```

**★ `btr_create()`（`btr0btr.cc:832`）——两个 segment + 根页**：

```cpp
ulint btr_create(ulint type, space_id_t space, space_index_t index_id,
                 dict_index_t *index, mtr_t *mtr) {
  if (type & DICT_IBUF) {
    /* Allocate first the ibuf header page */
    ibuf_hdr_block = fseg_create(space, 0, IBUF_HEADER + IBUF_TREE_SEG_HEADER, mtr);
    block = fseg_alloc_free_page(..., IBUF_TREE_ROOT_PAGE_NO, FSP_UP, mtr);
  } else {
    block = fseg_create(space, 0, PAGE_HEADER + PAGE_BTR_SEG_TOP, mtr);
    //  ★★ non-leaf segment，段头写在新分配出来的页上——这一页就是 root page
  }
  if (block == nullptr) return (FIL_NULL);

  page_no = block->page.id.page_no();      // ★★ root page no
  frame = buf_block_get_frame(block);

  if (type & DICT_IBUF) {
    flst_init(frame + PAGE_HEADER + PAGE_BTR_IBUF_FREE_LIST, mtr);
  } else {
    /* It is a non-ibuf tree: create a file segment for leaf pages */
    if (!fseg_create(space, page_no, PAGE_HEADER + PAGE_BTR_SEG_LEAF, mtr)) {
      btr_free_root(block, mtr);
      return (FIL_NULL);
    }
  }

  uint16_t page_create_type;
  if (dict_index_is_spatial(index))       page_create_type = FIL_PAGE_RTREE;
  else if (dict_index_is_sdi(index))      page_create_type = FIL_PAGE_SDI;
  else                                    page_create_type = FIL_PAGE_INDEX;

  page = page_create(block, mtr, dict_table_is_comp(index->table), page_create_type);
  btr_page_set_level(page, nullptr, 0, mtr);

  btr_page_set_index_id(page, page_zip, index_id, mtr);
  //  ★★★ root page 上写 index_id（PAGE_INDEX_ID）—— 物理层与 dict 层的另一个耦合点

  btr_page_set_next(page, page_zip, FIL_NULL, mtr);
  btr_page_set_prev(page, page_zip, FIL_NULL, mtr);

  /* Note: Insert Buffering is disabled for temporary tables ... */
  if (!(type & DICT_CLUSTERED) && !index->table->is_temporary()) {
    ibuf_reset_free_bits(block);
  }

  ut_ad(page_get_max_insert_size(page, 2) > 2 * BTR_PAGE_MAX_REC_SIZE);
  buf_stat_per_index->inc(index_id_t(space, index_id));
  return (page_no);      // ★ 返回 root page no
}
```

**③ 回填 DD**（`handler0alter.cc:4252`）：

```cpp
template <typename Table>
static void dd_commit_inplace_alter_table(const alter_table_old_info_t &old_info,
                                          dict_table_t *new_table,
                                          const Table *old_dd_tab, Table *new_dd_tab) {
  if (new_table->is_temporary()) {
    /* No need to fill in metadata for temporary tables,
    which would not be stored in Global DD */
    return;
  }
  ...
  dd_set_table_options(new_dd_tab, new_table);
  new_table->dd_space_id = dd_space_id;

  dd_write_table(dd_space_id, new_dd_tab, new_table);
  //  ★★★ 回填 root page no 的唯一入口 → dd_write_index() → se_private_data["root"]
  ...
}
```

> ★★ 两个物理层 ↔ 字典层的耦合点：**`PAGE_INDEX_ID`**（页上写 index id）与 **`se_private_data["root"]`**（DD 里写 root page no）。崩溃后正是靠这两个把物理页与字典重新对上。

**② 建树的主体**（`ddl0loader.cc:402`）：

```cpp
dberr_t Loader::scan_and_build_indexes() noexcept {
  auto cursor = Cursor::create_cursor(m_ctx);
  if (cursor == nullptr) return DB_OUT_OF_MEMORY;

  auto err = m_ctx.read_init(cursor);
  if (err == DB_SUCCESS) {
    cursor->open();
    innobase_rec_reset(m_ctx.m_table);
    ...
    /* Read clustered index of the table and create files for secondary
    index entries for merge sort and bulk build of the indexes. */
    err = cursor->scan(m_builders);      // ★ 扫聚簇索引 → 各索引的排序临时文件
    err = cursor->finish(err);
    if (err == DB_SUCCESS) {
      err = load();                       // ★ merge sort + bulk load 进 B-tree
    }
  }
  return err;
}
```

InnoDB 是 `HTON_SUPPORTS_ATOMIC_DDL`，时序是**引擎先建、DD 最后统一提交**：

1. `dict_create_index()` 拿 ID → `dict_create_index_tree_in_mem()` → `btr_create()` 返回 root page → `index->page = page_no`（物理先落盘）；成功后写 DDL log 兜底（`log_ddl->write_free_tree_log`）；
2. `dd_write_table()` → 对每个 `dd_index` 调 `dd_write_index()`，把 `id/root/space_id/trx_id` 写进**内存中的** `dd::Table` 对象；
3. server 层 `mysql_inplace_alter_table()` 在 `ha_commit_inplace_alter_table(commit=true)` **之后**才 `thd->dd_client()->store(altered_table_def)` 一次提交（`sql_table.cc:13612-13683`）。非原子 DDL 引擎则是 DD 先 update 并显式提交。

崩溃安全由 **DDL log** 保证（见 [`physical_storage.md`](physical_storage.md)）：建索引成功提交就删掉那条 DDL log，失败/崩溃则留下，post_ddl 回放时删树。

#### SDI 里的索引副本

⚠️ **勘误**：
- **不存在 `dict_sdi_create_idx()` / `dict_sdi_create_tablespace()` / `dict_sdi_t` / `dict_sdi_get_server_version()`**。真名：`dict_sdi_create_idx_in_mem()`（`dict0crea.cc:650`）、`dict_sdi_create()`（`dict0sdi.cc:125`）、`ib_sdi_create()` / `btr_sdi_create_index()`（`btr0btr.cc:4681`）。SDI 记录的键是 **`sdi_key_t`**（`sql/handler.h:126`），不是 `dict_sdi_t`。
- **SDI 不在 mysql.ibd 里**——**每个表空间一个 SDI B-tree**，根页号写在该表空间 **page 0 的 SDI Header**。
- **`SDI_VERSION` 有两个**，别混：`const uint32_t SDI_VERSION = 1`（`dict0dict.h:74`，写进 page 0）vs `dd::SDI_VERSION = 80019`（`sql/dd/impl/sdi.h:104`，写进 SDI JSON）。

**SDI 存哪张表、以什么 key**：

```cpp
// sql/handler.h:119 —— 只有两个 type
/** Id for identifying Table SDIs */
constexpr const uint32 SDI_TYPE_TABLE = 1;
/** Id for identifying Tablespace SDIs */
constexpr const uint32 SDI_TYPE_TABLESPACE = 2;

/** Key to identify a dictionary object */
struct sdi_key_t {
  /** Type of Object, For ex: column, index, etc */
  uint32 type;
  /** Object id which should be unique in tablespsace */
  uint64 id;
};
```

> ★ 两张副本：`{SDI_TYPE_TABLE /*1*/, table.id()}`（表 SDI）、`{SDI_TYPE_TABLESPACE /*2*/, tablespace.id()}`（表空间 SDI）。

**SDI "表"是内存构造的，DD 里不存在**（`dict0crea.cc:650`）：

```cpp
dict_index_t *dict_sdi_create_idx_in_mem(space_id_t space, bool space_discarded,
                                         uint32_t in_flags, bool is_create) {
  /* 18 = strlen(SDI) + Max digits of 4 byte spaceid (10) + 1 */
  char table_name[18];
  snprintf(table_name, sizeof(table_name), "SDI_" SPACE_ID_PF, space);   // ★ SDI_<space_id>

  constexpr size_t n_cols_sdi = 5;
  dict_table_t *table = dict_mem_table_create(table_name, space, n_cols_sdi, 0, 0,
                                              table_flags, 0);
  dict_mem_table_add_col(table, heap, "type", DATA_INT, DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "id", DATA_INT, DATA_NOT_NULL | DATA_UNSIGNED, 8, true);
  dict_mem_table_add_col(table, heap, "compressed_len", DATA_INT, DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "uncompressed_len", DATA_INT, DATA_NOT_NULL | DATA_UNSIGNED, 4, true);
  dict_mem_table_add_col(table, heap, "data", DATA_BLOB, DATA_NOT_NULL, 0, true);

  table->id = dict_sdi_get_table_id(space);      // ★ 0xFFFFFFFF00000000 | space

  dict_table_add_system_columns(table, heap);

  const char *index_name = "CLUST_IND_SDI";      // ★ 固定索引名
  dict_index_t *temp_index = dict_mem_index_create(
      table_name, index_name, space,
      DICT_CLUSTERED | DICT_UNIQUE | DICT_SDI, 2);

  temp_index->add_field("type", 0, true);        // ★ 聚簇键 = (type, id)
  temp_index->add_field("id", 0, true);
  temp_index->table = table;

  /* Disable AHI on SDI tables */
  temp_index->disable_ahi = true;
  //  ★ 注释原文：SDI 表的 index id 不唯一，不满足 AHI 的校验条件

  if (!space_discarded && !is_create) {
    index_root_page_num = fsp_sdi_get_root_page_num(space, page_size_t(flags), &mtr);
    //  ★ 从 page 0 的 SDI Header 读回根页号
  } else {
    index_root_page_num = FIL_NULL;
  }

  temp_index->id = dict_sdi_get_index_id();      // ★ UINT64_MAX，所有 SDI 索引共用
  dict_index_add_to_cache(table, temp_index, index_root_page_num, false);
  ...
}
```

```cpp
// include/dict0dict.ic:1330
static inline uint64_t dict_sdi_get_table_id(space_id_t space_id) {
  return ((~0ULL << 32) | space_id);       // table_id = 0xFFFFFFFF00000000 | space_id
}
static inline space_id_t dict_sdi_get_space_id(table_id_t table_id) {
  return (static_cast<space_id_t>(table_id));
}
static inline bool dict_index_is_sdi(const dict_index_t *index) {
  return (index->type & DICT_SDI);
}
static inline bool dict_table_is_sdi(table_id_t table_id) {
  return (table_id >= dict_sdi_get_table_id(SYSTEM_TABLE_SPACE));
}
static inline space_index_t dict_sdi_get_index_id() {
  return (std::numeric_limits<uint64_t>::max());   // ★ 所有 SDI 索引共用 UINT64_MAX
}
```

**SDI B-tree 的建立**（`btr0btr.cc:4681`）：

```cpp
dberr_t btr_sdi_create_index(space_id_t space_id, bool dict_locked) {
  sdi_table = dict_sdi_get_table(space_id, dict_locked, true);
  mtr.start();

  sdi_root_page_num = btr_sdi_create(space_id, &mtr, sdi_table);
  //  ★ btr_create(DICT_CLUSTERED|DICT_UNIQUE|DICT_SDI, ...) —— 页类型 FIL_PAGE_SDI

  if (sdi_root_page_num == FIL_NULL) { ... return DB_ERROR; }
  else {
    dict_index_t *index = sdi_table->first_index();
    index->page = sdi_root_page_num;      // ★ root page no 落到 dict_index_t::page
  }

  block = buf_page_get(page_id_t(space_id, 0), page_size, RW_SX_LATCH, ..., &mtr);
  page = buf_block_get_frame(block);

  /* Write SDI Index root page numbers to Page 0 */
  fsp_sdi_write_root_to_page(page, page_size, sdi_root_page_num, &mtr);

  uint32_t fsp_flags = space->flags;
  fsp_flags_set_sdi(fsp_flags);            // ★ 置 FSP_FLAGS_MASK_SDI 位
  mlog_write_ulint(FSP_HEADER_OFFSET + FSP_SPACE_FLAGS + page, fsp_flags,
                   MLOG_4BYTES, &mtr);
  mtr.commit();

  fil_space_set_flags(space, fsp_flags);
  return (DB_SUCCESS);
}
```

```cpp
// fsp0fsp.cc:3911 —— SDI Header 在 page 0，占 8 字节
page_no_t fsp_sdi_get_root_page_num(space_id_t space, const page_size_t &page_size,
                                    mtr_t *mtr) {
  ulint sdi_offset = fsp_header_get_sdi_offset(page_size);

  uint32_t sdi_ver = mach_read_from_4(page + sdi_offset);        // ★ 前 4 字节 = SDI_VERSION (=1)
  if (sdi_ver != SDI_VERSION) {
    ib::warn(ER_IB_MSG_426) << "SDI version mismatch. Expected: " << SDI_VERSION
                            << " Current version: " << sdi_ver;
  }

  page_no_t root = mach_read_from_4(page + sdi_offset + 4);      // ★ 后 4 字节 = root page no
  ut_ad(root > 2);
  return (root);
}

void fsp_sdi_write_root_to_page(page_t *page, const page_size_t &page_size,
                                page_no_t root_page_num, mtr_t *mtr) {
  ulint sdi_offset = fsp_header_get_sdi_offset(page_size);
  mlog_write_ulint(page + sdi_offset, SDI_VERSION, MLOG_4BYTES, mtr);
  mlog_write_ulint(page + sdi_offset + 4, root_page_num, MLOG_4BYTES, mtr);
}
```

```cpp
// include/fsp0fsp.h:69
/** The number of bytes required to store SDI root page number(4)
and SDI version(4) at Page 0 */
constexpr uint32_t FSP_SDI_HEADER_LEN = 8;
```

**写 SDI：zlib 压缩后再入库**（`dict0sdi.cc:297` + `include/dict0sdi.h:40`）：

```cpp
// dict0sdi.h:34
/** Size of sdi_key_t::type */
constexpr const uint32_t SDI_TYPE_LEN = 4;
/** Size of sdi_key_t::id */
constexpr const uint32_t SDI_KEY_LEN = 8;

// dict0sdi.h:40 —— Sdi_Compressor
class Sdi_Compressor {
 public:
  inline void compress() {
    uLongf zlen = compressBound(static_cast<uLong>(m_src_len));
    m_comp_sdi = static_cast<byte *>(ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, zlen));
    switch (compress2(m_comp_sdi, &zlen, src, static_cast<uLong>(m_src_len), 6)) {
      //                                                       ★ 压缩级别固定 6
      case Z_BUF_ERROR:   ib::fatal(...ER_IB_MSG_FAILED_SDI_Z_BUF_ERROR); break;
      case Z_MEM_ERROR:   ib::fatal(...ER_IB_MSG_FAILED_SDI_Z_MEM_ERROR); break;
      case Z_STREAM_ERROR: ib::fatal(...ER_IB_MSG_SDI_Z_STREAM_ERROR); break;
      case Z_OK: m_comp_len = zlen; break;
    }
  }
};
```

```cpp
// dict0sdi.cc:297 —— dict_sdi_set() 关键段
bool dict_sdi_set(handlerton *hton, const dd::Tablespace &tablespace,
                  const dd::Table *table, const sdi_key_t *sdi_key,
                  const void *sdi, uint64_t sdi_len) {
  if (dd_tablespace_is_discarded(&tablespace)) return (false);   // Claim success

  /* Check if dd:Table has valid se_private_id ... If not, we cannot proceed
  with storing SDI as the tablespace is not created yet. */
  if (table && (table->se_private_id() == dd::INVALID_OBJECT_ID) && ...) {
    return (false);     // ★ 第一次 store 时还没 se_private_id，跳过；稍后会再 store 一次
  }

  uint32_t space_id;
  dberr_t err = dict_sdi_exists(tablespace, &space_id);
  if (err != DB_SUCCESS) { ... }

  if (fsp_is_undo_tablespace(space_id) || fsp_is_system_temporary(space_id)) {
    return (false);     // ★ undo / 临时表空间没有 SDI
  }

  trx_t *trx = check_trx_exists(current_thd);
  trx_start_if_not_started(trx, true, UT_LOCATION_HERE);
  innobase_register_trx(hton, current_thd, trx);

  Sdi_Compressor compressor(static_cast<uint32_t>(sdi_len), sdi);
  compressor.compress();                  // ★ zlib 压缩

  err = ib_sdi_set(space_id, &ib_sdi_key, static_cast<uint32_t>(sdi_len),
                   compressor.get_comp_len(), compressor.get_data(), trx);
  ...
}
```

```cpp
// dict0sdi.cc:47 —— 判断是否有 SDI：本质是看 page 0 的 FSP_FLAGS_HAS_SDI
static dberr_t dict_sdi_exists(const dd::Tablespace &dd_space, uint32_t *space_id) {
  if (dd_space.se_private_data().get(dd_space_key_strings[DD_SPACE_ID], space_id)) { ... }
  if (fsp_is_undo_tablespace(*space_id) || fsp_is_system_temporary(*space_id)) {
    return (DB_SUCCESS);      /* Claim Success */
  }
  return (fsp_has_sdi(*space_id));
}
```

**hton 回调注册**（`ha_innodb.cc:5211`）：

```cpp
  innobase_hton->sdi_create = dict_sdi_create;
  innobase_hton->sdi_drop = dict_sdi_drop;
  innobase_hton->sdi_get_keys = dict_sdi_get_keys;
  innobase_hton->sdi_get = dict_sdi_get;
  innobase_hton->sdi_set = dict_sdi_set;
  innobase_hton->sdi_delete = dict_sdi_delete;
```

> ⚠️ **`dict_sdi_get()` 的整个实现被 `#if 0` 关掉了**（`dict0sdi.cc:226`，注释 `TODO: Enable in WL#9761`）——hton->sdi_get 指向的这个空壳 `ut_d(ut_error)`。真正能读 SDI 的路径是 `ibd2sdi` 工具直读页，或 `ib_sdi_get()`（`api0api.cc:2936`）。`dict_sdi_drop()` 同样被 `#if 0` 关掉。
>
> ★ server 侧的写入方：`dd::sdi_tablespace::store_tbl_sdi()` / `store_tsp_sdi()` / `drop_tbl_sdi()` / `drop_all_sdi()`（`sql/dd/impl/sdi_tablespace.cc:133`）。
>
> ⚠️ `sql/dd/impl/raw/raw_record.cc` **与 SDI 无关**——那只是 DD 表行的通用 store/read 封装。

SDI 不是另建一套 schema，而是**把 `dd::Table` 对象图（含全部 `dd::Index` / `dd::Index_element`）序列化成 rapidjson**，存在表所属表空间的 `SYS_SDI` 表里。

- 索引相关字段：`name`、`type`、`algorithm`、`is_visible`、`options`、`se_private_data`、`comment`、`ordinal_position`；`Index_element` 写 `ordinal_position`、`length`、`order`、`hidden`、`column_opx`（列交叉引用）；
- 写入链：`dd::sdi::store(thd, table)` → `sdi_tablespace::store_tbl_sdi()` → `hton->sdi_set()` → InnoDB `dict_sdi_set()`；
- 因此 **SDI 里直接可见索引的 `root`/`id`/`trx_id`**（随 JSON 一起序列化）——这是 `.ibd` 能"自带元数据"、崩溃或导入时重建 DD 的基础。

---

### 特殊索引的元数据

#### 功能/多值、全文、空间、不可见索引

| 类型 | DD 侧 | dict 侧 |
|---|---|---|
| **功能索引** | 生成一个隐藏生成列（`HT_HIDDEN_SQL`），`dd::Index_element` 指向它，**元素本身 `hidden=false`** | `type \|= DICT_VIRTUAL`；字段取 `dict_v_col_t` 再 cast 成 `dict_col_t` |
| **多值索引** | 同上（隐藏生成列，`is_array()`） | 再置 `DICT_MULTI_VALUE`（判定：`expr_item->returns_array() \|\| field->is_array()`） |
| **全文** | `IT_FULLTEXT` | `DICT_FTS`，**`n_uniq = 0`、不建 B 树**（`dict_create_index_tree_in_mem` 直接返回）；11 张辅助表由 `fts->doc_id_index` 与 `fts->indexes` 关联；`FTS_DOC_ID` 索引 `hidden = true` |
| **空间** | `IT_SPATIAL`；**SRID 权威值在 `dd::Column::srs_id()`**（列级） | `DICT_SPATIAL`（`ut_ad(n_fields == 1)`）；`fill_srid_value()` 读入 `index->srid`，SRS 对象缓存进 `rtr_srs` |
| **不可见** | `dd::Index::is_visible()` → `mysql.indexes.is_visible` → `KEY::is_visible` → `TABLE_SHARE::visible_indexes` 位图 | **`dict_index_t` 没有 `is_visible` 成员**——引擎完全不感知，DML 照常维护 |

不可见索引的优化器侧读取（`sql/table.cc:497-502`）：

```cpp
Key_map TABLE_SHARE::usable_indexes(const THD *thd) const {
  Key_map usable_indexes(keys_in_use);
  if (!thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES))
    usable_indexes.intersect(visible_indexes);
  return usable_indexes;
}
```

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `optimizer_switch='use_invisible_indexes=on'` | 让优化器看到 INVISIBLE 索引 |
| `mysql.innodb_index_stats` | InnoDB 持久统计落点（**不是** `mysql.index_stats`） |
| `mysql.index_stats` | DD 的动态统计（`dd::Index_stat`），由 `Dictionary_client` 管理 |
| `information_schema_stats_expiry` | `SHOW INDEX` 的 Cardinality 缓存有效期（默认 24h） |
| `dict_index_t::merge_threshold` | 6 位位域，默认 50（可用 `ALTER INDEX ... MERGE_THRESHOLD=n`） |

---

## Misc

### 易混淆点

- **`dict_index_t` 没有 `is_visible`**——不可见索引**只存在于 server 侧**（DD `is_visible` → `KEY::is_visible` → `TABLE_SHARE::visible_indexes`）。它的 `hidden` 成员是另一回事（SE 内部隐藏索引：`FTS_DOC_ID_INDEX` / `DB_ROW_ID` 主键）。
- **`n_uniq` ≠ `n_unique_in_tree`**：后者对**二级索引返回全部字段**（含追加 PK），是 B 树比较真正用的字段数。
- **整列索引的 `prefix_len == 0`**，但 DD 侧 `length` **总是写实际字节数**，"未设置"用 `(uint)-1`——两套表示语义不同。
- **`dd::Index_element::hidden` 只有一种情况为 true**：二级索引里被追加的 PK 列。功能索引/多值索引用的是**隐藏列**（`HT_HIDDEN_SQL`），不是隐藏元素。
- **8.0 正常启动不读 `SYS_INDEXES`**——它只在升级路径用；root page 从 `se_private_data["root"]` 回读。
- **InnoDB 持久统计写 `mysql.innodb_index_stats`**，与 DD 的 `mysql.index_stats` 是两张不同的表。
- **降序在两套字典里的表示不同**：DD 是 `Index_element::ORDER_DESC`，SQL 层是 `HA_REVERSE_SORT`，dict 侧是 `dict_field_t::is_ascending`（0=DESC）。

### 两套字典对照表

| 概念 | `dd::Index` / `dd::Index_element` | `dict_index_t` / `dict_field_t` |
|---|---|---|
| 类型 | `enum_index_type`（IT_PRIMARY/UNIQUE/MULTIPLE/FULLTEXT/SPATIAL） | `type` 位标志（DICT_CLUSTERED/UNIQUE/FTS/SPATIAL/VIRTUAL/MULTI_VALUE…） |
| 存储算法 | `enum_index_algorithm`（IA_BTREE/RTREE/HASH/FULLTEXT） | 隐含于 type（无独立字段） |
| 字段 | `Index_element`（column、length、order、hidden） | `dict_field_t`（col、prefix_len、is_ascending） |
| 前缀 | `length`（字节数，整列也写） | `prefix_len`（整列为 0） |
| 降序 | `order = ORDER_DESC` | `is_ascending = 0` |
| 物理定位 | `se_private_data`（`id`/`space_id`/`root`/`trx_id`） | `id` / `space` / `page` / `trx_id` |
| 可见性 | `is_visible` | **无** |
| 分区 | `mysql.index_partitions`（每索引每分区一行） | 每分区独立 `dict_table_t` + 独立索引对象 |

### 一句话总结

索引元数据 = **DD 存语义 + `se_private_data` 存物理 + `dict_index_t` 存运行期**的三层分工：DD 是唯一真相（通用 schema，不为引擎开专用列），`se_private_data` 是"通用 schema 的 SE 私有扩展槽"，`dict_index_t` 是开表时的一次性投影（随表缓存、随表淘汰）。理解它的关键是**root page no 这个唯一的物理锚点**——它既是两套字典的桥，也是"索引"这个概念从元数据走向物理存储的入口。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 16.1 Data Dictionary Schema*（`mysql.indexes` 等表结构）
- *MySQL 8.0 Reference Manual → 17.6.1.2 InnoDB and MySQL Data Dictionary*（SDI 与 .ibd）
- *MySQL 8.0 Reference Manual → 15.1.20.9 Secondary Indexes and Generated Columns*

**相关文档**
- 索引类型语义：[`types.md`](types.md)
- 索引的物理存储（root page / 段区 / 空间回收）：[`physical_storage.md`](physical_storage.md)
- B 树结构与操作：[`btr.md`](btr.md)
- DD 系统整体：[`../server/dd/dd.md`](../server/dd/dd.md)；引擎侧字典：[`../server/dd/innodb_dict.md`](../server/dd/innodb_dict.md)
- 索引统计：[`stats.md`](stats.md)
- 索引 DDL 生命周期：[`operations.md`](operations.md)
- 全文辅助表与 `FTS_DOC_ID`：[`inverted.md`](inverted.md)、[`../feat/fts.md`](../feat/fts.md)
- 空间 SRID：[`rtree.md`](rtree.md)、[`../server/datatype/gis.md`](../server/datatype/gis.md)

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

---

## 核心实现

### 主线与基础构件

#### 两套字典的分工与桥：`se_private_data`

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

InnoDB 是 `HTON_SUPPORTS_ATOMIC_DDL`，时序是**引擎先建、DD 最后统一提交**：

1. `dict_create_index()` 拿 ID → `dict_create_index_tree_in_mem()` → `btr_create()` 返回 root page → `index->page = page_no`（物理先落盘）；成功后写 DDL log 兜底（`log_ddl->write_free_tree_log`）；
2. `dd_write_table()` → 对每个 `dd_index` 调 `dd_write_index()`，把 `id/root/space_id/trx_id` 写进**内存中的** `dd::Table` 对象；
3. server 层 `mysql_inplace_alter_table()` 在 `ha_commit_inplace_alter_table(commit=true)` **之后**才 `thd->dd_client()->store(altered_table_def)` 一次提交（`sql_table.cc:13612-13683`）。非原子 DDL 引擎则是 DD 先 update 并显式提交。

崩溃安全由 **DDL log** 保证（见 [`physical_storage.md`](physical_storage.md)）：建索引成功提交就删掉那条 DDL log，失败/崩溃则留下，post_ddl 回放时删树。

#### SDI 里的索引副本

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

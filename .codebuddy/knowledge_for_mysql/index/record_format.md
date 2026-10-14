# 索引记录的存储格式：索引里到底存了什么字节

> 基于 MySQL 8.0.39 源码。本篇讲 **三种索引记录的字段组成与编码细节**：聚簇记录（用户列 + 系统列）、二级索引记录（索引列 + 追加 PK）、node pointer（键 + child 页号）——以及 `row_build_index_entry` 的逐段构造、info bits、变长编码、instant 与索引的交互。
>
> **边界**：通用行格式机制（record header 通用部分、offsets 数组协议、instant 行版本状态机全文）见 [`../innodb/physical/record.md`](../innodb/physical/record.md)——本篇只讲**"索引"视角的差异**；`dict_index_t` 的字段组成（n_uniq/追加 PK 的**定义**）见 [`metadata.md`](metadata.md)，本篇讲这些定义落到**字节**后的样子。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [聚簇记录：用户列 + 系统列](#聚簇记录用户列--系统列)
    - [二级索引记录：索引列 + 追加的 PK](#二级索引记录索引列--追加的-pk)
    - [node pointer：键 + child 页号](#node-pointer键--child-页号)
    - [`trx_id_offset`：定长键的快速通道](#trx_id_offset定长键的快速通道)
  - 写路径
    - [`row_build_index_entry`：从聚簇行构造索引项](#row_build_index_entry从聚簇行构造索引项)
  - 编码细节
    - [info bits 在索引里的使用](#info-bits-在索引里的使用)
    - [变长长度字节、pack key、前缀 + NULL、虚拟列、多值、降序](#变长长度字节pack-key前缀--null虚拟列多值降序)
  - instant 与索引
    - [instant ADD/DROP 与索引记录](#instant-adddrop-与索引记录)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引记录的存储格式 = 三种记录类型 × 各自的字段组成规则：

| 记录类型 | 字段组成 | 出现在 |
|---|---|---|
| 聚簇记录 | 用户列（全部）+ `DB_ROW_ID`(6B，条件性) + `DB_TRX_ID`(6B) + `DB_ROLL_PTR`(7B) | 聚簇索引叶子 |
| 二级记录 | 用户索引列（前缀截断）+ 追加的聚簇 n_uniq 列 | 二级索引叶子 |
| node pointer | 键列（聚簇 n_uniq / 二级全部）+ `DATA_SYS_CHILD`(4B 页号) | 所有非叶层 |

### 用途

- 理解"为什么二级索引记录比聚簇小"、"为什么回表必要"；
- 理解 `PAGE_MAX_TRX_ID`/delete-mark 依赖的物理基础（见 [`transaction.md`](transaction.md)）；
- 理解前缀索引、BLOB 索引、虚拟列索引的存储代价。

### 版本演进

- COMPACT（Barracuda）与 REDUNDANT（Antelope）并存；instant ADD/DROP（v1/v2 行版本）只影响**聚簇记录**；8.0.39 现状：`trx_id_offset` 快速通道与 offsets 兜底并存。

---

## 理论基础

### 设计思想与权衡

**1. 聚簇索引 = 整行数据**：`dict_index_build_internal_clust` 把**所有未被索引覆盖的用户列也追加进聚簇索引**——这就是"聚簇索引即表"的字节级体现：

```cpp
  /* Add to new_index non-system columns of table not yet included there */
  for (size_t i = 0; i < table->get_n_user_cols(); i++) {
    dict_col_t *col = table->get_col(i);
    ut_ad(col->mtype != DATA_SYS);

    if (indexed[col->ind]) {
      continue;                                    // 已在键列里（如 PK 列）→ 跳过
    }

    dict_index_add_col(new_index, table, col, 0, true);
    //                                    ★ prefix_len=0、ASC，按 table 定义顺序原样追加
    set_phy_pos(new_index->get_col(n_fields_processed));
  }
```

**2. 二级索引的"够用即止"**：二级记录只有"索引列 + 定位所需的最小 PK 列"，**没有任何系统列**（`DATA_TRX_ID`/`DATA_ROLL_PTR` 绝不进入二级索引）——这是二级索引更小、但无法独立判可见性的根源：

```cpp
  /* Add to new_index the columns necessary to determine the clustered
  index entry uniquely */
  for (i = 0; i < clust_index->n_uniq; i++) {
    field = clust_index->get_field(i);

    if (!indexed[field->col->ind]) {
      dict_index_add_col(new_index, table, field->col, field->prefix_len,
                         field->is_ascending);
    } else if (dict_index_is_spatial(index)) {
      /* For spatial index, we still need to add the field to index. */
      dict_index_add_col(new_index, table, field->col, field->prefix_len,
                         field->is_ascending);
    }
    //  ★★ 全函数只追加"聚簇索引的 uniq 字段"，从不出现 get_sys_col(DATA_TRX_ID)
  }
```

**3. 系统列的名字与长度**（⚠️ **不存在 `DB_ROW_ID` / `DB_TRX_ID` / `DB_ROLL_PTR` 这三个裸符号**）：

```cpp
/* Precise data types for system columns and the length of those columns;
NOTE: the values must run from 0 up in the order given! All codes must
be less than 256 */
/** row id: a 48-bit integer */
constexpr uint32_t DATA_ROW_ID = 0;          // data0type.h:177
/** stored length for row id */
constexpr uint32_t DATA_ROW_ID_LEN = 6;      // :179  ★ 6 字节

/** Transaction id: 6 bytes */
constexpr size_t DATA_TRX_ID = 1;            // :182
/** Transaction ID type size in bytes. */
constexpr size_t DATA_TRX_ID_LEN = 6;        // :185  ★ 6 字节

/** Rollback data pointer: 7 bytes */
constexpr size_t DATA_ROLL_PTR = 2;          // :188
/** Rollback data pointer type size in bytes. */
constexpr size_t DATA_ROLL_PTR_LEN = 7;      // :191  ★ 7 字节

/** number of system columns defined above */
constexpr uint32_t DATA_N_SYS_COLS = 3;      // :194
/** number of system columns for intrinsic temporary table */
constexpr uint32_t DATA_ITT_N_SYS_COLS = 2;  // :197  ★ intrinsic 表没有 ROLL_PTR

/** address of the child page in node pointer */
constexpr uint32_t DATA_SYS_CHILD = 7;       // :73   ★ 这是"类型码"，不是长度
```

```cpp
static_assert(DATA_TRX_ID_LEN == 6, "DATA_TRX_ID_LEN != 6!");
static_assert(DATA_ROLL_PTR_LEN == 7, "DATA_PTR_LEN != 7!");
static_assert(DATA_TRX_ID + 1 == DATA_ROLL_PTR, "DATA_TRX_ID value invalid!");
```

> ⚠️ **`DATA_SYS_CHILD_LEN` 不存在**——node pointer 的 4 字节页号是**硬编码**的（`mem_heap_alloc(heap, 4)` + `dtype_set(..., DATA_SYS_CHILD, DATA_NOT_NULL, 4)`）。

**4. 物理字段顺序**：系统列**不在最后**，而是紧跟键列之后、其余用户列之前：

```cpp
  /* Add system columns */

  size_t trx_id_pos = new_index->n_def;      // TRX_ID 的物理字段下标

  /* Add ROW ID */
  if (!dict_index_is_unique(index)) {        // ★ 判据是 DICT_UNIQUE，不是"有没有 PK"
    dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROW_ID), 0, true);
    set_phy_pos(table->get_sys_col(DATA_ROW_ID));
    trx_id_pos++;
  }

  /* Add TRX ID */
  dict_index_add_col(new_index, table, table->get_sys_col(DATA_TRX_ID), 0, true);
  set_phy_pos(table->get_sys_col(DATA_TRX_ID));
  ...
  /* Add ROLL PTR. UNDO logging is turned-off for intrinsic table and so
  DATA_ROLL_PTR system columns are not added as default system columns to such
  tables. */
  if (!table->is_intrinsic()) {              // ★ ROLL_PTR 唯一的条件性系统列
    dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROLL_PTR), 0, true);
    set_phy_pos(table->get_sys_col(DATA_ROLL_PTR));
  }
```

> ★★ **最终物理顺序** = `[PK 列] → [ROW_ID?] → TRX_ID → ROLL_PTR → [非键用户列，按 table 顺序]`
>
> ⚠️ **常见误解**：以为 `DB_ROW_ID` 只在"没有主键"时添加。**实际判据是 `if (!dict_index_is_unique(index))`**——即 `index->type & DICT_UNIQUE`。有 PK 时聚簇索引必定带 `DICT_UNIQUE`，所以才不加 ROW_ID。
> ⚠️ `ROLL_PTR` 的豁免条件是 **intrinsic 表**（优化器内部临时表，关了 UNDO）。

**5. `dtuple_t` 的布局**（`dfield` 数组紧跟结构体之后，一次分配）：

```cpp
#define DTUPLE_EST_ALLOC(n_fields) \
  (sizeof(dtuple_t) + (n_fields) * sizeof(dfield_t))

  tuple = (dtuple_t *)buf;
  tuple->info_bits = 0;
  tuple->n_fields = n_fields;
  tuple->n_v_fields = n_v_fields;
  tuple->n_fields_cmp = n_fields;   // 默认等于 n_fields，调用方常再收窄
  tuple->fields = (dfield_t *)&tuple[1];      // ★ dfield 数组紧跟 dtuple_t
  if (n_v_fields > 0) {
    tuple->v_fields = &tuple->fields[n_fields];  // 虚拟列字段排在普通字段之后
  } else {
    tuple->v_fields = nullptr;
  }
```

> ⚠️ **`dtuple_create_from_typ()` 不存在**——真名 `dtuple_create_from_mem()`（`data0data.ic:383`），上层封装为 `dtuple_create()` / `dtuple_create_with_vcol()`。
> ⚠️ `dict_index_add_col()` **不在 `dict0dict.cc`**——8.0 已重构到 `dict/dict.cc:50`。

**3. 系统列位置固定**：`DATA_ROW_ID=0/DATA_TRX_ID=1/DATA_ROLL_PTR=2` 由 `static_assert` 锁死（`dict0dict.cc:129-134`）；聚簇记录里系统列**固定在全部用户列之后**（`DB_TRX_ID`+`DB_ROLL_PTR` 连续 13 字节）——这使"读 trx_id 不必依赖 offsets"成为可能（`trx_id_offset`）。

**4. node pointer 的键 = "树内唯一键"**：聚簇非叶记录带 `n_uniq` 列、二级非叶记录带**全部**列（含追加 PK）——因为二级索引只有带上 PK 才能唯一定位子节点（见 [`metadata.md`](metadata.md) 的 `n_unique_in_tree`）。

---

## 核心实现

### 主线与基础构件

#### 聚簇记录：用户列 + 系统列

系统列常量（`data0type.h:177-186`）：

```cpp
constexpr uint32_t DATA_ROW_ID = 0;   // 6B，仅无显式 PK 或非唯一时
constexpr size_t  DATA_TRX_ID = 1;    // 6B
constexpr size_t  DATA_ROLL_PTR = 2;  // 7B（intrinsic 表没有：DATA_ITT_N_SYS_COLS==2）
```

COMPACT 聚簇叶子记录布局：

```
[PK 列 或 DB_ROW_ID][其余用户列][DB_TRX_ID 6B][DB_ROLL_PTR 7B]
```

组装细节（`dict_index_build_internal_clust`，`dict0dict.cc:2960-3158`）：

- 非唯一聚簇 `n_uniq = 1 + n_def`（追加 `DB_ROW_ID` 参与唯一性）；
- **`DB_ROLL_PTR` 最后追加**（3090-3097），intrinsic 表不加（无 undo）；
- **所有未被索引覆盖的用户列也追加**（3115-3126）——聚簇索引 = 整行。

系统列位置映射 `dict_index_t::get_sys_col_pos`：聚簇走 `dict_col_get_clust_pos`（列在聚簇索引内的位置）、二级走 `get_col_pos(系统列号)`。`dict_table_get_nth_col_pos`（MySQL 列号 → 聚簇内位置）被 row0ins/upd 的字段定位大量使用。

#### 二级索引记录：索引列 + 追加的 PK

追加发生在 `dict_index_build_internal_non_clust`（`dict0dict.cc:3163-3255`）：

```cpp
/* Add to new_index the columns necessary to determine the clustered
   index entry uniquely */
for (i = 0; i < clust_index->n_uniq; i++) {
  field = clust_index->get_field(i);
  if (!indexed[field->col->ind]) {
    dict_index_add_col(new_index, table, field->col, field->prefix_len,
                       field->is_ascending);
  } else if (dict_index_is_spatial(index)) {
    dict_index_add_col(new_index, table, field->col, field->prefix_len,
                       field->is_ascending);   // spatial 即使已含也强制追加
  }
}
```

三个细节：

- `indexed[]` 只对**整列**（`prefix_len == 0`）出现的列置位——只建了前缀的列不算"已含"，PK 列仍完整追加；
- 追加 PK 列时**沿用聚簇索引里的 `prefix_len`**——聚簇 PK 本身是前缀索引时，二级索引里存的也是同样长度的前缀；
- 无显式 PK 时 `clust_index->n_uniq` 含 `DB_ROW_ID`，故二级记录尾部追加 `DB_ROW_ID`。

即：**二级索引记录 = 用户索引列（各按 `prefix_len` 截断）+ 不在"整列已含"之列的聚簇 n_uniq 列；无任何系统列**。

#### node pointer：键 + child 页号

`dict_index_build_node_ptr`（`dict0dict.cc:3649-3709`）：

```cpp
n_unique = dict_index_get_n_unique_in_tree_nonleaf(index);  // 聚簇 n_uniq / 二级 n_fields / R-tree 1
tuple = dtuple_create(heap, n_unique + 1);
dtuple_set_n_fields_cmp(tuple, n_unique);   // ★ 比较时排除 child 字段
dict_index_copy_types(tuple, index, n_unique);
buf = mem_heap_alloc(heap, 4);
mach_write_to_4(buf, page_no);
dfield_set_data(dtuple_get_nth_field(tuple, n_unique), buf, 4);
dtype_set(dfield_get_type(...), DATA_SYS_CHILD, DATA_NOT_NULL, 4);
rec_copy_prefix_to_dtuple(tuple, rec, index, n_unique, heap);  // 只拷前 n_unique 个字段
dtuple_set_info_bits(tuple, dtuple_get_info_bits(tuple) | REC_STATUS_NODE_PTR);
```

- child 字段是 `DATA_SYS_CHILD` 定长 4B；
- 落页时 info bits 带 `REC_STATUS_NODE_PTR` → offsets 解析自动得到 `n_unique_in_tree_nonleaf + 1` 个字段（`rec.cc:63-88`）；
- R-tree node ptr 只有 MBR + child 两个字段（`DICT_INDEX_SPATIAL_NODEPTR_SIZE = 1`，见 [`rtree.md`](rtree.md)）。

#### `trx_id_offset`：定长键的快速通道

⚠️ **勘误**：`trx_id_offset` **不由 `rec_init_offsets()` 计算**——它在**建索引时**由 `dict_index_build_internal_clust()` 一次性算好，`rec_init_offsets()` 根本不碰它。

```cpp
  new_index->trx_id_offset = 0;              // 先置 0 = "不可用"
  ...
  for (size_t i = 0; i < trx_id_pos; i++) {
    ulint fixed_size =
        new_index->get_col(i)->get_fixed_size(dict_table_is_comp(table));

    if (fixed_size == 0) {
      new_index->trx_id_offset = 0;          // ★ 前面有变长列 → 偏移不固定
      break;
    }

    dict_field_t *field = new_index->get_field(i);
    if (field->prefix_len > 0) {
      new_index->trx_id_offset = 0;          // ★ 前面有前缀索引列 → 同样放弃
      break;
    }

    /* Add fixed_size to new_index->trx_id_offset.
    Because the latter is a bit-field, an overflow
    can theoretically occur. Check for it. */
    fixed_size += new_index->trx_id_offset;

    new_index->trx_id_offset = fixed_size;   // ★ 累加得 DB_TRX_ID 距 origin 的字节偏移

    if (new_index->trx_id_offset != fixed_size) {
      /* Overflow. Pretend that this is a variable-length PRIMARY KEY. */
      ut_d(ut_error);
      ut_o(new_index->trx_id_offset = 0);
      ut_o(break);
    }
  }
```

```cpp
  /** position of the trx id column in a clustered index record, if the fields
  before it are known to be of a fixed size, 0 otherwise */
  unsigned trx_id_offset : MAX_KEY_LENGTH_BITS;     // dict0mem.h:1096
```

> ★★ **两个归零条件**：① TRX_ID 之前有**任一变长列**；② TRX_ID 之前有**前缀索引列**。所以只要 PK 里有 VARCHAR，`trx_id_offset` 就是 0，只能走 offsets 兜底。
> ★ 它还受**位域宽度**限制（`MAX_KEY_LENGTH_BITS`），溢出时 `ut_error`（debug）/ 置 0（release）。

```cpp
void rec_init_offsets(const rec_t *rec, const dict_index_t *index,
                      ulint *offsets) {
  rec_offs_make_valid(rec, index, offsets);
  if (dict_table_is_comp(index->table)) {
    rec_init_offsets_new(rec, index, offsets);
  } else {
    rec_init_offsets_old(rec, index, offsets);
  }
}
//  ★ 确认：完全不碰 trx_id_offset
```

`dict_index_t::trx_id_offset`（12 位）：**当前面所有列定长且非前缀时**，缓存 `DB_TRX_ID` 在聚簇记录里的固定偏移；否则为 0、走 offsets 数组兜底（`dict0dict.cc:3059-3088`——遇变长字段或 `prefix_len > 0` 即归 0）。

读取（`row0row.ic:41-90`）：`row_get_trx_id_offset` 先试 `index->trx_id_offset`，为 0 时用 `get_sys_col_pos(DATA_TRX_ID)` + offsets。使用面：并行扫描（`row0pread.cc`）、`row0mysql.cc`、purge 前校验（`row0umod.cc:199-215`）、redo 日志（`row0log.cc:1120`）。**这就是 `dict_field_t::fixed_len` 的第二层配合**——它还驱动 offsets 缓存（`dict_index_try_cache_rec_offsets`：前 `n_unique_in_tree` 个字段 `fixed_len` 全非 0 才能缓存）。

---

### 写路径

#### `row_build_index_entry`：从聚簇行构造索引项

`row_build_index_entry_low`（`row0row.cc:59-318`）四段：

**段 1：entry 尺寸与比较字段数**（67-92）：二级 `entry_len = n_fields`（含追加 PK）；`n_fields_cmp = n_unique_in_tree`（聚簇 `n_uniq`、二级全部）——决定 B-tree 定位比较到哪；仅 INSERT 聚簇时创建 vcol 段并剔除 instant drop 列。

**段 2：虚拟列与 spatial**（102-135）：虚拟列分支**直接取 dtuple 中已物化的 `v_fields`**（`dtuple_get_nth_v_field(row, v_col->v_pos)`）——索引页存的是计算值，计算在构建 row 的环节（INSERT/UPDATE 在 `row0upd.cc:1946`、purge 在 `row0vers.cc:655` 调 `innobase_get_computed_value`）；spatial 首字段单独算 MBR 写 `DATA_MBR_LEN`。

**段 3：前缀截断与 BLOB 768 前缀规则**（245-314）：

```cpp
if (dfield_is_null(dfield)) continue;              // NULL：截断不执行
if ((!ind_field || ind_field->prefix_len == 0) &&
    (!dfield_is_ext(dfield) || index->is_clustered())) continue;  // 页内列/聚簇列：dfield_copy 即完成
...
if (ext && !col->is_virtual()) {
  const byte *buf = row_ext_lookup(ext, col_no, &len);   // 外置 BLOB 取回
  ...
  if (ind_field->prefix_len == 0) continue;   // DYNAMIC/COMPRESSED 整列索引：完整 BLOB 进二级索引
} else if (dfield_is_ext(dfield)) {
  /* REDUNDANT/COMPACT：聚簇记录里带 768B 本地前缀 + 20B 引用 */
  len -= BTR_EXTERN_FIELD_REF_SIZE;            // 去掉引用得到 768 前缀
}
if (ind_field->prefix_len) {
  len = dtype_get_at_most_n_mbchars(...);      // ★ 按多字节字符边界截断
  dfield_set_len(dfield, len);
}
```

- REDUNDANT/COMPACT：聚簇记录里外置列带 **768 字节本地前缀**（`REC_ANTELOPE_MAX_INDEX_COL_LEN`，含 20B 引用 `BTR_EXTERN_FIELD_REF_SIZE`）——二级索引取它的 768 前缀；
- DYNAMIC/COMPRESSED：BLOB 整体外置，前缀索引经 `row_ext_lookup` 取回再截断；**整列索引把完整 BLOB 写进二级索引**（受 3072 字节限制）；
- flag 差异：`ROW_BUILD_FOR_PURGE` 走本地前缀路径不查 ext；`ROW_BUILD_FOR_UNDO` 只取 20B 引用。

---

### 编码细节

#### info bits 在索引里的使用

⚠️ **勘误**：`REC_INFO_*` / `REC_STATUS_*` **不在 `rem0types.h`**（那个文件只有 `REC_MAX_N_FIELDS` 和行格式枚举），真身在 **`rem/rec.h:114-156`**。而且**不是 2 位，是 4 位**：

```cpp
/* Info bit denoting the predefined minimum record: this bit is set
if and only if the record is the first user record on a non-leaf
B-tree page that is the leftmost page on its level
(PAGE_LEVEL is nonzero and FIL_PAGE_PREV is FIL_NULL). */
constexpr uint32_t REC_INFO_MIN_REC_FLAG = 0x10UL;    // "左最小"记录，比较时永远最小
/** The deleted flag in info bits; when bit is set to 1, it means the record has
 been delete marked */
constexpr uint32_t REC_INFO_DELETED_FLAG = 0x20UL;    // delete-mark
/* Use this bit to indicate record has version */
constexpr uint32_t REC_INFO_VERSION_FLAG = 0x40UL;    // ★ 8.0.29+ 行版本（新 instant）
/** The instant ADD COLUMN flag. When it is set to 1, it means this record
was inserted/updated after an instant ADD COLUMN. */
constexpr uint32_t REC_INFO_INSTANT_FLAG = 0x80UL;    // ★ 老 instant ADD COLUMN（V1）

/* Record status values */
constexpr uint32_t REC_STATUS_ORDINARY = 0;   // 叶子页普通记录
constexpr uint32_t REC_STATUS_NODE_PTR = 1;   // 非叶子节点指针（末尾 4 字节子页号）
constexpr uint32_t REC_STATUS_INFIMUM = 2;
constexpr uint32_t REC_STATUS_SUPREMUM = 3;
```

**物理位置**（★ COMPACT 与 REDUNDANT 不同）：

```cpp
constexpr uint32_t REC_OLD_INFO_BITS = 6; /* This is single byte bit-field */
constexpr uint32_t REC_NEW_INFO_BITS = 5; /* This is single byte bit-field */
constexpr uint32_t REC_TMP_INFO_BITS = 1; /* This is single byte bit-field */
constexpr uint32_t REC_INFO_BITS_MASK = 0xF0UL;
constexpr uint32_t REC_INFO_BITS_SHIFT = 0;
```

```cpp
static inline void rec_set_info_bits_new(rec_t *rec, ulint bits) {
  ut_ad(rec_info_bits_valid(bits));
  rec_set_bit_field_1(rec, bits, REC_NEW_INFO_BITS, REC_INFO_BITS_MASK,
                      REC_INFO_BITS_SHIFT);   // COMPACT: origin-5 的高 4 位
}
static inline void rec_set_info_bits_old(rec_t *rec, ulint bits) {
  rec_set_bit_field_1(rec, bits, REC_OLD_INFO_BITS, REC_INFO_BITS_MASK,
                      REC_INFO_BITS_SHIFT);   // REDUNDANT: origin-6 的高 4 位
}
```

```cpp
/** Sets a bit field within 1 byte. */
static inline void rec_set_bit_field_1(rec_t *rec, ulint val, ulint offs,
                                       ulint mask, ulint shift) {
  mach_write_to_1(rec - offs,                 // ★★ 一律 rec - offs：extra 在 origin 之前
                  (mach_read_from_1(rec - offs) & ~mask) | (val << shift));
}
```

**info + status 一次写**（COMPACT 专有，两者不重叠）：

```cpp
static inline void rec_set_info_and_status_bits(rec_t *rec, ulint bits) {
  static_assert(((REC_NEW_STATUS_MASK >> REC_NEW_STATUS_SHIFT) &
                 (REC_INFO_BITS_MASK >> REC_INFO_BITS_SHIFT)) == 0,
                "REC_NEW_STATUS_MASK and REC_INFO_BITS_MASK overlap");
  rec_set_status(rec, bits & REC_NEW_STATUS_MASK);          // 低 3 位 → status
  rec_set_info_bits_new(rec, bits & ~REC_NEW_STATUS_MASK);  // 高 4 位 → info bits
}
```

**位域完整性的静态断言**（★ 保证 3 字节头所有 bit 都被分配，无空洞）：

```cpp
static_assert((REC_NEW_STATUS_MASK << (8 * (REC_NEW_STATUS - 3)) ^
               REC_HEAP_NO_MASK << (8 * (REC_NEW_HEAP_NO - 4)) ^
               REC_N_OWNED_MASK << (8 * (REC_NEW_N_OWNED - 3)) ^
               REC_INFO_BITS_MASK << (8 * (REC_NEW_INFO_BITS - 3)) ^
               0xFFFFFFUL) == 0,
              "sum of new-style masks != 0xFFFFFFUL");
```

> ★★ **`REC_INFO_VERSION_FLAG` 与 `REC_INFO_INSTANT_FLAG` 互斥**（`rec_convert_dtuple_to_rec_new()` 末尾 `ut_a(!(rec_get_instant_flag_new(rec) && rec_new_is_versioned(rec)))`）。三态：
> ```cpp
> enum class Rec_instant_state {
>   REC_IS_SIMPLE,      // Instant:0, Version:0
>   REC_IS_VERSIONED,   // Instant:0, Version:1  ← 8.0.29+ 新实现
>   REC_IS_INSTANT      // Instant:1, Version:0  ← V1 老实现
> };
> ```

`rec.h:114-126` 四位：`REC_INFO_MIN_REC_FLAG`(0x10)、`REC_INFO_DELETED_FLAG`(0x20)、`REC_INFO_VERSION_FLAG`(0x40)、`REC_INFO_INSTANT_FLAG`(0x80)。

- **`MIN_REC`**：只属于"非叶层最左页的第一条 node ptr"（根页提升/批量装载时置位，`btr0btr.cc:1586`；删除最左后 `btr_set_min_rec_mark` 补标）。比较语义（`rem0cmp.cc:614-630` 注释原文）："**defined as smaller than any other node pointer, independent of any ASC/DESC flags**"——它是非叶层的"infimum node pointer"。
- **`INSTANT/VERSION`**：**只在聚簇记录上置位**（写路径入口 `rec_convert_dtuple_to_rec_comp` 断言 `ut_ad(index->is_clustered())`，`rem0rec.cc:666-667`；两 bit 互斥，状态机在 `rem0rec.cc:1078-1097`）；**二级索引记录不带**。
- **`DELETED`**：见 [`transaction.md`](transaction.md)。

#### 变长长度字节、pack key、前缀 + NULL、虚拟列、多值、降序

**★ 权威布局图**（⚠️ **不在 `rem0rec.h`**——8.0 已迁到 `include/rem0rec.ic:40`）：

```cpp
/* Offsets of the bit-fields in a new-style record. NOTE! In the table the
most significant bytes and bits are written below less significant.

        (1) byte offset         (2) bit usage within byte
        downward from
        origin ->       1       8 bits relative offset of next record
                        2       8 bits relative offset of next record
                                  the relative offset is an unsigned 16-bit
                                  integer:
                                  (offset_of_next_record
                                   - offset_of_this_record) mod 64Ki,
                                  where mod is the modulo as a non-negative
                                  number;
                                  ...
                        3       3 bits status:
                                        000=conventional record
                                        001=node pointer record (inside B-tree)
                                        010=infimum record
                                        011=supremum record
                                        1xx=reserved
                                5 bits heap number
                        4       8 bits heap number
                        5       4 bits n_owned
                                4 bits info bits
*/
```

```cpp
/* Offsets of the bit-fields in an old-style record.
        origin ->       1       8 bits pointer to next record
                        2       8 bits pointer to next record
                        3       1 bit short flag
                                7 bits number of fields
                        4       3 bits number of fields
                                5 bits heap number
                        5       8 bits heap number
                        6       4 bits n_owned
                                4 bits info bits
*/
```

> ★ **COMPACT 头 5 字节**（`REC_N_NEW_EXTRA_BYTES = 5`），**REDUNDANT 头 6 字节**（`REC_N_OLD_EXTRA_BYTES = 6`）。extra 段全部位于 origin **之前**（地址更小方向），所以所有访问都是 `rec - offs`。
> ⚠️ 还有 `REC_N_TMP_EXTRA_BYTES = 1`（instant ADD COLUMN 后的临时文件格式）。

**★ extra 段的完整构成**（由 `rec_get_converted_size_comp_prefix_low()` 推出）：

```
extra = 5 (头) + ceil(nullable/8) (NULL 位图) + [1 版本 | 1~2 字段数] + 变长长度字节
```

```cpp
  extra_size += temp ? UT_BITS_IN_BYTES(n_null)
                     : REC_N_NEW_EXTRA_BYTES + UT_BITS_IN_BYTES(n_null);
```

**★ 变长长度字节的判定：阈值是 128，不是 255**（`rem0rec.cc:926`）：

```cpp
    if (fixed_len) {
      /* 定长字段：不写任何长度字节 */
      ...
    } else if (dfield_is_ext(field)) {
      /* extern（BLOB 外部页指针）：恒定 2 字节，高 2 位 = 11 */
      ut_ad(DATA_BIG_COL(col));
      *lens = (byte)(len >> 8) | 0xc0;         // ★★ 0xc0 = 1100_0000
      lens--;
      *lens = (byte)len;
      lens--;
    } else {
      if (len < 128 ||
          !DATA_BIG_LEN_MTYPE(dtype_get_len(type), dtype_get_mtype(type))) {
        *lens = (byte)len;                     // ★ 1 字节
        lens--;
      } else {
        ut_ad(len < 16384);
        *lens = (byte)(len >> 8) | 0x80;       // ★ 2 字节：首字节最高位=1，次高位=0（非 extern）
        lens--;
        *lens = (byte)len;
        lens--;
      }
    }
```

**编码小结**（`lens` 从高地址往低地址走，即"倒着写"）：

| 情形 | 字节数 | 首字节特征 |
|---|---|---|
| 定长列 | **0**（不写长度） | — |
| `len < 128` 或列本身最大长 ≤255 | 1 | — |
| `len ≥ 128` 且是大列 | 2 | 最高位 `1`，次高位 `0`（`0x80`） |
| extern（外部页） | 2 | 最高两位 `11`（`0xc0`） |

> ★ 读侧对称：`if (len & 0x80) { len <<= 8; len |= *lens--; offs += len & 0x3fff; if (len & 0x4000) → extern; }`
> ⚠️ **`len < 16384`**：2 字节格式最多表达 14 位 = 16383。

**★ COMPACT 的 NULL：既不写长度字节，也不占数据空间**：

```cpp
      if (!(dtype_get_prtype(type) & DATA_NOT_NULL)) {
        /* It's a nullable field */
        ut_ad(n_null--);

        if (UNIV_UNLIKELY(!(byte)null_mask)) {   // null_mask 溢出到 0x100 → 前进一字节
          (nulls)--;
          null_mask = 1;
        }

        /* Set the null bit if field is NULL */
        if (dfield_is_null(field)) {
          *nulls |= null_mask;                   // ★ 只置位
          null_mask <<= 1;
          return;                                // ★★ 直接返回：无长度、无数据
        }
        null_mask <<= 1;
      }
```

> ★ NULL 位图从**低地址字节的低位**开始，逐位左移；`nulls` 指针在溢出时**递减**（往更低地址走）。

**⚠️ REDUNDANT 完全相反**——NULL 也要写 end-offset 并占空间：

```cpp
    if (dfield_is_null(field)) {
      len = dtype_get_sql_null_size(dfield_get_type(field), 0);
      data_write_sql_null(rec + end_offset, len);   // ★★ 定长 NULL 仍占 fixed_len 字节！

      end_offset += len;
      if (no_ext_and_size_less_1_byte_off_limit) {
        ored_offset = end_offset | REC_1BYTE_SQL_NULL_MASK;   // 0x80
      } else {
        ored_offset = end_offset | REC_2BYTE_SQL_NULL_MASK;   // 0x8000
      }
    } else {
      ...
      if (dfield_is_ext(field)) {
        ored_offset |= REC_2BYTE_EXTERN_MASK;                 // 0x4000
      }
    }
```

> ⚠️ REDUNDANT **没有 NULL 位图**，用"每个字段的结束偏移 + 高位标志"表达。1 字节偏移（`REC_1BYTE_OFFS_LIMIT`）与 2 字节偏移二选一——判据是 `!has_ext && data_size <= REC_1BYTE_OFFS_LIMIT`。**1 字节格式无法表达 extern**。

**★ 虚拟列 / 多值索引：挂在数据区尾部，用 2 字节总长包围**：

```cpp
  /* reserve 2 bytes for writing length */
  byte *ptr = end;
  ptr += 2;                                    // 先留 2 字节

  for (ulint col_no = 0; col_no < num_v; col_no++) {
    const dict_v_col_t *col = dict_table_get_nth_v_col(index->table, col_no);

    if (col->m_col.ord_part || !dict_table_is_comp(index->table)) {
      ulint pos = col_no;
      pos += REC_MAX_N_FIELDS;                 // ★ 虚拟列 pos 用 col_no + 1023 编码
      ptr += mach_write_compressed(ptr, pos);  //   变长压缩整数（1~5 字节）

      vfield = dtuple_get_nth_v_field(v_entry, col->v_pos);

      if (dfield_is_multi_value(vfield)) {     // ★ 多值索引：专用序列化格式
        Multi_value_logger mv_logger(...);
        mv_logger.log(&ptr);
      } else {
        flen = vfield->len;
        if (flen != UNIV_SQL_NULL) { ... }
        ptr += mach_write_compressed(ptr, flen);
        if (flen != UNIV_SQL_NULL) {
          ut_memcpy(ptr, dfield_get_data(vfield), flen);
          ptr += flen;
        }
      }
    }
  }

  mach_write_to_2(end, ptr - end);             // ★ 回填 2 字节总长（大端）
```

> ★★ **虚拟列数据不会写进 B-tree 叶子记录本身**——它只出现在 ① index build 时的临时文件记录（merge/sort buffer）与 ② REDUNDANT 表。断言 `ut_ad(!v_entry || (index->is_clustered() && temp))`。

**★ 降序索引：不影响任何字节编码，只在比较时生效**：

```cpp
struct dict_field_t {
  ...
  unsigned is_ascending : 1; /*!< 0=DESC, 1=ASC */   // dict0mem.h:929
  ...
};
```

```cpp
      ret = cmp_data(
          type->mtype, type->prtype,
          dict_index_is_ibuf(index) || index->get_field(i)->is_ascending,  // ★ 在这里生效
          dtuple_b_ptr, dtuple_f_len, rec_b_ptr, rec_f_len);
```

> ⚠️ **不存在 `DATA_DESC`**（全库 grep 0 命中）。降序是纯粹的"比较方向"开关，**记录字节完全相同**。

**★ 前缀索引的截断：按"最多 N 个字符"而非字节**（`row_build_index_entry_low` 尾部）：

```cpp
    /* If a column prefix index, take only the prefix. */
    if (ind_field->prefix_len) {
      len = dtype_get_at_most_n_mbchars(
          col->prtype, col->mbminmaxlen, ind_field->prefix_len, len,
          static_cast<char *>(dfield_get_data(dfield)));
      dfield_set_len(dfield, len);             // ★ 多字节安全的截断
    }
```

> ⚠️ 另外：REDUNDANT/COMPACT 下 off-page 列要**剥掉 20 字节 blob ref**（`len -= BTR_EXTERN_FIELD_REF_SIZE`），只留 768 前缀。

**extra_size 的反推**（读侧，`rec_init_offsets_new` 末尾）：

```cpp
  *rec_offs_base(offsets) = (rec - (lens + 1)) | REC_OFFS_COMPACT;
  //  ★ (lens+1) 是变长长度列表的结尾，与 rec 相减即得 extra 总字节数
```

| 项 | 结论（证据） |
|---|---|
| **变长长度字节** | `fixed_len` 字段 0 字节；`DATA_BIG_COL`（定义长 >255 或 BLOB 系）且实际长度 ≥128 或外置时 2 字节，否则 1 字节（`rem0rec.cc:424-452`）；**前缀索引列 `fixed_len == prefix_len`，连长度字节都没有** |
| **pack key** | **InnoDB 未实现**：`ha_innodb.cc` 无 `HA_PACK_KEY`/`HA_BINARY_PACK_KEY` 声明、`dict_table_t` 无 `pack_type`——KEY 层传下的这两个 flag 被忽略（历史一贯如此；唯一相关机制是 REDUNDANT/COMPACT 的 768B 本地前缀，与 pack key 无关） |
| **前缀 + NULL** | NULL 不占数据区（`rem0rec.cc:633-642`）；**截断只对非 NULL 生效**（`row0row.cc:249-251` 的 `dfield_is_null → continue`） |
| **虚拟列** | 索引页存**计算值**（`row_build_index_entry_low` 直接取 `v_fields` 不重算）；purge 无 TABLE 对象时由 `row_vers_build_cur_vrow` → `innobase_get_computed_value` 重算物化 |
| **多值索引** | 每个数组元素生成一条**格式与普通二级索引完全相同**的记录（`Multi_value_entry_builder` 逐元素 `dfield_set_data` 后走普通 `row_ins_sec_index_entry`）——无任何特殊编码 |
| **降序索引** | **记录字节里没有任何 DESC 标记**——存储按字段物理布局，DESC 由比较器按 `is_ascending` 取反（见 [`types.md`](types.md)） |

---

### instant 与索引

- **`INSTANT/VERSION` 位只出现在聚簇记录**（见上）；`row_versions` 布尔成员在 `table->has_row_versions()` 时为真；
- **`fields_array`**（`dict0mem.h:1468`）：`fields_array[phy_pos] = 逻辑 i` 的反查表，仅 row_versions 表创建；
- **`dict_field_t::get_phy_pos`**：前缀字段返回 `col->get_prefix_phy_pos()`（高 16 位 + `0x8000` 标志），普通字段返回 `col->get_col_phy_pos()`——v2 instant 下 phy_pos 持久化进 DD；
- **instant DROP 列**：`n_fields = n_def - n_instant_drop_cols`、`n_total_fields = n_def`——**被删列仍占 dict 字段槽位，只是不出现在新写的记录里**（`rem0rec.cc:664-697` 按物理序遍历时 `is_instant_dropped()` 的列 `continue` 不写数据）；读旧行时给 `REC_OFFS_DROP` 偏移标记（长度仍正确读出供跳过）；
- **二级索引记录形态完全不受 instant 影响**：instant DROP 的列不可能在二级索引中，追加的 PK 列来自聚簇 `n_uniq`（不含被删列），其 offsets 解析也不走 instant 分支。

**★ 写侧：按物理序遍历，跳过 "本版本不可见" 的列**（`rem0rec.cc:943`）：

```cpp
    if (index->has_instant_cols_or_row_versions()) {
      ut_ad(index->is_clustered());

      size_t n_total_fields = index->get_n_total_fields();
      size_t n_fields_stored = 0;

      /* Traverse index fields in physical order and keep storing values from
      logical dtuple */
      for (size_t i = 0; i < n_total_fields && n_fields_stored < n_fields; i++) {
        size_t logical_i = i;

        {
          dict_field_t *phy_dict_field = index->get_physical_field(i);
          if (!phy_dict_field->col->is_visible_in_version(rec_version)) {
            continue;                          // ★ 该列在本行版本中不可见 → 记录里根本没有它
          }

          logical_i = (phy_dict_field - index->fields);  // ★ 物理序 → 逻辑序 反查
        }

        const dfield_t *field = &fields[logical_i];
        ...
        store_field(field, i);
        n_fields_stored++;
      }
    } else {
      for (size_t i = 0; i < n_fields; i++) {
        store_field(&fields[i], i);            // ★ 无 instant：物理序 == 逻辑序
      }
    }
```

> ★★ **遍历上界是 `n_total_fields`（含 instant dropped 列），终止条件是"已存够 `n_fields` 个"**——这就是"被删列仍占 dict 字段槽位但不出记录"的实现。

**★ 写侧 extra 段的两处 instant 追加**（`rem0rec.cc:783`）：

```cpp
        if (index->has_instant_cols_or_row_versions()) {
          if (is_store_version(index, n_fields)) {
            /* Store current version info in one byte. */
            rec_set_instant_row_version_new(rec, index->table->current_row_version);
            nulls -= 1;                        // ★ 行版本字节插在 null 位图之后
            rec_instant_info = Rec_instant_state::REC_IS_VERSIONED;
          } else {
            ut_ad(index->has_instant_cols());
            if (index->is_tuple_instant_format(n_fields)) {
              /* Not materializing instant default. So need to store in V1 */
              uint32_t n_fields_len = rec_set_n_fields(rec, n_fields);
              nulls -= n_fields_len;           // ★ V1：再往前写 1~2 字节"字段数"
              rec_instant_info = Rec_instant_state::REC_IS_INSTANT;
            }
          }
        }
```

> ★ 所以 **instant 记录的 extra 段比普通记录多 1~2 字节**（版本字节 或 字段数字节），且**位置在 NULL 位图之后**。

**★ `n_fields` 的最终收敛**（`dict0dict.cc:2480`）：

```cpp
  /* Set the n_fields value in new_index to the actual defined
  number of fields in the cache internal representation */
  new_index->n_fields = new_index->n_def;

  if (index->is_clustered() && table->has_row_versions()) {
    new_index->n_fields = new_index->n_def - table->get_n_instant_drop_cols();
    //  ★ instant DROP 掉的列不计入 n_fields
  }
  new_index->n_total_fields = new_index->n_def;   // ★ 含 instant dropped 列
```

**★ 读侧：三种"不在记录里"的字段**（`rec_get_nth_field_offs_low`）：

```cpp
  length = rec_offs_base(offsets)[1 + n];

  if (length & REC_OFFS_SQL_NULL) {
    length = UNIV_SQL_NULL;
  } else if (length & REC_OFFS_DEFAULT) {
    length = UNIV_SQL_ADD_COL_DEFAULT;      // ★ instant 默认值，不在记录里
  } else if (length & REC_OFFS_DROP) {
    length = UNIV_SQL_INSTANT_DROP_COL;     // ★ instant DROP，不在记录里
  } else {
    length &= REC_OFFS_MASK;
    length -= offs;
  }
```

> ⚠️ **`row_build_index_entry_low()` 里没有任何 `instant_default` 引用**（`grep instant_default row0row.cc` = 0 命中）。它只有两处 instant 相关：`entry_len -= get_n_instant_drop_cols()`（:71）与 `ut_ad(!col->is_instant_dropped())`（:111）。默认值填充在 `rec_convert_dtuple_to_rec_comp()` 里做。

**★ `dict_col_t::phy_pos` 的位打包**（前缀列会出现两次）：

```cpp
/*
    31                       16      15                     0
    +------------------------+--------+---------------------+
    |   prefix_phy_pos       | F      | col_phy_pos         |
    |   (高16位)              | (15bit)|(低15bit)            |
    +------------------------+--------+---------------------+
*/
  uint32_t phy_pos{UINT32_UNDEFINED};       // dict0mem.h:558

  bool has_prefix_phy_pos() const { return (phy_pos & 0x8000); }
  uint16_t get_prefix_phy_pos() const { return ((uint16_t)(phy_pos >> 16)); }
  void set_prefix_phy_pos(uint16_t prefix_pos) {
    phy_pos = prefix_pos;
    phy_pos = phy_pos << 16;
    phy_pos |= 0x8000;                      // ★ 置标志位
  }
  uint16_t get_col_phy_pos() const { return ((phy_pos & ~0x8000) & 0xFFFF); }
```

```cpp
  uint16_t get_phy_pos() const {
    if (prefix_len != 0) return col->get_prefix_phy_pos();  // 前缀字段取前缀位置
    return col->get_col_phy_pos();
  }
```

---

## 相关的系统变量/状态变量

⚠️ **勘误（重要）**：**InnoDB 的 sysvar 不在 `sql/sys_vars.cc`**——那个文件里 `row_format` 只命中 `require_row_format`（binlog 的 row image 要求，与 InnoDB 行格式无关）。**全部定义在 `storage/innobase/handler/ha_innodb.cc`**。

```cpp
// ha_innodb.cc:23075
static MYSQL_SYSVAR_ENUM(
    default_row_format, innodb_default_row_format, PLUGIN_VAR_RQCMDARG,
    "The default ROW FORMAT for all innodb tables created without explicit"
    " ROW_FORMAT. Possible values are REDUNDANT, COMPACT, and DYNAMIC."
    " The ROW_FORMAT value COMPRESSED is not allowed",
    nullptr, nullptr, DEFAULT_ROW_FORMAT_DYNAMIC,
    &innodb_default_row_format_typelib);
```

```cpp
/** Note we cannot use rec_format_enum because we do not allow
COMPRESSED row format for innodb_default_row_format option. */
enum default_row_format_enum {
  DEFAULT_ROW_FORMAT_REDUNDANT = 0,
  DEFAULT_ROW_FORMAT_COMPACT = 1,
  DEFAULT_ROW_FORMAT_DYNAMIC = 2,          // ★ 默认
};

static ulong innodb_default_row_format = DEFAULT_ROW_FORMAT_DYNAMIC;

static const char *innodb_default_row_format_names[] = {"redundant", "compact",
                                                        "dynamic", NullS};

static TYPELIB innodb_default_row_format_typelib = {
    array_elements(innodb_default_row_format_names) - 1,
    "innodb_default_row_format_typelib", innodb_default_row_format_names,
    nullptr};
```

> ⚠️ 真名 `innodb_default_row_format_typelib`，不是 `innodb_row_format_typelib`。**COMPRESSED 不允许作为默认值**。

```cpp
// ha_innodb.cc:1084 —— ★ THDVAR（会话+全局），不是 SYSVAR
static MYSQL_THDVAR_BOOL(strict_mode, PLUGIN_VAR_OPCMDARG,
                         "Use strict mode when evaluating create options.",
                         innodb_check_session_admin, nullptr, true);

// ha_innodb.cc:22574
static MYSQL_SYSVAR_ULONG(page_size, srv_page_size,
                          PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY |
                              PLUGIN_VAR_NOPERSIST,
                          "Page size to use for all InnoDB tablespaces.",
                          nullptr, nullptr, UNIV_PAGE_SIZE_DEF,
                          UNIV_PAGE_SIZE_MIN, UNIV_PAGE_SIZE_MAX, 0);

// ha_innodb.cc:22252
static MYSQL_SYSVAR_UINT(
    compression_level, page_zip_level, PLUGIN_VAR_RQCMDARG,
    "Compression level used for compressed row format.  0 is no compression"
    ", 1 is fastest, 9 is best compression and default is 6.",
    nullptr, nullptr, DEFAULT_COMPRESSION_LEVEL, 0, 9, 0);
```

**★ 三个已删除的变量**（全库 `*.cc/*.h/*.ic` 0 命中）：

| 变量 | 状态 |
|---|---|
| `innodb_large_prefix` | **8.0 已删**——只在 `mysql-test/.../notices_enable.test:406` 作为"预期报 Unknown system variable"的用例残留 |
| `innodb_file_format` / `innodb_file_format_max` | **8.0 已删**（Barracuda 成唯一格式） |
| `innodb_prefix_index_cluster_optimization` | **8.0 已删**（5.7 引入） |

**★ 行格式不存成员字段，而是编码进 `dict_table_t::flags` 位域**：

```cpp
enum rec_format_enum {
  REC_FORMAT_REDUNDANT = 0,
  REC_FORMAT_COMPACT = 1,
  REC_FORMAT_COMPRESSED = 2,
  REC_FORMAT_DYNAMIC = 3
};

/** dict_table_t::flags bit 0 is equal to 0 if the row format = Redundant */
constexpr uint32_t DICT_TF_REDUNDANT = 0;
/** dict_table_t::flags bit 0 is equal to 1 if the row format = Compact */
constexpr uint32_t DICT_TF_COMPACT = 1;
```

```cpp
static inline rec_format_t dict_tf_get_rec_format(uint32_t flags) {
  if (!DICT_TF_GET_COMPACT(flags))       return (REC_FORMAT_REDUNDANT);
  if (!DICT_TF_HAS_ATOMIC_BLOBS(flags))  return (REC_FORMAT_COMPACT);
  if (DICT_TF_GET_ZIP_SSIZE(flags))      return (REC_FORMAT_COMPRESSED);
  return (REC_FORMAT_DYNAMIC);
}
```

> ⚠️ **不存在 `dict_table_t::row_format`，也不存在 `dict_table_get_row_format()`**（全库 0 命中）。真名 `dict_tf_get_rec_format(uint32_t flags)`（`dict0dict.ic:441`）。InnoDB 内存字典没有 `row_format` 成员，只有 DD 层 `dd::Table::row_format()`。

**★ 状态变量：没有**。8.0.39 的 `innodb_status_variables[]` 与 `export_vars` 里**没有任何 row_format / record-format 统计**。唯一相关的是页面大小：

```cpp
  export_vars.innodb_page_size = UNIV_PAGE_SIZE;
```

**观测手段只有一条**：`INFORMATION_SCHEMA.INNODB_TABLES.ROW_FORMAT`（`i_s.cc:5325`）：

```cpp
  ulint compact = DICT_TF_GET_COMPACT(table->flags);
  ulint atomic_blobs = DICT_TF_HAS_ATOMIC_BLOBS(table->flags);

  if (!compact) {
    row_format = "Redundant";
  } else if (!atomic_blobs) {
    row_format = "Compact";
  } else if (DICT_TF_GET_ZIP_SSIZE(table->flags)) {
    row_format = "Compressed";
  } else {
    row_format = "Dynamic";
  }
  //  ★ 判定顺序与 dict_tf_get_rec_format() 完全一致
```

> ⚠️ **`INNODB_TABLESPACES.ROW_FORMAT` 是另一回事**——输出"表空间能容纳哪些行格式"（`Any` / `Compact or Redundant` / `Compressed` / `Dynamic` / `Undo`），是粗粒度的，**不等于表实际的行格式**。

**★ 长度与阈值常量速查**：

| 常量 | 值 | 说明 |
|---|---|---|
| `DATA_ROW_ID_LEN` / `DATA_TRX_ID_LEN` / `DATA_ROLL_PTR_LEN` | 6 / 6 / 7 | 系统列长度 |
| `DATA_ITT_N_SYS_COLS` | 2 | intrinsic 表无 ROLL_PTR |
| `REC_ANTELOPE_MAX_INDEX_COL_LEN` | 768 | REDUNDANT/COMPACT 外置列的本地前缀 |
| `REC_VERSION_56_MAX_INDEX_COL_LEN` | 3072 | 8.0 索引列上限 |
| `BTR_EXTERN_FIELD_REF_SIZE` | 20 | 本地前缀里的外置引用 |
| `DICT_MAX_FIXED_COL_LEN` | 768 | **超过它定长列会被降级为变长** |
| `REC_N_NEW_EXTRA_BYTES` / `REC_N_OLD_EXTRA_BYTES` | 5 / 6 | 记录头 |
| `REC_NODE_PTR_SIZE` | 4 | child 页号 |
| `REC_1BYTE_OFFS_LIMIT` | — | REDUNDANT 1 字节偏移的 data_size 上限 |
| `REC_OFFS_NORMAL_SIZE` / `SMALL_SIZE` | 100 / 10 | 栈上 offsets 缓冲常用尺寸 |

> ⚠️ `dict_index_add_col()` 里 **`fixed_len > 768` 的定长列会被降级为变长**（`field->fixed_len = 0`），目的是把 extern 标志塞进长度字。注释："*If it were changed, the disk format of some fixed-length columns would change, which would be a disaster.*"

---

## Misc

### 易混淆点

**① 系统列不在记录末尾**——`TRX_ID` / `ROLL_PTR` 紧跟**键列之后、其余用户列之前**（`dict0dict.cc:3044-3126` 的添加次序）。把"整行 + 尾部版本锚"当图是错的。

**② `DB_ROW_ID` 的判据不是"有没有主键"**，是 `if (!dict_index_is_unique(index))`。有 PK 的聚簇索引**必定**带 `DICT_UNIQUE`，所以才不加 ROW_ID。

**③ `DB_ROLL_PTR` 是唯一有第二个豁免条件的系统列**：`if (!table->is_intrinsic())`（优化器内部临时表关了 UNDO）。

**④ 二级索引没有任何系统列**——这是它不能独立判可见性的物理根源。连带设计：`PAGE_MAX_TRX_ID`（8 字节，**只存在于二级索引叶子页页头**）用于"整页可见性剪枝"：

```cpp
  if (page_get_max_trx_id(buf_block_get_frame(block)) < trx_id) {
    page_set_max_trx_id(block, page_zip, trx_id, mtr);
  }
```

> ★ 若 `PAGE_MAX_TRX_ID < read_view->m_up_limit_id`，该页所有记录对所有活跃事务都可见 → **跳过聚簇索引的可见性判断**。这正是"二级记录不存 trx_id"的必然补偿。

**⑤ node pointer 的 child 字段不参与比较**：

```cpp
  dtuple_set_n_fields_cmp(tuple, n_unique);   // ★ 比较字段数 = n_unique
  //  最后一个字段（4 字节页号）被排除
```

> 理由（注释原文）：*"on upper levels in the tree there may be identical node pointers with a different page number"*。

**⑥ InnoDB 不实现 pack key**——`HA_PACK_KEY` 在 DD options 里持久化但引擎忽略。

**⑦ 降序在字节里无标记**（不存在 `DATA_DESC`）、**多值记录无特殊编码**——两者都靠"比较器/构造器"而非"存储格式"实现。

**⑧ COMPACT 的 NULL 只占 1 bit，REDUNDANT 的 NULL 占位**：

| | COMPACT | REDUNDANT |
|---|---|---|
| NULL 表示 | null bitmap 的 1 bit | 字段结束偏移的 `0x80`/`0x8000` 标志 |
| NULL 占数据空间 | **0** | **定长类型占 `fixed_len` 字节** |
| NULL 写长度字节 | **否** | 是（写 end-offset） |

**⑨ `trx_id_offset` 只在 PK 全定长时有效**——PK 里有 VARCHAR 就归零，走 offsets 兜底。

**⑩ 前缀索引列连长度字节都没有**（`fixed_len == prefix_len`）。

**⑪ `MIN_REC` 是最左非叶层的"infimum node pointer"**，与 ASC/DESC 无关（`cmp_dtuple_rec_with_match_low` 开头特判）。

**⑫ offsets 是"结束偏移"数组，不是"起始偏移"**：`base[1..n]` 存各字段**结束**偏移，起始 = 前一个的结束。高位还塞了 4 个标志：

```cpp
constexpr uint32_t REC_OFFS_COMPACT   = 1U << 31;
constexpr uint32_t REC_OFFS_SQL_NULL  = 1U << 31;
constexpr uint32_t REC_OFFS_EXTERNAL  = 1 << 30;
constexpr uint32_t REC_OFFS_DEFAULT   = 1 << 29;   // instant 默认值，不在记录里
constexpr uint32_t REC_OFFS_DROP      = 1 << 28;   // instant DROP，不在记录里
constexpr uint32_t REC_OFFS_MASK      = REC_OFFS_DROP - 1;  // 0x0FFFFFFF
```

**⑬ offsets 的头在 Debug/Release 下大小不同**：

```cpp
#ifdef UNIV_DEBUG
constexpr uint32_t REC_OFFS_HEADER_SIZE = 4;   // [0]=n_alloc [1]=n_fields [2]=rec [3]=index
#else
constexpr uint32_t REC_OFFS_HEADER_SIZE = 2;
#endif
```

> ★ 所以 Release 下 `rec_offs_make_valid()` 是 `((void)0)`——省掉两次 store。

### 三种记录对照总图

```
聚簇叶子： [PK 列…] [ROW_ID 6B?] [TRX_ID 6B] [ROLL_PTR 7B] [非键用户列…]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
           ★ 系统列紧跟键列之后，其余用户列之前（不是末尾！）

二级叶子： [索引列…(可含前缀)] [追加的 PK 列…]      ← 无任何系统列
非叶：     [非叶唯一键字段…] [CHILD 4B]            ← child 不参与比较
```

| 记录类型 | 构造者 | 字段数来源 |
|---|---|---|
| 聚簇叶子 | `dict_index_build_internal_clust()` `dict0dict.cc:2960` | `rec_get_offsets()` → `dict_index_get_n_fields(index)` |
| 二级叶子 | `dict_index_build_internal_non_clust()` `dict0dict.cc:3163` | 同上；`n_uniq` = 索引列 + PK 列（非 unique 时） |
| node pointer | `dict_index_build_node_ptr()` `dict0dict.cc:3649` | `dict_index_get_n_unique_in_tree_nonleaf(index) + 1` |

**`n_uniq` 的三种取值**：

```cpp
// 聚簇
  if (dict_index_is_unique(index)) new_index->n_uniq = new_index->n_def;
  else                             new_index->n_uniq = 1 + new_index->n_def;  // +ROW_ID

// 二级
  if (dict_index_is_unique(index)) new_index->n_uniq = index->n_fields;       // 用户列自足
  else                             new_index->n_uniq = new_index->n_def;      // 索引列+PK列
```

**⚠️ 本次核查中不存在的符号清单**：

| 说法 | 真相 |
|---|---|
| `DB_ROW_ID` / `DB_TRX_ID` / `DB_ROLL_PTR` | 裸符号不存在，真名 `DATA_ROW_ID` / `DATA_TRX_ID` / `DATA_ROLL_PTR` |
| `DATA_SYS_CHILD_LEN` | 不存在，4 字节硬编码于 `dict0dict.cc:3693/3700` |
| `dict_index_build_internal_node_ptr()` | 不存在，真名 `dict_index_build_node_ptr()` |
| `dtuple_create_from_typ()` | 不存在，真名 `dtuple_create_from_mem()` |
| `rec_convert_dtuple_to_rec_redundant()` | 不存在，真名 `rec_convert_dtuple_to_rec_old()` |
| `rec_offs_set_base()` / `rec_offs_is_valid()` | 不存在，真名 `rec_offs_base()` / `rec_offs_validate()` |
| `rec_get_nth_field_extern()` / `rec_offs_nth_sql()` | 不存在，真名 `rec_offs_nth_extern()` / `rec_offs_nth_sql_null()` |
| `dtype_get_fixed_size()` | 不存在，真名 `dtype_get_fixed_size_low()` |
| `DATA_DESC` | 不存在，降序用 `dict_field_t::is_ascending` |
| `row_build_row()` | 不存在（0 命中），真名 `row_build()` / `row_build_low()` |
| `btr_pcur_open_with_no_init()` | 不存在（5.7 名），8.0 是 `btr_pcur_t::open_no_init()` |
| `dict_table_t::row_format` / `dict_table_get_row_format()` | 都不存在，用 `dict_tf_get_rec_format(flags)` |
| `rem0rec.h` 里的布局大注释 | 不在那里，已迁到 `rem0rec.ic:40` + `rem/rec.h:63-156` |

### 一句话总结

索引记录格式的核心是**"每种记录只存它职责所需的最少字段"**：聚簇存整行 + 版本锚（MVCC 的物理基础）、二级存"索引列 + 最小定位键"（换取更小体积、付出回表代价）、node pointer 存"树内唯一键 + 页号"（换取自上而下的定位）——三种记录的字段组成差异，正是"聚簇/二级/非叶"三种职责在字节上的投影。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 15.11.1 InnoDB Row Formats*（COMPACT/REDUNDANT 行格式）

**相关文档**
- 通用行格式（offsets 协议/变长编码/NULL 位图/instant 状态机全文）：[`../innodb/physical/record.md`](../innodb/physical/record.md)
- `dict_index_t` 字段组成定义（n_uniq/追加 PK）：[`metadata.md`](metadata.md)
- KEY 层描述（`key_length`/`store_length` 等 server 侧口径）：[`construction.md`](construction.md)
- 二级索引无系统列的后果（可见性/回表）：[`transaction.md`](transaction.md)
- node pointer 在 B-tree 里的操作：[`btr.md`](btr.md)
- R-tree 记录（MBR+child）：[`rtree.md`](rtree.md)
- 倒排记录（ilist 编码）：[`inverted.md`](inverted.md)
- 降序比较取反：[`types.md`](types.md)

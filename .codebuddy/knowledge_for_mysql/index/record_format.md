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

**1. 聚簇索引 = 整行数据**：`dict_index_build_internal_clust` 把**所有未被索引覆盖的用户列也追加进聚簇索引**（3115-3126 行的循环）——这就是"聚簇索引即表"的字节级体现。

**2. 二级索引的"够用即止"**：二级记录只有"索引列 + 定位所需的最小 PK 列"，**没有任何系统列**（`DB_TRX_ID/DB_ROLL_PTR` 绝不进入二级索引）——这是二级索引更小、但无法独立判可见性（见 [`transaction.md`](transaction.md)）的根源。

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

`rec.h:114-126` 四位：`REC_INFO_MIN_REC_FLAG`(0x10)、`REC_INFO_DELETED_FLAG`(0x20)、`REC_INFO_VERSION_FLAG`(0x40)、`REC_INFO_INSTANT_FLAG`(0x80)。

- **`MIN_REC`**：只属于"非叶层最左页的第一条 node ptr"（根页提升/批量装载时置位，`btr0btr.cc:1586`；删除最左后 `btr_set_min_rec_mark` 补标）。比较语义（`rem0cmp.cc:614-630` 注释原文）："**defined as smaller than any other node pointer, independent of any ASC/DESC flags**"——它是非叶层的"infimum node pointer"。
- **`INSTANT/VERSION`**：**只在聚簇记录上置位**（写路径入口 `rec_convert_dtuple_to_rec_comp` 断言 `ut_ad(index->is_clustered())`，`rem0rec.cc:666-667`；两 bit 互斥，状态机在 `rem0rec.cc:1078-1097`）；**二级索引记录不带**。
- **`DELETED`**：见 [`transaction.md`](transaction.md)。

#### 变长长度字节、pack key、前缀 + NULL、虚拟列、多值、降序

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

---

## 相关的系统变量/状态变量

| 项 | 值/说明 |
|------|------|
| `DATA_ROW_ID_LEN` / `DATA_TRX_ID_LEN` / `DATA_ROLL_PTR_LEN` | 6 / 6 / 7 字节（`DATA_ITT_N_SYS_COLS==2` intrinsic 无 roll_ptr） |
| `REC_ANTELOPE_MAX_INDEX_COL_LEN` | 768（REDUNDANT/COMPACT 外置列的本地前缀） |
| `BTR_EXTERN_FIELD_REF_SIZE` | 20（本地前缀里的外置引用长度） |
| `DATA_BIG_COL(col)` | 定义长 >255 或 BLOB 系 → 变长长度字节可变 2 字节 |
| `DATA_SYS_CHILD` | node pointer 的 child 字段类型（定长 4B） |
| `index->trx_id_offset` | 定长键缓存 DB_TRX_ID 偏移（12 位位域） |

---

## Misc

### 易混淆点

- **聚簇记录 = 整行**：未被索引覆盖的用户列也追加进聚簇索引（`dict_index_build_internal_clust` 的 3115 行循环）——聚簇索引比"PK 索引"存得多。
- **二级索引没有任何系统列**——这是它不能独立判可见性的物理根源。
- **node pointer 的 child 字段不参与比较**（`dtuple_set_n_fields_cmp(tuple, n_unique)`）——上层可能存在"键相同、child 页号不同"的多个 node ptr。
- **InnoDB 不实现 pack key**——`HA_PACK_KEY` 在 DD options 里持久化但引擎忽略。
- **降序在字节里无标记**、**多值记录无特殊编码**——两者都靠"比较器/构造器"而非"存储格式"实现。
- **前缀索引列连长度字节都没有**（`fixed_len == prefix_len`）。
- **`MIN_REC` 是最左非叶层的"infimum node pointer"**，与 ASC/DESC 无关。

### 三种记录对照总图

```
聚簇叶子： [PK|DB_ROW_ID][其余用户列][TRX_ID 6B][ROLL_PTR 7B]   ← 整行 + 版本锚
二级叶子： [索引列(前缀截断)][追加的聚簇 n_uniq 列]              ← 指针（无版本）
非叶：     [键列(n_unique_in_tree_nonleaf)][CHILD 4B]           ← 定位用
```

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

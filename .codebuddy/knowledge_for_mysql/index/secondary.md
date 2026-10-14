# 二级索引：完整生命周期与聚簇的互动

> 基于 MySQL 8.0.39 源码。本篇是**二级索引的整合专篇**：以"为什么存在 → 物理形态 → 生命周期（创建/插入/更新/删除/purge）→ 读路径 → 与聚簇的双向校验 → 权衡"为线，把散在 8 篇里的机制串成闭环。**各机制的详写仍在原篇**，本篇给路径图与衔接点；只有生命周期函数（`row_ins_sec_index_entry_low` / `row_upd_sec_index_entry_low` / `row_sel_sec_rec_is_for_clust_rec`）在本篇逐段。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [物理形态与代码里的识别](#物理形态与代码里的识别)
    - [entry 的字段对应：`n_fields_cmp`](#entry-的字段对应n_fields_cmp)
  - 写路径：生命周期
    - [创建与插入：分派 → 定位 →（查重）→ 写入](#创建与插入分派--定位-查重-写入)
    - [change buffer 的精确决策点](#change-buffer-的精确决策点)
    - [更新：`row_upd_changes_ord_field_binary` 与删旧插新](#更新row_upd_changes_ord_field_binary-与删旧插新)
    - [删除：两阶段](#删除两阶段)
  - 读路径
    - [`row_search_mvcc` 二级分支全景](#row_search_mvcc-二级分支全景)
    - [双向校验：`row_sel_sec_rec_is_for_clust_rec`](#双向校验row_sel_sec_rec_is_for_clust_rec)
    - [锁定读与二级索引上的锁](#锁定读与二级索引上的锁)
  - 回收
    - [purge：物理删除](#purge物理删除)
  - 权衡
    - [写入代价、读收益与尺寸对比](#写入代价读收益与尺寸对比)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

二级索引（secondary index）是**除聚簇索引外的所有普通索引**：记录 = 用户索引列 + 追加的聚簇键列，**不含任何系统列**（`DB_TRX_ID`/`DB_ROLL_PTR` 绝不进入）。它是一张"指向聚簇记录的指针表"——搜索靠它缩小范围，取数据靠回表。

### 用途

- 避免全表扫描（等值/范围定位）；
- 支撑覆盖索引（需要的列都在索引里时免回表，见 [`access.md`](access.md)）；
- 承载唯一性约束（见 [`constraint.md`](constraint.md)）。

### 版本演进

- 二级索引的 delete-mark 两阶段、change buffer 只服务二级索引、`PAGE_MAX_TRX_ID` 页级粗筛——都是 InnoDB 的老机制，8.0.39 语义未变。

---

## 理论基础

### 设计思想与权衡

**1. 聚簇索引解决"取行"，二级索引解决"找行"。** 聚簇把整行按主键序排好；二级索引按别的列序排好，但只存"定位所需的最少字段"。两条索引的分工 = 物理存储的两种角色。

**2. 二级索引是"多版本共享的指针"**：因为它没有版本信息（无 trx_id/roll_ptr），一条二级索引项可以被多个历史版本共享——这正是删除必须走 delete-mark 两阶段、判断"能不能物理删"必须回溯聚簇版本链的根本原因（详见 [`transaction.md`](transaction.md)）。

**3. 唯一性只能"加锁 + 查重"，不能靠 MVCC**（见 [`transaction.md`](transaction.md)「唯一性检查为什么必须加锁」）。

**4. 读/写/回收三方各自有不同的"二级索引观"**：

```
读：粗筛（PAGE_MAX_TRX_ID）→ ICP → 回表 → 校验（row_sel_sec_rec_is_for_clust_rec）
写：定位（PAGE_CUR_LE + BTR_INSERT）→ 查重加锁 → 乐观/悲观插入
回收：delete-mark 留下的记录由 purge 借 node->ref 定位后物理删除
```

### ★ 为什么二级索引存"主键值"而不是物理 rowid

二级索引的叶子记录里，指向聚簇记录的部分是**完整的主键值**，不是 `(page_no, heap_no)` 这样的物理位置。这是本篇最容易被忽略、但影响最大的一个决策：

| 候选 | 若采用会怎样 |
|---|---|
| **物理 rowid**（page no + offset） | 回表变成"直接定位页内偏移"——**读最快**（省掉一次从根到叶的 B+ 树搜索） |
| **主键值**（逻辑指针） | 回表必须走一次完整的聚簇索引搜索——读更慢，但…… |

★ **决定性约束：聚簇索引的物理位置会变。** 页分裂、页合并、`OPTIMIZE` 重建、purge 触发的页回收都会移动行。若二级索引存物理位置，则每次聚簇索引重组织都要回头更新**该行在所有二级索引里的指针**——一次行移动变成 N 次索引更新（N = 二级索引数），且这些更新本身又会引发新的页分裂。**写放大不可控**。

存主键值则把这个耦合彻底切断：只要主键不变，二级索引就不用动（改主键是显式的 DML，代价由用户承担且频率低）。

★ **这是"用读的代价换写的代价"**——与 [`transaction.md`](transaction.md) 讲的"二级索引不版本化"是**同一个取舍的两次体现**：InnoDB 在二级索引上系统性地选择了"让读路径变贵、让写路径变简单"。

**三个直接推论**（都很实用）：

1. **主键越长，所有二级索引越大**——每个二级索引的每条记录都要重复存一遍完整主键。这就是为什么"用短主键（自增 BIGINT 或 INT）"不只是页分裂的考量，**更是所有二级索引体积的考量**。复合主键尤其危险：N 列主键会在每个二级索引里重复 N 列。
2. **无主键表（GEN_CLUST_INDEX）用 6 字节隐藏 rowid**——二级索引存的就是这个 6 字节值。因为它是引擎生成、永不改变的，实际上比用户主键更稳定（但用户无法用它定位，也无法控制顺序）。
3. **回表的代价 = 一次聚簇索引点查**——所以"覆盖索引"（索引含全部所需列，无需回表）的收益，本质上就是**省掉这个"逻辑指针 → 物理行"的解引用**。见 [`access.md`](access.md)。

**他库对照**：Oracle 的二级索引（B*Tree）存的是 **rowid（物理）**，靠行迁移时的"转发地址"处理位置变化；PostgreSQL 的二级索引存 **ctid（物理）**，靠 **HOT 更新**（页内更新不改 ctid）来减轻更新代价——两者都是"物理指针 + 缓解机制"，MySQL 是"逻辑指针 + 解引用代价"。**没有谁更优，取决于更新模式**：MySQL 的方案在"主键稳定、更新频繁移动行"的场景下更省心，在"主键很长"的场景下代价明显。

---

## 核心实现

### 主线与基础构件

#### 物理形态与代码里的识别

记录形态（字节级见 [`record_format.md`](record_format.md)「二级索引记录」）：

```
[用户索引列（各按 prefix_len 截断）][追加的聚簇 n_uniq 列]
```

代码里"二级索引"的两种识别方式：

- 普通路径：`!index->is_clustered()`（+ `is_multi_value()` 三分支）；
- **页级 MVCC 语义路径**：`dict_index_is_sec_or_ibuf`（`dict0dict.ic:160`）——`!(type & DICT_CLUSTERED) || (type & DICT_IBUF)`，用于 `PAGE_MAX_TRX_ID` 的维护/校验（`page0page.cc:441/597/737`、`btr0load.cc:384` 等）——**聚簇页有记录内 trx_id 无需页级粗筛，二级/ibuf 页才有 `PAGE_MAX_TRX_ID`**。

```cpp
static inline ulint dict_index_is_sec_or_ibuf(const dict_index_t *index) {
  ulint type = index->type;
  return (!(type & DICT_CLUSTERED) || (type & DICT_IBUF));
  //  ★ 第二项必要：ibuf 树本身带 DICT_IBUF；change buffer 的 dummy index 在恢复期
  //    也可能被判成"非聚簇"，这里显式兜住
}
```

> ★★ 这个判据决定了一个页**要不要维护 `PAGE_MAX_TRX_ID`**——即"要不要参与二级索引页级 MVCC"。

**★ `DICT_*` 标志位全表**（⚠️ **不连续，缺 bit2**——老 `DICT_UNIVERSAL` 已移除）：

```cpp
constexpr uint32_t DICT_CLUSTERED   = 1;    // bit0
constexpr uint32_t DICT_UNIQUE      = 2;    // bit1
constexpr uint32_t DICT_IBUF        = 8;    // bit3   ← bit2 空缺
constexpr uint32_t DICT_CORRUPT     = 16;   // bit4
constexpr uint32_t DICT_FTS         = 32;   // bit5  不可与其他组合
constexpr uint32_t DICT_SPATIAL     = 64;   // bit6  不可与其他组合
constexpr uint32_t DICT_VIRTUAL     = 128;  // bit7
constexpr uint32_t DICT_SDI         = 256;  // bit8  仅内存结构
constexpr uint32_t DICT_MULTI_VALUE = 512;  // bit9
constexpr uint32_t DICT_IT_BITS = 10;       // unsigned type : 10
```

**★ 三个"数量"字段的区别**（这是理解二级索引比较语义的关键）：

```cpp
  /** number of fields from the beginning which are enough to determine an index
  entry uniquely */
  unsigned n_uniq : 10;              // ★ 唯一性字段数（不含 MVCC 隐式主键）
  /** number of fields defined so far */
  unsigned n_def : 10;
  /** number of fields in the index */
  unsigned n_fields : 10;            // ★ 内部表示总字段数（二级 = 键列 + 隐式主键）
  /** number of columns the user defined to be in the index */
  unsigned n_user_defined_cols : 10;
```

```cpp
static inline ulint dict_index_get_n_unique_in_tree(const dict_index_t *index) {
  if (index->is_clustered()) {
    return (dict_index_get_n_unique(index));   // 聚簇：只主键列，trx_id/roll_ptr 不参与
  }
  return (static_cast<uint16_t>(dict_index_get_n_fields(index)));
  //  ★★ 二级：= 全部字段数（含末尾隐式主键）——B-tree 内靠"键列+主键"唯一化
}
```

> ★★ 所以 **`row_ins_must_modify_rec()` 对二级索引的复用条件是"全字段相等"**（`low_match >= n_unique_in_tree == n_fields`），而注释明确说：即使唯一二级索引的非唯一列不同，也**不**就地更新（有锁语义顾虑）。
> ⚠️ **`dict_index_is_clustered()` / `dict_index_is_multi_value()` 这两个自由函数不存在**（全仓 0 命中）——8.0 一律用成员函数 `index->is_clustered()` / `is_multi_value()`。

#### entry 的字段对应：`n_fields_cmp`

`row_build_index_entry`（`row0row.cc:84-92`）：

```cpp
if (dict_index_is_ibuf(index)) {
  dtuple_set_n_fields_cmp(entry, entry_len);
} else {
  dtuple_set_n_fields_cmp(entry, dict_index_get_n_unique_in_tree(index));
}
```

二级索引 entry **物理上**含"用户列 + 追加 PK"，**比较字段数** = `n_unique_in_tree`（二级 = 全部字段）。插入定位（`PAGE_CUR_LE` 的 `low_match/up_match` 语义）与查重（临时改成 `n_unique`）都依赖这个值。

**★ 落点在 `row_build_index_entry_low()`**（`row0row.cc:59`）：

```cpp
dtuple_t *row_build_index_entry_low(const dtuple_t *row, const row_ext_t *ext,
                                    const dict_index_t *index, mem_heap_t *heap,
                                    ulint flag) {
  entry_len = dict_index_get_n_fields(index);   // ★ 已含末尾隐式主键
  ...
  if (dict_index_is_ibuf(index)) {
    dtuple_set_n_fields_cmp(entry, entry_len);
  } else {
    dtuple_set_n_fields_cmp(entry, dict_index_get_n_unique_in_tree(index));
  }
  for (i = 0; i < entry_len + num_v; i++) {
    ...
    if (UNIV_UNLIKELY(dfield_get_type(dfield2)->mtype == DATA_MISSING)) {
      /* The field has not been initialized in the row.
      This should be from trx_undo_rec_get_partial_row(). */
      return (nullptr);
      //  ★★ 这就是 row_purge_remove_sec_if_poss 里 "if (!entry) return" 的由来：
      //     undo 记录早于该索引创建 → 缺列 → 无法构造 entry → 跳过
    }
    ...
    if (ind_field->prefix_len) {
      len = dtype_get_at_most_n_mbchars(col->prtype, col->mbminmaxlen,
                                        ind_field->prefix_len, len, ...);
      dfield_set_len(dfield, len);              // ★ 前缀索引截断
    }
  }
```

```cpp
constexpr uint32_t ROW_BUILD_NORMAL      = 0;
constexpr uint32_t ROW_BUILD_FOR_PURGE   = 1;   // ★ purge 专用
constexpr uint32_t ROW_BUILD_FOR_UNDO    = 2;
constexpr uint32_t ROW_BUILD_FOR_INSERT  = 3;
```

> ⚠️ **`ROW_BUILD_FOR_PURGE` 在 `row_build_low()` / `row_build()` 里没有任何处理**——`row_build_low` 的第 1 参 `type` 只有 `ROW_COPY_POINTERS` / `ROW_COPY_DATA`。`ROW_BUILD_FOR_PURGE` 全仓只影响 `row_build_index_entry_low()`（唯一的专属分支在 spatial 首字段 MBR 的处理）。
> ⚠️ **`row_build()` 只对聚簇索引工作**（`ut_ad(index->is_clustered())`）——它是 `row_build_index_entry` 的**逆函数**。purge **不调** `row_build()`，而是用 `trx_undo_rec_get_partial_row()` 从 undo 恢复出 `node->row`。

**★ `row_search_index_entry()` 用"总字段数"严格匹配**（`row0row.cc:932`）：

```cpp
enum row_search_result row_search_index_entry(dict_index_t *index,
                                              const dtuple_t *entry, ulint mode,
                                              btr_pcur_t *pcur, mtr_t *mtr) {
  if (dict_index_is_spatial(index)) {
    rtr_pcur_open(index, entry, PAGE_CUR_RTREE_LOCATE, mode, pcur, ...);
  } else {
    pcur->open(index, 0, entry, PAGE_CUR_LE, mode, mtr, UT_LOCATION_HERE);
    //  ★ PAGE_CUR_LE：定位到"≤ entry 的最后一条"，靠 low_match 判断是否精确命中
  }

  switch (pcur->get_btr_cur()->flag) {
    case BTR_CUR_UNSET: ut_d(ut_error); ut_o(break);
    case BTR_CUR_DELETE_REF:
      ut_a(mode & BTR_DELETE && !dict_index_is_spatial(index));
      return (ROW_NOT_DELETED_REF);
    case BTR_CUR_DEL_MARK_IBUF:
    case BTR_CUR_DELETE_IBUF:
    case BTR_CUR_INSERT_TO_IBUF:
      return (ROW_BUFFERED);        // ★ 被 change buffer 缓冲
    case BTR_CUR_HASH: case BTR_CUR_HASH_NOT_ATTEMPTED:
    case BTR_CUR_HASH_FAIL: case BTR_CUR_BINARY:
      break;
  }

  low_match = pcur->get_low_match();
  rec = pcur->get_rec();
  n_fields = dtuple_get_n_fields(entry);      // ★★ 用总字段数，不是 n_fields_cmp

  if (page_rec_is_infimum(rec)) return (ROW_NOT_FOUND);
  else if (low_match != n_fields) return (ROW_NOT_FOUND);  // ★ 全字段匹配（含隐式 PK）

  return (ROW_FOUND);
}
```

```cpp
enum btr_cur_method {
  BTR_CUR_UNSET = 0,
  BTR_CUR_HASH = 1,
  BTR_CUR_HASH_NOT_ATTEMPTED,
  BTR_CUR_HASH_FAIL,
  BTR_CUR_BINARY,
  BTR_CUR_INSERT_TO_IBUF,     // ★ 已缓冲进 ibuf
  BTR_CUR_DEL_MARK_IBUF,
  BTR_CUR_DELETE_IBUF,
  BTR_CUR_DELETE_REF
};
```

---

### 写路径：生命周期

#### 创建与插入：分派 → 定位 →（查重）→ 写入

**分派**（`row_ins_index_entry`，`row0ins.cc:3304-3321`）三分支：

```cpp
if (index->is_clustered()) {
  return (row_ins_clust_index_entry(index, entry, thr, false));
} else if (index->is_multi_value()) {
  return (row_ins_sec_index_multi_value_entry(index, entry, multi_val_pos, thr));
} else {
  return (row_ins_sec_index_entry(index, entry, thr, false));
}
```

主循环 `row_ins`（`row0ins.cc:3567`）逐个索引插，跳过 `DICT_FTS` 与 corrupted，`DB_DUPLICATE_KEY` 记录 `trx->error_index` 后立即返回。**聚簇是 `first_index()` 先插入**。

**`row_ins_sec_index_entry`**（3159-3243）：先乐观（`BTR_MODIFY_LEAF`），`DB_FAIL` 后悲观（`BTR_MODIFY_TREE`，trx_id 传 0）；非 spatial 且非临时表时 `search_mode |= BTR_INSERT`（change buffer 总开关）。

**`row_ins_sec_index_entry_low`**（2795-3068）五步：

1. **搜索定位**：`btr_cur_search_to_nth_level(PAGE_CUR_LE, search_mode | BTR_INSERT, ...)`——用 LE 让 `low_match/up_match` 都有意义；非 `check_unique_secondary` 时加 `BTR_IGNORE_SEC_UNIQUE`；
2. **缓冲出口**：`cursor.flag == BTR_CUR_INSERT_TO_IBUF` → `goto func_exit`（已进 change buffer，完成）；
3. **唯一检查**：`dict_index_is_unique(index) && (low_match >= n_unique || up_match >= n_unique)` → `row_ins_scan_sec_index_for_duplicate`（查重中对候选记录加 S 锁）→ 重新定位再插入。时序注释原文（2979-2983）：

```cpp
/* We did not find a duplicate and we have now locked with s-locks
the necessary records to prevent any insertion of a duplicate by
another transaction. Let us now reposition the cursor and
continue the insertion. */
```

**不是"先插入后检查"**——查重在插入前完成、用锁防并发竞态、查重后必须重新 search 一次；

4. **实际写入**：`dup_chk_only` 直接返回；`row_ins_must_modify_rec`（旧记录 delete-marked 且足够匹配 → 复用）→ `row_ins_sec_index_entry_by_modify`（反标记 delete-mark）；否则乐观插入（`BTR_MODIFY_TREE` 下乐观失败再悲观）；
5. 成功后 `page_update_max_trx_id`（更新页级粗筛水位）。

**★ 完整五步源码**（`row0ins.cc:2795-3068`）：

```cpp
dberr_t row_ins_sec_index_entry_low(uint32_t flags, ulint mode,
                                    dict_index_t *index, mem_heap_t *offsets_heap,
                                    mem_heap_t *heap, dtuple_t *entry,
                                    trx_id_t trx_id, que_thr_t *thr,
                                    bool dup_chk_only) {
  ulint search_mode = mode;
  rec_offs_init(offsets_);
  ut_ad(!index->is_clustered());
  ut_ad(mode == BTR_MODIFY_LEAF || mode == BTR_MODIFY_TREE);

  mtr_start(&mtr);

  if (index->table->is_temporary()) {
    mtr.set_log_mode(MTR_LOG_NO_REDO);
  } else if (!dict_index_is_spatial(index)) {
    /* Enable insert buffering if it's neither temp-table nor spatial index. */
    search_mode |= BTR_INSERT;              // ★① change buffer 总开关
  }
  ...
  /* Note that we use PAGE_CUR_LE as the search mode, because then
  the function will return in both low_match and up_match of the
  cursor sensible values */
  if (!thr_get_trx(thr)->check_unique_secondary) {
    search_mode |= BTR_IGNORE_SEC_UNIQUE;
  }
  ...
    btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_LE, search_mode,
                                &cursor, 0, __FILE__, __LINE__, &mtr);

  if (cursor.flag == BTR_CUR_INSERT_TO_IBUF) {
    ut_ad(!dict_index_is_spatial(index));
    /* The insert was buffered during the search: we are done */
    goto func_exit;                         // ★② 已进 change buffer，直接返回
  }

  n_unique = dict_index_get_n_unique(index);

  if (dict_index_is_unique(index) &&
      (cursor.low_match >= n_unique || cursor.up_match >= n_unique)) {
    mtr_commit(&mtr);                       // ★★ 查重前先释放页闩
    ...
    err = row_ins_scan_sec_index_for_duplicate(flags, index, entry, thr, check,
                                               &mtr, offsets_heap);
    mtr_commit(&mtr);
    switch (err) {
      case DB_SUCCESS: break;
      case DB_DUPLICATE_KEY:
        if (!index->is_committed()) {
          dict_set_corrupted(index);
          err = DB_SUCCESS;                 // ★ 在线建索引期间的重 dup 不报错
        }
        [[fallthrough]];
      default: return err;
    }
    ...
    /* We did not find a duplicate and we have now locked with s-locks
    the necessary records to prevent any insertion of a duplicate by
    another transaction. Let us now reposition the cursor and
    continue the insertion. */
    btr_cur_search_to_nth_level(
        index, 0, entry, PAGE_CUR_LE,
        (search_mode & ~(BTR_INSERT | BTR_IGNORE_SEC_UNIQUE)), &cursor, 0,
        __FILE__, __LINE__, &mtr);          // ★③ 重新定位，且清掉会进 ibuf 的两个位
  }

  if (dup_chk_only) goto func_exit;

  if (row_ins_must_modify_rec(&cursor)) {   // ★④ 复用 delete-marked 记录
    offsets = rec_get_offsets(btr_cur_get_rec(&cursor), index, ...);
    err = row_ins_sec_index_entry_by_modify(flags, mode, &cursor, &offsets,
                                            offsets_heap, heap, entry, thr, &mtr);
  } else {
    if (mode == BTR_MODIFY_LEAF) {
      err = btr_cur_optimistic_insert(flags, &cursor, ...);
    } else {
      ut_ad(mode == BTR_MODIFY_TREE);
      if (buf_LRU_buf_pool_running_out()) { err = DB_LOCK_TABLE_FULL; goto func_exit; }
      err = btr_cur_optimistic_insert(flags, &cursor, ...);
      if (err == DB_FAIL) {
        err = btr_cur_pessimistic_insert(flags, &cursor, ...);
      }
    }
    if (err == DB_SUCCESS && trx_id) {      // ★⑤ 仅在 trx_id != 0 时更新
      page_update_max_trx_id(btr_cur_get_block(&cursor),
                             btr_cur_get_page_zip(&cursor), trx_id, &mtr);
    }
  }
func_exit:
  mtr_commit(&mtr);
  return err;
}
```

> ★★ **第 3 步重定位时清掉了 `BTR_INSERT | BTR_IGNORE_SEC_UNIQUE`**——保证二次搜索**不会**再进一次 change buffer。
> ★ 第 5 步的 `trx_id` **只有乐观轮次非 0**（外壳 `row_ins_sec_index_entry` 第二轮传 0），所以 `PAGE_MAX_TRX_ID` 只在乐观插入成功时更新。

**★ 复用条件 `row_ins_must_modify_rec()`**（⚠️ **8.0.39 函数体里没有 `REC_INFO_DELETED_FLAG` 检查**——那是它在 `row_ins_sec_index_entry_by_modify()` 内部做的）：

```cpp
static inline bool row_ins_must_modify_rec(const btr_cur_t *cursor) {
  /* NOTE: ... Because node pointers on upper levels of the B-tree may match more
  to entry than to actual user records on the leaf level, we have to check if
  the candidate record is actually a user record. A clustered index node pointer
  contains index->n_unique first fields, and a secondary index node pointer
  contains all index fields. */
  return (cursor->low_match >= dict_index_get_n_unique_in_tree(cursor->index) &&
          !page_rec_is_infimum(btr_cur_get_rec(cursor)));
}
```

```cpp
  update = row_upd_build_sec_rec_difference_binary(rec, cursor->index, *offsets,
                                                   entry, heap);
  if (!rec_get_deleted_flag(rec, rec_offs_comp(*offsets))) {
    /* We should never insert in place of a record that has not been
    delete-marked. The only exception is when online CREATE INDEX copied the
    changes that we already made to the clustered index ... */
    ut_a(update->n_fields == 0);
    ut_a(!cursor->index->is_committed());
    return (DB_SUCCESS);
  }
  if (mode == BTR_MODIFY_LEAF) {
    err = btr_cur_optimistic_update(flags | BTR_KEEP_SYS_FLAG, cursor, ...);
    ...
```

> ★★ **`BTR_KEEP_SYS_FLAG` 是关键**：就地复用 delete-marked 记录时**保留 `DB_TRX_ID`/`DB_ROLL_PTR` 系统列不动**（只清 delete-mark + 改用户列）。

**★ 查重 `row_ins_scan_sec_index_for_duplicate()` 的锁矩阵**（⚠️ 真名，不存在 `row_ins_duplicate_error_in_sec`）：

```cpp
  n_unique = dict_index_get_n_unique(index);
  /* If the secondary index is unique, but one of the fields in the
  n_unique first fields is NULL, a unique key violation cannot occur,
  since we define NULL != NULL in this case */
  if (!index->nulls_equal) {
    for (ulint i = 0; i < n_unique; i++) {
      if (UNIV_SQL_NULL == dfield_get_len(dtuple_get_nth_field(entry, i))) {
        return DB_SUCCESS;                     // ★ 含 NULL → 不可能重复
      }
    }
  }
  n_fields_cmp = dtuple_get_n_fields_cmp(entry);
  dtuple_set_n_fields_cmp(entry, n_unique);    // ★★ 只比用户唯一列，不含追加 PK
  pcur.open(index, 0, entry, PAGE_CUR_GE, ..., mtr, ...);
  allow_duplicates = row_allow_duplicates(thr);   // REPLACE / ODKU
  const bool skip_gap_locks = index->table->skip_gap_locks();
  do {
    ulint lock_type = skip_gap_locks ? LOCK_REC_NOT_GAP : LOCK_ORDINARY;
    if (page_rec_is_infimum(rec)) continue;
    const bool is_supremum = page_rec_is_supremum(rec);
    const bool is_next = !is_supremum && (cmp_dtuple_rec(entry, rec, index, offsets) < 0);
    if (flags & BTR_NO_LOCKING_FLAG) {
      /* Set no locks when applying log in online table rebuild. */
    } else if (allow_duplicates) {
      err = row_ins_set_rec_lock(LOCK_X, lock_type, block, rec, index, offsets, thr);
    } else {
      if (skip_gap_locks) { ... lock_type = LOCK_REC_NOT_GAP; }
      else if (is_supremum)  { lock_type = LOCK_ORDINARY; }   // 等价 GAP
      else if (is_next)      { lock_type = LOCK_GAP; }        // 只锁 gap
      else                   { lock_type = LOCK_ORDINARY; }    // 等值记录 → next-key
      err = row_ins_set_rec_lock(LOCK_S, lock_type, block, rec, index, offsets, thr);
    }
    ...
    if (!is_next && !index->allow_duplicates) {
      if (row_ins_dupl_error_with_rec(rec, entry, index, offsets)) {
        err = DB_DUPLICATE_KEY;
        thr_get_trx(thr)->error_index = index;   // ★★ 带出"哪个索引"给 SQL 层报错
        goto end_scan;
      }
    } else {
      ut_a(is_next || index->allow_duplicates);
      goto end_scan;                            // ★ 遇到第一条严格大于的就停
    }
  } while (pcur.move_to_next(mtr));
end_scan:
  dtuple_set_n_fields_cmp(entry, n_fields_cmp);  // 还原
```

```cpp
static bool row_ins_dupl_error_with_rec(const rec_t *rec, const dtuple_t *entry,
                                        dict_index_t *index, const ulint *offsets) {
  n_unique = dict_index_get_n_unique(index);
  matched_fields = 0;
  entry->compare(rec, index, offsets, &matched_fields);
  if (matched_fields < n_unique) return false;
  /* In a unique secondary index we allow equal key values if they
  contain SQL NULLs */
  if (!index->is_clustered() && !index->nulls_equal) {
    for (i = 0; i < n_unique; i++) {
      if (dfield_is_null(dtuple_get_nth_field(entry, i))) return false;
    }
  }
  return rec_get_deleted_flag(rec, rec_offs_comp(offsets)) == 0;
  //  ★★ 只有未被 delete-mark 的等值记录才算真冲突
}
```

> ★★ 最后一行最易被误解：**delete-marked 记录不算重复**（它正等着被 purge 或复用）。

**★ `row_ins()` 主循环**（`row0ins.cc:3539`）——`trx->error_index` 的来源：

```cpp
  while (node->index != nullptr) {
    if (node->index->type != DICT_FTS) {
      err = row_ins_index_entry_step(node, thr);
      switch (err) {
        case DB_SUCCESS: break;
        case DB_DUPLICATE_KEY:
          thr_get_trx(thr)->error_state = DB_DUPLICATE_KEY;
          thr_get_trx(thr)->error_index = node->index;
          [[fallthrough]];
        default:
          return err;                    // ★ 立刻返回，不再插后续索引
      }
    }
    node->index = node->index->next();
    node->entry = UT_LIST_GET_NEXT(tuple_list, node->entry);
    /* Skip corrupted secondary index and its entry */
    while (node->index && node->index->is_corrupted()) {
      node->index = node->index->next();
      node->entry = UT_LIST_GET_NEXT(tuple_list, node->entry);
    }
  }
  thr_get_trx(thr)->error_index = nullptr;   // ★ 全部成功后清空
```

> ★ **多值索引是"一条 row 拆 N 个 entry"**，每个都完整走一遍两轮插入；`multi_val_pos` 是锁等待后**续做**的断点（`row_ins_sec_index_multi_value_entry`）。

#### change buffer 的精确决策点

**不存在 `row_allow_ibuf_insert` 函数**——row0ins 从不直接调 `ibuf_insert`。真实链路：

```
row_ins_sec_index_entry_low: search_mode |= BTR_INSERT
  → btr_cur_search_to_nth_level: BTR_INSERT → BTR_INSERT_OP（btr0cur.cc:796）
    → ibuf_should_try(index, btr_op != BTR_INSERT_OP)   // btr0cur.cc:1064
      → 叶页不在 buffer pool → ibuf_insert(IBUF_OP_INSERT, tuple, index, page_id, ...)
        → cursor->flag = BTR_CUR_INSERT_TO_IBUF
```

`ibuf_should_try`（`ibuf0ibuf.ic:116`）的前置过滤：change buffering 非 NONE、`ibuf->max_size != 0`、非 DD 表空间、非聚簇、非 spatial、**无降序列**、表未 quiesce、`(ignore_sec_unique || !is_unique)`、`force_recovery < SRV_FORCE_NO_IBUF_MERGE`。**唯一二级索引的 INSERT 不进 ibuf**（`ignore_sec_unique=0` 且 is_unique → 排除）；但 delete-mark 操作传 `ignore_sec_unique=1` → **唯一索引的删除项可以进 ibuf**（delete-mark 不破坏唯一性）。14 条否决条件见 [`ibuf.md`](ibuf.md)。

#### 更新：`row_upd_changes_ord_field_binary` 与删旧插新

**主流程顺序**（`row0upd.cc:3239-3279`）：**先聚簇（`row_upd_clust_step`）、后逐个二级（`row_upd_sec_step`）**；`UPD_NODE_NO_ORD_CHANGE`（无排序字段变化）直接跳过所有二级。**rollback 严格逆序**（先摘二级引用、再动聚簇本体，`row0uins.cc:494-501`）——保证任何时刻二级记录都能回表找到聚簇记录（崩溃中间态一致）。

**跳过优化**（`row_upd_sec_step`，2520-2547）：`node->state != UPDATE_ALL_SEC` 且 `row_upd_changes_ord_field_binary == false` → 该二级索引完全不动。

**`row_upd_changes_ord_field_binary`**（`row0upd.cc:1485-1707`）逐段：

```cpp
n_unique = dict_index_get_n_unique(index);   // 只遍历唯一前缀（含追加 PK）
for (i = 0; i < n_unique; i++) {
  ...
  if (col->is_virtual()) upd_field = upd_get_field_by_field_no(update, v_col->v_pos, true);
  else upd_field = upd_get_field_by_field_no(update, dict_col_get_clust_pos(col, clust_index), false);
  if (upd_field == nullptr) continue;        // 不在 update vector → 未变
  if (row == nullptr) return (true);         // 无旧行（undo 路径）→ 视为已变
  if (!dfield_datas_are_binary_equal(dfield, &upd_field->new_val, ind_field->prefix_len)) {
    changes = true;
    if (non_mv_upd == nullptr || !dfield_is_multi_value(dfield)) { *non_mv_upd = true; break; }
  }
}
```

边界：spatial 首字段比 MBR 而非几何数据（`mbr_equal_cmp`）；前缀索引 + 外置列 `row_ext_lookup` 剥 `BTR_EXTERN_FIELD_REF_SIZE` 后按 `prefix_len` 比；**`row == nullptr`（无旧行）视为已变**——这是 undo 路径的安全兜底。

**更新本体**（`row_upd_sec_index_entry_low`，2224-2460）：`row_build_index_entry(node->row, ...)` 构造旧 entry → `row_search_index_entry` 定位 → **`btr_cur_del_mark_set_sec_rec` 旧项 delete-mark（不释放锁！）** → 非删除场景 `row_build_index_entry(node->upd_row, ...)` 构造新 entry → `row_ins_sec_index_entry` 插新项。三个细节：

- **delete-mark 不释放记录锁**——旧项上的 X 锁正是防并发插入相同唯一键的竞态手段，锁随事务 commit 回收；
- **`BTR_DELETE_MARK` 模式**允许删除项进 change buffer（此时唯一索引也放行）；
- **唯一索引更新无专门 flag**——新项的唯一性由插入侧的查重承担，且 `row_ins_must_modify_rec` 会复用/反标记那条 delete-marked 旧项（注释 3010-3012）。

**★ `row_upd_sec_index_entry_low()` 的"删旧 + 插新"两步**（`row0upd.cc:2224`）：

```cpp
  /* Build old index entry */
  entry = row_build_index_entry(node->row, node->ext, index, heap);
  ...
  /* We can only buffer delete-mark operations if there are no foreign key
  constraints referring to the index. Change buffering is disabled for
  temporary tables and spatial index. */
  mode = (referenced || index->table->is_temporary() ||
          dict_index_is_spatial(index))
             ? BTR_MODIFY_LEAF | BTR_ALREADY_S_LATCHED
             : BTR_MODIFY_LEAF | BTR_ALREADY_S_LATCHED | BTR_DELETE_MARK;
  //  ★★ BTR_DELETE_MARK ⇒ 删除项可以进 change buffer（此时唯一索引也放行）

  search_result = row_search_index_entry(index, entry, mode, &pcur, &mtr);
  ...
  case ROW_FOUND:
    /* Delete mark the old index record; it can already be delete marked if
    we return after a lock wait in row_ins_sec_index_entry() below */
    if (!rec_get_deleted_flag(rec, dict_table_is_comp(index->table))) {
      err = btr_cur_del_mark_set_sec_rec(flags, btr_cur, true, thr, &mtr);
      if (err != DB_SUCCESS) break;
    }
    if (referenced) { err = row_upd_check_references_constraints(...); }
    break;
  ...
  pcur.close();
  mtr_commit(&mtr);

  if (node->is_delete || err != DB_SUCCESS) goto func_exit;   // ★ DELETE 只删不插

  mem_heap_empty(heap);
  /* Build a new index entry */
  entry = row_build_index_entry(node->upd_row, node->upd_ext, index, heap);
  ut_a(entry);
  /* Insert new index entry */
  err = row_ins_sec_index_entry(index, entry, thr, false);    // ★ 再走完整插入流程
```

**★★ 更新跳过的状态机**（`row_upd_sec_step`，`row0upd.cc:2520`）：

```cpp
[[nodiscard]] static dberr_t row_upd_sec_step(upd_node_t *node, que_thr_t *thr) {
  ut_ad((node->state == UPD_NODE_UPDATE_ALL_SEC) ||
        (node->state == UPD_NODE_UPDATE_SOME_SEC));
  ut_ad(!node->index->is_clustered());

  bool non_mv_upd = false;

  if (node->state == UPD_NODE_UPDATE_ALL_SEC ||
      row_upd_changes_ord_field_binary(
          node->index, node->update, thr, node->row, node->ext,
          (node->index->is_multi_value() ? &non_mv_upd : nullptr))) {
    if (node->index->is_multi_value()) { ... }
    else { return (row_upd_sec_index_entry(node, thr)); }
  }

  return (DB_SUCCESS);        // ★★ 没变 → 该二级索引完全不动
}
```

```cpp
constexpr uint32_t UPD_NODE_SET_IX_LOCK      = 1;
constexpr uint32_t UPD_NODE_UPDATE_CLUSTERED = 2;
constexpr uint32_t UPD_NODE_INSERT_CLUSTERED = 3;
/** an ordering field of the clustered index record was changed, or this is a
 delete operation: should update all the secondary index records */
constexpr uint32_t UPD_NODE_UPDATE_ALL_SEC   = 5;
/** secondary index entries should be looked at and updated if an ordering field
 changed */
constexpr uint32_t UPD_NODE_UPDATE_SOME_SEC  = 6;

/** no secondary index record will be changed ... */
constexpr uint32_t UPD_NODE_NO_ORD_CHANGE = 1;
/** no record field size will be changed in the update */
constexpr uint32_t UPD_NODE_NO_SIZE_CHANGE = 2;
```

| 起点 | 条件 | 落点 |
|---|---|---|
| `SET_IX_LOCK` | 上完表 IX 锁 | `UPDATE_CLUSTERED` |
| `UPDATE_CLUSTERED` | `is_delete`（真 DELETE DML） | `UPD_NODE_UPDATE_ALL_SEC` |
| `UPDATE_CLUSTERED` | 聚簇有序列变了 → `row_upd_clust_rec_by_insert()` | `UPD_NODE_UPDATE_ALL_SEC` |
| `UPDATE_CLUSTERED` | 聚簇有序列没变 → `row_upd_clust_rec()` | `UPD_NODE_UPDATE_SOME_SEC` |
| 二级循环 | `UPD_NODE_UPDATE_ALL_SEC` | 逐索引**必做** |
| 二级循环 | `UPD_NODE_UPDATE_SOME_SEC` | 逐索引问 `row_upd_changes_ord_field_binary()`，**false 跳过** |

> ★ `row_upd()` 里还有第二个跳过出口：`node->index == nullptr || (!node->is_delete && (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE))` → 直接 `return DB_SUCCESS`。

**★★ `row_upd_changes_ord_field_binary` 是"二进制比较"不是"等值比较"**：

```cpp
  n_unique = dict_index_get_n_unique(index);      // ★ 只遍历用户唯一列
  clust_index = index->table->first_index();
  for (i = 0; i < n_unique; i++) {
    ind_field = index->get_field(i);
    col = ind_field->col;
    col_no = dict_col_get_no(col);

    if (col->is_virtual()) {
      vcol = reinterpret_cast<const dict_v_col_t *>(col);
      upd_field = upd_get_field_by_field_no(update, vcol->v_pos, true);
    } else {
      upd_field = upd_get_field_by_field_no(
          update, dict_col_get_clust_pos(col, clust_index), false);
    }
    if (upd_field == nullptr) continue;           // 不在 update vector → 未变
    if (row == nullptr) {
      ut_ad(ext == nullptr);
      return (true);                              // ★★ 无旧行（undo 路径）→ 视为已变（安全兜底）
    }
    ...
    if (!dfield_datas_are_binary_equal(dfield, &upd_field->new_val,
                                       ind_field->prefix_len)) {
      changes = true;
      if (non_mv_upd == nullptr || !dfield_is_multi_value(dfield)) {
        if (non_mv_upd != nullptr) *non_mv_upd = true;
        break;
      }
    }
  }
```

> ★★ **`'abc' → 'aBc'` 在 collation 下等价但二进制不同 → 仍返回 true → 仍走 delete-mark + insert**。这是"更新代价"的一个隐藏来源。
> ★ spatial 首字段比 MBR 而非几何数据（`mbr_equal_cmp`），且 `srid` 不同直接 `return true`。

**★ delete-mark 是原地改 info bits，redo 只有 3 字节**：

```cpp
dberr_t btr_cur_del_mark_set_sec_rec(ulint flags, btr_cur_t *cursor, bool val,
                                     que_thr_t *thr, mtr_t *mtr) {
  block = btr_cur_get_block(cursor);
  rec = btr_cur_get_rec(cursor);

  err = lock_sec_rec_modify_check_and_lock(flags, block, rec, cursor->index,
                                           thr, mtr);
  if (err != DB_SUCCESS) return (err);
  ...
  /* We do not need to reserve search latch, as the delete-mark flag is being
  updated in place and the adaptive hash index does not depend on it. */
  btr_rec_set_deleted_flag(rec, buf_block_get_page_zip(block), val);
  btr_cur_del_mark_set_sec_rec_log(rec, val, mtr);
  return (DB_SUCCESS);
}
```

```cpp
static inline void btr_cur_del_mark_set_sec_rec_log(rec_t *rec, bool val,
                                                    mtr_t *mtr) {
  byte *log_ptr = nullptr;
  if (!mlog_open(mtr, 11 + 1 + 2, log_ptr)) return;
  log_ptr = mlog_write_initial_log_record_fast(rec, MLOG_REC_SEC_DELETE_MARK,
                                               log_ptr, mtr);
  mach_write_to_1(log_ptr, val);  log_ptr++;
  mach_write_to_2(log_ptr, page_offset(rec));  log_ptr += 2;
  mlog_close(mtr, log_ptr);
}
```

> ★ **二级索引的 delete-mark 不写 undo**——回滚靠"再打一次"（`val=true` / `val=false` 两个方向）。
> ⚠️ **`btr_cur_del_mark_set_sec_rec_and_save()` 不存在**。
> ★★ **purge 里从不调用它** —— purge 用 `btr_cur_optimistic_delete()` / `pessimistic_delete()` 直接物理删；`btr_cur_del_mark_set_sec_rec()` 只由 **DML / DDL / rollback** 写入，purge 只用 `rec_get_deleted_flag()` 做**校验**（断言"只能 purge 已打标记的项"）。

**★★ rollback 是"先二级后聚簇"，与写路径相反**（这是铁证）：

```cpp
  node->index = node->table->first_index();
  ut_ad(node->index->is_clustered());
  /* Skip the clustered index (the first index) */
  node->index = node->index->next();          // ★★ 跳过聚簇

  err = row_undo_ins_remove_sec_rec(node, thr);
  if (err == DB_SUCCESS) {
    err = row_undo_ins_remove_clust_rec(node);   // ★ 最后才动聚簇本体
  }
```

```cpp
  node->index = node->table->first_index();
  ut_ad(node->index->is_clustered());
  /* Skip the clustered index (the first index) */
  node->index = node->index->next();          // row0umod.cc 同样
  ...
  if (err == DB_SUCCESS) {
    err = row_undo_mod_clust(node, thr);
  }
```

> ★ insert 的 undo 是**物理删除**（`btr_cur_optimistic_delete`），不是 delete-mark——因为 insert 的记录不可能被别的事务看到。
> ★ update 的 undo 要问 `row_vers_old_has_index_entry(false, ...)`：**旧版本是否还需要这条二级项** → 需要则只 delete-mark，不需要则物理删。
> ⚠️ **`row_undo_upd_del_mark_or_remove_sec()` 不存在**（真名 `row_undo_mod_del_mark_or_remove_sec()`）；且 **update/delete 的 undo 在 `row0umod.cc`，不在 `row0uins.cc`**（后者只管 insert undo）。

#### 删除：两阶段

第一阶段 `btr_cur_del_mark_set_sec_rec` 只打 delete-mark（逻辑删除，仍占槽位）；第二阶段由 purge 物理删除（见回收节）。为什么必须两阶段、`row_vers_old_has_index_entry` 的版本链回溯、delete-marked 项对查询/锁的影响——详见 [`transaction.md`](transaction.md)「delete-mark 两阶段」。

---

### 读路径

#### `row_search_mvcc` 二级分支全景

（标签名与行号确认于 `row0sel.cc`；`row_search_mvcc` 主链见 [`../innodb/row_search.md`](../innodb/row_search.md)）

```
PHASE 3：MVCC 粗筛
  consistent_read && !index->is_clustered()
  && !lock_sec_rec_cons_read_sees(rec, index, view)     // PAGE_MAX_TRX_ID 不可见
    → cons_read_requires_clust_rec = true
PHASE 5：must_get_clust || cons_read_requires_clust_rec → row_sel_get_clust_rec() 回表
主循环（二级 + 非锁读分支）：
  5394 lock_sec_rec_cons_read_sees 失败
    → row_search_idx_cond_check()                        // ★ ICP 判当前版本，省回表
        ICP_NO_MATCH     → goto next_rec
        ICP_OUT_OF_RANGE → DB_RECORD_NOT_FOUND
        ICP_MATCH        → goto requires_clust_rec
  5431 rec_get_deleted_flag(rec) → try_unlock(true); goto next_rec
  5461 row_search_idx_cond_check()（全部 rec 的 ICP 终检）
  5477 index != clust_index && need_to_access_clustered
requires_clust_rec:
  5498 row_sel_get_clust_rec_for_mysql(prebuilt, index, rec, ...)   // 回表 + 版本构建
        clust_rec == nullptr → goto next_rec（该版本在 read view 不存在）
  5528 clust_rec delete-marked → try_unlock; goto next_rec
  5550 idx_cond → row_sel_store_mysql_rec(..., clust_index, prebuilt->index, ...)
                 // ICP 列在二级可能是前缀/大小写不同 → 用聚簇值重建
```

关键点：**ICP 在二级索引上判的是当前版本**（保守剪枝——匹配了也可能回表被 old_vers 否决）；`lock_sec_rec_cons_read_sees` 的 `PAGE_MAX_TRX_ID` 粗筛见 [`transaction.md`](transaction.md)。

#### 双向校验：`row_sel_sec_rec_is_for_clust_rec`

回表拿到聚簇记录后，还要校验"这条二级索引项是否真的对应这条聚簇记录"（`row0sel.cc:179-346`）：

```cpp
static dberr_t row_sel_sec_rec_is_for_clust_rec(
    const rec_t *sec_rec, dict_index_t *sec_index, const rec_t *clust_rec,
    dict_index_t *clust_index, que_thr_t *thr, bool &is_equal) {
  if (rec_get_deleted_flag(clust_rec, ...)) { is_equal = false; return DB_SUCCESS; }
  n = dict_index_get_n_ordering_defined_by_user(sec_index);   // ★ 只比用户定义列
  for (i = 0; i < n; i++) {
    if (col->is_virtual()) {
      row = row_build(ROW_COPY_POINTERS, clust_index, clust_rec, ...);
      vfield = innobase_get_computed_value(row, v_col, ...);   // 虚拟列从聚簇重算
    } else {
      clust_pos = dict_col_get_clust_pos(col, clust_index);
      clust_field = rec_get_nth_field_instant(clust_rec, clust_offs, clust_pos, ...);
    }
    sec_field = rec_get_nth_field(nullptr, sec_rec, sec_offs, i, &sec_len);
    if (col->is_multi_value()) {
      if (!is_multi_value_clust_and_sec_equal(...)) { is_equal = false; goto func_exit; }
    } else if (0 != cmp_data_data(col->mtype, col->prtype, true, clust_field, len,
                                  sec_field, sec_len)) {          // ASC/DESC 不影响相等性
      is_equal = false; goto func_exit;
    }
  }
}
```

- 聚簇记录 delete-mark → 快速失败；**只比用户定义列**（不含追加 PK——追加 PK 的对应性由"回表定位"本身保证：二级 rec 里的 PK 构造 `clust_ref` 搜索聚簇树，旧版引用会落到不同记录）；
- 虚拟列从聚簇行**重算**；前缀截断到 `prefix_len` 再比；spatial 比 MBR；**多值列**交给 `is_multi_value_clust_and_sec_equal`（`data0data.cc:828`，聚簇侧 JSON 数组展开逐值比较）；
- 调用条件（3352-3370）：只有 `old_vers || 读未提交 || spatial || sec_rec delete-marked` 才校验——**只有回表拿到旧版本/二级项被标记时才可能"二级项与聚簇版本不对应"**。

**★★ 完整源码**（`row0sel.cc:179`）——⚠️ 源码注释明确写着 **"the comparison is NOT done as a binary comparison, but character fields are compared with collation!"**：

```cpp
static dberr_t row_sel_sec_rec_is_for_clust_rec(
    const rec_t *sec_rec, dict_index_t *sec_index, const rec_t *clust_rec,
    dict_index_t *clust_index, que_thr_t *thr, bool &is_equal) {
  is_equal = true;
  rec_offs_init(clust_offsets_);
  rec_offs_init(sec_offsets_);

  if (rec_get_deleted_flag(clust_rec, dict_table_is_comp(clust_index->table))) {
    /* The clustered index record is delete-marked; it is not visible in the
    read view. Besides, if there are any externally stored columns, some of
    them may have already been purged. */
    is_equal = false;
    return (DB_SUCCESS);                      // ★ 快速失败
  }

  clust_offs = rec_get_offsets(clust_rec, clust_index, clust_offs, ULINT_UNDEFINED, ...);
  sec_offs = rec_get_offsets(sec_rec, sec_index, sec_offs, ULINT_UNDEFINED, ...);

  n = dict_index_get_n_ordering_defined_by_user(sec_index);   // ★★ 只比用户定义列

  for (i = 0; i < n; i++) {
    ifield = sec_index->get_field(i);
    col = ifield->col;

    /* For virtual column, its value will need to be reconstructed from base
    column in cluster index */
    if (col->is_virtual()) {
      row = row_build(ROW_COPY_POINTERS, clust_index, clust_rec, clust_offs,
                      nullptr, nullptr, nullptr, &ext, heap);
      vfield = innobase_get_computed_value(row, v_col, clust_index, &heap, heap,
                                           nullptr, thr_get_trx(thr)->mysql_thd,
                                           thr->prebuilt->m_mysql_table,
                                           nullptr, nullptr, nullptr);
      if (vfield == nullptr) {
        /* This may happen e.g. when this statement is executed in
        read-uncommited isolation and value (like json function) depends on
        an externally stored lob (like json) which was not written yet. */
        err = DB_COMPUTE_VALUE_FAILED;
        goto func_exit;
      }
      clust_len = vfield->len;
      clust_field = static_cast<byte *>(vfield->data);
    } else {
      clust_pos = dict_col_get_clust_pos(col, clust_index);
      clust_field = rec_get_nth_field_instant(clust_rec, clust_offs, clust_pos,
                                              clust_index, &clust_len);
    }

    sec_field = rec_get_nth_field(nullptr, sec_rec, sec_offs, i, &sec_len);
    len = clust_len;

    if (ifield->prefix_len > 0 && len != UNIV_SQL_NULL && sec_len != UNIV_SQL_NULL &&
        !col->is_virtual()) {
      if (rec_offs_nth_extern(clust_index, clust_offs, clust_pos)) {
        len -= BTR_EXTERN_FIELD_REF_SIZE;
      }
      len = dtype_get_at_most_n_mbchars(col->prtype, col->mbminmaxlen,
                                        ifield->prefix_len, len, (char *)clust_field);
      //  ★ 前缀索引：按 prefix_len 截断到"最多 N 个字符"
      if (rec_offs_nth_extern(...) &&
          (len < sec_len || (dict_table_has_atomic_blobs(sec_index->table) && sec_len == 0))) {
        if (!row_sel_sec_rec_is_for_blob(...)) { is_equal = false; goto func_exit; }
        continue;
      }
    }

    if (dict_index_is_spatial(sec_index) && i == 0) {
      //  ★ spatial：从几何数据重算 MBR 再比
      if (rec_offs_nth_extern(clust_index, clust_offs, clust_pos)) {
        dptr = lob::btr_copy_externally_stored_field(...);
      }
      get_mbr_from_store(sec_index->rtr_srs.get(), dptr, ..., &tmp_mbr, nullptr);
      rtr_read_mbr(sec_field, &sec_mbr);
      if (!mbr_equal_cmp(sec_index->rtr_srs.get(), &sec_mbr, &tmp_mbr)) {
        is_equal = false; goto func_exit;
      }
    } else if (col->is_multi_value()) {
      //  ★ 多值：语义是"数组包含"，不是"等于"
      if (!is_multi_value_clust_and_sec_equal(clust_field, clust_len, sec_field,
                                              sec_len, col)) {
        is_equal = false; goto func_exit;
      }
    } else {
      /* We are testing for equality; ASC/DESC does not matter */
      if (0 != cmp_data_data(col->mtype, col->prtype, true, clust_field, len,
                             sec_field, sec_len)) {
        is_equal = false; goto func_exit;
      }
    }
  }
func_exit:
  if (UNIV_LIKELY_NULL(heap)) mem_heap_free(heap);
  return (err);
}
```

**★ 多值列的"包含"语义**（`data0data.cc:828`）：

```cpp
bool is_multi_value_clust_and_sec_equal(const byte *clust_field, uint64_t clust_len,
                                        const byte *sec_field, uint64_t sec_len,
                                        const dict_col_t *col) {
  if (clust_len == UNIV_SQL_NULL) {
    return (sec_len == UNIV_SQL_NULL);        // 都 NULL 才算等
  } else if (clust_len == UNIV_NO_INDEX_VALUE) {
    return (false);                           // 空数组不可能匹配任何 sec 值
  }
  ut_ad(clust_len == UNIV_MULTI_VALUE_ARRAY_MARKER);
  const multi_value_data *multi_value =
      reinterpret_cast<const multi_value_data *>(clust_field);
  return (multi_value->has(col->mtype, col->prtype, sec_field, sec_len));
  //  ★★ 线性扫数组，逐值 cmp_data_data
}
```

**★ 回表的真名是 functor，不是函数**（⚠️ 常见写法 `row_sel_get_clust_rec_for_mysql()` 其实是 `row0sel.cc:4472` 声明的**同名对象**）：

```cpp
/** Helper class to cache clust_rec and old_ver */
class Row_sel_get_clust_rec_for_mysql {
  const rec_t *cached_clust_rec;
  rec_t *cached_old_vers;
 public:
  dberr_t operator()(row_prebuilt_t *prebuilt, dict_index_t *sec_index,
                     const rec_t *rec, que_thr_t *thr, const rec_t **out_rec,
                     ulint **offsets, mem_heap_t **offset_heap,
                     const dtuple_t **vrow, mtr_t *mtr,
                     lob::undo_vers_t *lob_undo);
};
```

```cpp
  // ① 用 sec rec 里的 PK 列构造 clust_ref 搜索元组
  row_build_row_ref_in_tuple(prebuilt->clust_ref, rec, sec_index, *offsets);
  clust_index = sec_index->table->first_index();
  prebuilt->clust_pcur->open_no_init(clust_index, prebuilt->clust_ref,
                                     PAGE_CUR_LE, BTR_SEARCH_LEAF, 0, mtr, ...);
  clust_rec = prebuilt->clust_pcur->get_rec();

  // ② low_match 不足 → 回表失败（合法的罕见场景，源码打了长注释）
  if (!page_rec_is_user_rec(clust_rec) ||
      prebuilt->clust_pcur->get_low_match() < dict_index_get_n_unique(clust_index)) {
    ...
    clust_rec = nullptr; err = DB_SUCCESS; goto func_exit;
  }
```

```cpp
  // ③ 锁定读：回表后给 clust_rec 加 LOCK_REC_NOT_GAP
  if (prebuilt->select_lock_type != LOCK_NONE) {
    /* Try to place a lock on the index record; we are searching the clust rec
    with a unique condition, hence we set a LOCK_REC_NOT_GAP type lock */
    err = lock_clust_rec_read_check_and_lock(
        lock_duration_t::REGULAR, prebuilt->clust_pcur->get_block(), clust_rec,
        clust_index, *offsets, prebuilt->select_mode,
        static_cast<lock_mode>(prebuilt->select_lock_type), LOCK_REC_NOT_GAP, thr);
    ...
  } else {
    // ④ 一致性读：必要时回溯旧版本
    if (trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
        !lock_clust_rec_cons_read_sees(clust_rec, clust_index, *offsets,
                                       trx_get_read_view(trx))) {
      err = row_sel_build_prev_vers_for_mysql(...);
      ...
      clust_rec = old_vers;
    }
    // ⑤★★ 双向校验的触发条件
    if (clust_rec &&
        (old_vers || trx->isolation_level <= TRX_ISO_READ_UNCOMMITTED ||
         dict_index_is_spatial(sec_index) ||
         rec_get_deleted_flag(rec, dict_table_is_comp(sec_index->table)))) {
      bool rec_equal;
      err = row_sel_sec_rec_is_for_clust_rec(rec, sec_index, clust_rec,
                                             clust_index, thr, rec_equal);
      if (err != DB_SUCCESS) goto err_exit;
      else if (!rec_equal) clust_rec = nullptr;    // ★ 不对应 → 丢弃这一行
    }
  }
```

> ★ **另有一个服务于查询图路径的 `row_sel_get_clust_rec()`**（`row0sel.cc:797`），与上面**不是同一个东西**。
> ⚠️ `row_search_mvcc()` 里**没有** `cons_read_requires_clust_rec` 变量——那是 `row_sel()`（查询图路径）的；`row_search_mvcc` 的对应物是 `prebuilt->need_to_access_clustered` + `requires_clust_rec:` 标签。

> 另有 `row_sel_try_search_shortcut_for_mysql`（`row0sel.cc:3740` `ut_ad(index->is_clustered())`）——**无二级索引路径**（快捷搜索仅限聚簇）。

#### 锁定读与二级索引上的锁

锁定读（`FOR UPDATE`/`LOCK IN SHARE MODE`）走二级索引时，加锁行为与一致性读完全不同（分界是 `prebuilt->select_lock_type != LOCK_NONE`；gap 开关是局部变量 `set_also_gap_locks`，RC/DD 表时置 false，`row0sel.cc:4807-4817`）：

**① gap/next-key 锁加在二级索引上，不在聚簇上。** `sel_set_rec_lock` 按"当前扫描索引"分派——走二级扫描时对**二级记录**调 `lock_sec_rec_read_check_and_lock`（`row0sel.cc:1138-1180`）。主循环的锁类型选择（5246-5276）：`row_compare_row_to_range` 判定——命中且 gap 可能相交 → `LOCK_ORDINARY`（next-key）；唯一搜索命中未删记录 → `LOCK_REC_NOT_GAP`；未命中但 gap 相交 → `LOCK_GAP`。另外三个 gap 调用点（降序扫描先锁 next_rec、页 supremum 锁 ORDINARY、精确查找未命中锁 GAP）也全在二级索引上。

**② 两级加锁：回表后再给聚簇记录加 `LOCK_REC_NOT_GAP`**（`row0sel.cc:3280-3289`）：

```cpp
if (prebuilt->select_lock_type != LOCK_NONE) {
  /* Try to place a lock on the index record; we are searching
  the clust rec with a unique condition, hence
  we set a LOCK_REC_NOT_GAP type lock */
  err = lock_clust_rec_read_check_and_lock(..., clust_rec, clust_index, ...,
      prebuilt->select_lock_type, LOCK_REC_NOT_GAP, thr);
}
```

- 锁定读 = **二级 `X/S + (ORDINARY/REC_NOT_GAP/GAP)` + 聚簇 `X/S + REC_NOT_GAP`** 两把完整记录锁；
- **covering 二级索引的 `FOR UPDATE` 不回表 → 聚簇不加锁**（`need_to_access_clustered` 由"SELECT 需要聚簇列/spatial"决定，与锁定读无关——全仓无"锁定读强制回表"的代码）；
- **锁定读不看 read view**：`lock_sec_rec_cons_read_sees` 只在非锁定读分支被调用（5394-5395 的前置是 `select_lock_type == LOCK_NONE`）——锁定读读当前最新版本，用锁保护；
- **RC 下所有 lock_type 退化为 `LOCK_REC_NOT_GAP`**（`row_compare_row_to_range` 里 `trx->skip_gap_locks()` 短路 gap 判定）。

**③ 隐式锁的二级索引悖论**：二级记录没有 `DB_TRX_ID`，**放不了隐式锁**——插入路径（`row_ins_sec_index_entry_low`）除唯一查重 S 锁外**不给新记录建任何锁**，只做 `page_update_max_trx_id`。那并发事务怎么发现"这条二级项被未提交事务占有"？两步发现（`lock_sec_rec_some_has_impl`，`lock0lock.cc:1034-1069`）：

```cpp
max_trx_id = page_get_max_trx_id(page);
/* Some transaction may have an implicit x-lock on the record only
   if the max trx id for the page >= min trx id for the trx list ... */
if (!recv_recovery_is_on() && !can_older_trx_be_still_active(max_trx_id)) {
  trx = nullptr;                                    // ★ 快筛：页级 max_trx_id 不可能活跃
} else {
  trx = row_vers_impl_x_locked(rec, index, offsets);  // 回表：读聚簇 trx_id 判断
}
```

- **快筛**：二级页头的 `PAGE_MAX_TRX_ID`（最近一次改本页的 trx_id）不可能仍活跃 → 直接返回 nullptr（**无锁读的常态路径，不用回表**）；recovery 期间 max_trx_id 不可信，跳过快筛；

**★ 粗筛 vs 聚簇行级判定的对照**（这是"二级索引没有 trx_id"的直接后果）：

```cpp
bool lock_sec_rec_cons_read_sees(const rec_t *rec, const dict_index_t *index,
                                 const ReadView *view) {
  ut_ad(page_rec_is_user_rec(rec));
  if (recv_recovery_is_on()) return (false);
  else if (index->table->is_temporary()) return (true);   // 临时表不需 MVCC

  trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));
  //  ★★ 二级索引页没有 trx_id 列，只能用页头一个 8 字节值做"整页级"粗筛
  ut_ad(max_trx_id > 0);
  return (view->sees(max_trx_id));
}
```

```cpp
bool lock_clust_rec_cons_read_sees(const rec_t *rec, dict_index_t *index,
                                   const ulint *offsets, ReadView *view) {
  ut_ad(index->is_clustered());
  if (srv_read_only_mode || index->table->is_temporary()) return (true);
  trx_id_t trx_id = row_get_rec_trx_id(rec, index, offsets);
  //  ★★ 对照：聚簇有行级 DB_TRX_ID，可精确到"行"
  return (view->changes_visible(trx_id, index->table->name));
}
```

> ★★ **粗筛假阴性很常见**（页上任一行的修改都会抬高 `max_trx_id`，于是整页都被判"不确定"）→ 转 `requires_clust_rec` 回表。这是二级索引读路径的主要额外开销。

**★ 隐式锁两步发现的完整源码**（⚠️ 返回 `trx_t*` 不是 trx_id）：

```cpp
static trx_t *lock_sec_rec_some_has_impl(const rec_t *rec, dict_index_t *index,
                                         const ulint *offsets) {
  trx_t *trx;
  trx_id_t max_trx_id;
  const page_t *page = page_align(rec);
  ...
  max_trx_id = page_get_max_trx_id(page);

  /* Some transaction may have an implicit x-lock on the record only
  if the max trx id for the page >= min trx id for the trx list, or
  database recovery is running. We do not write the changes of a page
  max trx id to the log, and therefore during recovery, this value
  for a page may be incorrect. */
  if (!recv_recovery_is_on() && !can_older_trx_be_still_active(max_trx_id)) {
    trx = nullptr;                                  // ★① 快筛：省掉回表
  } else if (!lock_check_trx_id_sanity(max_trx_id, rec, index, offsets)) {
    trx = nullptr;                                  // 页损坏 → 避免 crash
  } else {
    trx = row_vers_impl_x_locked(rec, index, offsets);   // ★② 精判：回表 + 走 undo
  }
  return (trx);
}
```

```cpp
trx_t *row_vers_impl_x_locked(const rec_t *rec, const dict_index_t *index,
                              const ulint *offsets) {
  mtr_start(&mtr);
  clust_rec = row_get_clust_rec(BTR_SEARCH_LEAF, rec, index, &clust_index, &mtr);
  //  ★ 用二级 rec 内嵌的 PK 回表
  if (!clust_rec) {
    /* In a rare case it is possible that no clust rec is found ... there cannot
    be any implicit lock on the secondary index record, because an active
    transaction which has modified the secondary index record has also modified
    the clustered index record. ... */
    trx = nullptr;
  } else {
    trx = row_vers_impl_x_locked_low(clust_rec, clust_index, rec, index,
                                     offsets, &mtr);
  }
  mtr_commit(&mtr);
  return (trx);
}
```

```cpp
  trx_id = row_get_rec_trx_id(clust_rec, clust_index, clust_offsets);
  //  ★ 从 clust_rec 读出的唯一信息就是 DB_TRX_ID
  trx_t *trx = trx_rw_is_active(trx_id, true);
  ...
  bool looking_for_match = rec_get_deleted_flag(sec_rec, comp);
  //  ★ delete-mark 的 S 找"从 match 变 not-match"的版本；非 delete 的找"从 not-match 变 match"
  if (!row_vers_find_matching(looking_for_match, clust_index, clust_rec,
                              clust_offsets, sec_index, sec_rec, sec_offsets,
                              comp, trx_id, mtr, heap)) {
    trx_release_reference(trx); trx = nullptr;
  }
```

> ★ 对照 `lock_clust_rec_some_has_impl()`（`lock0priv.ic:52`）**只有一行**：`return row_get_rec_trx_id(rec, index, offsets);`——聚簇一步到位，不需要回表/走 undo。

**★ 物化**（`lock0lock.cc:5541`）——两种索引走的分支不同：

```cpp
void lock_rec_convert_impl_to_expl(const buf_block_t *block, const rec_t *rec,
                                   dict_index_t *index, const ulint *offsets) {
  if (index->is_clustered()) {
    trx_id_t trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);
    trx = trx_rw_is_active(trx_id, true);
  } else {
    ut_ad(!dict_index_is_online_ddl(index));
    trx = lock_sec_rec_some_has_impl(rec, index, offsets);   // ★★ 两步
    if (trx) { ut_ad(!lock_rec_other_trx_holds_expl(LOCK_S | LOCK_REC_NOT_GAP, trx, rec, block)); }
  }
  if (trx != nullptr) {
    ulint heap_no = page_rec_get_heap_no(rec);
    ut_ad(trx_is_referenced(trx));
    /* If the transaction is still active and has no explicit x-lock set on the
    record, set one for it. trx cannot be committed until the ref count is zero. */
    lock_rec_convert_impl_to_expl_for_trx(block, rec, index, offsets, trx, heap_no);
  }
}
```

**★ `need_to_access_clustered` 由"需要的列"决定，与锁定读无关**（这是"covering 索引 FOR UPDATE 不回表"的证据）：

```cpp
  if (!index->is_clustered() && templ->rec_field_no == ULINT_UNDEFINED) {
    prebuilt->need_to_access_clustered = true;   // ★ 列在二级索引里找不到 → 必须回表
  }
  /* For spatial index, we need to access cluster index. */
  if (dict_index_is_spatial(index)) {
    prebuilt->need_to_access_clustered = true;
  }
```

```cpp
  m_prebuilt->need_to_access_clustered = (index == clust_index);   // 初始化
  ...
  if (prebuilt->table->skip_gap_locks() ||
      (trx->skip_gap_locks() && prebuilt->select_lock_type != LOCK_NONE && ...)) {
    set_also_gap_locks = false;                                    // RC / DD 表
  }
```

**★ gap 锁三分支**（`row_compare_row_to_range` → `lock_type`）：

```cpp
    ulint lock_type;
    if (row_to_range_relation.row_can_be_in_range) {
      if (row_to_range_relation.gap_can_intersect_range) lock_type = LOCK_ORDINARY;
      else                                              lock_type = LOCK_REC_NOT_GAP;
    } else {
      if (row_to_range_relation.gap_can_intersect_range) lock_type = LOCK_GAP;
      else { err = DB_RECORD_NOT_FOUND; goto normal_return; }
    }
    err = sel_set_rec_lock(pcur, rec, index, offsets, ..., lock_type, thr, &mtr);
```

```cpp
  if (index->is_clustered()) {
    err = lock_clust_rec_read_check_and_lock(...);
  } else {
    if (dict_index_is_spatial(index)) { ... sel_set_rtr_rec_lock(...); }
    else { err = lock_sec_rec_read_check_and_lock(...); }
    //  ★★ 按"当前正在扫描的那个索引"分派，不是按表的聚簇索引
  }
```

> ★ `lock_sec_rec_read_check_and_lock()` 内部**也会先做隐式锁物化**（`if (!page_rec_is_supremum(rec)) lock_rec_convert_impl_to_expl(...)`）。
- **精判**：`row_vers_impl_x_locked`（`row0vers.cc:536`）用二级记录内嵌的 PK **回表**定位聚簇记录 → 读聚簇 `DB_TRX_ID` + 版本链判断"这条二级项是否由某活跃 trx 的插入/delete-mark 产生"；聚簇记录已被 purge → 返回 nullptr。

**④ 物化**（`lock_rec_convert_impl_to_expl`，`lock0lock.cc:5541`）：确认有隐式锁持有者后，**在二级索引上也建显式锁**——`lock_rec_add_to_queue(LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP, block, heap_no, index, trx)`，挂进与聚簇锁**同一个** lock_sys 哈希（按 (space, page) 分片，key = block+heap_no）。**这是"二级索引上出现锁"的途径二**（途径一是本事务显式锁定读）；物化只建 X 锁不建 gap 锁（隐式锁只保护"记录存在"，与 `LOCK_INSERT_INTENTION` 兼容）。

**物化调用点只有 5 处**：二级读加锁前（`lock_sec_rec_read_check_and_lock`，5721）、聚簇读加锁前（5770）、聚簇修改前（5618）、部分回滚（`row_convert_impl_to_expl_if_needed`，`row0undo.cc:339`——IODKU/REPLACE 中段回滚时防中途释放隐式锁）+ 本体。**提交与死锁检测都不物化**——提交时 trx 置 `TRX_STATE_COMMITTED_IN_MEMORY`，隐式锁随之失效（`trx0trx.cc:1929-1951` 注释："this particular point in time as the moment the trx's implicit locks become released"）。

**⑤ 与 change buffer / purge 的锁视角**：唯一索引 INSERT 不进 ibuf 的**锁原因**——唯一性靠查重时对"B 树中真实存在的相邻记录"加 S 锁；若允许进 ibuf，插入被延迟、无处挂锁、merge 时又不做唯一性检查（`ibuf_insert_to_index_page_low` 纯物理插入），两个并发 trx 的重复键会同时"通过"查重。purge 物理删除二级记录**不检查锁**（`row_purge_poss_sec` 只做版本判定）——"记录上必然无锁"由 purge view 语义保证（只删所有活跃事务都看不见的旧版本，锁必已释放）。

---

### 回收

#### purge：物理删除

purge 用 `node->ref`（主键值，undo 里没有物理位置）重新定位聚簇记录 → 对每个二级索引 `row_build_index_entry_low(node->row, ..., ROW_BUILD_FOR_PURGE)` 重建**旧** entry → `row_search_index_entry` 定位 → `row_purge_poss_sec`（聚簇已删 或 无存活旧版本需要）→ 乐观（可进 ibuf）/悲观物理删除。完整链路与"purge 会触发页合并"见 [`transaction.md`](transaction.md)「purge」。

**★ `row_purge_poss_sec()`——"能不能物理删"的唯一判据**（`row0purge.cc:261`）：

```cpp
bool row_purge_poss_sec(purge_node_t *node, dict_index_t *index,
                        const dtuple_t *entry) {
  bool can_delete;
  mtr_t mtr;
  ut_ad(!index->is_clustered());
  mtr_start(&mtr);

  can_delete =
      !row_purge_reposition_pcur(BTR_SEARCH_LEAF, node, &mtr) ||
      !row_vers_old_has_index_entry(true, node->pcur.get_rec(), &mtr, index,
                                    entry, node->roll_ptr, node->trx_id);
  //  ★★ 两个条件任一成立即可删：
  //    ① 聚簇记录已不存在（reposition 失败）→ 没人再引用这条二级项
  //    ② 沿 undo 版本链回查，没有"更老但仍未 purge"的版本含有这条 entry
  //    ★ 第一个参数 true = "purge 视角"（DML rollback 路径传 false）

  if (node->found_clust) node->pcur.commit_specify_mtr(&mtr);
  else                   mtr_commit(&mtr);
  return (can_delete);
}
```

```cpp
static bool row_purge_reposition_pcur(ulint mode, purge_node_t *node, mtr_t *mtr) {
  if (node->found_clust) {
    node->found_clust = node->pcur.restore_position(mode, mtr, UT_LOCATION_HERE);
  } else {
    node->found_clust =
        row_search_on_row_ref(&node->pcur, mode, node->table, node->ref, mtr);
    //  ★★ node->ref = 主键值 tuple，不是 (page_no, heap_no) 物理位置
    if (node->found_clust) node->pcur.store_position(mtr);
  }
  if (!node->found_clust) node->pcur.close();
  return (node->found_clust);
}
```

> ★★ **`node->ref` 的构造**（`row0purge.cc:1072`）：`ptr = trx_undo_rec_get_row_ref(ptr, clust_index, &(node->ref), node->heap);`——从 undo 解析出**主键值**，再 `row_search_on_row_ref()` 重新定位。这正是"purge 借主键重新定位"的源码落点，也是它**必然慢于物理指针**的原因。

**★ `row_vers_old_has_index_entry()` 的核心**（`row0vers.cc:977`）：

```cpp
  clust_index = index->table->first_index();
  if (also_curr && !rec_get_deleted_flag(rec, comp)) {
    row = row_build(ROW_COPY_POINTERS, clust_index, rec, clust_offsets, ..., &ext, heap);
    entry = row_build_index_entry(row, ext, index, heap);
    /* NOTE that we cannot do the comparison as binary fields because the row
    is maybe being modified so that the clustered index record has already been
    updated to a different binary value in a char field, but the collation
    identifies the old and new value anyway! */
    if (entry && dtuple_coll_eq(entry, ientry)) return true;   // ★ 按 collation 比较
  }
  version = rec;
  for (;;) {
    trx_undo_prev_version_build(rec, mtr, version, clust_index, clust_offsets,
                                heap, &prev_version, nullptr,
                                dict_index_has_virtual(index) ? &vrow : nullptr,
                                0, nullptr);
    mem_heap_free(heap2);
    if (!prev_version) return false;              // ★ 版本链到底 → 可以删
    ...
    if (!rec_get_deleted_flag(prev_version, comp)) {
      row = row_build(ROW_COPY_POINTERS, clust_index, prev_version, ...);
      entry = row_build_index_entry(row, ext, index, heap);
      if (entry && dtuple_coll_eq(entry, ientry)) return true;
      //  ★★ 把历史版本"重新投影"成二级索引条目，与待 purge 的 entry 比较
    }
    version = prev_version;
  }
```

> ★★ **注意 `dtuple_coll_eq` 而不是二进制比较**——与 `row_upd_changes_ord_field_binary` 的立场相反。purge 关心的是"逻辑上是否还是同一条"，更新关心的是"物理字节是否变了"。
> ⚠️ 该函数**不只被 purge 调用**——`row0umod.cc:493`（update 的 rollback）也用。

**★ 三态分发与"乐观 → 悲观"**（`row_purge_remove_sec_if_poss`）：

```cpp
static inline void row_purge_remove_sec_if_poss(purge_node_t *node,
                                                dict_index_t *index,
                                                const dtuple_t *entry) {
  if (!entry) {
    /* The node->row must have lacked some fields of this index. This is
    possible when the undo log record was written before this index was created. */
    return;                                        // ★ undo 早于索引创建 → 跳过
  }
  if (row_purge_remove_sec_if_poss_leaf(node, index, entry)) return;
retry:
  auto success = row_purge_remove_sec_if_poss_tree(node, index, entry);
  if (!success && n_tries < BTR_CUR_RETRY_DELETE_N_TIMES) {
    n_tries++;
    std::this_thread::sleep_for(std::chrono::milliseconds(BTR_CUR_RETRY_SLEEP_TIME_MS));
    goto retry;
  }
  ut_a(success);
}
```

```cpp
  mode = (index->table->is_temporary() || dict_index_is_spatial(index))
             ? BTR_MODIFY_LEAF
             : BTR_MODIFY_LEAF | BTR_DELETE;       // ★ 允许 change buffer 缓冲这次删除
  ...
  switch (search_result) {
    case ROW_FOUND:
      if (row_purge_poss_sec(node, index, entry)) {
        if (!rec_get_deleted_flag(btr_cur_get_rec(btr_cur), ...)) {
          ib::error(ER_IB_MSG_1008) << "tried to purge non-delete-marked record ...";
          ut_d(ut_error); ut_o(goto func_exit_no_pcur);
        }
        ...
        if (!btr_cur_optimistic_delete(btr_cur, 0, &mtr)) {
          success = false;                          // ★ 需要 SMO → 交给 _tree 版本
        }
      }
      [[fallthrough]];
    case ROW_NOT_DELETED_REF:   /* The index entry is still needed. */
    case ROW_BUFFERED:          /* The deletion was buffered. */
    case ROW_NOT_FOUND:         /* The index entry does not exist. */
      ...
  }
```

> ★ **`ROW_BUFFERED` 也算成功**——purge 的删除可以进 change buffer，之后由后台 merge 完成。
> ★ `row_purge_del_mark()` 的循环起点是 `node->index = clust_index->next()`（**跳过聚簇**），最后才 `row_purge_remove_clust_if_poss()`——因为聚簇记录上的 `DB_TRX_ID`/`DB_ROLL_PTR` 是版本链回查的起点，**必须先清完所有二级索引**。

**★ `PAGE_MAX_TRX_ID` 的位置与"聚簇恒 0"的三条证据**：

```cpp
/** highest id of a trx which may have modified a record on the page; trx_id_t;
defined only in secondary indexes and in the insert buffer tree */
constexpr uint32_t PAGE_MAX_TRX_ID = 18;       // ★ 8 字节，占 18..25
/** end of private data structure of the page header which are set in a page create */
constexpr uint32_t PAGE_HEADER_PRIV_END = 26;  // ★ 18 + 8 = 26
```

```cpp
  memset(page + PAGE_HEADER, 0, PAGE_HEADER_PRIV_END);
  //  ★★ 新建页时一次性清零 PAGE_HEADER..+26，PAGE_MAX_TRX_ID 完全落在区间内
```

```cpp
  if (!index->is_clustered() && !index->table->is_temporary()) {
    ut_a(max_trx_id);
    page_set_max_trx_id(block, buf_block_get_page_zip(block), max_trx_id, mtr);
    //  ★★ 只有非聚簇才回写（btr_lift_page_up / btr_page_empty）
  }
```

```cpp
  /* Mark that we are going to truncate the index tree
  We use PAGE_MAX_TRX_ID as it should be always 0 for clustered index. */
  mlog_write_ull(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), IB_ID_MAX, &mtr);
  //  ★★★ btr_truncate 直接把这个字段当"便签"用——因为聚簇页它必然是 0
```

```cpp
static inline void page_update_max_trx_id(buf_block_t *block, page_zip_des_t *page_zip,
                                          trx_id_t trx_id, mtr_t *mtr) {
  ut_ad(trx_id || recv_recovery_is_on());
  ut_ad(page_is_leaf(buf_block_get_frame(block)));
  if (page_get_max_trx_id(buf_block_get_frame(block)) < trx_id) {
    page_set_max_trx_id(block, page_zip, trx_id, mtr);   // ★ 单调取 max，只增不减
  }
}
```

> ⚠️ `PAGE_MAX_TRX_ID` **不在 `page0page.h`**——常量在 **`page0types.h:77`**；`page0page.h` 只有引用。

**★ 尺寸统计**（⚠️ **不存在 `dict_stats_update_if_needed()`**，真名 `dict_stats_update()`）：

```cpp
  table->stat_clustered_index_size = index->stat_index_size;   // 聚簇 = 链表头
  table->stat_sum_of_other_index_sizes =
      sum_of_index_sizes - index->stat_index_size;              // transient：减出来
```

```cpp
  table->stat_sum_of_other_index_sizes = 0;
  for (index = index->next(); index != nullptr; index = index->next()) {
    if (index->type & DICT_FTS || dict_index_is_spatial(index)) continue;
    dict_stats_analyze_index(index);
    table->stat_sum_of_other_index_sizes += index->stat_index_size;  // persistent：累加
  }
```

```cpp
  index->stat_index_size = btr_get_size(index, BTR_TOTAL_SIZE, &mtr);
  index->stat_n_leaf_pages = btr_get_size(index, BTR_N_LEAF_PAGES, &mtr);
```

```cpp
  table->stat_clustered_index_size = 1;
  /* 1 page for each index, not counting the clustered */
  table->stat_sum_of_other_index_sizes = UT_LIST_GET_LEN(table->indexes) - 1;  // 兜底
```

> ★ **FTS / spatial 索引不计入**这两个统计。
> ★ 精确统计（`ANALYZE`）前会 `srv_wake_purge_thread_if_not_active()`——因为二级索引里的 delete-mark 项会让 `btr_get_size` 统计到"即将被 purge 走"的页。

---

### 权衡

#### 写入代价、读收益与尺寸对比

| 维度 | 事实 | 出处 |
|---|---|---|
| 写入代价 | 每行插入要写 N 棵树（N = 二级索引数）；`row_ins` 主循环逐索引插入 | 本篇「创建与插入」 |
| 缓解 | change buffer 把"叶页不在 pool"的插入/删除缓冲化（唯一插入除外） | [`ibuf.md`](ibuf.md) |
| 更新代价 | 键值真变了才"delete-mark + 插新"（`row_upd_changes_ord_field_binary` 跳过优化） | 本篇「更新」 |
| 读收益 | 覆盖索引免回表（`need_to_access_clustered` 未置位）；否则回表一次 | [`access.md`](access.md) |
| 读代价 | 可见性必须回表（无 trx_id）；delete-marked 项堆积增加扫描成本 | [`transaction.md`](transaction.md) |
| 尺寸 | `table->stat_sum_of_other_index_sizes`（全部二级之和）vs `stat_clustered_index_size`（`dict0stats.cc:723-734` 聚合） | [`stats.md`](stats.md) |
| 页内 | 二级页有 `PAGE_MAX_TRX_ID`（聚簇恒 0）；AHI/change buffer 只服务二级 | [`physical_storage.md`](physical_storage.md)、[`ahi.md`](ahi.md) |

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `innodb_change_buffering` | change buffer 总开关（`ibuf_should_try` 首查） |
| `table->stat_sum_of_other_index_sizes` | 全部二级索引尺寸之和（vs 聚簇） |
| `PAGE_MAX_TRX_ID` | 二级索引页的可见性粗筛字段 |
| `BTR_CUR_INSERT_TO_IBUF` | 游标标志：插入已缓冲进 ibuf |
| `trx->check_unique_secondary` | 是否启用唯一二级索引检查（影响 `BTR_IGNORE_SEC_UNIQUE`） |
| `innodb_force_recovery` | ≥ `SRV_FORCE_NO_IBUF_MERGE` 时 ibuf 关闭 |

---

## Misc

### 易混淆点

- **不存在 `row_allow_ibuf_insert` 函数**——change buffer 入口在 `btr_cur_search_to_nth_level` 内部（`BTR_INSERT` + `ibuf_should_try` + 叶页不在 pool），row0ins 只设开关、看出口标志。
- **唯一二级索引的 INSERT 不进 ibuf，但它的 delete-mark 可以进**（`ibuf_should_try` 的 `ignore_sec_unique` 参数）。
- **唯一检查不是"先插入后检查"**——查重（加 S 锁）在插入前，之后**重新定位**再插入。
- **更新 = delete-mark 旧项 + 插新项，且 delete-mark 不释放锁**。
- **`row_upd` 先聚簇后二级，rollback 逆序**——保证任何时刻二级项都能回表找到聚簇记录。
- **`row_sel_sec_rec_is_for_clust_rec` 只比用户定义列**（追加 PK 的对应性由回表定位保证），且只有"回表拿到旧版本/二级项被标记"时才需要校验。
- **快捷搜索 `row_sel_try_search_shortcut_for_mysql` 仅限聚簇**。
- **代码里"二级"在页级 MVCC 语义的识别统一靠 `dict_index_is_sec_or_ibuf`**（ibuf 树按二级处理）。

**★★ 真名勘误全表**：

| 常见写法 | 8.0.39 真相 |
|---|---|
| `row_ins_sec_index_entry()` 里做定位/查重/写入 | **错**。它只是"乐观 → 失败则悲观"的**外壳**；五步全在 `row_ins_sec_index_entry_low()`（且 `low` 是**非 static 导出符号**） |
| `row_ins_duplicate_error_in_sec()` | **不存在**。真名 `row_ins_scan_sec_index_for_duplicate()` + `row_ins_dupl_error_with_rec()` |
| `row_ins_must_modify_rec()` 里查 `REC_INFO_DELETED_FLAG` | **8.0.39 函数体里没有**——delete-mark 检查在调用方 `row_ins_sec_index_entry_by_modify()` 内 |
| `row_ins_index_entry_set_ins_node()` | **不存在**。真名 `row_ins_index_entry_set_vals()` / `ins_node_set_new_row()` / `ins_node_create()` |
| `row_undo_upd_del_mark_or_remove_sec()` | **不存在**。真名 `row_undo_mod_del_mark_or_remove_sec()`；且 update/delete undo 在 **`row0umod.cc`**，只有 insert undo 在 `row0uins.cc` |
| `btr_cur_del_mark_set_sec_rec_and_save()` | **不存在**。只有 `..._set_sec_rec()` + `..._set_sec_rec_log()` |
| `row_sel_get_clust_rec_for_mysql()` 是函数 | **是 functor 类** `Row_sel_get_clust_rec_for_mysql`（`row0sel.cc:3096`），调用处的名字是**同名对象**（`:4472`） |
| `row_search_mvcc()` 里有 `cons_read_requires_clust_rec` | **没有**。该变量只在 `row_sel()`（查询图路径）；`row_search_mvcc` 用 `need_to_access_clustered` + `requires_clust_rec:` 标签 |
| `row_purge_del_mark_sec()` | **不存在**。真名 `row_purge_del_mark()` + `row_purge_remove_sec_if_poss()`（`_leaf` / `_tree`） |
| `dict_index_is_clustered()` / `dict_index_is_multi_value()` | **不存在**（0 命中）。8.0 一律成员函数 `index->is_clustered()` / `is_multi_value()` |
| `dict_stats_update_if_needed()` | **不存在**。真名 `dict_stats_update(table, dict_stats_upd_option_t)` |
| `dict_table_get_n_user_cols()` | **不存在**（0 命中）。用 `n_def` / `n_cols` / `dict_index_get_n_ordering_defined_by_user()` |
| `PAGE_MAX_TRX_ID` 在 `page0page.h` | **不在**。常量在 **`page0types.h:77`** |
| `page_mem_alloc` 初始化 `PAGE_MAX_TRX_ID` | **不是**。是 `page_create_low()` 的 `memset(page + PAGE_HEADER, 0, PAGE_HEADER_PRIV_END)` |
| `row_build_low()` 处理 `ROW_BUILD_FOR_PURGE` | **没有**。它的 `type` 只有 `ROW_COPY_POINTERS` / `ROW_COPY_DATA`；`ROW_BUILD_FOR_PURGE` 只影响 `row_build_index_entry_low()` |
| `rec_get_deleted_flag()` 在 `rem0rec.h` | 头文件**只有声明**；实现在 `rem0rec.ic:353` |
| `btr_cur_optimistic_insert()` 处理 `BTR_CUR_INSERT_TO_IBUF` | **没有任何处理**。`btr0cur.cc` 全文只在 `btr_cur_search_to_nth_level` 里**赋值**一次；真正处理方是 `row0ins.cc:2913` 的 `goto func_exit` |
| `lock_sec_rec_some_has_impl()` 返回 trx_id | 返回 **`trx_t *`** |
| `lock_clust_rec_some_has_impl()` 在 `lock0lock.cc` | 在 **`lock0priv.ic:52`**，只有一行 |
| `row_vers_old_has_index_entry()` 只被 purge 调用 | 也被 **update 的 rollback** 调用（`row0umod.cc:493`） |
| `row_sel_sec_rec_is_for_clust_rec()` 是二进制比较 | **明确不是**——源码注释 `NOT done as a binary comparison`，用 `cmp_data_data(..., true)` 按 collation |
| `is_multi_value_clust_and_sec_equal()` 在 `row0sel.cc` | 定义在 **`data0data.cc:828`** |
| `btr_cur_search_to_nth_level` 里 `BTR_INSERT` 直接进 ibuf | 实际是 `BTR_INSERT` → **`BTR_INSERT_OP`**（或 `BTR_INSERT_IGNORE_UNIQUE_OP`）→ `ibuf_should_try(index, btr_op != BTR_INSERT_OP)` → 页不在 BP 才 `ibuf_insert()` |

**★ 三个"结构性事实"**：

1. **`dict_index_get_n_unique_in_tree()` 对二级索引返回"全部字段数"，对聚簇返回 `n_uniq`**——这是"二级索引复用记录要求全字段相等"与"B-tree 内靠键列+主键唯一化"的共同根源。
2. **`node->index = clust_index->next()` 这个"跳过聚簇"的写法在三处成对出现**：purge（`row_purge_record_func`）、insert 的 rollback（`row0uins`）、update 的 rollback（`row0umod`）——**都是"先二级后聚簇"**，与写路径相反。
3. **delete-mark 与物理删除的分工是单向的**：`btr_cur_del_mark_set_sec_rec()` 只由 DML/DDL/rollback 写入，purge 从不调它；purge 只用 `rec_get_deleted_flag()` 做**校验**（断言"只能 purge 已打标记的项"），实际删除走 `btr_cur_optimistic_delete()` / `pessimistic_delete()`。

### 生命周期闭环总图

```
         ┌─ 创建/插入：row_ins_index_entry ─ 三分派 ─ row_ins_sec_index_entry[_low]
         │              [BTR_INSERT → ibuf 决策] → [唯一查重加锁 → 重定位] → [乐观/悲观插入]
         ├─ 更新：row_upd_sec_step → changes_ord_field_binary?（否→跳过）
         │        └ row_upd_sec_index_entry_low = delete-mark 旧项 + 插新项
写 ──────┤
         ├─ 删除：btr_cur_del_mark_set_sec_rec（第一阶段，锁不释放）
         │
读 ──────┼─ row_search_mvcc 二级分支：粗筛 → ICP → requires_clust_rec 回表
         │   └ row_sel_sec_rec_is_for_clust_rec 双向校验（用户列逐字段）
         │
回收 ────┴─ purge：node->ref 定位 → 旧 entry 重建 → row_purge_poss_sec → 物理删除
              （row_vers_old_has_index_entry 版本链回溯判定"是否还被引用"）
```

### 一句话总结

二级索引是"**只存定位键、不存版本、不存数据**"的指针表：写路径靠"查重加锁 + 乐观/悲观插入 + delete-mark 两阶段"维护，读路径靠"页级粗筛 + ICP 剪枝 + 回表 + 双向校验"取数，回收靠 purge 借主键重新定位——它的全部特殊性（回表、两阶段删除、change buffer 限定、PAGE_MAX_TRX_ID）都源自同一个事实：**二级索引记录里没有 `DB_TRX_ID`/`DB_ROLL_PTR`，也没有整行数据**。

---

## 参考

**相关文档**
- 记录格式（二级记录 = 索引列 + 追加 PK）：[`record_format.md`](record_format.md)
- n_uniq 与追加 PK 的定义：[`metadata.md`](metadata.md)
- delete-mark 两阶段与 `row_vers_old_has_index_entry`：[`transaction.md`](transaction.md)
- 回表链路与覆盖索引：[`access.md`](access.md)
- change buffer（14 条否决条件）：[`ibuf.md`](ibuf.md)
- 唯一性检查完整链路：[`constraint.md`](constraint.md)
- 取行主链 `row_search_mvcc`：[`../innodb/row_search.md`](../innodb/row_search.md)
- 二级索引类型语义：[`types.md`](types.md)
- 索引构造（entry 的 `n_fields_cmp` 上游）：[`construction.md`](construction.md)

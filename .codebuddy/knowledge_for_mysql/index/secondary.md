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
- **精判**：`row_vers_impl_x_locked`（`row0vers.cc:536`）用二级记录内嵌的 PK **回表**定位聚簇记录 → 读聚簇 `DB_TRX_ID` + 版本链判断"这条二级项是否由某活跃 trx 的插入/delete-mark 产生"；聚簇记录已被 purge → 返回 nullptr。

**④ 物化**（`lock_rec_convert_impl_to_expl`，`lock0lock.cc:5541`）：确认有隐式锁持有者后，**在二级索引上也建显式锁**——`lock_rec_add_to_queue(LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP, block, heap_no, index, trx)`，挂进与聚簇锁**同一个** lock_sys 哈希（按 (space, page) 分片，key = block+heap_no）。**这是"二级索引上出现锁"的途径二**（途径一是本事务显式锁定读）；物化只建 X 锁不建 gap 锁（隐式锁只保护"记录存在"，与 `LOCK_INSERT_INTENTION` 兼容）。

**物化调用点只有 5 处**：二级读加锁前（`lock_sec_rec_read_check_and_lock`，5721）、聚簇读加锁前（5770）、聚簇修改前（5618）、部分回滚（`row_convert_impl_to_expl_if_needed`，`row0undo.cc:339`——IODKU/REPLACE 中段回滚时防中途释放隐式锁）+ 本体。**提交与死锁检测都不物化**——提交时 trx 置 `TRX_STATE_COMMITTED_IN_MEMORY`，隐式锁随之失效（`trx0trx.cc:1929-1951` 注释："this particular point in time as the moment the trx's implicit locks become released"）。

**⑤ 与 change buffer / purge 的锁视角**：唯一索引 INSERT 不进 ibuf 的**锁原因**——唯一性靠查重时对"B 树中真实存在的相邻记录"加 S 锁；若允许进 ibuf，插入被延迟、无处挂锁、merge 时又不做唯一性检查（`ibuf_insert_to_index_page_low` 纯物理插入），两个并发 trx 的重复键会同时"通过"查重。purge 物理删除二级记录**不检查锁**（`row_purge_poss_sec` 只做版本判定）——"记录上必然无锁"由 purge view 语义保证（只删所有活跃事务都看不见的旧版本，锁必已释放）。

---

### 回收

#### purge：物理删除

purge 用 `node->ref`（主键值，undo 里没有物理位置）重新定位聚簇记录 → 对每个二级索引 `row_build_index_entry_low(node->row, ..., ROW_BUILD_FOR_PURGE)` 重建**旧** entry → `row_search_index_entry` 定位 → `row_purge_poss_sec`（聚簇已删 或 无存活旧版本需要）→ 乐观（可进 ibuf）/悲观物理删除。完整链路与"purge 会触发页合并"见 [`transaction.md`](transaction.md)「purge」。

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

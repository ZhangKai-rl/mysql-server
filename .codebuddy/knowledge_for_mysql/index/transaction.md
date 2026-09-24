# 索引 × 事务：MVCC、purge 与唯一性的联动

> 基于 MySQL 8.0.39 源码。本篇讲 **索引与事务机制的交叉地带**：二级索引为什么不能像聚簇那样直接删（delete-mark 两阶段）、`row_vers_old_has_index_entry` 的版本链回溯、purge 如何借索引定位并物理删除、唯一性检查为什么不能靠 MVCC 只能靠锁、以及长事务导致索引记录膨胀的机制。
>
> **边界**：MVCC 的可见性判定与 ReadView 本身见 [`../innodb/mvcc.md`](../innodb/mvcc.md)；行锁/gap lock/隐式锁见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)；undo 与 purge 的整体机制见 [`../innodb/undo_log.md`](../innodb/undo_log.md)；唯一性检查的完整链路（本篇只取 MVCC 交互部分）见 [`constraint.md`](constraint.md)。本篇是"索引"视角的整合。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [为什么二级索引没有 trx_id/roll_ptr](#为什么二级索引没有-trx_idroll_ptr)
    - [delete-mark 两阶段：`row_vers_old_has_index_entry`](#delete-mark-两阶段row_vers_old_has_index_entry)
  - 读路径
    - [二级索引的可见性：页级粗筛 + 回表](#二级索引的可见性页级粗筛--回表)
    - [delete-marked 记录对查询的影响](#delete-marked-记录对查询的影响)
  - 写路径
    - [purge：undo → 聚簇 ref → 索引定位 → 物理删除](#purgeundo--聚簇-ref--索引定位--物理删除)
    - [唯一性检查为什么必须加锁](#唯一性检查为什么必须加锁)
    - [长事务与索引记录膨胀](#长事务与索引记录膨胀)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引是 MVCC 与 purge 的**舞台**：一致性读靠索引找到记录再判版本；删除先给索引项打 delete-mark、purge 阶段才物理删除；唯一性检查借索引完成但靠锁保证。本篇把这三条交叉线讲透。

### 用途

- 理解"为什么删除了数据索引页不缩小"（delete-mark 未 purge）；
- 理解 purge 与二级索引的精确交互（何时能物理删、会不会触发页合并）；
- 理解唯一索引插入时的锁行为（为什么 RC/RR 都必须加锁）。

### 版本演进

- 二级索引 delete-mark 机制自 InnoDB 早期至今未变；`row_vers_old_has_index_entry` 在 8.0 增加了多值索引（`dict_index_has_virtual`）分支。

---

## 理论基础

### 设计思想与权衡

**1. 二级索引项只是"指针"，没有版本信息。** 二级索引记录 = 索引列 + 隐式 PK（没有 `DB_TRX_ID`/`DB_ROLL_PTR`）。代价是：判断可见性**必须回表**（版本链只能从聚簇记录出发）；收益是：二级索引记录更小、且多个历史版本可以**共享同一个索引项**——这正是 delete-mark 两阶段存在的原因。

**2. 删除是两阶段的。** 第一阶段只打 delete-mark（逻辑删除，仍占 B-tree 槽位），第二阶段由 purge 物理删除。**不能**像聚簇那样一步到位的原因：一条二级索引项可能还在被"某个未 purge 的旧版本"引用——只有回溯聚簇版本链才知道。

**3. purge 与 MVCC 的水位共享。** purge 能推进多远受最老 ReadView 约束（`purge_sys->view`）；`row_vers_old_has_index_entry` 的判定标准也是"trx id >= purge view"——两者用同一个水位，保证"读还在用的版本不会被删掉索引项"。

**4. 唯一性不能靠 MVCC，只能靠锁。** MVCC 只能判断"已存在的历史版本"，无法阻止"另一个活跃事务即将插入同值"。唯一性必须是"当前 + 未来"的保证——这是唯一性检查与一致性读在机制上的本质分叉。

---

## 核心实现

### 主线与基础构件

#### 为什么二级索引没有 trx_id/roll_ptr

源码注释（`row0sel.cc:5385-5392`）原文：

```cpp
      /* We are looking into a non-clustered index,
      and to get the right version of the record we
      have to look also into the clustered index: this
      is necessary, because we can only get the undo
      information via the clustered index record. */
```

三个直接后果：

1. **判断可见性必须回表**：`lock_sec_rec_cons_read_sees` 只能做粗筛，粗筛不过就 `requires_clust_rec`（见读路径节）；
2. **ICP 在二级索引上判断的是当前版本**（`row_search_idx_cond_check` 的 `rec` 是当前索引记录）——不匹配可以省掉回表（保守剪枝），匹配与否最终都由回表后的 old_vers 兜底；
3. **同一 key 的多个 delete-marked 索引项可以并存**（各对应不同 PK）——这直接说明"为什么不能直接删"。

#### delete-mark 两阶段：`row_vers_old_has_index_entry`

**阶段一：打 delete-mark**。`btr_cur_del_mark_set_sec_rec`（`btr0cur.cc:4598`）只做两件事：`btr_rec_set_deleted_flag(rec, ..., val)` + 写 redo。调用点：UPDATE 改索引列（`row0upd.cc:2133/2418`）、`row_delete_for_mysql`（`row0mysql.cc:1325`）、**回滚 UPDATE**（`row0umod.cc:496/724`）、DDL log 表。

**阶段二之前的判定**：回滚 UPDATE 时（`row0umod.cc:483-530`）最直白地展示了"两阶段"的边界：

```cpp
/* We should remove the index record if no prior version of the row,
   which cannot be purged yet, requires its existence. If some requires,
   we should delete mark the record. */
success = node->pcur.restore_position(BTR_SEARCH_LEAF, &mtr_vers, ...);
old_has = row_vers_old_has_index_entry(false, node->pcur.get_rec(), &mtr_vers,
                                       index, entry, 0, 0);
if (old_has) {
  err = btr_cur_del_mark_set_sec_rec(BTR_NO_LOCKING_FLAG, btr_cur, true, thr, &mtr);
} else {
  /* Remove the index record */   // 乐观/悲观物理删除
}
```

**`row_vers_old_has_index_entry`**（`row0vers.cc:983-1234`）逐段：

```cpp
bool row_vers_old_has_index_entry(
    bool also_curr,         // purge 传 true（把当前版本也算进去）
    const rec_t *rec,       // 聚簇记录；调用者必须持页 latch
    mtr_t *mtr, dict_index_t *index,
    const dtuple_t *ientry, // 待删的二级索引 entry
    roll_ptr_t roll_ptr, trx_id_t trx_id)
{
  // 前提：不持有 purge_sys->latch（ut_ad(!rw_lock_own(&purge_sys->latch, RW_LOCK_S))）
  if (also_curr && !rec_get_deleted_flag(rec, comp)) {
    row = row_build(ROW_COPY_POINTERS, clust_index, rec, ...);
    entry = row_build_index_entry(row, ext, index, heap);
    /* NOTE that we cannot do the comparison as binary fields because
       the row is maybe being modified so that the clustered index record
       has already been updated to a different binary value in a char
       field, but the collation identifies the old and new value anyway! */
    if (entry && dtuple_coll_eq(entry, ientry)) return true;   // 当前版本就要 → 不能删
  }
  version = rec;
  for (;;) {   // ★ 沿 undo 版本链回溯
    trx_undo_prev_version_build(rec, mtr, version, clust_index, clust_offsets,
                                heap, &prev_version, ...);
    if (!prev_version) return false;               // 版本链到底 → 可删
    if (!rec_get_deleted_flag(prev_version, comp)) {   // delete-marked 旧版本跳过
      row = row_build(ROW_COPY_POINTERS, clust_index, prev_version, ...);
      entry = row_build_index_entry(row, ext, index, heap);
      if (entry && dtuple_coll_eq(entry, ientry)) return true;
    }
    version = prev_version;
  }
}
```

关键点：

- **必须用 collation 比较**（`dtuple_coll_eq`）而非二进制比较——CHAR 列可能已被改成另一个二进制串，但 collation 下仍相等（注释原文，1125-1129 行）；
- **delete-marked 的旧版本跳过**——只关心"仍然存活"的版本；
- **版本链安全边界**：顶端由 mtr 的页 latch 锁住，底端由 purge view 保证（purge view 一定"超车"任何活跃 read view）——这两点使无锁遍历版本链是安全的。

---

### 读路径

#### 二级索引的可见性：页级粗筛 + 回表

`lock_sec_rec_cons_read_sees`（`lock0lock.cc:268-297`）：

```cpp
bool lock_sec_rec_cons_read_sees(const rec_t *rec, const dict_index_t *index,
                                 const ReadView *view) {
  if (recv_recovery_is_on()) return false;
  else if (index->table->is_temporary()) return true;
  trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));
  return (view->sees(max_trx_id));
}
```

整页共享一个 `PAGE_MAX_TRX_ID`（只增不减）：`view->sees(max_trx_id)` → 页上**所有**记录都可见（连 delete-mark 判定都只需看标记位）；否则**无法确定**，必须 `requires_clust_rec` 回表用聚簇版本链精确判定。`PAGE_MAX_TRX_ID` 不写 redo（安全方向的失真，详见 [`../innodb/mvcc.md`](../innodb/mvcc.md)）。

#### delete-marked 记录对查询的影响

`row_search_mvcc` 主分支（`row0sel.cc:5431-5459`）：

```cpp
if (rec_get_deleted_flag(rec, comp)) {
  /* The record is delete-marked: we can skip it */
  prebuilt->try_unlock(true);
  if (index == clust_index && unique_search && !prebuilt->used_in_HANDLER) {
    err = DB_RECORD_NOT_FOUND;
    goto normal_return;
  }
  goto next_rec;
}
```

- 二级索引上的 delete-mark 项：直接 `goto next_rec` 跳过（此刻 `rec` 可能已是 old_vers——不能假设它在 buffer pool 页上）；
- **但它仍占据 B-tree 槽位与锁空间**——加锁扫描时会锁住它，可能阻塞插入；
- 唯一二级索引有额外校验：`row_sel_sec_rec_is_for_clust_rec` 确认索引项确实对应那条聚簇记录（因为同 key 的多个 delete-marked 项并存，只 PK 不同）。

---

### 写路径

#### purge：undo → 聚簇 ref → 索引定位 → 物理删除

完整链路：

```
trx_purge() → clone_oldest_view(&purge_sys->view)   // 取最老 read view 作水位
  → trx_purge_attach_undo_recs()                    // 按 rseg 批量取 undo
  → que_thr_step → row_purge_step()
      → row_purge_parse_undo_rec()                  // undo → node->ref（PK tuple）
      → row_purge_record_func()
          TRX_UNDO_DEL_MARK_REC → row_purge_del_mark()
          TRX_UNDO_UPD_EXIST_REC → row_purge_upd_exist_or_extern_func()
```

**定位**（`row_purge_parse_undo_rec`，`row0purge.cc:856-1087`）：undo 记录里**没有物理位置**（记录可能已被页分裂搬走），只有主键值——`trx_undo_rec_get_row_ref` 取出 `node->ref` → `row_purge_reposition_pcur` → `row_search_on_row_ref` 重新定位聚簇记录；`node->row` 是"旧版本的部分行"（只含索引列），用于重建**旧**二级索引 entry。

**顺序**（`row_purge_del_mark`，`row0purge.cc:657-695`）：**先所有二级索引、最后聚簇**：

```cpp
while (node->index != nullptr) {
  if (node->index->type != DICT_FTS) {
    dtuple_t *entry = row_build_index_entry_low(node->row, nullptr, node->index,
                                                heap, ROW_BUILD_FOR_PURGE);
    row_purge_remove_sec_if_poss(node, node->index, entry);
  }
  node->index = node->index->next();
}
return row_purge_remove_clust_if_poss(node);
```

**什么时候能物理删**（`row_purge_poss_sec`，`row0purge.cc:275-298`）：

```cpp
can_delete =
    !row_purge_reposition_pcur(BTR_SEARCH_LEAF, node, &mtr) ||   // 聚簇记录已不存在
    !row_vers_old_has_index_entry(true, node->pcur.get_rec(), &mtr,
                                  index, entry, node->roll_ptr, node->trx_id);
```

即：**聚簇记录找不到了（已被删）或没有任何存活旧版本需要这个 entry** → 可删。

删除分乐观/悲观两档：

- **乐观**（`row_purge_remove_sec_if_poss_leaf`）：`row_search_index_entry(index, entry, mode | BTR_DELETE, ...)`——`BTR_DELETE` 允许 change buffering（临时表/spatial 除外，`ROW_BUFFERED` 不算失败）；先检查"只 purge delete-marked 记录"（否则报 `ER_IB_MSG_1008`）；
- **悲观**（`..._tree`，`BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE`）：`btr_cur_pessimistic_delete` 失败重试 `BTR_CUR_RETRY_DELETE_N_TIMES`。

`row_search_index_entry`（`row0row.cc:934-989`）定位的判定：`PAGE_CUR_LE` 打开（R-tree 用 `PAGE_CUR_RTREE_LOCATE`），要求 `low_match == n_fields`（全字段匹配，含隐式 PK）。

**purge 会不会触发页分裂/合并**：**会合并、不会分裂**（purge 只删不插）。`btr_cur_pessimistic_delete` 末尾 `btr_cur_compress_if_useful`（页数据低于 `merge_threshold` 即合并兄弟页释放本页）；spatial 索引有额外保护：本页是最后一页且持有谓词页锁时**跳过** purge（`row0purge.cc:535-551`）。

#### 唯一性检查为什么必须加锁

`row_ins_scan_sec_index_for_duplicate`（`row0ins.cc:1876-2054`）的核心顺序：**先加锁、后判重复**：

```cpp
pcur.open(index, 0, entry, PAGE_CUR_GE, BTR_SEARCH_LEAF, mtr, ...);
do {
  ...
  /* 对匹配记录加锁：RC 下 LOCK_REC_NOT_GAP；supremum → LOCK_ORDINARY；
     is_next（下一条比 entry 大）→ LOCK_GAP；否则（相等）→ LOCK_ORDINARY */
  err = row_ins_set_rec_lock(LOCK_S, lock_type, block, rec, index, offsets, thr);
  ...
  if (!is_next && !index->allow_duplicates) {
    if (row_ins_dupl_error_with_rec(rec, entry, index, offsets)) {
      err = DB_DUPLICATE_KEY;
      ...
    }
  } else goto end_scan;
} while (pcur.move_to_next(mtr));
```

`row_ins_dupl_error_with_rec` 的前提注释（1826-1828）："**we assume that the caller already has a record lock on the record!**"——delete-mark 的排除（`rec_get_deleted_flag(...) == 0` 才报冲突）发生在**拿到锁之后**。

为什么不能靠 MVCC：

1. MVCC 只能判断"已存在的历史版本"是否可见，**无法阻止另一个活跃事务即将插入同值**——唯一性必须是"当前 + 未来"的保证；
2. **delete-mark 项必须先加锁再判断**：一条 delete-marked 的唯一索引项可能属于一个**尚未提交**的删除事务（回滚后它又活了）。若不加锁就判定"已删、可插入"，会产生幻象唯一性冲突；
3. 聚簇索引版注释（`row0ins.cc:2160-2166`）补刀："unique non-clustered indexes 上可以存在**任意多条同 key 的 delete-marked 记录**（remember multiversioning），它们只差 row reference 部分——为避免竞态，必须**先插入、再检查**唯一性不被破坏"；
4. RC（skip_gap_locks）只是退化为 `LOCK_REC_NOT_GAP`/跳过 supremum 的 gap 锁，**record S 锁仍然要加**——RC/RR 的差别只在 gap 部分。

#### 长事务与索引记录膨胀

机制链条：每条 `TRX_UNDO_DEL_MARK_REC` / 改索引列的 `TRX_UNDO_UPD_EXIST_REC` 在被 purge 前，对应的二级索引项**一直以 delete-marked 形式留在 B-tree 中**（占槽位与锁资源）；purge 推进受最老 ReadView 约束；`row_vers_old_has_index_entry` 的判定也是"trx id >= purge view"。

**后果**：长事务 / 大 history list ⇒ 大量 delete-marked 二级索引项堆积 ⇒ 二级索引页膨胀、扫描需跳过更多记录、`PAGE_GARBAGE` 升高（未到 `merge_threshold` 不会自动回收）。

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `PAGE_MAX_TRX_ID` | 二级索引页的可见性粗筛字段（不写 redo） |
| `innodb_purge_threads` / `innodb_purge_batch_size` | purge 并发与批量 |
| `index->merge_threshold` = 50 | delete-mark 堆积页的回收阈值 |
| `srv_stats_include_delete_marked` | 统计是否包含 delete-marked 记录 |
| `history list length` | `SHOW ENGINE INNODB STATUS` 里，长事务的间接指标 |

---

## Misc

### 易混淆点

- **"删除数据后索引不缩小"的真相**：二级索引项在 purge 前**一直存在**（delete-mark 不是物理删除）；聚簇记录同理。
- **ICP 在二级索引上判断的是当前版本**——不匹配跳过是"保守剪枝"，匹配了也可能回表后被 old_vers 否决。
- **`row_vers_old_has_index_entry` 用 collation 相等而非二进制相等**——防止 CHAR 列改写但 collation 相等时误删索引项。
- **唯一性检查在 RC/RR 下都必须加锁**，差别只在 gap 锁部分（RC 跳过 gap 锁）。
- **purge 会触发页合并（`btr_cur_compress_if_useful`），不会触发分裂**。
- **purge 用 `node->ref`（主键值）重新定位，不用 undo 里的物理位置**——记录可能已被页分裂搬走。
- **聚簇索引与二级索引的删除语义不对称**：聚簇记录物理删除后其二级索引项还要逐个处理（先二级、后聚簇的顺序保证"聚簇没了索引还在"的窗口最小化）。

### 三条交叉线的总图

```
        ┌─ 读：PAGE_MAX_TRX_ID 粗筛 →（不过）回表走聚簇版本链
MVCC ───┤
        └─ 写：删除 = delete-mark（两阶段的第一阶段）
                 ↑ 引用计数式的判定：row_vers_old_has_index_entry 回溯版本链

        ┌─ 定位：undo → node->ref（PK）→ 重新搜索定位（不用物理位置）
purge ──┤
        └─ 删除：row_purge_poss_sec 判定 → 乐观（可进 ibuf）/ 悲观（可能页合并）

        ┌─ 唯一性：先加锁（S/next-key）再判重——MVCC 管不了"未来"
约束 ───┴─ 外键等：见 constraint.md
```

### 一句话总结

二级索引"没有版本信息"这一个事实，派生出了整片交叉地带的全部机制：可见性靠页级粗筛 + 回表、删除靠 delete-mark 两阶段 + 版本链回溯判定、purge 靠主键重新定位 + 水位约束、唯一性靠锁而非 MVCC——**索引承担的是"位置"，事务承担的是"时间"，两者在每一条记录上交接**。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.3 InnoDB Multi-Versioning*
- *MySQL 8.0 Reference Manual → 15.8.10 Purge Configuration*

**相关文档**
- MVCC/ReadView/可见性判定：[`../innodb/mvcc.md`](../innodb/mvcc.md)
- 行锁/gap/隐式锁：[`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- undo 与 purge 整体：[`../innodb/undo_log.md`](../innodb/undo_log.md)
- 唯一性检查完整链路（含外键）：[`constraint.md`](constraint.md)
- delete-mark 与页合并：[`btr.md`](btr.md)
- change buffer 与 purge 的配合（`BTR_DELETE` 模式）：[`ibuf.md`](ibuf.md)
- 取行主链 `row_search_mvcc`：[`../innodb/row_search.md`](../innodb/row_search.md)

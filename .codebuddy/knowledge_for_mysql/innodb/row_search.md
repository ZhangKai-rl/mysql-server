# InnoDB 行读取：row_search_mvcc 与游标推进

> 基于 MySQL 8.0.39 源码。承接 `server/handler.md` 的 handler 接口，讲 **InnoDB 侧**如何用持久游标（pcur）定位与推进、逐行返回记录。
>
> **边界**：SQL 层如何调 handler 见 [`../server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)；handler 接口语义（`position`/`ref`/`rnd_pos`）见 [`../server/handler.md`](../server/handler.md)；可见性判断（read view、版本回溯）见 [`mvcc.md`](mvcc.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [从 handler 到 row_search_mvcc](#从-handler-到-row_search_mvcc)
    - [行格式：rec_t 磁盘行与 record[0] 的转换](#行格式rec_t-磁盘行与-record0-的转换)
  - 行为与游标
    - [不同隔离级别下的表现](#不同隔离级别下的表现)
    - [direction 与游标推进](#direction-与游标推进)
    - [need_to_process](#need_to_process)
  - 读路径
    - [MVCC 可见性：快照读与二级索引回表](#mvcc-可见性快照读与二级索引回表)
    - [B-tree 定位：btr_cur_search_to_nth_level](#b-tree-定位btr_cur_search_to_nth_level)
    - [持久游标：store_position / restore_position](#持久游标store_position--restore_position)
  - 加锁
    - [加锁：sel_set_rec_lock 与隐式锁转显式](#加锁sel_set_rec_lock-与隐式锁转显式)
- [关键源码位置速查](#关键源码位置速查)
- [参考](#参考)

---

## 概述

### 是什么

SQL 层每次 `iterator->Read()` 穿过 handler 进入 `row_search_mvcc`，后者靠**持久游标（pcur）**定位 + 推进，每次返回一行。

### 与 mvcc.md 的边界

- `mvcc.md`：讲**可见性**——read view、版本回溯、读哪个版本
- 本文：讲**执行流程**——怎么走到游标、怎么推进、位置失效怎么恢复

---

## 理论基础

### row_search_mvcc：为什么是"怪兽函数"

`row_search_mvcc` 是 InnoDB 里最复杂的函数之一。`row0sel.h` 里三个并列函数的注释本身就是"复杂度分层"的最强证据：

- `row_search_no_mvcc`：*"Function is for temporary tables that are not shared across connections and so lot of complexity is reduced especially locking and transaction related."* —— 临时表/内部表走这条路，**砍掉了锁和事务复杂度**
- `row_search_mvcc`：*"mainly used for tables that are shared accorss connection and so it employs technique that can help re-construct the rows that transaction is suppose to see."* —— 共享表要走这条路

同一个函数体内，按 PHASE 注释处理至少这些模式：

| 模式 | 判定 |
|---|---|
| 唯一等值点查 | `unique_search` 标志 |
| 快照一致读（LOCK_NONE） | `select_lock_type == LOCK_NONE` → `trx_assign_read_view` |
| 加锁读 FOR UPDATE | `sel_set_rec_lock` |
| 二级索引扫描 + 回表 | `requires_clust_rec` 标签 |
| 全表扫 | `open_at_side(PAGE_CUR_G)` |
| 半一致读 | `use_semi_consistent` |
| 空间索引 R-tree | `dict_index_is_spatial` |
| AHI 捷径 | `row_sel_try_search_shortcut_for_mysql` |
| ICP | `row_search_idx_cond_check` |
| 行预取 | `can_prefetch_records` |

**为什么挤在一个函数里**：

1. **刻意分层**——"共享表 vs 临时表"已拆成 `row_search_mvcc` vs `row_search_no_mvcc`。共享表把 MVCC、行锁、二级索引回表、外存 BLOB、空间索引、预取、AHI 全堆在一起，才成了怪兽函数
2. **历史包袱**——函数头注释保留着 1997 年 Heikki Tuuri 的原味（"accorss" 拼写错误），主循环是 `goto` 标签串起的状态机（`rec_loop` / `next_rec` / `requires_clust_rec` / `lock_wait_or_error`），是长期增量演化的产物而非一次性设计
3. **为什么不能拆成多个小函数**——主循环的状态（`pcur` 位置、`mtr` 持有的 page latch、`did_semi_consistent_read`、`offsets` 归属的 rec 是聚簇还是二级索引、`moves_up` 方向）相互纠缠，尤其"恢复游标后是否要重处理当前记录"（`need_to_process`）跨越整个"取行→判定→加锁/回溯→返回→续读"流程，拆函数必然要引入新的状态对象

### 函数签名与入参（逐参数）

```cpp
dberr_t row_search_mvcc(byte *buf, page_cur_mode_t mode,
                        row_prebuilt_t *prebuilt, ulint match_mode,
                        const ulint direction)
```

| 参数 | 含义与取值 |
|---|---|
| `buf` | out：MySQL 行格式（server 层 record 布局）的输出缓冲 |
| `mode` | B-tree 搜索模式：`PAGE_CUR_G`(>) / `GE`(>=) / `L`(<) / `LE`(<=) / `CONTAIN`/`WITHIN`(R-tree) |
| `prebuilt` | in/out：总上下文。关键字段：`index`（本次索引）、`search_tuple`（定位键）、`pcur`（持久游标）、`select_lock_type`（LOCK_NONE 快照读 / LOCK_S / LOCK_X）、`mysql_template`（行转换模板）、`fetch_cache`（预取缓存） |
| `match_mode` | `0`=范围扫描；`ROW_SEL_EXACT`=等值点查；`ROW_SEL_EXACT_PREFIX`=前缀匹配 |
| `direction` | `0`=打开/定位游标（首次）；`ROW_SEL_NEXT`=取下一行；`ROW_SEL_PREV`=取上一行。`!=0` 时要求 `pcur` 已 `store_position` 过 |

### PHASE 1：记录缓冲 / 预取缓存（逐行）

```cpp
if (direction == 0) {
  trx->op_info = "starting index read";
  prebuilt->n_rows_fetched = 0;
  prebuilt->n_fetch_cached = 0;
  prebuilt->fetch_cache_first = 0;
  prebuilt->m_end_range = false;
  if (record_buffer != nullptr) record_buffer->reset();
  if (prebuilt->sel_graph == nullptr) row_prebuild_sel_graph(prebuilt);
} else {
  trx->op_info = "fetching rows";
  ...
  if (prebuilt->n_fetch_cached > 0) {
    row_sel_dequeue_cached_row_for_mysql(buf, prebuilt);   // ★ 缓存命中：直接弹出返回
    prebuilt->n_rows_fetched++;
    err = DB_SUCCESS;
    goto func_exit;
  } else if (prebuilt->m_end_range) {
    err = DB_RECORD_NOT_FOUND;
    goto func_exit;
  }
  ...
  mode = pcur->m_search_mode;                              // 沿用游标保存的搜索模式
}
```

- `direction == 0`（首次）：清零计数/缓存/结束标志；`row_prebuild_sel_graph` 构建哑 select 查询图——不是为了执行 Query Graph，而是为了拿到 `que_thr_t` 线程上下文（后续 `lock_table`、`row_mysql_handle_errors` 都要用）
- `direction != 0`（续读）：**缓存命中时直接 `row_sel_dequeue_cached_row_for_mysql` 弹出一行返回，完全不碰 B-tree**——这是预取优化的命中路径
- **为什么要缓存**：主循环每取一行都要「定位 B-tree 页 → 加 page latch → 转行 → 提交 mtr 释放 latch」。若一次 mtr 内能连续转多行放进缓存，之后若干次调用直接从内存弹行，省掉反复 latch/unlatch。`MYSQL_FETCH_CACHE_THRESHOLD = 4`（只有 `n_rows_fetched >= 4` 确认为长范围扫描才启用）、`MYSQL_FETCH_CACHE_SIZE = 8`（缓存上限）

### unique_search 判定（进入 PHASE 2 前）

```cpp
if (match_mode == ROW_SEL_EXACT && dict_index_is_unique(index) &&
    dtuple_get_n_fields(search_tuple) == dict_index_get_n_unique(index) &&
    (index->is_clustered() || !dtuple_contains_null(search_tuple))) {
  unique_search = true;
  ...
}
```

四个条件：等值点查 + 索引唯一 + 键字段数等于唯一列数（给了完整唯一键）+（聚簇索引 或 二级键不含 NULL）。二级唯一索引可能含多个 NULL（NULL != NULL），所以要求键中无 NULL。

### PHASE 2：AHI 自适应哈希捷径

```cpp
if (direction == 0 && unique_search && btr_search_enabled &&
    index->is_clustered() && !prebuilt->templ_contains_blob &&
    !prebuilt->used_in_HANDLER &&
    (prebuilt->mysql_row_len < UNIV_PAGE_SIZE / 8) && !prebuilt->innodb_api) {
  mode = PAGE_CUR_GE;
  if (trx->mysql_n_tables_locked == 0 && !prebuilt->ins_sel_stmt &&
      prebuilt->select_lock_type == LOCK_NONE &&
      trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
      MVCC::is_view_active(trx->read_view)) {
    ...
    switch (row_sel_try_search_shortcut_for_mysql(&rec, ...)) {
      case SEL_FOUND:    row_sel_store_mysql_rec(...); goto func_exit;   // 命中
      case SEL_EXHAUSTED: ... DB_RECORD_NOT_FOUND; goto func_exit;        // 确定无此行
      case SEL_RETRY:    break;                                           // 回退正常流程
    }
  }
}
```

- **外层 8 条件**：首次定位 + 唯一点查 + AHI 开 + 聚簇索引 + 无 BLOB（外存字段需释放 search latch，与 AHI 冲突）+ 非 HANDLER 滚动游标 + 行足够短 + 非 innodb_api
- **内层 5 条件**（快照读前提）：非 INSERT...SELECT + 非内部 select + LOCK_NONE + 隔离级别 > RU + read view 活跃
- `row_sel_try_search_shortcut_for_mysql` 三态：`SEL_FOUND`（AHI 命中，可见且非 delete-mark，直接转换返回）；`SEL_EXHAUSTED`（`up_match < n_fields` 键不存在，或 delete-mark，确定无此行）；`SEL_RETRY`（不可见需要 undo 回溯，或定位到非 user record，回退正常流程）
- ★ **捷径路径不 `store_position`**——AHI 快照读是一次性的，不需要给后续 NEXT 保存位置

### PHASE 3：打开 vs 恢复游标（核心分支）

```cpp
  if (direction != 0) {
    auto need_to_process = sel_restore_position_for_mysql(
        &same_user_rec, BTR_SEARCH_LEAF, pcur, moves_up, &mtr);
    if (need_to_process) {
      /* 恢复后游标已校正到新记录，直接 rec_loop 处理当前记录 */
    } else if (row_read_type != ROW_READ_DID_SEMI_CONSISTENT) {
      goto next_rec;   /* 游标停在"上次已返回的那条"上，先前进 */
    }
  } else if (dtuple_get_n_fields(search_tuple) > 0) {
    pcur->open_no_init(index, search_tuple, mode, BTR_SEARCH_LEAF, 0, &mtr, ...);
    rec = pcur->get_rec();
    /* 降序扫描 + gap 锁：对逻辑后继加 LOCK_GAP（防止 DESC 幻读） */
    ...
  } else if (mode == PAGE_CUR_G || mode == PAGE_CUR_L) {
    pcur->open_at_side(mode == PAGE_CUR_G, index, BTR_SEARCH_LEAF, false, 0, &mtr);
  }
```

- **`direction != 0`（恢复游标）**：`sel_restore_position_for_mysql` 返回 `need_to_process`——true = 恢复后游标已前进到"下一条待处理记录"（原记录被删/换页，函数内部已校正），直接落进 `rec_loop`；false = 游标停在"上次已返回的那条"上，必须 `goto next_rec` 先前进（除非是 DID_SEMI_CONSISTENT 需要悲观重读同一行）
- **`direction == 0 && search_tuple 有字段`（按键定位）**：`open_no_init` 从根下到叶子按 mode 定位；降序扫描（`!moves_up`，定位到 range 上界）时，幻影会从定位点"左边"插入，所以对逻辑后继加 `LOCK_GAP`
- **`direction == 0 && mode == G/L`（全表扫）**：`open_at_side` 打开到索引最左（G）/最右（L）叶子
- **`moves_up` 推导**：`direction == 0` 时 `GE/G/CONTAIN` → true（升序）；`direction == ROW_SEL_NEXT` → true；其余（PREV、或 0 且 L/LE 降序）→ false

**`sql_stat_start` 三态**（在打开游标前）：

```cpp
  if (!prebuilt->sql_stat_start) {
    /* 语句已开始：一致读必须有 read view（防御断言） */
  } else if (prebuilt->select_lock_type == LOCK_NONE) {
    trx_assign_read_view(trx);      // 快照读：分配 read view
    prebuilt->sql_stat_start = false;
  } else {
    err = lock_table(0, index->table, LOCK_S ? LOCK_IS : LOCK_IX, thr);   // 锁定读：加意向锁
    ...
  }
```

### PHASE 4：rec_loop 主循环（逐行，最重要）

**循环入口与中断检查**：

```cpp
rec_loop:
  if (trx_is_interrupted(trx)) {
    if (!spatial_search) pcur->store_position(&mtr);
    err = DB_INTERRUPTED;
    goto normal_return;
  }
  rec = pcur->get_rec();          // ★ 取游标当前指向的记录，本轮的处理对象
```

**infimum / supremum 检查**：

```cpp
  if (page_rec_is_infimum(rec)) { prev_rec = nullptr; goto next_rec; }
  if (page_rec_is_supremum(rec)) {
    // ① end_range 优化：用上一页最后一条 prev_rec 提前判断是否越界（end_loop>=100 节流）
    // ② supremum 加 gap 锁：LOCK_ORDINARY 锁住"页末尾之后的间隙"
    if (set_also_gap_locks && !trx->skip_gap_locks() && select_lock_type != LOCK_NONE && ...)
      sel_set_rec_lock(pcur, rec, index, offsets, ..., LOCK_ORDINARY, thr, &mtr);
    prev_rec = nullptr;
    goto next_rec;                 // supremum 永不进结果
  }
```

- infimum（页内最小哨兵）永不进结果、永不加锁，直接跳过
- supremum（页末尾哨兵）两步：① **end_range 优化**——扫描跨页且 server 下推了 `end_range` 时，用 `prev_rec` 提前判越界避免扫完整棵树；② **加 gap 锁**——对 supremum 加 `LOCK_ORDINARY`（next-key）锁住"页内最后一条之后"的间隙，阻止其他事务在页末尾插入

**页损坏检查**：`rec_get_next_offs` 读记录头里的 next 偏移，落在 `[PAGE_NEW_SUPREMUM, UNIV_PAGE_SIZE - PAGE_DIR)` 之外即页损坏。`srv_force_recovery > 0` 且升序时跳过本页剩余（抢救数据），否则报 `DB_CORRUPTION`。

**offsets 与 match_mode 过滤**：

```cpp
  offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED, ...);
  ...
  if (match_mode == ROW_SEL_EXACT) {
    if (0 != cmp_dtuple_rec(search_tuple, rec, index, offsets)) {
      /* 不匹配：锁定读先加 LOCK_GAP（锁住匹配键应当落下的间隙）*/
      sel_set_rec_lock(..., LOCK_GAP, ...);
      pcur->store_position(&mtr);
      pcur->m_rel_pos = BTR_PCUR_BEFORE;    // 这条不匹配记录将来作 index_next 起点
      err = DB_RECORD_NOT_FOUND;
      goto normal_return;
    }
  } else if (match_mode == ROW_SEL_EXACT_PREFIX) { ... /* 前缀同理 */ }
```

- ★ 注释强调：**此处不能信任游标的 `up_match`**（游标可能已 move 过绕回），必须重新 `cmp_dtuple_rec`
- 不匹配时 `m_rel_pos = BTR_PCUR_BEFORE`——这条记录将来可作 `index_next` 的起始，游标语义是"位于它之前"

**锁定读分支**（`select_lock_type != LOCK_NONE`）：

```cpp
    auto row_to_range_relation = row_compare_row_to_range(
        set_also_gap_locks, trx, unique_search, index, clust_index, rec, ...);
    ulint lock_type;
    if (row_can_be_in_range) {
      lock_type = gap_can_intersect_range ? LOCK_ORDINARY : LOCK_REC_NOT_GAP;
    } else {
      lock_type = gap_can_intersect_range ? LOCK_GAP : (DB_RECORD_NOT_FOUND);
    }

    const bool use_semi_consistent =
        row_read_type == ROW_READ_TRY_SEMI_CONSISTENT &&
        !unique_search && index == clust_index && !trx_is_high_priority(trx);
    err = sel_set_rec_lock(pcur, rec, index, offsets,
                           use_semi_consistent ? SELECT_SKIP_LOCKED : select_mode,
                           select_lock_type, lock_type, thr, &mtr);
```

- **锁类型三选一**（`row_compare_row_to_range` 算出记录与范围的关系）：记录在范围内 + 间隙相交 → `LOCK_ORDINARY`（next-key，默认 RR）；记录在范围内 + 间隙不相交（唯一点查）→ `LOCK_REC_NOT_GAP`；记录不在范围 + 间隙相交 → `LOCK_GAP`
- `use_semi_consistent` 四条件：TRY_SEMI + 非唯一点查 + 聚簇索引 + 非高优先级。启用时用 `SELECT_SKIP_LOCKED`——**遇到被锁记录返回 `DB_SKIP_LOCKED` 而不是阻塞建 WAITING 锁**
- `DB_SKIP_LOCKED` 两分支：显式 `SKIP LOCKED` → `goto next_rec`；半一致读 → `row_sel_build_committed_vers_for_mysql` 用 undo 构建**最新已提交版本**，`old_vers == nullptr`（未提交新插入）→ `next_rec`，否则 `rec = old_vers` 返回已提交版本
- `DB_LOCK_WAIT` → 清空 `new_rec_lock`，`goto lock_wait_or_error`

**快照读分支**（`select_lock_type == LOCK_NONE`）：

```cpp
  } else {
    if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
      /* 读未提交：直接读最新版本，什么都不做 */
    } else if (index == clust_index) {
      if (!lock_clust_rec_cons_read_sees(rec, index, offsets, trx_get_read_view(trx))) {
        row_sel_build_prev_vers_for_mysql(...);   // 不可见 → undo 回溯
        if (old_vers == nullptr) goto next_rec;    // read view 中还不存在
        rec = old_vers;
      }
    } else {
      /* 二级索引：记录不含完整版本链，不可信必须回表 */
      if (!lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
        switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
          case ICP_NO_MATCH: goto next_rec;          // ICP 不满足，省一次回表
          case ICP_OUT_OF_RANGE: ... goto idx_cond_failed;
          case ICP_MATCH: goto requires_clust_rec;
        }
      }
    }
  }
```

**delete-mark 跳过**：

```cpp
  if (rec_get_deleted_flag(rec, comp)) {
    prebuilt->try_unlock(true);     // 低隔离级别下释放刚加的锁
    if (index == clust_index && unique_search && !used_in_HANDLER) {
      err = DB_RECORD_NOT_FOUND;    // 唯一键下不可能有"第二条同键记录"
      goto normal_return;
    }
    goto next_rec;
  }
```

**ICP 三态**：`ICP_NO_MATCH`（索引上就不满足，跳过）/ `ICP_OUT_OF_RANGE`（越界，`goto idx_cond_failed`）/ `ICP_MATCH`（继续）。

**二级回表**（`requires_clust_rec` 标签）：

```cpp
  if (index != clust_index && prebuilt->need_to_access_clustered) {
  requires_clust_rec:
    mtr_has_extra_clust_latch = true;      // 回表在同一 mtr 里又 latch 聚簇页
    err = row_sel_get_clust_rec_for_mysql(prebuilt, index, rec, thr,
                                          &clust_rec, &offsets, &heap, ...);
    ...
    result_rec = clust_rec;                // 最终返回的是聚簇记录（或其旧版本）
  } else {
    result_rec = rec;
  }
```

- `mtr_has_extra_clust_latch`：回表在**同一个 mtr 里又 latch 聚簇页**，推进前必须 commit mtr 再 restart（避免持有二级页 latch 时 latch 聚簇页破坏 latch 顺序）
- `row_sel_get_clust_rec_for_mysql`：① 用二级记录的主键列构建 `clust_ref` ② 定位聚簇记录 ③ 锁定读加 `LOCK_REC_NOT_GAP`（等值回表不需 gap）④ 快照读不可见则 undo 回溯 + `row_sel_sec_rec_is_for_clust_rec` 校验旧版本是否真对应原二级记录

**转换 + 预取缓存 + 保存位置**：

```cpp
  row_sel_store_mysql_rec(buf, prebuilt, result_rec, vrow, ...);   // rec_t → record[0]

  if (record_buffer != nullptr ||
      ((match_mode == ROW_SEL_EXACT || n_rows_fetched >= MYSQL_FETCH_CACHE_THRESHOLD) &&
       prebuilt->can_prefetch_records())) {
    /* 缓存路径：入队，n_fetch_cached < max 则 goto next_rec 继续填 */
  } else {
    /* 直接返回单行 */
  }

  err = DB_SUCCESS;
  if (!unique_search || !index->is_clustered() || direction != 0 ||
      select_lock_type != LOCK_NONE || used_in_HANDLER || innodb_api) {
    pcur->store_position(&mtr);     // ★ 何时存位置
  }
  goto normal_return;
```

- ★ **缓存路径的关键行为**：函数不会立即返回，而是继续在**同一个 mtr/page latch 内**多抓几行填满缓存，直到缓存满才返回首行
- ★ **何时存位置（核心优化）**：唯一聚簇点查 + 一致读 + `direction == 0` + 非 HANDLER + 非 innodb_api 时**不存**——这种查询 fetch next/prev 必然 EOF，存了也白存

### PHASE 5：next_rec 推进

```cpp
next_rec:
  end_loop++;
  if (row_read_type == ROW_READ_DID_SEMI_CONSISTENT)
    row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;   // 半一致读标志是一次性的
  did_semi_consistent_read = false;
  prebuilt->new_rec_lock.reset();
  vrow = nullptr;

  if (mtr_has_extra_clust_latch || spatial_search) {
    /* 回表 latch 过聚簇页：必须 store → commit mtr → restart → restore */
    pcur->store_position(&mtr);
    mtr_commit(&mtr);
    mtr_has_extra_clust_latch = false;
    mtr_start(&mtr);
    if (sel_restore_position_for_mysql(&same_user_rec, BTR_SEARCH_LEAF, pcur,
                                       moves_up, &mtr)) {
      prev_rec = nullptr;
      goto rec_loop;               // 恢复后已校正到新记录，直接处理
    }
  }

  if (moves_up) {
    if (!pcur->move_to_next(&mtr)) {       // 已到索引尽头
      pcur->store_position(&mtr);
      err = (match_mode != 0) ? DB_RECORD_NOT_FOUND : DB_END_OF_INDEX;
      goto normal_return;
    }
  } else {
    if (!pcur->move_to_prev(&mtr)) goto not_moved;
  }
  goto rec_loop;                   // 新一轮处理下一条
```

- ★ **mtr 重启**：回表后（`mtr_has_extra_clust_latch`）推进前必须 `store_position → mtr_commit → mtr_start → restore_position`——持有二级页 latch 时访问聚簇页会破坏 latch 顺序
- **状态复位**：半一致读标志是一次性的、`new_rec_lock` 复位、`vrow` 置空
- 推进失败（索引尽头）：点查/前缀查 → `DB_RECORD_NOT_FOUND`；纯范围扫描 → `DB_END_OF_INDEX`

### lock_wait_or_error（锁等待/错误汇聚）

任何加锁失败/锁等待都汇聚到这里。先 `pcur->store_position`（**锁等待前必须保存位置**，唤醒后才能 restore 回冲突记录重试），`mtr_commit` 释放所有页 latch（不能带着 latch 去 sleep），`row_mysql_handle_errors` 返回 true（锁等待结束）则 restore 位置后 `goto rec_loop` 重试；返回 false（死锁/超时/被 kill）则 `goto func_exit`。

### 主循环状态流转图

```
                         ┌──────────────────────────────────────────┐
                         │          row_search_mvcc 入口            │
                         └──────────────────────────────────────────┘
                                         │
             ┌───────────────────────────┼──────────────────────────────┐
             │ direction==0              │ direction!=0                 │
             ▼                           ▼                              │
    [PHASE1 清零/建 sel_graph]   [PHASE1 尝试弹缓存行]──命中──► func_exit │
             │                           │ 未命中                        │
             ▼                           ▼                              │
    [unique_search 判定]        [mtr_start]                             │
             │                           │                              │
             ▼                           ▼                              │
    [PHASE2 AHI 捷径] ──FOUND/EXHAUSTED──► func_exit(成功/未找到)        │
             │ RETRY                                                   │
             ▼                                                          │
    [PHASE3 open_no_init / open_at_side / restore_position]             │
             │                                                          │
             ▼                                                          │
    ╔═══════════════════════════════════════════════════════════════════╪══╗
    ║                      rec_loop 主循环                              │  ║
    ║  中断?──►store_position──►normal_return                           │  ║
    ║   rec=get_rec()                                                  │  ║
    ║   ├ infimum ─────────────────────────► next_rec                  │  ║
    ║   ├ supremum ──加gap锁──► next_rec                               │  ║
    ║   ├ match_mode 过滤(不等→加gap锁→NOT_FOUND)                      │  ║
    ║   ├ 锁定读: sel_set_rec_lock                                     │  ║
    ║   │    ├ SUCCESS ───────────────► 继续                           │  ║
    ║   │    ├ SKIP_LOCKED(显式) ─────► next_rec                       │  ║
    ║   │    ├ SKIP_LOCKED(半一致)──► build_committed_vers             │  ║
    ║   │    └ LOCK_WAIT ─────────────► lock_wait_or_error             │  ║
    ║   ├ 快照读: 可见性判断                                          │  ║
    ║   │    ├ 聚簇不可见 → build_prev_vers: null→next_rec else替换rec│  ║
    ║   │    └ 二级不可见 → ICP过滤 → requires_clust_rec               │  ║
    ║   ├ delete-mark? ──►(唯一点查→NOT_FOUND) else next_rec           │  ║
    ║   ├ ICP: NO_MATCH→next_rec / OUT_OF_RANGE→idx_cond_failed        │  ║
    ║   ├ 二级回表: row_sel_get_clust_rec_for_mysql                    │  ║
    ║   ├ row_sel_store_mysql_rec 转换                                 │  ║
    ║   ├ 预取缓存? ──入队→缓存未满→next_rec                           │  ║
    ║   └ store_position → goto normal_return                          │  ║
    ╚═══════════════════════════════════════════════════════════════════╪══╝
                                                                      │
    next_rec: 状态复位 / mtr重启(回表后) / move_to_next(prev)          │
    │ 失败→not_moved→store_position→NOT_FOUND/END_OF_INDEX             │
    └──► goto rec_loop ◄───────────────────────────────────────────────┘

    lock_wait_or_error: store_position→mtr_commit→handle_errors
         ├ 锁等待结束→(表锁→wait_table_again / 行锁→restore→rec_loop)
         └ 不可重试→func_exit
```

| goto 标签 | 含义 | 去向 |
|---|---|---|
| `rec_loop` | 主循环入口 | `next_rec` / `normal_return` / `lock_wait_or_error` |
| `next_rec` | 推进到下一记录 | 推进后 `goto rec_loop` |
| `requires_clust_rec` | 二级回表 | 回表后继续或 `next_rec` |
| `idx_cond_failed` | ICP 越界收尾 | 存位置 → `normal_return` |
| `normal_return` | 正常返回 | `func_exit` 返回 err |
| `lock_wait_or_error` | 加锁失败/等待 | 恢复→`rec_loop` 或 `func_exit` |

---

### 快照读 vs 当前读：一刀切成两条互斥路径

同一个函数里用 `prebuilt->select_lock_type == LOCK_NONE` 一刀切开：

- **快照读**（LOCK_NONE）：read view + undo 回溯，**不碰锁**——读不阻塞写、写不阻塞读（MVCC 非锁定一致读）
- **当前读**（LOCK_S / LOCK_X）：`sel_set_rec_lock` 加锁，**不回溯版本**，只看最新版本——保证"读最新 + 防止幻读"

这比"单一行读取函数"合理，因为两者共享同一套"游标定位/推进/回表/行格式转换"基础设施，只有"拿哪一版 + 加不加锁"这一步分叉。

### 与 MyISAM 的对比

MyISAM 无 MVCC、无行锁、无 undo，读就是读：`ha_myisam::index_next` 直接调 `mi_rnext`，内部 `flag = SEARCH_BIGGER`，按文件偏移 `info->lastpos` 取下一条。**没有可见性判断、没有版本回溯、没有回表、没有持久游标（只有文件偏移 `lastpos`）。** 这是"读一行"复杂度差距的根源——InnoDB 用复杂度换来了并发。

---

## 核心实现

### 从 handler 到 row_search_mvcc

```
（SQL 层）TableScanIterator::Read
  → table->file->ha_rnd_next(m_record)      handler.cc:2969     ← server/InnoDB 分界面
    → ha_innobase::rnd_next                 ha_innodb.cc:10833
      → general_fetch(buf, ROW_SEL_NEXT, 0) ha_innodb.cc:10530
        → row_search_mvcc(...)              ha_innodb.cc:10558
```

索引扫描同理，`ha_rnd_next` 换成 `ha_index_next`（`index_next` → `general_fetch`）。

### rnd_init：全表扫描初始化（逐行）

```cpp
int ha_innobase::rnd_init(bool scan) {
  assert(table_share->is_missing_primary_key() ==
         (bool)m_prebuilt->clust_index_was_generated);

  int err = change_active_index(table_share->primary_key);

  /* Don't use semi-consistent read in random row reads (by position). */
  if (!scan) {
    m_prebuilt->row_read_type = ROW_READ_WITH_LOCKS;
  }

  m_start_of_scan = true;
  return err;
}
```

逐行：

- **断言**：表缺主键 ⇔ InnoDB 侧聚簇索引是内部生成的。全表扫描一定走聚簇索引，这个断言保证后面选的主键就是那颗聚簇索引
- **`change_active_index(table_share->primary_key)`**：把活跃索引切到主键。用户没定义主键时 `primary_key` 是 `MAX_KEY`，`innobase_get_index(MAX_KEY)` 返回 `first_index()`（聚簇索引）。⇒ **全表扫描本质是"聚簇索引的顺序扫描"**
- **`if (!scan)`**：`scan=false` 是"按位置随机读"（`rnd_pos` 场景）。半一致读只在顺序扫描判定 WHERE 时需要，随机读直接禁用，回退 `ROW_READ_WITH_LOCKS`
- **`m_start_of_scan = true`**：这是 `rnd_next` 区分"首次定位"与"续读"的标志

**为什么 rnd_init 之后 rnd_next 能一行行取**：`change_active_index` 做了三件事——① `active_index`/`prebuilt->index` 指向聚簇索引；② `init_search_tuples_types()` 初始化 `search_tuple`/`m_stop_tuple` 字段类型；③ `build_template(false)` 建立 MySQL↔InnoDB 行转换模板。这些是取行的"基础设施"。

### index 选取：change_active_index

`index_init` 整个函数就是 `change_active_index(keynr)`（第二个参数 `sorted` 被忽略，InnoDB 不关心结果是否要求有序）。核心：

```cpp
active_index = keynr;                            // server 层选的索引号
m_prebuilt->index = innobase_get_index(keynr);   // keynr → dict_index_t*
...
m_prebuilt->init_search_tuples_types();          // 按新索引重设 search_tuple 类型
build_template(false);                           // 重建行转换模板
```

- `innobase_get_index(keynr)`：`keynr != MAX_KEY` 时按名字在 dict cache 找；`keynr == MAX_KEY` 时返回 `first_index()`（聚簇索引）
- **`active_index` 是 `handler` 基类字段**，但对 InnoDB 而言唯一赋值点在 `change_active_index`（`index_init` 覆盖了基类实现且没调基类）

**覆盖索引 vs 回表在 prebuilt 里的体现**：

| 字段 | 含义 |
|---|---|
| `read_just_key` | `HA_EXTRA_KEYREAD` 置 1 = 覆盖索引，不回表 |
| `need_to_access_clustered` | 二级索引但需要的列不在索引里 = 1，必须回表 |

server 层判定"只需索引列"时调 `HA_EXTRA_KEYREAD` → `read_just_key=1`，随后 `build_template` 只构建索引列映射，`need_to_access_clustered` 为 0，`row_search_mvcc` 就不回表。

### index_read：定位 + 读第一行

`index_read` 自己**不再选索引**（索引是 `index_init`/`rnd_init` 已选好的，它用 `m_prebuilt->index`）。关键点：

- **`key_ptr != nullptr`**：把 MySQL key 转成 InnoDB 的 `search_tuple`
- **`key_ptr == nullptr`**（`index_first`/`index_last`）：`search_tuple` 置 0 字段 = "定位到索引最左/最右"，方向由 mode 决定
- **`find_flag` → mode + match_mode**：`HA_READ_KEY_EXACT` → `ROW_SEL_EXACT`；`HA_READ_PREFIX_LAST` → `ROW_SEL_EXACT_PREFIX`；`m_last_match_mode` 记住它供 `index_next_same` 用
- ★ **`row_search_mvcc(buf, mode, m_prebuilt, match_mode, 0)` 的第 5 个参数 `direction=0` 是"首次定位"**——这就是 `index_read` 和 `index_next` 的本质区别：一个传 `0`，一个传 `ROW_SEL_NEXT/PREV`

### rnd_next：全表扫描下一行（逐行）

```cpp
int ha_innobase::rnd_next(uchar *buf) {
  if (m_user_thd->transaction_rollback_request) return HA_ERR_GENERIC;
  ha_statistic_increment(&System_status_var::ha_read_rnd_next_count);

  if (m_start_of_scan) {
    error = index_first(buf);                 // 首次：定位到聚簇索引第一条
    if (error == HA_ERR_KEY_NOT_FOUND) {
      error = HA_ERR_END_OF_FILE;             // 空表：rnd 接口约定用 END_OF_FILE
    }
    m_start_of_scan = false;
  } else {
    error = general_fetch(buf, ROW_SEL_NEXT, 0);   // 续读
  }
  return error;
}
```

逐行：

- **回滚检查**：server 层已请求回滚（被 kill），直接返回
- **`m_start_of_scan` 是首次/续读的分水岭**（在 `rnd_init` 被置 true）
- 首次 = `index_first`（内部 `index_read(buf, nullptr, 0, HA_READ_AFTER_KEY)` → `PAGE_CUR_G` 定位最左），空表把 `HA_ERR_KEY_NOT_FOUND` 映射成 `HA_ERR_END_OF_FILE`
- 续读 = `general_fetch(buf, ROW_SEL_NEXT, 0)`：restore 游标后向后移动一条

**与 `index_next` 的差异**：`rnd_next` 自己处理"首次要 `index_first`"，且扫描对象是聚簇索引；`index_next` 无首次/续读区分（假定 `index_read` 已定位），可在任意活跃索引上走。

### index_next / index_prev / index_first / index_last

```cpp
int ha_innobase::index_next(uchar *buf) {
  ha_statistic_increment(&System_status_var::ha_read_next_count);
  return general_fetch(buf, ROW_SEL_NEXT, 0);
}
```

- `index_next_same`：只多传 `m_last_match_mode`（`ROW_SEL_EXACT/PREFIX`），要求"下一条也必须匹配同一 key"，用于 ref/eq_ref 的多行同 key 扫描
- `index_prev`：传 `ROW_SEL_PREV`，用于 `ORDER BY ... DESC` 逆序扫描
- `index_first` = `index_read(buf, nullptr, 0, HA_READ_AFTER_KEY)`（定位最左）；`index_last` = `HA_READ_BEFORE_KEY`（定位最右，配合 `index_prev` 逆序）

### general_fetch：续读的统一入口（逐行）

```cpp
int ha_innobase::general_fetch(uchar *buf, uint direction, uint match_mode) {
  const trx_t *trx = m_prebuilt->trx;
  ...
  if (!intrinsic && TrxInInnoDB::is_aborted(trx)) {
    innobase_rollback(ht, m_user_thd, false);
    return convert_error_code_to_mysql(DB_FORCED_ABORT, 0, m_user_thd);
  }
  auto ret = innobase_srv_conc_enter_innodb(m_prebuilt);   // 进入并发控制
  ...
  if (!intrinsic) {
    ret = row_search_mvcc(buf, PAGE_CUR_UNSUPP, m_prebuilt, match_mode, direction);
  } else {
    ret = row_search_no_mvcc(...);                          // intrinsic 临时表
  }
  innobase_srv_conc_exit_innodb(m_prebuilt);
  switch (ret) {
    case DB_SUCCESS: error = 0; break;
    case DB_RECORD_NOT_FOUND: error = HA_ERR_END_OF_FILE; break;
    case DB_END_OF_INDEX: error = HA_ERR_END_OF_FILE; break;
    ...
  }
  return error;
}
```

逐行：

- **`mode = PAGE_CUR_UNSUPP`**：续读时游标已定位，不再需要 `PAGE_CUR_GE/G/L` 这类定位 mode；`direction` 才是要传的（`row_search_mvcc` 见 `PAGE_CUR_UNSUPP` 就走 restore 而非 open）
- **`DB_RECORD_NOT_FOUND` 和 `DB_END_OF_INDEX` 都映射 `HA_ERR_END_OF_FILE`**——与 `index_read` 映射 `HA_ERR_KEY_NOT_FOUND` 不同，这是"续读到头" vs "定位未命中"的语义区别
- 首读由 `index_read`（索引路径）或 `rnd_init`+`index_first`（全表路径）完成；`general_fetch` 是"已定位之后"的通用续读

---

### 不同隔离级别下的表现

### 三把开关

隔离级别在取行路径上只通过**三把开关**起作用，其余逻辑完全共享：

1. **`select_lock_type`**（`LOCK_NONE` 快照读 vs `LOCK_S`/`LOCK_X` 当前读）
2. **`trx->read_view` 的 active/closed 状态**（RR 复用 vs RC 重建 vs RU 不用）
3. **`skip_gap_locks()` / `allow_semi_consistent()` / `releases_non_matching_rows()`**——三者其实是同一件事：RU/RC 为 true，RR/SE 为 false

```cpp
bool skip_gap_locks() const {
  switch (isolation_level) {
    case READ_UNCOMMITTED:
    case READ_COMMITTED:  return (true);
    case REPEATABLE_READ:
    case SERIALIZABLE:    return (false);
  }
}
bool allow_semi_consistent() const { return (skip_gap_locks()); }
```

### 四个隔离级别逐一

**READ UNCOMMITTED**（脏读）

非锁定读**直接跳过可见性检查**：

```cpp
} else {
  /* This is a non-locking consistent read: if necessary, fetch
  a previous version of the record */
  if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
    /* Do nothing: we let a non-locking SELECT read the
    latest version of the record */
  } else if (index == clust_index) { ... }
```

RU 直接返回当前记录（可能是未提交的最新版本）。AHI 捷径也要求 `isolation_level > TRX_ISO_READ_UNCOMMITTED`（RU 无 view 不走捷径）。`skip_gap_locks()==true`，锁定读不加 gap。

**READ COMMITTED**（read view 每语句重建 + 半一致读）

read view **每条语句重建**：语句结束时 `external_lock` 里 `<= RC` 则 `view_close`；下一条语句 `trx_assign_read_view` 看到 `!MVCC::is_view_active` → `view_open` 新建。

**半一致读只在 RC 生效**：`allow_semi_consistent() == skip_gap_locks()`，RU/RC 为 true。半一致读 = UPDATE/DELETE 扫描时目标行被别的事务锁住，先读**最近已提交版本**判断 WHERE 是否匹配，不匹配就跳过而不用等待：

```cpp
case DB_SKIP_LOCKED:
  ut_a(use_semi_consistent);
  ut_a(trx->allow_semi_consistent());
  row_sel_build_committed_vers_for_mysql(...);   // 读最新已提交版本，而非 read view 版本
  if (old_vers == nullptr) goto next_rec;         // 未提交的新插入
  did_semi_consistent_read = true;
  rec = old_vers;
```

**REPEATABLE READ（默认）**（快照固定）

read view **事务内复用**：语句结束**不**关闭（关闭条件 `<= RC` 不成立），`is_view_active` 一直 true → `trx_assign_read_view` 不重建 → 整个事务用同一快照：

```cpp
ReadView *trx_assign_read_view(trx_t *trx) {
  if (srv_read_only_mode) return (nullptr);
  else if (!MVCC::is_view_active(trx->read_view))
    trx_sys->mvcc->view_open(trx->read_view, trx);   // 只有 view 不活跃才重建
  return (trx->read_view);
}
```

`skip_gap_locks()==false`，锁定读加 next-key/gap 锁防幻读。

**SERIALIZABLE**（普通 SELECT 转当前读）

普通 SELECT 在**显式事务内**被提升为 `LOCK_S` 当前读：

```cpp
if (lock_type == F_RDLCK) {
  ...
  } else if (trx->isolation_level == TRX_ISO_SERIALIZABLE &&
             m_prebuilt->select_lock_type == LOCK_NONE &&
             thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
    m_prebuilt->select_lock_type = LOCK_S;    // ★ 普通 SELECT → 当前读
    m_stored_select_lock_type = LOCK_S;
  }
}
```

注意 AUTOCOMMIT=1 的一致读仍是 `LOCK_NONE`（只读事务无需串行化锁也能串行化）。`skip_gap_locks()==false`，加 gap 锁。

### 半一致读的四条件

```cpp
const bool use_semi_consistent =
    prebuilt->row_read_type == ROW_READ_TRY_SEMI_CONSISTENT &&
    !unique_search && index == clust_index && !trx_is_high_priority(trx);
```

1. `row_read_type == TRY`——前提 `allow_semi_consistent()`（= RC/RU）
2. `!unique_search`——唯一点查锁冲突应直接等待/报错，不适合跳过
3. `index == clust_index`——半一致读只对聚簇记录生效
4. `!trx_is_high_priority`——高优先级事务避免被饿死

### 全景对照表

| 维度 | RU | RC | RR（默认） | SERIALIZABLE |
|---|---|---|---|---|
| 普通 SELECT | LOCK_NONE（读最新） | LOCK_NONE | LOCK_NONE | 事务内 LOCK_S；autocommit LOCK_NONE |
| read view | 不建（可见性跳过） | 每语句重建 | 事务内复用（快照固定） | 同 RR（但普通 SELECT 转当前读） |
| 可见性检查 | 跳过（脏读） | 检查 + 回表回溯 | 检查 + 回表回溯 | 当前读，不查 view |
| 加行锁（普通 SELECT） | 否 | 否 | 否 | 事务内加 LOCK_S |
| gap 锁（锁定读） | 不加 | 不加 | 加 next-key/gap | 加 next-key/gap |
| 半一致读 | 允许（实践主要在 RC） | 生效 | 不允许 | 不允许 |

---

### direction 与游标推进

### direction 是什么

`row_search_mvcc` 的参数，表示**本次调用**是"打开游标(0)"还是"基于已有游标续读(`ROW_SEL_NEXT`=1 / `ROW_SEL_PREV`=-1)"。

它是**调用级**的，不是"行级"的，也**不是每定位一行就置 0**：

- **打开游标**：`direction=0`，只在扫描的第一次（`index_first`/`index_read`）出现，`pcur->open` 定位。`index_read` 里 `row_search_mvcc(buf, mode, m_prebuilt, match_mode, 0)`（`ha_innodb.cc:10304`）
- **续读**：之后每次 `rnd_next`/`index_next` 都传 `ROW_SEL_NEXT`（`general_fetch`，`ha_innodb.cc:10619`/`10853`）

`rnd_next` 内部用 `m_start_of_scan` 标志区分：首次 `index_first`，之后 `general_fetch(ROW_SEL_NEXT)`（`ha_innodb.cc:10844-10854`）。

### direction 与锁定读/半一致性读无关

它只回答"是否打开游标"。半一致性读的"第一次读"和"第二次重读"，direction 取决于该行在扫描中的位置（扫描首条则第一次 0、重读 NEXT；扫描中段则两次都 NEXT）。

重读"同一行"靠的**不是 direction**，而是：半一致性读没走 `next_rec`（游标不推进）+ 第二次进来 `row_read_type==DID` 且 `need_to_process==false` 时不 `goto next_rec`。

### 游标推进

`next_rec` 标签（`row0sel.cc:5803`）→ PHASE 5 → `if (moves_up) pcur->move_to_next(&mtr); else pcur->move_to_prev(...)`（`row0sel.cc:5908-5936`）→ `goto rec_loop` 回到循环头处理下一条。

半一致性读路径（`row0sel.cc:5288-5314`）**不经过** `next_rec`，`break` 后落到 `normal_return` 返回，游标停在原记录（`store_position` 已存），故下次读仍在同一行——这是"半一致性读重读同行"的机制基础。

---

### need_to_process

`sel_restore_position_for_mysql`（`row0sel.cc:3406`）的返回值：调 `pcur->restore_position` 恢复之前 `store_position` 保存的位置，返回 true 表示"恢复后需重新处理游标现在指向的记录"（原记录被删、位置失效时，按 `moves_up` 调 `move_to_next` 前移）。

在 `row0sel.cc:4887-4906`：

- `need_to_process == true` 且 `row_read_type == DID`：把 DID 复位成 TRY（记录已删，之前的半一致性读作废）
- `need_to_process == false` 且 `row_read_type == DID`：不 `goto next_rec`，继续处理同一行（半一致性读重读加锁的分支）

---

### 行格式：rec_t 磁盘行与 record[0] 的转换

### rec_t 的物理布局

COMPACT/DYNAMIC 记录（new-style）的位域布局，`rem0rec.ic` 的权威注释：

```
(1) byte offset downward from origin:
    1-2: 8+8 bits relative offset of next record
    3:   3 bits status (000=conventional / 001=node pointer / 010=infimum / 011=supremum)
         5 bits heap number
    4:   8 bits heap number
    5:   4 bits n_owned + 4 bits info bits
```

固定头 5 字节（`REC_N_NEW_EXTRA_BYTES = 5`）。`rec` 的 origin 指向**第一个数据字段**，固定头在 origin 之下（低地址方向）。

COMPACT 记录从低地址到高地址的完整布局：

```
[变长字段长度列表 lens][NULL bitmap][(instant 版本位 1B)][5B 固定头][字段数据...]
                                                       ^ rec origin 指向这里
```

info bits 四位的含义（`rem0rec.ic`）：`MIN_REC`（min rec 标志）/ `DELETED`（delete mark，`rec_get_deleted_flag` 读它）/ `INSTANT`（instant ADD COLUMN）/ `VERSION`（row version）。

### 字段寻址：rec_get_offsets

`row_search_mvcc` 每取一条记录都先：

```c
offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED, ...);
```

得到 `offsets` 数组——每个元素是"该字段结束偏移 + 标志位"（`REC_OFFS_SQL_NULL / REC_OFFS_DEFAULT / REC_OFFS_DROP / REC_OFFS_EXTERNAL`）。COMPACT 字段寻址的主循环（`rec_init_offsets_comp_ordinary`）：

```c
do {
  if (nullable) {                     /* 读 NULL bitmap */
    if (*nulls & null_mask) { len = offs | REC_OFFS_SQL_NULL; goto resolved; }
  }
  if (!field->fixed_len) {            /* 变长：读 lens 列表 */
    len = *lens--;
    if (DATA_BIG_COL(col)) {          /* 可能外部存储 / 两字节长度 */
      if (len & 0x80) { len <<= 8; len |= *lens--; offs += len & 0x3fff; len = offs; goto resolved; }
    }
    len = offs += len;
  } else {
    len = offs += field->fixed_len;   /* 定长 */
  }
resolved:
  rec_offs_base(offsets)[i + 1] = len;
} while (++i < rec_offs_n_fields(offsets));
```

### ★ row_sel_store_mysql_rec：取行的最后一步

头注释（`row0sel.h`）：

> "Convert a row in the Innobase format to a row in the MySQL format. Note that the template in prebuilt may advise us to copy only a few columns to mysql_rec, other columns are left blank."

主循环**只遍历 `prebuilt->mysql_template`（本次查询需要的列），不遍历索引全部字段**：

```c
for (ulint i = 0; i < prebuilt->n_template; i++) {
  const auto templ = &prebuilt->mysql_template[i];
  ulint field_no = rec_clust ? templ->clust_rec_field_no : templ->rec_field_no;
  row_sel_store_mysql_field(mysql_rec, prebuilt, rec, rec_index, prebuilt_index,
                            offsets, field_no, templ, sec_field_no, lob_undo, blob_heap);
}
```

⇒ **`DB_ROW_ID` / `DB_TRX_ID` / `DB_ROLL_PTR` 三个隐藏系统列默认不拷给 server**——除非查询显式 `SELECT _rowid`，模板里才会有对应列。MVCC 取 `DB_TRX_ID` 走的是另一条路（直接从 rec 读 `trx_id_offset`），不靠模板。

`row_sel_store_mysql_field` 做四件事：

**(1) NULL**：置 server 层 NULL bitmap 位，并把字段数据区拷成 default 值：

```c
if (len == UNIV_SQL_NULL) {
  /* MySQL assumes that the field for an SQL NULL value is set to the default value. */
  mysql_rec[templ->mysql_null_byte_offset] |= (byte)templ->mysql_null_bit_mask;
  memcpy(mysql_rec + templ->mysql_col_offset,
         (const byte *)prebuilt->default_rec + templ->mysql_col_offset,
         templ->mysql_col_len);
  return true;
}
```

**`mysql_null_byte_offset == 0` 的列，其 NULL bit 就落在 `record[0]` 第 0 字节**——这就是 server 层"`record[0]` 第一个字节是 NULL bitmap（前 8 列）"约定的来源。InnoDB 不自己造 NULL bitmap，而是把位写进 `mysql_template[]` 事先算好的偏移/掩码。

**(2) 变长字段**：真 VARCHAR 写 1/2 字节长度前缀（`row_mysql_store_true_var_len`），CHAR 则 pad 尾部空格。

**(3) 字节序**：InnoDB 存大端，server 层要小端：

```c
case DATA_INT:
  /* Convert integer data from Innobase to a little-endian format, sign bit restored to normal */
  ptr = dest + len;
  for (;;) { ptr--; *ptr = *data; if (ptr == dest) break; data++; }
  if (!templ->is_unsigned) dest[len - 1] = (byte)(dest[len - 1] ^ 128);
```

**(4) BLOB 外存**：外存字段在 rec 里只占 20 字节 `BTR_EXTERN_FIELD_REF_SIZE` 指针，`lob::btr_rec_copy_externally_stored_field` 才把 BLOB 页内容拷进 heap。**本地存放的小 BLOB 也要拷**，因为 `row_sel_field_store_in_mysql_format` 对 BLOB 只存指针，而 mtr commit 后 page latch 释放、指向页内的指针会失效：

```c
if (DATA_LARGE_MTYPE(templ->type) || DATA_GEOMETRY_MTYPE(templ->type)) {
  /* It is a BLOB field locally stored in the InnoDB record: we MUST copy its
     contents to prebuilt->blob_heap here because ... stores a pointer to the
     data, and the data ... will be invalid as soon as the mini-transaction is
     committed and the page latch ... is released. */
  data = static_cast<byte *>(mem_heap_dup(heap, data, len));
}
```

### ★ clust_templ_for_sec：扫描二级索引、但模板基于主键

`row_search_mvcc`（`row0sel.cc:4454`）与 `row_sel_store_mysql_rec`（`:2735`）里各有一个同名局部标志，语义一致：

```cpp
/* True if we are scanning a secondary index, but the template is based
on the primary index. */
bool clust_templ_for_sec;
```

**赋值**：

- `row_search_mvcc`（`:4839`）：`clust_templ_for_sec = index != clust_index && prebuilt->need_to_access_clustered;`
- `row_sel_store_mysql_rec`（`:2735`）：`clust_templ_for_sec = (sec_field_no != ULINT_UNDEFINED);`——只要该列在二级索引记录里有对应字段号就算

**取列时的字段号切换**（`:2750`，在 `row_sel_field_store_in_mysql_format` 路径上）：

```cpp
if (clust_templ_for_sec) {
  clust_field_no = field_no;      // 存下模板（基于 PK）的字段号
  field_no = sec_field_no;        // 暂时换成二级索引记录里的字段号
}
/* ...用 sec_field_no 从二级索引记录取数据... */
if (clust_templ_for_sec) {
  field_no = clust_field_no;      // 还原（:2876）
}
```

★★ **为什么能这么做**——因为 **InnoDB 的二级索引记录里隐含主键列**。于是"模板基于 PK 的那些列"中，有一部分是**可以直接从二级索引记录里读出来的**，根本不用回表。这正是"二级索引记录包含 PK"这一物理事实带来的红利，`clust_templ_for_sec` 就是把这件事显式化的标志。

配套处理还有几处，都体现"两套坐标共存"的复杂性：

| 位置 | 处理 |
|---|---|
| `:2637-2644` | end-range 检查的**长度断言放松**：索引可能只存列前缀，`len` 可能小于 `mysql_col_len`，故断言里加 `\|\| clust_templ_for_sec` |
| `:2745` | `rec_offs_validate` 用 `prebuilt_index`（模板所属索引）而非 `rec_index`（记录所属索引） |
| `:2916-2925` | 需要"存储二级索引记录中的所有聚簇列"时，遍历 `prebuilt_index` 的字段并映射回 `rec_index` 的字段号 |
| `:5021` | end_range_cache 构造时 `key_index = clust_index`（模板基于 PK） |

**为什么值得单独理解它**：这是"覆盖索引 / 回表"二分之外的**第三种形态**——不是纯覆盖（全在二级索引里），也不是普通回表（全去聚簇取），而是"**模板按 PK 建、但尽量从二级索引记录里直接抠出 PK 列，剩下的才回表**"。它就是 `row_sel_store_mysql_rec` 里最难的那段坐标换算。

---

### MVCC 可见性：快照读与二级索引回表

### 快照读（聚簇索引）

```c
if (index == clust_index) {
  if (!lock_clust_rec_cons_read_sees(rec, index, offsets, trx_get_read_view(trx))) {
    row_sel_build_prev_vers_for_mysql(...);   /* 不可见 → 沿 undo 回溯 */
    if (old_vers == nullptr) goto next_rec;    /* 该行在 read view 创建前还不存在 */
    rec = old_vers;
  }
}
```

`lock_clust_rec_cons_read_sees` 从聚簇记录读 `DB_TRX_ID`，交给 read view 的 `changes_visible` 判定。

### ★ 二级索引不能直接信，必须回表确认

`lock_sec_rec_cons_read_sees` 的注释是核心证据：

> "NOTE that a non-clustered index page contains so little information on its modifications that also in the case false, the present version of rec may be the right, but we must check this from the clustered index record."

原因：**二级索引记录本身不携带 `DB_TRX_ID` / `DB_ROLL_PTR`**（它们只存在于聚簇记录），二级索引只能拿到"页级"的 `PAGE_MAX_TRX_ID` 粗粒度提示：

```c
trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));
return (view->sees(max_trx_id));   /* sees(id) == id < m_up_limit_id */
```

- 返回 true（`max_trx_id < m_up_limit_id`）→ 肯定可见，可信
- 返回 false → **不确定**，`goto requires_clust_rec` 回表，用聚簇记录的 `DB_TRX_ID` 做精确判断，必要时再沿聚簇 undo 回溯

### 回表后的旧版本校验

回表拿到旧版本后，还要 `row_sel_sec_rec_is_for_clust_rec` 验证"这个旧版本聚簇记录是否真的对应原二级索引记录 rec"——**防止通过二级索引访问到快照里本不该存在的行**（比如二级记录是 delete-mark、或回表结果来自更早版本）。

### undo 回溯：row_vers_build_for_consistent_read（外层循环）

```c
version = rec;
for (;;) {
  bool purge_sees = trx_undo_prev_version_build(rec, mtr, version, index, *offsets,
                                                heap, &prev_version, nullptr, vrow, 0, lob_undo);
  if (prev_version == nullptr) { *old_vers = nullptr; break; }   // 回溯到 INSERT undo：快照前不存在
  *offsets = rec_get_offsets(prev_version, index, ...);
  trx_id = row_get_rec_trx_id(prev_version, index, *offsets);
  if (view->changes_visible(trx_id, ...)) {   // ★ 每步可见性判断在外层做
    *old_vers = rec_copy(buf, prev_version, *offsets);   // 可见 → 拷贝到 in_heap 返回
    break;
  }
  version = prev_version;                      // 不可见 → 继续回溯
}
```

- ★ **每步可见性判断在外层循环做**，内层 `trx_undo_prev_version_build` 只负责"单步还原"
- `prev_version == nullptr` → 回溯到 INSERT undo（第一次插入），记录在快照前不存在
- `purge_sees == false` → undo 日志已被 purge 清除，返回 `DB_MISSING_HISTORY`

### 单步回溯：trx_undo_prev_version_build

```c
roll_ptr = row_get_rec_roll_ptr(rec, index, offsets);
if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
  return true;   // INSERT undo：*old_vers 保持 nullptr，调用方判定"快照前不存在"
}
...
trx_undo_get_undo_rec(roll_ptr, rec_trx_id, heap, is_temp, ..., &undo_rec);   // 读 undo 记录（可能已被 purge）
ptr = trx_undo_rec_get_pars(undo_rec, &type, &cmpl_info, ..., &undo_no, &table_id, ...);
ptr = trx_undo_update_rec_get_sys_cols(ptr, &trx_id, &roll_ptr, &info_bits);  // 旧版本的 trx_id/roll_ptr/info_bits
ptr = trx_undo_update_rec_get_update(ptr, index, type, ..., &update, ...);    // 解析 update vector
```

按 undo 类型决定动作：

- **INSERT undo**：`trx_undo_roll_ptr_is_insert` 为真 → 这是该记录第一次插入，返回 `nullptr`
- **DELETE undo**（delete-mark）：undo 记录"本次做了 delete-mark"，update vector 携带**旧 info_bits**，应用后"补回被删除前的状态"（去掉 delete mark）
- **UPDATE undo**：update vector 是被改字段的**旧值**，应用后覆盖当前记录得更新前版本

旧版本的两种构造方式：

```c
if (row_upd_changes_field_size_or_external(index, offsets, update)) {
  // 字段长度或外部存储(BLOB)变化：必须整体重建记录
  entry = row_rec_to_index_entry(rec, index, offsets, heap);
  row_upd_index_replace_new_col_vals(entry, index, update, heap);
  *old_vers = rec_convert_dtuple_to_rec(buf, index, entry);
} else {
  // 字段定长：拷贝当前记录后原地 patch（快路径）
  *old_vers = rec_copy(buf, rec, offsets);
  row_upd_rec_in_place(*old_vers, index, offsets, update, nullptr);
}
```

- 定长字段走**原地 patch**（`rec_copy` + `row_upd_rec_in_place`），只覆盖被 update vector 命中的字段，效率高
- 变长或 BLOB 外存：记录物理布局会变，必须拆成 dtuple → 用旧值替换 → 重新编码成 rec

### 二级旧版本校验：row_sel_sec_rec_is_for_clust_rec（算法）

回表拿到旧版本后，逐字段比"二级记录的键字段"与"聚簇旧版本对应列"：

```c
if (rec_get_deleted_flag(clust_rec, ...)) { is_equal = false; return DB_SUCCESS; }  // 聚簇已 delete-mark
n = dict_index_get_n_ordering_defined_by_user(sec_index);   // 只比用户定义的排序列
for (i = 0; i < n; i++) {
  // 虚拟列：从聚簇行重新计算；普通列：dict_col_get_clust_pos 定位 + rec_get_nth_field_instant 取
  ...
  if (0 != cmp_data_data(col->mtype, col->prtype, true, clust_field, len, sec_field, sec_len)) {
    is_equal = false;   // 不匹配
    goto func_exit;
  }
}
```

**为什么需要**：二级索引不版本化但聚簇版本化。典型失配场景——事务 A 把列 c 从 5 改成 6（二级记录 c=6 插入、c=5 delete-mark），另一个事务的 read view 在 A 提交前创建，回表构造出 c=5 的旧聚簇版本，它显然不对应二级记录 c=6。

- 只遍历**用户定义的排序列**（`n_ordering_defined_by_user`），避免把二级索引内部追加的主键列重复比较
- 前缀索引、外部 BLOB（`row_sel_sec_rec_is_for_blob`）、空间 MBR、多值列各有专门比较分支
- `rec_equal == false` → 这条二级记录在 read view 中不可见或失配，跳过

---

### B-tree 定位：btr_cur_search_to_nth_level

> 本节从「取行」视角概述搜索；B-tree 搜索算法 / 分裂 / 合并 / AHI 的完整机制（latch_mode 全表、8.0 SMO 锁预测裁剪、意图升级重搜等）详见 [`btr.md`](../index/btr.md)。

`pcur->open_no_init` / `open_at_side` 内部真正干活的函数——从根下到叶子定位一条记录。这是"取行"的地基。

### 搜索模式换算：叶子语义 → 非叶语义（经典坑）

node_ptr 的 key 是其子页的**最小记录前缀**，每个 `(key, child_page_no)` 指向 `[key, 下一个 node_ptr 的 key)` 区间：

```cpp
switch (mode) {
  case PAGE_CUR_GE: page_mode = PAGE_CUR_L; break;   // leaf 找 >= X，非叶找 < X 的最大 node_ptr
  case PAGE_CUR_G:  page_mode = PAGE_CUR_LE; break;  // leaf 找 > X，非叶找 <= X 的最大 node_ptr
  default: page_mode = mode; break;
}
```

原因：若非叶也用 `PAGE_CUR_GE`，搜索 15 会命中 node_ptr=20，而 15 在 `[10,20)` 区间，会漏掉目标；用 `PAGE_CUR_L`（严格小于）命中 node_ptr=10，正确定位到 `[10-19]`。

### 主循环：逐层下降（latch coupling）

```cpp
search_loop:
  // 非叶层用 upper_rw_latch（继承 index lock 对应的页锁），叶层用 latch_mode
  ...
  block = buf_page_get_gen(page_id, page_size, rw_latch, root_guess, fetch, ..., mtr);

  if (height == ULINT_UNDEFINED) {
    height = btr_page_get_level(page);    // 第一次到 root，读树高
    index->search_info->root_guess = block;  // 缓存 root
  }

  if (height == 0) {
    // 到叶子：latch coupling 释放沿途所有上层 block
    for (; n_releases < n_blocks; n_releases++)
      mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases], tree_blocks[n_releases]);
    page_mode = mode;                     // 恢复原始搜索模式
  }

  // 页内二分定位（page_cur_search_with_match）
  ...
  if (level != height) {
    // 非叶：读 node_ptr 的 child page no，下降到下一层
    page_id.reset(space, btr_node_ptr_get_child_page_no(node_ptr, offsets));
    n_blocks++;
    goto search_loop;
  }
```

- **latch coupling**：`tree_blocks[]`/`tree_savepoints[]` 追踪沿途页面。每层拿到子页号后，判断"是否会触发 SMO（分裂/合并）"——不会则释放父页锁（只保留 root），会则保留父页锁直到目标层（因为 SMO 会修改父页）
- **`root_guess`**：缓存的 root block 指针，传给 `buf_page_get_gen` 作 hint

### 页内二分：page_cur_search_with_match（两级二分）

```cpp
// 1) 先对 page directory（slot 数组）二分，直到上下 slot 距离为 1
low = 0;
up = page_dir_get_n_slots(page) - 1;
while (up - low > 1) {
  mid = (low + up) / 2;
  mid_rec = page_dir_slot_get_rec(page_dir_get_nth_slot(page, mid));
  cur_matched_fields = min(low_matched_fields, up_matched_fields);
  cmp = tuple->compare(mid_rec, index, offsets, &cur_matched_fields);
  if (cmp > 0) { low = mid; low_matched_fields = cur_matched_fields; }
  else if (cmp) { up = mid; up_matched_fields = cur_matched_fields; }
  ...
}

// 2) 再对 slot 内 owned records 线性查找，直到 low_rec 与 up_rec 相邻
// 3) 定位：mode <= PAGE_CUR_GE 则 position 到 up_rec，否则 low_rec
```

- **`up_match`/`low_match` 跨层传递公共前缀**：`cur_matched_fields = min(low, up)` 作为下一轮比较的起始匹配数——父子层之间的公共前缀无需重比，这是 B-tree 搜索的经典优化
- `low_match` = 搜索键与左记录匹配的字段数，`up_match` = 与右记录匹配的字段数

### AHI 快速路径（在下降前）

```cpp
if (rw_lock_get_writer(btr_get_search_latch(index)) == RW_LOCK_NOT_LOCKED &&
    latch_mode <= BTR_MODIFY_LEAF && index->search_info->last_hash_succ &&
    !index->disable_ahi && !estimate && !dict_index_is_spatial(index) &&
    UNIV_LIKELY(btr_search_enabled) && !modify_external &&
    btr_search_guess_on_hash(tuple, mode, latch_mode, cursor, has_search_latch, mtr)) {
  btr_cur_n_sea++;
  return;
}
```

`btr_search_guess_on_hash`：`dtuple_hash` 算哈希 → AHI 哈希表取 rec 指针 → `buf_block_from_ahi` 反推 block → `buf_page_get_known_nowait` **nowait 方式上 latch（失败即回退，绝不在 AHI 路径阻塞）** → `btr_search_check_guess` 校验 index id/space/key 匹配。AHI 是"猜测"，每次先 `last_hash_succ = false`，校验通过才置 true。

### 返回后游标状态

`cursor->flag = BTR_CUR_BINARY`（非 AHI 路径）；`cursor->low_match/up_match` 写回（对 `PAGE_CUR_LE` 有意义，用于判断唯一性/回表命中）；叶页的 latch 由 mtr 持有。

---

### 加锁：sel_set_rec_lock 与隐式锁转显式

### sel_set_rec_lock 的分派

```cpp
if (index->is_clustered()) {
  err = lock_clust_rec_read_check_and_lock(...);   // 聚簇
} else if (dict_index_is_spatial(index)) {
  err = sel_set_rtr_rec_lock(...);                 // 空间索引：predicate lock
} else {
  err = lock_sec_rec_read_check_and_lock(...);     // 二级
}
```

`type` 参数：`LOCK_ORDINARY`（next-key）/ `LOCK_GAP` / `LOCK_REC_NOT_GAP`；`mode`：`LOCK_S`/`LOCK_X`；`sel_mode`：`SELECT_ORDINARY`/`SKIP_LOCKED`/`NOWAIT`。

### 隐式锁 → 显式锁：lock_rec_convert_impl_to_expl

**为什么要转换**：InnoDB 里事务修改聚簇记录时，默认**不创建显式记录锁**，而是把自己的 `trx_id` 写进记录的 `DB_TRX_ID` 字段、undo 地址写进 `DB_ROLL_PTR`——这就是**隐式锁**。任何后来者都能从记录上的 `trx_id` 推断"这条记录最新版本属于哪个事务"。

但隐式锁不是一个 `lock_t` 对象，**不在哈希表的锁队列里**，无法作为 wait-for 图节点，也无法被 `lock_rec_has_to_wait` 检测冲突。所以另一个事务要加锁时，必须先把隐式锁"物化"成显式 `LOCK_X | LOCK_REC_NOT_GAP` 的 `lock_t`：

```cpp
void lock_rec_convert_impl_to_expl(const buf_block_t *block, const rec_t *rec, ...) {
  if (index->is_clustered()) {
    trx_id_t trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);
    trx = trx_rw_is_active(trx_id, true);           // 读 DB_TRX_ID，找活跃事务
  } else {
    trx = lock_sec_rec_some_has_impl(rec, index, offsets);   // 二级：回表反推作者事务
  }
  if (trx != nullptr) {
    lock_rec_convert_impl_to_expl_for_trx(block, rec, index, offsets, trx, heap_no);
  }
}
```

### lock_rec_lock：快路径 / 慢路径

```cpp
static dberr_t lock_rec_lock(bool impl, select_mode sel_mode, ulint mode, ...) {
  switch (lock_rec_lock_fast(impl, mode, block, heap_no, index, thr)) {
    case LOCK_REC_SUCCESS:         return DB_SUCCESS;
    case LOCK_REC_SUCCESS_CREATED: return DB_SUCCESS_LOCKED_REC;
    case LOCK_REC_FAIL:
      return lock_rec_lock_slow(impl, sel_mode, mode, block, heap_no, index, thr);
  }
}
```

慢路径核心：

```cpp
if (held_lock != nullptr) return DB_SUCCESS;   // 已有足够强的锁

const auto conflicting = lock_rec_other_has_conflicting(mode, block, heap_no, trx);
if (conflicting.wait_for != nullptr) {
  switch (sel_mode) {
    case SELECT_SKIP_LOCKED: return DB_SKIP_LOCKED;
    case SELECT_NOWAIT:      return DB_LOCK_NOWAIT;
    case SELECT_ORDINARY: {
      RecLock rec_lock(thr, index, block, heap_no, mode);
      return rec_lock.add_to_waitq(conflicting.wait_for);   // 挂入等待队列，触发死锁检测
    }
  }
}
lock_rec_add_to_queue(LOCK_REC | mode, block, heap_no, index, trx);
return DB_SUCCESS_LOCKED_REC;
```

- **`lock_t` 结构**：`type_mode` 位图（`LOCK_REC=32`/`LOCK_WAIT=256`/`LOCK_GAP=512`/`LOCK_REC_NOT_GAP=1024`/`LOCK_INSERT_INTENTION=2048`，低 4 位是 S/X 模式）；`lock_rec_t` 含 `page_id` + **位图**（每 heap_no 一位）——同一事务同页同类型多锁合并成一个 `lock_t`
- **冲突检测 `lock_rec_has_to_wait`** 有几条经典规则：纯 gap 锁之间不冲突；记录锁不等待 gap 锁；gap 锁不等待 record-not-gap 锁；**没人需要等待 insert intention 锁**
- `impl=true`（如 UPDATE 的 `lock_clust_rec_modify_check_and_lock`）无冲突时**保持隐式锁**，不创建显式锁；`sel_set_rec_lock` 走 `impl=false`，总是创建显式记录锁

### 二级索引隐式锁的反推：row_vers_impl_x_locked

二级记录**没有 `DB_TRX_ID`**，所以 `lock_sec_rec_some_has_impl` 不能直接读 trx_id，而是先看页头的 `PAGE_MAX_TRX_ID` 粗筛，落在活跃区间才回表：

```cpp
max_trx_id = page_get_max_trx_id(page);
if (!recv_recovery_is_on() && !can_older_trx_be_still_active(max_trx_id)) {
  trx = nullptr;                                       // 快判断：绝无隐式锁
} else {
  trx = row_vers_impl_x_locked(rec, index, offsets);   // 回表聚簇推断
}
```

`row_vers_impl_x_locked_low` 的核心思想（源码注释有形式化推导）：二级索引不版本化，但"聚簇先改、二级后同步"，因此一条二级记录 S 要么与聚簇当前版本同步、要么与上一版本同步。算法沿聚簇版本链回溯，找 S 的"作者事务"（`row_vers_find_matching` 比较各版本字段值/delete-mark 状态）。

---

### 持久游标：store_position / restore_position

> 本节从「取行」视角讲恢复协议；`btr_pcur_t` 结构全字段、乐观/悲观恢复算法与跨页推进的锁序细节详见 [`btr.md`](../index/btr.md)。

`btr_pcur_t` 的"持久"含义：**释放 page latch、甚至页面被写出 buffer pool / 发生页分裂合并之后，游标依然能靠"记录的排序 key 前缀 + 相对位置"重新定位。**

保存字段 → 恢复用途：

| 保存 | 恢复 |
|---|---|
| `m_block_when_stored` | 乐观恢复：直接用 block 指针，不走 B-tree 搜索 |
| `m_modify_clock` | 乐观判断：clock 没变 ⇒ 页面未改，直接 offset 定位 |
| `m_old_rec`（key 副本） | 悲观恢复：clock 变了（页面被改/淘汰），用 key 从 root 重新搜索 |
| `m_rel_pos` | 恢复后微调：BEFORE/AFTER 需 move_to_next/prev |

这就是 `need_to_process` 的来由——`restore_position` 之后，原记录可能已被删除，需要决定是否重新处理当前记录。

### rel_pos 状态机

```cpp
auto success = pcur->restore_position(latch_mode, mtr, ...);
*same_user_rec = success;

switch (pcur->m_rel_pos) {
  case BTR_PCUR_ON:    // 游标原来正停在一条 user record 上
    if (!success && moves_up) { pcur->move_to_next(mtr); return true; }
    return !success;
  case BTR_PCUR_BEFORE:   // 停在记录前（PAGE_CUR_G/L 定位结果）
    if (moves_up && is_on_user_rec()) { pcur->move_to_next(mtr); return true; }
    return true;
  case BTR_PCUR_AFTER:    // 停在记录后（PAGE_CUR_LE 定位结果）
    if (is_on_user_rec() && !moves_up) pcur->move_to_prev(mtr);
    return true;
  case BTR_PCUR_BEFORE_FIRST_IN_TREE:
  case BTR_PCUR_AFTER_LAST_IN_TREE:
    return true;   // 树首/树尾：由调用方处理边界
}
```

`rel_pos` 各状态含义：

| rel_pos | 含义 |
|---|---|
| `BTR_PCUR_ON` | 游标正停在一条 user record 上 |
| `BTR_PCUR_BEFORE` | 停在该记录**前**（`PAGE_CUR_G`/`L` 定位结果） |
| `BTR_PCUR_AFTER` | 停在该记录**后**（`PAGE_CUR_LE` 定位结果） |
| `BTR_PCUR_BEFORE_FIRST_IN_TREE` | 全树第一条之前 |
| `BTR_PCUR_AFTER_LAST_IN_TREE` | 全树最后一条之后 |

`m_pos_state` 区分恢复方式：`IS_POSITIONED`（精确重新定位）vs `IS_POSITIONED_OPTIMISTIC`（乐观恢复到了同一物理记录，但 rel_pos 是 BEFORE/AFTER，位置还需按方向校正）。

### 乐观 vs 悲观恢复

```cpp
// 乐观恢复：仅当锁模式允许且非 intrinsic 表
if ((latch_mode == BTR_SEARCH_LEAF || ...) && !is_intrinsic()) {
  if (m_block_when_stored.run_with_hint([&](buf_block_t *hint) {
        return hint != nullptr &&
               btr_cur_optimistic_latch_leaves(hint, m_modify_clock, &latch_mode, &m_btr_cur, ...);
      })) {
    if (m_rel_pos == BTR_PCUR_ON) return true;   // 同一物理记录，直接成功
    ...
  }
}

// 悲观恢复：用保存的 key 从 root 重新 search
tuple = dict_index_build_data_tuple(index, m_old_rec, m_old_n_fields, heap);
switch (m_rel_pos) {
  case BTR_PCUR_ON:    mode = PAGE_CUR_LE; break;
  case BTR_PCUR_AFTER: mode = PAGE_CUR_G;  break;
  case BTR_PCUR_BEFORE: mode = PAGE_CUR_L; break;
}
open_no_init(index, tuple, mode, latch_mode, 0, mtr, ...);   // 内部即 btr_cur_search_to_nth_level
```

- **乐观恢复**：`buf_page_optimistic_get` 直接对保存的 block 上 latch，检查 `m_modify_clock` 是否与保存值一致——**clock 没变 ⇒ 页面未被修改 ⇒ 记录位置仍有效**，无需从 root 重走
- **悲观恢复**：clock 变了（页面被改/淘汰）→ 用 `m_old_rec` 重建搜索 tuple，按 rel_pos 映射搜索模式（`ON→LE` / `AFTER→G` / `BEFORE→L`），`open_no_init` 从 root 完整执行 `btr_cur_search_to_nth_level`
- **`same_user_rec`（out）** = 恢复在"排序前缀完全相同"的 user record 上（只有乐观 ON 或悲观 ON 且 `cmp_dtuple_rec==0` 才 true）；**`need_to_process`（返回值）** = 恢复/校正后游标是否停在需要立即处理的位置

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/handler/ha_innodb.cc:10833` | `ha_innobase::rnd_next` |
| `storage/innobase/handler/ha_innodb.cc:10530` | `general_fetch`（传 direction） |
| `storage/innobase/handler/ha_innodb.cc:10304` | `index_read` 里 direction=0 |
| `storage/innobase/handler/ha_innodb.cc:10844` | `m_start_of_scan` 区分首次/续读 |
| `storage/innobase/row/row0sel.cc:3406` | `sel_restore_position_for_mysql`（need_to_process） |
| `storage/innobase/row/row0sel.cc:4887` | need_to_process 与 DID 的分支 |
| `storage/innobase/row/row0sel.cc:5288` | 半一致性读路径（不经 next_rec） |
| `storage/innobase/row/row0sel.cc:5803` | `next_rec` 标签（游标推进入口） |
| `storage/innobase/row/row0sel.cc:5908` | `pcur->move_to_next` / `move_to_prev` |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Locking and Transaction Model*（半一致性读）
- *MySQL 8.0 Reference Manual → Consistent Nonlocking Reads*

**相关文档**
- 可见性判断见 [`mvcc.md`](mvcc.md)
- handler 接口见 [`../server/handler.md`](../server/handler.md)

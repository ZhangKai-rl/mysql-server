# 索引 × 事务：MVCC、purge 与唯一性的联动

> **一句话**：二级索引记录**没有版本信息**（无 `DB_TRX_ID` / `DB_ROLL_PTR`）这一个事实，派生出了整片交叉地带的全部机制——可见性只能"页级粗筛 + 回表"、删除只能"delete-mark 两阶段 + 版本链回溯"、purge 只能"按主键重新定位 + 受水位约束"、唯一性只能"加锁而非靠 MVCC"。**索引承担"位置"，事务承担"时间"，两者在每一条记录上交接。**

> **边界**：本篇是**索引视角**的整合——只讲"索引 × 事务"的**交叉地带**，两侧的纵深各有主场。
>
> | 要看什么 | 去哪篇 | 本篇**不**讲 |
> |---|---|---|
> | ReadView 四字段、`changes_visible` 可见性规则、RR/RC 下 view 创建时机 | [`../innodb/mvcc.md`](../innodb/mvcc.md) | 只讲 `sees()` 一行如何被页级粗筛使用 |
> | undo 记录格式、purge 线程模型 / rseg 轮转 / truncate / 限流 | [`../innodb/undo_log.md`](../innodb/undo_log.md) | 只讲 purge **判定接口**（`row_purge_poss_sec`）与"先二级后聚簇"的顺序约束 |
> | `trx_t` 生命周期、提交/回滚协议、内部 XA | [`../innodb/trx.md`](../innodb/trx.md) | 不讲 |
> | `lock_t` 体系、gap / next-key / 隐式锁转换 | [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md) | 只讲唯一性判重**为什么必须加锁**、加哪种 |
> | 唯一性检查的完整链路（含外键） | [`constraint.md`](constraint.md) | 只讲二级索引侧的加锁顺序 |
> | 二级索引的生命周期闭环 | [`secondary.md`](secondary.md) | 只讲 delete-mark 两阶段 |
>
> ⚠️ **与兄弟篇的重叠区（读时请注意分工）**：`row_vers_build_for_consistent_read` / `trx_undo_prev_version_build` / `row_vers_old_has_index_entry` / `row_purge_poss_sec` 这几个符号在 `mvcc.md`、`undo_log.md`、本篇**都会出现**（实测三篇共同覆盖的符号有 7 个）。分工是：
> - `mvcc.md`：**读到哪个版本**——判定规则与回溯算法本身；
> - `undo_log.md`：**版本怎么存、怎么被消费**——undo 格式与 purge 的执行过程；
> - **本篇**：这些机制**在二级索引上退化成了什么**——不版本化 ⇒ 必须回表、必须两阶段删、唯一性只能靠锁。
>
> 本篇贴的这些函数源码，是为了钉住"**索引侧的调用点与参数**"（如 `row_vers_old_has_index_entry` 的 `also_curr` 在 purge 侧传 true、回滚侧传 false），不是要替代另外两篇的剖析。

## 目录

- [概述](#概述)
  - 是什么
  - 用途
  - 版本演进
- [理论基础](#理论基础)
  - 为什么二级索引不版本化：用读的代价换写的代价
  - 为什么删除必须两阶段：索引项是"被共享的引用"
  - 为什么可见性判定是"页级粗筛 + 回表"两级
  - 为什么唯一性只能靠锁：MVCC 管不了"未来"
  - 为什么 purge 与读共享同一个水位
- [核心实现](#核心实现)
  - 二级索引没有 trx_id/roll_ptr 的三个后果
  - delete-mark 两阶段与 `row_vers_old_has_index_entry`
  - 读路径：页级粗筛 + 回表
  - 回表三步：定位 → 构造 old_vers → 校验"这条二级项真属于这个版本"
  - delete-marked 记录对查询的影响
  - purge：undo → 聚簇 ref → 索引定位 → 物理删除
  - 唯一性检查为什么必须加锁
  - 长事务与索引记录膨胀
- [可观测性](#可观测性)
- [Misc](#misc)
  - ★ 本机制里的工程实现技法
  - ★ 一次 DELETE 的完整账（把三阶段串起来）
  - 失效与退化场景
  - 易混淆点
  - 三条交叉线的总图
- [参考](#参考)

---

## 概述

### 是什么

索引是 MVCC 与 purge 的**舞台**：一致性读靠索引找到记录再判版本；删除先给索引项打 delete-mark、purge 阶段才物理删除；唯一性检查借索引完成但靠锁保证。

### 用途

- 理解"为什么删除了数据索引页不缩小"（delete-mark 未 purge）；
- 理解 purge 与二级索引的精确交互（何时能物理删、会不会触发页合并）；
- 理解唯一索引插入时的锁行为（为什么 RC/RR 都必须加锁）；
- 理解长事务如何导致二级索引膨胀。

### 版本演进

| 版本 | 变化 | 为什么 |
|------|------|--------|
| InnoDB 早期至今 | 二级索引 delete-mark 两阶段机制**未变** | 这是"索引项不版本化"的必然推论，改不动 |
| 8.0 | `row_vers_old_has_index_entry` 增加**虚拟列**分支（`dict_index_has_virtual`，不是"多值索引分支"） | 虚拟列的值不在聚簇记录里，回溯每个版本时要用 `row_vers_vc_matches_cluster` / `row_vers_build_clust_v_col` 现算 |
| 8.0.x | `PAGE_MAX_TRX_ID` 的粗筛、`innodb_purge_threads` 并发化 | 持续优化回表与 purge 吞吐 |
| 8.4 / 9.x | 本库未跟踪到本篇机制的结构性变化 | —— |

---

## 理论基础

### 为什么二级索引不版本化：用读的代价换写的代价

二级索引记录 = 索引列 + 隐式主键，**没有** `DB_TRX_ID` / `DB_ROLL_PTR`。这不是遗漏，而是被三个约束夹逼的选择：

| # | 约束 | 若二级索引也存版本信息会怎样 |
|---|---|---|
| 1 | **写放大** | 任何 UPDATE（**即使不碰索引列**）都要修改该行所有二级索引项的版本字段——一次行更新变成 N 次索引更新，且每次都是随机写 |
| 2 | **索引膨胀** | 二级索引自己也会产生版本链（每个历史版本一个索引项），B+ 树迅速膨胀，扫描性能崩塌 |
| 3 | **版本权威唯一** | 版本链只能有一个权威源头。若聚簇与二级索引各自维护版本，两处可能不一致（谁先 purge？以谁为准？） |

★ **选择的本质**：InnoDB 让二级索引项**不承载时间信息**，代价是判断可见性**必须回聚簇索引**走版本链（读路径变贵）；收益是写路径简洁、索引不膨胀。**这是"用读的代价换写的代价"**——因为 OLTP 里写放大和索引膨胀的代价远高于偶尔多一次回表。

★ 与这个选择**互为因果**的是：二级索引项可以被**多个历史版本共享**（同一个索引项对应聚簇记录的多个版本）——这正是删除必须两阶段的根源。

### 为什么删除必须两阶段：索引项是"被共享的引用"

一个二级索引项可能同时被**多个聚簇历史版本**引用。所以不能"看到一次删除就物理删掉索引项"——必须确认**没有任何存活版本还需要这个 entry**。

判定方式就是 `row_vers_old_has_index_entry()`：**沿聚簇版本链回溯**，逐版本检查"这个版本是否仍需这条索引项"。

★ **为什么不用引用计数？** 引用计数看似 O(1)，但版本会被 purge 动态清理，计数器的增减必须与版本生命周期严格同步——一旦失配（漏减）索引项就永远删不掉（泄漏）；而版本链回溯虽然 O(版本数)，但**只在 purge 时执行**（低频路径），且正确性天然由版本链本身保证（不需要额外状态）。**用低频的计算换掉一份必须永远正确的状态**，是这个设计的取舍。

### 为什么可见性判定是"页级粗筛 + 回表"两级

直接每条记录都回表太贵（一次聚簇索引搜索）。所以 InnoDB 加了一层**页级粗筛**：

- 每个二级索引页共享一个 `PAGE_MAX_TRX_ID`（**只增不减**的上界，记录"本页被修改过的最大 trx_id"）；
- 若 `view->sees(max_trx_id)`——连上界都比 ReadView 老，则**页内所有记录必然都可见**，无需回表；
- 否则无法确定 → `requires_clust_rec` 回表用聚簇版本链精确判定。

★ **为什么上界可用**：`PAGE_MAX_TRX_ID` 是单调的，任何修改本页的事务都会把它的 trx_id 刷上去。所以"上界可见 ⇒ 全页可见"在逻辑上成立（保守方向正确）。

#### 粗筛用的 `sees()` 只有一行：拿上界比 `m_up_limit_id`

`storage/innobase/include/read0types.h:167-196`：

```cpp
  /** Check whether the changes by id are visible.
  @param[in]    id      transaction id to check against the view
  @param[in]    name    table name
  @return whether the view sees the modifications of id. */
  [[nodiscard]] bool changes_visible(trx_id_t id,
                                     const table_name_t &name) const {
    ut_ad(id > 0);

    if (id < m_up_limit_id || id == m_creator_trx_id) {
      return (true);
    }

    check_trx_id_sanity(id, name);

    if (id >= m_low_limit_id) {
      return (false);

    } else if (m_ids.empty()) {
      return (true);
    }

    const ids_t::value_type *p = m_ids.data();

    return (!std::binary_search(p, p + m_ids.size(), id));
  }

  /**
  @param id             transaction to check
  @return true if view sees transaction id */
  bool sees(trx_id_t id) const { return (id < m_up_limit_id); }
```

/* 【注】`sees()` 就是一行 `id < m_up_limit_id`：只判"一定可见"，不判"一定不可见"。
   因此二级索引粗筛是**单向**的——`sees()` 为真 ⇒ 整页必然可见；为假 ⇒ 只是"不确定"，必须回表。 */

两个水位在 `ReadView::prepare()` 里定（`storage/innobase/read/read0read.cc:449-473`）：

```cpp
void ReadView::prepare(trx_id_t id) {
  ut_ad(trx_sys_mutex_own());

  m_creator_trx_id = id;

  m_low_limit_no = trx_get_serialisation_min_trx_no();

  m_low_limit_id = trx_sys_get_next_trx_id_or_no();

  ut_a(m_low_limit_no <= m_low_limit_id);

  if (!trx_sys->rw_trx_ids.empty()) {
    copy_trx_ids(trx_sys->rw_trx_ids);
  } else {
    m_ids.clear();
  }

  /* The first active transaction has the smallest id. */
  m_up_limit_id = !m_ids.empty() ? m_ids.front() : m_low_limit_id;

  ut_a(m_up_limit_id <= m_low_limit_id);

  ut_d(m_view_low_limit_no = m_low_limit_no);
  m_closed = false;
}
```

/* 【注】`m_up_limit_id` = 活跃事务里**最小**的 id（没有活跃事务就等于 `m_low_limit_id`）。
   所以 `id < m_up_limit_id` 意味着"这个 id 比所有活跃事务都老" ⇒ 铁定可见。 */

#### `PAGE_MAX_TRX_ID`：页头偏移 18 的 8 字节，只增不减

`storage/innobase/include/page0types.h:69-80`：

```cpp
/** last insert direction: PAGE_LEFT, ... */
constexpr uint32_t PAGE_DIRECTION = 12;
/** number of consecutive inserts to the same direction */
constexpr uint32_t PAGE_N_DIRECTION = 14;
/** number of user records on the page */
constexpr uint32_t PAGE_N_RECS = 16;
/** highest id of a trx which may have modified a record on the page; trx_id_t;
defined only in secondary indexes and in the insert buffer tree */
constexpr uint32_t PAGE_MAX_TRX_ID = 18;
/** end of private data structure of the page header which are set in a page
create */
constexpr uint32_t PAGE_HEADER_PRIV_END = 26;
```

/* 【注】注释明写 `defined only in secondary indexes and in the insert buffer tree`
   ——聚簇索引页**不用**这个字段（聚簇记录自带 `DB_TRX_ID`，走 `changes_visible()` 逐记录判）。 */

三个访问器（`storage/innobase/include/page0page.ic:58-91`）：

```cpp
/** Returns the max trx id field value. */
static inline trx_id_t page_get_max_trx_id(const page_t *page) /*!< in: page */
{
  ut_ad(page);

  return (mach_read_from_8(page + PAGE_HEADER + PAGE_MAX_TRX_ID));
}

/** Sets the max trx id field value if trx_id is bigger than the previous
 value. */
static inline void page_update_max_trx_id(
    buf_block_t *block,       /*!< in/out: page */
    page_zip_des_t *page_zip, /*!< in/out: compressed page whose
                             uncompressed part will be updated, or NULL */
    trx_id_t trx_id,          /*!< in: transaction id */
    mtr_t *mtr)               /*!< in/out: mini-transaction */
{
  ut_ad(block);
#ifndef UNIV_HOTBACKUP
  ut_ad(mtr_memo_contains(mtr, block, MTR_MEMO_PAGE_X_FIX));
#endif /* !UNIV_HOTBACKUP */
  /* During crash recovery, this function may be called on
  something else than a leaf page of a secondary index or the
  insert buffer index tree (dict_index_is_sec_or_ibuf() returns
  true for the dummy indexes constructed during redo log
  application).  In that case, PAGE_MAX_TRX_ID is unused,
  and trx_id is usually zero. */
  ut_ad(trx_id || recv_recovery_is_on());
  ut_ad(page_is_leaf(buf_block_get_frame(block)));

  if (page_get_max_trx_id(buf_block_get_frame(block)) < trx_id) {
    page_set_max_trx_id(block, page_zip, trx_id, mtr);
  }
}
```

/* 【注】"只增不减"就落在最后那个 `if (<)` 上——这是粗筛能用的**唯一**前提。 */

#### `PAGE_MAX_TRX_ID` 的 redo 行为与一条陈旧注释

`storage/innobase/page/page0page.cc:192-219`：

```cpp
/** Sets the max trx id field value.
@param[in,out] block Page
@param[in,out] page_zip Compressed page, or NULL
@param[in] trx_id Transaction id
@param[in,out] mtr Mini-transaction, or NULL */
void page_set_max_trx_id(buf_block_t *block, page_zip_des_t *page_zip,
                         trx_id_t trx_id, mtr_t *mtr) {
  page_t *page = buf_block_get_frame(block);
#ifndef UNIV_HOTBACKUP
  ut_ad(!mtr || mtr_memo_contains(mtr, block, MTR_MEMO_PAGE_X_FIX));
#endif /* !UNIV_HOTBACKUP */

  /* It is not necessary to write this change to the redo log, as
  during a database recovery we assume that the max trx id of every
  page is the maximum trx id assigned before the crash. */

  if (page_zip) {
    mach_write_to_8(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), trx_id);
    page_zip_write_header(page_zip, page + (PAGE_HEADER + PAGE_MAX_TRX_ID), 8,
                          mtr);
#ifndef UNIV_HOTBACKUP
  } else if (mtr) {
    mlog_write_ull(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), trx_id, mtr);
#endif /* !UNIV_HOTBACKUP */
  } else {
    mach_write_to_8(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), trx_id);
  }
}
```

/* 【注】★ 上面第 301-303 行那条"It is not necessary to write this change to the redo log"
   注释与紧随其后的代码**自相矛盾**：非压缩分支直接走 `mlog_write_ull(..., mtr)`，
   压缩分支走 `page_zip_write_header(..., mtr)`，两者都产生 redo。 */

`mlog_write_ull` 本体（`storage/innobase/mtr/mtr0log.cc:296-322`）：

```cpp
/** Writes 8 bytes to a file page. Writes the corresponding log
 record to the mini-transaction log, only if mtr is not NULL */
void mlog_write_ull(byte *ptr,    /*!< in: pointer where to write */
                    uint64_t val, /*!< in: value to write */
                    mtr_t *mtr)   /*!< in: mini-transaction handle */
{
  mach_write_to_8(ptr, val);

  if (mtr == nullptr) {
    return;
  }

  /* If no logging is requested, we may return now */
  byte *log_ptr = nullptr;
  if (!mlog_open(mtr, REDO_LOG_INITIAL_INFO_SIZE + 2 + 9, log_ptr)) {
    return;
  }

  log_ptr = mlog_write_initial_log_record_fast(ptr, MLOG_8BYTES, log_ptr, mtr);

  mach_write_to_2(log_ptr, page_offset(ptr));
  log_ptr += 2;

  log_ptr += mach_u64_write_compressed(log_ptr, val);

  mlog_close(mtr, log_ptr);
}
```

/* 【注】唯一的"不写"短路是 `mtr == nullptr` 或 `mtr_get_log_mode(mtr)` 为
   `MTR_LOG_NONE` / `MTR_LOG_NO_REDO`（`mtr0log.ic:46-51`）。 */

全部 `page_update_max_trx_id` 调用点（15 处，脚本 grep）：

| file:line | 场景 |
|---|---|
| `storage/innobase/row/row0ins.cc:3054` | 二级索引插入（`row_ins_sec_index_entry_low` 尾） |
| `storage/innobase/row/row0umod.cc:714` | UPDATE 的 delete-mark + insert |
| `storage/innobase/btr/btr0cur.cc:3213` | pessimistic insert（页分裂后） |
| `storage/innobase/btr/btr0cur.cc:4294` | pessimistic update 回填（`BTR_NO_LOCKING_FLAG` 时补更新） |
| `storage/innobase/lock/lock0lock.cc:5468` | `lock_rec_inherit_to_gap_if_gap_lock` |
| `storage/innobase/lock/lock0lock.cc:5692` | `lock_rec_convert_impl_to_expl`（二级索引加锁成功后） |
| `storage/innobase/lock/lock0prdt.cc:509` | spatial 谓词锁插入 |
| `storage/innobase/ibuf/ibuf0ibuf.cc:3238` | ibuf 插入（ibuf 树本身是二级索引树） |
| `storage/innobase/ibuf/ibuf0ibuf.cc:4146` | ibuf merge 回真实二级索引页 |
| `storage/innobase/page/page0page.cc:455 / 599 / 739` | 页重建 / reorg |
| `storage/innobase/gis/gis0rtree.cc:824` | rtree 页分裂拷页 |
| `storage/innobase/btr/btr0load.cc:386` | 建索引 bulk load |
| `storage/innobase/ddl/ddl0rtree.cc:184` | DDL 构建 rtree |
| `storage/innobase/row/row0log.cc:3374` | online DDL 行日志回放 |

其中 `btr0cur.cc:4282-4296` 的原文（顺带说明临时表为什么被排除）：

```cpp
  /* Multiple transactions cannot simultaneously operate on the
  same temp-table in parallel.
  max_trx_id is ignored for temp tables because it not required
  for MVCC. */
  if (dict_index_is_sec_or_ibuf(index) && !index->table->is_temporary()) {
    /* Update PAGE_MAX_TRX_ID in the index page header.
    It was not updated by btr_cur_pessimistic_insert()
    because of BTR_NO_LOCKING_FLAG. */
    buf_block_t *rec_block;

    rec_block = btr_cur_get_block(cursor);

    page_update_max_trx_id(rec_block, buf_block_get_page_zip(rec_block), trx_id,
                           mtr);
  }
```

★ **正确表述**：8.0.39 里 `PAGE_MAX_TRX_ID` 的更新是**写 redo 的**（`MLOG_8BYTES`）。真正保证"宁可慢不可错"的机制不是"不写 redo"，而是**崩溃恢复期间一律走保守路径**——`lock_sec_rec_cons_read_sees()` 在 `recv_recovery_is_on()` 时直接返回 `false`（强制回表），`lock_rec_convert_impl_to_expl` 也用 `if (!recv_recovery_is_on() && !can_older_trx_be_still_active(max_trx_id))` 绕开（`lock0lock.cc:1054`）。真正不写 redo 的只有 `mtr == nullptr` 的两处（`row0import.cc:2479`、`page0zip.cc:2607`）与临时表 / `MTR_LOG_NO_REDO` 场景——而临时表根本不参与 MVCC。

★ **代价**（去掉了"失真"那条）：页级粒度粗——一条新记录会让**整页**失去粗筛资格；且 `page_update_max_trx_id` 只增不减，**没有任何路径把它降下来**，所以一个页一旦被新事务碰过就永久失去粗筛资格（除非页重建时由 `page_set_max_trx_id` 从旧页拷回原值，见 `btr0btr.cc:1215` "Copy max trx id to recreated page"）。

### 为什么唯一性只能靠锁：MVCC 管不了"未来"

MVCC 回答的是"**过去**：某个历史版本对我是否可见"。唯一性要求的是"**现在 + 未来**：不允许另一个事务插入（或即将插入）相同的值"。

★ **约束**：快照读看不见其他活跃事务的未提交插入，而唯一性必须阻止它们。唯一能表达"阻止未来插入"的机制是**锁**——所以唯一性检查必然是"先加锁、后判重"，且 record S 锁在 RC/RR 下都要加（差别只在 gap 部分）。

### 为什么 purge 与读共享同一个水位

purge 能推进多远受**最老 ReadView**（`purge_sys->view`）约束；`row_vers_old_has_index_entry` 的 safe 边界也是"版本链底端由 purge view 保证"。两者共享同一水位，保证：

> 任何"还在被读"的版本，其索引项都不会被 purge 物理删除。

★ 这让版本链回溯可以**无锁**遍历：顶端由 mtr 的页 latch 锁住（不会被并发修改），底端由 purge view 兜住（不会被 purge 抽走）——两头都有安全边界，中间自然安全。

这个"底端闸门"就是 `trx_undo_get_undo_rec()`（`storage/innobase/trx/trx0rec.cc:2440-2461`）：

```cpp
  missing_history = purge_sys->view.changes_visible(trx_id, name);
  if (!missing_history) {
    *undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
  }
```

/* 【注】★ 这里用的不是"读事务的 view"，而是 **purge_sys->view**（全库最老的 view）。
   判据是"连最老的 view 都能看见这个 trx_id" ⇒ 它的 undo 已可被清理 ⇒ 返回
   `missing_history = true`，**不去取** undo 记录。这就是"底端由 purge view 兜住"的代码落点：
   一旦返回 false（`DB_MISSING_HISTORY`），回溯立刻停止，调用方得到"历史不可得"。 */

读侧的版本链回溯入口 `row_vers_build_for_consistent_read()`（`storage/innobase/row/row0vers.cc:1236-1330`）：

```cpp
dberr_t row_vers_build_for_consistent_read(
    const rec_t *rec, mtr_t *mtr, dict_index_t *index, ulint **offsets,
    ReadView *view, mem_heap_t **offset_heap, mem_heap_t *in_heap,
    rec_t **old_vers, const dtuple_t **vrow, lob::undo_vers_t *lob_undo) {
  DBUG_TRACE;
  const rec_t *version;
  rec_t *prev_version;
  trx_id_t trx_id;
  mem_heap_t *heap = nullptr;
  byte *buf;
  dberr_t err;

  ut_ad(index->is_clustered());
  ut_ad(mtr_memo_contains_page(mtr, rec, MTR_MEMO_PAGE_X_FIX) ||
        mtr_memo_contains_page(mtr, rec, MTR_MEMO_PAGE_S_FIX));
  ut_ad(!rw_lock_own(&(purge_sys->latch), RW_LOCK_S));

  ut_ad(rec_offs_validate(rec, index, *offsets));

  trx_id = row_get_rec_trx_id(rec, index, *offsets);

  /* Reset the collected LOB undo information. */
  if (lob_undo != nullptr) {
    lob_undo->reset();
  }

  ut_ad(!view->changes_visible(trx_id, index->table->name));

  ut_ad(!vrow || !(*vrow));

  version = rec;

  for (;;) {
    mem_heap_t *prev_heap = heap;

    heap = mem_heap_create(1024, UT_LOCATION_HERE);

    if (vrow) {
      *vrow = nullptr;
    }

    /* If purge can't see the record then we can't rely on
    the UNDO log record. */

    bool purge_sees =
        trx_undo_prev_version_build(rec, mtr, version, index, *offsets, heap,
                                    &prev_version, nullptr, vrow, 0, lob_undo);

    err = (purge_sees) ? DB_SUCCESS : DB_MISSING_HISTORY;

    if (prev_heap != nullptr) {
      mem_heap_free(prev_heap);
    }

    if (prev_version == nullptr) {
      /* It was a freshly inserted version */
      *old_vers = nullptr;
      ut_ad(!vrow || !(*vrow));
      break;
    }

    *offsets = rec_get_offsets(prev_version, index, *offsets, ULINT_UNDEFINED,
                               UT_LOCATION_HERE, offset_heap);
    ...
    trx_id = row_get_rec_trx_id(prev_version, index, *offsets);

    if (view->changes_visible(trx_id, index->table->name)) {
      /* The view already sees this version: we can copy
```

/* 【注】三个 `ut_ad` 就是"无锁遍历"的契约：① 必须持 `rec` 所在页的 latch（锁住栈顶）；
   ② **不得**持有 `purge_sys->latch`（否则 purge 被阻塞，回溯永远拿到底端）；
   ③ 进来时 `view` 一定看不见当前版本（否则不该回溯）。
   循环里每回溯一版就用 `view->changes_visible()` 判一次，命中即拷出 `old_vers` 并 break。 */

⚠️ **真名**：8.0.39 里**没有** `row_vers_build_prev_vers()`（全仓 0 命中），该函数已更名为 `row_vers_build_for_consistent_read()`。老的 `row0sel.cc` 内部路径（`row_sel_build_prev_vers()`、`row0sel.cc:688`）仍然存在，但它只是后者的薄封装。

---

## 核心实现

### 二级索引没有 trx_id/roll_ptr 的三个后果

源码注释（`row0sel.cc`）原文：

这段注释在 `storage/innobase/row/row0sel.cc:5385-5413`，它后面紧跟的就是"粗筛失败 → 回表"的分派：

```cpp
    } else {
      /* We are looking into a non-clustered index,
      and to get the right version of the record we
      have to look also into the clustered index: this
      is necessary, because we can only get the undo
      information via the clustered index record. */

      ut_ad(!index->is_clustered());

      if (!srv_read_only_mode &&
          !lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
        /* We should look at the clustered index.
        However, as this is a non-locking read,
        we can skip the clustered index lookup if
        the condition does not match the secondary
        index entry. */
        switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
          case ICP_NO_MATCH:
            goto next_rec;
          case ICP_OUT_OF_RANGE:
            err = DB_RECORD_NOT_FOUND;
            goto idx_cond_failed;
          case ICP_MATCH:
            goto requires_clust_rec;
        }

        ut_error;
      }
    }
```

/* 【注】★ 分派是三路的：ICP 明确不匹配 → 直接跳过（**不回表**）；
   越界 → 结束扫描；只有 ICP 匹配才 `goto requires_clust_rec`。
   这就是"ICP 能省掉一部分回表"的落点。若 `prebuilt->idx_cond == false`，
   `row_search_idx_cond_check()` 直接返回 `ICP_MATCH`（`row0sel.cc:3797-3799`），
   等价于无条件回表。 */

⚠️ **真名**：`row_search_mvcc()` 里的 `requires_clust_rec` 是 **goto 标签**，不是变量（全仓只有 7 处命中：`row0sel.cc:1456/1746/1801/1827/1842/5408/5478`）。真正叫**变量**的是老式 `row_sel_try_search_shortcut()` 路径里的 `cons_read_requires_clust_rec`，其唯一置位点在 `row0sel.cc:1799-1802`：

```cpp
    } else if (!srv_read_only_mode &&
               !lock_sec_rec_cons_read_sees(rec, index, node->read_view)) {
      cons_read_requires_clust_rec = true;
    }
```

进入回表只有两条路：**(A)** 上面那个 `goto`（粗筛不确定 + ICP 匹配）；**(B)** 5477 行 `if (index != clust_index && prebuilt->need_to_access_clustered)` 自然 fall-through（覆盖列不够，与 MVCC 无关）。

1. **判断可见性必须回表**：`lock_sec_rec_cons_read_sees` 只能粗筛，粗筛不过就 `requires_clust_rec`；
2. **ICP 在二级索引上判断的是当前版本**（`row_search_idx_cond_check` 的 `rec` 是当前索引记录）——不匹配可省掉回表（保守剪枝），但匹配与否最终由回表后的 old_vers 兜底；
3. **同一 key 的多个 delete-marked 索引项可以并存**（各对应不同 PK）——这直接说明"为什么不能直接删"。

### delete-mark 两阶段与 `row_vers_old_has_index_entry`

**阶段一：打 delete-mark**。`btr_cur_del_mark_set_sec_rec` 只做三件事：加锁检查 → 置位 → 写 redo。原文（`storage/innobase/btr/btr0cur.cc:4595-4634`）：

```cpp
/** Sets a secondary index record delete mark to true or false.
 @return DB_SUCCESS, DB_LOCK_WAIT, or error number */
dberr_t btr_cur_del_mark_set_sec_rec(
    ulint flags,       /*!< in: locking flag */
    btr_cur_t *cursor, /*!< in: cursor */
    bool val,          /*!< in: value to set */
    que_thr_t *thr,    /*!< in: query thread */
    mtr_t *mtr)        /*!< in/out: mini-transaction */
{
  buf_block_t *block;
  rec_t *rec;
  dberr_t err;

  block = btr_cur_get_block(cursor);
  rec = btr_cur_get_rec(cursor);

  err = lock_sec_rec_modify_check_and_lock(flags, btr_cur_get_block(cursor),
                                           rec, cursor->index, thr, mtr);
  if (err != DB_SUCCESS) {
    return (err);
  }

  ut_ad(page_rec_is_comp(rec) == dict_table_is_comp(cursor->index->table));

  DBUG_PRINT("ib_cur",
             ("delete-mark=%u sec %u:%u:%u in %s(" IB_ID_FMT ") by " TRX_ID_FMT,
              unsigned(val), block->page.id.space(), block->page.id.page_no(),
              unsigned(page_rec_get_heap_no(rec)), cursor->index->name(),
              cursor->index->id, trx_get_id_for_print(thr_get_trx(thr))));

  /* We do not need to reserve search latch, as the
  delete-mark flag is being updated in place and the adaptive
  hash index does not depend on it. */
  btr_rec_set_deleted_flag(rec, buf_block_get_page_zip(block), val);

  btr_cur_del_mark_set_sec_rec_log(rec, val, mtr);

  return (DB_SUCCESS);
}
```

/* 【注】① `lock_sec_rec_modify_check_and_lock()` 在前——打标记本身是一次**修改**，要拿锁；
   ② "不需要 reserve search latch" 是因为 AHI 不依赖 delete-mark 位（这是自适应的一个省 latch 手法）；
   ③ redo 类型是 `MLOG_REC_SEC_DELETE_MARK`（`btr_cur_del_mark_set_sec_rec_log`，`btr0cur.cc:4531-4555`）。 */

调用点：`row0umod.cc:496`（UPDATE 改索引列）、`row0umod.cc:724`、`row0upd.cc:2133 / 2418`、`row0mysql.cc:1325`、`log0ddl.cc:804`（DDL log 表）。

**两阶段的边界**（回滚 UPDATE 时最直白）：

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

**`row_vers_old_has_index_entry`**（`storage/innobase/row/row0vers.cc:977-1234`）。先看声明（`storage/innobase/include/row0vers.h:74-91`）：

```cpp
/** Finds out if a version of the record, where the version >= the current
 purge view, should have ientry as its secondary index entry. We check
 if there is any not delete marked version of the record where the trx
 id >= purge view, and the secondary index entry == ientry; exactly in
 this case we return true.
 @return true if earlier version should have */
bool row_vers_old_has_index_entry(
    bool also_curr,         /*!< in: true if also rec is included in the
                           versions to search; otherwise only versions
                           prior to it are searched */
    const rec_t *rec,       /*!< in: record in the clustered index; the
                            caller must have a latch on the page */
    mtr_t *mtr,             /*!< in: mtr holding the latch on rec; it will
                            also hold the latch on purge_view */
    dict_index_t *index,    /*!< in: the secondary index */
    const dtuple_t *ientry, /*!< in: the secondary index entry */
    roll_ptr_t roll_ptr,    /*!< in: roll_ptr for the purge record */
    trx_id_t trx_id);       /*!< in: transaction ID on the purging record */
/* 【注】首参确实叫 `also_curr`：purge 侧传 true（row0purge.cc:287，把**当前版本**也算进搜索范围），
   回滚 UPDATE 侧传 false（row0umod.cc:493，只看当前版本**之前**的版本）。两者语义不同。 */

```

定义开头（前提断言 + 两个安全边界）：

```cpp
bool row_vers_old_has_index_entry(bool also_curr, const rec_t *rec, mtr_t *mtr,
                                  dict_index_t *index, const dtuple_t *ientry,
                                  roll_ptr_t roll_ptr, trx_id_t trx_id) {
  ...
  ut_ad(mtr_memo_contains_page(mtr, rec, MTR_MEMO_PAGE_X_FIX) ||
        mtr_memo_contains_page(mtr, rec, MTR_MEMO_PAGE_S_FIX));
  ut_ad(!rw_lock_own(&(purge_sys->latch), RW_LOCK_S));

  clust_index = index->table->first_index();

  comp = page_rec_is_comp(rec);
  ut_ad(!dict_table_is_comp(index->table) == !comp);
  heap = mem_heap_create(1024, UT_LOCATION_HERE);
  clust_offsets = rec_get_offsets(rec, clust_index, nullptr, ULINT_UNDEFINED,
                                  UT_LOCATION_HERE, &heap);
  ...
  if (dict_index_has_virtual(index)) {
    v_heap = mem_heap_create(100, UT_LOCATION_HERE);
  }

  if (also_curr && !rec_get_deleted_flag(rec, comp)) {
    row_ext_t *ext;

    /* The top of the stack of versions is locked by the
    mtr holding a latch on the page containing the
    clustered index record. The bottom of the stack is
    locked by the fact that the purge_sys->view must
    'overtake' any read view of an active transaction.
    Thus, it is safe to fetch the prefixes for
    externally stored columns. */
    row = row_build(ROW_COPY_POINTERS, clust_index, rec, clust_offsets, nullptr,
                    nullptr, nullptr, &ext, heap);
```

/* 【注】★ 这段注释就是"两头都有安全边界"的**原文出处**：栈顶由 mtr 的页 latch 锁住，
   栈底由 purge_sys->view 必然"超车"任何活跃 read view 来保证。 */

非虚拟列的普通二级索引路径（含那段著名的 collation 注释，`row0vers.cc:1123-1133`）：

```cpp
    } else {
      entry = row_build_index_entry(row, ext, index, heap);

      /* If entry == NULL, the record contains unset BLOB
      pointers.  This must be a freshly inserted record.  If
      this is called from
      row_purge_remove_sec_if_poss_low(), the thread will
      hold latches on the clustered index and the secondary
      index.  Because the insert works in three steps:

              (1) insert the record to clustered index
              (2) store the BLOBs and update BLOB pointers
              (3) insert records to secondary indexes

      the purge thread can safely ignore freshly inserted
      records and delete the secondary index record.  The
      thread that inserted the new record will be inserting
      the secondary index records. */

      /* NOTE that we cannot do the comparison as binary
      fields because the row is maybe being modified so that
      the clustered index record has already been updated to
      a different binary value in a char field, but the
      collation identifies the old and new value anyway! */
      if (entry && dtuple_coll_eq(entry, ientry)) {
        mem_heap_free(heap);

        if (v_heap) {
          mem_heap_free(v_heap);
        }
        return true;
      }
    }
```

/* 【注】★ 第二个注释块解释了"三步插入"：聚簇插入 → 存 BLOB 并回填指针 → 才插二级索引。
   所以 purge 撞上 `entry == nullptr`（BLOB 指针还没填）时，那一定是一条**正在插入中**的新记录，
   可以安全跳过——插入线程自己会把二级索引项补上。这是不用加锁就能处理的竞态。 */

回溯主循环（`row0vers.cc:1151-1234`：）

```cpp
  for (;;) {
    heap2 = heap;
    heap = mem_heap_create(1024, UT_LOCATION_HERE);
    vrow = nullptr;

    trx_undo_prev_version_build(
        rec, mtr, version, clust_index, clust_offsets, heap, &prev_version,
        nullptr, dict_index_has_virtual(index) ? &vrow : nullptr, 0, nullptr);
    mem_heap_free(heap2); /* free version and clust_offsets */

    if (!prev_version) {
      /* Versions end here */

      mem_heap_free(heap);

      if (v_heap) {
        mem_heap_free(v_heap);
      }

      return false;
    }

    clust_offsets = rec_get_offsets(prev_version, clust_index, nullptr,
                                    ULINT_UNDEFINED, UT_LOCATION_HERE, &heap);
    ...
    if (!rec_get_deleted_flag(prev_version, comp)) {
      row_ext_t *ext;

      /* The stack of versions is locked by mtr.
      Thus, it is safe to fetch the prefixes for
      externally stored columns. */
      row = row_build(ROW_COPY_POINTERS, clust_index, prev_version,
                      clust_offsets, nullptr, nullptr, nullptr, &ext, heap);
      ...
      entry = row_build_index_entry(row, ext, index, heap);

      /* NOTE that we cannot do the comparison as binary
      fields because maybe the secondary index record has
      already been updated to a different binary value in
      a char field, but the collation identifies the old
      and new value anyway! */

      if (entry && dtuple_coll_eq(entry, ientry)) {
        mem_heap_free(heap);
        if (v_heap) {
          mem_heap_free(v_heap);
        }

        return true;
      }
    }

    version = prev_version;
  }
```

/* 【注】① `prev_version == nullptr` ⇒ 版本链到底 ⇒ `return false`（**可删**）；
   ② `rec_get_deleted_flag(prev_version)` 为真 ⇒ 整段跳过（delete-marked 的旧版本不算"还需要"）；
   ③ 每轮 `mem_heap_free(heap2)` 释放上一版的 heap——回溯是有界的，`heap` 不会累积。 */

★ 虚拟列分支的判据是 **`dict_index_has_virtual(index)`**（`row0vers.cc:1029/1046/1139/1158/1176/1202`），比的是 `row_vers_vc_matches_cluster()`（`row0vers.cc:1092`）；**`row_vers_find_matching` 不在这里**——它确实存在（`row0vers.cc:232`），但服务于 `row_vers_impl_x_locked`（调用点 `row0vers.cc:523`），与 purge 判定无关。

三个关键点：

- **必须用 collation 比较**（`dtuple_coll_eq`）而非二进制比较——CHAR 列可能已被改成另一个二进制串，但 collation 下仍相等（源码注释明说）；
- **delete-marked 的旧版本跳过**——只关心"仍然存活"的版本；
- **版本链的安全边界**：顶端由 mtr 的页 latch 锁住，底端由 purge view 保证（purge view 一定"老于"任何活跃 read view）——这两点使无锁遍历版本链是安全的。

### 读路径：页级粗筛 + 回表

`storage/innobase/lock/lock0lock.cc:259-297` 全文：

```cpp
/** Checks that a non-clustered index record is seen in a consistent read.

 NOTE that a non-clustered index page contains so little information on
 its modifications that also in the case false, the present version of
 rec may be the right, but we must check this from the clustered index
 record.

 @return true if certainly sees, or false if an earlier version of the
 clustered index record might be needed */
bool lock_sec_rec_cons_read_sees(
    const rec_t *rec,          /*!< in: user record which
                               should be read or passed over
                               by a read cursor */
    const dict_index_t *index, /*!< in: index */
    const ReadView *view)      /*!< in: consistent read view */
{
  ut_ad(page_rec_is_user_rec(rec));

  /* NOTE that we might call this function while holding the search
  system latch. */

  if (recv_recovery_is_on()) {
    return (false);

  } else if (index->table->is_temporary()) {
    /* Temp-tables are not shared across connections and multiple
    transactions from different connections cannot simultaneously
    operate on same temp-table and so read of temp-table is
    always consistent read. */

    return (true);
  }

  trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));

  ut_ad(max_trx_id > 0);

  return (view->sees(max_trx_id));
}
```

/* 【注】① 函数头注释就把"false 不代表不可见"讲死了：false 只意味着"**可能**需要更早版本"；
   ② `if / else if` 的**顺序**有意义：`recv_recovery_is_on()` 先于 `is_temporary()`，
   所以崩溃恢复期间**连临时表也返回 false**；
   ③ 与聚簇版 `lock_clust_rec_cons_read_sees()`（`lock0lock.cc:242`）不同——后者是
   `srv_read_only_mode || is_temporary()`，而**本函数里没有 `srv_read_only_mode` 判断**，
   只读模式的守卫放在调用方（`row0sel.cc:5394` 的 `!srv_read_only_mode &&`）；
   ④ 末尾 `ut_ad(max_trx_id > 0)` 要求页上必须有有效值，这也解释了为什么插入路径
   要无条件先 `page_update_max_trx_id()`。 */

整页共享一个 `PAGE_MAX_TRX_ID`（只增不减）：`view->sees(max_trx_id)` → 页上**所有**记录都可见（连 delete-mark 判定都只需看标记位）；否则必须 `requires_clust_rec` 回表精确判定。

### 回表三步：定位 → 构造 old_vers → 校验"这条二级项真属于这个版本"

回表的主体是 `Row_sel_get_clust_rec_for_mysql::operator()`（`storage/innobase/row/row0sel.cc:3149-3161`，定位）：

```cpp
  *out_rec = nullptr;
  trx = thr_get_trx(thr);

  row_build_row_ref_in_tuple(prebuilt->clust_ref, rec, sec_index, *offsets);

  clust_index = sec_index->table->first_index();

  prebuilt->clust_pcur->open_no_init(clust_index, prebuilt->clust_ref,
                                     PAGE_CUR_LE, BTR_SEARCH_LEAF, 0, mtr,
                                     UT_LOCATION_HERE);

  clust_rec = prebuilt->clust_pcur->get_rec();
```

/* 【注】`row_build_row_ref_in_tuple()` 把二级索引项里的**隐式主键**拼成 `clust_ref`，
   再用 `PAGE_CUR_LE` 定位——和 purge 的 `row_search_on_row_ref()` 是同一套思路
   （按主键重新定位，而不是靠任何物理位置）。 */

构造 old_vers（`row0sel.cc:3298-3330`）：

```cpp
  } else {
    /* This is a non-locking consistent read: if necessary, fetch
    a previous version of the record */

    old_vers = nullptr;

    /* If the isolation level allows reading of uncommitted data,
    then we never look for an earlier version */

    if (trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
        !lock_clust_rec_cons_read_sees(clust_rec, clust_index, *offsets,
                                       trx_get_read_view(trx))) {
      if (clust_rec != cached_clust_rec) {
        /* The following call returns 'offsets' associated with 'old_vers' */
        err = row_sel_build_prev_vers_for_mysql(
            trx->read_view, clust_index, prebuilt, clust_rec, offsets,
            offset_heap, &old_vers, vrow, mtr, lob_undo);

        if (err != DB_SUCCESS) {
          goto err_exit;
        }
        cached_clust_rec = clust_rec;
        cached_old_vers = old_vers;
      } else {
        err = DB_SUCCESS;
        old_vers = cached_old_vers;
        ...
      }

      if (old_vers == nullptr) {
        goto err_exit;
      }

      clust_rec = old_vers;
    }
```

/* 【注】① `cached_clust_rec` / `cached_old_vers` 是**同一条二级索引项连续回表**的缓存
   （`Row_sel_get_clust_rec_for_mysql` 类就为这两个成员而存在）——同一个 key 的多个
   delete-marked 项往往指向同一条聚簇记录，缓存能省掉重复的 undo 回溯；
   ② `isolation_level > TRX_ISO_READ_UNCOMMITTED` ⇒ RU 下**永不回溯**。 */

第三步的必要性写在源码注释里（`row0sel.cc:3310-3321`）：

```cpp
    /* If we had to go to an earlier version of row or the
    secondary index record is delete marked, then it may be that
    the secondary index record corresponding to clust_rec
    (or old_vers) is not rec; in that case we must ignore
    such row because in our snapshot rec would not have existed.
    Remember that from rec we cannot see directly which transaction
    id corresponds to it: we have to go to the clustered index
    record. A query where we want to fetch all rows where the
    secondary index value is in some interval would return
    a wrong result if we would not drop rows which we come to
    visit through secondary index records that would not really
    exist in our snapshot. */

    /* And for spatial index, since the rec is from shadow buffer,
    so we need to check if it's exactly match the clust_rec. */
    if (clust_rec &&
        (old_vers || trx->isolation_level <= TRX_ISO_READ_UNCOMMITTED ||
         dict_index_is_spatial(sec_index) ||
         rec_get_deleted_flag(rec, dict_table_is_comp(sec_index->table)))) {
      bool rec_equal;

      err = row_sel_sec_rec_is_for_clust_rec(rec, sec_index, clust_rec,
                                             clust_index, thr, rec_equal);
      if (err != DB_SUCCESS) {
        goto err_exit;
      } else if (!rec_equal) {
        clust_rec = nullptr;
      }
    }
```

/* 【注】★ 这段注释是本篇"为什么二级索引不能自己判可见性"的**权威表述**：
   从 `rec`（二级索引项）**看不出**它对应哪个 trx_id，必须回到聚簇记录才知道；
   而回到聚簇记录后看到的版本可能已经**不再需要**这条二级索引项了（值被改过），
   所以必须再比一次——否则区间查询会返回"快照里其实不存在"的行。 */

`row_sel_sec_rec_is_for_clust_rec()`（`row0sel.cc:162-346`）的比较核心：

```cpp
/** Returns true if the user-defined column values in a secondary index record
are alphabetically the same as the corresponding columns in the clustered
index record.
NOTE: the comparison is NOT done as a binary comparison, but character
fields are compared with collation!
...
@return DB_SUCCESS or error code */
static dberr_t row_sel_sec_rec_is_for_clust_rec(
    const rec_t *sec_rec, dict_index_t *sec_index, const rec_t *clust_rec,
    dict_index_t *clust_index, que_thr_t *thr, bool &is_equal) {
  ...
  is_equal = true;
  ...
  if (rec_get_deleted_flag(clust_rec, dict_table_is_comp(clust_index->table))) {
    /* The clustered index record is delete-marked;
    it is not visible in the read view.  Besides,
    if there are any externally stored columns,
    some of them may have been already purged. */
    is_equal = false;
    return (DB_SUCCESS);
  }
  ...
  n = dict_index_get_n_ordering_defined_by_user(sec_index);

  for (i = 0; i < n; i++) {
    ...
    /* For virtual column, its value will need to be
    reconstructed from base column in cluster index */
    if (col->is_virtual()) {
      ...
      vfield = innobase_get_computed_value(row, v_col, clust_index, &heap, heap,
                                           nullptr, thr_get_trx(thr)->mysql_thd,
                                           thr->prebuilt->m_mysql_table,
                                           nullptr, nullptr, nullptr);
      ...
    } else {
      clust_pos = dict_col_get_clust_pos(col, clust_index);

      clust_field = rec_get_nth_field_instant(clust_rec, clust_offs, clust_pos,
                                              clust_index, &clust_len);
    }

    sec_field = rec_get_nth_field(nullptr, sec_rec, sec_offs, i, &sec_len);
    ...
    } else if (col->is_multi_value()) {
      if (!is_multi_value_clust_and_sec_equal(clust_field, clust_len, sec_field,
                                              sec_len, col)) {
        is_equal = false;
        goto func_exit;
      }
    } else {
      /* We are testing for equality; ASC/DESC does not
      matter */
      if (0 != cmp_data_data(col->mtype, col->prtype, true, clust_field, len,
                             sec_field, sec_len)) {
        is_equal = false;
        goto func_exit;
      }
    }
  }
```

/* 【注】① 循环只比 `dict_index_get_n_ordering_defined_by_user()` 个**用户定义**排序列，
   不含隐式主键；② 虚拟列要**现算**（`innobase_get_computed_value`）；
   ③ 多值列走 `is_multi_value_clust_and_sec_equal`；④ 普通列用 `cmp_data_data`
   （**collation 比较**，与 `dtuple_coll_eq` 同源思路）；⑤ 聚簇记录是 delete-marked
   就直接 `is_equal = false`——因为它的 extern 列可能已被 purge。 */

### delete-marked 记录对查询的影响

`row_search_mvcc` 主分支（`storage/innobase/row/row0sel.cc:5426-5472`）：

```cpp
  /* NOTE that at this point rec can be an old version of a clustered
  index record built for a consistent read. We cannot assume after this
  point that rec is on a buffer pool page. Functions like
  page_rec_is_comp() cannot be used! */

  if (rec_get_deleted_flag(rec, comp)) {
    /* The record is delete-marked: we can skip it */

    /* No need to keep a lock on a delete-marked record in lower isolation
    levels - it's similar to when Server sees the WHERE condition doesn't match
    and calls unlock_row(). */
    prebuilt->try_unlock(true);

    /* This is an optimization to skip setting the next key lock
    on the record that follows this delete-marked record. This
    optimization works because of the unique search criteria
    which precludes the presence of a range lock between this
    delete marked record and the record following it.

    For now this is applicable only to clustered indexes while
    doing a unique search except for HANDLER queries because
    HANDLER allows NEXT and PREV even in unique search on
    clustered index. There is scope for further optimization
    applicable to unique secondary indexes. Current behaviour is
    to widen the scope of a lock on an already delete marked record
    if the same record is deleted twice by the same transaction */
    if (index == clust_index && unique_search && !prebuilt->used_in_HANDLER) {
      err = DB_RECORD_NOT_FOUND;

      goto normal_return;
    }

    goto next_rec;
  }
```

/* 【注】① 开头那条 NOTE 很关键：此刻的 `rec` **可能已是被回溯出来的 old_vers**，
   不在 buffer pool 页上，所以 `page_rec_is_comp()` 之类**不能用**（这里用的是预先算好的 `comp`）；
   ② 提前 `try_unlock(true)` 是"低隔离级别下 delete-marked 记录不必持锁"的优化；
   ③ `DB_RECORD_NOT_FOUND` 的短路**只对聚簇索引的唯一搜索**生效——二级索引没有这个优化，
   注释里明说 "There is scope for further optimization applicable to unique secondary indexes"。 */

回表之后还有**第二次** delete-mark 判定（`row0sel.cc:5587-5596`）——因为回表拿到的 `clust_rec` 可能是 old_vers，它自己也可能是 delete-marked：

```cpp
    if (rec_get_deleted_flag(clust_rec, comp)) {
      /* The record is delete marked: we can skip it */

      /* No need to keep a lock on a delete-marked record in lower isolation
      levels - it's similar to when Server sees the WHERE condition doesn't match
      and calls unlock_row(). */
      prebuilt->try_unlock(true);

      goto next_rec;
    }
```

以及末尾的收尾断言（`row0sel.cc:5605-5610`）：

```cpp
  /* We found a qualifying record 'result_rec'. At this point,
  'offsets' are associated with 'result_rec'. */

  ut_ad(rec_offs_validate(result_rec, result_rec != rec ? clust_index : index,
                          offsets));
  ut_ad(!rec_get_deleted_flag(result_rec, comp));
```

/* 【注】最后这句 `ut_ad(!rec_get_deleted_flag(result_rec, comp))` 是不变式：
   凡是**返回给 Server 的行**，一定不是 delete-marked。 */

- 二级索引上的 delete-mark 项：直接 `goto next_rec` 跳过（此刻 `rec` 可能已是 old_vers——不能假设它在 buffer pool 页上）；
- **但它仍占据 B-tree 槽位与锁空间**——加锁扫描时会锁住它，可能阻塞插入；
- 唯一二级索引有额外校验：`row_sel_sec_rec_is_for_clust_rec` 确认索引项确实对应那条聚簇记录（因为同 key 的多个 delete-marked 项并存，只 PK 不同）。

### purge：undo → 聚簇 ref → 索引定位 → 物理删除

```
trx_purge() → clone_oldest_view(&purge_sys->view)   // 取最老 read view 作水位
  → trx_purge_attach_undo_recs()                    // 按 rseg 批量取 undo
  → que_thr_step → row_purge_step()
      → row_purge_parse_undo_rec()                  // undo → node->ref（PK tuple）
      → row_purge_record_func()
          TRX_UNDO_DEL_MARK_REC → row_purge_del_mark()
          TRX_UNDO_UPD_EXIST_REC → row_purge_upd_exist_or_extern_func()
```

**定位**（`row_purge_parse_undo_rec`）：undo 记录里**没有物理位置**（记录可能已被页分裂搬走），只有主键值——`trx_undo_rec_get_row_ref` 取出 `node->ref` → `row_purge_reposition_pcur` → `row_search_on_row_ref` 重新定位聚簇记录；`node->row` 是"旧版本的部分行"（只含索引列），用于重建**旧**二级索引 entry。

函数尾部原文（`storage/innobase/row/row0purge.cc:1063-1087`）：

```cpp
  if (type == TRX_UNDO_UPD_EXIST_REC &&
      (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE) && !*updated_extern) {
    /* Purge requires no changes to indexes: we may return */
    goto close_exit;
  }

  ptr = trx_undo_rec_get_row_ref(ptr, clust_index, &(node->ref), node->heap);

  ptr = trx_undo_update_rec_get_update(ptr, clust_index, type, trx_id, roll_ptr,
                                       info_bits, node->heap, &(node->update),
                                       nullptr, type_cmpl);

  /* Read to the partial row the fields that occur in indexes */

  if (!(node->cmpl_info & UPD_NODE_NO_ORD_CHANGE)) {
    ptr = trx_undo_rec_get_partial_row(
        ptr, clust_index, &node->row, type == TRX_UNDO_UPD_DEL_REC, node->heap);
  }

  return (true);
}
```

/* 【注】① `UPD_NODE_NO_ORD_CHANGE` 是个关键短路：如果这次 UPDATE **没动任何索引列**
   （也没动 extern 字段），purge 完全可以不动索引，直接 `goto close_exit`；
   ② `node->ref`（主键 tuple）与 `node->row`（部分行）是**两个**不同的东西——前者用于定位，
   后者用于重建"旧"的二级索引 entry；③ `node->row` 只在"动了索引列"时才读，省一次构建。 */

重新定位的两个函数（`row0purge.cc:101-130` / `row0row.cc:828-862`）：

```cpp
/** Repositions the pcur in the purge node on the clustered index record,
 if found. If the record is not found, close pcur.
 @return true if the record was found */
static bool row_purge_reposition_pcur(
    ulint mode,         /*!< in: latching mode */
    purge_node_t *node, /*!< in: row purge node */
    mtr_t *mtr)         /*!< in: mtr */
{
  if (node->found_clust) {
    ut_ad(node->validate_pcur());

    node->found_clust =
        node->pcur.restore_position(mode, mtr, UT_LOCATION_HERE);

  } else {
    node->found_clust =
        row_search_on_row_ref(&node->pcur, mode, node->table, node->ref, mtr);

    if (node->found_clust) {
      node->pcur.store_position(mtr);
    }
  }

  /* Close the current cursor if we fail to position it correctly. */
  if (!node->found_clust) {
    node->pcur.close();
  }

  return (node->found_clust);
}
```

```cpp
/** Searches the clustered index record for a row, if we have the row reference.
 @return true if found */
bool row_search_on_row_ref(btr_pcur_t *pcur, /*!< out: persistent cursor, which
                                              must be closed by the caller */
                           ulint mode,       /*!< in: BTR_MODIFY_LEAF, ... */
                           dict_table_t *table, /*!< in: table */
                           const dtuple_t *ref, /*!< in: row reference */
                           mtr_t *mtr)          /*!< in/out: mtr */
{
  ulint low_match;
  rec_t *rec;
  dict_index_t *index;

  ut_ad(dtuple_check_typed(ref));

  index = table->first_index();

  ut_a(dtuple_get_n_fields(ref) == dict_index_get_n_unique(index));

  pcur->open(index, 0, ref, PAGE_CUR_LE, mode, mtr, UT_LOCATION_HERE);

  low_match = pcur->get_low_match();

  rec = pcur->get_rec();

  if (page_rec_is_infimum(rec)) {
    return false;
  }

  if (low_match != dtuple_get_n_fields(ref)) {
    return false;
  }

  return true;
}
```

/* 【注】① 用 `PAGE_CUR_LE` + 要求 `low_match == n_unique`（**全**主键字段匹配）才算找到——
   这是"按主键重新定位"的严格判据；② `row_purge_reposition_pcur` 有**两级**优化：
   `node->found_clust` 为真时直接 `restore_position()`（复用上次 store 的位置），
   只有第一次才真的 `row_search_on_row_ref()` 搜索。purge 对同一条 undo 要处理 N 个二级索引，
   这个 pcur 复用省掉 N-1 次搜索。 */

**顺序**（`row_purge_del_mark`）：**先所有二级索引、最后聚簇**：

`row_purge_del_mark()`（`storage/innobase/row/row0purge.cc:653-695`）的骨架——**完整原文见 [`../innodb/undo_log.md`](../innodb/undo_log.md)「单条 undo rec 的 purge」节**（purge 执行细节的主场在那里），这里只留索引侧要用的三个分派点：

```cpp
  while (node->index != nullptr) {
    /* skip corrupted secondary index */
    dict_table_skip_corrupt_index(node->index);

    row_purge_skip_uncommitted_virtual_index(node->index);
    ...
    if (node->index->type != DICT_FTS) {
      if (node->index->is_multi_value()) {
        row_purge_remove_multi_sec_if_poss(node, heap, false);
      } else {
        dtuple_t *entry = row_build_index_entry_low(
            node->row, nullptr, node->index, heap, ROW_BUILD_FOR_PURGE);
        row_purge_remove_sec_if_poss(node, node->index, entry);
      }

      mem_heap_empty(heap);
    }

    node->index = node->index->next();
  }

  return (row_purge_remove_clust_if_poss(node));
```

/* 【注】索引侧要记住的三点：① entry 用 `ROW_BUILD_FOR_PURGE` 从 `node->row`（**旧**版本的部分行）重建；
   ② 多值索引走 `row_purge_remove_multi_sec_if_poss`，整行删除时 `selected` 传 **false**；
   ③ 每个索引处理完 `mem_heap_empty(heap)`，heap 在循环外复用。 */

"先二级后聚簇"的**起点**在调用侧（`row0purge.cc:1105-1117`）：

```cpp
  clust_index = node->table->first_index();

  node->index = clust_index->next();
  ut_ad(!trx_undo_roll_ptr_is_insert(node->roll_ptr));

  switch (node->rec_type) {
    case TRX_UNDO_DEL_MARK_REC:
      purged = row_purge_del_mark(node);
```

/* 【注】★ `node->index = clust_index->next()` —— 链表起点**跳过聚簇索引**，
   所以 `while` 循环遍历的只有二级索引；循环结束后才 `row_purge_remove_clust_if_poss(node)`。
   这就是"先所有二级、最后聚簇"的代码证据（不是靠约定）。 */

/* 【注】`row_purge_del_mark` 里整行删除走多值分支时 `selected` 传的是 **`false`**
   （`row0purge.cc:675`），即"旧数组的每个元素都要删"；而 UPDATE 分支传 `!non_mv_upd`
   （`row0purge.cc:740`）。 */

★ 为什么这个顺序：保证"聚簇记录已删但二级索引项还在"的窗口**最小**（若反过来，中间崩溃会留下指向已删聚簇记录的索引项）。

⚠️ 本工作树 `row0purge.cc:679` 处有一条本地中文批注写着"row purge 时顺序为：clust rec -> sec rec"，与紧邻代码方向**相反**，属批注笔误，勿引用。

**什么时候能物理删**（`row_purge_poss_sec`）：

`storage/innobase/row/row0purge.cc:261-298` 全文：

```cpp
/** Determines if it is possible to remove a secondary index entry.
 Removal is possible if the secondary index entry does not refer to any
 not delete marked version of a clustered index record where DB_TRX_ID
 is newer than the purge view.

 NOTE: This function should only be called by the purge thread, only
 while holding a latch on the leaf page of the secondary index entry
 (or keeping the buffer pool watch on the page).  It is possible that
 this function first returns true and then false, if a user transaction
 inserts a record that the secondary index entry would refer to.
 However, in that case, the user transaction would also re-insert the
 secondary index record after purge has removed it and released the leaf
 page latch.
 @return true if the secondary index record can be purged */
bool row_purge_poss_sec(purge_node_t *node,    /*!< in/out: row purge node */
                        dict_index_t *index,   /*!< in: secondary index */
                        const dtuple_t *entry) /*!< in: secondary index entry */
{
  bool can_delete;
  mtr_t mtr;

  ut_ad(!index->is_clustered());
  mtr_start(&mtr);

  can_delete =
      !row_purge_reposition_pcur(BTR_SEARCH_LEAF, node, &mtr) ||
      !row_vers_old_has_index_entry(true, node->pcur.get_rec(), &mtr, index,
                                    entry, node->roll_ptr, node->trx_id);

  /* Persistent cursor is closed if reposition fails. */
  if (node->found_clust) {
    node->pcur.commit_specify_mtr(&mtr);
  } else {
    mtr_commit(&mtr);
  }

  return (can_delete);
}
```

/* 【注】① `||` 的短路顺序有讲究：先判"聚簇记录还在不在"，不在就直接 `can_delete = true`；
   ② 函数头注释承认**这个判定不是原子的**——可能先返回 true 再返回 false（用户事务并发插入），
   但没关系，因为用户事务随后会**自己重新插入**那条二级索引项；
   ③ `node->found_clust` 决定是否 `commit_specify_mtr`（保留 pcur 位置）还是普通 `mtr_commit`。 */

即：**聚簇记录找不到了（已被删）或没有任何存活旧版本需要这个 entry** → 可删。

删除分乐观/悲观两档。入口 `row_purge_remove_sec_if_poss()`（`row0purge.cc:578-615`）：

```cpp
/** Removes a secondary index entry if possible. */
static inline void row_purge_remove_sec_if_poss(
    purge_node_t *node,    /*!< in: row purge node */
    dict_index_t *index,   /*!< in: index */
    const dtuple_t *entry) /*!< in: index entry */
{
  ulint n_tries = 0;

  /*    fputs("Purge: Removing secondary record\n", stderr); */

  if (!entry) {
    /* The node->row must have lacked some fields of this
    index. This is possible when the undo log record was
    written before this index was created. */
    return;
  }

  if (row_purge_remove_sec_if_poss_leaf(node, index, entry)) {
    return;
  }
retry:
  auto success = row_purge_remove_sec_if_poss_tree(node, index, entry);
  /* The delete operation may fail if we have little
  file space left: TODO: easiest to crash the database
  and restart with more file space */

  if (!success && n_tries < BTR_CUR_RETRY_DELETE_N_TIMES) {
    n_tries++;

    std::this_thread::sleep_for(
        std::chrono::milliseconds(BTR_CUR_RETRY_SLEEP_TIME_MS));

    goto retry;
  }

  ut_a(success);
}
```

/* 【注】① `entry == nullptr` 直接返回——undo 记录写于"索引创建之前"时 `node->row` 缺字段，
   这是**在线加索引**与 purge 并存的历史兼容；② 重试耗尽是 `ut_a(success)`（**断言失败**，
   不是静默返回）；对比聚簇侧 `row_purge_remove_clust_if_poss()` 是 `return false`
   （`row0purge.cc:258`，允许"挂起 purge"）。 */

两个重试常量的真实值（`storage/innobase/include/btr0cur.h:769-776`）：

```cpp
/** If pessimistic delete fails because of lack of file space, there
is still a good change of success a little later.  Try this many
times. */
constexpr uint32_t BTR_CUR_RETRY_DELETE_N_TIMES = 100;
/** If pessimistic delete fails because of lack of file space, there
is still a good change of success a little later.  Sleep this many
milliseconds between retries. */
constexpr uint32_t BTR_CUR_RETRY_SLEEP_TIME_MS = 50;
```

/* 【注】★ 是 **100 次 × 50ms**（不是常见的"几次"说法），最坏要等 5 秒才触发 `ut_a`。 */

**乐观档** `row_purge_remove_sec_if_poss_leaf()`（模式选择是关键，`row0purge.cc:459-479`）：

```cpp
  if (!index->is_committed()) {
    /* For uncommitted spatial index, we also skip the purge. */
    if (dict_index_is_spatial(index)) {
      goto func_exit_no_pcur;
    }
    ...
    /* Change buffering is disabled for temporary tables. */
    mode = (index->table->is_temporary())
               ? BTR_MODIFY_LEAF | BTR_ALREADY_S_LATCHED
               : BTR_MODIFY_LEAF | BTR_ALREADY_S_LATCHED | BTR_DELETE;
  } else {
    ...
    /* Change buffering is disabled for temporary tables
    and spatial index. */
    mode = (index->table->is_temporary() || dict_index_is_spatial(index))
               ? BTR_MODIFY_LEAF
               : BTR_MODIFY_LEAF | BTR_DELETE;
  }
```

以及删之前的"只许删 delete-marked"校验（`row0purge.cc:502-528`）：

```cpp
    case ROW_FOUND:
      /* Before attempting to purge a record, check
      if it is safe to do so. */
      if (row_purge_poss_sec(node, index, entry)) {
        btr_cur_t *btr_cur = pcur.get_btr_cur();

        /* Only delete-marked records should be purged. */
        if (!rec_get_deleted_flag(btr_cur_get_rec(btr_cur),
                                  dict_table_is_comp(index->table))) {
          ib::error(ER_IB_MSG_1008)
              << "tried to purge non-delete-marked"
                 " record"
                 " in index "
              << index->name << " of table " << index->table->name
              << ": tuple: " << *entry << ", record: "
              << rec_index_print(btr_cur_get_rec(btr_cur), index);

          pcur.close();

          ut_d(ut_error);

          ut_o(goto func_exit_no_pcur);
        }
        ...
        if (!btr_cur_optimistic_delete(btr_cur, 0, &mtr)) {
          /* The index entry could not be deleted. */
          success = false;
        }
      }
```

/* 【注】`BTR_DELETE` 是这个模式标志的字面意义：**允许把这次删除缓存进 change buffer**
   （`row_search_index_entry` 可能返回 `ROW_BUFFERED`）。所以乐观档"不改树"是指**不做 SMO**，
   不是"不改页"。乐观失败（`btr_cur_optimistic_delete` 返回 false）才升级到悲观档。 */

- **悲观**（`..._tree`，`BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE`，`row0purge.cc:341`）：`btr_cur_pessimistic_delete`，失败重试 `BTR_CUR_RETRY_DELETE_N_TIMES`。悲观档里 `ROW_NOT_FOUND` 是**合法**结局，注释解释了原因（`row0purge.cc:345-354`）：

```cpp
    case ROW_NOT_FOUND:
      /* Not found.  This is a legitimate condition.  In a
      rollback, InnoDB will remove secondary recs that would
      be purged anyway.  Then the actual purge will not find
      the secondary index record.  Also, the purge itself is
      eager: if it comes to consider a secondary index
      record, and notices it does not need to exist in the
      index, it will remove it.  Then if/when the purge
      comes to consider the secondary index record a second
      time, it will not exist any more in the index. */
```

`row_search_index_entry` 定位的判定：`PAGE_CUR_LE` 打开（R-tree 用 `PAGE_CUR_RTREE_LOCATE`），要求 `low_match == n_fields`（全字段匹配，含隐式 PK）。

**purge 会不会触发页分裂/合并**：**会合并、不会分裂**（purge 只删不插）。`btr_cur_pessimistic_delete` 末尾 `btr_cur_compress_if_useful`（`storage/innobase/btr/btr0cur.cc:4660-4702`）：

```cpp
/** Tries to compress a page of the tree if it seems useful. It is assumed
 that mtr holds an x-latch on the tree and on the cursor page. To avoid
 deadlocks, mtr must also own x-latches to brothers of page, if those
 brothers exist. NOTE: it is assumed that the caller has reserved enough
 free extents so that the compression will always succeed if done!
 @return true if compression occurred */
bool btr_cur_compress_if_useful(
    btr_cur_t *cursor, /*!< in/out: cursor on the page to compress;
                       cursor does not stay valid if !adjust and
                       compression occurs */
    bool adjust,       /*!< in: true if should adjust the
                        cursor position even if compression occurs */
    mtr_t *mtr)        /*!< in/out: mini-transaction */
{
  /* Avoid applying compression as we don't accept lot of page garbage
  given the workload of intrinsic table. */
  if (cursor->index->table->is_intrinsic()) {
    return false;
  }
  ...
  return (btr_cur_compress_recommendation(cursor, mtr) &&
          btr_compress(cursor, adjust, mtr));
}
```

判定阈值（`storage/innobase/include/btr0cur.ic:104-132` 与 `btr0cur.h:609-612`）：

```cpp
/** In the pessimistic delete, if the page data size drops below this
limit, merging it to a neighbor is tried */
#define BTR_CUR_PAGE_COMPRESS_LIMIT(index) \
  ((UNIV_PAGE_SIZE * (ulint)((index)->merge_threshold)) / 100)
```

```cpp
static inline bool btr_cur_compress_recommendation(
    btr_cur_t *cursor, /*!< btr cursor */
    mtr_t *mtr)        /*!< mtr */
{
  const page_t *page;
  ...
  if ((page_get_data_size(page) < BTR_CUR_PAGE_COMPRESS_LIMIT(cursor->index)) ||
      ((btr_page_get_next(page, mtr) == FIL_NULL) &&
       (btr_page_get_prev(page, mtr) == FIL_NULL))) {
    /* The page fillfactor has dropped below a predefined
    minimum value OR the level in the B-tree contains just
    one page: we recommend compression if this is not the
    root page. */

    return (dict_index_get_page(cursor->index) != page_get_page_no(page));
  }

  return false;
}
```

默认值（`storage/innobase/include/dict0mem.h:1059`）：

```cpp
constexpr uint32_t DICT_INDEX_MERGE_THRESHOLD_DEFAULT = 50;
```

/* 【注】★ `merge_threshold` 是**百分比**不是页数：`BTR_CUR_PAGE_COMPRESS_LIMIT` =
   `UNIV_PAGE_SIZE * merge_threshold / 100`，判定是 `page_get_data_size(page) < 该值`。
   默认 50 ⇒ 16 KB 页上数据 < 8 KB 才尝试与兄弟页合并；**值越大越激进**。
   另有一个 OR 条件：本层只剩一页（前后指针都是 FIL_NULL）时也建议压缩（但不能是 root 页）。 */

spatial 索引有额外保护：本页是最后一页且持有谓词页锁时**跳过** purge。

**purge 的多值索引分支**：多值索引的二级项是"一行 N 条"，purge 要删的是**旧数组的（全部或部分）元素**，与普通二级"一次一条"不同。分流（`row_purge_del_mark` / `row_purge_upd_exist_or_extern_func`）：

```cpp
if (node->index->is_multi_value()) {
  row_purge_remove_multi_sec_if_poss(node, heap, selected);   // selected 语义见下
} else {
  dtuple_t *entry = row_build_index_entry_low(node->row, ..., ROW_BUILD_FOR_PURGE);
  row_purge_remove_sec_if_poss(node, node->index, entry);     // 一次一条
}
```

`selected` 参数语义：**整行删除 → `false`**（旧数组每个元素都要删）；**UPDATE → `!non_mv_upd`**（`row_upd_changes_ord_field_binary` 判定"只有多值列本身变了"才差量删——`non_mv_upd` 在 purge 时对 undo 重建的 update 向量**重算**，非 undo 里存的）。

`row_purge_remove_multi_sec_if_poss`（`row0purge.cc:637-651`）本体很短：

```cpp
Multi_value_entry_builder_normal mv_entry_builder(node->row, nullptr, index,
                                                  heap, false, selected);
for (dtuple_t *entry = mv_entry_builder.begin(); entry != nullptr;
     entry = mv_entry_builder.next()) {
  row_purge_remove_sec_if_poss(node, index, entry);   // ★ 每个元素复用普通删除
}
```

builder 是 `Multi_value_entry_builder_normal`（`storage/innobase/include/row0row.h:441-514`，`begin()` / `next()` 在基类 `row0row.h:331-371`）。**完整原文见 [`../innodb/undo_log.md`](../innodb/undo_log.md)「多值索引的 undo 表示」节**，这里只留两个关键行：

```cpp
  /** Prepare the entry when the entry is not passed in */
  void prepare_entry_if_necessary() override {
    if (m_check) {
      m_entry = row_build_index_entry(m_row, m_ext, m_index, m_heap);
    } else {
      /* If not check, then it's basically coming from purge. And actually,
      for multi-value index, this flag really doesn't matter. */
      m_entry = row_build_index_entry_low(m_row, m_ext, m_index, m_heap,
                                          ROW_BUILD_FOR_PURGE);
    }
  }
```

```cpp
    const auto len = m_mv_data->data_len[m_pos];
    dfield_set_data(field, m_mv_data->datap[m_pos], len);

    ++m_pos;
    return (m_entry);
```

/* 【注】★ 整个循环只有**一个** `m_entry`，每次 `next()` 只是把该字段的 data 指针改成
   `m_mv_data->datap[m_pos]` 并重设长度——多值 purge 因此**复用**普通二级索引的删除路径。
   `m_selected` 为真时先用 `skip()` 跳过 bitset 里 bit=0（新旧数组共有、未变化）的元素。
   ★ undo 里写的是**完整数组**，bitset 只是追加其后的"跳过未变化元素"掩码，
   不是"只写变化元素"——见 `undo_log.md`。 */

- **entry 构造不是单独一套**：builder 内部走 `row_build_index_entry_low(..., ROW_BUILD_FOR_PURGE)`（与普通二级**同一个**构造器），多值字段的数据指针指向整个 `multi_value_data`，`next()` 每次只把该字段指针换成 `datap[m_pos]`——旧数组每个元素得到一个单值 entry；
- **每个元素的删除判定与普通二级完全同路**：`row_purge_remove_sec_if_poss`（leaf 乐观 `BTR_DELETE` → tree 悲观）→ `row_purge_poss_sec` 本身**无多值分支**（多值语义体现在 `row_vers_old_has_index_entry` 的虚拟列路径：`row_vers_find_matching` 沿 undo 链重建各版本 vrow 再比较）；
- **bitset 如何作用于 purge**：旧多值数组从 undo 恢复时**自带 bitset**（`trx_undo_rec_get_partial_row` → `Multi_value_logger::read` 读回，UPDATE 时随数组序列化进 undo）——`selected=true` 时 `Multi_value_entry_builder_normal::skip()` 跳过 bit=0（两数组共有、未变化）的元素，只删真正被移出的元素；
- **change buffer 无多值否决**：`ibuf_should_try` 的否决条件里**没有多值**；多值索引可以是 UNIQUE（mtr 测试有 `UNIQUE INDEX i1((CAST(... AS UNSIGNED ARRAY)))`）——非唯一多值索引的 insert/delete 都可进 ibuf，唯一多值索引仅 delete/delete-mark 可进（insert 被 unique 否决）。ibuf 记录本身支持多值字段类型（`ibuf_rec_has_multi_value` 逐字段查 `DATA_MULTI_VALUE` 位）。

### 唯一性检查为什么必须加锁

`row_ins_scan_sec_index_for_duplicate` 的核心顺序：**先加锁、后判重复**：

`storage/innobase/row/row0ins.cc:1872-2054`（去掉了 DD 表与 supremum 的旁支，主体逐字）：

```cpp
/** Scans a unique non-clustered index at a given index entry to determine
 whether a uniqueness violation has occurred for the key value of the entry.
 Set shared locks on possible duplicate records.
 @return DB_SUCCESS, DB_DUPLICATE_KEY, or DB_LOCK_WAIT */
[[nodiscard]] static dberr_t row_ins_scan_sec_index_for_duplicate(
    ulint flags,         /*!< in: undo logging and locking flags */
    dict_index_t *index, /*!< in: non-clustered unique index */
    dtuple_t *entry,     /*!< in: index entry */
    que_thr_t *thr,      /*!< in: query thread */
    bool s_latch,        /*!< in: whether index->lock is being held */
    mtr_t *mtr,          /*!< in/out: mini-transaction */
    mem_heap_t *offsets_heap)
/*!< in/out: memory heap that can be emptied */
{
  ulint n_unique;
  ulint n_fields_cmp;
  btr_pcur_t pcur;
  dberr_t err = DB_SUCCESS;
  ulint allow_duplicates;
  ulint *offsets = nullptr;
  DBUG_TRACE;

  ut_ad(s_latch ==
        rw_lock_own_flagged(&index->lock, RW_LOCK_FLAG_S | RW_LOCK_FLAG_SX));

  n_unique = dict_index_get_n_unique(index);

  /* If the secondary index is unique, but one of the fields in the
  n_unique first fields is NULL, a unique key violation cannot occur,
  since we define NULL != NULL in this case */

  if (!index->nulls_equal) {
    for (ulint i = 0; i < n_unique; i++) {
      if (UNIV_SQL_NULL == dfield_get_len(dtuple_get_nth_field(entry, i))) {
        return DB_SUCCESS;
      }
    }
  }

  /* Store old value on n_fields_cmp */

  n_fields_cmp = dtuple_get_n_fields_cmp(entry);

  dtuple_set_n_fields_cmp(entry, n_unique);

  pcur.open(index, 0, entry, PAGE_CUR_GE,
            s_latch ? BTR_SEARCH_LEAF | BTR_ALREADY_S_LATCHED : BTR_SEARCH_LEAF,
            mtr, UT_LOCATION_HERE);
  allow_duplicates = row_allow_duplicates(thr);

  const bool skip_gap_locks = index->table->skip_gap_locks();
  /* Scan index records and check if there is a duplicate */

  do {
    const rec_t *rec = pcur.get_rec();
    const buf_block_t *block = pcur.get_block();

    /* For DD tables, We don't use next-key locking for duplicates
    found. This means it is possible for another transaction to
    insert a duplicate key value but MDL protection on DD tables
    will prevent insertion of duplicates into unique secondary indexes*/
    ulint lock_type = skip_gap_locks ? LOCK_REC_NOT_GAP : LOCK_ORDINARY;

    if (page_rec_is_infimum(rec)) {
      continue;
    }

    offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED,
                              UT_LOCATION_HERE, &offsets_heap);

    const bool is_supremum = page_rec_is_supremum(rec);
    const bool is_next =
        !is_supremum && (cmp_dtuple_rec(entry, rec, index, offsets) < 0);
    if (flags & BTR_NO_LOCKING_FLAG) {
      /* Set no locks when applying log
      in online table rebuild. */
    } else if (allow_duplicates) {
      /* If the SQL-query will update or replace duplicate key we will take
      X-lock for duplicates ( REPLACE, LOAD DATAFILE REPLACE, INSERT ON
      DUPLICATE KEY UPDATE). */
      err = row_ins_set_rec_lock(LOCK_X, lock_type, block, rec, index, offsets,
                                 thr);
    } else {
      if (skip_gap_locks) {
        /* Skip GAP lock on supremum */
        if (is_supremum) {
          continue;
        }
        /* Skip gap lock on next record. */
        if (is_next) {
          goto end_scan;
        }
        lock_type = LOCK_REC_NOT_GAP;
      } else if (is_supremum) {
        /* We use next key lock to possibly combine the locks in bitmap.
        Equivalent to LOCK_GAP. */
        lock_type = LOCK_ORDINARY;
      } else if (is_next) {
        /* Only gap lock is required on next record. */
        lock_type = LOCK_GAP;
      } else {
        /* Next key lock for all equal keys. */
        lock_type = LOCK_ORDINARY;
      }
      /* We only need to lock locations where duplicates could occur, which
      means rows which are equal, gaps between them, and gaps on both
      sides of them. We don't need to lock the first unequal record after the
      gap, just the gap before it.
      Note: This function will not be even called, and thus will not lock
      anything at all in case there wasn't any matching record, which is fine,
      because the B-tree page latch will be released only after inserting the
      implicitly locked record, so no protection is needed.*/
      err = row_ins_set_rec_lock(LOCK_S, lock_type, block, rec, index, offsets,
                                 thr);
    }

    switch (err) {
      case DB_SUCCESS_LOCKED_REC:
        err = DB_SUCCESS;
      case DB_SUCCESS:
        break;
      default:
        goto end_scan;
    }

    if (is_supremum) {
      continue;
    }

    if (!is_next && !index->allow_duplicates) {
      if (row_ins_dupl_error_with_rec(rec, entry, index, offsets)) {
        err = DB_DUPLICATE_KEY;

        thr_get_trx(thr)->error_index = index;
        ...
        goto end_scan;
      }
    } else {
      ut_a(is_next || index->allow_duplicates);
      goto end_scan;
    }
  } while (pcur.move_to_next(mtr));

end_scan:
  /* Restore old value */
  dtuple_set_n_fields_cmp(entry, n_fields_cmp);

  return err;
}
```

/* 【注】① `dtuple_set_n_fields_cmp(entry, n_unique)` 把比较字段数临时改成**唯一键全字段**，
   循环结束再还原——判重必须比全部 n_unique 个字段；② `is_next`（`cmp_dtuple_rec < 0`，
   即定位到的记录比待插入项大）时判重直接跳过，因为已经越过了所有相等项；
   ③ 普通 INSERT 加的是 **LOCK_S**（共享），只有 `row_allow_duplicates(thr)`
   （REPLACE / INSERT ... ON DUPLICATE KEY UPDATE / LOAD DATA REPLACE）才加 LOCK_X。 */

`row_ins_dupl_error_with_rec`（`row0ins.cc:1822-1861`）的前提注释与 delete-mark 排除：

```cpp
/** Checks if a unique key violation to rec would occur at the index entry
 insert.
 @return true if error */
static bool row_ins_dupl_error_with_rec(
    const rec_t *rec,      /*!< in: user record; NOTE that we assume
                           that the caller already has a record lock on
                           the record! */
    const dtuple_t *entry, /*!< in: entry to insert */
    dict_index_t *index,   /*!< in: index */
    const ulint *offsets)  /*!< in: rec_get_offsets(rec, index) */
{
  ulint matched_fields;
  ulint n_unique;
  ulint i;

  ut_ad(rec_offs_validate(rec, index, offsets));

  n_unique = dict_index_get_n_unique(index);

  matched_fields = 0;

  entry->compare(rec, index, offsets, &matched_fields);

  if (matched_fields < n_unique) {
    return false;
  }

  /* In a unique secondary index we allow equal key values if they
  contain SQL NULLs */

  if (!index->is_clustered() && !index->nulls_equal) {
    for (i = 0; i < n_unique; i++) {
      if (dfield_is_null(dtuple_get_nth_field(entry, i))) {
        return false;
      }
    }
  }

  return rec_get_deleted_flag(rec, rec_offs_comp(offsets)) == 0;
}
```

/* 【注】最后一行就是 delete-mark 排除：**delete-marked 的记录不算唯一性冲突**。
   而它前面那句 "NOTE that we assume that the caller already has a record lock"
   正是"先加锁、后判重"的硬契约——因为这条判定依赖"没有别人正在改这条记录的删除状态"。 */

加锁出口 `row_ins_set_rec_lock()`（`row0ins.cc:1324-1353`）：

```cpp
static dberr_t row_ins_set_rec_lock(lock_mode mode, ulint type,
                                    const buf_block_t *block, const rec_t *rec,
                                    dict_index_t *index, const ulint *offsets,
                                    que_thr_t *thr) {
  dberr_t err;

  ut_ad(rec_offs_validate(rec, index, offsets));

  if (index->is_clustered()) {
    err = lock_clust_rec_read_check_and_lock(
        lock_duration_t::AT_LEAST_STATEMENT, block, rec, index, offsets,
        SELECT_ORDINARY, mode, type, thr);
  } else {
    err = lock_sec_rec_read_check_and_lock(lock_duration_t::AT_LEAST_STATEMENT,
                                           block, rec, index, offsets,
                                           SELECT_ORDINARY, mode, type, thr);
  }

  return (err);
}
```

/* 【注】`lock_duration_t::AT_LEAST_STATEMENT` 很关键：判重锁要活到**语句结束**，
   因为 INSERT 是两阶段的（先判重加锁，后真正插入），中间不能松手。
   配套机制在 `lock_rec_inherit_to_gap`（`lock0lock.cc:2614-2619`）：即使 RC 会跳过 gap 继承，
   只要 `trx->lock.inherit_all` 为真（由 AT_LEAST_STATEMENT 设置）就**仍然继承** gap 锁。 */

为什么不能靠 MVCC：

1. MVCC 只能判断"已存在的历史版本"是否可见，**无法阻止另一个活跃事务即将插入同值**——唯一性是"当前 + 未来"的保证；
2. **delete-mark 项必须先加锁再判断**：一条 delete-marked 的唯一索引项可能属于一个**尚未提交**的删除事务（回滚后它又活了）。若不加锁就判定"已删、可插入"，会产生幻象唯一性冲突；
3. 聚簇索引版注释补刀：unique non-clustered indexes 上可以存在**任意多条同 key 的 delete-marked 记录**（remember multiversioning），它们只差 row reference 部分——为避免竞态，必须**先插入、再检查**；
4. 判重加的是 `LOCK_ORDINARY`（next-key），**与隔离级别无关**（见下）。

#### 唯一性判重的 gap 行为由"是否 DD/SDI 表"决定，与隔离级别无关

`row_ins_scan_sec_index_for_duplicate` 里的 `skip_gap_locks` 取自 **`index->table->skip_gap_locks()`**（`row0ins.cc:1925`），这是**表级**标志，只对 **DD 表 / SDI 表**为真（`storage/innobase/include/dict0mem.h:2727-2729`）：

```cpp
  /* GAP locks are skipped for DD tables and SDI tables
  @return true if table is DD table or SDI table, else false */
  inline bool skip_gap_locks() const;
```

/* 【注】★ **与事务隔离级别无关**。事务级确实另有一个 `trx_t::skip_gap_locks()`
   （`storage/innobase/include/trx0trx.h:1138-1149`，RC/RU 返回 true）：

   `switch (isolation_level) { case READ_UNCOMMITTED: case READ_COMMITTED: return (true); ... }`

   但它在**这个判重函数里根本没被使用**。所以普通唯一二级索引在 RR **和 RC** 下
   对"相等记录"加的都是 `LOCK_ORDINARY`（next-key）。 */

这与官方手册一致：RC 下去掉 gap lock 只针对**普通加锁读 / UPDATE / DELETE**，而"外键约束检查与重复键检查"是**保留 gap lock 的两个例外**。

隔离级别降级真正出现的地方在别处：
- 外键检查：`row0ins.cc:1400-1404` → `skip_gap_lock = (trx->isolation_level <= TRX_ISO_READ_COMMITTED) || table->skip_gap_locks();`
- 聚簇索引判重：无条件 `LOCK_REC_NOT_GAP`（`row0ins.cc:2196-2198` / `2230-2232`）。

★ **正确表述**：RC/RR 下唯一性检查**都要加 record 锁**（普通 INSERT 是 `LOCK_S`，REPLACE/ODKU 是 `LOCK_X`）；差别**不在**这里的 gap 部分——判重的 gap 行为由"是否 DD/SDI 表"决定，而非隔离级别。

### 长事务与索引记录膨胀

机制链条：每条 `TRX_UNDO_DEL_MARK_REC` / 改索引列的 `TRX_UNDO_UPD_EXIST_REC` 在被 purge 前，对应的二级索引项**一直以 delete-marked 形式留在 B-tree 中**（占槽位与锁资源）；purge 推进受最老 ReadView 约束。

**后果**：长事务 / 大 history list ⇒ 大量 delete-marked 二级索引项堆积 ⇒ 二级索引页膨胀、扫描需跳过更多记录、`PAGE_GARBAGE` 升高（未到 `merge_threshold` 不会自动回收）。

相关的三个变量在 8.0.39 里的真实定义（脚本 grep 实证，均存在）：

```cpp
/* storage/innobase/srv/srv0srv.cc:512-516 */
/* The number of purge threads to use.*/
ulong srv_n_purge_threads = 4;
/* the number of pages to purge in one batch */
ulong srv_purge_batch_size = 20;

/* storage/innobase/srv/srv0srv.cc:580 */
bool srv_stats_include_delete_marked = false;
```

/* 【注】`innodb_purge_threads` 默认 **4**（上限 32，`ha_innodb.cc:22076`）；
   `innodb_purge_batch_size` 默认 **20**（范围 1~5000，`ha_innodb.cc:22069-22074`）；
   `innodb_stats_include_delete_marked` 默认 **off**（`ha_innodb.cc:21999-22003`），
   消费点在 `dict0stats.cc:929 / 1354`——它决定统计采样时是否把 delete-marked 记录算进去。 */

---

## 可观测性

| 想看什么 | 手段 |
|---|---|
| purge 是否滞后 | `SHOW ENGINE INNODB STATUS` 的 **history list length**（越大越滞后）；`trx_purge` 线程的 `Purge done for trx's N` |
| 最老 ReadView（水位） | 同上 TRANSACTIONS 段的最老活跃事务；`information_schema.innodb_trx` 找运行最久的事务 |
| delete-marked 堆积 | 无直接计数——间接：`PAGE_GARBAGE`（页内）、二级索引扫描变慢、`ibd` 文件不缩小 |
| purge 并发与批量 | `innodb_purge_threads`、`innodb_purge_batch_size` |
| 页回收阈值 | `index->merge_threshold` 默认 50（`ALTER TABLE ... COMMENT='MERGE_THRESHOLD=40'` 可调） |
| 统计是否含 delete-marked | `srv_stats_include_delete_marked` |
| 唯一性冲突 | `ER_DUP_ENTRY`（1062）；锁等待看 `SHOW ENGINE INNODB STATUS` 的 LOCK WAIT |
| 二级索引页粗筛字段 | `PAGE_MAX_TRX_ID`（`SHOW ENGINE INNODB STATUS` 的页信息 / `innodb_ruby` 等工具） |

★ **排查口诀**：删了大量数据但 `.ibd` 不缩小 → 先看 history list length（purge 是否滞后）；若 purge 已追平但仍不缩小 → 是**页内碎片**（未达 `merge_threshold`），需要 `OPTIMIZE TABLE` 重建（见 [`operations.md`](operations.md)）。

---

## Misc

### ★ 本机制里的工程实现技法

0. **权威单一化**：版本信息只存在于聚簇索引一处（`DB_TRX_ID` / `DB_ROLL_PTR`），二级索引只承担"位置"。这是一切复杂度（回表、两阶段删除、版本链回溯）的**源头**，也是"写路径简洁、索引不膨胀"的**代价**。
1. **页级粗筛**：用一个"只增不减"的页级上界（`PAGE_MAX_TRX_ID`）把"逐记录判断"降为"整页判断"——用粗粒度状态换掉高频精确计算。
2. ** lazy 删除 + 版本链回溯**：不维护引用计数（状态易失配），改为在低频路径（purge）上做 O(版本数) 的回溯——**用计算换状态正确性**。
3. **安全方向的失真**：`PAGE_MAX_TRX_ID` 不写 redo，崩溃后可能偏新——偏新只会多回表（慢），不会误判可见性（错）。这是"宁可慢、不可错"的取舍。
4. **删除顺序约束**：先所有二级索引、后聚簇——把"索引项指向已删聚簇记录"的不一致窗口压到最小。
5. **collation 比较而非二进制比较**（`dtuple_coll_eq`）：版本链回溯时必须用 collation 相等，否则 CHAR 列改写后会被误判为"不需要这条索引项"而误删。
6. **"三步插入"留下的可判定竞态**：插入是"聚簇 → 填 BLOB 指针 → 二级索引"，purge 撞上 `entry == nullptr` 就能断定"这行还在插入中"从而安全跳过——**不加锁也能消歧**，靠的是把写入顺序本身变成一种协议。
7. **游标/版本缓存**：`Row_sel_get_clust_rec_for_mysql` 的 `cached_clust_rec` / `cached_old_vers`，与 purge 侧 `row_purge_reposition_pcur` 的 `node->found_clust` + `pcur.store_position()`，都是"同一条 undo / 同一个 key 要处理 N 个索引"这一重复模式的产物——把 N 次搜索降为 1 次。
8. **锁的时长而不是强度**：判重锁用 `lock_duration_t::AT_LEAST_STATEMENT`，并靠 `trx->lock.inherit_all` 让它在 RC 下也**能**继承成 gap 锁——用"时长"表达"这是个跨阶段（先判重、后插入）的约束"，而不是去加强锁模式。

### ★ 一次 DELETE 的完整账（把三阶段串起来）

| 阶段 | 谁在做 | 聚簇记录 | 二级索引项 | 可见性 |
|---|---|---|---|---|
| ① `DELETE` 执行 | 用户事务 | 置 delete-mark，写 `TRX_UNDO_DEL_MARK_REC` | `btr_cur_del_mark_set_sec_rec` 置 delete-mark（**不是删除**） | 新事务看不见（两处标记都在） |
| ② purge 之前 | —— | 仍在页上，可能已被别的 UPDATE 推进版本链 | 仍在 B-tree 里占槽位与锁空间 | 老 ReadView 仍能看到旧版本 |
| ③ purge 判定时 | purge 线程 | `row_purge_reposition_pcur` 按主键重定位 | `row_purge_poss_sec`：聚簇不在 **或** 无存活版本需要 → 可删 | 已没有任何 view 能看见 |
| ④ 物理删除 | purge 线程 | `row_purge_remove_clust_if_poss` | 先所有二级（乐观 → 悲观），最后聚簇 | —— |

★ **为什么第 ④ 步必须"先二级后聚簇"**：反过来就会留下"索引项指向已删聚簇记录"的窗口，而这个窗口内的崩溃无法自愈（索引项没有独立信息可以证明自己该消失）。先删二级则相反——即使中途崩溃，剩下的聚簇记录是 delete-marked，下次 purge 会重试；**不一致窗口的方向被选成"可自愈"的那个**。

★ **长事务把这张卡在哪一步**：卡在 ② → ③ 之间。purge 的水位（`purge_sys->view`）被最老的 ReadView 钉住，`row_vers_old_has_index_entry` 就必须往回看更多版本，于是 delete-marked 的二级索引项**既不能删、又一直占着槽位**——这就是"删了几百万行但 `.ibd` 不缩小"的完整机制。

### 失效与退化场景

| 场景 | 表现 | 判据 / 处置 |
|---|---|---|
| **长事务阻塞 purge** | delete-marked 索引项堆积、二级索引膨胀、history list 增长 | 找最老活跃事务并结束它 |
| **purge 跟不上写入** | history list length 持续上升，`ibd` 增长 | 调 `innodb_purge_threads`（上限 32） |
| **页级粗筛失效** | 一个页被新事务改过 → `PAGE_MAX_TRX_ID` 变新 → **整页**记录都要回表 | 高频更新热点页会放大回表 |
| **ICP 误剪枝** | ICP 在二级索引上判的是**当前版本**，若该行有未提交的旧版本，ICP 结果可能被回表后的 old_vers 否决 | 只能跳过（保守方向正确），不会漏 |
| **页未回收** | delete-marked 已 purge，但页内填充率未低于 `merge_threshold`(50) → 页不释放 | `OPTIMIZE TABLE` 或调 `MERGE_THRESHOLD` |
| **唯一性锁等待/死锁** | 并发插入同值时 record S 锁排队；间隙锁在 RR 下更易死锁 | 见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md) |
| **spatial 索引跳过 purge** | 最后一页且持有谓词页锁时跳过（保护机制），可能残留 | `row0purge.cc` 的特殊分支 |
| **临时表** | `lock_sec_rec_cons_read_sees` 对临时表直接返回 true（临时表无 MVCC 跨会话需求） | 判据见源码 |

### 易混淆点

- **"删除数据后索引不缩小"的真相**：二级索引项在 purge 前**一直存在**（delete-mark 不是物理删除）；聚簇记录同理。
- **ICP 在二级索引上判断的是当前版本**——不匹配跳过是"保守剪枝"，匹配了也可能回表后被 old_vers 否决。
- **`row_vers_old_has_index_entry` 用 collation 相等而非二进制相等**——防止 CHAR 列改写但 collation 相等时误删索引项。
- **唯一性检查在 RC/RR 下都必须加锁**，差别只在 gap 锁部分（RC 跳过 gap 锁）。
- **purge 会触发页合并（`btr_cur_compress_if_useful`），不会触发分裂**。
- **purge 用 `node->ref`（主键值）重新定位，不用 undo 里的物理位置**——记录可能已被页分裂搬走。
- **聚簇索引与二级索引的删除语义不对称**：聚簇记录物理删除后其二级索引项还要逐个处理（先二级、后聚簇）。

### 三条交叉线的总图

```
        ┌─ 读：PAGE_MAX_TRX_ID 粗筛 →（不过）回表走聚簇版本链
MVCC ───┤
        └─ 写：删除 = delete-mark（两阶段的第一阶段）
                 ↑ 引用式的判定：row_vers_old_has_index_entry 回溯版本链

        ┌─ 定位：undo → node->ref（PK）→ 重新搜索定位（不用物理位置）
purge ──┤
        └─ 删除：row_purge_poss_sec 判定 → 乐观（可进 ibuf）/ 悲观（可能页合并）

        ┌─ 唯一性：先加锁（S/next-key）再判重——MVCC 管不了"未来"
约束 ───┴─ 外键等：见 constraint.md
```

---

## 参考

**官方文档**

- *MySQL 8.0 Reference Manual → 17.3 InnoDB Multi-Versioning*
- *MySQL 8.0 Reference Manual → 17.7.1 InnoDB Locking*（gap lock 在 RC 下的两个例外：foreign-key constraint checking 与 duplicate-key checking）
- *MySQL 8.0 Reference Manual → 15.8.10 Purge Configuration*（`innodb_purge_threads` / `innodb_purge_batch_size`）
- *MySQL 8.0 Reference Manual → 17.9.1.5 Configuring Merge Threshold for Index Pages*（`MERGE_THRESHOLD`）

> 源码不单列位置索引（按 `_template.md` 自检 ⑧：位置索引与行号必然漂移）。本篇源码已在「核心实现」各段**就地贴出**，检索时用函数名即可；涉及的主要文件是 `lock0lock.cc`、`row0sel.cc`、`row0vers.cc`、`row0purge.cc`、`row0ins.cc`、`btr0cur.cc`、`trx0undo.cc`。

**相关文档**

- MVCC/ReadView/可见性判定：[`../innodb/mvcc.md`](../innodb/mvcc.md)
- 行锁/gap/隐式锁：[`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- undo 与 purge 整体：[`../innodb/undo_log.md`](../innodb/undo_log.md)
- 唯一性检查完整链路（含外键）：[`constraint.md`](constraint.md)
- delete-mark 与页合并：[`btr.md`](btr.md)
- change buffer 与 purge 的配合（`BTR_DELETE` 模式）：[`ibuf.md`](ibuf.md)
- 取行主链 `row_search_mvcc`：[`../innodb/row_search.md`](../innodb/row_search.md)
- 索引重建与页回收：[`operations.md`](operations.md)

# MVCC / 一致性读 深度解析

> 基于 MySQL 8.0.39 源码。涵盖 ReadView 结构与池化、可见性判定、undo 版本链回溯、半一致性读、purge 水位闭环、隔离级别全景。**边界**：undo 记录格式与 purge 执行流程见 [`undo_log.md`](undo_log.md)；事务对象 `trx_t` 与提交协议见 [`trx.md`](trx.md)；行锁（锁定读的加锁本身）见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)；取行主链见 [`row_search.md`](row_search.md)。本篇讲"读到哪个版本"的判定与实现。

> 交叉：本篇的回溯算法（`row_vers_build_for_consistent_read` / `trx_undo_prev_version_build`）在**二级索引**上会退化成"页级粗筛 + 回表"，并在删除时引出两阶段 delete-mark——那一侧的整合见 [`../index/transaction.md`](../index/transaction.md)（同一批符号在那里会从**索引视角**再讲一遍，注意分工）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件（先认识快照是什么、由谁持有）
    - [ReadView：结构与可见性判定](#readview结构与可见性判定)
    - [ReadView 的池化：MVCC 类](#readview-的池化mvcc-类)
    - [server 层配合：select_lock_type 决策](#server-层配合select_lock_type-决策)
  - 读路径：一致性读
    - [可见性判断：聚簇与二级索引](#可见性判断聚簇与二级索引)
    - [版本链：undo 反向重放](#版本链undo-反向重放)
    - [隔离级别全景与 ReadView 生命周期](#隔离级别全景与-readview-生命周期)
  - 写路径：锁定读降级与清理
    - [半一致性读（semi-consistent read）](#半一致性读semi-consistent-read)
    - [purge 与 ReadView 的水位闭环](#purge-与-readview-的水位闭环)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 实现无锁一致性读的基础：undo 日志保存行的历史版本，读事务依据 ReadView 判断哪些版本可见，从而不加锁读到一致性快照。它由三件东西构成：

1. **ReadView**——一个 ~100 字节级的小对象（四个 limit + 一个有序活跃事务 id 数组），即"快照"的完整表达；
2. **版本链**——不是链表，而是 `DB_ROLL_PTR` 指向 undo 记录、由 undo 反向重放逐版回退的"逻辑链"；
3. **purge 水位**——全局最老 ReadView 的 `m_low_limit_no`，钉死 undo 可清理的边界。

semi-consistent read（半一致性读）是 MVCC 思想在 UPDATE/DELETE 锁定读上的变体优化。

### 用途

- **一致性读**：普通 SELECT 不加锁读到事务一致性快照，读写互不阻塞。
- **锁定读可见性**：`SELECT ... FOR SHARE/FOR UPDATE`、UPDATE、DELETE 基于最新版本；冲突检测靠隐式锁转换（版本链参与"考古"）。
- **半一致性读**：RC/RU 下 UPDATE 扫描遇到被锁行不等待，先读最新已提交版本让 SQL 层过滤。
- **purge 边界**：undo 能清理到哪里，由全局最老 ReadView 决定。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | semi-consistent read 由 `innodb_locks_unsafe_for_binlog` 控制 |
| 8.0 | 移除该变量（全库零引用，仅剩 mysql-test 的 include 文件），RC 语义吸收其职责（`skip_gap_locks()`/`releases_non_matching_rows()`）；AC-NL-RO 快路径（指针低位 tag 无锁复用）；GTID persistor 压低 purge 边界（`reduce_low_limit`）；`SKIP LOCKED`/`NOWAIT` 语法与半一致性读共用 `DB_SKIP_LOCKED` 返回值 |

---

## 理论基础

### 设计思想与权衡

#### 1. 快照读无锁的代价 = 版本链遍历

一致性读没有"物化快照"，也没有 view 索引——每次读到不可见版本，都要沿 undo **反向重放**到可见版本（纯 CPU + undo 页读）。正确性完全押注在**一个不变量**上：**任何活跃 ReadView 需要的 undo 一定还在**。该不变量由三条 FACT（`read0read.cc` 文件头，Oracle 原始注释）共同维护：

- **FACT A**：二级索引页头的 `PAGE_MAX_TRX_ID` 若小于 view 的 `m_up_limit_id`，该页所有二级索引记录必然可见（免回表的正确性证明）；
- **FACT B**：聚簇索引版本链上，事务边界外的版本归属事务必然已提交；
- **FACT C**：purge 克隆全局最老 ReadView，其 `low_limit_no` ≤ 任何现有/未来 view——因此"trx_no < purge 边界"的 undo 绝不会被任何 view 需要。

**★★ FACT C 的 Oracle 原文**（`read0read.cc:40-182`，这是全篇最该记住的一段注释）：

```cpp
FACT C: Purge does not remove any delete-marked row that is visible
-------
in any cursor read view.

PROOF: We know that:
 1: Currently active read views in trx_sys_t::view_list are ordered by
    ReadView::low_limit_no in descending order, that is,
    newest read view first.

 2: Purge clones the oldest read view and uses that to determine whether there
    are any active transactions that can see the to be purged records.

Therefore any joining or active transaction will not have a view older
than the purge view, according to 1.

When purge needs to remove a delete-marked row from a secondary index,
it will first check that the DB_TRX_ID value of the corresponding
record in the clustered index is older than the purge view. It will
also check if there is a newer version of the row (clustered index
record) that is not delete-marked in the secondary index. If such a
row exists and is collation-equal to the delete-marked secondary index
record then purge will not remove the secondary index record.

Delete-marked clustered index records will be removed by
row_purge_remove_clust_if_poss(), unless the clustered index record
(and its DB_ROLL_PTR) has been updated. Every new version of the
clustered index record will update DB_ROLL_PTR, pointing to a new UNDO
log entry that allows the old version to be reconstructed. The
DB_ROLL_PTR in the oldest remaining version in the old-version chain
may be pointing to garbage (an undo log record discarded by purge),
but it will never be dereferenced, because the purge view is older
than any active transaction.

For details see: row_vers_old_has_index_entry() and row_purge_poss_sec()

Some additional issues:

What if trx_sys->view_list == NULL and some transaction T1 and Purge both
try to open read_view at same time. Only one can acquire trx_sys->mutex.
In which order will the views be opened? Should it matter? If no, why?

The order does not matter. No new transactions can be created and no running
RW transaction can commit or rollback (or free views). AC-NL-RO transactions
will mark their views as closed but not actually free their views.
```

> ⚠️ 注释里的 `trx_sys_t::view_list` 在 8.0.39 **已不存在**（改名 `MVCC::m_views`），`row_sel_get_clust_rec()` 也改名了——**Oracle 注释未随代码更新**。
> ★★ FACT C 的"最老 view 在链表尾、按 `low_limit_no` 降序"这条不变量，代码化证明是 `ReadView::le()` + `MVCC::validate()`：
>
> ```cpp
> struct ViewCheck {
>   ViewCheck() : m_prev_view() {}
>   void operator()(const ReadView *view) {
>     ut_a(m_prev_view == nullptr || view->is_closed() || view->le(m_prev_view));
>     m_prev_view = view;
>   }
>   const ReadView *m_prev_view;
> };
>
> bool MVCC::validate() const {
>   ViewCheck check;
>   ut_ad(trx_sys_mutex_own());
>   ut_list_map(m_views, check);
>   return (true);
> }
> ```
>
> ```cpp
>   bool le(const ReadView *rhs) const {
>     return (m_low_limit_no <= rhs->m_low_limit_no);
>   }   // ★ 仅 UNIV_DEBUG
> ```

**★ FACT A 的关键句**（二级索引粗筛免回表的正确性证明）：

```cpp
When we access a secondary key record maximum transaction id is fetched
from this record and this trx_id is compared to up_limit_id in the view.
If trx_id in the record is greater or equal than up_limit_id in the view
cluster record is accessed.  Because trx_id of the creating
transaction is stored when this view was created to the list of
trx_ids not seen by this read view previous version of the
record is requested to be built. This is build using clustered record.
If the secondary key record is delete-marked, its corresponding
clustered record can be already be purged only if records
trx_id < low_limit_no. Purge can't remove any record deleted by a
transaction which was active when cursor was created. But, we still
may have a deleted secondary key record but no clustered record. But,
this is not a problem because this case is handled in
row_sel_get_clust_rec() function which is called
whenever we note that this read view does not see trx_id in the
record. Thus, we see correct version. Q. E. D.
```

> ★ **"if records trx_id < low_limit_no"** 这句是 FACT A 与 FACT C 的接缝：二级索引项被 delete-mark 后，其聚簇记录**只有在 `trx_id < low_limit_no`（purge 边界）时才可能已被 purge**——这正是"回表找不到聚簇记录"是**合法场景**的证明。

#### 2. 二级索引页级粗筛：用"必要条件单侧判定"换掉回表

二级索引记录**不存储** `DB_TRX_ID`/`DB_ROLL_PTR`（省空间），单条记录无法精确判可见。InnoDB 的解法是页级粗筛：`sees(PAGE_MAX_TRX_ID)` 为 true 必可见（直接用）；为 false 只是"不确定"（回表精判）。其失真是**安全方向**的——`PAGE_MAX_TRX_ID` 不写 redo，崩溃恢复后可能偏大，只会多回表不会错读。

#### 3. trx_id 与 trx_no：两个序列各管一件事

| 序列 | 分配时机 | 用途 |
|------|---------|------|
| `trx->id` | 事务开始 | 写入记录的 `DB_TRX_ID`，只服务 MVCC 可见性 |
| `trx->no` | 提交时入 serialisation_list（提交序） | 只服务 purge 顺序与 `m_low_limit_no` |

ReadView 用 **id** 判可见性、用 **no** 定 purge 边界——`m_low_limit_id`/`m_up_limit_id`（id 语义）与 `m_low_limit_no`（no 语义）是两组不同量纲的字段，命名极易混淆。

**★★ 8.0 的关键变化：id 与 no 共用同一个原子计数器**：

```cpp
inline trx_id_t trx_sys_allocate_trx_id_or_no() {
  ut_ad(trx_sys_mutex_own() || trx_sys_serialisation_mutex_own());

  trx_id_t trx_id = trx_sys->next_trx_id_or_no.fetch_add(1);
  //  ★★ id 与 no 从同一个计数器取！只是分配时持有的 mutex 不同

  if (trx_id % trx_sys_get_trx_id_write_margin() == 0) {
    trx_sys_write_max_trx_id();   // ★ 每 256(=TRX_SYS_TRX_ID_WRITE_MARGIN) 个值持久化一次
  }
  return trx_id;
}

inline trx_id_t trx_sys_allocate_trx_id() {
  ut_ad(trx_sys_mutex_own());                    // ★ trx->id：trx_sys_mutex
  return trx_sys_allocate_trx_id_or_no();
}

inline trx_id_t trx_sys_allocate_trx_no() {
  ut_ad(trx_sys_serialisation_mutex_own());      // ★ trx->no：serialisation_mutex
  return trx_sys_allocate_trx_id_or_no();
}
```

> ★★ 正因为共用计数器，`ReadView::prepare()` 里的 `ut_a(m_low_limit_no <= m_low_limit_id)` 才有意义。

**★ `trx->no` 的唯一分配点**（不是 `trx_write_serialisation_history()` 直接分配）：

```cpp
static inline bool trx_add_to_serialisation_list(trx_t *trx) {
  trx_sys_serialisation_mutex_enter();

  trx->no = trx_sys_allocate_trx_no();       // ★★ 唯一的 trx->no 分配语句

  ut_d(trx_sys->rw_max_trx_no = trx->no);

  if (trx->read_only) {
    trx_sys_serialisation_mutex_exit();
    return false;                            // 只读事务拿到 no 但不入 serialisation_list
  }

  UT_LIST_ADD_LAST(trx_sys->serialisation_list, trx);

  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) == 1) {
    trx_sys->serialisation_min_trx_no.store(trx->no);   // ★ 首个元素决定 min（purge 边界）
  }

  trx_sys_serialisation_mutex_exit();
  return true;
}
```

```cpp
static inline void trx_erase_from_serialisation_list_low(trx_t *trx) {
  ut_ad(trx_sys_serialisation_mutex_own());
  UT_LIST_REMOVE(trx_sys->serialisation_list, trx);

  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) > 0) {
    trx_sys->serialisation_min_trx_no.store(
        UT_LIST_GET_FIRST(trx_sys->serialisation_list)->no);
  } else {
    trx_sys->serialisation_min_trx_no.store(trx_sys_get_next_trx_id_or_no());
    //  ★ 空表时 min 回落到全局计数器
  }
}
```

> ★★ **只有有 update_undo 的事务才分配 `trx->no`**（`trx_write_serialisation_history` 里的判据 `trx->rsegs.m_redo.update_undo != nullptr || m_noredo.update_undo != nullptr`）。纯 insert / 只读事务的 `no` 保持 `TRX_ID_MAX`。
> ⚠️ **不存在 `trx_sys_get_new_trx_id()` / `trx_sys_get_new_trx_no()` / `trx_get_serialisation_max_trx_no()`**——真名是 `trx_sys_allocate_trx_id()` / `trx_sys_allocate_trx_no()` / `trx_sys_oldest_trx_no()`；"max" 侧只有 debug-only 的 `trx_sys->rw_max_trx_no`。

**★ 三个"活跃事务"集合的分工**（极易混淆）：

| 集合 | 类型 | 用途 | 匹配结构 |
|---|---|---|---|
| `rw_trx_ids` | `std::vector<trx_id_t>` 有序 | **ReadView::m_ids 的取值来源**（快照拷贝） | 二分 |
| `rw_trx_list` | `UT_LIST(trx_t)` | 隐式锁载体（移除即释放隐式锁） | — |
| `serialisation_list` | `UT_LIST(trx_t, no_list)` | 按 `trx->no` 排序，决定 `m_low_limit_no` | — |
| `shards[].active_rw_trxs` | `Trx_by_id_with_min`（分片） | `trx_rw_is_active()` 的无锁快判 | hash+min_id |

```cpp
  /** Array of Read write transaction IDs for MVCC snapshot. A ReadView would
  take a snapshot of these transactions whose changes are not visible to it.
  We should remove transactions from the list before committing in memory and
  releasing locks to ensure right order of removal and consistent snapshot. */
  trx_ids_t rw_trx_ids;
```

**★ `trx_rw_is_active()` 的无锁快路径**（这段代码有一个极长的内存序论证注释，是理解 8.0 分片设计的入口）：

```cpp
static inline trx_t *trx_rw_is_active(trx_id_t trx_id, bool do_ref_count) {
  /* Fast checking without Trx_shard::mutex to see if trx_id can't be in the
  Trx_shard::active_rw_trxs because it's too small. ... */
  ut_ad(trx_id < trx_sys_get_next_trx_id_or_no());
  auto &shard = trx_sys->get_shard_by_trx_id(trx_id);
  if (trx_id < shard.active_rw_trxs.peek().min_id()) {
    return nullptr;              // ★ 无锁快路径：比分片最小 id 还小 → 一定不活跃
  }
  return trx_sys->latch_and_execute_with_active_trx(
      trx_id,
      [&](trx_t *trx) {
        if (trx != nullptr && do_ref_count) {
          trx_reference(trx);    // ★ do_ref_count=true 时防 trx 被释放
        }
        return trx;
      },
      UT_LOCATION_HERE);
}
```

> ⚠️ 头文件注释明写关键 caveat：**"positive result means only that the trx was active at some moment during the call, but it might have already become `TRX_STATE_COMMITTED_IN_MEMORY` before the call returns"**——所以对 positive 结果必须再确认。
> ⚠️ **不存在 `trx_rw_is_active_low()`**（8.0 分片重构后已删除）。

#### 4. purge 被钉死在最老 ReadView 上

purge 每批克隆最老 view（`clone_oldest_view`），三处一致使用同一水位：fetch 的停止条件、history 截断上限、读 undo 前的 missing_history 判定。**只要存在一个长期不关的 ReadView，undo 就不能 purge**——这是长事务危害的源码级根源。

### 理论溯源

- Bernstein & Goodman《Concurrency Control in Distributed Database Systems》(ACM Computing Surveys 1981)：多版本并发控制的经典形式化；InnoDB 的可见性判断即快照隔离（Snapshot Isolation）的单机实现。
- Bayer/Schek 及 PostgreSQL 的多版本堆内存储路线：与 InnoDB 的 undo 回滚段路线相对（见"他库对比"）。
- 事务时间戳排序（timestamp ordering）：`trx_id` 单调递增分配正是"用时间戳代替锁"思想的延续。

### 算法与数据结构

- 版本回溯复杂度 O(链长)，每步一次 undo 页读 + 一次 `mem_heap_create(1024)`；purge 周期性清理控制链长。
- 可见性判定是**三层漏斗**：O(1) 快路径（低于低水位/高于高水位）→ O(1)（活跃集为空）→ O(log n) 二分（`m_ids`）。活跃事务越多，第三层越贵——长事务毒害的第二处体现。

### 他库对比

| | InnoDB | PostgreSQL | Oracle |
|---|---|---|---|
| 旧版本存放 | undo（独立表空间） | 堆表内（新版本另存一行，旧版本原地） | undo/回滚段 |
| 读旧版本方式 | 沿 undo 反向重放 | 扫描堆内多版本 + xmin/xmax 判定 | 类似 InnoDB（ORA-01555 对应 missing history） |
| 空间回收 | purge（受最老 ReadView 约束） | VACUUM（受最老事务 id 约束，有回卷风险） | undo retention |

InnoDB 的 `DB_MISSING_HISTORY`（历史被 purge）与 Oracle 的 `ORA-01555 snapshot too old` 是同一问题的两个实现。

---

## 核心实现

### ReadView：结构与可见性判定

#### ReadView 完整结构（read0types.h）

```cpp
class ReadView {
  /** ReadView 专用的有序数组（非 std::vector 替代品） */
  class ids_t {
    trx_id_t *m_ptr;      // 数组内存
    ulint m_size;         // 有效元素数
    ulint m_reserved;     // 容量（起步 MIN_TRX_IDS=32，扩容翻倍）
   public:
    void reserve(ulint n);
    void resize(ulint n) { ut_ad(n <= capacity()); m_size = n; }  // 只改 size 不缩容
    void insert(value_type value);   // 有序插入：back()<value 走 push_back 快路径
    ...
  };
 private:
  /** 高水位：trx_id >= 它的事务在快照后才开始，修改一定不可见 */
  trx_id_t m_low_limit_id;
  /** 低水位：trx_id < 它的事务在快照前已全部提交，修改一定可见 */
  trx_id_t m_up_limit_id;
  /** 创建者事务 id；free view 中为 TRX_ID_MAX */
  trx_id_t m_creator_trx_id;
  /** 创建时刻仍活跃的 RW 事务 id 集合，严格升序（供二分） */
  ids_t m_ids;
  /** ★ purge 边界（trx_no 量纲！）：小于它的 update undo 可被 purge */
  trx_id_t m_low_limit_no;
  /** AC-NL-RO 事务"逻辑关闭"标记（与 close() 是两套机制） */
  bool m_closed;
  byte pad1[64 - sizeof(node_t)];   // 填充到独立 cache line，防伪共享
  node_t m_view_list;               // trx_sys 中 view 链表的节点
};
```

逐字段要点：

- **两个 limit 的方向**：`m_up_limit_id`（低水位）= 创建时刻**最小**活跃事务 id（利用 `rw_trx_ids` 恒升序的不变量，`m_ids.front()` 即最小值，无需显式求 min）；`m_low_limit_id`（高水位）= 创建时刻 `trx_sys_get_next_trx_id_or_no()`（下一个待分配 id）。
- `m_low_limit_no` 是 **trx_no 量纲**：取 `trx_get_serialisation_min_trx_no()`（serialisation_list 最小 no）。DEBUG 字段 `m_view_low_limit_no` 保存被 GTID persistor 压低前的原值。
- `m_ids` 严格升序是 `changes_visible` 能二分的前提；`ids_t::reserve` 至少分配 32 个（`MIN_TRX_IDS`），为 view 复用减少反复分配。

#### 可见性判定核心：`changes_visible` 逐行

```cpp
[[nodiscard]] bool changes_visible(trx_id_t id, const table_name_t &name) const {
  ut_ad(id > 0);

  if (id < m_up_limit_id || id == m_creator_trx_id) {
    return (true);               // ① 低于低水位：快照前已提交；或就是自己改的
  }

  check_trx_id_sanity(id, name); // ② 防御：id 不应 >= 全局 next_trx_id

  if (id >= m_low_limit_id) {
    return (false);              // ③ 高于高水位：快照后才开始的 trx，必不可见
  } else if (m_ids.empty()) {
    return (true);               // ④ 区间内但活跃集为空 → 已全部提交
  }

  const ids_t::value_type *p = m_ids.data();
  return (!std::binary_search(p, p + m_ids.size(), id));  // ⑤ 在活跃集中 → 不可见
}
```

三层漏斗：①③④ 都是 O(1)；⑤ 是 O(log n)。**活跃事务越多 `m_ids` 越大**——每次可见性判定越贵，这是长事务的第二个毒害点。

#### 快照生成：`prepare` 逐行

```cpp
void ReadView::prepare(trx_id_t id) {
  ut_ad(trx_sys_mutex_own());                          // 必须持 trx_sys->mutex

  m_creator_trx_id = id;
  m_low_limit_no = trx_get_serialisation_min_trx_no(); // purge 边界 = 最老未收尾 trx_no
  m_low_limit_id = trx_sys_get_next_trx_id_or_no();    // 高水位 = 下一个待分配 id

  if (!trx_sys->rw_trx_ids.empty()) {
    copy_trx_ids(trx_sys->rw_trx_ids);                 // 拷贝活跃 RW id（排除自己）
  } else {
    m_ids.clear();
  }
  /* 第一个活跃事务 id 最小 */
  m_up_limit_id = !m_ids.empty() ? m_ids.front() : m_low_limit_id;
  m_closed = false;
}
```

`copy_trx_ids` 的细节：创建者 id 用 `lower_bound` 定位后**两次 memmove 跳过**（避免逐元素比较）；拷贝末尾顺带刷新 `m_up_limit_id = m_ids.front()`。

### ReadView 的池化：MVCC 类

#### 结构：双链表 + 启动预分配

```cpp
class MVCC {
 private:
  view_list_t m_free;    // 空闲 view（可复用）
  view_list_t m_views;   // 活跃 + 已关闭（按 m_low_limit_no 降序，新 view 插头）
};
// 系统启动时：trx_sys->mvcc = ut::new_withkey<MVCC>(..., 1024);  // 预分配 1024 个
```

**8.0.39 中不存在 `view_pool_t`**（5.7 的概念，本仓库无法验证细节）；池就是 `MVCC::m_free` + `m_views`。`m_views` 按创建序倒排（新 view 用 `UT_LIST_ADD_FIRST` 插头），debug 断言 `ReadView::le` 保证按 `m_low_limit_no` 有序——这让 `get_oldest_view()` 从尾向头扫、跳过 closed 即得最老。

#### `view_open`：fast path / slow path 逐行

```cpp
void MVCC::view_open(ReadView *&view, trx_t *trx) {
  if (view != nullptr) {
    uintptr_t p = reinterpret_cast<uintptr_t>(view);
    view = reinterpret_cast<ReadView *>(p & ~1);      // 清低位 tag，还原真指针
    ut_ad(view->m_closed);

    if (trx_is_autocommit_non_locking(trx) && view->empty()) {
      view->m_closed = false;                         // 无锁"重新激活"
      if (view->m_low_limit_id == trx_sys_get_next_trx_id_or_no()) {
        return;    // ★ FAST PATH：期间没有任何新 trx_id 分配 → 旧快照仍精确
      } else {
        view->m_closed = true;   // 有新事务 → 复用会破坏快照，回退 slow path
      }
    }
  }

  trx_sys_mutex_enter();                              // SLOW PATH
  if (view != nullptr) {
    UT_LIST_REMOVE(m_views, view);
  } else {
    view = get_view();                                // m_free 摘头 / new
  }
  view->prepare(trx->id);
  UT_LIST_ADD_FIRST(m_views, view);
  trx_sys_mutex_exit();
}
```

三层策略：

1. **Fast path（零锁）**：仅限 AC-NL-RO 事务（autocommit 单语句只读，`trx->id == 0`）且上次快照活跃集为空。不持 `trx_sys->mutex` 直接复用——判据是"期间没有任何新事务分配 id"（`m_low_limit_id` 未变），此时旧快照与"此刻新建"完全等价。
2. 源码注释明说**这里与 purge 存在固有竞态**（"There is an inherent race here between purge and this thread"）：缓解方式是"先置 closed 再检查、必要时回退"，purge 侧跳过 closed view，代价是 purge 可能暂时保守。
3. **Slow path**：持 `trx_sys->mutex` 重新 `prepare`。

#### `view_close` / `view_release`：指针低位 tag（dual representation）

```cpp
void MVCC::view_close(ReadView *&view, bool own_mutex) {
  if (!own_mutex) {
    // AC-NL-RO 无锁路径：只置 m_closed + 指针打 tag（p | 0x1），对象留在 m_views
    view->m_closed = true;
    view = reinterpret_cast<ReadView *>((uintptr_t(view)) | 0x1);
  } else {
    // RW 事务提交路径：完整归还 m_free
    view->close();                    // m_creator_trx_id = TRX_ID_MAX
    UT_LIST_REMOVE(m_views, view);
    UT_LIST_ADD_LAST(m_free, view);
    view = nullptr;
  }
}
```

- `trx_t::read_view` 指向已关闭 view 时**最低位被置 1**，`MVCC::is_view_active()` 以 `!(intptr_t(view) & 0x1)` 判活——"关闭一个 view"因此可以完全不持锁。
- `get_oldest_view()` 跳过 closed 的（tag 置位者对 purge 不可见）；`view_release`（AC-NL-RO view 的真正回收）要求持 `trx_sys->mutex` 且断言 `m_creator_trx_id == 0`。

### server 层配合：select_lock_type 决策

`row_prebuilt_t::select_lock_type`（`LOCK_NONE/LOCK_S/LOCK_X`）由 handler 两个钩子决定：

**`ha_innobase::store_lock`**（语句级，决策矩阵要点）：

```cpp
} else if ((lock_type == TL_READ && in_lock_tables) || ... ||
           lock_type == TL_READ_WITH_SHARED_LOCKS ||
           (lock_type != TL_IGNORE && sql_command != SQLCOM_SELECT)) {
  if (trx->skip_gap_locks() && (lock_type == TL_READ || TL_READ_NO_INSERT) &&
      (INSERT_SELECT / REPLACE_SELECT / UPDATE / CREATE_SELECT)) {
    m_prebuilt->select_lock_type = LOCK_NONE;   // ★ RC/RU 下 INSERT..SELECT 等的内层 SELECT 用一致性读替代锁定读
  } else {
    m_prebuilt->select_lock_type = LOCK_S;      // SELECT ... IN SHARE MODE / LOCK TABLES READ
  }
} else if (lock_type != TL_IGNORE) {
  m_prebuilt->select_lock_type = LOCK_NONE;     // 普通 SELECT：无锁
}
```

**`ha_innobase::external_lock`**（事务级强化）：

```cpp
if (lock_type == F_WRLCK) {
  m_prebuilt->select_lock_type = LOCK_X;        // 写游标：LOCK_X
} else if (lock_type == F_RDLCK) {
  if (trx->isolation_level == TRX_ISO_SERIALIZABLE &&
      m_prebuilt->select_lock_type == LOCK_NONE && 在事务中) {
    m_prebuilt->select_lock_type = LOCK_S;      // ★ SERIALIZABLE 把普通 SELECT 升级为锁定读
  }
}
```

`SKIP LOCKED/NOWAIT` 语法映射为 `prebuilt->select_mode = SELECT_SKIP_LOCKED/SELECT_NOWAIT`（高优先级事务强制 ORDINARY）。

---

### 可见性判断：聚簇与二级索引

#### 聚簇索引：`lock_clust_rec_cons_read_sees` 逐行

```cpp
bool lock_clust_rec_cons_read_sees(const rec_t *rec, dict_index_t *index,
                                   const ulint *offsets, ReadView *view) {
  ut_ad(index->is_clustered());
  if (srv_read_only_mode || index->table->is_temporary()) {
    return (true);          // 只读模式 / 临时表：免检（临时表跨连接不共享）
  }
  trx_id_t trx_id = row_get_rec_trx_id(rec, index, offsets);  // 读 DB_TRX_ID
  return (view->changes_visible(trx_id, index->table->name)); // 精确判定
}
```

聚簇记录自带 `DB_TRX_ID`/`DB_ROLL_PTR`，可精确判定；不可见时走版本链回溯（下文）。

#### 二级索引：`lock_sec_rec_cons_read_sees` 与页级粗筛

```cpp
bool lock_sec_rec_cons_read_sees(const rec_t *rec, const dict_index_t *index,
                                 const ReadView *view) {
  if (recv_recovery_is_on()) {
    return (false);            // 崩溃恢复期一律按"不可见"（保守，强制回表）
  } else if (index->table->is_temporary()) {
    return (true);
  }
  trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));  // 页头 PAGE_MAX_TRX_ID
  return (view->sees(max_trx_id));   // 只需 max_trx_id < m_up_limit_id
}
```

为什么页级粗筛成立（FACT A）：若 `PAGE_MAX_TRX_ID < m_up_limit_id`，则**曾经修改过本页任何记录的事务全部在快照前提交**——本页二级索引记录必然是"提交版本"，直接可用。返回 false 只是"不确定"（注释原文："the present version of rec may be the right"）。

**`PAGE_MAX_TRX_ID` 的更新点**（`page_update_max_trx_id`，仅在旧值更小时写入）：

| 调用点 | 场景 |
|---|---|
| `lock_sec_rec_modify_check_and_lock` | 二级索引记录加 X 锁成功后 |
| `btr0cur.cc` 二级乐观更新 / 悲观插入后 | 记录可能跨页时补写 |
| `row0ins` / `row0umod` / `row0log` | 向二级索引插入成功后 |
| `ibuf0ibuf` | change buffer 应用与 merge |
| 页创建 / 分裂 / 重整 | 从旧页拷贝 max trx id |
| bulk load / DDL / import | 建页时 |

**不写 redo**（`page_set_max_trx_id` 注释原文："during a database recovery we assume that the max trx id of every page is the maximum trx id assigned before the crash"）——恢复后值可能偏大，`sees()` 判 false → 多回表，**安全方向的失真**。

#### 不可信时回表：ICP 先过滤 + 二次复核

`row_search_mvcc` 的 `requires_clust_rec` 路径（`row0sel.cc`）：

1. `lock_sec_rec_cons_read_sees` 为 false → 先用 `row_search_idx_cond_check` 做 **ICP 过滤**（不匹配直接跳过，省一次回表）；
2. 需要时回表：用二级索引记录中的主键列在聚簇索引 `PAGE_CUR_LE` 定位；不可见则 `row_sel_build_prev_vers_for_mysql` 回溯版本；
3. **回表后必须复核** `row_sel_sec_rec_is_for_clust_rec`：旧版本的二级索引列值可能与 `rec` 不同（该二级索引 entry 在快照中根本不存在），按 `dtuple_coll_eq` 语义比较，不符则该行视为不存在。

#### 加锁读方向：隐式锁显式化

二级索引记录无显式锁时可能存在**隐式 X 锁**（由活跃事务最近写入，靠 `DB_TRX_ID` 表达）。加锁前必须显式化：`lock_rec_convert_impl_to_expl` → `lock_sec_rec_some_has_impl` → `row_vers_impl_x_locked_low`。其算法（源码注释给出形式化定义 Def 1~5）：从聚簇最新版本沿 undo 回退，在**同一事务的版本区间**内寻找"matches 翻转点"（`row_vers_find_matching`）；若该事务仍活跃即隐式锁持有者。可返回假阳性但绝不假阴性。

### 版本链：undo 反向重放

#### roll_ptr 的编码（trx0undo.ic）

```cpp
inline roll_ptr_t trx_undo_build_roll_ptr(bool is_insert, space_id_t space_id,
                                          page_no_t page_no, ulint offset) {
  return (roll_ptr_t)is_insert << 55 | (roll_ptr_t)id << 48 |
         (roll_ptr_t)page_no << 16 | offset;
  // is_insert(1bit) | undo 表空间号(7bit) | 页号(32bit) | 页内偏移(16bit)
}
```

7 字节 `DB_ROLL_PTR` 指向**产生记录当前版本的那次修改**所写的 undo 记录。undo 记录头部保存旧版本的 `DB_TRX_ID`/`DB_ROLL_PTR`/info_bits——**旧版本的 roll_ptr 又指向更早一次修改的 undo 记录**，这就是"链"：由 undo 内嵌 sys 列接力，而非显式链表指针。

#### 读 undo 前的双保险：`trx_undo_get_undo_rec`

```cpp
rw_lock_s_lock(&purge_sys->latch, ...);
missing_history = purge_sys->view.changes_visible(trx_id, name);
if (!missing_history) {
  *undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
}
rw_lock_s_unlock(&purge_sys->latch);
return (missing_history);   // true = 该 undo 已被 purge（历史缺失）
```

对 purge view 都"可见"的修改说明没有任何 view 还需要它——继续读会产生 use-after-purge。这是**读路径防 purge 竞态的第二道闸**（第一道是 purge 自身推进边界）。

#### undo 记录的物理布局与解析族（trx0rec.cc）

版本链的"原料"是 undo 记录字节，解析族统一风格：**每个函数消费一段字节、返回剩余指针**，下一函数从返回处继续。

**物理布局**（写入端 `trx_undo_page_set_next_prev_and_add`，trx0rec.cc:182）：

```
[2B: next 偏移][1B: type_cmpl][MODIFY_BLOB 时 1B 保留 0x00]
[undo_no: much-compressed][table_id: much-compressed]
[update 类: 1B info_bits + trx_id(旧) + roll_ptr(旧)]
[主键引用: n_unique × (len+data)]
[update vector: n_fields + n×(field_no + 旧值)]
[虚拟列区段: 2B 长度前缀 + ...][2B: 本条记录起始偏移(prev 指针)]
```

- 记录**尾部 2 字节**写 `first_free`（本条起始偏移 = prev 指针）、**头部 2 字节**写下一条偏移（next 指针）——undo 页内是双向链；
- `type_cmpl` 字节：位 0-3 = 记录类型（`TRX_UNDO_INSERT_REC=11`/`UPD_EXIST_REC=12`/`UPD_DEL_REC=13`/`DEL_MARK_REC=14`）、位 4-5 = cmpl_info、位 6 = `TRX_UNDO_MODIFY_BLOB`、位 7 = `TRX_UNDO_UPD_EXTERN`。

**解析族逐个**：

- `trx_undo_rec_get_pars`（556）：读 type/cmpl_info/undo_no/table_id；
- `trx_undo_update_rec_get_sys_cols`（1720）：读 **info_bits + trx_id + roll_ptr**——存的是**旧版本**系统列，重建时覆盖到新行副本上，旧版本由此获得指向更早版本的 roll_ptr，链自然延续；
- `trx_undo_rec_get_col_val`（643）：单列值三种编码——SQL NULL（field=nullptr）/ 外部存储列（`UNIV_EXTERN_STORAGE_FIELD` 标记 + orig_len + BLOB 前缀）/ 普通内联值；**返回的 field 直接指向 undo 缓冲区不拷贝**（调用方必须保证 undo 副本存活）；
- `trx_undo_rec_get_row_ref`（686）：读聚簇 n_unique 个主键列的旧值（行的"身份"，purge 回表用；MVCC 重建**不需要**，用 `trx_undo_rec_skip_row_ref` 跳过）；
- `trx_undo_update_rec_get_update`（1740）：解析 update vector——`upd_create(n_fields + 2)` 的 **+2 是 trx_id/roll_ptr 两个系统伪字段**；`new_val` 装的是**旧值**（before-image）；`TRX_UNDO_DEL_MARK_REC` 时 `n_fields=0`（del-mark 不改用户字段）；虚拟列 `field_no >= REC_MAX_N_FIELDS` 经 `trx_undo_read_v_idx` 映射（且会多存一份 after-image）；
- `trx_undo_rec_get_partial_row`（1941）：**部分行**（purge 用）——只含"在任何索引中作为排序列出现"的列，缺失列 `mtype == DATA_MISSING` 区分于 SQL NULL。

**★★ roll_ptr 的真实位布局**（⚠️ **不存在** `ROLL_PTR_INSERT_FLAG` / `ROLL_PTR_SPACE_ID_MASK` / `ROLL_PTR_PAGE_NO_MASK` / `ROLL_PTR_OFFSET_MASK` 任何命名常量——全仓 0 命中，位布局**硬编码在移位表达式里**）：

```cpp
inline roll_ptr_t trx_undo_build_roll_ptr(bool is_insert, space_id_t space_id,
                                          page_no_t page_no, ulint offset) {
  roll_ptr_t roll_ptr;
  ulint id;
  ut_ad(offset < 65536);

  id = (fsp_is_undo_tablespace(space_id) ? undo::id2num(space_id) : 0);
  //  ★★ 存的是 undo tablespace number，不是 rseg slot（形参仍叫 rseg_id）

  roll_ptr = (roll_ptr_t)is_insert << 55 | (roll_ptr_t)id << 48 |
             (roll_ptr_t)page_no << 16 | offset;
  return (roll_ptr);
}

inline void trx_undo_decode_roll_ptr(roll_ptr_t roll_ptr, bool *is_insert,
                                     ulint *rseg_id, page_no_t *page_no,
                                     ulint *offset) {
  ut_ad(roll_ptr < (1ULL << 56));            // ★★ 有效位宽 56 bit，不是 64
  *offset = (ulint)roll_ptr & 0xFFFF;        // bit 0-15
  roll_ptr >>= 16;
  *page_no = (ulint)roll_ptr & 0xFFFFFFFF;   // bit 16-47
  roll_ptr >>= 32;
  *rseg_id = (ulint)roll_ptr & 0x7F;         // ★★ bit 48-54 只有 7 bit（0x7F），不是 8 bit
  roll_ptr >>= 7;
  *is_insert = roll_ptr;                     // bit 55
}
```

| bit | 宽度 | 含义 |
|---|---|---|
| 55 | 1 | `is_insert`（1 = insert undo） |
| 48-54 | **7** | undo tablespace number（非 undo 表空间为 0） |
| 16-47 | 32 | `page_no` |
| 0-15 | 16 | 页内 `offset` |

```cpp
static inline bool trx_undo_roll_ptr_is_insert(roll_ptr_t roll_ptr) {
  ut_ad(roll_ptr < (1ULL << 56));
  return roll_ptr >> 55;
}
static inline bool trx_undo_trx_id_is_insert(const byte *trx_id) {
  return (static_cast<bool>(trx_id[DATA_TRX_ID_LEN] >> 7));
  //  ★ 直接读紧随 DB_TRX_ID(6B) 之后 roll_ptr 首字节的最高位 = bit 55（7 字节大端）
}
```

> ★ 读写原语：`trx_write_roll_ptr()` = `mach_write_to_7()`，`trx_read_roll_ptr()` = `mach_read_from_7()`。

**★ undo 页内是双向链，`next` 与 `prev` 写在同一个 2 字节槽位**：

```cpp
static ulint trx_undo_page_set_next_prev_and_add(page_t *undo_page, byte *ptr,
                                                 mtr_t *mtr) {
  ulint first_free;
  ulint end_of_rec;
  byte *ptr_to_first_free;

  if (UNIV_UNLIKELY(trx_undo_left(undo_page, ptr) < 2)) {
    return (0);                                    // ★ 至少留 2 字节给"下一条指针"
  }
  ptr_to_first_free = undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE;
  first_free = mach_read_from_2(ptr_to_first_free);

  /* Write offset of the previous undo log record */
  mach_write_to_2(ptr, first_free);                // ★ rec 尾部 2B = prev
  ptr += 2;
  end_of_rec = ptr - undo_page;

  /* Write offset of the next undo log record */
  mach_write_to_2(undo_page + first_free, end_of_rec);   // ★ rec 头部 2B = next
  /* Update the offset to first free undo record */
  mach_write_to_2(ptr_to_first_free, end_of_rec);

  trx_undof_page_add_undo_rec_log(undo_page, first_free, end_of_rec, mtr);
  return (first_free);                             // ★ 返回值打进 roll_ptr 的 bit 0-15
}
```

```cpp
  return (undo_page + mach_read_from_2(rec - 2));  // trx_undo_page_get_prev_rec：往前 2 字节
  next = mach_read_from_2(rec);                    // trx_undo_page_get_next_rec：往后 2 字节
  if (next == end) return (nullptr);               // 本页最后一条
```

> ★ 布局：`[next:2B][type_cmpl:1B][...正文...][prev:2B]`。上一条记录的尾部 2B 与下一条记录的头部 2B 是**同一处内存被写了两次**。

**★ 解析族的两个关键函数**（版本链的"接缝"就在这里）：

```cpp
byte *trx_undo_update_rec_get_sys_cols(const byte *ptr, trx_id_t *trx_id,
                                       roll_ptr_t *roll_ptr, ulint *info_bits) {
  /* Read the state of the info bits */
  *info_bits = mach_read_from_1(ptr);          // ★ 1 字节 info_bits（含 REC_INFO_DELETED_FLAG）
  ptr += 1;
  /* Read the values of the system columns */
  *trx_id = mach_u64_read_next_compressed(&ptr);      // ★ 旧版本的 DB_TRX_ID
  *roll_ptr = mach_u64_read_next_compressed(&ptr);    // ★★ 旧版本的 DB_ROLL_PTR —— 链头指针
  return (const_cast<byte *>(ptr));
}
```

```cpp
  if (type != TRX_UNDO_DEL_MARK_REC) {
    n_fields = mach_read_next_compressed(&ptr);
  } else {
    n_fields = 0;      // ★ DEL_MARK 不记录任何被更新列的旧值（delete-mark 不改字段）
  }
  update = upd_create(n_fields + 2, heap);      // ★★ +2 = DB_TRX_ID + DB_ROLL_PTR 两个伪字段
  update->info_bits = info_bits;
  upd_field = upd_get_nth_field(update, n_fields);
  trx_write_trx_id(buf, trx_id);                 // ★ 把"旧版本 trx_id"作为 new_val
  upd_field_set_field_no(upd_field, index->get_sys_col_pos(DATA_TRX_ID), index);
  ...
  trx_write_roll_ptr(buf, roll_ptr);             // ★ 把"旧版本 roll_ptr"作为 new_val
  ...
  field_no = mach_read_next_compressed(&ptr);
  is_virtual = (field_no >= REC_MAX_N_FIELDS);   // ★ 虚拟列用 field_no + REC_MAX_N_FIELDS 编码
  if (is_virtual) {
    ptr = trx_undo_read_v_idx(index->table, ptr, first_v_col, &is_undo_log, &field_no);
    first_v_col = false;
  }
```

> ★★ 这就是"版本链"的实现：**旧版本的 `DB_ROLL_PTR` 被装进 `upd_t` 的 `new_val`，应用 update 时写回记录——于是新造出的 old_vers 自然指向更早一次修改的 undo**。没有显式链表指针。

```cpp
byte *trx_undo_rec_get_col_val(const byte *ptr, const byte **field, ulint *len,
                               ulint *orig_len) {
  *len = mach_read_next_compressed(&ptr);
  *orig_len = 0;
  switch (*len) {
    case UNIV_SQL_NULL:                          // ① SQL NULL
      *field = nullptr;
      break;
    case UNIV_EXTERN_STORAGE_FIELD:              // ② 外存列哨兵
      *orig_len = mach_read_next_compressed(&ptr);   // 本地前缀长度
      *len = mach_read_next_compressed(&ptr);        // 总长度
      *field = ptr;
      ptr += *len & ~SPATIAL_STATUS_MASK;            // ★ 高位借给 spatial status
      *len += UNIV_EXTERN_STORAGE_FIELD;             // ★ 加回哨兵作为标记
      break;
    default:                                     // ③ 普通内联值
      *field = ptr;                                  // ★★ 不拷贝！直接指向 undo 缓冲区
      ...
  }
  return (const_cast<byte *>(ptr));
}
```

> ⚠️ **返回的 field 直接指向 undo 缓冲区不拷贝**——调用方必须保证 undo 副本存活（这正是 `trx_undo_get_undo_rec_low()` 要 `trx_undo_rec_copy()` 到 heap 的原因）。

#### 版本回退一步：`trx_undo_prev_version_build`（trx0rec.cc）

```cpp
bool trx_undo_prev_version_build(const rec_t *index_rec, const mtr_t *index_mtr,
                                 const rec_t *rec, const dict_index_t *index,
                                 ulint *offsets, mem_heap_t *heap,
                                 rec_t **old_vers, mem_heap_t *v_heap,
                                 const dtuple_t **vrow, ulint v_status,
                                 lob::undo_vers_t *lob_undo) {
  roll_ptr = row_get_rec_roll_ptr(rec, index, offsets);
  *old_vers = nullptr;

  if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
    return true;          // ★ 本版本是 INSERT 产生：没有更旧版本（old_vers 保持 NULL）
  }

  if (trx_undo_get_undo_rec(roll_ptr, rec_trx_id, heap, is_temp, name, &undo_rec)) {
    if (v_status & TRX_UNDO_PREV_IN_PURGE) {
      undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);  // purge 自己调用：强制读
    } else {
      return false;       // ★ missing history：上层报 DB_MISSING_HISTORY
    }
  }

  ...解析 undo 头：type/cmpl_info/table_id...
  ptr = trx_undo_update_rec_get_sys_cols(ptr, &trx_id, &roll_ptr, &info_bits);
  // ★ 恢复"上一个版本"的 DB_TRX_ID / DB_ROLL_PTR / info_bits(delete-mark)

  ptr = trx_undo_update_rec_get_update(ptr, index, type, trx_id, roll_ptr,
                                       info_bits, heap, &update, lob_undo, type_cmpl);

  if (row_upd_changes_field_size_or_external(index, offsets, update)) {
    // 字段长度/外部存储变化：整行重建（dtuple → rec）
    entry = row_rec_to_index_entry(rec, index, offsets, heap);
    row_upd_index_replace_new_col_vals(entry, index, update, heap);
    *old_vers = rec_convert_dtuple_to_rec(buf, index, entry);
  } else {
    // ★ 快路径：原地拷贝 + 应用反向更新
    *old_vers = rec_copy(buf, rec, offsets);
    row_upd_rec_in_place(*old_vers, index, offsets, update, nullptr);
  }
  ...
}
```

**返回值三态**（务必区分）：

| 返回 | `*old_vers` | 语义 |
|------|------------|------|
| `true` | 非空 | 成功构建前一版本 |
| `true` | `nullptr` | 版本链到头（fresh insert）——该行在此快照中不存在 |
| `false` | — | **missing history**：undo 已被 purge（`DB_MISSING_HISTORY`） |

undo 类型常量：`TRX_UNDO_INSERT_REC=11`、`TRX_UNDO_UPD_EXIST_REC=12`、`TRX_UNDO_UPD_DEL_REC=13`（把 delete-mark 记录更新回正常）、`TRX_UNDO_DEL_MARK_REC=14`（只记 sys 列不记字段值）。

#### 一致性读的完整回溯：`row_vers_build_for_consistent_read` 逐行

```cpp
dberr_t row_vers_build_for_consistent_read(const rec_t *rec, mtr_t *mtr,
                                           dict_index_t *index, ulint **offsets,
                                           ReadView *view, mem_heap_t **offset_heap,
                                           mem_heap_t *in_heap, rec_t **old_vers,
                                           const dtuple_t **vrow, lob::undo_vers_t *lob_undo) {
  trx_id = row_get_rec_trx_id(rec, index, *offsets);
  ut_ad(!view->changes_visible(trx_id, index->table->name)); // 前置：当前版本必须不可见

  version = rec;
  for (;;) {
    heap = mem_heap_create(1024, ...);            // 每步新 heap，释放旧版本

    bool purge_sees = trx_undo_prev_version_build(rec, mtr, version, index, *offsets,
                                                  heap, &prev_version, ...);
    err = (purge_sees) ? DB_SUCCESS : DB_MISSING_HISTORY;   // undo 被清 → 报缺失

    if (prev_version == nullptr) {
      *old_vers = nullptr;      // fresh insert：行在快照中不存在
      break;
    }
    trx_id = row_get_rec_trx_id(prev_version, index, *offsets);
    if (view->changes_visible(trx_id, index->table->name)) {
      // 找到可见版本：拷贝进调用方的 in_heap（跨 mtr 存活）
      *old_vers = rec_copy(buf, prev_version, *offsets);
      break;
    }
    version = prev_version;     // 仍不可见 → 继续回溯
  }
  return err;
}
```

- **终止性**：`m_up_limit_id` 之下的事务都可见，版本链最底层要么是 insert（NULL）要么是已提交版本——除非 purge 破坏不变量（返回 `DB_MISSING_HISTORY`）。这正是 FACT C 的意义。
- **delete-mark 不在此特判**：可见版本若带 delete-mark（已提交的 delete），由调用方 `row_search_mvcc` 统一丢弃（`rec_get_deleted_flag` → `next_rec`）。

#### purge 方向的两个配套函数

| 函数 | 调用方 | 作用 |
|---|---|---|
| `row_vers_must_preserve_del_marked` | purge | delete-mark 记录的创建事务对 purge view 仍不可见 → 还需保留 |
| `row_vers_old_has_index_entry` | `row_purge_poss_sec` | purge 删二级索引 entry 前确认：逐版回退对每个未 delete-mark 版本构建其二级索引 entry（**按 collation 语义比较**，非二进制），有匹配则不能删 |
| `row_vers_impl_x_locked_low` | 隐式锁转换 | 见上文"隐式锁显式化" |
| `row_vers_build_for_semi_consistent_read` | 半一致性读 | 见「半一致性读」节 |

**★ `row_vers_must_preserve_del_marked()` 只有 5 行**（`row0vers.cc:592`）：

```cpp
bool row_vers_must_preserve_del_marked(trx_id_t trx_id, const table_name_t &name,
                                       mtr_t *mtr) {
  ut_ad(!rw_lock_own(&(purge_sys->latch), RW_LOCK_S));

  mtr_s_lock(&purge_sys->latch, mtr, UT_LOCATION_HERE);
  //  ★★ purge_sys->latch 由 mtr 接管，mtr_commit 时自动释放 —— 与手动 s_lock 的另两处不同

  return (!purge_sys->view.changes_visible(trx_id, name));
  //  ★ purge view 都还看不到这个 trx_id ⇒ 该 delete-marked 版本必须保留
}
```

> ★★ **`purge_sys->latch` 的三个使用点各不相同**：`trx0rec.cc:2447`（手动 `rw_lock_s_lock`）、`trx0rec.cc:2591`（手动，disowned-external）、`row0vers.cc:596`（`mtr_s_lock`，由 mtr 代管释放）。三个入口都断言"调用前未持有 S 锁"。

### purge 与 ReadView 的水位闭环

**★ `purge_sys->view` 是内嵌对象，不是指针**（`trx0purge.h:985`）：

```cpp
struct trx_purge_t {
  /** The latch protecting the purge view. A purge operation must acquire an
  x-latch here for the instant at which it changes the purge view: an undo
  log operation can prevent this by obtaining an s-latch here. */
  rw_lock_t latch;

  /** The purge will not remove undo logs which are >= this view (purge view) */
  ReadView view;          // ★★ 内嵌 ReadView 对象

  /** true if view is active */
  bool view_active;

  /** Limit up to which we have read and parsed the UNDO log records. Not
  necessarily purged from the indexes. */
  purge_iter_t iter;      // ★ 已"读+解析"到的位置

  /** The 'purge pointer' which advances during a purge, and which is used in
  history list truncation */
  purge_iter_t limit;     // ★ 已"确认 purge 完成"的位置
#ifdef UNIV_DEBUG
  purge_iter_t done;      // ★ 仅 debug：Innodb_purge_trx_id_age 的数据源
#endif
  bool next_stored;       // ⚠️ bool，不是 purge_iter_t
  purge_pq_t *purge_queue;   // ★ 最小堆（按 trx_no），取代老版 trx_purge_get_rseg_with_min_trx_no()
  ...
};
```

```cpp
struct purge_iter_t {
  /** Purge has advanced past all transactions whose number is less than this */
  trx_id_t trx_no;
  /** Purge has advanced past all records whose undo number is less than this. */
  undo_no_t undo_no;
  /** The last undo record resided in this space id */
  space_id_t undo_rseg_space;
  /** The transaction that created the undo log record, the Modifier trx id */
  trx_id_t modifier_trx_id;   // ★ 产生该 undo rec 的 trx id（不是 trx_no）
};
```

**★★ 水位刷新点：每个 purge batch 的开头**（`trx0purge.cc:2490`）：

```cpp
  rw_lock_x_lock(&purge_sys->latch, UT_LOCATION_HERE);   // ★ x-latch 保护"改 view"这一瞬间

  purge_sys->view_active = false;

  trx_sys->mvcc->clone_oldest_view(&purge_sys->view);   // ★★ 每批次重取最老 view

  purge_sys->view_active = true;

  rw_lock_x_unlock(&purge_sys->latch);
```

```cpp
void MVCC::clone_oldest_view(ReadView *view) {
  trx_sys_mutex_enter();
  ReadView *oldest_view = get_oldest_view();

  if (oldest_view == nullptr) {
    view->prepare(0);              // ★ 无任何活跃 view → 造"空 view"，水位取 0（可 purge 一切）
    trx_sys_mutex_exit();
  } else {
    view->copy_prepare(*oldest_view);   // ★ 锁内拷贝
    trx_sys_mutex_exit();
    view->copy_complete();              // ★ 锁外补全
  }

  /* Update view to block purging transaction till GTID is persisted. */
  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  auto gtid_oldest_trxno = gtid_persistor.get_oldest_trx_no();
  view->reduce_low_limit(gtid_oldest_trxno);   // ★★ GTID 未落盘 → 再压低水位
}
```

```cpp
ReadView *MVCC::get_oldest_view() const {
  ut_ad(trx_sys_mutex_own());
  for (view = UT_LIST_GET_LAST(m_views); view != nullptr;
       view = UT_LIST_GET_PREV(m_view_list, view)) {   // ★ 从链表尾（最老）往前扫
    if (!view->is_closed()) break;                     // ★ 跳过 closed（AC-NL-RO 的）
  }
  return (view);
}
```

> ⚠️ **`m_views` 是"最新 view 在头"**（`view_open` 用 `UT_LIST_ADD_FIRST`）——所以 `get_oldest_view()` 从尾找。文档里常写反。

**★ `copy_prepare` / `copy_complete` 为什么要拆成两步**：

```cpp
void ReadView::copy_prepare(const ReadView &other) {
  if (!other.m_ids.empty()) {
    m_ids.assign(p, p + other.m_ids.size());
  } else { m_ids.clear(); }
  m_up_limit_id = other.m_up_limit_id;
  m_low_limit_no = other.m_low_limit_no;     // ★★ purge 水位的真正载体
  m_low_limit_id = other.m_low_limit_id;
  m_creator_trx_id = other.m_creator_trx_id;
}

void ReadView::copy_complete() {
  ut_ad(!trx_sys_mutex_own());               // ★★ 必须在释放 trx_sys_mutex 之后
  if (m_creator_trx_id > 0) {
    m_ids.insert(m_creator_trx_id);          // ★ purge view 里 creator 也算"不可见"
  }
  if (!m_ids.empty()) {
    m_up_limit_id = std::min(m_ids.front(), m_up_limit_id);
  }
  m_creator_trx_id = 0;
}
```

> ★ `copy_complete()` 里的 `m_ids.insert()` 本身需要 `reserve`，可能分配内存——**所以必须在锁外做**。这是拆两步的原因。

**★★ 水位的三个消费点**（这就是"三处一致使用同一水位"的源码证据）：

```cpp
// ① fetch 的停止条件 —— trx0purge.cc:2286
  if (purge_sys->iter.trx_no >= purge_sys->view.low_limit_no()) {
    return nullptr;                          // ★★ 到此为止不再取 undo rec
  }
```

```cpp
// ② history 截断上限 —— trx0purge.cc:1638
  if (limit->trx_no >= view->low_limit_no()) {
    limit->trx_no = view->low_limit_no();    // ★★ 截断上限兜底：绝不超过 purge view 水位
    limit->undo_no = 0;
    limit->undo_rseg_space = SPACE_UNKNOWN;
  }
  ut_ad(limit->trx_no <= purge_sys->view.low_limit_no());
```

```cpp
// ③ 读 undo 前的 missing_history 判定 —— trx0rec.cc:2452
  missing_history = purge_sys->view.changes_visible(trx_id, name);
```

```cpp
// 另外两处 debug 断言：trx0purge.cc:1652 / 1944
  ut_ad(limit->trx_no <= purge_sys->view.low_limit_no());
  ut_ad(purge_sys->iter.trx_no < purge_sys->view.low_limit_no());
```

**★ `iter` 与 `limit` 的生产者-消费者关系**（`trx_purge_attach_undo_recs`）：

```cpp
  purge_sys->limit = purge_sys->iter;      // ★ 上一批已清理完毕 → limit 追平 iter

  while (n_pages_handled < batch_size) {
    if (trx_purge_check_limit()) {
      purge_sys->limit = purge_sys->iter;  // ★ limit 单调追到 iter
    }
    rec.undo_rec = trx_purge_fetch_next_rec(&rec.modifier_trx_id, &rec.roll_ptr,
                                            &n_pages_handled, heap);
    if (rec.undo_rec == &trx_purge_ignore_rec) continue;
    else if (rec.undo_rec == nullptr) break;   // ★ 撞到水位 → 结束本批
    purge_groups.add(rec);
  }
```

```cpp
static void trx_purge_truncate(void) {
  ut_ad(trx_purge_check_limit());
  if (purge_sys->limit.trx_no == 0) {
    trx_purge_truncate_history(&purge_sys->iter, &purge_sys->view);
  } else {
    trx_purge_truncate_history(&purge_sys->limit, &purge_sys->view);
  }
  trx_purge_truncate_undo_spaces();
}
```

```cpp
static inline bool trx_purge_check_limit(void) {
  return (purge_sys->iter.trx_no > purge_sys->limit.trx_no ||
          (purge_sys->iter.trx_no == purge_sys->limit.trx_no &&
           ((purge_sys->iter.undo_no >= purge_sys->limit.undo_no) ||
            (purge_sys->iter.undo_rseg_space != purge_sys->limit.undo_rseg_space))));
  //  ★★ 不变式：limit <= iter（同一 trx_no 用多个 rseg 时还要看 space_id）
}
```

> ⚠️ **`purge_sys->view_active` 在 8.0.39 只有赋值、无任何读取点**（`:293`、`:2492`、`:2497`）。真正的互斥保护是 `purge_sys->latch` 的 x-latch。
> ⚠️ **不存在 `trx_purge_get_rseg_with_min_trx_no()`**（5.6/5.7 老接口）——改用 `purge_sys->purge_queue` 最小堆 + `TrxUndoRsegsIterator::set_next()`。
> ⚠️ **`srv_purge_wakeup` / `srv_do_purge` / `srv_purge_coordinator_thread` 完全不读 `purge_sys->view`**——它们只看 `trx_sys->rseg_history_len`。

**★ `reduce_low_limit()`：GTID 压低水位的机制**：

```cpp
  void reduce_low_limit(trx_id_t trx_no) {
    if (trx_no < m_low_limit_no) {
      /* Save low limit number set for Read View for MVCC. */
      ut_d(m_view_low_limit_no = m_low_limit_no);
      m_low_limit_no = trx_no;      // ★★ purge 水位被 GTID 持久化进度"压低"
    }
  }
```

> ★ `m_view_low_limit_no` 是 **UNIV_DEBUG 专属**，保存压低前的原值——它是 `Innodb_purge_view_trx_id_age` 的数据源。

### 隔离级别全景与 ReadView 生命周期

**★ RC "每语句新快照"的两个钩子**（`store_lock` + `external_lock`）：

```cpp
  if (lock_type != TL_IGNORE && trx->n_mysql_tables_in_use == 0) {
    trx->isolation_level = innobase_trx_map_isolation_level(thd_get_trx_isolation(thd));

    if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
        MVCC::is_view_active(trx->read_view)) {
      /* At low transaction isolation levels we let each consistent read set
      its own snapshot */
      mutex_enter(&trx_sys->mutex);
      trx_sys->mvcc->view_close(trx->read_view, true);   // ★① 语句开始：关掉旧 view
      mutex_exit(&trx_sys->mutex);
    }
  }
```

```cpp
  } else if (prebuilt->select_lock_type == LOCK_NONE) {
    /* This is a consistent read */
    /* Assign a read view for the query */
    if (!srv_read_only_mode) {
      trx_assign_read_view(trx);            // ★② 语句中：重开
    }
    prebuilt->sql_stat_start = false;
```

```cpp
  if (trx->n_mysql_tables_in_use == 0) {
    ...
    } else if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
               MVCC::is_view_active(trx->read_view)) {
      mutex_enter(&trx_sys->mutex);
      trx_sys->mvcc->view_close(trx->read_view, true);   // ★③ 语句结束：再关
      mutex_exit(&trx_sys->mutex);
    }
  }
```

> ★★ RR 两个条件都不满足（`isolation_level > RC`）→ 不关不重开 → **整个事务一个 view**。这就是 RR 可重复读的全部实现。
> ⚠️ **RU 不是"不建 view"**——view 照样分配，只是 `row_search_mvcc` 的 else-if 链里显式跳过版本构建：
> ```cpp
>     if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
>       /* Do nothing: we let a non-locking SELECT read the latest version */
> ```

**★ SERIALIZABLE 的升级点在 `external_lock`**（不是 `store_lock`）：

```cpp
      if (m_prebuilt->table->is_dd_table || m_prebuilt->no_read_locking) {
        m_prebuilt->select_lock_type = LOCK_NONE;
      } else if (trx->isolation_level == TRX_ISO_SERIALIZABLE &&
                 m_prebuilt->select_lock_type == LOCK_NONE &&
                 thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
        m_prebuilt->select_lock_type = LOCK_S;   // ★★ 普通 SELECT 升级为锁定读
      }
```

> ★ 源码里有一张完整的决策表注释（`is_dd_table or skip_locking` × 隔离级别 × 三种 SELECT 形态），是理解"谁走快照读、谁走当前读"的权威来源。三条 note 尤其重要：
> - `[1]` 非锁定 SELECT 由 server 调 `store_lock(TL_READ)` → `LOCK_NONE`（ACL 表例外）
> - `[2]` `SELECT ... FOR SHARE` 由 `store_lock(TL_READ_WITH_SHARED_LOCKS)` → `LOCK_S`
> - `[3]` **AUTOCOMMIT=1 的只读事务**即使在 SERIALIZABLE 下也用 `LOCK_NONE`——因为已知只读，可用一致性读串行化

**★★ 提交时序的完整证据链**（"先退 rw_trx_ids 再放锁"）：

```cpp
static void trx_release_impl_and_expl_locks(trx_t *trx, bool serialised) {
  ...
  if (trx_sys_latch_is_needed) trx_sys_mutex_enter();       // ①

  if (trx->id > 0) {
    /* For consistent snapshot, we need to remove current transaction from
    running transaction id list for mvcc before doing commit and releasing
    locks. */
    trx_erase_lists(trx);                                   // ★★ ② 先退三个集合
  }
  ...
  if (trx_sys_latch_is_needed) trx_sys_mutex_exit();        // ③
  ...
  trx->state.store(TRX_STATE_COMMITTED_IN_MEMORY, ...);     // ④ 对外"已提交"
  ...
  if (serialised) {
    trx_sys_serialisation_mutex_enter();
    if (gtid_desc.m_is_set) { gtid_persistor.add(gtid_desc); }   // GTID 必须在退 serialisation_list 之前
    trx_erase_from_serialisation_list_low(trx);             // ★ ⑤ 再退 serialisation_list
    trx_sys_serialisation_mutex_exit();
  }

  lock_trx_release_locks(trx);                              // ★★★ ⑥ 最后才放锁
}
```

```cpp
static void trx_erase_lists(trx_t *trx) {
  ut_ad(trx->id > 0);
  ut_ad(trx_sys_mutex_own());

  trx_ids_t::iterator it = std::lower_bound(trx_sys->rw_trx_ids.begin(),
                                            trx_sys->rw_trx_ids.end(), trx->id);
  ut_ad(*it == trx->id);
  trx_sys->rw_trx_ids.erase(it);          // ★ 1. 退 rw_trx_ids（新建 view 不再含本 trx）
  ...
    trx_remove_from_rw_trx_list(trx);     // ★ 2. 退 rw_trx_list（隐式锁随之释放）
    if (trx->read_view != nullptr) {
      trx_sys->mvcc->view_close(trx->read_view, true);   // ★ 3. view 归还 m_free
    }
}
```

> ★★ ⑤ 的注释解释了为什么必须在 ② 之后：**"Otherwise a read-view could be created, which could still pretend that changes of this transaction are invisible, but related undo records could become purged (because trx->no would no longer protect them)."** 这是"事务可见性声明"与"undo 生命周期"之间的顺序依赖。

**★ AC-NL-RO 走的是另一条路**（不持锁、只打标志）：

```cpp
  if (trx_is_autocommit_non_locking(trx)) {
    ...
    if (trx->read_view != nullptr) {
      trx_sys->mvcc->view_close(trx->read_view, false);   // ★ own_mutex=false
    }
    MONITOR_INC(MONITOR_TRX_NL_RO_COMMIT);
    trx->state.store(TRX_STATE_NOT_STARTED, std::memory_order_relaxed);
  }
```

```cpp
static inline bool trx_is_autocommit_non_locking(const trx_t *t) {
  return t->auto_commit && t->will_lock == 0;
}
```

```cpp
  trx->auto_commit = (trx->api_trx && trx->api_auto_commit) ||
                     thd_trx_is_auto_commit(trx->mysql_thd);
  trx->read_only = (trx->api_trx && !trx->read_write) ||
                   (!trx->internal && thd_trx_is_read_only(trx->mysql_thd)) ||
                   srv_read_only_mode;
  if (!trx->auto_commit) {
    ++trx->will_lock;
  } else if (trx->will_lock == 0) {
    trx->read_only = true;
  }
```

> ★★ **`will_lock` 是"这条语句会不会加锁"的预声明**：autocommit 且无人加锁 → `will_lock == 0` → 标记 read_only → 走 AC-NL-RO 快路径。一旦显式 `FOR UPDATE`，`will_lock > 0`，立刻失去资格。

**★ 可观测性**（⚠️ 名字都很坑）：

```cpp
    {"trx_rseg_history_len", "transaction",
     "Length of the TRX_RSEG_HISTORY list", ... MONITOR_RSEG_HISTORY_LEN},
```

```cpp
    case MONITOR_RSEG_HISTORY_LEN:
      value = trx_sys->rseg_history_len;   // ★ std::atomic<uint64_t>
      break;
```

```cpp
  fprintf(file, "History list length " UINT64PF "\n",
          trx_sys->rseg_history_len.load());     // SHOW ENGINE INNODB STATUS
```

```cpp
  trx_sys->rseg_history_len.fetch_add(n_added_logs);   // 自增（trx0purge.cc:398）
  trx_sys->rseg_history_len.fetch_sub(1);              // 自减（trx0purge.cc:433）
```

```cpp
#ifdef UNIV_DEBUG
    {"purge_trx_id_age", (char *)&export_vars.innodb_purge_trx_id_age, SHOW_LONG, ...},
    {"purge_view_trx_id_age", (char *)&export_vars.innodb_purge_view_trx_id_age, SHOW_LONG, ...},
#endif /* UNIV_DEBUG */
```

> ⚠️ **不存在状态变量 `history_list_len`**——正名是 INNODB_METRICS 的 **`trx_rseg_history_len`**。
> ⚠️ **`Innodb_purge_trx_id_age` / `Innodb_purge_view_trx_id_age` 只在 `UNIV_DEBUG` 编译下存在**——release 版本 `SHOW STATUS` 看不到。且 `purge_trx_id_age` 依赖 `purge_sys->done`，也是 debug-only。
> ★ 生产排障能用的只有 `trx_rseg_history_len` + `information_schema.innodb_trx`（看 `trx_started` / `trx_rows_locked` 判断长事务）。

**★ purge lag → DML 延迟**（长事务反压 DML 的落点）：

```cpp
static ulint trx_purge_dml_delay(void) {
  ulint delay = 0;
  if (srv_max_purge_lag > 0 && trx_sys->rseg_history_len.load() >
                                   srv_n_purge_threads * srv_purge_batch_size) {
    float ratio = float(trx_sys->rseg_history_len.load()) / srv_max_purge_lag;
    if (ratio > 1.0) {
      delay = (ulint)((ratio - 0.9995) * 10000);   // ★ 超 lag 至少 5 微秒
    }
    if (delay > srv_max_purge_lag_delay) delay = srv_max_purge_lag_delay;
    MONITOR_SET(MONITOR_DML_PURGE_DELAY, delay);
  }
  return (delay);
}
```

```cpp
  srv_dml_needed_delay = trx_purge_dml_delay();   // trx_purge() 第一行
```

#### 逆变换两路径、部分行不变式与三态汇总

**版本重建 = "新行 + update undo → 旧行"的逆变换**，`trx_undo_prev_version_build` 内部两条路径：

- **快路径**（字段尺寸/外部存储不变）：`rec_copy` 复制当前行 → `row_upd_rec_in_place` 就地用 update vector 的 `new_val`（即旧值）覆盖对应字段，并把 trx_id/roll_ptr/info_bits 换成旧版本系统列；
- **慢路径**（尺寸或 extern 位变化）：`row_rec_to_index_entry` 转 dtuple → `row_upd_index_replace_new_col_vals` 替换新列值 → `rec_convert_dtuple_to_rec` 按新布局重建整条记录；
- **disowned-external 保护**：旧版本带 delete-mark 且"放弃外部数据"且 purge 已能看见该 trx → 被放弃的 BLOB 可能已被 purge → 直接返回"当作 fresh insert"（`old_vers` 保持 nullptr，防调用方断言失败）。

**★ 两条路径的完整源码**（`trx0rec.cc:2469-2681`）：

```cpp
bool trx_undo_prev_version_build(const rec_t *index_rec, mtr_t *index_mtr,
                                 const rec_t *rec, const dict_index_t *const index,
                                 ulint *offsets, mem_heap_t *heap, rec_t **old_vers,
                                 mem_heap_t *v_heap, const dtuple_t **vrow,
                                 ulint v_status, lob::undo_vers_t *lob_undo) {
  ut_ad(!rw_lock_own(&purge_sys->latch, RW_LOCK_S));   // ★ 进入前禁止已持有 purge latch
  ut_a(index->is_clustered());

  roll_ptr = row_get_rec_roll_ptr(rec, index, offsets);   // ★ 链头指针
  *old_vers = nullptr;

  if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
    /* The record rec is the first inserted version */
    return true;                                  // ★① 返回 true 但 old_vers 保持 NULL
  }

  rec_trx_id = row_get_rec_trx_id(rec, index, offsets);
  bool is_temp = index->table->is_temporary();

  if (trx_undo_get_undo_rec(roll_ptr, rec_trx_id, heap, is_temp,
                            index->table->name, &undo_rec)) {
    if (v_status & TRX_UNDO_PREV_IN_PURGE) {
      /* We are fetching the record being purged */
      undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
    } else {
      /* The undo record may already have been purged,
      during purge or semi-consistent read. */
      return false;                               // ★② DB_MISSING_HISTORY 的唯一源头
    }
  }

  ptr = trx_undo_rec_get_pars(undo_rec, &type, &cmpl_info, &dummy_extern,
                              &undo_no, &table_id, type_cmpl);

  if (table_id != index->table->id) {
    /* The table should have been rebuilt, but purge has not yet removed
    the undo log records for the now-dropped old table (table_id). */
    return true;                                  // ★③ 表被重建：停车（也返回 true + NULL）
  }

  ptr = trx_undo_update_rec_get_sys_cols(ptr, &trx_id, &roll_ptr, &info_bits);
  ptr = trx_undo_rec_skip_row_ref(ptr, index);   // ★ 版本回溯不需要主键值，只 skip
  ptr = trx_undo_update_rec_get_update(ptr, index, type, trx_id, roll_ptr,
                                       info_bits, heap, &update, lob_undo, type_cmpl);
  ut_a(ptr);

  if (row_upd_changes_field_size_or_external(index, offsets, update)) {
    // ═══════ 慢路径：长度/外部存储变化 → 全量重建 ═══════
    if ((update->info_bits & REC_INFO_DELETED_FLAG) &&
        row_upd_changes_disowned_external(update)) {
      bool missing_extern;
      rw_lock_s_lock(&purge_sys->latch, UT_LOCATION_HERE);
      missing_extern = purge_sys->view.changes_visible(trx_id, index->table->name);
      rw_lock_s_unlock(&purge_sys->latch);
      if (missing_extern) {
        /* treat as a fresh insert, not to cause assertion error at the caller. */
        if (update != nullptr) update->reset();
        return true;                              // ★④ disowned-external 保护
      }
    }
    entry = row_rec_to_index_entry(rec, index, offsets, heap);      // rec → dtuple
    row_upd_index_replace_new_col_vals(entry, index, update, heap); // 替换成旧值
    buf = static_cast<byte *>(
        mem_heap_alloc(heap, rec_get_converted_size(index, entry))); // ★ 长度要重算
    *old_vers = rec_convert_dtuple_to_rec(buf, index, entry);       // dtuple → rec
  } else {
    // ═══════ 快路径：长度不变 → 原地打补丁 ═══════
    buf = static_cast<byte *>(mem_heap_alloc(heap, rec_offs_size(offsets)));
    *old_vers = rec_copy(buf, rec, offsets);                        // 整条复制
    rec_offs_make_valid(*old_vers, index, offsets);
    row_upd_rec_in_place(*old_vers, index, offsets, update, nullptr); // 覆盖被改字段
  }

  if (v_status & TRX_UNDO_GET_OLD_V_VALUE) {
    row_upd_replace_vcol((dtuple_t *)*vrow, index->table, update, false, nullptr, nullptr);
  }
  ...
  if (vrow && !(cmpl_info & UPD_NODE_NO_ORD_CHANGE)) {
    //  ★★ "vrow 仍为 NULL" 是 row_vers_impl_x_locked_low 的关键假设
    if (!(*vrow)) {
      *vrow = dtuple_create_with_vcol(v_heap ? v_heap : heap,
                                      index->table->get_n_cols(),
                                      dict_table_get_n_v_cols(index->table));
      dtuple_init_v_fld(*vrow);
    }
    trx_undo_read_v_cols(index->table, ptr, *vrow, ...);
  }
  if (update != nullptr) update->reset();
  return true;
}
```

**★ 读 undo 前的双保险**（`trx0rec.cc:2440`，`purge_sys->latch` 的三个使用点之一）：

```cpp
[[nodiscard]] static bool trx_undo_get_undo_rec(roll_ptr_t roll_ptr,
                                                trx_id_t trx_id, mem_heap_t *heap,
                                                bool is_temp, const table_name_t &name,
                                                trx_undo_rec_t **undo_rec) {
  bool missing_history;
  rw_lock_s_lock(&purge_sys->latch, UT_LOCATION_HERE);

  missing_history = purge_sys->view.changes_visible(trx_id, name);
  //  ★★ purge view（=最老 view）都可见 ⇒ 没有任何 view 还需要它 ⇒ 可能已被清
  if (!missing_history) {
    *undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
  }
  rw_lock_s_unlock(&purge_sys->latch);
  return (missing_history);
}
```

```cpp
[[nodiscard]] static trx_undo_rec_t *trx_undo_get_undo_rec_low(roll_ptr_t roll_ptr,
                                                               mem_heap_t *heap,
                                                               bool is_temp) {
  trx_undo_decode_roll_ptr(roll_ptr, &is_insert, &rseg_id, &page_no, &offset);
  space_id = trx_rseg_id_to_space_id(rseg_id, is_temp);
  mtr_start(&mtr);
  undo_page = trx_undo_page_get_s_latched(page_id_t(space_id, page_no), page_size, &mtr);
  undo_rec = trx_undo_rec_copy(undo_page, static_cast<uint32_t>(offset), heap);
  //  ★★ 必须拷贝：mtr 立刻提交、latch 释放
  mtr_commit(&mtr);
  return (undo_rec);
}
```

> ★★ **三个"返回 true 但 `old_vers == nullptr`"的情形**常被漏掉：① insert undo（链到头）② `table_id` 不匹配（表被重建）③ disowned-external 已被 purge。它们都被伪装成"fresh insert"，目的是**让调用方不 assert**。

**★ "部分行"不变式的写侧证据**（`trx_undo_page_report_modify`，`trx0rec.cc:1513`）：

```cpp
  if (!update || !(cmpl_info & UPD_NODE_NO_ORD_CHANGE)) {
    //  ★ 只有"不是 NO_ORD_CHANGE"才写部分行
    ...
    if (col->ord_part) { ... }                    // ★ 普通列：仅 ord_part
    ...
    if (col->m_col.ord_part) { ... }              // ★ 虚拟列：仅 m_col.ord_part
  }
```

> ★★ 不变式：**任意索引的排序列，在新旧版本间要么值未变（可从新行继承），要么被 update 命中（旧值在 undo 中）**。这是"部分行"能成立的唯一理由。
> ⚠️ `UPD_NODE_NO_ORD_CHANGE` 时虚拟列旧值**整个不记录**——调用方必须遵守"索引列值未变"约定（`row0vers.cc:2662` 的 `if (vrow && !(cmpl_info & UPD_NODE_NO_ORD_CHANGE))`）。

**"部分行"成立的不变式**（写侧 `trx_undo_page_report_modify`，trx0rec.cc:1159）：普通列只写**被更新列**的旧值；虚拟列**仅当 `col->m_col.ord_part`（参与索引排序）**时才写旧值；未被更新的列不写（旧版本中与新版本同值，重建时沿袭）。⇒ **任意索引的排序列，在新旧版本间要么值未变（可从新行继承），要么被 update 命中（旧值在 undo 中）**。`UPD_NODE_NO_ORD_CHANGE` 是进一步优化：不触碰任何索引列时虚拟列旧值整个不记录——调用方必须遵守"索引列值未变"约定（`row0vers.cc:112-118` 注释）。

**三态返回值汇总**：

| 返回 | `*old_vers` | 语义 | 触发 |
|---|---|---|---|
| `true` | `nullptr` | 版本链到头 | `trx_undo_roll_ptr_is_insert`（fresh insert）/ `table_id` 不匹配（表已重建）/ disowned-external 被 purge |
| `true` | 非空 | 成功构建上一版本 | 正常逆变换 |
| `false` | `nullptr` | **历史缺失** | `purge_sys->view.changes_visible(rec_trx_id)` 为真——undo 可能已被 purge 清理 |

`row_vers_build_for_consistent_read` 把 `false` 映射为 `DB_MISSING_HISTORY`（"Required history data has been deleted"，对应 Oracle ORA-01555 同类事件）。差异：MySQL 在**普通非锁定一致读**中把 `old_vers == nullptr` 解释为"该行在本快照中不存在"，`goto next_rec` **静默跳过**（结果集缺行）；而锁读/回表等需要旧版本上下文的路径中错误向上传播。

**版本链遍历的代价**（每回退一个版本）：undo 页 S 锁 + 一次 mtr 往返 → 整条 undo 记录拷贝 → 全量解析（逐字段 compressed 解码）→ 行重建（快/慢路径）→ `mem_heap_create(1024)` 新堆 + 释放上一轮堆——链长 N 则全流程 × N，且 `trx_undo_get_undo_rec` 每版本一次 `purge_sys->latch` 全局 S 锁。

### 隔离级别全景与 ReadView 生命周期（源码逐段）

#### 分派入口：`trx_assign_read_view`

```cpp
ReadView *trx_assign_read_view(trx_t *trx) {
  if (srv_read_only_mode) {
    return (nullptr);                                   // 只读实例：彻底无 view
  } else if (!MVCC::is_view_active(trx->read_view)) {
    trx_sys->mvcc->view_open(trx->read_view, trx);      // 未 open（含 tag=1 已关闭）才 open
  }
  return (trx->read_view);
}
```

RR 的复用靠 `is_view_active`：RR 事务第一条一致性读 open 后指针一直有效（未打 tag）→ 后续语句直接复用。

#### 各隔离级别的行为

| 隔离级别 | view 行为 | 读到的版本 |
|---------|----------|-----------|
| READ UNCOMMITTED | 不建 view | 直读最新版（`row_search_mvcc`：RU 分支什么都不做） |
| READ COMMITTED | 每语句新建（语句边界关 + 重开） | 语句开始时刻已提交的版本 |
| REPEATABLE READ | 事务首次一致性读建，事务结束才关 | 事务内第一次读的快照 |
| SERIALIZABLE | 普通 SELECT 在 `external_lock` 被升级为 `LOCK_S`（锁定读） | 当前读 + gap 锁（无快照读） |

**RC 的"每语句新快照"实现**（两条 handler 钩子）：

- 语句开始（`ha_innobase::store_lock`）：

```cpp
if (trx->isolation_level <= TRX_ISO_READ_COMMITTED && MVCC::is_view_active(trx->read_view)) {
  /* At low transaction isolation levels we let
  each consistent read set its own snapshot */
  trx_sys->mvcc->view_close(trx->read_view, true);   // 上一条语句的 view 作废
}
```

- 语句中（`row_search_mvcc` 的 `sql_stat_start` 分支）：`LOCK_NONE` 时 `trx_assign_read_view(trx)` 重开。

RR 两个条件都不满足 → 不关不重开 → 整个事务一个 view。

#### `START TRANSACTION WITH CONSISTENT SNAPSHOT`

```cpp
static int innobase_start_trx_and_assign_read_view(handlerton *hton, THD *thd) {
  ...
  if (trx->isolation_level == TRX_ISO_REPEATABLE_READ) {
    trx_assign_read_view(trx);       // 事务第一条语句执行前就建快照
  } else {
    push_warning_printf(... "WITH CONSISTENT SNAPSHOT was ignored ...");  // 仅 RR 有效
  }
}
```

**HANDLER 语句特殊**：`ha_innobase` 的 handler open/read 强制 `trx_assign_read_view` + `select_lock_type = LOCK_NONE`——HANDLER 读永远按一致性读处理（注释："We let HANDLER always to do the reads as consistent reads"，即使 SERIALIZABLE）。

#### 提交时序：为什么必须"先退 rw_trx_ids 再放锁"

```
trx_commit
  └─ trx_write_serialisation_history        ← undo 收尾：分配 trx->no、入 serialisation_list、
  │                                            purge_queue、挂 rseg history list、mtr commit
  └─ trx_commit_in_memory                   ← 内存提交（对外可见）
        ├─ trx_release_impl_and_expl_locks
        │     ├─ trx_sys_mutex_enter()
        │     ├─ trx_erase_lists            ★ 顺序关键：
        │     │     ① rw_trx_ids.erase(...)      先退出快照集合
        │     │     ② trx_remove_from_rw_trx_list 再出 rw_trx_list（隐式锁随之消失）
        │     │     ③ mvcc->view_close(trx->read_view, true)   view 归还 m_free
        │     ├─ trx_erase_from_serialisation_list_low
        │     └─ lock_trx_release_locks     ④ 最后放行锁
        └─ 置 TRX_STATE_COMMITTED_IN_MEMORY
```

顺序正确性论证（`trx0sys.h` 注释原文）：**必须先从 `rw_trx_ids` 移除（①）再放锁（④）**——否则新 view 在"事务已放锁（其修改已生效可见）"与"id 仍在活跃集"之间会产生矛盾快照。

### 半一致性读（semi-consistent read）

#### 是什么

RC/RU 隔离级别下，UPDATE/DELETE 扫描遇到**被其他事务锁住的行**时不等待锁，而是返回该行**最新已提交版本**让 SQL 层判断是否满足 WHERE——不满足直接跳过，满足才回头加锁重读。"先乐观后悲观"的两阶段：读已提交版本时无锁（OCC），确认要更新再退化悲观加锁（lazy locking）。

#### 触发条件（四个同时成立）

```cpp
const bool use_semi_consistent =
    prebuilt->row_read_type == ROW_READ_TRY_SEMI_CONSISTENT
    && !unique_search && index == clust_index && !trx_is_high_priority(trx);
```

- `unique_search` = `match_mode == ROW_SEL_EXACT` 且唯一索引且给了完整唯一键——排除的是"精确唯一点查"（命中最多一行，语义要求精确，不走"先读旧版本再重读"；注意等值查普通二级索引**也算** `!unique_search`）。
- `index == clust_index`：版本链只挂在聚簇索引上（`DB_ROLL_PTR`），"读旧版本"只能在聚簇记录上做；函数内有 `ut_ad(index->is_clustered())`。
- `!trx_is_high_priority`：只有复制从库 SQL 线程 / MTS worker 是高优先级（`thd_tx_priority` 只由 `rpl_replica.cc` 设置），applier 的锁冲突可压过普通事务，不用半一致降级。

**★ 开关的落点**：`try_semi_consistent_read()` 有闸门，**RR 下即使 server 打开开关也不生效**：

```cpp
void ha_innobase::try_semi_consistent_read(bool yes) {
  ut_a(m_prebuilt->trx == thd_to_trx(ha_thd()));
  if (yes && m_prebuilt->trx->allow_semi_consistent()) {
    //  ★ allow_semi_consistent() == skip_gap_locks() == 隔离级别 <= RC
    m_prebuilt->row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;
  } else {
    m_prebuilt->row_read_type = ROW_READ_WITH_LOCKS;   // ★ RR/SERIALIZABLE 彻底关闭
  }
}

bool ha_innobase::was_semi_consistent_read(void) {
  return (m_prebuilt->row_read_type == ROW_READ_DID_SEMI_CONSISTENT);
  //  ⚠️ 只是布尔查询，不解锁
}
```

```cpp
  bool allow_semi_consistent() const { return (skip_gap_locks()); }
  bool releases_non_matching_rows() const { return skip_gap_locks(); }
  bool skip_gap_locks() const {
    switch (isolation_level) {
      case READ_UNCOMMITTED:
      case READ_COMMITTED:
        return (true);
      case REPEATABLE_READ:
      case SERIALIZABLE:
        return (false);
    }
  }
```

> ★★ **这三个方法在 8.0.39 语义完全等价**，都直接委派 `skip_gap_locks()`（"隔离级别 ≤ RC"）。所以"半一致读的正确性依赖不加 gap lock"这条在代码里是**自洽**的。

**★ `row_read_type` 的真名是 `ulint` 字段 + 三个 `constexpr`，不是 enum**：

```cpp
  ulint row_read_type;          /*!< ROW_READ_WITH_LOCKS if row locks
                                should be the obtained for records
                                under an UPDATE or DELETE cursor.
                                If trx_t::allow_semi_consistent()
                                returns true, this can be set to
                                ROW_READ_TRY_SEMI_CONSISTENT, ... */
```

```cpp
/* Values for row_read_type */
constexpr uint32_t ROW_READ_WITH_LOCKS = 0;
constexpr uint32_t ROW_READ_TRY_SEMI_CONSISTENT = 1;
constexpr uint32_t ROW_READ_DID_SEMI_CONSISTENT = 2;
```

**★★ 分派点：`case DB_SKIP_LOCKED` 里先查 `select_mode`**（这是"用户 `SKIP LOCKED`"与"半一致读"的分水岭）：

```cpp
    const bool use_semi_consistent =
        prebuilt->row_read_type == ROW_READ_TRY_SEMI_CONSISTENT &&
        !unique_search && index == clust_index && !trx_is_high_priority(trx);
    err = sel_set_rec_lock(
        pcur, rec, index, offsets,
        use_semi_consistent ? SELECT_SKIP_LOCKED : prebuilt->select_mode,
        //  ★★ 半一致读临时"借用" SKIP LOCKED 模式，避免建一个注定不等的 WAITING 锁
        prebuilt->select_lock_type, lock_type, thr, &mtr);

    switch (err) {
      case DB_SUCCESS_LOCKED_REC:
        if (trx->releases_non_matching_rows()) {
          prebuilt->new_rec_lock[row_prebuilt_t::LOCK_PCUR] = true;
        }
        err = DB_SUCCESS;
        [[fallthrough]];
      case DB_SUCCESS: ...
      case DB_SKIP_LOCKED:
        if (prebuilt->select_mode == SELECT_SKIP_LOCKED) {
          goto next_rec;                      // ★① 用户显式 SKIP LOCKED → 真跳行
        }
        ut_a(use_semi_consistent);
        ut_a(trx->allow_semi_consistent());
        /* The following call returns 'offsets' associated with 'old_vers' */
        row_sel_build_committed_vers_for_mysql(   // ★② 半一致读 → 构建"最后已提交版本"
            clust_index, prebuilt, rec, &offsets, &heap, &old_vers,
            need_vrow ? &vrow : nullptr, &mtr);
        err = DB_SUCCESS;
        if (old_vers == nullptr) {
          /* The row was not yet committed */
          goto next_rec;
        }
        did_semi_consistent_read = true;
        rec = old_vers;
        prev_rec = rec;
        break;
      case DB_LOCK_WAIT:
        /* Never unlock rows that were part of a conflict. */
        prebuilt->new_rec_lock.reset();       // ★★ 冲突行不算"新锁"
        ut_a(!use_semi_consistent);           // ★ 半一致路径永不产生 DB_LOCK_WAIT
        goto lock_wait_or_error;
```

**★ `new_rec_lock` 是 `std::bitset<2>`，不是 `bool[2]`**：

```cpp
  enum { LOCK_PCUR, LOCK_CLUST_PCUR, LOCK_COUNT, };
  /** normally false;
  if session is using READ COMMITTED or READ UNCOMMITTED isolation level, set in
  row_search_for_mysql() if we set a new record lock on the secondary or
  clustered index;
  this is used in row_try_unlock() when releasing the lock under the cursor if
  we determine after retrieving the row that it does not need to be locked
  ('mini-rollback')
    [LOCK_PCUR] corresponds to pcur, the first index we looked up
                (can be secondary or clustered!)
    [LOCK_CLUST_PCUR] corresponds to clust_pcur, ...
  The meaning of these booleans is:
    true = we've created a rec lock, which we might release as we "own" it
    false = we should not release any lock for this index as we either reused
            some existing lock, or there is some other reason, we should keep it */
  std::bitset<LOCK_COUNT> new_rec_lock;
  size_t new_rec_locks_count() const { return new_rec_lock.count(); }
```

**★ DID 标志只在返回前才真正写回**（`row0sel.cc:6076`）：

```cpp
  ut_ad(prebuilt->row_read_type != ROW_READ_WITH_LOCKS || !did_semi_consistent_read);
  if (prebuilt->row_read_type != ROW_READ_WITH_LOCKS) {   // ★ RR/SR 下不写标志
    if (did_semi_consistent_read) {
      prebuilt->row_read_type = ROW_READ_DID_SEMI_CONSISTENT;
    } else {
      prebuilt->row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;
    }
  }
```

**★ `unlock_row()` 对 DID 是"只复位不解锁"**（因为半一致成功意味着一把锁都没上）：

```cpp
  if (m_prebuilt->row_read_type == ROW_READ_DID_SEMI_CONSISTENT) {
    ut_ad(m_prebuilt->new_rec_locks_count() == 0);   // ★★ 半一致读成功 = 一把锁都没有
    m_prebuilt->row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;
  }
  m_prebuilt->try_unlock(false);
```

**★ 下一轮进来时 DID 的两个出口**（`row0sel.cc:4880`）：

```cpp
    auto need_to_process = sel_restore_position_for_mysql(
        &same_user_rec, BTR_SEARCH_LEAF, pcur, moves_up, &mtr);
    if (UNIV_UNLIKELY(need_to_process)) {
      if (UNIV_UNLIKELY(prebuilt->row_read_type == ROW_READ_DID_SEMI_CONSISTENT)) {
        /* We did a semi-consistent read, but the record was removed in
        the meantime. */
        prebuilt->row_read_type = ROW_READ_TRY_SEMI_CONSISTENT;   // 记录被删 → 退回 TRY
      }
    } else if (UNIV_LIKELY(prebuilt->row_read_type != ROW_READ_DID_SEMI_CONSISTENT)) {
      goto next_rec;      // ★★ DID 时不 goto next_rec —— 继续处理同一行（悲观重读）
    }
```
- **正确性依赖"不加 gap lock"**（`row0sel.cc` 注释）：gap lock 锁定范围，范围末尾行被删/purge 时半一致行为难证。因此严格限定在 `skip_gap_locks()` 为 true 的 RC/RU（`allow_semi_consistent() == skip_gap_locks()` 自洽）。

#### 加锁时机：两条路径

引擎层在 SQL 层 WHERE 过滤**之前**就尝试加锁：

- **路径 A（行未锁）**：`sel_set_rec_lock` 直接成功，返回 `DB_SUCCESS_LOCKED_REC`；RC/RU 下记 `new_rec_lock[...] = true`。SQL 层：WHERE 满足 → 保留锁更新；不满足 → `unlock_row` 释放这个新锁（`releases_non_matching_rows()` 仅 RC/RU 为 true）。
- **路径 B（行已被锁）**：以 `SELECT_SKIP_LOCKED` 模式请求，不排队直接返回 `DB_SKIP_LOCKED` → 构建旧版本、置 `DID`。**加锁被推迟到 WHERE 判定之后**：不满足 → 跳过；满足 → 重读该行，此时 `row_read_type == DID`（非 TRY），`use_semi_consistent = false`，改用 `prebuilt->select_mode` 普通模式悲观等待加锁。

结论：未锁行"读行即加锁"（WHERE 前）；已锁行才把加锁推迟到 WHERE 满足后。半一致性读省的是"已锁无关行"的等待与加锁。

#### 核心算法：沿 undo 回溯找「最后已提交」的版本

```cpp
void row_vers_build_for_semi_consistent_read(const rec_t *rec, mtr_t *mtr,
                                             dict_index_t *index, ulint **offsets,
                                             mem_heap_t **offset_heap, mem_heap_t *in_heap,
                                             const rec_t **old_vers, const dtuple_t **vrow) {
  version = rec;
  for (;;) {
    version_trx_id = row_get_rec_trx_id(version, index, *offsets);
    if (rec == version) rec_trx_id = version_trx_id;   // 记住最新版的 trx_id

    if (!trx_rw_is_active(version_trx_id, false)) {
    committed_version_trx:
      /* 找到了属于已提交事务的版本 */
      if (rec == version) {
        *old_vers = rec;                 // ① 最新版本身已提交 → 直接用，不拷贝
        break;
      }
      if (rec_trx_id == version_trx_id) {
        /* 回溯途中原事务提交了 → 弹回最新版 */
        version = rec;                   // ②
        *offsets = rec_get_offsets(version, index, ...);
      }
      *old_vers = rec_copy(buf, version, *offsets);   // ③ 返回"最后已提交"的版本
      break;
    }

    if (!trx_undo_prev_version_build(rec, mtr, version, index, *offsets, heap,
                                     &prev_version, in_heap, vrow, 0, nullptr)) {
      goto committed_version_trx;   // ④ missing history（purge 竞态）：把当前 version 当已提交版本返回
    }
    if (prev_version == nullptr) {
      *old_vers = nullptr;          // ⑤ 最新版本由活跃事务 INSERT → 无已提交版本
      break;
    }
    version = prev_version;         // 继续回溯
  }
}
```

**与一致性读的本质区别**：

| | 一致性读 `row_vers_build_for_consistent_read` | 半一致性读 `row_vers_build_for_semi_consistent_read` |
|---|---|---|
| 判定基准 | ReadView（快照） | **没有 view**；`trx_rw_is_active()` 查"修改者现在还活跃吗" |
| 目标版本 | 快照时刻可见的版本 | **此刻最后一个已提交**的版本（目标随时间前移） |
| 终止条件 | 版本对 view 可见 | 版本的 trx_id 不在活跃集合（已提交/已回滚） |
| missing history | 报 `DB_MISSING_HISTORY` | ④ 直接把当前 version 当已提交版本返回 |
| 回滚语义 | 不涉及 | 注释明说：回滚中事务保持 `TRX_STATE_ACTIVE` 直到回滚完成，活跃判定不会误判 |

这符合 UPDATE 当前读的语义：**更新必须基于最新已提交值，而非旧快照**——半一致性读比快照读"新"。

#### 三个边界行为

**① `old_vers == nullptr`（行从未提交）→ 直接跳过。** 调用方 `row_search_mvcc` 的 `DB_SKIP_LOCKED` 分支：

```cpp
row_sel_build_committed_vers_for_mysql(clust_index, prebuilt, rec, &offsets,
                                       &heap, &old_vers, ..., &mtr);
err = DB_SUCCESS;
if (old_vers == nullptr) {
  /* The row was not yet committed */
  goto next_rec;                    // 活跃事务 INSERT 的行，对 UPDATE 不可见
}
did_semi_consistent_read = true;
rec = old_vers;
prev_rec = rec;
```

**② delete-marked 的已提交版本 → 被公共检查统一跳过。** 半一致性分支 `break` 后 `rec = old_vers`（旧版本），继续走到锁定读/一致性读的公共路径 `rec_get_deleted_flag` 检查（注释原文 "at this point rec can be an old version of a clustered index record built for a consistent read"）——若"最后已提交版本"是 delete-marked（行已被并发事务删除），同样 `try_unlock` + `next_rec` 跳过。**这条检查是半一致性读与一致性读共用的最后一道闸**，`row_vers_build_for_semi_consistent_read` 本身不负责 delete-mark 语义。

**③ 包装层 `row_sel_build_committed_vers_for_mysql`**：半一致性读不直接调 `row_vers_build_for_semi_consistent_read`，中间隔这一层——负责 mem_heap 管理（分配/释放 `offset_heap`/`in_heap`）、把返回的 `old_vers` 与 offsets 对齐，以及 R-tree 断言等收尾。一致性读侧对应的包装是 `row_sel_build_prev_vers_for_mysql`。

#### 状态机与重读机制

`row_read_type` 三态（`row0mysql.h`）：

| 值 | 含义 |
|----|------|
| `ROW_READ_WITH_LOCKS`(0) | 正常加锁读 |
| `ROW_READ_TRY_SEMI_CONSISTENT`(1) | 尝试半一致性读 |
| `ROW_READ_DID_SEMI_CONSISTENT`(2) | 刚做了半一致性读（未持锁） |

**同一行的两次往返**（完整协议，引擎层与 SQL 层各走两遍）：

```
【第一遍：TRY，无锁试探】
SQL: try_semi_consistent_read(true) → row_read_type = TRY
row_search_mvcc:
  sel_set_rec_lock(SELECT_SKIP_LOCKED) → 行被锁 → DB_SKIP_LOCKED
  → row_sel_build_committed_vers_for_mysql → 最后已提交版本 old_vers
  → 返回 SQL 层，前置 row_read_type = DID      （未持锁！）
SQL 层 EvaluateJoinCondition(old_vers):
  ├─ 不满足 WHERE → unlock_row(): DID 只复位回 TRY，不释放锁（从未加锁）→ 下一行
  └─ 满足 WHERE → was_semi_consistent_read() == true → 继续读同一行
【第二遍：DID，悲观重读】
row_search_mvcc（游标 store_position 已存好，sel_restore_position_for_mysql 恢复）:
  row_read_type == DID → use_semi_consistent = false
  → sel_set_rec_lock(select_mode 普通模式)     ← 这一次来真的
      ├─ 锁已被对手释放 → 直接成功
      └─ 仍未释放 → DB_LOCK_WAIT → que_thr_stop 挂起，等锁
  → 成功后 row_read_type 置回 TRY
```

四个关键点：

1. **第二遍才可能真正阻塞**：DID 状态下走普通模式（不再 SELECT_SKIP_LOCKED），拿不到锁就 `DB_LOCK_WAIT` → `que_thr_stop` 挂起——**等待只发生在"WHERE 已确认要更新这一行"之后**，这是半一致性读省等待的精确边界。
2. **游标不推进**：半一致性路径不走 `next_rec`（PHASE 5 移动游标在 `next_rec` 标签内），`pcur->store_position` 存好位置，第二遍 `row_search_mvcc`（direction != 0）经 `sel_restore_position_for_mysql` 恢复位置；游标停同一行且 DID 时继续处理同一行（不 `goto next_rec`）。
3. **`DB_LOCK_WAIT` 时 `new_rec_lock.reset()`**（`"Never unlock rows that were part of a conflict"`）：路径 A 新加的锁若卷入锁等待冲突，这些行不算"新锁"，`unlock_row` 不会释放它们——锁语义的精确性要求。
4. **`SELECT_SKIP_LOCKED`（prebuilt 模式）vs `DB_SKIP_LOCKED`（返回值）别混淆**：用户显式 `SKIP LOCKED` 语法时 `select_mode == SELECT_SKIP_LOCKED` → 真跳过（`next_rec`）；半一致性读只是**临时借用**"不等待"语义（`select_mode` 本身未变），走构建旧版本。分派点就是 `case DB_SKIP_LOCKED` 里先查 `select_mode`。

#### server 层触发点

`sql/sql_update.cc` 三处调 `table->file->try_semi_consistent_read(true)`：单表 UPDATE 主表扫描前、多表 UPDATE 驱动表、`UpdateRowsIterator::Init()` 嵌套循环外表。InnoDB 侧 `ha_innobase::try_semi_consistent_read` 有 `trx->allow_semi_consistent()`（= 隔离级别 ≤ RC）闸门——**RR 下即使 server 打开开关也不生效**。

**缓冲 row id 的配合**：`UpdateRowsIterator` 分两阶段（扫描 + 按行 id 延迟更新）。半一致性读行在 `DoImmediateUpdatesAndBufferRowIds` 开头 `was_semi_consistent_read()` 直接 `return false`，让嵌套循环迭代器重读同一行（触发第 4 步的悲观加锁重读）。

#### 半一致读要的是「最新已提交版本」，不是「视图可见版本」

上面第 4 点说 `case DB_SKIP_LOCKED` 里"走构建旧版本"——落到代码是 `row_sel_build_committed_vers_for_mysql`，一个 26 行的薄包装：

```cpp
static void row_sel_build_committed_vers_for_mysql(
    dict_index_t *clust_index, row_prebuilt_t *prebuilt, const rec_t *rec,
    ulint **offsets, mem_heap_t **offset_heap,
    const rec_t **old_vers,    /* out: 旧版本；NULL = 该行在视图里还不存在（之后才插入的） */
    const dtuple_t **vrow,     /* out: 旧版本里的虚拟列 */
    mtr_t *mtr) {
  // old_vers_heap 复用：有就清空，没有才按当前 offsets 大小创建
  // ⇒ 每行重建一次堆太贵，这里按"用过的容量"反复复用
  if (prebuilt->old_vers_heap) {
    mem_heap_empty(prebuilt->old_vers_heap);
  } else {
    prebuilt->old_vers_heap =
        mem_heap_create(rec_offs_size(*offsets), UT_LOCATION_HERE);
  }

  // ★ 注意是 semi_consistent_read 版本，不是普通快照读的 row_vers_build_for_consistent_read
  row_vers_build_for_semi_consistent_read(rec, mtr, clust_index, offsets,
                                          offset_heap, prebuilt->old_vers_heap,
                                          old_vers, vrow);
}
```

★ **两个"构建版本"函数极易混淆**：

| 函数 | 场景 | 底层 |
|---|---|---|
| `row_sel_build_committed_vers_for_mysql` | **半一致读**（UPDATE 在 RC 下临时借用"不等待"语义） | `row_vers_build_for_semi_consistent_read` |
| `row_sel_build_prev_vers_for_mysql` | 普通快照读（含回表时发现不可见） | `row_vers_build_for_consistent_read` |

调用点把 `old_vers == nullptr` 的语义写得很直白：

```cpp
        ut_a(use_semi_consistent);
        ut_a(trx->allow_semi_consistent());
        /* The following call returns 'offsets' associated with 'old_vers' */
        row_sel_build_committed_vers_for_mysql(
            clust_index, prebuilt, rec, &offsets, &heap, &old_vers,
            need_vrow ? &vrow : nullptr, &mtr);
        ...
        if (old_vers == nullptr) {
          /* The row was not yet committed */
```

⇒ `nullptr` = 这一行**还没有已提交版本**（是别的事务刚插入、尚未提交）⇒ 对本 UPDATE 来说这行"不该被看到"，直接跳过，**不等锁**。这正是半一致读省掉等待的地方。

顺着往下挖，`row_vers_build_for_semi_consistent_read` 才是实质（109 行，**逐行剖析见上文「核心算法」**）。★ **两者的本质区别在签名上就能看出来**：

```
row_vers_build_for_consistent_read(..., read_view_t *view, ...)    ← 有 view
row_vers_build_for_semi_consistent_read(..., /* 没有 view */ ...)   ← 无 view
```

```cpp
void row_vers_build_for_semi_consistent_read(
    const rec_t *rec, mtr_t *mtr, dict_index_t *index, ulint **offsets,
    mem_heap_t **offset_heap, mem_heap_t *in_heap, const rec_t **old_vers,
    const dtuple_t **vrow) {
  version = rec;
  for (;;) {
    mem_heap_t *heap2;
    rec_t *prev_version;
    trx_id_t version_trx_id;

    version_trx_id = row_get_rec_trx_id(version, index, *offsets);
    if (rec == version) {
      rec_trx_id = version_trx_id;   // 记住最新版的事务 id，后面竞态判断要用
    }

    // ★ 判定标准：这个事务还活跃吗（即未提交）——完全不看 read view
    //   对比普通快照读（row0vers.cc:1328）：view->changes_visible(trx_id, name)
    //   ⇒ 半一致读要的是"最新的已提交版本"，不关心它是否在我的视图里可见
    if (!trx_rw_is_active(version_trx_id, false)) {
    committed_version_trx:
      /* We found a version that belongs to a committed transaction: return it. */
      if (rec == version) {
        *old_vers = rec;             // 最新版本身就是已提交的，直接用
        if (vrow) { *vrow = nullptr; }
        break;
      }

      /* We assume that a rolled-back transaction stays in
      TRX_STATE_ACTIVE state until all the changes have been rolled back... */
      // ★ 竞态：回溯过程中原事务提交了 ⇒ 回到最新版（rec_trx_id == version_trx_id
      //   说明当前版本就是最初那条记录的版本）
      //   注释明说：回滚中的事务仍算 ACTIVE ⇒ 不会误取回滚中途的版本
      if (rec_trx_id == version_trx_id) {
        version = rec;
        *offsets = rec_get_offsets(version, index, *offsets, ULINT_UNDEFINED,
                                   UT_LOCATION_HERE, offset_heap);
      }

      // 拷进调用方的 heap（即 prebuilt->old_vers_heap）
      buf = static_cast<byte *>(mem_heap_alloc(in_heap, rec_offs_size(*offsets)));
      *old_vers = rec_copy(buf, version, *offsets);
      rec_offs_make_valid(*old_vers, index, *offsets);
      if (vrow && *vrow) {
        *vrow = dtuple_copy(*vrow, in_heap);
        dtuple_dup_v_fld(*vrow, in_heap);   // 虚拟列要深拷
      }
      break;
    }

    heap2 = heap;
    heap = mem_heap_create(1024, UT_LOCATION_HERE);
    if (!trx_undo_prev_version_build(rec, mtr, version, index, *offsets, heap,
                                     &prev_version, in_heap, vrow, 0, nullptr)) {
      mem_heap_free(heap);
      heap = heap2;
      heap2 = nullptr;
      goto committed_version_trx;    // ★ 没有更早版本了 ⇒ 兜底返回当前版本
    }
    if (heap2) {
      mem_heap_free(heap2);          // 释放上一版本；heap / heap2 交替推进
    }
    if (prev_version == nullptr) {
      /* It was a freshly inserted version */
      *old_vers = nullptr;           // 刚插入的，没有更早版本
      break;
    }
    version = prev_version;
    *offsets = rec_get_offsets(version, index, *offsets, ULINT_UNDEFINED,
                               UT_LOCATION_HERE, offset_heap);
  }
  if (heap) { mem_heap_free(heap); }
}
```

**`committed_version_trx` 有两个入口**，第二个容易被忽略：`trx_undo_prev_version_build` 失败（没有更早版本）时**兜底返回当前版本**——哪怕最新版未提交，因为没有别的版本可给了。

**★ 为什么半一致读可以不管 read view**：它的目的不是"取出一行给我看"，而是**先判断这行是否满足 WHERE、值不值得等锁**。读的是**最新已提交状态**；若它满足 WHERE，后面还有一遍**悲观加锁重读**（走普通路径、按 read view 校验可见性）。⇒ 半一致读只是**省掉"这行根本不满足条件"时的无谓等待**，可见性的最终把关仍在第二遍。

| | 普通快照读 | 半一致读 |
|---|---|---|
| 判定 | `view->changes_visible(trx_id)` | `!trx_rw_is_active(trx_id)` |
| 找的是 | **我视图可见**的版本 | **最新已提交**的版本 |
| 用途 | 真正返回给用户的结果 | 判断要不要等锁 |

### purge 与 ReadView 的水位闭环（源码逐段）

#### trx_no 与 serialisation_list

事务提交时（有 update undo 者）：

```cpp
static inline bool trx_add_to_serialisation_list(trx_t *trx) {
  trx->no = trx_sys_allocate_trx_no();            // 从 next_trx_id_or_no 原子取号（提交序）
  UT_LIST_ADD_LAST(trx_sys->serialisation_list, trx);
  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) == 1) {
    trx_sys->serialisation_min_trx_no.store(trx->no);   // 列表第一个 = 最小 no
  }
}
```

只有产生 update undo 的事务才有 `trx->no`（只读事务跳过入队但仍分配 no）。`ReadView::prepare` 的 `m_low_limit_no` 就取 `serialisation_min_trx_no`——"此刻最老的、尚未完全收尾的已提交事务"。取号单调 + 列表单调推进 ⇒ 任何未来 view 的 `m_low_limit_no` 只会 ≥ 当前值——这是 view 链能按 `m_low_limit_no` 降序排的前提。

#### 为什么 purge 不能删"最老活跃 ReadView 之后的版本"

版本回溯（`trx_undo_prev_version_build`）需要**逐步**走到可见版本，删掉链中间的 undo 会让上层 view 撞上 missing history。purge view 克隆全局最老 view（FACT C：其 `low_limit_no` ≤ 任何现有/未来 view），因此"trx_no < purge 边界"的 undo **不可能被任何 view 需要**；边界之上的可能正被某个活跃 view 用来回溯，绝不能删。

#### clone_oldest_view 与三处水位使用

```cpp
void MVCC::clone_oldest_view(ReadView *view) {
  trx_sys_mutex_enter();
  ReadView *oldest_view = get_oldest_view();   // m_views 尾部第一个未 closed 的
  if (oldest_view == nullptr) {
    view->prepare(0);                          // 系统无 view：现场造一个（creator=0）
  } else {
    view->copy_prepare(*oldest_view);          // 持锁段只做纯拷贝
    trx_sys_mutex_exit();
    view->copy_complete();                     // 锁外：把 oldest 的 creator id 插回 m_ids
  }
  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  view->reduce_low_limit(gtid_persistor.get_oldest_trx_no());  // ★ GTID 未持久化的 undo 不能清（clone 插件约束）
}
```

purge 每批开始 `trx_purge` 调它刷新 `purge_sys->view`（嵌入式对象，不走池）。同一水位在三处一致使用，形成闭环：

| 位置 | 代码 | 作用 |
|------|------|------|
| fetch 停止条件 | `if (purge_sys->iter.trx_no >= purge_sys->view.low_limit_no()) return nullptr;` | 追平最老 view → 本批结束 |
| history 截断 | `trx_purge_truncate_history` 中 `limit->trx_no = view->low_limit_no()` | 截断水位压到 purge view 边界 |
| 读 undo 双保险 | `trx_undo_get_undo_rec` 的 missing_history 判定 | 防读已被物理截断的 undo |

#### 长事务危害的源码级根源

1. **`m_ids` 膨胀**：长事务存活期间所有新 view 都要拷贝更大的活跃集，每次 `changes_visible` 的二分更慢。
2. **undo 无法 purge**：`get_oldest_view` 冻结 → `low_limit_no` 冻结 → fetch 提前返回、截断被压 → history list 增长、undo 表空间膨胀（`SHOW ENGINE INNODB STATUS` 的 history list length）。
3. 正常情况下 `DB_MISSING_HISTORY` 不会发生（FACT C 保证）；一旦出现即不变量被破坏。

elect_mode = SELECT_SKIP_LOCKED/SELECT_NOWAIT`（高优先级事务强制 ORDINARY）。

---

## 核心调用栈

```
-- 一致性读（快照读） --
row_search_mvcc (row0sel.cc)
  ├─ trx_assign_read_view → MVCC::view_open（fast/slow path）
  ├─ 聚簇：lock_clust_rec_cons_read_sees → changes_visible（三层漏斗）
  │    └─ false → row_sel_build_prev_vers_for_mysql
  │          → row_vers_build_for_consistent_read（循环回溯）
  │              → trx_undo_prev_version_build（undo 反向重放一步）
  │                  → trx_undo_get_undo_rec（purge 边界双保险）
  ├─ 二级：lock_sec_rec_cons_read_sees（PAGE_MAX_TRX_ID 粗筛）
  │    └─ false → ICP 过滤 → requires_clust_rec 回表 → row_sel_sec_rec_is_for_clust_rec 复核
  └─ delete-mark 可见版本 → next_rec 丢弃

-- 半一致性读（RC/RU + UPDATE，同一行两次往返） --
sql_update.cc → try_semi_consistent_read(true) → row_read_type = TRY
row_search_mvcc（第一遍：无锁试探）
  ├─ sel_set_rec_lock(SELECT_SKIP_LOCKED) → DB_SKIP_LOCKED（行被锁）
  ├─ row_sel_build_committed_vers_for_mysql → row_vers_build_for_semi_consistent_read
  ├─ old_vers == nullptr（行未提交）→ next_rec；delete-marked → 公共检查跳过
  └─ 返回 DID → SQL 层：不满足 WHERE → unlock_row（复位 TRY，不释放锁）
row_search_mvcc（第二遍：DID，普通模式悲观重读同一行）
  └─ sel_set_rec_lock 成功 或 DB_LOCK_WAIT → que_thr_stop 挂起等锁

-- purge 水位 --
trx_purge → trx_sys->mvcc->clone_oldest_view(&purge_sys->view)
  ├─ trx_purge_fetch_next_rec：iter.trx_no >= view.low_limit_no() → 停
  ├─ trx_purge_truncate_history：截断水位 = view.low_limit_no()
  └─ 版本回溯读 undo：trx_undo_get_undo_rec 的 missing_history 判定
```

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `transaction_isolation` | REPEATABLE-READ | Global/Session | 决定 view 生命周期与半一致性读（RC/RU 才启用） |

> `innodb_locks_unsafe_for_binlog` 在 8.0 已移除（源码零引用）；其 RC 下弱化 gap 锁的职责并入隔离级别语义本身。

### 状态变量

- `SHOW ENGINE INNODB STATUS` 的 history list length（undo 积压，受最老 ReadView 约束）。
- `Information_schema.INNODB_TRX` 的 `trx_started`——定位长事务（长 view 的持有者）。

---

## Misc

### 易混淆点

- **`m_low_limit_id` vs `m_low_limit_no`**：前者是 trx_id 量纲的可见性高水位（`>=` 不可见）；后者是 trx_no 量纲的 purge 边界（`<` 可清理）。名字像，量纲和用途完全不同。
- **`view_close` 的两套机制**：`close()`（`m_creator_trx_id = TRX_ID_MAX`）与 `m_closed` 标记是两回事；AC-NL-RO 走 `m_closed` + 指针 tag 的无锁软关闭，RW 事务提交走硬关闭（归还 `m_free`）。
- **一致性读 vs 当前读**：普通 SELECT（`LOCK_NONE`）是快照读；UPDATE/DELETE/FOR UPDATE 是当前读（读最新版 + 加锁），**不开 view**。半一致性读是锁定读的降级变体，读"最新已提交"（比快照读新）。
- **版本链不是链表**：没有"前版本指针"字段，是 `DB_ROLL_PTR → undo 记录（内嵌旧版本 sys 列）→ 旧版本 roll_ptr → …` 的接力，由 undo 反向重放实现。
- **`DB_MISSING_HISTORY` ≠ bug**：正常时它不应出现（FACT C 保证）；`row_vers_build_for_semi_consistent_read` 对它是"把当前版本当已提交"的优雅降级，一致性读则报错（表示不变量被破坏）。

**★★ 真名勘误全表**：

| 常见写法 | 8.0.39 真相 |
|---|---|
| `trx_sys_get_new_trx_id()` / `trx_sys_get_new_trx_no()` | **不存在**。真名 `trx_sys_allocate_trx_id()` / `trx_sys_allocate_trx_no()`（前者须持 `trx_sys_mutex`，后者须持 `serialisation_mutex`）；只读下一个值用 `trx_sys_get_next_trx_id_or_no()` |
| `trx_get_serialisation_max_trx_no()` | **不存在**（0 命中）。只有 `trx_get_serialisation_min_trx_no()`；"max" 侧只有 debug-only 的 `trx_sys->rw_max_trx_no` |
| `trx_rw_is_active_low()` | **不存在**。分片重构后只剩 `trx_rw_is_active(trx_id, do_ref_count)` |
| `ReadView::complete()` | **不存在**。是 `close()`（置 `TRX_ID_MAX`）与 `copy_complete()`（purge 克隆用） |
| `MVCC::release_view()` | **不存在**。回收接口是 `view_release()`（free 但不 close）与 `view_close()` |
| `changes_visible()` / `sees()` 在 `read0read.cc` | **错**。二者是 `read0types.h` 里的 **inline 头文件实现**（`sees()` 只有一行） |
| `check_trx_id_sanity()` 在 `read0read.cc` | **错**。定义在 **`trx0sys.cc:63`**（`ReadView::` 的静态方法却放在 trx0sys.cc） |
| `ReadView` 用 `std::vector` 存活跃事务 | **错**。是私有内部类 `ids_t`（手写 `m_ptr/m_size/m_reserved`），注释明说 *"similar to a std::vector but it is not a drop in replacement"* |
| `trx_sys->mvcc` 是值成员 | **错**。是 `MVCC *mvcc` **指针** |
| `m_views` 链表"最老的在头" | **反了**。`view_open` 用 `UT_LIST_ADD_FIRST` → **最新在头**；`get_oldest_view()` 从 `UT_LIST_GET_LAST` 往前找 |
| `trx_purge_t::clone_oldest_view()` / `get_oldest_view()` | **不存在**（是 `MVCC::` 的）。purge 侧只是调用方 `trx_sys->mvcc->clone_oldest_view(&purge_sys->view)` |
| `trx_purge_get_rseg_with_min_trx_no()` | **不存在**（5.6/5.7 老接口）。改用 `purge_sys->purge_queue` 最小堆 + `TrxUndoRsegsIterator::set_next()` |
| `purge_sys->next_stored` 是 `purge_iter_t` | **错**。是 **`bool`** |
| `purge_sys->view_active` 是被判断的开关 | **只有赋值、无读取点**。真正的互斥保护是 `purge_sys->latch` 的 x-latch |
| `srv_purge_wakeup` / `srv_do_purge` / `srv_purge_coordinator_thread` 里用 purge view | **错**。三者**完全不读** `purge_sys->view`，只看 `trx_sys->rseg_history_len` |
| 状态变量 `history_list_len` | **不存在**该名字。正名是 INNODB_METRICS 的 **`trx_rseg_history_len`** |
| `Innodb_purge_trx_id_age` / `purge_view_trx_id_age` 一直可用 | **只在 `UNIV_DEBUG` 下存在**（注册与计算都在 `#ifdef` 里） |
| `row_read_type` 是 enum | **错**。是 `ulint` 字段 + 三个 **`constexpr uint32_t`** |
| `new_rec_lock` 是 `bool[2]` | **错**。是 **`std::bitset<2>`**，配套 `new_rec_locks_count()` |
| `was_semi_consistent_read()` 会解锁 | **不会**。只是布尔查询；解锁在 `unlock_row()` 里，且对 DID 是**只复位不解锁** |
| `ROLL_PTR_*_MASK` / `ROLL_PTR_INSERT_FLAG` 常量 | **全部不存在**（0 命中）。位布局硬编码在 `trx_undo_build_roll_ptr()` / `trx_undo_decode_roll_ptr()` 的移位表达式里 |
| roll_ptr 是 64 位 / rseg id 占 8 bit | **56 bit** / **7 bit**（`& 0x7F`），且存的是 **undo tablespace number** 不是 rseg slot |
| `trx_undo_get_undo_rec_low_detail` | **不存在**。真名 `trx_undo_get_undo_rec_low()` + `trx_undo_get_undo_rec()` |
| 历史丢失报错是 "Snapshot too old" / ORA-01555 文本 | **0 命中**。文本是 **"Required history data has been deleted"** |
| `DATA_TRX_ID` / `DATA_ROLL_PTR` / `DATA_ROW_ID` 在 `dict0mem.h` | 在 **`data/data0type.h:177/182/188`**，且是 **prtype 编号 0/1/2**（mtype 恒为 `DATA_SYS`），不是 mtype |
| Def 1~5 形式化注释在 `row_vers_find_matching()` | 在 **`row_vers_impl_x_locked_low()` 函数体内**（`row0vers.cc:307-493`） |
| RU 是"不建 ReadView" | **不准确**。view 照样分配，只是 `row_search_mvcc` 的 else-if 链里显式跳过版本构建 |

**★ 三个"结构性事实"**：

1. **`trx->id` 与 `trx->no` 共用同一个原子计数器 `trx_sys->next_trx_id_or_no`**，只是分配时持有的 mutex 不同。这正是 `ut_a(m_low_limit_no <= m_low_limit_id)` 成立的前提，也是"两个序列各管一件事但量纲可比"的原因。

2. **`purge_sys->view` 的三个消费点构成闭环**：fetch 停止条件（`iter.trx_no >= low_limit_no`）→ history 截断上限（`limit` 兜底到 `low_limit_no`）→ 读 undo 前的 missing_history 判定（`view.changes_visible(trx_id)`）。任何一处漏掉都会导致 use-after-purge。

3. **"返回 true 但 `old_vers == nullptr`"有三种情形**，都被刻意伪装成"fresh insert"：① insert undo（链到头）② `table_id` 不匹配（表被重建）③ disowned-external 已被 purge。它们的共同目的是**让调用方不 assert**——所以 `row_vers_build_for_consistent_read` 里 `prev_version == nullptr` 与 `purge_sees == false` 必须分开处理（前者静默跳过、后者报 `DB_MISSING_HISTORY`）。

**★ 一个隐藏的工程细节**：`MVCC::view_open()` 的 fast path 与 purge 存在 Oracle 注释自认的固有竞态——

```cpp
    /* NOTE: This can be optimised further, for now we only reuse the view if
    there are no active RW transactions.

    There is an inherent race here between purge and this thread. Purge will
    skip views that are marked as closed. Therefore we must set the low limit
    id after we reset the closed status after the check. */
```

> ★★ 缓解方式：**先置 `m_closed = false`、再检查 `m_low_limit_id`、必要时回退**（把 `m_closed` 置回 true 落 slow path）。代价是 purge 可能暂时保守（多看到一个已 closed 的旧 view）。

### 长事务为什么是毒瘤（源码级总结）

1. `m_ids` 膨胀 → 所有新 view 拷贝变慢 + `changes_visible` 二分变慢；
2. `m_low_limit_no` 冻结 → purge 停滞 → history list 增长、undo 表空间膨胀；
3. 二级索引回表增多：长时间运行后 `PAGE_MAX_TRX_ID` 普遍大于老 view 的 `m_up_limit_id`，粗筛失效。

---

## 参考

- Bernstein, P. & Goodman, N. *Concurrency Control in Distributed Database Systems*. ACM Computing Surveys, 1981.（多版本并发控制经典形式化）
- Berenson, H. et al. *A Critique of ANSI SQL Isolation Levels*. SIGMOD, 1995.（快照隔离与各异常现象的定义）
- *MySQL 8.0 Reference Manual → 17.7.2.3 Consistent Nonlocking Reads*、*17.7.2.4 Locking Reads*（一致性读与锁定读的官方语义）
- 内核月报：[MySQL · 引擎特性 · InnoDB MVCC 相关实现](http://mysql.taobao.org/monthly/2018/11/04/)、[MySQL · 引擎特性 · InnoDB 事务系统](http://mysql.taobao.org/monthly/2017/12/01/)

> 注意：月报基于较早版本，其函数名/行号与 8.0.39 有出入；本文所有函数名以 8.0.39 源码为准。

**相关文档**

- undo 记录格式与 purge 执行流程见 [`undo_log.md`](undo_log.md)
- 事务对象 `trx_t`、提交协议（`trx_commit` 全时序）见 [`trx.md`](trx.md)
- 行锁与隐式锁转换的加锁侧见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- 取行主链（`row_search_mvcc` 全流程、回表、ICP）见 [`row_search.md`](row_search.md)
- 二级索引记录格式（为什么没有 `DB_TRX_ID`）见 [`physical/record.md`](physical/record.md)

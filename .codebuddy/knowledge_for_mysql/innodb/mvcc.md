# MVCC / 一致性读 深度解析

> 基于 MySQL 8.0.39 源码。涵盖 ReadView 结构与池化、可见性判定、undo 版本链回溯、半一致性读、purge 水位闭环、隔离级别全景。**边界**：undo 记录格式与 purge 执行流程见 [`undo_log.md`](undo_log.md)；事务对象 `trx_t` 与提交协议见 [`trx.md`](trx.md)；行锁（锁定读的加锁本身）见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)；取行主链见 [`row_search.md`](row_search.md)。本篇讲"读到哪个版本"的判定与实现。

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

#### 2. 二级索引页级粗筛：用"必要条件单侧判定"换掉回表

二级索引记录**不存储** `DB_TRX_ID`/`DB_ROLL_PTR`（省空间），单条记录无法精确判可见。InnoDB 的解法是页级粗筛：`sees(PAGE_MAX_TRX_ID)` 为 true 必可见（直接用）；为 false 只是"不确定"（回表精判）。其失真是**安全方向**的——`PAGE_MAX_TRX_ID` 不写 redo，崩溃恢复后可能偏大，只会多回表不会错读。

#### 3. trx_id 与 trx_no：两个序列各管一件事

| 序列 | 分配时机 | 用途 |
|------|---------|------|
| `trx->id` | 事务开始 | 写入记录的 `DB_TRX_ID`，只服务 MVCC 可见性 |
| `trx->no` | 提交时入 serialisation_list（提交序） | 只服务 purge 顺序与 `m_low_limit_no` |

ReadView 用 **id** 判可见性、用 **no** 定 purge 边界——`m_low_limit_id`/`m_up_limit_id`（id 语义）与 `m_low_limit_no`（no 语义）是两组不同量纲的字段，命名极易混淆。

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

### 隔离级别全景与 ReadView 生命周期

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
- **正确性依赖"不加 gap lock"**（`row0sel.cc` 注释）：gap lock 锁定范围，范围末尾行被删/purge 时半一致行为难证。因此严格限定在 `skip_gap_locks()` 为 true 的 RC/RU（`allow_semi_consistent() == skip_gap_locks()` 自洽）。

#### 加锁时机：两条路径

引擎层在 SQL 层 WHERE 过滤**之前**就尝试加锁：

- **路径 A（行未锁）**：`sel_set_rec_lock` 直接成功，返回 `DB_SUCCESS_LOCKED_REC`；RC/RU 下记 `new_rec_lock[...] = true`。SQL 层：WHERE 满足 → 保留锁更新；不满足 → `unlock_row` 释放这个新锁（`releases_non_matching_rows()` 仅 RC/RU 为 true）。
- **路径 B（行已被锁）**：以 `SELECT_SKIP_LOCKED` 模式请求，不排队直接返回 `DB_SKIP_LOCKED` → 构建旧版本、置 `DID`。**加锁被推迟到 WHERE 判定之后**：不满足 → 跳过；满足 → 重读该行，此时 `row_read_type == DID`（非 TRY），`use_semi_consistent = false`，改用 `prebuilt->select_mode` 普通模式悲观等待加锁。

结论：未锁行"读行即加锁"（WHERE 前）；已锁行才把加锁推迟到 WHERE 满足后。半一致性读省的是"已锁无关行"的等待与加锁。

#### 核心算法：`row_vers_build_for_semi_consistent_read` 逐行

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

### purge 与 ReadView 的水位闭环

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
- 行锁与隐式锁转换的加锁侧见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)
- 取行主链（`row_search_mvcc` 全流程、回表、ICP）见 [`row_search.md`](row_search.md)
- 二级索引记录格式（为什么没有 `DB_TRX_ID`）见 [`physical/record.md`](physical/record.md)

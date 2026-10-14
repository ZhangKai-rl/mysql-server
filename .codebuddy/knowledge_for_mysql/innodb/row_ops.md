# 行操作层：从 SQL 语句到 btr 调用的状态机

> 基于 MySQL 8.0.39 源码。本篇讲 **InnoDB 行操作层**（`row0ins`/`row0upd`/`row0umod`/`row0uins`）：que 查询图执行器如何以"每步一个状态"反复驱动 INSERT/UPDATE/DELETE、乐观失败→悲观重试的 mode 升级协议、**锁等待的断点续传**、undo 先于改页的编排、以及回滚的逆状态机。
>
> **边界**：btr 层的树函数逐行见 [`../index/btr.md`](../index/btr.md)（本篇只到"行层怎么调用它们"）；que 执行模型（fork/thr/node 的图结构）见 [`query_graph.md`](query_graph.md)；取行主链见 [`row_search.md`](row_search.md)。本篇是"SQL 语句 → que 状态机 → btr 调用"这条链的中间层。
>
> **字节编码不在此篇**：entry 模板里的 `dtuple_t`/`dfield_t`/`dtype_t` 表示与**写方向的逐列编码**（`row_mysql_convert_row_to_innobase` 循环、`row_mysql_store_col_in_innobase_format` 的 190 行逐分支）见 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md) 的「三、InnoDB 内存层」与「五、转换桥梁」——那篇是"行数据表示"的主文档，本篇只讲流程、不重复字节层。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [que 驱动协议与节点结构](#que-驱动协议与节点结构)
    - [字节编码层：为什么不在本篇](#字节编码层为什么不在本篇)
  - INSERT 状态机
    - [状态序列与 `row_ins`](#状态序列与-row_ins)
    - [乐观失败 → 悲观重试](#乐观失败--悲观重试)
      - [`DB_FAIL` 是什么](#db_fail-是什么)
      - [mode 枚举全文](#mode-枚举全文)
      - [optimistic_insert 返回 DB_FAIL 的 5 类原因](#optimistic_insert-返回-db_fail-的-5-类原因)
      - [UPDATE 侧：乐观 update 从不返回 DB_FAIL](#update-侧乐观-update-从不返回-db_fail)
    - [先聚簇后二级与部分完成处理](#先聚簇后二级与部分完成处理)
  - UPDATE 状态机
    - [状态转换图与三路径分派](#状态转换图与三路径分派)
    - [锁等待续传的幂等性](#锁等待续传的幂等性)
  - undo 与回滚
    - [undo 先于改页](#undo-先于改页)
    - [回滚：同一协议的逆状态机](#回滚同一协议的逆状态机)
  - 错误处理
    - [错误矩阵：怎么停、怎么恢复](#错误矩阵怎么停怎么恢复)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

行操作层是"SQL 语句"与"btr 树函数"之间的中间层：一次 INSERT/UPDATE/DELETE **不是一次函数调用**，而是由 que 执行器（`que_thr_step`）以"每步一个状态"反复驱动 `row_ins_step`/`row_upd_step`；节点内部维护 state 枚举，一次 `*_step` 只推进部分状态；遇到 `DB_LOCK_WAIT` 时**状态原样保留**（断点续传），锁授予后重入同一节点继续。

### 用途

- 理解"一条语句插入百万行"的执行模型（每行一个状态循环）；
- 理解锁等待时事务为什么不会重复工作（断点续传）；
- 理解二级索引插入失败后聚簇记录怎么清理（语句级 savepoint 回滚）。

### 版本演进

- que 状态机自 InnoDB 早期至今未变；8.0 增加多值索引的断点字段（`ins_multi_val_pos` 等）与 instant/虚拟列分支。

---

## 理论基础

### 设计思想与权衡

**1. 状态机而非递归调用**：行操作拆成状态（IX 锁 → 取行 → 逐索引插入），每个状态是原子的、可中断的。**唯一的中断点语义 = 锁等待**——中断时"不推进任何指针"（`node->index`/`node->entry`/`*_multi_val_pos` 不动），恢复时从断点继续，天然幂等。

**2. 乐观/悲观双路径是 btr 层的 mode 升级，不是状态机重试**：`BTR_MODIFY_LEAF` 失败 → `BTR_MODIFY_TREE` 重新下树，行层状态机对此无感知——**重试发生在树操作内部**（cursor 级），状态机只看到最终结果。

**3. 先聚簇、后二级 + 失败靠 savepoint 回滚**：二级失败时行层**不做任何补救**（不 delete-mark 聚簇记录），由语句级 `trx_rollback_to_savepoint` 走 undo 逆状态机清理——把"错误恢复"集中到一条路径。

**4. undo 先于改页**：`trx_undo_report_row_operation` 永远在页内物理修改之前写——崩溃/回滚才有完整旧值。

**★★ 第 1 条的源码出处**（`storage/innobase/que/que0que.cc:55`，★ 这是"为什么用图 + 状态机而不是递归"的权威注释）：

```cpp
/* Short introduction to query graphs
   ==================================

A query graph consists of nodes linked to each other in various ways. The
execution starts at que_run_threads() which takes a que_thr_t parameter.
que_thr_t contains two fields that control query graph execution: run_node
and prev_node. run_node is the next node to execute and prev_node is the
last node executed.

Each node has a pointer to a 'next' statement, i.e., its brother, and a
pointer to its parent node. The next pointer is NULL in the last statement
of a block.

Loop nodes contain a link to the first statement of the enclosed statement
list. While the loop runs, que_thr_step() checks if execution to the loop
node came from its parent or from one of the statement nodes in the loop. If
it came from the parent of the loop node it starts executing the first
statement node in the loop. If it came from one of the statement nodes in
the loop, then it checks if the statement node has another statement node
following it, and runs it if so.
```

> ★★ **关键推论**：节点只有 **parent + brother 两个指针**，`run_node`/`prev_node` **两个字段**就能完整刻画"执行点"。递归调用栈无法表达"从中间某一步返回并继续"，也**无法在等待锁时把栈丢弃**——这正是选状态机的理由。

**★ 第 2 条需要修正**（见「乐观失败 → 悲观重试」一节的实证）：mode 升级**确实发生在 btr/tree 层**，但 **`DB_FAIL` 的捕获点在 row 层**（`row_ins_clust_index_entry()` 两次调用 `row_ins_clust_index_entry_low()`），所以"行层状态机完全无感知"不准确——**row 层感知的是 `DB_FAIL` 这个返回值并负责发起第二次**。

**★ 第 4 条的一个补充代价**：MySQL 接口下**每次进入 `row_upd()` 都要重算 `cmpl_info`**（引擎内部图才有编译期 `cmpl_info`）：

```cpp
  if (UNIV_LIKELY(node->in_mysql_interface)) {
    /* We do not get the cmpl_info value from the MySQL
    interpreter: we must calculate it on the fly: */
    if (node->is_delete || row_upd_changes_some_index_ord_field_binary(
                               node->table, node->update)) {
      node->cmpl_info = 0;
    } else {
      node->cmpl_info = UPD_NODE_NO_ORD_CHANGE;
    }
  }
```

---

## 核心实现

### 主线与基础构件

#### que 驱动协议与节点结构

**★ 分派器真名是 `que_thr_step()`**（`que0que.cc:925`，`static inline`）——**不存在 `que_node_step()`**：

```cpp
static inline que_thr_t *que_thr_step(que_thr_t *thr) {
  trx = thr_get_trx(thr);
  ut_ad(thr->state == QUE_THR_RUNNING);
  ut_a(trx->error_state == DB_SUCCESS);

  thr->resource++;
  node = thr->run_node;
  type = que_node_get_type(node);
  old_thr = thr;

  if (type & QUE_NODE_CONTROL_STAT) {
    if ((thr->prev_node != que_node_get_parent(node)) &&
        que_node_get_next(thr->prev_node)) {
      /* The control statements, like WHILE, always pass the
      control to the next child statement if there is any
      child left */
      thr->run_node = que_node_get_next(thr->prev_node);
    } else if (type == QUE_NODE_IF) { if_step(thr); }
    ...
  } else if (type == QUE_NODE_ASSIGNMENT) { assign_step(thr); }
  else if (type == QUE_NODE_SELECT)   { thr = row_sel_step(thr); }
  else if (type == QUE_NODE_INSERT)   { thr = row_ins_step(thr); }
  else if (type == QUE_NODE_UPDATE)   { thr = row_upd_step(thr); }
  ...
  else if (type == QUE_NODE_UNDO)     { thr = row_undo_step(thr); }
  else if (type == QUE_NODE_PURGE)    { thr = row_purge_step(thr); }
  else if (type == QUE_NODE_ROLLBACK) { thr = trx_rollback_step(thr); }
  else { ut_error; }

  if (type == QUE_NODE_EXIT) {
    old_thr->prev_node = que_node_get_containing_loop_node(node);
  } else {
    old_thr->prev_node = node;
  }
  return (thr);
}
```

> ★ `thr = row_upd_step(thr)` / `row_ins_step(thr)` 就是行操作层被 que 调用的**唯一切入点**。step 函数**自己**决定 `thr->run_node` 改成什么（换 sel_node 取下一行，或换 parent 结束），返回 thr 或 **nullptr（本线程结束）**。

**★ 驱动循环：外层壳 `que_run_threads()` + 内层 `que_run_threads_low()`**

```cpp
/* que0que.cc:1081 —— 外层：处理锁等待 */
void que_run_threads(que_thr_t *thr) {
loop:
  ut_a(thr_get_trx(thr)->error_state == DB_SUCCESS);

  que_run_threads_low(thr);

  switch (thr->state) {
    case QUE_THR_RUNNING:
      /* There probably was a lock wait, but it already ended
      before we came here: continue running thr */
      goto loop;

    case QUE_THR_LOCK_WAIT:
      lock_wait_suspend_thread(thr);          /* ★★ OS 线程真挂起（不是协程） */
      trx_mutex_enter(thr_get_trx(thr));
      if (thr_get_trx(thr)->error_state != DB_SUCCESS) {
        /* thr was chosen as a deadlock victim or there was
        a lock wait timeout */
        que_thr_dec_refer_count(thr, nullptr);
        trx_mutex_exit(thr_get_trx(thr));
        break;
      }
      trx_mutex_exit(thr_get_trx(thr));
      goto loop;                              /* ★★ 醒来后重新进入 low */

    case QUE_THR_COMPLETED:
    case QUE_THR_COMMAND_WAIT:
      break;
    default:
      ut_error;
  }
}
```

```cpp
/* que0que.cc:1024 —— 内层：每步一个状态反复驱动 */
static void que_run_threads_low(que_thr_t *thr) {
  do {
    log_free_check();                 /* 每步都检查 redo 空间 */
    next_thr = que_thr_step(thr);     /* ★★ 一步 */
    trx_mutex_enter(trx);
    if (next_thr != thr) {
      ut_a(next_thr == nullptr);
      /* This can change next_thr to a non-NULL value
      if there was a lock wait that already completed. */
      que_thr_dec_refer_count(thr, &next_thr);
      if (next_thr != nullptr) thr = next_thr;
    }
    trx_mutex_exit(trx);
  } while (next_thr != nullptr);
}
```

> ★★★ **断点续传的完整机制**：`lock_wait_suspend_thread()` 挂起 OS 线程 → 唤醒后 `goto loop` 重入 `que_run_threads_low()` → 此时 **`thr->run_node` 与 `node->state` 都还是挂起前的原值**，于是从原节点重入。唤醒侧 `que_thr_end_lock_wait()`（`:267`）**只改 state，不碰 `run_node`/`prev_node`**。

**★ 对比："从头开始"用的是另一个函数**（`que0que.cc:352`，★ 这是"续传 vs 重启"的分水岭）：

```cpp
  for (auto thr : fork->thrs) {
    switch (thr->state) {
      case QUE_THR_COMMAND_WAIT:
        que_thr_init_command(thr);        /* ★ 重置 run_node/prev_node = 从头 */
        return (thr);

      case QUE_THR_SUSPENDED:
        /* In this case the execution of the thread was
        suspended: no initial message is needed because
        execution can continue from where it was left */
        if (!suspended_thr) suspended_thr = thr;   /* ★★ 只记下来 */
        break;
      ...
    }
  }
  if (suspended_thr) {
    thr = suspended_thr;
    que_thr_move_to_run_state(thr);       /* ★★ 只复位 state，不动 run_node = 续传 */
  }
```

```cpp
static inline void que_thr_init_command(que_thr_t *thr) {
  thr->run_node = thr;
  thr->prev_node = thr->common.parent;
  que_thr_move_to_run_state(thr);
}
```

协议要点：

- `thr->run_node` 指向当前节点；step 函数**返回下一个要运行的 thr**（可能跳转到 sel_node 取下一行、可能 `nullptr` 停止）；
- **重入判定**：`thr->prev_node == parent` 判断"从父节点新进入"还是"锁等待后重入"，只有新进入才复位 `node->state`（`row0ins.cc:3631`/`row0upd.cc:3324`）；
- 锁等待挂起：`que_thr_stop_for_mysql` 置 `QUE_THR_COMPLETED` 并递减 `n_active_thrs`；若 `error_state == DB_LOCK_WAIT` 则**不停止**。

**★ 节点类型的多态基础**（`que0types.h:52`）——`que_common_t` 必须是子类第一个成员：

```cpp
struct que_common_t {
  ulint type;          /*!< query node type */
  que_node_t *parent;  /*!< back pointer to parent node, or NULL */
  que_node_t *brother; /* pointer to a possible brother node */
  dfield_t val;        /*!< evaluated value for an expression */
  ulint val_buf_size;
};

/* que0que.ic:74 */
static inline ulint que_node_get_type(const que_node_t *node) {
  return (reinterpret_cast<const que_common_t *>(node)->type);   /* ★ C 版多态 */
}
```

**★ `que_thr_t` 完整定义**（`que0que.h:240`）：

```cpp
struct que_thr_t {
  que_common_t common;   /*!< type: QUE_NODE_THR */
  ulint magic_n;
  que_node_t *child;
  que_t *graph;
  que_thr_state_t state; /*!< state of the query thread */
  bool is_active;
  /*------------------------------*/
  /* The following fields are private to the OS thread executing the
  query thread, and are not protected by any mutex: */
  que_node_t *run_node;  /*!< pointer to the node where the subgraph down
                         from this node is currently executed */
  que_node_t *prev_node; /*!< pointer to the node from which the control came */
  ulint resource;
  ulint lock_state;
  struct srv_slot_t *slot;              /* 锁等待槽 */
  /*------------------------------*/
  UT_LIST_NODE_T(que_thr_t) thrs;
  UT_LIST_NODE_T(que_thr_t) trx_thrs;
  UT_LIST_NODE_T(que_thr_t) queue;
  ulint fk_cascade_depth;               /* ★ 外键级联递归深度 */
  row_prebuilt_t *prebuilt;
};
constexpr uint32_t QUE_THR_MAGIC_N = 8476583;
```

> ★★ **`run_node`/`prev_node` 无 mutex 保护**——注释明说"private to the OS thread"。这正是"挂起/恢复天然安全"的原因：只有持有该 OS 线程的人会读写它们。

**★ 唯一真 enum：`QUE_THR_*`**（`que0que.h:217`）：

```cpp
/** Query thread states */
enum que_thr_state_t {
  QUE_THR_RUNNING,
  QUE_THR_PROCEDURE_WAIT,
  /** in selects this means that the thread is at the end of its result set
  ... in other statements, this means the thread has done its task */
  QUE_THR_COMPLETED,
  QUE_THR_COMMAND_WAIT,
  QUE_THR_LOCK_WAIT,
  QUE_THR_SUSPENDED
};

/** Query thread lock states */
enum que_thr_lock_t { QUE_THR_LOCK_NOLOCK, QUE_THR_LOCK_ROW, QUE_THR_LOCK_TABLE };
```

**★ `QUE_NODE_*` 与 `UPD_NODE_*` / `INS_NODE_*` 都不是 enum**（`que0que.h:372`，★ 全是 `constexpr uint32_t`）：

```cpp
/* Flag which is ORed to control structure statement node types */
constexpr uint32_t QUE_NODE_CONTROL_STAT = 1024;
constexpr uint32_t QUE_NODE_LOCK = 1;
constexpr uint32_t QUE_NODE_INSERT = 2;
constexpr uint32_t QUE_NODE_UPDATE = 4;      /* ★ 没有 QUE_NODE_DELETE（3 空缺） */
constexpr uint32_t QUE_NODE_CURSOR = 5;
constexpr uint32_t QUE_NODE_SELECT = 6;
constexpr uint32_t QUE_NODE_FORK = 8;
constexpr uint32_t QUE_NODE_THR = 9;
constexpr uint32_t QUE_NODE_UNDO = 10;
constexpr uint32_t QUE_NODE_COMMIT = 11;
constexpr uint32_t QUE_NODE_ROLLBACK = 12;
constexpr uint32_t QUE_NODE_PURGE = 13;
constexpr uint32_t QUE_NODE_SYMBOL = 14;
constexpr uint32_t QUE_NODE_FUNC = 16;
constexpr uint32_t QUE_NODE_PROC  = 18 + QUE_NODE_CONTROL_STAT;
constexpr uint32_t QUE_NODE_IF    = 19 + QUE_NODE_CONTROL_STAT;
constexpr uint32_t QUE_NODE_WHILE = 20 + QUE_NODE_CONTROL_STAT;
constexpr uint32_t QUE_NODE_ASSIGNMENT = 21;
constexpr uint32_t QUE_NODE_FOR  = 25 + QUE_NODE_CONTROL_STAT;
constexpr uint32_t QUE_NODE_EXIT = 29;
```

> ★★ **DELETE 没有独立节点类型**——复用 `QUE_NODE_UPDATE`，靠 **`upd_node_t::is_delete`** 区分。

**★ `UPD_NODE_*` 状态与 `cmpl_info` 标志**（`row0upd.h:787`，★ 注意两套常量的命名空间不同）：

```cpp
/* Node execution states */
/** execution came to the node from a node above and if the field
has_clust_rec_x_lock is false, we should set an intention x-lock on the table */
constexpr uint32_t UPD_NODE_SET_IX_LOCK = 1;
/** clustered index record should be updated */
constexpr uint32_t UPD_NODE_UPDATE_CLUSTERED = 2;
/* clustered index record should be inserted, old record is already delete marked */
constexpr uint32_t UPD_NODE_INSERT_CLUSTERED = 3;
/** an ordering field of the clustered index record was changed, or this is a
delete operation: should update all the secondary index records */
constexpr uint32_t UPD_NODE_UPDATE_ALL_SEC = 5;
/** secondary index entries should be looked at and updated if an ordering field
changed */
constexpr uint32_t UPD_NODE_UPDATE_SOME_SEC = 6;

/* Compilation info flags: these must fit within 2 bits; see trx0rec.h */
constexpr uint32_t UPD_NODE_NO_ORD_CHANGE = 1;   /* ★ 与 UPDATE_CLUSTERED=2 语义无关，别混 */
constexpr uint32_t UPD_NODE_NO_SIZE_CHANGE = 2;
```

**★ `INS_NODE_*` 状态**（`row0ins.h:205`，★ 只有 3 个，比 UPD 少）：

```cpp
/* Node execution states */
/** we should set an IX lock on table */
constexpr uint32_t INS_NODE_SET_IX_LOCK = 1;
/** row id should be allocated */
constexpr uint32_t INS_NODE_ALLOC_ROW_ID = 2;
/** index entries should be built and inserted */
constexpr uint32_t INS_NODE_INSERT_ENTRIES = 3;
```

**`ins_node_t`**（`row0ins.h:162`）关键成员：`state`、`index`（**下一个要插入的索引**，断点载体）、`entry_list`（每索引一个 entry 模板）、`ins_multi_val_pos`（多值断点）。

**`upd_node_t`**（`row0upd.h:703`）关键成员：`state`、`pcur`（定位聚簇记录的持久游标）、`cmpl_info`、`del_multi_val_pos`/`upd_multi_val_pos`。

> ★ 两个节点都有 `*_multi_val_pos`——8.0 多值索引引入的断点字段：`/** When there is a lock wait error, this remembers current position of the multi-value field, before which the values have been inserted */`

#### 字节编码层：为什么不在本篇

`ins_node_t::entry_list` 里的 `dtuple_t` 承载着"每列的字节 + 类型"，但它的**表示与编码规则**不在本篇：

- `dtype_t` 的位域打包、`dfield_t` 的值语义（`len == UNIV_SQL_NULL` 哨兵、`ext` 标记）；
- 写方向的逐列编码 `row_mysql_store_col_in_innobase_format`（190 行逐分支：`DATA_INT` 的"从尾向前 + 仅 signed 时 `^= 128`"、CHAR 剥尾空格、true VARCHAR 的 1/2 字节长度头、BLOB 的引用布局）；
- INSERT 整行的入口 `row_mysql_convert_row_to_innobase`（dfield 直接指向 `record[0]`，不复制）。

⇒ 都在 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md) 的「三、InnoDB 内存层」与「五、转换桥梁」两章详剖。**分工判据**：本篇回答"下一步做什么"（状态/断点/entry 模板），data_repr 回答"每个字节是什么"（表示/编码/两方向对称）——读 INSERT 的每列编码请到 data_repr，本篇不重复。

---

### INSERT 状态机

#### 状态序列与 `row_ins`

状态序列：`INS_NODE_SET_IX_LOCK →（INS_SEARCHED 时先走 sel_node 取行）→ INS_NODE_ALLOC_ROW_ID → INS_NODE_INSERT_ENTRIES →（每索引一轮）→ ALLOC_ROW_ID（下一行）`。

`row_ins`（`row0ins.cc:3539`）主体：

```cpp
if (node->state == INS_NODE_ALLOC_ROW_ID) {
  row_ins_alloc_row_id_step(node);            // dict_sys_get_new_row_id（聚簇非唯一才有）
  node->index = node->table->first_index();   // 先聚簇
  node->entry = UT_LIST_GET_FIRST(node->entry_list);
  node->state = INS_NODE_INSERT_ENTRIES;
}
while (node->index != nullptr) {              // 断点：锁等待后从失败索引继续
  if (node->index->type != DICT_FTS) {
    err = row_ins_index_entry_step(node, thr);
    switch (err) {
      case DB_SUCCESS: break;
      case DB_DUPLICATE_KEY:
        thr_get_trx(thr)->error_state = DB_DUPLICATE_KEY;
        thr_get_trx(thr)->error_index = node->index;
        [[fallthrough]];
      default:
        return err;                           // 停在这里；index/entry 未推进 → 续传点
    }
  }
  node->index = node->index->next();          // 聚簇 → 各二级
  node->entry = UT_LIST_GET_NEXT(tuple_list, node->entry);
}
node->state = INS_NODE_ALLOC_ROW_ID;          // 整行完成，取下一行
```

**聚簇唯一冲突**（`row_ins_duplicate_error_in_clust`，`row0ins.cc:2140`）发生在**插入之前**（`btr_cur_optimistic_insert` 之前）：`low_match/up_match >= n_unique` 时对候选重复记录加 S 锁（REPLACE/ODKU 加 X）→ `row_ins_dupl_error_with_rec` 判定 → 冲突直接 `DB_DUPLICATE_KEY`，**不产生任何页修改**。注意它可以返回 `DB_LOCK_WAIT`（加锁等待）——此时聚簇记录也尚未插入。

#### 乐观失败 → 悲观重试

分派（`row_ins_index_entry`，`row0ins.cc:3304`）三分支（聚簇/多值/普通二级）。聚簇路径（`row_ins_clust_index_entry`，3075）：

```cpp
err = row_ins_clust_index_entry_low(flags, BTR_MODIFY_LEAF, ...);   // 乐观
if (err != DB_FAIL) return err;
err = row_ins_clust_index_entry_low(flags, BTR_MODIFY_TREE, ...);   // 悲观重下树
```

`_low` 里 `BTR_MODIFY_TREE` 模式先**再试一次乐观插入**、`DB_FAIL` 才转 `btr_cur_pessimistic_insert`（`row0ins.cc:2547-2553`）——**重试发生在树层，行层状态机无感知**。二级索引结构相同（3159）；R-tree 在 MBR 变化时直接把 mode 升为 `BTR_MODIFY_TREE`（2881-2896）。

**★ 完整的两次调用（外层真升级点，`row_ins_clust_index_entry()`，`row0ins.cc:3070`）**：

```cpp
  err = row_ins_clust_index_entry_low(flags, BTR_MODIFY_LEAF, index, n_uniq,
                                      entry, thr, dup_chk_only);      /* 第 1 次：LEAF */
  DEBUG_SYNC(thr_get_trx(thr)->mysql_thd, "after_row_ins_clust_index_entry_leaf");

  if (err != DB_FAIL) return err;                                     /* ★★ DB_FAIL 的唯一捕获点 */

  /* Try then pessimistic descent to the B-tree */
  if (!index->table->is_intrinsic()) log_free_check();
  else { ... index->last_sel_cur->invalid = true; }

  err = row_ins_clust_index_entry_low(flags, BTR_MODIFY_TREE, index, n_uniq,
                                      entry, thr, dup_chk_only);      /* 第 2 次：TREE */
  return err;
```

```cpp
/* row0ins.cc:2547 —— _low 里的 TREE 分支：仍先试乐观 */
    } else {
      if (buf_LRU_buf_pool_running_out()) { err = DB_LOCK_TABLE_FULL; goto err_exit; }
      DEBUG_SYNC_C("before_insert_pessimitic_row_ins_clust");
      err = btr_cur_optimistic_insert(flags, cursor, &offsets, &offsets_heap,
                                      entry, &insert_rec, &big_rec, thr, &mtr);
      if (err == DB_FAIL) {
        err = btr_cur_pessimistic_insert(flags, cursor, &offsets, &offsets_heap,
                                         entry, &insert_rec, &big_rec, thr, &mtr);
        ...
      }
    }
```

> ★★ 所以是**两层**：外层 row 层捕获 `DB_FAIL` 发起第二次（LEAF→TREE）；内层 TREE 分支**再试一次乐观**（树锁已拿，乐观若成就省掉分裂）才真悲观。上面"行层状态机无感知"的说法**不准确**——**row 层感知 `DB_FAIL` 并负责发起第二次**。

#### ★ `DB_FAIL` 是什么

```cpp
/* storage/innobase/include/db0err.h:205 */
  /* The following are partial failure codes */
  DB_FAIL = 1000,
  DB_OVERFLOW,
  DB_UNDERFLOW,
  DB_STRONG_FAIL,
```

> ★★★ **`DB_FAIL` 是"局部失败码"（partial failure code），语义是"请用更强的 latch mode 重试"，不是真错误**。它**必须**在 row 层被消化；若冒泡到 `row_mysql_handle_errors()`，那里**没有 `DB_FAIL` case**，会落到 `default:` → `ib::fatal "Unknown error code"`。

#### ★ mode 枚举全文

```cpp
/* storage/innobase/include/btr0btr.h:61 */
enum btr_latch_mode : size_t {
  /** Search a record on a leaf page and S-latch it. */
  BTR_SEARCH_LEAF = RW_S_LATCH,
  /** (Prepare to) modify a record on a leaf page and X-latch it. */
  BTR_MODIFY_LEAF = RW_X_LATCH,
  /** Obtain no latches. */
  BTR_NO_LATCHES = RW_NO_LATCH,
  /** Start modifying the entire B-tree. */
  BTR_MODIFY_TREE = 33,             /* ★ 独立枚举值，不是位值 */
  /** Continue modifying the entire B-tree. */
  BTR_CONT_MODIFY_TREE = 34,        /* ★ SMO 中途再下降，不重新加 index 锁 */
  BTR_SEARCH_PREV = 35,
  BTR_MODIFY_PREV = 36,
  BTR_SEARCH_TREE = 37,
  BTR_CONT_SEARCH_TREE = 38
};
```

> ⚠️ **`BTR_MODIFY_EXTERNAL`(262144) 不是与它们并列的 mode**，它是 OR 到 `BTR_MODIFY_LEAF` 上的 flag，`BTR_LATCH_MODE_WITHOUT_INTENTION()` 会剥掉它。

#### ★★ `btr_cur_optimistic_insert()` 返回 `DB_FAIL` 的 5 类原因

| # | 条件 | 源码位置 |
|---|---|---|
| ① | 压缩页 `zip_pad` 预测插入后会过满 | `btr0cur.cc:2914` |
| ② | 有垃圾且 `max_size < rec_size`（重组后仍放不下） | `:2943-2950` |
| ③ | 无垃圾且 `max_size < rec_size` | `:2957-2963` |
| ④ | 聚簇叶子页需为未来 update 预留空间（`dict_index_get_space_reserve`）且存在可分裂记录 | `:3016-3051` ★ 主动放弃 |
| ⑤ | `page_cur_tuple_insert()` 返回 NULL（压缩页 / intrinsic 表 / 重组后仍失败） | `:3016+` |

```cpp
  LIMIT_OPTIMISTIC_INSERT_DEBUG(page_get_n_recs(page), goto fail);   /* ★ debug 强制失败 */

  if (leaf && page_size.is_compressed() &&
      (page_get_data_size(page) + rec_size >=
       dict_index_zip_pad_optimal_page_size(index))) {
  fail:
    err = DB_FAIL;                                     /* ★★ DB_FAIL 的唯一赋值点 */
    /* prefetch siblings of the leaf for the pessimistic operation */
    if (page_is_leaf(page)) btr_cur_prefetch_siblings(block);  /* ★ 顺便预读左右兄弟 */
```

> ★ 注意 `DB_TOO_BIG_RECORD`（超大记录）**不是** `DB_FAIL` —— 它是真错误，会直接上报。
> ★ 乐观失败时**顺便预读左右兄弟页**（`btr_cur_prefetch_siblings`）——为接下来的悲观分裂做准备。

#### ★★★ UPDATE 侧：乐观 update **从不返回 `DB_FAIL`**

这是最容易搞错的一处。`btr_cur_optimistic_update()` 的失败返回值是 `{DB_OVERFLOW, DB_UNDERFLOW, DB_ZIP_OVERFLOW}`：

```cpp
  if (!row_upd_changes_field_size_or_external(index, *offsets, update)) {
    /* The simplest and the most common case: the update does not
    change the size of any field ... */
    return (btr_cur_update_in_place(...));            /* ★ 就地更新，永不 DB_FAIL */
  }
  if (rec_offs_any_extern(*offsets)) {
  any_extern:
    btr_cur_prefetch_siblings(block);
    return (DB_OVERFLOW);                             /* ★ 有外存字段 */
  }
  ...
    if (!btr_cur_update_alloc_zip(...)) return (DB_ZIP_OVERFLOW);   /* ★ 压缩页放不下 */
  if (new_rec_size >= REC_MAX_DATA_SIZE) { err = DB_OVERFLOW; goto func_exit; }
  if (UNIV_UNLIKELY(page_get_data_size(page) - old_rec_size + new_rec_size <
                    BTR_CUR_PAGE_COMPRESS_LIMIT(index))) {
    err = DB_UNDERFLOW;                               /* ★ 页面会太空 */
    goto func_exit;
  }
```

**★★ `DB_FAIL` 是在 insert-by-modify 路径里被"翻译"出来的**（`row0ins.cc:283`、`row0ins.cc:361`）：

```cpp
    err = btr_cur_optimistic_update(flags | BTR_KEEP_SYS_FLAG, cursor, offsets,
                                    &offsets_heap, update, 0, thr,
                                    thr_get_trx(thr)->id, mtr);
    switch (err) {
      case DB_OVERFLOW:
      case DB_UNDERFLOW:
      case DB_ZIP_OVERFLOW:
        err = DB_FAIL;                                /* ★★★ 三个"空间类"失败统一翻译成 DB_FAIL */
      default:
        break;
    }
```

> ⚠️ **不存在 `btr_cur_can_only_modify_third_level`**（8.0.39 全仓 0 命中）。
> ★ 而 `btr_cur_pessimistic_update()` 的**第一件事**就是再调一次 `btr_cur_optimistic_update(flags | BTR_KEEP_IBUF_BITMAP, ...)`，只有那三个码才继续往下走分裂重插入（`btr0cur.cc:3993`）。

#### ★ 悲观插入不再"局部失败"

```cpp
/** Performs an insert on a page of an index tree. ...
 @return DB_SUCCESS or error number */     /* ★ 返回值里没有 DB_FAIL */
dberr_t btr_cur_pessimistic_insert(...) {
  ...
  err = btr_cur_ins_lock_and_undo(flags, cursor, entry, thr, mtr, &inherit);
  if (err != DB_SUCCESS) return (err);
  ...
  if (dict_index_get_page(index) == btr_cur_get_block(cursor)->page.id.page_no()) {
    *rec = btr_root_raise_and_insert(flags, cursor, offsets, heap, entry, mtr);
  } else {
    *rec = btr_page_split_and_insert(flags, cursor, offsets, heap, entry, mtr);
  }
  if (!*rec) return DB_OUT_OF_FILE_SPACE;
```

#### 先聚簇后二级与部分完成处理

`node->index = table->first_index()`（链序：聚簇 → 各二级）。**二级失败后行层不做任何补救**——`row_ins_step` 置 `trx->error_state` 返回 `nullptr`，清理在 MySQL 接口层：`row_mysql_handle_errors`（`row0mysql.cc:676`）对 `DB_DUPLICATE_KEY`/`DB_TOO_BIG_RECORD` 调 `trx_rollback_to_savepoint(trx, savept)`（注释原文 "Roll back the latest, possibly incomplete insertion or update"），savepoint 是 `row_insert_for_mysql` 进入时打的。语句回滚经 undo 逆状态机把已插入的聚簇/二级记录逐条删掉。唯一例外：搜索时命中 `BTR_CUR_INSERT_TO_IBUF`（已缓冲进 change buffer）直接返回成功。

---

### UPDATE 状态机

#### 状态转换图与三路径分派

```
UPD_NODE_SET_IX_LOCK ──→ UPDATE_CLUSTERED ──┬─ is_delete: delete-mark 聚簇 → UPDATE_ALL_SEC
                                            ├─ NO_ORD_CHANGE: 原地 → UPDATE_SOME_SEC
                                            ├─ 排序字段变: delete-mark 旧 + 插新 → UPDATE_ALL_SEC
                                            └─ DB_LOCK_WAIT: state 停在原地（断点）
UPDATE_ALL_SEC / UPDATE_SOME_SEC ──→ 逐二级 row_upd_sec_step ──→ 复位 UPDATE_CLUSTERED
```

`row_upd_clust_step`（3044）职责：`pcur->restore_position(BTR_MODIFY_LEAF)` 恢复游标 → 补聚簇 X 锁（`lock_clust_rec_modify_check_and_lock`）→ 三路径分派（`row_upd_changes_ord_field_binary` 判定 + `cmpl_info`）：

```cpp
  if (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE) {        /* 路径 B：原地（粗判已确定序不变） */
    err = row_upd_clust_rec(flags, node, index, offsets, &heap, thr, &mtr);
    goto exit_func;
  }

  row_upd_store_row(node, trx->mysql_thd, ...);

  if (row_upd_changes_ord_field_binary(index, node->update, thr, node->row,
                                       node->ext, nullptr)) {
    /* Update causes an ordering field (ordering fields within
    the B-tree) of the clustered index record to change: perform
    the update by delete marking and inserting.

    TODO! What to do to the 'Halloween problem', where an update
    moves the record forward in index so that it is again
    updated when the cursor arrives there? Solution: the
    read operation must check the undo record undo number when
    choosing records to update. MySQL solves now the problem
    externally! */

    err = row_upd_clust_rec_by_insert(flags, node, index, thr, referenced, &mtr);  /* 路径 C */
    if (err != DB_SUCCESS) goto exit_func;
    node->state = UPD_NODE_UPDATE_ALL_SEC;
  } else {
    err = row_upd_clust_rec(flags, node, index, offsets, &heap, thr, &mtr);        /* 路径 B */
    if (err != DB_SUCCESS) goto exit_func;
    node->state = UPD_NODE_UPDATE_SOME_SEC;
  }
  node->index = index->next();
```

> ★★ 三条路径的实际判据（`is_delete` 在最前面，原文 3135 行）：
> ① `node->is_delete` → **只打删除标记**（`row_upd_del_mark_clust_rec`）→ `UPDATE_ALL_SEC`
> ② `cmpl_info & UPD_NODE_NO_ORD_CHANGE` → **原地更新**（这是 `row_upd()` 刚用 `row_upd_changes_some_index_ord_field_binary()` 做的**粗判**，只看被改的列在**聚簇索引**里是否 `ord_part`）
> ③ `row_upd_changes_ord_field_binary(聚簇索引, ...)` 为**真** → **delete-mark + insert**（**细判**，拿真实旧行逐字段比较）；为假 → 原地更新
>
> ★ ②与③的差别：②是"update vector 层面"的静态判断，③是"新旧行逐字段"的动态判断。**③为真才走删插**。
> ★ 路径 C 注释里的 **"MySQL solves now the problem externally!"** 就是 InnoDB 承认不解决 Halloween Problem 的原文（A 侧详解见 [`../server/query/07_optimize/19_set_operation.md`](../server/query/07_optimize/19_set_operation.md)）。

**★ 粗判函数**（`row0upd.cc:1709`）：

```cpp
bool row_upd_changes_some_index_ord_field_binary(const dict_table_t *table,
                                                 const upd_t *update) {
  index = table->first_index();
  for (i = 0; i < upd_get_n_fields(update); i++) {
    upd_field = upd_get_nth_field(update, i);
    if (upd_fld_is_virtual_col(upd_field)) {
      if (dict_table_get_nth_v_col(index->table, upd_field->field_no)->m_col.ord_part)
        return true;
    } else {
      if (index->get_field(upd_field->field_no)->col->ord_part) return true;
    }
  }
  return false;
}
```

**★★★ 路径 C 的断点续传（全篇最完整的样板）**（`row_upd_clust_rec_by_insert`，`row0upd.cc:2629`）：

```cpp
  switch (node->state) {
    default:
      ut_error;
    case UPD_NODE_INSERT_CLUSTERED:
      /* A lock wait occurred in row_ins_clust_index_entry() in
      the previous invocation of this function. */
      row_upd_clust_rec_by_insert_inherit(index, nullptr, nullptr, entry,
                                          node->update);
      break;                                    /* ★★ 跳过 delete-mark，只做 BLOB 继承 */
    case UPD_NODE_UPDATE_CLUSTERED:
      /* This is the first invocation of the function where
      we update the primary key. Delete-mark the old record ... */
      rec = btr_cur_get_rec(btr_cur);
      ...
      if (rec_get_deleted_flag(rec, rec_offs_comp(offsets))) {
        /* If the clustered index record is already delete
        marked, then we are here after a DB_LOCK_WAIT.
        Skip delete marking clustered index and disowning its blobs. */
        ut_ad(rec_get_trx_id(rec, index) == trx->id);
        goto check_fk;                          /* ★★ 幂等保护：已打标记就跳过 */
      }
      err = btr_cur_del_mark_set_clust_rec(...);
      ...
      if (rec_offs_any_extern(offsets)) { ... btr_ctx.disown_inherited_fields(...); }
    check_fk:
      if (referenced) { err = row_upd_check_references_constraints(...); }
  }

  mtr_commit(mtr);

  err = row_ins_clust_index_entry(index, entry, thr, false);
  node->state = UPD_NODE_INSERT_CLUSTERED;      /* ★★★ insert 成功【之后】才改 state */
```

> ★★★ **这条是"断点续传"最完整的样板**，三重保险：
> ① 首轮（`UPD_NODE_UPDATE_CLUSTERED`）打 delete-mark；
> ② `row_ins_clust_index_entry()` **返回之后**才把 state 改成 `UPD_NODE_INSERT_CLUSTERED`——**即 insert 成功前 state 仍是 UPDATE_CLUSTERED**；
> ③ 若 insert 时锁等待，重入时 state 已是 INSERT_CLUSTERED → 走 `case UPD_NODE_INSERT_CLUSTERED` **跳过 delete-mark**；
> ④ 即便 state 还没改（锁等待发生在 delete-mark 内部），`rec_get_deleted_flag` 的幂等判断 `goto check_fk` 也能兜住。

原地路径（`row_upd_clust_rec`，2839）：`NO_SIZE_CHANGE` → `btr_cur_update_in_place`；否则 `btr_cur_optimistic_update`；失败**不是 DB_FAIL 重试两次**，而是直接 `restore_position(BTR_MODIFY_TREE)` + `btr_cur_pessimistic_update(flags | BTR_KEEP_POS_FLAG)`——X 锁已持有。

```cpp
  if (node->cmpl_info & UPD_NODE_NO_SIZE_CHANGE) {
    err = btr_cur_update_in_place(flags | BTR_NO_LOCKING_FLAG, btr_cur, offsets,
                                  node->update, node->cmpl_info, thr,
                                  thr_get_trx(thr)->id, mtr);
  } else {
    err = btr_cur_optimistic_update(flags | BTR_NO_LOCKING_FLAG, btr_cur, &offsets,
                                    offsets_heap, node->update, node->cmpl_info,
                                    thr, thr_get_trx(thr)->id, mtr);
  }
  if (err == DB_SUCCESS) goto success;

  mtr->commit();                                /* ★★ 必须先提交旧 mtr */
  if (buf_LRU_buf_pool_running_out()) { err = DB_LOCK_TABLE_FULL; goto func_exit; }
  mtr->start();
  ut_a(pcur->restore_position(BTR_MODIFY_TREE, mtr, UT_LOCATION_HERE));  /* ★★ mode 升级真身 */
  err = btr_cur_pessimistic_update(flags | BTR_NO_LOCKING_FLAG | BTR_KEEP_POS_FLAG, ...);
```

> ★★ **乐观与悲观是两个独立 mtr**——中间必须 `mtr->commit()`。因为悲观需要重新下降整棵树拿 index latch，而乐观只持有叶子页 latch。注释解释了为什么 `restore_position` 必然成功："this transaction has an s-lock or x-lock on the record and therefore other transactions cannot modify the record"。

`row_upd_sec_step`（2520）：`state == UPDATE_ALL_SEC || row_upd_changes_ord_field_binary(该索引)` 才更新该二级；否则跳过。二级更新 = delete-mark 旧项 + 插新项（`btr_cur_del_mark_set_sec_rec` + `row_ins_sec_index_entry`），**不写 undo**。

#### 锁等待续传的幂等性

MySQL 接口入口（`row_update_for_mysql`，`row0mysql.cc:2382-2413`）：

```cpp
run_again:
  row_upd_step(thr);
  err = trx->error_state;
  if (err != DB_SUCCESS) {
    que_thr_stop_for_mysql(thr);
    auto was_lock_wait = row_mysql_handle_errors(&err, trx, thr, &savept);
    if (was_lock_wait) { goto run_again; }   // 锁授予后原样重入
  }
```

**断点载体** = `node->state`/`node->index`/`node->entry`/`*_multi_val_pos` 全部保留原值。删插路径的幂等（`row_upd_clust_rec_by_insert`，2635）：`state == INSERT_CLUSTERED` + 记录已带 delete-mark + `rec_get_trx_id == trx->id` 时**跳过 delete-mark**（2688-2697）——锁等待重入不重复操作。BLOB 继承（`BTR_EXTERN_INHERITED_FLAG`）防回滚误删 off-page 数据。

---

### undo 与回滚

#### undo 先于改页

- **INSERT 聚簇**：`btr_cur_ins_lock_and_undo`（`btr0cur.cc:2778`）在 `page_cur_insert_rec_*` 之前写 `TRX_UNDO_INSERT_OP`；二级索引提前返回（`!index->is_clustered()`）**不写 undo**；
- **UPDATE 原地/悲观**：`btr_cur_upd_lock_and_undo`（3292）写 `TRX_UNDO_MODIFY_OP` 后才改记录；
- **DELETE/主键变更**：`btr_cur_del_mark_set_clust_rec`（4489）写 `TRX_UNDO_MODIFY_OP` 后才 `btr_rec_set_deleted_flag`；
- `trx_undo_report_row_operation`（`trx0rec.cc:2135`）返回 `roll_ptr` 写回记录 `DATA_ROLL_PTR` 字段。

#### 回滚：同一协议的逆状态机

`undo_node_t` 与正向状态机同构（`state` = `UNDO_NODE_FETCH_NEXT/INSERT/MODIFY`、`index` 断点、`update` 逆 update vector）。

**INSERT 回滚**（`row_undo_ins`，`row0uins.cc:466`）：**先删各二级条目、再删聚簇记录**（与正向相反）：

```cpp
node->index = node->table->first_index();
node->index = node->index->next();             // 跳过聚簇
err = row_undo_ins_remove_sec_rec(node, thr);  // 先二级
if (err == DB_SUCCESS) err = row_undo_ins_remove_clust_rec(node);  // 后聚簇
```

**UPDATE/DELETE 回滚**（`row_undo_mod`，`row0umod.cc:1272`）按 `rec_type` 三分支（`TRX_UNDO_UPD_EXIST_REC/UPD_DEL_REC/DEL_MARK_REC`）先回滚二级、再 `row_undo_mod_clust` 恢复聚簇（`UPD_DEL_REC` 还会重插聚簇记录）。

**★ `undo_node_t` 与三态枚举**（`row0undo.h:75`，★ 回滚状态机只有 3 个状态，比 INSERT/UPDATE 简单）：

```cpp
/** Execution state of an undo node */
enum undo_exec {
  UNDO_NODE_FETCH_NEXT = 1, /*!< we should fetch the next undo log record */
  UNDO_NODE_INSERT,         /*!< undo a fresh insert of a row to a table */
  UNDO_NODE_MODIFY          /*!< undo a modify operation (DELETE or UPDATE) */
};

struct undo_node_t {
  que_common_t common;      /*!< node type: QUE_NODE_UNDO */
  enum undo_exec state;
  trx_t *trx;
  roll_ptr_t roll_ptr;
  trx_undo_rec_t *undo_rec;
  undo_no_t undo_no;
  ulint rec_type;           /*!< TRX_UNDO_INSERT_REC, ... */
  trx_id_t new_trx_id;
  btr_pcur_t pcur;
  dict_table_t *table;
  ulint cmpl_info;
  upd_t *update;            /*!< update vector for a clustered index record */
  dtuple_t *ref;
  dtuple_t *row;
  dtuple_t *undo_row;
  dtuple_t *undo_ext;
  dict_index_t *index;      /*!< the next index whose record should be handled */
  mem_heap_t *heap;
  bool partial;             /*!< true if partial rollback */
};
```

**★★ 逆状态机的驱动：`row_undo()`**（`row0undo.cc:244`，★ "逆着弹栈"就是逆状态机的时间轴）：

```cpp
  if (node->state == UNDO_NODE_FETCH_NEXT) {
    node->undo_rec = trx_roll_pop_top_rec_of_trx(trx, trx->roll_limit,
                                                 &roll_ptr, node->heap);
    /* ★★ 从 undo 链顶（最近的修改）往回弹 */
    if (!node->undo_rec) {
      /* Rollback completed for this query thread */
      thr->run_node = que_node_get_parent(node);
      trx->roll_limit = 0;
      return (DB_SUCCESS);
    }
    node->roll_ptr = roll_ptr;
    node->undo_no = trx_undo_rec_get_undo_no(node->undo_rec);

    if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
      node->state = UNDO_NODE_INSERT;        /* ★ 迁移 1 */
    } else {
      node->state = UNDO_NODE_MODIFY;        /* ★ 迁移 2 */
    }
  }

  if (node->state == UNDO_NODE_INSERT) {
    err = row_undo_ins(node, thr);
    node->state = UNDO_NODE_FETCH_NEXT;      /* ★ insert undo 一条做完 → 立刻回 FETCH_NEXT */
  } else {
    ut_ad(node->state == UNDO_NODE_MODIFY);
    err = row_undo_mod(node, thr);           /* ★ MODIFY→FETCH_NEXT 在 row_undo_mod_clust() 里做 */
  }

  node->pcur.close();
  mem_heap_empty(node->heap);
  thr->run_node = node;                      /* ★★ 断点续传证据：run_node 指回自己 */
```

**★ 回滚也是 `que_run_threads` 驱动**（`trx0roll.cc:78`）——这就是"同一协议"：

```cpp
  roll_node = roll_node_create(heap);        /* ★ 回滚同样建一棵 que graph */
  if (savept != nullptr) { roll_node->partial = true; roll_node->savept = *savept; }
  trx->error_state = DB_SUCCESS;
  if (trx_is_rseg_updated(trx)) {
    thr = pars_complete_graph_for_exec(roll_node, trx, heap, nullptr);
    ut_a(thr == que_fork_start_command(...));
    que_run_threads(thr);                    /* ★★ 与 INSERT/UPDATE 完全一样 */
    ut_a(roll_node->undo_thr != nullptr);
    que_run_threads(roll_node->undo_thr);
    que_graph_free(...);
  }
```

**★★ 回滚内部有同一套 LEAF→TREE 升级**（`row_undo_ins_remove_clust_rec()`，`row0uins.cc:65`）：

```cpp
  auto success = node->pcur.restore_position(
      online ? BTR_MODIFY_LEAF | BTR_ALREADY_S_LATCHED : BTR_MODIFY_LEAF, &mtr, ...);
  ...
  if (btr_cur_optimistic_delete(btr_cur, 0, &mtr)) { err = DB_SUCCESS; goto func_exit; }

  node->pcur.commit_specify_mtr(&mtr);
retry:
  mtr_start(&mtr);
  success = node->pcur.restore_position(BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE,
                                        &mtr, UT_LOCATION_HERE);   /* ★★★ 同一协议 */
  btr_cur_pessimistic_delete(&err, false, btr_cur, 0, true, node->trx->id,
                             node->undo_no, node->rec_type, &mtr, &node->pcur, nullptr);
  if (err == DB_OUT_OF_FILE_SPACE && n_tries < BTR_CUR_RETRY_DELETE_N_TIMES) {
    node->pcur.commit_specify_mtr(&mtr);
    n_tries++;
    std::this_thread::sleep_for(std::chrono::milliseconds(BTR_CUR_RETRY_SLEEP_TIME_MS));
    goto retry;                              /* ★★ 回滚特有的"空间不足重试"（正向没有） */
  }
```

> ★ `row_undo_mod_clust()` 同构（`row0umod.cc:257`）：`row_undo_mod_clust_low(..., BTR_MODIFY_LEAF)` 失败 → `mtr_start()` 重来 → `row_undo_mod_clust_low(..., BTR_MODIFY_TREE)`。
> ★ 二级索引回滚也消化 `DB_FAIL`（`row_undo_mod_del_mark_sec`，`row0umod.cc:937`）：`if (err == DB_FAIL) err = row_undo_mod_del_unmark_sec_and_undo_update(BTR_MODIFY_TREE, ...)`。

**★★★ 回滚出错 = 崩溃**（`row_undo_step()`，`row0undo.cc:344`，★ 没有"优雅降级"）：

```cpp
  err = row_undo(node, thr);
  trx->error_state = err;
  if (err != DB_SUCCESS) {
    /* SQL error detected */
    if (err == DB_OUT_OF_FILE_SPACE) {
      ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_1041)
          << "Out of tablespace during rollback."
             " Consider increasing your tablespace.";
    }
    ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_1042)
        << "Error (" << ut_strerr(err) << ") in rollback.";
  }
```

> ★★ 这与正向操作截然相反：正向出错返回错误码交给 MySQL 层；**回滚出错直接 `ib::fatal` 崩溃**。因为回滚不能失败——失败就意味着事务原子性被破坏。

---

### 错误处理

#### 错误矩阵：怎么停、怎么恢复

| 错误 | 状态机如何停 | 如何恢复 |
|---|---|---|
| `DB_LOCK_WAIT` | 原样返回；节点状态与 index/entry/multi_val_pos **不推进**（断点） | `lock_wait_suspend_thread` 挂起，锁授予后 `goto run_again` 重入 |
| `DB_DUPLICATE_KEY` | 聚簇：插入前查重直接返回；二级：查重返回或插入后冲突 | `trx_rollback_to_savepoint` 回滚本语句（已插入部分经 undo 删除）；REPLACE/ODKU 上层走 X 锁路径 |
| `DB_TOO_BIG_RECORD` | 停止，部分索引可能已插入 | 同 savepoint 回滚 |
| `DB_DEADLOCK`/`DB_LOCK_TABLE_FULL` | 停止 | `trx_rollback_to_savepoint(trx, nullptr)` 整事务回滚 |
| `DB_RECORD_NOT_FOUND`（update） | 游标恢复失败 | 置回 `DB_SUCCESS`（行被本事务删了） |
| `DB_OUT_OF_FILE_SPACE`（undo 删除） | `BTR_CUR_RETRY_DELETE_N_TIMES` 次睡 `BTR_CUR_RETRY_SLEEP_TIME_MS` 重试 | 重试耗尽才报错 |

**resume 机制的本质**：`lock_wait_suspend_thread(thr)` 挂起后锁授予将 thr 置回 `QUE_THR_RUNNING`，而 `node->state`/`node->index`/`node->entry`/`*_multi_val_pos` 全部保留——精确断点续传。MySQL 接口侧不建完整 graph，而是**复用 ins_graph/upd_graph 里的单节点 thr** 手动循环 `row_ins_step`——这就是"SQL 语句 → que 状态机"的桥。

**★★ 集中处理点：`row_mysql_handle_errors()`**（`row0mysql.cc:676`，★ 这是"怎么停、怎么恢复"的真身）：

```cpp
static bool row_mysql_handle_errors(dberr_t *new_err, trx_t *trx,
                                    que_thr_t *thr, trx_savept_t *savept) {
  dberr_t err;
  bool was_lock_wait = false;
  THD *thd;
handle_new_error:
  err = trx->error_state;
  *new_err = err;
  thd = trx->mysql_thd;

  if (err == DB_LOCK_WAIT_TIMEOUT) {
    if (row_rollback_on_timeout) {                    /* ★ 受 innodb_rollback_on_timeout 控制 */
      trx_rollback_to_savepoint(trx, nullptr);        /* ★★ nullptr = 整事务回滚 */
    } else {
      /* Roll back the latest, possibly incomplete insertion or update */
      trx_rollback_to_savepoint(trx, *savept);        /* ★ 只回滚到语句 savepoint */
    }
    ...
  } else if (err == DB_DUPLICATE_KEY ||
             err == DB_FOREIGN_DUPLICATE_KEY ||
             err == DB_TOO_BIG_RECORD ||
             err == DB_UNDO_RECORD_TOO_BIG ||
             err == DB_CANNOT_ADD_CONSTRAINT ||
             err == DB_CANNOT_DROP_CONSTRAINT ||
             err == DB_NO_REFERENCED_ROW ||
             err == DB_ROW_IS_REFERENCED ||
             err == DB_NO_FK_ON_S_BASE_COL ||
             err == DB_CANT_CREATE_GEOMETRY_OBJECT ||
             err == DB_TOO_MANY_FIELDS ||
             err == DB_CORRUPTION ||
             err == DB_CANNOT_CREATE_GEOMETRY_OBJECT ||
             err == DB_LOCK_TABLE_FULL ||
             err == DB_TOO_MANY_CONCURRENT_TRXS ||
             err == DB_INDEX_CORRUPT ||
             err == DB_DATA_MISMATCH ||
             err == DB_ONLINE_LOG_TOO_BIG ||
             err == DB_MISSING_HISTORY ||
             err == DB_CLUSTER_NOT_FOUND ||
             err == DB_DEADLOCK) {
    /* Roll back the latest, possibly incomplete insertion or update */
    trx_rollback_to_savepoint(trx, *savept);          /* ★ 语句级回滚 */
  } else {
    ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_975) << "Unknown error code";  /* ★★ default */
  }
```

> ★★★ **`DB_FAIL` 不在上面的列表里** —— 它若冒泡到这里会落进 `default:` → `ib::fatal` 崩溃。所以 **`DB_FAIL` 必须在 row 层消化完**。

**★ 死锁 vs 锁等待超时：回滚范围不同**（★ 常被搞混）：

```cpp
/* 死锁（row0mysql.cc:747 附近） */
    case DB_DEADLOCK:
      /* Roll back the whole transaction; this will also reset
      the trx->error_state */
      trx_rollback_to_savepoint(trx, nullptr);        /* ★★ 整事务 */
```

```cpp
/* 锁等待超时（受开关控制，见上） */
  if (err == DB_LOCK_WAIT_TIMEOUT) {
    if (row_rollback_on_timeout) {
      trx_rollback_to_savepoint(trx, nullptr);        /* 整事务（非默认） */
    } else {
      trx_rollback_to_savepoint(trx, *savept);        /* 仅本语句（默认） */
    }
```

> ★★ 所以**默认配置下：死锁回滚整事务，锁等待超时只回滚本语句**。这是二者最重要的行为差异。

**★ 挂起点 `lock_wait_suspend_thread()`**（`lock0wait.cc:199`）——它**不重试**，只挂起：

```cpp
  const auto lock_wait_timeout = trx_lock_wait_timeout_get(trx);
  ...
  if (thr->state == QUE_THR_RUNNING) {
    /* The lock has already been released or this transaction
    was chosen as a deadlock victim: no need to suspend */
    if (trx->lock.was_chosen_as_deadlock_victim) {
      trx->error_state = DB_DEADLOCK;
      trx->lock.was_chosen_as_deadlock_victim = false;
    }
    return;                                          /* ★ 锁已授予 → 直接返回，不睡 */
  }
  slot = lock_wait_table_reserve_slot(thr, lock_wait_timeout);
  ...
  os_event_wait(slot->event);                        /* ★★ 真正的挂起 */
  ...
  lock_wait_table_release_slot(slot);
  if (trx->error_state == DB_DEADLOCK) return;
  if (trx->error_state == DB_LOCK_WAIT_TIMEOUT) MONITOR_INC(MONITOR_TIMEOUT);
  if (trx_is_interrupted(trx)) trx->error_state = DB_INTERRUPTED;
```

> ★★ **重试由调用方完成**，不在本函数内部：

```cpp
/* row0mysql.cc:1630 —— MySQL 接口的 mini 驱动器 */
  auto was_lock_wait = row_mysql_handle_errors(&err, trx, thr, &savept);
  ...
  if (was_lock_wait) goto run_again;                 /* ★★ 重入 row_ins_step / row_upd_step */
```

```cpp
/* row0mysql.cc:1595 —— INSERT 的 mini 驱动器（不经过 que_run_threads） */
run_again:
  thr->run_node = node;
  thr->prev_node = node;
  row_ins_step(thr);
```

**★ 唤醒侧**（`que0que.cc:267`）：

```cpp
que_thr_t *que_thr_end_lock_wait(trx_t *trx) {
  que_thr_t *const thr = trx->lock.wait_thr;
  bool const was_active = thr->is_active;
  que_thr_move_to_run_state(thr);          /* ★ 只改 state */
  trx->lock.que_state = TRX_QUE_RUNNING;
  trx->lock.wait_thr = nullptr;
  return !was_active ? thr : nullptr;      /* ★ 返回非 null 则由 OS 线程继续跑 */
}
```

**★ `DB_RECORD_NOT_FOUND` 的特殊处理**（★ 它是"本事务删过"的信号，不算错）：

```cpp
/* row_update_for_mysql，row0mysql.cc:2389 */
  if (err == DB_RECORD_NOT_FOUND) {
    trx->error_state = DB_SUCCESS;         /* ★★ 降级为成功 */
    trx->op_info = "";
    if (thr->fk_cascade_depth > 0) que_graph_free_recursive(node);
    goto error;
  }
```

**★ `que_thr_stop()` 决定"停在哪一个状态"**（`que0que.cc:676`）：

```cpp
bool que_thr_stop(que_thr_t *thr) {
  if (graph->state == QUE_FORK_COMMAND_WAIT) thr->state = QUE_THR_SUSPENDED;
  else if (trx->lock.que_state == TRX_QUE_LOCK_WAIT) {
    trx->lock.wait_thr = thr;
    thr->state = QUE_THR_LOCK_WAIT;                  /* ★ 锁等待 */
  } else if (trx->error_state != DB_SUCCESS &&
             trx->error_state != DB_LOCK_WAIT) {
    thr->state = QUE_THR_COMPLETED;                  /* ★ 真错误 → 结束 */
  } else if (graph->fork_type == QUE_FORK_ROLLBACK) thr->state = QUE_THR_SUSPENDED;
  else { ut_ad(graph->state == QUE_FORK_ACTIVE); return false; }
  return true;
}
```

> ★ 注意 `trx->error_state != DB_SUCCESS && != DB_LOCK_WAIT` 这条——**`DB_LOCK_WAIT` 不算"错误"，线程不被置为 COMPLETED**，这正是它能续传的原因。

**★ 相关错误码一览**：

| 码 | MySQL 侧 | 回滚范围 |
|---|---|---|
| `DB_DUPLICATE_KEY` | `ER_DUP_ENTRY` | 语句（`*savept`） |
| `DB_FOREIGN_DUPLICATE_KEY` / `DB_NO_REFERENCED_ROW` / `DB_ROW_IS_REFERENCED` | `ER_NO_REFERENCED_ROW_2` 等 | 语句 |
| `DB_TOO_BIG_RECORD` / `DB_UNDO_RECORD_TOO_BIG` / `DB_ONLINE_LOG_TOO_BIG` | `ER_TOO_BIG_ROWSIZE` 等 | 语句 |
| `DB_LOCK_WAIT_TIMEOUT` | `ER_LOCK_WAIT_TIMEOUT` | **语句**（默认）/ **整事务**（`innodb_rollback_on_timeout=ON`） |
| `DB_DEADLOCK` | `ER_LOCK_DEADLOCK` | **整事务** |
| `DB_LOCK_TABLE_FULL` | `ER_LOCK_TABLE_FULL` | 语句 |
| `DB_OUT_OF_FILE_SPACE`（回滚中） | — | **`ib::fatal` 崩溃** |
| `DB_RECORD_NOT_FOUND`（update） | — | 降级为 `DB_SUCCESS` |
| `DB_FAIL` | — | **必须在 row 层消化**，冒泡即 `ib::fatal` |

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `innodb_force_recovery` | 回滚禁用等容错级别 |
| `BTR_CUR_RETRY_DELETE_N_TIMES` / `BTR_CUR_RETRY_SLEEP_TIME_MS` | 删除重试次数/间隔 |
| `trx->error_index` | 冲突索引（报错用） |
| `trx->error_state` | 状态机错误出口 |
| `node->state` / `node->index` / `node->entry` | 断点续传载体 |

---

## Misc

### 易混淆点

- **乐观失败 → 悲观重试是 btr 层的 mode 升级（cursor 级重试），不是行状态机的状态回退**——状态机只看到最终结果。
- **二级失败后行层不补救**——清理靠语句级 savepoint 回滚（undo 逆状态机），"错误恢复"集中一条路径。
- **锁等待断点的载体是指针不推进**（`node->index`/`entry`/`*_multi_val_pos`），重入天然幂等；删插路径另有 delete-mark + trx_id 双检。
- **undo 永远先于改页**（三个调用点全在页修改前）；二级索引不写 undo。
- **回滚与正向顺序相反**：插入正向"先聚簇后二级"，回滚"先删二级再删聚簇"。
- **聚簇唯一查重发生在插入前**，冲突不产生任何页修改；但加锁等待可以在此发生（INSERT 的锁等待断点之一）。

### 三层视角总图

```
SQL 语句（row_insert_for_mysql 的 run_again 循环）
  → que 状态机（row_ins_step/row_upd_step，每步一个状态，锁等待断点续传）
      → 行操作（row_ins/row_upd：先聚簇后二级、查重、三路径分派）
          → btr 函数（btr_cur_optimistic/pessimistic_insert/update/delete）
              → 页操作（page_cur_tuple_insert 等）
回滚 = 同一协议的逆状态机（row_undo_ins/row_undo_mod，顺序相反）
```

### 一句话总结

行操作层的本质是**"可中断的逐步推进"**：que 执行器以状态机协议反复驱动行节点，锁等待是唯一的中断点（断点 = 指针不推进），乐观→悲观重试藏在 btr 层的 mode 升级里，undo 先于改页、回滚是逆状态机——理解它的关键就是**"状态 + 断点指针"两个词**。

---

## 参考

**相关文档**
- btr 层树函数逐行：[`../index/btr.md`](../index/btr.md)
- que 执行模型（fork/thr/node）：[`query_graph.md`](query_graph.md)
- 取行主链：[`row_search.md`](row_search.md)
- undo 记录格式与版本重建：[`mvcc.md`](mvcc.md)、[`undo_log.md`](undo_log.md)
- 事务提交/回滚整体：[`trx.md`](trx.md)
- 锁等待与唤醒：[`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- 唯一性检查（行层查重的锁细节）：[`../index/constraint.md`](../index/constraint.md)

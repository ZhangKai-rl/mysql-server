# InnoDB 锁机制深度解析

> 基于 MySQL 8.0.39 源码，涵盖 record lock / gap lock / next-key lock、锁与 MVCC 版本的关系、幻读、半一致性读与 gap lock 的边界。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [锁与 MVCC 版本的关系](#锁与-mvcc-版本的关系)
- [gap lock 与幻读](#gap-lock-与幻读)
- [半一致性读与 gap lock 的边界](#半一致性读与-gap-lock-的边界)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

InnoDB 的锁分两个正交维度：表级意向锁（IS/IX）与行级锁。行级锁按粒度分为 record lock（锁记录）、gap lock（锁间隙）、next-key lock（record + gap）。

### 用途

控制并发写（record lock 防"两个事务同时改同一行"）与并发读一致性（gap/next-key lock 防幻读）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | `innodb_locks_unsafe_for_binlog` 控制 RC 下是否禁用 gap lock（semi-consistent read 的前身） |
| 8.0 | 移除该变量，gap lock 由隔离级别决定（`skip_gap_locks()`） |

---

## 理论基础

### 设计模式

- 锁表（lock table）+ 等待图（wait-for graph）拆环死锁检测。
- latch ordering（`sync0types.h` 的 `latch_level_t`）预防锁顺序死锁。

### 相关论文

- ARIES（WAL 与锁的配合）；Gray & Reuter《Transaction Processing》的锁粒度分级。

### 算法与数据结构

- `lock_rec_t`：通过 `(space_id, page_no, heap_no)` 定位物理记录，挂在 rec 上。
- 等待图：`lock_wait` 记录 waiter→blocker，死锁检测做环检测。

---

## 锁与 MVCC 版本的关系

**核心结论：行锁锁的是"索引记录（rec，物理记录）"，不是 MVCC 版本。两者正交。**

- 锁（`lock_rec_t`）：控制"谁能修改这条记录"，定位靠 `(space_id, page_no, heap_no)`。
- 版本（MVCC）：记录上的 `DB_TRX_ID` + `DB_ROLL_PTR` 指向 undo 链，控制"谁能读到哪个历史值"。

事务 UPDATE 一条记录时同时发生两件事：在 rec 上加 X 锁（锁物理位置，阻止别人改），并把旧值写进 undo（生成旧版本）。所以"记录被别的锁占了"指 rec 上有不兼容锁（别人正在改/已改未提交），与版本无关。

semi-consistent read 的做法：rec 被锁加不了锁（要改得等），但 rec 的 undo 链上有"已提交的旧版本"，可沿链读出来（`row_vers_build_for_semi_consistent_read`）判断 WHERE。

---

## gap lock 与幻读

### 为什么需要 gap lock

幻读（phantom read）：同一事务内两次范围查询，第二次多出了新插入的行。RR 及以上隔离级别用 gap/next-key lock 锁住"间隙"，阻止别的记录插入该范围，从而防幻读。

### gap lock 依附于记录

gap lock 锁的是"间隙"（记录之间的空隙），但实现上必须**依附一条记录**表达：`LOCK_GAP` 类型的锁挂在某记录上，表示锁住"这条记录之前的空隙"。所以 gap lock 总是和某条"边界记录"绑定。

### gap lock 的精确语义（只禁止插入）

- `LOCK_GAP`：只禁止**向间隙 INSERT 新记录**（防幻读的"多出记录"那一半），**不管**已有记录的修改/删除。
- `LOCK_REC_NOT_GAP`（record lock）：禁止修改/删除**这条记录**。
- `LOCK_ORDINARY`（next-key lock = record + gap）：两者结合，才实现"范围既不被插入、已有记录也不被改删"的完整稳定。

所以"gap lock 禁止整个 range 任意变化"是误解——gap lock 单独只禁止"插入"。

### skip_gap_locks 与隔离级别

`trx_t::skip_gap_locks()`（`trx0trx.h:1138`）：READ UNCOMMITTED / READ COMMITTED 返回 true（不加 gap lock，容忍幻读）；REPEATABLE READ / SERIALIZABLE 返回 false（加 gap lock，防幻读）。

---

## 半一致性读与 gap lock 的边界

`row_compare_row_to_range`（`row0sel.cc:4281`）在加锁路径上判断"当前记录 vs 扫描范围"、决定加 `LOCK_ORDINARY`/`LOCK_REC_NOT_GAP`/`LOCK_GAP` 哪种锁。

`row0sel.cc:4335-4341` 注释 + 断言（`ut_ad(!trx->skip_gap_locks())`、`ut_ad(row_read_type == ROW_READ_WITH_LOCKS)`）的含义：

走到该分支说明前面的 `if (trx->skip_gap_locks() || ...)` 已 return 掉所有 RC/RU 情况，即当前必在 RR（加 gap lock）。而 semi-consistent read 只在"不加 gap lock"的 RC/RU 下启用（`allow_semi_consistent() == skip_gap_locks()`）。两者永不共存，因此无需证明"范围末尾行被锁+删除+purge，同时还在对它做半一致性读"的复杂场景。

"范围末尾的行" = 扫描范围（WHERE 划定的边界）处那条记录，它的前一个间隙需要被 gap lock 锁住；若该记录被删/purge，"给已不存在的记录锁间隙"变得微妙，这是注释不愿去证明的情形。

### 为什么本质互斥（为什么只能 RC/RU）

semi-consistent read 的优化手段是"读到不满足 WHERE 的**最新已提交版本**就**跳过这行、不锁它**"；gap lock 的使命是"**锁住范围边界**、防止新记录插入（防幻读）"。前者要求"允许跳过、允许范围变化"，后者要求"禁止插入、边界稳定"，语义互斥——同一行不可能既"跳过不锁"又"锁住它前面的间隙"。

注意：semi-consistent 读的是"最新已提交版本"，不是"未提交的新值"（被锁的 rec 上当前值属于未提交事务 X，不能用来判 WHERE），也不是"任意旧值"——是沿 undo 链找的第一个已提交版本（`row_vers_build_for_semi_consistent_read` 用 `trx_rw_is_active` 判断）。

### 为什么这个 case 与 RR/gap lock 绑定（RC 也有记录被删被 purge）

关键不在"记录被删被 purge"本身（RC 也有），而在"**要不要给这条被删的记录挂 gap lock 锁边界**"。

反例：`UPDATE t SET x=1 WHERE a>10` 扫到范围末尾 R（`a=100`），事务 X 删除 R 并已 purge（物理移除）。

- **RC（无 gap lock）**：只对 R 本身负责——读最新已提交版本判 WHERE，满足重读加锁/不满足跳过；R 被 purge 下次扫描自然看不到。**无需锁 R 前面的间隙**，锚点消失无害。
- **RR（有 gap lock）**：除处理 R 本身，还**必须给"R 之前的间隙"加 gap lock**（防 `a>100` 插入）。这个间隙锁必须**挂在 R 上**（gap lock 依附记录），R 被 purge 后索引里定位不到锚点，范围末尾边界锁失效 → 幻读。

所以 case 与 RR 的关联点是：**gap lock 的施加依赖"范围边界记录还存在、能当锚点"，而 semi-consistent read（读已提交版本、跳过、配合记录被删被 purge）会让锚点消失。** RC 没有"必须锁边界"这回事，故同样的"记录被删被 purge"在 RC 无害、在 RR 却让防幻读失效。

---

## 核心调用栈

```
-- 加锁 --
row_search_mvcc (row0sel.cc)
  → sel_set_rec_lock (row0sel.cc:1138)
    → lock_clust_rec_read_check_and_lock (聚集索引, lock0lock.cc)
      / lock_sec_rec_read_check_and_lock (二级索引)
        → lock_rec_lock (lock0lock.cc)  ← 冲突则进锁等待/死锁检测

-- 半一致性读拿旧版本（rec 被锁时不等待）--
row_search_mvcc (row0sel.cc:5288, case DB_SKIP_LOCKED)
  → row_sel_build_committed_vers_for_mysql (row0sel.cc:721)
    → row_vers_build_for_semi_consistent_read (row0vers.cc:1368)
      → trx_undo_prev_version_build (沿 undo 链回溯已提交版本)
```

---

## 锁的诊断与查看

### 方式一：SHOW ENGINE INNODB STATUS（文本 dump）

```sql
SET GLOBAL innodb_status_output_locks = ON;   -- 不打开看不到 IX 锁和完整记录锁列表
SHOW ENGINE INNODB STATUS\G
```

读 `---TRANSACTION` 段：

- `TABLE LOCK table ... lock mode IX`：表级意向锁。
- `RECORD LOCKS space id N page no N n bits N index XXX ... lock_mode X locks rec but not gap`：记录锁（space/page/索引名/锁模式）；`waiting` 表示等待中。
- `Record lock, heap no N PHYSICAL RECORD: ...`：具体锁住的记录（heap no + 字段 hex，含 `DB_TRX_ID`/`DB_ROLL_PTR`）。
- `TRX HAS BEEN WAITING N SEC FOR THIS LOCK TO BE GRANTED`：等待时长。

### 方式二：performance_schema（8.0 推荐，可 SQL 化）

```sql
SELECT * FROM performance_schema.data_locks;        -- 所有锁，一行一把（LOCK_TYPE/LOCK_MODE/LOCK_STATUS/LOCK_DATA/INDEX_NAME）
SELECT * FROM performance_schema.data_lock_waits;   -- 等待关系：requesting vs blocking
SELECT * FROM sys.innodb_lock_waits;                 -- 视图，直接给等待方/阻塞方/等待 SQL
```

配合 `information_schema.innodb_trx`（trx_query/trx_started/trx_state）。

### 源码出处

- `SHOW ENGINE INNODB STATUS` 锁段：`srv_printf_innodb_monitor`（srv0srv.cc:1350）→ `lock_print_info_all_transactions`（lock0lock.cc:4991）→ `lock_trx_print_wait_and_mvcc_state`（lock0lock.cc:4849）。
- `performance_schema.data_locks`：`Innodb_data_lock_iterator`（handler/p_s.cc:56）遍历锁系统填充，`LOCK_MODE` 由 `lock_mode_string`（lock0lock.cc:6004）生成。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `transaction_isolation` | REPEATABLE-READ | Global/Session | 决定是否加 gap lock（`skip_gap_locks()`） |
| `innodb_lock_wait_timeout` | 50 | Global/Session | 锁等待超时 |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| `Innodb_row_lock_waits` | 行锁等待次数 |
| `Innodb_row_lock_time` | 行锁等待总时长 |
| `Innodb_row_lock_current_waits` | 当前正在等待的行锁数 |

---

## Misc

### 锁 vs 版本（易混淆）

- "记录被锁" = rec 上有不兼容锁（要改它得等）。
- "读旧版本" = 沿 rec 的 undo 链找已提交历史值。
- 二者正交：一条记录可以同时"被锁着"（别人在改）和"有可读旧版本"（我能读）。

### 幻读 vs 不可重复读

- 不可重复读：同一行两次读到不同值（record lock 防）。
- 幻读：范围查询多出/少了行（gap/next-key lock 防）。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/include/trx0trx.h:1138` | `skip_gap_locks()`，RC/RU 为 true |
| `storage/innobase/row/row0sel.cc:4281` | `row_compare_row_to_range`：行与范围关系、决定锁类型 |
| `storage/innobase/row/row0sel.cc:4335` | 半一致性读与 gap lock 边界注释 + 断言 |
| `storage/innobase/row/row0sel.cc:1138` | `sel_set_rec_lock` 加锁入口 |
| `storage/innobase/row/row0vers.cc:1368` | `row_vers_build_for_semi_consistent_read`（读旧版本） |
| `storage/innobase/row/row0vers.cc:1396` | `trx_rw_is_active` 判断版本已提交 |
| `storage/innobase/lock/lock0lock.cc` | `lock_rec_lock` / 死锁检测 |

# MySQL 外部 XA 事务（分布式事务）

> **一句话概述**：外部 XA 是 MySQL 作为**资源管理器（RM）**暴露给外部事务管理器（TM）的 X/Open 分布式事务接口（`XA START/END/PREPARE/COMMIT/ROLLBACK/RECOVER`），由 TM 而非 MySQL 自己决定最终提交或回滚。
>
> **边界**：本文只讲**外部 XA**。内部 XA（binlog ↔ InnoDB 两阶段提交、`trx_prepare_low`、`MYSQL_BIN_LOG::prepare/commit`、`Xid_log_event`）见 [`../innodb/trx.md`](../innodb/trx.md) 与 [`replication/binlog.md`](replication/binlog.md)，本文仅在必要处交叉引用。

## 目录

- [1. 概述](#1-概述)
- [2. 理论基础](#2-理论基础)
- [3. 场景与用法](#3-场景与用法)
- [4. 核心实现](#4-核心实现)
- [5. 系统变量与错误码](#5-系统变量与错误码)
- [6. Misc：已知陷阱](#6-misc已知陷阱)
- [7. 参考](#7-参考)

---

## 1. 概述

### 1.1 是什么

XA 是 X/Open 定义的**分布式事务处理（DTP）接口标准**。MySQL 支持其中的 `xa_*` 子集：

| SQL | 作用 |
| --- | --- |
| `XA START xid [JOIN\|RESUME]` | 开启一个事务分支（branch），进入 ACTIVE |
| `XA END xid [SUSPEND [FOR MIGRATE]]` | 结束分支上的工作，进入 IDLE |
| `XA PREPARE xid` | 第一阶段的 prepare，进入 PREPARED（**可持久化、可崩溃恢复**） |
| `XA COMMIT xid [ONE PHASE]` | 第二阶段提交（ONE PHASE 时跳过 PREPARE） |
| `XA ROLLBACK xid` | 第二阶段回滚 |
| `XA RECOVER [CONVERT XID]` | 列出所有 PREPARED 状态的 XA 事务 |

### 1.2 MySQL 的定位：RM，不是 TM

MySQL 服务器自身**从不当 TM**。它只回答 TM 的问题："这个 xid 你准备好了吗？提交吧 / 回滚吧"。真正的 2PC 裁决权在外部 TM（应用、中间件、JTA 等）。这一点决定了外部 XA 的两个本质特征：

1. MySQL 必须把 PREPARED 状态**跨崩溃保留**（写 redo + binlog），因为裁决可能在重启之后才到达
2. MySQL **不会**自己决定悬挂事务的命运——没有 TM 的 `XA COMMIT`/`XA ROLLBACK`，它会一直挂在 `XA RECOVER` 里（见 [6. Misc](#6-misc已知陷阱)）

### 1.3 外部 XA vs 内部 XA（一句话）

| | 外部 XA | 内部 XA |
| --- | --- | --- |
| 协调者（TC） | **外部 TM** | **binlog（MySQL 自己）** |
| xid 来源 | 客户端 SQL 给出 | 服务器自动生成 `MySQLXid(server_id, my_xid)` |
| binlog 中的收尾事件 | `XA_prepare_log_event` + `Query_log_event("XA COMMIT ...")` | `Xid_log_event` |
| 崩溃恢复依据 | binlog 扫描出的 `Xa_state_list` | binlog 扫描出的 `Xid_commit_list` |
| 恢复后归宿 | 留在 PREPARED，等 TM | 立即 commit/rollback |

源码里区分内外的**唯一判据**是 `xid_t::get_my_xid()`：内部 XA 的 gtrid 以 `"MySQLXid"` 开头且长度固定，返回非 0；外部 XA 返回 0。

```cpp
// sql/xa.cc
my_xid xid_t::get_my_xid() const {
  if (gtrid_length == static_cast<long>(MYSQL_XID_GTRID_LEN) &&
      bqual_length == 0 &&
      !memcmp(data, MYSQL_XID_PREFIX, MYSQL_XID_PREFIX_LEN)) {
    my_xid tmp;
    memcpy(&tmp, data + MYSQL_XID_OFFSET, sizeof(tmp));
    return tmp;
  }
  return 0;
}
```

### 1.4 8.0 的关键演进（核实结果）

| 说法 | 8.0.39 核实结论 |
| --- | --- |
| 8.0.29+ "XA PREPARE 持久化改进" | ✅ 成立。`XID_STATE::m_is_binlogged` + `enum_ha_recover_xa_state::PREPARED_IN_TC` + `set_prepared_in_tc_in_engines` / `set_prepared_in_tc_by_xid` 这套"**SE 内标记已 prepare 于 TC**"的机制，让恢复时能区分"只在 SE 里 prepare 过（`PREPARED_IN_SE`）"与"TC 已确认（`PREPARED_IN_TC`）" |
| 8.0.30 引入 `xa_detach_on_prepare` | ⚠️ 本仓库无法证实具体小版本号。可确定：8.0.39 中它是 `SESSION` 级、默认 `true`，注释写明 "ON is the only safe choice for replication" |
| `xid_cache` / `XID_cache` | ❌ **8.0.39 已不存在**（全仓 grep 0 命中）。取代者是 `xa::Transaction_cache`（`sql/xa/transaction_cache.h`）：单例、`malloc_unordered_map<std::string, std::shared_ptr<Transaction_ctx>>`、由 `LOCK_transaction_cache` 保护 |
| `innodb_support_xa` | ❌ **8.0.39 已不存在**，8.0 起内部 XA 恒开启、不可关闭 |
| `Xid_log_event` 用于外部 XA | ❌ 错误。外部 XA PREPARE 写 `XA_prepare_log_event`（类型码 `XA_PREPARE_LOG_EVENT = 38`），见 [4.5](#45-binlog-中的-xa两种事件) |

---

## 2. 理论基础

### 2.1 X/Open DTP 模型与 MySQL 的映射

```
   ┌──────────────┐   SQL: XA START/END/PREPARE/COMMIT/ROLLBACK/RECOVER
   │   AP 应用    │ ──────────────────────────────────────┐
   └──────────────┘                                       │
   ┌──────────────┐   xa_* 或 TM 自有协议                  ▼
   │ TM 事务管理器 │ ───────────────────────► ┌────────────────────┐
   │ (JTA/中间件)  │ ◄─────────────────────── │  MySQL Server = RM │
   └──────────────┘   OK / XA_RB* / XAER_*   │  (sql/xa/*.cc)     │
                                             └─────────┬──────────┘
                                                       │ ha_* / handlerton
                                             ┌─────────▼──────────┐
                                             │  InnoDB (RM 实现)  │
                                             └────────────────────┘
```

- **AP**：写业务代码的人，调用 `XA START ...`
- **TM**：决定全局 commit/rollback。MySQL **不实现** TM
- **RM**：MySQL。`sql/xa/` 下 6 个 `Sql_cmd_xa_*` 类就是 RM 侧对 `xa_*` 的实现

### 2.2 两阶段提交对应关系

| 2PC 阶段 | MySQL 侧动作 | 关键函数 |
| --- | --- | --- |
| Phase 1 - 投票 | `ha_prepare_low()` → `innobase_xa_prepare`；把 `XA_prepare_log_event` 写入 binlog | `process_xa_prepare()` |
| Phase 2 - 裁决 | `tc_log->commit/rollback()` 或 `trx_coordinator::commit_detached_by_xid()` | `Sql_cmd_xa_commit/rollback` |
| 恢复 | 扫描 binlog → `Xa_state_list` → `recover_one_external_trx()` | `ha_recover()` |

### 2.3 presumed abort 还是 presumed commit？

**严格说两者都不是，但恢复行为等价于 presumed abort。**

- MySQL 作为 RM，在 `XA PREPARE` 崩溃后**保留**事务（`XA RECOVER` 可见），不会自作主张提交——这排除了 presumed commit
- 但**当 TC 侧没有任何"已 prepare"的证据时**，MySQL 恢复流程会把该事务**回滚**：

```cpp
// sql/xa/recovery.cc —— recover_one_external_trx()
case enum_ha_recover_xa_state::NOT_FOUND:
case enum_ha_recover_xa_state::PREPARED_IN_SE:
case enum_ha_recover_xa_state::ROLLEDBACK: {
  ... ht.rollback_by_xid(&ht, const_cast<XID *>(&xa_trx.id)); ...
}
```

即"**查无裁决证据 ⇒ 回滚**"，这正是 presumed abort 的核心思想。而 presumed abort 经典的"省去写 abort 日志"优化对 MySQL 不适用——binlog 里本来就没有裁决记录时，MySQL 必须做一次**实际回滚**。

### 2.4 与内部 XA 的本质差异

内部 XA 中 binlog **就是** TC，所以"提交 or 回滚"的裁决写在 binlog 里，恢复时读 binlog 即可自洽闭环；外部 XA 中 binlog 只是**记录者**，裁决权在外部 TM，因此恢复后只能"**停在 PREPARED 并把事务上下文复活**"（`Recovered_xa_transactions`），等 TM 上门。

### 2.5 他库对比：PostgreSQL 的 2PC

| | MySQL 外部 XA | PostgreSQL `PREPARE TRANSACTION` |
| --- | --- | --- |
| 标准 | X/Open XA（`xa_*` 语义） | 自创 SQL 语法，语义等价 2PC |
| 事务 ID | 三段式 `gtrid,bqual,formatID` | 单个字符串 gid（≤200 字节） |
| 显式 PREPARE 语句 | `XA END` + `XA PREPARE`（两步） | `PREPARE TRANSACTION 'gid'`（一步） |
| 查看悬挂事务 | `XA RECOVER`（需 `XA_RECOVER_ADMIN`） | `SELECT * FROM pg_prepared_xacts` |
| 崩溃后是否保留 | 是（依赖 binlog + redo，见 [6](#6-misc已知陷阱)） | 是（WAL 中记录 prepare） |
| 与流复制的关系 | PREPARE 会进 binlog，从库可见 prepared | PREPARE 写 WAL，备库可见 |

---

## 3. 场景与用法

### 3.1 状态机

`XID_STATE::xa_states`（`sql/xa.h`）：`XA_NOTR / XA_ACTIVE / XA_IDLE / XA_PREPARED / XA_ROLLBACK_ONLY`。

```
                  XA START xid
   XA_NOTR ───────────────────────► XA_ACTIVE
      ▲                                │  DML / SELECT ...
      │                                │  XA END xid
      │                                ▼
      │                             XA_IDLE
      │      XA COMMIT xid ONE PHASE   │  XA PREPARE xid
      └────────────────────────────────┤
                                       ▼
                                  XA_PREPARED ──── XA COMMIT   ──► XA_NOTR
                                       │        └── XA ROLLBACK ──► XA_NOTR
                                       │ RM 单方面回滚
                                       └─────────► XA_ROLLBACK_ONLY ──► (只能 ROLLBACK)

  边界校验：
  - XA START  要求 state == XA_NOTR，否则 ER_XAER_RMFAIL
  - XA END    要求 state ∈ {XA_ACTIVE, XA_ROLLBACK_ONLY}，否则 ER_XAER_RMFAIL
  - XA PREPARE 要求 state == XA_IDLE，否则 ER_XAER_RMFAIL
  - 任何阶段 xid 不匹配 → ER_XAER_NOTA；xid 重复 → ER_XAER_DUPID
```

### 3.2 xid 的三段式

SQL 层 `xid` 语法支持 1/2/3 段：

```sql
XA START 'gtrid';                     -- formatID=1, bqual=''（由解析器填 1L）
XA START 'gtrid','bqual';             -- formatID=1
XA START 'gtrid','bqual',12345;       -- 完整三段
```

构造最终落到 `xid_t::set`（`sql/xa.h`，与 X/Open XID 结构二进制兼容，`XIDDATASIZE=128`）：

```cpp
void set(long f, const char *g, long gl, const char *b, long bl) {
  formatID = f;
  memcpy(data, g, gtrid_length = gl);
  bqual_length = bl;
  if (bl > 0) memcpy(data + gl, b, bl);
}
```

长度上限：`MAXGTRIDSIZE 64`、`MAXBQUALSIZE 64`（`sql/handler.h`）。

**序列化形式**由 `serialize_xid()`（`sql/xa_aux.h`）产生，即 `X'hex',X'hex',fmt`。这个字符串同时被 `XA RECOVER` 输出、`operator<<`、以及 binlog 里 `XA COMMIT/ROLLBACK` 的 Query_log_event 文本使用——是 [4.5.2](#452-xa-commit--xa-rollback-以文本-query_log_event-记录与词法提取-xid) 的基础。

### 3.3 `xa_detach_on_prepare=ON` 的语义

`ON`（默认）：`XA PREPARE` 成功后，事务上下文**从连接上摘下**：

- 事务对象（含 `Transaction_ctx`、MDL backup、InnoDB trx 句柄）交给 `xa::Transaction_cache`
- 原连接立刻可以执行新语句、开新 XA
- **任何**连接（甚至不是原连接）都能 `XA COMMIT`/`XA ROLLBACK` 这个 xid

`OFF`（旧行为）：事务仍绑在原连接；其他连接不能 commit/rollback 它（`XA RECOVER` 看得到，但 commit 报 `ER_XAER_NOTA`）。

代价：`ON` 时**临时表不能用在 XA 里**（`ER_XA_TEMP_TABLE`），因为临时表生命周期绑定 session，detach 后会失效。

### 3.4 `XA RECOVER` 能看到什么

```cpp
// sql/xa/sql_xa_recover.cc
auto list = xa::Transaction_cache::get_cached_transactions();
for (const auto &transaction : list) {
  XID_STATE *xs = transaction->xid_state();
  if (xs->has_state(XID_STATE::XA_PREPARED)) { /* 输出一行 */ }
}
```

- 输出 4 列：`formatID / gtrid_length / bqual_length / data`
- `XA RECOVER CONVERT XID` 时 `data` 以 `0x...` 十六进制输出，否则输出原始二进制
- 需要 `XA_RECOVER_ADMIN` 权限
- **只显示 `XA_PREPARED`**，ACTIVE/IDLE 的事务不显示
- 跨连接全局可见（数据源是单例 cache），**包括崩溃恢复后复活的事务**（以 `start_detached_xa()` 插入，状态即 `XA_PREPARED`）

### 3.5 `XA COMMIT ... ONE PHASE`

要求 state == `XA_IDLE`（未经 PREPARE），直接走 `ha_commit_trans(thd, true)`。若开了 binlog，仍会写一条 `XA_prepare_log_event` 但 `one_phase=true`，恢复时映射到 `COMMITTED_WITH_ONEPHASE`，等价于"已提交"。

---

## 4. 核心实现

### 4.1 代码地图（8.0.39）

| 文件 | 职责 |
| --- | --- |
| `sql/xa.h` | `xid_t`(XID)、`XID_STATE`、`Recovered_xa_transactions` |
| `sql/xa.cc` | `ha_recover()`、`cleanup_trans_state()`、`find_trn_for_recover_and_check_its_state()` |
| `sql/xa/sql_cmd_xa.h` | 汇总 6 个命令类 |
| `sql/xa/sql_xa_start.cc` / `sql_xa_end.cc` | `trans_xa_start()` / `trans_xa_end()` |
| `sql/xa/sql_xa_prepare.cc` | `process_xa_prepare()` / `detach_xa_transaction()` / `reset_xa_connection()` |
| `sql/xa/sql_xa_commit.cc`、`sql_xa_rollback.cc` | attached / detached 两条路径 |
| `sql/xa/sql_xa_second_phase.{h,cc}` | 第二阶段公共骨架（detached 场景） |
| `sql/xa/sql_xa_recover.cc` | `XA RECOVER` |
| `sql/xa/recovery.cc` | `recover_one_ht` / `recover_one_internal_trx` / `recover_one_external_trx` |
| `sql/xa/transaction_cache.{h,cc}` | 取代已删除的 `xid_cache` |
| `sql/xa/xid_extract.{h,cc}` | 从 SQL 文本里用**正则**提取 XID |
| `sql/log_event.{h,cc}` | `XA_prepare_log_event` |
| `sql/binlog/recovery.cc` | `binlog::Binlog_recovery`，扫描 binlog 构造 `Xa_state_list` |

### 4.2 `XA START`

```cpp
// sql/xa/sql_xa_start.cc
bool Sql_cmd_xa_start::trans_xa_start(THD *thd) {
  XID_STATE *xid_state = thd->get_transaction()->xid_state();

  if (xid_state->has_state(XID_STATE::XA_IDLE) && m_xa_opt == XA_RESUME) {
    bool not_equal = !xid_state->has_same_xid(m_xid);
    if (not_equal) my_error(ER_XAER_NOTA, MYF(0));
    else { xid_state->set_state(XID_STATE::XA_ACTIVE); }
    return not_equal;
  }

  /* TODO: JOIN is not supported yet. */
  if (m_xa_opt != XA_NONE)                       my_error(ER_XAER_INVAL, MYF(0));
  else if (!xid_state->has_state(XID_STATE::XA_NOTR))
                                                 my_error(ER_XAER_RMFAIL, MYF(0), xid_state->state_name());
  else if (thd->locked_tables_mode || thd->in_active_multi_stmt_transaction())
                                                 my_error(ER_XAER_OUTSIDE, MYF(0));
  else if (!trans_begin(thd)) {
    xid_state->start_normal_xa(m_xid);           // XA_ACTIVE, m_is_detached=false
    if (xa::Transaction_cache::insert(m_xid, thd->get_transaction())) {
      xid_state->reset();                        // XID 重复 → 回滚
      trans_rollback(thd);
    }
  }
  return thd->is_error() || !xid_state->has_state(XID_STATE::XA_ACTIVE);
}
```

逐段解释：

1. **`RESUME`**：只从 `XA_IDLE` 恢复，且 xid 必须一致，否则 `ER_XAER_NOTA`。`JOIN` 未实现
2. **三重前置约束**：非 `XA_NONE` 选项 → `ER_XAER_INVAL`；已处于某 XA 状态 → `ER_XAER_RMFAIL`（RM 不能在同一会话嵌套全局事务）；`LOCK TABLES` 或已有多语句事务 → `ER_XAER_OUTSIDE`
3. **真正开始**：`trans_begin()` 起普通事务，再 `start_normal_xa()` 打上 XA 标记
4. **唯一性**：`Transaction_cache::insert()` 失败即 xid 已存在，报 `ER_XAER_DUPID` 并回滚

### 4.3 `XA END`

```cpp
// sql/xa/sql_xa_end.cc
bool Sql_cmd_xa_end::trans_xa_end(THD *thd) {
  XID_STATE *xid_state = thd->get_transaction()->xid_state();
  /* TODO: SUSPEND and FOR MIGRATE are not supported yet. */
  if (m_xa_opt != XA_NONE)                              my_error(ER_XAER_INVAL, MYF(0));
  else if (!xid_state->has_state(XID_STATE::XA_ACTIVE) &&
           !xid_state->has_state(XID_STATE::XA_ROLLBACK_ONLY))
                                                        my_error(ER_XAER_RMFAIL, MYF(0), xid_state->state_name());
  else if (!xid_state->has_same_xid(m_xid))             my_error(ER_XAER_NOTA, MYF(0));
  else if (!xid_state->xa_trans_rolled_back())          xid_state->set_state(XID_STATE::XA_IDLE);
  return thd->is_error() || !xid_state->has_state(XID_STATE::XA_IDLE);
}
```

★ `xa_trans_rolled_back()` 是关键分支：若 RM 曾单方面回滚（锁超时/死锁），状态是 `XA_ROLLBACK_ONLY`，此时**不进 IDLE**，后续只能 `XA ROLLBACK`。

### 4.4 `XA PREPARE`

```cpp
// sql/xa/sql_xa_prepare.cc
int process_xa_prepare(THD *thd) {
  int error{0};
  auto trn_ctx = thd->get_transaction();

  if (trn_ctx->is_active(Transaction_ctx::SESSION)) {
    bool gtid_error = false, need_clear_owned_gtid = false;
    std::tie(gtid_error, need_clear_owned_gtid) = commit_owned_gtids(thd, true);
    auto clean_up_guard = create_scope_guard([&]() {
      if (error != 0) ha_rollback_trans(thd, true);
      Commit_order_manager::wait_and_finish(thd, error);
      gtid_state_commit_or_rollback(thd, need_clear_owned_gtid, !error);
    });
    if (gtid_error) { assert(need_clear_owned_gtid); return error = 1; }
    if (Commit_order_manager::wait(thd)) { thd->commit_error = THD::CE_NONE; return error = 1; }

    Clone_handler::XA_Operation xa_guard(thd);   // 允许 SE 读到 GTID

    if (tc_log) {
      if ((error = tc_log->prepare(thd, /* all */ true)) != 0) return error;
    } else if ((error = trx_coordinator::set_prepared_in_tc_in_engines(thd, true)) != 0)
      return error;
  }
  return error;
}
```

逐段解释：

1. ★ **`commit_owned_gtids(thd, true)`**：**GTID 在 XA PREPARE 时就被分配并提交**，不是等到 XA COMMIT。这是 XA 与 GTID 的核心关系——PREPARE 是 binlog 上一个独立的事务边界，因此它单独消耗一个 GTID
2. **`Commit_order_manager::wait`**：仅对复制 applier 线程生效，保证从库提交序与主库一致
3. **两条 prepare 分支**：`tc_log` 存在（binlog 或 `tc_log_mmap`）走 `tc_log->prepare()`；否则（无 binlog 的 `TC_LOG_DUMMY` 路径）只调 `set_prepared_in_tc_in_engines()`，把 SE 内状态标记为 `PREPARED_IN_TC`
4. 失败由 `scope_guard` 统一 `ha_rollback_trans` + 归还 GTID

detach 部分：

```cpp
if (!is_xa_tran_detached_on_prepare(thd)) {   // 旧行为
  rollback_xa_tran.commit();
  return thd->is_error();
}
if (::detach_xa_transaction(thd)) return true;
rollback_xa_tran.commit();
::reset_xa_connection(thd);
```

`detach_xa_transaction()` 做三件事：`MDL_context_backup_manager::create_backup()`（保存事务持有的 MDL，供将来在别的连接上恢复）→ `xa::Transaction_cache::detach()` → 对每个参与 hton 调 `replace_native_transaction_in_thd(thd, nullptr, nullptr)`，触发 InnoDB 侧 `trx_disconnect_prepared`。

### 4.5 binlog 中的 XA：两种事件

#### 4.5.1 `XA_prepare_log_event`（外部 XA PREPARE 专用）

**`Xid_log_event`（`XID_EVENT`）不用于外部 XA**，它只服务内部 XA/DDL。外部 XA PREPARE 写的是类型码 38 的 `XA_prepare_log_event`：

```
+-------------------+-----------------+-----------------+-----------------+--------------------+
| one_phase (1 byte)| formatID (4)    | gtrid_length(4) | bqual_length(4) | data[gtrid+bqual]  |
+-------------------+-----------------+-----------------+-----------------+--------------------+
```

写入时机：`MYSQL_BIN_LOG::commit()` 中，当 `is_loggable_xa_prepare(thd)`（即 `XA_IDLE` 状态）或 `XA COMMIT ... ONE PHASE` 时：

```cpp
// sql/binlog.cc
XA_prepare_log_event end_evt(thd, xs->get_xid(), one_phase);
err = cache_mngr->trx_cache.finalize(thd, &end_evt, xs);
```

★ 注意它通过 `MYSQL_BIN_LOG::prepare()` 里 `if (!error && all && is_xa_prepare(thd)) return this->commit(thd, true);` **借道 binlog group commit** 落盘，因此 redo 与 binlog 的顺序保证复用内部 XA 的同一套机制。

#### 4.5.2 `XA COMMIT` / `XA ROLLBACK` 以文本 `Query_log_event` 记录与"词法提取 XID"

第二阶段**没有专用事件**，而是把 `XA COMMIT X'..',X'..',1` 这条 SQL 本身作为 `Query_log_event` 写入：

```cpp
// sql/binlog.cc —— MYSQL_BIN_LOG::write_xa_to_cache()
std::ostringstream oss;
oss << "XA " << (thd->lex->sql_command == SQLCOM_XA_COMMIT ? "COMMIT" : "ROLLBACK")
    << " " << *xid_to_write << std::flush;          // operator<<(ostream, xid_t) → serialize_xid()
Query_log_event qinfo(thd, query.data(), query.length(), false, true, true, 0, false);
return this->write_event(&qinfo);
```

反向解析（崩溃恢复扫描 binlog 时）必须**从 SQL 文本里把 XID 抠出来**——这就是 `xa::XID_extractor`（`sql/xa/xid_extract.cc`），用一条 `std::regex`：

```cpp
size_t xa::XID_extractor::extract(std::string const &source, size_t max_extractions) {
  static const std::regex xid_regex{
      "X'((?:[0-9a-fA-F][0-9a-fA-F]){1,64})?'"  // GTRID
      "(?:[[:space:]]*),(?:[[:space:]]*)"       // white-space and comma
      "X'((?:[0-9a-fA-F][0-9a-fA-F]){1,64})?'"  // BQUAL
      "(?:[[:space:]]*),(?:[[:space:]]*)"       // white-space and comma
      "(0|[1-9][0-9]{0,19})",                   // FORMATID
      std::regex::optimize};
  ...
  auto gtrid = xa::extractor::unhex(tokenizer[1]);
  auto bqual = xa::extractor::unhex(tokenizer[2]);
  xid.set(format_id, gtrid.data(), gtrid.length(), bqual.data(), bqual.length());
}
```

调用点（`sql/binlog/recovery.cc`）：

```cpp
void binlog::Binlog_recovery::process_query_event(Query_log_event const &ev) {
  std::string query{ev.query};
  if (query == "BEGIN" || query.find("XA START") == 0) this->process_start();
  else if (query == "COMMIT")                          this->process_commit();
  else if (query == "ROLLBACK")                        this->process_rollback();
  else if (is_atomic_ddl_event(&ev))                   this->process_atomic_ddl(ev);
  else if (query.find("XA COMMIT") == 0)               this->process_xa_commit(query);
  else if (query.find("XA ROLLBACK") == 0)             this->process_xa_rollback(query);
}

void binlog::Binlog_recovery::add_external_xid(std::string const &query,
                                               enum_ha_recover_xa_state state) {
  xa::XID_extractor tokenizer{query, 1};
  if (tokenizer.size() == 0) { this->m_is_malformed = true; return; }
  auto found = this->m_external_xids.find(tokenizer[0]);
  if (found != this->m_external_xids.end()) {
    assert(found->second != enum_ha_recover_xa_state::PREPARED_IN_SE);
    if (found->second != enum_ha_recover_xa_state::PREPARED_IN_TC) {
      this->m_is_malformed = true; return;      // ★ 同一 xid 两次裁决 → binlog 损坏
    }
  }
  this->m_external_xids[tokenizer[0]] = state;  // COMMITTED / ROLLEDBACK
}
```

★ **"词法提取"的代价**：恢复逻辑依赖文本前缀匹配（`query.find("XA COMMIT") == 0`）。任何以 `XA COMMIT`/`XA ROLLBACK` 开头却不含合法 XID 的语句都会被判为 binlog malformed；这也解释了为什么 `write_xa_to_cache()` 必须用固定的 `serialize_xid()` 格式生成文本。

#### 4.5.3 恢复态映射

| binlog 中看到 | `enum_ha_recover_xa_state` |
| --- | --- |
| `XA_prepare_log_event`（two-phase） | `PREPARED_IN_TC` |
| `XA_prepare_log_event`（`one_phase=true`） | `COMMITTED_WITH_ONEPHASE` |
| `Query_log_event` "XA COMMIT ..." | `COMMITTED` |
| `Query_log_event` "XA ROLLBACK ..." | `ROLLEDBACK` |

### 4.6 `XA COMMIT` / `XA ROLLBACK`：attached 与 detached 两条路

判据只有一句——`xid_state->has_same_xid(this->m_xid)`：

```cpp
// sql/xa/sql_xa_commit.cc
bool Sql_cmd_xa_commit::trans_xa_commit(THD *thd) {
  auto xid_state = thd->get_transaction()->xid_state();
  Clone_handler::XA_Operation xa_guard(thd);
  if (!xid_state->has_same_xid(this->m_xid)) return this->process_detached_xa_commit(thd);
  return this->process_attached_xa_commit(thd);
}
```

**attached**（本连接自己的 XA）：`process_attached_xa_commit()` 按状态分派——`XA_IDLE` + `ONE_PHASE` → `ha_commit_trans()`；`XA_PREPARED` + `XA_NONE` → 先拿 `MDL_key::COMMIT` 意向排他锁（与 FTWRL 互斥，拿不到报 `ER_XA_RETRY` 让用户重试），再 `tc_log->commit()`。

**detached**（`xa_detach_on_prepare=ON` 后的主流路径）共用 `Sql_cmd_xa_second_phase` 骨架：

```cpp
bool Sql_cmd_xa_commit::process_detached_xa_commit(THD *thd) {
  raii::Sentry<> dispose_guard{[this]() { this->dispose(); }};
  if (this->find_and_initialize_xa_context(thd)) return true;   // 从 Transaction_cache 找
  if (this->acquire_locks(thd)) return true;                    // xa_lock + MDL backup 恢复
  raii::Sentry<> acquired_locks_guard{[this]() { this->release_locks(); }};
  this->setup_thd_context(thd);                                 // GTID + 传递 is_binlogged 标志
  if (this->enter_commit_order(thd)) return true;               // 从库保序
  this->assign_xid_to_thd(thd);                                 // 把 xid 借挂到当前 THD
  if (!this->m_result) {
    if (tc_log == nullptr) this->m_result = trx_coordinator::commit_detached_by_xid(thd);
    else                   this->m_result = tc_log->commit(thd, /* all */ true);
  } else ::force_rollback(thd);
  this->exit_commit_order(thd);
  this->cleanup_context(thd);
  return this->m_result;
}
```

几个细节：

- `find_and_initialize_xa_context()` → `find_trn_for_recover_and_check_its_state()`：要求当前会话处于 `XA_NOTR` 且无活跃多语句事务；回调谓词 `is_detached()` 确保拿到的是 **detached** 事务（顺带规避与 `~THD()` 的竞态）。找不到 → `ER_XAER_NOTA`
- ★ `acquire_locks()` 拿 `XID_STATE::m_xa_lock`，防止两个会话并发对同一 xid 做 COMMIT/ROLLBACK 而向 binlog 写两条事件（注释明确指出这会破坏复制）；并做了"加锁后二次确认事务仍在 cache"的 double-check
- `setup_thd_context()` 把 detached 事务的 `is_binlogged()` 借道 THD 传给底层 binlog 例程，决定 `XA COMMIT` 是否写 binlog
- `force_rollback()` 临时把 `sql_command` 改成 `SQLCOM_XA_ROLLBACK`，让回滚栈按真正的 `XA ROLLBACK` 行为走（以便正确写 binlog）

### 4.7 崩溃恢复

`ha_recover()`（`sql/xa.cc`）分两轮扫描所有 SE 插件：

```cpp
if (plugin_foreach(nullptr, xa::recovery::recover_prepared_in_tc_one_ht,
                   MYSQL_STORAGE_ENGINE_PLUGIN, &info)) return 1;
if (plugin_foreach(nullptr, xa::recovery::recover_one_ht,
                   MYSQL_STORAGE_ENGINE_PLUGIN, &info)) return 1;
```

- 第 1 轮 `recover_prepared_in_tc_one_ht`：让 SE 把"已标记 prepare 于 TC"的事务回填进 `Xa_state_list`（合并 SE 侧与 TC 侧视角）
- 第 2 轮 `recover_one_ht`：遍历 SE 报告的所有 prepared 事务，按 `get_my_xid()==0` 分流：

```cpp
// sql/xa/recovery.cc
my_xid xid = xa_trx.id.get_my_xid();
if (!xid) {                                            // 外部 XA
  ::recover_one_external_trx(*info, *ht, xa_trx, external_stats);
  ++info->found_foreign_xids;
  continue;
}
if (info->dry_run) { ++info->found_my_xids; continue; }   // 内部 XA，无 TC 信息
::recover_one_internal_trx(*info, *ht, xa_trx, xid, internal_stats);
```

★ **`enum_ha_recover_xa_state` 实际是 6 个值**（`sql/handler.h`），不是常见资料说的 4 个：

```cpp
enum class enum_ha_recover_xa_state : int {
  NOT_FOUND = -1,               // 事务未找到
  PREPARED_IN_SE = 0,           // 仅在 SE 中 prepare
  PREPARED_IN_TC = 1,           // SE 与 TC 都已 prepare
  COMMITTED_WITH_ONEPHASE = 2,  // 一阶段提交
  COMMITTED = 3,                // 已提交
  ROLLEDBACK = 4                // 已回滚
};
```

`recover_one_external_trx()` 据此决策：

| 状态 | 动作 |
| --- | --- |
| `COMMITTED_WITH_ONEPHASE` / `COMMITTED` | `ht.commit_by_xid()` |
| `NOT_FOUND` / `PREPARED_IN_SE` / `ROLLEDBACK` | `ht.rollback_by_xid()` |
| **`PREPARED_IN_TC`** | `Recovered_xa_transactions::add_prepared_xa_transaction()` + `ht.set_prepared_in_tc_by_xid()`，**保留为 prepared** |

只有 `PREPARED_IN_TC` 的事务会被 `recover_prepared_xa_transactions()`（在 `dd::reset_tables_and_tablespaces()` 之后调用，避免死锁）插入 `xa::Transaction_cache` 并恢复其 MDL 备份，从而对新会话的 `XA RECOVER` / `XA COMMIT` 可见。

### 4.8 关键约束小结

| 约束 | 代码依据 |
| --- | --- |
| XA 中禁止隐式提交 | `sql_parse.cc`：`trans_check_state(thd)` 在 `stmt_causes_implicit_commit` 分支中直接 return -1，并保留 MDL |
| `xa_detach_on_prepare=ON` 时禁止临时表 | `sql_base.cc`：`ER_XA_TEMP_TABLE` |
| 复制过滤 + XA 不支持 | `sql_xa_prepare.cc`：applier 上空事务 → `ER_XA_REPLICATION_FILTERS` |
| XID 全局唯一 | `Transaction_cache::insert()` → `ER_XAER_DUPID` |
| GTID 在 PREPARE 而非 COMMIT 分配 | `process_xa_prepare()` 中 `commit_owned_gtids` |
| ★ **binlog 关闭 ⇒ prepared XA 重启后被回滚** | `ha_recover()`：无 `commit_list` 且无 `tc_heuristic_recover` 时 `dry_run=true`，或 `total_ha_2pc == opt_bin_log + 1` 时强制 `TC_HEURISTIC_RECOVER_ROLLBACK`；此时 `Xa_state_list` 为空 ⇒ 所有外部 XA 落到 `NOT_FOUND` ⇒ `rollback_by_xid` |
| 关于 `HA_IGNORE_DURABILITY` | ⚠️ **常见说法有误**：`MYSQL_BIN_LOG::prepare()` 对**所有** 2PC（含外部 XA PREPARE）都设置 `thd->durability_property = HA_IGNORE_DURABILITY`。redo 的 fsync 并非在 prepare 里做，而是在随后的 binlog group commit **flush 阶段**由 `ha_flush_logs(true)` 统一完成——"先刷 redo 再刷 binlog"的顺序保证对外部 XA 同样成立。即：不是"不适用"，而是"延后到 BGC flush 阶段统一 fsync" |

---

## 5. 系统变量与错误码

| 变量 | 作用域 | 默认 | 说明 |
| --- | --- | --- | --- |
| `xa_detach_on_prepare` | SESSION | `true` | `XA PREPARE` 时是否把事务从连接摘下。`ON` 时任意连接可 COMMIT/ROLLBACK，原连接可开新事务，但**禁用 XA 内临时表**。变量带 `IN_BINLOG` 与 `ON_CHECK(check_session_admin_outside_trx_outside_sf)`（不能在事务/存储函数内修改） |
| `innodb_support_xa` | — | — | ❌ **8.0.39 已不存在**，无需关注 |
| `tc_heuristic_recover` | GLOBAL（启动项） | `NOT_USED` | 影响 `ha_recover()` 的 `dry_run` 与外部事务默认裁决方向 |
| `max_prepared_stmt_count` | — | — | 与 XA 无关（预处理语句，不是 prepared 事务） |

XA 相关错误码：

| 错误 | 含义 |
| --- | --- |
| `ER_XAER_NOTA` (XAE04) | Unknown XID |
| `ER_XAER_INVAL` (XAE05) | 参数非法或命令不支持（如 `JOIN`/`SUSPEND`） |
| `ER_XAER_RMFAIL` (XAE07) | 当前状态不允许该命令（状态机违例） |
| `ER_XAER_DUPID` (XAE08) | XID 已存在 |
| `ER_XAER_OUTSIDE` (XAE09) | 全局事务之外还有未完成的工作 |
| `ER_XAER_RMERR` (XAE03) | RM 致命错误，需人工核对数据 |
| `ER_XA_RBROLLBACK` (XA100) / `ER_XA_RBDEADLOCK` (XA102) / `ER_XA_RBTIMEOUT` (XA106) | 事务分支已被回滚（一般 / 死锁 / 超时） |
| `ER_XA_RETRY` | RM 暂时无法提交，请重试（拿不到 COMMIT MDL 时） |
| `ER_XA_TEMP_TABLE` | `xa_detach_on_prepare=ON` 时 XA 内访问临时表 |
| `ER_XA_REPLICATION_FILTERS` | 复制过滤与 XA 混用 |

---

## 6. Misc：已知陷阱

1. **悬挂事务永远挂着**。没有 TM 发来 `XA COMMIT/ROLLBACK`，`XA RECOVER` 就一直列出该 xid，其行锁/MDL 备份也一直持有，会阻塞 DDL 与后续 DML。只能人工 `XA ROLLBACK`。`tc_heuristic_recover` 启动项**只影响内部 XA**（`recover_one_internal_trx`），外部 XA 不受其 `COMMIT`/`ROLLBACK` 选项控制。
2. **关闭 binlog 会让 prepared XA 在重启时全部回滚**（见 [4.8](#48-关键约束小结)）。所以生产上"XA + `skip-log-bin`"不是等价降级，而是**丢失跨崩溃的 prepared 语义**。
3. **XA 与复制**：8.0.39 中 `XA PREPARE` **一定**会写 `XA_prepare_log_event`（类型 38），副本应用该事件时会执行 prepare 并通过 `applier_reset_xa_trans()` 把事务从 applier 线程 detach 掉，因此**从库能看到 prepared 事务**——经典问题在 8.0 已解决。`XA COMMIT`/`XA ROLLBACK` 则以 `Query_log_event` 文本形式复制。注意 `xa_detach_on_prepare` 的注释："ON is the only safe choice for replication"——`OFF` 时副本侧的 prepared 事务仍绑定在 applier 连接上，语义与主库不一致。
4. ★ **GTID 消耗**：每个 XA 事务的 PREPARE 就已经分配并提交 GTID，因此 `gtid_executed` 里出现 XA 的 GTID **不代表**该 XA 已提交。用 GTID 判断"数据是否已提交"在 XA 场景下会误判。
5. **binlog 与 XA 的不一致风险**：`sql_xa_commit.cc` 里留有官方 TODO——GTID 落盘失败被迫回滚时，"XA rollback" 事件可能漏写进 binlog（`Todo/fixme: fix binlogging, "XA rollback" event could be missed out`）。
6. **文本解析的脆弱性**：`XA COMMIT`/`XA ROLLBACK` 的 XID 靠正则从 SQL 文本提取（`xa::XID_extractor`），恢复时若匹配不到合法 XID 会直接把 binlog 判为 malformed。

---

## 7. 参考

- 本仓库源码：`sql/xa.h`、`sql/xa.cc`、`sql/xa/` 目录全部文件、`sql/handler.h`（`enum_ha_recover_xa_state` / `Xa_state_list` / `ha_recover`）、`sql/tc_log.{h,cc}`、`sql/binlog.cc`、`sql/binlog/recovery.cc`、`sql/log_event.{h,cc}`、`libbinlogevents/include/control_events.h`
- 交叉引用：[`../innodb/trx.md`](../innodb/trx.md)（InnoDB 事务与 `trx_prepare_low`）、[`replication/binlog.md`](replication/binlog.md)（内部 XA 与 binlog group commit）
- MySQL 8.0 Reference Manual：XA Transactions / XA Restrictions
- X/Open CAE Specification, Distributed Transaction Processing: The XA Specification（1991）——`sql/xa.h` 中 `xid_t` 的注释明确声明与之二进制兼容

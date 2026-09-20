# 全局锁二件套 FTWRL 与备份锁深度解析

> 基于 MySQL 8.0.39 源码，涵盖 `FLUSH TABLES WITH READ LOCK` 的 `Global_read_lock`（GLOBAL S + COMMIT S 两段式）、8.0 的 `LOCK INSTANCE FOR BACKUP`（MDL `BACKUP_LOCK` namespace）、备份锁"语义 X = MDL S"的命名反转推导、两者的阻塞范围对比，以及 `SET GLOBAL read_only` 为何与 FTWRL 共用同一套机制。
>
> **边界**：本篇讲**锁整个实例的两把锁**；它们都不是独立锁系统，而是 **MDL 框架的组合用法**——MDL 本身的机制（18 namespace、优先级矩阵、死锁检测）见 [`mdl.md`](mdl.md)。表级锁 THR_LOCK 见 [`thr_lock.md`](thr_lock.md)，InnoDB 行锁见 [`innodb_trx_lock.md`](innodb_trx_lock.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 语境里的"全局锁"通常指两把——加上只读变量才构成完整的实例级只读：

| | SQL | 实现 | 本质 |
|---|---|---|---|
| **全局读锁** | `FLUSH TABLES WITH READ LOCK`（FTWRL） | `Global_read_lock` 类（`sql_class.h`） | `MDL_key::GLOBAL` 的 **S** + `MDL_key::COMMIT` 的 **S** |
| **备份锁** | `LOCK INSTANCE FOR BACKUP`（8.0） | `sql_backup_lock.cc` | `MDL_key::BACKUP_LOCK` 的 **S** |
| （第三件套） | `SET GLOBAL read_only=1` | 变量，不是锁 | 与 `super_read_only` 配合构成实例级只读 |

**关键认知**：`Global_read_lock` 类里除了两个 `MDL_ticket*` 什么都没有——它**不是一套锁机制，而是"用 MDL 组合出来的语义"**。源码注释第一句就点明了：

> Global read lock is implemented using metadata lock infrastructure.

### 用途

- **FTWRL**：为**一致性逻辑备份**提供"全库静止"——阻塞新的写、刷表闭表、再阻塞提交，从而拿到一个不再移动的 `SHOW MASTER STATUS` 位点（`mysqldump --single-transaction --master-data` 的经典用法）。
- **备份锁**：8.0 为**在线物理备份**（XtraBackup 等）设计——只挡 DDL 与 PURGE BINLOG 这类"会改变文件集/元数据"的操作，**放行 DML 与所有读**，代价远低于 FTWRL。

为什么 8.0 还要再加一把？因为 FTWRL 太重：它连提交都阻塞（备份期间业务完全写不进去）。物理备份真正需要的只是"**备份期间文件集不变**"——DDL 会改表结构文件，所以要挡；DML 只是改数据页，物理备份本身能容忍（靠 redo 收敛）。备份锁就是把这个精确语义用 MDL 表达出来。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.5 | MDL 引入，`Global_read_lock` 从"自实现"改为基于 MDL 的两段式（GLOBAL S + COMMIT S） |
| 5.x | FTWRL 是唯一的全局锁；`SET GLOBAL read_only` 另有一套实现 |
| 8.0 | 新增 `LOCK INSTANCE FOR BACKUP`（MDL `BACKUP_LOCK` namespace）+ `BACKUP_ADMIN` 权限；`SET GLOBAL read_only/super_read_only` 改为**复用** `Global_read_lock`（见「核心实现」） |
| 8.0 | `Global_read_lock` 的 `m_have_global_read_lock` 等布尔标志被 `enum_grl_state m_state` 三态取代 |

---

## 理论基础

### 设计思想与权衡

#### 一、为什么复用 MDL 而不是自造一套全局锁

FTWRL 要表达"阻塞所有写 + 阻塞提交"，最直白的做法是加一个全局标志位 + 一把 mutex。MySQL 选择了**把它表达成两个 scoped MDL**：GLOBAL S 挡住所有写语句（写语句自动取 GLOBAL IX），COMMIT S 挡住提交（提交路径取 COMMIT IX）。

收益是**免费获得 MDL 的全部能力**：优先级矩阵、死锁检测、PFS 可见、统一的 `lock_wait_timeout`、统一的 state 字符串。源码注释把这点讲得很透：

> How blocking of threads by global read lock is achieved: that's semi-automatic. We assume that any statement which should be blocked by global read lock will either open and acquires write-lock on tables or acquires metadata locks on objects it is going to modify. For any such statement global IX metadata lock is automatically acquired for its duration... **If deadlock happens it is detected by MDL subsystem and resolved in the standard fashion.**

代价是**阻塞范围依赖"写语句会自动取 GLOBAL IX"这个约定**（散落在 `open_table()` 等处），而不是由 FTWRL 自己枚举——注释用 "semi-automatic" 形容，也意味着**只改临时表的语句不受 FTWRL 约束**（原文括号：`unless they modify only temporary tables`），这是设计上接受的漏洞。

#### 二、★ 为什么必须是两阶段：一个三线程死锁

这是本机制最值得记住的设计决策。把"拿 GLOBAL S"与"拿 COMMIT S"合成一步会死锁，`sql/lock.cc` 的注释给出了完整的反例：

> 1) `lock_global_read_lock()`（阻止任何新的表写锁，即stall所有新更新）
> 2) `close_cached_tables()`（the FLUSH TABLES），它会等待当前已打开且正在被更新的表关闭
> 3) `make_global_read_lock_block_commit()`
>
> If we have merged 1) and 3) into 1), we would have had this deadlock:
> thd1: `SELECT * FROM t FOR UPDATE;`
> thd2: `UPDATE t SET a=1;` # 被 thd1 的行锁阻塞
> thd3: `FLUSH TABLES WITH READ LOCK;` # 被 thd2 的表实例阻塞在 `close_cached_tables()`
> thd1: `COMMIT;` # 被 thd3 阻塞
> **thd1 blocks thd2 which blocks thd3 which blocks thd1: deadlock.**

即：**必须在第 2 步闭表期间允许 COMMIT**，否则"持有行锁的事务"无法提交，闭表就永远等不到表关闭。代价是 FTWRL 存在一个"新更新已停、但已有事务还能提交"的中间窗口——这正是备份工具要先 FTWRL 再 `SHOW MASTER STATUS` 的原因（第 3 步之后位点才真正静止）。

注释还交代了一个"例外中的例外"：

> Note that we need to support that one thread does `FLUSH TABLES WITH READ LOCK;` and then `COMMIT;`（that's what innobackup does, for some good reason）. So in this exceptional case the COMMIT should not be blocked by the FLUSH TABLES WITH READ LOCK.

这个例外由 MDL 的 ticket 复用天然实现（持有 S 再请求 IX 会命中 `find_ticket()` 直接复用，不会自锁），无需额外代码。

#### 三、★ 备份锁的命名反转：用优先级矩阵防饿死

`acquire_exclusive_backup_lock()` 取的是 **`MDL_SHARED`**，`acquire_shared_backup_lock()` 取的是 **`MDL_INTENTION_EXCLUSIVE`**——API 名与 MDL 类型名**正好相反**。源码给了完整推导（`sql_backup_lock.cc` 注释原文）：

> IX and S locks are mutually incompatible. On the other hand both these lock types are compatible with themselves. **IX lock has lower priority than S locks.** So we can use S lock for rare Backup operation which should not be starved by more frequent DDL operations using IX locks.
>
> From all listed above follows that **S lock should be considered as Exclusive Backup Lock and IX lock should be considered as Shared Backup Lock.**

三层推理：① S 与 IX 互不兼容（满足互斥）；② S 与 S 兼容、IX 与 IX 兼容（同类可并存）；③ **IX 优先级低于 S**——所以把 S 给"稀有方"（备份），IX 给"高频方"（DDL），备份就不会被源源不断的 DDL 饿死。

"Shared Backup Lock"（IX）的"shared"是**指 DDL 之间互相共享**——多个 DDL 可以各自持 IX 并存，它们挡的是别人的 S（备份）。

代价是**命名与实现长期背离**，读代码时极易误判：`acquire_shared_backup_lock()` 拿到的其实是一把会挡住别人的 IX。缓解方式是 API 名用"语义名"（Exclusive/Shared Backup Lock）、内部用 MDL 类型名，两者靠这段注释锚定。

#### 四、失效场景与退化阈值

1. **FTWRL 不挡只读事务的提交**：提交路径只在 `rw_trans`（真正改了数据）时才取 COMMIT IX，只读事务不取——所以 FTWRL 期间纯只读事务仍能提交（这符合"只读不阻塞"的语义）。
2. **只改临时表的写语句不受 FTWRL 约束**（注释明写），因为它们不取表级写锁。
3. **DD 内部线程绕过 FTWRL**：`mark_ignore_global_read_lock()` 让数据字典的提交不被自己的 FTWRL 阻塞。
4. **备份锁不挡 DML**（见下），所以它不是"一致性逻辑备份"的工具——用它做 `mysqldump --single-transaction` 拿不到静止位点，这是**设计意图而非缺陷**。
5. **备份锁期间连持有者自己也不能 PURGE BINLOG**：`Shared_backup_lock_guard` 构造时先查 `owns_equal_or_stronger_lock(BACKUP_LOCK, "", "", MDL_SHARED)`，若已持有则直接判 `not_locked`。注释原文："PURGE BINARY LOG is not allowed even when instance is locked for backup by the same session."——**锁对"实例状态"不对"会话豁免"**。

### 理论溯源

- **多粒度封锁（multi-granularity locking）的 scoped 层**：Gray 1976 的经典方案用"意向锁 + 显式的库/表/行层级"实现多粒度。MySQL 的 MDL scoped 锁（GLOBAL / BACKUP_LOCK / COMMIT / TABLESPACE / SCHEMA）就是这套方案的"上层"——scoped 兼容矩阵（`sql/mdl.cc` 的注释矩阵）里 `IX` 与 `S` 不兼容、`S` 与 `S` 兼容，正是多粒度锁的标准形态。落点：`MDL_key::GLOBAL` 与 `MDL_key::BACKUP_LOCK` 都走 scoped 策略。
- **优先级调度防饿死**：备份锁选 S 的理由就是"稀有方拿高优先级"。这属于排队论里的"优先级 + 防饥饿"权衡，与 InnoDB 行锁授予时按 CATS 权重排序（见 `innodb_trx_lock.md`）是同一类问题的两种解法：一个靠**静态类型优先级**，一个靠**动态权重**。

### 他库对比与演进动机

| 数据库 | 全局/实例级只读怎么做 | 与 MySQL 的差异 |
|---|---|---|
| **PostgreSQL** | 没有 FTWRL 等价物。逻辑备份靠 `pg_dump` 的可重复读快照（MVCC，不阻塞写）；`pg_basebackup` 物理备份也不阻塞写 | PG 从一开始就有完整 MVCC，不需要"阻塞提交来拿静止位点"；MySQL 因为 binlog 位点必须静止才引入 FTWRL |
| **Oracle** | `ALTER DATABASE BEGIN BACKUP`（旧）/ RMAN（新）。RMAN 靠 redo 收敛，完全在线 | 与 MySQL 备份锁思路一致（不阻塞业务），但 Oracle 无 MDL 层，靠 redo/归档保证一致性 |
| **MySQL** | FTWRL（阻塞写+提交，拿静止位点）+ 备份锁（只挡 DDL） | 两套并存，因为两种备份需求不同（逻辑 vs 物理） |

**为什么演进成两把**：FTWRL 是为**逻辑备份**（需要静止的 binlog 位点）设计的，代价是阻塞提交；8.0 加备份锁是为**物理备份**设计的，它不需要静止位点，只需要文件集不变。二者不是替代关系而是分工——源码里也没有一处注释说"备份锁取代 FTWRL"。

---

## 核心实现

### 主链路

```
【FTWRL】
handle_reload_request()  (sql/sql_reload.cc, REFRESH_READ_LOCK 分支)
  ├─ thd->global_read_lock.lock_global_read_lock(thd)        ① GLOBAL namespace 取 S
  ├─ close_cached_tables(thd, tables, ...)                   ② 刷表 + 等表关闭（期间允许 COMMIT）
  └─ thd->global_read_lock.make_global_read_lock_block_commit(thd)   ③ COMMIT namespace 取 S
释放：UNLOCK TABLES / 连接断开 THD::cleanup() → unlock_global_read_lock()

【备份锁】
Sql_cmd_lock_instance::execute()
  ├─ check_backup_admin_privilege()         需要 BACKUP_ADMIN 权限
  └─ acquire_exclusive_backup_lock(thd, timeout, for_trx=false)   BACKUP_LOCK namespace 取 S
释放：UNLOCK INSTANCE / 连接断开 → release_backup_lock()
```

两条链路的共同点：**语句执行完锁不释放**（duration 是 `MDL_EXPLICIT`），必须显式 `UNLOCK` 或断开连接。

### FTWRL：`Global_read_lock`

#### 类结构

```cpp
class Global_read_lock {
 public:
  enum enum_grl_state { GRL_NONE, GRL_ACQUIRED, GRL_ACQUIRED_AND_BLOCKS_COMMIT };
  ...
  bool lock_global_read_lock(THD *thd);
  bool make_global_read_lock_block_commit(THD *thd);
  void unlock_global_read_lock(THD *thd);
  bool can_acquire_protection() const {
    if (m_state) { my_error(ER_CANT_UPDATE_WITH_READLOCK, MYF(0)); return true; }
    return false;
  }
  void set_explicit_lock_duration(THD *thd);
 private:
  static std::atomic<int32> m_atomic_active_requests;
  enum_grl_state m_state;
  /* In order to acquire the global read lock, the connection must
     acquire shared metadata lock in GLOBAL namespace, to prohibit all DDL. */
  MDL_ticket *m_mdl_global_shared_lock;
  /* Also ... acquire a shared metadata lock in COMMIT namespace, to prohibit commits. */
  MDL_ticket *m_mdl_blocks_commits_lock;
};
```

- 三态 `m_state` 精确对应"没拿 / 只拿了 GLOBAL S / 两段都拿了"。
- `m_atomic_active_requests` 是给 **InnoDB memcached 插件**看的全局计数器（注释：*Used by innodb memcached server to check if any connections have global read lock*）——加解锁前后各增减一次。
- `can_acquire_protection()` 被反向使用：别的语句想取 GLOBAL IX 前先问它，已持有 GRL 就报 `ER_CANT_UPDATE_WITH_READLOCK`。

#### 第一段：`lock_global_read_lock()`

```cpp
bool Global_read_lock::lock_global_read_lock(THD *thd) {
  if (!m_state) {
    MDL_request mdl_request;
    assert(!thd->mdl_context.owns_equal_or_stronger_lock(MDL_key::GLOBAL, "", "", MDL_SHARED));
    MDL_REQUEST_INIT(&mdl_request, MDL_key::GLOBAL, "", "", MDL_SHARED, MDL_EXPLICIT);

    /* Increment static variable first to signal innodb memcached server
       to release mdl locks held by it */
    Global_read_lock::m_atomic_active_requests++;
    if (thd->mdl_context.acquire_lock(&mdl_request, thd->variables.lock_wait_timeout)) {
      Global_read_lock::m_atomic_active_requests--;
      return true;
    }
    m_mdl_global_shared_lock = mdl_request.ticket;
    m_state = GRL_ACQUIRED;
  }
  /*
    We DON'T set global_read_lock_blocks_commit now, it will be set after
    tables are flushed ... Doing things in this order is necessary to avoid
    deadlocks (we must allow COMMIT until all tables are closed; ...).
  */
  return false;
}
```

三个要点：

1. **取的是 GLOBAL namespace 的 `MDL_SHARED`，duration `MDL_EXPLICIT`**——不随事务结束释放。
2. **本函数不做任何 flush/close**（`if (!m_state)` 里只拿锁）。闭表由调用方 `close_cached_tables()` 完成——这是两阶段能成立的前提：函数本身是"纯加锁"，调用方负责编排顺序。
3. `assert` 保证本线程此前没有 GLOBAL S（避免重复获取）；失败时回滚计数器。

#### 第二段：`make_global_read_lock_block_commit()`

```cpp
bool Global_read_lock::make_global_read_lock_block_commit(THD *thd) {
  /* If we didn't succeed lock_global_read_lock(), or if we already succeeded
     make_global_read_lock_block_commit(), do nothing. */
  if (m_state != GRL_ACQUIRED) return false;

  MDL_request mdl_request;
  MDL_REQUEST_INIT(&mdl_request, MDL_key::COMMIT, "", "", MDL_SHARED, MDL_EXPLICIT);
  if (thd->mdl_context.acquire_lock(&mdl_request, thd->variables.lock_wait_timeout))
    return true;
  m_mdl_blocks_commits_lock = mdl_request.ticket;
  m_state = GRL_ACQUIRED_AND_BLOCKS_COMMIT;
  return false;
}
```

`if (m_state != GRL_ACQUIRED) return false` 保证**幂等且顺序正确**：没走第一段就直接返回（不报错），已走过也直接返回。

**"阻塞提交"的原理**在提交路径 `ha_commit_trans()`：

```cpp
DEBUG_SYNC(thd, "ha_commit_trans_before_acquire_commit_lock");
if (rw_trans && !ignore_global_read_lock) {
  /*
    Acquire a metadata lock which will ensure that COMMIT is blocked
    by an active FLUSH TABLES WITH READ LOCK (and vice versa:
    COMMIT in progress blocks FTWRL).

    We allow the owner of FTWRL to COMMIT; we assume that it knows what it does.
  */
  MDL_REQUEST_INIT(&mdl_request, MDL_key::COMMIT, "", "", MDL_INTENTION_EXCLUSIVE,
                   MDL_EXPLICIT);
  if (thd->mdl_context.acquire_lock(&mdl_request, thd->variables.lock_wait_timeout)) {
    ha_rollback_trans(thd, all);
    return 1;
  }
  release_mdl = true;
}
```

- 提交取 **COMMIT namespace 的 IX**，与 FTWRL 的 COMMIT S 不兼容（scoped 矩阵 `IX` vs `S` → `-`），故被挂起。
- **只有 `rw_trans`（真正改了数据）才取**——只读事务不取，所以 FTWRL 不挡它们。
- "允许持有者自己 COMMIT" 无需特殊代码：`MDL_context::try_acquire_lock_impl()` 会先 `find_ticket()`，已持有 S 再请求 IX 命中复用。
- `ignore_global_read_lock` 让 DD 内部线程（`mark_ignore_global_read_lock()`）绕过自己的 FTWRL。

#### 释放（逆序）

```cpp
void Global_read_lock::unlock_global_read_lock(THD *thd) {
  assert(m_mdl_global_shared_lock && m_state);
  if (m_mdl_blocks_commits_lock) {                              // ① 先放 COMMIT S
    thd->mdl_context.release_lock(m_mdl_blocks_commits_lock);
    m_mdl_blocks_commits_lock = nullptr;
  }
  thd->mdl_context.release_lock(m_mdl_global_shared_lock);      // ② 再放 GLOBAL S
  Global_read_lock::m_atomic_active_requests--;
  m_mdl_global_shared_lock = nullptr;
  m_state = GRL_NONE;
}
```

#### 谁在调用：`SET GLOBAL read_only` 与 FTWRL 共用一套

| 调用点 | 场景 |
|---|---|
| `sql/sql_reload.cc` | **`FLUSH TABLES WITH READ LOCK`**（`REFRESH_READ_LOCK` 分支） |
| `sql/sys_vars.cc` | **`SET GLOBAL read_only=1`**（`fix_read_only`） |
| `sql/sys_vars.cc` | **`SET GLOBAL super_read_only=1`**（`fix_super_read_only`） |
| `sql/sql_parse.cc` | `UNLOCK TABLES` 顺带释放 GRL |
| `sql/sql_class.cc` | 连接断开 `THD::cleanup()` 释放 |
| `sql/rpl_source.cc` | `RESET MASTER` 内部自取自放 |

`sys_vars.cc` 的注释解释了为什么要借用：

> READ_ONLY=1 prevents write locks from being taken on tables and blocks transactions from committing. We therefore should make sure that no such events occur while setting the read_only variable. This is a 2 step process: [1] `lock_global_read_lock()` Prevents connections from obtaining new write locks on tables. Note that we can still have active rw transactions. [2] `make_global_read_lock_block_commit()` Prevents transactions from committing.

差别在于**持续时间**：FTWRL 结束后锁（`MDL_EXPLICIT`）常驻到 `UNLOCK TABLES`；而 `SET GLOBAL read_only` 用完**立即释放**（改完变量就走 `unlock_global_read_lock`）——它借 GRL 只是为了"设置变量这个瞬间"的原子性。另外若本会话已持有 GRL（`is_acquired()`），`fix_read_only` 直接设值不再加锁。

#### `FLUSH TABLES` 与 FTWRL 不是一回事

```cpp
if (options & (REFRESH_TABLES | REFRESH_READ_LOCK)) {
  if ((options & REFRESH_READ_LOCK) && thd) {
    if (thd->locked_tables_mode) { my_error(ER_LOCK_OR_ACTIVE_TRANSACTION, MYF(0)); return true; }
    tmp_write_to_binlog = 0;                                        // FTWRL 不写 binlog
    if (thd->global_read_lock.lock_global_read_lock(thd)) return true;
    if (close_cached_tables(thd, tables, ...)) result = true;
    if (thd->global_read_lock.make_global_read_lock_block_commit(thd)) {
      thd->global_read_lock.unlock_global_read_lock(thd);           // 不留半锁状态
      return true;
    }
  } else {
    /* 普通 FLUSH TABLES：只闭表，完全不碰 GRL */
    close_cached_tables(thd, tables, ..., LONG_TIMEOUT);
  }
}
```

| | `FLUSH TABLES` | `FLUSH TABLES WITH READ LOCK` |
|---|---|---|
| GRL | **完全不碰** | GLOBAL S + COMMIT S |
| 闭表 | 是 | 是（且夹在两段之间） |
| 结束后 | 锁立即消失 | 锁常驻到 `UNLOCK TABLES`/断开 |
| 写 binlog | 写 | **不写**（`tmp_write_to_binlog = 0`） |

注意失败路径会 `unlock_global_read_lock()`——**不留"只拿了第一段"的半锁状态**。

第三条路 `FLUSH TABLES ... FOR EXPORT`（`flush_tables_for_export()`）**不取 GRL**，注释原文："Don't acquire global IX as this will make this statement incompatible with FLUSH TABLES WITH READ LOCK"，它取 SNW 后进 LOCK TABLES 模式。

### 备份锁：`LOCK INSTANCE FOR BACKUP`

#### 加锁函数与命名反转

```cpp
static bool acquire_mdl_for_backup(THD *thd, enum_mdl_type mdl_type,
                                   enum_mdl_duration mdl_duration,
                                   ulong lock_wait_timeout) {
  MDL_request mdl_request;
  assert(mdl_type == MDL_SHARED || mdl_type == MDL_INTENTION_EXCLUSIVE);
  MDL_REQUEST_INIT(&mdl_request, MDL_key::BACKUP_LOCK, "", "", mdl_type, mdl_duration);
  return thd->mdl_context.acquire_lock(&mdl_request, lock_wait_timeout);
}

bool acquire_exclusive_backup_lock(THD *thd, ulong lock_wait_timeout, bool for_trx) {
  enum_mdl_duration duration = (for_trx ? MDL_TRANSACTION : MDL_EXPLICIT);
  return acquire_mdl_for_backup(thd, MDL_SHARED, duration, lock_wait_timeout);
}

bool acquire_shared_backup_lock(THD *thd, ulong lock_wait_timeout, bool for_trx) {
  enum_mdl_duration duration = (for_trx ? MDL_TRANSACTION : MDL_EXPLICIT);
  return acquire_mdl_for_backup(thd, MDL_INTENTION_EXCLUSIVE, duration, lock_wait_timeout);
}
```

`assert` 把"只允许这两种类型"钉死——命名反转的边界由它守住。`for_trx` 决定 duration：`MDL_TRANSACTION`（随事务提交释放）还是 `MDL_EXPLICIT`（必须 `UNLOCK INSTANCE`）。

#### `Shared_backup_lock_guard`：非阻塞版 + 同会话不免

```cpp
Shared_backup_lock_guard::Shared_backup_lock_guard(THD *thd) : m_thd(thd) {
  // If instance is locked for the backup, then even block operations requisting
  // shared backup lock. For example, PURGE BINARY LOG is not allowed even when
  // instance is locked for backup by the same session.
  if (thd->mdl_context.owns_equal_or_stronger_lock(MDL_key::BACKUP_LOCK, "", "",
                                                   MDL_SHARED)) {
    m_lock_state = Shared_backup_lock_guard::Lock_result::not_locked;
    return;
  }
  m_lock_state = try_acquire_shared_backup_lock(m_thd, false);
}

Shared_backup_lock_guard::~Shared_backup_lock_guard() {
  if (m_lock_state == Shared_backup_lock_guard::Lock_result::locked)
    release_backup_lock(m_thd);
}

Shared_backup_lock_guard::Lock_result
Shared_backup_lock_guard::try_acquire_shared_backup_lock(THD *thd, bool for_trx) {
  MDL_request mdl_request;
  const enum_mdl_duration duration = (for_trx ? MDL_TRANSACTION : MDL_EXPLICIT);
  MDL_REQUEST_INIT(&mdl_request, MDL_key::BACKUP_LOCK, "", "",
                   MDL_INTENTION_EXCLUSIVE, duration);
  if (thd->mdl_context.try_acquire_lock(&mdl_request))
    return Shared_backup_lock_guard::Lock_result::oom;
  if (mdl_request.ticket == nullptr)
    return Shared_backup_lock_guard::Lock_result::not_locked;
  return Shared_backup_lock_guard::Lock_result::locked;
}
```

两个反直觉点：

1. **构造即结果，不等待**：用 `try_acquire_lock`（非阻塞）+ `ticket == nullptr` 判定，拿不到立刻返回 `not_locked`。所以 `PURGE BINARY LOGS` 在备份期间是**立即报错**而不是等待——报的就是 `ER_CANNOT_PURGE_BINLOG_WITH_BACKUP_LOCK`。
2. **同会话不豁免**：`owns_equal_or_stronger_lock(..., MDL_SHARED)` 查的是"是否已持有 S 或更强"，**不区分是谁持有的**。持有备份锁的会话自己想 PURGE BINLOG 也会走到 `not_locked` 分支。注释原话见「理论基础」第 4 点。

`Lock_result` 三值：`not_locked = 0` / `locked = 1` / `oom = 2`（后者是内存不足或真正失败）。

#### 语句入口与权限

```cpp
static bool check_backup_admin_privilege(THD *thd) {
  Security_context *sctx = thd->security_context();
  if (!sctx->has_global_grant(STRING_WITH_LEN("BACKUP_ADMIN")).first) {
    my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "BACKUP_ADMIN");
    return true;
  }
  return false;
}

bool Sql_cmd_lock_instance::execute(THD *thd) {
  if (check_backup_admin_privilege(thd) ||
      acquire_exclusive_backup_lock(thd, thd->variables.lock_wait_timeout, false))
    return true;
  my_ok(thd);
  return false;
}
```

权限名 **`BACKUP_ADMIN`**（`LOCK INSTANCE` 与 `UNLOCK INSTANCE` 都要）。`for_trx=false` → `MDL_EXPLICIT`，语句结束不释放，断开时由 `THD::cleanup()` 兜底（`/* If Backup Lock was acquired it must be released on disconnect. */`）。

#### 谁在拿 Shared Backup Lock（IX）

**★ 关键结论：普通 DML（INSERT / UPDATE / DELETE）不拿备份锁。**

`lock_table_names()` 的判据是 `is_ddl_or_lock_tables_lock_request()`（即 `type >= MDL_SHARED_UPGRADABLE`），而 DML 的表级 MDL 是 `MDL_SHARED_WRITE`（< `MDL_SHARED_UPGRADABLE`），不满足条件——所以 DML 只取 GLOBAL IX，**完全不碰 `BACKUP_LOCK` namespace**。官方测试 `lock_backup.test` 原文佐证：

```
--echo # Test case 2: Check that DML statement is executed successfully
--echo # when LOCK INSTANCE was acquired from another connection.
INSERT INTO t1 VALUES (100);
COMMIT;
```

拿 IX 的调用点清单（全部是"会改变文件集/元数据"的操作）：

| 类别 | 调用点 |
|---|---|
| **DDL 主干** | `lock_table_names()`——`is_ddl_or_lock_tables_lock_request()` 且非 SRO 且非 `SQLCOM_LOCK_TABLES` 时自动加 **BACKUP IX + GLOBAL IX**，覆盖 CREATE/ALTER/DROP/RENAME 等 |
| `LOCK TABLES` 模式 | `acquire_backup_lock_in_lock_tables_mode()` |
| 表操作 | `mysql_rm_table()`（DROP TABLE）、`Sql_cmd_truncate_table::truncate_table()`、trigger 主体 |
| 组件 / 插件 / SERVER | `Sql_cmd_install_component`、`Sql_cmd_uninstall_component`、`Sql_cmd_common_server`、`mysql_install_plugin`/`mysql_uninstall_plugin` |
| 维护类 | `mysql_admin_table()`（REPAIR/OPTIMIZE；注释："Acquire backup lock explicitly since lock types used by admin statements won't cause its automatic acquisition in open_and_lock_tables()"）、`ANALYZE TABLE ... UPDATE/DROP HISTOGRAM` |
| SRS / 资源组 | `Sql_cmd_create_srs`、`Sql_cmd_drop_srs`、CREATE/ALTER/DROP RESOURCE GROUP |
| 实例操作 | `Rotate_innodb_master_key`（**同时取 exclusive + shared**）、`Innodb_redo_log` |
| 复制 | `STOP REPLICA`（用 `MDL_lock_guard` 取 BACKUP IX，失败报 `ER_RPL_CANT_STOP_REPLICA_WHILE_LOCKED_BACKUP`） |
| 引擎侧 | `dd_tablespace_get_mdl()`（InnoDB 后台取 tablespace MDL） |

用 `Shared_backup_lock_guard`（非阻塞版）的只有 PURGE BINLOG 系列：`purge_source_logs_to_file()`、`purge_source_logs_before_date()`、`MYSQL_BIN_LOG::auto_purge()`、复制 applier 侧日志清理。

`ER_CANNOT_PURGE_BINLOG_WITH_BACKUP_LOCK` 的唯一抛出点是 `purge_log_get_error_code()` 里的 `case LOG_INFO_BACKUP_LOCK`；错误文本："Could not purge binary logs since another session is executing LOCK INSTANCE FOR BACKUP. Wait for that session to release the lock."

### 两把锁的完整对比

| 维度 | FTWRL | `LOCK INSTANCE FOR BACKUP` |
|---|---|---|
| MDL 组合 | `GLOBAL` S + `COMMIT` S | `BACKUP_LOCK` S |
| 阻塞读 | 否 | 否 |
| **阻塞 DML** | **是**（写语句自动取 GLOBAL IX） | **否**（DML 不取 BACKUP 锁） |
| 阻塞 DDL | 是 | 是（DDL 自动取 BACKUP IX） |
| **阻塞 COMMIT** | **是**（COMMIT S vs 提交路径 IX；仅 `rw_trans`） | 否 |
| 刷表 / 闭表 | 是（`close_cached_tables()`） | 否 |
| 权限 | `RELOAD` + `LOCK TABLES` | `BACKUP_ADMIN` |
| 释放 | `UNLOCK TABLES` / 断开 | `UNLOCK INSTANCE` / 断开 |
| 拿不到时 | 等待（受 `lock_wait_timeout`） | 等待；但 PURGE BINLOG 走 guard **立即失败** |
| 多会话并存 | 可以（S 与 S 兼容） | 可以 |
| 典型用途 | 一致性**逻辑**备份（要静止 binlog 位点） | 在线**物理**备份（只挡 DDL） |

---

## ★ 本机制里的工程实现技法

### 一、用 scoped MDL 组合表达"全局语义"，而不是自造锁

`Global_read_lock` 类里除了两个 `MDL_ticket*` 什么都没有——"全局读锁"完全是 **GLOBAL S + COMMIT S 两个 scoped MDL 的语义叠加**。

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| 复用 MDL 表达全局语义 | FTWRL、备份锁 | 免费获得死锁检测 / 优先级矩阵 / PFS 可见 / 统一超时 / 统一 state 字符串 | 阻塞范围依赖"写语句会自动取 GLOBAL IX"这一**散落约定**（注释自称 semi-automatic）；只改临时表的语句不受约束 |
| 两段式加锁（加锁与编排分离） | `lock_global_read_lock` 只拿锁，闭表由调用方做 | 让"闭表期间允许 COMMIT"成为可能，避免三线程死锁 | 调用方必须按 1→2→3 顺序编排，顺序错了死锁会重现且无运行时检查 |
| 状态机用枚举而非布尔对 | `enum_grl_state` 三态 | 精确表达"半锁状态"，`make_*` 的幂等判断只需 `m_state != GRL_ACQUIRED` | — |

### 二、🔁 RAII guard 的两种形态：等待版 vs 立即失败版

备份锁有两个入口，体现了两种截然不同的等待策略：

| | `acquire_shared_backup_lock()` | `Shared_backup_lock_guard` |
|---|---|---|
| 获取方式 | `acquire_lock`（阻塞等待） | `try_acquire_lock`（非阻塞） |
| 拿不到 | 等待 `lock_wait_timeout` | 立即返回 `not_locked` |
| 生命周期 | 调用方手动 `release_backup_lock()` | **RAII**，析构自动释放 |
| 典型用户 | DDL、ANALYZE、INSTALL COMPONENT | PURGE BINARY LOGS |

RAII guard 的析构只做一件事：`if (m_lock_state == locked) release_backup_lock(m_thd);`——**只有真的拿到了才释放**，避免了"未获取却释放"的常见 bug。这是 RAII + 显式结果状态配合的干净写法。

### 三、"锁对实例状态、不对会话豁免"

`Shared_backup_lock_guard` 用 `owns_equal_or_stronger_lock()` 检测的是**实例级状态**（是否有人持有 S），不检查"是不是我持有的"。这与很多锁系统的"同会话可重入"直觉相反，但在这里是**刻意**的：备份期间 PURGE BINLOG 会删掉备份可能还没读的日志文件，谁执行都不行。

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| 用"是否已持有更强锁"代替"是否是自己" | `Shared_backup_lock_guard` 构造、DD 的 X 表锁路径 | 语义从"会话豁免"变成"实例状态检查"，规则更简单也更安全 | 持有备份锁的会话自己也被挡，可能与用户直觉冲突 |

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `lock_wait_timeout` | 31536000（1 年） | Global/Session | FTWRL / 备份锁的等待超时；超时报 `ER_LOCK_WAIT_TIMEOUT` |
| `read_only` | OFF | Global | 设为 ON 时**内部借用 GRL** 保证设置瞬间无写、无提交 |
| `super_read_only` | OFF | Global | 同上，且约束 `read_only` 的联动 |

（这两把锁本身没有专用计数器——它们的等待计入 MDL 的 `performance_schema.metadata_locks` 与 events_waits。）

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 谁被 FTWRL 挡住 | SQL | `performance_schema.processlist` 的 `State: Waiting for global read lock` |
| 谁的 COMMIT 被挡住 | SQL | `State: Waiting for commit lock`（仅改了数据的事务会出现） |
| 谁被备份锁挡住 | SQL | `State: Waiting for backup lock` |
| 当前持有哪些全局/备份锁 | PFS | `performance_schema.metadata_locks`，`OBJECT_TYPE` 为 `GLOBAL` / `COMMIT` / `BACKUP LOCK` |
| 是否正有人持全局读锁 | SQL | `SHOW OPEN TABLES` 看不到；用 `metadata_locks` 查 `OBJECT_TYPE='GLOBAL'` |

这三个 state 字符串来自 `MDL_key::m_namespace_to_wait_state_name[]` 数组，**下标就是 namespace 枚举序**：

```cpp
PSI_stage_info MDL_key::m_namespace_to_wait_state_name[NAMESPACE_END] = {
    {0, "Waiting for global read lock", 0, PSI_DOCUMENT_ME},   /* GLOBAL */
    {0, "Waiting for backup lock",      0, PSI_DOCUMENT_ME},   /* BACKUP_LOCK */
    ...
    {0, "Waiting for commit lock",      0, PSI_DOCUMENT_ME},   /* COMMIT */
```

**诊断意义**：三个 state 字符串能直接区分"卡在哪一把锁上"——`Waiting for global read lock`（FTWRL 挡写/DDL）、`Waiting for commit lock`（FTWRL 挡提交）、`Waiting for backup lock`（备份锁挡 DDL）。而 `Waiting for table metadata lock`（表级 MDL）与 `Waiting for table level lock`（THR_LOCK）是完全不同的两层，见 [`thr_lock.md`](thr_lock.md)。

---

## Misc

### 面向二次开发

**扩展点**：新增一种"实例级锁"的正确做法是**在 MDL 里加一个 scoped namespace**，而不是自造锁系统——要改：`MDL_key` 的 namespace 枚举、`m_namespace_to_wait_state_name[]`（同步加 state 字符串）、scoped 兼容矩阵、`MDL_context` 的自动获取逻辑、PFS 的 `OBJECT_TYPE` 映射。备份锁（8.0）就是按这条路加的。

**坑与已知缺陷**：

1. **FTWRL 与 `LOCK TABLES` 互斥**：`LOCK TABLES` 模式下执行 FTWRL 直接报 `ER_LOCK_OR_ACTIVE_TRANSACTION`。
2. **FTWRL 不写 binlog**（`tmp_write_to_binlog = 0`），但 `FLUSH TABLES` 写。
3. **`FLUSH TABLES ... FOR EXPORT` 不取 GRL**（注释：否则会与 FTWRL 不兼容）。
4. **备份锁不挡 DML**——用它做逻辑备份拿不到静止位点，这是设计意图不是 bug。
5. **备份锁期间连持有者自己也不能 PURGE BINLOG**（`Shared_backup_lock_guard` 的实例状态检查）。
6. **DD 内部线程用 `mark_ignore_global_read_lock()` 绕过 FTWRL**——否则数据字典自己的提交会被卡住。
7. **`RESET MASTER` 内部会自取 GRL 再自放**。
8. **多会话可同时持有 FTWRL / 备份锁**（都是 S，S 与 S 兼容）——所以"看到有人在等 `Waiting for global read lock`"不代表只有一把锁在生效。

**社区边界澄清**：备份锁（`LOCK INSTANCE FOR BACKUP`）是 **MySQL 8.0 社区版**特性；Percona 的 `LOCK TABLES FOR BACKUP`（5.x 时代的插件特性）**社区版没有**。InnoDB memcached 插件对 `m_atomic_active_requests` 的使用也是可选组件行为。

### 易混淆概念

- **"全局读锁"有两个含义**：指 `Global_read_lock`（FTWRL）时是"GLOBAL S + COMMIT S"；指 MDL 的 `MDL_key::GLOBAL` 时是**一个 namespace**（写语句自动取它的 IX）。前者用后者实现。
- **`SET GLOBAL read_only` 不是锁**（是个变量），但**设置它的过程**借用了 GRL——所以 read_only 与 FTWRL 会互相影响。
- **`acquire_shared_backup_lock()` 拿的是 IX**："shared"指 DDL 之间共享，不是指与备份共享。
- **`FLUSH TABLES` ≠ `FLUSH TABLES WITH READ LOCK`**：前者只闭表，锁不常驻。

---

## 参考

**论文 / 经典算法**
- J. Gray, R. Lorie, G. Putzolu, I. Traiger. *Granularity of Locks and Degrees of Consistency in a Shared Data Base*. 1976.（多粒度封锁与意向锁；落点：MDL 的 scoped 层 GLOBAL/BACKUP_LOCK/COMMIT 及其兼容矩阵）

**官方文档**
- *MySQL 8.0 Reference Manual → `FLUSH` Statement*
- *MySQL 8.0 Reference Manual → `LOCK INSTANCE FOR BACKUP` and `UNLOCK INSTANCE` Statements*
- *MySQL 8.0 Reference Manual → Backup Lock*（MEB/XtraBackup 用法）

**内核月报 / 技术文章**
- [《MySQL · 特性分析 · 备份锁》（数据库内核月报）](https://www.kancloud.cn/taobaomysql/monthly)——备份锁的设计背景；⚠️ 月报基于早期版本，函数名与行号与 8.0.39 基本对不上，本篇一律以本仓库源码为准。

**相关文档**
- MDL 框架（18 namespace、scoped 兼容矩阵、优先级与死锁检测）见 [`mdl.md`](mdl.md)——本篇的两把锁都是它的用法
- server 层表锁 THR_LOCK（MyISAM 真锁、InnoDB 空壳）见 [`thr_lock.md`](thr_lock.md)
- InnoDB 行锁与表锁见 [`innodb_trx_lock.md`](innodb_trx_lock.md)
- 全量盘点与归属判据见 [`../README.md`](../README.md)

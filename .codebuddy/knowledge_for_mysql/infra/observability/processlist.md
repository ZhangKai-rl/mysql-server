# 会话观测：四种入口的差异、代价与 State 体系

> **边界**：本篇讲"**怎么看到当前有哪些连接、它们在干什么**"这一观测对象的四种入口。
> - sys schema 的 `session` / `processlist` 视图细节见 [`sys_schema.md`](sys_schema.md)
> - PFS 事件表族（waits/stages/transactions）见 [`pfs/pfs_events.md`](pfs/pfs_events.md)
> - 观测手段全景与决策树见 [`../README.md`](../README.md)

## 目录

- [一、概述](#一概述)
- [二、理论基础：两条实现路径与代价来源](#二理论基础两条实现路径与代价来源)
- [三、入口一：`SHOW PROCESSLIST`（legacy，默认）](#三入口一show-processlistlegacy默认)
- [四、入口二：`INFORMATION_SCHEMA.PROCESSLIST`（已废弃）](#四入口二information_schemaPROCESSLIST已废弃)
- [五、入口三：★ `performance_schema.threads`（推荐）](#五入口三-performance_schematextthreads推荐)
- [六、入口四：`sys.session` / `sys.processlist`（人工排障首选）](#六入口四syssession--sysprocesslist人工排障首选)
- [七、★ State 字段体系](#七-state-字段体系)
- [八、四种入口横向对照](#八四种入口横向对照)
- [九、可观测性](#九可观测性)
- [十、★ 工程实现技法](#十-工程实现技法)
- [十一、Misc（易混淆点）](#十一misc易混淆点)
- [参考](#参考)

---

## 一、概述

### 是什么

"当前有哪些连接、分别在干什么"是最高频的观测需求。MySQL 给了**四种入口**，它们的**代价差异极大**，但**看起来的结果差不多**——这是最容易被忽视的坑：

| 入口 | 本质 |
|---|---|
| `SHOW [FULL] PROCESSLIST` | SQL 语句 |
| `INFORMATION_SCHEMA.PROCESSLIST` | 遗留 I_S 表（**已废弃**） |
| `performance_schema.threads` | PFS 表（**推荐**） |
| `sys.session` / `sys.processlist` | sys 视图（信息最全，人工排障首选） |

### 用途

- 看"谁在跑、跑了多久、卡在哪"
- 找长事务 / 慢查询 / 锁等待的源头
- 拿到 `PROCESSLIST_ID` 去 `KILL`
- 拿到 `THREAD_OS_ID` 去关联 OS 层（perf / eBPF / ps）

### 版本演进（源码级）

- **8.0 新增 `performance_schema_show_processlist` 开关**（`sql/sys_vars.cc`）：默认 `OFF`，即默认走 legacy 遍历 THD 的路径；开启后 `SHOW PROCESSLIST` 被**改写成一次对 PFS 表的 SELECT**
- **`INFORMATION_SCHEMA.PROCESSLIST` 已被标记废弃**（`sql/sql_show.cc` 里 `push_deprecated_warn(..., "performance_schema.processlist")`），并有全局计数器统计使用次数
- 术语变更：8.0 把 `master/slave` 改成 `source/replica`，State 名受 `terminology_use_previous` 影响

---

## 二、理论基础：两条实现路径与代价来源

### 2.1 ★ 关键事实：8.0.39 的 SHOW PROCESSLIST 有两条路径

`sql/sql_show.cc` 的分派：

```cpp
bool Sql_cmd_show_processlist::execute_inner(THD *thd) {
  /*
    If the Performance Schema is configured to support SHOW PROCESSLIST,
    then execute a query on performance_schema.processlist. Otherwise,
    fall back to the legacy method.
  */
  if (use_pfs()) {
    DEBUG_SYNC(thd, "pfs_show_processlist_performance_schema");
    return Sql_cmd_show::execute_inner(thd);
  } else {
    DEBUG_SYNC(thd, "pfs_show_processlist_legacy");
    mysqld_list_processes(thd, ...);
    return false;
  }
}
```

开关（`sql/sys_vars.cc`）：

```cpp
static Sys_var_bool Sys_pfs_processlist(
    "performance_schema_show_processlist",
    "Default startup value to enable SHOW PROCESSLIST "
    "in the performance schema.",
    GLOBAL_VAR(pfs_processlist_enabled), CMD_LINE(OPT_ARG), DEFAULT(false), ...);
```

⇒ **默认走 legacy 路径**（`performance_schema_show_processlist=OFF`）。

**PFS 路径做了什么**（`sql/sql_show_processlist.cc` 文档注释原文）：

```
  the parsed tree built for this query is in fact:

  SELECT * FROM
    (SELECT ID Id, USER User, HOST Host, DB db, COMMAND Command,
       TIME Time, STATE State, LEFT(INFO, <info_len>) Info
     FROM performance_schema.processlist)
    AS show_processlist;

  where info_len is 100 by default, otherwise 1024 for FULL PROCESSLIST.
```

⇒ **开启后，`SHOW PROCESSLIST` 完全退化为一次普通 SELECT**（有 join plan、可用索引、不遍历 THD）。

### 2.2 ★ legacy 路径的代价：不是"一把全局 mutex"，是分区锁 + 每 THD 多把锁

**遍历入口**（`sql/sql_show.cc`）：

```cpp
    Global_THD_manager::get_instance()->do_for_all_thd_copy(&list_process_list);
```

`sql/mysqld_thd_manager.cc`：

```cpp
void Global_THD_manager::do_for_all_thd_copy(Do_THD_Impl *func) {
  for (int i = 0; i < NUM_PARTITIONS; i++) {
    MUTEX_LOCK(lock_remove, &LOCK_thd_remove[i]);
    mysql_mutex_lock(&LOCK_thd_list[i]);

    /* Take copy of global_thread_list. */
    THD_array thd_list_copy(thd_list[i]);

    /*
      Allow inserts to global_thread_list. Newly added thd
      will not be accounted for when executing func.
    */
    mysql_mutex_unlock(&LOCK_thd_list[i]);

    std::for_each(thd_list_copy.begin(), thd_list_copy.end(), doit);
  }
}
```

（`NUM_PARTITIONS = 8`，见 `sql/mysqld_thd_manager.h`）

**精确的代价表述**（不要写成"持一把全局 mutex"）：

- **`LOCK_thd_remove[i]`（8 个分区之一）在整个遍历期间持有** ⇒ 阻塞该分区内所有 THD 的**注销/销毁**（连接退出路径）。这是 legacy 路径最主要的全局性代价
- `LOCK_thd_list[i]` 只在"复制指针数组"这一小段持有（`_copy` 后缀的意义），之后释放，允许新 THD 插入
- 复制出的快照**不含新建立的连接**（注释原文：`Newly added thd will not be accounted for when executing func.`）

**每个被检查的 THD 上还要持 3~4 把锁**（`List_process_list::operator()`）：

```cpp
    {
      MUTEX_LOCK(grd_secctx, &inspect_thd->LOCK_thd_security_ctx);
      ...
    }  // We've copied the security context, so release the lock.

    mysql_mutex_lock(&inspect_thd->LOCK_thd_data);
    ...
    thd_info->state_info = thread_state_info(m_client_thd, inspect_thd);
    mysql_mutex_unlock(&inspect_thd->LOCK_thd_data);

    mysql_mutex_lock(&inspect_thd->LOCK_thd_query);
```

`thread_state_info()` 内还有 `LOCK_current_cond`：

```cpp
static const char *thread_state_info(THD *invoking_thd, THD *inspected_thd) {
  if (inspected_thd->get_protocol()->get_rw_status()) {
    if (... == 2) return "Sending to client";
    if (inspected_thd->get_command() == COM_SLEEP) return "";
    return "Receiving from client";
  } else {
    MUTEX_LOCK(lock, &inspected_thd->LOCK_current_cond);
    const char *proc_info = inspected_thd->proc_info_session(invoking_thd);
    if (proc_info) return proc_info;
    if (inspected_thd->current_cond.load()) return "Waiting on cond";
    return nullptr;
  }
}
```

⇒ **代价模型**：对 N 个会话，要做 `N × (LOCK_thd_security_ctx + LOCK_thd_data + LOCK_current_cond + LOCK_thd_query)` 次加锁，并全程持有 `LOCK_thd_remove[partition]`。**会话数越大、并发越高，对连接建立/销毁路径的干扰越大；高并发短连接场景尤其明显。**

一个有意思的旁证——源码里这个调试点的名字本身就说明问题：

```cpp
    DBUG_EXECUTE_IF("processlist_acquiring_dump_threads_LOCK_thd_data", {
      if (inspect_thd->get_command() == COM_BINLOG_DUMP || ...)
        DEBUG_SYNC(m_client_thd, "processlist_after_LOCK_thd_list_before_LOCK_thd_data");
    });
```

⇒ **SHOW PROCESSLIST 卡在 `LOCK_thd_data` 上是真实存在过的场景**（binlog dump 线程）。

⚠️ **源码无直接证据的部分**：没有形如 "SHOW PROCESSLIST is expensive / 不要频繁执行" 的显式注释。上述代价结论由加锁路径推导，**不伪造成官方结论**。

### 2.3 PFS 路径不需要任何这些锁

`table_threads::make_row()` / `table_processlist::make_row()` 全部用**乐观锁 + 快照拷贝**（见第五节），不持 `LOCK_thd_*`，不阻塞连接退出。

### 2.4 设计权衡：为什么要有两条路径

PFS 路径是**可选的替代实现**（开关注释称其为 "Default startup value to enable SHOW PROCESSLIST in the performance schema"）。保留 legacy 是兼容性考虑；提供 PFS 路径是为了消除遍历 THD 的锁代价。

sys schema 的视图注释直接点名了这个动机（`scripts/sys_schema/views/p_s/processlist.sql`）：

```sql
-- A detailed non-blocking processlist view to replace
-- [INFORMATION_SCHEMA. | SHOW FULL] PROCESSLIST
--
-- Performs less locking than the legacy sources, whilst giving extra information.
```

---

## 三、入口一：`SHOW PROCESSLIST`（legacy，默认）

### 3.1 源码位置

| 文件 | 角色 |
|---|---|
| `sql/parse_tree_nodes.h` / `.cc` | `PT_show_processlist` 解析节点，`make_cmd()` 决定走 PFS 还是 legacy |
| `sql/sql_show_processlist.cc` | PFS 路径：把 SHOW 改写成 SELECT |
| `sql/sql_show.h` / `sql_show.cc` | legacy 路径：`mysqld_list_processes()` / `List_process_list` |
| `sql/sql_parse.cc` | `COM_PROCESS_INFO`（`mysqladmin processlist` 走的协议命令） |

命令类（`sql/sql_show.h`）：

```cpp
class Sql_cmd_show_processlist : public Sql_cmd_show {
 public:
  Sql_cmd_show_processlist() : Sql_cmd_show(SQLCOM_SHOW_PROCESSLIST) {}
  explicit Sql_cmd_show_processlist(bool verbose) : ..., m_verbose(verbose) {}
  void set_use_pfs(bool use_pfs) { m_use_pfs = use_pfs; }
  bool verbose() const { return m_verbose; }
};
```

### 3.2 输出字段与截断（★ 两条路径不一样，容易踩坑）

**legacy 路径**（`sql/sql_show.cc`）：

```cpp
  size_t max_query_length =
      (verbose ? thd->variables.max_allowed_packet : PROCESS_LIST_WIDTH);
```

- 非 FULL = **100**（`PROCESS_LIST_WIDTH`）
- FULL = **`max_allowed_packet`**（不是 1024！）

字段列表：

```cpp
  field_list.push_back(new Item_int(NAME_STRING("Id"), 0, MY_INT64_NUM_DECIMAL_DIGITS));
  field_list.push_back(new Item_empty_string("User", USERNAME_CHAR_LENGTH));
  field_list.push_back(new Item_empty_string("Host", HOSTNAME_LENGTH));
  field_list.push_back(field = new Item_empty_string("db", NAME_CHAR_LEN));
  field_list.push_back(new Item_empty_string("Command", 16));
  field_list.push_back(field = new Item_return_int("Time", 7, MYSQL_TYPE_LONG));
  field_list.push_back(field = new Item_empty_string("State", 30));   // ★ VARCHAR(30)
  field_list.push_back(new Item_empty_string("Info", max_query_length));
```

⚠️ **legacy 的 `State` 只有 30 字符宽**，而 PFS 是 64 ⇒ 长 state 名在 legacy 路径会被截断。

**PFS 路径**（`sql/sql_show_processlist.cc`）：

```cpp
  assert(PROCESS_LIST_WIDTH == 100);
  if (verbose) { ... "1024" ... } else { ... "100" ... }
```

- 非 FULL = 100；FULL = **1024**（通过 `LEFT(INFO, 1024)` 实现）

### 3.3 ★ 权限：谁能看谁的

**语句级前置检查**（`sql/sql_show.cc`）：

```cpp
bool Sql_cmd_show_processlist::check_privileges(THD *thd) {
  if (!thd->security_context()->priv_user().str[0] &&
      check_global_access(thd, PROCESS_ACL))
    return true;
  return Sql_cmd_show::check_privileges(thd);
}
```

⇒ **匿名用户（没有 priv_user）必须显式持有 `PROCESS` 权限**，否则直接拒绝执行。

**行级过滤**：

```cpp
    mysqld_list_processes(thd,
                          thd->security_context()->check_access(PROCESS_ACL)
                              ? NullS
                              : thd->security_context()->priv_user().str,
                          m_verbose, true);
```

回调里：

```cpp
      if (!inspect_thd->is_connected(true) ||
          (m_user && (inspect_thd->system_thread || !inspect_sctx_user.str ||
                      strcmp(inspect_sctx_user.str, m_user)))) return;
```

**语义**：

- 有 `PROCESS` → `m_user == NullS` → 看到**所有已连接会话，包括系统线程**
- 无 `PROCESS` → `m_user = priv_user` → 只看到 **user 名与自己完全相同**的会话；**系统线程一律不可见**
- ⚠️ **只比较 user，不比较 host** ⇒ 同名不同来源的账号之间能看到彼此

`mysqladmin processlist`（`COM_PROCESS_INFO`）逻辑相同，但 `verbose` 固定为 `false`（100 字符截断），且已 deprecated。

---

## 四、入口二：`INFORMATION_SCHEMA.PROCESSLIST`（已废弃）

### 4.1 ★ 纠偏：它不是 DD system view

**源码无 `sql/dd/impl/system_views/processlist.cc`**（按 `*processlist*` 搜索，sql/dd 下 0 命中）。它是**遗留 I_S 表**，定义在 `sql/sql_show.cc`：

```cpp
    {"PROCESSLIST", processlist_fields_info, fill_schema_processlist,
     make_old_format, nullptr, false},
```

### 4.2 字段定义

```cpp
ST_FIELD_INFO processlist_fields_info[] = {
    {"ID", 21, MYSQL_TYPE_LONGLONG, 0, MY_I_S_UNSIGNED, "Id", 0},
    {"USER", USERNAME_CHAR_LENGTH, MYSQL_TYPE_STRING, 0, 0, "User", 0},
    {"HOST", HOST_AND_PORT_LENGTH, MYSQL_TYPE_STRING, 0, 0, "Host", 0},
    {"DB", NAME_CHAR_LEN, MYSQL_TYPE_STRING, 0, 1, "Db", 0},
    {"COMMAND", 16, MYSQL_TYPE_STRING, 0, 0, "Command", 0},
    {"TIME", 7, MYSQL_TYPE_LONG, 0, 0, "Time", 0},
    {"STATE", 64, MYSQL_TYPE_STRING, 0, 1, "State", 0},
    {"INFO", PROCESS_LIST_INFO_WIDTH, MYSQL_TYPE_STRING, 0, 1, "Info", 0},
    {nullptr, 0, MYSQL_TYPE_STRING, 0, 0, nullptr, 0}};
```

⇒ 8 列；`INFO` 宽度 = `PROCESS_LIST_INFO_WIDTH` = **65535**（不截断）。

（`PROCESS_LIST_WIDTH`=100 用于 SHOW；`PROCESS_LIST_INFO_WIDTH`=65535 用于 I_S，见 `sql/sql_show.h`）

### 4.3 与 SHOW PROCESSLIST 的关系：复用同一套遍历，但代码是复制的两份

```cpp
static int fill_schema_processlist(THD *thd, Table_ref *tables, Item *) {
  Fill_process_list fill_process_list(thd, tables);
  if (!thd->killed) {
    Global_THD_manager::get_instance()->do_for_all_thd_copy(&fill_process_list);
  }
  return 0;
}
```

⇒ **同一个 `do_for_all_thd_copy()`**，同样的 `LOCK_thd_remove` + 每 THD 多把锁。

`Fill_process_list::operator()` 与 `List_process_list::operator()` **逻辑逐字基本相同**（同一套注释、同一 `DBUG_EXECUTE_IF`、同样的加锁序列、同样的 `thread_state_info()`），只是写入目标从 protocol 变成 `TABLE`。

差异只有两处：截断（65535 不截断）；权限判断位置（在回调内部现算）。

### 4.4 已被官方标记废弃

两条路径都会 push 警告：

```cpp
    push_deprecated_warn(m_client_thd, "INFORMATION_SCHEMA.PROCESSLIST",
                         "performance_schema.processlist");
    deprecated_use_i_s_processlist_last_timestamp = my_micro_time();
    deprecated_use_i_s_processlist_count++;
```

并有全局计数器：

```cpp
/** Count number of times information_schema.processlist has been used. */
std::atomic_ulong deprecated_use_i_s_processlist_count = 0;
/** Last time information_schema.processlist was used, as usec since epoch. */
std::atomic_ullong deprecated_use_i_s_processlist_last_timestamp = 0;
```

⇒ **不要用**。代价与 legacy SHOW 同量级，而且官方在统计你的使用次数（准备移除）。

---

## 五、入口三：★ `performance_schema.threads`（推荐）

### 5.1 表定义（`storage/perfschema/table_threads.cc`，24 列）

```cpp
    "  THREAD_ID BIGINT unsigned not null,\n"
    "  NAME VARCHAR(128) not null,\n"
    "  TYPE VARCHAR(10) not null,\n"
    "  PROCESSLIST_ID BIGINT unsigned,\n"
    "  PROCESSLIST_USER VARCHAR(32),\n"
    "  PROCESSLIST_HOST VARCHAR(255) CHARACTER SET ASCII default null,\n"
    "  PROCESSLIST_DB VARCHAR(64),\n"
    "  PROCESSLIST_COMMAND VARCHAR(16),\n"
    "  PROCESSLIST_TIME BIGINT,\n"
    "  PROCESSLIST_STATE VARCHAR(64),\n"
    "  PROCESSLIST_INFO LONGTEXT,\n"
    "  PARENT_THREAD_ID BIGINT unsigned,\n"
    "  `ROLE` VARCHAR(64),\n"
    "  INSTRUMENTED ENUM ('YES', 'NO') not null,\n"
    "  HISTORY ENUM ('YES', 'NO') not null,\n"
    "  CONNECTION_TYPE VARCHAR(16),\n"
    "  THREAD_OS_ID BIGINT unsigned,\n"
    "  RESOURCE_GROUP VARCHAR(64),\n"
    "  EXECUTION_ENGINE ENUM ('PRIMARY', 'SECONDARY'),\n"
    "  CONTROLLED_MEMORY BIGINT unsigned not null,\n"
    "  MAX_CONTROLLED_MEMORY BIGINT unsigned not null,\n"
    "  TOTAL_MEMORY BIGINT unsigned not null,\n"
    "  MAX_TOTAL_MEMORY BIGINT unsigned not null,\n"
    "  TELEMETRY_ACTIVE ENUM ('YES', 'NO') not null,\n"
    "  PRIMARY KEY (THREAD_ID) USING HASH,\n"
    "  KEY (PROCESSLIST_ID) USING HASH,\n"
    "  KEY (THREAD_OS_ID) USING HASH,\n"
    "  KEY (NAME) USING HASH,\n"
    "  KEY `PROCESSLIST_ACCOUNT` (PROCESSLIST_USER, PROCESSLIST_HOST) USING HASH,\n"
    "  KEY (PROCESSLIST_HOST) USING HASH,\n"
    "  KEY (RESOURCE_GROUP) USING HASH\n",
```

索引（PFS 语义下即"可下推的过滤键"）：`THREAD_ID`（主键）、`PROCESSLIST_ID`、`THREAD_OS_ID`、`NAME`、`(USER, HOST)`、`HOST`、`RESOURCE_GROUP`。

### 5.2 字段语义（逐列）

| 列 | 取值 | 说明 |
|---|---|---|
| `THREAD_ID` | `pfs->m_thread_internal_id` | PFS 内部**单调递增**的线程号；**连接断开后不复用**（与 PROCESSLIST_ID 相反）。前后台都有 |
| `NAME` | `safe_class->m_name` | 如 `thread/sql/one_connection`、`thread/innodb/...` |
| `TYPE` | **派生**：`m_processlist_id != 0 ? "FOREGROUND" : "BACKGROUND"` | 见 5.5 |
| `PROCESSLIST_ID` | `pfs->m_processlist_id`，0 → NULL | = SHOW 的 `Id`，**KILL 用的 id** |
| `PROCESSLIST_USER` / `_HOST` | 长度 0 → NULL | |
| `PROCESSLIST_COMMAND` | 后台线程为 NULL；脏读后 sanitize（`> COM_END` 时置 `COM_END`） | |
| `PROCESSLIST_TIME` | `now - m_start_time`；`m_start_time==0` → NULL | **空闲连接为 NULL（不是 0）** |
| `PROCESSLIST_STATE` | `find_stage_class(pfs->m_stage)->m_name` 剥前缀 | 超 64 字符**静默截断** |
| `PROCESSLIST_INFO` | LONGTEXT | **不截断**（比 SHOW 完整得多） |
| `PARENT_THREAD_ID` | 0 → NULL | 父子线程关系 |
| `ROLE` | 恒 NULL | 保留列 |
| `INSTRUMENTED` / `HISTORY` | 可 UPDATE | 见 5.6 |
| `CONNECTION_TYPE` | TCP/IP / Socket / SSL … | |
| `THREAD_OS_ID` | `pfs->m_thread_os_id`，0 → NULL | 见 5.4 |
| `RESOURCE_GROUP` | Resource Group 名 | |
| `EXECUTION_ENGINE` | `PRIMARY` / `SECONDARY` | secondary engine（如 HeatWave） |
| `CONTROLLED_MEMORY` / `MAX_*` / `TOTAL_MEMORY` / `MAX_TOTAL_MEMORY` | `&pfs->m_session_all_memory_stat` | 见 5.7 |

### 5.3 ★ 为什么推荐：不持全局锁

`make_row()` 全程**乐观锁 + 局部拷贝**：

```cpp
int table_threads::make_row(PFS_thread *pfs) {
  pfs_optimistic_state lock, session_lock, stmt_lock;

  /* Protect this reader against thread termination */
  pfs->m_lock.begin_optimistic_lock(&lock);
  /* Protect this reader against session attribute changes */
  pfs->m_session_lock.begin_optimistic_lock(&session_lock);
  ...
  if (!pfs->m_session_lock.end_optimistic_lock(&session_lock)) {
    /*
      One of the columns: PROCESSLIST_USER / PROCESSLIST_HOST
      is being updated.
      Do not discard the entire row.
      Do not loop waiting for a stable value.
      Just return NULL values.
    */
    m_row.m_user_name.reset();
    m_row.m_host_name.reset();
  }
  ...
  if (!pfs->m_lock.end_optimistic_lock(&lock)) return HA_ERR_RECORD_DELETED;
```

⇒ **重要语义**：它不是"读到一致快照"，而是"读到一半发现被改写就把这一组列置 NULL / 丢弃整行重试"。**`PROCESSLIST_USER/HOST/DB/INFO` 在高压力下可能读到 NULL**——这是设计使然，不是 bug。

### 5.4 ★ `THREAD_OS_ID`：唯一能连到 OS 层的列

来源（`storage/perfschema/pfs.cc`）：

```cpp
void pfs_set_thread_os_id_vc(PSI_thread *thread) {
  auto *pfs = reinterpret_cast<PFS_thread *>(thread);
  if (unlikely(pfs == nullptr)) return;
  pfs->m_thread_os_id = my_thread_os_id();
}
```

⇒ **`THREAD_OS_ID` = `my_thread_os_id()`**，即 OS 层的 **tid**（Linux 下就是 `gettid()`，也就是 `ps -L` / `top -H` 里的 TID，等于 `/proc/<pid>/task/<tid>`）。

**联动方式**（这是 SHOW / I_S 完全没有的能力）：

```sql
SELECT thread_id, processlist_id, thread_os_id, name, type
  FROM performance_schema.threads WHERE processlist_id = CONNECTION_ID();
```

拿到 `thread_os_id` 后可：

- `perf top -t <tid>` / `perf record -t <tid>`（按 tid 采样）
- `pidstat -t -p <pid>` 看该线程 CPU
- eBPF 工具（`offcputime` / `biolatency`）按 tid 过滤
- `cat /proc/<pid>/task/<tid>/stack`、`.../status`
- 与 `ps -L -p <pid>` 的 LWP 列直接对齐

### 5.5 ★ `TYPE='BACKGROUND'`：只有它能看到后台线程

```cpp
        case 2: /* TYPE */
          if (m_row.m_processlist_id != 0) {
            set_field_varchar_utf8mb4(f, "FOREGROUND", 10);
          } else {
            set_field_varchar_utf8mb4(f, "BACKGROUND", 10);
          }
```

**反证**：同一 PFS 引擎的 `performance_schema.processlist` 表**主动过滤掉后台线程**（`table_processlist.cc`）：

```cpp
  /* Ignore background threads. */
  if (pfs->m_user_name.length() == 0 || pfs->m_processlist_id == 0)
    return HA_ERR_RECORD_DELETED;
```

**旁证**：后台线程的 instrumented 默认开启（`pfs.cc`）：

```cpp
    } else {
      /* There is no setting for background threads */
      enabled = true;
      history = true;
    }
```

后台线程在 `NAME` 里以 `thread/sql/<name>`（如 `thread/sql/main`、`thread/sql/signal_handler`）或 `thread/innodb/...` 出现。

⇒ **`SHOW PROCESSLIST` / `I_S.PROCESSLIST` / `PFS.processlist` 都看不到后台线程**；只有 `performance_schema.threads` 能看 `TYPE='BACKGROUND'`（InnoDB 的 io / log / purge 线程、主线程等）。**这是它最不可替代的能力。**

### 5.6 `INSTRUMENTED` / `HISTORY`：可以单独关掉某个线程的采集

`table_threads` 是**可更新**表（`&pfs_updatable_acl`），只允许改这两列：

```cpp
      switch (f->field_index()) {
        case 13: /* INSTRUMENTED */
          value = (enum_yes_no)get_field_enum(f);
          m_row.m_psi->set_enabled(value == ENUM_YES);
          break;
        case 14: /* HISTORY */
          value = (enum_yes_no)get_field_enum(f);
          m_row.m_psi->set_history(value == ENUM_YES);
          break;
        default: return HA_ERR_WRONG_COMMAND;
      }
```

典型用法（压测时降低开销）：

```sql
UPDATE performance_schema.threads SET INSTRUMENTED='NO' WHERE ...;
UPDATE performance_schema.threads SET HISTORY='NO' WHERE ...;
```

权限：PFS 表的写操作走**表级 `UPDATE` 权限**（`pfs_updatable_acl` 只允许 TRUNCATE 之外的特定权限），不是 `PROCESS` 权限。

### 5.7 内存列（8.0.28+）

```cpp
        case 19: /* CONTROLLED_MEMORY */
        case 20: /* MAX_CONTROLLED_MEMORY */
        case 21: /* TOTAL_MEMORY */
        case 22: /* MAX_TOTAL_MEMORY */
          m_row.m_session_all_memory_row.set_field(f->field_index() - 19, f);
```

- `CONTROLLED_MEMORY`：被 PFS 追踪、受控的那部分内存（当前值）
- `MAX_CONTROLLED_MEMORY`：峰值
- `TOTAL_MEMORY` / `MAX_TOTAL_MEMORY`：含未受控部分的合计 / 峰值

⚠️ 只有在 **memory instrumentation 开启**时才有非零值（`setup_instruments` 里 `memory/%` 默认为关）。

---

## 六、入口四：`sys.session` / `sys.processlist`（人工排障首选）

`sys.session` 是 `sys.processlist` 的过滤版：

```sql
CREATE OR REPLACE ... VIEW session AS
SELECT * FROM sys.processlist
WHERE conn_id IS NOT NULL AND command != 'Daemon';
```

`sys.processlist` 有 **27 列**，基于 `performance_schema.threads` LEFT JOIN 6 张表（见 [`sys_schema.md`](sys_schema.md) 第四节的完整剖析）：`events_waits_current`（当前在等什么 + `source` 文件行号）、`events_stages_current`（阶段 + `progress` 百分比）、`events_statements_current`（语句/锁等待/CPU/临时表/全表扫描）、`events_transactions_current`（事务）、`x$memory_by_thread_by_current_bytes`（内存）、`session_connect_attrs` ×2（客户端 `pid` / `program_name`）。

⚠️ **两点代价**：

1. `ALGORITHM = TEMPTABLE` + 6 个 LEFT JOIN + `ORDER BY` ⇒ **本身不便宜，不适合高频采集**
2. **依赖 `events_*_current` consumer 开启**，否则那些列全是 NULL

⇒ **定位**：人工排障首选（信息最全、可读性好）；**监控采集请用 `performance_schema.threads`**。

---

## 七、★ State 字段体系

### 7.1 写入端：`THD::enter_stage()` 同时写两处

`sql/sql_class.cc`：

```cpp
  if (new_stage != nullptr) {
    const char *msg = new_stage->m_name;
    ...
    m_current_stage_key = new_stage->m_key;
    set_proc_info(msg);                       // ← 写 THD::m_proc_info（给 SHOW PROCESSLIST）

    m_stage_progress_psi =
        MYSQL_SET_STAGE(m_current_stage_key, calling_file, calling_line);   // ← 写 PFS m_stage
  }
```

⇒ 同一个 `PSI_stage_info::m_name` 被写到两个地方。

还有术语兼容层（`master/slave → source/replica`）：

```cpp
const char *THD::proc_info(const System_variables &sysvars) const {
  ...
  if ((ret != nullptr) && (version != terminology_use_previous::NONE)) {
    auto compatible_name_info =
        terminology_use_previous::lookup(PFS_CLASS_STAGE, ret, false);
    ...
    if (version <= compatible_name_info.version) ret = compatible_name_info.old_name;
  }
  return ret;
}
```

### 7.2 读取端（PFS）：剥前缀 + 静默截到 64

```cpp
  stage_class = find_stage_class(pfs->m_stage);
  if (stage_class != nullptr) {
    m_row.m_processlist_state_ptr =
        stage_class->m_name.str() + stage_class->m_prefix_length;
    m_row.m_processlist_state_length =
        stage_class->m_name.length() - stage_class->m_prefix_length;
    if (m_row.m_processlist_state_length > 64) {
      /*
        Column PROCESSLIST_STATE is VARCHAR(64)
        for compatibility reasons with the historical
        INFORMATION_SCHEMA.PROCESSLIST table.
        Stages however can have longer names.
        We silently truncate data here,
        so it fits into PROCESSLIST_STATE.
      */
      m_row.m_processlist_state_length = 64;
    }
  }
```

⇒ PFS 里 state 名带 `stage/sql/`、`stage/innodb/` 前缀，展示时剥掉；超 64 **静默截断**。

### 7.3 ★ State 的更新**不依赖** instrument 开关（关键证据）

`storage/perfschema/pfs.cc` `pfs_start_stage_v1()`：

```cpp
  /* Always update column threads.processlist_state. */
  pfs_thread->m_stage = key;
  /* Default value when the stage is not instrumented for progress */
  pfs_thread->m_stage_progress = nullptr;

  if (!flag_global_instrumentation) {
    return nullptr;
  }
  if (flag_thread_instrumentation && !pfs_thread->m_enabled) {
    return nullptr;
  }
  ...
```

**含义**：

- `pfs_thread->m_stage = key;` 发生在**所有 instrument 开关判断之前**
- ⇒ 即使全局/线程 instrument 关闭、即使 `events_stages_current` consumer 关闭，**`threads.PROCESSLIST_STATE` 依然会被更新**
- 被开关关掉的只是**往 `events_stages_*` 表写事件**（计时、统计、history），**不是 state 列本身**

⇒ 这就是"PFS 里查不到 stage 事件，但 `SHOW PROCESSLIST` / `threads.PROCESSLIST_STATE` 却看得见"的原因。

### 7.4 ★ State 与 SHOW PROCESSLIST 的 State **不是完全同一个来源**

共同上游都是 `enter_stage()`，但 **SHOW 走 `thread_state_info()`，优先级不同**：

```cpp
static const char *thread_state_info(THD *invoking_thd, THD *inspected_thd) {
  if (inspected_thd->get_protocol()->get_rw_status()) {
    if (... == 2) return "Sending to client";
    if (inspected_thd->get_command() == COM_SLEEP) return "";
    return "Receiving from client";
  } else {
    MUTEX_LOCK(lock, &inspected_thd->LOCK_current_cond);
    const char *proc_info = inspected_thd->proc_info_session(invoking_thd);
    if (proc_info) return proc_info;
    if (inspected_thd->current_cond.load()) return "Waiting on cond";
    return nullptr;
  }
}
```

**五点差异**：

1. **网络收发期间**：SHOW 显示 `Sending to client` / `Receiving from client`，而 PFS 显示最后的 stage
2. SHOW 有 `Waiting on cond` 这个 PFS 没有的回退值
3. SHOW 受 `terminology_use_previous` 影响显示旧名；**PFS 表不受影响**
4. `Killed` 是 `List_process_list` 里硬写的（`if (inspect_thd->killed == THD::KILL_CONNECTION) thd_info->proc_info = "Killed";`），PFS 侧无对应
5. legacy 的 State 宽 30，PFS 宽 64

⇒ **排查时以 `PFS.threads.PROCESSLIST_STATE` 为准**（更宽、不受术语回退影响、可与 `events_stages_*` 对齐）。

### 7.5 State 清单

#### SQL 层（`sql/mysqld.cc` `all_server_stages[]`，约 100 项）

注册：`mysql_stage_register(category, all_server_stages, count);`

**与 DDL / 运维最相关的**：

| State | 含义 |
|---|---|
| `preparing for alter table` | INPLACE 的 `::prepare()` |
| `altering table` | INPLACE 的 `::inplace_alter_table()`（真正建索引） |
| `committing alter table to storage engine` | INPLACE 的 `::commit()` |
| **`copy to tmp table`** | COPY 算法拷数据（**SQL 层唯一带 progress 的 stage**） |
| `creating table` / `After create` | CREATE TABLE 及收尾 |
| `rename` / `rename result table` | RENAME / COPY 换名 |
| `manage keys` | COPY 收尾 / MERGE 表 key 处理 |
| `setup` | TRUNCATE / 分区维护 |
| `discard_or_import_tablespace` | DISCARD / IMPORT TABLESPACE |
| `System lock` | `lock_tables` 阶段 |
| `checking permissions` / `Opening tables` / `closing tables` / `query end` / `freeing items` / `cleaning up` | 通用语句阶段 |
| `optimizing` / `preparing` / `statistics` / `executing` / `update` / `updating` / `Sending data` 相关 | 执行各阶段 |
| `waiting for handler commit` | 通用提交路径 |
| `Waiting for table flush` / `Waiting for GTID to be committed` / `Waiting for preceding transaction to commit` | 各类等待 |
| `Applying batch of row changes (write|update|delete)` | **复制 applier**（带 progress） |
| `Searching rows for update` / `Sending binlog event to replica` / `Reading event from the relay log` | 复制相关 |

⚠️ 三个**不是 stage instrument** 的伪 state（由 `thread_state_info()` 硬编码返回）：**`Sending to client`、`Receiving from client`、`Waiting on cond`**（外加 `Killed` 和 COM_SLEEP 时的 `""`）。

#### InnoDB 侧（`storage/innobase/srv/srv0start.cc` `srv_stages[]`，**12 项**）

⚠️ **纠偏：是 12 个，不是 13 个**（以源码数组为准）。

| 变量 | 名称字符串 | progress |
|---|---|---|
| `srv_stage_alter_table_read_pk_internal_sort` | `alter table (read PK and internal sort)` | ✅ |
| `srv_stage_alter_table_merge_sort` | `alter table (merge sort)` | ✅ |
| `srv_stage_alter_table_insert` | `alter table (insert)` | ✅ |
| `srv_stage_alter_table_flush` | `alter table (flush)` | ✅ |
| `srv_stage_alter_table_log_index` | `alter table (log apply index)` | ✅ |
| `srv_stage_alter_table_log_table` | `alter table (log apply table)` | ✅ |
| `srv_stage_alter_table_end` | `alter table (end)` | ✅ |
| `srv_stage_alter_tablespace_encryption` | `alter tablespace (encryption)` | ✅ |
| `srv_stage_buffer_pool_load` | `buffer pool load` | ✅ |
| `srv_stage_clone_file_copy` | `clone (file copy)` | ✅ |
| `srv_stage_clone_redo_copy` | `clone (redo copy)` | ✅ |
| `srv_stage_clone_page_copy` | `clone (page copy)` | ✅ |

```cpp
static PSI_stage_info *srv_stages[] = {
    &srv_stage_alter_table_end,          &srv_stage_alter_table_flush,
    &srv_stage_alter_table_insert,       &srv_stage_alter_table_log_index,
    &srv_stage_alter_table_log_table,    &srv_stage_alter_table_merge_sort,
    &srv_stage_alter_table_read_pk_internal_sort,
    &srv_stage_alter_tablespace_encryption, &srv_stage_buffer_pool_load,
    &srv_stage_clone_file_copy,          &srv_stage_clone_redo_copy,
    &srv_stage_clone_page_copy,
};
```

全部带 `PSI_FLAG_STAGE_PROGRESS` ⇒ **都可通过 `events_stages_current` 的 `WORK_COMPLETED/WORK_ESTIMATED` 算百分比**（`sys.processlist` 的 `progress` 列就是这么来的）。

⇒ **INPLACE DDL 的进度靠这 7 个 `alter table (...)` stage**；COPY DDL 靠 `stage/sql/copy to tmp table`。

---

## 八、四种入口横向对照

| 维度 | `SHOW PROCESSLIST`<br>(legacy，默认) | `SHOW PROCESSLIST`<br>(PFS 模式) | `I_S.PROCESSLIST` | **`PFS.threads`** | `sys.session` |
|---|---|---|---|---|---|
| 实现文件 | `sql/sql_show.cc` | `sql/sql_show_processlist.cc` | `sql/sql_show.cc` | `storage/perfschema/table_threads.cc` | `views/p_s/session.sql` |
| **持全局/分区锁** | **是**（`LOCK_thd_remove` + 每 THD 4 把） | 否 | **是**（同 legacy） | **否**（乐观锁） | 否 |
| **含后台线程** | 否 | 否 | 否 | **是**（`TYPE='BACKGROUND'`） | 否 |
| 字段数 | 8 | 8 | 8 | **24** | 27 |
| 需 PROCESS 权限 | **是** | 是 | 是 | **否**（SELECT 权限即可） | 否 |
| 有 `THREAD_OS_ID` | 否 | 否 | 否 | **是** | 否（有 `pid`/`program_name`） |
| 内存/等待/事务 | 否 | 否 | 否 | 内存 4 列 | **是**（最全） |
| INFO 截断 | 100 / `max_allowed_packet` | 100 / **1024** | **65535** 不截断 | **不截断** | 经 `format_statement` 截断到 64（可配） |
| State 宽度 | **30** | 64 | 64 | 64 | 64 |
| 已废弃 | 否 | 否 | **是** | 否 | 否 |
| **推荐场景** | 人工临时看一眼，**别进监控** | 兼容老脚本的过渡方案 | **不要用** | **★ 默认推荐**（采集/OS 联动/后台线程） | **人工排障首选**（别高频采集） |

---

## 九、可观测性

### 系统变量与状态变量

| 变量名 | 默认 | 作用域 | 说明 |
|---|---|---|---|
| `performance_schema_show_processlist` | **false** | Global | ★ 开启后 `SHOW PROCESSLIST` 走 PFS 路径（消除遍历 THD 的锁代价） |
| `performance_schema_session_connect_attrs_size` | 自动 | Global（只读） | 每线程连接属性缓冲；超限的属性被丢弃且**整行不返回** |
| `terminology_use_previous` | NONE | Global/Session | 影响 `SHOW PROCESSLIST` 的 State 显示旧名（PFS 表不受影响） |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 现在有谁在跑（采集） | SQL | **`performance_schema.threads`** |
| 现在有谁在跑（人看） | SQL | `sys.session` |
| **后台线程在干嘛** | SQL | `performance_schema.threads WHERE TYPE='BACKGROUND'`（**唯一入口**） |
| 要连到 OS 层排查 | SQL + 工具 | `threads.THREAD_OS_ID` → `perf -t` / `pidstat -t` / eBPF |
| 某个连接的等待/事务/内存 | SQL | `sys.session`，或 JOIN `events_waits_current` 等 |
| 账号/来源维度的连接数 | SQL | `performance_schema.accounts` / `users` / `hosts` |
| 客户端进程信息 | SQL | `performance_schema.session_connect_attrs`（`_pid` / `program_name`） |
| 要 KILL | SQL | 用 **`PROCESSLIST_ID`**（不能用 `THREAD_ID`） |

### 典型查询

```sql
-- ① 采集用（便宜、不持锁）：当前活跃会话
SELECT thread_id, processlist_id AS id, processlist_user, processlist_host,
       processlist_db, processlist_command, processlist_time,
       processlist_state, thread_os_id
  FROM performance_schema.threads
 WHERE type = 'FOREGROUND' AND processlist_command NOT IN ('Sleep','Daemon')
 ORDER BY processlist_time DESC;

-- ② ★ 看后台线程（SHOW PROCESSLIST 完全看不到）
SELECT thread_id, name, type, processlist_state, thread_os_id
  FROM performance_schema.threads WHERE type = 'BACKGROUND';

-- ③ 拿 OS tid 去 perf/eBPF
SELECT processlist_id, thread_os_id FROM performance_schema.threads
 WHERE processlist_id = CONNECTION_ID();

-- ④ 按账号看连接数（比扫 processlist 便宜得多）
SELECT user, host, current_connections, total_connections
  FROM performance_schema.accounts ORDER BY current_connections DESC;

-- ⑤ KILL 前确认身份（防止 id 已换人）
SELECT processlist_id, processlist_user, processlist_host,
       processlist_time, processlist_info
  FROM performance_schema.threads WHERE processlist_id = 123;
```

---

## 十、★ 工程实现技法

### 10.1 乐观锁 + "置 NULL 而非自旋等待"

`table_threads::make_row()` 的注释是极好的设计材料：

```cpp
    /*
      One of the columns: PROCESSLIST_USER / PROCESSLIST_HOST
      is being updated.
      Do not discard the entire row.
      Do not loop waiting for a stable value.
      Just return NULL values.
    */
```

⇒ **放弃一致性，用"部分列置 NULL"换取不阻塞写者**。这是 PFS"优先优化采集路径"原则在读取侧的体现（与 `pfs/pfs_events.md` 里"脏读 + 消毒"是同一套思路）。

代价：**高压力下 `PROCESSLIST_USER/HOST/DB/INFO` 可能读到 NULL**。

### 10.2 遍历 THD 的"先复制再释放"模式

`do_for_all_thd_copy()` 只短暂持 `LOCK_thd_list` 复制指针数组，然后释放、再逐项执行 ⇒ 允许新 THD 插入，但**快照不含新连接**。这是"缩小临界区"的经典手法。

（对比：全程持有的 `LOCK_thd_remove` 才是真正代价所在——它阻塞连接销毁。）

### 10.3 派生列 `TYPE` 而非存储列

`TYPE` 不存数据，由 `m_processlist_id != 0` 现算 ⇒ 省一列存储，且语义永远自洽。

### 10.4 可更新表实现"单线程粒度的开关"

`threads.INSTRUMENTED` / `HISTORY` 可 UPDATE ⇒ 能针对**单个线程**关掉采集（如排除监控连接自身），这是 `setup_instruments`（按 instrument 维度）做不到的粒度。

`sys.diagnostics()` 就用了这个：先把当前线程 `INSTRUMENTED` 关掉，避免采集到自己。

### 10.5 废弃机制的"计数器 + 警告"而非直接移除

`I_S.PROCESSLIST` 用 `push_deprecated_warn` + `deprecated_use_i_s_processlist_count` 统计 ⇒ 给用户迁移窗口，同时让官方知道还有多少人在用。

---

## 十一、Misc（易混淆点）

| 混淆 | 澄清 |
|---|---|
| `SHOW PROCESSLIST` 一定很慢？ | ⚠️ **8.0 有两条路径**。默认 legacy（持 `LOCK_thd_remove` + 每 THD 4 把锁）；开 `performance_schema_show_processlist=ON` 后退化为普通 SELECT |
| legacy 持的是"一把全局 mutex"？ | ❌ **不是**。是 **8 个分区之一的 `LOCK_thd_remove`**（阻塞连接销毁）+ 遍历期间每 THD 的 4 把锁 |
| `I_S.PROCESSLIST` 是 DD system view？ | ❌ **不是**，是遗留 I_S 表（`sql/sql_show.cc`），且**已废弃** |
| 四种入口都能看后台线程？ | ❌ **只有 `PFS.threads`**（`TYPE='BACKGROUND'`）；`PFS.processlist` 显式跳过 |
| 看别人的会话需要 PROCESS？ | `SHOW`/`I_S` **需要**；**`PFS.threads` 不需要**（SELECT 权限即可） |
| 无 PROCESS 时能看系统线程吗？ | ❌ 一律不可见（`inspect_thd->system_thread` 直接 return） |
| 无 PROCESS 时按 user 还是 user@host 过滤？ | ⚠️ **只比 user 不比 host** ⇒ 同名不同来源的账号互相可见 |
| `KILL` 用哪个 id？ | **必须用 `PROCESSLIST_ID`**，不能用 `THREAD_ID` |
| `PROCESSLIST_ID` 能当长期追踪主键？ | ❌ **不能**。`uint32` 计数器会回绕，`release_thread_id()` 会回收；长期追踪用 `THREAD_ID`，KILL 时再反查 |
| State 在 PFS 关了还更新吗？ | ✅ **更新**。`pfs_start_stage_v1()` 里 `m_stage = key` 在所有开关判断之前（关掉的只是写 stage **事件**） |
| `SHOW` 与 `PFS` 的 State 完全一致？ | ❌ **不同**。SHOW 走 `thread_state_info()`，网络收发时显示 `Sending/Receiving to client`，还有 `Waiting on cond`、`Killed`；PFS 显示 stage 名且不受术语回退影响 |
| State 会被截断吗？ | ✅ 会。PFS 静默截到 64；**legacy SHOW 只有 30** |
| `sys.session` 适合高频采集吗？ | ❌ **不适合**。`ALGORITHM=TEMPTABLE` + 6 个 LEFT JOIN + ORDER BY；采集请用 `PFS.threads` |
| `sys.session` 那些等待/事务列为什么全是 NULL？ | 因为 **`events_*_current` consumer 没开**（默认关） |
| `THREAD_OS_ID` 是 pid 吗？ | ❌ 是 **tid**（Linux `gettid()`）。用 `perf -t <tid>` / `ps -L` 的 LWP 列对齐 |

---

## 参考

- *MySQL 8.0 Reference Manual*: *SHOW PROCESSLIST Statement* / *The threads Table* / *The processlist Table* / *Performance Schema Session Attribute Tables*
- 源码：
  - legacy SHOW：`sql/sql_show.cc`（`mysqld_list_processes()` / `List_process_list` / `processlist_fields_info` / `thread_state_info()`）、`sql/sql_parse.cc`（`COM_PROCESS_INFO`）、`sql/sql_show.h`（`PROCESS_LIST_WIDTH`）
  - PFS 模式：`sql/sql_show_processlist.cc`、`sql/parse_tree_nodes.{h,cc}`、`sql/sys_vars.cc`（`performance_schema_show_processlist`）
  - `performance_schema.threads`：`storage/perfschema/table_threads.cc`（`Plugin_table` 24 列 / `make_row` / `read_row_values` / `update_row_values`）
  - `performance_schema.processlist`：`storage/perfschema/table_processlist.cc`（8 列 + `set_access()` 权限模型）
  - State：`sql/sql_class.cc`（`THD::enter_stage()` / `proc_info()`）、`sql/mysqld.cc`（`all_server_stages[]`，约 100 项）、`storage/innobase/srv/srv0start.cc`（`srv_stages[]`，**12 项**）、`storage/innobase/srv/srv0srv.cc`（名称常量）、`storage/perfschema/pfs.cc`（`pfs_start_stage_v1`）
  - THD 管理：`sql/mysqld_thd_manager.{h,cc}`（`do_for_all_thd_copy` / `LOCK_thd_list` / `LOCK_thd_remove` / `get_new_thread_id` / `release_thread_id`）
  - KILL：`sql/sql_parse.cc`（`SQLCOM_KILL` / `kill_one_thread` / `Find_thd_with_id`）
  - 连接属性：`storage/perfschema/table_session_connect_attrs.cc`
  - 账号维度：`storage/perfschema/table_{accounts,users,hosts}.cc`、`pfs_stat.h`（`PFS_connection_stat`）
- [`sys_schema.md`](sys_schema.md)（sys.session / sys.processlist 的完整剖析）· [`pfs/pfs_events.md`](pfs/pfs_events.md)（stages 事件的 `WORK_COMPLETED` / 默认开关）· [`../README.md`](../README.md)（观测手段全景与决策树）· [`../../feat/ddl/ddl.md`](../../feat/ddl/ddl.md)（DDL 场景中 State 与进度的具体用法）

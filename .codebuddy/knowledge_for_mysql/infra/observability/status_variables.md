# 状态变量与系统变量：体系、作用域与 counter/gauge 分类

> **边界**：本篇讲**状态变量（status variables）**与**系统变量（system variables）**这两套变量体系。
> - `SHOW STATUS` 等命令的实现机制见 [`show_commands.md`](show_commands.md)
> - PFS 事件表族见 [`pfs/pfs_events.md`](pfs/pfs_events.md)
>
> **一句话**：状态变量是"已经发生了多少"的计数器（多数**必须两次采样求差**），系统变量是"当前配置是什么"——两者常被混淆，但用法完全不同。

## 目录

- [一、概述](#一概述)
- [二、状态变量的定义机制](#二状态变量的定义机制)
- [三、★ 三个作用域（默认 SESSION）](#三-三个作用域默认-session)
- [四、存储与汇总时机](#四存储与汇总时机)
- [五、★ PFS 中的状态变量表](#五-pfs-中的状态变量表)
- [六、`SHOW STATUS` 与 PFS 表的关系](#六show-status-与-pfs-表的关系)
- [七、★ `SHOW STATUS` 的代价](#七-show-status-的代价)
- [八、关键状态变量清单](#八关键状态变量清单)
- [九、★ counter vs gauge 分类与采样套路](#九-counter-vs-gauge-分类与采样套路)
- [十、系统变量体系](#十系统变量体系)
- [十一、可观测性](#十一可观测性)
- [十二、Misc（易混淆点）](#十二misc易混淆点)
- [参考](#参考)

---

## 一、概述

### 是什么

| | 状态变量（status） | 系统变量（variables） |
|---|---|---|
| 语义 | **已发生多少**（计数器 / 当前量） | **当前配置是什么** |
| 可写 | ❌ 只读（`FLUSH STATUS` 可重置部分） | ✅ `SET GLOBAL` / `SET SESSION` / `SET PERSIST` |
| 典型 | `Questions`、`Threads_connected`、`Innodb_rows_read` | `innodb_buffer_pool_size`、`long_query_time`、`sql_mode` |
| 观测入口 | `SHOW [GLOBAL\|SESSION] STATUS` / `performance_schema.global_status` 等 | `SHOW VARIABLES` / `performance_schema.variables_info` / `variables_by_thread` |

### 用途

- 状态变量：**量化**数据库负载（QPS、临时表落盘率、行锁等待、IO…）
- 系统变量：**确认**配置是否生效、值从哪来（`variables_info` 能查来源）

---

## 二、状态变量的定义机制

### 2.1 结构体（`sql/system_variables.h`）

```c
/**
  Per thread status variables.
  Must be long/ulong up to last_system_status_var so that
  add_to_status/add_diff_to_status can work.
*/
struct System_status_var { ... };
```

成员如 `created_tmp_disk_tables`、`created_tmp_tables`、`questions`、`long_query_count`、`filesort_rows`、`filesort_merge_passes`、`bytes_received` / `bytes_sent`、`com_stat[(uint)SQLCOM_END]` 等。

**关键约束**：`first_system_status_var` 与 `LAST_STATUS_VAR` 两个宏界定"**可批量累加的连续 ulong 区间**"：

```
Variables after 'last_system_status_var' cannot be handled automatically by
add_to_status() and add_diff_to_status().
```

⇒ 落在这个区间内的成员天然是**累加型 counter**（见第九节）。

### 2.2 对外描述结构（`include/mysql/status_var.h`）

```c
enum enum_mysql_show_type { SHOW_UNDEF, SHOW_BOOL, SHOW_INT, SHOW_LONG, SHOW_LONGLONG,
  SHOW_CHAR, SHOW_CHAR_PTR, SHOW_ARRAY, SHOW_FUNC, SHOW_DOUBLE, ... };
enum enum_mysql_show_scope { SHOW_SCOPE_UNDEF, SHOW_SCOPE_GLOBAL,
  /* Server-only values. Not supported in plugins. */ SHOW_SCOPE_SESSION, SHOW_SCOPE_ALL };
struct SHOW_VAR { const char *name; char *value; enum_mysql_show_type type; ... };
```

`SHOW_SCOPE_ALL` = GLOBAL + SESSION 都有（即 "BOTH"）。

**类型里的隐藏语义**（`sql/sql_show.cc`）：

```c
    case SHOW_LONG_STATUS:
      ...
    case SHOW_LONG:
      /* the difference lies in refresh_status() */
    case SHOW_LONG_NOFLUSH:
```

⇒ `SHOW_LONG_STATUS` 会被 `FLUSH STATUS` 重置，`SHOW_LONG_NOFLUSH` 不会。

### 2.3 注册表（`sql/sql_show.cc`）

```c
typedef std::vector<SHOW_VAR> Status_var_array;
Status_var_array all_status_vars(0);
bool status_vars_inited = false;
/* Version counter, protected by LOCK_STATUS. */
ulonglong status_var_array_version = 0;
```

Server 侧清单在 `sql/mysqld.cc` 的 `SHOW_VAR status_vars[]`（已核实片段）：

```c
  {"Aborted_clients", (char *)&aborted_threads, SHOW_LONG, SHOW_SCOPE_GLOBAL},
  {"Aborted_connects", (char *)&show_aborted_connects, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
  {"Bytes_received", (char *)offsetof(System_status_var, bytes_received), SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Com", (char *)com_status_vars, SHOW_ARRAY, SHOW_SCOPE_ALL},
  {"Compression", (char *)&show_net_compression, SHOW_FUNC, SHOW_SCOPE_SESSION},
  {"Connections", (char *)&show_thread_id_count, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
  {"Created_tmp_disk_tables", offsetof(System_status_var, created_tmp_disk_tables), SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Created_tmp_tables",      offsetof(System_status_var, created_tmp_tables),      SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Open_tables", (char *)&show_open_tables, SHOW_FUNC, SHOW_SCOPE_ALL},
  {"Queries",     (char *)&show_queries,     SHOW_FUNC, SHOW_SCOPE_ALL},
  {"Questions",   offsetof(System_status_var, questions),        SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Slow_queries",offsetof(System_status_var, long_query_count), SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Sort_merge_passes", offsetof(System_status_var, filesort_merge_passes), SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Sort_rows",           offsetof(System_status_var, filesort_rows),           SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
  {"Threads_connected", (char *)&Connection_handler_manager::connection_count, SHOW_INT, SHOW_SCOPE_GLOBAL},
  {"Threads_running",   (char *)&show_num_thread_running, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
  {"Uptime", (char *)&show_starttime, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
  {"Uptime_since_flush_status", (char *)&show_flushstatustime, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
```

`com_status_vars[]` 里每条形如：

```c
  {"show_open_tables", offsetof(System_status_var, com_stat[(uint)SQLCOM_SHOW_OPEN_TABLES]),
   SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
```

⇒ 这就是 `Com_show_*` 家族的来源。

**InnoDB 侧**在 `storage/innobase/handler/ha_innodb.cc` 的 status vars 数组（前缀 `innodb_`）：

```c
  {"row_lock_current_waits", &export_vars.innodb_row_lock_current_waits, SHOW_LONG, SHOW_SCOPE_GLOBAL},
  {"row_lock_waits", &export_vars.innodb_row_lock_waits, SHOW_LONG, ...},
  {"row_lock_time", ..., SHOW_LONGLONG},
  {"row_lock_time_avg"/"row_lock_time_max", ..., SHOW_LONG},
  {"rows_deleted"/"rows_inserted", ..., SHOW_LONG},
  {"redo_log_enabled", ..., SHOW_BOOL},
```

取值由 `srv0srv.cc` 的 `srv_export_innodb_status()` 从 `srv_stats` / `export_vars` 刷新。

---

## 三、★ 三个作用域（默认 SESSION）

**默认 = SESSION**，语法层证据（`sql/sql_yacc.yy`）：

```
opt_var_type:
          %empty      { $$=OPT_SESSION; }
        | GLOBAL_SYM  { $$=OPT_GLOBAL; }
        | LOCAL_SYM   { $$=OPT_SESSION; }
        | SESSION_SYM { $$=OPT_SESSION; }
        ;
show_status_stmt:  SHOW opt_var_type STATUS_SYM opt_wild_or_where ...
```

⇒ **`SHOW STATUS` ≡ `SHOW SESSION STATUS` ≡ `SHOW LOCAL STATUS`**。

`PT_show_status::make_cmd()`：

```c
  if (m_var_type == OPT_SESSION) {
    if (build_show_session_status(m_pos, thd, lex->wild, m_where) == nullptr) return nullptr;
  } else if (m_var_type == OPT_GLOBAL) {
    if (build_show_global_status(m_pos, thd, lex->wild, m_where) == nullptr) return nullptr;
  }
```

**第三个作用域 = 跨线程聚合**：`performance_schema.status_by_user` / `status_by_host` / `status_by_account`——按 `user@host` 聚合多个连接的 SESSION 值。这是 **SHOW 语法表达不了的**，只能查 PFS 表。

---

## 四、存储与汇总时机

- per-THD：`thd->status_var`
- 全局：`global_status_var`（`sql/sql_show.cc`）

**连接退出时汇总**（`sql/sql_class.cc` `THD::cleanup_connection()`）：

```c
  mysql_mutex_lock(&LOCK_status);
  add_to_status(&global_status_var, &status_var);
  reset_system_status_vars(&status_var);
  mysql_mutex_unlock(&LOCK_status);
```

`THD::release_resources()` 中还有 PFS 侧聚合：

```c
  mysql_mutex_lock(&LOCK_status);
  add_to_status(&global_status_var, &status_var);
#ifdef HAVE_PSI_THREAD_INTERFACE
  if (m_psi != nullptr) { PSI_THREAD_CALL(aggregate_thread_status)(m_psi); }
#endif
```

⇒ **GLOBAL 值 = 已退出连接的累计 + 当前所有活连接的实时值**。后者在**读取时临时求和**（不落盘）：

```c
void calc_sum_of_all_status(System_status_var *to) {
  mysql_mutex_assert_owner(&LOCK_status);
  /* Get global values as base. */
  *to = global_status_var;
  Add_status add_status(to);
  Global_THD_manager::get_instance()->do_for_all_thd_copy(&add_status);
}
// class Add_status: if (!thd->status_var_aggregated) add_to_status(m_stat_var, &thd->status_var);
```

⚠️ 注意这里又出现了 `do_for_all_thd_copy`——与 legacy `SHOW PROCESSLIST` 同一套遍历（见 [`processlist.md`](processlist.md)）。

### ★ `SHOW STATUS` 自身不计入统计

`sql/sql_show.cc`：

```c
bool Sql_cmd_show_status::execute(THD *thd) {
  System_status_var old_status_var = thd->status_var;
  thd->initial_status_var = &old_status_var;
  bool status = Sql_cmd_show::execute(thd);
  // Don't log SHOW STATUS commands to slow query log
  thd->server_status &= ~(SERVER_QUERY_NO_INDEX_USED | SERVER_QUERY_NO_GOOD_INDEX_USED);
  // Restore status variables, as we don't want 'show status' to cause changes
  mysql_mutex_lock(&LOCK_status);
  add_diff_to_status(&global_status_var, &thd->status_var, &old_status_var);
  thd->status_var = old_status_var;
  ...
```

⇒ **观测行为本身不污染观测数据**——这是刻意设计（对比：很多监控系统做不到这点）。

### `FLUSH STATUS`

`refresh_status()`（`mysqld.cc`）：对每个 THD 先 `add_to_status` 再 `reset_system_status_vars`，并 `reset_pfs_status_stats()`。注释：`/* Reset global and session status variables. */`

权限（`sql/sql_reload.cc`）：

```c
  if ((op_type & REFRESH_STATUS) && !sctx->has_global_grant(STRING_WITH_LEN("FLUSH_STATUS")).first) {
    my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "RELOAD or FLUSH_STATUS");
```

⇒ 需要 **RELOAD 或 FLUSH_STATUS** 权限。

---

## 五、★ PFS 中的状态变量表

| 表 | 文件 | 定义 |
|---|---|---|
| `performance_schema.global_status` | `table_global_status.cc` | `VARIABLE_NAME VARCHAR(64) not null`, `VARIABLE_VALUE VARCHAR(1024)`, `PRIMARY KEY (VARIABLE_NAME) USING HASH` |
| `performance_schema.session_status` | `table_session_status.cc` | 同上（**当前连接**） |
| `performance_schema.status_by_thread` | `table_status_by_thread.cc` | `THREAD_ID BIGINT UNSIGNED not null` + 同上两列，PK `(THREAD_ID, VARIABLE_NAME)` |
| `performance_schema.status_by_user` | `table_status_by_user.cc` | 按 user 聚合 |
| `performance_schema.status_by_host` | `table_status_by_host.cc` | 按 host 聚合 |
| `performance_schema.status_by_account` | `table_status_by_account.cc` | 按 user@host 聚合 |

**可 TRUNCATE**（`pfs_truncatable_acl` + `delete_all_rows`）：`table_global_status::delete_all_rows()` 会在 `LOCK_status` 下依次 `reset_status_by_thread/account/user/host/global_status`。

`get_row_count()` 也须 `LOCK_status`（读 `all_status_vars.size()`）。

---

## 六、`SHOW STATUS` 与 PFS 表的关系

**是直接的语法糖/包装**（`sql/sql_show_status.cc` 注释原文见 [`show_commands.md`](show_commands.md) 2.1）：`SHOW GLOBAL STATUS` 被改写成

```sql
SELECT * FROM (SELECT VARIABLE_NAME AS Variable_name, VARIABLE_VALUE AS Value
               FROM performance_schema.global_status) global_status
```

⇒ 两者**没有本质差别**（同一条路径）。优化手段见第七节。

---

## 七、★ `SHOW STATUS` 的代价

代价全在 PFS 的"物化"过程。权威描述在 `storage/perfschema/pfs_variable.h` 头部：

```
  OVERVIEW
  Status and system variables are implemented differently in the server, but the
  steps to process them in the Performance Schema are essentially the same:
  1. INITIALIZE - Build or acquire a sorted list of variables ...
       - For status variables, copy existing global status array into a local
         array that can be used without locks. Expand nested sub arrays, indicated
         by a type of SHOW_ARRAY.
  2. MATERIALIZE - Convert the list of SHOW_VAR variables to string format, store in a local cache:
       - Resolve each variable according to the type.
       - Recursively process unexpanded nested arrays and callback functions.
       - Aggregate values across threads for global status.
       - Convert numeric values to a string.
       - Prefix variable name with the plugin name.
  3. OUTPUT - Iterate the cache for the SHOW command or table query.
  ...
  LOCK PRIORITIES
  Status Variables
    LOCK_status
      LOCK_thd_data                 (block THD delete)
```

落地（`table_global_status.cc`）：

```c
int table_global_status::rnd_init(bool /* scan */) {
  /* Build a cache of all global status variables. Sum across threads. */
  m_status_cache.materialize_global();
  return 0;
}
```

**结论**：

- `SHOW GLOBAL STATUS` 需要：在 `LOCK_status` 下复制 `all_status_vars`（数百项）→ 展开 `SHOW_ARRAY`（如 `Com`）→ 对 `SHOW_FUNC` 逐个执行回调 → **遍历所有 THD 做跨线程聚合** → 全部数值转字符串。**连接数越多越贵**
- 与直接查 PFS 表**没有本质差别**（就是同一条路径）

**优化手段**：

1. 加 `LIKE '前缀%'`——⚠️ **过滤发生在物化之后**（`index_init()` 同样物化一次），只省输出**不省物化**
2. 用 `SHOW SESSION STATUS` / `session_status`——**只物化当前线程，不做跨线程聚合**（便宜得多）
3. 高频采集用 `status_by_thread` 只取关心的变量 + WHERE
4. "复位式采样"：`FLUSH STATUS` 后等 N 秒再读（见第九节）

---

## 八、关键状态变量清单

> 以下名称均来自 `sql/mysqld.cc` 的 `status_vars[]` / `com_status_vars[]` 或 `ha_innodb.cc` 的 InnoDB status vars 数组。

### 连接 / 线程

| 变量 | 类型 | 说明 |
|---|---|---|
| `Threads_connected` | gauge | `&Connection_handler_manager::connection_count`，`SHOW_INT`，GLOBAL |
| `Threads_running` | gauge | `show_num_thread_running`，`SHOW_FUNC`，GLOBAL |
| `Connections` | counter | `show_thread_id_count`，GLOBAL |
| `Aborted_clients` / `Aborted_connects` | counter | GLOBAL |
| `Uptime` | gauge | `show_starttime`，GLOBAL |
| `Uptime_since_flush_status` | gauge | `FLUSH STATUS` 以来的秒数——**做差值时的分母可用它** |
| `Bytes_received` / `Bytes_sent` | counter | ALL |
| `Open_tables` | gauge | `SHOW_FUNC`，ALL |
| `Compression` / `Compression_algorithm` / `Compression_level` | — | `SHOW_FUNC`，**SESSION only** |

### 查询

| 变量 | 类型 | 说明 |
|---|---|---|
| `Questions` | counter | ALL |
| `Queries` | — | `SHOW_FUNC`，ALL |
| `Slow_queries` | counter | ALL |
| `Com_*` | counter | ALL，`SHOW_ARRAY` 展开（`Com_select`、`Com_show_status`、…） |

### 临时表 / 排序

| 变量 | 类型 | 说明 |
|---|---|---|
| `Created_tmp_tables` | counter | ALL |
| `Created_tmp_disk_tables` | counter | ALL |
| `Sort_rows` | counter | ALL |
| `Sort_merge_passes` | counter | ALL |

⇒ 常用组合：**`Created_tmp_disk_tables / Created_tmp_tables` = 临时表落盘比例**

### InnoDB 行操作（GLOBAL）

`Innodb_rows_read` / `Innodb_rows_inserted` / `Innodb_rows_updated` / `Innodb_rows_deleted`（counter，`SHOW_LONG`）

### ★ InnoDB 锁（counter 与 gauge 混合，最易判错的一组）

| 变量 | 类型 | 说明 |
|---|---|---|
| `Innodb_row_lock_waits` | **counter** | `srv_stats.n_lock_wait_count` |
| `Innodb_row_lock_current_waits` | **gauge** | `srv_stats.n_lock_wait_current_count`；`srv0mon.cc` 注释："Number of row locks currently being waited for" |
| `Innodb_row_lock_time` | counter | 累计等待 ms/1000 |
| `Innodb_row_lock_time_avg` / `_max` | **非累计** | **平均 / 最大值，不能求差**（见第九节） |
| `Innodb_truncated_status_writes` | counter | `SHOW ENGINE INNODB STATUS` 被截断的次数（见 [`show_commands.md`](show_commands.md) 4.5） |

### InnoDB 缓冲池 / redo

- `Innodb_pages_created` / `Innodb_pages_read` / `Innodb_pages_written`（counter，来自 `buf_pool->stat`）
- `Innodb_redo_log_enabled`（`bool`，GLOBAL）
- 其余（`Innodb_buffer_pool_*`、`Innodb_log_waits`、`Innodb_os_log_written` 等）请以 `ha_innodb.cc` 的 status vars 数组为准

---

## 九、★ counter vs gauge 分类与采样套路

### 9.1 分类依据（源码级三条）

1. **结构体注释**：`System_status_var` 落在 `[first_system_status_var, LAST_STATUS_VAR]` 区间的成员天然是**累加型 counter**（`add_to_status` 逐个相加）
2. **存储形式**：counter 来自 `System_status_var` 成员或全局 `ulong/ulonglong`（如 `aborted_threads`）；**gauge 多为 `SHOW_FUNC`**（`show_num_thread_running`、`show_open_tables`、`show_thread_id_count`、`show_starttime`）——函数现算当前值
3. **类型后缀**：`SHOW_LONG_STATUS` / `SHOW_LONGLONG_STATUS` 会被 `FLUSH STATUS` 重置；`SHOW_LONG_NOFLUSH` 不会

### 9.2 分类表

| 类型 | 判据 | 举例 | 用法 |
|---|---|---|---|
| **counter（必须两次采样求差）** | 落在可累加区间 / 名为 `*count*`、`*waits*`、`Com_*` / 类型 `SHOW_LONG_STATUS`、`SHOW_LONGLONG_STATUS` | `Questions`、`Connections`、`Bytes_received/sent`、`Created_tmp_tables`、`Created_tmp_disk_tables`、`Sort_rows`、`Sort_merge_passes`、`Slow_queries`、`Com_*`、`Innodb_rows_*`、`Innodb_row_lock_waits`、`Innodb_row_lock_time`、`Innodb_pages_read/written`、`Aborted_*` | `(v2-v1)/(t2-t1)`；更稳是与 `Uptime` 同时采样做比值 |
| **gauge（可直接读）** | `SHOW_FUNC` 现算 / 名字带 `current`、`connected`、`running` | `Threads_connected`、`Threads_running`、`Open_tables`、`Uptime`、`Uptime_since_flush_status`、`Innodb_row_lock_current_waits`、`Innodb_redo_log_enabled` | 直接读；**但对瞬时抖动敏感**，建议采多次取 p99/max |
| **非累计统计量（陷阱）** | 名字像时间但实为均值/最大值 | **`Innodb_row_lock_time_avg`、`Innodb_row_lock_time_max`** | ❌ **不能求差**；avg 是全程平均，长时间运行后对新变化不敏感 ⇒ 应配合 `Innodb_row_lock_waits` 的差值反推近期均值 |
| **速率型（InnoDB monitor 内）** | `%.2f xxx/s`，分母是"距上次打印的秒数" | `youngs/s`、`reads/s`、`hash searches/s` | ❌ **不能与状态变量速率混用**（两次 SHOW 间隔不同 ⇒ 分母不同） |

### 9.3 标准采样套路

```sql
-- ① 复位式（最准）：FLUSH STATUS 后等 N 秒再采
FLUSH STATUS;                       -- 需 RELOAD 或 FLUSH_STATUS 权限
-- 等 60 秒
SELECT VARIABLE_NAME, VARIABLE_VALUE
  FROM performance_schema.global_status
 WHERE VARIABLE_NAME IN ('Questions','Com_select','Created_tmp_tables',
                         'Created_tmp_disk_tables','Innodb_rows_read',
                         'Innodb_row_lock_waits','Innodb_row_lock_time');
-- 此时读到的就是"这 60 秒内的量"，可直接用（配合 Uptime_since_flush_status）

-- ② 差值式（不能 FLUSH 时）：采两次
--    t1: (v1, Uptime u1) → 等 N 秒 → t2: (v2, Uptime u2)
--    速率 = (v2 - v1) / (u2 - u1)

-- ③ 只看当前量（gauge，不用求差）
SELECT VARIABLE_VALUE FROM performance_schema.global_status
 WHERE VARIABLE_NAME IN ('Threads_connected','Threads_running',
                         'Innodb_row_lock_current_waits');

-- ④ 按账号聚合（SHOW 语法表达不了）
SELECT * FROM performance_schema.status_by_account
 WHERE VARIABLE_NAME IN ('Questions','Bytes_received');
```

---

## 十、系统变量体系

### 10.1 定义机制（`Sys_var_*` 族）

基类 `sql/set_var.h` 的 `class sys_var`；具体类型在 `sql/sys_vars.h`：

```c
class Sys_var_bool;  class Sys_var_charptr;  class Sys_var_double;
class Sys_var_enum;  class Sys_var_flagset;  class Sys_var_lexstring;
class Sys_var_multi_enum;  class Sys_var_plugin;  class Sys_var_set;
class Sys_var_tz;    class Sys_var_integer;
```

**整数族用模板 + typedef 生成**（源码注释原文）：

```c
  Sys_var_integer template is used to generate Sys_var_* classes
  for variables that represent the value as a signed or unsigned integer.
  They are Sys_var_uint, Sys_var_ulong, Sys_var_harows, Sys_var_ulonglong, and Sys_var_long.
  An integer variable has a minimal and maximal values, and a "block_size"
  (any valid value of the variable must be divisible by the block_size).
...
typedef Sys_var_integer<int32,     GET_UINT,     SHOW_INT,       false> Sys_var_int32;
typedef Sys_var_integer<uint,      GET_UINT,     SHOW_INT,       false> Sys_var_uint;
typedef Sys_var_integer<ulong,     GET_ULONG,    SHOW_LONG,      false> Sys_var_ulong;
typedef Sys_var_integer<ulonglong, GET_ULL,      SHOW_LONGLONG,  false> Sys_var_ulonglong;
```

`Sys_var_bool` 为什么不用 enum（注释解释了历史兼容原因）：

```
  multiple aliases (0=off=false, 1=on=true), but Sys_var_enum does not,
  so we could not use Sys_var_enum without breaking backward compatibility.
  Moreover, booleans default to false if option is not given, and true if option is given without value.
```

实际声明在 `sql/sys_vars.cc`。

### 10.2 ★ 作用域与可修改性

作用域宏（`sql/sys_vars.h`）：

```c
#define GLOBAL_VAR(X)   sys_var::GLOBAL, ...
#define SESSION_VAR(X)  sys_var::SESSION, offsetof(System_variables, X), ...
#define SESSION_ONLY(X) sys_var::ONLY_SESSION, offsetof(System_variables, X), ...
#define NO_CMD_LINE CMD_LINE(NO_ARG, -1)
```

⇒ 三档：**GLOBAL**（只有全局值）/ **SESSION**（全局+会话两份 = BOTH）/ **ONLY_SESSION**（只有会话值）。

标志位（`sql/set_var.h`）：

```c
    READONLY = 0x0400,          // 只能启动/配置文件设置
    ALLOCATED = 0x0800,
    INVISIBLE = 0x1000,         // 不显示
    TRI_LEVEL = 0x2000,         // default is neither GLOBAL nor SESSION
    NOTPERSIST = 0x4000,        // SET PERSIST 会报错
    HINT_UPDATEABLE = 0x8000,   // 可用 SET_VAR hint
    PERSIST_AS_READ_ONLY = 0x10000,   // 只能 PERSIST_ONLY
    /** Sensitive variable. If keyring is available, the variable will be
        persisted in mysqld-auto.cnf in encrypted format */
    SENSITIVE = ...
```

配套访问器：`is_readonly()` / `not_visible()` / `is_trilevel()` / `is_persist_readonly()` / `is_sensitive()` / `is_non_persistent()`

组合宏：`PREALLOCATED` / `NON_PERSIST` / `PERSIST_AS_READONLY` / `SENSITIVE`

**判读口径**：

- `READ_ONLY` ⇒ 只能启动/配置文件设置
- `NO_CMD_LINE` ⇒ 命令行不可用
- `PERSIST_AS_READ_ONLY` ⇒ 只能 `SET PERSIST_ONLY`（重启生效）
- `NOTPERSIST` ⇒ `SET PERSIST` 会报错
- `SENSITIVE` ⇒ 持久化时加密（需 keyring）

### 10.3 `variables_info` 表（查"值从哪来"）

⚠️ 它是 **`performance_schema.variables_info`**，不是 `information_schema` 的表。

```c
    "  VARIABLE_NAME varchar(64) not null,\n"
    "  VARIABLE_SOURCE ENUM('COMPILED','GLOBAL','SERVER','EXPLICIT','EXTRA',\n"
    "                       'USER','LOGIN','COMMAND_LINE','PERSISTED',\n"
    "                       'DYNAMIC') DEFAULT 'COMPILED',\n"
    "  VARIABLE_PATH varchar(1024),\n"
    "  MIN_VALUE varchar(64),\n"
    "  MAX_VALUE varchar(64),\n"
    "  SET_TIME TIMESTAMP(6) default null,\n"
    "  SET_USER CHAR(32) collate utf8mb4_bin default null,\n"
    "  SET_HOST CHAR(255) CHARACTER SET ASCII default null\n",
```

来源枚举（`include/mysql/components/services/system_variable_source_type.h`）：

```c
/**
  This enum values define how system variables are set. For example if a
  variable is set by global option file /etc/my.cnf then source will be
  set to GLOBAL, or if a variable is set from command line then source
  will hold value as COMMAND_LINE.
*/
enum enum_variable_source {
  COMPILED = 1, GLOBAL, SERVER, EXPLICIT, EXTRA, MYSQL_USER, LOGIN,
  COMMAND_LINE, PERSISTED, DYNAMIC
};
```

（SQL 里显示的 `USER` 对应 C 枚举 `MYSQL_USER`）

**用途**：回答"这个值到底是哪来的"——`COMPILED`（编译默认）/ `GLOBAL`（全局配置文件）/ `EXPLICIT`（用户指定的 my.cnf）/ `COMMAND_LINE`（命令行）/ `PERSISTED`（mysqld-auto.cnf）/ `DYNAMIC`（运行时 SET 且未持久化）；`SET_TIME`/`SET_USER`/`SET_HOST` 记录**谁在什么时候改的**。

⇒ **排查"配置怎么没生效"的第一步**。

### 10.4 ★ `SET PERSIST`（8.0 持久化）

语法（`sql_yacc.yy`）：

```
option_type:
          GLOBAL_SYM        { $$=OPT_GLOBAL; }
        | PERSIST_SYM       { $$=OPT_PERSIST; }
        | PERSIST_ONLY_SYM  { $$=OPT_PERSIST_ONLY; }
        | SESSION_SYM       { $$=OPT_SESSION; }
        ;
```

实现：`sql/persisted_variable.{h,cc}` 的单例 `Persisted_variables_cache`：

```c
// set PERSIST命令持久化全局变量到mysqld-auto.cnf，mysql启动的时候先读这个文件。
// 保证启动的时候参数修改不会丢失。
class Persisted_variables_cache final { ... };
```

文件格式是 **JSON**（`mysqld-auto.cnf`）：

```c
/* Keys used by mysqld-auto.cnf JSON file */
constexpr char s_key_value[] = "Value";
...
  New format for mysqld-auto.cnf is as below:
  {
    "mysql_static_parse_early_variables": { "variable_name_1" : { ...
```

敏感变量：`read_persist_file_v2()` 里有 `process_sensitive_section`（配合 keyring 加密，对应 `SENSITIVE` 标志）。

**要点**：

- `SET PERSIST` = `SET GLOBAL` + 写入 `mysqld-auto.cnf`（**运行时立即生效 + 重启后保留**）
- `SET PERSIST_ONLY` = 只写文件、**不改运行时值**（用于 `PERSIST_AS_READ_ONLY` 变量，重启生效）
- `RESET PERSIST` = 删除持久化项
- 文件位置：datadir 下（**具体路径拼接逻辑本次未展开取证**）

### 10.5 `variables_by_thread`

```c
    "  THREAD_ID BIGINT unsigned not null,\n"
    "  VARIABLE_NAME VARCHAR(64) not null,\n"
    "  VARIABLE_VALUE VARCHAR(1024),\n"
    "  PRIMARY KEY (THREAD_ID, VARIABLE_NAME) USING HASH\n",
```

只读（`&pfs_readonly_acl`，`perpetual = true`）。

**用途**：看"**某个连接实际生效的会话变量是什么**"——排查 `sql_mode`、`transaction_isolation`、各类超时设置的 per-connection 差异。同族还有 `variables_by_user` / `variables_by_host` / `variables_by_account`。

---

## 十一、可观测性

### 常用查询

```sql
-- ① 关键负载指标（复位式，推荐）
FLUSH STATUS;  -- 等 N 秒后
SELECT VARIABLE_NAME, VARIABLE_VALUE FROM performance_schema.global_status
 WHERE VARIABLE_NAME IN ('Questions','Com_select','Com_insert','Com_update','Com_delete',
                         'Created_tmp_tables','Created_tmp_disk_tables','Sort_merge_passes',
                         'Innodb_rows_read','Innodb_row_lock_waits','Innodb_row_lock_time',
                         'Innodb_pages_read','Innodb_pages_written','Bytes_received','Bytes_sent');

-- ② 瞬时量（gauge，直接读）
SELECT VARIABLE_NAME, VARIABLE_VALUE FROM performance_schema.global_status
 WHERE VARIABLE_NAME IN ('Threads_connected','Threads_running','Open_tables',
                         'Innodb_row_lock_current_waits','Uptime');

-- ③ 按账号看（SHOW 做不到）
SELECT user, host, VARIABLE_NAME, VARIABLE_VALUE
  FROM performance_schema.status_by_account WHERE VARIABLE_NAME = 'Questions';

-- ④ 某个变量的值是哪来的（排查配置不生效）
SELECT VARIABLE_NAME, VARIABLE_SOURCE, VARIABLE_PATH, SET_TIME, SET_USER, SET_HOST
  FROM performance_schema.variables_info
 WHERE VARIABLE_NAME IN ('innodb_buffer_pool_size','long_query_time','sql_mode');

-- ⑤ 看各连接实际生效的会话变量差异
SELECT t.PROCESSLIST_ID, v.VARIABLE_NAME, v.VARIABLE_VALUE
  FROM performance_schema.variables_by_thread v
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE v.VARIABLE_NAME IN ('sql_mode','transaction_isolation','long_query_time');
```

### 观测对象 → 手段 速查

| 我想看 | 入口 |
|---|---|
| QPS / TPS | `Questions` / `Com_*`（counter，求差） |
| 当前连接数 | `Threads_connected`（gauge） |
| 临时表落盘率 | `Created_tmp_disk_tables / Created_tmp_tables`（counter 比值） |
| 行锁等待 | `Innodb_row_lock_waits`（counter 求差）+ `Innodb_row_lock_current_waits`（gauge） |
| InnoDB 读写行 | `Innodb_rows_read/inserted/updated/deleted`（counter） |
| 缓冲池 IO | `Innodb_pages_read/written/created`（counter） |
| InnoDB monitor 是否截断 | `Innodb_truncated_status_writes`（counter） |
| 配置值从哪来 | `performance_schema.variables_info` |
| 各连接变量差异 | `performance_schema.variables_by_thread` |

---

## 十二、Misc（易混淆点）

| 混淆 | 澄清 |
|---|---|
| `SHOW STATUS` 默认看全局？ | ❌ **默认 SESSION**（`opt_var_type %empty → OPT_SESSION`） |
| GLOBAL 值是实时的？ | ⚠️ = 已退出连接累计 + **读取时临时求和**活连接（不落盘） |
| `SHOW STATUS` 自己会被统计进去？ | ❌ **不会**——刻意 `add_diff_to_status` 回滚自身影响 |
| `Innodb_row_lock_time_avg` 能求差？ | ❌ **不能**，它是全程平均值；应配合 `Innodb_row_lock_waits` 差值反推 |
| `Innodb_row_lock_current_waits` 是累计？ | ❌ 是 **gauge**（当前正在等的数量） |
| 加 `LIKE` 能省物化开销？ | ❌ **过滤在物化之后**，只省输出 |
| `SHOW GLOBAL STATUS` 与查 PFS 表有差别？ | ❌ **同一条路径**（前者是后者的语法糖） |
| `variables_info` 在 information_schema？ | ❌ 在 **`performance_schema`**；且它记录的是"来源"，不是当前值 |
| `SHOW VARIABLES` 改写成 `variables_info`？ | ❌ 改写成 `global_variables` / `session_variables` |
| `SET PERSIST` 与 `SET PERSIST_ONLY` | 前者 = SET GLOBAL + 写文件（立即生效）；后者 = **只写文件**（重启生效） |
| 持久化文件在哪 | datadir 下的 `mysqld-auto.cnf`（JSON 格式） |
| `FLUSH STATUS` 需要什么权限 | **RELOAD 或 FLUSH_STATUS** |

---

## 参考

- *MySQL 8.0 Reference Manual*: *Server Status Variables* / *Server System Variables* / *SET Syntax for Variable Assignment* / *Persisted System Variables*
- 源码：
  - 状态变量：`sql/system_variables.h`（`System_status_var`）、`include/mysql/status_var.h`（`SHOW_VAR` / `enum_mysql_show_scope`）、`sql/sql_show.cc`（`all_status_vars` / `calc_sum_of_all_status` / `Sql_cmd_show_status::execute`）、`sql/mysqld.cc`（`status_vars[]` / `com_status_vars[]`）、`sql/sql_class.cc`（`cleanup_connection` / `release_resources`）、`sql/sql_show_status.cc`（改写逻辑）、`sql/sql_reload.cc`（`FLUSH STATUS` 权限）
  - InnoDB 状态变量：`storage/innobase/handler/ha_innodb.cc`（InnoDB status vars 数组）、`storage/innobase/srv/srv0srv.cc`（`srv_export_innodb_status`）、`srv0mon.cc`（计数器注释）
  - PFS 表：`storage/perfschema/table_{global_status,session_status,status_by_thread,status_by_user,status_by_host,status_by_account}.cc`、`pfs_variable.h`（物化流程文档）
  - 系统变量：`sql/set_var.h`（`sys_var` / 标志位）、`sql/sys_vars.{h,cc}`（`Sys_var_*` 族与变量声明）、`sql/persisted_variable.{h,cc}`（`SET PERSIST`）、`storage/perfschema/table_variables_info.cc`、`table_variables_by_thread.cc`
- [`show_commands.md`](show_commands.md)（SHOW 命令族与改写机制）· [`processlist.md`](processlist.md)（`do_for_all_thd_copy` 的代价来源）· [`pfs/pfs_events.md`](pfs/pfs_events.md)（事件表族）· [`../README.md`](../README.md)（观测手段全景与决策树）

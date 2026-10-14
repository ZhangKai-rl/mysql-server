# 12 Prepared Statement 深度解析

> 基于 MySQL 8.0.39 源码。本篇回答三个问题：**PS 到底缓存了什么？社区版为什么不做 plan cache？"用预编译就不会走错索引"是真的吗？**
>
> **边界**：本篇讲 server 层的预编译机制（协议、`Prepared_statement` 生命周期、reprepare）。prepare 期的逻辑改写详见 [`06_resolver_prepare.md`](06_resolver_prepare.md)，optimize 过程详见 [`07_optimize/00_overview.md`](07_optimize/00_overview.md)。

## 目录

- [概述](#概述)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [参考](#参考)

---

## 概述

**是什么**：Prepared Statement（PS，预编译语句）是 MySQL 的"一次准备、多次执行"机制——客户端先发一条带 `?` 占位符的 SQL 模板，服务器解析、绑定、做完与元数据相关的改写后存下来；之后每次执行只需传 `stmt_id` + 参数值。

**解决什么问题**：

1. **省掉重复的 parse + resolve**：高频重复执行的语句，不需要每次重新走词法/语法分析和名字解析
2. **安全性**：参数值永不参与 SQL 解析，结构性防 SQL 注入
3. **协议效率**：二进制结果集（整数/浮点/时间定长编码）省去文本↔二进制转换

**一个必须先破除的误解**：**PS 不缓存执行计划。** 社区版 MySQL 没有 plan cache——每次 `COM_STMT_EXECUTE` 都会完整地重跑 optimize（join order、访问路径、代价估算）。PS 缓存的是**语法结构**，不是**执行计划**。

**在链路的哪个位置**：

```
COM_STMT_PREPARE  → Prepared_statement::prepare()   parse + resolve + 永久改写
COM_STMT_EXECUTE  → Prepared_statement::execute()   绑定参数 → 每次重做 optimize
                    └→ mysql_execute_command()      ★ 与 COM_QUERY 在此汇合
COM_STMT_CLOSE    → stmt_map.erase()                释放
```

上游是协议分发（[01 篇](01_protocol_to_dispatch.md)），下游与文本协议共用 `mysql_execute_command`——两条路在这里合流，之后完全一致。

**★ 源码对 PS 的自述**（`sql/sql_prepare.cc:24`，★ 官方定义）：

```cpp
/**
  @file

This file contains the implementation of prepared statements.

When one prepares a statement:

  - Server gets the query from client with command 'COM_STMT_PREPARE';
    in the following format:
    [COM_STMT_PREPARE:1] [query]
  - Parse the query and recognize any parameter markers '?' and
    store its information list in lex->param_list
  - Allocate a new statement for this prepare; and keep this in
    'thd->stmt_map'.
  - Without executing the query, return back to client the total
    number of parameters along with result-set metadata information
    (if any) in the following format: ...

  During prepare the tables used in a statement are opened, but no
  locks are acquired.  Table opening will block any DDL during the
  operation, and we do not need any locks as we neither read nor
  modify any data during prepare.  Tables are closed after prepare
  finishes.

When one executes a statement:

  - Server gets the command 'COM_STMT_EXECUTE' to execute the
    previously prepared query. If there are any parameter markers, then the
    client will send the data in the following format: ...
```
> ★★ 三条关键信息：① prepare = 解析 + 识别 `?` + 建 stmt 入 `thd->stmt_map` + **回元数据**；② **"Without executing the query"** —— prepare 不执行；③ **"tables are opened, but no locks are acquired"** —— prepare 只开表不锁表，表在 prepare 结束时就关闭。

**★ 官方 Doxygen 明确列举六个命令**（`sql/protocol_classic.cc:207`，⚠️ "五个"是漏了 `SEND_LONG_DATA`）：

```cpp
/**
  @page page_protocol_command_phase_ps Prepared Statements

  The prepared statement protocol was introduced in MySQL 4.1 and adds a
  few new commands:
    - @subpage page_protocol_com_stmt_prepare
    - @subpage page_protocol_com_stmt_execute
    - @subpage page_protocol_com_stmt_fetch
    - @subpage page_protocol_com_stmt_close
    - @subpage page_protocol_com_stmt_reset
    - @subpage page_protocol_com_stmt_send_long_data

  It also defines a more compact resultset format that is used instead of
  @ref page_protocol_com_query_response_text_resultset to return the results.

  @note Keep in mind that not all SQL statements can be prepared.

  @sa [WL#2871](https://dev.mysql.com/worklog/task/?id=2871)
*/
```
> ★ 这是唯一一处明确的 WL 编号：**WL#2871**（prepared statement protocol，MySQL 4.1 引入）。

---

## 设计思想与理论基础

### 设计思想与权衡

#### 核心命题：PS 是"语法结构缓存器"，不是"执行计划缓存器"

一次查询可以分成两段，MySQL 只缓存了第一段：

| 段 | 做什么 | 依赖什么 | PS 缓存吗 |
|---|---|---|---|
| **段 1** | SQL 文本 → 已解析、已绑定、已永久改写的语法树 | **只依赖元数据**（表定义、列定义、SQL 文本） | ✅ **缓存** |
| **段 2** | 语法树 → 执行计划（join order / 访问路径 / 迭代器） | 参数值、统计信息、运行环境 | ❌ **不缓存** |

**为什么恰好在这里切开？** 因为段 2 依赖三类"每次执行都可能变"的输入，而 prepare 期这些**全都还不确定**：

1. **参数值**——`? = 5` 和 `? = 5000000` 的选择率天差地别；`LIMIT ?` 更是直接改变 plan
2. **统计信息**——`records` / `cardinality` / `rec_per_key`，随数据变化，也不跨语句持久化在内存里
3. **运行环境**——`optimizer_switch`、hint、系统变量、二级引擎、当前事务可见性

而 prepare 期的状态是：**参数没绑、表没锁（prepare 只开表不加锁）、统计没采样**。所以段 2 只能在 execute 期做。

**★★★ `Prepared_statement` 类完整定义**（`sql/sql_prepare.h:340`，★ 核心命题的源码证据）：

```cpp
class Server_side_cursor;

/**
  Prepared_statement: a statement that can contain placeholders.
*/

class Prepared_statement final {
 public:
  /// Memory allocation arena, for permanent allocations to statement.
  Query_arena m_arena;                      /* ★ 注释明写 "for permanent allocations" */

  /// Array of parameters used for statement, may be NULL if there are none.
  Item_param **m_param_array{nullptr};

  /// Pointer to cursor, may be NULL if statement never used with a cursor.
  Server_side_cursor *m_cursor{nullptr};

  /// Used to check that the protocol is stable during execution
  const Protocol *m_active_protocol{nullptr};

  /// Number of parameters expected for statement
  uint m_param_count{0};

  uint m_last_errno{0};
  char m_last_error[MYSQL_ERRMSG_SIZE];

  /**
    Uniquely identifies each statement object in thread scope; change during
    statement lifetime.
  */
  const ulong m_id;

  LEX *m_lex{nullptr};  // parse tree descriptor      /* ★ 语法树，不是计划 */

  /// The query string associated with this statement.
  LEX_CSTRING m_query_string{NULL_CSTR};

  /// Performance Schema interface for a prepared statement.
  PSI_prepared_stmt *m_prepared_stmt{nullptr};

 private:
  /// True if statement is used with cursor, false if used in regular execution
  bool m_used_as_cursor{false};
  Query_result *m_regular_result{nullptr};   /* ★ 普通执行的结果接收器 */
  Query_result *m_cursor_result{nullptr};    /* ★ 游标执行的结果接收器（二选一） */
  Query_result *m_aux_result{nullptr};
  bool m_is_sql_prepare{false};
  bool m_in_use{false};
  bool m_with_log{false};
  LEX_CSTRING m_name{NULL_CSTR};
  LEX_CSTRING m_db{NULL_CSTR};

  /**
    The memory root to allocate parsed tree elements (instances of Item,
    Query_block and other classes).
  */
  MEM_ROOT m_mem_root;                       /* ★★ 跨语句存活的语法树内存 */
```
> ★★ **注意注释用词**：`m_mem_root` 的注释是 "allocate **parsed tree elements** (instances of Item, Query_block and other classes)" —— 语法树元素，**不是** plan 元素。

**★★★ 最硬的证据：PS 里没有任何计划类成员**（grep 负证据）：

```
$ rg -n 'JOIN|QEP_TAB|AccessPath|JOIN_TAB' sql/sql_prepare.h
Found 0 matching results
```
> ★★ `JOIN` / `QEP_TAB` / `AccessPath` / `JOIN_TAB` 在 `sql/sql_prepare.h` 里**一个都没有**。PS 对象里连一个计划指针都没有，也就无从缓存。

**★ 跨语句保有 vs 每次执行重建**（★ 完整清单）：

| 成员 | 位置 | 生命周期 |
|---|---|---|
| `m_arena`（指向 `m_mem_root`） | `:349` | **跨语句保有** |
| `m_mem_root` | `:421` | **跨语句保有**，只有 DEALLOCATE / 连接断开才 `Clear()` |
| `m_lex`（语法树） | `:372` | **跨语句保有** |
| `m_param_array` | `:352` | **跨语句保有** |
| `m_query_string` / `m_name` / `m_db` | `:375/402/415` | **跨语句保有** |
| `JOIN` / `QEP_TAB` / `AccessPath` | **不存在** | 每次执行在 `thd->mem_root` 上重建 |

**★★ THD 侧注释：这就是核心命题的官方表述**（`sql/sql_class.h:4339`）：

```cpp
  /**
    This memory root is used for two purposes:
    - for conventional queries, to allocate structures stored in main_lex
      during parsing, and allocate runtime data (execution plan, etc.)
      during execution.
    - for prepared queries, only to allocate runtime data. The parsed
      tree itself is reused between executions and thus is stored elsewhere.
  */
  MEM_ROOT main_mem_root;
```
> ★★★ **`"execution plan, etc."` 明确归属 runtime data，落在 THD 的 `main_mem_root`，每次执行都会清理**；而 PS 的 parsed tree "stored elsewhere"（即 PS 的 `m_mem_root`）。这段注释几乎逐字陈述了核心命题。

**★ 反面证据：JOIN 属于 prepare 期的"附加内存"**（`sql/sql_prepare.cc:2557`）：

```cpp
  /*
    While doing context analysis of the query (in prepare_query())
    we allocate a lot of additional memory: for open tables, JOINs, derived
    tables, etc.  Let's save a snapshot of current parse tree to the
    statement and restore original THD. In cases when some tree
    transformation can be reused on execute, we set again thd->mem_root from
    stmt->mem_root (see setup_wild for one place where we do that).
  */
  thd->swap_query_arena(arena_backup, &m_arena);
```
> ★ 官方把 JOIN 归入 "additional memory"（prepare 期临时产物），通过 `swap_query_arena` 把 THD 的内存根换回普通执行根——**JOIN 不落在 PS 的 `m_mem_root` 里**。

**★ `m_lex` 与 `thd->lex` 的关系：借壳**（`sql/sql_prepare.cc:389`，★ 完整见下）：

```cpp
    m_lex = thd->lex;
    thd->lex = stmt->m_lex;      /* ★★ 临时替换 */
```
> ★ 执行期间把 `thd->lex` 整体替换成 `stmt->m_lex`，执行完再换回。**PS 没有自己的执行引擎入口**——它只是"换了个 LEX 的普通查询"。这解释了为什么两条协议能共用 `mysql_execute_command`。

#### 代价要落到具体后果（权衡的另一半）

**⚠️ 误解："用 prepared statement 就不会走错索引"——恰恰相反。**

正因为每次 execute 都基于**当前参数值**重新估算，同一个 PS 在不同执行之间**可能切换执行计划**。这不是 bug，而是上面这个设计的必然：参数在 optimize 期是可见的（`const_for_execution() == true`），优化器当然要利用它。

其他真实代价：

- **单次执行的点查，PS 大概率比文本协议更慢**——多 2 个 RTT（prepare + close），却没省掉 optimize
- **PS 的真实收益**是省掉 parse + resolve + 永久改写 + 客户端拼串/转义，以及二进制结果集的转换开销
- 所以 PS 的价值只在**高频重复执行**和**安全性**，不是"执行更快"的万能药

**★★ 内存代价：PS 的 `m_mem_root` 跨语句永不释放**（`sql/sql_prepare.cc:2291`）：

```cpp
Prepared_statement::Prepared_statement(THD *thd_arg)
    : m_arena(&m_mem_root, Query_arena::STMT_INITIALIZED),
      m_id(++thd_arg->statement_id_counter),
      m_mem_root(key_memory_prepared_statement_main_mem_root,
                 thd_arg->variables.query_alloc_block_size) {
  *m_last_error = '\0';
}
```
> ★ `m_mem_root` 是 PS 的**值成员**，跟着 PS 对象活；PS key = `key_memory_prepared_statement_main_mem_root`（`performance_schema.memory` 表可查）。

```cpp
/* sql/sql_prepare.cc:2349 —— ★ 只有析构才 Clear */
Prepared_statement::~Prepared_statement() {
  if (m_used_as_cursor) close_cursor();
  if (m_regular_result != nullptr) destroy(m_regular_result);
  if (m_cursor_result != nullptr) destroy(m_cursor_result);
  if (m_aux_result != nullptr) destroy(m_aux_result);
  ...
  m_arena.free_items();
  if (m_lex != nullptr) {
    assert(m_lex->sphead == nullptr);
    lex_end(m_lex);
    m_lex->destroy();
    delete pointer_cast<st_lex_local *>(m_lex);  // TRASH memory
  }
}
```
> ★★ 注意注释：**"We have to call free on the items even if cleanup is called as some items, like Item_param, don't free everything until free_items()"** —— `Item_param` 必须 `free_items()` 才彻底释放。

**★★ 全局上限：官方承认是为了防 OOM 攻击**（`sql/mysqld.cc:1389`）：

```cpp
/**
  Limit of the total number of prepared statements in the server.
  Is necessary to protect the server against out-of-memory attacks.
*/
ulong max_prepared_stmt_count;
/**
  Current total number of prepared statements in the server. This number
  is exact, and therefore may not be equal to the difference between
  `com_stmt_prepare' and `com_stmt_close' (global status variables), as
  the latter ones account for all registered attempts to prepare
  a statement (including unsuccessful ones). ...
*/
ulong prepared_stmt_count = 0;
```
```cpp
/* sql/mysqld.cc:1568 —— ★ 专用全局锁：PS 创建/销毁在高并发下是全局串行点 */
/**
  The below lock protects access to two global server variables:
  max_prepared_stmt_count and prepared_stmt_count. ...
  As PREPARE/DEALLOCATE rate in a loaded server may be fairly high,
  we need a dedicated lock.
*/
mysql_mutex_t LOCK_prepared_stmt_count;
```
> ★★ 第二个隐藏代价：**全局锁争用**。PS 的创建/销毁在高并发下是全局串行点——这与"PS 是连接局部的"形成讽刺对比。

**★ 上限的注册与超限报错**（`sql/sys_vars.cc:3085` / `sql/sql_class.cc:2069`）：

```cpp
static Sys_var_ulong Sys_max_prepared_stmt_count(
    "max_prepared_stmt_count",
    "Maximum number of prepared statements in the server",
    GLOBAL_VAR(max_prepared_stmt_count), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, num_prepared_stmt_limit), DEFAULT(16382), BLOCK_SIZE(1),
    &PLock_prepared_stmt_count, NOT_IN_BINLOG, ON_CHECK(nullptr),
    ON_UPDATE(nullptr), nullptr,
    /* max_prepared_stmt_count is used as a sizing hint by the performance
       schema. */
    sys_var::PARSE_EARLY);
```
```cpp
static constexpr int num_prepared_stmt_limit = 4 * 1024 * 1024;   // sys_vars.cc:677
```
```cpp
  if (prepared_stmt_count >= max_prepared_stmt_count) {
    mysql_mutex_unlock(&LOCK_prepared_stmt_count);
    my_error(ER_MAX_PREPARED_STMT_COUNT_REACHED, MYF(0),
             max_prepared_stmt_count);
    goto err_max;
  }
```
> ★ 错误文案：`"Can't create more than max_prepared_stmt_count statements (current value: %lu)"`（`share/messages_to_clients.txt:5706`，SQLSTATE 42000）。
> ★★ **上限是全局的**——别的连接把名额占满，你的 `PREPARE` 就会失败。这是 PS 的一个反直觉特性。
> ★ `PARSE_EARLY` = 必须在 PFS 初始化前解析，因为 PFS 用它做 sizing hint。
> ⚠️ 注意 `insert()` 的顺序是**先 emplace 进 hash、后检查上限**，超限时 `err_max:` 回滚删除。

**★ 8.0 的一个真实性能代价演进**（`sql/sql_prepare.cc:2323`，★ WL#6570 TODO）：

```cpp
  /*
    @todo WL#6570
    In the if() above there is is_global_audit_mask_set()) .
    Because caching_sha2 is loaded by default, it this function return true
    which makes with_log always true.
    ...
  */
```
> ★ 8.0 默认加载 `caching_sha2_password` 后，审计插件使 `m_with_log` 恒为 true ⇒ **PS 每次执行都要做参数展开**（把 `?` 回填成字面量写日志）。这是 8.0 里 PS 的一个真实性能代价。

#### 为什么社区版不做 plan cache

这不是"漏做"，而是一组**彼此强化的取舍**，源码里有迹可循：

| # | 理由 | 依据 |
|---|---|---|
| 1 | **PS 是连接局部的**——要做全局 plan cache 必须先有全局的、按 SQL 文本哈希的、带并发控制的 plan 仓库（Oracle 的 library cache latch 是著名争用点）。MySQL 从来没有这个结构 | `thd->stmt_map`；`mysqld.cc` 注释明写 "currently connection-local" |
| 2 | **元数据/DDL 导致 plan 高频失效**——MySQL 的 DDL 是整表换版本，plan 失效是常态而非例外 | `check_and_update_table_version()`；`CF_REEXECUTION_FRAGILE` |
| 3 | **参数化使"一个模板一个计划"本来就错**——`?` 的值每次都可能改变最优计划 | `Item_param::const_for_execution()` |
| 4 | **统计信息本身不稳定**——InnoDB 的基数估计是随机采样，缓存的 plan 正确性难保证 | `stats.records` / `rec_per_key` 语义 |
| 5 | **optimize 在 MySQL 里相对便宜**——默认是受限深度的贪心搜索（`optimizer_search_depth`），不是重量级 plan 枚举 | `JOIN::optimize()` 四阶段 |
| 6 | **收益/复杂度比不划算**——MySQL 的典型 workload 是简单点查，plan cache 的实现与运维复杂度却极高 | 设计取向 |
| 7 | **有更便宜的替代：reprepare 已自动且廉价**——因为只缓存解析结果，重做 prepare 的成本是**常数**，与 plan 复杂度无关 | `MAX_REPREPARE_ATTEMPTS = 3` |

> **一个自我一致的设计闭环**：因为没有 plan cache，reprepare 才能做到廉价且自动（重做 prepare 只是重新解析）；又因为 reprepare 足够廉价，就更没有必要做 plan cache。这两件事互相支撑。

**★★★ grep 负证据：8.0.39 里没有任何 plan cache 残留**：

```
$ rg -n 'plan_cache|opt_plan_cache' sql/          → 0 matching results
$ find sql/ -name '*plan_cache*'                  → 0 files
$ rg -n 'query_cache' sql/                        → 仅 3 个文件，全部是废弃壳
```
> ★★ **plan cache 从未进入社区版代码树——连一个空壳变量都没有**。而 query cache 8.0 已彻底删除，只剩一个恒为 NO 的废弃变量：
> ```cpp
> /* sql/mysqld.cc:10282 */
>   have_query_cache = SHOW_OPTION_NO;
> ```
> ```cpp
> static Sys_var_have Sys_have_query_cache(    // sql/sys_vars.cc:6028
>     "have_query_cache",
>     "have_query_cache. "
>     "This variable is deprecated and will be removed in a future release.",
>     READ_ONLY NON_PERSIST GLOBAL_VAR(have_query_cache), NO_CMD_LINE,
>     NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(nullptr), ON_UPDATE(nullptr),
>     DEPRECATED_VAR(""));
> ```

**★★ `LEX::safe_to_cache_query` 不是 plan cache 开关**（`sql/sql_lex.h:4166`，⚠️ 名字极易误导）：

```cpp
  /**
    Whether this query will return the same answer every time, given unchanged
    data. Used to be for the query cache, but is now used to find out if an
    expression is usable for partitioning.
  */
  bool safe_to_cache_query;
```
> ★★ 它**曾经**服务于 query cache，8.0 里退化成"表达式能否用于分区裁剪"的确定性标记。**名字里有 "cache" 纯属历史遗留。** PS 路径里仅存两处使用，都是"日志/游标"语义。

**★★★ `execute_loop()`：分支只有三类，没有"命中缓存跳过"**（`sql/sql_prepare.cc:2988`）：

```cpp
bool Prepared_statement::execute_loop(THD *thd, String *expanded_query,
                                      bool open_cursor) {
  Reprepare_observer reprepare_observer;
  bool error;
  bool reprepared_for_types [[maybe_unused]] = false;

  if (m_arena.get_state() == Query_arena::STMT_ERROR) {
    my_message(m_last_errno, m_last_error, MYF(0));
    return true;
  }
  ...
  // Reprepare statement unconditionally if it contains UDF references
  if (m_lex->has_udf() && reprepare(thd)) return true;

  // Reprepare statement if protocol has changed.
  if (m_active_protocol != nullptr &&
      m_active_protocol != thd->get_protocol()) {
    assert(false);
    if (reprepare(thd)) return true;
  }

reexecute:
  if (!check_parameter_types()) {
    assert(!reprepared_for_types);
    reprepared_for_types = true;
    if (reprepare(thd)) return true;      /* ① 参数类型不匹配 → reprepare */
    goto reexecute;
  }
  ...
  error = execute(thd, expanded_query, open_cursor) || thd->is_error();
  thd->pop_reprepare_observer();

  if ((sql_command_flags[m_lex->sql_command] & CF_REEXECUTION_FRAGILE) &&
      error && !thd->is_fatal_error() && !thd->is_killed()) {
    if (reprepare_observer.is_invalidated()) {
      assert(thd->get_stmt_da()->mysql_errno() == ER_NEED_REPREPARE);
      if (reprepare_observer.can_retry()) {
        thd->clear_error();
        error = reprepare(thd);           /* ② 元数据失效 → reprepare */
        ...
    } else {
      /* ③ secondary engine 失败 → reprepare（或降级回主引擎） */
      const uint err_seen = thd->get_stmt_da()->mysql_errno();
      if (err_seen == ER_PREPARE_FOR_PRIMARY_ENGINE ||
          err_seen == ER_PREPARE_FOR_SECONDARY_ENGINE) { ... }
      ...
    }
    if (!error) goto reexecute;
  }
  reset_stmt_parameters(this);
  return error;
}
```
> ★★★ **读完整个函数：分支只有三类，全部是 reprepare**。没有任何 "plan 命中 → 直接执行" 的分支。这是"社区版无 plan cache"的函数级铁证。

**★★ 每次执行都完整跑一遍 `mysql_execute_command`**（`sql/sql_prepare.cc:3563`）：

```cpp
  old_stmt_arena = thd->stmt_arena;
  thd->stmt_arena = &m_arena;

  if (m_lex->check_preparation_invalid(thd)) return true;
  ...
  m_lex->keep_diagnostics = DA_KEEP_PARSE_ERROR;

  // Prepare for a new execution
  m_lex->clear_execution();        /* ★★ 每次都清掉"上次执行"的状态 */
  ...
  status = mysql_execute_command(thd, true);   /* ★★ 每次都完整跑一遍（含优化器） */
  if (status) return true;
```
> ★★ 两行定生死：`clear_execution()` + `mysql_execute_command()`。**没有任何缓存旁路。**
> ★ `execute()` 开头注释还明确说：ps-protocol 的 `COM_STMT_EXECUTE` **"have no parsing stage"** —— 它跳过的是**解析**，不是**优化/执行**。

**★ reprepare 的定义反证"计划未被缓存"**（`sql/sql_prepare.cc:3183`）：

```cpp
  Reprepare this prepared statement.

  Performs a light reset of the Prepared_statement object (resets its MEM_ROOT
  and clears its MEM_ROOT allocated members), and calls prepare() on it again.
  ...
```
> ★★ 如果计划被缓存，元数据变化时只需 invalidate 缓存条目；而这里的动作是 **MEM_ROOT 重置 + prepare() 重来**（整棵语法树重建），恰恰说明 prepare 的产物只有语法树，没有可独立失效的计划对象。

```cpp
/* sql/sql_prepare.cc:3323 —— PS 的全部可交换"资产" */
void Prepared_statement::swap_prepared_statement(Prepared_statement *copy) {
  std::swap(m_mem_root, copy->m_mem_root);
  m_arena.swap_query_arena(copy->m_arena, &tmp_arena);
  copy->m_arena.set_query_arena(tmp_arena);
  std::swap(m_lex, copy->m_lex);
  std::swap(m_query_string, copy->m_query_string);
  std::swap(m_arena.mem_root, copy->m_arena.mem_root);
  std::swap(m_param_array, copy->m_param_array);
  std::swap(m_param_count, copy->m_param_count);
  /* Don't swap flags: the copy has IS_SQL_PREPARE always set. */
  std::swap(m_name, copy->m_name);
  std::swap(m_db, copy->m_db);
  std::swap(m_cursor_result, copy->m_cursor_result);
  std::swap(m_regular_result, copy->m_regular_result);
  std::swap(m_aux_result, copy->m_aux_result);
  std::swap(m_cursor, copy->m_cursor);
}
```
> ★ PS 的全部资产 = mem_root + arena + lex + query_string + param_array + name/db + result/cursor。**没有 JOIN/plan 需要 swap**。

### 理论溯源

"prepare 一次、执行多次"是数据库的经典设计，但各家做到哪一步不同：

- **Oracle / SQL Server**：做到**全局 plan cache**——跨会话共享执行计划，按 SQL 文本/digest 哈希，代价是需要全局 latch、跨会话失效广播、LRU 老化
- **PostgreSQL**：`PREPARE` 后前 5 次用 **custom plan**（代入实际参数值优化），之后才考虑切 **generic plan**——等于承认"参数值影响计划"，用执行次数来试探
- **MySQL 社区版**：只做到**语法结构缓存**，optimize 每次重做，靠 reprepare 兜底元数据变化

MySQL 的定位是"每连接自给自足、无全局共享失效"，这与 plan cache 的全局成本模型冲突。

**★ SQL 标准在源码里的对应物**（`sql/sql_prepare.cc:1767` / `:1925`，★ 三条术语证据）：

```cpp
/**
  SQLCOM_PREPARE implementation.

    Prepare an SQL prepared statement. This is called from
    mysql_execute_command and should therefore behave like an
    ordinary query (e.g. should not reset any global THD data).
    ...
*/
```
```cpp
/**
  SQLCOM_EXECUTE implementation.

    Execute prepared statement using parameter values from
    lex->prepared_stmt_params and send result to the client using
    text protocol. This is called from mysql_execute_command and
    therefore should behave like an ordinary query (e.g. not change
    global THD data, such as warning count, server status, etc).
*/
```
> ★★ 两条注释都强调 **"should behave like an ordinary query"** —— PS 在 MySQL 内部**不是一条独立的执行路径**，而是"换了 LEX 的普通查询"。这是它与 Oracle/SQL Server 的 PS（独立 plan 缓存实体）的根本差异。

**★ "ps-protocol" 是源码里的正式术语**（`sql/sql_prepare.cc:3422`）：

```cpp
    SQL prepared statements (SQLCOM_EXECUTE) also have a parsing
    stage first (where we find out it's EXECUTE ... [USING ...]).

    However, ps-protocol prepared statements have no parsing stage for
    COM_STMT_EXECUTE before coming here, so we reset the condition info
    here.  Since diagnostics statements can't be prepared, we don't need
    to make an exception for them.
```
> ★ 二进制协议的 PS（"ps-protocol"）与 SQL 层的 `PREPARE ... EXECUTE ... USING` **分属两条入口**，但最终都汇聚到同一个 `Prepared_statement::execute_loop`。

**★ 唯一的 WL 编号**（`sql/protocol_classic.cc:207`）：

```cpp
  @sa [WL#2871](https://dev.mysql.com/worklog/task/?id=2871)
```
> ★ prepared statement protocol 是 MySQL 4.1 引入的（WL#2871）。**另一个常提到的 `@todo WL#6570` 出现在 `sql_prepare.cc:2323`**，指向 prepare/execute 边界的长期重构。

### 算法与数据结构

**`Prepared_statement`** 核心成员：`m_arena`（永久内存池）、`m_param_array`（`Item_param` 数组）、`m_lex`（语法树）、`m_mem_root`（Item/Query_block 的分配根）。

**`Query_arena` 状态机**是"永久分配 vs 临时分配"的开关：

```cpp
  /*
    The states reflects three different life cycles for three
    different types of statements:
    Prepared statement: STMT_INITIALIZED -> STMT_PREPARED -> STMT_EXECUTED.
    Stored procedure:   STMT_INITIALIZED_FOR_SP -> STMT_EXECUTED.
    Other statements:   STMT_REGULAR_EXECUTION never changes.
  */
  enum enum_state {
    STMT_INITIALIZED = 0, STMT_INITIALIZED_FOR_SP = 1,
    STMT_PREPARED = 2,    STMT_REGULAR_EXECUTION = 3,
    STMT_EXECUTED = 4,    STMT_ERROR = -1
  };
```

当 `thd->stmt_arena->is_regular()` 为假时，`Prepared_stmt_arena_holder` 会把 `mem_root` 临时切到 PS 的 arena，让解析期产生的 Item 和改写结果**活过这一次命令**——这是"永久变换"在内存分配层面的实现。

**`Item_param` 的三套类型**——设计意图写在注释里：

> 1. `data_type()` — 参数的**期望**类型（由 CAST、所在表达式、上下文决定）
> 2. `data_type_source()` — 客户端**送来**的类型
> 3. `data_type_actual()` — 经过转换后**实际存的值**的类型

三套并存是为了支持 reprepare：类型不兼容时重做 prepare。

**★ `Item_param` 的完整定义**（`sql/item.h:4600`，★ `?` 的物理载体）：

```cpp
/// Dynamic parameters used as placeholders ('?') inside prepared statements

class Item_param final : public Item, private Settable_routine_parameter {
 public:
  enum enum_item_param_state {
    NO_VALUE, NULL_VALUE, INT_VALUE, REAL_VALUE, STRING_VALUE,
    TIME_VALUE,      ///< holds TIME, DATE, DATETIME
    LONG_DATA_VALUE, DECIMAL_VALUE
  };                                     /* ★ 8 个取值 */

  String str_value_ptr;
  my_decimal decimal_value;
  union {
    longlong integer;
    double real;
    MYSQL_TIME time;
  } value;                               /* ★★ 一个槽位承载多种类型 */
 private:
  /**
    True if type of parameter is inherited from parent object (like a typecast).
    Reprepare of statement will not change this type.
    E.g, we have CAST(? AS DOUBLE), the parameter gets data type
    MYSQL_TYPE_DOUBLE and m_type_inherited is set true.
  */
  bool m_type_inherited{false};
  /**
    True if type of parameter has been pinned, attempt to use an incompatible
    actual type will cause error (no repreparation occurs), and value is
    subject to range check. This is used when the parameter is in a context
    where its type is imposed. For example, in LIMIT ?, we assign
    data_type() == integer, unsigned; and the provided value must be
    convertible to unsigned integer: passing a DOUBLE, instead of causing a
    repreparation as for an ordinary parameter, will cause an error; passing
    integer '-1' will also cause an error.
  */
  bool m_type_pinned{false};             /* ★★ LIMIT ? 的类型被"钉死"，不 reprepare 而是直接报错 */
  enum_field_types m_data_type_source{MYSQL_TYPE_INVALID};
  enum_field_types m_data_type_actual{MYSQL_TYPE_INVALID};
  bool m_unsigned_actual{false};
  const CHARSET_INFO *m_collation_source{nullptr};
  const CHARSET_INFO *m_collation_actual{nullptr};
  Item_result m_result_type{STRING_RESULT};
  enum_item_param_state m_param_state{NO_VALUE};
  ...
```
>> ★★ **"prepare 期不能折叠 `?`" 在类型系统层面的根因**就在这一行：
```cpp
  /*
    Parameter is treated as constant during execution, thus it will not be
    evaluated during preparation.
  */
  table_map used_tables() const override { return INNER_TABLE_BIT; }
```
> ★ `INNER_TABLE_BIT` ⇒ `const_item()`（要求 `used_tables()==0`）为**假**，但 `const_for_execution()`（只排除 `INNER_TABLE_BIT`）为**真**。所以 `?` 在 prepare 期不算常量、在执行期算——这正是"不能折叠但能优化"的机制。

**★ reprepare 后参数值的搬运用 swap 而非 copy**（`sql/item.cc:4730`）：

```cpp
  Move parameter values from the old parameters to the new
  one. We simply "exchange" the values, which allows
  to save on allocation and character set conversion in
  case a parameter is a string or a blob/clob.
  ...
*/
void Item_param::set_param_type_and_swap_value(Item_param *src) {
  m_data_type_source = src->m_data_type_source;
  m_data_type_actual = src->m_data_type_actual;
  m_unsigned_actual = src->m_unsigned_actual;
  m_collation_source = src->m_collation_source;
  m_collation_actual = src->m_collation_actual;

  null_value = src->null_value;
  assert(m_param_state == src->m_param_state);
  value = src->value;

  decimal_value.swap(src->decimal_value);
  str_value.swap(src->str_value);
  str_value_ptr.swap(src->str_value_ptr);
}
```
> ★ 用 swap 避免字符串/LOB 的重新分配与字符集转换。**侧面说明 reprepare 会重建 `Item_param` 对象**，值必须手工搬运。

**★ `Prepared_statement_map`：双索引 + 一级缓存**（`sql/sql_class.h:465`）：

```cpp
class Prepared_statement_map {
 private:
  malloc_unordered_map<ulong, std::unique_ptr<Prepared_statement>> st_hash;
  collation_unordered_map<std::string, Prepared_statement *> names_hash;
```
> ★ `st_hash`（id → `unique_ptr`，拥有所有权）+ `names_hash`（name → 裸指针，仅索引）。**容器本身无容量上限**。
> ★ `find(ulong)` 带 `m_last_found_statement` 一级缓存——客户端连续 EXECUTE 同一 stmt_id 时是 **O(1)**。
> ★ 一个精妙细节：**有名 PS 不能按 id 查**：
> ```cpp
> Prepared_statement *Prepared_statement_map::find(ulong id) {
>   if (m_last_found_statement == nullptr || id != m_last_found_statement->id()) {
>     Prepared_statement *stmt = find_or_nullptr(st_hash, id);
>     if (stmt && stmt->name().str) return nullptr;   /* ★ 防止 SQL 层名字空间与二进制协议 id 空间串台 */
>     m_last_found_statement = stmt;
>   }
>   return m_last_found_statement;
> }
> ```

### 他库对比与演进动机

| | Oracle / SQL Server | PostgreSQL | MySQL 社区版 |
|---|---|---|---|
| 缓存粒度 | 全局执行计划 | 每会话 prepared plan | 每会话**仅语法树** |
| 跨会话共享 | ✅ | ❌ | ❌ |
| optimize 重做 | 否（命中缓存） | 前 5 次是，后可切 generic | **每次都重做** |
| 失效机制 | 全局广播 | 版本检查 | reprepare（最多 3 次） |

**演进动机**：源码里散落的 `@todo WL#6570` 指向一条长期重构（"prepare once, execute many"），目标是彻底理清 prepare/execute 边界、去掉 `unprepare()` 残留。**社区版不是"不做 plan cache"，而是"连 prepared statement 本身的 prepare/execute 分离都还没做到位"**——plan cache 在这个优先级序列里排得非常靠后。

**★ 8.0 的 secondary engine 给 PS 带来第三条 reprepare 原因**（`sql/sql_prepare.cc:3085`，★ 一个新演进点）：

```cpp
      const uint err_seen = thd->get_stmt_da()->mysql_errno();
      if (err_seen == ER_PREPARE_FOR_PRIMARY_ENGINE ||
          err_seen == ER_PREPARE_FOR_SECONDARY_ENGINE) {
        assert(thd->secondary_engine_optimization() ==
               Secondary_engine_optimization::PRIMARY_TENTATIVELY);
        assert(!m_lex->unit->is_executed());
        thd->clear_error();
        if (err_seen == ER_PREPARE_FOR_SECONDARY_ENGINE) {
          thd->set_secondary_engine_optimization(
              Secondary_engine_optimization::SECONDARY);
          MYSQL_SET_PS_SECONDARY_ENGINE(m_prepared_stmt, true);
        } else {
          thd->set_secondary_engine_optimization(
              Secondary_engine_optimization::PRIMARY_ONLY);
          MYSQL_SET_PS_SECONDARY_ENGINE(m_prepared_stmt, false);
        }
        // Disable the general log. The query was written to the general log in
        // the first attempt to execute it. No need to write it twice.
        general_log_temporarily_disabled |= disable_general_log(thd);
        error = reprepare(thd);
      }
```
> ★ 这是 **HeatWave / RAPID 二级引擎**引入的：主引擎 vs 二级引擎的选择也会触发 reprepare。加上"参数类型不匹配"与"元数据失效"，8.0 的 PS 共有**三类** reprepare 触发条件。
> ★ 还有一层降级逻辑：二级引擎优化失败 → 关掉二级引擎再 reprepare 一次（"try again without it"）。

**★ 客户端演进的官方自述**（`libmysql/libmysql.cc:1317`）：

```cpp
  DESCRIPTION
    This is an entry point of the new API. Returned handle stands for
    a server-side prepared statement. Memory for this structure (~700
    bytes) is allocated using 'malloc'. Once created, the handle can be
    reused many times. ...
    'mysql_stmt_init()' is a pure local call, server side structure is
    created only in mysql_stmt_prepare.
    ...
    - reset statement with mysql_stmt_reset() (a no-op which will
      just return).
```
> ★ 客户端句柄约 **700 字节**；⚠️ `mysql_stmt_reset()` 在客户端是 **no-op**，但服务端 `mysqld_stmt_reset()` 会真的关游标清参数并回 OK 包。

**★ 两条入口的汇聚**（★ 演进收敛的证据）：

```cpp
/* 二进制协议入口：sql/sql_parse.cc:1932 */
        mysqld_stmt_execute(thd, stmt, com_data->com_stmt_execute.has_new_types,
                            com_data->com_stmt_execute.open_cursor, parameters);
```
> ★ 与文本协议入口（`SQLCOM_EXECUTE` → `mysql_sql_stmt_execute`）最终都汇到 `execute_loop`——**两条入口共享一个实现**。

---

## 核心实现

### 主链路

```
COM_STMT_PREPARE
  → mysqld_stmt_prepare()
      → Prepared_statement::prepare()      parse_sql + resolve + 永久改写
      → m_lex->cleanup(true) + close_thread_tables + MDL 回滚  ← prepare 期开表不加锁
      → m_arena.set_state(STMT_PREPARED)

COM_STMT_EXECUTE
  → mysqld_stmt_execute()
      → set_parameters()                   绑定 Item_param 值（convert_value 转换）
      → Prepared_statement::execute_loop()
          → check_parameter_types()        不兼容则 reprepare
          → push_reprepare_observer()      安装元数据观察者
          → Prepared_statement::execute()
              → m_lex->clear_execution()   ★ optimized = false
              → mysql_execute_command()    ★ 与 COM_QUERY 汇合
                  → open_tables + check_privileges + lock_tables   ← 每次都做
                  → unit->optimize()       ★ 每次重做（join order / access path / 迭代器）
                  → unit->execute()
              → lex->cleanup(true)         ★ 销毁 JOIN
          → 若 ER_NEED_REPREPARE → reprepare() → goto reexecute（最多 3 次）

COM_STMT_CLOSE
  → stmt_map.erase() → prepared_stmt_count--
```

### 协议层：五个 COM_STMT_* 命令

`dispatch_command` 里各分支的语义与回包行为：

| 命令 | 语义 | 是否回包 | 计数 |
|---|---|---|---|
| `COM_STMT_PREPARE` | 新建 `Prepared_statement` 并 prepare；回 stmt_id/列数/参数数/元数据 | 回（自定义格式） | `Com_stmt_prepare++` |
| `COM_STMT_EXECUTE` | 绑定参数 → `execute_loop()`（必要时 reprepare）→ 执行 | 回（OK / 二进制结果集） | `Com_stmt_execute++` |
| `COM_STMT_SEND_LONG_DATA` | 分片追加某占位符的 BLOB/TEXT 值，**不校验、不回包**，错在 execute 时暴露 | 不回包 | `Com_stmt_send_long_data++` |
| `COM_STMT_RESET` | 清 long data、关游标、清参数；**保留已 parse 的 LEX** | 回 OK | `Com_stmt_reset++` |
| `COM_STMT_CLOSE` | 从 `stmt_map` 摘除并析构 | 不回包 | `Com_stmt_close++` |

`mysqld_stmt_prepare` 里有个细节：prepare 的响应（列定义包）必须按二进制协议写，所以会**临时把协议切成二进制再切回来**。

**二进制协议 vs 文本协议**：二进制行 = `0x00` + NULL bitmap + 各列定长原生编码（int 就是 4/8 字节 little-endian，double 是 IEEE754）；文本协议每行是 length-encoded string。

**⚠️ 命令码是六个不是五个**（`include/my_command.h:75`，★ 注意 `COM_SET_OPTION` 插在中间）：

```cpp
  COM_STMT_PREPARE,      /**< See @ref page_protocol_com_stmt_prepare */     // 0x16
  COM_STMT_EXECUTE,      /**< See @ref page_protocol_com_stmt_execute */     // 0x17
  /** See  @ref page_protocol_com_stmt_send_long_data */
  COM_STMT_SEND_LONG_DATA,                                                    // 0x18
  COM_STMT_CLOSE,        /**< See @ref page_protocol_com_stmt_close */       // 0x19
  COM_STMT_RESET,        /**< See @ref page_protocol_com_stmt_reset */       // 0x1A
  COM_SET_OPTION,        /**< See @ref page_protocol_com_set_option */       // 0x1B
  COM_STMT_FETCH,        /**< See @ref page_protocol_com_stmt_fetch */       // 0x1C
```
> ★ `COM_STMT_CLOSE` 与 `COM_STMT_FETCH` **不相邻**——中间夹着 `COM_SET_OPTION`。"五个"是漏了 `SEND_LONG_DATA` 的常见口误。

**★ 命令名字符串**（`sql/sql_parse.cc:241`，★ `SHOW PROCESSLIST` 里看到的就是这些）：

```cpp
    "Prepare",
    "Execute",
    "Long Data",
    "Close stmt",
    "Reset stmt",
    "Set option",
    "Fetch",
```
> ⚠️ 注意**没有** `Stmt_` 前缀。数组下标 == `enum_server_command` 值。

**★ 六个 case 的共同前缀**（`sql/sql_parse.cc:1932`）：

```cpp
    case COM_STMT_EXECUTE: {
      thd->reset_for_next_command();
      MYSQL_NOTIFY_STATEMENT_QUERY_ATTRIBUTES(thd->m_statement_psi, false);
      Prepared_statement *stmt = nullptr;
      if (!mysql_stmt_precheck(thd, com_data, command, &stmt)) {
        PS_PARAM *parameters = com_data->com_stmt_execute.parameters;
        copy_bind_parameter_values(thd, parameters, ...);
        mysqld_stmt_execute(thd, stmt, com_data->com_stmt_execute.has_new_types,
                            com_data->com_stmt_execute.open_cursor, parameters);
        thd->bind_parameter_values = nullptr;
        thd->bind_parameter_values_count = 0;
      }
      break;
    }
```
> ★ 六个 case **全部先走统一的 `mysql_stmt_precheck()`** 做 stmt_id → `Prepared_statement*` 解析与校验。
> ★ `COM_STMT_CLOSE` 与 `COM_STMT_SEND_LONG_DATA` 都调了 `disable_status()`——**这两条命令不回任何报文**。

**★ `server_command_flags`：PS 命令的特殊待遇**（`sql/sql_parse.cc:512`）：

```cpp
  server_command_flags[COM_STMT_PREPARE] =
      CF_SKIP_QUESTIONS | CF_ALLOW_PROTOCOL_PLUGIN;
  server_command_flags[COM_STMT_EXECUTE] = CF_ALLOW_PROTOCOL_PLUGIN;
  server_command_flags[COM_STMT_SEND_LONG_DATA] = CF_ALLOW_PROTOCOL_PLUGIN;
  server_command_flags[COM_STMT_CLOSE] =
      CF_SKIP_QUESTIONS | CF_ALLOW_PROTOCOL_PLUGIN;
  server_command_flags[COM_STMT_RESET] =
      CF_SKIP_QUESTIONS | CF_ALLOW_PROTOCOL_PLUGIN;
  server_command_flags[COM_STMT_FETCH] = CF_ALLOW_PROTOCOL_PLUGIN;
```
> ★★ `CF_SKIP_QUESTIONS` 表示**不累加 `Questions` 状态量**——所以 `Questions` 只涨 EXECUTE/FETCH/SEND_LONG_DATA，不涨 PREPARE/CLOSE/RESET。

**★ `mysql_stmt_precheck()`：PREPARE 是唯一的新建点**（`sql/sql_prepare.cc:1582`）：

```cpp
    case COM_STMT_PREPARE: {
      if (!(*stmt = new Prepared_statement(thd)))
        goto silent_error;
      if (thd->stmt_map.insert(*stmt))
        goto silent_error;      /* ★ insert 失败时 stmt 已被删除 */
      break;
    }
```
> ★ 这是 `prepared_stmt_count` 唯一的自增入口（SQL 层 `PREPARE` 也走它）。
> ★ 错误处理分三档：CLOSE / SEND_LONG_DATA → `silent_error`（不报错）；FETCH / RESET / EXECUTE → `not_found`（报 `ER_UNKNOWN_STMT_HANDLER`）；参数个数不符 → `wrong_arg`（`ER_WRONG_ARGUMENTS`）。

**★ `mysqld_stmt_prepare()` 的三个关键点**（`sql/sql_prepare.cc:1514`）：

```cpp
  bool switch_protocol = thd->is_classic_protocol();
  if (switch_protocol) {
    // set the current client capabilities before switching the protocol
    thd->protocol_binary->set_client_capabilities(
        thd->get_protocol()->get_client_capabilities());
    thd->push_protocol(thd->protocol_binary);      /* ★ 切到二进制协议 */
  }
  ...
  if (stmt->prepare(thd, query, length, nullptr)) {
    MYSQL_DESTROY_PS(stmt->m_prepared_stmt);
    thd->stmt_map.erase(stmt);                     /* ★ 失败时连带 prepared_stmt_count-- */
  }
  thd->set_secondary_engine_optimization(saved_secondary_engine);
  if (switch_protocol) thd->pop_protocol();
  ...
  // Prepared_statement::prepare_query() sends metadata packet if success
```
> ★ 元数据由 `prepare_query()` 发，不在这里。

**★ `mysql_stmt_get_longdata()`：第六个命令的真名**（`sql/sql_prepare.cc:2089`，⚠️ **不存在** `mysqld_stmt_send_long_data()`）：

```cpp
void mysql_stmt_get_longdata(THD *thd, Prepared_statement *stmt,
                             uint param_number, uchar *str, ulong length) {
  thd->status_var.com_stmt_send_long_data++;
  Diagnostics_area new_stmt_da(false);
  thd->push_diagnostics_area(&new_stmt_da);

  Item_param *param = stmt->m_param_array[param_number];
  param->set_longdata((char *)str, length);        /* ★ 追加而非覆盖 */
  if (thd->get_stmt_da()->is_error()) {
    stmt->m_arena.set_state(Query_arena::STMT_ERROR);
    stmt->m_last_errno = thd->get_stmt_da()->mysql_errno();
    snprintf(stmt->m_last_error, ..., "%.*s", ...);
  }
  thd->pop_diagnostics_area();
}
```
> ★★ 出错**不立即回包**，只把错误暂存到 `stmt->m_last_error`——所以长数据传错，**报错会延迟到 EXECUTE 时才发出**。
> ⚠️ `mysqld_stmt_reset()` 的注释里还残留着旧函数名 `mysqld_stmt_send_long_data`，是历史笔误。

**★ 密码过期时 PS 命令的特权白名单**（`sql/sql_parse.cc:1789`，★ 一个有趣的细节）：

```cpp
    COM_QUERY/COM_STMT_PREPARE and COM_STMT_EXECUTE do a more
    fine-grained check later.
    COM_STMT_CLOSE and COM_STMT_SEND_LONG_DATA don't return anything.
    COM_PING only discloses information that the server is running, ...
  if (unlikely(thd->security_context()->password_expired() &&
               command != COM_QUERY && command != COM_STMT_CLOSE &&
               command != COM_STMT_SEND_LONG_DATA && command != COM_PING &&
               command != COM_QUIT && command != COM_STMT_PREPARE &&
               command != COM_STMT_EXECUTE)) {
    my_error(ER_MUST_CHANGE_PASSWORD, MYF(0));
    goto done;
  }
```
> ★ `COM_STMT_FETCH` 与 `COM_STMT_RESET` **不在白名单**，会被 `ER_MUST_CHANGE_PASSWORD` 拦。

#### 报文格式：逐个字段拆开

**`COM_STMT_PREPARE` 请求**（客户端 → 服务器）：就是命令字 + 带 `?` 的 SQL 模板文本，无额外字段。

**`COM_STMT_PREPARE` 响应**（服务器 → 客户端），共四段：

```
① prepare 状态包（13 字节定长）
   [0x00 OK 标志:1]
   [stmt_id:4]            ← 后续所有命令用它寻址
   [column_count:2]       ← 结果集列数
   [param_count:2]        ← ? 的个数
   [预留:1]               ← 对 4.1 以下客户端的兼容
   [warning_count:2]
   [metadata_follows:1]   ← 仅当客户端有 CLIENT_OPTIONAL_RESULTSET_METADATA

② 参数元数据（param_count 个 column-definition 包）
③ 列元数据（column_count 个 column-definition 包，可被上面的开关省略）
④ EOF 包
```

构造这个包的函数（`store_ps_status`）直接把字段按偏移写死：

```cpp
bool Protocol_classic::store_ps_status(ulong stmt_id, uint column_count,
                                       uint param_count, ulong cond_count) {
  uchar buff[13];
  buff[0] = 0;                     /* OK packet indicator */
  int4store(buff + 1, stmt_id);
  int2store(buff + 5, column_count);
  int2store(buff + 7, param_count);
  buff[9] = 0;                     // Guard against a 4.1 client
  uint16 tmp = min(static_cast<uint16>(cond_count), ...);
  int2store(buff + 10, tmp);
  if (has_client_capability(CLIENT_OPTIONAL_RESULTSET_METADATA)) {
    buff[12] = static_cast<uchar>(m_thd->variables.resultset_metadata);
    return my_net_write(&m_thd->net, buff, sizeof(buff));
  }
  return my_net_write(&m_thd->net, buff, sizeof(buff) - 1);
}
```

**`COM_STMT_EXECUTE` 请求**（客户端 → 服务器），按源码注释的权威描述：

```
[COM_STMT_EXECUTE:1]
[stmt_id:4]
[open_cursor:1]                          ← 是否开服务器端游标
[iteration-count:4]                      ← 预留，恒为 1
[NULL bitmap:(param_count+7)/8]          ★ 哪些参数是 NULL
[new_params_bound_flag:1]                ← 后面是否带类型信息
  ├─ 若为 1：[type+unsigned_flag:2] × param_count
  │    ★ 每参数只占 2 字节：低 15 位 = enum_field_types，
  │      最高位 bit15 = unsigned 标志（sint2korr 读出后 & 1<<15）
[参数值] × param_count
  ├─ 非字符串：定长原生编码（1/2/4/8 字节）
  └─ 字符串/二进制/DECIMAL/时间：lenenc 长度 + 数据
```

解析在 `get_command` 里，注意它一次跳过 5 字节（1 字节 cursor + 4 字节 iteration-count）：

```cpp
    case COM_STMT_EXECUTE: {
      if (input_packet_length < 9) goto malformed;
      data->com_stmt_execute.stmt_id = uint4korr(read_pos);
      read_pos += 4;  packet_left -= 4;
      data->com_stmt_execute.open_cursor = *read_pos;
      read_pos += 5;  packet_left -= 5;
      ...
      if (parse_query_bind_params(m_thd, stmt->m_param_count,
              &data->com_stmt_execute.parameters,
              &data->com_stmt_execute.has_new_types, ...))
        goto malformed;
      break;
    }
```

**二进制结果集的行格式**（服务器 → 客户端）：

```cpp
void Protocol_binary::start_row() {
  if (send_metadata) return Protocol_text::start_row();
  packet->length(bit_fields + 1);      // 1 字节包头 + NULL bitmap
  memset(packet->ptr(), 0, 1 + bit_fields);
  field_pos = 0;
}
```

即每行 = `0x00`（1 字节包头条） + NULL bitmap + 各列定长值。**DECIMAL 例外**——源码注释 "Decimals are sent as text, also over the binary protocol"，因为变长十进制没有定长二进制表示。

**★ NULL 位图为什么从 bit 2 开始**——`Protocol_binary::store_null()`：

```cpp
uint offset = (field_pos + 2) / 8 + 1, bit = (1 << ((field_pos + 2) & 7));
char *to = packet->ptr() + offset;
```

- `+1`：字节 0 是行头 `0x00`，位图从字节 1 开始；
- `field_pos + 2`：字节 1 的 bit 0、bit 1 **保留不用**，第一列的 NULL 位从字节 1 的 bit 2 起；
- 位图字节数相应为 `bit_fields = (num_cols + 9) / 8`（= `1 + ceil((cols+2)/8)`）。

**★ DATETIME / TIME 的 length 字节**（`store_datetime` / `store_time`，首字节存的是"数据长度"，总长 = length+1）：

| 类型 | length 取值 | 数据布局 |
|---|---|---|
| DATETIME/TIMESTAMP | **0 / 4 / 7 / 11** | year(int2)+month(1)+day(1) [+ hour+min+sec] [+ 微秒 int4] |
| TIME | **0 / 8 / 12** | neg(1) + days(int4) + hours(1) + min(1) + sec(1) [+ 微秒 int4] |

（⚠️ 网上常写 DATETIME "8/12"——8.0 源码是 0/4/7/11；全零值只发 1 个 length=0 字节。）

**另外四个命令的字节布局**（`Protocol_classic::parse_packet`，之前只有语义表没有布局）：

| 命令 | 布局 | 响应 |
|---|---|---|
| `COM_STMT_SEND_LONG_DATA` | `stmt_id:int4 + param_id:int2 + data`（头 6 字节 = `MYSQL_LONG_DATA_HEADER`） | **无** |
| `COM_STMT_CLOSE` | `stmt_id:int4`，仅此 | **无** |
| `COM_STMT_RESET` | `stmt_id:int4` | OK（成功）/ ERR |
| `COM_STMT_FETCH` | `stmt_id:int4 + num_rows:int4` | 行分批 + EOF |

**★ 两个"无响应"命令的错误语义完全不同**：

- `SEND_LONG_DATA`：出错**挂到语句上**（`m_arena.set_state(STMT_ERROR)` + `m_last_errno`），下一次 `COM_STMT_EXECUTE` 时才报——注释原文 *"we are not sending any return packets here"*. 处理函数真实名是 **`mysql_stmt_get_longdata()`**（不是网上常写的 `mysqld_stmt_send_long_data`，那名字只在注释里出现）。
- `CLOSE`：连出错都不回——`mysql_stmt_precheck()` 里找不到 stmt_id 走 **`silent_error`**（对比 RESET/FETCH 的 `not_found` 会发 ERR）。释放点是 `Prepared_statement::deallocate()`（与 SQL 层 `DEALLOCATE PREPARE` 共用）。

**语句 ID 管理**：`Prepared_statement_map`（`sql_class.h`，双 hash：`id -> stmt` + `name -> stmt`）。ID 是**每连接单调递增**的 `++thd->statement_id_counter`，非全局。★ 隔离设计：`find(id)` 对**命名语句**（SQL `PREPARE` 语法创建的）返回 nullptr——二进制协议语句与 Dynamic SQL 语句**互相不可见**。容量上限 `max_prepared_stmt_count`；连接断开时 `THD::cleanup()` 里 `stmt_map.reset()` 统一析构。

**服务端游标（`COM_STMT_FETCH` 的前提）**：EXECUTE 时带 `CURSOR_TYPE_READ_ONLY(1)` → `mysqld_stmt_execute` 里 `open_cursor` → `Prepared_statement::execute()` 走 `Server_side_cursor`（`sql/sql_cursor.h`）。8.0 **唯一实现是 `Materialized_cursor`**——结果先物化进临时表，FETCH 时 `ha_rnd_next` 逐行读、每批结尾发 EOF（`SERVER_STATUS_CURSOR_EXISTS`），读完发最终 EOF（`SERVER_STATUS_LAST_ROW_SENT`）并 `close()`。`CURSOR_TYPE_FOR_UPDATE` / `SCROLLABLE` **不支持**（服务端只认 READ_ONLY 位）。

#### ★ 服务端游标深度剖析

**只有一种实现。** 文件头注释直接定性：

```
  Declarations for implementation of server side cursors. Only
  read-only non-scrollable cursors are currently implemented.
```

全库搜索 `public Server_side_cursor` 只命中 `Materialized_cursor`，而且它带 `final`——作者不打算再派生。

> ⚠️ 但源码里保留着"非物化"的措辞（*In case we were not able to open a non-materialzed cursor...*），那是**历史遗留说法**，8.0.39 已无非物化实现。
> ⚠️ **`Cursor` 基类在 `sql/` 下不存在**；`Server_side_cursor` 在 **`sql/sql_cursor.h:51`**（`sql_prepare.h:340` 只有前向声明）。

**★★ `Server_side_cursor`：为什么游标必须自带内存根**（`sql/sql_cursor.h:51`，★ 理解 PS 内存模型的最佳素材）：

```cpp
class Server_side_cursor {
 public:
  /**
    For allocation of Handler objects for the temporary table. The prepared
    statement's mem_root cannot be used, since the handler may be created and
    deleted several times. The execution mem_root cannot be used since
    creation of and retrieval from a cursor are in different executions.
  */
  MEM_ROOT mem_root{key_memory_TABLE, 1024};

 protected:
  Query_arena m_arena;
  /** Row destination used for fetch */
  Query_result *m_result;

 public:
  Server_side_cursor(Query_result *result_arg)
      : m_arena(&mem_root, Query_arena::STMT_INITIALIZED),
        m_result(result_arg) {}
  virtual bool is_open() const = 0;
  virtual bool open(THD *thd) = 0;
  virtual bool fetch(ulong num_rows) = 0;
  virtual void close() = 0;
  virtual ~Server_side_cursor() { mem_root.Clear(); }
  static void *operator new(size_t size, MEM_ROOT *mem_root,
                            const std::nothrow_t & = std::nothrow) noexcept {
    return mem_root->Alloc(size);
  }
  static void operator delete(void *, size_t) {}    /* ★ 只 TRASH，不 free */
};
```
> ★★★ 那 5 行注释**一句话区分了三种内存根**：PS 根（handler 会反复创建/删除，不能用）、执行根（创建游标与 FETCH 取数属于**两次不同执行**，不能用）、游标根（必须独立）。

**★ `Materialized_cursor`：物化的第一手证据**（`sql/sql_cursor.cc:63`）：

```cpp
/**
  Materialized_cursor -- an insensitive materialized server-side
  cursor. The result set of this cursor is saved in a temporary
  table at open. The cursor itself is simply an interface for the
  handler of the temporary table.

  The materialized cursor is usually attached to a preparable statement
  through a query result object. The lifetime of the cursor is the same
  as the lifetime of the preparable statement. When the preparable statement
  is destroyed, the materialized cursor (including the temporary table) is
  also destroyed.
 */

class Materialized_cursor final : public Server_side_cursor {
  /// A fake unit to supply to Query_result_send when fetching
  Query_expression fake_query_expression;
  /// Cursor to the table that contains the materialized result
  TABLE *m_table{nullptr};
  /**
    List of items to send to client, copy of original items, but created in
    the cursor object's mem_root.
  */
  mem_root_deque<Item *> item_list;
  ulong fetch_limit{0};      /* ★ 本次 FETCH 的上界 */
  ulong fetch_count{0};      /* ★ 已取行数 */
  bool is_rnd_inited{false};
```
> ★ "insensitive"（不敏感）= 物化快照。**生命周期绑定 PS**："The lifetime of the cursor is the same as the lifetime of the preparable statement."
> ⚠️ **不存在** `materialize_cursor()` 函数；真名是类 `Query_result_materialize` + `new_cursor_result()` + `instantiate_tmp_table()`。
> ⚠️ **不存在** `open_cursor()` 函数；PS 路径是 `Prepared_statement::execute()` 内联的 `if (open_cursor) {...}`，SP 路径才是 `mysql_open_cursor()`（且注释明说 *"Only used for cursors created by stored procedures"*）。

**★★ 物化进临时表的两行铁证**（`sql/sql_cursor.cc:453`）：

```cpp
bool Query_result_materialize::prepare(THD *thd,
                                       const mem_root_deque<Item *> &fields,
                                       Query_expression *u) {
  unit = u;
  if (m_result->prepare(thd, fields, u)) return true;
  assert(table == nullptr && m_cursor == nullptr);

  m_cursor = new (thd->mem_root) Materialized_cursor(m_result);
  /*
    Objects associated with the temporary table should be created as follows:
    - Metadata about the temporary table are created on the Statement mem_root.
    - HANDLER objects are created on the mem_root of the materialized cursor,
      since the handler must be kept open for subsequent FETCH operations.
  */
  if (create_result_table(thd, *unit->get_unit_column_types(), false,
                          thd->variables.option_bits | TMP_TABLE_ALL_COLUMNS,
                          "", false, false)) {
    destroy(m_cursor);
    return true;
  }
  m_cursor->set_table(table);
  return false;
}

bool Query_result_materialize::start_execution(THD *thd) {
  // If UNION, we may call this function multiple times.
  if (table->is_created()) return false;

  MEM_ROOT *saved_mem_root = thd->mem_root;
  thd->mem_root = &m_cursor->mem_root;      /* ★★ handler 分配到游标的 mem_root */
  if (instantiate_tmp_table(thd, table)) {
    thd->mem_root = saved_mem_root;
    return true;
  }
  table->file->ha_extra(HA_EXTRA_IGNORE_DUP_KEY);
  if (table->hash_field) table->file->ha_index_init(0, false);
  thd->mem_root = saved_mem_root;
  return false;
}
```

**★ `open()`：置 `SERVER_STATUS_CURSOR_EXISTS`**（`sql/sql_cursor.cc:365`）：

```cpp
bool Materialized_cursor::open(THD *thd) {
  Query_arena backup_arena;
  thd->swap_query_arena(m_arena, &backup_arena);

  /* Create a list of fields and start sequential scan. */
  rc = m_result->prepare(thd, item_list, &fake_query_expression);
  rc = !rc && m_table->file->ha_rnd_init(true);    /* ★ 对临时表开顺序扫描 */
  is_rnd_inited = !rc;

  thd->swap_query_arena(backup_arena, &m_arena);

  /* Commit or rollback metadata in the client-server protocol. */
  if (!rc) {
    thd->server_status |= SERVER_STATUS_CURSOR_EXISTS;   /* ★★ 客户端靠这个位识别 */
    m_result->send_eof(thd);
  } else {
    m_result->abort_result_set(thd);
  }
  fetch_limit = 0;
  fetch_count = 0;
  return rc;
}
```

**★ `COM_STMT_FETCH` 报文只有 8 字节，无方向字段**（`sql/protocol_classic.cc:2903`）：

```cpp
    case COM_STMT_FETCH: {
      if (input_packet_length < 8) goto malformed;
      data->com_stmt_fetch.stmt_id = uint4korr(input_raw_packet);
      data->com_stmt_fetch.num_rows = uint4korr(input_raw_packet + 4);
      break;
    }
```
> ★ 只能顺序向前。⚠️ 顺带一提，`COM_STMT_FETCH` 的 Doxygen Payload 表（`protocol_classic.cc:2637`）把 command 写成 `[0x19] COM_STMT_CLOSE` —— 这是 **8.0.39 源码里已知文档笔误**（应为 `0x1C`）。

**★ 元数据必须显式保留**（`sql/sql_cursor.cc:308`，★ "物化会改变字段语义"的直接证据）：

```cpp
    /*
      Unless we preserve the original metadata, it will be lost,
      since new fields describe columns of the temporary table.
      Allocate a copy of the name for safety only. ...
    */
    auto it_org = VisibleFields(send_result_set_metadata).begin();
    auto it_dst = item_list.begin();
    while (it_dst != item_list.end() && ...) {
      Item_ident *ident = static_cast<Item_ident *>(item_dst);
      item_org->make_field(&send_field);
      ident->db_name = thd->mem_strdup(send_field.db_name);
      ident->table_name = thd->mem_strdup(send_field.table_name);
    }
```
> ★★ 物化后 `item_list` 指向**临时表的列**，原始库名/表名/列名会丢失——所以必须拷一份。这是"物化不是透明操作"的源码级证据。

**★ `close()` 不 free 表，析构才 free**（`sql/sql_cursor.cc:432`）：

```cpp
void Materialized_cursor::close() {
  if (is_rnd_inited) {
    (void)m_table->file->ha_rnd_end();
    is_rnd_inited = false;
  }
  close_tmp_table(m_table);      /* ★ 只关不释放 */
  m_arena.free_items();
  item_list.clear();
  mem_root.ClearForReuse();      /* ★ 内存根保留复用 */
}

Materialized_cursor::~Materialized_cursor() {
  assert(!is_open());
  if (m_table != nullptr) free_tmp_table(m_table);   /* ★ 真正的释放 */
}
```
> ★ 游标对象活着 = 临时表定义活着，**可以被重新 open**。真正释放只在析构。

**★ 游标销毁的三条路径**：`mysqld_stmt_reset()` → `close_cursor()`；`fetch()` 读到 EOF → 自动 `close()`；PS 析构（`CLOSE` / `DEALLOCATE` / 连接断开）→ 析构函数。
> ★ `Query_result_materialize::~Query_result_materialize(){ destroy(m_cursor); }`（`sql_cursor.cc:116`）再把 `Materialized_cursor` 干掉。

**★ `fetch()` 末尾的官方性能自评**（`sql/sql_cursor.cc:390`，★ 值得记住）：

```cpp
  /**
    If the cursor points after the last row, the fetch will automatically
    close the cursor and not send any data (except the 'EOF' packet
    with SERVER_STATUS_LAST_ROW_SENT). This is an extra round trip
    and probably should be improved to return
    SERVER_STATUS_LAST_ROW_SENT along with the last row.
  */
```
> ★ 官方承认：读完最后一行还要**多一次往返**。

**★ 物化判定是三层 AND：**

**第 1 层**——客户端必须在 EXECUTE 的 flags 里声明 `CURSOR_TYPE_READ_ONLY`：

```cpp
    bool open_cursor = execute_flags & (ulong)CURSOR_TYPE_READ_ONLY;
```

**第 2 层**——语句类型必须支持。默认 `may_use_cursor()` 返回 false，**只有 `Sql_cmd_select` 覆盖为 true**：

```cpp
  virtual bool may_use_cursor() const { return false; }   // sql_cmd_dml.h（默认）
  bool may_use_cursor() const override { return true; }   // sql_select.h（Sql_cmd_select）
```

⇒ **只有 SELECT 能用游标**；UPDATE/DELETE/INSERT、SHOW、CALL 都不行。入口断言：

```cpp
  /*
    Cursors are not supported for regular (non-prepared, non-SP) statements,
    and the statement must return data (usually a SELECT statement).
  */
  assert(sql_cmd->may_use_cursor() && !sql_cmd->is_regular());
```

**第 3 层**——结果接收器必须支持游标（`check_supports_cursor()`）。基类默认报错：

```cpp
  virtual bool check_supports_cursor() const {
    my_error(ER_SP_BAD_CURSOR_QUERY, MYF(0));   // "Cursor statement must be a SELECT"
    return true;
  }
```

`Query_result_send` 与 `Query_result_materialize` 返回 false（支持）；`Query_result_to_file`（`SELECT ... INTO OUTFILE`）与 `Query_dumpvar`（`SELECT ... INTO @var`）报错 `ER_SP_BAD_CURSOR_SELECT`（"Cursor SELECT must not have INTO"）。

**⇒ 可物化 = 纯 SELECT（不带 INTO、不写文件）；其余一律走 `Query_result_send` 边算边发。**

还有一类根本不生成游标的情况（注释点名了 `SHOW PRIVILEGES`）：

```cpp
  /*
    Execution was successful. For most queries, a cursor has been created
    and must be opened, however for some queries, no cursor is used.
    This is possible if some command writes directly to the network,
    bypassing Query_result mechanism.
  */
```

**位置怎么记住：**

```cpp
bool Materialized_cursor::fetch(ulong num_rows) {
  for (fetch_limit += num_rows; fetch_count < fetch_limit; fetch_count++) {
    if ((res = m_table->file->ha_rnd_next(m_table->record[0]))) break;
    if (m_result->send_data(thd, item_list)) return true;
  }
  switch (res) {
    case 0:
      thd->server_status |= SERVER_STATUS_CURSOR_EXISTS;
      m_result->send_eof(thd);                     // 还有行
      break;
    case HA_ERR_END_OF_FILE:
      thd->server_status |= SERVER_STATUS_LAST_ROW_SENT;
      m_result->send_eof(thd);
      close();                                     // 读完自动关
      break;
```

**位置 = 临时表的 `ha_rnd_next` 顺序扫描位置**，外加 `fetch_count` / `fetch_limit` 两个计数器完成"本次取 N 行"。只进不退 ⇒ **非 scrollable**。

**与 `Query_result` 体系的关系**：`Query_result_materialize` 是一个**装饰器**（继承 `Query_result_union`，包装真正的 `Query_result`）：

- `prepare()` → 建临时表 + 创建 `Materialized_cursor`
- `start_execution()` → `instantiate_tmp_table()`
- 行写入 → 继承 `Query_result_union::send_data()`（写临时表）
- `fetch()` 时 → 才把行通过被包装的 `m_result->send_data()` 真正发给客户端
- ⚠️ 它必须**保存原始元数据**（注释：*"Unless we preserve the original metadata, it will be lost, since new fields describe columns of the temporary table"*）

**只读性的四条证据**：① 文件头声明 ② 协议只认 `CURSOR_TYPE_READ_ONLY` 位（`FOR_UPDATE=2`/`SCROLLABLE=4` 服务端根本不检查）③ 客户端库直接拒绝 `> CURSOR_TYPE_READ_ONLY` ④ 结构性原因——结果被物化到独立临时表，游标位置是临时表的扫描位置，与基表行没有 handler 级对应关系，因此不存在 `WHERE CURRENT OF` / 定位更新的通路。

**与 SP 游标的区别**（`sp_cursor` 详见 [`../stored_program.md`](../stored_program.md)）：两者**共用同一套物化实现**（`sp_cursor` 的 `m_server_side_cursor` 就是 `Materialized_cursor`），区别在于入口与协议可见性：

| | PS 游标 | SP 游标 |
|---|---|---|
| 入口 | `new_cursor_result()`，执行期惰性装配 | `mysql_open_cursor()`，OPEN 指令时 |
| 结果去向 | `Query_fetch_protocol_binary` / `Query_result_send`，**客户端可见** | `Query_dumpvar`，**客户端完全看不到**，只看到 CALL 的多结果集 |
| fetch 粒度 | 客户端指定 `num_rows` | 固定 1 行 |
| 生命周期 | 随 PS，`RESET`/`CLOSE` 关闭 | 随 `sp_rcontext` 游标栈，作用域退出时销毁 |

**往返次数（RTT）对比**：

| 场景 | RTT |
|---|---|
| 文本协议执行 1 次 | **1** |
| 文本协议执行 N 次 | N（每次重解析） |
| PS：prepare + 执行 N 次 + close | **1 + N + 1** |

⇒ **PS 只在 N 足够大时才划算**；只执行一次反而多 2 个 RTT。这是很多驱动默认关闭服务端预编译的直接原因。

**为什么 PS 能防 SQL 注入（语义层面）**：

1. `?` 在 parser 里被识别为 `Item_param`（`PARAM_ITEM`），登记进 `LEX::param_list`——它从一开始就是**语法树上的叶子节点对象**，不是"待替换的字符串"
2. 参数值通过 `Item_param::set_int/set_double/set_str/...` 写进 `value` 联合体，**从不参与 SQL 文本拼装**
3. execute 期直接走 `mysql_execute_command()`，**没有 `parse_sql()`**——参数值没有第二次机会被当成 SQL 语法

唯一的"拼字符串"路径是**写日志**，且发生在参数值定型之后、且带引号转义（`query_val_str` 会用 `'...'` 包裹），只用于 general log / slow log / binlog。

> 反面案例：客户端"预编译"（驱动把 `?` 拼成转义字面量后发 `COM_QUERY`）丢掉了这层保护——所以 JDBC `useServerPrepStmts=false` 时必须在驱动侧严格转义。

### 核心难点：为什么 prepare 期不能折叠 `?`

这是理解 PS 行为的关键，源码里只有三行，但信息量极大：

```cpp
  /*
    Parameter is treated as constant during execution, thus it will not be
    evaluated during preparation.
  */
  table_map used_tables() const override { return INNER_TABLE_BIT; }
```

配合基类定义：

```cpp
  bool const_item() const { return (used_tables() == 0); }
  /**
    Returns true if item is constant during one query execution.
    If const_for_execution() is true but const_item() is false, value is
    not available before tables have been locked and parameters have been
    assigned values. This applies to - statement parameters ...
  */
  bool const_for_execution() const { return !(used_tables() & ~INNER_TABLE_BIT); }
```

**逐句解读**：

- `used_tables()` 返回 `INNER_TABLE_BIT`（非 0）⇒ `const_item() == false` ⇒ **prepare 期不能常量折叠** `? + 1`（值还没有），这保证了 prepare 结果对任何参数值都成立
- 但 `INNER_TABLE_BIT` 被 `const_for_execution()` 掩掉 ⇒ `const_for_execution() == true` ⇒ **optimize 期（值已绑定）把它当常量**

所以：range optimizer 能用参数做范围裁剪、能做 const table 检测、能做分区裁剪——**这就是"同一个 PS，不同参数得到不同执行计划"的源码级原因**。

prepare 期 `Item_param::fix_fields` 也印证了这一点——参数此时还是"未定型"：

```cpp
bool Item_param::fix_fields(THD *, Item **) {
  assert(!fixed);
  if (param_state() == NO_VALUE) {
    // Parameter has no value, set data type from context
    assert(data_type() == MYSQL_TYPE_INVALID);
    collation.set(default_charset());
    fixed = true;
    return false;
  }
  ...
```

真正的类型由上下文通过 `propagate_type()` 灌入；execute 期才由 `set_xxx()` 写入实际值，再经 `convert_value()` 做字符集/时间/数值转换。

### 无 plan cache 的证据链

源码里六处相互印证：

**证据 1：PS 容器是线程局部的**，压根没有全局 plan hash。`Prepared_statement` 的 id 是每线程自增：

```cpp
Prepared_statement::Prepared_statement(THD *thd_arg)
    : m_arena(&m_mem_root, Query_arena::STMT_INITIALIZED),
      m_id(++thd_arg->statement_id_counter),     // ← 线程内计数器
```

源码里的注释说得很直白：*"Prepared statements are currently connection-local: if the same SQL query text is prepared in two different connections, this counts as two distinct prepared statements."*

**证据 2：每次执行前把 `optimized` 打回 false**——注意只清 `optimized/executed/cleaned`，**`prepared` 保持 true**，这正是"解析结果复用、优化结果不复用"的状态机表达：

```cpp
  /// Clear execution state, needed before new execution of prepared statement
  void clear_execution() {
    optimized = false;
    executed  = false;
    cleaned   = UC_DIRTY;
  }
```

**证据 3：`Query_block::optimize()` 断言 `join == nullptr` 并 new 一个新 JOIN**：

```cpp
bool Query_block::optimize(THD *thd, bool finalize_access_paths) {
  assert(join == nullptr);                                    // 上一次的 JOIN 必须已销毁
  JOIN *const join_local = new (thd->mem_root) JOIN(thd, this);
  ...
  if (join->optimize(finalize_access_paths)) return true;     // 每次都真的优化
```

**证据 4：执行结束就销毁 JOIN**——`Query_expression::cleanup()` 的注释直接宣告：

```cpp
/**
  Cleanup this query expression object after preparation or one round
  of execution. After the cleanup, the object can be reused for a
  new round of execution, but a new optimization will be needed before
  the execution.                                   ★★★
*/
```

**证据 5：每次 execute 都重做 open tables + 权限检查 + 加锁**（`Sql_cmd_dml::execute()` 的 prepared 分支）：

```cpp
    cleanup(thd);
    if (open_tables_for_query(thd, lex->query_tables, 0)) goto err;
    ...
    if (check_privileges(thd)) goto err;
    ...
    if (lock_tables(thd, lex->query_tables, lex->table_count, 0)) goto err;
```

**证据 6：`save/restore_cmd_properties` 保存的全是"喂给优化器的输入"**。`Table_ref` 里那批 `*_saved` 成员的注释是理解这一切的钥匙：

```cpp
  /*
    All members whose names are suffixed with "_saved" are duplicated in
    class TABLE but actually belong in this class. They are saved from class
    TABLE when preparing a statement and restored when executing the statement.
  */
```

保存/恢复的是 `covering_keys`、`merge_keys`、`keys_in_use_for_query` 这些**优化器的输入**，没有一个是优化器的**输出**（执行计划）。

> **易误读点**：`first_execution` 这个变量常被当成"只优化一次"的证据，其实相反——它只被 `Query_block::save_properties()` 清掉，守护的是 **resolve 阶段**的永久改写（如 `apply_local_transforms` 里的 `assert(first_execution)`），跟 optimize 无关。

**一次 PS execute 的完整工作量**（与文本协议对比）：

| 阶段 | COM_QUERY | PS execute |
|---|---|---|
| 词法+语法分析 | ✔ | ✘ **省下** |
| 名字解析 / fix_fields / 权限预检 | ✔ | ✘ **省下** |
| 永久逻辑改写（semi-join、view merge…） | ✔ | ✘ **省下** |
| 打开表 / MDL / check privileges / lock | ✔ | ✔ 都要 |
| **JOIN::optimize（代价估算、join order、access path）** | ✔ | ✔ **都要重做** |
| 创建迭代器 / 执行 / 清理 | ✔ | ✔ 都要 |

### reprepare：元数据变化的自动修复

**检测**：每次 execute 开表时，`check_and_update_table_version()` 拿 `TABLE_SHARE` 的版本与 PS 语法树里记录的版本比对：

```cpp
static bool check_and_update_table_version(THD *thd, Table_ref *tables,
                                           TABLE_SHARE *table_share) {
  if (!tables->is_table_ref_id_equal(table_share)) {
    /*
      Version of the table share is different from the
      previous execution of the prepared statement, and it is
      unacceptable for this SQLCOM.
    */
    if (ask_to_reprepare(thd)) return true;
    /* Always maintain the latest version and type */
    tables->set_table_ref_id(table_share);
  }
  return false;
}
```

**触发者**：`Reprepare_observer`，只对带 `CF_REEXECUTION_FRAGILE` 标志的语句安装（`SELECT/UPDATE/DELETE/INSERT` 等）。该标志的注释本身就是设计说明：*"Indicates that the parse tree of such statement may contain rule-based optimizations that depend on metadata ... and consequently that the statement must be re-prepared whenever referenced metadata changes."*

**报的是什么错**：`ER_NEED_REPREPARE`——这是一个**纯内部的"软错误"**，用来中断当前执行并回溯到 `execute_loop()`，绝不暴露给客户端（除非重试 3 次仍失败）：

```cpp
bool Reprepare_observer::report_error(THD *thd) {
  ...
  thd->get_stmt_da()->set_error_status(thd, ER_NEED_REPREPARE);
  m_invalidated = true;
  m_attempt++;
  return true;
}
```

**流程**：`execute_loop()` 是重试循环，最多 3 次（`MAX_REPREPARE_ATTEMPTS = 3`，防止 DDL 持续并发变更导致活锁）。`reprepare()` 用 swap + scope guard 保证失败可回滚：

```cpp
  Prepared_statement copy(thd);
  swap_prepared_statement(&copy);
  auto copy_guard = create_scope_guard([&]() { swap_prepared_statement(&copy); });
  ...
  if (prepare_error) return true;          // 失败：guard 析构时自动换回旧数据
  ...
  copy_guard.commit();                     // 成功：不回滚，旧数据随 copy 析构
```

**触发场景**：表结构变更（DDL 换 `TABLE_SHARE` 版本）、视图变化、权限变化、UDF（**无条件** reprepare，所以 PS 对含 UDF 的语句完全无收益）、参数实际类型与 resolved 类型不兼容。

**对用户完全透明**：只看到一次 `COM_STMT_EXECUTE` 的响应；`Com_stmt_reprepare` 状态变量 +1 是唯一的外部信号。

### 执行到 InnoDB：存储引擎侧没有"预编译"这回事

PS 是**纯粹的 server 层机制**——存储引擎完全不知道 PS 的存在。完整的下游链路：

```
Prepared_statement::execute()
  → mysql_execute_command()                    ★ 与 COM_QUERY 汇合
      → Sql_cmd_dml::execute() 的 prepared 分支
          cleanup() → open_tables_for_query() → restore_cmd_properties() → check_privileges()
      → lock_tables()
      → execute_inner()
          → unit->optimize()                   ★ 每次重做（前面已证明）
          → unit->execute()
              → ExecuteIteratorQuery()
                  → RowIterator::Read()        火山模型，一次一行
                      → handler 接口：ha_index_read_map / ha_rnd_next / ha_rnd_pos ...
                          → InnoDB：row_search_mvcc()（索引扫描 + MVCC 可见性判断）
```

**对 InnoDB 而言，每次 execute 就是一次普通的语句执行**：

- 事务是否开启、read view 何时建立，由**事务边界**决定（autocommit 决定每条语句是否独立事务；显式事务内多次 execute 共用同一个 read view），与"这条语句是不是预编译的"完全无关
- 不会因为"预编译过"而复用任何 InnoDB 侧结构——没有引擎层的 prepared plan，也没有缓存的 cursor

> **易混淆的命名**：handler 接口里确实有 `prepare` 家族（如 `ha_prepare_low`），但那是**事务两阶段提交（2PC）/ XA 的 prepare 阶段**，与 SQL 的 Prepared Statement 同名却毫无关系。

**这引出一个直接推论**：PS 带来的全部收益都在 server 层（省 parse + resolve + 永久改写，以及二进制协议省下的文本↔二进制转换）；**InnoDB 侧该做的扫描、加锁、MVCC 可见性判断一次都不少**。所以"用了 PS 数据库压力就小很多"是夸大的——它省的是 CPU 里的解析与 SQL 文本处理，不是 IO 与行访问。

反过来，这也解释了一个现象：**PS 对简单主键点查的收益很小**（optimize 占比不高、IO 只有一页），而对**复杂 SQL 的高频重复执行**收益明显（parse + resolve + 大量 Item 构造被摊薄）。

### 服务端预编译 vs 客户端预编译

| 维度 | 服务端预编译（真 PS） | 客户端"预编译"（模拟） |
|---|---|---|
| 线路命令 | `COM_STMT_PREPARE/EXECUTE/CLOSE` | **只有 `COM_QUERY`** |
| 参数怎么走 | 二进制 `PS_PARAM` → `Item_param`，**不进 SQL 文本** | 驱动把 `?` 替换为转义字面量拼进 SQL |
| 解析次数 | 1 次 | **每次都完整解析** |
| 防注入 | 结构性保证 | 依赖驱动的转义实现 |
| `Prepared_stmt_count` | 增加 | **不增加** |

- JDBC 默认 `useServerPrepStmts=false` → 客户端模拟（须配 `cachePrepStmts=true` 才有意义）
- PHP PDO 默认 `ATTR_EMULATE_PREPARES=true` → 客户端模拟；`mysqli_stmt_prepare()` 走真 PS
- 判断"是不是真 PS"的可靠方法：看服务器上 `Com_stmt_prepare` / `Prepared_stmt_count` 有没有变化，而不是看代码里有没有 `prepare()`

**★★★ 一个重要纠正：libmysql 的 `mysql_stmt_prepare()` 真的发协议往返**（`libmysql/libmysql.cc:1461`，⚠️ 别把锅扣到 libmysql 上）：

```cpp
  if (stmt_command(mysql, COM_STMT_PREPARE, (const uchar *)query, length,
                   stmt)) {
    set_stmt_errmsg(stmt, &mysql->net);
    return 1;
  }

  if ((*mysql->methods->read_prepare_result)(mysql, stmt)) {
    set_stmt_errmsg(stmt, &mysql->net);
    return 1;
  }
  ...
  if (!(stmt->params = (MYSQL_BIND *)stmt->mem_root->Alloc(
            sizeof(MYSQL_BIND) * (stmt->param_count + stmt->field_count)))) { ... }
  stmt->bind = stmt->params + stmt->param_count;
  stmt->state = MYSQL_STMT_PREPARE_DONE;
```
> ★★ **C API 里没有本地插值分支**。`mysql_stmt_prepare()` 真的发 `COM_STMT_PREPARE`，再用 `read_prepare_result` 读回 `param_count` / `field_count` / 列元数据。"客户端模拟预编译"是 **JDBC**（`useServerPrepStmts=false`）与 **PDO**（`ATTR_EMULATE_PREPARES`）的行为，**不在本仓库源码里**。写文档时务必区分。
> ★ 顺带：重复 prepare 同一句柄会先发一次 `COM_STMT_CLOSE` 释放旧 stmt_id。

**★ 客户端 `MYSQL_STMT` 里没有任何 AST**（`include/mysql.h:680`，★ "客户端侧 PS ≠ 预编译"的直接证据）：

```cpp
/* statement handler */
typedef struct MYSQL_STMT {
  struct MEM_ROOT *mem_root; /* root allocations */
  LIST list;                 /* list to keep track of all stmts */
  MYSQL *mysql;              /* connection handle */
  MYSQL_BIND *params;        /* input parameters */
  MYSQL_BIND *bind;          /* output parameters */
  MYSQL_FIELD *fields;       /* result set metadata */
  MYSQL_DATA result;         /* cached result set */
  MYSQL_ROWS *data_cursor;   /* current row in cached result */
  /*
    mysql_stmt_fetch() calls this function to fetch one row (it's different
    for buffered, unbuffered and cursor fetch).
  */
  int (*read_row_func)(struct MYSQL_STMT *stmt, unsigned char **row);
  uint64_t affected_rows;
  uint64_t insert_id;
  unsigned long stmt_id;       /* ★★ Id for prepared statement（服务端返回的句柄） */
  unsigned long flags;         /* i.e. type of cursor to open */
  unsigned long prefetch_rows; /* number of rows per one COM_FETCH */
  unsigned int server_status;
  unsigned int param_count;    /* ★ PREPARE 回包解析出来的 */
  unsigned int field_count;
  enum enum_mysql_stmt_state state;
  ...
} MYSQL_STMT;
```
> ★ 只有 `stmt_id`（4 字节服务器句柄）、`param_count`、`field_count`、bind 数组和结果缓存。**没有任何 AST / 计划**。
> ★ `read_row_func` 是三态函数指针：buffered / unbuffered / cursor。

```cpp
/* include/mysql.h:585 —— ★ 只有 4 态，是纯客户端状态 */
enum enum_mysql_stmt_state {
  MYSQL_STMT_INIT_DONE = 1,
  MYSQL_STMT_PREPARE_DONE,
  MYSQL_STMT_EXECUTE_DONE,
  MYSQL_STMT_FETCH_DONE
};
```
> ★ 与服务端 `Query_arena` 的 6 态状态机**完全无关**。

```cpp
/* libmysql/libmysql.cc:1324 —— ★ 官方明确 */
    'mysql_stmt_init()' is a pure local call, server side structure is
    created only in mysql_stmt_prepare.
```

**★ 客户端硬编码拒绝可滚动游标**（`libmysql/libmysql.cc:2063`，★ "只读不可滚动"的客户端侧佐证）：

```cpp
    case STMT_ATTR_CURSOR_TYPE: {
      ulong cursor_type;
      cursor_type = value ? *static_cast<const ulong *>(value) : 0UL;
      if (cursor_type > (ulong)CURSOR_TYPE_READ_ONLY) goto err_not_implemented;
      stmt->flags = cursor_type;
      break;
    }
```
> ★ `> CURSOR_TYPE_READ_ONLY(1)` 一律 `CR_NOT_IMPLEMENTED`。所以 `CURSOR_TYPE_SCROLLABLE(4)` 在客户端就被拒绝了。

**★ 协议闭环：客户端如何知道服务端开了游标**（`libmysql/libmysql.cc:2126`）：

```cpp
static void prepare_to_fetch_result(MYSQL_STMT *stmt) {
  if (stmt->server_status & SERVER_STATUS_CURSOR_EXISTS) {
    stmt->mysql->status = MYSQL_STATUS_READY;
    stmt->read_row_func = stmt_read_row_from_cursor;    /* ★ 后续 fetch 才发 COM_STMT_FETCH */
  } else if (stmt->flags & CURSOR_TYPE_READ_ONLY) {
    ...
    if (stmt->mysql->status != MYSQL_STATUS_READY)
      mysql_stmt_store_result(stmt);
  } else {
    stmt->read_row_func = stmt_read_row_unbuffered;
  }
}
```
> ★ 服务端 `Materialized_cursor::open()` 置 `SERVER_STATUS_CURSOR_EXISTS` → 客户端选 `stmt_read_row_from_cursor` → 之后 `mysql_stmt_fetch()` 才发 `COM_STMT_FETCH`。

**★ `enum_cursor_type` 定义了 4 种但只实现 1 种**（`include/mysql_com.h:1045`）：

```cpp
enum enum_cursor_type {
  CURSOR_TYPE_NO_CURSOR = 0,
  CURSOR_TYPE_READ_ONLY = 1,
  CURSOR_TYPE_FOR_UPDATE = 2,
  CURSOR_TYPE_SCROLLABLE = 4,
  /**
    On when the client will send the parameter count
    even for 0 parameters.
  */
  PARAMETER_COUNT_AVAILABLE = 8
};
```
> ★ `FOR_UPDATE` / `SCROLLABLE` 在协议上有定义，但服务端**从不检查**（`execute_flags & (ulong)CURSOR_TYPE_READ_ONLY` 是唯一判定）。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `max_prepared_stmt_count` | **16382** | GLOBAL（可动态 SET） | 服务器级 PS 总数上限。范围 0~4194304，**设为 0 表示完全禁止预编译**。超限报 `ER_MAX_PREPARED_STMT_COUNT_REACHED` |

### 状态变量

| 变量名 | 含义 |
|---|---|
| `Prepared_stmt_count` | 全局**精确**的当前 PS 数（不含失败尝试，区别于 `Com_stmt_prepare - Com_stmt_close`） |
| `Com_stmt_prepare/execute/close/reset/fetch/send_long_data` | 各命令次数（含失败尝试） |
| `Com_stmt_reprepare` | **自动 reprepare 次数**——排查元数据抖动的关键指标 |

**排障提示**：

- `Prepared_stmt_count` 持续上涨 ⇒ 应用泄漏 PS 句柄（忘了 close）。配合连接池尤其致命——连接不真断，额度就一直占着，直到撞上 16382 上限
- `Com_stmt_reprepare` 频繁上涨 ⇒ 表结构/统计信息变更频繁，PS 反而在做额外工作

**★ 状态变量的注册真身**（`sql/mysqld.cc`，★ 六个 + 一个例外）：

```cpp
/* :4472 —— 六个命令计数器在 com_status_vars[] 里，注册名是小写 */
    {"stmt_execute", (char *)offsetof(System_status_var, com_stmt_execute),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    {"stmt_close", (char *)offsetof(System_status_var, com_stmt_close), ...},
    {"stmt_fetch", (char *)offsetof(System_status_var, com_stmt_fetch), ...},
    {"stmt_prepare", (char *)offsetof(System_status_var, com_stmt_prepare), ...},
    {"stmt_reset", (char *)offsetof(System_status_var, com_stmt_reset), ...},
    {"stmt_send_long_data",
     (char *)offsetof(System_status_var, com_stmt_send_long_data), ...},
```
> ★ 通过 `{"Com", (char *)com_status_vars, SHOW_ARRAY, SHOW_SCOPE_ALL}`（`mysqld.cc:9721`）统一加 `Com_` 前缀暴露。

```cpp
/* :9722 —— ★ Com_stmt_reprepare 是唯一例外，直接注册在 status_vars[] */
    {"Com_stmt_reprepare",
     (char *)offsetof(System_status_var, com_stmt_reprepare), SHOW_LONG_STATUS,
     SHOW_SCOPE_ALL},
```
> ★★ 因为它对应的是**服务端内部行为**（reprepare），不是一条客户端命令。

```cpp
/* :9520 —— Prepared_stmt_count 的取值函数（SHOW_FUNC） */
static int show_prepared_stmt_count(THD *, SHOW_VAR *var, char *buff) {
  var->type = SHOW_LONG;
  var->value = buff;
  mysql_mutex_lock(&LOCK_prepared_stmt_count);
  *((long *)buff) = (long)prepared_stmt_count;
  mysql_mutex_unlock(&LOCK_prepared_stmt_count);
  return 0;
}
```
```cpp
    {"Prepared_stmt_count", (char *)&show_prepared_stmt_count, SHOW_FUNC,
     SHOW_SCOPE_GLOBAL},      /* :9870 —— ★ 只有 GLOBAL，没有 SESSION 版 */
```

**★ 各计数点**（★ 源码证据）：

| 状态变量 | 累加点 |
|---|---|
| `Com_stmt_fetch` | `sql_prepare.cc:1979` `thd->status_var.com_stmt_fetch++;` |
| `Com_stmt_reset` | `sql_prepare.cc:2018` |
| `Com_stmt_send_long_data` | `sql_prepare.cc:2109` |
| `Com_stmt_close` | `sql_prepare.cc:3694`（`deallocate` 内） |
| `Com_stmt_execute` | `sql_prepare.cc:3422`（`execute()` 内） |

```cpp
/** Common part of DEALLOCATE PREPARE and mysqld_stmt_close. */
void Prepared_statement::deallocate(THD *thd) {
  /* We account deallocate in the same manner as mysqld_stmt_close */
  thd->status_var.com_stmt_close++;
  /* Statement map calls delete stmt on erase */
  thd->stmt_map.erase(this);
}
```
> ★ `Com_stmt_close` **同时覆盖**协议层 `COM_STMT_CLOSE` 与 SQL 层 `DEALLOCATE PREPARE`。

**★ 为什么 `Prepared_stmt_count ≠ Com_stmt_prepare − Com_stmt_close`**（`sql/mysqld.cc:1394`，★ 官方解释）：

```cpp
/**
  Current total number of prepared statements in the server. This number
  is exact, and therefore may not be equal to the difference between
  `com_stmt_prepare' and `com_stmt_close' (global status variables), as
  the latter ones account for all registered attempts to prepare
  a statement (including unsuccessful ones). ...
*/
```
> ★ `com_stmt_prepare` 统计的是**所有尝试**（含失败的），`prepared_stmt_count` 是**实际存活数**。

**★ 另一个 PS 相关变量**（`sql/sys_vars.cc:679`，⚠️ 容易被漏）：

```cpp
static Sys_var_long Sys_pfs_max_prepared_stmt_instances(
    "performance_schema_max_prepared_statements_instances",
    "Maximum number of instrumented prepared statements."
    " Use 0 to disable, -1 for automated scaling.",
    READ_ONLY GLOBAL_VAR(pfs_param.m_prepared_stmt_sizing),
    CMD_LINE(REQUIRED_ARG), VALID_RANGE(-1, num_prepared_stmt_limit),
    DEFAULT(PFS_AUTOSCALE_VALUE), BLOCK_SIZE(1), PFS_TRAILING_PROPERTIES);
```
> ★ 只读、启动前设置，控制 `performance_schema.prepared_statements_instances` 表行数上限。**除此之外 `sys_vars.cc` 里没有其它名字含 prepared 的系统变量**。
> ⚠️ **`prepared_stmt_count` 不是系统变量**——它是 `mysqld.cc:1404` 的普通全局 `ulong`。

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| "五个 COM_STMT_* 命令" | **六个**（漏了 `COM_STMT_SEND_LONG_DATA`）；且 `COM_SET_OPTION` 夹在 CLOSE 与 FETCH 之间 |
| 服务端有 `mysqld_stmt_send_long_data()` | **不存在**（仅注释残留）→ 真名 `mysql_stmt_get_longdata()` |
| `Server_side_cursor` 在 `sql_prepare.h` | 在 **`sql/sql_cursor.h:51`**；`sql_prepare.h:340` 只是前向声明 |
| 8.0 有多种游标类（含 `Cursor` 基类） | 只有 **`Materialized_cursor`**（`final`）；`Cursor` 基类**不存在** |
| "物化是为了不让客户端在 scan 中间改表" | **源码里没这句话** → 真实证据是 `create_result_table` + `instantiate_tmp_table` + handler 跨执行存活 + 元数据需显式保留 |
| `Prepared_statement_map` 有容量上限 | 容器**无上限**；上限来自**全局**计数器 `max_prepared_stmt_count`（默认 16382） |
| PS 与 `thd->lex` 共用同一个 LEX | **不共用** → 普通查询复用 `main_lex`；PS 有独立 `m_lex`，执行时 `set_thd_to_ps()` 借壳 |
| `LEX::safe_to_cache_query` 是 plan cache 开关 | **不是** → 注释："Used to be for the query cache, but is now used to find out if an expression is usable for partitioning" |
| "社区版有 plan cache 只是没开" | **从未有过** → `sql/` 下 `plan_cache` grep **0 命中**，无相关文件 |
| "PS 每次执行会命中缓存跳过优化" | **错** → `execute_loop()` 只有三类 reprepare 分支，`mysql_execute_command()` 每次都完整跑 |
| 客户端 `mysql_stmt_prepare` 是本地插值 | **错** → 真的发 `COM_STMT_PREPARE` 往返；本地插值是 JDBC / PDO 的行为，**不在本仓库** |
| `prepared_stmt_count` 是系统变量 | **不是** → 是 `mysqld.cc:1404` 的全局 `ulong`；`Prepared_stmt_count` 才是状态变量（`SHOW_FUNC`） |
| `Com_stmt_*` 都注册在 `status_vars[]` | 前 6 个走 `com_status_vars[]`（小写名）；只有 `Com_stmt_reprepare` 单独在 `status_vars[]` |
| `COM_STMT_FETCH` 有游标方向字段 | 没有 → 报文只有 8 字节（`stmt_id + num_rows`），只能顺序向前 |
| 服务端支持可滚动游标 | **不支持** → 源码注释 "Only read-only non-scrollable cursors"；客户端 `> CURSOR_TYPE_READ_ONLY` 直接拒绝 |
| `materialize_cursor()` / `open_cursor()` 函数 | **都不存在** → 类是 `Query_result_materialize`；PS 路径是 `execute()` 内联 `if (open_cursor)`，SP 路径是 `mysql_open_cursor()` |
| `Prepared_stmt_count = Com_stmt_prepare − Com_stmt_close` | **不等** → 后者统计"所有尝试（含失败）"，前者是精确存活数 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → C API Prepared Statement Interface*
- *MySQL 8.0 Reference Manual → Prepared Statements*（`PREPARE` / `EXECUTE` / `DEALLOCATE`）

**WorkLog**
- WL#6570 "prepare once, execute many" —— 源码中散落的 `@todo` 指向它，说明 prepare/execute 分离至今仍是未完成的历史重构

**相关文档**
- 上游（协议分发与报文）见 [`01_protocol_to_dispatch.md`](01_protocol_to_dispatch.md)
- prepare 期的逻辑改写（永久变换 vs 临时变换）见 [`06_resolver_prepare.md`](06_resolver_prepare.md)
- 每次 execute 重做的 optimize 过程见 [`07_optimize/00_overview.md`](07_optimize/00_overview.md)

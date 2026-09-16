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

#### 代价要落到具体后果（权衡的另一半）

**⚠️ 误解："用 prepared statement 就不会走错索引"——恰恰相反。**

正因为每次 execute 都基于**当前参数值**重新估算，同一个 PS 在不同执行之间**可能切换执行计划**。这不是 bug，而是上面这个设计的必然：参数在 optimize 期是可见的（`const_for_execution() == true`），优化器当然要利用它。

其他真实代价：

- **单次执行的点查，PS 大概率比文本协议更慢**——多 2 个 RTT（prepare + close），却没省掉 optimize
- **PS 的真实收益**是省掉 parse + resolve + 永久改写 + 客户端拼串/转义，以及二进制结果集的转换开销
- 所以 PS 的价值只在**高频重复执行**和**安全性**，不是"执行更快"的万能药

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

### 理论溯源

"prepare 一次、执行多次"是数据库的经典设计，但各家做到哪一步不同：

- **Oracle / SQL Server**：做到**全局 plan cache**——跨会话共享执行计划，按 SQL 文本/digest 哈希，代价是需要全局 latch、跨会话失效广播、LRU 老化
- **PostgreSQL**：`PREPARE` 后前 5 次用 **custom plan**（代入实际参数值优化），之后才考虑切 **generic plan**——等于承认"参数值影响计划"，用执行次数来试探
- **MySQL 社区版**：只做到**语法结构缓存**，optimize 每次重做，靠 reprepare 兜底元数据变化

MySQL 的定位是"每连接自给自足、无全局共享失效"，这与 plan cache 的全局成本模型冲突。

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

### 他库对比与演进动机

| | Oracle / SQL Server | PostgreSQL | MySQL 社区版 |
|---|---|---|---|
| 缓存粒度 | 全局执行计划 | 每会话 prepared plan | 每会话**仅语法树** |
| 跨会话共享 | ✅ | ❌ | ❌ |
| optimize 重做 | 否（命中缓存） | 前 5 次是，后可切 generic | **每次都重做** |
| 失效机制 | 全局广播 | 版本检查 | reprepare（最多 3 次） |

**演进动机**：源码里散落的 `@todo WL#6570` 指向一条长期重构（"prepare once, execute many"），目标是彻底理清 prepare/execute 边界、去掉 `unprepare()` 残留。**社区版不是"不做 plan cache"，而是"连 prepared statement 本身的 prepare/execute 分离都还没做到位"**——plan cache 在这个优先级序列里排得非常靠后。

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
[types_supplied_by_client:1]             ← 后面是否带类型信息
  ├─ 若为 1：[type:2][unsigned_flag:1] × param_count
[参数值] × param_count
  ├─ 非字符串：定长原生编码（1/2/4/8 字节）
  └─ 字符串/二进制：length-encoded 长度 + 数据
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

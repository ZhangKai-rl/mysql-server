# 存储程序：存储过程 / 函数 / 触发器 / 事件

> 存储程序是 SQL 层的**独立子系统**：有自己的解析器分支、指令流（`sp_instr`）、编译产物缓存（`sp_head`）、运行时上下文（`sp_rcontext`）。它是"一条 SQL 的旅程"之外的**第二条执行路径**——主链篇不会回头讲它。
>
> **边界**：本篇讲 **SQL 层语义**。`CREATE PROCEDURE/FUNCTION/TRIGGER/EVENT/VIEW` 这些对象**怎么持久化进 Data Dictionary**（`mysql.routines` / `triggers` / `events` / `tables` 的字段映射、参数、依赖采集、时区与 definer 处理）见姊妹篇 [`dd/stored_program_dd.md`](dd/stored_program_dd.md)；DD 本身的缓存/升级/SDI 见 [`dd/dd.md`](dd/dd.md)。读出侧（`I_S.ROUTINES` / `EVENTS` / `TRIGGERS` 视图）见 [`dd/system_views.md`](dd/system_views.md)。

## 目录

- [一句话机制](#一句话机制)
- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、主链路：CALL 的完整旅程](#一主链路call-的完整旅程)
  - [二、对象模型：sp_head 与指令流](#二对象模型sp_head-与指令流)
  - [三、指令集：20 条指令](#三指令集20-条指令)
  - [四、编译：单遍代码生成与 SP 专属优化器](#四编译单遍代码生成与-sp-专属优化器)
  - [五、执行期上下文：sp_rcontext](#五执行期上下文sp_rcontext)
  - [六、缓存：sp_cache](#六缓存sp_cache)
  - [七、触发器与事件](#七触发器与事件)
  - [八、权限、binlog 与限制](#八权限binlog-与限制)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 一句话机制

> 存储程序在 **CREATE 时（或首次从数据字典加载时）被解析一次**，编译成一条 `sp_instr` 指令流，连同每条语句的 LEX 树一起常驻在 `sp_head` 里；`sp_head` 按连接缓存在 `THD::sp_proc_cache / sp_func_cache`。调用时走**指令循环**：逐条 `sp_instr::execute()`，而语句类指令内部直接调 `mysql_execute_command()` **跳回主链**——于是形成"主链 → 指令流 → 主链 → …"的交替嵌套。

---

## 概述

### 是什么

存储程序是**预编译成指令流、缓存在 `sp_head` 里、按调用解释执行**的 SQL 层程序单元，包含四类：存储过程（`PROCEDURE`）、存储函数（`FUNCTION`）、触发器（`TRIGGER`）、事件（`EVENT`）。

源码对 `sp_head` 的定位一句话说清：

```cpp
  sp_head represents one instance of a stored program. It might be of any type
  (stored procedure, function, trigger, event).
```

**一个 `sp_head` = 一个存储程序实例**（注意是"实例"——递归时同一程序会有多个实例，见 [六](#六缓存sp_cache)）。

### 用途

- **封装业务逻辑**，减少网络往返（一次 CALL 代替 N 条语句）
- **最小权限**：用户只有 `EXECUTE` 权限，通过 `SQL SECURITY DEFINER` 以定义者身份访问无权访问的表
- **触发器**用于审计、派生列维护、跨表一致性
- **事件**用于定时任务（替代外部 cron）

**不适用场景**：重计算。因为"把循环写在 SQL 里"几乎总比"用 SP 循环逐行处理"快——每条指令都有固定的 digest/PSI/arena 开销（见 [九](#misc) 的性能陷阱）。

### 版本演进

| 维度 | 变化 |
|---|---|
| **元数据存储** | 5.x 存 `mysql.proc` 表；**8.0 起完全 DD 化**（`dd::Routine` / `dd::Event` / 触发器 DD 对象）。⚠️ 8.0.39 **已无 `mysql.proc` 读写路径**——这是易混淆点 |
| **原子 DDL** | 8.0 的原子 DDL 让 `CREATE/DROP PROCEDURE` 与 DD、binlog 原子提交（`sp_create_routine()` 里 `trans_commit_stmt` + `trans_commit`） |
| **触发器** | 8.0 支持同一 (event, action) 组合的**多个**触发器（此前每组合仅一个），改由 `Trigger_chain` 管理 |
| **权限变量** | `log_bin_trust_function_creators` 已标记 `DEPRECATED` |

> ⚠️ `sp_head` 本身的命名与结构从 5.x 延续至今；源码自认缓存实现"by no mean final"（见 [六](#六缓存sp_cache)）。

---

## 理论基础

### ① 为什么是"解释执行"而不是编译成原生代码

三个约束：

1. **SQL 语句本身就没有编译后端**。MySQL 没有 LLVM/JIT 后端，一条普通 SQL 也是"优化器产出 AccessPath → 迭代器解释执行"（见 [`query/09_executor_iterator.md`](query/09_executor_iterator.md)）。存储程序编译成原生代码没有可依托的后端。
2. **"编译"的收益有限**。存储程序里的 SQL **解析只做一次**，但**优化器每次执行都要重跑**（见 [一.3](#3指令循环从主链跳进指令流)）。真正能省掉的只有 parse 和语法树构建，收益不足以支撑一个原生编译器。
3. **跨会话共享需要平台无关的中间表示**。`sp_instr` 指令流就是这个 IR——它是 C++ 对象序列，天然可缓存、可重入。

**他库对比**：Oracle 的 PL/SQL 有独立的PL/SQL 引擎并支持原生编译；PostgreSQL 的 PL/pgSQL 也是解释执行，但 11+ 支持表达式 JIT。MySQL 社区长期没有投入这块，因为它定位在"OLTP 简单查询"，而存储程序恰恰是 Oracle 迁移场景的重灾区。

### ② ★ 为什么 `sp_head` 必须按连接缓存，而不能全局共享

源码给出了根因——`sp_head` **既不线程安全，也不无状态**：

```
  To achieve this, the implementation of #sp_head must be
  both thread-safe and stateless.
  Unfortunately, it is neither:

  - The class #sp_head is composed of #sp_instr instructions to represent the
    code, and these instructions in turn depend on Item objects … The various
    C++ Item classes are not currently thread-safe, since the evaluation of
    an Item at runtime involves methods like Item::fix_fields(), which modify
    the internal state of items, making them impossible to safely evaluate
    concurrently.

  - The class #sp_head itself contains attributes that describe
    the SQL logic of a Stored Program (which are safe to share),
    mixed with attributes that relate to the evaluation
    of this logic in a given instance to a Stored Program call
    (mostly the MEM_ROOT memory pool used during execution),
    which by definition cannot be shared.
```

即：**`Item::fix_fields()` 会修改 Item 的内部状态**，两个线程同时执行同一条指令会互相破坏。所以每个连接有自己的缓存。

连带后果是**递归也需要多个实例**：同一程序在缓存里是一条按递归层级排序的链，每层一个 `sp_head`。

### ③ 为什么触发器只能是行级（`FOR EACH ROW`）

MySQL **没有语句级触发器**（社区事实）。这直接影响复制语义：行级触发器对每一行都触发一次，在 statement 格式下从库重放会再触发一遍，若触发器含非确定性逻辑（`NOW()`、`UUID()`）就会主从不一致。

### ④ 为什么存储程序对复制不友好

- **非确定性**：`NOW()` / `UUID()` / `RAND()` 在 SP 里，SBR 下主从结果不同
- **临时表**：SP 内建临时表在 SBR 下不安全
- **unsafe 标记会向上传染**：

```cpp
    /*
      If this routine needs row-based binary logging, the entire top statement
      too (we cannot switch from statement-based to row-based only for this
      routine, as in statement-based the top-statement may be binlogged and
      the sub-statements not).
    */
    prelocking_ctx->set_stmt_unsafe_flags(unsafe_flags);
```

（与 [`replication/binlog_sbr.md`](replication/binlog_sbr.md) 的 unsafe 判定衔接。）

### ⑤ 为什么 SP 变量存在一张临时表里

见 [五.1](#1变量一张临时表--item_field-代理)——这是本篇最有意思的设计之一。

---

## 核心实现

## 一、主链路：CALL 的完整旅程

```
① 解析        PT_call::make_cmd() → Sql_cmd_call（并记录 prelocking 依赖）
② 预检查      precheck()   : EXECUTE 权限
③ prepare     prepare_inner(): 从缓存取（cache_only=true），校验参数个数/类型
④ 权限        check_privileges()：按 IN/OUT/INOUT 判 SELECT/UPDATE
⑤ execute     execute_inner():
                 sp_setup_routine()  取一个空闲 sp_head 实例（递归层）
                 ├─ 参数装填、切安全上下文、MYSQL_START_SP
                 └─ sp_head::execute_procedure() → 指令循环
⑥ my_ok()
```

### 1. 取实例：`sp_setup_routine()`

```cpp
  const ulong depth = type == enum_sp_type::PROCEDURE
                          ? thd->variables.max_sp_recursion_depth : 0;
  if (sp->m_first_free_instance) {
    assert(!(sp->m_first_free_instance->m_flags & sp_head::IS_INVOKED));
    if (sp->m_first_free_instance->m_recursion_level > depth) {
      recursion_level_error(thd, sp);
      return nullptr;
    }
    return sp->m_first_free_instance;
  }
  ulong level = sp->m_last_cached_sp->m_recursion_level + 1;
  if (level > depth) { recursion_level_error(thd, sp); return nullptr; }
  ...
  sp->m_last_cached_sp->m_next_cached_sp = new_sp;
  new_sp->m_recursion_level = level;
```

注意 **存储函数的 depth 恒为 0**——函数不能递归。

### 2. `execute_procedure()` 的五步

```
    Execute a procedure.
    The function does the following steps:
     - Set all parameters
     - changes security context for SUID routines
     - call sp_head::execute
     - copy back values of INOUT and OUT parameters
     - restores security context
```

### 3. 指令循环：从主链跳进指令流

`sp_head::execute()` 的主循环（核心，贴完整）：

```cpp
  do {
    sp_instr *i;

    /* get_instr returns NULL when we're done. */
    i = get_instr(ip);
    if (i == nullptr) { break; }

    /*
      We need to reset start_time to allow for time to flow inside a stored
      procedure. This is only done for SP since time is suppose to be constant
      during execution of triggers and functions.
    */
    reset_start_time_for_sp(thd);

    thd->stmt_arena = &i->m_arena;

    sql_digest_state digest_state;
    sql_digest_state *parent_digest = thd->m_digest;
    thd->m_digest = &digest_state;

    PSI_statement_locker_state psi_state;
    PSI_statement_info *psi_info = i->get_psi_info();
    PSI_statement_locker *parent_locker;
    parent_locker = thd->m_statement_psi;
    thd->m_statement_psi = MYSQL_START_STATEMENT(
        &psi_state, psi_info->m_key, thd->db().str, thd->db().length,
        thd->charset(), this->m_sp_share);

    err_status = i->execute(thd, &ip);

    MYSQL_END_STATEMENT(thd->m_statement_psi, thd->get_stmt_da());
    thd->m_statement_psi = parent_locker;
    thd->m_digest = parent_digest;

    cleanup_items(i->m_arena.item_list());
    thd->cleanup_after_query();
    // Release memory allocated during execution of the instruction
    execute_mem_root.ClearForReuse();

    /*
      Find and process SQL handlers unless it is a fatal error (fatal
      errors are not catchable by SQL handlers) or the connection has been
      killed during execution.
    */
    if (!thd->is_fatal_error() && !thd->killed &&
        thd->sp_runtime_ctx->handle_sql_condition(thd, &ip, i)) {
      err_status = false;
    }

  } while (!err_status && !thd->killed && !thd->is_fatal_error());
```

四个要点：

1. **`ip` 是循环的状态机变量**，由 `i->execute(thd, &ip)` 就地改写；`handle_sql_condition(thd, &ip, i)` **也可能改写 `ip`**（跳进 handler 体）
2. **每条指令一个执行期 arena**，执行完 `ClearForReuse()`——这就是为什么游标、变量必须另找归宿（见 [五](#五执行期上下文sp_rcontext)）
3. **每条指令单独 digest + 单独 PSI 事件**——所以 SP 内部语句在 PFS 里是各自独立的行（见 [`infra/pfs_statement.md`](../infra/observability/pfs/pfs_statement.md)）
4. 退出条件：`err_status` / `killed` / `is_fatal_error()`；正常结束是 `get_instr(ip) == nullptr`（`freturn` 写了 `UINT_MAX`）

进入循环前还有一整套环境切换（这是"SP 是语句中的语句"的关键）：切 `db`、**压新诊断区**、备份并切换 `sql_mode` 为 CREATE 时的值、切 `lex`、切 arena。

### 4. ★ 指令跳回主链：`sp_instr_stmt::exec_core()`

**只有 15 行，但它是两条执行路径的接缝**：

```cpp
bool sp_instr_stmt::exec_core(THD *thd, uint *nextp) {
  LEX *const lex = thd->lex;
  lex->set_sp_current_parsing_ctx(get_parsing_ctx());
  lex->sphead = thd->sp_runtime_ctx->sp;

  PSI_statement_locker *statement_psi_saved = thd->m_statement_psi;

  assert(lex->m_sql_cmd == nullptr || lex->m_sql_cmd->is_part_of_sp());

  bool rc = mysql_execute_command(thd);

  lex->set_sp_current_parsing_ctx(nullptr);
  lex->sphead = nullptr;
  thd->m_statement_psi = statement_psi_saved;

  *nextp = get_ip() + 1;

  return rc;
}
```

即：**直接调用主链入口 `mysql_execute_command()`**。

### 5. ★ 关键答案：SP 里的 SQL 每次重新解析吗？

**不重新解析，但优化器每次都跑。**

- **parse：只做一次**（CREATE 时或首次 `db_load_routine()` 时）。LEX 树常驻 `sp_head::main_mem_root`（注释：`SP-persistent memory root (for instructions and expressions)`）
- **optimize + execute：每次完整走一遍**（`mysql_execute_command` 照常开表、权限检查、优化器选计划）

**会触发重新解析的三种情况**：

1. 元数据变更 → `ER_NEED_REPREPARE` → 下一轮 `parse_expr()` 重解析（`validate_lex_and_execute_core()` 的 while 循环）
2. **含 UDF 的语句每次执行都重新解析**——这是一个明确的性能坑：

```cpp
    if (is_invalid() || (m_lex->has_udf() && !m_first_execution)) {
      free_lex();
      LEX *lex = parse_expr(thd, thd->sp_runtime_ctx->sp);
      ...
```

3. 游标 OPEN 可能触发重解析（`sp_instr_cpush` 注释：`OPEN may lead to re-parsing of the SELECT-statement`）

### 6. 与预编译的关系

**SP 里的普通 SQL 不是 `Prepared_statement`**——它由 `sp_instr_stmt` 持有自己在解析期构建的 LEX，执行时装进 `thd->lex`。全程没有 `Prepared_statement` 对象、没有 `COM_STMT_PREPARE`。

但**复用了 PS 的元数据失效机制**（`Reprepare_observer`、`CF_REEXECUTION_FRAGILE`、`check_preparation_invalid()`）——可以理解为"一个内嵌的、隐式的 prepared statement"。

⚠️ **唯一的例外是游标**：游标走真正的 preparable statement 路径，且被强制设为不可缓存：

```cpp
    /*
      Cursors cause queries to depend on external state, so they are
      noncacheable.
    */
    cursor_lex->safe_to_cache_query = false;
```

---

## 二、对象模型：sp_head 与指令流

### 为什么叫 "head"

⚠️ **源码无直接证据解释命名**。但结构上很清楚：

1. **一个存储程序 = 一个 sp_head（头）+ N 条 sp_instr（体）**。文档原文：

```
In every case, the parser reads the SQL text stream that
represents the code as input, and creates an internal representation
of the Stored Program as output, with one C++ object of type #sp_head.
```

2. `sp_head` 的成员全是"头部元数据"：`m_db / m_name / m_qname / m_params / m_body / m_defstr / m_definer_user / m_definer_host`
3. 对应 SQL 语法里 routine 的 "head"（`CREATE PROCEDURE ... (参数) 特性`）与 body（`BEGIN…END`）
4. 历史沿革：8.0 之前例程存在 `mysql.proc`，**一行就是一个"例程头"**；迁到 DD 后 `sp_head` 仍是那"一行"的内存表示

### 关键成员

| 成员 | 含义 |
|---|---|
| `Mem_root_array<sp_instr *> m_instructions` | 指令数组（注释：`/// The SP-instructions.`） |
| `MEM_ROOT main_mem_root` | **常驻内存池**——指令与表达式存活于此，跨多次执行 |
| `sp_pcontext *m_root_parsing_ctx` | 根解析上下文（最外层 BEGIN..END） |
| `sql_mode_t m_sql_mode` | CREATE 时的 `sql_mode`（执行时切回这个值） |
| `Security_context m_security_ctx` | definer 权限上下文 |
| `int64 m_sp_cache_version` | 入缓存时的全局版本号（失效判定） |
| `Stored_program_creation_ctx *m_creation_ctx` | CREATE 时字符集/排序规则快照 |
| `ulong m_recursion_level` | 递归层级（从 0 开始） |
| `sp_head *m_next_cached_sp / m_first_instance / m_first_free_instance / m_last_cached_sp` | **同一程序的多个实例链表（为递归）** |

`m_flags` 是一份"程序静态特征清单"，值得原样引用：

```cpp
    HAS_RETURN = 1,             // For FUNCTIONs only: is set if has RETURN
    MULTI_RESULTS = 8,          // Is set if a procedure with SELECT(s)
    CONTAINS_DYNAMIC_SQL = 16,  // Is set if a procedure with PREPARE/EXECUTE
    IS_INVOKED = 32,            // Is set if this sp_head is being used
    HAS_SET_AUTOCOMMIT_STMT = 64,
    HAS_COMMIT_OR_ROLLBACK = 128,
    LOG_SLOW_STATEMENTS = 256,  // Used by events
    LOG_GENERAL_LOG = 512,      // Used by events
    HAS_SQLCOM_RESET = 1024,
    HAS_SQLCOM_FLUSH = 2048,
    MODIFIES_DATA = 4096,
    HAS_TEMP_TABLE_DDL = 8192
```

这些 flag 直接驱动 [八](#八权限binlog-与限制) 的禁用清单与 binlog unsafe 判定。

⚠️ 注：`CONTAINS SQL` / `READS SQL DATA` / `MODIFIES SQL DATA` 这些用户声明的**特性基本被忽略**——`MODIFIES_DATA` 的注释明说：

```
      Marks routines that directly (i.e. not by calling other routines)
      change tables. … The latter are accepted by parser but pretty much
      ignored after that. We don't rely on them: a) for compatibility reasons.
      b) because in CONTAINS SQL case they don't provide enough information anyway.
```

### 生命周期

- **创建**：构造函数私有，只能通过 `sp_start_parsing()` 创建；预留 32 条指令
- **销毁**：`sp_head::destroy()`——注意 sp_head 本体就分配在自己的 `main_mem_root` 里：

```cpp
void sp_head::destroy(sp_head *sp) {
  /* Pull out main_mem_root as free_root will free the sp */
  MEM_ROOT own_root = std::move(sp->main_mem_root);
  sp->~sp_head();
  own_root.Clear();
}
```

⚠️ **8.0.39 的 `sp_head` 没有 `ref_count`**（检索无命中）。生命周期靠：① `sp_cache` 里 `unique_ptr` 的所有权；② `IS_INVOKED` 标志位 + `m_first_free_instance` 空闲链（不是引用计数）。

---

## 三、指令集：20 条指令

### 基类契约

```cpp
  virtual bool execute(THD *thd, uint *nextp) = 0;
  uint get_ip() const { return m_ip; }
  virtual uint get_cont_dest() const { return get_ip() + 1; }
```

`*nextp` 的契约注释很关键：

```
    @param[out] nextp  index of the next instruction to execute. (For most
                       instructions this will be the instruction following this
                       one). Note that this parameter is undefined in case of
                       errors, use get_cont_dest() to find the continuation
                       instruction for CONTINUE error handlers.
```

### 完整清单

| 指令 | 用途 |
|---|---|
| **`sp_instr_stmt`** | **几乎所有常规 SQL 语句**（注释：`represents almost all conventional SQL-statements`） |
| **`sp_instr_set`** | SP 变量赋值（`SET v = ...`） |
| `sp_instr_set_trigger_field` | 触发器里 `SET NEW.col = ...` |
| **`sp_instr_freturn`** | 存储函数 RETURN（写 `*nextp = UINT_MAX` 终止循环） |
| **`sp_instr_jump`** | 无条件跳转（`*nextp = m_dest`） |
| **`sp_instr_jump_if_not`** | 条件跳转（IF/WHILE/REPEAT 的反条件跳） |
| `sp_instr_set_case_expr` | 计算并保存 CASE 表达式 |
| `sp_instr_jump_case_when` | CASE 的 WHEN 分支比较跳转 |
| **`sp_instr_hpush_jump`** | 进入 handler 保护块：压 handler 入栈并跳过 handler 体 |
| **`sp_instr_hpop`** | 离开 handler 保护块 |
| **`sp_instr_hreturn`** | handler 体执行结束的返回 |
| **`sp_instr_cpush`** | **既实现 DECLARE CURSOR（入栈）也实现 OPEN（真正打开）** |
| `sp_instr_cpop` | BEGIN..END 结束时移除游标 |
| `sp_instr_copen` | OPEN（转调 `cpush::exec_core`） |
| `sp_instr_cclose` / `sp_instr_cfetch` | CLOSE / FETCH |
| `sp_instr_error` | 主动抛错（CASE 无匹配支时） |

`sp_instr_cpush` 身兼两职的原因（注释）：

```
  execute() operation "implements" DECLARE CURSOR statement -- it merely pushes
  a new cursor object into the stack in sp_rcontext object.
  exec_core() operation implements OPEN statement. It is important to implement
  OPEN statement in this instruction, because OPEN may lead to re-parsing of the
  SELECT-statement. So, the original Arena and parsing context must be used.
```

### ★ `get_cont_dest()`：为什么指令需要"继续地址"

基类默认是"下一条指令"，但带表达式的跳转指令**单独保存 `m_cont_dest`**，与跳转目标 `m_dest` 分开。

原因——**一条条件跳转指令实际有三个出口**：

```
  Note that the instruction jump_if_not 3(7) at position 0
  can actually jump to three locations:
  - When the evaluation of the condition "x < 0" is true,
    the next instruction will be position 1 (the "then" branch),
  - When the evaluation of the condition "x < 0" is false,
    the next instruction will be position 3 (the "else" branch),
  - When the evaluation of the condition "x < 0" results in an error,
    and when a continue handler exists for the error,
    the next instruction will be position 7,
    known as the "continuation" destination.
```

即：**出错被 CONTINUE handler 捕获时 `*nextp` 是未定义的，此时用 `get_cont_dest()`**——它通常是整个 IF/CASE 之后的那一条指令，与 if 的两个分支出口都不同。

### handler 三条指令在指令流中的位置

`SHOW PROCEDURE CODE` 的真实输出（`proc_2` 的 handler 版）：

```
Pos     Instruction
0       stmt 0 "SELECT "Start""
1       stmt 5 "INSERT INTO t1 VALUES (1)"
2       hpush_jump 5 1 CONTINUE
3       stmt 0 "SELECT "Oops""
4       hreturn 1
5       stmt 5 "INSERT INTO t1 VALUES (2)"
6       stmt 5 "INSERT INTO t1 VALUES (2)"
7       hpop 1
8       stmt 5 "INSERT INTO t1 VALUES (3)"
9       stmt 0 "SELECT "Finish""
```

`hpush_jump 5 1 CONTINUE` 的含义：`5` 是跳过的目标（handler 体之后），`1` 是**解析期条件表里的索引**（此处是 sqlexception），`CONTINUE` 是类型。

流程：**`hpush_jump` 进栈 → 任意指令出错 → 主循环 `handle_sql_condition()` 改写 `ip` → 跳到 handler 体 → `hreturn` 出栈并设 `ip` 为 `m_dest`(EXIT) 或 `continue_ip`(CONTINUE) → `exit_handler()` 弹诊断区**。

> handler 的**匹配算法、诊断区压栈、CONTINUE/EXIT 语义**详见 [`infra/diagnostics.md`](infra/diagnostics.md)。

---

## 四、编译：单遍代码生成与 SP 专属优化器

### 单遍 + backpatch

解析是**单遍**的，前向跳转的目标地址未知，用 backpatch 列表：

```
  The final destination of the label for the END CASE is not known yet,
  and the list of all the instructions … that need to point to this unknown
  destination (represented as ??) is maintained in a temporary structure
  used during code generation only. This structure is called the context
  back patch list.
  When the destination label is finally resolved to a destination (12),
  all the instructions pointing to that label … are back patched …
```

`sp_instr_jump::backpatch` 带断言防重复：

```cpp
  void backpatch(uint dest) override {
    /* Calling backpatch twice is a logic flaw in jump resolution. */
    assert(m_dest == 0);
    m_dest = dest;
  }
```

### SP 专属"优化器"（≠ 查询优化器）

`sp_head::optimize()` 做两件事：死代码删除、跳转短路。

```
  @attention Do not confuse #sp_head::optimize() with the component named
  the optimizer, as they are very different. The former is specific to
  Stored Programs, and focuses on improving the flow of statements …
```

它也必须知道 CONTINUE 的边：

```
    For continue handlers, all instructions in the scope of the handler
    are possible leads. For example, the instruction after freturn might
    be executed if the freturn triggers the condition handled by the
    continue handler.
```

### 一个"黑魔法"：先建空壳

```
    Note that there is some black magic here: since we are still currently parsing
    the content of a Stored Program (the DECLARE %CURSOR statement), the final
    "syntax" tree for the Stored Program (#sp_head) is not supposed to exist yet.
    The reason the #sp_head object is already available is that the actions in the
    <tt>CREATE PROCEDURE</tt>, … are implemented as a descendant parser
    (it created an empty #sp_head object first, filling the content later).
    Mixing code that way (descendant actions with ascendant parsing)
    is extremely sensitive to changes.
```

即 `CREATE` 的 bison action 是**后代解析器**——先建空 `sp_head`，再往里填指令。这也导致一个限制：

```
  A limiting consequence of this approach is that a stored program
  does not support nesting: it is impossible to embed one
  <tt>CREATE PROCEDURE</tt> into another,
  since the parser currently may only support one #sp_head object at a time.
```

---

## 五、执行期上下文：sp_rcontext

`sp_rcontext` 的类注释把三者分工说得很清楚：

```
  This class is a runtime context of a Stored Routine. It is used in an
  execution and is intended to contain all dynamic objects …, such as:
    - stored routine variables;
    - cursors;
    - handlers;

  Runtime context is used with sp_head class. sp_head class is intended to
  contain all static things, related to the stored routines (code, for example).
  ...
  There is a parsing context (an instance of sp_pcontext class), which is used
  on parsing stage. However, now it contains some necessary for an execution
  things … That's why runtime context needs a reference to the parsing context.
```

| 类 | 阶段 | 内容 |
|---|---|---|
| `sp_pcontext` | 解析期 | 符号表、作用域树、条件/handler 声明 |
| `sp_head` | 静态 | 指令流、LEX 树、元数据 |
| `sp_rcontext` | 运行期 | 变量值、游标、handler 栈 |

### 1. ★ 变量：一张临时表 + `Item_field` 代理

这是本篇最有意思的设计：**SP 变量不是数组，而是一张内部临时表**。

```
  Internally, a temporary table is created, with the following structure:
CREATE TEMPORARY TABLE `proc_7_vars` (
  `v1` int(11) DEFAULT NULL, …
) ENGINE=MyISAM DEFAULT CHARSET=latin1

  The real name of the table and the columns are purely internal,
  and the table is not accessible to regular statements for DDL or DML …
  The #TABLE handle that implements all the local variable storage
  is the member #sp_rcontext::m_var_table

  Inside a statement, local variables in a Stored Program
  are represented by the dedicated C++ class #Item_splocal.
  #Item_splocal really is a proxy exposing the interface needed to support #Item,
  which delegates to the underlying #sp_rcontext for reading or writing
  local variable values.
```

```cpp
bool sp_rcontext::init_var_items(THD *thd) {
  uint num_vars = m_root_parsing_ctx->max_var_index();
  m_var_items.reset(static_cast<Item **>(thd->alloc(num_vars * sizeof(Item *))), num_vars);
  for (uint idx = 0; idx < num_vars; ++idx) {
    if (!(m_var_items[idx] = new Item_field(m_var_table->field[idx]))) return true;
  }
  return false;
}
```

**为什么要这样**：变量可以是 BLOB/TEXT/DECIMAL 等任意类型，借 `TABLE`/`Field` 就复用了现成的类型系统与 `sp_eval_expr()` 的类型转换/存储逻辑。

### 2. 变量的作用域与查找

**编译期**：`sp_pcontext` 树为每个 BEGIN..END 建节点，以**深度优先**给所有符号统一编号（跨作用域不重号）：

```
    - Root:x --> 0, Root:y --> 1, Root:v1 --> 2, Root:v2 --> 3, Root:v3 --> 4,
    - A:v1 --> 5, A:v4 --> 6,
    - B:v2 --> 7, B:v4 --> 8,

    - There are two variables named v1, where the variable v1 from block A
      (represented as v1@5) eclipses the variable v1 from the root block
      (represented as v1@2).
```

**运行期**：**不按名字查**，按编译期算好的 offset 直接索引（`set_variable(thd, m_offset, ...)`）。名字只在 debug 版 `print()` 里用。

### 3. ★ 游标：为什么必须物化到临时表

```cpp
/**
  Materialized_cursor -- an insensitive materialized server-side
  cursor. The result set of this cursor is saved in a temporary
  table at open. The cursor itself is simply an interface for the
  handler of the temporary table.
*/
```

**为什么必须物化**，三个理由：

1. **游标是"迟钝的"(insensitive)**：SQL 标准要求游标打开后的结果集是快照。而 SP 在 FETCH 之间会执行其它 DML，甚至修改游标自己的基表——若直接读基表，结果集会随后续 DML 变化
2. **SP 的每指令 arena 会清空**：主循环里每条指令执行完 `execute_mem_root.ClearForReuse()`，而游标生命周期跨越多条指令
3. `push_cursor` 的注释也证明了这点：

```
    We should create cursors on the system heap because:
     - they could be (and usually are) used in several instructions,
       thus they can not be stored on an execution mem-root;
     - a cursor can be pushed/popped many times in a loop, having these objects
       on callers' mem-root would lead to a memory leak in every iteration.
```

游标栈必须跟 BEGIN..END 严格对应以便及时关闭（`pop_cursors()` 里自动 `close()`，SP 结束时 `pop_all_cursors()`）。

### 4. handler 栈

- `m_visible_handlers`：**已声明可见**的 handler 栈（由 `hpush_jump`/`hpop` 驱动）。为什么是栈：

```
    The SQL precedence rules for HANDLER dictates that
    the last installed (inner most) handler is always considered first,
    so the natural structure to represent what handler is active is a stack
```

- `m_activated_handlers`：**已激活**的 handler 调用帧

匹配算法与诊断区压栈 → 详见 [`infra/diagnostics.md`](infra/diagnostics.md)。

### 5. 递归：每次 CALL 新建 sp_rcontext

```cpp
  sp_rcontext *proc_runtime_ctx = sp_rcontext::create(thd, m_root_parsing_ctx, nullptr);
  ...
  thd->sp_runtime_ctx = parent_sp_runtime_ctx;
  ::destroy(func_runtime_ctx);
```

⚠️ 源码自己标了 TODO 嫌这个"每次新建"太浪费（见 Misc）。

**递归深度限制**：

| | 值 |
|---|---|
| `max_sp_recursion_depth` | **默认 0**（存储过程默认不能递归），范围 0..255 |
| 存储函数 | **恒为 0**（`ER_SP_NO_RECURSION`） |

---

## 六、缓存：sp_cache

### 位置与 key

**每连接两个**：

```cpp
  sp_rcontext *sp_runtime_ctx;
  sp_cache *sp_proc_cache;
  sp_cache *sp_func_cache;
```

**key = 完全限定名 `m_qname`（`db.name`），大小写不敏感**。

### 失效：全局版本号，不是逐条失效

```cpp
static std::atomic<int64> atomic_Cversion{0};

void sp_cache_invalidate() {
  atomic_Cversion++;
}
```

每个 sp_head 记住入缓存时的版本。回收是**惰性**的，且只在不在 SP 内部时才做：

```cpp
void sp_cache_flush_obsolete(sp_cache **cp, sp_head **sp) {
  if ((*sp)->sp_cache_version() < sp_cache_version() && !(*sp)->is_invoked()) {
    (*cp)->remove(*sp);
    *sp = nullptr;
  }
}
```

⚠️ 注释明确警告：`dont call this function when inside SP`（会让 sp_head 指针变野）。

### ★ 大小限制：不是 LRU，是"整表清空"

```cpp
  void enforce_limit(ulong upper_limit_for_elements) {
    if (m_hashtable.size() > upper_limit_for_elements) m_hashtable.clear();
  }
```

`stored_program_cache` 默认 **256**（范围 16..512K），每条语句结束时检查一次。

⚠️ **一次超限就把所有已编译 SP 全丢掉**——这是很陡的性能悬崖，源码自己列为已知问题。

### 递归为什么要多个实例

```
  /**
    A list of different recursion level instances for the same procedure.
    For every recursion level we have an sp_head instance. This instances
    connected in the list. The list ordered by increasing recursion level
    (m_recursion_level).
  */
```

根因就是 [理论基础②](#--为什么-sp_head-必须按连接缓存而不能全局共享)——sp_head 不线程安全、不无状态。

### 已知的四个缓存问题（源码自述）

```
  - Each #THD has its own cache, so each separate client connection
    to the server uses its own cache. …
  - If a given client constantly opens and closes
    a new connection to the server … the cache will be always empty,
    causing excessive parsing …
  - If a given client constantly keeps an existing connection
    for a long time … the cache size will grow … memory limits or expulsion
    of cold members of the stored procedure cache is not implemented.
  - Calling #sp_cache_invalidate() does not reclaim the cache memory.
```

还有一处"过度失效"：

```
    Note that because #THD::sp_proc_cache and #THD::sp_func_cache are
    both invalidated based on the same #atomic_Cversion counter,
    executing DROP PROCEDURE happens to invalidate the FUNCTION cache as well …
```

### 触发器与事件的缓存另有一套

- **触发器**：`Table_trigger_dispatcher` 挂在 `TABLE` 上，`sp_head *bodies[TRG_EVENT_MAX][TRG_ACTION_MAX]`；TABLE handle 全局缓存跨线程复用，所以触发器**跨连接共享**
- **事件**：**没有缓存**——

```
  There is no caching of #sp_head for multiple scheduling of an event.
  The method #Event_job_data::execute() invokes the parser
  every time an event is executed.
```

---

## 七、触发器与事件

### 触发器：时机严格

DELETE 的实现直接读出顺序：**BEFORE → 行操作 → AFTER**：

```cpp
  if (invoke_before_triggers) {
    if (table->triggers->process_triggers(thd, TRG_EVENT_DELETE,
                                          TRG_ACTION_BEFORE, false)) return true;
  }
  if (const int delete_error = table->file->ha_delete_row(table->record[0]); ...)
  ...
  if (invoke_after_triggers) {
    if (table->triggers->process_triggers(thd, TRG_EVENT_DELETE,
                                          TRG_ACTION_AFTER, false)) return true;
  }
```

**与外键的顺序**：BEFORE 触发器在 `ha_*_row` 之前，AFTER 在其之后；外键检查与级联由**存储引擎在 `ha_*_row` 内部**完成。所以顺序是：**BEFORE → 行操作（含引擎 FK 检查与级联）→ AFTER**。

> ⚠️ `sql/` 层没有显式调用 FK 检查的代码，上述顺序是从调用次序推出的结论，非注释原文。

### 触发器恒为 DEFINER

```cpp
  /*
    While parsing CREATE TRIGGER statement or loading trigger metadata from
    the Data Dictionary we guarantee that definer hasn't empty value.
    It means, that trigger can't never be NOT-SUID.
  */
  assert(m_chistics->suid != SP_IS_NOT_SUID);
```

### 触发器的"递归"

触发器里的 SQL **当然会再触发其它表的触发器**，每次都走 `execute_trigger()` → 新建 `sp_rcontext` → 新的指令循环。

⚠️ **深度限制不是 `max_sp_recursion_depth`**（该变量只在 CALL/函数调用路径生效）。约束来自：`thd->in_sub_stmt |= SUB_STMT_TRIGGER` 的子语句语义 + `sp_head::execute()` 开头的栈深度检查。**源码无"触发器递归层数"变量的直接证据。**

触发器里调用存储过程时，被调过程受"函数/触发器禁用清单"约束：

```cpp
  if (thd->in_sub_stmt) {
    const char *where = (thd->in_sub_stmt & SUB_STMT_TRIGGER ? "trigger" : "function");
    if (sp->is_not_allowed_in_function(where)) return true;
  }
```

### 事件调度器

两个线程类：`Event_scheduler`（调度主循环）+ `Event_worker_thread`（执行）。

**事件体就是一个存储过程**，而且**每次执行都重新解析**：

```cpp
  /*
    The default SQL security of a stored procedure is DEFINER. We
    have already activated the security context of the event, so
    let's execute the procedure with the invoker rights to save on
    resets of security contexts.
  */
  sp_sql->append(STRING_WITH_LEN("() SQL SECURITY INVOKER "));
  sp_sql->append(m_definition.str, m_definition.length);
```

（用 INVOKER 是为了省一次安全上下文重置——事件已经激活了 definer 上下文。）

---

## 八、权限、binlog 与限制

### SQL SECURITY DEFINER / INVOKER

```cpp
bool sp_head::set_security_ctx(THD *thd, Security_context **save_ctx) {
  ...
  if (m_chistics->suid != SP_IS_NOT_SUID &&
      m_security_ctx.change_security_context(thd, definer_user, definer_host,
                                             m_db.str, save_ctx)) return true;
  /*
    If we changed context to run as another user, we need to check the
    access right for the new context again as someone may have revoked
    the right to use the procedure from this user.
  */
  if (*save_ctx &&
      check_routine_access(thd, EXECUTE_ACL, m_db.str, m_name.str, ...)) {
    m_security_ctx.restore_security_context(thd, *save_ctx);
    ...
```

**为什么要切换**：存储程序是"以定义者权限访问调用者无权访问的表"的标准手段（最小权限：只给 `EXECUTE`）。

> 权限语义详见 [`auth/definer.md`](auth/definer.md)。

### binlog

- **CALL 自身从不写 binlog**：

```
    Never write CALL statements into binlog:
    - If the mode is non-prelocked, each statement will be logged separately.
    - If the mode is prelocked, the invoking statement will care about writing
      into binlog.
```

- **SP 内部每条语句各自写 binlog**，且 SP 变量会被替换成 `NAME_CONST(...)`
- **存储函数调用**在 statement 格式下被展开成一个 `SELECT db.func(常量参数)` 事件，并用 `start_union_events/stop_union_events` 包住函数内部子语句
- **CREATE PROCEDURE/FUNCTION/TRIGGER/EVENT** 写 DDL 语句本身

### 限制

**函数与触发器的禁用清单**（`is_not_allowed_in_function()`，一票否决）：

```cpp
    if (m_flags & CONTAINS_DYNAMIC_SQL)
      my_error(ER_STMT_NOT_ALLOWED_IN_SF_OR_TRG, MYF(0), "Dynamic SQL");
    else if (m_flags & MULTI_RESULTS)
      my_error(ER_SP_NO_RETSET, MYF(0), where);
    else if (m_flags & HAS_SET_AUTOCOMMIT_STMT)
      my_error(ER_SP_CANT_SET_AUTOCOMMIT, MYF(0));
    else if (m_flags & HAS_COMMIT_OR_ROLLBACK)
      my_error(ER_COMMIT_NOT_ALLOWED_IN_SF_OR_TRG, MYF(0));
    ...
```

⚠️ **"存储函数不能做 DML"这个常见说法不准确**。准确说法是：函数/触发器不能有**动态 SQL、返回结果集、SET autocommit、COMMIT/ROLLBACK、RESET、FLUSH**；普通 DML 在函数里是允许的（作为子语句参与外层事务）。**存储过程可以做 DML/DDL**。

**递归 CTE vs SP 递归**是两条完全不同的机制：

| | 递归 CTE | SP 递归 |
|---|---|---|
| 层次 | 主链优化器/执行器（查询执行树内迭代） | 解释器层嵌套调用 |
| 每层代价 | 无新 sp_head | 一个 `sp_head` 实例 + 一个 `sp_rcontext` |
| 约束 | `cte_max_recursion_depth` | `max_sp_recursion_depth` + 栈深度 |

二者不共享任何代码路径。

---

## 可观测性

- **`SHOW CREATE PROCEDURE`** / `I_S.ROUTINES` / `I_S.TRIGGERS` / `I_S.EVENTS`
- **PFS**：SP 内每条语句各自成事件、各自成 digest（因为主循环里每条指令都换 digest state 并开 PSI 事件）。16 个指令级 instrument（`SP_PSI_STATEMENT_INFO_COUNT 16`），名字形如 `"Stored Program: SET statement"`。详见 [`infra/pfs_statement.md`](../infra/observability/pfs/pfs_statement.md)
- **⚠️ `SHOW PROCEDURE CODE` 在 8.0 不可用**——`print()` 的实现仍在（debug 版），但该语句已被移除
- `max_sp_recursion_depth`（默认 0）、`stored_program_cache`（默认 256）

---

## Misc

### ★ 工程实现技法

1. **指令流 + 解释器**：`sp_instr` 类体系 + `execute(thd, &ip)` 的 `*nextp` 约定，把控制流完全交给指令自己
2. **单遍代码生成 + backpatch**：前向跳转先占位，目标解析后回填
3. **后代解析器**：`CREATE` 的 bison action 先建空 `sp_head`，再往里填指令
4. **变量借用临时表 + `Item_field` 代理**：复用 `Field` 的类型系统与转换逻辑
5. **常驻 MEM_ROOT 与执行期 arena 分离**：`main_mem_root`（跨执行）vs `execute_mem_root`（每指令清空）
6. **RAII**：`set_security_ctx` / `restore_security_context`、`Creation_ctx::set_n_backup` / `restore_env`

### 扩展点：新增一条 sp_instr

需要动：`sp_instr.h`（新类 + `psi_info` 定义）、`sp_head.cc` 的 `add_instr()` 调用点、`SHOW PROCEDURE CODE` 的 `print()`、以及 `SP_PSI_STATEMENT_INFO_COUNT` 上限（当前 16）。

### ★ 性能陷阱

1. **SP 内 SQL 不重新解析，但优化器每次都跑**——所以省掉的是 parse，不是 optimize
2. **含 UDF 的语句每次执行都重新解析**——明确的性能坑
3. **每指令固定开销**：digest/PSI/cleanup/mem_root Clear。一条 SP 里语句越多，占比越高 ⇒ **"把循环写在 SQL 里"通常比"用 SP 循环逐行"快得多**
4. **每次调用新建/销毁 `sp_rcontext`**，还要 `create_tmp_table_from_fields()` 建变量表
5. **缓存超限是整表清空**，会造成一波集中重新解析
6. **短连接场景下缓存永远为空**，每次调用都可能 `db_load_routine()` 重新解析

### 失效与退化场景

- 递归深度超限（`ER_SP_RECURSION_LIMIT` / `ER_SP_NO_RECURSION`）
- 游标未关闭会泄漏 TABLE 资源（虽有 `pop_cursors` 兜底）
- 非确定性函数导致主从不一致
- SP 内临时表在 SBR 下 unsafe

### 易混淆点

| 说法 | 真相 |
|---|---|
| "8.0 还读 `mysql.proc`" | ❌ 已完全 DD 化，无读写路径 |
| "存储函数不能做 DML" | ❌ 普通 DML 可以；禁用的是动态 SQL/结果集/事务控制/RESET/FLUSH |
| "SP 里每条 SQL 每次都重新解析" | ❌ 只解析一次；但优化器每次重跑 |
| "存储程序缓存是 LRU" | ❌ 是"超限整表清空" |
| "SP 递归受 `max_sp_recursion_depth`" | 存储过程是；**函数恒 0 不能递归**；触发器不受此变量约束 |
| "游标读的是基表" | ❌ 物化到临时表（insensitive） |
| "`DECLARE UNDO HANDLER`" | ❌ MySQL 从不支持，只有 CONTINUE/EXIT |

### 遗留 TODO（源码自述）

- `sp_head` 构造：`FIXME: the only use case when name is NULL is events, and it should be rewritten soon.`
- 栈溢出：`TODO: that should be replaced by proper handling of stack overrun error.`（提及 Bug#10100）
- 主循环 binlog：`TODO: consider changing the condition to "not inside event union".`
- **`TODO: we should create sp_rcontext once per command and reuse it on subsequent executions of a trigger/function.`**（两处）
- 权限检查缓存：`@todo Cache if the definer has the rights to use the object on the first usage and reset the cache only if someone does a GRANT statement that 'may' affect this.`
- 子语句表重新校验未实现：`The proper solution for the problem is to re-validate tables of substatements (Bug#12257, Bug#27011, Bug#32868, Bug#33000), but it's not implemented yet.`
- 缓存实现自认未完成：`@attention the current implementation of Stored Program caching is by no mean final`

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 25. Stored Objects*（存储过程/函数/触发器/事件/视图的总章）
- *MySQL 8.0 Reference Manual → 25.7 Stored Program Binary Logging*（复制语义）

**相关文档**
- definer 与 SQL SECURITY：[`auth/definer.md`](auth/definer.md)
- **`DECLARE HANDLER` 的条件匹配与诊断区机制**：[`infra/diagnostics.md`](infra/diagnostics.md)
- 语句级 PFS 监控（SP 内语句如何被埋点）：[`infra/pfs_statement.md`](../infra/observability/pfs/pfs_statement.md)
- 主链（SP 内每条 SQL 最终回到这条链）：[`query/README.md`](query/README.md)
- SBR 的 unsafe 判定：[`replication/binlog_sbr.md`](replication/binlog_sbr.md)
- 外键级联限制：[`../index/constraint.md`](../index/constraint.md)
- 视图（另一类"存储的对象"）：[`view.md`](view.md)

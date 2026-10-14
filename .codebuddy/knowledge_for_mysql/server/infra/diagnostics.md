# 诊断体系：Diagnostics_area 与条件处理

> 本篇讲 MySQL 的**诊断体系**——`Diagnostics_area`（诊断区）、`Sql_condition`（条件）、诊断区栈、条件处理器（`DECLARE HANDLER`）、以及诊断状态如何变成发往客户端的 OK/EOF/ERR 包。
>
> 它是**横切全库的子系统**：存储程序的 `DECLARE HANDLER` 建立在诊断区栈上，PFS 的 `SUM_ERRORS`/`SUM_WARNINGS`/`SUM_ROWS_AFFECTED` 从诊断区取值，`SHOW WARNINGS`、`ROW_COUNT()`、协议层错误包也都以它为唯一数据源。

## 目录

- [一句话机制](#一句话机制)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、主链路：一条语句的诊断区生命周期](#一主链路一条语句的诊断区生命周期)
- [二、诊断区栈：嵌套执行如何不互相污染](#二诊断区栈嵌套执行如何不互相污染)
- [三、条件体系：Sql_condition 与 SQLSTATE](#三条件体系sql_condition-与-sqlstate)
- [四、条件处理器：DECLARE HANDLER](#四条件处理器declare-handler)
- [五、SIGNAL 与 RESIGNAL](#五signal-与-resignal)
- [六、诊断状态如何变成协议包](#六诊断状态如何变成协议包)
- [七、与语句边界、PFS、结果函数的关系](#七与语句边界pfs结果函数的关系)
- [八、系统变量](#八系统变量)
- [九、已知坑与遗留 TODO](#九已知坑与遗留-todo)
- [参考](#参考)

---

## 一句话机制

> 每个 THD 持有**四个** `Diagnostics_area` 对象和一个"当前"指针；语句执行期间，诊断区记录**两类载荷**——① 错误/警告/Note 条件列表，② 语句完成状态（OK/EOF/ERROR + `affected_rows` + `last_insert_id`）。嵌套执行（存储程序、触发器、解析、handler）时靠**压栈新诊断区**隔离，退出时按规则回拷条件。语句结束时，诊断区的状态被**一一映射**成发往客户端的 OK / EOF / ERR 包。

---

## 设计思想与理论基础

### 为什么叫"诊断区"却装着 affected_rows

这是这套设计最容易误解的地方。诊断区对外叫 diagnostics，实质是**"语句完成状态的记录区"**——错误/警告只是其中一类载荷，另一类载荷是 DML 的受影响行数与自增 id。

源码注释直接说这些成员是 THD 旧字段的**替代者**：

```cpp
/**
  The number of rows affected by the last statement. This is
  semantically close to thd->row_count_func, but has a different
  life cycle. ...
  We could possibly merge the two, but life cycle of thd->row_count_func
  can not be changed.
*/
ulonglong m_affected_rows;

/**
  Similarly to the previous member, this is a replacement of
  thd->first_successful_insert_id_in_prev_stmt, which is used
  to implement LAST_INSERT_ID().
*/
ulonglong m_last_insert_id;
```

这也解释了为什么访问器上带着断言：

```cpp
ulonglong affected_rows() const { assert(m_status == DA_OK); return m_affected_rows; }
uint mysql_errno() const { assert(m_status == DA_ERROR); return m_mysql_errno; }
```

**只有 OK 包才带 affected_rows，只有 ERROR 状态才有 mysql_errno**——状态机不允许乱读。

### 五个状态，且不可重复赋值

```cpp
/**
  Stores status of the currently executed statement.
  Cleared at the beginning of the statement, and then
  can hold either OK, ERROR, or EOF status.
  Can not be assigned twice per statement.
*/
class Diagnostics_area {
```

```cpp
enum enum_diagnostics_status {
  /** The area is cleared at start of a statement. */
  DA_EMPTY = 0,
  /** Set whenever one calls my_ok(). */
  DA_OK,
  /** Set whenever one calls my_eof(). */
  DA_EOF,
  /** Set whenever one calls my_error() or my_message(). */
  DA_ERROR,
  /** Set in case of a custom response, such as one from COM_STMT_PREPARE. */
  DA_DISABLED
};
```

"不可重复赋值"由三处保护实现：`set_ok_status()` / `set_eof_status()` 都先判 `if (is_error() || is_disabled()) return;`；`set_error_status()` 在生产构建里也会拒绝覆盖 `DA_DISABLED`。

### 为什么需要诊断区栈

两个独立理由：

1. **解析期不能污染主诊断区**——解析会抛错/警告，但如果是诊断语句（`SHOW WARNINGS`、`GET DIAGNOSTICS`），上一句的条件必须保留下来才能回答问题。
2. **存储程序是"嵌套语句"**——一个 CALL 里每条 SQL 都有自己的 OK/EOF/ERROR，但只有最外层的完成状态能发给客户端；内层的条件必须按 SQL 标准**向上冒泡**而不覆盖外层。

栈 + `copy_conditions`（push 时把下层条件拷到新栈顶）就是这套语义的载体。

### 设计权衡：OOM 时也要能记录

诊断区把 `Sql_condition` 的基本属性**内联**进自己体内，注释解释了动机：

```cpp
/*
  This section contains basic attributes of Sql_condition to store
  information about error (SQL-condition of error severity) or OK-message.
  The attributes are inlined here (instead of using Sql_condition) to be able
  to store the information in case of out-of-memory error.
*/
```

即：**内存耗尽时无法再 new 一个 Sql_condition，但错误消息仍要能存下来**。这是"诊断必须永远可用"的硬约束。

---

## 一、主链路：一条语句的诊断区生命周期

### 1.1 THD 上有四个诊断区

```cpp
MEM_ROOT main_mem_root;
Diagnostics_area main_da;
Diagnostics_area m_parser_da;              /**< cf. get_parser_da() */
Diagnostics_area m_query_rewrite_plugin_da;
Diagnostics_area *m_query_rewrite_plugin_da_ptr;
Diagnostics_area *m_stmt_da;
```

| 对象 | 用途 | 注释 |
|---|---|---|
| `main_da` | 主诊断区（语句执行） | — |
| `m_parser_da` | 解析专用 | *"We need to have a clean DA in case errors or warnings are thrown during parsing, but we can't just reset the main DA in case we have a diagnostic statement on our hand that needs the old DA to answer questions about the previous execution. Keeping a static per-thread DA for parsing is less costly than allocating a temporary one for each statement we parse."* |
| `m_query_rewrite_plugin_da` | 查询重写插件专用 | *"we don't want conditions raised by plugins in either statement nor parser DA until we know which type of statement we have parsed"* |
| `m_stmt_da` | **当前指针** | `get_stmt_da()` |

⚠️ 切换当前诊断区**只有一条合法路径**：`THD::push_diagnostics_area()` / `pop_diagnostics_area()`。`Diagnostics_area::push_diagnostics_area()` 是 private 且不改 `m_stmt_da`（注释显式提醒），THD 版才是完整语义（DA 版 + 更新指针）。

### 1.2 生命周期

```
命令边界   reset_for_next_command()
             ├─ clear_error()
             ├─ reset_diagnostics_area()      ← 清状态与标量结果
             └─ reset_statement_cond_count()
           ↓
解析       push_diagnostics_area(parser_da, copy_conditions=false)
             ↓ 解析错误/警告落在 parser_da
           pop_diagnostics_area()
             ↓ 按结果设 lex->keep_diagnostics（见 7.1）
语句开始   reset_diagnostics_area()
           reset_condition_info()   ← 清条件列表（诊断语句除外）
           ↓
执行       条件产生 → push_warning / raise_condition
           完成   → my_ok() / my_eof() → set_ok_status / set_eof_status
           出错   → my_error() → raise_condition → set_error_status
           ↓
发送       send_statement_status()  ← DA 状态 → OK/EOF/ERR 包（见六）
           ↓
PFS       MYSQL_END_STATEMENT(..., thd->get_stmt_da())  ← 诊断区指针直接传给 PFS
```

---

## 二、诊断区栈：嵌套执行如何不互相污染

### 2.1 栈是侵入式链表，不是数组

⚠️ 诊断区里**没有** `m_stack` 成员，只有一个指针：

```cpp
/** Pointer to the Diagnostics Area below on the stack. */
Diagnostics_area *m_stacked_da;
```

即：栈通过"每个 DA 记住它下面那个 DA"串成**侵入式链表**，栈顶就是 `THD::m_stmt_da`。`get_stacked_da()` 只能看到"第二诊断区"一层。

```cpp
void Diagnostics_area::push_diagnostics_area(THD *thd, Diagnostics_area *da,
                                             bool copy_conditions) {
  assert(da->m_stacked_da == nullptr);
  da->m_stacked_da = this;
  if (copy_conditions) {
    da->copy_sql_conditions_from_da(thd, this);
    da->m_saved_warn_count = m_saved_warn_count;
    da->m_saved_error_count = m_saved_error_count;
  }
}

Diagnostics_area *Diagnostics_area::pop_diagnostics_area() {
  assert(m_stacked_da);
  Diagnostics_area *da = m_stacked_da;
  m_stacked_da = nullptr;
  return da;
}
```

`copy_conditions` 的语义注释引用了 SQL 标准：

```
  @param copy_conditions
              Copy the conditions from the new second Diagnostics Area
              to the new first Diagnostics Area, as per SQL standard.
```

⚠️ 注意 `pop_diagnostics_area()` **只摘链，不回拷条件**——回拷是调用方的责任。

### 2.2 压栈的时机

| 场景 | copy_conditions |
|---|---|
| **解析阶段**（每个命令） | `false` |
| query rewrite 插件 | `false` |
| **存储程序整体执行** | 默认 `true` |
| **handler 激活** | 默认 `true` |
| `SHOW WARNINGS` / `GET DIAGNOSTICS` | 各自新建 DA |
| prepared statement 执行 | 默认 `true` |
| **临时屏蔽错误**（打开损坏表、`SHOW CREATE TABLE` 收集警告等） | `false` |

### 2.3 四个语义相近但不同的清理/切换操作

| 操作 | 清什么 | 典型场景 |
|---|---|---|
| `set_ok_status(affected, id, msg)` | 置 `DA_OK` + 写 affected/last_insert_id + 快照条件数 | DML/DDL 成功结束（`my_ok()`） |
| `reset_condition_info()` | **只清条件，不动状态** | 语句开始时清警告；handler 退出前 |
| `reset_diagnostics_area()` | **只清状态与标量结果，不清条件列表** | 语句边界、`THD::clear_error()` |
| `pop_diagnostics_area()` | 只摘链（不回拷） | handler 退出、解析结束 |

`reset_condition_info()` 里那段注释解释了"为什么 `SELECT @@warning_count` 要特殊处理"：

```
Special case: @@session.error_count, @@session.warning_count
These appear in non-diagnostics statements (SELECT ... [INTO ...], etc.),
so we must clear the DA rather than keep it.  To keep these legacy
system variables working, we save the counts before clearing the
(rest of the) DA.  The system variables have special getters that access
the saved values where applicable.
```

### 2.4 三个"部分回拷"的姊妹函数

决定条件如何在栈间流动：

| 函数 | 语义 |
|---|---|
| `copy_sql_conditions_from_da()` | 拷全部（push 时按 SQL 标准用） |
| `copy_non_errors_from_da()` | `if (cond->severity() == SL_ERROR) continue;` 只拷非错误（"临时屏蔽错误"场景） |
| `copy_new_sql_conditions()` + `mark_preexisting_sql_conditions()` | 只拷 **handler 激活之后新增**的条件（handler 退出时用） |

---

## 三、条件体系：Sql_condition 与 SQLSTATE

### 3.1 Sql_condition

```cpp
/**
  Representation of a SQL condition.
  A SQL condition can be a completion condition (note, warning),
  or an exception condition (error, not found).
*/
class Sql_condition {
 public:
  enum enum_severity_level { SL_NOTE, SL_WARNING, SL_ERROR, SEVERITY_END };
```

接口几乎全私有，注释解释了设计意图：

```cpp
/*
  The interface of Sql_condition is mostly private, by design,
  so that only the following code:
  - various raise_error() or raise_warning() methods in class THD,
  - the implementation of SIGNAL / RESIGNAL / GET DIAGNOSTICS
  - catch / re-throw of SQL conditions in stored procedures (sp_rcontext)
  is allowed to create / modify a SQL condition.
  ...
  the only public interface available to the rest of the server implementation
  is the interface offered by the THD methods (THD::raise_error()),
  which should be used.
*/
```

### 3.2 SQLSTATE 是数据驱动的，不硬编码在 C++

运行时判定只有四个 inline 谓词（按前两位字符分类）：

- completion: `s[0]=='0' && s[1]=='0'`
- warning: `s[0]=='0' && s[1]=='1'`
- not found: `s[0]=='0' && s[1]=='2'`
- exception: 其余（含 `42S22` / `23000` / `HY000`）

**具体的 5 字符值来自 `share/messages_to_clients.txt`**，每条错误后面跟一个 SQLSTATE：

```
ER_BAD_FIELD_ERROR 42S22 S0022
        eng "Unknown column '%-.192s' in '%-.192s'"
```

运行时由生成的 `error_names_array` 查表：`mysql_errno_to_sqlstate()`。

`CLASS_ORIGIN` / `SUBCLASS_ORIGIN` 由 SQLSTATE 前两位反推，规则写在注释里：

```
If CLASS[1] is any of: 0 1 2 3 4 A B C D E F G H
and CLASS[2] is any of: 0-9 A-Z,
then let CLASS_ORIGIN = 'ISO 9075'. Otherwise 'MySQL'.
...
If CLASS_ORIGIN = 'ISO 9075' or SUBCLASS = '000',
then let SUBCLASS_ORIGIN = 'ISO 9075'. Otherwise 'MySQL'.
```

### 3.3 条件列表与 max_error_count

```cpp
Sql_condition *Diagnostics_area::push_warning(THD *thd, uint mysql_errno,
                                              const char *returned_sqlstate,
                                              Sql_condition::enum_severity_level severity,
                                              const char *message_text) {
  Sql_condition *cond = nullptr;
  if (m_allow_unlimited_conditions ||
      m_conditions_list.elements() < thd->variables.max_error_count) {
    cond = new (&m_condition_root) Sql_condition(...);
    if (cond) m_conditions_list.push_back(cond);
  }
  m_current_statement_cond_count_by_qb[(uint)severity]++;
  m_current_statement_cond_count++;
  return cond;
}
```

条件列表本身是 `I_P_List<Sql_condition, ..., I_P_List_counter, I_P_List_fast_push_back<...>>`——**计数 + O(1) 尾插 + 双向链表**。

### 3.4 SHOW WARNINGS / @@warning_count 的数据来源

`SHOW WARNINGS` 直接迭代 `first_da->sql_conditions()`，按级别位图过滤 + 应用 LIMIT：

```cpp
Diagnostics_area::Sql_condition_iterator it = first_da->sql_conditions();
while (!rc && (err = it++)) {
  /* Skip levels that the user is not interested in */
  if (!(levels_to_show & ((ulong)1 << err->severity()))) continue;
  if (++idx <= unit->offset_limit_cnt) continue;
  if (idx > unit->select_limit_cnt) break;
  ...store Level / Code / Message...
}
```

⚠️ **`@@warning_count` 包含 error 和 note**（三者相加），这是 MySQL 的历史语义：

```cpp
ulong Diagnostics_area::warn_count(THD *thd) const {
  ...
  return ...SL_NOTE] + ...SL_ERROR] + ...SL_WARNING];
}
```

级别名：`const LEX_CSTRING warning_level_names[] = {"Note"}, {"Warning"}, {"Error"}, {"?"}`。

---

## 四、条件处理器：DECLARE HANDLER

### 4.1 三个类的分工

| 类 | 角色 |
|---|---|
| `sp_handler` | 解析期对象：类型 + 作用域 + 捕获的条件值列表 |
| `sp_rcontext::sp_handler_entry` | 运行时"可见 handler"栈项 |
| `sp_rcontext::Handler_call_frame` | 一次 handler 调用帧（含自己的 `Diagnostics_area handler_da`） |

```cpp
/// This class represents a call frame of SQL-handler (one invocation of a
/// handler). Basically, it's needed to store continue instruction pointer for
/// CONTINUE SQL-handlers.
const sp_handler *handler;
Sql_condition *sql_condition;   ///< 触发这次激活的条件
uint continue_ip;               ///< "The attribute contains 0 for EXIT-handlers."
Diagnostics_area handler_da;    ///< "The Diagnostics Area which will be pushed when the handler activates
                                ///   and popped when the handler completes."
```

### 4.2 ★ UNDO handler 确认不支持

三重证据：

```cpp
/// Enumeration of possible handler types.
/// Note: UNDO handlers are not (and have never been) supported.
enum enum_type { EXIT, CONTINUE };
```

语法规则里 UNDO 被注释掉：`/*| UNDO_SYM      { QQ No yet } */`，且 `UNDO_SYM` 标为 `/* FUTURE-USE */`。

（`sp_instr_hreturn` 注释里出现 "EXIT or UNDO handler" 是历史措辞。）

### 4.3 CONTINUE vs EXIT：分叉点只在"返回地址"

```cpp
bool sp_instr_hreturn::execute(THD *thd, uint *nextp) {
  /*
    Obtain next instruction pointer (m_dest is set for EXIT handlers, retrieve
    the instruction pointer from runtime context for CONTINUE handlers).
  */
  *nextp = m_dest ? m_dest : rctx->get_last_handler_continue_ip();
```

| 类型 | 返回地址 | 效果 |
|---|---|---|
| **CONTINUE** | `cur_spi->get_cont_dest()`（触发条件那条指令的下一条） | 回到出错语句**之后**继续 |
| **EXIT** | 编译期写死的 `m_dest`（handler 所在块的出口） | **跳出块** |

### 4.4 handle_sql_condition() 流程

1. **fatal 子语句错误直接放弃本层**：`if (thd->is_fatal_sub_stmt_error && m_in_sub_stmt) return false;`——注释说要去外层找 handler
2. **错误分支**：`find_handler(sqlstate, errno, SL_ERROR)`；拿不到条件就现造一个临时条件（注释解释了两种成因：诊断区满 / 直接用了 `set_error_status(uint)`）
3. **警告/Note 分支**：遍历条件列表找**最后一个** WARNING/NOTE（注释：*"In MySQL most substantial warning is the last one."*），并**忽略 preexisting 条件**——理由是避免"在我们正在调用的 handler 里再声明 handler 时，catcher 变得无法预测"
4. **装帧 + 清错 + 压栈**：

```cpp
uint continue_ip = handler_entry->handler->type == sp_handler::CONTINUE
                       ? cur_spi->get_cont_dest() : 0;
...
thd->clear_error();
thd->killed = THD::NOT_KILLED;  // Some errors set thd->killed (e.g. "bad data").

thd->push_diagnostics_area(&frame->handler_da);
frame->handler_da.mark_preexisting_sql_conditions();
frame->handler_da.reset_statement_cond_count();
*ip = handler_entry->first_ip;
```

5. **退出**：`exit_handler()` → `pop_handler_frame()`：

```cpp
void sp_rcontext::pop_handler_frame(THD *thd) {
  // Also pop matching DA and copy new conditions.
  assert(thd->get_stmt_da() == &frame->handler_da);
  thd->pop_diagnostics_area();
  // Out with the old, in with the new!
  thd->get_stmt_da()->reset_condition_info(thd);
  thd->get_stmt_da()->copy_new_sql_conditions(thd, &frame->handler_da);
  delete frame;
}
```

语义注释：

```
The handler has successfully completed. ...
The diagnostics area of the caller should then contain only any
conditions pushed during handler execution. The conditions present
when the handler was activated should be removed since they have
been successfully handled.
```

最后还要防重复触发：`/* Condition was successfully handled, reset condition count so we don't trigger the handler again for the same warning. */`

> ⚠️ 一个关键细节：handler 的"捕获"发生在**指令边界**（`sp_instr_stmt` 执行完之后检查），不是在 raise 的瞬间。

### 4.5 匹配规则：越具体越优先

条件值类型：`enum enum_type { ERROR_CODE, SQLSTATE, WARNING, NOT_FOUND, EXCEPTION };`

优先级由 `(!found_cv || found_cv->type > X)` 体现——`ERROR_CODE`(=0) 优先于 `SQLSTATE`(=1)；`WARNING/NOT_FOUND/EXCEPTION` 只在还没有更具体命中时才生效。

| 声明 | 判定 |
|---|---|
| `FOR 1051` | `sql_errno == cv->mysqlerr` |
| `FOR SQLSTATE '42S22'` | `strcmp(sql_state, cv->sql_state) == 0` |
| `FOR SQLWARNING` | `is_sqlstate_warning(sql_state) \|\| severity == SL_WARNING` |
| `FOR NOT FOUND` | `is_sqlstate_not_found(sql_state)` |
| `FOR SQLEXCEPTION` | `is_sqlstate_exception(sql_state) && severity == SL_ERROR` |
| `FOR <条件名>` | 编译期已消解成上述五种之一 |

**作用域递归规则**（注释原文）：

```
// 2. The current context has HANDLER_SCOPE. That means, we're inside an
//    SQL-handler block ... parent's handler don't catch conditions from this
//    scope. Instead, we should try to find first parent context (we might
//    have nested handler declarations), which has REGULAR_SCOPE ...
while (p && p->m_scope == HANDLER_SCOPE) p = p->m_parent;
```

即：**handler 自己块里抛出的条件不会被同块/父 handler 递归捕获**，必须跳到最近的 REGULAR_SCOPE 再向父级找。

还有个容易踩的时序坑（源码给的例子）：

```
CREATE PROCEDURE p()
BEGIN
  DECLARE v INT DEFAULT 'get'; -- raises SQL-warning here
  DECLARE EXIT HANDLER ...     -- this handler does not catch the warning
END
```

---

## 五、SIGNAL 与 RESIGNAL

### 5.1 SIGNAL：先清空再抛

按 WL#2110 语义：

```
WL#2110 SIGNAL specification says:
  When SIGNAL is executed, it has five effects, in the following order:
    (1) First, the Diagnostics Area is completely cleared. So if the
    SIGNAL is in a DECLARE HANDLER then any pending errors or warnings
    are gone. So is 'row count'.
```

严重级别由 SQLSTATE 类推导：`01`→`SL_WARNING`/`ER_SIGNAL_WARN`，`02`→`SL_ERROR`/`ER_SIGNAL_NOT_FOUND`，其它→`SL_ERROR`/`ER_SIGNAL_EXCEPTION`。（`SIGNAL should not signal SL_NOTE`。）

### 5.2 RESIGNAL：在 handler 内重新抛出

必须在 handler 内（否则 `ER_RESIGNAL_WITHOUT_ACTIVE_HANDLER`）。流程：先 `pop_diagnostics_area()` 回到外层 → 可选地用 `signal_value` 造副本（`reserve_number_of_conditions(thd, 1)` 腾位）→ `raise_condition()` → **把 handler DA 重新压回去**。

为什么 pop+push 而不是原地改，注释给了四种情形并解释：

```
We now have the following possibilities:
1) We RESIGNAL a warning
2) We RESIGNAL an error for which there is another CONTINUE handler.
3) We RESIGNAL an error for which there is another EXIT handler.
4) We RESIGNAL an error for which there is no appropriate handler.
... By doing pop+push rather than nothing, we make sure that the condition
modified by RESIGNAL (if any) is modified on both the first and the second
Diagnostics Area (not just in the first).
```

一句话：**RESIGNAL = 把当前被捕获的条件（可修改/可替换）重新抛给外层作用域**，使其能被外层 handler 或客户端看到。

---

## 六、诊断状态如何变成协议包

```cpp
void THD::send_statement_status() {
  assert(!get_stmt_da()->is_sent());
  Diagnostics_area *da = get_stmt_da();
  switch (da->status()) {
    case Diagnostics_area::DA_ERROR:
      error = m_protocol->send_error(da->mysql_errno(), da->message_text(),
                                     da->returned_sqlstate());
      break;
    case Diagnostics_area::DA_EOF:
      error = m_protocol->send_eof(server_status, da->last_statement_cond_count());
      break;
    case Diagnostics_area::DA_OK:
      error = m_protocol->send_ok(server_status, da->last_statement_cond_count(),
                                  da->affected_rows(), da->last_insert_id(),
                                  da->message_text());
      break;
    case Diagnostics_area::DA_DISABLED:
      break;
    case Diagnostics_area::DA_EMPTY:
    default:
      assert(0);
      error = m_protocol->send_ok(server_status, 0, 0, 0, nullptr);
      break;
  }
  if (!error) da->set_is_sent(true);
}
```

| 诊断区状态 | 协议包 | 携带 |
|---|---|---|
| `DA_OK` | **OK** | affected_rows、last_insert_id、warning 计数 |
| `DA_EOF` | **EOF** | warning 计数 |
| `DA_ERROR` | **ERR** | errno、message、sqlstate |
| `DA_DISABLED` | 什么都不发 | — |
| `DA_EMPTY` | `assert(0)` | 说明语句既没 OK 也没 ERROR（不该发生） |

⚠️ 底层 `net_send_error()` 的设计注释明确它的适用面很窄：

```
Design note:
net_printf_error and net_send_error are low-level functions
that shall be used only when a new connection is being
established or at server startup.
```

主链路一律走 `send_statement_status()`。

### DA_DISABLED 什么时候用

```cpp
/**
  Mark the Diagnostics Area as 'DISABLED'.
  This is used in rare cases when the COM_ command at hand sends a response
  in a custom format. One example is COM_STMT_PREPARE.
*/
```

实际调用点：`COM_STMT_PREPARE`、`COM_STMT_SEND_LONG_DATA`、`COM_STMT_CLOSE`、`COM_QUIT`（*"Don't send anything back"*）、认证握手完成、统计类自定义响应。

注意它**不是** `SHOW WARNINGS` 用的机制——`SHOW WARNINGS` 用的是"压一个新 DA"。

---

## 七、与语句边界、PFS、结果函数的关系

### 7.1 语句开始会清警告，但诊断语句例外

```cpp
thd->get_stmt_da()->reset_diagnostics_area();
if ((thd->lex->keep_diagnostics != DA_KEEP_PARSE_ERROR) &&
    (thd->lex->keep_diagnostics != DA_KEEP_DIAGNOSTICS)) {
  /*
    No parse errors, and it's not a diagnostic statement:
    remove the sql conditions from the DA!
    For diagnostic statements we need to keep the conditions
    around so we can inspec them.
  */
  thd->get_stmt_da()->reset_condition_info(thd);
}
```

由一个 LEX 提示位控制：

```cpp
enum enum_keep_diagnostics {
  DA_KEEP_NOTHING = 0,   /**< keep nothing */
  DA_KEEP_DIAGNOSTICS,   /**< keep the diagnostics area */
  DA_KEEP_COUNTS,        /**< keep @warning_count / @error_count */
  DA_KEEP_PARSE_ERROR,   /**< keep diagnostics area after parse error */
  DA_KEEP_UNSPECIFIED    /**< keep semantics is unspecified */
};
```

★ **这正是 `SHOW WARNINGS;` 能看到上一条 `INSERT` 的警告、而 `SELECT 1;` 会清空它们的机制来源。**

### 7.2 ★ ROW_COUNT() / LAST_INSERT_ID() 读的是 THD 字段，不是诊断区

⚠️ 一个容易搞错的点：**三者都不直接读诊断区**，读的是 THD 上的独立字段（与诊断区是"双写/同源"关系）：

| 函数 | 读什么 |
|---|---|
| `ROW_COUNT()` | `thd->get_row_count_func()`（`m_row_count_func`） |
| `FOUND_ROWS()` | `thd->found_rows()` → `previous_found_rows` |
| `LAST_INSERT_ID()` | `thd->first_successful_insert_id_in_prev_stmt` |

而诊断区的 `m_affected_rows` / `m_last_insert_id` 是**发给客户端 OK 包的那一份**（注释说它们是 THD 旧字段的 replacement，`my_ok()` 里双写）。

`ROW_COUNT()` 的取值规则是一份完整注释：

```
ROW_COUNT is assigned according to the following rules:
  - In my_ok():
    - for DML statements: to the number of affected rows;
    - for DDL statements: to 0.
  - In my_eof(): to -1 to indicate that there was a result set.
    We derive this semantics from the JDBC specification ...
  - In my_error(): to -1 to be compatible with the MySQL C API and
    MySQL ODBC driver.
  - For SIGNAL statements: to 0 per WL#2110 specification ...
```

### 7.3 PFS：诊断区指针被直接传进去

```cpp
MYSQL_END_STATEMENT(thd->m_statement_psi, thd->get_stmt_da());
```

`pfs_end_statement_vc()` 里 `reinterpret_cast<Diagnostics_area *>(stmt_da)`，按状态取值：

```cpp
case Diagnostics_area::DA_OK:
  ... pfs->m_rows_affected = da->affected_rows();
      pfs->m_warning_count = da->last_statement_cond_count();
      memcpy(pfs->m_sqlstate, "00000", SQLSTATE_LENGTH);   break;
case Diagnostics_area::DA_EOF:
  pfs->m_warning_count = da->last_statement_cond_count();  break;
case Diagnostics_area::DA_ERROR:
  ... pfs->m_sql_errno = da->mysql_errno();
      memcpy(pfs->m_sqlstate, da->returned_sqlstate(), SQLSTATE_LENGTH);
      pfs->m_error_count++;                                break;
```

三个要点：

- PFS 的 warning_count 来自 **`last_statement_cond_count()`（语句完成时的快照）**，不是当前列表长度
- `DA_EOF` 只贡献 warning 计数，**不贡献 rows_affected**
- `DA_OK` 时 sqlstate 固定 `"00000"`

（详见 [`pfs_statement.md`](../../infra/observability/pfs/pfs_statement.md)。）

### 7.4 事务回滚不碰诊断区

⚠️ **源码无证据**表明事务回滚会主动操纵诊断区。合理结论：回滚路径若引擎报错，走的是普通的 `my_error()` → 置 `DA_ERROR`；**回滚本身不重置诊断区**。清理只发生在语句/命令边界和显式调用点。

---

## 八、系统变量

| 变量 | 默认 | 作用 |
|---|---|---|
| `max_error_count` | 1024 | 单条语句最多存多少条件（note/warning/error **共享额度**） |

范围 `0..65535`。设为 0 则一条都不存（但计数照涨）。

---

## 九、已知坑与遗留 TODO

### 9.1 ★ max_error_count 满了：静默丢弃，但计数器照涨

`push_warning()` 在列表满时**返回 nullptr 且不插入**，但 `m_current_statement_cond_count++` 在 `if` 外面。后果由注释点明：

```
This may be higher than warn_list.elements() if we have
had more warnings than thd->variables.max_error_count.
```

即 **`@@warning_count` 会大于 `SHOW WARNINGS` 实际返回的行数**——"计数正确、内容丢失"的取舍。

连带影响：

- `handle_sql_condition()` 里 `da->error_condition()` 可能为 NULL（*"if the Diagnostics Area was full when the error was raised"*），此时要现造临时条件
- RESIGNAL 会主动 `reserve_number_of_conditions(thd, 1)` 丢**最老的**来腾位
- 想"不丢"的场景用 `Diagnostics_area tmp_da(true)`（`allow_unlimited_conditions=true`），如 `SHOW CREATE TABLE` 收集警告

### 9.2 诊断区切换的真实代价

⚠️ **不是"存储程序每条语句 push/pop"**。实际是：

- 每次**存储程序调用** push 一次（整个 routine 体共用）
- 每次 **handler 激活** push 一次
- 每次 **解析** push 一次（用静态成员避免每次分配）

真正的代价在 **push 时的条件拷贝**：`copy_conditions` **默认 true**，会把下层全部条件**逐条 new + 拷贝**到新栈顶。所以在"循环里反复报错并被 CONTINUE handler 捕获"的存储程序里，这是 **O(循环次数 × 条件数)** 的拷贝开销。

### 9.3 遗留 TODO（源码自述）

| 位置 | 内容 |
|---|---|
| `clear_error()` | `@todo: To silence an error, one should use Internal_error_handler mechanism. In future this function will be removed.` |
| `my_ok()` 上方 | 光秃秃的 `// TODO` |
| 诊断区"Design notes" Future work | `Change (#1) (my_error) to generate errors in UTF8. See WL#751 (Recoding of error messages)`；`Change (#4 and #5) to store message text in UTF8 natively` |
| `my_message_sql()` | `TODO: assert(error != 0);`（配注释：*"Calling my_error() with error == 0 is a bug."*） |
| `reset_for_next_command()` | `// TODO: Why on earth is this here?! ... /Matz` |

字符集现状：`my_error()` 用 `error_message_charset_info`，而 `SIGNAL`/`RESIGNAL` 内部用 UTF8——两套字符集在 `eval_signal_informations()` 里转换。

### 9.4 版本演进的一处化石

`THD::is_error()` 的文档注释至今写着：

```
Please use this method instead of direct access to net.report_error.
```

说明历史上错误状态确实存放在 `THD::net`（`NET::last_error/last_errno/report_error`）上，后来才统一收敛到诊断区——这条迁移**至今未完成**（`m_row_count_func` 与 `m_affected_rows` 两套生命周期并存，注释明说 `life cycle of thd->row_count_func can not be changed`）。

> ⚠️ "5.x → 8.0"的提交级演进链**源码无直接证据**，上述是间接证据（化石注释 + 成员注释自称"替代品" + WL#2110/WL#751 工作项编号）。

---

## 参考

**源码**
- `sql/sql_error.h` / `sql_error.cc` —— `Diagnostics_area`、`Sql_condition`
- `sql/sql_class.h` —— THD 上的四个诊断区、`raise_condition` / `send_statement_status`
- `sql/sp_pcontext.{h,cc}` / `sql/sp_rcontext.{h,cc}` —— handler 声明、匹配、激活与退出
- `sql/sp_instr.cc` —— `hpush_jump` / `hpop` / `hreturn` 三条指令
- `sql/sql_signal.cc` —— SIGNAL / RESIGNAL
- `share/messages_to_clients.txt` —— 错误码与 SQLSTATE 的数据源（构建期由 `comp_err` 生成 `mysqld_error.h` / `errmsg.sys`，见 [`build.md`](build.md)）

**相关文档**
- PFS 如何消费诊断区（ERRORS/WARNINGS/ROWS_AFFECTED）：[`pfs_statement.md`](../../infra/observability/pfs/pfs_statement.md)
- 存储程序执行（`DECLARE HANDLER` 的运行宿主、指令循环）：[`../stored_program.md`](../stored_program.md)
- 协议层结果发送：`../query/01_protocol_to_dispatch.md`

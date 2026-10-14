# 23 optimizer_trace 的实现原理：一份零成本的栈式 JSON 生成器

> 基于 MySQL 8.0.39。本篇剖析 optimizer_trace **基础设施本身**是怎么工作的：`Opt_trace_context`（会话级上下文）与 `Opt_trace_struct`/`Opt_trace_object`/`Opt_trace_array`（RAII 栈）如何协作，让优化器深处的埋点不需要层层传参；`is_started()` 如何做到"未开启时零成本"；trace 如何带 `feature` 分类、如何进 `information_schema.optimizer_trace`，以及那个 SQL SECURITY DEFINER 安全洞是怎么堵的。
>
> **边界**：本篇讲"trace 机制怎么实现"。**怎么用 trace 观察优化器**（读哪些节点、排查哪些问题）见 [`../11_explain_and_trace.md`](../../../infra/observability/explain_trace.md)。DBUG 通道见 [`../../infra/dbug.md`](../../infra/dbug.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、三个类的分工](#一三个类的分工)
  - [二、为什么 context 要持"当前结构"指针](#二为什么-context-要持当前结构指针)
  - [三、Opt_trace_struct：RAII 栈 + is_started 零成本](#三opt_trace_structraii-栈--is_started-零成本)
  - [四、add 系列重载与 feature 机制](#四add-系列重载与-feature-机制)
  - [五、start/end 生命周期与 OFFSET/LIMIT](#五startend-生命周期与-offsetlimit)
  - [六、进 information_schema：填充与 SUID 安全洞](#六进-information_schema填充与-suid-安全洞)
  - [七、JSON 生成：end_marker 与 one_line](#七json-生成end_marker-与-one_line)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

optimizer_trace 是 MySQL 优化器自带的**结构化执行记录器**。你在 [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)、[`17_optimizer_decisions.md`](17_optimizer_decisions.md) 里看到的 `optimizing_distinct_group_by_order_by`、`semijoin_strategy_choice`、`rows_estimation` 这些节点，都是优化器代码里一句句 `Opt_trace_object trace(...)` 埋点写出来的 JSON。

它由四层组成：

| 层 | 文件 | 角色 |
|---|---|---|
| `Opt_trace_context` | `sql/opt_trace_context.h` | 会话级上下文，挂在 `THD::opt_trace` 上，**任何时刻都可用** |
| `Opt_trace_struct` / `Opt_trace_object` / `Opt_trace_array` | `sql/opt_trace.h` | RAII 栈元素，代表"正在写的一个对象/数组" |
| 实现 | `sql/opt_trace.cc` | 栈操作、内存、JSON 序列化 |
| I_S 出口 | `sql/opt_trace2server.cc` | 把 trace 填进 `information_schema.optimizer_trace` |

### 一个反直觉的开场

优化器源码里到处是：

```cpp
Opt_trace_object trace_derived(trace, "derived");
trace_derived.add("table", ...).add("select#", ...);
```

**但这些 `Opt_trace_object` 的构造，在没有开启 trace 时几乎什么也不做**——因为构造函数第一行就是：

```cpp
if (unlikely(ctx_arg->is_started())) { ... }
```

`is_started()` 是两个指针判空（见第三章），且被 `unlikely()` 提示分支预测器"这基本是 false"。这就是"optimizer_trace 关闭时零开销"的源码级保证——生产环境默认关闭 trace，这些埋点不会拖慢优化器。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.6 | optimizer_trace 引入（`opt_trace.h/cc`、`opt_trace2server.cc` 首次出现） |
| 5.7 | `optimizer_trace_features` 变量引入（按 feature 过滤，见第四章） |
| 8.0 | 迭代器化后 trace 埋点扩散到 `join_optimizer/` 与执行器 |

---

## 理论基础

### 栈式 JSON 生成：为什么用"当前结构指针"而不是传参

这是 optimizer_trace 最核心的设计决策，源码注释用了一整段来解释。核心矛盾是：

**trace 埋点常常在很深的调用栈里，中间隔着很多不写 trace 的帧。**

注释给的例子（gdb `backtrace` 记法，内层帧在前）：

```
#0  Item_in_subselect::single_value_transformer   ← 打开 "transformation" 对象
#1  Item_in_subselect::select_in_like_transformer ← 不写 trace
#2  Item_allany_subselect::select_transformer     ← 不写 trace
#3  Query_block::prepare                          ← 打开 "join_preparation" 对象
```

`#3` 打开的 `join_preparation` 对象，理论上要穿过 `#2`、`#1` 两个**不写 trace** 的帧，才能到 `#0` 让 `transformation` 对象挂上去。如果靠函数参数传这个对象，就得**给 `#1`、`#2` 这些本来不关心 trace 的函数都加一个参数**，改一堆函数原型——这在工程上不可接受。

所以 optimizer_trace 的解法是：**context 维护一个"当前打开的 object/array"指针，深处的帧直接 grab 它**。

```
context.current_struct  ← 始终指向"当前正在写的最内层结构"
  ├─ 某函数构造 Opt_trace_object("join_preparation")
  │     → 保存 current_struct 为 parent，把自己设为 current_struct
  ├─ 中间不写 trace 的帧什么都不做（current_struct 保持指向 join_preparation）
  ├─ 深处构造 Opt_trace_object("transformation")
  │     → 同样入栈，parent = join_preparation
  │     → 写 key/value
  │     → 析构，current_struct 恢复为 join_preparation
  └─ "join_preparation" 析构，current_struct 恢复为更外层的结构
```

这就是一个**用 RAII 实现的栈**。这个设计把"传参"变成了"隐式的上下文"，代价是：

- 依赖**作用域正确性**（对象的析构顺序 = 嵌套顺序，C++ 保证）；
- 埋点必须**成对出现**（有 `Opt_trace_object` 就要让它析构，否则栈会乱）。

### 为什么能用栈，而不用树

JSON 本身是树，但**生成 JSON 的过程天然是深度优先的**：你打开一个对象、写完它的内容、再关闭它，这个过程就是 DFS。DFS 的当前路径正好可以用一个栈表示（"当前打开的结构"就是栈顶）。所以不需要显式建树，栈顶指针就够用。

### 他库对比

| 库 | 类似机制 |
|---|---|
| **MySQL** | optimizer_trace（栈式 JSON + feature 过滤） |
| **PostgreSQL** | `EXPLAIN (VERBOSE, FORMAT JSON)` + `debug_print_plan` / `debug_print_rewritten`（不是常驻的 trace，是显式打印） |
| **Oracle** | `DBMS_XPLAN` + `10053` 事件（CBO trace，纯文本而非 JSON） |
| **SQL Server** | 执行计划 XML + 缺少计划缓存的诊断 |

MySQL 的独特点在于：**它是一个常驻的、结构化的、可查询（I_S）的 trace**，而 Oracle 的 10053 是文本 dump、PG 的 debug_print 是临时打印。

---

## 核心实现

### 一、三个类的分工

| 类 | 关系 | 职责 |
|---|---|---|
| `Opt_trace_context` | 值成员在 `THD` 里（`Opt_trace_context opt_trace;`） | 会话状态：enabled、当前 trace、历史 trace 列表、feature、I_S 支持 |
| `Opt_trace_struct` | 基类 | 一个"结构"（对象或数组）的 RAII 句柄，提供 `add()` 系列 |
| `Opt_trace_object` | 继承 `Opt_trace_struct` | 对象：`{ "key": {...} }`，**必须带 key** |
| `Opt_trace_array` | 继承 `Opt_trace_struct` | 数组：`"key": [...]`，元素**不带 key** |

`Opt_trace_object` 与 `Opt_trace_array` 的区别只有 `requires_key`：object 为 true（`{key: value}`），array 为 false（`[value, value]`）。这个区别通过虚函数表达，头注释还专门解释了为什么不用模板——因为 `Opt_trace_struct::do_construct()` 里要能拿到它（跨层访问）。

### 二、为什么 context 要持"当前结构"指针

见「理论基础」的 backtrace 例子。补充源码里的两个直接证据：

1. `Opt_trace_context::get_current_stmt_in_gen()` —— 让 `Opt_trace_struct` 知道"自己属于哪条语句的 trace"。
2. `Opt_trace_struct` 构造时接收 `Opt_trace_context*` 而不是 `Opt_trace_stmt*`，因为它要从 context 拿"当前结构"。

### 三、Opt_trace_struct：RAII 栈 + is_started 零成本

#### 3.0 三个类的权威定义（完整源码）

> **勘误（重要）**：8.0.39 的 optimizer trace **不用 `Json_writer`**（全仓唯一的 `class Json_writer` 是 keyring 组件里的无关实现，`sql/` 下零引用），也**没有 DOM/语法树**。它是**流式 `String::append`** + 一个"当前打开结构栈"。所以写进去就改不了，截断只能是硬切。

**累积器 `class Buffer`**（`sql/opt_trace.cc:56`）：

```cpp
namespace {
/**
  A wrapper of class String, for storing query or trace.
  Any memory allocation error in this class is reported by my_error(), see OOM_HANDLING in opt_trace.h.
*/
class Buffer {
 private:
  size_t allowed_mem_size;  ///< allowed memory size for this String
  size_t missing_bytes;     ///< how many bytes could not be added   ★ 截断计数
  String string_buf;        // ★★ 唯一的"存储"就是一个平铺字符串：没有树、没有节点、没有语法校验

 public:
  Buffer() : allowed_mem_size(0), missing_bytes(0) {}

  size_t alloced_length() const { return string_buf.alloced_length(); }
  size_t length() const { return string_buf.length(); }
  void prealloc();  ///< pro-actively extend buffer if soon short of space
  char *c_ptr_safe() { return string_buf.c_ptr_safe(); }
  const char *ptr() const { return string_buf.ptr(); }

  void append(const char *str, size_t length);
  void append(const char *str) { return append(str, strlen(str)); }
  /// 与 append() 相同，但对字符串值做 JSON 转义
  void append_escaped(const char *str, size_t length);
  void append(char chr);

  size_t get_allowed_mem_size() const { return allowed_mem_size; }
  size_t get_missing_bytes() const { return missing_bytes; }
  void set_allowed_mem_size(size_t a) { allowed_mem_size = a; }
};
}  // namespace
```

**`class Opt_trace_stmt`**（`sql/opt_trace.cc:108`）——**一条语句**的 trace（"CALL + 每个子语句"各一份）：

```cpp
class Opt_trace_stmt {
 public:
  Opt_trace_stmt(Opt_trace_context *ctx_arg);
  void end();                              // ★ 收尾：补 '\0' + DBUG 全文 + 必要时 restore_I_S
  bool has_ended() const { return ended; }
  void set_allowed_mem_size(size_t size);
  void set_query(const char *query, size_t length, const CHARSET_INFO *charset);

  /* 写 */
  bool open_struct(const char *key, Opt_trace_struct *ots,
                   bool wants_disable_I_S, char opening_bracket);
  void close_struct(const char *saved_key, bool has_disabled_I_S,
                    char closing_bracket);
  void separator();     /// Put optional comma, newline and indentation
  void next_line();     /// Put newline and indentation
  void add(const char *key, const char *val, size_t val_length, bool quotes,
           bool escape);

  /* 读 */
  void fill_info(Opt_trace_info *info) const;
  size_t alloced_length() const {
    return trace_buffer.alloced_length() + query_buffer.alloced_length();
  }
  void missing_privilege();
  bool support_I_S() const { return I_S_disabled == 0; }
  void disable_I_S() { ++I_S_disabled; }
  void restore_I_S() { --I_S_disabled; }
  const char *make_unknown_key();

 private:
  bool ended;
  /**
    0 <=> this trace should be in information_schema.
    Once disabled, it must stay disabled until re-enabled at the same stack
    frame. ... it is sufficient ... to have a counter incremented each time
    we get a request to disable I_S.
  */
  int I_S_disabled;
  bool missing_priv;
  Opt_trace_context *ctx;
  Opt_trace_struct *current_struct;         ///< current open structure
  Prealloced_array<Opt_trace_struct *, 16> stack_of_current_structs;
  // ★ 这"栈"就是全部语法状态：缩进 = 2 * size()
  Buffer trace_buffer;   ///< Where the trace is accumulated
  Buffer query_buffer;   ///< Where the original query is put
  uint unknown_key_count;
  char unknown_key[24];
};
```

**`class Opt_trace_struct`**（`sql/opt_trace.h:446`）——object 与 array 的公共基类，**RAII 本体**：

```cpp
class Opt_trace_struct {
 protected:
  Opt_trace_struct(Opt_trace_context *ctx_arg, bool requires_key_arg,
                   const char *key, Opt_trace_context::feature_value feature)
      : started(false) {
    // A first inlined test
    if (unlikely(ctx_arg->is_started())) {   // ★★ 第一道 inline 快路径：trace 没开 ⇒ 变 dummy
      do_construct(ctx_arg, requires_key_arg, key, feature);
    }
    /* Otherwise, just leave "started" to false, it marks the structure is dummy. */
  }
  ~Opt_trace_struct() {
    if (unlikely(started)) do_destruct();    // ★★ 析构 = 自动写 '}' 或 ']'
  }

 public:
  /// RAII 的例外：手动提前结束（end() 后再析构不会重复关闭）
  void end() {
    if (unlikely(started)) do_destruct();
  }

  /* add_alnum：值只含 7-bit 可打印 ASCII（无 " 与 \）时用它，不转义、更快
     add_utf8：任意 UTF8，会做 JSON 转义，用于表名/WHERE 等来自服务器对象的字符串 */
  Opt_trace_struct &add_alnum(const char *key, const char *value) {
    if (likely(!started)) return *this;      // ★ 每个 add 都有这个 inline if ⇒ 关闭时零调用
    return do_add(key, value, strlen(value), false);
  }
  Opt_trace_struct &add_utf8(const char *key, const char *value,
                             size_t val_length) {
    if (likely(!started)) return *this;
    return do_add(key, value, val_length, true);
  }
  Opt_trace_struct &add(const char *key, Item *item) {
    if (likely(!started)) return *this;
    return do_add(key, item);                // ★ Item 直接 print() 成 SQL 文本
  }
  Opt_trace_struct &add(const char *key, bool value) { ... }
  Opt_trace_struct &add(const char *key, longlong value) { ... }
  Opt_trace_struct &add(const char *key, ulonglong value) { ... }
  Opt_trace_struct &add(const char *key, double value) { ... }
  Opt_trace_struct &add(const char *key, const Cost_estimate &cost) { ... }
  Opt_trace_struct &add_null(const char *key) { ... }
  Opt_trace_struct &add_utf8_table(const Table_ref *tab) { ... }
  Opt_trace_struct &add_select_number(uint select_number) {
    return add("select#", select_number);
  }

  bool set_not_empty() {                     // ★ 决定"前面要不要补逗号"
    const bool old_empty = empty;
    empty = false;
    return old_empty;
  }
  const char *check_key(const char *key);    // ★ 运行时救错

 private:
  /// 故意只声明不定义：编译期禁止 add(key, const char*)，逼你用 add_alnum/add_utf8
  Opt_trace_struct &add(const char *key, const char *value);
  Opt_trace_struct &add(const char *key);

  bool started;              ///< Whether the structure does tracing or is dummy
  bool requires_key;         ///< true: object（要求 key）；false: array（禁止 key）
  bool has_disabled_I_S;     ///< 该结构所属 feature 被关 ⇒ 整棵子树不写 I_S
  bool empty;
  Opt_trace_stmt *stmt;
  const char *saved_key;     ///< 闭括号时要回写 "/* key */"
#ifndef NDEBUG
  char previous_key[25];     ///< 检测同一 object 内连续重复 key
#endif
};
```

`Opt_trace_object` / `Opt_trace_array` 只是两个子类，唯一区别是 `requires_key`：

```cpp
// sql/opt_trace.h:799
class Opt_trace_object : public Opt_trace_struct {
 public:
  Opt_trace_object(Opt_trace_context *ctx, const char *key,
                   Opt_trace_context::feature_value feature =
                       Opt_trace_context::MISC)
      : Opt_trace_struct(ctx, true, key, feature) {}     // requires_key=true ⇒ '{'
  Opt_trace_object(Opt_trace_context *ctx,
                   Opt_trace_context::feature_value feature =
                       Opt_trace_context::MISC)
      : Opt_trace_struct(ctx, true, nullptr, feature) {}
};

class Opt_trace_array : public Opt_trace_struct {
 public:
  Opt_trace_array(Opt_trace_context *ctx, const char *key, ...)
      : Opt_trace_struct(ctx, false, key, feature) {}    // requires_key=false ⇒ '['
  ...
};
```

**"写"的实现**（`sql/opt_trace.cc:478-596`）——这就是全部的"JSON 写入器"：

```cpp
namespace {
/// opening and closing symbols for arrays ([]) and objects ({})
const char brackets[] = {'[', '{', ']', '}'};
inline char opening_bracket(bool requires_key) { return brackets[requires_key]; }
inline char closing_bracket(bool requires_key) { return brackets[requires_key + 2]; }
}  // namespace

void Opt_trace_struct::do_construct(Opt_trace_context *ctx,
                                    bool requires_key_arg, const char *key,
                                    Opt_trace_context::feature_value feature) {
  saved_key = key;
  requires_key = requires_key_arg;
  stmt = ctx->get_current_stmt_in_gen();
  has_disabled_I_S = !ctx->feature_enabled(feature);   // ★ feature 关 ⇒ 整棵子树不写 I_S
  empty = true;
  if (likely(!stmt->open_struct(key, this, has_disabled_I_S,
                                opening_bracket(requires_key))))
    started = true;                                    // ★ open_struct 返回 true（OOM）⇒ 本结构变 dummy
}

void Opt_trace_struct::do_destruct() {
  assert(started);
  stmt->close_struct(saved_key, has_disabled_I_S, closing_bracket(requires_key));
  started = false;
}

bool Opt_trace_stmt::open_struct(const char *key, Opt_trace_struct *ots,
                                 bool wants_disable_I_S, char opening_bracket) {
  if (support_I_S()) {
    if (wants_disable_I_S) {
      /*
        User requested no tracing for this structure's feature. We are
        entering a disabled portion; put an ellipsis "..." to alert the user.
        Disabling applies to all the structure's children.
      */
      if (current_struct != nullptr) {
        if (key != nullptr)
          current_struct->add_alnum(key, "...");   // ★ feature 被关 ⇒ trace 里是 "key": "..."
        else
          current_struct->add_alnum("...");
      }
    } else {
      trace_buffer.prealloc();
      add(key, &opening_bracket, 1, false, false);
    }
  }
  if (wants_disable_I_S) ctx->disable_I_S_for_this_and_children();
  ...
  current_struct = ots;
  return false;
}

void Opt_trace_stmt::close_struct(const char *saved_key, bool has_disabled_I_S,
                                  char closing_bracket) {
  current_struct = stack_of_current_structs.back();
  stack_of_current_structs.pop_back();
  if (support_I_S()) {
    next_line();
    trace_buffer.append(closing_bracket);
    if (ctx->get_end_marker() && saved_key != nullptr) {
      trace_buffer.append(STRING_WITH_LEN(" /* "));   // ★★ end_markers_in_json=ON ⇒ 写 " /* key */"
      trace_buffer.append(saved_key);                 //    这是**非法 JSON**，纯为可读性（源码自己承认）
      trace_buffer.append(STRING_WITH_LEN(" */"));
    }
  }
  if (has_disabled_I_S) ctx->restore_I_S();
}

void Opt_trace_stmt::separator() {
  assert(support_I_S());
  // Put a comma first, if we have already written an object at this level.
  if (current_struct != nullptr) {
    if (!current_struct->set_not_empty()) trace_buffer.append(',');
    next_line();
  }
}

const char my_spaces[] = "                    ...";   // 128 个空格

void Opt_trace_stmt::next_line() {
  if (ctx->get_one_line()) return;             // ★ one_line=ON ⇒ 不换行不缩进
  trace_buffer.append('\n');
  size_t to_be_printed = 2 * stack_of_current_structs.size();   // ★★ 缩进完全由栈深决定
  ...
}

void Opt_trace_stmt::add(const char *key, const char *val, size_t val_length,
                         bool quotes, bool escape) {
  if (!support_I_S()) return;                  // ★ 只写 DBUG 的 trace：这里直接 return
  separator();
  if (current_struct != nullptr) key = current_struct->check_key(key);
  if (key != nullptr) {
    trace_buffer.append('"');
    trace_buffer.append(key);
    trace_buffer.append(STRING_WITH_LEN("\": "));
  }
  if (quotes) trace_buffer.append('"');
  if (escape)
    trace_buffer.append_escaped(val, val_length);
  else
    trace_buffer.append(val, val_length);
  if (quotes) trace_buffer.append('"');
}
```

**标量 → 字符串**（`sql/opt_trace.cc:290-377`，"没有魔法"）：

```cpp
Opt_trace_struct &Opt_trace_struct::do_add(const char *key, bool val) {
  const LEX_CSTRING *text = &bool_as_text[val];        // "false" / "true"
  stmt->add(key, text->str, text->length, false, false);   // quotes=false ⇒ 裸 true/false
  return *this;
}

Opt_trace_struct &Opt_trace_struct::do_add(const char *key, double val) {
  char buf[sizeof("-1.23456e+308")];
  const int len = std::snprintf(buf, sizeof(buf), "%g", val);
  // ★★ double 用 %g ⇒ 只有 6 位有效数字！trace 里 cost 的精度损失就源于此
  stmt->add(key, buf, len, false, false);
  return *this;
}

Opt_trace_struct &Opt_trace_struct::do_add(const char *key, Item *item) {
  char buff[256];
  String str(buff, sizeof(buff), system_charset_info);
  str.length(0);
  if (item != nullptr) {
    // QT_TO_SYSTEM_CHARSET because trace must be in UTF8
    item->print(current_thd, &str,
                enum_query_type(QT_TO_SYSTEM_CHARSET | QT_SHOW_SELECT_NUMBER |
                                QT_NO_DEFAULT_DB));
    return do_add(key, str.ptr(), str.length(), true);   // ★ 需要转义
  } else
    return do_add_null(key);
}

Opt_trace_struct &Opt_trace_struct::do_add(const char *key,
                                           const Cost_estimate &value) {
  return do_add(key, value.total_cost());      // ★ Cost_estimate → total_cost()
}
```

**`check_key()`：运行时救错**（`sql/opt_trace.cc:379`）：

```cpp
const char *Opt_trace_struct::check_key(const char *key) {
  stmt->assert_current_struct(this);           // 必须加到最内层结构
  bool has_key = key != nullptr;
  if (unlikely(has_key != requires_key)) {
    // fix the key to produce correct JSON syntax:
    key = has_key ? nullptr : stmt->make_unknown_key();
    // ★ 往 array 里传了 key ⇒ 丢掉；往 object 里没传 key ⇒ 造 "unknown_key_N"
    has_key = !has_key;
  }
  if (has_key) {
    /* Check that we're not having two identical consecutive keys in one object */
    assert(strncmp(previous_key, key, sizeof(previous_key) - 1) != 0);
    strncpy(previous_key, key, sizeof(previous_key) - 1);
  }
  return key;
}
```

⇒ **因为没有 DOM，"写错了改不回来"**：`end_markers_in_json` 的 `/* key */` 是非法 JSON；key 重复、array 带 key 只能运行时补救；截断是硬切。

#### 3.1 构造函数

```cpp
Opt_trace_struct(Opt_trace_context *ctx_arg, bool requires_key_arg,
                 const char *key, Opt_trace_context::feature_value feature)
    : ctx(ctx_arg) {
  if (unlikely(ctx_arg->is_started())) {
    do_construct(ctx_arg, requires_key_arg, key, feature);
  }
}
```

★ 三个要点：

- **`unlikely(is_started())`**：分支预测提示"基本为 false"，未开启 trace 时这条路径几乎不执行。
- **`is_started()` 的定义**：

```cpp
bool is_started() const {
  return unlikely(pimpl != nullptr) && pimpl->current_stmt_in_gen != nullptr;
}
```

两个判据：`pimpl` 非空（这个 session 曾经开启过 trace，pimpl 是惰性创建的）且当前有正在生成的语句。

- **`do_construct()` 是虚函数**，`Opt_trace_object` / `Opt_trace_array` 各自实现入栈动作。

#### 3.2 析构 = 出栈

对象析构时从栈里弹掉自己，恢复父结构。C++ 的栈展开（scope exit）保证了"嵌套的 trace 对象按正确顺序析构"。

★ 这意味着：**一个 `Opt_trace_object` 的生命周期就是它在 trace 里"打开到关闭"的区间**。你在源码里看到某个 `Opt_trace_object` 声明在函数开头，那这个节点就覆盖了整个函数体。

#### 3.3 add 系列

```cpp
Opt_trace_struct &add(const char *key, bool value);
Opt_trace_struct &add(const char *key, int value);
Opt_trace_struct &add(const char *key, ulonglong value);
Opt_trace_struct &add(const char *key, double value);
Opt_trace_struct &add(const char *key, const char *value);
Opt_trace_struct &add(const char *key, Item *item);
Opt_trace_struct &add(const char *key, const Cost_estimate &cost);
...
```

返回 `Opt_trace_struct&` 本身，所以可以**链式调用**（`trace_derived.add(...).add(...)`）。

### 四、add 系列重载与 feature 机制

#### 4.1 feature：只 trace 关心的部分

`optimizer_trace_features` 变量按位掩码过滤：

| feature | 位 | 含义 |
|---|---|---|
| `greedy_search` | 1<<0 | join order 的贪心搜索 |
| `range_optimizer` | 1<<1 | range 优化器的代价分析 |
| `dynamic_range` | 1<<2 | 动态 range（每次访问时重算） |
| `repeated_subselect` | 1<<3 | 子查询的重复执行 |
| `MISC` | 1<<7 | ★ 兜底分类，**不能被禁用**（注释：否则 empty trace） |

`feature_enabled()`：

```cpp
bool feature_enabled(feature_value f) const {
  return unlikely(pimpl != nullptr) && (pimpl->features & f);
}
```

`Opt_trace_object` 构造时的 `feature` 参数就是声明"这个节点属于哪个 feature"。`MISC` 是兜底，且通过"父节点 feature 的继承"——**禁用 MISC 会让整个 trace 变空**，所以它不能被用户禁用。

#### 4.2 为什么 MISC 不能禁用

头注释说得很清楚：MISC 包括"顶层对象"，而 feature 是**沿父子链继承**的——顶层是 MISC，那所有子节点默认也"继承" MISC 属性，禁了 MISC 就什么都看不到了。

### 五、start/end 生命周期与 OFFSET/LIMIT

#### 5.0 start() / end() / purge_stmts() 完整源码

```cpp
// sql/opt_trace.cc:818
bool Opt_trace_context::start(bool support_I_S_arg,
                              bool support_dbug_or_missing_priv,
                              bool end_marker_arg, bool one_line_arg,
                              long offset_arg, long limit_arg,
                              ulong max_mem_size_arg, ulonglong features_arg) {
  if (I_S_disabled != 0) {
    DBUG_PRINT("opt", ("opt_trace is already disabled"));
    support_I_S_arg = false;      // ★ 上层曾 disable（如 insufficient privilege）⇒ 本语句不进 I_S
  }

  /*
    Decide on optimizations possible to realize the requested support. ...
  */
  if (!support_I_S_arg && !support_dbug_or_missing_priv) {
    // The statement will not do tracing.
    if (likely(pimpl == nullptr) || pimpl->current_stmt_in_gen == nullptr) {
      /*
        This should be the most commonly taken branch in a release binary,
        when the connection rarely has optimizer tracing runtime-enabled.
        It's thus important that it's optimized: we can short-cut the creation
        and starting of Opt_trace_stmt ...
      */
      return false;               // ★★ 热路径：trace 关闭时零分配直接返回
    }
    /*
      If we come here, there is a parent statement which has a trace.
      Imagine that we don't create a trace for the child statement here. Then
      trace structures of the child will be accidentally attached to the
      parent's trace (as it is still 'current_stmt_in_gen') ... To avoid this,
      we create a trace for the child but with I_S output disabled.
    */
  }

  if (pimpl == nullptr &&
      ((pimpl = new_nothrow_w_my_error<Opt_trace_context_impl>()) == nullptr))
    return true;                  // ★ 第一次开 trace 才堆分配 pimpl（THD 体积不受影响）

  if (I_S_disabled == 0) {
    /*
      Here we allow a stored routine's sub-statement to enable/disable tracing,
      or change settings. Thus in a stored routine's body, there can be some
      'SET OPTIMIZER_TRACE="enabled=[on|off]"' to trace only certain
      sub-statements.
    */
    pimpl->end_marker = end_marker_arg;      // @@end_markers_in_json
    pimpl->one_line = one_line_arg;          // optimizer_trace 的 one_line flag
    pimpl->offset = offset_arg;
    pimpl->limit = limit_arg;
    pimpl->max_mem_size = max_mem_size_arg;
    // MISC always on
    pimpl->features = Opt_trace_context::feature_value(features_arg |
                                                       Opt_trace_context::MISC);
    // ★★ MISC 强制 OR 进去：关掉它会连顶层对象一起关 ⇒ 空 trace
  }

  if (support_I_S_arg && pimpl->offset >= 0) {
    /* If outside the offset/limit window, no need to support I_S */
    if (pimpl->since_offset_0 < pimpl->offset) {
      support_I_S_arg = false;
    } else if (pimpl->since_offset_0 >= (pimpl->offset + pimpl->limit)) {
      support_I_S_arg = false;
    }
    pimpl->since_offset_0++;      // ★ 只在 offset>=0 时递增
  }
  // ★ offset<0 时不做窗口判定 ⇒ 每条都先 trace，靠 purge_stmts() 滑动淘汰

  {
    /*
      We don't allocate it in THD's MEM_ROOT as it must survive until a next
      statement (SELECT) reads the trace.
    */
    Opt_trace_stmt *stmt = new_nothrow_w_my_error<Opt_trace_stmt>(this);

    if (unlikely(stmt == nullptr ||
                 pimpl->stack_of_current_stmts.push_back(
                     pimpl->current_stmt_in_gen)))   // ★ 父语句压栈（CALL ⇄ 子语句）
      goto err;

    Opt_trace_stmt_array *list;
    if (support_I_S_arg)
      list = &pimpl->all_stmts_for_I_S;      // ★ 进"可见"队列
    else {
      stmt->disable_I_S();  // no need to fill a not-shown JSON trace
      list = &pimpl->all_stmts_to_del;       // ★ 只写 DBUG 的 trace 直接进"待删"队列
    }

    if (unlikely(list->push_back(stmt))) goto err;

    pimpl->current_stmt_in_gen = stmt;

    // As we just added one trace, maybe the previous ones are unneeded now
    purge_stmts(false);
    // This purge may have freed space, compute max allowed size:
    stmt->set_allowed_mem_size(allowed_mem_size_for_current_stmt());
    return false;
  err:
    delete stmt;
    assert(0);
    return true;
  }
}

void Opt_trace_context::end() {
  assert(I_S_disabled >= 0);
  if (likely(pimpl == nullptr)) return;
  if (pimpl->current_stmt_in_gen != nullptr) {
    pimpl->current_stmt_in_gen->end();     // ★ 补 '\0' + DBUG 全文 + 必要时 restore_I_S
    Opt_trace_stmt *const parent = pimpl->stack_of_current_stmts.back();
    pimpl->stack_of_current_stmts.pop_back();
    pimpl->current_stmt_in_gen = parent;   // ★ 弹栈回到父语句（CALL 的 trace）
    if (parent != nullptr) {
      /* Parent regains control, now it needs to be told that its child has
         used space, and thus parent's allowance has shrunk. */
      parent->set_allowed_mem_size(allowed_mem_size_for_current_stmt());
    }
    purge_stmts(false);
  }
}
```

**"只保留最后一条"到底怎么实现**（`sql/opt_trace.cc:991` 的 `purge_stmts()`）：

```cpp
void Opt_trace_context::purge_stmts(bool purge_all) {
  if (!purge_all && pimpl->offset >= 0) {
    /* This case is managed in Opt_trace_context::start() */
    return;      // ★ offset>=0：窗口在 start() 里判，这里不动
  }
  long idx;
  /*
    Start from the newest traces (array's end), scroll back in time. This
    direction is necessary, as we may delete elements from the array ...
  */
  for (idx = (pimpl->all_stmts_for_I_S.size() - 1); idx >= 0; idx--) {
    // offset can be negative, so cast size() to signed!
    if (!purge_all && ((static_cast<long>(pimpl->all_stmts_for_I_S.size()) +
                        pimpl->offset) <= idx)) {
      /* OFFSET mandates that this trace should be kept; move to previous */
      // ★★ 核心公式：保留条件 idx >= size + offset
      //    offset=-1 ⇒ 只留 idx == size-1（最后一条）——这就是"覆盖"的真相
    } else {
      if (likely(!pimpl->all_stmts_to_del.push_back(
              pimpl->all_stmts_for_I_S.at(idx))))
        pimpl->all_stmts_for_I_S.erase(idx);   // 先从"可见队列"摘掉
      else { /* OOM. Cannot purge ... */ }
    }
  }
  /* Examine list of "to be freed" traces and free what can be */
  for (idx = (pimpl->all_stmts_to_del.size() - 1); idx >= 0; idx--) {
    Opt_trace_stmt *stmt = pimpl->all_stmts_to_del.at(idx);
    if (!stmt->has_ended() || skip_del) {
      /*
        This trace is not finished, freeing it now would lead to use of
        freed memory if a structure is later added to it. ... (CALL + 子语句的例子)
        So if a trace is not finished, we will wait until it is and
        re-consider it then (which is why this function is called in
        Opt_trace_stmt::end() too).
      */
    } else {
      pimpl->all_stmts_to_del.erase(idx);
      delete stmt;      // ★ 真正 free 的时机：end() 之后（未 end 的 trace 不能删，会 UAF）
    }
  }
}
```

**内存预算不是"每条 1MB"**（`sql/opt_trace.cc:1075`）：

```cpp
size_t Opt_trace_context::allowed_mem_size_for_current_stmt() const {
  size_t mem_size = 0;
  int idx;
  for (idx = (pimpl->all_stmts_for_I_S.size() - 1); idx >= 0; idx--) {
    const Opt_trace_stmt *stmt = pimpl->all_stmts_for_I_S.at(idx);
    mem_size += stmt->alloced_length();
  }
  // Even to-be-deleted traces use memory, so consider them in sum
  for (idx = (pimpl->all_stmts_to_del.size() - 1); idx >= 0; idx--) {
    const Opt_trace_stmt *stmt = pimpl->all_stmts_to_del.at(idx);
    mem_size += stmt->alloced_length();
  }
  /* The current statement is in exactly one of the two lists above */
  mem_size -= pimpl->current_stmt_in_gen->alloced_length();
  size_t rc =
      (mem_size <= pimpl->max_mem_size) ? (pimpl->max_mem_size - mem_size) : 0;
  return rc;   // ★ 预算 = max_mem_size − 其他所有 trace 已占（含待删队列）
}
```

**截断判定**（`sql/opt_trace.cc:625` 的 `append_escaped()` / `:713` 的 `append()`）——三个 `append` 版本都是同一套 up-front 丢弃：

```cpp
void Buffer::append(const char *str, size_t length) {
  if (alloced_length() >= allowed_mem_size) {
    missing_bytes += length;      // ★ 只累加"少了多少字节"，不写任何内容
    return;
  }
  string_buf.append(str, length);
}

void Buffer::append(char chr) {
  if (alloced_length() >= allowed_mem_size) {
    missing_bytes++;              // ★★ 连 '}' 都可能被丢 ⇒ JSON 结尾可能不完整
    return;
  }
  string_buf.append(chr);
}
```

⇒ 判定用的是 **`alloced_length()`（已分配容量）而非 `length()`（已用长度）**，加上 `prealloc()` 会预留 `ALIGN_SIZE(1)+1` 的安全边距，所以实际写出的 JSON 通常**略小于** `max_mem_size`。

**QUERY 会抢 TRACE 的预算**（`sql/opt_trace.cc:448`，极反直觉）：

```cpp
void Opt_trace_stmt::set_query(const char *query, size_t length,
                               const CHARSET_INFO *charset) {
  assert(query_buffer.ptr() == nullptr);   // Should be called only once per statement
  query_buffer.set_charset(charset);
  if (!support_I_S()) return;              // 不进 I_S ⇒ 不浪费资源存 query
  // We are taking a bit of space from 'trace_buffer'.
  size_t available = (trace_buffer.alloced_length() >=
                      trace_buffer.get_allowed_mem_size())
                         ? 0
                         : (trace_buffer.get_allowed_mem_size() -
                            trace_buffer.alloced_length());
  query_buffer.set_allowed_mem_size(available);
  query_buffer.append(query, length);
  // Space which query took is taken out of the trace:
  const size_t new_allowed_mem_size =
      (query_buffer.alloced_length() >= trace_buffer.get_allowed_mem_size())
          ? 0
          : (trace_buffer.get_allowed_mem_size() -
             query_buffer.alloced_length());
  trace_buffer.set_allowed_mem_size(new_allowed_mem_size);
  // ★ 所以 max_mem_size 很小而 SQL 很长时，TRACE 会是空串
}
```

#### 5.1 start 的参数

```cpp
bool start(bool support_I_S, bool support_dbug_or_missing_priv,
           bool end_marker, bool one_line, long offset, long limit,
           ulong max_mem_size, ulonglong features);
```

| 参数 | 含义 |
|---|---|
| `support_I_S` | 这条语句的 trace 是否进 I_S |
| `end_marker` | 关闭对象时是否加人类可读注释（`} /* key */`） |
| `one_line` | 单行紧凑输出（给程序读），还是多行缩进（给人读） |
| `offset` / `limit` | 限制 trace 的生产（对应 `optimizer_trace_offset`/`limit` 变量） |
| `max_mem_size` | 所有记忆的 trace 累计大小上限 |
| `features` | 只 trace 这些 feature |

#### 5.2 为什么有 offset/limit

trace 可能很大，尤其 `greedy_search` 会为每个候选 join 序都写一堆。offset/limit 让"只记录第 N 条语句的一部分"成为可能，配合 I_S 表里的 `MISSING_BYTES_BEYOND_MAX_MEM_SIZE` 列告知"有没有被截断"。

### 六、进 information_schema：填充与 SUID 安全洞

> `fill_optimizer_trace_info()`，`sql/opt_trace2server.cc`。

#### 6.1 表结构

| 列 | 类型 | 含义 |
|---|---|---|
| `QUERY` | string | 原始查询 |
| `TRACE` | string | trace JSON |
| `MISSING_BYTES_BEYOND_MAX_MEM_SIZE` | long | 超限被截断的字节数 |
| `INSUFFICIENT_PRIVILEGES` | tiny | 是否因权限不足而为空 |

#### 6.2 ★ SUID 安全洞与堵法

这是这个文件里最有价值的一段注释，讲的是一个**真实的安全漏洞**：

```
When executing a routine which is SQL SECURITY DEFINER, opt-trace specific
checks are done with the connected user's privileges; this isn't respecting
the meaning of SQL SECURITY DEFINER. If a highly privileged user doesn't know
that, he may confidently execute a routine, while this routine nastily uses
the connected user's privileges to be allowed to do tracing and gain
knowledge about secret objects.
```

场景：**SQL SECURITY DEFINER** 的存储过程用**定义者（高权限）** 的权限执行，但 optimizer_trace 的权限检查却用**连接用户**的权限做。于是：高权限用户以为自己在安全地执行 routine，routine 却用连接用户（低权限）的身份去开 trace，**通过 trace 泄露 secret 对象的信息**。

堵法（`fill_optimizer_trace_info` 开头）：

```cpp
if (!(thd->security_context()->check_access((GLOBAL_ACLS & ~GRANT_ACL),
                                            tables->get_db_name())) &&
    (连接用户的 user/host != 主 security_context 的 user/host))
  return 0;   // 让 I_S.OPTIMIZER_TRACE 看起来为空
```

即：**当 SUID 上下文不是连接用户本人时，让 I_S.OPTIMIZER_TRACE 返回空**，除非该 SUID 上下文拥有全部全局权限（那样它本来就有权 trace 一切）。

★ 这是一个"优化器 trace 也不放过安全边界"的典型例子——trace 看起来是"只读调试信息"，但它的信息量足以构成侧信道。

#### 6.2.1 I_S 表注册与填充函数（完整源码）

> **勘误**：表**不是** `sql/dd/impl/system_views/optimizer_trace.cc`（该文件不存在），填充函数也**不是** `fill_schema_optimizer_trace`，而是 `fill_optimizer_trace_info`（`sql/opt_trace2server.cc:480`），走的是遗留硬编码 `schema_tables[]`。

```cpp
// sql/sql_show.cc:5118 —— 遗留式 I_S 表：字段描述 + fill 回调，不走 DD system view
extern ST_FIELD_INFO optimizer_trace_info[];    // 定义在 opt_trace2server.cc:534

ST_SCHEMA_TABLE schema_tables[] = {
    {"COLUMN_PRIVILEGES", column_privileges_fields_info,
     fill_schema_column_privileges, nullptr, nullptr, false},
    ...
    {"OPTIMIZER_TRACE", optimizer_trace_info, fill_optimizer_trace_info,
     nullptr, nullptr, false},
    ...
};
```

```cpp
// sql/opt_trace2server.cc:480
int fill_optimizer_trace_info(THD *thd, Table_ref *tables, Item *) {
  TABLE *table = tables->table;
  Opt_trace_info info;

  /*
    When executing a routine which is SQL SECURITY DEFINER, opt-trace specific
    checks are done with the connected user's privileges; this isn't
    respecting the meaning of SQL SECURITY DEFINER. If a highly privileged
    user doesn't know that, he may confidently execute a routine, while this
    routine nastily uses the connected user's privileges to be allowed to do
    tracing and gain knowledge about secret objects.
    This possibility is prevented, by making I_S.OPTIMIZER_TRACE look empty
    when read from a security context which isn't the connected user's context;
    with an exception if the SUID security context has all global privileges
    (in which case the nasty definer has anyway all rights to trace everything).
    ...
  */
  ...（SUID 判定，见 6.2）

  for (Opt_trace_iterator it(&thd->opt_trace); !it.at_end(); it.next()) {
    it.get_value(&info);
    restore_record(table, s->default_values);
    /*
      We will put the query, which is in character_set_client, into a column
      using character_set_client; this is better than UTF8 (see BUG#57306).
    */
    table->field[0]->store(info.query_ptr,
                           static_cast<uint>(info.query_length),
                           info.query_charset);              // ★ 列 0 = QUERY
    table->field[1]->store(info.trace_ptr,
                           static_cast<uint>(info.trace_length),
                           system_charset_info);             // ★ 列 1 = TRACE（被硬截断的 JSON）
    table->field[2]->store(info.missing_bytes, true);        // ★ 列 2 = MISSING_BYTES_BEYOND_MAX_MEM_SIZE
    table->field[3]->store(info.missing_priv, true);         // ★ 列 3 = INSUFFICIENT_PRIVILEGES
    if (schema_table_store_record(thd, table)) return 1;
  }
  return 0;
}
```

**`missing_bytes` 从 Buffer 走到列**（`sql/opt_trace.cc:598`）：

```cpp
void Opt_trace_stmt::fill_info(Opt_trace_info *info) const {
  if (unlikely(info->missing_priv = missing_priv)) {
    info->trace_ptr = info->query_ptr = "";     // ★ 缺权限 ⇒ 整行看起来是空的
    info->trace_length = info->query_length = 0;
    info->query_charset = &my_charset_bin;
    info->missing_bytes = 0;                    // ★ 缺权限时不报 missing bytes
  } else {
    info->trace_ptr = trace_buffer.ptr();
    info->trace_length = trace_buffer.length();
    info->query_ptr = query_buffer.ptr();
    info->query_length = query_buffer.length();
    info->query_charset = query_buffer.charset();
    info->missing_bytes =
        trace_buffer.get_missing_bytes() + query_buffer.get_missing_bytes();
    // ★ trace 丢的 + query 丢的，两者相加
  }
}
```

⇒ **`"missing_bytes_beyond_max_mem_size": N` 这段 JSON 不存在**（全仓 `sql/` 下只有大写列名 `MISSING_BYTES_BEYOND_MAX_MEM_SIZE` 出现一次）。它是独立的第 3 列；JSON 本身是被硬截断的，通常已不是合法 JSON。

**生命周期终点没有独立函数**——就是 `~Opt_trace_start()` 的析构（RAII）。起点在 `sql/sql_parse.cc:3207`：

```cpp
  Opt_trace_start ots(thd, all_tables, lex->sql_command, &lex->var_list,
                      thd->query().str, thd->query().length, nullptr,
                      thd->variables.character_set_client);
  // ★ 语句级 trace 的唯一入口；离开作用域（含栈展开）即结束

  Opt_trace_object trace_command(&thd->opt_trace);              // 根对象 '{'
  Opt_trace_array trace_command_steps(&thd->opt_trace, "steps"); // "steps": [
```

终点 `Opt_trace_stmt::end()`（`sql/opt_trace.cc:418`）：

```cpp
void Opt_trace_stmt::end() {
  assert(stack_of_current_structs.size() == 0);   // ★ 所有结构都已闭合（RAII 保证）
  assert(I_S_disabled >= 0);
  ended = true;
  /*
    Because allocation is done in big chunks, buffer->Ptr[str_length]
    may be uninitialized while buffer->Ptr[allocated length] is 0, so we
    must use c_ptr_safe() as we want a 0-terminated string ...
  */
  trace_buffer.c_ptr_safe();   // ★ 补 '\0'：只有 end() 之后 TRACE 才是 0 结尾
  // Send the full nice trace to DBUG.
  DBUG_EXECUTE("opt", {
    const char *trace = trace_buffer.c_ptr_safe();
    DBUG_LOCK_FILE;
    fputs("Complete optimizer trace:", DBUG_FILE);
    fputs(trace, DBUG_FILE);
    fputs("\n", DBUG_FILE);
    DBUG_UNLOCK_FILE;
  });   // ★★ "完整漂亮"的 trace 只送往 DBUG；I_S 里的可能已被截断
  if (unlikely(missing_priv)) ctx->restore_I_S();
  // ★ 关键：把 missing_privilege() 加的 I_S 禁用在这里还回去，
  //   否则"一次缺权限 ⇒ 整个连接余生都没 trace"
}
```

#### 6.3 Opt_trace_iterator

遍历所有 remember 的 trace（按 oldest→newest 返回），配合 `get_value()` 取出 `Opt_trace_info{query, trace, missing_bytes, missing_priv}` 逐条 store 进表。

### 七、JSON 生成：end_marker 与 one_line

`end_marker` 参数控制关闭对象时是否追加 `/* key */` 注释：

```json
"key_foo": {
            多行内容
          } /* key_foo */
```

头注释明确说明：**这不是合法 JSON，纯为人类可读**。注释里还提到 YAML 支持 `#` 注释，"我们只需要把当前项的 `,` 放到 `#` 前面就行"——即未来若想支持 YAML 输出，改动很小。

`one_line` 则是给程序/网络传输用的紧凑模式（无缩进）。

---

## ★ 本机制里的工程实现技法

### 1. 用 RAII 实现隐式栈，避免层层传参

这是全篇最重要的技法。核心矛盾（深栈埋点 vs 函数签名）的解法是：**把"当前结构"塞进 context，用 C++ 作用域的生命周期管理入栈/出栈**。代价是埋点必须成对、顺序必须正确，但换来了"任意深处都能写 trace 而无需改函数签名"。

### 2. `unlikely()` + 指针判空 = 零成本开关

```cpp
if (unlikely(ctx_arg->is_started())) { ... }
```

未开启 trace 时，这只是一个"几乎不命中的分支"。生产环境关闭 trace 的性能影响趋近于零。这是"零成本抽象"的教科书式实现：**开关成本压在分支预测上，而不是运行时判断上**。

### 3. 用位掩码做 feature 过滤 + 继承

feature 是 `1<<N` 的位掩码，父节点的 feature 沿继承链传递。`MISC = 1<<7` 故意取最大位且"不可禁用"，保证总有一个兜底分类。

### 4. `add()` 返回 `*this` 实现链式调用

大量 `add()` 重载 + 返回引用，让埋点代码写成一句流畅的链式调用，减少样板。

### 5. 用引用计数管理"临时禁用 I_S"

```cpp
void disable_I_S_for_this_and_children() { ++I_S_disabled; ... }
void restore_I_S() { --I_S_disabled; assert(I_S_disabled >= 0); ... }
```

`missing_privilege()` 会临时禁用 I_S，靠引用计数保证"禁用/恢复"正确配对，且 `end()` 时自动恢复（头注释强调这一点是"确保 missing_privilege 不会把 I_S 支持关掉整个连接生命周期"的关键）。

### 6. 安全边界写进 I_S 填充函数

SUID 安全洞的堵法不是靠"trace 生成时检查权限"，而是**在 I_S 填充时检查**——这样 trace 本身照常生成（不影响优化），只在"读"的时候做权限隔离。这是"读侧防护"的思路。

---

## 可观测性

### 相关变量

| 变量 | 作用 |
|---|---|
| `optimizer_trace` | flagset：`enabled` / `one_line` / `default` |
| `optimizer_trace_features` | flagset：`greedy_search` / `range_optimizer` / `dynamic_range` / `repeated_subselect` / `default` |
| `optimizer_trace_offset` / `optimizer_trace_limit` | 限制 trace 生产 |
| `optimizer_trace_max_mem_size` | 累计内存上限 |

### I_S 表

```sql
SELECT QUERY, TRACE, MISSING_BYTES_BEYOND_MAX_MEM_SIZE, INSUFFICIENT_PRIVILEGES
FROM information_schema.optimizer_trace;
```

### 怎么读源码时找到埋点

| 想看什么 | grep 什么 |
|---|---|
| 某个 trace 节点在哪埋的 | 那个节点名（如 `semijoin_strategy_choice`） |
| 某段代码有没有埋点 | 看有没有 `Opt_trace_object` / `Opt_trace_array` |
| feature 属于哪类 | 构造时的 `feature` 参数 |

★ 反过来也成立：**读优化器源码时，`Opt_trace_object` 的出现就是"这里是一个决策/阶段边界"的标记**。它们是理解控制流的免费路标。

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 新增一个 feature 分类 | `opt_trace_context.h` 的 `feature_value` 枚举 + `feature_names[]` + `default_features`（头注释提醒了三处要同步） |
| 新增一种可 trace 的类型 | `Opt_trace_struct::add()` 加一个重载 + `do_add()` |
| 让 trace 输出 YAML | 头注释里说只需把 `,` 移到 `#` 前（见 `end_marker` 注释） |
| 新增 I_S 列 | `optimizer_trace_info[]` + `Opt_trace_info` 结构 |

### 已知缺陷

- **`add(Item*)` 的序列化可能不完整**：Item 树很复杂，trace 里的 Item 只是摘要。
- **trace 可能很大**：`greedy_search` 为每个候选写大量数据，靠 `max_mem_size` 截断。
- **SUID 安全洞是补丁式堵漏**：靠"读侧检查"而非"生成侧隔离"，语义上仍有边界情况（注释里自己说明了"除非有全部全局权限"的例外）。

### 社区边界澄清

- optimizer_trace 是**社区版完整可用**的功能，不是 Oracle 私有。
- 但 **Oracle 内部的诊断工具**（如更底层的 CBO trace）不在社区版里，不要混淆。
- `optimizer_trace_features` 的 `repeated_subselect` 等分类，是给"某类优化单独开 trace"用的，社区版同样支持。

---

## 参考

**论文 / 理论**

- 无特定论文。这是一个**工程实现**（栈式 JSON 生成器 + RAII），最接近的思想是"零成本抽象"与"日志系统的结构化输出"（如 Go 的 `slog`、Rust 的 `tracing` span）

**官方文档**

- MySQL 8.0 Reference Manual, "Tracing the Optimizer"（`OPTIMIZER_TRACE` 的使用）
- MySQL 8.0 Reference Manual, "The Information Schema OPTIMIZER_TRACE Table"

**源码出处**

- `sql/opt_trace_context.h` —— `Opt_trace_context` 声明与设计注释（本篇第二章的核心动机、第三章的 `is_started()`、第四章的 feature 枚举）
- `sql/opt_trace.h` —— `Opt_trace_struct` / `Opt_trace_object` / `Opt_trace_array`（RAII 与 add 系列）
- `sql/opt_trace.cc` —— 栈操作与 JSON 序列化实现
- `sql/opt_trace2server.cc` —— `fill_optimizer_trace_info()`（I_S 填充 + SUID 安全洞）

**相关文档**

- [`../11_explain_and_trace.md`](../../../infra/observability/explain_trace.md) —— 怎么用 trace（读节点、排查）
- [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)、[`17_optimizer_decisions.md`](17_optimizer_decisions.md) —— 各决策点埋了哪些 trace 节点
- [`../../infra/dbug.md`](../../infra/dbug.md) —— DBUG 通道（与 trace 的另一条调试路径）

# 23 optimizer_trace 的实现原理：一份零成本的栈式 JSON 生成器

> 基于 MySQL 8.0.39。本篇剖析 optimizer_trace **基础设施本身**是怎么工作的：`Opt_trace_context`（会话级上下文）与 `Opt_trace_struct`/`Opt_trace_object`/`Opt_trace_array`（RAII 栈）如何协作，让优化器深处的埋点不需要层层传参；`is_started()` 如何做到"未开启时零成本"；trace 如何带 `feature` 分类、如何进 `information_schema.optimizer_trace`，以及那个 SQL SECURITY DEFINER 安全洞是怎么堵的。
>
> **边界**：本篇讲"trace 机制怎么实现"。**怎么用 trace 观察优化器**（读哪些节点、排查哪些问题）见 [`../11_explain_and_trace.md`](../11_explain_and_trace.md)。DBUG 通道见 [`../../infra/dbug.md`](../../infra/dbug.md)。

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

- [`../11_explain_and_trace.md`](../11_explain_and_trace.md) —— 怎么用 trace（读节点、排查）
- [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)、[`17_optimizer_decisions.md`](17_optimizer_decisions.md) —— 各决策点埋了哪些 trace 节点
- [`../../infra/dbug.md`](../../infra/dbug.md) —— DBUG 通道（与 trace 的另一条调试路径）

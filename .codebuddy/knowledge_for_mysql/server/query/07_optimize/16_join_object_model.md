# 16 优化器核心对象模型与生命周期：谁在什么时候写谁

> 基于 MySQL 8.0.39。本篇从**对象视角**（而不是算法视角）把优化期的全部核心结构归位：`JOIN` / `JOIN_TAB` / `QEP_TAB` / `QEP_shared` / `POSITION` / `Key_use` / `Temp_table_param` / `ORDER_with_src` / `ref_items`，以及它们从 `lex_start` 到 `JOIN::destroy` 的完整生命周期与内存归属。
>
> **边界**：本篇讲"字段是什么、谁写、谁读、何时失效"。字段背后的**算法**不在这里：
> - `POSITION` 字段语义与 join order 搜索 → [`physical/06_join_order.md`](physical/06_join_order.md)
> - `Key_use` 语义与 `find_best_ref` 代价 → [`physical/07_access_method.md`](physical/07_access_method.md)
> - `AccessPath` 结构与两条创建路径 → [`../08_access_path.md`](../08_access_path.md)
> - range 优化器内部（`SEL_ARG` / `QUICK_*`）→ [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)
> - 分组/去重相关标志的推导过程 → [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)
> - hypergraph 的对象模型（无 QEP_TAB）→ [`physical/09_hypergraph.md`](physical/09_hypergraph.md)

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、四层对象全景](#一四层对象全景)
  - [二、JOIN：字段分组剖析](#二join字段分组剖析)
  - [三、表的三种化身：JOIN_TAB / QEP_TAB / QEP_shared](#三表的三种化身join_tab--qep_tab--qep_shared)
  - [四、五个数组：谁分配、谁重排、谁被置空](#四五个数组谁分配谁重排谁被置空)
  - [五、Temp_table_param：临时表的参数母版](#五temp_table_param临时表的参数母版)
  - [六、ORDER_with_src 与 ref_items 切片](#六order_with_src-与-ref_items-切片)
  - [七、TABLE 上的优化器相关成员](#七table-上的优化器相关成员)
  - [八、完整生命周期与内存归属](#八完整生命周期与内存归属)
  - [九、新旧优化器对象模型对照](#九新旧优化器对象模型对照)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`JOIN` 是 MySQL 经典优化器**唯一**的会话对象：一个 `Query_block` 一次优化建一个，它既是"优化结果容器"，也是优化过程中的"草稿板"。围绕它有一族伴生结构：

| 结构 | 一句话定位 |
|---|---|
| `JOIN` | query block 级的全局状态黑板（条件、表序、聚合、临时表、计划产物） |
| `JOIN_TAB` | **规划期**对一张表的抽象（候选键、依赖、行数、range 计划） |
| `QEP_TAB` | **执行准备期**对一张表的抽象（op_type、filesort、semi-join 策略、写回函数） |
| `QEP_shared` | 上述两者**共享**的那一半数据（table / position / type / condition / keys / range_scan / ref） |
| `POSITION` | 一个"表 + 访问方法 + 前缀代价"的 POD 快照，构成 `positions[]` / `best_positions[]` |
| `Key_use` | 一条等值谓词被抽象成的"索引查找弧"，构成 `JOIN::keyuse_array` |
| `Temp_table_param` | 建内部临时表用的参数模板；`JOIN::tmp_table_param` 是"母版" |
| `ORDER_with_src` | `ORDER*` 链表 + **溯源标签**（EXPLAIN 用） |
| `ref_items` | SELECT list 的**多份切片**，供临时表/窗口阶段换指针用 |

### 为什么需要这一篇

读 `sql_optimizer.cc` 的典型体验是：**每个函数都在读写 `JOIN` 的几十个成员，但没有任何一处说明这些成员此刻处于什么状态**。

反直觉的例子 —— 下面这几个指针在 `JOIN::optimize()` 的不同时刻**指向不同的东西，或干脆为空**：

```
make_join_plan() 之前 :  join_tab == nullptr, best_ref == nullptr, qep_tab == nullptr
make_join_plan() 之内 :  join_tab 有效, best_ref 有效（会被重排）, qep_tab == nullptr
get_best_combination():  join_tab 被置空, best_ref 有效, qep_tab 有效
make_tmp_tables_info():  best_ref 被置空, qep_tab 有效
```

也就是说：**`join_tab` 和 `best_ref` 在正常路径上都会被人主动置空，而它们指向的内存仍然活着**。不懂这条约定，读到 `QEP_TAB::cleanup()` 里那句"若 `join()->qep_tab` 非空则什么都不做"就会一头雾水。

本篇的目标就是让读者**随机打开 `sql_optimizer.cc` 的任意一处都能对上状态**。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.0–5.5 | `JOIN` + `JOIN_TAB` 一元结构：优化与执行共用同一批 `JOIN_TAB`（含 `read_first_record` 等函数指针） |
| 5.6 | 拆出 `QEP_TAB` + `QEP_shared`：规划期与执行期通过共享数据避免拷贝 |
| 5.7 | `Temp_table_param` 独立成头文件 |
| 8.0.0+ | `ORDER_with_src` 引入（`src` / `const_optimized` 溯源）；`ref_items` 切片机制成熟 |
| 8.0.14+ | `Window` 与 `JOIN::m_windows`；windowing 用独立的 `REF_SLICE_WIN_*` 切片 |
| 8.0.20–8.0.22 | `AccessPath` 引入：`JOIN::m_root_access_path` 成为优化器产物；`create_access_paths()` 把 `QEP_TAB` 链翻译成树 |
| 8.0.22 | hypergraph 优化器入场：`FindBestQueryPlan()` 直接产出 `AccessPath`，完全不碰 `QEP_TAB` / `POSITION` |
| 8.0.2x | `JOIN::filesorts_to_cleanup` 加入：**只有 hypergraph 用**（旧优化器把 filesort 挂在 `QEP_TAB` 上） |
| 8.0.39 | 现状：两套对象模型并存，由 `LEX::using_hypergraph_optimizer()` 分流 |

---

## 理论基础

### 可变黑板 vs 不可变计划树

MySQL 的经典优化器选择了**可变黑板（mutable blackboard）**风格：

- 一个 `JOIN` 持有一切；
- 表集合用**多个平行数组**表示，随优化推进被反复 `memmove` / `swap` / `memcpy`；
- 优化完成后，数组逐个被"拔掉"，最后只剩 `qep_tab`。

**被否决的方案**：函数式 / 不可变计划树。hypergraph 优化器恰恰走了这条路（`AccessPath` 一旦建好基本只读）。

**收益**：

- 内存 O(N) 而不是 O(搜索过的节点数)：`positions[]` 是**一条路径**的栈，不是 2^N 的 memo 表；
- 搜索可以就地改数组、`memcpy` 回退（`best_extension_by_limited_search` 用 `memcpy(saved_refs, ...)` 恢复），没有分配器压力；
- 大量代码可以按"约定顺序"读写同一字段，不必层层传参。

**代价**（这才是重点）：

- **同一字段在不同阶段语义不同**。典型是 `POSITION::filter_effect`：规划期它是"本表剩余谓词的过滤系数"，而进入 `make_join_readinfo` 之后它已经被折叠进 `rows_fetched`，不再是独立语义；
- **没有版本快照**。无法同时保留两个候选计划（这直接导致本篇后面讲的"五个数组只能逐个拔掉"）；
- **状态耦合**。写一个字段可能悄悄影响三个读者，所以源码里到处是"必须在 X 之后调用"的注释（例如 `set_plan_state()` 必须在 `make_tmp_tables_info()` 之后，因为它可能给首表加 sort）。

**为什么 MySQL 至今保留这套**：`JOIN` 承载的不只是优化器，还有执行器的状态（semi-join 策略、filesort、临时表）。把它拆成不可变树意味着执行器整体重写——8.0 的迭代器化改造只做了"物理计划树 + 迭代器树"这一层，`JOIN` 本身没动。

### 三数组分工：为什么一个表有三种表示

`JOIN_TAB`、`QEP_TAB`、`POSITION` 三者不是冗余，而是**三个不同时刻的快照**：

```
JOIN_TAB   : 规划期。回答"这张表有哪些访问方式可选、依赖谁"
POSITION   : 搜索期。回答"选定某个访问方式后，这个前缀的代价是多少"
QEP_TAB    : 执行期。回答"具体怎么读、要不要排序、要不要物化"
```

它们按"规划 → 搜索 → 执行"的时序存在，前一者的结论是后一者的输入。这就是 `QEP_TAB::init(JOIN_TAB *)` 存在的原因——把规划期的结论搬进执行期对象。

---

## 核心实现

### 一、四层对象全景

```
THD
 └─ LEX                          一次查询一个，持有 Query_expression 树
     └─ Query_expression (unit)  ← 一个 SELECT 语句（含 UNION / set-op）
         ├─ m_root_access_path   ← 整个 unit 的最终物理计划
         ├─ m_root_iterator      ← 整个 unit 的最终迭代器（★ 属于 unit，不属于 JOIN）
         └─ Query_block (QB)     ← 一个 SELECT 块
             ├─ join (JOIN*)     ← 该 QB 的优化器对象
             ├─ base_ref_items   ← SELECT list 的"原始切片"
             └─ 嵌套 Query_expression ← 子查询 / 派生表
```

**一个易错点**：`m_root_iterator` 与 `m_root_access_path` 在**两个层次上都有**：

- `Query_block` 的优化产物挂在 `JOIN::m_root_access_path`（在 `sql/sql_optimizer.h` 的 JOIN 类里）
- `Query_expression`（unit）持有最终的 `m_root_access_path` 与 `m_root_iterator`（在 `sql/sql_lex.h` 里），它是 `CreateIteratorFromAccessPath()` 的产物与归属者

也就是说：**迭代器树的所有者是 unit，不是 query block**。多表 JOIN 与 UNION 的迭代器都在 unit 层被组装（`sql_union.cc` 里 `m_root_iterator = CreateIteratorFromAccessPath(...)`）。

### 二、JOIN：字段分组剖析

> 定义于 `sql/sql_optimizer.h`。以下按"用途分组"而非声明顺序。

#### 2.1 表与 join 序

| 字段 | 类型 | 谁写 | 谁读 | 备注 |
|---|---|---|---|---|
| `tables` | `uint` | `init_planner_arrays` | 全流程 | **总数**，含 const 表、semi-join 物化表、临时表槽位 |
| `primary_tables` | `uint` | 同上 | 全流程 | 输入表数，含 semi-join 物化出的临时表 |
| `const_tables` | `uint` | `extract_const_tables` | 全流程 | 被判定为常量的表数 |
| `join_tab` | `JOIN_TAB*` | `init_planner_arrays` | `make_join_plan`、range 优化 | **tentative plan**，`get_best_combination()` 后**被置空** |
| `best_ref` | `JOIN_TAB**` | 同上，搜索期被重排 | `best_access_path`、`test_skip_sort` | 指针数组，**重排的是指针不是对象**；`make_tmp_tables_info` 后**被置空** |
| `map2table` | `JOIN_TAB**` | 同上 | name resolution、range | table_map 位序 → JOIN_TAB 的映射，**全程不变** |
| `qep_tab` | `QEP_TAB*` | `get_best_combination` | 执行准备、迭代器、EXPLAIN | 最终存活的数组 |

`tables` / `primary_tables` / `const_tables` 的关系是理解一切下标运算的前提：

```
索引区间（JOIN_TAB 数组的五段）：
[0, const_tables)                        常量表
[const_tables, primary_tables)           非常量的主表
[primary_tables, primary_tables+tmp_tables)  中间临时表（排序/去重）
... holes ...
semi-join 物化表
```

`plan_is_const()` 定义为 `const_tables == primary_tables`，`plan_is_single_table()` 定义为 `primary_tables - const_tables == 1` —— 这两个谓词在 [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) 里被反复使用。

**权威语义注释**（`sql/sql_optimizer.h:192`，这段注释是理解一切下标运算的钥匙）：

```cpp
  /**
    Before plan has been created, "tables" denote number of input tables in the
    query block and "primary_tables" is equal to "tables".
    After plan has been created (after JOIN::get_best_combination()),
    the JOIN_TAB objects are enumerated as follows:
    - "tables" gives the total number of allocated JOIN_TAB objects
    - "primary_tables" gives the number of input tables, including
      materialized temporary tables from semi-join operation.
    - "const_tables" are those tables among primary_tables that are detected
      to be constant.
    - "tmp_tables" is 0, 1 or 2 (more if windows) and counts the maximum
      possible number of intermediate tables in post-processing (ie sorting and
      duplicate removal).
      Later, tmp_tables will be adjusted to the correct number of
      intermediate tables, @see JOIN::make_tmp_tables_info.
    - The remaining tables (ie. tables - primary_tables - tmp_tables) are
      input tables to materialized semi-join operations.
    The tables are ordered as follows in the join_tab array:
     1. const primary table
     2. non-const primary tables
     3. intermediate sort/group tables
     4. possible holes in array
     5. semi-joined tables used with materialization strategy
  */
  uint tables{0};          ///< Total number of tables in query block
  uint primary_tables{0};  ///< Number of primary input tables in query block
  uint const_tables{0};    ///< Number of primary tables deemed constant
  uint tmp_tables{0};      ///< Number of temporary tables used by query
```

**四个数组指针的声明**（`sql/sql_optimizer.h:154`）：

```cpp
  /**
    Optimal query execution plan. Initialized with a tentative plan in
    JOIN::make_join_plan() and later replaced with the optimal plan in
    get_best_combination().
  */
  JOIN_TAB *join_tab{nullptr};
  /// Array of QEP_TABs
  QEP_TAB *qep_tab{nullptr};

  /**
    Array of plan operators representing the current (partial) best
    plan. The array is allocated in JOIN::make_join_plan() and is valid only
    inside this function. Initially (*best_ref[i]) == join_tab[i].
    The optimizer reorders best_ref.
  */
  JOIN_TAB **best_ref{nullptr};
  /// mapping between table indexes and JOIN_TABs
  JOIN_TAB **map2table{nullptr};

  /**
    This is the result of join optimization.

    @note This is a scratch array, not used after get_best_combination().
  */
  POSITION *best_positions{nullptr};

  /* Current join optimization state */
  POSITION *positions{nullptr};
```

> ⚠️ **`best_ref` 的注释是过时的**：它说 *"is valid only inside this function"（`make_join_plan`）*，但实际 `make_join_plan()` 之后的 `optimize_distinct_group_order()` / `test_skip_sort()` / `make_join_readinfo()` 都在读写它。**以调用点为准，不要被这条注释误导。**

#### 2.2 条件

| 字段 | 谁写 | 谁读 | 备注 |
|---|---|---|---|
| `where_cond` | `get_optimizable_conditions` → `optimize_cond` | `make_join_plan`、EXPLAIN | 注释写明"若没有表则执行期直接用；否则拆成片段挂到各 JOIN_TAB，之后不再用" |
| `having_cond` | 同上 | `make_tmp_tables_info`、执行期 | 可能被下推到临时表，此时置空 |
| `having_for_explain` | `make_tmp_tables_info`（下推前保存） | EXPLAIN | 下推导致 `having_cond` 置空，为 EXPLAIN 留副本 |
| `cond_equal` | `build_equal_items` | `substitute_for_best_equal_field` | 顶层等值类（见 [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md)） |

**`where_cond` 的权威注释**（`sql/sql_optimizer.h:495`）——注意它明确说了"有表时会被撕成片段挂到各 JOIN_TAB，之后不再用"：

```cpp
  /**
    Optimized WHERE clause item tree (valid for one single execution).
    Used in JOIN execution if no tables. Otherwise, attached in pieces to
    JOIN_TABs and then not used in JOIN execution.
    Printed by EXPLAIN EXTENDED.
    Initialized by Query_block::get_optimizable_conditions().
  */
  Item *where_cond;
  /**
    Optimized HAVING clause item tree (valid for one single execution).
    Used in JOIN execution, as last "row filtering" step. With one exception:
    may be pushed to the JOIN_TABs of temporary tables used in DISTINCT /
    GROUP BY (see JOIN::make_tmp_tables_info()); in that case having_cond is
    set to NULL, but is first saved to having_for_explain so that EXPLAIN
    EXTENDED can still print it.
    Initialized by Query_block::get_optimizable_conditions().
  */
  Item *having_cond;
  Item *having_for_explain;  ///< Saved optimized HAVING for EXPLAIN
```

**构造期的防御性初值**（`sql/sql_optimizer.cc:164`）——⚠️ **是 4 个成员，不是 2 个**：

```cpp
JOIN::JOIN(THD *thd_arg, Query_block *select)
    : query_block(select),
      thd(thd_arg),
      grouped(select->is_explicitly_grouped()),
      const_table_map(INNER_TABLE_BIT),
      found_const_table_map(INNER_TABLE_BIT),
      fields(&select->fields),
      tmp_table_param(thd_arg->mem_root),
      lock(thd->lock),
      implicit_grouping(select->is_implicitly_grouped()),
      select_distinct(select->is_distinct()),
      keyuse_array(thd->mem_root),
      order(select->order_list.first, ESC_ORDER_BY),
      group_list(select->group_list.first, ESC_GROUP_BY),
      m_windows(select->m_windows),
      /*
        Those four members are meaningless before JOIN::optimize(), so force a
        crash if they are used before that.
      */
      where_cond(reinterpret_cast<Item *>(1)),        // ①
      having_cond(reinterpret_cast<Item *>(1)),       // ②
      having_for_explain(reinterpret_cast<Item *>(1)),// ③
      tables_list(reinterpret_cast<Table_ref *>(1)),  // ④ ★ 这三个之外还有 tables_list
      current_ref_item_slice(REF_SLICE_SAVED_BASE),
      with_json_agg(select->json_agg_func_used()) {
  rollup_state = RollupState::NONE;
  if (select->order_list.first) explain_flags.set(ESC_ORDER_BY, ESP_EXISTS);
  if (select->group_list.first) explain_flags.set(ESC_GROUP_BY, ESP_EXISTS);
  if (select->is_distinct()) explain_flags.set(ESC_DISTINCT, ESP_EXISTS);
  if (m_windows.elements > 0) explain_flags.set(ESC_WINDOWING, ESP_EXISTS);
  // Calculate the number of groups
  for (ORDER *group = group_list.order; group; group = group->next)
    send_group_parts++;
}
```

> 地址 `0x1` 在用户态永远不可读，因此任何"在 `optimize()` 之前触碰这四个成员"的代码都会在 dereference 处**硬崩**——等价于断言但不依赖 `assert()`（release 版也生效）。真正的赋值发生在 `Query_block::get_optimizable_conditions()`。
>
> ★ 同一个构造函数里还顺带做了 **EXPLAIN 溯源的初始化**：`explain_flags.set(ESC_ORDER_BY, ESP_EXISTS)` 等四行。这就是"优化期还没开始，EXPLAIN 已经知道查询里有哪些子句"的原因。

#### 2.3 分组 / 排序 / 聚合

| 字段 | 谁写 | 备注 |
|---|---|---|
| `order` / `group_list` | 构造函数取 `select->order_list` / `group_list`；后续被 `optimize_distinct_group_order` 改写 | 类型是 `ORDER_with_src`，带溯源 |
| `simple_order` / `simple_group` | `remove_const` / `test_skip_sort` | 是否只涉及第一张非常量表 |
| `skip_sort_order` | `optimize_distinct_group_order` / `test_skip_sort` | 顺序已满足，无需排序 |
| `m_ordered_index_usage` | `test_skip_sort` | 三态枚举：`VOID` / `GROUP_BY` / `ORDER_BY` |
| `need_tmp_before_win` | `JOIN::optimize`、`test_skip_sort` | 窗口前必须先物化 |
| `streaming_aggregation` | `optimize_distinct_group_order`、`make_tmp_tables_info` | 流式聚合 vs 临时表聚合的开关 |
| `grouped` / `implicit_grouping` / `select_distinct` | 构造函数取 query_block 状态；后续被改写 | 显式分组 / 隐式分组（无 GROUP BY 但有聚合）/ DISTINCT |
| `m_windows` / `m_windows_sort` / `m_windowing_steps` | prepare 与 optimize | 窗口函数相关 |
| `sum_funcs` | `optimize` 收集 | 聚合函数指针数组 |
| `rollup_group_items` / `rollup_sums` | rollup 处理 | 见 [`../runtime/06_rollup.md`](../runtime/06_rollup.md) |
| `with_json_agg` | prepare | JSON 聚合函数会强制 filesort（见 15 篇第四章） |

#### 2.4 计划产物与执行期

| 字段 | 谁写 | 备注 |
|---|---|---|
| `m_root_access_path` | `create_access_paths()` | 该 QB 的 AccessPath 树根 |
| `m_root_access_path_no_in2exists` | `JOIN::optimize` | IN→EXISTS 转换产生的条件的"第二套计划"，用于 `EXPLAIN` 与某些执行分支 |
| `tmp_table_param` | 全程被改 | **母版**，每个真正的临时表会拷贝一份 |
| `tmp_fields` | `make_tmp_tables_info` | 按 `REF_SLICE_*` 分片的字段列表 |
| `hash_table_generation` | `clear_hash_tables` | 每次自增，告诉 HashJoinIterator 不能再用旧哈希表 |
| `filesorts_to_cleanup` | hypergraph 专用 | 旧优化器把 filesort 挂在 `QEP_TAB` 上，不需要这个数组 |
| `explain_flags` | EXPLAIN 收集阶段 | 供 `EXPLAIN` 输出 GROUP BY / ORDER BY / DISTINCT 的 QEP 细节 |
| `error` | `optimize` / `exec` | 错误码 |

#### 2.5 搜索期状态（`positions` / `best_positions`）

见第四章。

### 三、表的三种化身：JOIN_TAB / QEP_TAB / QEP_shared

```
                    QEP_shared_owner            （持有 QEP_shared，protected）
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
      JOIN_TAB                         QEP_TAB
  （规划期，sql/sql_select.h）    （执行期，sql/sql_executor.h）
      候选键 / 依赖 / 行数          op_type / filesort / semi-join 策略
      range 计划 / Key_use 引用     ref_item_slice / tmp_table_param
```

`QEP_shared` 存放两者共享的那一半，`QEP_shared_owner` 把它保护起来、只向 `JOIN_TAB` 与 `QEP_TAB` 暴露 getter/setter（源码注释明说了这个设计意图）。

#### 3.1 QEP_shared 的字段（共享部分）

| 字段 | 含义 | 谁写 |
|---|---|---|
| `m_join` | 回指 JOIN | 构造 |
| `m_idx` | 在数组中的下标；`NO_PLAN_IDX` 表示还没定 | `set_idx`，**一生只能设一次**（`assert(m_idx == NO_PLAN_IDX)`） |
| `m_table` | 对应的 TABLE（可能是内部临时表） | `set_table` |
| `m_position` | **指向 `best_positions` 数组的元素**，含代价信息 | `set_position` |
| `m_ref` | `Index_lookup`，ref 访问的键值绑定 | `best_access_path` |
| `m_index` / `m_type` | 选中的索引号 / `join_type`（`JT_REF` / `JT_ALL` / `JT_EQ_REF` …） | `best_access_path` |
| `m_condition` | 该表的条件（可引用前缀表与外层表） | `make_join_readinfo` / `finalize_table_conditions` |
| `m_keys` | `Key_map`：本表可用的索引位图 | `update_ref_and_keys` |
| `m_records` | 估算行数 | 统计 / range 估算 |
| `m_range_scan` | 该表的 range 计划（`AccessPath*`） | range 优化器 |
| `prefix_tables_map` / `added_tables_map` | 前缀可用表集合 / 本步新增表 | `set_prefix_tables` |
| `m_first_inner/m_last_inner/m_first_upper` | 外连接的边界下标 | `set_semijoin_embedding` |
| `m_first_sj_inner/m_last_sj_inner` | semi-join 的边界下标 | 同上 |
| `m_condition_is_pushed_to_sort` | 条件是否已被下推到排序前面 | `push_index_cond` / 排序相关 |

★ 注意 `m_position` 指向的是 **`best_positions`**（搜索结果数组）而不是 `positions`（当前搜索栈）。这是"搜索结束后结果被固化"的实现方式。

#### 3.1.1 `QEP_shared` 完整定义（`sql/sql_opt_exec_shared.h:236`）

```cpp
/// Holds members common to JOIN_TAB and QEP_TAB.
class QEP_shared {
 public:
  QEP_shared()
      : m_join(nullptr),
        m_idx(NO_PLAN_IDX),
        m_table(nullptr),
        m_position(nullptr),
        m_sj_mat_exec(nullptr),
        m_first_sj_inner(NO_PLAN_IDX),
        m_last_sj_inner(NO_PLAN_IDX),
        m_first_inner(NO_PLAN_IDX),
        m_last_inner(NO_PLAN_IDX),
        m_first_upper(NO_PLAN_IDX),
        m_ref(),
        m_index(0),
        m_type(JT_UNKNOWN),
        m_condition(nullptr),
        m_keys(),
        m_records(0),
        prefix_tables_map(0),
        added_tables_map(0),
        m_ft_func(nullptr),
        m_skip_records_in_range(false) {}

  /*
    Simple getters and setters. They are public. However, this object is
    protected in QEP_shared_owner, so only that class and its children
    (JOIN_TAB, QEP_TAB) can access the getters and setters.
  */
  // ★★ 封装意图就写在这句注释里：public 是"对 owner 家族 public"，
  //     对外被 QEP_shared_owner::m_qs 的 protected 挡住

  plan_idx idx() const {
    assert(m_idx >= 0);  // Index must be valid
    return m_idx;
  }
  void set_idx(plan_idx i) {
    assert(m_idx == NO_PLAN_IDX);  // Index should not change in lifetime
    m_idx = i;
  }                                // ★★ 一生只能设一次
  ...
  void set_prefix_tables(table_map prefix_tables_arg,
                         table_map prev_tables_arg) {
    prefix_tables_map = prefix_tables_arg;
    added_tables_map = prefix_tables_arg & ~prev_tables_arg;
  }

 private:
  JOIN *m_join;

  /**
     Index of structure in array:
     - NO_PLAN_IDX if before get_best_combination()
     - index of pointer to this JOIN_TAB, in JOIN::best_ref array
     - index of this QEP_TAB, in JOIN::qep array.
  */
  plan_idx m_idx;   // ★ 规划期语义 = best_ref 下标；执行期语义 = qep_tab 下标
                    //   数值相同但含义不同 —— 这正是"共享"的代价

  /// Corresponding table. Might be an internal temporary one.
  TABLE *m_table;

  /// Points into best_positions array. Includes cost info.
  POSITION *m_position;    // ★★ JOIN_TAB 与 QEP_TAB 靠共享它共享"该表代价/行数结论"

  /**
    Struct needed for materialization of semi-join. Set for a materialized
    temporary table, and NULL for all other join_tabs ...
  */
  Semijoin_mat_exec *m_sj_mat_exec;

  /**
    Boundaries of semijoin inner tables around this table. Valid only once
    final QEP has been chosen. ... last_sj_inner is not set for Duplicates
    Weedout.
  */
  plan_idx m_first_sj_inner, m_last_sj_inner;

  plan_idx m_first_inner;  ///< first inner table for including outer join
  plan_idx m_last_inner;   ///< last table table for embedding outer join
  plan_idx m_first_upper;  ///< first inner table for embedding outer join

  /**
     Used to do index-based look up based on a key value.
     Used when we read constant tables, in misc optimization (like
     remove_const()), and in execution.
  */
  Index_lookup m_ref;

  /// ID of index used for index scan or semijoin LooseScan
  uint m_index;

  /// Type of chosen access method (scan, etc).
  enum join_type m_type;

  /**
    Table condition, ie condition to be evaluated for a row from this table.
    Notice that the condition may refer to rows from previous tables in the
    join prefix, as well as outer tables.
  */
  Item *m_condition;

  /**
    Whether the condition in m_condition is evaluated in front of a sort,
    so that it does not need to be evaluated again ...
    Note that m_condition remains non-nullptr in this case, for purposes
    of the (non-tree) EXPLAIN and for filesort to build up its read maps.
  */
  bool m_condition_is_pushed_to_sort = false;

  /**
     All keys with can be used.
     Used by add_key_field() (optimization time) and execution of dynamic
     range (DynamicRangeIterator), and EXPLAIN.
  */
  Key_map m_keys;

  /**
     Either number of rows in the table or 1 for const table.
     Used in optimization, and also in execution for FOUND_ROWS().
  */
  ha_rows m_records;

  /**
     Non-NULL if quick-select used.
     Filled in optimization, converted to a RowIterator before execution
     (used to find rows), and in EXPLAIN.
  */
  AccessPath *m_range_scan = nullptr;

  /*
    Maps below are shared because of dynamic range: in execution, it needs to
    know the prefix tables, to find the possible QUICK methods.
  */
  table_map prefix_tables_map;   // ★ 动态 range 在执行期也要用，所以必须共享
  table_map added_tables_map;

  /** FT function */
  Item_func_match *m_ft_func;

  /**
    Set if index dive can be skipped for this query.
    See comments for check_skip_records_in_range_qualification.
  */
  bool m_skip_records_in_range;  // ★ 在 QEP_shared 里，不在 JOIN_TAB
};
```

```cpp
// sql/sql_opt_exec_shared.h:42 —— plan_idx 为什么是「有符号」
/**
   This represents the index of a JOIN_TAB/QEP_TAB in an array. "plan_idx":
   "Plan Table Index". It is signed, because:
   - firstmatch_return may be PRE_FIRST_PLAN_IDX (it can happen that the first
   table of the plan uses FirstMatch: SELECT ... WHERE literal IN (SELECT
   ...)).
   - it must hold the invalid value NO_PLAN_IDX ...; this invalid value must
   itself be different from PRE_FIRST_PLAN_IDX, to distinguish "FirstMatch to
   before-first-table" (firstmatch_return==PRE_FIRST_PLAN_IDX) from "No
   FirstMatch" (firstmatch_return==NO_PLAN_IDX).
*/
using plan_idx = int;
#define NO_PLAN_IDX (-2)  ///< undefined index
#define PRE_FIRST_PLAN_IDX \
  (-1)  ///< right before the first (first's index is 0)
```

#### 3.1.2 `QEP_shared_owner`：用继承代替 friend 的"半公开"封装（`sql/sql_opt_exec_shared.h:478`）

```cpp
/// Owner of a QEP_shared; parent of JOIN_TAB and QEP_TAB.
class QEP_shared_owner {
 public:
  QEP_shared_owner() : m_qs(nullptr) {}

  /// Instructs to share the QEP_shared with another owner
  void share_qs(QEP_shared_owner *other) { other->set_qs(m_qs); }
  // ★★ JOIN_TAB → QEP_TAB 的"共享而非拷贝"就发生在这三行
  void set_qs(QEP_shared *q) {
    assert(!m_qs);   // ★ 一个 owner 一生只能绑定一次 QEP_shared
    m_qs = q;
  }

  // Getters/setters forwarding to QEP_shared:
  JOIN *join() const { return m_qs ? m_qs->join() : nullptr; }  // ★ 唯一容忍 m_qs==nullptr 的
  void set_join(JOIN *j) { return m_qs->set_join(j); }

  plan_idx idx() const { return m_qs->idx(); }
  void set_idx(plan_idx i) { return m_qs->set_idx(i); }
  qep_tab_map idx_map() const {
    assert(m_qs->idx() < static_cast<plan_idx>(CHAR_BIT * sizeof(qep_tab_map)));
    return qep_tab_map{1} << m_qs->idx();
  }

  TABLE *table() const { return m_qs->table(); }
  POSITION *position() const { return m_qs->position(); }
  void set_position(POSITION *p) { return m_qs->set_position(p); }
  ...
  Key_map &keys() const { return m_qs->keys(); }
  ha_rows records() const { return m_qs->records(); }
  AccessPath *range_scan() const { return m_qs->range_scan(); }
  void set_range_scan(AccessPath *q) { return m_qs->set_range_scan(q); }
  table_map prefix_tables() const { return m_qs->prefix_tables(); }
  ...
  bool skip_records_in_range() const { return m_qs->skip_records_in_range(); }

  void qs_cleanup();

 protected:
  QEP_shared *m_qs;  // qs stands for Qep_Shared
  // ★★ protected 是封装的关键：外部拿不到 m_qs，只能用上面 40 多个转发接口
};
```

> ★ **`JOIN_TAB` / `QEP_TAB` 自己并没有 `idx()` / `type()` / `condition()` / `keys()` / `position()` / `ref()` 这些成员**——全部由 `QEP_shared_owner` 转发自 `QEP_shared`。这是读 `sql_optimizer.cc` 时最容易误判的一点：`tab->position()` 看起来像 JOIN_TAB 的字段，实际是两层转发。

#### 3.2 QEP_TAB 独有的字段（执行期）

| 字段 | 含义 |
|---|---|
| `table_ref` | `Table_ref*` |
| `op_type` | 枚举：`OT_NONE` / `OT_AGGREGATE` / `OT_AGGREGATE_INTO_TMP_TABLE` / `OT_MATERIALIZE` / `OT_WINDOW` … |
| `filesort` | 本表的 `Filesort*` |
| `tmp_table_param` | **本表的**临时表参数（从 JOIN 的母版拷贝） |
| `ref_item_slice` | 当前使用的 `ref_items` 切片（初值 `REF_SLICE_SAVED_BASE`） |
| `m_keyread_optim` | `set_keyread_optim()` 在 `set_plan_state()` 里快照 `table->key_read` |
| `m_reversed_access` | 是否反向扫描 |
| `flush_weedout_table` / `check_weed_out_table` | Duplicate Weedout 的临时表 |
| `firstmatch_return` / `loosescan_key_len` / `match_tab` | FirstMatch / LooseScan 的状态（详见 [`logical/03_semijoin.md`](logical/03_semijoin.md)） |
| `rematerialize` | 依赖表函数需要每次重新物化 |
| `having` | 下推到本临时表的 HAVING |
| `Setup_func` 枚举 | `NO_SETUP` / `MATERIALIZE_TABLE_FUNCTION` / `MATERIALIZE_DERIVED` / `MATERIALIZE_SEMIJOIN` |

`QEP_TAB::init(JOIN_TAB *jt)` 是两者的接口点：把规划期结论搬进执行期对象。

**为什么 QEP_TAB 只在旧优化器使用？** 因为它承载的是"表链 + 每表 op_type"的左深树模型，而 hypergraph 直接产出 AccessPath 树，没有"每表一个 QEP_TAB"的概念。

#### 3.3 `JOIN_TAB` 独有（规划期，`sql/sql_select.h:599`）

候选键（`const_keys` / `checked_keys` / `skip_scan_keys` / `needed_reg` / `quick_order_tested`）、依赖表集合（`dependent` / `key_dependent`）、`worst_seeks`、`found_records` / `read_time`、`emb_sj_nest` 等，服务于 `best_access_path` 的代价决策。

```cpp
class JOIN_TAB : public QEP_shared_owner {
 public:
  JOIN_TAB();

  void set_table(TABLE *t);          // 定义在 sql_select.cc:3495，双向绑定 reginfo.join_tab

  /// Sets the pointer to the join condition of Table_ref
  void init_join_cond_ref(Table_ref *tl);
  Item *join_cond() const { return *m_join_cond_ref; }
  /**
     Sets join condition
     @note this also changes Table_ref::m_join_cond.
  */
  void set_join_cond(Item *cond) { *m_join_cond_ref = cond; }

  /// Set the combined condition for a table (may be performed several times)
  void set_condition(Item *to) {
    if (condition() != to) {
      m_qs->set_condition(to);
      // Condition changes, so some indexes may become usable or not:
      quick_order_tested.clear_all();   // ★ 条件一变，range 分析的缓存失效
    }
  }

  Key_use *keyuse() const { return m_keyuse; }  // ★ 指向 JOIN::keyuse_array 的某处，不是独立数组
  void set_keyuse(Key_use *k) { m_keyuse = k; }

  Table_ref *table_ref; /**< points to table reference */

 private:
  Key_use *m_keyuse; /**< pointer to first used key */
  /**
    Pointer to the associated join condition:
    - if this is a table with position==NULL (e.g. internal sort/group
      temporary table), pointer is NULL
    - otherwise, pointer is the address of some Table_ref::m_join_cond.
      Thus, the pointee is the same as Table_ref::m_join_cond ...
  */
  Item **m_join_cond_ref;   // ★ 存"指针的指针"：写 JOIN_TAB 即写 Table_ref

 public:
  /**
    The maximum value for the cost of seek operations for key lookup
    during ref access. ... Without having this upper limit for the cost of
    ref access, table scan would be more likely to be chosen for cases
    where ref access performs better.
  */
  double worst_seeks;       // ★ ref 代价的"上限封顶"

  /** Keys with constant part. Subset of keys. */
  Key_map const_keys;       // ★ const_keys ⊆ keys()
  Key_map checked_keys;
  /**
    Keys which can be used for skip scan access. We store it
    separately from tab->const_keys & join_tab->keys() to avoid
    unnecessary printing of the prossible keys in EXPLAIN output ...
  */
  Key_map skip_scan_keys;
  Key_map needed_reg;

  /**
    Used to avoid repeated range analysis for the same key in
    test_if_skip_sort_order(). ... quick_order_tested is cleared every time
    the select condition for this JOIN_TAB changes ...
   */
  Key_map quick_order_tested;

  ha_rows found_records;   // 被"独立"访问方法扫描的行数
  double read_time;

  /**
    The set of tables that this table depends on. Used for outer join and
    straight join dependencies.
  */
  table_map dependent;
  /**
    The set of tables that are referenced by key from this table.
  */
  table_map key_dependent;

  /* SemiJoinDuplicateElimination variables: */
  Table_ref *emb_sj_nest;   // ★ 最近的 semijoin/antijoin nest
  nested_join_map embedding_map;
  ...
};
```

> ⚠️ **勘误**：`m_skip_records_in_range` **不在** `JOIN_TAB`，它在 `QEP_shared`（`sql_opt_exec_shared.h:474`），`JOIN_TAB` 通过 `QEP_shared_owner::skip_records_in_range()` 访问。

#### 3.4 `QEP_TAB` 完整定义（`sql/sql_executor.h:263`）

```cpp
class QEP_TAB : public QEP_shared_owner {
 public:
  QEP_TAB()
      : QEP_shared_owner(),
        table_ref(nullptr),
        flush_weedout_table(nullptr),
        check_weed_out_table(nullptr),
        firstmatch_return(NO_PLAN_IDX),
        loosescan_key_len(0),
        match_tab(NO_PLAN_IDX),
        rematerialize(false),
        not_used_in_distinct(false),
        having(nullptr),
        tmp_table_param(nullptr),
        filesort(nullptr),
        ref_item_slice(REF_SLICE_SAVED_BASE),   // ★ 初值是 SAVED_BASE(3)，不是 ACTIVE(0)
        m_keyread_optim(false),
        m_reversed_access(false),
        lateral_derived_tables_depend_on_me(0) {}

  /// Initializes the object from a JOIN_TAB
  void init(JOIN_TAB *jt);        // 定义在 sql/sql_optimizer.cc:1320
  void cleanup();                 // 定义在 sql/sql_select.cc:3520

  Item *condition_optim() const { return m_condition_optim; }
  void set_condition_optim() { m_condition_optim = condition(); }
  bool keyread_optim() const { return m_keyread_optim; }
  void set_keyread_optim() {
    if (table()) m_keyread_optim = table()->key_read;   // ★ 快照 TABLE::key_read
  }

  void set_table(TABLE *t) {
    m_qs->set_table(t);
    if (t) t->reginfo.qep_tab = this;      // ★★ 双向绑定
  }

  bool do_firstmatch() const { return firstmatch_return != NO_PLAN_IDX; }
  bool do_loosescan() const { return loosescan_key_len; }
  bool starts_weedout() const { return flush_weedout_table; }
  bool finishes_weedout() const { return check_weed_out_table; }

 public:
  Table_ref *table_ref;

  /* Variables for semi-join duplicate elimination */
  SJ_TMP_TABLE *flush_weedout_table;   // Weedout 起点：写 rowid
  SJ_TMP_TABLE *check_weed_out_table;  // Weedout 终点：查重

  /*
    If set, means we should stop join enumeration after we've got the first
    match and return to the specified join tab. May be PRE_FIRST_PLAN_IDX
    which means stopping join execution after the first match.
  */
  plan_idx firstmatch_return;   // ★ 三态：NO_PLAN_IDX(-2)=不做 / PRE_FIRST_PLAN_IDX(-1) / >=0

  /*
    Length of key tuple (depends on #keyparts used) to use for loose scan.
    If zero, means that loosescan is not used.
  */
  uint loosescan_key_len;
  plan_idx match_tab;

  enum Setup_func {
    NO_SETUP,
    MATERIALIZE_TABLE_FUNCTION,
    MATERIALIZE_DERIVED,
    MATERIALIZE_SEMIJOIN
  };
  Setup_func materialize_table = NO_SETUP;
  bool using_dynamic_range = false;

  /** true <=> remove duplicates on this table. */
  bool needs_duplicate_removal = false;

  Item *having;   // 下推到本临时表的 HAVING

  // Operation between the previous QEP_TAB and this one.
  enum enum_op_type {
    OT_NONE,                        // Regular nested loop.
    OT_AGGREGATE,                   // Aggregate (GROUP BY).
    OT_MATERIALIZE,                 // 以下四个是"末尾的临时表操作"
    OT_AGGREGATE_THEN_MATERIALIZE,
    OT_AGGREGATE_INTO_TMP_TABLE,
    OT_WINDOWING_FUNCTION,
    OT_BNL,                         // Block-nested loop (rewritten to hash join)
    OT_BKA                          // Batch key access
  } op_type = OT_NONE;

  /* Tmp table info */
  Temp_table_param *tmp_table_param;   // ★ 这份才是"被执行"的；JOIN::tmp_table_param 只是母版

  /* Sorting related info */
  Filesort *filesort;

  /**
    If we pushed a global ORDER BY down onto this first table, that ORDER BY
    list will be preserved here.
   */
  ORDER *filesort_pushed_order = nullptr;

  uint ref_item_slice;             // 读本表前要切到哪一片
  Item *m_condition_optim = nullptr;   // 优化器定下的条件快照（供 EXPLAIN）
  bool m_keyread_optim;            // "只走索引"决策快照
  bool m_reversed_access;

  qep_tab_map lateral_derived_tables_depend_on_me;

  Mem_root_array<const AccessPath *> *invalidators = nullptr;
};
```

**`QEP_TAB::init()`**（`sql/sql_optimizer.cc:1320`）——只有三行，但每一行都是关键：

```cpp
void QEP_TAB::init(JOIN_TAB *jt) {
  jt->share_qs(this);      // ① JOIN_TAB 把自己的 QEP_shared 指针共享给 QEP_TAB
  set_table(table());      // ② 只为回写 table()->reginfo.qep_tab = this
  table_ref = jt->table_ref;   // ③
}
```

**`JOIN_TAB::set_table()` vs `QEP_TAB::set_table()`**（两条双向绑定，槽位不同）：

```cpp
// sql/sql_select.cc:3495
void JOIN_TAB::set_table(TABLE *t) {
  if (t != nullptr) t->reginfo.join_tab = this;   // ★ 先绑
  m_qs->set_table(t);                             //   后设
}

// sql/sql_executor.h:299
void QEP_TAB::set_table(TABLE *t) {
  m_qs->set_table(t);                             // ★ 先设
  if (t) t->reginfo.qep_tab = this;               //   后绑
}
```

> ★ `TABLE::reginfo` **同时有 `join_tab` 与 `qep_tab` 两个槽位**——这就是"一表两身份"的物理体现。两者顺序相反，但都指向同一个 `QEP_shared`。

**`QEP_TAB::cleanup()`**（`sql/sql_select.cc:3520`）：

```cpp
void QEP_TAB::cleanup() {
  // Delete parts specific of QEP_TAB:
  destroy(filesort);
  filesort = nullptr;

  TABLE *const t = table();

  if (t != nullptr) {
    t->reginfo.qep_tab = nullptr;   // ★ 解除反向绑定
    t->const_table = false;  // Note: Also done in TABLE::init()
  }

  // Delete shared parts:
  qs_cleanup();

  // Order of qs_cleanup() and this, matters:
  if (op_type == QEP_TAB::OT_MATERIALIZE ||
      op_type == QEP_TAB::OT_AGGREGATE_THEN_MATERIALIZE ||
      op_type == QEP_TAB::OT_AGGREGATE_INTO_TMP_TABLE ||
      op_type == QEP_TAB::OT_WINDOWING_FUNCTION) {
    if (t != nullptr)  // Check tmp table is not yet freed.
    {
      close_tmp_table(t);
      free_tmp_table(t);
    }
    destroy(tmp_table_param);
    tmp_table_param = nullptr;
  }
  if (table_ref != nullptr && table_ref->uses_materialization()) {
    assert(t == table_ref->table);
    t->merge_keys.clear_all();
    t->quick_keys.clear_all();
    t->covering_keys.clear_all();
    t->possible_quick_keys.clear_all();

    close_tmp_table(t);
  }
}
```

#### 3.5 `POSITION`：必须是 POD（`sql/sql_select.h:352`）

```cpp
/**
  A position of table within a join order. This structure is primarily used
  as a part of @c join->positions and @c join->best_positions arrays.
  ...
   - Semi-join strategy choice. Note that there are two different
     representation formats:
      1. The one used during join optimization
      2. The one used at plan refinement/code generation stage.
      We call fix_semijoin_strategies_for_picked_join_order() to switch
      between #1 and #2.
  ...
  This class has to stay a POD, because it is memcpy'd in many places.
*/
struct POSITION {                       // ★ 是 struct，且注释明说必须保持 POD
  /**
    The number of rows that will be fetched by the chosen access
    method per each row combination of previous tables. ...
  */
  double rows_fetched;

  /**
    Cost of accessing the table in course of the entire complete join
    execution ... read_cost does NOT include cost of processing rows within
    the executor (row_evaluate_cost).
  */
  double read_cost;

  /**
    The fraction of the 'rows_fetched' rows that will pass the table
    conditions that were NOT used by the access method. ...
    @note With condition filtering enabled, it is possible to get
    a fanout = rows_fetched * filter_effect that is less than 1.0. ...
  */
  float filter_effect;                  // ★ 是 float 不是 double

  /**
    prefix_rowcount: The number of row combinations that will be
    joined to the next table in the join sequence.
    For a joined table it is calculated as
      prefix_rowcount =
          last_table.prefix_rowcount * rows_fetched * filter_effect
  */
  double prefix_rowcount;
  double prefix_cost;

  JOIN_TAB *table;

  /**
    NULL  -  'index' or 'range' or 'index_merge' or 'ALL' access is used.
    Other - [eq_]ref[_or_null] access is used. Pointer to {t.keypart1 = expr}
  */
  Key_use *key;                         // ★ 非 NULL ⇔ 走 ref 类访问

  table_map ref_depend_map;
  bool use_join_buffer;

  /**
    Join optimizer sets this for the *last* join_tab in the
    duplicate-generating range. That is, in order to interpret this field,
    one needs to traverse join->[best_]positions array from right to left.
    When you see a join table with sj_strategy!= SJ_OPT_NONE, some other
    field (depending on the strategy) tells how many preceding positions
    this applies to. The values of covered_preceding_positions->sj_strategy
    must be ignored.
  */
  uint sj_strategy;                     // ★★ "从右往左读"语义
  uint n_sj_tables;
  table_map dups_producing_tables;

  /* LooseScan / FirstMatch / Duplicate Weedout / SJ-Materialization-Scan
     各自的一组状态字段（first_loosescan_table、loosescan_key、
     first_firstmatch_table、first_dupsweedout_table、sjm_scan_last_inner ...） */

  void set_prefix_join_cost(uint idx, const Cost_model_server *cm) {
    if (idx == 0) {
      prefix_rowcount = rows_fetched;
      prefix_cost = read_cost + cm->row_evaluate_cost(prefix_rowcount);
    } else {
      prefix_rowcount = (this - 1)->prefix_rowcount * rows_fetched;
      prefix_cost = (this - 1)->prefix_cost + read_cost +
                    cm->row_evaluate_cost(prefix_rowcount);
    }
    prefix_rowcount *= filter_effect;   // ★ 先算 join 行数，再乘过滤系数
  }

 private:
  table_map m_suffix_lateral_deps;
};
```

> ★ **"必须保持 POD"这条约束解释了整个搜索算法的实现方式**：`positions[]` 作为栈被 `memcpy` 备份/回退（`best_extension_by_limited_search`），`POSITION` 一旦有了虚函数或非平凡成员，这套 memcpy 就失效了。

#### 3.6 `Key_use`：**是 class 不是 struct**（`sql/sql_select.h:174`）

```cpp
class Key_use {
 public:
  // We need the default constructor for unit testing.
  Key_use()
      : table_ref(nullptr), val(nullptr), used_tables(0), key(0), keypart(0),
        optimize(0), keypart_map(0), ref_table_rows(0), null_rejecting(false),
        cond_guard(nullptr), sj_pred_no(UINT_MAX), bound_keyparts(0),
        fanout(0.0), read_cost(0.0) {}
  ...
  Table_ref *table_ref;  ///< table owning the index
  /**
    Value used for lookup into @c key. It may be an Item_field, a
    constant or any other expression. If @c val contains a field from
    another table, then we have a join condition, and the table(s) of
    the field(s) in @c val should be before @c table in the join plan.
  */
  Item *val;                     // ★★ join 序约束的来源
  /**
    All tables used in @c val, that is all tables that provide bindings
    for the expression @c val. These tables must be in the plan before
    executing the equi-join described by a Key_use. ...
  */
  table_map used_tables;
  uint key;                  ///< number of index
  uint keypart;              ///< used part of the index
  uint optimize;             ///< 0, or KEY_OPTIMIZE_*
  key_part_map keypart_map;
  ha_rows ref_table_rows;    ///< Estimate of how many rows for a key value
  /**
    If true, the comparison this value was created from will not be
    satisfied if val has NULL 'value' ...
  */
  bool null_rejecting;       // ★ 据此可推导 NOT NULL，触发外连接简化
  /**
    !NULL - This Key_use was created from an equality that was wrapped into
            an Item_func_trig_cond. This means the equality (and validity of
            this Key_use element) can be turned on and off. ...
  */
  bool *cond_guard;
  /**
     0..63    @<=@> This was created from semi-join IN-equality # sj_pred_no.
     UINT_MAX  Otherwise
  */
  uint sj_pred_no;

  /*
    The three members below are different from the rest of Key_use: they are
    set only by Optimize_table_order, and they change with the currently
    considered join prefix.
  */
  // ★★ 前 10 个字段"静态"（只依赖 SQL 文本），后 3 个"随前缀动态变化"
  key_part_map bound_keyparts;
  double fanout;
  double read_cost;
};
```

> ★ 最后那句注释是理解 `optimize_keyuse()` 的关键：**`Key_use` 里有一半字段是"缓存"，每次 join 前缀变化都要重算**——这就是为什么 `best_access_path` 里能直接读 `keyuse->read_cost` 而不必重新估算。

### 四、五个数组：谁分配、谁重排、谁被置空

这是本篇最实用的一节。六个指针的生命周期（**全部实测，括号内是确切行号**）：

| 指针 | 分配于 | 重排/改写 | 被置空于 | 真·存活区间 |
|---|---|---|---|---|
| `join_tab` | `init_planner_arrays`（`sql_optimizer.cc:5444`） | 不重排（重排的是 `best_ref` 里的指针） | **`get_best_combination`（:3261）** | `make_join_plan` 开头 ~ `get_best_combination` 中段 |
| `best_ref` | 同上（:5452） | **搜索时被反复重排/回退**；`get_best_combination` 里再按 `best_positions` 重排一次 | **从不置空** | 规划期 ~ `JOIN::destroy` |
| `map2table` | 同上（:5458） | 只在 `get_best_combination`（:3235）写入一次 | 从不置空 | 全程 |
| `positions` | 同上（:5462） | 搜索时作为**栈**被写入 | **`make_join_plan`（:5405）** | 搜索期 |
| `best_positions` | 同上（:5464） | `consider_plan` 回写最优前缀 | **`make_join_plan`（:5414）** | 搜索期 ~ `get_best_combination` |
| `qep_tab` | **`alloc_qep`（:1313，由 `optimize():1011` 调用）** | 不再重排 | **从不置空**（只逐个 `cleanup()`） | 执行准备 ~ `JOIN::destroy` |

> ⚠️ **三处勘误（原表格与源码不符）**：
> 1. **`best_ref` 从不被置空**。它一直活到 `JOIN::destroy()`，后者只遍历它调用 `tab->cleanup()`，**不置 nullptr、不释放**（内存靠 mem_root 回收）。"make_tmp_tables_info 之后被置空"是错的。
> 2. **`qep_tab` 不是 `get_best_combination()` 分配的**。它由 `JOIN::alloc_qep()`（`sql_optimizer.cc:1308`）分配，调用点 `optimize():1011`，**晚于** `get_best_combination()`（后者在 `make_join_plan():5408` 内部被调）。
> 3. **`QEP_TAB::init()` 不由 `get_best_combination()` 调用**，只由 `alloc_qep()`（:1316）调用。

**`init_planner_arrays()` 完整源码**（`sql/sql_optimizer.cc:5437`）：

```cpp
bool JOIN::init_planner_arrays() {
  // Up to one extra slot per semi-join nest is needed (if materialized)
  const uint sj_nests = query_block->sj_nests.size();
  const uint table_count = query_block->leaf_table_count;

  assert(primary_tables == 0 && tables == 0);

  if (!(join_tab = alloc_jtab_array(thd, table_count))) return true;   // ①

  /*
    We add 2 cells:
    - because planning stage uses 0-termination so needs +1
    - because after get_best_combination, we don't use 0-termination but
    need +2, to host at most 2 tmp sort/group/distinct tables.
  */
  if (!(best_ref = (JOIN_TAB **)thd->alloc(
            sizeof(JOIN_TAB *) *
            (table_count + sj_nests + 2 + m_windows.elements))))       // ②
    return true;

  // sort/group tmp tables have no map
  if (!(map2table = (JOIN_TAB **)thd->alloc(sizeof(JOIN_TAB *) *
                                            (table_count + sj_nests))))
    return true;                                                       // ③

  if (!(positions = new (thd->mem_root) POSITION[table_count]))         // ④
    return true;

  if (!(best_positions = new (thd->mem_root) POSITION[table_count + sj_nests]))
    return true;                                                       // ⑤

  /*
    Initialize data structures for tables to be joined.
    Initialize dependencies between tables.
  */
  JOIN_TAB **best_ref_p = best_ref;
  Table_ref *tl = query_block->leaf_tables;

  for (JOIN_TAB *tab = join_tab; tl; tab++, tl = tl->next_leaf, best_ref_p++) {
    *best_ref_p = tab;              // ★ 初始时 (*best_ref[i]) == join_tab[i]
    TABLE *const table = tl->table;
    tab->table_ref = tl;
    tab->set_table(table);
    const int err = tl->fetch_number_of_rows();
    if (err) {
      table->file->print_error(err, MYF(0));
      return true;
    }
    table->init_cost_model(cost_model());

    all_table_map |= tl->map();
    tab->set_join(this);

    if (tl->is_updated() || tl->is_deleted()) {
      // As we update or delete rows, we can't read the index
      table->no_keyread = true;
    }

    tab->dependent = tl->dep_tables;  // Initialize table dependencies
    if (query_block->is_recursive()) {
      if (query_block->recursive_reference != tl)
        tab->dependent |= query_block->recursive_reference->map();
      else {
        // Recursive reference mustn't use any index
        table->covering_keys.clear_all();
        table->keys_in_use_for_group_by.clear_all();
        table->keys_in_use_for_order_by.clear_all();
      }
    }
    if (tl->schema_table) table->file->stats.records = 2;   // ★ I_S 表硬编码 2 行
    table->quick_condition_rows = table->file->stats.records;

    tab->init_join_cond_ref(tl);

    if (tl->outer_join_nest()) {
      tab->embedding_map = 0;
      for (Table_ref *embedding = tl->embedding; embedding;
           embedding = embedding->embedding) {
        NESTED_JOIN *const nested_join = embedding->nested_join;
        tab->embedding_map |= nested_join->nj_map;
        tab->dependent |= embedding->dep_tables;
      }
    } else if (tab->join_cond()) { ... }

    if (tl->is_derived() && tl->derived_query_expression()->m_lateral_deps)
      has_lateral = true;

    tables++;  // Count number of initialized tables
  }

  primary_tables = tables;
  *best_ref_p = nullptr;  // Last element of array must be NULL

  return false;
}
```

> ★ **`alloc_jtab_array()` 是"共享 QEP_shared"的根源**（`sql_optimizer.cc:3028`）——它同时分配**一对平行数组** `JOIN_TAB[]` 与 `QEP_shared[]`，然后 `t[i].set_qs(qs + i)`：

```cpp
static JOIN_TAB *alloc_jtab_array(THD *thd, uint table_count) {
  JOIN_TAB *t = new (thd->mem_root) JOIN_TAB[table_count];
  if (!t) return nullptr;

  QEP_shared *qs = new (thd->mem_root) QEP_shared[table_count];
  if (!qs) return nullptr;

  for (uint i = 0; i < table_count; ++i) t[i].set_qs(qs++);

  return t;
}
```

**`make_join_plan()` 里两个"退役"点**（`:5405` / `:5414`）：

```cpp
  refine_best_rowcount();

  positions = nullptr;  // But keep best_positions for get_best_combination

  // Generate an execution plan from the found optimal join order.
  if (get_best_combination()) return true;

  // Cleanup after update_ref_and_keys has added keys for derived tables.
  if (query_block->materialized_derived_table_count) finalize_derived_keys();

  // No need for this struct after new JOIN_TAB array is set up.
  best_positions = nullptr;
```

**`get_best_combination()` 的关键片段**（`:3070`，★ `join_tab = nullptr` 的真实位置）：

```cpp
  // To check that we fill the array correctly: fill it with zeros first
  memset(best_ref, 0,
         sizeof(JOIN_TAB *) * (primary_tables + sjm_nests + num_tmp_tables));
  ...
  for (uint tableno = 0; tableno < tables; tableno++) {
    POSITION *const pos = best_positions + tableno;   // ★ 读搜索结果
    ...
    const uint target =
        (remaining_sjm_inner--) > 0 ? inner_target++ : outer_target++;
    JOIN_TAB *const tab = pos->table;

    best_ref[target] = tab;      // ★★ 重排：SJM 内表放尾部，外表放头部
    tab->set_idx(target);        // ★ set_idx 的"一生一次"在这里兑现
    tab->set_position(pos);      // ★★ QEP_shared::m_position := best_positions + i
    ...
    map2table[tab->table_ref->tableno()] = tab;
  }

  tables += sjm_nests + num_tmp_tables;
  primary_tables = outer_target;
  ...
  // make array unreachable: should walk JOIN_TABs by best_ref now
  join_tab = nullptr;            // ★★ 交棒点（:3261）
```

```cpp
// sql/sql_optimizer.cc:1308 —— qep_tab 的真正诞生地
bool JOIN::alloc_qep(uint n) {
  static_assert(MAX_TABLES <= INT_MAX8, "plan_idx needs to be wide enough.");

  ASSERT_BEST_REF_IN_JOIN_ORDER(this);

  qep_tab = new (thd->mem_root)
      QEP_TAB[n + 1];        // The last one holds only the final op_type.
  if (!qep_tab) return true;
  for (uint i = 0; i < n; ++i) qep_tab[i].init(best_ref[i]);   // ★ QEP_TAB::init 唯一调用点
  return false;
}
```

> ★ `qep_tab` 数组长度是 `n + 1`：**最后那个多余的槽位只用来放"最终 op_type"**（`qep_tab[primary_tables + tmp_tables].op_type = get_end_select_func()`，见 [`sql_executor.cc:574`](../runtime/04_window_function.md) 相关讨论与 `sql_select.cc:4914`）。这是"表链 + 末尾一个收尾操作"模型的直接体现。

声明与源码注释（`sql/sql_optimizer.h`）：

```cpp
  /**
    Optimal query execution plan. Initialized with a tentative plan in
    JOIN::make_join_plan() and later replaced with the optimal plan in
    get_best_combination().
  */
  JOIN_TAB *join_tab{nullptr};
  /// Array of QEP_TABs
  QEP_TAB *qep_tab{nullptr};

  /**
    Array of plan operators representing the current (partial) best
    plan. The array is allocated in JOIN::make_join_plan() and is valid only
    inside this function. Initially (*best_ref[i]) == join_tab[i].
    The optimizer reorders best_ref.
  */
  JOIN_TAB **best_ref{nullptr};
  /// mapping between table indexes and JOIN_TABs
  JOIN_TAB **map2table{nullptr};

  /**
    This is the result of join optimization.

    @note This is a scratch array, not used after get_best_combination().
  */
  POSITION *best_positions{nullptr};

  /* Current join optimization state */
  POSITION *positions{nullptr};
  /******* Join optimization state members end *******/
```

★ `best_ref` 的注释与上表**不一致**，值得留意：注释说它 *"is valid only inside this function"（`make_join_plan`）*，但上表给出的存活区间是"规划期 ~ 计划改进早期"——因为计划改进（`make_join_plan` 之后的若干步骤）仍在读写它。**注释是过时的，实际行为以调用点为准**；读代码时不要被这条注释误导，以为出了 `make_join_plan` 就没人碰 `best_ref`。

另外两条注释与表格互为印证：`best_positions` 明说"scratch array，`get_best_combination()` 之后不再用"——但 `QEP_shared::m_position` 仍指向它（这正是上表最后一列的隐患）；`positions` 的注释说它是**栈**（"we also maintain a stack of join optimization states in join->positions[]"），对应搜索期的前缀回溯。

由此得出两条读代码时的经验规则：

1. **看到 `best_ref[i]` 就说明还在规划期或计划改进早期**；看到 `qep_tab[i]` 就说明已进入执行准备期。两者不会同时出现在同一个函数里（所以有 `ASSERT_BEST_REF_IN_JOIN_ORDER` 这类断言宏）。
2. **`POSITION*` 只在"读代价"时出现**，因为它的唯一用途是记录前缀代价与 semi-join 策略。`tab->position()->sj_strategy` 这种写法在计划改进期仍然有效，就是因为 `best_positions` 还没释放。

### 五、Temp_table_param：临时表的参数母版

> 定义于 `sql/temp_table_param.h`。

`JOIN::tmp_table_param` 是一份**母版**：源码注释写得很清楚——"每个临时表有它自己的 `tmp_table_param`，这里的那个是被 `create_intermediate_table()` 当作模板临时使用的"。

关键字段：

| 字段 | 含义 | 谁写 |
|---|---|---|
| `sum_func_count` | 聚合函数个数 | `count_field_types` |
| `group_parts` / `group_length` | 分组键的列数与字节数 | `calc_group_buffer` |
| `allow_group_via_temp_table` | 能否用临时表唯一键做增量聚合（**默认 true**） | 见 [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) 第六章 |
| `precomputed_group_by` | LIS 是否已预计算分组 | `make_tmp_tables_info`（读 `is_agg_loose_index_scan`） |
| `hidden_field_count` / `func_count` | 隐藏列（GROUP BY 列、聚合列）计数 | `count_field_types` |

字段声明与源码注释（`sql/temp_table_param.h`），三条注释都有表格里没有的信息：

```cpp
  uint func_count;
  /**
    Number of fields in the query that have aggregate functions. Note
    that the optimizer may choose to optimize away these fields by
    replacing them with constants, in which case sum_func_count will
    need to be updated.
    @see optimize_aggregated_query, count_field_types
  */
  uint sum_func_count;
  uint hidden_field_count;
  uint group_parts, group_length, group_null_parts;
  /**
    Whether we allow running GROUP BY processing into a temporary table,
    i.e., keeping many different aggregations going at once without
    having ordered input. This is usually the case, but is currently not
    supported for aggregation UDFs, aggregates with DISTINCT, or ROLLUP.

    Note that even if this is true, the optimizer may choose to not use
    a temporary table, as it is often more efficient to just read along
    an index.
   */
  bool allow_group_via_temp_table{true};
```

1. **`sum_func_count` 会被"优化掉"**：聚合字段可能被优化器替换成常量，届时这个计数必须更新——所以它是个**会被回写**的字段，不是一次算完就不变。
2. ★ **`allow_group_via_temp_table` 默认 true，但注释点名三个不支持的场景**：aggregation UDF、带 `DISTINCT` 的聚合、`ROLLUP`。
3. ★ **true 也不等于一定走临时表**：注释后半句明确说，即使允许，优化器也可能**沿索引直接读**而不落临时表——因为那往往更高效。⇒ 看这个字段判断"是否用临时表"会误判，它只是**许可**，不是决定。

（`func_type` 位图也值得一提，虽不在上表：它用 `CFT_*` 枚举位标记 `m_func` 的类型，让 `copy_funcs()` 能**不经过虚函数调用**就快速判断哪些 item 需要复制。）
| `end_write_records` | 写入方式 | `make_tmp_tables_info` |
| `m_window` / rollup 相关 | 窗口与 rollup 的临时表形态 | 窗口/rollup 处理 |

**拷贝构造函数**（`sql/temp_table_param.h:234`）——"母版 + 拷贝"模式的基础：

```cpp
  Temp_table_param(MEM_ROOT *mem_root, const Temp_table_param &other)
      : copy_fields(mem_root),                     // ★ 新建空容器，不是拷贝
        group_buff(other.group_buff),
        items_to_copy(other.items_to_copy),
        keyinfo(other.keyinfo),
        end_write_records(other.end_write_records),
        func_count(other.func_count),
        sum_func_count(other.sum_func_count),
        hidden_field_count(other.hidden_field_count),
        group_parts(other.group_parts),
        group_length(other.group_length),
        group_null_parts(other.group_null_parts),
        allow_group_via_temp_table(other.allow_group_via_temp_table),  // ★ 透传
        outer_sum_func_count(other.outer_sum_func_count),
        using_outer_summary_function(other.using_outer_summary_function),
        table_charset(other.table_charset),
        schema_table(other.schema_table),
        precomputed_group_by(other.precomputed_group_by),              // ★ 透传
        force_copy_fields(other.force_copy_fields),
        skip_create_table(other.skip_create_table),
        bit_fields_as_long(other.bit_fields_as_long),
        can_use_pk_for_unique(other.can_use_pk_for_unique),
        force_hash_field_for_unique(other.force_hash_field_for_unique),
        m_window_frame_buffer(other.m_window_frame_buffer),
        m_window(other.m_window) {}

  void cleanup() { copy_fields.clear(); }
```

> ★ **两个细节**：
> 1. `copy_fields` 是**新建空容器**而不是拷贝——因为 `Copy_field` 内部持有指向源/目的 `Field*` 的指针，跨临时表实例不能复制。
> 2. 成员 `m_last_operation_is_distinct`（用于 INTERSECT/EXCEPT）在拷贝构造中**未被透传**——默认初始化为 `false`。这是一个已知的不对称。

**为什么需要"母版 + 拷贝"**：一个查询可能有多个临时表（DISTINCT 一个、GROUP BY 一个、窗口每个窗口一个），它们共享大部分参数但各自有差异。拷贝母版再改是成本最低的做法。

### 六、ORDER_with_src 与 ref_items 切片

#### 6.1 ORDER_with_src

```
ORDER_with_src
 ├─ ORDER *order            ← 排序项链表
 ├─ enum_src src            ← ESC_ORDER_BY / ESC_GROUP_BY / ESC_DISTINCT
 └─ bool const_optimized    ← 是否已被 remove_const 处理过
```

溯源标签存在的唯一理由是**可观测性**：排序项在优化期会被消、被改写、被合并（DISTINCT 会被改写成 GROUP BY），若不记来源，EXPLAIN 就无法解释"这个 filesort 是谁要求的"。

代价：每次改写都要手工带上 `src`，漏传即丢溯源。源码里 `ORDER_with_src(o, ESC_DISTINCT)`、`ORDER_with_src(..., group_list.src, /*const_optimized=*/true)` 这类写法遍布。

#### 6.2 ref_items 切片

`JOIN::ref_items` 是 `Ref_item_array` 数组，按 `REF_SLICE_*` 分片，每片是"SELECT list 在某个执行阶段的表达式列表"：

| 切片 | 用途 |
|---|---|
| `REF_SLICE_ACTIVE`（0） | 当前生效的那一片（可切换） |
| `REF_SLICE_SAVED_BASE` | 原始 SELECT list 的备份 |
| `REF_SLICE_TMP1` / `REF_SLICE_TMP2` | 第一/第二个临时表之后的版本 |
| `REF_SLICE_WIN_1 + N` | 每个窗口函数两片（输入/输出） |

**为什么需要多份**：写临时表时，`SUM(x)` 的值被物化成了临时表的一列；之后要"读回"这个物化值，就需要一个新的 `Item_field`（指向临时表列）来替换原来的 `Item_sum`。所以每个物化阶段都要生成一套新表达式。

切换切片的惯用法是 RAII：

```cpp
Switch_ref_item_slice(join, new_slice);   // 构造时切过去，析构时切回来
```

**切片的完整枚举**（`sql/sql_opt_exec_shared.h:583`；★ 是**匿名 enum**，不存在名为 `enum_ref_item_slice` 的类型）：

```cpp
/**
  Symbolic slice numbers into JOIN's arrays ref_items, tmp_fields and
  tmp_all_fields
  See also the comments on JOIN::ref_items.
*/
enum {
  /**
     The slice which is used during evaluation of expressions; Item_ref::ref
     points there. This is the only slice that is not allocated on the heap;
     it always points to query_block->base_ref_items.
     ...
     Assuming no index on t1, this will require creating a temporary table
     consisting only of REVERSE(t1), and then sorting it before grouping.
     During execution of the query creating the temporary table, we will
     have an Item_func_reverse pointing to a Field for t1 ... However, when
     reading from "tmp", it would be wrong to use that Item_func_reverse, as
     the Field no longer exists. Thus, we create a slice (in REF_SLICE_TMP1)
     with new Item pointers, where Item_func_reverse is replaced by an
     Item_field that reads from the right field in the temporary table.
     ...
     In such cases, the pointers in this slice are _overwritten_ (using memcpy)
     by e.g. REF_SLICE_TMP1 for as long as we read from the temporary table.
     Switch_ref_item_slice provides an encapsulation of the overwriting,
     and the optimizer stores a copy of the original Item pointers in the
     REF_SLICE_SAVED_BASE slice so that it is possible to copy them back
     when we are done.

     @todo It would probably be better to store the active slice index in
     current_thd and do the indirection in Item_ref_* instead of copying the
     slices around.
  */
  REF_SLICE_ACTIVE = 0,
  /// The slice with pointers to columns of 1st group-order-distinct tmp table
  REF_SLICE_TMP1,
  /// The slice with pointers to columns of 2nd group-order-distinct tmp table
  REF_SLICE_TMP2,
  /**
     The slice with pointers to columns of table(s), ie., the actual Items.
     Only used for queries involving temporary tables or the likes; for simple
     queries, they always live in REF_SLICE_ACTIVE, so we don't need a copy
     here.
  */
  REF_SLICE_SAVED_BASE,
  /// The slice with pointers to columns of 1st tmp table of windowing
  REF_SLICE_WIN_1     // ★ 没有 WIN_2：多窗口用 WIN_1 + n 的算术下标
};
```

```cpp
// sql/sql_optimizer.h:1091 —— RAII 类（全文只有 8 行）
class Switch_ref_item_slice {
  JOIN *join;
  uint saved;

 public:
  Switch_ref_item_slice(JOIN *join_arg, uint new_v)
      : join(join_arg), saved(join->get_ref_item_slice()) {
    if (!join->ref_items[new_v].is_null()) join->set_ref_item_slice(new_v);
    // ★ 守卫：目标 slice 未分配则静默不切换（析构时也不会破坏状态）
  }
  ~Switch_ref_item_slice() { join->set_ref_item_slice(saved); }
};
```

★ **切换切片的真实语义是 `memcpy`，不是改指针**（`sql_optimizer.h:669`）：

```cpp
  void set_ref_item_slice(uint sliceno) {
    assert((int)sliceno >= 1);
    if (current_ref_item_slice != sliceno) {
      copy_ref_item_slice(REF_SLICE_ACTIVE, sliceno);
      // ★★ 把 ref_items[N] 的内容 memcpy 进 ref_items[0](ACTIVE)
      current_ref_item_slice = sliceno;
    }
  }

  void copy_ref_item_slice(Ref_item_array dst_arr, Ref_item_array src_arr) {
    assert(dst_arr.size() >= src_arr.size());
    void *dest = dst_arr.array();
    const void *src = src_arr.array();
    if (!src_arr.is_null())
      memcpy(dest, src, src_arr.size() * src_arr.element_size());
  }
```

> 所以 Item 树里所有 `Item_ref` 的 `ref` 指针**始终指向 slice 0 的槽位**——"切换切片"只是换槽位内容。这是"同一棵 Item 树、多套字段指针"得以共存的实现方式。

```cpp
// sql/sql_optimizer.cc:204
bool JOIN::alloc_ref_item_slice(THD *thd_arg, int sliceno) {
  assert(sliceno > 0);
  assert(ref_items[sliceno].is_null());
  size_t count = ref_items[0].size();
  Item **slice = thd_arg->mem_root->ArrayAlloc<Item *>(count);
  if (slice == nullptr) return true;
  ref_items[sliceno] = Ref_item_array(slice, count);
  return false;
}

// sql/sql_optimizer.cc:214
bool JOIN::alloc_indirection_slices() {
  const int num_slices = REF_SLICE_WIN_1 + m_windows.elements;

  assert(ref_items == nullptr);
  ref_items = (*THR_MALLOC)->ArrayAlloc<Ref_item_array>(num_slices);
  if (ref_items == nullptr) return true;

  tmp_fields =
      (*THR_MALLOC)
          ->ArrayAlloc<mem_root_deque<Item *>>(num_slices, *THR_MALLOC);
  if (tmp_fields == nullptr) return true;

  return false;
}
```

`QEP_TAB::ref_item_slice` 记录"这张表当前用哪一片"，初值 `REF_SLICE_SAVED_BASE`。

### 七、TABLE 上的优化器相关成员

这些字段**同时被优化器和执行器读写**，是最容易混淆所有权的地方：

| 成员 | 谁写 | 说明 |
|---|---|---|
| `possible_quick_keys` | `update_ref_and_keys` | 本表可能用到的 range 键位图 |
| `quick_keys` / `quick_rows` / `quick_cost` | range 优化器 | 已选定的 range 计划摘要 |
| `covering_keys` | `update_ref_and_keys` | 覆盖索引位图（决定 `key_read`） |
| `keys_in_use_for_query` / `keys_in_use_for_group_by` / `keys_in_use_for_order_by` | `update_ref_and_keys` / `add_loose_index_scan_and_skip_scan_keys` | 三个用途各异的 `Key_map` |
| `reginfo.qep_tab` | `QEP_TAB::set_table`（**双向绑定**） | TABLE 反查 QEP_TAB |
| `key_read` | 优化器设、`set_plan_state` 快照 | 只走索引不回表 |
| `read_set` / `write_set` | prepare 与 DML | 列的读写位图 |
| `file->stats.records` / `records_in_range` | 引擎 | 统计信息，见 [`01_cost_model.md`](01_cost_model.md) |

★ `reginfo.qep_tab` 是一条**反向指针**：`QEP_TAB::set_table()` 里同时写 `t->reginfo.qep_tab = this`。某些迭代器（如排序迭代器）需要从 `TABLE` 反查 `QEP_TAB`，这条指针就是为此存在。它也是"为什么 `TABLE` 不是纯粹的存储层对象"的原因。

### 八、完整生命周期与内存归属

```
① lex_start()
   └─ 预建 Query_expression / Query_block（空壳）
        JOIN 此时还不存在

② Query_block::prepare()
   ├─ 解析与绑定；JOIN 仍未创建
   └─ 见 ../06_resolver_prepare.md

③ Query_expression::optimize()  ← unit 级递归入口（sql_union.cc:985）
   └─ Query_block::optimize()
        ├─ new JOIN(thd, this)             ← JOIN 诞生（4 个成员被置毒值）
        ├─ JOIN::optimize()                （sql_optimizer.cc:338）
        │    ├─ :696  make_join_plan()
        │    │    ├─ :5314  init_planner_arrays()   ← 五个数组分配
        │    │    ├─ :5394  Optimize_table_order     ← 写 positions / best_positions
        │    │    ├─ :5405  positions = nullptr
        │    │    ├─ :5408  get_best_combination()
        │    │    │          ├─ :3204  best_ref 重排
        │    │    │          ├─ :3206  tab->set_position(pos)
        │    │    │          └─ :3261  join_tab = nullptr   ★ 交棒
        │    │    └─ :5414  best_positions = nullptr
        │    ├─ :703   ASSERT_BEST_REF_IN_JOIN_ORDER
        │    ├─ :1011  alloc_qep(tables)            ← qep_tab 诞生 + QEP_TAB::init
        │    ├─ :1020  make_join_readinfo()
        │    ├─ :1023  make_tmp_tables_info()       ← 建临时表
        │    ├─ :1037  create_access_paths()        ← 产出 m_root_access_path
        │    ├─ :1064  push_to_engines()
        │    └─ :1067  set_plan_state(PLAN_READY)   ← m_keyread_optim 快照
        └─ 递归：derived / 子查询的 unit 各自 optimize

④ 迭代器创建——★ 不存在 Query_expression::create_iterators()
   Query_expression::optimize(..., create_iterators=true, ...)
     └─ sql_union.cc:1128
          m_root_iterator = CreateIteratorFromAccessPath(
              thd, m_root_access_path, join, /*eligible_for_batch_mode=*/true);
          // ★ m_root_iterator 是 Query_expression 的成员 ⇒ 迭代器树归 unit 所有
   （另一入口：Query_expression::force_create_iterators()，sql_union.cc:1179）

⑤ 执行 ExecuteIteratorQuery()

⑥ 收尾
   ├─ JOIN::cleanup(bool full)   ← 部分清理，可重复调用（PS 复用场景）
   ├─ JOIN::destroy()            ← 彻底销毁：释放 qep_tab、filesort、临时表、Item 列表
   └─ thd->lex->destroy() → lex_end()
```

**内存归属**：

- `JOIN` 本身与 `keyuse_array`、`tmp_table_param` 都是**值成员或基于 `THD::mem_root` 的容器**，随 JOIN 一起释放；
- `join_tab` / `best_ref` / `qep_tab` / `positions` / `best_positions` 数组在 `init_planner_arrays` / `get_best_combination` 里显式分配，由 `JOIN::destroy` 释放；
- `m_root_access_path` 树在 hypergraph 路径下由 `JOIN` 的 `m_access_paths` 持有；
- 整个 LEX / JOIN / Item 树最终靠 `thd->mem_root->ClearForReuse()` **一次性回收**（见 [`../01_protocol_to_dispatch.md`](../01_protocol_to_dispatch.md)），这是 MySQL 不做精细内存管理的根本设计。

**`cleanup` vs `destroy` 的区别**（PS 场景下很重要）：

- ⚠️ **`JOIN::cleanup()` 在 8.0.39 里是**无参**的**（`void cleanup();`，声明 `sql_optimizer.h:699` 的注释是 *"Cleanup this JOIN. Not a full cleanup. reusable?"*，定义 `sql_select.cc:1986`）。5.7 时代的 `cleanup(bool full)` 已被删除。它可被调用多次，用于"同一条预编译语句的多次执行之间"复位可变状态；
- `JOIN::destroy()`（定义 `sql_select.cc:1888`）只调一次。

```cpp
// sql/sql_select.cc:1986
void JOIN::cleanup() {
  DBUG_TRACE;

  assert(const_tables <= primary_tables && primary_tables <= tables);

  if (qep_tab || join_tab || best_ref) {
    for (uint i = 0; i < tables; i++) {
      QEP_TAB *qtab;
      TABLE *table;
      if (qep_tab) {
        assert(!join_tab);            // ★★ qep_tab 与 join_tab 互斥
        qtab = &qep_tab[i];
        table = qtab->table();
      } else {
        qtab = nullptr;
        table = (join_tab ? &join_tab[i] : best_ref[i])->table();
      }
      if (!table) continue;
      cleanup_table(table);
    }
  } else if (thd->lex->using_hypergraph_optimizer()) {
    for (Table_ref *tl = query_block->leaf_tables; tl; tl = tl->next_leaf) {
      cleanup_table(tl->table);
    }
    for (JOIN::TemporaryTableToCleanup cleanup : temp_tables) {
      cleanup_table(cleanup.table);
    }
  }
}
```

> ★ `qep_tab ? ... : (join_tab ? ... : best_ref[i])` 这三选一正是**生命周期三相的判据**：已建 qep_tab ⇒ 执行期；否则 join_tab 还在 ⇒ 规划期；否则只能走 best_ref。

```cpp
// sql/sql_select.cc:1888
void JOIN::destroy() {
  cond_equal = nullptr;

  set_plan_state(NO_PLAN);

  if (qep_tab) {
    assert(!join_tab);
    for (uint i = 0; i < tables; i++) {
      TABLE *table = qep_tab[i].table();
      if (table != nullptr) {
        // These were owned by the root iterator, which we just destroyed.
        // Keep filesort_free_buffers() from trying to call CleanupAfterQuery()
        // on them.
        table->sorting_iterator = nullptr;          // ★ 迭代器树已销毁，断引用
        table->duplicate_removal_iterator = nullptr;
      }
      qep_tab[i].cleanup();
    }
  } else if (thd->lex->using_hypergraph_optimizer()) {
    ...
    for (JOIN::TemporaryTableToCleanup cleanup : temp_tables) {
      ...
      close_tmp_table(cleanup.table);
      free_tmp_table(cleanup.table);
      ::destroy(cleanup.temp_table_param);
    }
    for (Filesort *filesort : filesorts_to_cleanup) {
      ::destroy(filesort);                          // ★ filesort 只在 hypergraph 这里释放
    }
    temp_tables.clear();
    filesorts_to_cleanup.clear();
  }
  if (join_tab || best_ref) {
    for (uint i = 0; i < tables; i++) {
      JOIN_TAB *const tab = join_tab ? &join_tab[i] : best_ref[i];
      tab->cleanup();                               // ★ 只 cleanup，数组本身不释放
    }
  }

  // Run Cached_item DTORs!
  group_fields.destroy_elements();
  semijoin_deduplication_fields.destroy_elements();

  tmp_table_param.cleanup();

  /* Cleanup items referencing temporary table columns */
  if (tmp_fields != nullptr) {
    cleanup_item_list(tmp_fields[REF_SLICE_TMP1]);
    cleanup_item_list(tmp_fields[REF_SLICE_TMP2]);
    for (uint widx = 0; widx < m_windows.elements; widx++) {
      cleanup_item_list(tmp_fields[REF_SLICE_WIN_1 + widx]);
    }
  }

  destroy_sj_tmp_tables(this);

  List_iterator<Semijoin_mat_exec> sjm_list_it(sjm_exec_list);
  Semijoin_mat_exec *sjm;
  while ((sjm = sjm_list_it++)) ::destroy(sjm);
  sjm_exec_list.clear();

  keyuse_array.clear();
  // Free memory for rollup arrays
  if (query_block->olap == ROLLUP_TYPE) {
    rollup_group_items.clear();
    rollup_group_items.shrink_to_fit();
    rollup_sums.clear();
    rollup_sums.shrink_to_fit();
  }
}
```

> ⚠️ **勘误：`destroy()` 一个数组都不 free**。`join_tab` / `best_ref` / `map2table` / `positions` / `best_positions` 全部靠 `thd->mem_root` 一次性回收；`destroy()` 只做逐个 `cleanup()` + 关闭临时表 + 释放 filesort/hash 表。上表"被置空于 `JOIN::destroy`"的说法是错的。

**`set_plan_state()`**（`sql_optimizer.cc:1280`）——它不只是改状态，还顺手做 `m_keyread_optim` 快照：

```cpp
void JOIN::set_plan_state(enum_plan_state plan_state_arg) {
  // A plan should not change to another plan:
  assert(plan_state_arg == NO_PLAN || plan_state == NO_PLAN);
  if (plan_state == NO_PLAN && plan_state_arg != NO_PLAN) {
    if (qep_tab != nullptr) {
      /*
        We want to cover primary tables, tmp tables. Note that
        make_tmp_tables_info() may have added a sort to the first non-const
        primary table, so it's important to do this assignment after
        make_tmp_tables_info().
      */
      // ★★ 这条注释就是"必须在 make_tmp_tables_info() 之后调用"的源码依据
      for (uint i = const_tables; i < tables; ++i) {
        qep_tab[i].set_condition_optim();
        qep_tab[i].set_keyread_optim();   // ★ 快照 table->key_read
      }
    }
  }

  DEBUG_SYNC(thd, "before_set_plan");

  // If SQLCOM_END, no thread is explaining our statement anymore.
  const bool need_lock = thd->query_plan.get_command() != SQLCOM_END;

  if (need_lock) thd->lock_query_plan();
  plan_state = plan_state_arg;
  if (need_lock) thd->unlock_query_plan();
}
```

```cpp
// sql/sql_optimizer.h:745 —— 真名是 JOIN 的嵌套枚举 enum_plan_state（不是顶层 Plan_state）
  enum enum_plan_state {
    NO_PLAN,      ///< No plan is ready yet
    ZERO_RESULT,  ///< Zero result cause is set
    NO_TABLES,    ///< Plan has no tables
    PLAN_READY    ///< Plan is ready
  };
```

**`create_access_paths()`**（`sql/sql_executor.cc:2960`）——`m_root_access_path` 的唯一写入点：

```cpp
void JOIN::create_access_paths() {
  assert(m_root_access_path == nullptr);

  AccessPath *path = create_root_access_path_for_join();
  path = attach_access_paths_for_having_and_limit(path);
  path = attach_access_path_for_update_or_delete(path);

  m_root_access_path = path;
}
```

**"best_ref 有效区间"的形式化判据**（`sql/sql_optimizer.h:1082`）：

```cpp
/**
  Use this in a function which depends on best_ref listing tables in the
  final join order. If 'tables==0', one is not expected to consult best_ref
  cells, and best_ref may not even have been allocated.
*/
#define ASSERT_BEST_REF_IN_JOIN_ORDER(join)                                 \
  do {                                                                      \
    assert((join)->tables == 0 || ((join)->best_ref && !(join)->join_tab)); \
  } while (0)
```

> ★ 注意它**不检查 `qep_tab`**——`qep_tab` 是否已建由调用方各自保证。所以"看到 `ASSERT_BEST_REF_IN_JOIN_ORDER` 只能说明 `join_tab` 已交棒"，不能说明 `qep_tab` 已存在。

### 九、新旧优化器对象模型对照

| 维度 | 经典优化器 | hypergraph |
|---|---|---|
| 表的表示 | `JOIN_TAB`（规划）→ `QEP_TAB`（执行） | 无；只有 `RelationalExpression` 与 `AccessPath` |
| 计划搜索中间态 | `positions[]` / `best_positions[]` / `best_ref[]` | `CostingReceiver` 的 Pareto 前沿 + `m_access_paths` 哈希 memo |
| 有序性 | `m_ordered_index_usage` 三态（**只能选一个**） | `InterestingOrder` 集合，可保留多个候选序 |
| 临时表归属 | `QEP_TAB::tmp_table_param` | `JOIN::temp_tables` 数组 |
| filesort 归属 | `QEP_TAB::filesort` | `JOIN::filesorts_to_cleanup` |
| 最终产物 | `JOIN::m_root_access_path` | 同（两条路汇合） |
| 半连接 | 五种策略模拟，状态散在 QEP_TAB | 一等公民（`RelationalExpression::SEMIJOIN`） |

★ 一个容易误解的点：**hypergraph 并不复用 `JOIN` 的大部分字段**。它只用 `JOIN` 作为"query block 级容器"（conditions、`tmp_table_param`、windows、query_block 指针），而把 join 序、访问方法、代价全部放进自己的结构。所以读 hypergraph 代码时看到 `join->xxx` 多数是那些"容器字段"。

---

## ★ 本机制里的工程实现技法

### 1. 用"必然崩溃的初值"代替断言

```cpp
// JOIN 构造函数（sql/sql_optimizer.cc:184）—— 实际是 4 个成员
      /*
        Those four members are meaningless before JOIN::optimize(), so force a
        crash if they are used before that.
      */
      where_cond(reinterpret_cast<Item *>(1)),
      having_cond(reinterpret_cast<Item *>(1)),
      having_for_explain(reinterpret_cast<Item *>(1)),
      tables_list(reinterpret_cast<Table_ref *>(1)),
```

注释解释：这四个成员在 `JOIN::optimize()` 之前无意义，用非法值初始化，使任何提前访问都**必然段错误**，比 `assert` 更可靠（`assert` 在 release 构建里被编译掉）。

同样的技法还有 `plan_idx` 的 `NO_PLAN_IDX` 哨兵值：它不是一个"未初始化"值，而是一个**可判定的合法状态**。

### 2. `QEP_shared_owner` 的"半公开"封装

`QEP_shared` 的 getter/setter 全是 public，但它被 `QEP_shared_owner` 以 **protected 成员**持有，因此只有 `JOIN_TAB` / `QEP_TAB` 能访问。这是一个"用继承关系代替 friend 声明"的封装技巧：

```
实际效果：QEP_shared 对外完全不可见，对两个子类完全可见
```

### 3. `set_idx` 的"一生一次"约束

```cpp
void set_idx(plan_idx i) {
  assert(m_idx == NO_PLAN_IDX);   // Index should not change in lifetime
  m_idx = i;
}
```

下标一旦设定就不允许再变。这是因为 `m_idx` 被大量缓存（外连接边界、`first_sj_inner` 等都存的是下标而非指针），改动会引发连锁失效。

### 4. 数组"拔除"而不是"释放"

`join_tab = nullptr` / `best_ref = nullptr` 只是**断引用**，内存仍在（由 `JOIN::destroy` 统一释放）。这么做的目的是**让"误用"变成可检测的**：后续代码若访问 `join->best_ref` 会拿到 nullptr 立即崩溃，而不是拿到一个"已经过期的顺序"。

### 5. `Temp_table_param` 的拷贝构造

`Temp_table_param` 有显式拷贝构造函数（源码里逐个字段拷贝，包括 `allow_group_via_temp_table`）。这是"母版 + 拷贝"模式的基础——每个临时表拷贝一份母版再改，避免重新计算全部参数。

### 6. RAII 切换 `ref_items` 切片

`Switch_ref_item_slice` 是 RAII：构造时切到目标切片，析构时切回。配合 `QEP_TAB::ref_item_slice` 记录"当前片"，使得"读基表字段"与"读临时表字段"可以在同一个 Item 树下切换而无需重建表达式树。

### 7. 双向绑定 TABLE ↔ QEP_TAB

`QEP_TAB::set_table()` 同时写 `t->reginfo.qep_tab = this`。双向指针方便但危险：清理时必须两边都断（源码里 `QEP_TAB::cleanup()` 会写 `t->reginfo.qep_tab = nullptr`）。

---

## 可观测性

### 调试时的定位手段

| 想看什么 | 看哪个对象 |
|---|---|
| 当前 join 顺序 | `join->best_ref[]`（规划期）或 `join->qep_tab[]`（执行期） |
| 每表的访问方法与代价 | `qep_tab[i].type()` / `qep_tab[i].position()->prefix_cost` / `rows_fetched` |
| 半连接策略 | `qep_tab[i].position()->sj_strategy`、`firstmatch_return`、`loosescan_key_len` |
| 是否要临时表 | `join->need_tmp_before_win`、`qep_tab[i].op_type`、`join->tmp_table_param.sum_func_count` |
| 排序来源 | `join->order.src`、`join->group_list.src`（`ESC_ORDER_BY` / `ESC_GROUP_BY` / `ESC_DISTINCT`） |
| 当前 SELECT list 用哪套表达式 | `join->get_ref_item_slice()` / `qep_tab[i].ref_item_slice` |
| 最终计划 | `join->m_root_access_path`（block 级）、unit 的 `m_root_access_path`（unit 级） |
| 最终迭代器 | unit 的 `m_root_iterator` |

### 常用验证手段

- `EXPLAIN FORMAT=tree`：直接看 AccessPath 树（见 [`../11_explain_and_trace.md`](../../../infra/observability/explain_trace.md)）
- `EXPLAIN FORMAT=json` 的 `cost_info`：读的是 `POSITION` 的代价字段
- optimizer trace 的 `join_optimization` → `plan_前缀` 节点：逐步记录 `best_ref` 的演化（见 [`physical/06_join_order.md`](physical/06_join_order.md)）
- `ASSERT_BEST_REF_IN_JOIN_ORDER` 等断言宏：debug 构建下可以验证"当前是否处于 best_ref 有效区间"

---

## Misc

### 扩展点：改这一块要动哪些地方

| 想做什么 | 要动的地方 |
|---|---|
| 给优化器加一个新状态字段 | 加在 `JOIN`；若需要跨阶段可见，考虑它属于"规划期"还是"执行期"——放错位置会导致它在某个阶段被置空 |
| 加一种表的访问方法 | `join_type` 枚举 + `best_access_path` 的代价分支 + `QEP_TAB` 的字段 + `create_access_paths` 的翻译 + 对应 `RowIterator` |
| 加一种临时表用途 | `QEP_TAB::op_type` 枚举 + `Setup_func` + `make_tmp_tables_info` 的分派 + `Temp_table_param` 的新字段（记得同步拷贝构造） |
| 加一个执行阶段（需要新切片） | 新的 `REF_SLICE_*` + `alloc_ref_item_slice` + 切换点的 RAII |
| 让 hypergraph 复用某个经典优化器字段 | 先确认该字段在 hypergraph 路径下是否真的被填充——很多字段在 hypergraph 下是空的 |

### 已知缺陷与历史包袱

- `JOIN` 类过大（一千多行、上百个成员），是公认的技术债；源码注释里有多处 `@todo WL#6570` 指向"prepare once, execute many"这条未完成重构（它想做的正是理清 `JOIN` 的生命周期）。
- `join_tab` / `best_ref` 的"置空"约定没有集中文档，只在零散注释里。
- `where_cond` 的语义在"有表 / 无表"两种情况下不同（注释明说），容易误用。
- `best_positions` 被标为 "scratch array" 却仍被 `QEP_shared::m_position` 长期引用，属于"名义上退役、实际仍被依赖"。

### 社区边界澄清

- **没有计划缓存**：`JOIN` 每次执行重建，社区版不缓存（见 [`../12_prepared_statement.md`](../12_prepared_statement.md)）。所以本篇的生命周期对**每一次** `COM_STMT_EXECUTE` 都会完整走一遍。
- **`QEP_TAB` 不是"查询执行计划"的通用概念**：它是 MySQL 特有的"表链 + 每表 op_type"模型，不要与 PG 的 `PlanState` 或 Oracle 的执行计划行源一一对应。
- **hypergraph 不是"另一个 JOIN"**：它复用 `JOIN` 作为容器但不复用其表格结构，读代码时不要用经典优化器的字段去套。

---

## 参考

**论文 / 理论**

- Graefe《Volcano—An Extensible and Parallel Query Evaluation System》(1990) —— 迭代器模型；MySQL 把"物理计划树"与"执行状态"分开（AccessPath vs RowIterator）正源于此，但 `JOIN` 本身仍保留了可变黑板风格
- Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979) —— `POSITION` 里"前缀代价"的概念直接来自 System R

**WorkLog**

- WL#6570 "prepare once, execute many" —— 源码中散落的 `@todo`，涉及 `JOIN` 生命周期、`cleanup`/`destroy` 边界的历史遗留

**官方文档**

- MySQL 8.0 Internals / Source Code Documentation（MySQL 官方源码文档，其中对 `JOIN` 与 `QEP_TAB` 的历史描述仍可参考，但与 8.0.39 已有偏差）

**相关文档**

- [`physical/06_join_order.md`](physical/06_join_order.md) —— `POSITION` 全字段与搜索算法
- [`physical/07_access_method.md`](physical/07_access_method.md) —— `Key_use` 与访问方法代价
- [`../08_access_path.md`](../08_access_path.md) —— AccessPath 结构与两条创建路径
- [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) —— 分组/排序标志的推导
- [`physical/09_hypergraph.md`](physical/09_hypergraph.md) —— hypergraph 的对象模型
- [`../01_protocol_to_dispatch.md`](../01_protocol_to_dispatch.md) —— mem_root 一次性回收

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

#### 2.2 条件

| 字段 | 谁写 | 谁读 | 备注 |
|---|---|---|---|
| `where_cond` | `get_optimizable_conditions` → `optimize_cond` | `make_join_plan`、EXPLAIN | 注释写明"若没有表则执行期直接用；否则拆成片段挂到各 JOIN_TAB，之后不再用" |
| `having_cond` | 同上 | `make_tmp_tables_info`、执行期 | 可能被下推到临时表，此时置空 |
| `having_for_explain` | `make_tmp_tables_info`（下推前保存） | EXPLAIN | 下推导致 `having_cond` 置空，为 EXPLAIN 留副本 |
| `cond_equal` | `build_equal_items` | `substitute_for_best_equal_field` | 顶层等值类（见 [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md)） |

**构造期的防御性初值**：JOIN 构造函数把 `where_cond` / `having_cond` 初始化为 `reinterpret_cast<Item *>(1)` —— 一个必然崩溃的地址，注释解释是"这两个成员在 `JOIN::optimize()` 之前无意义，若被误用则强制崩溃"。这是一个**用非法值代替断言**的工程技法（见「工程实现技法」）。

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

#### 3.3 JOIN_TAB 独有（规划期）

候选键（`keys()` / `const_keys`）、依赖表集合（`dependent`）、`key_dependent`、`m_skip_records_in_range`（跳过 `records_in_range` 估算的开关）等，服务于 `best_access_path` 的代价决策。

### 四、五个数组：谁分配、谁重排、谁被置空

这是本篇最实用的一节。六个指针的生命周期：

| 指针 | 分配 | 重排/改写 | 被置空于 | 存活区间 |
|---|---|---|---|---|
| `join_tab` | `init_planner_arrays` | `get_best_combination` 用 `best_ref` 重排后**置空** | `get_best_combination` | `make_join_plan` 内部 |
| `best_ref` | 同上 | **搜索时被反复重排/回退**（`memcpy` 备份恢复） | `make_tmp_tables_info` 之后 | 规划期 ~ 计划改进早期 |
| `map2table` | 同上 | 从不 | 不置空（随 JOIN 释放） | 全程 |
| `positions` | 同上 | 搜索时作为**栈**被写入 | `make_join_plan` 结束时释放 | 搜索期 |
| `best_positions` | 同上 | `consider_plan` 回写最优前缀 | 注释明说"**scratch array，get_best_combination 之后不再用**"，但 `QEP_shared::m_position` 仍指向它 | 搜索期 ~ 执行准备 |
| `qep_tab` | `get_best_combination` | 不再重排 | `JOIN::destroy` | 执行准备 ~ 执行结束 |

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
| `end_write_records` | 写入方式 | `make_tmp_tables_info` |
| `m_window` / rollup 相关 | 窗口与 rollup 的临时表形态 | 窗口/rollup 处理 |

**为什么需要"母版 + 拷贝"**：一个查询可能有多个临时表（DISTINCT 一个、GROUP BY 一个、窗口每个窗口一个），它们共享大部分参数但各自有差异。拷贝母版再改是成本最低的做法（见「工程实现技法」的拷贝构造函数）。

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

③ Query_expression::optimize()  ← unit 级递归入口
   └─ Query_block::optimize()
        ├─ new JOIN(thd, this)          ← JOIN 诞生（分配在 THD 的 mem_root？见下）
        ├─ JOIN::optimize()
        │    ├─ init_planner_arrays()   ← 分配 join_tab/best_ref/map2table/positions/best_positions
        │    ├─ make_join_plan()        ← 搜索；结束时 join_tab/positions 退役
        │    ├─ get_best_combination()  ← 分配 qep_tab 数组；join_tab 置空
        │    ├─ make_tmp_tables_info()  ← 建临时表；best_ref 置空
        │    └─ create_access_paths()   ← 产出 m_root_access_path
        └─ 递归：derived / 子查询的 unit 各自 optimize

④ Query_expression::create_iterators()（或在 optimize 末尾）
   └─ CreateIteratorFromAccessPath()
        └─ unit 的 m_root_iterator  ← 迭代器树归 unit 所有

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

- `JOIN::cleanup()` 可被调用多次（源码注释明说），用于"同一条预编译语句的多次执行之间"复位可变状态；
- `JOIN::destroy()` 只调一次，真正释放。

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
// JOIN 构造函数
where_cond(reinterpret_cast<Item *>(1)),
having_cond(reinterpret_cast<Item *>(1)),
```

注释解释：这两个成员在 `JOIN::optimize()` 之前无意义，用非法值初始化，使任何提前访问都**必然段错误**，比 `assert` 更可靠（`assert` 在 release 构建里被编译掉）。

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

- `EXPLAIN FORMAT=tree`：直接看 AccessPath 树（见 [`../11_explain_and_trace.md`](../11_explain_and_trace.md)）
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

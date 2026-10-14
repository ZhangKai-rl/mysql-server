# 22 优化器 Worklog 与版本演进时间线

> 基于 MySQL 8.0.39。本篇收集两样东西：**源码里真实存在的 Worklog 引用**（它们是"为什么这段代码写成这样"的第一手答案），以及**优化器的版本演进时间线**（用来判断"某个行为在哪个版本变了"）。
>
> 这不是"如何研发"的方法论，而是**阅读源码时的字典**：当你看到 `@todo WL#6570` 这种活化石、或想知道"这个优化是哪年加的"，来这里查。
>
> **边界**：本篇只整理**优化器/解析**相关的时间线与 WL。InnoDB 的 WL 见 [`../../../innodb/`](../../../innodb/) 各篇；复制与 binlog 的 WL 不在本篇范围。

## 目录

- [概述](#概述)
- [一、版本演进时间线](#一版本演进时间线)
- [二、Worklog 清单（按主题）](#二worklog-清单按主题)
- [三、源码里的"活化石"：散落的 @todo](#三源码里的活化石散落的-todo)
- [四、经典 bad case 与源码根因](#四经典-bad-case-与源码根因)
- [可观测性](#可观测性)
- [**五、本仓库 WL# 引用总盘点（按出现次数）**](#五本仓库-wl-引用总盘点按出现次数)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 为什么需要这一篇

读 MySQL 优化器源码会遇到三类困惑，本篇分别对应：

| 困惑 | 本篇哪一节 |
|---|---|
| "这个 `@todo WL#6570` 是什么意思？" | 第三节 |
| "这个函数为什么写得这么别扭？" | 第二节（WL 解释了历史约束） |
| "这个行为在 5.7 和 8.0 不一样，哪个版本改的？" | 第一节 |

★ 特别强调第一类：MySQL 源码里有**大量**指向未完成 WL 的 `@todo`，它们不是"待办清单"，而是**历史约束的化石**——理解了它们，才能理解为什么某些代码看起来"不干净"。

### WL 编号怎么查

MySQL 的 Worklog 编号（`WL#nnnn`）在源码注释里直接出现。官方入口：

- MySQL 官方的 Worklog 归档（Oracle 内部的开发任务系统，部分编号可在 MySQL 的 release notes / 源码注释里交叉验证）
- 更可靠的交叉验证方式：**在源码里 grep** `WL#` —— 编号旁通常有说明性注释

★ 本篇列出的 WL 编号**全部来自本仓库源码注释**，不是凭记忆（见每项的"源码出处"列）。

---

## 一、版本演进时间线

### 关键版本与优化器/解析器的变化

| 版本 | 变化 | 影响本篇哪些文档 |
|---|---|---|
| **5.6** | MRR / BKA；semi-join 五种策略（含 LooseScan / DuplicateWeedout / FirstMatch / Materialize）；子查询物化引擎；**optimizer trace**；持久化统计 | 03/06/09/02 篇、可观测性 |
| **5.7** | 代价模型重构为可配置表（`mysql.server_cost` / `mysql.engine_cost`）；`EXPLAIN FORMAT=JSON`；生成列（虚拟列） | 01_cost_model |
| **8.0.0** | 数据字典（DD）化：view 定义移入 DD；`utf8mb4` 成为默认字符集、`utf8mb4_0900_ai_ci`（**NO PAD**）成为默认 collation | 20_view_resolution、21_collation |
| **8.0.2** | **CTE**（`WITH`）与**窗口函数**；**直方图**（`CREATE HISTOGRAM`）；派生表物化与合并的持续改进 | 05_contextualize、runtime/04_window |
| **8.0.13** | **函数索引**（functional key parts，基于生成列实现） | 13_functional_mv_index |
| **8.0.17** | **多值索引**（multi-valued index，JSON 数组） | 13_functional_mv_index |
| **8.0.18** | **Hash Join**（取代 BNL 的默认地位） | 09_executor_iterator、runtime/05_join_buffer |
| **8.0.18** | **`EXPLAIN ANALYZE`** | 11_explain_and_trace |
| **8.0.22** | **`AccessPath` + RowIterator 改造完成**；**hypergraph 优化器实验性引入**（默认 off）；不可见索引 | 08_access_path、09_hypergraph |
| **8.0.28** | **CSE（公共子表达式消除）** 进入 hypergraph 路径 | physical/18_hypergraph_advanced |
| **8.0.31** | **INTERSECT / EXCEPT** 进入社区版；`Query_term` 树重构完成 | 19_set_operation、05_contextualize |
| **8.0.39**（本仓库） | 现状：两套优化器并存（greedy / hypergraph），`Query_term` 与 `Query_expression::first_query_block()` 链表两套表示并存 | 00_overview |

### 两条并行的演进主线

```
主线 A：物理计划的表达方式
  5.x   JOIN_TAB 链（优化与执行共用）
  5.6   拆出 QEP_TAB + QEP_shared
  8.0.22 AccessPath 树 + RowIterator 树（两套：经典优化器的 create_access_paths，
         hypergraph 的 FindBestQueryPlan 直出）

主线 B：优化器算法
  5.0-5.5  纯规则 + 简单代价
  5.6      greedy 搜索 + semi-join 策略
  8.0.22   hypergraph（DPhyp + 成本接收器 + interesting orders）
```

★ 主线 A 完成于 8.0.22，主线 B **仍在进行中**（两套并存，hypergraph 默认关闭）。这就是为什么 8.0.39 的 `JOIN::optimize` 里会同时看到"旧的对象模型"和"新的 AccessPath"。

---

## 二、Worklog 清单（按主题）

> 全部编号来自本仓库源码注释，列中标出了 grep 到的位置。

### 2.1 优化器基础设施

| WL | 主题 | 源码出处 | 说明 |
|---|---|---|---|
| **WL#5257** | Optimizer trace API | `sql/opt_trace.cc`、`opt_trace.h`、`opt_trace2server.cc` 的文件头注释（*"Implementation of the Optimizer trace API"*） | 全部 optimizer trace 节点的基础设施 |
| **WL#6570** | prepare once, execute many | 散落在 `sql_base.cc`、`item.cc`、`item_func.cc`、`sql_tmp_table.cc`、`sql_prepare.cc`、`table.h` | 详见第三节——**本仓库最大的"活化石群"** |
| **WL#2489** | better only_full_group_by | `sql/aggregate_check.h`（注释：*"In WL#2489, we have implemented the optional standard feature T301"*） | SQL 标准可选特性 T301，函数依赖检查 |

### 2.2 子查询与 semi-join

| WL | 主题 | 源码出处 | 说明 |
|---|---|---|---|
| **WL#1110** | 子查询物化引擎 | （见 [`logical/02_subquery.md`](logical/02_subquery.md) 引用） | 物化引擎的引入，是"IN→EXISTS vs 物化"决策的基础 |
| **WL#4389** | Transform EXISTS subqueries to semi-join | （见 [`logical/03_semijoin.md`](logical/03_semijoin.md) 引用） | 8.0.16 起 EXISTS 也能转 semi-join，扩展了转换适用范围 |
| **WL#5561** | semi-join 相关处理 | `sql/join_optimizer/join_optimizer.cc`（注释：*"removal happens for semijoin (Complete details in WL#5561)"*） | hypergraph 里 semi-join 是一等公民，这里的注释引用了该 WL 的细节 |

### 2.3 Item 与表达式

| WL | 主题 | 源码出处 | 说明 |
|---|---|---|---|
| **WL#5800** | `Item_cond` 的改造 | `sql/item_cmpfunc.h`（注释：*"Item_cond instead. See WL#5800"*） | 与条件树的表示有关 |
| **WL#12108** | 比较函数的适用范围 | `sql/item_cmpfunc.h`（注释：*"This function will limit itself to comparison between regular..."*） | 限定了比较只能在常规类型间进行 |
| **WL#7384** | 表达式的不确定性 | `sql/item_func.cc`（注释：*"uncertain. See WL#7384"*） | 与"能否在优化期求值"相关 |
| **WL#6059** | RIGHT JOIN 转换 | `sql/table.h`（注释：*"when WL#6059 is merged in (it really converts RIGHT JOIN to ...)"*） | 影响 `first_leaf_table()` 等接口 |

### 2.4 视图与字典

| WL | 主题 | 源码出处 |
|---|---|---|
| **WL#9446** | view 相关（DD） | `sql/dd_sql_view.cc` |

### 2.5 与优化器相邻、读代码时会遇到的 WL

| WL | 主题 | 源码出处 |
|---|---|---|
| WL#7069 | SDI（Serialized Dictionary Information） | `sql/sdi_utils.cc` |
| WL#2623 | NDB handler 的属性 | `sql/handler.cc` |
| WL#7909 / WL#9831 | JSON 自动包装规则 | `sql/item_json_func.cc` |
| WL#2111 | `CURRENT_*` 关键字的保留性 | `sql/sql_yacc.yy`（注释：*"not reserved in MySQL per WL#2111 specification"*） |

---

## 三、源码里的"活化石"：散落的 @todo

### 3.1 WL#6570（prepare once, execute many）

这是本仓库**出现次数最多**的未完成任务，grep 到的位置：

| 文件 | 注释要点 |
|---|---|
| `sql/sql_base.cc` | 多处 `WL#6570 remove-after-qa`；`@todo WL#6570 - is this reasonable???`；`move this assignment to a more strategic place?` |
| `sql/item.cc` | 多处 `WL#6570 remove-after-qa` |
| `sql/item_func.cc` | `@todo WL#6570 Should we return collation of Item node or variable entry?`；`change has effects:`；并明确提示 *"Don't forget to grep for WL#6570 in the whole tree, including mtr"* |
| `sql/sql_tmp_table.cc` | `@todo WL#6570 - might be allocated on THD->mem_root`；`Unsure if this is wise: We may choose a different engine on...` |
| `sql/sql_prepare.cc` | `@todo WL#6570` |
| `sql/table.h` | `@todo WL#6570 with prepare-once, replace with first_leaf_table()` |

**它想做什么**：把"prepare 一次、执行多次"的语义彻底理清。当前 8.0 的 prepare 与执行之间的边界并不干净（很多状态在 optimize 期被就地改写），PS 复用时需要靠 `cleanup()` / `thd->change_item_tree()` 这类机制补救。

**对读者的意义**：看到 `WL#6570` 就理解为"**这块代码的状态边界不干净，是历史遗留**"，不要以为当前写法是精心设计的结果。

**★★ 精确计数（纠正）**：全仓 **39 次**（`sql/` 33 + `mysql-test/` 6）。之前说的"33 次"大概率只 grep 了 `sql/`。

**★ 全部 33 处 `sql/` 命中的语义分组**：

| 类别 | 次数 | 形态 |
|---|---|---|
| `remove-after-qa` | **12** | 一行注释 + 一条 `assert(thd->stmt_arena->is_regular() \|\| !thd->lex->is_exec_started());` |
| `@todo WL#6570 ...` | 13 | 未决设计问题 |
| 纯说明 / 已注释掉的旧代码 | 8 | 如 `sql_optimizer.cc:403` 的 `{  // WL#6570` |

**★ `remove-after-qa` 的标准形态**（`sql/sql_update.cc:330`，★ 12 处全同）：

```cpp
    *it = cloned_field;
    // WL#6570 remove-after-qa
    assert(thd->stmt_arena->is_regular() || !thd->lex->is_exec_started());
```

> ★ 断言语义："当前 arena 是常规 arena，或还没进入 EXECUTE 阶段"——即"**这块代码只在 prepare 期跑**"。WL#6570 完成后应当整体删掉。

**★★ 唯一给出 WL#6570 语义定义的地方**（`sql/table.h:3234`）：

```cpp
  /// Return any leaf table that is not an inner table of an outer join
  /// @todo WL#6570 with prepare-once, replace with first_leaf_table()
  /// when WL#6059 is merged in (it really converts RIGHT JOIN to
  /// LEFT JOIN so the first leaf is part of a LEFT JOIN,
  /// guaranteed).
  Table_ref *any_outer_leaf_table() {
```

> ★★ **`prepare-once` 就是 WL#6570 的核心语义**（prepare 一次、execute 多次）。同时暴露它的前置项 **WL#6059**（RIGHT JOIN → LEFT JOIN 转换）。

**★ 最有价值的一处：作者自己留了 grep 提示**（`sql/item_func.cc:6762`）：

```cpp
bool Item_func_get_user_var::resolve_type(THD *thd) {
  set_nullable(true);
  /*
    @todo WL#6570 change has effects:
    - bad: in packet.test (see comment there), ps_<n><engine>.test
    (e.g. ps_5merge.test) (see comment in include/ps_conv.inc),
    ps_w_max_indexes_64
    - good: type_temporal_fractional.test (see comment in that file)
    - fixes failure of: sysschema.pr_statement_performance_analyzer and
    sysschema.format_statement.
    Don't forget to grep for "WL#6570" in the whole tree, including mtr
    tests.
  */
```

> ★★ 作者坦白这个改动**让某些测试变坏、某些变好**，并留下"记得全树 grep WL#6570"的提示——**这正是本篇方法论的来源**。

**★ 唯一在代码（而非注释）里的 WL#6570**（`sql/sql_optimizer.cc:398`）：

```cpp
  // if (query_block->materialized_derived_table_count) {
  {  // WL#6570
    for (Table_ref *tl = query_block->leaf_tables; tl; tl = tl->next_leaf) {
```

> ★ 原本的 `if` 条件被**注释掉**、换成无条件块，只用 `// WL#6570` 标记——典型的"先跑起来再说"。

**★ "prepare once" 的另一处直接描述**（`sql/sql_base.cc:7953`）：

```cpp
    /*
      @todo WL#6570 - is this reasonable???
      Also refactor this code to replace "cached_table" with "table_ref" -
      as there is no longer need for more than one resolving, hence
      no "caching" as well.
    */
```

> ★ "**no longer need for more than one resolving**"——WL#6570 后只 resolve 一次。

**★ 6 处 `mysql-test/` 命中**（★ 其中一处是整个测试文件的存在理由）：

```cpp
  ##todo WL#6570 i==3 and i==4 seem to do the same?        /* t/func_comparison.test:1823 */
--echo # It is a behaviour change introduced by WL#6570.   /* t/user_var.test:585 */
--echo # Test for a memory leak in WL#6570                 /* t/case.test:354 */
#@todo WL#6570:                                            /* t/ps_type_deduction.test:1  ←★ 第 1 行 */
```

### 3.2 其它值得记住的 TODO

| 位置 | 内容 |
|---|---|
| `sql/sql_optimizer.cc`（`test_skip_sort`） | `TODO: Explain the allow_group_via_temp_table part of the test below` —— 作者自己承认判定式难以解释（见 [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)） |
| `sql/sql_optimizer.cc`（ref access） | `@todo Change how ref access for BINARY/VARBINARY fields are done so that only qualifying rows are returned from the storage engine`（见 [`21_collation_index_usability.md`](21_collation_index_usability.md)） |
| `sql/item.cc`（`agg_item_set_converter`） | `TODO: avoid conversion of any values with repertoire ASCII and 7bit-ASCII-compatible, not only numeric/datetime origin` |
| `sql/item.cc`（`agg_item_set_converter`） | `@todo - check why the constructors may return error` |
| `sql/sql_select.cc`（set operation） | `TODO: Query_term and first_query_block() should be kept in sync` / 关于两套表示并存的计划 |
| `sql/table.cc` | `TODO: wl#7840 to get a more light weight parsing of expressions` |
| `sql/sql_partition.cc` | `TODO: add optimization to use index if possible, see WL#5397` |

---

## 四、经典 bad case 与源码根因

> 这一节是"问题驱动"的索引：给出症状 → 源码根因 → 深入哪一篇。每个 bad case 的根因都在本目录的某篇里有完整剖析。

### 4.1 走错索引 / 突然变慢

| 症状 | 根因 | 深入 |
|---|---|---|
| 有索引却全表扫 | 行数估算失真（index dive vs `rec_per_key`）；或多列条件假设列独立 | [`01_cost_model.md`](01_cost_model.md) |
| 长 `IN` 列表突然变慢 | `range_optimizer_max_mem_size` 触发降级，range 候选消失 | [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |
| 刷新统计后计划突变 | 自动重算阈值（变更超约 10%） | [`01_cost_model.md`](01_cost_model.md) |
| `WHERE varchar_col = 123` 不走索引 | 类型聚合提升到 REAL 域 | [`21_collation_index_usability.md`](21_collation_index_usability.md) |

### 4.2 改写没生效

| 症状 | 根因 | 深入 |
|---|---|---|
| semi-join 没生效 | 16 条准入不满足；或 LooseScan 因"表交织"被否 | [`logical/03_semijoin.md`](logical/03_semijoin.md)、[`17_optimizer_decisions.md`](17_optimizer_decisions.md) |
| derived / view 没 merge | 12 步里某步不满足（聚合、UNION、LIMIT…）；或外层上下文否决（连 `ALGORITHM=MERGE` 也救不了） | [`logical/04_logical_join.md`](logical/04_logical_join.md)、[`20_view_resolution.md`](20_view_resolution.md) |
| `Using temporary; Using filesort` 消不掉 | GROUP BY 实现是纯规则判定，代价不参与 | [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) |
| `SQL_BIG_RESULT` 反而变慢 | 它的实现是"关掉用索引做 GROUP BY"，把索引扫描逼成全表扫+filesort | [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) |

### 4.3 计划不稳定

| 症状 | 根因 | 深入 |
|---|---|---|
| 同一条 SQL 两次执行计划不同 | 无计划缓存，统计信息在两次之间被更新 | [`../12_prepared_statement.md`](../12_prepared_statement.md)、[`14_plan_stability.md`](14_plan_stability.md) |
| hypergraph 与经典优化器结果差异大 | 两套代价模型量纲不同（hypergraph 的 `constexpr` 全 0.1） | [`physical/18_hypergraph_advanced.md`](physical/18_hypergraph_advanced.md) |
| UNION 加了 ORDER BY 就变慢 | 顶层 ORDER BY 强制物化（源码注释自陈是遗留决定） | [`19_set_operation.md`](19_set_operation.md) |

### 4.4 升级相关

| 症状 | 根因 | 深入 |
|---|---|---|
| 5.7 → 8.0 后 CHAR 列行为变化 | 默认 collation 从 PAD SPACE 改为 NO PAD | [`21_collation_index_usability.md`](21_collation_index_usability.md) |
| 5.7 → 8.0 后某些 GROUP BY 顺序变了 | 8.0 移除了 GROUP BY 的隐式排序 | [`../runtime/01_filesort.md`](../runtime/01_filesort.md)、[`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) |
| 8.0.22 后 EXPLAIN 输出格式变了 | `EXPLAIN FORMAT=tree` 引入 | [`../11_explain_and_trace.md`](../../../infra/observability/explain_trace.md) |

---

## 可观测性

### 判断"这个行为属于哪个版本"

1. 先用 `SELECT version();` 确认
2. 查本仓库源码（`MYSQL_VERSION` 文件给出准确版本）
3. 对照第一节的时间线

### grep 出本仓库所有的 WL 引用

```
grep -rn "WL#" sql/ | head -50
```

★ 这是**最可靠的 WL 编号来源**：源码注释里的编号旁通常有说明，比外部资料更可信。

### 查看本仓库的准确版本

仓库根目录的 `MYSQL_VERSION` 文件（本仓库为 8.0.39）。

**★★ 版本证据链（三处，缺一不可）**

**① `MYSQL_VERSION`（根目录，唯一的真实版本源）**：

```
MYSQL_VERSION_MAJOR=8
MYSQL_VERSION_MINOR=0
MYSQL_VERSION_PATCH=39
MYSQL_VERSION_EXTRA=
MYSQL_VERSION_STABILITY="LTS"
```

⚠️ **`include/mysql_version.h` 在仓库里不存在**——只有模板 `mysql_version.h.in`，里面全是 cmake 占位符：

```c
#define MYSQL_SERVER_VERSION       "@VERSION@"
#define MYSQL_VERSION_ID            @MYSQL_VERSION_ID@
#define MYSQL_VERSION_STABILITY     @MYSQL_VERSION_STABILITY@
```

**② `cmake/mysql_version.cmake:102` —— `MYSQL_VERSION_ID` 的唯一算出点**：

```cmake
  MATH(EXPR MYSQL_VERSION_ID
    "10000*${MAJOR_VERSION} + 100*${MINOR_VERSION} + ${PATCH_VERSION}")
```

> ★ 8.0.39 → `MYSQL_VERSION_ID = 10000*8 + 100*0 + 39 = **80039**`。
> ⚠️ `CMakeLists.txt` 里 grep `MYSQL_VERSION_ID` **0 命中**——它靠 `INCLUDE(mysql_version)`（`:587`）间接引入。

**③ `sql/dd/dd_version.h:210` —— DD 版本 ≠ 服务器版本**：

```cpp
static const uint DD_VERSION = 80023;
static_assert(DD_VERSION <= MYSQL_VERSION_ID,
              "This release can not use a version number from the future");
```

> ★★ **8.0.39 的 DD schema 停在 80023（8.0.23 发布）**——即 DD schema 从 8.0.23 起**连续 16 个补丁版未变**。`static_assert` 保证"DD 不能来自未来"。

---

## ★★ 五、本仓库 WL# 引用总盘点（按出现次数）

全仓 grep `WL#[0-9]+`：**≥ 3627 行命中**（工具输出上限截断，真实总量更大）。下表是**逐个 grep 得到的精确计数**：

| WL# | 全仓命中 | 代码目录 | `mysql-test/` | 主题（源码自陈） |
|---|---:|---:|---:|---|
| **WL#6570** | **39** | 33 | 6 | prepare once / execute many |
| WL#6599 | 82 | 4 | 78 | New Data Dictionary + I_S 集成 |
| WL#5561 | 41 | 1 | 40 | semijoin（**hypergraph 目录下唯一引用**） |
| WL#7743 | 41 | 4 | 37 | 原子 DDL（"changes to DDL-related"） |
| WL#6049 | 18 | 10 | 8 | 外键 MDL |
| WL#7909 | 7 | 4 | 3 | Server-side JSON functions |
| WL#929 | 6 | 2 | 4 | CHECK CONSTRAINTS |
| WL#751 | 5 | 4 | 1 | Recoding of error messages |
| WL#5257 | 5 | 4 | 1 | Optimizer trace 首个 API |
| WL#9533 | 4 | 4 | 0 | InnoDB 内部 dict cache 失效 |
| WL#6391 | 4 | 2 | 2 | DD 表访问矩阵 |
| WL#6629 | 3 | 3 | 0 | `check is_7bit` |
| WL#11320 | 3 | 3 | 0 | 存储引擎 commit hook |
| WL#5800 | 2 | 2 | 0 | `Item_cond` 相关 |
| WL#3404 | 2 | 2 | 0 | auto_increment 间隔 |
| WL#2623 | 2 | 2 | 0 | MRR / NDB handler |

> ★ 只有 **`components/` 下 2 个文件**含 WL#：`logging/log_sink_test.cc`（WL#9651，5 处，只在测试断言里留名）与 `test/test_mysql_runtime_error.cc:110`（`wl#11080`，**全仓唯一小写写法**）。

### 几个"活得最久"的活化石

**① WL#9533：整段代码被 `#ifdef` 关掉**（`sql/sql_table.cc:1509`，4 处同文）：

```cpp
#ifdef DISABLED_UNTIL_WL9533
    /*
      TODO: Simply removing entries from InnoDB internal cache breaks
            its FK checking logic at the moment. This is to be solved
            as part of WL#9533. We might have to replace invalidation
            with cache update to do this.
    */
    if ((parent_it.second)->dict_cache_reset)
```

> ★ 宏名就叫 `DISABLED_UNTIL_WL9533`——**到 8.0.39 仍未完成，代码是"活着的死代码"**。

**② WL#7743：DD 重命名顺序被反转的第一手说明**（`sql/sql_table.cc:10771`）：

```cpp
  /*
    Note that before WL#7743 we have renamed table in the data-dictionary
    before renaming it in storage engine. However with WL#7743 engines
    supporting atomic DDL are allowed to update dd::Table object describing
    new version of table in handler::rename_table(). Hence it should saved
    after this call.
```

配合 `sql/dd/dd_kill_immunizer.h:41` 的自陈：

```cpp
  Note:
    DD operations are made immune to KILL operations till WL#7743 and WL#7016
    are implemented. So as part of these WL's DD_kill_immunizer should be
    removed.
```

> ★ `DD_kill_immunizer` 至今仍在 8.0.39——它是等 WL#7743/7016 落地的**临时补丁**。

**③ WL#5257：优化器 trace 的出生证**（5 处全在文件头）：

```cpp
  API for the Optimizer trace (WL#5257)                    /* sql/opt_trace.h:49 */
   Implementation of the Optimizer trace API (WL#5257)      /* sql/opt_trace.cc:26 */
```

**④ WL#6049：InnoDB 侧把"外键元数据锁缺失"当已知债一路背到 8.0.39**（10 处）：

```cpp
  dictionary cache (work around the lack of WL#6049). */    /* handler0alter.cc:6780 */
      because WL#6049 (FK MDL) has not been                 /* handler0alter.cc:7707 */
```

**⑤ WL#3404：auto_increment 只保留"第一个区间"**（`sql/discrete_interval.h:93`）：

```cpp
   beginning of the first interval (because WL#3404 is not yet
   anything else than the first interval, in the list.  When WL#3404 is
```

**⑥ 唯一把"废弃 → 移除"版本计划写死在源码里的**（`sql/sql_show.cc:4964`）：

```cpp
  /* INFORMATION_SCHEMA.TABLESPACES is deprecated in 8.0 by WL#14064.
   * This should be removed in 9.0 (or next GA) by WL#14065 */
```

### ★★ 版本 → WL 对照表：`sql/dd/dd_version.h`（本树最完整的演进记录）

```cpp
  The data dictionary version number is the MySQL server version number
  of the first MySQL server version that published a given database schema.
  The format is Mmmdd with M=Major, m=minor, d=dot, so that MySQL 8.0.4 is
  encoded as 80004.
```

| 行号 | 版本 | 备注 |
|---|---|---|
| `:63` | `Published in 8.0.3-RC` | |
| `:65` | `Introduced in MySQL 8.0.0 by WL#6378. Never published in a GA version.` | |
| `:74` | `80004: Published in 8.0.4-RC` | |
| `:103` | `80011: Published in 8.0 GA` | |
| `:128` | `80012: Published in 8.0.12` | |
| `:136` | `80013: Published in 8.0.13` | |
| `:143` | `80014: Published in 8.0.14` | |
| `:150` | `80015: **Not published.** DD version still at 80014 in server 8.0.15` | ★ 跳版 |
| `:155` | `80016: Published in 8.0.16` | `- WL#929 - CHECK CONSTRAINTS` |
| `:167` | `80017: Published in 8.0.17` | |
| `:175` | `80021: Published in 8.0.21` | ★ 18/19/20 未单独发 |
| `:188` | `80022: Published in 8.0.22` | |
| `:195` | `80023: Current` | ★ 8.0.39 停在这里 |
| `:203` | `80024: Next DD version number after the previous is public` | 未激活 |

> ★★ 这是整棵树里**最完整的一份"版本 → worklog"对照表**。注意两处跳版：**8.0.15 明确标注 "Not published"**，**8.0.18/19/20 未单独发 DD 版本**。

### ★ 版本演进在源码里的其它痕迹

| 位置 | 原文 |
|---|---|
| `sql/dd_table_share.cc:1361` | `// Beginning in 8.0.12 HA_PACK_KEY and HA_BINARY_PACK_KEY are no longer set` |
| `sql/auth/auth_common.h:427` | `// Added in 8.0.13` / `// Added in 8.0.14`（虚函数表末尾加新列） |
| `sql/sql_table.cc:7727` | `introduced in version 8.0.18`（函数参数就叫 `master_version`） |
| `sql/log_event.cc:4268` | `All the SQL_MODEs ... were removed in 8.0.5` |
| `sql/sys_vars.cc:7685` | `option is set to BEFORE_8_0_26 ...`（8.0.26 改了 PFS 线程名，专门造兼容开关） |
| `sql/server_component/clone_protocol_service.cc:259` | `constexpr unsigned long CLONE_BACKPORT_VERSION = 37;` ★ 罕见：版本号写进常量 |
| `sql/join_optimizer/interesting_orders.cc:97` | `As rough reference: As of 8.0.26, ...` |

### ★★ hypergraph 优化器：源码里几乎没有版本标记

| 检查项 | 结果 |
|---|---|
| `sql/join_optimizer/` 全目录 grep `WL#\d+` | **只有 1 处**：`join_optimizer.cc:3300` 的 **WL#5561**，且讲的是 **semijoin + outer join 转换**，不是 hypergraph |
| `sql/join_optimizer/` 全目录 grep `@todo` | **0 命中** |
| 该目录下唯一的 MySQL 版本号 | `interesting_orders.cc:97` 的 `As of 8.0.26`（只是性能参照点） |
| INTERSECT / EXCEPT 的引入版本 | **源码里无版本标记**（`sql/` 下 grep `8\.0\.31` **0 命中**） |

> ★★ **新优化器与老优化器记录"未完成项"的方式完全不同**：老优化器用 `@todo`（`sql_optimizer.cc` 14 处、`sql_planner.cc` 8 处、`sql_select.cc` 8 处），**hypergraph 一个 @todo 都没有**——它改用 `@file` 里的 limitations 列表 + MTR 的 `--skip_if_hypergraph`。
>
> ★ 所以 **hypergraph 的"worklog 未完成清单"真实位置是 MTR**：
>
> ```
> --skip_if_hypergraph  # Does not use index range scan yet. TODO: Enable it after WL#14806.
> ```
> （`mysql-test/suite/innodb/t/bug32554667.test:16` —— **WL#14806 = index range scan**，8.0.39 的 hypergraph 还不支持。）

**★ hypergraph 的动机与论文出处**（43 个文件中**只有 7 个**有 `@file` 块，其余只有版权头）：

```cpp
/* sql/join_optimizer/join_optimizer.h:27 —— 自陈现状 */
  It is intended to eventually take over completely from the older join
  optimizer based on prefix search (sql_planner.cc and related code),
  and is nearly feature complete, but is currently in the early stages
  with a very simplistic cost model and certain limitations.
  The most notable ones are that we do not support:
    - Hints (except STRAIGHT_JOIN).
    - TRADITIONAL and JSON formats for EXPLAIN (use FORMAT=tree).
    - UPDATE.
```

```cpp
/* sql/join_optimizer/subgraph_enumeration.h:27 —— 算法与论文 */
  The algorithm is described in the paper “Dynamic Programming Strikes
  Back” by Neumann and Moerkotte. ... Some critical details are still
  missing, which we've had to fill in ourselves.
```

```cpp
/* sql/join_optimizer/graph_simplification.h:27 —— 与 LinDP++ 对比 */
  This isn't necessarily an optimal or even particularly good algorithm;
  e.g. LinDP++ [Rad19] claims significantly better results, especially
  on joins that are 40 tables or more.
```

```cpp
/* sql/join_optimizer/node_map.h:31 —— 硬上限 */
  Since our graphs can never have more than 61 tables, node sets and edge
  lists are implemented using 64-bit bit sets.
```

> ★ 其余三篇论文引用：`interesting_orders.h:27`（[Neu04]/[Neu04b]/[Sim96]）、`online_cycle_finder.h:33`（[Pea03]在线环检测）。

### ★ `@todo` 的分布（Top 10）

| 排名 | 文件 | 次数 |
|---:|---|---:|
| 1 | `sql/sql_optimizer.cc` | **14** |
| 2 | `sql/binlog.cc` | 11 |
| 2 | `sql/sql_base.cc` | 11 |
| 2 | `sql/sql_resolver.cc` | 11 |
| 5 | `sql/rpl_replica.cc` | 10 |
| 5 | `sql/sql_lex.h` | 10 |
| 7 | `sql/sql_update.cc` / `sql/item.h` | 9 |
| 9 | `sql/handler.cc` / `sql/item.cc` / `sql/item_cmpfunc.cc` / `sql/sql_class.h` / **`sql/sql_planner.cc`** / `sql/sql_select.cc` / `sql/log_event.cc` | 各 8 |

> ★ 关键观察：**优化器三件套合计 30 处**（`sql_optimizer.cc` 14 + `sql_planner.cc` 8 + `sql_select.cc` 8），而 `sql/join_optimizer/` **0 处**。`sql/query_term.cc` 也是 **0 处**——INTERSECT/EXCEPT 新代码与老代码在这点上差别显著。
> ★ **`sql_planner.cc` 的 8 个 @todo 没有一个提到 WL#**——老优化器已经"停止标记 worklog"。

---

## Misc

### 已知缺陷（跨版本的长期问题）

- **两套表示并存**：`Query_term` 树与 `Query_expression::first_query_block()` 链表并存（源码 TODO 计划合并）。
- **`Query_block` 与 `JOIN` 的生命周期边界不干净**（WL#6570 未完成）。
- **hypergraph 代价模型过粗**（全部常量 0.1），与经典优化器的代价不可直接比较。
- **无计划缓存 / 无 SPM**：社区版没有"固定历史好计划"的能力，只有 hint 这种手工手段。
- **`test_skip_sort` 的判定式作者自己都解释不清**（源码 TODO 至今未解决）。

### 社区边界澄清

- **hypergraph 优化器是实验特性**：源码注释明说"故意不写进文档"，默认 off，行为可能在后续版本变化。
- **Worklog 是 Oracle 内部开发任务系统**：WL 编号本身没有公开全文，只能通过源码注释与 release notes 交叉验证。
- **版本时间线里的"大致版本"**：部分特性（如直方图、降序索引）的精确引入版本应以官方 release notes 为准；本篇只列有把握的大版本节点。

### ★★ 真名勘误与"查不到"清单

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| WL#6570 在源码里出现 33 次 | **39 次**（`sql/` 33 + `mysql-test/` 6）。"33"大概率只 grep 了 `sql/` |
| `include/mysql_version.h` 里写着 8.0.39 | **该文件不存在**。只有模板 `mysql_version.h.in`（全 `@VERSION@` 占位符）；真实数字在根目录 `MYSQL_VERSION` |
| `MYSQL_VERSION_ID` 定义在 `CMakeLists.txt` | **0 命中**。定义在 **`cmake/mysql_version.cmake:102`**：`10000*M + 100*m + p` ⇒ 8.0.39 = **80039** |
| DD 版本号 = 服务器版本号 | **不是**。`DD_VERSION = 80023`（8.0.23 起未变），靠 `static_assert(DD_VERSION <= MYSQL_VERSION_ID)` 解耦 |
| INTERSECT / EXCEPT 源码里有版本标记（8.0.31） | **没有**。`sql/` 下 grep `8\.0\.31` **0 命中** |
| hypergraph 优化器有专门的 WL# 注释 | **没有**。`sql/join_optimizer/` 全目录仅 1 处 WL#（`join_optimizer.cc:3300` 的 **WL#5561**），且讲的是 semijoin，不是 hypergraph |
| hypergraph 各文件头都有 @file 说明 | **43 个文件中只有 7 个**有 `@file` 块，其余只有 GPL 版权头 |
| hypergraph 目录里有 @todo | **0 命中**。它的"未完成清单"在 **MTR 的 `--skip_if_hypergraph`** |
| `@todo` 集中在优化器 | `sql_optimizer.cc` 确以 14 处居首，但 `binlog.cc`/`sql_base.cc`/`sql_resolver.cc` 各 11 处紧随 |
| `sql/query_term.cc` 里有 @todo | **0 命中** |
| `sql_planner.cc` 的 @todo 会提 WL# | **8 个 @todo 无一提 WL#**——老优化器已"停止标记 worklog" |
| 全仓 WL# 总数可精确统计 | 工具输出上限导致只能给出**下界 ≥ 3627**；上表 16 个重点 WL 的计数是逐个 grep 的**精确值** |
| `WL#6059` 已完成 | 源码在 `sql/table.h:3236` 把它列为 **WL#6570 的前置项**，未给完成状态 |

### 扩展点

| 想做什么 | 怎么做 |
|---|---|
| 查某个 WL 的更多细节 | 在源码里 grep 该编号（注释里通常有上下文）；结合 MySQL release notes |
| 确认某行为在哪个版本引入 | 用 `git log -S` 在 MySQL 源码仓库里搜相关函数/常量（本仓库有完整 git 历史） |
| 追踪某个 TODO 是否已被修复 | grep 该 TODO 文本，若消失说明在新版本被解决 |

---

## 参考

**官方资源**

- MySQL 8.0 Release Notes（各版本优化器相关改动的第一手来源）
- MySQL 8.0 Reference Manual, "What Is New in MySQL 8.0"
- MySQL 源码仓库的 git 历史（`git log -S <symbol>` 可精确追溯某个优化的引入时间与 Worklog 号）

**源码出处（本篇所有 WL 编号的来源）**

- `sql/opt_trace.cc` / `opt_trace.h` / `opt_trace2server.cc` —— WL#5257
- `sql/aggregate_check.h` —— WL#2489
- `sql/join_optimizer/join_optimizer.cc` —— WL#5561
- `sql/item_cmpfunc.h` —— WL#5800、WL#12108
- `sql/item_func.cc` —— WL#7384、WL#6570
- `sql/item.cc`、`sql/sql_base.cc`、`sql/sql_tmp_table.cc`、`sql/sql_prepare.cc`、`sql/table.h` —— WL#6570
- `sql/dd_sql_view.cc` —— WL#9446
- `sql/sql_yacc.yy` —— WL#2111

**相关文档**

- [`00_overview.md`](00_overview.md) —— 四阶段框架与两条优化路径
- [`14_plan_stability.md`](14_plan_stability.md) —— 计划稳定性与能力边界
- [`17_optimizer_decisions.md`](17_optimizer_decisions.md) —— 决策全景（含 bad case 定位入口）
- [`../05_contextualize.md`](../05_contextualize.md) —— `Query_term` 重构
- [`../08_access_path.md`](../08_access_path.md) —— AccessPath 改造（8.0.22）

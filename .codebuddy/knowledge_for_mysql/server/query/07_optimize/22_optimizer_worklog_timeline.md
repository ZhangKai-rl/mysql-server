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
| **8.0.20** | **`EXPLAIN ANALYZE`** | 11_explain_and_trace |
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
| 8.0.22 后 EXPLAIN 输出格式变了 | `EXPLAIN FORMAT=tree` 引入 | [`../11_explain_and_trace.md`](../11_explain_and_trace.md) |

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
- [`../08_access_path/README.md`](../08_access_path/README.md) —— AccessPath 改造（8.0.22）

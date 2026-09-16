# 03 CTE 与递归 CTE：WITH 的解析与执行（算法级）

> 基于 MySQL 8.0.39 源码。本篇覆盖：`WITH` 如何解析与存储、非递归 CTE 为何就是 derived table、merge 与 materialize 的决策链、**物化表的自动建索引**、递归 CTE 的 seed + 收敛循环、共享物化，以及递归的陷阱与性能特征。
>
> **边界**：本篇讲 CTE 的执行期实现。优化器侧"CTE 为何不能 merge、与 derived 的结构差异"见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)；物化用的临时表本身见 [`07_temptable.md`](07_temptable.md)。

## 目录

- [先有个整体印象](#先有个整体印象)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、WITH 的解析与存储](#一with-的解析与存储)
- [二、非递归 CTE：就是 derived table](#二非递归-cte就是-derived-table)
- [三、物化表的自动建索引](#三物化表的自动建索引)
- [四、递归 CTE 的核心算法](#四递归-cte-的核心算法)
- [五、共享物化与重扫](#五共享物化与重扫)
- [六、递归的限制、陷阱与性能特征](#六递归的限制陷阱与性能特征)
- [七、深潜补充：克隆表、校验清单、递归的三个细节](#七深潜补充克隆表校验清单递归的三个细节)
- [参考](#参考)

---

## 先有个整体印象

> ⚠️ **版本事实**：8.0.39 **没有** `sql/sql_cte.h/.cc`。CTE 逻辑已分散到 `sql_derived.cc`、`sql_union.cc`、`sql_parse.cc`、`parse_tree_nodes.cc`。后解析数据结构叫 `Common_table_expr`。

CTE 的本质：**非递归 CTE 在解析层就被做成了 derived table**（复用 merge/materialize 全部机制），差别只在"多次引用共享一次物化"；**递归 CTE** 则多出"seed + 递归步收敛循环"，核心是 `FollowTailIterator` 读"上一轮新增的行"。

```
非递归 CTE ──▶ 就是 derived table（merge 或 materialize）
递归 CTE   ──▶ MaterializeIterator::MaterializeRecursive：
                 1. seed（anchor）物化一次
                 2. 递归块反复物化，直到不再新增行（收敛）
```

---

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 为什么非递归 CTE 要走 derived table 这条路

CTE 和 derived table（`FROM (SELECT ...)`）在语义上几乎等价。MySQL 的选择是**不另起炉灶**——`find_common_table_expr` 找到 CTE 定义后，直接用它构造一个 `PT_subquery` 并标记为 derived table，从而完整复用已有的 `merge_derived` / `materialize_derived` 机制。

**收益**：优化器的 merge 决策、物化、临时表管理、共享引用全部白拿，代码量极小。
**代价**：CTE 也一并继承了 derived table 的全部限制（不是所有 CTE 都能 merge）。

#### 2. merge 还是 materialize：一场经典的权衡

非递归 CTE（derived table）有两种执行方式：

| | **merge**（合并进外层查询） | **materialize**（物化成临时表） |
|---|---|---|
| 做法 | 把 CTE 的定义**展开**进外层 WHERE/JOIN，当成一个整体优化 | 先执行 CTE 把结果写进临时表，再当普通表访问 |
| 优点 | 外层谓词可**下推**进 CTE、能用一个全局 join order、可能避免物化 | 只执行一次（多次引用共享）、阻断优化爆炸 |
| 缺点 | 复杂 CTE 展开后会拖垮 join 搜索空间 | 谓词下推被阻断、额外写临时表、可能因降级到磁盘而变慢 |

MySQL 默认**倾向 merge**（`derived_merge` 默认 ON），只要结构允许就合并。这与 PostgreSQL 12 之前"CTE 总是物化"形成鲜明对比（见他库对比）。

#### 3. 递归 CTE 为什么必须是"迭代 + 临时表"，而不是真递归

递归 CTE（`WITH RECURSIVE`）在语义上是**不动点迭代**：从 anchor 出发反复应用递归规则，直到不再产生新行。

MySQL 的实现是**执行期的迭代循环**（`MaterializeRecursive` 里的 `do...while`），不是调用栈意义上的递归：

- 每一轮把新产生的行**追加**进同一张物化临时表
- 递归引用用一个特殊迭代器 `FollowTailIterator` 只读"上一轮新增的行"
- 收敛条件：某一轮没有新增任何行

**为什么用迭代而不是递归**：迭代让"已算出的行"天然落在一张表里，递归引用就是一次顺序扫描；同时深度由数据决定而非调用栈，避免了栈溢出。代价是**每轮都要重新扫描新增行**，复杂度随数据规模上升（见性能特征）。

#### 4. 为什么没有环检测：一个明确的设计取舍

如果递归产生环（如 `SELECT n+1 FROM cte WHERE ...` 永远能产生新行），MySQL **不会**检测"这一行是不是已经出现过"。它只靠 `cte_max_recursion_depth`（默认 1000）在**轮数**上兜底。

**这是个权衡**：

- **检测环的代价**：每产生一行都要在整张已物化表里查找是否已存在（O(n) 甚至需要额外索引），对合法的深递归是纯浪费
- **不检测的代价**：有环时只能跑到深度上限然后报错，浪费了一千轮的计算

MySQL 选择后者——把"写对递归终止条件"的责任交给用户。（PostgreSQL 同样不做环检测。）

**顺带的坑**：因为不检测重复行，如果递归逻辑本身会重复产生已有行（且不是环），MySQL 会**一直产生直到深度上限**——即使逻辑上"早该停了"。

### 理论溯源

递归 CTE 来自 **SQL:1999 标准**的递归查询（recursive query），理论基础是**最小不动点（least fixed point, LFP）**语义：

```
结果 = 反复应用递归规则，直到结果集不再增长（到达不动点）
```

这也是为什么 SQL 标准要求递归 CTE 必须是 `UNION`（或 `UNION ALL`）+ anchor/递归两段结构——它在形式上是标准的 LFP 迭代。

`WITH RECURSIVE` 的这个结构约束在 MySQL 里有硬校验（见后面"结构校验"）。

### 算法与数据结构

- **`Common_table_expr`**：CTE 的运行时结构，三个关键成员——`references`（定义之外的非递归引用）、`recursive`（是否递归）、`tmp_tables`（共享物化的所有 `Table_ref`）
- **`FollowTailIterator`**：递归 CTE 的核心迭代器，读"上一轮新增的行"，且**不让底层游标真正撞 EOF**
- **`MaterializeIterator`**：物化的通用算子，`is_recursive()` 时走 `MaterializeRecursive()`
- **`Derived_key`**：物化表上自动建立的"可用键"描述（见下一节）

### 他库对比与演进动机

| | MySQL 8.0 | PostgreSQL | SQL Server |
|---|---|---|---|
| 非递归 CTE 默认 | **倾向 merge**（`derived_merge=ON`） | **12 之前总是物化**（著名的 optimization fence，谓词下推被阻断）；12 起可 `NOT MATERIALIZED` | 通常展开（类似 merge） |
| 递归实现 | 迭代 + 临时表 + `FollowTailIterator` | 迭代 + tuplestore（WorkTable Scan） | 迭代 + 栈式 spool |
| 环检测 | ❌ 无（靠深度上限） | ❌ 无 | ❌ 无 |
| 递归深度限制 | `cte_max_recursion_depth`（默认 1000） | 无限制（靠 `max_stack_depth`/内存） | `MAXRECURSION`（默认 100，可用 hint 改到 32767） |

> **PG 的教训值得记住**：PG 12 之前"CTE 总是物化"导致大量性能问题（外层谓词下推不进 CTE，CTE 被全量执行），社区为此呼吁多年，12 才允许 `NOT MATERIALIZED`。MySQL 从一开始就走 merge 优先，避开了这个坑——但代价是复杂 CTE 展开后可能拖慢 join 搜索。

### 版本演进与能力清单

**引入历史：8.0 首次引入，5.x 完全没有。**

源码最硬的证据是一处注释直称 CTE 为"新特性"（`composite_iterators.cc` 的递归物化里，讲为什么递归 CTE 的截断**不继承旧 SELECT 的宽松行为**）：

```cpp
// ...as WITH RECURSIVE is a new feature we don't have to carry the
// permissiveness of the past, so we send an error even if in non-strict mode.
```

"新特性、不必背负历史包袱"——这句话同时证明了两点：CTE 是后加的、且设计者有意让它更严格。开发按三个工作日志推进（测试文件头可证）：

| 工作日志 | 内容 |
|---|---|
| `WL#883` | 非递归 `WITH`（`with_non_recursive.inc` 首行） |
| `WL#3634` | 递归 CTE（`with_recursive_bugs.test` 反复出现） |
| `WL#9248` | 递归所需的 InnoDB 临时表配套改动 |

`cte_max_recursion_depth` 由 `Bug#26136509` 引入（测试文件头明写 "ADD A MAX_RECURSION VARIABLE TO LIMIT RECURSION IN CTES"）。

⚠️ **诚实标注**：具体到 8.0.14 / 8.0.19 / 8.0.22 这些小版本的精确改动，**本源码树没有版本号标注**（`8.0.` 正则搜 `with_*` 测试 0 命中），需结合官方 Release Notes。源码能证明的是"能力落点"，不能证明"哪个小版本引入"。

**关键纠偏：非递归 CTE 不是"总是物化"。**

这是最容易误解的一点。MySQL 从 CTE 诞生起就走 derived table 的 merge/materialize 决策链——**非递归 CTE 默认倾向 merge，`derived_merge` 开关和 `MERGE/NO_MERGE` hint 都对它生效**。`Table_ref::is_mergeable()` 里甚至专门为 CTE 写了分支：

```cpp
// If the table's content is non-deterministic and the query references it
// multiple times, merging it has the risk of creating different contents.
Common_table_expr *cte = common_table_expr();
if (cte != nullptr && cte->references.size() >= 2 &&
    derived->uncacheable & UNCACHEABLE_RAND)
  return false;   // 多次引用 + RAND 才禁止 merge
```

只有**递归 CTE 必须物化**（迭代 + 临时表，`derived_merge` 不参与）。

**8.0 后期可源码佐证的能力演进**（版本号需外部 changelog 交叉定位）：

- **EXPLAIN 专门显示 CTE**：`Common_table_expr::name` 字段注释 "Used only for EXPLAIN FORMAT=tree"；`explain_access_path.cc` 打印 "Materialize CTE / recursive CTE / union CTE"；`with_explain.test` 明确"多次引用只在第一处展开、其余显示 `<derivedN>`，让用户看懂是单次物化"。
- **hypergraph 优化器改变 CTE 计划**：测试里大量 `--skip_if_hypergraph # Chooses a different plan`，是 8.0.23 引入 hypergraph 后 CTE 执行路径变化的源码可见痕迹。

**语法能力清单（8.0.39 逐一核实）**：

| 能力 | 支持 | 说明 |
|---|---|---|
| 普通 CTE / 多 CTE（逗号分隔） | ✅ | `with_list: with_list ',' common_table_expr` |
| `WITH RECURSIVE` | ✅ | 与普通 WITH 同一条 `with_clause` 规则，`RECURSIVE_SYM` 只是可选关键字 |
| `WITH RECURSIVE` 但无递归成员 | ✅ | 合法（测试注释明说 "ok to have the RECURSIVE word without any recursive member"） |
| WITH 前置 / 内嵌（子查询里再 WITH） | ✅ | 内嵌时前向引用仍报 `ER_NO_SUCH_TABLE` |
| CTE 引用 CTE | ✅ | 前向引用禁止（`ER_NO_SUCH_TABLE`） |
| 列名覆盖 `WITH cte(col1,col2)` | ✅ | 空列名表 `()` 报 parse error；列数不符报 `ER_VIEW_WRONG_LIST` |
| 同一 CTE 多次引用 `FROM cte a, cte b` | ✅ | 解析期 `reparse_common_table_expr` 克隆子树，执行期共享物化、各自游标 |
| `SEARCH DEPTH FIRST / BREADTH FIRST` | ❌ | `lex.h` 无 SEARCH/DEPTH/BREADTH 关键字 |
| `CYCLE` 子句 | ❌ | `lex.h` 无 CYCLE 关键字；深度优先要 `STRAIGHT_JOIN`+`ORDER BY PATH` 手写 |
| 互递归（mutual recursion） | ❌ | 报 `ER_NO_SUCH_TABLE`，"cycles must have one node only" |
| `WITH ... UPDATE/DELETE` | ✅ | `update_stmt`/`delete_stmt` 以 `opt_with_clause` 开头 |
| `WITH ... INSERT/REPLACE` | ❌ | `insert_stmt`/`replace_stmt` 直接以 `INSERT_SYM`/`REPLACE_SYM` 开头，无 `opt_with_clause` |

**"多次引用"限制的真正边界**：非递归 CTE 可随意多次引用（克隆 + 共享物化）；但**递归块内**递归表只能被引用**一次**、且不能出现在子查询里（`ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE`，落点 `Table_ref::set_recursive_reference()`）。这与 SQL:1999 的约束一致。

**演进遗留的 TODO（源码自述未完成方向）**：

1. 共享物化的 lateral 失效检查目前只判断"是否写入了新行"，`composite_iterators.cc` 注释：*"TODO: It would be better ... to check the actual column values instead of just whether we've seen any new rows"*——暗示未来可能朝**值级环检测**演进。（LATERAL 机制的系统讲解见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md#六lateral让派生表看见同层兄弟表)）
2. `MAXRECURSION` 类 hint 未实现：*"a MAX_RECURSION hint (if we featured one) may interrupt"*——"if we featured one" 明示 MySQL 没有 SQL Server 那种 `OPTION (MAXRECURSION n)`，只有 `cte_max_recursion_depth`（可经 `SET_VAR` hint 改）。
3. EXPLAIN 打印位置：*"TODO(sgunders): Consider printing CTE query plans on the top level ... instead?"*

---

## 一、WITH 的解析与存储

### 1.1 WITH 挂到 Query_expression

```cpp
bool PT_with_clause::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  pc->select->master_query_expression()->m_with_clause = this;  // 挂到所属 QE
  return false;
}
```

WITH 是 `<query expression>` 的前缀，所以存到 `Query_expression::m_with_clause`（不是 query block）。

### 1.2 Common_table_expr：三个数组是理解执行的关键

```cpp
class Common_table_expr {
  Mem_root_array<Table_ref *> references;  // 定义之外的非递归引用
  bool recursive;                          // 是否递归 CTE
  Mem_root_array<Table_ref *> tmp_tables;  // 共享物化的所有 Table_ref
};
```

- `references`：CTE 定义**之外**的引用。
- `recursive`：标记为递归。
- `tmp_tables`：共享物化时指向同一临时表的所有 Table_ref（首个持有真正 create 出来的 TABLE）。

### 1.3 名字查找：find_common_table_expr

从当前 query block 沿 `master_query_expression()->m_with_clause` 逐层向外查找（支持 CTE 作用域嵌套）。找到后：

- **非递归引用**：用 CTE 定义构造 `PT_subquery`，标 `node->m_is_derived_table = true`——**复用派生表机制**。
- **递归引用**：塞入 `(select 0)` 的 dummy 定义占位（`reparse_common_table_expr`），先蒙混过 name resolution。

`make_subquery_node` 保证同一 CTE 被多次引用时各自有独立子树（`references.size() >= 2` 时重新 parse 一份）。

### 1.4 递归/前向引用判定：PT_with_clause::lookup

`m_most_inner_in_parsing` 记录当前正在 contextualize 的 CTE。在 WITH 列表从左到右扫描，一旦越过"正在解析的那个 CTE"（right_bound），再往后就是前向引用——被禁止，除非声明 `RECURSIVE` 且匹配的是它自己（自引用 = 递归引用）。

---

## 二、非递归 CTE：就是 derived table

**核心结论**：非递归 CTE 引用在解析层被做成了 derived table，完整复用 `merge_derived`（合并）或 `materialize_derived`（物化）。

- **merge 判定**：`Query_expression::is_mergeable()`——无 set-op、无 GROUP BY/HAVING/DISTINCT/LIMIT/窗口 才可 merge
- **与普通 derived table 的差别**：①所有引用共享一个 `Common_table_expr`；②多次引用时物化一次、共享

### merge 的完整决策链（优先级从高到低）

源码注释把优先级写得很清楚：

```cpp
  /*
    Check whether derived table is mergeable, and directives allow merging;
    priority order is:
    - ALGORITHM says MERGE or TEMPTABLE
    - hint specifies MERGE or NO_MERGE (=materialization)
    - optimizer_switch's derived_merge is ON and heuristic suggests merge
  */
  if (derived_table->algorithm == VIEW_ALGORITHM_TEMPTABLE ||
      !derived_query_expression->is_mergeable())
    return false;

  if (derived_table->algorithm == VIEW_ALGORITHM_UNDEFINED) {
    const bool merge_heuristic =
        (derived_table->is_view() || allow_merge_derived) &&
        derived_query_expression->merge_heuristic(thd->lex);
    if (!hint_table_state(thd, derived_table, DERIVED_MERGE_HINT_ENUM,
                          merge_heuristic ? OPTIMIZER_SWITCH_DERIVED_MERGE : 0))
      return false;
  }
```

**逐段解读**：

1. **`ALGORITHM=TEMPTABLE` 或结构不可 merge** ⇒ 直接物化（显式指令 + 结构限制最高优先级）
2. **`ALGORITHM=UNDEFINED`**（默认）时看两条：
   - `merge_heuristic`：启发式（视图或允许 merge 的 derived，且查询表达式本身适合合并）
   - 再经 **hint（`MERGE`/`NO_MERGE`）** 与 **`optimizer_switch=derived_merge`** 共同决定
3. 另有约束：`STRAIGHT_JOIN` 时不能 merge 含 semi-join nest 的查询块（会破坏连接顺序语义）

### 物化的执行入口

- `resolve_derived()`：prepare 期解析
- `create_materialized_table()` + `materialize_derived()`：真正建表并填充

---

## 三、物化表的自动建索引

物化的 derived/CTE 表**不是**一张只能全表扫描的笨表——MySQL 会根据外层对它的等值引用**自动建键**，让后续访问能走 ref。

`update_derived_keys` 的注释说明了用途：

```cpp
/**
  @brief
  Update derived table's list of possible keys

  @details
  This function creates/extends a list of possible keys for this derived
  table/view. For each table used by a value from the 'values' array the
  corresponding possible key is extended to include the 'field'.
  If there is no such possible key, then it is created. field's
  part_of_key bitmaps are updated accordingly.
  @see add_derived_key
*/
```

真正建键的 `add_derived_key`：

```cpp
static bool add_derived_key(THD *thd, List<Derived_key> &derived_key_list,
                            Field *field, table_map ref_by_tbl) {
  uint key = 0;
  Derived_key *entry = nullptr;
  List_iterator<Derived_key> ki(derived_key_list);

  /* Search for already existing possible key. */
  while ((entry = ki++)) {
    key++;
    if (ref_by_tbl) {
      /* Search for the entry for the specified table.*/
      if (entry->referenced_by & ref_by_tbl) break;
    } else {
      /*
        Search for the special entry that should contain fields referred
        from any table.
      */
      if (!entry->referenced_by) break;
    }
  }
  /* Add new possible key if nothing is found. */
  if (!entry) {
    key++;
    entry = new (thd->mem_root) Derived_key();
    if (!entry) return true;
    entry->referenced_by = ref_by_tbl;
    entry->used_fields.clear_all();
    if (derived_key_list.push_back(entry, thd->mem_root)) return true;
  }
  /* Don't create keys longer than REF access can use. */
  if (entry->used_fields.bits_set() < MAX_REF_PARTS) {
    field->part_of_key.set_bit(key - 1);
    field->set_flag(PART_KEY_FLAG);
    entry->used_fields.set_bit(field->field_index());
    entry->key_part_count++;
  }
  return false;
}
```

**机制要点**：

- **按"被哪个表引用"分组建键**：`referenced_by` 是表位图，对每个引用它的表单独维护一个键（`ref_by_tbl` 为 0 时是"任意表都能用"的通用键）
- **键不能无限长**：`used_fields.bits_set() < MAX_REF_PARTS`——超过 ref 访问能用的键部分数就不再加列，因为建了也用不上
- **三类字段跳过**（`update_derived_keys` 开头）：CREATE VIEW 的上下文分析、BLOB 字段、零长字段

**为什么重要**：如果没有这个机制，物化后的 CTE 每次被外层访问都要全表扫描，物化路径会比 merge 慢得多。有了它，`WHERE cte.col = t.col` 这类引用可以在物化表上走 ref。

> 这也解释了为什么"能 merge 就 merge"虽然是默认，但物化路径并没有被放弃——物化 + 自动建索引在某些情况下（尤其是多次引用共享时）反而更优。

---

## 四、递归 CTE 的核心算法

### 4.1 结构校验

遍历 UNION 的所有 query block，校验 SQL 标准约束：

- 必须是 UNION（`ER_CTE_RECURSIVE_REQUIRES_UNION`）
- 递归块必须在 UNION 下、禁止右嵌套 UNION
- 递归成员不允许 ORDER BY/LIMIT/DISTINCT
- 非递归块（anchor）必须排在递归块之前
- 至少要有 anchor

最后 `derived->first_recursive = last_non_recursive->next_query_block()` 指向第一个递归块。

### 4.2 prepare 阶段：建临时表 + 递归引用替换

```cpp
if (sl == first_recursive) {
  derived_table->setup_materialized_derived_tmp_table(thd);   // anchor 类型确定后建表
}
if (sl->recursive_reference)
  derived_table->common_table_expr()->substitute_recursive_reference(thd, sl);
```

`substitute_recursive_reference`：递归引用此前是 `(select 0)` 的 dummy derived table，这里克隆一个指向物化临时表的 TABLE 句柄，摘掉 dummy 的 query expression，变成"直接读临时表"的引用。

### 4.3 运行期：seed + 收敛循环

`MaterializeIterator` 检测到 `is_recursive()` 时走 `MaterializeRecursive()`，算法注释写明：

```
1. 物化所有非递归 query block（seed/anchor），一次。
2. 物化所有递归 query block。
3. 重复 2 直到没有 block 再写入任何行（收敛）。
   每次物化只看到上一轮新增的行（FollowTailIterator）。
```

核心循环：

```cpp
ha_rows stored_rows = 0;
// 把总行数指针交给每个递归 reader（FollowTailIterator）
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (qb.is_recursive_reference)
    qb.recursive_reader->set_stored_rows_pointer(&stored_rows);

// 第一步：物化 seed
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (!qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);

// 第二步+第三步：反复物化递归块直到收敛
ha_rows last_stored_rows;
do {
  last_stored_rows = stored_rows;
  for (const QueryBlock &qb : m_query_blocks_to_materialize)
    if (qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);
} while (stored_rows > last_stored_rows);   // 不再新增即收敛
```

### 4.4 FollowTailIterator：读"上一轮新增的行"

```cpp
int FollowTailIterator::Read() {
  if (m_read_rows == *m_stored_rows) {
    return -1;   // 已读到当前总行数 → EOF；关键是不让底层 cursor 真正撞 EOF，
                 // 否则 MEMORY/InnoDB 的 scan 会永久卡在 EOF，看不到后续插入的新行
  }
  if (m_read_rows == m_end_of_current_iteration) {
    if (++m_recursive_iteration_count > thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;   // 超深度报错
    }
    m_end_of_current_iteration = *m_stored_rows;  // 本轮终点 = 当前总行数
  }
  int err = table()->file->ha_rnd_next(m_record);  // 顺序读下一行
  if (err) return HandleError(err);
  ++m_read_rows;
  return 0;
}
```

精髓：

- `m_stored_rows` 指向物化器的总行数计数器。
- 每个递归步开始时 `m_end_of_current_iteration = *m_stored_rows` 记下"本步之前已有多少行"。
- 递归引用只顺序扫描 `[上一轮终点, 当前总数)` 区间，即**上一轮新产生**的行。
- 扫描位置追上总行数时**返回 EOF 而不推进底层游标**（关键！），从而下一轮写入新行后还能继续读。

### 4.5 深度限制

`cte_max_recursion_depth` 默认 **1000**，SESSION 级。

---

## 五、共享物化与重扫

### 5.1 多引用只物化一次：clone_tmp_table

同一 CTE 的第二个及后续引用不再 `create_tmp_table`，而是 `open_table_from_share` 基于同一个 `TABLE_SHARE` 克隆一个新的 TABLE 句柄——**共享底层数据、各自独立游标**。第一个是写者，克隆是读者。

`setup_materialized_derived_tmp_table`：`tmp_tables.size() > 0` 时复用，否则首次真正创建。

### 5.2 rematerialize 参数

`MaterializeIterator::Init`：

- 非相关 CTE `rematerialize=false`：多次引用共享一次物化，之后只重扫。
- 相关（lateral/correlated）CTE 或依赖外层值的派生表 `rematerialize=true`：每次 `Init()` 清空重建。

`use_shared_cte_materialization` 判定：`m_cte != nullptr && !m_rematerialize` 且 `tmp_tables` 里已有 `materialized==true` 的表，则复用不再物化。

### 5.3 FOLLOW_TAIL 访问路径

```cpp
if (range_scan != nullptr)      path = range_scan;
else if (table_ref->is_recursive_reference()) path = NewFollowTailAccessPath(...);  // 递归引用
else                            path = NewTableScanAccessPath(...);
```

`AccessPath::FOLLOW_TAIL` → `FollowTailIterator`。

---

## 六、递归的限制、陷阱与性能特征

### 6.1 UNION 与 UNION ALL 的语义差异（最容易踩的坑）

递归 CTE 中 `UNION` 与 `UNION ALL` 的差别比普通查询更关键：

- **`UNION ALL`**：每轮新产生的行全部追加，**不做去重**——可能产生重复行，也可能因此**永不收敛**
- **`UNION`（DISTINCT）**：每轮产生的新行会与已有行**去重**——这既是防重复的保障，也是**提前收敛**的常见原因

因为 MySQL 不做环检测，`UNION`（去重）实际上是用户手上**唯一的"防环"手段**：只要某一轮产生的行全都已存在，去重后新增为 0，循环立即收敛。反之用 `UNION ALL` 且递归逻辑会重复产生相同行时，会一直跑到 `cte_max_recursion_depth` 才报错。

因此源码对 `UNION DISTINCT` 与 `UNION ALL` 的**混用顺序**有硬校验：

```cpp
        if (sl == derived->last_distinct() && sl->next_query_block()) {
          // 例：anchor UNION ALL rec1 UNION DISTINCT rec2 UNION ALL rec3
          my_error(ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST, ...
                   " UNION DISTINCT then UNION ALL, in recursive "
```

即：**一旦出现 `UNION DISTINCT`，它之后不允许再跟 `UNION ALL`**（去重后再追加会破坏语义）。

### 6.2 列类型由 anchor 决定

递归 CTE 的列类型完全由**非递归块（anchor）**决定——递归块产出的值必须能装进 anchor 定义的类型。

这带来一个隐蔽问题：若 anchor 定义 `VARCHAR(10)` 而递归部分产出更长的值，**非严格模式下会截断**，而截断后的值可能永远无法收敛 ⇒ 无限递归。所以 MySQL 在递归物化时**强制严格模式**（`Strict_error_handler`），即使 session 不是严格模式也要报错。

### 6.3 性能特征：为什么递归 CTE 容易慢

| 因素 | 影响 |
|---|---|
| **每轮全量扫描新增行** | 递归引用读的是 `[上轮终点, 当前总数)`，随结果集增大，每轮扫描量递增 |
| **物化表上的查找无索引**（除非触发自动建键且是等值引用） | 递归步内部的连接/过滤往往退化为扫描 |
| **递归深度 = 迭代轮数** | 每一轮都是一次完整的子查询执行 + 物化写入 |
| **可能降级到磁盘** | 结果集超过 `temptable_max_ram` 后转 InnoDB（见 [07_temptable.md](07_temptable.md)），递归写入代价陡增 |

**实践含义**：递归 CTE 适合"深度小、每轮增量小"的场景（如组织树展开、路径枚举）。深度大或每轮产生大量行时，复杂度会快速恶化。

### 6.4 本版能力边界

- **无环检测**（见设计思想一节）
- **无 `MAXRECURSION` 式的 hint**：`cte_max_recursion_depth` 是唯一控制手段（源码里仅注释提及过这类 hint，未实现）
- **递归块内禁止 ORDER BY / LIMIT / DISTINCT**（破坏逐轮追加语义）

---

## 七、深潜补充：克隆表、校验清单、递归的三个细节

### 7.1 clone_tmp_table 的关键参数

```cpp
open_table_from_share(..., db_stat=0, EXTRA_RECORD | DELAYED_OPEN);
```

注释原文明确：**否则写者写行会覆盖读者刚读的行**——`db_stat=0` + `EXTRA_RECORD` 保证克隆表有独立的 record buffer，读者和写者互不干扰。

其余：`t->reginfo.lock_type = TL_WRITE`；`hash_field` 一并拷贝。

### 7.2 结构校验的完整清单

除"必须有 UNION"之外，还有 5 条容易被忽略的禁止项：

| 禁止项 | 原因 |
|--------|------|
| ORDER BY over UNION | 会引入 MyISAM 临时表（递归不能用） |
| 右嵌套 UNION | — |
| 递归块内有 ORDER BY/LIMIT/DISTINCT | 破坏逐轮追加语义 |
| `last_distinct` 后跟 ALL | 去重语义冲突（见 6.1） |
| 非递归块必须在最前 | — |

### 7.3 共享物化的读者同步

`use_shared_cte_materialization` 判定后只置 `table->materialized = true`；`Table_ref::materialize_derived` 经 **`Derived_refs_iterator`** 把所有**克隆**都标记为已物化（不只是主表）。

### 7.4 递归的三个细节

1. **强制严格模式**（`Strict_error_handler`）：列类型由 **anchor 成员**决定，若非严格模式下截断（如 anchor 是 `VARCHAR(10)`，递归部分产出更长值），会导致**无限递归**（值永不收敛）——所以非严格模式也要报错。
2. **深度计数是"轮数"不是行数**：`m_recursive_iteration_count` 在 `Init` 归零，**先自增再比较**。`cte_max_recursion_depth` 默认 **1000**。
3. **本版无 `MAX_RECURSION` hint**——只在注释里被提及，实际只能靠系统变量。

### 7.5 溢出转 InnoDB 与相关 CTE 重置

- **溢出**：`MaterializeQueryBlock` 中 `create_ondisk_from_heap` 后要调 `recursive_reader->RepositionCursorAfterSpillToDisk()`——因为引擎换了，游标必须重定位。
- **相关 CTE 重置**：`m_cte->clear_all_references()` 清空所有 `tmp_tables` 并回绕游标；必须让 `materialized = false`，否则写者会跳过物化。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → WITH (Common Table Expressions)*
- *MySQL 8.0 Reference Manual → Recursive Common Table Expressions*（`cte_max_recursion_depth`）

**理论**
- SQL:1999 标准的递归查询（recursive query）——最小不动点（least fixed point）语义，是 `WITH RECURSIVE` 必须写成 anchor + UNION 递归块的根本原因

**相关文档**
- 优化器侧（CTE 为何不能 merge、与 derived 的差异）见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)
- 物化用的内部临时表（TempTable / MEMORY 引擎、降级链）见 [`07_temptable.md`](07_temptable.md)

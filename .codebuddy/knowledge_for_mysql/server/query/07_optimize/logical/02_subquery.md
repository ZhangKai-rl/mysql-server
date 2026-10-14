# 02 子查询改写：决策树与 IN2EXISTS / 物化 / anti-join / MIN-MAX

> 子查询**改写决策**（prepare/optimize 阶段如何等价变换）。semi-join 已独立成章（03 篇），本篇覆盖其余改写路径。运行期执行见 [`../../runtime/02_subquery_runtime.md`](../../runtime/02_subquery_runtime.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、Subquery_strategy 状态机与生命周期](#一subquery_strategy-状态机与生命周期)
- [二、resolve_subquery 总入口](#二resolve_subquery-总入口)
- [三、IN → IN2EXISTS](#三in--in2exists)
- [四、IN → 物化与代价比较](#四in--物化与代价比较)
- [五、NOT IN / NOT EXISTS → anti-join](#五not-in--not-exists--anti-join)
- [六、ANY / ALL → MIN/MAX](#六any--all--minmax)
- [七、标量子查询 → derived](#七标量子查询--derived)
- [八、flatten_subqueries 与恒假删除](#八flatten_subqueries-与恒假删除)
- [相关参数](#相关参数)

---

## 设计思想与理论基础

### 子查询的分类：是三维，不是"语法形态"

MySQL 对子查询的判定不是"语法形态"单维度，而是**（形态 × 位置 × 到子句根的连接方式）**三维：

| 维度 | 源码载体 | 取值 |
|---|---|---|
| **形态** | `Item_subselect::subs_type` | SINGLEROW / EXISTS / IN / ALL / ANY |
| **位置** | `Collect_scalar_subquery_info::Location` | L_SELECT / L_WHERE / L_HAVING / L_JOIN_COND |
| **连接方式** | `enum_condition_context` + `embedding_join_nest` | ANDS（AND 顶层）/ ANDS_ORS / NEITHER；`(Table_ref*)1`=WHERE 的 AND 部分、nest 指针=ON 的 AND 部分、**NULL=其他位置（且"不是改写候选"）** |

同一句 `IN (SELECT ...)`，在 WHERE 的 AND 顶层、在 OR 下、在 ON 里、在 SELECT list 里，走的路径完全不同——semi-join 要求 ANDS 且在 WHERE/ON；derived 容 OR；标量→derived 禁 JOIN_COND 位置的相关子查询。

> 后面八节讲"每种改写怎么做"，本章讲"为什么按这个维度分、以及为什么需要这么多条路"。

### 相关 vs 不相关：为什么是关键分野

**"相关"在代码里就是一个位**（`UNCACHEABLE_DEPENDENT`）。真正的判定者是左表达式注入后的 `Item_ref`：

```cpp
// sql/item.h:5826
  table_map used_tables() const override {
    if (depended_from != nullptr) return OUTER_REF_TABLE_BIT;   // ★ 相关子查询的标志位
    const table_map map = ref_item()->used_tables();
    if (map != 0) return map;
    // rollup constant: ensure it is non-constant by returning RAND_TABLE_BIT
    if (has_rollup_expr()) return RAND_TABLE_BIT;
    return 0;
  }
```

```cpp
// sql/item_subselect.cc:376 —— 什么情况下左列必须被标成"外引用"
bool Item_in_subselect::mark_as_outer(Item *left_row, size_t col) {
  const Item *left_col = left_row->element_index(col);
  return !left_col->const_item() ||
         (!abort_on_null && left_col->is_nullable()) ||
         (left_row->type() == SUBSELECT_ITEM && !left_col->basic_const_item());
}
```

> ★ 注意第二条：**即使左列是常量，只要可能为 NULL 且我们要区分 NULL/FALSE，也必须标成 outer**——否则优化器会把 `trig_cond` 常量折叠掉。

**IN2EXISTS 会"制造"相关性，物化时要能区分**（`item_subselect.cc:1848`）：

```cpp
    assert(in2exists_info == nullptr);
    in2exists_info = new (thd->mem_root) In2exists_info;
    in2exists_info->dependent_before =
        unit->uncacheable & UNCACHEABLE_DEPENDENT;      // ★ 改写【前】是否相关
    if (!left_expr->const_item()) unit->uncacheable |= UNCACHEABLE_DEPENDENT;
    in2exists_info->dependent_after =
        unit->uncacheable & UNCACHEABLE_DEPENDENT;      // ★ 改写【后】是否相关
```

```cpp
  /// Is reliable only if IN->EXISTS has been done.
  bool dependent_before_in2exists() const {
    return in2exists_info->dependent_before;
  }
```

> ★★ 这就是物化可行性判定"⑦ correlated" 用的 `dependent_before_in2exists()`——**IN2EXISTS 自己造成的相关不算数**，因为选物化时会把注入条件摘掉。
>
> ★ 物化路线的"去相关"（`finalize_materialization_transform`）：
> ```cpp
>   join->query_block->uncacheable &= ~UNCACHEABLE_DEPENDENT;
>   unit->uncacheable &= ~UNCACHEABLE_DEPENDENT;
>   //  ★ 子查询重新变成非相关，才能一次性物化
> ```

**非确定性是另一条独立判据**（`UNCACHEABLE_RAND`）——它在代价比较里被单独降级：

```cpp
// sql_optimizer.cc:11132
  if (allowed_strategies == Subquery_strategy::CANDIDATE_FOR_IN2EXISTS_OR_MAT &&
      (query_expression()->uncacheable & UNCACHEABLE_RAND))
    allowed_strategies = Subquery_strategy::SUBQ_EXISTS;   // ★ 非确定性 → 强制 EXISTS
```

**最终决策由优化期的 `JOIN::decide_subquery_strategy()` 拍板**（`sql/sql_optimizer.cc:11061`）：

```cpp
bool JOIN::decide_subquery_strategy() {
  assert(query_expression()->item);

  switch (query_expression()->item->substype()) {
    case Item_subselect::IN_SUBS:
    case Item_subselect::ALL_SUBS:
    case Item_subselect::ANY_SUBS:
      // All of those are children of Item_in_subselect and may use EXISTS
      break;
    default:
      return false;
  }

  Item_in_subselect *const in_pred =
      static_cast<Item_in_subselect *>(query_expression()->item);

  Subquery_strategy chosen_method = in_pred->strategy;
  // Materialization does not allow UNION so this can't happen:
  assert(chosen_method != Subquery_strategy::SUBQ_MATERIALIZATION);

  if ((chosen_method == Subquery_strategy::CANDIDATE_FOR_IN2EXISTS_OR_MAT) &&
      compare_costs_of_subquery_strategies(&chosen_method))
    return true;

  switch (chosen_method) {
    case Subquery_strategy::SUBQ_EXISTS:
      if (query_block->m_windows.elements > 0)  // grep for WL#10431
      {
        my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                 "the combination of this ALL/ANY/SOME/IN subquery with this"
                 " comparison operator and with contained window functions");
        return true;
      }
      return in_pred->finalize_exists_transform(thd, query_block);
    case Subquery_strategy::SUBQ_MATERIALIZATION:
      return in_pred->finalize_materialization_transform(thd, this);
    default:
      assert(false);
      return true;
  }
}
```

> ★★ **物化不在 resolver 里选**。resolver 阶段（`select_in_like_transformer`）**无条件先把 IN→EXISTS 做掉**，只置"候选"状态 `CANDIDATE_FOR_IN2EXISTS_OR_MAT`；真正的 EXISTS / 物化二选一在**优化期**这里，选中物化后再用 `remove_in2exists_conds()` **反悔撤销**。

**"相关性"在 MySQL 里不是一个标记，而是一个参与所有 `used_tables()` 位运算的伪表位。**

`table_map` 是 64 位位图，其中 3 位被"子查询属性"占用：

```cpp
constexpr const size_t MAX_TABLES_FOR_SIZE{sizeof(table_map) * 8};
constexpr const size_t MAX_TABLES{MAX_TABLES_FOR_SIZE - 3};   // = 61
constexpr const table_map INNER_TABLE_BIT{1ULL << (MAX_TABLES + 0)};
constexpr const table_map OUTER_REF_TABLE_BIT{1ULL << (MAX_TABLES + 1)};
constexpr const table_map RAND_TABLE_BIT{1ULL << (MAX_TABLES + 2)};
```

注释对 `OUTER_REF_TABLE_BIT` 的说明是：*"all subquery items between the column reference and the query block where the column is resolved, have this bit set"*——**相关性会沿 Item 树向上传播**。因此相关子查询天然"逐外层行变化"，无法"算一次缓存"。

这个代价被**显式建模**进代价公式（`compare_costs_of_subquery_strategies()`）：

```cpp
const double subq_executions = calculate_subquery_executions(in_pred, trace);
const double cost_exists = subq_executions * saved_best_read;
const double cost_mat_table = sjm.materialization_cost.total_cost();
const double cost_mat = cost_mat_table + subq_executions * sjm.lookup_cost.total_cost();
```

`subq_executions` 沿父 join 链累乘 fanout 得到。

⇒ **这个公式就是"相关 vs 不相关"的量化表达**：EXISTS 成本线性于外层行数；物化把成本拆成"一次性建表 + 每次 O(1) 探测"。

**去相关在 MySQL 里有两层含义**（很多资料混淆）：

1. **semi-join 层**：`decorrelate_condition()` / `decorrelate_equality()`，把 `outer_expr OP inner_expr` 从子查询 WHERE 摘出来，变成 sj nest 的 `sj_outer_exprs`/`sj_inner_exprs`
2. **标量→derived 层**：把相关谓词提升为 **derived 的 GROUP BY 键**——"按相关键分组，一次算完所有组，再按键回连"。这正是教科书式 decorrelation 在 MySQL 的落地形态

### 为什么需要四条出路：在四维资源上取舍

`IN (SELECT ...)` 有四条出路，本质是在**（表预算 / 内存 / CPU / 重排自由度）**四维上取舍：

| 出路 | 消耗 | 换取 | 变差的场景 |
|---|---|---|---|
| **semi-join 拍平** | **表预算**（不可逆） | 重排自由度 + 五种去重策略 | 有 GROUP BY/HAVING/聚合/窗口/UNION、不在 AND 顶层、表数超 `MAX_TABLES` |
| **物化** | 内存 / 临时表 | 子查询只算一次 | 相关、非确定性、多列可空、类型不兼容 |
| **IN→EXISTS** | **CPU**（线性于外层 fanout） | 零额外资源，永远可行 | 相关 + 外层大表 ⇒ 灾难 |
| **转 derived**（8.0） | 一次物化 | 可吃 hash join + 自由重排 | 默认 OFF；去不了相关就报错 |

**semi-join 为什么是首选**——四条证据：

1. `resolve_subquery()` 判定链里排第一，命中即 `choice_made = true`，其余短路
2. 变换把子查询的表、条件**整体搬进外层 FROM/WHERE**，之后统一 join 排序、选 access path
3. **不可逆**：`flatten_subqueries()` 头注释——"performed **once in query lifetime and is irreversible**"
4. **去相关的价值**（`decorrelate_condition()` 头注释，本章最好的一段原文）：

> "The purpose of decorrelation is to be able to use more execution strategies. **Without decorrelation, EXISTS is limited to FirstMatch and DupsWeedout strategies. Decorrelation enables LooseScan and Materialization.**"

**它吃表预算**是最"工程"却关键的约束：

```cpp
// Add the tables in the subquery nest plus one in case of materialization:
const uint tables_added =
    subq_item->unit->first_query_block()->leaf_table_count + 1;
if (table_count + tables_added <= MAX_TABLES && !...has_aj_nests)
  subq_item->strategy = Subquery_strategy::SEMIJOIN;
```

`MAX_TABLES = 61`，超限的子查询被退回 `UNSPECIFIED` 再走 IN2EXISTS。**这是"为什么不能只用 semi-join"的最硬证据：它吃全局资源。**

资源不够时还要**优先保住最贵的**（调度启发式）：相关子查询优先、表多者优先、靠前的优先。

### 两阶段决策：为什么有的策略先"候选"

`Subquery_strategy` 枚举注释：

> "Sometimes the strategy is first only a candidate, then the real decision happens in a second phase. Other times the first decision is final."

原因是**阶段能力不同**：resolver 阶段没有 `best_read`/fanout，算不出 `subq_executions`，做不了代价决策；optimizer 阶段才有。所以

- 能在 resolver 定死的就定死（semi-join 结构性、derived 结构性、EXISTS 直接终态）
- 定不死的登记 `CANDIDATE_FOR_IN2EXISTS_OR_MAT`，等 `decide_subquery_strategy()`

"候选"同时也是**一个待处理的注册队列**（push 进 `sj_candidates`，在 `flatten_subqueries()` 里统一排序/拍平）。

### 设计哲学：非确定性为什么不能物化

这是本篇唯一一处可以直接谈"设计哲学"的源码证据。`decorrelate_condition()` 的头注释把"能否物化/去相关"从工程限制上升到了**用户直觉语义**：

```
1. Non-deterministic function as substitute for expression from outer query block:
   A  SELECT * FROM t1 WHERE RAND() IN (SELECT t2.x FROM t2)
   B  SELECT * FROM t1 WHERE EXISTS (SELECT * FROM t2 WHERE RAND() = t2.x);
   The intuitive interpretation of the IN subquery is that the random function
   is evaluated per row of the outer query block, whereas in the EXISTS subquery,
   it should be evaluated per row of the inner query block...
...
Thus, the intuitive interpretation is to avoid materialization for subqueries
with non-deterministic components in the inner query block, and hence
such predicates will not be decorrelated.
```

⇒ **这不是实现难度问题，而是"用户对非确定函数求值次数的直觉"问题**：`RAND() IN (SELECT ...)` 用户期望 RAND 每行外层算一次；一旦物化（子查询只算一次），语义就变了。

### NULL 语义：三值逻辑怎么保住

`NOT IN` 遇 NULL 返回空集是经典陷阱。源码用多层机制保住三值：

- **`abort_on_null`**——顶层 vs 非顶层的分水岭。顶层（WHERE）FALSE 与 UNKNOWN 等价，可以忽略；`NOT IN` 不行，**必须仍执行子查询**再判定
- **`Item_is_not_null_test`**——用副作用 `owner->was_null |= 1` 在只有二值的 EXISTS 里"偷渡"三值信息；且**故意伪造 `RAND_TABLE_BIT`** 让优化器不敢把它从 HAVING 下推到 WHERE
- **物化引擎的 NULL 二次探测**——未找到匹配时再用 `null_ref_key` 探一次 NULL

anti-join 侧极保守：一旦 `null_problem` 且任一列可空 → 直接退回。**这就是 `NOT IN` 在有 NULL 列时永远拿不到 anti-join 的直接原因。**

### 演进痕迹（代码化石）

| 痕迹 | 说明 |
|---|---|
| `WL#1110` | 子查询物化引擎 `subselect_hash_sj_engine` 的"出生证" |
| `BUG#36752` | 物化的类型/BLOB 限制，注释自称 **"This is a temporary fix"**，从 5.6 拖到 8.0.39 未移除 |
| `Bug#14215895` | UNION 子查询 + `LIMIT 1` 不能让每个分支提前终止 |
| `WL#6570` | prepare-once 重构，大量 `@todo`/`remove-after` 是未完成的活化石 |
| `unique_subquery` / `index_subquery` | `subselect_uniquesubquery_engine` 类已不存在，**只剩 EXPLAIN 兼容字符串** |
| `SubqueryExecMethod` | 这个枚举**在代码里根本不存在**，只活在一条 TODO 注释里 |
| `WL#10431` | 窗口函数场景下强制物化（IN2EXISTS 需要把 `outer_expr = WF` 塞进 WHERE/HAVING，而 WF 不允许出现在那里） |

### 参数

| 参数 | 默认 | 说明 |
|---|---|---|
| `semijoin` | **ON** | semi-join 总开关 |
| `materialization` | **ON** | 物化总开关（**同时管 semi-join 的 Materialize 策略与子查询物化**） |
| `subquery_materialization_cost_based` | **ON** | ON=代价比较；OFF=无条件物化 |
| `subquery_to_derived` | **OFF** | IN→derived / 标量→derived；secondary engine 下被强制打开 |

---

## 一、Subquery_strategy 状态机与生命周期

### 1.1 四类子查询

`Item_subselect` 按 `substype()` 分四类：

| 类型 | 类 | 说明 |
|------|-----|------|
| `SINGLEROW_SUBS` | `Item_singlerow_subselect` | 标量（单行单列） |
| `EXISTS_SUBS` | `Item_exists_subselect` | EXISTS / NOT EXISTS |
| `IN_SUBS` | `Item_in_subselect` | IN / NOT IN / =ANY |
| `ALL_SUBS`/`ANY_SUBS` | `Item_allany_subselect` | ALL/ANY 比较 |

### 1.2 Subquery_strategy 枚举（item_subselect.h:398）

```cpp
enum class Subquery_strategy : int {
  UNSPECIFIED,                     // 初始态，尚未决策
  CANDIDATE_FOR_IN2EXISTS_OR_MAT,  // 暂定 IN→EXISTS，保留物化可能
  CANDIDATE_FOR_SEMIJOIN,          // 满足 16 条件，登记到 sj_candidates
  CANDIDATE_FOR_DERIVED_TABLE,     // 满足 derived 条件
  SEMIJOIN,                        // 已拍平为 semi/anti-join nest
  DERIVED_TABLE,                   // 已改成 LEFT JOIN derived
  SUBQ_EXISTS,                     // 最终采用 IN→EXISTS
  SUBQ_MATERIALIZATION,            // 最终采用 hash semi-join 物化
  DELETED,                         // 子查询恒假被删除
};
```

**状态迁移图**：

```
UNSPECIFIED ──(16条件)──> CANDIDATE_FOR_SEMIJOIN ──> SEMIJOIN
     │
     ├─(derived条件)──> CANDIDATE_FOR_DERIVED_TABLE ──> DERIVED_TABLE
     │
     └─(select_transformer)──> CANDIDATE_FOR_IN2EXISTS_OR_MAT
                                      │
                          decide_subquery_strategy（JOIN::optimize 内）
                                      │
                    ┌─────────────────┴───────────────┐
                    v                                 v
               SUBQ_EXISTS                    SUBQ_MATERIALIZATION
```

核心：这是贯穿全程的**两阶段决策**——resolver 阶段先定大方向（semi-join / derived / 暂定 EXISTS），optimizer 阶段再在 EXISTS 与物化之间做代价决策。

### 1.3 Item_subselect 生命周期

**(1) fix_fields（item_subselect.cc:533）**：

```cpp
if (!(res = subquery->prepare(thd))) {          // 递归 prepare 内层 Query_expression
  changed = true;
  accumulate_properties();                      // 汇总 INNER_TABLE_BIT 等属性
  if (substitution) {                           // transformer 生成了替换物
    (*ref) = substitution;                      // 替换 *ref
    if (!(*ref)->fixed) ret = (*ref)->fix_fields(thd, ref);  // 二次 fix_fields
    return ret;
  }
  ...
}
```

关键：`subquery->prepare()` 会触发 `resolve_subquery` 及 transformer；若 transformer 已把本 `Item_in_subselect` 替换成 `Item_in_optimizer`（`substitution`），则替换 `*ref` 并递归 `fix_fields`——形成"两次 fix_fields"机制。

**(2) optimize 阶段 `decide_subquery_strategy`（sql_optimizer.cc:11061）**——见第四节。

**(3) 执行期 engine 分派 `Item_subselect::exec`（item_subselect.cc:634）**：

```cpp
bool Item_subselect::exec(THD *thd) {
  // subselect_hash_sj_engine 自己建迭代器，不走 exec
  bool should_create_iterators =
      !(indexsubquery_engine != nullptr &&
        indexsubquery_engine->engine_type() == subselect_indexsubquery_engine::HASH_SJ_ENGINE);
  if (!unit->is_optimized()) { unit->optimize(...); }   // 懒优化
  ...
  if (indexsubquery_engine != nullptr) {
    return indexsubquery_engine->exec(thd);   // 物化引擎 / 索引查找引擎
  } else {
    return subquery->exec(thd);               // 普通 EXISTS 子查询
  }
}
```

执行分派点：有 `indexsubquery_engine` 走引擎（物化 lookup 或 index subquery），否则直接执行内层 `subquery`（EXISTS）。

---

## 二、resolve_subquery 总入口

`sql/sql_resolver.cc:1368-1633`，在 `Query_block::prepare` 里调用。核心三段：

### 2.1 半连接 16 条件（:1527-1567）

⚠️ **勘误**：`resolve_subquery` 定义在 **`sql/sql_resolver.cc:1368`**（含 `:1348-1366` 的 doxygen）；`:1527` 只是函数体内 `SecondaryEngineFlags engine_flags = 0;` 那一行。三段的行号应为：**① 1532–1567 ② 1605–1627 ③ 1629–1632**。

**条件清单的真实源码**（注释编号 0–16，其中 3x 并入 3，代码里恰为 **16 个 `&&` 合取项**）：

```cpp
  /*
    Check if we're in subquery that is a candidate for flattening into a
    semi-join (which is done in flatten_subqueries()). The requirements are:
      0. Semi-join is enabled (cf. hints)
      1. Subquery predicate is an IN/=ANY or EXISTS predicate
      2. Subquery is a simple query block (not a set operation or a
         parenthesized query expression).
      3. Subquery is not grouped (explicitly or implicitly)
         3x: outer aggregated expression are not accepted
      4. Subquery does not use HAVING
      5. Subquery does not use windowing functions
      6. Subquery predicate is (a) in an ON/WHERE clause,
         and (b) at the AND-top-level of that clause. Note for 6a:
         Semijoin transformations of subqueries in ON cause the
         join nests to no longer be acceptable as a join tree, which
         disturbs the hypergraph optimizer, so we disable them
         for that case (6x). ...
      7. Parent query block accepts semijoins (i.e we are not in a subquery of
      a single table UPDATE/DELETE ...)
      8. We're not in a confluent table-less subquery, like "SELECT 1".
      9. No execution method was already chosen (by a prepared statement)
      10. Parent query block is not a confluent table-less query block.
      11. Neither parent nor child query block has straight join.
      12. Parent query block does not prohibit semi-join.
      13. LHS of IN predicate is deterministic
      14. The surrounding truth test, and the nullability of expressions,
      are compatible with the conversion.
      15. Antijoins are supported, or it's not an antijoin (it's a semijoin).
      16. OFFSET starts from the first row and LIMIT is not 0.
  */
  SecondaryEngineFlags engine_flags = 0;
  if (const handlerton *secondary_engine = SecondaryEngineHandlerton(thd);
      secondary_engine != nullptr) {
    engine_flags = secondary_engine->secondary_engine_flags;
  }
  if (semijoin_enabled(thd) &&                                     // 0
      predicate != nullptr &&                                      // 1
      is_simple_query_block() &&                                   // 2
      no_aggregates &&                                             // 3,3x,4,5
      (outer->resolve_place == Query_block::RESOLVE_CONDITION ||   // 6a
       (outer->resolve_place == Query_block::RESOLVE_JOIN_NEST &&  // 6a
        (!thd->lex->using_hypergraph_optimizer() ||
         (thd->secondary_engine_optimization() ==
              Secondary_engine_optimization::SECONDARY &&
          !Overlaps(engine_flags,
              MakeSecondaryEngineFlags(
                  SecondaryEngineFlag::SUPPORTS_NESTED_LOOP_JOIN)))))) &&  // 6x
      outer->condition_context == enum_condition_context::ANDS &&          // 6b
      outer->sj_candidates &&                                              // 7
      leaf_table_count > 0 &&                                              // 8
      predicate->strategy == Subquery_strategy::UNSPECIFIED &&             // 9
      outer->leaf_table_count > 0 &&                                       // 10
      !((active_options() | outer->active_options()) &
        SELECT_STRAIGHT_JOIN) &&                                           // 11
      !(outer->active_options() & SELECT_NO_SEMI_JOIN) &&                  // 12
      deterministic &&                                                     // 13
      predicate->choose_semijoin_or_antijoin() &&                          // 14
      (!cannot_do_antijoin || !predicate->can_do_aj) &&                    // 15
      is_row_count_valid_for_semi_join()) {                                // 16
    DBUG_PRINT("info", ("Subquery is semi-join conversion candidate"));

    /* Notify in the subquery predicate where it belongs in the query graph */
    predicate->embedding_join_nest = outer->resolve_nest;

    /* Register the subquery for further processing in flatten_subqueries() */
    predicate->strategy = Subquery_strategy::CANDIDATE_FOR_SEMIJOIN;
    outer->sj_candidates->push_back(predicate);
    choice_made = true;
  }
```

> ★ 命中后**只是登记候选**（`CANDIDATE_FOR_SEMIJOIN` + push 进 `outer->sj_candidates`），真正的展平在 `flatten_subqueries()` → `convert_subquery_to_semijoin()`（`sql_resolver.cc:2992`）。

**条件 16 的委托函数**（`sql_resolver.cc:1327`）：

```cpp
bool Query_block::is_row_count_valid_for_semi_join() {
  if (offset_limit != nullptr &&
      (!offset_limit->const_item() || offset_limit->val_int() != 0))
    return false;
  if (select_limit != nullptr &&
      (!select_limit->const_item() || select_limit->val_int() == 0))
    return false;
  return true;
}
```

**条件 0 的委托函数**（`sql_lex.cc:4551`）：

```cpp
bool Query_block::semijoin_enabled(const THD *thd) const {
  return opt_hints_qb ? opt_hints_qb->semijoin_enabled(thd)
                      : thd->optimizer_switch_flag(OPTIMIZER_SWITCH_SEMIJOIN);
}
```

> ⚠️ **不存在**独立的 `Item_subselect::is_convertible_to_semijoin()`——候选判定就是上面这一个内联的 16 项 `&&` 表达式。

```cpp
if (semijoin_enabled(thd) &&                                     // 0
    predicate != nullptr &&                                      // 1
    is_simple_query_block() &&                                   // 2
    no_aggregates &&                                             // 3,3x,4,5
    (outer->resolve_place == Query_block::RESOLVE_CONDITION ||   // 6a
     (outer->resolve_place == Query_block::RESOLVE_JOIN_NEST &&  // 6a
      (!thd->lex->using_hypergraph_optimizer() || ...))) &&
    outer->condition_context == enum_condition_context::ANDS &&  // 6b
    outer->sj_candidates &&                                      // 7
    leaf_table_count > 0 &&                                      // 8
    predicate->strategy == Subquery_strategy::UNSPECIFIED &&     // 9
    outer->leaf_table_count > 0 &&                               // 10
    !((active_options() | outer->active_options()) & SELECT_STRAIGHT_JOIN) &&  // 11
    !(outer->active_options() & SELECT_NO_SEMI_JOIN) &&          // 12
    deterministic &&                                             // 13
    predicate->choose_semijoin_or_antijoin() &&                  // 14
    (!cannot_do_antijoin || !predicate->can_do_aj) &&            // 15
    is_row_count_valid_for_semi_join()) {                        // 16
  predicate->strategy = Subquery_strategy::CANDIDATE_FOR_SEMIJOIN;
  outer->sj_candidates->push_back(predicate);
  choice_made = true;
}
```

16 条条件的逐条语义（对应源码注释）：

| # | 条件 |
|---|------|
| 0 | `semijoin_enabled`（开关或 hint） |
| 1 | 必须是 `IN/=ANY` 或 `EXISTS` |
| 2 | 简单 Query_block（非 UNION） |
| 3-5 | 无 GROUP BY / 聚合 / HAVING / 窗口 |
| 6a/6b | 位于 WHERE/ON 的 **AND 顶层** |
| 7 | 外层允许 semi-join |
| 8 | 内层有真实表 |
| 9 | 策略未定（PS 已定则跳过） |
| 10 | 外层有表 |
| 11 | 内外层均无 STRAIGHT_JOIN |
| 12 | 外层未禁止 semi-join |
| 13 | IN 左表达式确定 |
| 14 | `choose_semijoin_or_antijoin()`（见第五节） |
| 15 | anti-join 被支持（或这不是 anti-join） |
| 16 | LIMIT/OFFSET 合法 |

### 2.2 derived-table 候选（:1605-1627）

```cpp
  /*
    If semijoin failed, try a transformation to a derived table:
    FROM ot WHERE ot.x IN (SELECT y FROM it1, it2)
    =>
    FROM ot LEFT JOIN (SELECT DISTINCT y FROM it1, it2) AS derived
            ON ot.x=derived.y
    WHERE derived.y IS NOT NULL.

    Applicability constraints have numbers which are the same as in the list of
    the previous block. Reasons may be different though.
      1. Subquery predicate is an IN/=ANY or EXISTS predicate
      2. Subquery is a simple query block ...
      3. If this is [NOT] EXISTS, there is no aggregation
      6. Subquery predicate is
        6a. in WHERE clause (we have not implemented the transformation for the
        ON clause)
        6b. linked to the root of that clause with ANDs or ORs.
      7. Parent query block accepts semijoins ...
      9. No execution method was already chosen (by a prepared statement)
      10. Parent select has tables, as we'll link to them with LEFT JOIN
      12. Parent query block does not prohibit semi-join.
      13. LHS of IN predicate is deterministic
      14. The surrounding truth test, and the nullability of expressions,
      are compatible with the conversion.
      16. The left argument isn't a row (multi-column) subquery; it would lead
      to creating conditions like WHERE (outer_subq) =
      ROW(derived.col1,derived.col2), which would complicate code.
      17. Certain other subquery transformations, incompatible with this one,
      have not been done.
  */

  if (!choice_made && try_convert_to_derived && predicate != nullptr &&  // 1
      is_simple_query_block() &&                                         // 2
      (in_predicate != nullptr || no_aggregates) &&                      // 3
      outer->resolve_place == Query_block::RESOLVE_CONDITION &&          // 6a
      outer->condition_context != enum_condition_context::NEITHER &&     // 6b
      outer->sj_candidates &&                                            // 7
      predicate->strategy == Subquery_strategy::UNSPECIFIED &&           // 9
      outer->leaf_table_count &&                                         // 10
      !(outer->active_options() & SELECT_NO_SEMI_JOIN) &&                // 12
      deterministic &&                                                   // 13
      predicate->choose_semijoin_or_antijoin() &&                        // 14
      !(in_predicate != nullptr &&                                       // 16
        in_predicate->left_expr->type() == Item::SUBSELECT_ITEM &&
        in_predicate->left_expr->cols() > 1) &&
      !thd->lex->m_subquery_to_derived_is_impossible) {                  // 17
    assert(outer->resolve_nest == nullptr);
    /* Register the subquery for further processing in flatten_subqueries() */
    outer->sj_candidates->push_back(predicate);
    predicate->strategy = Subquery_strategy::CANDIDATE_FOR_DERIVED_TABLE;
    predicate->outer_condition_context = outer->condition_context;
    choice_made = true;
  }
```

> ★ 条件 17 `m_subquery_to_derived_is_impossible` 是**互斥闸门**：`single_value_in_to_exists_transformer()` 与 MIN/MAX 改写末尾都会 `thd->lex->m_subquery_to_derived_is_impossible = true;`，表示"已经做了别的改写，不能再转 derived"。
>
> ★ 条件 6b 与半连接不同：这里是 `!= NEITHER`（**AND 或 OR 都行**），而半连接要求 `== ANDS`。

**开关 `try_convert_to_derived`**：

```cpp
  const bool try_convert_to_derived =
      (thd->optimizer_switch_flag(OPTIMIZER_SWITCH_SUBQUERY_TO_DERIVED) ||
       // a certain secondary engine doesn't support subqueries
       (thd->lex->m_sql_cmd != nullptr &&
        thd->secondary_engine_optimization() ==
            Secondary_engine_optimization::SECONDARY));
```

> ⚠️ **`subquery_to_derived` 默认是 off**——它**不在** `OPTIMIZER_SWITCH_DEFAULT` 里（`sql/sys_vars.cc:197-211`）。

不满足 semi-join 时，若 `subquery_to_derived` 开关允许（**默认 OFF**，见第九节），再试 derived 转换——把子查询变成 `LEFT JOIN derived ON ...`。**注意：derived 候选用的是同一套 `sj_candidates` 登记，但判定条件不同**（注释里编号与 semi-join 那 16 条对齐，缺的编号就是"不适用/未校验"的）：

```cpp
// sql_resolver.cc:1605
if (!choice_made && try_convert_to_derived && predicate != nullptr &&  // 1  IN/EXISTS
    is_simple_query_block() &&                                         // 2  非 UNION
    (in_predicate != nullptr || no_aggregates) &&                      // 3  IN 或有聚合也可（EXISTS 须无聚合）
    outer->resolve_place == Query_block::RESOLVE_CONDITION &&          // 6a 只在 WHERE（ON 没实现）
    outer->condition_context != enum_condition_context::NEITHER &&     // 6b AND/OR 顶层均可（比 semi-join 宽松）
    outer->sj_candidates &&                                            // 7  外层允许
    predicate->strategy == Subquery_strategy::UNSPECIFIED &&           // 9  策略未定
    outer->leaf_table_count &&                                         // 10 外层有表（要 LEFT JOIN 挂上去）
    !(outer->active_options() & SELECT_NO_SEMI_JOIN) &&                // 12 外层未禁止
    deterministic &&                                                   // 13 LHS 确定
    predicate->choose_semijoin_or_antijoin() &&                        // 14 nullability 兼容
    !(in_predicate != nullptr &&                                       // 16 左表达式不能是"多列子查询"
      in_predicate->left_expr->type() == Item::SUBSELECT_ITEM &&
      in_predicate->left_expr->cols() > 1) &&
    !thd->lex->m_subquery_to_derived_is_impossible) {                  // 17 无其它不兼容转换
  predicate->strategy = Subquery_strategy::CANDIDATE_FOR_DERIVED_TABLE;
  predicate->outer_condition_context = outer->condition_context;
  outer->sj_candidates->push_back(predicate);
  choice_made = true;
}
```

17 条判定条件的逐条语义（对齐源码注释 1~17）：

| # | 条件 | 与 semi-join 的差异 |
|---|------|--------------------|
| 1 | `IN/=ANY` 或 `EXISTS` | 同 |
| 2 | 简单 Query_block（非 UNION） | 同（secondary engine 不支持 setop DISTINCT） |
| 3 | `IN` 或有聚合（`EXISTS` 须无聚合） | **比 semi-join 宽松**：IN 聚合可转 derived，但 EXISTS 不行 |
| 6a | 只在 **WHERE**（ON 没实现） | **比 semi-join 严**：semi-join 允许 ON |
| 6b | WHERE 顶层（AND **或 OR**） | **比 semi-join 宽松**：semi-join 只要 AND 顶层 |
| 7 | 外层允许 | 同 |
| 9 | 策略未定 | 同 |
| 10 | 外层有表 | 同（要 LEFT JOIN 挂 derived） |
| 12 | 外层未禁止 semi-join | 同 |
| 13 | LHS 确定 | 同 |
| 14 | nullability 兼容 | 同 |
| 16 | 左表达式**不能是多列子查询** | derived 独有：`(subq) = ROW(c1,c2)` 会生成复杂条件，代码不支持 |
| 17 | 无其它不兼容转换 | derived 独有：`m_subquery_to_derived_is_impossible` 兜底 |

**为什么 derived 条件"有的更严、有的更松"**：两者转换目标不同——semi-join 是"拍平成 nest 塞进 join tree"（需要 AND 顶层、无聚合，才能保证去重语义）；derived 是"物化成 derived 表 LEFT JOIN"（LEFT JOIN 天然能表达 OR、能带聚合，但 6a 只实现了 WHERE、16 多列会生成复杂条件）。所以 6b 更松（OR 也行）、3 更松（IN 可带聚合）、但 6a/16 更严（ON 没实现、多列不支持）。

### 2.3 兜底 select_transformer（:1629）

```cpp
  if (!choice_made) {
    return subq_predicate->select_transformer(thd, this);
  }
  return false;
}
```

**`select_transformer` 是纯虚函数**（`item_subselect.h:195`），四个子类的实分派表：

| 类 | `substype()` | `select_transformer` | 位置 |
|---|---|---|---|
| `Item_subselect` | `UNKNOWN_SUBS` | **纯虚 `= 0`** | `item_subselect.h:191,195` |
| `Item_singlerow_subselect` | `SINGLEROW_SUBS` | 标量子查询常量折叠 / 直接替换为 SELECT list 项 | `item_subselect.cc:1116` |
| `Item_maxmin_subselect` | 继承 `SINGLEROW_SUBS` | **无覆盖**，继承上者 | `item_subselect.h:374` |
| `Item_exists_subselect` | `EXISTS_SUBS` | **内联**，仅置 `strategy = SUBQ_EXISTS` | `item_subselect.h:461` |
| `Item_in_subselect` | `IN_SUBS` | `select_in_like_transformer(…, &eq_creator)` | `item_subselect.cc:2384` |
| `Item_allany_subselect` | `ALL_SUBS` / `ANY_SUBS` | `select_in_like_transformer(…, func)` | `item_subselect.cc:2723` |

```cpp
// item_subselect.h:461 —— EXISTS 的最简实现
  bool select_transformer(THD *, Query_block *) override {
    strategy = Subquery_strategy::SUBQ_EXISTS;
    return false;
  }

// item_subselect.cc:2384
bool Item_in_subselect::select_transformer(THD *thd, Query_block *select) {
  return select_in_like_transformer(thd, select, &eq_creator);
  //                                              ★ IN 语义即 =ANY
}

// item_subselect.cc:2723
bool Item_allany_subselect::select_transformer(THD *thd, Query_block *select) {
  if (upper_item) upper_item->show = true;
  return select_in_like_transformer(thd, select, func);
  //  ★ func = > < >= <= 对应的 Comp_creator*
}
```

**`Subquery_strategy` 完整状态机**（`item_subselect.h:392`，9 个值）：

```cpp
enum class Subquery_strategy : int {
  UNSPECIFIED,                        /// Nothing decided yet
  CANDIDATE_FOR_IN2EXISTS_OR_MAT,     /// 待定：EXISTS 或物化
  CANDIDATE_FOR_SEMIJOIN,             /// 半连接候选
  CANDIDATE_FOR_DERIVED_TABLE,        /// derived 候选
  SEMIJOIN,                           /// 已定型：半连接
  DERIVED_TABLE,                      /// 已定型：derived
  SUBQ_EXISTS,                        /// 已定型：EXISTS
  SUBQ_MATERIALIZATION,               /// 已定型：物化（HASH_SJ_ENGINE）
  DELETED,                            /// 子查询被删（恒假）
};
```

```
UNSPECIFIED
  ├─(resolve_subquery 16 条件)→ CANDIDATE_FOR_SEMIJOIN →(flatten)→ SEMIJOIN →(convert_subquery_to_semijoin)
  ├─(derived 段)→ CANDIDATE_FOR_DERIVED_TABLE → DERIVED_TABLE
  └─(兜底 select_in_like_transformer:2445)→ CANDIDATE_FOR_IN2EXISTS_OR_MAT
        └─(优化期 decide_subquery_strategy)→ SUBQ_EXISTS 或 SUBQ_MATERIALIZATION
              └─(cleanup)→ 回到 CANDIDATE_FOR_IN2EXISTS_OR_MAT   ★ PS 二次执行可重选
```

```cpp
// item_subselect.cc:480 —— PS 重执行能重选的秘密
void Item_in_subselect::cleanup() {
  ...
  switch (strategy) {
    case Subquery_strategy::SUBQ_MATERIALIZATION:
      if (in2exists_info->dependent_after) {
        unit->first_query_block()->uncacheable |= UNCACHEABLE_DEPENDENT;
        unit->uncacheable |= UNCACHEABLE_DEPENDENT;
      }
      [[fallthrough]];
    case Subquery_strategy::SUBQ_EXISTS:
      /*
        Back to EXISTS_OR_MAT, so that next execution of this statement can
        choose between the two.
      */
      unit->global_parameters()->select_limit = nullptr;
      strategy = Subquery_strategy::CANDIDATE_FOR_IN2EXISTS_OR_MAT;
      break;
    default: break;
  }
  Item_subselect::cleanup();
}
```

> ⚠️ **`strategy` 不在 `Item_subselect`**，在 `Item_exists_subselect`（`item_subselect.h:432`）——所以 `Item_singlerow_subselect` 没有 `strategy`。
>
> ⚠️ **8.0.39 不存在 `exec_method` / `SubqueryExecMethod`**——只出现在 `resolve_subquery` 的一条 TODO 注释里（`sql_resolver.cc:1412`）。

```cpp
if (!choice_made) {
  return subq_predicate->select_transformer(thd, this);
}
```

多态分派：
- `Item_exists_subselect::select_transformer` → 直接 `strategy = SUBQ_EXISTS`（EXISTS 不物化）
- `Item_in_subselect::select_transformer` → `select_in_like_transformer(thd, select, &eq_creator)`
- `Item_allany_subselect::select_transformer` → `select_in_like_transformer(thd, select, func)`

`select_in_like_transformer`（item_subselect.cc:2404）关键：

```cpp
if (strategy == Subquery_strategy::UNSPECIFIED)
  strategy = Subquery_strategy::CANDIDATE_FOR_IN2EXISTS_OR_MAT;   // 暂定 EXISTS，保留物化

if (left_expr->cols() == 1) {
  if (single_value_transformer(thd, select, func)) return true;   // 先试 MIN/MAX，再 IN→EXISTS
} else {
  if (func != &eq_creator) { my_error(ER_OPERAND_COLUMNS, ...); return true; }
  if (row_value_transformer(thd, select)) return true;            // 多列只允许 =ANY / <>ALL
}
```

---

## 三、IN → IN2EXISTS

### 3.1 注入左表达式（旧版 create_ref）

`single_value_transformer` 里先创建注入（item_subselect.cc:1824）：

```cpp
if (substitution == nullptr) {
  substitution = optimizer;                       // 用 Item_in_optimizer 包裹
  thd->lex->set_current_query_block(select->outer_query_block());
  if (optimizer->fix_left(thd, nullptr)) return true;

  // 旧版 create_ref：把外层左表达式作为 Item_ref 注入内层
  Item_ref *const left = new Item_ref(
      &select->context, (Item **)optimizer->get_cache(), in_left_expr_name);
  mark_as_outer(left_expr, 0);
  left->depended_from = select->outer_query_block();
  m_injected_left_expr = left;
  ...
}
```

`Item_in_optimizer` 内部维护 `cache`（`Item_cache`），缓存外层左表达式当前行的值；`new Item_ref(..., optimizer->get_cache(), ...)` 把这个**缓存槽**以 `Item_ref` 形式注入内层——这就是旧版 `create_ref` 的等价物。

### 3.2 三个分支

`single_value_in_to_exists_transformer`（item_subselect.cc:1907）分三支：

```
oe IN (SELECT ie FROM ... WHERE subq_where)
=> EXISTS (SELECT 1 FROM ... WHERE subq_where AND oe=ie)
```

**分支 1：有 HAVING/GROUP BY/聚合/窗口**（:1922）→ 注入 **HAVING**：

```cpp
if (select->having_cond() != nullptr || select->is_explicitly_grouped() ||
    select->with_sum_func || select->has_windows()) {
  Item_ref_null_helper *ref_null = new Item_ref_null_helper(
      &select->context, this, &select->base_ref_items[0]);   // 引用内层第 0 列
  Item_bool_func *item = func->create(m_injected_left_expr, ref_null);  // outer=inner
  item->set_created_by_in2exists();
  ...
  select->set_having_cond(and_items(select->having_cond(), item));
  select->having_cond()->apply_is_true();
```

聚合场景下内层表达式在分组后才可得，等价比较必须放 HAVING（不能放 WHERE）。`ref_null_helper` 引用 `base_ref_items[0]` 使比较对象是"分组后的内层值"。

**分支 2：普通单表（无聚合）**（:1990）→ 注入 **WHERE**：

```cpp
Item *orig_item = select->single_visible_field();
if (!select->source_table_is_one_row() || select->where_cond() != nullptr) {
  Item_bool_func *item = func->create(m_injected_left_expr, orig_item);
  item->set_created_by_in2exists();
  if (!abort_on_null && orig_item->is_nullable()) {     // 需区分 NULL/FALSE
    Item_bool_func *having = new Item_is_not_null_test(this, orig_item);
    select->set_having_cond(having);                    // HAVING ie IS NOT NULL
    item = new Item_cond_or(item, new Item_func_isnull(orig_item));  // (oe=ie OR ie IS NULL)
  }
  if (!abort_on_null && left_expr->is_nullable()) {     // 外层可空
    item = new Item_func_trig_cond(item, get_cond_guard(0), nullptr, NO_PLAN_IDX,
                                   Item_func_trig_cond::OUTER_FIELD_IS_NOT_NULL);
  }
  select->set_where_cond(and_items(select->where_cond(), item));
  select->where_cond()->apply_is_true();
  in2exists_info->added_to_where = true;
```

**分支 3：常量单行子查询**（:2093）→ 直接降级为普通比较 `substitution = func->create(left_expr, orig_item)`，不做 EXISTS（如 `x IN (SELECT 5)` → `x=5`）。

### 3.3 NULL/FALSE 区分（三值逻辑陷阱）

⚠️ **勘误**：`Item_in_optimizer` **没有 `m_cond` 成员**，只有两个私有字段；`pushed_cond_guards` 也不在它那里，而在 `Item_in_subselect`（`item_subselect.h:611`）。

```cpp
// sql/item_cmpfunc.h:492
class Item_in_optimizer final : public Item_bool_func {
 private:
  Item_cache *cache{nullptr};
  /**
    Stores the value of "NULL IN (SELECT ...)" for uncorrelated subqueries:
      UNKNOWN - "NULL in (SELECT ...)" has not yet been evaluated
      FALSE   - result is FALSE
      TRUE    - result is NULL
  */
  int result_for_null_param{UNKNOWN};      // ★ 三值记忆化
 public:
  Item_in_optimizer(Item_in_subselect *item)
      : Item_bool_func(pointer_cast<Item *>(item)) { set_subquery(); }
  bool fix_left(THD *thd, Item **ref);
  longlong val_int() override;
  Item_cache **get_cache() { return &cache; }   // ★ 注入点拿的就是这个地址
};
```

**`val_int()` 全文**（`item_cmpfunc.cc:2338`）——三值逻辑的落点：

```cpp
longlong Item_in_optimizer::val_int() {
  assert(fixed);
  Item_in_subselect *const subqpred = down_cast<Item_in_subselect *>(args[0]);

  cache->store(subqpred->left_expr);      // ★ 惰性：只登记要缓存的表达式
  cache->cache_value();                   // ★ 真正求值一次，之后子查询里所有 Item_ref 读同一份快照

  if (cache->null_value) {
    /*
      We're evaluating
      "<outer_value_list> [NOT] IN (SELECT <inner_value_list>...)"
      where one or more of the outer values is NULL.
    */
    if (subqpred->abort_on_null) {
      /*
        We're evaluating a top level item, e.g.
        "<outer_value_list> IN (SELECT <inner_value_list>...)",
        and in this case a NULL value in the outer_value_list means
        that the result shall be NULL/FALSE (makes no difference for
        top level items). ...
      */
      null_value = true;                  // ★ 顶层谓词：UNKNOWN 与 FALSE 等价 → 直接判 NULL
    } else {
      /*
        We're evaluating an item where a NULL value in either the
        outer or inner value list does not automatically mean that we
        can return NULL/FALSE. An example of such a query is
        "<outer_value_list> NOT IN (SELECT <inner_value_list>...)"
        ... The result when there is at least one NULL value in
        <outer_value_list> is: NULL if the SELECT evaluated over the
        non-NULL values produces at least one row, FALSE otherwise
      */
      bool all_left_cols_null = true;
      const uint ncols = cache->cols();

      /*
        Turn off the predicates that are based on column compares for
        which the left part is currently NULL
      */
      for (uint i = 0; i < ncols; i++) {
        if (cache->element_index(i)->null_value)
          subqpred->set_cond_guard_var(i, false);   // ★★ 关掉对应列的 trig_cond
        else
          all_left_cols_null = false;
      }

      if (all_left_cols_null && result_for_null_param != UNKNOWN &&
          !subqpred->dependent_before_in2exists()) {
        /*
           This subquery was originally not correlated. ... All values in the
           left expression are NULL, and we have already evaluated the
           subquery for all NULL values: return the same result we did last
           time without evaluating the subquery.
        */
        null_value = result_for_null_param;         // ★ 三值记忆化生效
      } else {
        /* The subquery has to be evaluated */
        (void)subqpred->val_bool_naked();
        if (!subqpred->value)
          null_value = subqpred->null_value;        // 没找到行
        else
          null_value = true;                        // ★ 找到行 + 左值有 NULL → 部分匹配 → UNKNOWN
        if (all_left_cols_null) result_for_null_param = null_value;
      }

      /* Turn all predicates back on */
      for (uint i = 0; i < ncols; i++) subqpred->set_cond_guard_var(i, true);
    }
    cache->store(subqpred->left_expr);
    return subqpred->translate(null_value, false);
  }
  bool result = subqpred->val_bool_naked();         // ★ 左值无 NULL → 直接转发
  null_value = subqpred->null_value;
  cache->store(subqpred->left_expr);
  return subqpred->translate(null_value, result);
}
```

> ★★ **三层协同**：`set_cond_guard_var(i, false)` 关掉左值为 NULL 的那些列的 `Item_func_trig_cond` → 注入的 `ic_k = oc_k` 条件**整条消失** → 子查询退化成"只查剩下非 NULL 列的部分匹配"→ 若仍能产出一行，说明存在部分匹配 ⇒ UNKNOWN。

**注入体为什么是 `Item_ref` 而不是直接 `Item_cache*`**：

```cpp
// item_subselect.cc:1834（single_value_transformer 内）
    Item_ref *const left = new Item_ref(
        &select->context, (Item **)optimizer->get_cache(), in_left_expr_name);
    //                     ★★ 二级指针：指向 &Item_in_optimizer::cache
    if (mark_as_outer(left_expr, 0))
      left->depended_from = select->outer_query_block();
    m_injected_left_expr = left;
```

> ★ 三个理由：① `Item_ref` 有 `depended_from`，能把 `used_tables()` 变成 `OUTER_REF_TABLE_BIT`，**让子查询正式变成相关**；② `Item_ref` 持有 `Item**`，子查询里多处引用同一个指针槽，`Item_in_optimizer` 换 `cache` 指向时自动同步；③ 行情形 `Item_cache_row::addr(i)` 给出每列独立槽位。
>
> ⚠️ **`Item_direct_ref` 在 8.0.39 已不存在**（全仓 0 命中）——常见说法过时。

**`Item_cache` 由 `fix_left()` 按类型造**（`item_cmpfunc.cc:2156`）：

```cpp
bool Item_in_optimizer::fix_left(THD *thd, Item **) {
  Item *left = down_cast<Item_in_subselect *>(args[0])->left_expr;
  if (left->propagate_type(thd, MYSQL_TYPE_VARCHAR)) return true;
  assert(cache == nullptr);
  cache = Item_cache::get_cache(left);   // ★ 按 result_type 造 int/real/decimal/str/datetime/json/row
  if (cache == nullptr) return true;
  cache->setup(left);
  used_tables_cache = left->used_tables();
  ...
  if (const_item()) cache->store(left);
  return false;
}
```

```cpp
Item_cache *Item_cache::get_cache(const Item *item, const Item_result type) {
  switch (type) {
    case INT_RESULT:
      if (item->data_type() == MYSQL_TYPE_BIT) return new Item_cache_bit(...);
      return new Item_cache_int(item->data_type());
    case REAL_RESULT:   return new Item_cache_real();
    case DECIMAL_RESULT: return new Item_cache_decimal();
    case STRING_RESULT:
      if (item->is_temporal()) return new Item_cache_datetime(item->data_type());
      if (item->data_type() == MYSQL_TYPE_JSON) return new Item_cache_json();
      return new Item_cache_str(item);
    case ROW_RESULT: return new Item_cache_row();   // ★ 行 IN
    default: assert(0); return nullptr;
  }
}
```

**`translate()`：把裸 IN 的三值翻成 IS TRUE/FALSE/NOT 的二值**（`item_subselect.cc:1364`）：

```cpp
bool Item_exists_subselect::translate(bool &null_v, bool v) {
  if (null_v) {                       // Naked IN returns UNKNOWN
    assert(substype() != EXISTS_SUBS);
    switch (value_transform) {
      case BOOL_IDENTITY:
      case BOOL_NEGATED:  return false;
      case BOOL_IS_TRUE:
      case BOOL_IS_FALSE: null_v = false; return false;
      case BOOL_NOT_TRUE:
      case BOOL_NOT_FALSE: null_v = false; return true;
      default: assert(false); return false;
    }
  }
  // Naked IN returns 'v'
  switch (value_transform) {
    case BOOL_IDENTITY: case BOOL_IS_TRUE: case BOOL_NOT_FALSE: ...
  }
}
```

> ⚠️ **`Item_in_subselect::val_bool()` 永远走不到**——函数体是 `assert(false); my_error(ER_INTERNAL_ERROR...)`。真正入口是 **`val_bool_naked()`**（`:1647`）：
> ```cpp
> bool Item_in_subselect::val_bool_naked() {
>   if (exec(current_thd)) { reset(); return false; }
>   if (was_null && !value) null_value = true;
>   assert(optimizer);   // 取反 / IS TRUE 一律留给 Item_in_optimizer
>   return value;
> }
> ```

SQL 里 `oe IN (subq)` 有 UNKNOWN，`EXISTS` 只有 TRUE/FALSE。还原三值语义的两件武器：

**`Item_func_trig_cond` 的三种触发类型（item_cmpfunc.h:814）**：

```cpp
enum enum_trig_type {
  IS_NOT_NULL_COMPL,      // 行被 NULL-complement 时禁用（外层 LEFT JOIN）
  FOUND_MATCH,            // 未找到匹配行时禁用（WHERE 下推 + anti-join）
  OUTER_FIELD_IS_NOT_NULL // IN→EXISTS：outer_field 为 NULL 时禁用
};
```

求值语义（item_cmpfunc.cc:7123）：`return *trig_var ? args[0]->val_int() : 1;`——触发器为假时返回 TRUE(1)（短路，不影响外层 AND/OR）。

**`Item_ref_null_helper`（item.h:6162）**：对内层 SELECT 列表第 0 列的 `Item_ref` 包装，`used_tables()` 故意加 `RAND_TABLE_BIT`，防止被优化器从 HAVING 移到 WHERE（聚合场景下内层值分组后才有意义）。

| 触发类型 | 场景 | 位置 |
|---|---|---|
| `OUTER_FIELD_IS_NOT_NULL` | IN→EXISTS：外层左值为 NULL 时短路谓词 | item_subselect.cc:1966/2006/2036 等 |
| `IS_NOT_NULL_COMPL` | 外层 join 条件下推：NULL-complemented 行不参与 | sql_optimizer.cc:8676/8722 |
| `FOUND_MATCH` | anti-join 的 IS NULL：匹配状态确定后才检测 | sql_optimizer.cc:8644 |

---

## 四、IN → 物化与代价比较

### 4.1 决策时机

`decide_subquery_strategy`（sql_optimizer.cc:11061），在 `JOIN::optimize` 里调用（`sql_optimizer.cc:5400`）。此时已先算了 EXISTS 计划，`compare_costs_of_subquery_strategies` 复用它做基线。

### 4.2 允许策略集（`Query_block::subquery_strategy`，sql_lex.cc:4525）

```cpp
if (m_windows.elements > 0)
  return Subquery_strategy::SUBQ_MATERIALIZATION;   // 有窗口函数 → 强制物化
if (opt_hints_qb) {
  Subquery_strategy strategy = opt_hints_qb->subquery_strategy();
  if (strategy != UNSPECIFIED) return strategy;     // SUBQUERY() hint 优先
}
if (thd->optimizer_switch_flag(OPTIMIZER_SWITCH_MATERIALIZATION))
  return thd->optimizer_switch_flag(OPTIMIZER_SWITCH_SUBQ_MAT_COST_BASED)
             ? CANDIDATE_FOR_IN2EXISTS_OR_MAT          // 代价比较
             : SUBQ_MATERIALIZATION;                   // 无条件物化
return SUBQ_EXISTS;                                    // 关闭物化
```

### 4.3 成本比较（compare_costs_of_subquery_strategies，:11122）

```cpp
const double saved_best_read = best_read;          // 保存 EXISTS 基线
...
if (in_pred->in2exists_added_to_where()) {
  // 需要为"无外层引用"的物化形态重算计划
  allow_outer_refs = false;
  optimize_semijoin_nests_for_materialization(this);
  Optimize_table_order(thd, this, nullptr).choose_table_order();
}
Semijoin_mat_optimize sjm;
calculate_materialization_costs(this, nullptr, primary_tables, &sjm);

const double subq_executions = calculate_subquery_executions(in_pred, trace);
const double cost_exists = subq_executions * saved_best_read;
const double cost_mat_table = sjm.materialization_cost.total_cost();
const double cost_mat = cost_mat_table + subq_executions * sjm.lookup_cost.total_cost();
const bool mat_chosen = cost_mat < cost_exists;
```

**关键**：`subquery_executions` 沿父 join 的 `prefix_rowcount` 累乘（`calculate_subquery_executions`，:11232）——这是 EXISTS 的软肋（每外层行重跑子查询），也是物化（建表一次 + 每次探测）胜出的关键。

```cpp
// calculate_subquery_executions 核心
double parent_fanout;
if (subquery->in_cond_of_tab != NO_PLAN_IDX) {
  const uint idx = subquery->in_cond_of_tab;
  parent_fanout = parent_join->qep_tab[idx].position()->rows_fetched;
  if (idx > parent_join->const_tables)
    parent_fanout *= parent_tab[-1].position()->prefix_rowcount;  // 累乘前缀行数
} else {
  parent_fanout = parent_join->best_rowcount;       // SELECT/HAVING 里的子查询
}
subquery_executions *= parent_fanout;               // 逐层累乘
```

### 4.4 物化可行性（subquery_allows_materialization，sql_resolver.cc:984）

⚠️ **勘误**：它**不检查 LIMIT**（LIMIT 已在 `select_in_like_transformer` 的 `#ifndef NDEBUG` 里被 assert 排除）。真实拒绝条件共 **10 条**，每条都带 `cause` 字符串（会写进 optimizer trace）：

```cpp
bool Item_in_subselect::subquery_allows_materialization(
    THD *thd, Query_block *query_block, const Query_block *outer) {
  const uint elements = unit->first_query_block()->num_visible_fields();
  assert(elements >= 1);
  assert(left_expr->cols() == elements);

  OPT_TRACE_TRANSFORM(&thd->opt_trace, trace_wrapper, trace_mat,
                      query_block->select_number, "IN (SELECT)",
                      "materialization");

  const char *cause = nullptr;
  if (substype() != Item_subselect::IN_SUBS) {
    // Subq-mat cannot handle 'outer_expr > {ANY|ALL}(subq)'...
    cause = "not an IN predicate";                                     // ① 必须是 IN
  } else if (m_subquery_used_tables & RAND_TABLE_BIT) {
    // Subquery with a random function cannot be materalized.
    // But random function in left expression is OK
    cause = "non-deterministic";                                       // ② 非确定性
  } else if (!query_block->is_simple_query_block()) {
    cause = "in set operation or a parenthesized query expression";    // ③ UNION / 带括号
  } else if (!query_block->master_query_expression()
                  ->first_query_block()->leaf_tables) {
    cause = "no inner tables";                                         // ④ 无内表
  } else if (!outer->join) {
    /*
      Maybe this is a subquery of a single table UPDATE/DELETE (TODO:
      handle this by switching to multi-table UPDATE/DELETE).
    */
    cause = "parent query has no JOIN";                                // ⑤ 父无 JOIN
  } else if (!outer->leaf_tables) {
    // The upper query is SELECT ... FROM DUAL. No gain in materializing.
    cause = "no tables in outer query";                                // ⑥ 父无表
  } else if (dependent_before_in2exists()) {
    /*
      Subquery should not be correlated; the correlation due to predicates
      injected by IN->EXISTS does not count as we will remove them if we
      choose materialization.
      TODO: This is an overly restrictive condition. It can be extended to:
         (Subquery is non-correlated ||
          Subquery is correlated to any query outer to IN predicate ||
          (Subquery is correlated to the immediate outer query &&
           Subquery !contains {GROUP BY, ORDER BY [LIMIT],
           aggregate functions}) && subquery predicate is not under "NOT IN"))
    */
    cause = "correlated";                                              // ⑦ 原生相关
  } else {
    /*
      Check that involved expression types allow materialization.
      This is a temporary fix for BUG#36752; ...
    */
    assert(left_expr->fixed);
    bool has_nullables = left_expr->is_nullable();

    uint i = 0;
    for (Item *const inner_item : unit->first_query_block()->visible_fields()) {
      Item *const outer_item = left_expr->element_index(i++);
      if (!types_allow_materialization(outer_item, inner_item)) {
        cause = "type mismatch";                                       // ⑧ 类型不允许
        break;
      }
      if (inner_item->is_blob_field()) {
        cause = "inner blob";                                          // ⑨ 内层 BLOB
        break;
      }
      has_nullables |= inner_item->is_nullable();
    }

    if (!cause) {
      trace_mat.add("has_nullable_expressions", has_nullables);
      /*
        Subquery materialization cannot handle NULLs partial matching
        properly, yet. If the outer or inner values are NULL, the
        subselect_hash_sj_engine may reply FALSE when it should reply UNKNOWN.
        So, we must limit it to those three cases:
        - when FALSE and UNKNOWN are equivalent answers. I.e. this is a
        top-level predicate (this implies it is not negated).
        - when outer and inner values cannot be NULL.
        - when there is a single inner column (because for this we have a
        limited implementation of NULLs partial matching).
      */
      trace_mat.add("treat_UNKNOWN_as_FALSE", abort_on_null);

      if (!abort_on_null && has_nullables && (elements > 1))
        cause = "cannot_handle_partial_matches";                       // ⑩ NULL 部分匹配
      else {
        trace_mat.add("possible", true);
        return true;
      }
    }
  }
  assert(cause != nullptr);
  trace_mat.add("possible", false).add_alnum("cause", cause);
  return false;
}
```

**物化的"反悔"：`remove_in2exists_conds()`**（`item_subselect.cc:406`）：

```cpp
Item *remove_in2exists_conds(Item *conds) {
  bool modified = false;
  List<Item> new_conds;
  if (WalkConjunction(conds, [&modified, &new_conds](Item *cond) {
        if (cond->created_by_in2exists()) {
          modified = true;
          return false;      // ★ 丢弃带 created_by_in2exists 标记的合取项
        } else {
          return new_conds.push_back(cond);
        }
      })) {
    return nullptr;
  }
  return modified ? CreateConjunction(&new_conds) : conds;
}
```

> ★ 所有 IN2EXISTS 注入的条件都打了 `set_created_by_in2exists()` 标记——这就是"能精确撤销"的前提。

**两条路线的最终落地对比**：

```cpp
// EXISTS 路线（item_subselect.cc:383）—— 加 LIMIT 1
bool Item_in_subselect::finalize_exists_transform(THD *thd, Query_block *qb) {
  if (!(unit->global_parameters()->select_limit = new Item_int(1))) return true;
  unit->global_parameters()->m_internal_limit = true;
  if (unit->set_limit(thd, unit->global_parameters())) return true;
  if (unit->finalize(thd)) return true;
  qb->join->allow_outer_refs = true;      // for JOIN::set_prefix_tables()
  strategy = Subquery_strategy::SUBQ_EXISTS;
  return false;
}

// 物化路线（item_subselect.cc:422）—— 撤销 in2exists + 去相关 + 挂引擎
bool Item_in_subselect::finalize_materialization_transform(THD *thd, JOIN *join) {
  assert(unit->global_parameters()->select_limit == nullptr);
  strategy = Subquery_strategy::SUBQ_MATERIALIZATION;

  if (join->where_cond)  join->where_cond  = remove_in2exists_conds(join->where_cond);
  if (join->having_cond) join->having_cond = remove_in2exists_conds(join->having_cond);
  unit->change_to_access_path_without_in2exists(thd);   // ★ hypergraph 专用
  assert(!in2exists_info->dependent_before);
  if (unit->finalize(thd)) return true;

  join->query_block->uncacheable &= ~UNCACHEABLE_DEPENDENT;
  unit->uncacheable &= ~UNCACHEABLE_DEPENDENT;          // ★ 去相关

  subselect_hash_sj_engine *const new_engine =
      new (thd->mem_root) subselect_hash_sj_engine(this, unit);
  if (!new_engine) return true;
  if (new_engine->setup(thd, *unit->get_unit_column_types())) {
    new_engine->cleanup(); destroy(new_engine); return true;
  }
  indexsubquery_engine = new_engine;                    // ★★ 引擎挂载点
  join->allow_outer_refs = false;
  return false;
}
```

> ⚠️ **不存在** `Item_in_subselect::setup_engine()` / `create_engine()`。引擎挂载就上面这一处（物化），加 `sql_optimizer.cc:1446` 的 `set_indexsubquery_engine()`（index lookup）。

不能物化的清单：

| 条件 | 原因 |
|------|------|
| 非 `IN_SUBS` | 仅 IN 可物化（非 ANY/ALL） |
| 内层有随机函数 | 非确定 |
| 非简单块（UNION） | 结构复杂 |
| 内层无表 | 无物化意义 |
| 外层无 JOIN | 单表 UPDATE/DELETE |
| 外层无表 | FROM DUAL |
| 原本相关（`dependent_before_in2exists`） | 相关不能物化（IN2EXISTS 注入不算） |
| 类型不允许 / 内层 BLOB | BUG#36752 |
| 多列且可空且 `!abort_on_null` | 无法处理 NULL 部分匹配 |

### 4.5 落地（finalize_materialization_transform，item_subselect.cc:422）

```cpp
strategy = Subquery_strategy::SUBQ_MATERIALIZATION;
// 撤销 IN->EXISTS 注入的条件
if (join->where_cond)  join->where_cond  = remove_in2exists_conds(join->where_cond);
if (join->having_cond) join->having_cond = remove_in2exists_conds(join->having_cond);
join->query_block->uncacheable &= ~UNCACHEABLE_DEPENDENT;  // 不再相关
subselect_hash_sj_engine *new_engine = new (thd->mem_root) subselect_hash_sj_engine(this, unit);
new_engine->setup(thd, *unit->get_unit_column_types());   // 建去重临时表
indexsubquery_engine = new_engine;
```

`remove_in2exists_conds`（item_subselect.cc:406）按 `created_by_in2exists` 标记删除所有注入谓词。

### 4.6 subselect_hash_sj_engine 执行期原理

⚠️ **勘误**：**不存在 `subselect_uniquesubquery_engine`**（8.0.39 已删除，全仓 0 命中）。只剩两个引擎，枚举是三项（`item_subselect.h:120`）：

```cpp
  enum enum_engine_type { OTHER_ENGINE, INDEXSUBQUERY_ENGINE, HASH_SJ_ENGINE };
```

**类声明**（`item_subselect.h:874`）——注意 `mat_table_has_nulls` 三态：

```cpp
class subselect_hash_sj_engine final : public subselect_indexsubquery_engine {
 private:
  /* true if the subquery was materialized into a temp table. */
  bool is_materialized;
  // true if we know for sure that there are zero rows in the table.
  bool has_zero_rows = false;
  /**
     Existence of inner NULLs in materialized table:
     By design, other values than IRRELEVANT_OR_FALSE are possible only if the
     subquery has only one inner expression.
  */
  enum nulls_exist {
    NEX_IRRELEVANT_OR_FALSE = 0,   /// none, or they don't matter
    NEX_UNKNOWN = 1,               /// they matter, and we don't know yet
    NEX_TRUE = 2                   /// they matter, and we know there exists one
  };
  enum nulls_exist mat_table_has_nulls;
  Query_expression *const unit;
  unique_ptr_destroy_only<RowIterator> m_iterator;
  AccessPath *m_root_access_path;
  Query_result_interceptor *saved_result{nullptr};

 public:
  bool setup(THD *thd, const mem_root_deque<Item *> &tmp_columns);
  void cleanup() override;
  bool exec(THD *thd) override;
  enum_engine_type engine_type() const override { return HASH_SJ_ENGINE; }
};
```

**`setup()`：去重的落点**（`item_subselect.cc:3248`，节选）：

```cpp
  if (!(tmp_result_sink = new (thd->mem_root) Query_result_union()))
    return true;
  if (tmp_result_sink->create_result_table(
          thd, tmp_columns,
          true,  // Eliminate duplicates          ★★ 去重就在这里
          thd->variables.option_bits | TMP_TABLE_ALL_COLUMNS,
          "<materialized_subquery>", true, true))
    return true;
  //  ★ 临时表在最左前缀上建唯一（或 hash）索引，重复行插入时被拒

  tmp_table = tmp_result_sink->table;
  tmp_key = tmp_table->key_info;
  ...
  type = (tmp_table->key_info[0].flags & HA_NOSAME) ? JT_EQ_REF : JT_REF;
  ...
  ref.items[part_no] = item->left_expr->element_index(part_no);
  //  ★ 探测用的 key 直接来自外层 left_expr（不经过 Item_cache）
  ...
    if (nullable &&  // nullable column in tmp table,
                     // and UNKNOWN should not be interpreted as FALSE
        !item->abort_on_null) {
      assert(tmp_key_parts == 1);   // 必须是单列
      ref.null_ref_key = cur_ref_buff;      // ★ 准备"搜 NULL"的 key buffer
      mat_table_has_nulls = NEX_UNKNOWN;    // ★ "可能有内 NULL，还不知道"
    } else {
      ref.null_ref_key = nullptr;
      mat_table_has_nulls = NEX_IRRELEVANT_OR_FALSE;
    }
```

**`exec()`：懒物化 + NULL 处理**（`item_subselect.cc:3531`，节选）：

```cpp
bool subselect_hash_sj_engine::exec(THD *thd) {
  if (!is_materialized) {
    thd->lex->set_current_query_block(unit->first_query_block());
    assert(unit->first_query_block()->master_query_expression()->is_optimized());

    // Init() triggers materialization.
    bool error = m_iterator->Init();       // ★★ 第一次执行才物化（懒物化）
    if (error || thd->is_fatal_error()) return true;
    is_materialized = true;

    // See if we have zero rows or not.
    table->file->info(HA_STATUS_VARIABLE);
    if (table->file->ha_table_flags() & HA_STATS_RECORDS_IS_EXACT) {
      has_zero_rows = (table->file->stats.records == 0);
    } else {
      if (table->file->inited) table->file->ha_index_or_rnd_end();
      TableScanIterator scan(thd, table, /*expected_rows=*/-1.0, nullptr);
      int ret = scan.Read();
      if (ret == 1 || thd->is_error()) return true;
      has_zero_rows = (ret == -1);
    }
  }

  if (has_zero_rows) { item->value = false; return false; }   // 正确答案为 FALSE

  if (item->left_expr->element_index(0)->null_value) {
    /*
      The first outer expression oe1 is NULL. ... Do as if searching with all
      triggered conditions disabled: this would surely find a row. The caller
      will translate this to UNKNOWN.
    */
    assert(item->left_expr->cols() == 1);
    item->value = true;     // ★★ 谎报"找到了行"，交由 Item_in_optimizer 翻译成 UNKNOWN
    return false;
  }

  hash = 0;
  bool found;
  if (ExecuteExistsQuery(thd, item->unit, m_iterator.get(), &found)) return true;
  item->value = found;
  item->assigned(true);

  if (!found &&  // no exact match
      mat_table_has_nulls != NEX_IRRELEVANT_OR_FALSE) {
    if (mat_table_has_nulls == NEX_UNKNOWN) {   // We do not know yet
      *ref.null_ref_key = 1;                    // ★ 把 NULL 标志字节置 1
      if (!table->file->inited &&
          table->file->ha_index_init(ref.key, false)) return true;
      if (safe_index_read(table, ref) == 1) return true;
      *ref.null_ref_key = 0;                    // prepare for next searches
      mat_table_has_nulls =
          table->has_row() ? NEX_TRUE : NEX_IRRELEVANT_OR_FALSE;
      //                     ★ "有没有内 NULL"只查一次，之后记忆化
    }
    if (mat_table_has_nulls == NEX_TRUE) {
      /*
        There exists an inner NULL. The correct answer is UNKNOWN.
        Do as if searching with all triggered conditions enabled; that
        would not find any match, but Item_is_not_null_test would notice a
        NULL:
      */
      item->value = false;
      item->was_null = true;   // ★★ value=FALSE + was_null=TRUE → 上层读成 UNKNOWN
    }
  }
  return false;
}
```

> ★★ **去重 + NULL 的小结（源码级）**：
> - **去重** = `create_result_table(..., /*Eliminate duplicates=*/true, ...)`，靠临时表上的唯一 / hash 索引。
> - **NULL** 只在**单列 + 可为 NULL + 需要区分 NULL/FALSE** 时才处理：`ref.null_ref_key` 指向 key buffer 的 NULL 标志字节，`exec()` 在未命中时把它置 1 做一次"搜 NULL"，命中则 `was_null = true`。**多列情形根本进不了物化**（`subquery_allows_materialization` 的 `cannot_handle_partial_matches`）。

**`cleanup()` 与生命周期**（`:3484`）：

```cpp
void subselect_hash_sj_engine::cleanup() {
  is_materialized = false;
  if (result != nullptr) result->cleanup();   // Resets the temp table as well
  m_root_access_path = nullptr;
  m_iterator.reset();
  if (table != nullptr) {
    if (table->file->inited) table->file->ha_index_end();
    close_tmp_table(table);
    free_tmp_table(table);
    table = nullptr;
  }
  if (unit->is_executed()) unit->reset_executed();
}
```

> ★ `Item_subselect::exec()`（`:634`）里有句注释说明了分派：`subselect_hash_sj_engine` **自建迭代器，不走 `subquery->exec()`**：
> ```cpp
>   bool should_create_iterators =
>       !(indexsubquery_engine != nullptr &&
>         indexsubquery_engine->engine_type() ==
>             subselect_indexsubquery_engine::HASH_SJ_ENGINE);
>   ...
>   if (indexsubquery_engine != nullptr) {
>     return indexsubquery_engine->exec(thd);
>   } else {
>     return subquery->exec(thd);
>   }
> ```

- **建表（一次）**：`Query_result_union::create_result_table(..., true /*distinct*/)` 建带唯一索引的临时表，`m_iterator->Init()` 执行子查询并去重写入。
- **探测**：`ExecuteExistsQuery` 用基于 `NewRefAccessPath` 的迭代器做索引查找，配合等值 post-filter 过滤假阳性。
- **NULL 三值修正**：未找到匹配且内层可能有 NULL 时，二次探测 `null_ref_key`，把结果修正为 UNKNOWN（`was_null=true`）。

---

## 五、NOT IN / NOT EXISTS → anti-join

### 5.1 choose_semijoin_or_antijoin（item_subselect.cc:1533）

MySQL **没有独立 `convert_subquery_to_antijoin`**——anti-join 与 semi-join 共用 `convert_subquery_to_semijoin`，由 `can_do_aj` 区分。

```cpp
bool Item_exists_subselect::choose_semijoin_or_antijoin() {
  can_do_aj = false;
  bool might_do_sj = false, might_do_aj = false, null_problem = false;
  switch (value_transform) {
    case BOOL_IS_TRUE:    might_do_sj = true; break;   // IN / EXISTS
    case BOOL_NOT_TRUE:   might_do_aj = true; break;   // NOT IN / NOT EXISTS
    case BOOL_IS_FALSE:   might_do_aj = true; null_problem = true; break;
    case BOOL_NOT_FALSE:  might_do_sj = true; null_problem = true; break;
    default:              return false;
  }
  if (substype() == EXISTS_SUBS) null_problem = false;  // EXISTS 永不返回 NULL
  if (null_problem) {
    if (down_cast<Item_in_subselect *>(this)->left_expr->is_nullable())
      return false;                                     // 可空 → 退回
    for (Item *inner : unit->first_query_block()->visible_fields())
      if (inner->is_nullable()) return false;
  }
  can_do_aj = might_do_aj;
  return true;
}
```

四种 `value_transform` 判定表：

| value_transform | 语义 | 转换方向 | null_problem |
|---|---|---|---|
| `BOOL_IS_TRUE` | `x IN (subq) IS TRUE` | semijoin | 否 |
| `BOOL_NOT_TRUE` | `x IN (subq) IS NOT TRUE`（NOT IN） | **antijoin** | 否 |
| `BOOL_IS_FALSE` | `x IN (subq) IS FALSE` | **antijoin** | **是** |
| `BOOL_NOT_FALSE` | `x IN (subq) IS NOT FALSE` | semijoin | **是** |

### 5.2 三值陷阱

**关键**：`NOT IN` 语义是 `NOT (x IN subq)`，三值下 `NOT UNKNOWN = UNKNOWN`，而 anti-join 只有 TRUE/FALSE。所以 LHS 或 inner 可空时不能做 anti-join（`null_problem` 检测到可空直接 `return false` 退回 IN2EXISTS/物化）。`NOT EXISTS` 恒非 NULL，无条件可做。

`NOT UNKNOWN = UNKNOWN` 由 `translate()` 精确建模（item_subselect.cc:1364）：

```cpp
bool Item_exists_subselect::translate(bool &null_v, bool v) {
  if (null_v) {  // 裸 IN 返回 UNKNOWN
    switch (value_transform) {
      case BOOL_IDENTITY: case BOOL_NEGATED:
        return false;                    // UNKNOWN 保持（NOT UNKNOWN = UNKNOWN）
      case BOOL_IS_TRUE: case BOOL_IS_FALSE:
        null_v = false; return false;    // UNKNOWN IS TRUE → FALSE
      case BOOL_NOT_TRUE: case BOOL_NOT_FALSE:
        null_v = false; return true;     // UNKNOWN IS NOT TRUE → TRUE
    }
  }
  ...
}
```

### 5.3 LEFT JOIN + IS NULL 建模

`convert_subquery_to_semijoin` 的 anti-join 分支（sql_resolver.cc:3117）把 `(aj-left-nest)` 与 `(aj-nest)` 用 LEFT JOIN 连接；`x IS NULL` 谓词**留到优化阶段才加**（`JOIN::attach_join_conditions`，sql_optimizer.cc:8830）：

```cpp
if (lt->table_ref->embedding && lt->table_ref->embedding->is_aj_nest() &&
    last_tab == lt->first_inner() && !lt->table_ref->join_cond()) {
  Item *cond = new Item_func_false();     // 哨兵：代表 IS NULL
  cond->item_name.set(antijoin_null_cond);  // "<ANTIJOIN-NULL>"
  cond = add_found_match_trig_cond(this, last_tab, cond, lt->first_upper());
  cond = new Item_func_trig_cond(cond, nullptr, this, last_tab,
                                 Item_func_trig_cond::IS_NOT_NULL_COMPL);
  ...
  lt->table()->reginfo.not_exists_optimize = true;   // NOT EXISTS 短路优化
```

双层 `trig_cond`：外层 `IS_NOT_NULL_COMPL`（B 为 NULL-complemented 时为假 → IS NULL 检测成立）、内层 `FOUND_MATCH`（匹配状态确定后才评估）。等价于 `WHERE B.col IS NULL`（B 无匹配 → NULL-complemented 行 → 保留）。

---

## 六、ANY / ALL → MIN/MAX

### 6.1 single_value_transformer（item_subselect.cc:1730）

触发条件：非等值比较（`!eqne_op()`）、非相关（`!unit->uncacheable`）、三值语义安全。

```cpp
if (!func->eqne_op() && !unit->uncacheable &&
    (abort_on_null || (upper_item && upper_item->ignore_unknown()) ||
     (!left_expr->is_nullable() && !subquery_maybe_null))) {
  Item_sum_hybrid *item;
  if (func->l_op()) {
    // (ALL && (> || >=)) || (ANY && (< || <=))
    item = new Item_sum_max(select->base_ref_items[0]);
  } else {
    // (ALL && (< || <=)) || (ANY && (> || >=))
    item = new Item_sum_min(select->base_ref_items[0]);
  }
  select->base_ref_items[0] = item;          // 覆写 SELECT 列表第一位置
  ...
  subquery = new Item_singlerow_subselect(select);   // 包装为标量子查询
  substitution = func->create(left_expr, subquery);  // b > (SELECT MAX(a) ...)
}
```

### 6.2 l_op() 与方向反转

`l_op()` 反映的是"**ANY 方向下的左偏序**"（`<`/`<=` 为 true，`>`/`>=` 为 false）。但 ALL 场景方向被 invert：

| 原谓词 | creator | l_op() | 构造 | 结果 |
|---|---|---|---|---|
| `b > ANY(subq)` | Gt(>) | false | Item_sum_min | `b > MIN(a)` |
| `b < ANY(subq)` | Lt(<) | true | Item_sum_max | `b < MAX(a)` |
| `b > ALL(subq)` | Gt(>) | false | **Item_sum_min→MAX** | `b > MAX(a)` |
| `b < ALL(subq)` | Lt(<) | true | **Item_sum_max→MIN** | `b < MIN(a)` |

关键点：**`l_op()` 标记的是"原始 ANY 方向的左偏序"，ALL 场景方向被 invert 后聚合选择随之翻转**——这正是 `b > ALL` 取 MAX、`b > ANY` 取 MIN 的根因。

---

## 七、标量子查询 → derived

### 7.0 入口 `transform_scalar_subqueries_to_join_with_derived`（8.0.16，逐行剖析）

`sql_resolver.cc:7577`，`Query_block::prepare` 里调用（`:471`）。把 SELECT/WHERE/HAVING/JOIN cond 里的**标量子查询**改写成 `LEFT JOIN derived`，消除"逐外层行执行子查询"的开销。

```cpp
// sql_resolver.cc:7577
bool Query_block::transform_scalar_subqueries_to_join_with_derived(THD *thd) {
  if (thd->lex->m_subquery_to_derived_is_impossible) return false;  // ① 全局开关/不兼容
  if (leaf_table_count == 0 || thd->lex->set_var_list.elements > 0) // ② 前置门槛
    return false;

  Item::Collect_scalar_subquery_info subqueries;                   // ③ 收集容器

  // ④ 先收集 JOIN cond（必须最先，因为它要嵌套在外连接表之后、内表之前）
  if (walk_join_conditions(m_table_nest, [&](Item **expr_p) { ... collect ... }, &subqueries))
    return true;
  // ⑤ 再收集 WHERE
  // ⑥ 再收集 SELECT list（visible_fields）
  // ⑦ 再收集 HAVING

  // ⑧ 隐式分组（无 GROUP BY 但有聚合）时，可能需要先 transform_grouped_to_derived
  if (is_implicitly_grouped()) {
    bool need_new_outer = false;
    for (auto subquery : subqueries.m_list) {
      if (!query_block_contains_subquery(this, subq->unit)) continue;
      if (subquery.m_location & L_SELECT) need_new_outer = true;  // SELECT 里有标量子查询
      if (subquery.m_location & L_HAVING) return false;            // HAVING 有 → 放弃
    }
    if (need_new_outer) {                                          // ⑨ 先转分组为 derived
      bool break_off = false;
      if (transform_grouped_to_derived(thd, &break_off)) return true;
      if (break_off) return false;
    }
  }

  // ⑩ 逐个转 derived（见 7.2）
  for (auto subquery : subqueries.m_list) { ... transform_subquery_to_derived ... }
}
```

逐行解释的关键点：

1. **① `m_subquery_to_derived_is_impossible`**：一个"全局放弃"标志。一旦某处发现当前查询无法做标量→derived（如 HAVING 含子查询、多列子查询等），置位后整个转换跳过。
2. **② 两个门槛**：`leaf_table_count == 0`（`SELECT (SELECT ...)` 这种无 FROM 表的裸查询没法 LEFT JOIN）；`set_var_list.elements > 0`（有 `@a :=` 用户变量赋值，改写会改变变量求值顺序）。
3. **④ 收集顺序是硬约束**（源码注释 7585-7591）：**JOIN cond 里的必须先收集**，因为它要嵌套在外连接表之后、内表之前；其它位置的标量子查询要等 JOIN cond 就位后再加。
4. **⑧ 隐式分组的特殊处理**：`SELECT SUM(a), (SELECT SUM(b) FROM t3) FROM t1` 这种无 GROUP BY 但带聚合 + 标量子查询的查询，直接 LEFT JOIN derived 会算错——必须**先把聚合部分 `transform_grouped_to_derived` 转成一个 derived**，再对剩下的标量子查询做转换。若 HAVING 里也有子查询则直接 `return false` 放弃（源码注释给了两个反面例子）。

### 7.1 收集（sql_resolver.cc:7594）

按 JOIN cond → WHERE → SELECT → HAVING 顺序 `walk` 收集标量子查询到 `m_list`。

### 7.2 逐个子查询转 derived（:7707）

```cpp
for (auto subquery : subqueries.m_list) {
  Item_singlerow_subselect *const subq = subquery.item;
  if (query_block_contains_subquery(this, subs_query_expression) ||
      (subq->const_item() && subs_query_expression->is_optimized()))
    continue;
  bool needs_cardinality_check = !subquery.m_implicitly_grouped_and_no_union;
  Item *lifted_where = nullptr;
  if (subquery.m_correlation_map != 0) {   // 相关子查询
    if (subs_query_expression->first_query_block()
            ->supported_correlated_scalar_subquery(thd, &subquery, &lifted_where))
      return true;
    if (lifted_where == nullptr) continue; // 相关但不可提 → 跳过
    needs_cardinality_check = false;       // 相关子查询用 GROUP BY 物化
  }
  transform_subquery_to_derived(thd, &tl, subs_query_expression, subq,
                                /*use_inner_join=*/false,  // LEFT JOIN
                                needs_cardinality_check, ...);
```

```sql
SELECT (SELECT COUNT(a) FROM t2) + a FROM t1
=> SELECT derived.cnt + t1.a FROM t1
     LEFT JOIN (SELECT COUNT(a) AS cnt FROM t2) derived ON TRUE
```

**用 LEFT JOIN**（`use_inner_join=false`）——标量子查询空集返回 NULL，不能丢外层行。

### 7.3 needs_cardinality_check 运行时检查

为 scalar 子查询追加 `COUNT(*)` 列，并在 JOIN 条件上挂 `COUNT(*) <= 1`（用 `Item_func_reject_if` 表示"违反即报错"，对应标量子查询返回多行报 `ER_SUBQUERY_NO_1_ROW`）。相关标量子查询走 GROUP BY 物化，天然每组一行，故 `needs_cardinality_check=false`。

### 7.4 相关标量子查询：相关谓词提升为 GROUP BY 键

`supported_correlated_scalar_subquery`（:7350）从 WHERE 分离相关谓词；`decorrelate_derived_scalar_subquery_pre`（:6852）把提升谓词里的内层字段加进 SELECT 列表与 GROUP BY。`(内层字段, 外层字段)` 等值谓词提升为 JOIN 条件，内层字段成为 derived 的 GROUP BY 键——每个外层相关值对应 derived 一组（一行），等价于"逐外层行执行相关子查询"。不满足"单列字段等值"的谓词（如 `t1.a > t2.b`）被 `is_correlated_predicate_eligible` 拒绝。

---

## 八、flatten_subqueries 与恒假删除

`sql_resolver.cc:3811`。semi-join / derived 候选在这里被真正拍平。四步：

### 8.1 自底向上排序（:3844）

```cpp
subq_item->sj_convert_priority =
    (((dependent * MAX_TABLES_FOR_SIZE) +  // 相关优先
      child_query_block->leaf_table_count) * 65536) +  // 表多者次之
    (65536 - subq_no);                       // 位置靠前者再优先
std::sort(..., [](a, b) { return a->sj_convert_priority > b->sj_convert_priority; });
```

### 8.2 恒假检测 + 真值替换（:3913）

```cpp
const uint tables_added = subq_item->unit->first_query_block()->leaf_table_count + 1;
if (table_count + tables_added <= MAX_TABLES &&         // (1) 表数预算
    !subq_item->unit->first_query_block()->has_aj_nests)  // (2)
  subq_item->strategy = Subquery_strategy::SEMIJOIN;

Item *subq_where = subq_item->unit->first_query_block()->where_cond();
bool cond_value = true;
if (subq_where && subq_where->const_item() && ... &&
    simplify_const_condition(thd, &subq_where, false, &cond_value))
  return true;
if (!cond_value) {                                     // 恒假 → 删除子查询
  subq_item->walk(&Item::clean_up_after_removal, ...);
}
// 在父 WHERE/ON 用 TRUE/FALSE 替换 IN(subq)
Item *truth_item = (cond_value || subq_item->can_do_aj)
    ? new Item_func_true() : new Item_func_false();
replace_subcondition(thd, tree, subq_item, truth_item, false);
```

### 8.3 转 semi-join / 兜底 IN2EXISTS（:3979）

`SEMIJOIN` 的调 `convert_subquery_to_semijoin`；仍 `UNSPECIFIED` 的（因表数预算或 anti-join 限制退回）重新走 `select_transformer` 做 IN2EXISTS。

### 8.4 与 04 篇 merge_derived 的关系

| | flatten_subqueries | merge_derived（04 篇） |
|---|---|---|
| 对象 | WHERE/ON 里的**子查询谓词** | FROM 列表里的**派生表** |
| 动作 | IN/EXISTS 的表并入父 join（semi/anti nest）或转 derived | derived table 合并回外层 |
| 开关 | semi-join / subquery_to_derived | derived_merge |

两者都在 resolver 阶段，但作用于不同对象。`flatten_subqueries` 里 `CANDIDATE_FOR_DERIVED_TABLE` 生成的 derived 是 **materialized derived**（不参与 merge_derived）。

---

## 相关参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `subquery_materialization_cost_based` | **ON** | 物化与否走代价比较（`OPTIMIZER_SWITCH_SUBQ_MAT_COST_BASED`，bit 15） |
| `derived_merge` | **ON** | 派生表合并（04 篇，`OPTIMIZER_SWITCH_DERIVED_MERGE`，bit 18） |
| `subquery_to_derived` | **OFF** | ⚠ 标量→derived / IN→joined derived 的开关（bit 22，**不在默认集合**） |

> ⚠️ **关键纠正**：`subquery_to_derived` 默认是 **OFF**，不是常见误解的"默认 on"。这意味着第七节的标量→derived、第二节的 IN→derived 在默认会话下**不会走**（除非 secondary engine 要求）。默认 ON 的只有 `subquery_materialization_cost_based` 和 `derived_merge`。

三个开关的位定义在 `sql/sql_const.h:214/217/221`，名字数组在 `sql/sys_vars.cc:3463/3466/3470`，默认集合在 `sql/sys_vars.cc:197-211`。

**`optimizer_switch` 的完整名称数组**（`sql/sys_vars.cc:3447`）——⚠️ 它是 `Sys_var_flagset`，**没有每个 flag 独立的 `Sys_var_*` 定义**；"定义处"= 数组条目（外部名）+ `sql_const.h` 位常量（内部名），两者顺序必须一致：

```cpp
/**
  @note
  @b BEWARE! These must have the same order as the \#defines in sql_const.h!
*/
static const char *optimizer_switch_names[] = {
    "index_merge", "index_merge_union", "index_merge_sort_union",
    "index_merge_intersection", "engine_condition_pushdown",
    "index_condition_pushdown", "mrr", "mrr_cost_based",
    "block_nested_loop", "batched_key_access",
    "materialization",                          // ← bit 10
    "semijoin",                                 // ← bit 11
    "loosescan",                                // ← bit 12
    "firstmatch",                               // ← bit 13
    "duplicateweedout",                         // ← bit 14
    "subquery_materialization_cost_based",      // ← bit 15
    "use_index_extensions", "condition_fanout_filter", "derived_merge",
    "use_invisible_indexes", "skip_scan", "hash_join",
    "subquery_to_derived",                      // ← bit 22
    "prefer_ordering_index", "hypergraph_optimizer",
    "derived_condition_pushdown", "default", NullS};
```

**位常量**（`sql/sql_const.h:196`）：

```cpp
constexpr const uint64_t OPTIMIZER_SWITCH_MATERIALIZATION{1ULL << 10};
constexpr const uint64_t OPTIMIZER_SWITCH_SEMIJOIN{1ULL << 11};
constexpr const uint64_t OPTIMIZER_SWITCH_LOOSE_SCAN{1ULL << 12};
constexpr const uint64_t OPTIMIZER_SWITCH_FIRSTMATCH{1ULL << 13};
constexpr const uint64_t OPTIMIZER_SWITCH_DUPSWEEDOUT{1ULL << 14};
constexpr const uint64_t OPTIMIZER_SWITCH_SUBQ_MAT_COST_BASED{1ULL << 15};
constexpr const uint64_t OPTIMIZER_SWITCH_DERIVED_MERGE{1ULL << 18};
constexpr const uint64_t OPTIMIZER_SWITCH_SUBQUERY_TO_DERIVED{1ULL << 22};
constexpr const uint64_t OPTIMIZER_SWITCH_PREFER_ORDERING_INDEX{1ULL << 23};
constexpr const uint64_t OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER{1ULL << 24};
constexpr const uint64_t OPTIMIZER_SWITCH_DERIVED_CONDITION_PUSHDOWN{1ULL << 25};
constexpr const uint64_t OPTIMIZER_SWITCH_LAST{1ULL << 26};
```

> ⚠️ **`subquery_materialization_cost_based` 的内部宏名是 `OPTIMIZER_SWITCH_SUBQ_MAT_COST_BASED`**（缩写 SUBQ_MAT）——按外部名去找宏会找不到。

**默认集合**（`sql/sys_vars.cc:197`）——注意 **bit 22 不在里面**：

```cpp
// Including the switch in this set, makes its default 'on'
static constexpr const unsigned long long OPTIMIZER_SWITCH_DEFAULT{
    OPTIMIZER_SWITCH_INDEX_MERGE | OPTIMIZER_SWITCH_INDEX_MERGE_UNION |
    OPTIMIZER_SWITCH_INDEX_MERGE_SORT_UNION |
    OPTIMIZER_SWITCH_INDEX_MERGE_INTERSECT |
    OPTIMIZER_SWITCH_ENGINE_CONDITION_PUSHDOWN |
    OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN | OPTIMIZER_SWITCH_MRR |
    OPTIMIZER_SWITCH_MRR_COST_BASED | OPTIMIZER_SWITCH_BNL |
    OPTIMIZER_SWITCH_MATERIALIZATION | OPTIMIZER_SWITCH_SEMIJOIN |
    OPTIMIZER_SWITCH_LOOSE_SCAN | OPTIMIZER_SWITCH_FIRSTMATCH |
    OPTIMIZER_SWITCH_DUPSWEEDOUT | OPTIMIZER_SWITCH_SUBQ_MAT_COST_BASED |
    OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS |
    OPTIMIZER_SWITCH_COND_FANOUT_FILTER | OPTIMIZER_SWITCH_DERIVED_MERGE |
    OPTIMIZER_SKIP_SCAN | OPTIMIZER_SWITCH_HASH_JOIN |
    OPTIMIZER_SWITCH_PREFER_ORDERING_INDEX |
    OPTIMIZER_SWITCH_DERIVED_CONDITION_PUSHDOWN};
    //  ★ 没有 OPTIMIZER_SWITCH_SUBQUERY_TO_DERIVED（→ off）
    //  ★ 没有 OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER（→ off）
```

**逐个核实表**：

| 外部名 | 是否存在 | 位 | 默认 | 与子查询改写的关系 |
|---|---|---|---|---|
| `semijoin` | ✅ | 11 | **on** | 半连接候选准入（`sql_resolver.cc:1532`） |
| `materialization` | ✅ | 10 | **on** | 允许走物化 |
| `subquery_materialization_cost_based` | ✅ | 15 | **on** | 物化 vs EXISTS 走代价比较；off 则强制物化 |
| `loosescan` | ✅ | 12 | **on** | 半连接执行策略之一 |
| `firstmatch` | ✅ | 13 | **on** | 半连接执行策略之一 |
| `duplicateweedout` | ✅ | 14 | **on** | 半连接执行策略之一 |
| `subquery_to_derived` | ✅ | 22 | **off** | 标量→derived / IN→joined derived |
| `derived_merge` | ✅ | 18 | **on** | 派生表合并（与子查询改写无关） |
| `derived_condition_pushdown` | ✅ | 25 | **on** | 派生表条件下推（无关） |
| `prefer_ordering_index` | ✅ | 23 | **on** | ORDER BY 优化（无关） |
| `hypergraph_optimizer` | ✅ | 24 | **off** | 影响半连接条件 6x |

> ⚠️ 常见误称：**不存在** `Query_block::semijoin_candidates`；真名 `sj_candidates`（`sql_lex.h`）。也**不存在** `Item_allany_subselect::l_op()`——`l_op()` 是 `Comp_creator` 的方法（`func->l_op()`），在 `single_value_transformer:1766` 被用来决定 MAX 还是 MIN。

**`check_optimizer_switch` 的一个反直觉行为**（`sql/sys_vars.cc:3413`）：

```cpp
  if (current_hypergraph_optimizer && !want_hypergraph_optimizer) {
    // Don't turn off the hypergraph optimizer on set optimizer_switch=DEFAULT.
    // This is so that mtr --hypergraph should not be easily cancelled in the
    // middle of a test, unless the test explicitly meant it.
    if (var->value == nullptr) {
      var->save_result.ulonglong_value |= OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER;
    }
  }
```

> ★ `SET optimizer_switch = DEFAULT` **不会**关掉已开启的 hypergraph 位。

**读取点速查**：
- `Query_block::semijoin_enabled()` 声明 `sql/sql_lex.h:1597`，定义 `sql_lex.cc:4551`
- `Query_block::subquery_strategy()` 声明 `sql/sql_lex.h:1582`，定义 `sql_lex.cc:4525`
- `transform_scalar_subqueries_to_join_with_derived()` 唯一调用点 `sql/sql_resolver.cc:471`
- `flatten_subqueries()` 调用点：`sql_resolver.cc:549`、`sql_delete.cc:872`、`sql_update.cc:1787`

---


## 参考

**论文**
- **Galindo-Legaria《Parameterized Queries and Nesting Equivalences》(2001)** —— 子查询去关联化、semi-join 转换的理论基础

**官方文档**
- *MySQL 8.0 Reference Manual → Optimizing IN/=ANY Subqueries*
- *MySQL 8.0 Reference Manual → Optimizing Subqueries with Materialization*
- *MySQL 8.0 Reference Manual → Optimizing Derived Tables and View References*

**内核月报**
- 2020/10《子查询优化（一）：从 EXISTS 到 semi-join》（IN2EXISTS 注入流程）
- 2020/10《子查询优化（二）：物化与代价决策》

**阿里云 PolarDB 官方文档**
- [《图解 MySQL 8.0 优化器对子查询 JOIN 与分区表的转换优化》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/optimizer-based-query-conversion-in-mysql-8) —— `resolve_subquery` 的 semi-join/materialization 判定条件、IN→EXISTS、ANY/ALL→MIN/MAX、标量子查询→derived（本篇决策树对齐此文）

**相关文档**
- semi-join 见 [`03_semijoin.md`](03_semijoin.md)
- 运行期执行见 [`../../runtime/02_subquery_runtime.md`](../../runtime/02_subquery_runtime.md)
- 派生表合并见 [`04_logical_join.md`](04_logical_join.md)
- 表与 `TABLE`/`TABLE_SHARE` 对象见 [`../../../table.md`](../../../table.md)

**社区文章**
- [MySQL · 源码分析 · Derived table 代码分析（梁辰）](https://zhuanlan.zhihu.com/p/443656156) —— `resolve_derived` / merge 与物化的判定条件

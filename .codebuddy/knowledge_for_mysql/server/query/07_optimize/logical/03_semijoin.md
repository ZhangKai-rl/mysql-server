# 03 Semi-join：语义、五种策略与完整执行流程

> Semi-join 是 MySQL 优化 IN/EXISTS 子查询的核心机制，独立成章。本篇综合内核月报《Semi-join优化与执行逻辑》(2021/06)、《semi-join四个执行strategy》(2020/07)、《Semijoin 丛林小道全览》(2024/06) 与 8.0.39 源码。

## 目录

- [一、Semi-join 要解决什么问题](#一semi-join-要解决什么问题)
- [二、设计思想与理论基础](#二设计思想与理论基础)
- [三、Notation：ot/ct/nt/it](#三notationotctntit)
- [四、Rewrite Phase：子查询拍平成 semi-join](#四rewrite-phase子查询拍平成-semi-join)
- [五、Optimize Phase：五种策略的代价选择](#五optimize-phase五种策略的代价选择)
- [六、Execution Phase：每种策略怎么执行](#六execution-phase每种策略怎么执行)
- [七、五种策略的 JOIN ORDER 矩阵](#七五种策略的-join-order-矩阵)
- [八、optimizer_switch 与 Hint](#八optimizer_switch-与-hint)
- [九、深潜：早停对照、NOT IN 灾难与 antijoin 演进](#九深潜早停对照not-in-灾难与-antijoin-演进)

---

## 一、Semi-join 要解决什么问题

`IN` / `EXISTS` 子查询是**存在性判断**：只关心"外表行是否在内表有匹配"，不关心匹配了几次。

- 按原始语义，每行外表都要单独执行子查询，相关子查询会被重复执行，效率很低。
- 而 SPJ 子查询的存在性判断**可以转换成外表与内表的 JOIN**。
- 但普通 JOIN 会让外表行因内表多行匹配而**重复膨胀**；`IN`/`EXISTS` 语义要求**只过滤、不膨胀**。

所以 semi-join 的核心矛盾是：**既要 JOIN（复用 join reordering 的灵活性），又要去重（不膨胀）**。五种策略就是五种"去重"方式。

```
Country.code IN (SELECT City.Country FROM City WHERE Population > 7e6)
```
Country 的一行 `code='CN'`，如果 City 里有多个城市 Population > 7e6，普通 JOIN 会输出多行 Country；semi-join 只输出一行。

---

## 二、设计思想与理论基础

### 为什么必须有五种策略？

**关键认知：semi-join 不是一个"算子"，而是"join order 上的一段区间 + 一个去重义务"。**

源码 `setup_semijoin_dups_elimination()` 的注释开篇就点明了这个模型：

```
The join order has "duplicate-generating ranges", and every range is
served by one strategy or a combination of FirstMatch with some other strategy.

"Duplicate-generating range" is defined as a range within the join order
that contains all of the inner tables of a semi-join.
```

区间的位置由 join reordering 决定，而去重能力又反过来约束 join order——正是这个**双向耦合**导致了"必须有多种策略"。

**最硬的源码证据**在 `advance_sj_state()` 里：

```
Use the strategy if
 * it is cheaper then what we've had, or
 * we haven't picked any other semi-join strategy yet
In the second case, we pick this strategy unconditionally because
comparing cost without semi-join duplicate removal with cost with
duplicate removal is not an apples-to-apples comparison.
```

> **"未去重的代价"和"已去重的代价"根本不同量纲，无法直接比较。**

所以优化器不能只挑"最便宜的算子"，而必须保证**每个前缀上至少有一个策略能让去重成立**——这就是五策略必须共存的形式化理由。整个 `advance_sj_state()` 的核心不变量是位图 `dups_producing_tables`：**"还没被任何策略消除重复的内表集合，结束时必须为 0"**。

### 分类学：3 类去重思路 × 4 个正交维度

**维度一（最本质）：去重发生在什么时刻**

| 类别 | 成员 | 机制 | 重复行是否真被生成过 |
|---|---|---|---|
| **① 提前退出（prevention）** | FirstMatch、LooseScan | 控制流层面"一组只取第一个" | 否（内表侧被短路/跳过） |
| **② 事后剔除（cure）** | DuplicateWeedout | 先按内连接跑出全部重复，再用 rowid 唯一键过滤 | **是** |
| **③ 预先做成无重复的块（normalize）** | Materialize×2 | 先物化内表并去重，之后退化成普通 join | 否（内表侧已去重） |

**维度二：依赖什么物理设施**

| 设施 | 策略 |
|---|---|
| 只靠执行控制流（零内存、零临时表） | FirstMatch |
| 必须依赖**索引** | LooseScan |
| 依赖**临时表** | DuplicateWeedout、MaterializeScan/Lookup |
| 物化表上还要能**建索引** | MaterializeLookup（Scan 不需要） |

Materialize 的两个子型就是按"用不用索引"拆出来的——源码明说 MaterializeLookup 不能用于 BLOB/GEOMETRY 列（"since indexes are not supported for BLOB columns"），而 Scan 可以。

**维度三：谁驱动（内表在前还是外表在前）**

- **外表驱动**：FirstMatch、MaterializeLookup
- **内表驱动**：LooseScan（要求"相关外表都还没进前缀"）
- **双向皆可**：MaterializeScan、DuplicateWeedout

**维度四（最精妙）：FirstMatch 与 LooseScan 的进入条件严格互补**

- FirstMatch 要求：相关外表**全部**已进前缀 —— `!(remaining_tables & outer_corr_tables)`
- LooseScan 要求：**还有**相关外表没进前缀 —— `remaining_tables_incl & sj_depends_on`

给定同一个 join 前缀，FM 与 LS 恰好覆盖两种**互斥**情形。**少了任何一个，就有一整类 join order 无法去重。**

### 各策略分别牺牲了什么

| 策略 | 收益 | 牺牲 |
|---|---|---|
| **FirstMatch** | 零临时表零内存，只改跳转指针 | 内表必须连续成块；多张内表时**禁用 join buffer**；不能跨 embedding nest 跳转 |
| **LooseScan** | 行数直接用 `rec_per_key` 塌缩（`rowcount /= rpc`），**不物化不建临时表** | 必须内表驱动（放弃所有"外表在前"的 join order）；强索引要求；IN 表达式 ≤ 64；**不能用于 antijoin** |
| **MaterializeLookup** | 内表彻底去重，后续 fanout 用 `distinct_rowcount` | 非平凡相关子查询**完全不可用**（`sj_corr_tables != 0` 直接放弃）；内表必须连续成块；类型/长度/BLOB 三道关 |
| **MaterializeScan** | 允许物化表当驱动表，外表顺序自由 | 同上，且要预付 `materialization_cost`（建表 + 写 `mat_rowcount` 行），**哪怕外表只输出 1 行也要付** |
| **DuplicateWeedout** | 对 join order **零约束** → 能把 semi-join 当普通 join 重排 | 重复行真的被生成、真的写过临时表（一次写 + 一次唯一键探查）；临时表可能落盘 |

### 理论溯源：为什么 MySQL 搞这么多种？

semi-join 在关系代数里是 R ⋉ S = π_R(R ⋈ S)，分布式数据库用"半连接缩减"减少传输量。但 **MySQL 的实现目标不同**：它要避免的是**行膨胀**（内表重复匹配导致外表行被放大）。

**与 PostgreSQL / Oracle 的关键差异，根源是历史包袱**：

MySQL 老优化器只有 nested loop 家族，五种策略全部是"在 nested loop 框架内去重"：

- **FirstMatch** = 循环短路
- **LooseScan** = 利用索引顺序（B-Tree 是 MySQL 唯一原生索引结构）
- **Weedout** = 临时表唯一键 —— **因为没有 hash 表算子，只能拿临时表当 hash set 用**
- **Materialize** = 临时表 + `<auto_distinct_key>` 唯一键，同样是"用临时表模拟 hash 去重"

对比：PostgreSQL 的 `Hash Semi Join` 用内存 hash 表，一侧 build 一侧 probe，**天然"每个 probe 行只输出一次"**——因为 hash join 是"整块"算子，**去重建在算子内部**，不需要为不同 join order 设计不同的去重机制。

MySQL 直到 8.0 才有 hash join，而在 hash join 里 semi-join 天然被支持：

```cpp
case JoinType::SEMI:
  // Semijoin should return the first matching row, and then go to the next
  // row from the probe input.
```

**这正是"五种策略"这个复杂度的历史由来**——它是一个"没有 hash join 的年代"留下的解决方案集合。

### 三级兜底

1. **策略内兜底**：DuplicateWeedout 无条件采纳（"not an apples-to-apples"）
2. **退化成内连接**：`pull_out_semijoin_tables()` 把被 eq_ref 唯一确定的内表抽出 nest；nest 被抽空则整个 semi-join 消失。hypergraph 源码直接写了这条等价律：*"If the inner side is known to be free of duplicates on the key ... semijoin is equivalent to inner join"*。注意副作用：**pullout 会把不相关子查询变成相关的，从而失去 Materialize / LooseScan 资格**
3. **转成 EXISTS**：表数超 `MAX_TABLES`(61)、子查询含 antijoin nest、或有 UNION/HAVING/聚合/LIMIT 等 → 退回 `IN→EXISTS`

### 参数

`optimizer_switch` 里 semijoin 相关的五个开关（`semijoin` / `firstmatch` / `loosescan` / `materialization` / `duplicateweedout`）**全部默认 on**。两个易错点：

- 控制 semi-join 物化的开关名是 **`materialization`**（不是 `semijoin_materialization`），它与"子查询物化"**共用同一个 flag**
- **antijoin（NOT IN / NOT EXISTS）会被强制砍掉 LooseScan**：

```cpp
if (sj_nest->is_aj_nest()) {
  // only these are possible with NOT EXISTS/IN:
  sj_enabled_strategies &= FIRSTMATCH | MATERIALIZATION | DUPSWEEDOUT;
}
```

---

## 三、Notation：ot/ct/nt/it

内核月报 2021/06 定义的记法，理解五种策略的前提：

| 记号 | 含义 |
|------|------|
| `ot` | outer table，外层表（含相关与不相关） |
| `ct` | **non-trivially correlated** outer table，子查询 WHERE 里依赖的外层表 |
| `nt` | non-correlated outer table，不相关的外层表 |
| `it` | inner table，内层（子查询）的表 |

`sj_depends_on = ot + ct`（所有被 semi-join 条件依赖的外层表），`sj_corr_tables = ct`（只含相关外层表）。这个区分在选择 join order 时至关重要。

---

## 四、Rewrite Phase：子查询拍平成 semi-join

### 3.1 resolve_subquery：收集候选（准入条件）

`sql/sql_resolver.cc:1368`。只有满足条件的 `IN/=ANY/EXISTS` 子查询才进 `sj_candidates`：

1. 子查询是 `IN`/`=ANY`/`EXISTS`，出现在 WHERE 或 ON 的**最高层**（可被 AND 包裹，OR 里不行）
2. 单一查询块，无 UNION
3. 无 HAVING、无聚合函数、无窗口函数
4. 无 LIMIT
5. 外层无 `STRAIGHT_JOIN`
6. LHS 确定（无 `RAND()`）

> 完整 16 条判定见 `resolve_subquery` 内注释（`sql_resolver.cc:1489-1567`）。不满足的走 `select_transformer`（IN2EXISTS / 物化 / MIN-MAX，见 02 篇）。

**源码：候选登记的那一段**（`sql/sql_resolver.cc:1527`）：

```cpp
// sql/sql_resolver.cc:1532 —— 唯一的"能否转 semijoin"判定点（没有 is_convertible_to_semijoin() 这个函数）
  DBUG_PRINT("info", ("Checking if subq can be converted to semi-join"));
  const bool no_aggregates = !is_grouped() && !with_sum_func &&
                             having_cond() == nullptr && !has_windows();
                             // ★ 3/3x/4/5 一次性打包：无 GROUP BY、无聚合、无 HAVING、无窗口函数

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
        (!thd->lex->using_hypergraph_optimizer() ||                // 6x
         (thd->secondary_engine_optimization() ==
              Secondary_engine_optimization::SECONDARY &&
          !Overlaps(engine_flags,
              MakeSecondaryEngineFlags(
                  SecondaryEngineFlag::SUPPORTS_NESTED_LOOP_JOIN)))))) &&
      outer->condition_context == enum_condition_context::ANDS &&  // 6b
      outer->sj_candidates &&                                      // 7
      leaf_table_count > 0 &&                                      // 8
      predicate->strategy == Subquery_strategy::UNSPECIFIED &&     // 9
      outer->leaf_table_count > 0 &&                               // 10
      !((active_options() | outer->active_options()) &             // 11
        SELECT_STRAIGHT_JOIN) &&                                   // 11
      !(outer->active_options() & SELECT_NO_SEMI_JOIN) &&          // 12
      deterministic &&                                             // 13
      predicate->choose_semijoin_or_antijoin() &&                  // 14
      (!cannot_do_antijoin || !predicate->can_do_aj) &&            // 15
      is_row_count_valid_for_semi_join()) {                        // 16
    DBUG_PRINT("info", ("Subquery is semi-join conversion candidate"));

    /* Notify in the subquery predicate where it belongs in the query graph */
    predicate->embedding_join_nest = outer->resolve_nest;

    /* Register the subquery for further processing in flatten_subqueries() */
    predicate->strategy = Subquery_strategy::CANDIDATE_FOR_SEMIJOIN;
    outer->sj_candidates->push_back(predicate);
    // ★ 这里只是"登记"，真正的转换发生在 flatten_subqueries()
    choice_made = true;
  }
  ...（subquery_to_derived 分支）

  if (!choice_made) {
    return subq_predicate->select_transformer(thd, this);
    // ★ 失败回退：IN→EXISTS / 物化 / MIN-MAX
  }
  return false;
```

**`predicate` 的取值是"能不能进 semijoin"的第一道闸**（`:1425`）：

```cpp
  // Predicate for possible semi-join candidates (IN and EXISTS)
  Item_exists_subselect *const predicate =
      subq_predicate->substype() == Item_subselect::EXISTS_SUBS ||
              subq_predicate->substype() == Item_subselect::IN_SUBS
          ? down_cast<Item_exists_subselect *>(subq_predicate)
          : nullptr;
  // ★ 标量子查询(SINGLEROW_SUBS) / ALL_SUBS / ANY_SUBS ⇒ predicate == nullptr ⇒ 永不进 semijoin
```

⇒ **勘误（常见误解）**：文档里常说的"`convert_subquery_to_semijoin` 处理四种 Item（`singlerow`/`exists`/`in`/`allany`）"**不成立**。真相是：`= ANY` 在**解析期**就被 `all_any_subquery_creator()`（`sql/sql_parse.cc:6703`）改写成 `Item_in_subselect`，`<> ALL` 被改写成 `NOT IN`（`truth_transformer(BOOL_NEGATED)`）；剩下的 `ALL/ANY` 变体 `substype()` 是 `ALL_SUBS`/`ANY_SUBS`，在第一道闸就被拒。所以 `convert_subquery_to_semijoin()` 的 `assert` 里只有 `IN_SUBS || EXISTS_SUBS` 两类。

```cpp
// sql/sql_parse.cc:6703 —— = ANY / <> ALL 在解析期就被改写
  if ((cmp == &comp_eq_creator) && !all)  //  = ANY <=> IN
    return new Item_in_subselect(left_expr, query_block);

  if ((cmp == &comp_ne_creator) && all)   // <> ALL <=> NOT IN
  {
    Item *i = new Item_in_subselect(left_expr, query_block);
    if (i == nullptr) return nullptr;
    Item *neg_i = i->truth_transformer(nullptr, Item::BOOL_NEGATED);
    if (neg_i != nullptr) return neg_i;
    return new Item_func_not(i);
  }
```

**16 条拒绝条件逐条对照**（含上面没展开的两条）：

| 编号 | 条件 | 含义 |
|---|---|---|
| 0 | `semijoin_enabled(thd)` | `optimizer_switch=semijoin=off` 或 `SEMIJOIN()` hint 关闭 |
| 1 | `predicate != nullptr` | 只接受 `IN_SUBS` / `EXISTS_SUBS` |
| 2 | `is_simple_query_block()` | **含 UNION / 括号包裹的 query expression 直接拒** |
| 3,3x,4,5 | `no_aggregates` | GROUP BY / 聚合函数 / HAVING / 窗口函数 |
| 6a | `resolve_place` | 必须在 WHERE 或 ON（不能在 SELECT list / ORDER BY） |
| **6x** | hypergraph 判定 | **坑**：hypergraph 优化器下 **ON 子句中的 semijoin 被禁**（会破坏 join tree），除非 secondary engine |
| 6b | `condition_context == ANDS` | 必须是 AND 顶层；`OR` / `NOT` 包裹下拒绝 |
| 7 | `outer->sj_candidates` | 单表 UPDATE/DELETE 的子查询没有 sj_candidates |
| 8 | `leaf_table_count > 0` | "SELECT 1" 这类无表子查询 |
| 9 | `strategy == UNSPECIFIED` | PS 二次执行已选定执行方式 |
| 10 | `outer->leaf_table_count > 0` | 父查询无表 |
| 11 | `!(... & SELECT_STRAIGHT_JOIN)` | **子或父任一有 STRAIGHT_JOIN 即拒** |
| 12 | `!(outer->... & SELECT_NO_SEMI_JOIN)` | `NO_SEMIJOIN` hint |
| 13 | `deterministic` | IN 左侧含 `RAND()` 等非确定性 |
| 14 | `choose_semijoin_or_antijoin()` | 真值测试与 NULL-able 兼容性（决定 semijoin 还是 antijoin） |
| 15 | `!cannot_do_antijoin \|\| !can_do_aj` | secondary engine 不支持 antijoin；或非相关 `NOT EXISTS` 走"优化期常量求值"更划算 |
| 16 | `is_row_count_valid_for_semi_join()` | 见下 |

**第 16 条：LIMIT / OFFSET 判定**（`sql/sql_resolver.cc:1336`）：

```cpp
bool Query_block::is_row_count_valid_for_semi_join() {
  if (offset_limit != nullptr &&
      (!offset_limit->const_item() || offset_limit->val_int() != 0))
    return false;   // ★ OFFSET 非 0（或非 const）→ 拒绝：semijoin 会丢行/多行

  if (select_limit != nullptr &&
      (!select_limit->const_item() || select_limit->val_int() == 0))
    return false;   // ★ LIMIT 0 → 子查询恒空，转了没意义

  return true;
}
```

**策略枚举**（`sql/item_subselect.h:398`）——`CANDIDATE_FOR_SEMIJOIN → SEMIJOIN` 的迁移就发生在下一节：

```cpp
enum class Subquery_strategy : int {
  UNSPECIFIED,
  SUBQ_EXISTS,
  SUBQ_MATERIALIZATION,
  CANDIDATE_FOR_IN2EXISTS_OR_MAT,
  /// Candidate for semi-join flattening
  CANDIDATE_FOR_SEMIJOIN,
  SEMIJOIN,
  CANDIDATE_FOR_DERIVED_TABLE,
  DERIVED_TABLE,
  DELETED
};
```

### 3.2 flatten_subqueries：bottom-up + 优先级

`sql/sql_resolver.cc:3811`。**自底向上**拍平（内层子查询先拍平，外层拍平时可能把内层一起合并）。

优先级（`sj_convert_priority`，`:3867`）：

```cpp
sj_convert_priority = ((dependent * MAX_TABLES_FOR_SIZE) + leaf_table_count) * 65536 + (65536 - subq_no);
```

从高到低三层：**相关子查询 > 内层表多 > 出现早**。因为表数上限 `MAX_TABLES`，重要的先拍平。

**完整函数**（`sql/sql_resolver.cc:3811`）：

```cpp
bool Query_block::flatten_subqueries(THD *thd) {
  DBUG_TRACE;
  assert(has_sj_candidates());

  Item_exists_subselect **subq, **subq_begin = sj_candidates->begin(),
                                **subq_end = sj_candidates->end();
  Opt_trace_context *const trace = &thd->opt_trace;

  /*
    Semijoin flattening is bottom-up. Indeed, we have this execution flow,
    note: for SELECT#1 WHERE X IN (SELECT #2 WHERE Y IN (SELECT#3)) :

    Query_block::prepare() (select#1)
       -> fix_fields() on IN condition
           -> Query_block::prepare() on subquery (select#2)
               -> fix_fields() on IN condition
                    -> Query_block::prepare() on subquery (select#3)
                    -> flatten_subqueries: merge #3 in #2
               <- Query_block::prepare()
       <- fix_fields()
       -> flatten_subqueries: merge #2 in #1

    Note that flattening of #(N) is done by its parent JOIN#(N-1), because
    there are cases where flattening is not possible and only the parent can
    know.
   */
  uint subq_no;
  for (subq = subq_begin, subq_no = 0; subq < subq_end; subq++, subq_no++) {
    auto subq_item = *subq;
    /*
      Some subqueries may have been deleted, remove them fully before sorting
      sj_candidates and subsequent processing:
    */
    if (subq_item->strategy == Subquery_strategy::DELETED) {
      sj_candidates->erase_value(subq_item);
      subq--;  // So that the next iteration will handle the next subquery.
      subq_end = sj_candidates->end();  // array's end moved.
      continue;
    }
    // Transformation of IN and EXISTS subqueries is supported
    assert(subq_item->substype() == Item_subselect::IN_SUBS ||
           subq_item->substype() == Item_subselect::EXISTS_SUBS);

    Query_block *child_query_block = subq_item->unit->first_query_block();

    // Check that we proceeded bottom-up
    assert(child_query_block->sj_candidates == nullptr);

    bool dependent = subq_item->unit->uncacheable & UNCACHEABLE_DEPENDENT;
    subq_item->sj_convert_priority =
        (((dependent * MAX_TABLES_FOR_SIZE) +  // dependent subqueries first
          child_query_block->leaf_table_count) *
         65536) +           // then with many tables
        (65536 - subq_no);  // then based on position
                            // ★ 优先级编码：高 16 位 = 相关*MAX_TABLES + 表数；低 16 位 = 65536-序号
    /* ... */
  }

  /*
    Pick which subqueries to convert:
      sort the subquery array
      - prefer correlated subqueries over uncorrelated;
      - prefer subqueries that have greater number of outer tables;
  */
  std::sort(subq_begin, subq_begin + sj_candidates->size(),
            [](Item_exists_subselect *el1, Item_exists_subselect *el2) {
              return el1->sj_convert_priority > el2->sj_convert_priority;
            });
                            // ★ 降序 ⇒ 相关 > 表多 > 早出现

  // A permanent transformation is going to start, so:
  Prepared_stmt_arena_holder ps_arena_holder(thd);

  // Transform certain subquery predicates to derived tables
  for (subq = subq_begin; subq < subq_end; subq++) {
    auto subq_item = *subq;
    if (subq_item->strategy != Subquery_strategy::CANDIDATE_FOR_DERIVED_TABLE)
      continue;
    OPT_TRACE_TRANSFORM(trace, oto0, oto1,
                        subq_item->unit->first_query_block()->select_number,
                        "IN (SELECT)", "joined derived table");
    oto1.add("chosen", true);
    if (transform_table_subquery_to_join_with_derived(thd, subq_item))
      return true;
  }

  /*
    Replace all subqueries to be flattened with a truth predicate.
    Generally, this predicate is TRUE, but if the subquery has a WHERE condition
    that is always false, replace with a FALSE predicate. In the latter case,
    also avoid converting the subquery to a semi-join.
  */
  uint table_count = leaf_table_count;
  for (subq = subq_begin; subq < subq_end; subq++) {
    auto subq_item = *subq;
    if (subq_item->strategy != Subquery_strategy::CANDIDATE_FOR_SEMIJOIN)
      continue;

    // Add the tables in the subquery nest plus one in case of materialization:
    const uint tables_added =
        subq_item->unit->first_query_block()->leaf_table_count + 1;
                            // ★ +1：为 semijoin materialization 预留一个临时表槽位

    // (1) Not too many tables in total.
    // (2) This subquery contains no antijoin nest (anti/semijoin nest cannot
    // include antijoin nest for implementation reasons, see advance_sj_state()).
    if (table_count + tables_added <= MAX_TABLES &&           // (1)
        !subq_item->unit->first_query_block()->has_aj_nests)  // (2)
      subq_item->strategy = Subquery_strategy::SEMIJOIN;
                            // ★★ 真正的"能否转"二次判定：表数超限 / 子块内嵌 antijoin → 放弃

    Item *subq_where = subq_item->unit->first_query_block()->where_cond();
    /*
      A predicate can be evaluated to ALWAYS TRUE or ALWAYS FALSE when it
      has only const items. If found to be ALWAYS FALSE, do not include
      the subquery in transformations.
    */
    bool cond_value = true;
    if (subq_where && subq_where->const_item() &&
        !subq_where->walk(&Item::is_non_const_over_literals, enum_walk::POSTFIX,
                          nullptr) &&
        simplify_const_condition(thd, &subq_where, false, &cond_value))
      return true;

    if (!cond_value) {
      // Unlink and delete this subquery's query expression
      Item::Cleanup_after_removal_context ctx(this);
      subq_item->walk(&Item::clean_up_after_removal, walk_options,
                      pointer_cast<uchar *>(&ctx));
    }

    if (subq_item->strategy == Subquery_strategy::SEMIJOIN)
      table_count += tables_added;

    if (subq_item->strategy != Subquery_strategy::SEMIJOIN &&
        subq_item->strategy != Subquery_strategy::DELETED) {
      subq_item->strategy = Subquery_strategy::UNSPECIFIED;
      continue;   // ★ 回退：留给后面 select_transformer() 做 IN→EXISTS
    }
    /*
      In WHERE/ON of parent query, replace IN (subq) with truth value:
      - When subquery is converted to anti/semi-join: truth value true.
      - When subquery WHERE cond is false: IN returns FALSE, so truth value
      false if a semijoin (IN) and truth value true if an antijoin (NOT IN).
    */
    Item *truth_item =
        (cond_value || subq_item->can_do_aj)
            ? implicit_cast<Item *>(new (thd->mem_root) Item_func_true())
            : implicit_cast<Item *>(new (thd->mem_root) Item_func_false());
    if (truth_item == nullptr) return true;
    Item **tree = (subq_item->embedding_join_nest == nullptr)
                      ? &m_where_cond
                      : subq_item->embedding_join_nest->join_cond_ref();
    if (replace_subcondition(thd, tree, subq_item, truth_item, false))
      return true;
  }

  /* Transform the selected subqueries into semi-join */
  for (subq = subq_begin; subq < subq_end; subq++) {
    auto subq_item = *subq;
    if (subq_item->strategy != Subquery_strategy::SEMIJOIN) continue;
    OPT_TRACE_TRANSFORM(
        trace, oto0, oto1, subq_item->unit->first_query_block()->select_number,
        "IN (SELECT)", subq_item->can_do_aj ? "antijoin" : "semijoin");
    oto1.add("chosen", true);
    if (convert_subquery_to_semijoin(thd, *subq)) return true;
  }

  /*
    Finalize the subqueries that we did not convert,
    ie. perform IN->EXISTS rewrite.
  */
  for (subq = subq_begin; subq < subq_end; subq++) {
    auto subq_item = *subq;
    if (subq_item->strategy != Subquery_strategy::UNSPECIFIED) continue;
    subq_item->changed = false;
    subq_item->fixed = false;

    Query_block *save_query_block = thd->lex->current_query_block();
    thd->lex->set_current_query_block(subq_item->unit->first_query_block());

    // This is the only part of the function which uses a JOIN.
    if (subq_item->select_transformer(thd,
                                      subq_item->unit->first_query_block()))
      return true;

    thd->lex->set_current_query_block(save_query_block);

    subq_item->changed = true;
    subq_item->fixed = true;

    /*
      If the Item has been substituted with another Item (e.g an
      Item_in_optimizer), resolve it and add it to proper WHERE or ON clause.
      ...
    */
    Item *substitute = subq_item->substitution;
    if (substitute == nullptr) continue;
    const bool do_fix_fields = !substitute->fixed;
    const bool subquery_in_join_clause =
        subq_item->embedding_join_nest != nullptr;

    Item **tree = subquery_in_join_clause
                      ? (subq_item->embedding_join_nest->join_cond_ref())
                      : &m_where_cond;
    if (replace_subcondition(thd, tree, *subq, substitute, do_fix_fields))
      return true;
    subq_item->substitution = nullptr;
  }

  sj_candidates->clear();
  return false;
}
```

⇒ **坑**：`table_count + tables_added <= MAX_TABLES` 是**按优先级贪心**的——排序靠后的子查询会被**静默降级**成 IN→EXISTS，EXPLAIN 里表现为 `SUBQUERY`/`DEPENDENT SUBQUERY` 而非 semijoin。这也是"同一 SQL 多加几个表就突然变慢"的常见原因之一。

### 3.3 convert_subquery_to_semijoin：四种模式 + 完整流程（逐行剖析）

`sql/sql_resolver.cc:2992`。把内层表"拉进"外层 join tree。**四种模式**由两个维度决定：子查询类型（`IN` vs `EXISTS`）× 是否否定（`do_aj`）：

| 模式 | substype | `can_do_aj` | 建 nest 类型 | 表达式处理 |
|------|----------|-------------|-------------|-----------|
| **IN**（positive） | `IN_SUBS` | false | sj-nest | `build_sj_exprs` 配对 `oe[i]↔ie[i]` |
| **EXISTS**（positive） | `EXISTS_SUBS` | false | sj-nest | SELECT list 不用（`:3273-3282`） |
| **NOT IN**（negative） | `IN_SUBS` | true | aj-nest | 同 IN，但 `LEFT JOIN + IS NULL` |
| **NOT EXISTS**（negative） | `EXISTS_SUBS` | true | aj-nest | 同 EXISTS，但 `LEFT JOIN + IS NULL` |

```cpp
// sql_resolver.cc:2992
bool Query_block::convert_subquery_to_semijoin(THD *thd,
                                               Item_exists_subselect *subq_pred) {
  assert(subq_pred->substype() == Item_subselect::IN_SUBS ||
         subq_pred->substype() == Item_subselect::EXISTS_SUBS);
  table_map outer_tables_map = all_tables_map();          // ① 记录外层表位图
  const bool do_aj = subq_pred->can_do_aj;                // ② anti-join？(NOT IN/NOT EXISTS)

  // ③ 找插入点：三种情形（见下表）
  if (subq_pred->embedding_join_nest != nullptr) {
    outer_join = ...is_inner_table_of_outer_join();
    if (nest->nested_join) { emb_tbl_nest = nest; ... }             // (a) 括号 nest 内
    else if (!nest->outer_join) { emb_tbl_nest = nest->embedding; } // (b) INNER JOIN 兄弟
    else { /* (c) LEFT JOIN 内：用 wrap_nest 包裹 */ }
  }
  // else 子查询在 WHERE（emb_tbl_nest == nullptr）

  // ④ anti-join 时，先建 aj-left-nest（把 emb_join_list 所有表挪进去）
  if (do_aj) {
    wrap_nest = new_nested_join("(aj-left-nest)", ...);
    for (outer_tbl : *emb_join_list) wrap_nest->m_tables.push_back(outer_tbl); // 清空 emb_join_list
    emb_join_list->clear(); emb_join_list->push_back(wrap_nest);
    outer_join = true;
  }

  // ⑤ 建 sj-nest / aj-nest
  Table_ref *sj_nest = new_nested_join(do_aj ? "(aj-nest)" : "(sj-nest)", ...);
  emb_join_list->push_front(sj_nest);                    // push_front：aj 是 LEFT JOIN 右参数
  sj_nest->merge_underlying_tables(subq_query_block);    // ⑥ 把子查询表合并进 nest

  // ⑦ 配对表达式 / 去相关化
  if (substype == IN_SUBS) {
    build_sj_exprs(... sj_outer_exprs, sj_inner_exprs ...);   // IN：oe↔ie 配对
  } else { /* EXISTS：SELECT list 清理掉 */ }
  decorrelate_condition(sj_decor, nullptr);              // ⑧ 内层 WHERE 等值谓词提升
  build_sj_cond(thd, nested_join, ..., &sj_cond);        // ⑨ 逐对 AND 成 oe=ie

  // ⑩ anti-join 收尾：sj-nest 变 LEFT JOIN 右表，ON = sj_cond
  if (do_aj) {
    sj_nest->outer_join = true;
    sj_nest->set_join_cond(sj_cond);                     // x IS NULL 留给优化阶段再加
    this->outer_join |= sj_nest->nested_join->used_tables;
  }
}
```

**③ 三种插入位置**（`:3020-3114`）：

| 位置 | 处理 | emb_join_list 指向 |
|------|------|-------------------|
| WHERE 里 | sj-nest 直接进 FROM | 顶层 `m_table_nest` |
| `INNER JOIN tblX ON (subquery AND cond)` | sj-nest 成为 tblX 兄弟 | tblX 的 embedding 的 join list |
| `LEFT JOIN tbl ON (subquery AND cond)` | `wrap_nest` 包住 `tbl SJ (subq)`，outer join 标志上移 | wrap_nest 的 m_tables |

LEFT JOIN 内层最麻烦（`:3046-3113`）：直接插表会破坏 outer join 的可空语义，必须用 `wrap_nest` 把 `tbl SJ (subq_tables)` 整体包住，并把 `outer_join` 标志和 join cond 从 `tbl` 上移到 `wrap_nest`（`:3083-3087`）。

**④ anti-join 的 aj-left-nest**（`:3165-3179`）：anti-join 建模成 `(左表) LEFT JOIN (aj-nest) ON subq_cond`，所以先把 emb_join_list 里所有现有表挪进一个 `(aj-left-nest)`，清空原 join list，只留 aj-left-nest，再 `push_front` aj-nest 作为 LEFT JOIN 右参数。**`x IS NULL` 条件不在这一步加**——注释（`:3161-3163`）明确说留到优化阶段，因为那时才有 QEP_TAB 能设 trig cond 的 `found`/`not_null_compl` 指针。

**关键：`push_front` 而不是 `push_back`**（`:3197`）——anti-join 时 sj-nest 是 LEFT JOIN 的右参数，而 LEFT JOIN 的右参数必须排在左参数之前（join tree 里右表先）。

**⑩ anti-join 收尾**：`set_join_cond(sj_cond)` 把 semi-join 条件挂成 LEFT JOIN 的 ON，配合优化阶段的 `x IS NULL` 完成 `NOT IN/NOT EXISTS` 语义。

**完整源码**（`sql/sql_resolver.cc:2992-3447`，按执行顺序贴出关键段）：

```cpp
bool Query_block::convert_subquery_to_semijoin(
    THD *thd, Item_exists_subselect *subq_pred) {
  Table_ref *emb_tbl_nest = nullptr;
  mem_root_deque<Table_ref *> *emb_join_list = &m_table_nest;
  DBUG_TRACE;

  assert(subq_pred->substype() == Item_subselect::IN_SUBS ||
         subq_pred->substype() == Item_subselect::EXISTS_SUBS);
  // ★ 只有两类，没有 singlerow / allany 分支

  bool outer_join = false;  // True if predicate is inner to an outer join

  // Save the set of tables in the outer query block:
  table_map outer_tables_map = all_tables_map();   // ①
  const bool do_aj = subq_pred->can_do_aj;         // ② anti-join？

  /*
    Find out where to insert the semi-join nest and the generated condition.
    For t1 LEFT JOIN t2, embedding_join_nest will be t2.
  */
  if (subq_pred->embedding_join_nest != nullptr) {              // ③
    outer_join = subq_pred->embedding_join_nest->is_inner_table_of_outer_join();

    if (subq_pred->embedding_join_nest->nested_join) {
      /* (a) ... [LEFT] JOIN ( ... ) ON (subquery AND condition) ... */
      emb_tbl_nest = subq_pred->embedding_join_nest;
      emb_join_list = &emb_tbl_nest->nested_join->m_tables;
    } else if (!subq_pred->embedding_join_nest->outer_join) {
      /* (b) ... INNER JOIN tblX ON (subquery AND condition) ...
         The sj-nest will be tblX's "sibling" ... */
      emb_tbl_nest = subq_pred->embedding_join_nest->embedding;
      if (emb_tbl_nest) emb_join_list = &emb_tbl_nest->nested_join->m_tables;
    } else {
      /* (c) ... LEFT JOIN tbl ON (on_expr AND subq_pred) ...
         tbl will be replaced with ( tbl SJ (subq_tables) ) */
      Table_ref *outer_tbl = subq_pred->embedding_join_nest;
      Table_ref *const wrap_nest = Table_ref::new_nested_join(
          thd->mem_root, "(sj-wrap)", outer_tbl->embedding,
          outer_tbl->join_list, this);
      if (wrap_nest == nullptr) return true;

      wrap_nest->nested_join->m_tables.push_back(outer_tbl);
      outer_tbl->embedding = wrap_nest;
      outer_tbl->join_list = &wrap_nest->nested_join->m_tables;

      /*
        wrap_nest will take place of outer_tbl, so move the outer join flag
        and join condition.
      */
      wrap_nest->outer_join = outer_tbl->outer_join;
      outer_tbl->outer_join = false;
      wrap_nest->set_join_cond(outer_tbl->join_cond());
      outer_tbl->set_join_cond(nullptr);

      for (auto li = wrap_nest->join_list->begin();
           li != wrap_nest->join_list->end(); ++li) {
        if (*li == outer_tbl) { *li = wrap_nest; break; }
      }

      /*
        outer_tbl is replaced by wrap_nest. Any subquery which was attached to
        outer_tbl must be attached to embedding_join_nest instead.
      */
      for (Item_exists_subselect *subquery : (*sj_candidates)) {
        if (subquery->embedding_join_nest == outer_tbl)
          subquery->embedding_join_nest = wrap_nest;
      }

      emb_join_list = &wrap_nest->nested_join->m_tables;
      emb_tbl_nest = wrap_nest;
    }
  }
  // else subquery is in WHERE.

  if (do_aj) {                                                   // ④
    /* A negated IN/EXISTS ... 三种情形 (a)(b)(c) 见注释：
       - move all tables of emb_join_list into a new aj-left-nest
       - emb_join_list is now empty
       - put subq_tables in a new aj-nest
       - add the subq's subq_cond to aj-nest's ON
       - add a LEFT JOIN operator between the aj-left-nest and aj-nest
       - do not yet add 'x IS NULL to WHERE' (add it in optimization phase
         when we have the QEP_TABs so we can set up the 'found'/
         'not_null_compl' pointers in trig conds). */
    Table_ref *const wrap_nest = Table_ref::new_nested_join(
        thd->mem_root, "(aj-left-nest)", emb_tbl_nest, emb_join_list, this);
    if (wrap_nest == nullptr) return true;

    // Go through tables of emb_join_list, insert them in wrap_nest
    for (Table_ref *outer_tbl : *emb_join_list) {
      wrap_nest->nested_join->m_tables.push_back(outer_tbl);
      outer_tbl->embedding = wrap_nest;
      outer_tbl->join_list = &wrap_nest->nested_join->m_tables;
    }
    // FROM clause is now only the new left nest
    emb_join_list->clear();
    emb_join_list->push_back(wrap_nest);
    outer_join = true;   // ★ antijoin 本质上就是 LEFT JOIN + IS NULL
  }

  Table_ref *const sj_nest = Table_ref::new_nested_join(          // ⑤
      thd->mem_root, do_aj ? "(aj-nest)" : "(sj-nest)", emb_tbl_nest,
      emb_join_list, this);
  if (sj_nest == nullptr) return true;
  NESTED_JOIN *const nested_join = sj_nest->nested_join;

  /*
    Using push_front, as sj_nest may be right arg of LEFT JOIN if
    antijoin, and right args of LEFT JOIN go before left arg.
  */
  emb_join_list->push_front(sj_nest);   // ★ push_front 的原因

  /*
    Natural joins inside a semi-join nest were already processed when the
    subquery went through initial preparation.
  */
  sj_nest->nested_join->natural_join_processed = true;

  Query_block *const subq_query_block = subq_pred->unit->first_query_block();
  nested_join->query_block_id = subq_query_block->select_number;

  // Merge tables from underlying query block into this join nest
  if (sj_nest->merge_underlying_tables(subq_query_block))         // ⑥
    return true;

  /* Add tables from subquery at end of leaf table chain ... */
  Table_ref *tl;
  for (tl = leaf_tables; tl->next_leaf; tl = tl->next_leaf) {
  }
  tl->next_leaf = subq_query_block->leaf_tables;
  m_table_list.push_back(&subq_query_block->m_table_list);

  // Walk through child's tables and adjust table map
  uint table_no = leaf_table_count;
  for (tl = subq_query_block->leaf_tables; tl; tl = tl->next_leaf, table_no++) {
    tl->dep_tables <<= leaf_table_count;
    tl->set_tableno(table_no);
  }

  subq_query_block->m_table_list.clear();
  subq_query_block->leaf_tables = nullptr;

  // Adjust table and expression counts in parent query block:
  derived_table_count += subq_query_block->derived_table_count;
  materialized_derived_table_count +=
      subq_query_block->materialized_derived_table_count;
  table_func_count += subq_query_block->table_func_count;
  has_sj_nests |= subq_query_block->has_sj_nests;
  has_aj_nests |= subq_query_block->has_aj_nests;
  leaf_table_count += subq_query_block->leaf_table_count;
  cond_count += subq_query_block->cond_count;
  ...

  if (outer_join) propagate_nullability(&sj_nest->nested_join->m_tables, true);

  nested_join->sj_outer_exprs.clear();
  nested_join->sj_inner_exprs.clear();

  if (subq_pred->substype() == Item_subselect::IN_SUBS) {         // ⑦
    // IN / =ANY：配对 oe[i] ↔ ie[i]
    build_sj_exprs(thd, &nested_join->sj_outer_exprs,
                   &nested_join->sj_inner_exprs, subq_pred, subq_query_block);
  } else {  // this is EXISTS
    /* Expressions from the SELECT list will not be used; unlike in the case of
       IN, they are not part of sj_inner_exprs. */
    for (Item *item : subq_query_block->visible_fields()) {
      Item::Cleanup_after_removal_context ctx(this);
      item->walk(&Item::clean_up_after_removal, walk_options,
                 pointer_cast<uchar *>(&ctx));
    }
  }

  {
    /*
      The WHERE clause and the join conditions may contain equalities that may
      be leveraged by semi-join strategies (e.g to set up key lookups in
      semi-join materialization), decorrelate them ...
    */
    Semijoin_decorrelation sj_decor(&sj_nest->nested_join->sj_outer_exprs,
                                    &sj_nest->nested_join->sj_inner_exprs,
                                    // decorrelate only equalities
                                    /*op_types=*/nullptr);
    // ★ op_types==nullptr ⇒ 只去相关化等值(EQ)；非等值去相关化留给 hypergraph

    if (subq_query_block->where_cond() &&
        subq_query_block->decorrelate_condition(sj_decor, nullptr))
      return true;

    if (walk_join_list(
            subq_query_block->m_table_nest, [&](Table_ref *tr) -> bool {
              return !tr->is_inner_table_of_outer_join() && tr->join_cond() &&
                     subq_query_block->decorrelate_condition(sj_decor, tr);
            }))
      return true;
  }

  // Unlink the subquery's query expression:
  subq_query_block->master_query_expression()->exclude_level();
  merge_contexts(subq_query_block);
  repoint_contexts_of_join_nests(subq_query_block->m_table_nest);
  fix_tables_after_pullout(this, subq_query_block, sj_nest, 0, 0);

  Item *sj_cond = subq_query_block->where_cond();
  if (sj_cond != nullptr) sj_cond->fix_after_pullout(this, subq_query_block);

  // Assign the set of non-trivially tables after decorrelation
  nested_join->sj_corr_tables =
      (sj_cond != nullptr ? sj_cond->used_tables() & outer_tables_map : 0);
  // ★ sj_corr_tables = 去相关化之后"仍然相关"的外层表
  //   ≠ 0 ⇒ 无法使用 Materialization / LooseScan

  walk_join_list(subq_query_block->m_table_nest, [&](Table_ref *tr) -> bool {
    if (tr->join_cond())
      nested_join->sj_corr_tables |=
          tr->join_cond()->used_tables() & outer_tables_map;
    if (tr->is_derived() && tr->uses_materialization())
      nested_join->sj_corr_tables |=
          tr->derived_query_expression()->m_lateral_deps;
    return false;
  });

  // Build semijoin condition using the inner/outer expression list
  if (build_sj_cond(thd, nested_join, subq_query_block, outer_tables_map,
                    &sj_cond))                                    // ⑨
    return true;

  assert(sj_cond != nullptr);

  if (!sj_cond->fixed) {
    Opt_trace_array sj_on_trace(&thd->opt_trace,
                                "evaluating_constant_semijoin_conditions");
    sj_cond->apply_is_true();
    if (sj_cond->fix_fields(thd, &sj_cond)) return true;
  }

  sj_nest->set_sj_or_aj_nest();
  assert(sj_nest->join_cond() == nullptr);

  if (do_aj) {                                                    // ⑩
    sj_nest->outer_join = true;
    sj_nest->set_join_cond(sj_cond);
    this->outer_join |= sj_nest->nested_join->used_tables;
    if (emb_tbl_nest == nullptr)
      nest_last_join(thd);  // as is done for a true LEFT JOIN
  }

  /*
    sj_depends_on contains the set of outer tables referred in the
    subquery's WHERE clause as well as tables referred in the IN predicate's
    left-hand side, and lateral dependencies from materialized derived tables
    contained in the original subquery.
  */
  nested_join->sj_depends_on =
      nested_join->sj_corr_tables | (sj_cond->used_tables() & outer_tables_map);
  // ★ sj_depends_on ⊇ sj_corr_tables：前者多了 outer_expr 引用的表

  if (do_aj) { /* Condition remains attached to inner table, as for LEFT JOIN */
  } else if (emb_tbl_nest) {
    // Inject semi-join condition into parent's join condition
    emb_tbl_nest->set_join_cond(and_items(emb_tbl_nest->join_cond(), sj_cond));
    if (emb_tbl_nest->join_cond() == nullptr) return true;
    emb_tbl_nest->join_cond()->apply_is_true();
    if (!emb_tbl_nest->join_cond()->fixed &&
        emb_tbl_nest->join_cond()->fix_fields(thd,
                                              emb_tbl_nest->join_cond_ref()))
      return true;
  } else {
    // Inject semi-join condition into parent's WHERE condition
    m_where_cond = and_items(m_where_cond, sj_cond);
    if (m_where_cond == nullptr) return true;
    m_where_cond->apply_is_true();
    if (m_where_cond->fix_fields(thd, &m_where_cond)) return true;
  }

  Item *cond = emb_tbl_nest ? emb_tbl_nest->join_cond() : m_where_cond;
  if (cond && cond->const_item() &&
      !cond->walk(&Item::is_non_const_over_literals, enum_walk::POSTFIX,
                  nullptr)) {
    bool cond_value = true;
    if (simplify_const_condition(thd, &cond, false, &cond_value)) return true;
    if (!cond_value) {
      /*
        Parent's condition is always FALSE. Thus: ...(c) the outer/inner sj
        expressions are Items which point into the SJ condition, so at 2nd
        execution they won't be fixed => clearing them prevents a bug.
      */
      clear_sj_expressions(nested_join);   // ★ PS 二次执行的保护
    }
  }

  if (do_aj)
    has_aj_nests = true;
  else
    has_sj_nests = true;  // This query block has semi-join nests

  return false;
}
```

⇒ 两个容易看漏的语义：**(1) `sj_corr_tables` 与 `sj_depends_on` 不等价**——前者是"去相关化后仍相关"，后者再并上 `sj_cond` 引用的外层表；策略准入用的是不同成员（LooseScan 用 `sj_corr_tables`、FirstMatch/Materialize 用 `sj_depends_on`）。**(2) 条件注入分三处**（aj 挂 ON / 有 emb_tbl_nest 挂 join cond / 否则挂 WHERE），antijoin 的 `x IS NULL` 故意留到优化阶段才加。

### 3.4 五种 SQL 等价变换形式（内核月报 2020/10 原文）

`convert_subquery_to_semijoin` 的注释列了完整变换：

```
1. IN/=ANY:
   FROM ot1...otN WHERE (oe1..oeM) IN (SELECT ie1..ieM FROM it1..itK [WHERE inner-cond])
   => FROM (ot1..otN) SJ (it1..itK) ON (oe1..oeM)=(ie1..ieM) [AND inner-cond]

2. EXISTS:
   WHERE EXISTS(SELECT ... FROM it1..itK [WHERE inner-cond])
   => FROM (ot) SJ (it) [ON inner-cond]

3. NOT EXISTS:
   => FROM (ot) AJ (it) [ON inner-cond] WHERE ... AND is-null-cond(it1)

4. NOT IN:
   => FROM (ot) AJ (it) ON (oe)=(ie) [AND inner-cond]
```

anti-join（AJ）建模成 **LEFT JOIN + 右表键 IS NULL**。

### 3.5 去相关化 + 建 sj 条件

- `build_sj_exprs`（`:2833`）：IN 的左右两边配对成 `sj_outer_exprs[i] ↔ sj_inner_exprs[i]`
- `decorrelate_condition`（`:3284`）：把内层 WHERE 里的 `it.c = ot.c` 提升成 sj 连接条件
- `build_sj_cond`（`:2481`）：逐对 AND 成 `oe[i]=ie[i]`，常量对折叠
- `simplify_joins`：去掉嵌套 sj-nest（`A SJ (B SJ (C))` → `A SJ (B JOIN C)`）

**`build_sj_exprs()`**（`sql/sql_resolver.cc:2833`）——IN 的左右两边配对：

```cpp
static bool build_sj_exprs(THD *thd, mem_root_deque<Item *> *sj_outer_exprs,
                           mem_root_deque<Item *> *sj_inner_exprs,
                           Item_exists_subselect *subq_pred,
                           Query_block *subq_query_block) {
  Item_in_subselect *in_subq_pred = down_cast<Item_in_subselect *>(subq_pred);
  assert(in_subq_pred->left_expr->fixed);

  /*
    We have a special case for IN predicates with a scalar subquery or a
    row subquery in the predicand (left operand), such as this:
     (SELECT 1,2 FROM t1) IN (SELECT x,y FROM t2)
    We cannot make the join condition 1=x AND 2=y, since that might evaluate
    to true even if t1 is empty. Instead make the join condition
    (SELECT 1,2 FROM t1) = (x,y) in this case.
  */
  // ★ 左侧是子查询时不能逐列拆：空集的语义会错
  Item_subselect *left_subquery =
      (in_subq_pred->left_expr->type() == Item::SUBSELECT_ITEM)
          ? static_cast<Item_subselect *>(in_subq_pred->left_expr)
          : nullptr;

  if (left_subquery &&
      (left_subquery->substype() == Item_subselect::SINGLEROW_SUBS)) {
    mem_root_deque<Item *> ref_list(thd->mem_root);
    Item *header = subq_query_block->base_ref_items[0];
    for (uint i = 1; i < in_subq_pred->left_expr->cols(); i++) {
      ref_list.push_back(subq_query_block->base_ref_items[i]);
    }
    Item_row *right_expr = new Item_row(header, ref_list);
    if (!right_expr) return true;

    sj_outer_exprs->push_back(in_subq_pred->left_expr);
    sj_inner_exprs->push_back(right_expr);   // ★ 整行比较
  } else {
    for (uint i = 0; i < in_subq_pred->left_expr->cols(); i++) {
      Item *const li = in_subq_pred->left_expr->element_index(i);
      sj_outer_exprs->push_back(li);
      sj_inner_exprs->push_back(subq_query_block->base_ref_items[i]);
      // ★ 逐列配对：oe_i ↔ ie_i，下标严格对齐
    }
  }
  return false;
}
```

**`build_sj_cond()`**（`:2481`）——把配对 AND 成 `oe=ie`：

```cpp
bool Query_block::build_sj_cond(THD *thd, NESTED_JOIN *nested_join,
                                Query_block *subq_query_block,
                                table_map outer_tables_map, Item **sj_cond) {
  if (nested_join->sj_inner_exprs.empty()) {
    // Semi-join materialization requires a key, push a constant integer item
    Item *const_item = new Item_int(1);
    if (const_item == nullptr) return true;
    nested_join->sj_inner_exprs.push_back(const_item);
    nested_join->sj_outer_exprs.push_back(const_item);
    // ★★ 坑：EXISTS 子查询没有 IN 等值 ⇒ 塞一个常量 1=1 当"key"占位，
    //    否则 semijoin materialization 无法在临时表上建键
  }

  auto ii = nested_join->sj_inner_exprs.begin();
  auto oi = nested_join->sj_outer_exprs.begin();
  while (ii != nested_join->sj_inner_exprs.end() &&
         oi != nested_join->sj_outer_exprs.end()) {
    bool should_remove = false;
    Item *inner = *ii;
    Item *outer = *oi;
    outer->fix_after_pullout(this, subq_query_block);
    inner->fix_after_pullout(this, subq_query_block);

    Item_func_eq *item_eq = new Item_func_eq(outer, inner);
    if (item_eq == nullptr) return true;
    Item *predicate = item_eq;
    if (!item_eq->fixed && item_eq->fix_fields(thd, &predicate)) return true;

    // Evaluate if the condition is on const expressions
    if (predicate->const_item() &&
        !(predicate)->walk(&Item::is_non_const_over_literals,
                           enum_walk::POSTFIX, nullptr)) {
      bool cond_value = true;
      /* Push ignore / strict error handler */
      ...
      bool err = eval_const_cond(thd, predicate, &cond_value);
      ...
      if (cond_value) {
        /* Remove the expression ... if this is not the last expression in the
           list. Semijoin processing expects at least one inner/outer expression
           in the list if there is a sj_nest present. */
        if (nested_join->sj_inner_exprs.size() != 1) {
          should_remove = true;
        }
      } else {
        /* Remove all the expressions ... Add an always false condition. */
        nested_join->sj_inner_exprs.clear();
        nested_join->sj_outer_exprs.clear();
        Item *new_item = new Item_func_false();
        if (new_item == nullptr) return true;
        (*sj_cond) = new_item;
        break;   // ★ 恒假 ⇒ 整个 semijoin 退化成 FALSE
      }
    }
    (*sj_cond) = and_items(*sj_cond, predicate);
    if (*sj_cond == nullptr) return true;

    nested_join->sj_corr_tables |= inner->used_tables() & outer_tables_map;
    // ★★ 坑：只要 inner 表达式引用了外层表 ⇒ 记为"非平凡相关" ⇒ 禁用 materialization

    if (should_remove) {
      ii = nested_join->sj_inner_exprs.erase(ii);
      oi = nested_join->sj_outer_exprs.erase(oi);
    } else {
      ++ii, ++oi;
    }
  }
  return false;
}
```

**去相关化的两个类/函数**（`:2586` / `:2632` / `:2767`）：

```cpp
/// Context object used by semijoin equality decorrelation code.
class Semijoin_decorrelation {
  mem_root_deque<Item *> *sj_outer_exprs, *sj_inner_exprs;
  /// If nullptr: only a=b is decorrelated.
  /// Otherwise, a OP b is decorrelated for OP in <>, >=, >, <=, <, and
  /// for each decorrelated SJ outer/inner pair, located at position N
  /// in sj_outer_exprs and sj_inner_exprs, we store, at the
  /// same position in op_types, the operator's type code representing
  /// "outer OP inner" (for example, LE_FUNC for outer<=inner as well as
  /// inner>=outer).
  Mem_root_array<Item_func::Functype> *op_types;
  ...
  bool decorrelate_only_eq() const { return op_types == nullptr; }
  ...
};

static bool decorrelate_equality(Semijoin_decorrelation &sj_decor,
                                 Item_func *func, bool *was_correlated) {
  *was_correlated = false;
  Item_bool_func2 *bool_func = down_cast<Item_bool_func2 *>(func);
  Item *const left = bool_func->arguments()[0];
  Item *const right = bool_func->arguments()[1];
  Item *inner = nullptr;
  Item *outer = nullptr;
  table_map left_used_tables = left->used_tables() & ~INNER_TABLE_BIT;
  table_map right_used_tables = right->used_tables() & ~INNER_TABLE_BIT;

  /* Predicates that have non-deterministic elements are not decorrelated. */
  if ((left_used_tables & RAND_TABLE_BIT) ||
      (right_used_tables & RAND_TABLE_BIT))
    return false;   // ★ 含 RAND() 的谓词不去相关

  if (left_used_tables == OUTER_REF_TABLE_BIT) {
    outer = left;
  } else if (!(left_used_tables & OUTER_REF_TABLE_BIT)) {
    inner = left;
  }
  if (right_used_tables == OUTER_REF_TABLE_BIT) {
    outer = right;
  } else if (!(right_used_tables & OUTER_REF_TABLE_BIT)) {
    inner = right;
  }
  if (inner == nullptr || outer == nullptr) return false;

  // Equalities over row items cannot be decorrelated
  if (outer->type() == Item::ROW_ITEM) return false;

  sj_decor.add_outer(outer);
  sj_decor.add_inner(inner);
  if (sj_decor.add_op_type(
          // use canonical form "outer OP inner":
          (outer == left) ? bool_func->functype() : bool_func->rev_functype()))
    return true;

  *was_correlated = true;
  return false;
}

static inline bool can_decorrelate_operator(Item_func *func, bool only_eq) {
  auto op_type = func->functype();
  switch (op_type) {
    case Item_func::EQ_FUNC:
      return true;
    case Item_func::NE_FUNC:
    case Item_func::LT_FUNC:
    case Item_func::LE_FUNC:
    case Item_func::GT_FUNC:
    case Item_func::GE_FUNC:
      return !only_eq;   // ★ 只有传了 op_types（非 nullptr）才处理非等值
    default:
      return false;
  }
}

bool Query_block::decorrelate_condition(Semijoin_decorrelation &sj_decor,
                                        Table_ref *join_nest) {
  Item *base_cond =
      join_nest == nullptr ? where_cond() : join_nest->join_cond();
  Item_cond *cond;
  Item_func *func;
  assert(base_cond != nullptr);

  if (base_cond->type() == Item::FUNC_ITEM &&
      (func = down_cast<Item_func *>(base_cond)) &&
      can_decorrelate_operator(func, sj_decor.decorrelate_only_eq())) {
    bool was_correlated;
    if (decorrelate_equality(sj_decor, func, &was_correlated)) return true;
    if (was_correlated) {  // The simple equality has been decorrelated
      if (join_nest == nullptr)
        set_where_cond(nullptr);
      else  // Join conditions cannot be empty so install a TRUE value
        join_nest->set_join_cond(new Item_func_true());
    }
  } else if (base_cond->type() == Item::COND_ITEM &&
             (cond = down_cast<Item_cond *>(base_cond)) &&
             cond->functype() == Item_func::COND_AND_FUNC) {
    List<Item> *args = cond->argument_list();
    List_iterator<Item> li(*args);
    Item *item;
    while ((item = li++)) {
      if (item->type() == Item::FUNC_ITEM &&
          (func = down_cast<Item_func *>(item)) &&
          can_decorrelate_operator(func, sj_decor.decorrelate_only_eq())) {
        bool was_correlated;
        if (decorrelate_equality(sj_decor, func, &was_correlated)) return true;
        if (was_correlated) li.remove();
        // ★ 谓词被"摘走"：从 WHERE 中删除，只留在 sj_inner/outer_exprs 里
        //   因为它已经变成 semijoin 的 ON 条件
      }
    }
    if (args->is_empty()) {  // All predicates have been decorrelated
      if (join_nest == nullptr)
        set_where_cond(nullptr);
      else
        join_nest->set_join_cond(new Item_func_true());
    }
  }
  return false;
}
```

> **勘误**：常见资料里写的 `fix_semijoin_expr_list()` 在 8.0.39 **不存在**（5.6 时代 API）；`substitute_for_best_equal_field()`（`sql/sql_optimizer.cc:4773`）属于**等值传播**阶段，不是 semijoin 去相关化路径的成员。

---

## 五、Optimize Phase：五种策略的代价选择

### 4.1 make_join_plan 的前置

```
pull_out_semijoin_tables            把 EQ_REF 的 it 抽出 sj-nest（1:1 不膨胀）
set_semijoin_embedding              设每个 join_tab->emb_sj_nest
update_semijoin_strategies          设 sj_enabled_strategies
optimize_semijoin_nests_for_materialization  预优化可物化的 sj-nest 内表 join order
```

`pull_out_semijoin_tables` 的关键：EQ_REF 保证"能 join 到且只能 join 到一条"，对存在性语义无贡献，抽出去后可缩小去重范围。

### 4.2 advance_sj_state：贪心搜索中的策略状态机

`sql/sql_planner.cc:4105-4599`。在 `best_extension_by_limited_search` 每加一张表后调用，为当前 join 前缀判定/更新五种策略。核心不变量是 `dups_producing_tables`（还没被任何策略消除的 sj 内层表位图，结束时必须为 0）。

每种策略在 `POSITION` 上有一组状态变量：

| 策略 | 状态变量 |
|------|---------|
| FirstMatch | `first_firstmatch_table`、`firstmatch_need_tables` |
| LooseScan | `first_loosescan_table`、`loosescan_need_tables` |
| Materialize | `sjm`（Semijoin_mat_optimize） |
| DuplicateWeedout | `first_dupsweedout_table`、`dupsweedout_tables` |

**完整源码**（`sql/sql_planner.cc:4105-4599`，按策略分段）：

```cpp
void Optimize_table_order::advance_sj_state(table_map remaining_tables,
                                            const JOIN_TAB *new_join_tab,
                                            uint idx) {
  Opt_trace_context *const trace = &thd->opt_trace;
  Table_ref *const emb_sj_nest = new_join_tab->emb_sj_nest;
  POSITION *const pos = join->positions + idx;
  double best_cost = pos->prefix_cost;
  double best_rowcount = pos->prefix_rowcount;
  uint sj_strategy = SJ_OPT_NONE;  // Initially: No chosen strategy

  /*
    Semi-join nests cannot be nested ... In fact, doing this may cause
    undesirable effects because all tables within a semi-join nest have
    emb_sj_nest != NULL, which triggers several of the actions inside this
    function.
  */
  assert(emb_sjm_nest == nullptr);

  // remaining_tables include the current one:
  assert(remaining_tables & new_join_tab->table_ref->map());
  const table_map remaining_tables_incl = remaining_tables;
  // And add the current table to the join prefix:
  remaining_tables &= ~new_join_tab->table_ref->map();

  Opt_trace_array trace_choices(trace, "semijoin_strategy_choice");

  /* Initialize the state or copy it from prev. tables */
  pos->cur_embedding_map = cur_embedding_map;
  if (idx == join->const_tables) {
    pos->dups_producing_tables = 0;
    pos->first_firstmatch_table = MAX_TABLES;
    pos->first_loosescan_table = MAX_TABLES;
    pos->dupsweedout_tables = 0;
    pos->sjm_scan_need_tables = 0;
    pos->sjm_scan_last_inner = 0;
  } else {
    pos->dups_producing_tables = pos[-1].dups_producing_tables;

    // FirstMatch
    pos->first_firstmatch_table = pos[-1].first_firstmatch_table;
    pos->first_firstmatch_rtbl = pos[-1].first_firstmatch_rtbl;
    pos->firstmatch_need_tables = pos[-1].firstmatch_need_tables;

    // LooseScan
    pos->first_loosescan_table = (pos[-1].sj_strategy == SJ_OPT_LOOSE_SCAN)
                                     ? MAX_TABLES
                                     : pos[-1].first_loosescan_table;
    // ★ 上一个已经用了 LS ⇒ 重置，避免同一 nest 连续两次 LooseScan
    pos->loosescan_need_tables = pos[-1].loosescan_need_tables;

    // MaterializeScan
    pos->sjm_scan_need_tables = (pos[-1].sj_strategy == SJ_OPT_MATERIALIZE_SCAN)
                                    ? 0
                                    : pos[-1].sjm_scan_need_tables;
    pos->sjm_scan_last_inner = pos[-1].sjm_scan_last_inner;

    // Duplicate Weedout
    pos->dupsweedout_tables = pos[-1].dupsweedout_tables;
    pos->first_dupsweedout_table = pos[-1].first_dupsweedout_table;
  }

  table_map handled_by_fm_or_ls = 0;

  /* ===== FirstMatch Strategy ===== */
  if (pos->first_firstmatch_table != MAX_TABLES) {
    const Table_ref *first_emb_sj_nest =
        join->positions[pos->first_firstmatch_table].table->emb_sj_nest;
    if (emb_sj_nest != first_emb_sj_nest) {
      // Can't handle interleaving between tables from the
      // semi-join that FirstMatch is handling and any other tables.
      pos->first_firstmatch_table = MAX_TABLES;
    }
  }

  if (emb_sj_nest && emb_sj_nest->nested_join->sj_enabled_strategies &
                         OPTIMIZER_SWITCH_FIRSTMATCH) {
    const table_map outer_corr_tables = emb_sj_nest->nested_join->sj_depends_on;
    const table_map sj_inner_tables = emb_sj_nest->sj_inner_tables;
    /*
      Enter condition:
       1. The next join tab belongs to semi-join nest
       2. We're not in a duplicate producer range yet
       3. All outer tables that the subquery is correlated with, or
          referred to from the outer_expr, are in the join prefix
    */
    if (pos->dups_producing_tables == 0 &&        // (2)
        !(remaining_tables & outer_corr_tables))  // (3)
    {
      /* Start tracking potential FirstMatch range */
      pos->first_firstmatch_table = idx;
      pos->firstmatch_need_tables = 0;
      pos->firstmatch_rtbl = remaining_tables;
      assert(sj_inner_tables == (remaining_tables_incl & sj_inner_tables));
    }

    if (pos->first_firstmatch_table != MAX_TABLES) {
      /* Record that we need all of this semi-join's inner tables */
      pos->firstmatch_need_tables |= sj_inner_tables;

      if (outer_corr_tables & pos->first_firstmatch_rtbl) {
        /* Trying to add an sj-inner table whose sj-nest has an outer
           correlated table that was not in the prefix. FirstMatch can't be used. */
        pos->first_firstmatch_table = MAX_TABLES;
      } else if (!(pos->firstmatch_need_tables & remaining_tables)) {
        // Got a complete FirstMatch range.

        /* We cannot FirstMatch to a different embedding nest, e.g. for
           B LEFT JOIN (C SEMIJOIN D ON B.X=D.Y) and table order B-D-C we
           cannot jump from D to B. This would cause non-hierarchical joins.
           So we check that the jump won't leave from a still-open nest:
           cur_embedding_map at the last table of this firstmatch range
           must be included in cur_embedding_map at the target of the jump. */
        nested_join_map cur_embedding_map_at_jump_target =
            pos->first_firstmatch_table > join->const_tables
                ? join->positions[pos->first_firstmatch_table - 1]
                      .cur_embedding_map
                : 0;
        if ((cur_embedding_map_at_jump_target & cur_embedding_map) !=
            cur_embedding_map) {
          pos->first_firstmatch_table = MAX_TABLES;
        } else {
          // Calculate access paths and cost
          double cost, rowcount;
          Opt_trace_object trace_one_strategy(trace);
          trace_one_strategy.add_alnum("strategy", "FirstMatch");
          (void)semijoin_firstmatch_loosescan_access_paths(
              pos->first_firstmatch_table, idx, remaining_tables, false,
              &rowcount, &cost);
          /*
            We don't yet know what are the other strategies, so pick FirstMatch.
            We ought to save the alternate POSITIONs produced by
            semijoin_firstmatch_loosescan_access_paths() but the problem is that
            providing save space uses too much space. Instead, we will
            re-calculate the alternate POSITIONs after we've picked the best QEP.
          */
          sj_strategy = SJ_OPT_FIRST_MATCH;
          best_cost = cost;
          best_rowcount = rowcount;
          trace_one_strategy.add("cost", best_cost).add("rows", best_rowcount);
          handled_by_fm_or_ls = pos->firstmatch_need_tables;
          trace_one_strategy.add("chosen", true);
        }
      }
    }
  }

  /* ===== LooseScan Strategy ===== */
  {
    if (pos->first_loosescan_table != MAX_TABLES) {
      const Table_ref *first_emb_sj_nest =
          join->positions[pos->first_loosescan_table].table->emb_sj_nest;
      if (first_emb_sj_nest->sj_inner_tables & remaining_tables_incl) {
        // Stage 2: Accept remaining tables from the semi-join nest:
        if (emb_sj_nest != first_emb_sj_nest) {
          pos->first_loosescan_table = MAX_TABLES;
        } else {
          /*
            NestedLoopSemiJoinWithDuplicateRemovalIterator takes a
            single-table iterator as left argument, and inner-joins
            it with the set of other SJ-inner tables. E.g. it doesn't work for
            A SEMI JOIN (B LEFT JOIN C) with B as LooseScan table. So:
            - if we're now at the second SJ-inner table (1), and
            - this table belongs to a join nest which is outer-joined to
              the first SJ-inner table (2), or is directly outer-joined to the
              first SJ-inner table (3),
            - then both tables are not inner-joined together and LooseScan is
              impossible.
          */
          if (idx == pos->first_loosescan_table + 1 &&
              ((pos->table->table_ref->outer_join_nest() !=
                join->positions[pos->first_loosescan_table]
                    .table->table_ref->outer_join_nest()) ||
               pos->table->table_ref->outer_join))
            pos->first_loosescan_table = MAX_TABLES;
        }
      } else {
        // Stage 3: Accept outer dependent and non-dependent tables:
        assert(emb_sj_nest != first_emb_sj_nest);
        if (emb_sj_nest != nullptr) pos->first_loosescan_table = MAX_TABLES;
      }
    }

    /* We may consider the LooseScan strategy if ... (8 条见 4.3) */
    if (emb_sj_nest && ... ) {
      pos->first_loosescan_table = idx;
      pos->loosescan_need_tables = emb_sj_nest->sj_inner_tables |
                                   emb_sj_nest->nested_join->sj_depends_on;
    }

    if ((pos->first_loosescan_table != MAX_TABLES) &&
        !(remaining_tables & pos->loosescan_need_tables)) {
      /* Ok we have all LooseScan sj-nest's inner tables and outer correlated
         tables into the prefix. */
      double cost, rowcount;
      Opt_trace_object trace_one_strategy(trace);
      trace_one_strategy.add_alnum("strategy", "LooseScan");
      /* If this function returns 'false', it means LS is impossible
         (didn't find a suitable index, etc). */
      if (semijoin_firstmatch_loosescan_access_paths(pos->first_loosescan_table,
                                                     idx, remaining_tables,
                                                     true, &rowcount, &cost)) {
        /* We don't yet have any other strategies that could handle this
           semi-join nest ... so unconditionally pick the LooseScan. */
        sj_strategy = SJ_OPT_LOOSE_SCAN;
        best_cost = cost;
        best_rowcount = rowcount;
        handled_by_fm_or_ls = join->positions[pos->first_loosescan_table]
                                  .table->emb_sj_nest->sj_inner_tables;
      }
      trace_one_strategy.add("chosen", sj_strategy == SJ_OPT_LOOSE_SCAN);
    }
  }

  if (emb_sj_nest) pos->dups_producing_tables |= emb_sj_nest->sj_inner_tables;
  pos->dups_producing_tables &= ~handled_by_fm_or_ls;
  // ★★ 核心不变量：dups_producing_tables = 还没被任何策略消除的 sj 内层表

  /* ===== MaterializeLookup and MaterializeScan strategy handler ===== */
  const int sjm_strategy = semijoin_order_allows_materialization(
      join, remaining_tables, new_join_tab, idx);
  if (sjm_strategy == SJ_OPT_MATERIALIZE_SCAN) {
    /*
      We cannot evaluate this option now. This is because we cannot
      account for fanout of sj-inner tables yet:
        ntX  SJM-SCAN(it1 ... itN) | ot1 ... otN  |
      The simple way to model this is to remove SJM-SCAN(...) fanout once
      we reach the point #2.
    */
    if (pos->sjm_scan_need_tables && emb_sj_nest != nullptr &&
        emb_sj_nest !=
            join->positions[pos->sjm_scan_last_inner].table->emb_sj_nest)
      /* Prevent that inner tables of different semijoin nests are
         interleaved for MatScan. */
      pos->sjm_scan_need_tables = 0;
    else {
      pos->sjm_scan_need_tables = emb_sj_nest->sj_inner_tables |
                                  emb_sj_nest->nested_join->sj_depends_on;
      pos->sjm_scan_last_inner = idx;
      Opt_trace_object(trace)
          .add_alnum("strategy", "MaterializeScan")
          .add_alnum("choice", "deferred");   // ★ 延后到"到达点#2"再算
    }
  } else if (sjm_strategy == SJ_OPT_MATERIALIZE_LOOKUP) {
    double cost, rowcount;
    semijoin_mat_lookup_access_paths(idx, emb_sj_nest, &rowcount, &cost);
    Opt_trace_object trace_one_strategy(trace);
    trace_one_strategy.add_alnum("strategy", "MaterializeLookup")
        .add("cost", cost).add("rows", rowcount)
        .add("duplicate_tables_left", pos->dups_producing_tables != 0);
    if (cost < best_cost || pos->dups_producing_tables) {
      /*
        NOTE: When we pick to use SJM[-Scan] we don't memcpy its POSITION
        elements to join->positions as that makes it hard to return things
        back when making one step back in join optimization. That's done
        after the QEP has been chosen.
      */
      sj_strategy = SJ_OPT_MATERIALIZE_LOOKUP;
      best_cost = cost;
      best_rowcount = rowcount;
      pos->dups_producing_tables &= ~emb_sj_nest->sj_inner_tables;
    }
    trace_one_strategy.add("chosen", sj_strategy == SJ_OPT_MATERIALIZE_LOOKUP);
  }

  /* MaterializeScan second phase check */
  if (pos->sjm_scan_need_tables && emb_sj_nest != nullptr &&
      emb_sj_nest !=
          join->positions[pos->sjm_scan_last_inner].table->emb_sj_nest)
    pos->sjm_scan_need_tables = 0;

  if (pos->sjm_scan_need_tables && /* Have SJM-Scan prefix */
      !(pos->sjm_scan_need_tables & remaining_tables)) {
    Table_ref *const sjm_nest =
        join->positions[pos->sjm_scan_last_inner].table->emb_sj_nest;
    double cost, rowcount;
    Opt_trace_object trace_one_strategy(trace);
    trace_one_strategy.add_alnum("strategy", "MaterializeScan");
    semijoin_mat_scan_access_paths(pos->sjm_scan_last_inner, idx,
                                   remaining_tables, sjm_nest, &rowcount,
                                   &cost);
    trace_one_strategy.add("cost", cost).add("rows", rowcount)
        .add("duplicate_tables_left", pos->dups_producing_tables != 0);
    /* Use the strategy if
        * it is cheaper then what we've had, or
        * we haven't picked any other semi-join strategy yet
       In the second case, we pick this strategy unconditionally because
       comparing cost without semi-join duplicate removal with cost with
       duplicate removal is not an apples-to-apples comparison. */
    if (cost < best_cost || pos->dups_producing_tables) {
      sj_strategy = SJ_OPT_MATERIALIZE_SCAN;
      best_cost = cost;
      best_rowcount = rowcount;
      pos->dups_producing_tables &= ~sjm_nest->sj_inner_tables;
    }
    trace_one_strategy.add("chosen", sj_strategy == SJ_OPT_MATERIALIZE_SCAN);
  }

  /* ===== Duplicate Weedout strategy handler ===== */
  {
    if (emb_sj_nest) {
      if (!pos->dupsweedout_tables) pos->first_dupsweedout_table = idx;
      pos->dupsweedout_tables |= emb_sj_nest->sj_inner_tables |
                                 emb_sj_nest->nested_join->sj_depends_on;
    }

    if (pos->dupsweedout_tables &&
        !(remaining_tables & pos->dupsweedout_tables)) {
      Opt_trace_object trace_one_strategy(trace);
      trace_one_strategy.add_alnum("strategy", "DuplicatesWeedout");
      double rowcount, cost;
      semijoin_dupsweedout_access_paths(pos->first_dupsweedout_table, idx,
                                        &rowcount, &cost);
      trace_one_strategy.add("cost", cost).add("rows", rowcount)
          .add("duplicate_tables_left", pos->dups_producing_tables != 0);
      /* Use the strategy if
          * it is cheaper then what we've had, and strategy is enabled, or
          * we haven't picked any other semi-join strategy yet
         The second part is necessary because this strategy is the last one
         to consider (it needs "the most" tables in the prefix) and we can't
         leave duplicate-producing tables not handled by any strategy. */
      if ((cost < best_cost &&
           join->positions[pos->first_dupsweedout_table]
                   .table->emb_sj_nest->nested_join->sj_enabled_strategies &
               OPTIMIZER_SWITCH_DUPSWEEDOUT) ||
          pos->dups_producing_tables) {
        sj_strategy = SJ_OPT_DUPS_WEEDOUT;
        best_cost = cost;
        best_rowcount = rowcount;
        pos->dups_producing_tables &= ~pos->dupsweedout_tables;
      }
      trace_one_strategy.add("chosen", sj_strategy == SJ_OPT_DUPS_WEEDOUT);
    }
  }

  pos->sj_strategy = sj_strategy;
  /*
    If a semi-join strategy is chosen, update cost and rowcount in positions
    ... Tables in a semijoin range, except the last in range, won't have their
    prefix_costs changed below; this is normal ... duplicates elimination
    happens only at the last table in range.
  */
  if (sj_strategy != SJ_OPT_NONE)
    pos->set_prefix_cost(best_cost, best_rowcount);
}
```

**物化准入**（`sql/sql_planner.cc:2209`）——注意这里**没有代价比较**：

```cpp
static int semijoin_order_allows_materialization(const JOIN *join,
                                                 table_map remaining_tables,
                                                 const JOIN_TAB *tab,
                                                 uint idx) {
  assert(!(remaining_tables & tab->table_ref->map()));
  /*
   Check if
    1. We're in a semi-join nest that can be run with SJ-materialization
    2. All the tables from the subquery are in the prefix
  */
  const Table_ref *emb_sj_nest = tab->emb_sj_nest;
  if (!emb_sj_nest || !emb_sj_nest->nested_join->sjm.positions ||
      (remaining_tables & emb_sj_nest->sj_inner_tables))
    return SJ_OPT_NONE;
  // ★ sjm.positions == nullptr ⇒ 该 nest 压根没算过物化代价（optimize_semijoin_nests_for_materialization 阶段被否）

  /* Walk back and check if all immediately preceding tables are from
     this semi-join. */
  const uint n_tables = my_count_bits(emb_sj_nest->sj_inner_tables);
  for (uint i = 1; i < n_tables; i++) {
    if (join->positions[idx - i].table->emb_sj_nest != emb_sj_nest)
      return SJ_OPT_NONE;   // ★ 内表必须连续，不能与外表交错
  }

  /* Must use MaterializeScan strategy if there are outer correlated tables
     among the remaining tables, otherwise, if possible, use MaterializeLookup.*/
  if ((remaining_tables & emb_sj_nest->nested_join->sj_depends_on) ||
      !emb_sj_nest->nested_join->sjm.lookup_allowed) {
    if (emb_sj_nest->nested_join->sjm.scan_allowed)
      return SJ_OPT_MATERIALIZE_SCAN;
    return SJ_OPT_NONE;
  }
  return SJ_OPT_MATERIALIZE_LOOKUP;
  // ★ 能 Lookup 就 Lookup，不与 Scan 比价
}
```

### 4.3 五种策略详解

#### FirstMatch（首匹配）

**原理**：外表行与内表 JOIN 得到一条输出后，**直接跳过所有剩余内表行**，读下一行外表。

**进入条件**（源码注释明确列了 3 条）：

```
1. 下一张 join tab 属于这个 semi-join nest
2. 还没进入 duplicate producer range（dups_producing_tables == 0）
3. 所有相关外表（sj_depends_on / 被 outer_expr 引用的表）都在 join 前缀
```

```cpp
if (pos->dups_producing_tables == 0 &&        // (2)
    !(remaining_tables & outer_corr_tables))  // (3)
{
  pos->first_firstmatch_table = idx;   // 开始跟踪 FirstMatch range
  pos->firstmatch_need_tables = 0;
  pos->first_firstmatch_rtbl = remaining_tables;
}
```

**两条"中途放弃"的检查**（不是进入时判定，而是**追踪过程中**发现不行就置 `MAX_TABLES`）：

- **交织检查**：FirstMatch 不能处理"它正在处理的 semi-join 的表"与"其它 nest 的表"交织。源码注释给了两种形态的对照：

```
Intertwined tables: ot - FM(it11 - it21 - it12 - it22)   ← 交织，不允许
Grouped tables:     ot - FM(it11 - it12) - FM(it21 - it22) ← 分组，允许
```

- **非层次 join 检查**（`cur_embedding_map`）：不能跳出一个"还开着的 nest"。注释给的例子是 `B LEFT JOIN (C SEMIJOIN D ON B.X=D.Y)`，表序 `B-D-C` 时，从 D 跳回 B 会穿过还没闭合的 outer join nest，导致非层次 join——所以检查 `cur_embedding_map` 是否包含在跳转目标的 `cur_embedding_map` 里。

#### LooseScan（松散扫描）

**原理**：内表驱动表用索引做"松散扫描"（只扫描索引键不同的记录，一个 group 只取首行），天然去重。

**进入条件**（源码注释列了完整的 8 条，比 FirstMatch 严得多）：

```
1a. 下一张表是 SJ 内表
1b. LooseScan 对该 nest 启用（OPTIMIZER_SWITCH_LOOSE_SCAN）
2.  IN 表达式数 ≤ 64   ★ 因为要"fit in bitmap"（位图大小限制）
3.  是该 semi-join 的第一张表
4.  不在另一个 semi-join range 内（dups_producing_tables == 0）
5.  所有非 IN 等值的相关引用都已绑定
6.  但还有些 IN 等值没绑定  ← 否则就该走 FirstMatch 了
7.  有能被这张表索引处理的等值（keyuse() != nullptr）
8.  不是派生表/视图（临时限制）
```

```cpp
if (emb_sj_nest &&                                                  // (1a)
    emb_sj_nest->nested_join->sj_enabled_strategies &
        OPTIMIZER_SWITCH_LOOSE_SCAN &&                              // (1b)
    emb_sj_nest->nested_join->sj_inner_exprs.size() <= 64 &&        // (2) 位图限制
    ((remaining_tables_incl & emb_sj_nest->sj_inner_tables) ==
     emb_sj_nest->sj_inner_tables) &&                               // (3)
    pos->dups_producing_tables == 0 &&                              // (4)
    !(remaining_tables_incl & emb_sj_nest->nested_join->sj_corr_tables) &&  // (5)
    (remaining_tables_incl & emb_sj_nest->nested_join->sj_depends_on) &&    // (6)
    new_join_tab->keyuse() != nullptr &&                            // (7)
    !new_join_tab->table_ref->uses_materialization())               // (8)
{
  pos->first_loosescan_table = idx;
  pos->loosescan_need_tables =
      emb_sj_nest->sj_inner_tables | emb_sj_nest->nested_join->sj_depends_on;
}
```

★ **为什么 FirstMatch 与 LooseScan "相反"**：看条件 (6)——LooseScan 要求"还有相关外表没进前缀"，而 FirstMatch 要求"相关外表都在前缀"。所以两者是对同一类输入的**互斥分流**：相关外表在前缀 → FirstMatch；相关外表还在后面 → LooseScan（用索引跳扫，把相关外表留到后面）。

**两条额外的放弃检查**：

- **LooseScan 的 outer join 限制**：LooseScan 要求驱动表与其它内表是 **inner join**。注释给了反例 `A SEMI JOIN (B LEFT JOIN C)`——B 做 LooseScan 时，B 与 C 是 outer join，LooseScan 不可能。判定是"第二张内表时，若两表的 `outer_join_nest` 不同、或第二表直接是 `outer_join`，则置 `MAX_TABLES`"。

- **stage 机制**：LooseScan 的追踪分三个阶段（Stage 1 只有驱动表、Stage 2 加同 nest 的内表、Stage 3 加相关外表），进入 Stage 3 时若又碰到别的 nest 的内表，也置 `MAX_TABLES`（因为 LooseScan 不能处理与其它 nest 交织，与 FirstMatch 的交织检查同理）。

**索引可用性判定**（8.0 已无 `setup_semijoin_loosescan()`，改为在代价计算里判定）：`semijoin_loosescan_fill_driving_table_position()`（`sql/sql_planner.cc:1609`）。先看"哪些 sj 等值已绑定"：

```cpp
// sql/sql_planner.cc:1515
static ulonglong get_bound_sj_equalities(const JOIN_TAB *tab,
                                         table_map not_available_tables) {
  ulonglong bound_sj_equalities = 0;
  auto it_o = tab->emb_sj_nest->nested_join->sj_outer_exprs.begin();
  auto it_i = tab->emb_sj_nest->nested_join->sj_inner_exprs.begin();
  for (uint i = 0; it_o != tab->emb_sj_nest->nested_join->sj_outer_exprs.end();
       ++i, ++it_o, ++it_i) {
    Item *outer = *it_o, *inner = *it_i;
    if (!((not_available_tables)&outer->used_tables())) {
      bound_sj_equalities |= 1ULL << i;
      continue;   // ★ outer 表达式只依赖已可用表 ⇒ 该 sj 等值"已绑定"
    }
    /*
      Now we look at equality propagation, to discover that a semi-join
      equality is bound, when the outer or inner expression is a field
      involved in some other non-semi-join equality. ...
    */
    Item *item;
    if (outer->type() == Item::FIELD_ITEM)
      item = outer;
    else if (inner->type() == Item::FIELD_ITEM)
      item = inner;
    else
      continue;
    Item_field *const item_field = static_cast<Item_field *>(item);
    Item_equal *item_equal = item_field->item_equal;
    if (!item_equal) {
      Table_ref *const nest = item_field->table_ref->outer_join_nest();
      item_equal = item_field->find_item_equal(nest ? nest->cond_equal
                                                    : tab->join()->cond_equal);
    }
    if (item_equal) {
      /* If the multiple equality {[optional_constant,] col1, col2...} contains
         (1) a constant  (2) or a column from an available table
         then the semi-join equality is bound. */
      if (item_equal->const_arg() ||
          (item_equal->used_tables() & ~not_available_tables))
        bound_sj_equalities |= 1ULL << i;
    }
  }
  return bound_sj_equalities;
}
```

```cpp
// sql/sql_planner.cc:1609（关键段）
bool Optimize_table_order::semijoin_loosescan_fill_driving_table_position(
    const JOIN_TAB *tab, table_map remaining_tables, uint idx,
    double prefix_rowcount, POSITION *pos) {
  ...
  const ulonglong bound_sj_equalities =
      get_bound_sj_equalities(tab, excluded_tables | remaining_tables);

  pos->read_cost = DBL_MAX;
  pos->use_join_buffer = false;
  ...
  Opt_trace_array trace_all_idx(trace, "indexes");

  for (Key_use *keyuse = tab->keyuse(); keyuse->table_ref == tab->table_ref;) {
    const uint key = keyuse->key;
    Key_use *const start_key = keyuse;
    ...
    ulonglong handled_sj_equalities = 0;
    key_part_map handled_keyparts = 0;
    uint max_keypart = 0;

    // For each keypart
    while (keyuse->table_ref == tab->table_ref && keyuse->key == key) {
      const uint keypart = keyuse->keypart;
      for (; keyuse->table_ref == tab->table_ref && keyuse->key == key &&
             keyuse->keypart == keypart;
           ++keyuse) {
        /* If this Key_use is not about a semi-join equality, or references an
           excluded table, or does not reference a not-yet-available table, or
           is for fulltext, or is over a prefix, then it is not a "handled sj
           equality". */
        if ((keyuse->sj_pred_no == UINT_MAX) ||
            (excluded_tables & keyuse->used_tables) ||
            !(remaining_tables & keyuse->used_tables) ||
            (keypart == FT_KEYPART) ||
            (table->key_info[key].key_part[keypart].key_part_flag &
             HA_PART_KEY_SEG))
          continue;   // ★★ 前缀索引(HA_PART_KEY_SEG)被拒：前缀相同但完整值不同会被误判重复
        handled_sj_equalities |= 1ULL << keyuse->sj_pred_no;
        handled_keyparts |= keyuse->keypart_map;
        assert(max_keypart <= keypart);  // see sort_keyuse()
        max_keypart = keypart;
      }
    }

    const key_part_map bound_keyparts = start_key->bound_keyparts;

    /*
      We can use semi-join LooseScan if duplicate elimination is going to work
      for all semi-join equalities. Duplicate elimination:
      - works for a bound semi-join equality, because this equality is tested
        before the nested loop leaves the last inner table of this semi-join nest.
      - works for a handled semi-join equality thanks to key comparison; key
        comparison works if:
          * the handled key parts are over a full field (not a prefix ...)
          * and any key part before the handled key parts, is bound ...
      That gives us these requirements:
      1. All IN-equalities are either bound or handled.
      2. No hole in sequence of key parts.
      An example where (2) matters:
        SELECT * FROM ot1
        WHERE a IN (SELECT it1.b FROM it1 JOIN it2 ON it1.a = it2.a).
      Say the plan is it1-ot1-it2 and it1 has an index on (a,b). ... there is a
      hole. If the rows of it1 are, in index order: (X,Z),(Y,Z), then the key
      comparison will let both rows pass; after joining with ot1 this will
      duplicate any row of ot1 having ot1.a=Z.
      We add this third requirement:
      3. At least one IN-equality is handled. (In theory superfluous.)
    */
    if ((handled_sj_equalities | bound_sj_equalities) !=          // (1)
        LOWER_BITS(ulonglong,
                   tab->emb_sj_nest->nested_join->sj_inner_exprs.size())) {
      trace_idx.add("index_handles_needed_semijoin_equalities", false);
      continue;
    }
    if (handled_keyparts == 0) {                                   // (3)
      trace_idx.add("some_index_part_used", false);
      continue;
    }
    if ((LOWER_BITS(key_part_map, max_keypart + 1) &                // (2)
         ~(bound_keyparts | handled_keyparts)) != 0) {
      trace_idx.add("index_can_remove_duplicates", false);
      continue;
    }

    // Ok, can use the strategy

    if (bound_keyparts & 1) {
      /* ref 访问：直接用 Key_use 的代价与 fanout */
      if (start_key->read_cost < pos->read_cost) {
        pos->loosescan_key = key;
        pos->read_cost = start_key->read_cost;
        pos->rows_fetched = start_key->fanout;
        pos->loosescan_parts = max_keypart + 1;
        pos->key = start_key;
        ...
      }
    } else if (tab->table()->covering_keys.is_set(key)) {
      /* 没有可用的 bound 等值 ⇒ 实际是覆盖索引全扫 */
      double rowcount = rows2double(tab->table()->file->stats.records);
      const double cost =
          tab->table()->file->index_scan_cost(key, 1, rowcount).total_cost();
      /* Now find out how many different keys we will get ... */
      if (tab->table()->key_info[key].has_records_per_key(max_keypart)) {
        const rec_per_key_t rpc =
            tab->table()->key_info[key].records_per_key(max_keypart);
        rowcount = rowcount / rpc;   // ★★ LooseScan 的行数折扣：records / records_per_key
      }
      if (cost < pos->read_cost) {
        pos->loosescan_key = key;
        pos->read_cost = cost;
        pos->rows_fetched = rowcount;
        pos->loosescan_parts = max_keypart + 1;
        pos->key = nullptr;
        ...
      }
    }
  }

  if (pos->read_cost != DBL_MAX) {
    pos->read_cost += derived_mat_cost;
    pos->filter_effect = calculate_condition_filter(
        tab, pos->key, ~remaining_tables & ~excluded_tables, pos->rows_fetched,
        false, false, trace_ls);
    return true;   // 找到可用索引
  }
  return false;    // 没找到 ⇒ advance_sj_state 里 LS 分支判 false ⇒ 不选 LS
}
```

⇒ **三条要求的直觉**：(1) 每个 IN 等值要么在进 nest 前就能算出来（bound），要么能被索引键比较兜住（handled）；(2) 索引前缀必须**连续无洞**，否则中间缺的那一段会让不同组混淆；(3) 至少要有一个等值靠索引（否则退化成 FirstMatch 就行）。另外 **`HA_PART_KEY_SEG`（前缀索引）直接出局**——这是最容易踩的坑：`col(10)` 前缀索引上 LooseScan 会漏行。

> 关于"索引必须有序"：8.0.39 已无 `HA_READ_ORDER` 判定；等价物是执行侧 `setup_semijoin_dups_elimination()` 里的 `set_need_sorted_output(tab->range_scan())`（`sql/sql_select.cc:1566`）——要求 range scan 保持索引顺序（MRR 会破坏顺序）。

#### MaterializeLookup / MaterializeScan（物化）

**原理**：内表物化到去重临时表，外层与临时表 JOIN。根据临时表是扫描还是索引查找分两种。

**适用条件**：内表作为连续块全部进前缀；类型可物化（无 BLOB、key 长度不超限）。

```cpp
if ((remaining_tables & emb_sj_nest->nested_join->sj_depends_on) || !lookup_allowed) {
  if (scan_allowed) return SJ_OPT_MATERIALIZE_SCAN;   // 还有相关外表 → 只能扫
}
return SJ_OPT_MATERIALIZE_LOOKUP;                      // 否则索引查找
```

#### DuplicateWeedout（重复剔除）

**原理**：兜底策略。把半连接当普通内连接执行，但用临时表按**外层表 rowid** 去重。对 join order 几乎无约束。

**适用条件**：最通用，几乎任意 join order。选择条件里 `pos->dups_producing_tables`（还有未消除重复）时**无条件采纳**（因为"未去重成本"和"已去重成本"不是同量纲）。

#### 代价计算与"先选后重算"

五个策略的**代价估算**收敛在少数几个函数上：

| 策略 | 代价函数 |
|---|---|
| FirstMatch / LooseScan | `semijoin_firstmatch_loosescan_access_paths(first, idx, remaining, is_loosescan, &rowcount, &cost)`——**同一个函数服务两个策略**，靠 `is_loosescan` 参数区分（FM 估算"匹配行数"，LS 估算"松散扫描行数"） |
| MaterializeLookup / Scan | `calculate_materialization_costs()`——算"物化 vs 不物化"两条路，选便宜的那条 |
| DuplicateWeedout | 兜底，几乎不比价（见上） |

**两个关键设计妥协**：

1. **先选后重算**：`advance_sj_state` 在搜索中只保留"当前最优策略"，不保存各策略的备选 `POSITION`（源码注释：*"we need to save POSITIONs somewhere but reserving space for all cases would require too much space. We will re-calculate POSITION structures later on."*）。等 join order 定下来后，由 `fix_semijoin_strategies` 重新算一遍。**代价**：搜索期与最终期的计算必须严格一致，否则计划自相矛盾（该函数历史上出过 bug）。

2. **"无条件选中" LooseScan**：当 LooseScan 的 range 完整时，源码注释明说"unconditionally pick the LooseScan"——理由是其它策略（Weedout/Materialize）需要**至少同样多的表**进前缀才被考虑，此刻 LooseScan 已经是最优候选，无需比价。

### 4.4 fix_semijoin_strategies：确定最终策略

贪心搜索每加一张表判定一次，前后可能选不同策略。`fix_semijoin_strategies` 从后往前遍历（靠后的记录更大前缀的最优策略），把最终策略记到第一个内表上（`n_sj_tables` + `sj_strategy`）。

**完整源码**（`sql/sql_planner.cc:3360-3496`）：

```cpp
bool Optimize_table_order::fix_semijoin_strategies() {
  table_map remaining_tables = 0;
  table_map handled_tables = 0;

  if (join->query_block->sj_nests.empty()) return false;

  Opt_trace_context *const trace = &thd->opt_trace;

  for (uint tableno = join->tables - 1; tableno != join->const_tables - 1;
       tableno--) {   // ★ 从右往左：sj_strategy 记录在 range 的"最后一张表"上
    POSITION *const pos = join->best_positions + tableno;

    if ((handled_tables & pos->table->table_ref->map()) ||
        pos->sj_strategy == SJ_OPT_NONE) {
      remaining_tables |= pos->table->table_ref->map();
      continue;
    }

    uint first = 0;
    if (pos->sj_strategy == SJ_OPT_MATERIALIZE_LOOKUP) {
      Table_ref *const sjm_nest = pos->table->emb_sj_nest;
      const uint table_count = my_count_bits(sjm_nest->sj_inner_tables);
      /*
        This memcpy() copies a partial QEP produced by
        optimize_semijoin_nests_for_materialization() (source) into the final
        top-level QEP (target), in order to re-use the source plan for
        to-be-materialized inner tables.
        It is however possible that the source QEP had picked
        some semijoin strategy (noted SJY), different from
        materialization. The target QEP rules (it has seen more tables), but
        this memcpy() is going to copy the source stale strategy SJY,
        wrongly. Which is why sj_strategy of each table of the
        duplicate-generating range then becomes temporarily unreliable. It is
        fixed for the first table of that range right after the memcpy(), and
        fixed for the rest of that range at the end of this iteration by
        setting it to SJ_OPT_NONE). But until then, pos->sj_strategy should
        not be read.
      */
      memcpy(pos - table_count + 1, sjm_nest->nested_join->sjm.positions,
             sizeof(POSITION) * table_count);
      // ★★ 坑：把 nest 内部预先优化好的子计划 memcpy 回主计划，会带来"过期策略"，
      //    必须靠下面的 for 循环把非首表的 sj_strategy 清成 SJ_OPT_NONE
      first = tableno - table_count + 1;
      join->best_positions[first].n_sj_tables = table_count;
      join->best_positions[first].sj_strategy = SJ_OPT_MATERIALIZE_LOOKUP;

      Opt_trace_object trace_final_strategy(trace);
      trace_final_strategy.add_alnum("final_semijoin_strategy",
                                     "MaterializeLookup");
    } else if (pos->sj_strategy == SJ_OPT_MATERIALIZE_SCAN) {
      const uint last_inner = pos->sjm_scan_last_inner;
      Table_ref *const sjm_nest =
          (join->best_positions + last_inner)->table->emb_sj_nest;
      const uint table_count = my_count_bits(sjm_nest->sj_inner_tables);
      first = last_inner - table_count + 1;
      assert((join->best_positions + first)->table->emb_sj_nest == sjm_nest);
      memcpy(join->best_positions + first,  // stale semijoin strategy here too
             sjm_nest->nested_join->sjm.positions,
             sizeof(POSITION) * table_count);
      join->best_positions[first].sj_strategy = SJ_OPT_MATERIALIZE_SCAN;
      join->best_positions[first].n_sj_tables = table_count;

      Opt_trace_object trace_final_strategy(trace);
      trace_final_strategy.add_alnum("final_semijoin_strategy",
                                     "MaterializeScan");
      // Recalculate final access paths for this semi-join strategy
      double rowcount, cost;
      semijoin_mat_scan_access_paths(last_inner, tableno, remaining_tables,
                                     sjm_nest, &rowcount, &cost);
    } else if (pos->sj_strategy == SJ_OPT_FIRST_MATCH) {
      first = pos->first_firstmatch_table;
      Opt_trace_object trace_final_strategy(trace);
      trace_final_strategy.add_alnum("final_semijoin_strategy", "FirstMatch");

      // Recalculate final access paths for this semi-join strategy
      double rowcount, cost;
      (void)semijoin_firstmatch_loosescan_access_paths(
          first, tableno, remaining_tables, false, &rowcount, &cost);

      if (pos->table->emb_sj_nest->is_aj_nest()) {
        /*
          Antijoin doesn't use the execution logic of FirstMatch. So we
          won't set it up; and we won't either have the incompatibilities of
          FirstMatch with outer join. Declare that we don't use it:
        */
        pos->sj_strategy = SJ_OPT_NONE;   // ★ antijoin 即使"选中"FirstMatch 也不用它
      } else {
        join->best_positions[first].sj_strategy = SJ_OPT_FIRST_MATCH;
        join->best_positions[first].n_sj_tables = tableno - first + 1;
      }
    } else if (pos->sj_strategy == SJ_OPT_LOOSE_SCAN) {
      first = pos->first_loosescan_table;
      Opt_trace_object trace_final_strategy(trace);
      trace_final_strategy.add_alnum("final_semijoin_strategy", "LooseScan");

      double rowcount, cost;
      (void)semijoin_firstmatch_loosescan_access_paths(
          first, tableno, remaining_tables, true, &rowcount, &cost);

      POSITION *const first_pos = join->best_positions + first;
      first_pos->sj_strategy = SJ_OPT_LOOSE_SCAN;
      first_pos->n_sj_tables =
          my_count_bits(first_pos->table->emb_sj_nest->sj_inner_tables);
    } else if (pos->sj_strategy == SJ_OPT_DUPS_WEEDOUT) {
      /*
        Duplicate Weedout starting at pos->first_dupsweedout_table, ending at
        this table.
      */
      first = pos->first_dupsweedout_table;
      join->best_positions[first].sj_strategy = SJ_OPT_DUPS_WEEDOUT;
      join->best_positions[first].n_sj_tables = tableno - first + 1;

      Opt_trace_object trace_final_strategy(trace);
      trace_final_strategy.add_alnum("final_semijoin_strategy",
                                     "DuplicateWeedout");
    }

    for (uint i = first; i <= tableno; i++) {
      /*
        Eliminate stale strategies. See comment in the
        SJ_OPT_MATERIALIZE_LOOKUP case above.
      */
      if (i != first) join->best_positions[i].sj_strategy = SJ_OPT_NONE;
      handled_tables |= join->best_positions[i].table->table_ref->map();
    }

    remaining_tables |= pos->table->table_ref->map();
  }

  assert(remaining_tables == (join->all_table_map & ~join->const_table_map));
  return false;
}
```

⇒ **"先选后重算"在这里收口**：每个策略分支都要**重新调一遍**对应的 `semijoin_*_access_paths()` 做"final calculations"，因为 `advance_sj_state()` 阶段的中间结果没有保存（源码注释：*"we need to save POSITIONs somewhere but reserving space for all cases would require too much space"*）。这也是该函数历史上反复出 bug 的根因——搜索期与最终期的计算必须严格一致。

> **勘误**：`convert_sj_to_subquery()`（5.7 时代"执行期把 semijoin 还原成子查询"的回退函数）在 8.0.39 **已不存在**（全树 0 命中）。现在的等价行为只有两处：优化期 `flatten_subqueries()` 把不合格候选降级为 `UNSPECIFIED`；执行期 `FindSubstructure()` 无法处理的重复消除被塞进 `unhandled_duplicates`，由顶层补一个 weedout 解决。

---

## 六、Execution Phase：每种策略怎么执行

> 本篇从**优化器视角**讲"选定策略后如何落到 QEP/迭代器"。**运行期的执行机制**（物化引擎 `subselect_hash_sj_engine`、IN2EXISTS 的 `Item_in_optimizer`、子查询缓存）详见 [`../../runtime/02_subquery_runtime.md`](../../runtime/02_subquery_runtime.md)。

**`setup_semijoin_dups_elimination()`**（`sql/sql_select.cc:1523`）创建具体执行结构。三种 NL 型策略（FirstMatch / LooseScan / DuplicateWeedout）在同一个 `switch` 里落地（物化策略不在此处——它在 `optimize_semijoin_nests_for_materialization` 阶段已把内表换成 sjm 临时表，见 `assert(false)` 那一行）：

```cpp
// sql/sql_select.cc:1523
static bool setup_semijoin_dups_elimination(JOIN *join, uint no_jbuf_after) {
  uint tableno;
  THD *thd = join->thd;
  DBUG_TRACE;
  ASSERT_BEST_REF_IN_JOIN_ORDER(join);

  if (join->query_block->sj_nests.empty()) return false;

  QEP_TAB *const qep_array = join->qep_tab;
  for (tableno = join->const_tables; tableno < join->primary_tables;) {
#ifndef NDEBUG
    const bool tab_in_sj_nest = join->best_ref[tableno]->emb_sj_nest != nullptr;
#endif
    QEP_TAB *const tab = &qep_array[tableno];
    POSITION *const pos = tab->position();

    if (pos->sj_strategy == SJ_OPT_NONE) {
      tableno++;  // nothing to do
      continue;
    }
    QEP_TAB *last_sj_tab = tab + pos->n_sj_tables - 1;
    // 选择 semi-join stratege
    switch (pos->sj_strategy) {
      case SJ_OPT_MATERIALIZE_LOOKUP:
      case SJ_OPT_MATERIALIZE_SCAN:
        assert(false);  // Should not occur among "primary" tables
        // Do nothing
        tableno += pos->n_sj_tables;
        break;

      case SJ_OPT_LOOSE_SCAN: {
        assert(tab_in_sj_nest);  // First table must be inner
        /* We jump from the last table to the first one */
        tab->match_tab = last_sj_tab->idx();

        /* For LooseScan, duplicate elimination is based on rows being sorted
           on key. We need to make sure that range select keeps the sorted index
           order. (When using MRR it may not.)
           Note: need_sorted_output() implementations for range select classes
           that do not support sorted output, will trigger an assert. This
           should not happen since LooseScan strategy is only picked if sorted
           output is supported. */
        if (tab->range_scan()) {
          assert(used_index(tab->range_scan()) == pos->loosescan_key);
          set_need_sorted_output(tab->range_scan());   // ★ 顺序性要求落在这里
        }

        const uint keyno = pos->loosescan_key;
        assert(tab->keys().is_set(keyno));
        tab->set_index(keyno);

        /* Calculate key length */
        uint keylen = 0;
        for (uint kp = 0; kp < pos->loosescan_parts; kp++)
          keylen += tab->table()->key_info[keyno].key_part[kp].store_length;
        tab->loosescan_key_len = keylen;   // ★ 执行期 key_cmp 只比这 keylen 个字节

        if (pos->n_sj_tables > 1) {
          // ★ 多表 LooseScan：只有首表做索引去重，其余退化为 FirstMatch
          last_sj_tab->firstmatch_return = tab->idx();
          last_sj_tab->match_tab = last_sj_tab->idx();
        }
        tableno += pos->n_sj_tables;
        break;
      }

      case SJ_OPT_DUPS_WEEDOUT: {
        assert(tab_in_sj_nest);  // First table must be inner
        /*
          Consider a semijoin of one outer and one inner table, both
          with two rows. The inner table is assumed to be confluent ...
          If normal nested loop execution is used, we do not need to
          include semi-join outer table rowids in the duplicate
          weedout temp table since NL guarantees that outer table rows
          are encountered only consecutively and because all rows in
          the temp table are deleted for every new outer table
          combination ...
          Now consider the join buffering algorithms (BNL/BKA). These
          algorithms join each inner row with outer rows in "reverse"
          order compared to NL. Effectively, this means that outer
          table rows may be encountered multiple times in a
          non-consecutive manner:
            NL:                 BNL/BKA:
            ot1.row1|it1.row1   ot1.row1|it1.row1
            ot1.row1|it1.row2   ot1.row2|it1.row1
            ot1.row2|it1.row1   ot1.row1|it1.row2
            ot1.row2|it1.row2   ot1.row2|it1.row2
          It is clear from the above that there is no place we can
          empty the temp table like we do in NL to avoid storing outer
          table rowids.
          Below we check if join buffering might be used. If so, set
          first_table to the first non-constant table so that outer
          table rowids are included in the temp table. Do not destroy
          other duplicate elimination methods.
        */
        uint first_table = tableno;
        for (uint sj_tableno = tableno; sj_tableno < tableno + pos->n_sj_tables;
             sj_tableno++) {
          if (join->best_ref[sj_tableno]->use_join_cache() &&
              sj_tableno <= no_jbuf_after) {
            /* Join buffering will probably be used */
            first_table = join->const_tables;
            // ★ 用了 join buffer ⇒ 必须把外表 rowid 也写进去重临时表
            break;
          }
        }
        ...   // 建 rowid 临时表（见 5.3）
        tableno += pos->n_sj_tables;
        break;
      }

      case SJ_OPT_FIRST_MATCH: {
        /*
          Setup a "jump" from the last table in the range of inner tables
          to the last outer table before the inner tables.
        */
        plan_idx jump_to = tab->idx() - 1;
        assert(tab_in_sj_nest);  // First table must be inner
        for (QEP_TAB *tab_in_range = tab; tab_in_range <= last_sj_tab;
             tab_in_range++) {
          if (!join->best_ref[tab_in_range->idx()]->emb_sj_nest) {
            /*
              Let last non-correlated table be jump target for
              subsequent inner tables.
            */
            assert(false);  // no "split jump" should exist.
            jump_to = tab_in_range->idx();
          } else {
            /*
              Assign jump target for last table in a consecutive range of
              inner tables.
            */
            if (tab_in_range == last_sj_tab ||
                !join->best_ref[tab_in_range->idx() + 1]->emb_sj_nest) {
              tab_in_range->firstmatch_return = jump_to;   // ★ split jump
              tab_in_range->match_tab = last_sj_tab->idx();
            }
          }
        }
        tableno += pos->n_sj_tables;
        break;
      }
    }
  }
  return false;
}
```

⇒ **两处反直觉**：① 物化策略在这个 switch 里是 `assert(false)`——因为物化 nest 的内表**已经不在 primary tables 里**（只剩 sjm 临时表）；② FirstMatch 的 `assert(false) // no "split jump" should exist.` 说明 8.0 里"交织的 split jump"实际已不可达，只剩"连续内表 range 的单一跳转"。

每种策略产生特定的 QEP_TAB 序列（内核月报 2021/06 的图）：

### 5.0 五种策略的 AccessPath 形态总览（8.0 迭代器化之后）

迭代器化之后，"策略 → QEP_TAB 序列"只是中间态，最终每种策略落成**特定的 AccessPath 树形状**。这是 `ConnectJoins()` 里完成的，与第八章讲的连接树翻译同源。对照表：

| 策略 | AccessPath 形态 | 关键构造函数 | 去重靠什么 |
|---|---|---|---|
| **FirstMatch** | `NestedLoop(JoinType::SEMI)` | `CreateNestedLoopAccessPath(..., JoinType::SEMI)` | SEMI join type：内层"找到第一个匹配就停"（`NestedLoopIterator` 状态机切回 `NEEDS_OUTER_ROW`） |
| **LooseScan（单表）** | 表访问 + `REMOVE_DUPLICATES_ON_INDEX` | `NewRemoveDuplicatesOnIndexAccessPath` | 索引去重节点（见 [`../../08_access_path.md`](../../08_access_path.md)） |
| **LooseScan（多表）** | `NestedLoopSemiJoinWithDuplicateRemoval` | `NewNestedLoopSemiJoinWithDuplicateRemovalAccessPath` | semijoin NestedLoop 与索引去重**合一** |
| **Duplicate Weedout** | 子树 + `WEEDOUT` 节点 | `CreateWeedoutOrLimitAccessPath`（带 `flush_weedout_table`） | 独立 WEEDOUT 节点（内部是 rowid 临时表） |
| **MaterializeLookup** | `MATERIALIZE` + ref 访问 | `NewMaterializeAccessPath`（`MATERIALIZE_SEMIJOIN` 分支递归建 virtual join） | 物化临时表的唯一键 |
| **MaterializeScan** | `MATERIALIZE` + 全扫 | 同上，但物化表上无索引查找 | 物化临时表 |

几个要点：

**① FirstMatch 没有专门的 AccessPath 类型。** 源码里 `join_type = (substructure == SEMIJOIN) ? JoinType::SEMI : JoinType::OUTER`——它退化成"`JoinType::SEMI` 的 NestedLoop"，靠 join type 这个标志让迭代器短路。这是"策略用标志表达、不新增节点类型"的典型。

**② `add_limit_1`：没有左臂的 semijoin 用 LIMIT 1 兜底。** 当整个切片都是 semijoin（例如对 const 表 semijoin，或 outer join 内部的 semijoin）时，`path == nullptr`（没有左臂可接），于是给子树包 `NewLimitOffsetAccessPath(limit=1)`——用 LIMIT 1 实现"找到即停"。源码注释明说：*"If the entire slice is a semijoin ... solve it by using LIMIT 1."*

**③ 多表 LooseScan 只有首表是 LooseScan，其余退化为 FirstMatch。** 关键注释：

```
LooseScan against multiple tables always puts the non-first tables in
FirstMatch.
```

也就是说 `FindSubstructure()` 返回 `SEMIJOIN` 时，`NewNestedLoopSemiJoinWithDuplicateRemovalAccessPath` 只对首表做索引去重，其余表靠 FirstMatch 语义（`firstmatch_return` 标记）。若这个 semijoin 恰好被 outer join 覆盖，则去重要**放到 join 之后**（`remove_duplicates_loose_scan = true`），注释说这是"安全选项，且不更慢，因为反正会插 LIMIT 1"。

**④ Weedout 的代价是"瞎抄的"。** `Substructure::WEEDOUT` 分支里，`CreateWeedoutOrLimitAccessPath` 之后有一句 *"Copy costs (even though it makes no sense for the LIMIT 1 case)"*——即 WEEDOUT 节点的代价直接从子树复制，作者自己都知道这不精确。

**⑤ Materialize 在"另一个世界"里递归建树。** `MATERIALIZE_SEMIJOIN` 分支（在 `GetTableAccessPath` 里）递归调用 `ConnectJoins()` 把 semi-join 内层当成一个**独立的 virtual join** 建树，再包 `NewMaterializeAccessPath`（详见 [`../../08_access_path.md`](../../08_access_path.md) 的"GetTableAccessPath 五分支"）。物化之后，外层用 ref 访问物化临时表（Lookup）或全扫（Scan），这就是两种 Materialize 策略的分野。

### 5.1 FirstMatch：split jump

```
(ot|nt)*  [ it ((it|nt)* it) ]  (nt)*
```

内表 range 内可以有 nt 表。执行时用 "split jump"：内层找到匹配后 `firstmatch_return` 跳回 range 起始的外表下一行，保证内表不重复行：

```
ot -> [it1 -> nt1 -> it3]
1  ->  1   ->  1    ->  1    <- jump（回 ot 读下一行）
2  ->  1   ->  1
2  ->  2   ->  1    ->  1    <- jump
```

`NestedLoopIterator` 的 `JoinType::SEMI`：找到一行匹配后状态切回 `NEEDS_OUTER_ROW`，实现短路。

### 5.2 LooseScan：loosescan_buf

```
(ot|ct|nt) [ loosescan_tbl (ot|nt|it)* it ]  (ot|nt)*
```

所有相关外表（ct）必须在内表前（loosescan 对内表去重，相关外表会决定内表内容，必须先去重）。执行：

- `loosescan_tbl->loosescan_buf` / `loosescan_key_len`：保存 loose scan keypart 的 buffer + 长度
- 读内表一行 → `key_cmp` 比较索引 key，相同则跳过（skip 重复 index entry）
- 后续 range 用 FirstMatch 机制

### 5.3 DuplicateWeedout：rowid 临时表

```
(ot|nt)*  [ it ((it|ot|nt)* (it|ot))]  (nt)*
```

执行（`do_sj_dups_weedout`，`sql_executor.cc:3386`）：

1. 收集参与去重的各表 `position()` 得到的 rowid
2. 拼接成一个 tuple 写入临时表
3. **key 长度分界**（内核月报 2020/07 细节）：`jt_rowid_offset + null_bytes > 512` → 建 `Field_longlong` 的 hash_field；`≤ 512` → 建 `Field_varstring` 的 rowids key
4. `ha_write_row` 写入，重复则丢弃该行

**join buffer 的影响**（内核月报 2021/06 强调）：range 内没用 join buffer 时，可用 `have_confluent_row` 标记简化（prefix 每次来的是新行才处理）；用了 join buffer 则无法保证，只能把整个 range 的所有 rowid 都作为 distinct key。

### 5.4 Materialize：物化钩子

物化策略的 inner table 不在 primary tables 中（只有 sjm table）。执行时 sjm 对应的 QEP_TAB 开始执行时，先 `prepare_scan` → `join_materialize_semijoin` 完成内表物化 + 去重，后续无需特殊处理：

- **MaterializeLookup**：`outer table → temporary table`（外层每行 ref 查找）
- **MaterializeScan**：`temporary table → outer table`（物化表全扫）

---

## 七、五种策略的 JOIN ORDER 矩阵

内核月报 2024/06《Semijoin 丛林小道全览》总结（用 `ct1,ct2` 外表 + `it1,it2` 内表，`WHERE ct1.a IN (SELECT it1.a FROM it1,it2 WHERE it2.b=ct2.b)`）：

| 策略 | JOIN ORDER 支持 |
|------|----------------|
| **DuplicateWeedout** | 最通用，几乎任意 join order |
| **MaterializeScan** | 内表整体物化后，外表可任意调整，物化表可在外表前/后 |
| **MaterializeLookup** | 所有 sj 依赖外表必须先于内表 |
| **FirstMatch** | 依赖外表在内表前，内表连续，不穿插外表 |
| **LooseScan** | 要求 loose scan 内表后至少一个关联外表，内表连续，索引列覆盖后续 sj 条件 |

结论：DuplicateWeedout 最通用，其他策略只适用特定 join order，**组合起来才能给优化器足够的 join order 灵活性**。

---

## 八、optimizer_switch 与 Hint

`optimizer_switch` 里 semi-join 相关开关（内核月报 2020/07）：

| 参数 | 作用 |
|------|------|
| `semijoin` | semi-join 总开关 |
| `materialization` | 物化策略 |
| `firstmatch` | FirstMatch |
| `loosescan` | LooseScan |
| `duplicateweedout` | DuplicateWeedout |

Optimizer Hint 指定策略：

```sql
SELECT /*+ SEMIJOIN(DUPSWEEDOUT) */ ... WHERE s_suppkey IN (SELECT ps_suppkey ...)
```

---


## 九、深潜：早停对照、NOT IN 灾难与 antijoin 演进

### 9.1 早停机制的跨算法对照

MySQL 的 semi/anti join 早停（"找到一个匹配就跳下一个外层行"）**散落在执行器各处**，对照如下：

| 执行路径 | 早停实现 | 落点 |
|---|---|---|
| **Nested Loop**（FirstMatch） | `JoinType::SEMI` 状态机：找到匹配后切回 `NEEDS_OUTER_ROW`（split jump） | 本篇 5.1 |
| **Hash Join** | `JoinType::SEMI`：probe 到第一行匹配即返回，去下一行 probe | `05_join_buffer` / hash_join_iterator |
| **LooseScan** | 索引 group 跳跃（天然跳过重复，不用"找到再跳"） | 本篇 5.2 |
| **DuplicateWeedout** | **不是早停**——先按内连接跑全，再 rowid 去重 | 本篇 5.3 |

**与 PG 的对照（为什么两边长得不一样）**：

PG 侧（《PostgreSQL 子查询优化（七）》的核心结论）：NL/Hash/Merge 三种算法的早停**源码几乎完全相同**（`nl_NeedNewOuter` / `HJ_NEED_NEW_OUTER` / `EXEC_MJ_NEXTOUTER`，仅状态变量名不同），是**刻意的一致性设计**——因为 PG 的 SEMI/ANTI 是**一种显式 join 节点**（`JOIN_SEMI`/`JOIN_ANTI`），早停自然收拢在 join 节点层。

MySQL 侧恰好相反：**没有统一的 semi/anti join 节点**，早停是**各策略各自实现**的（NL 状态机 / hash 的 probe 短路 / LooseScan 的索引跳跃）。这正是"五种策略"架构的直接后果——**策略分散，早停也分散**。

另外 MySQL **没有 Merge Join**（`AccessPath` 枚举只有 `NESTED_LOOP_JOIN`/`BKA_JOIN`/`HASH_JOIN`），所以"三算法对照"在 MySQL 只剩"两算法 + 索引跳跃"。

### 9.2 NOT IN 的灾难：为什么永远优先 NOT EXISTS

NULL **语义**部分见 [`02_subquery.md`](02_subquery.md)（`abort_on_null`、null_problem 导致 NOT IN 拿不到 antijoin）。这里讲**性能**。

| 写法 | PG | MySQL | 复杂度 |
|---|---|---|---|
| `NOT EXISTS` | ANTI JOIN（显式节点） | antijoin（FirstMatch/Weedout 等） | O(LHS + RHS) |
| `NOT IN`（无 NULL 保护） | **SubPlan**：物化 RHS + 每行全表扫 | **物化子查询**：物化 RHS + 每行探查 | **O(LHS × RHS)** |
| `LEFT JOIN ... IS NULL` | 自动转 ANTI JOIN（`reduce_outer_joins`） | aj-nest 收尾就是 `LEFT JOIN + IS NULL`（本篇 3.3） | O(LHS + RHS) |

两边结论一致：**NOT IN 在含 NULL 时是灾难，永远优先 NOT EXISTS**。区别只在"灾难路径"的名字：PG 叫 SubPlan，MySQL 叫物化子查询（见 [`../../runtime/08_materialization.md`](../../runtime/08_materialization.md)）。

### 9.3 antijoin 的版本演进

- **MySQL 8.0.17 起**支持 antijoin 优化（NOT IN / NOT EXISTS → 反连接）。此前只能走物化子查询（每行探查）。⚠️ 版本号来自官方 Release Notes，**源码无版本痕迹**（与 CTE 演进同理）
- antijoin 是 **semi-join 框架的"负模式"**（aj-nest，见本篇 3.3 的 NOT IN 表格），不是独立的 join 节点
- **策略受限**：antijoin 会强制砍掉 LooseScan（`is_aj_nest()` 只允许 FirstMatch / Materialization / DupsWeedout，见本篇参数节）
- **NULL 保护的必要性**：NOT IN 走 antijoin 的前提是子查询结果不含 NULL 或列不可空，否则退回物化（02 篇的 null_problem 分析）

对比 PG：PG 的 ANTI JOIN 是显式 join 节点（`JOIN_ANTI`），MySQL 是半连接框架里的负模式——又一个"节点 vs 策略"的架构差异。

### 9.4 节点 vs 策略：与 PG 的架构全面对照

#### 先纠正一个常见误解：PG 的 SEMI JOIN 不是"只有一种实现"

PG 的 `JOIN_SEMI` 是**逻辑节点**，节点之下有三种物理实现竞争——**Hash Semi Join / Merge Semi Join / Nested Loop Semi Join**（回归测试的 EXPLAIN 里就能看到这三种名字）。优化器为同一节点生成多条 path，代价选择。

所以"PG 只有一种、MySQL 有五种"是对比**错了维度**：

| 层 | PG | MySQL（旧优化器） | MySQL（hypergraph） |
|---|---|---|---|
| 逻辑层 | `JOIN_SEMI`/`JOIN_ANTI` 节点 | semi-join 框架（aj-nest 负模式） | `RelationalExpression::SEMIJOIN/ANTIJOIN` |
| **算法维度** | Hash / Merge / NL **三算法竞争** | NL + hash（8.0.18 起，**无 Merge**） | Hash / NL 竞争（`ProposeHashJoin` / `ProposeNestedLoopJoin`） |
| **去重维度** | 节点内置（找到即跳，写一处） | **五策略**（FM/LS/Weedout/Mat×2） | `DeduplicateForSemijoin`（LIMIT 1 / REMOVE_DUPLICATES） |
| 执行器 | 节点直接执行 | `JoinType::SEMI/ANTI`（NL/Hash 迭代器分发） | 同左 |
| 开关 | `enable_nestloop/hashjoin/mergejoin`（**逐算法**） | `semijoin/firstmatch/loosescan/...`（**逐策略**）；`hash_join` 开关**已失效** | 仅 `hypergraph_optimizer` 总开关 |

#### 源码揭示的三个真相

**① 执行器层 MySQL 其实有显式 SEMI/ANTI**

`sql/join_type.h`：

```cpp
enum class JoinType { INNER, OUTER, ANTI, SEMI, FULL_OUTER };
```

`HashJoinIterator` 与 `NestedLoopIterator` 都按它分发行为（SEMI 找到即跳、ANTI 见行即弃）。所以"MySQL 无显式节点"**只对旧优化器层成立**——执行器层与 PG 的节点模型对得上。

**② hash semi join 不是第六种策略，与五策略正交**

`advance_sj_state` 全文**没有任何 hash join 分支**；`SJ_OPT_*` 枚举只有 6 值（含 NONE，无 HASH）。hash semi join 发生在**策略定稿之后**的执行准备阶段：

```
五策略（advance_sj_state → fix_semijoin_strategies）：决定"去重方式"
      ↓ 定稿
ConnectJoins() → CreateHashJoinAccessPath()：决定"连接算法"（NL 或 hash）
```

即：**五策略管"怎么去重"，hash join 管"怎么连接"**——两个正交维度。这也解释了为什么 `semijoin` 策略开关与 hash join 互不干扰。

**③ hypergraph 优化器完全不读五策略**

`sj_strategy`、`SJ_OPT_*`、五策略开关在 `sql/join_optimizer/` **零引用**。hypergraph 把 semi join 表达为 `RelationalExpression::SEMIJOIN`（与 `JoinType::SEMI` **同值**），代价模型同时提议 Hash/NL 两种候选，去重用 `DeduplicateForSemijoin`。

**所以 MySQL 的新优化器其实已经走向 PG 式的"节点 + 算法竞争"模型**；五策略是旧优化器专属遗产。

#### "谁更优"的准确回答

**不是"策略多 = 更优"**：

1. **MySQL 五策略是"NL-only 年代"的去重补丁**——因为 8.0.18 前没有 hash join，只能在 NL 框架内用各种 trick 做去重（每种策略都有严格前提：LooseScan 要索引、Materialize 要临时表、Weedout 零约束但重复行真的产生）。它的"丰富"恰恰暴露了当年的短板。
2. **PG 的 Hash Semi Join 在多数场景下不逊于、甚至优于 MySQL 五策略**——O(1) 探测、可并行、去重建在算子内。
3. **但 MySQL 确有 PG 没有的独特维度**：LooseScan 的索引跳跃（PG 的 Merge Semi Join 需要两个有序输入，LooseScan 只要一个索引）、FirstMatch 的零内存、以及**逐策略开关**的 DBA 可调性。
4. **准确的表述**：两者在"物理实现多样性"上其实相当（PG 3 算法 × 1 去重 vs MySQL 2 算法 × 6 去重策略），但**多样性的维度不同**——PG 在算法维度（源自设计），MySQL 在去重维度（源自历史）。

#### 三层三套抽象（旧优化器路径的代价）

同一句"t1 SEMI JOIN t2"在旧优化器路径是**三个不互通的类型系统**：

```
优化器层：SJ_OPT_LOOSE_SCAN 等策略状态机（附着在 join 顺序搜索上）
   ↓ 执行准备期二次解码（ConnectJoins / FindSubstructure）
AccessPath 层：NESTED_LOOP_SEMIJOIN_WITH_DUPLICATE_REMOVAL 等节点
   ↓
迭代器层：JoinType::SEMI / NestedLoopSemiJoinWithDuplicateRemovalIterator
```

而 hypergraph 路径消除了"策略→节点"的翻译（`RelationalExpression` 与 `JoinType` **共享枚举值**）。PG 则从头到尾只有一棵 plan node 树——**这是 PG 架构统一性的真正优势**：语义在逻辑层定一次，物理层只竞争算法。

### 9.5 已知正确性缺陷：EXISTS 转 semi-join（Bug #110819）

这是本目录记录的**第一个正确性 bug**（而非性能问题），值得单独记住。

#### 现象

数据：3 行，全部是 `("a", 1)`，id 为 1/2/3：

```sql
SELECT id FROM bug_t t1 WHERE EXISTS (
  SELECT 1 FROM bug_t t2
  WHERE t1.id > t2.id                  -- 相关、非等值
    AND t1.col_str = t2.col_str        -- 等值
    AND t1.col_int = t2.col_int);
-- 期望 {2, 3}；实际只返回 {2}
```

#### 关键事实

| 项 | 值 |
|---|---|
| Bug 号 | **#110819** |
| 标题 | EXISTS with dependent subquery can return incorrect results |
| 严重性 | **S2 (Serious)** |
| 状态 | **Verified**（2023-04 提交） |
| 影响版本 | **8.0.16+**（8.0.15 及以下不复现） |
| 标签 | **regression**（回归） |
| 规避方法 | `SET optimizer_switch='semijoin=off';` |

#### 根因：WL#4389「Transform EXISTS subqueries to semi-join」

官方 bisect 定位到首个坏提交（Roy Lyseng，2018-11）：

> **WL#4389** Transform EXISTS subqueries to semi-join — *Extend semi-join check to accept EXISTS subqueries in addition to IN. Filter out non-deterministic subqueries.*

即：**8.0.16 把 semi-join 转换的适用范围从 `IN` 扩展到了 `EXISTS`**——这正是本篇"四种 SQL 等价变换形式"里 EXISTS 那一条的由来。这个扩展引入了正确性回归。

#### Bug #28805105 **不是**成因（源码已推翻报告者的推断）

8.0.16 changelog 里那条：

> ...but because the subquery is not correlated... we use **two equal constant items as keys**, to ensure that the materialized query gets the constant as a key (and so that the materialized table consists of at most one row). (Bug #28805105)

报告者怀疑它导致本 bug。**源码证明不成立**，两道独立排除：

**排除一：补丁根本不触发**。它在 `Query_block::build_sj_cond()` 里，唯一触发条件是 `sj_inner_exprs` 为空：

```cpp
if (nested_join->sj_inner_exprs.empty()) {
  Item *const_item = new Item_int(1);
  nested_join->sj_inner_exprs.push_back(const_item);   // "两个相等常量"= 同一个 Item_int(1) 塞两边
  nested_join->sj_outer_exprs.push_back(const_item);
}
```

本例两条等值被去相关塞进 `sj_inner_exprs = (t2.col_str, t2.col_int)`，**非空** ⇒ 不触发。

**排除二：物化被独立关掉**。即使触发，`sql/sql_optimizer.cc` 的 `optimize_semijoin_nests_for_materialization()` 有：

```cpp
if (sj_nest->nested_join->sj_corr_tables) continue;   // 相关 semi-join 不做物化
```

本例 `sj_corr_tables = {t1}` ≠ 0 ⇒ 物化整条路径出局。

**最硬的证据**：`JOIN::setup_semijoin_materialized_table()` 用 `sj_inner_exprs` 建物化表，本例列只有 `(col_str, col_int)` —— **物化表里根本没有 `t2.id`**，`t1.id > t2.id` 在其上无法求值。所以物化不可能产生 `{2}`（若忽略残差得 `{1,2,3}`；若 t2.id 固定为 X 得 `{X+1..3}`，无解为 `{2}`）。

#### 真正的机制：LooseScan 的"命中即跳组"

转换后三个条件命运不同（关键）：

| 条件 | 处理 |
|---|---|
| `t1.col_str = t2.col_str` | **去相关** → 进 `sj_inner_exprs`（成为 key） |
| `t1.col_int = t2.col_int` | **去相关** → 进 `sj_inner_exprs`（成为 key） |
| `t1.id > t2.id` | **不去相关**：`can_decorrelate_operator()` 对 `GT_FUNC` 返回 false（semi-join 路径 `op_types == nullptr` 只去相关 `=`）→ **残留**在 WHERE |

LooseScan 逐行推演（`col_str` 索引跳组）：

1. `'a'` 只有**一组**；组内按二级索引 `(col_str, PK)` 序为 t2.id = 1, 2, 3
2. LooseScan **每组只取第一行** → **t2.id = 1**
3. `t1.id=2`：`2 > 1` ✓ → 输出 **2**，随即**跳过整组**（`tab->match_tab = last_sj_tab->idx()`）
4. ⇒ **t2.id=2、t2.id=3 永远不会被取到**，`t1.id=3` 唯一能成立的路径（需 `3 > 2`）从未被探测 → **丢失**

> 本质：LooseScan 把作用在**组内非分组列 `t2.id`** 上的相关非等值条件，误当成"只依赖分组键 `col_str`"的条件。

#### 一处精妙的护栏（也是未完全确定的点）

LooseScan 准入条件 (5) 与 (6) 对"t1 是否在剩余表"的要求**互斥**：

```cpp
!(remaining_tables_incl & sj_corr_tables) &&   // (5) 要求 t1 已进前缀
(remaining_tables_incl  & sj_depends_on)       // (6) 要求 t1 仍在剩余
```

本例 `sj_corr_tables == sj_depends_on == {t1}` ⇒ 两者不可能同时成立 ⇒ **LooseScan 应被排除**。护栏意图见源码注释：*"All non-IN-equality correlation references from this sj-nest are bound"*。

于是**静态推演的结果是 8.0.39 应返回正确的 `{2,3}`**，与 bug 报告（8.0.16+ 复现）**矛盾**。矛盾焦点收敛到一个运行期量：`sj_corr_tables = sj_cond->used_tables() & outer_tables_map`，它受 `Item_cond::used_tables_cache` 是否刷新影响——**此层无法仅凭读源码定死，不做断言**。

**一锤定音的方法**：`EXPLAIN` + `SET optimizer_trace='enabled=on'`，看 `semijoin_strategy_choice` 与 `final_semijoin_strategy`：
- 显示 **LooseScan** → 坐实上述机制（护栏失效，`sj_corr_tables` 实为 0）
- 显示 FirstMatch / DuplicateWeedout 且结果仍错 → 问题在别处

> 另：FirstMatch（仅 t1→t2 顺序可用）与 DuplicateWeedout 的逐行推演均为正确的 `{2,3}`。

#### 为什么值得记

1. **semi-join 是"改写"，不是"等价保证"**——它是一个**有正确性边界的优化**。加了 `semijoin` 转换，就可能引入结果错误（不同于代价估算错误只影响性能）。
2. **EXISTS 的转换比 IN 晚、也更危险**：IN→semi-join 从 5.6 就有，EXISTS→semi-join 是 **8.0.16 才引入**（WL#4389），所以它是"新且未经充分验证"的那一块。
3. **排查信号**：遇到 `EXISTS` + 相关子查询 + 结果行数偏少，优先试 `semijoin=off` 验证；若规避后正确，即命中此 bug。

> ⚠️ 本仓库为 8.0.39，该 bug 报告的影响范围标注为 "8.0.16+, 8.0.33"，且报告未显示已修复版本——**8.0.39 很可能仍受影响**，实际遇到时请以复现为准。

---

## 参考

**论文**
- **Galindo-Legaria《Parameterized Queries and Nesting Equivalences》(2001)** —— semi-join 转换的理论基础

**官方文档**
- *MySQL 8.0 Reference Manual → Semi-join Transformations*

**内核月报**
- **2024/06《Semijoin 丛林小道全览》** —— 五种策略与 JOIN ORDER 矩阵
- **2021/06《Semi-join 优化与执行逻辑》**
- **2020/07《semi-join 四个执行 strategy》** —— 四种策略（DuplicateWeedout / FirstMatch / LooseScan / Materialize）与 PolarDB 并行加速

**阿里云 PolarDB 官方文档**
- [《图解 MySQL 8.0 优化器对子查询 JOIN 与分区表的转换优化》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/optimizer-based-query-conversion-in-mysql-8) —— `convert_subquery_to_semijoin` 四种模式（IN/EXISTS/NOT EXISTS/NOT IN）、`flatten_subqueries` 优先级、anti-join 的 `can_do_aj`（本篇 3.1/3.3 对齐此文）

**已知缺陷**
- [MySQL Bug #110819](https://bugs.mysql.com/bug.php?id=110819) —— *EXISTS with dependent subquery can return incorrect results*（8.0.16+ 回归，WL#4389 引入，S2，见 9.5）


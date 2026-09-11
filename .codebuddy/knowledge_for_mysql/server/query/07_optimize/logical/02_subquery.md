# 02 子查询改写：决策树与 IN2EXISTS / 物化 / anti-join / MIN-MAX

> 子查询**改写决策**（prepare/optimize 阶段如何等价变换）。semi-join 已独立成章（03 篇），本篇覆盖其余改写路径。运行期执行见 [`../../runtime/02_subquery_runtime.md`](../../runtime/02_subquery_runtime.md)。

## 目录

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

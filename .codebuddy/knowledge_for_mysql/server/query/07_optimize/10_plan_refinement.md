# 10 计划改进（Plan Refinement）

> 物理优化**之后**的收尾：Ordering index 选择（避免排序）、条件最终化、访问方法 setup、引擎下推、临时表与排序。
>
> 分工：**ICP 的 SQL 层决策**在本篇，**ICP 的 handler 接口与 InnoDB 实现**在 [05 篇 4 节](logical/05_logical_predicate.md)。
>
> ★ 相邻主题：GROUP BY / DISTINCT 在优化期被改写的**判定过程**（本篇只讲"排序怎么选"）见 [15 篇](15_groupby_distinct_order.md)；`QEP_TAB` / `POSITION` 等结构的字段级剖析与生命周期见 [16 篇](16_join_object_model.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [零、执行顺序与三条依赖链](#零执行顺序与三条依赖链)
- [一、Ordering index 选择（避免排序）](#一ordering-index-选择避免排序)
- [二、条件最终化与访问方法 setup](#二条件最终化与访问方法-setup)
- [三、push_to_engines：两层下推](#三push_to_engines两层下推)
- [四、setup_join_buffering](#四setup_join_buffering)
- [五、临时表与排序 make_tmp_tables_info](#五临时表与排序-make_tmp_tables_info)

---

## 设计思想与理论基础

### "计划精化"是什么

join order 定下来（`make_join_plan`）之后，计划还不能直接执行——仍有一堆收尾工作：条件最终化、访问方法 setup（定索引 / keyread / ICP）、有序索引选择、join buffer 决策、临时表与 filesort 安排、引擎下推。

这一整段在 `JOIN::optimize()` 的后半段，本篇称之为**计划精化（plan refinement）**。

**为什么它必须是一个有严格顺序的独立阶段？** 因为这些东西之间存在**语义依赖**——不是随意编排，而是被依赖关系逼出来的。本篇"三条关键依赖链"就是这份约束清单。

### 核心难点：为什么顺序不能调

1. **条件归属 → ORDER/GROUP 化简 → join buffer → 有序索引**
   — 关键：**join buffer 与有序索引互斥**，所以 buffering 决策必须早于 `test_skip_sort`
2. **alloc_qep → 定索引 → 裁条件 → keyread/ICP**
   — 关键：**ICP 必须在条件裁剪之后**，否则下推的可能是马上要被删掉的冗余谓词
3. **临时表 / Filesort → AccessPath → 引擎改写**
   — 关键：中间不能插入 Iterator 创建

### 为什么叫"精化"而不是"优化"

关键区别：**这一步大多不再做代价比较**，而是**确定性的、规则驱动的收尾**：

| 步骤 | 性质 |
|---|---|
| `test_skip_sort`（有序索引免排序） | 规则：检查第一张非 const 表能否用索引免排序 |
| `finalize_table_conditions` | 规则：把条件下推到能求值的最早位置 |
| keyread / ICP | 规则：能用就用 |

对比：join order 搜索是**代价驱动**（06 篇），访问方法选择部分代价驱动（07 篇）。

> 所以优化器整体是"**代价搜索 + 规则收尾**"两段式，本篇属于后一段。这也是 00 篇所说"MySQL 是 CBO + 规则混合"的又一处具体体现。

### 一个贯穿的设计张力：有序性 vs 批处理

`test_skip_sort`（用有序索引避免排序）与 `setup_join_buffering`（用 join buffer 批处理）**互斥**。

理论解释：join buffer 会**打乱行的到达顺序**（批量读 + 批量匹配），而"用有序索引免排序"要求行**按索引顺序流出**。两者对"行的顺序"要求相反，因此不能同时用。

这是优化器里典型的**互斥权衡**：**省排序**（用有序索引）vs **省随机 IO**（用批处理）。

### 他库对比

PostgreSQL 没有这样独立的"精化"阶段——它的计划生成与代价估算更一体化，等价工作分散在 `create_plan` / `set_plan_references` 等步骤中。MySQL 的这个阶段更多是**历史产物**：早期优化器结构（QEP_TAB 数组 + 函数指针）决定了收尾工作必须按固定顺序"就地修改"计划结构。

---

## 零、执行顺序与三条依赖链

**先看一份"与常见说法不符"的纠正表**（5.7 口径在 8.0 大量改号/改名/删函数）：

| 常见说法 | 8.0.39 真相 |
|---|---|
| `choose_plan()` 在 `sql/sql_planner.cc` | **已删除**；内容被摊进 `JOIN::make_join_plan()`，join order 由 `Optimize_table_order::choose_table_order()` 负责 |
| `make_join_readinfo()` 在 `sql/sql_executor.cc` | 在 **`sql/sql_select.cc:3282`** |
| `make_tmp_tables_info()` 是自由函数 | 是 **`JOIN::make_tmp_tables_info()`**，`sql/sql_select.cc:4354` |
| `test_if_skip_sort_order()` 改名成 `is_ordering_satisfied_by_key()` | **没有这个函数**；仍是 `test_if_skip_sort_order()`（`sql_optimizer.cc:2193`）+ `JOIN::test_skip_sort()`（`:1642`） |
| `create_root_access_path()` | 改名 **`JOIN::create_root_access_path_for_join()`**，`sql/sql_executor.cc:2970` |
| `ConnectJoins()` 在 `join_optimizer/` | 在 **`sql/sql_executor.cc:2356`**；`join_optimizer/` 下的是 hypergraph 的 `FindBestQueryPlan()`，两套并行实现 |
| join buffer 用 `calc_used_blob_length()` | **已删除**；BNL 在运行期被 hash join 取代，只有 BKA 还保留真正 join buffer（`BKAIterator`） |
| `JOIN_CACHE` 是带 `init()/put_record()` 的大类 | **退化成只有一个 enum 的空壳**（`sql/sql_join_buffer.h:27`） |
| `Window::setup_windows()` | 是 **`setup_windows2()`**（`sql/window.cc:1308`），几乎什么都不做；refinement 阶段的窗口临时表全在 `make_tmp_tables_info()` |

**`JOIN::optimize()` 中 `make_join_plan()` 之后的完整源码**（`sql/sql_optimizer.cc:585-1105`，按执行顺序，略去纯错误清理）：

```cpp
  // ...（前置：常量折叠 / 零结果早退 / derived / optimize_aggregated_query）
  if (thd->lex->using_hypergraph_optimizer()) {
    // ★ hypergraph 走 FindBestQueryPlan() 后直接 return —— 完全不进入本主题的任何代码
  }

  assert(!thd->lex->using_hypergraph_optimizer());

  // Set up join order and initial access paths
  THD_STAGE_INFO(thd, stage_statistics);
  if (make_join_plan()) {                       // ★① 选 join order + 访问方法（含 get_best_combination）
    if (thd->killed) thd->send_kill_message();
    return true;
  }

  ASSERT_BEST_REF_IN_JOIN_ORDER(this);

  if (zero_result_cause != nullptr) {           // Can be set by make_join_plan()
    create_access_paths_for_zero_rows();
    goto setup_subq_exit;
  }

  if (rollup_state == RollupState::NONE) {
    /* Remove distinct if only const tables */
    select_distinct &= !plan_is_const();
  }
  ...
  /*
    Among the equal fields belonging to the same multiple equality
    choose the one that is to be retrieved first and substitute
    all references to these in where condition for a reference for
    the selected field.
  */
  if (where_cond) {
    where_cond =
        substitute_for_best_equal_field(thd, where_cond, cond_equal, map2table);   // ★②
    ...
  }

  /* Perform the same optimization on field evaluation for all join conditions */
  for (uint i = const_tables; i < tables; ++i) {
    JOIN_TAB *const tab = best_ref[i];
    if (tab->position() && tab->join_cond()) {
      tab->set_join_cond(substitute_for_best_equal_field(
          thd, tab->join_cond(), tab->cond_equal, map2table));
      ...
    }
  }

  if (init_ref_access()) { error = 1; return true; }

  // Update table dependencies after assigning ref access fields
  update_depend_map();

  THD_STAGE_INFO(thd, stage_preparing);

  if (make_join_query_block(this, where_cond)) {   // ★③ 读常量表 + 二次 range 重估
    if (thd->is_error()) return true;
    zero_result_cause = "Impossible WHERE noticed after reading const tables";
    create_access_paths_for_zero_rows();
    goto setup_subq_exit;
  }

  error = -1; /* if goto err */

  if (optimize_distinct_group_order()) return true;   // ★④ DISTINCT/GROUP/ORDER 合并处理

  // ★ join buffer 总开关：SQL_NO_CACHE / FULLTEXT 一律禁用
  if ((query_block->active_options() & SELECT_NO_JOIN_CACHE) ||
      query_block->ftfunc_list->elements)
    no_jbuf_after = 0;

  if (query_block->has_ft_funcs() && optimize_fts_query()) return true;

  /* ... HAVING 的 const 部分化简、cast 注入、replace_index_subquery() ... */

  {
    /*
      If the hint FORCE INDEX FOR ORDER BY/GROUP BY is used for the first
      table (it does not make sense for other tables) then we cannot do join
      buffering.
    */
    if (!plan_is_const()) {
      const TABLE *const first = best_ref[const_tables]->table();
      if ((first->force_index_order && !order.empty()) ||
          (first->force_index_group && !group_list.empty()))
        no_jbuf_after = 0;
    }

    bool simple_sort = true;
    Table_map_restorer deps_lateral(&deps_of_remaining_lateral_derived_tables);
    // Check whether join cache could be used
    for (uint i = const_tables; i < tables; i++) {          // ★⑤ join buffer 决策循环
      JOIN_TAB *const tab = best_ref[i];
      if (!tab->position()) continue;
      if (setup_join_buffering(tab, this, no_jbuf_after)) return true;
      if (tab->use_join_cache() != JOIN_CACHE::ALG_NONE) simple_sort = false;
      assert(tab->type() != JT_FT ||
             tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
      ...
    }
    if (!simple_sort) {
      /*
        A join buffer is used for this table. We here inform the optimizer
        that it should not rely on rows of the first non-const table being in
        order thanks to an index scan; indeed join buffering of the present
        table subsequently changes the order of rows.
      */
      simple_order = simple_group = false;   // ★★ 任一表用 buffer ⇒ 整条"首表有序"假设作废
    }
  }

  if (!plan_is_const() && !order.empty()) {
    /* Force using of tmp table if sorting by a SP or UDF function ... */
    for (ORDER *tmp_order = order.order; tmp_order; tmp_order = tmp_order->next) {
      Item *item = *tmp_order->item;
      if (item->is_expensive()) {
        simple_order = simple_group = false;   // ★ 排 UDF ⇒ 强制临时表（不排序）
        break;
      }
    }
  }

  /*
    ★⑥ need_tmp_before_win 判定（8 条，见 make_tmp_tables_info 一节）
    (1) ROLLUP + (DISTINCT|windows|ORDER BY)
    (2) !plan_is_const()
    (3) GROUP BY 且 !simple_group
    (4)(5) DISTINCT 或 ORDER BY 且 !simple_order
    (6) 有 GROUP BY 又有 ORDER BY（不看是否各自能用索引！）
    (7) SQL_BUFFER_RESULT
    (8) 首个窗口步骤需要排序且非单表
  */
  if (rollup_state != RollupState::NONE &&  // (1)
      (select_distinct || has_windows || !order.empty()))
    need_tmp_before_win = true;

  if (!need_tmp_before_win && implicit_grouping &&
      contains_non_aggregated_fts()) {
    need_tmp_before_win = true;   // 全文列参与聚合 ⇒ 必须物化
  }

  if (!plan_is_const())  // (2)
  {
    if ((!group_list.empty() && !simple_group) ||                       // (3)
        (!has_windows && (select_distinct ||                            // (4)
                          (!order.empty() && !simple_order) ||          // (5)
                          (!group_list.empty() && !order.empty()))) ||  // (6)
        ((query_block->active_options() & OPTION_BUFFER_RESULT) &&
         !has_windows && !(query_expression()->derived_table &&
                           query_expression()->derived_table->uses_materialization())) ||  // (7)
        (has_windows && (primary_tables - const_tables) > 1 &&          // (8)
         m_windows[0]->needs_sorting() && !group_optimized_away))
      need_tmp_before_win = true;
  }

  DBUG_EXECUTE("info", TEST_join(this););

  if (alloc_qep(tables)) return (error = 1);   // ★⑦ JOIN_TAB → QEP_TAB

  if (!plan_is_const()) {
    // Test if we can use an index instead of sorting
    test_skip_sort();                          // ★⑧ 能否用索引免排序
    if (finalize_table_conditions(thd)) return true;   // ★⑨ 条件最终化
  }

  if (make_join_readinfo(this, no_jbuf_after)) return true;   // ★⑩ keyread / ICP / semijoin 标记

  if (make_tmp_tables_info()) return true;                    // ★⑪ 临时表 + filesort

  /*
    If we decided to not sort after all, update the cost of the JOIN.
  */
  if (sort_cost > 0.0 &&
      !explain_flags.any(ESP_USING_FILESORT, ESC_WINDOWING)) {
    best_read -= sort_cost;   // ★ 才发现不用 filesort ⇒ 把排序代价从 best_read 减回去
    sort_cost = 0.0;
  }

  // ★ make_tmp_tables_info() 可能改过 precomputed_group_by ⇒ 必须重算
  count_field_types(query_block, &tmp_table_param, *fields, false, false);

  create_access_paths();                       // ★⑫ QEP → AccessPath 树

  if (thd->is_error()) return true;            // 建 Iterator 可能求值常量 hash join 条件失败

  /*
    At this stage, we have set up an AccessPath 'plan'. Traverse the
    AccessPath structures and find components which may be offloaded to
    the engines. This process is allowed to modify the AccessPath itself.
    ... Thus no Iterators should be set up until after push_to_engines()
    has completed.
  */
  if (push_to_engines()) return true;          // ★⑬ 下推到引擎（可改 AccessPath）

  // Make plan visible for EXPLAIN
  set_plan_state(PLAN_READY);                  // ★⑭ 冻结

  DEBUG_SYNC(thd, "after_join_optimize");
  return false;

setup_subq_exit:
  // 零结果路径：也要造出 ZeroRows / ZeroRowsAggregated 的 AccessPath
  assert(zero_result_cause != nullptr);
  assert(m_root_access_path != nullptr);
  ...
  if (!qep_tab && best_ref) {
    if (alloc_qep(tables)) return true;
    unplug_join_tabs();
  }
  set_plan_state(ZERO_RESULT);
  return false;
}
```

**调用顺序一句话**：`make_join_plan()` → `substitute_for_best_equal_field` → `make_join_query_block()` → `optimize_distinct_group_order()` → `setup_join_buffering()` 循环 → `need_tmp_before_win` 判定 → `alloc_qep()` → `test_skip_sort()` → `finalize_table_conditions()` → `make_join_readinfo()` → `make_tmp_tables_info()` → `count_field_types()` → `create_access_paths()` → `push_to_engines()`。

| # | 行号 | 调用 | 阶段 |
|---|---|---|---|
| 0 | `:696` | `make_join_plan()` | ③（前置） |
| 1 | `:736-768` | `substitute_for_best_equal_field` | ① |
| 2 | `:770` | `init_ref_access()` | |
| 3 | **`:781`** | **`make_join_query_block`**（条件最终化 + 二次 range 重估） | ③ |
| 4 | **`:796`** | **`optimize_distinct_group_order`** | ① |
| 5 | `:894-906` | `setup_join_buffering` | ③ |
| 6 | `:975-1007` | `need_tmp_before_win` 判定 | |
| 7 | `:1011` | `alloc_qep` | |
| 8 | **`:1015`** | **`test_skip_sort`**（Ordering index） | ④ |
| 9 | **`:1017`** | **`finalize_table_conditions`** | ④ |
| 10 | **`:1020`** | **`make_join_readinfo`**（keyread + ICP） | ④ |
| 11 | **`:1023`** | **`make_tmp_tables_info`** | ④ |
| 12 | `:1037` | `create_access_paths` | 代码生成 |
| 13 | **`:1064`** | **`push_to_engines`** | ④ |
| 14 | `:1067` | `set_plan_state(PLAN_READY)` | 冻结 |

### 三条关键依赖链

1. **条件归属 → ORDER/GROUP 化简 → join buffer → 有序索引**
   `make_join_query_block` → `optimize_distinct_group_order` → `setup_join_buffering` → `test_skip_sort`。
   **join buffer 与有序索引互斥**（见 1.3），所以 buffering 决策必须早于 test_skip_sort。
2. **alloc_qep → 定索引 → 裁条件 → keyread/ICP**
   `alloc_qep` → `test_skip_sort` → `finalize_table_conditions` → `make_join_readinfo`。
   **ICP 必须在条件裁剪之后**，否则下推的可能是马上要被删掉的冗余谓词。
3. **临时表/Filesort → AccessPath → 引擎改写**
   `make_tmp_tables_info` → `create_access_paths` → `push_to_engines`。中间不能插入 Iterator 创建。

---

## 一、Ordering index 选择（避免排序）

核心问题：**用某个索引的天然顺序 vs filesort 排序，哪个便宜？**

### 1.1 test_if_order_by_key（`sql_optimizer.cc:1795`）

判定"索引 idx 的天然顺序能否满足 ORDER/GROUP 列表"。

> ⚠️ **返回值语义纠正常见误解**：返回的是**扫描方向**（`+1` 正向可用 / `-1` 需反向可用 / `0` 不可用），**不是"用了几个 keypart"**。后者通过出参 `*used_key_parts` 返回，且**可能大于** `user_defined_key_parts`（PK 后缀扩展时）。

**逐段算法**：

**① 常量 keypart 跳过**——这就是 `WHERE a=1 ORDER BY a,b` 中 a 能被跳过的原因：

```cpp
key_part_map const_key_parts = table->const_key_parts[idx];
for (; order; order = order->next, const_key_parts >>= 1) {
  ...
  for (; const_key_parts & 1 && key_part < key_part_end &&
         (order_src->is_const_optimized() || key_part->field != field);
       const_key_parts >>= 1) {
    key_part++;
  }
```

- `const_key_parts` 是**位掩码**（`table->const_key_parts[idx]`），第 i 位为 1 表示 keypart i 在 WHERE 中已判定为常量。由 `update_const_equal_items`（`sql_executor.cc:466`）填充。
- 前面的 `JOIN::remove_const()`（`:10175`）已把 ORDER BY 中的常量项删掉并标记 `const_optimized=true`，所以进入本函数时 ORDER 链表通常只剩 `b`。
- **本质**：`a` 在结果集中只有单一取值，索引里 `a` 这一层的排序无意义，可直接"下沉"到 `b` 层——`(a,b)` 索引在 `a=const` 的横截面上就是按 `b` 有序的。

**② 排序项必须是 Item_field**：`real_item()->type() != Item::FIELD_ITEM` → 返回 0。`ORDER BY a+1`、`ORDER BY f(a)` 都失败。

**③ 聚簇主键后缀扩展**（`use_index_extensions`，默认 **on**）：

```cpp
if (key_part == key_part_end) {   // 二级索引 keypart 用完，ORDER 还有剩余
  if (!on_pk_suffix &&
      (table->file->ha_table_flags() & HA_PRIMARY_KEY_IN_READ_INDEX) &&  // InnoDB 设此位
      table->s->primary_key != MAX_KEY && table->s->primary_key != idx) {
    on_pk_suffix = true;
    key_part = table->key_info[table->s->primary_key].key_part;   // 继续用 PK 匹配
```

InnoDB 二级索引叶子隐含 PK 列作为后缀，所以 `INDEX(a)` 实际等价于 `(a, pk1, ...)`——`ORDER BY a, id`（id 为 PK）即使 a 非唯一也能靠扩展部分保证确定性顺序。

**④ 字段匹配与方向判定**：

```cpp
if (key_part->field != field || !field->part_of_sortkey.is_set(idx)) return 0;
if (order->direction != ORDER_NOT_RELEVANT) {
  const enum_order keypart_order =
      (key_part->key_part_flag & HA_REVERSE_SORT) ? ORDER_DESC : ORDER_ASC;
  int cur_scan_dir = (order->direction == keypart_order) ? 1 : -1;
  if (reverse && cur_scan_dir != reverse) return 0;    // ★ 必须同向
  reverse = cur_scan_dir;
}
```

- `part_of_sortkey` 只给**完整 keypart 且引擎支持 `HA_READ_ORDER`** 的列置位（`table.cc:764`）——前缀索引列、R-Tree 不进，故不能用于免排序。
- **一致性检查**：所有排序项必须同向。`ORDER BY a ASC, b DESC` 在单一方向上无法满足（除非用 8.0 的降序索引 `INDEX(a ASC, b DESC)`，此时 `HA_REVERSE_SORT` 让 `b DESC` 反而对应 `cur_scan_dir = +1`）。
- **混合升降序 + 反向 → 禁用 range**：`if (mixed_order && reverse < 0) *skip_quick = true;`——多 range 反向扫描需要重排 range 列表，混合索引无法简单重排。

**⑤ 反向能力校验**：`index_flags(idx, n-1, true) & HA_READ_PREV`。PK 扩展场景要求二级索引和 PK **都**支持。

### 1.2 test_if_cheaper_ordering（`sql_select.cc:5081`）—— 核心

**基准**：`read_time = tab->position()->read_cost`（**当前已选访问方法的代价**）。
`fanout` = 本表之后所有表 `rows_fetched × filter_effect` 之积（本表 1 行在最终结果里膨胀成多少行）。

**select_limit 的三重缩放**（完整语义）：

**① group 分支**——GROUP BY 时输出结果 1 行 = 1 个 group，而产出 1 个 group 平均要读 `rec_per_key(group 列)` 条索引项：

```cpp
if (select_limit > table_records / rec_per_key) select_limit = table_records;
else select_limit = (ha_rows)(select_limit * rec_per_key);
```
上限保护：若 `L > 组数（table_records/rec_per_key）` 说明要全表。

**② fanout 分支**——本表 1 行 → 后面表膨胀 fanout 行，所以拿到最终结果前 L 行只需读 `L/fanout` 行：

```cpp
if (fanout == 0) select_limit = HA_POS_ERROR;      // 除零 → 'infinite'
else if (fanout >= 0) select_limit = max(select_limit / fanout, 1.0);
```
`fanout < 0` 表示"未知"（段 1 的 break），**不缩放**（保守）。

**③ refkey_rows_estimate 分支**——假设排序索引与当前 ref/range 条件不相关：

```cpp
if (select_limit > refkey_rows_estimate) select_limit = table_records;
else select_limit = select_limit * table_records / refkey_rows_estimate;
```
沿排序索引走时，平均每 `table_records / refkey_rows_estimate` 条记录才有 1 条满足当前 ref 条件。

> **实现细节**：`select_limit` 是**按值传入的形参**，循环内就地改写且**不重置**——第 2 个及以后的候选索引，缩放起点沿用上次结果。这只是用于代价比较，真正返回的 `best_select_limit` 只在 `best_key` 更新时记录。

**索引顺序扫描代价公式**（`:5243`）：

```cpp
rec_per_key = keyinfo->records_per_key(keyinfo->user_defined_key_parts - 1);
const double index_scan_time =
    select_limit / rec_per_key *
    min<double>(table->file->page_read_cost(nr, rec_per_key),
                table->file->table_scan_cost().total_cost());
```

**推导**：
1. `rec_per_key` 是**整个索引**的——一个完整 key value 平均对应 `rec_per_key` 行
2. 这 `rec_per_key` 行在索引里**连续存放**（同 key 值内按 rowid/PK 有序），回表视作"一次连续小簇"
3. 扫 `select_limit` 条索引项 ⇒ `select_limit / rec_per_key` 个"回表簇访问"
4. 每簇最坏读 `rec_per_key` 个不同 page，但**整簇代价不可能超过全表扫一次** → 取 `min(...)`

**切换决策**（`:5256`）：

```cpp
if (((cur_access_method == JT_ALL || cur_access_method == JT_INDEX_SCAN) &&
     (is_covering || group || table->force_index_order)) ||
    index_scan_time < read_time) {
```

- **无条件切换**（不看代价）：当前是全表扫/索引扫，且候选是覆盖索引 / GROUP BY / `FORCE INDEX FOR ORDER BY`。理由：反正是全扫，换成能提供顺序的（覆盖）索引纯赚（省掉 filesort）
- **比代价**：其余情况要求 `index_scan_time < read_time`

> **比较基准是什么**：`read_time` 是**读**的代价，不含 filesort。filesort 代价在计划生成阶段已通过 `consider_plan()`（`sql_planner.cc:2496`）计入 `join->best_read`——**且只在"排序表 ≠ 第一个非 const 表"时才加**（若排序表就是首表，`sort_cost` 保持 0）。所以这里的语义是"用有序索引读 vs 用当前方法读 + 已记账的 filesort"。

**平局打破**（`:5274-5288`）：

| 情况 | 规则 |
|---|---|
| `select_limit <= min(quick_records, best_records)`（都够用） | 选 **keypart 更少**（key 更短）的 |
| 否则 | 选 **`quick_records` 更小**（选择性更好）的 |
| 附加 | **正扫优于反扫**（`direction > best_key_direction`），即使 key 更长 |

### 1.3 test_if_skip_sort_order（`sql_optimizer.cc:2193`）—— 编排

**流程**：

1. **早退**：`JT_EQ_REF/JT_CONST/JT_SYSTEM`（最多 1 行）→ 直接 true
2. **usable_keys** = 所有排序字段 `part_of_sortkey` 位图的**交集**；`JT_REF_OR_NULL` / `JT_FT` 直接拒绝
3. **当前索引不可用 → 找 subkey**：`test_if_subkey`（`:2072`）找以原 ref_key 的 keyparts 为**前缀**、且顺序匹配、key_length 最小的索引。ref 场景只置 `set_up_ref_access_to_key`（推迟改），**range 场景必须重跑 `test_quick_select`** 重建 QUICK
4. **代价搜索**：`prefer_ordering_index`（默认 **on**）或 FORCE INDEX 时调 `test_if_cheaper_ordering`
5. **多表反向保护**（见下）
6. **改写访问方法**

**多表 join 的反向保护**（`:2477-2492`）——为什么 filesort + join cache 通常更快：

```cpp
if (!is_force_index && (select_limit >= table_records) &&
    (tab->type() == JT_ALL && join->primary_tables > join->const_tables + 1) &&
    ((unsigned)best_key != table->s->primary_key ||
     !table->file->primary_key_is_clustered())) {
  can_skip_sorting = false;
```

四个条件同时成立才放弃有序索引：① 无 FORCE INDEX；② **没有能提前终止的 LIMIT**（`select_limit >= table_records`）；③ **多表 join 且首表是全表扫**；④ 选出的索引不是聚簇 PK。

**理由**：走有序索引意味着首表必须按索引顺序出 row；但内层表若用 join cache（BNL），cache 要求先批量攒够外层行再一次探测内层——**两者互斥**。而 filesort 只在最后排一次，内层全程可用 join cache 批量探测，IO 上通常远省。聚簇 PK 例外：按聚簇 PK 读 = 天然顺序 IO，不比全表扫差。

**改写访问方法的具体动作**（`:2591-2717`）：

| 场景 | 动作 |
|---|---|
| 转索引扫描 | `ref().key=-1`、`set_index(best_key)`、`set_type(JT_INDEX_SCAN)`、`set_range_scan(nullptr)`、`ha_index_or_rnd_end()` |
| 转 range | `set_type(calc_join_type(range_scan))`、`use_quick=QS_RANGE`；loose index scan 还要置 `precomputed_group_by=true`（故后面必须重跑 `count_field_types`） |
| ref 换 key | `create_ref_for_key()` **从零重建** `ref()`（两个索引的 search tuple 可能不同）；可能再经 `can_switch_from_ref_to_range` 转 range |
| 覆盖性回退 | 新索引非覆盖 → `set_keyread(false)`（撤销 "Using index"） |
| Temptable 临时表 | 不支持 `index_first()/index_last()` → 不建 index scan |

**反向三态**：`make_reverse()`（range）/ `reversed_access = true`（ref、index scan）。**反向时故意把 `changed_key` 设成当前 key**——后面 `push_index_cond`（`sql_select.cc:2928`）会因 `reversed_access` 直接 return，即**反向访问不做 ICP**（组合有性能回退）。

**rows_fetched 修正**（`:2727`）：成功且是 index scan 且有 LIMIT 时，`rows_fetched` 改为 `select_limit`（EXPLAIN rows 更真实）。

### 1.4 test_skip_sort（`sql_optimizer.cc:1642`）

**为什么只对第一个非 const 表判断**：

1. 代码硬约束：`JOIN_TAB *const tab = best_ref[const_tables];` + `assert(tab->idx() == const_tables)`
2. **算法原因**：嵌套循环 join 的**输出行序由最外层决定**。只有第一个非 const 表按索引顺序出 row，整个 join 输出才可能有序。第 2 个及以后的表即使用有序索引，其顺序也会被外层表的循环"打散"
3. 额外约束：即使首表有序，后续任何表用 join cache 仍会打乱行序——所以 `JOIN::optimize:907-915` 在 buffering 后清 `simple_order/simple_group`

**GROUP BY 分支 vs ORDER BY 分支**：

| | GROUP BY | ORDER BY |
|---|---|---|
| 可跳过性 | `simple_group && !select_distinct` | `simple_order \|\| skip_sort_order`，且 `!m_windows_sort` |
| 外层开关 | `SELECT_BIG_RESULT`/`with_json_agg`（否则没必要强行避免物化） | 无 |
| limit | `need_tmp_before_win ? HA_POS_ERROR : m_select_limit` | `m_select_limit` |
| Key_map | `keys_in_use_for_group_by` | `keys_in_use_for_order_by` |
| 失败后果 | 强制临时表但**不排序**（`simple_order=simple_group=false`） | 走 filesort |

> `simple_group`/`simple_order` 来自 `JOIN::remove_const()`（`:10175`）：排序/分组只引用首个非 const 表的字段，且该表**不是 LEFT JOIN 内表**（否则 NULL 补行破坏顺序）。

`m_ordered_index_usage` 枚举（`sql_optimizer.h:395`）：`ORDERED_INDEX_VOID` / `ORDERED_INDEX_GROUP_BY` / `ORDERED_INDEX_ORDER_BY`。消费者是 `make_tmp_tables_info:4759`（决定是否加 filesort）。

---

## 二、条件最终化与访问方法 setup

### 2.1 finalize_table_conditions（`sql_optimizer.cc:9177`）

两步算法：

**① `reduce_cond_for_table`（`:9042`）——删掉被 ref 保证为真的谓词**

`test_if_ref`（`:8987`）的判定链：

```
1. 访问类型不是 JT_REF_OR_NULL（它语义是 x=y OR x IS NULL，比 x=y 弱）
2. part_of_refkey()：在 tab->ref() 的 search tuple 中找到与 field 对应的 item
3. ref_item->eq(right_item, true)：谓词右侧与 ref 查找键是同一个表达式
4. ref_lookup_subsumes_comparison()：确认 ref 查找语义蕴含该比较（含 NULL 处理）
   → 判定冗余 ⇒ 整个 EQ 谓词被移除
```

**典型收益**：`t1 JOIN t2 ON t2.a=t1.a WHERE t2.a = 5`，t2 用 ref 访问 `a=t1.a`，条件 `t2.a = <ref key>` 在取到行后必然为真 → 删掉，省一次每行求值。

**AND/OR/TRIG_COND 的处理**：AND 逐参数递归（恒真则 remove）；OR 任一参数恒真则整体恒真；**TRIG_COND(FOUND_MATCH)** 把 inner tables 并入 `null_extended`（这些表可能被 NULL 扩展，其上的谓词**不能**被消除）。

**② `cache_const_expr`**——用 `Item::compile()` 的 analyzer+transformer 双遍机制，把常量子表达式包进 `Item_cache`，只在首次求值时算一次。同时作用于每个表条件和 HAVING。

### 2.2 make_join_readinfo（`sql/sql_select.cc:3282`，**不在 sql_executor.cc**）

**完整源码**：

```cpp
/**
  Plan refinement stage: do various setup things for the executor
  @param join          Join being processed
  @param no_jbuf_after Don't use join buffering after table with this number.
*/
bool make_join_readinfo(JOIN *join, uint no_jbuf_after) {
  const bool statistics = !join->thd->lex->is_explain();
  const bool prep_for_pos = join->need_tmp_before_win ||
                            join->select_distinct ||
                            !join->group_list.empty() || !join->order.empty() ||
                            join->m_windows.elements > 0;

  ASSERT_BEST_REF_IN_JOIN_ORDER(join);
  Opt_trace_context *const trace = &join->thd->opt_trace;
  Opt_trace_object wrapper(trace);
  Opt_trace_array trace_refine_plan(trace, "refine_plan");   // ★ trace: refine_plan

  // ★ semijoin 去重执行标记（LooseScan / FirstMatch jump / Weedout 临时表）在这里落定
  if (setup_semijoin_dups_elimination(join, no_jbuf_after))
    return true;

  for (uint i = join->const_tables; i < join->tables; i++) {
    QEP_TAB *const qep_tab = &join->qep_tab[i];
    if (!qep_tab->position()) continue;

    JOIN_TAB *const tab = join->best_ref[i];
    TABLE *const table = qep_tab->table();
    Table_ref *const table_ref = qep_tab->table_ref;
    /*
     Need to tell handlers that to play it safe, it should fetch all
     columns of the primary key of the tables: this is because MySQL may
     build row pointers for the rows ...
    */
    if (prep_for_pos) table->prepare_for_position();   // ★ 要建临时表/去重 ⇒ 必须能定位行

    Opt_trace_object trace_refine_table(trace);
    trace_refine_table.add_utf8_table(table_ref);

    if (tab->use_join_cache() != JOIN_CACHE::ALG_NONE)
      qep_tab->init_join_cache(tab);       // ★ join buffer 落地（ALG_BNL→OT_BNL / ALG_BKA→OT_BKA）

    switch (qep_tab->type()) {
      case JT_EQ_REF:
      case JT_REF_OR_NULL:
      case JT_REF:
      case JT_SYSTEM:
      case JT_CONST:
        if (table->covering_keys.is_set(qep_tab->ref().key) &&
            !table->no_keyread)
          table->set_keyread(true);         // ★ 覆盖索引 ⇒ 只读索引
        else
          qep_tab->push_index_cond(tab, qep_tab->ref().key,
                                   &trace_refine_table);   // ★ ICP 下推
        break;

      case JT_ALL:
        join->thd->set_status_no_index_used();
        qep_tab->using_dynamic_range = (tab->use_quick == QS_DYNAMIC_RANGE);
        [[fallthrough]];
      case JT_INDEX_SCAN:
        if (tab->position()->filter_effect != COND_FILTER_STALE_NO_CONST &&
            !tab->sj_mat_exec()) {
          /*
            rows_w_const_cond is # of rows which will be read by the access
            method, minus those which will not pass the constant condition;
            ... Such number is useful inside the planner, but obscure to the
            reader of EXPLAIN; so we put the real count of read rows into
            rows_fetched, and move the constant condition's filter to
            filter_effect.
          */
          double rows_w_const_cond = qep_tab->position()->rows_fetched;
          table_ref->fetch_number_of_rows();
          tab->position()->rows_fetched =
              static_cast<double>(table->file->stats.records);
          if (tab->position()->filter_effect != COND_FILTER_STALE) {
            // Constant condition moves to filter_effect:
            if (tab->position()->rows_fetched == 0)
              tab->position()->filter_effect = 0.0f;
            else
              tab->position()->filter_effect *= static_cast<float>(
                  rows_w_const_cond / tab->position()->rows_fetched);
          }
        }
        if (qep_tab->using_dynamic_range) {
          join->thd->set_status_no_good_index_used();
          if (statistics) join->thd->inc_status_select_range_check();
        } else {
          if (statistics) {
            if (i == join->const_tables)
              join->thd->inc_status_select_scan();
            else
              join->thd->inc_status_select_full_join();
          }
        }
        break;

      case JT_RANGE:
      case JT_INDEX_MERGE:
        qep_tab->using_dynamic_range = (tab->use_quick == QS_DYNAMIC_RANGE);
        if (statistics) {
          if (i == join->const_tables)
            join->thd->inc_status_select_range();
          else
            join->thd->inc_status_select_full_range_join();
        }
        if (!table->no_keyread && qep_tab->type() == JT_RANGE) {
          if (table->covering_keys.is_set(used_index(qep_tab->range_scan()))) {
            assert(used_index(qep_tab->range_scan()) != MAX_KEY);
            table->set_keyread(true);
          }
          if (!table->key_read)
            qep_tab->push_index_cond(tab, used_index(qep_tab->range_scan()),
                                     &trace_refine_table);
        }
        if (tab->position()->filter_effect != COND_FILTER_STALE_NO_CONST) {
          double rows_w_const_cond = qep_tab->position()->rows_fetched;
          // ★ range 的最终形态：rows_fetched 换成 range_scan 估算的输出行数
          qep_tab->position()->rows_fetched =
              tab->range_scan()->num_output_rows();
          if (tab->position()->filter_effect != COND_FILTER_STALE) {
            if (tab->position()->rows_fetched == 0)
              tab->position()->filter_effect = 0.0f;
            else
              tab->position()->filter_effect *= static_cast<float>(
                  rows_w_const_cond / tab->position()->rows_fetched);
          }
        }
        break;

      case JT_FT:
        if (tab->join()->fts_index_access(tab)) {
          table->set_keyread(true);
          table->covering_keys.set_bit(tab->ft_func()->key);
        }
        break;
      default:
        assert(0);
        break;
    }

    if (tab->position()->filter_effect <= COND_FILTER_STALE) {
      /*
        Give a proper value for EXPLAIN.
        For performance reasons, we do not recalculate the filter for
        non-EXPLAIN queries; thus, EXPLAIN CONNECTION may show 100%
        for a query.
      */
      tab->position()->filter_effect =
          (join->thd->lex->is_explain() || need_cost_update ||
           (join->m_select_limit != HA_POS_ERROR &&
            !Overlaps(join->thd->variables.option_bits, OPTION_BIG_SELECTS)))
              ? calculate_condition_filter(
                    tab,
                    (tab->ref().key != -1) ? tab->position()->key : nullptr,
                    tab->prefix_tables() & ~table_ref->map(),
                    tab->position()->rows_fetched, false, false,
                    trace_refine_table)
              : COND_FILTER_ALLPASS;   // ★★ 非 EXPLAIN 时直接拍成 100%：refinement 后的代价是"近似"的
      ...
    }

    qep_tab->set_reversed_access(tab->reversed_access);   // ★ DESC 索引扫描标记

    // Materialize derived tables prior to accessing them.
    if (table_ref->is_table_function()) {
      qep_tab->materialize_table = QEP_TAB::MATERIALIZE_TABLE_FUNCTION;
      if (tab->dependent) qep_tab->rematerialize = true;
    } else if (table_ref->uses_materialization()) {
      qep_tab->materialize_table = QEP_TAB::MATERIALIZE_DERIVED;
    }
    if (qep_tab->sj_mat_exec())
      qep_tab->materialize_table = QEP_TAB::MATERIALIZE_SEMIJOIN;

    if (table_ref->is_derived() &&
        table_ref->derived_query_expression()->m_lateral_deps) {
      // ★ LATERAL 派生表：找到它最后一个依赖表，由那张表的读取触发重新物化
      auto deps = table_ref->derived_query_expression()->m_lateral_deps;
      plan_idx last = NO_PLAN_IDX;
      for (JOIN_TAB **tab2 = join->map2table; deps; tab2++, deps >>= 1) {
        if (deps & 1) last = std::max(last, (*tab2)->idx());
      }
      if (last != NO_PLAN_IDX) {
        QEP_TAB &t = join->qep_tab[last];
        t.lateral_derived_tables_depend_on_me |= TableBitmap(i);
        trace_refine_table.add_utf8("rematerialized_for_each_row_of",
                                    t.table()->alias);
      }
    }
  }
  return false;
}
```

⇒ **三个反直觉点**：① `keyread` 与 `ICP` 是**互斥**的（覆盖索引就只读索引，不再下推）；② `rows_fetched` 在 refinement 里被**换成真实读行数**，把常量条件的过滤效果搬到 `filter_effect`，纯为 EXPLAIN 可读性；③ 非 EXPLAIN 时 `filter_effect` 直接置 `COND_FILTER_ALLPASS`——**refinement 之后的代价数字是近似的**。

### 2.2.1 本文档对 2.2 的小结

按 `QEP_TAB::type()` 分派：

| 类型 | 动作 |
|---|---|
| `JT_EQ_REF/REF/REF_OR_NULL/SYSTEM/CONST` | 覆盖索引（含聚簇 PK）→ `set_keyread(true)`；**否则** `push_index_cond` |
| `JT_ALL` | `set_status_no_index_used()`；EXPLAIN 展示修正（见下） |
| `JT_INDEX_SCAN` | EXPLAIN 展示修正 |
| `JT_RANGE` | 先试 `set_keyread(true)`；**若没进覆盖读则再试** `push_index_cond` |
| `JT_INDEX_MERGE` | 不做（used_index 为 MAX_KEY） |
| `JT_FT` | `fts_index_access` 成功 → `set_keyread(true)` |

**覆盖索引优先于 ICP 的原因**：覆盖读时已经不回表，ICP 省无可省。

**EXPLAIN 展示修正**（`JT_ALL`/`JT_INDEX_SCAN`/range）：计划阶段的 `rows_fetched` 是"扣掉常量条件后要读的行数"，对用户不直观；这里改回**真实扫描行数** `stats.records`，并把常量条件的过滤效果**搬回** `filter_effect`：

```cpp
tab->position()->filter_effect *= (float)(rows_w_const_cond / tab->position()->rows_fetched);
```
保证 `rows × filtered` 不变。

**覆盖索引 `set_keyread(true)`**：

```cpp
// sql/table.cc:6126
if (flag && !key_read) {
  key_read = true;
  if (is_created()) file->ha_extra(HA_EXTRA_KEYREAD);   // 只读索引不回表
}
```

判定依据 `TABLE::covering_keys`：初值 `s->keys_for_keyread`（要求 `index_flags & HA_KEYREAD_ONLY`），随后**随列被读取而收窄**——`mark_column_used()` 每次把字段加入 read_set 时就 `covering_keys.intersect(field->part_of_key)`。**一旦引用了不在某索引里的列，该索引就不再"覆盖"**。

**EXPLAIN "Using index" 输出**（`opt_explain.cc:1571`）：
- `JT_INDEX_SCAN/JT_CONST` 用 `covering_keys.is_set(tab->index())`
- 其它用 `table->key_read` 或 `tab->keyread_optim()`（后者在 `set_plan_state()` 里快照，**必须在 `make_tmp_tables_info` 之后**，因为它可能给首表加 sort）

### 2.3 push_index_cond（`sql_select.cc:2920`）—— SQL 层决策

**前置 early-out**：`reversed_access`（反向访问）、InnoDB 内部临时表、索引含虚拟生成列。

**other_tbls_ok**（`:2941`）：

```cpp
const bool other_tbls_ok =
    !((type() == JT_ALL || type() == JT_INDEX_SCAN || type() == JT_RANGE ||
       type() == JT_INDEX_MERGE) &&
      join_tab->use_join_cache() == JOIN_CACHE::ALG_BNL);
```

**扫描类 + BNL 时，下推条件不能引用其它非 const 表的字段**——BNL 会攒多行，条件中的"其它表字段"在攒行期间可能变化。ref/eq_ref（BKA）下 `other_tbls_ok` 为 true。

**8 项准入**：① 有 condition ② 引擎 `HA_DO_INDEX_COND_PUSHDOWN` ③ 开关+hint ④ 非多表 UPDATE/DELETE ⑤ `!has_guarded_conds()` ⑥ 非 JT_CONST/JT_SYSTEM ⑦ 非聚簇主键。

**下推与残留处理**：

```cpp
Item *idx_cond = make_cond_for_index(condition(), tbl, keyno, other_tbls_ok);
if ((idx_cond->used_tables() & table_ref->map()) == 0) return;   // 不含本表字段 → 推了没用
...
idx_remainder_cond = tbl->file->idx_cond_push(keyno, idx_cond);   // 返回残留
if (idx_remainder_cond != idx_cond) ref().disable_cache = true;    // 禁用 eq_ref lookup cache
set_condition(idx_remainder_cond);                                 // 残留留在 server
```

> handler 接口、`uses_index_fields_only` 四条限制、InnoDB 实现、EXPLAIN 输出见 **[05 篇 4 节](logical/05_logical_predicate.md)**。

---

## 三、push_to_engines：两层下推

**这是最容易混淆的一对概念**：

| 维度 | **handlerton 级**（整查询 offload） | **handler 级**（经典 ICP） |
|---|---|---|
| 函数 | `JOIN::push_to_engines()`（`sql_optimizer.cc:1143`） | `handler::idx_cond_push()`（`handler.h:5958`） |
| 粒度 | **整棵 AccessPath 树**（join/filter/aggregate） | **单表单索引上的一个布尔条件** |
| 目标 | 卸载到引擎（NDB pushed join、secondary engine offload） | 引擎内提前过滤索引记录，减少回表 |
| 输入 | `AccessPath *root_path` + `JOIN *` | `keyno` + `Item *idx_cond`（限索引列） |
| 时机 | `JOIN::optimize:1064`（`create_access_paths` 之后、Iterator 创建之前） | `make_join_readinfo`（`:1020`）内，**更早** |
| 副作用 | **会改写 AccessPath 树** | 只影响该 handler 的 `pushed_idx_cond` |
| 数量 | 每查询最多 1 个 handlerton（`break`） | 每个可用表各一次 |
| 失败处理 | 返回 true → optimize 失败 | 返回残留条件，server 保留求值（永远安全） |

```cpp
// sql/sql_optimizer.cc:1143
for (Table_ref *tl = query_block->leaf_tables; tl; tl = tl->next_leaf) {
  const handlerton *hton = tl->table->file->hton_supporting_engine_pushdown();
  if (hton != nullptr) {
    if (unlikely(hton->push_to_engine(thd, m_root_access_path, this))) return true;
    break;   // Assume that at most a single handlerton per query support pushdown
  }
}
```

**关键约束**（`:1049-1063` 注释）：push_to_engines **允许修改 AccessPath 本身**（移除已下推的 FILTER、改 JOIN 算法、改聚合表达式），所以**必须在创建 Iterator 之前**完成。

**谁实现**：本仓库中只有 **NDB**（`ha_ndbcluster.cc:12664` 注册 `ndbcluster_push_to_engine`）。（社区版无 HeatWave 实现。）

---

## 四、setup_join_buffering

> 本篇讲**优化器的 buffer 决策**（能否用、用 BNL 还是 BKA、外连接/semijoin 约束）。**执行器侧的缓冲机制**（BNL 改写 hash join、BKA + DS-MRR、hash join 的 build/probe/spill）详见 [`../../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md)。

`sql_optimizer.cc:3451`。**扫描类 → BNL，ref 类 → BKA**（基于 MRR）。

**参数**：`bnl` 默认 **on**，`bka` 默认 **off**（不在 `OPTIMIZER_SWITCH_DEFAULT` 里）；`join_buffer_size` 默认 **256KB**。

**外连接/semijoin 约束**（节选）：

| 约束 | 说明 |
|---|---|
| 首表永不 buffer（`tableno == const_tables`） | 没有可缓存的外层行 |
| 外连接内表：nest 的 `first_inner` 不 buffer → 本表也不 | match flag 只能挂在第一个内表的 cache 上 |
| `SJ_OPT_FIRST_MATCH` | 仅当 semijoin 只含一个内表才允许 |
| `SJ_OPT_LOOSE_SCAN` / 被物化的第一个表 | 禁止 |
| `DUPS_WEEDOUT` / `NONE` | 允许 |
| LATERAL 派生表依赖区间内的表 | 禁止 |

**BKA 三个否决条件**：MRR 不可用（`rows == HA_POS_ERROR`）/ 用了 MRR 默认实现（不是真 MRR）/ 引擎声明 `HA_MRR_NO_ASSOCIATION`。

**`revise_cache_usage`**（`:3326`）：本表不用 buffer 时，要**向前回溯**关掉依赖它的表的 buffer（外连接场景关掉整个 nest 之前的表）。

> **Hash join 不在本函数**：8.0.39 的 hash join 只在 **hypergraph 优化器**路径生成；传统优化器只有 BNL/BKA。

---

## 五、临时表与排序 make_tmp_tables_info

`JOIN::make_tmp_tables_info()`，`sql/sql_select.cc:4354`（**是 JOIN 成员函数，不是自由函数**）。关键源码：

```cpp
bool JOIN::make_tmp_tables_info() {
  assert(!join_tab);
  mem_root_deque<Item *> *curr_fields = fields;
  bool materialize_join = false;
  uint curr_tmp_table = const_tables;
  TABLE *exec_tmp_table = nullptr;

  auto cleanup_tmp_tables_on_error =
      create_scope_guard([this, &curr_tmp_table] { ... });   // 出错时关掉已建的临时表

  /*
    If the plan is constant, we will not do window tmp table processing
    cf. special code path for handling const plans.
  */
  m_windowing_steps = m_windows.elements > 0 && !plan_is_const() &&
                      !implicit_grouping && !group_optimized_away;   // ★ 窗口步骤开关
  ...
  Opt_trace_array trace_tmp(trace, "considering_tmp_tables");   // ★ trace: considering_tmp_tables

  /*
    In this function, we may change having_cond into a condition on a
    temporary sort/group table, so we have to assign having_for_explain now:
  */
  having_for_explain = having_cond;

  /*
    The loose index scan access method guarantees that all grouping or
    duplicate row elimination (for distinct) is already performed
    during data retrieval, and that all MIN/MAX functions are already
    computed for each group. ...
  */
  if (qep_tab && qep_tab[0].range_scan() &&
      is_loose_index_scan(qep_tab[0].range_scan()))
    tmp_table_param.precomputed_group_by =
        !is_agg_loose_index_scan(qep_tab[0].range_scan());   // ★ LIS 已完成分组

  /*
    Create the first temporary table if distinct elimination is requested or
    if the sort is too complicated to be evaluated as a filesort.
  */
  if (need_tmp_before_win) {
    curr_tmp_table = primary_tables;
    tmp_tables++;
    ...
    // ★ 备份 base slice，因为后续步骤会覆盖它（1-3 号 slice）
    if (alloc_ref_item_slice(thd, REF_SLICE_SAVED_BASE)) return true;
    copy_ref_item_slice(REF_SLICE_SAVED_BASE, REF_SLICE_ACTIVE);
    current_ref_item_slice = REF_SLICE_SAVED_BASE;

    ORDER_with_src tmp_group;
    if (!simple_group && !(test_flags & TEST_NO_KEY_GROUP) && !with_json_agg)
      tmp_group = group_list;   // ★ GROUP BY 只有 !simple_group 时才做进临时表唯一键

    tmp_table_param.hidden_field_count = CountHiddenFields(*fields);

    if (create_intermediate_table(&qep_tab[curr_tmp_table], *fields, tmp_group,
                                  !group_list.empty() && simple_group))
      return true;
    exec_tmp_table = qep_tab[curr_tmp_table].table();

    if (exec_tmp_table->s->is_distinct) optimize_distinct();

    if (alloc_ref_item_slice(thd, REF_SLICE_TMP1)) return true;

    // Change sum_fields reference to calculated fields in tmp_table
    if (streaming_aggregation || qep_tab[curr_tmp_table].table()->group ||
        tmp_table_param.precomputed_group_by) {
      if (change_to_use_tmp_fields(fields, thd, ref_items[REF_SLICE_TMP1],
                                   &tmp_fields[REF_SLICE_TMP1], ...))
        return true;
    } else {
      if (change_to_use_tmp_fields_except_sums(...)) return true;
    }
    curr_fields = &tmp_fields[REF_SLICE_TMP1];
    set_ref_item_slice(REF_SLICE_TMP1);
    qep_tab[curr_tmp_table].ref_item_slice = REF_SLICE_TMP1;
    setup_tmptable_write_func(&qep_tab[curr_tmp_table], &trace_this_outer);
    // ★ 决定 op_type：OT_AGGREGATE_INTO_TMP_TABLE / OT_AGGREGATE_THEN_MATERIALIZE / OT_MATERIALIZE

    /*
      If having is not handled here, it will be checked before the row is sent
      to the client. ...
      NOTE : We cannot apply having after distinct. If columns of having are
             not part of select distinct, then distinct may remove rows
             which can satisfy having.
    */
    if (having_cond &&
        (streaming_aggregation ||
         (exec_tmp_table->s->is_distinct && group_list.empty()))) {
      if ((!select_distinct && rollup_state == RollupState::NONE) &&
          add_having_as_tmp_table_cond(curr_tmp_table))
        return true;
      qep_tab[curr_tmp_table].having = having_cond;
      having_cond = nullptr;   // ★★ HAVING 被搬走（后面 create_root_access_path_for_join 再取回）
    }
    ...
    if (exec_tmp_table->group) {   // Already grouped
      /* Check if group by has to respect ordering. If true, move group by to order by. */
      if (order.empty() && !skip_sort_order) {
        for (ORDER *group = group_list.order; group; group = group->next) {
          if (group->direction != ORDER_NOT_RELEVANT) {
            order = group_list; /* order by group */
            break;
          }
        }
      }
      group_list.clean();
    }

    /*
      ★ 第二张临时表：GROUP BY 与 ORDER BY 不一致 / DISTINCT 带 rollup / 有窗口步骤
      If we have different sort & group then we must sort the data by group
      and copy it to a second temporary table.
    */
    if ((!group_list.empty() &&
         (!test_if_subpart(group_list.order, order.order) || select_distinct ||
          m_windowing_steps || rollup_state != RollupState::NONE)) ||
        (select_distinct && (tmp_table_param.using_outer_summary_function ||
                             rollup_state != RollupState::NONE))) {
      ...
      curr_tmp_table++;
      tmp_tables++;
      trace_this_tbl.add("adding_tmp_table_in_plan_at_position", curr_tmp_table)
          .add_alnum("cause", "sorting_to_make_groups");

      ORDER_with_src dummy;
      if (create_intermediate_table(&qep_tab[curr_tmp_table], *curr_fields,
                                    dummy, true))
        return true;

      if (!group_list.empty()) {
        explain_flags.set(group_list.src, ESP_USING_TMPTABLE);
        if (!plan_is_const())   // No need to sort a single row
        {
          // ★ filesort 挂在"上一张表"上（filesort 只能排单表）
          if (add_sorting_to_table(curr_tmp_table - 1, &group_list,
                                   /*sort_before_group=*/true))
            return true;
        }
        if (make_group_fields(this, this)) return true;
      }
      ...
      if (alloc_ref_item_slice(thd, REF_SLICE_TMP2)) return true;
      ...
      setup_tmptable_write_func(&qep_tab[curr_tmp_table], &trace_this_tbl);
    }

    if (qep_tab[curr_tmp_table].table()->s->is_distinct)
      select_distinct = false; /* Each row is unique */

    if (select_distinct && group_list.empty() && !m_windowing_steps) {
      if (having_cond) {
        qep_tab[curr_tmp_table].having = having_cond;
        having_cond->update_used_tables();
        having_cond = nullptr;
      }
      // ★★ 8.0 迭代器模型下 DISTINCT = "去重 filesort"，不是在临时表里删行
      qep_tab[curr_tmp_table].needs_duplicate_removal = true;
      trace_this_outer.add("reading_from_table_eliminates_duplicates", true);
      explain_flags.set(ESC_DISTINCT, ESP_DUPS_REMOVAL);
      select_distinct = false;
    }
    ...
  }

  /*
    ★ 不需要临时表的"流式聚合"路径（AggregateIterator）
    Set up structures for a temporary table but do not actually create
    the temporary table if ... implicitly grouped / simple grouping / LIMIT 1 / ROLLUP
  */
  if ((grouped || implicit_grouping) && !m_windowing_steps) {
    if (make_group_fields(this, this)) return true;
    if (make_sum_func_list(*curr_fields, true, true)) return true;
    const bool need_distinct =
        !(qep_tab && qep_tab[0].range_scan() &&
          is_agg_loose_index_scan(qep_tab[0].range_scan()));
    if (prepare_sum_aggregators(sum_funcs, need_distinct)) return true;
    if (setup_sum_funcs(thd, sum_funcs) || thd->is_fatal_error()) return true;
  }

  if (qep_tab && (!group_list.empty() ||
                  (!order.empty() && !m_windowing_steps /* [1] */))) {
    /*
      [1] above: too early to do query ORDER BY if we have windowing; must
      wait till after window processing.
    */
    if (having_cond && group_list.empty() && !streaming_aggregation &&
        rollup_state == RollupState::NONE) {
      if (add_having_as_tmp_table_cond(curr_tmp_table)) return true;
    }
    if (grouped)
      m_select_limit = HA_POS_ERROR;
    else if (!need_tmp_before_win) {
      /*
        We can abort sorting after thd->select_limit rows if there are no
        filter conditions for any tables after the sorted one. ...
      */
      for (uint i = const_tables + 1; i < primary_tables; i++) {
        QEP_TAB *const tab = qep_tab + i;
        if (tab->condition() ||                                  // 1
            (best_ref[tab->idx()]->keyuse() &&                   // 2
             tab->first_inner() == NO_PLAN_IDX)) {
          /* We have to sort all rows */
          m_select_limit = HA_POS_ERROR;
          break;
        }
      }
    }
    ORDER_with_src order_arg = group_list.empty() ? order : group_list;
    // ★ 只有 m_ordered_index_usage 未覆盖所需 clause，才真的加 filesort
    if (qep_tab &&
        m_ordered_index_usage != (group_list.empty()
                                      ? ORDERED_INDEX_ORDER_BY
                                      : ORDERED_INDEX_GROUP_BY) &&
        !m_windowing_steps) {
      QEP_TAB *const sort_tab = &qep_tab[curr_tmp_table];
      if (need_tmp_before_win && !materialize_join && !exec_tmp_table->group)
        explain_flags.set(order_arg.src, ESP_USING_TMPTABLE);

      if (add_sorting_to_table(curr_tmp_table, &order_arg,
                               /*sort_before_group=*/false))
        return true;
      /*
        filesort_limit: Return only this many rows from filesort().
        We can use select_limit_cnt only if we have no group_by and 1 table.
        This allows us to use Bounded_queue for queries like:
          "select * from t1 order by b desc limit 1;"
      */
      if (sort_tab->filesort)
        sort_tab->filesort->limit =
            (has_group_by || (primary_tables > curr_tmp_table + 1) ||
             calc_found_rows)
                ? m_select_limit
                : query_expression()->select_limit_cnt;
    }
  }

  // ★ 每个 window 一张临时表 + REF_SLICE_WIN_wn
  if (qep_tab && m_windowing_steps) {
    for (uint wno = 0; wno < m_windows.elements; wno++) {
      tmp_table_param.m_window = m_windows[wno];
      ...
      if (create_intermediate_table(tab, *curr_fields, dummy, false)) return true;
      if (alloc_ref_item_slice(thd, widx)) return true;
      ...
      if (m_windows[wno]->needs_buffering()) {
        if (CreateFramebufferTable(thd, tmp_table_param, *query_block,
                                   *orig_fields, *curr_fields,
                                   tab->tmp_table_param->items_to_copy,
                                   m_windows[wno]))
          return true;    // ★ 窗口帧缓冲临时表
      }
      ORDER_with_src w_partition(m_windows[wno]->sorting_order(), ESC_WINDOWING);
      if (w_partition.order != nullptr) {
        // ★ PARTITION BY/ORDER BY 的排序挂在"上一张"表上
        if (add_sorting_to_table(curr_tmp_table - 1, &w_partition,
                                 /*sort_before_group=*/false))
          return true;
      }
      if (m_windows[wno]->is_last()) {
        ...
        if (!tab->filesort && !tab->table()->s->keys && ...) {
          /*
            Last tmp table of execution; no sort, no duplicate elimination, ...
            hence any row needn't be written to tmp table's storage; send it out
            to query's result instead:
          */
          m_windows[wno]->set_short_circuit(true);   // ★ 最后一张窗口表可不落盘
        }
      }
      if (having_cond != nullptr) {
        tab->having = having_cond;
        having_cond = nullptr;
      }
    }
  }

  refresh_base_slice();
  fields = curr_fields;
  set_ref_item_slice(REF_SLICE_SAVED_BASE);
  if (qep_tab) {
    qep_tab[primary_tables + tmp_tables].op_type = get_end_select_func();
  }
  grouped = has_group_by;
  unplug_join_tabs();   // ★ 此后 JOIN_TAB 与 QEP_TAB 分离
  ...
  return false;
}
```

**`count_field_types()`**（`sql/sql_select.cc:3910`）——它决定 `sum_func_count` 与 `allow_group_via_temp_table`，进而影响 `test_skip_sort()`：

```cpp
void count_field_types(const Query_block *query_block, Temp_table_param *param,
                       const mem_root_deque<Item *> &fields,
                       bool reset_with_sum_func, bool save_sum_fields) {
  param->sum_func_count = 0;
  param->func_count = fields.size();
  param->hidden_field_count = 0;
  param->outer_sum_func_count = 0;
  save_sum_fields |= param->precomputed_group_by;

  for (Item *field : fields) {
    Item *real = field->real_item();
    Item::Type real_type = real->type();

    if (real_type == Item::SUM_FUNC_ITEM && !real->m_is_window_function) {
      if (!field->const_item()) {
        Item_sum *sum_item = down_cast<Item_sum *>(field->real_item());
        if (sum_item->aggr_query_block == query_block) {
          if (!sum_item->allow_group_via_temp_table)
            param->allow_group_via_temp_table = false;   // ★ UDF 聚合 ⇒ 禁止"临时表分组"
          param->sum_func_count++;
          param->func_count += sum_item->argument_count();
        }
      } else if (save_sum_fields) { ... }
    } else if (real_type == Item::SUM_FUNC_ITEM) {
      assert(real->m_is_window_function);
      param->func_count += down_cast<Item_sum *>(real)->argument_count();
    } else {
      if (reset_with_sum_func) field->reset_aggregation();
      if (field->has_aggregation()) param->outer_sum_func_count++;
    }
  }
}
```

七个阶段：

| 阶段 | 行号 | 关键决策 |
|---|---|---|
| 0 前置 | 4354-4416 | `m_windowing_steps`；**loose index scan 特判**：`precomputed_group_by = !is_agg_loose_index_scan(...)`（LIS 已完成分组和 MIN/MAX 预计算） |
| 1 第一张临时表 | 4422-4550 | `tmp_group` 只在 `!simple_group` 时传——即"是否把 GROUP BY 做进临时表的唯一键" |
| 2 HAVING/聚合 | 4502-4550 | HAVING 可下推为临时表条件（**DISTINCT 存在时不能**）；临时表已分组 → `order = group_list` |
| 3 第二张临时表 | 4558-4640 | **触发条件**：GROUP BY 与 ORDER BY 顺序不一致（`!test_if_subpart`）、或有 DISTINCT/window/ROLLUP → "先排序再分组" |
| 4 DISTINCT | 4641-4654 | `needs_duplicate_removal = true` |
| 5 filesort | 4712-4789 | `m_ordered_index_usage` 与所需 clause **不同**才加 filesort |
| 6 window | 4791-4897 | 每个 window 一张临时表 + `REF_SLICE_WIN_wn` slice |
| 7 收尾 | 4899-4928 | `refresh_base_slice()`、`unplug_join_tabs()` |

**filesort limit 优化**（`:4780`）：

```cpp
sort_tab->filesort->limit =
    (has_group_by || (primary_tables > curr_tmp_table + 1) || calc_found_rows)
        ? m_select_limit
        : query_expression()->select_limit_cnt;
```

只有"无 GROUP BY + 单表 + 非 calc_found_rows"时才把 `select_limit_cnt` 下压给 filesort → 允许 `Bounded_queue`（`ORDER BY b DESC LIMIT 1` 只需维护 1 个元素的堆）。

---


## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → ORDER BY Optimization*（`test_if_skip_sort_order` 的行为）
- *MySQL 8.0 Reference Manual → Index Condition Pushdown Optimization*
- *MySQL 8.0 Reference Manual → Block Nested-Loop and Batched Key Access Joins*、`Hash Join Optimization`
- *MySQL 8.0 Reference Manual → Internal Temporary Table Use in MySQL*

**内核月报**
- **2021/06《Order By 优化逻辑代码分析》** —— `test_skip_sort` / `test_if_order_by_key` 的索引免排序判定 + `filesort` 执行流程


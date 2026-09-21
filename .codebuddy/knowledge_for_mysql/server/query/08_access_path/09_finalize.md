# 08.9 最终化：FinalizePlanForQueryBlock

> `sql/join_optimizer/finalize_plan.cc`。AccessPath 树"建好"之后、"可执行"之前，还有一道**最终化**：建临时表、把 Item 树改写指向临时表字段、设置聚合器、下推引擎。本篇逐段剖析这个函数——它是 hypergraph 路径的"计划收尾"，也是**只能调用一次**的破坏性操作（见 [`../07_optimize/physical/18_hypergraph_advanced.md`](../07_optimize/physical/18_hypergraph_advanced.md) 第三章的概览，本篇是完整展开）。

## 目录

- [函数主体与四个前置分支](#函数主体与四个前置分支)
- [第一步：合并相邻 FILTER](#第一步合并相邻-filter)
- [第二步：主遍历（post-order）](#第二步主遍历post-order)
- [AGGREGATE 分支里的一条重要注释](#aggregate-分支里的一条重要注释)
- [收尾](#收尾)

---

## 函数主体与四个前置分支

```cpp
bool FinalizePlanForQueryBlock(THD *thd, Query_block *query_block) {
  assert(query_block->join->needs_finalize);
  query_block->join->needs_finalize = false;          // ① 幂等保护

  AccessPath *const root_path = query_block->join->root_access_path();
  assert(root_path != nullptr);
  if (root_path->type == AccessPath::EQ_REF) {
    // None of the finalization below is relevant to point selects, so just
    // return immediately.
    return false;                                     // ② 点查短路
  }

  // If the query is offloaded to an external executor, we don't need to create
  // the internal temporary tables or filesort objects, or rewrite the Item tree
  // to point into them.
  if (!IteratorsAreNeeded(thd, root_path)) {
    return false;                                     // ③ 二级引擎短路
  }

  Query_block *old_query_block = thd->lex->current_query_block();
  thd->lex->set_current_query_block(query_block);     // ④ 切换当前 block
  ...
```

| 分支 | 说明 |
|---|---|
| ① `needs_finalize` 置 false | **幂等保护**：assert + 立即清标志，重复调用会在 debug 构建崩（这就是"只能调一次"的运行期体现） |
| ② `EQ_REF` 点查短路 | 注释明说"以下最终化对点查都不适用"——单行 eq_ref 不需要临时表/聚合器/Item 改写 |
| ③ `IteratorsAreNeeded` 短路 | 二级引擎用外部执行器时（`USE_EXTERNAL_EXECUTOR`），不建内部临时表、filesort、不改写 Item 树——**最终化的一切产物都是给"内部执行"用的** |
| ④ 切换 `current_query_block` | 后续建临时表、改写 Item 都要工作在目标 block 的上下文里（THD 的很多操作依赖 current_query_block） |

---

## 第一步：合并相邻 FILTER

```cpp
  // We might have stacked multiple FILTERs on top of each other.
  // Combine these into a single FILTER:
  WalkAccessPaths(
      root_path, query_block->join, WalkAccessPathPolicy::ENTIRE_QUERY_BLOCK,
      [](AccessPath *path, JOIN *join) {
        if (path->type == AccessPath::FILTER) {
          AccessPath *child = path->filter().child;
          if (child->type == AccessPath::FILTER &&
              child->filter().materialize_subqueries ==
                  path->filter().materialize_subqueries) {
            // Combine conditions into a single FILTER.
            Item *condition = new Item_cond_and(child->filter().condition,
                                                path->filter().condition);
            condition->quick_fix_field();
            condition->update_used_tables();
            condition->apply_is_true();
            path->filter().condition = condition;
            path->filter().child = child->filter().child;
          }
        }
        return false;
      },
      /*post_order_traversal=*/true);
```

三个要点：

1. **合并条件**：child 也是 FILTER **且**两者的 `materialize_subqueries` 一致。这个一致性的原因是——`materialize_subqueries=true` 的 FILTER 在执行时可能触发子查询物化，与 `false` 的合并会改变子查询求值时机。

2. **新建 Item 的"三步必修课"**：`quick_fix_field()` + `update_used_tables()` + **`apply_is_true()`**。比 [`physical/18_hypergraph_advanced.md`](../07_optimize/physical/18_hypergraph_advanced.md) 里 CSE 说的"两步"多一步——`apply_is_true()` 处理 `WHERE` 语义对未知值的"truthiness"要求（如 `AND` 表达式遇到 NULL 的行为）。**合并条件时漏掉这一步会导致 NULL 语义错误**。

3. **post-order 遍历**：从叶子往根合并，保证"连续叠放的 FILTER"被逐层吃掉（child 先合并完，parent 再看 child 是否还能继续并）。

---

## 第二步：主遍历（post-order）

```cpp
  Mem_root_array<const Func_ptr_array *> applied_replacements(thd->mem_root);
  TABLE *last_window_temp_table = nullptr;
  unsigned num_windows_seen = 0;
  bool after_aggregation = false;
  WalkAccessPaths(
      root_path, query_block->join, WalkAccessPathPolicy::ENTIRE_QUERY_BLOCK,
      [&, ...](AccessPath *path, JOIN *join) {
        if (error) return true;
        DelayedCreateTemporaryTable(thd, query_block, path, after_aggregation,
                                    &last_window_temp_table, &num_windows_seen);

        const mem_root_deque<Item *> *original_fields = join->fields;
        UpdateReferencesToMaterializedItems(
            thd, query_block, path, after_aggregation, &applied_replacements);
        if (path->type == AccessPath::WINDOW) {
          FinalizeWindowPath(thd, query_block, *original_fields,
                             applied_replacements, path);
        } else if (path->type == AccessPath::AGGREGATE) {
          ... /* 见下一节 */
        }
        if (AddCachesAroundConstantConditionsInPath(path)) { error = true; return true; }
        return false;
      },
      /*post_order_traversal=*/true);
```

主遍历每到一个节点做四件事（顺序敏感）：

### ① DelayedCreateTemporaryTable：**延迟**建临时表

物化节点的 `temp_table` 在建树时是 `nullptr`（见 `00_structure.md` 的 WINDOW 说明），在这里才真正建 `TABLE` 对象。**"Delayed"的含义**：建表需要知道"输入行有哪些列"（尤其聚合后列集变了），所以要等到**子树处理完**才建——这就是用 post-order 的原因。

三个状态变量跨节点传递：
- `last_window_temp_table` / `num_windows_seen`：多个窗口共享/区分临时表
- `after_aggregation`：布尔状态机——**越过 AGGREGATE 节点后，后续节点面对的是聚合后的列集**

### ② UpdateReferencesToMaterializedItems：Item 改写

把 `Item_field`（指向基表列）替换成指向临时表列。替换规则记录在 `applied_replacements`（`Func_ptr_array` 数组）里，**跨节点累积**——聚合前后的两次替换都要记录，供 AGGREGATE 分支与窗口分支使用。

### ③ WINDOW 分支：FinalizeWindowPath

用 `original_fields`（进入该节点前的 fields 快照）+ 累积的 `applied_replacements` 最终化窗口路径——窗口函数的 frame 处理与缓冲（详见 [`../runtime/04_window_function.md`](../runtime/04_window_function.md)）。

### ④ AddCachesAroundConstantConditionsInPath

在常量条件周围包 `Cached_item`（如 `WHERE b = const` 的 `b` 不必每行重求值）。它在每个节点都调用，因为常量条件可能出现在任何位置。

---

## AGGREGATE 分支里的一条重要注释

```cpp
        } else if (path->type == AccessPath::AGGREGATE) {
          for (Cached_item &ci : join->group_fields) {
            for (const Func_ptr_array *earlier_replacement : applied_replacements) {
              thd->change_item_tree(
                  ci.get_item_ptr(),
                  FindReplacementOrReplaceMaterializedItems(
                      thd, ci.get_item(), *earlier_replacement,
                      /*need_exact_match=*/true));
            }
          }

          // Set up aggregators, now that fields point into the right temporary
          // table.
          const bool need_distinct =
              true;  // We don't support loose index scan yet.
          for (Item_sum **func_ptr = join->sum_funcs; *func_ptr != nullptr;
               ++func_ptr) {
            Item_sum *func = *func_ptr;
            Aggregator::Aggregator_type type =
                need_distinct && func->has_with_distinct()
                    ? Aggregator::DISTINCT_AGGREGATOR
                    : Aggregator::SIMPLE_AGGREGATOR;
            if (func->set_aggregator(type) || func->aggregator_setup(thd)) { ... }
          }
          after_aggregation = true;
        }
```

三个要点：

### ① `change_item_tree` 而不是直接赋值

分组键（`group_fields` 里的 `Cached_item`）的 Item 替换用 `thd->change_item_tree()`——**支持 PS 复用时回滚**（见 [`../07_optimize/21_collation_index_usability.md`](../07_optimize/21_collation_index_usability.md) 的"就地永久替换"技法对照）。`need_exact_match=true` 要求精确匹配（分组键错配是正确性问题）。

### ② ★ "We don't support loose index scan yet"——hypergraph 不支持 LIS

`need_distinct` 硬编码为 `true`，注释明说 **hypergraph 还不支持松索引扫描（LIS）**。后果：经典优化器里 `GROUP_INDEX_SKIP_SCAN` 可以让聚合走"预计算分组"（`precomputed_group_by`，见 [`../07_optimize/15_groupby_distinct_order.md`](../07_optimize/15_groupby_distinct_order.md)），而 hypergraph 里**带 `DISTINCT` 的聚合一律走 `DISTINCT_AGGREGATOR`**——这也是 hypergraph 计划与经典优化器计划性能差异的一个已知来源。

### ③ `set_aggregator` 必须在 Item 改写**之后**

注释：*"Set up aggregators, now that fields point into the right temporary table."* 聚合器要初始化"往哪里聚合"，而"哪里"是临时表列——所以顺序是：先建临时表 → 再改写 Item 指向 → **最后**设置聚合器。这个顺序由 post-order 遍历 + `after_aggregation` 状态机保证。

---

## 收尾

```cpp
  if (query_block->join->push_to_engines()) return true;

  thd->lex->set_current_query_block(old_query_block);
  return error;
```

最后 `push_to_engines()`（引擎条件下推 + 二级引擎卸载判定，见 [`../07_optimize/17_optimizer_decisions.md`](../07_optimize/17_optimizer_decisions.md) 决策 31/34），并恢复 `current_query_block`。

## 完整调用栈

```
Query_block::optimize()（hypergraph 分支）
  └─ FindBestQueryPlan()
       └─ EnumerateAllConnectedPartitions() → 产出带位图的 AccessPath 树
  └─ ExpandFilterAccessPaths()                     ← 位图 → FILTER
  └─ FinalizePlanForQueryBlock(thd, query_block)   ← 本篇
       ├─ 前置：needs_finalize / EQ_REF 短路 / IteratorsAreNeeded
       ├─ WalkAccessPaths(post-order)：合并相邻 FILTER
       ├─ WalkAccessPaths(post-order)：
       │     ├─ DelayedCreateTemporaryTable        （延迟建临时表）
       │     ├─ UpdateReferencesToMaterializedItems（Item → 临时表列）
       │     ├─ WINDOW → FinalizeWindowPath
       │     ├─ AGGREGATE → change_item_tree + set_aggregator
       │     └─ AddCachesAroundConstantConditionsInPath
       └─ push_to_engines()
  └─ CreateIteratorFromAccessPath()                 ← 见 06_rowiterator.md
```

## 参考

- [`../07_optimize/physical/18_hypergraph_advanced.md`](../07_optimize/physical/18_hypergraph_advanced.md) —— 最终化的概览视角（与本篇互补：那里讲"为什么"，本篇讲"每一步做什么"）
- [`00_structure.md`](00_structure.md) —— `temp_table` 为何建树时是 `nullptr`
- [`../07_optimize/15_groupby_distinct_order.md`](../07_optimize/15_groupby_distinct_order.md) —— `precomputed_group_by`（LIS 预计算）的经典优化器对照

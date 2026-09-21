# 08.2 构建（新优化器）：hypergraph 直出 AccessPath

> hypergraph 优化器**不经过 `QEP_TAB`**，直接产出 AccessPath。本篇讲"直出"这条路的实现细节：`CostingReceiver::Propose*` 系列如何一边搜索一边 new 出带代价的 AccessPath。建图与 DPhyp 枚举见 [`../07_optimize/physical/09_hypergraph.md`](../07_optimize/physical/09_hypergraph.md)，CSE/最终化见 [`../07_optimize/physical/18_hypergraph_advanced.md`](../07_optimize/physical/18_hypergraph_advanced.md)。

## 目录

- [与旧优化器的本质差异](#与旧优化器的本质差异)
- [Propose 模式：栈上值对象 → MEM_ROOT](#propose-模式栈上值对象--mem_root)
- [ProposeTableScan 逐段剖析](#proposetablescan-逐段剖析)
- [ProposeRefAccess 逐段剖析](#proposerefaccess-逐段剖析)
- [代价随 Propose 就填好](#代价随-propose-就填好)

---

## 与旧优化器的本质差异

| 维度 | 旧优化器 | hypergraph |
|---|---|---|
| 中间结构 | `QEP_TAB` 链（左深树） | `RelationalExpression` / `JoinHypergraph` |
| 产出方式 | `create_access_paths()` 把 QEP_TAB **翻译**成树 | **搜索的同时直接 new 出 AccessPath** |
| cost 何时填 | 翻译时从 `POSITION` 搬（`SetCostOnTableAccessPath`） | **Propose 的时候就填好** |
| 谓词表达 | `Item` 树 | 位图（`filter_predicates` 等，见 `05_parameterization.md`） |

★ 最本质的差异：**"搜索 = 构造"合二为一**。旧优化器先在 `POSITION` 数组上做搜索（可变黑板，见 [`../07_optimize/16_join_object_model.md`](../07_optimize/16_join_object_model.md)），搜索完才翻译成树；hypergraph 的 DPhyp 是**自底向上递归构造候选**的，每个子问题的解就是一个 AccessPath 子树，所以"找到一个候选"和"new 出一个 AccessPath"是同一件事。

---

## Propose 模式：栈上值对象 → MEM_ROOT

`CostingReceiver` 的三个单表 Propose 函数（`ProposeTableScan` / `ProposeIndexScan` / `ProposeRefAccess`）有一个统一模式：

```cpp
bool CostingReceiver::ProposeTableScan(TABLE *table, int node_idx, ...) {
  ...
  AccessPath path;                              // ① 先在栈上建值对象
  path.type = AccessPath::TABLE_SCAN;           // ② 填字段（像填 struct 一样）
  path.table_scan().table = table;
  path.count_examined_rows = true;
  path.ordering_state = 0;                       // ③ interesting order 状态
  ...
  path.num_output_rows_before_filter = num_output_rows;  // ④ 代价现在就填
  path.init_cost = path.init_once_cost = 0.0;
  path.cost_before_filter = path.cost = cost;
  ...
  // 需要长期持有时才搬上 MEM_ROOT：
  AccessPath *new_path = new (m_thd->mem_root) AccessPath(path);  // ⑤ 落盘
  ...
}
```

★ 与旧优化器工厂函数的对比：

| | 旧优化器工厂（`New*`） | hypergraph Propose |
|---|---|---|
| 创建方式 | 直接 `new (mem_root)` + 填字段 | **先栈上值对象，需要时再 `new (mem_root)` 拷贝** |
| cost | 大多留 `-1.0` 后填 | **现在就填好** |
| 探索性 | 无（工厂即最终） | **有**——栈上对象可以建了又丢弃，只有被采纳才落盘 |

"栈上值对象 → 落盘"是 Propose 的精髓：DPhyp 会探索大量候选，很多是**试探性的**（算了代价发现不优就丢弃），如果每个都 `new (mem_root)` 就会污染 MEM_ROOT。所以先在栈上构造、比较，只有"进入 Pareto 前沿"的候选才搬上 MEM_ROOT。

---

## ProposeTableScan 逐段剖析

> 这是 hypergraph 直出最完整的一个例子，逐段拆解（`join_optimizer.cc`）。

### ① 递归 CTE 的 FOLLOW_TAIL

```cpp
if (tl->is_recursive_reference()) {
  path.type = AccessPath::FOLLOW_TAIL;
  path.follow_tail().table = table;
  assert(forced_leftmost_table == 0);  // There can only be one, naturally.
  forced_leftmost_table = NodeMap{1} << node_idx;
}
```

递归自引用用 `FOLLOW_TAIL`（读临时表尾部），且**强制它是最左表**（`forced_leftmost_table`）。

### ② 代价与行数：Propose 时就填

```cpp
const double num_output_rows = table->file->stats.records;
const double cost = table->file->table_scan_cost().total_cost();

path.num_output_rows_before_filter = num_output_rows;
path.init_cost = path.init_once_cost = 0.0;
path.cost_before_filter = path.cost = cost;
```

行数用 `stats.records`，代价用 `table_scan_cost().total_cost()`（handler 接口）。全表扫的 `init_cost` 是 0（读第一行不需要额外初始化）。

### ③ 单表 UPDATE/DELETE 的就地修改候选

```cpp
if (IsBitSet(node_idx, m_immediate_update_delete_candidates)) {
  path.immediate_update_delete_table = node_idx;
  // 若扫描用的是聚簇键、且修改了主键，则不能就地改
  if (IsUpdateStatement(m_thd) &&
      Overlaps(table->file->ha_table_flags(), HA_PRIMARY_KEY_IN_READ_INDEX) &&
      !table->s->is_missing_primary_key() &&
      is_key_used(table, table->s->primary_key, table->write_set)) {
    path.immediate_update_delete_table = -1;
  }
}
```

`immediate_update_delete_table` 是"单表 DML 就地修改"的优化标志（详见 DML 篇），这里在 Propose 阶段就预判：**若表扫走的是聚簇键（primary key），修改主键值就不能就地改**（因为会破坏扫描顺序）。

### ④ information_schema 表：先填充再扫

```cpp
if (tl->schema_table != nullptr && tl->schema_table->fill_table) {
  AccessPath *new_path = new (m_thd->mem_root) AccessPath(path);
  AccessPath *materialize_path =
      NewMaterializeInformationSchemaTableAccessPath(m_thd, new_path, tl, nullptr);
  materialize_path->num_output_rows_before_filter = num_output_rows;
  materialize_path->init_cost = path.cost;       // Rudimentary.
  materialize_path->init_once_cost = path.cost;  // Rudimentary.
  ...
  materialize_path->filter_predicates = path.filter_predicates;
  materialize_path->delayed_predicates = path.delayed_predicates;
  new_path->filter_predicates.Clear();
  new_path->delayed_predicates.Clear();
  ...
  path = *materialize_path;
}
```

三个动作：包一层 `MATERIALIZE_INFORMATION_SCHEMA_TABLE`（I_S 表必须先 fill 再扫）；把内层 TABLE_SCAN 的 `filter_predicates` / `delayed_predicates` **上移**到物化层（因为谓词要在物化后应用）；代价标 `// Rudimentary`（粗略，作者自己承认）。

### ⑤ 派生表 / 表函数：包 MATERIALIZE + parameter_tables

```cpp
} else if (tl->uses_materialization()) {
  AccessPath *stable_path = new (m_thd->mem_root) AccessPath(path);
  ...
  if (tl->is_table_function()) {
    materialize_path = NewMaterializedTableFunctionAccessPath(thd, table, tl->table_function, stable_path);
    ...
    materialize_path->parameter_tables = GetNodeMapFromTableMap(
        tl->table_function->used_tables() & ~PSEUDO_TABLE_BITS,
        m_graph->table_num_to_node_num);
    if (Overlaps(tl->table_function->used_tables(),
                 OUTER_REF_TABLE_BIT | RAND_TABLE_BIT)) {
      // Make sure the table function is never hashed, ever.
      materialize_path->parameter_tables |= RAND_TABLE_BIT;
    }
  }
  ...
}
```

★ 这里就是 `05_parameterization.md` 里 `parameter_tables` 与 `RAND_TABLE_BIT` 的**实际写入点**：

- 表函数的 `used_tables()` 转成 `parameter_tables`（表函数依赖哪些表）
- 若依赖外层引用或非确定性（`OUTER_REF_TABLE_BIT | RAND_TABLE_BIT`），额外置 `RAND_TABLE_BIT` 哨兵——注释明说 "**Make sure the table function is never hashed, ever**"（永远不能做 hash join 右侧）

---

## ProposeRefAccess 逐段剖析

> ref 访问是"下推 join 条件"的主战场，也是 `parameter_tables` 的另一个来源。

### ① 逐个 keypart 匹配 sargable 谓词

```cpp
for (unsigned keypart_idx = 0; keypart_idx < usable_keyparts; ++keypart_idx) {
  const KEY_PART_INFO &keyinfo = key->key_part[keypart_idx];
  bool matched_this_keypart = false;
  for (const SargablePredicate &sp : m_graph->nodes[node_idx].sargable_predicates) {
    if (!sp.field->part_of_key.is_set(key_idx)) continue;   // Quick reject
    Item_func_eq *item = down_cast<Item_func_eq *>(m_graph->predicates[sp.predicate_index].condition);
    if (sp.field->eq(keyinfo.field)) {
      const table_map other_side_tables = sp.other_side->used_tables() & ~PSEUDO_TABLE_BITS;
      if (IsSubset(other_side_tables, allowed_parameter_tables)) {
        parameter_tables |= other_side_tables;
        matched_this_keypart = true;
        keyparts[keypart_idx].field = sp.field;
        keyparts[keypart_idx].condition = item;
        keyparts[keypart_idx].val = sp.other_side;
        keyparts[keypart_idx].null_rejecting = true;
        ...
        ++matched_keyparts;
        break;
      }
    }
  }
  if (!matched_this_keypart) break;   // 某个 keypart 断了，后面的 keypart 全无意义
}
```

★ 两个关键点：

1. **前缀匹配规则**：`if (!matched_this_keypart) break;` —— 一旦某个 keypart 匹配不上，后面所有 keypart 都放弃。这是索引最左前缀规则的体现（ref access 只能匹配连续 keypart）。

2. **`parameter_tables` 的来源**：`parameter_tables |= other_side_tables` —— **ref access 依赖的"等式另一边"的表，就是 `parameter_tables`**。这正是 `05_parameterization.md` 里"下推的 join 条件"这一来源的源码实锤：`t1.x = t2.x` 下推成 t1 的 ref access，则 t2 进 `parameter_tables`。

### ② 去重剪枝

```cpp
if (matched_keyparts == 0) return false;
if (parameter_tables != allowed_parameter_tables) {
  // We've already seen this before, with a more lenient subset, so don't try it again.
  return false;
}
```

第二段是重要的**剪枝**：`parameter_tables` 相同的 ref 候选只保留第一次（`parameter_tables` 是"哪些表已经可用"的集合，相同集合的 ref 是等价的，不必重复提议）。

### ③ `KeypartForRef` 结构

每个匹配的 keypart 记录五元组：`field`（哪个列）、`condition`（等值 `Item`）、`val`（另一边的值）、`null_rejecting`、`used_tables`。这是后续 `NewRefAccessPath` 的原料。

---

## 代价随 Propose 就填好

总结 hypergraph 直出与旧优化器的最大差异：**代价在 Propose 阶段就用 `table->file->*_cost()` 填好了**，而不是"先搜索后翻译时从 `POSITION` 搬"。

带来的后果：

1. **AccessPath 出生即完整**（带类型 + 参数 + 代价），不需要 `SetCostOnTableAccessPath` 这类后补步骤
2. **试探性候选零成本丢弃**：栈上建、比价、不采纳就丢弃，只有 Pareto 前沿的才 `new (mem_root)`
3. **与 `CostingReceiver` 的 Pareto 锦标赛衔接**：Propose 出的带价 AccessPath 直接进入候选池比较（见 09 篇）

---

## 与 09 篇的分工

本篇聚焦"**Propose 如何 new 出 AccessPath**"；`FoundJoin` 如何把多个 Propose 出的单表 path **组合成 join 节点**（NestedLoop / HashJoin / SemiJoin），以及 Pareto 前沿如何淘汰候选，是 [`../07_optimize/physical/09_hypergraph.md`](../07_optimize/physical/09_hypergraph.md) 的 CostingReceiver 章节。两者合起来才是"hypergraph 直出"的完整图景。

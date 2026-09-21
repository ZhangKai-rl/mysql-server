# 08.1 构建（旧优化器）：QEP_TAB → AccessPath

> 经典优化器如何把 `QEP_TAB` 链翻译成 AccessPath 树。核心是 `ConnectJoins()`——它是整个构建过程的灵魂。hypergraph 的直出路径见 `02_construction_hypergraph.md`。

## 目录

- [入口：create_access_paths](#入口create_access_paths)
- [ConnectJoins 要解决的三个问题](#connectjoins-要解决的三个问题)
- [GetTableAccessPath 五分支](#gettableaccesspath-五分支)
- [create_table_access_path 三分支](#create_table_access_path-三分支)
- [代价怎么从 POSITION 搬到 AccessPath](#代价怎么从-positon-搬到-accesspath)
- [包裹顺序](#包裹顺序)
- [BNL 被换成 HashJoin](#bnl-被换成-hashjoin)

---

## 入口：create_access_paths

```cpp
void JOIN::create_access_paths() {
  assert(m_root_access_path == nullptr);
  AccessPath *path = create_root_access_path_for_join();
  path = attach_access_paths_for_having_and_limit(path);
  path = attach_access_path_for_update_or_delete(path);
  m_root_access_path = path;
}
```

`create_root_access_path_for_join()` → **`ConnectJoins()`** 是 QEP_TAB 链翻译的核心。

---

## ConnectJoins 要解决的三个问题

函数签名有 11 个参数、其中 4 个是递归传递的 vector，本身就说明了复杂度：

```cpp
AccessPath *ConnectJoins(plan_idx upper_first_idx, plan_idx first_idx,
                         plan_idx last_idx, QEP_TAB *qep_tabs, THD *thd,
                         CallingContext calling_context,
                         vector<PendingCondition> *pending_conditions,
                         vector<PendingInvalidator> *pending_invalidators,
                         vector<PendingCondition> *pending_join_conditions,
                         qep_tab_map *unhandled_duplicates,
                         table_map *conditions_depend_on_outer_tables);
```

它要同时解决三个问题：

### ① 树形：左深树 + 三种嵌套子结构

MySQL 的 join 树一般是**左深树**，所以内层循环可以"从左边开始，一张张往右接"。但表序列里会嵌着三类子结构，每种都靠 `QEP_TAB` 上的下标标记：

| 子结构 | 怎么标记 | 处理方式 |
|---|---|---|
| **外连接** | 在切片的第一张表上设 `first_inner()` / `last_inner()` | 递归处理 `[first_inner, last_inner]` 这段，作为外连接的右臂，再继续外层内连接 |
| **semi-join（FirstMatch）** | 表 N 设了"first match 指向表 M" | 递归处理子切片 `[M+1, N]`，作为 semijoin 右侧 |
| **Duplicate Weedout** | `Substructure::WEEDOUT` | 包一层去重节点 |

★ 源码注释有一条很关键的警示：**不能用 `first_sj_inner()` 来检测 semijoin，因为它不随 join 顺序重排而更新**（join optimizer 会重排表，但 `first_sj_inner()` 不会跟着改）。这就是为什么检测逻辑绕了一圈用 `firstmatch_return`。这个坑在 `JOIN::optimize` 里到处都是——凡是"存了下标"的地方，都要问一句"重排后它还对吗"。

外连接与 semijoin **可以嵌套**，所以 `FindSubstructure()` 必须挑出**最外层**的那个结构来递归，否则会同一个 semijoin 反复处理（`calling_context` 参数就是为了防止无限递归）。

### ② 条件：什么时候能求值

这是 `ConnectJoins` 最微妙的部分。概念上 SQL 的条件在**所有表都 join 完之后**才求值；但为了效率，我们希望**尽早求值**：

| 上下文 | 条件何时求值 |
|---|---|
| 只有内连接 | 一旦条件涉及的表都读完了，立即求值 |
| **在外连接的右臂内部** | **必须延迟**——外连接会补 NULL 行，提前求值会错 |

延迟靠 `pending_conditions` 向量实现：递归进入外连接右臂时，把"本该在这里求值的 WHERE 谓词"推进向量，等 join 完成后由 `FinishPendingOperations()` 统一挂成 `FILTER` 节点。

★ `pending_conditions` 是否为 `nullptr` 就是"我现在是不是在外连接右臂里"的**标志**：

- `nullptr` → 不在外连接内侧，条件可以立即求值
- 非 `nullptr` → 在外连接内侧，谓词必须推迟

这是一种"用指针的空/非空承载上下文"的写法。递归调用时四种传法各不相同：

```
SEMIJOIN + UseHashJoin
      → 传 &subtree_pending_conditions（新向量）
        因为要把 join conditions 上提到本层，好挂到 hash join 迭代器上
SEMIJOIN + 非 hash join
      → 传 pending_conditions（继承父层）
OUTER_JOIN 且已在某个外连接内侧
      → 继承 pending_conditions，继续推迟 WHERE
OUTER_JOIN 且不在外连接内侧
      → 传新的 &subtree_pending_conditions，本层 ON 条件可就地挂
```

递归返回后，用 `PickOutConditionsForTableIndex()` 把"属于这一层的 ON 条件"从 pending 向量里挑出来，挂到刚建好的 join 节点**正上方**。

### ③ anti-join 的特判（NOT IN 场景）

```cpp
if (qep_tab->table()->reginfo.not_exists_optimize) {
  join_type = (pending_conditions == nullptr) ? JoinType::ANTI : JoinType::OUTER;
}
```

`not_exists_optimize` 表示"找到一条就不需要再找了"（`NOT IN` 的典型优化）。但**只有当不在另一个外连接的右臂内时，才能启用 antijoin**——否则会让上层外连接本不该产生 NULL 行的地方产生 NULL 行。

还有一个更刁的特例：`NOT IN` 的 antijoin 会在**同一张表上**设置 found 触发器，条件被包在 `not_null_compl` 与 `found` 里。如果照常处理，这个条件会把所有输出行都杀掉。所以要专门识别 `it->cond->item_name.ptr() == antijoin_null_cond`，把它**从 pending 里删掉**，并用它把 join 标记成"真正的 ANTI"（即使它嵌在外连接里）。

★ 这类"语义正确性的补丁"是 `ConnectJoins` 里最难读的部分——它的逻辑不是"怎么建树"，而是"哪些情况下不能按常规建树"。

### FinishPendingOperations：统一收口

```cpp
AccessPath *FinishPendingOperations(
    THD *thd, AccessPath *path, QEP_TAB *remove_duplicates_loose_scan_qep_tab,
    const vector<PendingCondition> &pending_conditions,
    table_map *conditions_depend_on_outer_tables) {
  path = PossiblyAttachFilter(path, pending_conditions, thd,
                              conditions_depend_on_outer_tables);

  if (remove_duplicates_loose_scan_qep_tab != nullptr) {
    KEY *key = qep_tab->table()->key_info + qep_tab->index();
    AccessPath *old_path = path;
    path = NewRemoveDuplicatesOnIndexAccessPath(
        thd, path, qep_tab->table(), key, qep_tab->loosescan_key_len);
    CopyBasicProperties(*old_path, path);
  }
  return path;
}
```

两件事：把积压的条件包成 `FILTER`（`PossiblyAttachFilter` 会合并相邻 filter 并判断能否省略），以及给 LooseScan 包一层 `REMOVE_DUPLICATES_ON_INDEX`。

### 顶层外连接的"假单行表"

```cpp
bool is_top_level_outer_join =
    calling_context == TOP_LEVEL &&
    qep_tabs[first_idx].last_inner() != NO_PLAN_IDX;

if (is_top_level_outer_join) {
  path = NewFakeSingleRowAccessPath(thd, /*count_examined_rows=*/false);
}
```

若**第一张表就是外连接的内侧表**，说明它左边隐含着"一个或多个 const 表"，于是先造一个 `FAKE_SINGLE_ROW` 作为左臂。

---

## GetTableAccessPath 五分支

每张表的基础访问路径由 `GetTableAccessPath()` 决定，按 `QEP_TAB::materialize_table` 分五路：

| 分支 | 产出 | 说明 |
|---|---|---|
| `MATERIALIZE_DERIVED` | `GetAccessPathForDerivedTable()` | 派生表：用 `qep_tab->access_path()` 作为内部查询，包一层物化 |
| `MATERIALIZE_TABLE_FUNCTION` | `NewMaterializedTableFunctionAccessPath()` | JSON_TABLE 等表函数 |
| `MATERIALIZE_SEMIJOIN` | ★ **递归调用 `ConnectJoins()`** | 见下 |
| （非物化）`schema_table->fill_table` | `NewMaterializeInformationSchemaTableAccessPath()` | information_schema 表必须先填充再扫 |
| （普通表） | `qep_tab->access_path()` | 就是 `create_table_access_path()` 产出的那个 |

★ **`MATERIALIZE_SEMIJOIN` 分支最值得注意**：物化 semi-join 的内层是一组表，它递归调用 `ConnectJoins()` 把这组表连接成一棵子树，源码注释称之为 **"virtual join"**——

> Handle this subquery as we would a completely separate join, even though the tables are part of the same JOIN object.

即：**同一个 `JOIN` 对象里的两组表，通过递归 `ConnectJoins` 被当成两个独立的 join 来建树**。这个递归还要处理三件事：

1. **weedout 收尾**：递归期间可能有"本该去重但没去重"的表（`unhandled_duplicates`），在虚拟 join 结束时补一个 `WEEDOUT` 节点
2. **NULL 过滤**：物化 semi-join 基于 ref access，而 **ref access 的语义是 `NULL = NULL`，但 `IN` 表达式不应如此**。所以要给每个 nullable 的 `sj_inner_exprs` 加 `Item_func_isnotnull`。这对 `IN` 只是优化（等值传播已经过滤掉了），但**对 `NOT IN` 是正确性的必需**
3. **代价**：`EstimateMaterializeCost()` 回填

---

## create_table_access_path 三分支

```cpp
AccessPath *create_table_access_path(THD *thd, TABLE *table, AccessPath *range_scan,
                                     Table_ref *table_ref, POSITION *position,
                                     bool count_examined_rows) {
  AccessPath *path;
  if (range_scan != nullptr) {
    range_scan->count_examined_rows = count_examined_rows;
    path = range_scan;
  } else if (table_ref != nullptr && table_ref->is_recursive_reference()) {
    path = NewFollowTailAccessPath(thd, table, count_examined_rows);
  } else {
    path = NewTableScanAccessPath(thd, table, count_examined_rows);
  }
  if (position != nullptr) {
    SetCostOnTableAccessPath(*thd->cost_model(), position,
                             /*is_after_filter=*/false, path);
  }
  return path;
}
```

三选一：有 range 计划就用它；否则若是 `WITH RECURSIVE` 的自引用就用 `FOLLOW_TAIL`（读临时表尾部而非重扫）；否则全表扫。

---

## 代价怎么从 POSITION 搬到 AccessPath

```cpp
void SetCostOnTableAccessPath(const Cost_model_server &cost_model,
                              const POSITION *pos, bool is_after_filter,
                              AccessPath *path) {
  double num_rows_after_filtering = pos->rows_fetched * pos->filter_effect;
  if (is_after_filter) {
    path->set_num_output_rows(num_rows_after_filtering);
  } else {
    path->set_num_output_rows(pos->rows_fetched);
  }

  double cost = pos->read_cost + cost_model.row_evaluate_cost(num_rows_after_filtering);
  if (pos->prefix_rowcount <= 0.0) {
    path->cost = cost;
  } else {
    // Scale the estimated cost to being for one loop only, to match the
    // measured costs.
    path->cost = cost * num_rows_after_filtering / pos->prefix_rowcount;
  }
}
```

★ 两个要点：

1. **`num_output_rows` 有两态**：`is_after_filter=false` 时是过滤前行数（`rows_fetched`），`true` 时是过滤后（`rows_fetched * filter_effect`）。所以 `FILTER` 节点**下方**记过滤前行数、上方记过滤后——读 `EXPLAIN FORMAT=JSON` 的 `rows_examined_per_scan` / `rows_produced_per_join` 时要分清。

2. **代价被缩放到"单次循环"**：`cost * num_rows_after_filtering / prefix_rowcount`。因为 `POSITION::read_cost` 是**整个前缀**的累计代价，而 AccessPath 上记的是"这个节点每被调用一次要花多少"。这个缩放是为了让估算值与 `EXPLAIN ANALYZE` 实测值可比较。

（另有一处注释坦承"we don't try to adjust for the filtering here; we estimate the same cost as the table itself"——即过滤不减代价，这是一个已知的粗略处理。）

---

## 包裹顺序

得到 `table_path` 之后，在把它接进 join 之前，还会依次包若干层。顺序是固定的：

```
① BKA        → NewMRRAccessPath()（把随机索引查找换成按 rowid 排序的顺序查找，即 DS-MRR）
② 条件下推   → PossiblyAttachFilter(predicates_below_join)
               除非 condition_is_pushed_to_sort()（条件已推到排序前面）
③ LooseScan  → NewRemoveDuplicatesOnIndexAccessPath()（仅单表 LooseScan）
④ 失效器     → NewInvalidatorAccessPathForTable()（lateral derived table 的缓存失效）
⑤ 接进 join  → NestedLoop / HashJoin / ...
```

**备注 ②**：`condition_is_pushed_to_sort()` 为真时**不挂 filter**，只把条件依赖的表并入 `conditions_depend_on_outer_tables`（条件已被推到 filesort 前面）。

**备注 ③**：只有 `do_loosescan() && match_tab == i`（LooseScan 只命中这一张表）才在这里包去重节点。**多表 LooseScan** 由 `NestedLoopSemiJoinWithDuplicateRemovalIterator` 处理——本质是"semijoin 的 NestedLoop + RemoveDuplicatesOnIndex 合二为一"。

**备注 ④（lateral derived table）** 的 pending 机制很典型：若被失效的表属于**更高的外连接 nest**，不能立即发 invalidator——因为当前外连接可能还会发出 NULL 补行，而那些补行也需要失效缓存。所以推进 `pending_invalidators`，等回到同一 nest 再处理。注释里那句 "But if we deal with them later than that, it might be too late!" 说明这里对时序很敏感。

---

## BNL 被换成 HashJoin

```cpp
SplitConditions(qep_tab->condition(), qep_tab, &predicates_below_join,
                &predicates_above_join,
                replace_with_hash_join ? pending_join_conditions : nullptr, ...);
```

`replace_with_hash_join` 的判定是 `UseHashJoin(qep_tab) && !QueryMixesOuterBKAAndBNL(join)`。注释说明：**在 `create_access_paths` 阶段就已经决定可以把 BNL 换成 hash join 了，所以这里不再重复检查可行性**。

换成 hash join 时，`ExtractJoinConditions()` 把等值与非等值连接条件**全部上提**到 `join_conditions`，之后挂到 hash join 迭代器上；留在 `predicates_below_join` 的则仍然是 filter。

### 一个完整例子

```sql
SELECT * FROM t1 LEFT JOIN t2 ON t2.a = t1.a WHERE t1.b > 10;
```

`ConnectJoins` 的处理：

```
1. FindSubstructure 在 t2 上发现 first_inner/last_inner → OUTER_JOIN 子结构
2. 递归：[t1] 先建 TABLE_SCAN(t1)
3. 递归：[t2] 建 TABLE_SCAN(t2)，用 DIRECTLY_UNDER_OUTER_JOIN 上下文
4. ON 条件 (t2.a = t1.a) 由 PickOutConditionsForTableIndex 挑出，挂到 join 节点上
5. WHERE t1.b > 10：
   - 若不在外连接右臂内 → 可以在读 t1 后立即求值（FILTER 挂在 TABLE_SCAN(t1) 上）
   - 若在外连接右臂内     → 进 pending_conditions，由 FinishPendingOperations 挂到 join 之上
```

★ 第 5 步的两条路径，就是"同样的 WHERE，位置不同"的全部由来——也是 `EXPLAIN` 里 `Filter:` 节点位置差异的根源。

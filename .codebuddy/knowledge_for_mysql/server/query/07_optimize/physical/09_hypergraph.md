# 09 Hypergraph 优化器：JoinHypergraph 与 DPhyp

> 8.0.22 引入的新优化器，基于 DPhyp 算法。本篇全部算法级。

## 目录

- [零、定位：为什么需要它](#零定位为什么需要它)
- [一、开关与入口](#一开关与入口)
- [二、FindBestQueryPlan 全流程](#二findbestqueryplan-全流程)
- [三、JoinHypergraph 构建](#三joinhypergraph-构建)
- [四、DPhyp 算法](#四dphyp-算法)
- [五、CostingReceiver](#五costingreceiver)
- [六、图简化](#六图简化)
- [七、FinalizePlan](#七finalizeplan)
- [八、深潜：候选锦标赛与 LogicalOrderings](#八深潜候选锦标赛与-logicalorderings)
- [九、已知限制](#九已知限制)

---

## 零、定位：为什么需要它

旧优化器只能产出**左深树**（left-deep tree）：每次只能把一张表加到已定前缀的右侧。hypergraph 优化器打破这个限制，支持 **bushy tree**（任意形态的 join 树）。

```
逻辑查询树
   │  FindBestQueryPlan()（join_optimizer.cc:6392）
   ▼
AccessPath 树（直接产出，跳过 QEP_TAB）      ← 与旧优化器汇合点
```

> ⚠️ 状态：**默认关闭**，且 `WITH_HYPERGRAPH_OPTIMIZER` 只在 **debug 构建默认 ON**（`CMakeLists.txt:2124-2133`）。

---

## 一、开关与入口

> ⚠️ **四处位置纠错**（本版本实测）：
> 1. **没有 `costing_receiver.h`**——`CostingReceiver` 定义在 `join_optimizer.cc:165`
> 2. `EnumerateAllConnectedPartitions` **不在** join_optimizer.cc，在 `subgraph_enumeration.h:667`（header-only 模板）
> 3. `FinalizePlanForQueryBlock` 在 **`finalize_plan.cc:656`**（不是 join_optimizer.cc）
> 4. DPhyp 的真实函数名是 `FindNeighborhood` / `EnumerateComplementsTo` / `ExpandSubgraph` / `ExpandComplement` / `TryConnecting`（**没有** `AdvanceNode`/`ExpandCsg`/`EmitCsgCmpPair`）

### 1.1 开关是一个"冻结的布尔快照"

```cpp
// sql/sql_lex.h:3907
bool using_hypergraph_optimizer() const { return m_using_hypergraph_optimizer; }
private:
  bool m_using_hypergraph_optimizer{false};
```

**它在 parse 结束后就冻结，不是每次执行查开关**。原因（`sql_optimizer.cc:685-692` 注释）：预编译语句/存储程序用的是 **prepare 时**的优化器，后续执行不再重读 `optimizer_switch`。

赋值点：`sql_select.cc:556`（普通 SELECT）、`sql_parse.cc:3917`（存储程序）、`sql_prepare.cc:1050/1129`（PREPARE）、`set_var.cc:1425`、`sql_cmd_ddl_table.cc:372`、`sql_load.cc:2145`。

位定义 `OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER{1ULL << 24}`（`sql_const.h:223`）；名字表 `sys_vars.cc:3472`（源码注释：*"Deliberately not documented"*）。

**默认 off——判据是默认值常量里没有这一位**（`sys_vars.cc:196-211` 的 `OPTIMIZER_SWITCH_DEFAULT` 无此位）。

两个反直觉行为（`sys_vars.cc:3413-3438`）：

```cpp
if (current_hypergraph && !want_hypergraph) {
  // SET optimizer_switch=DEFAULT 不会把它关掉（防止 MTR --hypergraph 被中途取消）
  if (var->value == nullptr) var->save_result.ulonglong_value |= OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER;
} else if (!current_hypergraph && want_hypergraph) {
#ifdef WITH_HYPERGRAPH_OPTIMIZER
  push_warning(..., ER_WARN_HYPERGRAPH_EXPERIMENTAL);   // 允许，发实验性警告
#else
  my_error(ER_HYPERGRAPH_NOT_SUPPORTED_YET, MYF(0), "use in non-debug builds");
#endif
```

### 1.2 分流是硬分叉，且 plan 两次

```cpp
// sql/sql_optimizer.cc:610
if (thd->lex->using_hypergraph_optimizer()) {
  Item *where_cond_no_in2exists = remove_in2exists_conds(where_cond);
  ...
  m_root_access_path = FindBestQueryPlan(thd, query_block, trace_ptr);          // :622
  if (finalize_access_paths && m_root_access_path != nullptr)
    if (FinalizePlanForQueryBlock(thd, query_block)) return true;               // :624
  ...
  m_root_access_path_no_in2exists = FindBestQueryPlan(thd, query_block, trace_ptr); // :657
  return false;                                                                  // :678
}
// :681 注释 "All of this is never called for the hypergraph join optimizer!"
```

两点：
- **硬分叉**：hypergraph 分支在 `:678` 就 return，后面整个旧优化器永不执行
- **plan 两次**：`m_root_access_path`（含 in2exists）与 `m_root_access_path_no_in2exists`（去掉 in2exists），供上层在"semijoin 物化 vs in2exists"间二选一

**两个重要 assert**（`:6399-6401`）：hypergraph **不做 const table 优化**、**不在优化期执行子查询**（`OPTION_NO_CONST_TABLES | OPTION_NO_SUBQUERY_DURING_OPTIMIZATION`）。

### 1.3 未编译时：源码仍编译进 mysqld

`sql/CMakeLists.txt:419-434` 里 hypergraph 源码**无条件编译**。所以没有"编译期剔除 → 运行期 fallback"。退化路径是：

1. `mysqld.cc:4964` 启动时调 `update_optimizer_switch()`
2. `sys_vars.cc:3406-3411`：未定义宏时把位清掉
3. 用户 `SET ...=on` → `check_optimizer_switch` 报 `ER_HYPERGRAPH_NOT_SUPPORTED_YET`

即：**hypergraph 分支永远存在于二进制里，未定义宏时只是无法被打开**。

---

## 二、FindBestQueryPlan 全流程

定义 `join_optimizer.cc:6392-7004`。

| 步骤 | 行号 | 说明 |
|---|---|---|
| `CheckSupportedQuery()` | 6395（定义 4846） | 只查 secondary engine（见第八节） |
| `JoinHypergraph graph(...)` + `MakeJoinHypergraph()` | 6424/6426 | 建超图 |
| `where_is_always_false` 短路 | 6430-6437 | 直接返回 ZERO_ROWS |
| `FindSargablePredicates()` | 6439（定义 6282） | 找可下推谓词 |
| `CacheCostInfoForJoinConditions()` | 6443（定义 6318） | **缓存**每条 join condition 的属性，避免在 O(枚举量) 循环里反复遍历 |
| `materializable_predicates` 位图 | 6466-6472 | 标记含子查询、可选先物化的谓词 |
| `BuildInterestingOrders()` | 6508（build_interesting_orders.cc:392） | 收集 ORDER BY/GROUP BY/DISTINCT/window 需要的序 |
| `InjectCastNodes()` | 6513（定义 5240） | 插入 CAST 保证类型可比 |
| 构造 `CostingReceiver` | 6526 | |
| 单表快路径 `FoundSingleNode(0)` | 6533-6539 | `graph.edges.empty()` 时跳过枚举 |
| **`EnumerateAllConnectedPartitions()`** | **6540** | **DPhyp 主循环** |
| 超限 → `SimplifyQueryGraph()` 重跑 | 6541-6576 | 见第六节 |
| 补 WHERE 谓词 + `ExpandFilterAccessPaths()` | 6634-6725 | |
| GROUP BY / 聚合 | 6727-6816 | `CreateStreamingAggregationPath`(5343) |
| HAVING | 6827-6834 | `ApplyHavingCondition`(5404) |
| 窗口函数 | 6851-6860 | `ApplyWindowFunctions`(5979) |
| DISTINCT / ORDER BY | 6867-6879 | `ApplyDistinctAndOrder`(5492) |
| LIMIT/OFFSET | 6883-6901 | |
| DELETE/UPDATE | 6903-6935 | |
| **按 cost 取最小** | **6951-6955** | `std::min_element(cost)` |
| 写回 `best_rowcount/best_read` | 6990-6992 | |

**为什么最后一步才比 cost**：中间阶段保留的是 **Pareto 前沿**（cost / init_cost / ordering / parameterization 多维权衡），只有到根之后"其他维度都不再重要"，才退化成单目标取最小。

---

## 三、JoinHypergraph 构建

### 3.1 Node / Hyperedge 结构（`hypergraph.h`）

```cpp
struct Node {                                    // :51
  std::vector<unsigned> complex_edges, simple_edges;
  NodeMap simple_neighborhood = 0;
  char padding[...];                             // 对齐到 64 字节
};
static_assert(sizeof(Node) >= 64);

struct Hyperedge {                               // :80
  NodeMap left;
  NodeMap right;
};
```

要点：
- 端点是 **`NodeMap` 位图**（左右各一个），两边非空且不相交
- **边被存两份**：`AddEdge(l,r)` 同时压入 `(l,r)` 和 `(r,l)`（`hypergraph.cc:34-46`）。挂在某节点的边，该节点一定在 `left`——这样 `FindNeighborhood` 只需检查 `IsSubset(e.left, subgraph)`，不用判方向，**省 30% 时间**。所以 `graph.edges[2i]` 与 `graph.edges[2i+1]` 互为反向
- **simple vs complex 分离**：simple = 左右都单 bit。所有 simple 邻居压缩成 `simple_neighborhood` 一个位图，`FindNeighborhood` 一次 `|=` 搞定
- `sizeof(Node) >= 64`——对齐 cache line
- `NodeMap` 是 **`uint64_t` 位集**（`node_map.h:40`），最多 61 张表

### 3.2 建图流程（`make_join_hypergraph.cc:3474`）

| 行 | 步骤 | 作用 |
|---|---|---|
| 3487 | `num_tables==1` → `MakeSingleTableHypergraph` | 单表快路径 |
| 3500 | `MakeRelationalExpressionFromJoinList`（306） | `Table_ref` 树 → `RelationalExpression` 代数树 |
| 3504 | `ComputeCompanionSets`（355） | companion set（可自由重排的内连接连通块） |
| 3506 | `FlattenInnerJoins`（462） | 无条件内连接压平 |
| 3525 | `PushDownJoinConditions`（1931） | ON 条件下推 |
| 3536-3549 | `EarlyExpandMultipleEquals` / `ExtractConditions` / `ReorderConditions` | WHERE 拆合取项、展开多等值、尽量下推 |
| 3553/3568 | `UnflattenInnerJoins`（505） | 恢复二叉形式 |
| 3584 | `MakeHashJoinConditions`（2201） | 分离出 `equijoin_conditions` |
| **3615** | **`MakeJoinGraphFromRelationalExpression`**（3138） | **真正建 hypergraph** |
| 3626 | `AddCycleEdges`（2991） | 补 cycle 边 |
| 3645 | `CompleteFullMeshForMultipleEqualities`（3303） | 多等值补全网格 |
| 3689 | 剩余 WHERE → `AddPredicate`（2853） | |

**`MakeRelationalExpressionFromJoinList`（306-353）**——注意 **semi/anti join 是一等公民**：

```cpp
if (tl->is_sj_or_aj_nest()) {
  join->type = tl->is_sj_nest() ? RelationalExpression::SEMIJOIN
                                : RelationalExpression::ANTIJOIN;
} else {
  if (tl->outer_join)      join->type = RelationalExpression::LEFT_JOIN;
  else if (tl->straight)   join->type = RelationalExpression::STRAIGHT_INNER_JOIN;
  else                     join->type = RelationalExpression::INNER_JOIN;
}
```

不像旧优化器那样靠 weedout/firstmatch/loosescan/materialization 五种策略去"模拟"（`join_optimizer.cc:396` 注释明确：*"The hypergraph optimizer does not use weedout"*）。

**`MakeJoinGraphFromRelationalExpression`（3138）**——后序遍历，**每个非叶子 join 算子 → 恰好一条 hyperedge**：

```cpp
if (expr->type == RelationalExpression::TABLE) { graph->graph.AddNode(); ... return; }
MakeJoinGraphFromRelationalExpression(thd, expr->left,  trace, graph);
MakeJoinGraphFromRelationalExpression(thd, expr->right, trace, graph);
expr->nodes_in_subtree = left->nodes_in_subtree | right->nodes_in_subtree;

const Hyperedge edge = FindHyperedgeAndJoinConflicts(thd, used_nodes, expr, graph);
graph->graph.AddEdge(edge.left, edge.right);

double selectivity = 1.0;
for (Item *item : expr->equijoin_conditions)
  selectivity *= EstimateSelectivity(current_thd, item, trace);
graph->edges.push_back(JoinPredicate{expr, selectivity, estimated_bytes_per_row, ...});
```

每条边带一个 `JoinPredicate`（`expr` 指针 + 综合 selectivity + 估计行宽 + 函数依赖 FD）。

### 3.3 CD-C 算法：TES 与 conflict rules（核心）

`FindHyperedgeAndJoinConflicts`（`make_join_hypergraph.cc:2715`）注释明说：

> *"This function is almost verbatim the CD-C algorithm from 'On the correct and complete enumeration of the core search space' by Moerkotte et al [Moe13]."*

**概念**：conflict rule（CR）`A → B` 表示"若 A 中任一表在 join 中，则 B 中所有表都必须存在"。三张查表 `OperatorsAreAssociative`（734）/ `OperatorsAreLeftAsscom`（776）/ `OperatorsAreRightAsscom`（823）判定 (父算子, 子算子) 组合能否做结合律 / l-asscom / r-asscom 改写；**不能就加一条 CR**。

**CR 折叠进 TES**——`AbsorbConflictRulesIntoTES`（`2614`）是**不动点迭代**：

```cpp
do {
  prev_total_eligibility_set = total_eligibility_set;
  for (const ConflictRule &rule : *conflict_rules) {
    if (Overlaps(rule.needed_to_activate_rule, total_eligibility_set)) {
      // 该 CR 恒生效 ⇒ 把 required_nodes 并进 TES
      total_eligibility_set |= rule.required_nodes;
    }
  }
  // 删掉 required_nodes 已是 TES 子集的 CR（已被边编码）
  conflict_rules->erase(std::remove_if(...), conflict_rules->end());
} while (total_eligibility_set != prev_total_eligibility_set && !conflict_rules->empty());
```

**这就是 TES 从"语法上的 SES"膨胀成"语义上必需的集合"的过程**，也是 hyperedge 端点会包含多于谓词所提及表的原因。剩下未能折叠的 CR 留在 `expr->conflict_rules`，由 `PassesConflictRules` 在 `FoundSubgraphPair` 里逐条检查（`join_optimizer.cc:3039`）。

**笛卡尔积处理**（`2775-2793`）：join 条件为空时 TES 为空，无法构造两端非空的边 → 把整个左/右子树塞进 TES。注释明说这会**禁止一些本合法的 join 序**，但保证不产生非法计划（"may make overly broad hyperedges… we will disallow otherwise valid plans (but never allow invalid plans)"）。

### 3.4 为什么叫 "hypergraph"（三类多节点端点）

| 情形 | 例子 |
|---|---|
| **① 外连接 / semi / anti join 的重排约束** | `t1 LEFT JOIN (t2 JOIN t3 USING(y)) ON t1.x=t2.x` → 边 `({t2,t3}, {t1})` |
| **② 真正的超谓词（hyperpredicate）** | `t1.a + t2.b = t3.c`（不可拆成二元等值对）→ 天然对应 `({t1,t2}, {t3})` |
| **③ cycle 边** | `AddCycleEdges`（2991）：`left = IsolateLowestBit(used_nodes); right = used_nodes & ~left` → 真生成 `{R1} ↔ {R2,R3}` |

**多表等值 `t1.a = t2.a = t3.a` 用"补全网格"而非单条三端点边**（`CompleteFullMeshForMultipleEqualities`，3303）：

```cpp
for (Item_field &left_field : item_equal->get_fields())
  for (Item_field &right_field : item_equal->get_fields())
    if (right_table_idx > left_table_idx)
      AddMultipleEqualityPredicate(...);   // 加 simple 边
```

**为什么**：DPhyp 的 `FindNeighborhood` 只枚举节点子集来增长子图；一条 `{t1,t2,t3}` 三端点边意味着必须三表同时到位才能连接，会**禁止掉 t1⋈t2 先做**这种合法且常优的 bushy 计划。改成 t1-t2、t1-t3、t2-t3 三条 simple 边后，既能枚举所有合法序，又能各自估价。

---

## 四、DPhyp 算法

全部在 **`subgraph_enumeration.h`**（header-only 模板，因为要实例化 `TrivialReceiver`/`CostingReceiver`/单测 receiver 三种）。

### 4.1 术语对照（`subgraph_enumeration.h:30-73` 的算法说明）

| 论文概念 | 源码 |
|---|---|
| `Solve()` | `EnumerateAllConnectedPartitions`（667） |
| `EmitCsg()` | `EnumerateComplementsTo`（351） |
| `EnumerateCsgRec()` | `ExpandSubgraph`（425） |
| `EnumerateCmpRec()` | `ExpandComplement`（581） |
| `N(S, X)` 邻域 | `FindNeighborhood`（285） |
| E↓'(S,X) / E↓(S,X) | `full_neighborhood` / `neighborhood` |
| csg-cmp-pair 回调 | `Receiver::FoundSubgraphPair` |
| `HasSeen(S)` 连通性测试 | `CostingReceiver::HasSeen`（`join_optimizer.cc:212`） |

### 4.2 主循环（667-691）

```cpp
template <class Receiver>
bool EnumerateAllConnectedPartitions(const Hypergraph &g, Receiver *receiver) {
  for (int seed_idx = g.nodes.size() - 1; seed_idx >= 0; --seed_idx) {
    if (receiver->FoundSingleNode(seed_idx)) return true;      // ① 先回调单节点
    NodeMap seed = TableBitmap(seed_idx);
    NodeMap forbidden = TablesBetween(0, seed_idx);            // ② 更小的节点一律禁
    NodeMap full_neighborhood = 0;
    NeighborhoodCache cache(0);
    NodeMap neighborhood = FindNeighborhood(g, seed, forbidden, seed,
                                            &cache, &full_neighborhood);
    if (EnumerateComplementsTo(g, seed_idx, seed, full_neighborhood,
                               neighborhood, receiver)) return true;   // ③ 找 complement
    if (ExpandSubgraph(g, seed_idx, seed, full_neighborhood, neighborhood,
                       forbidden | seed, receiver)) return true;       // ④ 继续长大
  }
  return false;
}
```

- **① 先回调单节点**（base table 的 access path 生成）——保证"小子集先于大子集被访问"这一 DP 前提
- **② 倒序枚举种子 + `forbidden`**：第 `seed_idx` 轮只考虑"**最小节点恰为 seed_idx**"的子图，更小的节点一律 forbidden。这是 DPhyp **不重复枚举**的核心
- **③④** 对每个 csg 做两件事：找 complement 配对、继续长大

### 4.3 FindNeighborhood（285-341）—— N(S, X)

```cpp
for (size_t node_idx : BitsSetIn(to_search)) {
  neighborhood |= g.nodes[node_idx].simple_neighborhood;      // simple 边：一次 |=

  for (size_t edge_idx : g.nodes[node_idx].complex_edges) {
    const Hyperedge e = g.edges[edge_idx];
    if (IsSubset(e.left, subgraph) && !Overlaps(e.right, subgraph | forbidden)) {
      full_neighborhood |= e.right;                            // E↓'(S,X)
      if (!Overlaps(e.right, neighborhood))                    // 未被 subsumed
        neighborhood |= IsolateLowestBit(e.right);             // E↓(S,X)：取代表节点
    }
  }
}
neighborhood &= ~(subgraph | forbidden);
```

四个要点：

1. **只扫 `just_grown_by`（新增节点）**，不扫整个 subgraph——上一轮邻域节点要么已进 S 要么已进 X，两侧都排除，不可能再贡献新边
2. **最小化（subsumption）是启发式**：若 `e.right` 与已有 `neighborhood` 有交集就跳过。完全最小化是 minimum set problem，O(n²/log n) 太贵
3. **代表节点** `IsolateLowestBit(e.right)`：子集枚举器 `NonzeroSubsetsOf` 只能枚举节点子集，不能枚举 hypernode 子集，所以每个 hypernode 取一个代表（注释："any node that's part of the hypernode would do"）
4. **性能**：占 DPhyp 总耗时 **20–70%**（微基准平均 ~40%）

**`NeighborhoodCache`（163-205）单元素缓存**：若本次 `just_grown_by` 是上次集合的超集，就续用上次邻域只算增量。`m_taboo_bit` 避免每轮都冲掉缓存（低度图损失 15-20%，星形高度图提速 60%+）。

### 4.4 EnumerateComplementsTo（351-416）—— EmitCsg

```cpp
NodeMap forbidden = TablesBetween(0, lowest_node_idx);   // ★ 重置，不含上层 forbidden
NeighborhoodCache cache(neighborhood);
for (size_t seed_idx : BitsSetInDescending(neighborhood)) {    // 倒序选 complement 种子
  NodeMap seed = TableBitmap(seed_idx);
  // 单节点 complement：直接判定连通
  if (Overlaps(g.nodes[seed_idx].simple_neighborhood, subgraph))
    for (edge_idx : g.nodes[seed_idx].simple_edges)
      if (Overlaps(e.right, subgraph))
        receiver->FoundSubgraphPair(subgraph, seed, edge_idx / 2);   // ★ /2 还原边下标
  ...
  NodeMap new_forbidden = forbidden | subgraph | (neighborhood & TablesBetween(0, seed_idx));
  //                                              ↑ ★ 论文漏掉、Moerkotte 后来修正的一点
  if (ExpandComplement(g, lowest_node_idx, subgraph, full_neighborhood, seed,
                       new_neighborhood, new_forbidden, receiver)) return true;
}
```

- **`forbidden` 重置**：上一层的 forbidden 节点**可能属于 complement**
- **`new_forbidden |= neighborhood & TablesBetween(0, seed_idx)`**：注释 396-404 明确指出**这是论文漏掉的、《Building Query Compilers》才修正的一点**——不这么做，`{R1,R2,R3}` 团的 complement `{R2,R3}` 会被重复枚举
- **`edge_idx / 2`**：因为边存了两份，除以 2 还原成 `JoinHypergraph::edges` 下标

### 4.5 ExpandSubgraph（425-514）—— 两趟循环保证 DP 拓扑序

```cpp
// 第一趟：对所有已连通的增长子图，枚举 complement
for (NodeMap grow_by : NonzeroSubsetsOf(neighborhood)) {
  NodeMap grown_subgraph = subgraph | grow_by;
  if (receiver->HasSeen(grown_subgraph)) {          // ★ 连通性判定 + DP 表查询
    NodeMap new_neighborhood = FindNeighborhood(g, grown_subgraph, forbidden, grow_by, ...);
    new_neighborhood |= forbidden & ~TablesBetween(0, lowest_node_idx);   // 修补
    new_neighborhood |= neighborhood;
    if (EnumerateComplementsTo(...)) return true;
  }
}
// 第二趟：才递归继续长大
for (NodeMap grow_by : NonzeroSubsetsOf(neighborhood)) {
  NodeMap grown_subgraph = subgraph | grow_by;
  NodeMap new_forbidden = (forbidden | neighborhood) & ~grown_subgraph;
  ...
  if (ExpandSubgraph(...)) return true;
}
```

**为什么必须两趟**（注释 486-488）：*"We need to do this after EnumerateComplementsTo() has run on all of them … to guarantee that we will see any smaller subgraphs before larger ones."*——**这是 DP 拓扑序的保证**。

**`HasSeen` 承担双重职责**：既查 DP 表是否已有该子集的 access path，又当**连通性判定**（注释 445-447：*"The candidate subgraphs that are connected will previously have been seen as csg-cmp-pairs, and thus, we can ask the receiver!"*）。

### 4.6 TryConnecting（532-565）—— 剪枝

```cpp
template <class Receiver>
bool TryConnecting(const Hypergraph &g, NodeMap subgraph,
                   NodeMap subgraph_full_neighborhood,
                   NodeMap complement, Receiver *receiver) {
  for (size_t node_idx : BitsSetIn(complement & subgraph_full_neighborhood)) {   // ★ 剪枝
    ...
    if (IsolateLowestBit(e.left) == node && IsSubset(e.left, complement) &&
        IsSubset(e.right, subgraph))
      receiver->FoundSubgraphPair(subgraph, complement, edge_idx / 2);
  }
}
```

**剪枝**：只遍历 `complement & subgraph_full_neighborhood`——连接边必须触及 subgraph 的 E↓'，不在这个集合里的 complement 节点不可能有连接边。

---

## 五、CostingReceiver

定义在 **`join_optimizer.cc:165`**（注释 :148 起），实例化 `:6526`。

**核心机制**：

- `m_access_paths`（`:309`）= `mem_root_unordered_map<NodeMap, AccessPathSet>`：**每个连通子图保留一组 Pareto 最优候选路径**
- `ProposeAccessPath()`（`:4550`）通过 `CompareAccessPaths()`（`:4635`）保留非支配路径
- DPhyp 回调：`FoundSingleNode()`（`:667`）、`FoundSubgraphPair()`（`:3020`）

**代价模型与旧优化器共用** `Cost_model_server`（证据 `:6521-6523`：`node.table->init_cost_model(thd->cost_model())`）。

**propose 函数族**（都在 `join_optimizer.cc`）：`ProposeTableScan`(:2185)、`ProposeIndexScan`(:2334)、`ProposeRefAccess`(:1788)、`ProposeIndexMerge`(:1579)、`ProposeHashJoin`(:3259)、`ProposeNestedLoopJoin`(:3827)、`ProposeAccessPath`(:4550)。

**几处与旧优化器的重要差异**：
- **不做 const table**（`:6399` 的 assert）
- **不用 weedout**（`:396` 注释）
- **每个 propose 会跑两轮** `{false, true}`（物化 / 不物化子查询），由 `materializable_predicates` 位图驱动
- **Pareto 前沿而非单一最优**：同时考虑 cost / init_cost / ordering / parameterization

---

## 六、图简化

`graph_simplification.cc`，实现 [Neu09]《Query Simplification: Graceful Degradation for Join-Order Optimization》。

**触发**：`EnumerateAllConnectedPartitions` 返回 true 有两种含义：(a) 出错；(b) **超过 `optimizer_max_subgraph_pairs` 上限**（由 `CostingReceiver::FoundSubgraphPair` 主动 `return true`，`join_optimizer.cc:3027`）。

若是 (b)：

```cpp
SimplifyQueryGraph(thd, thd->variables.optimizer_max_subgraph_pairs, &graph, trace);
...
receiver = CostingReceiver(..., /*subgraph_pair_limit=*/-1, ...);   // ★ 整个 receiver 重建
if (EnumerateAllConnectedPartitions(graph.graph, &receiver) && thd->is_error()) return nullptr;
```

**做法**：评估相邻 join 对的顺序收益，若"必须 A 在 B 前"，就把 A 并入 B 的 hyperedge，禁止 B-before-A，缩小搜索空间。

**为什么要全量 reset receiver 重跑**（注释 6550-6558）：简化后每个子图可能产生更多 Pareto 前沿上的 access path，重用来反而更慢。

> 注释明确（`:54-59`）：它只解决"子图对爆炸"，**不解决单子图内候选路径过多**。

---

## 七、FinalizePlan

`FinalizePlanForQueryBlock()` 在 **`finalize_plan.cc:656`**（声明 `join_optimizer.h:145`）。

**注意**：FILTER/SORT/AGGREGATE/LIMIT 等节点**不是**在这里补的，它们已在 `FindBestQueryPlan` 后处理阶段加入（见第二节）。FinalizePlan 做的是"落地"：

- EQ_REF 点查直接返回（`:662`）
- 合并叠放的 FILTER（`:678-700`）
- 后序遍历 AccessPath 树（`:707-758`）：`DelayedCreateTemporaryTable()`(:713)、`UpdateReferencesToMaterializedItems()`(:717)、`FinalizeWindowPath()`(:719)、设置 aggregator(:722)
- `push_to_engines()`（`:760`）

---

## 八、深潜：候选锦标赛与 LogicalOrderings

### 8.1 澄清：只有 DPhyp，没有 DPccp

全目录搜索 `DPccp` **零命中**——MySQL 8.0 只实现了 DPhyp（`subgraph_enumeration.h:667`，注释引 Moerkotte & Neumann CIDR 2021）。两者都枚举 csg-cmp-pair，区别：DPccp（Moerkotte 2006）需**预处理连通子图集合**（代价 O(#ccp)），DPhyp 用"最小节点 + forbidden + 邻域增量增长"**直接生成**，无需预处理。真正的"切换"只有两处：无超边时单表快路径直调 `FoundSingleNode(0)`（`join_optimizer.cc:6533-6539`）；超限后 `SimplifyQueryGraph` 重建 receiver 重跑（`:6540-6575`）。

### 8.2 AccessPathSet：DP 表兼支配结构

`CostingReceiver::m_access_paths`（`join_optimizer.cc:309`）是 DP 表，也是支配结构：

```cpp
struct AccessPathSet {   // :277-297
  vector<AccessPath *> paths;
  FunctionalDependencySet active_functional_dependencies;  // 按子图共享
  bitset obsolete_orderings;
  bool always_empty;
};
```

`HasSeen`（`:212-214`）即 `m_access_paths.count(subgraph)`——DP 表兼连通性判定。`root_candidates()`（`:221-226`）取全节点子图的 paths。FD 集合按子图存一份共享值，`FoundSubgraphPair`（`:3080-3083`）合并 `left | right | edge->functional_dependencies`。

### 8.3 ProposeAccessPath 锦标赛（`:4550-4705`）

所有 `ProposeXxx` 只做一件事：**构造一个候选交给锦标赛，不立即替换**。判定逻辑：

- 空表（always_empty）直接收下（`:4578-4588`）
- 否则对每个已有路径 `CompareAccessPaths`：
  - `SECOND_DOMINATES` / `IDENTICAL` → 丢新路径
  - `FIRST_DOMINATES` → 记录插入位并删除被支配者
- 最后 `CommitBitsetsToHeap`（`:4703`）——配合 `m_overflow_bitset_mem_root`（`:426-445`）避免每个路径都分配堆内存（大部分 bitset 很小，溢出才上堆）

### 8.4 CompareAccessPaths 多维支配（`:4134-4240`）

| 维度 | 规则 |
|------|------|
| `parameter_tables` | **子集偏序**：参数更少者优（`:4150-4157`） |
| ordering | 参数化路径的 `ordering_state` 视为 **0**（`:4165-4166`，Postgres 技巧：NLJ 破坏右侧序，参数化路径必在内侧） |
| ordering 双向 | `MoreOrderedThan` 双向比较（`:4167-4174`） |
| rowid 安全 | `safe_for_rowid`（`:4180`）、`immediate_update_delete`（`:4188`） |
| 代价 | cost/init_cost/rescan/rows 用 **1.01 模糊因子**（`:4198-4216`） |

模糊规则：两边都 better → `DIFFERENT_STRENGTHS`（**共存**，都保留）；模糊相等 → `SLIGHTLY_BETTER` 再细分。

### 8.5 sort-ahead：ProposeAccessPathWithOrderings（`:4736-4844`）

先 `emplace` 进 `m_access_paths`（**assert 同一 nodes 的 fd_set/obsolete_orderings 必须一致**，`:4749-4750`）；`ZERO_ROWS` 清空候选并置 always_empty（`:4760-4767`）；随后对 `m_sort_ahead_orderings` 逐一：`ApplyFDs(SetOrder(idx), fd_set)` 得新 state（`:4813-4814`）→ `MoreOrderedThan` 判定（`:4815`）→ 值得就构造**无 filesort** 的 SORT 路径再 ProposeAccessPath。参数化路径不做 sort-ahead（`:4792-4794`）。

### 8.6 FoundSubgraphPair 细节（`:3020-3218`）

- 超限 `++m_num_seen_subgraph_pairs > limit` 主动 `return true` 触发图简化（`:3027-3032`）
- `PassesConflictRules` 过滤（`:3039`）
- commutative 时**固定小表做 hash 侧**（`:3154-3165`）
- **semi→inner 重写**：条件 `ordering_idx_needed_for_semijoin_rewrite != -1`（`:3055-3057`），join 后该 ordering 标 obsolete（`:3086-3091`）
- `always_empty` 时先建 join 路径再包 `ZERO_ROWS`（`:3191-3211`）

### 8.7 LogicalOrderings：FD 推导与"序是否已隐含"

**FD 三个来源**（`build_interesting_orders.cc`）：

| 来源 | 条件 | always_active |
|------|------|--------------|
| ① join 条件（`:155-182`） | 仅 INNER/STRAIGHT_INNER/SEMIJOIN 的等值/join 条件（**外连接不成立**，`:141-147`） | false |
| ② 非 join 谓词（`:196-209`） | `always_active = TES 单表且无 PSEUDO_TABLE_BITS`（`:200-202`） | 视情况 |
| ③ 唯一索引（`:211-257`） | `HA_NOSAME` 且无 `HA_NULL_PART_KEY`，keypart→其余字段 | **true** |

`AddFunctionalDependencyFromCondition`（`:74`）只认 `IS NULL` 和 `EQ_FUNC`：`item=const → {}→item`。

**合成（NFSM，`interesting_orders.cc:1476-1590 BuildNFSM`）**：NFSM 状态 = 一个 Ordering；初始态有到每个可产 ordering 的"构造边"；BFS 展开：decay FD（epsilon）把 kOrder→grouping/rollup、去尾元素（`:1521-1548`）；普通 FD 用 `FunctionalDependencyApplies` 找 head 匹配的起点（`:1562`）；EQUIVALENCE 是**替换**同位置元素（`:1573-1585`）。

**NFSM→DFSM（`:1960`）**：标准子集构造；初始 DFSM 态含 ε-闭包（`ExpandThroughAlwaysActiveFDs`，`:1977-1978`）；`ApplyFDs`（`:360-380`）循环走 `next_state[fd_idx]` 至不动点。

**序蕴含判定的本质——查表不推导**：`DoesFollowOrder`（h:456-466）只查 `follows_interesting_order` bitset；`MoreOrderedThan`（h:493-506）额外比较 `can_reach_interesting_order`（未来可达），并支持 `obsolete_orderings` 掩码。**"某序是否已隐含"就是比较 DFSM 态的两个预计算 bitset**，不是逐个推导序。

**剪枝三件套**：`ImpliedByEarlierElements`（`:825-868`，查重复元素 + FD head⊆prefix + EQUIVALENCE 双向）；`PreReduceOrderings`（`:892`，只对 always-active FD 安全归约——Neu04 论文缺陷的补丁）；`PruneUninterestingOrders`（`:392-448`，把不能变有趣的索引序逐尾缩短并重映射）；homogenized ordering（h:649-662）让等价列产生的单表序互不可支配，减少 Pareto 前沿候选数。

## 九、已知限制

`CheckSupportedQuery`（`join_optimizer.cc:4846`）本版本**非常精简，只有一条**：

```cpp
if (thd->lex->m_sql_cmd != nullptr &&
    thd->lex->m_sql_cmd->using_secondary_storage_engine() &&
    !Overlaps(EngineFlags(thd), MakeSecondaryEngineFlags(
        SecondaryEngineFlag::SUPPORTS_HASH_JOIN,
        SecondaryEngineFlag::SUPPORTS_NESTED_LOOP_JOIN))) {
  my_error(ER_HYPERGRAPH_NOT_SUPPORTED_YET, MYF(0), "the secondary engine in use");
  return true;
}
```

其它报 `ER_HYPERGRAPH_NOT_SUPPORTED_YET` 的地方（**不在** CheckSupportedQuery 内）：

- 非 debug 构建开开关（`sys_vars.cc:3434`）
- `EXPLAIN FORMAT=TRADITIONAL`（`opt_explain.cc:1915`、`:2274`）—— **hypergraph 只支持 TREE/JSON**

架构层面的限制（来自 assert 与注释）：不做 const table、不在优化期求值子查询、不用 weedout。

---


## 参考

**论文**
- **Moerkotte & Neumann《Dynamic Programming Strikes Back》(CIDR 2021)** —— **DPhyp**。`EnumerateAllConnectedPartitions` 直接实现此论文
- **Moerkotte et al.《On the correct and complete enumeration of the core search space》([Moe13])** —— **CD-C 算法**。`FindHyperedgeAndJoinConflicts` 源码注释明说 "almost verbatim"
- **Moerkotte《Building Query Compilers》(treatise)** —— 补正了 DPhyp 论文漏掉的 complement 枚举细节（`EnumerateComplementsTo` 的 `new_forbidden`）
- **Neumann [Neu09]《Query Simplification: Graceful Degradation for Join-Order Optimization》** —— `graph_simplification.cc` 实现此论文
- **Graefe《The Cascades Framework》(1995)** —— 设计参照系


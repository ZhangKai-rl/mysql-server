# 06 Join Order 搜索：贪心 + 限深 DFS

> 物理优化③的另一半：**决定表的连接顺序**。访问方法的代价计算见 07 篇，semi-join 策略嵌入见 03 篇。
>
> ★ 本篇按**算法**讲。`POSITION` / `best_ref` 这些结构在对象模型中的位置与完整生命周期见 [16 篇](../16_join_object_model.md)第四章；"join order"作为一个**决策点**的性质、判据与可覆盖性见 [17 篇](../17_optimizer_decisions.md)第二章。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、算法选择与 search_depth](#一算法选择与-search_depth)
- [二、总入口：choose_table_order](#二总入口choose_table_order)
- [三、greedy_search：外层贪心框架](#三greedy_search外层贪心框架)
- [四、best_extension_by_limited_search 主循环](#四best_extension_by_limited_search-主循环)
- [五、三种剪枝的精确条件](#五三种剪枝的精确条件)
- [六、数据结构：POSITION / best_ref / best_positions](#六数据结构position--best_ref--best_positions)
- [七、condition filtering 在 join order 中的作用](#七condition-filtering-在-join-order-中的作用)
- [八、半连接与物化表的嵌入](#八半连接与物化表的嵌入)
- [九、与 hypergraph 的分流](#九与-hypergraph-的分流)
- [十、相关参数](#十相关参数)

---

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 为什么是"限深 DFS + 剪枝"，而不是 System R / DPccp

首先要纠正一个常见说法：**它不是"纯贪心"，而是一个可以把穷举度从 1 调到 N 的连续旋钮**（`search_depth` 就是那个旋钮）。源码注释给了复杂度：`O(N * N^search_depth / search_depth)`，当 `search_depth >= N` 时为 **`O(N!)`**。

而且 **N 不是表数，而是"非 eq_ref 表 + eq_ref 组"数**——这个定义本身就是为控制爆炸做的。

**为什么不是 DP**（源码没有一行明说，但结构上有硬证据）：

- 老优化器**没有按子集索引的 DP memo 数组**——只有 `join->positions[]`（单条路径）、`best_positions`（一个最优完整计划）、`best_read`（一个全局上界）。`table_map` 只用于"剩余表集合"和依赖判断
- ⇒ 它是 **DFS + 分支限界（branch & bound）**，不是 DP
- **收益**：内存 O(N) 而非 O(2^n)；规划时间可用 `search_depth` 硬性封顶
- **代价**：没有最优子结构复用，同一子集在不同路径被重复计算；且**默认就放弃了最优性**

**放弃最优性的官方表述**：`optimizer_prune_level` 的定义文本——"0 - do not apply any heuristic, thus perform **exhaustive** search; 1 - prune plans based on number of retrieved rows"；以及剪枝注释——"This heuristic **may miss the optimal QEPs**"。

#### 2. 61 表上限的来历与设计关系

`table_map = uint64_t`，`MAX_TABLES = 64 - 3 = 61`（3 位被 `INNER_TABLE_BIT` / `OUTER_REF_TABLE_BIT` / `RAND_TABLE_BIT` 占用）。三层关系：

1. **位图是老优化器一切效率的前提**（O(1) 集合运算），也是 61 上限的来源
2. **61 这个数直接决定了"必须贪心"**：61! 是天文数字，仅靠限深不够，必须有剪枝
3. **位图是稀缺资源**：连"要不要把子查询拍平成 semi-join"都要按启发式优先级抢名额（每个 nest 还要**预留一个物化槽位**）

#### 3. 左深树假设的代价

源码证据（`sql_executor.cc`）："The join tree in MySQL is generally a **left-deep tree**"；遇到非左深时——"which we **cannot handle at this point**"，处理办法是**关掉功能**（BKA / hash join 直接关）而不是支持它。

- **收益**：计划表示退化成长度 N 的数组 → 执行器"从左到右卷起来" → 搜索只需枚举排列 → 空间 O(N)，且能复用"前缀代价单调"做分支限界
- **代价**：**bushy 计划完全不可达**（如 `(A⋈B)⋈(C⋈D)` 这种两个分支各有选择性谓词的形状）；右深/深树也不可达

**hypergraph 的演进动机**（DPhyp）：① bushy 可达（枚举连通子图对）；② 把"哪些顺序合法"从搜索过程**移出去、编码进图结构**（外连接/反连接的重排序限制用**超边**表达）。

> **新老设计的分水岭**：老优化器把左深当成**全局不可动摇的结构假设**；hypergraph 把左深当成**一类特定计划（索引查找）上可证明无损的空间缩减技巧**（借自 Postgres："such plans never gain anything from being bushy"）。

#### 4. 三种剪枝为什么都要（互补关系）

| | 触发条件 | 安全性 | 砍掉什么 |
|---|---|---|---|
| **cost 剪枝** | `prefix_cost >= best_read`，**不受 prune_level 管**，需 `found_plan_with_allowed_sj` | **安全**（依赖前缀代价单调） | 已比全局最优差的分支 |
| **启发式剪枝** | `prune_level==1` 且兄弟候选在 rows + cost 双维被支配 | **不安全**（注释承认） | 同层被支配的兄弟前缀 |
| **eq_ref 扩展** | `prune_level==1 && key!=null && rows_fetched<=1.0` | 近似安全（1:1 连接代价与顺序无关） | 等价排列（**直接降 N**） |

**为什么缺一不可**：

- **cost 剪枝砍不动"等价物"**：eq_ref 排列的代价完全相同，永远只满足 `==` 不满足 `>=`，一根都剪不掉 ⇒ 必须由 eq_ref 扩展处理
- **cost 剪枝在找到第一个完整计划前几乎不工作**（`best_read` 初值 `DBL_MAX`），且只沿 cost 一维 ⇒ 启发式补"rows + cost 双维支配"
- 启发式不安全 ⇒ 由 `prune_level` 显式开关，留 `prune_level=0` 当"后悔药"
- eq_ref 扩展只适用唯一键 1:1 的**连续链**，对普通表无能为力

> **注意**：`prune_level=0`（"exhaustive"）时 cost 剪枝**依然生效**——所以 MySQL 的"穷举"实际是"穷举 + 分支限界"。

**源码自认的启发式性**：启发式剪枝里有 `/* TODO: What is the reasoning behind this condition? */`；`almost_equal` 的 10% 容差是因为"存储引擎统计是浮点近似值，严格 `==` 会全部失配"（另留 `@todo`：更好的做法是看索引是否 unique，而不是比数字）。

#### 5. `found_plan_with_allowed_sj` 为什么必须（正确性约束，不是优化）

`join->best_read` 是全局代价上界。若当前 best plan 用了一个**被 `optimizer_switch` 禁用的策略**（如关掉 DuplicateWeedout 时的 DW），那 `best_read` 就是"**不可用计划的代价**"。拿它当上界会：把所有本来合法、但代价略高的分支全部剪掉，**最后可能连一个可用计划都找不到**。

所以条件写成 `prefix_cost >= best_read && found_plan_with_allowed_sj`——**只有手上已有"策略全部可用"的计划时，`best_read` 才是可信上界**。且 `greedy_search` 每轮重置 `best_read = DBL_MAX; found_plan_with_allowed_sj = false`，不跨轮继承。

#### 6. 为什么 `best_rowcount` / `best_cost` 必须是局部变量

它们在 `best_extension_by_limited_search()` 函数体开头声明（初值 `DBL_MAX`），因为该函数**递归**——每进一层新建一对。真实语义是"**当前前缀的这一个扩展层里，已试过的兄弟候选中最好的那个**"。

**如果改成全局会怎样**：一旦某个深度较浅、代价很小的前缀被记录下来，后续所有更深的（本来就更大代价的）前缀都会立刻"被支配"而全部剪掉——**搜索会在第一个候选之后立即终止**。这不是激进剪枝，是**直接失效**。

对比：`best_read` 全局（完整计划的代价可比）；`best_rowcount/best_cost` 必须每层独立（**不同前缀、不同深度的部分计划根本不可比**）。且局部变量随栈帧丢弃、无需回滚——而共享数组 `best_ref[]` 就必须显式 `memcpy` 保存/恢复。

#### 7. 为什么 `std::swap` 无条件执行

注释原文：

> **Don't move swap inside conditional code**: All items should be uncond. swapped to maintain '#rows-ordered' best_ref[]. **This is critical for early pruning of bad plans.**

三个要点：

1. 这是一条明确的**防回归警告**（防止后人把 swap 挪进 `if` 里）
2. 维持 `best_ref[]` 按**访问行数升序** ⇒ 先试小表/选择性强的表 ⇒ 先得到低代价前缀 ⇒ `best_read` 很早被拉紧 ⇒ cost 剪枝才能"early"地大杀四方
3. 顺序乱了则第一个试到的可能是巨大扇出的前缀，`best_read` 长期虚高，**剪枝形同虚设**

有序性的来源是 `Join_tab_compare_default`——**先保证拓扑/键依赖合法，再按 `found_records` 升序，最后用指针地址 tie-break 保证排序确定性**。

> 补充：`greedy_search` 提交时用 `memmove`（稳定旋转），`best_extension` 探索时用 `std::swap`。

#### 8. "找不到可用表"时没有回退——靠前置条件保证

循环体的 `if` 可能对所有表都不成立，此时函数什么也不做地返回、`best_read` 仍为 `DBL_MAX`。而 `greedy_search` 的处理是**断言**（`assert(join->best_read < DBL_MAX)`），不是回退。

正确性外包给两件事：

1. `Join_tab_compare_default` 的**拓扑排序**（`dependent` 优先）
2. `JOIN::propagate_dependencies()` 用 **Warshall 算法 O(N³)** 构造依赖传递闭包

⇒ 用一次性 O(N³) 预处理，换取搜索过程中"**永不碰壁**"。这就是主循环能写得这么乐观的原因。

### 理论溯源

join 枚举的三代算法（完整谱系见 [`../00_overview.md`](../00_overview.md) 的「优化器的理论谱系」）：

| 算法 | 搜索空间 | 能否 bushy | MySQL |
|---|---|---|---|
| **System R（Selinger 1979）** | 左深树 + 按子集 DP，O(2^n)，用 **interesting orders** 扩展状态 | ❌ | **思想上继承**（代价模型/左深树/选择率），**算法上未采用**——没有 memo 表，是 DFS + 分支限界 |
| **DPccp（Moerkotte 2006）** | 枚举 **csg-cmp-pair**，O(3^n) | ✅ | 未直接实现 |
| **DPhyp（Moerkotte & Neumann, SIGMOD 2008）** | DPccp + **超图**（超边编码非内连接的重排序限制） | ✅ | hypergraph 的 `EnumerateAllConnectedPartitions` 直接实现此论文 |
- **CD-C 冲突规则（Moerkotte et al.）**：把"哪些顺序合法"编码进图结构

#### System R 的遗产在旧优化器里的落点

旧优化器"思想上继承 System R、算法上未采用"（上表第一行）。具体对应关系：

| 理论要素 | 源码落点 | 关键证据 |
|---|---|---|
| **代价模型** | `Cost_model_table`：`page_read_cost` / `row_evaluate_cost` / `key_compare_cost` | `page_read_cost` 按 `table_in_memory_estimate()` 把页数拆成"在内存"和"在磁盘"两部分加权——IO+CPU 公式的具体实现 |
| **选择率独立连乘** | `Item_cond_and::get_filtering_effect` | 注释直接写明 *"Conjunction of independent events: P(A and B ...) = P(A) * P(B) * ..."*；OR 同理用容斥 |
| **左深树** | `best_extension_by_limited_search` | 状态 = `join->positions[idx]`（长度 idx 的前缀）+ 剩余表集合；转移只有"前缀末尾追加一张表"，递归只走 `idx + 1` |
| **interesting orders** | **弱化实现**：`keys_in_use_for_order_by` → `test_skip_sort` → `test_if_skip_sort_order` → `test_if_cheaper_ordering` | ⚠️ 顺序**不进入搜索状态**——它是表序定下来之后的一个**后处理 pass**：检查第一张非 const 表能否用索引免排序，再用 `sort_by_table` 把有序性反馈进代价/剪枝。这与 System R 把 interesting order 作为 **DP 状态维度**是不同的 |
| **"不是 DP"** | `join->positions` | 分配大小 = `table_count`（**线性**），不是 2^N；注释说 *"we also maintain a stack of join optimization states"*。唯一的"记忆"是 `join->best_read` 这个全局上界 |

> **最重要的一条**：MySQL 旧优化器**没有按子集建 DP 表**（`positions` 是线性的），它是 **DFS + 分支限界**。所以"System R 的 DP"在 MySQL 里只留下了代价模型、选择率估算、左深树这三个思想遗产，算法骨架并不相同。

### 他库对比与演进动机

- **两条路径并存**：由 `optimizer_switch=hypergraph_optimizer` 分流，**默认 off**。打开会告警：*"The hypergraph optimizer is **highly experimental**… Do not enable it unless you are a MySQL developer"*，且该开关名在 `optimizer_switch` 列表里被注释为 **"Deliberately not documented"**
- **官方定位**（`join_optimizer.h` 文件头）："intended to **eventually take over completely**"，但当前 "nearly feature complete, but… **a very simplistic cost model**"
- ⇒ 老优化器不是"遗留垃圾"，而是当前**唯一**支持 Hints、EXPLAIN TRADITIONAL/JSON、UPDATE、临时表聚合、MIN/MAX 优化的路径
- **行为兼容性**：prepared statement / 存储程序用的是"**准备时**"的优化器，之后每次执行不再检查开关（同一条 PS 不会因开关变化突然换优化器）

---

## 一、算法选择与 search_depth

MySQL 旧优化器是**贪心 + 深度受限的 DFS（backtracking）**：

```cpp
// sql/sql_planner.cc:2077
uint Optimize_table_order::determine_search_depth(uint search_depth, uint table_count) {
  if (search_depth > 0) return search_depth;
  const uint max_tables_for_exhaustive_opt = 7;   // :2081
  if (table_count <= max_tables_for_exhaustive_opt)
    search_depth = table_count + 1;               // 穷举
  else
    search_depth = max_tables_for_exhaustive_opt; // 贪心（限深 7）
  return search_depth;
}
```

- **表数 ≤ 7** → 穷举（等价于整棵 join 树 DP）
- **表数 > 7** → 贪心，每步用限深 7 的 DFS 找最优"下一张表"

> 只支持 **left-deep tree**（左深树）：每次只能把一张表加到已定前缀的右侧。这是旧优化器的根本限制，hypergraph（09 篇）要打破的正是这一点。

---

## 二、总入口：choose_table_order

### 2.1 完整调用链

```
JOIN::optimize()                          sql_optimizer.cc:610 / 696
  └─> JOIN::make_join_plan()              sql_optimizer.cc:5307
        ├─ init_planner_arrays()          sql_optimizer.cc:5314 -> :5437
        ├─ update_ref_and_keys()          :5343
        ├─ extract_const_tables()         :5362
        ├─ extract_func_dependent_tables():5365
        ├─ estimate_rowcount()            :5371 -> :5896
        └─ Optimize_table_order(...).choose_table_order()   :5394
              ├─ (straight) optimize_straight_join()  sql_planner.cc:2117
              └─ (else)     greedy_search()           sql_planner.cc:2327
                               └─ best_extension_by_limited_search()  :2718
```

> ⚠️ 版本注意：8.0.39 **没有 `Optimize_table_order::best()`**（那是 5.x 的历史名字，已合并）。唯一 public 入口是 `choose_table_order()`（`sql_planner.h:85`）。`init_planner_arrays` 也不在搜索类里，而在 `make_join_plan` 里调用。

### 2.2 `make_join_plan` 编排函数逐行剖析（sql_optimizer.cc:5307）

`make_join_plan` 是 join 规划的**编排函数**——它本身不做 join order 搜索（那是 2.3 的 `choose_table_order`），而是做**搜索前的准备 + 调用搜索 + 搜索后的收尾**。三步结构：

```cpp
bool JOIN::make_join_plan() {
  // ── 第一段：搜索前准备（建立搜索所需的一切输入）─────────────
  if (init_planner_arrays()) return true;              // :5314 初始化 best_ref/positions/join_tab

  if (query_block->outer_join || query_block->is_recursive()) {  // :5318
    if (propagate_dependencies()) { ... }              // :5319 外连接依赖传播（非法 join order 报错）
    init_key_dependencies();                           // :5335
  }

  if (where_cond || query_block->outer_join) {         // :5342
    if (update_ref_and_keys(thd, &keyuse_array, ...))  // :5343 构建 Key_use（ref 访问的基础）
      return true;
  }

  if (!sj_pullout_done && !sj_nests.empty() &&         // :5353
      pull_out_semijoin_tables(this))                  // :5354 拉出 semi-join 表
    return true;

  if (!(active_options() & OPTION_NO_CONST_TABLES)) {  // :5360
    if (extract_const_tables()) return true;           // :5362 检测 const 表（0/1 行）
    if (extract_func_dependent_tables()) return true;  // :5365 函数依赖表
  }

  if (estimate_rowcount()) return true;                // :5371 首次 fanout 估计

  if (opt_hints_qb && !(active_options() & SELECT_STRAIGHT_JOIN))  // :5377
    opt_hints_qb->apply_join_order_hints(this);        // :5379 应用 join order hint

  if (sj_nests) {                                      // :5381
    set_semijoin_embedding();                          // :5382
    update_semijoin_strategies(thd);                   // :5383
  }

  if (!plan_is_const()) optimize_keyuse();             // :5386 优化 key 使用

  if (sj_nests && optimize_semijoin_nests_for_materialization(this))  // :5390
    return true;

  // ── 第二段：核心搜索（greedy + 限深 DFS）────────────────────
  if (Optimize_table_order(thd, this, nullptr).choose_table_order())  // :5394
    return true;

  // ── 第三段：搜索后收尾（把最优 join order 落成 QEP_TAB）──────
  if (query_expression()->item && decide_subquery_strategy())  // :5401 子查询 In-to-exists vs 物化
    return true;

  refine_best_rowcount();                              // :5403 精化行数

  positions = nullptr;                                 // :5405 保留 best_positions

  if (get_best_combination()) return true;             // :5408 ★ 把最优顺序落成 JOIN_TAB 数组

  if (materialized_derived_table_count) finalize_derived_keys();  // :5411

  if (const_table_map != found_const_table_map)        // :5420
    zero_result_cause = "no matching row in const table";
  return false;
}
```

**逐段解释**（为什么是这个顺序）：

1. **第一段是"喂料"**：`choose_table_order`（第二段）要回答"下一张表选谁最便宜"，它需要三样输入——**依赖关系**（② `propagate_dependencies`，决定哪些表能先 join）、**每张表的访问代价**（③ `update_ref_and_keys` 建 Key_use + ⑤ const 表检测 + ⑥ 行数估计）、**每张表的候选集合**（⑧ semi-join 策略）。所以第一段本质是"**把 choose_table_order 需要的一切提前算好**"。

2. **⑤ const 表检测必须在搜索前**：`extract_const_tables` 把 `WHERE pk=常量` 的表直接读出（0 或 1 行），它们在 join order 里**固定在最前面**（`const_tables` 前缀），根本不参与搜索。这就是为什么 `choose_table_order` 里 `join->const_tables` 是已知的、搜索从 `const_tables` 之后开始。

3. **⑥ `estimate_rowcount` 的"首次"性**：这里只是第一遍 fanout 估计（用 `records_per_key` 粗估），真正精确的代价在 `choose_table_order` → `greedy_search` → `best_access_path` 里逐表算。所以这里叫 "first estimate"。

4. **⑪ 是整个函数的灵魂**：`Optimize_table_order(...).choose_table_order()` 临时构造一个搜索对象，跑完 greedy/DFS 后析构。**make_join_plan 本身没有搜索逻辑**，它只是"编排者"——把准备（①~⑩）、搜索（⑪）、落盘（⑫~⑮）串起来。

5. **⑭ `get_best_combination` 是搜索的"出口"**：`choose_table_order` 产出的是 `best_positions`（抽象的顺序），`get_best_combination` 把它翻译成真正的 `JOIN_TAB` 数组（含 semi-join nest 展开、物化表插入），之后 `create_access_paths` 才能读它。

**一句话**：`make_join_plan` = **喂料（依赖/Key_use/const/行数）→ 搜索（choose_table_order）→ 落盘（get_best_combination）**。理解了这三段，就知道它为什么叫 "make **plan**"——它是"规划"这个动作的总调度，而 `choose_table_order` 只是其中的"选顺序"环节。

### 2.3 `choose_table_order`（sql_planner.cc:1950）

```cpp
bool Optimize_table_order::choose_table_order() {
  got_final_plan = false;
  // 给 const 表打一致的 prefix_cost 基准
  for (uint i = 0; i < join->const_tables; i++)
    (join->positions + i)->set_prefix_cost(0.0, 1.0);   // :1956-1957

  if (join->const_tables == join->tables) {              // :1960 全 const 短路
    memcpy(join->best_positions, join->positions, sizeof(POSITION) * join->const_tables);
    join->best_read = 1.0; join->best_rowcount = 1;
    got_final_plan = true; return false;
  }

  const bool straight_join =
      join->query_block->active_options() & SELECT_STRAIGHT_JOIN;  // :1971-1972

  if (emb_sjm_nest) {                                    // :1975 半连接物化 nest
    merge_sort(join->best_ref + join->const_tables,
               join->best_ref + join->tables, Join_tab_compare_embedded_first(emb_sjm_nest));
    join_tables = emb_sjm_nest->sj_inner_tables;
  } else {
    if (straight_join)                                   // :1992
      merge_sort(join->best_ref + join->const_tables,
                 join->best_ref + join->tables, Join_tab_compare_straight());
    else                                                 // :1995
      merge_sort(join->best_ref + join->const_tables,
                 join->best_ref + join->tables, Join_tab_compare_default());
    join_tables = join->all_table_map & ~join->const_table_map;
  }
  ...
  if (straight_join)                                     // :2025
    optimize_straight_join(join_tables);
  else {
    if (greedy_search(join_tables)) return true;         // :2028
  }
  got_final_plan = true;
  if (fix_semijoin_strategies()) return true;            // :2039
  return false;
}
```

**逐段解释**：

1. `:1956` 给 const 表占的 `positions[0..const_tables)` 打一致的 `prefix_cost=0.0` 基准，使后续 `set_prefix_join_cost` 的累加公式有统一起点。
2. `:1960` 全 const 短路：整条查询全是 const 表，`memcpy` 到 `best_positions` 即完成。
3. `:1971` **straight_join 的唯一判据**是 `active_options() & SELECT_STRAIGHT_JOIN`，**没有 `JOIN::straight_join` 成员**。
4. `:1992-1999` 关键：先用 `merge_sort` 把 `best_ref` 按"依赖 → 记录数 → 指针地址"排序（straight 则只按依赖、保持书写顺序）。**这个排序结果就是后续搜索遍历候选的顺序**——排序本身是一种贪心启发。
5. `:2025` 分流：straight → `optimize_straight_join`，否则 → `greedy_search`。
6. `:2039` 搜索完后 `fix_semijoin_strategies()` 把半连接策略从"后向表示"转成"前向表示"。

### 2.4 straight_join：`optimize_straight_join`（sql_planner.cc:2117）

```cpp
void Optimize_table_order::optimize_straight_join(table_map join_tables) {
  uint idx = join->const_tables;
  double rowcount = 1.0, cost = 0.0;
  for (JOIN_TAB **pos = join->best_ref + idx; *pos; idx++, pos++) {  // :2130
    JOIN_TAB *const s = *pos;
    POSITION *const position = join->positions + idx;
    best_access_path(s, join_tables, idx, false, rowcount, position);  // :2146 只选访问方式
    position->set_prefix_join_cost(idx, cost_model);   // :2149 累加前缀代价
    position->no_semijoin();                           // :2151 STRAIGHT_JOIN 禁半连接
    rowcount = position->prefix_rowcount;
    cost = position->prefix_cost;
    join_tables &= ~(s->table_ref->map());
  }
  memcpy(join->best_positions, join->positions, sizeof(POSITION) * idx);  // :2168
  join->best_read = cost - 0.001;                        // :2176 -0.001 是浮点可重复 hack
  join->best_rowcount = (ha_rows)rowcount;
}
```

顺序已由 `merge_sort(..., Join_tab_compare_straight())` 固定，这里**只沿 `best_ref` 单向走一遍**：对每张表选访问方式（不换顺序）、累加前缀代价、禁用半连接、最后一次性 `memcpy`。

---

## 三、greedy_search：外层贪心框架

`sql/sql_planner.cc:2327`。

```cpp
uint idx = join->const_tables;                    // const 表不参与重排，已在头部固定
do {
  join->best_read = DBL_MAX;
  if (best_extension_by_limited_search(remaining_tables, idx, search_depth)) return true;

  if (size_remain <= search_depth || use_best_so_far) {
    return false;                                 // DFS 已搜完全部表，best_positions 即完整最优
  }

  best_pos = join->best_positions[idx];           // 只取最优扩展的"第一张表"
  best_table = best_pos.table;
  join->positions[idx] = best_pos;                // 固定到部分计划

  // 把 best_table 挪到 best_ref[] 的 idx 处（memmove 腾位 + 赋值）
  best_idx = idx;
  JOIN_TAB *pos = join->best_ref[best_idx];
  while (pos && best_table != pos) pos = join->best_ref[++best_idx];
  memmove(join->best_ref + idx + 1, join->best_ref + idx,
          sizeof(JOIN_TAB *) * (best_idx - idx));
  join->best_ref[idx] = best_table;

  remaining_tables &= ~(best_table->table_ref->map());
  --size_remain; ++idx;
} while (true);
```

**逐段解释**：

1. `idx = join->const_tables`：常量表已在 plan 头部固定（见 07 篇），**不参与重排**。
2. 每次迭代调 `best_extension_by_limited_search` 做一次**限深 DFS**，结果存 `best_positions`。
3. 若 `size_remain <= search_depth`，DFS 已搜完剩余所有表，`best_positions` 就是完整最优，直接返回。
4. 否则（表太多，DFS 只搜了 7 层）**只取最优扩展的第一张表**固定下来，`best_ref` 里把它挪到 `idx`（`memmove` 腾位）。
5. `remaining_tables` 去掉这张表，`idx++` 循环——**标准贪心：每次局部最优选下一张表**。

---

## 四、best_extension_by_limited_search 主循环

`sql/sql_planner.cc:2718`。这是整个搜索的核心。

### 4.1 函数头与候选遍历

```cpp
bool Optimize_table_order::best_extension_by_limited_search(
    table_map remaining_tables, uint idx, uint current_search_depth) {
  const Cost_model_server *const cost_model = join->cost_model();
  double best_rowcount = DBL_MAX;   // 本层剪枝用的"最佳行数"基准
  double best_cost = DBL_MAX;       // 本层剪枝用的"最佳代价"基准
  table_map eq_ref_extended(0);     // 已被 EQ_REF 链吞掉并剪掉的表

  JOIN_TAB *saved_refs[MAX_TABLES];
  memcpy(saved_refs, join->best_ref + idx,
         sizeof(JOIN_TAB *) * (join->tables - idx));   // 保存 best_ref 尾部
  ...
  // :2759 —— 关键：不是位图遍历，而是遍历"按 #rows 排序"的 best_ref[]
  for (JOIN_TAB **pos = join->best_ref + idx; *pos && !use_best_so_far; pos++) {
    JOIN_TAB *const s = *pos;
    const table_map real_table_bit = s->table_ref->map();

    // swap 必须在条件判断之外无条件执行，维持 best_ref[] 的 #rows 有序性
    std::swap(join->best_ref[idx], *pos);                       // :2768

    if ((remaining_tables & real_table_bit) &&                  // 表还没用过
        !(eq_ref_extended & real_table_bit) &&                  // 没被 EQ_REF 剪掉
        !(remaining_tables & s->dependent) &&                   // 依赖已满足
        (!idx || !check_interleaving_with_nj(s))) {             // 不破坏嵌套外连接
      POSITION *const position = join->positions + idx;
      deps_lateral.restore();                                   // 弹出上次尝试的 lateral 状态

      best_access_path(s, remaining_tables, idx, false,        // :2787
                       idx ? (position - 1)->prefix_rowcount : 1.0, position);
      position->set_prefix_join_cost(idx, cost_model);          // :2791
      ...
      if (has_sj) advance_sj_state(remaining_tables, s, idx);   // :2798
      else        position->no_semijoin();
```

**逐段解释**：

1. `for (JOIN_TAB **pos = best_ref + idx; ...)`：遍历 `best_ref[idx..]`。`best_ref` 已在 `choose_table_order` 里按"记录数少优先"排序，所以**候选顺序本身是一种贪心启发**。
2. `std::swap(best_ref[idx], *pos)` 把候选表 `s` 提到前缀末尾 `idx`（逻辑上"把 s 追加到前缀"）。swap **无条件执行**（在 if 之外），让 `best_ref` 始终维持行数有序——注释明确说这对"early pruning of bad plans"至关重要。
3. 候选合法四条件：在 remaining 里、没被 eq_ref 链剪掉、依赖表都在前缀里（`!(remaining_tables & s->dependent)`）、不违反嵌套外连接交错（`check_interleaving_with_nj`）。
4. `best_access_path` 只算"把 s 加到当前前缀"的访问代价；`set_prefix_join_cost` 累加为前缀代价；`advance_sj_state`/`no_semijoin` 维护半连接状态。

### 4.2 递归与叶子回写

```cpp
      recalculate_lateral_deps_incrementally(idx + 1);
      const table_map remaining_tables_after =
          (remaining_tables & ~real_table_bit);

      if ((current_search_depth > 1) && remaining_tables_after) {  // :2860
        // EQ_REF 剪枝3：第一个 EQ_REF 才触发（见 5.3）
        ...
        // 普通深度优先递归：只追加一张表，深度-1
        if (best_extension_by_limited_search(remaining_tables_after, idx + 1,
                                             current_search_depth - 1))  // :2908
          return true;
      } else {
        if (consider_plan(idx, &trace_one_table)) return true; // :2913 叶子：记录 plan
      }
      backout_nj_state(remaining_tables, s);                   // :2923 撤销嵌套状态
    }
  }
done:
  memcpy(join->best_ref + idx, saved_refs,
         sizeof(JOIN_TAB *) * (join->tables - idx));           // :2929 恢复 best_ref
  return false;
}
```

- 递归深度 `current_search_depth-1`、索引 `idx+1`；到叶子（`current_search_depth==1` 或表用完）时调 `consider_plan` 记录 plan。
- 函数末尾 `memcpy` 恢复 `best_ref`，保证返回上层时数组顺序与进入时一致（配合 swap 的对称性）。
- `idx` 起点是 `join->const_tables`，不是 0。

### 4.3 `set_prefix_join_cost` 公式（sql_select.h:565）

```cpp
void set_prefix_join_cost(uint idx, const Cost_model_server *cm) {
  if (idx == 0) {
    prefix_rowcount = rows_fetched;
    prefix_cost = read_cost + cm->row_evaluate_cost(prefix_rowcount);
  } else {
    prefix_rowcount = (this - 1)->prefix_rowcount * rows_fetched;
    prefix_cost = (this - 1)->prefix_cost + read_cost +
                  cm->row_evaluate_cost(prefix_rowcount);
  }
  prefix_rowcount *= filter_effect;      // 条件过滤系数最后乘上
}
```

累加公式：

- `prefix_rowcount(i) = prefix_rowcount(i-1) * rows_fetched(i) * filter_effect(i)`
- `prefix_cost(i) = prefix_cost(i-1) + read_cost(i) + row_evaluate_cost(prefix_rowcount(i))`

注意 `read_cost` 是"访问当前表的 IO 代价（已乘上前缀行数）"，`filter_effect` 只缩小行数、不缩小已发生的 `read_cost`。

### 4.4 best_positions 的回写（`consider_plan`，sql_planner.cc:2555）

```cpp
if (chosen) {
  memcpy((uchar *)join->best_positions, (uchar *)join->positions,
         sizeof(POSITION) * (idx + 1));
```

是**整块 `memcpy`**，不是逐字段复制。`sizeof(POSITION) * (idx+1)` 覆盖前缀 `[0..idx]`（含 const 表前缀）。这正是 POSITION 必须是 POD 的原因（见第六节）。

---

## 五、三种剪枝的精确条件

### 5.1 剪枝 1：按代价剪（`prune_by_cost`，:2817，精确）

```cpp
if (position->prefix_cost >= join->best_read && found_plan_with_allowed_sj) {
  trace_one_table.add("pruned_by_cost", true);
  backout_nj_state(remaining_tables, s);
  continue;
}
```

- `join->best_read` 是迄今最优完整 plan 的代价，初值 `DBL_MAX`（`greedy_search` 每轮重置）。
- `prefix_cost` 单调不减，若"前缀+s"已 ≥ 最优，再往下加表只会更贵，必剪。**精确剪枝**（不漏最优）。
- `found_plan_with_allowed_sj` 是保护：若当前最好 plan 用了被禁用的半连接策略，不能剪（得继续找合法 plan）。

### 5.2 剪枝 2：启发式剪（`pruned_by_heuristic`，:2832，可能漏最优）

```cpp
if (prune_level == 1) {
  if (best_rowcount > position->prefix_rowcount ||
      best_cost > position->prefix_cost ||
      (idx == join->const_tables &&        // s 是 QEP 第一张表
       s->table() == join->sort_by_table)) {
    if (best_rowcount >= position->prefix_rowcount &&
        best_cost >= position->prefix_cost &&
        (!(s->key_dependent & remaining_tables) || position->rows_fetched < 2.0)) {
      best_rowcount = position->prefix_rowcount;
      best_cost = position->prefix_cost;
    }
  } else if (found_plan_with_allowed_sj) {
    trace_one_table.add("pruned_by_heuristic", true);
    backout_nj_state(remaining_tables, s);
    continue;
  }
}
```

- `best_rowcount` / `best_cost` 是**本层函数调用的局部变量**（初值 `DBL_MAX`），记录"本层已见、Pareto 意义上的最佳前缀"。
- 外层 if：候选"行数或代价至少一个维度优于基准" OR "是第一张表且正好是 `sort_by_table`" → 进入（不剪）。
- 内层 if：候选两维度都不差于基准、且 `!(s->key_dependent & remaining_tables) || rows_fetched < 2.0`，才更新基准。
- `sort_by_table` 特例：若候选是"我们希望按其排序的表"且是第一张表，强制不剪——因为把它放第一可能省一次 filesort。
- **这是贪心启发式，可能漏掉真正最优**（源码注释明确"非穷尽搜索"）。`prune_level == 0` 时整个块关闭，退化为穷尽。

### 5.3 剪枝 3：EQ_REF 扩展（`eq_ref_extension_by_limited_search`，:3065）

若刚加的 `s` 是 EQ_REF（`key != nullptr && rows_fetched <= 1.0`），且是**第一个** EQ_REF（`eq_ref_extended == 0`），就一次性把后续所有"同代价 EQ_REF 表"追加进来。

**核心判定 `almost_equal`（:2947，10% 容差）**：

```cpp
static inline bool almost_equal(double left, double right) {
  const double boundary = 0.1;  // 10 percent limit
  return ((left >= right * (1.0 - boundary)) && (left <= right * (1.0 + boundary)));
}
```

```cpp
const bool added_to_eq_ref_extension =
    position->key &&
    almost_equal(position->read_cost, (position - 1)->read_cost) &&
    almost_equal(position->rows_fetched, (position - 1)->rows_fetched);  // :3141-3144
```

**为什么只从第一个触发**：连续 EQ_REF 是 1:1 关系，`read_cost`/`rows_fetched` 几乎相同，**排列顺序不影响总代价**——逐表枚举这些等价排列纯属浪费 CPU。第一个 EQ_REF 触发后，链内返回被吞掉的表位图 `eq_ref_ext`，外层 `best_extension` 里 `!(eq_ref_extended & real_table_bit)` 判定为已处理，直接 `pruned_by_eq_ref_heuristic` 跳过。

**容差原因**：索引统计是浮点数、不同引擎 rec_per_key 可能不精确，允许 10% 误差仍视为"1:1 等价"；若不等价则 `added_to_eq_ref_extension=false`，回退到普通 `best_extension`。

---

## 六、数据结构：POSITION / best_ref / best_positions

### 6.1 POSITION 完整字段（sql_select.h:352）

> 注释明确：`This class has to stay a POD, because it is memcpy'd in many places.`

```cpp
struct POSITION {
  double rows_fetched;                 // :371  访问方法每次前缀行组合产出的行数
  double read_cost;                    // :382  该表在整个 join 中的 IO 代价
  float filter_effect;                 // :416  剩余条件过滤系数(0..1)
  double prefix_rowcount;              // :434  前缀输出行组合数
  double prefix_cost;                  // :435  前缀累计代价
  JOIN_TAB *table;                     // :437
  Key_use *key;                        // :443  ref/eq_ref 用的 Key_use；NULL=非 ref
  table_map ref_depend_map;            // :446  ref 依赖的前缀表
  bool use_join_buffer;                // :447  是否 BNL
  uint sj_strategy;                    // :461  半连接去重策略(只设在"产生重复范围的最后一个表")
  uint n_sj_tables;                    // :467  该策略覆盖的后续表数
  table_map dups_producing_tables;     // :474  前缀中还没去重方案的半连接内表
  uint first_loosescan_table;          // :479  LooseScan 第一张表
  table_map loosescan_need_tables;     // :484
  uint loosescan_key;                  // :492
  uint loosescan_parts;                // :493
  uint first_firstmatch_table;         // :500  FirstMatch 第一张内表
  nested_join_map cur_embedding_map;   // :505
  table_map first_firstmatch_rtbl;     // :510
  table_map firstmatch_need_tables;    // :515
  uint first_dupsweedout_table;        // :519  Weedout 第一张表
  table_map dupsweedout_tables;        // :524
  uint sjm_scan_last_inner;            // :528  SJ-Materialization-Scan 最后内表
  table_map sjm_scan_need_tables;      // :534
private:
  table_map m_suffix_lateral_deps;     // :586  table 及后续表的 lateral 依赖缓存
};
```

> 注意：旧版本字段 `sjmat_lookup_tables` / `sjmat_exhausted_tables` / `first_sj_inner_tables` 在 8.0.39 **已不存在**。

核心字段与 join order 的关系：

| 字段 | 作用 |
|------|------|
| `rows_fetched` | `selectivity(access_cond) * cardinality(table)`，用于算 fanout |
| `read_cost` | 单次访问代价 × 前缀行数（**不含** row_evaluate_cost） |
| `filter_effect` | `rows_fetched * filter_effect` 才是传给下一表的真正 fanout |
| `prefix_rowcount` | 到当前表为止的前缀输出行组合数 |
| `prefix_cost` | 前缀累计代价 |

### 6.2 POD 约束

POSITION 必须是 POD（平凡可拷贝）：

1. **不能有虚函数**（否则有 vptr，memcpy 会拷贝错误 vptr）
2. **不能有拥有所有权的指针/深拷贝成员**（`std::string`/`std::vector`/智能指针都不行）；`table`/`key` 指针只是**借用**
3. 内联成员函数不影响布局（不增加数据成员、不引入 vptr）

这决定了 `consider_plan` 用 `memcpy` 而非逐字段赋值。

### 6.3 三个数组的关系

| 结构 | 位置 | 说明 |
|------|------|------|
| `join->best_ref` | `sql_optimizer.h:164` | `JOIN_TAB**`，"候选顺序"，搜索中反复 `memmove`/`swap` 重排 |
| `join->best_positions` | `sql_optimizer.h:312` | DFS 搜出的当前最优完整计划（临时） |
| `join->positions` | `sql_optimizer.h:317` | 贪心每步固定下来的最终计划 |

分配在 `init_planner_arrays()`（`sql_optimizer.cc:5437`），其中 `best_positions` 比 `positions` 多 `sj_nests` 个槽（物化半连接 nest 需要额外槽）。

---

## 七、condition filtering 在 join order 中的作用

### 7.1 filter_effect 的计算与使用

`calculate_condition_filter`（`sql_planner.cc:1243`）在 `best_access_path` 里被调用，结果写进 `pos->filter_effect`（:1217），随后 `set_prefix_join_cost` 末尾 `prefix_rowcount *= filter_effect`。

**闭环**：过滤系数直接缩小下一张表的 fanout 基数 → 影响后续每张表 `best_access_path` 传入的 `prefix_rowcount` → 影响 ref 代价 `prefix_rowcount * find_cost_for_ref(...)` 与扫描代价 → **最终改变表选择顺序**。

是否计算的开关是 `OPTIMIZER_SWITCH_COND_FANOUT_FILTER`（默认 ON）。`choose_table_order` 里会先预计算 `TABLE::cond_set`（`:2007-2019`）——把 WHERE 里所有列标出来，避免每张表都算过滤。

### 7.2 与 07 篇的衔接

- `estimate_rowcount`（`sql_optimizer.cc:5896`）在 join order 搜索**之前**执行，为每张表设 `records`/`found_records`/`read_time`/`range_scan()`。
- `find_best_ref` 里基于 `records_per_key` 算出 `cur_fanout` 存入 `start_key->fanout`。
- `best_access_path` 读取 `best_ref->fanout` 作为 `rows_fetched`（:1028），再乘以前缀 `prefix_rowcount` 得到本表实际读取行数，用 `prefix_rowcount * find_cost_for_ref()` 把"单次 ref 代价"放大为"前缀多次执行总代价"。

即：**join order 层的 fanout 完全依赖 `find_best_ref` 里基于 `records_per_key` 的 `cur_fanout`**（详见 07 篇）。

---

## 八、半连接与物化表的嵌入

`best_extension_by_limited_search` 里除了访问方法，还要处理：

- **semi-join 策略**：每加一张表后调 `advance_sj_state`（`sql_planner.cc:4105`）判定/更新五种策略（详见 **03 篇**）
- **物化表**（sjm）：`get_best_combination` 时把 inner tables 放到 primary tables 之后，把 sjm 表放进 primary tables

搜索结束后 `fix_semijoin_strategies` 从后往前确定最终策略（03 篇 4.4 节）。

---

## 九、与 hypergraph 的分流

`sql_optimizer.cc:610`：

```cpp
if (thd->lex->using_hypergraph_optimizer()) {
  m_root_access_path = FindBestQueryPlan(thd, query_block, trace_ptr);
  return false;
}
// All of this is never called for the hypergraph join optimizer!
assert(!thd->lex->using_hypergraph_optimizer());
...
if (make_join_plan()) { ... }   // 旧 greedy 路径
```

判定函数 `LEX::using_hypergraph_optimizer()`（`sql_lex.h:3914`），由 optimizer switch `hypergraph_optimizer` 决定。

**greedy 只支持 left-deep 的代码体现**：`best_extension` 的递归只能"往前缀末尾追加一张表"（`:2908` 的 `idx+1`），`greedy_search` 每次只把 `best_table` 插到 `best_ref[idx]`。整个搜索树任意时刻的状态都是"有序前缀 + 无序剩余"，新增表只能挂前缀末尾——**没有"把两个已构建好的 partial plan 再 join 起来"的代码路径**，故只能产生 left-deep 计划。hypergraph 用 DPhyp 枚举任意连通子图对（见 09 篇），支持 bushy。

---

## 十、相关参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `optimizer_search_depth` | **62**（`MAX_TABLES+1`，0=自动） | 手动指定 DFS 深度；0 时按表数走 7 表分界。`MAX_TABLES=61`（`sql_const.h:107`） |
| `optimizer_prune_level` | **1** | 1=启用启发式剪枝（可能漏最优）；0=关闭，退化为穷尽 |

读入：`Optimize_table_order` 构造函数 `sql_planner.cc:125-141`。`optimizer_search_depth` 注册在 `sys_vars.cc:3315`（`VALID_RANGE(0, MAX_TABLES+1)`，`DEFAULT(MAX_TABLES+1)`）。

---


## 参考

**论文**
- **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979，System R)** —— 左深树 DP 与贪心搜索的经典方案

**官方文档**
- *MySQL 8.0 Reference Manual → Optimizing Queries with EXPLAIN*（`optimizer_search_depth` / `optimizer_prune_level`）
- *MySQL 8.0 Reference Manual → SELECT Statement → JOIN Clause*（STRAIGHT_JOIN）

**相关文档**
- 访问方法代价见 [`07_access_method.md`](07_access_method.md)
- semi-join 策略嵌入见 [`../logical/03_semijoin.md`](../logical/03_semijoin.md)
- hypergraph 优化器见 [`09_hypergraph.md`](09_hypergraph.md)
- 知乎：https://zhuanlan.zhihu.com/p/644832644
- https://zhuanlan.zhihu.com/p/632872022

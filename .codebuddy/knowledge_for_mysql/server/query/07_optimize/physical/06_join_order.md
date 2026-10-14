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
- [十一、ORDER BY / GROUP BY 反向约束 join order（`sort_by_table`）](#十一order-by--group-by-反向约束-join-ordersort_by_table)

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

### 搜索空间与贪心的分层（总图）

```
N 张表的 join order 搜索空间 = N! 个排列（左深树）
  t1◄t2◄t3◄t4   t1◄t2◄t4◄t3   t1◄t3◄t2◄t4  ...  共 4! = 24 棵（N=4）

greedy_search 把它切成 N 轮，每轮只"深度优先地看 search_depth 层"：

  round 1:  [ ][ ][ ][ ][ ]        剩余 {t1..tN}
            └─ best_extension_by_limited_search(idx=0, depth=search_depth)
               逐层展开                     ──► 取最优的第一个表 t_a 固定
  round 2:  [t_a][ ][ ][ ][ ]      剩余 {t1..tN} \ {t_a}
            └─ 同上                          ──► 固定 t_b
  round 3:  [t_a][t_b][ ][ ][ ]
  ...
  round N:  [t_a][t_b][...][tN]    → best_positions → get_best_combination()

  每一轮内层递归里，三道剪枝决定"这个分支还走不走"：
    ┌ prune_by_cost       前缀代价已 ≥ 当前最优（精确，不漏最优）
    ├ pruned_by_heuristic 行数/代价都不占优（prune_level=1，可能漏最优）
    └ eq_ref 链           把等值唯一键连成的链当一个实体，一次塞进多张表

  代价沿前缀累加（POSITION 是一个栈）：
    prefix_rowcount = prev.prefix_rowcount * rows_fetched * filter_effect
    prefix_cost     = prev.prefix_cost + read_cost + row_evaluate_cost(prev.prefix_rowcount * rows_fetched)

  唯一的"外部约束"：ORDER BY / GROUP BY 只落在一张表上时 → sort_by_table
    首表不是它 → consider_plan() 加 sort_cost = prefix_rowcount（软惩罚）
    首表是它   → 豁免 pruned_by_heuristic（避免被启发式砍掉）
```

★ 这张图说明两件事：① **贪心不是"每层选一个最优"那么简单**——每轮内部是完整的深度优先 + 剪枝；② **`search_depth` 决定了每轮往里看多深**，`depth ≥ N` 时它退化为穷举（8.0.39 默认 62，即默认穷举）。

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

**★★★ 但是——这条 7 表启发式默认根本不会被执行。**

```cpp
/* sql/sys_vars.cc:3315 */
static Sys_var_ulong Sys_optimizer_search_depth(
    "optimizer_search_depth", ..., 
    VALID_RANGE(0, MAX_TABLES + 1), DEFAULT(MAX_TABLES + 1), BLOCK_SIZE(1));
```

> ★★★ **`optimizer_search_depth` 的默认值是 `MAX_TABLES + 1 = 62`，不是 0。** 而 `determine_search_depth()` 第一行就是 `if (search_depth > 0) return search_depth;`——**62 > 0，直接原样返回**。
>
> 所以：**8.0.39 默认做的就是"深度 62 的限深 DFS"**，而合法查询最多 61 张表 ⇒ **默认等价于完整穷举**。只有用户显式 `SET optimizer_search_depth=0` 才会触发上面那套 7 表启发式。
>
> 这个默认值的语义是"**尽可能完整**"，兜底靠另外两样：`prune_level=1` 的三种剪枝，以及二级引擎的 `use_best_so_far`。**文档标题里常说的"7 表贪心"是默认值被误读后的结果**。

**★ `MAX_TABLES` = 61 是推导出来的，源码里没有字面量 61**：

```cpp
/* sql/sql_const.h:105 */
/** Use for sizing ONLY */
constexpr const size_t MAX_TABLES_FOR_SIZE{sizeof(table_map) * 8};   /* table_map = uint64 ⇒ 64 */
/** Max tables in join */
constexpr const size_t MAX_TABLES{MAX_TABLES_FOR_SIZE - 3};          /* ★ 64 - 3 = 61 */

constexpr const table_map INNER_TABLE_BIT{1ULL << (MAX_TABLES + 0)};
constexpr const table_map OUTER_REF_TABLE_BIT{1ULL << (MAX_TABLES + 1)};
constexpr const table_map RAND_TABLE_BIT{1ULL << (MAX_TABLES + 2)};
constexpr const table_map PSEUDO_TABLE_BITS{
    INNER_TABLE_BIT | OUTER_REF_TABLE_BIT | RAND_TABLE_BIT};
```

> ★ 减 3 是给三个"伪表位"：`INNER_TABLE_BIT`（含参数/访问表的子查询/函数的表达式）、`OUTER_REF_TABLE_BIT`（外层引用）、`RAND_TABLE_BIT`（非确定表达式）。它们的详细语义在 `sql_const.h:79-104` 的注释里。
> ★ `MAX_TABLES` 在 planner 内部还被用作**哨兵下标**（不是表数上限）：`pos->first_firstmatch_table = MAX_TABLES;`（`:4139`）表示"尚无 FirstMatch 起点"。

**★ 另一个旋钮 `optimizer_prune_level`**（两种剪枝的开关）：

```cpp
/* sql/sys_vars.cc:3305 —— 默认 1 = 启用启发式剪枝 */
static Sys_var_ulong Sys_optimizer_prune_level(
    "optimizer_prune_level",
    "Controls the heuristic(s) applied during query optimization to prune "
    "less-promising partial plans from the optimizer search space. "
    "Meaning: 0 - do not apply any heuristic, thus perform exhaustive "
    "search; 1 - prune plans based on number of retrieved rows",
    HINT_UPDATEABLE SESSION_VAR(optimizer_prune_level), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, 1), DEFAULT(1), BLOCK_SIZE(1));
```

两者在构造函数里**一次性**读入 `const` 成员（`sql_planner.cc:125`）：

```cpp
      search_depth(determine_search_depth(thd->variables.optimizer_search_depth,
                                          join->tables - join->const_tables)),
      prune_level(thd->variables.optimizer_prune_level),
```

> ★ 注意 `table_count = join->tables - join->const_tables`——**const 表不参与 join order 搜索**。且 `search_depth` / `prune_level` 都是 **`const uint`**，优化过程中不可变。

> 只支持 **left-deep tree**（左深树）：每次只能把一张表加到已定前缀的右侧。这是旧优化器的根本限制，hypergraph（09 篇）要打破的正是这一点。源码自陈见 `join_optimizer/estimate_selectivity.cc:282`：*"the old join optimizer's way of thinking, where one made a long chain of (left-deep) nested tables ... In the hypergraph optimizer, we no longer have a chain"*。

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

**★★ 三处必须澄清的细节**：

**① 那处"无条件执行"的重排是 `memmove` + 赋值，不是 `std::swap`。** `std::swap` 在 `sql_planner.cc` 里只有 2 处，都在**内层**搜索函数（`best_extension_by_limited_search:2768`、`eq_ref_extension_by_limited_search:3096`）。`greedy_search` 用的是 2412-2414 的 `memmove` 整段右移 + 头部覆写——**等价于把 `best_table` 旋转到 `idx` 位置**。

```cpp
    memmove(join->best_ref + idx + 1, join->best_ref + idx,
            sizeof(JOIN_TAB *) * (best_idx - idx));
    join->best_ref[idx] = best_table;   /* ★ 无任何 if (best_idx != idx) 守卫 */
```

> ★ 说它"无条件执行"是对的：`best_idx == idx` 时 `memmove` 长度为 0，退化为自赋值。注释原文（`:2409-2411`）：*"Maintain '#rows-sorted' order of 'best_ref[]'"*——**维持 best_ref 的"行数有序"是早期剪枝有效性的前提**。

**② `greedy_search` 里没有 `best_rowcount` / `best_cost` 局部变量。** 那两个量属于 `best_extension_by_limited_search`（`sql_planner.cc:2733-2734`），只用于 `prune_level == 1` 的启发式剪枝。`greedy_search` 自己的局部状态是 `best_pos` / `best_table` / `best_idx`。

**③ "找不到可用表"确实没有回退**——唯一保障是一条断言：

```cpp
    assert(join->best_read < DBL_MAX);   /* ★ 2353：没有 else / fallback 分支 */
```

> ★ release 版本断言被编译掉，若真出现"一个可用表都找不到"，`best_positions` 里会残留 `DBL_MAX`。**这是靠前置条件（依赖传播已保证至少有一个候选）兜住的，不是靠防御性代码。**

**★★ 每轮开始的两个重置**（★ 这是理解"贪心 + 剪枝"如何协同的关键）：

```cpp
  do {
    /* Find the extension of the current QEP with the lowest cost */
    join->best_read = DBL_MAX;          /* ★ 上界清零 */
    join->best_rowcount = HA_POS_ERROR;
    found_plan_with_allowed_sj = false; /* ★★ 合法性闸门清零 */
    if (best_extension_by_limited_search(remaining_tables, idx, search_depth))
      return true;
```

> ★★★ **`found_plan_with_allowed_sj` 每轮都被清零**，意味着**每轮开始时两种剪枝的闸门是关着的**，直到 `consider_plan()` 找到第一个"只用允许策略"的完整计划才打开。这是刻意的：否则一旦先搜到一个"用了被禁用策略但更便宜"的计划，剪枝会立刻把通往合法计划的分支全剪掉。

**★ 预排序：三个比较器**（★ 只在 `choose_table_order()` 里做**一次**，搜索函数内不再调用）：

```cpp
bool Join_tab_compare_default::operator()(const JOIN_TAB *jt1,
                                          const JOIN_TAB *jt2) const {
  if (jt1->dependent & jt2->table_ref->map()) return false;   /* ① 依赖序 */
  if (jt2->dependent & jt1->table_ref->map()) return true;

  const bool jt1_keydep_jt2 = jt1->key_dependent & jt2->table_ref->map();
  const bool jt2_keydep_jt1 = jt2->key_dependent & jt1->table_ref->map();
  if (jt1_keydep_jt2 && !jt2_keydep_jt1) return false;        /* ② key 依赖序 */
  if (jt2_keydep_jt1 && !jt1_keydep_jt2) return true;

  if (jt1->found_records > jt2->found_records) return false;  /* ③ 小表优先 */
  if (jt1->found_records < jt2->found_records) return true;

  return jt1 < jt2;                                          /* ④ 指针地址兜底 */
}
```

> ★★ **这个序关系不是传递的**——`sql_planner.h:264` 的注释给了反例：`a: records=3`、`b: records=4`、`c: dependent=b, records=2`，则 `a<b`（记录少）、`b<c`（c 依赖 b）、`c<a`（记录少）形成环。
>
> ★★ 正因为它不传递，**不能用 `std::sort`**（不同平台实现不同会导致 EXPLAIN 漂移，MTR 会挂）。MySQL 为此在 `sql/merge_sort.h` 自己实现了 `merge_sort()`（元素 < 5 退化成 `insert_sort`），文件头注释原文就说明了这个理由。

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

**★★★ 纠正：EQ_REF 扩展**省的是"排列"，不是"深度"。**调用点 2885 与内部自递归 3191 **都**传 `current_search_depth - 1`——深度照常消耗。Doxygen（`:2976-2979`）的说法才是准确的：

> *"this chain of eq_refs will be handled as a single entity wrt. the full 'greedy' exploration ... This will reduce the 'N' in the O(N!) complexity"*

它做的是：**把一条 eq_ref 链当成一个实体**，链内每张表一旦 `almost_equal()` 成立就顺序附加并**立刻 return**（`:3226`），不再对链内 N 张表做 O(N!) 排列。返回后被吃掉的表写进 `eq_ref_extended`，主循环再用 `pruned_by_eq_ref_heuristic` 把它们全部剪掉。

**★ 触发它的三个条件**（`:2869-2872`，★ 与剪枝 2 共用同一个 `prune_level` 开关）：

```cpp
        if (prune_level == 1 &&             // 1) 剪枝启用
            position->key != nullptr &&     // 2) 走了 ref/eq_ref 访问
            position->rows_fetched <= 1.0)  // 3) 1::1 —— 唯一键，取到 <= 1 行
```

**★ 链本身还有自己的准入**（`eq_ref_extension_by_limited_search` 内的 5 条，★ 与外层 4 条不同）：

```cpp
    if (s->keyuse() &&                             // 1) 可能用 keyref
        (remaining_tables & real_table_bit) &&     // 2) 还在剩余集合
        !(remaining_tables & s->dependent) &&      // 3) 不依赖未进前缀的表
        (~remaining_tables & s->key_dependent) &&  // 4) ★ key 依赖的表必须已在前缀（保证链贴合前缀）
        (!idx || !check_interleaving_with_nj(s)))  // 5) nest 交织检查
```

> ★ 第 4 条是"链"成立的关键：`~remaining_tables & s->key_dependent` 为真，说明它所依赖的表**都已在前缀里**，所以才能无代价地贴上去。

**★ 链吃光所有表时——全函数唯一的 `goto`**（`:2891`）：

```cpp
            if (eq_ref_extended == remaining_tables) goto done;
```

---

### ★★ 三种剪枝的真实控制流（总结）

| 顺序 | 判定 | 行为 |
|---|---|---|
| 0 | 候选资格 4 条（`:2770-2773`） | 不合格 → 落到 `backout_nj_state` |
| 1 | `prune_by_cost`（`:2817`） | **`continue`** |
| 2 | `pruned_by_heuristic`（`:2832`） | **`continue`** |
| 3a | 首次 eq_ref → 调 `eq_ref_extension_by_limited_search` | 吃光 → **`goto done`**；否则 **`continue`** |
| 3b | 已被链收录 → `pruned_by_eq_ref_heuristic`（`:2900`） | **`continue`** |
| — | 都通过 | 递归下潜 或 落叶子 `consider_plan` |

> ★★★ **三种剪枝全部只 `continue`（跳过当前候选表），没有任何一个"终止整个分支"**。唯一终止整个循环的是 2891 的 `goto done`，以及循环条件里的 `!use_best_so_far`（由二级引擎在 `consider_plan` 里置位）。
>
> ★ 这是理解剪枝强度的关键：**剪枝只砍"当前这一层"的某个候选，不砍整棵子树**。

**★★ 剪枝 1 的完整判定**（★ 精确剪枝，但带一个合法性闸门）：

```cpp
      if (position->prefix_cost >= join->best_read &&
          found_plan_with_allowed_sj) {
        trace_one_table.add("pruned_by_cost", true);
        backout_nj_state(remaining_tables, s);
        continue;
      }
```

> ★ `found_plan_with_allowed_sj` 是闸门：**若至今没找到"只用允许策略"的计划，即使代价更差也必须继续搜**——因为继续下去可能换成合法策略。

**★★ 剪枝 2 的完整判定**（★ 含 "may miss the optimal QEPs" 原文在 `:2829-2830`）：

```cpp
      /*
        Prune some less promising partial plans. This heuristic may miss
        the optimal QEPs, thus it results in a non-exhaustive search.
      */
      if (prune_level == 1) {
        if (best_rowcount > position->prefix_rowcount ||         /* A 行数更少 */
            best_cost > position->prefix_cost ||                 /* B 代价更小 */
            (idx == join->const_tables &&                        /* C 特例：第一张表且是 ORDER BY 的表 */
             s->table() == join->sort_by_table)) {
          if (best_rowcount >= position->prefix_rowcount &&      /* 全面更优才刷新层内基线 */
              best_cost >= position->prefix_cost &&
              /* TODO: What is the reasoning behind this condition? */
              (!(s->key_dependent & remaining_tables) ||
               position->rows_fetched < 2.0)) {
            best_rowcount = position->prefix_rowcount;
            best_cost = position->prefix_cost;
          }
        } else if (found_plan_with_allowed_sj) {                 /* ★ 三个都不成立 → 剪 */
          trace_one_table.add("pruned_by_heuristic", true);
          backout_nj_state(remaining_tables, s);
          continue;
        }
      }
```

> ★ 分支 C 的意义：**第一张表若是 `sort_by_table`，无条件放行**——因为它可能带来"免排序"的巨大收益，不能按行数/代价砍掉。
> ⚠️ 内层那个 `TODO: What is the reasoning behind this condition?` 是源码里**未解的自问**。

**★★ 递归终止条件**（⚠️ **不是** `idx == join->tables`）：

```cpp
      const table_map remaining_tables_after = (remaining_tables & ~real_table_bit);
      if ((current_search_depth > 1) && remaining_tables_after) {
        ... 递归下潜 ...
      } else {
        if (consider_plan(idx, &trace_one_table)) return true;   /* 落叶子 */
        assert((remaining_tables_after != 0) ||
               ((cur_embedding_map == 0) &&                      /* 完整计划必须无未闭合 nest */
                (join->positions[idx].dups_producing_tables == 0) &&
                (join->deps_of_remaining_lateral_derived_tables == 0)));
      }
```

> ★★ 语义是"**还有深度** 且 **还有表**"才下潜；二者任一不满足就落叶子。这正是"限深"的含义——**深度耗尽时即使还剩表，也直接把当前部分计划当作候选回写**。

**★ `consider_plan()` 的三元式**（★ 真身在 `sql_planner.cc:2480`，文档原稿写的 2555 只是函数体内的 `if (chosen) {`）：

```cpp
  bool cheaper = cost < join->best_read;
  bool chosen = found_plan_with_allowed_sj ? (plan_uses_allowed_sj && cheaper)
                                           : (plan_uses_allowed_sj || cheaper);
```

> ★★★ **这是"单调收敛"开关**：一旦已有合法计划，任何**只用被禁用策略**的更便宜计划都不能上位；但在此之前，"合法"或"更便宜"二者满足其一即可——**保证搜索最终一定收敛到一个合法计划**。

**★ 回写时的 `-0.001`**（★ 一个容易被忽略的稳定性技巧）：

```cpp
    join->best_read = cost - 0.001;   /* 让后续严格小于比较稳定，避免平票抖动 */
```

**★ `restore_prev_nj_state()` 不存在**——真名是 `backout_nj_state()`（`sql_planner.h:164` / `sql_planner.cc:4654`），是 `check_interleaving_with_nj()` 的逆操作（逐级 `nj_counter--`，只在"从完全覆盖变为部分覆盖"时才继续向上）。另有 RAII 的 `Table_map_restorer`（`sql_optimizer.h:1160`）负责 `deps_of_remaining_lateral_derived_tables`。

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

### 6.4 `POSITION` 结构体（`sql/sql_select.h:325-587`）

```cpp
/**
  A position of table within a join order. This structure is primarily used
  as a part of @c join->positions and @c join->best_positions arrays.

  One POSITION element contains information about:
   - Which table is accessed
   - Which access method was chosen
      = Its cost and \#of output records
   - Semi-join strategy choice. Note that there are two different
     representation formats:
      1. The one used during join optimization
      2. The one used at plan refinement/code generation stage.
      We call fix_semijoin_strategies_for_picked_join_order() to switch
      between #1 and #2. See that function's comment for more details.
   ...
  This class has to stay a POD, because it is memcpy'd in many places.
*/

struct POSITION {
  /**
    The number of rows that will be fetched by the chosen access
    method per each row combination of previous tables. That is:

      rows_fetched = selectivity(access_condition) * cardinality(table)
    ...
    @note For index/table scans, rows_fetched may be less than
    the number of rows in the table because the cost of evaluating
    constant conditions is included in the scan cost ...
  */
  double rows_fetched;

  /**
    Cost of accessing the table in course of the entire complete join
    execution, i.e. cost of one access method use (e.g. 'range' or
    'ref' scan ) multiplied by estimated number of rows from tables
    earlier in the join sequence.

    read_cost does NOT include cost of processing rows within the
    executor (row_evaluate_cost).
  */
  double read_cost;

  /**
    The fraction of the 'rows_fetched' rows that will pass the table
    conditions that were NOT used by the access method. If, e.g.,

      "SELECT ... WHERE t1.colx = 4 and t1.coly @> 5"

    is resolved by ref access on t1.colx, filter_effect will be the
    fraction of rows that will pass the "t1.coly @> 5" predicate. The
    valid range is 0..1 ...
  */
  float filter_effect;

  /**
    prefix_rowcount and prefix_cost form a stack of partial join
    order costs and output sizes

    prefix_rowcount: The number of row combinations that will be
    joined to the next table in the join sequence.

    For a joined table it is calculated as
      prefix_rowcount =
          last_table.prefix_rowcount * rows_fetched * filter_effect

    @see filter_effect

    For a semijoined table it may be less than this formula due to
    duplicate elimination.
  */
  double prefix_rowcount;
  double prefix_cost;

  JOIN_TAB *table;

  /**
    NULL  -  'index' or 'range' or 'index_merge' or 'ALL' access is used.
    Other - [eq_]ref[_or_null] access is used. Pointer to {t.keypart1 = expr}
  */
  Key_use *key;

  /** If ref-based access is used: bitmap of tables this table depends on  */
  table_map ref_depend_map;
  bool use_join_buffer;
```

/* 【注】★ "This class has to stay a POD, because it is memcpy'd in many places"——
   这就是为什么 `best_positions` 与 `positions` 之间靠 `memcpy` 快照、回溯不用深拷贝。
   semijoin 策略成员（`sj_strategy` / `dups_producing_tables` / `first_*match_table` / ...）
   在 `sql/sql_select.h:449-534`，同样逐字段带注释。 */

### 6.5 前缀代价的精确公式 `set_prefix_join_cost()`

`sql/sql_select.h:544-575`：

```cpp
  /**
    Set complete estimated cost and produced rowcount for the prefix of tables
    up to and including this table, in the join plan.

    @param cost     Estimated cost
    @param rowcount Estimated row count
  */
  void set_prefix_cost(double cost, double rowcount) {
    prefix_cost = cost;
    prefix_rowcount = rowcount;
  }
  /**
    Set complete estimated cost and produced rowcount for the prefix of tables
    up to and including this table, calculated from the cost of the previous
    stage, the fanout of the current stage and the cost to process a row at
    the current stage.

    @param idx      Index of position object within array, if zero there is no
                    "previous" stage that can be added.
    @param cm       Cost model that provides the actual calculation
  */
  void set_prefix_join_cost(uint idx, const Cost_model_server *cm) {
    if (idx == 0) {
      prefix_rowcount = rows_fetched;
      prefix_cost = read_cost + cm->row_evaluate_cost(prefix_rowcount);
    } else {
      prefix_rowcount = (this - 1)->prefix_rowcount * rows_fetched;
      prefix_cost = (this - 1)->prefix_cost + read_cost +
                    cm->row_evaluate_cost(prefix_rowcount);
    }
    prefix_rowcount *= filter_effect;
  }
```

★ **公式要写准**（源码里 `filter_effect` 是**最后统一乘**的，对 `idx == 0` 也生效）：

```
prefix_rowcount = prev.prefix_rowcount * rows_fetched * filter_effect      (idx > 0)
prefix_cost     = prev.prefix_cost + read_cost
                  + row_evaluate_cost(prev.prefix_rowcount * rows_fetched)  ← 用的是「未乘 filter」的中间值
```

/* 【注】注意 `row_evaluate_cost()` 吃的是**未乘 filter** 的中间 rowcount——
   即"取出来多少行就评估多少行"，过滤只减少**传给下一张表**的行数，不减少本层的评估代价。
   这与 `calculate_condition_filter` 的语义一致（filter 是 post-read filtering）。 */

调用点：`sql/sql_planner.cc:2149`（`optimize_straight_join`）、`:2791`（`best_extension_by_limited_search` 里比 `best_rowcount > position->prefix_rowcount`）、`:3149`（`eq_ref_extension_by_limited_search`）；新 AccessPath 侧有一处镜像实现（`sql/sql_executor.cc:1906-1919`，注释自称 "even though the cost calculation doesn't make a lot of sense"）。

`best_ref` / `best_positions` / `positions` 的声明（`sql/sql_optimizer.h:158-164`、`:307-317`）：

```cpp
  /**
    Array of plan operators representing the current (partial) best
    plan. The array is allocated in JOIN::make_join_plan() and is valid only
    inside this function. Initially (*best_ref[i]) == join_tab[i].
    The optimizer reorders best_ref.
  */
  JOIN_TAB **best_ref{nullptr};
```

```cpp
  /**
    This is the result of join optimization.

    @note This is a scratch array, not used after get_best_combination().
  */
  POSITION *best_positions{nullptr};

  /******* Join optimization state members start *******/

  /* Current join optimization state */
  POSITION *positions{nullptr};
```

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

### 7.3 `calculate_condition_filter()` 本体：filter 是怎么算出来的

声明与函数头注释（`sql/sql_planner.h:200-254`，核心部分）：

```cpp
/**
  Calculate 'Post read filtering' effect of JOIN::conds for table
  'tab'. Only conditions that are not directly involved in the chosen
  access method shall be included in the calculation of this 'Post
  read filtering' effect.

  The function first identifies fields that are directly used by the
  access method. This includes columns used by range and ref access types,
  and predicates on the identified columns (if any) will not be taken into
  account when the filtering effect is calculated.
  ...
    1) Row estimates from the range optimizer
    2) Row estimates from index statistics (records per key)
    3) Guesstimates
  ...
  @return  the 'post read filtering' effect (between 0 and 1) of
           JOIN::conds
*/
float calculate_condition_filter(const JOIN_TAB *const tab,
                                 const Key_use *const keyuse,
                                 table_map used_tables, double fanout,
                                 bool is_join_buffering, bool write_to_trace,
                                 Opt_trace_object &parent_trace);
```

/* 【注】★ 关键设计：**被访问方法"用掉"的列不算过滤**。ref/range 已经把那些谓词算进
   `rows_fetched` 了，若再算一次就重复打折。所以函数第一步是把这些列标进 `TABLE::tmp_set`。 */

第一步：把访问方法已用的列排除（`sql/sql_planner.cc:1334-1403` 节选）：

```cpp
  /*
    If ref/range access, the condition is already included in the
    record estimate. The fields used by the ref/range access method
    shall not contribute to the filtering estimate since 'filter' is
    percentage of fetched rows that are filtered away.
  */
  if (keyuse) {
    const KEY *key = table->key_info + keyuse->key;
    ...
      while (curr_ku->table_ref == tab->table_ref &&         // 1)
             curr_ku->key == keyuse->key &&                  // 1)
             curr_ku->keypart_map & keyuse->bound_keyparts)  // 2)
      {
        bitmap_set_bit(&table->tmp_set,
                       key->key_part[curr_ku->keypart].field->field_index());
        curr_ku++;
      }
  } else if (tab->range_scan())
    get_fields_used(tab->range_scan(), &table->tmp_set);

  /*
    Early exit if the only conditions for the table refers to columns
    used by the access method.
  */
  if (bitmap_is_subset(&table->cond_set, &table->tmp_set)) {
    assert(filter == COND_FILTER_ALLPASS);
    goto cleanup;
  }
```

第二步：优先用 range 优化器的行估计（`sql/sql_planner.cc:1404-1445`）：

```cpp
  /*
    If the range optimizer has made row estimates for predicates that
    are not used by the chosen access method, the estimate from the
    range optimizer is used as filtering effect for those fields. We
    do this because the range optimizer is more accurate than index
    statistics.
  */
  if (!table->quick_keys.is_clear_all()) {
    char buf[MAX_FIELDS / 8];
    ...
        const float selectivity = static_cast<float>(table->quick_rows[keyno]) /
                                  static_cast<float>(tab->records());
        // Cannot possible access more rows than there are in the table
        filter *= std::min(selectivity, 1.0f);
      }
    }
  }
```

第三步：剩下的谓词交给 `Item::get_filtering_effect()`，并做两道下限保护（`sql/sql_planner.cc:1447-1502`）：

```cpp
  if (tab->join()->where_cond &&                             // 1)
      !bitmap_is_subset(&table->cond_set, &table->tmp_set))  // 2)
  {
    /*
      Get filtering effect for predicates that are not already
      reflected in 'filter'. The below call gets this filtering effect
      based on index statistics and guesstimates.
    */
    filter *= tab->join()->where_cond->get_filtering_effect(
        tab->join()->thd, tab->table_ref->map(), used_tables, &table->tmp_set,
        static_cast<double>(tab->records()));
  }

  /*
    Cost calculations and picking the right join order assumes that a
    positive number of output rows from each joined table. We assume
    that at least one row in the table match the condition.  Not all
    code is able to cope with estimates of less than one row.  (For
    example, DupsWeedout may include extra tables in its
    duplicate-eliminating range in such cases.)
  */
  filter = max(filter, 1.0f / tab->records());

  /*
    For large tables, the restriction above may still give very small
    numbers when calculating fan-out.  The code below makes sure that
    there is a lower limit on fan-out.
    TODO: Should evaluate whether this restriction makes sense.  It
          can cause the estimated size of the result set to be
          different for different join orders. However, some unwanted
          effects on DBT-3 was observed when removing it, so keeping
          it for now.
  */
  if ((filter * fanout) < 0.05F) filter = 0.05F / static_cast<float>(fanout);

cleanup:
  filtering_effect_trace.end();
  parent_trace.add("final_filtering_effect", filter);

  // Clear tmp_set so it can be used elsewhere
  bitmap_clear_all(&table->tmp_set);
  assert(filter >= 0.0F && filter <= 1.0F);
  return filter;
}
```

/* 【注】两道保护都值得记：① `filter >= 1/records` —— 保证"至少有一行匹配"，
   否则后续 fanout 可能变成 0，DupsWeedout 等策略会算错；
   ② `filter * fanout >= 0.05` —— 保证 fanout 不低于 0.05。
   ★ 注意注释自承："it can cause the estimated size of the result set to be
   **different for different join orders**"（不同 join order 的结果集估算会不一致）。 */

### 7.4 没有统计时的 guesstimate 常数

`sql/item.h:93-115`：

```cpp
/**
  Default condition filtering (selectivity) values used by
  get_filtering_effect() and friends when better estimates
  (statistics) are not available for a predicate.
*/
/**
  For predicates that are always satisfied. Must be 1.0 or the filter
  calculation logic will break down.
*/
#define COND_FILTER_ALLPASS 1.0f
/// Filtering effect for equalities: col1 = col2
#define COND_FILTER_EQUALITY 0.1f
/// Filtering effect for inequalities: col1 > col2
#define COND_FILTER_INEQUALITY 0.3333f
/// Filtering effect for between: col1 BETWEEN a AND b
#define COND_FILTER_BETWEEN 0.1111f
/**
   Value is out-of-date, will need recalculation.
   This is used by post-greedy-search logic which changes the access method and
   thus makes obsolete the filtering value calculated by best_access_path(). For
   example, test_if_skip_sort_order().
*/
#define COND_FILTER_STALE -1.0f
```

/* 【注】即：**等值 0.1、不等值 0.3333、BETWEEN 0.1111、无法估计 1.0**。
   `COND_FILTER_STALE = -1.0` 是"已过期需重算"的标记——greedy_search 之后
   `test_if_skip_sort_order()` 改了访问方法，之前算的 filter 就作废了。 */

有直方图时优先用直方图 selectivity（`Item_func_ne::get_filtering_effect`，`sql/item_cmpfunc.cc:2465-2481`）：

```cpp
  double selectivity;
  if (!get_histogram_selectivity(thd, fld->field, args, arg_count,
                                 histograms::enum_operator::NOT_EQUALS_TO, this,
                                 fld->field->table->s, &selectivity))
    return static_cast<float>(selectivity);

  return 1.0f - fld->get_cond_filter_default_probability(rows_in_table,
                                                         COND_FILTER_EQUALITY);
```

/* 【注】返回类型是"成功时返回 false"的反直觉约定：`!get_histogram_selectivity(...)`
   为真 ⇒ 拿到 selectivity ⇒ 直接返回它。 */

### 7.5 fanout 与 filter 在 `best_access_path()` 里的合流

产生（`find_best_ref()` 末尾，`sql/sql_planner.cc:687-689`）：

```cpp
    start_key->bound_keyparts = found_part;
    start_key->fanout = cur_fanout;
    start_key->read_cost = cur_read_cost;
```

读取与"ref 是否值得"的判据（`sql/sql_planner.cc:1022-1033`、`:1087-1088`）：

```cpp
  // Look for the best ref access if the storage engine supports index access.
  if (tab->keyuse() != nullptr &&
      (table->file->ha_table_flags() & HA_NO_INDEX_ACCESS) == 0)
    best_ref =
        find_best_ref(tab, remaining_tables, idx, prefix_rowcount,
                      &found_condition, &ref_depend_map, &used_key_parts);

  double rows_fetched = best_ref ? best_ref->fanout : DBL_MAX;
  /*
    Cost of executing the best access method prefix_rowcount
    number of times
  */
  double best_read_cost = best_ref ? best_ref->read_cost : DBL_MAX;
```

```cpp
  if (rows_fetched < tab->found_records &&  // (1a)
      best_read_cost <= tab->read_time)     // (1b)
```

写回 `POSITION`（`sql/sql_planner.cc:1193-1224`）：

```cpp
  /*
    Storage engines that track exact sizes may report an empty table
    as having row count equal to 0.
    If this table is an inner table of an outer join, adjust row count to 1,
    so that the join planner can make a better fanout calculation for
    the remaining tables of the join. (With size 0, the fanout would always
    become 0, meaning that the cost of adding one more table would also
    become 0, regardless of access method).
  */
  if (rows_fetched == 0.0 &&
      (join->query_block->outer_join & tab->table_ref->map()))
    rows_fetched = 1.0;

  /*
    Do not calculate condition filtering unless 'ref' access is
    chosen. The filtering effect for all the scan types of access
    (range/index scan/table scan) has already been calculated.
  */
  if (best_ref)
    filter_effect = calculate_condition_filter(
        tab, best_ref, ~remaining_tables & ~excluded_tables, rows_fetched,
        false, false, trace_access_scan);

  best_read_cost += derived_mat_cost;
  pos->filter_effect = filter_effect;
  pos->rows_fetched = rows_fetched;
  pos->read_cost = best_read_cost;
  pos->key = best_ref;
  pos->table = tab;
  pos->ref_depend_map = ref_depend_map;
  pos->loosescan_key = MAX_KEY;
  pos->use_join_buffer = best_uses_jbuf;
```

/* 【注】① 外连接内表的 `rows_fetched` 保底为 1 —— 否则 fanout=0 会让"再加一张表"的
   代价恒为 0，后续表随便选；② scan 类的 filter 已在 `calculate_scan_cost()` 里算过
   （`sql/sql_planner.cc:787-798`），这里**只给 ref 算**，避免重复。 */

---

## 八、半连接与物化表的嵌入

`best_extension_by_limited_search` 里除了访问方法，还要处理：

- **semi-join 策略**：每加一张表后调 `advance_sj_state`（`sql_planner.cc:4105`）判定/更新五种策略（详见 **03 篇**）
- **物化表**（sjm）：`get_best_combination` 时把 inner tables 放到 primary tables 之后，把 sjm 表放进 primary tables

搜索结束后 `fix_semijoin_strategies` 从后往前确定最终策略（03 篇 4.4 节）。

**★★ 物化 semi-join nest 的递归优化：复用同一个类，只是第 3 参非 NULL**

⚠️ **不是**"独立的 JOIN / 独立优化器"。`Optimize_table_order` 构造函数的第 3 参 `sjm_nest_arg` 非 NULL 时，成员 `emb_sjm_nest` 非 NULL → 进入"nest 模式"。**唯一**以非 NULL 构造的调用点在 `sql_optimizer.cc:6632`：

```cpp
/* optimize_semijoin_nests_for_materialization() 内 */
      if (Optimize_table_order(join->thd, join, sj_nest).choose_table_order())
        return true;                        /* ★ 递归进 join order 搜索 */
      ...
      memcpy(sj_nest->nested_join->sjm.positions,
             join->best_positions + join->const_tables,
             sizeof(POSITION) * n_tables);  /* ★ nest 的局部最优 join order 被快照 */
```

nest 模式下 `choose_table_order()` 走不同分支：

```cpp
  if (emb_sjm_nest) {
    merge_sort(join->best_ref + join->const_tables, join->best_ref + join->tables,
               Join_tab_compare_embedded_first(emb_sjm_nest));  /* nest 内表顶到最前 */
    join_tables = emb_sjm_nest->sj_inner_tables;                /* ★ 只枚举 nest 内表 */
  }
  ...
  // Remaining part of this function not needed when processing semi-join nests.
  if (emb_sjm_nest) return false;      /* ★ 不做 fix_semijoin_strategies */
  if (fix_semijoin_strategies()) return true;
```

> ★ `Join_tab_compare_embedded_first`：属于目标 nest 的表**无条件排前面**，其余情况完全退化为 `Join_tab_compare_default`。
> ⚠️ **两个易混字段**：`Optimize_table_order::emb_sjm_nest`（本类成员，构造期固定）vs `JOIN_TAB::emb_sj_nest`（由 `JOIN::set_semijoin_embedding()` 逐表设置）。`Join_tab_compare_embedded_first` 拿**后者**与**前者**比较。
> ★ nest 模式下 `has_sj = false` ⇒ **不调 `advance_sj_state()`**（`advance_sj_state` 里还有 `assert(emb_sjm_nest == nullptr)`）。

**★ 额外槽位机制**（⚠️ 原文"每个 nest 至多 1 个"）：

```cpp
/* JOIN::init_planner_arrays()，sql_optimizer.cc:5437 */
  // Up to one extra slot per semi-join nest is needed (if materialized)
  const uint sj_nests = query_block->sj_nests.size();
  const uint table_count = query_block->leaf_table_count;

  best_ref       = alloc(table_count + sj_nests + 2 + m_windows.elements);
  map2table      = alloc(table_count + sj_nests);
  positions      = new POSITION[table_count];              /* ★ 当前搜索用：无额外槽 */
  best_positions = new POSITION[table_count + sj_nests];   /* ★★ 最终计划：多 sj_nests 槽 */
```

> ★ 额外槽位是给 `best_positions` 的——物化 nest 的临时表要占一个 `POSITION`。`positions`（搜索工作数组）**没有**额外槽。
> ★ `sj_nests` 必须在 `pull_out_semijoin_tables()` **之后**才取——源码注释 `// Changed by pull-out`（nest 可能被整个抽空并从 `sj_nests` 删除）。

**★ `pull_out_semijoin_tables()`：哪些 nest 表能"上提"成普通表**

判据是**函数依赖**——该表能否用 `eq_ref(外部/剩余 nest 表)` 唯一访问（`find_eq_ref_candidate`）。不动点循环（上提一张可能解锁另一张）。但**代价**是：

```cpp
            sj_nest->nested_join->sj_corr_tables |= tbl->map();   /* ★ nest 变 correlated */
            sj_nest->nested_join->sj_depends_on |= tbl->map();
```

> ★★ Doxygen 的 NOTE 点出这个反直觉后果：**上提会让原本不相关的子查询变相关，从而禁用 Materialization 与 LooseScan**。所以 pull-out 不是纯收益——它换来更简单的 join order，代价是丢掉两种 semi-join 策略。
> ★ antijoin nest（`is_aj_nest()`）**不参与** pull-out。
> ★ 受 `query_block->sj_pullout_done` 保护，**只在首次优化时跑**。

**★ `sj_enabled_strategies` 位图的构造**（⚠️ 真名 `Query_block::update_semijoin_strategies()`，在 `sql/sql_lex.cc:4556`，**不是** `Optimize_table_order` 的方法）：

```cpp
void Query_block::update_semijoin_strategies(THD *thd) {
  uint sj_strategy_mask =
      OPTIMIZER_SWITCH_FIRSTMATCH | OPTIMIZER_SWITCH_LOOSE_SCAN |
      OPTIMIZER_SWITCH_MATERIALIZATION | OPTIMIZER_SWITCH_DUPSWEEDOUT;  /* ★ 只有这 4 个 */
  uint opt_switches = thd->variables.optimizer_switch & sj_strategy_mask;
  for (Table_ref *sj_nest : sj_nests) {
    Table_ref *table = sj_nest->nested_join->m_tables.back();   /* ★ 从 nest 最后一张表取 hint */
    sj_nest->nested_join->sj_enabled_strategies =
        (table->opt_hints_qb && !is_secondary_engine_optimization)
            ? table->opt_hints_qb->sj_enabled_strategies(opt_switches)
            : opt_switches;
    if (sj_nest->is_aj_nest())
      sj_nest->nested_join->sj_enabled_strategies &=
          OPTIMIZER_SWITCH_FIRSTMATCH | OPTIMIZER_SWITCH_MATERIALIZATION |
          OPTIMIZER_SWITCH_DUPSWEEDOUT;                         /* ★ antijoin 砍掉 LOOSE_SCAN */
  }
}
```

```cpp
/* Opt_hints_qb::sj_enabled_strategies()，sql/opt_hints.cc:256 */
uint Opt_hints_qb::sj_enabled_strategies(uint opt_switches) const {
  if (semijoin_hint) {
    const uint strategies = semijoin_hint->get_args();
    if (semijoin_hint->switch_on())  // SEMIJOIN hint
      return (strategies == 0) ? opt_switches : strategies;  /* ★ 有参 → 以 hint 为准 */
    return ~strategies & opt_switches;                       /* ★ NO_SEMIJOIN → 减掉 */
  }
  return opt_switches;
}
```

**★★ `get_best_combination()`：join order 搜索的"落地"出口**

它把 `best_positions` 翻译成 QEP_TAB 顺序，并处理物化 nest 的"表重排"：

```cpp
  uint outer_target = 0;
  uint inner_target = primary_tables + num_tmp_tables;   /* ★ nest 内表 → 尾部 */
  ...
  if (has_semijoin) {
    for (uint tableno = 0; tableno < primary_tables;) {
      if (sj_is_materialize_strategy(best_positions[tableno].sj_strategy)) {
        sjm_nests++;
        inner_target -= (best_positions[tableno].n_sj_tables - 1);
        tableno += best_positions[tableno].n_sj_tables;   /* ★ 跳过整个 nest */
      } else tableno++;
    }
  }
  ...
    const uint target =
        (remaining_sjm_inner--) > 0 ? inner_target++ : outer_target++;   /* ★ 分派 */
```

> ★★ **物化 nest 的 N 张内表被压缩成"1 个临时表槽（在 primary 区）+ N 个尾部内表槽"**。这就是 `init_planner_arrays()` 预留 `sj_nests` 槽位的消费点（`best_positions + sjm_index`）。
> ⚠️ 它**必须**在 `fix_semijoin_strategies()` 之后调用——Doxygen 原文：*"It must be called after Optimize_table_order::fix_semijoin_strategies()"*。

**★ `fix_semijoin_strategies()` 全仓库只有 1 个调用点**：`sql_planner.cc:2039`。
⚠️ **`fix_semijoin_strategies_for_picked_join_order()` 不存在**——`sql/sql_select.h:337/463` 只是残留注释提及的旧名。

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

**★★★ 分流不只是"换了 join order 算法"——hypergraph 有三条根本性差异**

**① 不做 const 表，也不在优化期求值子查询**（`sql_optimizer.cc:422`）：

```cpp
  if (thd->lex->using_hypergraph_optimizer()) {
    for (Query_expression *unit = ...; unit; unit = unit->next_query_expression()) {
      if (!unit->is_optimized() && unit->optimize(...)) return true;  /* 先优化所有子查询 */
    }
    // The hypergraph optimizer does not do const tables,
    // nor does it evaluate subqueries during optimization.
    query_block->add_active_options(OPTION_NO_CONST_TABLES |
                                    OPTION_NO_SUBQUERY_DURING_OPTIMIZATION);
  }
```

**② 完全不走 `make_join_plan()` / `get_best_combination()`**——源码自陈的分界线：

```cpp
  // ----------------------------------------------------------------------------
  //       All of this is never called for the hypergraph join optimizer!
  // ----------------------------------------------------------------------------
  assert(!thd->lex->using_hypergraph_optimizer());
```

**③ 同一个 query block 会被规划两次**（`sql_optimizer.cc:620-656`）：

```cpp
    Item *where_cond_no_in2exists = remove_in2exists_conds(where_cond);
    ...
    m_root_access_path = FindBestQueryPlan(thd, query_block, trace_ptr);
    ...
    if (where_cond != where_cond_no_in2exists ||
        having_cond != having_cond_no_in2exists) {
      where_cond = where_cond_no_in2exists;
      having_cond = having_cond_no_in2exists;
      m_root_access_path_no_in2exists =
          FindBestQueryPlan(thd, query_block, trace_ptr);   /* ★ 第二套计划 */
    }
```

> ★ 原因注释说得很清楚：外层 query block 可能根据代价决定"撤掉 IN→EXISTS 改走物化"，那需要一套**不带 in2exists 附加条件**的计划。两套计划并存，执行期二选一。

**★ `hypergraph_optimizer` 不是独立系统变量**，是 `optimizer_switch` 的第 24 位 flag，且**不在默认集合里**：

```cpp
/* sql/sql_const.h:223 */
constexpr const uint64_t OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER{1ULL << 24};
```

```cpp
/* sql/sys_vars.cc:3472 —— ★ 源码原注释 */
    "hypergraph_optimizer",  // Deliberately not documented below.
```

> ★★ **"故意不写进文档"是源码注释的原文**。且帮助文本里确实没有它——`Sys_optimizer_switch` 的 description 列举了所有开关，**唯独漏了 `hypergraph_optimizer`**。

**★ 打开时会告警 / 可能被拒绝**（`sql/sys_vars.cc:3406`）：

```cpp
#ifdef WITH_HYPERGRAPH_OPTIMIZER
    // Allow, with a warning.
    push_warning(thd, Sql_condition::SL_WARNING, ER_WARN_DEPRECATED_SYNTAX,
                 ER_THD(thd, ER_WARN_HYPERGRAPH_EXPERIMENTAL));
    return false;
#else
    // Disallow; the hypergraph optimizer is not ready for production yet.
    my_error(ER_HYPERGRAPH_NOT_SUPPORTED_YET, MYF(0), "use in non-debug builds");
    return true;
#endif
```

> ★ CMake 默认：debug 构建 `WITH_HYPERGRAPH_OPTIMIZER=ON`，否则 OFF（`CMakeLists.txt:2124`）。且 `optimizer_switch=DEFAULT` 时**不会**把它关掉（防止 MTR `--hypergraph` 中途被取消）。

**★ hypergraph 的"深度"旋钮是另一个变量**（⚠️ **不吃** `optimizer_search_depth`）：

```cpp
/* sql/sys_vars.cc:3326 */
static Sys_var_ulong Sys_optimizer_max_subgraph_pairs(
    "optimizer_max_subgraph_pairs", ..., 
    "Ignored by the old (non-hypergraph) join optimizer",   /* ★ 源码自陈 */
    VALID_RANGE(1, INT_MAX), DEFAULT(100000), BLOCK_SIZE(1));
```

> ★ 默认 100000，是"子图对数量"上限而非"深度"。经典优化器完全忽略它。

**★★ 算法根本差别的源码自陈**

| | 经典（`sql_planner.cc`） | hypergraph（`join_optimizer/`） |
|---|---|---|
| 算法 | **前缀式贪心 + 限深 DFS** | **DPhyp：枚举所有连通子图划分**（`EnumerateAllConnectedPartitions`） |
| 形状 | 只 left-deep | bushy（任意形状） |
| 候选保留 | **只保留一个** `best_positions` | **保留 Pareto 前沿的多个候选**（`root_candidates`），最后才按总代价选一 |
| 论文 | — | Moerkotte & Neumann《Dynamic Programming Strikes Back》(SIGMOD 2008) |

```cpp
/* sql/join_optimizer/estimate_selectivity.cc:282 —— ★ 源码自陈"链"的差别 */
  // There is a challenge in that the Item::get_filtering_effect() API
  // is intrinsically locked to the old join optimizer's way of thinking,
  // where one made a long chain of (left-deep) nested tables, ...
  // In the hypergraph optimizer, we no longer have a chain, and always
  // estimate selectivity for applicable conditions only;
```

```cpp
/* sql/join_optimizer/join_optimizer.h:27 —— 自陈的目标与当前限制 */
  It is intended to eventually take over completely from the older join
  optimizer based on prefix search (sql_planner.cc and related code),
  ... The most notable ones are that we do not support:
    - Hints (except STRAIGHT_JOIN).
    - TRADITIONAL and JSON formats for EXPLAIN (use FORMAT=tree).
    - UPDATE.
```

**greedy 只支持 left-deep 的代码体现**：`best_extension` 的递归只能"往前缀末尾追加一张表"（`:2908` 的 `idx+1`），`greedy_search` 每次只把 `best_table` 插到 `best_ref[idx]`。整个搜索树任意时刻的状态都是"有序前缀 + 无序剩余"，新增表只能挂前缀末尾——**没有"把两个已构建好的 partial plan 再 join 起来"的代码路径**，故只能产生 left-deep 计划。hypergraph 用 DPhyp 枚举任意连通子图对（见 09 篇），支持 bushy。

---

## 十、相关参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `optimizer_search_depth` | **62**（`MAX_TABLES+1`，0=自动） | 手动指定 DFS 深度；0 时按表数走 7 表分界。`MAX_TABLES=61`（`sql_const.h:107`） |
| `optimizer_prune_level` | **1** | 1=启用启发式剪枝（可能漏最优）；0=关闭，退化为穷尽 |

读入：`Optimize_table_order` 构造函数 `sql_planner.cc:125-141`。`optimizer_search_depth` 注册在 `sys_vars.cc:3315`（`VALID_RANGE(0, MAX_TABLES+1)`，`DEFAULT(MAX_TABLES+1)`）。

注册原文（`sql/sys_vars.cc:3305-3324`）：

```cpp
/// @todo change to enum
static Sys_var_ulong Sys_optimizer_prune_level(
    "optimizer_prune_level",
    "Controls the heuristic(s) applied during query optimization to prune "
    "less-promising partial plans from the optimizer search space. "
    "Meaning: 0 - do not apply any heuristic, thus perform exhaustive "
    "search; 1 - prune plans based on number of retrieved rows",
    HINT_UPDATEABLE SESSION_VAR(optimizer_prune_level), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, 1), DEFAULT(1), BLOCK_SIZE(1));

static Sys_var_ulong Sys_optimizer_search_depth(
    "optimizer_search_depth",
    "Maximum depth of search performed by the query optimizer. Values "
    "larger than the number of relations in a query result in better "
    "query plans, but take longer to compile a query. Values smaller "
    "than the number of tables in a relation result in faster "
    "optimization, but may produce very bad query plans. If set to 0, "
    "the system will automatically pick a reasonable value",
    HINT_UPDATEABLE SESSION_VAR(optimizer_search_depth), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, MAX_TABLES + 1), DEFAULT(MAX_TABLES + 1), BLOCK_SIZE(1));
```

/* 【注】`MAX_TABLES` 是**推导值**不是硬编码：`sql/sql_const.h:105,108`
   `MAX_TABLES_FOR_SIZE = sizeof(table_map)*8`，`MAX_TABLES = MAX_TABLES_FOR_SIZE - 3`
   ⇒ 61；减掉的 3 位是 `INNER_TABLE_BIT` / `OUTER_REF_TABLE_BIT` / `RAND_TABLE_BIT`
   三个"伪表"位。故 `search_depth` 取值 0..62，**默认 62**。 */

对应的 C 变量（`sql/system_variables.h:255-260`）与读入点（`sql/sql_planner.cc:125-131`）：

```cpp
  ulong net_wait_timeout;
  ulong net_write_timeout;
  ulong optimizer_prune_level;
  ulong optimizer_search_depth;
  ulong optimizer_max_subgraph_pairs;
```

```cpp
Optimize_table_order::Optimize_table_order(THD *thd_arg, JOIN *join_arg,
                                           Table_ref *sjm_nest_arg)
    : thd(thd_arg),
      join(join_arg),
      search_depth(determine_search_depth(thd->variables.optimizer_search_depth,
                                          join->tables - join->const_tables)),
      prune_level(thd->variables.optimizer_prune_level),
```

成员声明（`sql/sql_planner.h:92-96`）——两个都是 `const`，构造时固化、搜索全程只读：

```cpp
  const uint search_depth;  // Maximum search depth to apply in greedy search
  const uint prune_level;   // pruning heuristics to be applied
                            // (0 = EXHAUSTIVE, 1 = PRUNE_BY_TIME_OR_ROWS)
```

`determine_search_depth()`（`sql/sql_planner.cc:2044-2094`，注释略）：

```cpp
uint Optimize_table_order::determine_search_depth(uint search_depth,
                                                  uint table_count) {
  if (search_depth > 0) return search_depth;
  /* TODO: this value should be determined dynamically, based on statistics: */
  const uint max_tables_for_exhaustive_opt = 7;

  if (table_count <= max_tables_for_exhaustive_opt)
    search_depth =
        table_count + 1;  // use exhaustive for small number of tables
  else
    /*
      TODO: this value could be determined by some mapping of the form:
      depth : table_count -> [max_tables_for_exhaustive_opt..MAX_EXHAUSTIVE]
    */
    search_depth = max_tables_for_exhaustive_opt;  // use greedy search

  return search_depth;
}
```

/* 【注】★ 因为默认 `search_depth = 62 > 0`，**7 表启发式默认根本不会被触发**——
   默认等价于完整穷举。只有显式 `SET optimizer_search_depth = 0` 才走 `表数 <= 7 ⇒ depth = 表数+1`
   （等价穷举）/ `表数 > 7 ⇒ depth = 7`（真贪心）这条派生规则。 */

`condition_fanout_filter` 是 `optimizer_switch` 的一个 flag 而非独立 sysvar（`sql/sql_const.h:216`）：

```cpp
constexpr const uint64_t OPTIMIZER_SWITCH_COND_FANOUT_FILTER{1ULL << 17};
```

默认掩码里含该位（`sql/sys_vars.cc:208`）：

```cpp
    OPTIMIZER_SWITCH_COND_FANOUT_FILTER | OPTIMIZER_SWITCH_DERIVED_MERGE |
```

---


---

## 十一、ORDER BY / GROUP BY 反向约束 join order（`sort_by_table`）

前面十章都是"代价决定顺序"。这一章是唯一一处**顺序被代价之外的因素反向约束**的地方：ORDER BY / GROUP BY 若只落在一张表上，优化器会想办法把那张表放到 join 的**首位**，从而让索引直接给出有序输出、省掉 filesort。

### 11.1 先判定"排序是否只落在一张表上"

`JOIN::optimize()` 里（`sql/sql_optimizer.cc:596-598`）：

```cpp
  sort_by_table = get_sort_by_table(order.order, group_list.order,
                                    query_block->leaf_tables);
```

```cpp
/**
  Return table number if there is only one table in sort order
  and group and order is compatible, else return 0.
*/

static TABLE *get_sort_by_table(ORDER *a, ORDER *b, Table_ref *tables) {
  DBUG_TRACE;
  table_map map = (table_map)0;

  if (!a)
    a = b;  // Only one need to be given
  else if (!b)
    b = a;

  for (; a && b; a = a->next, b = b->next) {
    if (!(*a->item)->eq(*b->item, true)) return nullptr;
    map |= a->item[0]->used_tables();
  }
  map &= ~INNER_TABLE_BIT;
  if (!map || (map & (RAND_TABLE_BIT | OUTER_REF_TABLE_BIT))) return nullptr;

  for (; !(map & tables->map()); tables = tables->next_leaf)
    ;
  if (map != tables->map()) return nullptr;  // More than one table
  DBUG_PRINT("exit", ("sort by table: %d", tables->tableno()));
  return tables->table;
}
```

/* 【注】判定是**苛刻**的：ORDER BY 与 GROUP BY 必须逐项 `eq()` 相等，且涉及的
   `used_tables()` 必须**恰好等于某一张 leaf table 的 map**——多一张表就返回 nullptr，
   这个机制直接失效（后面所有分支都不再生效）。 */

落点成员（`sql/sql_optimizer.h:172`）：

```cpp
  TABLE *sort_by_table{nullptr};
```

### 11.2 硬前提：索引消解 ORDER BY 只对"第一个非 const 表"成立

`test_if_skip_sort_order()` 里的断言（`sql/sql_optimizer.cc:2211-2212`）：

```cpp
  /* Check that we are always called with first non-const table */
  assert((uint)tab->idx() == join->const_tables);
```

/* 【注】这条 assert 是整个机制的**物理约束**：join 输出的有序性只能由"驱动表"的索引提供，
   一旦驱动表不是它，后面加进来的表每次嵌套循环都会把顺序打乱 ⇒ 只能 filesort。 */

配套的 `simple_order` 开关由 `remove_const()` 判定（契约注释 `sql_optimizer.cc:10160-10173`）：

```cpp
  simple_order is set to true if sort_order only uses fields from head table
  and the head table is not a LEFT JOIN table.
```

判据行（`sql_optimizer.cc:10206`）：

```cpp
  *simple_order = !first_tab->join_cond();
```

`simple_order == false` 时（`sql_optimizer.cc:1700-1703`）根本不尝试 `test_if_skip_sort_order()`，直接走临时表。

### 11.3 约束怎么进 greedy_search：成本惩罚 + 剪枝豁免（不是重排）

**(a) `consider_plan()` 给"首表不是 sort_by_table"的 plan 加一笔排序成本**（`sql/sql_planner.cc:2480-2502`）：

```cpp
bool Optimize_table_order::consider_plan(uint idx,
                                         Opt_trace_object *trace_obj) {
  double cost = join->positions[idx].prefix_cost;
  double sort_cost = 0;
  double windowing_cost = 0;
  /*
    We may have to make a temp table, note that this is only a
    heuristic since we cannot know for sure at this point.
    Hence it may be too pessimistic.
    ...
  */
  if (join->sort_by_table &&
      join->sort_by_table !=
          join->positions[join->const_tables].table->table()) {
    sort_cost = join->positions[idx].prefix_rowcount;
    cost += sort_cost;
    trace_obj->add("sort_cost", sort_cost).add("new_cost_for_plan", cost);
  }
```

/* 【注】★ 惩罚量就是 `prefix_rowcount`——"要排序的行数"。这让"不把 sort_by_table 放首位"
   的 join order 变贵，但**不禁止**它：代价足够低时仍可胜出。这是**软约束**。 */

**(b) `best_extension_by_limited_search()` 对"首表 == sort_by_table"豁免剪枝**（`sql/sql_planner.cc:2828-2853`）：

```cpp
      /*
        Prune some less promising partial plans. This heuristic may miss
        the optimal QEPs, thus it results in a non-exhaustive search.
      */
      if (prune_level == 1) {
        if (best_rowcount > position->prefix_rowcount ||
            best_cost > position->prefix_cost ||
            (idx == join->const_tables &&  // 's' is the first table in the QEP
             s->table() == join->sort_by_table)) {
          if (best_rowcount >= position->prefix_rowcount &&
              best_cost >= position->prefix_cost &&
              /* TODO: What is the reasoning behind this condition? */
              (!(s->key_dependent & remaining_tables) ||
               position->rows_fetched < 2.0)) {
            best_rowcount = position->prefix_rowcount;
            best_cost = position->prefix_cost;
          }
        } else if (found_plan_with_allowed_sj) {
          DBUG_EXECUTE("opt",
                       print_plan(join, idx + 1, position->prefix_rowcount,
                                  position->read_cost, position->prefix_cost,
                                  "pruned_by_heuristic"););
          trace_one_table.add("pruned_by_heuristic", true);
          backout_nj_state(remaining_tables, s);
          continue;
        }
      }
```

/* 【注】`idx == join->const_tables && s->table() == join->sort_by_table` 这个条件把
   "候选表就是 sort_by_table 且正被放在首位"的情况从 `pruned_by_heuristic` 里捞出来——
   否则它可能因为代价略高被启发式剪掉，而它其实是唯一能省掉 filesort 的 plan。 */

**(c) 若首表是 sort_by_table 却选不出索引访问，则彻底放弃**（`sql/sql_planner.cc:1226-1230`）：

```cpp
  if (!best_ref && idx == join->const_tables && table == join->sort_by_table &&
      join->query_expression()->select_limit_cnt >= rows_fetched) {
    trace_access_scan.add("use_tmp_table", true);
    join->sort_by_table = (TABLE *)1;  // Must use temporary table
  }
```

/* 【注】`(TABLE *)1` 是个**哨兵值**而非真指针——置上之后，上面 (a) 的
   `join->sort_by_table != positions[const_tables].table->table()` 恒为真
   （1 永远不等于任何 TABLE*），于是**所有** plan 都被加 sort_cost。 */

**(d) `optimize_straight_join()` 结尾同样的加成**（`sql/sql_planner.cc:2164-2166`）：

```cpp
  if (join->sort_by_table &&
      join->sort_by_table != join->positions[join->const_tables].table->table())
    cost += rowcount;  // We have to make a temp table
```

### 11.4 两种 ORDER BY 的特例

`test_if_ft_index_order()`（`sql/sql_optimizer.cc:1721-1739`）——全文索引的 `ORDER BY MATCH(...) DESC`：

```cpp
/**
  Test if ORDER BY is a single MATCH function(ORDER BY MATCH)
  and sort order is descending.

  @param order                 pointer to ORDER struct.

  @retval
    Pointer to MATCH function if order is 'ORDER BY MATCH() DESC'
  @retval
    NULL otherwise
*/

static Item_func_match *test_if_ft_index_order(ORDER *order) {
  if (order && order->next == nullptr && order->direction == ORDER_DESC &&
      is_function_of_type(*order->item, Item_func::FT_FUNC))
    return down_cast<Item_func_match *>(*order->item)->get_master();

  return nullptr;
}
```

`SELECT DISTINCT` 在**单表**时被改写成 GROUP BY，从而也能走索引（`sql/sql_optimizer.cc:1526-1598` 的 `create_order_from_distinct()` 调用段）——注意前置条件 `plan_is_single_table()`，多表时不成立。

### 11.5 `set_prefix_tables()` 在这里**不**参与决定顺序

`JOIN::set_prefix_tables()`（`sql/sql_optimizer.cc:5192-5264`）虽然名字像"设置前缀表"，但它在 `get_best_combination()` 里被调用（`sql/sql_optimizer.cc:3271-3277`），即 **join order 已定之后**：

```cpp
  if (!plan_is_const()) {
    // Assign map of "available" tables to all tables belonging to query block
    set_prefix_tables();
    adjust_access_methods();
  }
```

它按 `best_ref[]` 的既定顺序**累加** `table_map`，为每个 JOIN_TAB 记录"前缀里有哪些表可用"，供条件下推（`make_cond_for_table`）、ref 构造（`create_ref_for_key`）与执行期动态 range 使用；`JOIN_TAB::prefix_tables()` 的实现在 `sql/sql_opt_exec_shared.h:310-311`，成员叫 `prefix_tables_map`（不是 `m_prefix_tables`）。

★ **一句话总结**：`sort_by_table` 是 ORDER BY / GROUP BY 进入 join order 的**唯一入口**，作用方式是"给不放首位的 plan 加 `prefix_rowcount` 的排序成本 + 给放首位的情况豁免启发式剪枝"；**没有任何代码会重排或强制约束搜索顺序**。而 `sort_by_first_tables` / `first_tables` 这类"强制前缀表集合"在 8.0.39 **不存在**（全仓 0 命中）——那是 hypergraph 优化器（`sql/join_optimizer/interesting_orders.*`）的概念。

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

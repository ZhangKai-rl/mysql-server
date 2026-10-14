# 08 Range 优化：SEL_TREE / SEL_ARG 区间森林与 range 访问方法

> Range 优化是**访问方法选择**里最复杂的一块（属于物理优化③），独立成篇。本篇覆盖：WHERE 如何变成区间森林、森林如何合并、如何变成多种 range 访问方法、代价怎么算、执行期怎么用。
>
> ★ 相邻主题：`GROUP_INDEX_SKIP_SCAN` 在**优化期如何被识别与被否决**（LIS 与分组实现的关系）见 [15 篇第五章](../15_groupby_distinct_order.md)；"这个条件为什么建不出 range"（类型聚合提升到 REAL 域）见 [21 篇](../21_collation_index_usability.md)第三章。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、range 优化在优化器中的位置](#一range-优化在优化器中的位置)
- [二、get_mm_tree：从 WHERE 构造 SEL_TREE](#二get_mm_tree从-where-构造-sel_tree)
- [三、SEL_ARG 的三重结构与区间操作](#三sel_arg-的三重结构与区间操作)
- [四、各 range 访问方法的判定与实现](#四各-range-访问方法的判定与实现)
- [五、代价与行数估算（index dive vs 统计）](#五代价与行数估算index-dive-vs-统计)
- [六、长 IN list 字面量的处理](#六长-in-list-字面量的处理)
- [七、从 SEL_TREE 到执行](#七从-sel_tree-到执行)

---

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 为什么是 SEL_ARG「区间森林」——三重结构各管什么

`tree.h` 里有长达 200 行的设计文档，核心一句是：

> It represents a condition in a special form (we don't have a name for it ATM). **The SEL_ARG::next/prev is "OR", and next_key_part is "AND".**

| 结构 | 字段 | 解决什么 |
|---|---|---|
| **红黑树** | `left/right/parent/color` | 按区间下界 **O(log n) 定位** + 有序插入 |
| **双向链表** | `next/prev` | 同 key part 内的**有序遍历与线性归并** |
| **next_key_part** | 指向 `SEL_ROOT` | 多列前缀的**递归 AND** |

**为什么三者不可替代**：

- `key_or` 用红黑树 `find_range()` 定位插入点
- 但**区间合并**（判断重叠/相邻）只能在 `next/prev` 链上做——因为合并要看**前驱/后继区间的上界**，这是链表语义、不是搜索树语义
- 多列 AND 递归：`next_key_part` 必须指向 `SEL_ROOT`（一棵完整子树），而不是单个 `SEL_ARG`

**为什么不直接用扁平区间列表或位图**（源码 `SPACE COMPLEXITY NOTES` 直接回答了）：

- **扁平列表** = 展开后的 DNF，多列 IN 会**笛卡尔爆炸**：`(kp1 IN n1) AND (kp2 IN n2) AND (kp3 IN n3)` ⇒ `n1*n2*n3` 条区间
- **位图**需要离散可枚举域，而 range 面对的是 `VARCHAR/DECIMAL/DATETIME` 等**有序连续域**上的开闭区间（`min_flag/max_flag` 携带 `NEAR_MIN/NEAR_MAX/NO_MIN_RANGE/NO_MAX_RANGE`），根本建不了位图
- **SEL_ARG 图**通过 `next_key_part` **共享子树**，让"后继 keypart 的同一段条件"只存一份

但源码也诚实承认**共享不是万能的**：存在 WHERE 长度 `O(3*#max_keyparts)`、而 SEL_ARG 图大小 `O(2^(#max_keyparts/2))` 的查询——**这就是后面 8MB 内存闸的直接动因**。

**一个常被忽略的关键设计：引用计数 + lazy copy**。`SEL_ROOT::use_count` 做 copy-on-write；`key_or` 里若两棵树都被共享（`use_count > 0`）就 clone，**且选元素更少的一方**（最便宜）。

#### 2. range 分析为什么在多个阶段反复做

先修正：**`create_access_paths` 不重跑**（它直接复用 `tab->range_scan()`）。真正重跑 `test_quick_select()` 的有 5 处：`estimate_rowcount`、`test_if_skip_sort_order`（2 处）、`can_switch_from_ref_to_range`、`make_join_query_block`（2 处），以及执行期的 `DynamicRangeIterator::Init`（**每行外层表重算一次**）。

**为什么必须分多次**——四类"之前不知道的信息"：

- **(a) 前驱表的常量值（MAYBE_KEY）**：`SEL_ROOT::Type::MAYBE_KEY` 的注释——"There is a range predicate that refers to another table. The range access method cannot be used on this index **unless that other table is earlier in the join sequence**"。所以 `test_quick_select` 才有 `prev_tables` / `read_tables` 两个参数
- **(b) 排序方向**：注释明说前次调用没考虑 ASC/DESC，"so in case of DESC ordering we still need to recheck"
- **(c) LIMIT**：`test_quick_select` 开头就有 `if (limit < records)`，所以 `make_join_query_block` 的 `LOW_LIMIT` 重跑条件才有意义
- **(d) 表依赖 / 条件上移**（`NOT_FIRST_TABLE`）

**代价与收益**：每次重跑都要重建整棵森林 + **重新做 index dive**。源码承认这个开销（`check_stack_overrun` 按 `3 * STACK_MIN_SIZE + sizeof(RANGE_OPT_PARAM)`），而"**只对 OOM 报一次警告**"的设计正是因为会重跑多次。

**本质**：join order、常量传播、排序需求、LIMIT 四者**互相依赖**。MySQL 选的是"**贪心 + 事后补救**"而不是联合求解——证据是 `make_join_query_block` 里那段自嘲注释：*"Access method changed. This is after deciding join order… so the info updated below will **not have any effect** on the execution plan."*

#### 3. 为什么需要这么多种 range 访问类型

`test_quick_select` 里的**竞争顺序本身就是设计说明**（每种都在跟"覆盖索引全扫"基线比价）：

```
1. 覆盖索引全扫（基线，只挑最短覆盖索引）
2. get_best_group_min_max  → GROUP_INDEX_SKIP_SCAN
3. get_best_skip_scan      → INDEX_SKIP_SCAN
4. get_key_scans_params    → INDEX_RANGE_SCAN
5. get_best_ror_intersect  → ROR-intersection
6. get_best_disjunct_quick → index merge
```

**skip scan 的动机：复合索引前导列缺失**。形式化条件是 `SELECT A, B, C FROM T WHERE EQ(A) AND RNG(C)`，索引 `I = <A, B, C>`，其中 **B 非空且无条件**（就是缺失那一列）。执行是"for each eq_prefix → for each distinct B → 子区间扫 C"。

**为什么不能直接用 index merge**（三个硬理由）：

1. index merge 要求每个分支是**完整可用的 range**，而 B 列**根本没有条件**，无法构成任何 range
2. **index merge 不支持覆盖索引**（`index_merge.h` 明确列为限制），而 skip scan **强制要求覆盖索引**
3. index merge 无法**枚举 B 的 distinct 值**

**为什么不全扫**：skip scan 用 `index_next_different()` **跳组**（"skip"的由来），代价是 `O(distinct(A,B))` 次 B 树定位，而非 `O(rows)` 次记录读取。

> 额外设计点：skip scan 的候选索引走**单独的 `skip_scan_keys`**，不走 `const_keys`——因为前导列缺失，该索引不可能用于 ref 访问。

**MIN/MAX 优化**（注意它**不在** `range_optimizer/` 目录，在 `sql/opt_sum.cc`）：这是最激进的一种——**整个 JOIN 都可能被消掉**，直接返回常量行。它与 range 是不同层次：range 优化"怎么读行"，MIN/MAX 优化"**根本不用读行**"（`HA_READ_KEY_OR_NEXT` 一次搞定）。且它要**在优化期就执行**（把聚合函数替换成常量）。

> ★ **完整机制见 [`../15_groupby_distinct_order.md`](../15_groupby_distinct_order.md) 2.5**：三道硬否决、`find_key_for_maxmin()` 的四条约束（含**降序 keypart 不支持**）、`is_simple_predicate()` 认哪三种 WHERE 形态、`matching_cond()` 遇 OR 直接放弃、以及**为什么取到端点行后还要用 `reckey_in_range()` 回判一次**。本篇只保留"它与 range 是不同层次"这个对照关系。

**loose scan / GROUP_MIN_MAX**：把 GROUP BY + 聚合**整个下沉到访问层**（连 `AggregateIterator` 都省了）。它与 skip scan 的本质区别：loose scan 要求 **gap 列必须是常量**（"the predicates provide constants to fill the gap in the index"），而 skip scan 是**枚举** gap 列的 distinct 值。

**index merge 为什么分 ROR-intersection / ROR-union / sort-union**：

- **ROR**（rowid ordered）要求分支输出 rowid 有序，才能做**多路归并求交集**，否则必须落 `Unique`/临时表。能否 ROR 由 `is_key_scan_ror()` 判定：等值前缀之后的剩余 keypart 必须与聚簇 PK 前缀一致
- **CPK 的特殊处理**（很精妙）：聚簇 PK 含所有字段，用它做 intersection 的常规扫描没意义——所以**不用它取行，而是用它的条件去过滤其他 key 扫描得到的 rowid**
- **union vs sort-union**：ROR-union 每个分支还要 ROR；不能用 ROR 就退化到 `IndexMergeIterator` + `Unique` 去重

#### 4. index dive 为什么对 range 特别重要

**答案一句话**：`records_per_key` 只对**完整前缀等值**有意义。范围谓词（`a > 5 AND a < 100`、`LIKE 'x%'`、`BETWEEN`）**没有任何统计能回答**，只能真的下到 B 树里量一下两个边界之间有多少记录。

源码也确认"**只有等值区间才配用统计**"（`eq_ranges_exceeds_limit` 的注释：必须是等值、且非 `x IS NULL`）。

**InnoDB 侧怎么估**（`btr_estimate_n_rows_in_range`）：两次 B 树下降找分叉层 → 分叉在叶层则**精确**（`nth_rec` 相减）；分叉很高则**采样**，且**最多读 10 页**（`N_PAGES_READ_LIMIT`）。几个有意思的细节：

- 并发下重试 4 次，失败返回固定值 10（`rows_in_range_arbitrary_ret_val`）
- **怕优化器误判空集**：`if (n_rows == 0) n_rows = 1`——注释说优化器会基于"0 行"直接返回 Empty set，即使加锁读也该真的搜索
- **上界钳制**：`n_rows > table_n_rows / 2 && !exact` ⇒ 砍到一半，避免离谱估计

**无 dive 的三类兜底**：(a) FORCE INDEX 的 `skip_records_in_range`；(b) 引擎能力不足（NULL_RANGE 导致全扫被拒、`HA_ONLY_WHOLE_INDEX`）；(c) `records_in_range` 返回 `HA_POS_ERROR`（整个 MRR 方案作废）。

#### 5. 长 IN list 的设计

- **DNF 展开**：每次 `tree_or` 都要 `find_range` + RB insert + 可能的 clone，总体 **O(N log N)**
- **NOT IN 的代价是源码明确承认的 O(N²)**：注释说 *"the range analyzer will use O(N²) memory (**which is probably a bug**)"*，并引用 BUG#15872 / BUG#21282 ⇒ **NOT IN 超过 1000 条直接放弃构造 SEL_TREE**（`NOT_IN_IGNORE_THRESHOLD`）。设计权衡很清楚：与其 O(N²) 冒 OOM 风险，不如放弃（源码甚至认为"大 NOT IN 列表本来也不太可能产出好的 range 访问"）
- **排序 + 二分（`in_vector`）**有两层目的：①**执行期**无 range 时 O(log N) 求值；②**优化期**构造区间。**NULL 不入数组**（"避免二分误匹配"）
- **为什么不显式去重**：①重复值产生同一个单点区间，`key_or` 的合并逻辑会自然并掉；②排序已使重复值相邻，"跳过相邻相等"就等价去重，不必为每种类型（`in_string/in_longlong/in_double/...`）各实现一次；③去重会破坏 `value_to_item(pos)` 的下标语义
- **8MB 内存闸（`range_optimizer_max_mem_size`）**：`SEL_ARG/SEL_TREE` 全分配在 `param->temp_mem_root`，超限抛 `EE_CAPACITY_EXCEEDED` → 被 `Range_optimizer_error_handler` **降级成 warning** → 之后所有 `param->has_errors()` 检查短路返回 `nullptr` → **退化为全表扫**。且"只报一次警告"（因为多阶段重跑）。
  > 注意：**group-min-max 仍可能被选中**——注释说 "it can be constructed **no matter if there is a range tree**"

#### 6. 8.0 的 range_optimizer 重构

`sql/opt_range.cc` **已不存在**，拆成 **16 个** `range_optimizer/*.cc`（15 个是"每个访问类型一套"，外加 `partition_pruning.cc` 复用同一套基础设施）。每个访问类型都有 **`_plan`（规划，返回 `AccessPath`）+ 非 `_plan`（执行期 Iterator）**双文件。

**最可核实的动机**：与 AccessPath / Iterator 化强耦合——旧架构 range 优化器返回 `QUICK_SELECT_I*`（**既是执行器又是计划**），新架构劈成两半。

**最重要的架构收益**：粒度化 API 让 **hypergraph 能绕过 `test_quick_select`**，直接调用 `get_mm_tree` + `check_quick_select` + `get_ranges_from_tree` 三个原语；而且它**逐谓词调用 `get_mm_tree` 再自己 `tree_and`**，从而精确追踪"哪个谓词被哪个 range 覆盖/吸收"（`tree_applied_predicates` vs `tree_subsumed_predicates`）——**这在拆分前不可能做到**，因为旧代码里 `get_mm_tree` 只在 `test_quick_select` 内部被整体调用一次。

> 遗留痕迹说明重构是渐进的：`ROR_SCAN_INFO` 的 TODO、"this is my (sgunders') understanding as of September 2021: …although it seems the code for this was never written"。

### 失效场景与已知短板

#### 多列范围的限制（源码最坦白的自我批评）

**range 只能对最后一个 keypart 用不等式**。源码注释直接编码了这条规则：

```
/* "(kp1 > c1) AND (kp2 OP c2) AND ..." -> (kp1 > c1) */
```

工程表达是 `num_exact_key_parts`（被压低），并有"last-ditch effort"：`a >= 3 AND b IN (4,9,10)` ⇒ 从 `(3,4)` 开始扫（而不是 `(3,-inf)`），**但之后必须 recheck**。

#### NULL 处理

- `x IS NULL` 在 range 里是**单点区间**（`is_singlepoint` 明确算作单点）
- 但 **`x IS NULL` 不能用索引统计**——三处独立注释说法一致："the number of rows with this value are likely to be very different"
- 外连接 inner 表的 `IS NULL` **直接禁用 range**
- `NULL_RANGE` 可能导致全表扫而被拒绝

#### collation / 类型转换

- 不可比（不同 collation、DATETIME vs TIME）时 `warn_index_not_applicable`
- **LIKE 一律标记 `inexact`**：注释说 Unicode collation 有 contractions 等，"will frequently be a bit too broad"
- **方言坑**：`"int_col > 'foo'"` 被解释为 `"int_col > 0"` 而非永假——注释原话："MySQL's SQL dialect has some **strange interpretations**"
- **前缀索引导致去重失败**（很隐蔽）：`VARCHAR(3)` 列 + `INDEX(col(1))`，"f" 与 "foo" 在索引里不可区分 ⇒ NOT IN 的区间必须从 `<=` 而非 `<` 开始

#### index merge 的四大限制（`index_merge.h` 原文）

1. **能用 range 就永不用 index merge**（即使 range 更贵）
2. 不支持覆盖索引（上一条的推论）
3. 复杂嵌套 AND/OR 会漏掉一些方案，"the choice of read plan may depend on **the order of conjuncts/disjuncts**"
4. 没有 `index_merge_ref`

其他：**DESC 排序禁用** ROR-intersection 与 index merge union；**DELETE 禁用** ROR-intersection（"Simultaneous key scans and row deletes on several handler objects are not allowed"）；**XOR 直接放弃**；几何类型不能做 AND。

### ICP 与 range 的边界

边界由 `num_exact_key_parts` 精确定义：

| 条件类别 | range 处理 | ICP / Filter |
|---|---|---|
| 索引前缀上的等值（构成 range 边界） | ✅ 完全吸收（subsumed） | 不需要 |
| 最后一个 keypart 上的不等式 | ✅ 作为 range 边界 | 不需要 |
| **非等值 keypart 之后的 keypart 条件** | ❌（`num_exact_key_parts` 被压低） | ✅ **必须 recheck** |
| **前缀索引列上的条件** | 部分（边界是截断值） | ✅ |
| **LIKE / 类型转换 / 非可比 collation** | 产生**过宽**的 range（`inexact`） | ✅ |
| 不在该索引中的列 | ❌ | 视情况 |

> **关键结论**：`SEL_TREE::inexact` 在旧优化器下**不影响正确性**（"The old join optimizer **always** does this"——无条件 recheck）。它真正发挥作用是在**超图优化器**里：避免重复计算 selectivity、避免多余的 Filter。

### 理论溯源

- 区间表示与红黑树 + 有序链表的组合是 range optimization 的经典做法
- 引用计数 + copy-on-write 用于对抗"共享子树的组合爆炸"

### 他库对比与演进动机

- **重构收益之一是可测试性**：`unittest/gunit/fake_range_opt_param.h` 提供 `Fake_RANGE_OPT_PARAM`，`opt_range-t.cc` 直接 include `range_optimizer/internal.h`
- **重构收益之二**是让 hypergraph 能复用 range 原语（前面已述）
- 分区裁剪（`partition_pruning.cc`）也复用同一套 `RANGE_OPT_PARAM` + `SEL_ARG` 基础设施

---

## 一、range 优化在优化器中的位置

### 1.1 入口 test_quick_select

`sql/range_optimizer/range_optimizer.cc:524`。它在两个时机被调用：

| 时机 | 调用点 | 说明 |
|------|--------|------|
| **第一次** | `estimate_rowcount` → `get_quick_record_count`（`sql_optimizer.cc:6218`） | join order 还没定，只能传 `prev_tables = 0`（不能用引用其他表的谓词） |
| **第二次（重估）** | `make_join_query_block`（`sql_optimizer.cc:9882-9905`） | join order 已定，前缀表已知，range 能用上更多谓词（详见 07 篇 E 节） |

> range 优化器有个铁律（源码注释 `range_optimizer.cc:817-819`）：**range 优化器只在 range 优于全表扫描时才会产出 range scan**，所以调用方不需要再和全表扫比。

### 1.2 主流程（完整源码）

```cpp
// sql/range_optimizer/range_optimizer.cc:524
int test_quick_select(THD *thd, MEM_ROOT *return_mem_root,
                      MEM_ROOT *temp_mem_root, Key_map keys_to_use,
                      table_map prev_tables, table_map read_tables,
                      ha_rows limit, bool force_quick_range,
                      const enum_order interesting_order, TABLE *table,
                      bool skip_records_in_range, Item *cond,
                      Key_map *needed_reg, bool ignore_table_scan,
                      Query_block *query_block, AccessPath **path) {
  DBUG_TRACE;
  *path = nullptr;
  needed_reg->clear_all();

  if (keys_to_use.is_clear_all()) return 0;          // 退出 1：没有可用索引

  const Cost_model_server *const cost_model = thd->cost_model();
  ha_rows records = table->file->stats.records;
  if (!records) records++;
  double scan_time =
      cost_model->row_evaluate_cost(static_cast<double>(records)) + 1;
  Cost_estimate cost_est = table->file->table_scan_cost();
  cost_est.add_io(1.1);
  cost_est.add_cpu(scan_time);
  if (ignore_table_scan) {                           // FORCE INDEX 等：全表代价设成无穷
    scan_time = DBL_MAX;
    cost_est.set_max_cost();
  }
  if (limit < records) {
    cost_est.reset();
    // Force to use index
    cost_est.add_io(
        table->cost_model()->page_read_cost(static_cast<double>(records)) + 1);
    cost_est.add_cpu(scan_time);
  } else if (cost_est.total_cost() <= 2.0 && !force_quick_range)
    return 0;                                        // 退出 2：小表（全表代价 <=2.0）不值得 range

  Opt_trace_context *const trace = &thd->opt_trace;
  Opt_trace_object trace_range(trace, "range_analysis");
  Opt_trace_object(trace, "table_scan")
      .add("rows", table->file->stats.records)
      .add("cost", cost_est);

  keys_to_use.intersect(table->keys_in_use_for_query);
  if (keys_to_use.is_clear_all()) return 0;          // 退出 3

  /*
    Use the 3 multiplier as range optimizer allocates big RANGE_OPT_PARAM
    structure and may evaluate a subquery expression ...
  */
  uchar buff[STACK_BUFF_ALLOC];
  if (check_stack_overrun(thd, 3 * STACK_MIN_SIZE + sizeof(RANGE_OPT_PARAM),
                          buff)) {
    return 0;                                        // 退出 4：栈溢出保护
  }

  /* set up parameter that is passed to all functions */
  RANGE_OPT_PARAM param;
  if (setup_range_optimizer_param(thd, return_mem_root, temp_mem_root,
                                  keys_to_use, table, query_block, &param)) {
    return 0;                                        // 退出 5
  }
  thd->push_internal_handler(&param.error_handler);   // ★ range_optimizer_max_mem_size 的 OOM handler
  auto cleanup = create_scope_guard([thd] { thd->pop_internal_handler(); });

  const bool index_merge_allowed =
      thd->optimizer_switch_flag(OPTIMIZER_SWITCH_INDEX_MERGE);
  const bool index_merge_union_allowed =
      index_merge_allowed &&
      thd->optimizer_switch_flag(OPTIMIZER_SWITCH_INDEX_MERGE_UNION);
  const bool index_merge_sort_union_allowed =
      index_merge_allowed &&
      thd->optimizer_switch_flag(OPTIMIZER_SWITCH_INDEX_MERGE_SORT_UNION);
  const bool index_merge_intersect_allowed =
      index_merge_allowed &&
      thd->optimizer_switch_flag(OPTIMIZER_SWITCH_INDEX_MERGE_INTERSECT);

  /* Calculate cost of full index read for the shortest covering index */
  if (!table->covering_keys.is_clear_all()) {
    int key_for_use = find_shortest_key(table, &table->covering_keys);
    assert(key_for_use != MAX_KEY);

    Cost_estimate key_read_time = param.table->file->index_scan_cost(
        key_for_use, 1, static_cast<double>(records));
    key_read_time.add_cpu(
        cost_model->row_evaluate_cost(static_cast<double>(records)));

    bool chosen = false;
    if (key_read_time < cost_est) {
      cost_est = key_read_time;
      chosen = true;
    }
    Opt_trace_object trace_cov(trace, "best_covering_index_scan",
                               Opt_trace_context::RANGE_OPTIMIZER);
    trace_cov.add_utf8("index", table->key_info[key_for_use].name)
        .add("cost", key_read_time)
        .add("chosen", chosen);
    if (!chosen) trace_cov.add_alnum("cause", "cost");
  }

  AccessPath *best_path = nullptr;
  double best_cost = cost_est.total_cost();   // ★★ best_cost 初值 = 全表 / 最短覆盖索引扫描

  SEL_TREE *tree = nullptr;
  if (cond) {
    {
      Opt_trace_array trace_setup_cond(trace, "setup_range_conditions");
      tree = get_mm_tree(thd, &param, prev_tables | INNER_TABLE_BIT,
                         read_tables | INNER_TABLE_BIT,
                         table->pos_in_table_list->map(),
                         /*remove_jump_scans=*/true, cond);
    }
    if (tree) {
      if (tree->type == SEL_TREE::IMPOSSIBLE) {
        trace_range.add("impossible_range", true);
        cost_est.reset();
        cost_est.add_io(static_cast<double>(HA_POS_ERROR));
        return -1;                              // 退出 6：条件恒假
      }
      /*
        If the tree can't be used for range scans, proceed anyway, as we
        can construct a group-min-max quick select
      */
      if (tree->type != SEL_TREE::KEY) {
        trace_range.add("range_scan_possible", false);
        if (tree->type == SEL_TREE::ALWAYS)
          trace_range.add_alnum("cause", "condition_always_true");
        tree = nullptr;                         // ★ tree 置空，但下面仍可试 group min/max
      }
    }
  }

  /*
    Try to construct a GroupIndexSkipScanIterator.
    Notice that it can be constructed no matter if there is a range tree.
  */
  AccessPath *group_path = get_best_group_min_max(
      thd, &param, tree, interesting_order, skip_records_in_range, best_cost);
  if (group_path) {
    DBUG_EXECUTE_IF("force_lis_for_group_by", group_path->cost = 0.0;);
    param.table->quick_condition_rows =
        min<double>(group_path->num_output_rows(), table->file->stats.records);
    Opt_trace_object grp_summary(trace, "best_group_range_summary",
                                 Opt_trace_context::RANGE_OPTIMIZER);
    if (unlikely(trace->is_started()))
      trace_basic_info(thd, group_path, &param, &grp_summary);
    if (group_path->cost < best_cost) {
      grp_summary.add("chosen", true);
      best_path = group_path;
      best_cost = best_path->cost;
    } else
      grp_summary.add("chosen", false).add_alnum("cause", "cost");
  }

  bool force_skip_scan = hint_table_state(thd, param.table->pos_in_table_list,
                                          SKIP_SCAN_HINT_ENUM, 0);

  if (thd->optimizer_switch_flag(OPTIMIZER_SKIP_SCAN) || force_skip_scan) {
    AccessPath *skip_scan_path =
        get_best_skip_scan(thd, &param, tree, interesting_order,
                           skip_records_in_range, force_skip_scan);
    if (skip_scan_path) {
      param.table->quick_condition_rows = min<double>(
          skip_scan_path->num_output_rows(), table->file->stats.records);
      Opt_trace_object summary(trace, "best_skip_scan_summary",
                               Opt_trace_context::RANGE_OPTIMIZER);
      if (unlikely(trace->is_started()))
        trace_basic_info(thd, skip_scan_path, &param, &summary);

      if (skip_scan_path->cost < best_cost || force_skip_scan) {
        summary.add("chosen", true);
        best_path = skip_scan_path;
        best_cost = best_path->cost;
      } else
        summary.add("chosen", false).add_alnum("cause", "cost");
    }
  }

  if (tree && (best_path == nullptr || !get_forced_by_hint(best_path))) {
    dbug_print_tree("final_tree", tree, &param);

    MY_BITMAP needed_fields;
    if (fill_used_fields_bitmap(&param, &needed_fields)) {
      return 0;
    }

    {
      /* Calculate cost of single index range scan and possible intersections */
      Opt_trace_object trace_range_alt(trace, "analyzing_range_alternatives",
                                       Opt_trace_context::RANGE_OPTIMIZER);
      AccessPath *range_path = get_key_scans_params(
          thd, &param, tree, false, true, interesting_order,
          skip_records_in_range, best_cost, /*ror_only=*/false, needed_reg);

      if (range_path) {
        best_path = range_path;
        best_cost = best_path->cost;
      }

      /*
        Simultaneous key scans and row deletes on several handler
        objects are not allowed so don't use ROR-intersection for
        table deletes. Also, ROR-intersection cannot return rows in
        descending order
      */
      if ((thd->lex->sql_command != SQLCOM_DELETE) &&
          (index_merge_allowed ||
           hint_table_state(thd, param.table->pos_in_table_list,
                            INDEX_MERGE_HINT_ENUM, 0)) &&
          interesting_order != ORDER_DESC) {
        AccessPath *rori_path = get_best_ror_intersect(
            thd, &param, table, index_merge_intersect_allowed, tree,
            &needed_fields, best_cost,
            /*force_index_merge_result=*/true, /*reuse_handler=*/true);
        if (rori_path) {
          best_path = rori_path;
          best_cost = best_path->cost;
        }
      }
    }

    // Here we calculate cost of union index merge
    if (!tree->merges.is_empty()) {
      // Cannot return rows in descending order.
      if ((index_merge_allowed ||
           hint_table_state(thd, param.table->pos_in_table_list,
                            INDEX_MERGE_HINT_ENUM, 0)) &&
          interesting_order != ORDER_DESC && param.table->file->stats.records) {
        AccessPath *best_conj_path = nullptr, *new_conj_path = nullptr;
        Opt_trace_array trace_idx_merge(trace, "analyzing_index_merge_union",
                                        Opt_trace_context::RANGE_OPTIMIZER);

        for (SEL_IMERGE &imerge : tree->merges) {
          new_conj_path = get_best_disjunct_quick(
              thd, &param, table, index_merge_union_allowed,
              index_merge_sort_union_allowed, index_merge_intersect_allowed,
              skip_records_in_range, &needed_fields, &imerge, best_cost,
              needed_reg);
          if (new_conj_path)
            param.table->quick_condition_rows =
                min<double>(param.table->quick_condition_rows,
                            new_conj_path->num_output_rows());
          if (!best_conj_path ||
              (new_conj_path && new_conj_path->cost < best_conj_path->cost)) {
            best_conj_path = new_conj_path;
          }
        }
        if (best_conj_path) best_path = best_conj_path;
      }
    }
  }

  /*
    If we got a read plan, return it, but only if the storage engine supports
    using indexes for access.
  */
  if (best_path && (table->file->ha_table_flags() & HA_NO_INDEX_ACCESS) == 0) {
    records = best_path->num_output_rows();
    *path = best_path;
  }

  if (unlikely(trace->is_started() && best_path)) {
    Opt_trace_object trace_range_summary(trace, "chosen_range_access_summary");
    {
      Opt_trace_object trace_range_plan(trace, "range_access_plan");
      trace_basic_info(thd, best_path, &param, &trace_range_plan);
    }
    trace_range_summary.add("rows_for_plan", best_path->num_output_rows())
        .add("cost_for_plan", best_path->cost)
        .add("chosen", true);
  }

  if (records == 0) {
    return -1;                          // 退出 7：估算 0 行 ⇒ impossible
  } else {
    return *path != nullptr;            // 退出 8：1 = 找到 range，0 = 没找到
  }
}
```

⇒ 三个反直觉点：① **调用顺序是 group min/max → skip scan → 单索引 range → ROR-intersect → index merge union**，`best_cost` 单调收紧；② `get_best_group_min_max()` 在 `tree == nullptr` 时**也会调用**（注释明写 "no matter if there is a range tree"）；③ `interesting_order == ORDER_DESC` 会**直接禁掉** ROR-intersect 与 index merge union。

**结构示意**（上面完整源码的骨架，便于速读）：

```cpp
// range_optimizer.cc:524（结构示意）
AccessPath *test_quick_select(THD *thd, MEM_ROOT *return_mem_root, ...) {
  // 1. 算 table scan 代价作基准
  const Cost_estimate table_scan_cost = table->file->table_scan_cost();
  if (table_scan_cost.total_cost() <= 2.0) return nullptr;   // :560 小表短路

  // 2. 准备参数
  RANGE_OPT_PARAM param;
  if (setup_range_optimizer_param(thd, &param, &temp_mem_root, keys_to_use,
                                  table, cond)) return nullptr;      // :586

  // 3. 从 WHERE 构造区间森林
  SEL_TREE *tree = get_mm_tree(thd, &param, prev_tables, read_tables,
                               cond, ...);                           // :642

  // 4. 依次尝试各 range 访问方法，取代价最小
  if (tree) {
    if (tree->type == SEL_TREE::IMPOSSIBLE) { ... return nullptr; }

    get_best_group_min_max(&param, tree, &best_path, ...);    // :672  MIN/MAX 优化
    get_best_skip_scan(&param, tree, &best_path, ...);        // :693  skip scan
    get_key_scans_params(&param, tree, index_read_must_be_used,
                         order_direction, skip_records_in_range,
                         &best_path, ...);                    // :733  普通 range
    get_best_ror_intersect(&param, tree, &best_path, ...);    // :758  ROWID_INTERSECTION
    get_best_disjunct_quick(&param, tree, &best_path, ...);   // :782  ROWID_UNION / INDEX_MERGE
  }
  return best_path;
}
```

**关键**：各访问方法**不是互斥选择，而是各自尝试、比代价**（`best_path` 在函数间传递，代价更低就替换）。

### 1.3 RANGE_OPT_PARAM（`range_opt_param.h`）

range 优化的全局参数包：目标表、可用索引集合 `keys`、每个索引的 keypart 数、mem_root、真实索引标志等。它决定了 range 优化"能在哪些索引上工作"。

---

## 二、get_mm_tree：从 WHERE 构造 SEL_TREE

`sql/range_optimizer/range_analysis.cc:845`。**递归遍历 WHERE 的 Item 树，把每个谓词翻译成索引上的区间，再用 AND/OR 合并成森林。**

### 2.1 递归分发（完整源码）

```cpp
// sql/range_optimizer/range_analysis.cc:845
SEL_TREE *get_mm_tree(THD *thd, RANGE_OPT_PARAM *param, table_map prev_tables,
                      table_map read_tables, table_map current_table,
                      bool remove_jump_scans, Item *cond) {
  SEL_TREE *ftree = nullptr;
  bool inv = false;                 // ★ inv = 谓词被取反（NOT IN / NOT BETWEEN）
  DBUG_TRACE;

  if (param->has_errors()) return nullptr;

  if (cond->type() == Item::COND_ITEM) {          // ★ AND / OR 递归
    Item_func::Functype functype = down_cast<Item_cond *>(cond)->functype();

    SEL_TREE *tree = nullptr;
    bool first = true;
    for (Item &item : *down_cast<Item_cond *>(cond)->argument_list()) {
      SEL_TREE *new_tree = get_mm_tree(thd, param, prev_tables, read_tables,
                                       current_table, remove_jump_scans, &item);
      if (param->has_errors()) return nullptr;
      if (first) {
        tree = new_tree;
        first = false;
        continue;
      }
      if (functype == Item_func::COND_AND_FUNC) {
        tree = tree_and(param, tree, new_tree);          // AND → 区间求交
        if (tree && tree->type == SEL_TREE::IMPOSSIBLE) break;   // ★ AND 短路
      } else {  // OR
        tree = tree_or(param, remove_jump_scans, tree, new_tree);  // OR → 区间求并（可能产生 index merge 候选）
        if (tree == nullptr || tree->type == SEL_TREE::ALWAYS) break;  // ★ OR 短路
      }
    }
    return tree;
  }

  if (cond->const_item() && !cond->is_expensive()) {   // ★ 常量条件短路
    const SEL_TREE::Type type =
        cond->val_int() ? SEL_TREE::ALWAYS : SEL_TREE::IMPOSSIBLE;
    return new (param->temp_mem_root)
        SEL_TREE(type, param->temp_mem_root, param->keys);
  }

  // This used to be a guard against predicates like “WHERE x;”. But these are
  // now always rewritten to “x <> 0”, so it does not trigger there.
  // However, it is still relevant for subselects.
  if (cond->type() != Item::FUNC_ITEM) {
    return nullptr;   // ★ 子查询等非 FUNC_ITEM 直接放弃
  }

  Item_func *cond_func = (Item_func *)cond;
  if (cond_func->functype() == Item_func::BETWEEN ||
      cond_func->functype() == Item_func::IN_FUNC)
    inv = ((Item_func_opt_neg *)cond_func)->negated;
  else {
    Item_func::optimize_type opt_type = cond_func->select_optimize(thd);
    if (opt_type == Item_func::OPTIMIZE_NONE) return nullptr;
  }

  /*
    Notice that all fields that are outer references are const during
    the execution and should not be considered for range analysis ...
  */
  switch (cond_func->functype()) {
    case Item_func::BETWEEN: {          // ★ BETWEEN：拆成 GE + LE 后 tree_and
      Item *const arg_left = cond_func->arguments()[0];
      if (!arg_left->is_outer_reference() &&
          arg_left->real_item()->type() == Item::FIELD_ITEM) {
        Item_field *field_item = down_cast<Item_field *>(arg_left->real_item());
        ftree = get_full_func_mm_tree(thd, param, prev_tables, read_tables,
                                      current_table, remove_jump_scans,
                                      field_item, cond_func, nullptr, inv);
      }
      // ... 处理 "const BETWEEN f1 AND f2" 的反向形式（value 参数被复用成实参下标！）
      SEL_TREE *tree = nullptr;
      for (uint i = 1; i < cond_func->arg_count; i++) {
        Item *const arg = cond_func->arguments()[i];
        if (!arg->is_outer_reference() &&
            arg->real_item()->type() == Item::FIELD_ITEM) {
          Item_field *field_item = down_cast<Item_field *>(arg->real_item());
          SEL_TREE *tmp = get_full_func_mm_tree(
              thd, param, prev_tables, read_tables, current_table,
              remove_jump_scans, field_item, cond_func,
              reinterpret_cast<Item *>(i), inv);
          if (inv) {
            tree = !tree ? tmp : tree_or(param, remove_jump_scans, tree, tmp);
            if (tree == nullptr) break;
          } else
            tree = tree_and(param, tree, tmp);
        } else if (inv) {
          tree = nullptr;
          break;
        }
      }
      ftree = tree_and(param, ftree, tree);
      break;
    }

    case Item_func::JSON_CONTAINS:
    case Item_func::JSON_OVERLAPS:
    case Item_func::MEMBER_OF_FUNC:
    case Item_func::IN_FUNC: {         // ★ IN / NOT IN / 多值索引
      Item *predicand = cond_func->key_item();
      if (!predicand) return nullptr;
      predicand = predicand->real_item();
      if (predicand->type() != Item::FIELD_ITEM &&
          predicand->type() != Item::ROW_ITEM)
        return nullptr;
      ftree = get_full_func_mm_tree(thd, param, prev_tables, read_tables,
                                    current_table, remove_jump_scans, predicand,
                                    cond_func, nullptr, inv);
      break;
    }

    case Item_func::MULT_EQUAL_FUNC: {  // ★ 等值传播：a=b AND b=3 ⇒ 也给 a 建树
      Item_equal *item_equal = down_cast<Item_equal *>(cond);
      Item *value = item_equal->const_arg();
      if (value == nullptr) return nullptr;
      table_map ref_tables = value->used_tables();
      for (Item_field &field_item : item_equal->get_fields()) {
        Field *field = field_item.field;
        table_map param_comp = ~(prev_tables | read_tables | current_table);
        if (!((ref_tables | field_item.table_ref->map()) & param_comp)) {
          SEL_TREE *tree =
              get_mm_parts(thd, param, prev_tables, read_tables, item_equal,
                           field, Item_func::EQ_FUNC, value);
          ftree = !ftree ? tree : tree_and(param, ftree, tree);
        }
      }
      return ftree;
    }

    default: {          // ★ 其余：<, <=, =, >=, >, LIKE, IS NULL, GIS
      Item *const arg_left = cond_func->arguments()[0];
      assert(!ftree);
      if (!arg_left->is_outer_reference() &&
          arg_left->real_item()->type() == Item::FIELD_ITEM) {
        Item_field *field_item = down_cast<Item_field *>(arg_left->real_item());
        Item *value =
            cond_func->arg_count > 1 ? cond_func->arguments()[1] : nullptr;
        ftree = get_full_func_mm_tree(thd, param, prev_tables, read_tables,
                                      current_table, remove_jump_scans,
                                      field_item, cond_func, value, inv);
      }
      /*
        Even if get_full_func_mm_tree() was executed above and did not
        return a range predicate it may still be possible to create one
        by reversing the order of the operands ... WHERE t1.a OP t2.b ...
      */
      Item *arg_right;
      if (!ftree && cond_func->have_rev_func() &&      // ★ 反转："2 <= a" → "a >= 2"
          (arg_right = cond_func->arguments()[1]) &&
          !arg_right->is_outer_reference() &&
          arg_right->real_item()->type() == Item::FIELD_ITEM) {
        Item_field *field_item =
            down_cast<Item_field *>(arg_right->real_item());
        Item *value = arg_left;
        ftree = get_full_func_mm_tree(thd, param, prev_tables, read_tables,
                                      current_table, remove_jump_scans,
                                      field_item, cond_func, value, inv);
      }
    }
  }
  return ftree;
}
```

### 2.1.1 各算子的"总开关"：`get_func_mm_tree()`（`range_analysis.cc:553`）

> **勘误**：8.0 里**没有** `Item_func_eq::get_mm_tree()` 这类虚函数（`grep "::get_mm_tree"` 0 命中）。所有算子集中在这一个 `static` 函数的 switch 里；`get_mm_leaf()` / `get_mm_parts()` 也不是 `Field` 的成员，而是文件内 `static` 函数。

```cpp
static SEL_TREE *get_func_mm_tree(THD *thd, RANGE_OPT_PARAM *param,
                                  table_map prev_tables, table_map read_tables,
                                  bool remove_jump_scans, Item *predicand,
                                  Item_func *cond_func, Item *value, bool inv) {
  SEL_TREE *tree = nullptr;
  if (param->has_errors()) return nullptr;

  switch (cond_func->functype()) {
    case Item_func::XOR_FUNC:
      return nullptr;  // Always true (don't use range access on XOR). See WL#5800

    case Item_func::NE_FUNC:                 // ★ != → (x < v) OR (x > v)
      if (predicand->type() == Item::FIELD_ITEM) {
        Field *field = down_cast<Item_field *>(predicand)->field;
        tree = get_ne_mm_tree(thd, param, prev_tables, read_tables,
                              remove_jump_scans, cond_func, field, value, value);
      }
      break;

    case Item_func::BETWEEN:                 // ★ BETWEEN → GE + LE；NOT BETWEEN → get_ne_mm_tree
      if (predicand->type() == Item::FIELD_ITEM) {
        Field *field = down_cast<Item_field *>(predicand)->field;
        if (!value) {
          if (inv) {
            tree = get_ne_mm_tree(thd, param, prev_tables, read_tables,
                                  remove_jump_scans, cond_func, field,
                                  cond_func->arguments()[1],
                                  cond_func->arguments()[2]);
          } else {
            tree = get_mm_parts(thd, param, prev_tables, read_tables, cond_func,
                                field, Item_func::GE_FUNC,
                                cond_func->arguments()[1]);
            if (tree) {
              tree = tree_and(param, tree,
                              get_mm_parts(thd, param, prev_tables, read_tables,
                                           cond_func, field, Item_func::LE_FUNC,
                                           cond_func->arguments()[2]));
            }
          }
        } else
          tree = get_mm_parts(
              thd, param, prev_tables, read_tables, cond_func, field,
              (inv ? (value == reinterpret_cast<Item *>(1) ? Item_func::GT_FUNC
                                                           : Item_func::LT_FUNC)
                   : (value == reinterpret_cast<Item *>(1)
                          ? Item_func::LE_FUNC : Item_func::GE_FUNC)),
              cond_func->arguments()[0]);
      }
      break;

    case Item_func::IN_FUNC: {               // ★ IN / NOT IN
      Item_func_in *in_pred = down_cast<Item_func_in *>(cond_func);
      tree = get_func_mm_tree_from_in_predicate(thd, param, prev_tables,
                                                read_tables, remove_jump_scans,
                                                predicand, in_pred, inv);
    } break;

    case Item_func::JSON_CONTAINS:
    case Item_func::JSON_OVERLAPS: {
      tree = get_func_mm_tree_from_json_overlaps_contains(
          thd, param, prev_tables, read_tables, remove_jump_scans, predicand,
          cond_func);
    } break;

    case Item_func::MEMBER_OF_FUNC:          // ★ 多值索引（8.0.17+）
      ...
      break;

    default:                                 // ★ eq / lt / le / gt / ge / LIKE / ISNULL / GIS
      if (predicand->type() == Item::FIELD_ITEM) {
        Field *field = down_cast<Item_field *>(predicand)->field;
        /*
           Here the function for the following predicates are processed:
           <, <=, =, >=, >, LIKE, IS NULL, IS NOT NULL and GIS functions.
           If the predicate is of the form (value op field) it is handled
           as the equivalent predicate (field rev_op value), e.g.
           2 <= a is handled as a >= 2.
        */
        Item_func::Functype func_type =
            (value != cond_func->arguments()[0])
                ? cond_func->functype()
                : ((Item_bool_func2 *)cond_func)->rev_functype();
        tree = get_mm_parts(thd, param, prev_tables, read_tables, cond_func,
                            field, func_type, value);
      }
  }
  return tree;
}
```

**结构示意**（上面的递归骨架）：

```cpp
SEL_TREE *get_mm_tree(THD *thd, RANGE_OPT_PARAM *param, ...) {
  switch (cond->type()) {
    case Item::COND_ITEM: {                        // AND / OR
      // 递归每个子项生成 SEL_TREE，AND 用 tree_and、OR 用 tree_or
      // AND 得 IMPOSSIBLE 或 OR 得 ALWAYS 就提前 break
    }
    case Item::FUNC_ITEM: {
      switch (down_cast<Item_func *>(cond)->functype()) {
        case Item_func::BETWEEN:      // x BETWEEN a AND b
        case Item_func::IN_FUNC:      // x IN (...)
        case Item_func::EQ_FUNC:      // x = const
        case Item_func::LT_FUNC / LE_FUNC / GT_FUNC / GE_FUNC / NE_FUNC:
        case Item_func::ISNULL_FUNC:  // x IS NULL
        case Item_func::LIKE_FUNC:    // x LIKE 'prefix%'
        ...
      }
    }
  }
}
```

### 2.2 常量条件的短路（`:881-890`）

`val_int()` 为真 → `ALWAYS`（满区间），假 → `IMPOSSIBLE`（整个 range 优化放弃）。

### 2.3 BETWEEN 如何变成区间

`x BETWEEN a AND b` 被**拆成 `x >= a AND x <= b`**，分别调 `get_mm_parts(GE_FUNC)` / `get_mm_parts(LE_FUNC)` 生成两个 SEL_TREE，再 `tree_and` 求交。

`get_mm_parts`（`range_analysis.cc:1074`）造区间时定开闭，`get_mm_leaf`（`:1370`）分配区间节点：

```cpp
root = new SEL_ARG(field, str, str, ...);   // 先造 min==max 的单点 [x,x]
switch (type) {
  case Item_func::LT_FUNC:
  case Item_func::LE_FUNC:
    tree->root->max_flag = NEAR_MAX;      // 右开
    tree->root->min_flag = NO_MIN_RANGE;  // 下界 -inf
    break;
  case Item_func::GT_FUNC:
  case Item_func::GE_FUNC:
    tree->root->min_flag = NEAR_MIN;      // 左开
    tree->root->max_flag = NO_MAX_RANGE;  // 上界 +inf
    break;
}
```

**`SEL_ARG` 边界语义**（`tree.h:469`）：

| flag | 含义 |
|------|------|
| `NEAR_MIN` | 左开（`> key`） |
| `NEAR_MAX` | 右开（`< key`） |
| `NO_MIN_RANGE` | 下界 -inf |
| `NO_MAX_RANGE` | 上界 +inf |
| 两者为 0 且 min==max | 闭区间单点（等值） |

于是 `a<=x<=b` 经 `tree_and` 求交后得到 `min_value=a, max_value=b, flag=0` 的**闭区间 [a,b]**。

### 2.4 IN 列表如何变成红黑树多点（`range_analysis.cc:364-381`）

`get_func_mm_tree_from_in_predicate` 把 `col IN (c1..cN)` **显式展开成 DNF**：

```cpp
// range_analysis.cc:364
if (predicand->type() == Item::FIELD_ITEM) {
  Field *field = down_cast<Item_field *>(predicand)->field;
  SEL_TREE *tree = get_mm_parts(thd, param, prev_tables, read_tables, op,
                                field, Item_func::EQ_FUNC, op->arguments()[1]);
  if (tree) {
    Item **arg, **end;
    for (arg = op->arguments() + 2, end = arg + op->argument_count() - 2;
         arg < end; arg++) {
      tree = tree_or(param, remove_jump_scans, tree,
                     get_mm_parts(thd, param, prev_tables, read_tables, op,
                                  field, Item_func::EQ_FUNC, *arg));
    }
  }
  return tree;
}
```

**逐段解释**：

1. 每个列表项 `ci` 都被当成一个独立的 `col = ci` 等值谓词，先对 `args[1]`（第一项）调 `get_mm_parts` 得到第一棵 SEL_TREE。
2. 然后对 `args[2..N]` 循环：每项先 `get_mm_parts`（造单点区间 `[ci,ci]`），再 `tree_or`（内部 `key_or`）合并到累积结果。
3. 这是"**IN = col=c1 OR col=c2 OR ... OR col=cN**"的显式 DNF 展开——**不是**先构造 N 个 SEL_ARG 再一次性合并，而是**增量式**：每个元素生成一个叶子，逐次插入红黑树。

**复杂度 O(N log N)**：每个元素 `get_mm_parts` O(1) 造单点 + `key_or` 内部 `find_range` + `rb_insert` 各 O(log N)，N 个合计 **O(N log N)**。

> ⚠️ 注意：`(a,b) IN ((1,2),(3,4))` 的 ROW 形式（`:382-431`）构建 `(a=1 AND b=2) OR (a=3 AND b=4)` 的 DNF，逻辑类似。

**NULL 项的处理**（`range_analysis.cc:1570`）：IN 列表里的 NULL 走 `EQ_FUNC`（普通 `=`），`save_value_and_handle_conversion` 存 NULL 后 `field->is_real_null()` 为真 → 返回 `SEL_ROOT::Type::IMPOSSIBLE`，在 `key_or` 里被静默丢弃。所以 `col IN (1,2,NULL,3)` 最终只留下 `col IN (1,2,3)` 的等值区间，NULL/UNKNOWN 部分由执行期 `Item_func_in::val_int` 的 `have_null` 兜底。

### 2.5 LIKE 与 IS NULL

- **LIKE 'prefix%'**：可转 range（`prefix <= x < prefix_max`），因为 B-tree 有序；`LIKE '%x'` 不行（无前缀）。
- **IS NULL**：转成一个特殊的 NULL 区间（`is_null_interval()`，见 5.2 里"IS NULL 不计入 eq_range 计数"的说明）。

---

## 三、SEL_ARG 的三重结构与区间操作

### 3.1 三重结构（这是理解 range 优化的核心）

| 结构 | 位置 | 作用 |
|------|------|------|
| `SEL_ROOT` | `tree.h:69` | 单个索引的**红黑树森林根**（一层 keypart 的所有区间） |
| `SEL_ARG` | `tree.h:469` | 区间节点 |
| `SEL_TREE` | `tree.h:882` | 整个表的森林数组 `keys[]` + index merge 列表 `merges`；`Type{IMPOSSIBLE, ALWAYS, KEY}` |

`SEL_ARG` 同时有**三组指针**，这是它最难懂的地方：

```cpp
class SEL_ARG {
  // 1. next/prev：同一层 keypart 内，按 min 值排序的双向链表（用于顺序枚举区间）
  SEL_ARG *next, *prev;
  // 2. left/right/parent + color：红黑树（用于 log n 查找、插入平衡）
  SEL_ARG *left, *right, *parent;
  // 3. next_key_part：复合索引的下一层 keypart 的森林（"AND" 的层级展开）
  SEL_ROOT *next_key_part;
};
```

**三者的分工**：
- **红黑树**：支持快速查找/插入（构造时用）
- **next/prev 链表**：支持按序遍历（枚举区间、合并时用）
- **next_key_part**：复合索引的下一层级（`kp1=1 AND kp2 IN(1,2)` 时，`kp2` 的森林挂在 `kp1=1` 这个节点的下面）

#### 3.1.1 三个类的权威定义（`sql/range_optimizer/tree.h`）

```cpp
// sql/range_optimizer/tree.h:69
/**
  A graph of (possible multiple) key ranges, represented as a red-black
  binary tree. There are three types (see the Type enum); if KEY_RANGE,
  we have zero or more SEL_ARGs, described in the documentation on SEL_ARG.

  As a special case, a nullptr SEL_ROOT means a range that is always true.
  This is true both for keys[] and next_key_part.
*/
class SEL_ROOT {
 public:
  enum class Type {          // ★ 8.0 是 enum class，必须写全 SEL_ROOT::Type::KEY_RANGE
    /** The range predicate for this index is always false. */
    IMPOSSIBLE,
    /**
      There is a range predicate that refers to another table. The range
      access method cannot be used on this index unless that other table is
      earlier in the join sequence. The bit representing the index is set in
      JOIN_TAB::needed_reg ... After deciding on join order, the optimizer may
      chose to rerun the range optimizer for tables with such dependencies.
    */
    MAYBE_KEY,
    /** There is a range condition that can be used on this index. */
    KEY_RANGE
  } type;

  SEL_ARG *root;             // ★ 红黑树根；type==KEY_RANGE 且树空时是 null_element（等价 IMPOSSIBLE）

  /**
    Number of references to this SEL_ARG tree. References may be from
    SEL_ARG::next_key_part of SEL_ARGs from earlier keyparts or
    SEL_TREE::keys[i]. The SEL_ARG trees are re-used in a lazy-copy manner
    based on this reference counting.        // ★ 写时复制靠 use_count
  */
  ulong use_count{0};

  /** Number of nodes in the RB-tree, not including sentinels. */
  uint16 elements{0};

  void insert(SEL_ARG *key);            // 插入 + rb_insert 重平衡
  void tree_delete(SEL_ARG *key);
  SEL_ARG *find_range(const SEL_ARG *key) const;
  SEL_ROOT *clone_tree(RANGE_OPT_PARAM *param) const;
  int store_min_key(KEY_PART *key, uchar **range_key, uint *range_key_flag,
                    uint last_part, bool start_key);
  int store_max_key(...);
};
```

```cpp
// sql/range_optimizer/tree.h:469
class SEL_ARG {
 public:
  uint8 min_flag{0}, max_flag{0};      // ★ 位标记，取值见 include/my_base.h:1079 enum key_range_flags

  /**
    maybe_flag signals that this range is AND-ed with some unknown range
    (a MAYBE_KEY node). This means that the range could be smaller than
    what it would otherwise denote; e.g.
      (0 < x < 3) AND x=( SELECT ... )
    could in reality be e.g. (1 < x < 2) ...
  */
  bool maybe_flag{false};

  uint8 part{0};                       // 第几个 keypart（0-based）

  enum ha_rkey_function rkey_func_flag; // 仅当 min_flag == GEOM_FLAG 时有效
  Field *field{nullptr};
  uchar *min_value, *max_value;         // Pointer to range   ★ [min,max] 存在这里

  // rb tree
  SEL_ARG *left, *right;                /* R-B tree children */
  SEL_ARG *next, *prev;                 /* Links for bi-directional interval list */
  SEL_ARG *parent{nullptr};             /* R-B tree parent (nullptr for root) */
  /*
    R-B tree of intervals covering keyparts consecutive to this SEL_ARG. ...
  */
  SEL_ROOT *next_key_part{nullptr};     // ★★ next_key_part = AND

  enum leaf_color { BLACK, RED } color;
  bool is_ascending;                    // ★ 8.0 的降序索引支持

  /* 区间运算 */
  SEL_ARG *clone_and(SEL_ARG *arg, MEM_ROOT *mem_root) {  // 求交
    uchar *new_min, *new_max;
    uint8 flag_min, flag_max;
    if (cmp_min_to_min(arg) >= 0) { new_min = min_value;  flag_min = min_flag; }
    else                          { new_min = arg->min_value; flag_min = arg->min_flag; }
    if (cmp_max_to_max(arg) <= 0) { new_max = max_value;  flag_max = max_flag; }
    else                          { new_max = arg->max_value; flag_max = arg->max_flag; }
    return new (mem_root)
        SEL_ARG(field, part, new_min, new_max, flag_min, flag_max,
                maybe_flag && arg->maybe_flag, is_ascending,
                min_flag & GEOM_FLAG ? rkey_func_flag : HA_READ_INVALID);
  }
  bool copy_min(SEL_ARG *arg) { ... }   // max(this->min, arg->min) <= x <= this->max
  bool copy_max(SEL_ARG *arg) { ... }

  /* 把 [min,max] 写进 key buffer —— 注意"开区间后不再写后续 keypart" */
  int store_min_value(uint length, uchar **min_key, uint min_key_flag) {
    /* "(kp1 > c1) AND (kp2 OP c2) AND ..." -> (kp1 > c1) */
    if ((min_flag & GEOM_FLAG) ||
        (!(min_flag & NO_MIN_RANGE) &&
         !(min_key_flag & (NO_MIN_RANGE | NEAR_MIN)))) {
      if (maybe_null() && *min_value) {
        **min_key = 1;
        memset(*min_key + 1, 0, length - 1);
      } else
        memcpy(*min_key, min_value, length);
      (*min_key) += length;
      return 1;       // ★ 返回 1 表示这个 keypart 真的写进了 key buffer
    }
    return 0;         // ★ 前面已是开区间 ⇒ 后续 keypart 对 range 边界无贡献
  }
  int store_max_value(uint length, uchar **max_key, uint max_key_flag) { ... }

  void store_min_max_values(uint length, uchar **min_key, uint min_flag,
                            uchar **max_key, uint max_flag, int *min_part,
                            int *max_part) {
    if (is_ascending) {      // ★ DESC keypart 时 min/max 互换
      *min_part += store_min_value(length, min_key, min_flag);
      *max_part += store_max_value(length, max_key, max_flag);
    } else {
      *max_part += store_min_value(length, max_key, min_flag);
      *min_part += store_max_value(length, min_key, max_flag);
    }
  }

  SEL_ARG *rb_insert(SEL_ARG *leaf);
  uint get_max_flag() {
    return (is_ascending ? max_flag : invert_min_flag(min_flag));  // ★ DESC 翻转
  }
};
```

```cpp
// sql/range_optimizer/tree.h:882 —— 森林
class SEL_TREE {
 public:
  /**
    IMPOSSIBLE: if keys[i]->type == SEL_ROOT::Type::IMPOSSIBLE for some i,
      then type == SEL_TREE::IMPOSSIBLE. ...
    ALWAYS: if either (keys[i]->is_always()) or (keys[i] == NULL) for all i,
      then type == SEL_TREE::ALWAYS. ...
    KEY: There are range predicates that can be used on at least one index.
  */
  enum Type { IMPOSSIBLE, ALWAYS, KEY } type;

  /**
    Whether this SEL_TREE is an inexact (too broad) representation of the
    predicates it is based on; ... If a SEL_TREE is inexact, the predicates
    must be rechecked after the range scan, using a filter. ...
  */
  bool inexact = false;

  Mem_root_array<SEL_ROOT *> keys;   // ★★ 每个索引一棵 SEL_ROOT —— 这就是"森林"
  Key_map keys_map;                  /* bitmask of non-NULL elements in keys */

  /* Possible ways to read rows using Index merge (sort) union. */
  List<SEL_IMERGE> merges;           // ★ index merge union 候选

  /* The members below are filled/used only after get_mm_tree is done */
  Key_map ror_scans_map;             /* bitmask of ROR scan-able elements in keys */
  uint n_ror_scans;
  ROR_SCAN_INFO **ror_scans;
  ROR_SCAN_INFO **ror_scans_end;
};
```

**SEL_ARG 的语义一句话**（`tree.h:248` 的注释原文）：**`next/prev` 是 OR，`next_key_part` 是 AND**。所以它不是"每个维度一棵独立树"，而是**区间图（interval graph）**：一条 `kp1 区间 → next_key_part → kp2 区间 → …` 的路径表示一个多维盒子，`[min_value,max_value] + min_flag/max_flag`（`NEAR_MIN/NEAR_MAX/NO_MIN_RANGE/NO_MAX_RANGE`）描述每一维的闭/开/无穷端点。

### 3.2 insert（`tree.cc:1727`）

按 min 值二叉查找插入位置 → `rb_insert` 重平衡红黑树 → 维护 `next/prev` 有序链表。所以 `IN(1,2,3)` 是一棵含 3 个节点、按值有序的红黑树 + 一条 1→2→3 的链表。

### 3.3 key_and：求交（`tree.cc:896`）

同 part 的两棵红黑树**归并**——双指针按序扫描，`get_range` 跳过无重叠区间，重叠时 `clone_and` 求交并递归合并 `next_key_part`。交集公式（`clone_and`，`tree.h:604`）：

```cpp
// 交集 = max(min) + min(max)
if (cmp_min_to_min(arg) >= 0) new_min = min_value; else new_min = arg->min_value;
if (cmp_max_to_max(arg) <= 0) new_max = max_value; else new_max = arg->max_value;
```

无交集返回 `IMPOSSIBLE`（`x>10 AND x<5` 这类恒假）。

### 3.4 key_or：求并（`tree.cc:1089`）

比 AND 复杂，`find_range` 定位起始区间后三种位置关系：

- **相邻**（且 `next_key_part` 相同）→ 无缝并成一个连续区间
- **完全分离** → 直接 `insert` 成独立新区间
- **重叠** → 向后扫描被覆盖的区间逐个 `tree_delete`，取 `min(min)` + `max(max)` 扩展并集

若合并结果覆盖 `[-inf,+inf]` 即满区间恒真。

### 3.5 复合索引的 next_key_part 递归

`kp1=1 AND kp2 IN(1,2)` 时，`key_and` 合并两个 `kp1` 节点会递归合并它们的 `next_key_part`（`tree.cc:982`），把 `kp2` 的区间树挂到 `kp1` 节点的 `next_key_part` 上，形成"keypart 层级图"。

**这是 range 优化能处理复合索引的关键**——`(a,b)` 上的 `a=1 AND b>5` 会变成"a 层的 [1,1] 节点，其 next_key_part 是 b 层的 (5,+inf]"。

---

## 四、各 range 访问方法的判定与实现

`test_quick_select` 依次尝试以下访问方法，各自产出一种 `AccessPath::Type`：

### 4.1 INDEX_RANGE_SCAN（普通 range，`get_key_scans_params`）

`index_range_scan_plan.cc:820`。对每个可用索引算 range 扫描的 rows 和 cost，取代价最小者。
- 产出 `AccessPath::INDEX_RANGE_SCAN`（`index_range_scan_plan.cc:972`）
- EXPLAIN 显示 `range`

**完整源码**：

```cpp
// sql/range_optimizer/index_range_scan_plan.cc:820
AccessPath *get_key_scans_params(THD *thd, RANGE_OPT_PARAM *param,
                                 SEL_TREE *tree, bool index_read_must_be_used,
                                 bool update_tbl_stats,
                                 enum_order order_direction,
                                 bool skip_records_in_range,
                                 const double cost_est, bool ror_only,
                                 Key_map *needed_reg) {
  uint idx, best_idx = 0;
  SEL_ROOT *key, *key_to_read = nullptr;
  ha_rows best_records = 0;
  uint best_mrr_flags = 0, best_buf_size = 0;
  double read_cost = cost_est;      // ★ 初值是外部传进来的 best_cost（全表/覆盖索引）
  Opt_trace_context *const trace = &thd->opt_trace;
  Opt_trace_array ota(trace, "range_scan_alternatives");   // ★ optimizer_trace 的这一节

  tree->ror_scans_map.clear_all();
  tree->n_ror_scans = 0;
  bool use_cheapest_index_merge = false;
  bool force_index_merge =
      idx_merge_hint_state(thd, param->table, &use_cheapest_index_merge);

  for (idx = 0; idx < param->keys; idx++) {      // ★ 遍历"可用"索引（param->keys）
    key = tree->keys[idx];
    if (key) {                                   // 跳过 A：该索引没有 SEL_ARG 树
      ha_rows found_records;
      Cost_estimate cost;
      uint mrr_flags = 0, buf_size = 0;
      uint keynr = param->real_keynr[idx];
      if (key->type == SEL_ROOT::Type::MAYBE_KEY || key->root->maybe_flag)
        needed_reg->set_bit(keynr);              // ★ 通知 join 优化器"有表依赖"

      bool read_index_only =
          index_read_must_be_used
              ? true
              : (bool)param->table->covering_keys.is_set(keynr);

      Opt_trace_object trace_idx(trace);
      trace_idx.add_utf8("index", param->table->key_info[keynr].name);
      bool is_ror_scan, is_imerge_scan;
      found_records = check_quick_select(
          thd, param, idx, read_index_only, key, update_tbl_stats,
          order_direction, skip_records_in_range, &mrr_flags, &buf_size, &cost,
          &is_ror_scan, &is_imerge_scan);

      if (found_records != HA_POS_ERROR && ror_only && !is_ror_scan) {
        trace_idx.add("chosen", false).add_alnum("cause", "not_rowid_ordered");
        continue;                                // 跳过 B：ROR-union 构建时只要 ROR 扫描
      }
      if (!compound_hint_key_enabled(param->table, keynr,
                                     INDEX_MERGE_HINT_ENUM)) {
        trace_idx.add("chosen", false).add_alnum("cause", "index_merge_hint");
        continue;                                // 跳过 C：被 INDEX_MERGE hint 排除
      }

      if (found_records != HA_POS_ERROR && thd->opt_trace.is_started()) {
        Opt_trace_array trace_range(&thd->opt_trace, "ranges");
        ... append_range_all_keyparts(...) ...

        /// No cost calculation when index dive is skipped.
        if (skip_records_in_range)
          trace_idx.add_alnum("index_dives_for_range_access",
                              "skipped_due_to_force_index");
        else
          trace_idx.add("index_dives_for_eq_ranges",
                        !param->use_index_statistics);   // ★ trace 里这一项 = eq_range_index_dive_limit 的结果

        trace_idx.add("rowid_ordered", is_ror_scan)
            .add("using_mrr", !(mrr_flags & HA_MRR_USE_DEFAULT_IMPL))
            .add("index_only", read_index_only)
            .add("in_memory", cur_key.in_memory_estimate());
        ...
      }

      if ((found_records != HA_POS_ERROR) && is_ror_scan) {
        tree->n_ror_scans++;
        tree->ror_scans_map.set_bit(idx);        // ★ 给后面的 get_best_ror_intersect 用
      }

      if (found_records != HA_POS_ERROR &&
          (read_cost > cost.total_cost() ||
           /* Ignore cost check if INDEX_MERGE hint is used ... */
           (force_index_merge &&
            (!use_cheapest_index_merge || !key_to_read)))) {
        trace_idx.add("chosen", true);
        read_cost = cost.total_cost();
        best_records = found_records;
        key_to_read = key;
        best_idx = idx;
        best_mrr_flags = mrr_flags;
        best_buf_size = buf_size;
        is_best_idx_imerge_scan = is_imerge_scan;
      } else {
        trace_idx.add("chosen", false);
        if (found_records == HA_POS_ERROR)
          if (key->type == SEL_ROOT::Type::MAYBE_KEY)
            trace_idx.add_alnum("cause", "depends_on_unread_values");  // 跳过 D
          else
            trace_idx.add_alnum("cause", "no_valid_range_for_this_index");
        else
          trace_idx.add_alnum("cause", "cost");                        // 跳过 E：比全表还贵
      }
    }
  }

  if (key_to_read == nullptr) return nullptr;

  Quick_ranges ranges(param->return_mem_root);
  unsigned used_key_parts, num_exact_key_parts;
  if (get_ranges_from_tree(param->return_mem_root, param->table,
                           param->key[best_idx], param->real_keynr[best_idx],
                           key_to_read, MAX_REF_PARTS, &used_key_parts,
                           &num_exact_key_parts, &ranges)) {
    return nullptr;
  }

  KEY *used_key = &param->table->key_info[param->real_keynr[best_idx]];

  // ★★ range 结果落成 AccessPath 的地方
  AccessPath *path = new (param->return_mem_root) AccessPath;
  path->type = AccessPath::INDEX_RANGE_SCAN;
  path->cost = read_cost;
  path->set_num_output_rows(best_records);
  path->index_range_scan().index = param->real_keynr[best_idx];
  path->index_range_scan().num_used_key_parts = used_key_parts;
  path->index_range_scan().used_key_part = param->key[best_idx];
  path->index_range_scan().ranges = &ranges[0];
  path->index_range_scan().num_ranges = ranges.size();
  path->index_range_scan().mrr_flags = best_mrr_flags;
  path->index_range_scan().mrr_buf_size = best_buf_size;
  path->index_range_scan().can_be_used_for_ror =
      tree->ror_scans_map.is_set(best_idx);
  path->index_range_scan().need_rows_in_rowid_order =
      false;  // May be changed by callers later.
  path->index_range_scan().can_be_used_for_imerge = is_best_idx_imerge_scan;
  path->index_range_scan().reuse_handler = false;
  path->index_range_scan().geometry = (used_key->flags & HA_SPATIAL);
  path->index_range_scan().reverse =
      false;  // May be changed by make_reverse() later.
  return path;
}
```

**"何时跳过某个索引"逐条**：

| # | 判定 | 位置 | trace 的 cause |
|---|---|---|---|
| A | `tree->keys[idx] == nullptr`（该索引无可用区间） | `:851` | 无（连 trace object 都不建） |
| B | `ror_only && !is_ror_scan` | `:872` | `not_rowid_ordered` |
| C | 被 `INDEX_MERGE` hint 排除 | `:876` | `index_merge_hint` |
| D | `check_quick_select()` 返回 `HA_POS_ERROR` | `:941` | `depends_on_unread_values` / `no_valid_range_for_this_index` |
| E | `read_cost <= cost.total_cost()`（比全表还贵） | `:921` | `cost` |

⇒ **坑**：`force_index_merge` 分支会**绕过代价比较**强行选中第一个可用索引（`:929`）。另外 D 类失败时 `ror_scans_map` 也不置位，所以该索引后续也无法参与 ROR-intersection。

### 4.2 INDEX_SKIP_SCAN（skip scan，`get_best_skip_scan`）

`index_skip_scan_plan.cc:502`。**复合索引第一个 keypart 没有谓词，但后续 keypart 有时**仍能用索引：

```
索引 (gender, age)，WHERE age > 30（gender 无谓词）
→ 枚举 gender 的每个 distinct 值，在每个值内做 age>30 的 range 扫描（"跳跃"扫描）
```

适用条件：第一个 keypart 的 NDV 较小（枚举成本可接受）。产出 `INDEX_SKIP_SCAN`。

### 4.3 GROUP_INDEX_SKIP_SCAN（MIN/MAX 优化，`get_best_group_min_max`）

`group_index_skip_scan_plan.cc:996`。**`SELECT MIN(a) FROM t` 可以直接取索引端点**（B-tree 最左/最右叶子），不需要扫全表。

```
MIN(a) → 索引第一个条目；MAX(a) → 索引最后一个条目
GROUP BY b 时 → 对每个 b 的分组各取一个端点
```

产出 `GROUP_INDEX_SKIP_SCAN`。这是"为什么 `MIN/MAX` 走索引极快"的原因。

### 4.4 ROWID_INTERSECTION（ROR-intersection，`get_best_ror_intersect`）

`rowid_ordered_retrieval_plan.cc:770`。**多个索引的 range 各取 rowid，取交集后再回表**：

```
WHERE a=1 AND b=2（a、b 各有一个索引）
→ 用 idx_a 扫出 rowid 集合 A，用 idx_b 扫出 rowid 集合 B
→ 取 A∩B（有序 rowid 归并），再按 rowid 回表取整行
```

条件：各索引扫描必须能**按 rowid 顺序**输出（`HA_KEY_SCAN_NOT_ROR` 未设置）。产出 `ROWID_INTERSECTION`。

#### 4.4.1 贪心搜索（`find_intersect_order` + `get_best_ror_intersect`）

真实实现分两步，函数名与旧资料不同（`find_min_ror_intersection_scan` 只是伪代码注释）：

**排序**（`find_intersect_order`，:229）：**选择排序**而非 `std::sort`——每轮选出"覆盖剩余字段最多的索引"，把它覆盖的字段从 `fields_to_cover` 里减掉：

```cpp
for (place = start; place < end - 1; place++) {
  for (current = place + 1; current < end; current++) {
    bitmap_intersect(&covered_fields_remaining, &fields_to_cover);  // 重算剩余覆盖数
    if (is_better_intersect_match(*best, *current)) best = current;
  }
  bitmap_subtract(&fields_to_cover, &(*best)->covered_fields);
  if (bitmap_is_clear_all(&fields_to_cover)) return;   // 列已全覆盖
}
```

比较器 `is_better_intersect_match`（:196）：先 `num_covered_fields_remaining` 多者优先（覆盖剩余列多），再 `records` 少者优先（扫描行数少）。目标：**用尽量少的索引覆盖尽量多字段**。注释明确承认"不是全排列，贪心可能找不到全局最优"。

**增量贪心**（`get_best_ror_intersect`，:862）：按排序后顺序逐个尝试加入交集：

```cpp
while (cur_ror_scan != ror_scans_end && !intersect->is_covering) {
  if (!ror_intersect_add(intersect, needed_fields, *cur_ror_scan, false, ...)) {
    cur_ror_scan++; continue;     // 不降低代价 → 跳过
  }
  *(intersect_scans_end++) = *cur_ror_scan++;
  if (intersect->total_cost < min_cost) {   // 维护局部最优
    ror_intersect_cpy(intersect_best, intersect);
    intersect_scans_best = intersect_scans_end;
    min_cost = intersect->total_cost;
  }
}
```

早停条件 `!intersect->is_covering`：一旦交集已覆盖所有 `needed_fields`（覆盖索引），不再需要更多索引（再也不会回表）。

#### 4.4.2 选择率乘数（`ror_intersect_add` + `ror_scan_selectivity`，bad case 根源）

`ror_intersect_add`（:610）核心：

```cpp
double selectivity_mult = ror_scan_selectivity(info, ror_scan);
if (selectivity_mult == 1.0 && !ignore_cost) return false;  // 不提升选择率 → 拒绝
info->out_rows *= selectivity_mult;                         // 更新输出行数
// 非 CPK：累加 index_records / index_scan_cost，union covered_fields
bitmap_union(&info->covered_fields, &ror_scan->covered_fields);
if (!is_covering && bitmap_is_subset(needed_fields, &covered_fields)) is_covering = true;
info->total_cost = info->index_scan_cost;
if (!info->is_covering)
  get_sweep_read_cost(table, out_rows, ..., &sweep_cost);   // 未覆盖 → 加回表
```

`ror_scan_selectivity`（:464）计算乘数——把索引的 keypart 按"是否已被交集覆盖"分段，**未覆盖段的联合选择率连乘**：

```cpp
ha_rows prev_records = stats.records;   // n_0 = 全表行数
for (SEL_ROOT *sel_root = scan->sel_root; sel_root; sel_root = next_key_part) {
  cur_covered = 该 keypart 字段是否已被 covered_fields 覆盖;
  if (cur_covered != prev_covered) {    // 覆盖状态翻转才 fetch 一次前缀行数
    // 构造 part0..part(p-1) 前缀 key，取 n_{p-1}
    if (use_index_statistics && has_records_per_key(part))
      records = records_per_key(part);              // 单列统计
    else
      records = records_in_range(keynr, min_range, max_range);  // index dive
    if (cur_covered)  selectivity_mult *= records / prev_records;  // uncovered→covered
    else              prev_records = records;                     // covered→uncovered
  }
  prev_covered = cur_covered;
}
if (!prev_covered)  selectivity_mult *= quick_rows[keynr] / prev_records;  // 最后一段
```

**核心公式**：`selectivity_mult = Π(n_k / n_{k-1})`，只对"未覆盖段"连乘；已覆盖段（该列已在别的索引里出现过）选择率 = 1。`n_{p-1}` 用 `records_in_range`（前缀 key index dive）或单列 `records_per_key` 估算。

> ⚠️ **bad case 根因**：`covered_fields` 只剔除"同一列已在别的索引出现过"，**剩余的不同列之间一律当作独立相乘**（`:415` 注释"F and t are considered independent"）。若两列强相关（如 `status` 与 `category` 耦合），独立乘积远小于真实条件概率 → `out_rows` 严重低估 → 回表代价低估 → 错选 index merge。**源码里没有任何 correlation / dependency 检测**（`test_quick_select` 注释 `range_optimizer.cc:484` 把它列为 TODO）。

#### 4.4.3 回表代价（`get_sweep_read_cost`，`handler.cc:7230`）

```cpp
double n_blocks = ceil(data_file_length / IO_SIZE);
double busy_blocks = n_blocks * (1.0 - pow(1.0 - 1.0/n_blocks, rows));  // 用 rows 当随机点
cost->add_io(cost_model->page_read_cost(busy_blocks));
```

**"用 rows 当 pages"的精确含义**：把 `nrows` 个 rowid 当随机点，套"行在页上均匀独立分布"模型，估计会命中多少个 distinct 页（`busy_blocks`），再按内存/磁盘页计费。不是直接把 rows 当 pages。

### 4.5 ROWID_UNION（ROR-union，`get_ror_union_path`）

`range_optimizer.cc:837`（旧资料写 952 是错的）。**多个 OR 分支各取 rowid，取并集**：

```cpp
// 对每个 OR 分支：复用 get_best_ror_intersect 求该分支最优计划（可为多索引交集）
for (tree_it : imerge->trees) {
  *cur_roru_plan = get_best_ror_intersect(..., *tree_it, ...);
  roru_index_cost += (*cur_roru_plan)->cost;                    // 累加索引扫描代价
  roru_total_records += (*cur_roru_plan)->num_output_rows();
  roru_intersect_part *= (*cur_roru_plan)->num_output_rows() / stats.records;  // 去重率累乘
}
roru_total_records -= roru_intersect_part * stats.records;      // 扣掉全体重叠项
// 总代价 = 回表 + 索引扫描和 + 优先级队列去重
roru_total_cost = sweep_cost + roru_index_cost
                + key_compare_cost(roru_total_records * log2(n_scans));
```

**去重率公式**（展开后）：`total = SUM(rows_i) - table_rows * PROD(rows_i / table_rows)`——即容斥原理的**第一层近似**（只扣"全体共同"一项，交叉项用非相关性假设替代）。代码注释（`range_optimizer.cc:896`）说明前提是"各分支不共享 keypart"。

**去重代价**：ROR-Union 运行时用大小为 `n_scans` 的**最小堆多路归并**去重，比较次数 `total_records * log2(n)`，代价 `key_compare_cost(...)`。`ROWID_UNION` 的孩子可以是 `ROWID_INTERSECTION`（`(a=1 AND b=2) OR (c=3)` 这种混合）。

### 4.6 INDEX_MERGE（sort-union，`get_best_disjunct_quick`）

`range_optimizer.cc:1038`（旧资料写 1203 是错的）。当各 OR 分支**不能按 rowid 顺序输出**时，退化为"先排序 rowid 再归并"（sort-union）。产出 `INDEX_MERGE`。

代价累加：

```cpp
for (tree_it : imerge->trees) {
  *cur_child = get_key_scans_params(...);      // 每分支最优单索引 range
  imerge_cost += (*cur_child)->cost;           // 累加索引扫描代价
  non_cpk_scan_records += (*cur_child)->num_output_rows();   // 非 CPK 行数
}
// 回表（只对非 CPK 行）+ Unique 去重
get_sweep_read_cost(table, non_cpk_scan_records, ..., &sweep_cost);
imerge_cost += sweep_cost;
imerge_cost += Unique::get_use_cost(non_cpk_scan_records, ref_length, sortbuff_size, cost_model);
// 输出行数：不扣重叠项！
imerge_path->set_num_output_rows(min(non_cpk + cpk, stats.records));
```

**`Unique::get_use_cost`**（`sql/uniques.cc:572`）：内存 RB-tree 建树（每插入约 `2*log2(n)` 次比较，代价 `key_compare_cost`）+ 元素溢出内存时写树到磁盘（`disk_seek_base_cost`）+ 多路归并 + 读回结果。

> ⚠️ **Sort-Union 与 ROR-Union 的关键区别**：Sort-Union **不算去重率**——输出行数 = `min(Σ分支行数, 表行数)`，只把各分支行数直接相加兜底到表行数，**不扣重叠项**；而 ROR-Union 显式扣了 `PROD` 重叠项。后果：Sort-Union 的 rows 估计偏悲观（高估），回表代价也按未去重的 `non_cpk_scan_records` 算。

> **执行期**：三种 index merge 的运行形态（`RowIDIntersectionIterator` / `RowIDUnionIterator` / `IndexMergeIterator`）是火山模型迭代器，见 [`../../09_executor_iterator.md`](../../09_executor_iterator.md) 2.3 节。上面估算的回表（`ha_rnd_pos`）、堆归并（`key_compare_cost`）、去重（`Unique`）各自对应那里的执行步骤。

### 4.7 各访问方法汇总

| AccessPath::Type | 生成函数 | 文件:行 |
|---|---|---|
| `INDEX_RANGE_SCAN` | `get_key_scans_params` | `index_range_scan_plan.cc:972` |
| `INDEX_SKIP_SCAN` | `get_best_skip_scan` | `index_skip_scan_plan.cc:502` |
| `GROUP_INDEX_SKIP_SCAN` | `get_best_group_min_max` | `group_index_skip_scan_plan.cc:996` |
| `ROWID_INTERSECTION` | `get_best_ror_intersect` | `rowid_ordered_retrieval_plan.cc:770` |
| `INDEX_MERGE` | `get_best_disjunct_quick` | `range_optimizer.cc:1038` |
| `ROWID_UNION` | `get_ror_union_path` | `range_optimizer.cc:837` |
| `DYNAMIC_INDEX_RANGE_SCAN` | 执行期包装 | `ref_row_iterators.cc:542` |

---

## 五、代价与行数估算（index dive vs 统计）

### 5.1 check_quick_select（`index_range_scan_plan.cc:568`）

把 SEL_ARG 图包装成 `RANGE_SEQ_IF` 迭代器（`sel_arg_range_seq_init`/`_next`，`:218`/`:306`），交给 handler 的 `multi_range_read_info_const` **边遍历边累加 rows**——**这就是 index dive 的实际发生点**。

**完整源码**（`index_range_scan_plan.cc:568`）：

```cpp
ha_rows check_quick_select(THD *thd, RANGE_OPT_PARAM *param, uint idx,
                           bool index_only, SEL_ROOT *tree,
                           bool update_tbl_stats, enum_order order_direction,
                           bool skip_records_in_range, uint *mrr_flags,
                           uint *bufsize, Cost_estimate *cost,
                           bool *is_ror_scan, bool *is_imerge_scan) {
  uchar min_key[MAX_KEY_LENGTH + MAX_FIELD_WIDTH];
  uchar max_key[MAX_KEY_LENGTH + MAX_FIELD_WIDTH];

  Sel_arg_range_sequence seq(param, is_ror_scan, min_key, max_key,
                             skip_records_in_range);
  RANGE_SEQ_IF seq_if = {sel_arg_range_seq_init, sel_arg_range_seq_next,
                         nullptr};
  // ★ 优化期直接遍历 SEL_ARG 图（不是 QUICK_RANGE 数组），省一次物化
  handler *file = param->table->file;
  ha_rows rows;
  uint keynr = param->real_keynr[idx];

  /* Handle cases when we don't have a valid non-empty list of range */
  if (!tree) return HA_POS_ERROR;
  if (tree->type == SEL_ROOT::Type::IMPOSSIBLE) return 0L;
  if (tree->type != SEL_ROOT::Type::KEY_RANGE || tree->root->part != 0)
    return HA_POS_ERROR;   // ★ 第一个 keypart 没条件 ⇒ 不能用 range

  seq.keyno = idx;
  seq.real_keyno = keynr;
  seq.start = tree->root;
  seq.range_count = 0;
  seq.max_key_part = 0;

  /*
    If there are more equality ranges than specified by the
    eq_range_index_dive_limit variable we switches from using index
    dives to use statistics.                       // ★★ eq_range_index_dive_limit 判定点
  */
  uint range_count = 0;
  param->use_index_statistics = eq_ranges_exceeds_limit(
      tree, &range_count, thd->variables.eq_range_index_dive_limit);

  *is_imerge_scan = true;
  *is_ror_scan = !(file->index_flags(keynr, 0, true) & HA_KEY_SCAN_NOT_ROR);

  *mrr_flags = (order_direction == ORDER_DESC) ? HA_MRR_USE_DEFAULT_IMPL : 0;
  *mrr_flags |= HA_MRR_NO_ASSOCIATION;
  /* Pass HA_MRR_SORTED to see if MRR implementation can handle sorting. */
  if (order_direction != ORDER_NOT_RELEVANT) *mrr_flags |= HA_MRR_SORTED;

  bool pk_is_clustered = file->primary_key_is_clustered();
  if (index_only &&
      (file->index_flags(keynr, seq.max_key_part, true) & HA_KEYREAD_ONLY) &&
      !(pk_is_clustered && keynr == param->table->s->primary_key))
    *mrr_flags |= HA_MRR_INDEX_ONLY;

  if (thd->lex->sql_command != SQLCOM_SELECT)
    *mrr_flags |= HA_MRR_SORTED;  // Assumed to give faster ins/upd/del

  *bufsize = thd->variables.read_rnd_buff_size;
  // Sets is_ror_scan to false for some queries, e.g. multi-ranges
  rows = file->multi_range_read_info_const(keynr, &seq_if, (void *)&seq, 0,
                                           bufsize, mrr_flags, cost);
  // ★★ 行数与代价都在这里算：内部逐 range 调 records_in_range()

  if (rows != HA_POS_ERROR) {
    param->table->quick_rows[keynr] = rows;      // ★ 第 9 章的 condition filtering 会消费它
    if (update_tbl_stats) {
      param->table->quick_keys.set_bit(keynr);
      param->table->quick_key_parts[keynr] = seq.max_key_part + 1;
      param->table->quick_n_ranges[keynr] = seq.range_count;
      param->table->quick_condition_rows =
          min(param->table->quick_condition_rows, rows);
    }
    param->table->possible_quick_keys.set_bit(keynr);
  }

  /*
    Check whether ROR scan could be used. It cannot be used if
    1. Index algo is not HA_KEY_ALG_BTREE or HA_KEY_ALG_SE_SPECIFIC ...
    2. Any of the keyparts in the index chosen is descending. Desc
       indexes do not work well for ROR scans, except for clustered PK.
    3. SE states the index can't be used for ROR. ...
    4. Index contains virtual columns. RowIDIntersectionIterator
       and RowIDUnionIterator do read_set manipulations in reset(),
       which breaks virtual generated column's computation logic ...
  */
  enum ha_key_alg key_alg = param->table->key_info[seq.real_keyno].algorithm;

  KEY_PART_INFO *key_part = param->table->key_info[keynr].key_part;
  KEY_PART_INFO *key_part_end =
      key_part + param->table->key_info[keynr].user_defined_key_parts;
  for (; key_part != key_part_end; ++key_part) {
    if (key_part->key_part_flag & HA_REVERSE_SORT) {
      // ROR will be enabled again for clustered PK, see 'else if' below.
      *is_ror_scan = false;
      *is_imerge_scan = false;      // ★ 降序 keypart 同时也不能做 index merge
      break;
    }
  }
  if (((key_alg != HA_KEY_ALG_BTREE) &&
       (key_alg != HA_KEY_ALG_SE_SPECIFIC)) ||
      (file->index_flags(keynr, 0, true) & HA_KEY_SCAN_NOT_ROR) ||
      param->table->index_contains_some_virtual_gcol(keynr)) {
    *is_ror_scan = false;
  } else if (param->table->s->primary_key == keynr && pk_is_clustered) {
    /* Clustered PK scan is always a ROR scan ... */
    *is_ror_scan = true;
  }

  return rows;
}
```

⇒ **两个副作用容易被忽略**：① `quick_rows[keynr]` 会在这里被填，第 9 章的 `calculate_condition_filter()` 直接消费它（range 估行优先于 rec_per_key 与 guesstimates）；② 降序 keypart 会**同时**关掉 ROR 与 index merge（`is_ror_scan = false; is_imerge_scan = false;`）。

### 5.2 每个 range 的 rows（`handler::multi_range_read_info_const`，`handler.cc:6260`）

```cpp
if ((range.range_flag & UNIQUE_RANGE) && !(range.range_flag & NULL_RANGE))
  rows = 1;                          // 唯一索引全等值 ⇒ 最多 1 行
else if (range.range_flag & SKIP_RECORDS_IN_RANGE && !(range.range_flag & NULL_RANGE)) {
  if ((range.range_flag & EQ_RANGE) &&
      (keyparts_used = my_count_bits(range.start_key.keypart_map)) &&
      table->key_info[keyno].has_records_per_key(keyparts_used - 1))
    rows = table->key_info[keyno].records_per_key(keyparts_used - 1);   // 统计估算
  else
    rows = 1;                        // FORCE INDEX 下 cost 会被忽略
} else {
  rows = this->records_in_range(keyno, min_endp, max_endp);             // ★ index dive
}
total_rows += rows;
```

**cost 公式**（`handler.cc:6294`）：

```cpp
if (*flags & HA_MRR_INDEX_ONLY)
  *cost = index_scan_cost(keyno, n_ranges, total_rows);   // 覆盖索引，只读索引页
else
  *cost = read_cost(keyno, n_ranges, total_rows);         // 要回表
cost->add_cpu(cost_model->row_evaluate_cost(total_rows) + 0.01);
```

- **IO**：`n_ranges` 参与计算（每个 range 至少一次 B-tree 下降，近似一次随机页访问）
- **CPU**：`row_evaluate_cost(total_rows) + 0.01`（0.01 是每个 range 的启动开销）

#### 5.2.1 `QUICK_RANGE`：区间如何从 SEL_ARG 落到执行期

> **勘误**：8.0.39 里 `QUICK_SELECT_I` / `QUICK_RANGE_SELECT` / `QUICK_INDEX_MERGE_SELECT` 等**全部类已被删除**（`sql/` 下 0 命中）。对应物是 `QUICK_RANGE` 结构 + 各类 `RowIterator`（range 的是 `IndexRangeScanIterator`）。

```cpp
// sql/range_optimizer/range_optimizer.h:69
class QUICK_RANGE {
 public:
  uchar *min_key, *max_key;
  uint16 min_length, max_length;

  /// Stores bitwise-or'ed bits defined in enum key_range_flags.
  uint16 flag;

  /**
    Stores one of the HA_READ_MBR_XXX items in enum ha_rkey_function, only
    effective when flag has a GEOM_FLAG bit.
  */
  enum ha_rkey_function rkey_func_flag;
  key_part_map min_keypart_map,  // bitmap of used keyparts in min_key
      max_keypart_map;           // bitmap of used keyparts in max_key

  QUICK_RANGE(); /* Full range */
  QUICK_RANGE(MEM_ROOT *mem_root, const uchar *min_key_arg, uint min_length_arg,
              key_part_map min_keypart_map_arg, const uchar *max_key_arg,
              uint max_length_arg, key_part_map max_keypart_map_arg,
              uint flag_arg, enum ha_rkey_function rkey_func);

  void make_min_endpoint(key_range *kr) {
    kr->key = (const uchar *)min_key;
    kr->length = min_length;
    kr->keypart_map = min_keypart_map;
    kr->flag = ((flag & NEAR_MIN) ? HA_READ_AFTER_KEY
                                  : (flag & EQ_RANGE) ? HA_READ_KEY_EXACT
                                                      : HA_READ_KEY_OR_NEXT);
  }

  void make_max_endpoint(key_range *kr) {
    kr->key = (const uchar *)max_key;
    kr->length = max_length;
    kr->keypart_map = max_keypart_map;
    /*
      We use READ_AFTER_KEY here because if we are reading on a key
      prefix we want to find all keys with this prefix
    */
    kr->flag = (flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
  }
};

using Quick_ranges = Mem_root_array<QUICK_RANGE *>;
```

```cpp
// sql/range_optimizer/range_optimizer.cc:270
QUICK_RANGE::QUICK_RANGE()
    : min_key(nullptr), max_key(nullptr), min_length(0), max_length(0),
      flag(NO_MIN_RANGE | NO_MAX_RANGE),      // ★ 默认构造 = 全区间 (−inf, +inf)
      rkey_func_flag(HA_READ_INVALID),
      min_keypart_map(0), max_keypart_map(0) {}
```

`flag` 的取值（`include/my_base.h:1079`）——**`eq_range_index_dive_limit` 与 `FORCE INDEX` 都靠它表达**：

```cpp
enum key_range_flags {
  NO_MIN_RANGE = 1 << 0,  ///< from -inf
  NO_MAX_RANGE = 1 << 1,  ///< to +inf
  NEAR_MIN = 1 << 2,      /*  X < key, i.e. not including the left endpoint */
  NEAR_MAX = 1 << 3,      /* X > key, i.e. not including the right endpoint */
  UNIQUE_RANGE = 1 << 4,  /* 唯一索引的全等值区间 */
  EQ_RANGE = 1 << 5,      /* 全等值区间（不一定唯一） */
  NULL_RANGE = 1 << 6,    /* 至少一个 keypart 是 IS NULL */
  GEOM_FLAG = 1 << 7,     /* R-tree，用 HA_READ_MBR_XXX */
  SKIP_RANGE = 1 << 8,    /* Deprecated */
  /*
    Used to indicate that index dives can be skipped. This can happen when:
    a) Query satisfies JOIN_TAB::check_skip_records_in_range_qualification. OR
    b) Used together with EQ_RANGE to indicate that index statistics should be
       used instead of sampling the index.       // ★ eq_range_index_dive_limit
  */
  SKIP_RECORDS_IN_RANGE = 1 << 9,
  /* Keypart is reverse-ordered (DESC) and ranges needs to be scanned backward. */
  DESC_FLAG = 1 << 10,
};
```

#### 5.2.2 `IndexRangeScanIterator`：8.0 的"QUICK_RANGE_SELECT"

```cpp
// sql/range_optimizer/index_range_scan.h:61
class IndexRangeScanIterator : public RowIDCapableRowIterator {
 protected:
  handler *file;
  uint index;                       /* Index this quick select uses */
  bool in_ror_merged_scan;
  MY_BITMAP column_bitmap;

  Bounds_checked_array<QUICK_RANGE *> ranges;  /* ordered array of range ptrs */

  /* Range pointers to be used when not using MRR interface */
  QUICK_RANGE **cur_range;
  QUICK_RANGE *last_range;

  /* Members needed to use the MRR interface */
  QUICK_RANGE_SEQ_CTX qr_traversal_ctx;
  uint mrr_flags;
  uint mrr_buf_size;                /* copy from thd->variables.read_rnd_buff_size */
  HANDLER_BUFFER *mrr_buf_desc;

  KEY_PART_INFO *key_part_info;

  const bool need_rows_in_rowid_order;   // ★ 老说的 need_rows_in_row（真名带 rowid）
  const bool reuse_handler;
  ...
 public:
  bool Init() override;
  int Read() override;
  void UnlockRow() override {
    // Override TableRowIterator::UnlockRow(), since we may use
    // a different handler from m_table->file.
    file->unlock_row();
  }
  uchar *last_rowid() const override {
    assert(need_rows_in_rowid_order);
    return file->ref;
  }
};
```

**`Read()` 的实现**（`index_range_scan.cc:352`）——它本身极薄，真正的活儿全在 MRR：

```cpp
int IndexRangeScanIterator::Read() {
  MY_BITMAP *const save_read_set = table()->read_set;
  MY_BITMAP *const save_write_set = table()->write_set;

  if (in_ror_merged_scan) {
    /*
      We don't need to signal the bitmap change as the bitmap is always the
      same for this table()->file
    */
    table()->column_bitmaps_set_no_signal(&column_bitmap, &column_bitmap);
  }

  char *dummy;
  int result = file->ha_multi_range_read_next(&dummy);
  // ★★ 一次调用推进 MRR：跨 range 的切换、回表排序、buffer 消费都由 handler 完成

  if (in_ror_merged_scan) {
    /* Restore bitmaps set on entry */
    table()->column_bitmaps_set_no_signal(save_read_set, save_write_set);
    if (result == 0) {
      file->position(table()->record[0]);   // ★ ROR 场景必须记 rowid 供上层归并
    }
  }
  if (result == 0) {
    if (m_examined_rows != nullptr) {
      ++*m_examined_rows;
    }
    return 0;
  }
  return HandleError(result);
}
```

**`ranges` 数组怎么喂给 handler**（`index_range_scan.cc:303`，`shared_reset()`）：

```cpp
bool IndexRangeScanIterator::shared_reset() {
  last_range = nullptr;
  cur_range = ranges.begin();

  if (InitIndexRangeScan(table(), file, index, mrr_flags, in_ror_merged_scan,
                         &column_bitmap)) {
    return true;
  }

  /* Allocate buffer if we need one but haven't allocated it yet */
  if (mrr_buf_size && !mrr_buf_desc) {
    unsigned buf_size = mrr_buf_size;
    uchar *mrange_buff = nullptr;
    while (buf_size &&
           !my_multi_malloc(key_memory_IndexRangeScanIterator_mrr_buf_desc,
                            MYF(MY_WME), &mrr_buf_desc, sizeof(*mrr_buf_desc),
                            &mrange_buff, buf_size, NullS)) {
      /* Try to shrink the buffers until both are 0. */
      buf_size /= 2;      // ★ MRR buffer 分配失败就折半重试（可以退到 0 = 不用 MRR）
    }
    ...
  }

  RANGE_SEQ_IF seq_funcs = {quick_range_seq_init, quick_range_seq_next,
                            nullptr};
  if (int error = file->multi_range_read_init(
          &seq_funcs, this, ranges.size(), mrr_flags,
          mrr_buf_desc ? mrr_buf_desc : &empty_buf);
      error != 0) {
    (void)report_handler_error(table(), error);
    return true;
  }
  return false;
}
```

**执行期把 `QUICK_RANGE` 翻译成 `KEY_MULTI_RANGE`**（`index_range_scan.cc:160`）：

```cpp
uint quick_range_seq_next(range_seq_t rseq, KEY_MULTI_RANGE *range) {
  QUICK_RANGE_SEQ_CTX *ctx = reinterpret_cast<QUICK_RANGE_SEQ_CTX *>(rseq);
  if (ctx->cur == ctx->last) return 1; /* no more ranges */

  QUICK_RANGE *cur = *(ctx->cur);
  key_range *start_key = &range->start_key;
  key_range *end_key = &range->end_key;

  start_key->key = cur->min_key;
  start_key->length = cur->min_length;
  start_key->keypart_map = cur->min_keypart_map;
  start_key->flag =
      ((cur->flag & NEAR_MIN)
           ? HA_READ_AFTER_KEY
           : (cur->flag & EQ_RANGE) ? HA_READ_KEY_EXACT : HA_READ_KEY_OR_NEXT);
  end_key->key = cur->max_key;
  end_key->length = cur->max_length;
  end_key->keypart_map = cur->max_keypart_map;
  /*
    We use HA_READ_AFTER_KEY here because if we are reading on a key
    prefix. We want to find all keys with this prefix.
  */
  end_key->flag =
      (cur->flag & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_AFTER_KEY);
  range->range_flag = cur->flag;
  ctx->cur++;
  return 0;
}
```

### 5.3 eq_range_index_dive_limit 的判定（`index_range_scan_plan.cc:1222`）

```cpp
static bool eq_ranges_exceeds_limit(const SEL_ROOT *keypart, uint *count, uint limit) {
  if (limit == 0) return false;      // 特性关闭，永远 dive
  if (limit == 1) return true;
  for (SEL_ARG *r = keypart->root->first(); r; r = r->next) {
    if (!r->min_flag && !r->max_flag &&          // 是等值区间（不是 </>）
        !r->cmp_max_to_min(r) &&                 // 上下界相同（不是 BETWEEN）
        !r->is_null_interval())                  // IS NULL 不计数
    {
      if (r->next_key_part && r->next_key_part->root->part == r->part + 1)
        eq_ranges_exceeds_limit(r->next_key_part, count, limit);   // ★ 沿 keypart 递归
      else
        (*count)++;                  // 到叶子 ⇒ 一条等值路径
      if (*count >= limit) return true;
    }
  }
  return false;
}
```

**算法要点**：
- 沿 `next_key_part` **递归计数**"等值路径"条数。`a IN(1..100) AND b IN(1..100)` 计出 10000 条路径，`a IN(1..100)` 只计 100 条。
- **`x IS NULL` 不计数**：NULL 行数与 `records_per_key` 统计差异可能极大，用统计会严重失真。
- **默认值 `eq_range_index_dive_limit = 200`**（`sys_vars.cc:3925`）；`0` = 永远 index dive。

### 5.4 三条跳过 index dive 的途径

| 途径 | 来源 | 说明 |
|------|------|------|
| `UNIQUE_RANGE` | `handler.cc:6262` | 唯一索引全等值，直接 `rows=1`，连统计都不查 |
| `use_index_statistics` | `eq_ranges_exceeds_limit` | 等值区间太多（≥200），改用 `records_per_key` |
| `skip_records_in_range` | `check_skip_records_in_range_qualification`（`sql_optimizer.cc:6145`） | FORCE INDEX 且只一个候选索引时，代价数字反正不影响选择，dive 纯属浪费 |

### 5.5 核实过的默认参数

| 变量 | 默认值 | 位置 |
|------|--------|------|
| `eq_range_index_dive_limit` | **200**（0 = 永远 dive） | `sys_vars.cc:3925` |
| `range_optimizer_max_mem_size` | **8388608**（8 MB，0 = 不限） | `sys_vars.cc:3339` |
| `range_alloc_block_size` | **4096** | `sys_vars.cc:1935` |
| index_merge 系列开关 | 默认**全开** | `sys_vars.cc:197-200` |
| `test_quick_select` 小表短路 | `table_scan_cost <= 2.0` | `range_optimizer.cc:560` |

### 5.6 quick_keys 缓存与两次重估（"先入为主"的历史坑）

**背景**（阿里月报 2015/11）：range 代价计算存在"先入为主"——多个索引代价相同时只缓存第一个，导致后续不满足"ref/range 同索引"的优化条件而选错索引。

**缓存写入**（`check_quick_select`，`index_range_scan_plan.cc:627`）：

```cpp
rows = file->multi_range_read_info_const(keynr, &seq_if, (void *)&seq, 0, ...);
if (rows != HA_POS_ERROR) {
  param->table->quick_rows[keynr] = rows;
  if (update_tbl_stats) {
    param->table->quick_keys.set_bit(keynr);          // ★ 写入缓存位图
    param->table->quick_key_parts[keynr] = seq.max_key_part + 1;
    param->table->quick_n_ranges[keynr] = seq.range_count;
  }
  param->table->possible_quick_keys.set_bit(keynr);
}
```

配套数组 `quick_rows` / `quick_key_parts` / `quick_n_ranges`（`table.h:1689`）保存每个索引的估算行数、keypart 数、区间数——**这些就是被 `find_best_ref` 复用的缓存**。

**"ref/range 同索引"的判定**（`find_best_ref`，`sql_planner.cc:556`）：

```cpp
// range 用了更多 keypart → 拒绝 ref，改用 range
if (!table_deps && table->quick_keys.is_set(key) &&         // (1) 缓存里有这个索引
    table->quick_key_parts[key] > cur_used_keyparts)        // (2) range 用更多 keypart
{
  trace_access_idx.add("chosen", false).add_alnum("cause", "range_uses_more_keyparts");
  continue;   // 放弃 ref 候选
}
```

反向启发在 `best_access_path`（`sql_planner.cc:1104`）：ref 用 keypart 数 ≥ range 时，拒绝 range（`heuristic_index_cheaper`）。

**8.0.39 的现状**（已核实）：

1. **"只缓存第一个索引"的缺陷已不存在**：`get_key_scans_params`（`index_range_scan_plan.cc:850`）的 `for (idx = 0; idx < param->keys; idx++)` 遍历**所有**可用索引，每个都调 `check_quick_select`（`update_tbl_stats=true`），没有"找到第一个就 break"。所以每个可用索引都会 `quick_keys.set_bit(keynr)`，不会漏缓存。
2. **但"两次 test_quick_select 之间不清缓存"仍存在**：`quick_keys` 只在 `TABLE::reset()`（语句间，`table.cc:4181`）和物化表 `QEP_TAB::cleanup()`（`sql_select.cc:3554`）清空。**同一语句内第一次（`estimate_rowcount`）与第二次重估（`make_join_query_block`）之间不清空**——若两次的 `usable_keys`/条件不同，第一次写过的位可能残留，影响 `sql_optimizer.cc:9948` 的 `quick_keys.is_clear_all()` 判断（决定 `QS_DYNAMIC_RANGE` vs `QS_RANGE`）。

#### 5.6.1 残留位如何改变结论：`QS_DYNAMIC_RANGE` vs `QS_RANGE`（完整源码）

`enum quick_type { QS_NONE, QS_RANGE, QS_DYNAMIC_RANGE };`（`sql_select.h:323`）。下面这段就是"陈旧 `quick_keys`"实际产生影响的地方（`make_join_query_block`，`sql_optimizer.cc:9948`）：

```cpp
// sql/sql_optimizer.cc:9948
if (!tab->table()->quick_keys.is_subset(tab->checked_keys) ||
    !tab->needed_reg.is_subset(tab->checked_keys)) {
  tab->keys().merge(tab->table()->quick_keys);
  tab->keys().merge(tab->needed_reg);

  /*
    The logic below for assigning tab->use_quick is strange.
    It bases the decision of which access method to use
    (dynamic range, range, scan) based on seemingly
    unrelated information like the presence of another index
    with too bad selectivity to be used.

    Consider the following scenario:

    The join optimizer has decided to use join order
    (t1,t2), and 'tab' is currently t2. Further, assume that
    there is a join condition between t1 and t2 using some
    range operator (e.g. "t1.x < t2.y").

    It has been decided that a table scan is best for t2.
    make_join_query_block() then reran the range optimizer a few
    lines up because there is an index 't2.good_idx'
    covering the t2.y column. If 'good_idx' is the only
    index in t2, the decision below will be to use dynamic
    range. However, if t2 also has another index 't2.other'
    which the range access method can be used on but
    selectivity is bad (#rows estimate is high), then table
    scan is chosen instead.

    Thus, the choice of DYNAMIC RANGE vs SCAN depends on the
    presence of an index that has so bad selectivity that it
    will not be used anyway.
  */
  if (!tab->needed_reg.is_clear_all() &&                   // (a) 还有表依赖未解决
      (tab->table()->quick_keys.is_clear_all() ||          // (b) 残留/新写的 quick_keys 为空
       (tab->range_scan() &&
        (tab->range_scan()->num_output_rows() >= 100.0))))  // (c) 或 range 估行 >= 100
  {
    tab->use_quick = QS_DYNAMIC_RANGE;   // ★ 每行外层行重跑 test_quick_select
    tab->set_type(JT_ALL);               // ★ 但计划类型仍是全表扫
  } else
    tab->use_quick = QS_RANGE;           // ★ 用这次重估得到的静态 range 计划
}

if (tab->type() != orig_join_type || tab->range_scan() != orig_range_scan)
  tab->position()->filter_effect = COND_FILTER_STALE;   // ★ 访问方法变了 ⇒ 过滤系数作废重算
}
```

**逐条解读**：

- 判据 (b) 是 `quick_keys` **位图是否为空**，而不是"有没有更好的 range"。所以 `estimate_rowcount` 阶段写进去的位若残留到这一步，会让它认为"有可用 range"从而**不走** `QS_DYNAMIC_RANGE`。
- `QS_DYNAMIC_RANGE` 的语义：`tab->set_type(JT_ALL)` 保留全表扫作为兜底，**执行期**由 `DynamicRangeIterator`（`ref_row_iterators.cc:542`）对每一行外层行重新调 `test_quick_select`。条件 (c) `>= 100.0` 是阈值：range 估行太少（<100）时动态重估不划算。
- 访问方法一变，`filter_effect = COND_FILTER_STALE` ⇒ 第 9 章的 condition filtering 必须重算（range 估行与过滤系数互相依赖，这正是"range 分析要分多阶段"的另一个证据）。

> ★ 源码自己在注释里承认这段逻辑 "**is strange**"：*选择 DYNAMIC RANGE 还是 SCAN，取决于是否存在一个"选择性差到根本不会被用"的索引*。这是"贪心 + 事后补救"架构最直白的自我批评。

**结论**：2015 年的"先入为主只缓存第一个"已修复，但"陈旧缓存"问题（两次重估不清空）是缓存生命周期设计的一部分，属已知特性而非 bug。

---

## 六、长 IN list 字面量的处理

> 本节专题：`WHERE col IN (1,2,3,...,N)`（**字面量列表**，不是 IN 子查询）从 parse 到执行的完整路径，以及大 N 时的性能退化点。这是 range 优化里最常见的性能问题来源。

### 6.1 从 parse 到 fix_fields：`Item_func_in` 的排序与去重

**`Item_func_in` 结构**（`item_cmpfunc.h:2053`）：

```cpp
class Item_func_in final : public Item_func_opt_neg {
  in_vector *m_const_array{nullptr};   // ⚠ 8.0.39 成员名，非 5.x 的 array/create_array()
  bool have_null{false};
  bool m_populated{false};
 private:
  bool m_values_are_const{true};       // 列表项都是 const
  bool m_need_populate{false};         // 每次执行需重新填充
  ...
};
```

`args[0]` 是被测值（LHS），`args[1..arg_count-1]` 是列表项（RHS）。`negated` 标志（继承自 `Item_func_opt_neg`）表示 `NOT IN`。

**排序去重的真相**（`in_vector::fill`，`item_cmpfunc.cc:4232`）：

```cpp
bool in_vector::fill(Item **items, uint item_count) {
  m_used_size = 0;
  for (uint i = 0; i < item_count; i++) {
    set(m_used_size, items[i]);          // 求值并暂存
    if (!items[i]->null_value) m_used_size++;   // 跳过 NULL
  }
  sort_array();                          // std::sort，O(N log N)
  return m_used_size < item_count;       // true = 发现过 NULL
}
```

**三个关键澄清**（容易误解）：

1. **只排序、不显式去重**：`in_vector::fill` 只 `std::sort`，**没有** `std::unique`。重复值保留，去重语义在 range 优化的 `key_or` 合并阶段自然发生。
2. **排序只服务于执行期二分查找**：触发条件（`resolve_type` 末尾，`:5287`）是"全部列表项 `const_for_execution()` 且类型统一且非 JSON"，才建 `m_const_array`。此时 `val_int`（`:5336`）用 `std::binary_search` 做 **O(log N)** 查找；否则退化为线性扫描 **O(N)**。
3. **plain IN 的 range 分析不排序**：`get_mm_tree` 按原始参数顺序逐元素处理（见 6.2），与 `in_vector` 的排序是两回事。

### 6.2 range 分析的 DNF 展开与复杂度

`col IN (c1..cN)` 被显式展开为 `col=c1 OR ... OR col=cN`（见 2.4 节），逐元素 `get_mm_leaf` 造单点 + `tree_or`/`key_or` 插入红黑树。

### 6.3 完整复杂度表（`col IN (1..N)`）

| 步骤 | 位置 | 复杂度 |
|------|------|--------|
| parse 构造 args | `sql_yacc.yy:10302` | O(N) |
| resolve/fix + 类型收集 | `item_cmpfunc.cc:5014` | O(N) |
| 排序（仅执行期二分数组） | `in_vector::fill:4232` | O(N log N) |
| range：逐元素 get_mm_leaf | `range_analysis.cc:365` | O(N) |
| range：逐元素 tree_or→key_or | `tree.cc:1089` + `rb_insert` | **O(N log N)** |
| range：eq_ranges_exceeds_limit 计数 | `index_range_scan_plan.cc:1222` | O(N) |
| 计划：index dive（未超限） | `check_quick_select` | O(N × dive 成本) |
| 计划：统计估算（超限） | `sel_arg_range_seq_next` | O(N)（查 rec_per_key） |

### 6.4 大 N 的退化点：`eq_range_index_dive_limit` 的边界

两个退化点，**`eq_range_index_dive_limit`（默认 200）只缓解第二个**：

1. **SEL_ARG 红黑树构造 O(N log N) + O(N) 内存**：每个列表项产生一个 SEL_ARG 节点（含 key image 的独立分配）。N=10 万就是 10 万个节点——这是**无法被 dive limit 缓解的**（哪怕统计估算 O(1)，你仍然要先把 10 万个节点造出来）。
2. **index dive O(N × 单次 descend)**：每个等值区间一次 `records_in_range`（真 B+ 树 descend）。超过 `eq_range_index_dive_limit` 后改成统计估算，这一步从 O(N × dive) 降到 O(N)。

> **关键结论**：`eq_range_index_dive_limit` 只缓解 index dive，**不缓解 SEL_ARG 树构造**。所以超长 IN list 即使开了统计估算，range 分析本身仍然很慢——这是理解"为什么长 IN 是性能杀手"的核心。

### 6.5 标准 MySQL 8.0.39 没有 inlist2join；10 万项的行为

**标准 MySQL 8.0.39 没有把 IN list 转 join/物化的优化**：全库 `grep inlist2join` 返回 0 匹配（`inlist2join` 是 Percona Server 的扩展特性）。标准 MySQL 对 IN 字面量的全部手段只有：

- range 优化（本路径）
- 执行期二分查找数组（`in_vector`）
- 无 range 可用时退化为全表扫描 + `val_int` 过滤

**10 万项 IN 的实际行为**——不是报错、不是截断，而是**撞内存闸 → 降级为警告 → 放弃 range**：

1. range 优化的 `temp_mem_root` 容量上限被设为 `range_optimizer_max_mem_size`（`range_optimizer.cc:352`）：

```cpp
temp_mem_root->set_max_capacity(thd->variables.range_optimizer_max_mem_size);  // 默认 8MB
temp_mem_root->set_error_for_capacity_exceeded(true);
```

2. 超限后 `EE_CAPACITY_EXCEEDED` 被 `Range_optimizer_error_handler`（`internal.h:85`）**降级为警告** `ER_CAPACITY_EXCEEDED_IN_RANGE_OPTIMIZER`。
3. `param->has_errors()` 为真 → `get_mm_tree`/`get_key_scans_params` 返回空 → **该索引的 range 计划被放弃**，MySQL 回退到全表扫描（或其它索引计划）。

> 补充：**NOT IN 有显式阈值** `NOT_IN_IGNORE_THRESHOLD = 1000`（`range_analysis.cc:218`），超 1000 项直接不构造 SEL_TREE（NOT IN 的区间数接近 2N+1 且很少值得优化）；但 **plain IN 没有**这个计数阈值，只有 `range_optimizer_max_mem_size`（8MB）这一道内存闸。

### 6.6 IN 子查询 vs IN list 字面量

**解析期就分流成两套完全不同的类**（`sql_yacc.yy:10283`）：

```cpp
predicate:
  bit_expr IN_SYM table_subquery
    { $$ = NEW_PTN Item_in_subselect(@$, $1, $3); }   // IN (子查询)
  | bit_expr IN_SYM '(' expr ',' expr_list ')'
    { $$ = NEW_PTN Item_func_in(@$, $6, false); }     // IN (字面量列表)
```

| | `Item_func_in` | `Item_in_subselect` |
|---|---|---|
| 继承 | `Item_func_opt_neg` → `Item_func` | `Item_exists_subselect` → `Item_subselect` |
| `type()` | `FUNC_ITEM` | `SUBSELECT_ITEM` |
| 优化入口 | `select_optimize()` 返回 `OPTIMIZE_KEY`（请 range 优化器处理） | `select_transformer` / 物化 / semi-join（02 篇） |
| 能否转 semi-join/物化 | **否**（无任何 transformer） | 能 |

range 分析入口的硬性区分（`range_analysis.cc:895`）：`cond->type() != FUNC_ITEM` 直接返回空——所以 IN 子查询到不了 range 的 `IN_FUNC` 分支，走的是 02 篇的子查询改写。**IN list 字面量永远不会被转成 semi-join 或物化**。

---

## 七、从 SEL_TREE 到执行

### 7.1 QUICK_RANGE（`range_optimizer/range_optimizer.h:69`）

```cpp
class QUICK_RANGE {
  uchar *min_key, *max_key;
  uint16 min_length, max_length;
  uint16 flag;                     // NEAR_MIN/NEAR_MAX/EQ_RANGE/UNIQUE_RANGE/NULL_RANGE...
  key_part_map min_keypart_map, max_keypart_map;
};
```

**flag → SE API 的映射**（`:121` / `:160`）：

| flag | ha_rkey_function | 语义 |
|---|---|---|
| `NEAR_MIN` | `HA_READ_AFTER_KEY` | `> key` |
| `EQ_RANGE` | `HA_READ_KEY_EXACT` | `= key` |
| 默认 | `HA_READ_KEY_OR_NEXT` | `>= key` |
| `NEAR_MAX` | `HA_READ_BEFORE_KEY` | `< key` |
| 默认 | `HA_READ_AFTER_KEY` | `<= key`（由 end_range 截断） |

### 7.2 SEL_ARG 图 → Quick_ranges（`get_ranges_from_tree_given_base`，`index_range_scan_plan.cc:1060`）

**深度优先展开**，把"图"摊平成不相交的有序区间列表：

1. **沿 `next/prev` 遍历**（枚举 OR）
2. **沿 `next_key_part` 递归**（拼接 AND）——但**只有本层是等值**（`min_flag==0 && max_flag==0` 且 min==max）才能继续往下一层，把 `(a=3, b<1)` 拼成多元组区间 `(3,-inf) <= (a,b) < (3,1)`
3. **非等值层必须停下**：B-tree range scan 只支持**最后一个 keypart 上的不等式**。但代码做了优化：即使丢弃下层谓词，也用下层的 min/max 收紧本层端点（`a >= 3 AND b IN(4,9,10)` ⇒ 从 `(3,-inf)` 提升到 `(3,4)`）
4. `num_exact_key_parts` 记录"从第几个 keypart 开始谓词被丢弃"
5. DESC 索引要 `invert_min_flag/invert_max_flag` 翻转

结果 `ranges` 是**有序不相交**的 `Quick_ranges`。

### 7.3 执行期 IndexRangeScanIterator

```cpp
// sql/range_optimizer/index_range_scan.h:61
class IndexRangeScanIterator : public RowIDCapableRowIterator {
  Bounds_checked_array<QUICK_RANGE *> ranges;   // 有序区间指针数组
  QUICK_RANGE_SEQ_CTX qr_traversal_ctx;         // MRR 遍历上下文
  bool need_rows_in_rowid_order;                // ROR 归并时按 rowid 顺序
};

// Init: index_range_scan.cc:303
RANGE_SEQ_IF seq_funcs = {quick_range_seq_init, quick_range_seq_next, nullptr};
file->multi_range_read_init(&seq_funcs, this, ranges.size(), mrr_flags, mrr_buf);

// Read: index_range_scan.cc:352
int result = file->ha_multi_range_read_next(&dummy);
```

**执行期数据流**：

```
WHERE cond
 → get_mm_tree()          → SEL_TREE（森林：keys[] + merges）
 → get_key_scans_params() → check_quick_select() 算 rows/cost（index dive 在此）
 → get_ranges_from_tree() → Quick_ranges（有序不相交 QUICK_RANGE*）
 → AccessPath{INDEX_RANGE_SCAN}
 → IndexRangeScanIterator{ranges}
 → Init(): multi_range_read_init(&seq_funcs, ...)
 → Read(): ha_multi_range_read_next()  ← quick_range_seq_next 把 QUICK_RANGE 翻成 key_range
```

**动态 range**（`DynamicRangeIterator`，`ref_row_iterators.cc:542`）：每一行外层行重新调 `test_quick_select`，产出新的 AccessPath。这是 `QS_DYNAMIC_RANGE` 的运行形态。

---


## 参考

**内核月报**
- **2021/06《Range (Min-Max Tree) 结构分析》** —— `SEL_TREE` / `SEL_ARG` 区间森林的本源解析
- **2024/09《MySQL Index-Merge 代价估算原理》** —— ROR-intersection 的选择率乘数、贪心搜索、去重率、bad case 根因（4.4/4.5/4.6 节的直接来源）
- **2019/05《Skip Scan Range》** —— Loose Skip Scan 的优化思路与 InnoDB 执行步骤（4.2 节）
- **2015/11《MySQL 优化器 range 的代价计算》** —— "先入为主"缓存坑（5.6 节）

**官方文档**
- *MySQL 8.0 Reference Manual → Range Optimization*
- *MySQL 8.0 Reference Manual → Multi-Range Read Optimization*、`Index Merge Optimization`


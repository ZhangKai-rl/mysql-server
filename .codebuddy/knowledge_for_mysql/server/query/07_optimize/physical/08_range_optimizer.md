# 08 Range 优化：SEL_TREE / SEL_ARG 区间森林与 range 访问方法

> Range 优化是**访问方法选择**里最复杂的一块（属于物理优化③），独立成篇。本篇覆盖：WHERE 如何变成区间森林、森林如何合并、如何变成多种 range 访问方法、代价怎么算、执行期怎么用。

## 目录

- [一、range 优化在优化器中的位置](#一range-优化在优化器中的位置)
- [二、get_mm_tree：从 WHERE 构造 SEL_TREE](#二get_mm_tree从-where-构造-sel_tree)
- [三、SEL_ARG 的三重结构与区间操作](#三sel_arg-的三重结构与区间操作)
- [四、各 range 访问方法的判定与实现](#四各-range-访问方法的判定与实现)
- [五、代价与行数估算（index dive vs 统计）](#五代价与行数估算index-dive-vs-统计)
- [六、长 IN list 字面量的处理](#六长-in-list-字面量的处理)
- [七、从 SEL_TREE 到执行](#七从-sel_tree-到执行)

---

## 一、range 优化在优化器中的位置

### 1.1 入口 test_quick_select

`sql/range_optimizer/range_optimizer.cc:524`。它在两个时机被调用：

| 时机 | 调用点 | 说明 |
|------|--------|------|
| **第一次** | `estimate_rowcount` → `get_quick_record_count`（`sql_optimizer.cc:6218`） | join order 还没定，只能传 `prev_tables = 0`（不能用引用其他表的谓词） |
| **第二次（重估）** | `make_join_query_block`（`sql_optimizer.cc:9882-9905`） | join order 已定，前缀表已知，range 能用上更多谓词（详见 07 篇 E 节） |

> range 优化器有个铁律（源码注释 `range_optimizer.cc:817-819`）：**range 优化器只在 range 优于全表扫描时才会产出 range scan**，所以调用方不需要再和全表扫比。

### 1.2 主流程

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

### 2.1 递归分发

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

`index_range_scan_plan.cc`。对每个可用索引算 range 扫描的 rows 和 cost，取代价最小者。
- 产出 `AccessPath::INDEX_RANGE_SCAN`（`index_range_scan_plan.cc:972`）
- EXPLAIN 显示 `range`

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


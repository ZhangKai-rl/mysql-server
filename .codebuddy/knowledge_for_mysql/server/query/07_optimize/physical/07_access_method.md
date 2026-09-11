# 07 访问方法选择：const 表 / ref / range / scan 的代价决斗

> 物理优化③的核心之一：为每张表选"怎么读"。本篇覆盖 const 表检测、Key_use 数组、ref 访问代价、ref/range/scan 的最终比较、二次 range 重估。

## 目录

- [一、总览](#一总览)
- [二、const table 检测](#二const-table-检测)
- [三、Key_use 数组：ref 访问的基础](#三key_use-数组ref-访问的基础)
- [四、find_best_ref：ref 访问的代价与 fanout](#四find_best_refref-访问的代价与-fanout)
- [五、best_access_path：ref / range / scan 的最终比较](#五best_access_pathref--range--scan-的最终比较)
- [六、二次 range 重估](#六二次-range-重估)
- [七、最终落地与 EXPLAIN type 映射](#七最终落地与-explain-type-映射)

---

## 一、总览

`make_join_plan`（`sql_optimizer.cc:5307`）里访问方法相关的顺序是刻意设计的：

```cpp
update_ref_and_keys(...)          // :5342  建 Key_use 数组（ref 的基础）
extract_const_tables()            // :5360  单行表 → const
extract_func_dependent_tables()   //       函数依赖表 → const
update_sargable_from_const(...)   // const 读出来后，更多谓词变 sargable
estimate_rowcount()               // :5371  每表第一次 range 分析
optimize_keyuse()                 // :5386
Optimize_table_order::choose_table_order()  // :5394  ★ join order + 访问方法搜索
```

**依赖关系**：`update_ref_and_keys` 必须先跑（函数依赖 const 检测要用 Key_use）→ const 表必须先读出（它的值让更多谓词变常量，进而能做 range）→ `estimate_rowcount` 最后（range 分析需要知道哪些表是 const）。

---

## 二、const table 检测

### 2.1 extract_const_tables（`sql_optimizer.cc:5607`）

主循环把每张表分成三态 `enum_const_table_extraction`：

| 态 | 触发条件 | 语义 |
|---|---|---|
| `extract_no_table` | 在嵌套 join 里（`:5622`）/ 在 sj/aj nest 里（`:5627`） | 完全不提取（NULL 补行语义太复杂，8.0 不做） |
| `extract_empty_table` | 是某外连接的唯一内表（`:5633`） | 只处理"确实是 0 行"的情况（内表无匹配也要出 NULL 补行，不能当"值已确定"） |
| `extract_const_table` | 默认 | 最宽松：0/1 行都算 |

`extract_const_table` 的判定式（`:5655-5660`）：

```cpp
if ((table->s->system ||                        // 表定义上就一行
     table->file->stats.records <= 1 ||         // 统计上 0 或 1 行
     all_partitions_pruned_away) &&             // 分区全裁掉
    !tab->dependent &&                          // 不依赖其他表
    (table->file->ha_table_flags() & HA_STATS_RECORDS_IS_EXACT) &&  // 统计必须精确
    !tl->is_fulltext_searched())
  mark_const_table(tab, nullptr);
```

> ⚠️ **核实过的坑**：`HA_STATS_RECORDS_IS_EXACT` 在 InnoDB **只有临时表才申报**（`ha_innodb.cc:6338`）；MyISAM/HEAP/TempTable 都申报。**所以对普通 InnoDB 永久表，这条路径永远不会触发**——InnoDB 上同样场景走 2.2 的唯一键路径，EXPLAIN 显示 `const` 而非 `system`。

### 2.2 extract_func_dependent_tables（`sql_optimizer.cc:5701`）

**不动点循环**（`do { ... } while ((const_table_map & found_ref) && ref_changed)`）：新产生的 const 表可能让别的表也变 const，要反复扫到不再变化。

核心：唯一索引全 keypart 等值（`:5783-5838`）

```cpp
// 扫同一索引的所有 Key_use，构造两个位图
if (!((~found_const_table_map) & keyuse->used_tables))
  const_ref.set_bit(keyuse->keypart);      // 右值只依赖已确认 const 的表
else
  refs |= keyuse->used_tables;             // 还依赖哪些表（"还在等谁"）
eq_part.set_bit(keyuse->keypart);          // 哪些 keypart 有等值谓词

// 判定
if (eq_part.is_prefix(user_defined_key_parts) &&   // 等值谓词覆盖索引前缀（最左前缀）
    !fulltext && !outer_join_nest && !sj_aj_nest &&
    !(join_cond && join_cond->is_expensive()) &&
    !(ha_table_flags() & HA_BLOCK_CONST_TABLE) && table->is_created()) {
  if (keyinfo->flags & HA_NOSAME) {          // 唯一索引
    if (const_ref == eq_part) {              // 所有 keypart 右值都是常量
      mark_const_table(tab, start_keyuse);
      create_ref_for_key(...);
      join_read_const_table(tab, ...);       // ★ 立即读
      break;
    } else
      found_ref |= refs;                     // 记下"我等这些表变 const"
  } else if (const_ref == eq_part)
    tab->const_keys.set_bit(key);            // 非唯一 ⇒ 不能 const，但可做 range
}
```

**算法要点**：
- `eq_part.is_prefix(...)` 体现**最左前缀原则**：等值谓词必须覆盖索引的连续前缀
- `const_ref == eq_part`：全 keypart 右值都是常量 ⇒ 唯一索引最多一行 ⇒ const
- **连锁传播**：右值还依赖别的表时记进 `found_ref`，等那些表变 const 后下一轮再来（这就是不动点循环的意义）

### 2.3 mark_const_table（`sql_optimizer.cc:8496`）

```cpp
position->rows_fetched = 1.0;
position->prefix_rowcount = 1.0;
position->read_cost = 0.0;
position->filter_effect = 1.0;
// 把 tab 轮换到 best_ref[const_tables] 位置
tab->set_type(key ? JT_CONST : JT_SYSTEM);
const_table_map |= tab->table_ref->map();
const_tables++;
```

**const 表在代价模型里是免费的**：读一次、产出恰好一行、无过滤损失。它们占据 `positions[]` 最前面的槽位，**不参与 join order 搜索**。

### 2.4 join_read_const_table（`sql/sql_executor.cc:3484`）

**优化期就真的把数据读出来**：

- `JT_SYSTEM` → `read_system()`（`:3583`）：`ha_rnd_init` + `ha_rnd_next`，全表扫第一行
- `JT_CONST` → `read_const()`（`:3615`）：`ha_index_init(key)` + `ha_index_read_map(HA_READ_KEY_EXACT)`；若 ref 索引是覆盖索引则开 `keyread`

读完之后（`:3553-3568`）**刷新等值传播**：

```cpp
if (join->where_cond && update_const_equal_items(thd, join->where_cond, tab))
  return 1;
// 再遍历所有 leaf table 的 ON 条件
```

`update_const_equal_items` 把 `Item_equal` 里属于该表的字段替换成**刚读出来的实际常量值**。这就是"等值传播/常量折叠的前提"：`t1` 变 const 并读出值后，`t1.a` 才真正可折叠成字面量 `5`，进而让 `t2.a > t1.a` 变成 `t2.a > 5`（sargable 谓词）。

### 2.5 为什么 const 表必须在 join order 之前处理

1. **搜索空间坍塌**：一张表变 const ⇒ join order 搜索空间从 n! 降到 (n-1)!
2. **代价估计的基数**：非 const 表的 range 分析需要右值已确定
3. **Key_use 的可用性**：`find_best_ref` 用 `join->const_table_map` 判断 keyuse 能否当常量求值
4. **连锁传播需要不动点**：必须依赖"已读出的值"
5. **执行期不再访问**：`table->const_table = true` + `optimized_away = true`，执行期 ConstIterator 只返回 `record[0]`

---

## 三、Key_use 数组：ref 访问的基础

### 3.1 update_ref_and_keys（`sql_optimizer.cc:8279`）

把 WHERE/JOIN 的等值谓词转成 `Key_use` 数组。三个条件来源：

| 来源 | `usable_tables` | 说明 |
|---|---|---|
| WHERE（`:8323`） | `~outer_join` | **排除外连接内表**（会破坏 NULL 补行语义） |
| 每张表的 ON（`:8347-8362`） | 该表自己的 map | ON 条件只能为该表产生 keyuse |
| 嵌套 join 的 ON（`:8365`） | — | `add_key_fields_for_nj` |
| 派生表（`:8372`） | — | `generate_derived_keys()` 给物化派生表**动态建索引**并生成 keyuse |

**笛卡尔展开**（`add_key_part`，`:7768`）：一个谓词 `t.a = X` 里字段 `t.a` 可能出现在**多个索引的多个 keypart** 上，代码遍历所有索引 × 所有 keypart，凡匹配就生成一条 `Key_use`。

### 3.2 排序与去重（`:8394-8438`）

排序规则 `sort_keyuse`（`:7917`）四级：**(tableno, key, keypart, 常量优先)**。这是 `find_best_ref` 里"连续扫描同一索引"写法能成立的唯一原因。

两条去重规则：
- **`:8415` `prev->keypart + 1 < use->keypart`**：剔除"没有前序 keypart"的（如索引 `(a,b,c)` 但只有 `b<5`）——没有最左前缀定位不了
- **`:8416` 同 keypart 且已有常量**：`a=3 AND b=7 AND b=t2.d` 时，`b=t2.d`（ref）被 `b=7`（const）淘汰——**常量永远比 ref 强**

### 3.3 等值传播如何参与（多等值 a=b=c）

`add_key_fields` 的 `OPTIMIZE_EQUAL` case（`:7722`）：

```cpp
Item_equal *item_equal = (Item_equal *)cond;
Item *const_item = item_equal->const_arg();
if (const_item) {
  // 有常量：为每个字段都生成 field = const 的 Key_use（O(n)）
  for (Item_field &item : item_equal->get_fields())
    add_key_field(..., &item, true, &const_item, 1, ...);
} else {
  // 无常量：O(n²) 全排列，为每一对 (outer, inner) 生成 outer = inner
  for (Item_field &outer : item_equal->get_fields())
    for (Item_field &inner : item_equal->get_fields())
      if (!outer.field->eq(inner.field))
        add_key_field(..., &outer, true, inner_ptr, 1, ...);
}
```

**为什么必须双向全生成**：`Key_use` 是**有向弧**（"用 inner 的值去查 outer 的索引"）。join order 还没定时不知道谁先谁后，所以两个方向都要准备好。`find_best_ref` 里 `(remaining_tables & keyuse->used_tables) continue` 会在实际规划时滤掉方向错误的弧。

---

## 四、find_best_ref：ref 访问的代价与 fanout

`sql/sql_planner.cc:207`。

### 4.1 索引等级枚举（`:228`）

```cpp
enum idx_type { CLUSTERED_PK, UNIQUE, NOT_UNIQUE, FULLTEXT };
```

注释明确说 *"code below relies on this element definition order"*——**数值越小等级越高**。`best_found_keytype` 初值是 `NOT_UNIQUE`。

### 4.2 双层循环与 4 条可用性过滤（`:284-366`）

外层按 keypart，内层按"该 keypart 的候选 keyuse"（同一个 keypart 可能有多条 keyuse，要选最好的）。

| 过滤 | 代码 | 含义 |
|---|---|---|
| ① `excluded_tables` | `:318` | semijoin 跨界引用等 |
| ② `remaining_tables & used_tables` | `:319` | **join order 约束落地点**：值来自还没进前缀的表，本轮不能用 |
| ③ 两个 REF_OR_NULL keypart | `:320-321` | `col1=x OR col1 IS NULL` 只能有一个 |
| ④ 常量 NULL 且引擎需全扫 | `:328-333` | `HA_TABLE_SCAN_ON_NULL`（如 NDB hash 索引） |

选最优 keyuse 用 `prev_record_reads(join, idx, ...)`（`:345-355`）：估算"前缀表能产出多少种不同的 key 组合"，取最小。直觉：若 `t1.a` 只有 3 个不同值，对 `t2` 做 3 次索引查找就够了，不必 `prefix_rowcount` 次。

### 4.3 keytype 判定与 EQ_REF（`:395-434`）

```cpp
const bool all_key_parts_covered =
    (found_part == LOWER_BITS(key_part_map, actual_key_parts(keyinfo)));
const bool all_key_parts_non_null =
    (ref_or_null_part == 0 && null_rejecting_part == LOWER_BITS(...));

if (all_key_parts_covered && (keyinfo->flags & HA_NOSAME)) {
  if (key == primary_key && primary_key_is_clustered())
    cur_keytype = CLUSTERED_PK;                      // 聚集主键
  else if ((keyinfo->flags & HA_NULL_PART_KEY) == 0)
    cur_keytype = UNIQUE;                            // 唯一且无可空 keypart
  else if (all_key_parts_non_null)
    cur_keytype = UNIQUE;                            // 唯一但可空，且谓词拒绝 NULL
}

if (all_key_parts_covered && !ref_or_null_part) {
  cur_used_keyparts = (uint)~0;
  if (keyinfo->flags & HA_NOSAME && ((flags & HA_NULL_PART_KEY) == 0 || all_key_parts_non_null)) {
    cur_read_cost = prev_record_reads(join, idx, table_deps) * page_read_cost(1.0);
    cur_fanout = 1.0;                                // ★ EQ_REF：最多 1 行
  }
  ...
}
```

**EQ_REF 的代价**：`prev_record_reads(...) × page_read_cost(1.0)`（不是 `prefix_rowcount`！），fanout 硬编码 1.0。

### 4.4 fanout 的三级来源

| 级别 | 来源 | 代码 |
|---|---|---|
| ① range 重估（最优先） | `table->quick_rows[key]` | `:454-455`、`486-493`、`550-555`、`650-657` |
| ② `records_per_key` 统计 | `keyinfo->records_per_key(n-1)` | `:462-464`、`558-559` |
| ③ 无统计启发式 | 1% / 10% 线性插值 | `:605-630` |
| 特例 EQ_REF / FULLTEXT | 硬编码 1.0 | `:434` / `:684` |

**第三级的完整公式**（`:605-630`，已核实常数）：

```cpp
rec_per_key = has_records_per_key(user_defined_key_parts-1)
            ? records_per_key(user_defined_key_parts-1)
            : records / distinct_keys_est + 1;      // 默认 distinct_keys_est = records/10 ⇒ ≈11

if (rec_per_key / records >= 0.01)
  tmp_fanout = rec_per_key;                          // 区分度已够低，直接用
else {
  const double a = records * 0.01;                   // 第一个 keypart 假设匹配 1%
  if (user_defined_key_parts > 1)
    tmp_fanout = (cur_used_keyparts * (rec_per_key - a) +
                  a * user_defined_key_parts - rec_per_key) / (user_defined_key_parts - 1);
  else
    tmp_fanout = a;                                  // 单 keypart 索引 ⇒ records * 1%
  tmp_fanout = max(tmp_fanout, 1.0);
}
if (ref_or_null_part) cur_fanout *= 2.0;             // 需要两次查找（值 + NULL）
```

这就是文档里常说的**"首列 1%、整 key 10%"启发式**的来源（`distinct_keys_est = records / MATCHING_ROWS_IN_OTHER_TABLE`，`MATCHING_ROWS_IN_OTHER_TABLE = 10`，`sql_planner.cc:91`）。

### 4.5 索引选择规则：`高等级无条件胜出`（`:709-724`）

```cpp
if (best_found_keytype >= NOT_UNIQUE && cur_keytype >= NOT_UNIQUE)
  new_candidate = cur_ref_cost < best_ref_cost;      // ① 都是低等级 ⇒ 比代价
else if (best_found_keytype == cur_keytype)
  new_candidate = cur_ref_cost < best_ref_cost;      // ② 同等级 ⇒ 比代价
else if (best_found_keytype > cur_keytype)
  new_candidate = true;                              // ③ 当前等级更高 ⇒ 无条件胜出！
```

**为什么高等级不看代价**：`CLUSTERED_PK` 索引即数据（无回表），`UNIQUE` fanout=1。代价模型对"回表"的建模随引擎/缓冲池状态波动很大，**用确定等级序取代不确定代价比较，稳定性更高**——这是 MySQL 长期演进的工程决策。

另外 `:728-731`：一旦找到 `CLUSTERED_PK` 立即 `break`（搜索剪枝）。

---

## 五、best_access_path：ref / range / scan 的最终比较

`sql/sql_planner.cc:981`。

### 5.1 四条启发式短路（`:1087-1133`）

| # | 条件 | trace cause | 语义 |
|---|---|---|---|
| **1** | `rows_fetched < found_records && best_read_cost <= read_time` | `cost` | **ref 必胜**：行数更少且代价不高于扫描 |
| **2** | range 与 ref 同一索引且 ref 用的 keypart 更多 | `heuristic_index_cheaper` | 同索引上 ref 定位更精确，ref 不差于 range |
| **3** | `HA_TABLE_SCAN_ON_INDEX`（InnoDB）且有覆盖索引且有 ref | `covering_index_better_than_full_scan` | 表扫本身就是扫聚集索引，覆盖索引避免回表 |
| **4** | `force_index && best_ref && !range_scan` | `force_index` | FORCE INDEX 要求走索引，排除全表扫 |

**关于短路 1 的"不公平比较"**（注释 `:1054-1072`）：`best_read_cost` 是 ref 执行 `prefix_rowcount` 次的总代价，而 `read_time` 是**单次**扫描代价。若用了 join buffer，扫描不会执行 `prefix_rowcount` 次，而是 `prefix_rowcount / 每缓冲行数` 次——比例未知（1 到 `prefix_rowcount` 之间）。为**不低估**扫描代价，启发式假设最坏情况（比例=1）。更精细的比较留给 `calculate_scan_cost`。

### 5.2 calculate_scan_cost（`:770`）

`rows_after_filtering` 三种来源：
1. **条件过滤开启**（默认 `condition_fanout_filter=on`）：`calculate_condition_filter` 用**常量条件**算过滤系数
2. 否则用 range 估的 `quick_condition_rows`
3. 否则若 `found_condition`：`found_records * 0.75`（**25% 启发式**——让 join order 倾向把有条件的表排前面）

**代价公式**：

```cpp
// range 扫描（:833-836）—— range 优化器保证 range 已优于全表扫，不再比
scan_and_filter_cost = prefix_rowcount *
    (range_scan->cost + row_evaluate_cost(found_records - rows_after_filtering));

// 全表/索引扫描，无 join buffer（:863-866）
scan_and_filter_cost = prefix_rowcount *
    (single_scan_read_cost + row_evaluate_cost(records - rows_after_filtering));

// 全表/索引扫描，有 join buffer（BNL，:886-893）
buffer_count = 1.0 + (cache_record_length * prefix_rowcount) / join_buff_size;
scan_and_filter_cost = buffer_count *
    (single_scan_read_cost + row_evaluate_cost(records - rows_after_filtering));
```

**`buffer_count` 的语义**：IO 上表只在"join buffer 填满一次"时扫一遍，填满几次就扫几遍。注意用的是 `1.0 +`（不是 `ceil`），注释说明用 floor 更精确但会多花 5% 的计划搜索时间。

> 已知缺陷（注释 `:828-831`）：**range/index_merge 分支不考虑 join buffer**，只有 ALL/index 考虑。

### 5.3 最终 PK 与写回 POSITION（`:1159-1224`）

```cpp
const double scan_total_cost = scan_read_cost +
    cost_model->row_evaluate_cost(prefix_rowcount * rows_after_filtering);

if (best_ref == nullptr ||
    (scan_total_cost < best_read_cost +
        cost_model->row_evaluate_cost(prefix_rowcount * rows_fetched))) {
  best_ref = nullptr;                 // ★ POSITION::key == nullptr 就是"用扫描"的信号
  rows_fetched = rows_after_filtering;
  best_uses_jbuf = !disable_jbuf;
}
...
pos->key = best_ref;                  // 写回，供 join order 搜索消费
pos->rows_fetched = rows_fetched;
pos->read_cost = best_read_cost;
```

两边都是"引擎 IO + 全部 CPU"，公平比较。`pos->key == nullptr` ⇒ 后续 `init_ref_access` 不建 ref，type 保持 `JT_ALL`/`JT_INDEX_SCAN`/`JT_RANGE`。

**外连接空表修正**（`:1202-1204`）：`rows_fetched == 0` 且是外连接内表 → 钳到 1.0（0 会让后续所有表的 fanout 乘法都得 0，join order 无法区分访问方法）。

---

## 六、二次 range 重估

join order 定完后（`make_join_query_block`），第 i 张表的**前驱表集合已确定**，所以：
- 原来"非 sargable"的 `t2.a > t1.a` 现在**变成 sargable**（t1 在前缀里了）
- 表条件被 `make_cond_for_table` 裁剪后更精确
- 有 LIMIT 时可能值得为"提前终止"重选索引

### 6.1 recheck_reason 判定（`:9764-9788`）

```cpp
if (cond &&                                 // (1a) 有条件
    (tab->keys() != tab->const_keys) &&     // (1b) ★ 有"非纯常量"的条件可用索引
    (i > 0 || (子查询 && cond->is_outer_reference())))   // (1c) 有前驱表
  recheck_reason = NOT_FIRST_TABLE;
else if (!tab->const_keys.is_clear_all() && // (2a) 有纯常量条件可用索引
         i == join->const_tables &&         // (2b) 是第一张非 const 表
         (select_limit_cnt < rows_fetched * filter_effect) &&  // (2c) LIMIT < 预计产出
         !join->calc_found_rows)            // (2d) 非 SQL_CALC_FOUND_ROWS
  recheck_reason = LOW_LIMIT;

if (HA_NO_INDEX_ACCESS) recheck_reason = DONT_RECHECK;        // 引擎不支持索引
if (sj_strategy == SJ_OPT_LOOSE_SCAN) recheck_reason = DONT_RECHECK;  // 已锁定索引
```

**(1b) 是核心判据**：`tab->keys()`（所有可能用到的索引）≠ `tab->const_keys`（只依赖常量的条件能用的索引）⇒ 存在依赖其他表的条件可用索引。第一次 range 分析时（`get_quick_record_count`）只能用 `const_keys`，**现在 join order 定了，非 const 部分也能用了**。

### 6.2 调用与回写（`:9882-9946`）

```cpp
if (tab->range_scan()) { destroy(tab->range_scan()); tab->set_type(JT_ALL); }
search_if_impossible = test_quick_select(
    thd, thd->mem_root, &temp_mem_root,
    usable_keys,                                   // ★ 用 tab->keys()（比第一次的 const_keys 范围大）
    used_tables & ~tab->table_ref->map(),          // ★ prev_tables：已在前缀中的表
    0,
    calc_found_rows ? HA_POS_ERROR : select_limit_cnt,
    false, interesting_order, tab->table(),
    tab->skip_records_in_range(), tab->condition(), ...);
tab->set_range_scan(range_scan);
```

**Impossible WHERE vs Impossible ON**（`:9911-9934`）：第一次返回 impossible 时，**必须再跑一次去掉 ON 条件**：
- 去掉 ON 后**仍** impossible ⇒ 真正的 `Impossible WHERE`（整个查询空）
- 去掉后**不** impossible ⇒ 只是 `Impossible ON`（外连接 ON 恒假，走 NULL 补行，查询继续）

`QS_DYNAMIC_RANGE` 判定（`:9981-9988`）：有 `needed_reg` 且（range 没用上索引 **或** range 估的行数 ≥ 100）⇒ 留着 range 计划但**执行期动态决定**是否使用。

---

## 七、最终落地与 EXPLAIN type 映射

### 7.1 JT_* → AccessPath::Type（`QEP_TAB::access_path()`，`sql_executor.cc:3701`）

| JT_* | AccessPath::Type | 迭代器 |
|---|---|---|
| `JT_REF` | `REF` | `RefIterator` |
| `JT_REF_OR_NULL` | `REF_OR_NULL` | `RefOrNullIterator`（两次查找：值 + NULL） |
| `JT_EQ_REF` | `EQ_REF` | `EQRefIterator`（找到就停） |
| `JT_CONST` / `JT_SYSTEM` | `CONST_TABLE` | `ConstIterator`（**直接返回优化期读好的 record[0]**） |
| `JT_INDEX_SCAN` | `INDEX_SCAN` | `IndexScanIterator` |
| `JT_ALL` / `JT_RANGE` / `JT_INDEX_MERGE` | 走 `create_table_access_path`（`:4690`） | TABLE_SCAN / 各 range 类型 |

`create_table_access_path` 只有三条路：有 range_scan 就复用它、是 CTE 递归引用就 `FOLLOW_TAIL`、否则 `TABLE_SCAN`。

### 7.2 EXPLAIN type 映射（`opt_explain.cc:115`）

```cpp
const char *join_type_str[] = {
    "UNKNOWN", "system", "const",    "eq_ref",      "ref",        "ALL",
    "range",   "index",  "fulltext", "ref_or_null", "index_merge"};
```

| join_type | EXPLAIN | 触发条件 |
|---|---|---|
| `JT_SYSTEM` | `system` | `mark_const_table` 传 `key == nullptr`（表本身就 ≤1 行） |
| `JT_CONST` | `const` | `mark_const_table` 传 `key != nullptr`（唯一键定位） |
| `JT_EQ_REF` | `eq_ref` | `create_ref_for_key` `sql_select.cc:2534` |
| `JT_REF` | `ref` | `sql_select.cc:2520` |
| `JT_RANGE` | `range` | `calc_join_type`（`sql_select.cc:5435`） |
| `JT_INDEX_MERGE` | `index_merge` | `calc_join_type`（`:5439`） |
| `JT_ALL` | `ALL` | 未选到索引，或 dynamic range |
| `JT_INDEX_SCAN` | `index` | `make_join_readinfo` |

**`create_ref_for_key` 的最终 type 判定**（`sql/sql_select.cc:2512-2536`）：

```cpp
if ((actual_key_flags(keyinfo) & HA_NOSAME) == 0 ||            // 非唯一
    ((actual_key_flags(keyinfo) & HA_NULL_PART_KEY) && !null_rejecting_key) ||  // 可空且谓词不拒 NULL
    keyparts != actual_key_parts(keyinfo))                     // 没用满 keypart
  j->set_type(null_ref_key ? JT_REF_OR_NULL : JT_REF);         // ⇒ REF
else if (keyuse_uses_no_tables && !(ha_table_flags() & HA_BLOCK_CONST_TABLE))
  j->set_type(JT_CONST);                                       // ⇒ CONST
else
  j->set_type(JT_EQ_REF);                                      // ⇒ EQ_REF
```

**注意第二条**：唯一索引但**含可空 keypart 且谓词不拒绝 NULL** ⇒ 退化成 REF——因为 MySQL 唯一索引允许多个 NULL，一个 key 值可能匹配多行。

---


## 参考

**论文**
- **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979)** —— 访问路径（access path）选择与代价比较

**官方文档**
- *MySQL 8.0 Reference Manual → How MySQL Chooses Indexes*、`Index Hints`


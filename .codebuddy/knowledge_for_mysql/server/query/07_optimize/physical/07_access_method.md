# 07 访问方法选择：const 表 / ref / range / scan 的代价决斗

> 物理优化③的核心之一：为每张表选"怎么读"。本篇覆盖 const 表检测、Key_use 数组、ref 访问代价、ref/range/scan 的最终比较、二次 range 重估。

## 目录

- [一、总览](#一总览)
- [二、设计思想与理论基础](#二设计思想与理论基础)
- [三、const table 检测](#三const-table-检测)
- [四、Key_use 数组：ref 访问的基础](#四key_use-数组ref-访问的基础)
- [五、find_best_ref：ref 访问的代价与 fanout](#五find_best_refref-访问的代价与-fanout)
  - [4.4 use_index_extensions：唯一性判定的两个口径](#44-use_index_extensions唯一性判定的两个口径)
- [六、best_access_path：ref / range / scan 的最终比较](#六best_access_pathref--range--scan-的最终比较)
- [七、二次 range 重估](#七二次-range-重估)
- [八、Index_lookup：ref 访问的键值绑定](#八index_lookupref-访问的键值绑定)
  - [8.3 store_key 与常量固化](#83-键值绑定store_key-与常量固化ref-为什么快)
  - [8.4 cond_guards：NOT IN 与退化成全表扫描](#84--cond_guardsnot-in-与退化成全表扫描)
  - [8.6 keypart_hash](#86-keypart_hash8-0-新增)
  - [8.7 disable_cache 与 ICP](#87-disable_cache-与-icp一个反直觉现象)
- [九、最终落地与 EXPLAIN type 映射](#九最终落地与-explain-type-映射)

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

## 二、设计思想与理论基础

### 反常识：EXPLAIN 的 11 个 type 不是优化器的内部分类

8.0.39 的 `enum join_type` 确实有 11 个（`system` / `const` / `eq_ref` / `ref` / `ALL` / `range` / `index` / `fulltext` / `ref_or_null` / `index_merge`）。但优化器内部真正使用的分类只有 **4 级**：

```cpp
// Index type, note that code below relies on this element definition order
enum idx_type { CLUSTERED_PK, UNIQUE, NOT_UNIQUE, FULLTEXT };
```

而且选择规则里等级是**硬序**——高等级无条件胜出：

```cpp
if (best_found_keytype >= NOT_UNIQUE && cur_keytype >= NOT_UNIQUE)
  new_candidate = cur_ref_cost < best_ref_cost;      // 同级才比代价
else if (best_found_keytype == cur_keytype)
  new_candidate = cur_ref_cost < best_ref_cost;
else if (best_found_keytype > cur_keytype)
  new_candidate = true;                              // 高等级无条件胜出
```

**结论**：EXPLAIN 的 11 类是给**用户看的"执行器形态"标签**；优化器内部的"分类"是 {聚集主键 / 唯一 / 非唯一 / 全文} 4 类 + 代价比较。这解释了为什么这套 type 看起来不正交——**它不是正交分解，而是"枚举最佳执行器形态"**。

### 真正的正交维度只有两个

| 维度 | 取值 | 对应 type |
|---|---|---|
| **① 定位基数**（一次探测产出几行） | 0/1 行 | `const` / `system` / `eq_ref` |
| | n 行 | `ref` / `ref_or_null` |
| **② 索引取值形态** | 等值 | `const`/`eq_ref`/`ref`/`ref_or_null`/`fulltext` |
| | 多点 / 区间 | `range`、`index_merge` |
| | 全量 | `ALL`、`index` |
| | 无索引可用 | `ALL` |

- "是否走索引"**不是**独立维度（与 ①② 高度相关）
- **输出是否有序 / 是否需要回表**才是 `index` 与 `ALL` 分家的真正理由（见下）
- `fulltext` / `index_merge` / `ref_or_null` 是**历史补丁**，不属于分类学

### 为什么 `index`（全索引扫描）要单列？

`index` 不是"比 ALL 差一点点的扫描"，而是"**换了一个更瘦的物理对象扫描**" + "顺带白送有序性"。三处证据：

**① ALL→index 的升级是非代价的硬性启发式**（这是 `type=index` 最常见的来源）：

```cpp
/**
 An utility function - apply heuristics and optimize access methods to tables.
 Currently this function can change REF to RANGE and ALL to INDEX scan if
 latter is considered to be better (not cost-based) than the former.
*/
void JOIN::adjust_access_methods() {
```

注意函数头注释里的 **"not cost-based"**——设计者把 `index` 当成"确定优于 ALL"，而不是"待比的候选"。

**② 代价公式上"索引条目比整行瘦"**：`index_scan_cost` 按 `key_length + ref_length` 算每页条目数，`table_scan_cost` 按整个 `data_file_length` 算。

**③ InnoDB 的 ALL 本来就是在扫聚集索引**：

```cpp
/*
  Used to avoid scanning full tables on an index. If this flag is set then
  the handler always has a primary key (hidden if not defined) and this
  index is used for scanning rather than a full table scan in all
  situations. No separate data/index file.
*/
#define HA_TABLE_SCAN_ON_INDEX (1 << 2)
```

即：**ALL 扫的是聚集索引（含所有列），换一个更窄的覆盖二级索引一定更省 IO**。这正是"覆盖索引时不回表"在代价模型里的落点。

### const / system 为什么必须在 join order 之前提出来

**收益**（`extract_const_tables()` 注释明说）：

```
Tables that are extracted have their rows read before actual execution
starts ... Thus, they do not take part in join order optimization process,
which can significantly reduce the optimization time.
```

第二重收益容易忽略：**优化期就能剪掉整个查询**——`zero_result_cause = "Impossible WHERE noticed after reading const tables"`。读 const 表这个动作本身就承担了一部分谓词求解，能比任何代价估算更早判空。

**代价**（这部分常被忽略）：

1. **优化期真实 IO**——`join_read_const_table()` 在 optimize 阶段就真去读行。优化本身有磁盘/缓冲池副作用
2. **值被冻结在优化期**——常量传播把 `t1.a` 替换成读出来的字面量，此后执行期不再重新求值
3. **适用面被三条限制卡死**（注释原文）：依赖其他表 / 无精确统计 / 全文检索
4. 引擎可拒绝 const 化：`HA_BLOCK_CONST_TABLE`
5. **外连接的唯一内表只能"确认 0 行"才算 const**——NULL 补行语义下"1 行"并不唯一确定

### eq_ref 为什么必须单列：fanout ≡ 1 是能被"白嫖"的强性质

- **代价估算上**：EQ_REF 的 cost 退化成 `prev_record_reads × page_read_cost(1.0)`，**完全不看 `rec_per_key`**——因为不需要估计"返回多少行"
- **NULL 破坏唯一性**：唯一索引允许同一 key 值出现多个 NULL，因此 `HA_NULL_PART_KEY && !null_rejecting_key` ⇒ 退化为 REF。**唯一性带来的"fanout ≤ 1"必须以"谓词拒绝 NULL"为前提**
- **函数依赖判定**：eq_ref 意味着"这张表被前驱表唯一决定"，所以它的 rowid **不必进** DuplicateWeedout 的临时表（`sj_table_is_included()` 直接 `return false`）。这是只有 fanout=1 才能做的推导

### 覆盖索引的价值：唯一代码落点

`find_cost_for_ref()` 的三分支是"回表 vs 不回表"的分水岭：

```cpp
if (table->covering_keys.is_set(keyno)) {
  // We can use only index tree
  return index_scan_cost(keyno, 1, num_rows).total_cost();
}
if (keyno == table->s->primary_key && table->file->primary_key_is_clustered()) {
  return read_cost(keyno, 1, num_rows).total_cost();   // 索引即数据
}
return min(table->file->page_read_cost(keyno, num_rows), worst_seeks);  // 回表
```

### index_merge：动机与"为什么常常更慢"

**ROR（Rowid-Ordered Retrieval）** 要求扫出来的 rowid **有序**。对 InnoDB 二级索引，**只有"前缀全部等值"时 rowid 才按主键有序**；一旦是范围扫描，输出按 (索引列, 主键) 有序而 rowid 无序。

⇒ **index_merge intersection 在 InnoDB 上几乎只能用于多个等值条件**——这是它适用面窄、常常让人"感觉没用上"的根因。

代价模型三段解释了"为什么反而更慢"：① 要扫 N 个索引（N 份索引 IO）② 回表是**随机 sweep read**（攒 rowid 再 `ha_rnd_pos`，而非单索引 range 的局部有序回表）③ `Unique` 的排序去重成本。

**它的定位从来不是"与 range 平起平坐的候选"**，源码注释说死了：

```
* index_merge will never be used if range scan is possible (even if
  range scan is more expensive)
```

### index dive vs 统计估算的取舍

| | dive | 统计 |
|---|---|---|
| 做法 | 每个 range 调 `records_in_range()`（InnoDB 真下潜 B-tree 采样若干页） | 用 `records_per_key(n-1)` = records / n_diff |
| 精度 | 高 | 假设数据均匀分布 |
| 开销 | **O(range 数) 次引擎调用** | O(1) |

`eq_range_index_dive_limit` 默认 **200**，超过就切统计。两条细节：

- **对 NULL 一律不用统计**（"x IS NULL" 的行数 "likely to be very different"）
- 唯一索引等值直接 `rows = 1`，**连统计都不用**

第三条取舍：range 优化有**内存硬预算** `range_optimizer_max_mem_size`（默认 8 MiB），超了就静默退回 table scan。与"dive vs 统计"是同一类取舍——**用可控的资源上限换优化本身的确定性开销**。

### 与 System R 的对照：一个反直觉的结论

System R 是"**先独立为每张表选出最优 access path，再在此之上做 join order 搜索**"。

MySQL 旧优化器相反：**把 access path 的选择嵌进 join order 搜索的每一次扩展里**——`find_best_ref()` 的入参就带着 join 上下文（`prefix_rowcount`），返回的代价前缀乘的也是它。

**后果**：MySQL 的"最优访问方法"是**相对于某个 join 前缀**的局部最优，不是全局最优。所以 join order 定完之后必须"二次 range 重估"（见第六章）——前驱表确定后，原来非 sargable 的谓词变 sargable 了。

**反直觉的是**：hypergraph 优化器反而**更接近 System R**——它在 `FoundSingleNode()` 里为每个节点独立 propose 全部候选（table scan / index scan / ref / 参数化 ref / range / fulltext / index merge），再由 DPhyp 在子图上做连接。

### `unique_subquery` / `index_subquery` 为什么还在？

**它们根本不是 `join_type` 枚举成员。** 8.0.39 的枚举只有 11 个，不含这两个。它们是 `opt_explain.cc` 里的**显示别名**，注释自己就写了原因：

```cpp
/*
  For backward-compatibility, we have special presentation of "index
  lookup used for in(subquery)": we do not show "ref/etc", but
  "index_subquery/unique_subquery".
*/
if (join->query_expression()->item->engine_type() ==
    Item_subselect::INDEXSUBQUERY_ENGINE)
  str = (j_t == JT_EQ_REF) ? "unique_subquery" : "index_subquery";
```

即：**EXPLAIN 输出是用户可见的兼容性契约**。真正的访问方法只有 `eq_ref` / `ref`；这两个名字只在"某个 ref/eq_ref 恰好被用作 `IN (子查询)` 的索引查找"时换个显示名。现代 MySQL 里它们几乎见不到，是因为这类查询现在大多走 semi-join 了。

### 参数

| 参数 | 默认 | 说明 |
|---|---|---|
| `eq_range_index_dive_limit` | **200** | 等值 range 数超过它就改用统计（0=永远 dive） |
| `range_optimizer_max_mem_size` | **8388608**（8 MiB） | range 优化内存硬预算，超了放弃 range |
| `max_seeks_for_key` | **ULONG_MAX**（默认不生效） | clamp ref 的估算行数 |
| `optimizer_search_depth` | **MAX_TABLES + 1**（默认穷举） | 0=自动选合理值 |
| `optimizer_prune_level` | **1** | 0=不剪枝穷举；1=按行数启发式剪枝 |

⚠️ **`optimizer_switch` 里不存在 `range_scan` 开关**。range 只能整体通过 `range_optimizer_max_mem_size`（设小到触发放弃）或 `FORCE INDEX` / `IGNORE INDEX` 间接影响。

---

## 三、const table 检测

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

## 四、Key_use 数组：ref 访问的基础

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

## 五、find_best_ref：ref 访问的代价与 fanout

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

**★ 四个累积位图**（★ 本函数真正的"状态",每个 key 一组，出了内层循环就用它们做判定）：

```cpp
    key_part_map found_part = 0;           /* 当前 join 前缀下"可用"的 keypart */
    key_part_map const_part = 0;           /* 被【常量】绑定的 keypart（:334 置位） */
    key_part_map null_rejecting_part = 0;  /* NULL-rejecting 的 keypart（:343 置位） */
    key_part_map ref_or_null_part = 0;     /* "col=.. OR col IS NULL" 的 keypart（:363 置位） */
```

```cpp
        /* const_part / found_part 的置位点（:324-336） */
        if (!(keyuse->used_tables & ~join->const_table_map)) {
          // keyuse can be const-evaluated, if needed, using the const tables.
          if (!keyuse->null_rejecting &&                          // 过滤④
              keyuse->val->is_null() &&
              (table->file->index_flags(key, 0, false) & HA_TABLE_SCAN_ON_NULL))
            continue;
          const_part |= keyuse->keypart_map;    /* ★ 依赖表 ⊆ const_table_map ⇒ 常量绑定 */
        }
        found_part |= keyuse->keypart_map;      /* ★ 只要没被 4 条过滤掉就算"找到" */

        /* null_rejecting_part 的置位点（:337-344） */
        if (keypart != FT_KEYPART) {
          const bool keyinfo_maybe_null =
              keyinfo->key_part[keypart].field->is_nullable() ||
              tab->table()->is_nullable();
          if (keyuse->null_rejecting || !keyuse->val->is_nullable() ||
              !keyinfo_maybe_null)
            null_rejecting_part |= keyuse->keypart_map;
        }
```

> ★★ **`found_condition` 的语义比名字宽**（`:377`：`*found_condition |= (0 != found_part)`）——只要**有一个** keypart 被绑定就算 true，**包括 ref 用不上的那种**（例如索引 `(a,b)` 但只有 `b` 的条件）。这是它与"能否用 ref"的关键区别。

**★★ `keytype` 判定**（`:395-403`，★ 注意用的是 `keyinfo->flags`，**不是** `actual_key_flags()`）：

```cpp
      if (all_key_parts_covered && (keyinfo->flags & HA_NOSAME)) {
        if (key == table->s->primary_key &&
            table->file->primary_key_is_clustered())
          cur_keytype = CLUSTERED_PK;                 /* ★ 最高等级 */
        else if ((keyinfo->flags & HA_NULL_PART_KEY) == 0)
          cur_keytype = UNIQUE;                       /* 不可空唯一索引 */
        else if (all_key_parts_non_null)
          cur_keytype = UNIQUE;                       /* 可空唯一但全 keypart NULL-rejecting */
      }

      if (cur_keytype > best_found_keytype) {         /* ★ 枚举顺序即优先级：值越大越差 */
        trace_access_idx.add("chosen", false)
            .add_alnum("cause", "heuristic_eqref_already_found");
        if (unlikely(!test_all_ref_keys)) continue;   /* ★ 短路：已找到更高等级，跳过本索引 */
        /* 否则继续算 bound_keyparts/read_cost/fanout —— loosescan 驱动表需要填满所有 key */
      }
```

> ★★ **`test_all_ref_keys` 是这个短路的开关**。它只在 LooseScan 驱动表时为 true（`semijoin_firstmatch_loosescan_access_paths()` 里 `test_all_ref_keys = is_ls_driving_tab`）——因为 loosescan 要比较所有索引，不能只留最优那个。

**★★★ fanout 的三个来源（纠正：不含 `COND_FILTER_*`）**

`find_best_ref()` 内部**没有任何** `COND_FILTER_*`。真实三来源：

| 来源 | 代码 | 说明 |
|---|---|---|
| ① `table->quick_rows[key]` | `:454`（全键 const）、`:486`、`:551`（部分键）、`:651` | **复用 range 优化器的估计**（其底层才是 `records_in_range`）。四个复用点即源码注释里的 `ReuseRangeEstimateForRef-1..4` |
| ② `keyinfo->records_per_key(n-1)` | `:462`、`:558`、`:606` | `rec_per_key` 统计 |
| ③ 经验公式 | `:467`、`:611`、`:620-628` | 无统计时的兜底，见下 |

来源①的四个复用点各有严格前提（`ReuseRangeEstimateForRef-3` 的条件最完整）：

```cpp
        if (table->quick_keys.is_set(key) && !table_deps &&                    // (C1) 全是常量
            table->quick_key_parts[key] == cur_used_keyparts &&                // (C2) range 用了同样多的 keypart
            table->quick_n_ranges[key] == 1 + (ref_or_null_part ? 1 : 0))      // (C3) 区间数匹配
        {
          cur_fanout = (double)table->quick_rows[key];
        }
```

> ★ (C3) 解释了 ref_or_null 为什么是 **2 个区间**——一次查等值、一次查 IS NULL，正好对应下面 fanout 翻倍。

来源③的两个兜底公式：

```cpp
  /* (a) 全键、无 rec_per_key：偏好更长的键（:467-473） */
              cur_fanout =
                  ((double)tab->records() / (double)distinct_keys_est *
                   (1.0 + ((double)(table->s->max_key_length - keyinfo->key_length) /
                           (double)table->s->max_key_length)));
              if (cur_fanout < 2.0) cur_fanout = 2.0;  /* Can't be as good as a unique */

  /* (b) 部分键、无 rec_per_key：1%/10 线性插值（:605-630） */
            /*
              Assume that the first key part matches 1% of the file
              and that the whole key matches 10 (duplicates) or 1 (unique) records.
              ...
              records = (x * (b-a) + a*c-b)/(c-1)
            */
              const double a = tab->records() * 0.01;      /* ★ 第一个 keypart 匹配 1% */
              if (keyinfo->user_defined_key_parts > 1)
                tmp_fanout = (cur_used_keyparts * (rec_per_key - a) +
                              a * keyinfo->user_defined_key_parts - rec_per_key) /
                             (keyinfo->user_defined_key_parts - 1);
              else tmp_fanout = a;
```

> ★★ 两个"魔法数"：**0.01（首 keypart 匹配 1%）** 与 **`MATCHING_ROWS_IN_OTHER_TABLE = 10`**（`sql_planner.cc:91`，`distinct_keys_est = records / 10`）。这就是"无统计时纯靠拍"的具体落点。

**★ ref_or_null 让 fanout 翻倍**（`:633-636`）：

```cpp
          if (ref_or_null_part) {
            // We need to do two key searches to find key
            cur_fanout *= 2.0;
          }
```

**★★ `use_index_extensions` 在本函数内的两个口径**（★ 这是"规划期认为 eq_ref、建 ref 时退化成 ref"的一个潜在差异源）：

| 用途 | 口径 | 代码 |
|---|---|---|
| **能否算"全 keypart 覆盖"**（决定 eq_ref/UNIQUE） | `actual_key_parts(keyinfo)`（**含**索引扩展的尾部 PK 列） | `:379-384` |
| **rec_per_key 取哪一列**（全键分支） | `actual_key_parts(keyinfo) - 1` | `:462` |
| **无 rec_per_key 时的插值公式** | `keyinfo->user_defined_key_parts`（**不含**扩展） | `:606-625` |

**★★★ 索引选择规则："高等级无条件胜出"**（`:696-724`，★ 全篇最核心的三行）：

```cpp
    /*
      The current index usage is better than the best index usage found
      so far if:
       1) ... REF, and the current index has a lower cost
       2) ... both indexes are UNIQUE, and the current index has a lower cost
       3) The access type of the current index is better than that of the best
          index (EQ_REF better than REF, Clustered PK better than EQ_REF etc)
    */
    bool new_candidate = false;

    if (best_found_keytype >= NOT_UNIQUE && cur_keytype >= NOT_UNIQUE)
      new_candidate = cur_ref_cost < best_ref_cost;   // 1) 都是 REF 类 → 比代价
    else if (best_found_keytype == cur_keytype)
      new_candidate = cur_ref_cost < best_ref_cost;   // 2) 同等级 → 比代价
    else if (best_found_keytype > cur_keytype)
      new_candidate = true;                           // 3) ★★ 等级更高 → 无视代价直接替换

    if (new_candidate) {
      *ref_depend_map = table_deps;
      *used_key_parts = cur_used_keyparts;
      best_ref = start_key;         /* ★ 指向本 key 的【第一个】Key_use，不是"最佳 keypart"那个 */
      best_ref_cost = cur_ref_cost;
      best_found_keytype = cur_keytype;
    }

    if (best_found_keytype == CLUSTERED_PK) {
      trace_access_idx.add_alnum("cause", "clustered_pk_chosen_by_heuristics");
      if (unlikely(!test_all_ref_keys)) break;        /* ★ 拿到最高等级 → 直接结束整个外层循环 */
    }
```

**★ 三个副作用回填**（`:687-689`，★ 只写本 key 的**第一个** Key_use）：

```cpp
    start_key->bound_keyparts = found_part;
    start_key->fanout = cur_fanout;
    start_key->read_cost = cur_read_cost;
```

> ★ 这三个字段的注释在 `sql/sql_select.h:267-300` 明确说 *"set only by Optimize_table_order, and they change with the currently considered join prefix"*——**它们随 join 前缀变化**，所以必须在每次 `best_access_path()` 中重算。

**★★ `find_best_ref()` 不产出 `JT_EQ_REF`**（★ 常见误解）。它只产出 `idx_type` 与 `Key_use*`。真正的 `JT_EQ_REF`/`JT_REF`/`JT_REF_OR_NULL` 在 **`create_ref_for_key()`（`sql/sql_select.cc:2515-2536`）**：

```cpp
  else if (((actual_key_flags(keyinfo) & HA_NOSAME) == 0) ||        /* ① 非唯一索引 */
           ((actual_key_flags(keyinfo) & HA_NULL_PART_KEY) &&
            !null_rejecting_key) ||                                  /* ② 含可空 keypart 且不拒 NULL */
           keyparts != actual_key_parts(keyinfo)) {                  /* ③ 没用满 keypart */
    /* Must read with repeat */
    j->set_type(null_ref_key ? JT_REF_OR_NULL : JT_REF);
    j->ref().null_ref_key = null_ref_key;
  } else if (keyuse_uses_no_tables && !(... HA_BLOCK_CONST_TABLE)) {
    j->set_type(JT_CONST);
    j->position()->rows_fetched = 1.0;
  } else {
    j->set_type(JT_EQ_REF);        /* ★ 唯一 + 用满 keypart ⇒ eq_ref */
    j->position()->rows_fetched = 1.0;
  }
```

> ★★ **注意两个函数用的 flags 不同**：`find_best_ref` 用 `keyinfo->flags`，`create_ref_for_key` 用 `actual_key_flags(keyinfo)`（受 `use_index_extensions` 影响）。**这是"规划期算成 eq_ref、建 ref 时退化成 ref"的机制来源**。

**★ 部分键必须从最左前缀开始**（`:499` + `:664-668` 两处）：

```cpp
      } else if ((found_part & 1) &&        /* ★ 第一个 keypart（bit0）必须有谓词 */
                 (!(table->file->index_flags(key, 0, false) & HA_ONLY_WHOLE_INDEX) ||
                  all_key_parts_covered)) {
      ...
      } else {
        // No useful predicates on the first keypart; cannot use key
        trace_access_idx.add("usable", false).add("chosen", false);
        continue;
      }
```

而"连续性"由 `calc_length_and_keyparts()` 在**建 ref 时**保证（`sql_select.cc:2288`）：`keyparts == keyuse->keypart` 时才 `keyparts++`——一旦某级缺失，后面的 keypart 编号对不上，**全部丢弃**。

**★ `cur_used_keyparts` 有两个口径**（★ 易踩）：

```cpp
  cur_used_keyparts = (uint)~0;                  /* 全键覆盖：哨兵 ~0，不是真实个数 */
  cur_used_keyparts = max_part_bit(found_part);  /* 部分键：真实个数 = 最高置位 + 1 */
```

> ★ `~0` 这个哨兵在 `best_access_path()` 里与 `table->quick_key_parts[]` 比较时天然"很大"，语义正确但读起来很坑。

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

### 4.4 use_index_extensions：唯一性判定的两个口径

上面 4.3 的两个判据——**段数**和**唯一性**——在源码里其实各有两个口径，由 `use_index_extensions`（默认 **on**）切换。这是索引扩展影响优化器决策的唯一通道。

**构造侧**（追加隐藏 PK part、四条失效规则、可观测性）见 [`index/construction.md`](../../../../index/construction.md)，本节只讲**优化器消费侧**。

#### 三个消费点

开关的全部作用就是三个读取函数：

```cpp
uint actual_key_parts(const KEY *key_info) {
  return current_thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS)
             ? key_info->actual_key_parts
             : key_info->user_defined_key_parts;
}

uint actual_key_flags(const KEY *key_info) {
  return current_thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS)
             ? key_info->actual_flags
             : key_info->flags;
}
```

第三个是 `Field::is_part_of_actual_key()`（回答"该列是否属于这个索引的实际段"），它多一个条件：

```cpp
bool Field::is_part_of_actual_key(THD *thd, uint cur_index,
                                  KEY *cur_index_info) const {
  return thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS) &&
                 !(cur_index_info->flags & HA_NOSAME)
             ? part_of_key.is_set(cur_index)
             : part_of_key_not_extended.is_set(cur_index);
}
```

⚠️ 注意 `!(flags & HA_NOSAME)`：**唯一键不参与扩展判断**——与构造侧"唯一键不追加 PK part"是同一个约定。

**分歧的根源在构造侧**：`add_pk_parts_to_sk` 追加成功后只写 `sk->actual_flags |= HA_NOSAME`，**`flags` 保持原样**（开表时 `actual_flags` 才从 `flags` 初始化）。于是"用户声明的唯一性"与"实际唯一性"成了两个不同的位。

#### ★ 两个口径不一致

| 阶段 | 函数 | 段数判据 | 唯一性判据 | 受开关影响 |
|---|---|---|---|---|
| 代价估算 | `find_best_ref` | `actual_key_parts()` **受** | `keyinfo->flags` **不受** | 段数受、唯一性不受 |
| type 落地 | `create_ref_for_key` | `actual_key_parts()` **受** | `actual_key_flags()` **受** | 两处都受 |

```cpp
// find_best_ref
const bool all_key_parts_covered =
    (found_part == LOWER_BITS(key_part_map, actual_key_parts(keyinfo)));  // 段数：含扩展
...
if (all_key_parts_covered && (keyinfo->flags & HA_NOSAME)) {              // 唯一性：不含扩展
  ...cur_keytype = UNIQUE / CLUSTERED_PK;
}
```

```cpp
// create_ref_for_key
} else if (((actual_key_flags(keyinfo) & HA_NOSAME) == 0) ||             // 唯一性：含扩展
           ((actual_key_flags(keyinfo) & HA_NULL_PART_KEY) &&
            !null_rejecting_key) ||
           keyparts != actual_key_parts(keyinfo)) {
  j->set_type(null_ref_key ? JT_REF_OR_NULL : JT_REF);
  j->ref().null_ref_key = null_ref_key;
} else {
  j->set_type(JT_EQ_REF);
  j->position()->rows_fetched = 1.0;
}
```

**后果：扩展带来的"实际唯一"只在 type 落地时兑现，代价估算阶段不兑现。**

具体推演（`KEY(a)` 非唯一，InnoDB 里实为 `(a, PK)`，扩展后 `actual_flags` 带 `HA_NOSAME`；查询同时给定 `a` 与 PK 列的等值条件）：

- `create_ref_for_key`：`actual_key_flags` 有 `HA_NOSAME`、且 `keyparts == actual_key_parts` → **`JT_EQ_REF`**，并置 `rows_fetched = 1.0`
- `find_best_ref`：`flags` 无 `HA_NOSAME` → 判 `NOT_UNIQUE` → `cur_fanout` 走 `rec_per_key` 估算，**拿不到 4.3 里那个硬编码的 1.0**

所以估算偏保守（fanout 用统计值而非 1.0），但执行确实是 eq_ref、至多一行。

> ⚠️ 反过来就是该开关**最直接可观测的效果**：关掉后 `actual_key_flags()` 退化为 `flags`，上面这类场景**从 eq_ref 退化成 ref**。这也是判断"某条 SQL 是否吃到索引扩展红利"的最简验证——对比开关前后的 `EXPLAIN` type。

#### 段数口径的另一面：开关 ON 会让"覆盖全部段"更难

`all_key_parts_covered` 要求覆盖 `actual_key_parts()`。开关 ON 时它是**含隐藏 PK 的段数**，因此只给二级索引列条件（如仅 `WHERE a = 1`）就**不再满足**该判据。

这是正确行为而非退化：扩展后索引是 `(a, PK)`，只定 `a` 本就不能唯一确定一行，不该走 4.3 的 "use eq key" 分支。

#### 排查要点

- `EXPLAIN` 的 type 是 `eq_ref` 还是 `ref`，是观察该开关的唯一直接手段
- ⚠️ `SHOW CREATE TABLE` 显示的是 `flags`（用户声明），优化器用的是 `actual_flags`（含隐藏扩展）——**两者不一致是常态**，排查"为什么这个二级索引被当成唯一"时必须看 `actual_flags`

### 4.5 fanout 的三级来源

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

### 4.6 索引选择规则：`高等级无条件胜出`（`:709-724`）

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

## 六、best_access_path：ref / range / scan 的最终比较

`sql/sql_planner.cc:981`。

### 5.1 四条启发式短路（`:1087-1133`）

| # | 条件 | trace cause | 语义 |
|---|---|---|---|
| **1** | `rows_fetched < found_records && best_read_cost <= read_time` | `cost` | **ref 必胜**：行数更少且代价不高于扫描 |
| **2** | range 与 ref 同一索引且 ref 用的 keypart 更多 | `heuristic_index_cheaper` | 同索引上 ref 定位更精确，ref 不差于 range |
| **3** | `HA_TABLE_SCAN_ON_INDEX`（InnoDB）且有覆盖索引且有 ref | `covering_index_better_than_full_scan` | 表扫本身就是扫聚集索引，覆盖索引避免回表 |
| **4** | `force_index && best_ref && !range_scan` | `force_index` | FORCE INDEX 要求走索引，排除全表扫 |

**关于短路 1 的"不公平比较"**（注释 `:1054-1072`）：`best_read_cost` 是 ref 执行 `prefix_rowcount` 次的总代价，而 `read_time` 是**单次**扫描代价。若用了 join buffer，扫描不会执行 `prefix_rowcount` 次，而是 `prefix_rowcount / 每缓冲行数` 次——比例未知（1 到 `prefix_rowcount` 之间）。为**不低估**扫描代价，启发式假设最坏情况（比例=1）。更精细的比较留给 `calculate_scan_cost`。

**★★ 四条短路的真实形态**（★ 不是 `goto`/`return`，是一条 `if / else if ×3 / else` 链——**前 3 条只写 trace 就跳过扫描估算，第 4 条 `else` 才真正算 scan**）：

```cpp
  if (rows_fetched < tab->found_records &&      // (1a)
      best_read_cost <= tab->read_time)         // (1b)
  {
    trace_access_scan.add("chosen", false).add_alnum("cause", "cost");
  } else if (tab->range_scan() && best_ref &&                            // (2)
             used_index(tab->range_scan()) == best_ref->key &&
             used_key_parts >= table->quick_key_parts[best_ref->key] &&
             tab->range_scan()->type != AccessPath::GROUP_INDEX_SKIP_SCAN &&
             tab->range_scan()->type != AccessPath::INDEX_SKIP_SCAN)
  {
    trace_access_scan.add("chosen", false)
        .add_alnum("cause", "heuristic_index_cheaper");
  } else if ((table->file->ha_table_flags() & HA_TABLE_SCAN_ON_INDEX) &&  //(3)
             !table->covering_keys.is_clear_all() && best_ref &&
             (!tab->range_scan() ||
              (tab->range_scan()->type == AccessPath::ROWID_INTERSECTION &&
               best_ref->read_cost < tab->range_scan()->cost)))
  {
    trace_access_scan.add("chosen", false)
        .add_alnum("cause", "covering_index_better_than_full_scan");
  } else if ((table->force_index && best_ref && !tab->range_scan()))      // (4)
  {
    trace_access_scan.add_alnum("access_type", "scan")
        .add("chosen", false).add_alnum("cause", "force_index");
  } else {
    /* ★ 只有走到这里才真正算扫描代价 */
    double rows_after_filtering;
    double scan_read_cost = calculate_scan_cost(
        tab, idx, best_ref, prefix_rowcount, found_condition, disable_jbuf,
        &rows_after_filtering, &trace_access_scan);
    ...
  }
```

> ★★ 短路 (2) 的**两个 8.0 新增排除**：`GROUP_INDEX_SKIP_SCAN` 与 `INDEX_SKIP_SCAN` 不在短路范围内——因为 loose scan 与 ref 用的不是同一段索引，不能简单判定"ref 更精确"。
> ★ 短路 (3) 里 `ROWID_INTERSECTION`（index merge）是**例外**：只有当 `best_ref->read_cost < range_scan->cost` 时才敢判它输——注释明说 *"We can not say for sure that INDEX_MERGE quick select is always faster than ref access"*。
> ⚠️ **`best_access_path()` 里没有 `straight_join` / `found_const` 这两个变量**。`straight_join` 是 `choose_table_order()` 的局部变量（`:1971`），只用于决定走 `optimize_straight_join()` 还是 `greedy_search()`。

**★ 调用骨架**（`best_access_path` 本身不循环，调用方有三处）：

```cpp
/* ① optimize_straight_join()（:2117-2162）—— 顺序推进 */
  for (JOIN_TAB **pos = join->best_ref + idx; *pos; idx++, pos++) {
    best_access_path(s, join_tables, idx, false, rowcount, position);
    position->set_prefix_join_cost(idx, cost_model);      /* ★ 由 read_cost/rows_fetched/filter_effect 推出 prefix_* */
    rowcount = position->prefix_rowcount;                 /* ★★ 滚动前缀 → 喂给下一张表 */
    cost = position->prefix_cost;
  }

/* ② greedy_search()（:2786-2791） */
      best_access_path(s, remaining_tables, idx, false,
                       idx ? (position - 1)->prefix_rowcount : 1.0, position);
      position->set_prefix_join_cost(idx, cost_model);

/* ③ best_extension_by_limited_search()（:3738-3742）★ 唯一传 disable_jbuf=true 的地方 */
      test_all_ref_keys = is_ls_driving_tab;
      double prefix_rowcount = rowcount * inner_fanout * outer_fanout;
      best_access_path(tab, remaining_tables, i, i < no_jbuf_before,
                       prefix_rowcount, dst_pos);
      test_all_ref_keys = false;
```

> ★ `choose_table_order()` 开头给 const 表打的种子是 `set_prefix_cost(0.0, 1.0)`——**代价 0、行数 1**，这是整个前缀代价链的起点。

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

**★★ 写回 POSITION 的全部 8 行**（`:1217-1224`，★ 文档原稿说"写回在 1159-1224"其实**只有这 8 行**——1159-1188 只是在"scan 胜出"时改写**局部变量**）：

```cpp
  best_read_cost += derived_mat_cost;      /* LATERAL 派生表物化代价在此并入 */
  pos->filter_effect = filter_effect;
  pos->rows_fetched = rows_fetched;
  pos->read_cost = best_read_cost;
  pos->key = best_ref;                     /* ★★ nullptr ⟺ 选了扫描；非 null ⟺ 选了 [eq_]ref[_or_null] */
  pos->table = tab;
  pos->ref_depend_map = ref_depend_map;
  pos->loosescan_key = MAX_KEY;
  pos->use_join_buffer = best_uses_jbuf;   /* 仅 scan 胜出时才可能为真 */
```

> ★★ **`pos->read_cost` 不含 `row_evaluate_cost`**——那部分由调用方的 `set_prefix_join_cost()` 统一加。这是最容易算错代价的一处口径。
> ⚠️ 这里**没有**写 `prefix_cost` / `prefix_rowcount`——那是 `set_prefix_join_cost()` 的事。

**★★ `POSITION` 完整定义**（⚠️ 在 **`sql/sql_select.h:352`**，不是 `sql_planner.h`；且必须保持 POD，注释原文 *"This class has to stay a POD, because it is memcpy'd in many places"*）：

```cpp
struct POSITION {
  /* 每个前一行组合取到的行数 = selectivity(access_condition) * cardinality(table) */
  double rows_fetched;

  /* 整个 join 执行中访问本表的代价 = 单次访问代价 × 前缀行数。
     ★ 不含 row_evaluate_cost */
  double read_cost;

  /* 未被访问方法使用的表条件，让多少比例的 rows_fetched 通过。合法范围 0..1
     ★ 注意：可能出现 rows_fetched * filter_effect < 1.0（见原注释的例子） */
  float filter_effect;

  double prefix_rowcount;   /* 交给下一张表的行组合数 */
  double prefix_cost;

  JOIN_TAB *table;

  /* ★★ NULL = index/range/index_merge/ALL；非 NULL = [eq_]ref[_or_null]，指向 Key_use */
  Key_use *key;

  table_map ref_depend_map;
  bool use_join_buffer;

  uint sj_strategy;
  uint n_sj_tables;
  table_map dups_producing_tables;
  /* ... LooseScan / FirstMatch / DuplicateWeedout / SJM-Scan 各自的状态字段 ... */
};
```

**★ 两个 `set_prefix_*` 的计算式**（★ 都是**内联成员函数**，不是独立函数）：

```cpp
  void set_prefix_cost(double cost, double rowcount) {
    prefix_cost = cost;
    prefix_rowcount = rowcount;      /* 直接赋值：const 表 set_prefix_cost(0.0, 1.0)，loosescan 手工回填 */
  }

  void set_prefix_join_cost(uint idx, const Cost_model_server *cm) {
    if (idx == 0) {
      prefix_rowcount = rows_fetched;
      prefix_cost = read_cost + cm->row_evaluate_cost(prefix_rowcount);
    } else {
      prefix_rowcount = (this - 1)->prefix_rowcount * rows_fetched;
      prefix_cost = (this - 1)->prefix_cost + read_cost +
                    cm->row_evaluate_cost(prefix_rowcount);
    }
    prefix_rowcount *= filter_effect;    /* ★★ filter_effect 最后才乘 */
  }
```

> ★★★ **`filter_effect` 是在 `prefix_rowcount` 算完之后才乘的**，而 `prefix_cost` 里的 `row_evaluate_cost(prefix_rowcount)` 用的是**未乘** `filter_effect` 的值。语义精确对应："**取出来的行都要 evaluate 一遍，通过过滤的行才进入下一级扇出**"。

---

## 七、二次 range 重估

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

**★★ 两个位置勘误**：

1. ⚠️ **`enum enum_recheck_reason`（含 `RECHECK_NONE`/`RECHECK_RANGE`/`RECHECK_INDEX_MERGE`）在 8.0.39 不存在**——那是 5.x 的写法。真实形态是 `make_join_query_block()` 内的**匿名局部 enum**：

```cpp
/* sql/sql_optimizer.cc:9753 */
          enum { DONT_RECHECK, NOT_FIRST_TABLE, LOW_LIMIT }
              recheck_reason = DONT_RECHECK;
```

2. ⚠️ **recheck 逻辑不在 `JOIN::adjust_access_methods()`**。`adjust_access_methods()`（`:2952-3026`）只做两件启发式替换：`JT_ALL → JT_INDEX_SCAN`（覆盖索引更窄）与 `JT_REF → JT_RANGE`。整段 recheck 在 **`make_join_query_block()`**（入口 `:9579`）内部的 `:9753-9946`。

**★ `adjust_access_methods()` 全文骨架**（★ 二次 range 重估的调用点 #2 就在这里）：

```cpp
void JOIN::adjust_access_methods() {
  for (uint i = const_tables; i < tables; i++) {
    JOIN_TAB *const tab = best_ref[i];
    if (tab->type() == JT_ALL) {
      /* ALL → INDEX_SCAN：纯启发式，非代价决策 */
      if (!tab->table()->no_keyread &&                    // 1 允许 keyread
          !tab->table()->covering_keys.is_clear_all() &&   // 2 有覆盖索引
          !tl->uses_materialization()) {                   // 3 非派生/物化
        if (tab->position()->sj_strategy != SJ_OPT_LOOSE_SCAN)
          tab->set_index(find_shortest_key(tab->table(), &tab->table()->covering_keys));
        tab->set_type(JT_INDEX_SCAN);
      }
    } else if (tab->type() == JT_REF) {
      if (can_switch_from_ref_to_range(thd, tab, ORDER_NOT_RELEVANT, false)) {
        /* ★★ 调用点 #2：recheck_range = false，不重跑 range 优化器 */
        tab->set_type(JT_RANGE);
        tab->use_quick = QS_RANGE;
        tab->position()->filter_effect = COND_FILTER_STALE;   /* ★ 旧过滤系数作废 */
      } else {
        ::destroy(tab->range_scan());
        tab->set_range_scan(nullptr);
      }
    }
    /* ... 这里还可能把整个查询判成 "Impossible WHERE noticed after reading const tables" ... */
  }
}
```

> ★ 注释里还留着一个被 BUG#35850 否掉的改动：*"See bug #26447: Using the clustered index for a table scan is always faster than using a secondary index"*——因为磁盘缓存场景下反而变慢，已注释掉。

**★ `can_switch_from_ref_to_range()` 只有 2 个调用点**（不是 3 个），参数语义完全不同：

| # | 位置 | `ordering` | `recheck_range` | 行为 |
|---|---|---|---|---|
| 1 | `:2621`（`test_if_skip_sort_order()` 内） | `order.order->direction`（可能 ASC/DESC） | **`true`** | **重跑一次** range 优化器（只针对 ref 那一个索引），为 ORDER BY 早停 |
| 2 | `:2991`（`adjust_access_methods()` 内） | `ORDER_NOT_RELEVANT` | **`false`** | **不重跑**，只拿现有 `range_scan` 比 key 长度 |

**★ `test_quick_select()` 的返回是三态**（`sql/range_optimizer/range_optimizer.cc:515-518`）：

```cpp
  RETURN
   -1 if impossible select (i.e. certainly no rows will be selected)
    0 if can't use quick_select
    1 if found usable ranges and quick select has been successfully created.
```

> ★ 注意**不是**"返回行数"——行数要取 `range_scan->num_output_rows()`。重估段正是靠 `test_quick_select(...) < 0` 来判 "impossible"。
> ★ 重估段**调用了两次**：第二次是为了区分"impossible WHERE"与"impossible ON"（外连接场景下 ON 不成立不等于整个查询空）。

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

## 八、Index_lookup：ref 访问的键值绑定

前几章讲的是**优化器怎么选**访问方法（Key_use 候选 → find_best_ref 代价 → best_access_path 比较）。本章讲选定的 ref 访问**怎么变成可执行的东西**——即 `Index_lookup` 这个结构体。它是优化器与执行器之间的**接口结构**：优化期填充，执行期消费。

### 8.1 术语澄清：代码里有四个互不相干的 `ref`

结构体头注释交代了身份与历史包袱：

```cpp
/**
  Structure used for index-based lookups.

  @todo Remove any references to the old name: it used to be called TABLE_REF,
        hence many variables still have the ref prefix. In MySQL "index lookup"
        and "ref access" are used interchangeable but in EXPLAIN we output the
        former.
*/
```

⚠️ **代码里遍地的 `ref` 前缀（`m_ref`、`init_ref()`、`create_ref_for_key()`）来自旧名 `TABLE_REF`**——那条 `@todo` 至今未完成，所以 8.0.39 仍是满屏 `ref`。这是"第四个意思"，与本篇反复强调的易混淆命名并列：

| # | `ref` 是什么 | 出处 |
|---|---|---|
| 1 | **`handler::ref` = 行位置（rowid）** | `handler.h` 注释：*"Pointer to current row"* |
| 2 | **EXPLAIN `type` 列的访问类型串** | `"ref"` / `"eq_ref"` / `"ref_or_null"` |
| 3 | **EXPLAIN `ref` 列**（`const` / `func` / 列名） | 见 8.8 |
| 4 | **`Index_lookup` 的 `ref` 前缀** | 本章主角，历史遗留 |

而注释说 "in EXPLAIN we output the former"（输出 index lookup）指的是 **EXPLAIN FORMAT=JSON**——传统 EXPLAIN 仍叫 `ref`/`eq_ref`，JSON 里统一叫 `index_lookup`。

### 8.2 结构全字段

```cpp
struct Index_lookup {
  bool key_err;                 // 上次 construct_lookup() 是否失败
  uint key_parts;               // 参与查找的 keypart 数
  uint key_length;              // key buffer 字节长度（= EXPLAIN key_len）
  int key;                      // 索引编号
  uchar *key_buff;              // 待查找的键值缓冲区
  uchar *key_buff2;             // key_buff + key_length（仅 EQRefIterator 用）
  store_key **key_copy;         // ★ 每个 keypart 一个"值搬运器"
  Item **items;                 // 每个 keypart 的表达式
  bool **cond_guards;           // 可开关条件（见 8.4）
  key_part_map null_rejecting;  // 位图：items[i] 为 NULL 则不可能匹配
  table_map depend_map;         // 依赖哪些表
  uchar *null_ref_key;          // REF_OR_NULL 的 NULL 标志字节
  ha_rows use_count;            // 行缓存的锁计数
  bool disable_cache;           // ICP 场景下禁用行缓存
  ulonglong *keypart_hash;      // 8.0 新增：所有字段哈希合并成一个值
};
```

**★ 三处修正**（逐个核实）：

1. ⚠️ **真身在 `sql/sql_opt_exec_shared.h:67-176`，不在 `sql_executor.h` / `sql_select.h`**（后两处只有前向引用与形参）。且它是 **`struct` 不是 class**。
2. ⚠️ **`key_buff2` 不是 `key_buff + key_length`**（源码注释 `///< key_buff+key_length` 是历史遗留误导）。`init_ref()` 里**独立分配了两块**等长缓冲：

```cpp
bool init_ref(THD *thd, unsigned keyparts, unsigned length, unsigned keyno,
              Index_lookup *ref) {
  ref->key_parts = keyparts;
  ref->key_length = length;
  ref->key = keyno;
  if (!(ref->key_buff = thd->mem_root->ArrayAlloc<uchar>(ALIGN_SIZE(length))) ||
      !(ref->key_buff2 =
            thd->mem_root->ArrayAlloc<uchar>(ALIGN_SIZE(length))) ||   /* ★ 独立第二块 */
      !(ref->key_copy = thd->mem_root->ArrayAlloc<store_key *>(keyparts)) ||
      !(ref->items = thd->mem_root->ArrayAlloc<Item *>(keyparts)) ||
      !(ref->cond_guards = thd->mem_root->ArrayAlloc<bool *>(keyparts)))
    return true;
  ref->key_err = true;
  ref->null_rejecting = 0;
  ref->use_count = 0;
  ref->disable_cache = false;
  return false;
}
```

3. ⚠️ **`"8.0 新增"` 的 `keypart_hash` 与"ref 缓存"完全无关**。它只在**物化子查询 + `tmp_table->hash_field`（键总长超限时的唯一约束哈希列）**时出现，见 8.6。

`key_copy` 字段的注释原文（★ 这是理解 ref 为什么快的关键）：

```cpp
  /**
     Used to store the value from each keypart field. These values are
     used for ref access. If key_copy[key_part] == NULL it means that
     the value is constant and does not need to be reevaluated
  */
  store_key **key_copy;
```

`cond_guards` 的语义原文（★ 见 8.4）：

```cpp
  /*
    Array of pointers to trigger variables. Some/all of the pointers may be
    NULL.  The ref access can be used iff

      for each used key part i, (!cond_guards[i] || *cond_guards[i])
    ...
  */
```

**★ `store_key` 类族只有 4 个**（⚠️ `store_key_field` / `store_key_item` / `store_key_null` **都不存在**，那是 5.6 的写法，8.0 已被 `get_store_key()` 的三分支取代）：

```cpp
/* 工厂：sql/sql_select.cc:2589 */
static store_key *get_store_key(THD *thd, Item *val, table_map used_tables,
                                table_map const_tables,
                                const KEY_PART_INFO *key_part, uchar *key_buff,
                                uint maybe_null) {
  if (key_part->field->is_array())          /* 多值索引（JSON / 函数索引） */
    return new store_key_json_item(thd, key_part->field, key_buff + maybe_null,
                                   maybe_null ? key_buff : nullptr,
                                   key_part->length, val,
                                   (!((~const_tables) & used_tables)));
  if (!((~const_tables) & used_tables))     /* ★ used_tables ⊆ const_tables ⇒ 常量 */
    return new store_key_const_item(thd, key_part->field, key_buff + maybe_null,
                                    maybe_null ? key_buff : nullptr,
                                    key_part->length, val);
  return new store_key(...);                /* 非常量 ⇒ 每行都要 copy() */
}
```

**★★ "常量固化"的两个落点**（★ ref 为什么快）：

```cpp
/* ① store_key_const_item::copy_inner() —— 只算一次并缓存结果 */
class store_key_const_item final : public store_key {
  int cached_result = -1;                 /* -1 = 尚未算过 */
 protected:
  enum store_key_result copy_inner() override {
    if (cached_result == -1) {
      cached_result = store_key::copy_inner();   /* ★★ 常量只算一次，之后永远复用 */
    }
    return static_cast<store_key_result>(cached_result);
  }
};

/* ② init_ref_part() —— 优化期就把值写进 key_buff，并把 key_copy[i] 置 nullptr */
    (void)s_key->copy();                  /* ★ 优化期固化 */
    if (s_key->null_key)
      ref->key_copy[part_no] = s_key;     /* 常量但为 NULL ⇒ 执行期仍需重算（Full scan on NULL key） */
    else
      ref->key_copy[part_no] = nullptr;   /* ★★ 真正固化：置 nullptr */

/* ③ 执行期 construct_lookup() —— 跳过 nullptr */
  for (uint part_no = 0; part_no < ref->key_parts; part_no++) {
    store_key *s_key = ref->key_copy[part_no];
    if (!s_key) continue;                 /* ★ nullptr = 已固化，直接跳过 */
    if (s_key->copy() != store_key::STORE_KEY_OK || thd->is_error()) {
      result = true; break;
    }
  }
```

**★ `copy()` 刻意关掉截断检查**（★ 键缓冲的截断绝不能被外层 INSERT/UPDATE 提升为错误）：

```cpp
store_key::store_key_result store_key::copy() {
  thd->check_for_truncated_fields = CHECK_FIELD_IGNORE;   /* ★ 关键 */
  thd->variables.sql_mode &= ~(MODE_NO_ZERO_IN_DATE | MODE_NO_ZERO_DATE);
  result = copy_inner();
  thd->check_for_truncated_fields = saved_check_for_truncated_fields;
  thd->variables.sql_mode = sql_mode;
  return result;
}
```

`store_key_result` 三态：`STORE_KEY_OK` / `STORE_KEY_FATAL`（致命，放弃查找）/ `STORE_KEY_CONV`（截断了，仍可继续）。

**★ `null_rejecting` 的真正计算点**（⚠️ **不在** `calc_length_and_keyparts()`——那个函数只算 `length/keyparts/dep_map/maybe_null`）：

```cpp
/* sql/sql_select.cc:2344 —— init_ref_part() 内，唯一置位点 */
  // Set ref as "null rejecting" only if either side is really nullable:
  if (null_rejecting && (nullable || val->is_nullable()))
    ref->null_rejecting |= (key_part_map)1 << part_no;
```

三段流转在 `add_not_null_conds()`（`sql_optimizer.cc:6383`）的注释里写得很清楚：

```cpp
      1. update_ref_and_keys() accumulates info about null-rejecting
         predicates in in Key_field::null_rejecting
      1.1 add_key_part saves these to Key_use.
      2. create_ref_for_key copies them to Index_lookup.
      3. add_not_null_conds adds "x IS NOT NULL" to join_tab->m_condition of
         appropriate JOIN_TAB members.
```

> ★ 第 3 步就是 **"Early NULLs Filtering"**——把 `IS NOT NULL` 推给**外层表**，让它在更早的阶段就过滤掉。而 `impossible_null_ref()` 是 **"Late NULLs Filtering"**（执行期在每个 `Iterator::Read()` 开头调用）。两个过滤器同名不同阶段。

两个内联方法值得单说：

```cpp
  /// @returns whether the reference contains NULL values which could never give a match.
  bool impossible_null_ref() const {
    if (null_rejecting == 0) return false;
    for (uint i = 0; i < key_parts; i++) {
      if ((null_rejecting & 1 << i) && items[i]->is_null()) return true;
    }
    return false;
  }
```

这是**执行期的 "late NULLs filtering"**——不等引擎返回，先判"这个 key 里含 NULL，而该 keypart 是 null-rejecting，必然无匹配"，直接跳过索引查找。

### 8.3 ★ 键值绑定：`store_key` 与常量固化（ref 为什么快）

`key_copy` 数组每个元素是一个"值搬运器"，负责把一个 `Item` 的值按 **MySQL key 格式**写进 `key_buff` 的对应槽位。

四个场景对应四个类：

| 类 | 场景 |
|---|---|
| `store_key`（基类） | 非常量值（依赖外层表），每次都要重新求值 |
| `store_key_const_item` | 常量值，结果只算一次（有 `cached_result`） |
| `store_key_hash_item` | 物化临时表走 hash 去重（见 8.6） |
| `store_key_json_item` | JSON 表达式上的函数索引（多值索引） |

选择逻辑就一个函数 `get_store_key()`。注意它传给 `store_key` 的两个指针：

```cpp
return new (thd->mem_root) store_key(thd, key_part->field, key_buff + maybe_null,
                                     maybe_null ? key_buff : nullptr, ...);
```

**每个 keypart 槽位的第一个字节是 NULL 标志字节，后面才是值**——这是 8.5 的物理基础。

#### ★ `key_copy[i] == NULL` = 常量、执行期完全不用重算

这是 "ref 访问为什么快" 的核心之一，实现在 `init_ref_part()`：

```cpp
  if (used_tables & ~INNER_TABLE_BIT) {
    /* Comparing against a non-constant. */
    ref->key_copy[part_no] = s_key;
  } else {
    /*
      The outer reference is to a const table, so we copy the value
      straight from that table now (during optimization), instead of from
      the temporary table created during execution.
      ...
      It should be reevaluated in ::exec() if
      constant evaluated to NULL value which we might need to
      handle as a special case during JOIN::exec()
      (As in : 'Full scan on NULL key')
    */
    (void)s_key->copy();
    if (s_key->null_key)
      ref->key_copy[part_no] = s_key;   // 常量求值成 NULL，执行期还需重算
    else
      ref->key_copy[part_no] = nullptr; // ★ 固化了，执行期直接跳过
  }
```

而执行期的填充循环就一行 `if (!s_key) continue;`：

```cpp
bool construct_lookup(THD *thd, TABLE *table, Index_lookup *ref) {
  for (uint part_no = 0; part_no < ref->key_parts; part_no++) {
    store_key *s_key = ref->key_copy[part_no];
    if (!s_key) continue;      // ★ 常量 keypart 直接跳过
    if (s_key->copy() != store_key::STORE_KEY_OK || thd->is_error()) {
      result = true; break;
    }
  }
}
```

**完整链条**：`WHERE t1.a = 5 AND t1.b = t2.b`（`a` 的 keypart 是常量）→ `key_copy[0] == nullptr` → 每个外层行只需为 keypart 1 做一次 `save_in_field()` → key buffer 里 `5` 那一段永远不动 → 每次迭代成本是 **O(变化的那几个 keypart)**，不是 O(全部)，更不是重新走一遍表达式求值。

> 这也解释了 EXPLAIN `ref` 列里的 `const` 是什么——见 8.8。

真正的搬运靠 `to_field`：它是构造时用 `new_key_field()` 造出来的"指向 `key_buff` 内某段内存的 Field"，所以 `item->save_in_field(to_field)` 直接按 key 格式写进缓冲区（含 NULL 标志字节、BLOB 的 2 字节长度前缀）。

#### ★ copy() 的静默语义：`(void)` 扔返回值与"合法的脏 key_buff"

**这是 ref 访问最反直觉的一段设计，也是一类真实 0 行 bug 的根源**（判例见文末）。

**第一层：copy() 外壳主动关闭一切报错**：

```cpp
store_key::store_key_result store_key::copy() {
  enum store_key_result result;
  THD *thd = current_thd;
  enum_check_fields saved_check_for_truncated_fields =
      thd->check_for_truncated_fields;
  sql_mode_t sql_mode = thd->variables.sql_mode;
  thd->variables.sql_mode &= ~(MODE_NO_ZERO_IN_DATE | MODE_NO_ZERO_DATE);

  thd->check_for_truncated_fields = CHECK_FIELD_IGNORE;

  result = copy_inner();

  thd->check_for_truncated_fields = saved_check_for_truncated_fields;
  thd->variables.sql_mode = sql_mode;

  return result;
}
```

两个"静默开关"：

1. **`check_for_truncated_fields = CHECK_FIELD_IGNORE`**——`save_in_field()` 内部所有截断/降级路径（`TYPE_NOTE_TRUNCATED`/`TYPE_WARN_TRUNCATED` 等）产生的警告全部被吞；
2. **临时关掉 `NO_ZERO_IN_DATE`/`NO_ZERO_DATE`**——对日期键part，非法日期（`0000-00-00`、越界日）**不报错、写出零值**。对日期分区键这是致命的组合。

**第二层：截断在映射函数里被"洗白"成 OK**：

```cpp
static store_key::store_key_result type_conversion_status_to_store_key(
    THD *thd, type_conversion_status ts) {
  switch (ts) {
    case TYPE_OK:
      return store_key::STORE_KEY_OK;
    case TYPE_NOTE_TRUNCATED:
    case TYPE_WARN_TRUNCATED:
    case TYPE_NOTE_TIME_TRUNCATED:
      if (thd->check_for_truncated_fields)
        return store_key::STORE_KEY_CONV;
      else
        return store_key::STORE_KEY_OK;   // ★ IGNORE 模式下"截断"= OK！
    case TYPE_WARN_OUT_OF_RANGE:
    case TYPE_WARN_INVALID_STRING:
    case TYPE_ERR_NULL_CONSTRAINT_VIOLATION:
    case TYPE_ERR_BAD_VALUE:
    case TYPE_ERR_OOM:
      return store_key::STORE_KEY_FATAL;
  }

  assert(false);  // not possible
}
```

**五个返回值的全谱系**：

| save_in_field 结果 | IGNORE 模式下映射 | 含义 |
|-------------------|------------------|------|
| `TYPE_OK` | `STORE_KEY_OK` | 原值完整写入 |
| `TYPE_NOTE_TRUNCATED` / `TYPE_WARN_TRUNCATED` / `TYPE_NOTE_TIME_TRUNCATED` | **`STORE_KEY_OK`**（★ 洗白点） | 值被截断写入（≠原值），当成功 |
| `TYPE_WARN_OUT_OF_RANGE` | `STORE_KEY_FATAL` | 越界：写的是裁剪后的边界值 |
| `TYPE_WARN_INVALID_STRING` / `TYPE_ERR_BAD_VALUE` | `STORE_KEY_FATAL` | 非法值：写的是零值/默认值 |
| `TYPE_ERR_NULL_CONSTRAINT_VIOLATION` / `TYPE_ERR_OOM` | `STORE_KEY_FATAL` | 真错误 |

> 注意 FATAL 也不等于"key_buff 没被写"——`save_in_field` 失败路径**往往已经把零值/边界值写进 `to_field`**（Field 的失败语义是"写入一个安全值 + 返回失败状态"）。所以无论映射成 OK 还是 FATAL，**key_buff 里都可能已经是"≠WHERE 原值"的字节**；区别只是 FATAL 会被 `construct_lookup` 的执行期检查捕获（见 8.3 上方代码），而常量 part 的 OK 路径连这个都没有。

`copy()` 外壳已把开关置 IGNORE ⇒ **截断类失败返回 `STORE_KEY_OK`**（连 CONV 都不给）。`key_buff` 里写进的是**截断后的值**——不是 WHERE 里的原值。

**第三层：常量路径连返回值都扔掉**（`init_ref_part()`，就是上一节 8.3 那段 `(void)s_key->copy()`）：

```cpp
    /*
      Note:
        Result check of store_key::copy() is unnecessary,
        it could be an error returned by store_key::copy() method
        but stored value is not null and default value could be used
        in this case. Methods which used for storing the value
        should be responsible for proper null value setting
        in case of an error. Thus it's enough to check s_key->null_key
        value only.
    */
    (void)s_key->copy();
```

注释原文承认：**copy() 的结果检查是不必要的**——失败时"缓冲区里存的值非 NULL，就用默认值"。官方语义 = *"key_buff 里是什么就用什么，失败不报错"*。

**第四层：key_buff 分配即脏**：

```cpp
bool init_ref(THD *thd, unsigned keyparts, unsigned length, unsigned keyno,
              Index_lookup *ref) {
  ref->key_parts = keyparts;
  ref->key_length = length;
  ref->key = keyno;
  if (!(ref->key_buff = thd->mem_root->ArrayAlloc<uchar>(ALIGN_SIZE(length))) ||
      !(ref->key_buff2 =
            thd->mem_root->ArrayAlloc<uchar>(ALIGN_SIZE(length))) ||
      !(ref->key_copy = thd->mem_root->ArrayAlloc<store_key *>(keyparts)) ||
      !(ref->items = thd->mem_root->ArrayAlloc<Item *>(keyparts)) ||
      !(ref->cond_guards = thd->mem_root->ArrayAlloc<bool *>(keyparts))) {
    return true;    // 只有 OOM 才算失败
  }
```

五个数组全部 `ArrayAlloc`，**没有任何一处清零**：`key_buff`/`key_buff2` 是 `uchar`（placement new 是 no-op）、`key_copy`/`items`/`cond_guards` 是指针数组（**连指针都不清零**，靠后续循环逐个赋值，OOM 早退时剩余槽是垃圾）。`key_buff2` 是 EQRefIterator 单行缓存的"上次 key 快照"，首次 `memcmp(key_buff2, key_buff)` 之前也依赖 `use_count==0` 短路保护。

`ArrayAlloc<uchar>` 对 `uchar`（内置类型）placement new 是 no-op，MEM_ROOT 的 `Alloc` 本身也不清零（arena 复用 + malloc 堆复用）——**key_buff 的初值是内存里恰好残留的旧内容**（上一个语句/上一个连接用过的 key_buff）。

**四层叠加的后果**：常量 ref 的 key_buff = 「脏初值 + copy() 若静默失败只写了部分/写入了截断值」，且上层零感知。正常情况 copy() 成功覆盖全部字节，一切安好；一旦触发截断路径，key_buff 的语义就是"部分新值 + 部分残留"，且**无任何报错**。

#### 判例：分区表 ref 等值查询 0 行跳跃（真实 bug 模式）

**复现**：短连接循环执行 `SELECT count(*) FROM t PARTITION (p_x) WHERE tran_date = '<该分区内的日期>'`——count 在 0 与正确值之间跳跃，`Handler_read_key` 0/1 交替。

**因果链**（每一步都有源码落点）：

```
key_buff = ArrayAlloc<uchar>          → 初值 = 内存残留（第四层）
copy(): CHECK_FIELD_IGNORE + 关 NO_ZERO_DATE
                                      → 日期截断/零日期静默（第一、二层）
type_conversion_status_to_store_key   → 截断 = STORE_KEY_OK（第二层）
(void)s_key->copy()                   → 常量 part 固化，执行期不再重写（第三层）
get_partition_set(): key_buff 二进制  → key_restore 还原出"漂移的日期值"
  → 分区函数（如 TO_DAYS）            → 分区号 P ≠ 数据实际所在分区
prune_partition_set(): [P,P] ∩ read_partitions = ∅
                                      → start_part = end_part + 1
partition_scan_set_up()               → EOF，扫描未进引擎（Handler_read_key=0）
```

**为什么"跳跃"**：copy() 成功的轮次 key_buff 正确 → 正常行数（`Handler_read_key=1`）；静默截断的轮次 key_buff 是截断值/残留混合 → P 漂移 → 0 行。是否截断取决于连接的 sql_mode/字符集等会话上下文与残留内容的组合，短连接循环下呈随机分布。

**为什么 `PARTITION (p_x)` 是放大器**：它把 `read_partitions` 钉死为单分区位图——漂移的 P 只要 ≠ p_x，交集必空；不加 PARTITION 子句时 read_partitions 是优化期裁剪结果（往往多个分区），漂移的 P 仍可能落在其中，返回**别的分区的 0 行**（更隐蔽）。

**规避**：等值改区间（`tran_date >= '...' AND tran_date < '...'`）——`get_partition_set` 的快速路径要求 `flag == HA_READ_KEY_EXACT`（见 [`index/partitioning.md`](../../../../index/partitioning.md)），区间走优化期 range 裁剪，完全绕开 store_key 的静默路径。

#### 调用时机（不是每次 Read 都调）

- `RefIterator::Read()`：只在 `m_first_record_since_init` 时（每次 `Init()` 后第一次 Read = 每个新外层行一次）
- `EQRefIterator::Read()`：每次都调（它本身就是"每个外层行一次"）
- BKA 的 `MrrNextCallback()`：每拿一个 buffer 行调一次

调用顺序固定：`construct_lookup()`（填 `key_buff`）→ `impossible_null_ref()`（late NULLs filtering）→ `ha_index_read_map(..., HA_READ_KEY_EXACT)`。

#### `key_buff2`：eq_ref 单行缓存的"上次 key 快照"

全库只有 `EQRefIterator` 用它。用途是保存"上一次成功查找时的 key"，与新 key 比对来判断能否复用 `record[0]` 里缓存的那一行：

```cpp
  if (!read_row)
    // Last lookup found a row, copy its key to secondary buffer
    memcpy(m_ref->key_buff2, m_ref->key_buff, m_ref->key_length);
  m_ref->key_err = construct_lookup(thd(), table(), m_ref);
  if (!read_row &&
      memcmp(m_ref->key_buff2, m_ref->key_buff, m_ref->key_length) != 0)
    read_row = true;
```

注意顺序：必须在 `construct_lookup()` 覆写 `key_buff` **之前** memcpy。

### 8.4 ★ cond_guards：NOT IN 与"退化成全表扫描"

这是 `Index_lookup` 里最精妙的一块。

#### 机制

```cpp
  /*
    Array of pointers to trigger variables. ...
    The ref access can be used iff
      for each used key part i, (!cond_guards[i] || *cond_guards[i])
    ...
    If a table in a subquery has this it means that the table access
    will switch from ref access to table scan when the outer query
    produces a NULL value to be checked for in the subquery. This will
    be used by NOT IN subqueries and IN subqueries which need to distinguish
    NULL and FALSE, where ignore_unknown() is false.
  */
```

"可开关条件"就是 `Item_func_trig_cond`——一个**带守卫变量的谓词**：`val(P/g) = g ? val(P) : true`。守卫为 false 时该谓词恒真（等于没这个条件）。

#### 为什么需要它

`NULL IN (SELECT ...)` 的结果**既可能是 NULL 也可能是 FALSE**。IN→EXISTS 把 `<expr>` 下推成 `f2 = <expr>` 后，`<expr>` 为 NULL 时这个等式恒为 UNKNOWN，会漏掉"应该扫描全表的情形"。所以需要两件事同时发生：关掉下推的 WHERE 条件 + **把 ref 退化成全表扫描**。

判据是 `ignore_unknown()`（`= abort_on_null`）：调用方不关心 UNKNOWN 与 FALSE 的区别时，不注入 trig_cond，也就不需要 cond_guards。

#### 退化在哪实现：`AlternativeIterator`

计划侧（`QEP_TAB::access_path()` 末尾）创建两条路，让执行期二选一：

```cpp
  if (used_ref->cond_guards[key_part_idx] != nullptr) {
    AccessPath *table_scan_path = NewTableScanAccessPath(...);
    path = NewAlternativeAccessPath(thd, path, table_scan_path, used_ref);
    break;
  }
```

那段长注释还承认了一个未做的优化：

```
    Note that ideally, we'd plan a completely separate plan for the NULL case,
    as there might be e.g. a different index we could scan on, or even a
    different optimal join order. (Note, however, that for the case of multiple
    columns in the expression, we could get 2^N different plans.)
```

执行侧在 `Init()` 里选，并用 `m_last_iterator_inited` 避免反复切换：

```cpp
bool AlternativeIterator::Init() {
  m_iterator = m_source_iterator.get();
  for (bool *cond_guard : m_applicable_cond_guards) {
    if (!*cond_guard) { m_iterator = m_table_scan_iterator.get(); break; }
  }
  if (m_iterator != m_last_iterator_inited) {
    m_table->file->ha_index_or_rnd_end();   // 两条路扫描方式不同，要结束当前的
    m_last_iterator_inited = m_iterator;
  }
  return m_iterator->Init();
}
```

⚠️ 注意**检查频率**：`AlternativeIterator` 只在 `Init()` 检查（每个外层行一次），不是每行 `Read()`；但 WHERE 侧的"关闭"是**逐行**的（`Item_func_trig_cond::val_int()` 每次求值都读守卫变量）。

传统 EXPLAIN 里这显示为 **`Full scan on NULL key`**（Extra 列）。

#### 优化期的三个副作用

`has_guarded_conds()` 为真时会**禁用**三样东西（都是硬编码的短路）：

```cpp
if (tab->has_guarded_conds()) goto no_join_cache;        // 禁用 BKA
!has_guarded_conds() && ...                              // 禁用 ICP
if (ref->has_guarded_conds()) return nullptr;            // 禁用 cond 消除
```

### 8.5 null_rejecting 与 REF_OR_NULL

#### null_rejecting 位图

设置链条（源码自述）：`update_ref_and_keys()` 累积到 `Key_use` → `create_ref_for_key` 拷进 `Index_lookup` → `add_not_null_conds()` 加 `IS NOT NULL`。

判定规则（**只有 `<=>`、`IS NULL`、`IS NOT NULL` 三种谓词在 NULL 操作数上可能为真**）：

```cpp
  const bool null_rejecting = cond->functype() != Item_func::EQUAL_FUNC &&
                              cond->functype() != Item_func::ISNULL_FUNC &&
                              cond->functype() != Item_func::ISNOTNULL_FUNC;
```

三个用途：早期 NULL 过滤（优化期加 `IS NOT NULL`）、晚期 NULL 过滤（执行期 `impossible_null_ref()`）、**区分 eq_ref 与 ref**（唯一索引 + null-rejecting 才能判 eq_ref）。

#### null_ref_key：REF_OR_NULL 的两阶段

`null_ref_key` 指向该 keypart 槽位的**首字节**（就是 NULL 标志字节）：`*null_ref_key == 1` 时该 keypart 被解释为 SQL NULL。

⚠️ **不是"一次 lookup 同时找值和找 NULL"**，而是同一份 key buffer 上**做两次查找**：

```
Init():  *null_ref_key = false
阶段 1:  NULL 字节=0 → ha_index_read_map + ha_index_next_same 遍历精确匹配值的行
耗尽后:  *null_ref_key = true; m_reading_first_row = true; return Read();  // 递归
阶段 2:  NULL 字节=1 → 再查一次，找 NULL 行
```

⚠️ **限制：最多只能有一个 ref_or_null keypart**——因为 `null_ref_key` 只有一个字节、状态机只有一个布尔（`m_reading_first_row`），无法表达 2 个 keypart 的 4 种组合。源码注释直说：

```
      Moreover, the execution layer is limited to maximum one ref_or_null
      keypart, as Index_lookup::null_ref_key is only one byte.
```

第二个及以后带 `KEY_OPTIMIZE_REF_OR_NULL` 的 keyuse 会被直接丢弃。

### 8.6 keypart_hash（8.0 新增）

```cpp
  /*
    If non-nullptr, all the fields are hashed together through functions
    in store_key (with the result being put into this field), as opposed to
    being matched against individual fields in the associated KEY's key parts.
   */
```

即：不逐个 keypart 塞进 key buffer 比对，而是**把所有字段哈希合并成一个 8 字节值**，用这个单值做一次单 keypart 查找。切换点在 `FindKeyBufferAndMap()`：

```cpp
  if (ref->keypart_hash != nullptr) {
    return make_pair(pointer_cast<uchar *>(ref->keypart_hash), key_part_map{1});
  }
```

**⚠️ 关键澄清：它与 hash join 无关。** 唯一设置点是 `subselect_hash_sj_engine::setup()`，服务于**物化子查询 / semi-join 物化临时表**上的 index lookup。触发条件是临时表启用了 `hash_field`（唯一键太长 / keypart 太多 / INTERSECT·EXCEPT 等）。

代价是**必须后置过滤**——源码明说哈希不唯一（且截断会产生 false positive）：

```
    - because of truncation (if the outer column type's length is bigger than
      the inner column type's, index lookup will use a truncated outer
      value as search key, yielding false positives).
    - because the index is over hash_field and thus not unique.
```

反向 ref 不支持：`RefIterator<true>::Read()` 首行即 `assert(m_ref->keypart_hash == nullptr)`。

### 8.7 disable_cache 与 ICP：一个反直觉现象

```cpp
  /*
    true <=> disable the "cache" as doing lookup with the same key value may
    produce different results (because of Index Condition Pushdown)
  */
```

**被禁用的缓存不是 handler 层的**，而是 `EQRefIterator` 在 `record[0]` 里的"单行缓存"，其前提是"同样的 key ⇒ 同样的一行"。

**ICP 为什么打破这个前提**：被下推的索引条件**允许引用其他表的列**（`make_cond_for_index` 的 `other_tbls_ok`），而 ref/eq_ref 的 `other_tbls_ok` 恰恰是 true。于是——key 相同 ≠ 其他表的列值相同。例如 ref 键是 `t2.a = t1.a`，ICP 条件里还有 `t2.b > t1.c`，同一 key 用不同的 `t1.c` 再查一次，命中与否就变了。

所以源码选择"只要推了 ICP 就禁用缓存"：

```cpp
      /*
        Disable eq_ref's "lookup cache" if we've pushed down an index
        condition.
        TODO: This check happens to work on current ICP implementations, but
        there may exist a compliant implementation that will not work
        correctly with it. Sort this out when we stabilize the condition
        pushdown APIs.
      */
      if (idx_remainder_cond != idx_cond) {
        ref().disable_cache = true;
      }
```

⚠️ 那条 TODO 承认这只是**碰巧对当前 ICP 实现有效**，不是干净的设计。

禁用后：每次都真正执行 `ha_index_read_map()`，跳过 `key_buff2` 的 memcpy/memcmp，也不再复用旧行。

> 一致性佐证：只有"缓存未禁用"的 eq_ref 才需要 `AggregateIterator` 特别保护（注释：*"so that the cache in EQRefIterator is not disturbed by AggregateIterator's switching between groups"*）。

### 8.8 谁填、谁读，以及 EXPLAIN 两列的来源

**填充方**：`create_ref_for_key()`（旧优化器，由 `JOIN::init_ref_access()` 调用）；hypergraph 侧是 `ProposeRefAccess()`，复用同一对 helper（`init_ref` / `init_ref_part`）。

**读取方**：`RefIterator` / `RefOrNullIterator` / `EQRefIterator` / `read_const()` / `PushedJoinRefIterator`（NDB）／ BKA 的 `MultiRangeRowIterator`。

#### EXPLAIN `key_len` 列 = `key_length`

`key_length = Σ keypart.store_length`（含 NULL 标志字节与长度前缀），直接对应：

```cpp
if (tab->ref().key_parts)
  return explain_key_and_len_index(tab->ref().key, tab->ref().key_length,
                                   tab->ref().key_parts);
```

#### EXPLAIN `ref` 列 = `key_copy` 的"名字"

```cpp
  for (uint part_no = 0; part_no < key_parts; part_no++) {
    const store_key *const s_key = key_copy[part_no];
    if (s_key == nullptr) {
      if (fmt->entry()->col_ref.push_back(STORE_KEY_CONST_NAME)) return true;  // "const"
    } else if (fmt->entry()->col_ref.push_back(s_key->name())) return true;
  }
```

而 `store_key::name()` 的规则是：

| 情况 | 显示 |
|---|---|
| `Item::FIELD_ITEM` | `item->full_name()`（列名） |
| 其他 | `"func"` |
| `key_copy[i] == nullptr` | `"const"` |

★ **这就是 8.3 的可见证据：EXPLAIN 显示 `const` 的那一列，正是优化期已固化、执行期完全不参与 `construct_lookup()` 的 keypart。**

### 8.9 限制与遗留 TODO

| 限制 | 说明 |
|---|---|
| **keypart 上限 16** | `MAX_REF_PARTS{16}`；`assert(keyparts <= MAX_REF_PARTS)` |
| **最多一个 ref_or_null keypart** | `null_ref_key` 只有一个字节（8.5） |
| **hypergraph 不支持 REF_OR_NULL** | TODO：`"When we get support for REF_OR_NULL, set null_ref_key = key_buff here"` |
| **hypergraph 的 cond_guard 恒为 nullptr** | 即 8.4 那套机制只在旧优化器路径生效 |
| **反向 ref 三条限制** | 不支持 `keypart_hash`、不支持 ICP（assert `pushed_idx_cond == nullptr`）、且 `push_index_cond` 对 `reversed_access` 直接 return（`@todo: historical limitation, lift it!`） |
| **旧名清理未完成** | `@todo Remove any references to the old name: it used to be called TABLE_REF` |

一个容易忽略的点：**`Index_lookup` 是执行期可变状态**（每次外层行都会重写 `key_buff`）。`SortingIterator` 的注释因此把它当作"脏数据源"——重复 `Init()` 时必须**重新排序**而不能只回绕：

```
  // NOTE: If you call Init() again, SortingIterator will actually
  // do a _new sort_, not just rewind the iterator. This is because a
  // Index_lookup we depend on may have changed so the produced record
  // set could be different from what we had last time.
```

---

## 九、最终落地与 EXPLAIN type 映射

### 9.1 JT_* → AccessPath::Type（`QEP_TAB::access_path()`，`sql_executor.cc:3701`）

| JT_* | AccessPath::Type | 迭代器 |
|---|---|---|
| `JT_REF` | `REF` | `RefIterator` |
| `JT_REF_OR_NULL` | `REF_OR_NULL` | `RefOrNullIterator`（两次查找：值 + NULL） |
| `JT_EQ_REF` | `EQ_REF` | `EQRefIterator`（找到就停） |
| `JT_CONST` / `JT_SYSTEM` | `CONST_TABLE` | `ConstIterator`（**直接返回优化期读好的 record[0]**） |
| `JT_INDEX_SCAN` | `INDEX_SCAN` | `IndexScanIterator` |
| `JT_ALL` / `JT_RANGE` / `JT_INDEX_MERGE` | 走 `create_table_access_path`（`:4690`） | TABLE_SCAN / 各 range 类型 |

`create_table_access_path` 只有三条路：有 range_scan 就复用它、是 CTE 递归引用就 `FOLLOW_TAIL`、否则 `TABLE_SCAN`。

**★ `QEP_TAB::access_path()` 的 switch 全文**（★ 行号与文档一致：`sql/sql_executor.cc:3701`）：

```cpp
AccessPath *QEP_TAB::access_path() {
  assert(!m_reversed_access || type() == JT_REF || type() == JT_INDEX_SCAN);
  Index_lookup *used_ref = nullptr;   /* ★ 只有会用到 ref 的类型才置非空 */
  AccessPath *path = nullptr;

  switch (type()) {
    case JT_REF:
      path = NewRefAccessPath(thd, table(), &ref(), use_order(),
                              m_reversed_access, true);
      used_ref = &ref();  break;
    case JT_REF_OR_NULL:
      path = NewRefOrNullAccessPath(thd, table(), &ref(), use_order(), true);
      used_ref = &ref();  break;
    case JT_CONST:
      path = NewConstTableAccessPath(thd, table(), &ref(), true);
      break;                          /* ★ const 表不设 used_ref —— 不可能被 cond_guard 关掉 */
    case JT_EQ_REF:
      path = NewEQRefAccessPath(thd, table(), &ref(), true);
      used_ref = &ref();  break;
    case JT_FT:
      path = NewFullTextSearchAccessPath(thd, table(), &ref(), ft_func(),
                                         use_order(), ..., true);
      used_ref = &ref();  break;
    case JT_INDEX_SCAN:
      path = NewIndexScanAccessPath(thd, table(), index(), use_order(),
                                    m_reversed_access, true);
      break;                          /* ★ 索引扫描不用 ref */
    case JT_ALL:
    case JT_RANGE:
    case JT_INDEX_MERGE:
      if (using_dynamic_range)
        path = NewDynamicIndexRangeScanAccessPath(thd, table(), this, true);
      else
        path = create_table_access_path(thd, table(), range_scan(),
                                        table_ref, position(), true);
      break;
    default: assert(false); break;
  }
  if (position() != nullptr)
    SetCostOnTableAccessPath(*thd->cost_model(), position(), false, path);
  /* ... cond_guards → ALTERNATIVE 包装（见 8.4）... */
  /* ... filesort → SORT 包装 ... */
  return path;
}
```

**★★ `AccessPath::Type` 枚举全文**（⚠️ 是 **`enum Type : uint8_t`**，**不是** `enum class Type`；`sql/join_optimizer/access_path.h:209-266`）：

```cpp
  enum Type : uint8_t {
    // Basic access paths (no children)
    TABLE_SCAN, INDEX_SCAN, REF, REF_OR_NULL, EQ_REF, PUSHED_JOIN_REF,
    FULL_TEXT_SEARCH, CONST_TABLE, MRR, FOLLOW_TAIL,
    INDEX_RANGE_SCAN, INDEX_MERGE, ROWID_INTERSECTION, ROWID_UNION,
    INDEX_SKIP_SCAN, GROUP_INDEX_SKIP_SCAN, DYNAMIC_INDEX_RANGE_SCAN,

    // 不对应具体表
    TABLE_VALUE_CONSTRUCTOR, FAKE_SINGLE_ROW, ZERO_ROWS, ZERO_ROWS_AGGREGATED,
    MATERIALIZED_TABLE_FUNCTION, UNQUALIFIED_COUNT,

    // Joins
    NESTED_LOOP_JOIN, NESTED_LOOP_SEMIJOIN_WITH_DUPLICATE_REMOVAL,
    BKA_JOIN, HASH_JOIN,

    // Composite
    FILTER, SORT, AGGREGATE, TEMPTABLE_AGGREGATE, LIMIT_OFFSET, STREAM,
    MATERIALIZE, MATERIALIZE_INFORMATION_SCHEMA_TABLE, APPEND, WINDOW,
    WEEDOUT, REMOVE_DUPLICATES, REMOVE_DUPLICATES_ON_INDEX,
    ALTERNATIVE,            /* ★ 8.4 的 AlternativeIterator 就是它 */
    CACHE_INVALIDATOR,

    // 修改表
    DELETE_ROWS, UPDATE_ROWS,
  } type;
```

> ★ 注意 `AccessPath` 的粒度**远细于** `JT_*`（11 个）：它把 join 算法、物化、排序、去重、窗口全列为一等类型，而 `JT_*` 只描述"单表怎么取数"。这正是 8.0 引入 AccessPath 的动机——**经典优化器只输出 `JT_*`，hypergraph 需要更丰富的表达**。

### 9.2 EXPLAIN type 映射（`opt_explain.cc:115`）

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


---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `best_access_path()` / `find_best_ref()` / `calculate_scan_cost()` 在 `sql/sql_optimizer.cc` | **全在 `sql/sql_planner.cc`**（8.0 已把 `Optimize_table_order` 整体搬过去）。`sql_optimizer.cc` 里只剩几处注释提到它们 |
| `calculate_scan_cost()` 在 `sql_optimizer.cc:770` | 行号巧合对了，**文件错了** → `sql/sql_planner.cc:770-903` |
| `POSITION` 在 `sql_planner.h` | 定义在 **`sql/sql_select.h:352-587`**；`sql_planner.h` 只有前向声明。且注释明说 *"has to stay a POD, because it is memcpy'd in many places"* |
| `set_prefix_join_cost()` / `set_prefix_cost()` 是独立函数 | 是 `POSITION` 的**内联成员函数** |
| 索引等级枚举是 `FT_KEYPART` / `MAX_KEYPART` / `MAX_REF_PARTS` | 等级枚举是函数内局部 **`enum idx_type { CLUSTERED_PK, UNIQUE, NOT_UNIQUE, FULLTEXT }`**；`FT_KEYPART` 是宏 `= MAX_REF_PARTS+10 = 26`；`MAX_REF_PARTS = 16`；**`MAX_KEYPART` 不存在** |
| `find_best_ref()` 产出 `JT_EQ_REF` | **不产出**。它只产 `idx_type` + `Key_use*`；真正的 `JT_EQ_REF`/`JT_REF`/`JT_REF_OR_NULL` 在 **`create_ref_for_key()`（`sql_select.cc:2515-2536`）** |
| fanout 三来源含 `COND_FILTER_*` | **不含**。真实三来源：`quick_rows[key]`（range 估计）/ `records_per_key()` / 经验公式。`COND_FILTER_*` 属于 **filter_effect**，由 `calculate_condition_filter()` 算，与 fanout 是两回事 |
| `covering_keys` 在 `find_best_ref()` 里起作用 | **函数体内没有**。它在 `find_cost_for_ref()`（`:151`）与 `best_access_path()` 里起作用 |
| `set_join_buffer_properties()` 调用 `find_best_ref` | **该函数不存在**（全仓库只有一句注释提到）。唯一调用点是 `best_access_path()` `:1025` |
| `find_best_ref()` 里有 `const_cond` 标志 | **不存在**（只有注释里的 `ref_const_cond` 概念）。`const_cond_filter` 在 `calculate_scan_cost()` 里 |
| 二次 range 重估在 `JOIN::adjust_access_methods()` | **半错**。该函数只做 `ALL→INDEX_SCAN` 与 `REF→RANGE` 两个启发式替换；recheck 整段在 **`make_join_query_block()`** 的 `:9753-9946` |
| `enum enum_recheck_reason`（`RECHECK_NONE`/`RECHECK_RANGE`/`RECHECK_INDEX_MERGE`） | **不存在**（5.x 的写法）。是 `make_join_query_block()` 内的**匿名局部 enum** `{DONT_RECHECK, NOT_FIRST_TABLE, LOW_LIMIT}` |
| `best_access_path()` 有 3 条启发式短路 | **4 条**，且是一条 `if / else if ×3 / else` 链——**前 3 条只写 trace 就跳过扫描估算，第 4 条 `else` 才真正算 scan** |
| `best_access_path()` 里有 `straight_join` / `found_const` 短路 | **两个都不存在**。`straight_join` 是 `choose_table_order()` 的局部变量；`found_const` 这个标识符全 `sql/` 下不存在（对应物是 `JOIN::found_const_table_map`） |
| `can_switch_from_ref_to_range()` 有 3 个调用点 | **只有 2 个**：`:2621`（`recheck_range=true`）与 `:2991`（`recheck_range=false`） |
| `test_quick_select()` 返回行数 | 返回 **-1 / 0 / 1** 三态（impossible / 不能用 / 成功）。行数是 `range_scan->num_output_rows()` |
| 写回 POSITION 在 `:1159-1224` | **半错**。1159-1188 只是改写**局部变量**；真正的 `pos->xxx =` 只有 **1217-1224 八行** |
| `class Index_lookup` 在 `sql_executor.h` | 是 **`struct`**，在 **`sql/sql_opt_exec_shared.h:67-176`** |
| `Index_lookup` 有 `key_copy()` 方法 | **同名异物**：① 自由函数 `key_copy()`（`sql/key.cc:136`，row format → key format，**与 ref 无关**）；② 成员 `store_key **key_copy`（每 keypart 的取/存值器数组） |
| `store_key_field` / `store_key_item` / `store_key_null` | **都不存在**（0 命中）。实际只有 4 个：`store_key` / `store_key_hash_item` / `store_key_const_item` / `store_key_json_item` |
| `null_rejecting` 由 `calc_length_and_keyparts()` 计算 | **不是**。那个函数只算 `length/keyparts/dep_map/maybe_null`。真正置位点是 **`init_ref_part()`（`sql_select.cc:2344`）** |
| `keypart_hash` 是"ref 缓存优化" | **不是**。它只在**物化子查询 + `tmp_table->hash_field`（超长键的唯一约束哈希列）**时出现，让查找绕过逐 keypart 匹配直接用 8 字节哈希。普通 ref 缓存是 `EQRefIterator` 的 `key_buff2` |
| `disable_cache` 是独立的 eq_ref 缓存开关 | 默认 `false`，但**唯一设置点是 ICP 下推成功**（`sql_select.cc:3057`）——ICP 会**强制关掉** eq_ref 的一行缓存 |
| `key_buff2` 是 `key_buff + key_length` | **不是**（注释是历史遗留误导）。`init_ref()` **独立分配了两块** `ALIGN_SIZE(length)` 内存 |
| `AccessPath::Type` 是 `enum class Type` | 是 **`enum Type : uint8_t`**（无 `class`） |
| `join_type_str` 在 `opt_explain.h` | 声明在 `opt_explain.h:79`，**定义在 `opt_explain.cc:115-117`**。`JT_*` 枚举在 **`sql/sql_opt_exec_shared.h:186-233`** |
| `store_key` 实现在 `sql/field.cc` | **不在**。全在 `sql/sql_select.{h,cc}`；`field.cc` 只提供被它调用的 `Field::new_key_field()` |
| `keypart_map` 是 `uint` | 是 `typedef ulong key_part_map;`（`include/my_base.h:1008`） |

**★ 四个"结构性事实"**

1. **"高等级无条件胜出"是三行代码，也是整个 ref 决策的灵魂**。`find_best_ref()` 末尾：等级不同时 `new_candidate = true`（**无视代价**），等级相同时才比 `cur_ref_cost`。等级由 `enum idx_type` 的顺序定义（`CLUSTERED_PK < UNIQUE < NOT_UNIQUE < FULLTEXT`），而升级为 `UNIQUE` 需要两个前提：**全部 keypart 被等值覆盖** + **索引唯一（或全 keypart NULL-rejecting）**。所以"eq_ref 优先于 ref"不是口号，就是"枚举值比较"这一行。而 `CLUSTERED_PK` 拿到后直接 `break`——**连剩下的索引都不看了**。

2. **`find_best_ref()` 与 `create_ref_for_key()` 用不同的 `flags`，这是"规划期算成 eq_ref、建 ref 时退化"的机制来源**。前者用 `keyinfo->flags`（**不含**索引扩展位），后者用 `actual_key_flags(keyinfo)`（**受 `use_index_extensions` 影响**）。同理 `actual_key_parts()` 与 `user_defined_key_parts` 在 `find_best_ref` 内部就有两个不同用途的口径——"能不能算全键覆盖"用前者，"无 rec_per_key 时怎么估 fanout"用后者。

3. **`pos->read_cost` 不含 `row_evaluate_cost`，`filter_effect` 最后才乘**。写回 POSITION 的 8 行里只写了 `rows_fetched / read_cost / filter_effect / key`，`prefix_cost / prefix_rowcount` 是调用方 `set_prefix_join_cost()` 加 CPU 代价后推出的。而 `set_prefix_join_cost()` 里 `prefix_rowcount *= filter_effect` 在**最后一行**——`row_evaluate_cost` 用的是**未乘** filter_effect 的值，精确对应"取出来的行都要 evaluate，通过过滤的才进入下一级扇出"。**这三处口径是手工算代价时最容易错的地方**。

4. **ICP 与 eq_ref 缓存的冲突是一个真实的两难**。`disable_cache` 的唯一设置点是 ICP 下推成功（`idx_remainder_cond != idx_cond`）。注释说明了原因：ICP 把条件下沉到引擎后，同一 key 值在不同外层行下**可能返回不同结果集**，复用"上一行缓存"就会错。所以**ICP 生效 ⇒ eq_ref 失去"key 不变就不重读"的优化**——这是优化器里少见的"一个优化关掉另一个优化"的显式权衡。

> 注：`sql/sql_planner.cc`、`sql/sql_select.cc`、`sql/sql_opt_exec_shared.h` 等文件中的部分 `/* 【注】... */` 与 `★` 注释为本工作区人工批注，非 Oracle 上游原文。

---

## 参考

**论文**
- **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979)** —— 访问路径（access path）选择与代价比较

**官方文档**
- *MySQL 8.0 Reference Manual → How MySQL Chooses Indexes*、`Index Hints`


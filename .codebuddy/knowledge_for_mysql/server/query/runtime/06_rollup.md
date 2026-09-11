# 06 ROLLUP：GROUP BY 的多层汇总

> `GROUP BY ... WITH ROLLUP` 在普通分组之上额外输出各层次前缀汇总行（如按 `(year,country,product)` 分组时，还输出 `(year,country)`、`(year)`、全局汇总）。本篇讲 8.0 重构后的实现：**多层聚合封装 + 分组列 NULL 化 + 流式聚集**。

## 目录

- [一、语义与 level 约定](#一语义与-level-约定)
- [二、多层聚合：Item_rollup_sum_switcher](#二多层聚合item_rollup_sum_switcher)
- [三、分组列封装：Item_rollup_group_item](#三分组列封装item_rollup_group_item)
- [四、流式聚集：AggregateIterator 状态机](#四流式聚集aggregateiterator-状态机)
- [五、rollup 对优化器的影响](#五rollup-对优化器的影响)
- [六、深潜补充：GROUPING()、resolver 后处理与一条纠错](#六深潜补充grouping-resolver-后处理与一条纠错)

---

## 一、语义与 level 约定

### 1.1 level 编号（item_func.h:1589 注释）

`GROUP BY a,b WITH ROLLUP` 有 `send_group_parts = 2` 个分组列，rollup level 语义是"**当前行还有多少个分组列是活跃的（未变成 NULL）**"：

```
     a     b      rollup level
     1     1      2          ← 明细行（a、b 都真实）
     1     2      2
     1     NULL   1          ← 只按 a 汇总，b 置 NULL
     2     1      2
     2     NULL   1
     NULL  NULL   0          ← grand total，全 NULL
```

- level = 2：明细；level = 1：按 `a` 汇总（`b` 置 NULL）；level = 0：grand total
- 每个分组列有一个 **min_rollup_level**（等于它在 GROUP BY 列表中的下标）：`a` 的 min level 是 0，`b` 的是 1
- 当 `level <= min_rollup_level` 时，该列置 NULL

### 1.2 olap 标志与语法

`sql/parser_yystype.h:179`：`enum olap_type { UNSPECIFIED_OLAP_TYPE, ROLLUP_TYPE }`。语法 `sql_yacc.yy:12545` 把 `WITH ROLLUP` 归约成 `ROLLUP_TYPE`，`parse_tree_nodes.cc:274` 赋给 `select->olap`。全优化器/执行器判断"是否 rollup"就是 `query_block->olap == ROLLUP_TYPE`。

### 1.3 resolve_rollup（prepare 阶段，sql_resolver.cc:5063）

```cpp
bool Query_block::resolve_rollup(THD *thd) {
  uint send_group_parts = group_list_size();
  for (auto it = fields.begin(); it != fields.end(); ++it) {
    Item *item = *it;
    Item *new_item;
    if (Item_sum *item_sum; item->type() == Item::SUM_FUNC_ITEM &&
                            !item->const_item() &&
                            (item_sum = down_cast<Item_sum *>(item),
                             item_sum->aggr_query_block == this)) {
      new_item = create_rollup_switcher(thd, this, item_sum, send_group_parts);  // 顶层聚合
    } else {
      new_item = resolve_rollup_item(thd, item);   // 普通列/表达式
    }
    *it = new_item;
  }
}
```

**两类替换**：
- 本层**顶层聚合函数** → `create_rollup_switcher`（每 level 复制一份聚合，见第二节）
- 其他项 → `resolve_rollup_item`（把匹配分组列的子表达式替换成 `Item_rollup_group_item`，见第三节）

`create_rollup_switcher`（:5029）：

```cpp
List<Item> alternatives;
alternatives.push_back(item);                       // level 0 用原聚合
for (int level = 0; level < send_group_parts; ++level) {
  Item_sum *new_item = down_cast<Item_sum *>(item->copy_or_same(thd));
  new_item->make_unique();
  alternatives.push_back(new_item);
}
Item_rollup_sum_switcher *new_item = new Item_rollup_sum_switcher(&alternatives);
query_block->rollup_sums.push_back(new_item);
```

原聚合 + `send_group_parts` 份拷贝 = `send_group_parts + 1` 份（对应 level 0..send_group_parts）。

---

## 二、多层聚合：Item_rollup_sum_switcher

### 2.1 结构（item_sum.h:2708）

```cpp
class Item_rollup_sum_switcher final : public Item_sum {
  explicit Item_rollup_sum_switcher(List<Item> *sum_func_per_level)
      : Item_sum((*sum_func_per_level)[0]), m_num_levels(sum_func_per_level->size()) {
    args = ArrayAlloc<Item *>(sum_func_per_level->size());
    int i = 0;
    for (Item &item : *sum_func_per_level) args[i++] = &item;
    set_distinct(master()->has_with_distinct());   // ★ DISTINCT 标志也复制
    set_data_type_from_item(master());
  }
  void set_current_rollup_level(int level) { m_current_rollup_level = level; }
  inline Item_sum *master() const { return child(0); }
 private:
  inline Item_sum *child(size_t i) const { return down_cast<Item_sum *>(args[i]); }
  const int m_num_levels;
  int m_current_rollup_level = INT_MAX;
};
```

- 继承 `Item_sum`，用 `args[]` 持有 `m_num_levels` 个 `Item_sum*`（**每个 rollup level 一份聚合函数**）
- `master()` = `child(0)` = level 0 那份，类型/属性从它继承
- `m_current_rollup_level` 记录当前 level，求值时决定读哪个子聚合

### 2.2 求值：只切下标，零拷贝

```cpp
double Item_rollup_sum_switcher::val_real() {
  double res = current_arg()->val_real();   // args[m_current_rollup_level]
  if ((null_value = current_arg()->null_value)) return 0.0;
  return res;
}
```

所有 `val_*`/`is_null` 统一委托给 `args[m_current_rollup_level]`——**读值阶段不复制、不搬运，只改一个下标**。

### 2.3 按 level 增量聚合

```cpp
bool Item_rollup_sum_switcher::reset_and_add_for_rollup(int last_unchanged_group_item_idx) {
  for (int i = 0; i < m_num_levels; ++i) {
    if (i >= last_unchanged_group_item_idx)
      child(i)->reset_and_add();      // 该层分组键变了：清零重来
    else
      child(i)->aggregator_add();     // 该层仍属同一粗汇总：继续累加
  }
  return false;
}

inline bool aggregator_add_all() {
  for (int i = 0; i < m_num_levels; ++i) child(i)->aggregator_add();  // 同组：全累加
}
```

- `last_unchanged_group_item_idx` = "上一个组和当前组中**最后一个没变的分组列下标 + 1**"
- 例：`GROUP BY a,b,c` 从 `(1,2,3)` 切到 `(1,2,9)`：`first_changed_idx=2`（c 变）→ level 0/1/2 继续加，level 3（明细）重置
- 从 `(1,2,3)` 切到 `(1,8,9)`：level 0/1 继续加，level 2/3 重置

### 2.4 为什么每 level 复制一份

1. **状态隔离**：SUM/COUNT/AVG/MIN/MAX 各有内部累积状态，ROLLUP 需同时维护 `()`、`(a)`、`(a,b)`、明细 4 个并行聚合，必须各自独立
2. **DISTINCT 正确性**：`COUNT(DISTINCT x)` 每 level 的去重集合不同，必须每 level 一个 `Distinct` aggregator（`set_distinct` 已复制，`:2730`）
3. **求值零拷贝**：读值只切下标，不需要 5.7 的 `Item_copy` 结果搬移

---

## 三、分组列封装：Item_rollup_group_item

### 3.1 结构与 val_*（item_func.h:1614 / item_func.cc:4018）

```cpp
class Item_rollup_group_item final : public Item_func {
  Item_rollup_group_item(int min_rollup_level, Item *inner_item)
      : Item_func(inner_item), m_min_rollup_level(min_rollup_level) {
    set_nullable(true);               // rollup 行会产生 NULL
    set_rollup_expr();
  }
  bool rollup_null() const { return m_current_rollup_level <= m_min_rollup_level; }
  void set_current_rollup_level(int level) { m_current_rollup_level = level; }
 private:
  const int m_min_rollup_level;       // 构造时固定 = GROUP BY 下标
  int m_current_rollup_level = INT_MAX;  // 运行期广播
};
```

```cpp
longlong Item_rollup_group_item::val_int() {
  if (rollup_null()) { null_value = true; return 0; }   // 该层置 NULL
  longlong res = args[0]->val_int();
  if ((null_value = args[0]->null_value)) return 0;
  return res;
}
```

- `rollup_null()` 是唯一判定：`m_current_rollup_level <= m_min_rollup_level` 时置 NULL
- `m_current_rollup_level` 初值 `INT_MAX` = "未被激活时永不置 NULL"

### 3.2 min_rollup_level 的确定（find_in_group_list，sql_resolver.cc:4738）

`rollup_level` 就是该列在 GROUP BY 列表中的**下标**：`GROUP BY a,b` 里 `a` 下标 0 → min level 0，`b` 下标 1 → min level 1。匹配用 `eq(..., false)` 非二进制比较，优先别名精确匹配。

`wrap_grouped_expressions_for_rollup`（:4787）在表达式树里递归查找匹配分组列的叶子，替换成 `Item_rollup_group_item`；若参数是 `GROUPING()` 但列不在 GROUP BY 里，报 `ER_FIELD_IN_GROUPING_NOT_GROUP_BY`。

替换后 `refresh_comparators_after_rollup`（:4944）重刷比较器——分组列现在可能为 NULL，之前被判为常量的比较器缓存已失效。

---

## 四、流式聚集：AggregateIterator 状态机

### 4.1 为什么必须流式

ROLLUP 需要"**按分组键有序**"的输入，才能通过相邻行比较识别组边界、并知道**第一列在哪一层开始变化**（`first_changed_idx`）。临时表聚集（hash/index）是乱序输入下的散列表合并，没有"有序前缀"概念，无法自然产生"前缀汇总行"。

依据：`optimize_rollup`（`sql_optimizer.cc:11347`）强制 `allow_group_via_temp_table = false`，所以 rollup **只能走 `AggregateIterator` 流式聚集**，不能走 `TemptableAggregateIterator`。

### 4.2 状态机（composite_iterators.cc:238）

```cpp
enum {
  READING_FIRST_ROW,          // 读第一行
  LAST_ROW_STARTED_NEW_GROUP, // 刚读到新组首行，处理旧组
  OUTPUTTING_ROLLUP_ROWS,     // 输出各层次汇总行
  DONE_OUTPUTTING_ROWS
} m_state;
```

**(1) LAST_ROW_STARTED_NEW_GROUP**（:292）核心：

```cpp
SetRollupLevel(m_join->send_group_parts);   // 回到明细层重算旧组首行
swap(m_first_row_this_group, m_first_row_next_group);
LoadIntoTableBuffers(... m_first_row_this_group.ptr());

for (Item_sum **item = m_join->sum_funcs; *item != nullptr; ++item) {
  if (m_rollup)
    down_cast<Item_rollup_sum_switcher *>(*item)
        ->reset_and_add_for_rollup(m_last_unchanged_group_item_idx);
  ...
}

for (;;) {
  int err = m_source->Read();
  if (err == -1) {   // EOF
    if (m_rollup && m_join->send_group_parts > 0) {
      m_last_unchanged_group_item_idx = 0;
      m_state = OUTPUTTING_ROLLUP_ROWS;   // 还要输出各级汇总 + grand total
    }
    return 0;
  }
  int first_changed_idx = update_item_cache_if_changed(m_join->group_fields);
  if (first_changed_idx >= 0) {   // 分组变了
    StoreFromTableBuffers(... m_first_row_next_group);   // 暂存新组首行
    LoadIntoTableBuffers(... m_first_row_this_group.ptr());  // 恢复旧组首行
    if (m_rollup) {
      m_last_unchanged_group_item_idx = first_changed_idx + 1;
      if (first_changed_idx < m_join->send_group_parts - 1)
        m_state = OUTPUTTING_ROLLUP_ROWS;    // 变了非最后一列 → 输出低层汇总
      else
        m_state = LAST_ROW_STARTED_NEW_GROUP;
    }
    return 0;
  }
  // 同组：所有 level 聚合器累加
  for (Item_sum **item = m_join->sum_funcs; *item != nullptr; ++item)
    if (m_rollup) down_cast<Item_rollup_sum_switcher *>(*item)->aggregator_add_all();
    ...
}
```

**(2) OUTPUTTING_ROLLUP_ROWS**（:411）：

```cpp
SetRollupLevel(m_current_rollup_position - 1);   // 每次递减一层
if (m_current_rollup_position <= m_last_unchanged_group_item_idx) {
  if (m_seen_eof) m_state = DONE_OUTPUTTING_ROWS;
  else m_state = LAST_ROW_STARTED_NEW_GROUP;
}
return 0;
```

**输出顺序**（以 `GROUP BY a,b` 为例）：明细行（level 2）→ `(a, NULL)` 汇总（level 1）→ `(NULL, NULL)` grand total（level 0）。靠 `m_current_rollup_position` 从 `send_group_parts` 向下递减逐个产生。

### 4.3 分组变化检测：Cached_item

`update_item_cache_if_changed`（`sql_executor.cc:4147`）逆序遍历所有 `Cached_item`（每个对应一个分组列），`cmp()` 比较当前值 vs 缓存值，返回**最靠前发生变化的分组列下标**：

```cpp
bool Cached_item_str::cmp() {
  String *res = item->val_str(&tmp_value);
  if (item->null_value) {
    if (null_value) return false;   // 都 NULL：未变（NULL==NULL 视为同组）
    null_value = true; return true;
  } else if (null_value || sortcmp(&value, res, item->collation.collation) != 0) {
    null_value = false; value.copy(*res); return true;   // 记住新值
  }
  return false;
}
```

### 4.4 SetRollupLevel：广播 level

```cpp
void AggregateIterator::SetRollupLevel(int level) {
  if (m_rollup && m_current_rollup_position != level) {
    m_current_rollup_position = level;
    for (Item_rollup_group_item *item : m_join->rollup_group_items)
      item->set_current_rollup_level(level);       // 决定哪列置 NULL
    for (Item_rollup_sum_switcher *item : m_join->rollup_sums)
      item->set_current_rollup_level(level);       // 决定读哪个子聚合
  }
}
```

---

## 五、rollup 对优化器的影响

| 限制 | 位置 | 原因 |
|------|------|------|
| 禁 Loose Index Scan | `group_index_skip_scan_plan.cc:322` | rollup 需按 level 输出，LIS 无法配合 |
| 禁索引化 `AGGFN(DISTINCT)` | `sql_optimizer.cc:8048` | `is_indexed_agg_distinct` 返回 false |
| HAVING 不能下推 | `sql_derived.cc:1316` | rollup 会给分组列制造 NULL，下推改语义 |
| ORDER BY 不能被 GROUP BY 前缀消去 | `sql_optimizer.cc:1629` | rollup 打乱顺序 |
| rollup + DISTINCT/窗口/ORDER 需先物化 | `sql_optimizer.cc:975` | `need_tmp_before_win = true` |
| GROUP BY 常量列不消除 | `sql_optimizer.cc:1601` | rollup 每 level 生成独立行，结构须保留 |

**DISTINCT 语义**：`can_skip_distinct()`（`sql_lex.h:1462`）rollup 时返回 false，保留 DISTINCT 去重；`COUNT(DISTINCT x)` 每 level 各自去重（`set_distinct` 已复制）。

**8.0 重构 vs 5.7**：`Item_rollup_sum_switcher` 取代旧的 `Item_copy` 方案——去掉结果搬移（只切下标）、聚合状态真正隔离（每 level 一份 Aggregator）、group item 对象化（`Item_rollup_group_item` + min/current level）。

---


## 六、深潜补充：GROUPING()、resolver 后处理与一条纠错

### 6.1 GROUPING() 函数实现（`Item_func_grouping`）

`item_sum.h:2694-2706`。`fix_fields`（`item_sum.cc:6172`）的三条约束：禁止位置参数（`:6178-1183`）、**最多 64 个参数**（位掩码，`:6194-6197`）、必须 `olap != UNSPECIFIED_OLAP_TYPE` 且不在 WHERE/JOIN 条件（`:6205-6210`）。

**核心求值**（`val_int`，`:6227-6236`）：

```cpp
if (has_rollup_result(real_item))
  result += 1ULL << (arg_count - (i + 1));   // ★ 第一个参数对应最高位
```

`has_rollup_result`（`sql_executor.cc:308-330`）：递归穿过 CACHE_ITEM/FUNC_ITEM/COND_ITEM，判据 `is_rollup_group_wrapper(item) && item->rollup_null()`。

**防常量折叠**：`used_tables_cache |= all_tables_map()`（`item_sum.cc:6188`、`:6298`）——否则优化器会把它当常量提前算掉（而它的值依赖当前 rollup level，是运行期才定的）。

### 6.2 SetRollupLevel 的短路与 Init 技巧

`composite_iterators.cc:444` 有 `m_current_rollup_position != level` **短路**——所以 `Init()` 里必须先：

```cpp
m_current_rollup_position = -1;      // :194
SetRollupLevel(INT_MAX);             // :195 用不可能的值强制"变化"，让后续广播真正生效
```

否则后续 `SetRollupLevel(send_group_parts)` 会因为"值没变"被短路跳过，rollup 层切换失效。

另：`READING_FIRST_ROW → LAST_ROW_STARTED_NEW_GROUP` 是 **`[[fallthrough]]`**（`:290`），不是 break。

### 6.3 resolve_rollup_item 的后处理（`sql_resolver.cc:4974-5027`）

替换后：`update_used_tables()`（`:4997`）→ 用 walker 把整树标 nullable，**但在 `ROLLUP_SUM_SWITCHER_FUNC` 处 stop**（`:5008-5024`，避免递归进 switcher 内层）。

两个配套动作：
- `mark_item_as_maybe_null_if_rollup_item`（`:4899-4910`）：HAVING 解析**前**把 GROUP BY 列标 nullable，防止 `IS NULL` 被常量折叠掉
- `refresh_comparators_after_rollup`（`:4944-4962`）：对 GE/GT/LT/LE/EQ/NE/EQUAL 重调 `set_cmp_func()`（因为列的可空性变了，比较函数要重选）
- `fulltext_uses_rollup_column`（`:5094-5123`）：MATCH() 参数含 rollup 列 → 拒绝

### 6.4 hypergraph 优化器下的 rollup

`finalize_plan.cc` 的 `CollectItemsWithoutRollup`（`:112`、`:165-166`）：物化时 rollup item 被剔除，但需把其**非 rollup 子项**加回 `items_to_copy`（否则物化表缺列）；替换走 `is_rollup_group_wrapper → unwrap_rollup_group`（`:420-421`）；`path->sort().unwrap_rollup`（`:451`）。

### 6.5 ⚠️ 纠错：不存在"TemptableAggregateIterator 的 rollup 三阶段"

如果你的资料来源提到 "rollup 在 TemptableAggregateIterator 里有 output/input/state 三阶段"——**本代码树没有**：

- `sql/iterators/` 下**无** `rolling_*` 文件，`sql/` 下**无** `rollup.cc`
- `TemptableAggregateIterator`（`composite_iterators.cc:1658/1703/1789/1890`）**没有任何 rollup 分支**

**根本原因**：`JOIN::optimize_rollup()`（`sql_optimizer.cc:11347-11348`）强制 `tmp_table_param.allow_group_via_temp_table = false`——**带 rollup 时禁止走临时表聚合**，所以 rollup 只能走流式 `AggregateIterator`（第四节的四态机），不可能进 TemptableAggregateIterator。

### 6.6 Item_rollup_group_item::used_tables()

`item_func.h:1635-1643`：内层是**常量**时要 `| RAND_TABLE_BIT`，防止被当作常量折叠（rollup 下"常量"列在高层会被 NULL 化，不是真常量）。

## 参考

**内核月报**
- **2024/05《MySQL 深潜 - 重构后的 ROLLUP 实现》** —— `Item_rollup_sum_switcher` 多层聚合 + 流式聚集（本文的直接来源）

**官方文档**
- *MySQL 8.0 Reference Manual → GROUP BY Modifiers → WITH ROLLUP*
- *MySQL 8.0 Reference Manual → The GROUPING() Function*

**相关文档**
- 普通 GROUP BY 的临时表聚集见 [`01_filesort_and_temptable.md`](01_filesort_and_temptable.md)
- 窗口函数（同为聚合家族）见 [`04_window_function.md`](04_window_function.md)
- 计划改进阶段的临时表决策见 [`../07_optimize/10_plan_refinement.md`](../07_optimize/10_plan_refinement.md)

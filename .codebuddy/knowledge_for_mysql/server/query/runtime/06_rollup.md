# 06 ROLLUP：GROUP BY 的多层汇总

> `GROUP BY ... WITH ROLLUP` 在普通分组之上额外输出各层次前缀汇总行（如按 `(year,country,product)` 分组时，还输出 `(year,country)`、`(year)`、全局汇总）。本篇讲 8.0 重构后的实现：**多层聚合封装 + 分组列 NULL 化 + 流式聚集**。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、语义与 level 约定](#一语义与-level-约定)
- [二、多层聚合：Item_rollup_sum_switcher](#二多层聚合item_rollup_sum_switcher)
- [三、分组列封装：Item_rollup_group_item](#三分组列封装item_rollup_group_item)
- [四、流式聚集：AggregateIterator 状态机](#四流式聚集aggregateiterator-状态机)
- [五、rollup 对优化器的影响](#五rollup-对优化器的影响)
- [六、深潜补充：GROUPING()、resolver 后处理与一条纠错](#六深潜补充grouping-resolver-后处理与一条纠错)

---

## 设计思想与理论基础

### ROLLUP 解决什么问题

多维分析里的"小计/合计"需求：按 `(year, country, product)` 分组时，还想要 `(year,country)` 小计、`(year)` 小计、全局总计。

不用 ROLLUP 的写法：4 个 `GROUP BY` 再用 `UNION ALL` 拼起来 —— **4 次扫描 + 4 次排序**。
用 ROLLUP：**一次扫描**产出所有层次。

### 语义本质：分组集的层次结构

ROLLUP 是 **GROUPING SETS 的一个特例**（前缀链）：

```
ROLLUP(a, b, c)  ≡  GROUPING SETS { (a,b,c), (a,b), (a), () }
```

三者表达能力由强到弱：

| 语法 | 产生的分组集 | 数量 | MySQL 8.0.39 |
|---|---|---|---|
| **GROUPING SETS** | 任意指定 | 任意 | ❌ 不支持 |
| **CUBE** | 所有子集组合 | 2ⁿ | ❌ 不支持 |
| **ROLLUP** | 前缀链 | n+1 | ✅ 支持 |

**ROLLUP 只产生 n+1 个分组集（一条链），CUBE 产生 2ⁿ 个** —— 这正是 ROLLUP 能被高效实现的关键：分组集构成一条**链**，可以靠一次排序 + 流式聚集顺序产出（本文第四章）。CUBE 的组合不构成链，需要更重的机制。

### 核心难点：NULL 的歧义

ROLLUP 用"把被 rollup 掉的分组列置 NULL"来表示汇总行。但**真实数据里本来也可能有 NULL**，于是产生歧义：

```
a=1, b=NULL    ← 这是"a=1 且 b 真实为 NULL"的明细行，还是"a=1 的小计行"？
```

SQL 标准为此引入 **`GROUPING()` 函数**：返回该列是否**因 rollup 而**被置 NULL（1 = rollup 产生的 NULL，0 = 真实值）。实现见本文第六章。

> 这是 ROLLUP 设计里最容易被忽略、但语义上最关键的一点：**汇总标记不能只靠 NULL 承载**。没有 `GROUPING()`，含 NULL 的数据就无法正确区分明细行与小计行。

### 实现思路：一次扫描 + 多层聚合器

要在一次扫描中同时产出 n+1 个层次的聚合值，就必须**为每个层次各维护一套聚合器**：

| 组件 | 职责 | 本文 |
|---|---|---|
| `Item_rollup_sum_switcher` | 包住聚合函数，内部持有 n+1 个聚合器 | 第二章 |
| `Item_rollup_group_item` | 包住分组列，按 level 决定是否输出 NULL | 第三章 |
| `AggregateIterator` | 流式状态机，在分组边界切换 level | 第四章 |

**代价是显式的**：聚合器数量 × (n+1)，内存与计算都按层次数放大。

### 他库对比

Oracle / PostgreSQL / SQL Server 都支持完整的 `GROUPING SETS` / `CUBE` / `ROLLUP`。MySQL 只实现了 `ROLLUP`——从 `olap_type` 枚举只有 `ROLLUP_TYPE` 一个值（本文 1.2 节）就能看出。这既是实现简化的选择，也是能力上的已知缺口。

### 代价要点

1. **聚合器 × (n+1)**：内存与计算按层次数放大
2. **排序要求**：多层汇总依赖输入已按分组列排序（filesort 或索引）
3. **优化器受限**：rollup 会禁用部分优化（本文第五章）

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

**★ 枚举只有两个值**——没有 `CUBE_TYPE`，也没有 `UNSURE_OLAP_TYPE`：

```cpp
/* sql/parser_yystype.h:179 */
enum olap_type { UNSPECIFIED_OLAP_TYPE, ROLLUP_TYPE };
```

> ⚠️ **两处常见位置错误**：① `olap` 不是 `ORDER` 的成员——它在 **`Query_block`** 上（`sql/sql_lex.h:2253`）；② `ORDER` 结构体本身在 **`sql/table.h:282`**，不是 `sql_lex.h`。`ORDER` 与 rollup 相关的成员只有 `rollup_item`（`sql/table.h:302`）。

```cpp
/* sql/sql_lex.h:2252-2253 */
  /// Indicates whether this query block contains the WITH ROLLUP clause
  olap_type olap{UNSPECIFIED_OLAP_TYPE};
```

```cpp
/* sql/table.h:302 —— ORDER 里与 rollup 相关的唯一成员 */
  Item_rollup_group_item *rollup_item{nullptr};
```

**文法**（`sql/sql_yacc.yy:12525-12559`）——`olap_opt` 只有两个产生式：

```yacc
opt_group_clause:
          %empty { $$= NULL; }
        | GROUP_SYM BY group_list olap_opt
          {
            $$= NEW_PTN PT_group($3, $4);
          }
        ;

olap_opt:
          %empty { $$= UNSPECIFIED_OLAP_TYPE; }
        | WITH_ROLLUP_SYM { $$= ROLLUP_TYPE; }
            /*
              'WITH ROLLUP' is needed for backward compatibility,
              and cause LALR(2) conflicts.
              This syntax is not standard.
              MySQL syntax: GROUP BY col1, col2, col3 WITH ROLLUP
              SQL-2003: GROUP BY ... ROLLUP(col1, col2, col3)
            */
        ;
```

**★ `WITH CUBE` 在 8.0.39 完全不支持**（三条实证）：

```yacc
689  %token  CUBE_SYM 356                      /* SQL-2003-R —— 仅声明，无任何产生式使用 */
1226 %token  OBSOLETE_TOKEN_893 893            /* was: WITH_CUBE_SYM —— 已废弃为占位 */
```

MTR 用例标题原文即写明：

```
# Old, never implemented CUBE syntax
GROUP BY product, country_id, year WITH CUBE;
ERROR 42000: You have an error in your SQL syntax; ... near 'WITH CUBE' at line 2
```

> ⚠️ `sql/item_sum.h:2688` 的 `Item_func_grouping` 注释里提到 "ROLLUP and CUBE" 是**历史措辞残留**，不要当成 CUBE 还在的证据。

落库（`PT_group::contextualize`，`parse_tree_nodes.cc:271-285`）——唯一与 ROLLUP 直接相关的语法报错是"不能用于全局 union 参数"：

```cpp
  switch (olap) {
    case UNSPECIFIED_OLAP_TYPE:
      break;
    case ROLLUP_TYPE:
      if (select->linkage == GLOBAL_OPTIONS_TYPE) {
        my_error(ER_WRONG_USAGE, MYF(0), "WITH ROLLUP",
                 "global union parameters");
        return true;
      }
      select->olap = ROLLUP_TYPE;
      break;
    default:
      assert(!"unexpected OLAP type!");
  }
```

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

**根本理由：同一个输入行，对不同 level 的聚合器要做不同动作——高层"继续累加"、低层"新组开始"。两种语义无法共用一份累加器。**

证据就是 `reset_and_add_for_rollup()` 里的那个切分：

```cpp
/* sql/item_sum.cc:6371-6387 */
void Item_rollup_sum_switcher::clear() {
  for (int i = 0; i < m_num_levels; ++i) {
    child(i)->clear();
  }
}

bool Item_rollup_sum_switcher::reset_and_add_for_rollup(
    int last_unchanged_group_item_idx) {
  for (int i = 0; i < m_num_levels; ++i) {
    if (i >= last_unchanged_group_item_idx) {
      if (child(i)->reset_and_add()) return true;    /* ★ 本层"新组开始"：重置后累加 */
    } else {
      if (child(i)->aggregator_add()) return true;   /* ★ 高层"仍在进行"：只累加不重置 */
    }
  }
  return false;
}
```

> ⚠️ **真名提醒**：**不存在** `aggregation_update()` / `aggregation_clear()` / `reset_and_add_if_needed()`（全 `sql/` grep 命中 0）。底层原语是 `Item_sum::aggregator_add()` / `aggregator_clear()`（`item_sum.h:677/683`），rollup 专用的是上面这两个。

类注释的权威表述（`item_sum.h:2708-2714`）：

> *"When aggregating, every aggregator is either **reset or updated** as per the correct level, and when returning a value, the correct child item corresponding to the current rollup level is queried."*

**副本怎么造出来**（resolver，`sql/sql_resolver.cc:5029-5053`）：

```cpp
Item *create_rollup_switcher(THD *thd, Query_block *query_block, Item_sum *item,
                             int send_group_parts) {
  List<Item> alternatives;
  alternatives.push_back(item);                  /* level 0：原始（最细粒度） */
  for (int level = 0; level < send_group_parts; ++level) {
    Item_sum *new_item = down_cast<Item_sum *>(item->copy_or_same(thd));
    if (new_item == nullptr) return nullptr;
    new_item->make_unique();                     /* ★ distinct 聚合各自独立的去重树 */
    if (alternatives.push_back(new_item)) return nullptr;
  }
  Item_rollup_sum_switcher *new_item = new Item_rollup_sum_switcher(&alternatives);
  if (new_item == nullptr || query_block->rollup_sums.push_back(new_item))
    return nullptr;
  new_item->quick_fix_field();
  return new_item;
}
```

> ★ `alternatives` 共 `1 + send_group_parts` 份。`make_unique()` 很关键：否则 `COUNT(DISTINCT x)` 这类聚合的多个 level 会**共用一棵去重树**，互相踩踏。

**`sum_funcs` 数组按 level 扩容**（`sql/sql_select.cc:4071-4086`），源码注释直说：

```cpp
bool JOIN::alloc_func_list() {
  func_count = tmp_table_param.sum_func_count;
  /*
    If we are using rollup, we need a copy of the summary functions for
    each level
  */
  if (rollup_state != RollupState::NONE) func_count *= (send_group_parts + 1);
  ...
  /* This must use calloc() as rollup_make_fields depends on this */
  sum_funcs = (Item_sum **)thd->mem_calloc(sizeof(Item_sum **) * (func_count + 1) + ...);
```

**运行期被真正消费的两处**（`composite_iterators.cc`）：

```cpp
      // ① 进入新组：按 last_unchanged_group_item_idx 切一刀
      for (Item_sum **item = m_join->sum_funcs; *item != nullptr; ++item) {
        if (m_rollup) {
          if (down_cast<Item_rollup_sum_switcher *>(*item)
                  ->reset_and_add_for_rollup(m_last_unchanged_group_item_idx))
            return true;
        } else {
          if ((*item)->reset_and_add()) return true;
        }
      }
```

```cpp
        // ② 同一组内继续读行：全 level 都 add
        for (Item_sum **item = m_join->sum_funcs; *item != nullptr; ++item) {
          if (m_rollup) {
            if (down_cast<Item_rollup_sum_switcher *>(*item)->aggregator_add_all())
              return 1;
          } else {
            if ((*item)->aggregator_add()) return 1;
          }
        }
```

`m_last_unchanged_group_item_idx` 的含义（类注释原文，讲得非常清楚）：

```cpp
  /**
    For rollup: The index of the first group item that did _not_ change when we
    last switched groups. E.g., if we have group fields A,B,C,D and then switch
    to group A,B,E,D, this value will become 1 (which means that we need
    to output rollup rows for 2 -- A,B,E,NULL -- and then 1 -- A,B,NULL,NULL).
    m_current_rollup_position will count down from the end until it becomes
    less than this value.

    If we do not have rollup, this value is perennially zero.
   */
  int m_last_unchanged_group_item_idx;
```

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

**`min_rollup_level` 就是该分组列在 GROUP BY 列表中的从 0 开始的下标**：

```cpp
ORDER *Query_block::find_in_group_list(Item *item, int *rollup_level) const {
  Item *real_item = item->real_item();
  if (real_item->type() == Item::CACHE_ITEM) {
    // Unwrap the cache, if any. NOTE: There should never be any caches
    // in the GROUP BY list, so we don't need to unwrap any from there.
    real_item = down_cast<const Item_cache *>(real_item)->get_example();
  }

  ORDER *best_candidate = nullptr;
  int idx = 0;
  for (ORDER *group = group_list.first; group; group = group->next, ++idx) {
    Item *group_item = *group->item;
    if (real_item->eq(group_item->real_item(), /*binary_cmp=*/false)) {
      if (item->item_name.ptr() != nullptr &&
          group_item->item_name.ptr() != nullptr &&
          item->item_name.eq(group_item->item_name)) {
        // Match on group _and_ alias; return immediately.
        if (rollup_level != nullptr) *rollup_level = idx;
        return group;
      } else if (best_candidate == nullptr) {
        // Match on group but not alias; it's a good candidate,
        // but only if we don't find a better match. (If there
        // are multiple such candidates, we use the leftmost one.)
        if (rollup_level != nullptr) *rollup_level = idx;
        best_candidate = group;
      }
    }
  }
  return best_candidate;
}
```

> ★ **别名优先**：`SELECT a AS c GROUP BY a,b,c` 里 `c` 会命中 idx=2（而不是 `a` 的 idx=0）。且"只匹配分组、不匹配别名"时取**最左**候选。

**包装动作**（`sql/sql_resolver.cc:4787-4817`）——注意 `group->rollup_item` 只回填**第一个**：

```cpp
static ReplaceResult wrap_grouped_expressions_for_rollup(
    Query_block *select, Item *item, Item *parent, unsigned argument_idx) {
  if (is_rollup_group_wrapper(item->real_item())) {
    // This item must already be a group item ... (also, don't traverse further).
    return {ReplaceResult::REPLACE, item};
  }

  int rollup_level = 0;
  ORDER *group = select->find_in_group_list(item, &rollup_level);
  if (group != nullptr) {
    Item_rollup_group_item *new_item = new Item_rollup_group_item(rollup_level, item);
    if (new_item == nullptr || select->rollup_group_items.push_back(new_item))
      return {ReplaceResult::ERROR, nullptr};
    new_item->quick_fix_field();
    if (group->rollup_item == nullptr) {
      group->rollup_item = new_item;      /* ★ 只记第一个 */
    }
    return {ReplaceResult::REPLACE, new_item};
  } else if (parent != nullptr && parent->type() == Item::FUNC_ITEM &&
             down_cast<Item_func *>(parent)->functype() == Item_func::GROUPING_FUNC) {
    my_error(ER_FIELD_IN_GROUPING_NOT_GROUP_BY, MYF(0), (argument_idx + 1));
    return {ReplaceResult::ERROR, nullptr};
  }
  return {ReplaceResult::KEEP_TRAVERSING, nullptr};
}
```

> ⚠️ 所以 `GROUP BY f1,f1,f2` 时只有第一个 `f1` 有 `rollup_item`。

**`find_in_group_list` 的第二个用途**（不取 level，只判"是否在 GROUP BY 里"）——**提前把列标 nullable，防止常量折叠**：

```cpp
void Query_block::mark_item_as_maybe_null_if_rollup_item(Item *item) {
  if (find_in_group_list(item, /*rollup_level=*/nullptr)) {
    /*
      If this item is present in GROUP BY clause, set m_nullable
      to true, as ROLLUP will generate NULLs for this column.
      This prevents the optimizer from constant-folding away
      IS NULL expressions (e.g. in HAVING). This must be done
      before we start resolving subselects in m_having_cond.
    */
    item->set_nullable(true);
  }
}
```

⇒ 这条解释了一个容易困惑的现象：`HAVING a IS NULL` 在带 ROLLUP 时**不会被优化掉**，因为 `a` 被强制标记成 nullable。

`rollup_level` 就是该列在 GROUP BY 列表中的**下标**：`GROUP BY a,b` 里 `a` 下标 0 → min level 0，`b` 下标 1 → min level 1。匹配用 `eq(..., false)` 非二进制比较，优先别名精确匹配。

`wrap_grouped_expressions_for_rollup`（:4787）在表达式树里递归查找匹配分组列的叶子，替换成 `Item_rollup_group_item`；若参数是 `GROUPING()` 但列不在 GROUP BY 里，报 `ER_FIELD_IN_GROUPING_NOT_GROUP_BY`。

替换后 `refresh_comparators_after_rollup`（:4944）重刷比较器——分组列现在可能为 NULL，之前被判为常量的比较器缓存已失效。

---

## 四、流式聚集：AggregateIterator 状态机

### 4.1 为什么必须流式

**唯一的原因在 `JOIN::optimize_rollup()`——把 `allow_group_via_temp_table` 置为 false**：

```cpp
/* sql/sql_optimizer.cc:11339-11352 */
/**
  Optimize rollup specification.

  Allocate objects needed for rollup processing.
*/
bool JOIN::optimize_rollup() {
  tmp_table_param.allow_group_via_temp_table = false;
  rollup_state = RollupState::INITED;
  tmp_table_param.group_parts = send_group_parts;
  return false;
}
```

（调用点 `sql/sql_optimizer.cc:373`：`if (query_block->olap == ROLLUP_TYPE && optimize_rollup()) return true;`）

后果链条：

```cpp
/* sql/sql_tmp_table.cc:882-888 —— create_tmp_table() 开头 */
  if (!param->allow_group_via_temp_table)
    group = nullptr;  // Can't use group key

  if (group != nullptr) distinct = false;  // Can't use distinct
```

`group = nullptr` ⇒ 临时表**不建分组唯一键** ⇒ 无法走 `TemptableAggregateIterator` 那条"靠唯一键定位分组行 + 增量 update"的路径。

底层 `Item_sum` 的注释说明了两条路径的分野：

```cpp
    This set of methods (result_item(), reset_field, update_field()) of
    Item_sum is used only if allow_group_via_temp_table is true. Otherwise
    copy_or_same() is used to obtain a copy of this item.
```

```cpp
  Note that update_field/reset_field are not in that
  class, because they're simply not called when
  GROUP BY/DISTINCT can be handled with help of index on grouped
  fields (allow_group_via_temp_table is false);
```

⇒ **为什么 ROLLUP 不能走临时表路径**：临时表那条路依赖"一个分组一行 + 唯一键定位"，而 ROLLUP 要**额外吐出**总计/小计行——这些行没有对应的输入行，临时表里无处安放。所以只能流式：边读边聚合，组切换时**凭空吐出**若干汇总行。

而这正是 `AggregateIterator` 必须"存/恢复行"的原因（类注释里那张小图讲得很清楚）：

```cpp
/**
  ...
  AggregateIterator needs to be able to save and restore rows; it doesn't know
  when a group ends until it's seen the first row that is part of the _next_
  group. When that happens, it needs to tuck away that next row, and then
  restore the previous row so that the output row gets the correct grouped
  values. A simple example, doing SELECT a, SUM(b) FROM t1 GROUP BY a:

    t1.a  t1.b                                       SUM(b)
     1     1     <-- first row, save it                1
     1     2                                           3
     1     3                                           6
     2     1     <-- group changed, save row
    [1     1]    <-- restore first row, output         6
                     reset aggregate              -->  0
    [2     1]    <-- restore new row, process it       1
     2    10                                          11
                 <-- EOF, output                      11

  To save and restore rows like this, it uses the infrastructure from
  pack_rows.h ...
*/
```

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

ROLLUP 会**关掉一大批优化**。逐条给出源码判据：

**① 不能用临时表聚合**（见 4.1）——`allow_group_via_temp_table = false`。

**② 不能用 loose index scan（GROUP_MIN_MAX）**——optimizer trace 里会看到 `"cause": "rollup"`：

```cpp
  /* Perform few 'cheap' tests whether this access method is applicable. */
  if (!join)
    cause = "no_join";
  else if (join->primary_tables != 1) /* Query must reference one table. */
    cause = "not_single_table";
  else if (join->query_block->olap == ROLLUP_TYPE) /* Check (B3) for ROLLUP */
    cause = "rollup";
  ...
```

**③ 不能用 indexed `AGGFN(DISTINCT)` 优化**：

```cpp
bool is_indexed_agg_distinct(JOIN *join, mem_root_deque<Item_field *> *out_args) {
  if (join->primary_tables > 1 ||             /* reference more than 1 table */
      join->select_distinct ||                /* or a DISTINCT */
      join->query_block->olap == ROLLUP_TYPE) /* Check (B3) for ROLLUP */
    return false;
```

**④ ★ ROLLUP 与 DISTINCT 不互斥，但强制物化**（这条常被写成"互斥"，是错的）：

```cpp
  /*
    Check if we need to create a temporary table prior to any windowing.

    (1) If there is ROLLUP, which happens before DISTINCT, windowing and ORDER
    BY, any of those clauses needs the result of ROLLUP in a tmp table.
    ...
  */
  if (rollup_state != RollupState::NONE &&  // (1)
      (select_distinct || has_windows || !order.empty()))
    need_tmp_before_win = true;
```

即：**ROLLUP 在前、DISTINCT 在后**，DISTINCT 必须先把 ROLLUP 的结果物化下来。唯一与 ROLLUP 直接相关的报错是"不能用于全局 union 参数"，不是与 DISTINCT 冲突。

**⑤ ORDER BY 是 GROUP BY 前缀时不能消掉 ORDER BY**（因为 rollup 插入的 NULL 汇总行会破坏顺序）：

```cpp
  if ((test_if_subpart(group_list.order, order.order) && !m_windows_sort &&
       query_block->olap != ROLLUP_TYPE) ||
      (group_list.empty() && tmp_table_param.sum_func_count)) {
    if (!order.empty()) {
      order.clean();
      trace_opt.add("removed_order_by", true);
    }
```

**⑥ 常量分组项不能被剔除**（`remove_const` 的第三参为 false）——否则 level 编号会错位：

```cpp
  group_list = ORDER_with_src(
      remove_const(group_list.order, where_cond,
                   rollup_state == RollupState::NONE, &simple_group, true),
      group_list.src, /*const_optimized=*/true);
```

**⑦ HAVING 不能下推成临时表条件**：

```cpp
    /*
      If we have already done the group, add HAVING to sorted table except
      when rollup is present
    */
    if (having_cond && group_list.empty() && !streaming_aggregation &&
        rollup_state == RollupState::NONE) {
      if (add_having_as_tmp_table_cond(curr_tmp_table)) return true;
    }
```

**⑧ ★ 函数依赖（FD）推导被整体放弃**——这是理解 ROLLUP 与 `ONLY_FULL_GROUP_BY` 交互的关键：

```cpp
  if (select->olap != UNSPECIFIED_OLAP_TYPE) {
    /*
      - the syntactical transformation of ROLLUP is to make a union of
        queries, and in each such query, some group column references are
        replaced with a NULL literal.
      - functional dependencies should be recognized only after that
        transformation. But there cannot be a key-based or equality-based
        functional dependency on a NULL literal.
        Moreover, there are no FDs in a UNION.
        So if the query has ROLLUP, we can stop here. Above, we have tested the
        column against the group columns and that's enough.
      See test group_by_fd_no_prot for examples of queries with ROLLUP which
      are accepted or not.
    */
    return false;
  }
```

派生表/视图同理：

```cpp
  if (mat_query_expression->is_set_operation() ||
      mat_query_block->olap != UNSPECIFIED_OLAP_TYPE)
    return;  // If UNION, EXCEPT, INTERSECT or ROLLUP, no FD
```

**⑨ 全文索引与 ROLLUP 互斥**（`MATCH` 只作用于基列）：

```cpp
  if (found) {
    my_error(ER_NOT_SUPPORTED_YET, MYF(0),
             "reading non-aggregated results of the MATCH full-text search "
             "function after GROUP BY WITH ROLLUP");
    return true;
  }
```

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

**`resolve_rollup_item()` 的完整实现**（`sql/sql_resolver.cc:4974-5027`）——三条后处理都很讲究：

```cpp
Item *Query_block::resolve_rollup_item(THD *thd, Item *item) {
  ReplaceResult result =
      wrap_grouped_expressions_for_rollup(this, item, nullptr, 0);
  if (result.action == ReplaceResult::ERROR) {
    return nullptr;
  } else if (result.action == ReplaceResult::REPLACE) {
    item->set_nullable(true);
    return result.replacement;
  }
  bool changed = false;
  bool error = WalkAndReplace(
      thd, item,
      [this, &changed](Item *inner_item, Item *parent, unsigned argument_idx) {
        ReplaceResult inner_result = wrap_grouped_expressions_for_rollup(
            this, inner_item, parent, argument_idx);
        changed |= (inner_result.action == ReplaceResult::REPLACE);
        return inner_result;
      });
  if (error) return nullptr;
  if (changed) {
    if (refresh_comparators_after_rollup(item)) return nullptr;     /* ① */
    item->update_used_tables();                                     /* ② */
    // Since item is now nullable, mark every expression (except rollup sum
    // functions) depending on it as also potentially nullable. (This is a
    // conservative choice; ...)
    Update_nullability_for_rollup_items info;
    if (WalkItem(item, enum_walk::PREFIX | enum_walk::POSTFIX,
                 [&info](Item *inner_item) {
                   if (info.is_stopped(inner_item)) {
                     return false;
                   } else if (inner_item->type() == Item::SUM_FUNC_ITEM &&
                              down_cast<Item_sum *>(inner_item)->real_sum_func() ==
                                  Item_sum::ROLLUP_SUM_SWITCHER_FUNC) {
                     info.stop_at(inner_item);                       /* ③ 遇 switcher 停止下钻 */
                     return false;
                   } else {
                     inner_item->set_nullable(true);
                     return false;
                   }
                 })) {
      return nullptr;
    }
  }
  return item;
}
```

> ★ **三个设计点**：① 重建 `Arg_comparator`（rollup 解析发生在 comparator 建立**之后**，原本判为 const 的东西可能变非 const）；② 重算 `used_tables`；③ **遇到 `Item_rollup_sum_switcher` 就停止下钻**——它内部的子聚合器不该被污染成 nullable。

```cpp
/**
  Refreshes the comparators after ROLLUP resolving.

  This is needed because ROLLUP resolving happens after the comparators have
  been set up. In ROLLUP resolving, it may turn out that something initially
  believed to be constant, is not constant after all (e.g., group items that may
  be NULL in some cases). So we call set_cmp_func() to make Arg_comparator
  adjust/remove its caches accordingly.
*/
static bool refresh_comparators_after_rollup(Item *item) {
  return WalkItem(item, enum_walk::POSTFIX, [](Item *inner_item) {
    if (inner_item->type() != Item::FUNC_ITEM) return false;
    switch (down_cast<Item_func *>(inner_item)->functype()) {
      case Item_func::GE_FUNC:
      case Item_func::GT_FUNC:
      case Item_func::LT_FUNC:
      case Item_func::LE_FUNC:
      case Item_func::EQ_FUNC:
      case Item_func::NE_FUNC:
      case Item_func::EQUAL_FUNC:
        return down_cast<Item_bool_func2 *>(inner_item)->set_cmp_func();
      default:
        return false;
    }
  });
}
```

> ⚠️ **一处过时注释**：`resolve_rollup()` 上方的 doc 写 "Resolve items in SELECT list **and ORDER BY list**"，但函数体**只遍历 `fields`**。ORDER BY 此时尚未 setup（`setup_order()` 在 `sql_resolver.cc:363-369`，晚于 `resolve_rollup` 的调用点 340）。实证：

```cpp
  // References to ROLLUP columns in SELECT and HAVING are represented
  // by Item_rollup_group_items. ...
  // The references in ORDER BY and GROUP BY are not wrapped in
  // Item_rollup_group_item, so we need to search for them.
```

### 6.4 hypergraph 优化器下的 rollup

**★ 8.0.39 的 hypergraph 优化器**支持** ROLLUP**——有专门的 `kRollup` ordering 种类：

```cpp
    /// Specific sequence of m_elements, and specific direction of each element.
    /// Needed for e.g. ORDER BY.
    kOrder,

    ///  Specific sequence of m_elements, but each element may be ordered in any
    /// direction. Needed for ROLLUP:
    kRollup,

    /// Elements may appear in any sequence and may be ordered in any direction.
    /// Needed for GROUP BY (with out ROLLUP), DISCTINCT, semi-join etc.
    kGroup
```

生成与转换：

```cpp
      case Ordering::Kind::kOrder:
        if (m_rollup) {
          AddRollupFromOrder(thd, state_idx, old_ordering);
        } else { ... }
      case Ordering::Kind::kRollup:
        assert(m_rollup);
        AddGroupingFromRollup(thd, state_idx, old_ordering);
        break;
```

```cpp
void LogicalOrderings::AddRollupFromOrder(THD *thd, int state_idx,
                                          const Ordering &ordering) {
  assert(m_rollup);
  assert(ordering.GetKind() == Ordering::Kind::kOrder);
  OrderingElementsGuard tmp_guard(this, thd->mem_root);
  Ordering::Elements &tmp = tmp_guard.Get();
  std::copy(ordering.GetElements().cbegin(), ordering.GetElements().cend(),
            tmp.begin());

  for (size_t i = 0; i < ordering.size(); ++i) {
    tmp[i].direction = ORDER_NOT_RELEVANT;          /* ★ rollup 不关心方向 */
    if (!m_items[m_items[tmp[i].item].canonical_item].used_in_grouping) {
      // Pruned away.
      return;
    }
  }

  Ordering rollup = Ordering(tmp.prefix(ordering.size()), Ordering::Kind::kRollup);
  AddEdge(thd, state_idx, /*required_fd_idx=*/0, rollup);
}
```

`AccessPath::AGGREGATE` 携带 `rollup` 标志，代价模型**专门估算 rollup 额外行**：

```cpp
AccessPath CreateStreamingAggregationPath(THD *thd, AccessPath *path, JOIN *join,
                                          bool rollup, double row_estimate,
                                          string *trace) {
  ...
  AccessPath aggregate_path;
  aggregate_path.type = AccessPath::AGGREGATE;
  aggregate_path.aggregate().child = child_path;
  aggregate_path.aggregate().rollup = rollup;
  aggregate_path.set_num_output_rows(row_estimate);
  EstimateAggregateCost(&aggregate_path, query_block, trace);
  return aggregate_path;
}
```

```cpp
  // If we have ROLLUP, there will be additional rollup rows. If we group on N
  // ... Estimating row count for ROLLUP on %zu terms.
```

`EXPLAIN FORMAT=JSON` 里会显式打出 `"rollup": true`：

```cpp
        } else if (path->aggregate().rollup) {
          error |= AddMemberToObject<Json_boolean>(obj, "rollup", true);
          description = "Group aggregate with rollup: ";
        }
```

> ⚠️ 所以"hypergraph 不支持 ROLLUP"是**旧说法**。grep `rollup` 于 `sql/join_optimizer/` 命中 >99 处，**没有任何** `ER_NOT_SUPPORTED_YET` 或 `assert(!rollup)` 形式的"不支持"证据。唯一的 `ER_NOT_SUPPORTED_YET + ROLLUP` 组合在 `sql/sql_optimizer.cc:294`，原因是 **FTS/MATCH** 而非 rollup 本身。

`finalize_plan.cc` 的 `CollectItemsWithoutRollup`（`:112`、`:165-166`）：物化时 rollup item 被剔除，但需把其**非 rollup 子项**加回 `items_to_copy`（否则物化表缺列）；替换走 `is_rollup_group_wrapper → unwrap_rollup_group`（`:420-421`）；`path->sort().unwrap_rollup`（`:451`）。

### 6.5 ⚠️ 纠错：不存在"TemptableAggregateIterator 的 rollup 三阶段"

如果你的资料来源提到 "rollup 在 TemptableAggregateIterator 里有 output/input/state 三阶段"——**本代码树没有**：

- `sql/iterators/` 下**无** `rolling_*` 文件，`sql/` 下**无** `rollup.cc`
- `TemptableAggregateIterator`（`composite_iterators.cc:1658/1703/1789/1890`）**没有任何 rollup 分支**

**根本原因**：`JOIN::optimize_rollup()`（`sql_optimizer.cc:11347-11348`）强制 `tmp_table_param.allow_group_via_temp_table = false`——**带 rollup 时禁止走临时表聚合**，所以 rollup 只能走流式 `AggregateIterator`（第四节的四态机），不可能进 TemptableAggregateIterator。

### 6.6 Item_rollup_group_item::used_tables()

```cpp
  table_map used_tables() const override {
    /*
      If underlying item is non-constant, return its used_tables value.
      Otherwise, ensure it is non-constant by adding RAND_TABLE_BIT.
    */
    return args[0]->const_for_execution()
               ? (args[0]->used_tables() | RAND_TABLE_BIT)
               : args[0]->used_tables();
  }
  void update_used_tables() override {
    Item_func::update_used_tables();
    set_rollup_expr();
  }
```

> ★ 即使内层是**常量**也要 `| RAND_TABLE_BIT` 把自己伪装成"非常量"——rollup 列的取值依赖**运行时 level**，不只依赖表内容。否则优化器会把它常量折叠掉。

同款处理也出现在 `Item::used_tables()` 的通用实现里：

```cpp
    if (map != 0) return map;
    // rollup constant: ensure it is non-constant by returning RAND_TABLE_BIT
    if (has_rollup_expr()) return RAND_TABLE_BIT;
    return 0;
```

```cpp
  bool has_rollup_expr() const { return m_accum_properties & PROP_ROLLUP_EXPR; }
  /// Set the property: this item (tree) contains a reference to a ROLLUP expr
  void set_rollup_expr() { m_accum_properties |= PROP_ROLLUP_EXPR; }
```

对照 `Item_rollup_sum_switcher::used_tables()`——它直接转发给 `master()`：

```cpp
  table_map used_tables() const override { return master()->used_tables(); }
```

### 6.7 ★ GROUPING() 的判据不是"值是否为 NULL"

这是个容易搞错的点：`Item_func_grouping::val_int()` **不看值**，而是看参数是否被 rollup 包装且当前处于 rollup NULL 状态：

```cpp
longlong Item_func_grouping::val_int() {
  longlong result = 0;
  for (uint i = 0; i < arg_count; i++) {
    Item *real_item = args[i]->real_item();
    if (has_rollup_result(real_item)) {
      result += 1ULL << (arg_count - (i + 1));     /* ★ 位掩码：最左参数 = 最高位 */
    }
  }
  return result;
}
```

```cpp
bool has_rollup_result(Item *item) {
  item = item->real_item();
  if (is_rollup_group_wrapper(item) &&
      down_cast<Item_rollup_group_item *>(item)->rollup_null()) {
    return true;                                   /* ★ 判据是 rollup_null()，不是 is_null() */
  }
  if (item->type() == Item::CACHE_ITEM) {
    return has_rollup_result(down_cast<Item_cache *>(item)->example);
  } else if (item->type() == Item::FUNC_ITEM) { /* 递归各参数 */ }
  else if (item->type() == Item::COND_ITEM) { /* 递归 */ }
  return false;
}
```

⇒ **真实 NULL 不会被置位**，只有 rollup 合成的 NULL 会。这正是 `GROUPING()` 存在的全部意义。

（`Item_func_grouping` 的注释里写 "NULL_RESULT_ITEM" 是历史说法，勿当真。）

另外 `GROUPING()` 的两个硬限制在 `fix_fields()` 里：

```cpp
  // Make GROUPING function dependent upon all tables (prevents const-ness)
  used_tables_cache |= thd->lex->current_query_block()->all_tables_map();

  /*
    More than 64 args cannot be supported as the bitmask which is
    used to represent the result cannot accommodate.
  */
  if (arg_count > 64) {
    my_error(ER_INVALID_NO_OF_ARGS, MYF(0), "GROUPING", arg_count, "64");
    return true;
  }

  /*
    GROUPING() is not allowed in a WHERE condition or a JOIN condition and
    cannot be used without rollup.
  */
  if (select->olap == UNSPECIFIED_OLAP_TYPE ||
      select->resolve_place == Query_block::RESOLVE_JOIN_NEST ||
      select->resolve_place == Query_block::RESOLVE_CONDITION) {
    my_error(ER_INVALID_GROUP_FUNC_USE, MYF(0));
    return true;
  }
```

---

## Misc

### ★★★ 真名勘误

| 常见说法 | 8.0.39 实际 |
|---|---|
| `ORDER` 有 `olap` 成员，定义在 `sql_lex.h` | **双错**。`olap` 是 **`Query_block`** 的成员（`sql_lex.h:2253`）；`ORDER` 定义在 **`sql/table.h:282`**，与 rollup 相关的成员只有 `rollup_item` |
| `enum olap_type` 有 `CUBE_TYPE` / `UNSURE_OLAP_TYPE` | 只有 **`UNSPECIFIED_OLAP_TYPE, ROLLUP_TYPE`** 两个值；正确拼写是 `UNSPECIFIED`（不是 `UNSURE`） |
| `WITH CUBE` 被支持 | **不支持**。`CUBE_SYM` 仅 token 声明无产生式；`WITH_CUBE_SYM` 已废弃为 `OBSOLETE_TOKEN_893`；MTR 标题原文 "Old, **never implemented** CUBE syntax" |
| `aggregation_update()` / `aggregation_clear()` / `reset_and_add_if_needed()` | **都不存在**（grep 0 命中）。真名：`Item_sum::aggregator_add()` / `aggregator_clear()`，rollup 专用 `reset_and_add_for_rollup()` |
| `Item_func_grouping` 在 `item_func.h` | 在 **`sql/item_sum.h:2694`**；`item_func.h` 里只有枚举 `GROUPING_FUNC` |
| `resolve_rollup` 处理 ORDER BY 里的分组列 | **不处理**。只遍历 `fields`；doc 注释里的 "and ORDER BY list" 是**过时注释** |
| ROLLUP 与 DISTINCT 互斥 | **不互斥**。共存但强制物化（`need_tmp_before_win = true`） |
| hypergraph 不支持 ROLLUP | **8.0.39 已支持**（`Ordering::Kind::kRollup` + `AccessPath::aggregate().rollup` + 代价估算） |
| `GROUPING()` 判据是"值是否为 NULL" | 判据是 `has_rollup_result()` 检查 **`rollup_null()`**，与值无关 |
| `Item_rollup_sum_switcher` 在 `item_sum.h:2708` | 2708-2714 只是 doc 注释，**类在 2715-2806** |
| `AggregateIterator` 在 `composite_iterators.cc:238` | `Read()` 确在 238，但**类定义在 `composite_iterators.h:205`** |

### 一句话总结

ROLLUP 的实现是**两个包装器 + 一个状态机**：
- `Item_rollup_group_item` 负责"把分组列变 NULL"（`rollup_null()` = `level <= min_level`）
- `Item_rollup_sum_switcher` 负责"每个 level 一份聚合器"（`args[level]` 切下标，零拷贝）
- `AggregateIterator` 负责"level 广播 + 组切换时凭空吐出汇总行"（`SetRollupLevel()` + `OUTPUTTING_ROLLUP_ROWS` 状态倒数）

代价是：`allow_group_via_temp_table = false` ⇒ **禁掉临时表聚合、loose index scan、indexed AGGFN(DISTINCT)、FD 推导**这一整批优化。

`item_func.h:1635-1643`：内层是**常量**时要 `| RAND_TABLE_BIT`，防止被当作常量折叠（rollup 下"常量"列在高层会被 NULL 化，不是真常量）。

## 参考

**内核月报**
- **2024/05《MySQL 深潜 - 重构后的 ROLLUP 实现》** —— `Item_rollup_sum_switcher` 多层聚合 + 流式聚集（本文的直接来源）

**官方文档**
- *MySQL 8.0 Reference Manual → GROUP BY Modifiers → WITH ROLLUP*
- *MySQL 8.0 Reference Manual → The GROUPING() Function*

**相关文档**
- 普通 GROUP BY 的临时表聚集见 [`01_filesort.md`](01_filesort.md)
- 窗口函数（同为聚合家族）见 [`04_window_function.md`](04_window_function.md)
- 计划改进阶段的临时表决策见 [`../07_optimize/10_plan_refinement.md`](../07_optimize/10_plan_refinement.md)

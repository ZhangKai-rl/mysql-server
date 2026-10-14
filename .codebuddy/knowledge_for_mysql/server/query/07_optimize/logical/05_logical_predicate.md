# 05 逻辑优化：等值传播、常量折叠、谓词下推、ICP

> 本篇覆盖 WHERE 条件的四类逻辑优化，全部算法级。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [零、流水线与常见误解](#零流水线与常见误解)
- [一、等值传播（Item_equal 等价类）](#一等值传播item_equal-等价类)
- [二、常量折叠](#二常量折叠)
- [三、谓词下推](#三谓词下推)
- [四、索引条件下推 ICP](#四索引条件下推-icp)

---

## 设计思想与理论基础

> 本章只回答"为什么这么设计"，实现细节见后面各节。

### A. 四类谓词优化的共性：三种原子操作

MySQL 里 WHERE 条件是一棵 `Item` 树。等值传播、常量折叠、谓词下推、ICP 看似无关，实则都在做**三种原子操作**：

| 原子操作 | 语义 | 对应优化 |
|---|---|---|
| **推导 derive** | 从已知谓词推出新谓词 | 等值传播、常量传播 |
| **移动 move** | 语义不变，改变求值点 | 谓词下推、ICP |
| **消解 resolve** | 优化期就求值，用 `Item_func_true/false` 替换/删除子树 | 常量折叠、平凡条件消除 |

⇒ 统一定义：**谓词优化 = 在不改变结果集的前提下，改变"哪些谓词存在"、"每个谓词在哪求值"、"哪些能在优化期定死"，使每一行被丢弃得尽可能早、尽可能便宜。**

**为什么"移动谓词"能带来收益？** 看 `POSITION` 的递推式：

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
    prefix_rowcount *= filter_effect;
  }
```

即 `prefix_rowcount(i) = prefix_rowcount(i-1) × rows_fetched(i) × filter_effect(i)`。

⇒ **谓词收益 = 选择率 × 它所在层的输入行数，且会被其后所有层连乘放大。** 越早生效的选择率，被后续层放大的次数越多，省得越多。

**一个关键且常被忽略的点**：`filter_effect` 只统计"**没被访问方法吃掉**"的谓词（源码注释：*"the fraction of the rows_fetched rows that will pass the table conditions that were NOT used by the access method"*）。

⇒ 由此推出：**下推的极致不是"挂到更早的表"，而是"变成访问方法"。** 一个谓词若变成 SARGable，它就从 `filter_effect` 挪进了 `rows_fetched`，直接减少**读**的行数而非读完后过滤的行数。这正是 B 节里 "sargable transitive closure" 比 "join transitive closure" 更值钱的原因。

**四类优化的顺序**（`optimize_cond()` 头注释就是官方定义）：

```
equality_propagation → constant_propagation → trivial_condition_removal
```

- **(a) 必须在 (b) 之前**：(b) 的注释原文 *"By transitivity, this also applies to MEPs, so the MEP in a) will become 42=x=y=z"*——没有 (a) 产出的 MEP 就没有 b 的输入；且 (b) 只认二元 `EQ_FUNC`/`EQUAL_FUNC`，**不认 `MULT_EQUAL_FUNC`**，所以两者是互补而非竞争
- **(c) 必须最后**：它是唯一**删除节点**的 pass，删除会破坏后续推导所需的语料。这是编译器优化的标准范式：**先做常量传播（增量），再做死代码消除（消减）**
- **ICP 必须最后**：它的第一个参数是 `keyno`（选中索引号）——没有 join order 和 access type 就没有"索引列"这个概念；且它的输入 `QEP_TAB::condition()` 本身就是普通下推的产物

一个不对称（源码直接可证）：`optimize_cond` 对 HAVING 的调用传 `join_list == nullptr`，所以 **HAVING 不做等值传播**（等值传播依赖 join 结构与 ON 的继承链），但常量折叠与平凡消除照做。

### B. 为什么需要"等价类"这个数据结构

**不用等价类、直接两两传播会怎样？两条路都走不通。**

- **路线一（只保留原谓词，临时传播）**：判断"f3 属于哪个类"每次都要扫描全部谓词，等于反复重算闭包
- **路线二（预先物化 n² 个谓词）**：`f1=f2 AND f2=f3 AND ...` 要完备必须物化 O(n²) 个 `fi=fj`，Item 树会膨胀，PS 重执行要回滚的改动量爆炸

**等价类的解法：用 O(n) 存储表达 O(n²) 的语义。** `Item_equal` 一个类注释就是设计说明书：

```cpp
/*
  An item of this class Item_equal(f1,f2,...fk) represents a
  multiple equality f1=f2=...=fk.
*/
```

成员是 `List<Item_field> fields`（O(n)）+ 可选 `Item *m_const_arg`。**它实质上就是 union-find，只是实现很"土"**：

| union-find 原语 | MySQL 实现 |
|---|---|
| **find** | `find_item_equal()` 沿 `cond_equal->current_level` 线性扫，再沿 `upper_levels` 往上爬 |
| **union** | `Item_equal::merge()` 只有 `fields.concat(&item->fields);` —— **O(1)** |
| 路径压缩 / 按秩合并 | **没有** |

**为什么"够用"**：等值谓词数量级是几十，`merge` 是 O(1)，路径压缩属于过度设计。

**关键对照**——O(n²) 的展开真实存在，但它是**消费端**的临时数组，不是存储端的 Item 树：

```cpp
} else {
  // 无常量：所有有序对 (field_i, field_j), i≠j —— O(n²)
  for (Item_field &outer : item_equal->get_fields())
    for (Item_field &inner : item_equal->get_fields())
      if (!outer.field->eq(inner.field))
        add_key_field(..., &outer, true, &inner_ptr, 1, ...);
}
```

`add_key_fields` 生成的是 `Key_use` 数组（一次性推导，用完就丢）。⇒ **等价类的价值就在于把 O(n²) 从"持久表示"降级为"一次性推导"。**

**还有等价类独有、两两传播根本做不到的能力：延迟决策。** O(n²) 展开必须在展开时决定"选哪个代表"，但"哪个最好"依赖 join 顺序——而 join 顺序在 `build_equal_items` 阶段还没定。`Item_equal` 把等价类整体留着，直到 `substitute_for_best_equal_field`（join order 定完后）才挑代表。**它是一份"尚未兑现的选择权"。**

**两层传递闭包**（类注释给出业界术语）：

1. **join transitive closure**——让任意两表之间被视为有等值连接条件，**枚举出原本不存在的访问路径**（扩大 join order 搜索空间）
2. **sargable transitive closure**——`f1=...=fk AND P(fi) ⇒ P(fj)`，把非 SARGable 谓词变成 SARGable

**为什么"推导新谓词"比"改写已有谓词"更有价值**：改写只是换个写法（谓词数不变），推导是 0→1（信息量净增）；改写有位置依赖（必须知道 join 顺序），推导没有；**推导出来的进了代价模型（变成 range/ref 压低 `rows_fetched`），改写出来的进不了**。

**MySQL 的关键取舍**——业界常规做法是"预处理时把闭包谓词全加上"，MySQL 偏不：

```cpp
  Both features are usually supported by preprocessing original query and
  adding additional predicates.
  We do not just add predicates, we rather dynamically replace some
  predicates that can not be used to access tables in the investigated
  plan for those, obtained by substitution of some fields for equal fields,
  that can be used.
```

**`m_const_arg` 的威力与陷阱**。一旦等价类里出现常量，`eliminate_item_equal` 做**星形展开**——每个 field 直接与常量配对：`Item_equal(5, t1.a, t2.b)` ⇒ `t1.a = 5 AND t2.b = 5`。三层威力：全列钉死成常量、矛盾检测免费（`compare_const` 里发现 `a=1 AND a=2` 就置 `cond_false` 并把 `used_tables_cache` 置 0，上层直接判 Impossible WHERE）、const 表读出后还能继续注入。

四个陷阱：

1. **有常量后，等价类不再是"纯等值连接条件"**——`contains_only_equi_join_condition() { return const_arg() == nullptr; }`。所以 `t1.a=t2.a AND t2.a=5` 这类查询，等值传播反而**关掉了 hash join 的大门**（展开后连接键消失）
2. **外连接列的"常量"不是常量**——`update_const()` 里 `if (item->const_item() && !item->is_outer_field())`。外连接内表的列可能因"空表→补 NULL"呈现 const 状态，那个 NULL 不能传播
3. **`val_int()` 只能对"已读到的字段"求值**——类注释：*"We have to take care of restricting the predicate ... to the projection of known fields"*。等价类里可能含还没读的表的字段
4. **PS 重执行要能回滚**——`Item_equal` 对象本身每次执行重建，所以内部修改不需登记；但 `eliminate_item_equal` 生成的新 `Item_func_eq` 挂进 AND 列表，**必须** `thd->change_item_tree` 登记

**★★ `Item_equal` 类的真身**（`sql/item_cmpfunc.h:2561`，⚠️ 三处常见误称）：

```cpp
class Item_equal final : public Item_bool_func {
  /// List of equal field items.
  List<Item_field> fields;                    /* ⚠️ 不是 Item_equal_fields，就是一个 List */
  /// Optional constant item equal to all the field items.
  Item *m_const_arg{nullptr};                 /* ★ 存常量的槽真名 m_const_arg，
                                                  访问器 const_arg()/set_const_arg() */
  /// Helper for comparing the fields.
  cmp_item *eval_item{nullptr};
  /// Helper for comparing constants.
  Arg_comparator cmp;
  /// Flag set to true if the equality is known to be always false.
  bool cond_false{false};                     /* ★ a=1 AND a=2 时置此位 */
  /// Should constants be compared as datetimes?
  bool compare_as_dates{false};

 public:
  Item *const_arg() const { return m_const_arg; }
  void set_const_arg(Item *c) { m_const_arg = c; }
  bool compare_const(THD *thd, Item *c);
  bool add(THD *thd, Item *c, Item_field *f);
  bool add(THD *thd, Item *c);
  void add(Item_field *f);
  uint members();
  bool contains(const Field *field) const;
  Item_field *get_first() { return fields.head(); }
  Item_field *get_subst_item(const Item_field *field);
  bool merge(THD *thd, Item_equal *item);
  bool update_const(THD *thd);
  enum Functype functype() const override { return MULT_EQUAL_FUNC; }
  const char *func_name() const override { return "multiple equal"; }
  optimize_type select_optimize(const THD *) override { return OPTIMIZE_EQUAL; }
  bool contains_only_equi_join_condition() const override {
    return const_arg() == nullptr;            /* ★ 有常量 ⇒ 不再是纯等值连接条件 */
  }
  ...
  bool cast_incompatible_args(uchar *) override {
    // Multiple equality nodes (Item_equal) should have been
    // converted back to simple equalities (Item_func_eq) by
    // substitute_for_best_equal_field before cast nodes are injected.
    assert(false);                            /* ★★ G 节元原则证据：Item_equal 必须在
                                                  注入 cast 前被拆回 Item_func_eq，否则崩溃 */
    return false;
  }
  ...
  Item *m_const_folding[2];  ///< temporary area used for constant folding
                             /* ★ MULT_EQUAL_FUNC 没有 args[]，折叠时借这两个槽冒充 */
```
> ⚠️ **真名核查**：`Item_equal_fields` **不存在**；`get_const()` / `const_item` 成员**不存在**；`Item_equal::compare_type()` **不存在**（类型解析走 `resolve_type()` + `eval_item`）；`COMPARE_ITEMS_TO` **全库 0 命中**。

**★★ 等价类合并就是链表 `concat`，不是 union-find**（`sql/item_cmpfunc.cc:6817`，★ 这是最重要的纠错）：

```cpp
bool Item_equal::merge(THD *thd, Item_equal *item) {
  fields.concat(&item->fields);        /* ★★ 链表拼接：O(1) 搬指针，没有 union/find、没有路径压缩 */
  Item *c = item->m_const_arg;
  if (c) {
    /*
      The flag cond_false will be set to 1 after this, if
      the multiple equality already contains a constant and its
      value is  not equal to the value of c.
    */
    if (add(thd, c)) return true;
  }
  cond_false |= item->cond_false;
  if (cond_false) used_tables_cache = 0;      /* ★ 冲突 ⇒ 位图清零 ⇒ const_item() 立即为真 */
  return false;
}
```
> ⚠️ **全库唯一提到 union-find 的地方是 hypergraph 的 `graph_simplification.cc:143`**，且明确说"MAX_TABLES 太小，用 bitset 就够了"。**`Item_equal` 与 union-find 无关。**

**★ 矛盾检测：第二个常量进来才现形**（`sql/item_cmpfunc.cc:6746`）：

```cpp
bool Item_equal::compare_const(THD *thd, Item *c) {
  if (compare_as_dates) {
    cmp.set_datetime_cmp_func(this, &c, &m_const_arg);
    cond_false = cmp.compare();
  } else {
    Item_func_eq *func = new Item_func_eq(c, m_const_arg);
    if (func == nullptr) return true;
    if (func->set_cmp_func()) return true;
    func->quick_fix_field();
    cond_false = !func->val_int();
  }
  if (thd->is_error()) return true;
  if (cond_false) used_tables_cache = 0;   /* ★★ 关键：位图清零 ⇒ 上层判 Impossible WHERE */
  return false;
}

bool Item_equal::add(THD *thd, Item *c, Item_field *f) {
  if (cond_false) return false;
  if (m_const_arg == nullptr) {
    assert(f);
    m_const_arg = c;
    compare_as_dates = f->is_temporal_with_date();
    return false;                          /* 第一个常量只是"登记" */
  }
  return compare_const(thd, c);            /* ★ 第二个常量才做比较 */
}
```

**★ `update_const()`：外连接列的"伪常量"必须排除**（`sql/item_cmpfunc.cc:6852`）：

```cpp
bool Item_equal::update_const(THD *thd) {
  List_iterator<Item_field> it(fields);
  Item *item;
  while ((item = it++)) {
    if (item->const_item() &&
        /*
          Don't propagate constant status of outer-joined column.
          Such a constant status here is a result of:
            a) empty outer-joined table: in this case such a column has a
               value of NULL; but at the same time other arguments of
               Item_equal don't have to be NULLs ...
          or
            b) outer-joined table contains only 1 row: the result of
               this column is equal to a row field value *or* NULL.
          Both values are inacceptable as Item_equal constants.
        */
        !item->is_outer_field()) {
      it.remove();                          /* ★ 字段升级成常量，从 fields 挪到 m_const_arg */
      if (add(thd, item)) return true;
    }
  }
  return false;
}
```

**★ `val_int()` 只跟"已读的 const 表"比**（`sql/item_cmpfunc.cc:7042`，★ 陷阱 3 的源码）：

```cpp
longlong Item_equal::val_int() {
  Item_field *item_field;
  if (cond_false) return 0;
  List_iterator_fast<Item_field> it(fields);
  Item *item = m_const_arg != nullptr ? m_const_arg : it++;
  eval_item->store_value(item);
  if ((null_value = item->null_value)) return 0;
  while ((item_field = it++)) {
    /* Skip fields of non-const tables. They haven't been read yet */
    if (item_field->field->table->const_table) {   /* ★★ 只跟已读的表比 */
      const int rc = eval_item->cmp(item_field);
      if ((rc == static_cast<int>(true)) || (null_value = (rc == UNKNOWN)))
        return 0;
    }
  }
  return 1;
}
```

**★ `COND_EQUAL`：层级链，不是并查集树**（`sql/item_cmpfunc.h:2701`，⚠️ 不在 `sql_lex.h`）：

```cpp
class COND_EQUAL {
 public:
  uint max_members;               /* max number of members the current level
                                     list and all lower level lists */
  COND_EQUAL *upper_levels;       /* multiple equalities of upper and levels */
  List<Item_equal> current_level; /* list of multiple equalities of
                                     the current and level           */
  COND_EQUAL() { upper_levels = nullptr; }
};
```
> ★ `current_level` = 本 AND 层的等价类集合；`upper_levels` = 指向外层（WHERE / 外层 ON）。**查找是沿链向上线性搜索**（`Item_field::find_item_equal`）。

**★ HAVING 不做等值传播的源码依据**（`sql/sql_optimizer.cc:10353`）：

```cpp
  /*
    Build all multiple equality predicates and eliminate equality
    predicates that can be inferred from these multiple equalities. ...
    This is performed for the WHERE condition and any join conditions, but
    not for the HAVING condition.
  */                                /* ★★ 源码明说 */
  if (join_list) {                  /* ★ join_list==nullptr 即 HAVING，整个分支跳过 */
    ...
    if (build_equal_items(thd, *cond, cond, nullptr, true, join_list, cond_equal))
      return true;
  }
```
> ★ 调用侧印证：`optimize_cond(thd, &having_cond, &cond_equal, nullptr, ...)` —— 第 4 实参就是 `nullptr`（`sql_optimizer.cc:487`）。

**★ `eliminate_item_equal`：O(n) 星形展开，不是 O(n²)**（`sql/sql_optimizer.cc:4606`）：

```cpp
static Item *eliminate_item_equal(THD *thd, Item *cond,
                                  COND_EQUAL *upper_levels,
                                  Item_equal *item_equal) {
  List<Item> eq_list;
  Item *eq_item = nullptr;
  if (item_equal->const_item() && !item_equal->val_int())
    return new Item_func_false();         /* ★ 等价类恒假 ⇒ 直接返回 false */
  Item *const item_const = item_equal->const_arg();
  auto it = item_equal->get_fields().begin();
  if (!item_const) {
    /*
      If there is a const item, match all field items with the const item,
      otherwise match the second and subsequent field items with the first one:
    */
    it++;                                 /* ★★ O(n) 展开：n 个成员只生成 n-1 个等值，
                                               而不是两两组合的 n(n-1)/2 */
  }
  while (it != item_equal->get_fields().end()) {
    Item_field *item_field = &*it++;
    Item_equal *const upper = item_field->find_item_equal(upper_levels);
    if (upper) {                          /* ★ 上层已有 → 不重复生成 */
      ...
      if (!(tab && sj_is_materialize_strategy(tab->get_sj_strategy()))) {
        ...
        if (item_match != item_field) continue;
      }
    }
    Item *const head =
        item_const ? item_const : item_equal->get_subst_item(item_field);
    if (head == item_field) continue;
    ...
    eq_item = new Item_func_eq(item_field, head);
    ...
    if (item_const != nullptr) {
      eq_item->apply_is_true();
      Item::cond_result res;
      if (fold_condition(thd, eq_item, &eq_item, &res)) return nullptr;
      if (res == Item::COND_FALSE) {
        eq_item = new (thd->mem_root) Item_func_false();
        return eq_item;  // entire AND is false
      } else if (res == Item::COND_TRUE) {
        eq_item = new (thd->mem_root) Item_func_true();
      }
    }
  }
```
> ★ 展开时**每个新等值都立刻过一遍 `fold_condition`** —— 所以 `a=1 AND a=2` 在展开这一步就被判死。

**★ "选最好的字段"的判据：执行计划里的表序号**（`sql/sql_optimizer.cc:4517`）：

```cpp
static int compare_fields_by_table_order(Item_field *field1, Item_field *field2,
                                         JOIN_TAB **table_join_idx) {
  int cmp = 0;
  bool outer_ref = false;
  if (field1->is_outer_reference()) { outer_ref = true; cmp = -1; }
  if (field2->is_outer_reference()) { outer_ref = true; cmp++; }
  if (outer_ref) return cmp;              /* ★ 外层引用永远最优：它在外层已可用 */

  /*
    table_join_idx is NULL if this function was not called from JOIN::optimize()
    but from e.g. mysql_delete() or mysql_update(). ...
  */
  if (!table_join_idx) return 0;

  cmp = table_join_idx[field1->table_ref->tableno()]->idx() -
        table_join_idx[field2->table_ref->tableno()]->idx();
  return cmp < 0 ? -1 : (cmp ? 1 : 0);    /* ★★ 唯一判据：idx() 越小越"好" */
}
```
> ★ 排序后 `fields.head()` 就是"最早可读的表"的字段——这就是"延迟决策"的兑现方式。

**★★ 五个 Impossible WHERE 判定点**（`sql/sql_optimizer.cc`，★ 2.5 节核心）：

```cpp
/* ① :479 —— optimize_cond 返回 COND_FALSE */
    if (query_block->cond_value == Item::COND_FALSE) {
      zero_result_cause = "Impossible WHERE";
      best_rowcount = 0;
      create_access_paths_for_zero_rows();
      goto setup_subq_exit;
    }
/* ② :784 —— make_join_query_block 失败（真正来源在 ④） */
  if (make_join_query_block(this, where_cond)) {
    if (thd->is_error()) return true;
    zero_result_cause = "Impossible WHERE noticed after reading const tables";
    create_access_paths_for_zero_rows();
    goto setup_subq_exit;
  }
/* ③ :3021 —— adjust_access_methods 里 impossible_range */
    if (!tab->const_keys.is_clear_all() &&
        tab->table()->reginfo.impossible_range && ...)
      zero_result_cause = "Impossible WHERE noticed after reading const tables";
/* ④ :9637 —— ★ const 表条件求值为假（"after reading const tables"的真正来源） */
  if (const_cond != nullptr &&
      evaluate_during_optimization(const_cond, join->query_block)) {
    const bool const_cond_result = const_cond->val_int() != 0;
    ...
    } else {
      DBUG_PRINT("info", ("Found impossible WHERE condition"));
      return true;
    }
  }
/* ⑤ :9912 —— test_quick_select 二次探测（区分 impossible WHERE 与 impossible ON） */
              /*
                Before reporting "Impossible WHERE" for the whole query
                we have to check isn't it only "impossible ON" instead
              */
              if (!tab->join_cond())
                return true;  // No ON, so it's really "impossible WHERE"
```
> ★★ 第 ⑤ 点的注释解释了一个 EXPLAIN 细节：**外连接下必须先排除"只是 ON 恒假"**，否则会误报 Impossible WHERE。

**★ "Always true" 的判定点**（`sql_optimizer.cc:9630`）：

```cpp
      if (join->plan_is_const() &&
          !(cond->used_tables() & ~join->const_table_map) &&
          !cond->is_expensive()) {
        DBUG_PRINT("info", ("Found always true WHERE condition"));
        join->where_cond = nullptr;      /* ★ 整个 WHERE 被置空 */
      }
```

**⚠️ `COND_IMPOSSIBLE` / `COND_TRUE` 常量不存在**。真名是枚举 `Item::cond_result { COND_UNDEF, COND_OK, COND_TRUE, COND_FALSE }`（`sql/item.h:951`）+ 具体 Item 类 `Item_func_true`（`TRUE_FUNC`）/ `Item_func_false`（`FALSE_FUNC`）（`sql/item_cmpfunc.h:377`）+ 字符串 `zero_result_cause`。
> ⚠️ `cond_is_false()` / `cond_is_true()` 也不存在 → 真名 `can_evaluate_condition()` + `eval_const_cond()`：
> ```cpp
> bool eval_const_cond(THD *thd, Item *cond, bool *value) {
>   // Function may be used both during resolving and during optimization:
>   assert(cond->may_evaluate_const(thd));   /* ★ 求值前的最后一道断言 */
>   *value = cond->val_bool();
>   return thd->is_error();
> }
> ```

### C. 常量折叠：三档"常量"与为什么不折叠所有

MySQL 把"常量"分三档，语义完全不同：

| 档位 | 判定 | 谁能用 |
|---|---|---|
| `basic_const_item()` | 就是字面量，求值无副作用 | `resolve_const_item` 的最低档 |
| `const_item()` | `used_tables() == 0` | 优化期可求值 |
| `const_for_execution()` | `!(used_tables() & ~INNER_TABLE_BIT)` | 表锁好、PS 参数绑定后才可求值 |

**为什么必须排除子查询/非确定性/参数？靠"伪表位"，不靠 flag。** `RAND()` 通过 `get_initial_pseudo_tables()` 返回 `RAND_TABLE_BIT`；`Item_param::used_tables()` 返回 `INNER_TABLE_BIT`——**一次位或运算就把"不确定性"传播到整棵树上**。

第二道防线是 `is_non_const_over_literals` walker（位图挡不住副作用函数、SP 变量、cache item）：`Item_param` / `Item_ref` / `Item_cache` / `get_lock` 系列都 override 成 `true`。

**为什么折叠放在 `fix_fields` 里而不是单独的 pass**：

1. `fix_fields` 自底向上、只跑一次，且此时类型/校对集/**比较函数**都已确定——`val_int()` 需要 `set_cmp_func()` 已设好比较器，这只有在 `fix_fields` 之后才成立
2. **零额外遍历**——`Item_cond::fix_fields` 本来就要遍历参数列表
3. **能顺手把整棵子树摘掉**（`li.remove()`）
4. **真正的性能实质不是"把 1+2 变成 3"**，而是 `Arg_comparator::set_cmp_func` → `cache_converted_constant`：`WHERE int_col = '3'` 只做一次字符串→整数转换，而不是每行一次

**为什么不折叠所有能折叠的**——`Item_cond::fix_fields` 里那一长串守卫，每一条都是一次真实事故：

| 守卫 | 为什么 |
|---|---|
| `ref != nullptr` | 调用方必须允许改树 |
| `select->first_execution` | **PS/SP 重执行时树已被改过**，再折叠会二次删除 |
| `ignore_unknown()` | **三值逻辑**：只有顶层布尔才能二值化（非顶层可返回 UNKNOWN） |
| `!select->has_ft_funcs()` | `ftfunc_list` 持有指针，删掉会悬空 |
| `can_remove_cond` | IN→EXISTS 改造出来的条件不能删 |
| `!is_view_context_analysis()` | 视图上下文分析阶段没有真实数据 |
| `!walk(is_non_const_over_literals)` | 排除 param / SP args / 副作用函数 |

**关于 PS 参数的不对称**：`Item_param` 在 prepare 期被两道防线挡住（不可回滚的删除不能做），但 optimize 期 `fold_condition` 用 `const_for_execution()`——`WHERE col > ?` **可以折叠**（类型域折叠：`tinyint_col > 999` ⇒ 恒假）。⇒ **可回滚的折叠可以做，不可回滚的删除必须等参数确定或只在首次执行做。**

还有一层"主动不折叠"：`can_evaluate_condition()` 要求 `!is_expensive()`——注释 *"Used during optimization to avoid computing expensive expressions during this phase"*。**昂贵的常量表达式故意留到执行期。**

**★ 三档常量的源码真身**（`sql/item.h:2280`，★ 每个都只有一行位图判断）：

```cpp
  /**
    Returns true if item is constant, regardless of query evaluation state.
    An expression is constant if it:
    - refers no tables.
    - refers no subqueries that refers any tables.
    - refers no non-deterministic functions.
    - refers no statement parameters.
    - contains no group expression under rollup
  */
  bool const_item() const { return (used_tables() == 0); }
                                          /* ★ 第 1 档（最严）：位图全 0 */
  /**
    Returns true if item is constant during one query execution.
    If const_for_execution() is true but const_item() is false, value is
    not available before tables have been locked and parameters have been
    assigned values. This applies to
    - statement parameters
    - non-dependent subqueries
    - deterministic stored functions that contain SQL code.
  */
  bool const_for_execution() const {
    return !(used_tables() & ~INNER_TABLE_BIT);
  }
                                          /* ★ 第 2 档：只容忍 INNER_TABLE_BIT */

  /**
    Return true if this is a const item that may be evaluated in
    the current phase of statement processing. ...
  */
  bool may_evaluate_const(const THD *thd) const;   /* ★ 第 3 档（阶段相关） */

  /**
    @returns true if this item is non-deterministic, which means that a
             has a component that must be evaluated once per row in
             execution of a JOIN query.
  */
  bool is_non_deterministic() const { return used_tables() & RAND_TABLE_BIT; }
                                          /* ⚠️ 真名核查：存在，就是 RAND_TABLE_BIT 位测试 */
  bool is_outer_reference() const {
    return used_tables() & OUTER_REF_TABLE_BIT;
  }
```

**★ `may_evaluate_const()` 让三档合流**（`sql/item.cc:1203`）：

```cpp
bool Item::may_evaluate_const(const THD *thd) const {
  // Ensure tables are locked whenever preparation is complete
  assert(!thd->lex->is_exec_started() || thd->lex->is_query_tables_locked());
  return !(thd->lex->context_analysis_only & CONTEXT_ANALYSIS_ONLY_VIEW) &&
         (const_item() ||
          (const_for_execution() && thd->lex->is_exec_started()));
}
```
> ★ 三分支：① 分析视图 ⇒ 一律不折叠；② `const_item()` ⇒ 随时可折叠；③ `const_for_execution()` 但要等 `is_exec_started()`（表已锁）。

**★ `evaluate_during_optimization()`：子查询这道坎**（`sql/sql_optimizer.cc:11429`）：

```cpp
bool evaluate_during_optimization(const Item *item, const Query_block *select) {
  /*
    Should only be called on items that are const_for_execution(), as those
    items are the only ones that are allowed to be evaluated during optimization
    in the first place.
    Additionally, allow items that only access tables in JOIN::const_table_map.
    This should not be necessary, but the const_for_execution() property is not
    always updated correctly by update_used_tables() for certain subqueries.
  */
  assert(item->const_for_execution() ||
         (item->used_tables() & ~select->join->const_table_map) == 0);

  // If the Item does not access any tables, it can always be evaluated.
  if (item->const_item()) return true;

  return !item->has_subquery() || (select->active_options() &
                                   OPTION_NO_SUBQUERY_DURING_OPTIMIZATION) == 0;
}
```

**★★ 折叠入口是三层，且会"二次折叠"**（`sql/sql_optimizer.cc:10402`，⚠️ 不只是 `fold_condition`）：

```cpp
static bool can_evaluate_condition(THD *thd, Item *condition) {
  return condition->const_for_execution() && !condition->is_expensive() &&
         evaluate_during_optimization(condition,
                                      thd->lex->current_query_block());
}

/**
  Calls fold_condition. If that made the condition constant for execution,
  simplify and fold again. @see fold_condition() for arguments.
*/
static bool fold_condition_exec(THD *thd, Item *cond, Item **retcond,
                                Item::cond_result *cond_value) {
  if (fold_condition(thd, cond, retcond, cond_value)) return true;
  if (*retcond != nullptr &&
      can_evaluate_condition(thd, *retcond))  // simplify further maybe
    return remove_eq_conds(thd, *retcond, retcond, cond_value);
                                          /* ★★ 二次折叠：折叠后再判一次，能则递归回去
                                              —— 这是一个不动点迭代，不是单次遍历 */
  return false;
}
```

**★★ "为什么不折叠所有"的两处硬证据**（`sql/sql_const_folding.cc:1298`）：

```cpp
    } else if (args[i]->const_for_execution() && type != Item::SUBSELECT_ITEM) {
      /*
        Re test on Item::SUBSELECT_ITEM above: we exclude optimize time
        evaluation of constant subqueries for now; since it could still be
        expensive to evaluate and we have no cost model to decide whether
        folding it would save total time spent. It may turn out not to be
        executed at all.
      */                                /* ★ 证据 1：常量子查询不折叠 —— "没有代价模型，
                                              而且它可能根本不会被执行" */
      if (type != Item::FUNC_ITEM) {
        seen_constant = true;
      } else {
        /*
          This "else"-block covers constants that have the shape of
          function calls. We only try to fold two kinds of functions here:
          1) Monadic minus, 2) Item_datetime_literal.
        */
        const auto f = down_cast<Item_func *>(args[i]);
        const auto ft = f->functype();
        if ((ft == Item_func::NEG_FUNC &&
             f->arguments()[0]->basic_const_item()) ||
            ft == Item_func::DATETIME_LITERAL) {
          seen_constant = true;
        }                               /* ★★ 证据 2：函数形常量只认两种——
                                              一元负号（且参数是基本常量）与 DATETIME_LITERAL。
                                              其余（含所有非确定性函数、SP 变量）一律不折叠 */
      }
    }
```
> ★ `seen_field && seen_constant` 同时成立才折叠，否则退化为 `fold_arguments()` 只折叠参数。

**★ `MULT_EQUAL_FUNC` 借两元素 scratch 数组冒充 args**（`sql_const_folding.cc:1317`，★ 一个巧妙的阻抗匹配）：

```cpp
  if (func_type != Item_func::MULT_EQUAL_FUNC) {
    args = func->arguments();
  } else {
    /*
      Impedance mispatch between how field, constant is represented for the
      normal comparison functions and multi-equal: we use a scratch two args
      array and update the multi-equal's normal constant later.
    */
    const auto equal = down_cast<Item_equal *>(func);
    // Use first field:  any one will do: they have the same type
    equal->m_const_folding[0] = equal->get_first();
    equal->m_const_folding[1] = equal->const_arg();
    args = equal->m_const_folding;
  }
```

**★ `fold_condition` 的返回值语义**（`sql/sql_const_folding.h:36`）：

```cpp
  @param[out] cond_value resulting value of the condition
     - COND_OK:    condition must be evaluated (e.g int == 3)
     - COND_TRUE:  always true                 (e.g signed tinyint < 128)
     - COND_FALSE: always false                (e.g unsigned tinyint < 0)
```
> ★ 典型：`signed tinyint < 128` ⇒ `COND_TRUE`；`unsigned tinyint < 0` ⇒ `COND_FALSE`；`int_col > 2147483647` ⇒ `COND_FALSE`。这就是**类型域折叠**。

### D. 谓词下推：两道门与外连接的守卫

`make_cond_for_table(cond, tables, used_table, ...)` 是**递归 + 两道门**：

**第一道门（归属）**：这条件跟我要服务的表有关系吗？

```cpp
if (used_table &&                                     // 1
    !(cond->used_tables() & used_table) &&            // 2
    !(cond->is_expensive() && used_table == tables))  // 3
  return nullptr;
```

**第二道门（可求值）**：这条件引用的表都读到了吗？

```cpp
if ((cond->used_tables() & ~tables) ||
    (!used_table && exclude_expensive_cond && cond->is_expensive()))
  return nullptr;
```

**切分规则**：AND 可拆（每个合取项独立递归）；**OR 全或无**（任一 disjunct 提不出来就整个放弃——只保留 A 会错误过滤掉 B 为真的行，这是 soundness 问题不是性能问题）。

两道门合起来保证：**每个条件挂到"所有依赖字段都可用的第一张表"，且只挂一次**。整个下推只需 **O(条件数 × 树深)**。

两个细节：第一张表额外加 `const_table_map | OUTER_REF_TABLE_BIT`（否则常量条件永远过不了第一道门）；最后一张表加 `RAND_TABLE_BIT`（否则 `RAND()` 这类条件会挂不到任何表而丢失）。

**为什么下推要小心外连接**：内连接里 WHERE 与 ON 语义等价可随便挪；外连接里**不成立**——`t1 LEFT JOIN t2 ON P` 对不匹配的行会补一条全 NULL 的 t2 行，这条补出来的行**没有被 ON 之外的条件约束过**。

MySQL 的答案不是"不下推"，而是**下推 + 用 `Item_func_trig_cond` 守卫**：

- `IS_NOT_NULL_COMPL`——"我正在生成 NULL 补行，快把条件关掉"
- `FOUND_MATCH`——"只有内层外连接找到匹配才打开"

⇒ **外连接下推的正确性靠"条件可以被动态开关"，而不是靠"选对位置"。** `Item_func_trig_cond` 把"位置"语义化成了"守卫位"。

**下推 vs 延迟求值的张力**——不是"推得越早越好"：昂贵的常量条件**故意不折叠不下推**（挂在第一张表上留到执行期）；HAVING 不能下推到 WHERE（`SUM(x)` 在聚合前根本不存在）。

⇒ 原则：**默认下推，但对三类条件说不**——昂贵的、语义上尚不可求值的（聚合/窗口）、会改变 NULL 补行语义的（外连接/反连接）。

**★ `make_cond_for_table()` 只返回 `Item *`（两态）**（`sql/sql_optimizer.cc:9441`，⚠️ 常见说法"三返回值"是错的）：

```cpp
  @retval <>NULL Generated condition
  @retval = NULL Already checked, OR error    /* ★ NULL = "已检查过 / 恒真"，不是"出错" */

  @details
    ...
    This function can be used to extract conditions relevant for a table
    in a join order. Together with its caller, it will ensure that all
    conditions are attached to the first table in the join order where all
    necessary fields are available, and it will also ensure that a given
    condition is attached to only one table.
    ...
    The function takes into account that some parts of the condition are
    guaranteed to be true by employed 'ref' access methods (the code that
    does this is located at the end, search down for "EQ_FUNC").
```
> ★ 最后一句解释了为什么 ref 访问下 `t1.a=5` 不会再出现在表条件里——**ref access 已经保证了它**。

**★★ `Item_func_trig_cond`：守卫谓词的完整定义**（`sql/item_cmpfunc.h:785`）：

```cpp
/*
  trigcond<param>(arg) ::= param? arg : true
  ...
  A guarded predicate P/g is evaluated to true if the value of the
  guard g is false, otherwise it is evaluated to the same value that
  the predicate P: val(P/g)= g ? val(P):true.
  Guarded predicates allow us to include predicates into a conjunction
  conditionally. ...
*/

class Item_func_trig_cond final : public Item_bool_func {
 public:
  enum enum_trig_type {
    /**
      This trigger type deactivates join conditions when a row has been
      NULL-complemented. ...
    */
    IS_NOT_NULL_COMPL,
    /**
      This trigger type deactivates predicated from WHERE condition when no
      row satisfying the join condition has been found. ...
    */
    FOUND_MATCH,
    /**
       In IN->EXISTS subquery transformation, new predicates are added: ...
    */
    OUTER_FIELD_IS_NOT_NULL
  };                                     /* ⚠️ 三个值，不是两个 */
 private:
  /** Pointer to trigger variable */
  bool *trig_var;
  const JOIN *m_join;
  plan_idx m_idx;
  enum_trig_type trig_type;
  ...
  longlong val_int() override;
  enum Functype functype() const override { return TRIG_COND_FUNC; }
  /// '@<if@>', to distinguish from the if() SQL function
  const char *func_name() const override { return "<if>"; }
```
```cpp
/* sql/item_cmpfunc.cc:7123 —— ★ 守卫语义就一行 */
longlong Item_func_trig_cond::val_int() {
  if (trig_var == nullptr) {
    // We don't use trigger conditions for IS_NOT_NULL_COMPL / FOUND_MATCH in
    // the iterator executor (except for figuring out which conditions are join
    // conditions and which are from WHERE), so we remove them whenever we can.
    ...
    assert(trig_type == IS_NOT_NULL_COMPL || trig_type == FOUND_MATCH);
    return args[0]->val_int();           /* ★ 8.0 迭代器执行器里守卫已被拆解 */
  }
  return *trig_var ? args[0]->val_int() : 1;   /* ★★ 守卫关 → 返回 1（TRUE，不过滤） */
}
```
```cpp
/* :7178 —— EXPLAIN / optimizer_trace 里 <if>(found_match(t2), ..., true) 的来源 */
void Item_func_trig_cond::print(...) const {
  switch (trig_type) {
    case IS_NOT_NULL_COMPL:      str->append("is_not_null_compl"); break;
    case FOUND_MATCH:            str->append("found_match"); break;
    case OUTER_FIELD_IS_NOT_NULL: str->append("outer_field_is_not_null"); break;
  }
  if (m_join != nullptr) {
    get_table_range(&first_table, &last_table);
    str->append("(");
    str->append(first_table->table->alias);
    if (first_table != last_table) { str->append(".."); str->append(last_table->table->alias); }
    str->append(")");
  }
```

**★ 两种守卫的工厂与"层层包"**（`sql/sql_optimizer.cc:8638`）：

```cpp
static Item *add_found_match_trig_cond(JOIN *join, plan_idx idx, Item *cond,
                                       plan_idx root_idx) {
  for (; idx != root_idx; idx = join->best_ref[idx]->first_upper()) {
                                         /* ★★ 嵌套外连接沿 first_upper() 层层包 */
    if (!(cond = new Item_func_trig_cond(cond, nullptr, join, idx,
                                         Item_func_trig_cond::FOUND_MATCH)))
      return nullptr;
    cond->quick_fix_field();
    cond->update_used_tables();          /* ★ 必须立刻刷新，否则下推会算错字段齐备性 */
  }
  return cond;
}
```
```cpp
/* sql_optimizer.cc:8655 —— ON 条件下推用 IS_NOT_NULL_COMPL */
  Item *cond = make_cond_for_table(thd, join_cond, const_table_map, table_map(0), false);
  if (cond) {
    if (!is_sj_mat_cond) {
      cond = new Item_func_trig_cond(cond, nullptr, this, first_inner,
                                     Item_func_trig_cond::IS_NOT_NULL_COMPL);
      ...
    }
    if (best_ref[first_inner]->and_with_condition(cond)) return true;
  }
```
> ★ **ON 条件用 `IS_NOT_NULL_COMPL`（补出来的 NULL 行不该再被 ON 判假）；下推到 inner 表的 WHERE 条件用 `FOUND_MATCH`。**

**★★★ ICP 对 trig_cond 的门禁：只认 `IS_NOT_NULL_COMPL`**（`sql/sql_optimizer.cc:6527`）：

```cpp
  Restrict some function types from being pushed down to storage engine:
  a) Don't push down the triggered conditions with exception for
  IS_NOT_NULL_COMPL trigger condition since the NULL-complemented rows are added
  at a later stage in the iterators, so we won't see NULL-complemented rows when
  evaluating it as an index condition. Nested outer joins execution code may
  need to evaluate a condition several times (both triggered and untriggered).
     TODO: Consider cloning the triggered condition and using the copies for ...
  b) Stored functions contain a statement that might start new operations (like
     DML statements) from within the storage engine. ...
  c) Subqueries might contain nested subqueries and involve more tables.
  d) Do not push down internal functions of type DD_INTERNAL_FUNC. When ICP is
     enabled, pushing internal functions to storage engine for evaluation will
     open data-dictionary tables. In InnoDB storage engine this will result in
     situation like recursive latching of same page by the same thread. ...
*/
bool uses_index_fields_only(Item *item, TABLE *tbl, uint keyno,
                            bool other_tbls_ok) {
  // Restrictions b and c.
  if (item->has_stored_program() || item->has_subquery()) return false;
  if (item->const_for_execution()) return true;
  ...
      if (func_type == Item_func::DD_INTERNAL_FUNC)  // Restriction d.
        return false;

      // Restriction a.
      if (func_type == Item_func::TRIG_COND_FUNC &&
          down_cast<Item_func_trig_cond *>(item_func)->get_trig_type() !=
              Item_func_trig_cond::IS_NOT_NULL_COMPL) {
        return false;                    /* ★★ FOUND_MATCH / OUTER_FIELD_IS_NOT_NULL 一律拒 */
      }
  ...
    case Item::FIELD_ITEM: {
      ...
      return item_field->field->part_of_key.is_set(keyno) &&
             item_field->field->type() != MYSQL_TYPE_GEOMETRY &&
             item_field->field->type() != MYSQL_TYPE_BLOB;
    }
    default:
      return false; /* Play it safe, don't push unknown non-const items */
  }
}
```
> ★★ **为什么 `IS_NOT_NULL_COMPL` 是唯一例外**：NULL-补行是在迭代器**更晚**的阶段才造出来的，引擎层根本看不到——所以"这行不是 NULL-补的"在 ICP 里恒真，可以放心推。
> ★ `DD_INTERNAL_FUNC` 推下去会去开数据字典表 → InnoDB **同页递归 latch**。这是四类限制里最"硬"的一条。

**★ `make_cond_remainder()`：AND 可剔、OR 不可剔**（`sql/sql_select.cc:2863`）：

```cpp
static Item *make_cond_remainder(Item *cond, bool exclude_index) {
  if (exclude_index && cond->marker == Item::MARKER_ICP_COND_USES_INDEX_ONLY)
    return nullptr; /* Already checked */    /* ★ marker = "已被 ICP 吃掉" */
  if (cond->type() == Item::COND_ITEM) {
    if (((Item_cond *)cond)->functype() == Item_func::COND_AND_FUNC) {
      ...
      while ((item = li++)) {
        Item *fix = make_cond_remainder(item, exclude_index);
        if (fix) { new_cond->argument_list()->push_back(fix); ... }   /* ★ AND：剔掉被吃的 */
      }
    } else {
      ...
      while ((item = li++)) {
        Item *fix = make_cond_remainder(item, false);
        if (!fix) return (Item *)nullptr;    /* ★★ OR：任一支被吃 → 整支 OR 都不能剔 */
        ...
      }
      new_cond->apply_is_true();             /* OR 节点补 IS TRUE 包装 */
    }
  }
  return cond;
}
```

### E. ICP：省的是回表 I/O，不是 CPU

不做 ICP 时的数据流：引擎定位索引条目 → **回表读整行** → 交给 server → server 过滤 → 丢弃。

**错位点**：过滤所需的信息（索引列的值）引擎早就有了，但它不知道要过滤什么；知道要过滤什么的 server，要等整行都读上来才能过滤。中间那次回表（二级索引 → 聚簇索引）对最终被丢弃的行是**纯浪费**。

InnoDB 侧的收益点：

```cpp
switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
  case ICP_NO_MATCH:    goto next_rec;          // ★ 不匹配，跳过回表
  case ICP_MATCH:       goto requires_clust_rec;
}
```

**★ "省的是回表 I/O"的源码级铁证**（`row0sel.cc:5396`，★ 注释原话）：

```cpp
      if (!srv_read_only_mode &&
          !lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
        /* We should look at the clustered index.
        However, as this is a non-locking read,
        we can skip the clustered index lookup if
        the condition does not match the secondary
        index entry. */
                                  /* ★★★ "本该去看聚簇索引，ICP 说不匹配就跳过" */
        switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
          case ICP_NO_MATCH:
            goto next_rec;        /* ★ 省掉一次聚簇索引 B 树查找（最贵的那部分 I/O） */
          case ICP_OUT_OF_RANGE:
            err = DB_RECORD_NOT_FOUND;
            goto idx_cond_failed; /* ★ 整段扫描提前终止 */
          case ICP_MATCH:
            goto requires_clust_rec;   /* 只有 MATCH 才真的回表 */
        }
      }
```

**★ 第二个收益：省锁**（`row0sel.cc:5461`，★ 常被漏掉）：

```cpp
  /* Check if the record matches the index condition. */
  switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
    case ICP_NO_MATCH:
      prebuilt->try_unlock(true);      /* ★★ ICP 不匹配的记录立刻解锁 */
      goto next_rec;
    case ICP_OUT_OF_RANGE:
      err = DB_RECORD_NOT_FOUND;
      prebuilt->try_unlock(true);
      goto idx_cond_failed;
    case ICP_MATCH:
      break;
  }
```

**★★ 为什么 CPU 一次没省**（`row0sel.cc:3780`，★ 物化 + 回调 server 的 Item 树）：

```cpp
static ICP_RESULT row_search_idx_cond_check(...) {
  if (!prebuilt->idx_cond) {
    return (ICP_MATCH);                  /* ★ 没推 ICP → 一律 MATCH（零开销短路） */
  }
  MONITOR_INC(MONITOR_ICP_ATTEMPTS);

  /* Convert to MySQL format those fields that are needed for
  evaluating the index condition. */
  for (i = 0; i < prebuilt->idx_cond_n_cols; i++) {
    const mysql_row_templ_t *templ = &prebuilt->mysql_template[i];
    if (templ->is_virtual) continue;
    if (!row_sel_store_mysql_field(mysql_rec, prebuilt, rec, prebuilt->index,
                                   prebuilt->index, offsets,
                                   templ->icp_rec_field_no, templ, ...)) {
      return (ICP_NO_MATCH);             /* 物化失败当作不匹配 */
    }
  }                                      /* ★ 只物化 ICP 需要的那几列（idx_cond_n_cols） */

  /* We assume that the index conditions on
  case-insensitive columns are case-insensitive. The
  case of such columns may be wrong in a secondary
  index, if the case of the column has been updated in
  the past, or a record has been deleted and a record
  inserted in a different case. */
  result = innobase_index_cond(prebuilt->m_mysql_handler);
                                         /* ★★ 回调 server 层的 Item 树求值 —— CPU 一次没省 */
  switch (result) {
    case ICP_MATCH:      MONITOR_INC(MONITOR_ICP_MATCH); return result;
    case ICP_NO_MATCH:   MONITOR_INC(MONITOR_ICP_NO_MATCH); return result;
    case ICP_OUT_OF_RANGE:
      MONITOR_INC(MONITOR_ICP_OUT_OF_RANGE);
      const auto record_buffer = row_sel_get_record_buffer(prebuilt);
      if (record_buffer) record_buffer->set_out_of_range(true);
      return result;
  }
}
```
> ★★ 关键在于 `innobase_index_cond()` 里是 `h->pushed_idx_cond->val_int()` —— **下推的是一棵活的 Item 树，比较逻辑仍由 server 执行**。所以 ICP 省的是"回表"与"锁"，比较次数一次不少。

**★ 四个 ICP 计数器**（`storage/innobase/srv/srv0mon.cc:1344`，★ `SHOW ENGINE INNODB STATUS` 里能看到）：

```cpp
    {"icp_attempts",     "icp", "Number of attempts for index push-down condition checks", ...},
    {"icp_no_match",     "icp", "Index push-down condition does not match", ...},
    {"icp_out_of_range", "icp", "Index push-down condition out of range", ...},
    {"icp_match",        "icp", "Index push-down condition matches", ...},
```
> ★ `icp_no_match / icp_attempts` 就是**省下的回表比例**——这是评估 ICP 收益最直接的指标。

**一个常被忽略的反向证据：覆盖索引时根本不做 ICP。**

```cpp
case JT_REF:
  if (table->covering_keys.is_set(qep_tab->ref().key) && !table->no_keyread)
    table->set_keyread(true);              // ★ 覆盖索引 → keyread，不做 ICP
  else
    qep_tab->push_index_cond(tab, qep_tab->ref().key, &trace_refine_table);
```

这不是遗漏，而是**ICP 的收益来源本身就是"避免回表"**——覆盖索引已经不回表了，ICP 没有可省的东西（反而多一次 `val_int()`）。**这一行反向证明了：ICP 省的是回表 I/O，不是 CPU。**

**ICP 与普通下推是两级火箭**，层次不同：

| | 普通下推 | ICP |
|---|---|---|
| 移动的边界 | server 层内部（算子树之间） | **跨 server ↔ engine 边界** |
| 收益 | 减少后续算子**输入行数** | 减少**回表 I/O**（结果集行数不变） |
| 代价模型看见吗 | 看见（`filter_effect`） | **看不见**（在 `make_join_readinfo` 之后才设，join order 早定了） |

```
WHERE（join 之上）
  →[普通下推]→ tab->condition()（挂到最早能求值的表）
  →[ICP]→ 引擎（在索引条目上求值，不满足就不回表）
```

**还有第三层 `engine_condition_pushdown`（ECP）**，与 ICP 完全不是一回事：它推的是**整个条件**（不限于索引列），接口 `handler::cond_push`；触发点只有**单表 UPDATE/DELETE**（SELECT 不走），主要实现方是 NDB。

**ICP 的 8 项判定**里几条值得注意：非聚簇主键才做（*"The performance improvement of pushing an index condition on a clustered key is much lower"*——主键扫描"索引即数据"，ICP 无可省）；虚拟生成列上的索引不支持；`reversed_access` 直接 return（注释 "historical limitation, lift it!"）。

### F. 理论溯源与参数

**老优化器没有任何论文引用**，但教科书术语被直接写进了注释，本身就是溯源证据：

| 优化 | 源码里的术语痕迹 | 理论源头 |
|---|---|---|
| 等值传播 | *"join transitive closure"* / *"search argument transitive closure"* | 查询重写传递闭包；SARGable 是 System R 时代的术语 |
| 传递闭包算法 | `propagate_dependencies` 注释：*"**Warshall's algorithm** is used to build the transitive closure"* | Warshall 1962 |
| 谓词下推 | *"attached to the first table in the join order where all necessary fields are available"* | System R（Selinger 1979）的 "push selects down" |
| 常量折叠 | `basic_const_item()` / `is_non_const_over_literals` 的分层 | 编译器的常量传播 + 副作用分析 |

**反差很有意思**：8.0 新写的 **hypergraph 优化器大量引用论文**（`[Neu04]`、`[Neu04b]`、`[Sim96]`、`[Moe13]`、DPhyp、Vance & Maier 1995）。⇒ **老优化器是"工程驱动、无文献"的，新优化器是"论文驱动"的**——这是 8.0 两套优化器并存留下的明显风格断层。

**三个开关默认都是 on**：

| 开关 | bit | 默认 | 作用 |
|---|---|---|---|
| `engine_condition_pushdown` | `1<<4` | **on** | ECP：整个条件下推给引擎，**只在单表 UPDATE/DELETE 生效**，主要给 NDB |
| `index_condition_pushdown` | `1<<5` | **on** | ICP：`idx_cond_push`，hint 名 `NO_ICP(t1 idx)` |
| `condition_fanout_filter` | `1<<17` | **on** | **不是谓词优化开关，是估算开关**——关掉后 `filter_effect` 不参与，退化为 `quick_condition_rows` |

**★ `optimizer_switch` 全 27 个 flag**（`sql/sys_vars.cc:3443`，★ 注释要求与 `sql_const.h` 严格同序）：

```cpp
/**
  @note
  @b BEWARE! These must have the same order as the \#defines in sql_const.h!
*/
static const char *optimizer_switch_names[] = {
    "index_merge", "index_merge_union", "index_merge_sort_union",
    "index_merge_intersection",
    "engine_condition_pushdown",      /* ★ ECP：整个条件下推给引擎（主要给 NDB） */
    "index_condition_pushdown",       /* ★ ICP */
    "mrr", "mrr_cost_based", "block_nested_loop", "batched_key_access",
    "materialization", "semijoin", "loosescan", "firstmatch",
    "duplicateweedout", "subquery_materialization_cost_based",
    "use_index_extensions", "condition_fanout_filter", "derived_merge",
    "use_invisible_indexes", "skip_scan", "hash_join",
    "subquery_to_derived", "prefer_ordering_index",
    "hypergraph_optimizer",  // Deliberately not documented below.
    "derived_condition_pushdown",     /* ★ derived 条件下推（8.0.22+） */
    "default", NullS};
```
```cpp
/* sql/sql_const.h:194 —— bit 定义 */
constexpr const uint64_t OPTIMIZER_SWITCH_ENGINE_CONDITION_PUSHDOWN{1ULL << 4};
constexpr const uint64_t OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN{1ULL << 5};
```
> ⚠️ **`engine_condition_pushdown` 与 `index_condition_pushdown` 是两个不同的开关**，EXPLAIN 里也对应两个不同的 tag：`ET_USING_PUSHED_CONDITION`（"Using pushed condition"）vs `ET_USING_INDEX_CONDITION`（"Using index condition"）。

**★ `NO_ICP` 是 key-level hint**（`sql/opt_hints.h:64` / `sql/sql_hints.yy:579`）：

```cpp
enum opt_hints_enum {
  BKA_HINT_ENUM = 0,
  BNL_HINT_ENUM,
  ICP_HINT_ENUM,        /* ★ 存在 */
  MRR_HINT_ENUM,
  ...
```
```cpp
key_level_hint_type_off:
          NO_ICP_HINT
          {
            $$= ICP_HINT_ENUM;         /* ★ 归在 key_level 下 → 按 (table, keyno) 生效 */
          }
```
> ★ 所以写法是 `NO_ICP(t1 idx_a)`，不是 `NO_ICP(t1)`。
> ★ 判定合并点：`hint_key_state(thd, table_ref, keyno, ICP_HINT_ENUM, OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN)`（`sql_select.cc:2987`）。

**★ 理论溯源：教科书术语的源码出处**（F 节，★ 老优化器"无文献但有术语"）：

| 理论名称 | 源码对应 | 位置 |
|---|---|---|
| **join transitive closure** | `Item_equal` 类注释："When feature 1 is supported they say that join transitive closure is employed." | `sql/item_cmpfunc.h:2539` |
| **sargable transitive closure** | 同注释："When feature 2 is supported they say that search argument transitive closure is employed." | `sql/item_cmpfunc.h:2542` |
| **selection / predicate pushdown** | `make_cond_for_table()` "attached to the first table in the join order where all necessary fields are available" | `sql/sql_optimizer.cc:9461` |
| **guarded predicate**（守卫谓词） | `Item_func_trig_cond` 类注释："val(P/g)= g ? val(P):true" | `sql/item_cmpfunc.h:790` |
| **covering index 判定** | `uses_index_fields_only()` 的 `part_of_key.is_set(keyno)` | `sql/sql_optimizer.cc:6568` |
| **hypergraph 的 FILTER 下推** | `PushDownCondition()`（★ 新优化器独有，老优化器没这个函数） | `sql/join_optimizer/make_join_hypergraph.cc:1643` |

```cpp
/* sql/item_cmpfunc.h:2539 —— ★ 两个术语的原文 */
  When feature 1 is supported they say that join transitive closure
  is employed.
  When feature 2 is supported they say that search argument transitive
  closure is employed.
```
> ★ hypergraph 里复用同一个 `make_cond_for_table`，只是"表集合"换成了 hyperedge 的 `tables_in_subtree`：
> ```cpp
>     Item *partial_cond = make_cond_for_table(
>         current_thd, cond, expr->left->tables_in_subtree, /*used_table=*/0,
>         /*exclude_expensive_cond=*/true);
>     if (partial_cond != nullptr) {
>       PushDownCondition(partial_cond, expr->left, ...);
>     }
> ```

### G. 一条贯穿全章的元原则

> **"推导"和"移动"是廉价的、收益乘性的；"物化"是昂贵的、收益一次性的。** 所以：能用结构（等价类、table_map 位图）延迟决策的，就不要提前物化；能证明某个变换安全的，才做；证明不了的，加守卫（`Item_func_trig_cond`、trig_cond 白名单、`is_non_const_over_literals`）而不是放弃。

对应到代码：等值传播不物化 n² 个谓词；不预先加闭包谓词（动态替换）；常量折叠不单开 pass（挂在 `fix_fields` 上）；下推不复制条件（用位运算判定"最早求值点"）；外连接下推 + trig_cond 守卫；ICP 不进代价模型（它是纯执行期收益，做了就是白赚）。

**★ 语义不变性的三处源码"誓言"**（★ 元原则的硬证据）：

**① `fold_condition` 的 Doxygen**（`sql/sql_const_folding.h:47`）：

```cpp
  The constants are always converted to constants of the type of the field, no
  widening of the field type in therefore necessary after this folding for
  comparison to occur. When converting constants to decimal (when the field is a
  decimal), the constant will receive the same number of fractional decimals as
  the field. If decimal constant fraction truncation occurs, the comparison
  {GT,GE,LT,LE} logic is adjusted to preserve semantics without widening the
  field's type.
                                  /* ★★ "adjusted to preserve semantics" */
```

**② 改动算子时必须"补偿"截断**（`sql/sql_const_folding.cc:1527`，★ 一个具体例子）：

```cpp
        case RP_INSIDE_TRUNCATED:
          /*
            The constant now has a smaller absolute value.
            Adjust comparison functions to compensate for this. For example,
            if we have a decimal field f of type DECIMAL(3,1) and compare:
                   f >= 10.13
            we adjust the predicate to
                   f > 10.1
            since we know the value of f has to be either 10.1 or 10.2
            in this "vicinity". If it is 10.1, we need to remove the equal part
            of the comparison to preserve semantics. ...
          */
          if (adjust_cmp_op(thd, func, left_has_field, negative, ref_or_field,
                            *c, retcond))
            return true;
```
> ★★ 这就是"折叠必须保持语义"的代价：**常量截断后要同步改算子**（`>=` → `>`），否则会多算/少算边界值。

**③ `Item_equal::cast_incompatible_args` 的断言**（`sql/item_cmpfunc.h:2603`）：

```cpp
  bool cast_incompatible_args(uchar *) override {
    // Multiple equality nodes (Item_equal) should have been
    // converted back to simple equalities (Item_func_eq) by
    // substitute_for_best_equal_field before cast nodes are injected.
    assert(false);
    return false;
  }
```
> ★★ 这是一条**编译期契约**：`Item_equal` 必须在注入 cast 节点前被拆回 `Item_func_eq`。违反就直接 crash——注释即规范。

**★ 三种原子操作的统一框架：`walk` / `transform` / `compile`**（`sql/item.h:2472`，★ A 节核心）：

```cpp
  /**
    Perform a generic transformation of the Item tree, by adding zero or
    more additional Item objects to it.
    ...
    Item::transform() should handle all transformations during preparation.
    Notice that all transformations are permanent; they are not rolled back.

    Use Item::compile() to perform transformations during optimization.
  */
  virtual Item *transform(Item_transformer transformer, uchar *arg);
```
```cpp
  /**
    ...
    @code
    compile()
    {
      if (this->*some_analyzer(...))
      {
        compile children if any;
        return this->*some_transformer(...);
      }
      else
        return this;
    }
    @endcode

    i.e. analysis is performed top-down while transformation is done
    bottom-up. If no transformation is applied, the item is returned unchanged.
    ...
    The function is supposed to be used during the optimization stage of
    query execution. All new allocations are recorded using
    THD::change_item_tree() so that they can be rolled back after execution.
  */
  virtual Item *compile(Item_analyzer analyzer, uchar **arg_p,
                        Item_transformer transformer, uchar *arg_t) {
    if ((this->*analyzer)(arg_p)) return ((this->*transformer)(arg_t));
    return this;
  }
```
> ★★ 分工极清晰：
> - `transform()` = **prepare 阶段**，永久、不可回滚（折叠用）
> - `compile()` = **optimize 阶段**，analyzer 自上而下剪枝 + transformer 自下而上替换，用 `change_item_tree` 登记以便 PS/SP 回滚（等值替换、ICP 用）
> - `walk()` = 只读遍历（检查用）

```cpp
/* sql/item.cc:779 —— 基类 transform 只有一行，递归在各子类的实现里 */
Item *Item::transform(Item_transformer transformer, uchar *arg) {
  return (this->*transformer)(arg);
}
```
```cpp
/* sql/item_cmpfunc.cc:5775 —— Item_cond 的递归：AND/OR 层逐子项替换 */
Item *Item_cond::transform(Item_transformer transformer, uchar *arg) {
  List_iterator<Item> li(list);
  Item *item;
  while ((item = li++)) {
    Item *new_item = item->transform(transformer, arg);
    if (new_item == nullptr) return nullptr;
    if (new_item != item) li.replace(new_item);
  }
  return Item_func::transform(transformer, arg);
}
```
> ★ 等值传播的实际调用点（`sql/sql_optimizer.cc:4373`）：
> ```cpp
>       cond = cond->compile(&Item::subst_argument_checker, &is_subst_valid,
>                            &Item::equal_fields_propagator, (uchar *)inherited);
> ```
> ★ `subst_argument_checker` 有一条重要限制（`sql/item.cc:6043`）：
> ```cpp
> bool Item_field::subst_argument_checker(uchar **arg) {
>   return (result_type() != STRING_RESULT) || (*arg);
> }                                  /* ★★ 字符串类型默认不做等值替换（collation 风险） */
> ```
> ⚠️ `Item::replace_item()` **不存在**。

---

## 零、流水线与常见误解

```
prepare 期（每次 prepare 一次）
  ├─ Item_cond::fix_fields → remove_const_conds    item_cmpfunc.cc:5595/5676
  ├─ simplify_joins（OJ→IJ, ON→WHERE）              sql_resolver.cc:1951（04 篇）
  ├─ simplify_const_condition                      sql_resolver.cc:934
  └─ push_conditions_to_derived_tables             sql_resolver.cc:620（由 :834 调用）

optimize 期（每次 execute 都重做）
  ├─ optimize_cond(WHERE)                          sql_optimizer.cc:474
  │    ├─ (a) build_equal_items          等值传播    :4467
  │    ├─ (b) propagate_cond_constants   常量传播    :4945
  │    └─ (c) remove_eq_conds            平凡条件消除 :10440（内含 fold_condition）
  ├─ optimize_cond(HAVING)                         :487
  ├─ make_join_plan / update_ref_and_keys（用 Item_equal 展开 Key_use）
  ├─ substitute_for_best_equal_field(WHERE)        :736   ★ 需 join 顺序先定
  ├─ substitute_for_best_equal_field(每个 join_cond) :753
  ├─ make_join_query_block                         :781
  │    ├─ add_not_null_conds                       :6394
  │    ├─ make_cond_for_table(const_table_map)     :9610 → const_cond->val_int() :9622
  │    ├─ 主循环 make_cond_for_table(prefix, added) :9675
  │    └─ JOIN::attach_join_conditions             :8776
  └─ make_join_readinfo → QEP_TAB::push_index_cond :1020 → sql_select.cc:2920（ICP）
```

> ⚠️ **五处名称/位置勘误**：
> - `Item_equal::m_fields` **不存在**，成员是 `List<Item_field> fields`（`item_cmpfunc.h:2563`）
> - `Item_equal_iterator` 8.0 已删除（5.7 才有），改用 `get_fields()` + range-for
> - `eval_const_cond()` 是**自由函数**（`item_func.cc:306`），不是 Item_equal 成员
> - `make_cond_after_sjm` 8.0 已删除，功能拆成三处（见 3.3）
> - `make_join_select` 已改名 **`make_join_query_block`**（`sql_optimizer.cc:9579`）

---

## 一、等值传播（Item_equal 等价类）

### 1.1 Item_equal 结构（`item_cmpfunc.h:2561-2699`）

```cpp
class Item_equal final : public Item_bool_func {
  List<Item_field> fields;        // :2563  等价类成员（只能是 Item_field）
  Item *m_const_arg{nullptr};     // :2565  唯一的常量代表元
  cmp_item *eval_item{nullptr};   // :2567  val_int() 的比较器
  Arg_comparator cmp;             // :2569  compare_const() 用
  bool cond_false{false};         // :2571  ★ 两个不等常量 ⇒ 恒假
  bool compare_as_dates{false};   // :2573
  Item *m_const_folding[2];       // :2695  常量折叠临时区
  enum Functype functype() const override { return MULT_EQUAL_FUNC; }
};
```

一个 `Item_equal` 就是一个**等价类**：`fields` 里所有字段两两相等，`m_const_arg` 存常量（最多一个）。它不是 `a=b AND b=c` 的语法树，而是一个单节点，`print()` 输出 `multiple equal(a, b, c)`。

### 1.2 build_equal_items（`sql_optimizer.cc:4467`）完整算法

**是 union-find 吗？不是。** 是 **"线性 find + 立即 eager union（链表 concat）"**，没有 rank/路径压缩。

```cpp
bool build_equal_items(THD *thd, Item *cond, Item **retcond,
                       COND_EQUAL *inherited, bool do_inherit,
                       mem_root_deque<Table_ref *> *join_list,
                       COND_EQUAL **cond_equal_ref) {
  if (cond) {
    if (build_equal_items_for_cond(thd, cond, &cond, inherited, do_inherit))
      return true;
    cond->update_used_tables();
    if (顶层是 AND)
      cond_equal = &down_cast<Item_cond_and *>(cond)->cond_equal;   // ★ 挂在 AND 节点上
    else if (整个 WHERE 就是一个多等值) { ... }
  }
  if (cond_equal) { cond_equal->upper_levels = inherited; inherited = cond_equal; }

  if (join_list) {                            // ★ 递归下沉到 ON 条件
    for (Table_ref *table : *join_list) {
      if (table->join_cond_optim()) {
        build_equal_items(thd, table->join_cond_optim(), &join_cond,
                          inherited, do_inherit, nested_join_list, &table->cond_equal);
        table->set_join_cond_optim(join_cond);
      }
    }
  }
}
```

`COND_EQUAL`（`item_cmpfunc.h:2701`）：
```cpp
class COND_EQUAL {
  uint max_members;
  COND_EQUAL *upper_levels;        // 指向外层 AND 层 —— 形成"继承链"
  List<Item_equal> current_level;  // 本 AND 层的所有等价类
};
```
`Item_cond_and` 自带 `COND_EQUAL cond_equal`（`:2713`）。**等价类按 AND 层挂在 AND 节点上，通过 `upper_levels` 串成从内层到外层的链表。**

**递归体 `build_equal_items_for_cond`（`:4237`）—— 两趟扫描**：

```cpp
if (and_level) {
  while ((item = li++)) {                              // ① 第一趟：只扫本层合取项
    if (check_equality(thd, item, &cond_equal, &eq_list, &equality)) return true;
    if (equality) li.remove();                         // ② 被吸收的 = 从 AND 摘掉
  }
  ...
  item_cond_and->cond_equal = cond_equal;              // ⑤ 挂到 AND 节点
  inherited = &item_cond_and->cond_equal;              // ⑥ 下层继承本层
}
li.rewind();
while ((item = li++)) {                                // ⑦ 第二趟：递归子项
  build_equal_items_for_cond(thd, item, &new_item, inherited, do_inherit);
  if (new_item != item) li.replace(new_item);
}
if (and_level) {
  args->concat(&eq_list);
  args->concat((List<Item> *)&cond_equal.current_level);   // ⑨ ★ 塞回 AND 参数列表
}
```

- **为什么先扫完本层再递归下层**：注释 `:4218-4225`——"lower AND levels need to know about all possible Item_equal objects in upper levels"。下层 OR 分支里的 `b=c` 需要看到上层已建好的 `=(a,b)` 才能合并。
- **⑨ 极重要**：`Item_equal` 被当作普通合取项**塞回 AND 的参数列表**。所以优化后 WHERE 形态是 `AND(其他谓词..., Item_equal(...), ...)`，且 `cond_equal.current_level` 与 `argument_list()` **同时持有**这些指针。
- **join_list 循环**（ON 继承）：`(t1,t2) LEFT JOIN (t3,t4) ON t1.a=t3.a AND t2.a=t4.a WHERE t1.a=t2.a`，ON 继承 `=(t1.a,t2.a)` 后可推出 `=(t1.a,t2.a,t3.a,t4.a)`。注意用 `join_cond_optim()`（优化期副本），不污染永久的 `join_cond()`。

**复杂度**：`find_item_equal` 是线性查找 O(d·E·m)（d=继承链深、E=等价类数、m=平均成员数），总约 O(k·d·E·m)。

### 1.3 等价类合并的四种 case（`check_simple_equality`，`:3864-3931`）

```cpp
if (left_field->eq(right_field)) {          // f = f 特例
  // (NULL = NULL) 是 UNKNOWN 不是 TRUE
  *simple_equality = !((left_field->is_nullable() ||
                        left_field->table->is_nullable()) && !left_item_equal);
  return false;
}
if (left_item_equal && left_item_equal == right_item_equal) {
  *simple_equality = true;                  // 已在同一类 ⇒ 冗余，丢弃
  return false;
}
if (left_copyfl) {                          // ★ 来自上层：写时复制
  left_item_equal = new Item_equal(left_item_equal);
  cond_equal->current_level.push_back(left_item_equal);
}
if (right_copyfl) { /* 同理 */ }

if (left_item_equal) {
  if (!right_item_equal)
    left_item_equal->add(right_item);                       // Case 2
  else {
    left_item_equal->merge(thd, right_item_equal);          // Case 4: UNION
    // 从 current_level 摘除被吞并者
  }
} else {
  if (right_item_equal) right_item_equal->add(left_item);    // Case 3
  else { new Item_equal(left_item, right_item); ... }         // Case 1
}
```

| left 找到? | right 找到? | 动作 | 代价 |
|---|---|---|---|
| 否 | 否 | `new Item_equal(l,r)` | O(1) |
| 是 | 否 | `left->add(r)` | O(1) |
| 否 | 是 | `right->add(l)` | O(1) |
| 是 | 是（不同类） | `left->merge(right)` + 摘除 right | concat O(1)，摘除 O(E) |
| 是 | 是（同类） | 谓词冗余，丢弃 | — |

**`copyfl`（copy-flag）是 8.0 特有的"跨层写时复制"**：`find_item_equal` 沿 `upper_levels` 向上找，若命中的是外层的 `Item_equal`，**不能就地修改**（外层 AND 的其他兄弟分支还要用它），必须 `new Item_equal(外层对象)` 拷一份到本层。

**`f = f` 不能省**：`(NULL = NULL)` 是 UNKNOWN。只有字段 NOT NULL 且表不是 nullable 时才能把 `f=f` 当恒真丢弃。

**加入等价类的前置门槛**（`:3862`）：
```cpp
if (!left_field->eq_def(right_field)) return false;
```
**只有类型定义完全相同（`Field::eq_def`）的字段才允许进同一个多等值**——这正是 `propagate_cond_constants` 还须存在的理由（见 2.4）。

### 1.4 `t1.a = t2.b AND t2.b = 5` ⇒ `t1.a = 5`（三条路径）

**路径 1（主路径）**：`Item_equal` 天然携带常量
1. `check_equality(t1.a = t2.b)` → Case 1 → `Item_equal(t1.a, t2.b)`
2. `check_equality(t2.b = 5)` 走 field=const 分支（`:3934-4035`）→ `add(thd, 5, t2.b)` → `m_const_arg = 5`
3. 结果 `Item_equal(5, t1.a, t2.b)`，物化成真正的谓词在 `eliminate_item_equal`（`:4606`）

**常量合并与矛盾检测**（`Item_equal::compare_const`，`item_cmpfunc.cc:6741`）：
```cpp
bool Item_equal::compare_const(THD *thd, Item *c) {
  ...
  Item_func_eq *func = new Item_func_eq(c, m_const_arg);
  func->set_cmp_func();
  cond_false = !func->val_int();            // ★ 优化期直接求值 c = m_const_arg
  if (cond_false) used_tables_cache = 0;    // 恒假 ⇒ 变常量项，可被上层摘除
}
```
**`a=1 AND a=2` 恒假就是这里检测出来的**——同时实现了常量传播与矛盾检测。

**`eliminate_item_equal` 的星形展开**（`:4606`）：
```cpp
Item *const item_const = item_equal->const_arg();
auto it = item_equal->get_fields().begin();
if (!item_const) it++;                    // 无常量：第2..n个字段与第1个配对
while (it != ...) {
  Item_field *item_field = &*it++;
  ...
  Item *const head = item_const ? item_const : item_equal->get_subst_item(item_field);
  eq_item = new Item_func_eq(item_field, head);       // ★ 生成 t1.a = 5 / t2.b = 5
  if (item_const != nullptr) {
    eq_item->apply_is_true();
    if (fold_condition(thd, eq_item, &eq_item, &res)) ...   // ★ 立即折叠
    if (res == Item::COND_FALSE) return new Item_func_false();
  }
}
```
**有常量时每个 field 都与常量配对**（不是与首字段配对），所以 `Item_equal(5, t1.a, t2.b)` 展开成 `t1.a = 5 AND t2.b = 5`——**这就是 `t1.a = 5` 的产出点**。
副作用：此时**不会**生成 `t1.a = t2.b`，`contains_only_equi_join_condition()` 返回 false（`const_arg()==nullptr` 才算纯等值连接），hash join 不能用它当连接键。

**路径 2**：`propagate_cond_constants`（见 2.4）——处理被 `eq_def` 或校对集/类型规则拒绝、没能进 Item_equal 的等值。

**路径 3**：const 表读出后（`sql_executor.cc:466` `update_const_equal_items`）→ `Item_equal::update_const`（`item_cmpfunc.cc:6848`）：
```cpp
while ((item = it++)) {
  if (item->const_item() && !item->is_outer_field()) {   // ★ 外连接列不传播
    it.remove();
    if (add(thd, item)) return true;                     // 变成 m_const_arg
  }
}
```
外连接列的 const 状态可能来自"空表→NULL"或"单行表→值 or NULL"，两者都不能当常量。

### 1.5 substitute_for_best_equal_field（`:4773`）——为什么"选最好的字段"

```cpp
if (and_level) {
  cond_list->disjoin((List<Item> *)&cond_equal->current_level);   // ① 摘出 Item_equal
  auto cmp = [table_join_idx](Item_field *f1, Item_field *f2) {
    return compare_fields_by_table_order(f1, f2, table_join_idx);
  };
  while ((item_equal = it++)) item_equal->sort(cmp);              // ② 按连接顺序排序
}
... 递归处理其余谓词 ...
while ((item_equal = it++))                                       // ④ 展开成简单等值
  cond = eliminate_item_equal(thd, cond, cond_equal->upper_levels, item_equal);
```

**"最好"= 连接顺序中最早出现的表的字段**（`compare_fields_by_table_order`，`:4537`）：外层引用最优，否则比 `idx()`。

**为什么必须选最好**：`t1 JOIN t2 ON t1.a=t2.a WHERE t2.a > 100`，顺序 t1→t2。把 `t2.a > 100` 改写成 `t1.a > 100`，则它的 `used_tables()` 只含 t1，`make_cond_for_table` 就能把它下推到 **t1**——读 t1 时就过滤，而不是等 t2 join 完。这是 **SARG 传递闭包 + 谓词提前** 的组合拳。

**调用点 `:736` 必须在 join 顺序定完之后**——"最优字段"依赖访问顺序。

排序算法是 `List::sort`（`sql_list.h:533`）——**O(n²) 交换排序，只交换 info 指针不动节点**，目的是让排序前创建的迭代器排序后仍可安全使用。

### 1.6 多等值 `a=b=c` 与 Key_use 的 O(n²) 展开

`update_ref_and_keys` → `add_key_fields` 的 `OPTIMIZE_EQUAL` 分支（`:7722`）：
```cpp
if (const_item) {
  // 有常量：n 个 (field, const) 对 —— O(n)
  for (Item_field &item : item_equal->get_fields())
    add_key_field(..., &item, true, &const_item, 1, ...);
} else {
  // 无常量：所有有序对 (field_i, field_j), i≠j —— O(n²)
  for (Item_field &outer : item_equal->get_fields())
    for (Item_field &inner : item_equal->get_fields())
      if (!outer.field->eq(inner.field))
        add_key_field(..., &outer, true, &inner_ptr, 1, ...);
}
```
`Item_equal` 语法上只有 1 个节点，但为让**任意两表之间**都能被识别为可 ref 访问，必须为 n(n-1) 个有序对各生成 Key_field。`=(t1.a,t2.a,t3.a,t4.a)` 生成 12 个候选——这就是 `max_equal_elems`（`:4289`）被记录的原因（用于预估数组大小）。

范围优化器侧（`range_analysis.cc:971`）同样展开：无 `const_arg` 的多等值对 range 无用（`if (value == nullptr) return nullptr`）。

---

## 二、常量折叠

> ⚠️ **最重要的纠正**：**MySQL 8.0.39 没有通用算术折叠器**。`Item_func_plus` 没有任何"若参数全常量则替换自身为字面量"的逻辑——**`WHERE int_col > 1+2` 不会被改写成 `int_col > 3`**。

### 2.1 8.0 的"常量折叠"实际由五处机制组成

| 阶段 | 机制 | 位置 | 做什么 |
|---|---|---|---|
| resolve/fix_fields | `Item_cond::fix_fields` → `remove_const_conds` | `item_cmpfunc.cc:5590/5676` | 把 AND/OR 里**整个常量布尔合取项**求值，替换为 `Item_func_true/false` 或直接删除 |
| resolve | `Arg_comparator::set_cmp_func` → `cache_converted_constant` | `item_cmpfunc.cc:1466` | 常量类型与比较类型不同时包 `Item_cache` 并 `setup()` 预取值 → **每行不再重复类型转换**（这是"折叠"的性能实质） |
| resolve | `simplify_const_condition` | `sql_resolver.cc:934` | 整个 WHERE/HAVING/ON 是常量时 → nullptr / true / false |
| optimize | `propagate_cond_constants` → `resolve_const_item` | `sql_optimizer.cc:4945` / `item.cc:9317` | **唯一把常量表达式物化成字面量 Item 的地方** |
| optimize | `remove_eq_conds` → `eval_const_cond` / `fold_condition` | `:10440` / `item_func.cc:306` / `sql_const_folding.cc:1255` | 布尔短路 + `field cmp const` 的**类型域折叠** |

```cpp
// sql/item_cmpfunc.cc:1466
static Item **cache_converted_constant(THD *thd, Item **value,
                                       Item **cache_item, Item_result type) {
  if (!(thd->lex->context_analysis_only & CONTEXT_ANALYSIS_ONLY_VIEW) &&
      (*value)->const_for_execution() && type != (*value)->result_type()) {
    Item_cache *cache = Item_cache::get_cache(*value, type);
    cache->setup(*value);                    // ★ 预取值
    *cache_item = cache;
    return cache_item;
  }
  return value;
}
```
这就是 `WHERE int_col = '3'` 只做一次字符串→整数转换的原因。

### 2.2 `const_item()` 靠 used_tables 位图，没有独立标记

```cpp
// sql/item.h:2281
/**
  An expression is constant if it:
  - refers no tables.  - refers no subqueries that refers any tables.
  - refers no non-deterministic functions.  - refers no statement parameters.
*/
bool const_item() const { return (used_tables() == 0); }
```

**关键设计**："非确定性"和"参数"不是靠 flag，而是靠**伪表位**（`sql_const.h` 的 `RAND_TABLE_BIT`/`INNER_TABLE_BIT`/`OUTER_REF_TABLE_BIT`）：
- `Item_param::used_tables()` 返回 `INNER_TABLE_BIT` → `const_item()` 为 false，但 `const_for_execution()` 为 true
- `RAND()` 通过 `get_initial_pseudo_tables()` 返回 `RAND_TABLE_BIT`

```cpp
void Item_func::update_used_tables() {
  used_tables_cache = get_initial_pseudo_tables();     // ← 伪表位在此注入
  not_null_tables_cache = 0;
  for (uint i = 0; i < arg_count; i++) {
    args[i]->update_used_tables();
    used_tables_cache |= args[i]->used_tables();
    if (null_on_null) not_null_tables_cache |= args[i]->not_null_tables();
  }
}
```

`Item_equal::update_used_tables()`（`item_cmpfunc.cc:7004`）有个重要特例：`if (cond_false) return;` —— 恒假时 `used_tables()==0`，于是 `const_item()` 为真，`eliminate_item_equal` 的 `if (item_equal->const_item() && !item_equal->val_int())` 就能捕获它返回 `Item_func_false`。

### 2.3 fold_condition 实际做什么（`sql_const_folding.cc:1255`）

**只处理 `<field> <cmp> <const>` 形态**，而且 `<const>` 必须是 `basic_const_item`、一元负号、或 datetime literal：

```cpp
// :1340
} else if (args[i]->const_for_execution() && type != Item::SUBSELECT_ITEM) {
  if (type != Item::FUNC_ITEM) seen_constant = true;
  else {
    /* We only try to fold two kinds of functions here:
       1) Monadic minus, 2) Item_datetime_literal. */
    const auto f = down_cast<Item_func *>(args[i]);
    if ((ft == Item_func::NEG_FUNC && f->arguments()[0]->basic_const_item()) ||
        ft == Item_func::DATETIME_LITERAL)
      seen_constant = true;
  }
}
if (!(seen_field && seen_constant)) return fold_arguments(thd, func);  // 否则只递归
```

**它真正做的是"类型域折叠 + 比较符规范化"**，靠 `Range_placement` 枚举：

| placement | 效果 |
|---|---|
| `RP_OUTSIDE_HIGH/LOW` | `tinyint_col = 999` → 恒假（可空列转 `col <> col` 保持 NULL 语义） |
| `RP_ON_MIN/ON_MAX` | `tinyint_col <= -128` → 化简成 `= -128`（**范围扫描降级为等值查找**） |
| `RP_ROUNDED_DOWN/UP` | FLOAT/DECIMAL 精度截断后改比较符：`float(5,2) < 123.123` → `<= 123.12` |
| `RP_INSIDE_TRUNCATED/YEAR_HOLE` | 落入类型"空洞"（如 year 0）⇒ 恒假 |

`fold_condition` 的调用者只有两处：`fold_condition_exec`（`sql_optimizer.cc:10418`）和 `eliminate_item_equal`（`:4712`）。

### 2.4 propagate_cond_constants（`sql_optimizer.cc:4945`）—— 两轮传播

```cpp
if (cond->type() == Item::COND_ITEM) {
  while ((item = li++))
    propagate_cond_constants(thd, &save, and_level ? cond : item, item);  // ① 递归下降
  if (and_level) {
    while ((cond_cmp = cond_itr++)) {                                     // ③ 第二轮
      Item **args = cond_cmp->cmp_func->arguments();
      if (!args[0]->const_item())
        change_cond_ref_to_const(thd, &save, cond_cmp->and_level,
                                 cond_cmp->and_level, args[0], args[1]);
    }
  }
} else if (and_father != cond && cond->marker != Item::MARKER_CONST_PROPAG) {
  if (是 EQ_FUNC 或 EQUAL_FUNC) {
    if (right_const) {
      Item *item = args[1];
      resolve_const_item(thd, &item, args[0]);        // ⑥ ★ 物化成字面量
      thd->change_item_tree(&args[1], item);
      func->update_used_tables();
      change_cond_ref_to_cond(thd, save_list, and_father, and_father, args[0], args[1]);  // ⑦
    }
  }
}
```

**`resolve_const_item`（`item.cc:9317`）—— 唯一物化字面量的地方**：
```cpp
if (item->basic_const_item()) return false;      // 已经是字面量
switch (res_type) {
  case INT_RESULT: {
    longlong result = item->val_int();           // ★ 优化期求值
    new_item = item->unsigned_flag ? new Item_uint(...) : new Item_int(...);
  }
  case REAL_RESULT: { double result = item->val_real(); ... new Item_float(...); }
  ...
}
```
**这里 `1+2` 才真正变成 `Item_int(3)`**——但仅当它出现在 `field = 1+2` 的等值谓词里。

**两轮实现传递闭包**：`a=b AND b=c AND c=5`
- 第一轮：`c=5` 传给 `b`，得到 `b=5` 记入 `save_list`（`COND_CMP{and_father, func}`）
- 第二轮：`b=5` 传给 `a`

**注意这是有界迭代（2 轮），不是不动点**——但 `Item_equal` 已吃掉大部分等值，剩下的链一般很短。

`change_cond_ref_to_const`（`:4859`）的守卫条件：比较上下文一致（`cmp_context`）、校对集一致、用 `clone_item()` 克隆而非共享、`thd->change_item_tree` 保证 PS 重执行可回滚。

**为什么 `Item_equal` 之外还需要它**：`eq_def` 严格相等的限制把类型不完全一致的等值排除在多等值之外，这些只能靠它处理。

### 2.5 Impossible WHERE / Always true —— 五个判定点

| # | 时机 | 位置 | 说明 |
|---|---|---|---|
| 1 | prepare | `simplify_const_condition`（`sql_resolver.cc:934`） | 整个条件是常量时求值，用 `Ignore_error_handler`/`Strict_error_handler` 包裹避免误报错误 |
| 2 | optimize | `remove_eq_conds`（`:10440`） | AND 出现 FALSE ⇒ 整体 FALSE；OR 出现 TRUE ⇒ 整体 TRUE |
| 3 | 读完 const 表 | `make_join_query_block:9608-9644` | `const_cond->val_int()` 真正求值 → `zero_result_cause = "Impossible WHERE noticed after reading const tables"` |
| 4 | 多等值恒假 | `eliminate_item_equal:4611` | 返回 `Item_func_false` |
| 5 | range 优化 | `make_join_query_block:9891-9934` | `test_quick_select < 0`；**必须先去掉 ON 再跑一次才能区分 Impossible WHERE 与 Impossible ON** |

`remove_eq_conds` 的三值汇聚：
```cpp
case Item::COND_FALSE:
  if (and_level) { *cond_value = tmp_cond_value; *retcond = nullptr; return false; }
  break;                          // AND 有 FALSE ⇒ 整体 FALSE
case Item::COND_TRUE:
  if (!and_level) { *cond_value = tmp_cond_value; *retcond = nullptr; return false; }
  break;                          // OR 有 TRUE ⇒ 整体 TRUE
```

守卫（`can_evaluate_condition`，`:10408`）：
```cpp
condition->const_for_execution() && !condition->is_expensive() &&
evaluate_during_optimization(condition, thd->lex->current_query_block())
```
三个条件缺一不可。**hypergraph 优化器会设 `OPTION_NO_SUBQUERY_DURING_OPTIMIZATION`**（`:437-438`），此时子查询不能在优化期求值。

短路整个查询（`:480-485`）：`cond_value == COND_FALSE` → `zero_result_cause = "Impossible WHERE"` + `create_access_paths_for_zero_rows()` + `goto setup_subq_exit`（跳过整个 planner）。

---

## 三、谓词下推

### 3.1 make_cond_for_table（`sql_optimizer.cc:9492`）—— AND 可拆 / OR 全或无

```cpp
Item *make_cond_for_table(THD *thd, Item *cond, table_map tables,
                          table_map used_table, bool exclude_expensive_cond) {
  // 第一道门：归属判定
  if (used_table && !(cond->used_tables() & used_table) &&
      !(cond->is_expensive() && used_table == tables))
    return nullptr;

  if (cond->type() == Item::COND_ITEM) {
    if (是 AND) {
      while ((item = li++)) {
        Item *fix = make_cond_for_table(thd, item, tables, used_table, ...);
        if (fix) new_cond->argument_list()->push_back(fix);   // ★ AND：能提多少提多少
      }
      // case 0 → nullptr（恒真）; case 1 → 剥掉多余 AND 层
    } else {  // OR
      while ((item = li++)) {
        Item *fix = make_cond_for_table(thd, item, tables, table_map(0), ...);
        if (!fix) return nullptr;                             // ★ OR：全或无
        new_cond->argument_list()->push_back(fix);
      }
    }
  }
  // 第二道门：可求值判定
  if ((cond->used_tables() & ~tables) ||
      (!used_table && exclude_expensive_cond && cond->is_expensive()))
    return nullptr;
  return cond;
}
```

**参数语义**：`tables` = 当前可用字段值的表集合（连接前缀）；`used_table` = 正在为其提取条件的表（本步新增；可为 0 表示"为所有表提取"）。

**OR 为什么是全或无**：`A OR B` 中若只保留 A，会**错误过滤掉 B 为真的行**（不满足 soundness）。递归时传 `table_map(0)` 而非 `used_table`——OR 内部的子项不需要引用当前表，只需"所有引用的表都可用"。

**两道门组合保证 4 个不变式**：① 每个条件挂到"所有依赖字段都可用的第一张表"；② 每个条件只挂一次；③ 常量条件额外加 `const_table_map | OUTER_REF_TABLE_BIT`；④ 非确定性表达式由最后一张表加 `RAND_TABLE_BIT` 保证每行都算。

### 3.2 外连接下推的语义约束：两种 trig_cond 守卫

`t1 LEFT JOIN t2 ON P` 的语义决定：

| 变换 | 合法? | 原因 |
|---|---|---|
| ON → WHERE | ✘ | 会消除 NULL 补行 |
| ON → 内表 table condition | ✔（需守卫） | 补 NULL 行时必须**关闭**该条件 |
| WHERE → 内表 condition | ✔（需守卫） | 只有已产出匹配行后才可判定 |
| WHERE 拒 NULL ⇒ LJ→IJ | ✔ | `simplify_joins`（04 篇） |

**守卫 1：`IS_NOT_NULL_COMPL`** —— 生成 NULL 补行之前关闭条件。
**守卫 2：`FOUND_MATCH`** —— 只有内嵌外连接找到匹配后才打开。

`attach_join_condition_to_nest`（`sql_optimizer.cc:8665`）是 ON 下推的完整算法：
```cpp
// ① 常量部分 → 挂到第一张内表
Item *cond = make_cond_for_table(thd, join_cond, const_table_map, table_map(0), false);
if (cond) {
  cond = new Item_func_trig_cond(cond, nullptr, this, first_inner,
                                 Item_func_trig_cond::IS_NOT_NULL_COMPL);
  best_ref[first_inner]->and_with_condition(cond);
}
// ② 非常量部分：切分到各内表
for (plan_idx i = first_inner; i <= last_tab; ++i) {
  cond = make_cond_for_table(thd, join_cond, prefix_tables, added_tables, false);
  if (cond == nullptr) continue;
  cond = add_found_match_trig_cond(this, best_ref[i]->first_inner(), cond,
                                   is_sj_mat_cond ? NO_PLAN_IDX : first_inner);
  cond = new Item_func_trig_cond(cond, nullptr, this, first_inner,
                                 Item_func_trig_cond::IS_NOT_NULL_COMPL);
  best_ref[i]->and_with_condition(cond);
}
```

`add_found_match_trig_cond`（`:8638`）沿 `first_upper()` 链向外爬，每层内嵌外连接套一个 `FOUND_MATCH` 守卫：
```cpp
for (; idx != root_idx; idx = join->best_ref[idx]->first_upper()) {
  cond = new Item_func_trig_cond(cond, nullptr, join, idx,
                                 Item_func_trig_cond::FOUND_MATCH);
}
```
`root_idx` 决定爬到哪停：处理 WHERE 时是 `NO_PLAN_IDX`（爬到顶），处理某 nest 的 ON 时是该 nest 的 `first_inner`（不越界）。

**调用时机**（`attach_join_conditions:8776`）：**只有到达某外连接的最后一张内表时**才能处理它的 ON。外层 for 循环是因为一张表可能同时是多个嵌套外连接的最后内表。

### 3.3 semi-join materialization 后的条件处理（取代 5.7 的 make_cond_after_sjm）

8.0 分三步：

**(1) `JOIN::update_equalities_for_sjm()`（`:5136`）**：物化后 `<subquery>` 临时表成为新"表"，把临时表列**加入**等价类（`equality_substitution_transformer`，`item_cmpfunc.cc:7325`），并把后续表的 `keyuse->val` **改指**临时表列。

**(2) `Item_equal::get_subst_item()` 的 SJM 分支（`item_cmpfunc.cc:7240`）**：
```cpp
if (sj_is_materialize_strategy(field_tab->get_sj_strategy())) {
  /* 例：ot1 ot2 <subquery> ot3 SJM(it1 it2 it3)
     等值 ot2.col = <subquery>.col = it1.col = it2.col
     为 it2.col 找替代必须选 it1.col，不能选 ot2.col ——
     因为 it2.col 在物化阶段求值，那时外层表还没读 */
  plan_idx first = field_tab->first_sj_inner(), last = field_tab->last_sj_inner();
  while ((item = it++)) {
    plan_idx idx = item->field->table->reginfo.join_tab->idx();
    if (idx >= first && idx <= last) return item;        // 只在 SJM nest 内选
  }
}
```
**核心思想**：不再"事后重算条件"，而是让**字段替换本身**感知 SJM 边界。

**(3) SJ nest 的 WHERE 下推（`attach_join_conditions:8796`）**：`is_sj_mat_cond = true` 时**不加 `IS_NOT_NULL_COMPL` 守卫**——从物化视角看，SJ nest 的第一张内表就是"顶层"。

### 3.4 条件下推到 derived table（`push_conditions_to_derived_tables`，`sql_resolver.cc:620`）

> 注意是 **prepare 期**，且只在最外层 query block 触发，然后自顶向下递归（使被推下的条件还能继续往更深的派生表推）。

**6 条合法性判定**（`can_push_condition_to_derived`，`sql_derived.cc:1028`）：
```cpp
hint_table_state(..., OPTIMIZER_SWITCH_DERIVED_CONDITION_PUSHDOWN) &&   // 1 开关/hint
!unit->has_any_limit() &&                                              // 2 有 LIMIT
!is_inner_table_of_outer_join() &&                                     // 3 ★ 外连接内表
!(common_table_expr() && (references.size() >= 2 || recursive)) &&      // 4 CTE 共享/递归
(thd->lex->set_var_list.elements == 0) &&                              // 5 用户变量
!unit->m_reject_multiple_rows;                                         // 6 基数校验
```
**第 3 条又是一处 NULL 补行保护**：外连接内表会产生更多 NULL 补行。

**主算法（`make_cond_for_derived`，`sql_derived.cc:1048`）—— 三级切分**：
1. `push_past_window_functions()`：列全在窗口函数 PARTITION BY 中 → 越过窗口函数放到 **HAVING**
2. `push_past_group_by()`：列全在 GROUP BY 中 → 越过聚合放到 **WHERE**
3. `replace_columns_in_cond()`：外层列重映射到派生表内部表达式

`extract_cond_for_table`（`:1160`）用 marker（而非返回值）判定 OR 的全或无，且**同一个函数被复用三次**（`m_checking_purpose` 三态：`CHECK_FOR_DERIVED`/`CHECK_FOR_HAVING`/`CHECK_FOR_WHERE`），每次换不同的列检查器。

**开关 `derived_condition_pushdown` 默认 on**（`sys_vars.cc:211` 在 `OPTIMIZER_SWITCH_DEFAULT` 中）。

**★ 主循环真身**（`sql/sql_resolver.cc:620`，⚠️ 是 `Query_block` 的方法不是 `JOIN` 的）：

```cpp
bool Query_block::push_conditions_to_derived_tables(THD *thd) {
  if (materialized_derived_table_count > 0)
    for (Table_ref *tl = leaf_tables; tl; tl = tl->next_leaf) {
      if (tl->is_view_or_derived() && tl->uses_materialization() &&
          where_cond() && tl->can_push_condition_to_derived(thd)) {
        Item **where = where_cond_ref();   /* ★ 取 WHERE 的引用 → 能原地改写 */
        Condition_pushdown cp(*where, tl, thd, trace);
        if (cp.make_cond_for_derived()) return true;
        *where = cp.get_remainder_cond();  /* ★ 推不下去的留在外层 WHERE */
      }
    }
  /* 自顶向下递归，使嵌套 derived 可以"越推越深" */
  for (Query_expression *unit = first_inner_query_expression(); unit;
       unit = unit->next_query_expression()) {
    for (Query_block *sl = unit->first_query_block(); sl;
         sl = sl->next_query_block()) {
      if (sl->push_conditions_to_derived_tables(thd)) return true;
    }
  }
  return false;
}
```
> ★ 被 merge 掉的 derived（`uses_materialization()==false`）不走这条路。

**★ 条件的三级切分：`Condition_pushdown` 的四个输出槽**（`sql/sql_derived.h:32`）：

```cpp
  Item *m_cond_to_push{nullptr};   /* 输出①：可下推的那一份（UNION 时 clone 成多份） */
  Item *m_having_cond{nullptr};    /* 输出②：能过窗口函数、不能过 GROUP BY → derived 的 HAVING */
  Item *m_where_cond{nullptr};     /* 输出③：连 GROUP BY 都能过 → derived 的 WHERE（最深） */
  Item *m_remainder_cond{nullptr}; /* 输出④：一点都推不动 → 留在**外层** WHERE */

  enum enum_checking_purpose {
    CHECK_FOR_DERIVED,   /* 能不能搬进派生表 */
    CHECK_FOR_HAVING,    /* 能不能过窗口函数 */
    CHECK_FOR_WHERE      /* 能不能过 GROUP BY */
  };                     /* ★ 三阶段状态机，同一个 extract_cond_for_table 复用三次 */
```
```cpp
/* sql_derived.cc:1083 —— ★ 最终是"改写 derived 的 WHERE/HAVING 子句" */
    if (push_past_window_functions()) return true;   /* 第 1 关：越过窗口函数 */
    if (m_having_cond == nullptr) continue;
    if (push_past_group_by()) return true;           /* 第 2 关：越过 GROUP BY */
    ...
    if (replace_columns_in_cond(&m_where_cond, false)) return true;
                                  /* ★ 外层列名 → derived 内部表达式（如 n → i+10） */
    if (m_where_cond &&
        attach_cond_to_derived(qb->where_cond(), m_where_cond, false))
      return true;                /* ★★ 直接改写 derived 的 WHERE —— 不是加 Filter */
```

**★ 6 条禁令的官方注释**（`sql/sql_derived.cc:1005`）：

```cpp
/**
  A condition cannot be pushed down to derived table if any of
  the following holds true:
  1. Hint and/or optimizer switch DERIVED_CONDITION_PUSHDOWN is off.
  2. If it has LIMIT - If the query expression underlying the derived
  table has LIMIT, then the pushed condition would affect the number
  of rows that would be fetched.
  3. It cannot be an inner table of an outer join - that would lead to
  more NULL-complemented rows.
  4. This cannot be a CTE having derived tables being referenced
  multiple times - there is only one temporary table for both references,
  if materialized ("shared materialization"). Also, we cannot push
  conditions down to CTEs that are recursive.
  5. If the derived query block has any user variable assignments -
  would affect the result of evaluating assignments to user variables
  in SELECT list of the derived table.
  6. The derived table stems from a scalar to derived table transformation
  which relies on cardinality check.
*/
```
> ★ 第 2 条最容易被忽略：**有 LIMIT 就不能推**——"先过滤再取 N" 与 "先取 N 再过滤" 语义不同。

### 3.5 条件的最终形态

`QEP_shared::m_condition`（`sql_opt_exec_shared.h:419`）是一个多层包裹的 AND 树：

```
tab->condition() = AND(
  add_not_null_conds() 加的 "x IS NOT NULL"                    :6421
  make_cond_for_table(WHERE, prefix, added) 的产物
    └─ 若在外连接内：Item_func_trig_cond(FOUND_MATCH, ...) 逐层包裹   :9705
  attach_join_condition_to_nest(ON) 的产物                     :8665
    └─ Item_func_trig_cond(IS_NOT_NULL_COMPL, first_inner)     :8722
       └─ 内含 Item_func_trig_cond(FOUND_MATCH, ...)           :8715
  反连接首内表的 Item_func_false()                             :8847
)
```

ICP 之后会被重写（`sql_select.cc:3062`）：`set_condition(idx_remainder_cond)` —— 引擎已负责的部分被剥离。

---

## 四、索引条件下推 ICP

### 4.1 原理与开关

**原理**：普通 index scan 是"引擎按索引定位 → 回表取整行 → server 层用 `m_condition` 过滤"。ICP 把 `m_condition` 中**只引用索引列**的部分交给引擎，引擎在**索引条目上**（不回表）就先求值，不满足则跳过，省掉回表 I/O。

**开关**（已核实默认 **on**）：
```cpp
// sql/sql_const.h:197
constexpr const uint64_t OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN{1ULL << 5};
// sql/sys_vars.cc:196-211 —— 在 OPTIMIZER_SWITCH_DEFAULT 集合里
OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN | ...
```
名字表 `sys_vars.cc:3453`（第 6 项）；hint 名 `NO_ICP(t1 idx)` / `ICP(...)`。

### 4.2 push_index_cond 的 8 项判定（`sql_select.cc:2920`）

```cpp
if (condition() &&                                                          // 0
    tbl->file->index_flags(keyno, 0, true) & HA_DO_INDEX_COND_PUSHDOWN &&   // 1 引擎支持
    hint_key_state(thd, table_ref, keyno, ICP_HINT_ENUM,
                   OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN) &&            // 2 开关/hint
    sql_command != SQLCOM_UPDATE_MULTI &&                                   // 3
    sql_command != SQLCOM_DELETE_MULTI &&                                   // 3
    !has_guarded_conds() && type() != JT_CONST && type() != JT_SYSTEM &&    // 4,5
    !(keyno == tbl->s->primary_key && tbl->file->primary_key_is_clustered())) // 6
{
  Item *idx_cond = make_cond_for_index(condition(), tbl, keyno, other_tbls_ok);
  ...
  idx_remainder_cond = tbl->file->idx_cond_push(keyno, idx_cond);           // ★ 下推
  if (idx_remainder_cond != idx_cond) ref().disable_cache = true;           // ★
  Item *row_cond = make_cond_remainder(condition(), true);
  set_condition(idx_remainder_cond);                                        // 剥离已下推部分
}
```

| # | 判定 | 原理 |
|---|---|---|
| 3 | 多表 UPDATE/DELETE | 同一个 handler 既 select 又 update，引擎在 update 阶段也会应用下推条件 → 找不到/更新错行 |
| 4 | `has_guarded_conds()` | 子查询"Full scan on NULL key"会在执行期动态开关 `cond_guards`，引擎侧无法感知 |
| 5 | JT_CONST/JT_SYSTEM | 只读一次然后复用，若下推条件引用前面表的字段只会在第一次求值 |
| 6 | 聚簇主键 | 主键扫描本来就"索引即数据"，ICP 收益极低 |

**`ref().disable_cache = true`**：eq_ref 有"相同 key 复用上次结果"的缓存；ICP 后同一 key 在不同外层行下结果可能不同（条件引用了其它表），必须禁用。

**`other_tbls_ok`**（`:2953`）：若本表用 ALL/INDEX_SCAN/RANGE/INDEX_MERGE 且使用 **BNL** 缓存，则不允许引用其它表的字段——BNL 时引擎按"本表顺序"扫描，无法知道当前对应哪一行外层行。BKA 是逐 key 访问，可以。

### 4.3 make_cond_for_index 与 uses_index_fields_only

`make_cond_for_index`（`sql_select.cc:2796`）与 `make_cond_for_table` **完全同构的 AND 可拆/OR 全或无骨架**，差别在于判定谓词换成 `uses_index_fields_only`，且额外维护 `Item::marker = MARKER_ICP_COND_USES_INDEX_ONLY` 供 `make_cond_remainder` 反向剔除。

> `make_cond_remainder`（`:2863`）的关键细节：进入 OR 分支后递归传 `exclude_index = false`。因为 OR 只有**所有** disjunct 都可下推时才整体下推（此时 OR 节点自己带 marker 被入口剔除）；否则 OR 整体没下推，其内部任何 disjunct 都不能剔除。

`uses_index_fields_only`（`sql_optimizer.cc:6509`）—— 四条限制：

```cpp
// Restriction b, c
if (item->has_stored_program() || item->has_subquery()) return false;
// Restriction d
if (func_type == Item_func::DD_INTERNAL_FUNC) return false;
// Restriction a：只允许 IS_NOT_NULL_COMPL
if (func_type == Item_func::TRIG_COND_FUNC &&
    down_cast<Item_func_trig_cond *>(item_func)->get_trig_type() !=
        Item_func_trig_cond::IS_NOT_NULL_COMPL)
  return false;
// 字段级判定
case Item::FIELD_ITEM:
  return item_field->field->part_of_key.is_set(keyno) &&        // ★ 在该索引中
         item_field->field->type() != MYSQL_TYPE_GEOMETRY &&
         item_field->field->type() != MYSQL_TYPE_BLOB;
```

| 限制 | 原因 |
|---|---|
| **a. 触发条件** | 只允许 `IS_NOT_NULL_COMPL`——NULL 补行在 server 层迭代器阶段才生成，引擎侧永远看不到，该守卫恒真。而 `FOUND_MATCH` 在嵌套外连接下可能两态求值，引擎无法处理 |
| **b. `has_stored_program()`** | 存储函数内部可能起新语句（DML），从引擎内部回调 server 不是所有 SE 都支持 |
| **c. `has_subquery()`** | 可能嵌套更多表 |
| **d. `DD_INTERNAL_FUNC`** | 会打开数据字典表，InnoDB 里造成**同线程对同页的递归加 latch** |

**核心不变式**：下推条件涉及的本表字段**全部可从索引条目读出**，因此引擎不需回表就能求值。（GEOMETRY/BLOB 排除：前缀索引存的不是完整值。）

### 4.4 handler 接口与 InnoDB 实现

```cpp
// sql/handler.h:5958  契约见 :5933-5956
virtual Item *idx_cond_push(uint keyno, Item *idx_cond) {
  return idx_cond;          // 基类：不接受，全部还给 server
}
```
**协议**：引擎自由决定接受多少；返回它**不**负责的部分；**全接受返回 NULL**；全不接受返回入参本身。

```cpp
// storage/innobase/handler/ha_innodb.cc:23820 —— InnoDB 全盘接受
Item *ha_innobase::idx_cond_push(uint keyno, Item *idx_cond) {
  pushed_idx_cond = idx_cond;
  pushed_idx_cond_keyno = keyno;
  in_range_check_pushed_down = true;       // ★ 副作用
  return nullptr;                          // 全部接受
}
```

**`in_range_check_pushed_down = true` 的副作用容易被忽略**：它让 server 层 `handler::compare_key` 直接返回 0（`handler.cc:7478`）——**范围上界检查的责任也转给引擎**。所以 InnoDB 的 ICP 回调必须自己做 end_range 判断：

```cpp
// ha_innodb.cc:23503
ICP_RESULT innobase_index_cond(ha_innobase *h) {
  if (h->end_range && h->compare_key_icp(h->end_range) > 0)
    return ICP_OUT_OF_RANGE;
  return h->pushed_idx_cond->val_int() ? ICP_MATCH : ICP_NO_MATCH;   // ★ 直接调 server 层 Item 树
}
```

**架构重点**：引擎直接调用 server 层的 Item 树。因此调用前必须已把索引条目的列值填到 `table->record[0]`。准备工作在 `build_template`（`ha_innodb.cc:8358`）计算 `templ->icp_rec_field_no`（每列在**扫描索引**记录里的字段序号，可能是前缀）。

**求值点** `row_search_idx_cond_check`（`row0sel.cc:3782`），调用点 `:4716`/`:5401`/`:5462`：
```cpp
switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
  case ICP_NO_MATCH:    goto next_rec;          // ★ 不匹配，跳过回表
  case ICP_MATCH:       goto requires_clust_rec;
}
```
**这就是 ICP 最大的性能收益**：二级索引扫描时先过滤再回聚簇，不匹配的行完全跳过回表。

**断言保障**（`handler.cc:3248` 等 8 处）：`assert(!pushed_idx_cond || buf == table->record[0])`——ICP 求值依赖 `record[0]`，所有 `ha_index_*` 入口都必须用 `record[0]`。这解释了 `ref_row_iterators.cc:447` 的 `assert(pushed_idx_cond == nullptr)`（某些用备用 buffer 的迭代器不允许 ICP）。

### 4.5 EXPLAIN 输出

**传统 EXPLAIN**（`opt_explain.cc:993`）——判定三项：
```cpp
if (keyno != MAX_KEY && keyno == table->file->pushed_idx_cond_keyno &&
    table->file->pushed_idx_cond) {
  push_extra(ET_USING_INDEX_CONDITION, buff);      // "Using index condition"
}
```
**直接读 handler 状态**——只有 `push_index_cond` 真的把条件交给引擎并被接受时才会显示。JSON 格式 key 是 `index_condition`（`opt_explain_json.cc:64`）。

**FORMAT=TREE/ANALYZE**（`explain_access_path.cc:134-165`）：`", with index condition: " + ItemToString(pushed_idx_cond)`。

### 4.6 ICP 对代价模型的影响：**零**

ICP 的代价**完全没有被纳入代价模型**——优化器在估算 range/ref 扫描的行数时，用的是 `records_in_range`/`records_per_key`，**不会**因为"ICP 能过滤掉一部分"而下调。`pushed_idx_cond` 只在 `make_join_readinfo` 之后（`sql_select.cc:2920`）才设置，而那时 join order 与访问方法都已定完。

这意味着 **ICP 是一个纯粹的执行期优化**：它降低实际 I/O，但不影响计划选择。所以 EXPLAIN 的 `rows` 估算不会因 ICP 而变少。

**★★ "零影响"的 grep 负证据**（★ 全库核查，这是本节最硬的论证）：

| 搜索 | 路径 | 结果 |
|---|---|---|
| `icp` / `idx_cond` | `sql/opt_costmodel.cc` | **0 命中** |
| `icp` / `idx_cond` / `pushed` | `sql/range_optimizer/range_optimizer.cc` | **0 命中** |
| `icp` / `idx_cond` | `sql/**/*cost*.cc` | **0 命中** |
| `icp` | `sql/join_optimizer/` | 仅子串误匹配，**无真实 ICP 引用** |
| `icp` / `idx_cond` | `sql/sql_optimizer.cc` | 仅 9 处：`fix_ICP:` 标签、goto、注释、`assert` —— **全部与代价计算无关** |

**★ 时序证据：ICP 在计划定稿后才 push**（`sql/sql_optimizer.cc:2774`）：

```cpp
  if (changed_key >= 0) {
    // switching to another index
    // Should be no pushed index conditions at this point
    assert(!table->file->pushed_idx_cond);
                                      /* ★★★ 此刻必然为空 —— ICP 还没推 */
  }
```

**★ 一个硬编码的回避规则（不是代价估算）**（`sql/sql_optimizer.cc:2704`）：

```cpp
        /*
          The current implementation of the reverse RefIterator does not
          work well in combination with ICP and can lead to increased
          execution time. Setting changed_key to the current key
          (based on that we change the access order for the key) will
          ensure that a pushed index condition will be cancelled.
        */
        changed_key = tab->ref().key;
```
> ★★ 反向 ref + ICP 会让执行**变慢**——但 MySQL 用的是"硬编码回避"，**不是代价模型算出来的**。这从反面印证了代价模型里没有 ICP 项。

**★ ICP 唯一的"账本"副作用**（`sql/sql_opt_exec_shared.h:117`）：

```cpp
  /*
    true <=> disable the "cache" as doing lookup with the same key value may
    produce different results (because of Index Condition Pushdown)
  */
  bool disable_cache;
```
> ★ 这是**正确性补丁**（同 key 两次查可能结果不同），不是代价项。触发点：`if (idx_remainder_cond != idx_cond) { ref().disable_cache = true; }`（`sql_select.cc:3052`）。

**★ 反向 range scan 里的 ICP 边界约束**（`reverse_index_range_scan.cc:168`，★ 少有人知）：

```cpp
    /*
      If we have pushed an index condition (ICP) and this quick select
      will use ha_index_prev() to read data, we need to let the
      handler know where to end the scan in order to avoid that the
      ICP implementation continues to read past the range boundary.
    */
    if (table()->file->pushed_idx_cond) {
      if (!eqrange_all_keyparts) {
        key_range min_range;
        last_range->make_min_endpoint(&min_range);
```
```cpp
        /*
          HA_ERR_END_OF_FILE is returned both when the table is empty and when
          there are no qualifying records in the range (when using ICP).
          Interpret this return value as "no qualifying rows in the range" to
          avoid loss of records. ...
        */
```
> ★★ ICP 让 `HA_ERR_END_OF_FILE` 的语义变**二义**了（空表 vs range 内无合格行）——这是 ICP 引入的一个副作用。

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| `Item_equal_fields` 是专门容器 | **不存在** → 真实成员 `List<Item_field> fields` |
| `Item_equal` 用 `get_const()` / `const_item` 成员存常量 | **都不存在** → 真名 `Item *m_const_arg`，访问器 `const_arg()` / `set_const_arg()` |
| `Item_equal::compare_type()` | **不存在** → 类型解析走 `resolve_type()` + `cmp_item *eval_item` |
| `COMPARE_ITEMS_TO` | **全库 0 命中，不存在** |
| `Item_equal` 重写了 `const_item()` | **没有** → 继承 `Item::const_item() { return used_tables()==0; }` |
| 等价类是 union-find | **不是** → `List` + `merge()` 里 `fields.concat()` 链表合并 |
| HAVING 也走 `build_equal_items` | **错** → `optimize_cond` 里 `if (join_list)` 才调，HAVING 传 `nullptr` |
| `COND_IMPOSSIBLE` / `COND_TRUE` 常量 | **不存在** → `Item::cond_result{COND_UNDEF,COND_OK,COND_TRUE,COND_FALSE}` + `Item_func_true`/`Item_func_false` + 字符串 `zero_result_cause` |
| `cond_is_false()` / `cond_is_true()` | **不存在** → `can_evaluate_condition()` + `eval_const_cond()` + `fold_condition()` |
| `is_non_deterministic()` 不存在 | **存在** → `used_tables() & RAND_TABLE_BIT` |
| `Item::replace_item()` | **不存在** → `replace_equal_field` / `replace_item_field` / `equal_fields_propagator`，由 `compile()` 驱动 |
| `COND_EQUAL` 在 `sql_lex.h` | **在 `sql/item_cmpfunc.h:2701`** |
| 常量折叠只用 `fold_condition` | 入口是 `remove_eq_conds` → `fold_condition_exec` → `fold_condition`，且**二次折叠** |
| `push_conditions_to_derived_tables` 已改名/迁出 | **仍在 `sql_resolver.cc:620`**，是 `Query_block` 的成员（不是 `JOIN` 的） |
| 下推到 derived = 物化后加 Filter | **错** → 改写 derived 自己的 `WHERE`/`HAVING`；发生在 **prepare 阶段** |
| `pushdown_condition` 是自由函数 | **不存在该符号** → `class Condition_pushdown` + `make_cond_for_derived()` |
| `make_cond_for_table()` 返回三个值 | **只返回 `Item *`（两态）** |
| `enum_trig_type` 只有两个值 | **三个**：`IS_NOT_NULL_COMPL` / `FOUND_MATCH` / `OUTER_FIELD_IS_NOT_NULL` |
| ICP 下推时所有 trig_cond 都可推 | **只认 `IS_NOT_NULL_COMPL`**，另两种一律拒 |
| ICP 省的是 CPU | **省的是回表 I/O + 锁**；比较仍在 server 的 Item 树里做 |
| ICP 会被计入代价 | **零影响** → 代价模型全库 grep 无 ICP 项；ICP 在计划定稿后才 push |
| `NO_ICP` hint 是表级 | 是 **key-level hint**（`NO_ICP(t1 idx_a)`） |
| `derived_condition_pushdown` 默认 off | 默认 **on** |
| ICP 只有 InnoDB 有 | MyISAM 也实现了（`index_cond_func_myisam`）；`HA_DO_INDEX_COND_PUSHDOWN` 是通用 flag |
| `Using index condition` 是优化器打的标记 | 判定读的是 **handler 运行时成员** `pushed_idx_cond_keyno` + `pushed_idx_cond` |

---


## 参考

**论文**
- **Selinger et al.《Access Path Selection》(SIGMOD 1979)** —— 等值传播与选择性估算的源头

**官方文档**
- *MySQL 8.0 Reference Manual → Condition Filtering*
- *MySQL 8.0 Reference Manual → Index Condition Pushdown Optimization*
- *MySQL 8.0 Reference Manual → Constant-Folding Optimization*

**内核月报**
- **2021/07《条件优化与执行分析》** —— `optimize_cond` 四步：`extract_common_cond`（OR 提取公共子条件）/ `build_equal_items`（等值传播）/ `propagate_cond_constants`（常量传播）/ `remove_eq_conds`（冗余条件去除）


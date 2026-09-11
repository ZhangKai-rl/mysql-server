# 05 逻辑优化：等值传播、常量折叠、谓词下推、ICP

> 本篇覆盖 WHERE 条件的四类逻辑优化，全部算法级。

## 目录

- [零、流水线与常见误解](#零流水线与常见误解)
- [一、等值传播（Item_equal 等价类）](#一等值传播item_equal-等价类)
- [二、常量折叠](#二常量折叠)
- [三、谓词下推](#三谓词下推)
- [四、索引条件下推 ICP](#四索引条件下推-icp)

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


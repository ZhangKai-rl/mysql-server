# 18 Hypergraph 优化器高阶模块：CSE、谓词归位、计划最终化、新代价模型与二级引擎

> 基于 MySQL 8.0.39。本篇覆盖 `common_subexpression_elimination`、`finalize_plan`、`join_optimizer/cost_model`、`replace_item`、二级引擎五块，以及它们嵌在 `FindBestQueryPlan → CreateIteratorFromAccessPath` 主链路上的位置。
>
> **边界**：建图（CD-C / TES 膨胀）、DPhyp 五函数、`CostingReceiver` 的 Pareto 锦标赛、`SimplifyQueryGraph` 见 [`09_hypergraph.md`](09_hypergraph.md)；选择率估算见 [`../01_cost_model.md`](../01_cost_model.md)；旧优化器 greedy 搜索见 [`06_join_order.md`](06_join_order.md)。本篇不重写它们。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、CSE：公共子表达式消除](#一cse公共子表达式消除)
  - [二、谓词最终归位：从位图回到 Item 树](#二谓词最终归位从位图回到-item-树)
  - [三、FinalizePlanForQueryBlock：让计划可执行](#三finalizeplanforqueryblock让计划可执行)
  - [四、join_optimizer/cost_model：hypergraph 自己的代价模型](#四join_optimizercost_modelhypergraph-自己的代价模型)
  - [五、replace_item：Item 树的批量替换](#五replace_itemitem-树的批量替换)
  - [六、二级引擎：改代价、拒计划、接管执行](#六二级引擎改代价拒计划接管执行)
  - [七、完整调用栈](#七完整调用栈)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 五个模块在链路上的位置

DPhyp 枚举完之后，产出的那棵 `AccessPath` 树**还不能执行**：谓词还是一串位图下标，临时表还是 `nullptr`，`Filesort` 对象还没构造，SELECT list 里的 `Item` 还指向基表而不是物化后的临时表字段。

| 模块 | 时机 | 一句话 |
|---|---|---|
| **CSE**（`CommonSubexpressionElimination`） | 建图**之前** | 把 `(a AND b) OR (a AND c)` 改写成 `a AND (b OR c)`，让 `a` 成为一条**独立谓词** |
| **谓词归位**（`ApplyPredicatesForBaseTable` / `ApplyDelayedPredicatesAfterJoin` / `ExpandFilterAccessPaths`） | 枚举之中 + 枚举之后 | 把"已应用谓词"的位图翻译成真正的 `Item_cond_and`，包成 `FILTER` 节点 |
| **`FinalizePlanForQueryBlock`** | 枚举之后、迭代器之前 | 建临时表、建 `Filesort`、把 Item 改写指向临时表、调 `push_to_engines()` |
| **`join_optimizer/cost_model`** | 枚举**过程中**被调用 | 一套自己的 `k*OneRowCost` 常量 + 复用旧 `Cost_model_server` 的双轨估算 |
| **二级引擎** | 枚举过程中 + 最终化末尾 | 让外部引擎改代价、拒计划、接管执行 |

### 一个贯穿的例子

```sql
SELECT * FROM t1 WHERE (t1.a = 1 AND t1.b > 3) OR (t1.a = 1 AND t1.b < 0);
```

在旧优化器眼里这是一个**不可分割的谓词**：`Item_cond_or` 是整体，只能整体挂在 `FILTER` 上（或走 `SEL_ARG` 的 DNF 展开进 range 优化器，那是另一条路）。

hypergraph 里，`MakeJoinHypergraph()` 开头会对所有条件跑一遍 CSE（源码注释原文：`Run simple CSE on all conditions`），把它改写成：

```
t1.a = 1 AND (t1.b > 3 OR t1.b < 0)
```

再由 `ExtractConditions` 拆成**两条独立谓词**：

- `t1.a = 1` —— sargable，可以进 `sargable_predicates`，可以变 ref access / range scan
- `(t1.b > 3 OR t1.b < 0)` —— 普通 filter，各自估选择率

这就是 CSE 头注释自己写的动机：

> *The primary motivation is that such split-out items are more versatile; they can be pushed independently, be made into hash join conditions etc. However, an added bonus is that the expressions will simply execute faster.*

注意动机里的**主次**：提速只是 "added bonus"，**"让谓词变得可独立处置"才是主因**。这一点决定了 CSE 为什么被放在**建图之前**而不是代价阶段。

---

## 理论基础

### 这不是编译器里的 CSE

必须先澄清误解：这个名字容易让人以为是编译器那种"把重复计算的表达式抽成临时变量"的优化。**不是**。

| | 编译器 CSE | MySQL hypergraph CSE |
|---|---|---|
| 处理对象 | 任意表达式（算术、函数调用） | **只有布尔的 AND/OR 结构** |
| 手段 | 提取公共子表达式到临时变量，只算一次 | **布尔代数因子提取**：`ab + ac → a(b + c)` |
| 收益 | 省计算 | **让谓词可独立下推 / 可做 join 条件**（提速是附带） |
| 触发 | 数据流分析（可用表达式） | 纯语法结构匹配（`Item::eq(binary_cmp=true)`） |

#### 三个"叫 CSE 但不是一回事"的东西

| 名称 | 出处 | 做什么 | 与 MySQL 的关系 |
|---|---|---|---|
| **编译器 CSE** | Aho, Lam, Sethi, Ullman《Compilers: Principles, Techniques, and Tools》(龙书) 第 10 章 | 数据流分析求**可用表达式**（available expressions），把重复计算的表达式抽成临时变量，只算一次 | **同名但不同物**。MySQL 不做这个，也不求值 |
| **优化器 CSE（Cascades/Volcano）** | Graefe《The Cascades Framework for Query Optimization》(1995)；Volcano (1993) | 通过 **memoization（group / group expression）自动去重**等价的逻辑表达式——重复子表达式天然收敛到同一个 group | MySQL 经典优化器没有；**hypergraph 的枚举有 memoization**，但那是另一处机制，不是 `common_subexpression_elimination.cc` 做的 |
| **布尔因子提取**（MySQL 这个） | 布尔代数分配律；逻辑综合里的 "factoring / factored form"（Brayton 系的多级逻辑综合） | `ab + ac → a(b + c)`，在 DNF 中提出公共合取因子 | ★ **MySQL 实际做的就是这个** |

★ 关键结论：**MySQL 的 CSE 与数据库领域通常说的"优化器 CSE"（Cascades 的 memoization）不是同一件事**。它借用了编译器的名字，做的是布尔代数因子提取。这是读源码时最容易产生误解的地方，也是为什么本篇第一章要先澄清。

它本质上是**分配律的反向应用**：析取范式（DNF）中公共合取因子的提取。所以源码注释特别强调它"不下降进非 AND/OR 的子表达式"：

```
1 + ((a AND b) OR (a AND c))    ← 不动
```

因为 `1 + X` 里 `X` 不是布尔上下文，改写没有意义（也不安全）。

### 为什么 hypergraph 特别需要它

旧优化器把 WHERE 当一棵活树整体处理，谓词不拆散；hypergraph 则要求**每条谓词是独立可处置的原子**（要能单独估选择率、单独决定放进 `sargable` 还是 `filter`、单独作为 join 条件）。

所以 CSE 是为 hypergraph 的谓词模型"预处理原料"——把用户写出来的嵌套布尔结构，尽量拆成扁平的独立原子。这也解释了它为什么放在建图之前。

### 布尔因子的判定：`AlwaysPresent` 的三条规则

判断"某个子项是否对整个表达式成真必需"，这是提取公共因子的前提。三条递归规则：

```
1. expr 与 item 结构相同                     → 必需
2. expr 是 OR，且它的每个分支都必需 item      → 必需（两边都要）
3. expr 是 AND，且任一因子必需 item           → 必需（一边就够）
```

对应源码注释里的例子：

```
(item AND x) OR (item AND y) OR (z AND w AND item)
```

三个分支里 `item` 都必需，所以可以提出来。注意规则 2 与规则 3 的**对偶性**：OR 要求全部分支都有，AND 只要求一个因子有——这正是布尔代数的语义。

### 他库对比

| 库 | 做法 |
|---|---|
| **MySQL 经典优化器** | 无此步骤；DNF 展开只发生在 range 优化器里（`SEL_ARG`），且只针对 sargable 谓词 |
| **MySQL hypergraph** | 建图前做一次纯语法的布尔因子提取 |
| **PostgreSQL** | `canonicalize_qual()` 做 `OR` 的展平与常量折叠，但**不做**这种跨分支因子提取 |
| **SQL Server / Orca** | 有更通用的布尔表达式规范化（含 CNF/DNF 转换与因子提取） |

---

## 核心实现

### 一、CSE：公共子表达式消除

> `sql/join_optimizer/common_subexpression_elimination.cc`，整个文件只有一个对外函数。

#### 1.1 主函数

```cpp
Item *CommonSubexpressionElimination(Item *cond) {
  if (!IsOr(cond)) {
    // Not an OR expression, but we could have something within it.
    return cond;
  }

  Item_cond_or *or_item = down_cast<Item_cond_or *>(cond);
  if (or_item->argument_list()->is_empty()) {
    // An OR with no elements is a false condition.
    return new Item_func_false;
  }

  // Find all items in the first AND of the OR group (or first Item,
  // if it's not an AND conjunction). For each of them, we check
  // if they exist in all the other ANDs as well.
  List<Item> common_items;
  Item *first_group = or_item->argument_list()->head();
  if (IsAnd(first_group)) {
    Item_cond_and *and_group = down_cast<Item_cond_and *>(first_group);
    for (Item &and_arg : *and_group->argument_list()) {
      if (AlwaysPresent(or_item, &and_arg)) {
        common_items.push_back(&and_arg);
      }
    }
  } else {
    if (AlwaysPresent(or_item, first_group)) {
      common_items.push_back(first_group);
    }
  }

  if (common_items.is_empty()) {
    // No common items, so no CSE is possible.
    return cond;
  }

  Item *remainder = OrGroupWithSomeRemoved(or_item, common_items);
  if (remainder != nullptr) {
    common_items.push_back(remainder);
  }
  assert(!common_items.is_empty());
  return CreateConjunction(&common_items);
}
```

**算法逐段解释**：

1. **只对 OR 生效**。非 OR 直接返回（注释说"里面可能还有东西"——但它不递归下降，所以实际是不处理）。
2. **空 OR = FALSE**。`remove_eq_conds()` 把恒假分支删光后可能留下空 `Item_cond_or`，这里规范化成 `Item_func_false`。
3. **候选只从第一个分支取**。这是一个**范围限制**：它只检查第一个 AND 组的成员是否"处处存在"，不去枚举其它分支里的潜在公共因子。代价是会漏掉一些可提取的情形（例如 `(a AND b) OR (c AND b)` 里 `b` 是公共的，但 `b` 在第一个分支里存在所以能被找到——这个例子恰好可行；而 `(x AND b) OR (y AND b)` 同样可行）。注意注释里坦承 `AlwaysPresent()` 有"wasted work"：它对第一个分支自己也检查了一遍。
4. **`AlwaysPresent` 判定**"该子项在整个 OR 中是否必需"。
5. **`OrGroupWithSomeRemoved` 求余项**：把公共项从每个分支里"置 TRUE 移除"，得到剩余部分。若某个分支被掏空（恒真），返回 `nullptr` 表示"整个 OR 恒真"，此时不加余项。
6. **合成**：`CreateConjunction(common_items + remainder)` —— 公共项与余项的合取。

#### 1.2 AlwaysPresent

> ⚠️ **勘误**：CSE **不在** `join_optimizer.cc`，而是独立文件 **`sql/join_optimizer/common_subexpression_elimination.cc`**；且它工作在 **Item 树上、在超图构建之前**（`EarlyNormalizeConditions()` → `CSEConditions()`，`make_join_hypergraph.cc:2265`），**不接触 `Predicate` 与位图**。

**doxygen（`common_subexpression_elimination.h:29`）**——它明说只做一种改写：

```cpp
/**
  Do simple CSE (common subexpression elimination) on “item”, and return
  the answer. The CSE done is exclusively moving common expressions out
  of conjunctions-of-disjunctions, ie. it rewrites

    (a AND b) OR (a AND c)

  into

    a AND (b OR c)

  The primary motivation is that such split-out items are more versatile;
  they can be pushed independently, be made into hash join conditions etc.
  However, an added bonus is that the expressions will simply execute faster.

  This function does not descend into subexpressions that are not AND/OR
  conjunctions, so e.g. an expression like

    1 + ((a AND b) OR (a AND c))

  will be left as-is.
 */
```

**`AlwaysPresent()` 完整源码**（`common_subexpression_elimination.cc:44`）——三条判据：

```cpp
/**
  Check if “item” is necessary to make the expression true.
  This is case if “expr” is either:

   - The same as “item”, or
   - (something necessary) OR (something necessary)
   - (something necessary) AND anything

  A typical case would be of the latter would be

    (item AND x) OR (item AND y) OR (z AND w AND item)
 */
bool AlwaysPresent(Item *expr, const Item *item) {
  if (expr->eq(item, /*binary_cmp=*/true)) {
    return true;                       // ★ 判据①：纯语法结构等价
  }

  // (something necessary) AND (anything), e.g. item AND x AND y.
  if (IsAnd(expr)) {
    for (Item &sub_item : *down_cast<Item_cond_and *>(expr)->argument_list()) {
      if (AlwaysPresent(&sub_item, item)) {
        return true;                   // ★ 判据②：AND 下「任一」子项成立即可
      }
    }
    return false;
  }

  // (something necessary) OR (something necessary),
  // e.g. item AND (item OR x).
  if (IsOr(expr)) {
    for (Item &sub_item : *down_cast<Item_cond_or *>(expr)->argument_list()) {
      if (!AlwaysPresent(&sub_item, item)) {
        return false;                  // ★ 判据③：OR 下必须「所有」子项都成立
      }
    }
    return true;
  }

  // Something else.
  return false;                        // ★ 非 AND/OR（比较、函数…）一律 false，不下钻
}
```

```cpp
// :38
bool IsOr(const Item *item) {
  return item->type() == Item::COND_ITEM &&
         down_cast<const Item_cond *>(item)->functype() ==
             Item_func::COND_OR_FUNC;
}
// sql/item_cmpfunc.h:2812
inline bool IsAnd(const Item *item) {
  return item->type() == Item::COND_ITEM &&
         down_cast<const Item_cond *>(item)->functype() ==
             Item_func::COND_AND_FUNC;
}
```

**主函数**（`:178`）——注意候选集只来自**第一支**，不是两两比较：

```cpp
Item *CommonSubexpressionElimination(Item *cond) {
  if (!IsOr(cond)) return cond;

  Item_cond_or *or_item = down_cast<Item_cond_or *>(cond);
  if (or_item->argument_list()->is_empty()) {
    return new Item_func_false;        // ★ 空 OR ⇒ 恒假
  }

  List<Item> common_items;
  Item *first_group = or_item->argument_list()->head();
  if (IsAnd(first_group)) {
    Item_cond_and *and_group = down_cast<Item_cond_and *>(first_group);
    for (Item &and_arg : *and_group->argument_list()) {   // ★ 只遍历第一支
      if (AlwaysPresent(or_item, &and_arg)) {
        common_items.push_back(&and_arg);
      }
    }
  } else {
    if (AlwaysPresent(or_item, first_group)) common_items.push_back(first_group);
  }

  if (common_items.is_empty()) return cond;

  Item *remainder = OrGroupWithSomeRemoved(or_item, common_items);
  if (remainder != nullptr) {          // ★ nullptr = 恒真 ⇒ 整支丢弃
    common_items.push_back(remainder);
  }
  return CreateConjunction(&common_items);
}
```

> ★ 还有一个 `MatchesAny()`（`:86`）也是 `eq(binary_cmp=true)` 的调用点，用于"待移除列表"的命中判断。

**调用点**（`make_join_hypergraph.cc:2241`）——改写后必须重新拆分：

```cpp
void CSEConditions(THD *thd, Mem_root_array<Item *> *conditions) {
  bool need_resplit = false;
  for (Item *&item : *conditions) {
    Item *new_item = CommonSubexpressionElimination(item);
    if (new_item != item) { need_resplit = true; item = new_item; }
  }
  if (need_resplit) *conditions = ResplitConditions(thd, *conditions);
}
```

```cpp
bool AlwaysPresent(Item *expr, const Item *item) {
  if (expr->eq(item, /*binary_cmp=*/true)) return true;

  if (IsAnd(expr)) {
    for (Item &sub_item : *down_cast<Item_cond_and *>(expr)->argument_list()) {
      if (AlwaysPresent(&sub_item, item)) return true;
    }
    return false;
  }

  if (IsOr(expr)) {
    for (Item &sub_item : *down_cast<Item_cond_or *>(expr)->argument_list()) {
      if (!AlwaysPresent(&sub_item, item)) return false;
    }
    return true;
  }

  return false;
}
```

- `eq(item, binary_cmp=true)` 是**结构等价**判定（不是值等价），且 binary 比较不做字符集转换——两个语法结构相同的表达式才算同一个。
- AND：任一子项必需 → 必需。
- OR：全部子项必需 → 必需。
- 其它类型（比较、函数、常量）：只做结构等价。

#### 1.3 OrGroupWithSomeRemoved：移除后的恒真传播

```cpp
Item *OrGroupWithSomeRemoved(Item_cond_or *or_item,
                             const List<Item> &items_to_remove) {
  List<Item> new_args;
  for (Item &item : *or_item->argument_list()) {
    if (MatchesAny(&item, items_to_remove)) {
      return nullptr;                    // 该分支恒真 ⇒ 整个 OR 恒真
    } else if (IsAnd(&item)) {
      List<Item> and_args;
      ExtractItemsExceptSome(down_cast<Item_cond_and *>(&item),
                             items_to_remove, &and_args);
      if (and_args.is_empty()) {
        return nullptr;                  // 分支被掏空 ⇒ 恒真
      } else {
        new_args.push_back(CreateConjunction(&and_args));
      }
    } else if (IsOr(&item)) {
      Item *new_item = OrGroupWithSomeRemoved(
          down_cast<Item_cond_or *>(&item), items_to_remove);
      if (new_item == nullptr) return nullptr;   // x OR TRUE ⇒ TRUE
      new_args.push_back(new_item);
    } else {
      new_args.push_back(&item);
    }
  }

  assert(!new_args.is_empty());
  if (new_args.size() == 1) {
    return new_args.head();
  } else {
    Item_cond_or *item_or = new Item_cond_or(new_args);
    item_or->update_used_tables();
    item_or->quick_fix_field();
    return item_or;
  }
}
```

三个要点：

- **恒真的短路传播**：任何一个分支变成 TRUE，整个 OR 就是 TRUE，直接 `return nullptr`。这不是错误路径，而是"提取后原条件退化"的正常结果。
- **递归下降**：OR 里的 OR 会被递归处理，所以嵌套结构也能提取。
- **新建 Item 必须补两步**：`update_used_tables()` 重算依赖表位图、`quick_fix_field()` 修正字段引用。**新建 Item 忘了这两步是这类改写代码的典型 bug 来源**。

注释里给了三个例子，非常直观：

```
(a AND b) OR (c AND d), remove (b)     => a OR (c AND d)
(a AND b) OR (c AND d), remove (b,c)   => a OR d
(a AND b) OR (c AND d), remove (a,b)   => nullptr   （恒真）
```

#### 1.4 限制与边界

| 限制 | 说明 |
|---|---|
| **只在 hypergraph 路径调用** | 调用点在 `MakeJoinHypergraph()` 里，经典优化器完全不跑（这解释了为什么旧优化器看不到这个改写） |
| **无代价阈值** | 纯规则：能提就提，不比较"提了是否更便宜" |
| **不下沉到非 AND/OR** | `1 + ((a AND b) OR (a AND c))` 不动 |
| **候选只取第一分支** | 会漏掉部分可提取情形 |
| **要求结构等价** | `a=1` 与 `1=a` 是否等价取决于 `eq()` 的实现，不做代数规范化 |
| **只处理顶层** | 函数返回后不递归处理新产生的结构 |

---

### 二、谓词最终归位：从位图回到 Item 树

#### 2.0 论文背景：谓词移动（predicate move-around）

"谓词该放在哪里"是查询优化的经典问题，学术上有清晰的谱系：

| 论文 | 贡献 |
|---|---|
| **Levy, Mumick, Sagiv《Query Optimization by Predicate Move-Around》(VLDB 1994)** ★ | 提出 **predicate move-around**：先把谓词**沿查询图上提**（pull up），再统一下推（push down）。论文明确指出"move-around precedes pushdown"，因为只有先上提才能发现更多可下推的机会 |
| **Hellerstein & Stonebraker《Predicate Migration: Optimizing Queries with Expensive Predicates》(SIGMOD 1993)** | 针对**昂贵谓词**（UDF）的迁移优化，把"何时求值"作为优化变量 |
| **Galindo-Legaria & Rosenthal《Outerjoin Simplification and Reordering for Query Optimization》(ACM TODS 1997)** | 外连接场景下谓词移动的合法性条件——这决定了"哪些谓词能穿过外连接下推" |
| **Graefe《Query Evaluation Techniques for Large Databases》(ACM CSUR 1993)** | 教科书式的谓词下推分类（sargable / 残留谓词） |

★ MySQL 在这个谱系里的位置：**只有下推，没有上提**。经典优化器的 `optimize_cond` 做等值传播与常量传播，hypergraph 的 `ExtractConditions` 把谓词分配到各个 join 边——都没有 Levy-Mumick-Sagiv 那种"先上提再下推"的完整 move-around。

这也解释了本篇的一个观察：**谓词的位置在枚举期被固定成位图，最后才展开**。hypergraph 的做法是用"位图 + 延迟展开"来近似表达"谓词可以在多个位置应用"，但真正的 move-around（在查询图上自由移动以求最优）并没有实现。

#### 2.1 为什么是位图

hypergraph 枚举时，为了**避免反复复制 Item 树**，只记录"哪些谓词已经被应用"——用位图下标（`filter_predicates` / `delayed_predicates`）。这样每个候选计划的"谓词集合"只是一个整数集合，比较与合并都很便宜。

代价是：枚举结束后必须把位图**翻译回真正的 Item 树**，否则没法执行。

#### 2.2 三个函数

> ⚠️ **勘误**：常见的 `CreateFilterPath()` / `AddFilter()` **都不存在**。真实链条是
> `ConditionFromFilterPredicates()` → `ExpandSingleFilterAccessPath()` → `ExpandFilterAccessPaths()`，全在 **`sql/join_optimizer/access_path.cc`**。

**① 位图 → Item 树的那一跳**（`access_path.cc:1342`）：

```cpp
Item *ConditionFromFilterPredicates(const Mem_root_array<Predicate> &predicates,
                                    OverflowBitset mask,
                                    int num_where_predicates) {
  List<Item> items;
  for (int pred_idx : BitsSetIn(mask)) {         // ★ 遍历位图中每个置位下标
    if (pred_idx >= num_where_predicates) break; // ★ 越过 num_where_predicates 的是 sargable join 谓词，不作 FILTER
    items.push_back(predicates[pred_idx].condition);
  }
  return CreateConjunction(&items);              // ★ 0 项→nullptr，1 项→原样，多项→AND
}
```

```cpp
// sql/sql_executor.cc:736
Item *CreateConjunction(List<Item> *items) {
  if (items->size() == 0) return nullptr;
  if (items->size() == 1) return items->head();
  Item_cond_and *condition = new Item_cond_and(*items);
  condition->quick_fix_field();
  condition->update_used_tables();
  condition->apply_is_true();
  return condition;
}
```

**② 单节点展开**（`access_path.cc:1353`，节选两个关键分支）：

```cpp
void ExpandSingleFilterAccessPath(THD *thd, AccessPath *path, const JOIN *join,
                                  const Mem_root_array<Predicate> &predicates,
                                  unsigned num_where_predicates) {
  // Expand join filters for nested loop joins.
  if (path->type == AccessPath::NESTED_LOOP_JOIN && ...) {
    List<Item> items;
    for (size_t filter_idx :
         BitsSetIn(path->nested_loop_join().equijoin_predicates)) {  // ★ join 条件也存成位图
      items.push_back(expr->equijoin_conditions[filter_idx]);
      ...
    }
    for (Item *condition : expr->join_conditions) items.push_back(condition);

    AccessPath *filter_path = new (thd->mem_root) AccessPath;
    filter_path->type = AccessPath::FILTER;
    filter_path->filter().child = right_path;
    filter_path->filter().condition = CreateConjunction(&items);
    path->nested_loop_join().inner = filter_path;
    path->nested_loop_join().already_expanded_predicates = true;
    // ★ 幂等保护：同一子路径可能被多个 root candidate 共享
  }

  // 哈希 join 走成环边时，把残留的 join 条件「并回」哈希 join 条件
  if (path->type == AccessPath::HASH_JOIN && ...) {
    MoveFilterPredicatesIntoHashJoinCondition(thd, path, predicates,
                                              num_where_predicates);
  }

  // Expand filters _after_ the access path (these are much more common).
  Item *condition = ConditionFromFilterPredicates(
      predicates, path->filter_predicates, num_where_predicates);
  if (condition == nullptr) return;

  AccessPath *new_path = new (thd->mem_root) AccessPath(*path);
  path->type = AccessPath::FILTER;              // ★ 当前节点就地变成 FILTER
  path->filter().condition = condition;
  path->filter().child = new_path;

  // Clear filter_predicates, but keep applied_sargable_join_predicates.
  MutableOverflowBitset applied_sargable_join_predicates =
      path->applied_sargable_join_predicates().Clone(thd->mem_root);
  applied_sargable_join_predicates.ClearBits(0, num_where_predicates);
  path->filter_predicates = std::move(applied_sargable_join_predicates);
}
```

**③ 整树递归**（`access_path.cc:1463`）：

```cpp
void ExpandFilterAccessPaths(THD *thd, AccessPath *path_arg, const JOIN *join,
                             const Mem_root_array<Predicate> &predicates,
                             unsigned num_where_predicates) {
  WalkAccessPaths(path_arg, join, WalkAccessPathPolicy::ENTIRE_QUERY_BLOCK,
                  [thd, &predicates, num_where_predicates](
                      AccessPath *path, const JOIN *sub_join) {
                    ExpandSingleFilterAccessPath(
                        thd, path, sub_join, predicates, num_where_predicates);
                    return false;
                  });
}
```

**`was_join_condition` 的消费点**（`access_path.cc:1286`）：

```cpp
  for (int filter_idx : BitsSetIn(path->filter_predicates)) {
    if (filter_idx >= num_where_predicates) break;
    const Predicate &predicate = predicates[filter_idx];
    if (!predicate.was_join_condition) continue;   // ★ 只搬「本该是 join 条件」的那些
    ...
  }
  path->filter_predicates = OverflowBitset::Xor(
      thd->mem_root, path->filter_predicates, std::move(moved_predicates));
```

#### 2.4 最终归位判据的唯一出处（`join_optimizer.cc:6642`）

```cpp
  for (size_t i = 0; i < graph.num_where_predicates; ++i) {
    // Apply any predicates that don't belong to any
    // specific table, or which are nondeterministic.
    if (!Overlaps(graph.predicates[i].total_eligibility_set,
                  TablesBetween(0, graph.nodes.size())) ||
        Overlaps(graph.predicates[i].total_eligibility_set, RAND_TABLE_BIT)) {
      fd_set |= graph.predicates[i].functional_dependencies;
      has_final_predicates = true;
    }
  }
  ...
        for (size_t i = 0; i < graph.num_where_predicates; ++i) {
          if (!Overlaps(...) || Overlaps(..., RAND_TABLE_BIT)) {
            filter_predicates.SetBit(i);           // ★ 先只在位图里记一笔
            ...
          }
        }
        ExpandFilterAccessPaths(thd, &path, join, graph.predicates,
                                graph.num_where_predicates);
        //                       ★ 紧跟其后才把位图展开成真 FILTER 节点
```

> ★ 两个条件：① `TES` 与任何表都不相交（跨表/无表，无法下推）；② 命中 `RAND_TABLE_BIT`（非确定性，必须最后算）。

```cpp
// sql/join_optimizer/access_path.h:112 —— Predicate 的完整字段
struct Predicate {
  Item *condition;
  hypergraph::NodeMap used_nodes;
  hypergraph::NodeMap total_eligibility_set;   // ★ TES
  double selectivity;
  bool was_join_condition = false;             // ★ 因成环被降级为 WHERE 的 join 条件
  int source_multiple_equality_idx = -1;       // ★ 多路等值去重用的不透明下标
  FunctionalDependencySet functional_dependencies;
  Mem_root_array<int> functional_dependencies_idx;
  Mem_root_array<ContainedSubquery> contained_subqueries;
};
```

| 函数 | 职责 |
|---|---|
| `ApplyPredicatesForBaseTable` | 对单张基表，把"应用于它的谓词"收集起来，包成 `FILTER` 节点挂在表访问之上；同时计算函数依赖（FD）、决定子查询物化的二选一 |
| `ApplyDelayedPredicatesAfterJoin` | 处理**多等值（multiple equality）**：这类谓词只能在 join 之后应用一次，不能重复下推到两侧 |
| `ExpandFilterAccessPaths` | 遍历整棵树，把所有带谓词位图的节点统一展开成 `FILTER` 节点（真正的收尾动作） |

**"多等值只应用一次"** 是一个容易忽略的正确性要点：`a = b = c` 展开成的等值条件，如果既下推到左表又下推到右表，就会重复计算（虽然结果不变，但代价与执行语义会错）。所以它被标为 "delayed"，统一在 join 之后处理。

#### 2.3 为什么必须等到最后

因为谓词应用的**位置**取决于最终选中的 join 顺序与 join 类型：同一个谓词在 A⋈B 与 B⋈A 下应该挂在不同节点上。枚举期固定下来会限制搜索空间，所以只记位图、最后统一展开。

---

### 三、FinalizePlanForQueryBlock：让计划可执行

> `sql/join_optimizer/finalize_plan.cc`。

#### 3.1 只能调用一次

源码注释在 `join_optimizer.h` 里明确写了：某些阶段可以多次调用，但 **`FinalizePlanForQueryBlock()` 只能调用一次，因为最终化会生成（并改写）状态**。

这一点很关键：它是"破坏性操作"，改写 Item 树指向临时表、真正创建临时表对象。所以 `sql_union.cc` 里对它的调用有严格的条件守卫。

#### 3.2 主要工作

**真实源码**（`finalize_plan.cc:656`）——⚠️ 只做 5 件事，**不建 iterator**：

```cpp
bool FinalizePlanForQueryBlock(THD *thd, Query_block *query_block) {
  assert(query_block->join->needs_finalize);   // ★ 「只能调一次」的保证
  query_block->join->needs_finalize = false;   // ★ 立刻翻 false，第二次调用会 assert 失败

  AccessPath *const root_path = query_block->join->root_access_path();
  assert(root_path != nullptr);
  if (root_path->type == AccessPath::EQ_REF) {
    return false;                              // ★ point select 快路径，全部跳过
  }

  // If the query is offloaded to an external executor, we don't need to create
  // the internal temporary tables or filesort objects, or rewrite the Item tree
  // to point into them.
  if (!IteratorsAreNeeded(thd, root_path)) return false;

  /* ① 合并堆叠的 FILTER */
  WalkAccessPaths(root_path, query_block->join,
                  WalkAccessPathPolicy::ENTIRE_QUERY_BLOCK,
                  [](AccessPath *path, JOIN *) {
                    if (path->type == AccessPath::FILTER &&
                        child->type == AccessPath::FILTER &&
                        child->filter().materialize_subqueries ==
                            path->filter().materialize_subqueries) {
                      Item *condition = new Item_cond_and(
                          child->filter().condition, path->filter().condition);
                      condition->quick_fix_field();
                      condition->update_used_tables();
                      condition->apply_is_true();
                      path->filter().condition = condition;
                      path->filter().child = child->filter().child;
                    }
                    return false;
                  }, /*post_order_traversal=*/true);

  /* ② 自底向上第二趟：真正干活 */
  WalkAccessPaths(root_path, query_block->join,
                  WalkAccessPathPolicy::ENTIRE_QUERY_BLOCK,
                  [&](AccessPath *path, JOIN *join) {
                    DelayedCreateTemporaryTable(thd, query_block, path,
                                                after_aggregation,
                                                &last_window_temp_table,
                                                &num_windows_seen);
                    // ★ 真正 create_tmp_table() 的地方

                    UpdateReferencesToMaterializedItems(
                        thd, query_block, path, after_aggregation,
                        &applied_replacements);
                    // ★ 把 SELECT list / IODKU / ORDER BY / HAVING 的 Item
                    //   改指向临时表字段

                    if (path->type == AccessPath::WINDOW) { ... }
                    else if (path->type == AccessPath::AGGREGATE) {
                      thd->change_item_tree(
                          ci.get_item_ptr(),
                          FindReplacementOrReplaceMaterializedItems(
                              thd, ci.get_item(), *earlier_replacement,
                              /*need_exact_match=*/true));   // ★ 调 replace_item
                      ...
                      after_aggregation = true;
                    }
                    if (AddCachesAroundConstantConditionsInPath(path)) { ... }
                    return false;
                  }, /*post_order_traversal=*/true);

  if (query_block->join->push_to_engines()) return true;   // ★③ HG 路径的 push_to_engines 在这里
  thd->lex->set_current_query_block(old_query_block);
  return error;
}
```

> ⚠️ **勘误**：`finalize_plan.h:38` 那个**三参数**声明 `(thd, query_block, root_path)` 是**僵尸声明**——全仓无定义、无调用；`finalize_plan.cc` 根本不 include `finalize_plan.h`（它 include `join_optimizer.h`）。真实签名是**两参数**。

**`DelayedCreateTemporaryTable()`（`:480`）**——临时表为什么必须"延迟"到这一步：

```cpp
static void DelayedCreateTemporaryTable(THD *thd, Query_block *query_block,
                                        AccessPath *path,
                                        bool after_aggregation,
                                        TABLE **last_window_temp_table,
                                        unsigned *num_windows_seen) {
  if (path->type == AccessPath::WINDOW) {
    ++*num_windows_seen;
    window->set_is_last(*num_windows_seen == query_block->join->m_windows.size());
    path->window().temp_table = CreateTemporaryTableFromSelectList(...);
    *last_window_temp_table = path->window().temp_table;   // ★ 传给紧随其后的 MATERIALIZE
  } else if (path->type == AccessPath::MATERIALIZE) {
    if (path->materialize().param->table == nullptr) {
      if (*last_window_temp_table != nullptr) {
        // A materialization that comes directly after a window;
        // it's intended to materialize the output of that window.
        path->materialize().param->table = *last_window_temp_table;
      } else {
        TABLE *table = CreateTemporaryTableFromSelectList(thd, query_block, nullptr, ...);
        path->materialize().param->table = table;
      }
      EstimateMaterializeCost(thd, path);   // ★ 表建好后才估代价（此时才有真实 handler）
    }
  } else if (path->type == AccessPath::STREAM) { ... }
}
```

**`UpdateReferencesToMaterializedItems()`（`:381`）**——"物化参数自身也要改"的落点：

```cpp
  if (temp_table_param != nullptr) {
    // Update source references in this materialization.
    for (const Func_ptr_array *earlier_replacement : *applied_replacements) {
      for (Func_ptr &func : *temp_table_param->items_to_copy) {
        func.set_func(FindReplacementOrReplaceMaterializedItems(
            thd, func.func(), *earlier_replacement, /*need_exact_match=*/true));
      }
      // ★★ 这就是注释里 "t1.x -> <temp2>.x" 被改写成 "<temp1>.x -> <temp2>.x" 的地方
    }
    applied_replacements->push_back(temp_table_param->items_to_copy);

    ReplaceSelectListWithTempTableFields(thd, join, *temp_table_param->items_to_copy);
    if (thd->lex->sql_command == SQLCOM_INSERT_SELECT) {
      ReplaceUpdateValuesWithTempTableFields(...);   // ★ IODKU
    }
  } else if (path->type == AccessPath::SORT) {
    for (const Func_ptr_array *earlier_replacement : *applied_replacements) {
      ReplaceOrderItemsWithTempTableFields(thd, path->sort().order,
                                           *earlier_replacement);   // ★ ORDER BY
    }
    // Set up a Filesort object for this sort.
    path->sort().filesort = new (thd->mem_root)
        Filesort(thd, CollectTables(thd, path), /*keep_buffers=*/false,
                 path->sort().order, path->sort().limit,
                 path->sort().remove_duplicates, path->sort().force_sort_rowids,
                 path->sort().unwrap_rollup);        // ★④ 建 Filesort 对象（不是 iterator）
    join->filesorts_to_cleanup.push_back(path->sort().filesort);
  } else if (path->type == AccessPath::FILTER) {
    const bool need_exact_match = !after_aggregation;   // ★ HAVING 包在 Item_aggregate_ref 里
    path->filter().condition = FindReplacementOrReplaceMaterializedItems(...);
  }
```

> ★ 那段 44 行块注释（`:611-655`）解释了为什么"物化参数的源也要改"：*"Say that we first have a materialization that copies t1.x -> \<temp1\>.x. After that, we have a materialization that copies t1.x -> \<temp2\>.x. For this to work properly, we obviously need to go in and modify the second one so that it instead says \<temp1\>.x -> \<temp2\>.x"*

```
FinalizePlanForQueryBlock()
  ├─ if (!IteratorsAreNeeded(thd, root_path))   ← 二级引擎用外部执行器时，跳过大部分工作
  ├─ 为物化节点计算并回填 MaterializePathParameters
  │     （临时表的列、键、行数；见 materialize_path_parameters.h）
  ├─ 真正创建临时表对象（AccessPath 上的 temp_table 在此之前是 nullptr）
  ├─ 创建 Filesort 对象（排序参数在此之前只是参数，不是对象）
  ├─ 把 SELECT list / ORDER BY / GROUP BY 的 Item 改写指向临时表字段
  │     （靠 replace_item.cc 的批量替换 + Temp_table_param 的更新）
  └─ push_to_engines()（引擎条件下推；经典优化器里这一步在 JOIN::optimize 末尾）
```

`IteratorsAreNeeded()` 的短路是一个值得注意的设计：**如果二级引擎声明要用外部执行器（`USE_EXTERNAL_EXECUTOR`），MySQL 就不建迭代器，也就不需要最终化**——计划直接交给外部执行器。

#### 3.3 与经典优化器的对照

| 动作 | 经典优化器 | hypergraph |
|---|---|---|
| 建临时表 | `make_tmp_tables_info` | `FinalizePlanForQueryBlock` |
| 建 Filesort | 挂在 `QEP_TAB` | `JOIN::filesorts_to_cleanup` |
| Item 改写指向临时表 | `change_to_use_tmp_fields` | `replace_item.cc` |
| 引擎条件下推 | `push_to_engines`（`JOIN::optimize` 末尾） | 同，但在最终化里调 |

---

### 四、join_optimizer/cost_model：hypergraph 自己的代价模型

> `sql/join_optimizer/cost_model.h/cc`。

#### 4.1 为什么有两套

| | 经典优化器 | hypergraph |
|---|---|---|
| 代价来源 | `Cost_model_table` / `Cost_model_server`（可从 `mysql.server_cost` / `mysql.engine_cost` 表配置） | `join_optimizer/cost_model` 的**编译期常量** + 复用 `Cost_model_server` 的部分函数 |

hypergraph 用**编译期 `constexpr` 常量**而不是可配置表：

```cpp
constexpr double kApplyOneFilterCost   = 0.1;
constexpr double kAggregateOneRowCost  = 0.1;
constexpr double kSortOneRowCost       = 0.1;
constexpr double kHashBuildOneRowCost  = 0.1;
constexpr double kHashProbeOneRowCost  = 0.1;
constexpr double kHashReturnOneRowCost = 0.07;    // ★ 唯一的例外
constexpr double kMaterializeOneRowCost = 0.1;
constexpr double kWindowOneRowCost     = 0.1;
```

**除 `kHashReturnOneRowCost = 0.07` 外全是 `0.1`** —— 即"每处理一行 0.1 个代价单位"。这是**刻意的粗粒度**：枚举过程中要为海量候选算代价，代价函数必须极快，不能去查可配置的代价表。

★ 那个 `0.07` 值得单独说：它是"从建好的哈希表里取出一行"的代价，比 build（0.1）和 probe（0.1）都便宜——因为哈希查找已完成，这里只是读一次结果行。在一组几乎全是 0.1 的常量里它是唯一被单独调过的，说明作者确实区分过"probe 一次"与"probe 命中后返回一行"的开销。（旧版本文档曾漏掉这个常量与 `kMaterializeOneRowCost`，并误称"全都是 0.1"。）

★ 这是一个重要的架构信号：**hypergraph 的代价模型是"够用就行"的**，它的精度目标不是"选对索引"，而是"在 DPhyp 的候选空间里做相对排序"。精度敏感的部分（表扫描、ref 访问）仍然复用旧的 `Cost_model_server`。

#### 4.2 估算函数族

**8 个常量（`cost_model.h:47`）——含那段著名的自嘲注释**：

```cpp
// These are extremely arbitrary cost model constants. We should revise them
// based on actual query times (possibly using linear regression?), and then
// put them into the cost model to make them user-tunable.
constexpr double kApplyOneFilterCost = 0.1;
constexpr double kAggregateOneRowCost = 0.1;
constexpr double kSortOneRowCost = 0.1;
constexpr double kHashBuildOneRowCost = 0.1;
constexpr double kHashProbeOneRowCost = 0.1;
constexpr double kHashReturnOneRowCost = 0.07;   // ★ 唯一非 0.1
constexpr double kMaterializeOneRowCost = 0.1;
constexpr double kWindowOneRowCost = 0.1;

// :39
constexpr size_t kMaxItemLengthEstimate = 4096;  // ★ blob 宽度的启发式上限
// :59
constexpr ha_rows kRowEstimateFallback = 1000;
```

**`FindOutputRowsForJoin()`（`cost_model.h:143`，inline 在头文件）**——4 个分支：

```cpp
inline double FindOutputRowsForJoin(double left_rows, double right_rows,
                                    const JoinPredicate *edge) {
  double fanout = right_rows * edge->selectivity;
  if (edge->expr->type == RelationalExpression::LEFT_JOIN) {
    // For outer joins, every outer row produces at least one row (if none
    // are matching, we get a NULL-complemented row).
    // Note that this can cause inconsistent row counts; see bug #33550360
    // and/or JoinHypergraph::has_reordered_left_joins.
    fanout = std::max(fanout, 1.0);                 // ① 外连接：至少 1 行（NULL 补全）
  } else if (edge->expr->type == RelationalExpression::SEMIJOIN) {
    // ... we want the probability that at least one row is matching, which is
    // something else entirely. However, given that we only have selectivity to
    // work with, we don't really have anything better than to estimate it as a
    // normal join and cap the result at selectivity 1.0 ...
    fanout = std::min(fanout, 1.0);                 // ② 半连接：每外表行最多 1 行
  } else if (edge->expr->type == RelationalExpression::ANTIJOIN) {
    // Antijoin are estimated as simply the opposite of semijoin (see above),
    // but wrongly estimating 0 rows (or, of course, a negative amount) could be
    // really bad, so we assume at least 10% coming out as a fudge factor.
    // It's better to estimate too high than too low here.
    fanout = std::max(1.0 - fanout, 0.1);           // ③ 反连接：1-fanout，下限 10%
  }
  return left_rows * fanout;                        // ④ 默认（inner）：直接乘
}
```

**`EstimateFilterCost()` 与 `FilterCost` 四元组**（`cost_model.h:64` / `:105`）：

```cpp
struct FilterCost {
  double cost_if_not_materialized{0.0};
  double init_cost_if_not_materialized{0.0};
  double cost_if_materialized{0.0};
  double cost_to_materialize{0.0};   // ★ 四种情形：不物化/不物化启动/已物化/物化开销
};

// 廉价重载（子查询结果已缓存时跳过遍历）
inline FilterCost EstimateFilterCost(
    THD *thd, double num_rows,
    const Mem_root_array<ContainedSubquery> &contained_subqueries) {
  FilterCost cost;
  cost.cost_if_not_materialized = num_rows * kApplyOneFilterCost;
  cost.cost_if_materialized = num_rows * kApplyOneFilterCost;
  for (const ContainedSubquery &subquery : contained_subqueries) {
    AddCost(thd, subquery, num_rows, &cost);
  }
  return cost;
}
```

**`AddCost()` 的三种子查询策略**（`cost_model.cc:123`）：

```cpp
    case ContainedSubquery::Strategy::kMaterializable: {
      Cost_model_server::enum_tmptable_type tmp_table_type;
      if (subquery.row_width * num_rows < thd->variables.max_heap_table_size) {
        tmp_table_type = Cost_model_server::MEMORY_TMPTABLE;   // ★ 按 max_heap_table_size 选内存/磁盘
      } else {
        tmp_table_type = Cost_model_server::DISK_TMPTABLE;
      }
      cost->cost_if_materialized += thd->cost_model()->tmptable_readwrite_cost(
          tmp_table_type, /*write_rows=*/0, /*read_rows=*/num_rows);
      cost->cost_to_materialize +=
          subquery.path->cost + kMaterializeOneRowCost * subquery.path->num_output_rows();
      cost->cost_if_not_materialized += num_rows * subquery.path->cost;  // ★ 不物化 = 每行重跑
    } break;
    case ContainedSubquery::Strategy::kNonMaterializable: ...
    case ContainedSubquery::Strategy::kIndependentSingleRow:
      cost->init_cost_if_not_materialized += subquery.path->cost;   // ★ 标量子查询只算一次
      break;
```

**`EstimateSortCost()`（`cost_model.cc:92`）**——`n + k·log₂k`：

```cpp
    // Filesort's complexity is O(n + k log k) with a limit, or O(n log n)
    // without. ... So we always calculate it as n + k log k:
    sort_cost = kSortOneRowCost *
                (num_input_rows +
                 num_output_rows * std::max(log2(num_output_rows), 1.0));
  }
  path->cost = path->init_cost = child->cost + sort_cost;   // ★ 排序是全量操作：init_cost == cost
  path->init_once_cost = 0.0;                               // ★ 重扫要重排，无可复用部分
```

**`EstimateAggregateCost()`（`cost_model.cc:818`）**——按**输入**行数计费：

```cpp
  path->cost = child->cost +
               kAggregateOneRowCost * std::max(0.0, child->num_output_rows());
  path->ordering_state = child->ordering_state;   // ★ AGGREGATE 不改变 ordering state
```

**`EstimateHashJoinKeyWidth()` / `EstimateRowWidthForJoin()`（`make_join_hypergraph.cc:2567` / `:2796`）**——⚠️ 不在 `cost_model`：

```cpp
size_t EstimateHashJoinKeyWidth(const RelationalExpression *expr) {
  size_t ret = 0;
  for (Item_eq_base *join_condition : expr->equijoin_conditions) {
    // We heuristically limit our estimate of blobs to 4 kB.
    // Otherwise, the mere presence of a LONGBLOB field would mean
    // we'd estimate essentially infinite row width for a join.
    const Item *left = join_condition->get_arg(0);
    const Item *right = join_condition->get_arg(1);
    ret += min<size_t>(
        max<size_t>(left->max_char_length(), right->max_char_length()),
        kMaxItemLengthEstimate);          // ★ 取左右较大者，再被 4096 截断
  }
  return ret;
}

size_t EstimateRowWidthForJoin(const JoinHypergraph &graph,
                               const RelationalExpression *expr) {
  size_t ret = EstimateHashJoinKeyWidth(expr);
  for (int node_idx : BitsSetIn(expr->nodes_in_subtree)) {
    const TABLE *table = graph.nodes[node_idx].table;
    for (uint i = 0; i < table->s->fields; ++i) {
      if (bitmap_is_set(table->read_set, i)) {    // ★ 只算真正被读的列
        ret += min<size_t>(table->field[i]->max_data_length(),
                           kMaxItemLengthEstimate);
      }
    }
  }
  // Heuristically add 20 bytes for LinkedImmutableString and hash table
  // overhead. ...
  ret += 20;                                       // ★ 固定 +20，纯拍脑袋
  return ret;
}
```

**`rescan_cost()` 与三个造价字段**（`access_path.h:363`）：

```cpp
  /// Expected cost to read all of this access path once; -1.0 for unknown.
  double cost{-1.0};
  /// Expected cost to initialize ... cost to read k out of N rows would be
  /// init_cost + (k/N) * (cost - init_cost) ...
  double init_cost{-1.0};
  /// Of init_cost, how much of the initialization needs only to be done
  /// once per query block. ... A typical example is a materialized table
  /// with rematerialize=false ...
  double init_once_cost{0.0};

  double rescan_cost() const { return cost - init_once_cost; }
  // ★ 重扫代价 = 总代价 − 可复用的一次性初始化代价
```

> ★ 存 `init_once_cost` 而不存 `rescan_cost` 的原因（源码注释）：*"since nearly all paths have zero init_once_cost, storing that instead allows us to skip a lot of repeated `path->init_once_cost = path->init_cost` calls"*。

**hypergraph 与老代价模型的接缝**（`cost_model.cc:59`）——两处刻意的 hack：

```cpp
double EstimateCostForRefAccess(THD *thd, TABLE *table, unsigned key_idx,
                                double num_output_rows) {
  // ... if we expect an index lookup to give e.g. 0.9 rows on average, the cost
  // model will assume we get back 0 -- and even worse, InnoDB's cost model gives
  // a cost of exactly zero for this case ...
  const double hacked_num_output_rows = ceil(num_output_rows);   // ★ 向上取整，避免 0.9→0→代价 0

  // We call find_cost_for_ref(), which is the same cost model used
  // in the old join optimizer, but without the “worst_seek” cap, ...
  return find_cost_for_ref(thd, table, key_idx, hacked_num_output_rows,
                           /*worst_seeks=*/DBL_MAX);             // ★ 去掉 worst_seek 上限
}
```

| 函数 | 用途 |
|---|---|
| `EstimateCostForRefAccess` | ref 访问的代价 |
| `EstimateSortCost` | 排序（含行数与记录宽度） |
| `EstimateMaterializeCost` | 物化 |
| `EstimateAggregateCost` | 聚合 |
| `EstimateStreamCost` | 流式节点 |
| `EstimateLimitOffsetCost` | LIMIT/OFFSET |
| `EstimateWindowCost` | 窗口函数 |
| `EstimateDeleteRowsCost` / `EstimateUpdateRowsCost` | DML |
| `FindOutputRowsForJoin` | join 输出行数（含 semi-join 的缩减） |
| `AddCost` | 子查询代价（含物化 vs 不物化两态） |

`AddCost` 里的两态值得注意：

```cpp
cost.cost_if_not_materialized = num_rows * kApplyOneFilterCost;
cost.cost_if_materialized     = num_rows * kApplyOneFilterCost;
```

物化与不物化在**这个代价模型里被算成相等**，真正的差异体现在别处（子查询自身的代价）。这是"粗粒度"的又一例证。

---

### 五、replace_item：Item 树的批量替换

最终化阶段需要把 SELECT list 里的 `Item`（原本指向基表字段）替换成指向临时表字段的 `Item_field`。难点是：**Item 树是共享的**，同一个 `Item` 可能被多处引用（WHERE、HAVING、ORDER BY、聚合函数内部），粗暴替换会破坏其它引用。

`replace_item.cc` 提供的机制是按"替换规则"遍历并替换，且注释里特别提到"更新 `Temp_table_param` 本身时也要用它"。

相关场景：

- 物化之后，外层对物化表的引用要换指
- CSE 之后（理论上）也可以用同一套机制做替换
- rollup 的 `unwrap_rollup_group`（见 [`../../runtime/06_rollup.md`](../../runtime/06_rollup.md)）也依赖这类替换

---

### 六、二级引擎：改代价、拒计划、接管执行

#### 6.1 三个介入点

| 介入点 | 机制 |
|---|---|
| **改代价** | 引擎通过 `secondary_engine_flags` 声明它关心哪些上下文（见下），并在代价回调里调整代价 |
| **拒计划** | 引擎可以对某些 AccessPath 返回"我不支持"，该候选被丢弃 |
| **接管执行** | 若代价低于 `secondary_engine_cost_threshold`，把查询卸载到二级引擎 |

#### 6.2 二级引擎标志：不遍历树就知道上下文

**能力位只有 4 个**（`sql/handler.h:2460`）——⚠️ **不存在** `sql/join_optimizer/secondary_engine_flags.h`：

```cpp
using SecondaryEngineFlags = uint64_t;
enum class SecondaryEngineFlag : SecondaryEngineFlags {
  SUPPORTS_HASH_JOIN = 0,
  SUPPORTS_NESTED_LOOP_JOIN = 1,

  // If this flag is set, aggregation (GROUP BY and DISTINCT) do not require
  // ordered inputs and create unordered outputs. This is typically the case
  // if they are implemented using hash-based techniques.
  AGGREGATION_IS_UNORDERED = 2,

  /// This flag can be set to signal that a secondary storage engine will not
  /// use MySQL's executor (see JOIN::override_executor_func). In this case, it
  /// doesn't need MySQL's execution data structures, like internal temporary
  /// tables, filesort objects or iterators. If the flag is set,
  /// FinalizePlanForQueryBlock() will not make any changes to the plan, and
  /// CreateIteratorFromAccessPath() will not be called.
  USE_EXTERNAL_EXECUTOR = 3,
};

inline constexpr SecondaryEngineFlags MakeSecondaryEngineFlags() { return 0; }
template <typename... Args>
constexpr SecondaryEngineFlags MakeSecondaryEngineFlags(
    SecondaryEngineFlag flag1, Args... rest) {
  return (uint64_t{1} << static_cast<int>(flag1)) |
         MakeSecondaryEngineFlags(rest...);
}
```

**"不遍历树就知道上下文"的另一半**（`secondary_engine_costing_flags.h:29`）——4 个零注释的代价期标志：

```cpp
using SecondaryEngineCostingFlags = uint64_t;
enum class SecondaryEngineCostingFlag : SecondaryEngineCostingFlags {
  HAS_MULTIPLE_BASE_TABLES,                 // =0 已在枚举 join（≥2 张基表）
  CONTAINS_AGGREGATION_ACCESSPATH,          // =1 计划中含聚合
  CONTAINS_WINDOW_ACCESSPATH,               // =2 计划中含窗口函数
  HANDLING_DISTINCT_ORDERBY_LIMITOFFSET     // =3 正在处理 DISTINCT/ORDER BY/LIMIT
};
```

声明在 `JoinHypergraph` 上（`make_join_hypergraph.h:88`）——源码自己承认这是权宜之计：

```cpp
  /// Flags set when AccessPaths are proposed to secondary engines for costing.
  /// The intention of these flags is to avoid traversing the AccessPath tree to
  /// check for certain criteria.
  /// TODO (tikoldit) Move to JOIN or Secondary_engine_execution_context, so
  /// that JoinHypergraph can be immutable during planning
  SecondaryEngineCostingFlags secondary_engine_costing_flags{};
```

**全部 6 个赋值点**：

| 位置 | 动作 | 含义 |
|---|---|---|
| `join_optimizer.cc:667` | `&= ~HAS_MULTIPLE_BASE_TABLES` | `FoundSingleNode` 里**清位**（单表） |
| `:3020` | `\|= HAS_MULTIPLE_BASE_TABLES` | `FoundSubgraphPair` 里**置位**（已进 join） |
| `:6559` | `= {}` | 图简化后重建 receiver 时**整体复位** |
| `:6727` | `\|= CONTAINS_AGGREGATION_ACCESSPATH` | 进 GROUP BY 前 |
| `:6851` | `\|= CONTAINS_WINDOW_ACCESSPATH` | 有窗口函数 |
| `:6867` | `\|= HANDLING_DISTINCT_ORDERBY_LIMITOFFSET` | 进 DISTINCT/ORDER BY/LIMIT 阶段 |

**hook 的函数类型与契约**（`handler.h:2380`）——注释里明列可改的 7 个字段与必须满足的不变量：

```cpp
using secondary_engine_modify_access_path_cost_t = bool (*)(
    THD *thd, const JoinHypergraph &hypergraph, AccessPath *access_path);
```

```
可改：init_once_cost / init_cost / cost / cost_before_filter
      num_output_rows / num_output_rows_before_filter / secondary_engine_data
不变式：init_cost <= cost_before_filter <= cost
        num_output_rows <= num_output_rows_before_filter
返回 true = 拒绝该 path（可先 my_error()）
```

**介入点①改代价**（`join_optimizer.cc:4550`，`ProposeAccessPath()` 开头）：

```cpp
  if (m_secondary_engine_cost_hook != nullptr) {
    // If an error was raised by a previous invocation of the hook, reject all
    // paths.
    if (m_thd->is_error()) return nullptr;

    if (m_secondary_engine_cost_hook(m_thd, *m_graph, path)) {
      return nullptr;                     // ★ 介入点②的原子操作：这条 path 消失
    }
    assert(!m_thd->is_error());
    assert(path->init_cost <= path->cost);
    if (!IsEmpty(path->filter_predicates)) {
      assert(path->num_output_rows() <= path->num_output_rows_before_filter);
      assert(path->cost_before_filter <= path->cost);
    }
  }
```

**"All plans were rejected" 的两处报出点**：

```cpp
// :6593 —— join 枚举后拿不到 root candidate
  if (secondary_engine_cost_hook != nullptr &&
      (!receiver.HasSeen(TablesBetween(0, graph.nodes.size())) ||
       receiver.root_candidates().empty())) {
    my_error(ER_SECONDARY_ENGINE, MYF(0),
             "All plans were rejected by the secondary storage engine");
    return nullptr;
  }

// :6943 —— 后处理阶段（sort/limit/grouping）被拒空
  if (root_candidates.empty()) {
    assert(secondary_engine_cost_hook != nullptr);
    my_error(ER_SECONDARY_ENGINE, MYF(0),
             "All plans were rejected by the secondary storage engine");
    return nullptr;
  }
```

> ★ `HasSecondaryEngineCostHook()`（`:257`）是把"候选为空"从**内部 bug** 转为**合法状态**的判据——`ApplyDistinctAndOrder()` 等处都有 `assert(receiver.HasSecondaryEngineCostHook())`（`:5505`、`:5656`）。

⚠️ **勘误**：`EngineFlags()` / `SecondaryEngineCostHook()` 都收 **`const THD *`**；前者**不做任何翻译**，直接返回 `hton->secondary_engine_flags`，无引擎时返回宽松默认 `{HASH_JOIN, NESTED_LOOP_JOIN}`。另有一条更早的拒绝路径 `CheckSupportedQuery()`（`:4846`）：若二级引擎两种 join 都不支持，直接报 `ER_HYPERGRAPH_NOT_SUPPORTED_YET`。

⚠️ **这里其实是两组不同的枚举**，源码分属两个文件，用途也不同——混在一张表里容易误读。

**`SecondaryEngineCostingFlag`**（`secondary_engine_costing_flags.h`）——随**代价回调**传递的"树结构特征"：

| 标志 | 含义 |
|---|---|
| `HAS_MULTIPLE_BASE_TABLES` | 当前计划涉及多张基表 |
| `CONTAINS_AGGREGATION_ACCESSPATH` | 含聚合节点 |
| `CONTAINS_WINDOW_ACCESSPATH` | 含窗口节点 |
| `HANDLING_DISTINCT_ORDERBY_LIMITOFFSET` | 正在处理 DISTINCT / ORDER BY / LIMIT-OFFSET |

**`SecondaryEngineFlag`**（`sql/handler.h`）——**引擎能力声明**，与代价回调无关：

| 标志 | 值 | 含义 |
|---|---|---|
| `SUPPORTS_HASH_JOIN` | 0 | 引擎支持 hash join |
| `SUPPORTS_NESTED_LOOP_JOIN` | — | 引擎支持嵌套循环 |
| `USE_EXTERNAL_EXECUTOR` | 3 | 引擎自带外部执行器（决定还要不要建迭代器） |

前者是**性能设计**：代价回调在枚举热路径上被调用，遍历整棵树太贵，于是把"树的结构特征"压缩成一个 bitmask 随代价一起传递。后者是**能力查询**，引擎启动时声明一次。

#### 6.3 IteratorsAreNeeded

```cpp
bool IteratorsAreNeeded(const THD *thd, AccessPath *root_path) {
  const handlerton *secondary_engine = SecondaryEngineHandlerton(thd);
  if (secondary_engine == nullptr) return true;         // 无二级引擎 → 需要
  if (!ReferencesSecondaryEngineBaseTables(root_path)) return true;  // 查询被优化掉 → 需要
  return !IsBitSet(static_cast<int>(SecondaryEngineFlag::USE_EXTERNAL_EXECUTOR),
                   secondary_engine->secondary_engine_flags);
}
```

三条规则的理由都写在注释里，第二条尤其微妙：**即使查询被完全优化掉，仍然建迭代器**——因为二级引擎可能决定不卸载这个查询。

#### 6.4 社区边界（★ 重要）

**社区版 MySQL 只有框架与接口，没有真正卸载查询的引擎。** 真正实现卸载的是 **Oracle HeatWave**（闭源服务）。具体：

- `secondary_engine_cost_threshold`（默认极大值）在社区版默认不触发卸载
- `USE SECONDARY ENGINE` 语法与 `ALTER TABLE ... SECONDARY_ENGINE` 都在，但没有可用的二级引擎就没有实际效果
- 文档里提到的 RAPID 是 HeatWave 的引擎名

不要把这些当成社区版能力，也不要把"向量化执行"当成 MySQL 的能力——那是 HeatWave 的。

---

### 七、完整调用栈

```
Query_expression::optimize()
 └─ Query_block::optimize()
     └─ JOIN::optimize()
         ├─ [hypergraph 分支，sql/sql_optimizer.cc]
         │   └─ FindBestQueryPlan(thd, query_block)      make_join_hypergraph.cc
         │        ├─ CommonSubexpressionElimination()    ★ 建图前，对所有条件跑一遍
         │        │    （源码注释：Run simple CSE on all conditions）
         │        ├─ MakeJoinHypergraph()                ← 09 篇：CD-C / TES
         │        │    └─ ExtractConditions / ExtractFunctionalDependencies
         │        ├─ SimplifyQueryGraph()                ← 09 篇：图简化重跑
         │        ├─ EnumerateAllConnectedPartitions()   ← 09 篇：DPhyp
         │        │    └─ CostingReceiver::FoundSingleNode / FoundJoin
         │        │         ├─ ApplyPredicatesForBaseTable()      ★ 谓词归位（每步）
         │        │         ├─ ApplyDelayedPredicatesAfterJoin()  ★ 多等值
         │        │         └─ cost_model::Estimate*(...)         ★ 新代价模型
         │        ├─ ExpandFilterAccessPaths()           ★ 谓词位图 → FILTER 节点
         │        └─ FinalizePlanForQueryBlock()         ★ 本篇第三章
         │             ├─ IteratorsAreNeeded()?          ★ 二级引擎短路
         │             ├─ 计算 MaterializePathParameters
         │             ├─ 创建临时表 / Filesort
         │             ├─ replace_item / 更新 Temp_table_param
         │             └─ push_to_engines()
         └─ create_access_paths()（经典优化器分支）

CreateIteratorFromAccessPath()                            ← 08/09 篇
```

---

## ★ 本机制里的工程实现技法

### 1. 纯语法的结构等价（`eq(binary_cmp=true)`）

CSE 用 `Item::eq()` 做**结构等价**而不是值等价。这意味着：

- 不做代数规范化（`a+0` 与 `a` 不等价）
- 不做常量折叠（`1+2` 与 `3` 不等价——注意 [`logical/05_logical_predicate.md`](../logical/05_logical_predicate.md) 里指出 8.0 无通用算术折叠器）
- `binary_cmp=true` 表示不做字符集转换，两个 collation 不同的同构表达式不算同一个

这是"保守正确"的选择：宁可漏掉优化，也不错改语义。

### 2. `nullptr` 作为"恒真"的信号值

`OrGroupWithSomeRemoved` 用返回 `nullptr` 表示"移除后表达式恒真"。这是一个**信号值**（sentinel）用法：调用方必须检查，且语义要靠注释才能读懂（返回值类型 `Item*` 本身不含"可能是空"的信息）。

### 3. 新建 Item 后的两步必修课

```cpp
Item_cond_or *item_or = new Item_cond_or(new_args);
item_or->update_used_tables();
item_or->quick_fix_field();
```

`update_used_tables()` 重算 `used_tables()` 位图（优化器到处依赖它做下推判定），`quick_fix_field()` 修正字段缓存。**忘了第一步会导致谓词下推错误**——这是 Item 树改写类代码的通用陷阱。

### 4. 用 bitmask 代替树遍历（SecondaryEngineCostingFlag）

把"树的结构特征"压缩成一个 bitmask，让代价回调不必遍历 AccessPath。这是热路径上的典型优化：**用一次性计算的元数据，替代每次重复遍历**。

> ⚠️ 这里是 **`SecondaryEngineCostingFlag`**（`join_optimizer/secondary_engine_costing_flags.h`，随代价回调传递的树结构特征），**不是** `SecondaryEngineFlag`（`sql/handler.h`，引擎能力声明）。两者名字极像但归属不同，见 6.2。

### 5. `constexpr` 代价常量 vs 可配置代价表

hypergraph 用编译期常量（全是 0.1），经典优化器用可配置的 `mysql.server_cost` 表。这不是遗漏，而是**枚举热路径的性能取舍**：DPhyp 要评估的候选数量远大于 greedy 搜索，代价函数必须极快且可内联。

### 6. 破坏性的最终化只能调一次

`FinalizePlanForQueryBlock` 会真正创建对象并改写 Item 树，因此不可重复调用。这与经典优化器里 `JOIN::cleanup()` 可多次调用形成对照——**"幂等的清理"与"一次性的最终化"是两种不同契约**，源码注释专门区分了它们。

---

## 可观测性

### 怎么确认 CSE 生效了

CSE 本身**没有 trace 节点**（它跑在 trace 建立之前，且是纯语法改写）。验证方法：

```sql
SET optimizer_switch='hypergraph_optimizer=on';
EXPLAIN FORMAT=TREE
SELECT * FROM t1 WHERE (t1.a=1 AND t1.b>3) OR (t1.a=1 AND t1.b<0);
```

若 CSE 生效，`t1.a = 1` 会变成一条**独立可下推**的条件（表现为它出现在基表访问上，而不是整体的 FILTER 里），而 `(b>3 OR b<0)` 单独成一条 filter。

对照旧优化器（`hypergraph_optimizer=off`）执行同一条，整个 OR 会作为一个整体 filter 出现。

### 各模块的观测入口

| 模块 | 观测手段 |
|---|---|
| CSE | `EXPLAIN FORMAT=TREE` 的谓词形态（无 trace 节点） |
| 谓词归位 | `EXPLAIN FORMAT=TREE` 里 `Filter: ...` 的位置 |
| 最终化 | `EXPLAIN FORMAT=TREE` 的 `Materialize` / `Sort:` 节点参数 |
| 新代价模型 | `EXPLAIN FORMAT=JSON` 的 `cost_info`（注意量纲与经典优化器不同） |
| 二级引擎 | `secondary_engine_cost_threshold` 变量、`EXPLAIN` 里是否出现二级引擎标记 |

### 相关变量

| 变量 | 作用 |
|---|---|
| `hypergraph_optimizer`（switch） | 开关 hypergraph；**源码注释明说"故意不写进文档"** |
| `secondary_engine_cost_threshold` | 低于此代价才卸载到二级引擎 |
| 无 | CSE **没有**开关（随 hypergraph 一起启用，无法单独关闭） |

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 让 CSE 处理更多形态（如 OR 里的 OR 公共因子） | `CommonSubexpressionElimination` 的候选收集部分（目前只取第一分支） |
| 让 CSE 下沉到非 AND/OR | 需在主函数加递归下降，但要注意布尔上下文判定 |
| 新增一种代价估算 | `cost_model.h/cc` 加 `Estimate*()`，并在对应 AccessPath 类型创建处调用 |
| 新增一个 `SecondaryEngineFlag`（引擎能力）或 `SecondaryEngineCostingFlag`（树特征） | 前者在 `sql/handler.h`、后者在 `join_optimizer/secondary_engine_costing_flags.h`——**别搞混**，两者的枚举 + 设置它的位置 |
| 新增一种最终化动作 | `FinalizePlanForQueryBlock`（注意它只能调一次，动作要幂等或放在正确顺序） |

### 已知缺陷

- **CSE 只在 hypergraph 路径生效**，经典优化器完全没有这个改写。这意味着同一条 SQL 在两条路径下语义等价但性能可能差很多。
- **CSE 无 trace、无开关**，出问题只能靠 `EXPLAIN FORMAT=TREE` 反推。
- **hypergraph 代价常量几乎全是 0.1**（唯一例外 `kHashReturnOneRowCost = 0.07`），精度目标是"候选间相对排序"而非绝对准确；这导致 hypergraph 与经典优化器选出的计划难以直接比较代价。
- `common_subexpression_elimination.cc` 抽成独立文件后修过 crash bug（见文件头的版权年份与相关 Bug 记录），说明这类树改写容易出错。

### 社区边界澄清（★）

- **没有可用的二级引擎**：社区版只有框架 + 接口 + 变量，真正的卸载引擎是 Oracle HeatWave（闭源）
- **没有向量化执行**：HeatWave 有，社区版没有
- **CSE 不是编译器 CSE**：只做布尔因子提取，不做通用公共子表达式提取，也不做临时变量提升
- **hypergraph 是实验特性**：默认 off，源码注释明说"故意不写进文档"

---

## 参考

**论文 / 理论**

- **Moerkotte & Neumann《Dynamic Programming Strikes Back》(SIGMOD 2008)** —— DPhyp，本篇各模块服务的枚举框架（见 [`09_hypergraph.md`](09_hypergraph.md)）
- **Aho, Lam, Sethi, Ullman《Compilers: Principles, Techniques, and Tools》第 10 章** —— 编译器 CSE（可用表达式分析）。★ 与 MySQL 的 CSE 同名不同物，本篇第一章专门澄清
- **Graefe《The Cascades Framework for Query Optimization》(1995)** / **《The Volcano Optimizer Generator》(1993)** —— 优化器意义上的 CSE 是通过 **memoization（group / group expression）** 自动去重等价表达式。**MySQL 的 `common_subexpression_elimination.cc` 不做这个**
- **布尔代数分配律；逻辑综合中的 factoring / factored form**（Brayton 系的多级逻辑综合）—— MySQL CSE 的实际数学基础：`ab + ac → a(b + c)`
- **Levy, Mumick, Sagiv《Query Optimization by Predicate Move-Around》(VLDB 1994)** ★ —— 谓词上提 + 下推的完整框架；MySQL 只有下推没有上提（见本篇第二章 2.0）
- **Hellerstein & Stonebraker《Predicate Migration: Optimizing Queries with Expensive Predicates》(SIGMOD 1993)** —— 昂贵谓词的迁移
- **Galindo-Legaria & Rosenthal《Outerjoin Simplification and Reordering for Query Optimization》(ACM TODS 1997)** —— 外连接下谓词移动的合法性
- **Graefe《Query Evaluation Techniques for Large Databases》(ACM CSUR 1993)** —— 谓词下推与物化的经典分类

**WorkLog**

- hypergraph 优化器相关 WL（8.0.22 引入，后续版本持续扩展 `secondary_engine_costing_flags.h` 与 CSE）

**官方文档**

- MySQL 8.0 Reference Manual, "Secondary Engines"（框架说明；但真正的引擎是 HeatWave 产品文档）
- MySQL 8.0 Reference Manual, "Optimizer Hints"（`SET_VAR` 可用于 `secondary_engine_cost_threshold`）

**相关文档**

- [`09_hypergraph.md`](09_hypergraph.md) —— 建图、DPhyp 枚举、CostingReceiver
- [`../01_cost_model.md`](../01_cost_model.md) —— 经典代价模型与 `server_cost` / `engine_cost` 可配置常量（与本篇第四章的 `constexpr` 形成对照）
- [`../../08_access_path.md`](../../08_access_path.md) —— AccessPath 结构与 `ExpandFilterAccessPaths`
- [`../../06_resolver_prepare.md`](../../06_resolver_prepare.md) —— 经典优化器侧的谓词改写（与 CSE 对比）
- [`../../runtime/06_rollup.md`](../../runtime/06_rollup.md) —— rollup 的 Item 替换（与 `replace_item` 同源）

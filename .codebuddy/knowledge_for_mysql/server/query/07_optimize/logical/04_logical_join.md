# 04 逻辑优化：连接简化（外连接转内连接 + derived/view/CTE merge）

> 本篇覆盖 `simplify_joins`（外连接转内连接、ON 上提、嵌套 join 扁平化）与 `merge_derived`（派生表/视图/CTE 合并），全部算法级。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [零、阶段定位与调用链](#零阶段定位与调用链)
- [一、join nest 数据结构与不变式](#一join-nest-数据结构与不变式)
- [二、两个前置归一化](#二两个前置归一化)
- [三、simplify_joins Pass 1：外连接转内连接](#三simplify_joins-pass-1外连接转内连接)
- [四、simplify_joins Pass 2：嵌套 join 扁平化](#四simplify_joins-pass-2嵌套-join-扁平化)
- [五、derived / view / CTE merge](#五derived--view--cte-merge)
- [六、LATERAL：让派生表看见同层兄弟表](#六lateral让派生表看见同层兄弟表)
- [七、semi-join nest 收缩](#七semi-join-nest-收缩pull_out_semijoin_tables)
- [八、已知限制与"本版本不具备的能力"](#八已知限制与本版本不具备的能力)

---

## 设计思想与理论基础

> 后面七节回答"怎么做"；本章回答"**为什么不得不这么做**"。核心两件事：**外连接为什么难优化**、**两个 Pass 为什么必须是这个顺序**。

### 1. 外连接为什么比内连接难优化

**内连接的自由度**：⋈ 可交换、可结合。MySQL 把这份自由度**编码进了数据结构**——hypergraph 的 `FlattenInnerJoins()` 把无连接条件的 INNER JOIN 折成 n 元 `MULTI_INNER_JOIN`，注释说动机是"more flexible pushdown ... **no matter how the join tree was written by the user**"；`MakeJoinGraphFromRelationalExpression()` 的注释一句话概括：**"inner joins are more freely reorderable than outer joins"**。

**外连接：结合律只有一个方向成立**。`OperatorsAreAssociative()` 的头注释：

```cpp
// Returns true if (t1 <a> t2) <b> t3 === t1 <a> (t2 <b> t3).
//
// Note that this is not symmetric; e.g.
//
//   (t1 JOIN t2) LEFT JOIN t3 === t1 JOIN (t2 LEFT JOIN t3)
//
// but
//
//   (t1 LEFT JOIN t2) JOIN t3 != t1 LEFT JOIN (t2 JOIN t3)
```

即：把 ⟕ **往外提**（成为根）合法；把 ⟕ **往里塞**（塞进右子树）非法。

**具体反例**（用行说明，比符号直观）。设 `A = {a:1}`、`B = ∅`、`C = {x:2}`，条件 `ON c.x = 1`：

| 形式 | 计算 | 结果 |
|---|---|---|
| `(A LEFT JOIN B ON TRUE) JOIN C ON c.x=1` | `A⟕B = {(1,NULL)}`，再 `⋈ σ(C) = ⋈ ∅` | **0 行** |
| `A LEFT JOIN (B JOIN C ON c.x=1) ON TRUE` | `B⋈σ(C) = ∅`，`A⟕∅ = {(1,NULL,NULL)}` | **1 行** |

连行数都不一样——**差别完全来自 NULL 补行**。

**注意 `(LEFT_JOIN, LEFT_JOIN)` 这对不是无条件成立**：

```cpp
if ((a.type == LEFT_JOIN || a.type == FULL_OUTER_JOIN) && b.type == LEFT_JOIN) {
  // True if and only if the second join predicate rejects NULLs on all tables in e2.
  return IsNullRejecting(b, b.left->tables_in_subtree);
}
```

外连接之间要"可结合"，前提是**第二个谓词拒 NULL**——这正是"外连接转内连接"在代数层面的等价表述。⇒ **null-rejecting 不是拍脑袋的优化技巧，它就是让外连接重新获得代数自由度的充要条件。**

**nest 就是"重排序约束的载体"**。老优化器 `check_interleaving_with_nj()` 的头注释给出两条限制：

```
LIMITATIONS ON JOIN ORDER
  1. "Outer tables first" - any "outer" table must be before any
     corresponding "inner" table.
  2. "No interleaving" - tables inside a nested join must form a
     continuous sequence in join order
```

并给出反例：`t0 join t1 left join (t2 join t3) on cond1` 中，join order `t1 t2 t0 t3` 非法——t0 的 WHERE 谓词会在 nest 算完前施加，**可能错误地丢掉本该被 NULL 补出来的行**。

⇒ **这就是"嵌套 join 扁平化为什么必要"的答案**：nest 不是装饰，它是一条"这些表必须连成一段、且外表必须先走"的约束。nest 越少越浅，可枚举的 join order 越多。

### 2. NULL 补行：复杂度的根源，与 null-rejecting 的定理

**定义**：

```
A ⟕_p B  =  (A ⋈_p B)  ∪  N(B)
N(B)     =  { a ∘ (NULL, …, NULL) | a ∈ A 且 ¬∃b∈B: p(a,b) }
```

`N(B)`（NULL-complemented rows）两个关键特征：① **整行全 NULL**——B 的每一列都是 NULL；② 与内连接部分**不相交**——它代表"没匹配上"。

外连接全部的复杂度都来自 `N(B)`：破坏结合律、让谓词下推变非法、让去重/聚合/semi-join 都要额外处理"这一行是不是补出来的"、要求执行器维护 match flag。

**定义（null-rejecting）**：谓词 `P` 对表集 `T` 拒 NULL ⟺ 当 `T` 中所有表的所有列均为 NULL 时，`P` 必为 FALSE 或 UNKNOWN。

hypergraph 侧的注释是这个定义的逐字表述：

```cpp
// Returns whether the join condition for "expr" is null-rejecting (also known
// as strong or strict) on the given relations; that is, if it is guaranteed to
// return FALSE or NULL if _all_ tables in "tables" consist only of NULL values.
// (This means that adding tables in "tables" which are not part of any of the
// predicates is legal, and has no effect on the result.)
```

"also known as **strong or strict**" 说明这套术语是学术界通用的，MySQL 只是沿用。

**定理（外连接退化定理）**：设后置过滤器 `W`（WHERE 语境，`UNKNOWN ≡ FALSE`）。若 `W` 对 `T ⊆ B` 拒 NULL，则 `σ_W(A ⟕_p B) = σ_W(A ⋈_p B)`。

**证明**：

```
σ_W(A ⟕_p B) = σ_W((A ⋈_p B) ∪ N(B))          [定义]
             = σ_W(A ⋈_p B) ∪ σ_W(N(B))       [选择对并可分配]
```

而 `N(B)` 中任一元组在 `T` 上全 NULL（特征①），由定义 `W` 对其必为 FALSE 或 UNKNOWN；WHERE 语境下 `UNKNOWN ≡ FALSE`，故 `σ_W(N(B)) = ∅` ⟹ 两者相等。∎

**两条推论，正好对应代码里的两个动作**：

1. `table->outer_join = false`——⟕ 退化为 ⋈
2. `*cond = and_conds(*cond, join_cond)` + `set_join_cond(nullptr)`——ON 从"外连接的匹配条件"变成普通后置过滤条件，搬到 WHERE

第 2 步**不是可选的装饰**：补行已消失，ON 再留在 nest 上就没有语义作用了；而且它是**让 nest 变成"无 ON 的纯括号"从而能被 Pass 2 溶解的唯一办法**。

**为什么"只要 nest 里有一张表被拒 NULL"就够了**：因为 `N(B)` 是整行全 NULL，`W` 只要拒绝 B 中**任意一张**表，所有补行就一起被过滤。所以判据是**有交集即可**：

| 位置 | 判据 |
|---|---|
| 经典优化器 | `if (!table->outer_join \|\| (used_tables & not_null_tables))` |
| hypergraph | `Overlaps(tables, cond->not_null_tables())`（是 `Overlaps` 不是 `IsSubset`） |

⇒ **一个重要精度结论**：MySQL 把"拒 NULL"压缩成**表级位图**，而 NULL 补行的粒度恰好也是"整表全 NULL"——两者对齐，所以"取并 + 有交集"在语义层面**零损失**。

**逆否：为什么 `IS NULL` 不能转**。`WHERE t2.b IS NULL` 对 t2 **不**拒 NULL ⇒ `not_null_tables = ∅` ⇒ 不转。这正是 `LEFT JOIN ... WHERE t2.pk IS NULL`（反连接惯用法）能保住语义的原因——它不会被"优化"成 INNER JOIN。**"不能转"和"能转"同样重要。**

### 3. 转成内连接后解锁了什么

| 解锁项 | 源码证据要点 |
|---|---|
| **join order 自由度** | 解除"外表优先 + 不交错"两条限制 |
| **谓词下推** | hypergraph 注释：下推会 *"remove rows that should otherwise be output (as NULL-complemented ones)"* |
| **派生表条件下推** | `can_push_condition_to_derived()` 明确排除 `is_inner_table_of_outer_join()` |
| **分区裁剪 / 提前判空** | zero-result 判定带 `!is_inner_table_of_outer_join()` |
| **MIN/MAX 优化** | `opt_sum.cc` 明确跳过外连接表 |
| **函数依赖 / ONLY_FULL_GROUP_BY** | *"weak-to-strong, which is unusable, becomes strong-to-strong"* |
| **join buffering** | 外连接 nest 首张内表不能用 buffer 则整 nest 禁用 |
| **semi-join** | nest 消失后不再被外连接 nest 包裹 |
| **nest 溶解本身** | Pass 2（见下） |

⇒ **这就是"外连接转内连接"收益最大的原因**：一次转换，9 条通道同时打开。而 `simplify_joins()` 头注释的措辞是 **might**——它只承诺"可选计划集变大"，不承诺代价变小。

### 4. 为什么分两个 Pass，顺序能否颠倒

两个 Pass 在同一个函数体内：Pass 1 是主循环（外连转内连 + ON 上提 + `dep_tables` 记账 + 位图上推），Pass 2 是紧随其后的第二个循环（扁平化 + 嵌套 SJ nest 溶解）。

**顺序不可颠倒，四条独立的硬理由**：

**(a) Pass 2 的准入条件由 Pass 1 创造。** Pass 2 判定 `nested_join != nullptr && table->join_cond() == nullptr`。外连接 nest **天生带 ON**，只有 Pass 1 成功后才 `set_join_cond(nullptr)`。⇒ **Pass 2 就是 Pass 1 的收割阶段。**

**(b) 颠倒则收益归零。** 先跑 Pass 2，所有外连接 nest 都还带着 ON，一个都溶不掉——整个"外连接简化"退化成"去括号"。

**(c) Pass 2 要消费 Pass 1 算出的 `dep_tables`。** `tbl->dep_tables |= table->dep_tables;` 把依赖位图下传给孩子，而 `dep_tables` 只在 Pass 1 的 WHERE 那趟才算。

**(d) 颠倒后需要外层不动点循环。** Pass 2 在递归归程上每个 `join_list` 只跑一次；放在 Pass 1 前面，Pass 1 之后新产生的可溶 nest 就没人再溶了。

**Pass 1 内部"递归① 必须先于递归②"是正确性边界**（不是效率考虑），注释逐字写着：

```
Thus, considering this example:
(A LEFT JOIN B ON JC) WHERE W ,
we'll "confront W with A LEFT JOIN B": this will, recursively,
- confront W with B,
- confront W with A.
...
We will not confront JC with B or A, it wouldn't make sense, as JC isn't a
post-filter for their join operation.
```

⇒ **只有"后置过滤器"才有权判定一个外连接能否转内连接。** nest 自己的 ON 是它内部各成员的后置过滤器，对它自己、对它兄弟、对它外层都不是。

**为什么自底向上**（`simplify_joins()` 的 IMPLEMENTATION 注释 *"On the recursive ascent all attributes are calculated..."*）：

1. **数据流强制**：父 nest 的判定输入是 `used_tables`（nest 内叶子表 map 的**并**）和 `not_null_tables`（各成员拒 NULL 位图的**并**），**只能由孩子算出来**
2. **级联方向**：`cond` 自顶向下传，而"cond 变强"自底向上发生。靠"列表逆序 + `fix_fields` 重算"实现**单趟收敛**，不做不动点迭代

**规则驱动还是代价驱动？纯规则，且是刻意的。** 判定式是布尔的，全程没有 cost 结构。

**为什么敢这样**：转换是**语义等价**的（上面的定理），而"更多 join order"意味着搜索空间**只包含式扩大**（原计划仍在新空间里）。所以承诺的是**可选集变大**，不是代价变小。

**代价是转换不可逆**：`propagate_nullability()` 的副作用撤不回来。`subquery_to_derived` 的注释自曝：

```
// We could use LEFT JOIN unconditionally and let simplify_joins()
// convert it to INNER JOIN, but the conversion is not perfect, as
// not all effects of propagate_nullability() are undone.
```

⇒ 8.0 宁可让 `subquery_to_derived` **直接生成 INNER JOIN**，也不走"先 LEFT JOIN 再撤销"这条路。

### 5. 与 derived merge 的关系：咬合的齿轮

两者都在 prepare 期完成，方向相反，且**互为前提、互为目的**：

1. **merge 制造 nest，simplify 溶解 nest**。`merge_underlying_tables()` 把 derived 的 `Table_ref` 原地升级成 `NESTED_JOIN`，这个 nest **注定要被 Pass 2 溶掉**
2. **merge 把判断题转交给 simplify 用通用逻辑回答**。`merge_where()` 总是把 derived 的 WHERE 并进本 nest 的 ON，**不管有没有外连接**：无外连接 → Pass 1 上提到 WHERE → Pass 2 溶解；有外连接且不拒 NULL → ON 不上提 → WHERE 正确留在 ON 位置。**整个 `merge_where` 里没有任何一处 `if (outer_join)` 分支**——正确性完全由上面的定理保证
3. **因果环**：merge 必须先于 simplify（否则拉平后无法做外连转内连）；simplify 又必须后于 merge 才能吃到二阶收益

⇒ **一句话**：**merge 是把"看不见的黑盒"打开成 join 树；simplify 是把 join 树上的"不可动之分"抹掉。** 两者合起来，外层的 join order 才能把内层表真正交错进来。

### 6. 理论溯源，与 MySQL 的差距

- **Rao & Ross,《Outerjoin Simplification and Reordering for Query Optimization》, SIGMOD 1998**——外连接简化（null-rejecting ⇒ 退化）与外连接重排序的理论源头
- **MySQL 源码里没有这些论文的引用痕迹**（`sql/` 下搜 `Rao`/`Galindo` 0 匹配）。8.0.39 里唯一显式引用的 join 序理论是 **[Moe13]**（Moerkotte et al.），且只出现在 hypergraph 侧

**有意思的信号**：MySQL 把 1998 年那套"简化"结论当成常识直接用（连术语都沿用学术界的 *"also known as strong or strict"*），却把 citation 花在了更晚的 join 序枚举理论上——前者的"常识化"程度极高。

**MySQL 相对理论的差距**（源码自曝）：

| 保守点 | 证据 |
|---|---|
| ★ **只实现"简化"半边，基本没实现"重排序"半边** | `FlattenInnerJoins()` 注释逐字承认：*"Note that this (currently) does not do any rewrites to flatten even more. E.g., for the tree (a JOIN (b LEFT JOIN c)), it would be beneficial to use associativity to rewrite into (a JOIN b) LEFT JOIN c ..."*——**明说"有益但没做"**。这正是 Rao & Ross 标题里 "and Reordering" 那半边 |
| 不会"制造"后置过滤器 | 只用**已经是**后置过滤器的 cond 判定 |
| 只处理 LEFT JOIN | RIGHT JOIN 在解析期就被归一化 |
| 没有 FULL OUTER JOIN | 只有枚举值，注释说 *"we will be needing it when we actually implement full outer join"* |
| HAVING 从不参与判定 | 全库只有一个非递归调用点，传 `&m_where_cond` |

⇒ **重排序半边在 8.0 里是以"不重排 + 把非法重排堵住"的形式存在的**：hypergraph 用 conflict rule（`{t2}→{t3}`）堵，老优化器用"外表优先 + 不交错"堵。两条路都是**保守但安全**的——源码明确写着"宁可禁掉合法计划，也绝不允许非法计划"。

### 7. 参数：连接简化**没有**开关

`optimizer_switch_names[]` 里**没有 `outer_join_simplification`**，也没有任何 `outer_join*` / `simplification` 项。`simplify_joins()` 的调用点外层**没有任何 `optimizer_switch` 判断**——它是**无条件执行**的（符合上文：语义等价变换，不是"可选优化策略"）。

相关但间接的开关：

| 开关 | 默认 | 关系 |
|---|---|---|
| `derived_merge` | **ON** | 关掉 ⇒ merge 不发生 ⇒ 二阶效应丢失 |
| `derived_condition_pushdown` | ON | 受 `!is_inner_table_of_outer_join()` 限制 |
| `hypergraph_optimizer` | **OFF** | 换 join order 搜索器，但**不换掉** `simplify_joins` |

唯一能跳过 `simplify_joins()` 的不是开关，而是两个结构性条件：`skip_local_transforms`（INSERT 的部分路径）、建视图时的 `is_view_context_analysis()`。

⇒ 顺带一个可观察现象：**同一个视图定义，在"创建时"和"被查询时"走的简化路径不同**——创建时不简化、不检查 ONLY_FULL_GROUP_BY，实际使用时才报错。

---

## 零、阶段定位与调用链

> ⚠️ **架构纠正**：连接简化属于 **prepare 期的永久变换**，不属于 `JOIN::optimize`。入口有 `assert(first_execution)`（`sql_resolver.cc:754`）——对 Prepared Statement 只做一次，改动直接落在 `Table_ref`/`NESTED_JOIN` 上并被后续所有 execute 复用。

```
Query_block::prepare
  ├─ resolve_placeholder_tables()   sql_resolver.cc:1284
  │     └─ 对每个 view/derived：resolve_derived（内层完整 prepare）→ merge_derived
  │        ★ merge 是自内向外逐层展开
  └─ [仅最外层] apply_local_transforms()   sql_resolver.cc:751
        ├─ delete_unused_merged_columns     :760
        ├─ 递归子块                          :762-765
        ├─ simplify_joins                   :768   ★ 在所有 merge 完成之后
        ├─ record_join_nest_info            :770
        ├─ build_bitmap_for_nested_joins    :771
        ├─ check_only_full_group_by         :797
        └─ push_conditions_to_derived_tables :834
```

**为什么 merge 必须先于 simplify**：merge 把 derived 的表拉平进外层后，`simplify_joins` 才能在拉平后的表列表上做外连接转内连接。顺序反了会漏判。

> ⚠️ **重要纠错**：`REMOVE_OUTER_JOIN`、`outer_join_elimination`、`opt_table_elimination.cc`、`eliminate_tables`、`Eliminate_tables` 在 8.0.39 **全部不存在**（table elimination 是 **MariaDB 5.3+** 的特性，Sergei Petrunia 实现）。MySQL 8.0 的连接简化只有三类：**外连接转内连接**（`OUTER_JOIN_TO_INNER`）、**ON 上提**（`JOIN_COND_TO_WHERE`）、**去括号**（`PAREN_REMOVAL`）+ 半连接拍平（`SEMIJOIN`）。

---

## 一、join nest 数据结构与不变式

### 1.1 四个字段

| 字段 | 位置 | 含义 |
|------|------|------|
| `Table_ref::nested_join` | `sql/table.h:3771` | 非 NULL ⟺ 这是个 **join nest**（不是真实表）。四种来源：外连接内侧、内连接括号、derived/view merge 产物、sj/aj nest |
| `Table_ref::embedding` | `table.h:3773` | 直接包含本表的 nest 的 `Table_ref`；顶层成员为 nullptr |
| `Table_ref::join_list` | `table.h:3775` | 直接包含本表的那个 deque 的地址 |
| `NESTED_JOIN::m_tables` | `sql/nested_join.h:85` | nest 的成员列表 |

`NESTED_JOIN` 关键成员（`nested_join.h:78-144`）：`m_tables`(:85)、`used_tables`(:86)、`not_null_tables`(:87)、`nj_map`(:116)、`sj_depends_on`(:123)、`sj_corr_tables`(:129)、`sj_inner_exprs`(:142)。

### 1.2 八条不变式

1. `t->nested_join != nullptr` ⟺ t 是 join nest（不是真实表）。
2. `t->join_list` 是"直接包含 t 的 deque 地址"：`&t->embedding->nested_join->m_tables`，或（顶层）`&query_block->m_table_nest`。
3. `t->embedding != nullptr` ⟹ `t ∈ *t->join_list` 且 `t->join_list == &t->embedding->nested_join->m_tables`。**每次结构改动处都显式维护**：建 nest(`sql_parse.cc:6278`)、扁平化(`sql_resolver.cc:2184`)、derived merge(`table.cc:4376`)、sj 拉出(`sql_optimizer.cc:6844`)、aj wrap(`sql_resolver.cc:3172`)。
4. `nest->nested_join->used_tables` = 该 nest 下所有叶子表 map 的**并**。唯一赋值点在 simplify_joins（`sql_resolver.cc:2053`），**simplify 之前它无效**。
5. simplify 之后：`nested_join != nullptr` ⟹ `join_cond() != nullptr || is_sj_nest()`（由 `sql_optimizer.cc:5032` 的断言强制）。
6. `outer_join` 标志**只在右操作数上**，RIGHT JOIN 已归一化。
7. **`m_tables`/`m_table_nest` 是查询文本的逆序**（见 2.1）。
8. `Query_block::m_table_nest`（`sql_lex.h:2124`）是本 query block 的最外层 join 列表；`m_current_table_nest` + `Query_block::embedding`（`:2126/:2128`）是**只在 contextualize 期使用的游标对偶**。

---

## 二、两个前置归一化

### 2.1 RIGHT JOIN → LEFT JOIN（`parse_tree_nodes.cc:153-181`）

```cpp
bool PT_joined_table::contextualize_tabs(Parse_context *pc) {
  bool was_right_join = m_type & JTT_RIGHT;
  if (was_right_join) {
    m_type = static_cast<PT_joined_table_type>((m_type & ~JTT_RIGHT) | JTT_LEFT);
    std::swap(m_left_pt_table, m_right_pt_table);      // :161 交换两侧
  }
  ...
  if (m_type & JTT_LEFT) {
    m_right_table_ref->outer_join = true;              // :176 只有右操作数带 outer_join
    if (was_right_join) {
      m_right_table_ref->join_order_swapped = true;    // 仅供 EXPLAIN/SHOW CREATE VIEW 回显
      m_right_table_ref->query_block->set_right_joins();
    }
  }
}
```

**算法意义**：优化器内部**只有 LEFT JOIN**（`table.h:3723-3726` 注释）。这是第一个"连接简化"：6 种 join 语法归一到 2 种。

### 2.2 join_list 是逆序的（`sql_parse.cc:6321`）

```cpp
bool Query_block::add_joined_table(Table_ref *table) {
  m_current_table_nest->push_front(table);   // ★ push_FRONT
  table->join_list = m_current_table_nest;
  table->embedding = embedding;
}
```

所以 `FROM t1 LEFT JOIN t2 ... LEFT JOIN t3 ...` 在 `m_table_nest` 里遍历顺序是**自右向左**。这直接解释了 `simplify_joins` 头注释那句话：

> *"As join list contains join tables in the reverse order sequential elimination of outer joins does not require extra recursive calls."*

即：**级联转换只需单趟从右到左扫描，不需要迭代到不动点**——右侧 LEFT JOIN 的 ON 并入 WHERE 后，左侧 nest 在同一趟后续迭代就能看到被加强的 WHERE。

### 2.3 单表/空 nest 消除在**解析期**（`sql_parse.cc:6236`）

```cpp
Table_ref *Query_block::end_nested_join() {
  ptr = embedding;
  m_current_table_nest = ptr->join_list;      // 游标回升
  embedding = ptr->embedding;
  nested_join = ptr->nested_join;
  if (nested_join->m_tables.size() == 1) {    // 单成员 nest → 直接提升，nest 消失
    Table_ref *embedded = nested_join->m_tables.front();
    m_current_table_nest->pop_front();
    embedded->join_list = m_current_table_nest;
    embedded->embedding = embedding;
    m_current_table_nest->push_front(embedded);
    ptr = embedded;
  } else if (nested_join->m_tables.empty()) { // 空 nest → 删除
    m_current_table_nest->pop_front();
    ptr = nullptr;
  }
  return ptr;
}
```

`FROM (t1)` 在 contextualize 阶段就不会留下 `NESTED_JOIN`。**所以不要说"simplify_joins 消除单表 nest"**。

---

## 三、simplify_joins Pass 1：外连接转内连接

`sql/sql_resolver.cc:1951-2218`。

核心语义（注释 `:1974-1996`）：**把"后过滤器 cond"与 nest 里每个成员"对质"**。若 cond 对某张内表的 NULL-complemented 行"拒绝 NULL"，该外连接可转内连接。

### 3.1 递归结构：两次递归 + 位图上推（`:1997-2056`）

```cpp
for (Table_ref *table : *join_list) {
  table_map used_tables;
  table_map not_null_tables = 0;
  NESTED_JOIN *nested_join = table->nested_join;

  if (nested_join != nullptr) {
    if (table->join_cond() != nullptr) {
      Item *join_cond = table->join_cond();
      // 递归①：用 nest 自己的 ON 去简化 nest 内部
      simplify_joins(thd, &nested_join->m_tables, false,
                     in_sj || table->is_sj_or_aj_nest(), &join_cond, changelog);
      if (join_cond != table->join_cond()) {
        table->set_join_cond(join_cond);
        if (table->is_sj_or_aj_nest() && join_cond->const_item())
          clear_sj_expressions(nested_join);
      }
    }
    nested_join->used_tables     = 0;      // ★ 清零
    nested_join->not_null_tables = 0;      // ★ 清零
    // 递归②：把外层 cond 与 nest 每个成员逐一对峙
    simplify_joins(thd, &nested_join->m_tables, top,
                   in_sj || table->is_sj_or_aj_nest(), cond, changelog);
    used_tables     = nested_join->used_tables;
    not_null_tables = nested_join->not_null_tables;
  } else {
    used_tables = table->map();                                    // 叶子表
    if (*cond != nullptr) not_null_tables = (*cond)->not_null_tables();
  }

  if (table->embedding != nullptr) {                               // 位图上推
    table->embedding->nested_join->used_tables     |= used_tables;
    table->embedding->nested_join->not_null_tables |= not_null_tables;
  }
```

**逐段解释**：

- **递归①**（用 nest 自己的 ON 简化内层）：对 `(A LJ (B LJ C ON JC2) ON JC1) WHERE W`，`JC1` 是内层 `B LJ C` 这个运算的**后置过滤器**，所以可用它判定 `B LJ C` 能否转内连接。**反过来绝不拿 JC1 去简化 A 或外层**——正确性边界。
- **★ 清零（`:2039-2040`）**：递归① 期间孩子们累加进 `nested_join` 位图的内容**被全部丢弃**。递归① 的唯一目的是副作用（转换更内层）；它算出的位图基于 ON 而非外层 cond，对本层判定**无意义**。真正参与本层判定的位图只能来自递归②。
- **递归②**（外层 cond 与每个成员对峙）：`cond` 不变、`top` 不变原样传下去。含义：`(A LJ B ON JC) WHERE W` 中 W 是外部过滤器，若 W 在 B 被 NULL 补齐时为 FALSE，则 LJ 可转 JOIN。
- **叶子表**：`not_null_tables = cond->not_null_tables()`（**未按本表裁剪**，裁剪由 3.2 的 `&` 完成）。
- **方向总结**：nest 树上 `cond` **自顶向下**传递；`used_tables/not_null_tables` 合并与**判定是自底向上（后序）**。同层内因逆序是**从右往左**扫描，cond 单调增长，天然实现级联的一趟收敛。

### 3.2 判定式与形式化证明（`:2058-2102`）

```cpp
if (!table->outer_join || (used_tables & not_null_tables)) {
  if (table->outer_join) {
    *changelog |= OUTER_JOIN_TO_INNER;
    table->outer_join = false;                      // ★ 唯一清掉 outer_join 的地方
  }
  if (table->join_cond() != nullptr) {
    *changelog |= JOIN_COND_TO_WHERE;
    Item *i1 = *cond, *i2 = table->join_cond();
    // 视图行级过滤必须排在用户存储过程之前（防止行泄露）
    if (table->is_view() && i1->has_stored_program()) std::swap(i1, i2);
    Item_cond_and *new_cond = down_cast<Item_cond_and *>(and_conds(i1, i2));
    new_cond->apply_is_true();                      // ★ 必须
    Item *cond_after_fix = new_cond;
    if (new_cond->fix_fields(thd, &cond_after_fix)) return true;   // ★ 重算 not_null_tables
    *cond = cond_after_fix;
    table->set_join_cond(nullptr);                  // ON 清空 → Pass 2 的入场券
  }
}
```

**形式化证明**：令 `A LJ B ON JC` 的结果 `R = (A ⋈_JC B) ∪ N`，其中 `N` 是 NULL 补齐元组集。施加后置过滤器 `W`（WHERE 语境下 UNKNOWN ≡ FALSE）。

若 W 对某张 `T ⊆ B` 拒绝 NULL（T 全列为 NULL 时 W 必为 FALSE/UNKNOWN），则：

```
σ_W(N) = ∅  ⟹  σ_W(R) = σ_W(A ⋈_JC B)  ⟹  σ_W(A LJ B ON JC) = σ_{W ∧ JC}(A × B)
```

两个结论对应代码的两件事：`outer_join = false`（LJ⇒⋈）与 `*cond = cond AND join_cond`（JC 从 ON 挪到 post-filter）。**第 2 步是第 1 步的必然推论**，且是必需的——把 JC 移出去才能让 nest 变成"无 ON 的纯括号"，从而被 Pass 2 扁平化。

**`apply_is_true()` 与 `fix_fields` 都是必需的**：前者保证新 AND 走"取并集"而非"取交集"；后者**重算 `not_null_tables_cache`**，这是级联转换的关键。

### 3.3 not_null_tables 的三条规则与判例表

`not_null_tables()`（`item.h:2241`）语义：**"若这些表（因外连接）被 NULL-complemented，该表达式会变成 NULL"**。注意它针对**外连接的 NULL 补行**，不是列 NOT NULL 约束。

| 规则 | 位置 |
|---|---|
| 默认：`not_null_tables() = used_tables()` | `item.h:2241` |
| **`null_on_null` 是总闸**：`if (null_on_null) not_null_tables_cache \|= args[i]->not_null_tables();` | `item_func.cc:719` |
| `null_on_null` 默认 true；`Item_func_sp`、CASE 系列等为 false | `item_func.h:153-166` |

**AND / OR 的差异**（`item_cmpfunc.cc:5514` / `:5616`）：

```cpp
if (func_type == COND_AND_FUNC && ignore_unknown()) {
  not_null_tables_cache = 0;                              // AND：从空集开始
  ...
  not_null_tables_cache |= item->not_null_tables();       // 取并集
} else {
  not_null_tables_cache = ~(table_map)0;                  // OR：从全集开始
  ...
  not_null_tables_cache &= item->not_null_tables();       // 取交集
}
```

- **AND 取并集**：任一合取项在 T 为 NULL 时为 FALSE ⇒ 整个 AND 为 FALSE ⇒ T 拒 NULL。
- **OR 取交集**：OR 为 FALSE 需要**所有**分支都 FALSE。
- `ignore_unknown()`（= `abort_on_null`）由 `apply_is_true()` 设置。**WHERE/ON/HAVING 的顶层条件都会被 `apply_is_true()` 并沿 AND/OR 递归下传**。

**判例表**（都能从上面规则推出）：

| WHERE 谓词 | not_null_tables | 能否转内连接 | 依据 |
|---|---|---|---|
| `t2.b < 5` | `{t2}` | ✅ | `Item_func_lt`，null_on_null=true |
| `t2.b IS NOT NULL` | `{t2}` | ✅ | `Item_func_isnotnull` 构造时 null_on_null=false，但 **`apply_is_true()` 把它设回 true** |
| `t2.b IS NULL` | `∅` | ❌ | `Item_func_isnull` 的 null_on_null=false 且**无** `apply_is_true` 覆写 |
| `t2.b=1 OR t1.c=2` | `∅` | ❌ | OR 取交集 |
| `t2.b=1 OR t2.c=2` | `{t2}` | ✅ | 交集仍含 t2 |
| `IF(t1.x, t2.a, 1)` | `∅` | ❌ | IF：`T1(e1)∩T1(e2)` |
| `t2.a BETWEEN t1.x AND t1.y` | `{t2}∪{t1}` | ✅ | BETWEEN 的三参数合并 |
| 含子查询的谓词 | 子查询部分为 0 | 取决于外层 | `Item_subselect::not_null_tables()` 恒 0 |

> **`t2.b IS NULL` 返回 ∅ 正是 `LEFT JOIN ... WHERE x IS NULL` 反连接惯用法能保住语义的原因**——它不会把 LEFT JOIN 转成 INNER JOIN。

### 3.4 级联完整走查

```sql
SELECT * FROM t1 LEFT JOIN t2 ON t2.a=t1.a
                 LEFT JOIN t3 ON t3.b=t2.b
 WHERE t3.c IS NOT NULL
```

结构 `nest2{ nest1{t1,t2}, t3 }`，逆序扫描：

1. **t3**：`W = t3.c IS NOT NULL`，`not_null_tables = {t3}`，交集非空 → t3 转内连接；`W := W AND t3.b=t2.b`，**`fix_fields` 重算 `not_null_tables = {t2,t3}`**。
2. **nest1**（含 t1,t2）：`used_tables={t1,t2}`，新的 `W` 的 not_null_tables = `{t2,t3}` → 交集 `{t2}` 非空 → t2 转内连接；`W := W AND t2.a=t1.a`。
3. **结果**：`FROM t1, t2, t3 WHERE t3.c IS NOT NULL AND t3.b=t2.b AND t2.a=t1.a`。

**反例**：把 `IS NOT NULL` 改成 `IS NULL` → not_null_tables = ∅ → t3 的 LEFT JOIN 保留，t2 的也保留（W 未被加强），整棵 nest 树原样不动。

### 3.5 dep_tables 记账（`:2104-2153`，仅 `top` 那趟）

```cpp
if (!top) continue;                                    // ★ 钉死在唯一一趟

if (table->join_cond() != nullptr) {                   // 只剩真外连接
  table->dep_tables |= table->join_cond()->used_tables();
  table->dep_tables &= ~table->embedding->nested_join->used_tables;   // 剔除同 nest
  table->embedding->join_cond_dep_tables |= table->join_cond()->used_tables();
}
if (prev_table != nullptr) {
  if (prev_table->straight || straight_join)
    prev_table->dep_tables |= used_tables;             // STRAIGHT_JOIN 硬编码顺序
  if (prev_table->join_cond() != nullptr) {
    prev_table->dep_tables |= table->join_cond_dep_tables;
    // "至少一张外表必须先行"规则
    if ((((prev_table->join_cond()->used_tables() & ~PSEUDO_TABLE_BITS)
          & ~prev_used_tables) & used_tables) == 0)
      prev_table->dep_tables |= used_tables;
  }
}
prev_table = table;
```

- **2107 `if (!top) continue`**：一张表会被访问多次（WHERE 趟 + 每个包含它的 nest 的 ON 趟）。`dep_tables` 是幂等 OR，但 `prev_table` 链和 `join_cond_dep_tables` 传播若跑多趟会污染语义。
- **2146-2150**：若 ON 完全不引用左邻表（如 `ON RAND()>0.5`、`ON 1=1`），单看 ON 无法产生依赖，但执行器要求外连接内侧至少有一张外表在前——无条件补上。`& ~PSEUDO_TABLE_BITS` 排除 `RAND_TABLE_BIT`，否则 `ON ...RAND()...` 会让掩码非零从而跳过补偿。

---

## 四、simplify_joins Pass 2：嵌套 join 扁平化

`sql_resolver.cc:2156-2196`。

```cpp
for (auto li = join_list->begin(); li != join_list->end();) {
  Table_ref *table = *li;
  NESTED_JOIN *nested_join = table->nested_join;
  if (table->is_sj_nest() && !in_sj) {
    *changelog |= SEMIJOIN;                            // ★ 顶层 SJ nest 保留
  } else if (nested_join != nullptr && table->join_cond() == nullptr) {
    *changelog |= PAREN_REMOVAL;
    for (Table_ref *tbl : nested_join->m_tables) {
      tbl->embedding  = table->embedding;              // 孩子改认祖父
      tbl->join_list  = table->join_list;
      tbl->dep_tables |= table->dep_tables;
    }
    li = join_list->erase(li);
    li = join_list->insert(li, nested_join->m_tables.begin(),
                           nested_join->m_tables.end());
    continue;                                          // ★ 不推进迭代器
  }
  ++li;
}
```

### 4.1 扁平化的充要条件

**`nested_join != nullptr && join_cond() == nullptr`，且不是"顶层 SJ nest"。**

| nest 类型 | join_cond | is_sj_nest | is_aj_nest | 扁平化？ |
|---|---|---|---|---|
| 内连接括号 `(t1,t2)` | Pass 1 后 = nullptr | false | false | ✅ |
| 已转内连接的 outer join nest | Pass 1 已清空 | false | false | ✅ |
| 仍是真外连接 | ≠ nullptr | false | false | ❌ |
| SJ nest（顶层，in_sj=false） | nullptr | **true** | false | ❌ 被第一个 if 截住 |
| SJ nest（嵌在别的 SJ/AJ 里） | nullptr | true | false | ✅ |
| AJ nest | ≠ nullptr | false | **true** | ❌ 永不 |

`is_sj_nest()` / `is_aj_nest()`（`table.h:2989-2994`）：
```cpp
bool is_sj_nest() const { return m_is_sj_or_aj_nest && !m_join_cond; }
bool is_aj_nest() const { return m_is_sj_or_aj_nest && m_join_cond; }
```

### 4.2 为什么 AJ 不能溶解、SJ 可以

- `A SJ (B SJ C)` ≡ `A SJ (B ⋈ C)`：SJ 只关心"至少一个匹配"，内层重复行不影响外层语义 → 可溶解。
- `A AJ (B SJ C)` ≡ `A AJ (B ⋈ C)`：同理。
- `A SJ (B AJ C)` **≢** `A SJ (B ⋈ C)`：AJ 是"不存在匹配"，换成 `⋈` 直接反转语义 → 禁止。代码靠"AJ 一定带 join_cond"这个不变式天然拦住。

### 4.3 `continue` 不推进迭代器 —— 一趟完成传递闭包

插入孩子后下一轮重新检查刚插入的第一个孩子。若它自己也是可扁平 nest，继续溶解。**单趟**彻底拉平多层括号，不需要外层不动点循环。

### 4.4 收尾两个断言

- `record_join_nest_info`（`:2234`）：填 `Query_block::outer_join`（simplify 后仍是外连接内侧的表位图）、`sj_nests`。**只有 simplify 之后它才有意义**——`check_only_full_group_by` 依赖它。
- `build_bitmap_for_nested_joins`（`sql_optimizer.cc:5025`）：`:5032` 的断言 `assert((join_cond() != nullptr) || is_sj_nest())` 是 **simplify 正确性的运行时校验**。

---

## 五、derived / view / CTE merge

`merge_derived`（`sql_resolver.cc:3462-3705`）。

### 5.1 三层门禁

**第 1 层：技术硬约束 `Query_expression::is_mergeable()`（`sql_lex.cc:3807`）**

```cpp
bool Query_expression::is_mergeable() const {
  if (is_set_operation()) return false;
  Query_block *const select = first_query_block();
  return !select->is_grouped() && select->having_cond() == nullptr &&
         !select->is_distinct() && select->has_tables() &&
         !select->has_limit() && !select->has_windows();
}
```

| # | 不可 merge 条件 | 为什么 |
|---|---|---|
| 1 | `is_set_operation()` | UNION/EXCEPT/INTERSECT 无法作为单个 join 项（实现限制） |
| 2 | `is_grouped()`（含隐式聚合） | 改变基数与求值时机 |
| 3 | `having_cond() != nullptr` | HAVING 语义上在分组后求值 |
| 4 | `is_distinct()` | 去重是基数变换 |
| 5 | `!has_tables()` | `(SELECT 1)` merge 后 nest 为空 |
| 6 | `has_limit()` | LIMIT 作用于子查询行数，外层无处表达 |
| 7 | `has_windows()` | 窗口函数需要确定的行集 |

**第 2 层：CTE 确定性约束 `Table_ref::is_mergeable()`（`table.cc:6488`）**

```cpp
Common_table_expr *cte = common_table_expr();
if (cte != nullptr && cte->references.size() >= 2 &&
    derived->uncacheable & UNCACHEABLE_RAND)
  return false;
```

CTE 语义是"物化一次、所有引用看到同一份内容"；merge 是把定义**复制**到每个引用点，两次引用会得到不同随机值 → 禁止。

**第 3 层：策略/环境门禁（`merge_derived:3465-3518`）**

优先级链（`:3486-3504`）：**ALGORITHM > hint(`MERGE`/`NO_MERGE`) > `optimizer_switch=derived_merge` + heuristic**。

`merge_heuristic`（`sql_lex.cc:3836`）：
```cpp
if (lex->set_var_list.elements != 0) return false;              // 有 @v:= 赋值
for (Item *item : select->visible_fields())
  if (item->has_subquery() && !item->const_for_execution()) return false;  // 非常量子查询
```
（后者：用户很可能故意用 derived 求物化，避免子查询被外层每行重算。）

补充门禁：外层 `STRAIGHT_JOIN` 且 derived 内有 sj/aj nest（`:3512`）、表数超 `MAX_TABLES`（`:3517`）。

调用点 `resolve_placeholder_tables`（`:1290-1321`）：**先 `resolve_derived`（内层完整 prepare）再 merge**，所以 merge 自内向外；`assert(derived_query_expression->is_prepared())`（`:3472`）是守卫。

### 5.2 merge 的 12 个步骤（关键几步）

**步骤 1-3：把 derived 的 Table_ref 原地升级成 NESTED_JOIN**

```cpp
// sql_resolver.cc:3563
if (!(derived_table->nested_join = new (thd->mem_root) NESTED_JOIN)) return true;
if (derived_table->merge_underlying_tables(derived_query_block)) return true;

// sql/table.cc:4372
bool Table_ref::merge_underlying_tables(Query_block *select) {
  for (Table_ref *tl : select->m_table_nest) {       // ★ 用内层的顶层 join 列表
    tl->embedding = this;
    tl->join_list = &nested_join->m_tables;
    nested_join->m_tables.push_back(tl);
  }
}
```

**核心思想**：不新建 Table_ref，而是把 derived 自己的 `Table_ref` 原地升级成 `NESTED_JOIN`。外层 join 树形状（`embedding`/`join_list`/`outer_join`/`join_cond`）完全不用改——"这个位置是一张表"变成"这个位置是一个括号 nest"。

**步骤 4：leaf_tables 拼接 + 表号平移**
```cpp
leaf->dep_tables <<= table_adjust;     // :3575  位图左移即完成重编号
```
并累加 `cond_count`/`between_count`（`:3594`，决定 KEYUSE/SEL_ARG 数组预分配大小，漏加会越界）。

**步骤 5：nullability 传播**（`:3609` → `propagate_nullability`，`sql_resolver.cc:4041`）
```cpp
void propagate_nullability(mem_root_deque<Table_ref *> *tables, bool nullable) {
  for (Table_ref *tr : *tables) {
    if (tr->table && !tr->table->is_nullable() && (nullable || tr->outer_join))
      tr->table->set_nullable();
    if (tr->nested_join == nullptr) continue;
    propagate_nullability(&tr->nested_join->m_tables, nullable || tr->outer_join);
  }
}
```
derived 原本是一张"可为 NULL 的表"，merge 后底层 `TABLE` 必须显式标 nullable，否则 NULL 补行会被误判。

**★ 步骤 6：merge_where —— WHERE 合进 ON，不是合进 WHERE（`table.cc:4487`）**

```cpp
bool Table_ref::merge_where(THD *thd) {
  Item *const condition = derived_query_expression()->first_query_block()->where_cond();
  if (!condition) return false;
  derived_where_cond = condition;                        // 单独保存（已 fixed）
  set_join_cond(and_conds(join_cond(), condition));      // ★ 总是并入本 nest 的 ON
  return join_cond() == nullptr;
}
```

**这是最重要的算法点**：derived 的 WHERE **总是先并入本 nest 的 ON**，无论有无外连接：

- **无外连接**：nest 是"带 ON 的括号"，`simplify_joins` Pass 1 会把 ON 上提到外层 WHERE，Pass 2 再扁平化 → derived 的 WHERE 与外层 WHERE 自动合并。
- **有外连接**：nest 的 `outer_join == true`，若外层 WHERE 不拒 NULL 则不上提 → derived 的 WHERE **正确留在 ON 位置**。

**不需要任何分支判断——正确性完全由后续 simplify_joins 的通用逻辑保证**。`SELECT * FROM t1 LEFT JOIN (SELECT * FROM t2 WHERE t2.x=1) d ON d.a=t1.a` 里的 `t2.x=1` 绝不会被错误提到外层 WHERE。

> **HAVING 不需要处理**——`is_mergeable()` 已排除 `having_cond() != nullptr`。唯一需处理的非 WHERE 子句是 ORDER BY。

**步骤 7：field_translation —— 列引用重定向表（`table.cc:4537`）**

建 `(列名 → 内层 select-list Item*)` 映射。外层对 `d.c` 的解析命中它，产出 `Item_direct_view_ref` 包装。**不复制表达式，共享指针 + 包装**。

**步骤 8-10：摘除内层、重编号、修表号**
```cpp
derived_query_expression->exclude_level();                     // :3620 保留更深层子查询
derived_table->set_derived_query_expression((Query_expression *)1);  // :3623 poison
merge_contexts(derived_query_block);                           // :3626
repoint_contexts_of_join_nests(derived_query_block->m_table_nest);   // :3628
remap_tables(thd);                                             // :3631
fix_tables_after_pullout(this, derived_query_block, derived_table,
                         table_adjust, ...);                   // :3634
```
`fix_tables_after_pullout`（`:2318`）递归修复已 fixed 但表号变了的 Item 的 `used_tables_cache`/`not_null_tables_cache`。**它也重算 `not_null_tables_cache`**——这是 `simplify_joins` 稍后能用它判定的前提。

**步骤 11：ORDER BY —— 多数情况被丢弃**（`:3638-3690`）

只在极窄条件下上提：语句是 SELECT/单表 UPDATE/单表 DELETE，且外层不是 set operation、不分组、不 DISTINCT、自身无 ORDER BY、且**外层 FROM 只有这一个表引用**。否则直接 `empty_order_list` 丢弃，trace 打 `removed_ordering`。

### 5.3 视图合并 = derived merge（同一个函数）

**共用 `Query_block::merge_derived()`**，差异只有 6 处：

| 差异 | view | derived/CTE |
|---|---|---|
| `ALGORITHM=MERGE/TEMPTABLE` | 有（来自 DD） | 无 |
| `allow_merge_derived` 闸门 | **豁免** | 受约束 |
| 可更新/可插入性重算 | 有（`:3546-3560`） | 无 |
| simplify_joins 的 SP 顺序保护 | 有（`:2078`） | 无 |
| trace 标签 | `"view"` | `"derived"` |

`optimizer_switch=derived_merge` 对两者都生效。

### 5.4 递归 CTE 不能 merge 的三重原因

1. **形式约束**：递归 CTE 必须是 UNION（`sql_derived.cc:324`），而 `is_mergeable()` 第一行 `if (is_set_operation()) return false`。
2. **语义/算法**：递归语义是**最小不动点迭代**，关系代数里没有等价的有限 join 表达式。执行器用 `MaterializeIterator::MaterializeRecursive()`（`composite_iterators.cc:938`）循环物化直到 `stored_rows` 不再增长。
3. **实现机制**：递归自引用在 prepare 期就被替换成临时表克隆（`Common_table_expr::substitute_recursive_reference`，`sql_derived.cc:240`），`exclude_tree()` + `set_derived_query_expression(nullptr)` 之后它已不是 derived table。执行期走 `FollowTailAccessPath`。

### 5.5 merge 之后为什么能用上索引（四条机制）

| 机制 | 说明 |
|---|---|
| **1. 投影替换** | `d.a` 变成指向 `Item_field(t.a)` 的 `Item_direct_view_ref`，`used_tables()` 等于 `{t}` → `update_ref_and_keys` 能识别为 KEYUSE（ref access）、范围优化器能建 SEL_ARG（range scan） |
| **2. WHERE 合并** | `merge_where` + `simplify_joins` 把两层 WHERE 合成单一 AND 树，等值传播/常量传播能做全局推理（物化路径被临时表边界切断） |
| **3. nest 扁平化** | 内层表可与外层表**任意交错**排序（不扁平则临时表是不可分割单元） |
| **4. 二阶效应** | merge 后外层对 `d.c` 的谓词变成对 `t.c` 的谓词，`not_null_tables` 含 t → LEFT JOIN 可被转 INNER JOIN |

---

## 六、LATERAL：让派生表看见同层兄弟表

`LATERAL` 是 SQL:1999 保留字（源码 token 注释就标着 `/* SQL-1999-R */`），作用是**让 FROM 里的派生表能引用同一 FROM 子句中排在它左边的表**。

### 6.1 本质：改的是"外层名字解析上下文"

派生表的"外层"是谁，由 `lateral` 这一个标志决定（`parse_tree_nodes.cc`，`PT_derived_table::contextualize`）：

```cpp
/*
  Determine the immediate outer context for the derived table:
  - if lateral: context of query which owns the FROM i.e. outer_query_block
  - if not lateral: context of query outer to query which owns the FROM.
*/
if (!m_lateral) {
  pc->thd->lex->push_context(outer_query_block->context.outer_context);
}
```

**非 LATERAL 时跳过一层**（派生的"外层"是拥有 FROM 的查询的再外层），于是同一 FROM 的兄弟表**不可见**；**LATERAL 时不跳**，兄弟表进入可见范围。

对应的硬检查在名字解析时（`item.cc`）：

```cpp
// A non-lateral derived table cannot see tables of its owning query
if (place == CTX_DERIVED && select->end_lateral_table == nullptr) continue;
```

`continue` 表示跳过本层继续往外层找，全部找不到就报 `ER_BAD_FIELD_ERROR`（Unknown column）——这就是没有 LATERAL 时的症状。

`end_lateral_table` 则负责**只让左边的表可见**：

```cpp
if (first_table && first_table->query_block &&
    first_table->query_block->end_lateral_table)
  last_table = first_table->query_block->end_lateral_table;
```

所以 `FROM lateral (select t1.a) dt LEFT JOIN t1` 会报错——`t1` 在右边。

### 6.2 易混淆：相关派生表 ≠ LATERAL

MySQL **允许**派生表引用"更外层查询"的表（相关派生表，EXPLAIN 显示 `DEPENDENT DERIVED`）；**不允许**的是引用同一 FROM 的兄弟表。LATERAL 补的正是后者。

判定标准在 `sql_resolver.cc`（semi-join 上拉时派生表如何"变成" LATERAL）：

> *"some outer ref is now a neighbour in FROM: we have made 'tr' LATERAL"*

即 **有外层引用 + 引用目标在同一 FROM ⇒ 就是 LATERAL**。反过来说，LATERAL 就是"写进 FROM 的相关子查询"。

### 6.3 m_lateral_deps：标记在 Query_expression 上

⚠️ 一个反直觉的点：lateral 标记**不在 `Table_ref`**（`sql/table.h` 搜 `lateral` 零命中，也没有 `Table_ref::is_lateral()`），而在 `Query_expression`：

```cpp
/**
  If 'this' is body of lateral derived table:
  map of tables in the same FROM clause as this derived table, and to which
  the derived table's body makes references.
  In pre-resolution stages, this is OUTER_REF_TABLE_BIT, just to indicate
  that this has LATERAL; after resolution ... this is the proper map.
*/
table_map m_lateral_deps;
```

三态语义：未写 LATERAL → `0`；解析前写了 → `OUTER_REF_TABLE_BIT`（只是打标记）；解析后 → 真实表集合（去掉伪表位）。

若写了 LATERAL 但体内实际没引用任何同层表，会退回 `0`（源码注释：*"it will be handled as if LATERAL hadn't been specified"*）。

### 6.4 依赖如何变成 join 顺序约束（呼应 3.5）

`m_lateral_deps` 最终并进 `dep_tables`——与 3.5 节的外连接、`STRAIGHT_JOIN` 走**同一条记账链**：

```cpp
dep_tables |= derived->m_lateral_deps;
```

再由 `tab->dependent = tl->dep_tables` 传给计划器，闸门是 `best_extension_by_limited_search` 里的：

```cpp
!(remaining_tables & s->dependent)
```

只要该表还依赖任何未进计划的表，就不能放到当前位置。

⚠️ **这里没有专门的错误消息**——"依赖表必须排在前"完全靠计划搜索**静默剪枝**保证。`share/messages_to_clients.txt` 里没有任何 `ER_LATERAL_*` 错误码。

补充：derived 被 merge 后，lateral 属性下放到内部表、自身清零——*"The 'laterality' of this nest is not interesting anymore; it was transferred to underlying tables."*

### 6.5 执行语义：不是字面的"每行重算"

精确说法是：**对其依赖表中"在计划里排最后"的那张表，每换一行重新物化一次**。

```cpp
// We identified the last dependency of table_ref in the plan, and it's
// the table whose reading must trigger rematerialization of table_ref.
```

只挂最后一张是为了减少无谓刷新（`sql_executor.h`：*"for efficiency (less useless calls to QEP_TAB::refresh_lateral())"*）。测试里数过：t1 两行、t2 通过 join 产生多行，但 `handler_write` 仍是 2——**依赖行没变就不重算**。

运行时靠 `CacheInvalidatorIterator` 的 `generation` 计数触发（`Init()` / `Read()` / `SetNullRowFlag()` 都 ++），`MaterializeIterator` 比较代次决定是否重算。

### 6.6 代价与限制

| 限制 | 说明 |
|---|---|
| **禁用 join buffer** | 缓冲打乱行的到达顺序会导致反复重算。源码原话 *"it's very inefficient. So we forbid join buffering"* |
| **不能作为 hash join 右侧** | hypergraph 把 lateral 依赖转成 `AccessPath::parameter_tables`，非零就不能上 hash join 右支 |
| **代价模型** | `lateral_derived_cost` = 单次物化代价 × 重算次数 ÷ 读取次数（optimizer trace 有 `lateral_materialization` 节点） |
| **只能引用左边的表** | `end_lateral_table` 机制 |
| **必须带别名** | 语法层就报 `ER_DERIVED_MUST_HAVE_ALIAS` |
| **不能引用 SELECT 列表别名** | `is_item_list_lookup = false` |
| **LATERAL 是保留字** | `create table lateral(a int)` 报 parse error |

**聚合的处理是"最省事的实现"，不是标准**：lateral 派生表里的聚合不会解析到直接外层（因为读 FROM 表发生在聚合之前），测试注释直言 *"This was the simplest behaviour to implement"*，并指出 SQL Server 和 PG 会拒绝这种查询。

### 6.7 与共享物化的冲突（唯一与 CTE 的交集）

这是 LATERAL 与 CTE 唯一的交界，也是 [`runtime/03_cte.md`](../../runtime/03_cte.md) 里那个 TODO 的由来。

共享物化的语义是"多个引用共用一份内容、读可以交错"；lateral 的语义是"每行要有自己的内容"——**本质冲突**。代码用 `m_rematerialize` 把关：

```cpp
const bool use_shared_cte_materialization =
    !table()->materialized && m_cte != nullptr && !m_rematerialize && ...
```

只要 `m_rematerialize == true`，共享物化就被短路。而判断"依赖行变了没有"的 `generation` 只能说明"看到新行"、不能说明"LATERAL 真正用到的列值变了"，于是有了那个保守的 TODO：

> *"TODO: It would be better, although probably much harder, to check the actual column values instead of just whether we've seen any new rows."*

源码里"lateral CTE"是**正式承认的概念**（`sql_resolver.cc`）：*"it means we now have a 'lateral CTE'"*。

---

## 七、semi-join nest 收缩（pull_out_semijoin_tables）

`sql_optimizer.cc:6756-6867`。这是 MySQL 里**唯一**用"唯一键 ⇒ 至多一行 ⇒ 语义不变"论证来改动 join 结构的算法，与 MariaDB table elimination 的正确性论证同源。

```cpp
table_map dep_tables = 0;
for (Table_ref *tbl : sj_nest->nested_join->m_tables)
  if (tbl->dep_tables & sj_nest->nested_join->used_tables)
    dep_tables |= tbl->dep_tables;                    // 被别人依赖的不能拉出

bool pulled_a_table;
do {                                                  // ★ 不动点循环
  pulled_a_table = false;
  for (Table_ref *tbl : sj_nest->nested_join->m_tables) {
    if (tbl->table && !(pulled_tables & tbl->map()) && !(dep_tables & tbl->map())) {
      if (find_eq_ref_candidate(tbl, sj_nest->nested_join->used_tables & ~pulled_tables)) {
        pulled_a_table = true;
        pulled_tables |= tbl->map();
        sj_nest->nested_join->sj_corr_tables |= tbl->map();
        sj_nest->nested_join->sj_depends_on  |= tbl->map();
      }
    }
  }
} while (pulled_a_table);
```

**算法**：
1. `dep_tables`：nest 内被别人依赖的表不能拉出（如 `t1 SJ (t2 LJ t3) ON t1.a=t2.pk` 里 t2 不能拉出，t3 依赖它）。
2. `find_eq_ref_candidate` 判断 tbl 能否被剩余表通过 **eq_ref**（唯一/主键全 keypart 等值绑定、键不可为 NULL）访问。若能，说明 tbl 对已绑定表**函数依赖**（每个外侧组合最多匹配 1 行），拉出它不影响去重语义。
3. **不动点**：拉出一张表会解锁另一张（新拉出的表变成可提供的绑定），所以迭代。
4. **代价**：拉出会把原本不相关的子查询变成相关子查询（`sj_corr_tables`/`sj_depends_on`），禁掉 Materialization 和 LooseScan 策略。

`:6852-6857`：SJ nest 被掏空后整体删除，semi-join 退化为普通内连接——**这是 MySQL 里最接近"nest 消除"的地方**。

---

## 八、已知限制与"本版本不具备的能力"

| 限制 | 说明 |
|---|---|
| **没有 table elimination** | MariaDB 独有，8.0 无 `opt_table_elimination.cc`。最接近的是 `pull_out_semijoin_tables`（六） |
| **未 merge 的 derived 内部 LEFT JOIN 无法被下推条件转成 INNER** | `sql_resolver.cc:829-832`：条件下推发生在 simplify_joins 之后 |
| **propagate_nullability 的副作用不可撤销** | `sql_resolver.cc:5692` 注释"conversion is not perfect"。所以 `subquery_to_derived` 直接生成 INNER JOIN 而非"先 LEFT JOIN 再撤销" |
| **derived 的 ORDER BY 多数被丢弃** | `sql_resolver.cc:3686-3690` |
| **递归 CTE 永不 merge** | 见 5.4 |

---


## 参考

**论文**
- **Moerkotte et al.《On the correct and complete enumeration of the core search space》([Moe13])** —— 连接顺序约束的理论（CD-C 算法在 `09_hypergraph.md` 中落地）

**官方文档**
- *MySQL 8.0 Reference Manual → Outer Join Simplification*
- *MySQL 8.0 Reference Manual → Optimizing Derived Tables and View References*

**内核月报**
- **2024/06《连接消除》** —— 外连接 / 内连接 / 半连接消除原理


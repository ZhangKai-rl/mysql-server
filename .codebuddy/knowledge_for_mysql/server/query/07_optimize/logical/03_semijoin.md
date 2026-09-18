# 03 Semi-join：语义、五种策略与完整执行流程

> Semi-join 是 MySQL 优化 IN/EXISTS 子查询的核心机制，独立成章。本篇综合内核月报《Semi-join优化与执行逻辑》(2021/06)、《semi-join四个执行strategy》(2020/07)、《Semijoin 丛林小道全览》(2024/06) 与 8.0.39 源码。

## 目录

- [一、Semi-join 要解决什么问题](#一semi-join-要解决什么问题)
- [二、设计思想与理论基础](#二设计思想与理论基础)
- [三、Notation：ot/ct/nt/it](#三notationotctntit)
- [四、Rewrite Phase：子查询拍平成 semi-join](#四rewrite-phase子查询拍平成-semi-join)
- [五、Optimize Phase：五种策略的代价选择](#五optimize-phase五种策略的代价选择)
- [六、Execution Phase：每种策略怎么执行](#六execution-phase每种策略怎么执行)
- [七、五种策略的 JOIN ORDER 矩阵](#七五种策略的-join-order-矩阵)
- [八、optimizer_switch 与 Hint](#八optimizer_switch-与-hint)
- [九、深潜：早停对照、NOT IN 灾难与 antijoin 演进](#九深潜早停对照not-in-灾难与-antijoin-演进)

---

## 一、Semi-join 要解决什么问题

`IN` / `EXISTS` 子查询是**存在性判断**：只关心"外表行是否在内表有匹配"，不关心匹配了几次。

- 按原始语义，每行外表都要单独执行子查询，相关子查询会被重复执行，效率很低。
- 而 SPJ 子查询的存在性判断**可以转换成外表与内表的 JOIN**。
- 但普通 JOIN 会让外表行因内表多行匹配而**重复膨胀**；`IN`/`EXISTS` 语义要求**只过滤、不膨胀**。

所以 semi-join 的核心矛盾是：**既要 JOIN（复用 join reordering 的灵活性），又要去重（不膨胀）**。五种策略就是五种"去重"方式。

```
Country.code IN (SELECT City.Country FROM City WHERE Population > 7e6)
```
Country 的一行 `code='CN'`，如果 City 里有多个城市 Population > 7e6，普通 JOIN 会输出多行 Country；semi-join 只输出一行。

---

## 二、设计思想与理论基础

### 为什么必须有五种策略？

**关键认知：semi-join 不是一个"算子"，而是"join order 上的一段区间 + 一个去重义务"。**

源码 `setup_semijoin_dups_elimination()` 的注释开篇就点明了这个模型：

```
The join order has "duplicate-generating ranges", and every range is
served by one strategy or a combination of FirstMatch with some other strategy.

"Duplicate-generating range" is defined as a range within the join order
that contains all of the inner tables of a semi-join.
```

区间的位置由 join reordering 决定，而去重能力又反过来约束 join order——正是这个**双向耦合**导致了"必须有多种策略"。

**最硬的源码证据**在 `advance_sj_state()` 里：

```
Use the strategy if
 * it is cheaper then what we've had, or
 * we haven't picked any other semi-join strategy yet
In the second case, we pick this strategy unconditionally because
comparing cost without semi-join duplicate removal with cost with
duplicate removal is not an apples-to-apples comparison.
```

> **"未去重的代价"和"已去重的代价"根本不同量纲，无法直接比较。**

所以优化器不能只挑"最便宜的算子"，而必须保证**每个前缀上至少有一个策略能让去重成立**——这就是五策略必须共存的形式化理由。整个 `advance_sj_state()` 的核心不变量是位图 `dups_producing_tables`：**"还没被任何策略消除重复的内表集合，结束时必须为 0"**。

### 分类学：3 类去重思路 × 4 个正交维度

**维度一（最本质）：去重发生在什么时刻**

| 类别 | 成员 | 机制 | 重复行是否真被生成过 |
|---|---|---|---|
| **① 提前退出（prevention）** | FirstMatch、LooseScan | 控制流层面"一组只取第一个" | 否（内表侧被短路/跳过） |
| **② 事后剔除（cure）** | DuplicateWeedout | 先按内连接跑出全部重复，再用 rowid 唯一键过滤 | **是** |
| **③ 预先做成无重复的块（normalize）** | Materialize×2 | 先物化内表并去重，之后退化成普通 join | 否（内表侧已去重） |

**维度二：依赖什么物理设施**

| 设施 | 策略 |
|---|---|
| 只靠执行控制流（零内存、零临时表） | FirstMatch |
| 必须依赖**索引** | LooseScan |
| 依赖**临时表** | DuplicateWeedout、MaterializeScan/Lookup |
| 物化表上还要能**建索引** | MaterializeLookup（Scan 不需要） |

Materialize 的两个子型就是按"用不用索引"拆出来的——源码明说 MaterializeLookup 不能用于 BLOB/GEOMETRY 列（"since indexes are not supported for BLOB columns"），而 Scan 可以。

**维度三：谁驱动（内表在前还是外表在前）**

- **外表驱动**：FirstMatch、MaterializeLookup
- **内表驱动**：LooseScan（要求"相关外表都还没进前缀"）
- **双向皆可**：MaterializeScan、DuplicateWeedout

**维度四（最精妙）：FirstMatch 与 LooseScan 的进入条件严格互补**

- FirstMatch 要求：相关外表**全部**已进前缀 —— `!(remaining_tables & outer_corr_tables)`
- LooseScan 要求：**还有**相关外表没进前缀 —— `remaining_tables_incl & sj_depends_on`

给定同一个 join 前缀，FM 与 LS 恰好覆盖两种**互斥**情形。**少了任何一个，就有一整类 join order 无法去重。**

### 各策略分别牺牲了什么

| 策略 | 收益 | 牺牲 |
|---|---|---|
| **FirstMatch** | 零临时表零内存，只改跳转指针 | 内表必须连续成块；多张内表时**禁用 join buffer**；不能跨 embedding nest 跳转 |
| **LooseScan** | 行数直接用 `rec_per_key` 塌缩（`rowcount /= rpc`），**不物化不建临时表** | 必须内表驱动（放弃所有"外表在前"的 join order）；强索引要求；IN 表达式 ≤ 64；**不能用于 antijoin** |
| **MaterializeLookup** | 内表彻底去重，后续 fanout 用 `distinct_rowcount` | 非平凡相关子查询**完全不可用**（`sj_corr_tables != 0` 直接放弃）；内表必须连续成块；类型/长度/BLOB 三道关 |
| **MaterializeScan** | 允许物化表当驱动表，外表顺序自由 | 同上，且要预付 `materialization_cost`（建表 + 写 `mat_rowcount` 行），**哪怕外表只输出 1 行也要付** |
| **DuplicateWeedout** | 对 join order **零约束** → 能把 semi-join 当普通 join 重排 | 重复行真的被生成、真的写过临时表（一次写 + 一次唯一键探查）；临时表可能落盘 |

### 理论溯源：为什么 MySQL 搞这么多种？

semi-join 在关系代数里是 R ⋉ S = π_R(R ⋈ S)，分布式数据库用"半连接缩减"减少传输量。但 **MySQL 的实现目标不同**：它要避免的是**行膨胀**（内表重复匹配导致外表行被放大）。

**与 PostgreSQL / Oracle 的关键差异，根源是历史包袱**：

MySQL 老优化器只有 nested loop 家族，五种策略全部是"在 nested loop 框架内去重"：

- **FirstMatch** = 循环短路
- **LooseScan** = 利用索引顺序（B-Tree 是 MySQL 唯一原生索引结构）
- **Weedout** = 临时表唯一键 —— **因为没有 hash 表算子，只能拿临时表当 hash set 用**
- **Materialize** = 临时表 + `<auto_distinct_key>` 唯一键，同样是"用临时表模拟 hash 去重"

对比：PostgreSQL 的 `Hash Semi Join` 用内存 hash 表，一侧 build 一侧 probe，**天然"每个 probe 行只输出一次"**——因为 hash join 是"整块"算子，**去重建在算子内部**，不需要为不同 join order 设计不同的去重机制。

MySQL 直到 8.0 才有 hash join，而在 hash join 里 semi-join 天然被支持：

```cpp
case JoinType::SEMI:
  // Semijoin should return the first matching row, and then go to the next
  // row from the probe input.
```

**这正是"五种策略"这个复杂度的历史由来**——它是一个"没有 hash join 的年代"留下的解决方案集合。

### 三级兜底

1. **策略内兜底**：DuplicateWeedout 无条件采纳（"not an apples-to-apples"）
2. **退化成内连接**：`pull_out_semijoin_tables()` 把被 eq_ref 唯一确定的内表抽出 nest；nest 被抽空则整个 semi-join 消失。hypergraph 源码直接写了这条等价律：*"If the inner side is known to be free of duplicates on the key ... semijoin is equivalent to inner join"*。注意副作用：**pullout 会把不相关子查询变成相关的，从而失去 Materialize / LooseScan 资格**
3. **转成 EXISTS**：表数超 `MAX_TABLES`(61)、子查询含 antijoin nest、或有 UNION/HAVING/聚合/LIMIT 等 → 退回 `IN→EXISTS`

### 参数

`optimizer_switch` 里 semijoin 相关的五个开关（`semijoin` / `firstmatch` / `loosescan` / `materialization` / `duplicateweedout`）**全部默认 on**。两个易错点：

- 控制 semi-join 物化的开关名是 **`materialization`**（不是 `semijoin_materialization`），它与"子查询物化"**共用同一个 flag**
- **antijoin（NOT IN / NOT EXISTS）会被强制砍掉 LooseScan**：

```cpp
if (sj_nest->is_aj_nest()) {
  // only these are possible with NOT EXISTS/IN:
  sj_enabled_strategies &= FIRSTMATCH | MATERIALIZATION | DUPSWEEDOUT;
}
```

---

## 三、Notation：ot/ct/nt/it

内核月报 2021/06 定义的记法，理解五种策略的前提：

| 记号 | 含义 |
|------|------|
| `ot` | outer table，外层表（含相关与不相关） |
| `ct` | **non-trivially correlated** outer table，子查询 WHERE 里依赖的外层表 |
| `nt` | non-correlated outer table，不相关的外层表 |
| `it` | inner table，内层（子查询）的表 |

`sj_depends_on = ot + ct`（所有被 semi-join 条件依赖的外层表），`sj_corr_tables = ct`（只含相关外层表）。这个区分在选择 join order 时至关重要。

---

## 四、Rewrite Phase：子查询拍平成 semi-join

### 3.1 resolve_subquery：收集候选（准入条件）

`sql/sql_resolver.cc:1368`。只有满足条件的 `IN/=ANY/EXISTS` 子查询才进 `sj_candidates`：

1. 子查询是 `IN`/`=ANY`/`EXISTS`，出现在 WHERE 或 ON 的**最高层**（可被 AND 包裹，OR 里不行）
2. 单一查询块，无 UNION
3. 无 HAVING、无聚合函数、无窗口函数
4. 无 LIMIT
5. 外层无 `STRAIGHT_JOIN`
6. LHS 确定（无 `RAND()`）

> 完整 16 条判定见 `resolve_subquery` 内注释（`sql_resolver.cc:1489-1567`）。不满足的走 `select_transformer`（IN2EXISTS / 物化 / MIN-MAX，见 02 篇）。

### 3.2 flatten_subqueries：bottom-up + 优先级

`sql/sql_resolver.cc:3811`。**自底向上**拍平（内层子查询先拍平，外层拍平时可能把内层一起合并）。

优先级（`sj_convert_priority`，`:3867`）：

```cpp
sj_convert_priority = ((dependent * MAX_TABLES_FOR_SIZE) + leaf_table_count) * 65536 + (65536 - subq_no);
```

从高到低三层：**相关子查询 > 内层表多 > 出现早**。因为表数上限 `MAX_TABLES`，重要的先拍平。

### 3.3 convert_subquery_to_semijoin：四种模式 + 完整流程（逐行剖析）

`sql/sql_resolver.cc:2992`。把内层表"拉进"外层 join tree。**四种模式**由两个维度决定：子查询类型（`IN` vs `EXISTS`）× 是否否定（`do_aj`）：

| 模式 | substype | `can_do_aj` | 建 nest 类型 | 表达式处理 |
|------|----------|-------------|-------------|-----------|
| **IN**（positive） | `IN_SUBS` | false | sj-nest | `build_sj_exprs` 配对 `oe[i]↔ie[i]` |
| **EXISTS**（positive） | `EXISTS_SUBS` | false | sj-nest | SELECT list 不用（`:3273-3282`） |
| **NOT IN**（negative） | `IN_SUBS` | true | aj-nest | 同 IN，但 `LEFT JOIN + IS NULL` |
| **NOT EXISTS**（negative） | `EXISTS_SUBS` | true | aj-nest | 同 EXISTS，但 `LEFT JOIN + IS NULL` |

```cpp
// sql_resolver.cc:2992
bool Query_block::convert_subquery_to_semijoin(THD *thd,
                                               Item_exists_subselect *subq_pred) {
  assert(subq_pred->substype() == Item_subselect::IN_SUBS ||
         subq_pred->substype() == Item_subselect::EXISTS_SUBS);
  table_map outer_tables_map = all_tables_map();          // ① 记录外层表位图
  const bool do_aj = subq_pred->can_do_aj;                // ② anti-join？(NOT IN/NOT EXISTS)

  // ③ 找插入点：三种情形（见下表）
  if (subq_pred->embedding_join_nest != nullptr) {
    outer_join = ...is_inner_table_of_outer_join();
    if (nest->nested_join) { emb_tbl_nest = nest; ... }             // (a) 括号 nest 内
    else if (!nest->outer_join) { emb_tbl_nest = nest->embedding; } // (b) INNER JOIN 兄弟
    else { /* (c) LEFT JOIN 内：用 wrap_nest 包裹 */ }
  }
  // else 子查询在 WHERE（emb_tbl_nest == nullptr）

  // ④ anti-join 时，先建 aj-left-nest（把 emb_join_list 所有表挪进去）
  if (do_aj) {
    wrap_nest = new_nested_join("(aj-left-nest)", ...);
    for (outer_tbl : *emb_join_list) wrap_nest->m_tables.push_back(outer_tbl); // 清空 emb_join_list
    emb_join_list->clear(); emb_join_list->push_back(wrap_nest);
    outer_join = true;
  }

  // ⑤ 建 sj-nest / aj-nest
  Table_ref *sj_nest = new_nested_join(do_aj ? "(aj-nest)" : "(sj-nest)", ...);
  emb_join_list->push_front(sj_nest);                    // push_front：aj 是 LEFT JOIN 右参数
  sj_nest->merge_underlying_tables(subq_query_block);    // ⑥ 把子查询表合并进 nest

  // ⑦ 配对表达式 / 去相关化
  if (substype == IN_SUBS) {
    build_sj_exprs(... sj_outer_exprs, sj_inner_exprs ...);   // IN：oe↔ie 配对
  } else { /* EXISTS：SELECT list 清理掉 */ }
  decorrelate_condition(sj_decor, nullptr);              // ⑧ 内层 WHERE 等值谓词提升
  build_sj_cond(thd, nested_join, ..., &sj_cond);        // ⑨ 逐对 AND 成 oe=ie

  // ⑩ anti-join 收尾：sj-nest 变 LEFT JOIN 右表，ON = sj_cond
  if (do_aj) {
    sj_nest->outer_join = true;
    sj_nest->set_join_cond(sj_cond);                     // x IS NULL 留给优化阶段再加
    this->outer_join |= sj_nest->nested_join->used_tables;
  }
}
```

**③ 三种插入位置**（`:3020-3114`）：

| 位置 | 处理 | emb_join_list 指向 |
|------|------|-------------------|
| WHERE 里 | sj-nest 直接进 FROM | 顶层 `m_table_nest` |
| `INNER JOIN tblX ON (subquery AND cond)` | sj-nest 成为 tblX 兄弟 | tblX 的 embedding 的 join list |
| `LEFT JOIN tbl ON (subquery AND cond)` | `wrap_nest` 包住 `tbl SJ (subq)`，outer join 标志上移 | wrap_nest 的 m_tables |

LEFT JOIN 内层最麻烦（`:3046-3113`）：直接插表会破坏 outer join 的可空语义，必须用 `wrap_nest` 把 `tbl SJ (subq_tables)` 整体包住，并把 `outer_join` 标志和 join cond 从 `tbl` 上移到 `wrap_nest`（`:3083-3087`）。

**④ anti-join 的 aj-left-nest**（`:3165-3179`）：anti-join 建模成 `(左表) LEFT JOIN (aj-nest) ON subq_cond`，所以先把 emb_join_list 里所有现有表挪进一个 `(aj-left-nest)`，清空原 join list，只留 aj-left-nest，再 `push_front` aj-nest 作为 LEFT JOIN 右参数。**`x IS NULL` 条件不在这一步加**——注释（`:3161-3163`）明确说留到优化阶段，因为那时才有 QEP_TAB 能设 trig cond 的 `found`/`not_null_compl` 指针。

**关键：`push_front` 而不是 `push_back`**（`:3197`）——anti-join 时 sj-nest 是 LEFT JOIN 的右参数，而 LEFT JOIN 的右参数必须排在左参数之前（join tree 里右表先）。

**⑩ anti-join 收尾**：`set_join_cond(sj_cond)` 把 semi-join 条件挂成 LEFT JOIN 的 ON，配合优化阶段的 `x IS NULL` 完成 `NOT IN/NOT EXISTS` 语义。

### 3.4 五种 SQL 等价变换形式（内核月报 2020/10 原文）

`convert_subquery_to_semijoin` 的注释列了完整变换：

```
1. IN/=ANY:
   FROM ot1...otN WHERE (oe1..oeM) IN (SELECT ie1..ieM FROM it1..itK [WHERE inner-cond])
   => FROM (ot1..otN) SJ (it1..itK) ON (oe1..oeM)=(ie1..ieM) [AND inner-cond]

2. EXISTS:
   WHERE EXISTS(SELECT ... FROM it1..itK [WHERE inner-cond])
   => FROM (ot) SJ (it) [ON inner-cond]

3. NOT EXISTS:
   => FROM (ot) AJ (it) [ON inner-cond] WHERE ... AND is-null-cond(it1)

4. NOT IN:
   => FROM (ot) AJ (it) ON (oe)=(ie) [AND inner-cond]
```

anti-join（AJ）建模成 **LEFT JOIN + 右表键 IS NULL**。

### 3.5 去相关化 + 建 sj 条件

- `build_sj_exprs`（`:2833`）：IN 的左右两边配对成 `sj_outer_exprs[i] ↔ sj_inner_exprs[i]`
- `decorrelate_condition`（`:3284`）：把内层 WHERE 里的 `it.c = ot.c` 提升成 sj 连接条件
- `build_sj_cond`（`:2481`）：逐对 AND 成 `oe[i]=ie[i]`，常量对折叠
- `simplify_joins`：去掉嵌套 sj-nest（`A SJ (B SJ (C))` → `A SJ (B JOIN C)`）

---

## 五、Optimize Phase：五种策略的代价选择

### 4.1 make_join_plan 的前置

```
pull_out_semijoin_tables            把 EQ_REF 的 it 抽出 sj-nest（1:1 不膨胀）
set_semijoin_embedding              设每个 join_tab->emb_sj_nest
update_semijoin_strategies          设 sj_enabled_strategies
optimize_semijoin_nests_for_materialization  预优化可物化的 sj-nest 内表 join order
```

`pull_out_semijoin_tables` 的关键：EQ_REF 保证"能 join 到且只能 join 到一条"，对存在性语义无贡献，抽出去后可缩小去重范围。

### 4.2 advance_sj_state：贪心搜索中的策略状态机

`sql/sql_planner.cc:4105-4599`。在 `best_extension_by_limited_search` 每加一张表后调用，为当前 join 前缀判定/更新五种策略。核心不变量是 `dups_producing_tables`（还没被任何策略消除的 sj 内层表位图，结束时必须为 0）。

每种策略在 `POSITION` 上有一组状态变量：

| 策略 | 状态变量 |
|------|---------|
| FirstMatch | `first_firstmatch_table`、`firstmatch_need_tables` |
| LooseScan | `first_loosescan_table`、`loosescan_need_tables` |
| Materialize | `sjm`（Semijoin_mat_optimize） |
| DuplicateWeedout | `first_dupsweedout_table`、`dupsweedout_tables` |

### 4.3 五种策略详解

#### FirstMatch（首匹配）

**原理**：外表行与内表 JOIN 得到一条输出后，**直接跳过所有剩余内表行**，读下一行外表。

**适用条件**：相关外层表（`sj_depends_on`）都在前缀；内表连续；当前表是最后一个内表；`dups_producing_tables == 0`。

```cpp
if (pos->dups_producing_tables == 0 && !(remaining_tables & outer_corr_tables)) {
  pos->first_firstmatch_table = idx;   // 开始跟踪 FM 范围
}
```

#### LooseScan（松散扫描）

**原理**：内表驱动表用索引做"松散扫描"（只扫描索引键不同的记录，一个 group 只取首行），天然去重。

**适用条件**（与 FirstMatch **相反**）：相关外层表**不在**前缀；内表驱动表有索引（`keyuse() != nullptr`）；`sj_inner_exprs ≤ 64`。

```cpp
if (remaining_tables_incl & emb_sj_nest->nested_join->sj_depends_on &&  // 还有相关外表在后面
    new_join_tab->keyuse() != nullptr) {                                // 有索引
  pos->first_loosescan_table = idx;
}
```

#### MaterializeLookup / MaterializeScan（物化）

**原理**：内表物化到去重临时表，外层与临时表 JOIN。根据临时表是扫描还是索引查找分两种。

**适用条件**：内表作为连续块全部进前缀；类型可物化（无 BLOB、key 长度不超限）。

```cpp
if ((remaining_tables & emb_sj_nest->nested_join->sj_depends_on) || !lookup_allowed) {
  if (scan_allowed) return SJ_OPT_MATERIALIZE_SCAN;   // 还有相关外表 → 只能扫
}
return SJ_OPT_MATERIALIZE_LOOKUP;                      // 否则索引查找
```

#### DuplicateWeedout（重复剔除）

**原理**：兜底策略。把半连接当普通内连接执行，但用临时表按**外层表 rowid** 去重。对 join order 几乎无约束。

**适用条件**：最通用，几乎任意 join order。选择条件里 `pos->dups_producing_tables`（还有未消除重复）时**无条件采纳**（因为"未去重成本"和"已去重成本"不是同量纲）。

### 4.4 fix_semijoin_strategies：确定最终策略

贪心搜索每加一张表判定一次，前后可能选不同策略。`fix_semijoin_strategies` 从后往前遍历（靠后的记录更大前缀的最优策略），把最终策略记到第一个内表上（`n_sj_tables` + `sj_strategy`）。

---

## 六、Execution Phase：每种策略怎么执行

> 本篇从**优化器视角**讲"选定策略后如何落到 QEP/迭代器"。**运行期的执行机制**（物化引擎 `subselect_hash_sj_engine`、IN2EXISTS 的 `Item_in_optimizer`、子查询缓存）详见 [`../../runtime/02_subquery_runtime.md`](../../runtime/02_subquery_runtime.md)。

`setup_semijoin_dups_elimination`（`sql/sql_select.cc`）创建具体执行结构。每种策略产生特定的 QEP_TAB 序列（内核月报 2021/06 的图）：

### 5.1 FirstMatch：split jump

```
(ot|nt)*  [ it ((it|nt)* it) ]  (nt)*
```

内表 range 内可以有 nt 表。执行时用 "split jump"：内层找到匹配后 `firstmatch_return` 跳回 range 起始的外表下一行，保证内表不重复行：

```
ot -> [it1 -> nt1 -> it3]
1  ->  1   ->  1    ->  1    <- jump（回 ot 读下一行）
2  ->  1   ->  1
2  ->  2   ->  1    ->  1    <- jump
```

`NestedLoopIterator` 的 `JoinType::SEMI`：找到一行匹配后状态切回 `NEEDS_OUTER_ROW`，实现短路。

### 5.2 LooseScan：loosescan_buf

```
(ot|ct|nt) [ loosescan_tbl (ot|nt|it)* it ]  (ot|nt)*
```

所有相关外表（ct）必须在内表前（loosescan 对内表去重，相关外表会决定内表内容，必须先去重）。执行：

- `loosescan_tbl->loosescan_buf` / `loosescan_key_len`：保存 loose scan keypart 的 buffer + 长度
- 读内表一行 → `key_cmp` 比较索引 key，相同则跳过（skip 重复 index entry）
- 后续 range 用 FirstMatch 机制

### 5.3 DuplicateWeedout：rowid 临时表

```
(ot|nt)*  [ it ((it|ot|nt)* (it|ot))]  (nt)*
```

执行（`do_sj_dups_weedout`，`sql_executor.cc:3386`）：

1. 收集参与去重的各表 `position()` 得到的 rowid
2. 拼接成一个 tuple 写入临时表
3. **key 长度分界**（内核月报 2020/07 细节）：`jt_rowid_offset + null_bytes > 512` → 建 `Field_longlong` 的 hash_field；`≤ 512` → 建 `Field_varstring` 的 rowids key
4. `ha_write_row` 写入，重复则丢弃该行

**join buffer 的影响**（内核月报 2021/06 强调）：range 内没用 join buffer 时，可用 `have_confluent_row` 标记简化（prefix 每次来的是新行才处理）；用了 join buffer 则无法保证，只能把整个 range 的所有 rowid 都作为 distinct key。

### 5.4 Materialize：物化钩子

物化策略的 inner table 不在 primary tables 中（只有 sjm table）。执行时 sjm 对应的 QEP_TAB 开始执行时，先 `prepare_scan` → `join_materialize_semijoin` 完成内表物化 + 去重，后续无需特殊处理：

- **MaterializeLookup**：`outer table → temporary table`（外层每行 ref 查找）
- **MaterializeScan**：`temporary table → outer table`（物化表全扫）

---

## 七、五种策略的 JOIN ORDER 矩阵

内核月报 2024/06《Semijoin 丛林小道全览》总结（用 `ct1,ct2` 外表 + `it1,it2` 内表，`WHERE ct1.a IN (SELECT it1.a FROM it1,it2 WHERE it2.b=ct2.b)`）：

| 策略 | JOIN ORDER 支持 |
|------|----------------|
| **DuplicateWeedout** | 最通用，几乎任意 join order |
| **MaterializeScan** | 内表整体物化后，外表可任意调整，物化表可在外表前/后 |
| **MaterializeLookup** | 所有 sj 依赖外表必须先于内表 |
| **FirstMatch** | 依赖外表在内表前，内表连续，不穿插外表 |
| **LooseScan** | 要求 loose scan 内表后至少一个关联外表，内表连续，索引列覆盖后续 sj 条件 |

结论：DuplicateWeedout 最通用，其他策略只适用特定 join order，**组合起来才能给优化器足够的 join order 灵活性**。

---

## 八、optimizer_switch 与 Hint

`optimizer_switch` 里 semi-join 相关开关（内核月报 2020/07）：

| 参数 | 作用 |
|------|------|
| `semijoin` | semi-join 总开关 |
| `materialization` | 物化策略 |
| `firstmatch` | FirstMatch |
| `loosescan` | LooseScan |
| `duplicateweedout` | DuplicateWeedout |

Optimizer Hint 指定策略：

```sql
SELECT /*+ SEMIJOIN(DUPSWEEDOUT) */ ... WHERE s_suppkey IN (SELECT ps_suppkey ...)
```

---


## 九、深潜：早停对照、NOT IN 灾难与 antijoin 演进

### 9.1 早停机制的跨算法对照

MySQL 的 semi/anti join 早停（"找到一个匹配就跳下一个外层行"）**散落在执行器各处**，对照如下：

| 执行路径 | 早停实现 | 落点 |
|---|---|---|
| **Nested Loop**（FirstMatch） | `JoinType::SEMI` 状态机：找到匹配后切回 `NEEDS_OUTER_ROW`（split jump） | 本篇 5.1 |
| **Hash Join** | `JoinType::SEMI`：probe 到第一行匹配即返回，去下一行 probe | `05_join_buffer` / hash_join_iterator |
| **LooseScan** | 索引 group 跳跃（天然跳过重复，不用"找到再跳"） | 本篇 5.2 |
| **DuplicateWeedout** | **不是早停**——先按内连接跑全，再 rowid 去重 | 本篇 5.3 |

**与 PG 的对照（为什么两边长得不一样）**：

PG 侧（《PostgreSQL 子查询优化（七）》的核心结论）：NL/Hash/Merge 三种算法的早停**源码几乎完全相同**（`nl_NeedNewOuter` / `HJ_NEED_NEW_OUTER` / `EXEC_MJ_NEXTOUTER`，仅状态变量名不同），是**刻意的一致性设计**——因为 PG 的 SEMI/ANTI 是**一种显式 join 节点**（`JOIN_SEMI`/`JOIN_ANTI`），早停自然收拢在 join 节点层。

MySQL 侧恰好相反：**没有统一的 semi/anti join 节点**，早停是**各策略各自实现**的（NL 状态机 / hash 的 probe 短路 / LooseScan 的索引跳跃）。这正是"五种策略"架构的直接后果——**策略分散，早停也分散**。

另外 MySQL **没有 Merge Join**（`AccessPath` 枚举只有 `NESTED_LOOP_JOIN`/`BKA_JOIN`/`HASH_JOIN`），所以"三算法对照"在 MySQL 只剩"两算法 + 索引跳跃"。

### 9.2 NOT IN 的灾难：为什么永远优先 NOT EXISTS

NULL **语义**部分见 [`02_subquery.md`](02_subquery.md)（`abort_on_null`、null_problem 导致 NOT IN 拿不到 antijoin）。这里讲**性能**。

| 写法 | PG | MySQL | 复杂度 |
|---|---|---|---|
| `NOT EXISTS` | ANTI JOIN（显式节点） | antijoin（FirstMatch/Weedout 等） | O(LHS + RHS) |
| `NOT IN`（无 NULL 保护） | **SubPlan**：物化 RHS + 每行全表扫 | **物化子查询**：物化 RHS + 每行探查 | **O(LHS × RHS)** |
| `LEFT JOIN ... IS NULL` | 自动转 ANTI JOIN（`reduce_outer_joins`） | aj-nest 收尾就是 `LEFT JOIN + IS NULL`（本篇 3.3） | O(LHS + RHS) |

两边结论一致：**NOT IN 在含 NULL 时是灾难，永远优先 NOT EXISTS**。区别只在"灾难路径"的名字：PG 叫 SubPlan，MySQL 叫物化子查询（见 [`../../runtime/08_materialization.md`](../../runtime/08_materialization.md)）。

### 9.3 antijoin 的版本演进

- **MySQL 8.0.17 起**支持 antijoin 优化（NOT IN / NOT EXISTS → 反连接）。此前只能走物化子查询（每行探查）。⚠️ 版本号来自官方 Release Notes，**源码无版本痕迹**（与 CTE 演进同理）
- antijoin 是 **semi-join 框架的"负模式"**（aj-nest，见本篇 3.3 的 NOT IN 表格），不是独立的 join 节点
- **策略受限**：antijoin 会强制砍掉 LooseScan（`is_aj_nest()` 只允许 FirstMatch / Materialization / DupsWeedout，见本篇参数节）
- **NULL 保护的必要性**：NOT IN 走 antijoin 的前提是子查询结果不含 NULL 或列不可空，否则退回物化（02 篇的 null_problem 分析）

对比 PG：PG 的 ANTI JOIN 是显式 join 节点（`JOIN_ANTI`），MySQL 是半连接框架里的负模式——又一个"节点 vs 策略"的架构差异。

### 9.4 节点 vs 策略：与 PG 的架构全面对照

#### 先纠正一个常见误解：PG 的 SEMI JOIN 不是"只有一种实现"

PG 的 `JOIN_SEMI` 是**逻辑节点**，节点之下有三种物理实现竞争——**Hash Semi Join / Merge Semi Join / Nested Loop Semi Join**（回归测试的 EXPLAIN 里就能看到这三种名字）。优化器为同一节点生成多条 path，代价选择。

所以"PG 只有一种、MySQL 有五种"是对比**错了维度**：

| 层 | PG | MySQL（旧优化器） | MySQL（hypergraph） |
|---|---|---|---|
| 逻辑层 | `JOIN_SEMI`/`JOIN_ANTI` 节点 | semi-join 框架（aj-nest 负模式） | `RelationalExpression::SEMIJOIN/ANTIJOIN` |
| **算法维度** | Hash / Merge / NL **三算法竞争** | NL + hash（8.0.18 起，**无 Merge**） | Hash / NL 竞争（`ProposeHashJoin` / `ProposeNestedLoopJoin`） |
| **去重维度** | 节点内置（找到即跳，写一处） | **五策略**（FM/LS/Weedout/Mat×2） | `DeduplicateForSemijoin`（LIMIT 1 / REMOVE_DUPLICATES） |
| 执行器 | 节点直接执行 | `JoinType::SEMI/ANTI`（NL/Hash 迭代器分发） | 同左 |
| 开关 | `enable_nestloop/hashjoin/mergejoin`（**逐算法**） | `semijoin/firstmatch/loosescan/...`（**逐策略**）；`hash_join` 开关**已失效** | 仅 `hypergraph_optimizer` 总开关 |

#### 源码揭示的三个真相

**① 执行器层 MySQL 其实有显式 SEMI/ANTI**

`sql/join_type.h`：

```cpp
enum class JoinType { INNER, OUTER, ANTI, SEMI, FULL_OUTER };
```

`HashJoinIterator` 与 `NestedLoopIterator` 都按它分发行为（SEMI 找到即跳、ANTI 见行即弃）。所以"MySQL 无显式节点"**只对旧优化器层成立**——执行器层与 PG 的节点模型对得上。

**② hash semi join 不是第六种策略，与五策略正交**

`advance_sj_state` 全文**没有任何 hash join 分支**；`SJ_OPT_*` 枚举只有 6 值（含 NONE，无 HASH）。hash semi join 发生在**策略定稿之后**的执行准备阶段：

```
五策略（advance_sj_state → fix_semijoin_strategies）：决定"去重方式"
      ↓ 定稿
ConnectJoins() → CreateHashJoinAccessPath()：决定"连接算法"（NL 或 hash）
```

即：**五策略管"怎么去重"，hash join 管"怎么连接"**——两个正交维度。这也解释了为什么 `semijoin` 策略开关与 hash join 互不干扰。

**③ hypergraph 优化器完全不读五策略**

`sj_strategy`、`SJ_OPT_*`、五策略开关在 `sql/join_optimizer/` **零引用**。hypergraph 把 semi join 表达为 `RelationalExpression::SEMIJOIN`（与 `JoinType::SEMI` **同值**），代价模型同时提议 Hash/NL 两种候选，去重用 `DeduplicateForSemijoin`。

**所以 MySQL 的新优化器其实已经走向 PG 式的"节点 + 算法竞争"模型**；五策略是旧优化器专属遗产。

#### "谁更优"的准确回答

**不是"策略多 = 更优"**：

1. **MySQL 五策略是"NL-only 年代"的去重补丁**——因为 8.0.18 前没有 hash join，只能在 NL 框架内用各种 trick 做去重（每种策略都有严格前提：LooseScan 要索引、Materialize 要临时表、Weedout 零约束但重复行真的产生）。它的"丰富"恰恰暴露了当年的短板。
2. **PG 的 Hash Semi Join 在多数场景下不逊于、甚至优于 MySQL 五策略**——O(1) 探测、可并行、去重建在算子内。
3. **但 MySQL 确有 PG 没有的独特维度**：LooseScan 的索引跳跃（PG 的 Merge Semi Join 需要两个有序输入，LooseScan 只要一个索引）、FirstMatch 的零内存、以及**逐策略开关**的 DBA 可调性。
4. **准确的表述**：两者在"物理实现多样性"上其实相当（PG 3 算法 × 1 去重 vs MySQL 2 算法 × 6 去重策略），但**多样性的维度不同**——PG 在算法维度（源自设计），MySQL 在去重维度（源自历史）。

#### 三层三套抽象（旧优化器路径的代价）

同一句"t1 SEMI JOIN t2"在旧优化器路径是**三个不互通的类型系统**：

```
优化器层：SJ_OPT_LOOSE_SCAN 等策略状态机（附着在 join 顺序搜索上）
   ↓ 执行准备期二次解码（ConnectJoins / FindSubstructure）
AccessPath 层：NESTED_LOOP_SEMIJOIN_WITH_DUPLICATE_REMOVAL 等节点
   ↓
迭代器层：JoinType::SEMI / NestedLoopSemiJoinWithDuplicateRemovalIterator
```

而 hypergraph 路径消除了"策略→节点"的翻译（`RelationalExpression` 与 `JoinType` **共享枚举值**）。PG 则从头到尾只有一棵 plan node 树——**这是 PG 架构统一性的真正优势**：语义在逻辑层定一次，物理层只竞争算法。

### 9.5 已知正确性缺陷：EXISTS 转 semi-join（Bug #110819）

这是本目录记录的**第一个正确性 bug**（而非性能问题），值得单独记住。

#### 现象

数据：3 行，全部是 `("a", 1)`，id 为 1/2/3：

```sql
SELECT id FROM bug_t t1 WHERE EXISTS (
  SELECT 1 FROM bug_t t2
  WHERE t1.id > t2.id                  -- 相关、非等值
    AND t1.col_str = t2.col_str        -- 等值
    AND t1.col_int = t2.col_int);
-- 期望 {2, 3}；实际只返回 {2}
```

#### 关键事实

| 项 | 值 |
|---|---|
| Bug 号 | **#110819** |
| 标题 | EXISTS with dependent subquery can return incorrect results |
| 严重性 | **S2 (Serious)** |
| 状态 | **Verified**（2023-04 提交） |
| 影响版本 | **8.0.16+**（8.0.15 及以下不复现） |
| 标签 | **regression**（回归） |
| 规避方法 | `SET optimizer_switch='semijoin=off';` |

#### 根因：WL#4389「Transform EXISTS subqueries to semi-join」

官方 bisect 定位到首个坏提交（Roy Lyseng，2018-11）：

> **WL#4389** Transform EXISTS subqueries to semi-join — *Extend semi-join check to accept EXISTS subqueries in addition to IN. Filter out non-deterministic subqueries.*

即：**8.0.16 把 semi-join 转换的适用范围从 `IN` 扩展到了 `EXISTS`**——这正是本篇"四种 SQL 等价变换形式"里 EXISTS 那一条的由来。这个扩展引入了正确性回归。

#### Bug #28805105 **不是**成因（源码已推翻报告者的推断）

8.0.16 changelog 里那条：

> ...but because the subquery is not correlated... we use **two equal constant items as keys**, to ensure that the materialized query gets the constant as a key (and so that the materialized table consists of at most one row). (Bug #28805105)

报告者怀疑它导致本 bug。**源码证明不成立**，两道独立排除：

**排除一：补丁根本不触发**。它在 `Query_block::build_sj_cond()` 里，唯一触发条件是 `sj_inner_exprs` 为空：

```cpp
if (nested_join->sj_inner_exprs.empty()) {
  Item *const_item = new Item_int(1);
  nested_join->sj_inner_exprs.push_back(const_item);   // "两个相等常量"= 同一个 Item_int(1) 塞两边
  nested_join->sj_outer_exprs.push_back(const_item);
}
```

本例两条等值被去相关塞进 `sj_inner_exprs = (t2.col_str, t2.col_int)`，**非空** ⇒ 不触发。

**排除二：物化被独立关掉**。即使触发，`sql/sql_optimizer.cc` 的 `optimize_semijoin_nests_for_materialization()` 有：

```cpp
if (sj_nest->nested_join->sj_corr_tables) continue;   // 相关 semi-join 不做物化
```

本例 `sj_corr_tables = {t1}` ≠ 0 ⇒ 物化整条路径出局。

**最硬的证据**：`JOIN::setup_semijoin_materialized_table()` 用 `sj_inner_exprs` 建物化表，本例列只有 `(col_str, col_int)` —— **物化表里根本没有 `t2.id`**，`t1.id > t2.id` 在其上无法求值。所以物化不可能产生 `{2}`（若忽略残差得 `{1,2,3}`；若 t2.id 固定为 X 得 `{X+1..3}`，无解为 `{2}`）。

#### 真正的机制：LooseScan 的"命中即跳组"

转换后三个条件命运不同（关键）：

| 条件 | 处理 |
|---|---|
| `t1.col_str = t2.col_str` | **去相关** → 进 `sj_inner_exprs`（成为 key） |
| `t1.col_int = t2.col_int` | **去相关** → 进 `sj_inner_exprs`（成为 key） |
| `t1.id > t2.id` | **不去相关**：`can_decorrelate_operator()` 对 `GT_FUNC` 返回 false（semi-join 路径 `op_types == nullptr` 只去相关 `=`）→ **残留**在 WHERE |

LooseScan 逐行推演（`col_str` 索引跳组）：

1. `'a'` 只有**一组**；组内按二级索引 `(col_str, PK)` 序为 t2.id = 1, 2, 3
2. LooseScan **每组只取第一行** → **t2.id = 1**
3. `t1.id=2`：`2 > 1` ✓ → 输出 **2**，随即**跳过整组**（`tab->match_tab = last_sj_tab->idx()`）
4. ⇒ **t2.id=2、t2.id=3 永远不会被取到**，`t1.id=3` 唯一能成立的路径（需 `3 > 2`）从未被探测 → **丢失**

> 本质：LooseScan 把作用在**组内非分组列 `t2.id`** 上的相关非等值条件，误当成"只依赖分组键 `col_str`"的条件。

#### 一处精妙的护栏（也是未完全确定的点）

LooseScan 准入条件 (5) 与 (6) 对"t1 是否在剩余表"的要求**互斥**：

```cpp
!(remaining_tables_incl & sj_corr_tables) &&   // (5) 要求 t1 已进前缀
(remaining_tables_incl  & sj_depends_on)       // (6) 要求 t1 仍在剩余
```

本例 `sj_corr_tables == sj_depends_on == {t1}` ⇒ 两者不可能同时成立 ⇒ **LooseScan 应被排除**。护栏意图见源码注释：*"All non-IN-equality correlation references from this sj-nest are bound"*。

于是**静态推演的结果是 8.0.39 应返回正确的 `{2,3}`**，与 bug 报告（8.0.16+ 复现）**矛盾**。矛盾焦点收敛到一个运行期量：`sj_corr_tables = sj_cond->used_tables() & outer_tables_map`，它受 `Item_cond::used_tables_cache` 是否刷新影响——**此层无法仅凭读源码定死，不做断言**。

**一锤定音的方法**：`EXPLAIN` + `SET optimizer_trace='enabled=on'`，看 `semijoin_strategy_choice` 与 `final_semijoin_strategy`：
- 显示 **LooseScan** → 坐实上述机制（护栏失效，`sj_corr_tables` 实为 0）
- 显示 FirstMatch / DuplicateWeedout 且结果仍错 → 问题在别处

> 另：FirstMatch（仅 t1→t2 顺序可用）与 DuplicateWeedout 的逐行推演均为正确的 `{2,3}`。

#### 为什么值得记

1. **semi-join 是"改写"，不是"等价保证"**——它是一个**有正确性边界的优化**。加了 `semijoin` 转换，就可能引入结果错误（不同于代价估算错误只影响性能）。
2. **EXISTS 的转换比 IN 晚、也更危险**：IN→semi-join 从 5.6 就有，EXISTS→semi-join 是 **8.0.16 才引入**（WL#4389），所以它是"新且未经充分验证"的那一块。
3. **排查信号**：遇到 `EXISTS` + 相关子查询 + 结果行数偏少，优先试 `semijoin=off` 验证；若规避后正确，即命中此 bug。

> ⚠️ 本仓库为 8.0.39，该 bug 报告的影响范围标注为 "8.0.16+, 8.0.33"，且报告未显示已修复版本——**8.0.39 很可能仍受影响**，实际遇到时请以复现为准。

---

## 参考

**论文**
- **Galindo-Legaria《Parameterized Queries and Nesting Equivalences》(2001)** —— semi-join 转换的理论基础

**官方文档**
- *MySQL 8.0 Reference Manual → Semi-join Transformations*

**内核月报**
- **2024/06《Semijoin 丛林小道全览》** —— 五种策略与 JOIN ORDER 矩阵
- **2021/06《Semi-join 优化与执行逻辑》**
- **2020/07《semi-join 四个执行 strategy》** —— 四种策略（DuplicateWeedout / FirstMatch / LooseScan / Materialize）与 PolarDB 并行加速

**阿里云 PolarDB 官方文档**
- [《图解 MySQL 8.0 优化器对子查询 JOIN 与分区表的转换优化》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/optimizer-based-query-conversion-in-mysql-8) —— `convert_subquery_to_semijoin` 四种模式（IN/EXISTS/NOT EXISTS/NOT IN）、`flatten_subqueries` 优先级、anti-join 的 `can_do_aj`（本篇 3.1/3.3 对齐此文）

**已知缺陷**
- [MySQL Bug #110819](https://bugs.mysql.com/bug.php?id=110819) —— *EXISTS with dependent subquery can return incorrect results*（8.0.16+ 回归，WL#4389 引入，S2，见 9.5）


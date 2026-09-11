# 03 Semi-join：语义、五种策略与完整执行流程

> Semi-join 是 MySQL 优化 IN/EXISTS 子查询的核心机制，独立成章。本篇综合内核月报《Semi-join优化与执行逻辑》(2021/06)、《semi-join四个执行strategy》(2020/07)、《Semijoin 丛林小道全览》(2024/06) 与 8.0.39 源码。

## 目录

- [一、Semi-join 要解决什么问题](#一semi-join-要解决什么问题)
- [二、Notation：ot/ct/nt/it](#二notationotctntit)
- [三、Rewrite Phase：子查询拍平成 semi-join](#三rewrite-phase子查询拍平成-semi-join)
- [四、Optimize Phase：五种策略的代价选择](#四optimize-phase五种策略的代价选择)
- [五、Execution Phase：每种策略怎么执行](#五execution-phase每种策略怎么执行)
- [六、五种策略的 JOIN ORDER 矩阵](#六五种策略的-join-order-矩阵)
- [七、optimizer_switch 与 Hint](#七optimizer_switch-与-hint)

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

## 二、Notation：ot/ct/nt/it

内核月报 2021/06 定义的记法，理解五种策略的前提：

| 记号 | 含义 |
|------|------|
| `ot` | outer table，外层表（含相关与不相关） |
| `ct` | **non-trivially correlated** outer table，子查询 WHERE 里依赖的外层表 |
| `nt` | non-correlated outer table，不相关的外层表 |
| `it` | inner table，内层（子查询）的表 |

`sj_depends_on = ot + ct`（所有被 semi-join 条件依赖的外层表），`sj_corr_tables = ct`（只含相关外层表）。这个区分在选择 join order 时至关重要。

---

## 三、Rewrite Phase：子查询拍平成 semi-join

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

## 四、Optimize Phase：五种策略的代价选择

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

## 五、Execution Phase：每种策略怎么执行

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

## 六、五种策略的 JOIN ORDER 矩阵

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

## 七、optimizer_switch 与 Hint

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


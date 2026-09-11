# 06 prepare / resolve：名字解析、类型绑定与逻辑改写

> 本篇覆盖：逻辑查询树如何从"语法正确但语义未定"变成"语义完整、可优化"（第 ②→③ 层转换）。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、prepare 的两层入口](#一prepare-的两层入口)
- [二、Query_block::prepare 的 46 步](#二query_blockprepare-的-46-步)
- [三、setup_* 函数族](#三setup_-函数族)
- [四、fix_fields：一棵树绑定出语义](#四fix_fields一棵树绑定出语义)
- [五、name resolution 规则](#五name-resolution-规则)
- [六、子查询改写与 semi-join](#六子查询改写与-semi-join)
- [七、derived table：merge 还是 materialize](#七derived-tablemerge-还是-materialize)
- [八、窗口函数 setup](#八窗口函数-setup)
- [九、prepare 之后才存在的东西](#九prepare-之后才存在的东西)
- [核心调用栈](#核心调用栈)

---

## 设计思想与理论基础

### name resolution：编译器符号表的数据库版

prepare 的核心之一是 **name resolution（名字解析）**——把 SQL 里的裸标识符（`a`）解析成具体的实体（`t` 表的第几列、`Field*` 指针）。这正是编译器里**符号表 + 作用域**在数据库语境下的实现：

| 编译器概念 | MySQL 对应 |
|---|---|
| 符号表 | `TABLE_SHARE` 的列定义（`Field*` 数组） |
| 作用域 | 子查询嵌套的可见性规则（内层能看外层，外层不能看内层） |
| 绑定 | `Item_field::fix_fields` 把 `Item_field` 的 `field` 指针填上 |
| 类型检查 | 比较运算两边的类型兼容、collation 推导、nullability |

`fix_fields` 就是对 Item 树的**自底向上求值**：先修叶子（字段、常量），再修内节点（运算符、函数），逐层把 `fixed` 标志置真——和编译器的属性文法求值同构。

### "同一棵树从松到紧"：resolve 不是重建，是补全

prepare 的一个关键设计是（整体印象里的表格）：**它不新建一棵树**，而是把 contextualize 产出的同一棵 Item 树从"未绑定"变成"已绑定"。这决定了：

- 内存模型简单：一棵树从 ② 层走到 ③ 层，对象不迁移
- 但代价是**每个对象都要能表达"未解析"状态**（`field == nullptr`、`fixed == false`），代码里到处是"先判 fixed 再求值"的防御

这个选择的反面是"解析即重建"（如 PostgreSQL 的 parse analysis 会产出新的 Query 树）。MySQL 选择原地补全，是历史与内存模型（mem_root 一次性回收）共同决定的。

### 永久变换 vs 临时变换：prepare 与 optimize 的分野

prepare 里做的改写（semi-join、derived merge、常量消除）是**永久的**——`assert(first_execution)`，Prepared Statement 只做一次。而 `optimize_cond` 类的谓词优化是**临时的**，每次 execute 重做。这个分野的动机是：

- 永久变换改变的是**逻辑结构**（树长什么样），与具体数据无关，做一次就够
- 临时变换依赖**统计信息/常量值**，可能每次执行都不同

它解释了为什么"逻辑优化"横跨 prepare 和 optimize 两个阶段（07_optimize/00_overview 详述）——**先规则（永久）、后代价（临时）**。

> **阶段命名的澄清**：`resolver` 和 `prepare` 是**同一个阶段**——入口都是 `Query_block::prepare`（`sql_resolver.cc:179`）。"resolver"指它内部的 `setup_*` + `fix_fields` 子步骤，"prepare"是入口函数名、范围更广（还含 transform）。而 **transform（semi-join flatten / derived merge / simplify_joins / 常量消除）是 prepare 末尾的永久子步骤**，精确定位全在 `Query_block::prepare` 的调用栈上：`flatten_subqueries`(:549)、`apply_local_transforms`(:593，内含 `simplify_joins` :768)、`merge_derived`(:1309)。`JOIN::optimize`（`sql_optimizer.cc:338`）里**找不到这些结构 transform 的调用**——它只做 `optimize_cond`（临时）+ `make_join_plan` + `create_access_paths`（规划，不改逻辑树结构）。

---

## 先有个整体印象

contextualize（05 篇）结束后，我们有了一棵"语法正确"的树：`Item_field(a)` 知道列名叫 `a`，但**不知道 `a` 是哪张表的哪一列**；`Table_ref(t)` 知道有个表叫 `t`，但**还没打开**。

prepare（也叫 resolve，`Query_block::prepare`）就是补上这些"语义空白"。官方注释（`sql_resolver.cc:130-145`）把它归成四条主线：

```
1. Resolve table and column information.        名字解析：列名 → Field*，表名 → TABLE*
2. Resolve all expressions (item trees).         类型绑定：used_tables、类型、collation、nullability
3. Prepare all subqueries recursively.           子查询递归 prepare
4. Apply permanent transformations.              不可逆改写：semi-join、derived merge、常量消除
```

**一个最直观的例子**：prepare 前后，同一个 `Item_field` 对象的变化：

| | prepare 前 | prepare 后 |
|---|-----------|-----------|
| `item_name` | `"a"` | `"a"`（不变） |
| `field` | **`nullptr`** | 指向 `t` 表的真实 `Field*` |
| `table_ref` | `nullptr` | 指向 `Table_ref(t)` |
| `fixed` | `false` | **`true`** |
| `used_tables()` | 未计算 | `t` 的 bit map |

关键点：**不是新建一棵树，而是同一棵 Item 树从"未绑定"变成"已绑定"**。这就是第 ③ 层 IR 的本质 —— 它和 ② 是同一批对象，只是状态变了。

### 逻辑树 vs 绑定树：同一批对象的两副面孔

这是理解 ②→③ 的核心，用一个完整例子看"逻辑树"和"绑定树"到底差在哪。

SQL：`SELECT a, b+1 FROM t WHERE b > 5`

**逻辑树（②，04 篇 contextualize 之后、prepare 之前）**：

```
Query_block
├─ fields = [
│    Item_field("a",   field=nullptr, table_ref=nullptr, fixed=false),
│    Item_func_plus(
│      Item_field("b", field=nullptr, ...),
│      Item_int(1)
│    )
│  ]
├─ m_table_list = [ Table_ref("t", table=nullptr) ]      ← 表还没打开
└─ m_where_cond = Item_func_gt(
     Item_field("b", field=nullptr, ...),
     Item_int(5)
   )
```

此时每个 `Item_field` 只有**名字字符串**（`item_name="a"`），`field`/`table_ref` 都是 `nullptr`——它"知道有个叫 a 的列，但不知道 a 是哪张表的第几列、什么类型"。

**绑定树（③，prepare 之后）**：

```
Query_block                                     ← 同一批对象！
├─ fields = [
│    Item_field("a",   field=&t.a, table_ref=&Table_ref(t), fixed=true),
│    Item_func_plus(
│      Item_field("b", field=&t.b, ...),
│      Item_int(1)
│    )
│  ]
├─ m_table_list = [ Table_ref("t", table=&TABLE_t) ]     ← 表已打开
└─ m_where_cond = Item_func_gt(
     Item_field("b", field=&t.b, ...),
     Item_int(5)
   )
```

**内存视角看"同一批对象"**——树的结构一个字节都没变，只有指针从空变实：

```
逻辑树（②）                          绑定树（③）
─────────────────────               ─────────────────────
Item_field("a")                     Item_field("a")
  field ──────────▶ nullptr           field ──────────▶ &t.a（真实 Field*）
  table_ref ──────▶ nullptr           table_ref ──────▶ Table_ref(t)
  fixed = false                       fixed = true
  used_tables = 未计算                 used_tables = t 的位图
                                        ↑
                          同一个对象！只是指针被填上了
```

**所以"逻辑树"和"绑定树"不是两棵树，是同一棵树的两种状态**：

| | 逻辑树（②） | 绑定树（③） |
|---|---|---|
| 载体 | `Query_block` + `Item` 树 | **同一批对象** |
| `Item_field::field` | `nullptr`（只有列名） | 真实 `Field*`（知道列定义） |
| `Table_ref::table` | `nullptr`（表没打开） | `TABLE*`（表已打开） |
| `Item::fixed` | `false` | `true` |
| 含义 | "要查 a 这一列" | "要查 t 表的第 3 列（int 类型）" |

**为什么这样设计**（不新建树而是原地填充）：因为 `Item` 对象的数量庞大（一个复杂查询几千个节点），如果 prepare 新建一棵树，等于把全部节点复制一遍，内存翻倍。原地填充让 ②→③ 的代价是"填几个指针"，而不是"重建整棵树"。代价是每个 `Item` 都要能表达"未绑定"状态（`field==nullptr`），代码里到处是"先判 fixed 再求值"的防御。

> 对比 Postgres：它的 parse analysis 会**产出新的 Query 树**（解析即重建），MySQL 选择**原地补全**。这是历史与 mem_root 一次性回收模型共同决定的取舍。

### 全景图：prepare 内部的 resolve / transform 分界

`Query_block::prepare`（`sql_resolver.cc:179`）内部 46 步，收敛成**两大阶段**。下面是"每个函数属于哪个阶段、被谁调用、对应本文哪一节"的全景图——**读后面任何一节时，先回到这张图定位**：

```
Query_block::prepare                       ← 被 Query_expression::prepare 逐块调用
│
├─【resolve 阶段】名字解析 + 类型绑定（46 步前半，第 1~34 步）
│   │
│   ├─ [6] setup_tables ──────────────── 建 Table_ref / join nest（"三"表格第 1 行）
│   │
│   ├─ [7] resolve_placeholder_tables ── 解析派生表/视图/CTE
│   │      ├─ merge_derived ──────────── 派生表"拍平"进外层（"七·7.1"）★
│   │      └─ setup_materialized_derived 派生表物化
│   │
│   ├─ [14] setup_wild ───────────────── 展开 SELECT *（"三"）
│   ├─ [15] setup_base_ref_items ──────── 分配 base_ref_items 数组
│   ├─ [16] setup_fields ──────────────── SELECT 列表逐列解析（"三·3.1"）★
│   │        └─ Item_field::fix_fields ─── 列名→Field*（"四·4.2"）
│   │             └─ find_field_in_tables ─ 符号表查找 + 歧义检测（"四"）★
│   ├─ [19] setup_conds ───────────────── WHERE + 所有 ON（"三"）
│   ├─ [20] setup_group / [24] HAVING fix_fields / [27] setup_order / [29] resolve_limits
│   │
├─【transform 阶段】永久改写（46 步后半，第 35~43 步）
│   │
│   ├─ [35] resolve_subquery ──────────── 子查询候选登记（"六·6.2"）
│   ├─ [36] transform_scalar_subqueries 标量子查询 → 派生表
│   ├─ [42] flatten_subqueries ────────── IN/EXISTS → semi-join（"六·6.3"）★
│   └─ [43] apply_local_transforms ────── 局部变换（"六·6.4"）★
│            ├─ simplify_joins ────────── 外连接转内连接（"六·6.4"）
│            ├─ check_only_full_group_by ─ ONLY_FULL_GROUP_BY 检查
│            └─ prune_partitions ──────── 分区裁剪
│
└─（注：merge_derived 属于 resolve 阶段，不是 transform！它由第 7 步
    resolve_placeholder_tables 调用，处理"派生表定义"的解析，虽然它会
    永久改写结构，但触发点在 resolve 阶段的名字解析里。）
```

**三条读图线索**：

1. **`setup_*` 全是 resolve 阶段**（第 6~29 步），它们的共同点是"填 `Field*`/`Table_ref*`/`fixed` 标志"，不改变树的结构。
2. **`flatten_*` / `apply_local_transforms` / `simplify_joins` 是 transform 阶段**（第 35~43 步），它们的共同点是"永久改写树结构"（`assert(first_execution)`）。
3. **带 ★ 的是本文有逐行剖析的**，其余只有表格一句话。

---

## 一、prepare 的两层入口

```cpp
// sql/sql_union.cc:694
bool Query_expression::prepare(THD *thd, Query_result *sel_result, ...);
// sql/sql_resolver.cc:179
bool Query_block::prepare(THD *thd, mem_root_deque<Item *> *insert_field_list);
```

**外层（Query_expression）for 循环遍历所有 Query_block**：

```cpp
// sql/sql_union.cc:734-757
  for (Query_block *sl = first_query_block(); sl; sl = sl->next_query_block()) {
    ...
    if (sl->prepare(thd, insert_field_list)) return true;   // :757
  }
  ...
  prepare_query_term(...);        // :864-867  递归处理 set-op 树的"后处理 query block"
  set_prepared();                 // :874  prepared = true
```

> 注意：UNION 各分支的 fake query block 不是在 734 那个循环里 prepare 的，而是在 `prepare_query_term()`（`sql_union.cc:414`，定义在 :565/623）里递归处理。

**入口在 `Sql_cmd_dml::prepare`**（`sql/sql_select.cc:495`），它调 `prepare_inner()`（:568）最终走到 `Query_block::prepare`。

---

## 二、Query_block::prepare 的 46 步

（`sql/sql_resolver.cc:179` → `:606`，按代码实际执行顺序）

| # | 行号 | 步骤 |
|---|---|---|
| 1 | 190 | `is_table_value_constructor` → 短路到 `prepare_values()`（VALUES 表值构造器） |
| 2 | 194 | `propagate_nullability(&m_table_nest, false)` |
| 3 | 207-212 | 计算 `allow_merge_derived`（允许 merge 派生表） |
| 4 | 232-236 | 设 `mark_used_columns` / `want_privilege`（derived 处理期间延迟权限检查） |
| 5 | 242 | `is_item_list_lookup = false`（lateral 不允许引用 select list） |
| 6 | **246** | **`setup_tables()`** —— 建 leaf table 链表、分配 tableno/map |
| 7 | 248-250 | 有 derived → **`resolve_placeholder_tables()`**（内含 merge_derived / setup_materialized_derived） |
| 8 | 253-255 | `check_view_privileges()` |
| 9 | 257 | `is_item_list_lookup = true` |
| 10 | 260-262 | `setup_natural_join_row_types()` |
| 11 | **264-265** | **`set_sj_candidates(&sj_candidates_local)`** —— 建立 semi-join 候选数组 |
| 12 | 274 | `parsing_place = CTX_NONE` |
| 13 | 276 | `resolve_place = RESOLVE_SELECT_LIST` |
| 14 | 278 | **`setup_wild()`** —— 展开 `SELECT *` |
| 15 | **279** | **`setup_base_ref_items()`** —— 分配 `base_ref_items` 数组 |
| 16 | **281-284** | **`setup_fields()`** —— SELECT list 的 fix_fields（递归触发子查询 prepare） |
| 17 | 286 | `resolve_place = RESOLVE_NONE` |
| 18 | 292 | `allow_sum_func &= ~(1 << nest_level)`（禁用 WHERE/GROUP BY 里的聚合） |
| 19 | **298** | **`setup_conds()`** —— WHERE + 所有 ON（含常量折叠） |
| 20 | **302-303** | **`setup_group()`** → `hidden_group_field_count` |
| 21 | 306 | 恢复 `allow_sum_func` |
| 22 | 309 | `m_deny_window_func |= ...`（HAVING 不允许窗口函数） |
| 23 | 311-316 | ROLLUP 标记 + `update_used_tables()` |
| 24 | **319-338** | **HAVING 的 fix_fields** |
| 25 | **340-341** | **`resolve_rollup()`** |
| 26 | 345-362 | HAVING 常量折叠（`simplify_const_condition`） |
| 27 | **365-369** | **`setup_order()`** → `hidden_order_field_count`(:376) |
| 28 | 371-374 | `fulltext_uses_rollup_column()` 检查 |
| 29 | **379** | **`resolve_limits()`** |
| 30 | 394-399 | `remove_redundant_subquery_clauses()` |
| 31 | **408-411** | **`Window::setup_windows1()`**（在 setup_order 之后、setup_order_final 之前） |
| 32 | 415-418 | **`setup_order_final()`** |
| 33 | 422-423 | `can_skip_distinct()` → 去掉 `SELECT_DISTINCT` |
| 34 | 437 | `opt_trace_print_expanded_query()` |
| 35 | **446-453** | **`resolve_subquery()`**（仅当本块是 IN/ANY/ALL/EXISTS 的子查询） |
| 36 | **466-472** | **`transform_scalar_subqueries_to_join_with_derived()`** |
| 37 | 481-495 | `m_having_cond->split_sum_func2()` + `inner_sum_func_list` 拆分 |
| 38 | 497-514 | 新增聚合时重建 rollup switcher |
| 39 | 516-530 | GROUP BY 里的 BIT 列加隐藏列 |
| 40 | 533-544 | `lift_fulltext_from_having_to_select_list()` + **`setup_ftfuncs()`** |
| 41 | 547 | `query_result()->prepare()` |
| 42 | **549-551** | **`flatten_subqueries()`**（semi-join flatten）→ `set_sj_candidates(nullptr)` |
| 43 | **559-594** | **`apply_local_transforms()`**（`simplify_joins` + `prune_partitions` + 条件下推） |
| 44 | 597 | `Window::eliminate_unused_objects()` |
| 45 | 601-602 | `resolve_rollup_wfs()`（ROLLUP + 窗口函数） |
| 46 | 605 | `return false` |

**观察这条顺序能看出的几个设计约束**：

1. **先表后列**：`setup_tables`（6）在 `setup_fields`（16）之前 —— 必须先知道有哪些表，才能解析列名。
2. **SELECT 列表先于 WHERE**（16 在 19 之前）—— 因为 ORDER BY/GROUP BY 可以引用 SELECT 列表别名。
3. **窗口函数夹在 ORDER BY 和 order_final 之间**（31 在 27 与 32 之间）—— 因为窗口的 PARTITION/ORDER 要复用 ORDER BY 的解析结果。
4. **flatten 在最后**（42）—— 先完成本块解析，才能把子查询拍平进来。

### 2.1 prepare 主函数逐行剖析（为什么是这个顺序）

46 步看似平铺，其实收敛成**四个阶段**，每阶段有严格先后依赖。`Query_block::prepare`（`sql_resolver.cc:179`）的骨架：

```cpp
bool Query_block::prepare(THD *thd, ...) {
  // ── 阶段 A：先表后列（符号表先建，引用后解析）────────────
  if (setup_tables(thd, get_table_list(), false)) return true;    // :246
  if (derived_table_count && resolve_placeholder_tables(thd, true))// :248
    return true;
  if (setup_wild(thd)) return true;                                // :278  展开 *
  if (setup_base_ref_items(thd)) return true;                      // :279
  if (setup_fields(thd, ..., &fields, base_ref_items)) return true; // :281

  // ── 阶段 B：条件与分组（列已绑定，表达式才能 resolve）────
  if (setup_conds(thd)) return true;                               // :298  WHERE+ON
  if (group_list.elements && setup_group(thd)) return true;        // :302
  if (m_having_cond) { m_having_cond->fix_fields(...); }           // :319-326
  if (order_list.elements) setup_order(thd, ...);                  // :365
  if (resolve_limits(thd)) return true;                            // :379

  // ── 阶段 C：子查询改写（本块 resolve 完，才能改子查询）────
  if (unit->item && unit->is_leaf_block(this) &&                   // :446-449
      !thd->lex->is_view_context_analysis() && resolve_subquery(thd))
    return true;
  if (has_sj_candidates() && flatten_subqueries(thd)) return true; // :549

  // ── 阶段 D：局部变换（顶层 QB 发起，递归下推）──────────────
  if (!thd->lex->is_view_context_analysis() &&                    // :559-565
      (outer_query_block() == nullptr || ...) && !skip_local_transforms)
    if (apply_local_transforms(thd, true)) return true;            // :593
  return false;
}
```

逐段解释：

**阶段 A —— 先表后列**：`setup_tables`(:246) 必须先于 `setup_fields`(:281)，因为 `Item_field::fix_fields` 靠 `find_field_in_tables` 查表定义。没有表，列名就是悬空的引用。这是编译器"符号表先建立、引用后解析"的直接体现。中间夹着 `setup_wild`(:278) 展开 `*`、`setup_base_ref_items`(:279) 分配 `base_ref_items` 数组——它们都依赖 `setup_tables` 已分配好 `tableno`/`map`。

**阶段 B —— 条件与分组**：WHERE/ON(:298) 在 SELECT 列表(:281) 之后，因为 GROUP BY/ORDER BY 能引用 SELECT 列表的别名（`SELECT a+b AS c ... ORDER BY c`）。HAVING(:319) 又在 GROUP(:302) 之后，因为 HAVING 的聚合要等 GROUP 处理完 `allow_sum_func` 状态（:292 禁用、:306 恢复）。ORDER(:365)/LIMIT(:379) 最后，因为它们还要引用 hidden item。

**阶段 C —— 子查询改写**：`resolve_subquery`(:452) 和 `flatten_subqueries`(:549) 放在最后，因为 flatten 是**自底向上**的——外层 QB 要等内层子查询先 prepare 完（见 6.2 源码注释），才能决定是否拍平。这也解释了为什么 46 步里 transform 是 42/43 步，而不是更早。

**阶段 D —— 局部变换**：`apply_local_transforms`(:593) 由**顶层 QB 发起**（`outer_query_block() == nullptr` 才进），然后递归下推。注释(:584-591) 说明动机："Local transforms are applied **after query block merging**"——先等派生表 merge 完，再统一 simplify，避免 merge 前后各做一次（那会浪费且可能出错）。

---

## 三、setup_* 函数族

| 函数 | 位置 | 一句话职责 |
|------|------|-----------|
| `setup_tables` | `sql_resolver.cc:1169` | 构造 leaf table 链表（`make_leaf_tables` :1177），分配 `tableno`/`map`。**没有它所有 Item_field 会被当成 const**（注释 :1162） |
| `resolve_placeholder_tables` | `sql_resolver.cc:1284` | 遍历 FROM：view/derived 调 `resolve_derived`；可 merge 调 `merge_derived`(:1309)；其余物化 |
| `setup_wild` | `sql_resolver.cc:1645` | 展开 `SELECT *`；`EXISTS(SELECT *)` 特化为 `Item_int(1)`(:1677) |
| `setup_base_ref_items` | `sql_lex.cc:2486` | 分配 `base_ref_items`（`Ref_item_array`，`sql_lex.h:2159`） |
| `setup_fields` | `sql_base.cc:8971` | SELECT list 逐个 fix_fields + 聚合拆分 + 权限检查 |
| `setup_conds` | `sql_resolver.cc:1701` | WHERE(:1723) + 所有 ON(:1762 `setup_join_cond`) + 常量折叠 |
| `setup_group` | `sql_resolver.cc:4705` | 遍历 group_list，禁止 GROUP BY 里出现聚合/窗口函数 |
| `setup_order` | `sql_resolver.cc:4529` | 解析 ORDER BY：先在 SELECT list 里定位（位置/别名），找不到就作为 hidden item 加进 fields |
| `setup_order_final` | `sql_resolver.cc:4660` | 窗口 setup 后收尾 ORDER BY |
| `resolve_limits` | `sql_resolver.cc:894` | LIMIT/OFFSET 的 fix_fields |
| `setup_ftfuncs` | `sql_base.cc:10295` | 每个 MATCH 调 `fix_index`，等价 MATCH 归并到 master |
| `resolve_rollup` | `sql_resolver.cc:5063` | ROLLUP：SELECT 顶层聚合替换成 `Item_rollup_sum_switcher` |
| `resolve_rollup_wfs` | `sql_resolver.cc:5158` | 窗口函数体内的 ROLLUP 引用替换 |
| `setup_natural_join_row_types` | `sql_base.cc:8850` | NATURAL/USING join 的行类型与公共列 |

### 3.1 setup_fields 驱动循环逐行剖析

> **上下文**：被 `Query_block::prepare` 第 16 步调用（`:281`），属于 **resolve 阶段**。它驱动 SELECT 列表的逐列 `fix_fields`，是"resolve 怎么遍历 Item 树"的载体。

`setup_fields`（`sql_base.cc:8971`）是"逐列 fix_fields"的驱动循环，也是理解"resolve 怎么遍历 Item 树"的关键：

```cpp
bool setup_fields(THD *thd, ..., mem_root_deque<Item *> *fields,
                  Ref_item_array ref_item_array) {
  ...
  for (auto it = fields->begin(); it != fields->end(); ++it) {
    Item *item = *it;
    Item **item_pos = &*it;                       // 可替换指针！
    // ① 名字解析 + 类型绑定（可能替换 item 本身）
    if ((!item->fixed && item->fix_fields(thd, item_pos)) ||
        (item = *item_pos)->check_cols(1))
      return true;
    // ② 聚合函数拆分
    if (split_sum_funcs &&
        ((item->has_aggregation() && !(item->type() == Item::SUM_FUNC_ITEM &&
                                       !item->m_is_window_function)) ||
         item->has_wf()))
      item->split_sum_func(thd, ref_item_array, fields);
    // ③ 累积 select list 涉及的表
    select->select_list_tables |= item->used_tables();
    // ④ split 可能往 fields 加 item，迭代器失效需重建
    if (old_size != fields->size())
      it = std::find(fields->begin(), fields->end(), item);
  }
}
```

逐段解释：

1. **`Item **item_pos = &*it`** —— 这是最容易被忽略的细节：传的是**指针的指针**。因为 `fix_fields` 可能把 item **替换成另一个对象**（视图展开替换成定义表达式、别名引用换成 `Item_ref`、常量折叠换成 `Item_func_true`），所以必须用二级指针让被调方改写 `fields` 里的槽位。这就是 `fix_fields(THD *, Item **reference)` 第二个参数是 `Item**` 的原因。

2. **`item->fix_fields(thd, item_pos)`** —— 驱动名字解析（见 4.2 的三步查找）。`!item->fixed` 短路避免重复 fix（Prepared Statement 二次执行时已 fixed 的跳过）。

3. **`split_sum_func`** —— 把聚合函数从表达式里"拆"出来，作为 hidden item 加进 `fields`（临时表字段）。这是 MySQL 聚合的特殊机制：`SELECT a+SUM(b) FROM t` 里 `SUM(b)` 要先算出来，`a` 才能和它相加，所以把 `SUM(b)` 拆成独立的 hidden 列。

4. **`select_list_tables |= item->used_tables()`** —— 累积 select list 涉及的表位图，供后续判断（如是否需要临时表、read_set 计算）。

5. **迭代器失效重建** —— `split_sum_func` 会往 `fields` 这个 deque 里 `push_back` 新 item，导致原 `it` 失效，所以比较 `old_size != fields->size()` 后重新 `std::find` 定位。这是"边遍历边修改容器"的经典陷阱的解法。

### 3.2 prepare 的三类非解析改写：标注 / 物化 / 精简

`prepare` 的主体是名字解析（`setup_*` + `fix_fields`），但 46 步里还混着一类**不解析任何名字、只标注语义或改写逻辑结构**的工作。它们按架构分三类，且在 46 步里的先后顺序正好对应"**先标注 → 再物化 → 最后精简**"：

| 类别 | 函数 | 46 步 | 对查询做什么 | 产生新结构？ |
|------|------|-------|-------------|-------------|
| **语义标注** | `propagate_nullability` | 2 | 给表打"可空"标签（`TABLE::set_nullable`） | 否，只改元数据 |
| **结构物化** | `setup_table_function` | 7 | 表函数（JSON_TABLE 等）→ 物化临时表 | 是 |
| **结构精简** | `remove_redundant_subquery_clauses` | 30 | 删子查询冗余 ORDER/DISTINCT/GROUP | 是（删减） |
| **结构精简** | `can_skip_distinct` | 33 | 删多余 DISTINCT | 是（删减） |

**架构关系**：`propagate_nullability` 必须在最前——nullable 是后续一切改写（`simplify_joins` 的 `not_null_tables`、访问方法不能选 `EQ_REF`、anti-join 的 `IS NULL` 惯用法）的前置语义。精简必须在最后——要等 `setup_group`/`setup_order` 都跑完，才知道哪些 GROUP/DISTINCT/ORDER 是冗余的。下面按这个架构顺序逐行剖析。

**① `propagate_nullability`：语义标注（46 步第 2 步，逐行剖析）**

外连接 `A LEFT JOIN B` 中，B 可能被 NULL 补全，这个"可空性"要**传播**到 B 的所有列和引用它的表达式。`propagate_nullability(&m_table_nest, false)` 在 prepare 最开头做，因为 nullable 会影响后续一切（访问方法不能用 `EQ_REF`、`simplify_joins` 依赖 `not_null_tables`、`IS NULL` 反连接惯用法靠"内表可空"保留语义）。

```cpp
// sql_resolver.cc:4041
void propagate_nullability(mem_root_deque<Table_ref *> *tables, bool nullable) {
  for (Table_ref *tr : *tables) {                                    // ① 遍历本层 table list
    if (tr->table && !tr->table->is_nullable() &&
        (nullable || tr->outer_join))                                // ② 判定是否标 nullable
      tr->table->set_nullable();                                     // ③ 置 nullable 标志
    if (tr->nested_join == nullptr) continue;                        // ④ 非嵌套 join 跳过
    propagate_nullability(&tr->nested_join->m_tables,
                          nullable || tr->outer_join);               // ⑤ 递归，nullable 累积
  }
}
```

逐行解释：

| 行 | 逻辑 | 说明 |
|----|------|------|
| ② | `!is_nullable() && (nullable || tr->outer_join)` | 表"尚未标 nullable"，且（**父层传下来的 nullable** 或 **自己就是 outer join 内表**）才置位 |
| ⑤ | `nullable || tr->outer_join` | **nullable 是累积参数**：只要路径上经过任一 outer join 内表，往下就全 nullable |

**关键设计——`nullable` 是"向下累积的可空性"**：顶层调用传 `false`（`&m_table_nest, false`），每递归进一层 `LEFT JOIN` 内表，`nullable` 就变 `true` 并继续向下传播。所以 `t1 LEFT JOIN (t2 LEFT JOIN t3)` 里，t2、t3 都 nullable，而 `t1` 永远不 nullable。

配套：`convert_subquery_to_semijoin` 里 anti-join 化后也调 `propagate_nullability(&sj_nest->..., true)`（`:3265`）——因为 anti-join 建模成 `LEFT JOIN + IS NULL`，右表（原子查询表）必须标 nullable。

**② `setup_table_function`：结构物化（46 步第 7 步，逐行剖析）**

表函数（`JSON_TABLE(...)` 等）本质是"**物化成临时表**"的 derived table 特例，在 prepare 里做**两遍**：先建表结构、再解析参数。

```cpp
// sql_derived.cc:944
bool Table_ref::setup_table_function(THD *thd) {
  assert(is_table_function());                        // ① 前置：必须真·表函数
  set_uses_materialization();                         // ② 标记走物化
  query_block->end_lateral_table = this;              // ③ LATERAL 边界：表函数自动 LATERAL

  if (table_function->init()) return true;            // ④ 第一遍开始：初始化函数（确定输出列）

  if (table_function->create_result_table(thd, 0LL, alias))  // ⑤ 建结果临时表
    return true;
  table = table_function->table;                      // ⑥ 拿到结果表 TABLE
  table->pos_in_table_list = this;                    // ⑦ 反指回本 Table_ref
  table->s->tmp_table = NON_TRANSACTIONAL_TMP_TABLE;  // ⑧ 标记为临时表

  if (is_inner_table_of_outer_join())                 // ⑨ 外连接内表
    table->set_nullable();                            //     → 标 nullable（同 ① 的可空语义）

  const char *saved_where = thd->where;               // ⑩ 保存错误上下文
  thd->where = "a table function argument";
  enum_mark_columns saved_mark = thd->mark_used_columns;
  thd->mark_used_columns = MARK_COLUMNS_READ;         //    参数里的列要标记为"读"
  if (table_function->init_args()) return true;       // ⑪ 第二遍：解析参数（列引用在此 fix_fields）

  thd->mark_used_columns = saved_mark;                // ⑫ 恢复上下文
  set_privileges(SELECT_ACL);
  query_block->end_lateral_table = nullptr;           // ⑬ 复位 LATERAL 边界
  thd->where = saved_where;
  return false;
}
```

**"两遍"的本质**：

| 遍 | 调用 | 做什么 |
|----|------|--------|
| 第一遍 | `init()` + `create_result_table()` | 确定输出列结构，建临时表 |
| 第二遍 | `init_args()` | 解析函数的参数（参数里的列引用、表达式在此 `fix_fields`） |

为什么必须两遍：**`JSON_TABLE` 的输出列是参数决定的**——`JSON_TABLE(doc, '$.a' COLUMNS(x INT ...))` 要先知道 `COLUMNS` 定义才能建表，但参数里的 `doc`（可能是外层表的列）又需要 name resolution。所以先 `init()` 解析结构、`create_result_table` 建表，再 `init_args()` 解析参数。

两个关键点：

- **③/⑬ LATERAL 边界**：表函数天然 LATERAL（能引用 FROM 里它**前面**的表），`end_lateral_table = this` 保证它**不能**访问它后面的表（否则 name resolution 越界）。setup 完复位 `nullptr`。
- **⑨ nullable**：表函数若在 outer join 内表，同样 `set_nullable()`——和 ① `propagate_nullability` 是**同一套可空语义**（这就是"标注"和"物化"的交集）。

调用点在 `Query_block::setup_tables`（`sql_resolver.cc:1315`）：derived/视图/表函数统一走"merge 或 materialize"分支，表函数走 `setup_table_function`（`:1314`），普通物化 derived 走 `setup_materialized_derived`（`:1318`）。

**③ `remove_redundant_subquery_clauses`：结构精简（46 步第 30 步，逐行剖析）**

对 `IN/ANY/ALL/EXISTS` 子查询，如果它**没有聚合、没有 HAVING**，那么它的 `ORDER BY`/`DISTINCT`/`GROUP BY` 对结果集（只判断"存在性/成员性"）毫无影响，可以删掉。

```cpp
// sql_resolver.cc:4142
bool Query_block::remove_redundant_subquery_clauses(THD *thd,
                                                    int hidden_group_field_count) {
  Item_subselect *subq_predicate = master_query_expression()->item;   // ① 拿到所属子查询谓词
  enum change { REMOVE_NONE = 0, REMOVE_ORDER = 1 << 0,
                REMOVE_DISTINCT = 1 << 1, REMOVE_GROUP = 1 << 2 };    // ② 位图枚举三种可删项
  uint possible_changes;

  if (subq_predicate->substype() == Item_subselect::SINGLEROW_SUBS) { // ③ 标量子查询
    if (has_limit()) return false;              // 有 LIMIT 就不动（行数语义）
    possible_changes = REMOVE_ORDER;            // 标量只删 ORDER BY
  } else {                                      // ④ IN/ANY/ALL/EXISTS
    assert(...);
    possible_changes = REMOVE_ORDER | REMOVE_DISTINCT | REMOVE_GROUP; // 三种都可删
  }

  uint changelog = 0;
  if ((possible_changes & REMOVE_ORDER) && order_list.elements) {     // ⑤ 删 ORDER
    changelog |= REMOVE_ORDER;
    if (empty_order_list(this)) return true;
  }
  if ((possible_changes & REMOVE_DISTINCT) && is_distinct()) {        // ⑥ 删 DISTINCT
    changelog |= REMOVE_DISTINCT;
    remove_base_options(SELECT_DISTINCT);
  }
  if ((possible_changes & REMOVE_GROUP) && group_list.elements &&     // ⑦ 删 GROUP：须无聚合/无 HAVING/无 ROLLUP/无窗口
      !agg_func_used() && !having_cond() && olap == UNSPECIFIED_OLAP_TYPE &&
      m_windows.elements == 0) {
    changelog |= REMOVE_GROUP;
    for (ORDER *g = group_list.first; g != nullptr; g = g->next) {    // 遍历每个 group 表达式
      if (g->is_item_original()) {                                    // 只清理"原始"item（非别名引用）
        Item::Cleanup_after_removal_context ctx(this);                // 构造清理上下文
        (*g->item)->walk(&Item::clean_up_after_removal, walk_options, // 递归清理 item 树
                         pointer_cast<uchar *>(&ctx));                 //（移除子查询注册等副作用）
      }
    }
    group_list.clear();                                               // 清空 group_list
    while (hidden_group_field_count-- > 0) {                          // 回退隐藏分组列
      fields.pop_front();                                             // 从 select list 弹出
      base_ref_items[fields.size()] = nullptr;                        // 同步清空 base_ref_items
    }
  }
  if (changelog) { /* opt_trace 记录 removed_ordering/removed_distinct/removed_grouping */ }
  return false;
}
```

逐行解释的关键点：

1. **③ 标量 vs 表子查询分两档**：`SINGLEROW_SUBS`（标量）只能删 `ORDER BY`（因为标量结果行数本身有"≤1 行"约束，删 DISTINCT/GROUP 会改变行数语义）；`IN/ANY/ALL/EXISTS` 三种都能删。
2. **③ `has_limit()` 直接 return**：子查询带 `LIMIT` 时行数语义受 LIMIT 约束，一个都不删——这是"存在性判断"与"行数语义"冲突时，**行数优先**。
3. **⑦ 删 GROUP 有 4 个前置条件**：`!agg_func_used()`（有聚合函数，GROUP 就有去重分组语义，不能删）、`!having_cond()`（HAVING 依赖分组）、`olap == UNSPECIFIED_OLAP_TYPE`（ROLLUP 不能删）、`m_windows.elements == 0`（窗口依赖分组顺序）。
4. **⑦ 删 GROUP 后要回退 `hidden_group_field_count` 个隐藏列**：`setup_group()` 往 `fields` 里塞的隐藏分组列（用于 ORDER BY 引用未在 SELECT 的 group 列）要一并 `pop_front` 掉，否则 select list 长度错位。

```sql
-- 例：IN 只关心"值在不在"，不关心顺序/去重/分组
WHERE t1.c2 IN (SELECT DISTINCT c1 FROM t2 GROUP BY c1, c2 ORDER BY c1)
-- 优化为 =>
WHERE t1.c2 IN (SELECT c1 FROM t2)
```

**④ `can_skip_distinct`：结构精简（46 步第 33 步，逐行剖析）**

`SELECT DISTINCT c1, MAX(c2) FROM t1 GROUP BY c1` 里的 `DISTINCT` 是多余的——`GROUP BY c1` 已经保证 c1 唯一。

```cpp
// sql_lex.h:1462
bool can_skip_distinct() const {
  return is_grouped() && hidden_group_field_count == 0 &&
         olap == UNSPECIFIED_OLAP_TYPE;
}
```

三个条件逐条：

| 条件 | 含义 | 为什么 |
|------|------|--------|
| `is_grouped()` | 有 GROUP BY | DISTINCT 的去重由 GROUP BY 的分组天然保证 |
| `hidden_group_field_count == 0` | 无隐藏分组列 | 若有"不在 SELECT 列表的 group 列"，分组行并非在 SELECT 投影上唯一 |
| `olap == UNSPECIFIED_OLAP_TYPE` | 无 ROLLUP | ROLLUP 会追加 NULL 汇总行，破坏唯一性 |

调用点（`sql_resolver.cc:422`）：

```cpp
if (is_distinct() && can_skip_distinct()) {
  remove_base_options(SELECT_DISTINCT);   // 去掉 DISTINCT 位，后续不再去重
}
```

### 3.3 setup_conds 与 resolve_limits：条件与 LIMIT 的逐行剖析

**`setup_conds`**（46 步第 19 步，`sql_resolver.cc:1701`）—— WHERE + 所有 ON 的 fix_fields：

```cpp
bool Query_block::setup_conds(THD *thd) {
  const bool it_is_update = (this == thd->lex->query_block) &&   // ① 顶层 QB 且是 UPDATE/INSERT 场景
                            thd->lex->which_check_option_applicable();
  const bool save_is_item_list_lookup = is_item_list_lookup;
  is_item_list_lookup = false;                                   // ② 条件里禁止引用 SELECT 列表别名

  if (m_where_cond) {                                            // ③ WHERE 的 fix_fields
    resolve_place = Query_block::RESOLVE_CONDITION;              // ④ 标记"当前在解析条件"
    thd->where = "where clause";                                 // ⑤ 错误消息上下文
    if ((!m_where_cond->fixed && m_where_cond->fix_fields(thd, &m_where_cond)) ||
        m_where_cond->check_cols(1))
      return true;

    // ⑥ WHERE 是常量 → 常量折叠（1=1 消掉 / 1=0 恒假）
    if (m_where_cond->const_item() && !thd->lex->is_view_context_analysis() &&
        !m_where_cond->walk(&Item::is_non_const_over_literals, enum_walk::POSTFIX, nullptr) &&
        simplify_const_condition(thd, &m_where_cond))
      return true;
    resolve_place = Query_block::RESOLVE_NONE;                   // ⑦ 复位
  }

  // ⑧ 所有 ON 条件的递归解析
  if (!m_table_nest.empty() && setup_join_cond(thd, &m_table_nest, it_is_update))
    return true;

  is_item_list_lookup = save_is_item_list_lookup;                // ⑨ 恢复
  return false;
}
```

逐行解释的关键点：

1. **② `is_item_list_lookup = false`**：WHERE/ON 里**不能**引用 SELECT 列表的别名（`SELECT a+b AS c ... WHERE c>1` 报错），只有 GROUP BY/ORDER BY/HAVING 可以。所以解析条件期间临时关闭，⑨ 再恢复。
2. **④ `resolve_place = RESOLVE_CONDITION`**：全局"当前解析位置"标记，两个下游消费它：
   - `resolve_subquery` 的半连接判定（`outer->resolve_place == RESOLVE_CONDITION`，即 02 篇 16 条件之 6a）
   - lateral 派生表"能引用哪些表"的边界判断
3. **⑥ 常量折叠的三个前置**：`const_item()`（是常量）+ 非视图上下文分析 + **`!is_non_const_over_literals`**（`WHERE 'a' = (SELECT ...)` 这种"字面量套子查询"不能折叠——子查询有副作用）。满足才 `simplify_const_condition`。

**`setup_join_cond`**（`:1762`）—— ON 的递归解析：

```cpp
bool Query_block::setup_join_cond(THD *thd, mem_root_deque<Table_ref *> *tables, bool in_update) {
  for (Table_ref *tr : *tables) {
    if (tr->nested_join != nullptr &&                              // ① 先递归进嵌套 join nest
        setup_join_cond(thd, &tr->nested_join->m_tables, in_update))
      return true;

    Item **ref = tr->join_cond_ref();
    if (join_cond) {
      resolve_place = Query_block::RESOLVE_JOIN_NEST;              // ② ON 的 resolve 位置
      resolve_nest = tr;                                           // ③ 当前 nest（lateral 边界）
      thd->where = "on clause";
      if ((!join_cond->fixed && join_cond->fix_fields(thd, ref)) || check_cols(1))
        return true;
      cond_count++;                                                // ④ 条件计数

      if ((*ref)->const_item() && !is_view_context_analysis() && ... &&
          simplify_const_condition(thd, ref, remove_cond))         // ⑤ ON 常量折叠
        return true;
      resolve_place = Query_block::RESOLVE_NONE;
      resolve_nest = nullptr;
    }
    if (in_update) { /* ⑥ 视图 CHECK OPTION */ }
  }
  return false;
}
```

三个关键点：

1. **① 先递归后处理**：嵌套 join nest 的 ON **先于**外层解析——外层 ON 可能引用内层 nest 的表，内层的名字解析上下文要先建好。
2. **③ `resolve_nest = tr`**：记录"当前在哪个 nest 的 ON 里"，是 **lateral 派生表可见表边界**的依据——ON 里只能引用本 nest 之前（含）的表（`setup_table_function` 的 `end_lateral_table` 是同一套机制）。
3. **⑤ ON 常量折叠语义与 WHERE 不同**：`ON TRUE` 时 `remove_cond=true` 直接删条件；`ON FALSE` 变恒假——**外连接走 NULL 补行、内连接不产生匹配**，而 WHERE 恒假是整个查询空。注释 `:1774` 的 `remove_cond` 参数就是干这个的。

**`resolve_limits`**（46 步第 29 步，`:894`）：

```cpp
bool Query_block::resolve_limits(THD *thd) {
  if (offset_limit != nullptr) {
    if (offset_limit->fix_fields(thd, nullptr)) return true;       // ① fix_fields
    if (offset_limit->data_type() == MYSQL_TYPE_INVALID) {         // ② 类型未定 → 强制 LONGLONG
      if (offset_limit->propagate_type(thd, Type_properties(MYSQL_TYPE_LONGLONG, true)))
        return true;
      offset_limit->pin_data_type();                               // ③ 钉死类型
    }
  }
  if (select_limit != nullptr) { /* ④ LIMIT 行数同理 */ }
  return false;
}
```

**关键点**：LIMIT/OFFSET 的参数被**强制成 LONGLONG**（② `propagate_type(MYSQL_TYPE_LONGLONG)`）——这就是 `LIMIT 'abc'` 报错、`LIMIT 1.5` 被拒绝的根源。③ `pin_data_type()` 钉死类型，防止后续改写（如条件下推）再动它。

---

## 四、fix_fields：一棵树绑定出语义

### 4.1 基类只做标记

```cpp
// sql/item.cc:4893
bool Item::fix_fields(THD *, Item **) {
  assert(is_contextualized());
  assert(fixed == 0 || basic_const_item());
  fixed = true;      // 就这一件事
  return false;
}
```

真正的活在各子类。`Item_func::fix_fields`（`item_func.cc:397`）是**自底向上递归**的典型：

```cpp
// sql/item_func.cc:397
bool Item_func::fix_fields(THD *thd, Item **) {
  ...
  if (arg_count) {                       // 1. 递归 fix 每个参数
    for (arg = args, arg_end = args + arg_count; arg != arg_end; arg++) {
      if (fix_func_arg(thd, arg)) return true;
    }
  }
  if (resolve_type(thd) || thd->is_error())   // 2. 类型/collation 定型
    return true;
  fixed = true;                          // 3. 标记
  return false;
}
```

`fix_func_arg`（`:429`）自底向上累积 `used_tables`：

```cpp
  used_tables_cache |= item->used_tables();   // :444  OR 累积
```

> ⚠️ **命名变更**：`fix_length_and_dec()` 在 8.0.39 已整体改名 **`resolve_type()`**（`item.h:5679`，纯虚）。网上资料写 `fix_length_and_dec` 的，只剩 `item_subselect.{h,cc}` 里还有。

### 4.2 Item_field::fix_fields —— 列名 → Field*

**定义 `item.cc:5690`**，算法伪码在注释 `item.cc:5648-5671`：

```cpp
// sql/item.cc:5720-5725
  from_field = find_field_in_tables(            // 第 1 步：查 FROM 列表
      thd, this, context->first_name_resolution_table,
      context->last_name_resolution_table, reference,
      ...);
  if (from_field == not_found_field || from_field == nullptr) {
    if (qb->is_item_list_lookup) {              // 第 2 步：查 SELECT list 别名
      if (find_item_in_list(thd, this, &qb->fields, &res, &counter, &resolution))
        return true;
      ...
    }
  }
  ...
  if (!from_field) { fix_outer_field(...); }    // 第 3 步：查外层（关联列）
  ...
  set_field(from_field);                        // :5870  最终绑定 Field*
```

三个查找函数：

| 函数 | 位置 | 职责 |
|------|------|------|
| `find_field_in_tables` | `sql_base.cc:7919` | 遍历 name resolution 上下文，**歧义报 `ER_NON_UNIQ_ERROR`**（循环 :8019） |
| `find_field_in_table_ref` | `sql_base.cc:7734` | 单表查找：视图 `find_field_in_view`(:7786)、实表 `find_field_in_table`(:7792)、NATURAL/USING(:7819) |
| `find_field_in_table` | `sql_base.cc:7658` | 在 `TABLE::field[]` 里线性 `my_strcasecmp` 比较 |

`Item_field::set_field`（`item.cc:2881`）：

```cpp
void Item_field::set_field(Field *field_par) {
  table_ref = field_par->table->pos_in_table_list;   // ← Table_ref*
  field_index = field_par->field_index();
  field = result_field = field_par;                  // ← Field*
  set_nullable(...);
}
```

#### find_field_in_tables：符号表查找的逐行剖析

`find_field_in_tables`（`sql_base.cc:7919`）是名字解析的核心，`Item_field::fix_fields` 第 1 步调的正是它。骨架：

```cpp
Field *find_field_in_tables(THD *thd, Item_ident *item, Table_ref *first_table,
                            Table_ref *last_table, Item **ref, ...) {
  Field *found = nullptr;
  const char *db = item->db_name;
  const char *table_name = item->table_name;
  const char *name = item->field_name;
  ...
  // ① 遍历 name resolution 上下文链表（作用域！）
  for (cur_table = first_table; cur_table != last_table;
       cur_table = cur_table->next_name_resolution_table) {
    Field *cur_field = find_field_in_table_ref(   // ② 单表查找
        thd, cur_table, name, length, ..., ref, ...);
    if (cur_field) {
      if (db) return cur_field;      // ③ 完全限定名（db.t.c）唯一，直接返回
      if (found) {                   // ④ 第二次命中 → 歧义！
        if (report_error == REPORT_ALL_ERRORS ||
            report_error == IGNORE_EXCEPT_NON_UNIQUE)
          my_error(ER_NON_UNIQ_ERROR, MYF(0),
                   table_name ? item->full_name() : name, thd->where);
        return nullptr;
      }
      found = cur_field;             // ⑤ 第一次命中，记下继续找（看是否歧义）
    }
  }
  if (found) return found;           // ⑥ 唯一命中，返回
  // ⑦ 未找到 → 区分"表不存在" vs "列不存在"
  if (table_name && cur_table == first_table && ...)
    my_error(ER_UNKNOWN_TABLE, ...);   // db.t.c 里 t 不存在
  else
    my_error(ER_BAD_FIELD_ERROR, ...); // 列不存在
}
```

逐段解释：

1. **① 作用域遍历**：`first_table` → `last_table` 的 `next_name_resolution_table` 链表，就是 name resolution context。**这个范围就是"作用域"**——内层子查询的可见表集合，由 contextualize 阶段建立（`context.first_name_resolution_table`）。相关子查询找外层列时，就是靠 `fix_outer_field` 逐层往外扩大这个范围。

2. **② 单表查找**：`find_field_in_table_ref`（`sql_base.cc:7734`）处理一张表——视图会递归展开（`find_field_in_view`）、实表线性匹配（`find_field_in_table`，在 `TABLE::field[]` 里 `my_strcasecmp` 比较）、NATURAL/USING 特判。

3. **③ 完全限定名**：`db` 非空说明是 `db.table.col` 三级名，MySQL 认为它**不可能有歧义**（同名表不能出现在一个 query 里），所以第一个命中就直接返回，不检查后续。

4. **④ 歧义检测**：`found` 已经有值，又命中一个 → 这就是 `ER_NON_UNIQ_ERROR`（"Column 'x' in field list is ambiguous"）的诞生地。注意要 `IGNORE_EXCEPT_NON_UNIQUE` 才报，因为某些场景（如 view 分析）要忽略歧义继续。

5. **⑤ 延迟返回**：第一次命中**不立即返回**，而是记下 `found` 继续遍历——因为要检测歧义。这解释了为什么名字解析是 O(表数) 而非 O(1)。

6. **⑦ 两种错误**：`table_name` 非空但没匹配到任何表 → `ER_UNKNOWN_TABLE`（表名写错）；表都在但列没找到 → `ER_BAD_FIELD_ERROR`（列名写错）。这区分对用户排错很关键。

### 4.3 常量折叠

两层：

**(a) `Item_cond` 在 fix_fields 时直接消常量子条件**（`item_cmpfunc.cc:5590-5595` → `remove_const_conds` :5676，替换成 `Item_func_true`/`false`）。

**(b) 顶层整树折叠** `simplify_const_condition`（`sql_resolver.cc:123`，内部 `eval_const_cond`）：WHERE(:1731)、ON(:1787)、HAVING(:357)。

> 注意 `sql/sql_const_folding.cc` 是**独立模块，在 optimize 阶段跑**，不属于 prepare。

### 4.4 used_tables 与类型

- `Item_field::used_tables()`：`item.cc:3128` —— `depended_from ? OUTER_REF_TABLE_BIT : table_ref->map()`；const 表返回 0
- 类型/字符集推导：`agg_item_collations`（`item.cc:2513`）、`agg_item_collations_for_comparison`（`:2548`）
- `Query_block::update_used_tables()`：`sql_resolver.cc:856`

---

## 五、name resolution 规则

官方注释（`item.cc:5648-5671`）定义的解析顺序：

```
resolve_column_reference([T_j].col_ref_i)
{
  search for col_ref_i [in T_j] in the FROM clause of Q;          ← ① 本层 FROM
  if NOT found AND there are outer queries {
    for each outer query Q_k from inner-most {
      search col_ref_i [in T_j] in FROM of Q_k;                   ← ② 外层 FROM（逐层）
      if not found: search in SELECT and GROUP of Q_k;            ← ③ 外层 SELECT/GROUP
    }
  }
}
```

即：**本层 FROM → 本层 SELECT 别名 → 外层 FROM（逐层）→ 外层 SELECT/GROUP**。

### 关联列（correlated column）如何标记

链路：`Item_field::fix_fields`(:5811) → `fix_outer_field`（`item.cc:5251`）→ `mark_as_dependent`（`item.cc:4963`）：

```cpp
  mark_item->depended_from = last;      // Item_ident::depended_from（item.cc:4970）
  resolved_item->depended_from = last;
  current->mark_as_dependent(last, false);   // Query_block::mark_as_dependent（sql_lex.cc:2377）
```

三个"关联"标记：

1. `Item_ident::depended_from` → `used_tables()` 返回 `OUTER_REF_TABLE_BIT`
2. `Query_block::uncacheable |= UNCACHEABLE_DEPENDENT`
3. `Query_expression::accumulate_used_tables(OUTER_REF_TABLE_BIT)`

外层引用落在**分组查询的 SELECT/HAVING** 时，还会被包成 `Item_outer_ref`（`item.cc:5379/5416`）—— ROLLUP 语义需要。

---

## 六、子查询改写与 semi-join

### 6.1 两套枚举，别混淆

**执行策略**（`sql_select.h:309-314`，是宏不是 enum）：

```cpp
#define SJ_OPT_NONE 0
#define SJ_OPT_DUPS_WEEDOUT 1          // Duplicate Weedout
#define SJ_OPT_LOOSE_SCAN 2            // LooseScan
#define SJ_OPT_FIRST_MATCH 3           // FirstMatch
#define SJ_OPT_MATERIALIZE_LOOKUP 4    // Materialize (lookup)
#define SJ_OPT_MATERIALIZE_SCAN 5      // MaterializeScan
```

**子查询候选策略**（`item_subselect.h:398-418`，`enum class Subquery_strategy`）：`UNSPECIFIED / CANDIDATE_FOR_IN2EXISTS_OR_MAT / CANDIDATE_FOR_SEMIJOIN / CANDIDATE_FOR_DERIVED_TABLE / SEMIJOIN / DERIVED_TABLE / SUBQ_EXISTS / SUBQ_MATERIALIZATION / DELETED`。

### 6.2 改写函数

| 函数 | 位置 | 做什么 |
|------|------|--------|
| `resolve_subquery` | `sql_resolver.cc:1368` | 候选登记：semi-join / antijoin / derived，否则 `select_transformer`（IN→EXISTS、物化、ALL/ANY→MIN/MAX） |
| `convert_subquery_to_semijoin` | `sql_resolver.cc:2992` | 把 IN/EXISTS 变 semi-join nest（注释 :2900 列 5 种形态） |
| `flatten_subqueries` | `sql_resolver.cc:3811` | 按 `sj_convert_priority` 排序（:3867：相关优先、表多优先、位置靠前优先）后逐个 flatten |
| `transform_scalar_subqueries_to_join_with_derived` | `sql_resolver.cc:7577` | 标量子查询 → LEFT JOIN 派生表（`subquery_to_derived=on`） |
| `apply_local_transforms` | `sql_resolver.cc:751` | 单块局部变换：删 merged 未用列、`simplify_joins`、`ONLY_FULL_GROUP_BY` 检查、分区裁剪、条件下推 |
| `simplify_joins` | `sql_resolver.cc:1951` | OUTER JOIN → INNER JOIN、join cond 上提、去括号 |

> **flatten 是"由外向内、自底向上"**（`sql_resolver.cc:3824-3843` 注释），而子查询的 prepare 是"由内向外"被 `fix_fields` 驱动 —— 两个方向别搞反。

### 6.3 flatten_subqueries 逐行剖析

> **上下文**：被 `Query_block::prepare` 第 42 步调用（`:549`），属于 **transform 阶段**（永久改写）。它把 IN/EXISTS 子查询"拍平"成 semi-join。

`flatten_subqueries`（`sql_resolver.cc:3811`）把 IN/EXISTS 子查询"拍平"成 semi-join，核心是**排序 + 两类转换**：

```cpp
bool Query_block::flatten_subqueries(THD *thd) {
  assert(has_sj_candidates());
  // ① 遍历候选，算优先级 + 清理 DELETED
  for (subq = subq_begin; subq < subq_end; subq++, subq_no++) {
    if (subq_item->strategy == Subquery_strategy::DELETED) {
      sj_candidates->erase_value(subq_item);   // 已删的移除
      subq--; subq_end = sj_candidates->end();
      continue;
    }
    bool dependent = subq_item->unit->uncacheable & UNCACHEABLE_DEPENDENT;
    subq_item->sj_convert_priority =
        (((dependent * MAX_TABLES_FOR_SIZE) +      // 相关子查询优先
          child_query_block->leaf_table_count) *   // 表多的优先
         65536) +
        (65536 - subq_no);                        // 位置靠前的优先
  }
  // ② 按优先级降序排序
  std::sort(subq_begin, ..., [](a, b) { return a->sj_convert_priority > b->sj_convert_priority; });

  // ③ 永久变换开始
  Prepared_stmt_arena_holder ps_arena_holder(thd);

  // ④ 第一类转换：IN (SELECT) → joined derived table
  for (...) if (strategy == CANDIDATE_FOR_DERIVED_TABLE)
    transform_table_subquery_to_join_with_derived(thd, subq_item);

  // ⑤ 第二类转换：IN/EXISTS → semi-join（表数检查 + 常量折叠）
  for (...) {
    if (strategy != CANDIDATE_FOR_SEMIJOIN) continue;
    if (table_count + tables_added <= MAX_TABLES && !has_aj_nests)   // 表数上限 + 无 anti-join
      subq_item->strategy = Subquery_strategy::SEMIJOIN;
    // 常量折叠：子查询 WHERE 恒 false → 整个子查询删掉
    if (subq_where && subq_where->const_item() && !cond_value)
      subq_item->walk(&Item::clean_up_after_removal, ...);
  }
}
```

逐段解释：

1. **① 优先级编码**（`:3868-3872`）：`sj_convert_priority` 把三个因素编码成一个整数——**相关子查询**（`dependent`）权重最高（乘以 `MAX_TABLES_FOR_SIZE`），**表多**次之（`leaf_table_count`），**位置靠前**最次。这样排序后"最该先拍平的"排最前。

2. **② 排序**：为什么相关子查询优先？因为相关子查询不能被留到"物化"或"后处理"路径，必须先决定能否拍平；而表多的先拍平，能最大化减少后续 join 的表数。

3. **③ `Prepared_stmt_arena_holder`**：这是"永久变换"的显式标记——切换 mem_root 到 PS arena（因为这是不可逆的结构改写，且 PS 只做一次，用独立 arena 避免污染语句 arena）。

4. **④⑤ 两类转换**：`CANDIDATE_FOR_DERIVED_TABLE` → 转 joined derived table；`CANDIDATE_FOR_SEMIJOIN` → 转 semi-join。后者有两个门槛：**表数不超过 `MAX_TABLES`**（否则 join 优化器处理不了）、**子查询里不能有 anti-join nest**（实现限制，见 `advance_sj_state`）。

5. **常量折叠**（`:3937-3949`）：子查询 WHERE 是纯常量且恒 false（`SELECT ... WHERE 1=0`），整个子查询直接删掉——这是"永久变换里也夹着常量折叠"的例子。

### 6.4 apply_local_transforms 与 simplify_joins 逐行剖析

> **上下文**：被 `Query_block::prepare` 第 43 步调用（`:593`），属于 **transform 阶段**，由顶层 QB 发起、递归下推。`simplify_joins` 是它的核心子步骤。

`apply_local_transforms`（`sql_resolver.cc:751`）是 prepare 末尾的"局部变换总入口"，`simplify_joins` 是它的核心子步骤：

```cpp
bool Query_block::apply_local_transforms(THD *thd, bool prune) {
  assert(first_execution);          // ★ 永久变换铁证：只允许第一次执行

  // ① 删 merged 派生表里没用的列
  if (derived_table_count) delete_unused_merged_columns(&m_table_nest);

  // ② 递归子查询（自底向上：先内层，后外层）
  for (unit = first_inner_query_expression(); unit; unit = unit->next_query_expression())
    for (qt : unit->query_terms<>())
      if (qt->query_block()->apply_local_transforms(thd, true)) return true;

  // ③ 核心：外连接转内连接 + 去括号 + join cond 上提
  if (simplify_joins(thd, &m_table_nest, true, false, &m_where_cond)) return true;

  // ④ ONLY_FULL_GROUP_BY 检查（必须在 simplify_joins 之后）
  if ((is_distinct() || is_grouped()) && (thd->variables.sql_mode & MODE_ONLY_FULL_GROUP_BY) &&
      check_only_full_group_by(thd))
    return true;

  // ⑤ 分区裁剪
  if (partitioned_table_count && prune) { prune_partitions(...); }

  // ⑥ 条件下推到派生表（只有顶层 QB 做）
  if (outer_query_block() == nullptr && push_conditions_to_derived_tables(thd)) return true;
  return false;
}
```

逐段解释：

1. **`assert(first_execution)`** —— 这一行是"transform 是永久变换"的**代码级铁证**：Prepared Statement 的 `apply_local_transforms` 只允许第一次执行，第二次执行（EXECUTE）会触发断言。这就是 2.1 阶段 D 说的"永久 vs 临时"的分界。

2. **①→② 顺序**：先删 merged 未用列，再递归子查询——**自底向上**（内层先做，外层后做），因为外层的 simplify 依赖内层已经 resolve/merge 完。

3. **③ `simplify_joins`** 是核心（见下）。它之后才有 ④⑤⑥，顺序是硬约束：注释（:774-833）说明——`simplify_joins` 把外连接转内连接，**增强了函数依赖**（weak→strong），`check_only_full_group_by` 才能正确判断；而条件下推（⑥）会替换 WHERE 里的列，**会破坏 ONLY_FULL_GROUP_BY 的检查**，所以必须排在检查之后。

4. **⑥ 只有顶层做**：`outer_query_block() == nullptr` 才下推——因为下推要从最外层开始，逐层往里。

`simplify_joins`（`sql_resolver.cc:1951`）的核心是**用 NULL-rejecting 谓词把外连接转内连接**：

```cpp
bool Query_block::simplify_joins(THD *thd, mem_root_deque<Table_ref *> *join_list,
                                 bool top, bool in_sj, Item **cond, uint *changelog) {
  for (Table_ref *table : *join_list) {
    table_map not_null_tables = table_map(0);
    if (nested_join != nullptr) {          // 嵌套 join：递归处理
      if (table->join_cond() != nullptr)
        simplify_joins(thd, &nested_join->m_tables, false, ..., &join_cond, changelog);
      ...
    } else {
      used_tables = table->map();
      if (*cond != nullptr) not_null_tables = (*cond)->not_null_tables();
    }
    // ★ 关键判定：外连接 + 内表被 NULL-rejecting 谓词覆盖 → 转内连接
    if (!table->outer_join || (used_tables & not_null_tables)) {
      // 转 inner join：清 outer_join 标志，join cond 上提到 WHERE/上层
    }
  }
}
```

**核心思想**：外连接 `A LEFT JOIN B ON JC` 中，B 可能被 NULL-complemented。但若存在谓词 W 使得"W 为真 ⟹ B 的某列非 NULL"，则 B 实际**不可能被 NULL 补全**，LEFT JOIN 等价于 INNER JOIN，可以转换。`(*cond)->not_null_tables()` 正是计算"哪些表被 NULL-rejecting 谓词覆盖"——`used_tables & not_null_tables` 命中就转换。这是外连接消除的经典算法（Galindo-Legaria 的 join 化简理论）。

---

## 七、derived table：merge 还是 materialize

**merge 的入口在 `sql_resolver.cc:3462`**（不是 `sql_derived.cc`）：

```cpp
// sql_resolver.cc:3465-3518（判定节选）
  if (!derived_table->is_view_or_derived() || derived_table->is_merged())
    return false;                                                   // :3465
  ...
  if (derived_table->algorithm == VIEW_ALGORITHM_TEMPTABLE ||
      !derived_query_expression->is_mergeable())
    return false;                                                   // :3493
```

**materialize 的触发条件**：`ALGORITHM=TEMPTABLE`、不可 merge（UNION / 聚合 / DISTINCT / GROUP BY / LIMIT / 窗口 / select list 含子查询）、hint `NO_MERGE`、`derived_merge=off`、STRAIGHT_JOIN+sj nest、表数超 `MAX_TABLES`。

- merge 成功后：`set_merged()`(:3520) → `merge_underlying_list`(:3538) → `merge_where`(:3614) → **`create_field_translation`**(:3616，derived 列 → 底层表达式映射) → `exclude_level`(:3620) → `remap_tables`(:3631)
- materialize：`Table_ref::setup_materialized_derived`（`sql_derived.cc:808`）→ `setup_materialized_derived_tmp_table`(:820)

### 7.1 merge_derived 逐行剖析

> **上下文**：被 `resolve_placeholder_tables`（`Query_block::prepare` 第 7 步，`:248`）调用，属于 **resolve 阶段**（不是 transform）——虽然它会永久改写结构，但触发点是"派生表定义"的解析。

`merge_derived`（`sql_resolver.cc:3462`）把可合并的派生表"拍平"进外层查询。它的核心不是"把表列表接上来"，而是**建立字段映射**：

```cpp
bool Query_block::merge_derived(THD *thd, Table_ref *derived_table) {
  // ── 判定链：能不能 merge ──────────────────────────
  if (!derived_table->is_view_or_derived() || derived_table->is_merged())
    return false;                                          // :3465 已 merge 过 / 不是 derived
  if (derived_table->algorithm == VIEW_ALGORITHM_TEMPTABLE ||
      !derived_query_expression->is_mergeable())
    return false;                                          // :3493 显式 TEMPTABLE / 不可 merge
  if (derived_table->algorithm == VIEW_ALGORITHM_UNDEFINED) {
    bool merge_heuristic = (is_view() || allow_merge_derived) &&
                           derived_query_expression->merge_heuristic(lex);
    if (!hint_table_state(thd, derived_table, DERIVED_MERGE_HINT_ENUM,
                          merge_heuristic ? ... : 0))
      return false;                                        // :3501 hint 说 NO_MERGE
  }
  if ((active_options() & SELECT_STRAIGHT_JOIN) &&
      (derived_query_block->has_sj_nests || has_aj_nests))
    return false;                                          // :3512 STRAIGHT_JOIN 不能 merge sj nest
  if (leaf_table_count + derived_query_block->leaf_table_count - 1 > MAX_TABLES)
    return false;                                          // :3517 表数超限

  // ── 执行 merge ────────────────────────────────────
  derived_table->set_merged();                             // :3520 打标
  // ① 底层表列表接上来
  derived_table->merge_underlying_list = derived_query_block->get_table_list();
  // ② 字段映射（核心！）—— derived 列 → 底层表达式
  create_field_translation(...);                           // :3616
  // ③ 合并 WHERE
  merge_where(...);                                        // :3614
  // ④ 重映射表引用
  remap_tables(...);                                       // :3631
}
```

逐段解释：

1. **判定链（`:3465-3517`）**：五道关卡——已 merge 过 / 显式 `ALGORITHM=TEMPTABLE` / hint `NO_MERGE` / `STRAIGHT_JOIN`+sj nest / 表数超 `MAX_TABLES`。任何一道不过就放弃 merge，走物化。这是"代价最小、收益最大"的决策——merge 让 join 优化器能自由选择连接顺序（收益大），但有实现限制（表数上限、sj 冲突）。

2. **② `create_field_translation`（核心）**：这是 merge 的真正难点。derived 表对外暴露一组"输出列"（如 `SELECT a+b AS c, d*2 AS e FROM t`），外层引用的是 `c`/`e`。merge 之后 `c`/`e` 这个"层"消失了，外层对 `c` 的引用必须**重写成底层表达式 `a+b`**。`create_field_translation` 建立的正是这个映射表（`derived 列 → 底层 Item 表达式`），供 ④ `remap_tables` 把外层所有引用 `c` 的 `Item_field` 替换成 `a+b`。

3. **③④ 顺序**：先 `merge_where`（把 derived 的 WHERE 合并进外层），再 `remap_tables`（重写引用）——因为重写时 WHERE 里的引用也要一起 remap。

4. **为什么这是"永久变换"的典型**：merge 之后 derived 表对象还在（`Table_ref` 保留作标记），但它的查询块已被"吸收"进外层，**再也无法拆回来**——所以它和 semi-join flatten、simplify_joins 一样，是 `assert(first_execution)` 的永久改写。

---

## 八、窗口函数 setup

> ⚠️ 文件位置：**`sql/window.cc`**（不是 `sql_window.cc`）。

| 函数 | 位置 | 阶段 |
|------|------|------|
| `Window::setup_windows1` | `window.cc:1090` | **prepare 阶段**（调用点 `sql_resolver.cc:408`） |
| `Window::setup_windows2` | `window.cc:1308` | **optimize 阶段**（`sql_optimizer.cc:370`，只做 `check_border_sanity2`，因为 PS 二次执行时边界可能是 `?` 参数） |

`setup_windows1` 做的事：窗口数上限检查 → 解析 PARTITION/ORDER（`:1112`）→ **构建窗口继承邻接表并检查环**（`:1132`，`ER_WINDOW_CIRCULARITY_IN_WINDOW_GRAPH`）→ SQL2011 SR 10.c/d/e 检查（`:1172`）→ `check_border_sanity1`。

---

## 九、prepare 之后才存在的东西

（这些是第 ③ 层 IR 相对第 ② 层的"增量"，也是判断"是否已 prepare"的依据）

| 产物 | 位置 |
|------|------|
| `Query_block::base_ref_items`（`Ref_item_array`） | 分配于 `setup_base_ref_items` `sql_lex.cc:2486`；是"第 i 列 → fixed Item*"的权威索引 |
| `fields` 里的 **hidden items** + `hidden_group_field_count` / `hidden_order_field_count` | `sql_resolver.cc:303` / `:376` |
| 每个 `Item` 的 `fixed=true` + `Item_field::field/field_index/table_ref` | `set_field` `item.cc:2881` |
| **semi-join / antijoin nest** + `sj_candidates` | `flatten_subqueries` 后；`sql_lex.h:2468` |
| **`leaf_tables` / `leaf_table_count` / `Table_ref::map` / `tableno`** | `setup_tables` `sql_resolver.cc:1169` |
| `Window` 内部状态 | `setup_windows1` |
| 关联标记（`depended_from` / `UNCACHEABLE_DEPENDENT` / `OUTER_REF_TABLE_BIT`） | 第五节 |
| `Query_expression::prepared = true` | `set_prepared` `sql_lex.h:1158`，调用点 `sql_union.cc:874` |

**反例**（不是 prepare 产物，用于对照）：`JOIN` 对象（`sql_resolver.cc:183` 有 `assert(join == nullptr)`，它在 optimize 阶段才创建）、`AccessPath` 树、`sql_const_folding.cc` 的结果 —— 都是 optimize 产物。

---

## 核心调用栈

```
Sql_cmd_dml::execute()                         sql/sql_select.cc:675
 └─ prepare(thd)                               sql/sql_select.cc:495
     ├─ precheck()      :531（粗粒度权限）
     ├─ open_tables_for_query()  :542
     └─ prepare_inner() :568
         └─ Query_expression::prepare()        sql/sql_union.cc:694
             └─ for (sl : first_query_block..) {
                   sl->prepare()               sql/sql_union.cc:757
               }
                 └─ Query_block::prepare()     sql/sql_resolver.cc:179
                     ├─ setup_tables()         :246
                     ├─ resolve_placeholder_tables()  :248
                     ├─ setup_wild()           :278
                     ├─ setup_base_ref_items() :279
                     ├─ setup_fields()         :281
                     │    └─ Item_field::fix_fields()   item.cc:5690
                     │         └─ find_field_in_tables()  sql_base.cc:7919
                     ├─ setup_conds()          :298（WHERE + ON）
                     ├─ setup_group()          :302
                     ├─ setup_order()          :365
                     ├─ Window::setup_windows1():408   window.cc:1090
                     ├─ resolve_subquery()     :446
                     ├─ flatten_subqueries()   :549
                     └─ apply_local_transforms():559（simplify_joins + prune_partitions）
```

---


## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → SELECT Statement*（名字解析与列引用规则）
- *MySQL 8.0 Reference Manual → Optimization*（prepare 阶段的改写概览）

**内核月报**
- 2024/04《MySQL 查询优化分析 - 基础概念》—— prepare 与 optimize 的阶段边界

**阿里云 PolarDB 官方文档**
- [《MySQL 8.0 Server 层架构的查询解析与优化原理》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/architecture-of-mysql-server-in-mysql-8) —— prepare/rewrite 阶段总览（setup+fix+transform 函数清单）
- [《图解 MySQL 8.0 优化器内部查询解析机制》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/optimizer-based-query-resolution-in-mysql-8) —— Setup and Resolve 详解（`propagate_nullability`、`merge_derived`、`setup_fields`、`resolve_rollup`、`remove_redundant_subquery_clauses`、`Window::setup_windows1` 等，本篇"三/四/六/七/八"逐行剖析对齐此文）
- [《图解 MySQL 8.0 优化器对子查询 JOIN 与分区表的转换优化》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/optimizer-based-query-conversion-in-mysql-8) —— Transformation 详解（`resolve_subquery`、`flatten_subqueries`、`convert_subquery_to_semijoin` 四种模式、`apply_local_transforms`、`push_conditions_to_derived_tables`，本篇"六"逐行剖析对齐此文）

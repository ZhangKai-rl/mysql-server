# 02 子查询：运行期执行策略

> **一句话**：子查询的执行策略不是一组并列的选项，而是**"嵌套迭代 ↔ 去关联化"之间的一条谱系**——从"外层每行重跑内层"（IN2EXISTS，最不去关联化），到"内层跑一次存起来"（物化），到"彻底变成 join"（semi-join 扁平化，最去关联化）。选哪一档取决于**语义可行性**与**代价**，本篇讲每一档在运行期到底怎么执行。

> **边界**：本篇讲**运行期执行**。优化器侧的策略选择与改写（为什么选这个策略、semi-join 的准入条件、IN2EXISTS 与物化的代价比较）见 [`../07_optimize/logical/02_subquery.md`](../07_optimize/logical/02_subquery.md) 与 [`../07_optimize/logical/03_semijoin.md`](../07_optimize/logical/03_semijoin.md)；物化机制本身见 [`08_materialization.md`](08_materialization.md)；迭代器体系见 [`../09_executor_iterator.md`](../09_executor_iterator.md)。

## 目录

- [概述](#概述)
  - 是什么
  - 用途
  - 版本演进
- [理论基础](#理论基础)
  - 为什么策略是一条"去关联化谱系"
  - 为什么 semi-join 需要五种策略：去重机制的成本谱系
  - 为什么 IN2EXISTS 与物化必须二选一
  - 为什么物化之后还要后过滤
  - trig_cond：为什么需要"可以不算数的条件"
- [核心实现](#核心实现)
  - Subquery_strategy 与运行期落地
  - 物化执行：subselect_hash_sj_engine
  - IN2EXISTS 执行：Item_in_optimizer
  - semi-join 五种策略的运行期
  - 标量子查询
  - EXISTS 子查询
  - 子查询缓存
  - 深潜：物化引擎的建表 / 迭代器 / trig_cond
- [可观测性](#可观测性)
- [Misc](#misc)
  - ★ 本机制里的工程实现技法
  - 失效与退化场景
  - 易混淆点
- [参考](#参考)

---

## 概述

### 是什么

子查询在 prepare 阶段被改写成几种形态，运行期对应不同执行路径：

```
IN (SELECT ...)
 ├─ 扁平化成 semi-join ──▶ JOIN 执行器 5 种策略（FirstMatch/LooseScan/Weedout/Materialize）
 └─ 未扁平化
     ├─ IN2EXISTS ──▶ 外层每行重跑内层 EXISTS + LIMIT 1
     └─ 物化 ──────▶ 首次物化去重临时表，之后逐行 ref 查找
标量子查询 ──▶ 每次 exec()，结果存 Item_cache
EXISTS ──────▶ 一行即停短路
```

★ **关键区分两套"物化"**（别混淆）：semi-join 的 `MaterializeAccessPath` 物化的是**扁平化后的 nest**；`subselect_hash_sj_engine` 物化的是**未扁平化的 IN 子查询**。两者共用 `MaterializeIterator`，但触发时机与语义完全不同。

### 用途

本篇回答：一个子查询在 EXPLAIN 里显示 `FirstMatch` / `LooseScan` / `Materialize` / `DEPENDENT SUBQUERY` 时，**运行期到底在做什么**、每行数据是怎么流动、哪里可能成为性能瓶颈。

### 版本演进

| 版本 | 变化 | 为什么 |
|------|------|--------|
| 5.6 | **semi-join 五种策略 + 子查询物化**引入 | 此前 `IN (SELECT...)` 只能 IN2EXISTS（外层每行重跑），大表上极慢 |
| 5.6~5.7 | anti-join（`NOT IN`）优化、`subselect_indexsubquery_engine`（把可转 ref 的子查询直接走索引） | 继续压缩"重跑内层"的场景 |
| **8.0** | 执行器**迭代器化**：策略从 `QEP_TAB` 上的执行改为 AccessPath → RowIterator（如 `NestedLoopIterator` 的 `JoinType::SEMI`、`WeedoutIterator`、`RemoveDuplicatesOnIndexIterator`） | 与优化器重构配套，策略语义不变但载体全换 |
| 8.0.39（本库基线） | `subselect_hash_sj_engine::create_iterators` 的三个反直觉设计（永不用 EQRef/Alternative、唯一键补 LIMIT 1、伪装成 derived table） | 见「深潜」 |
| 8.4 / 9.x | 本库未跟踪到子查询运行期策略的结构性变化 | —— |

---

## 理论基础

### 为什么策略是一条"去关联化谱系"

子查询的字面语义是**嵌套迭代**（nested iteration）：对外层每一行，重新执行一次内层查询。代价是 O(外层行数 × 内层单次成本)。

关系代数的核心洞察是：**大多数子查询可以"去关联化"（unnesting / decorrelation）成 join**，把 O(n×m) 变成 O(n+m) 或带索引的 O(n log m)。

MySQL 的策略正好落在这条谱系的三档：

| 档位 | 策略 | 内层执行次数 | 语义要求 |
|---|---|---|---|
| **不去关联化** | IN2EXISTS | **外层每行一次** | 无（最通用，任何子查询都能这么跑） |
| **半去关联化** | 物化（`subselect_hash_sj_engine`） | **一次**（存进临时表，外层查表） | 内层不依赖外层（非相关），且可以接受"去重" |
| **完全去关联化** | semi-join 扁平化 | **零次**（变成 join 的一部分） | 最严格：必须是 `IN/=ANY`、无聚合/LIMIT、能改写成半连接 |

★ **约束**：越往右越快，但**语义要求越严**。所以优化器的工作就是"尽可能往右推，推不动就往左退一档"——这也是为什么 `EXPLAIN` 里同时存在 `MATERIALIZED`、`DEPENDENT SUBQUERY` 和普通的 join 计划。

### 为什么 semi-join 需要五种策略：去重机制的成本谱系

半连接的语义是"外层行只要匹配到**至少一个**内层行就输出一次"。难点在**去重**——普通 join 会让一个外层行因多个匹配产生多份输出。五种策略本质是**五种去重机制**，按"依赖什么前提"与"成本"排列：

| 策略 | 去重机制 | 依赖前提 | 成本 |
|---|---|---|---|
| **FirstMatch** | 找到第一个匹配就**短路**回外层 | 内层能被快速定位（有索引） | 最低——不产生多余行 |
| **LooseScan** | 内层按索引**有序**输出，跳过相邻重复 key | 内层索引有序 | 低——只比较多一行 |
| **Materialize** | 内层物化成去重临时表 | 内层不依赖外层 | 中——建表 + 查找 |
| **Duplicate Weedout** | 先当**内连接**跑，再按外层 rowid 组合去重 | **无前提** | 高——临时表 + 每行写入判重 |
| （MaterializeScan） | 物化后全扫连接 | 同上 | 中高 |

★ 排布规律：**越靠上的策略越依赖索引/有序性（前提强、成本低），越靠下越通用（前提弱、成本高）**。Weedout 之所以存在，就是因为它是唯一"什么前提都不需要"的兜底——当内层既无索引可短路、又不能物化时，只能先当内连接跑完再去重。

### 为什么 IN2EXISTS 与物化必须二选一

两者都能处理"未扁平化的 IN 子查询"，成本模型完全不同：

| | IN2EXISTS | 物化 |
|---|---|---|
| 内层执行 | **外层每行一次**（带 `LIMIT 1` 短路） | **一次**全跑完 |
| 主要成本 | 外层行数 × 单次索引查找 | 内层一次全跑 + 建临时表 + 外层每行 ref 查找 |
| 适合 | 外层行数**少**、内层有**好索引** | 外层行数**多**、内层**大**或索引差 |

★ 为什么不能"总是物化"：物化要建临时表（可能落盘）、要写全部内层行，当外层只有几行时这完全是浪费。反之，外层有百万行时，每行跑一次内层就是百万次索引查找。**判据是"外层行数 × 单次成本" vs "物化固定成本"**——优化器算完二选一，由 `finalize_exists_transform()` / `finalize_materialization_transform()` 定死（`CANDIDATE_FOR_IN2EXISTS_OR_MAT` 是"双重候选"状态）。

### 为什么物化之后还要后过滤

物化表的索引查找是**近似的**，两处假阳性必须靠精确比较兜底：

1. **截断假阳性**：索引按前缀比较时，VARCHAR(10) 列里 `'abcdefghij'` 与 `'abcdefghijXYZ'` 可能被判等；
2. **哈希碰撞**：`hash_field` 模式下不同值可能算出相同 hash。

所以 `subselect_hash_sj_engine::setup()` 在物化后还要逐列造 `Item_func_eq` 加进 `Item_cond_and` 做后过滤。★ 这是"**用便宜的近似查找做剪枝、用精确比较做确认**"的两阶段模式——与 bloom filter 的思路同构。

### trig_cond：为什么需要"可以不算数的条件"

IN2EXISTS 把 `oe IN (SELECT ie ...)` 改写成 `EXISTS(SELECT 1 ... WHERE oe = ie ...)`，把外层值注入内层谓词。问题来了：**当 `oe` 为 NULL 时，这个谓词不应该生效**（`NULL IN (...)` 是 UNKNOWN，不能靠 `oe = ie` 判断）。

但 Item 树一旦构造就是静态的，运行期无法"把某个条件摘掉"。MySQL 的解法是 **trig_cond（触发条件）**：

```cpp
// item_cmpfunc.cc
return *trig_var ? args[0]->val_int() : 1;   // 守卫变量为真才求值，否则当作真（1）
```

守卫变量 `*trig_var` 为真时正常求值；为假时**返回 1（当作真）**——即"这个条件暂时不算数"。★ 这是"**用数据（guard 变量）控制静态表达式树的求值**"的经典技巧：不需要重建 Item 树，只需改一个 bool。

---

## 核心实现

### Subquery_strategy 与运行期落地

#### 0. 先看纠正表（8.0.39 实测）

| 常见说法 / 网上资料 | 8.0.39 真相 |
|---|---|
| `class subselect_engine` 抽象基类 + `enum enum_engine_type` + `cols()/set_row()` | **已彻底删除**（全树 0 定义，只在 mysql-test 的**过期结果文件**里作为文本残留） |
| `subselect_single_select_engine` / `subselect_union_engine`（含 `Query_result_union` 内部类） | **已删除**；UNION 子查询统一走 `Query_expression::execute()` |
| `subselect_rowid_intersect_engine` | **从未存在**（误传） |
| `enum class SubqueryExecMethod` + 成员 `Item_subselect::exec_method` | **不存在**；全树唯一 1 处命中是 `sql/sql_resolver.cc:1412` 的一条 `@todo` 注释。真实枚举是 `Item_exists_subselect::strategy`（`Subquery_strategy`） |
| `Item_subselect::get_engine_for_execution()` | **不存在**；只有成员 `subselect_indexsubquery_engine *indexsubquery_engine` |
| `Item_cache_wrapper` 类 | **不存在**；缓存是原生 `Item_cache*` 数组 + `Item_in_optimizer::cache` |
| `subselect_hash_sj_engine` 已被 `MaterializeIterator` 取代 | **类仍在使用**：物化动作下放给 `MaterializeIterator`，engine 退化为"持有 iterator + 判空短路 + NULL 语义修正"的壳 |
| `Item_in_subselect::val_bool()` 是求值入口 | **不是**：`val_int/val_real/val_str/val_decimal/val_bool` 全部 `assert(false)`，真实入口是 `val_bool_naked()` |
| `st_bits` 位图去重 | **不存在**；去重是临时表上的**唯一索引**（或 `hash_field` 哈希去重） |
| `m_count` 计数短路 | **不存在**；短路用 `has_zero_rows` 与 `Item_singlerow_subselect::no_rows` |
| `Item_in_subselect::is_correlated()` | **不存在**；用 `in2exists_info->dependent_before/after` 判断 |

#### 0.1 `Item_subselect` 基类（`sql/item_subselect.h:83`，执行相关成员）

```cpp
class Item_subselect : public Item_result_field {
 private:
  bool value_assigned; /* value already assigned to subselect */
  // ★ 执行期"结果是否已落地"的唯一标志；exec() 前由 ExecuteExistsQuery/ClearForExecution 清 0
  /**
      Whether or not execution of this subselect has been traced by
      optimizer tracing already. If optimizer trace option
      REPEATED_SUBSELECT is disabled, this is used to disable tracing
      after the first one.
  */
  bool traced_before;   // ★ 相关子查询每行都 exec() ⇒ 靠它避免 optimizer_trace 被刷爆

 public:
  Item *substitution;      // ★ 改写产物（如 Item_in_optimizer），fix_fields() 里 *ref = substitution

  /* unit of subquery */
  Query_expression *unit;  // ★★ 执行期的真正主角：子查询 = 一个 Query_expression（不是 JOIN）

  int in_cond_of_tab;

  // For EXPLAIN.
  enum enum_engine_type { OTHER_ENGINE, INDEXSUBQUERY_ENGINE, HASH_SJ_ENGINE };
  enum_engine_type engine_type() const;

 protected:
  void init(Query_block *select, Query_result_subquery *result);
  /// The inner part of the subquery.
  unique_ptr_destroy_only<SubqueryWithResult> subquery;   // ★ 非 engine 路径的执行载体

  /// Only relevant for Item_in_subselect; optimized structure used for
  /// execution in place of running the entire subquery.
  subselect_indexsubquery_engine *indexsubquery_engine = nullptr;  // ★★ 唯一的"执行方式"开关

  table_map used_tables_cache;   // ★ 决定 const_item()
  uint max_columns;
  enum_parsing_context parsing_place;

 public:
  enum subs_type { UNKNOWN_SUBS, SINGLEROW_SUBS, EXISTS_SUBS, IN_SUBS,
                   ALL_SUBS, ANY_SUBS };

  void cleanup() override;
  virtual void reset() { null_value = true; }
  virtual bool select_transformer(THD *thd, Query_block *select) = 0;
  bool assigned() const { return value_assigned; }
  void assigned(bool a) { value_assigned = a; }
  virtual bool exec(THD *thd);                       // ★ 执行期总入口（虚）
  table_map used_tables() const override { return used_tables_cache; }
  table_map not_null_tables() const override { return 0; }
  void update_used_tables() override;
  void set_indexsubquery_engine(subselect_indexsubquery_engine *eng) {
    indexsubquery_engine = eng;   // ★ 由 JOIN::replace_index_subquery() / finalize_materialization_transform() 注入
  }
  bool is_evaluated() const;
  bool is_uncacheable() const;
};
```

**`Item_subselect::exec()` 完整**（`sql/item_subselect.cc:634`）——两种执行路径的分派点：

```cpp
bool Item_subselect::exec(THD *thd) {
  /* Do not execute subselect in case of a fatal error or if the query has been killed. */
  if (thd->is_error() || thd->killed) return true;         // ① 快速失败

  assert(!thd->lex->is_view_context_analysis());
  DBUG_EXECUTE_IF("subselect_exec_fail", return true;);

  /*
    Disable tracing of subquery execution if
    1) this is not the first time the subselect is executed, and
    2) REPEATED_SUBSELECT is disabled
  */
  Opt_trace_context *const trace = &thd->opt_trace;
  const bool disable_trace =                                // ② 第 2 次起关掉 trace
      traced_before &&
      !trace->feature_enabled(Opt_trace_context::REPEATED_SUBSELECT);
  Opt_trace_disable_I_S disable_trace_wrapper(trace, disable_trace);
  traced_before = true;

  Opt_trace_object trace_wrapper(trace);
  Opt_trace_object trace_exec(trace, "subselect_execution");
  trace_exec.add_select_number(unit->first_query_block()->select_number);
  Opt_trace_array trace_steps(trace, "steps");

  // subselect_hash_sj_engine creates its own iterators; it does not call exec().
  bool should_create_iterators =                             // ③ 物化引擎自己造 iterator
      !(indexsubquery_engine != nullptr &&
        indexsubquery_engine->engine_type() ==
            subselect_indexsubquery_engine::HASH_SJ_ENGINE);

  /*
    Normally, the unit would be optimized here, but statements like DO and SET
    may still rely on lazy optimization. Also, we might not have iterators,
    so make sure to create them if they're missing.
  */
  if (!unit->is_optimized()) {                               // ④ "懒优化"：DO/SET 里的子查询此刻才 optimize
    if (unit->optimize(thd, /*materialize_destination=*/nullptr,
                       /*create_iterators=*/false,
                       /*finalize_access_paths=*/false))
      return true;
    /*
      NOTE: We defer finalization and creating iterators to the statements
      below; asking optimize() to finalize a plan with two choices
      (materialization and exists) rightfully causes a failed assertion. ...
      we will always implicitly choose in2exists in these cases (DO/SET).
    */
  }
  if (should_create_iterators && unit->root_access_path() != nullptr) {
    if (unit->finalize(thd)) return true;
    if (unit->force_create_iterators(thd)) return true;
  }
  if (indexsubquery_engine != nullptr) {                     // ⑤★ 二分派
    return indexsubquery_engine->exec(thd);                  //   engine 路径（index lookup / 物化）
  } else {
    return subquery->exec(thd);                              //   Query_expression 路径
  }
}
```

**"含子查询的 Item 永远不 const"的机制**（`sql/item_subselect.cc:213` / `:277` / `:824`）：

```cpp
void Item_subselect::accumulate_expression(Item *item) {
  if (item->used_tables() & ~OUTER_REF_TABLE_BIT)
    m_subquery_used_tables |= INNER_TABLE_BIT;   // ★ 子查询里只要有表/表达式就置 INNER_TABLE_BIT
  set_nullable(is_nullable() || item->is_nullable());
}

void Item_subselect::update_used_tables() {
  if (!unit->uncacheable) {
    // There is no expression with outer reference, randomness or side-effect,
    // so the subquery's content depends only on its inner tables:
    m_subquery_used_tables &= INNER_TABLE_BIT;
  }
  used_tables_cache = m_subquery_used_tables;    // ★ used_tables() != 0 ⇒ const_item()==false
}
```

`RAND_TABLE_BIT` 不是子查询自带，而是从 `unit->uncacheable & UNCACHEABLE_RAND` **主动传染**（`:586`）：

```cpp
  if ((uncacheable = unit->uncacheable)) {
    if (uncacheable & UNCACHEABLE_RAND) {
      m_subquery_used_tables |= RAND_TABLE_BIT;
      used_tables_cache |= RAND_TABLE_BIT;   // ★ 子查询含 RAND() ⇒ 父 Item 也带 ⇒ 永不被当 const 折叠
    }
  }
```

#### 0.2 无 engine 时的默认路径：`SubqueryWithResult`（`sql/item_subselect.h:736`）

```cpp
class SubqueryWithResult {
 public:
  SubqueryWithResult(Query_expression *u, Query_result_interceptor *res,
                     Item_subselect *si);
  void cleanup();
  bool prepare(THD *thd);
  bool exec(THD *thd);
  ...
 private:
  Query_result_interceptor *result;   // ★ 结果 sink
  Item_subselect *item;
  enum Item_result res_type;
  enum_field_types res_field_type;
  bool maybe_null;
  Query_expression *unit;
  void set_row(const mem_root_deque<Item *> &item_list, Item_cache **row,
               bool never_empty);
};

SubqueryWithResult::SubqueryWithResult(Query_expression *u,
                                       Query_result_interceptor *res,
                                       Item_subselect *si)
    : result(res), item(si), ..., unit(u) {
  unit->item = si;    // ★★ 反向指针：Query_expression::item → 这个 Item_subselect
}

bool SubqueryWithResult::exec(THD *thd) {
  assert(unit->is_optimized());
  char const *save_where = thd->where;
  const bool res = unit->execute(thd);    // ★ 真正的执行交给 Query_expression
  thd->where = save_where;
  return res;
}
```

结果"下沉"的三个 sink（都在 `item_subselect.cc` 内，**不是**已删除的 engine 内部类）：

```cpp
// sql/item_subselect.cc:859 —— 标量子查询：多于一行的错误在这里报
bool Query_result_scalar_subquery::send_data(
    THD *thd, const mem_root_deque<Item *> &items) {
  Item_singlerow_subselect *it = down_cast<Item_singlerow_subselect *>(item);
  if (it->assigned()) {
    my_error(ER_SUBQUERY_NO_1_ROW, MYF(0));   // ★ 第二次 send_data ⇒ 有第 2 行
    return true;
  }
  uint i = 0;
  for (Item *val_item : VisibleFields(items)) {
    it->store(i++, val_item);                 // 逐列存进 Item_cache
  }
  if (thd->is_error()) return true;
  it->assigned(true);
  return false;
}

// :1319 —— EXISTS：找到任意一行 ⇒ true
bool Query_result_exists_subquery::send_data(THD *, const mem_root_deque<Item *> &) {
  Item_exists_subselect *it = (Item_exists_subselect *)item;
  /* A subquery may be evaluated 1) by executing the JOIN 2) by optimized
     functions (index_subquery, subquery materialization). It's only in (1)
     that we get here when we find a row. In (2) "value" is set elsewhere. */
  it->value = true;
  it->assigned(true);
  return false;
}
```

公共工具 `ExecuteExistsQuery()`（`sql/item_subselect.cc:3086`）——"只读一行就停"：

```cpp
bool ExecuteExistsQuery(THD *thd, Query_expression *unit, RowIterator *iterator,
                        bool *found) {
  ...
  if (unit->ClearForExecution()) return true;   // ★ 相关子查询每次重算前都要清干净
  unit->set_executed();
  thd->get_stmt_da()->reset_current_row_for_condition();
  if (iterator->Init()) return true;            // ★ Init 而不是 execute：只要一行
  int error = iterator->Read();
  if (error == 1 || thd->is_error()) return true;
  *found = (error == 0);
  return false;
}
```

⇒ **非相关子查询"只算一次"的机制**在 `Query_expression::execute()`（`sql/sql_union.cc:1813`）：`if (is_executed() && !uncacheable) return false;`

### Subquery_strategy 与运行期落地（小结）

`Subquery_strategy`（`item_subselect.h`）枚举与运行期对应：

| 策略 | 运行期怎么执行 |
|------|---------------|
| `SUBQ_EXISTS` | `Item_exists_subselect::val_bool()` → `exec()` → 内层重跑 |
| `SUBQ_MATERIALIZATION` | `subselect_hash_sj_engine::exec()`：首次物化 + ref 查找 |
| `SEMIJOIN` | 已扁平化，走 JOIN 执行器的五种策略 |
| `DERIVED_TABLE` | 变成带 `MaterializeIterator` 的派生表 |
| `DELETED` | 表达式树已移除（恒假等） |

### 物化执行：subselect_hash_sj_engine

**临时表建在哪**（`setup()`）：用 `Query_result_union::create_result_table` 建**去重临时表**（`is_union_distinct=true`——IN 语义里重复值不影响结果），并建唯一/哈希索引。然后为每列构造 `store_key`（把外层左表达式值 encode 成查找 key），并生成后过滤条件 `tmp_table.col == left_expr`。

**exec 的惰性物化 + ref 查找**：

```cpp
if (!is_materialized) {                        // 1. 首次：惰性物化
  bool error = m_iterator->Init();             //   MaterializeIterator 物化整个内层
  is_materialized = true;
  table->file->info(HA_STATUS_VARIABLE);
  has_zero_rows = (stats.records == 0);        // 2. 0 行短路
}
if (has_zero_rows) { item->value = false; return false; }
if (item->left_expr->element_index(0)->null_value) {  // 3. 左 NULL 短路
  item->value = true; return false;            //   结果 UNKNOWN，由上层翻译
}
bool found;
ExecuteExistsQuery(thd, item->unit, m_iterator.get(), &found);  // 4. ref 查找，只读一行
item->value = found;
if (!found && mat_table_has_nulls != NEX_IRRELEVANT_OR_FALSE) {
  /* 5. 未命中但内层可能有 NULL：查一次 NULL 行，结果 UNKNOWN */
}
```

五个要点：

1. **惰性物化**：首次求值才 `Init()` 触发 `MaterializeIterator`——若外层一行都没有，物化根本不发生；
2. **0 行短路**：物化表空则 IN 恒 FALSE；
3. **左 NULL 短路**：`oe IS NULL` 时 IN 是 UNKNOWN；
4. **ref 查找**：`ExecuteExistsQuery` 只读一行（`RefAccessPath + Filter`），命中即 true；
5. **NULL 补查**：未命中但内层含 NULL 时结果是 UNKNOWN 而非 FALSE。

`has_zero_rows` 的判定有个细节：`HA_STATS_RECORDS_IS_EXACT` 时直接用 `stats.records`，否则**真起一个 `TableScanIterator` 读一行**才能确认是否为空。

### IN2EXISTS 执行：Item_in_optimizer（完整源码）

**`Item_in_subselect::exec()`**（`sql/item_subselect.cc:753`）——比基类多了"左值没变就复用"：

```cpp
bool Item_in_subselect::exec(THD *thd) {
  assert(strategy != Subquery_strategy::SUBQ_MATERIALIZATION ||
         indexsubquery_engine->engine_type() ==
             subselect_indexsubquery_engine::HASH_SJ_ENGINE);
  /*
    Initialize the cache of the left predicate operand. This has to be done as
    late as now, because Cached_item directly contains a resolved field (not
    an item, and in some cases (when temp tables are created), these fields
    end up pointing to the wrong field. ...
    TODO: the cache should be applied conditionally based on: rules ... or on
    a cost-based basis ...
  */
  if (need_expr_cache && !left_expr_cache &&
      strategy == Subquery_strategy::SUBQ_MATERIALIZATION &&
      init_left_expr_cache(thd))
    return true;                        // ★ 只有"物化策略"才建左值缓存（IN→EXISTS 不需要）

  if (left_expr_cache != nullptr) {
    const int result = update_item_cache_if_changed(*left_expr_cache);
    if (left_expr_cache_filled &&  // cache was previously filled
        result < 0)  // new value is identical to previous cached value
    {
      /* We needn't do a full execution, can just reuse "value", "was_null",
         "null_value" of the previous execution. */
      return false;                     // ★★ 左值没变 ⇒ 直接复用上次结果，不重跑子查询
    }
    left_expr_cache_filled = true;
  }

  const bool uncacheable = unit->uncacheable || indexsubquery_engine != nullptr;
  if (unit->is_executed() && uncacheable) {
    /* Second or later execution ... so clear out state from the previous run(s).
       Note that subselect_hash_sj_engine and subselect_indexsubquery_engine
       are both uncacheable (due to dependency on the left side), even if the
       underlying unit is not marked as such. */
    null_value = false;                 // ★ 清上一轮的 UNKNOWN 痕迹
    was_null = false;
  }
  return Item_subselect::exec(thd);
}
```

**`Item_in_optimizer` 类**（`sql/item_cmpfunc.h:492`）：

```cpp
class Item_in_optimizer final : public Item_bool_func {
 private:
  Item_cache *cache{nullptr};           // ★ 左表达式的缓存（min/max 改写时也复用它）
  /**
    Stores the value of "NULL IN (SELECT ...)" for uncorrelated subqueries:
      UNKNOWN - "NULL in (SELECT ...)" has not yet been evaluated
      FALSE   - result is FALSE
      TRUE    - result is NULL
  */
  int result_for_null_param{UNKNOWN};   // ★ 非相关子查询的"NULL 左值"结果记忆

 public:
  Item_in_optimizer(Item_in_subselect *item)
      : Item_bool_func(pointer_cast<Item *>(item)) { set_subquery(); }
  bool fix_fields(THD *, Item **) override;
  bool fix_left(THD *thd, Item **ref);
  longlong val_int() override;
  void cleanup() override;
  Item_cache **get_cache() { return &cache; }
};
```

**`Item_in_optimizer::val_int()` 完整**（`sql/item_cmpfunc.cc:2338`）——IN 三值逻辑的总闸：

```cpp
longlong Item_in_optimizer::val_int() {
  assert(fixed);
  Item_in_subselect *const subqpred = down_cast<Item_in_subselect *>(args[0]);

  cache->store(subqpred->left_expr);    // ① 先把左值算一遍存进 Item_cache（避免子查询里反复求值）
  cache->cache_value();                 // ② 立刻物化缓存值（不是惰性）

  if (cache->null_value) {
    /* We're evaluating "<outer_value_list> [NOT] IN (SELECT ...)" where one or
       more of the outer values is NULL. */
    if (subqpred->abort_on_null) {
      /* We're evaluating a top level item ... a NULL value in the
         outer_value_list means that the result shall be NULL/FALSE (makes no
         difference for top level items). The cached value is NULL, so just
         return NULL. */
      null_value = true;                // ③ 顶层 IN：左值含 NULL ⇒ 直接 NULL，不执行子查询
    } else {
      /* We're evaluating an item where a NULL value in either the outer or
         inner value list does not automatically mean that we can return
         NULL/FALSE. An example: "<outer_value_list> NOT IN (SELECT ...)"
         The result ... is: NULL if the SELECT evaluated over the non-NULL
         values produces at least one row, FALSE otherwise */
      bool all_left_cols_null = true;
      const uint ncols = cache->cols();

      /* Turn off the predicates that are based on column compares for
         which the left part is currently NULL */
      for (uint i = 0; i < ncols; i++) {
        if (cache->element_index(i)->null_value)
          subqpred->set_cond_guard_var(i, false);   // ④ 动态关掉对应列的 trigcond
        else
          all_left_cols_null = false;
      }

      if (all_left_cols_null && result_for_null_param != UNKNOWN &&
          !subqpred->dependent_before_in2exists()) {
        /* This subquery was originally not correlated. The IN->EXISTS
           transformation may have made it correlated, but only to the left
           expression. All values in the left expression are NULL, and we have
           already evaluated the subquery for all NULL values: return the same
           result we did last time without evaluating the subquery. */
        null_value = result_for_null_param;         // ⑤ 全 NULL 左值 + 非相关 ⇒ 复用记忆结果
      } else {
        /* The subquery has to be evaluated */
        (void)subqpred->val_bool_naked();           // ⑥ 真执行
        if (!subqpred->value)
          null_value = subqpred->null_value;
        else
          null_value = true;                        // ⑦ 有匹配但左值含 NULL ⇒ UNKNOWN
        if (all_left_cols_null) result_for_null_param = null_value;   // ⑧ 记忆化
      }

      /* Turn all predicates back on */
      for (uint i = 0; i < ncols; i++) subqpred->set_cond_guard_var(i, true);
    }
    cache->store(subqpred->left_expr);  // ⑨ 重新 store（子查询执行可能污染了 cache 指向的 field）
    return subqpred->translate(null_value, false);
  }
  bool result = subqpred->val_bool_naked();         // ⑩ 左值无 NULL：直接转发
  null_value = subqpred->null_value;
  cache->store(subqpred->left_expr);                // ⑪ 同上，重新绑定
  return subqpred->translate(null_value, result);   // ⑫ NOT / IS TRUE / IS FALSE 都在这里翻
}
```

**真正的求值入口是 `val_bool_naked()`**（`sql/item_subselect.cc:1647`）——`val_bool()` 等都是 `assert(false)`：

```cpp
bool Item_in_subselect::val_bool() {
  // Substituted with Item_in_optimizer, so this function is never used
  assert(false);
  my_error(ER_INTERNAL_ERROR, MYF(0), "Invalid function call");
  return error_int();
}

bool Item_in_subselect::val_bool_naked() {
  assert(fixed);
  if (exec(current_thd)) {              // ① 执行子查询
    reset();
    return false;
  }
  if (was_null && !value) null_value = true;   // ②★ 三值逻辑落点：内层见过 NULL 且没匹配 ⇒ UNKNOWN
  /*
    This is the value of the naked IN. Negation, or applying of IS TRUE/FALSE,
    is left to the parent Item_in_optimizer, so make sure it's there:
  */
  assert(optimizer);                    // ③ 裸 IN 不做 NOT / IS TRUE，全交给外层
  return value;
}
```

**UNKNOWN 的"副作用"来源**（`Item_is_not_null_test` / `Item_ref_null_helper`）：

```cpp
// sql/item_cmpfunc.cc:6184 —— 人工 HAVING 的副作用
longlong Item_is_not_null_test::val_int() {
  assert(fixed);
  assert(used_tables_cache != 0);
  if (args[0]->is_null()) {
    owner->was_null |= 1;               // ★★ 把 owner(Item_in_subselect) 的 was_null 置位
    return 0;
  } else
    return 1;
}

// sql/item.cc:4902 —— 左值引用读到 NULL 时的副作用
longlong Item_ref_null_helper::val_int() {
  auto tmp = super::val_int();
  owner->was_null |= null_value;
  return tmp;
}
```

**`EXISTS` 的差异**（`Item_exists_subselect::val_bool()`，`sql/item_subselect.cc:1608`）：

```cpp
bool Item_exists_subselect::val_bool() {
  assert(fixed);
  if (exec(current_thd)) {              // ★ 走基类 exec()，Item_exists_subselect 不 override exec
    reset();
    return false;
  }
  // EXISTS can never return NULL value
  assert(!null_value);                  // ★★ EXISTS 永不为 NULL —— 这是硬断言
  return translate(null_value, value);
}
```

**`translate()`：三值 → 二值 + NULL**（`sql/item_subselect.cc:1364`）：

```cpp
bool Item_exists_subselect::translate(bool &null_v, bool v) {
  if (null_v)  // Naked IN returns UNKNOWN
  {
    assert(substype() != EXISTS_SUBS);  // ★ 只有 IN/ANY/ALL 可能到这里；纯 EXISTS 不可能
    switch (value_transform) {
      case BOOL_IDENTITY:
      case BOOL_NEGATED:
        return false;
      case BOOL_IS_TRUE:
      case BOOL_IS_FALSE:
        null_v = false;
        return false;
      case BOOL_NOT_TRUE:
      case BOOL_NOT_FALSE:
        null_v = false;
        return true;
      default: assert(false); return false;
    }
  }
  // Naked IN returns 'v'
  switch (value_transform) {
    case BOOL_IDENTITY:
    case BOOL_IS_TRUE:
    case BOOL_NOT_FALSE:
      return v;
    case BOOL_NEGATED:
    case BOOL_NOT_TRUE:
    case BOOL_IS_FALSE:
      return !v;
    default: assert(false); return v;
  }
}
```

⇒ **三个反直觉点**：① `Item_in_optimizer::val_int()` 结尾**两次** `cache->store(left_expr)`（子查询执行可能污染 cache 指向的 field）；② 物化引擎里"左值是 NULL"时**故意把 `value` 置 TRUE**，靠上层 `translate()` 翻成 UNKNOWN；③ `was_null` 才是 UNKNOWN 的载体，它由 `Item_is_not_null_test` / `Item_ref_null_helper` 的**副作用**写入。

### IN2EXISTS 执行：Item_in_optimizer（小结）

改写（`single_value_in_to_exists_transformer`）把 `oe IN (SELECT ie ...)` 变成 `EXISTS(SELECT 1 ... WHERE oe = ie ...)`，注入的等值谓词把外层值"推进"内层。

运行期核心 `Item_in_optimizer::val_int`：

```cpp
longlong Item_in_optimizer::val_int() {
  Item_in_subselect *subqpred = down_cast<Item_in_subselect *>(args[0]);
  cache->store(subqpred->left_expr);   // 1. 求外层左表达式，存入缓存
  cache->cache_value();
  if (cache->null_value) {             // 2. 外层含 NULL
    if (subqpred->abort_on_null) {     //   顶层裸 IN：直接 NULL
      null_value = true;
    } else {                           //   NOT IN 等：关掉 NULL 列的 trig_cond 再跑
      (void)subqpred->val_bool_naked();
    }
    return subqpred->translate(null_value, false);
  }
  bool result = subqpred->val_bool_naked();   // 3. 正常路径：执行内层 EXISTS
  null_value = subqpred->null_value;
  return subqpred->translate(null_value, result);
}
```

1. 外层左值存 `cache`（`Item_cache`），用于判断 NULL 和触发 trig_cond；
2. 外层 NULL 时：`abort_on_null`（裸 IN）直接 UNKNOWN；否则（NOT IN）用 `set_cond_guard_var(i, false)` 关掉左列为 NULL 的比较谓词再跑内层；
3. 正常路径 `val_bool_naked()` → `exec()` → `unit->execute()` **重跑内层**；`LIMIT 1` 保证找到一行即停。

### semi-join 五种策略的运行期

**FirstMatch → `NestedLoopIterator`（`JoinType::SEMI`）**：

```cpp
if (m_join_type == JoinType::SEMI) {   // 找到一行内层匹配后
  m_state = NEEDS_OUTER_ROW;           // 不继续扫内层，直接回外层
} else {
  m_state = READING_INNER_ROWS;
}
```

每个外层行只输出一次（第一个匹配），实现半连接的"去重 + 短路"。

**LooseScan → `RemoveDuplicatesOnIndexIterator`**：依赖"内层按索引有序输出"，去重只需比较**相邻行**的 key：

```cpp
if (!m_first_row && key_cmp(m_key->key_part, m_key_buf, m_key_len) == 0) {
  continue;   // 相同 key，跳过
}
m_first_row = false;
key_copy(m_key_buf, m_table->record[0], m_key, m_key_len);
```

★ 注意"只删**相邻**重复"这个前提——输入必须已按分组有序（见 [`03_cte.md`](03_cte.md) 与去重三兄弟的对比）。多表 LooseScan 用 `NestedLoopSemiJoinWithDuplicateRemovalIterator`：外层按 key 去重 + 内层只读一个匹配。

**Duplicate Weedout → `WeedoutIterator`**：**把半连接当普通内连接执行，但按外层表 rowid 去重**：

```cpp
for (SJ_TMP_TABLE_TAB *tab = m_sj->tabs; ...) {
  table->file->position(table->record[0]);   // 收集参与去重的表的 rowid
}
ret = do_sj_dups_weedout(thd(), m_sj);       // 用临时表判断 rowid 组合是否出现过
```

`do_sj_dups_weedout` 把多个表的 rowid 拼成一行，`check_unique_constraint` + `ha_write_row` 去重。若"confluent"（全字段去重）退化为 `LIMIT 1`。

**Materialize lookup / scan → `MaterializeAccessPath`**：规划期选 lookup 还是 scan（内层不依赖外层前面的表且可 lookup → `SJ_OPT_MATERIALIZE_LOOKUP`，否则 `SJ_OPT_MATERIALIZE_SCAN`）。运行期把半连接内层当独立"虚拟 join"递归建树 → 对可空列加 `IS NOT NULL`（**因为 handler 层把 NULL=NULL 当相等，而 IN 语义要求 NULL≠NULL**）→ 用 `NewMaterializeAccessPath` 物化。

### 标量子查询

`Item_singlerow_subselect::val_int`：

```cpp
longlong Item_singlerow_subselect::val_int() {
  if (!no_rows && !exec(current_thd) && !value->null_value) {
    null_value = false;
    return value->val_int();    // 从 Item_cache 读结果
  } else {
    reset();
    return error_int();
  }
}
```

- 每次 `val_*` 都调 `exec()`；结果由 `Query_result_scalar_subquery::send_data` 写入 `Item_cache value`（`store` + `cache_value` 快照）；
- 多行 → `send_data` 抛 `ER_SUBQUERY_NO_1_ROW`；
- `no_rows`（内层空结果）时直接返回 NULL / 错误值；
- ★ 所以"缓存"是**本次执行结果的快照**，不是跨外层行的结果缓存——**外层每行都会重跑一次标量子查询**（除非优化器能提升为 derived table / 半连接）。

### EXISTS 子查询

```cpp
bool Item_exists_subselect::val_bool() {
  if (exec(current_thd)) { reset(); return false; }
  assert(!null_value);                    // EXISTS 永不返回 NULL
  return translate(null_value, value);
}
```

短路机制在 `Query_result_exists_subquery::send_data`：收到第一行就把 `value = true`，配合 `LIMIT 1`，内层找到一行即停。

### 子查询缓存

> ⚠️ MySQL 8.0 **没有** `Subquery_cache` 类（那是 MariaDB 的特性）。8.0 的"缓存"是三处机制：

1. **左值缓存 `left_expr_cache`**（`Item_in_subselect`，仅 `SUBQ_MATERIALIZATION`）：`update_item_cache_if_changed` 比较当前值与缓存值，未变则复用上次结果，跳过 ref 查找；
2. **`result_for_null_param`**（`Item_in_optimizer`）：不相关 IN 子查询外层全 NULL 时，缓存 `NULL IN (...)` 的结果；
3. **物化临时表跨行复用**：`is_materialized` 保证临时表只物化一次，之后逐行 ref。

#### 深潜前：物化引擎完整源码

**`subselect_indexsubquery_engine`**（`sql/item_subselect.h:813`，8.0 仅存的两个 engine 之一）：

```cpp
/**
  A subquery execution engine that evaluates the subquery by doing index
  lookups in a single table's index.

  This engine is used to resolve subqueries in forms
    outer_expr IN (SELECT tbl.key FROM tbl WHERE subq_where)
  or, row-based:
    (oe1, .. oeN) IN (SELECT key_part1, ... key_partK FROM tbl WHERE subqwhere)
  i.e. the subquery is a single table SELECT without GROUP BY, aggregates, etc.
*/
class subselect_indexsubquery_engine {
 protected:
  Query_result_union *result = nullptr;
  TABLE *table{nullptr};
  Table_ref *table_ref{nullptr};
  Index_lookup ref;              // ★ 8.0 用 Index_lookup（旧版叫 TABLE_REF）：key_buff/key_copy/items/key_parts/null_ref_key
  join_type type{JT_UNKNOWN};    // JT_EQ_REF / JT_REF / JT_REF_OR_NULL
  Item *cond;                    /* The WHERE condition of subselect */
  ulonglong hash;                /* Hash value calculated by RefIterator */
  /*
    The "having" clause. This clause (further referred to as "artificial
    having") was inserted by subquery transformation code. It contains
    Item(s) that have a side-effect: they record whether the subquery has
    produced a row with NULL certain components. We need to use it for cases
    like (oe1, oe2) IN (SELECT t.key, t.no_key FROM t1) where we do index
    lookup on t.key=oe1 but need also to check if there was a row such that
    t.no_key IS NULL.
  */
  Item *having;                  // ★★ "人工 HAVING"：带副作用，用来把 UNKNOWN 与 FALSE 区分开

  Item_in_subselect *item;

 public:
  enum enum_engine_type { INDEXSUBQUERY_ENGINE, HASH_SJ_ENGINE };
  virtual bool exec(THD *thd);
  virtual enum_engine_type engine_type() const { return INDEXSUBQUERY_ENGINE; }
  virtual void create_iterators(THD *) {}
};

bool subselect_indexsubquery_engine::exec(THD *thd) {   // sql/item_subselect.cc:3134
  Query_expression *unit = item->unit;
  bool found;
  /*
    ... by asking the iterator for the inner query for a single row, and then
    immediately stopping. The iterator would usually do a simple ref lookup.
  */
  if (ExecuteExistsQuery(thd, unit, unit->root_iterator(), &found)) return true;
  item->value = found;
  item->assigned(true);
  return false;
}
```

**`subselect_hash_sj_engine`**（`sql/item_subselect.h:874`）——半连接物化：

```cpp
class subselect_hash_sj_engine final : public subselect_indexsubquery_engine {
 private:
  bool is_materialized;          // ★★ "第一次执行才物化"的开关
  bool has_zero_rows = false;    // ★ 空表短路（取代网上说的 m_count）
  /**
     Existence of inner NULLs in materialized table:
     By design, other values than IRRELEVANT_OR_FALSE are possible only if the
     subquery has only one inner expression.
  */
  enum nulls_exist {
    NEX_IRRELEVANT_OR_FALSE = 0,
    NEX_UNKNOWN = 1,
    NEX_TRUE = 2
  };
  enum nulls_exist mat_table_has_nulls;   // ★ 三态记忆：物化表里有没有 NULL（决定 UNKNOWN）
  Query_expression *const unit;
  unique_ptr_destroy_only<RowIterator> m_iterator;   // ★ 物化 + 查找都在这条 iterator 链里
  AccessPath *m_root_access_path;
  Query_result_interceptor *saved_result{nullptr};

 public:
  bool setup(THD *thd, const mem_root_deque<Item *> &tmp_columns);
  void cleanup() override;
  bool exec(THD *thd) override;
  enum_engine_type engine_type() const override { return HASH_SJ_ENGINE; }
  void create_iterators(THD *thd) override;
};
```

**`setup()`：建临时表 + 唯一索引 + 后过滤条件**（`sql/item_subselect.cc:3248`，关键段）：

```cpp
  /* 1. Create/initialize materialization related objects */
  if (!(tmp_result_sink = new (thd->mem_root) Query_result_union()))
    return true;
  if (tmp_result_sink->create_result_table(
          thd, tmp_columns,
          true,  // ★★ Eliminate duplicates —— 去重就是临时表上的唯一索引（或 hash_field），不是位图
          thd->variables.option_bits | TMP_TABLE_ALL_COLUMNS,
          "<materialized_subquery>", true, true))
    return true;

  tmp_table = tmp_result_sink->table;
  tmp_key = tmp_table->key_info;
  if (tmp_table->hash_field) {
    tmp_key_parts = CountVisibleFields(tmp_columns);
    key_length = ALIGN_SIZE(tmp_table->s->reclength);
  } else {
    tmp_key_parts = tmp_key->user_defined_key_parts;
    key_length = ALIGN_SIZE(tmp_key->key_length) * 2;
  }
  result = tmp_result_sink;

  /* 2. Create/initialize execution related objects */
  table = tmp_table;
  ref.key = 0;                                    /* The only temp table index. */
  ref.key_length = tmp_key->key_length;
  type = (tmp_table->key_info[0].flags & HA_NOSAME) ? JT_EQ_REF : JT_REF;
  // ★ 唯一索引 → EQ_REF，否则 REF
  if (tmp_table->hash_field) ref.keypart_hash = &hash;

  /*
    Create an artificial condition to post-filter those rows matched by index
    lookups that cannot be distinguished by the index lookup procedure, for
    example: - because of truncation (if the outer column type's length is
    bigger than the inner column type's, index lookup will use a truncated
    outer value as search key, yielding false positives).
             - because the index is over hash_field and thus not unique.
  */
  assert(cond == nullptr);
  if (!(cond = new Item_cond_and)) return true;
  ...
  for (uint part_no = 0; part_no < tmp_key_parts; part_no++) {
    ref.items[part_no] = item->left_expr->element_index(part_no);   // ★ 左表达式每列 = 查找 key 来源
    if (!(right_col_item = new Item_field(...)) ||
        !(eq_cond = new Item_func_eq(ref.items[part_no], right_col_item)) ||
        ((Item_cond_and *)cond)->add(eq_cond)) { ... }

    if (nullable &&  // nullable column in tmp table, and UNKNOWN should not be interpreted as FALSE
        !item->abort_on_null) {
      // It must be the single column, or we wouldn't be here
      assert(tmp_key_parts == 1);
      // Be ready to search for NULL into inner column:
      ref.null_ref_key = cur_ref_buff;            // ★ 为"物化表里有没有 NULL"预留探测缓冲
      mat_table_has_nulls = NEX_UNKNOWN;          // ★ 三态初始为"未知"
    } else {
      ref.null_ref_key = nullptr;
      mat_table_has_nulls = NEX_IRRELEVANT_OR_FALSE;
    }
    ...
  }
  ref.key_err = true;
  ref.key_parts = tmp_key_parts;
  if (cond->fix_fields(thd, &cond)) return true;
```

**`create_iterators()`：物化下放给 `MaterializeIterator` 的源码证据**（`sql/item_subselect.cc:3418`）：

```cpp
void subselect_hash_sj_engine::create_iterators(THD *thd) {
  if (unit->root_access_path() == nullptr) {
    m_root_access_path =
        NewZeroRowsAccessPath(thd, "Not optimized, outer query is empty");   // ★ 外层空 ⇒ 连物化都不做
    m_iterator = CreateIteratorFromAccessPath(thd, m_root_access_path, nullptr, true);
    return;
  }

  /* We're only ever reading one row from the iterator ... (As a
     microoptimization, we add a LIMIT 1 if there's a filter and the index is
     unique ...) Also, note that we never need to worry about searching for
     NULLs ... subqueries with JT_REF_OR_NULL are always transformed with
     IN-to-EXISTS, and thus, their artificial HAVING rejects NULL values. */
  assert(type != JT_REF_OR_NULL);
  AccessPath *path = NewRefAccessPath(thd, table, &ref, false, false, false);

  if (type == JT_EQ_REF && (cond != nullptr || having != nullptr)) {
    path = NewLimitOffsetAccessPath(thd, path, /*limit=*/1, /*offset=*/0, false, false, nullptr);
  }
  if (cond != nullptr) path = NewFilterAccessPath(thd, path, cond);   // setup() 造的等值后过滤
  if (having != nullptr) path = NewFilterAccessPath(thd, path, having);

  /*
    This impersonates the materialized table as a derived table. However, there
    are certain aspects of a derived table that are NOT set ... However, it
    works for the time being (partially because TABLE object's
    pos_in_table_list is nullptr).
  */
  table_ref->set_derived_query_expression(unit);   // ★ 把临时表伪装成"派生表"
  path = GetAccessPathForDerivedTable(             // ★★ 物化在这里：生成 MATERIALIZE access path
      thd, table_ref, table, /*rematerialize=*/false,
      /*invalidators=*/nullptr, /*need_rowid=*/false, path);

  m_root_access_path = path;
  JOIN *join = unit->is_set_operation() ? nullptr : unit->first_query_block()->join;
  unit->finalize(thd);
  m_iterator = CreateIteratorFromAccessPath(thd, path, join, true);

  // The unit is not supposed to be executed by itself now.
  unit->clear_root_access_path();                  // ★ 抢走 unit 的 iterator：unit 不再自执行
}
```

**`exec()` 完整**（`sql/item_subselect.cc:3531`）——NULL 三态与判空短路都在这里：

```cpp
static int safe_index_read(TABLE *table, const Index_lookup &ref) {
  int error = table->file->ha_index_read_map(      // ★★ 唯一一处直接 ha_index_read_map：只用于探测物化表里是否存在 NULL
      table->record[0], ref.key_buff, make_prev_keypart_map(ref.key_parts),
      HA_READ_KEY_EXACT);
  if (error) return report_handler_error(table, error);
  return 0;
}

bool subselect_hash_sj_engine::exec(THD *thd) {
  /* Optimize and materialize the subquery during the first execution of
     the subquery predicate. */
  if (!is_materialized) {                          // ★★ "第一次执行才物化"
    thd->lex->set_current_query_block(unit->first_query_block());
    assert(unit->first_query_block()->master_query_expression()->is_optimized());

    // Init() triggers materialization. (It also triggers some unneeded setup
    // of the RefIterator, but it is cheap.)
    bool error = m_iterator->Init();               // ★ MaterializeIterator::Init() 真跑一遍写进临时表
    if (error || thd->is_fatal_error()) return true;
    /* TODO: - Unlock all subquery tables as we don't need them ... */
    is_materialized = true;

    // See if we have zero rows or not.
    table->file->info(HA_STATUS_VARIABLE);
    if (table->file->ha_table_flags() & HA_STATS_RECORDS_IS_EXACT) {
      has_zero_rows = (table->file->stats.records == 0);   // ★ 引擎能给精确行数就直接判
    } else {
      // Index must be closed before starting to scan.
      if (table->file->inited) table->file->ha_index_or_rnd_end();
      TableScanIterator scan(thd, table, -1.0, nullptr);
      int ret = scan.Read();
      if (ret == 1 || thd->is_error()) return true;
      has_zero_rows = (ret == -1);                 // ★ 否则扫一行确认
    }
  }

  if (has_zero_rows) {
    item->value = false;                           // ★ 空物化表 ⇒ IN 恒 FALSE（NULL IN (空集) 也是 FALSE）
    return false;
  }

  /*
    Here we could be brutal and set item->null_value. But we prefer to be
    well-behaved and rather set the properties which
    Item_in_subselect::val_bool() and Item_in_optimizer::val_int() expect ...
  */
  if (item->left_expr->element_index(0)->null_value) {
    /* The first outer expression oe1 is NULL ... The correct answer is
       UNKNOWN. Do as if searching with all triggered conditions disabled:
       this would surely find a row. The caller will translate this to UNKNOWN. */
    assert(item->left_expr->cols() == 1);
    item->value = true;                            // ★★ 反直觉：故意置 TRUE，让上层 translate() 翻成 UNKNOWN
    return false;
  }

  hash = 0;
  bool found;
  if (ExecuteExistsQuery(thd, item->unit, m_iterator.get(), &found)) {  // ★ 走 iterator 链查找
    return true;
  }
  item->value = found;
  item->assigned(true);

  if (!found &&  // no exact match
      mat_table_has_nulls != NEX_IRRELEVANT_OR_FALSE) {
    /* There is only one outer expression. It's not NULL. exec() above has set
       the answer to FALSE, but if there exists an inner NULL in the temporary
       table, then the correct answer is UNKNOWN, so let's find out. */
    if (mat_table_has_nulls == NEX_UNKNOWN)        // We do not know yet
    {
      // Search for NULL inside tmp table, and remember the outcome.
      *ref.null_ref_key = 1;                       // 把 key 置成 NULL 去查
      if (!table->file->inited && table->file->ha_index_init(ref.key, false))
        return true;
      if (safe_index_read(table, ref) == 1) return true;   // ★ 直接走 handler，绕过 iterator
      *ref.null_ref_key = 0;  // prepare for next searches of non-NULL
      mat_table_has_nulls =
          table->has_row() ? NEX_TRUE : NEX_IRRELEVANT_OR_FALSE;   // ★ 记忆化：下次不用再探
    }
    if (mat_table_has_nulls == NEX_TRUE) {
      /* There exists an inner NULL. The correct answer is UNKNOWN. Do as if
         searching with all triggered conditions enabled; that would not find
         any match, but Item_is_not_null_test would notice a NULL: */
      item->value = false;
      item->was_null = true;                       // ★★ was_null 才是 UNKNOWN 的真正载体
    }
  }
  return false;
}
```

**`cleanup()`**（`sql/item_subselect.cc:3495`）——每次执行结束都要重置，否则下次执行会复用旧物化表：

```cpp
void subselect_hash_sj_engine::cleanup() {
  is_materialized = false;                 // ★ 下次执行重新物化
  if (result != nullptr) result->cleanup();
  m_root_access_path = nullptr;
  m_iterator.reset();
  if (table != nullptr) {
    if (table->file->inited) table->file->ha_index_end();
    close_tmp_table(table);
    free_tmp_table(table);
    table = nullptr;
  }
  if (unit->is_executed()) unit->reset_executed();
}
```

### 深潜：物化引擎的建表 / 迭代器 / trig_cond

**setup() 建表细节**：

- **`key_length` 分两种**：有 `hash_field` 时 `key_length = ALIGN_SIZE(reclength)`；否则 `ALIGN_SIZE(key_length) * 2`（后者要同时存"原始值"与"用于比较的值"，防截断误判）；
- **访问类型**：`type = (flags & HA_NOSAME) ? JT_EQ_REF : JT_REF`——唯一索引走 EQ_REF，否则 REF；
- **`store_key` 的选择**：哈希列走 `store_key_hash_item`；普通列走 `store_key`；**可空列的 key buffer 首位留 NULL 字节**（这是后过滤与 NULL 补查能工作的前提）。

**后过滤为什么必须**：见「理论基础」。为给这些 `Item_func_eq` 做 `fix_fields`，代码会造一个临时 `Table_ref("<materialized_subquery>")`。

**create_iterators 的三个反直觉设计**：

1. **永不用 `EQRefIterator` / `AlternativeIterator`**（源码注释）：物化表是临时表，语义上不会出现"NULL 导致 ref 非法"的情况（而 `ALTERNATIVE` 正是为那种情况准备的）；
2. **`JT_EQ_REF` 且有 cond/having 时补 `LIMIT 1`**——唯一键定位，多读无意义；
3. **把物化表伪装成 derived table** 调 `GetAccessPathForDerivedTable(rematerialize=false, invalidators=nullptr)`——**这正是"子查询物化不参与 lateral 失效"的源码根据**（对比真正的 derived table 会传 invalidators）；
4. 末尾 `unit->clear_root_access_path()`。

**cleanup 与重执行**：每次执行后 `is_materialized = false`、`close_tmp_table`/`free_tmp_table`、`unit->reset_executed()`。★ **Prepared Statement 重执行会重新物化**——物化不是跨语句持久的。

**NULL 补查的真实实现**：未命中但内层可能有 NULL 时，`*ref.null_ref_key = 1`（把 key 设成 NULL）后走 `safe_index_read`（`ha_index_read_map(..., HA_READ_KEY_EXACT)`）。结果用**三态枚举**缓存（`NEX_UNKNOWN` / `NEX_IRRELEVANT_OR_FALSE` / `NEX_TRUE`）。

**trig_cond 机制**：

- `pushed_cond_guards` 分配：单值、行值两处；
- 求值核心：`return *trig_var ? args[0]->val_int() : 1;`——守卫变量为真才求值，否则当作真；
- 类型 `OUTER_FIELD_IS_NOT_NULL`；`Item_is_not_null_test::val_int` 的副作用 `owner->was_null |= 1` 再返回 0。

**`subselect_indexsubquery_engine` 从哪来**：`JOIN::replace_index_subquery` 在子查询可转成 `JT_EQ_REF`/`JT_REF`（或 `JT_REF_OR_NULL` 且 having 源自 in2exists）时创建该引擎。它的 `exec` **直接复用 `unit->root_iterator()`**——不走重新执行，而是走优化好的迭代器树。

---

## 可观测性

| 想看什么 | 手段 |
|---|---|
| 用了哪个策略 | `EXPLAIN` 的 `select_type`（`DEPENDENT SUBQUERY` / `MATERIALIZED` / `SIMPLE`）与 `Extra`（`FirstMatch(t2)`、`LooseScan(m,n)`、`Using temporary; Start temporary/End temporary` = Weedout、`Materialize`） |
| 半连接是否生效 | `EXPLAIN` 里子查询消失了（变成 join）就是半连接成功；仍是 `DEPENDENT SUBQUERY` 说明没扁平化 |
| 策略选择的依据 | `SET optimizer_trace="enabled=on"` → trace 的 `subquery` / `semijoin_optimization` 段（含每种策略的代价比较与淘汰原因） |
| 物化是否发生 | `EXPLAIN FORMAT=JSON` 的 `materialized_from_subquery`；传统格式里 `select_type=MATERIALIZED` |
| 开关 | `optimizer_switch` 的 `semijoin`、`materialization`、`subquery_materialization_cost_based`（基于代价的物化选择） |
| 相关子查询的代价 | `EXPLAIN ANALYZE` 看 `DEPENDENT SUBQUERY` 的实际循环次数（rows × loops） |

★ **排查口诀**：看到 `DEPENDENT SUBQUERY` 且外层行数大 → 优先怀疑"没扁平化、也没物化"（可能因为聚合/LIMIT/相关列导致半连接不适用），此时考虑改写 SQL（改成 JOIN 或 EXISTS）。

---

## Misc

### ★ 本机制里的工程实现技法

1. **trig_cond：用数据控制静态表达式树**——不需要重建 Item 树，改一个 bool 就能让某个条件"暂时不算数"（返回 1）。这是处理 NULL 语义的关键技巧。
2. **近似剪枝 + 精确确认**（物化后过滤）：用便宜的索引/哈希查找剪枝，用逐列 `Item_func_eq` 精确确认——与 bloom filter 同构的两阶段模式。
3. **三态枚举缓存 NULL 补查结果**（`NEX_UNKNOWN` / `NEX_IRRELEVANT_OR_FALSE` / `NEX_TRUE`）：避免每次未命中都重查 NULL 行。
4. **惰性物化**：`is_materialized` 标志让物化推迟到第一次真正求值——外层零行时完全不物化。
5. **伪装成 derived table 复用路径**：`GetAccessPathForDerivedTable(rematerialize=false, invalidators=nullptr)`——用同一套物化机制表达"不参与 lateral 失效"的语义，不新增代码路径。
6. **可空列 key buffer 首位留 NULL 字节**：让"查 NULL 行"变成普通的 key 构造，不需要特殊的查找分支。

### 失效与退化场景

| 场景 | 表现 | 判据 |
|---|---|---|
| 子查询含聚合 / `LIMIT` / `UNION` | **不能扁平化成 semi-join**，退到物化或 IN2EXISTS | 半连接准入条件（见 02 篇） |
| 相关子查询（引用外层列） | 不能物化（内层依赖外层），退到 IN2EXISTS | `SUBQ_MATERIALIZATION` 的适用条件 |
| 外层行数大 + 走 IN2EXISTS | **每行重跑内层** —— 最经典的慢查询形态 | `EXPLAIN` 的 `DEPENDENT SUBQUERY` |
| 物化表太大 | 临时表可能落盘（`TempTable` 超过 `temptable_max_ram` 转 mmap/InnoDB） | 见 [`07_temptable.md`](07_temptable.md) |
| 物化表的索引是哈希/前缀 | 假阳性 → 后过滤必须跑（不能省） | 后过滤条件存在 |
| 标量子查询在外层每行 | **每行重跑一次**（缓存只在本行内有效） | `Item_singlerow_subselect::val_*` 每次 `exec()` |
| PS 重执行 | **重新物化**（`is_materialized` 重置） | cleanup 逻辑 |
| Weedout 的临时表 | 每行都要 `ha_write_row` 判重，成本高 | `do_sj_dups_weedout` |

### 易混淆点

- ⚠️ **MySQL 8.0 没有 `Subquery_cache` 类**（MariaDB 特性），也没有 `subselect_rowid_merge_engine`（MariaDB）——别在 MySQL 源码里找。
- ⚠️ **两套"物化"不是一回事**：`MaterializeAccessPath`（semi-join 扁平化后）vs `subselect_hash_sj_engine`（未扁平化的 IN）。
- ⚠️ **标量子查询的 `Item_cache` 不是跨行缓存**——是本次执行的快照，外层每行都会重跑。
- ⚠️ **LooseScan 只删相邻重复行**——输入必须已按 key 有序，否则去重结果错误。
- ⚠️ **物化表走 `JT_EQ_REF` 但不用 `EQRefIterator`**——因为临时表不会出现"NULL 导致 ref 非法"。
- ⚠️ **`translate()` 负责三值逻辑**：子查询内部的 bool 结果要经过 `translate(null_value, value)` 翻译成 SQL 的三值语义（UNKNOWN 的处理集中在这里）。

---

## 参考

**论文**

- **Galindo-Legaria《Parameterized Queries and Nesting Equivalences》(2001)**——子查询去关联化与执行策略的理论基础

**官方文档**

- *MySQL 8.0 Reference Manual → Optimizing Subqueries with Materialization*
- *MySQL 8.0 Reference Manual → Optimizing IN/=ANY Subqueries*
- *MySQL 8.0 Reference Manual → Optimizing Derived Tables*

**内核月报**

- **2021/06《Semi-join 优化与执行逻辑》**

**源码**

- `sql/item_subselect.h` / `item_subselect.cc`——`Subquery_strategy`、`subselect_hash_sj_engine`、`Item_exists_subselect`、`Item_singlerow_subselect`
- `sql/item_cmpfunc.cc`——`Item_in_optimizer::val_int`、trig_cond 求值、`Item_is_not_null_test`
- `sql/sql_optimizer.cc`——`JOIN::replace_index_subquery`
- `sql/composite_iterators.cc`——`NestedLoopIterator`、`WeedoutIterator`、`RemoveDuplicatesOnIndexIterator`、`NestedLoopSemiJoinWithDuplicateRemovalIterator`

**两侧分工**

- 优化器侧（策略选择与改写）：[`../07_optimize/logical/03_semijoin.md`](../07_optimize/logical/03_semijoin.md)、[`../07_optimize/logical/02_subquery.md`](../07_optimize/logical/02_subquery.md)
- 物化机制本身：[`08_materialization.md`](08_materialization.md)
- 迭代器体系：[`../09_executor_iterator.md`](../09_executor_iterator.md)
- 临时表容器：[`07_temptable.md`](07_temptable.md)

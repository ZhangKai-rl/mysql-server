# 11 Optimizer Hints：`/*+ ... */` 干预优化器

> 标准 MySQL 5.7.7+ 的官方 hint 语法（`/*+ ... */`），用于给优化器提供现成的优化决策、缩小计划搜索空间。本篇讲它的完整管线：**独立 hint 语法器 → PT_hint_list 语法树 → Opt_hints 四层对象树 → 优化期按类型分发**。
>
> **边界**：hint 是"干预优化器"的机制，是横切主题，不属四阶段任何单一阶段；它最终作用到 06~08 篇的 join order / 访问方法 / range 决策点上。
>
> ⚠️ 与 Statement Outline（阿里云 RDS/PolarDB 私有特性）区分：Statement Outline 是把已知好计划**固化并强制复现**，标准 MySQL 8.0.39 没有；本文讲的是标准 MySQL 的 `/*+ */` hint。

## 目录

- [一、架构总览：双语法器 + 四层对象树](#一架构总览双语法器--四层对象树)
- [二、hint 的语法与解析](#二hint-的语法与解析)
- [三、Opt_hints 四层对象树](#三opt_hints-四层对象树)
- [四、hint 如何映射到优化器（三类）](#四hint-如何映射到优化器三类)
- [五、JOIN_ORDER 如何强制连接顺序](#五join_order-如何强制连接顺序)
- [六、INDEX 如何限制候选索引](#六index-如何限制候选索引)
- [七、SET_VAR / MAX_EXECUTION_TIME / RESOURCE_GROUP](#七set_var--max_execution_time--resource_group)
- [八、8.0.39 已废弃或未生效的 hint](#八8039-已废弃或未生效的-hint)

---

## 一、架构总览：双语法器 + 四层对象树

**核心结论**：8.0.39 的 Optimizer Hints 是"**双语法器 + 四层对象树 + 优化期按类型分发**"的架构：

1. 主语法器 `sql_yacc.yy` 里**没有** hint 产生式；hint 的产生式在**独立的小 Bison 语法器** `sql/sql_hints.yy`。
2. 主词法器 `sql_lex.cc` 识别到 hintable 关键字（SELECT/INSERT/UPDATE/DELETE/REPLACE）后，若紧跟 `/*+`，就调小语法器 `HINT_PARSER_parse` 把注释解析成 `PT_hint_list`。
3. `PT_hint_list`/`PT_hint` 在名字解析阶段（`contextualize()`）转成**四层 `Opt_hints` 对象树**（global → query block → table → key）。
4. 优化阶段通过 `hint_key_state`、`hint_table_state`、`apply_join_order_hints`、`update_index_hint_maps` 读取这棵树。

> ⚠️ 常见误解纠正：网上资料常说的 `Opt_hints::update_parse_time_state`、`PT_hint::make_opt_hint` **在 8.0.39 不存在**。对应职责分别是各 `PT_*_hint::contextualize()` 与 `Opt_hints*::set_switch`/`register_child`。

---

## 二、hint 的语法与解析

### 2.1 主词法器识别（sql_lex.cc:905）

```cpp
// sql/sql_lex.cc:905（find_keyword 内）
lip->yylval->optimizer_hints = nullptr;
if (symbol->group & SG_HINTABLE_KEYWORDS) {   // SELECT/INSERT/UPDATE/DELETE/REPLACE
  lip->add_digest_token(symbol->tok, lip->yylval);
  if (consume_optimizer_hints(lip)) return ABORT_SYM;
  lip->skip_digest = true;
}
```

`consume_optimizer_hints`（`:865`）做"探测 + 调小语法器"：

```cpp
// 跳过空白，若接下来三字节是 / * +，则确认是 hint 注释
if (lip->yyPeekn(whitespace) == '/' && lip->yyPeekn(whitespace + 1) == '*' &&
    lip->yyPeekn(whitespace + 2) == '+') {
  Hint_scanner hint_scanner(lip->m_thd, ...);
  PT_hint_list *hint_list = nullptr;
  int rc = HINT_PARSER_parse(lip->m_thd, &hint_scanner, &hint_list);  // 小语法器
  ...
  lip->yylval->optimizer_hints = hint_list;   // 写回主 token 语义值
  return false;
}
```

**算法**：关键字后紧跟 `/*+` → 构造 `Hint_scanner`，调 `sql_hints.yy` 生成的 `HINT_PARSER_parse` 解析注释体 → 结果 `hint_list` 存进 `yylval->optimizer_hints`，主语法器再把它挂到 `PT_query_specification`/`PT_update`/`PT_delete`/`PT_insert`。

### 2.2 小语法器 sql_hints.yy 的结构

```yacc
hint_list: hint | hint_list hint ;
hint:
      index_level_hint      /* INDEX/NO_INDEX 等带索引参数 */
    | table_level_hint      /* BKA/BNL/NO_ICP 等表级 */
    | qb_level_hint         /* SEMIJOIN/SUBQUERY/JOIN_ORDER 等 */
    | qb_name_hint          /* QB_NAME(x) */
    | max_execution_time_hint
    | set_var_hint
    | resource_group_hint
```

语义策略参数（`SEMIJOIN(strategy)` 的参数被 OR 成一个位图）：

```yacc
semijoin_strategy:
      FIRSTMATCH_HINT      { $$ = OPTIMIZER_SWITCH_FIRSTMATCH; }
    | LOOSESCAN_HINT       { $$ = OPTIMIZER_SWITCH_LOOSE_SCAN; }
    | MATERIALIZATION_HINT { $$ = OPTIMIZER_SWITCH_MATERIALIZATION; }
    | DUPSWEEDOUT_HINT     { $$ = OPTIMIZER_SWITCH_DUPSWEEDOUT; }
```

### 2.3 hint 能出现的位置

- **SELECT / UPDATE / DELETE / INSERT / REPLACE**：关键字后紧跟 `/*+`（`sql_yacc.yy:1988` 声明这些 token 携带 hint 列表）
- **子查询**：每个 `query_specification` 自带 `opt_hints`，子查询内也可写 hint
- **`@qb_name` 跨查询块定位**：表级/键级 hint 的 `table_name@qb_name` 语法允许外层 hint 指定作用到哪个查询块

---

## 三、Opt_hints 四层对象树

### 3.1 四层结构（opt_hints.h:149）

```
Opt_hints_global                      语句级（一个 LEX 一个）
   └── Opt_hints_qb[]                 查询块级（每个 Query_block 一个）
          └── Opt_hints_table[]       表级
                 └── Opt_hints_key[]  索引级
```

`Opt_hints` 基类（`opt_hints.h:162`）核心成员：

```cpp
class Opt_hints {
  const LEX_CSTRING *name;              // 空=global, QB名=qb级, 表名=表级, 索引名=key级
  Opt_hints *parent;                    // 父节点
  Opt_hints_map hints_map;              // hint 状态位图
  Mem_root_array<Opt_hints *> child_array;
  bool resolved;                        // 是否已绑定到真实对象
};
```

- `Opt_hints_global`（:350）：额外持 `max_exec_time`、`sys_var_hint`
- `Opt_hints_qb`（:372）：额外持 `select_number`、`subquery_hint`/`semijoin_hint`、`join_order_hints` 数组
- `Opt_hints_table`（:549）：额外持 `keyinfo_array`、6 个 compound key hint
- `Opt_hints_key`（:641）：仅重写 `append_name`

### 3.2 hint 枚举与元信息（opt_hints.h:64）

```cpp
enum opt_hints_enum {
  BKA_HINT_ENUM, BNL_HINT_ENUM, ICP_HINT_ENUM, MRR_HINT_ENUM,
  NO_RANGE_HINT_ENUM, MAX_EXEC_TIME_HINT_ENUM, QB_NAME_HINT_ENUM,
  SEMIJOIN_HINT_ENUM, SUBQUERY_HINT_ENUM, DERIVED_MERGE_HINT_ENUM,
  JOIN_PREFIX_HINT_ENUM, JOIN_SUFFIX_HINT_ENUM, JOIN_ORDER_HINT_ENUM,
  JOIN_FIXED_ORDER_HINT_ENUM, INDEX_MERGE_HINT_ENUM, RESOURCE_GROUP_HINT_ENUM,
  SKIP_SCAN_HINT_ENUM, HASH_JOIN_HINT_ENUM, INDEX_HINT_ENUM,
  JOIN_INDEX_HINT_ENUM, GROUP_INDEX_HINT_ENUM, ORDER_INDEX_HINT_ENUM,
  DERIVED_CONDITION_PUSHDOWN_HINT_ENUM, MAX_HINT_ENUM
};
```

每个枚举对应 `opt_hint_info[]`（`opt_hints.cc:65`）一条元信息，三个字段：`check_upper_lvl`（可否指定在多级）、`switch_hint`（开关型 vs 复杂型）、`irregular_hint`（打印特殊）。

### 3.3 `Opt_hints_map`（值位图 + 指定位图）

```cpp
class Opt_hints_map {
  Bitmap<64> hints;            // hint state（开关值）
  Bitmap<64> hints_specified;  // 是否已指定
};
```

`set_switch` 同时写两个位图：`hints` 写开关值、`hints_specified` 标记"已指定"。区分"指定为 off"与"未指定"——后者要回落到 `optimizer_switch`。

---

## 四、hint 如何映射到优化器（三类）

### 4.1 开关型（NO_ICP / NO_MRR / BKA / BNL / MERGE）

核心读取 `Opt_hints::get_switch`（`opt_hints.cc:116`）：

```cpp
bool Opt_hints::get_switch(opt_hints_enum type_arg) const {
  if (is_specified(type_arg)) return hints_map.switch_on(type_arg);
  if (opt_hint_info[type_arg].check_upper_lvl) return parent->get_switch(type_arg);  // 查父级
  return false;
}
```

对外封装（`opt_hints.cc:891`）：

```cpp
bool hint_key_state(const THD *thd, const Table_ref *table, uint keyno,
                    opt_hints_enum type_arg, uint optimizer_switch) {
  ...  // hint 已指定则覆盖，否则回落到 optimizer_switch_flag
}
```

**"hint 优先、开关兜底"**。消费实例：

- NO_ICP → `sql_select.cc:2985` `hint_key_state(..., ICP_HINT_ENUM, OPTIMIZER_SWITCH_INDEX_CONDITION_PUSHDOWN)`
- NO_MRR → `handler.cc:6558` `hint_key_state(..., MRR_HINT_ENUM, OPTIMIZER_SWITCH_MRR)`
- NO_BNL → `sql_planner.cc:1004` `hint_table_state(..., BNL_HINT_ENUM, OPTIMIZER_SWITCH_BNL)`
- NO_MERGE → `sql_resolver.cc:3501` `hint_table_state(..., DERIVED_MERGE_HINT_ENUM, ...)`

### 4.2 枚举型（SEMIJOIN / SUBQUERY）

`Opt_hints_qb::semijoin_enabled`（`opt_hints.cc:239`）：

```cpp
bool Opt_hints_qb::semijoin_enabled(const THD *thd) const {
  if (subquery_hint) return false;                    // SUBQUERY hint 禁用半连接
  if (semijoin_hint) {
    if (semijoin_hint->switch_on()) return true;      // SEMIJOIN(...) 强制开
    if (semijoin_hint->get_args() == 0) return false; // NO_SEMIJOIN() 强制关
  }
  return thd->optimizer_switch_flag(OPTIMIZER_SWITCH_SEMIJOIN);
}
```

`sj_enabled_strategies`（:256）处理策略子集：`SEMIJOIN(a,b)` 只启用 a/b 两种策略；`NO_SEMIJOIN(a)` 从开关里剔除 a。

`subquery_strategy`（:270）：`SUBQUERY(MATERIALIZATION)` → `SUBQ_MATERIALIZATION`，`SUBQUERY(INTOEXISTS)` → `SUBQ_EXISTS`。消费点在 `Query_block::subquery_strategy`（`sql_lex.cc:4525`，见 02 篇 4.2 节）。

### 4.3 复杂型（JOIN_ORDER / INDEX）

见第五、六节。

---

## 五、JOIN_ORDER 如何强制连接顺序

### 5.1 JOIN_ORDER → 表依赖（set_join_hint_deps，opt_hints.cc:459）

```cpp
for (const Hint_param_table *hint_table = hint_table_list->begin();
     hint_table < hint_table_list->end(); hint_table++) {
  for (uint i = 0; i < join->tables; i++) {
    const Table_ref *table = join->join_tab[i].table_ref;
    if (!compare_table_name(hint_table, table)) {
      if (join->const_table_map & table->map()) break;   // const 表不参与
      JOIN_TAB *tab = &join->join_tab[i];
      tab->dependent |= hint_tab_map;        // ★ 表 i 依赖前面所有 hint 表
      hint_tab_map |= tab->table_ref->map();
      break;
    }
  }
}
```

**核心算法**：把 `JOIN_ORDER(t1,t2,t3)` 线性化成链式依赖——`t2 依赖 t1`、`t3 依赖 t1|t2`。第 2 张 hint 表依赖第 1 张，第 3 张依赖前两张，以此类推。

非 hint 表的处理（`get_other_dep`，:351）：

| hint | 非 hint 表的依赖 |
|------|------------------|
| `JOIN_PREFIX(t1,t2)` | 非 hint 表依赖全部 hint 表（hint 表必须在最前） |
| `JOIN_SUFFIX(t1,t2)` | hint 表依赖其他所有表（hint 表必须在最后） |
| `JOIN_ORDER(t1,t2)` | 非 hint 表不额外加依赖（可自由穿插） |

最后 `propagate_dependencies`（`sql_optimizer.cc:5556`）求传递闭包 + 检测环（自依赖 = 成环）。

### 5.2 依赖如何被遵守（best_extension_by_limited_search）

在 06 篇的主循环里（`sql_planner.cc:2770`）：

```cpp
if ((remaining_tables & real_table_bit) &&
    !(eq_ref_extended & real_table_bit) &&
    !(remaining_tables & s->dependent) &&   // ★ 依赖已满足才可扩展
    (!idx || !check_interleaving_with_nj(s))) {
```

`!(remaining_tables & s->dependent)` 表示"表 s 依赖的所有表都已不在 remaining（即已在前缀中）"。因为 JOIN_ORDER 已经把 `dependent` 设成它前面的 hint 表集合，贪心搜索就**不可能**把 hint 顺序打乱——违反依赖的表在候选过滤时被跳过。

### 5.3 JOIN_FIXED_ORDER 等价于 STRAIGHT_JOIN

`parse_tree_hints.cc:266`：

```cpp
case JOIN_FIXED_ORDER_HINT_ENUM:
  if (qb->get_switch(JOIN_PREFIX_HINT_ENUM) || ...)  // 与 JOIN_ORDER 系列互斥
    conflict = true;
  else
    pc->select->add_base_options(SELECT_STRAIGHT_JOIN);  // 等价 STRAIGHT_JOIN
  break;
```

`JOIN_FIXED_ORDER` 直接设 `SELECT_STRAIGHT_JOIN` 位，语义**完全等价**于 STRAIGHT_JOIN：保持 FROM 书写顺序。在 `choose_table_order`（06 篇 2.2）里走 `straight_join` 分支 → `optimize_straight_join` 只算访问方式、不搜 join order。

应用时机（`sql_optimizer.cc:5377`）：

```cpp
if (query_block->opt_hints_qb &&
    !(query_block->active_options() & SELECT_STRAIGHT_JOIN))  // fixed order 已由 straight 分支处理
  query_block->opt_hints_qb->apply_join_order_hints(this);
```

---

## 六、INDEX 如何限制候选索引

### 6.1 update_index_hint_map（集合运算，opt_hints.cc:599）

```cpp
void Opt_hints_table::update_index_hint_map(Key_map *keys_to_use,
                                            Key_map *available_keys_to_use,
                                            opt_hints_enum type_arg) {
  if (is_resolved(type_arg)) {
    Key_map *keys_specified_in_hint = get_compound_key_hint(type_arg)->get_key_map();
    if (get_switch(type_arg)) {
      if (keys_specified_in_hint->is_clear_all())
        keys_to_use->merge(*available_keys_to_use);       // INDEX(t) 全可用
      else {
        keys_to_use->merge(*keys_specified_in_hint);      // INDEX(t i1,i2)
        keys_to_use->intersect(*available_keys_to_use);   // ∩ 可用集
      }
    } else {
      if (keys_specified_in_hint->is_clear_all()) keys_to_use->clear_all();
      else keys_to_use->subtract(*keys_specified_in_hint); // NO_INDEX(t i1)
    }
  }
}
```

**集合运算**：`INDEX(t i1,i2)` = 指定键 ∩ 可用集；`NO_INDEX(t i1)` = 从候选减掉。结果写进 `TABLE::keys_in_use_for_query/group_by/order_by`（`update_index_hint_maps`，:641）。

### 6.2 range/ref 如何读取

- range 优化器（`range_optimizer.cc:382`）遍历 `keys_to_use`（即 `keys_in_use_for_query`），被减掉的索引自然不被考虑
- `NO_RANGE_OPTIMIZATION(t idx)` 额外在 `hint_key_state(..., NO_RANGE_HINT_ENUM, 0)` 剔除
- `find_best_ref` 遍历 `keys_in_use_for_query`，`NO_INDEX` 移除的索引不进 ref 候选
- `INDEX_MERGE` / `SKIP_SCAN` 用 `compound_hint_key_enabled`（`opt_hints.cc:932`）按键过滤

---

## 七、SET_VAR / MAX_EXECUTION_TIME / RESOURCE_GROUP

### 7.1 SET_VAR（语句级临时改变量）

`PT_hint_sys_var::contextualize`（`parse_tree_hints.cc:524`）校验变量存在且标了 `HINT_UPDATEABLE`，注册进 `Sys_var_hint`。

执行前 `update_vars`（`sql_parse.cc:3350`）、执行后 `restore_vars`（`:4879`）：

```cpp
void Sys_var_hint::restore_vars(THD *thd) {
  for (Hint_set_var *hint_var : var_list) {
    if (hint_var->save_value) {
      std::swap(var->value, hint_var->save_value);  // 换回旧值
      var->update(thd);
      std::swap(var->value, hint_var->save_value);  // 换回 hint 值供重复执行
    }
  }
}
```

用"记录旧值 → 语句末换回"的方式实现，PS 重复执行时也能正确恢复。从库不应用 SET_VAR hint（`thd->slave_thread` 检查）。

### 7.2 MAX_EXECUTION_TIME

仅 SELECT、非存储程序、非子查询（`parse_tree_hints.cc:497`），写 `lex->max_execution_time`。消费点（`sql_select.cc:164`）：

```cpp
return (thd->lex->max_execution_time ? thd->lex->max_execution_time
                                     : thd->variables.max_execution_time);
```

优先级：hint > session 变量 `max_execution_time`（默认 0 = 无超时）。

### 7.3 RESOURCE_GROUP

`PT_hint_resource_group::contextualize`（:566）写 `thd->resource_group_ctx()`，执行前 `switch_resource_group_if_needed`（`sql_prepare.cc:3659`），执行后恢复。

---

## 八、8.0.39 已废弃或未生效的 hint

| hint | 状态 |
|------|------|
| `MAX_STATEMENT_TIME` | 5.7.7 旧名，8.0 **已彻底移除**（全库 0 匹配），用 `MAX_EXECUTION_TIME` 替代 |
| `HASH_JOIN` / `NO_HASH_JOIN` | 枚举仍存在、语法器仍登记，但 **8.0.39 无任何代码消费**（grep `HASH_JOIN_HINT_ENUM` 无 `hint_table_state` 调用）。hash join 由 hypergraph 优化器按 `optimizer_switch hash_join` 决定，hint 基本不生效 |
| 旧式 `USE INDEX`/`FORCE INDEX`/`IGNORE INDEX` | 不是 `/*+ */` optimizer hint；若新式 INDEX hint 生效会被跳过（`sql_resolver.cc:1220`） |

---


## 参考

**内核月报**
- **2020/09《MySQL · Optimizer · Optimizer Hints》** —— hint 机制的四层结构、三类影响方式、系统价值（本文的直接来源）

**官方文档**
- *MySQL 8.0 Reference Manual → Optimizer Hints*（`/*+ */` 语法、各 hint 语义、`QB_NAME`）
- *MySQL 8.0 Reference Manual → Index-Level Optimizer Hints*（8.0.20+）

**相关文档**
- JOIN_ORDER 依赖如何被搜索遵守见 [`physical/06_join_order.md`](physical/06_join_order.md)
- INDEX 如何影响 range/ref 见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)、[`physical/07_access_method.md`](physical/07_access_method.md)
- SEMIJOIN/SUBQUERY 策略见 [`logical/03_semijoin.md`](logical/03_semijoin.md)、[`logical/02_subquery.md`](logical/02_subquery.md)

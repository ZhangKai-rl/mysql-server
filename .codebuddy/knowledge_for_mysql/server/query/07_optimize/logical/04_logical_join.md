# 04 逻辑优化：连接简化（外连接转内连接 + derived/view/CTE merge）

> 本篇覆盖 `simplify_joins`（外连接转内连接、ON 上提、嵌套 join 扁平化）与 `merge_derived`（派生表/视图/CTE 合并），全部算法级。

## 目录

- [零、阶段定位与调用链](#零阶段定位与调用链)
- [一、join nest 数据结构与不变式](#一join-nest-数据结构与不变式)
- [二、两个前置归一化](#二两个前置归一化)
- [三、simplify_joins Pass 1：外连接转内连接](#三simplify_joins-pass-1外连接转内连接)
- [四、simplify_joins Pass 2：嵌套 join 扁平化](#四simplify_joins-pass-2嵌套-join-扁平化)
- [五、derived / view / CTE merge](#五derived--view--cte-merge)
- [六、semi-join nest 收缩](#六semi-join-nest-收缩pull_out_semijoin_tables)
- [七、已知限制与"本版本不具备的能力"](#七已知限制与本版本不具备的能力)

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

## 六、semi-join nest 收缩（pull_out_semijoin_tables）

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

## 七、已知限制与"本版本不具备的能力"

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


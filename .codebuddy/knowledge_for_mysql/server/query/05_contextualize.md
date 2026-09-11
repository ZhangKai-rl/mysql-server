# 05 contextualize：Parse Tree → 逻辑查询树

> 本篇覆盖：Parse Tree 如何"落地"成 `LEX` + `Query_expression/Query_block` + `Item` 树（第 ①→② 层 IR 转换）。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [关键问题：contextualize 是谁的过程](#关键问题contextualize-是谁的过程)
- [一、调用链全景](#一调用链全景)
- [二、Query_block / Query_expression：创建还是填充](#二query_block--query_expression创建还是填充)
- [三、Item 与 itemize（为什么 Item 不走 contextualize）](#三item-与-itemize为什么-item-不走-contextualize)
- [四、递归的本质 + 逐函数算法讲解](#四递归的本质--逐函数算法讲解)
- [五、Query_term 树：8.0.31 的重构](#五query_term-树8031-的重构)
- [六、完整分步示例](#六完整分步示例)
- [七、will_contextualize 标志](#七will_contextualize-标志)
- [核心调用栈](#核心调用栈)

---

## 设计思想与理论基础

### 语法树如何"落地"成语义对象

contextualize 是编译原理三段式的**第三段（语义分析）**：把上下文无关的 Parse Tree，翻译成上下文相关的、可继续处理的逻辑树。

- Parse Tree 只知道"有个标识符叫 `a`"（语法层面）
- contextualize 把它变成 `Item_field`，并挂进 `Query_block::fields`（语义层面）

这在编译原理里对应 **AST（抽象语法树）的注解/翻译**阶段——语法树本身不携带语义，需要一次遍历，用符号表和上下文信息把它"充实"。

### 树的全景：MySQL 的"一批对象、状态递进"

理解 contextualize 之前，先要建立"MySQL 到底有几棵树"的全局观。**MySQL 不是"每阶段一棵新树"，而是"一批对象、状态递进"**——这是它和教科书数据库最大的不同：

| 阶段 | 树 | 载体 | 是"新树"还是"旧树升级" |
|------|----|------|----------------------|
| 语法分析（04 篇） | **Parse Tree** | `PT_*` / `PTI_*` | 新树，用完即弃 |
| 语义分析（本篇 contextualize） | **逻辑查询树 + Item 树** | `Query_expression`/`Query_block`/`Query_term` + `Item` | **新树诞生** |
| 语义绑定（06 篇 fix_fields） | **绑定的 Item 树** | 同一批 `Item`，`Item_field::field` 被填上 | **没有新树！旧树升级** |

**关键点**：contextualize 之后的"树"就是最终那批对象。`fix_fields` 不产生新树，它只是给这批 Item 树上的 `Item_field` 节点填上 `Field*` 指针、确定类型：

```
contextualize:  PTI_simple_ident_ident(a)（语法节点）──落地──▶ Item_field（field=nullptr, fixed=false）
                                                              │
fix_fields（06 篇）:                                           │ 就地填充，不换对象
                                                              ▼
                                                  Item_field（field=&t1.a, fixed=true）
                                                              ↑
                                          —— 同一个对象，只是状态位变了
```

**这就是"为什么 MySQL 没有独立 AST 层"的深层原因**：表达式对象 `Item` 本身就是 `Parse_tree_node`（`item.h:882`），从 parse 期创建 → contextualize 期归位 → fix_fields 期绑定 → 优化期改写 → 执行期求值，**自始至终是同一批对象，只是状态位在变**（`contextualized` → `fixed`）。

对比教科书：
- 教科书：语法树 → 语义注解产生**新的 AST** → 中间代码产生**新的 IR**
- MySQL：PT 树 → 落地成 QE/QB/Item 树（**这里才产生新树**）→ 之后全是**就地改写，不再产生新树**

> 所以完整 IR 链（README 的 6 层图）有 **5 棵主干树 + 2 棵旁路树，不是三棵**：
>
> - **语义阶段（04~06 篇）**：① Parse Tree（`PT_*`/`PTI_*`，用完即弃）、② 逻辑查询树（`Query_expression`/`Query_block`/`Query_term`）、②' Item 树（表达式，挂载在 QB 上）
> - **物理/执行阶段（07~09 篇）**：④ AccessPath 树、⑤ RowIterator 树 —— 这两棵是 optimize/执行阶段的**全新对象**，与语义阶段的树不是同一批
> - **旁路**：`SEL_ARG`/`SEL_ROOT`/`SEL_TREE`（range 区间森林）、`JOIN_TAB`/`QEP_TAB`（旧优化器局部结构）
>
> 所谓"一批对象、状态递进"**只适用于语义阶段的前三棵树**：PT 树 → 逻辑查询树是"换新树"（② 是 ① 落地出来的新对象），Item 树在 ②→③ 是"同一批对象升级"（fix_fields 就地填 `Field*`）。但 ④ AccessPath、⑤ RowIterator 是 optimize 阶段**另起的新树**，不是语义阶段那批对象的延续。本篇（04）只覆盖 ①→② 这一次"换树"。

### 为什么 Item 不走 contextualize，而走 itemize

这是本篇最体现设计品味的一点（第三节详述）：contextualize 处理**结构**（哪个 ident 是表、哪个是列、嵌套 SELECT 归到哪），Item 表达式树则用 **itemize** 单独处理。分工的动机：

- contextualize 面对的是 `PT` 节点的**树形递归**（表、列、子查询的归位）
- Item 表达式的绑定需要**类型信息、运算符重载、比较方向**，逻辑上独立，且 Item 树在 prepare 阶段还会被 `fix_fields` 二次处理

把两者分开，让"结构归位"和"表达式求值语义"各自保持简单——**单一职责**。

### 8.0.31 的 Query_term 重构：对齐 SQL 标准

8.0.31 引入了 `Query_term`（第五节），把 `UNION`/`INTERSECT`/`EXCEPT` 的组织从"链表套链表"改成标准的**二叉树**。动机是 SQL 标准的集合运算优先级（`INTERSECT` 高于 `UNION`）要求一棵能表达结合性的树，而不是平铺的链表。这是 MySQL 逐步向 SQL 标准收敛的一个例子——**历史包袱（5.x 的链表实现）在 8.0.31 被正视并修正**。

### 演进

- **5.x**：parse tree 与 Item 直接建，contextualize 的职责模糊
- **8.0**：contextualize 成为独立的、职责清晰的阶段（`sql_parse.cc` 里 `THD::sql_parser` 之后紧接 `contextualize`）
- **8.0.31**：`Query_term` 重构，集合运算表达对齐 SQL 标准

---

## 先有个整体印象

`contextualize` 的字面意思是"**使……进入上下文**"。

Parse Tree 是**上下文无关**的：它只知道"这里有个 `ident` 叫 `a`"，不知道 `a` 是哪张表的哪一列。

contextualize 做的事就是**把这棵树"落地"**：

| Parse Tree 里的 | contextualize 后变成 |
|-----------------|---------------------|
| 一个 `ident(a)` | `Item_field`，挂进 `Query_block::fields` |
| 一个 `ident(t)` | `Table_ref*`，挂进 `Query_block::m_table_list` |
| `WHERE` 后的表达式 | `Query_block::m_where_cond` |
| `ORDER BY` 列表 | `Query_block::order_list` |
| 嵌套的 SELECT | 新的 `Query_block` + `Query_expression`（挂到父块上） |
| `UNION` / `INTERSECT` | `Query_term` 树节点 |

**一句话**：contextualize 是"语法结构 → 语义对象"的翻译过程，产物是 LEX + QE/QB/Query_term + Item 树。

---

## 关键问题：contextualize 是谁的过程

> 用户提问："contextualize 是不是逻辑查询树和 parse tree 都有这个过程？"

**答案：只有 Parse Tree 有。逻辑查询树是产物，不是执行者。**

三条判定依据：

**1. `contextualize()` 是 `Parse_tree_node` 的虚函数**（`parse_tree_node_base.h:199`）。`Query_block` / `Query_expression` 根本**没有**这个方法 —— 你可以自己 grep 验证。

**2. 有个容易混淆的点：`Item` 也继承 `Parse_tree_node`**（`item.h:882`），所以表达式节点"技术上"也有 `contextualize()`。但 MySQL 把它**私有化并禁用**了：

```cpp
// sql/item.h:1138-1146
 private:
  /*
    Hide the contextualize*() functions: call/override the itemize()
    in Item class tree instead.
  */
  bool contextualize(Parse_context *) override {
    assert(0);
    return true;
  }
```

Item 的正门改叫 **`itemize()`**（`item.cc:631`），内部再调基类的 `contextualize()`。

**3. 逻辑查询树的"内容"是被 contextualize 填充的**，但树本身（顶层的 QE + QB）**在解析之前就已经建好了** —— 见下一节，这点很反直觉。

所以准确的调用关系是：

```
PT_select_stmt::make_cmd()                    ← Parse Tree 根节点
  └─ PT_query_expression::contextualize()
      └─ PT_query_specification::contextualize()
          ├─ 填充 Query_block（fields / table_list / where / order / group / limit）
          ├─ PT_table_factor_table_ident::contextualize()  → 建 Table_ref
          └─ Item::itemize()  → 内部调 Parse_tree_node::contextualize()
```

---

## 一、调用链全景

**入口不是 yacc，而是 parse 结束后的 `make_sql_cmd`**：

```
dispatch_command → ... → parse_sql                       sql_parse.cc:7067
  → THD::sql_parser()                                    sql_class.cc:3048
      → MYSQLparse(this, &root)                          sql_class.cc:3067   （Bison）
      → lex->make_sql_cmd(root)                          sql_class.cc:3077
          → parse_tree->make_cmd(thd)                    sql_lex.cc:4973
              → PT_select_stmt::make_cmd()               parse_tree_nodes.cc:705
                  → m_qe->contextualize(&pc)             parse_tree_nodes.cc:711
```

```cpp
// sql/sql_class.cc:3061
Parse_tree_root *root = nullptr;
if (MYSQLparse(this, &root) || is_error()) { cleanup_after_parse_error(); return true; }
if (root != nullptr && lex->make_sql_cmd(root)) return true;   // :3077
```

**`Parse_context`**：contextualize 的"上下文"载体，构造时初始化栈：

```cpp
// sql/parse_tree_node_base.cc:29
Parse_context::Parse_context(THD *thd_arg, Query_block *sl_arg)
    : thd(thd_arg), mem_root(thd->mem_root), select(sl_arg), m_stack(thd->mem_root) {
  m_stack.push_back(QueryLevel(thd->mem_root, SC_TOP));    // :34
}
```

`m_stack` 是 `QueryLevel` 的栈，用来在遍历中收集子 term（集合操作树靠它组装，见第五节）。

### 1.1 ParseContext：contextualize 的"环境数据"（逐行剖析）

官方注释（`parse_tree_node_base.h:127`）一句话定位：**"Environment data for the contextualization phase"**——它是贯穿整个 contextualize 递归的**环境对象**，把"当前在解析什么"的所有状态打包在一起：

```cpp
// sql/parse_tree_node_base.h:129-143
struct Parse_context {
  THD *const thd;                        ///< 当前线程句柄
  MEM_ROOT *mem_root;                    ///< 当前内存根
  Query_block *select;                   ///< 当前 Query_block（作用域指针）★
  mem_root_deque<QueryLevel> m_stack;    ///< 辅助 Query_term 树构建 ★
  bool finalize_query_expression();
  Parse_context(THD *thd, Query_block *sl);
  bool is_top_level_union_all(Surrounding_context op);
};
```

逐成员解释：

| 成员 | 类型 | 作用 | 谁在改它 |
|------|------|------|---------|
| `thd` | `THD *const` | const 指针，整个 contextualize 期间不变。用于报错（`syntax_error_at`）、查 `lex`、拿 `mem_root` | 从不改（const） |
| `mem_root` | `MEM_ROOT *` | 所有 contextualize 期 `new` 的对象（`Query_term`、`Item`、`Table_ref`）都从这里分配 | 构造时 `= thd->mem_root`，不改 |
| **`select`** | `Query_block *` | **"当前作用域指针"**——指向"现在正在填充哪个 QB"。所有 `pc->select->fields/table_list/...` 的写入都落在这个 QB 上 | 子查询切换作用域时改（`PT_subquery` 新建 inner_pc，或 `PT_query_expression` 切到 post-processing block） |
| **`m_stack`** | `mem_root_deque<QueryLevel>` | **Query_term 树的构建栈**。子节点 contextualize 完成后把自己 push 进父层 `m_elts`，父节点再取出组装 | 各节点的 contextualize 里 push/pop |

**关键洞察：`select` 和 `m_stack` 是两套正交的"上下文"**：
- `select` 回答"**这个表达式归哪个 QB**"（作用域语义）
- `m_stack` 回答"**这个 QB 在 Query_term 树里挂哪**"（结构组装）

两者解耦，所以"切作用域"（改 `select`）和"组装集合操作树"（改 `m_stack`）互不干扰——这是理解子查询和 UNION 处理的关键。

构造函数（`parse_tree_node_base.cc:29-35`）逐行：

```cpp
Parse_context::Parse_context(THD *thd_arg, Query_block *sl_arg)
    : thd(thd_arg),                      // ① 记 THD
      mem_root(thd->mem_root),           // ② mem_root 直接取 THD 的
      select(sl_arg),                    // ③ 初始作用域 = 传入的 QB
      m_stack(thd->mem_root) {           // ④ 栈也用 mem_root 分配
  m_stack.push_back(QueryLevel(thd->mem_root, SC_TOP));  // ⑤ 压入最底层的 SC_TOP
}
```

- **③** `select(sl_arg)`：传入的 `sl_arg` 就是 `lex_start` 预建好的**顶层 QB**（`make_cmd` 里 `thd->lex->current_query_block()`）。整个递归从"填充顶层 QB"开始。
- **⑤** 压入 `SC_TOP`：这是栈底"兜底层"，保证最后 `finalize_query_expression` 能从栈底取出**唯一一个根 Query_term**。

### 1.2 QueryLevel：Query_term 树的"层"（逐行剖析）

```cpp
// sql/parse_tree_node_base.h:111-125
struct QueryLevel {
  Surrounding_context m_type;              // 本层是什么语法上下文
  mem_root_deque<Query_term *> m_elts;     // 收集本层的子 Query_term ★
  bool m_has_order{false};                 // 本层是否带 ORDER BY
  QueryLevel(MEM_ROOT *mem_root, Surrounding_context sc, bool has_order = false)
      : m_type(sc), m_elts(mem_root), m_has_order(has_order) {}
};
```

逐成员解释：

| 成员 | 作用 | 举例 |
|------|------|------|
| `m_type` | 本层的语法上下文（`Surrounding_context` 枚举） | `SC_QUERY_EXPRESSION` / `SC_QUERY_SPECIFICATION` / `SC_SUBQUERY` / `SC_UNION_ALL` / `SC_TOP` ... |
| **`m_elts`** | **临时容器**：收集本层所有子 `Query_term`。子节点 contextualize 完成后 `push_back` 自己，父节点再从中取出组装 | 见下面"自底向上组装" |
| `m_has_order` | 本层是否带 `ORDER BY` | 用于 `is_top_level_union_all` 判断 UNION ALL 能否流式化 |

源码注释（`parse_tree_node_base.h:114-119`）说清了 `m_elts` 的四个用途：

> `m_elts` 在 contextualize 过程中充当临时容器，用于：
> 1. **收集子节点**：递归处理 Parse Tree 时，子节点把自己 push 到父节点的 elts 中
> 2. **延迟组合**：收集完所有子节点后，再决定如何组合成父节点
> 3. **支持多路操作**：UNION/INTERSECT/EXCEPT 可能有多个子节点（`A UNION B UNION C`）
> 4. **树构建**：最终从 elts 中取出节点，构建 Query_term 树

**"自底向上组装"的精髓**：`m_elts` 让 Query_term 树得以**后序构建**——子节点先完成（把自己 push 进父层），父节点后完成（从自己的 `m_elts` 取出子节点组装）。这对应 4.2 节 `PT_query_expression::contextualize` 里 `:1229` 的 `pc->m_stack.back().m_elts.push_back(pc->select)`。

`Surrounding_context` 枚举（`parse_tree_node_base.h:97-109`）：

```cpp
enum Surrounding_context {
  SC_TOP,                      // 栈底兜底层
  SC_TABLE_VALUE_CONSTRUCTOR,  // VALUES (...)
  SC_QUERY_EXPRESSION,         // query_expression（可带 ORDER/LIMIT）
  SC_SUBQUERY,                 // 子查询
  SC_UNION_DISTINCT, SC_UNION_ALL,
  SC_INTERSECT_DISTINCT, SC_INTERSECT_ALL,
  SC_EXCEPT_DISTINCT, SC_EXCEPT_ALL
};
```

每个"层"对应一个语法结构层级：压入 `SC_QUERY_EXPRESSION` 表示"进入一个 query expression"，压入 `SC_UNION_ALL` 表示"进入一个 UNION ALL 集合运算"。栈的深度 = SQL 嵌套深度。

### 1.3 finalize_query_expression：递归收尾挂根（逐行剖析）

```cpp
// sql/parse_tree_node_base.cc:42-51
bool Parse_context::finalize_query_expression() {
  QueryLevel ql = m_stack.back();                 // ① 取栈顶（最外层）
  m_stack.pop_back();                             // ② 弹出
  assert(ql.m_elts.size() == 1);                  // ③ 断言恰好收集到 1 个根
  Query_term *top = ql.m_elts.back();             // ④ 取出这个根
  top = top->pushdown_limit_order_by();           // ⑤ ORDER/LIMIT 下推到正确位置
  select->master_query_expression()->set_query_term(top);  // ⑥ 挂根
  if (top->validate_structure(nullptr)) return true;       // ⑦ 校验树结构
  return false;
}
```

逐行解释：

1. **① ②** 取栈顶——递归结束后，`m_stack` 应该只剩 `SC_TOP` 这一层（所有中间层都被各自的 contextualize pop 掉了）。
2. **③** `assert(ql.m_elts.size() == 1)`：**整条 SQL 必须收敛成一颗树**，所以最外层 `m_elts` 恰好 1 个根。多于 1 个说明有 Query_term 没被正确组装。
3. **⑤** `pushdown_limit_order_by()`（`query_term.cc:74`）：把挂在"查询表达式层"的 ORDER BY/LIMIT **下推**到正确的 Query_term 上（如 `SELECT ... UNION SELECT ... ORDER BY x` 的 ORDER BY 挂在最外层 set-op 上，而不是某个分支）。
4. **⑥** `set_query_term(top)`：**挂根**——把构建好的 Query_term 树根挂到 `Query_expression::m_query_term`。至此 Query_term 树完整。
5. **⑦** `validate_structure()`（`query_term.cc:162`）：校验树结构（set-op 的左右子树类型合法、unary 的括号合法等）。

**这个函数的本质**：contextualize 递归是"分散组装"（每个节点各自 push 子 term），`finalize_query_expression` 是"集中收尾"——取出唯一的根、下推 ORDER/LIMIT、挂到 QE、校验。**没有它，Query_term 树就只是一堆散在栈里的碎片。**

---

## 二、Query_block / Query_expression：创建还是填充

**结论：顶层是"解析前预建"，内容由 contextualize 填充；Query_term 树 100% 由 contextualize 创建。**

### 2.1 顶层 QE + QB 在解析前就建好了

```cpp
// sql/sql_lex.cc:511
bool lex_start(THD *thd) {
  LEX *lex = thd->lex;
  lex->thd = thd;
  lex->reset();
  thd->init_cost_model();
  const bool status = lex->new_top_level_query();      // :521
  assert(lex->current_query_block() == nullptr);
  lex->m_current_query_block = lex->query_block;       // :523
  return status;
}
```

```cpp
// sql/sql_lex.cc:780
bool LEX::new_top_level_query() {
  assert(unit == nullptr && query_block == nullptr);
  query_block = new_query(nullptr);                    // :789
  if (query_block == nullptr) return true;
  unit = query_block->master_query_expression();       // :792
  return false;
}
```

真正的 `new`：

```cpp
// sql/sql_lex.cc:598  LEX::create_query_expr_and_block
auto *const new_expression = new (thd->mem_root) Query_expression(ctx);   // :607
auto *const new_query_block =
    new (thd->mem_root) Query_block(thd->mem_root, where, having);        // :611
if (current_query_block != nullptr)
  new_expression->include_down(this, current_query_block);                // :616
new_query_block->include_down(this, new_expression);                      // :618
new_query_block->parent_lex = this;
new_query_block->include_in_global(&this->all_query_blocks_list);         // :621
new_expression->set_query_term(new_query_block);                          // :629
```

> 源码注释（`sql_lex.cc:511` 附近）自己都承认这是历史妥协："These objects should rather be created by the parser bottom-up."（理想情况应该由 parser 自底向上创建）

### 2.2 其它创建点

| 场景 | 创建位置 |
|------|---------|
| 子查询的 QE + QB | `PT_subquery::contextualize` `parse_tree_nodes.cc:4121` → `LEX::new_query` `sql_lex.cc:645` → 同一个 `create_query_expr_and_block` |
| UNION 第 2..N 个分支的 QB | `PT_set_operation::contextualize_setop` `:1663` → `LEX::new_set_operation_query` `sql_lex.cc:704` → `new_empty_query_block` `sql_lex.cc:587`（**复用同一个 QE**，只加兄弟 QB） |
| set-op 的 post-processing block | `Query_expression::create_post_processing_block` `sql_lex.cc:743` |
| **Query_term 树节点** | 只在 contextualize 中创建：`parse_tree_nodes.cc:1677/1680/1683`（union/except/intersect）、`:4037/4054/4076/4099`（unary） |

### 2.3 内容完全是 contextualize 填的

| 内容 | 填充位置 |
|------|---------|
| `fields` | `PT_select_item_list::contextualize` `parse_tree_nodes.cc:3615` |
| `context.table_list` | `PT_query_specification::contextualize` `:1192-1194` |
| `m_where_cond` / `m_having_cond` | `:1202` / `:1203` |
| `group_list` | `PT_group::contextualize` `:261` |
| `order_list` | `PT_order::contextualize` `:295` |
| `select_limit` / `offset_limit` | `PT_limit_clause::contextualize` `:3632` |
| `m_with_clause` | `PT_with_clause::contextualize` `:1948` |
| `m_query_term`（根） | `Parse_context::finalize_query_expression` `parse_tree_node_base.cc:48` |

### 2.4 QE/QB 完整结构展开（为什么这么绕）

> **命名历史**：`Query_block`/`Query_expression` 是 8.0 的命名。5.7 时代叫 `SELECT_LEX`/`SELECT_LEX_UNIT`（即 `st_select_lex`/`st_select_lex_unit`），8.0 重构时改成标准的"查询块/查询表达式"，与 SQL 标准的 \<query block\>/\<query expression\> 术语对齐（阿里云 PolarDB 文档《MySQL 8.0 Server 层架构的查询解析与优化原理》也专门提到了这个改名）。网上大量 5.7 资料写 `SELECT_LEX`，看到时别慌，就是现在的 `Query_block`。

QE/QB 之所以"绕"，是因为 MySQL 用**两套结构并存**来表达同一个查询：一套旧的 **master/slave/next 三指针链表**（表达嵌套 + 兄弟），一套新的 **m_query_term 树**（8.0.31 起，表达集合操作的嵌套）。先分清两者，再看完整例子。

**① 三指针链表（嵌套 + 兄弟）**——QE 和 QB 交替出现：

| 指针 | QB 指向 | QE 指向 |
|------|--------|---------|
| `master`（外层） | 外层 QE（`master_query_expression()`） | 外层 QB |
| `slave`（第一个内层） | 第一个子查询 QE（`first_inner_query_expression()`） | 第一个 QB（`first_query_block()`） |
| `next`（兄弟） | 同 QE 内下一个 QB（UNION 分支） | 同 QB 内下一个子查询 |

**② m_query_term 树（集合操作）**——表达 UNION/INTERSECT/EXCEPT 的嵌套与结合性（见本篇"五、Query_term 树"）。

**完整例子**：

```sql
SELECT * FROM t1
WHERE t1.a IN (SELECT * FROM t2 UNION SELECT * FROM t3)
UNION
SELECT * FROM t4
ORDER BY 1
```

展开成这样的结构（注意 QE 和 QB 的**交替嵌套**，以及同一层 UNION 分支用 `next` 串）：

```
LEX
└─ unit = QE1（顶层查询表达式：整个 UNION）
        │
        │ m_query_term（集合操作树）
        │   Query_term_union（UNION）
        │     ├─ 左：Query_term_unary ─► QB1（SELECT * FROM t1 WHERE ...）
        │     └─ 右：Query_term_unary ─► QB4（SELECT * FROM t4）
        │   + 一个 post-processing QB（承载 ORDER BY 1，挂在 union 节点上）
        │
        │ slave（first_query_block）
        ▼
   QB1（SELECT * FROM t1 WHERE t1.a IN (...)）
     ├─ fields = [ * 展开后的 Item_field 列表 ]
     ├─ m_table_list = [ Table_ref(t1) ]
     ├─ m_where_cond = Item_in_subselect
     │      └─ unit ──────────────┐
     ├─ master = QE1              │
     ├─ slave（第一个子查询）────────┘
     │
     ▼
   QE2（IN 子查询的 QE：SELECT * FROM t2 UNION SELECT * FROM t3）
     ├─ m_query_term = Query_term_union
     │     ├─ 左：QB2（SELECT * FROM t2）
     │     └─ 右：QB3（SELECT * FROM t3）
     ├─ master = QB1
     └─ slave = QB2
```

**读这张图的三条线索**：

1. **QE 和 QB 严格交替**：`LEX → QE1 → QB1 → QE2 → QB2`。QE 是"壳"（承载集合操作），QB 是"肉"（承载 FROM/WHERE/SELECT 列表）。子查询、UNION 分支都产生新的 QE/QB。

2. **UNION 的分支是"兄弟"**：QB1 和 QB4（顶层 UNION 的左右分支）不在同一棵 m_query_term 树上直接连，而是作为 `Query_term_union` 的左右子节点；旧结构里它们靠 `next` 指针串（`first_query_block()` 遍历）。

3. **子查询是"孩子"**：QB1 的 `slave` 指向 QE2（IN 子查询的 QE），而 `m_where_cond` 里的 `Item_in_subselect::unit` 也指向 QE2——**同一个 QE2 被两处引用**（一处是 QB1.slave 链表，一处是 Item 表达式的 unit 指针）。

**两套结构的关系**（这是最绕的一点）：`m_query_term` 树只管"集合操作的嵌套"（哪个 QB 是 UNION 的左分支），而 master/slave/next 管"整体嵌套"（子查询挂在哪、UNION 分支怎么串）。两者**互相独立、各自维护**，8.0.31 引入 m_query_term 是为了让集合操作能表达结合性（`INTERSECT` 优先于 `UNION`），但旧的 master/slave 链表为兼容没删——所以现在一个查询要同时维护两套结构。

### 2.5 Table_ref 的 join nest 树（table list 怎么组织）

除了 QE/QB 树，每个 QB 内部还有**另一棵独立的树**——FROM 子句的 join nest 树，载体是 `Table_ref`（`sql/table.h:2800`）。这就是阿里云文档提到的 "table list 组织"。

一个 QB 的 FROM 子句（如 `(A JOIN B ON ...) LEFT JOIN C ON ...`）里的表，**不是平铺链表，而是一棵嵌套 join 树**，靠几个指针组织：

| 指针 | 成员 | 作用 |
|------|------|------|
| `next_local` | `table.h:3496` | 同一 join nest 内的下一个表（兄弟） |
| `embedding` | `table.h:3774` | 外层 join nest（父） |
| `nested_join` | `table.h:3772` | 内层 join nest（子，若这是嵌套 join） |
| `next_leaf` | `table.h:3688` | leaf 表链表（`setup_tables` 后扁平遍历用） |
| `next_name_resolution_table` | `table.h:3576` | 名字解析链表（`fix_fields` 遍历用） |

例：`SELECT * FROM (A JOIN B ON A.x=B.x) LEFT JOIN C ON A.y=C.y`

```
QB1
└─ m_table_list（join nest 树）
     └─ Table_ref(LEFT JOIN 这个 nest 的"壳")
          ├─ embedding = nullptr（最外层）
          ├─ nested_join = NESTED_JOIN（内层 nest）
          │    ├─ m_tables = [ Table_ref(A), Table_ref(B) ]   ← next_local 串
          │    │                A.next_local = B
          │    └─ join_cond = A.x = B.x
          └─ next_local = Table_ref(C)                        ← 同层兄弟
               └─ join_cond = A.y = C.y
```

**关键：QE/QB 树和 Table_ref join nest 树是三个正交维度**：

- **QE/QB 树**表达"查询块的嵌套"（子查询、UNION 分支）——本篇 2.4 讲的
- **Table_ref join nest 树**表达"FROM 子句里 join 的嵌套"（`(A JOIN B) JOIN C` vs `A JOIN (B JOIN C)`）——本节讲的

同一个 QB 里，QE/QB 维度是"一个节点"（QB1），但 Table_ref 维度是一棵 join nest 树。前者回答"有几个 SELECT 块"，后者回答"这个 SELECT 的 FROM 里表怎么 join 的"。

> Table_ref 的完整结构（`position`/`ref` 行定位、leaf_tables、`map`/`tableno`）在 [`../handler.md`](../handler.md) 详讲，这里只讲它作为"join nest 树"的组织。

---

## 三、Item 与 itemize（为什么 Item 不走 contextualize）

### 3.1 继承关系

```cpp
// sql/item.h:882
class Item : public Parse_tree_node {
  typedef Parse_tree_node super;      // :883
```

**这是 MySQL 没有独立 AST 层的根因**：表达式在 parse 期创建的对象，一直用到执行期，只是状态在变（`contextualized` → `fixed`）。

#### 继承树全景图（两套树 + 一座桥）

contextualize 之所以"子类繁杂"，是因为它要处理**两套继承树**，中间靠 `PTI_*` 这座"桥"连接：

```
                 Parse_tree_node（模板基类，parse_tree_node_base.h:149）
                 └─ virtual contextualize(Context *pc)   ← 语义分析正门
                          │
        ┌─────────────────┴───────────────────────┐
        │                                          │
   【PT 结构树】                              【Item 表达式树】
   Parse_tree_root : Parse_tree_node         Item : Parse_tree_node (item.h:882)
   │  └─ make_cmd()（唯一接口）               │  （contextualize 被私有化禁掉，
   ├─ PT_select_stmt（根，产出 Sql_cmd）       │   正门改叫 itemize）
   ├─ PT_query_expression                    ├─ Item_func（运算符/函数）
   ├─ PT_query_specification                 │    └─ Item_func_gt/eq/lt/...
   ├─ PT_union / PT_except / PT_intersect    ├─ Item_field（列引用，fix_fields 绑定）
   ├─ PT_table_factor_*（表/派生表/JOIN）     ├─ Item_int / Item_string（常量）
   ├─ PT_order / PT_group / PT_limit          ├─ Item_ref（别名引用）
   │                                          ├─ Item_sum（聚合函数）
   │                                          └─ Item_subselect（子查询）
   │
   └─────────────── 桥：PTI_* ────────────────┘
      Parse_tree_item : Item（parse_tree_helpers.h:74，双重身份）
      ├─ PTI_simple_ident_ident   itemize → Item_field / Item_ref
      ├─ PTI_comp_op              itemize → Item_func_gt/eq/lt
      ├─ PTI_where / PTI_having   itemize → make_condition(...)
      └─ PTI_text_literal         （直接继承 Item_string，不是占位符）
```

**三句话看懂这张图**：

1. **两套树都从 `Parse_tree_node` 继承**，都有 `contextualize`，但 Item 把它私有化禁掉（`assert(0)`），改走 `itemize`——这就是"为什么 Item 不走 contextualize"的答案。
2. **`PTI_*` 是"占位符"**：语法动作里 `new PTI_comp_op(...)` 只是记下"左操作数 + 比较符 + 右操作数"，到 `itemize` 时才决定换成 `Item_func_gt` 还是 `Item_func_eq`。它同时是语法节点（PT）和表达式占位（Item）。
3. **`PTI_text_literal`、`PTI_count_sym` 是例外**：它们直接继承 `Item_string`/`Item_sum_count`，parse 期就是最终对象，不经过"占位符 → 变身"。

#### itemize 的分发调用链（占位符如何"变身"）

以 `WHERE b > 1` 为例，看一个 `PTI_*` 占位符怎么一路分发成具体 Item：

```
语法动作：NEW_PTN PTI_comp_op(b, GT, 1)          sql_yacc.yy:10265
   │
   ▼
PTI_comp_op::itemize(pc, &res)                   parse_tree_items.cc:166
   ├─ super::itemize(pc, res)        ──▶ Item::itemize（注册进 THD::item_list）
   ├─ left->itemize(pc, &left)       ──▶ PTI_simple_ident_ident(b)::itemize
   │                                        └─ new Item_field(..., "b")   ← 左操作数落地
   ├─ right->itemize(pc, &right)     ──▶ Item_int(1)::itemize（常量，直接）
   └─ *res = (*boolfunc2creator)(...)->create(left, right)   :171
             └─ new Item_func_gt(Item_field(b), Item_int(1))  ★ 丢弃 PTI_comp_op
```

**关键点**：`PTI_comp_op` 这个占位符在 `itemize` 末尾被**丢弃**，`*res` 换成新建的 `Item_func_gt`。这就是"占位符 → 真实 Item"的典型模式——`PTI_*` 节点是"待定类型的壳"，itemize 决定壳里装什么，然后把壳换掉。对比 `PTI_simple_ident_ident` 分发到 `Item_field`（列）或 `Item_ref`（别名），取决于上下文。

### 3.2 `Item::itemize` 是正门

```cpp
// sql/item.cc:631
bool Item::itemize(Parse_context *pc, Item **res) {
  if (skip_itemize(res)) return false;          // :632  非 parser 造的 Item 跳过
  if (super::contextualize(pc)) return true;    // :633  栈溢出检查 + 打 contextualized 标记

  // Add item to global list
  pc->thd->add_item(this);                      // :636  挂进 THD::item_list，便于统一释放
  if (pc->select) {                             // :641
    const enum_parsing_context place = pc->select->parsing_place;
    if (place == CTX_SELECT_LIST || place == CTX_HAVING ||
        place == CTX_ORDER_BY) {
      pc->select->select_n_having_items++;      // :645
    }
  }
  return false;
}
```

**`skip_itemize` 的依据**是 `is_parser_item`（`item.h:3483`）：

- `Item(const POS&)`（parser 专用构造）→ **true**（`item.cc:196`）
- `Item()` / `Item(THD*, const Item*)` → **false**（`item.cc:141 / 168`，这两个构造里已经 `add_item`）

**为什么要有这个区分**：优化器、执行器在运行期也会 new 出 Item（比如等值传播、临时表字段），这些不是 parser 产物，不该再走 itemize。

---

## 四、递归的本质 + 逐函数算法讲解

### 4.0 先回答"是递归吗"：是，而且是**两套正交的递归**

`contextualize` 的"递归"不是一套，而是**两套叠加**：

```
第一套：Query_term 树的递归（骨架）
  PT_query_expression → PT_query_specification / PT_union / PT_subquery ...
  方向：自顶向下调用，自底向上组装（子节点完成时 push 进父层的 m_elts）
  产物：Query_term 树（Query_block 叶子 + Query_term_unary/set_op 内部节点）

第二套：Item 树的递归（血肉）
  Item_func → 遍历 args[] → Item_field / Item_int ...
  方向：纯自顶向下（itemize 递归子表达式）
  产物：Item 表达式树
```

**两者关系**：`contextualize` 负责"结构层"（query expression / query block / union / from / order），每当遇到一个**表达式槽位**（select list、WHERE、HAVING、ORDER BY 项），就调用 `itemize` 切到第二套递归，在当前 `pc->select` 作用域内构建 Item 树。

一句话：**`contextualize` 决定"作用域是谁"（当前 QB），`itemize` 决定"表达式长什么样"（Item 树）。**

---

### 4.1 总入口 `PT_select_stmt::make_cmd`（`parse_tree_nodes.cc:705`）

注意：它**不是** `contextualize`，而是 `make_cmd`（被 `LEX::make_sql_cmd` 调用），是整条递归的**发起者**：

```cpp
Sql_cmd *PT_select_stmt::make_cmd(THD *thd) {
  Parse_context pc(thd, thd->lex->current_query_block());   // :706  ← 用预建的顶层 QB 当初始作用域
  thd->lex->sql_command = m_sql_command;                    // :708  ← 填 LEX 层

  if (m_qe->contextualize(&pc)) return nullptr;             // :711  ← 递归真正入口
  ...
  if (pc.finalize_query_expression()) return nullptr;       // :726  ← 递归结束后收尾挂根
  ...
  if (thd->lex->sql_command == SQLCOM_SELECT)
    return new (thd->mem_root) Sql_cmd_select(thd->lex->result);   // :750
  else
    return new (thd->mem_root) Sql_cmd_do(nullptr);
}
```

逐段解释：

1. **`:706`** `Parse_context pc(thd, thd->lex->current_query_block())`：`current_query_block()` 返回的就是 `lex_start` 阶段预建好的顶层 QB。**这个 QB 就是整个递归的"初始作用域"**——之后所有 `pc->select` 默认都指向它。
2. **`:708`** 先填 LEX 层的 `sql_command`（这是"六层产物"里 LEX 层的填充）。
3. **`:711`** `m_qe->contextualize(&pc)`：递归真正开始。`m_qe` 是 `PT_query_expression`（Parse Tree 的根），从这里往下走。
4. **`:726`** `pc.finalize_query_expression()`：递归结束后，把 `m_stack` 栈底收集到的唯一一个 `Query_term` 挂到 `Query_expression` 上（见 4.2 末）。
5. **`:750`** 用 `thd->lex->result` 构造 `Sql_cmd_select`——注意此时 LEX/QE/QB/Item 树**已经被填好了**，`make_cmd` 只是把它们包成命令对象返回。

---

### 4.2 `PT_query_expression::contextualize`（`:3988`）

```cpp
bool PT_query_expression::contextualize(Parse_context *pc) {
  pc->m_stack.push_back(
      QueryLevel(pc->mem_root, SC_QUERY_EXPRESSION, m_order != nullptr)); // :3989
  if (contextualize_safe(pc, m_with_clause)) return true;                 // :3991

  if (Parse_tree_node::contextualize(pc) || m_body->contextualize(pc))    // :3995
    return true;

  QueryLevel ql = pc->m_stack.back();                                     // :3998
  Query_term *expr = ql.m_elts.back();                                    // :3999
  pc->m_stack.pop_back();                                                 // :4000

  switch (expr->term_type()) {
    case QT_UNARY:       // :4004  括号化 qe：建 post-processing block 挂 ORDER/LIMIT
    case QT_QUERY_BLOCK: // :4051  → contextualize_order_and_limit()（:4052）
    case QT_UNION: case QT_EXCEPT: case QT_INTERSECT:  // :4068-4104
  }
  return false;
}
```

逐段解释：

1. **`:3989-3990`** 压入 `SC_QUERY_EXPRESSION` 层，`m_has_order = (m_order != nullptr)`——记录"本层是否带 ORDER BY"，用于判断 UNION ALL 能否流式化。
2. **`:3991`** 若有 `WITH` 先处理（本例无）。
3. **`:3995`** 两个调用：
   - `Parse_tree_node::contextualize(pc)`：**显式调基类默认实现**，给当前节点打"已 contextualize"标记（防止重复处理）。
   - `m_body->contextualize(pc)`：递归处理 body。对普通 SELECT，body 是 `PT_query_specification`。
4. **`:3998-4000`** body 处理完后，栈顶 `m_elts` 里已经被子节点 push 了一个 `Query_term`（本例就是那个 QB 自身）。取出来、弹出本层。
5. **`:4051-4065`** `QT_QUERY_BLOCK` 分支：调 `contextualize_order_and_limit` 处理外层 `ORDER BY`/`LIMIT`，然后把 QB push 回上层。

> 关键：对**单查询块**，这里的 QB 就是直接 `pc->select`（不新建）；`Query_term` 树最终只有一层。

---

### 4.3 `PT_query_specification::contextualize`（`:1170`）—— 最核心的填充函数

```cpp
bool PT_query_specification::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;                              // :1171
  pc->m_stack.push_back(QueryLevel(pc->mem_root, SC_QUERY_SPECIFICATION)); // :1172
  pc->select->parsing_place = CTX_SELECT_LIST;                            // :1173

  if (options.save_to(pc)) return true;                                   // :1180

  if (item_list->contextualize(pc)) return true;                          // :1182

  assert(pc->select->parsing_place == CTX_SELECT_LIST);
  pc->select->parsing_place = CTX_NONE;                                   // :1186

  if (!from_clause.empty()) {                                             // :1190
    if (contextualize_array(pc, &from_clause)) return true;               // :1191
    pc->select->context.table_list =                                      // :1192
        pc->select->context.first_name_resolution_table =
            pc->select->get_table_list();
  }

  if (itemize_safe(pc, &opt_where_clause) ||                              // :1197
      contextualize_safe(pc, opt_group_clause) ||
      itemize_safe(pc, &opt_having_clause))
    return true;

  pc->select->set_where_cond(opt_where_clause);                           // :1202
  pc->select->set_having_cond(opt_having_clause);                         // :1203

  pc->select->parsing_place = CTX_SELECT_LIST;                            // :1223
  if (contextualize_safe(pc, opt_window_clause)) return true;             // :1224
  pc->select->parsing_place = CTX_NONE;                                   // :1225

  QueryLevel ql = pc->m_stack.back();                                     // :1227
  pc->m_stack.pop_back();                                                 // :1228
  pc->m_stack.back().m_elts.push_back(pc->select);                        // :1229
  return (opt_hints != nullptr ? opt_hints->contextualize(pc) : false);   // :1230
}
```

**这是理解"一步步构建什么"的关键函数**，逐段解释：

1. **`:1171`** `super::contextualize`：`PT_query_primary` 是空类没有 override，一路继承到基类默认 no-op（只打标记）。
2. **`:1172`** 压入 `SC_QUERY_SPECIFICATION` 层。
3. **`:1173`** `parsing_place = CTX_SELECT_LIST`：**告诉后续 itemize"我正处于 select list 上下文"**。这是一个"模式开关"——Item 靠它统计 `select_n_having_items`、决定名字解析行为。
4. **`:1180`** `options.save_to(pc)`：把语法层收集的 SELECT 选项（DISTINCT、SQL_CACHE 等）写入 `pc->select` 的 base/active options。
5. **`:1182`** `item_list->contextualize(pc)`：select list 里每个项走 itemize（见 4.5），结果填进 `fields`。**这就是"构建 select list"这一步**。
6. **`:1186`** 恢复 `parsing_place = CTX_NONE`（对称复位，保证下一次 itemize 前状态正确）。
7. **`:1190-1194`** 处理 FROM：`contextualize_array` 逐个调 `from_clause` 里每个 `PT_table_reference::contextualize`，然后**把刚建好的 `table_list` 同时挂到 `context.table_list` 和 `context.first_name_resolution_table`**（后者是名字解析的入口，见 06 篇）。
8. **`:1197-1203`** 处理 WHERE/GROUP/HAVING：`itemize_safe` 递归建 Item 树，`set_where_cond`/`set_having_cond` 把结果写进 QB。
9. **`:1227-1229`** 收尾：弹出本层，**把自己填好的 `pc->select` push 进父层 `m_elts`**——这是 Query_term 树"自底向上组装"的关键动作。

> 注意：**整个函数里没有任何 `new Query_block`**。它纯粹往 `pc->select`（现成的 QB）里填字段。这印证了第二节的结论：QB 是预建的空壳，contextualize 只做填充。

---

### 4.4 表引用 `PT_table_factor_table_ident::contextualize`（`:3639`）

```cpp
bool PT_table_factor_table_ident::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;

  THD *thd = pc->thd;
  Yacc_state *yyps = &thd->m_parser_state->m_yacc;

  m_table_ref = pc->select->add_table_to_list(                         // :3645
      thd, table_ident, opt_table_alias, 0, yyps->m_lock_type, yyps->m_mdl_type,
      opt_key_definition, opt_use_partition, nullptr, pc);
  if (m_table_ref == nullptr) return true;                             // :3648
  if (pc->select->add_joined_table(m_table_ref)) return true;          // :3649
  return false;
}
```

**关键：`add_table_to_list()`（`sql_parse.cc:5943`）才是真正 `new Table_ref` 的地方**：

```cpp
// sql/sql_parse.cc:5984-5995（关键片段）
Table_ref *ptr = new (thd->mem_root) Table_ref;   // ← new 在这里
...
ptr->query_block = this;
ptr->table_name = table_name->table.str;
ptr->alias = alias_str;
ptr->is_alias = alias != nullptr;
...
```

逐段解释：

1. `add_table_to_list` 负责：校验表名、解析库名/CTE、`new Table_ref`、填 `query_block/table_name/alias/db`、设 tableno 和锁，并链入 `table_list` 链表。
2. `add_joined_table()`（`sql_parse.cc:6321`）把 `Table_ref` 挂进当前 join nest：`m_current_table_nest->push_front(table)` + 设 `join_list` / `embedding`。

其余表引用节点速查（机制相同，不再展开）：

| 节点 | 位置 | 做什么 |
|------|------|--------|
| `PT_derived_table` | `:1331` | 切 `CTX_DERIVED`（`:1334`）→ 递归 contextualize 子查询（`:1349`）→ `new Table_ident(unit)`（`:1359`）→ `add_table_to_list`（`:1362`）→ 派生列名（`:1365`）→ LATERAL 标记（`:1366`） |
| `PT_joined_table_on` | `:3679` | `push_new_name_resolution_context`（`:3682`）→ `parsing_place=CTX_ON`（`:3689`）→ `on->itemize`（`:3691`）→ `add_join_on`（`:3698`） |
| `PT_cross_join` | `:3673` | `nest_last_join(thd)`（`:3675`） |

---

### 4.5 表达式槽位：itemize（第二套递归）

select list 里的 `a` 被 Bison 包成 `PTI_expr_with_alias`，其 `itemize`（`parse_tree_items.cc:335`）：

```cpp
bool PTI_expr_with_alias::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || expr->itemize(pc, &expr)) return true;   // :336

  if (alias.str) {                                                        // :338
    ...
    expr->item_name.copy(alias.str, alias.length, ...);                   // :344
  } else if (!expr->item_name.is_set()) {
    expr->item_name.copy(expr_loc.start, (uint)(expr_loc.end - expr_loc.start), ...);
  }
  *res = expr;                                                            // :349
  return false;
}
```

逐段解释：

1. **`:336`** 两个调用：
   - `super::itemize` → `Item::itemize`（`item.cc:631`）：把 wrapper 自身注册进全局 item 列表 + 打标记。
   - `expr->itemize(pc, &expr)`：**递归进真正的表达式**。对 `a`，`expr` 是 `Item_ident/Item_field`；对 `c > 5`，`expr` 是 `Item_func_gt`。
2. **`:338-347`** 处理别名/列名：有别名拷别名，没别名用原始文本生成 `item_name`。
3. **`:349`** `*res = expr`：**替换输出指针**——把外层拿到的 wrapper 替换成真正的表达式 Item。**这是 itemize 与 contextualize 的关键区别：itemize 可以"换掉自己"（返回不同对象）。**

WHERE 的 `PTI_context::itemize`（`parse_tree_items.cc:684`），展示"作用域上下文切换 + 递归"：

```cpp
bool PTI_context::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;

  pc->select->parsing_place = m_parsing_place;   // :687  临时设 CTX_WHERE

  if (expr->itemize(pc, &expr)) return true;     // :689  递归 itemize c > 5

  if (!expr->is_bool_func()) {                   // :691
    expr = make_condition(pc, expr);             // :692  包成 expr <> 0 谓词
  }
  ...
  pc->select->parsing_place = CTX_NONE;          // :698  复位
  expr->apply_is_true();                         // :700

  *res = expr;
  return false;
}
```

逐段解释：

1. **`:687`** 临时把 `parsing_place` 设为 `CTX_WHERE`——让表达式里的 `Item_field` 知道自己处于 WHERE（影响名字解析/统计）。
2. **`:689`** `expr->itemize` 递归。`Item_func::itemize`（`item_func.cc:352`）会遍历 `args[]`，对每个子表达式（`Item_field`、`Item_int`）递归 itemize。**这就是第二套递归的载体**。
3. **`:691-694`** 若表达式不是布尔函数，`make_condition` 把它包成 `expr <> 0` 转成谓词。
4. **`:700`** `apply_is_true()`：标记"作为布尔条件使用"。

---

### 4.6 ORDER BY `PT_order::contextualize`（`:288`）

```cpp
bool PT_order::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;

  pc->select->parsing_place = CTX_ORDER_BY;      // :291
  if (order_list->contextualize(pc)) return true; // :294
  pc->select->order_list = order_list->value;     // :295  填 QB.order_list
  if (pc->select->parsing_place == CTX_ORDER_BY)
    pc->select->parsing_place = CTX_NONE;         // :298-299
  return false;
}
```

对单查询块，`PT_query_expression::contextualize_order_and_limit` 走**可吸收分支**（`can_absorb_order_and_limit` 恒 true），直接在当前 QB 里 itemize `ORDER BY a` 并 `:295` 写进 `order_list`。**不需要新建"假 QB"**（UNION 场景才需要）。

---

### 4.7 作用域切换：子查询

子查询的核心问题是：内层表达式要填进**内层的 QB**，而不是外层。MySQL 的做法是**新建一个 `Parse_context` 对象**（不是改原 `pc`）：

```cpp
bool PT_subquery::contextualize(Parse_context *pc) {
  ...
  Query_block *child = lex->new_query(pc->select);   // :4121  新建 QE + QB
  ...
  Parse_context inner_pc(pc->thd, child);            // :4124  新作用域
  inner_pc.m_stack.push_back(QueryLevel(pc->mem_root, SC_SUBQUERY));  // :4125
  ...
  if (qe->contextualize(&inner_pc)) return true;      // :4129  用 inner_pc 递归
  ...
  if (inner_pc.finalize_query_expression()) return true;  // :4137
  lex->pop_context();                                 // :4141  恢复名字解析上下文
}
```

逐段解释：

1. **`:4121`** `lex->new_query(pc->select)`：基于"外层当前 QB"创建一个**新的 QE + QB**（`new Query_block` 在这里发生），并建立 master/slave 关系。
2. **`:4124`** `Parse_context inner_pc(pc->thd, child)`：**换作用域的本质——`inner_pc.select` 指向 child QB**。子查询内部的 `pc->select` 全是 child。
3. **`:4125`** 新 context 压入 `SC_SUBQUERY` 层（这层会阻止某些优化，如 UNION ALL 流式化）。
4. **`:4129`** 用 `inner_pc` 递归处理子查询。
5. **`:4141`** 回到外层后 `lex->pop_context()` 恢复名字解析上下文（见下）。

名字解析上下文栈 `push_context`/`pop_context`（`sql_lex.h:4506`）维护 `LEX::context_stack`，用于**相关子查询找外层列**。典型切换（UNION 的 ORDER BY/LIMIT 挂在结果上时）：

```cpp
Query_block *orig_query_block = pc->select;      // 保存外层 QB
pc->select = ex->query_block();                  // 切到"假 QB"(post-processing block)
lex->push_context(&pc->select->context);         // 压入新名字解析上下文
... 处理 ORDER BY / LIMIT ...
lex->pop_context();                              // 弹出
pc->select = orig_query_block;                   // 恢复
```

---

### 4.8 一句话总结"一步步构建什么"

```
lex_start 预建空壳：LEX + QE1 + QB1（new 在 sql_lex.cc:607/611）
        │
        ▼  PT_select_stmt::make_cmd 发起递归
        │
        ▼  PT_query_specification::contextualize（自顶向下）
        │     ├─ options.save_to         → QB1.base_options
        │     ├─ item_list->contextualize → QB1.fields（itemize 递归建 Item 树）
        │     ├─ from_clause             → QB1.table_list（new Table_ref）
        │     ├─ opt_where_clause        → QB1.m_where_cond（Item 树）
        │     └─ 把自己 push 进父层 m_elts（自底向上组装 Query_term 树）
        │
        ▼  PT_query_expression::contextualize 收尾
        │     └─ contextualize_order_and_limit → QB1.order_list
        │
        ▼  pc.finalize_query_expression()
              └─ QE1.set_query_term(QB1)   ← 挂根
```

产物是**六层**：LEX、QE、QB、Query_term、Table_ref、Item 树——都是同一批对象逐步填充/组装的结果，最后得到可交给 prepare/optimize 的逻辑查询树。

---

## 五、Query_term 树：8.0.31 的重构

### 5.0 先定位：Query_term 是什么，和 QE / QB 什么关系

**一句话**：QE 是"一条完整查询"，Query_term 树是"这条查询里集合运算的结构"，QB 是"最小的查询单元（单个 SELECT）"。三者是**嵌套**关系，不是平级：

```
Query_expression（QE）            = <query expression> = 整条查询（含最外层 ORDER BY / LIMIT）
  └─ m_query_term（Query_term 树根，sql_lex.h:786）  = <query expression body>
        ├─ 叶子节点 = Query_block（QB）= <query specification>（单个 SELECT ... FROM ... WHERE）
        └─ 内部节点 = Query_term_set_op（UNION/INTERSECT/EXCEPT）+ Query_term_unary（括号 + 该层 ORDER/LIMIT）
```

**这些名字来自 SQL 标准的 query 语法层级**。`query_term.h:51-79` 的注释引了完整语法定义，这是理解一切的基础（`INTERSECT` 绑定比 `UNION`/`EXCEPT` 更紧）：

```bnf
<query expression> ::=
    [ <with clause> ] <query expression body>
    [ <order by clause> ] [ <limit/offset> ]

<query expression body> ::=
     <query term>
   | <query expression body> UNION  [ ALL | DISTINCT ] <query term>
   | <query expression body> EXCEPT [ ALL | DISTINCT ] <query term>

<query term> ::=
     <query primary>
   | <query expression body> INTERSECT [ ALL | DISTINCT ] <query primary>

<query primary> ::=
     <simple table>
   | ( <query expression body> [ <order by clause> ] [ <limit/offset> ] )

<simple table> ::=
     <query specification>         -- 普通 SELECT ...
   | <table value constructor>     -- VALUES (...)
   | <explicit table>              -- TABLE t
```

**完整映射：SQL 语法 → MySQL 类**（不只是"三个层级"，而是 6 层 + 三种 leaf）：

| SQL 语法层级 | MySQL 类 | 粒度 |
|-------------|---------|------|
| `<query expression>` | `Query_expression`（QE） | 语句级（含 WITH + 最外层 ORDER/LIMIT） |
| `<query expression body>` | `Query_term` 树（`QE.m_query_term`） | body 级（集合运算的嵌套） |
| `<query term>` | `Query_term`（抽象基类） | 项级（5 种节点类型） |
| `<query primary>` | `Query_term_unary`（括号化 body）+ simple table | 无独立类 |
| `<simple table>` | 无独立类 | 三种 leaf 的统称 |
| `<query specification>` | `Query_block`（`QT_QUERY_BLOCK`） | SELECT 级 |
| `<table value constructor>` | `Query_block`（`is_table_value_constructor`） | VALUES |
| `<explicit table>` | `Query_block` | `TABLE t` |

**关键洞察 0：`Query_block` 承担三种 leaf 角色**

这是最反直觉的一点——**`Query_block` 不只是"SELECT 查询块"，它同时是 `<query specification>`、`<table value constructor>`、`<explicit table>` 三种语法成分的载体**（`query_term.h:36-38` 注释原文，`sql_lex.h:1304` 说 "a query block, **aka a query specification**"）。三种 leaf 靠标志区分，而非不同的类：

| leaf | 语法 | 区分标志 | 例子 |
|------|------|---------|------|
| **query specification** | `SELECT ...` | 默认 | `SELECT a FROM t` |
| **table value constructor** | `VALUES (...)` | `is_table_value_constructor = true`（`sql_lex.h:2259`） | `VALUES (1,'a'),(2,'b')` |
| **explicit table** | `TABLE t`（8.0.19） | 等价 `SELECT * FROM t` 的简写 | `TABLE t` |

所以"query specification"**不是和 QB 并列的另一个东西，而是 QB 的默认形态**——`Query_block` 这个类名里的 "block" 就是"查询块"，它的官方定义（`sql_lex.h:1304-1306`）就是 "a query consisting of a SELECT keyword, followed by a table list, optionally followed by WHERE / GROUP BY, etc."。只有 `VALUES (...)` 和 `TABLE t` 这两种"非 SELECT"的 simple table 才需要额外标志（`is_table_value_constructor`）或命令类型来区分。

> 源码佐证：`VALUES` 在 prepare/optimize 里被特殊对待——`Query_block::prepare` 第 1 步 `is_table_value_constructor → prepare_values()`（`sql_resolver.cc:190`）；`sql_executor.cc:2977`、`sql_resolver.cc:5511` 都对 `is_table_value_constructor` 单独分支。

**关键点 1：QE "拥有" Query_term 树，Query_term 树"由" QB 组成**。所以关系是"容器 → 树 → 叶子"。

**关键点 2：简单查询时没有 Query_term 超结构**。`SELECT * FROM t1`（无集合运算）时，`QE.m_query_term` 直接就是一个 QB（`is_simple()` 返回 true，`sql_lex.h:919`）：

```
SELECT * FROM t1                    →  QE.m_query_term = QB（直接是叶子，无 set_op）
SELECT * FROM t1 UNION SELECT ...   →  QE.m_query_term = Query_term_union（两个 QB 叶子）
```

**关键点 3：QB 有双重身份**（`query_term.h:115-122`）：
1. 作为 Query_term 树的**叶子**（代表 query specification）
2. 作为非叶子节点（set_op/unary）的 **companion Query_block**（承载该层的 ORDER BY/LIMIT），通过 `Query_term::query_block()` 访问

**关键点 4：Query_term 只表达"集合运算的嵌套"，不表达"子查询嵌套"**。子查询（`WHERE x IN (SELECT ...)`）的嵌套靠旧的 master/slave 指针表达（2.4 节），Query_term 树不管这个。

**一个具体例子**：

```sql
SELECT a FROM t1 UNION SELECT b FROM t2 ORDER BY a
```

```
QE（整条语句）
 ├─ m_query_term = Query_term_union（QT_UNION）
 │     ├─ 左叶子：QB1（SELECT a FROM t1）
 │     └─ 右叶子：QB2（SELECT b FROM t2）
 └─ 最外层 "ORDER BY a" → 挂在 QE 的 post_processing block（`query_block()` 返回的 companion QB）
```

**为什么会有"Query_term 是什么"的困惑**：因为 QE/QB 是 5.x 就有的老结构，Query_term 是 8.0.31 才引入的新结构。三者不是平级关系，而是**嵌套关系**——QE 包着 Query_term 树，Query_term 树的叶子是 QB。但直觉上容易以为 QE/QB/Query_term 是三个并列的"查询表示"，实际上 QE 和 Query_term 是"容器 vs 内容"，Query_term 和 QB 是"树 vs 叶子"。

### 5.1 类层次（`sql/query_term.h`）

```
Query_term                     (抽象)     :211
├─ Query_block                            sql_lex.h:1311，term_type()=QT_QUERY_BLOCK :1325
└─ Query_term_set_op           (抽象)     :405
   ├─ Query_term_union                    :498   QT_UNARY? 否：QT_UNION :505
   ├─ Query_term_intersect                :511   QT_INTERSECT :518
   ├─ Query_term_except                   :524   QT_EXCEPT :531
   └─ Query_term_unary                    :557   QT_UNARY :570
```

- 类型枚举 `Query_term_type`：`query_term.h:89-102`
- **`Query_expression` 不是 `Query_term`**，它持有根：`Query_term *m_query_term{nullptr}`（`sql_lex.h:786`）

### 5.2 组装过程

载体是 `Parse_context::m_stack`，元素是 `QueryLevel`（`parse_tree_node_base.h:111-125`，`m_elts` 是 `mem_root_deque<QueryTerm*>`）。

> ⚠️ 8.0.39 **没有** `add_query_term()` 这个函数，子节点收集一律用 `m_elts.push_back(...)`。

```cpp
// sql/parse_tree_nodes.cc:1654  PT_set_operation::contextualize_setop
  pc->m_stack.push_back(QueryLevel(pc->mem_root, context));       // :1657
  if (m_lhs->contextualize(pc)) return true;                      // :1660
  pc->select = pc->thd->lex->new_set_operation_query(pc->select); // :1663 第 2 个 QB
  if (pc->select == nullptr || m_rhs->contextualize(pc)) return true;  // :1665

  Query_term_set_op *setop = nullptr;
  switch (setop_type) {
    case QT_UNION:     setop = new (pc->mem_root) Query_term_union(pc->mem_root);     break;  // :1677
    case QT_EXCEPT:    setop = new (pc->mem_root) Query_term_except(pc->mem_root);    break;  // :1680
    case QT_INTERSECT: setop = new (pc->mem_root) Query_term_intersect(pc->mem_root); break;  // :1683
  }
  merge_descendants(pc, setop, ql);        // :1690  N 元扁平化（定义 :1516）
  setop->label_children();                 // :1691
  Query_expression *qe = pc->select->master_query_expression();
  if (setop->set_block(qe->create_post_processing_block(setop))) return true;  // :1694
  pc->m_stack.back().m_elts.push_back(setop);   // :1695
```

收尾：

```cpp
// sql/parse_tree_node_base.cc:42
bool Parse_context::finalize_query_expression() {
  QueryLevel ql = m_stack.back();
  m_stack.pop_back();
  assert(ql.m_elts.size() == 1);
  Query_term *top = ql.m_elts.back();
  top = top->pushdown_limit_order_by();                     // :47  query_term.cc:74
  select->master_query_expression()->set_query_term(top);   // :48
  if (top->validate_structure(nullptr)) return true;        // :49  query_term.cc:162
  return false;
}
```

### 5.3 这次重构解决什么问题

`sql/query_term.h:33-49` 的官方说明：

> 最初 MySQL 只支持 UNION，用"一个 `Query_expression` 通过 next 指针拥有多个 `Query_block`"来隐式表达。这很简单，但限制了我们：
> - 只能是**左深嵌套的 UNION**
> - 无法表达 **INTERSECT / EXCEPT**
> - 无法表达**多层 `<query primary>` 嵌套 ORDER BY/LIMIT**，例如
>   `(((SELECT a,b FROM t) ORDER BY a LIMIT 5) ORDER BY -b LIMIT 3) ORDER BY a;`

新设计的关键：**每个非叶节点都带一个 companion `Query_block`**（`Query_term_set_op::m_block` `:410`）来承载该层的 ORDER BY / LIMIT，从而支持多层。旧的 `fake_query_block` 已被移除（`query_term.h:183-185`）。

---

## 六、完整分步示例

`SELECT a FROM t WHERE b > 1 ORDER BY c`

### Step 0：解析前，LEX 已预建顶层 QE/QB

`lex_start()` → `new_top_level_query()` → `create_query_expr_and_block()`：

- `QE1 = new Query_expression`（`sql_lex.cc:607`）
- `QB1 = new Query_block`（`sql_lex.cc:611`）
- `QE1->set_query_term(QB1)`（`:629`）
- `lex->unit = QE1`、`lex->query_block = QB1`、`m_current_query_block = QB1`

此时 QB1 是空壳：`fields` 空、`m_table_list` 空、`m_where_cond = nullptr`。

### Step 1：词法

`MYSQLlex`（`sql_lex.cc:1304`）产出 token 流：
`SELECT, ident(a), FROM, ident(t), WHERE, ident(b), '>', NUM(1), ORDER, BY, ident(c)`

### Step 2：归约出 Parse Tree

见 [04 篇](04_parser.md#示例一条-sql-产生哪些-pt-节点)：得到 `PT_select_stmt → PT_query_expression → PT_query_specification → {...}`

### Step 3~4：`THD::sql_parser` → `make_sql_cmd` → `PT_select_stmt::make_cmd`

`Parse_context pc(thd, QB1)`（`parse_tree_nodes.cc:706`），栈初始为 `{ QueryLevel(SC_TOP) }`

### Step 5~6：`PT_query_specification::contextualize`

1. `parsing_place = CTX_SELECT_LIST`（`:1173`）
2. `item_list->contextualize`（`:1182`）→ `PT_select_item_list`（`:3613`）→ **`QB1->fields = [Item_field(a)]`**
   - 其中 `PTI_simple_ident_ident::itemize`（`parse_tree_items.cc:353`）→ `new Item_field(POS(), NullS, NullS, "a")`（`:379`）
   - 注意：此时 `Item_field` 只有名字，**`field == nullptr`**（未绑定真实列）
3. `from_clause` → `PT_table_factor_table_ident::contextualize`（`:3639`）→ `add_table_to_list()` 建 **`Table_ref(t)`**
4. `QB1->context.table_list = QB1->get_table_list()`（`:1192-1194`）
5. `itemize_safe(&opt_where_clause)`（`:1197`）
   - `PTI_where` → `PTI_context::itemize`（`:684`）：`parsing_place = CTX_WHERE`（`:687`）
   - `PTI_comp_op::itemize`（`:166`）：左右分别 itemize 成 `Item_field(b)` / `Item_int(1)`，然后 **`*res = Item_func_gt(left, right)`**（`:171`）
6. `QB1->set_where_cond(Item_func_gt)`（`:1202`）
7. 弹栈，把 `QB1` 交给父层 `m_elts`（`:1227-1229`）

### Step 7：回到 `PT_query_expression::contextualize`（`QT_QUERY_BLOCK`，`:4051`）

`contextualize_order_and_limit()`（`:4052` → `:1255`）：

- `PT_order::contextualize`（`:288`）：`parsing_place = CTX_ORDER_BY`（`:291`）→ `PTI(c)->itemize` ⇒ `Item_field(c)` → **`QB1->order_list = [ORDER{Item_field(c), ASC}]`**（`:295`）

栈顶是 `SC_TOP`，所以**不**额外包 `Query_term_unary`（`:4053` 条件不成立）

### Step 8：`pc.finalize_query_expression()`

- `QE1->set_query_term(QB1)`（`parse_tree_node_base.cc:48`）
- `validate_structure()`（`:49`）

### Step 9：生成 Sql_cmd

```cpp
// parse_tree_nodes.cc:750
return new (thd->mem_root) Sql_cmd_select(thd->lex->result);
```
存入 `lex->m_sql_cmd`（`sql_lex.cc:4973`）

### 最终对象图

```
LEX
├─ unit                  = QE1
├─ query_block           = QB1
├─ all_query_blocks_list = { QB1 }
├─ sql_command           = SQLCOM_SELECT
└─ m_sql_cmd             = Sql_cmd_select*

QE1 : Query_expression
├─ m_query_term       = QB1      （is_simple() == true）
└─ first_query_block() = QB1

QB1 : Query_block : public Query_term
├─ term_type()   = QT_QUERY_BLOCK
├─ fields        = [ Item_field(a) ]        ← 未 fix_fields，field 仍为 nullptr
├─ m_table_list  = [ Table_ref(t) ]
├─ m_where_cond  = Item_func_gt(Item_field(b), Item_int(1))
├─ order_list    = [ ORDER{ Item_field(c), ASC } ]
└─ parsing_place = CTX_NONE

Parse Tree (PT_select_stmt → ...)  ⇒ 从此无人引用，随 mem_root 回收
```

**注意最后一行**：`Item_field` 现在只有名字字符串，真正的列绑定（`Field*`）要到 **prepare 阶段的 `fix_fields()`** 才完成 —— 那是 [06 篇](06_resolver_prepare.md) 的内容。

---

## 七、will_contextualize 标志

```cpp
// sql/sql_lex.h:4368
/**
  Used to inform the parser whether it should contextualize the parse
  tree. When we get a pure parser this will not be needed.
*/
bool will_contextualize;      // :4372
```

检查点：

| 位置 | 场景 |
|------|------|
| `sql_yacc.yy:233` | `CONTEXTUALIZE(x)` |
| `sql_yacc.yy:242` | `CONTEXTUALIZE_VIEW(x)` |
| `sql_yacc.yy:256` | `ITEMIZE(x, y)` |
| `parse_tree_helpers.h:172` | `contextualize_array` |
| `sql_lex.cc:4970` | `make_sql_cmd` 开头直接 return |

**现状**：初值 true（`sql_lex.cc:3604`），8.0.39 服务端代码**没有任何地方置 false**，唯一赋值在单测（`unittest/gunit/character_set_deprecation-t.cc:44`）。这是为"未来纯 parser"预留的兼容开关。

---

## 核心调用栈

```
THD::sql_parser()                              sql_class.cc:3048
 ├─ MYSQLparse(this, &root)                    sql_class.cc:3067   → ① Parse Tree
 └─ lex->make_sql_cmd(root)                    sql_class.cc:3077
     └─ LEX::make_sql_cmd()                    sql_lex.cc:4969
         └─ parse_tree->make_cmd(thd)          sql_lex.cc:4973
             └─ PT_select_stmt::make_cmd()     parse_tree_nodes.cc:705
                 ├─ Parse_context pc(thd, QB1)              :706
                 ├─ m_qe->contextualize(&pc)                :711
                 │   └─ PT_query_expression::contextualize  :3988
                 │       └─ PT_query_specification::contextualize  :1170
                 │           ├─ item_list->contextualize     :1182 → fields
                 │           ├─ contextualize_array(from)    :1191 → Table_ref
                 │           ├─ itemize_safe(where)          :1197 → Item 树
                 │           └─ m_stack.back().m_elts.push_back(QB1)  :1229
                 ├─ contextualize_order_and_limit()          :4052 → order_list
                 └─ pc.finalize_query_expression()           :726
                     └─ Parse_context::finalize_query_expression  parse_tree_node_base.cc:42
                         └─ QE1->set_query_term(QB1)                :48

【子查询】PT_subquery::contextualize          parse_tree_nodes.cc:4109
           └─ lex->new_query()                sql_lex.cc:645
【集合操作】PT_set_operation::contextualize_setop  parse_tree_nodes.cc:1654
           └─ new Query_term_union/except/intersect  :1677-1683
```

---


## 参考

**书籍**
- **Aho & Ullman《Principles of Compiler Design》（龙书）** —— 语义分析、符号表。MySQL 的 `contextualize()` 对应这一阶段

**官方文档**
- *MySQL 8.0 Reference Manual → SELECT Statement*（8.0.31 起 `Query_term` 重构，支持嵌套集合操作）
- [阿里云 PolarDB《MySQL 8.0 Server 层架构的查询解析与优化原理》](https://help.aliyun.com/zh/polardb/polardb-for-mysql/architecture-of-mysql-server-in-mysql-8) —— QE/QB 树组织、`SELECT_LEX`→`Query_block` 改名历史、prepare/rewrite/optimize 三阶段总览（本篇 2.4/2.5 的 QE/QB 结构与 Table_ref join nest 树展开对齐此文）


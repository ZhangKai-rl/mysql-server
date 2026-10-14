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
- [八、PT_* 大家族全景（403 个类）](#八pt_-大家族全景)
- [九、PTI_* 占位符（已迁入专篇）](#九pti_-占位符已迁入专篇)
- [十、QB / QE / QT 成员全景与生命周期（对象视角）](#十query_block--query_expression--query_term-成员全景与生命周期对象视角)（★ 8 个表达式槽位彼此独立 / 三条链 / `Query_term` 两分支继承 / PS 重执行四步 / 从不调析构）（★ 36 个类的完整剖析已迁入专篇 [`11_parse_tree.md`](11_parse_tree.md) 第十章）
  - [36 个类的完整清单 / 变身目标 / 五种变身模式 / "为什么需要占位符"归因表](11_parse_tree.md#十pti_-解析期占位-item-全景36-个类逐个讲透)（★ 已迁入专篇；本章只保留"何时被消费、由谁遍历"的流程视角）
  - [8.1 规模与范畴](#81-规模与范畴一句话)（★ ≈403 个类 / 17 个范畴；完整分类大表 + 代表实现见 [`11_parse_tree.md`](11_parse_tree.md) 3.1）
  - [8.4 override 统计](#84--override-统计两套正交递归的量化证据)（contextualize ≈162 vs itemize 92）
  - [8.5 树的形状](#85--树的形状一个完整-select-的-pt-树)（★ 完整 SELECT 的 PT 树实例 + 深度 5→9→15）
  - [8.7 生命周期](#87-生命周期pt-节点一次性消费)（★ 四种丢弃机制；`item_list` 里只有 Item）
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
> 所谓"一批对象、状态递进"**只适用于语义阶段的前三棵树**：PT 树 → 逻辑查询树是"换新树"（② 是 ① 落地出来的新对象），Item 树在 ②→③ 是"同一批对象升级"（fix_fields 就地填 `Field*`）。但 ④ AccessPath、⑤ RowIterator 是 optimize 阶段**另起的新树**，不是语义阶段那批对象的延续。本篇（05 contextualize）只覆盖 ①→② 这一次"换树"。

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

⚠️ **勘误**：`struct Parse_context` **不在** `sql_lex.h`，在 **`sql/parse_tree_node_base.h:126`**；三个成员函数全在 **`sql/parse_tree_node_base.cc`**（`:29` / `:42` / `:53`）。

```cpp
// sql/parse_tree_node_base.h:126
/**
  Environment data for the contextualization phase
*/
struct Parse_context {
  THD *const thd;                      ///< Current thread handler
  MEM_ROOT *mem_root;                  ///< Current MEM_ROOT
  Query_block *select;                 ///< Current Query_block object
  mem_root_deque<QueryLevel> m_stack;  ///< Aids query term tree construction
  /// Call upon parse completion.
  /// @returns true on error, else false
  bool finalize_query_expression();
  Parse_context(THD *thd, Query_block *sl);
  bool is_top_level_union_all(
      Surrounding_context op);  ///< Determine if there is anything but
                                ///< UNION ALL above in m_stack
};
```

```cpp
// sql/parse_tree_node_base.cc:29
Parse_context::Parse_context(THD *thd_arg, Query_block *sl_arg)
    : thd(thd_arg),
      mem_root(thd->mem_root),
      select(sl_arg),
      m_stack(thd->mem_root) {
  m_stack.push_back(QueryLevel(thd->mem_root, SC_TOP));
  // ★★ 栈底永远是 SC_TOP —— 保证任何 pop 都不会空栈
}

// :53
bool Parse_context::is_top_level_union_all(Surrounding_context op) {
  if (op == SC_EXCEPT_ALL || op == SC_INTERSECT_ALL) return false;
  assert(op == SC_UNION_ALL);
  for (size_t i = m_stack.size(); i > 0; i--) {      // ★ 自顶向下扫整条栈
    switch (m_stack[i - 1].m_type) {
      case SC_UNION_DISTINCT:
      case SC_INTERSECT_DISTINCT:
      case SC_INTERSECT_ALL:
      case SC_EXCEPT_DISTINCT:
      case SC_EXCEPT_ALL:
      case SC_SUBQUERY:
        return false;
      case SC_QUERY_EXPRESSION:
        // Ordering above this level in the context stack (syntactically
        // outside) precludes streaming of UNION ALL.
        if (m_stack[i - 1].m_has_order) return false;
        [[fallthrough]];
      default:;
    }
  }
  return true;
}
```

> ★ `is_top_level_union_all()` 是 `merge_descendants()` 里"要不要把下层 UNION ALL 折叠上来"的决策依据：能流式输出就不折叠（见 §5.2）。

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
// sql/parse_tree_node_base.h:97
class Query_term;
enum Surrounding_context {
  SC_TOP,
  SC_QUERY_SPECIFICATION,
  SC_TABLE_VALUE_CONSTRUCTOR,
  SC_QUERY_EXPRESSION,
  SC_SUBQUERY,
  SC_UNION_DISTINCT,
  SC_UNION_ALL,
  SC_INTERSECT_DISTINCT,
  SC_INTERSECT_ALL,
  SC_EXCEPT_DISTINCT,
  SC_EXCEPT_ALL
};

struct QueryLevel {
  Surrounding_context m_type;
  /** elts: elements 的缩写
   * elts 在 contextualize 过程中充当临时容器，用于：
     收集子节点: 在递归处理 Parse Tree 时，子节点将自己 push 到父节点的 elts 中
     延迟组合: 收集完所有子节点后，再决定如何组合成父节点
     支持多路操作: 对于 UNION/INTERSECT/EXCEPT，可能有多个子节点（如 A UNION B UNION C）
     树构建: 最终从 elts 中取出节点，构建 Query_term 树
   */
  mem_root_deque<Query_term *> m_elts;
  bool m_has_order{false};
  QueryLevel(MEM_ROOT *mem_root, Surrounding_context sc, bool has_order = false)
      : m_type(sc), m_elts(mem_root), m_has_order(has_order) {}
};
```

> ★ `m_has_order` 唯一的置值处是 `PT_query_expression::contextualize` 开头的 `m_order != nullptr`（`:3990`）——它决定上层 `is_top_level_union_all()` 是否因为"外面有 ORDER BY 会物化"而否决流式 UNION ALL。

**`finalize_query_expression()`（`:42`）——contextualize 的"交卷"动作**：

```cpp
/**
  Set the parsed query expression's query term. For its construction, see
  parse_tree_nodes.cc's contextualize methods. Query_term is documented in
  query_term.h .
*/
bool Parse_context::finalize_query_expression() {
  QueryLevel ql = m_stack.back();                          // ★ 取栈顶
  m_stack.pop_back();
  assert(ql.m_elts.size() == 1);                           // ★★ 一个 QE 最终只能收敛出 1 个根
  Query_term *top = ql.m_elts.back();
  top = top->pushdown_limit_order_by();                    // ★ 折叠冗余 unary + 建 m_parent
  select->master_query_expression()->set_query_term(top);  // ★★ 挂到 Query_expression
  if (top->validate_structure(nullptr)) return true;
  return false;
}
```

> ★★ **contextualize 结束时 Query_term 树尚未建成**——只把元素进/出 `m_stack`；真正的定型在这三步：`pushdown_limit_order_by()` → `set_query_term()` → `validate_structure()`。

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

⚠️ **勘误**：8.0.39 已无 `TABLE_LIST`，类改名 **`Table_ref`**（`sql/table.h:2800`）；`NESTED_JOIN` **不在** `table.h`，而在独立的 **`sql/nested_join.h:78`**；它的子节点容器叫 **`m_tables`**，不是 `join_list`（`Table_ref::join_list` 是指向父 nest 那个 deque 的**回指**）。

```cpp
// sql/nested_join.h:70
/**
  Struct NESTED_JOIN is used to represent how tables are connected through
  outer join operations and semi-join operations to form a query block.
  Out of the parser, inner joins are also represented by NESTED_JOIN
  structs, but these are later flattened out by simplify_joins().
*/
struct NESTED_JOIN {
  NESTED_JOIN() : m_tables(*THR_MALLOC), sj_outer_exprs(*THR_MALLOC),
                  sj_inner_exprs(*THR_MALLOC) {}

  /// list of tables referenced in the nested join
  mem_root_deque<Table_ref *> m_tables;      // ★★ 子节点容器
  table_map used_tables{0};        /* bitmap of tables in the nested join */
  table_map not_null_tables{0};    /* tables that rejects nulls           */
  plan_idx first_nested{0};
  bool natural_join_processed{false};
  uint nj_total{0};
  uint nj_counter{0};
  /// Bit identifying this nested join. Only nested joins representing the
  /// outer join structure need this ...
  nested_join_map nj_map{0};
  table_map sj_depends_on{0};
  table_map sj_corr_tables{0};
  uint query_block_id{0};
  uint sj_enabled_strategies{0};
  mem_root_deque<Item *> sj_outer_exprs, sj_inner_exprs;
  Semijoin_mat_optimize sjm;
};
```

**`Table_ref` 上构成树的四个指针**（`sql/table.h`）：

```cpp
  /// Is non-NULL if this table reference is a nested join ...
  NESTED_JOIN *nested_join{nullptr};                // ★ 非空 ⇒ 本节点是树枝（nest），不是叶子表
  /// The nested join containing this table reference.
  Table_ref *embedding{nullptr};                    // ★ 指向父 nest
  /// The join list immediately containing this table reference
  mem_root_deque<Table_ref *> *join_list{nullptr};  // ★ 指向父 nest 的 NESTED_JOIN::m_tables
  /// True if right argument of LEFT JOIN ...
  /// (RIGHT JOINs are converted to LEFT JOIN during contextualization)
  bool outer_join{false};
```

**两条并行链表**（`table.h:3488`）：

```cpp
  /*
    List of tables local to a subquery or the top-level SELECT (used by
    SQL_I_List). Considers views as leaves (unlike 'next_leaf' below).
    Created at parse time in Query_block::add_table_to_list() ->
    table_list.link_in_list().
  */
  Table_ref *next_local{nullptr};                   // ★ 侵入式链表，头在 Query_block::m_table_list.first

  /* link in a global list of all queries tables */
  Table_ref *next_global{nullptr}, **prev_global{nullptr};

  /*
    List of nodes in a nested join tree, that should be considered as
    leaves with respect to name resolution. The leaves are: views,
    top-most nodes representing NATURAL/USING joins, subqueries, and
    base tables. ...
  */
  Table_ref *next_name_resolution_table{nullptr};
  // ★ 名字解析专用链：初始与 next_local 完全重合，
  //   遇 NATURAL/USING 时被 store_top_level_join_columns() 改写
```

**`Query_block::add_table_to_list()`（定义居然在 `sql/sql_parse.cc:5945`，不在 `sql_lex.cc`）**——挂链表的部分（`:6100`）：

```cpp
  /* Store the table reference preceding the current one. */
  if (m_table_list.elements > 0) {
    // 侵入式链表：由"链表尾指针"反算前一个节点
    previous_table_ref =
        (Table_ref *)((char *)m_table_list.next -
                      ((char *)&(ptr->next_local) - (char *)ptr));
    previous_table_ref->next_name_resolution_table = ptr;   // ★ 名字解析链先接上
  }

  m_table_list.link_in_list(ptr, &ptr->next_local);   // ★ 挂进 Query_block 的 FROM 链
  ptr->next_name_resolution_table = nullptr;
  ptr->partition_names = partition_names;
  lex->add_to_query_tables(ptr);                      // ★ 同时挂进 LEX 全局表链（next_global）
  ...
  return ptr;
}
```

**join nest 的开关栈**（`sql_parse.cc:6192`）：

```cpp
bool Query_block::init_nested_join(THD *thd) {   // 进入 '('
  Table_ref *const ptr = Table_ref::new_nested_join(
      thd->mem_root, "(nested_join)", embedding, m_current_table_nest, this);
  if (ptr == nullptr) return true;

  m_current_table_nest->push_front(ptr);              // ★ 挂进父 deque（push_front ⇒ 子节点倒序存放）
  embedding = ptr;                                    // ★ 下压栈
  m_current_table_nest = &ptr->nested_join->m_tables; // ★ 当前待填充 deque 换成新 nest
  return false;
}

Table_ref *Query_block::end_nested_join() {           // 离开 ')'
  ptr = embedding;
  m_current_table_nest = ptr->join_list;              // ★ 弹栈：回到父 deque
  embedding = ptr->embedding;                         // ★ 弹栈：回到父 nest
  nested_join = ptr->nested_join;
  if (nested_join->m_tables.size() == 1) {
    // ★ 只有一个孩子 ⇒ nest 无意义，把孩子提上来，nest 消失
    Table_ref *embedded = nested_join->m_tables.front();
    m_current_table_nest->pop_front();
    embedded->join_list = m_current_table_nest;
    embedded->embedding = embedding;
    m_current_table_nest->push_front(embedded);
    ptr = embedded;
  } else if (nested_join->m_tables.empty()) {
    m_current_table_nest->pop_front();
    ptr = nullptr;
  }
  return ptr;
}
```

**`PT_table_factor_table_ident::contextualize()`（`:3639`）——表引用进树的落点**：

```cpp
bool PT_table_factor_table_ident::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;

  THD *thd = pc->thd;
  Yacc_state *yyps = &thd->m_parser_state->m_yacc;

  m_table_ref = pc->select->add_table_to_list(      // ★★ PT → Table_ref 的唯一落点
      thd, table_ident, opt_table_alias, 0, yyps->m_lock_type, yyps->m_mdl_type,
      opt_key_definition, opt_use_partition, nullptr, pc);
  if (m_table_ref == nullptr) return true;
  if (pc->select->add_joined_table(m_table_ref)) return true;  // ★ 挂进当前 nest
  return false;
}
```

同族几个（`parse_tree_nodes.cc:3653-3715`）：

```cpp
// (a, b) → 把最后 N 个表收成一个 join nest
bool PT_table_reference_list_parens::contextualize(Parse_context *pc) {
  if (super::contextualize(pc) || contextualize_array(pc, &table_list))
    return true;
  assert(table_list.size() >= 2);
  m_table_ref = pc->select->nest_last_join(pc->thd, table_list.size());
  return m_table_ref == nullptr;
}

bool PT_joined_table_on::contextualize(Parse_context *pc) {
  if (this->contextualize_tabs(pc)) return true;
  if (push_new_name_resolution_context(pc, this->m_left_table_ref,
                                       this->m_right_table_ref)) {
    this->error(pc, this->m_join_pos);
    return true;
  }
  Query_block *sel = pc->select;
  sel->parsing_place = CTX_ON;                 // ★ ON 子句的解析位置
  if (super::contextualize(pc) || on->itemize(pc, &on)) return true;
  if (!on->is_bool_func()) { on = make_condition(pc, on); if (on == nullptr) return true; }
  add_join_on(this->m_right_table_ref, on);
  pc->thd->lex->pop_context();
  sel->parsing_place = CTX_NONE;
  m_table_ref = pc->select->nest_last_join(pc->thd);
  return m_table_ref == nullptr;
}
```

> ⚠️ **`SELECT_LEX` 在 8.0.39 已彻底不存在**——全仓无任何 typedef / using / #define 兼容别名。`sql_lex.h:1307` 的注释写着"这个类之前叫 SELECT_LEX"；而 `Select_lex_visitor`（`sql_lex.h:120`）是访问器类，**不是别名**。

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

**完整源码**（`sql/parse_tree_nodes.cc:705-753`）：

```cpp
Sql_cmd *PT_select_stmt::make_cmd(THD *thd) {
  Parse_context pc(thd, thd->lex->current_query_block());
  // ★ 构造 Parse_context；构造函数里 m_stack.push_back(QueryLevel(SC_TOP))

  thd->lex->sql_command = m_sql_command;

  if (m_qe->contextualize(&pc)) {          // ★★ 整棵 Parse Tree 在这里被 contextualize
    return nullptr;
  }

  const bool has_into_clause_inside_query_block = thd->lex->result != nullptr;

  if (has_into_clause_inside_query_block && m_into != nullptr) {
    my_error(ER_MULTIPLE_INTO_CLAUSES, MYF(0));
    return nullptr;
  }
  if (contextualize_safe(&pc, m_into)) {   // ★ nullptr 安全的变参版，见 §4.5
    return nullptr;
  }

  if (pc.finalize_query_expression()) return nullptr;
  // ★★ Query_term 树在此定型（pushdown_limit_order_by → set_query_term → validate_structure）

  if (m_into != nullptr && m_has_trailing_locking_clauses) {
    push_warning(thd, ER_WARN_DEPRECATED_INNER_INTO);
  } else if (has_into_clause_inside_query_block &&
             thd->lex->unit->is_set_operation()) {
    if (!m_qe->has_trailing_into_clause()) {
      push_warning(thd, ER_WARN_DEPRECATED_INNER_INTO);
    } else if (m_has_trailing_locking_clauses) {
      push_warning(thd, ER_WARN_DEPRECATED_INNER_INTO);
    }
  }

  DBUG_EXECUTE_IF("ast", Query_term *qn =
                             pc.select->master_query_expression()->query_term();
                  std::ostringstream buf; qn->debugPrint(0, buf);
                  DBUG_PRINT("ast", ("\n%s", buf.str().c_str())););
  // ★ debug 金钥匙：SET SESSION debug = 'd,ast:O,/tmp/mysqld.trace'

  // ★★ Sql_cmd 在 contextualize 全部完成之后才创建
  if (thd->lex->sql_command == SQLCOM_SELECT)
    return new (thd->mem_root) Sql_cmd_select(thd->lex->result);
  else  // (thd->lex->sql_command == SQLCOM_DO)
    return new (thd->mem_root) Sql_cmd_do(nullptr);
}
```

> ⚠️ **勘误**：常见说法"先建 `Sql_cmd_select` 再 contextualize"是**反的**——`Sql_cmd` 是最后一步，且 DO 语句返回的是 `Sql_cmd_do`。
>
> ⚠️ **`finalize_query_expression()` 不是 `Query_expression` 的方法**，是 `Parse_context::finalize_query_expression()`，定义在 `sql/parse_tree_node_base.cc:42`。

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

⚠️ **勘误**：`PT_query_expression::contextualize()` **不调** `finalize_query_expression()`——它只负责把元素推进/弹出 `m_stack`，收尾由 `make_cmd` / `PT_subquery` 统一调用。

**完整源码**（`sql/parse_tree_nodes.cc:3988-4107`）：

```cpp
bool PT_query_expression::contextualize(Parse_context *pc) {
  pc->m_stack.push_back(                       // ★ 进栈：本 QE 一层
      QueryLevel(pc->mem_root, SC_QUERY_EXPRESSION, m_order != nullptr));
  if (contextualize_safe(pc, m_with_clause)) return true;

  if (Parse_tree_node::contextualize(pc) || m_body->contextualize(pc))
    return true;                               // ★ 递归进 body

  QueryLevel ql = pc->m_stack.back();
  Query_term *expr = ql.m_elts.back();         // ★ body 的产出落在栈顶这层
  pc->m_stack.pop_back();

  switch (expr->term_type()) {                 // ★ 按 body 类型分三种组装方式
    case QT_UNARY: {                           // (SELECT ...) 带括号
      Query_term_unary *ex = down_cast<Query_term_unary *>(expr);
      Query_expression *qe = pc->select->master_query_expression();

      // The setting of no_table_names_allowed in the created post processing
      // block below to false foreshadows our removing the parentheses in
      // Query_term::pushdown_limit_order_by. ...
      if (ex->set_block(qe->create_post_processing_block(ex))) return true;

      if (m_order != nullptr)
        ex->query_block()->order_list = m_order->order_list->value;
      if (m_limit != nullptr) {
        ex->query_block()->select_limit = m_limit->limit_options.limit;
        ex->query_block()->offset_limit = m_limit->limit_options.opt_offset;
      }

      Query_block *orig_query_block = pc->select;      // ★ 作用域切换：保存
      LEX *lex = pc->thd->lex;
      pc->select = ex->query_block();
      lex->push_context(&pc->select->context);
      assert(pc->select->parsing_place == CTX_NONE);
      if (contextualize_safe(pc, m_order, m_limit)) return true;
      lex->pop_context();
      pc->select = orig_query_block;                  // ★ 恢复

      if (pc->m_stack.back().m_type == SC_QUERY_EXPRESSION) {  // 3 层以上
        expr = new (pc->mem_root) Query_term_unary(pc->mem_root, expr);
        if (expr == nullptr) return true;
      }
      QueryLevel upper = pc->m_stack.back();
      pc->m_stack.pop_back();
      ql.m_elts.pop_back();
      if (upper.m_type == SC_UNION_DISTINCT || upper.m_type == SC_UNION_ALL ||
          upper.m_type == SC_EXCEPT_DISTINCT || upper.m_type == SC_EXCEPT_ALL ||
          upper.m_type == SC_INTERSECT_DISTINCT ||
          upper.m_type == SC_INTERSECT_ALL)
        ql = upper;                            // ★ 外层是集合操作 ⇒ 合并进那一层
      ql.m_elts.push_back(expr);
      pc->m_stack.push_back(ql);
    } break;
    case QT_QUERY_BLOCK: {                     // 最普通的单 SELECT
      if (contextualize_order_and_limit(pc)) return true;
      if (pc->m_stack.back().m_type == SC_QUERY_EXPRESSION) {  // 2 层以上
        expr = new (pc->mem_root) Query_term_unary(pc->mem_root, expr);
        if (expr == nullptr) return true;
      }
      QueryLevel upper = pc->m_stack.back();
      pc->m_stack.pop_back();
      ql.m_elts.pop_back();
      if (upper.m_type == SC_UNION_DISTINCT || upper.m_type == SC_UNION_ALL ||
          upper.m_type == SC_EXCEPT_DISTINCT || upper.m_type == SC_EXCEPT_ALL ||
          upper.m_type == SC_INTERSECT_DISTINCT ||
          upper.m_type == SC_INTERSECT_ALL)
        ql = upper;
      ql.m_elts.push_back(expr);
      pc->m_stack.push_back(ql);
    } break;
    case QT_UNION:
    case QT_EXCEPT:
    case QT_INTERSECT: {
      auto ex = down_cast<Query_term_set_op *>(expr);
      const bool already_decorated =
          ex->query_block()->order_list.elements > 0 ||
          ex->query_block()->select_limit != nullptr;
      if (already_decorated) {                 // 自身已有 ORDER/LIMIT ⇒ 再包一层 unary
        ex = new (pc->mem_root) Query_term_unary(pc->mem_root, ex);
        if (ex == nullptr) return true;
        Query_expression *qe = pc->select->master_query_expression();
        if (ex->set_block(qe->create_post_processing_block(ex))) return true;
      }
      if (m_order != nullptr)
        ex->query_block()->order_list = m_order->order_list->value;
      if (m_limit != nullptr) {
        ex->query_block()->select_limit = m_limit->limit_options.limit;
        ex->query_block()->offset_limit = m_limit->limit_options.opt_offset;
      }

      Query_block *orig_query_block = pc->select;
      LEX *lex = pc->thd->lex;
      pc->select = ex->query_block();          // ★ 切到 setop 的 post-processing block
      lex->push_context(&pc->select->context);
      assert(pc->select->parsing_place == CTX_NONE);
      if (contextualize_safe(pc, m_order, m_limit)) return true;
      lex->pop_context();
      pc->select = orig_query_block;           // ★ 恢复

      if (pc->m_stack.back().m_type == SC_QUERY_EXPRESSION) {
        ex = new (pc->mem_root) Query_term_unary(pc->mem_root, ex);
        if (ex == nullptr) return true;
      }
      pc->m_stack.back().m_elts.push_back(ex); // ★ setop 直接并入上一层
    } break;
  }
  return false;
}
```

**`contextualize_order_and_limit()`（`:1255`）——ORDER BY / LIMIT 归属的判定点**：

```cpp
bool PT_query_expression::contextualize_order_and_limit(Parse_context *pc) {
  /*
    Quick reject test. We don't need to do anything if there are no limit
    or order by clauses.
  */
  if (m_order == nullptr && m_limit == nullptr) return false;

  if (m_body->can_absorb_order_and_limit(m_order != nullptr,
                                         m_limit != nullptr)) {
    if (contextualize_safe(pc, m_order, m_limit)) return true;  // ★ 能吸收 → 就地挂当前 QB
  } else {
    LEX *lex = pc->thd->lex;
    Query_block *orig_query_block = pc->select;
    pc->select = nullptr;                      // ★ 不能吸收 → 置空（名字解析"无表可见"）
    lex->push_context(&pc->select->context);
    assert(pc->select->parsing_place == CTX_NONE);

    bool res = contextualize_safe(pc, m_order, m_limit);

    lex->pop_context();
    pc->select = orig_query_block;
    if (res) return true;
  }
  return false;
}
```

> ★ `can_absorb_order_and_limit()` 是 `PT_query_expression_body` 的纯虚方法，那段 40 行注释（`parse_tree_nodes.h:725-776`）给了 6 个例：**能被吸收**的是内层没写 ORDER BY/LIMIT 的括号；**不能**的是内层已有 ORDER BY 或 LIMIT。

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

⚠️ **勘误**：它**不创建** `Query_block`（QB 由 `LEX::new_query()` / `new_set_operation_query()` 预建），也**不直接调** `add_table_to_list()`（那是 `PT_table_factor_table_ident` 的事）。它全程只写 `pc->select` 的**槽位**。

**完整源码**（`sql/parse_tree_nodes.cc:1170-1231`）：

```cpp
bool PT_query_specification::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  pc->m_stack.push_back(QueryLevel(pc->mem_root, SC_QUERY_SPECIFICATION));
  pc->select->parsing_place = CTX_SELECT_LIST;      // ★ 槽位 0：解析位置

  if (options.query_spec_options & SELECT_HIGH_PRIORITY) {
    Yacc_state *yyps = &pc->thd->m_parser_state->m_yacc;
    yyps->m_lock_type = TL_READ_HIGH_PRIORITY;
    yyps->m_mdl_type = MDL_SHARED_READ;
  }
  if (options.save_to(pc)) return true;             // ★ 槽位 1：select_options

  if (item_list->contextualize(pc)) return true;    // ★ 槽位 2：select list

  // Ensure we're resetting parsing place of the right select
  assert(pc->select->parsing_place == CTX_SELECT_LIST);
  pc->select->parsing_place = CTX_NONE;

  if (contextualize_safe(pc, opt_into1)) return true;   // INTO 前置形式

  if (!from_clause.empty()) {
    if (contextualize_array(pc, &from_clause)) return true;   // ★ 槽位 3：FROM
    pc->select->context.table_list =                          // ★ 重设名字解析起点
        pc->select->context.first_name_resolution_table =
            pc->select->get_table_list();
  }

  if (itemize_safe(pc, &opt_where_clause) ||        // ★ 槽位 5：WHERE
      contextualize_safe(pc, opt_group_clause) ||   // ★ 槽位 6：GROUP BY
      itemize_safe(pc, &opt_having_clause))         // ★ 槽位 7：HAVING
    return true;

  pc->select->set_where_cond(opt_where_clause);
  pc->select->set_having_cond(opt_having_clause);

  /*
    Window clause is resolved under CTX_SELECT_LIST and not
    under CTX_WINDOW. Reasons being:
    1. Window functions are part of select list and the
    resolution of window definition happens along with
    window functions.
    ...
  */
  pc->select->parsing_place = CTX_SELECT_LIST;      // ★ 槽位 8：WINDOW
  if (contextualize_safe(pc, opt_window_clause)) return true;
  pc->select->parsing_place = CTX_NONE;

  QueryLevel ql = pc->m_stack.back();               // ★ 出栈
  pc->m_stack.pop_back();
  pc->m_stack.back().m_elts.push_back(pc->select);
  // ★★ 把自己（这个 QB）交给父层的 m_elts —— 这就是"子节点 push 到父节点"

  return (opt_hints != nullptr ? opt_hints->contextualize(pc) : false);
}
```

> ★★ 最后那行 `pc->m_stack.back().m_elts.push_back(pc->select)` 是理解整套机制的**枢纽**：`QueryLevel::m_elts` 是临时容器，每个子节点把自己塞进父层，父层收集完再决定怎么组合（见 §5.2）。
>
> ★ WINDOW 子句刻意复用 `CTX_SELECT_LIST` 而非 `CTX_WINDOW`，源码列了 5 条理由（窗口函数是 select list 的一部分、命名窗口要跟引用它的函数一起解析等）。

**集合操作：三兄弟都只有 2 行**（`:1699` / `:1704` / `:1709`）：

```cpp
bool PT_union::contextualize(Parse_context *pc) {
  return contextualize_setop(pc, QT_UNION,
                             m_is_distinct ? SC_UNION_DISTINCT : SC_UNION_ALL);
}
bool PT_except::contextualize(Parse_context *pc) {
  return contextualize_setop(pc, QT_EXCEPT,
                             m_is_distinct ? SC_EXCEPT_DISTINCT : SC_EXCEPT_ALL);
}
bool PT_intersect::contextualize(Parse_context *pc) {
  return contextualize_setop(
      pc, QT_INTERSECT, m_is_distinct ? SC_INTERSECT_DISTINCT : SC_INTERSECT_ALL);
}
```

```cpp
// :1654
bool PT_set_operation::contextualize_setop(Parse_context *pc,
                                           Query_term_type setop_type,
                                           Surrounding_context context) {
  pc->m_stack.push_back(QueryLevel(pc->mem_root, context));
  if (super::contextualize(pc)) return true;

  if (m_lhs->contextualize(pc)) return true;        // ★ 左操作数

  pc->select = pc->thd->lex->new_set_operation_query(pc->select);
  // ★★ 为右侧新建 Query_block 并把 pc->select 切过去

  if (pc->select == nullptr || m_rhs->contextualize(pc)) return true;

  pc->thd->lex->pop_context();

  QueryLevel ql = pc->m_stack.back();
  pc->m_stack.pop_back();

  Query_term_set_op *setop = nullptr;
  switch (setop_type) {
    case QT_UNION:   setop = new (pc->mem_root) Query_term_union(pc->mem_root); break;
    case QT_EXCEPT:  setop = new (pc->mem_root) Query_term_except(pc->mem_root); break;
    case QT_INTERSECT: setop = new (pc->mem_root) Query_term_intersect(pc->mem_root); break;
    default: assert(false);
  }
  if (setop == nullptr) return true;

  merge_descendants(pc, setop, ql);                 // ★★ N 元扁平化折叠，见 §5.2
  setop->label_children();

  Query_expression *qe = pc->select->master_query_expression();
  if (setop->set_block(qe->create_post_processing_block(setop))) return true;
  // ★ 取代旧版 fake_query_block

  pc->m_stack.back().m_elts.push_back(setop);
  return false;
}
```

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

⚠️ **勘误**：`PT_item_list::contextualize()` **不是** `itemize_safe`。前者在 `parse_tree_helpers.h:117`（内联），后者是 `parse_tree_nodes.cc:132` 匿名 namespace 里的**单个 `Item*`** 辅助函数。

```cpp
// sql/parse_tree_helpers.h:109
class PT_item_list : public Parse_tree_node {
 public:
  mem_root_deque<Item *> value;

  bool contextualize(Parse_context *pc) override {
    if (super::contextualize(pc)) return true;
    for (Item *&item : value) {
      if (item->itemize(pc, &item)) return true;
      //  ★★ 注意是 Item *&（引用）—— 占位符就地被新 Item 替换
    }
    return false;
  }
  ...
};

// parse_tree_nodes.cc:104（匿名 namespace）
template <typename Context, typename Node>
bool contextualize_safe(Context *pc, Node node) {
  if (node == nullptr) return false;      // ★ nullptr 视为成功
  return node->contextualize(pc);
}

template <typename Context>
bool contextualize_safe(Context *pc, mem_root_deque<Item *> *list) {
  if (list == nullptr) return false;
  for (Item *&item : *list) {
    if (item->itemize(pc, &item)) return true;
  }
  return false;
}

template <typename Context, typename Node, typename... Nodes>
bool contextualize_safe(Context *pc, Node node, Nodes... nodes) {
  return contextualize_safe(pc, node) || contextualize_safe(pc, nodes...);
  //  ★ 变参短路版：contextualize_safe(pc, m_order, m_limit)
}

bool itemize_safe(Parse_context *pc, Item **item) {
  if (*item == nullptr) return false;
  return (*item)->itemize(pc, item);
}
```

```cpp
// sql/parse_tree_helpers.h:170 —— 模板，只能放头文件
template <typename Context, typename Array>
bool contextualize_array(Context *pc, Array *array) {
  for (auto it : *array) {
    if (pc->thd->lex->will_contextualize && it->contextualize(pc)) return true;
    //  ★★ will_contextualize 为 false 时整棵子树被跳过（只做语法检查的场景）
  }
  return false;
}
```

**`PT_select_item_list::contextualize()`（`:3613`）——只有 3 行**：

```cpp
bool PT_select_item_list::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;   // 先逐 Item::itemize
  pc->select->fields = value;                  // ★ 落地到 Query_block::fields
  return false;
}
```

**`Order` 链复用 `PT_order_expr`**（`parse_tree_nodes.h:214`）：

```cpp
class PT_order_list : public Parse_tree_node {
 public:
  SQL_I_List<ORDER> value;

  bool contextualize(Parse_context *pc) override {
    if (super::contextualize(pc)) return true;
    for (ORDER *o = value.first; o != nullptr; o = o->next) {
      if (static_cast<PT_order_expr *>(o)->contextualize(pc)) return true;
      //  ★ ORDER 结构被向下转型成 PT_order_expr，复用 SQL_I_List 链表
    }
    return false;
  }
};

// parse_tree_nodes.cc:305
bool PT_order_expr::contextualize(Parse_context *pc) {
  return super::contextualize(pc) || item_initial->itemize(pc, &item_initial);
  //  ★ 单个排序表达式 → Item::itemize（PT → Item 的桥梁）
}
```

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

⚠️ **勘误**："作用域切换 = 保存/恢复 `pc->select`" **只对派生表成立**。**`PT_subquery::contextualize()` 完全不动 `pc->select`**——它另起一个独立的 `Parse_context`。这是两种截然不同的范式，务必区分。

**范式 A：`PT_subquery`（`:4109`）——另起 `Parse_context`**

```cpp
bool PT_subquery::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;

  LEX *lex = pc->thd->lex;
  if (!lex->expr_allows_subquery || lex->sql_command == SQLCOM_PURGE) {
    error(pc, pos);            // 本位置不允许子查询（某些 DDL 子句）
    return true;
  }

  // Create a Query_expression and Query_block for the subquery's query expression.
  Query_block *child = lex->new_query(pc->select);   // ★ 新建内层 QB（连带建好它的 QE）
  if (child == nullptr) return true;

  Parse_context inner_pc(pc->thd, child);            // ★★ 新建一个独立 Parse_context
  inner_pc.m_stack.push_back(QueryLevel(pc->mem_root, SC_SUBQUERY));
  //  ★ 内层栈底是 SC_SUBQUERY —— 隔离外层的 UNION/ORDER BY 等上下文

  if (m_is_derived_table) child->linkage = DERIVED_TABLE_TYPE;

  if (qe->contextualize(&inner_pc)) return true;     // ★ 用 inner_pc 递归整棵子查询 PT

  if (qe->has_into_clause()) { my_error(ER_MISPLACED_INTO, MYF(0)); return true; }

  query_block = inner_pc.select->master_query_expression()->first_query_block();
  if (inner_pc.finalize_query_expression()) return true;   // ★ 子查询的 Query_term 树在此定型
  inner_pc.m_stack.pop_back();
  assert(inner_pc.m_stack.size() == 0);

  lex->pop_context();

  pc->select->n_child_sum_items += child->n_sum_items;     // ★ 统计量累加回外层
  for (Query_block *temp = child; temp != nullptr; temp = temp->next_query_block()) {
    pc->select->select_n_where_fields += temp->select_n_where_fields;
    pc->select->select_n_having_items += temp->select_n_having_items;
  }
  return false;
  // ★★ 注意：pc->select 从头到尾没被改过，所以这里没有任何"恢复"代码
}
```

**范式 B：`PT_derived_table`（`:1331`）——保存 / 恢复 `pc->select`**

```cpp
bool PT_derived_table::contextualize(Parse_context *pc) {
  Query_block *outer_query_block = pc->select;   // ★★ 保存
  outer_query_block->parsing_place = CTX_DERIVED;
  assert(outer_query_block->linkage != GLOBAL_OPTIONS_TYPE);

  /*
    Determine the immediate outer context for the derived table:
    - if lateral: context of query which owns the FROM i.e. outer_query_block
    - if not lateral: context of query outer to query which owns the FROM.
  */
  if (!m_lateral) {
    pc->thd->lex->push_context(outer_query_block->context.outer_context);
    //  ★ 非 LATERAL：名字解析上下文压"外外层" ⇒ 派生表看不见同 FROM 的兄弟表
  }

  if (m_subquery->contextualize(pc)) return true;   // ★ 进入范式 A

  if (!m_lateral) pc->thd->lex->pop_context();
  outer_query_block->parsing_place = CTX_NONE;

  Query_expression *unit = pc->select->first_inner_query_expression();
  pc->select = outer_query_block;                   // ★★ 恢复
  Table_ident *ti = new (pc->mem_root) Table_ident(unit);   // ★ QE 包装成"表"的标识
  if (ti == nullptr) return true;

  m_table_ref = pc->select->add_table_to_list(pc->thd, ti, m_table_alias, 0,
                                              TL_READ, MDL_SHARED_READ);
  //  ★ 派生表也是通过 add_table_to_list 挂成 Table_ref
  if (m_table_ref == nullptr) return true;
  if (column_names.size()) m_table_ref->set_derived_column_names(&column_names);
  if (m_lateral) {
    m_table_ref->derived_query_expression()->m_lateral_deps = OUTER_REF_TABLE_BIT;
  }
  if (pc->select->add_joined_table(m_table_ref)) return true;
  return false;
}
```

**范式 C：临时切换三段式**（`PT_query_expression:4027`、`contextualize_order_and_limit:1267`）

```cpp
  Query_block *orig_query_block = pc->select;      // ① 保存
  LEX *lex = pc->thd->lex;
  pc->select = ex->query_block();                  // ② 切换
  lex->push_context(&pc->select->context);         //    同步压名字解析上下文
  assert(pc->select->parsing_place == CTX_NONE);
  if (contextualize_safe(pc, m_order, m_limit)) return true;
  lex->pop_context();                              // ③ 恢复
  pc->select = orig_query_block;
```

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

`query_term.h:33` 的文件头注释**用 SQL 标准文法直接回答了这个问题**（节选）：

```cpp
/**
  This class hierarchy is used to represent SQL structures between \<query
  expression\> and \<query specification\>. ... Originally, MySQL supported only
  one set operation, UNION. This was implicitly represented by having one
  Query_expression object own several Query_block objects via next pointers.
  This made things simple, but limited us to left-deep nesting of unions, and
  also from representing INTERSECTION and EXCEPT, as well as nesting several
  layers of \<query primary\>s containing \<query expression body\>s, e.g.

     (((SELECT a,b FROM t) ORDER BY a LIMIT 5) ORDER BY -b LIMIT 3) ORDER BY a;

  could not be supported. With the present class hierarchy we enable the full
  set of set operations and arbitrary nesting ...

  <query expression> ::=
      [ <with clause> ] <query expression body>
      [ <order by clause> ] [ <limit/offset> ]

  <query expression body> ::=
     <query term>
   | <query expression body> UNION [ ALL | DISTINCT ] <query term>
   | <query expression body> EXCEPT [ ALL | DISTINCT ] <query term>

  <query term> ::=
     <query primary>
   | <query expression body> INTERSECT [ ALL | DISTINCT ] <query primary>

  <query primary> ::= <simple table>
                    | <left paren> <query expression body>
                      [ <order by clause> ] [ <limit/offset> ] <right paren>

  <simple table> ::= <query specification>
                   | <table value constructor>
                   | <explicit table>

  Note that INTERSECT binds tighter than UNION and EXCEPT.
*/
```

> ★ 一句话：**`Query_expression` = \<query expression\>，`Query_block` = \<query specification\>，而 `Query_term` 树就是夹在两者之间的\<query expression body\> 的显式表示**（8.0.31 之前这块是隐式的，只靠 `next` 指针串 `Query_block`）。

**三段关键结构性说明**（`query_term.h:109`）：

```cpp
/**
  Query term tree structure. There are five node types, cf. Query_term_type.
  Leaf nodes are Query_block objects. We have three kinds of n-ary set operation
  nodes corresponding to INTERSECT, UNION and EXCEPT. Finally, we have a "unary"
  node which essentially adds a ORDER BY/LIMIT over another node.

  Query blocks serve a dual purpose: they represent the query specification and
  table constructors of the query. As such they are the leaf nodes of the query
  tree.  But they also serve as a way to realize ORDER BY and LIMIT for non-leaf
  nodes, accessed via the function Query_term::query_block().  Therefore, every
  non-leaf node in the tree has a companion Query_block to hold ORDER BY and
  LIMIT information.  ...
*/
```

> ★★ **"Query_block 双重身份"** 是理解这张图的钥匙：它既是**叶子**（查询说明），又被非叶子节点**借用**来承载 ORDER BY / LIMIT。这个"被借用的 QB"就是 `Query_term_set_op::m_block`，由 `create_post_processing_block()` 造出——它**取代了旧版的 `fake_query_block`（8.0.31 后已删除）**，且**不在** `first_query_block()` / `next_query_block()` 链上。

```cpp
  // 注：所有叶子 query block 仍通过 next 指针挂在 Query_expression 下（slave 链），
  //     嵌套则由左侧箭头（m_children）表达。
  //     每个节点还有一个指回父节点 m_parent 的反向指针（图中未画出）。
  //
  //     简单情形（单个 query specification）没有上层结构：
  //     Query_expression 的 m_query_term 就只是一个 Query_block。
  //
  //     树可用 debug trace 关键字 "ast" 打印：
  //     SET SESSION debug = 'd,ast:O,/tmp/mysqld.trace';
*/
```

**配套：`create_post_processing_block()`（`sql_lex.cc:743`）**：

```cpp
Query_block *Query_expression::create_post_processing_block(
    Query_term_set_op *term) {
  Query_block *first_qb = first_query_block();
  Query_block *const qb = first_qb->parent_lex->new_empty_query_block();
  if (qb == nullptr) return nullptr;
  qb->include_standalone(this);          // ★ 挂 master 但【不】进 slave/next 链
  qb->select_number = INT_MAX;
  qb->linkage = GLOBAL_OPTIONS_TYPE;
  qb->select_limit = nullptr;
  qb->set_context(first_qb->context.outer_context);
  /* allow item list resolving in fake select for ORDER BY */
  qb->context.resolve_in_select_list = true;
  qb->no_table_names_allowed =
      !term->is_unary() || first_qb->is_ordered() || first_qb->has_limit();
  first_qb->parent_lex->pop_context();
  return qb;
}
```

> ⚠️ `Query_block::include_standalone()`（`sql_lex.cc:4403`）的注释**仍是陈旧注释**——它还写着"This function is exclusive for Query_expression::add_fake_query_block()"，但 `add_fake_query_block()` 在 8.0.31 后已不存在，现在唯一调用方就是 `create_post_processing_block()`。

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

⚠️ **勘误（本节错得最多）**：

| 常见说法 | 8.0.39 真相 |
|---|---|
| `Query_term::Type`（嵌套 enum class） | **不存在**。是**文件作用域的普通 enum** `Query_term_type`（`query_term.h:89`） |
| `QT_NARY` / `QT_BINARY` | **不存在**。枚举只有 5 个值 |
| `class Query_term_nary` | **不存在**。N 元集合操作的共同基类叫 **`Query_term_set_op`**（注释明写 "including unary"） |
| `m_type` 成员 | **不存在**。类型由纯虚 `virtual Query_term_type term_type() const = 0` 提供 |
| `m_result_columns` | **不存在** |
| `Query_term::set_parent()` | **不存在**。只有 `parent()` getter；`m_parent` 由 `pushdown_limit_order_by()` 直接写 |
| `Query_term::print()` | **不存在**。是纯虚 `debugPrint(int, std::ostringstream&)` |

**节点类型枚举**（`query_term.h:89`）：

```cpp
/// Node type of the query term tree nodes
enum Query_term_type {
  /// Represents Query specification, table value constructor and
  /// explicit table
  QT_QUERY_BLOCK,
  /// Represents a query primary with parentesized query expression body with
  /// order by clause and/or limit/offset clause. If none of order by
  /// or limit is present, we collapse this level of parentheses.
  QT_UNARY,
  /// Represents the three set operations. Nodes are N-ary, i.e. a node can hold
  /// two or more operands.
  QT_INTERSECT,
  QT_EXCEPT,
  QT_UNION
};

/// Query term iterator template argument type: how to visit nodes in tree
enum Visit_order { QTC_POST_ORDER, QTC_PRE_ORDER };
/// Query term iterator template argument type: whether to visit leaf nodes
enum Visit_leaves { VL_VISIT_LEAVES, VL_SKIP_LEAVES };
```

**`class Query_term` 关键成员**（`query_term.h:211`，省略 print 系列）：

```cpp
class Query_term {
 public:
  /// Called after contextualization to simplify query. It also sets up the
  /// parent pointers.
  Query_term *pushdown_limit_order_by(Query_term_set_op *parent = nullptr);

  /// Return true if structure is too deep, i.e. more than MAX_SELECT_NESTING.
  bool validate_structure(const Query_term *parent, int depth = 0) const;

  std::pair<bool, bool> redundant_order_by(Query_block *block, int level);

  virtual Query_term_type term_type() const = 0;   // ★ 类型靠虚函数，没有 m_type
  virtual const char *operator_string() const = 0;
  virtual ~Query_term() = default;
  virtual size_t child_count() const { return 0; }
  virtual void label_children() = 0;

 protected:
  /// Back pointer to the node whose child we are, or nullptr (root term).
  Query_term_set_op *m_parent{nullptr};
  /// If parent is non-null, this holds the index of the current sibling.
  uint m_sibling_idx{0};

 public:
  Query_term_set_op *parent() const { return m_parent; }   // ★ 只有 getter
  void set_sibling_idx(uint idx) { m_sibling_idx = idx; }
  uint sibling_idx() { return m_sibling_idx; }

  virtual void cleanup(bool) { assert(false); }   // 必须被覆盖
  virtual void destroy_tree() = 0;
  virtual bool open_result_tables(THD *thd, int level) { assert(false); return false; }

  virtual void debugPrint(int level, std::ostringstream &buf) const = 0;
  static void indent(int level, std::ostringstream &buf);
  void printPointers(std::ostringstream &buf) const;

  /// The query_block which holds the ORDER BY and LIMIT information for this
  /// set operation. ... for the root being a simple query block, this returns
  /// self.
  virtual Query_block *query_block() const = 0;

 protected:
  Query_result *m_setop_query_result{nullptr};
  bool m_owning_operand{false};
  Table_ref *m_result_table{nullptr};
  mem_root_deque<Item *> *m_fields{nullptr};
};
```

**`Query_term_set_op`（真实"N 元"基类，`query_term.h:403`）**：

```cpp
/// Common base class for n-ary set operations, including unary.
class Query_term_set_op : public Query_term {
  /// Replaces the old "fake" query block for post processing result set with
  /// ORDER BY, LIMIT.
  Query_block *m_block{nullptr};

 protected:
  Query_term_set_op(MEM_ROOT *mem_root) : m_children(mem_root) {}

 public:
  /// Cardinality is one for unary, two or more for UNION, EXCEPT, INTERSECT
  mem_root_deque<Query_term *> m_children;      // ★★ 不是数组，是 mem_root_deque，且 public
  /// Index of last query expression which has <set-op> DISTINCT on its left.
  int64_t m_last_distinct;
  /// Presently only needed by EXCEPT: the index of the first DISTINCT operand.
  int64_t m_first_distinct;
  /// true if the result of this set operation is materialized.
  bool m_is_materialized{true};

  Query_block *query_block() const override { return m_block; }
  bool set_block(Query_block *b) { assert(!m_block); if (b == nullptr) return true; m_block = b; return false; }

  void label_children() override {
    uint idx = 0;
    for (auto child : m_children) { child->set_sibling_idx(idx++); child->label_children(); }
  }
  size_t child_count() const override { return m_children.size(); }
  void destroy_tree() override {
    m_parent = nullptr;
    for (Query_term *child : m_children) child->destroy_tree();
    m_children.clear();
  }
  bool has_mixed_distinct_operators();
  bool is_unary() const { return term_type() == QT_UNARY; }
};
```

**四个子类各 5 行**（`query_term.h:497-560`）：

```cpp
/// Node type for n-ary UNION
class Query_term_union : public Query_term_set_op {
 public:
  Query_term_union(MEM_ROOT *mem_root) : Query_term_set_op(mem_root) {}
  Query_term_type term_type() const override { return QT_UNION; }
  const char *operator_string() const override { return "union"; }
  void debugPrint(int level, std::ostringstream &buf) const override;
};
// Query_term_intersect → QT_INTERSECT / "intersect"
// Query_term_except   → QT_EXCEPT   / "except"

/// A <query primary> which is a parenthesized query expression body with
/// order by clause and/or limit/offset clause ...
class Query_term_unary : public Query_term_set_op {
 public:
  Query_term_unary(MEM_ROOT *mem_root, Query_term *t)
      : Query_term_set_op(mem_root) {
    m_last_distinct = 0;          // ★ unary 的 last_distinct 恒为 0
    m_children.push_back(t);
    t->set_sibling_idx(0);
  }
  Query_term_type term_type() const override { return QT_UNARY; }
  const char *operator_string() const override { return "result"; }
  void debugPrint(int level, std::ostringstream &buf) const override;
};
```

**`Query_block` 自己就是 `Query_term` 的叶子子类**（`sql_lex.h:1311`）：

```cpp
class Query_block : public Query_term {
 public:
  Query_term_type term_type() const override { return QT_QUERY_BLOCK; }
  const char *operator_string() const override { return "query_block"; }
  Query_block *query_block() const override {
    return const_cast<Query_block *>(this);   // ★ 叶子：query_block() 返回 self
  }
  void label_children() override {}
  void destroy_tree() override { m_parent = nullptr; }
};
```

**`Query_expression` 侧只有一个指针**（`sql_lex.h:782`）：

```cpp
  Query_term *m_query_term{nullptr};   // ★ 整棵树的根，就这一个成员

  Query_term *query_term() const { return m_query_term; }
  void set_query_term(Query_term *qt) { m_query_term = qt; }   // ★ 一行 setter，折叠不在这里
  Query_term_set_op *set_operation() const {
    return is_simple() ? nullptr : down_cast<Query_term_set_op *>(m_query_term);
  }
  /// @return true for a query expression without UNION/INTERSECT/EXCEPT ...
  bool is_simple() const { return m_query_term->term_type() == QT_QUERY_BLOCK; }
  /// 取代旧的 fake_query_block
  inline Query_block *global_parameters() const {
    return query_term()->query_block();
  }
```

**`pushdown_limit_order_by()`——括号折叠 + 建父指针的真身**（`query_term.cc:74`，节选 QT_UNARY 分支）：

```cpp
Query_term *Query_term::pushdown_limit_order_by(Query_term_set_op *parent) {
  m_parent = parent;                          // ★ 父指针唯一写入点
  switch (term_type()) {
    case QT_UNION: case QT_INTERSECT: case QT_EXCEPT: {
      auto setop = down_cast<Query_term_set_op *>(this);
      for (Query_term *&child : setop->m_children) {
        const uint sibling_idx = child->sibling_idx();
        child = child->pushdown_limit_order_by(down_cast<Query_term_set_op *>(this));
        child->set_sibling_idx(sibling_idx);
      }
    } break;
    case QT_UNARY: {
      auto *unary = down_cast<Query_term_unary *>(this);
      Query_block *const this_block = unary->query_block();
      Query_block *const child_block = unary->m_children[0]->query_block();
      if (this_block->order_list.elements == 0 &&
          child_block->absorb_limit_of(this_block)) {
        // Eliminate this level by pushing down LIMIT/OFFSET, if any.
        //   (SELECT a,c FROM t1 ORDER BY a LIMIT 4) LIMIT 2
        // → SELECT a,c FROM t1 ORDER BY a LIMIT 2
        return unary->m_children[0]->pushdown_limit_order_by(parent);
      } else {
        if (child_block->order_list.elements == 0 &&
            child_block->select_limit == nullptr) {
          // 内层没 ORDER BY / LIMIT ⇒ 把外层的都推下去，消掉这一层
          child_block->order_list = this_block->order_list;
          child_block->absorb_limit_of(this_block);
          child_block->m_windows.prepend(&this_block->m_windows);
          ...
          return unary->m_children[0]->pushdown_limit_order_by(parent);
        } else {
          unary->m_children[0] = unary->m_children[0]->pushdown_limit_order_by(
              down_cast<Query_term_set_op *>(this));
        }
      }
    } break;
    case QT_QUERY_BLOCK: break;
  }
  return this;
}
```

**`validate_structure()`（`query_term.cc:162`）**：

```cpp
bool Query_term::validate_structure(const Query_term *parent, int level) const {
  assert(m_parent == parent);
  if (level > MAX_SELECT_NESTING) {
    my_error(ER_TOO_HIGH_LEVEL_OF_NESTING_FOR_SELECT, MYF(0));
    return true;
  }
  if (term_type() == QT_QUERY_BLOCK) return false;
  for (Query_term *child :
       down_cast<const Query_term_set_op *>(this)->m_children) {
    if (child->validate_structure(this, level + 1)) return true;
  }
  query_block()->renumber(query_block()->parent_lex);
  return false;
}
```

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

## 八、PT_* 大家族全景

> ★ **完整剖析已独立成篇：[`11_parse_tree.md`](11_parse_tree.md)**——基类（`Parse_tree_root` / `Parse_tree_node_tmpl<Context>`）逐行剖析、两套正交递归、`PT_*` 17 个范畴的**代表类完整 `contextualize()` 代码**、与 yacc 的对应、`PT_query_specification` 深剖。
>
> 本节只保留**流程视角**：这个家族有多大、命名怎么区分、contextualize 递归的"输入"长什么样、两套递归各有多大规模。

### 8.1 规模与范畴（一句话）

`PT_*` ≈**403** 个类（非 PTI 的 `PT_*` = 367，`PTI_*` = 36），分 **17 个范畴**——完整分类大表、每类的代表实现与真实代码见 [`11_parse_tree.md`](11_parse_tree.md) 3.1 / 3.2。

### 8.3 命名规律与"身份"体系

**`Parse_tree_item`：PTI_* 的共同基类（`parse_tree_helpers.h:58`）**

```cpp
/**
  Base class for parse-time Item objects

  Parse-time Item objects are placeholders for real Item objects: in some
  cases it is not easy or even possible to decide what exact Item class object
  we need to allocate in the parser. Parse-time Item objects are intended
  to defer real Item object allocation to the contextualization phase (see
  the Item::itemize() function).

  This wrapper class overrides abstract virtual functions of the parent
  class with dummy wrappers to make C++ compiler happy.
*/
class Parse_tree_item : public Item {
 public:
  explicit Parse_tree_item(const POS &pos) : Item(pos) {}
  //  ★ 传 POS ⇒ is_parser_item = true ⇒ 一定会走 itemize（不会被 skip_itemize 短路）

  enum Type type() const override { return INVALID_ITEM; }  // ★ 占位符的 type
  double val_real() override { assert(0); return 0; }
  longlong val_int() override { assert(0); return 0; }
  String *val_str(String *) override { assert(0); return nullptr; }
  my_decimal *val_decimal(my_decimal *) override { assert(0); return nullptr; }
  bool get_date(MYSQL_TIME *, uint) override { assert(0); return false; }
  bool get_time(MYSQL_TIME *) override { assert(0); return false; }
};
```

> ⚠️ **不是所有 `PTI_*` 都继承 `Parse_tree_item`**：
> - `PTI_text_literal : public Item_string`（`parse_tree_items.h:239`）
> - `PTI_count_sym : public Item_sum_count`（`:412`）
>
> 这两个在语法期语义就确定了，**不需要占位**，它们的 `itemize()` 只是"就地完成初始化"（`*res` 仍是 `this`）。

**变身的三条路线：`PTI_simple_ident_ident::itemize()`（`parse_tree_items.cc:353`）**

```cpp
bool PTI_simple_ident_ident::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  //  ★ 先跑 Item::itemize：登记到 thd 的 item 列表 + select_n_having_items++

  THD *thd = pc->thd;
  LEX *lex = thd->lex;
  sp_pcontext *pctx = lex->get_sp_current_parsing_ctx();
  sp_variable *spv;

  if (pctx && (spv = pctx->find_variable(ident.str, ident.length, false))) {
    /* We're compiling a stored procedure and found a variable */
    if (!lex->parsing_options.allows_variable) {
      my_error(ER_VIEW_SELECT_VARIABLE, MYF(0));
      return true;
    }
    *res = create_item_for_sp_var(thd, ident, spv, ...);   // ★ 路线①：SP 局部变量 → Item_splocal
    lex->safe_to_cache_query = false;
  } else {
    if ((pc->select->parsing_place != CTX_HAVING) ||
        (pc->select->get_in_sum_expr() > 0)) {
      *res = new (pc->mem_root) Item_field(POS(), NullS, NullS, ident.str);
      //  ★ 路线②：普通列名 → Item_field（db/table 留空，等 fix_fields 解析）
    } else {
      *res = new (pc->mem_root) Item_ref(POS(), NullS, NullS, ident.str);
      //  ★ 路线③：HAVING 中的裸列名 → Item_ref（可能引用 select list 别名）
    }
    if (*res == nullptr || (*res)->itemize(pc, res)) return true;
    //  ★ 新造的 Item 也要再 itemize 一次
  }
  return *res == nullptr;
}
```

**`PTI_comp_op::itemize()`（`parse_tree_items.cc:166`）——"三段式 + 工厂造对象"**：

```cpp
bool PTI_comp_op::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || left->itemize(pc, &left) ||
      right->itemize(pc, &right))     // ★ 自己 → 左 → 右，递归下沉
    return true;

  *res = (*boolfunc2creator)(false)->create(left, right);
  //      ★★ 语法期存下的"函数工厂"在此现场造出 Item_func_eq / _ne / _lt ...
  return *res == nullptr;
}

bool PTI_comp_op_all::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || left->itemize(pc, &left) ||
      subselect->contextualize(pc))   // ★ 右侧是 PT_subquery ⇒ 走 contextualize，不是 itemize
    return true;

  *res = all_any_subquery_creator(left, comp_op, is_all, subselect->value());
  return *res == nullptr;
}
```

```cpp
// sql/parse_tree_items.cc:335 —— 别名处理
bool PTI_expr_with_alias::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || expr->itemize(pc, &expr)) return true;

  if (alias.str) {
    expr->item_name.copy(alias.str, alias.length, system_charset_info, false);
    //  ★ 有 AS alias
  } else if (!expr->item_name.is_set()) {
    expr->item_name.copy(expr_loc.start,
                         (uint)(expr_loc.end - expr_loc.start), pc->thd->charset());
    //  ★ 无别名 ⇒ 用原始 SQL 文本片段做默认列名
  }
  *res = expr;   // ★ "变身"方式：丢弃占位符，把内层 expr 提升为结果
  return false;
}
```

**`Item::itemize()` 本体（`item.cc:631`）——没有对 `Parse_tree_item` 的任何运行时特判**：

```cpp
bool Item::itemize(Parse_context *pc, Item **res) {
  if (skip_itemize(res)) return false;
  //  ★ return !is_parser_item; —— 优化期新建的 Item 直接原样返回，不变身
  if (super::contextualize(pc)) return true;
  //  ★ super 是 Parse_tree_node_tmpl<Parse_context>

  pc->thd->add_item(this);
  if (pc->select) {
    const enum_parsing_context place = pc->select->parsing_place;
    if (place == CTX_SELECT_LIST || place == CTX_HAVING || place == CTX_ORDER_BY) {
      pc->select->select_n_having_items++;
    }
  }
  return false;
}
```

```cpp
// sql/item.h:1143 —— Item 家族屏蔽 contextualize，只暴露 itemize
 private:
  bool contextualize(Parse_context *) override { assert(0); return true; }
 protected:
  bool skip_itemize(Item **res) { *res = this; return !is_parser_item; }
// sql/item.h:3483
  const bool is_parser_item;  ///< true if allocated directly by parser
```

> ★★ **整套"变身"机制没有一处 `dynamic_cast`**：靠 `Item(const POS&)` 构造置 `is_parser_item = true`（`item.cc:196`）+ 虚函数分派完成。这是"编译期约定 + 一个 bool"的干净设计。

| 前缀 | 含义 | 基类 | 是否 Item | 数量 |
|---|---|---|---|---|
| `PT_` | Parse Tree 纯语法节点 | `Parse_tree_node`（= `Parse_tree_node_tmpl<Parse_context>`） | 否 | ≈367 |
| `PTI_` | Parse Tree **Item**，解析期占位符 | `Parse_tree_item : public Item` | **是** | **36** |

★ `PTI_*` 的本质，源码注释（`parse_tree_helpers.h`）说得很直白：

```
  Base class for parse-time Item objects

  Parse-time Item objects are placeholders for real Item objects: in some
  cases it is not easy or even possible to decide what exact Item class object
  we need to allocate in the parser. Parse-time Item objects are intended
  to defer real Item object allocation to the contextualization phase ...
```

⚠️ 而这些占位符的 `val_*()` **全部是 `assert(0)` 桩**——一个 `PTI_*` 若逃过 `itemize()` 进入执行期，debug 下**直接崩**。这是设计上的硬约束（保证占位符必被替换）。

**构词规律**（实测归纳，读源码时很好用）：

- `<语句>_stmt` ⇒ **语句根**（`PT_select_stmt`、`PT_create_table_stmt`、`PT_truncate_table_stmt`）
- `<范畴>_base` ⇒ **抽象基**（`PT_show_base`、`PT_base_index_option`、`PT_role_or_privilege`、`PT_column_attr_base`）
- `PT_create_<x>_option` / `PT_alter_table_<动作>` / `PT_show_<对象>` ⇒ **一个语法选项一个类**
- ★ **成员名 ≈ yacc 规则名**：`opt_hints`、`opt_where_clause`、`opt_group_clause`、`opt_having_clause`、`opt_window_clause`、`opt_key_definition`——看到成员名就能反查语法规则
- **新旧判断**：新增成员用 `m_` 前缀（`m_body`、`m_order`、`m_with_clause`、`m_lateral`），老代码无前缀

**基类接口**（`parse_tree_node_base.h`，模板 `Parse_tree_node_tmpl<Context>`）：

| 成员 | 作用 | 默认实现 |
|---|---|---|
| `operator new(size_t, MEM_ROOT*, nothrow_t&)` | 放置式 new，节点只能分配在 MEM_ROOT 上 | `mem_root->Alloc(size)` |
| `operator delete` | **只做 `TRASH`（debug 毒化），不真正释放** | 空 + TRASH |
| 拷贝构造/赋值 | **故意 undefined（private）**，禁止拷贝 | — |
| `bool contextualized`（仅 debug） | "已 contextualize"标记，构造时 false | — |
| `virtual bool contextualize(Context*)` | 核心；**默认实现即递归入口** | `check_stack_overrun` → `assert(!contextualized)` → 置 true |
| `error()` / `errorf()` | **延迟报错**：parser 期不报，contextualize 期按源码位置报 | `thd->syntax_error_at(pos)` |

三个关键语义：

1. **返回值**：`false = 成功`、`true = 出错`
2. **默认实现即递归入口**：几乎所有 override 都先 `if (super::contextualize(pc)) return true;` 再逐子节点递归——所以全家族统一 `typedef X super;`
3. ★ **上下文是模板参数**：`Parse_tree_node`（`Parse_context`）／`Table_ddl_node`（`Table_ddl_parse_context`，多带 `Alter_info`/`HA_CREATE_INFO`）／`Parse_tree_node_tmpl<Column_parse_context>`（列属性）。**同一套机制、三种上下文**——用类型系统防止误用（DDL 节点能拿 `alter_info`，查询节点拿不到）


### 8.3 ★ contextualize 递归的"输入"长什么样

（这棵树在 [`11_parse_tree.md`](11_parse_tree.md) 第三章被逐步展开：`PT_select_stmt` / `PT_query_expression` / `PT_query_specification` 各自的 `contextualize()` 做了什么。）

### 8.5 ★ 树的形状：一个完整 SELECT 的 PT 树

以 `SELECT a, (SELECT MAX(x) FROM t2) AS m FROM t1 JOIN (SELECT * FROM t3) AS d ON d.id=t1.id WHERE a>10 GROUP BY a HAVING COUNT(*)>1 WINDOW w AS (...) ORDER BY a LIMIT 5` 为例（括号内是**源码真实成员名**）：

```
PT_select_stmt                              ← 根（Parse_tree_root::make_cmd）
│  m_sql_command, m_qe, m_into, m_has_trailing_locking_clauses
└─ m_qe : PT_query_expression
   │  m_with_clause, m_body, m_order, m_limit
   ├─ m_with_clause : PT_with_clause
   │  └─ PT_with_list → PT_common_table_expr → PT_subquery → (递归回 PT_query_expression)
   ├─ m_body : PT_query_specification
   │  ├─ opt_hints        : PT_hint_list → PT_hint / PT_qb_level_hint
   │  ├─ item_list        : PT_select_item_list → PTI_* / Item*   ← ★ 第二条(Item)递归在此分叉
   │  ├─ opt_into1        : PT_into_destination(_outfile/_dumpfile)
   │  ├─ from_clause[]    : PT_table_reference*
   │  │  └─ PT_joined_table_on
   │  │     ├─ m_left_pt_table  : PT_table_factor_table_ident  (t1)
   │  │     ├─ m_right_pt_table : PT_derived_table             ((SELECT * FROM t3) d)
   │  │     │                       └─ m_subquery : PT_subquery → PT_query_expression → ... ★递归
   │  │     └─ (ON 条件) Item*  ← PTI_comp_op 变身后的 Item_func_eq
   │  ├─ opt_where_clause  : Item*   (PTI_comp_op → Item_func_gt)
   │  ├─ opt_group_clause  : PT_group → PT_order_list → PT_order_expr（同时是 ORDER 对象）
   │  ├─ opt_having_clause : Item*
   │  └─ opt_window_clause : PT_window_list → PT_window（同时是 Window 对象）
   │                            ├─ PT_order_list (PARTITION BY / ORDER BY)
   │                            └─ PT_frame → PT_borders → PT_border ×2, PT_exclusion
   ├─ m_order : PT_order → PT_order_list → PT_order_expr
   └─ m_limit : PT_limit_clause → (PTI_limit_option_ident | PTI_int_splocal | Item*)
```

**深度**（只算 PT 树，不含 Item 表达式子树）：

| 路径 | 层数 |
|---|---|
| 普通单表 SELECT | **5–6** |
| 带一个派生表/标量子查询 | **9–10**（`... → PT_derived_table → PT_subquery → PT_query_expression → PT_query_specification → PT_table_factor_table_ident` = 9） |
| 三层嵌套派生表 | ≈**15–17** |
| UNION | 起点 +3 |
| WITH | 起点 +5 |
| WINDOW | 5（`PT_query_specification → PT_window_list → PT_window → PT_frame → PT_borders → PT_border`） |

⇒ **循环体是 `PT_subquery → PT_query_expression → PT_query_specification`（每层 +3~4）**——嵌套有多深，树就有多深。

★ **深度的两道保护**：① 基类默认实现里每次递归都 `check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy)`；② yacc 自身的递归限制。

★ **两条递归的交汇点**是 `PT_item_list::contextualize()`：

```cpp
  bool contextualize(Parse_context *pc) override {
    if (super::contextualize(pc)) return true;
    for (Item *&item : value) {
      if (item->itemize(pc, &item)) return true;   // ← Item 树递归在这里分叉出去
    }
    return false;
  }
```


### 8.4 ★ override 统计：两套正交递归的量化证据

| 方法 | 声明数 | 分布 |
|---|---|---|
| `contextualize()` | **≈162**（扣除基类默认与 Item 隐藏版） | nodes.h **114**、partitions.h 24、column_attrs.h 12、hints.h 8、window.h 2、handler.h 1、helpers.h 1 |
| `itemize()` | **92** | items.h **33**、item_func.h 20、item_strfunc.h 11、item.h 8、item_timefunc.h 5、cmpfunc 3、item_sum 3、其余 |

⇒ **contextualize 的 override 集中在 PT 头文件、itemize 的集中在 Item 头文件**——这就是第三章"两套正交递归"最硬的量化证据。

另：`parse_tree_items.h` 里 36 个 `PTI_*` 有 33 个 override 了 `itemize()`。**没 override 的 3 个是 `PTI_text_literal`、`PTI_where`、`PTI_having`**——其中 `PTI_text_literal` 是抽象中间层（只为共享 `is_7bit`/`literal` 字段），`PTI_where`/`PTI_having` 复用基类 `PTI_context::itemize`。

> ⚠️ **"没 override" ≠ "本身就是真 Item"**（这是易错点）：真正**直接继承真实 Item** 的有 **7 个**（`PTI_function_call_nonkeyword_now`→`Item_func_now_local`、`PTI_text_literal`→`Item_string`、`PTI_literal_underscore_charset_{hex,bin}_num`→`Item_string`、`PTI_variable_aux_set_var`→`Item_func_set_user_var`、`PTI_user_variable`→`Item_func_get_user_var`、`PTI_count_sym`→`Item_sum_count`），再经 `PTI_text_literal` **间接**继承 `Item_string` 的还有 4 个 ⇒ 合计 **11 个 PTI 血统里带着真 Item**。而它们**几乎全都仍 override 了 `itemize()`**——此时 itemize 不是用来变身，而是用来**补完构造 / 做 LEX 副作用**（见 9.6）。

（`contextualize` override ≈163、`itemize` ≈91，其中 33 次集中在 `PTI_*`——统计口径与解读见 [`11_parse_tree.md`](11_parse_tree.md) 6.2。）

### 8.7 生命周期：PT 节点"一次性消费"

**创建**：parser 期全部 `new(YYMEM_ROOT)` ⇒ 落在 `thd->mem_root`。
**使用**：`MYSQLparse` 返回根 → `THD::sql_parser()` → `LEX::make_sql_cmd(root)` → `root->make_cmd(thd)` → 整树 contextualize 一遍，信息搬进 `LEX`/`Query_block`/`Table_ref`/`Query_term`/`Alter_info`。
★ **销毁：没有"销毁 PT 树"的代码**——节点从不 `delete`（基类 `operator delete` 只 TRASH 不释放），随 MEM_ROOT 整体消亡。

**★ PT 节点被丢弃的四种机制**（这是本架构的默认契约，不是例外）：

1. **`PTI_*` 变身**（覆盖全部表达式）：`item->itemize(pc, &item)` 第二个参数是 `Item **`——**就地替换指针**，原 `PTI_*` 成为垃圾
2. **解析期就地 `MAKE_CMD`**：`grant`、`revoke`、SP 内部语句在 yacc 动作里直接 `MAKE_CMD(tmp)`，根节点在 parse 过程中就被吃掉，不再作为 `root` 返回（源码注释：*"If 'root' is NULL on success, then the parser has already called lex->make_sql_cmd() internally."*）
3. **"身份转移"型**：`PT_order_expr : public Parse_tree_node, public ORDER`、`PT_window : public Parse_tree_node, public Window`——contextualize 后以 `ORDER`/`Window` 身份活下去，PT 身份作废。`PT_order_list::contextualize()` 里 `static_cast<PT_order_expr *>(o)->contextualize(pc)` 就是明证
4. **"假节点"型**：`PT_query_primary : public PT_query_expression_body {}` 是空壳（只为给 `PT_query_specification` 与 `PT_table_value_constructor` 一个共同父亲）；`PT_explicit_table` 只改构造参数

⇒ **能长期存活的只有少数"身份转移"型**，其余全部一次性消费。

**★ `THD::item_list` 里只有 Item，PT 节点不进去**：

| 对象 | 是 Item 吗 | 进 `item_list` 吗 | 释放 |
|---|---|---|---|
| 普通 `PT_*`（如 `PT_query_specification`） | 否 | **否** | 随 MEM_ROOT |
| `PTI_*`（占位符 Item） | 是 | **是**（构造即入链） | 随 MEM_ROOT |
| 变身后的真 Item（如 `Item_func_gt`） | 是 | **是**（`itemize()` 里 `add_item`） | 随 MEM_ROOT |

（PT 节点全在 `thd->mem_root`、contextualize 后即弃、基类**没有** `cleanup()`——见 [`11_parse_tree.md`](11_parse_tree.md) 1.7。）

## 九、PTI_* 占位符（已迁入专篇）

> ★ **36 个类的完整剖析已迁入专篇：[`11_parse_tree.md`](11_parse_tree.md) 第十章**——完整清单（类名 / 继承自 / 变身目标 / 变身模式）、重点类的 `itemize()` 完整代码、以及"**为什么需要占位符**"的十类归因表。
>
> 本章只保留**流程视角**的一点：`PTI_*` 在 contextualize 递归中**何时被消费、由谁遍历**。

### 9.1 遍历点：`PT_item_list` 与 `itemize_safe`

`PTI_*` 自己不会主动"跑起来"——它们被三类调用者驱动：

| 调用者 | 代码 | 效果 |
|---|---|---|
| `PT_item_list::contextualize()` | `for (Item *&item : value) item->itemize(pc, &item);` | ★ 引用遍历，**原地改写槽位**，壳被丢弃 |
| `itemize_safe(pc, &field)` | `if (*item == nullptr) return false; return (*item)->itemize(pc, item);` | 单槽位（如 `opt_where_clause`）的空安全版本 |
| 父 `PTI_*` 的 `itemize()` | `left->itemize(pc, &left)` / `expr->itemize(pc, &expr)` | ★ 递归下钻（传的是**成员槽位地址**） |

⇒ 所以"`PTI_*` 全部失效"的时刻是：**所有被 `contextualize()` / `itemize()` 覆盖到的槽位都走完一遍之后**。`PTI_context` 末尾的 `assert(pc->select->parsing_place == m_parsing_place);` 就是这条不变式的运行期守卫。

## 十、`Query_block` / `Query_expression` / `Query_term` 成员全景与生命周期（对象视角）

> **与姊妹篇的分工**：[`11_parse_tree.md`](11_parse_tree.md) 讲 **PT 构造侧**（PT 树如何被填充进 QE/QB、`Query_term` 树如何组装）；`06_resolver_prepare.md` 讲 **prepare 流程**；本章讲**这三个类本身长什么样、活多久**（对象视角）。
>
> 源码坐标：`[Q1] sql/sql_lex.h` · `[Q2] sql/sql_lex.cc` · `[Q3] sql/query_term.h` · `[Q4] sql/query_term.cc` · `[Q5] sql/sql_union.cc` · `[Q6] sql/sql_union.h` · `[Q8] sql/sql_parse.cc` · `[Q10] sql/item.h`

**一句话定位**：`Query_block`（QB）= 一个 query specification，**一组彼此独立的表达式槽位** + 表清单 + 名字解析上下文；`Query_expression`（QE）= 一个 query expression，**一棵 `Query_term` 树** + 结果出口 + 执行期状态；`Query_term`（QT）= 树节点抽象。

### 10.1 ★ `Query_block` 成员全景

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                Query_block  ( : public Query_term )               │
 ├──────────────────────────────────────────────────────────────────┤
 表达式槽位组    fields  m_where_cond  m_having_cond  group_list      │
 (彼此无共同    order_list  select_limit  offset_limit  m_windows     │
  父节点!)                                                           │
 ├──────────────────────────────────────────────────────────────────┤
 表/名字解析组  m_table_list  m_table_nest  m_current_table_nest      │
                context(Name_resolution_context)  leaf_tables         │
                embedding  sj_nests  recursive_reference             │
 ├──────────────────────────────────────────────────────────────────┤
 选项/状态组    m_base_options  m_active_options  parsing_place       │
                in_sum_expr  resolve_place  olap  linkage             │
                select_number  select_n_where_fields                  │
                select_n_having_items  n_sum_items  n_child_sum_items │
                n_scalar_subqueries                                   │
 ├──────────────────────────────────────────────────────────────────┤
 聚合/GROUP BY  m_agg_func_used  hidden_group_field_count             │
                hidden_order_field_count  inner_sum_func_list         │
                join(JOIN*)  ← 优化期才挂                            │
 ├──────────────────────────────────────────────────────────────────┤
 树位置/链表    (继承) m_parent  m_sibling_idx                        │
                master(QE*)  next  slave  link_next                   │
 ├──────────────────────────────────────────────────────────────────┤
 ref 数组       base_ref_items  m_saved_base_items(PS 快照)           │
 └──────────────────────────────────────────────────────────────────┘
```

#### 10.1.1 ★ 表达式槽位组（理解"一个查询不是一棵树"的关键）

```cpp
// [Q1]
  /**
    All expressions needed after join and filtering, ie., select list,
    group by list, having clause, window clause, order by clause,
    including hidden fields.
    Does not include join conditions nor where clause.
    Currently, all hidden items must be before all visible items.
   */
  mem_root_deque<Item *> fields;

  List<Window> m_windows;
  SQL_I_List<Table_ref> m_table_list{};
  SQL_I_List<ORDER> order_list{};      Group_list_ptrs *order_list_ptrs{nullptr};
  SQL_I_List<ORDER> group_list{};      Group_list_ptrs *group_list_ptrs{nullptr};
  Item *select_limit{nullptr};         ///< LIMIT clause, NULL if no limit
  Item *offset_limit{nullptr};         ///< OFFSET clause
  Item *m_where_cond;
  Item *m_having_cond;
```

★ **这 8 个槽位彼此独立，没有任何共同的 `Item` 父节点**。不存在"一个查询 = 一棵表达式树"这种东西。三个直接后果：

1. **遍历一个查询块 = 遍历所有槽位**——`Query_block::walk()` 就是把 processor 依次喂给 `fields` / where / having / group_list / order_list / m_windows / join 条件；
2. 任何"改写查询块"的转换（半连接、`split_sum_funcs`、条件下推）也必须**逐槽位**改写并用 `thd->change_item_tree()` 登记，才能整体回滚；
3. `m_where_cond` 必须走 `where_cond_ref()` 拿 `Item **`（`change_item_tree` 需要**地址**）——这就是为什么要提供 `where_cond_ref()` / `having_cond_ref()` 而不只是 getter。

另注意 `select_limit` / `offset_limit` 是 **`Item *` 而不是 `ha_rows`**：LIMIT 在 prepare 前是表达式（PS 里可以是 `?`），到 `Query_expression::set_limit()` 才求值成计数器（那两个计数器在 **QE** 上，不在 QB 上）。

`order_list_ptrs` / `group_list_ptrs` 是 PS 用的 **next 指针副本**——优化期 `remove_const()` 会剪链，重执行时要靠它重建。

#### 10.1.2 选项：★ 8.0.39 已无裸 `base_options`

```cpp
// [Q1] —— private，对外只有 active_options() 与 make_active_options()
  ulonglong m_base_options{0};   ///< 解析期定下，之后不再改
  ulonglong m_active_options{0}; ///< 每执行重算（含 session option_bits）
```

```cpp
// [Q2]
void Query_block::make_active_options(ulonglong added_options,
                                      ulonglong removed_options) {
  m_active_options =
      (m_base_options | added_options | parent_lex->statement_options() |
       parent_lex->thd->variables.option_bits) & ~removed_options;
}
```

⇒ **base/active 分离正是为 PS 重执行设计的**：session 的 `option_bits` 每次执行可能变，所以每执行重算一次。

#### 10.1.3 三个"分组"谓词的层次

```cpp
  bool is_explicitly_grouped() const { return group_list.elements != 0; }
  bool is_implicitly_grouped() const {
    return m_agg_func_used && group_list.elements == 0;   // 没写 GROUP BY 但用了聚合 → 至多 1 行
  }
  bool is_grouped() const { return group_list.elements > 0 || m_agg_func_used; }
```

`is_explicitly_grouped()` ⊂ `is_grouped()` ⊃ `is_implicitly_grouped()`。`hidden_group_field_count` / `hidden_order_field_count` 记的是"为 GROUP BY / ORDER BY 偷偷塞进 `fields` 的 hidden item 个数"，既决定 `can_skip_distinct()`，也决定 `remove_hidden_items()` 砍多少前缀。

★ `join` 是**优化期才挂上来的产物**：`join != nullptr` ⟺ "已经 optimize 过"——这是 `cleanup()` / `restore_cmd_properties()` 里 `assert(join == nullptr)` 的依据。

#### 10.1.4 `base_ref_items`：为什么需要"预留槽位"

```cpp
// [Q2]
bool Query_block::setup_base_ref_items(THD *thd) {
  uint order_group_num = (order_list.elements + group_list.elements) * 2;
  if (is_distinct()) { /* BIT 列额外计数 */ }
  Query_arena *arena = thd->stmt_arena;
  uint n_elems = n_sum_items + n_child_sum_items + fields.size() +
                 select_n_having_items + select_n_where_fields +
                 order_group_num + n_scalar_subqueries;
```

| 加数 | 为什么 |
|---|---|
| `n_sum_items` | 本层聚合/窗口函数，`split_sum_funcs` 后每个 `Item_sum` 要一槽 |
| `n_child_sum_items` | **内层**子查询的聚合若依赖本层列，会被 `inner_sum_func_list` 提升为本层 hidden item |
| `fields.size()` | 已含 hidden 前缀的 select/group/order 表达式 |
| `select_n_having_items` | SELECT+HAVING+ORDER BY 里可能再被拆出的子表达式（★ 由 `Item::itemize()` 在 parse 期累加） |
| `select_n_where_fields` | WHERE 里可能被 decorrelate 成派生表等值条件的表达式 |
| `order_group_num` | (ORDER BY + GROUP BY) ×2（`find_order_in_list()` 可能新增） |
| `n_scalar_subqueries` | 标量子查询 `scalar_to_derived` 改造后可能替换成字段 |

★ `Query_arena *arena = thd->stmt_arena`：PS 下数组分配在**语句 arena**（跨执行存活）——这是"PS 能复用 QB"的第一块基石。另有 `m_saved_base_items`（prepare 末尾的前缀快照）供重执行回填。

### 10.2 `Query_expression` 成员全景

```cpp
// [Q1]
  Query_expression *next;  Query_expression **prev;   // 同一 QB 内的兄弟 QE
  Query_block *master;                                // 本 QE 属于哪个 QB（顶层为 NULL）
  Query_block *slave;                                 // 本 QE 的第一个 QB
  Query_term *m_query_term{nullptr};                  // ★ 8.0.31 后唯一的树根
  Query_result *m_query_result;                       // 顶层结果出口
  bool prepared; bool optimized; bool executed;
  enum_clean_state cleaned;                           // UC_DIRTY / UC_PART_CLEAN / UC_CLEAN
  unique_ptr_destroy_only<RowIterator> m_root_iterator;
  AccessPath *m_root_access_path = nullptr;
  PT_with_clause *m_with_clause;
  Query_block *first_recursive;                       // 递归 CTE
  table_map m_lateral_deps;                           // LATERAL 依赖位图
```

★ **8.0.39 已不存在 `union_result` / `m_union_rec` / `fake_query_block`**——set-op 的临时表改由 `Query_term` 自己持有（`m_setop_query_result` → `Query_result_union` 的 `TABLE *table`、`m_result_table`），后处理块由 `Query_term_set_op::m_block` 承担。

#### ★ 三条链必须分清

```
  纵向：master/slave（QE 拥有 QB，QB 拥有内层 QE）

        LEX
         │ unit  ("顶层 QE"，永远是 lex->unit，master==nullptr)
         ▼
   ┌───────────────────────────┐
   │ Query_expression  #1      │
   │  m_query_term ────────────┼──► Query_term 树
   │  slave                    │
   └─────────────┬─────────────┘
                 │ slave = 第一个 Query_block
                 ▼
   ┌───────────────────────────┐      next（兄弟链：同一 QE 内的 QB）
   │ Query_block A             │ ───────────┐
   │  master ──────────────────┼──► QE #1   │
   └───────────────────────────┘           ▼
   ┌───────────────────────────┐      next
   │ Query_block B             │ ──────────► nullptr
   └───────────────────────────┘
                 │ slave / next（QE 兄弟链：同一 QB 内的子查询/DT/CTE）
                 ▼
   ┌───────────────────────────┐      next
   │ Query_expression  #2      │ ──────────► Query_expression #3 ─► nullptr
   └───────────────────────────┘

  另有一条"全局链" LEX::all_query_blocks_list（link_next/link_prev），
  串起语句里**所有** QB，供 LEX::clear_execution() 遍历。
```

| 链 | 指针 | 用途 |
|---|---|---|
| QB 兄弟链 | `QE::slave` + `QB::next` | prepare/optimize/execute 主体循环 |
| QE 兄弟链 | `QB::slave` + `QE::next/prev` | 遍历某 QB 下的所有子查询/派生表/CTE |
| QB 全局链 | `LEX::all_query_blocks_list` | 语句级：`clear_execution()`、EXPLAIN 编号 |

顶层 QE 是**假的**概念——它只是 `LEX::unit`，语义上就是普通 `Query_expression`，只不过 `master == nullptr`。

### 10.3 ★ `Query_term` 继承与三者关系

```cpp
// [Q3] —— 全部是普通 public 单继承，★ 没有虚继承、没有多重继承
class Query_term {                       // 抽象基类
 public:
  virtual Query_term_type term_type() const = 0;
  virtual const char *operator_string() const = 0;
 protected:
  Query_term_set_op *m_parent{nullptr};
  uint m_sibling_idx{0};
};

class Query_term_set_op : public Query_term {
  /// Replaces the old "fake" query block for post processing result set
  /// with ORDER BY, LIMIT.
  Query_block *m_block{nullptr};
 public:
  mem_root_deque<Query_term *> m_children;
  int64_t m_last_distinct;  int64_t m_first_distinct;
  Query_block *query_block() const override { return m_block; }
};

class Query_term_union : public Query_term_set_op { ... };
class Query_term_intersect : public Query_term_set_op { ... };
class Query_term_except : public Query_term_set_op { ... };
class Query_term_unary : public Query_term_set_op { ... };

// [Q1] —— Query_block 是★直接★继承 Query_term
class Query_block : public Query_term {
  Query_term_type term_type() const override { return QT_QUERY_BLOCK; }
  Query_block *query_block() const override {
    return const_cast<Query_block *>(this);      // ★ 叶子：返回 this
  }
  void label_children() override {}              // 没有孩子
};
```

⇒ **正确层次是两分支，不是一条链**：

```
                            Query_term  (abstract)
                                  │
                ┌─────────────────┴──────────────────┐
          Query_block                          Query_term_set_op (abstract)
          QT_QUERY_BLOCK                             │
          (叶子，query_block() 返回 this)             │
                                    ┌────────────────┼────────────────┬──────────────┐
                            Query_term_unary   Query_term_union  Query_term_intersect  Query_term_except
```

★ 三个易错点：① `Query_block` **不是** `Query_term_set_op` 的子类（它没有 `m_children`）；② `Query_term_unary` 是 `Query_term_set_op` 的子类（退化的一元 set-op，只为挂 ORDER BY/LIMIT）；③ 全非虚继承，向下转型必须显式 `down_cast<>`。

#### ★ 三者关系总图

```
  例子：((SELECT a FROM t1 UNION SELECT a FROM t2 ORDER BY a LIMIT 5)
         INTERSECT SELECT a FROM t3) ORDER BY a LIMIT 3;

    Query_expression (lex->unit)
    ├── m_query_term ──────────────────────────────┐
    ├── m_query_result = Query_result_send          │
    ├── slave ──► QB(t1) ─next─► QB(t2) ─next─► QB(t3) ─► null
    │              （真 SELECT 的 QB 全在兄弟链上）   │
    └──────────────────────────────────────────────┘
                                                    ▼
          ┌──────────────────────────┐   m_block（后处理块）
          │ Query_term_unary         │ ─► Query_block ORDER BY a LIMIT 3
          │ m_parent = nullptr       │   linkage=GLOBAL_OPTIONS_TYPE
          └───────────┬──────────────┘   select_number = INT_MAX（★不在兄弟链上）
                      │ m_children[0]
                      ▼
          ┌──────────────────────────┐   m_block
          │ Query_term_intersect     │ ─► Query_block
          └───────┬──────────┬───────┘
                  ▼          ▼
        ┌──────────────────┐  ┌──────────────────┐
        │ Query_term_union │  │ Query_block      │ ← 叶子 SELECT a FROM t3
        │ m_block ─────────┼─►│ Query_block      │   query_block() 返回 this
        └────┬────────┬────┘  └──────────────────┘
             ▼        ▼
     ┌───────────────┐ ┌───────────────┐
     │ Query_block   │ │ Query_block   │ ← 真 SELECT（在兄弟链上）
     └───────────────┘ └───────────────┘

  读法：竖线（m_children）= 语法嵌套结构；横线（slave/next）= 执行顺序；
        每个非叶子的 m_block = 后处理块，只能经 Query_term::query_block() 到达。
```

⇒ 后处理块用 `include_standalone()` 挂 `master` 但**故意不链 `next`**，所以主体循环 `for (sl = first_query_block(); sl; sl = sl->next_query_block())` 永远走不到它。这也是 `Query_expression::cleanup()` 要用 `query_terms<QTC_PRE_ORDER>()` 遍历**树**（而非兄弟链）的原因——只有树能覆盖后处理块。

### 10.4 ★ 生命周期

```
  ┌──────────┬───────────────┬──────────────┬──────────────┬──────────┬──────────┐
  │ 解析前     │ contextualize │ prepare      │ optimize     │ execute  │ cleanup  │
  ├──────────┼───────────────┼──────────────┼──────────────┼──────────┼──────────┤
QE│ new_top_  │ 填充/新建      │ prepared=true│ 建 access    │ executed │ cleaned  │
  │ level_    │ m_query_term  │ 建 result    │ path/iter    │ =true    │ destroy()│
  │ query()   │（见 11 篇）    │ tmp table    │              │          │          │
  ├──────────┼───────────────┼──────────────┼──────────────┼──────────┼──────────┤
QB│ 同上建1个  │ fields/where/ │ setup_base_  │ new JOIN     │ 跑 iter  │ cleanup()│
  │ 空 QB     │ group/order   │ ref_items()  │ join=…       │          │ join=null│
  ├──────────┼───────────────┼──────────────┼──────────────┼──────────┼──────────┤
QT│ 无        │ 建树 +        │ prepare_     │ setup_       │ open_    │ destroy_ │
  │(is_simple)│ pushdown_     │ query_term() │ materialize_ │ result_  │ tree()   │
  │          │ limit_order_by│ 递归后序       │ set_op()     │ tables() │          │
  └──────────┴───────────────┴──────────────┴──────────────┴──────────┴──────────┘
                                    ▲                                      │
                                    └──── PS 重执行只回退到这条线 ──────────┘
                              clear_execution() + restore_cmd_properties()
                              + bind_fields()；QE/QB/Item 全部复用，PT 已丢弃
```

#### 10.4.1 解析前：为什么必须预建顶层 QE + QB

```cpp
// [Q2]
bool LEX::new_top_level_query() {
  assert(unit == nullptr && query_block == nullptr);
  query_block = new_query(nullptr);
  if (query_block == nullptr) return true;
  unit = query_block->master_query_expression();
  return false;
}
```

`lex_start()` 注释自己承认这是历史包袱：

```cpp
  note: The function creates a query_block and a query_block_query_expression
  object.  These objects should rather be created by the parser bottom-up.
```

★ **为什么必须在解析前建**：MySQL 的 parser 是"**边归约边 contextualize**"（非纯语法树），yacc action 需要一个**已存在的"当前 QB"作为落点**——`SELECT` 列表往 `fields` 里 `add_item_to_list()`、`FROM` 往 `m_table_list`、`WHERE` 落到 `m_where_cond`。这些都是**写入槽位**而非构造对象。若改成"纯解析器 + 自底向上构造"，这步可取消（注释里的 TODO 正此意）。

#### 10.4.2 prepare：`base_ref_items` 的分配点

顺序不能颠倒：`*` 展开（`setup_wild`）会往 `fields` 加元素，而 `fields.size()` 是数组大小的加数之一。

```cpp
  if (with_wild && setup_wild(thd)) return true;
  if (setup_base_ref_items(thd)) return true;   // ★ 数组在此定长
  if (setup_fields(thd, ...)) ...
```

非叶子走 `prepare_query_term()` **自底向上**递归：先递归孩子，再为本层建 `Query_result_union`（临时表）并 `set_owning_operand()`。

#### 10.4.3 ★★ PS 重执行：为什么 QE/QB 能复用而 PT 不能

核心事实：**PT 树在 contextualize 之后就被丢弃**（只存活到 prepare 前）；QE/QB/QT 是"结构对象"，只描述结构、不含一次性语法信息，故可跨执行存活。**但每个 Item 里缓存的 `Field *` / `TABLE *` 指向上一轮打开的 TABLE，必须重新绑定。**

四步（顺序固定）：

**① `LEX::clear_execution()`**——遍历全局链 `all_query_blocks_list` 覆盖**所有** QE（包括后处理块所属的），清 `optimized/executed/cleaned`；★ **`prepared` 不清**（resolved 的 Item 树是复用的核心资产）。

**② `Query_block::restore_cmd_properties()`**：

```cpp
// [Q2]
  // Restore base_ref_items. Do this before we dive into subqueries, so that
  // their outer references point to valid items when they update used tables.
  std::copy(m_saved_base_items.begin(), m_saved_base_items.end(),
            base_ref_items.begin());
  ...
  assert(join == nullptr);
  cond_count = saved_cond_count;
  // Restore GROUP BY / ORDER BY list  ← 用 *_ptrs 快照重建 next 链
```

三件事：`base_ref_items` **前缀回填**（hidden 槽位在 optimize 期可能被改写）、`cond_count` 回填、GROUP/ORDER 的 `next` 链重建（这就是 `order_list_ptrs` 存在的唯一理由）。

**③ `bind_fields()`**（`[Q8]` / `[Q10]`）：

```cpp
void bind_fields(Item *first) {
  for (Item *item = first; item; item = item->next_free) item->bind_fields();
}
bool Sql_cmd_dml::restore_cmd_properties(THD *thd) {
  lex->restore_cmd_properties();
  bind_fields(thd->stmt_arena->item_list());   // ★ arena 里登记的全部 Item
  return false;
}
```

`Item::bind_fields()` 是虚函数（默认空）；`Item_field::bind_fields()` 按 `field_index` 重取 `Field*`。`item.h` 的注释是理解整套机制的钥匙：

```cpp
    ... Item classes should rather reference Table_ref and Item_field objects
    instead of TABLE and Field, because these classes support dynamic rebinding
    of objects before each execution. See Item::bind_fields() which binds new
    objects per execution and Item::cleanup() that deletes references to such
    objects.
```

**④ 反向 `cleanup()` 解绑**：`cleanup_items()` 逐 Item `cleanup()` 清掉悬垂引用；`Query_block::cleanup(full)` 中 `full=false` 是"部分清理"（保留 JOIN 供 `EXPLAIN FOR CONNECTION`），`full=true` 才 `::destroy(join)` 并 `join = nullptr`，且只有 full 才 `remove_hidden_items()`（让 `fields` 回到 prepare 结束的样子）。

⇒ **为什么"PT 不能复用"**：PT 携带语法细节（token 位置、未 resolve 的字面量、临时 `List<Item>`），且 `contextualize()` 是**一次性写入**语义（大量 `assert(first_execution)`）。重执行走 ①②③④，完全不经 PT。

#### 10.4.4 销毁：`destroy()` ≠ `cleanup()`

- `cleanup()` 让对象**可复用**；
- `destroy()` 释放**自带资源**（临时表、Window、JOIN 自有 mem_root），然后 `invalidate()` 把所有链表指针置空。

★ `Query_block::destroy()` 里那句注释是最终答案：

```cpp
  // Our destructor is not called, so we need to make sure
  // all the memory for these arrays is freed.
```

⇒ 所有 QE/QB/QT/Item 都用 `new (thd->mem_root)` 分配，**永不调用析构函数**（`Query_term::~Query_term()` 是 `= default` 且从未被调）。非 mem_root 资源（临时表句柄、ROLLUP 的堆数组）必须手工 `destroy()` 清理；内存本身随语句结束 `free_root(thd->mem_root)` 一次性蒸发。PS 则把对象放进 arena 长期持有，直到 `Prepared_statement` 销毁。

### 10.5 速查表

| 问题 | 答案（8.0.39） |
|---|---|
| 一个查询是一棵表达式树吗 | 不是。是 `fields / m_where_cond / m_having_cond / group_list / order_list / select_limit / offset_limit / m_windows` 等**独立槽位**的集合 |
| `base_options` 还在吗 | 不在。是 `m_base_options`(private) + `m_active_options`，对外只有 `active_options()` |
| `union_result` / `fake_query_block` 还在吗 | 都不在。改由 `Query_term::m_setop_query_result` + `m_result_table` + `Query_term_set_op::m_block` |
| `Query_block` 怎么继承 `Query_term` | `class Query_block : public Query_term`，**普通 public 单继承，非虚非多重**；`query_block()` 返回 `this` |
| `Query_term_unary` 在哪一层 | `Query_term_set_op` 的子类（退化的一元 set-op，只为挂 ORDER BY/LIMIT） |
| 后处理块在 `slave/next` 链上吗 | **不在**。用 `include_standalone()` 只挂 `master`，只能经 `Query_term::query_block()` 到达 |
| `base_ref_items` 何时分配 | `Query_block::prepare()` → `setup_base_ref_items()`；大小 = 7 项计数器之和（见 10.1.4） |
| PS 重执行四步 | `clear_execution()` → `restore_cmd_properties()` → `bind_fields()` →（上次末尾）`cleanup()` |
| 对象内存怎么释放 | **从不调析构**；`destroy()` 只清自有资源，内存随 `thd->mem_root` 释放 |

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


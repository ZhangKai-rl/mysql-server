# 04 Parser：从 token 流到 Parse Tree

> 本篇覆盖：语法分析如何把 token 流变成 **Parse Tree（第 ① 层 IR）**，以及它的内存归属。词法分析（字符流 → token 流）见 [02 Lexer](02_lexer.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、sql_yacc.yy 的整体结构](#一sql_yaccyy-的整体结构)
- [二、四个关键宏](#二四个关键宏)
- [三、Bison 生成物在哪](#三bison-生成物在哪)
- [四、LEX 结构](#四lex-结构)
- [五、关键产生式速查](#五关键产生式速查)
- [六、Parse Tree 节点体系](#六parse-tree-节点体系)
- [七、PTI_\* 的双重身份](#七pti_-的双重身份)
- [八、Parse Tree 的内存管理](#八parse-tree-的内存管理)
- [示例：一条 SQL 产生哪些 PT 节点](#示例一条-sql-产生哪些-pt-节点)

---

## 设计思想与理论基础

### 分阶段编译：词法 → 语法 → 语义

Parser 处于经典编译原理（Aho & Ullman 龙书）**三段式的前两段**：

| 阶段 | 本篇 | 输入 → 输出 | 特点 |
|---|---|---|---|
| 词法分析 | [02 Lexer](02_lexer.md) | 字符流 → token 流 | 无结构、正则可描述 |
| 语法分析 | 本篇主体 | token 流 → Parse Tree | **上下文无关**、上下文无关文法描述 |
| 语义分析 | 下一篇 04 | Parse Tree → 逻辑树 | 上下文相关、需要符号表 |

**为什么分阶段**：词法的正则文法无法表达 SQL 的括号嵌套、递归结构，而上下文无关文法（CFG）又无法表达"这个 ident 是列名还是表名"这类依赖上下文的判断。分层让每一层只处理它能形式化描述的那部分。

### 为什么用 LALR(1) 生成器，而非手写递归下降

MySQL 用 **Bison**（Yacc 的 GNU 版）从 `sql_yacc.yy` 生成 LALR(1) 表驱动解析器，而不是手写递归下降。权衡：

- **语法规模**：SQL 有上千条产生式、大量优先级/结合性声明（`%left`/`%prec`）。手写递归下降要为每条规则写函数，维护成本极高；表驱动的状态机统一处理
- **代价**：LALR(1) 的表对"归约/移入冲突"敏感（`%expect N` 声明了允许的冲突数），报错信息不如手写解析器友好
- **遗留痕迹**：`sql_yacc.yy` 里大量 `/* must be last */`、优先级 hack，是几十年增量演进的必然结果——没有一个"从头设计"的 SQL 解析器能躲开这些

### 为什么要有 PT / PTI 两层

8.0 引入了 `PT_*`（Parse Tree，纯语法节点）和 `PTI_*`（Parse Tree Item，表达式的语法节点）两套类，而不是像 5.x 那样在语义动作里直接 `new Item`。这层设计的意义：

- **语法与语义解耦**：Parse Tree 是纯语法的，不碰 `Field`、不碰 `TABLE`，所以同一个 Parse Tree 能被"contextualize"成不同的语义对象（也支撑了 `PREPARE` 一次、`EXECUTE` 多次）
- **可复用/可重写**：后续的 query rewrite 插件（`sql_query_rewrite.cc`）在 Parse Tree 层做改写，比在 Item 层安全得多

### 演进

- **5.x**：语义动作直接建 `Item`，语法树与语义对象纠缠，Parse Tree 概念不清晰
- **8.0**：正式确立 PT/PTI 类体系（`sql/parse_tree_nodes.h`），Parse Tree 成为独立的第 ① 层 IR——这是后面 contextualize（04）能独立成篇的前提

---

## 先有个整体印象

Parser 做的事就一件：**按语法规则把 token 流归约成一棵树**。

这棵树是**纯语法的、上下文无关的** —— 它只知道"这里有个 SELECT、有个 FROM、有个 WHERE"，**不知道** `t` 是哪张表、`a` 是哪一列、这个列存不存在。把语法树"落地"成有语义的结构，是下一篇 [04 contextualize](05_contextualize.md) 的事。

```
SELECT a FROM t WHERE b > 1 ORDER BY c
   │  词法
   ▼
[SELECT][ident a][FROM][ident t][WHERE][ident b][>][NUM 1][ORDER][BY][ident c]
   │  语法归约
   ▼
PT_select_stmt
  └─ PT_query_expression
      └─ PT_query_specification
          ├─ PT_select_item_list → [PTI_expr_with_alias → PTI_simple_ident_ident(a)]
          ├─ [PT_table_factor_table_ident(t)]
          ├─ PTI_where → PTI_comp_op(PTI(b), >, Item_int(1))
          └─ PT_order → PT_order_list → [PT_order_expr(PTI(c), ASC)]
```

---

## 一、sql_yacc.yy 的整体结构

文件共 **18300 行**，是 MySQL 源码里最大的文件之一。

| 区段 | 行号 | 内容 |
|------|------|------|
| `%{` 前导区开始 | `:32` | |
| 关键宏 | `:36-46` | `YYP` / `YYLIP` / `YYPS` / `YYMEM_ROOT` / `Lex` / `Select` |
| `yyoverflow` / `MYSQL_YYABORT` | `:192-220` | |
| **`#define NEW_PTN`** | **`:222`** | |
| `CONTEXTUALIZE` / `CONTEXTUALIZE_VIEW` / `ITEMIZE` / `MAKE_CMD` | `:228-270` | 见下节 |
| `yyerror` | `:299+` | |
| `#include` 区 | `:481-485` | 含 `parse_tree_items.h` |
| `%}` 前导区结束 | `:513` | |
| `%start start_entry` | `:515` | |
| `%parse-param {THD *YYTHD}` / `{Parse_tree_root **parse_tree}` | `:519-520` | **入参/出参** |
| `%define api.pure` | `:523` | 可重入 |
| `%expect 63` | `:529` | 预期 63 个移进/归约冲突 |
| `%token` 段 | `:585` ~ 约 `:1400` | |
| 优先级段 | `:1422-1458` | 含 `CONDITIONLESS_JOIN`(1435)、`SUBQUERY_AS_EXPR`(1454) |
| `%type` 段 | `:1460` ~ 约 `:2200` | |
| **`%%`** | **`:2208`** | **全文只有一个 `%%`**：规则区从 2209 到文件末尾，**没有 epilogue 段** |
| `start_entry:` | `:2232` | |
| `sql_statement:` | `:2278` | |
| **`simple_statement_or_begin`** | **`:2328-2329`** | `*parse_tree = $1;` ← **输出 Parse Tree 根** |

### `%expect 63` 说明什么

63 个移进/归约冲突，说明 MySQL 的语法**不是纯 LALR(1) 可解析的**。典型例子是 `WITH ROLLUP` —— 需要 LALR(2)，MySQL 在词法层特判绕过：

```cpp
// sql/sql_lex.cc:1336-1350（MYSQLlex 内）
// WITH + ROLLUP 的 LALR(2) → LALR(1) 特判
```

另一类冲突来自 SQL 语法本身的二义性（`CONDITIONLESS_JOIN` 用于 `STRAIGHT_JOIN` 之类），靠优先级声明解决。

---

## 二、四个关键宏

这四个宏是理解"parse 与 contextualize 如何交织"的钥匙：

```cpp
// sql/sql_yacc.yy:222
#define NEW_PTN new(YYMEM_ROOT)          // YYMEM_ROOT = YYTHD->mem_root（:40）
```

```cpp
// sql/sql_yacc.yy:228-235
#define CONTEXTUALIZE(x)                                    \
  do {                                                      \
    std::remove_reference<decltype(*x)>::type::context_t pc(YYTHD, Select); \
    if (YYTHD->is_error() ||                                            \
        (YYTHD->lex->will_contextualize && (x)->contextualize(&pc)))    \
      MYSQL_YYABORT;                                                    \
  } while(0)
```

```cpp
// sql/sql_yacc.yy:251-258
#define ITEMIZE(x, y)  /* 同理，控制 Item::itemize 是否被调用 */
```

```cpp
// sql/sql_yacc.yy:265-270
#define MAKE_CMD(x)  /* 在语法动作内部直接调 Lex->make_sql_cmd() */
```

**`MAKE_CMD` 的直接调用点**：`:4376, 4942, 8444, 8456, 8467, 16371, 16434` —— 这些语句（DDL、管理语句等）在**语法动作内部**就转成 `Sql_cmd`，`*parse_tree` 保持 NULL。

> **重要**：**只有 SELECT / DO 走 `THD::sql_parser` 里的 `make_sql_cmd`**（`sql_class.cc:3077`）。其他语句类型在 yacc action 里就完成了转换。

**`will_contextualize` 标志**（`sql/sql_lex.h:4372`）：

```cpp
/**
  Used to inform the parser whether it should contextualize the parse
  tree. When we get a pure parser this will not be needed.
*/
bool will_contextualize;
```

- 初值 `true`（`sql_lex.cc:3604`）
- **现状**：8.0.39 服务端代码里**没有任何地方把它置 false**，唯一赋值在单测里（`unittest/gunit/character_set_deprecation-t.cc:44`）
- 设计意图：支持"纯语法解析、推迟 contextualize"（view 定义、PS 二次解析、SP 预解析）。注释 "When we get a pure parser this will not be needed" 说明这是历史遗留开关

---

## 三、Bison 生成物在哪：CMake 构建期代码生成

**`sql_yacc.cc` / `sql_yacc.h` 不在源码树里，是 CMake 配置阶段调用 Bison 从 `sql_yacc.yy` 生成的。** 这是"构建期代码生成"——你 clone 下来的源码只有 `.yy` 语法源文件，`.cc` 是 `cmake` 时现生成的。

### 两条构建路径（`sql/CMakeLists.txt:1306-1363`）

CMake 先用 `FIND_PACKAGE(BISON)`（`cmake/bison.cmake`）探测系统有没有 bison，然后二选一：

```cmake
IF(BISON_FOUND)                    # 有 bison（开发机 / git clone）
  SET(USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT OFF)
ELSE()                             # 没 bison（release tarball 源码包）
  SET(USE_BISON_RESULTS_FROM_MAKE_DIST_DEFAULT ON)
ENDIF()
OPTION(USE_BISON_RESULTS_FROM_MAKE_DIST ...)
```

| 路径 | 触发条件 | 做法 |
|------|---------|------|
| **本地生成** | `BISON_FOUND`（装了 bison） | `BISON_TARGET(mysql_parser ...)` 从 `.yy` 现生成 `.cc` |
| **copy 预生成** | 没装 bison，或显式 `-DUSE_BISON_RESULTS_FROM_MAKE_DIST=ON` | 把 source tarball 里**已经预生成好的** `sql_yacc.cc/h` copy 到 build 目录 |

**为什么要预生成打进源码包**：发行版源码包要让用户"不装 bison 也能编译"，所以 release 时把生成物塞进 tarball（`CMakeLists.txt:1319-1333` 的 `FOREACH(genfile sql_yacc.h sql_yacc.cc ...)` 检查这些文件是否存在）。

### 本地生成的具体配置

```cmake
# sql/CMakeLists.txt:1336-1342
BISON_TARGET(mysql_parser
  ${CMAKE_CURRENT_SOURCE_DIR}/sql_yacc.yy      # 输入：语法源文件
  ${CMAKE_CURRENT_BINARY_DIR}/sql_yacc.cc      # 输出：生成的 parser C++ 代码
  COMPILE_FLAGS
  "--name-prefix=MYSQL --yacc ${BISON_FLAGS_WARNINGS} ${BISON_NO_LINE_OPT}"
  DEFINES_FILE ${CMAKE_CURRENT_BINARY_DIR}/sql_yacc.h   # 输出：token 定义头
  )
```

关键参数：

| 参数 | 作用 |
|------|------|
| `--name-prefix=MYSQL` | 所有 `yy*` 符号加前缀 ⇒ `yyparse→MYSQLparse`、`yylex→MYSQLlex`（见 02 篇） |
| `--yacc` | 兼容传统 yacc 语法（而不是 POSIX yacc） |
| `${BISON_FLAGS_WARNINGS}` | 警告开关 |
| `${BISON_NO_LINE_OPT}` | 控制 `#line` 指令 |
| `DEFINES_FILE` | 额外生成 `sql_yacc.h`（token 枚举 + `YYSTYPE` 声明，被 lexer/parser 代码 include） |

### 还有第二个 Bison 生成器

`sql/CMakeLists.txt:1344-1350` 还有一个 `hints_parser`（`sql_hints.yy`，`--name-prefix=HINT_PARSER_`），用于解析 `/*+ optimizer hint */` 注释里的 hint 语法，生成 `sql_hints.yy.cc`。它和主 parser 是两套独立的 LALR 状态机。

### 生成物长什么样

`sql_yacc.cc` 是 Bison 产出的**表驱动 parser**，含：
- **状态转移表**（`yypact`/`yydefact`/`yypgoto`/`yydefgoto`/`yytable`/`yycheck` 等常量数组）
- **归约动作**（`case N:` 里的语义动作代码，即 `.yy` 文件里 `{ }` 里的 `NEW_PTN`/`CONTEXTUALIZE` 等，被原样内联）
- **三个栈**（状态栈/语义值栈/位置栈，见本篇"八、Parse Tree 的内存管理"）

`sql_yacc.h` 是 token 定义头：`enum yytokentype`（`SELECT_SYM`/`FROM_SYM`/`IDENT`...）+ `YYSTYPE` 联合 + `MYSQLparse` 声明。lexer（`sql_lex.cc`）和 parser 都 include 它，共享 token 编号和语义值类型。

---

## 四、LEX 结构

> 注意：8.0 里 `LEX` 是 **`struct`** 不是 class：

```cpp
// sql/sql_lex.h:3876
struct LEX : public Query_tables_list {
  friend bool lex_start(THD *thd);

  Query_expression *unit;                 ///< 最外层 Query_expression    :3879
  Query_block *query_block;               ///< 第一个 Query_block         :3881
  Query_block *all_query_blocks_list;     ///< 全部 Query_block 的链表     :3882
 private:
  Query_block *m_current_query_block;     ///< 解析期"当前"QB             :3885
 public:
  inline Query_block *current_query_block() const { ... }        // :3888
  inline void set_current_query_block(Query_block *select) { ... } // :3897
```

| 成员 | 位置 |
|------|------|
| `unit` | `sql/sql_lex.h:3879` |
| `query_block` | `:3881` |
| `all_query_blocks_list` | `:3882` |
| `m_current_query_block` + 访问器 | `:3885 / 3888 / 3897` |
| `sql_command` | `:2675`（在基类 `Query_tables_list`） |
| `m_sql_cmd` | `:4090` |
| `parsing_options`（类型 `st_parsing_options`） | 成员 `:4182`；类型 `:3306-3312` |
| `will_contextualize` | `:4372` |
| `make_sql_cmd` 声明 | `:4561` |

`LEX::make_sql_cmd` 实现（`sql/sql_lex.cc:4969`）：

```cpp
bool LEX::make_sql_cmd(Parse_tree_root *parse_tree) {
  if (!will_contextualize) return false;            // :4970
  m_sql_cmd = parse_tree->make_cmd(thd);            // :4973
  if (m_sql_cmd == nullptr) return true;
  assert(m_sql_cmd->sql_command_code() == sql_command);   // :4976
  return false;
}
```

其它关键方法：`new_empty_query_block` `sql_lex.cc:587`、`create_query_expr_and_block` `:598`、`new_query` `:645`、`new_set_operation_query` `:704`、`new_top_level_query` `:780`。

---

## 五、关键产生式速查

| 语法结构 | 产生式行号 | 产出的 PT 对象 |
|----------|-----------|---------------|
| **输出根节点** | `:2328-2329` | `*parse_tree = $1` |
| `select_stmt` | `:9824` | `PT_select_stmt`（`:9827`） |
| `query_expression` | `:9920` | `PT_query_expression`（`:9925`）；带 WITH 版 `:9932` |
| `query_expression_body` | `:9936` | `PT_union`(9947) / `PT_except`(9952) / `PT_intersect`(9957) |
| **`query_specification`** | **`:9988`** | `PT_query_specification`（`:9999` / `:10020`） |
| `opt_from_clause` / `from_clause` / `table_reference_list` | `:10034 / 10039 / 10048` | |
| `select_item` | `:10182` | `PTI_expr_with_alias`（`:10186`） |
| **`expr`** | **`:10205`** | |
| `bool_pri` / `predicate` | `:10254 / 10283` | `PTI_comp_op` 在 `:10265` |
| `table_reference` | `:11839` | |
| **`joined_table`** | **`:11936`** | `PT_joined_table_on`(11939) / `using`(11944) / `cross_join`(11957)；结合性大段注释 `:11858-11935` |
| `table_factor` | `:12036` | |
| `single_table` | `:12062` | `PT_table_factor_table_ident`（`:12065`） |
| `derived_table` | `:12074` | `PT_derived_table`（`:12086` / `:12094`） |
| `where_clause` | `:12395` | `PTI_where`（`:12396`） |
| `opt_having_clause` | `:12399` | `PTI_having`（`:12403`） |
| `order_clause` / `order_list` | `:12586 / 12593` | `PT_order`(12589) / `PT_order_list`(12601) |
| `group_list` | `:12527` | |
| `opt_limit_clause` | `:12618` | |
| `order_expr` | `:14833` | `PT_order_expr`（`:14836`） |
| `simple_ident` | `:14847` | `PTI_simple_ident_ident`（`:14850`） |
| `table_subquery` / `subquery` | `:17320 / 17324` | `PT_subquery`（`:17327`） |

---

## 六、Parse Tree 节点体系

### 基类

```cpp
// sql/parse_tree_nodes.h:159
class Parse_tree_root {
  ...
 public:
  virtual Sql_cmd *make_cmd(THD *thd) = 0;      // ← 唯一接口
};
```

```cpp
// sql/parse_tree_node_base.h:149
template <typename Context>
class Parse_tree_node_tmpl { ... };
typedef Parse_tree_node_tmpl<Parse_context> Parse_tree_node;   // :256
```

基类 `contextualize()`（`parse_tree_node_base.h:199`）：

```cpp
virtual bool contextualize(Context *pc) {
  uchar dummy;
  if (check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy)) return true;
#ifndef NDEBUG
  assert(!contextualized);
  contextualized = true;      // 打标记，防止重复 contextualize
#endif
  return false;
}
```

### PT_* 主要节点（全部在 `sql/parse_tree_nodes.h`）

| 类 | 类定义行 | `contextualize` 实现 | 做了什么 |
|---|---------|---------------------|---------|
| `PT_select_stmt` | 1690 | `parse_tree_nodes.cc:705`（是 `make_cmd` 不是 contextualize） | 顶层入口，最后返回 `Sql_cmd_select` |
| `PT_query_expression` | 1463 | `parse_tree_nodes.cc:3988` | 压 QueryLevel、处理 ORDER/LIMIT、组装 Query_term |
| `PT_query_specification` | 1344 | `parse_tree_nodes.cc:1170` | **核心**：填 fields / table_list / where / having / group / order |
| `PT_select_item_list` | — | `parse_tree_nodes.cc:3613` | `pc->select->fields = value` |
| `PT_table_factor_table_ident` | 424 | `parse_tree_nodes.cc:3639` | `add_table_to_list()` 建 `Table_ref`（`:3645`） |
| `PT_derived_table` | 484 | `parse_tree_nodes.cc:1331` | 递归 contextualize 子查询、建 `Table_ident`、处理 LATERAL |
| `PT_joined_table_on` | 583 | `parse_tree_nodes.cc:3679` | 建 name resolution context、`add_join_on` |
| `PT_joined_table_using` | 596 | `parse_tree_nodes.cc:3706` | `add_join_natural` |
| `PT_cross_join` | 571 | `parse_tree_nodes.cc:3673` | `nest_last_join` |
| `PT_order` | 202 | `parse_tree_nodes.cc:288` | 切 `parsing_place=CTX_ORDER_BY`、填 `order_list` |
| `PT_group` | — | `parse_tree_nodes.cc:252` | 填 `group_list`，方向改 `ORDER_NOT_RELEVANT` |
| `PT_limit_clause` | — | `parse_tree_nodes.cc:3619` | `select_limit` / `offset_limit` |
| `PT_subquery` | — | `parse_tree_nodes.cc:4109` | `lex->new_query()` 建子 QE+QB |
| `PT_with_clause` | — | `parse_tree_nodes.cc:1945` | 挂 `m_with_clause` |

### 表达式节点 `PTI_*`（`sql/parse_tree_items.h`）

它们继承 `Parse_tree_item`（`parse_tree_helpers.h:74`），而 `Parse_tree_item : public Item`，`Item : public Parse_tree_node` —— **双重身份**，见下节。

| 类 | .h 行 | itemize 实现 | 转化结果 |
|---|-------|-------------|---------|
| `PTI_comp_op` | 59 | `parse_tree_items.cc:166` | `Item_func_gt/eq/lt/...`（`:171`） |
| `PTI_simple_ident_ident` | 99 | `:353` | `Item_field`（`:379`）或 `Item_ref`（`:381`） |
| `PTI_simple_ident_q_2d` / `_3d` | 134 / 115 | `:409 / 388` | table.col / db.table.col |
| `PTI_singlerow_subselect` | 434 | `:313` | 标量子查询 |
| `PTI_exists_subselect` | 446 | `:320` | EXISTS |
| `PTI_expr_with_alias` | 489 | `:335` | 设置 `item_name` |
| `PTI_where` / `PTI_having` | 558 / 564 | `:684`（`PTI_context`） | 切 parsing_place + `make_condition` |
| `PTI_truth_transform` | 46 | `:474` | IS TRUE / IS FALSE / NOT |
| `PTI_text_literal` | 239 | — | **直接继承 `Item_string`**（非占位符） |
| `PTI_count_sym` | 412 | `:582` | **直接继承 `Item_sum_count`** |

---

## 七、PTI_* 的双重身份

这是 MySQL 与很多数据库不同的一点（也是"没有独立 AST 层"的根因）。

继承链：

```
Parse_tree_item  :  public Item
                            ↑
Item  :  public Parse_tree_node      (item.h:882)
```

所以 `PTI_*` 既是语法节点、又是表达式节点。`parse_tree_helpers.h:58-73` 的注释说得很清楚：它们是**占位符（placeholder）**，真正的 Item 类型要到 `contextualize`/`itemize` 阶段才决定。

### 设计思想：为什么要有 PTI（语法与语义解耦）

`PTI_*` 的存在，本质是解决一个硬约束：**语法分析器遇到表达式时，还不知道它是什么语义**。

解析 `WHERE b > 1` 时，语法分析器只知道"有个比较运算 `>`，左边 `b`，右边 `1`"。但它**不知道**：

- `b` 是列、还是函数、还是聚合？—— 要查表定义（语义阶段才知道）
- `>` 是普通比较、还是子查询的 `> ANY`？—— 也依赖上下文

所以语法阶段**不能**直接 `new Item_func_gt(...)`（那是语义对象，需要知道操作数类型），只能先 new 一个占位符 `PTI_comp_op`，只记录"左操作数 + 比较符 + 右操作数"这三个**语法事实**。

这个"占位符"设计带来三个好处：

1. **语法与语义解耦**：Parse Tree 是纯语法的，不碰 `Field`/`TABLE`。同一棵 Parse Tree 能被 contextualize 成不同的语义对象——`PTI_simple_ident_ident` 可能变成 `Item_field`（列）也可能变成 `Item_ref`（别名），取决于上下文。
2. **支撑 PREPARE 一次、EXECUTE 多次**：纯语法的 Parse Tree 可以缓存，每次 EXECUTE 重新 contextualize（不同的绑定变量 → 不同的语义对象）。
3. **可改写**：query rewrite 插件（`sql_query_rewrite.cc`）在 Parse Tree 层改写，比在 Item 层安全——语法树还没绑定语义，改写不会破坏类型/引用。

> 所以 `PTI_*` 是"**语法与语义解耦**"设计思想的产物，C++ 的多重继承（`Parse_tree_item : Item : Parse_tree_node`）+ 占位符模式只是实现手段，不是设计本身。对比 `PTI_text_literal`、`PTI_count_sym`：它们直接继承 `Item_string`/`Item_sum_count`，因为字符串字面量和 COUNT 的语义在语法阶段就确定了，不需要占位符。

### 例子：`PTI_comp_op` 如何"变身"

语法动作（`sql_yacc.yy:10265`）只创建一个占位节点，记录"左操作数 / 比较类型 / 右操作数"：

```cpp
// sql/parse_tree_items.cc:166
bool PTI_comp_op::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res) || left->itemize(pc, &left) ||
      right->itemize(pc, &right))
    return true;

  *res = (*boolfunc2creator)(false)->create(left, right);   // :171
  return *res == nullptr;
}
```

**关键在 `:171`**：`PTI_comp_op` 本身被**丢弃**，`*res` 换成新建的 `Item_func_gt` / `Item_func_eq` / …

这就是"占位符 → 真实 Item"的典型模式：**PTI 节点是"待定类型的壳"，itemize 时决定壳里装什么，然后把壳换掉**。

### 例子：`PTI_where` 如何切上下文

```cpp
// sql/parse_tree_items.cc:684
bool PTI_context::itemize(Parse_context *pc, Item **res) {
  if (super::itemize(pc, res)) return true;
  pc->select->parsing_place = m_parsing_place;      // :687  CTX_WHERE / CTX_HAVING
  if (expr->itemize(pc, &expr)) return true;
  if (!expr->is_bool_func()) {
    expr = make_condition(pc, expr);                // :692  非布尔 → 包一层 truth test
    if (expr == nullptr) return true;
  }
  pc->select->parsing_place = CTX_NONE;             // :698
  expr->apply_is_true();                            // :700
  *res = expr;
  return false;
}
```

`parsing_place` 很重要：它决定 `Item::itemize` 里的统计（`select_n_having_items++`）和后续名字解析的合法性判断。

---

## 八、Parse Tree 的内存管理

```cpp
// sql/sql_yacc.yy:40
#define YYMEM_ROOT (YYTHD->mem_root)
// sql/sql_yacc.yy:222
#define NEW_PTN new(YYMEM_ROOT)
```

对应 placement new（`parse_tree_node_base.h:163`）：

```cpp
static void *operator new(size_t size, MEM_ROOT *mem_root,
                          const std::nothrow_t & = std::nothrow) noexcept {
  return mem_root->Alloc(size);        // :166
}
static void operator delete(void *ptr, size_t size) {
  TRASH(ptr, size);                    // :170  只做内存填充标记，不释放
}
```

**三条结论**：

1. **所有 PT 节点都在 `thd->mem_root`**，从不逐个 `delete`
2. contextualize 完成后 **PT 节点不再被任何指针引用**（`THD::sql_parser` 里的 `root` 是局部变量，`sql_class.cc:3061`）
3. 语句结束 `mem_root->ClearForRecy()`（`sql_parse.cc:2522`）整块回收 —— 详见 [01](01_protocol_to_dispatch.md#七mem_root为什么-parse-tree-从不-delete)

### 9.1 reentrant：解析器如何拿到 mem_root

Bison 生成的解析器是 **reentrant（可重入）** 的——不依赖全局变量，状态全通过参数传递。`sql_yacc.yy:519-523`：

```yacc
%parse-param { class THD *YYTHD }                 // yyparse 第一个参数是 THD*
%parse-param { class Parse_tree_root **parse_tree } // 第二个参数是解析树出参
%lex-param   { class THD *YYTHD }                 // yylex 也拿到 THD*
%define api.pure                                   // 可重入（线程安全）
```

`api.pure` 是 key：传统 Bison 用全局变量 `yylval`/`yychar` 传状态，多线程会踩踏；`api.pure` 改成参数传递，每个线程传自己的 THD。宏链把 THD 连到 mem_root（`sql_yacc.yy:40/222`）：

```
YYMEM_ROOT = YYTHD->mem_root
NEW_PTN    = new(YYMEM_ROOT) = new(YYTHD->mem_root)   ← placement new 到 mem_root
```

### 9.2 语义值栈：`YYSTYPE` 数组，在函数栈上

Bison 的解析器维护三个栈（生成的 `sql_yacc.cc:25057-25069`）：

```c
yy_state_t yyssa[YYINITDEPTH];   // ① 状态栈（语法状态机当前状态）
YYSTYPE    yyvsa[YYINITDEPTH];   // ② 语义值栈（每个符号的值）★
YYLTYPE    yylsa[YYINITDEPTH];   // ③ 位置栈（符号在源码的位置）
```

**语义值栈是 `YYSTYPE` 数组，初始在函数栈上**（`yyvsa[100]` 局部数组）；不够时 `YYSTACK_ALLOC`（= `alloca`，`sql_yacc.cc:2642`）扩容，**依然在栈上**，上限 `YYMAXDEPTH=3200`。每个语法符号占一个槽位，槽位内容是 `YYSTYPE` union（`parser_yystype.h:340`）。

### 9.3 指针型 vs 值型：union 成员的两种形态

关键前提：**语义值栈槽位在栈上**。槽位里的 union 成员分两种形态，对应两种内存归属：

| | 指针型 | 值型 |
|---|---|---|
| union 成员 | `Item *item`、`Table_ident *table` | `Mem_root_array_YY<...> table_reference_list` |
| 槽位存什么 | 8 字节指针 | **整个对象外壳**（4 成员 ~32 字节） |
| 对象住哪 | mem_root（堆） | **栈上的 union 槽位** |
| 数据住哪 | 对象内部 → mem_root | 外壳的 `m_array` → mem_root |

```
指针型：槽位(栈) ──8B 指针──→ 对象(mem_root 堆)

值型：  槽位(栈) 存整个 Mem_root_array_YY 外壳
            ├ m_root   ──→ 指向 THD::mem_root
            └ m_array  ──→ 元素数据(mem_root 堆)
```

**"值型"不是"数据在栈上"，而是"控制块（外壳）在栈上、数据在堆上"**。`Mem_root_array_YY` 的外壳（`m_root/m_array/m_size/m_capacity` 四个成员）躺在栈槽位里；它管理的元素数据（`m_array` 指向的连续内存）在 mem_root 堆上。

### 9.4 union 要求平凡类型 → 为什么 `_YY` 无 ctor/dtor

C++ union 的成员**必须是平凡类型（trivial）**——不能有非平凡构造/析构，因为 union 不知道当前激活哪个成员、无法自动调用谁的 ctor/dtor。

所以：
- **能放指针就放指针**（`Item *`、`Table_ident *`）——指针平凡
- **必须放值类型时**，就得让容器本身变平凡：`Mem_root_array_YY` 故意不写 ctor/dtor（`mem_root_array.h:415` 注释 `No CTOR/DTOR for this class!`），用 `init()` 手动初始化代替

```yacc
// sql_yacc.yy:7885（index_options 规则，值型用法）
%empty { $$.init(YYMEM_ROOT); }
  | fulltext_index_options
    {
      $$.init(YYMEM_ROOT);          // 在栈槽位里就地初始化外壳
      if ($$.push_back($1)) ...     // push_back 时数据才从 mem_root 分配
    }
```

对比有 ctor/dtor 的 `mem_root_deque`，只能作指针型：

```yacc
// sql_yacc.yy:2679
$$ = NEW_PTN mem_root_deque<Item *>(YYMEM_ROOT);   // 对象在 mem_root，槽位存指针
```

### 9.5 `init` 做了什么：只初始化外壳，不分配数据

```cpp
// mem_root_array.h:74-81
void init(MEM_ROOT *root) {
  m_root = root;       // 记住 mem_root 指针
  m_array = nullptr;   // 数据指针置空（还没分配）
  m_size = 0;          // 0 个元素
  m_capacity = 0;      // 0 容量
}
```

`init` 只初始化外壳 4 个成员，**一行内存都不分配**。真正的数据内存，到第一次 `push_back → reserve → m_root->Alloc(n * sizeof(T))`（`mem_root_array.h:155-158`）才从 mem_root 分配。

### 9.6 完整生命周期闭环

```
① yyparse 入口：状态栈/语义值栈/位置栈 都在函数栈上（yyvsa 等局部数组）
② lexer 填 token 值：yylex(yylval, yythd) 写 *yylval（数字→num、关键字→lexer.keyword）
③ 归约执行动作：
     指针型 → $$ = NEW_PTN X(args)，对象在 mem_root，槽位存指针
     值型   → $$.init(YYMEM_ROOT)，外壳在槽位就地 init，数据 push_back 时才分配
④ 根规则归约：$$ 是 Parse_tree_root*，经 parse_tree 出参传回
⑤ 语句结束：THD::mem_root 整体释放 → 指针型对象 + 值型数据 + Item 全回收
            栈上的值型外壳随函数返回自然消亡
```

**闭环关键**：指针型对象生命周期 = mem_root（语句级）；值型外壳生命周期 = 栈帧（归约期），但外壳里 `m_array` 的数据生命周期 = mem_root（语句级）。两者殊途同归——所有"数据"最终在 mem_root，语句结束统一回收；区别只在"控制块（外壳）"住栈上还是堆上。

---

## 示例：一条 SQL 产生哪些 PT 节点

`SELECT a FROM t WHERE b > 1 ORDER BY c`

| 语法规则 | 行号 | 产出 |
|---------|------|------|
| `simple_ident: ident` | `:14847-14851` | `PTI_simple_ident_ident(a)` |
| `select_item` | `:10182-10188` | `PTI_expr_with_alias(PTI(a), "")` |
| `select_item_list` | `:10176` | `PT_select_item_list` |
| `single_table` | `:12062-12066` | `PT_table_factor_table_ident(t)` |
| `where_clause: WHERE expr` | `:12395-12397` | `PTI_where(PTI_comp_op(PTI(b), >, Item_int(1)))`（`PTI_comp_op` 产生式 `:10265`） |
| `order_expr` | `:14833-14838` | `PT_order_expr(PTI(c), ORDER_ASC)` |
| `order_list` | `:12593-12606` | `PT_order_list` |
| `order_clause` | `:12586-12591` | `PT_order` |
| `query_specification` | `:9988-10032` | `PT_query_specification`（`:9999`） |
| `query_expression` | `:9920-9934` | `PT_query_expression`（`:9925`） |
| `select_stmt` | `:9824-9835` | `PT_select_stmt`（`:9827`） |
| `simple_statement_or_begin` | `:2328-2329` | **`*parse_tree = PT_select_stmt*`** |

产出后，这棵树会被交给 `LEX::make_sql_cmd` → `PT_select_stmt::make_cmd` → **contextualize**（见 [04](05_contextualize.md)）。

---


## 参考

**书籍 / 论文**
- **Aho & Ullman《Principles of Compiler Design》（龙书）** —— 词法分析、语法分析、Parse Tree 的经典分阶段思想

**官方文档**
- *MySQL 8.0 Reference Manual → Lexical Structure*
- *MySQL 8.0 Reference Manual → Keywords and Reserved Words*


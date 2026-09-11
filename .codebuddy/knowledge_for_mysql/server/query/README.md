# MySQL 查询处理管线（从收包到执行）

> 基于 MySQL 8.0.39 源码。本目录覆盖 server 层"SQL 文本 → 执行结果"的完整链路，重点是**中间表示（IR）的逐层转换**。

## 目录

- [这个目录讲什么](#这个目录讲什么)
- [一图看懂：6 层中间表示](#一图看懂6-层中间表示)
- [到底有几棵树](#到底有几棵树)
- [完整主链调用栈](#完整主链调用栈)
- [文件索引](#文件索引)
- [关键答疑](#关键答疑)
- [论文与设计思想](#论文与设计思想)
- [与其他文档的关系](#与其他文档的关系)

---

## 这个目录讲什么

一条 SQL 在 server 层的旅程：

```
客户端包 → 命令分发 → 词法/语法分析 → 语义分析 → 逻辑改写 → 优化 → 物理计划 → 执行器
```

**不包含**（各有专门文档）：

| 主题 | 去哪看 |
|------|--------|
| handler / handlerton 分层、行定位（`position`/`ref`/`rnd_pos`） | [`../handler.md`](../handler.md) |
| InnoDB 侧取行（`row_search_mvcc`、游标推进） | [`../../innodb/row_search.md`](../../innodb/row_search.md) |
| MVCC 可见性判断 | [`../../innodb/mvcc.md`](../../innodb/mvcc.md) |
| binlog 与提交 | [`../replication/binlog.md`](../replication/binlog.md) |

> 协议层（VIO / NET / 包格式 / 收发包）已并入 [01 篇](01_protocol_to_dispatch.md)，不再单独成文。

---

## 一图看懂：阶段 → IR → 文档

SQL 处理是**七个阶段、每阶段产出一棵树**。先看"动作 + 产出 + 文档"的总映射，再看下面的树图：

| 阶段（动作） | 产出 IR（树） | 文档 |
|-------------|--------------|------|
| ① 词法 lexer | token 流 | [02_lexer](02_lexer.md) |
| ①' 语法记法（怎么读官方文档 EBNF） | ——（语法篇的前置） | [03_grammar_notation](03_grammar_notation.md) |
| ② 语法 parser | Parse Tree（`PT_*`/`PTI_*`） | [04_parser](04_parser.md) |
| ③ 语义 contextualize（含 itemize） | 逻辑查询树 QE/QB/Query_term + Item 树 | [05_contextualize](05_contextualize.md) |
| ④ prepare（resolve + transform） | 绑定/改写后的 Item 树 + 逻辑树 | [06_resolver_prepare](06_resolver_prepare.md) |
| ⑤ optimize | AccessPath 物理计划 | [07_optimize](07_optimize/README.md) |
| ⑥ 迭代器化 | RowIterator 树 | [08_access_path](08_access_path.md) → [09_executor](09_executor_iterator.md) |
| ⑦ execute | 结果集 | [09_executor](09_executor_iterator.md) |

> **补充与易混点**：
> ① `itemize` **不是独立阶段**，是 ③ contextualize 的表达式子机制（`PT_*::contextualize` 管"结构归位"，`Item::itemize` 管"表达式归位"）。
> ② `resolve` 和 `transform` 是 ④ prepare 的**两个子阶段**（都在 `Query_block::prepare` 内部），resolve 管名字解析、transform 管永久改写。
> ③ **`rewrite`（重写）有两层**：**query rewrite 插件**（`invoke_pre_parse_rewrite_plugins`/`invoke_post_parse_rewrite_plugins`，parse 前后、**SQL 文本级**，见 [01 篇](01_protocol_to_dispatch.md)）+ **transform**（prepare 期、**逻辑树级**的等价变换，即 ④ 里的 transform）。前者是 Rewriter 插件按 DBA 规则改 SQL 文本，后者是优化器等价改写逻辑树。
> ④ **`planning`（规划）= `make_join_plan`**（`sql_optimizer.cc:696` 调，:5307 定义），是 join order 搜索（greedy + 限深 DFS）+ 访问方法选择，在 ⑤ optimize **内部**。标准术语里 "plan" 决定"怎么做"、"optimize" 优化"怎么做"，MySQL 把它们揉在 `JOIN::optimize` 一个函数里，所以上面没单独列"规划"阶段。

MySQL 处理 SQL 的本质是**一棵树变成下一棵树**。每一层都有自己的载体类和转换函数：

```
SQL 文本
   │  ① lexer                      MYSQLlex/lex_one_token   sql/sql_lex.cc:1304/1374
   ▼  token 流
   │  ② parser                     MYSQLparse (sql_yacc.yy → sql_yacc.cc)
   ▼
┌─────────────────────────────────────────────┐
│ Parse Tree    PT_* / PTI_*                  │  纯语法，上下文无关，用完即弃
│               sql/parse_tree_nodes.h        │
└─────────────────────────────────────────────┘
   │  ③ contextualize + itemize    PT_select_stmt::make_cmd()  parse_tree_nodes.cc:705
   │     ├ PT_*::contextualize（结构归位）
   │     └ Item::itemize（表达式归位）
   ▼
┌─────────────────────────────────────────────┐
│ 逻辑查询树    Query_expression /             │  一个 QB = 一个 SELECT 块
│               Query_block / Query_term       │  嵌套 = 子查询 / 集合操作
│               sql/sql_lex.h:766 / 1311       │
│  + Item 树    Item : public Parse_tree_node  │  表达式树，挂载在 QB 上
│               sql/item.h:882                 │
└─────────────────────────────────────────────┘
   │  ④ prepare                    Query_block::prepare()  sql/sql_resolver.cc:179
   │     ├ resolve：setup_* + fix_fields（列名→Field*，同一批 Item 升级）
   │     └ transform：flatten/simplify/merge（永久改写）
   ▼
┌─────────────────────────────────────────────┐
│ 已绑定/改写的 Item 树 + 逻辑树               │  fixed = true，semi-join/derived 已改写
└─────────────────────────────────────────────┘
   │  ⑤ optimize
   │     ├─（旁路）SEL_ARG / SEL_ROOT / SEL_TREE  区间红黑树森林      range_optimizer/tree.h
   │     └─（旁路）JOIN_TAB / QEP_TAB（仅旧优化器）                   sql_select.h:599
   ▼
┌─────────────────────────────────────────────┐
│ AccessPath 树  44 种 type                    │  物理计划，新旧优化器的统一 IR
│                join_optimizer/access_path.h  │
└─────────────────────────────────────────────┘
   │  ⑥ CreateIteratorFromAccessPath()   join_optimizer/access_path.cc:379
   ▼
┌─────────────────────────────────────────────┐
│ RowIterator 树  火山模型                     │  与 AccessPath 1:1 对应
│                iterators/row_iterator.h:82   │
└─────────────────────────────────────────────┘
   │  ⑦ ExecuteIteratorQuery()      sql/sql_union.cc:1672
   ▼
  执行
```

---

## 到底有几棵树

**共 8 棵树：6 棵主干 + 2 棵旁路**。它们之间是"三次换新树 + 一次升级"，不是"每阶段一棵新树"，也不是"一批对象贯穿到底"：

| # | 树 | 载体 | 谁生产 | 换新树? |
|---|----|------|--------|---------|
| ① | **Parse Tree** | `PT_*` / `PTI_*` | Bison 动作 `NEW_PTN` | —— |
| ② | **逻辑查询树** | `Query_expression` / `Query_block` / `Query_term` | `lex_start()` 预建 + contextualize 填充 | **换新树①**（PT→逻辑树） |
| ③ | **Item 表达式树** | `Item` 子类 | parse 期 `new` + `itemize` 注册 | **升级不换树**（fix_fields 就地填 `Field*`） |
| ④ | **Table_ref / join-nest 树** | `Table_ref` + `NESTED_JOIN` | `setup_tables` + contextualize | 随逻辑树原地改 |
| ⑤ | **AccessPath 树** | `AccessPath` | `create_access_paths()` / `FindBestQueryPlan()` | **换新树②**（逻辑树→物理计划） |
| ⑥ | **RowIterator 树** | `RowIterator` 子类 | `CreateIteratorFromAccessPath()` | **换新树③**（物理计划→迭代器） |

旁路（不占主链，局部/临时结构）：

| 树 | 用途 |
|----|------|
| **`SEL_ARG` / `SEL_ROOT` / `SEL_TREE`** | range 优化的区间红黑树森林，最终转成 `AccessPath::INDEX_RANGE_SCAN` |
| **`JOIN_TAB` / `QEP_TAB`** | 旧优化器的 join order / 执行计划表，被翻译成 AccessPath 后不再需要 |

**关键澄清**：
- 所谓"一批对象状态递进"**只适用于 Item 树**（③）：parse 创建 → itemize 归位 → fix_fields 绑定 → 优化改写 → 执行求值，自始至终同一批 `Item` 对象。
- ①→②（Parse Tree → 逻辑查询树）、②→⑤（逻辑树 → AccessPath）、⑤→⑥（AccessPath → RowIterator）是**三次真正的"换新树"**——每步都用全新的类层次（`PT_*` → `Query_*` → `AccessPath` → `RowIterator`）。
- ④ Table_ref 树容易被忽略，但它是一棵独立的树（FROM 子句的 join nest 结构），`simplify_joins`/`merge_derived` 在 prepare 阶段原地改写它。

---

## 完整主链调用栈

从连接线程到执行，每个函数带文件:行号（可直接当断点序列）：

```
sql/conn_handler/connection_handler_per_thread.cc:246   handle_connection()
 └─ :304  while (thd_connection_alive(thd)) { if (do_command(thd)) break; }
     │
     ├─ sql/sql_parse.cc:1308   do_command()
     │    ├─ :1381  thd->get_protocol()->get_command(&com_data, &command)
     │    │           └─ sql/protocol_classic.cc:2991  Protocol_classic::get_command()
     │    │                └─ :1410  read_packet() → my_net_read()
     │    └─ :1440  dispatch_command(thd, &com_data, command)
     │
     └─ sql/sql_parse.cc:1688   dispatch_command()
          ├─ :1809  mysql_audit_notify(MYSQL_AUDIT_COMMAND_START)
          └─ :2011  case COM_QUERY:
               ├─ :2016  alloc_query()
               └─ :2055  dispatch_sql_command()          ← 8.0.3x 起取代 mysql_parse()
                    ├─ :5253  lex_start()                → sql/sql_lex.cc:511
                    ├─ :5256  invoke_pre_parse_rewrite_plugins()
                    ├─ :5271  parse_sql()                → sql/sql_parse.cc:7067
                    │    └─ :7142  thd->sql_parser()     → sql/sql_class.cc:3048
                    │         ├─ :3067  MYSQLparse()          → ① Parse Tree
                    │         └─ :3077  lex->make_sql_cmd()   → sql/sql_lex.cc:4969
                    │              └─ PT_select_stmt::make_cmd()  parse_tree_nodes.cc:705
                    │                   └─ :711  contextualize()  → ② 逻辑查询树
                    ├─ :5272  invoke_post_parse_rewrite_plugins()
                    └─ :5374  mysql_execute_command()    → sql/sql_parse.cc:2946
                         └─ :4724  lex->m_sql_cmd->execute(thd)
                              └─ sql/sql_select.cc:675  Sql_cmd_dml::execute()
                                   ├─ :531  precheck()（粗粒度权限）
                                   ├─ :542  open_tables_for_query()
                                   ├─ :568  prepare_inner()  → Query_block::prepare()  → ③
                                   └─ :1011 unit->optimize()  → ④ AccessPath
                                        └─ sql/sql_union.cc:1128  CreateIteratorFromAccessPath()  → ⑤
                                             └─ :1672  ExecuteIteratorQuery()  → 执行

【收尾】
sql/sql_parse.cc:5419  thd->lex->destroy()
sql/sql_parse.cc:5420  thd->end_statement()      → lex_end()
sql/sql_parse.cc:5421  thd->cleanup_after_query() → 清理 Item 列表
sql/sql_parse.cc:2522  thd->mem_root->ClearForReuse()   ← Parse Tree 在此"消失"
```

---

## 文件索引

### 主链（按时间顺序，01 → 11）

| 文件 | 阶段 | 核心内容 |
|------|------|----------|
| [01_protocol_to_dispatch.md](01_protocol_to_dispatch.md) | 收包 → 命令分发 | **协议层**（VIO/NET/Protocol：4 字节包头、16MB 分片、length-coded integer、OK/ERR/EOF、回包）+ 握手认证、THD、线程模型、`do_command`、`dispatch_command`、LEX 生命周期、mem_root 一次性回收、Prepared Statement 差异 |
| [02_lexer.md](02_lexer.md) | ① 词法 | `MYSQLlex`、`lex_one_token` 状态机（逐状态）、`Lex_input_stream` 输入缓冲与回退、`find_keyword` 关键字识别、`YYSTYPE` 语义值回传、字符集驱动状态表、LALR(2) 特判 |
| [03_grammar_notation.md](03_grammar_notation.md) | ①' 语法记法 | **官方文档 EBNF 怎么读**：BNF/EBNF 记法、`[]`/`{}`/`\|`/`...` 每个符号的含义、EBNF ↔ sql_yacc.yy 互转（`[x]`↔`opt_x`、`x,...`↔`x_list`）、SELECT 语法逐行对照、与词法/语法解析的对应 |
| [04_parser.md](04_parser.md) | ② 语法 → Parse Tree | `sql_yacc.yy` 结构、`NEW_PTN` + mem_root、LEX 结构、PT/PTI 节点体系 |
| [05_contextualize.md](05_contextualize.md) | ①→② | `contextualize()` 全景、各类节点做什么、Query_term 树（8.0.31 重构）、Item 与 `itemize`、完整分步示例 |
| [06_resolver_prepare.md](06_resolver_prepare.md) | ②→③ | `Query_block::prepare()` 46 步、`setup_*` 函数族、`fix_fields`、name resolution、semi-join/derived 改写 |
| **[07_optimize/](07_optimize/README.md)** | ③→④ | **优化器（11 篇，子目录）**：代价模型与统计、`logical/` 逻辑优化（子查询/semi-join/连接简化/谓词）、`physical/` 物理优化（join order/访问方法/range/hypergraph）、计划改进 |
| [08_access_path.md](08_access_path.md) | ④ | AccessPath 44 种类型、新旧优化器两条创建路径、与 Iterator 1:1 |
| **[09_executor_iterator.md](09_executor_iterator.md)** | ⑤ | **RowIterator 火山模型（算法级）**：火山契约、`unlock_row` 三种例外、NLJ 的 NULL 补全状态机、HashJoin 三形态与 chunk 估算、BKA 的 MRR cookie、**index merge 三个迭代器**（多路归并取交集/堆归并去重/两阶段 Unique）、排序/物化在 `Init()`、显式栈翻译、`ExecuteIteratorQuery` |
| [10_dml.md](10_dml.md) | ⑤ DML 分支 | **DML 与 SELECT 的差异篇**（SELECT 是主链 02~09，不重复）：`Sql_cmd_dml` 类体系（SELECT 也继承它）、INSERT 的 `write_record` 链（不走迭代器）、UPDATE/DELETE 单表快路径 vs 多表迭代器、Halloween 问题与两阶段读、与 SELECT 在 read_set/write_set/ICP/覆盖索引的差异 |

### 横切内容（不占主链步骤）

| 文件 / 目录 | 说明 |
|------|------|
| **[runtime/](runtime/README.md)** | **执行期专题（6 篇）**：filesort 与临时表、子查询运行期、CTE、窗口函数、join buffer、ROLLUP。是 09 篇执行器能力的横向展开 |
| [11_explain_and_trace.md](11_explain_and_trace.md) | **调试与可观测性**（非主链步骤）：EXPLAIN（TRADITIONAL/TREE/JSON/ANALYZE）、optimizer trace。它横跨优化器与执行器，是"观察主链的工具" |

---

## 关键答疑

### Q：`contextualize()` 是 Parse Tree 的过程，还是逻辑查询树也有？

**A：只有 Parse Tree 有。逻辑查询树是产物，不是执行者。**

准确地说：

- `contextualize()` 是 **`Parse_tree_node` 的虚函数**（`parse_tree_node_base.h:199`）
- `Query_block` / `Query_expression` **没有** `contextualize()` 方法 —— 它们是**被创建和被填充的对象**
- 但有个容易混淆的点：**`Item` 也继承自 `Parse_tree_node`**（`item.h:882`），所以表达式节点也有 `contextualize()`。不过 Item 把它设成私有并 `assert(0)`（`item.h:1143`），真正的入口改叫 **`itemize()`**（`item.cc:631`）

所以调用关系是：

```
PT_select_stmt::make_cmd()             ← Parse Tree 根节点
  └─ PT_query_expression::contextualize()
      └─ PT_query_specification::contextualize()
          ├─ 创建/填充 Query_block（fields、table_list、where_cond、order_list …）
          ├─ PT_table_factor_table_ident::contextualize()  → 建 Table_ref
          └─ Item::itemize()  → 内部再调 Parse_tree_node::contextualize()
```

**一句话**：`contextualize` 是 Parse Tree 的方法，做的是"把语法树落地成 LEX + QE/QB + Item 树"。

### Q：为什么 MySQL 没有独立的 AST 层？

很多数据库是 Parse Tree → AST → 逻辑计划三步，MySQL 只有两步，因为 **`Item` 本身就是 `Parse_tree_node`**（`item.h:882`）：表达式在 parse 期创建的对象，到 contextualize 期归位，到 prepare 期绑定类型——**自始至终是同一批对象**，只是状态在变（`contextualized` → `fixed`）。

这带来两个后果：

1. 无法做"纯语法解析"后再决定语义（所以有 `will_contextualize` 标志做兼容，见 03）
2. Parse Tree 和 Item 树在表达式部分是**重合**的，`PTI_*` 节点既是语法节点又是 Item 占位符

### Q：新旧优化器怎么并存？

- **旧优化器**：`JOIN::optimize()` → 用 `QEP_TAB` 表示左深树 → `JOIN::create_access_paths()` 翻译成 AccessPath
- **新 hypergraph**：`FindBestQueryPlan()` 直接产出 AccessPath，完全绕过 QEP_TAB
- **汇合点**：两者都产出 AccessPath，然后走同一个 `CreateIteratorFromAccessPath()`
- 开关：`set optimizer_switch="hypergraph_optimizer=on"`（`sys_vars.cc:3472`，打开时有实验性警告）

---

## 论文与设计思想

| 论文 / 理论 | 影响哪一层 |
|-------------|-----------|
| **Aho & Ullman《Principles of Compiler Design》**（龙书） | Parse Tree → 语义分析的经典分阶段思想 |
| **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979，System R)** | 代价模型、选择性估算、左深树 DP —— MySQL 旧优化器的代价公式来源 |
| **Graefe《Volcano—An Extensible and Parallel Query Evaluation System》(1990)** | **火山模型（Volcano iterator model）** —— MySQL 8.0 的 `RowIterator` 就是它的实现：`Init()` / `Read()` 契约 |
| **Graefe《The Cascades Framework for Query Optimization》(1995)** | 现代优化器框架（规则 + 代价），MySQL 未采用，但是 hypergraph 重构的参照系 |
| **Moerkotte & Neumann《Dynamic Programming Strikes Back》(CIDR 2021)** | **DPhyp 算法** —— hypergraph 优化器 `EnumerateAllConnectedPartitions` 直接实现此论文，支持 bushy tree |
| **Galindo-Legaria《Parameterized Queries and Nesting Equivalences》(2001)** | 子查询去关联化、semi-join 转换的理论基础 |

**设计思想主线**：MySQL 8.0 近年的重构（8.0.22 AccessPath、8.0.31 Query_term、迭代器化）本质上是**一次"去 JOIN 化"**——把过去揉在 `JOIN` 结构里的逻辑查询树、物理计划、执行状态拆开，让优化器可替换、执行器只认 AccessPath/Iterator。

---

## 与其他文档的关系

```
                    客户端
                      │ 协议包
┌─────────────────────▼──────────────────────────────┐
│ 【server/query/】  本目录：SQL 文本 → 结果          │
│                                                     │
│  01 协议+分发 → 02 词法 → 03 语法记法 → 04 parser   │
│      → 05 contextualize → 06 prepare → 07_optimize/ │
│      → 08 AccessPath → 09 迭代器 ─┬─ (SELECT)       │
│                   └─ 10 DML                         │
│                                                     │
│  横切： runtime/（执行期专题） 11（EXPLAIN/trace）  │
└─────────────────────┬──────────────────────────────┘
                      │ ha_rnd_next / ha_index_next / ha_rnd_pos
┌─────────────────────▼──────────────────────────────┐
│ server/handler.md   handler / handlerton 分层        │
│                     行定位 position/ref/rnd_pos      │  ← server 与引擎的分界面
└─────────────────────┬──────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────┐
│ innodb/row_search.md  row_search_mvcc、游标推进      │
│ innodb/mvcc.md        可见性判断（read view）        │
│ innodb/*.md           buffer pool / redo / undo ...  │
└────────────────────────────────────────────────────┘
```

**边界约定**（按"第一主语"归属）：

| 内容 | 归属 | 理由 |
|---|---|---|
| 迭代器怎么调 handler | `query/09` | 主语是 SQL 层执行器 |
| `position`/`ref`/`rnd_pos` 语义 | `handler.md` | 主语是存储引擎接口 |
| `row_search_mvcc`、direction、游标推进 | `innodb/row_search.md` | 主语是 InnoDB |
| DML 的两阶段读 | `query/10_dml.md` | 主语是 SQL 层 DML |

- `07_optimize/` 讲"**优化器怎么产出 AccessPath**"（04 与 06 之间的那一大段）
- `runtime/` 讲"**执行期具体能力怎么实现**"（09 篇框架的横向展开）
- `11` 是"**观察主链的工具**"，横跨优化器与执行器，不占主链步骤

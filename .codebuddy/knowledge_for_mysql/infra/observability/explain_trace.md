# EXPLAIN 与 optimizer trace：看懂执行计划

> 本篇覆盖：如何把 AccessPath 树打印成人类可读的计划（EXPLAIN），以及 optimizer trace 如何记录优化过程。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、EXPLAIN 的三种输出路径](#一explain-的三种输出路径)
- [二、传统表格格式](#二传统表格格式)
- [三、TREE / JSON：基于 AccessPath](#三tree--json基于-accesspath)
- [四、EXPLAIN ANALYZE](#四explain-analyze)
- [五、optimizer trace](#五optimizer-trace)

---

## 设计思想与理论基础

### 计划呈现的两条路线：重放 vs 转译

EXPLAIN 把 AccessPath 树（08 篇）翻译成人类可读的计划，有两条路线，本篇的结构由此而来：

- **传统表格（TRADITIONAL）**：走 `join->qep_tab`（`QEP_TAB` 链），输出"一张表一行"的扁平表。这是**历史路线**，服务于旧优化器的 QEP 结构
- **TREE / JSON**：直接遍历 AccessPath 树，输出缩进树或 JSON。这是**新路线**，与 hypergraph / AccessPath 对齐

两条路线并存，是因为 `QEP_TAB` 与 AccessPath 两套结构在 8.0 共存（07 篇的"两条创建路径"）。EXPLAIN 必须**同时支持两种数据源**——这是"计划 IR 迁移期"的典型双轨现象。

### 可观测性的两个层次：计划 vs 实际

| 工具 | 回答 | 实现 |
|---|---|---|
| EXPLAIN | "优化器**打算**怎么做" | 静态遍历计划树 |
| EXPLAIN ANALYZE | "实际**真的**这么做了，耗时多少" | **真执行一遍**，`TimingIterator` 计时 |
| optimizer trace | "为什么**选**了这个计划" | 优化器边算边记 JSON 日志 |

三者合起来才构成完整的可观测性：**计划 → 实际 → 决策依据**。这比"只有 EXPLAIN"的数据库（如 5.x 早期）强得多——5.7 引入 optimizer trace、8.0 引入 ANALYZE，都是补"为什么"和"实际上如何"这两块拼图。

**★ 上表"实现"列的源码位置：唯一的 `explain_query()` 分叉点**（`opt_explain.cc:2219`）：

```cpp
bool explain_query(THD *explain_thd, const THD *query_thd, Query_expression *unit) {
  const bool other = (explain_thd != query_thd);
  const bool secondary_engine = SecondaryEngineHandlerton(query_thd) != nullptr;
  LEX *lex = explain_thd->lex;
  if (lex->explain_format->is_iterator_based()) {
    if (lex->is_explain_analyze) {
      if (secondary_engine) { my_error(ER_NOT_SUPPORTED_YET, ...); return true; }
      if (unit->root_iterator() == nullptr) {
        // TODO(sgunders): Remove when the iterator executor supports all queries.
        my_error(ER_NOT_SUPPORTED_YET, MYF(0), "EXPLAIN ANALYZE on this query");
        unit->set_executed();
        return true;
      }
      // Run the query, but with the result suppressed.
      Query_result_null null_result;              /* ★★ 结果直接丢弃 */
      unit->set_query_result(&null_result);
      explain_thd->running_explain_analyze = true;
      unit->execute(explain_thd);                 /* ★★★ 真执行一遍 */
      explain_thd->running_explain_analyze = false;
      unit->set_executed();
      if (query_thd->is_error()) return true;
    }
    return ExplainIterator(explain_thd, query_thd, unit);
  }

  // Non-iterator-based formats are not supported with EXPLAIN ANALYZE.
  if (lex->is_explain_analyze)
    my_error(ER_NOT_SUPPORTED_YET, MYF(0),
             (lex->explain_format->is_hierarchical()
                  ? "EXPLAIN ANALYZE with JSON format"
                  : "EXPLAIN ANALYZE with TRADITIONAL format"));
```

> ★★★ **`is_iterator_based()` 是"计划 vs 实际"的真正分水岭**。只有 iterator-based 格式（TREE、hypergraph 下的 JSON）才支持 ANALYZE。传统表格与线性 JSON 直接报 `ER_NOT_SUPPORTED_YET`。

**★★ 两个层次在同一个函数里体现得最清楚：`AddPathCosts()`**（`join_optimizer/explain_access_path.cc:796`）：

```cpp
  /* 层次一：计划（永远输出） */
  error |= AddMemberToObject<Json_double>(obj, "estimated_first_row_cost", first_row_cost);
  error |= AddMemberToObject<Json_double>(obj, "estimated_total_cost", cost);
  error |= AddMemberToObject<Json_double>(obj, "estimated_rows", path->num_output_rows());

  /* ★★ 层次二：实际（仅当 explain_analyze） */
  if (explain_analyze) {
    int num_init_calls = 0;
    if (path->iterator != nullptr) {
      const IteratorProfiler *const profiler = path->iterator->GetProfiler();
      if ((num_init_calls = profiler->GetNumInitCalls()) != 0) {
        AddMemberToObject<Json_double>(obj, "actual_first_row_ms", ...);
        AddMemberToObject<Json_double>(obj, "actual_last_row_ms", ...);
        AddMemberToObject<Json_double>(obj, "actual_rows", ...);
        AddMemberToObject<Json_int>(obj, "actual_loops", num_init_calls);
      }
    }
    if (num_init_calls == 0) {   /* ★ 没执行过 → 全部 JSON null */
      AddMemberToObject<Json_null>(obj, "actual_first_row_ms");
      AddMemberToObject<Json_null>(obj, "actual_last_row_ms");
      AddMemberToObject<Json_null>(obj, "actual_rows");
      AddMemberToObject<Json_null>(obj, "actual_loops");
    }
  }
```

> ★★ `estimated_*` 来自优化器代价模型，**永远输出**；`actual_*` 来自 `IteratorProfiler`，**仅在 ANALYZE 下**；没执行过的节点是 JSON `null`。

**★ "实际"数据的采集器是 `TimingIterator`**——它**不是**在计划里插入一个节点，而是**包装**每个迭代器：几乎所有迭代器创建都走 `NewIterator()`，ANALYZE 时自动套一层计时壳（`iterators/timing_iterator.h:214`，详见 [4.2](#42-timingiterator-计时)）。

### trace 的代价与取舍

optimizer trace 的机制是**在优化器关键路径埋点**（`Opt_trace_object`），默认**关闭**——因为它要记录大量中间量（每个候选计划的代价、每个表的估算行数），开着的开销和日志体积都不小。这是典型的"**可观测性按需开启**"设计，与 `optimizer_switch` 里那些默认关闭的 debug 开关同理。

---

## 先有个整体印象

优化器产出 AccessPath 树（08 篇）后，EXPLAIN 要把它翻译成三种形态：

```
EXPLAIN 输出
 ├─ FORMAT=TRADITIONAL（默认）  遍历 QEP_TAB，一张表一行
 ├─ FORMAT=TREE                遍历 AccessPath 树，缩进树形
 └─ FORMAT=JSON                遍历 AccessPath 树，JSON
```

**EXPLAIN ANALYZE** 还会**真正执行一遍**查询，用 `TimingIterator` 收集每步的实际时间与行数。

> ⚠️ 文件位置：本版本的 explain 代码在 **`sql/opt_explain.cc`**（不是 `explain.cc`）。

### EXPLAIN 的 SQL 变体（有哪些）

| SQL | 说明 |
|-----|------|
| `EXPLAIN SELECT ...` | 默认 TRADITIONAL 表格 |
| `EXPLAIN FORMAT=TREE SELECT ...` | 树形输出 |
| `EXPLAIN FORMAT=JSON SELECT ...` | JSON 输出 |
| `EXPLAIN ANALYZE SELECT ...` | 真执行一遍 + 实际耗时/行数（8.0.18+） |
| `EXPLAIN ANALYZE FORMAT=TREE SELECT ...` | 树形 + actual time |
| `EXPLAIN FOR CONNECTION <id>` | 看其他连接的 plan（已基本废弃，走 `SQLCOM_EXPLAIN_OTHER`） |
| `DESC` / `DESCRIBE` | EXPLAIN 的同义词（`DESC SELECT ...`） |
| `EXPLAIN <table>` | 等价 `DESCRIBE <table>`，看**表结构**（不是执行计划） |

**★ 语法原文**（`sql/sql_yacc.yy:13995`）：

```cpp
/* A Oracle compatible synonym for show */
describe_stmt:
          describe_command table_ident opt_describe_column
          { $$= NEW_PTN PT_show_fields(@$, Show_cmd_type::STANDARD, $2, $3); }
        ;

explain_stmt:
          describe_command opt_explain_options explainable_stmt
          { $$= NEW_PTN PT_explain($2.explain_format_type, $2.is_analyze,
                                   $2.is_explicit, $3); }
        ;

explainable_stmt:
          select_stmt
        | insert_stmt
        | replace_stmt
        | update_stmt
        | delete_stmt
        | FOR_SYM CONNECTION_SYM real_ulong_num
          { $$= NEW_PTN PT_explain_for_connection(static_cast<my_thread_id>($3)); }
        ;

describe_command: DESC | DESCRIBE ;

opt_explain_format:
          %empty
          { $$.is_explicit = false;
            $$.explain_format_type = YYTHD->variables.explain_format; }
        | FORMAT_SYM EQ ident_or_text
          { $$.is_explicit = true;
            if (is_identifier($3, "JSON")) $$.explain_format_type = Explain_format_type::JSON;
            else if (is_identifier($3, "TRADITIONAL")) ... TRADITIONAL;
            else if (is_identifier($3, "TREE")) ... TREE;
            else {
              // This includes even TRADITIONAL_STRICT. Since this value is
              // only meant for mtr infrastructure temporarily, we don't want
              // the user to explicitly use this value in EXPLAIN statements.
              my_error(ER_UNKNOWN_EXPLAIN_FORMAT, MYF(0), $3.str);
              MYSQL_YYABORT;
            } }

opt_explain_options:
          ANALYZE_SYM opt_explain_format  { $$ = $2; $$.is_analyze = true; }
        | opt_explain_format              { $$ = $1; $$.is_analyze = false; }
        ;
```

> ⚠️ 注意 `describe_command` 只含 `DESC` 与 `DESCRIBE`——**`EXPLAIN` 是单独的关键字**（走同一条 `explain_stmt` 产生式）。

**★★ 三处需要纠正的"想当然"**：

| 想当然 | 真相 |
|---|---|
| 存在 `SQLCOM_EXPLAIN` | **不存在**。只有 **`SQLCOM_EXPLAIN_OTHER`**（仅供 `EXPLAIN FOR CONNECTION`）。普通 `EXPLAIN <stmt>` **沿用被解释语句自身的 sql_command**（`SQLCOM_SELECT` 等），靠 `LEX::explain_format != nullptr` 判定 —— 即 `bool is_explain() const { return explain_format != nullptr; }`（`sql_lex.h:3904`） |
| `EXPLAIN INTO <var>` 存在 | **不存在**。`opt_explain_options` 只有 `ANALYZE` 与 `FORMAT=` 两种前缀 |
| 存在 `EXPLAIN EXTENDED` / `EXPLAIN PARTITIONS` | **不存在**（5.x 遗留，8.0 已删）。分区信息并入 `partitions` 列 |

**★ 可被 EXPLAIN 的只有 9 个命令**（`sql_parse.cc:1173`，靠 `CF_CAN_BE_EXPLAINED` 标志）：

```cpp
bool is_explainable_query(enum enum_sql_command command) {
  return (sql_command_flags[command] & CF_CAN_BE_EXPLAINED) != 0;
}
```

即 `SELECT` / `INSERT` / `INSERT_SELECT` / `REPLACE` / `REPLACE_SELECT` / `UPDATE` / `UPDATE_MULTI` / `DELETE` / `DELETE_MULTI`。

> **注意**：`EXPLAIN <table>` 和 `EXPLAIN SELECT ...` 语义完全不同——前者是 `SHOW COLUMNS` 的别名（走 `describe_stmt` → `PT_show_fields`），后者才产生执行计划。别混。

---

## 一、EXPLAIN 的三种输出路径

`explain_query()` 先判断格式（`opt_explain.cc:2226-2256`）：`is_iterator_based()` 为真（TREE/JSON）走 iterator 路径（`ExplainIterator` `:2090`），否则走传统路径（`Explain_join` `:2070`）。

| 格式 | 遍历对象 | 路径 |
|------|----------|------|
| TRADITIONAL | `join->qep_tab`（QEP_TAB） | `Explain_join::shallow_explain`（`opt_explain.cc:1250`） |
| TREE | AccessPath | `PrintQueryPlan` → `ExplainAccessPath` |
| JSON | AccessPath | 同上，输出 JSON |

**★ `explain_format` 的唯一赋值点：`PT_explain::make_cmd()`**（`parse_tree_nodes.cc:3506`，★ 五处）：

```cpp
  switch (m_format) {
    case Explain_format_type::TRADITIONAL:
    case Explain_format_type::TRADITIONAL_STRICT:
      /*
        With no format specified:
        - With ANALYZE, convert TRADITIONAL[_STRICT] to TREE unconditionally.
        - With hypergraph, convert TRADITIONAL to TREE. Don't convert
          TRADITIONAL_STRICT, because it's purpose is to prevent exactly this
          silent conversion with hypergraph. TRADITIONAL_STRICT will throw an
          error, later.
      */
      if (!m_explicit_format &&
          (m_analyze ||
           (thd->optimizer_switch_flag(OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER) &&
            m_format == Explain_format_type::TRADITIONAL))) {
        lex->explain_format = new (thd->mem_root) Explain_format_tree;
      } else {
        lex->explain_format = new (thd->mem_root) Explain_format_traditional;
      }
      break;
    case Explain_format_type::JSON: {
      lex->explain_format = new (thd->mem_root) Explain_format_JSON(
          thd->optimizer_switch_flag(OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER)
              ? Explain_format_JSON::FormatVersion::kIteratorBased
              : Explain_format_JSON::FormatVersion::kLinear);
      break;
    }
    case Explain_format_type::TREE:
      lex->explain_format = new (thd->mem_root) Explain_format_tree;
      break;
  }
  lex->is_explain_analyze = m_analyze;
  Sql_cmd *ret = m_explainable_stmt->make_cmd(thd);   /* ★ 沿用底层语句的 cmd */
```

> ★★ 三个"静默转换"都在这里：**①`EXPLAIN ANALYZE` 不写 FORMAT 时被强制转成 TREE**；② hypergraph 下 TRADITIONAL 被静默转成 TREE；③ hypergraph 下 JSON 用 `kIteratorBased` 版本。
> ★ `TRADITIONAL_STRICT` **只能由系统变量设置**，显式写进 SQL 会报 `ER_UNKNOWN_EXPLAIN_FORMAT`（注释明说它是给 MTR 用的）。

**★ 基类 `Explain_format`**（`opt_explain_format.h:500`）——两个纯虚方法定下两条路线：

```cpp
class Explain_format {
 protected:
  Query_result *output;
 public:
  virtual bool is_hierarchical() const = 0;      /* ★ 树形 vs 扁平表格 */

  /**
    Whether the format closely resembles the final plan to be executed by
    execution iterators ... These formats share a common logic that uses
    AccessPath structure to generate the information, so they all display
    exactly the same information, even though the style of each format
    might be different.
  */
  virtual bool is_iterator_based() const { return false; }   /* ★★ 计划 vs 实际的分水岭 */

  virtual bool send_headers(Query_result *result) { output = result; return false; }
  virtual bool begin_context(enum_parsing_context context,
                             Query_expression *subquery = nullptr,
                             const Explain_format_flags *flags = nullptr) = 0;
  virtual bool end_context(enum_parsing_context context) = 0;
  virtual bool flush_entry() = 0;                /* 传统：输出一行 */
  virtual qep_row *entry() = 0;
};
```

**★ 三个实现类的对比**：

| 类 | 文件 | `is_hierarchical` | `is_iterator_based` | 数据容器 |
|---|---|---|---|---|
| `Explain_format_traditional` | `opt_explain_traditional.h:44` | `false` | `false` | `qep_row column_buffer` |
| `Explain_format_tree` | **同上 `:69`**（⚠️ 不在 `join_optimizer/`） | `false` | **`true`** | 先生成 JSON 再渲染 |
| `Explain_format_JSON` | `opt_explain_json.h:37`（⚠️ 大写 JSON） | **`true`** | 取决于 `m_version` | JSON 树 |

```cpp
class Explain_format_traditional : public Explain_format {
  class Item_null *nil;
  qep_row column_buffer;  ///< buffer for the current output row
 public:
  bool is_hierarchical() const override { return false; }
  bool begin_context(...) override { return false; }   /* ★ 空实现：扁平，无上下文 */
  bool end_context(...) override { return false; }
  bool flush_entry() override;
  qep_row *entry() override { return &column_buffer; }
};
```

> ★★ **TREE 的实现最反直觉**：它**不经过 `qep_row` 体系**——`begin_context`/`end_context`/`flush_entry`/`entry` 全是 `assert(false)`。它先生成完整 JSON，再由 `ExplainJsonToString()` / `ExplainPrintTreeNode()` 渲染成缩进文本。**这就是"TREE 与 hypergraph-JSON 显示完全相同信息"的原因**（基类注释原话）。
> ★ JSON / TREE 都只有**一列**，列名就叫 `EXPLAIN`，长度 78（`opt_explain_json.cc:2082`）。

**★ 格式类型枚举**（⚠️ **不在 `opt_explain_format.h`**，在 `sql/system_variables.h:121`）：

```cpp
enum class Explain_format_type : ulong {
  TRADITIONAL = 0,
  TRADITIONAL_STRICT = 1,
  TREE = 2,
  JSON = 3
};
```

---

## 二、传统表格格式

`Explain_join::shallow_explain()`（`opt_explain.cc:1250`）遍历 **QEP_TAB**，不是 AccessPath：

```cpp
for (size_t t = 0, cnt = fmt->is_hierarchical() ? join->primary_tables
                                                : join->tables;
     t < cnt; t++) {
  if (explain_qep_tab(t)) return true;
}
```

`explain_qep_tab()`（`:1375`）从 `QEP_TAB` 取 `type`/`condition`/`range_scan`/`keys` 等，逐行输出。

> 这就是为什么传统 EXPLAIN 只能表达**左深树** —— 它本质是 QEP_TAB 数组，每张表一行。

### 2.1 传统表格的 12 列含义

| 列 | 含义 | 打印位置 |
|----|------|---------|
| `id` | 查询块编号（对应 `select#N`） | `explain_qep_tab` |
| `select_type` | 查询类型（`SIMPLE`/`PRIMARY`/`SUBQUERY`/`DERIVED`/`UNION`...） | `Explain_select` |
| `table` | 表名/别名（或 `<derivedN>`/`<unionN>`） | `col_table_name` |
| `type` | 访问类型（`join_type_str[]`） | `col_join_type` |
| `possible_keys` | 候选索引 | `explain_possible_keys` |
| `key` | 实际选的索引 | `explain_key_and_len` |
| `key_len` | 索引使用字节数 | `explain_key_and_len` |
| **`ref`** | **用哪个列/常量匹配索引** | `explain_ref`（见 2.2） |
| `rows` | 估算扫描行数 | `col_rows` |
| `filtered` | WHERE 过滤后剩余比例 | `col_filtered` |
| `Extra` | 附加信息（`Using where`/`Using index`/`Using temporary`...） | `col_extra` |

**★ 12 列的承载结构 `qep_row`**（`opt_explain_format.h:282`）：

```cpp
  column<uint> col_id;                      ///< "id": seq. number of SELECT
  column<enum_explain_type> col_select_type;///< "select_type"
  mem_root_str col_table_name;              ///< "table"
  List<const char> col_partitions;          ///< "partitions"
  mem_root_str col_join_type;               ///< "type", see join_type_str array
  List<const char> col_possible_keys;       ///< "possible_keys"
  mem_root_str col_key;                     ///< "key"
  mem_root_str col_key_len;                 ///< "key_len"
  List<const char> col_ref;                 ///< "ref"
  column<float> col_filtered;               ///< "filtered": % of rows filtered
  List<extra> col_extra;                    ///< "extra"
  // non-TRADITIONAL stuff:
  mem_root_str col_message;                 ///< replaces "Extra" if not empty
  mem_root_str col_attached_condition;      ///< former "Using where"
  column<ulonglong> col_rows;               ///< "rows"
```

> ★ 两类容器：`column<T>`（数值，内部有 `nil` 标志表示 NULL）与 `mem_root_str`（字符串，在 MEM_ROOT 上分配）。`is_empty()` 为真就输出 `Item_null`。

**★ 表头定义：`THD::send_explain_fields()`**（`sql_class.cc:1889`，⚠️ **不存在 `explain_columns()` 函数**）：

```cpp
int THD::send_explain_fields(Query_result *result) {
  field_list.push_back(new Item_return_int("id", 3, MYSQL_TYPE_LONGLONG));
  field_list.push_back(new Item_empty_string("select_type", 19, cs));
  item = new Item_empty_string("table", NAME_CHAR_LEN, cs);
  item->set_nullable(true);
  /* Maximum length of string that make_used_partitions_str() can produce */
  item = new Item_empty_string("partitions", MAX_PARTITIONS * (1 + FN_LEN), cs);
  item->set_nullable(true);
  item = new Item_empty_string("type", 10, cs);
  item->set_nullable(true);
  item = new Item_empty_string("possible_keys", NAME_CHAR_LEN * MAX_KEY, cs);
  item->set_nullable(true);
  item = new Item_empty_string("key", NAME_CHAR_LEN, cs);
  item->set_nullable(true);
  item = new Item_empty_string("key_len", NAME_CHAR_LEN * MAX_KEY);
  item->set_nullable(true);
  item = new Item_empty_string("ref", NAME_CHAR_LEN * MAX_REF_PARTS, cs);
  item->set_nullable(true);
  item = new Item_return_int("rows", 10, MYSQL_TYPE_LONGLONG);
  item->set_nullable(true);
  item = new Item_float(NAME_STRING("filtered"), 0.1234, 2, 4);
  item->set_nullable(true);
  field_list.push_back(new Item_empty_string("Extra", 255, cs));
  item->set_nullable(true);
  ...
}
```

**★★ 每列的赋值点**（`opt_explain.cc`）：

| 列 | 赋值函数 | 行号 | 数据来自 |
|---|---|---|---|
| `id` | `Explain::explain_id()` | 774 | `query_block->select_number` |
| `select_type` | `Explain::explain_select_type()` | 779 | `query_block->type()`（→ `Query_block::type_str[]`，`sql_lex.cc:140`） |
| `table` | `Explain_table::explain_table_name()` | 1757 | `table->alias` |
| `partitions` | `Explain_table_base::explain_partitions()` | 931 | `make_used_partitions_str(table->part_info, ...)` |
| `type` | `Explain_join::explain_join_type()` | 1495 | `join_type_str[j_t]` |
| `possible_keys` | `explain_possible_keys()` | 938 | 遍历 `usable_keys` 位图 |
| `key`+`key_len` | `explain_key_and_len()` | 1514 / 973 | `tab->ref().key` 或 `range_scan_path` |
| `ref` | `explain_ref_key()` | 617 | `store_key::name()`（见 2.2） |
| `rows`+`filtered` | `explain_rows_and_filtered()` | 1532 | `POSITION::rows_fetched` / `filter_effect` |
| `Extra` | `flush_entry()` | trad. 218 | `col_extra` 列表查表拼接 |

```cpp
bool Explain::explain_id() {
  fmt->entry()->col_id.set(query_block->select_number);
  return false;
}

bool Explain_table::explain_table_name() {
  return fmt->entry()->col_table_name.set(table->alias);   /* ★ 别名，不是真表名 */
}

bool Explain_table_base::explain_possible_keys() {
  if (usable_keys.is_clear_all()) return false;
  if ((table->file->ha_table_flags() & HA_NO_INDEX_ACCESS) != 0) return false;
  for (uint j = 0; j < table->s->keys; j++) {
    if (usable_keys.is_set(j) &&
        fmt->entry()->col_possible_keys.push_back(table->key_info[j].name))
      return true;
  }
  return false;
}
```

**★ `type` 列的一个历史遗留**——`IN (subquery)` 会改写显示（`opt_explain.cc:1495`）：

```cpp
  if ((j_t == JT_EQ_REF || j_t == JT_REF || j_t == JT_REF_OR_NULL) &&
      join->query_expression()->item) {
    /*
      For backward-compatibility, we have special presentation of "index
      lookup used for in(subquery)": we do not show "ref/etc", but
      "index_subquery/unique_subquery".
    */
    if (join->query_expression()->item->engine_type() ==
        Item_subselect::INDEXSUBQUERY_ENGINE)
      str = (j_t == JT_EQ_REF) ? "unique_subquery" : "index_subquery";
  }
```

> ⚠️ 所以 `type` 列**可能出现 `index_subquery` / `unique_subquery`**，而这两个**不在 `join_type_str[]` 数组里**——是上面这段硬编码替换的结果。

**`type` 列的取值**（`join_type_str[]`，`opt_explain.cc:115`）：

```cpp
const char *join_type_str[] = {
    "UNKNOWN", "system", "const",    "eq_ref",      "ref",        "ALL",
    "range",   "index",  "fulltext", "ref_or_null", "index_merge"};
```

`type` 列 = **这张表的访问方法**（access method）。它的"怎么选出来"是 07 篇《访问方法选择》的主题（const 检测 / Key_use / EQ_REF 判定 / 代价决斗），这里只讲**每个取值意味着什么、怎么读**。从好到坏：

| type | 含义 | 触发条件（简化） | 迭代器 |
|------|------|----------------|--------|
| `system` | 表只有 0 或 1 行（系统表 / 统计精确 ≤1） | `JT_SYSTEM`（`mark_const_table` 传 `key==nullptr`） | `ConstIterator` |
| `const` | 唯一键等值定位，**最多 1 行，优化期就读出** | `JT_CONST`（`mark_const_table` 传 `key!=nullptr`） | `ConstIterator` |
| `eq_ref` | 每行外层值经**唯一键**精确匹配内层**恰好 1 行** | `JT_EQ_REF` | `EQRefIterator` |
| `ref` | **非唯一键**等值匹配（一次可能匹配多行） | `JT_REF` | `RefIterator` |
| `ref_or_null` | ref + 额外查一次 NULL（`col=x OR col IS NULL`） | `JT_REF_OR_NULL` | `RefOrNullIterator` |
| `range` | 用索引查一个**范围**（`>`/`<`/`BETWEEN`/`IN`） | `JT_RANGE` | 各 range 迭代器 |
| `index` | **全索引扫描**（扫索引而非表，通常覆盖索引避免回表） | `JT_INDEX_SCAN` | `IndexScanIterator` |
| `ALL` | **全表扫描** | `JT_ALL` | `TableScanIterator` |
| `fulltext` | 全文索引 | `JT_FT` | — |
| `index_merge` | 多个索引**合并**（intersection / union / sort_union） | `JT_INDEX_MERGE` | `IndexMergeIterator` |

**`eq_ref` vs `ref` 的精确判定**（`create_ref_for_key`，`sql_select.cc:2512-2536`）——这是最容易混的一对：

```cpp
if ((actual_key_flags(keyinfo) & HA_NOSAME) == 0 ||            // ① 非唯一索引
    ((actual_key_flags(keyinfo) & HA_NULL_PART_KEY) &&
     !null_rejecting_key) ||                                   // ② 唯一但可空，且谓词不拒 NULL
    keyparts != actual_key_parts(keyinfo))                     // ③ 唯一但没用满 keypart
  set_type(JT_REF);                                            // ⇒ ref
else
  set_type(JT_EQ_REF);                                         // ⇒ eq_ref
```

三个退化成 `ref` 的情形：

1. **① 非唯一索引**：一个 key 值可能匹配多行 → 不是"恰好 1 行"，只能是 `ref`。
2. **② 唯一索引但含可空 keypart，且谓词不拒绝 NULL**：MySQL 唯一索引允许**多个 NULL**（`UNIQUE KEY (a)` 里 `a` 可以为 NULL 多行），所以一个 key 值可能匹配多行 → 退化成 `ref`。
3. **③ 唯一索引但没用满 keypart**：复合唯一键 `(a,b)` 只用 `a` 做等值，`a` 相同可能有多个 `b` → 不是唯一定位 → `ref`。

**`const` 的坑**（07 篇 2.1 核实过）：`system` 只在"表定义上就 ≤1 行"时出现；InnoDB 永久表**永远不会**走 `system`（`HA_STATS_RECORDS_IS_EXACT` 只有临时表申报），同样场景 InnoDB 显示 `const`（走唯一键路径）。

> 完整的选择机制（Key_use 数组怎么建、EQ_REF 的 fanout 为什么是 1.0、ref/range/scan 怎么代价决斗、二次 range 重估）见 [07 篇](../../server/query/07_optimize/physical/07_access_method.md)。

### 2.2 ref 列：怎么来的、代码哪里打印、代表什么

这是最容易看不懂的一列。**ref 列回答："用哪个值去匹配索引的每一部分"**。

打印链（`opt_explain.cc`）：

```cpp
// Explain_join::explain_ref（:1527）
bool Explain_join::explain_ref() {
  if (!tab) return false;
  return explain_ref_key(fmt, tab->ref().key_parts, tab->ref().key_copy);
}

// explain_ref_key（:617）
static bool explain_ref_key(Explain_format *fmt, uint key_parts,
                            store_key *key_copy[]) {
  if (key_parts == 0) return false;
  for (uint part_no = 0; part_no < key_parts; part_no++) {
    const store_key *const s_key = key_copy[part_no];
    if (s_key == nullptr) {
      // 常量键不需要拷贝 → 打印 "const"
      fmt->entry()->col_ref.push_back(STORE_KEY_CONST_NAME);
    } else {
      fmt->entry()->col_ref.push_back(s_key->name());   // 列名 或 "func"
    }
  }
  return false;
}
```

每个键部分（key part）对应一个 `store_key`，它的 `name()` 决定打印什么（`sql_select.h:817`）：

```cpp
virtual const char *name() const {
  if (item->type() == Item::FIELD_ITEM) {
    return item->full_name();   // 列引用 → "db.t1.a"
  } else {
    return "func";              // 表达式/函数 → "func"
  }
}
```

所以 ref 列的三种取值：

| ref 值 | 含义 | 触发场景 |
|--------|------|---------|
| `const` | 用常量匹配索引 | `WHERE a = 5`（`s_key == nullptr`，常量不需拷贝） |
| `db.t1.a` | 用另一张表的列匹配 | `t1 JOIN t2 ON t2.a = t1.a`（t2 的索引 a 用 t1.a 匹配） |
| `func` | 用函数/表达式结果匹配 | `WHERE a = func(b)` |
| `NULL` | 没有按值查找 | `type = ALL/index/range` 时（非 ref 类访问） |

**关键理解**：`ref` 列是 `key` 列的"补充说明"——`key` 告诉你"用哪个索引"，`ref` 告诉你"索引每一部分去匹配什么值"。它只对按值查找的访问类型（`const`/`eq_ref`/`ref`/`ref_or_null`）有意义；`type = ALL`（全表扫）时 ref 为 NULL。

**数据来源**：`tab->ref()` 是 `QEP_TAB` 的 `TABLE_REF` 结构，它的 `key_parts`/`key_copy[]` 由优化器在选好访问方法后填入（`pick_table_access_method` 阶段），每个 key part 一个 `store_key`，记录"用哪个 Item 的值构造这个 key part"。

### 2.3 filtered 列：WHERE 过滤后剩余比例（计算逻辑）

`filtered` 回答："这张表读出来的行，经过 WHERE/ON 条件过滤后还剩百分之几"。

打印代码（`explain_rows_and_filtered`，`opt_explain.cc:1532-1549`）：

**完整原文**（`opt_explain.cc:1532`，★ 含 `EXPLAIN FOR CONNECTION` 的特殊分支）：

```cpp
bool Explain_join::explain_rows_and_filtered() {
  if (!tab || tab->table_ref->schema_table) return false;

  POSITION *const pos = tab->position();

  if (explain_thd->lex->sql_command == SQLCOM_EXPLAIN_OTHER &&
      skip_records_in_range) {
    // Skipping col_rows, col_filtered, col_prefix_rows will set them to NULL.
    fmt->entry()->col_cond_cost.set(0);
    fmt->entry()->col_read_cost.set(0.0);
    fmt->entry()->col_prefix_cost.set(0);
    fmt->entry()->col_data_size_query.set("0");
  } else {
    fmt->entry()->col_rows.set(static_cast<ulonglong>(pos->rows_fetched));
    fmt->entry()->col_filtered.set(
        pos->rows_fetched
            ? static_cast<float>(100.0 * tab->position()->filter_effect)
            : 0.0f);
    ...
  }
```

> ★★ 注意分支条件：`EXPLAIN FOR CONNECTION`（`SQLCOM_EXPLAIN_OTHER`）**且** `skip_records_in_range`（强制跳过 index dive）时，`rows`/`filtered`/各项 cost **全部留 NULL**（不 set 即 NULL），只有 `col_data_size_query` 被设成 `"0"`。这是"看别人的连接时为了避免干扰而不做 index dive"的副作用。
> ★ 所以 **`rows` 是 `POSITION::rows_fetched`（预计扫描行数，不是返回行数）**——这正是 `rows × filtered%` 才有意义的理由。

所以 **`filtered = 100.0 × filter_effect`**。`filter_effect` 是 `POSITION` 上的字段，即"这张表上条件的**过滤系数**"（剩余比例），来源在访问方法选择阶段（07 篇 `best_access_path` / `calculate_scan_cost`）：

- 条件过滤开启（默认 `condition_fanout_filter=on`）→ `calculate_condition_filter` 用常量条件算过滤系数
- 否则 → 各种启发式（如无可用估计时 `found_records * 0.75` 的 25% 过滤假设）

两个注意点：

1. **`filtered` 只有传统 EXPLAIN 有**，且 `EXPLAIN CONNECTION`（`SQLCOM_EXPLAIN_OTHER`）会跳过（设成 NULL）。
2. **`rows × filtered%` 才是这张表真正"传给下一张表"的行数**。多表 join 时，前一张表的 `rows × filtered%` 会作为下一张表 `rows` 的基数来源——所以看 EXPLAIN 时 `rows` 是"扫描行数"，`rows × filtered%` 才是"有效行数"。

### 2.4 Extra 列：附加信息（30+ 种取值）

`Extra` 是"这一行表还有什么额外动作"。它由 `explain_extra`（`opt_explain.cc:1571`）+ `explain_extra_common`（`:992`）逐项 `push_extra(ET_XXX)` 拼出来，取值枚举在 `Extra_tag`（`opt_explain_format.h:60-98`），字符串映射在 `traditional_extra_tags[]`（`opt_explain_traditional.cc:48-84`）。**一个表可以同时挂多个 Extra**（用 `; ` 连接）。

**最核心的几个（必须懂）**：

| Extra | 含义 | 代码 |
|-------|------|------|
| `Using where` | **server 层还要再过滤**。不是"用了 WHERE"，而是"WHERE 里有存储引擎处理不了、返回行后 server 层还要逐行判断的部分"（如非 sargable 谓词、或 ON 转成 WHERE 的残留） | `explain_extra_common :1068`（`condition` 非空） |
| `Using index` | **覆盖索引**：只读索引不回表（`covering_keys.is_set`） | `explain_extra :1602` |
| `Using index condition` | **ICP**（Index Condition Pushdown）：索引条件推给存储引擎在索引层先过滤 | `explain_extra_common :993`（`pushed_idx_cond`） |
| `Using temporary` | 要建**临时表**（GROUP BY/DISTINCT 无法用索引、UNION、子查询等） | `explain_tmptable_and_filesort` |
| `Using filesort` | 要**排序**（ORDER BY 无法用索引满足） | `explain_tmptable_and_filesort` |
| `Using join buffer` | 用 **join buffer**（BNL/hash join/Batched Key Access） | `explain_extra :1664`（`OT_BNL`/`OT_BKA`） |

**`Using where` 和 `Using index condition` 的区别**（最容易混）：

- `Using index condition`（ICP）：条件**下推到存储引擎**，在**索引扫描时**就过滤（如 `WHERE a>5 AND b='x'`，索引 `(a)` 扫 `a>5` 后，`b='x'` 交给引擎在回表前判）。
- `Using where`：条件**没下推**，引擎返回行后 **server 层**再 `Item::val_*` 逐行判断（如条件里有函数、或索引没法承载的部分）。
- 两者可以**同时出现**：`Using where; Using index condition` 表示一部分下推、一部分留在 server 层。

**semi-join / anti-join 相关的（03 篇策略的 Extra 表现）**：

| Extra | 含义 | 对应策略 |
|-------|------|---------|
| `FirstMatch` | 首匹配去重 | FirstMatch 策略（`:1629`） |
| `LooseScan` | 松索引扫描去重 | LooseScan 策略（`:1621`） |
| `Start temporary` / `End temporary` | 临时表去重的起止边界 | DuplicateWeedout 策略（`:1623`/`:1627`） |
| `Not exists` | 外连接 + NOT EXISTS 优化（`LEFT JOIN ... WHERE right IS NULL` 只取无匹配行） | — |
| `Using index for group-by` | 用**松索引扫描**直接做 GROUP BY（不排序） | `explain_extra :1603`（GROUP_INDEX_SKIP_SCAN） |
| `Using index for skip scan` | **index skip scan** | `explain_extra :1608` |

**其余常见取值速查**：

| Extra | 含义 |
|-------|------|
| `Using MRR` | Multi-Range Read：多范围读取按 rowid 排序批量回表 |
| `Distinct` | DISTINCT 去重（`:1618`） |
| `Range checked for each record` | 每行动态决定用哪个索引（dynamic range，`:1062`） |
| `Using pushed condition` | NDB 引擎的条件下推（`:1078`） |
| `Full scan on NULL key` | `ref_or_null` 的 NULL 分支走全扫（`:1661`） |
| `Rematerialize` | lateral derived 表被重复物化（`:1644`） |
| `Impossible ON condition` | const 表的 ON 恒假（`:1580`） |
| `const row not found` / `unique row not found` | const/唯一键定位时没找到行（`:1574`/`:1577`） |
| `Backward index scan` | 反向索引扫描（ORDER BY DESC 用索引倒序） |
| `Recursive` | 递归 CTE（`:91`） |
| `Using secondary engine` | 走了 secondary engine（`:1693`） |

**★★ 机制：Extra 不是字符串，是 `Extra_tag` 枚举 + 查表**

```cpp
/* opt_explain_format.h:54 */
enum Extra_tag {
  ET_none, ET_USING_TEMPORARY, ET_USING_FILESORT, ET_USING_INDEX_CONDITION,
  ET_USING, ET_RANGE_CHECKED_FOR_EACH_RECORD, ET_USING_PUSHED_CONDITION,
  ET_USING_WHERE, ET_NOT_EXISTS, ET_USING_MRR, ET_USING_INDEX,
  ET_FULL_SCAN_ON_NULL_KEY, ET_USING_INDEX_FOR_GROUP_BY,
  ET_USING_INDEX_FOR_SKIP_SCAN, ET_DISTINCT, ET_LOOSESCAN,
  ET_START_TEMPORARY, ET_END_TEMPORARY, ET_FIRST_MATCH, ET_MATERIALIZE,
  ET_START_MATERIALIZE, ET_END_MATERIALIZE, ET_SCAN, ET_USING_JOIN_BUFFER,
  ET_CONST_ROW_NOT_FOUND, ET_UNIQUE_ROW_NOT_FOUND, ET_IMPOSSIBLE_ON_CONDITION,
  ET_PUSHED_JOIN, ET_FT_HINTS, ET_BACKWARD_SCAN, ET_RECURSIVE,
  ET_TABLE_FUNCTION, ET_SKIP_RECORDS_IN_RANGE, ET_USING_SECONDARY_ENGINE,
  ET_REMATERIALIZE, ET_total
};
```

```cpp
/* opt_explain_traditional.cc:43 —— 字符串映射表 */
static const char *traditional_extra_tags[ET_total] = {
    nullptr,                            // ET_none
    "Using temporary",                  // ET_USING_TEMPORARY
    "Using filesort",                   // ET_USING_FILESORT
    "Using index condition",            // ET_USING_INDEX_CONDITION
    "Using",                            // ET_USING   ★ head，后面接 data
    "Range checked for each record",    // ET_RANGE_CHECKED_FOR_EACH_RECORD
    "Using pushed condition", "Using where", "Not exists", "Using MRR",
    "Using index", "Full scan on NULL key", "Using index for group-by",
    "Using index for skip scan", "Distinct", "LooseScan",
    "Start temporary", "End temporary", "FirstMatch",
    "Materialize", "Start materialize", "End materialize", "Scan",
    "Using join buffer", "const row not found", "unique row not found",
    "Impossible ON condition",
    "",                                 // ET_PUSHED_JOIN  ★ head 为空串
    "Ft_hints:", "Backward index scan", "Recursive", "Table function:",
    "Index dive skipped due to FORCE", "Using secondary engine", "Rematerialize"
};
```

> ★ `push_extra()` 只推**枚举 tag**，字符串在 `traditional_extra_tags[]`（表格）/ `json_extra_tags[]`（JSON 属性名）里**查表**得到。所以同一个 tag 在不同格式下是不同呈现。
> ★ **`ET_USING` 的 head 是 `"Using "`**——`Using union(...)` / `Using intersect(...)` 是它加上 `add_info_string()` 拼出的 data 组成的。

**★ `"Using where; Using index"` 里那个 `; ` 的来源**（`flush_entry()`，`opt_explain_traditional.cc:244`）：

```cpp
  } else if (!column_buffer.col_extra.is_empty()) {
    StringBuffer<64> buff(system_charset_info);
    List_iterator<qep_row::extra> it(column_buffer.col_extra);
    qep_row::extra *e;
    while ((e = it++)) {
      assert(traditional_extra_tags[e->tag] != nullptr);
      if (buff.append(traditional_extra_tags[e->tag])) return true;
      if (e->data) {
        bool brackets = false;
        switch (e->tag) {
          case ET_RANGE_CHECKED_FOR_EACH_RECORD:
          case ET_USING_INDEX_FOR_GROUP_BY:
          case ET_USING_JOIN_BUFFER:
          case ET_FIRST_MATCH:
          case ET_REMATERIALIZE:
            brackets = true;  // for backward compatibility
            break;
          default: break;
        }
        if (e->tag != ET_FIRST_MATCH && e->tag != ET_PUSHED_JOIN && buff.append(" "))
          return true;
        if (brackets && buff.append("(")) return true;
        if (buff.append(e->data)) return true;
        if (brackets && buff.append(")")) return true;
      }
      if (buff.append("; ")) return true;      /* ★★ 分号在这里 */
    }
    if (!buff.is_empty()) buff.length(buff.length() - 2);   /* 去掉末尾 "; " */
```

> ★ **括号只对 5 个 tag 自动加**（上面 switch 里那 5 个），其余 tag 的括号（如 `union(...)`）来自 data 字符串本身。
> ★ `col_message` 与 `col_extra` **互斥**——有 message 就不打印 extra。

**★★★ 四处必须纠正的"想当然"**：

| 想当然 | 真相 |
|---|---|
| `Impossible WHERE` 是 push_extra 写入的 | **不是**。它属于 **`col_message`**（整行 Extra 只有这一条），走 `Explain_no_table::explain_extra()`（`opt_explain.cc:811`），**不进 `col_extra` 列表** |
| `Using join buffer (BKA)` | 真串是 **`Using join buffer (Batched Key Access)`**；BNL 被改写打印成 **`Using join buffer (hash join)`** |
| `Using index for order by` | **不存在**。只有 `Using index for group-by`（连字符） |
| `Materialize` / `Start materialize` / `End materialize` / `Scan` | **在数组里定义了，但 8.0.39 全代码无任何 `push_extra()` 调用点** —— 实际输出不会出现（历史遗留） |

> ★ 同理 `Scanned N databases` / `Skip_open_table` / `Open_full_table` / `Open_trigger_table` **在 8.0.39 已全部不存在**（5.x I_S 遗留，已删）。

**★ index merge 的 data 拼接**（`range_optimizer/path_helpers.h:295`，★ 一个反直觉细节）：

```cpp
    case AccessPath::INDEX_MERGE: {
      bool first = true;
      str->append(STRING_WITH_LEN("sort_union("));   /* ★★ 一律拼成 sort_union */
      // For EXPLAIN compatibility with older versions, PRIMARY is always
      // printed last.
      for (bool print_primary : {false, true}) {
        for (AccessPath *child : *path->index_merge().children) {
          const bool is_primary = table->file->primary_key_is_clustered() &&
                                  used_index(child) == table->s->primary_key;
          if (is_primary != print_primary) continue;
          ...
        }
      }
      str->append(')');
```

> ★★ 所以 **`EXPLAIN` 里看到 `Using sort_union(...)` 不一定真是 sort-union 算法**——`INDEX_MERGE` 类型一律打印成 `sort_union(`（注释明说是为了兼容旧版）。这跟 07 篇里"`sort_union` 名字是历史遗留"是同一件事。

### 2.5 key / key_len / possible_keys：索引三兄弟

**`possible_keys`**（`explain_possible_keys`，`opt_explain.cc:938`）：优化器认为"**可能**用得上"的候选索引——就是 `usable_keys` 位图（`tab->keys()`）里置位的所有索引名：

```cpp
// opt_explain.cc:938
if (usable_keys.is_clear_all()) return false;                       // 无候选 → NULL
if ((table->file->ha_table_flags() & HA_NO_INDEX_ACCESS) != 0) return false;  // 引擎不支持索引
for (uint j = 0; j < table->s->keys; j++) {
  if (usable_keys.is_set(j) && ...push_back(table->key_info[j].name))  // 逐个输出索引名
}
```

**读法**：`possible_keys` 是**候选清单，不保证最终用**。它为空（NULL）说明这张表没有任何索引可用（或引擎不支持索引），只能全表扫。

**`key`**（`explain_key_and_len`，`:1514`）：实际选中的索引。三条数据源：

```cpp
if (tab->ref().key_parts)                              // ref 类：ref 的索引
  → explain_key_and_len_index(tab->ref().key, tab->ref().key_length, tab->ref().key_parts);
else if (type == JT_INDEX_SCAN || type == JT_FT)       // index scan：tab->index()
  → explain_key_and_len_index(tab->index());
else if (type == JT_RANGE || type == JT_INDEX_MERGE ...)  // range：range_scan_path 的索引
  → explain_key_and_len_quick(range_scan_path);
```

**`key_len`**（`:979-989`）：索引使用的字节数。**最容易踩的坑：ref 类是"实际用到的 keypart 长度和"，不是整个索引长度**。它的来源是 `init_ref` 里的逐 keypart 累加（`sql_select.cc:2295`）：

```cpp
length += keyinfo->key_part[keyuse->keypart].store_length;   // 每个用到的 keypart 的存储长度
```

`store_length`（索引内每个 keypart 的存储长度）规则：

| 列类型 | store_length |
|--------|-------------|
| 定长列（`INT`/`DATE`/`CHAR(n)`×charset 字节） | `pack_length()`（如 INT = 4） |
| **可空列** | 上面 **+1 字节**（NULL 标记） |
| **变长列**（VARCHAR/TEXT/BLOB） | 数据长度 **+2 字节**（长度前缀，`HA_KEY_BLOB_LENGTH`） |

**key_len 是读 EXPLAIN 的利器**——`key_len` < 索引总长，说明**只用了索引前缀**（最左前缀没全部命中）：

```sql
CREATE INDEX idx ON t(a, b);          -- a: INT NOT NULL(4), b: VARCHAR(10) utf8mb4(10*4+2=42)
EXPLAIN SELECT * FROM t WHERE a=1 AND b='x';   -- key_len=46（4+42，两列都用了）
EXPLAIN SELECT * FROM t WHERE a=1;             -- key_len=4（只用前缀 a，b 没用上）
```

> 注意 `index` 扫描（全索引扫）时 `key_len` 是**整个索引长度**（`key_info[key].key_length`，`:975`），与 ref 类的"实际用到的长度"口径不同。

**★ 两个赋值函数的原文**（`opt_explain.cc:960`）：

```cpp
bool Explain_table_base::explain_key_and_len_index(int key, uint key_length,
                                                   uint key_parts) {
  assert(key != MAX_KEY);

  char buff_key_len[24];
  const KEY *key_info = table->key_info + key;
  const size_t length =
      longlong10_to_str(key_length, buff_key_len, 10) - buff_key_len;
  const bool ret = explain_key_parts(key, key_parts);
  return (ret || fmt->entry()->col_key.set(key_info->name) ||
          fmt->entry()->col_key_len.set(buff_key_len, length));
}

bool Explain_table_base::explain_key_and_len_quick(AccessPath *range_scan) {
  char buff_key_len[24];
  const size_t length =
      longlong10_to_str(get_max_used_key_length(range_scan), buff_key_len, 10) -
      buff_key_len;
  const bool ret = explain_key_parts(-1, 0);
  return (ret || fmt->entry()->col_key.set(::used_index(range_scan) == MAX_KEY
                                               ? "ALL"
                                               : used_key_name(range_scan)) ||
          fmt->entry()->col_key_len.set(buff_key_len, length));
}
```

> ★ `key_len` 是**格式化成十进制字符串**存进 `col_key_len` 的（不是数值）——所以它是 `mem_root_str` 而非 `column<uint>`。
> ★ range 路径下若 `used_index()==MAX_KEY`（如 index merge 无单一索引），`key` 列显示 **`ALL`**。

---

## 三、TREE / JSON：基于 AccessPath

核心在 **`sql/join_optimizer/explain_access_path.cc`**：

- `SetObjectMembers()`（`:934`）：巨型 `switch (path->type)`，为每个类型生成 `operation` 描述与 JSON 成员
- `ExplainAccessPath()`（`:1665`）：递归构建 JSON 树
- `PrintQueryPlan()`（`:1727`）：生成最终 JSON（`ExplainJsonToString` `:1761`）或树形文本（`Explain_format_tree`）

TREE 输出形如：

```
-> Nested loop inner join  (cost=0.55 rows=0)
    -> Filter: (t1.a = t2.a)  (cost=0.35 rows=1)
        -> Table scan on t1  (cost=0.25 rows=1)
    -> Index lookup on t2  (cost=0.30 rows=1)
```

**`cost`/`rows` 的来源**：`rows` 来自 `num_output_rows()`；`cost` 来自 AccessPath 的 **`init_cost`** 字段——**不是 `cost` 字段**。

> ⚠️ 展示层与内部层的差异：EXPLAIN 打印的是"读第一行"的代价（`init_cost`，源码注释明说它更直观、也便于与 `EXPLAIN ANALYZE` 对照），而优化器内部比价用的是"读全量"的 `cost`。同一个查询里，"全量代价高但首行快"的计划在 EXPLAIN 上会显得比实际便宜。三字段语义见 [`08_access_path.md`](../../server/query/08_access_path.md)「代价三字段与过滤前后两套值」。

**★★★ 三个函数各自的职责与"TREE 怎么来的"**：

```cpp
/* sql/join_optimizer/explain_access_path.cc:1727 —— 统一入口 */
std::string PrintQueryPlan(THD *thd, const THD *query_thd,
                           Query_expression *unit, ...);
```

```cpp
/* :1665 —— 递归构建 JSON 树 */
static void ExplainAccessPath(const AccessPath *path, const THD *query_thd,
                              const Query_block *query_block,
                              Join_type join_type, Json_array *parent,
                              bool is_root_of_joined_subtree,
                              const char *attached_condition, ...);
```

```cpp
/* :934 —— 巨型 switch：为每个 AccessPath::Type 生成 operation 描述 */
static bool SetObjectMembers(const AccessPath *path, const THD *query_thd,
                             const Query_block *query_block,
                             const func_ptr_array *materialize_path_param,
                             ...);
```

> ★★ **TREE 与 hypergraph-JSON 是同一份 JSON 的两种渲染**：`ExplainAccessPath()` 构建的是 JSON 树，`PrintQueryPlan()` 之后按格式分派——
> - `Explain_format_tree::ExplainJsonToString()`（`explain_access_path.cc:1830`）→ `ExplainPrintTreeNode()`（`:1854`）渲染成缩进文本；
> - `Explain_format_JSON` 直接输出 JSON 字符串。
>
> 这正好对应基类注释那句 *"they all display exactly the same information, even though the style of each format might be different"*。

**★ `operation` 名是怎么来的**（★ `SetObjectMembers` 里逐类型硬编码，举例）：

```cpp
    case AccessPath::TABLE_SCAN: {
      SetObjectMembers(...);
      error |= AddMemberToObject<Json_string>(obj, "operation", "Table scan");
      ...
    case AccessPath::INDEX_RANGE_SCAN:
      ... "operation", "Index range scan" ...
    case AccessPath::NESTED_LOOP_JOIN:
      ... "operation", "Nested loop inner join" ...
    case AccessPath::HASH_JOIN:
      ... "operation", "Hash join" ...
    case AccessPath::FILTER:
      ... "operation", "Filter" ...
```

> ★ 所以 TREE 里的 `Nested loop inner join` / `Table scan on t1` / `Filter: (...)` 这些字样**全部是 `SetObjectMembers()` 里逐类型硬编码的字符串**，没有统一的命名规则。这也是为什么新增一种 AccessPath 类型必须同步改这个函数。

**★ JSON 有"两套版本"**（`opt_explain_json.h:37`）：

```cpp
class Explain_format_JSON : public Explain_format {
 public:
  enum class FormatVersion { kLinear, kIteratorBased };
  Explain_format_JSON(FormatVersion version) : ..., m_version(version) {}
  /// Format versions newer than Linear are always going to be iterator-based.
  bool is_iterator_based() const override {
    return m_version >= FormatVersion::kIteratorBased;
  }
```

> ★ `kLinear` = 老的 QEP_TAB 体系（线性 join）；`kIteratorBased` = hypergraph + AccessPath。**只有后者支持 `EXPLAIN ANALYZE`**。选择逻辑在 `PT_explain::make_cmd()`（见一、三种输出路径）。

> ⚠️ `explain_json_format_version`（8.3 加的变量）**本版本不存在**，全库 0 匹配。

---

## 四、EXPLAIN ANALYZE

### 4.1 真正执行一遍

`explain_query()` 在 iterator 路径下先跑一遍查询，但结果丢弃（`opt_explain.cc:2242-2250`）：

```cpp
// Run the query, but with the result suppressed.
Query_result_null null_result;
unit->set_query_result(&null_result);
explain_thd->running_explain_analyze = true;
unit->execute(explain_thd);
```

### 4.2 TimingIterator 计时

`NewIterator()`（`timing_iterator.h:221`）在 `is_explain_analyze` 时把真实迭代器包进 `TimingIterator<RealIterator>`：

```cpp
template <class RealIterator, class... Args>
unique_ptr_destroy_only<RowIterator> NewIterator(THD *thd, MEM_ROOT *mem_root, Args &&... args) {
  if (thd->lex->is_explain_analyze) {
    return ... TimingIterator<RealIterator>(thd, ...);
  } else {
    return ... RealIterator(thd, ...);
  }
}
```

`IteratorProfilerImpl`（`timing_iterator.h:40`）维护：`m_num_init_calls`（loops）、`m_num_rows`、`m_elapsed_first_row`、`m_elapsed_other_rows`。

> 特殊：`MaterializeIterator` 和 `TemptableAggregateIterator` **不用** TimingIterator 包裹，自己内部带 profiler（`composite_iterators.cc:657/1608`），以正确拆分"物化时间"与"扫描时间"。

### 4.3 输出

`explain_access_path.cc:881-908` 从 `path->iterator->GetProfiler()` 取数据写 JSON：`actual_first_row_ms` / `actual_last_row_ms` / `actual_rows` / `actual_loops`。树形输出（`:2010-2058`）：

```
-> Table scan on t1  (cost=... rows=...) (actual time=0.5..1.2 rows=100 loops=1)
```

---

## 五、optimizer trace

### 5.1 开关

`SET optimizer_trace="enabled=on"`，注册在 `sys_vars.cc:3543`。配套变量：`optimizer_trace_features`(:3558)、`optimizer_trace_offset`(:3576)、`optimizer_trace_limit`(:3583)、`optimizer_trace_max_mem_size`(:3589)。

### 5.2 结构

- `Opt_trace_context`（`opt_trace_context.h:90`）：每会话一个，挂 `THD::opt_trace`
- `Opt_trace_object`（`opt_trace.h:799`）/ `Opt_trace_array`（`:829`）：RAII 管理的 JSON 构建器
- 未开启时所有 `add_*` 是快速 no-op
- 顶层是 JSON object，含 `steps` 数组（执行阶段示例 `sql_union.cc:1676-1682`）

### 5.3 特性开关

`feature_enabled()`（`opt_trace_context.h:235`）控制追踪哪些特性：`GREEDY_SEARCH` / `RANGE_OPTIMIZER` / `DYNAMIC_RANGE` / `REPEATED_SUBSELECT` / `MISC`。

### 5.4 输出

查 `information_schema.OPTIMIZER_TRACE`（每 trace 一行，字段 QUERY/TRACE/MISSING_BYTES/INSUFFICIENT_PRIVILEGES），由 `Opt_trace_iterator`（`opt_trace.h:407`）遍历。

---


## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → EXPLAIN Output Format*（传统表格格式各列含义）
- *MySQL 8.0 Reference Manual → EXPLAIN ANALYZE*（ANALYZE 的 `actual time` / `loops` / `rows`）
- *MySQL 8.0 Reference Manual → Tracing the Optimizer*（optimizer trace 的 JSON 结构）


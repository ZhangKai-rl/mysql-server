# 生成列深度解析

> 基于 MySQL 8.0.39 源码，涵盖 VIRTUAL/STORED 生成列的元数据模型、读/写两条求值链路、虚拟列二级索引的 InnoDB 实现、undo/purge 与 binlog 复制中的特殊处理，以及隐藏生成列家族（功能索引 / 多值索引 / GIPK）。
>
> **边界**：本篇讲生成列这一特性在 SQL 层与 InnoDB 层的完整闭环。Item 表达式树本身的求值机制（`fix_fields`/`save_in_field`）见 [`../server/query/13_item_expression.md`](../server/query/13_item_expression.md)；DDL 的通用框架（COPY/INPLACE 决策、row log）见 [`../innodb/ddl.md`](../innodb/ddl.md)；binlog 组提交与行事件格式见 [`../server/replication/binlog.md`](../server/replication/binlog.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [语法与元数据模型](#语法与元数据模型yacc--dd--打开时重解析)
  - [写入路径](#写入路径谁被算何时算算进哪个-buffer)
  - [读取路径](#读取路径handler-包装层的一次拦截)
  - [优化器与生成列：表达式替换](#优化器与生成列表达式替换substitute_gc)
  - [虚拟列二级索引：InnoDB 侧元数据模型](#虚拟列二级索引innodb-侧元数据模型)
  - [写入引擎：mysql 行 → dtuple 的 vcol 通道](#写入引擎mysql-行--dtuple-的-vcol-通道)
  - [数据格式：dtuple 双段与索引记录中的 vcol](#数据格式dtuple-双段与索引记录中的-vcol)
  - [求值回调：innobase_get_computed_value](#求值回调innobase_get_computed_value-的六个调用场景)
  - [读引擎：row_search_mvcc 的 vcol 数据流](#读引擎row_search_mvcc-的-vcol-数据流)
  - [UPDATE：update vector 的 vcol 维护](#updateupdate-vector-的-vcol-维护)
  - [undo 格式：vcol 旧值的编码](#undo-格式vcol-旧值的编码)
  - [MVCC 版本链与 purge](#mvcc-版本链与-purgedata_missing-与回溯填充)
  - [online DDL 与外键级联](#online-ddl-与外键级联另两条拉路径)
  - [binlog 与复制](#binlog-与复制不对称的列镜像)
  - [DDL 变更](#ddl-变更adddropmodify-与-instant)
  - [隐藏生成列家族](#隐藏生成列家族一个模板的三种复用)
  - [系统表与视图](#系统表与视图生成列如何对外可见)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

生成列（generated column）是**值由表达式计算得到、不能显式赋值**的列。按是否物理落盘分为两种：

| 类型 | 聚簇索引记录中是否存储 | 能否建二级索引 | 读时代价 |
|---|---|---|---|
| `VIRTUAL`（默认） | 不存储 | 能（值物化在索引页） | 每次读都要算 |
| `STORED` | 存储（等价普通列 + 写时算） | 能 | 读零开销 |

```sql
CREATE TABLE t (
  a INT,
  b INT GENERATED ALWAYS AS (a + 1) VIRTUAL,   -- 虚拟列：行里没有 b
  c INT AS (a * 2) STORED,                     -- 存储列：行里有 c
  KEY (b)                                       -- 虚拟列索引：索引页里有 b
);
```

### 用途

一个反直觉的事实开场：**虚拟列在主键/聚簇索引记录里不占一个字节，但它上面的二级索引却是"真材实料"的**——索引页里存着每个 `b` 的值。这意味着一张表的行宽和某个查询能走的索引是两件事：为查询建虚拟列索引，几乎不给每行增加任何存储成本。

它解决的核心问题是"**在表达式结果上建索引**"：

- `WHERE a + 1 = 5` 永远用不上 `a` 的索引，但 `b AS (a+1)` 的索引可以；
- `WHERE JSON_EXTRACT(doc, '$.price') = 9.99` 在 JSON 列上无法定位，虚拟列索引可以。

在整条链路中它的位置是横切的：DDL 时进元数据（DD），打开表时重解析表达式，DML 读/写时参与求值，优化器把它当普通列选索引，引擎层为它物化索引条目并维护 undo/binlog。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.7（WL#411） | 引入生成列：VIRTUAL/STORED 两种，GA 支持虚拟列二级索引（INPLACE 建索引）；**原始 HLS 曾限制只有 STORED 列可建索引**（虚拟列索引被划给函数索引 WL#1075），GA 前放开；早期版本 InnoDB 仍为虚拟列在行里预留"垃圾占位"，直到 WL#8114 才完全跳过 |
| 5.7 后期（WL#8114） | InnoDB 完全跳过虚拟列存储：聚簇记录中 vcol 从"占位垃圾值"变为真正缺席 |
| 8.0.0 | 数据字典重构：表达式文本从 `.frm` 移到 DD 表（`mysql.columns` 的 `generation_expression` / `generation_expression_utf8` 双列），打开表时重解析 |
| 8.0.13（WL#10761） | **功能索引**：`INDEX((CAST(j->>'$.x' AS UNSIGNED)))` 以"隐藏虚拟生成列 + 普通索引"实现——WL#411 FUTURE PLAN 里留给 WL#1075 的目标落地 |
| 8.0.13 | `DEFAULT (<expr>)` 列默认值表达式，复用 `Value_generator` 基础设施 |
| 8.0.16（WL#6049） | CHECK 约束同样挂到 `Value_generator` 上 |
| 8.0.21（WL#8763） | 多值索引（`CAST(json AS UNSIGNED ARRAY)`）：typed-array 类型的隐藏虚拟列 + 多值索引 |
| 8.0.23（WL#13601） | GIPK（不可见主键）：无主键表自动加隐藏虚拟列 `my_row_id` |
| 8.0.29 | instant ADD/DROP COLUMN 覆盖纯虚拟列的增删（`INSTANT_VIRTUAL_ONLY`），vcol 索引随 instant add 一起建 |

---

## 理论基础

### 设计思想与权衡

**1. VIRTUAL 与 STORED 的分野是空间-时间权衡的显式化。**

虚拟列把"算"推迟到每次读；存储列把"算"提前到每次写。默认 VIRTUAL 的选择背后的判断是：行存储按字节收费（页、redo、undo、binlog 全都跟着变大），而计算能力通常过剩。代价的另外一半同样真实：

- 虚拟列上建二级索引后，**索引页里存了 vcol 值、聚簇行里却没有**。于是"索引里有、行里没有"的非对称成为一切复杂性的根源：回表校验要重算、UPDATE 要先算新值才能定位旧索引记录、purge 线程要能独立求值。
- `update_generated_read_fields` 每次读行都要对 `read_set` 里的 vcol 跑一遍表达式树（`save_in_field`），全表扫描含 vcol 时会放大 CPU 开销——这正是"读零开销"的存储列存在的原因。

**2. 计算职责放在 SQL 层，引擎通过回调借用——这是一个反向依赖。**

InnoDB 没有表达式引擎。但虚拟列索引的插入、purge、外键级联都需要 vcol 值，而这些时刻常常发生在引擎内部深处（甚至 purge 线程里，那时根本没有 `TABLE` 对象）。MySQL 的方案是让引擎**持有 mysql 行格式的模板（`vc_templ`），把基列值摆进 buffer，回调 server 层求值**（`handler::my_eval_gcolumn_expr`）。

本来可以那样做、但没有：把 vcol 值在 INSERT 时就随 `dtuple_t` 一起塞给引擎，引擎只当数据搬运工。这确实发生了——正常 DML 路径 vcol 值由 server 算好、经 `row_mysql_convert_row_to_innobase` 装入 dtuple 的 vfield 段，引擎插入索引时直接用。但 purge 时没有 server 上下文、外键级联时基列值尚未落定，这些路径绕不过回调。于是两套机制并存：**正常路径"推"值，异常路径"拉"值**。回调接口（`my_eval_gcolumn_expr` / `my_eval_gcolumn_expr_with_open` / `my_prepare_gcolumn_template`）正是为"拉"而设的。

**3. 二级索引必须物化 vcol 值，没有第二种选择。**

B+ 树按索引记录排序、用索引记录定位聚簇行。若索引记录里只有基列没有 vcol 值，比较器无值可比。所以"虚拟"的边界被精确划在**聚簇索引之外**：vcol 在聚簇记录中缺席，在二级索引记录中物化。这不是实现偷懒，是数据结构决定的必然。

**4. 非确定性禁止：正确性优先于表达能力。**

`validate_value_generator_expr` 直接拒绝非确定性函数（`RAND()`、`NOW()` 等）、子查询、存储函数、变量引用。理由不只在"值会漂移"，更硬的是**复制正确性**：row-based 复制下备库重算 vcol，若表达式非确定，主备立刻分叉；`Value_generator::m_backup_binlog_stmt_flags` 的存在说明 SBR 下还要把表达式的"语句不安全标志"备份下来合并进语句的 unsafe flags。

**5. 权衡的另一半：失效场景与退化点。**

- **覆盖索引例外**：`update_generated_read_fields` 里 `active_index != MAX_KEY && table->key_read` 时直接返回——索引提供了 vcol 值。但反向优化缺失：`SELECT b FROM t WHERE a = 5`（b 依赖 a）**不会**用 `a` 的索引做覆盖扫描再算 b。函数头注释明说：MySQL 的 index-scan 决策逻辑把 b 当作独立于 a 的列，Bug#21815348 就是这事的改进请求，至今未做。后果：这类查询要么回表，要么靠用户显式给 b 建索引。
- **前缀索引的 vcol**：`KEY(b(10))` 时索引里只有前缀，读出来要按 CHAR 语义补空格（`row_sel_store_mysql_field` 里 `templ->is_virtual && mysql_col_len > len` 分支）。
- **BLOB 虚拟列的 `keep_old_value`**：引擎不存 vcol 的旧值，UPDATE 时 BLOB vcol 的新旧值都只在 server 内存里，`update_generated_columns` 必须 `blob->keep_old_value()`，否则引擎维护索引时拿不到 before-image。
- **vcol 链式依赖有方向**：生成列只能引用表中**先于它定义**的列（含更早的生成列），自引用/前向引用报 `ER_GENERATED_COLUMN_NON_PRIOR`。这直接让 `vfield` 数组顺序 = 求值拓扑序，求值器单遍扫描即可，无需拓扑排序。代价是定义顺序变成语义的一部分。WL#411 原始设计即选定了此语义（与 MariaDB"禁止 GC 间引用"的分歧点，见「他库对比」）。

**6. 语法选型的一手决策记录（WL#411 附录 B）。** 为什么是 `AS (expr) VIRTUAL` 而不是别的：

- **Oracle 语法** `... AS (expr) VIRTUAL` 被否决——Oracle 里省略 VIRTUAL 仍是虚拟列，"VIRTUAL"是语义噪音，照搬有误导风险；
- **SQL Server 语法** `AS expr PERSISTED` 被认为"不地道"（has an unEnglish air）；
- 早期 HLS 还提过 `[[NOT] VIRTUAL]`（默认 VIRTUAL、用 NOT VIRTUAL 表示存储），最终换成更明确的 STORED；
- 定案：非标准的 `[VIRTUAL | STORED]` 后缀——直接、明确、不与标准语义冲突。

同期放弃的还有标准 SQL（T175）的**隐式类型推导**（`GENERATED ALWAYS AS (5)` 不写类型）：做类型推导需要过早 `fix_fields`，5.7 的架构做不到，只能要求显式类型。这个"架构限制塑造语法"的案例与「表达式长度 64K」（从补丁的 255 字节放宽以容纳 JSON/GIS 长表达式）一起，说明生成列的语言设计同时被两个方向拉扯：标准对齐 vs Server 架构能力。

**7. 求值时序的精确位置：BEFORE 触发器之后、约束检查之前。** WL#411 明确定义 STORED 生成列在 INSERT/UPDATE 的 BEFORE 触发器之后、CHECK 约束之前求值。后果有二：触发器对基列的修改会改变生成列的值（因此触发器路径要二次重算，见「写入路径」）；触发器对生成列本身的赋值被丢弃（重算覆盖）。这个时序 8.0 未变。

### 理论溯源

- **空间-时间权衡（space-time tradeoff）**：VIRTUAL vs STORED 的经典实例。生成列的语义来自 SQL 标准的 `<column definition>` 扩展，MySQL 的实现形态（默认 virtual、可 persist）与 SQL Server 的 computed column / Oracle 11g 的 virtual column 一脉相承。
- **函数索引（functional index）**：数据库在 `f(x)` 上建索引的通用方案有二——直接支持表达式索引（Oracle、PostgreSQL 的语法），或先造一个 `f(x)` 的隐藏列再给隐藏列建普通索引（SQL Server 的 computed column index）。MySQL 走了后者，功能索引 = 隐藏虚拟生成列（`HT_HIDDEN_SQL`）+ 普通二级索引，全部复用生成列既有基础设施。这个选择让功能索引免费获得 vcol 的一切维护逻辑（undo 里的 vcol 值、purge 重算、binlog 过滤），但代价是索引列名 `!hidden!` 式样、`SHOW CREATE TABLE` 里多出一列。
- **惰性求值（lazy evaluation）**：虚拟列的读取路径是惰性的——只有进了 `read_set` 的 vcol 才会在 `update_generated_read_fields` 里被算。

### 算法与数据结构

**`Value_generator`**（生成列表达式在 SQL 层的句柄）：

```cpp
class Value_generator {
  Item *expr_item{nullptr};          // 解析后的表达式树（每 TABLE 一份）
  LEX_STRING expr_str{nullptr, 0};   // DD 里的表达式文本（TABLE_SHARE 持有）
  uint32 m_backup_binlog_stmt_flags{0}; // 解析时语句的 unsafe flags 备份
  Item *item_list{nullptr};          // 解析产生的所有 Item（用于统一释放）
  MY_BITMAP base_columns_map;        // 依赖的基列位图
  enum_field_types field_type{MYSQL_TYPE_INVALID};
  bool stored_in_db{false};          // VIRTUAL or STORED
  uint num_non_virtual_base_cols{0};
};
```

- `expr_str` 是持久化形态（DD 文本），`expr_item` 是运行形态（Item 树）。TABLE_SHARE 只有前者，每个 TABLE 打开时各自解析出后者——因为 Item 树不能跨线程共享。
- `base_columns_map` 是位图而非列表：UPDATE 时判断"这个生成列要不要重算"退化为一次 `bitmap_is_overlapping(write_set, base_columns_map)`，O(1)。它在 `register_base_columns` 里靠 `Item::walk(&Item::mark_field_in_map)` 收集——**复用了 Item 树的 visitor 基础设施**，没有为生成列写专门的表达式遍历。

**`TABLE::vfield`**：`Field**` 的 NULL 结尾数组，成员是全部生成列的 `Field*`，按 `field_index` 升序。`TABLE_SHARE::vfields` 存数量。求值器（`update_generated_columns`）只扫这一个数组，`has_gcol()` 即 `vfield != nullptr`。

**InnoDB 侧 `dict_v_col_t` 与 `dict_vcol_templ_t`**：`dict_v_col_t` 记录 vcol 的基列指针数组 `base_col[]`（InnoDB 也持有一份依赖关系，用于外键级联和 DDL 场景）；`dict_vcol_templ_t` 是**mysql 行格式的模板缓存**——`vtempl[]` 前 `n_col` 个是基列的 `mysql_row_templ_t`，后 `n_v_col` 个是 vcol 的模板。引擎求值时把基列值按模板摆进一个 `rec_len` 大小的 buffer，直接调 server 求值。

### 他库对比与演进动机

| 数据库 | 机制 | 差异 |
|---|---|---|
| Oracle | virtual column（11g）+ 函数索引（直接表达式） | 与 MySQL 同为虚拟列路线，但函数索引是原生表达式索引，不是隐藏列 |
| PostgreSQL | generated column（12+，**仅 STORED**），表达式索引原生支持 | 没有 VIRTUAL；PG 的函数索引直接表达式，无隐藏列 |
| SQL Server | computed column（可 `PERSISTED`）+ computed column index | 与 MySQL 最像：非持久 computed column 即虚拟列，索引方案同为"隐藏物化" |
| MariaDB | `GENERATED ALWAYS AS (expr) [VIRTUAL\|PERSISTENT]`（**早于 MySQL**） | 语法同源（MySQL 5.7 是对齐标准的后发者），但**禁止生成列引用生成列**；MySQL 允许引用先定义的 GC（WL#411 明确记录了这一分歧与代价：解析验证更复杂、必须按定义序求值） |

演进动机链（结合 WL#411 原文）：5.7 引入生成列的动机有三——**统一复杂表达式**（多个查询共享同一条条件）、**物化缓存**（STORED 列算一次存下来）、以及 WL#1075 函数索引落地前的**过渡方案**（"STORED 列 + 二级索引"间接获得函数索引效果，代价是数据双份存储，原文明说"真正的函数索引只需存储一次"）；8.0 把表达式从 `.frm` 挪进 DD 是字典重构的顺带产物；8.0.13 功能索引（WL#10761）把 WL#411 FUTURE PLAN 的目标落地为"隐藏 vcol + 索引"，实现上零新机制、全复用；GIPK 与多值索引同理，都是"隐藏 vcol + 索引"模板的复用。一条清晰的"渐进式兼容"路线：5.7 先接受存储浪费（垃圾占位 + 双份数据）换快速交付，中期由 WL#8114/WL#10761 清理，长期解除 WL#411 里自认"人为"的限制（作者原话：基列↔虚拟列互转限制"看起来完全是人为的"）。

---

## 核心实现

### 主链路

**写（INSERT/UPDATE）**：

```
mysql_insert / Sql_cmd_update
  → table->mark_generated_columns(is_update)     // 生成列标记进 read/write_set
  → fill_record(..., table)                       // 用户值入库
    → update_generated_write_fields(write_set, table)
      → update_generated_columns(table, ..., virtual_only=false)
        → vfield 逐个: gcol_info->expr_item->save_in_field(field)  // SQL 层算值
  → handler::ha_write_row / ha_update_row
    → binlog_log_row(...)                         // 行镜像打包（INSERT 含 vcol）
    → ha_innobase::write_row
      → row_insert_for_mysql
        → row_mysql_convert_row_to_innobase      // mysql 行 → dtuple，vcol 值进 vfield 段
          → row_ins_clust_index_entry            // 聚簇索引只写物理列
          → row_ins_sec_index_entry              // 二级索引（含 vcol 索引）写索引记录
```

**读（SELECT）**：

```
TableScanIterator / IndexRangeScanIterator
  → handler::ha_rnd_next / ha_index_next(buf, ...)
    → m_update_generated_read_fields = table->has_gcol()
    → index_next / rnd_next                      // 引擎取行
    → update_generated_read_fields(buf, table, active_index)
      // covering index 时跳过；否则 update_generated_columns(read_set, virtual_only=true)
  → 上层 Item_field::val_* 从 record[0] 读 vcol 值
```

两条链的共同汇点是 `update_generated_columns`——一个按 `vfield` 单遍扫描的求值器。

### 语法与元数据模型：yacc → DD → 打开时重解析

生成列的语法规则把表达式直接当作 `expr` 规则解析（`field_def` 的一个分支）：

```
field_def:
      type opt_collate opt_generated_always
      AS '(' expr ')'
      opt_stored_attribute opt_column_attribute_list
      { $$ = NEW_PTN PT_generated_field_def($1, $6, $8, opt_attrs); }

opt_stored_attribute:
      %empty      { $$ = Virtual_or_stored::VIRTUAL; }   // 默认 VIRTUAL
    | VIRTUAL_SYM { $$ = Virtual_or_stored::VIRTUAL; }
    | STORED_SYM  { $$ = Virtual_or_stored::STORED; }
```

解析产物 `PT_generated_field_def` 在 contextualize 阶段把 `Item` 树与 `Virtual_or_stored` 装进 `Create_field::gcol_info`（`Value_generator*`）。至此表达式还是"活的 Item"；持久化时（`fill_dd_column_from_create_field`）用 `print_expr` 把 Item 树**反序列化回文本**，存入 DD 列 `generation_expression_utf8`，`is_virtual` 位记 VIRTUAL/STORED。

打开表时的对称动作（`open_table_from_share`）对每个带 `gcol_info` 的字段调 `unpack_value_generator`——这是本篇第一个值得展开的机制：

```cpp
bool unpack_value_generator(THD *thd, TABLE *table,
                            Value_generator **val_generator, ...) {
  LEX *const save_lex = thd->lex;
  LEX new_lex;
  thd->lex = &new_lex;                         // 全新 LEX，与主查询隔离
  if (lex_start(thd)) { ... }

  Query_arena val_generator_arena(&table->mem_root, ...);  // 挂在表 mem_root
  thd->swap_query_arena(val_generator_arena, &save_arena);
  thd->want_privilege = 0;                     // 打开表不需要权限检查

  // 拼出 "PARSE_GCOL_EXPR (<DD 里的表达式文本>)"
  char *gcol_expr_str = ...;                   // mem_root 分配
  memcpy(gcol_expr_str, PARSE_GCOL_KEYWORD.str, ...);   // "parse_gcol_expr"
  ...
  // 用专用 parser state 重入解析器
  Gcol_expr_parser_state parser_state;
  parser_state.init(thd, gcol_expr_str, str_len);
  if (parse_sql(thd, &parser_state, nullptr)) return true;
  *val_generator = parser_state.result;        // 拿到新的 Item 树

  // fix_fields + 合法性校验（禁止非确定函数/子查询/存储函数/变量）
  if (fix_value_generator_fields(thd, table, *val_generator, source, ...)) ...

  // 收集依赖列到 base_columns_map
  if ((*val_generator)->register_base_columns(table)) return true;
  (*val_generator)->backup_stmt_unsafe_flags(new_lex.get_stmt_unsafe_flags());
  return false;
}
```

要点：

1. **`PARSE_GCOL_EXPR` 是一个伪关键字语法**。yacc 里有专门的 `GRAMMAR_SELECTOR_GCOL IDENT_sys '(' expr ')'` 起始规则：解析器见到 `parse_gcol_expr` 开头就走这条，把 `expr` 挂进新建的 `Value_generator` 返回。这是"DD→文本→yacc 重解析"往返（与分区信息的 `GRAMMAR_SELECTOR_PART` 同构），避免了在 DD 里存二进制序列化的 Item 树——后者不可版本迁移。
2. **每次打开表都重新解析**（5.7 如此，8.0 换 DD 后依旧）。所以打开一张多生成列表的成本包含 N 次表达式解析；表达式文本解析后，`item_list` 记录全部 Item 供 `closefrm` 统一释放。
3. **权限零检查**（`want_privilege = 0`）：DD 里的表达式是 DBA 在建表时已授权的，打开者无需再被授权。

`fix_value_generator_fields` 之后的 `validate_value_generator_expr` 是合法性闸门：非确定函数（`expr->is_non_deterministic()`）、`PARAM_ITEM`、系统/用户变量（`GSYSVAR_FUNC`/`GUSERVAR_FUNC`）全部拒绝，并 `walk` 整棵树检查命名函数白名单。报错信息精确到函数名：

```
Expression of generated column 'a' contains a disallowed function: `p1`.
```

`register_base_columns` 收集依赖：

```cpp
bool Value_generator::register_base_columns(TABLE *table) {
  bitmap_init(&base_columns_map, bitbuf, table->s->fields);
  MY_BITMAP *save_old_read_set = table->read_set;
  table->read_set = &base_columns_map;               // 借用 read_set 位图
  Mark_field mark_fld(MARK_COLUMNS_TEMP);
  expr_item->walk(&Item::mark_field_in_map, enum_walk::PREFIX,
                  pointer_cast<uchar *>(&mark_fld)); // 走树收集 Item_field
  table->read_set = save_old_read_set;
  ...
}
```

把 `base_columns_map` 临时塞给 `table->read_set` 再走树，是对"位图收集依赖"惯用法（`mark_column_used` 系列）的复用。

### 写入路径：谁被算、何时算、算进哪个 buffer

写路径的第一问是"这次写哪些生成列需要算"。`TABLE::mark_generated_columns(bool is_update)` 在 `mark_columns_needed_for_insert`/`mark_columns_needed_for_update` 里被调：

```cpp
void TABLE::mark_generated_columns(bool is_update) {
  if (is_update) {
    for (vfield_ptr = vfield; *vfield_ptr; vfield_ptr++) {
      tmp_vfield = *vfield_ptr;
      /*
        We need to evaluate the GC if:
        - it depends on any updated column
        - or it is virtual indexed, for example:
           * UPDATE changes the primary key's value, and the virtual index
             is a secondary index which includes the pk's value
           * the gcol is in a multi-column index, and UPDATE changes another
             column of this index
           * in both cases the entry in the index needs to change, so needs to
             be located first, for that the GC's value is needed.
      */
      if ((!tmp_vfield->stored_in_db && tmp_vfield->m_indexed) ||
          bitmap_is_overlapping(write_set,
                                &tmp_vfield->gcol_info->base_columns_map)) {
        tmp_vfield->table->mark_column_used(tmp_vfield, MARK_COLUMNS_WRITE);
        tmp_vfield->table->mark_column_used(tmp_vfield, MARK_COLUMNS_READ); // 旧值
        bitmap_updated = true;
      }
    }
  } else {   // INSERT：所有生成列都算
    for (vfield_ptr = vfield; *vfield_ptr; vfield_ptr++)
      tmp_vfield->table->mark_column_used(tmp_vfield, MARK_COLUMNS_WRITE);
  }
  if (bitmap_updated) file->column_bitmaps_signal();
}
```

两个洞察：

- UPDATE 的重算条件是**依赖重叠或虚拟列被索引**。第二条件很微妙：vcol 值没变也要算（且读旧值），因为多列索引里别的列变了/主键变了，vcol 索引条目要跟着搬，定位旧条目需要 vcol 值——**vcol 索引把 vcol 的求值从"值变了才算"拖成"涉及它就必算"**。
- `mark_column_used(MARK_COLUMNS_WRITE)` 对 gcol 会继续调 `mark_gcol_in_maps`：gcol 进 `write_set`，其全部基列进 `read_set`（基列是虚拟列时递归标进 `write_set`）。注意基列进 read_set 但**不动 `covering_keys`**——所以 `SELECT gcol FROM t` 仍可选只覆盖 gcol 的索引；引擎侧 `build_template_needs_field` 在 key_read 下忽略 read_set，不会白读基列。这个注释里点明的依赖关系，是两层协定的典型样本。

真正的计算发生在 `fill_record` 末尾：

```cpp
// sql_base.cc fill_record()
...
  if (table->has_gcol() &&
      update_generated_write_fields(bitmap ? bitmap : table->write_set, table))
    return true;
```

```cpp
bool update_generated_write_fields(const MY_BITMAP *bitmap, TABLE *table) {
  return update_generated_columns(table, bitmap, false,     // virtual_only=false
                                  table->fields_set_during_insert);
}

static bool update_generated_columns(TABLE *table, const MY_BITMAP *columns,
                                     bool virtual_only,
                                     MY_BITMAP *updated_columns) {
  for (Field **field_ptr = table->vfield; *field_ptr != nullptr; ++field_ptr) {
    Field *field = *field_ptr;
    if (virtual_only && !field->is_virtual_gcol()) continue;   // 读路径跳过 stored
    if (!bitmap_is_set(columns, field->field_index())) continue;

    // 虚拟 BLOB 列：保留旧值供引擎 UPDATE 使用
    if (field->handle_old_value()) {
      const auto blob = down_cast<Field_blob *>(field);
      blob->keep_old_value();
      blob->set_keep_old_value(true);
    }

    type_conversion_status status =
        field->gcol_info->expr_item->save_in_field(field, false);  // 求值+类型转换
    if (status != TYPE_OK && thd->is_error()) return true;
    if (updated_columns != nullptr)
      bitmap_set_bit(updated_columns, field->field_index());       // 记入
                                                                   // fields_set_during_insert
  }
  return false;
}
```

- 单遍扫描 `vfield` 就是全部算法——生成列只能引用先定义的列，vfield 升序即拓扑序，**无需拓扑排序**。
- `save_in_field` 同时做表达式求值与向目标类型转换；严格模式下的截断/溢出会报错回滚。
- `updated_columns`（`table->fields_set_during_insert`）记录本次算过哪些 gcol：后续 CHECK 约束求值、`INSERT ... ON DUPLICATE KEY UPDATE`、触发器重算都要用。触发器会改基列值，因此 `Table_triggers_list` 的路径在触发器跑完后**再算一次**（`has_updated_trigger_fields(write_set)` 分支），且此时 `blobs_need_not_keep_old_value()`——第一次算已留了 before-image，第二次不留。

### 读取路径：handler 包装层的一次拦截

读路径挂在 handler 的每个取行包装函数上（`ha_rnd_next`/`ha_index_next`/`ha_index_read_map`/`ha_multi_range_read_next`/`ha_ft_read`/`ha_sample_next` 等），模式统一：

```cpp
int handler::ha_index_next(uchar *buf) {
  ...
  m_update_generated_read_fields = table->has_gcol();
  result = index_next(buf);                       // 引擎取行
  if (!result && m_update_generated_read_fields) {
    result = update_generated_read_fields(buf, table, active_index);
    m_update_generated_read_fields = false;
  }
  ...
}
```

`m_update_generated_read_fields` 是 handler 上的标志位，在 wrapper 内先置位、取行后复位，作用是让"读行后统一补算"这一动作贯穿所有取行接口（引擎本身从不修改它——覆盖扫描的短路发生在 `update_generated_read_fields` 内部的 `key_read` 分支）。

```cpp
bool update_generated_read_fields(uchar *buf, TABLE *table, uint active_index) {
  if (current_thd->is_error()) return true;
  if (active_index != MAX_KEY && table->key_read) {
    /* 覆盖索引提供了所有需要的列（含生成列），跳过计算。
       Bug#21815348：优化器不会为 "依赖 A 的 B" 选 A 的索引做
       index-only scan，所以到不了这个分支；若做了，这里要改。 */
    return false;
  }
  if (buf != table->record[0])
    repoint_field_to_record(table, table->record[0], buf);   // Field 指针重定位
  const bool error =
      update_generated_columns(table, table->read_set, true /*virtual_only*/, nullptr);
  if (buf != table->record[0])
    repoint_field_to_record(table, buf, table->record[0]);
  return error;
}
```

- `virtual_only=true`：stored 列的值引擎已随行返回，只有虚拟列需要补算。
- 计算范围由 `read_set` 圈定：查询没用到、也没被索引的 vcol 不算——**惰性求值**的落点。
- `repoint_field_to_record` 是 mysql 层的老机制：`Field` 对象内嵌指向 record buffer 的指针，MRR/排序等把行读进临时 buffer 时要把这些指针重定位过去再求值（`Field` 不持有 buffer 所有权）。

### 优化器与生成列：表达式替换（substitute_gc）

生成列索引要生效，还差最后一环：**查询里的表达式要能"认出"等价于某个生成列**。`WHERE a + 1 = 5` 与 `WHERE b = 5`（`b AS (a+1)`）语义相同，但前者的 Item 树里没有 `Item_field(b)`，range 优化器无从下手。`substitute_gc` 在优化阶段把前者改写成后者：

```cpp
// sql_optimizer.cc: Query_block::optimize 的入口处
if ((where_cond || !group_list.empty() || !order.empty()) &&
    substitute_gc(thd, query_block, where_cond, group_list.order, order.order)) {
  // 替换发生时，被换进的字段要计入 all_fields（隐藏部分）
  count_field_types(query_block, &tmp_table_param, query_block->fields, false, false);
}
```

调用点共三处：`Query_block::optimize`（SELECT/多表 DML）、`Sql_cmd_delete::delete_from_single_table`、`Sql_cmd_update`（单表 DELETE/UPDATE 的 prepare）。替换分两层：

**第一层：构造"可替换生成列"候选集。** `substitute_gc` 遍历所有表，把**被索引**（`part_of_key` 或前缀键）且键未被 hint 禁用的生成列收集进 `indexed_gc` 列表（opt_trace 记 `substitute_generated_columns` 节）。

**第二层：对 WHERE/ORDER/GROUP 的 Item 树 walk `gc_subst_transformer`。** 支持谓词类型有明确清单：`EQ/LT/LE/GE/GT`、`BETWEEN`、`IN`、`MEMBER OF`、`JSON_CONTAINS`、`JSON_OVERLAPS`。核心匹配逻辑在 `substitute_gc_expression`：

```cpp
static bool substitute_gc_expression(Item **expr, Item **value,
                                     List<Field> *gc_fields, Item_result type,
                                     Item_func *predicate) {
  List_iterator<Field> li(*gc_fields);
  Item_field *item_field = nullptr;
  while (Field *field = li++) {
    // 检查该生成列有没有可用键（含前缀键，受 keys_in_use_for_query 约束）
    Key_map tkm = field->part_of_key;
    tkm.merge(field->part_of_prefixkey);
    tkm.intersect(field->table->keys_in_use_for_query);
    /*
      Don't substitute if:
      1) Key is disabled
      2) It's a multi-valued index's field and predicate isn't MEMBER OF
    */
    if (tkm.is_clear_all() ||
        (field->is_array() && predicate->functype() != Item_func::MEMBER_OF_FUNC))
      continue;
    /* 功能索引的隐藏列还要求 collation 与表达式一致（bug#27337092）；
       普通生成列的这一检查推迟到后续修复（影响面大） */
    if (!(field->is_field_for_functional_index() &&
          field->match_collation_to_optimize_range() &&
          (*expr)->collation.collation != field->charset())) {
      item_field = get_gc_for_expr(*expr, field, type);   // ★ 等价性判定
      if (item_field != nullptr) break;
    }
  }

  if (item_field == nullptr) return false;

  /* 找到等价生成列：把表达式替换成 Item_field，并登记到 change_item_tree */
  ...
  thd->change_item_tree(expr, item_field);
  // Adjust the predicate.
  return predicate->resolve_type(thd);
}
```

等价性判定发生在 `get_gc_for_expr`（递归比较 Item 树：把候选生成列的 `gcol_info->expr_item` 与谓词中的表达式做 `eq()` 逐节点比对，含 collation 归一）。替换成功即 `change_item_tree(expr, item_field)`——谓词瞬间变成对生成列字段的引用，后续 range 优化器把它当普通索引列用（KEY_PART_INFO 直接指向 vcol 字段，物理层对 vcol 索引零特判，见引擎各节）。

设计上值得注意的三点：

1. **替换是"为索引服务"的**——候选集只收**被索引**的生成列（`part_of_key ∩ keys_in_use_for_query`），`WHERE a+1=5` 在 b 无索引时保持原样。生成列不是全局查询重写机制，是索引匹配的延伸。这也正是 WL#411 FUTURE PLAN 里"查询重写：SELECT s2 不展开为 (s1+5)"的反面——MySQL 的替换方向是**表达式 → 生成列**（为了用索引），从不做**生成列 → 表达式**的展开。
2. **MEMBER OF/JSON_CONTAINS/JSON_OVERLAPS 的替换要求右值常量且可被 coerce**（`coerce_json_value(no_error=true)`，转换失败放弃替换）——多值索引替换的正确性由类型强转保证，代价是不匹配时静默放弃。
3. **ORDER BY/GROUP BY 的替换**（`substitute_gc` 后半段）：表达式匹配成功后换成 `Item_field` 并**加入 all_fields 隐藏部分**（`count_field_types` 重新计数），让排序/分组直接利用索引序。

### 虚拟列二级索引：InnoDB 侧元数据模型

InnoDB 字典里 vcol 是一等公民，有自己的结构链：

```cpp
struct dict_v_col_t {
  dict_col_t m_col;              // vcol 自身的 dict_col_t（is_virtual()==true）
  dict_col_t **base_col;         // 基列指针数组（InnoDB 也持有一份依赖关系）
  ulint num_base;                // 基列个数
  ulint v_pos;                   // 在表的 vcol 数组中的位置（与字段序一致）
  dict_v_idx_list *v_indexes;    // 引用该 vcol 的虚拟索引列表（索引 + 列位置）
};
```

- `dict_index_t` 用 `DICT_VIRTUAL` 类型标志区分（`dict_index_has_virtual()` 即 `index->type & DICT_VIRTUAL`）；索引字段数组里 vcol 位置的 `get_col(i)` 返回的是 `dict_v_col_t::m_col`。
- `dict_table_t::n_v_cols`（虚拟列总数）与 `n_v_def`（当前已定义数）分离：DDL 中间态下两者可以不一致。
- **`v_indexes` 是 undo 编码的基础**（见后文 undo 小节）：undo 里不直接存 vcol 位置，而是存"这个 vcol 被哪些索引、以第几列引用"的清单，读回时按索引 id 在当前字典里匹配——这天然处理了索引被 DROP/重建的场景。
- 表首次打开时构建模板缓存 `dict_vcol_templ_t`（`dict_table_t::vc_templ`）：`vtempl[]` 前 `n_col` 个是**被引用的基列**的 `mysql_row_templ_t`，后 `n_v_col` 个是 vcol 自己的模板。它把"引擎内部 dtuple 坐标系"与"mysql record buffer 坐标系"对齐，是回调求值的基础设施。`innobase_build_v_templ` 先扫 `base_col` 标出 `marker[]`（哪些列是基列），再遍历 `table->field[]`：虚拟列走 `innobase_vcol_build_templ` 填 vcol 模板，被标记的物理列填基列模板。模板在字典锁保护下构建，且带引用计数（`get_ref_count() == 1` 时才刷新，防止并发 handler 踩坏别人的模板）；分区表由 `Ha_innopart_share::set_v_templ` 对每个分区表重复同一过程。

单个模板的填充逻辑完整贴出：

```cpp
static void innobase_vcol_build_templ(const TABLE *table,
                                      const dict_index_t *clust_index,
                                      Field *field, const dict_col_t *col,
                                      mysql_row_templ_t *templ, ulint col_no) {
  if (col->is_virtual()) {
    templ->is_virtual = true;
    templ->col_no = col_no;
    templ->clust_rec_field_no = ULINT_UNDEFINED;   // ★ 聚簇记录里没有这个字段
    templ->rec_field_no = col->ind;
  } else {
    templ->is_virtual = false;
    templ->clust_rec_field_no = dict_col_get_clust_pos(col, clust_index);
    ut_a(templ->clust_rec_field_no != ULINT_UNDEFINED);
    templ->rec_field_no = templ->clust_rec_field_no;
  }

  templ->icp_rec_field_no = ULINT_UNDEFINED;

  if (field->is_nullable()) {                      // mysql 行 NULL 位图坐标
    templ->mysql_null_byte_offset = field->null_offset();
    templ->mysql_null_bit_mask = (ulint)field->null_bit;
  } else {
    templ->mysql_null_bit_mask = 0;
  }

  templ->mysql_col_offset = static_cast<ulint>(get_field_offset(table, field));
  templ->mysql_col_len = static_cast<ulint>(field->pack_length());

  /* 多值索引的索引字段长度与列实际数据长度不同 */
  if (templ->is_virtual && innobase_is_multi_value_fld(field)) {
    templ->mysql_mvidx_len = static_cast<ulint>(field->key_length());
    templ->is_multi_val = true;
  } else {
    templ->mysql_mvidx_len = 0;
    templ->is_multi_val = false;
  }

  templ->type = col->mtype;
  templ->mysql_type = static_cast<ulint>(field->type());
  if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR)
    templ->mysql_length_bytes = field->get_length_bytes();
  templ->charset = dtype_get_charset_coll(col->prtype);
  templ->mbminlen = col->get_mbminlen();
  templ->mbmaxlen = col->get_mbmaxlen();
  templ->is_unsigned = col->prtype & DATA_UNSIGNED;
}
```

三个要点：

1. **`clust_rec_field_no = ULINT_UNDEFINED` 是模板层面对"聚簇行里没有 vcol"的直接表达**。引擎里一切"从聚簇记录按字段号取数据"的逻辑（`rec_get_nth_field`、`row_sel_field_store_in_mysql_format` 的簇定位）看到这个哨兵就知道该字段不从聚簇记录走。同一份模板同时服务两个方向：基列（`clust_rec_field_no` 有效）用于"从聚簇行把基列搬进 mysql buffer"，vcol（`rec_field_no = col->ind` 有效）用于"把算好的值从 mysql buffer 搬回 dtuple 的 vfield"。
2. `mysql_col_offset` 来自 `get_field_offset(table, field)`——mysql record buffer 里**虚拟列有位置**（server 的行布局按字段序排），引擎只管把值摆到那个偏移。NULL 位同理（`field->null_offset()`/`null_bit`）。
3. 多值索引特判：索引里存的是 JSON 数组元素，字段长度按 `key_length()` 而非列 `pack_length()`。

### 写入引擎：mysql 行 → dtuple 的 vcol 通道

DML 写行时 server 已算好 vcol 值，引擎的职责是把它从 mysql record 转进 dtuple。`row_mysql_convert_row_to_innobase` 是唯一入口：

```cpp
static void row_mysql_convert_row_to_innobase(
    dtuple_t *row, row_prebuilt_t *prebuilt, const byte *mysql_rec,
    mem_heap_t **heap) {
  const mysql_row_templ_t *templ;
  dfield_t *dfield;
  ulint i;
  ulint n_col = 0;      // 物理列游标 → dtuple 的 fields 段
  ulint n_v_col = 0;    // 虚拟列游标 → dtuple 的 vfields 段
  ulint n_m_v_col = 0;

  ut_ad(prebuilt->template_type == ROW_MYSQL_WHOLE_ROW);
  ut_ad(prebuilt->mysql_template);

  for (i = 0; i < prebuilt->n_template; i++) {
    bool is_multi_val = false;
    templ = prebuilt->mysql_template + i;

    if (templ->is_virtual) {
      ut_ad(n_v_col < dtuple_get_n_v_fields(row));
      dfield = dtuple_get_nth_v_field(row, n_v_col);   // ★ 装进 vfield 段
      n_v_col++;
      if (dfield_is_multi_value(dfield)) {
        is_multi_val = true;
        n_m_v_col++;
      }
    } else {
      dfield = dtuple_get_nth_field(row, n_col);       // 物理列
      n_col++;
    }

    if (templ->mysql_null_bit_mask != 0) {
      /* Column may be SQL NULL */
      if (mysql_rec[templ->mysql_null_byte_offset] &
          (byte)(templ->mysql_null_bit_mask)) {
        dfield_set_null(dfield);
        continue;                                      // NULL：不搬数据
      }
    }

    if (is_multi_val) {
      dict_v_col_t *v_col = dict_table_get_nth_v_col(prebuilt->table, n_v_col - 1);
      innobase_get_multi_value(prebuilt->m_mysql_table, v_col->m_col.ind,
                               dfield, &prebuilt->mv_data[n_m_v_col - 1], 0,
                               dict_table_is_comp(prebuilt->table),
                               prebuilt->heap);
      /* 多值数据必须深拷贝：之后再有 vcol 需要计算时，
      server 会覆写这块内存（insert by modify 场景） */
      if (*heap == nullptr) {
        *heap = mem_heap_create(128, UT_LOCATION_HERE);
      }
      dfield_multi_value_dup(dfield, *heap);
    } else {
      row_mysql_store_col_in_innobase_format(
          dfield, prebuilt->ins_upd_rec_buff + templ->mysql_col_offset,
          true /* MySQL row format data */,
          mysql_rec + templ->mysql_col_offset, templ->mysql_col_len,
          dict_table_is_comp(prebuilt->table));

      /* server 对 BLOB 虚拟字段处理有已知问题，
      必须用引擎自己的内存复制一份 */
      if (templ->is_virtual && DATA_LARGE_MTYPE(dfield_get_type(dfield)->mtype)) {
        if (*heap == nullptr) {
          *heap = mem_heap_create(dfield->len, UT_LOCATION_HERE);
        }
        dfield_dup(dfield, *heap);
      }
    }
  }

  /* FTS doc id 若未由用户提供则在此分配 */
  if (prebuilt->table->fts) {
    fts_create_doc_id(prebuilt->table, row, prebuilt->heap);
  }
}
```

结构上值得注意：

- **模板序列 = 物理列（`n_col` 个）接虚拟列（`n_v_col` 个）**，与 dtuple 的 fields/vfields 双段结构一一对应，两个游标分别推进。`dtuple_create_with_vcol(heap, n_cols, n_v_cols)` 创建的 row 就是这种双段形态——聚簇 entry 的 fields 段只含物理列，`row_ins_clust_index_entry` 写聚簇记录时天然写不进 vcol。
- 值转换走 `row_mysql_store_col_in_innobase_format`（mysql 格式 → InnoDB 内部格式：字节序、CHAR 去空格、时区等），vcol 与物理列同一转换路径，只是目标 dfield 不同。
- **BLOB vcol 必须 `dfield_dup` 深拷贝**：mysql record 的内存属于 server（`ins_upd_rec_buff` 只是引用），之后若引擎内部再触发 vcol 计算（insert by modify），server 会覆写那块内存；注释点明"server has issue regarding handling BLOB virtual fields"。多值数据同理必须 `dfield_multi_value_dup`。

随后 `row_ins_clust_index_entry` 写聚簇记录（无 vcol），`row_ins_sec_index_entry` → `row_ins_index_entry_set_vals` 构建索引 entry：vcol 字段从 row 的 vfield 段取（`dtuple_get_nth_v_field(row, v_col->v_pos)`），物理字段从 fields 段取——**vcol 值从此物理存在于二级索引页**。这是整个机制最关键的一步不对称。

### 数据格式：dtuple 双段与索引记录中的 vcol

**dtuple 的双段布局。** 携带 vcol 的 dtuple 由 `dtuple_create_with_vcol(heap, n_fields, n_v_fields)` 创建：fields 段（物理列）与 vfields 段（虚拟列）连续分配，`dtuple_get_nth_v_field(row, v_pos)` 直接索引第二段。`dtuple_init_v_fld` 把整个 vfields 段初始化成 `DATA_MISSING`（`mtype = DATA_MISSING, len = UNIV_SQL_NULL`）——`DATA_MISSING` 是贯穿引擎的"这个 vcol 值尚未获得"哨兵，读路径跳过、MVCC 回溯填充、DDL 重算，全看它。

**二级索引记录里 vcol 是"普通字段"。** 物化之后，vcol 在索引记录中的物理布局与普通字段**完全同构**：占 NULL 位图一位、变长列有长度字节、定长列定长存放。`rec_get_nth_field` / `rec_get_offsets` / `rec_init_offsets` 对含 vcol 的索引记录**没有任何特判**——物理层根本不知道什么是"虚拟列"。这是物化方案最大的工程收益：整条 B+ 树基础设施（搜索、分裂、MVCC、崩溃恢复）零改动即支持 vcol 索引。

区分发生在**内存表示**层面：vcol 字段的 `dfield_t::type.prtype` 带三个专用标志：

| 标志 | 语义 | 设置点 |
|---|---|---|
| `DATA_VIRTUAL` | 该 dfield 是虚拟列字段 | `innobase_get_computed_value` 返回时、`dict_col_copy_type` 拷贝 vcol 类型时 |
| `DATA_MISSING` | 值未获得（哨兵） | `dtuple_init_v_fld` |
| `DATA_MULTI_VALUE` | 多值索引的数组字段 | 多值 vcol 相关路径 |

`row_upd_store_v_row` 里按 `new_val.type.prtype & DATA_VIRTUAL` 过滤 update vector、`cmp_dfield_dfield` 对 vcol 用同样的比较逻辑——都是靠 prtype 而非位置判断。注意 `DATA_VIRTUAL` 不进索引记录（页上字段不带此标志），只存在于内存 dfield。

**索引 entry 的构建**（`row_ins_index_entry_set_vals`）：遍历 `n_fields + num_v` 个位置，越过物理字段数后切换到 vfield 段：

```cpp
for (i = 0; i < n_fields + num_v; i++) {
  dict_field_t *ind_field = nullptr;
  dfield_t *field;
  const dfield_t *row_field;
  ...
  if (i >= n_fields) {
    /* This is virtual field */
    field = dtuple_get_nth_v_field(entry, i - n_fields);
    col = &dict_table_get_nth_v_col(index->table, i - n_fields)->m_col;
  } else {
    field = dtuple_get_nth_field(entry, i);
    ind_field = index->get_field(i);
    col = ind_field->col;
    ...
  }
  ...
  if (col->is_virtual()) {
    row_field = dtuple_get_nth_v_field(row, v_col->v_pos);   // 值来源：行 vfield
  } else {
    row_field = dtuple_get_nth_field(row, ind_field->col->ind);
  }
  ...
}
```

索引 entry 里 vcol 字段的位置由 `index->get_field(i)->col->is_virtual()` 判定，值统一从聚簇行的 vfield 段取——与 entry 自身是"物理字段在前、vcol 字段在后"的布局无关，索引定义里 vcol 可以出现在任意位置（`v_indexes->nth_field` 记录列位置）。

**前缀 vcol 索引的物理截断。** `KEY(b(10))` 时索引记录只存 b 的前 10 个字符（按 `prefix_len` 截断，索引 entry 构建时 `len = dtype_get_at_most_n_mbchars(...)` 处理），回读时引擎要按 CHAR 语义补空格/对齐——`row_sel_store_mysql_field` 里 `templ->is_virtual && mysql_col_len > len` 的分支注释明说：前缀虚拟索引读不出完整值，只能在 server 层算，但 server 假设返回的值尾部带填充空格，引擎侧先补上。MVCC 校验侧（`row_vers_vc_matches_cluster`）则反过来：undo 记的是 `prefix_len * mbmaxlen` 字节，utf8mb4 下与索引记录实际长度不同，比较前把重建值对齐到 `field1->len`。

**stored 生成列的行格式**：STORED 列在聚簇记录里就是普通字段（有物理位置、参与 undo/redo），InnoDB 字典里唯一的额外记录是 `dict_s_col_t`（`dict_table_t::s_cols` 列表，`dict_mem_table_add_s_col` 维护）：保存 stored 列的基列指针，供**外键检查**用——FK 引用的列若是 stored 生成列的基列，级联更新会改写 stored 值，`dict_foreigns_has_s_base_col` 直接拒绝（`DB_NO_FK_ON_S_BASE_COL`）。也就是说 stored 列在存储上零特殊、在字典上为约束语义保留了一份依赖记录。

### 求值回调：innobase_get_computed_value 的六个调用场景

vfield 段里没有值（或值不可信）时，引擎走"拉"路径：

```cpp
dfield_t *innobase_get_computed_value(
    const dtuple_t *row, const dict_v_col_t *col, const dict_index_t *index,
    mem_heap_t **local_heap, mem_heap_t *heap, const dict_field_t *ifield,
    THD *thd, TABLE *mysql_table, const dict_table_t *old_table,
    upd_t *parent_update, dict_foreign_t *foreign) {
  const mysql_row_templ_t *vctempl =
      index->table->vc_templ->vtempl[index->table->vc_templ->n_col + col->v_pos];
  byte rec_buf1[REC_VERSION_56_MAX_INDEX_COL_LEN];
  byte rec_buf2[REC_VERSION_56_MAX_INDEX_COL_LEN];
  byte *mysql_rec;
  ...
  /* 行宽 < 3072 时复用栈上两块 buffer，否则从 local_heap 分配 */
  if (!heap || index->table->vc_templ->rec_len >= REC_VERSION_56_MAX_INDEX_COL_LEN) {
    if (*local_heap == nullptr)
      *local_heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);
    mysql_rec = static_cast<byte *>(mem_heap_alloc(*local_heap, ...rec_len));
    buf = static_cast<byte *>(mem_heap_alloc(*local_heap, ...rec_len));
  } else {
    mysql_rec = rec_buf1;
    buf = rec_buf2;
  }

  /* 第 1 步：把基列值按模板摆进 mysql_rec */
  for (ulint i = 0; i < col->num_base; i++) {
    dict_col_t *base_col = col->base_col[i];
    const dfield_t *row_field = nullptr;
    uint32_t col_no = base_col->ind;
    const mysql_row_templ_t *templ = index->table->vc_templ->vtempl[col_no];
    const byte *data;

    if (parent_update != nullptr)      // 外键级联：基列新值在 update vector 里
      row_field = innobase_get_field_from_update_vector(foreign, parent_update, col_no);
    if (row_field == nullptr)          // 否则取行里的当前值
      row_field = dtuple_get_nth_field(row, col_no);
    data = static_cast<const byte *>(row_field->data);
    len = row_field->len;

    if (row_field->ext) {              // 外置列：把 BLOB 拉回内存
      if (*local_heap == nullptr)
        *local_heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);
      data = lob::btr_copy_externally_stored_field(
          thd_to_trx(thd), clust_index, &len, nullptr, data, page_size,
          dfield_get_len(row_field), false, *local_heap);
    }

    if (len == UNIV_SQL_NULL) {
      mysql_rec[templ->mysql_null_byte_offset] |= (byte)templ->mysql_null_bit_mask;
      memcpy(mysql_rec + templ->mysql_col_offset,
             index->table->vc_templ->default_rec + templ->mysql_col_offset,
             templ->mysql_col_len);    // NULL 时回填 default_rec 的默认值
    } else {
      row_sel_field_store_in_mysql_format(
          mysql_rec + templ->mysql_col_offset, templ, index,
          templ->clust_rec_field_no, (const byte *)data, len, ULINT_UNDEFINED);
      if (templ->mysql_null_bit_mask)
        mysql_rec[templ->mysql_null_byte_offset] &= ~(byte)templ->mysql_null_bit_mask;
    }
  }

  field = dtuple_get_nth_v_field(row, col->v_pos);

  /* 第 2 步：指定要算哪个 vcol，回调 server 层 */
  MY_BITMAP column_map;
  my_bitmap_map col_map_storage[bitmap_buffer_size(REC_MAX_N_FIELDS)];
  bitmap_init(&column_map, col_map_storage, REC_MAX_N_FIELDS);
  bitmap_set_bit(&column_map, col->m_col.ind);

  Temp_table_handle tblhdl;
  if (mysql_table == nullptr) {
    if (vctempl->type == DATA_BLOB) {   // BLOB vcol：预留 blob ref 空间
      ...row_mysql_store_blob_ref(mysql_rec + vctempl->mysql_col_offset, ...);
    }
    /* 无现成 TABLE 时临时开一个（purge 等场景） */
    mysql_table = tblhdl.open(thd, index->table->vc_templ->db_name.c_str(),
                              index->table->vc_templ->tb_name.c_str());
  }
  if (mysql_table) {
    ret = handler::my_eval_gcolumn_expr(
        thd, mysql_table, &column_map, (uchar *)mysql_rec,
        (col->m_col.is_multi_value() ? &mv_data_ptr : nullptr),
        (col->m_col.is_multi_value() ? &mv_length : nullptr));
  } else {
    return nullptr;
  }

  if (ret != 0) {                       // 求值失败
    ...return (nullptr);
  }

  if (vctempl->mysql_null_bit_mask &&
      (mysql_rec[vctempl->mysql_null_byte_offset] & vctempl->mysql_null_bit_mask)) {
    dfield_set_null(field);             // 结果为 NULL
    field->type.prtype |= DATA_VIRTUAL;
    if (col->m_col.is_multi_value())
      field->type.prtype |= DATA_MULTI_VALUE;
    return (field);
  }
  ... // 非 NULL：从 mysql_rec 读出值 dfield_set_data；多值还要解析 JSON 数组
}
```

几点设计细节：

- **栈 buffer 优化**：行宽 < 3072 字节时用栈上 `rec_buf1/rec_buf2`，避免每次回调分配堆内存——vcol 求值可能发生在每条 UPDATE 的热路径上（`row_upd_build_difference_binary` 对每条被更新行跑一遍）。超过阈值才退到 `local_heap`。
- `default_rec` 是建模板时从 `table->s->default_values` 拷贝的全列默认值行：基列为 NULL 时不能只置 NULL 位（内存可能是脏的），要回填默认值保证 server 求值时读到确定性的数据。
- 多值 vcol 走 `mv_data_ptr`/`mv_length` 专用通道：server 算出的 JSON 二进制再被 `innobase_store_multi_value` 解析成数组 dfield。
- 返回值带 `DATA_VIRTUAL` prtype 标志——引擎侧用它识别 vcol 字段（如 `row_upd_store_v_row` 按 `new_val.type.prtype & DATA_VIRTUAL` 过滤 update vector）。

六个调用场景汇总：

| 场景 | 调用点 | TABLE 来源 |
|---|---|---|
| 回表校验（索引记录 vs 聚簇行） | `row_sel_sec_rec_is_for_clust_rec` | prebuilt 现成 `m_mysql_table` |
| UPDATE 计算旧 vcol 值 | `row_upd_build_difference_binary` | 调用者传入 |
| 回滚/删除行重建 vrow | `row_upd_store_v_row` | 调用者传入 |
| MVCC 版本链填充 vrow | `row_vers_build_clust_v_col` 等 | `current_thd` / 临时开表 |
| online DDL 建 vcol 索引 | ddl0builder 的 `get_virtual_column` | ctx 里的 `m_my_table` |
| 外键级联维护 vcol 索引 | `row_ins_foreign_fill_virtual` | `current_thd` |

回调的 server 侧 `my_eval_gcolumn_expr_helper` 与之对称：`repoint_field_to_record(table, old_buf, record)` 把 TABLE 的 Field 指到引擎拼好的 buffer，`dbug_tmp_use_all_columns` 放开 read/write_set 限制（引擎请求的列可能不在本查询的 set 里），再对请求列**及其依赖的虚拟列闭包**逐个 `save_in_field`。purge 线程走 `my_eval_gcolumn_expr_with_open`：没有现成 TABLE，就 `open_table_uncached` 临时开一个（`open_in_engine=false`，否则与 purge 上下文死锁），算完还要 `copy_blob_data` 把 blob 拷进引擎分配的持久内存——TABLE 关闭后 Item 内存即失效。

这个回调设计回答了理论部分的权衡问题：**引擎的所有 vcol 需求最终都借 server 的表达式引擎满足，引擎只负责摆数据**。代价也摆在那里：purge 中算 vcol 要开表（DD 访问），一条链上既有锁序又有性能风险。

### 读引擎：row_search_mvcc 的 vcol 数据流

扫描含 vcol 的二级索引时，`row_search_mvcc` 的第一件事是决定要不要维护 vrow（vcol 值的 dtuple 载体）：

```cpp
bool need_vrow = dict_index_has_virtual(prebuilt->index) &&
                 (prebuilt->read_just_key || prebuilt->m_read_virtual_key);
```

两个条件的语义：

- `read_just_key`（covering scan）：优化器认定索引覆盖查询列（含 vcol），vcol 值要从**索引记录**里读出来填进结果行；
- `m_read_virtual_key`：分区表跨分区索引扫描时由 `ha_innopart::index_read` 置位——跨分区的扫描状态（记录缓冲、优先级队列）需要在分区之间携带 key 里的 vcol 值，否则切分区后 key 信息丢失。

回表时 vrow 一路向下传递（`row_sel_get_clust_rec_for_mysql(..., need_vrow ? &vrow : nullptr, ...)`），聚簇行拿不到 vcol 时从二级索引记录补：

```cpp
/* row_sel_fill_vrow：从含 vcol 的二级索引记录提取 vcol 值到 dtuple 的 vfield 段 */
static void row_sel_fill_vrow(const rec_t *rec, dict_index_t *index,
                              const dtuple_t **vrow, mem_heap_t *heap) {
  ulint offsets_[REC_OFFS_NORMAL_SIZE];
  ulint *offsets = offsets_;
  rec_offs_init(offsets_);
  ut_ad(!(*vrow));
  offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED,
                            UT_LOCATION_HERE, &heap);
  *vrow = dtuple_create_with_vcol(heap, 0, dict_table_get_n_v_cols(index->table));
  dtuple_init_v_fld(*vrow);                 // 全部置 DATA_MISSING

  for (ulint i = 0; i < dict_index_get_n_fields(index); i++) {
    const dict_field_t *field = index->get_field(i);
    const dict_col_t *col = field->col;
    if (col->is_virtual()) {
      const byte *data;
      ulint len;
      data = rec_get_nth_field(index, rec, offsets, i, &len);   // 索引记录里读
      const dict_v_col_t *vcol = reinterpret_cast<const dict_v_col_t *>(col);
      dfield_t *dfield = dtuple_get_nth_v_field(*vrow, vcol->v_pos);
      dfield_set_data(dfield, data, len);
      col->copy_type(dfield_get_type(dfield));
    }
  }
}
```

`DATA_MISSING` 是 vrow 的初始态标记：vrow 只保证**索引 key 里出现的** vcol 有值，其余保持 MISSING——消费方（`row_sel_store_mysql_rec`）看到 MISSING 就知道不能从这里取。

聚簇行转 mysql 行时 vcol 的最终裁决（`row_sel_store_mysql_rec` 主循环）：

```cpp
for (ulint i = 0; i < prebuilt->n_template; i++) {
  const auto templ = &prebuilt->mysql_template[i];

  /* 多值列永远不出现在查询结果里，跳过 */
  if (templ->is_multi_val) {
    ut_ad(templ->is_virtual);
    continue;
  }

  if (templ->is_virtual && rec_index->is_clustered()) {
    /* 非覆盖扫描、非 virtual key read、或无聚簇行 → 跳过，留给 server 补算 */
    if ((prebuilt_index != nullptr &&
         !dict_index_has_virtual(prebuilt_index)) ||
        (!prebuilt->read_just_key && !prebuilt->m_read_virtual_key) ||
        !rec_clust) {
      continue;
    }

    dict_v_col_t *col =
        dict_table_get_nth_v_col(rec_index->table, templ->clust_rec_field_no);
    ut_ad(vrow);
    const auto dfield = dtuple_get_nth_v_field(vrow, col->v_pos);

    /* 分区表可能只物化了 key 里的 vcol，非 key vcol 保持 DATA_MISSING → 跳过 */
    if (dfield_get_type(dfield)->mtype == DATA_MISSING) {
      ut_ad(prebuilt->m_read_virtual_key);
      ut_ad(prebuilt_index->get_col_pos(col->v_pos, false, true) == ULINT_UNDEFINED);
      continue;
    }

    if (dfield->len == UNIV_SQL_NULL) {              // NULL 位 + 默认值回填
      mysql_rec[templ->mysql_null_byte_offset] |= (byte)templ->mysql_null_bit_mask;
      memcpy(mysql_rec + templ->mysql_col_offset,
             (const byte *)prebuilt->default_rec + templ->mysql_col_offset,
             templ->mysql_col_len);
    } else {                                         // 值写进 mysql 行
      row_sel_field_store_in_mysql_format(
          mysql_rec + templ->mysql_col_offset, templ, rec_index,
          templ->clust_rec_field_no, (const byte *)dfield->data, dfield->len,
          ULINT_UNDEFINED);
      if (templ->mysql_null_bit_mask)
        mysql_rec[templ->mysql_null_byte_offset] &= ~(byte)templ->mysql_null_bit_mask;
    }
    continue;
  }
  ... // 非虚拟列/非聚簇读的常规路径
}
```

注意这里的 `templ` 是 `prebuilt->mysql_template`（每次扫描按 read_set/ICP 需求由 `build_template_field` 现建），**与表级共享的 `vc_templ` 是两套模板**：`build_template_field` 对 vcol 直接设 `clust_rec_field_no = v_no`（vcol 位置复用这个字段号），而 `vc_templ` 里 vcol 的 `clust_rec_field_no` 是 `ULINT_UNDEFINED`（因为回调求值不需要聚簇定位）。同一个 `clust_rec_field_no` 成员两处语义：基列时是聚簇字段号，vcol 时是 v_pos——读引擎代码时必须先分清当前是哪套模板。

prefetch cache 的对称处理（`row_sel_pop_cached_row`）：

```cpp
for (i = 0; i < prebuilt->n_template; i++) {
  templ = prebuilt->mysql_template + i;
  /* Skip virtual columns */
  if (templ->is_virtual) {
    if (!(dict_index_has_virtual(prebuilt->index) && prebuilt->read_just_key))
      continue;                       // 非覆盖扫描：缓存行里没有 vcol
    if (templ->is_multi_val) continue;
  }
  row_sel_copy_cached_field_for_mysql(buf, cached_rec, templ);
}
```

覆盖扫描时 vcol 值在 `row_sel_store_mysql_rec` 阶段就已写进缓存的 mysql 行，这里直接整段拷贝；非覆盖扫描时缓存行里 vcol 位置是脏的，必须跳过、由 server 的 `update_generated_read_fields` 补算。**同一份行数据在两条路径上的 vcol 空位约定完全相反，这是读路径最容易踩的坑。**

**回表校验**：`row_sel_sec_rec_is_for_clust_rec` 比较索引记录与聚簇行是否匹配时，vcol 无法从聚簇行取，只能现场重算后与索引里的物化值比对：

```cpp
if (col->is_virtual()) {
  const dict_v_col_t *v_col = reinterpret_cast<const dict_v_col_t *>(col);
  row = row_build(ROW_COPY_POINTERS, clust_index, clust_rec, clust_offs, ...);
  vfield = innobase_get_computed_value(row, v_col, clust_index, &heap, heap,
                                       nullptr, ..., thr->prebuilt->m_mysql_table,
                                       nullptr, nullptr, nullptr);
  if (vfield == nullptr) {            // READ-UNCOMMITTED 下 JSON 外置列未写完
    err = DB_COMPUTE_VALUE_FAILED;
    goto func_exit;
  }
  clust_len = vfield->len;
  clust_field = static_cast<byte *>(vfield->data);
}
```

`DB_COMPUTE_VALUE_FAILED` 的失效边界写死在注释里：READ-UNCOMMITTED 下基列是未提交的 JSON 外置 LOB 时求值失败。ICP（index condition pushdown）则直接受益于物化：`row_search_idx_cond_check` 对 `templ.is_virtual && templ.icp_rec_field_no != ULINT_UNDEFINED` 的列从索引记录读值——vcol 上的条件下推到引擎后无需回表即能判断，`icp_rec_field_no` 由 `innobase_index_cond` 在索引里定位 vcol 列时填充。

### UPDATE：update vector 的 vcol 维护

UPDATE 的核心是 `row_upd_build_difference_binary`——聚簇行新旧二进制比对、生成 update vector。vcol 的处理是独立的第二段：

```cpp
/* 注释原文：即使没有非虚拟列（基列）变化，仍要构建被索引虚拟列的值，
   让 undo log 记录它们（供 purge/MVCC 用） */
if (n_v_fld > 0) {
  row_ext_t *ext;
  mem_heap_t *v_heap = nullptr;
  THD *thd;
  ...
  ut_ad(!update->old_vrow);

  for (i = 0; i < n_v_fld; i++) {
    const dict_v_col_t *col = dict_table_get_nth_v_col(index->table, i);

    if (!col->m_col.ord_part) {          // 只处理被索引的 vcol
      continue;
    }

    if (update->old_vrow == nullptr) {   // 惰性构建旧行 dtuple
      update->old_vrow =
          row_build(ROW_COPY_POINTERS, index, rec, offsets, index->table,
                    nullptr, nullptr, &ext, heap);
    }

    dfield = dtuple_get_nth_v_field(entry, i);      // 新行里的 vcol 值（server 算好）

    dfield_t *vfield = innobase_get_computed_value(
        update->old_vrow, col, index, &v_heap, heap, nullptr, thd,
        mysql_table, nullptr, nullptr, nullptr);    // 旧值：从旧行现场重算

    if (vfield == nullptr) {
      *error = DB_COMPUTE_VALUE_FAILED;
      return nullptr;
    }

    /* 新旧二进制相等 → 不生成 upd_field：undo 省空间、索引条目不用动 */
    if (!dfield_data_is_binary_equal(dfield, vfield->len,
                                     static_cast<byte *>(vfield->data))) {
      upd_field = upd_get_nth_field(update, n_diff);
      upd_field->old_v_val = static_cast<dfield_t *>(
          mem_heap_alloc(heap, sizeof *upd_field->old_v_val));
      dfield_copy(upd_field->old_v_val, vfield);    // ★ 旧值进 update vector
      dfield_copy(&(upd_field->new_val), dfield);
      upd_field_set_v_field_no(upd_field, i, index);  // vcol 位置编码
      n_diff++;
    }
  }
  ...
}
```

三个洞察：

1. **即使基列完全没变，被索引 vcol 也必算**——注释明说目的在 undo：undo 里要存 old_v_val，供 purge/MVCC 重建索引条目。这是 vcol 索引带来的固定写放大，与值变不变无关。
2. **旧 vcol 值只能现场重算**：旧行是聚簇记录（`row_build` 成 dtuple），旧 vcol 值无处可存——它是"引擎从 server 拿新值那一刻才消失"的量。所以对 `old_vrow` 再走一遍回调。`dfield_data_is_binary_equal` 的二进制比较省掉了值不变时的 undo 与索引维护。
3. `upd_field_set_v_field_no` 把 vcol 位置写进 upd_field 的 `field_no`（编码规则见 undo 小节），`upd_field_t::old_v_val` 成为 undo 与回滚的共同载体。

回滚/删除方向（`row_upd_store_v_row`）为行版本填充 vcol 旧值，三来源递进：

```cpp
static void row_upd_store_v_row(upd_node_t *node, const upd_t *update, THD *thd,
                                TABLE *mysql_table) {
  ...
  for (i = 0; i < n_upd; i++) {                  // 对每个被索引 vcol
    const dict_v_col_t *col = ...;
    dfield = dtuple_get_nth_v_field(node->row, col_no);

    for (i = 0; i < n_upd; i++) {                // 来源 1：update vector 里有 old_v_val
      const upd_field_t *upd_field = upd_get_nth_field(update, i);
      if (!(upd_field->new_val.type.prtype & DATA_VIRTUAL) ||
          upd_field->field_no != col->v_pos) {
        continue;
      }
      dfield_copy_data(dfield, upd_field->old_v_val);
      ...dup...
      break;
    }

    /* 来源 2：update->old_vrow 里有旧值 */
    if (i >= n_upd) {
      if (update) {
        if (update->old_vrow == nullptr) {
          /* 只发生在 cascade update：vcol 不受影响，置 NULL 安全 */
          dfield_set_null(dfield);
        } else {
          dfield_t *vfield = dtuple_get_nth_v_field(update->old_vrow, col_no);
          dfield_copy_data(dfield, vfield);
          ...
          /* 旧值非 NULL 就直接用，不需要回调；
             只有 NULL 时（old_vrow 未算过）才兜底计算 */
          if (dfield_is_null(dfield)) {
            if (!new_val_v_cols_dup) {
              row_upd_dup_v_new_vals(update);   // 先 dup new_val，防止被覆盖
              new_val_v_cols_dup = true;
            }
            innobase_get_computed_value(node->row, col, index, &heap,
                                        node->heap, nullptr, thd, mysql_table,
                                        nullptr, nullptr, nullptr);
          }
        }
      } else {
        /* 来源 3：删除行，无 update vector —— 只能现场计算 */
        innobase_get_computed_value(node->row, col, index, &heap, node->heap,
                                    nullptr, thd, mysql_table, nullptr,
                                    nullptr, nullptr);
      }
    }
  }
}
```

来源 2 里的注释点出一个精妙的成本优化：`old_vrow` 里该 vcol 若是 NULL，说明**它从未被算过**，此时才回调；且回调前必须先 `row_upd_dup_v_new_vals` 深拷贝所有 new_val——因为 `innobase_get_computed_value` 会写 dtuple 的 vfield，可能覆盖 update vector 引用的内存。

### undo 格式：vcol 旧值的编码

undo 里 vcol 字段用 `field_no >= REC_MAX_N_FIELDS` 的越界值编码（`is_virtual = (field_no >= REC_MAX_N_FIELDS)`），随后紧跟一段"v_idx"元数据，记录该 vcol 被哪些索引以第几列引用：

```cpp
static byte *trx_undo_log_v_idx(page_t *undo_page, const dict_table_t *table,
                                ulint pos, byte *ptr, bool first_v_col) {
  ut_ad(pos < table->n_v_def);
  dict_v_col_t *vcol = dict_table_get_nth_v_col(table, pos);
  ulint n_idx = vcol->v_indexes->size();          // 从字典直接读引用计数
  byte *old_ptr;
  ut_ad(n_idx > 0);

  /* 预留：版本标记 1B + 总长 2B + n_idx 压缩 + 每个索引 {id, nth_field} 压缩 */
  ulint size = n_idx * (5 + 5) + 5 + 2 + (first_v_col ? 1 : 0);
  if (trx_undo_left(undo_page, ptr) < size) return (nullptr);

  if (first_v_col) {
    mach_write_to_1(ptr, VIRTUAL_COL_UNDO_FORMAT_1);   // ★ 首个 vcol 前写版本标记
    ptr += 1;
  }
  old_ptr = ptr;
  ptr += 2;                                          // 总长先占位

  ptr += mach_write_compressed(ptr, n_idx);
  for (it = vcol->v_indexes->begin(); it != vcol->v_indexes->end(); ++it) {
    dict_v_idx_t v_index = *it;
    ptr += mach_write_compressed(ptr, static_cast<ulint>(v_index.index->id));
    ptr += mach_write_compressed(ptr, v_index.nth_field);
  }
  mach_write_to_2(old_ptr, ptr - old_ptr);           // 回填总长
  return (ptr);
}
```

读侧对称，且**按索引 id 在当前字典里匹配**：

```cpp
static const byte *trx_undo_read_v_idx_low(const dict_table_t *table,
                                           const byte *ptr, ulint *col_pos) {
  ulint len = mach_read_from_2(ptr);
  const byte *old_ptr = ptr;
  *col_pos = ULINT_UNDEFINED;
  ptr += 2;
  ulint num_idx = mach_read_next_compressed(&ptr);
  const dict_index_t *clust_index = table->first_index();

  for (ulint i = 0; i < num_idx; i++) {
    space_index_t id = mach_read_next_compressed(&ptr);
    ulint pos = mach_read_next_compressed(&ptr);
    const dict_index_t *index = clust_index->next();

    while (index != nullptr) {
      if (index->id == id) {               // 在当前字典里按 id 找索引
        const dict_col_t *col = index->get_col(pos);
        ut_ad(col->is_virtual());
        const dict_v_col_t *vcol = reinterpret_cast<const dict_v_col_t *>(col);
        *col_pos = vcol->v_pos;            // 换算成 v_pos 返回
        return (old_ptr + len);
      }
      index = index->next();
    }
  }
  return (old_ptr + len);                  // 找不到 → col_pos 保持 UNDEFINED
}
```

设计意图：

- **不直接存 vcol 位置、存"索引引用清单"**：undo 可能被读到时索引已被 DROP/重建，直接存位置会错位。按 id 匹配失败 → `col_pos = ULINT_UNDEFINED` → 调用方（`trx_undo_rec_get_col_val` 等）把该字段标为"不再需要"（`upd_field->field_no = REC_MAX_N_FIELDS` 后跳过），undo 天然容忍 DDL 与索引变更。
- **版本标记**：`trx_undo_read_v_idx` 对首个 vcol 读一字节判断是否 `VIRTUAL_COL_UNDO_FORMAT_1`——undo 与 online DDL 的 row log 共用这条解析代码，标记区分两者；非 undo 格式（旧格式/row log）退化为 `*field_no -= REC_MAX_N_FIELDS` 直接解出 v_pos。

undo 记录本体在 `trx_undo_page_report_modify` 里写 vcol 字段时有两处特判：

```cpp
// ① 不再被任何索引的 vcol 直接不写 undo（online DDL 期间常见的中间态）
if (upd_fld_is_virtual_col(fld) &&
    dict_table_get_nth_v_col(table, pos)->v_indexes->empty()) {
  n_updated--;
}

// ② vcol 字段：field_no 之后先写 v_idx 元数据，再写值
if (is_virtual) {
  ut_ad(fld->field_no < table->n_v_def);
  ptr = trx_undo_log_v_idx(undo_page, table, fld->field_no, ptr, first_v_col);
  first_v_col = false;
  ...
}

// ③ vcol 先写旧值（old_v_val）……
if (is_virtual) {
  field = static_cast<byte *>(fld->old_v_val->data);
  flen = fld->old_v_val->len;
  if (flen != UNIV_SQL_NULL) flen = std::min(flen, max_v_log_len);
}
...ut_memcpy(ptr, field, flen); ptr += flen;

// ④ ……再补写新值（普通字段只写旧值）
/* Also record the new value for virtual column */
if (is_virtual) {
  field = static_cast<byte *>(fld->new_val.data);
  flen = fld->new_val.len;
  if (flen != UNIV_SQL_NULL) flen = std::min(flen, max_v_log_len);
  ...
  ut_memcpy(ptr, field, flen);
  ptr += flen;
}
```

**vcol 在 undo 里记一对值（旧+新）**，普通字段只记旧值。原因：回滚方向需要旧值恢复索引条目，purge/MVCC 方向需要知道"这个版本之后的 vcol 值"来重建二级索引记录，两者在 undo 链上都要可读。此外值写入前都按 `max_v_log_len = dict_max_v_field_len_store_undo()` 截断——undo 只为"更新索引记录"存必要字节（前缀索引只记前缀长度），不是无条件整值落盘。

### MVCC 版本链与 purge：DATA_MISSING 与回溯填充

MVCC 回放旧版本时 vcol 同样要重建。`row_vers_build_cur_vrow_low` 沿 undo 链回溯填充 vrow：

```cpp
static void row_vers_build_cur_vrow_low(
    bool in_purge, const rec_t *rec, dict_index_t *clust_index,
    ulint *clust_offsets, dict_index_t *index, roll_ptr_t roll_ptr,
    trx_id_t trx_id, mem_heap_t *v_heap, const dtuple_t **vrow, mtr_t *mtr) {
  ...
  *vrow = dtuple_create_with_vcol(v_heap, 0, num_v);
  dtuple_init_v_fld(*vrow);
  for (i = 0; i < num_v; i++)
    dfield_get_type(dtuple_get_nth_v_field(*vrow, i))->mtype = DATA_MISSING;

  version = rec;
  /* purge 线程加 TRX_UNDO_PREV_IN_PURGE，且都要求取出 old_v_val */
  const ulint status = in_purge
                           ? TRX_UNDO_PREV_IN_PURGE | TRX_UNDO_GET_OLD_V_VALUE
                           : TRX_UNDO_GET_OLD_V_VALUE;

  while (!all_filled) {
    ...row_get_rec_roll_ptr(version, ...)...
    /* 沿 undo 链回放一个版本；vrow 作为输出被逐渐填充 */
    trx_undo_prev_version_build(rec, mtr, version, clust_index, clust_offsets,
                                heap, &prev_version, nullptr, vrow, status,
                                nullptr);
    if (!prev_version) break;             // 版本链到头

    all_filled = true;
    for (i = 0; i < dict_index_get_n_fields(index); i++) {
      const dict_field_t *ind_field = index->get_field(i);
      if (!ind_field->col->is_virtual()) continue;
      const dict_v_col_t *v_col = ...;
      field = dtuple_get_nth_v_field(*vrow, v_col->v_pos);
      if (dfield_get_type(field)->mtype == DATA_MISSING) {
        all_filled = false;               // 还有 vcol 没拿到旧值
        break;
      }
    }

    /* 到达目标版本（trx_id 边界或 roll_ptr 匹配）即停 */
    trx_id_t rec_trx_id = row_get_rec_trx_id(prev_version, clust_index, clust_offsets);
    if (rec_trx_id < trx_id || roll_ptr == cur_roll_ptr) break;
    version = prev_version;
  }
  ...
}
```

要点：

- `TRX_UNDO_GET_OLD_V_VALUE` 是 `trx_undo_prev_version_build` 的状态位：回放时顺带把 undo 里存的 `old_v_val` 填进 vrow——vcol 旧值被设计进 undo 之后，版本链重建只需"按链回溯收集"，**不需要对每个历史版本重算表达式**。
- 填不满（链到头仍有 MISSING）是合法状态：调用方兜底走 `row_vers_build_sec_v_col` / `innobase_get_computed_value` 现场计算。

一致性校验侧，`row_vers_vc_matches_cluster` 把 undo 重建的 vcol 值与二级索引 entry 逐一比对：

```cpp
/* 先比非 vcol 列（含 PK）：row_vers_non_vc_index_entry_match */
...
while (n_cmp_v_col < n_fields - n_non_v_col) {      // 还有 vcol 没比完
  ...trx_undo_prev_version_build(..., vrow, status, ...)...
  for (i = 0; i < dict_index_get_n_fields(index); i++) {
    const dict_col_t *col = index->get_field(i)->col;
    if (!col->is_virtual()) continue;
    field1 = dtuple_get_nth_field(ientry, i);        // 索引 entry 里的物化值
    field2 = dtuple_get_nth_v_field(*vrow, v_col->v_pos);  // undo 重建值

    if (dfield_get_type(field2)->mtype != DATA_MISSING && !compare[v_col->v_pos]) {
      /* 前缀索引：undo 记 prefix_len*mbmaxlen 字节，实际索引记录可能更短，
         比较前把 field2 长度对齐到索引记录实际长度 */
      if (ind_field->prefix_len != 0 && !dfield_is_null(field2) &&
          field2->len > ind_field->prefix_len)
        field2->len = ind_field->prefix_len;
      ulint mbmax_len = DATA_MBMAXLEN(field2->type.mbminmaxlen);
      if (ind_field->prefix_len != 0 && !dfield_is_null(field2) && mbmax_len > 1)
        field2->len = field1->len;

      if (...cmp_dfield_dfield(field2, field1, ...) != 0) {
        return (false);                  // 不匹配：这条索引记录不属于这个版本
      }
      compare[v_col->v_pos] = true;
      n_cmp_v_col++;
    }
  }
  ...
}
```

这段代码解释了**前缀 vcol 索引的 undo 对齐问题**：undo 记的是截断到 `prefix_len * mbmaxlen` 字节的值，而索引记录实际长度按字符数截断，utf8mb4 下两者字节数不同，比较前必须把重建值对齐到索引记录的实际长度——这是字符集与 vcol 前缀索引叠加出的一个隐蔽细节。

purge 删旧索引条目时同理：`row_purge_remove_sec_if_poss` 沿 undo 拿到 `old_v_val` 构建旧 entry，若 undo 里没有（旧格式/被优化掉）则由 `row_vers` 现场重算。整个"undo 存 vcol 旧值"的设计闭环于此：**写路径多付一点 undo 空间，换 purge 不依赖 server 表达式求值即可删除旧版本索引记录**——后者在 purge 线程里做是昂贵且有锁序风险的。

### online DDL 与外键级联：另两条"拉"路径

**online DDL 建 vcol 索引**（`ddl0builder.cc` 的 `get_virtual_column`）：扫描聚簇行构建索引 entry 时，vcol 字段从行里拿不到（聚簇行不存），对每个 vcol 字段回调求值：

```cpp
// ddl0builder get_virtual_column（节选）
auto v_col = reinterpret_cast<const dict_v_col_t *>(col);
const auto clust_index = m_ctx.m_new_table->first_index();
auto key_buffer = m_thread_ctxs[ctx.m_thread_id]->m_key_buffer;
...
if (col->is_multi_value()) {
  src_field = dtuple_get_nth_v_field(ctx.m_row.m_ptr, v_col->v_pos);
  if (ctx.m_n_mv_rows_to_add == 0) {
    auto p = m_v_heap.get();
    src_field = innobase_get_computed_value(
        ctx.m_row.m_ptr, v_col, clust_index, &p, key_buffer->heap(), ifield,
        m_ctx.thd(), ctx.m_my_table, m_ctx.m_old_table, nullptr, nullptr);
    m_v_heap.reset(p);
    if (src_field == nullptr) { ...DB_COMPUTE_VALUE_FAILED; }
    ...
```

`m_ctx.m_old_table` 参数在此发挥作用：instant add vcol + 建索引的 DDL 里，`row_log_table_apply` 重放期间的旧行要用旧表的模板/字典求值，新行用新表。DDL 期间 vc_templ 被替换成扩展版（`handler0alter.cc` 里 `add_v` 把新 vcol 追加进 `s_templ`），新旧两套模板在同一个 builder 上下文里共存。

**外键级联**（`row_ins_foreign_fill_virtual`）：虽然虚拟列不能直接做外键列，但**基列被 FK 引用时**，父行 UPDATE CASCADE / DELETE SET NULL 会改子行基列，所有依赖这些基列的 vcol 索引条目必须跟着维护：

```cpp
static void row_ins_foreign_fill_virtual(upd_node_t *cascade, const rec_t *rec,
                                         dict_index_t *index, upd_node_t *node,
                                         dict_foreign_t *foreign, dberr_t *err) {
  ...
  ulint n_v_fld = index->table->n_v_def;
  ...
  update->old_vrow = row_build(ROW_COPY_POINTERS, index, rec, offsets,
                               index->table, nullptr, nullptr, &ext, update->heap);
  n_diff = update->n_fields;
  update->n_fields += n_v_fld;

  if (index->table->vc_templ == nullptr) {
    /* 级联发生在重启后也可能：vc_templ 惰性初始化 */
    innobase_init_vc_templ(index->table);
  }

  for (ulint i = 0; i < n_v_fld; i++) {
    dict_v_col_t *col = dict_table_get_nth_v_col(index->table, i);
    auto it = v_cols->find(col);                 // foreign->v_cols：基列被本 FK 引用的 vcol 集合
    if (it == v_cols->end()) continue;

    dfield_t *vfield = innobase_get_computed_value(
        update->old_vrow, col, index, &v_heap, update->heap, nullptr, thd,
        nullptr, nullptr, nullptr, nullptr);
    ...
    upd_field->old_v_val = ...dfield_copy(upd_field->old_v_val, vfield);
    upd_field_set_v_field_no(upd_field, i, index);

    if (node->is_delete ? (foreign->type & DICT_FOREIGN_ON_DELETE_SET_NULL)
                        : (foreign->type & DICT_FOREIGN_ON_UPDATE_SET_NULL)) {
      uint32_t col_match_count = dict_vcol_base_is_foreign_key(col, foreign);
      if (col_match_count == col->num_base) {
        /* 全部基列被 SET NULL → vcol 也为 NULL */
        dfield_set_null(&upd_field->new_val);
      } else if (col_match_count == 0) {
        /* 无基列受影响 → 新值=旧值 */
        dfield_copy(&(upd_field->new_val), vfield);
      } else {
        /* 部分基列被 SET NULL → 用 update vector 混合基列值重算 */
        for (uint32_t j = 0; j < col->num_base; j++) {
          dict_col_t *base_col = col->base_col[j];
          dfield_t *row_field = innobase_get_field_from_update_vector(
              foreign, node->update, base_col->ind);
          if (row_field != nullptr) dfield_set_null(row_field);
        }
        dfield_t *new_vfield = innobase_get_computed_value(
            update->old_vrow, col, index, &v_heap, update->heap, nullptr, thd,
            nullptr, nullptr, node->update, foreign);   // parent_update 生效
        dfield_copy(&(upd_field->new_val), new_vfield);
      }
    }

    if (!node->is_delete && (foreign->type & DICT_FOREIGN_ON_UPDATE_CASCADE)) {
      /* CASCADE：基列全改成父行新值后重算 */
      dfield_t *new_vfield = innobase_get_computed_value(
          update->old_vrow, col, index, &v_heap, update->heap, nullptr, thd,
          nullptr, nullptr, node->update, foreign);
      ...
    }
    n_diff++;
  }
  update->n_fields = n_diff;
}
```

这里能看到 `parent_update`/`foreign` 两个参数的全部用途：级联场景下基列的"新值"不在 `old_vrow` 里而在**父行 update vector**里（`innobase_get_field_from_update_vector` 按基列号取），回调前先把受影响基列置 NULL、再让 server 用"update vector 覆盖后的行"求值。SET NULL 的三分叉（全部/部分/无关基列）说明外键语义与生成列语义的组合被精确枚举了。

**索引统计信息**无需特殊机制：`dict_stats_analyze_index` 采样二级索引时 vcol 值就在索引记录里，统计与普通索引相同——这也是物化方案相比"纯虚拟索引"的又一个免费收益。

### binlog 与复制：不对称的列镜像

主库 `binlog_log_row` 用 `table->write_set` 打包行镜像，虚拟生成列在三种事件中待遇不同：

- **INSERT（WRITE_ROWS_EVENT）**：vcol 在镜像里（write_set 含它）。代码注释明说 "all generated columns are required to be written into the binlog"——非 NDB 引擎总是把生成列打进 insert 事件。**虚拟列也打**：主库算好直接给，备库不必重算。
- **UPDATE/DELETE**：`binlog_prepare_row_images` 链（`mark_columns_per_binlog_row_image`）按 `binlog_row_image` 收缩 read_set，执行期为维护索引而夹带的 vcol 会被清出镜像（`binlog_prepare_row_images` 注释：spurious columns 由 `mark_columns_needed_for_update` 与它共同清理）。
- **功能索引隐藏列**：永不出镜。`pack_row` 用 `ReplicatedColumnsView` + `outbound_func_index` 过滤器，`bitmap_intersect(pack_row_tmp_set, 过滤后的位图)`——隐藏生成列（`HT_HIDDEN_SQL`）不序列化进事件。GIPK 列在源/备不对称时同样有 `inbound_gipk`/`outbound_gipk` 过滤。

备库 apply（`Rows_log_event::do_apply_row`）把位图反转：`bitmap_set_all(write_set)` 后 `mark_generated_columns(true)` 把要重算的 gcol 标出来。UPDATE 事件里没有 vcol 列（主库没打），apply 前 `Rows_log_event::update_generated_columns` 按规则补算：

```cpp
Table_columns_view<> updatable_columns_view{
    this->m_table,
    [=](TABLE const *table, size_t column_index) -> bool {
      auto field = table->field[column_index];
      if (field->is_field_for_functional_index())  return true;  // 功能索引列
      if (!is_after_image &&
          !bitmap_is_subset(&field->gcol_info->base_columns_map,
                            &this->m_local_cols))   // 基列没变的：不重算
        return true;
      if (!is_after_image) return true;             // before-image 不算
      return column_index < this->m_cols.n_bits ||  // 源端没有这列
             field->is_virtual_gcol();              // 或虚拟列 → 要算
    },
    Table_columns_view<>::VFIELDS_ONLY};
if (updatable_columns_view.filtered_size() != 0 &&
    this->update_generated_columns(updatable_columns_view.get_included_fields_bitmap()))
  return HA_ERR_OUT_OF_MEM;
```

规则可概括为：**after-image 里没有、但备库执行需要的列**——虚拟列、只存在于备库的生成列、基列变了但主库未打包的 stored 列——由备库重算；主库已打包的列（INSERT 的 vcol、镜像里的 stored 列）直接用。主备表结构可能不同（备库多/少生成列），这套"按本地字段补算"的设计让 RBR 复制对生成列天然容忍结构差异。SBR 路径则靠 `m_backup_binlog_stmt_flags`：解析生成列表达式时备份 unsafe 标志，执行语句时合并，防止非确定表达式在语句复制下漂移。

### DDL 变更：ADD/DROP/MODIFY 与 instant

`fill_alter_inplace_info` 的列比对把生成列变更细分为独立 flag：`ADD_VIRTUAL_COLUMN` / `DROP_VIRTUAL_COLUMN` / `ALTER_VIRTUAL_GCOL_EXPR` / `ALTER_STORED_GCOL_EXPR` / `ALTER_VIRTUAL_COLUMN_TYPE` / `ALTER_STORED_COLUMN_TYPE` / `ALTER_VIRTUAL_COLUMN_ORDER`（virtual↔stored 之间切换走 `ALTER_STORED_COLUMN_TYPE`，代码注释明确"Modification of storage attribute is not supported"且断言 `is_virtual_gcol()` 不变——即 VIRTUAL 改 STORED 不被支持，需重建）。

这些 flag 在 InnoDB 的 inplace 决策中被分到不同集合，构成生成列 DDL 的"能力阶梯"：

```cpp
/* 决策中被忽略（不改存储）的变更 */
static const HA_ALTER_FLAGS INNOBASE_INPLACE_IGNORE =
    ALTER_COLUMN_DEFAULT | ALTER_COLUMN_COLUMN_FORMAT |
    ALTER_COLUMN_STORAGE_TYPE | ALTER_RENAME | CHANGE_INDEX_OPTION |
    ADD_CHECK_CONSTRAINT | DROP_CHECK_CONSTRAINT | SUSPEND_CHECK_CONSTRAINT |
    ALTER_COLUMN_VISIBILITY;

/* INSTANT 允许的操作 */
static const HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    ALTER_COLUMN_NAME | ADD_VIRTUAL_COLUMN | DROP_VIRTUAL_COLUMN |
    ALTER_VIRTUAL_COLUMN_ORDER | ADD_STORED_BASE_COLUMN |
    ALTER_STORED_COLUMN_ORDER | DROP_STORED_COLUMN;

/* 不重建表即可 inplace 的操作 */
static const HA_ALTER_FLAGS INNOBASE_ALTER_NOREBUILD =
    ... | DROP_INDEX | ... | ALTER_COLUMN_NAME | ALTER_COLUMN_EQUAL_PACK_LENGTH |
    ADD_VIRTUAL_COLUMN | DROP_VIRTUAL_COLUMN | ALTER_VIRTUAL_COLUMN_ORDER |
    ALTER_COLUMN_INDEX_LENGTH;
```

- **ADD/DROP 虚拟列 → instant**（`INNOBASE_INSTANT_ALLOWED`）：`innobase_support_instant` 判定为 `INSTANT_VIRTUAL_ONLY`（纯虚拟列增删，可叠加列重命名）——虚拟列不占聚簇行空间，增删不触碰任何行数据，只改字典与 DD。
- **`ALTER_VIRTUAL_COLUMN_ORDER`（虚拟列顺序调整）也可 instant/norebuild**：vcol 顺序只影响 `v_pos` 编号与求值拓扑（生成列只能引用先定义列），不涉及物理行。
- **修改生成列表达式（`ALTER_VIRTUAL_GCOL_EXPR`）不在任何快速集合** → 回退 COPY 全表重建，逐行重算新表达式。
- **默认值变化连带的重算**（`VIRTUAL_GCOL_REEVAL` / `STORED_GCOL_REEVAL`，由 `fill_alter_inplace_info` 里 `gcols_with_unchanged_expr` 遍历 + `check_gcol_depend_default_processor` walk 表达式找依赖后置位）同样脱离快速路径，走表重建。

instant add 虚拟列同时建 vcol 索引的路径里，`handler0alter.cc` 的 `add_v`（`dict_add_v_col_t`）把新 vcol 传给 `innobase_build_v_templ` 扩展 vc_templ，`row_log_table_apply` 重放期间用旧模板求值旧列、新模板求值新列——这是 `innobase_get_computed_value` 的 `old_table` 参数存在的意义（`ddl0builder` 的 `m_ctx.m_old_table` 同理）。

其余约束：删除生成列的基列被拒（`ER_DEPENDENT_BY_GENERATED_COLUMN`）；`EXCHANGE PARTITION` 校验两侧表 `is_virtual_gcol()` 与 `NOT_NULL_FLAG` 完全一致（`ER_UNSUPPORTED_ACTION_ON_GENERATED_COLUMN`）；分区表的分区键是生成列时，列本身不可删/改。

### 隐藏生成列家族：一个模板的三种复用

| 特性 | 隐藏列 | 实现要点 |
|---|---|---|
| 功能索引 `INDEX((expr))` | `HT_HIDDEN_SQL` 虚拟列 | 列名自动生成（含非法字符，需反引号转义打印）；不能直接引用，表达式需"重写"成引用隐藏列的形式 |
| 多值索引 `CAST(j AS UNSIGNED ARRAY)` | typed-array 隐藏虚拟列 | 一个 JSON 数组一行多值，索引记录一条多条目；`dfield` 带 `DATA_MULTI_VALUE` 标志 |
| GIPK | 隐藏虚拟列 `my_row_id` + 隐藏索引 | 行无主键时的兜底主键；binlog 跨版本时按需过滤（`is_gipk_present_on_source_table`） |

它们共用前面全部机制：打开时重解析表达式、vfield 求值、vc_templ 回调、binlog 过滤——差异只在**可见性**（`dd::Column::enum_hidden_type`）与**表达式的写入口**（功能索引由 `Add_drop_index` 的 key part 表达式直接生成 `Create_field`，见 `sql_table.cc` 里 `gcol_info->expr_item = kp->get_expression()` 的路径）。

### 系统表与视图：生成列如何对外可见

**DD 存两份表达式文本。** `mysql.columns` 表里生成列占两列：`generation_expression`（binary，建表时的原始字符集）与 `generation_expression_utf8`（`dd_table.cc` 在 `fill_dd_column_from_create_field` 里用 `convert_and_print` 转成 system charset 的副本，注释明说 "Prepare UTF expression for IS"）。后者是 information_schema 的数据源；`is_virtual` 位存 VIRTUAL/STORED。注意 DD 里存的是 **`print_expr` 规范化后的打印文本**，不是用户原始书写——`SHOW CREATE TABLE` 显示的表达式可能与建表语句字面不同。

**I_S.COLUMNS 的两个关键列**（`sql/dd/impl/system_views/columns.cc` 的视图定义）：

```sql
-- GENERATION_EXPRESSION：直接读 DD 的 utf8 副本
IFNULL(col.generation_expression_utf8, '')

-- EXTRA：由内部函数按 DD 位拼装
INTERNAL_GET_DD_COLUMN_EXTRA(ISNULL(col.generation_expression_utf8),
  col.is_virtual, col.is_auto_increment, col.update_option,
  IF(LENGTH(col.default_option), TRUE, FALSE), col.options, col.hidden, tbl.type)
```

`INTERNAL_GET_DD_COLUMN_EXTRA` 是 I_S 视图专用的内部函数（`Item_func_internal_get_dd_column_extra`），核心分支一行定音：

```cpp
if (is_not_generated_column) {
  ... // DEFAULT_GENERATED / on update / auto_increment
} else {
  oss << (is_virtual ? "VIRTUAL GENERATED" : "STORED GENERATED");   // ← 这里
}
```

所以用户看到的 `EXTRA = "VIRTUAL GENERATED"` 不是存储值，而是**打开 I_S 时从 DD 的 `is_virtual` 位实时计算**出来的——生成列属性在系统视图层没有冗余存储。

**I_S.STATISTICS 的 EXPRESSION 列**（功能索引专用）：

```sql
-- 只对功能索引的隐藏列（hidden='SQL'）展示表达式，普通列/普通 vcol 索引为 NULL
IF (col.hidden = 'SQL', col.generation_expression_utf8, NULL)
```

功能索引在 `STATISTICS` 里以隐藏列 + `EXPRESSION` 列展示表达式，而不是把表达式塞进 `COLUMN_NAME`。

**SHOW CREATE TABLE / SHOW COLUMNS 的打印路径**（`sql_show.cc`）：

```cpp
if (field->gcol_info) {
  packet->append(STRING_WITH_LEN(" GENERATED ALWAYS"));
  packet->append(STRING_WITH_LEN(" AS ("));
  field->gcol_info->print_expr(thd, &s);      // Item 树 → SQL 文本
  packet->append(s);
  packet->append(STRING_WITH_LEN(")"));
  if (field->stored_in_db)
    packet->append(STRING_WITH_LEN(" STORED"));
  else
    packet->append(STRING_WITH_LEN(" VIRTUAL"));
}
```

`print_expr` 把 Item 树**反向打印**成 SQL 文本——这是生成列表达式从"文本 → Item 树"之外的第三条转换路径（正向解析 `PARSE_GCOL_EXPR`、DD 落盘 `fill_dd_column_from_create_field`、展示打印 `print_expr`）。功能索引的 key part 打印时特判 `is_field_for_functional_index()`，输出 `((expr))` 而非列名（2255 行分支）。`SHOW COLUMNS` 实际读 I_S 临时表，`get_schema_column_record` 里同样经 `print_expr` 填 `COLUMN_GENERATION_EXPRESSION` 字段。

**InnoDB 侧的字典内省**：`I_S.INNODB_COLUMNS` **包含虚拟列行**——`i_s_dict_fill_innodb_columns` 接收 `nth_v_col` 参数，遍历 `dict_table_t::v_cols` 链把 vcol 与物理列一并列出（`has_virtual_cols` 分支）。此外还有专门的 `I_S.INNODB_VIRTUAL` 视图（插件 `i_s_innodb_virtual`）：字段 `TABLE_ID / POS / BASE_POS`，从 `dd_columns` 系统表（`SYS_COLUMNS` 记录里存的 vcol-基列映射）逐对输出——**这是 InnoDB 字典里 vcol 依赖关系（`dict_v_col_t::base_col[]`）的对外窗口**，一行一个 (vcol, base_col) 对。观测生成列的入口：语义层看 `I_S.COLUMNS`（EXTRA/GENERATION_EXPRESSION）+ `I_S.STATISTICS`（EXPRESSION），引擎字典层看 `I_S.INNODB_VIRTUAL`。

---

## Misc

### 虚拟列上的限制清单

| 限制 | 出处 |
|---|---|
| 生成列表达式不能是 ROW 值、不能含子查询/存储函数/命名函数白名单外函数、不能引用变量 | `pre_validate_value_generator_expr` / `validate_value_generator_expr` |
| 非确定性函数（`NOW()`、`RAND()` 等）禁止；UDF/存储函数一律禁用（WL#411 原文："我们无法信任它们声明的 deterministic 属性"） | `validate_value_generator_expr` 的 `is_non_deterministic()` 分支 |
| 只能引用先于自己定义的列（含更早的生成列）；自引用/前向引用报 `ER_GENERATED_COLUMN_NON_PRIOR` | vfield 升序 = 求值拓扑序 |
| 表达式长度上限 64K（WL#411 从补丁的 255 字节放宽，容纳 JSON/GIS 长表达式） | DDL 校验 |
| 不能有显式 DEFAULT、不能是 AUTO_INCREMENT | DDL 校验 |
| 虚拟列不能做外键列 | `ER_FK_CANNOT_USE_VIRTUAL_COLUMN` |
| 虚拟列上不能建 FULLTEXT / SPATIAL 索引 | `ER_FULLTEXT_FUNCTIONAL_INDEX` / `ER_SPATIAL_FUNCTIONAL_INDEX` |
| INSERT 不能给生成列赋值（`DEFAULT` 除外） | `ER_NON_DEFAULT_VALUE_FOR_GENERATED_COLUMN` |
| VIRTUAL ↔ STORED 属性切换需表重建 | `fill_alter_inplace_info` 断言 |
| 生成列做分区键时列本身不可 DROP/MODIFY | gcol 依赖检查 |
| `CREATE TABLE ... LIKE` 总是包含生成列（不支持标准 F385 的 `EXCLUDING GENERATED` / DROP EXPRESSION 降级为基列） | WL#411 FUTURE PLAN |

### 概念澄清

- **生成列的 "VIRTUAL" 与 InnoDB 页格式的 "virtual" 无关**：前者指"聚簇记录中不物理存储"（DD 里 `is_virtual` 位）；后者在引擎代码里泛指"非物理列"这一属性（`dict_col_t::is_virtual()`）。vcol 在二级索引中是物理存在的。
- **`vfield` 不是"虚拟列数组"**：它是**全部生成列**（virtual + stored）的数组。`is_virtual_gcol()` = `gcol_info && !stored_in_db` 才区分。
- **5.7 的 `vcol_set` 已不存在**：8.0 用 `vfield` 数组 + `Value_generator::base_columns_map` 取代了 5.7 的位图式 `TABLE::vcol_set`。读 5.7 时代资料时注意。

---

## 参考

**社区文章**
- [MySQL 源码阅读 — virtual generated column — 西西弗斯](https://zhuanlan.zhihu.com/p/649144461)

**官方文档**
- *MySQL 8.0 Reference Manual → Data Types → CREATE TABLE and Generated Columns*
- *MySQL 8.0 Reference Manual → CREATE TABLE ... Indexes and Functional Key Parts*
- WorkLog: [WL#411 Generated columns](https://dev.mysql.com/worklog/task/?id=411)（5.7 引入；本篇「理论基础」的语法选型、三动机、人为限制均取自其原文与附录 B）
- WorkLog: [WL#8114 InnoDB: Store virtual columns in the clustered index? / skip virtual columns](https://dev.mysql.com/worklog/task/?id=8114)（InnoDB 完全跳过虚拟列存储）
- WorkLog: [WL#1075 Functional indexes](https://dev.mysql.com/worklog/task/?id=1075)（WL#411 预留的目标）、[WL#10761](https://dev.mysql.com/worklog/task/?id=10761)（实现）
- WorkLog: [WL#8763 Multi-valued index](https://dev.mysql.com/worklog/task/?id=8763)、[WL#13601 Invisible primary keys](https://dev.mysql.com/worklog/task/?id=13601)

**内核月报 / 技术文章**
- 阿里内核月报：*[MySQL · 特性分析 · 生成列与索引](http://mysql.taobao.org/monthly/)*（5.7 视角，函数名与 8.0.39 差异大，机制以本篇源码核实为准）

**相关文档**
- 表达式树（`Item`、`fix_fields`、`save_in_field`）：[`../server/query/13_item_expression.md`](../server/query/13_item_expression.md)
- JSON 上的索引方案（生成列 8.1 / 功能索引 8.2 / 多值索引 8.3，与本篇互为补充）：[`../server/datatype/json.md`](../server/datatype/json.md)
- DDL 框架（instant DDL、row log）：[`../innodb/ddl.md`](../innodb/ddl.md)
- binlog 行事件与复制：[`../server/replication/binlog.md`](../server/replication/binlog.md)
- handler 接口与位图协议（read_set/write_set/covering）：[`../server/handler.md`](../server/handler.md)
- 同类"DD→文本→重解析"往返：分区信息见 [`partitioning.md`](partitioning.md)

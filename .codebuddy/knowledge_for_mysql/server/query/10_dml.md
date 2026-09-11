# 10 DML 执行链路（INSERT / UPDATE / DELETE）

> 基于 MySQL 8.0.39 源码。本篇讲 **DML（INSERT/UPDATE/DELETE）与 SELECT 的差异**——SELECT 是主链（02~09 篇已讲透），这里只讲 DML 不同的地方，共用部分不重复。
>
> **边界**：协议 / 解析 / contextualize 见 01~04；优化器见 [`07_optimize/`](07_optimize/README.md)；handler 接口与 `rnd_pos` 见 [`../handler.md`](../handler.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先厘清概念：DQL 就是主链本身](#先厘清概念dql-就是主链本身)
- [一、类体系与入口分流](#一类体系与入口分流)
- [二、INSERT：不走迭代器的一条链](#二insert不走迭代器的一条链)
- [三、UPDATE：单表快路径 vs 多表](#三update单表快路径-vs-多表)
- [四、DELETE：单表快路径 vs 多表](#四delete单表快路径-vs-多表)
- [五、buffer row id 两阶段读](#五buffer-row-id-两阶段读)
- [六、与 SELECT 在 prepare / optimize 的差异](#六与-select-在-prepare--optimize-的差异)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)

---

## 设计思想与理论基础

### Halloween Problem：DML 与 SELECT 最本质的差别

DML 和 SELECT 的执行模型差异，根源是一个经典理论问题——**Halloween Problem（万圣节问题）**，得名于 IBM 的研究者在万圣节那天发现它。

问题本身：`UPDATE t SET c = c + 1 WHERE c > 10`，如果用索引 `(c)` 一边扫描一边更新，被更新的行**可能移动到扫描游标还没扫到的位置**，于是被再次扫到、再次更新——最坏情况无限循环。

```
扫描索引 c=10 → 读到行 → 更新 c=11 → 行在索引里"前移"到游标前方
              → 游标继续向后扫 → 又读到这行（现在 c=11）→ 又更新 c=12 → ...
```

**SELECT 没有这个问题**——它只读不写，游标位置不受影响。所以 SELECT 可以直接"扫到哪算哪"，而 DML 必须额外处理"更新是否会改变扫描键"。

### 两种解法，对应 UPDATE 的两条路径

MySQL 的应对正是本篇第三、五节讲的两条路径，它们对应两种思路：

| 思路 | MySQL 实现 | 代价 |
|---|---|---|
| **safe update on fly（安全就地更新）** | 判定更新**不改变**扫描键/索引键时，直接 `ha_update_row`，边扫边改 | 零额外代价，但要精确判定"安全" |
| **buffer row id 两阶段读** | 先扫出所有目标行的 row id 存进临时表/`Unique`，扫完再回表 `rnd_pos` 逐个更新 | 多一趟扫描 + 缓冲，但绝对安全 |

关键洞察（`safe_update_on_fly` 的判定）：只有**更新会改变"决定扫描顺序的键"**时才有万圣节问题。若 `WHERE` 用的索引和 `SET` 改的列不相干，就地更新是安全的——`UPDATE t SET non_key_col = 1 WHERE key_col = 5` 不改变 `key_col`，游标不会乱。

### 为什么 INSERT 干脆不走迭代器

INSERT 与 SELECT 差异更大：它**没有"读一个结果集"这个动作**，而是逐行 `write_record` 落库（第二节）。所以 INSERT 不套火山模型（09 篇），是一条独立的 `Sql_cmd_insert_values::execute_inner` 链。这是"执行模型按操作类型分流"的体现——**读用迭代器、写走专用路径**。

---

## 先厘清概念：DQL 就是主链本身

**为什么 10 篇只讲 DML 不讲 SELECT**——因为 SELECT（DQL）**不是没讲，它就是主链**：

```
04_parser → 05_contextualize → 05_prepare → 07_optimize → 08_access_path → 09_executor_iterator
        ↑ 这条链从头到尾讲的就是 SELECT
```

`10_dml.md` 的定位**不是"补一篇 DML"**，而是"**记录 DML 与 SELECT 不一样的地方**"。INSERT/UPDATE/DELETE 是主链上走岔路的分支，需要单独说明它们岔在哪：

| 主题 | 在哪讲 |
|------|--------|
| 协议 / 解析 / contextualize | 01 ~ 03（DML 与 SELECT 完全相同） |
| prepare 阶段 | 04（共用）+ 本篇第六节（DML 多出来的部分） |
| 优化器 | 07_optimize/（DML 复用，但目标表有额外约束，见本篇第六节） |
| **执行阶段** | **本篇（这是 DML 与 SELECT 差异最大的地方）** |

**术语口径**：SQL 标准里 DML 其实包含 SELECT，但 MySQL 社区习惯把 DML 特指 INSERT/UPDATE/DELETE——这恰好与源码一致：`Sql_cmd_dml` 是 INSERT/UPDATE/DELETE 的基类，而 `Sql_cmd_select` 也继承 `Sql_cmd_dml` 但重写 `is_data_change_stmt()` 返回 `false`。本篇沿用这个口径。

---

## 一、类体系与入口分流

### 1.1 继承树

```
Sql_cmd
└── Sql_cmd_dml                    (sql/sql_cmd_dml.h:35)
    ├── Sql_cmd_select            (sql/sql_select.h:74)   ← SELECT 也继承它！
    ├── Sql_cmd_insert_base       (sql/sql_insert.h:223)
    │   ├── Sql_cmd_insert_values (sql/sql_insert.h:314)
    │   └── Sql_cmd_insert_select (sql/sql_insert.h:334)
    ├── Sql_cmd_update            (sql/sql_update.h:126)
    └── Sql_cmd_delete            (sql/sql_delete.h:38)
```

关键点：**`Sql_cmd_select` 继承 `Sql_cmd_dml`**（`sql_select.h:74`），但：

```cpp
// sql/sql_select.h:74-93
class Sql_cmd_select : public Sql_cmd_dml {
  bool is_data_change_stmt() const override { return false; }  // :82
  bool may_use_cursor() const override { return true; }        // :89
};
```

所以 DML 与 SELECT **共享同一套 `execute()` 六阶段骨架**（`precheck → open_tables → prepare → optimize → execute_inner`），差异只在两个钩子：`prepare_inner`（命令特有 prepare）和 `execute_inner`（命令特有执行），以及 `is_data_change_stmt()` 驱动的数据变更行为（IGNORE 错误处理器、`run_before_dml_hook` 等）。

### 1.2 对象在哪 new（纠正一个常见误解）

**`Sql_cmd_*` 对象不是 `mysql_execute_command` 里 new 的，而是在 parser 阶段**（`PT_*::make_cmd()`）：

```cpp
// sql/parse_tree_nodes.cc:917    PT_delete::make_cmd
return new (thd->mem_root) Sql_cmd_delete(is_multitable(), &delete_tables);

// sql/parse_tree_nodes.cc:972    PT_update::make_cmd
return new (thd->mem_root) Sql_cmd_update(is_multitable, &value_list->value);

// sql/parse_tree_nodes.cc:1126-1132  PT_insert::make_cmd
Sql_cmd_insert_base *sql_cmd;
if (has_query_block())  // 有 SELECT/UNION 右侧 → insert_select
  sql_cmd = new (thd->mem_root) Sql_cmd_insert_select(is_replace, lex->duplicates);
else                    // 纯 VALUES → insert_values
  sql_cmd = new (thd->mem_root) Sql_cmd_insert_values(is_replace, lex->duplicates);
```

`mysql_execute_command`（`sql_parse.cc:3676`）对 `SQLCOM_INSERT/UPDATE/DELETE` 只做一行：

```cpp
res = lex->m_sql_cmd->execute(thd);   // 统一入口，靠虚函数分派
```

### 1.3 为什么 INSERT VALUES 和 INSERT SELECT 是两个类

`PT_insert::make_cmd` 用 `has_query_block()` 在**语法解析阶段**就区分：

- **`Sql_cmd_insert_values`**（`is_single_table_plan()==true`）：无 SELECT 迭代器，`execute_inner` 自己写 `for (values : insert_many_values)` 循环逐行 `write_record`。数据源是内存里的 `insert_many_values`（二维数组），不生成 JOIN / AccessPath。
- **`Sql_cmd_insert_select`**：不重写 `execute_inner`，走基类默认实现——右侧 SELECT 走**正常迭代器**，`Query_result_insert::send_data` 逐行把 SELECT 结果写进目标表。

各 DML 类的 `sql_command_code()`：

| 类 | 返回 |
|---|---|
| `Sql_cmd_insert_values` | `is_replace ? SQLCOM_REPLACE : SQLCOM_INSERT` |
| `Sql_cmd_insert_select` | `is_replace ? SQLCOM_REPLACE_SELECT : SQLCOM_INSERT_SELECT` |
| `Sql_cmd_update` | `multitable ? SQLCOM_UPDATE_MULTI : SQLCOM_UPDATE` |
| `Sql_cmd_delete` | `multitable ? SQLCOM_DELETE_MULTI : SQLCOM_DELETE` |

---

## 二、INSERT：不走迭代器的一条链

### 2.1 `Sql_cmd_insert_values::execute_inner` 主循环（sql_insert.cc:477）

核心逐行循环在 `:576-637`：

```cpp
for (const List_item *values : insert_many_values) {
  restore_record(insert_table, s->default_values);          // 恢复默认值模板
  if (validate_default_values_of_unset_fields(thd, insert_table)) { has_error = true; break; }
  if (fill_record_n_invoke_before_triggers(                 // 求值 + BEFORE INSERT 触发器
          thd, &info, insert_field_list, *values, insert_table,
          TRG_EVENT_INSERT, insert_table->s->fields, true, nullptr)) {
    has_error = true; break;
  }
  if (check_that_all_fields_are_given_values(thd, insert_table, table_list)) {
    has_error = true; break;                                // 无默认值 → ER_NO_DEFAULT_FOR_FIELD
  }
  const int check_result = table_list->view_check_option(thd);
  if (check_result == VIEW_CHECK_SKIP) continue;
  else if (check_result == VIEW_CHECK_ERROR) { has_error = true; break; }
  if (invoke_table_check_constraints(thd, insert_table)) {  // CHECK 约束
    if (thd->is_error()) { has_error = true; break; }
    continue;                                               // IGNORE 降级为 warning，跳过
  }
  if (write_record(thd, insert_table, &info, &update)) { has_error = true; break; }
  thd->get_stmt_da()->inc_current_row_for_condition();
}
```

逐段解释：

1. **循环体 = 一个 VALUES 元组 → 一行**：`insert_many_values` 是 `mem_root_deque<List_item *>`，`INSERT ... VALUES (..),(..),(..)` 就是三次迭代。
2. `restore_record(table, s->default_values)`：把 `record[0]` 恢复成默认值模板，避免上一行值残留。
3. `fill_record_n_invoke_before_triggers`：求值 VALUES 表达式写进 `record[0]`，**并在此触发 BEFORE INSERT 触发器**。
4. `check_that_all_fields_are_given_values`：检查 `write_set` 里未被赋值的字段是否有默认值/可 NULL。
5. `write_record`：真正落库（见 2.2）。
6. `has_error` 一旦置位立刻 `break`——多行 VALUES 是"**一旦出错即停止**"，但已插入的行是否回滚由外层 `trans_rollback_stmt` 决定（见 6.6）。

### 2.2 `write_record` 与三种重复键策略（sql_insert.cc:1793）

`write_record` 的职责是**重复键处理 + 实际写行 + AFTER 触发器**（auto_increment 在 `ha_write_row` 内部由 `update_auto_increment` 生成，check constraint / trigger 由调用方做）。

三种策略的核心分叉：

```cpp
// sql_insert.cc:1810-1816
const enum_duplicates duplicate_handling = info->get_duplicate_handling();
if (duplicate_handling == DUP_REPLACE || duplicate_handling == DUP_UPDATE) {
  while ((error = table->file->ha_write_row(table->record[0]))) {  // 循环处理 dupp key
    ...
    is_duplicate_key_error =
        (error == HA_ERR_FOUND_DUPP_KEY || error == HA_ERR_FOUND_DUPP_UNIQUE);
    ...
```

| 策略 | 处理 |
|------|------|
| 普通 INSERT（`DUP_ERROR`） | `ha_write_row` 报错直接 `print_error` → `before_trg_err`（回滚 auto_increment） |
| **INSERT IGNORE** | 执行器压入 `Ignore_error_handler`（`sql_select.cc:704`）把错误降级为 warning；同时 `HA_EXTRA_IGNORE_DUP_KEY` 让引擎底层直接吞掉 dupp key |
| **ODKU / REPLACE** | 进 while 循环，捕获 dupp key 后转 UPDATE（ODKU）或 delete+insert（REPLACE），见 2.3 |

### 2.3 ON DUPLICATE KEY UPDATE：dupp key → ha_update_row（sql_insert.cc:1917）

```cpp
if (duplicate_handling == DUP_UPDATE) {
  store_record(table, insert_values);              // 保存"新行"，供 VALUES(col) 引用
  restore_record(table, record[1]);                // record[1] = 已存在的旧行
  if (fill_record_n_invoke_before_triggers(        // 在旧行基础上套 UPDATE 列表
          thd, update, *update->get_changed_columns(),
          *update->update_values, table, TRG_EVENT_UPDATE, 0, true, &is_row_changed))
    goto before_trg_err;
  ...
  if ((error = table->file->ha_update_row(table->record[1], table->record[0])) &&
      error != HA_ERR_RECORD_IS_THE_SAME) { ... }   // 旧 record[1] → 新 record[0]
}
```

关键步骤：

1. `ha_write_row` 撞 dupp key → `get_dup_key(error)` 拿冲突 key 号
2. `key_copy` + `ha_index_read_idx_map(HA_READ_KEY_EXACT)` 把旧行读进 `record[1]`（引擎支持 `HA_DUPLICATE_POS` 则直接 `ha_rnd_pos`）
3. `store_record(insert_values)` 存新行（ODKU 的 `VALUES(col)` 引用它），`restore_record(record[1])` 恢复旧行
4. `fill_record_n_invoke_before_triggers(TRG_EVENT_UPDATE)` 把 UPDATE 列表套到 `record[0]`（新值）
5. `ha_update_row(旧 record[1], 新 record[0])` —— 这就是"转 UPDATE"

`HA_ERR_RECORD_IS_THE_SAME` 表示新旧相同（no-op），不算 updated。

**REPLACE**（`:2057`）语义是 **DELETE 旧行 + 重插**：除非满足"最后唯一键 + 未被外键引用 + 无 DELETE 触发器"才转 `ha_update_row` 优化，否则先 `ha_delete_row`（触发 DELETE 触发器）再回到 while 循环重插——因为 REPLACE 要正确触发外键级联，转 UPDATE 会破坏语义。

### 2.4 INSERT ... SELECT：`Query_result_insert::send_data`（sql_insert.cc:2344）

```cpp
bool Query_result_insert::send_data(THD *thd, const mem_root_deque<Item *> &values) {
  store_values(thd, values);                     // SELECT 列 → record[0]
  if (table_list) {
    switch (table_list->view_check_option(thd)) { ... }
  }
  if (invoke_table_check_constraints(thd, table)) return thd->is_error();
  error = write_record(thd, table, &info, &update);   // 与 VALUES 路径共用
  if (!error && (table->triggers || ...)) {
    restore_record(table, s->default_values);         // 触发器/ODKU 改过字段，还原
  }
  return error;
}
```

`send_data` 是 SELECT 迭代器**每产出一行就回调一次**的地方。它与 VALUES 路径共用 `write_record`，差异只在：SELECT 每行写入后要 `restore_record` 还原 `record[0]` 里未参与 INSERT 的列。

右侧 SELECT **有独立的 AccessPath 和迭代器**：`Query_result_insert` 被挂为整个 `Query_expression` 的 query result，SELECT 的 `send_data` 被替换成 `Query_result_insert::send_data`——"读 SELECT 行"与"写 insert 表"在同一迭代器流水线里逐行串起来，**中间没有物化**（除非目标表也出现在 SELECT 里触发 `OPTION_BUFFER_RESULT`）。

---

## 三、UPDATE：单表快路径 vs 多表

### 3.1 判定（sql_update.cc:1804）

```cpp
bool Sql_cmd_update::execute_inner(THD *thd) {
  thd->table_map_for_update = tables_for_update;
  if (is_empty_query()) { ...; my_ok(thd); return false; }
  return multitable ? Sql_cmd_dml::execute_inner(thd)  // 多表：走迭代器
                    : update_single_table(thd);         // 单表：快路径
}
```

`multitable` 可能被 prepare 阶段改掉（`sql_update.cc:1543/1551/1557`）：
- 语法单表但目标是**多表视图** → `multitable = true`
- 使用 **hypergraph 优化器** → 强制 `multitable = true`（统一执行路径）
- 单表但 WHERE 有相关子查询满足 `should_switch_to_multi_table_if_subqueries` → `multitable = true`

**单表快路径 `update_single_table`（:367）**：`test_quick_select` 选 range scan、`get_index_for_order` 处理 ORDER BY、`prune_partitions` 分区裁剪，然后 `while (iterator->Read())` 逐行 `ha_update_row`。**关键：它不生成 `AccessPath::UPDATE_ROWS`**，直接 `init_table_iterator` 扫表。

### 3.2 `UpdateRowsIterator` 两阶段（sql_update.cc:2864 / 2418 / 2584）

`Read()` 是"**一次调用跑完整条语句**"：

```cpp
int UpdateRowsIterator::Read() {
  // 阶段一：消费 join 所有行
  while (!local_error) {
    const int read_error = m_source->Read();
    if (read_error < 0) break;                        // EOF
    local_error = DoImmediateUpdatesAndBufferRowIds(...);
  }
  // 阶段二：执行延迟更新
  if (!local_error || cannot_safely_rollback(...)) {
    if (DoDelayedUpdates(...)) local_error = true;
  }
  return local_error ? 1 : -1;
}
```

**阶段一 `DoImmediateUpdatesAndBufferRowIds`（:2418）**：遍历 `m_update_tables`，每张表二选一：
- `table == m_immediate_table` → **立即更新**：`fill_record` + `ha_update_row`
- 否则 → **缓冲**：把 rowid + 新值写进该表的**临时表**（不是 `open_cached_file`，见第五节）

**阶段二 `DoDelayedUpdates`（:2584）**：遍历延迟表，从临时表 `ha_rnd_next` 取 rowid → `PositionScanOnRow`（`ha_rnd_pos`）回表 → `ha_update_row`。

### 3.3 哪些表能立即更新（Halloween 问题）

`safe_update_on_fly`（`sql_update.cc:2036`）只允许 **join 顺序最外层的那张表**（`GetImmediateUpdateTable` 只看 `qep_tab[0]`）。判定核心：

```cpp
if (unique_table(table_ref, all_tables, false)) return false;   // 自连接
if (table->part_info && num_partitions_used() > 1 &&
    partition_key_modified(table, table->write_set)) return false;  // 改分区键
switch (join_tab->type()) {
  case JT_SYSTEM: case JT_CONST: case JT_EQ_REF: return true;
  case JT_REF: case JT_REF_OR_NULL:
    return !is_key_used(table, join_tab->ref().key, table->write_set);
  case JT_RANGE: case JT_INDEX_MERGE:
    if (uses_index_on_fields(join_tab->range_scan(), table->write_set)) return false;
    ...
}
```

一句话：**改索引键 / 分区键 / 影响后续 join 选行的列 → 不能立即更新**，否则同一行可能被扫两次（Halloween 问题）。

### 3.4 多表 UPDATE 的限制（sql_update.cc:1751）

- **不能 ORDER BY / LIMIT**：`ER_WRONG_USAGE`（多表视图隐藏的情况也检查）
- **不能同一表 join 多次**：`unique_table` 检查，报 `ER_UPDATE_TABLE_USED`
- **禁用 join buffer / hash join**：强制 `SELECT_NO_JOIN_CACHE`（`:1575`）——因为 `UpdateRowsIterator` 假设 NLJ（`safe_update_on_fly`）；hypergraph 例外，它自己把 rowid 从 hash join buffer 拷进 `table->file->ref`

---

## 四、DELETE：单表快路径 vs 多表

### 4.1 判定（sql_delete.cc:890）

```cpp
return multitable ? Sql_cmd_dml::execute_inner(thd)  // 多表：DeleteRowsIterator
                  : delete_from_single_table(thd);     // 单表：快路径
```

与 UPDATE 完全对称。单表 `delete_from_single_table`（:204）有额外优化：**无 WHERE、无 LIMIT、非 ROW binlog、无 DELETE 触发器时直接 `ha_delete_all_rows`**（:318）。

### 4.2 `DeleteRowsIterator` 与 UpdateRowsIterator 的异同（sql_delete.cc:916/1186）

相同：都是 `Read()` 一次跑完、两阶段（`DoImmediateDeletesAndBufferRowIds` :1044 → `DoDelayedDeletes` :1113）、都先定 `m_immediate_tables`。

不同：**DELETE 缓冲的是 rowid（不是新值），用的是 `Unique` 树而非临时表**：

```cpp
// sql_delete.cc:1027-1036
auto tempfile = make_unique_destroy_only<Unique>(
    thd()->mem_root, refpos_order_cmp, table->file,
    table->file->ref_length, thd()->variables.sortbuff_size);
```

- 阶段一：立即表 `ha_delete_row`，延迟表 `tempfile->unique_add(table->file->ref)` 存 rowid
- 阶段二：`Unique::get` 取 rowid → `init_table_iterator(ignore_not_found_rows=true)` → `ha_delete_row`
- **`ignore_not_found_rows=true`** 是因为外键级联可能已把行删掉，回表 `rnd_pos` 找不到要忽略

**DELETE 仍需两阶段**：非最外层/被再次引用的表边扫边删会破坏 join 一致性；且 `ha_delete_row` 需要完整行（`SetUpTablesForDelete` 里 `covering_keys.clear_all()` + `prepare_for_position()`，:953-961）。

### 4.3 DELETE 的 ORDER BY / LIMIT（单表支持，多表不支持）

是**语法层限制**（`sql_yacc.yy:13482` 单表语法有 `opt_order_clause opt_simple_limit`，`:13496` 多表语法只有 `opt_where_clause`）。

原因：单表 DELETE 的"行流"是确定的（一张表 + ORDER BY 排序 + LIMIT 截断），可 Filesort 排序后逐行删；多表 DELETE 的行流是 join 结果，且每个目标表 rowid 分开缓冲到各自 `Unique`，**不存在统一的、可排序可截断的行流**，ORDER BY/LIMIT 语义无意义。

### 4.4 外键级联

**由 InnoDB 在 `ha_delete_row` 内部处理，server 层不显式级联**：

```cpp
// handler.cc:8059
int handler::ha_delete_row(const uchar *buf) {
  mark_trx_read_write();
  error = delete_row(buf);              // 虚函数 → InnoDB（内部做 CASCADE/SET NULL/RESTRICT）
  if (unlikely(error)) return error;
  error = binlog_log_row(table, buf, nullptr, log_func);   // binlog
  return 0;
}
```

server 层只做：BEFORE DELETE 触发器 → `ha_delete_row` → AFTER DELETE 触发器。`ignore_not_found_rows=true` 正是这个语义的体现（父表行可能已被级联删掉）。

---

## 五、buffer row id 两阶段读

> **先纠正一个版本前提**：8.0.39 里 `open_cached_file + position() + rnd_pos` 两阶段模式**只用于"单表 UPDATE 改扫描键 / 带 ORDER BY"的快路径**（`sql_update.cc:647-822`）。**多表 UPDATE 的延迟缓冲用临时表（MEMORY→InnoDB），多表 DELETE 用 `Unique` 树**——它们同样是"记 rowid → 回表"的思想，但载体不同。

### 5.1 为什么需要两阶段

**改扫描键 / ORDER BY / 多表**时不能边扫边改：改了索引键会让记录在索引里移位，导致**同一行被扫两次（Halloween）或漏行**。所以先只记主键/rowid，扫完再回表。

### 5.2 多表 UPDATE 的临时表缓冲（sql_update.cc:2095）

每个延迟表建一张临时表，rowid 字段建唯一键去重：

```cpp
// StoreRowId（:2125）
table->file->position(table->record[0]);   // NLJ 下主动取 rowid
tmp_table->visible_field_ptr()[field_num]->store(
    (char *)table->file->ref, table->file->ref_length, &my_charset_bin);

// PositionScanOnRow（:2153）
table->file->ha_rnd_pos(table->record[0], tmp_table->...->data_ptr());  // 回表
```

### 5.3 单表快路径的 open_cached_file（sql_update.cc:749）

```cpp
if (open_cached_file(tempfile, mysql_tmpdir, TEMP_PREFIX, DISK_BUFFER_SIZE, MYF(MY_WME))) ...
// 阶段一：扫出匹配行，只把 rowid 写 tempfile
while (!(error = iterator->Read()) && !thd->killed) {
  table->file->position(table->record[0]);
  if (my_b_write(tempfile, table->file->ref, table->file->ref_length)) { ... }
}
// 阶段二：SortFileIndirectIterator 读 tempfile → rnd_pos 回表 → ha_update_row
iterator = NewIterator<SortFileIndirectIterator>(thd, ..., tempfile, ...);
```

### 5.4 落盘阈值（已核实）

- `include/my_io.h:159/164`：`IO_SIZE{4096}`，`DISK_BUFFER_SIZE{IO_SIZE * 16}` = **65536 = 64KB**
- `mysys/mf_cache.cc:53`：`open_cached_file` 只是 `init_io_cache(-1, cache_size, WRITE_CACHE, ...)`——**文件描述符传 -1，不立刻建磁盘文件**
- `mysys/mf_iocache.cc:41`：临时文件**只在写满 64KB 缓冲或显式 `my_b_flush_io_cache` 时才真正落盘**

所以：行位置数据 **< 64KB 时纯内存不落盘**；> 64KB 才在 tmpdir 溢出成临时文件（关闭即删）。

### 5.5 `position()` 存的是什么

`position()` 写 `table->file->ref`（引擎的 rowid/主键，长度 `ref_length`），**不是"主键列"**。InnoDB 有主键存主键值、无主键存隐藏 row id（见 [`../handler.md`](../handler.md)）。第二阶段 `ha_rnd_pos(ref)` 随机回表。

---

## 六、与 SELECT 在 prepare / optimize 的差异

### 6.1 read_set / write_set 的确定

SELECT 只需要 `read_set`（读哪些列）；DML 额外需要 **`write_set`（改哪些列）+ 前镜像 read_set**（binlog/UNDO/触发器/约束需要的额外列）：

```cpp
// table.cc:5565  mark_columns_needed_for_delete
void TABLE::mark_columns_needed_for_delete(THD *thd) {
  mark_columns_per_binlog_row_image(thd);       // RBR 前镜像
  if (triggers && triggers->mark_fields(TRG_EVENT_DELETE)) return;
  if (file->ha_table_flags() & HA_REQUIRES_KEY_COLUMNS_FOR_DELETE) { ... }  // 键列
  if (file->ha_table_flags() & HA_PRIMARY_KEY_REQUIRED_FOR_DELETE) { ... }  // 主键
  if (vfield) mark_generated_columns(true);     // 虚拟生成列（UNDO 需要）
}

// table.cc:5642  mark_columns_needed_for_update —— 额外还有
if (table_check_constraint_list != nullptr) mark_check_constraint_columns(true);
```

`write_set` 由 `setup_fields(..., column_update=true)` 在解析 SET 列表时填——`column_update=true` 是 UPDATE 特有分支，让 `Item_field::fix_fields` 走"目标列"解析路径（写 write_set），SELECT 完全不会走。

### 6.2 优化目标不同

DML 的 optimize 是"**在满足写安全约束下的最优扫描计划**"，额外受三条约束：

1. **目标表需要完整行** → 覆盖索引禁用（`covering_keys.clear_all()` / `no_keyread = true`，`sql_update.cc:1655` / `sql_update.cc:1882`）
2. **不能重读同一行** → 立即更新只允许 join 最外层表，其余缓冲
3. **不能 join buffer/hash join**（老优化器）

最终在 AccessPath 顶上包一层 `UPDATE_ROWS` / `DELETE_ROWS`（见 6.5）。

### 6.3 分区裁剪

单表快路径显式调用 `prune_partitions`（`sql_update.cc:474` / `sql_delete.cc:389`）；全剪掉时 `all_partitions_pruned_away=true`，DML **直接短路返回 OK，不碰任何分区**。`write_set` 必须在裁剪前就绪（裁剪用它判断能否剪锁）。

### 6.4 ICP 的限制

`QEP_TAB::push_index_cond`（`sql_select.cc:2985`）明确排除多表 UPDATE/DELETE：

```cpp
join_->thd->lex->sql_command != SQLCOM_UPDATE_MULTI &&
join_->thd->lex->sql_command != SQLCOM_DELETE_MULTI &&
```

原因（`:2960` 注释）：**同一个 handler 既做 join 扫描又做更新**，引擎若把 pushed index condition 也套到"定位要更新的行"上（`ha_rnd_pos` 回表），会导致**找不到记录或更新错记录**。单表快路径不走 QEP_TAB 流程，不受此限。

### 6.5 AccessPath：UPDATE_ROWS / DELETE_ROWS

`create_table_access_path`（`sql_executor.cc:4690`）只生成**底层扫描**；`UPDATE_ROWS`/`DELETE_ROWS` 由 `JOIN::attach_access_path_for_update_or_delete`（`:2924`）包在最外层：

```cpp
// sql_executor.cc:2924
if (command == SQLCOM_UPDATE_MULTI) {
  path = NewUpdateRowsAccessPath(thd, path, target_tables, GetImmediateUpdateTable(...));
} else if (command == SQLCOM_DELETE_MULTI) {
  path = NewDeleteRowsAccessPath(thd, path, target_tables, GetImmediateDeleteTables(...));
}
```

结构（`access_path.h:1207`）：

```cpp
struct {
  AccessPath *child;                // 下方 join/扫描路径
  table_map tables_to_update;       // 目标表位图
  table_map immediate_tables;       // 能"边扫边写"的表位图（子集）
} update_rows;
```

`immediate_tables ⊆ target_tables` 是优化器在规划阶段决定的，迭代器创建时据此决定 `m_immediate_table`——打通"AccessPath 语义 → RowIterator 两阶段执行"。

### 6.6 隐式提交：DML 不隐式提交

**DDL** 在执行前经 `stmt_causes_implicit_commit`（`sql_parse.cc:396`）强制提交已有事务；INSERT/UPDATE/DELETE 的 `sql_command_flags` **不含** `CF_IMPLICIT_COMMIT_BEGIN`，所以不预提交。

语句结束的提交在 `mysql_execute_command` 尾部（`sql_parse.cc:4924`）：

```cpp
if (thd->is_error() ...) trans_rollback_stmt(thd);
else { trans_commit_stmt(thd); }
```

`trans_commit_stmt`（`transaction.cc:513`）内部：**autocommit=1** 时 `in_active_multi_stmt_transaction()==false`，`ha_commit_trans` 真正落盘；**显式 BEGIN 事务里**只结束语句级 savepoint，**不提交整个事务**。这就是"DML 不隐式提交"的精确含义——它随 autocommit 提交或留在外层事务，而 DDL 无论 autocommit 与否都先强制提交。

> 注：`handler::ha_autocommit_or_rollback` 这个符号在 8.0.39 已不存在（只是历史注释残留），实际入口是 `trans_commit_stmt`/`trans_rollback_stmt`。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `autocommit` | **1** | 每条 DML 自动提交；0 则进显式事务 |
| `sql_safe_updates` | **0** | 1 时禁止无 WHERE 的 UPDATE/DELETE（防误删） |
| `sort_buffer_size` | 256KB | DELETE 延迟表 `Unique` 的内存上限，超出溢写磁盘 |
| `max_sort_length` | 1024 | `Unique` 比较 rowid 的最大长度 |
| `unique_checks` | 1 | 关闭可跳过唯一键检查（引擎优化） |

### 状态变量

| 变量 | 说明 |
|------|------|
| `Com_insert` / `Com_update` / `Com_delete` | 各类 DML 执行次数 |
| `Com_insert_select` / `Com_replace` | 特殊 DML 执行次数 |
| `Handler_write` / `Handler_update` / `Handler_delete` | 引擎层写/改/删次数 |
| `Rows_tmp` | 延迟更新/删除用的临时行数 |

---


## 参考

**论文**
- D. Chamberlin, M. Astrahan, et al. *A History and Evaluation of System R*. CACM 1981.（Halloween Problem 的原始出处与解决思路）

**官方文档**
- *MySQL 8.0 Reference Manual → INSERT / UPDATE / DELETE Statement*
- *MySQL 8.0 Reference Manual → INSERT ... ON DUPLICATE KEY UPDATE Statement*

**内核月报**
- 阿里云/腾讯云数据库内核月报中关于"UPDATE 两阶段执行与 Halloween Problem"的专题

**相关文档**
- SELECT 主链见 [`02`](04_parser.md) ~ [`07`](09_executor_iterator.md)
- 优化器见 [`07_optimize/`](07_optimize/README.md)
- handler 接口与 `rnd_pos` 见 [`../handler.md`](../handler.md)

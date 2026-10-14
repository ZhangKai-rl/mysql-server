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

⚠️ **勘误**：**不存在 `sql/sql_cmd_dml.cc`**（`sql/` 下只有 `sql_cmd_dml.h`）。`Sql_cmd_dml::prepare()` / `execute()` / `execute_inner()` 定义在 **`sql/sql_select.cc:495 / 675 / 1008`**。

**`Sql_cmd_dml::execute_inner()` 的默认实现——"DML 复用 SELECT 主链"的实质**（`sql_select.cc:1008`）：

```cpp
/**
  Execute a DML statement.
  This is the default implementation for a DML statement and uses a
  nested-loop join processor per outer-most query block.
*/
bool Sql_cmd_dml::execute_inner(THD *thd) {
  Query_expression *unit = lex->unit;

  if (unit->optimize(thd, /*materialize_destination=*/nullptr,
                     /*create_iterators=*/true, /*finalize_access_paths=*/true))
    return true;

  accumulate_statement_cost(lex);

  if (optimize_secondary_engine(thd)) return true;

  lex->set_exec_completed();
  if (lex->is_explain()) {
    if (explain_query(thd, thd, unit)) return true;
  } else {
    if (unit->execute(thd)) return true;
  }
  return false;
}
```

> ★ 所谓"走 SELECT 主链"，就是这一句 `unit->optimize() + unit->execute()`。DELETE / UPDATE 只是在 `root_access_path` 之上多包了一层 `DELETE_ROWS` / `UPDATE_ROWS`（→ `DeleteRowsIterator` / `UpdateRowsIterator`）。

**重写 `execute_inner()` 的真实清单**（只有 3 个 DML，另外 `Sql_cmd_call` 一个）：

| 类 | 声明位置 | 分流表达式 |
|---|---|---|
| `Sql_cmd_delete` | `sql/sql_delete.h:57` | `multitable ? Sql_cmd_dml::execute_inner(thd) : delete_from_single_table(thd)` |
| `Sql_cmd_update` | `sql/sql_update.h:145` | `multitable ? Sql_cmd_dml::execute_inner(thd) : update_single_table(thd)` |
| `Sql_cmd_insert_values` | `sql/sql_insert.h:327` | 唯一一个 INSERT 的 `execute_inner`，无分流 |

> ⚠️ `Sql_cmd_insert_base` **没有** `execute_inner`；`Sql_cmd_insert_select` 也**没有**——它靠基类 `Sql_cmd_dml::execute()` + 结果 sink `Query_result_insert`。

```cpp
// sql/sql_insert.h:223 —— INSERT 家族真实位置在 sql_insert.h，不在 sql_cmd_dml.h
class Sql_cmd_insert_base : public Sql_cmd_dml {
 protected:
  bool precheck(THD *thd) override;
  bool check_privileges(THD *thd) override;
  bool prepare_inner(THD *thd) override;
  bool restore_cmd_properties(THD *thd) override;
 public:
  mem_root_deque<Item *> insert_field_list;
  /// Row data to insert/replace. For ... VALUES (row1),(row2),...
  /// the list contains N List_item lists: one List_item per row.
  mem_root_deque<List_item *> insert_many_values;   // ★ 主循环遍历的就是它
  uint column_count;
  uint value_count;
  mem_root_deque<Item *> update_field_list;   // ON DUPLICATE KEY UPDATE
  mem_root_deque<Item *> update_value_list;
  /// ON DUPLICATE KEY UPDATE reference to VALUES.. as a derived table.
  Table_ref *values_table{nullptr};           // ★ 8.0 把 VALUES(..) 建模成派生表
  const enum_duplicates duplicates;
};

class Sql_cmd_insert_values : public Sql_cmd_insert_base {
 protected:
  bool execute_inner(THD *thd) override;      // :327 全 INSERT 链路唯一一个
};

class Sql_cmd_insert_select : public Sql_cmd_insert_base {
  // ★ 没有 execute_inner —— 走基类 + Query_result_insert
};
```

```cpp
// sql/sql_update.h:126
class Sql_cmd_update final : public Sql_cmd_dml {
 private:
  bool update_single_table(THD *thd);
  bool multitable;                    // ★ 快路径 / 慢路径的唯一开关
  table_map tables_for_update{0};
};

// sql/sql_delete.h:38
class Sql_cmd_delete final : public Sql_cmd_dml {
 private:
  bool delete_from_single_table(THD *thd);
  bool multitable;                    // ★ 同上
  SQL_I_List<Table_ref> *delete_tables;
};
```

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

### 2.1.1 ★ `fill_record()`：把值列表逐列写进 `record[0]`（`sql_base.cc:9505-9581`）

上面主循环第 3 步的 `fill_record_n_invoke_before_triggers()` 就是它 + BEFORE 触发器的包装。这是"**值 → 行缓冲**"的唯一入口。

> **先说清输入输出与它做了什么**：
> - **输入**：`fields`（目标列列表）+ `values`（值表达式列表，与 `fields` 一一对应）+ `bitmap`（本次要写哪些列，`nullptr` = 全部）
> - **输出**：`record[0]` 被**逐列就地填好**；返回 `bool`（true = 出错）
> - **做了什么**：遍历列 → 对每列求值 → 转调 `save_in_field()` 写进该列偏移；**生成列先跳过**，等所有基列写完再统一算

完整函数 + 关键行就近注释：

```cpp
bool fill_record(THD *thd, TABLE *table, const mem_root_deque<Item *> &fields,
                 const mem_root_deque<Item *> &values, MY_BITMAP *bitmap,
                 MY_BITMAP *insert_into_fields_bitmap,
                 bool raise_autoinc_has_expl_non_null_val) {
  DBUG_TRACE;

  assert(CountVisibleFields(fields) == CountVisibleFields(values));  // 列数与值数相等（只数可见列）
  ...
  auto value_it = VisibleFields(values).begin();
  for (Item *fld : VisibleFields(fields)) {
    // ★ 可更新视图：视图的列先映射回基表的列（最终落的仍是基表 record[0]）
    Item_field *const field = fld->field_for_view_update();
    assert(field != nullptr && field->table_ref->table == table);

    Field *const rfield = field->field;
    Item *value = *value_it++;

    // bitmap 过滤：只处理"本次要写"的列（UPDATE 的 SET 列表、部分列 INSERT）
    if (bitmap && !bitmap_is_set(bitmap, rfield->field_index())) continue;

    // ★ 记下"这一列本次被写过"——后面 check_that_all_fields_are_given_values 靠它找"没给值"的列
    bitmap_set_bit(table->fields_set_during_insert, rfield->field_index());
    if (insert_into_fields_bitmap)
      bitmap_set_bit(insert_into_fields_bitmap, rfield->field_index());

    // ★ 生成列先跳过：它可能引用其它列，必须等所有基列写完再算（见循环后）
    if (rfield->is_gcol()) continue;

    if (raise_autoinc_has_expl_non_null_val && rfield == table->next_number_field)
      table->autoinc_field_has_explicit_non_null_value = true;   // 用户显式给了 auto_increment 值

    /* We handle errors from save_in_field() by first checking the return
    value and then testing thd->is_error(). ... */
    // ★ 真正写入：Item → Field
    //   完整剖析（模板方法 save_in_field → save_in_field_inner）见
    //   13_item_expression.md「4.2 ★ save_in_field()」；
    //   再往下的 Field::store / get_int 编码见 data_repr.md「填充链路」
    if (value->save_in_field(rfield, false) < 0) {
      my_error(ER_UNKNOWN_ERROR, MYF(0));
      return true;
    }
    if (thd->is_error()) return true;   // 可能不返回负值但已置错（源码 @todo 自己承认这别扭）
  }

  // ★ 第二阶段：所有基列写完后才统一计算生成列
  if (table->has_gcol() &&
      update_generated_write_fields(bitmap ? bitmap : table->write_set, table))
    return true;

  return thd->is_error();
}
```

**五个要点**：

1. **★ 生成列是两阶段的（最核心）**：循环里 `if (rfield->is_gcol()) continue;` 先跳过所有生成列，循环结束后才 `update_generated_write_fields()` 统一计算。⇒ 因为生成列**可以引用其它列**（甚至别的生成列），必须等依赖列全部就位才能求值。这也是为什么那个调用在循环**外**
2. **`bitmap` 决定"写哪些列"**：UPDATE 传 SET 列表的位图、部分列 INSERT 传对应位图；传 `nullptr` 表示全部列
3. **`fields_set_during_insert` 是"被写过"的记录**：主循环第 4 步 `check_that_all_fields_are_given_values()` 就靠它找出"没给值又没默认值"的列 → `ER_NO_DEFAULT_FOR_FIELD`
4. **可更新视图在这里归一**：`fld->field_for_view_update()` 把视图列映射回基表列，所以写视图最终写的还是基表的 `record[0]`
5. **错误处理是两层判据**：`save_in_field() < 0` **或** `thd->is_error()`——源码注释自己承认这很别扭（`@todo ... enum which should never be a negative value`）

⇒ **与上一节主循环的因果**：调用它之前必须先 `restore_record(table, s->default_values)`（主循环第 2 步）。因为 `record[0]` 是**复用且不清零**的缓冲（见 [`data_repr.md`](../../infra/structure/data_repr.md)「`record[0]` 的生命周期」）——若不先恢复默认值模板，上一行残留的值会因为"本次没被 `fill_record` 覆盖"而**漏进新行**。

### 2.2 `write_record` 与三种重复键策略（sql_insert.cc:1793）

⚠️ **勘误**：`write_record()` **不是 switch**，而是一个 **`while ((error = table->file->ha_write_row(...)))` 重试循环**（`:1814`）——`DUP_REPLACE` 删掉旧行后会回到循环顶部**再写一次**。

```cpp
// sql/sql_insert.cc:1793
bool write_record(THD *thd, TABLE *table, COPY_INFO *info, COPY_INFO *update) {
  ulonglong prev_insert_id = table->file->next_insert_id;   // ★ 进入本行前保存自增游标，出错要归还
  ulonglong insert_id_for_cur_row = 0;
  MEM_ROOT mem_root(PSI_NOT_INSTRUMENTED, 256);
  info->stats.records++;                                    // ★ "Records:" 计数
  const enum_duplicates duplicate_handling = info->get_duplicate_handling();

  if (duplicate_handling == DUP_REPLACE || duplicate_handling == DUP_UPDATE) {
    while ((error = table->file->ha_write_row(table->record[0]))) {   // ★ 第一次写
      if (!table->file->is_ignorable_error(error)) goto err;
      is_duplicate_key_error =
          (error == HA_ERR_FOUND_DUPP_KEY || error == HA_ERR_FOUND_DUPP_UNIQUE);
      if (!is_duplicate_key_error) {
        info->last_errno = error;
        table->file->print_error(error, MYF(0));
        if (thd->is_error()) goto before_trg_err;
        goto ok_or_after_trg_err;   // IGNORE 下非重复错误被吞掉
      }
      if ((int)(key_nr = table->file->get_dup_key(error)) < 0) {
        error = HA_ERR_FOUND_DUPP_KEY;
        goto err;
      }
      table->use_all_columns();                     // ★ 冲突处理要读整行，临时放开 read_set

      /* REPLACE 撞自增唯一键 → 直接报错，不做替换 */
      if (duplicate_handling == DUP_REPLACE && table->next_number_field &&
          key_nr == table->s->next_number_index && (insert_id_for_cur_row > 0))
        goto err;

      if (table->file->ha_table_flags() & HA_DUPLICATE_POS) {
        if (table->file->ha_rnd_pos(table->record[1], table->file->dup_ref))
          goto err;                                 // ★ 旧行读进 record[1]
      } else if (key_nr < MAX_KEY) {
        key_copy((uchar *)key, table->record[0], table->key_info + key_nr, 0);
        if ((error = table->file->ha_index_read_idx_map(
                 table->record[1], key_nr, (uchar *)key, HA_WHOLE_KEY,
                 HA_READ_KEY_EXACT)))                // ★ 按冲突键读旧行到 record[1]
          goto err;
      }
      /* ↓ 见 §2.3 / §2.4 的 DUP_UPDATE 与 DUP_REPLACE 分支 */
    }
    ...
  } else if ((error = table->file->ha_write_row(table->record[0]))) {   // ★ :2163 DUP_ERROR
    info->last_errno = error;
    myf error_flags = MYF(0);
    if (table->file->is_fatal_error(error)) error_flags |= ME_FATALERROR;
    table->file->print_error(error, error_flags);
    if (thd->is_error()) goto before_trg_err;       // ★ IGNORE 时这里不成立 → 不报错
    table->file->restore_auto_increment(prev_insert_id);
    goto ok_or_after_trg_err;
  }
  ...
err : {
  info->last_errno = error;
  if (table->file->is_fatal_error(error)) error_flags |= ME_FATALERROR;
  table->file->print_error(error, error_flags);     // ★ 最终用户看到的 "Duplicate entry ..."
}
before_trg_err:
  table->file->restore_auto_increment(prev_insert_id);   // ★ 失败归还自增游标
  ...
  return true;
}
```

**三种策略的落点汇总**：

| 策略 | 落点 | 行数 |
|---|---|---|
| `DUP_ERROR`（默认 / IGNORE） | 单次 `ha_write_row(record[0])`，不重试 | `:2163` |
| `DUP_UPDATE`（ODKU） | `fill_record_n_invoke_before_triggers` + `ha_update_row(record[1], record[0])` | `:1952` / `:2021` |
| `DUP_REPLACE` 快路径 | `ha_update_row(record[1], record[0])`（内部转 UPDATE） | `:2109` |
| `DUP_REPLACE` 慢路径 | `ha_delete_row(record[1])` + **回到 while 顶部重插** | `:2129` |

> ★ **`ER_DUP_ENTRY` 不在 `sql_insert.cc` 抛出**。完整链条是：
> `write_record` 的 `table->file->print_error(error, flags)` → `handler::print_error()` → `case HA_ERR_FOUND_DUPP_KEY`（`sql/handler.cc:4229`）→ `print_keydup_error()`（`:4127`）→ `my_printf_error(ER_DUP_ENTRY, ...)`（`:4117`）。
>
> ★ `IGNORE` 之所以能吞掉重复键：`print_error` 降级为 warning 后 `thd->is_error()` 仍为 false，于是 `:2173` 的 `if (thd->is_error()) goto before_trg_err;` 不成立。

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

⚠️ **勘误**：`:1917` **不是独立函数**，是 `write_record()` 内部的一个 `if (duplicate_handling == DUP_UPDATE)` 分支。

```cpp
      if (duplicate_handling == DUP_UPDATE) {                    // :1917
        /* The insert has failed, store the insert_id generated for
           this row to be re-used for the next insert. */
        if (insert_id_for_cur_row > 0) prev_insert_id = insert_id_for_cur_row;

        store_record(table, insert_values);                      // :1931 ★ 存到 table->insert_values 供 VALUES(col) 引用
        restore_record(table, record[1]);                        // :1940 ★ 旧行 record[1] 拷回 record[0] 作为 UPDATE 基底

        table->autoinc_field_has_explicit_non_null_value = false;
        bool is_row_changed = false;
        if (fill_record_n_invoke_before_triggers(
                thd, update, *update->get_changed_columns(),
                *update->update_values, table, TRG_EVENT_UPDATE, 0, true,
                &is_row_changed))                                // :1952 ★ 用 SET 列表覆盖 record[0] + BEFORE UPDATE 触发器
          goto before_trg_err;

        bool insert_id_consumed = false;
        if (table->autoinc_field_has_explicit_non_null_value &&
            (insert_id_for_cur_row > 0)) {
          const ulonglong auto_incr_val = table->next_number_field->val_int();
          if (auto_incr_val == insert_id_for_cur_row) {
            insert_id_consumed = true;
          } else if (table->file->auto_inc_interval_for_cur_row.in_range(
                         auto_incr_val)) {
            my_error(ER_AUTO_INCREMENT_CONFLICT, MYF(0));
            goto before_trg_err;
          }
        }
        if (!insert_id_consumed)
          table->file->restore_auto_increment(prev_insert_id);   // ★ 插入失败 → 归还自增区间，避免跳号

        info->stats.touched++;                                   // ★ "匹配到"行数（CLIENT_FOUND_ROWS 用）
        if (is_row_changed) {
          if (invoke_table_check_constraints(thd, table)) { ... }
          if ((error = table->file->ha_update_row(table->record[1],
                                                  table->record[0])) &&   // :2021 ★ DUP_UPDATE 落点
              error != HA_ERR_RECORD_IS_THE_SAME) {
            table->file->print_error(error, error_flags);
            if (thd->is_error()) goto before_trg_err;
            goto ok_or_after_trg_err;
          }
          if (error != HA_ERR_RECORD_IS_THE_SAME) info->stats.updated++;
          else error = 0;
          insert_id_for_cur_row = table->file->insert_id_for_cur_row = 0;  // :2048
          // ★ ODKU 走 UPDATE 语义：不影响 LAST_INSERT_ID()
          info->stats.copied++;
        }
        trg_error = (table->triggers &&
                     table->triggers->process_triggers(thd, TRG_EVENT_UPDATE,
                                                       TRG_ACTION_AFTER, true));
        goto ok_or_after_trg_err;   // ★ 不再回头重试 INSERT
      }
```

**`DUP_REPLACE` 的两条路**（`:2106`）——是否能"内部转 UPDATE"的判据：

```cpp
        if (last_uniq_key(table, key_nr) &&
            !table->s->is_referenced_by_foreign_key() &&
            (!table->triggers || !table->triggers->has_delete_triggers())) {
          if ((error = table->file->ha_update_row(table->record[1],
                                                  table->record[0])) &&   // :2109 快路径
              error != HA_ERR_RECORD_IS_THE_SAME)
            goto err;
          if (error != HA_ERR_RECORD_IS_THE_SAME) info->stats.deleted++;
          else error = 0;
          goto after_trg_n_copied_inc;   // ★ 假装是 INSERT → 跑 AFTER INSERT 触发器
        } else {
          if (table->triggers && table->triggers->process_triggers(
                  thd, TRG_EVENT_DELETE, TRG_ACTION_BEFORE, true))
            goto before_trg_err;
          if ((error = table->file->ha_delete_row(table->record[1]))) goto err;  // :2129 慢路径：真删
          info->stats.deleted++;
          if (!table->file->has_transactions())
            thd->get_transaction()->mark_modified_non_trans_table(Transaction_ctx::STMT);
          /* Let us attempt do write_row() once more */   // ★ 回到 while 顶部重插
        }
```

> ★ 慢路径的三个触发条件（源码注释说得很清楚）：① 不是最后一个唯一键；② 表被外键引用（InnoDB 的 FK 在"内部转 UPDATE"下不按手册语义工作）；③ 表上有 DELETE 触发器。

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
bool Query_result_insert::send_data(THD *thd,
                                    const mem_root_deque<Item *> &values) {
  Autoinc_field_has_explicit_non_null_value_reset_guard after_each_row(table);
  thd->check_for_truncated_fields = CHECK_FIELD_WARN;
  store_values(thd, values);                       // ★ 值落点（内部也走 fill_record_n_invoke_before_triggers）
  thd->check_for_truncated_fields = CHECK_FIELD_ERROR_FOR_NULL;
  if (thd->is_error()) return true;

  prepare_triggers_for_insert_stmt(thd, table);

  if (table_list) {  // Not CREATE ... SELECT
    switch (table_list->view_check_option(thd)) {  // ★ 这里确实是 switch（视图 CHECK OPTION）
      case VIEW_CHECK_SKIP:  return false;
      case VIEW_CHECK_ERROR: return true;
    }
  } else if (info.stats.records == 0 && run_before_dml_hook(thd)) {
    return true;
  }

  if (invoke_table_check_constraints(thd, table)) return thd->is_error();

  error = write_record(thd, table, &info, &update);   // ★★ 与 VALUES 路径共用 —— INSERT 链路唯一落盘点

  if (!error && (table->triggers || info.get_duplicate_handling() == DUP_UPDATE))
    restore_record(table, s->default_values);   // ★ 下一行要干净起点
  if (!error && table->next_number_field) {
    if (thd->first_successful_insert_id_in_cur_stmt == 0)
      autoinc_value_of_last_inserted_row = table->next_number_field->val_int();
    table->next_number_field->reset();          // ★ 多行插入时每行都要"看起来是 NULL"
  }
  return error;
}

void Query_result_insert::store_values(THD *thd,
                                       const mem_root_deque<Item *> &values) {
  if (CountVisibleFields(*fields) != 0) {          // 有列清单
    restore_record(table, s->default_values);
    if (!validate_default_values_of_unset_fields(thd, table))
      fill_record_n_invoke_before_triggers(thd, &info, *fields, values, table,
                                           TRG_EVENT_INSERT, table->s->fields,
                                           true, nullptr);
  } else
    fill_record_n_invoke_before_triggers(thd, table->field, values, table,
                                         TRG_EVENT_INSERT, table->s->fields);
  //  ★ 无列清单 → 走 Field** 版（sql_base.cc:10037）
  check_that_all_fields_are_given_values(thd, table, table_list);
}
```

> ★ 两条 INSERT 路径（VALUES / SELECT）**在 `write_record()` 汇合**——这是理解 INSERT 链路的关键分叉点。

**`fill_record()` 的落值点（`sql_base.cc:9505`）——值真正写进 `record[0]`**：

```cpp
  auto value_it = VisibleFields(values).begin();
  for (Item *fld : VisibleFields(fields)) {          // ★ 逐列遍历（跳过隐藏列）
    Item_field *const field = fld->field_for_view_update();
    Field *const rfield = field->field;
    Item *value = *value_it++;

    if (bitmap && !bitmap_is_set(bitmap, rfield->field_index())) continue;
    bitmap_set_bit(table->fields_set_during_insert, rfield->field_index());
    if (rfield->is_gcol()) continue;                 // ★ 生成列不在这里填

    if (raise_autoinc_has_expl_non_null_val && rfield == table->next_number_field)
      table->autoinc_field_has_explicit_non_null_value = true;

    if (value->save_in_field(rfield, false) < 0) {   // ★★ 唯一的写值点：Item → Field
      my_error(ER_UNKNOWN_ERROR, MYF(0));
      return true;
    }
    if (thd->is_error()) return true;
  }

  if (table->has_gcol() &&
      update_generated_write_fields(bitmap ? bitmap : table->write_set, table))
    return true;                                     // ★ 生成列在所有基列写完后统一计算
```

> ⚠️ **勘误**：`fill_record()` 有**两个重载**（`:9505` Item 列表版 / `:9931` `Field**` 版），且 INSERT 主路径**不直接调它**，而是调 `fill_record_n_invoke_before_triggers()`（`:9787`）。默认值不在这里——来自调用方的 `restore_record(table, s->default_values)`（`execute_inner:585`）与 `fill_function_defaults` lambda（`:9799`，CURRENT_TIMESTAMP / ON UPDATE 走这条）。
>
> ⚠️ **`invoke_table_value_constraints()` 不存在**（全仓 0 命中）。8.0.39 的对应物是 **`invoke_table_check_constraints()`**。

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

**完整源码**（`sql/sql_update.cc:1804-1821`）——只有 18 行：

```cpp
bool Sql_cmd_update::execute_inner(THD *thd) {
  // Binary logging happens at execution, and needs this information:
  thd->table_map_for_update = tables_for_update;

  if (is_empty_query()) {                        // 分支 0：分区裁剪 / impossible WHERE
    if (lex->is_explain()) {
      Modification_plan plan(thd, MT_UPDATE, /*table_arg=*/nullptr,
                             "No matching rows after partition pruning", true, 0);
      return explain_single_table_modification(thd, thd, &plan, lex->query_block);
    }
    my_ok(thd);
    return false;
  }
  return multitable ? Sql_cmd_dml::execute_inner(thd)   // ★ 慢路径：JOIN + Query_result_update + UpdateRowsIterator
                    : update_single_table(thd);         // ★ 快路径：单表专用，不走 JOIN
}
```

⚠️ **勘误**：**判定不是"表有几个"**，而是 `multitable` 标志，且它在 `prepare_inner()` 里被**四处改写**（`sql_update.cc:1536-1559`）：

```cpp
  if (!multitable) {
    if (!table_list->is_updatable()) {
      my_error(ER_NON_UPDATABLE_TABLE, MYF(0), table_list->alias, "UPDATE");
      return true;
    }
    // Perform multi-table operation if table to be updated is multi-table view
    if (table_list->is_multiple_tables()) multitable = true;      // 改写点①：多表视图
  }

  // The hypergraph optimizer has a unified execution path for single-table and
  // multi-table UPDATE, and does not need to distinguish between the two. This
  // enables it to perform optimizations like sort avoidance and semi-join
  // flattening even if features specific to single-table UPDATE (that is, ORDER
  // BY and LIMIT) are used.
  if (lex->using_hypergraph_optimizer()) {
    multitable = true;                                            // 改写点②：hypergraph 一律走慢路径
  }

  if (!multitable && select->first_inner_query_expression() != nullptr &&
      should_switch_to_multi_table_if_subqueries(thd, select, table_list))
    multitable = true;                                            // 改写点③：相关子查询

  // 改写点④：构造函数 Sql_cmd_update(bool multitable_arg, ...)，由语法决定
```

> ★★ 改写点② 是 8.0 的重要变化：**hypergraph 优化器下单表 UPDATE 也走慢路径**，因此单表 UPDATE 的 ORDER BY / LIMIT 在 hypergraph 下可用（老优化器下单表快路径才支持）。DELETE 完全对称（`sql_delete.cc:776-778`）。

**哪些表能"立即更新"——双轨判定**：

```cpp
// sql/sql_update.cc:2995 —— 老优化器：计划【之后】判定
table_map GetImmediateUpdateTable(const JOIN *join, bool single_target) {
  if (join->zero_result_cause != nullptr) return 0;
  // The hypergraph optimizer determines the immediate update tables during
  // planning, not after planning.
  assert(!join->thd->lex->using_hypergraph_optimizer());

  const QEP_TAB *const first_table = &join->qep_tab[0];
  const Table_ref *const first_table_ref = first_table->table_ref;
  if (!first_table_ref->is_updated()) return 0;      // 最外层表不是更新目标 → 无人能立即更新

  if (!single_target) CollectColumnsReferencedInJoinConditions(join, first_table_ref);

  const bool can_update_on_the_fly =
      safe_update_on_fly(first_table, first_table_ref, join->tables_list);
  bitmap_clear_all(&first_table_ref->table->tmp_set);
  return can_update_on_the_fly ? first_table_ref->map() : 0;
}
```

```cpp
// sql/sql_update.cc:2036 —— safe_update_on_fly 的 switch
  if (unique_table(table_ref, all_tables, false)) return false;   // ★ 自连接：一定不行
  if (bitmap_is_overlapping(&table->tmp_set, table->write_set)) return false;

  switch (join_tab->type()) {
    case JT_SYSTEM: case JT_CONST: case JT_EQ_REF:
      return true;                                   // 最多一行，不会再被读到
    case JT_REF: case JT_REF_OR_NULL:
      return !is_key_used(table, join_tab->ref().key, table->write_set);
    case JT_RANGE: case JT_INDEX_MERGE:
      if (uses_index_on_fields(join_tab->range_scan(), table->write_set))
        return false;
      [[fallthrough]];
    case JT_ALL:
      if ((table->file->ha_table_flags() & HA_PRIMARY_KEY_IN_READ_INDEX) &&
          table->s->primary_key < MAX_KEY)
        return !is_key_used(table, table->s->primary_key, table->write_set);
      return true;
    ...
  }
```

> ★ hypergraph 侧对应物是 `AccessPath::immediate_update_delete_table`（`access_path.h:349`），**在规划期间**就填好，不走这个函数（源码注释明说）。

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

⚠️ **勘误**：`UpdateRowsIterator::Init()` **不建临时表、也不调 `open_cached_file`**——临时表在 `Query_result_update::optimize()`（`:2184`）里用 `create_tmp_table()` 建好。`Init()` 只做两件事：

```cpp
// sql/sql_update.cc:2843
bool UpdateRowsIterator::Init() {
  if (m_source->Init()) return true;

  if (m_outermost_table != nullptr &&
      !Overlaps(m_hash_join_tables,
                m_outermost_table->pos_in_table_list->map())) {
    // As it's the first table in the join, and we're doing a nested loop join,
    // the table is the left argument of that nested loop join; thus, we can ask
    // for semi-consistent read.
    m_outermost_table->file->try_semi_consistent_read(true);
  }
  return false;
}
```

**`Read()` 的两阶段骨架**（`:2864`）——注意它**自写 binlog**：

```cpp
int UpdateRowsIterator::Read() {
  while (!local_error) {
    const int read_error = m_source->Read();
    if (read_error > 0 || thd()->is_error()) { local_error = true; }
    else if (read_error < 0) { break; }                    // EOF
    else if (thd()->killed) { thd()->send_kill_message(); return 1; }
    else {
      // ★★ 阶段一：逐行 —— immediate 表就地改，其余把 rowid 灌进临时表
      local_error = DoImmediateUpdatesAndBufferRowIds(&trans_safe, &transactional_tables);
      diagnostics->inc_current_row_for_condition();
    }
  }

  // ★★ 阶段二：扫描结束后做 delayed updates
  if (!local_error ||
      thd()->get_transaction()->cannot_safely_rollback(Transaction_ctx::STMT)) {
    THD_STAGE_INFO(thd(), stage_updating_reference_tables);
    if (DoDelayedUpdates(&trans_safe, &transactional_tables)) local_error = true;
  }

  if (!local_error || thd()->get_transaction()->cannot_safely_rollback(Transaction_ctx::STMT)) {
    if (mysql_bin_log.is_open()) {
      int errcode = 0;
      if (!local_error) thd()->clear_error();
      else errcode = query_error_code(thd(), killed_status == THD::NOT_KILLED);
      if (thd()->binlog_query(THD::ROW_QUERY_TYPE, thd()->query().str,
                              thd()->query().length, transactional_tables,
                              false, false, errcode))
        local_error = true;                                // ★ binlog 写失败 → 回滚
    }
  }
  return local_error ? 1 : -1;
}
```

**阶段一的两条分支**（`DoImmediateUpdatesAndBufferRowIds`，`:2418`）：

```cpp
  for (Table_ref *cur_table = m_update_tables; cur_table != nullptr;
       cur_table = cur_table->next_local) {
    TABLE *table = cur_table->table;
    uint offset = cur_table->shared;
    if (table->has_null_row() || table->has_updated_row()) continue;

    if (table == m_immediate_table) {                      // ★ 分支 A：立即更新
      table->set_updated_row();
      store_record(table, record[1]);                      // 旧值 → record[1]
      bool is_row_changed = false;
      if (fill_record_n_invoke_before_triggers(
              thd(), m_update_operations[offset], *m_fields_for_table[offset],
              *m_values_for_table[offset], table, TRG_EVENT_UPDATE, 0, false,
              &is_row_changed))
        return true;
      ++m_found_rows;
      if (is_row_changed) {                                // ★ no-op 更新不计入 updated
        ...
        if (m_updated_rows == 0)
          m_outermost_table->file->ha_extra(HA_EXTRA_PREPARE_FOR_UPDATE);
        ++m_updated_rows;
        if ((error = table->file->ha_update_row(table->record[1],
                                                table->record[0])) &&
            error != HA_ERR_RECORD_IS_THE_SAME) { ... }
      }
    } else {                                               // ★ 分支 B：延迟更新
      TABLE *tmp_table = m_tmp_tables[offset];
      int field_num = 0;
      StoreRowId(table, tmp_table, field_num++, m_hash_join_tables);
      for (TABLE &tbl : m_unupdated_check_opt_tables)
        StoreRowId(&tbl, tmp_table, field_num++, m_hash_join_tables);

      /* Store regular updated fields in the row. */
      if (fill_record(thd(), tmp_table,
                      tmp_table->visible_field_ptr() + 1 +
                          m_unupdated_check_opt_tables.size(),
                      *m_values_for_table[offset], nullptr, nullptr, false))
        return true;

      if (!check_unique_constraint(tmp_table))
        return false;  // skip adding duplicate record to the temp table

      /* Write row, ignoring duplicated updates to a row */
      error = tmp_table->file->ha_write_row(tmp_table->record[0]);
      if (error != HA_ERR_FOUND_DUPP_KEY && error != HA_ERR_FOUND_DUPP_UNIQUE) {
        if (error && (create_ondisk_from_heap(thd(), tmp_table, error, true, true, nullptr) ||
                      tmp_table->file->ha_index_init(0, false)))
          return true;                                     // ★ heap 撑爆 → 转 on-disk
        ++m_found_rows;
      }
    }
  }
```

**阶段二 `DoDelayedUpdates()` 的核心循环**（`:2584`）——按 rowid 回表、拷贝新值、比对、更新：

```cpp
    for (;;) {
      if ((local_error = tmp_table->file->ha_rnd_next(tmp_table->record[0]))) {
        if (local_error == HA_ERR_END_OF_FILE) break;
        if (local_error == HA_ERR_RECORD_DELETED) continue;   // May happen on dup key
        ...
      }
      int field_num = 0;
      if (PositionScanOnRow(table, table, tmp_table, field_num++)) goto err;  // ★ rnd_pos 回表
      for (TABLE &tbl : m_unupdated_check_opt_tables)
        if (PositionScanOnRow(table, &tbl, tmp_table, field_num++)) goto err;

      table->set_updated_row();
      store_record(table, record[1]);
      for (copy_field_ptr = m_copy_fields; copy_field_ptr != copy_field_end;
           copy_field_ptr++)
        copy_field_ptr->invoke_do_copy();                     // ★ 新值从临时表拷回
      if (table->vfield && update_generated_write_fields(table->write_set, table))
        goto err;
      ...
      if (!records_are_comparable(table) || compare_records(table)) {   // ★ no-op 过滤
        if (m_update_operations[offset]->set_function_defaults(table)) goto err;
        ...
        local_error = table->file->ha_update_row(table->record[1], table->record[0]);
        if (!local_error) ++m_updated_rows;
        else if (local_error == HA_ERR_RECORD_IS_THE_SAME) local_error = 0;
        ...
      }
    }
```

> ★ 统计数回填路径：`Query_result_update::send_eof()`（`:2942`）用 `down_cast<UpdateRowsIterator *>(unit->root_iterator()->real_iterator())` 取出 `found_rows()` / `updated_rows()`。

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

⚠️ **勘误**：`DeleteRowsIterator` **不在** `composite_iterators.cc`。类声明在 **`sql/iterators/delete_rows_iterator.h:41`**，全部实现在 **`sql/sql_delete.cc:916-1254`**（`UpdateRowsIterator` 同理，声明在 `sql/iterators/update_rows_iterator.h:46`，实现在 `sql_update.cc`）。

```cpp
// sql/iterators/delete_rows_iterator.h:41
/// An iterator that deletes all rows returned by its child iterator.
class DeleteRowsIterator final : public RowIterator {
 public:
  DeleteRowsIterator(THD *thd, unique_ptr_destroy_only<RowIterator> source,
                     JOIN *join, table_map tables_to_delete_from,
                     table_map immediate_tables);
  bool Init() override;
  int Read() override;
  void SetNullRowFlag(bool) override { assert(false); }
  void UnlockRow() override { assert(false); }

 private:
  unique_ptr_destroy_only<RowIterator> m_source;
  JOIN *m_join;
  table_map m_tables_to_delete_from;
  table_map m_immediate_tables;            // ★ "立即删除"不是独立迭代器，是一个位图
  table_map m_hash_join_tables;            // ★ hash join 下不能调 position()
  table_map m_transactional_tables{0};
  table_map m_tables_with_before_triggers{0};
  table_map m_tables_with_after_triggers{0};
  /// Temporary files holding row IDs to delete after the scan ... is complete.
  Mem_root_array<unique_ptr_destroy_only<Unique>> m_tempfiles;
  Mem_root_array<TABLE *> m_delayed_tables;
  ha_rows m_deleted_rows{0};
  bool m_has_delayed_deletes{false};

  bool DoImmediateDeletesAndBufferRowIds();
  bool DoDelayedDeletes();
  bool DoDelayedDeletesFromTable(TABLE *table);
};
```

**两阶段的落点**（`sql_delete.cc:1040-1100`）：

```cpp
bool DeleteRowsIterator::DoImmediateDeletesAndBufferRowIds() {
  if (thd()->lex->is_explain_analyze) return false;   // ★ EXPLAIN ANALYZE 下不真删

  int unique_counter = 0;
  for (Table_ref *tr = m_join->query_block->leaf_tables; tr != nullptr;
       tr = tr->next_leaf) {
    if (!Overlaps(tr->map(), m_tables_to_delete_from)) continue;
    const bool immediate = Overlaps(tr->map(), m_immediate_tables);
    TABLE *const table = tr->table;
    Unique *const tempfile = immediate ? nullptr : m_tempfiles[unique_counter++].get();

    if (table->has_null_row() || table->has_deleted_row()) continue;  // ★ 外连接补的 NULL 行不能删

    // Hash joins have already copied the row ID from the join buffer into
    // table->file->ref. Nested loop joins have not, so we call position().
    if (!Overlaps(tr->map(), m_hash_join_tables))
      table->file->position(table->record[0]);       // ★ rowid → handler::ref

    if (immediate) {
      table->set_deleted_row();
      if (DeleteCurrentRowAndProcessTriggers(thd(), table, ...))
        return true;                                  // ★ 就地 ha_delete_row
    } else {
      m_has_delayed_deletes = true;
      if (tempfile->unique_add(table->file->ref)) return true;   // ★ rowid 进 Unique
    }
  }
  return false;
}
```

```cpp
// sql/sql_delete.cc:112 —— 真正执行删除（含 BEFORE/AFTER 触发器）
  if (invoke_before_triggers &&
      table->triggers->process_triggers(thd, TRG_EVENT_DELETE,
                                        TRG_ACTION_BEFORE, false))
    return true;
  if (const int delete_error = table->file->ha_delete_row(table->record[0]);
      delete_error != 0) {
    table->file->print_error(delete_error, error_flags);
    return thd->is_error();     // ★ IGNORE 可能把错误降级成 warning
  }
  ++*deleted_rows;
  ...
```

**两者异同对照表**：

| 维度 | `DeleteRowsIterator` | `UpdateRowsIterator` |
|---|---|---|
| 声明 | `delete_rows_iterator.h:41` | `update_rows_iterator.h:46` |
| 实现 | `sql_delete.cc:916-1254` | `sql_update.cc:2843-3040+` |
| 构造参数 | 4 参（靠 `m_join` 反查 leaf_tables） | 11 参（额外带 `Copy_field*`、`COPY_INFO**`、tmp_tables 等） |
| 核心流程 | **完全同构**：扫 child → immediate 就地做 + delayed 缓存 rowid → 扫完做 delayed → 自写 binlog | 同构（`DoImmediateUpdatesAndBufferRowIds` / `DoDelayedUpdates`） |
| `Init()` | ① `CheckSqlSafeUpdate()` ② AFTER 触发器 → `HA_EXTRA_DELETE_CANNOT_BATCH` ③ IGNORE → `HA_EXTRA_IGNORE_DUP_KEY` ④ 为 delayed 表建 `Unique` | ① 只 `m_source->Init()` ② outermost 表开 `try_semi_consistent_read(true)`；**不做** safe-update 检查（那在 `sql_update.cc:2220`） |
| 析构 | 无 | 有 `~UpdateRowsIterator()`，关掉 semi-consistent read |
| delayed 回表 | `init_table_iterator(ignore_not_found_rows=true)` + `start/end_bulk_delete()` | tmp table + `Copy_field` 把新值拷回目标表 |
| 事务跟踪 | `table_map m_transactional_tables` 位图 | `bool trans_safe` + `bool transactional_tables` 两个标量 |
| 计数 | `m_deleted_rows`，`Read()` 内 `set_row_count_func()` | `m_found_rows` / `m_updated_rows`，由 `Query_result_update::send_eof()` 取 |
| EXPLAIN ANALYZE | 显式跳过真删 | 无此短路 |

> ⚠️ **`ImmediateDeleteRowsIterator` / `FoundRowsIterator` / `ImmediateUpdateRowsIterator` 全都不存在**（全仓 0 命中）。
>
> ⚠️ **`Query_result_delete::do_deletes()` / `do_table_deletes()` 也不存在**（5.x 老 API）。`Query_result_delete`（`sql_delete.cc:99`）只有 3 个成员，`send_data()` 直接 `assert(false)`；它只负责最后的 `my_ok(thd, thd->get_row_count_func())`。
>
> ⚠️ **`Query_result_update::do_updates()` 是僵尸声明**（`sql_update.h:118`，无定义、无调用）；`do_table_updates` 全仓 0 命中。

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

**三件套源码**（`sql/sql_update.cc:2095-2173`）：

```cpp
// ① 在临时表里为 rowid 预留一个 varstring 字段
static bool AddRowIdAsTempTableField(THD *thd, TABLE *table,
                                     mem_root_deque<Item *> *fields) {
  /*
    Signal each table ... for which we will store row position in the temporary
    table that we need a position to be read first.
  */
  table->prepare_for_position();      // ★ 通知 handler：每次读都要把 rowid 放进 table->file->ref

  /*
    A tmp table is moved to InnoDB if it doesn't fit in memory,
    and InnoDB does not support fixed length string fields bigger
    than 1024 bytes, so use a variable length string field.
  */
  Field_varstring *field = new (thd->mem_root) Field_varstring(
      table->file->ref_length, false, table->alias, table->s, &my_charset_bin);
  ...
  return fields->push_back(ifield);
}

// ② 把当前行的 rowid 写进临时表字段
static void StoreRowId(TABLE *table, TABLE *tmp_table, int field_num,
                       table_map hash_join_tables) {
  // Hash joins have already copied the row ID from the join buffer into
  // table->file->ref. Nested loop joins have not, so we call position() to get
  // the row ID from the handler.
  if (!Overlaps(hash_join_tables, table->pos_in_table_list->map())) {
    table->file->position(table->record[0]);   // ★★ rowid → table->file->ref[0..ref_length)
  }
  tmp_table->visible_field_ptr()[field_num]->store(
      pointer_cast<const char *>(table->file->ref), table->file->ref_length,
      &my_charset_bin);
  tmp_table->visible_field_ptr()[field_num]->set_notnull();
}

// ③ 按 rowid 精确回表
static bool PositionScanOnRow(TABLE *updated_table, TABLE *table,
                              TABLE *tmp_table, int field_num) {
  /*
    The row-id is after the "length bytes", and the storage
    engine knows its length. Pass the "data_ptr()" instead of
    the "field_ptr()" to ha_rnd_pos().
  */
  if (const int error = table->file->ha_rnd_pos(
          table->record[0],
          const_cast<uchar *>(
              tmp_table->visible_field_ptr()[field_num]->data_ptr()))) {
    ...
  }
  return false;
}
```

> ★★ **`position()` 存的是 `table->file->ref` 中 `ref_length` 字节的引擎私有行标识**。InnoDB 下就是**主键（无显式 PK 则是隐式 6 字节 ROWID）的 memcmp 可比较编码**，由 `handler::position()` 填写，回表时交给 `ha_rnd_pos()`。它不是 MySQL 逻辑行号，也**不保证跨 UPDATE 稳定**（改了 PK 就变）。
>
> ★ `m_hash_join_tables` 的存在理由（源码注释）：hash join 下 rowid **已经在** `handler::ref` 里（由 join buffer 拷过来），此时再调 `position()` 会把它**覆盖成错值**（多半是该表最后读到的那行）。

**单表快路径的 `open_cached_file`（`sql_update.cc:749` 附近）**：

```cpp
        IO_CACHE *tempfile =
            (IO_CACHE *)my_malloc(key_memory_TABLE_sort_io_cache,
                                  sizeof(IO_CACHE), MYF(MY_FAE | MY_ZEROFILL));

        if (open_cached_file(tempfile, mysql_tmpdir, TEMP_PREFIX,
                             DISK_BUFFER_SIZE, MYF(MY_WME))) {   // ★ 64 KiB
          my_free(tempfile);
          return true;
        }

        while (!(error = iterator->Read()) && !thd->killed) {
          ...
          table->file->position(table->record[0]);                // ★ 阶段一：只收 rowid
          if (my_b_write(tempfile, table->file->ref, table->file->ref_length)) {
            error = 1;
            break;
          }
          if (!--limit && using_limit) { error = -1; break; }
        }
        ...
        if (reinit_io_cache(tempfile, READ_CACHE, 0L, false, false)) error = 1;
        ...
        iterator = NewIterator<SortFileIndirectIterator>(   // ★ 阶段二：从 rowid 文件 rnd_pos 回表
            thd, thd->mem_root, Mem_root_array<TABLE *>{table}, tempfile,
            /*ignore_not_found_rows=*/false, /*has_null_flags=*/false,
            /*examined_rows=*/nullptr);
```

> ⚠️ **勘误**：落盘阈值**不是** `my_tmp_threshold`（全仓不存在，是 5.x 遗留概念），也不是 `thd->variables.*`。单表快路径用的是 `DISK_BUFFER_SIZE = IO_SIZE * 16 = 4096 * 16 = 65536`（64 KiB，`include/my_io.h:159/164`）。且文件是**惰性创建**的——写满 buffer 才触发 `my_b_flush_io_cache()` → `real_open_cached_file()`。
>
> ★ 多表路径的阈值则是 `min(tmp_table_size, max_heap_table_size)`，转成 `share->max_rows = 阈值 / reclength`（`sql/sql_tmp_table.cc:2206-2215`）；heap 撑爆后由 `create_ondisk_from_heap()` 转 InnoDB/MyISAM。
>
> ⚠️ 另：多表 UPDATE 在 8.0.39 **不再用 `Unique` 去重**，改成在 rowid 字段上建**唯一索引**（`tmp_param->group_parts = 1`、`group->item = &temp_fields.front()`），重复由 `HA_ERR_FOUND_DUPP_KEY` 直接吞掉；另有 `check_unique_constraint()` 做 hash 预检。`Unique` 现在主要服务 DELETE。

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

**语句级提交的落点**（`sql/sql_parse.cc:4918-4934`，`dispatch_command` 收尾）：

```cpp
    if ((thd->is_error() && !early_error_on_rep_command) ||
        (thd->variables.option_bits & OPTION_MASTER_SQL_ERROR))
      trans_rollback_stmt(thd);
    else {
      thd->get_stmt_da()->set_overwrite_status(true);
      trans_commit_stmt(thd);          // ★★ 每条语句末尾走的是【语句级】提交
      thd->get_stmt_da()->set_overwrite_status(false);
    }
```

```cpp
// sql/transaction.cc:513
bool trans_commit_stmt(THD *thd, bool ignore_global_read_lock) {
  assert(!thd->in_sub_stmt);
  thd->get_transaction()->merge_unsafe_rollback_flags();

  if (thd->get_transaction()->is_active(Transaction_ctx::STMT)) {
    res = ha_commit_trans(thd, false, ignore_global_read_lock);   // ★★ all = false
    if (!thd->in_active_multi_stmt_transaction())
      trans_reset_one_shot_chistics(thd);
  } else if (tc_log)
    res = tc_log->commit(thd, false);

  thd->get_transaction()->reset(Transaction_ctx::STMT);   // ★ 只清语句级上下文
  return res;
}
```

**为什么这就是"不隐式提交"**（`sql/handler.cc:1615`）：

```cpp
int ha_commit_trans(THD *thd, bool all, bool ignore_global_read_lock) {
  /*
    'all' means that this is either an explicit commit issued by
    user, or an implicit commit issued by a DDL.
  */
  Transaction_ctx *trn_ctx = thd->get_transaction();
  Transaction_ctx::enum_trx_scope trx_scope =
      all ? Transaction_ctx::SESSION : Transaction_ctx::STMT;

  /*
    "real" is a nick name for a transaction for which a commit will
    make persistent changes. E.g. a 'stmt' transaction inside a 'all'
    transaction is not 'real': even though it's possible to commit it,
    the changes are not durable as they might be rolled back if the
    enclosing 'all' transaction is rolled back.
  */
  bool is_real_trans = all || !trn_ctx->is_active(Transaction_ctx::SESSION);
  ...
  assert(!trn_ctx->is_active(Transaction_ctx::STMT) || !all);
```

> ★★ **答案就在这三行**：
> - 事务里（`BEGIN` 之后）SESSION 事务活跃 → `is_real_trans = false` → **不落盘、不真提交**
> - `autocommit=1` 且没开事务 → SESSION 不活跃 → `is_real_trans = true` → 这次语句级提交顺带把整个事务提交掉
>
> 这就是"autocommit=1 时每条 DML 自己提交"和"显式事务里 DML 不提交"的**同一段代码**解释。
>
> ★ 真正会**隐式提交**的是 DDL：它们调 `trans_commit_implicit()` / `trans_commit()`（`all = true`）。另外还有一条硬约束 `assert(!is_active(STMT) || !all)`——语句事务还挂着时绝不允许提交 SESSION 事务。

**对比：真正的提交**（`sql/transaction.cc:233`）：

```cpp
bool trans_commit(THD *thd, bool ignore_global_read_lock) {
  thd->server_status &=
      ~(SERVER_STATUS_IN_TRANS | SERVER_STATUS_IN_TRANS_READONLY);
  res = ha_commit_trans(thd, true, ignore_global_read_lock);   // ★ all = true
```

---

### 6.7 系统变量与状态变量（逐个核实）

| 变量 | 结论 | 定义处 / 说明 |
|---|---|---|
| `sql_safe_updates` | ✅ 存在 | `sql/sys_vars.cc:5582`；`Sys_var_bit`，位 `OPTION_SAFE_UPDATES`，**默认 false**，SESSION |
| `safe_updates` | ❌ **不存在** | 它是 `sql_safe_updates` 的命令行等价写法，不是独立 Sys_var 对象 |
| `updatable_views_with_limit` | ✅ 存在 | `sql/sys_vars.cc:5068`；enum {NO, YES}，**默认 YES(1)**，SESSION |
| `optimizer_switch` 的 `condition_fanout_filter` | ✅ 存在 | 名字数组 `optimizer_switch_names[]`，`sql/sys_vars.cc:3465` |
| `max_join_size` | ✅ 存在 | `sql/sys_vars.cc:3057`；默认 `HA_POS_ERROR`（不限制），`ON_UPDATE(fix_max_join_size)` |
| `large_files_support` | ✅ 存在 | `sql/sys_vars.cc:2392`；**READ_ONLY + NON_PERSIST**，默认 `sizeof(my_off_t) > 4` |
| `myisam_repair_threads` | ❌ **不存在** | 8.0.39 全部 `.cc` 0 命中，只剩 mysql-test 遗留文件 |

```cpp
// sql/sys_vars.cc:5582
static Sys_var_bit Sys_safe_updates("sql_safe_updates", "sql_safe_updates",
                                    HINT_UPDATEABLE SESSION_VAR(option_bits),
                                    NO_CMD_LINE, OPTION_SAFE_UPDATES,
                                    DEFAULT(false));
//  ★ 不是独立 bool 字段，而是 option_bits 里的一个位
//  ★ 生效点：DeleteRowsIterator::Init() → CheckSqlSafeUpdate()（sql_delete.cc:966）
//           多表 UPDATE：sql_update.cc:2220  CheckSqlSafeUpdate(thd, join)

// sql/sys_vars.cc:5066
// why ENUM and not BOOL ?
static const char *updatable_views_with_limit_names[] = {"NO", "YES", nullptr};
static Sys_var_enum Sys_updatable_views_with_limit(
    "updatable_views_with_limit", ...
    updatable_views_with_limit_names, DEFAULT(true));   // ★ 下标 1 = "YES"
```

**状态变量**（`sql/mysqld.cc` 的 `status_vars[]`，`:9768-9822`）：

```cpp
    {"Handler_delete", (char *)offsetof(System_status_var, ha_delete_count),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    {"Handler_update", (char *)offsetof(System_status_var, ha_update_count),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    {"Handler_write", (char *)offsetof(System_status_var, ha_write_count),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    {"Handler_read_first", ... ha_read_first_count ...},
    {"Handler_read_key",   ... ha_read_key_count   ...},
    {"Handler_read_next",  ... ha_read_next_count  ...},
    {"Handler_read_rnd",   ... ha_read_rnd_count   ...},
    {"Handler_read_rnd_next", ... ha_read_rnd_next_count ...},
```

> ⚠️ **勘误**：`Handler_write` / `Handler_update` / `Handler_delete` **不是 server 的 `ha_write_row()` / `ha_update_row()` / `ha_delete_row()` 自增的**——`sql/handler.cc:8060-8088` 里没有任何计数语句。真实自增点在各引擎自己的实现里，调 `ha_statistic_increment()`（`:2696`）：
> ```cpp
> void handler::ha_statistic_increment(ulonglong System_status_var::*offset) const {
>   if (table && table->in_use) (table->in_use->status_var.*offset)++;
> }
> ```
> InnoDB 在 `storage/innobase/handler/ha_innodb.cc:9941`（`ha_innobase::delete_row`，起点 `:9917`）；MyISAM `ha_myisam.cc:1481`；MEMORY `ha_heap.cc:244`；TempTable `handler.cc:698`。全部 `Handler_*` 作用域都是 `SHOW_SCOPE_ALL`（GLOBAL + SESSION）。

### 6.8 外键级联：server 侧只有 prelocking

⚠️ **勘误**：**MySQL server 侧没有任何"级联删除执行代码"**。server 只做两件事：

**① prelocking**（`sql/sql_base.cc:4416`，真实函数名 `process_table_fks()`）：

```cpp
  if (is_insert || is_update) {
    for (TABLE_SHARE_FOREIGN_KEY_INFO *fk = share->foreign_key; ...) {
      (void)sp_add_used_routine(prelocking_ctx, thd->stmt_arena,
          Sroutine_hash_entry::FK_TABLE_ROLE_PARENT_CHECK, ...);
    }
  }
  if (is_update || is_delete) {
    for (TABLE_SHARE_FOREIGN_KEY_PARENT_INFO *fk_p = share->foreign_key_parent; ...) {
      if (RULE_NO_ACTION || RULE_RESTRICT)
        → FK_TABLE_ROLE_CHILD_CHECK;
      if (RULE_CASCADE || RULE_SET_NULL || RULE_SET_DEFAULT)
        → FK_TABLE_ROLE_CHILD_UPDATE;
      if (is_delete && fk_p->delete_rule == dd::Foreign_key::RULE_CASCADE)
        → FK_TABLE_ROLE_CHILD_DELETE;      // ★ 只有 delete_rule == CASCADE
    }
  }
```

> ★ 为什么必须 prelocking：级联动作是**引擎（InnoDB）内部发起**的，server 不知道它会碰哪些表，只能提前把父/子表打开并加 MDL 锁。
>
> ★ 这也解释了 `sql_delete.cc:1131` 那句注释：*"Ignore any rows not found in reference tables as they may already have been deleted by foreign key handling"* → delayed delete 回表时行可能已被 FK 级联删掉，所以 `init_table_iterator(ignore_not_found_rows=true)`。

**② 错误码翻译**（`sql/handler.cc:4199` / `4337`）：

```cpp
    case HA_ERR_ROW_IS_REFERENCED:        // 1451，删父行被子表引用
      thd->push_internal_handler(&foreign_key_error_handler);
      get_error_message(error, &str);
      my_error(ER_ROW_IS_REFERENCED_2, errflag, str.c_ptr_safe());
      thd->pop_internal_handler();
      return;
    case HA_ERR_NO_REFERENCED_ROW:        // 1452，插/改子行父键不存在
      ... my_error(ER_NO_REFERENCED_ROW_2, errflag, str.c_ptr_safe());
```

**级联执行在 InnoDB 侧**（真实函数名）：

| 符号 | 位置 |
|---|---|
| `row_ins_check_foreign_constraint()` | `storage/innobase/row/row0ins.cc:1369` |
| `row_upd_check_references_constraints()` | `storage/innobase/row/row0upd.cc:180`（DELETE 前检查子表引用） |
| `row_ins_cascade_ancestor_updates_table()` | `row0ins.cc:401` |
| `row_ins_cascade_n_ancestors()` | `row0ins.cc:425` |
| `row_ins_cascade_calc_update_vec()` | `row0ins.cc:446` |

```cpp
  /* We do not allow cyclic cascaded updating (DELETE is allowed,
  but not UPDATE) of the same table, as this can lead to an infinite cycle. */
  if (!cascade->is_delete &&
      row_ins_cascade_ancestor_updates_table(cascade, table)) {
    err = DB_ROW_IS_REFERENCED;
    row_ins_foreign_report_err("Trying an update, possibly causing a cyclic"
                               " cascaded update\n in the child table,", ...);
  }
  if (row_ins_cascade_n_ancestors(cascade) >= FK_MAX_CASCADE_DEL) {
    err = DB_FOREIGN_EXCEED_MAX_CASCADE;
    row_ins_foreign_report_err("Trying a too deep cascaded delete or update\n", ...);
  }
```

> ★ 级联深度上限 `FK_MAX_CASCADE_DEL`，超限报 `DB_FOREIGN_EXCEED_MAX_CASCADE`（`ha_innodb.cc:2114` 翻成 `HA_ERR_FOREIGN_EXCEED_MAX_CASCADE`）。是否检查外键由 `trx->check_foreigns` 控制，来源是 `OPTION_NO_FOREIGN_KEY_CHECKS`。

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

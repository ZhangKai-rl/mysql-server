# 13 函数索引与多值索引

> 8.0 的两个索引新特性：**函数索引**（8.0.13+，`CREATE INDEX idx ON t ((LOWER(col)))`）和**多值索引**（8.0.17+，`WHERE 1 MEMBER OF (arr->'$.x')`）。共同本质：**把表达式/数组变成一个隐藏生成列，索引建在隐藏列上**。本篇讲它们如何创建、优化器如何匹配、执行期如何工作。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、共同本质：隐藏生成列](#一共同本质隐藏生成列)
- [二、函数索引](#二函数索引)
- [三、多值索引](#三多值索引)
- [四、GC substitution：优化器如何匹配](#四gc-substitution优化器如何匹配)
- [五、执行期去重（多值索引）](#五执行期去重多值索引)
- [六、两者对比](#六两者对比)

---

## 设计思想与理论基础

### 解决什么问题

传统索引只能索引**列**。但查询条件常作用在表达式上：

```sql
WHERE LOWER(name) = 'x'
WHERE CAST(data->>'$.id' AS UNSIGNED) = 5
```

这类条件**用不上普通索引**——索引里存的是原始值，不是表达式的结果。两类扩展应运而生：

- **函数索引**（functional index）：索引表达式的结果
- **多值索引**（multi-valued index）：索引 JSON 数组的**每一个元素**

**★ 官方自述：函数索引就是隐藏生成列**（`sql/error_handler.h:296`，★ 全篇题眼）：

```cpp
/**
  The purpose of this error handler is to print out more user friendly error
  messages when an error regarding a functional index happens. Since functional
  indexes are implemented as hidden generated columns with an auto-generated
  name, we would end up printing errors like "Out of range value for column
  '912ec803b2ce49e4a541068d495ab570' at row 0". With this error handler, we
  end up printing something like "Out of range value for functional index
  'functional_index_2' at row 0" instead.
  ...
*/
class Functional_index_error_handler : public Internal_error_handler {
```
> ★★ 官方注释原话：**"functional indexes are implemented as hidden generated columns with an auto-generated name"**。
> ★ 顺带说明：隐藏列名对用户不可读（一串 `!hidden!...` 或历史版本的 hash），所以专门做了一个**错误码翻译器**（见 2.2 节）。

**★ 第二个官方自述**（`sql/sql_table.cc:7741`，`add_functional_index_to_create_list` 的 Doxygen）：

```cpp
/**
  Prepares a functional index by adding a hidden indexed generated column for
  the key part.

  A functional index is implemented as a hidden generated column over the
  expression specified in the index, and the hidden generated column is then
  indexed. This function adds a hidden generated column to the Create_list,
  and updates the key specification to point to this new column. ...
```
> ⚠️ 同一段注释下面写"the generated column is given a name that is **a hash of the key name**" —— 这是**过时说法**，实际代码生成的是明文 `!hidden!<index_name>!<key_part>!<counter>`（见 §一）。

**★ 版本证据：WL#1075 + I_S 版本 80013**（`sql/dd/info_schema/metadata.h:75`，★ 函数索引 8.0.13 引入的最硬证据）：

```cpp
  - WL#1075 adds one column to INFORMATION_SCHEMA.STATISTICS: "EXPRESSION".
    This column prints out the expression for functional key parts, or SQL NULL
    if it is a regular key part. For functional key parts, COLUMN_NAME is set to
    SQL NULL.

  80013: Published in 8.0.13
```
```cpp
/* sql/dd/dd_version.h:113 —— hidden 列从 BOOL 扩成 ENUM，正是函数索引的产物 */
  - Bug#27499518 changed the data type for the column 'hidden' in table
    'columns' from BOOL to ENUM('Visible', 'SE', 'SQL').
```
```cpp
/* sql/dd/dd_version.h:195 —— 8.0.23 再由 WL#10905 加 'USER'（INVISIBLE 列） */
  - WL#10905 adds new hidden type 'USER' to mysql.columns.hidden column.
```
> ★ 这条时间线正好对应 `enum_hidden_type` 的 4 个取值。**多值索引（8.0.17）在源码里没有对应的 WL 编号**（grep 无命中），文档若要写 WL 号需引外部资料。

### 共同本质：隐藏生成列（本篇最重要的思想）

MySQL 实现这两者的方式出人意料地统一——**都编译成一个隐藏的虚拟生成列**：

```sql
CREATE INDEX idx ON t ((LOWER(name)))
      ↓ 内部等价于
ADD COLUMN !hidden!idx!0!0  AS (LOWER(name)) VIRTUAL   -- HT_HIDDEN_SQL
CREATE INDEX idx ON t (!hidden!idx!0!0)
```

源码依据：`add_functional_index_to_create_list()` 走的是**与生成列相同的 DDL 路径**——`generate_create_field()` 推类型、标记 `hidden = HT_HIDDEN_SQL` 且 `stored_in_db = false`、索引 key part 改写为指向该隐藏列。

**为什么这个设计好**：

1. **零新增机制**——表达式求值、类型推导、DDL 校验、binlog 记录全部复用既有生成列框架
2. **优化器无需特殊处理**——它看到的就是"一个普通列 + 一个普通索引"，匹配靠 GC substitution（本文第四章）
3. **`HT_HIDDEN_SQL` 保证对 SQL 不可见**——`SELECT *` 不返回、`DESCRIBE` 不显示

**代价**：隐藏列**占用列数配额**（与字段数上限等约束共享），且继承了生成列的全部限制。

**★★★ 核心代码：`add_functional_index_to_create_list()` 造出隐藏列的 6 行**（`sql/sql_table.cc:7853`，★ 本篇最重要的源码）：

```cpp
  cr->field_name = field_name;
  cr->field = nullptr;
  cr->hidden = dd::Column::enum_hidden_type::HT_HIDDEN_SQL;
  cr->stored_in_db = false;

  Value_generator *gcol_info = new (thd->mem_root) Value_generator();
  gcol_info->expr_item = kp->get_expression();
  gcol_info->set_field_stored(false);
  gcol_info->set_field_type(cr->sql_type);
  cr->gcol_info = gcol_info;
  ...
  alter_info->flags |= Alter_info::ALTER_ADD_COLUMN;
  kp->set_name_and_prefix_length(field_name, 0);   /* ★★ key part 从"表达式"改指回"列名" */
  return cr;
```
> ★★ 最后一行 `kp->set_name_and_prefix_length(field_name, 0)` 是"降维"的收尾：**从此优化器/存储引擎看到的就只是一个普通二级索引**，表达式被完全藏起来了。

**★ 前置的五步流水线**（`sql/sql_table.cc:7763`）：

```cpp
  // A functional index cannot be a primary key
  if (key_spec->type == KEYTYPE_PRIMARY) {
    my_error(ER_FUNCTIONAL_INDEX_PRIMARY_KEY, MYF(0));
    return nullptr;                                    /* ① 拒主键 */
  }

  // If the key isn't given a name explicitly by the user, we must auto-generate
  // a name here. ... since we want the hidden generated column name to be
  // based on the index name.
  if (key_spec->name.str == nullptr) {
    std::string key_name;  int count = 2;
    key_name.assign("functional_index");
    while (key_name_exists(alter_info->key_list, key_name, nullptr)) {
      key_name.assign("functional_index_");
      key_name.append(std::to_string(count++));        /* ② 自动命名 functional_index[_N] */
    }
    ...
  }

  const Access_bitmask saved_privilege = thd->want_privilege;
  thd->want_privilege = SELECT_ACL;                    /* ③ 权限临时降到 SELECT_ACL */

  {
    auto handler_guard = create_scope_guard(           /* scope guard 保证还原 */
        [thd, saved_privilege]() { thd->want_privilege = saved_privilege; });
    Functional_index_error_handler error_handler(
        {key_spec->name.str, key_spec->name.length}, thd);

    Item *expr = kp->get_expression();
    if (expr->type() == Item::FIELD_ITEM) {
      my_error(ER_FUNCTIONAL_INDEX_ON_FIELD, MYF(0));  /* ④ 不能是裸字段 */
      return nullptr;
    }

    if (pre_validate_value_generator_expr(
            kp->get_expression(), key_spec->name.str, VGS_GENERATED_COLUMN)) {
      return nullptr;                                  /* ★★ 复用生成列的合法性校验 */
    }

    Replace_field_processor_arg replace_field_argument(
        thd, &alter_info->create_list, create_info, key_spec->name.str);
    if (expr->walk(&Item::replace_field_processor, enum_walk::PREFIX,
                   reinterpret_cast<uchar *>(&replace_field_argument))) {
      return nullptr;
    }
    if (kp->resolve_expression(thd)) return nullptr;    /* ⑤ 定类型 */
  }

  const char *field_name = make_functional_index_column_name(
      {key_spec->name.str, key_spec->name.length}, key_part_number,
      alter_info->create_list, thd->stmt_arena->mem_root);
```
> ★★ 第 ④ 步的 `pre_validate_value_generator_expr(..., VGS_GENERATED_COLUMN)` 是**"函数索引复用生成列全套基础设施"的直接证据**——连校验函数都是同一个。
> ★ 为什么必须先给索引起名：因为隐藏列名由索引名派生，而普通索引的名字要到 `prepare_key()` 才定——那太晚了。

**★ 语法产生式真名是 `key_part_with_expression`**（`sql/sql_yacc.yy:8092`，⚠️ 不是 `key_part_spec`）：

```cpp
key_part_with_expression:
          key_part
        | '(' expr ')' opt_ordering_direction
          {
            $$= NEW_PTN PT_key_part_specification($2, $4);
            if ($$ == NULL)
              MYSQL_YYABORT;
          }
```
> ★ `key_part_with_expression` **包含** `key_part` 作为第一候选 —— 普通索引和函数索引走**同一条 key_list**，区别只在语义层。
> ★ 语义层是 `PT_key_part_specification`（`sql/parse_tree_nodes.h:2046`），两个构造重载就是分野：一个存 `LEX_CSTRING column_name`，一个存 `Item *expression`。
> ⚠️ `INDEX((a+1))`（函数索引）与 `INDEX(a(10))`（前缀长度）在 yacc 里是**两条不同产生式**。

**★ 一个"鸡生蛋"问题的解法**（`sql/field.h:1872`，★ 理解为什么校验分散在多阶段）：

```cpp
/**
  This class is a substitute for the Field classes during CREATE TABLE

  When adding a functional index at table creation, we need to resolve the
  expression we are indexing. All functions that references one or more
  columns expect a Field to be available. But during CREATE TABLE, we only
  have access to Create_field. So this class acts as a substitute for the
  Field classes so that expressions can be properly resolved. Thus, trying
  to call store or val_* on this class will cause an assertion.
*/
class Create_field_wrapper final : public Field {
```
> ★ CREATE TABLE 时只有 `Create_field` 没有 `Field`，而表达式 resolve 必须要 `Field` ⇒ 造一个不可 `store`/`val_*` 的假 `Field`。

**★ `Value_generator` 是三合一**（`sql/field.h:468`）：

```cpp
enum Value_generator_source : short {
  VGS_GENERATED_COLUMN = 0,  // Value generator for GENERATED_COLUMN.
  VGS_DEFAULT_EXPRESSION,    // Value generator for Default expression.
  VGS_CHECK_CONSTRAINT       // Value generator for check constraints.
};
```
> ★ 生成列 / DEFAULT 表达式 / CHECK 约束共用同一个 `Value_generator`，靠 `source` 区分错误码。函数索引传的是 `VGS_GENERATED_COLUMN`。
> ★ 注意 `expr_item` 的注释：TABLE_SHARE 里为 NULL、TABLE 里才有值（**Item 树不可共享**）——这解释了为什么函数索引的表达式要在每次开表时重新 parse。

**★ `KEY::is_functional_index()`：索引结构完全无感**（`sql/key.cc:50`）：

```cpp
bool KEY::is_functional_index() const {
  for (uint i = 0; i < actual_key_parts; ++i) {
    if (key_part[i].field->is_field_for_functional_index()) {
      return true;
    }
  }
  return false;
}
```
> ★ 判定落在"key part 的 field 是不是隐藏函数索引列"上 —— 再次印证"函数索引 = 普通索引 + 隐藏生成列"。

### 多值索引的特殊性：一对多

多值索引与函数索引的关键差异：JSON 数组的**每个元素**都要建索引项 ⇒ **一行产生多个索引项**。两个理论后果：

1. 行与索引项是**一对多**，同一个索引项可能对应多行（多个数组含相同元素）
2. 因此查询可能返回**重复行** ⇒ 必须去重（本文第五章讲执行期去重）

这也是为什么多值索引的匹配只支持特定函数——`MEMBER OF` / `JSON_CONTAINS` / `JSON_OVERLAPS`：只有它们的语义能明确表达"数组与值的关系"。

**★ 一对多的物理展开**（`storage/innobase/include/row0row.h:318`，★ 详见 §3.1）：

```cpp
  /** Get next index entry based on next multi-value data.
  If the previous value is null, then always no next.
  @return next index entry, or nullptr if no more multi-value data */
  dtuple_t *next() {
    ...
    const auto len = m_mv_data->data_len[m_pos];
    dfield_set_data(field, m_mv_data->datap[m_pos], len);
    ++m_pos;
    return (m_entry);       /* ★ 同一个 entry 模板，只改数据指针 → 下一条索引记录 */
  }
```
> ★★ 一行 N 个数组元素 ⇒ **N 条二级索引记录，共享同一个 rowid**。这是"一对多"的全部实现，也是后续所有复杂性（去重、不能覆盖、不能有序）的根源。

**★ 由此推出的三条连锁后果**：

| 后果 | 原因 | 实现 |
|---|---|---|
| 查询可能返回重复行 | 同一 rowid 出现 N 次 | `m_unique` + `HA_ERR_KEY_NOT_FOUND` 续读（§五） |
| 不能做覆盖索引 | 去重要 rowid，必须额外读 PK | `index_range_scan.cc:291`；且 `HA_KEYREAD_ONLY` 被清 |
| 不能有序检索 | 数组元素间**没有定义顺序** | `Field_typed_array::key_cmp` 恒返回 -1；`HA_READ_ORDER` 被清 |

```cpp
/* sql/field.h:4197 —— ★ 官方对"数组无序"的表述 */
  /**
    These methods are used by handler to prevent returning a row past the
    end_range during range access. Since there's no order defined for sorting
    set of arrays, always return -1 here, allowing all records fetched from
    SE to be returned to server. They will be filtered by WHERE condition later.
  */
  int key_cmp(const uchar *, const uchar *) const override { return -1; }
```

**★ 引擎支持面：只有 InnoDB**（`storage/innobase/handler/ha_innodb.cc:2981`）：

```cpp
          HA_DESCENDING_INDEX | HA_MULTI_VALUED_KEY_SUPPORT |
```
> ★ `HA_MULTI_VALUED_KEY_SUPPORT` 全仓命中仅 3 处：定义（`handler.h:515`）、校验（`sql_table.cc:4842`）、声明（`ha_innodb.cc:2981`）。**MyISAM / NDB / TempTable 均未声明**。

### 他库对比

- PostgreSQL 从很早就有表达式索引（`CREATE INDEX ... ON t (lower(name))`），是**一等公民**概念
- MySQL 直到 **8.0.13** 才引入函数索引，多值索引更晚（8.0.17）
- **差异在约束而非功能**：MySQL 是"借道隐藏生成列"实现，因此继承生成列的限制（不能用不确定函数如 `RAND()`/`NOW()`、不能含子查询、不能建在 LOB 上、不能作主键）

**★ MySQL 的"借道"在源码里的代价**（★ 与 PG 的一等公民实现对比）：

| 维度 | PostgreSQL（一等公民） | MySQL（借道隐藏生成列） |
|---|---|---|
| 表达式存在哪 | 索引自己的 `indexprs` | **隐藏生成列的 `generation_expression`**；`dd::Index` 无 expression 字段 |
| 错误码 | 索引自己的 | 底层抛 `ER_GENERATED_COLUMN_*`，靠 `Functional_index_error_handler` 翻译 |
| 优化器 | 直接匹配 index expression | 靠 GC substitution 反查等价隐藏列（第四章） |
| 类型推导 | — | Item → 临时表 Field → Create_field（`generate_create_field` 绕一圈） |
| CREATE TABLE 期 | — | 需要 `Create_field_wrapper` 假 Field 来 resolve |

> ★ 反过来看，**"零新增机制"是收益**：表达式求值、类型推导、DDL 校验、binlog 记录全部复用既有生成列框架。

**★ 8.0.13 / 8.0.17 的版本证据**（`sql/dd/info_schema/metadata.h:75` / `sql/sql_table.cc:7725`）：

```cpp
  80013: Published in 8.0.13     /* ★ WL#1075 函数索引在这一版引入 */
```
```cpp
  // From version 8.0.18, the server will sort functional index columns last in
  // the table.                   /* ★ 8.0.18 起的隐藏列排序行为变更 */
```
> ⚠️ **多值索引（8.0.17）在源码里没有对应 WL 编号**（grep `WL ?#?[0-9]{4,5}` 在相关文件只命中 WL#7909 / WL#9831，均属 JSON 规范）。文档写 WL 号需引外部资料，不能标为源码证据。

### 代价要点

1. 生成列的限制全部继承（见上）
2. 多值索引的**执行期去重**是额外代价
3. 隐藏列占配额

**★ 代价 1：range over 多值索引 = 一串 MEMBER OF 的 OR**（`sql/range_optimizer/range_optimizer.cc:1552`）：

```cpp
  // Range scans over multi-valued indexes use a sequence of MEMBER OF
  // predicates ORed together.
  if (key_part->field->is_array()) {
    print_key_value(out, key_part, min_key);
    out->append(STRING_WITH_LEN(" MEMBER OF ("));
    const std::string expression = ItemToString(
        down_cast<Item_func *>(key_part->field->gcol_info->expr_item)
            ->get_arg(0));  // Strip off CAST(... AS <type> ARRAY).
```

**★ 代价 2：不相交的两个 range 对多值索引不意味着恒假**（`sql/range_optimizer/tree.cc:960`）：

```cpp
  // Two non-overlapped key ranges for multi-valued index do not mean
  // an always false condition.
  // For example, "1 member of(f) AND 2 member of(f)" for f=[1, 2].
  if (key1->root->field->is_array() || key2->root->field->is_array()) {
    return and_all_keys(param, key1, key2);   /* ★ 普通索引此时可直接判 false */
  }
```
> ★★ 普通索引能做的重要剪枝，多值索引**做不了**。

**★ 代价 3：DECIMAL 必须为多值索引保留精度**（`sql/field.h:2133`）：

```cpp
  /**
    Normally, the underlying decimal code will degrade values' excessive
    precision. ... This is fine for general purpose, but isn't usable for
    multi-valued index. Field_typed_array uses a field for conversion and it
    expects the value read from it to be exactly same as it would be stored
    in SE, i.e with preserved precision. Otherwise, SE won't be able to
    index it.
    TRUE here tells underlying DECIMAL reading code to keep the precision as
    is.
  */
  bool m_keep_precision{false};
```
> ★ 多值索引逼着 DECIMAL 读路径保留精度，**这对普通查询是额外负担**。

**★ 代价 4：重复值在写入时就要去重**（`sql/field.cc:9905`）：

```cpp
        /*
          Non-strict mode issue:
          While consisting of unique values, bad input can cause duplicates
          after coercion. Remove them, as SE doesn't expect dups in the array.
          This is why we need to sort & remove duplicates only after
          processing all keys.      /* ★ 必须等所有值 coerce 完才能去重 */
        */
```

---

## 一、共同本质：隐藏生成列

两者走**同一条 DDL 代码路径** `add_functional_index_to_create_list()`（`sql_table.cc:7763`）：

1. 解析索引表达式，`generate_create_field()` 推导出列类型
2. 创建 `Create_field`，标记 `hidden = HT_HIDDEN_SQL`、`stored_in_db = false`（虚拟列）
3. `gcol_info->expr_item = 表达式`（保存原始表达式）
4. `kp->set_name_and_prefix_length(field_name, 0)` 把索引 key part 改写成指向隐藏列

**隐藏列命名**（`make_functional_index_column_name`，:7690）：`!hidden!<index_name>!<key_part_number>!<counter>`。

**隐藏标志**（`field.h:875`）：

```cpp
bool is_field_for_functional_index() const {
  return hidden() == dd::Column::enum_hidden_type::HT_HIDDEN_SQL && gcol_info != nullptr;
}
```

`hidden` 分四类：`HT_VISIBLE` / `HT_HIDDEN_USER`（用户 INVISIBLE）/ `HT_HIDDEN_SE`（引擎内部）/ `HT_HIDDEN_SQL`（SQL 层内部，函数索引和多值索引用这个）。

**★ `enum_hidden_type` 真名与取值**（`sql/dd/types/column.h:95`，⚠️ 嵌在 `dd::Column` 里不是全局枚举）：

```cpp
  enum class enum_hidden_type {
    /// The column is visible (a normal column)
    HT_VISIBLE = 1,
    /// The column is completely invisible to the server
    HT_HIDDEN_SE = 2,
    /// The column is visible to the server, but hidden from the user.
    /// This is used for i.e. implementing functional indexes.
    HT_HIDDEN_SQL = 3,
    /// User table column marked as INVISIBLE by using the column visibility
    /// attribute. Column is hidden from the user unless it is explicitly
    /// referenced in the statement. Column is visible to the server.
    HT_HIDDEN_USER = 4
  };
```
> ★★ `HT_HIDDEN_SQL` 与 `HT_HIDDEN_USER` 的根本差别：**后者显式引用时用户可见，前者永远不可见**（见下文 `Item_field::fix_fields`）。

**★ 命名规则明文，不是 hash**（`sql/sql_table.cc:7690`，⚠️ 上方 Doxygen 说 hash 是**过时的**）：

```cpp
/**
  Create a name for the hidden generated column that represents the functional
  key part.

  The name is a string on the form `!hidden!index_name!key_part!counter`. The
  counter is usually 0, but is incremented until a unique name is found, in the
  unlikely event that there is another column with the same name. The index_name
  part may be truncated to make sure the column name does not exceed the maximum
  column name length (NAME_CHAR_LEN).
  ...
*/
static const char *make_functional_index_column_name(
    std::string_view key_name, unsigned key_part_number,
    const List<Create_field> &fields, MEM_ROOT *mem_root) {
  for (unsigned count = 0;; ++count) {
    string name("!hidden!");
    name += key_name;
    string suffix("!");
    suffix += to_string(key_part_number);
    suffix += '!';
    suffix += to_string(count);

    // If the name is so long that we hit the NAME_CHAR_LEN limit, truncate the
    // index name part, so that there is enough room to add the suffix with the
    // key part number and the counter. (If we had truncated the counter, we
    // could loop forever because the generated name is the same in each
    // iteration.)
    name.resize(min(name.size(), NAME_CHAR_LEN - suffix.size()));
    name.append(suffix);
    ...
    if (column_name_exists(fields, name)) continue;
    return strmake_root(mem_root, name.data(), name.size());
  }
}
```
> ★ 截断时**只截 index_name，绝不截 counter**，否则死循环。名字以 `!hidden!` 开头（且含 `!`），用户几乎不可能撞名——但仍用 `column_name_exists` 循环防重。

**★★ "对 SQL 不可见"的三处实现**（★ 分别对应 SELECT * / SHOW CREATE / 显式引用）：

**① `SELECT *` 不返回**（`sql/sql_base.cc:9378`，⚠️ **不是** `visible_field_count`）：

```cpp
        /*
          If the column is hidden from users and not used in USING clause of
          a join, do not add this column in place of '*'.
        */
        bool is_hidden = field->field->is_hidden();
        is_hidden &= (tables->join_using_fields == nullptr ||
                      !test_if_string_in_list(field->field_name,
                                              tables->join_using_fields));
        if (is_hidden) continue;
```
> ★ 用的是 `is_hidden()`（含 `HT_HIDDEN_USER`），所以 INVISIBLE 列也不会被 `*` 展开。

**② `SHOW CREATE TABLE` 跳过列定义，但索引里还原成表达式**（`sql/sql_show.cc:2049` / `:2253`）：

```cpp
  for (ptr = first_field; (field = *ptr); ptr++) {
    // Skip hidden system fields.
    if (field->is_hidden_by_system()) continue;      /* ★ 列定义里跳过 */
```
```cpp
        // If this fields represents a functional index, print the expression
        // instead of the column name.
        if (key_part->field->is_field_for_functional_index()) {
          assert(key_part->field->gcol_info);
          StringBuffer<STRING_BUFFER_USUAL_SIZE> s;
          s.set_charset(system_charset_info);
          key_part->field->gcol_info->print_expr(thd, &s);
          packet->append("(");
          packet->append(s);
          packet->append(")");                       /* ★ 打印成 ((expr)) */
        } else {
          append_identifier(thd, packet, key_part->field->field_name, ...);
        }
```
> ★★ **列定义被隐藏，但索引定义里被还原成 `(expr)`** —— 用户看到的是 `KEY idx ((a+1))`，看不到底下的列。

**③ 显式引用 → 报"列不存在"**（`sql/item.cc:5860`，★ `HT_HIDDEN_SQL` 与 `HT_HIDDEN_USER` 的行为分水岭）：

```cpp
  if (from_field->is_hidden_by_system()) {
    /*
      This field is either hidden by the storage engine or SQL layer. In
      either case, report column "not found" error.
    */
    my_error(ER_BAD_FIELD_ERROR, MYF(0), from_field->field_name, thd->where);
    return true;
  }
```
> ★ `SELECT !hidden!idx!0!0 FROM t` → `ER_BAD_FIELD_ERROR`（"Unknown column"）。
> ★★ 注意用的是 `is_hidden_by_system()`（**不含** `HT_HIDDEN_USER`）—— 所以 **INVISIBLE 列显式引用是允许的，函数索引隐藏列不允许**。

**★ 五个可见性访问器**（`sql/field.h:829`，★ 含一个实用的调试开关）：

```cpp
  bool is_hidden() const {
    return hidden() != dd::Column::enum_hidden_type::HT_VISIBLE &&
           DBUG_EVALUATE_IF("show_hidden_columns", false, true);
  }
  bool is_hidden_by_system() const {
    return (hidden() == dd::Column::enum_hidden_type::HT_HIDDEN_SE ||
            hidden() == dd::Column::enum_hidden_type::HT_HIDDEN_SQL) &&
           DBUG_EVALUATE_IF("show_hidden_columns", false, true);
  }
  bool is_hidden_by_user() const {
    return hidden() == dd::Column::enum_hidden_type::HT_HIDDEN_USER;
  }
  /**
    @returns true if this is a hidden field that is used for implementing
             functional indexes. Note that if we need different types of hidden
             fields in the future (like invisible columns), this function needs
             to be changed so it can distinguish between the different "types"
             of hidden.
  */
  bool is_field_for_functional_index() const {
    return hidden() == dd::Column::enum_hidden_type::HT_HIDDEN_SQL &&
           gcol_info != nullptr;
  }
```
> ★★ **排查函数索引时 `SET debug='+d,show_hidden_columns'` 就能看见隐藏列** —— 这是最实用的调试技巧。
> ⚠️ `is_field_for_functional_index()` 额外要求 `gcol_info != nullptr`，比 `Create_field` 版（`create_field.cc:785`，只看 `HT_HIDDEN_SQL`）更严。

**★ `SHOW KEYS` 里 COLUMN_NAME 是 NULL**（`sql/sql_show.cc:4295`）：

```cpp
      if (key_part->field && key_part->field->is_hidden_by_system()) {
        table->field[TMP_TABLE_KEYS_COLUMN_NAME]->set_null();
      } else {
        table->field[TMP_TABLE_KEYS_COLUMN_NAME]->store(str, strlen(str), cs);
```
> ★ 与 WL#1075 的注释完全对应：函数索引那行 `COLUMN_NAME` 为 NULL，`EXPRESSION` 列才是表达式。

**★ `I_S.COLUMNS` 的 EXTRA**（`sql/sql_show.cc:4105`）：

```cpp
        type.append(STRING_WITH_LEN("STORED GENERATED"), cs);
      else
        type.append(STRING_WITH_LEN("VIRTUAL GENERATED"), cs);
    }
    // Column visibility attribute.
    if (field->is_hidden_by_user()) {
      if (type.length() > 0) type.append(STRING_WITH_LEN(" "), cs);
      type.append(STRING_WITH_LEN("INVISIBLE"), cs);
    }
```

**★ 数据字典层没有函数索引的独立表示**（`sql/dd/types/index.h:93`，★ 负面证据）：

```cpp
  virtual bool is_generated() const = 0;
  virtual void set_generated(bool generated) = 0;
  /////////////////////////////////////////////////////////////////////////
  // hidden.
  /////////////////////////////////////////////////////////////////////////
  virtual bool is_hidden() const = 0;
  virtual void set_hidden(bool hidden) = 0;
```
> ★★ `dd::Index` **没有 expression 字段**。函数索引在 DD 里就是一个普通 `dd::Index`，指向一个 `hidden='SQL'` 的 `dd::Column`。（`is_hidden()` 是给 GIPK 隐藏主键索引用的，与函数索引无关。）

---

## 二、函数索引

### 2.1 创建与校验

`add_functional_index_to_create_list()` 的校验：

```cpp
if (key_spec->type == KEYTYPE_PRIMARY) {          // 不能是主键
  my_error(ER_FUNCTIONAL_INDEX_PRIMARY_KEY, ...);
}
if (expr->type() == Item::FIELD_ITEM) {            // 不能索引裸列
  my_error(ER_FUNCTIONAL_INDEX_ON_FIELD, ...);
}
if (pre_validate_value_generator_expr(...)) return;  // 校验非法函数/变量/子查询
if (is_blob(cr->sql_type)) {                        // 不能索引 LOB
  my_error(ER_FUNCTIONAL_INDEX_ON_LOB, ...);
}
```

**determinism 校验**（`validate_value_generator_expr`，`table.cc:2299`）：
- 非确定性函数（`RAND()/NOW()/UUID()`）→ 拒绝
- 系统变量 `@@x` / 用户变量 `@x` / prepared 参数 → 拒绝
- 子查询 / 存储程序 → 拒绝
- 引用自身 / 排在其后的生成列 / AUTO_INCREMENT 列 → 拒绝（`check_function_as_value_generator`，`item.cc:1110`）

### 2.2 限制

不能是 PRIMARY KEY、不能索引裸字段、不能索引 LOB、必须确定性函数。

**★ 校验入口：复用生成列的 `pre_validate_value_generator_expr()`**（`sql/field.cc:1184`）：

```cpp
bool pre_validate_value_generator_expr(Item *expression, const char *name,
                                       Value_generator_source source) {
  enum error_type { ER_GENERATED_ROW, ER_NAMED_FUNCTION, MAX_ERROR };
  int error_code_map[][MAX_ERROR] = {
      // Generated column
      {ER_GENERATED_COLUMN_ROW_VALUE,
       ER_GENERATED_COLUMN_NAMED_FUNCTION_IS_NOT_ALLOWED},
      // Default expression
      {ER_DEFAULT_VAL_GENERATED_ROW_VALUE,
       ER_DEFAULT_VAL_GENERATED_NAMED_FUNCTION_IS_NOT_ALLOWED},
      // Check constraint
      {ER_CHECK_CONSTRAINT_ROW_VALUE,
       ER_CHECK_CONSTRAINT_NAMED_FUNCTION_IS_NOT_ALLOWED}};

  // ROW values are not allowed
  if (expression->type() == Item::ROW_ITEM) {
    my_error(error_code_map[source][ER_GENERATED_ROW], MYF(0), name);
    return true;
  }
  ...
  if (expression->walk(&Item::check_function_as_value_generator,
                       enum_walk::SUBQUERY_POSTFIX,
                       pointer_cast<uchar *>(&checker_args))) {
    my_error(checker_args.err_code, MYF(0), name,
             checker_args.banned_function_name);
    return true;
  }
```
> ★ 函数索引走 `VGS_GENERATED_COLUMN`（第一列），所以 ROW 值 → `ER_GENERATED_COLUMN_ROW_VALUE`，非法函数 → `ER_GENERATED_COLUMN_NAMED_FUNCTION_IS_NOT_ALLOWED`。

**★★ 白名单式设计：基类一律否决**（`sql/item.cc:790`）：

```cpp
bool Item::check_function_as_value_generator(uchar *checker_args) {
  ...
  func_arg->err_code = func_arg->get_unnamed_function_error_code();
  return true;      /* ★★ 基类无条件返回 true = 否决 */
}
```
> ★★ 只有**明确 override 过**的 Item 子类才被允许出现在生成列/函数索引表达式里。这是"白名单"，不是"黑名单"。

**★ 两条具体限制及根因**（`sql/item.cc:1110`，`Item_field` 的 override）：

```cpp
  /*
    Don't allow the GC (or default expression) to refer itself or another GC
    (or default expressions) that is defined after it.
  */
  if ((func_args->source != VGS_CHECK_CONSTRAINT) &&
      (field->is_gcol() || field->has_insert_default_general_value_expression()) &&
      field->field_index() >= fld_idx) {
    func_args->err_code = (func_args->source == VGS_GENERATED_COLUMN)
                              ? ER_GENERATED_COLUMN_NON_PRIOR : ...;
    return true;
  }
  /*
    If a generated column, default expression or check constraint depends
    on an auto_increment column:
    - calculation of the generated value is done before write_row(),
    - but the auto_increment value is determined in write_row() by the
    engine.
    So this case is forbidden.
  */
  if (field->is_flag_set(AUTO_INCREMENT_FLAG)) {
    func_args->err_code = (func_args->source == VGS_GENERATED_COLUMN)
                              ? ER_GENERATED_COLUMN_REF_AUTO_INC : ...;
    return true;
  }
```
> ★★ **"不能引用 AUTO_INCREMENT 列"的根因**：生成值在 `write_row()` **之前**算，而 auto_inc 值由引擎在 `write_row()` **里**定 —— 时序上无解。

**★ 开表时的二次校验**（`sql/table.cc:2337`）：

```cpp
  // System variables or parameters are not allowed
  else if (expr->type() == Item::PARAM_ITEM ||
           (func_item != nullptr &&
            (func_item->functype() == Item_func::GSYSVAR_FUNC ||
             func_item->functype() == Item_func::GUSERVAR_FUNC))) {
    my_error(error_code_map[source][ER_VARIABLES], MYF(0), source_name);
    return true;
  }
  assert(expr->cols() == 1);
  // Sub-queries are not allowed (already checked by parser, hence the assert)
  assert(!expr->has_subquery());
  ...
  // Stored programs are not allowed. This case is already covered, but still
  // keeping it here as a safetynet.
  if (expr->has_stored_program()) { assert(false); ... }
```
> ★ 子查询是**解析器阶段**就挡掉的（`assert` 说明走到这儿必然已无）。

**★ 错误码清单**（`share/messages_to_clients.txt`，★ 逐个真名核查）：

| 错误码 | 行号 | 文案 |
|---|---|---|
| `ER_FUNCTIONAL_INDEX_PRIMARY_KEY` | 8714 | "The primary key cannot be a functional index" |
| `ER_FUNCTIONAL_INDEX_ON_FIELD` | 8732 | "Functional index on a column is not supported. Consider using a regular index instead." |
| `ER_FUNCTIONAL_INDEX_ON_LOB` | 8717 | "Cannot create a functional index on an expression that returns a BLOB or TEXT. Please consider using CAST." |
| `ER_FUNCTIONAL_INDEX_FUNCTION_IS_NOT_ALLOWED` | 8720 | "Expression of functional index '%s' contains a disallowed function." |
| `ER_FUNCTIONAL_INDEX_REF_AUTO_INCREMENT` | 8708 | "Functional index '%.64s' cannot refer to an auto-increment column." |
| `ER_FUNCTIONAL_INDEX_ON_JSON_OR_GEOMETRY_FUNCTION` | 8705 | "Cannot create a functional index on a function that returns a JSON or GEOMETRY value." |
| `ER_FULLTEXT_FUNCTIONAL_INDEX` | 8723 | "Fulltext functional index is not supported." |
| `ER_SPATIAL_FUNCTIONAL_INDEX` | 8726 | "Spatial functional index is not supported." |
| `ER_WRONG_KEY_COLUMN_FUNCTIONAL_INDEX` | 8729 | "The used storage engine cannot index the expression '%s'." |
| `ER_CANNOT_DROP_COLUMN_FUNCTIONAL_INDEX` | 8711 | "Cannot drop column '%-.64s' because it is used by a functional index..." |
| `ER_WARN_DATA_TRUNCATED_FUNCTIONAL_INDEX` | ~8699 | "Data truncated for functional index '%s' at row %ld" |
| `ER_FUNCTIONAL_INDEX_DATA_IS_TOO_LONG` | 9179 | "Data too long for functional index '%-.64s'." |
| `ER_FUNCTIONAL_INDEX_ROW_VALUE_IS_NOT_ALLOWED` | 8849 | ⚠️ **定义存在但该函数不抛它** |

> ⚠️ **`ER_FUNCTIONAL_INDEX_ROW_VALUE_IS_NOT_ALLOWED` 虽然定义了，但 `add_functional_index_to_create_list()` 走的是生成列错误码路径**，命中的是 `ER_GENERATED_COLUMN_ROW_VALUE`。"定义存在 ≠ 是该路径抛出的错误。"

**★★ 错误码翻译器**（`sql/error_handler.cc:330`，★ "函数索引 = 隐藏生成列"在错误信息层面的代价）：

```cpp
bool Functional_index_error_handler::handle_condition(
    THD *, uint sql_errno, const char *,
    Sql_condition::enum_severity_level *level, const char *) {
  switch (sql_errno) {
    case ER_JSON_USED_AS_KEY:
      res_errno = ER_FUNCTIONAL_INDEX_ON_JSON_OR_GEOMETRY_FUNCTION; break;
    case ER_TRUNCATED_WRONG_VALUE:
      res_errno = ER_WARN_DATA_TRUNCATED_FUNCTIONAL_INDEX; print_row = true; break;
    case ER_JT_VALUE_OUT_OF_RANGE:
    case ER_WARN_DATA_OUT_OF_RANGE:
      res_errno = ER_WARN_DATA_OUT_OF_RANGE_FUNCTIONAL_INDEX; print_row = true; break;
    case ER_GENERATED_COLUMN_REF_AUTO_INC:
      res_errno = ER_FUNCTIONAL_INDEX_REF_AUTO_INCREMENT; break;
    case ER_GENERATED_COLUMN_NAMED_FUNCTION_IS_NOT_ALLOWED:
    case ER_GENERATED_COLUMN_FUNCTION_IS_NOT_ALLOWED:
      res_errno = ER_FUNCTIONAL_INDEX_FUNCTION_IS_NOT_ALLOWED; break;
    ...
```
> ★ 底层抛的都是 `ER_GENERATED_COLUMN_*`，必须靠这个 handler **换皮**成 `ER_FUNCTIONAL_INDEX_*`。

**★ 主键 / FULLTEXT / SPATIAL 三连拒**（`sql/sql_table.cc:4890`）：

```cpp
    if (key->type == KEYTYPE_FULLTEXT) {
      errmsg = "Fulltext index on virtual generated column";
      functional_index_error_handler.force_error_code(ER_FULLTEXT_FUNCTIONAL_INDEX);
    } else if (key->type == KEYTYPE_SPATIAL ||
               sql_field->sql_type == MYSQL_TYPE_GEOMETRY) {
      errmsg = "Spatial index on virtual generated column";
      functional_index_error_handler.force_error_code(ER_SPATIAL_FUNCTIONAL_INDEX);
    } else if (key->type == KEYTYPE_PRIMARY) {
      errmsg = "Defining a virtual generated column as primary key";
      functional_index_error_handler.force_error_code(ER_FUNCTIONAL_INDEX_PRIMARY_KEY);
    }
```
> ★ 函数索引不能做主键是**两处独立拦截**：这里（虚拟生成列路径）+ `sql_table.cc:7766`（函数索引路径）。

**★ DROP 的两个方向都堵死**（`sql/sql_table.cc:16159`，`handle_drop_functional_index`）：

```cpp
          for (uint k = 0; k < key_info.user_defined_key_parts; ++k) {
            const KEY_PART_INFO &key_part = key_info.key_part[k];
            if (key_part.field->is_field_for_functional_index()) {
              // Add column to drop list
              Alter_drop *column_drop = new (thd->mem_root)
                  Alter_drop(Alter_drop::COLUMN, key_part.field->field_name);
              alter_info->drop_list.push_back(column_drop);     /* ① DROP INDEX 连带 DROP 隐藏列 */
              alter_info->flags |= Alter_info::ALTER_DROP_COLUMN;
            }
          }
        ...
        if (my_strcasecmp(..., field->field_name, drop->name) == 0 &&
            field->is_field_for_functional_index()) {
          my_error(ER_CANNOT_DROP_COLUMN_FUNCTIONAL_INDEX, MYF(0),
                   field->field_name);                          /* ② 单独 DROP 隐藏列 → 报错 */
          return true;
        }
```

**★ 隐藏列依赖的基础列不能 DROP**（`sql/sql_table.cc:14610`）：

```cpp
        if (vfield->is_gcol()) {
          if (vfield->is_field_for_functional_index())
            my_error(ER_DEPENDENT_BY_FUNCTIONAL_INDEX, MYF(0), field->field_name);
          else
            my_error(ER_DEPENDENT_BY_GENERATED_COLUMN, MYF(0), field->field_name);
        }
```

**★ 8.0.18 起的排序行为变更**（`sql/sql_table.cc:7725`，★ 隐藏列排最后且按名排序）：

```cpp
/**
  Whether or not we have a replication setup, _and_ the master sorts
  functional index columns last in the table. Sorting said columns last was
  introduced in version 8.0.18, ...
*/
static bool is_not_slave_or_master_sorts_functional_index_columns_last(
    uint32_t master_version) {
  return master_version >= 80018 && master_version != UNKNOWN_SERVER_VERSION;
}
```
> ★ OLD→NEW 复制时必须与主库行为一致，否则表结构会分叉。

---

## 三、多值索引

### 3.1 本质：数组 → 多条索引记录

多值索引的表达式是 `CAST(json_expr AS type ARRAY)`，`returns_array()==true`，隐藏列类型是 `Field_typed_array`（继承 `Field_json`）。InnoDB 把数组每个元素展开成一条二级索引记录。

`Field_typed_array`（`field.h:4162`）内部维护 `m_conv_item`（元素类型对应的普通 Field）用于类型转换与 key 比较。

**★★★ 一对多的真正展开器：`Multi_value_entry_builder`**（`storage/innobase/include/row0row.h:318`）：

```cpp
  dtuple_t *next() {
    if (m_mv_data == nullptr || m_pos >= m_mv_data->num_v) return (nullptr);
    dfield_t *field = dtuple_get_nth_field(m_entry, m_mv_field_no);
    ut_ad(dfield_is_multi_value(field));
    ...
    const auto len = m_mv_data->data_len[m_pos];
    dfield_set_data(field, m_mv_data->datap[m_pos], len);
                                    /* ★★ 同一个 entry 模板，把多值字段的数据指针依次
                                       指向数组第 m_pos 个值 → 一条索引记录 */
    ++m_pos;
    return (m_entry);
  }
```

```cpp
/* storage/innobase/row/row0ins.cc:3260 —— ★ "For each value" 的权威表述 */
/** Inserts an entry into a secondary index, which is created for
multi-value field. For each value to be inserted, it tries first optimistic,
then pessimistic descent down the tree. ...
@param[in]      index           secondary index which is for multi-value field
@param[in,out]  multi_val_pos   the start position to insert next multi-value
                                data, ... */
static dberr_t row_ins_sec_index_multi_value_entry(dict_index_t *index,
                                                   dtuple_t *entry,
                                                   uint32_t &multi_val_pos,
                                                   que_thr_t *thr) {
  ut_ad(index->is_multi_value());
  dberr_t err = DB_SUCCESS;
  Multi_value_entry_builder_insert mv_entry_builder(index, entry);

  for (dtuple_t *mv_entry = mv_entry_builder.begin(multi_val_pos);
       mv_entry != nullptr; mv_entry = mv_entry_builder.next()) {
                                    /* ★★★ 一个数组 → N 次循环 → N 条二级索引记录，
                                       共享同一个 rowid(PK) */
    err = row_ins_sec_index_entry(index, mv_entry, thr, false);
    if (err != DB_SUCCESS) {
      multi_val_pos = mv_entry_builder.last_multi_value_position();
      return (err);
    }
  }
  multi_val_pos = 0;
  return (err);
}
```
```cpp
/* row0ins.cc:3313 —— ★ 多值索引必定是二级索引 */
  if (index->is_clustered()) {
    return (row_ins_clust_index_entry(index, entry, thr, false));
  } else if (index->is_multi_value()) {
    return (row_ins_sec_index_multi_value_entry(index, entry, multi_val_pos, thr));
  } else {
    return (row_ins_sec_index_entry(index, entry, thr, false));
  }
```
> ⚠️ **`row0sel.cc` 里几乎没有多值索引专用扫描代码**（全文件 `multi_value` 只有 2 处：`:324` 一致性校验、`:2518` 的 fatal 保护）。真正的"一对多展开"在 **`row0ins.cc:3260`**。

**★ InnoDB 侧的标志**（★ 索引级 + 列级两套）：

```cpp
/** Multi-value index */
constexpr uint32_t DICT_MULTI_VALUE = 512;              /* dict0mem.h:112 */
  bool is_multi_value() const { return (type & DICT_MULTI_VALUE); }   /* :1344 */
```
```cpp
/** Virtual column */
constexpr uint32_t DATA_VIRTUAL = 8192;
/** Multi-value Virtual column */
constexpr uint32_t DATA_MULTI_VALUE = 16384;            /* data0type.h:225 */
  bool is_multi_value() const { return ((prtype & DATA_MULTI_VALUE) != 0); }  /* dict0mem.h:674 */
```

**★ server 层 JSON → InnoDB 多值 dfield**（`storage/innobase/handler/ha_innodb.cc:23778`）：

```cpp
  if (col->m_col.is_multi_value()) {
    Field_typed_array *fld;
    fld = down_cast<Field_typed_array *>(mysql_table->field[col->m_col.ind]);
                                    /* ★ 多值索引的隐藏列在 server 层就是 Field_typed_array */
    json_binary::Value v(json_binary::parse_binary(mv_data_ptr, mv_length));
    multi_value_data *value = nullptr;
    bool succ = innobase_store_multi_value(
        v, value, fld, field, dict_table_is_comp(index->table), heap);
```

**★★ `Field_typed_array` 的设计意图**（`sql/field.h:4133`，★ Doxygen 原文）：

```cpp
/**
  Field that stores array of values of the same type.                      

  This Field class is used together with Item_func_array_cast class
  (CAST( .. AS .. ARRAY) function) in implementation of multi-valued index.
  Effectively it's a JSON field that contains a single JSON array. When a
  JSON value is stored, it's checked to be either a scalar, or an array.
  All source values are converted using the internal conversion field and
  stored as an array. Field_typed_array ensures that all values stored
  in the array have the same type and precision - the one specified by user.
  This way InnoDB doesn't have to do the conversion on its own and can easily
  index them.                                                              

  The Field_typed_array always reports type of its element and from this
  point of view it's undistinguishable from regular field having the same
  type. Due to that, fields are differentiated by is_array() property.
  Field_typed_array returns true, all other fields - false.
  ...
  @see Item_func_array_cast
*/
```
> ★★ 两个关键点：① **类型统一在 server 层收敛**，SE 拿到的是可直接索引的同类型原子值；② 对外 `type()` 返回**元素类型**，普通代码看不出它是数组 —— 靠 `is_array()` 区分。

```cpp
class Field_typed_array final : public Field_json {    /* ★★ 派生 Field_json，不是 Field_blob */
  /// Conversion item_field
  Item_field *m_conv_item{nullptr};
  enum_field_types m_elt_type;
  uint m_elt_decimals;
  const CHARSET_INFO *m_elt_charset;
  const bool unsigned_flag;
 public:
  enum_field_types type() const override { return real_type_to_type(m_elt_type); }
                                    /* ★ 伪装成元素类型 */
  enum_field_types binlog_type() const override { return MYSQL_TYPE_TYPED_ARRAY; }
                                    /* ★ 只有 binlog 元数据才暴露"这是 TYPED_ARRAY" */
  bool is_array() const override { return true; }
  ...
  /**
    Get name of the index defined over this field.

    Since typed array fields can be created only as an underlying GC field of
    a multi-valued functional index, there's always only one index defined
    over the field.
  */
  const char *get_index_name() const;
```
> ⚠️ **不存在** `m_array` 成员；不存在 `pack()`/`unpack()` 覆写（存储完全复用 `Field_json` 的 JSON binary）。
> ⚠️ `Field_typed_array` **不在** `sql/json_dom.h`，只在 `sql/field.h:4162`。

**★ 一个索引只允许一个多值列**（`storage/innobase/include/dict0mem.h:1635`，★ InnoDB 侧铁证）：

```cpp
      /* Only one multi-value field, if not match then no match. */
      if (col->is_multi_value()) {          /* ★ 遇到第一个多值列就停 */
        break;
      }
```

**★ 相关常量（真名核查）**：

```cpp
/* include/my_base.h:554 —— ⚠️ 不在 sql/handler.h */
/** Multi-valued key */
#define HA_MULTI_VALUED_KEY (1 << 19)
```
```cpp
/* sql/handler.h:515 —— 这是引擎"能力位"，不是索引 flag */
/**
  Supports multi-valued index
*/
#define HA_MULTI_VALUED_KEY_SUPPORT (1LL << 55)
```
> ★ `HA_MULTI_VALUED_KEY_SUPPORT` 全仓命中仅 3 处：定义、校验（`sql_table.cc:4842`）、声明（`ha_innodb.cc:2981`）。**本树中只有 InnoDB 支持多值索引**。

### 3.2 DDL 校验

```cpp
if (kp->has_expression() && kp->get_expression()->returns_array()) {
  if (mv_key_parts++) {                              // 每索引最多 1 个多值 key part
    my_error(ER_NOT_SUPPORTED_YET, "more than one multi-valued key part");
  }
  if (!(se_index_flags & HA_MULTI_VALUED_KEY_SUPPORT)) {  // 引擎能力位
    my_error(ER_CHECK_NOT_IMPLEMENTED, "multi-valued indexes");
  }
  if (kp->is_explicit()) {                            // 不允许 ASC/DESC
    my_error(ER_WRONG_USAGE, "explicit index order");
  }
}
```

引擎能力位 `HA_MULTI_VALUED_KEY_SUPPORT (1LL << 55)`（`handler.h:515`），InnoDB 声明支持（`ha_innodb.cc:2981`）。运行时 `KEY::flags` 的 `HA_MULTI_VALUED_KEY` 位在打开表时按 `field->is_array()` 设置（`dd_table_share.cc:1263`）。

### 3.3 MEMBER OF 语义

`Item_func_member_of`（`item_json_func.h:1059`）：`args[0]` = 标量，`args[1]` = 数组；`key_item()` 返回 `args[1]`（索引候选是右操作数）；`select_optimize()==OPTIMIZE_KEY` 表示优先走索引。

`val_int()`（`item_json_func.cc:3944`，不走索引的全表路径）：左右任一 NULL → 结果 NULL；右侧非数组退化为标量相等；数组被 `Item_cache_json` 缓存则 `sort()` 后 `binary_search()`，否则线性遍历。

**★ 类定义**（`sql/item_json_func.h:1059`）：

```cpp
class Item_func_member_of : public Item_bool_func {
 public:
  Item_func_member_of(const POS &pos, Item *a, Item *b)
      : Item_bool_func(pos, a, b) {}
  const char *func_name() const override { return "member of"; }
  enum Functype functype() const override { return MEMBER_OF_FUNC; }
  bool resolve_type(THD *thd) override {
    if (param_type_is_default(thd, 0, 2, MYSQL_TYPE_JSON)) return true;
    args[0]->mark_json_as_scalar();    /* ★★ 左值被强制标记为"原子标量" */
    return false;
  }
  bool gc_subst_analyzer(uchar **) override { return true; }
  optimize_type select_optimize(const THD *) override { return OPTIMIZE_KEY; }
  longlong val_int() override;
  Item *key_item() const override { return args[1]; }
                                      /* ★★ key_item 是第 2 个参数（右边那个数组/字段）
                                         这是多值索引匹配的关键 */
  enum_const_item_cache can_cache_json_arg(Item *arg) override {
    return (arg == args[1]) ? CACHE_JSON_VALUE
                            : ((arg == args[0]) ? CACHE_JSON_ATOM : CACHE_NONE);
  }
};
```
> ★ `mark_json_as_scalar()` —— 左操作数**不能是数组/对象**，必须是原子标量。

**★ `val_int()` 完整实现**（`sql/item_json_func.cc:3944`）：

```cpp
longlong Item_func_member_of::val_int() {
  null_value = false;
  try {
    // arg 0 is the value to lookup
    if (get_json_atom_wrapper(args, 0, func_name(), &m_doc_value, &conv_buf,
                              &doc_a, nullptr, true) ||    /* ★ 取"原子" */
        args[0]->null_value) {
      null_value = true;
      return 0;
    }
    // arg 1 is the array to look up value in
    if (get_json_wrapper(args, 1, &m_doc_value, func_name(), &doc_b) ||
        args[1]->null_value) {
      null_value = true;
      return 0;
    }
    // If it's cached as JSON, pre-sort array (only) for faster lookups
    if (args[1]->type() == Item::CACHE_ITEM &&
        args[1]->data_type() == MYSQL_TYPE_JSON) {
      Item_cache_json *cache = down_cast<Item_cache_json *>(args[1]);
      if (!(is_doc_b_sorted = cache->is_sorted())) {
        cache->sort();                                     /* ★ 常量数组先排序 */
        if (cache->val_json(&doc_b)) return error_int();
        is_doc_b_sorted = true;
      }
    }
    null_value = false;
    if (doc_b.type() != enum_json_type::J_ARRAY)
      return (!doc_a.compare(doc_b));    /* ★★ 右值非数组 → 退化为"等值比较" */
    else if (is_doc_b_sorted) {
      Json_array *arr = down_cast<Json_array *>(doc_b.to_dom());
      return arr->binary_search(doc_a.to_dom());           /* ★ 二分查找 */
    } else {
      for (uint i = 0; i < doc_b.length(); i++) {          /* ★ 线性逐个 compare */
        Json_wrapper elt = doc_b[i];
        if (!doc_a.compare(elt)) return true;
      }
    }
```
> ★★ `return (!doc_a.compare(doc_b))` —— **`MEMBER OF` 在右值非数组时就是等值比较**，这是它与 `JSON_CONTAINS` 的关键语义差异（后者要求右值必须是 JSON）。

**★ 语法：`OF` 关键字可选**（`sql/sql_yacc.yy:10323`）：

```cpp
        | bit_expr MEMBER_SYM opt_of '(' simple_expr ')'
          {
            $$= NEW_PTN Item_func_member_of(@$, $1, $5);
          }
```
> ★ 所以 `x MEMBER OF (a)` 与 `x MEMBER(a)` **都合法**。左 = `bit_expr`，右 = `simple_expr`（括号内）。

**★ 与 `JSON_CONTAINS` 的对照**（`sql/item_json_func.h:391`）：

```cpp
  bool resolve_type(THD *thd) override {
    if (param_type_is_default(thd, 0, 1, MYSQL_TYPE_JSON)) return true;
    if (param_type_is_default(thd, 1, 3)) return true;
                    /* ★ 第 2 个参数可以是标量(1)或数组(3)，不像 MEMBER OF 那样
                       把参数 0 强制标量化 */
```

**★★ 可替换参数位置的权威说明**（`sql/item_func.cc:1273`，★ 源码注释直接点明差异）：

```cpp
  The following predicates' arguments could be transformed:
  - EQ_FUNC, LT_FUNC, LE_FUNC, GE_FUNC, GT_FUNC, JSON_OVERLAPS
    - Left _or_ right argument if the opposite argument is a constant.
  - IN_FUNC, BETWEEN
    - Left argument if all other arguments are constant and of the same type.
  - MEMBER OF
    - Right argument if left argument is constant.                 /* ★ 只替换右边（数组侧） */
  - JSON_CONTAINS
    - First argument if the second argument is constant.           /* ★ 只替换第 1 个（文档侧） */
  ...
  Note: Range optimizer is used with multi-value indexes and it prefers
  constants. ... the range optimizer is currently not able to
  handle multi-valued indexes with dynamic ranges, hence we use only constants
  in these cases.                                                  /* ★★ 限制：不支持 dynamic range */
```
> ★★ **`MEMBER OF` 替换右参数，`JSON_CONTAINS` 替换第 1 个参数** —— 方向相反，极易记错。

---

## 四、GC substitution：优化器如何匹配

> 核心机制：优化阶段把 WHERE 里与隐藏生成列表达式**语义等价**的表达式，替换成该隐藏列的 `Item_field`，之后 range/ref 优化器就能像普通列一样看到它。

### 4.1 入口 `substitute_gc`（sql_optimizer.cc:1189）

```cpp
// 收集所有"参与索引 + 表达式是函数"的 GC 字段
for (Table_ref *tl : leaf_tables) {
  for (Field *fld : tl->table->field) {
    if (fld->is_gcol() &&
        !(fld->part_of_key.is_clear_all() && fld->part_of_prefixkey.is_clear_all()) &&
        fld->gcol_info->expr_item->can_be_substituted_for_gc())
      indexed_gc.push_back(fld);
  }
}
// 对 WHERE 树做两遍 walk：analyzer + transformer
where_cond->compile(&Item::gc_subst_analyzer, ..., &Item::gc_subst_transformer, &indexed_gc);
```

### 4.2 谓词变换 `Item_func::gc_subst_transformer`（item_func.cc:1300）

```cpp
switch (functype()) {
  case EQ_FUNC: case LT_FUNC: ... case GT_FUNC: {
    // 形式: <expr> OP <const> 或 <const> OP <expr>
    if (args[0]->can_be_substituted_for_gc() && is_const_or_outer_reference(args[1])) {
      func = args; val = args[1];
    } else if (args[1]->can_be_substituted_for_gc() && is_const_or_outer_reference(args[0])) {
      func = args + 1; val = args[0];
    } else break;
    substitute_gc_expression(func, nullptr, gc_fields, val->result_type(), this);
    break;
  }
  case MEMBER_OF_FUNC: {   // 多值索引：左值常量 + 右值可替换
    if (args[0]->const_for_execution() && !args[0]->is_null() &&
        args[1]->can_be_substituted_for_gc(/*array=*/true))
      substitute_gc_expression(args + 1, args, gc_fields, args[0]->result_type(), this);
    break;
  }
}
```

`LOWER(col)='abc'` 命中路径：`args[0]=LOWER(col)`（可替换）、`args[1]='abc'`（常量）→ 进入 `substitute_gc_expression`。

### 4.3 找到等价字段并替换 `get_gc_for_expr`（item_func.cc:1042）

```cpp
Item_field *get_gc_for_expr(const Item *func, Field *fld, Item_result type, ...) {
  func = func->real_item();
  Item *expr = fld->gcol_info->expr_item;   // 生成列的原始表达式

  // 剥离 CAST / COLLATE / JSON_UNQUOTE 外壳（用户 WHERE 里没写这层壳）
  for (Functype ft : {COLLATE_FUNC, TYPECAST_FUNC, JSON_UNQUOTE_FUNC}) {
    if (is_function_of_type(expr, ft) && !is_function_of_type(func, ft))
      expr = down_cast<Item_func *>(expr)->get_arg(0);
  }
  if (!expr->can_be_substituted_for_gc(fld->is_array())) return nullptr;
  if (type == fld->result_type() && func->eq(expr, bin_cmp)) {   // ★ 表达式等价
    fld->table->mark_column_used(fld, MARK_COLUMNS_READ);         // 标记读隐藏列
    return new Item_field(fld);
  }
  return nullptr;
}
```

**匹配算法**：`func->eq(expr, bin_cmp)` 是 Item 树的**结构化等价比较**（递归比较函数类型、参数、常量值），不是字符串比较。匹配成功后 `thd->change_item_tree(expr, item_field)` 把 WHERE 里的 `LOWER(col)` 就地换成 `Item_field(隐藏列)`，后续 range 优化器当普通索引列处理。

### 4.4 哪些表达式可替换 `can_be_substituted_for_gc`（item.cc:7590）

```cpp
bool Item::can_be_substituted_for_gc(bool array) const {
  switch (real_item()->type()) {
    case FUNC_ITEM: case COND_ITEM: return true;   // 函数/条件可替换
    case FIELD_ITEM: return array;                  // 只有多值索引（字段上定义）才允许
    default: return false;                          // 常量不可替换
  }
}
```

### 4.5 多值索引的 range 转换（range_analysis.cc:621）

GC 替换后，`MEMBER OF` 变成 `field = coerced_const`，range 优化器的 `get_func_mm_tree` 识别 `MEMBER_OF_FUNC`：

```cpp
case Item_func::MEMBER_OF_FUNC:
  if (predicand->type() == Item::FIELD_ITEM && predicand->returns_array()) {
    Field_typed_array *field = down_cast<Field_typed_array *>(...);
    Item *arg = cond_func->arguments()[0];   // 已 coerce 的左常量
    Json_wrapper wr;
    if (arg->val_json(&wr)) break;
    if (wr.type() == enum_json_type::J_NULL) break;   // NULL 不能走 range

    // 关键技巧：临时设 const_table，让 get_mm_parts 用常量构造等值 range
    const bool save_const = field->table->const_table;
    field->table->const_table = true;
    tree = get_mm_parts(thd, param, ..., field, Item_func::EQ_FUNC, predicand);
    field->table->const_table = save_const;
  }
  break;
```

**核心**：把 `MEMBER OF` 转成一个**等值 range**（`EQ_FUNC`）——因为多值索引把数组展开成多行，查某个元素等价于查等值键。

---

## 五、执行期去重（多值索引）

**问题**：一个 JSON 数组 `[1,1,2]` 会为元素 `1,1,2` 各产生一条二级索引记录，全部指向同一个主键 row。`1 MEMBER OF(arr)` 会命中两条 `1` 记录，同一行会被返回两次。

**解法**：`HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER` 唯一记录过滤器。

### 5.1 启用（handler.cc:8489）

```cpp
if (operation == HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER) {
  assert(inited == INDEX && (key_info[active_index].flags & HA_MULTI_VALUED_KEY));
  if (!m_unique &&
      (!(m_unique = new Unique_on_insert(ref_length)) || m_unique->init()))
    return HA_ERR_OUT_OF_MEM;
  m_unique->reset(true);   // 清空 rowid 集合
}
```

`m_unique` 是 `Unique_on_insert`（B-tree/哈希集合，key 是 `ref_length` 字节的 rowid）。

### 5.2 去重判定 `filter_dup_records`（handler.cc:8483）

```cpp
bool handler::filter_dup_records() {
  position(table->record[0]);        // 取当前行 rowid 到 ref 缓冲区
  return m_unique->unique_add(ref);  // 已存在则返回 true（重复）
}
```

### 5.3 扫描路径调用（do-while 过滤）

`ha_index_next`（handler.cc:3262）：

```cpp
if (!result && !mrr_have_range && m_unique != nullptr && filter_dup_records())
  result = HA_ERR_KEY_NOT_FOUND;   // 重复 → 继续拉下一条
```

MRR 扫描 `multi_range_read_next`（handler.cc:6494）用 `do-while` 循环：

```cpp
} while ((result == HA_ERR_END_OF_FILE) ||
         (m_unique && (dup_found = filter_dup_records())) && !range_res);
```

**机制**：每读到一条索引记录，`filter_dup_records()` 用 rowid 判重；重复则继续拉下一条，直到拉到**未见过的新 rowid**——这就是 07 篇提到的 do-while 过滤循环。

**★★★ 完整的 do-while 循环**（`sql/handler.cc:6449`，★ 5.3 的主体）：

```cpp
int handler::multi_range_read_next(char **range_info) {
  int result = HA_ERR_END_OF_FILE;
  int range_res = 0;
  bool dup_found = false;
  // For a multi-valued index the unique filter have to be used for correct
  // result
  assert(!(table->key_info[active_index].flags & HA_MULTI_VALUED_KEY) ||
         m_unique);            /* ★★ 多值索引 MRR 必须有 m_unique，否则结果就是错的 */

  if (!mrr_have_range) { mrr_have_range = true; goto start; }

  do {                          /* ★★ 外层 do-while：命中重复就继续取下一条 */
    if (!((mrr_cur_range.range_flag & UNIQUE_RANGE) &&
          (mrr_cur_range.range_flag & EQ_RANGE))) {
      result = read_range_next();
      if (!result) continue;
      if (result != HA_ERR_END_OF_FILE) break;
    } else {
      if (was_semi_consistent_read()) goto scan_it_again;
    }
  start:
    while (!(range_res = mrr_funcs.next(mrr_iter, &mrr_cur_range))) {
    scan_it_again:
      result = read_range_first(...);
      if (result != HA_ERR_END_OF_FILE) break;
    }
  } while (((result == HA_ERR_END_OF_FILE) ||
            (m_unique && (dup_found = filter_dup_records()))) &&
                                    /* ★ filter_dup_records() 返回 true = rowid 已见过 → 丢弃继续 */
           !range_res);

  *range_info = mrr_cur_range.ptr;
  if (dup_found && range_res) result = HA_ERR_END_OF_FILE;
                                    /* ★ 最后一条是重复且所有 range 已耗尽 → 正常收尾成 EOF */
  return result;
}
```

**★ 去重开关的真名是 `m_unique`**（`sql/handler.h:4649`，⚠️ **不存在** `in_use_dedup` / `m_is_dedup`）：

```cpp
  /* Filter row ids to weed out duplicates when multi-valued index is used */
  Unique_on_insert *m_unique;
```

**★ `HA_ERR_KEY_NOT_FOUND` 的语义被复用**（`sql/handler.cc:3329`，★ Doxygen 明写）：

```cpp
/**
  Reads the next row via index.
  ...
  @return Operation status.
    @retval  0                    Success
    @retval  HA_ERR_END_OF_FILE   Row not found
    @retval  HA_ERR_KEY_NOT_FOUND This return value indicates duplicate row
                                  returned from storage engine during
                                  multi-value index read.
    @retval  != 0                 Error
*/
int handler::ha_index_next(uchar *buf) {
  ...
  // Filter duplicate records from multi-value index read.
  // (m_unique != nullptr in case of multi-value index read)
  // In case of range scan, duplicate records are filtered in
  // multi_range_read_next()
  if (!result && !mrr_have_range && m_unique != nullptr && filter_dup_records())
    result = HA_ERR_KEY_NOT_FOUND;
```
> ★★ **重复行被翻译成 `HA_ERR_KEY_NOT_FOUND` 而不是 EOF** —— 上层据此"继续读"而不是"结束"。这是一个复用现有错误码表达新语义的技巧。
> ★ 同样的分支出现在 6 处：`ha_index_read_map`(3267) / `ha_index_read_last_map`(3293) / `ha_index_next`(3363) / `ha_index_prev`(3405) / `ha_index_first`(3443) / `ha_index_next_same`(3518)。

**★ 过滤器只在"非 MRR"路径生效**（★ 注意 `!mrr_have_range` 这个条件）：

```cpp
  if (!result && !mrr_have_range && m_unique != nullptr && filter_dup_records())
```
> ★ range scan 走 MRR，去重交给 `multi_range_read_next()` 的 do-while；非 MRR 路径（ref/index read）才在这里直接过滤。

**★★ ref 访问的第二个 do-while**（`sql/iterators/ref_row_iterators.cc:712`）：

```cpp
  } else {
    // Fetch unique rows matching the Ref Key in case of multi-value index
    do {
      error = table()->file->ha_index_next_same(
          table()->record[0], key_buff_and_map.first, m_ref->key_length);
    } while (error == HA_ERR_KEY_NOT_FOUND && m_is_mvi_unique_filter_enabled);
                                    /* ★★ 靠 HA_ERR_KEY_NOT_FOUND 反复跳到下一行 */
  }
```

**★ 迭代器侧的去重开关**（`sql/iterators/ref_row_iterators.cc:337`，真名 `m_is_mvi_unique_filter_enabled`）：

```cpp
  // Enable & reset unique record filter for multi-valued index
  if (table()->key_info[m_ref->key].flags & HA_MULTI_VALUED_KEY) {
    table()->file->ha_extra(HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER);
    table()->prepare_for_position();      /* ★ 要 position()/rowid，必须先 prepare_for_position */
    m_is_mvi_unique_filter_enabled = true;
  }
```
```cpp
RefOrNullIterator::~RefOrNullIterator() {
  if (table()->key_info[m_ref->key].flags & HA_MULTI_VALUED_KEY && table()->file)
    table()->file->ha_extra(HA_EXTRA_DISABLE_UNIQUE_RECORD_FILTER);
}
```

**★★ 去重的隐藏代价：必须额外读 PK**（`sql/range_optimizer/index_range_scan.cc:291`）：

```cpp
  // Enable & reset unique record filter for multi-valued index
  if (table()->key_info[index].flags & HA_MULTI_VALUED_KEY) {
    file->ha_extra(HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER);
    // Add PK's fields to read_set as unique filter uses rowid to skip dups
    if (table->s->primary_key != MAX_KEY)
      table->mark_columns_used_by_index_no_reset(table->s->primary_key,
                                                 table->read_set);
                                    /* ★★ 去重要 rowid，所以必须额外读 PK 列 */
  }
```
> ★★ 这解释了为什么**多值索引天然难做覆盖索引**——即便"逻辑上"覆盖了，为了去重也得回表读 PK。

**★ 一个已知缺陷：MRR 下多值索引的端点比较被跳过**（`sql/handler.cc:7479`）：

```cpp
  /*
    Virtual fields are not updated during multi-valued index read in MRR.
    Hence key comparison is skipped for MV index.
    TODO: Disable MRR on MV index or implement a comparison logic.
  */
  if (!(table->key_info[active_index].flags & HA_MULTI_VALUED_KEY)) {
    cmp = key_cmp(range_key_part, range->key, range->length);
  }
```
> ★ 对应 `Field_typed_array` 里那句注释：*"Since there's no order defined for sorting set of arrays, always return -1 here, allowing all records fetched from SE to be returned to server. They will be filtered by WHERE condition later."* —— **多值索引没有"数组间的顺序"**，所以 end_range 比较恒返回 -1。

**★ `key_cmp_if_same` 强制走 MEMBER OF 而非 memcmp**（`sql/key.cc:271`）：

```cpp
    /*
      Multi-valued column's are compared using MEMBER OF operation implemented
      in Field_typed_array::key_cmp() instead of mem compare.
    */
    if (key_part->bin_cmp && !is_multi_valued_index && ...) {
      if (memcmp(key, table->record[0] + key_part->offset, length)) return true;
    } else {
      if (key_part->field->key_cmp(key, key_part->length)) return true;
                                    /* ★ → Field_typed_array::key_cmp，语义 = MEMBER OF */
    }
```

**★ DS-MRR 与 index merge 的清理**（★ 容易遗漏的两处）：

```cpp
/* sql/handler.cc:6636 —— DS-MRR 的第二个 handler(h2) 也要装去重器 */
    if ((table->key_info[mrr_keyno].flags & HA_MULTI_VALUED_KEY) &&
        (retval = h2->ha_extra(HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER)))
      goto error;
```
```cpp
/* sql/range_optimizer/index_merge.cc:65 —— 子扫描用了多值索引就要手工关 */
    disable_unique_filter |=
        Overlaps(table()->key_info[range->index].flags, HA_MULTI_VALUED_KEY);
    range->file = nullptr;
  }
  if (disable_unique_filter)
    table()->file->ha_extra(HA_EXTRA_DISABLE_UNIQUE_RECORD_FILTER);
```

### 5.4 多值键比较 `Field_typed_array::key_cmp`（field.cc:9978）

多值列不能直接 `memcmp`，因为"键 = 某元素"才是相等：

```cpp
int Field_typed_array::key_cmp(const uchar *key_ptr, uint key_length) const {
  // 把 key 还原成 JSON 标量，再对数组逐元素比较，任一元素等于 key 即命中
  ...
  for (uint i = 0; i < col_val.length(); i++) {
    Json_wrapper elt = col_val[i];
    if (key.compare(elt) == 0) return 0;
  }
  return -1;
}
```

### 5.5 多值索引的限制

`ha_innodb.cc:6391`：多值索引**不支持** `HA_READ_ORDER`（有序扫描）和 `HA_KEYREAD_ONLY`（覆盖索引），因为一个文档的多个元素插入顺序无关、同一行对应多条索引记录。

```cpp
  /* Multi-valued keys don't support ordered retrieval, neither they're
  suitable for keyread only retrieval. */
  if (table_share->key_info[key].flags & HA_MULTI_VALUED_KEY) {
    flags &= ~(HA_READ_ORDER | HA_KEYREAD_ONLY);   /* ★★ 两条硬限制的物理来源 */
  }
```

**★ DDL 校验原文**（`sql/sql_table.cc:4836`）：

```cpp
          if (kp->has_expression() && kp->get_expression()->returns_array()) {
            if (mv_key_parts++) {                 /* ★ 第二个多值键部分 → 报错 */
              my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                       "more than one multi-valued key part per index");
              return true;
            }
            if (!(se_index_flags & HA_MULTI_VALUED_KEY_SUPPORT)) {
              my_error(ER_CHECK_NOT_IMPLEMENTED, MYF(0),
                       "multi-valued indexes");   /* ★ 引擎能力位检查 */
              return true;
            }
            if (kp->is_explicit()) {              /* ★ 不允许 ASC/DESC */
              my_error(ER_WRONG_USAGE, MYF(0), "multi-valued index",
                       "explicit index order");
              return true;
            }
          }
```

**★ 字符集只能用 binary / utf8mb4_0900_bin**（`sql/item_create.cc:1857`，★ 注释解释了原因）：

```cpp
    /*
      Multi-valued index currently only supports two character sets: binary for
      BINARY(x) keys and my_charset_utf8mb4_0900_bin for CHAR(x) keys. The
      latter one is because it's closest to binary in terms of sort order and
      doesn't pad spaces. This is important because JSON treats e.g. "abc" and
      "abc " as different values and a space padding charset will cause
      inconsistent key handling.
    */
    if (cast_type.charset != nullptr && cast_type.charset != &my_charset_bin) {
      my_error(ER_NOT_SUPPORTED_YET, MYF(0),
               "specifying charset for multi-valued index");
      return true;
    }
```
> ★★ **根因**：JSON 把 `"abc"` 与 `"abc "` 当作不同值，而 PAD SPACE 排序规则会让二者键处理不一致。

**★ 容量限制与三个错误码**（`sql/field.cc:9868`，`store_array`）：

```cpp
        table->file->ha_mv_key_capacity(&max_num_keys, &max_keys_length);
                                    /* ★ 向 SE 询问"单条记录最多几个值 / 总共多少字节" */
        ...
          if (elt.type() == enum_json_type::J_NULL) {
            /*
              Unlike SQL NULL, JSON null is a value, but a special one and it
              can't be coerced to any data type. The latter means it can't be
              indexed by relational SE. Due to that an error is thrown.
            */
            my_error(ER_INVALID_JSON_VALUE_FOR_FUNC_INDEX, MYF(0),
                     get_index_name());       /* ★ JSON null 不能进多值索引 */
            return TYPE_ERR_BAD_VALUE;
          }
        ...
        /*
          Non-strict mode issue:
          While consisting of unique values, bad input can cause duplicates
          after coercion. Remove them, as SE doesn't expect dups in the array.
          This is why we need to sort & remove duplicates only after
          processing all keys.
        */
        if (array->size() > 1)
          array->remove_duplicates(type() == MYSQL_TYPE_VARCHAR ? m_elt_charset
                                                                : nullptr);
        if (array->size() > max_num_keys) {
          my_error(ER_EXCEEDED_MV_KEYS_NUM, MYF(0), get_index_name(),
                   array->size() - max_num_keys);      /* ★ 限制 1：值个数 */
          return TYPE_ERR_BAD_VALUE;
        }
        if (keys_length > max_keys_length) {
          my_error(ER_EXCEEDED_MV_KEYS_SPACE, MYF(0), get_index_name(),
                   (keys_length - max_keys_length));   /* ★ 限制 2：总字节 */
          return TYPE_ERR_BAD_VALUE;
        }
```
```cpp
      case enum_json_type::J_BOOLEAN:
        my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                 "CAST-ing JSON BOOLEAN type to array");   /* ★ 限制 3 */
      case enum_json_type::J_OBJECT:
        my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                 "CAST-ing JSON OBJECT type to array");    /* ★ 限制 4 */
```
> ★ 空数组 → 存成非 NULL 的空数组（该行在多值索引里**一条记录都没有**）。标量退化成 1 元素数组时**不做**条数/长度检查。

**★ 容量上限不是固定常量，是按 undo 页贪心估算**（`storage/innobase/handler/ha_innodb.cc:23862`）：

```cpp
/** Return max limits for a single set of multi-valued keys
...
  /* The limit of multi-value should be checked against undo page size,
  because a record length can not be longer than an undo page size. ...
  Therefore, only the best effort can be done in this function. ...
  So it's the greedy estimation which will try to give a most relaxed
  restriction on the table, ... The DB_UNDO_RECORD_TOO_BIG error would be
  raised if the actual user record exceeds the undo page size. */
  size_t free_space = trx_undo_max_free_space();
  /* PK always takes place in log page, take that into account */
  uint pk = table->s->primary_key;
  if (pk != MAX_KEY) {
    free_space -= table->s->key_info[pk].key_length;
  } else {
    free_space -= DATA_ROW_ID_LEN;
  }
```
> ★★ 上限的**物理来源**：一条记录不能长过一个 undo page。

**★ 完整限制清单**（★ 源码证据汇总）：

| 限制 | 证据 | 报错/行为 |
|---|---|---|
| 每索引只能**一个**多值键部分 | `sql_table.cc:4836`；`dict0mem.h:1635` | `ER_NOT_SUPPORTED_YET` |
| 引擎必须支持 | `sql_table.cc:4842` | `ER_CHECK_NOT_IMPLEMENTED` |
| 多值键部分不能显式排序 | `sql_table.cc:4847` | `ER_WRONG_USAGE` |
| 只能是**二级索引** | `row0ins.cc:3313` | 分派保证 |
| 字符集只能 binary / utf8mb4_0900_bin | `item_create.cc:1865` | `ER_NOT_SUPPORTED_YET` |
| 不支持 BOOLEAN/OBJECT ARRAY | `field.cc:9931` | `ER_NOT_SUPPORTED_YET` |
| JSON null 不能入索引 | `field.cc:9832` | `ER_INVALID_JSON_VALUE_FOR_FUNC_INDEX` |
| 值个数 / 总字节上限 | `field.cc:9916/9921` | `ER_EXCEEDED_MV_KEYS_NUM` / `_SPACE` |
| 数组写进标量键部分 | `messages_to_clients.txt:9182` | `ER_WRONG_MVI_VALUE` |
| **不能做覆盖索引 / 有序检索** | `ha_innodb.cc:6391` | 无报错，能力位被清 |
| **不做全索引扫描** | `sql_optimizer.cc:1979` | assert + 不选为最短索引 |
| 不支持 dynamic range | `item_func.cc:1291` | 优化器放弃 |
| 主从复制行定位**跳过**多值索引 | `log_event.cc:8358` | "have only part of value" |
| 不能在存储例程里用 `CAST(... AS ... ARRAY)` | `item_create.cc:1857` | `ER_WRONG_USAGE` |
| MRR 下虚拟列不刷新，端点比较跳过 | `handler.cc:7482` | 注释标注 TODO |

```cpp
/* sql/sql_optimizer.cc:1979 —— ★ 多值索引不参与"最短索引"竞选 */
          A multi-valued key requires unique filter, and won't be the most
          fast option even if it will be the shortest one.
        const KEY &key_ref = table->key_info[nr];
        assert(!(key_ref.flags & HA_MULTI_VALUED_KEY) &&
               !(key_ref.flags & HA_SPATIAL));
```
```cpp
/* sql/log_event.cc:8358 —— ★ 主从复制跳过多值索引 */
        /* - Skip multi-valued keys as they have only part of value and can't
             fully identify a record */
          keyinfo->flags & HA_MULTI_VALUED_KEY || !keyinfo->is_visible) {
        continue;
      }
```
> ⚠️ **`ER_MULTI_VALUED_INDEX_*` 系列错误码不存在**（grep 0 命中）。实际用的是 `ER_EXCEEDED_MV_KEYS_NUM` / `ER_EXCEEDED_MV_KEYS_SPACE` / `ER_WRONG_MVI_VALUE` + 通用码组合。

---

## 六、两者对比

| 维度 | 函数索引 | 多值索引 |
|------|---------|---------|
| 隐藏列类型 | 普通 `Field` | `Field_typed_array`（继承 Field_json） |
| 表达式形式 | 任意确定性标量 `LOWER(col)` | `CAST(... AS type ARRAY)` |
| 一行对应索引记录数 | 1 条 | N 条（N=数组元素数） |
| 匹配谓词 | EQ/LT/GT/IN/BETWEEN | MEMBER OF（及 JSON_CONTAINS/OVERLAPS） |
| 匹配后动作 | `change_item_tree` → 隐藏列 | 额外 coerce 左值 → `get_mm_parts(EQ_FUNC)` |
| 排序/覆盖 | 支持 | 不支持（`HA_READ_ORDER`/`HA_KEYREAD_ONLY`） |
| 去重 | 不需要 | 必须 `UNIQUE_RECORD_FILTER` |
| 每索引多值列数 | — | 最多 1 个 |

**EXPLAIN / SHOW CREATE TABLE**：两者都显示为 `KEY idx ((表达式))`（`sql_show.cc:2255` 打印 `gcol_info->print_expr`），因为隐藏列也满足 `is_field_for_functional_index()`。

**★★ 分叉点 1：数组字段只允许 `MEMBER OF` 走通用替换路径**（`sql/item_func.cc:1123`）：

```cpp
    /*
      Don't substitute if:
      1) Key is disabled
      2) It's a multi-valued index's field and predicate isn't MEMBER OF
    */
    if (tkm.is_clear_all() ||                            // (1)
        (field->is_array() && predicate->functype() !=   // (2)
                                  Item_func::MEMBER_OF_FUNC))
      continue;                                         /* ★★ 数组字段只认 MEMBER OF */
    ...
  if (item_field->returns_array() && value) {            /* ★ 多值索引专属分支 */
    ...
    // Don't substitute if value can't be coerced to field's type
    if (afld->coerce_json_value(&wr, /*no_error=*/true, to_wr.get()))
      return false;                                     /* ★ coerce 失败 → 静默放弃索引 */
    Item_json *jsn = new (thd->mem_root) Item_json(std::move(to_wr), ...);
    thd->change_item_tree(value, jsn);                  /* ★ 常量替换成 coerce 后的 JSON 常量 */
  }
  thd->change_item_tree(expr, item_field);
```

**★★ 分叉点 2：CONTAINS/OVERLAPS 走专属替换器**（`sql/item_func.cc:1190`，★ 注释解释了为什么不能复用）：

```cpp
/**
  ... substitute_gc_expression() can't be used to these functions
  as it's tailored to handle regular single-valued indexes and doesn't ensure
  correct coercion of all values to lookup in multi-valued index.
                                    /* ★★ 必须逐个值 coerce */
*/
static void gc_subst_overlaps_contains(Item **func, Item **vals,
                                       Json_wrapper &vals_wr,
                                       List<Field> *gc_fields) {
  for (Field &fld : *gc_fields) {
    bool can_use_index = true;
    if (tkm.is_clear_all() || !fld.is_array()) continue;   /* ★ 只要数组字段 */
    ...
    for (uint i = 0; i < len; i++) {
      Json_wrapper elt = vals_wr[i];
      Json_wrapper res;
      if (afld->coerce_json_value(&elt, true, &res)) {
        can_use_index = false;       /* ★★ 任一值 coerce 失败 → 整个索引不可用 */
        found = nullptr;
        break;
      }
      coerced_keys->append_clone(res.to_dom());
    }
```

**★★ 分叉点 3：range 优化器按算子分派**（`sql/range_optimizer/range_analysis.cc:614`）：

```cpp
    case Item_func::JSON_CONTAINS:
    case Item_func::JSON_OVERLAPS: {
      tree = get_func_mm_tree_from_json_overlaps_contains(
          thd, param, prev_tables, read_tables, remove_jump_scans, predicand,
          cond_func);
    } break;

    case Item_func::MEMBER_OF_FUNC:
      if (predicand->type() == Item::FIELD_ITEM && predicand->returns_array()) {
        Field_typed_array *field = down_cast<Field_typed_array *>(
            down_cast<Item_field *>(predicand)->field);
        Item *arg = cond_func->arguments()[0];
        Json_wrapper wr;
        if (arg->val_json(&wr)) break;
        if (wr.type() == enum_json_type::J_NULL) break;

        // Fake const table for get_mm_parts(), as we are using constants from
        // JSON array
        const bool save_const = field->table->const_table;
        field->table->const_table = true;   /* ★★ 把表临时伪装成 const table */
        field->set_notnull();
        field->coerce_json_value(&wr, true, nullptr);

        tree = get_mm_parts(thd, param, prev_tables, read_tables, cond_func,
                            field, Item_func::EQ_FUNC, predicand);
                                    /* ★★ MEMBER OF 直接退化成 EQ 的单点 range */
        field->table->const_table = save_const;
      }
      break;
```
> ★★ **`MEMBER OF` 最终被翻译成一个 `EQ_FUNC` 的单点 range** —— 这是多值索引能复用现有 range 框架的关键。

**★ 优化器 sargable 分析：三种算子各走各的分支**（`sql/sql_optimizer.cc:7551`）：

```cpp
      else if (cond_func->functype() == Item_func::MEMBER_OF_FUNC &&
               is_local_field(cond_func->key_item())) {
        // The predicate is <val> IN (<typed array>)      /* ★ 注释把 MEMBER OF 类比成 IN */
        add_key_equal_fields(thd, key_fields, *and_level, cond_func, ...);
      } else if (cond_func->functype() == Item_func::JSON_CONTAINS ||
                 cond_func->functype() == Item_func::JSON_OVERLAPS) {
        /*
          Applicability analysis was done during substitute_gc().
          Check here that a typed array field is used and there's a key over it.
          1) func has a key item
          2) key item is a local field
          3) key item is a typed array field
        */
        if (!cond_func->key_item() ||                  // 1
            !is_local_field(cond_func->key_item()) ||  // 2
            !cond_func->key_item()->returns_array())   // 3
          break;
```
> ★ 三项检查缺一不可。

**★ JSON_CONTAINS 只支持"数组在左"一种形态**（`sql/range_optimizer/range_analysis.cc:439`）：

```cpp
  \verbatim
    This function builds SEL_TREE out of JSON_OEVRLAPS() of form:
      JSON_OVERLAPS(typed_array_field, "[<val>,...,<val>]")
      JSON_OVERLAPS("[<val>,...,<val>]", typed_array_field)
      JSON_CONTAINS(typed_array_field, "[<val>,...,<val>]")
    where
      typed_array_field is a field which has multi-valued index defined on it
      <val>             each value in the array is coercible to the array's type
    These conditions are pre-checked in substitute_gc().
  \endverbatim
```

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| 隐藏生成列用全局 `enum class Column_hidden_type` | 真名 **`dd::Column::enum_hidden_type`**（嵌在 `dd::Column` 里），4 个取值 |
| `TABLE::hidden_generated_column_count` | **不存在**（`TABLE` 里带 hidden 的只有 `hidden_field_count`，且只用于临时表） |
| `Field::is_stored_gcol()` | **不存在** → 只有 `is_gcol()` / `is_virtual_gcol()`；`stored_in_db` 是 **bool 成员不是函数** |
| 隐藏列判定字段叫 `is_hidden` | `Create_field` 里成员就叫 **`hidden`**；`Field` 里是私有 `m_hidden`，经 5 个访问器暴露 |
| 语法产生式叫 `key_part_spec` | 语法层是 **`key_part_with_expression`**（含普通 `key_part` 作为第一候选）；语义层是 `PT_key_part_specification`；DDL 内部结构才是 `Key_part_spec` |
| `SELECT *` 靠 `visible_field_count` 排除隐藏列 | **错** → 靠 `insert_fields()` 里的 `field->is_hidden()` |
| 隐藏列名是 index name 的 hash | **过时说法**（Doxygen 没改）→ 实际是明文 `!hidden!<index_name>!<key_part>!<counter>` |
| `dd::Index` 有 expression 字段 | **没有** → 只有 `is_generated()` / `is_hidden()`；表达式存在隐藏生成列里 |
| 函数索引能作主键 | 不能，**两处**独立拦截（`sql_table.cc:4907` 与 `:7766`） |
| `ER_FUNCTIONAL_INDEX_ROW_VALUE_IS_NOT_ALLOWED` 是主错误码 | 定义存在但**该函数不走它** → 实际命中 `ER_GENERATED_COLUMN_ROW_VALUE`，再被 handler 翻译 |
| `HA_MULTI_VALUED_KEY` 在 `sql/handler.h` | **不在** → `include/my_base.h:554`，值 `1 << 19`；`handler.h:515` 的是引擎能力位 `HA_MULTI_VALUED_KEY_SUPPORT (1LL << 55)` |
| `Field_typed_array` 派生 `Field_blob` / 有 `m_array` | **错** → 派生 **`Field_json`**，无 `m_array`；"多个值"就是一个 JSON 数组二进制 |
| `Field_typed_array` 在 `sql/json_dom.h` | **不存在**（只在 `sql/field.h:4162`） |
| 去重开关叫 `in_use_dedup` / `m_is_dedup` | **不存在** → 真名 `handler::m_unique`（`Unique_on_insert*`）；迭代器侧另有一个 `m_is_mvi_unique_filter_enabled` |
| InnoDB 多值标志是某 `DICT_` flag | 索引级 `DICT_MULTI_VALUE=512`（`dict0mem.h:112`）；列级 `DATA_MULTI_VALUE=16384`（`data0type.h:225`） |
| `row0sel.cc` 有多值索引扫描专用代码 | **基本没有**（仅 2 处）→ 真正的一对多展开在 **`row0ins.cc:3260`** |
| 有 `ER_MULTI_VALUED_INDEX_*` 错误码系列 | **不存在**（grep 0 命中）→ 实际是 `ER_EXCEEDED_MV_KEYS_NUM` / `_SPACE` / `ER_WRONG_MVI_VALUE` |
| `MEMBER OF` 只能用于数组 | 源码允许非数组右值 → **退化为等值比较**（`!doc_a.compare(doc_b)`）；左值必须是原子标量 |
| `MEMBER OF` 与 `JSON_CONTAINS` 替换同侧参数 | **方向相反** → MEMBER OF 替换**右**参数，JSON_CONTAINS 替换**第 1** 个参数 |
| 多值索引能做覆盖索引 | **不能** → 去重要 rowid，必须额外读 PK；且 `HA_KEYREAD_ONLY` 被清 |
| 多值索引容量上限是固定常量 | **不是** → 按 undo 页容量"贪心估算"（`ha_innodb.cc:23866`） |
| 源码里有多值索引的 WL# 编号 / Doxygen 页 | **都不存在** |
| 多值索引隐藏列可以是任意表达式 | **不能** → "typed array fields can be created only as an underlying GC field of a multi-valued functional index" |

---


## 参考

**内核月报**
- **2019/02《8.0 Functional index 的实现过程》** —— 函数索引的隐藏列落地、`add_functional_index_to_create_list` 流程（本文第二、四节的直接来源）
- **2019/09《Multi-Valued Indexes 简述》** —— 多值索引的数组展开、MEMBER OF 匹配（本文第三、五节的背景）

**官方文档**
- *MySQL 8.0 Reference Manual → CREATE INDEX → Functional Key Parts*（8.0.13+）
- *MySQL 8.0 Reference Manual → Multi-Valued Indexes*（8.0.17+）
- *MySQL 8.0 Reference Manual → The MEMBER OF() Operator*

**相关文档**
- range 优化（`get_mm_parts`/`get_mm_leaf`）见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)
- 多值索引执行期过滤见 [`../09_executor_iterator.md`](../09_executor_iterator.md)（MVI 的 do-while）
- JSON 数据类型见 [`../../../server/datatype/json.md`](../../../server/datatype/json.md)

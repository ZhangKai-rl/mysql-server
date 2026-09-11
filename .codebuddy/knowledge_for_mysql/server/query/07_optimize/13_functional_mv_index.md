# 13 函数索引与多值索引

> 8.0 的两个索引新特性：**函数索引**（8.0.13+，`CREATE INDEX idx ON t ((LOWER(col)))`）和**多值索引**（8.0.17+，`WHERE 1 MEMBER OF (arr->'$.x')`）。共同本质：**把表达式/数组变成一个隐藏生成列，索引建在隐藏列上**。本篇讲它们如何创建、优化器如何匹配、执行期如何工作。

## 目录

- [一、共同本质：隐藏生成列](#一共同本质隐藏生成列)
- [二、函数索引](#二函数索引)
- [三、多值索引](#三多值索引)
- [四、GC substitution：优化器如何匹配](#四gc-substitution优化器如何匹配)
- [五、执行期去重（多值索引）](#五执行期去重多值索引)
- [六、两者对比](#六两者对比)

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

---

## 三、多值索引

### 3.1 本质：数组 → 多条索引记录

多值索引的表达式是 `CAST(json_expr AS type ARRAY)`，`returns_array()==true`，隐藏列类型是 `Field_typed_array`（继承 `Field_json`）。InnoDB 把数组每个元素展开成一条二级索引记录。

`Field_typed_array`（`field.h:4162`）内部维护 `m_conv_item`（元素类型对应的普通 Field）用于类型转换与 key 比较。

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

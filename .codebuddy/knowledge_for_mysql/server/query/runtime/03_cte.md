# 03 CTE 与递归 CTE：WITH 的解析与执行（算法级）

> 本篇覆盖：`WITH` 子句如何解析、存储，非递归 CTE 如何复用派生表机制，以及**递归 CTE 的 seed+收敛循环**。

## 目录

- [先有个整体印象](#先有个整体印象)
- [一、WITH 的解析与存储](#一with-的解析与存储)
- [二、非递归 CTE：就是 derived table](#二非递归-cte就是-derived-table)
- [三、递归 CTE 的核心算法](#三递归-cte-的核心算法)
- [四、共享物化与重扫](#四共享物化与重扫)
- [五、深潜补充：克隆表、校验清单、递归的三个细节](#五深潜补充克隆表校验清单递归的三个细节)

---

## 先有个整体印象

> ⚠️ **版本事实**：8.0.39 **没有** `sql/sql_cte.h/.cc`。CTE 逻辑已分散到 `sql_derived.cc`、`sql_union.cc`、`sql_parse.cc`、`parse_tree_nodes.cc`。后解析数据结构叫 `Common_table_expr`（`sql/table.h:4339`）。

CTE 的本质：**非递归 CTE 在解析层就被做成了 derived table**（复用 merge/materialize 全部机制），差别只在"多次引用共享一次物化"；**递归 CTE** 则多出"seed + 递归步收敛循环"，核心是 `FollowTailIterator` 读"上一轮新增的行"。

```
非递归 CTE ──▶ 就是 derived table（merge 或 materialize）
递归 CTE   ──▶ MaterializeIterator::MaterializeRecursive：
                 1. seed（anchor）物化一次
                 2. 递归块反复物化，直到不再新增行（收敛）
```

---

## 一、WITH 的解析与存储

### 1.1 WITH 挂到 Query_expression

```cpp
// parse_tree_nodes.cc:1945
bool PT_with_clause::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  pc->select->master_query_expression()->m_with_clause = this;  // 挂到所属 QE
  return false;
}
```

WITH 是 `<query expression>` 的前缀，所以存到 `Query_expression::m_with_clause`（不是 query block）。

### 1.2 Common_table_expr：三个数组是理解执行的关键

```cpp
// sql/table.h:4339
class Common_table_expr {
  Mem_root_array<Table_ref *> references;  // 定义之外的非递归引用
  bool recursive;                          // 是否递归 CTE
  Mem_root_array<Table_ref *> tmp_tables;  // 共享物化的所有 Table_ref
};
```

- `references`：CTE 定义**之外**的引用。
- `recursive`：标记为递归。
- `tmp_tables`：共享物化时指向同一临时表的所有 Table_ref（首个持有真正 create 出来的 TABLE）。

### 1.3 名字查找：find_common_table_expr（`sql_parse.cc:5733`）

从当前 query block 沿 `master_query_expression()->m_with_clause` 逐层向外查找（支持 CTE 作用域嵌套）。找到后：

- **非递归引用**：用 CTE 定义构造 `PT_subquery`，标 `node->m_is_derived_table = true`——**复用派生表机制**。
- **递归引用**：塞入 `(select 0)` 的 dummy 定义占位（`reparse_common_table_expr`），先蒙混过 name resolution。

`make_subquery_node`（`sql_parse.cc:5711`）保证同一 CTE 被多次引用时各自有独立子树（`references.size() >= 2` 时重新 parse 一份）。

### 1.4 递归/前向引用判定：PT_with_clause::lookup（`sql_parse.cc:5811`）

`m_most_inner_in_parsing` 记录当前正在 contextualize 的 CTE。在 WITH 列表从左到右扫描，一旦越过"正在解析的那个 CTE"（right_bound），再往后就是前向引用——被禁止，除非声明 `RECURSIVE` 且匹配的是它自己（自引用 = 递归引用）。

---

## 二、非递归 CTE：就是 derived table

**核心结论**：非递归 CTE 引用在解析层被做成了 derived table，完整复用 `merge_derived`（合并）或 `materialize_derived`（物化）。

- **merge 判定**：`Query_expression::is_mergeable()`（`sql_lex.cc:3807`）——无 set-op、无 GROUP BY/HAVING/DISTINCT/LIMIT/窗口 才可 merge；`merge_heuristic`（`:3836`）再排除含相关子查询/修改变量。
- **与普通 derived table 的差别**：①所有引用共享一个 `Common_table_expr`；②多次引用时物化一次、共享。

---

## 三、递归 CTE 的核心算法

### 3.1 结构校验（`sql_derived.cc:324`）

遍历 UNION 的所有 query block，校验 SQL 标准约束：

- 必须是 UNION（`ER_CTE_RECURSIVE_REQUIRES_UNION`）
- 递归块必须在 UNION 下、禁止右嵌套 UNION
- 递归成员不允许 ORDER BY/LIMIT/DISTINCT
- 非递归块（anchor）必须排在递归块之前
- 至少要有 anchor

最后 `derived->first_recursive = last_non_recursive->next_query_block()` 指向第一个递归块。

### 3.2 prepare 阶段：建临时表 + 递归引用替换（`sql_union.cc:741`）

```cpp
if (sl == first_recursive) {
  derived_table->setup_materialized_derived_tmp_table(thd);   // anchor 类型确定后建表
}
if (sl->recursive_reference)
  derived_table->common_table_expr()->substitute_recursive_reference(thd, sl);
```

`substitute_recursive_reference`（`sql_derived.cc:240`）：递归引用此前是 `(select 0)` 的 dummy derived table，这里克隆一个指向物化临时表的 TABLE 句柄，摘掉 dummy 的 query expression，变成"直接读临时表"的引用。

### 3.3 运行期：seed + 收敛循环（`composite_iterators.cc:1004`）

`MaterializeIterator` 检测到 `is_recursive()` 时走 `MaterializeRecursive()`，算法注释（:979-1003）写明：

```
1. 物化所有非递归 query block（seed/anchor），一次。
2. 物化所有递归 query block。
3. 重复 2 直到没有 block 再写入任何行（收敛）。
   每次物化只看到上一轮新增的行（FollowTailIterator）。
```

核心循环（`composite_iterators.cc:1056`）：

```cpp
ha_rows stored_rows = 0;
// 把总行数指针交给每个递归 reader（FollowTailIterator）
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (qb.is_recursive_reference)
    qb.recursive_reader->set_stored_rows_pointer(&stored_rows);

// 第一步：物化 seed
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (!qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);

// 第二步+第三步：反复物化递归块直到收敛
ha_rows last_stored_rows;
do {
  last_stored_rows = stored_rows;
  for (const QueryBlock &qb : m_query_blocks_to_materialize)
    if (qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);
} while (stored_rows > last_stored_rows);   // 不再新增即收敛
```

### 3.4 FollowTailIterator：读"上一轮新增的行"（`basic_row_iterators.cc:358`）

```cpp
int FollowTailIterator::Read() {
  if (m_read_rows == *m_stored_rows) {
    return -1;   // 已读到当前总行数 → EOF；关键是不让底层 cursor 真正撞 EOF，
                 // 否则 MEMORY/InnoDB 的 scan 会永久卡在 EOF，看不到后续插入的新行
  }
  if (m_read_rows == m_end_of_current_iteration) {
    if (++m_recursive_iteration_count > thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;   // 超深度报错
    }
    m_end_of_current_iteration = *m_stored_rows;  // 本轮终点 = 当前总行数
  }
  int err = table()->file->ha_rnd_next(m_record);  // 顺序读下一行
  if (err) return HandleError(err);
  ++m_read_rows;
  return 0;
}
```

精髓：

- `m_stored_rows` 指向物化器的总行数计数器。
- 每个递归步开始时 `m_end_of_current_iteration = *m_stored_rows` 记下"本步之前已有多少行"。
- 递归引用只顺序扫描 `[上一轮终点, 当前总数)` 区间，即**上一轮新产生**的行。
- 扫描位置追上总行数时返回 EOF 而不推进底层游标（关键！），从而下一轮写入新行后还能继续读。

### 3.5 深度限制

`cte_max_recursion_depth`（`sys_vars.cc:1012`）默认 **1000**，SESSION 级。

---

## 四、共享物化与重扫

### 4.1 多引用只物化一次：clone_tmp_table（`sql_derived.cc:170`）

同一 CTE 的第二个及后续引用不再 `create_tmp_table`，而是 `open_table_from_share` 基于同一个 `TABLE_SHARE` 克隆一个新的 TABLE 句柄——**共享底层数据、各自独立游标**。第一个是写者，克隆是读者。

`setup_materialized_derived_tmp_table`（`sql_derived.cc:841`）：`tmp_tables.size() > 0` 时复用，否则首次真正创建。

### 4.2 rematerialize 参数

`MaterializeIterator::Init`（`composite_iterators.cc:824`）：

- 非相关 CTE `rematerialize=false`：多次引用共享一次物化，之后只重扫。
- 相关（lateral/correlated）CTE 或依赖外层值的派生表 `rematerialize=true`：每次 `Init()` 清空重建。

`use_shared_cte_materialization` 判定（`:841`）：`m_cte != nullptr && !m_rematerialize` 且 `tmp_tables` 里已有 `materialized==true` 的表，则复用不再物化。

### 4.3 FOLLOW_TAIL 访问路径

`create_table_access_path`（`sql_executor.cc:4693`）：

```cpp
if (range_scan != nullptr)      path = range_scan;
else if (table_ref->is_recursive_reference()) path = NewFollowTailAccessPath(...);  // 递归引用
else                            path = NewTableScanAccessPath(...);
```

`AccessPath::FOLLOW_TAIL`（`access_path.h:222`）→ `FollowTailIterator`（`access_path.cc:498`）。

---


## 五、深潜补充：克隆表、校验清单、递归的三个细节

### 5.1 clone_tmp_table 的关键参数（`sql_derived.cc:170-229`）

```cpp
open_table_from_share(..., db_stat=0, EXTRA_RECORD | DELAYED_OPEN);   // :187-203
```

注释原文明确：**否则写者写行会覆盖读者刚读的行**——`db_stat=0` + `EXTRA_RECORD` 保证克隆表有独立的 record buffer，读者和写者互不干扰。

其余：`t->reginfo.lock_type = TL_WRITE`；`hash_field` 一并拷贝（`:219-221`）。

### 5.2 结构校验的完整清单（`sql_derived.cc:324-428`）

除"必须有 UNION"之外，还有 5 条容易被忽略的禁止项：

| 禁止项 | 位置 | 原因 |
|--------|------|------|
| ORDER BY over UNION | `:331-352` | 会引入 MyISAM 临时表（递归不能用） |
| 右嵌套 UNION | `:369-380` | — |
| 递归块内有 ORDER BY/LIMIT/DISTINCT | `:381-394` | 破坏逐轮追加语义 |
| `last_distinct` 后跟 ALL | `:395-412` | 去重语义冲突 |
| 非递归块必须在最前 | `:414-425` | — |

### 5.3 共享物化的读者同步

`composite_iterators.cc:844-850`：`use_shared_cte_materialization` 判定后只置 `table->materialized = true`；`Table_ref::materialize_derived`（`sql_derived.cc:1698-1706`）经 **`Derived_refs_iterator`**（`table.h:4379`）把所有**克隆**都标记为已物化（不只是主表）。

### 5.4 递归的三个细节

1. **强制严格模式**（`composite_iterators.cc:1039-1047` 的 `Strict_error_handler`）：列类型由 **anchor 成员**决定，若非严格模式下截断（如 anchor 是 `VARCHAR(10)`，递归部分产出更长值），会导致**无限递归**（值永不收敛）——所以非严格模式也要报错。
2. **深度计数是"轮数"不是行数**：`m_recursive_iteration_count` 在 `Init` 归零（`:347-349`），**先自增再比较**（`:399-406`）。`cte_max_recursion_depth` 默认 **1000**（`sys_vars.cc:1012-1018`）。
3. **本版无 `MAX_RECURSION` hint**——只在 `composite_iterators.cc:1024` 的注释里被提及，实际只能靠系统变量。

### 5.5 溢出转 InnoDB 与相关 CTE 重置

- **溢出**：`MaterializeQueryBlock` 中 `create_ondisk_from_heap`（`composite_iterators.cc:1151-1173`）后要调 `recursive_reader->RepositionCursorAfterSpillToDisk()`（`basic_row_iterators.cc:433`）——因为引擎换了，游标必须重定位。
- **相关 CTE 重置**：`m_cte->clear_all_references()`（`sql_union.cc:1589-1617`）清空所有 `tmp_tables` 并回绕游标；注释指出必须让 `materialized = false`，否则写者会跳过物化。

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → WITH (Common Table Expressions)*
- *MySQL 8.0 Reference Manual → Recursive Common Table Expressions*（`cte_max_recursion_depth`）

**两侧分工**
- 优化器侧（CTE 为何不能 merge、与 derived 的差异）见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)

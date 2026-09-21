# 08.3 构建（集合操作层）：UNION / INTERSECT / EXCEPT

> 集合操作在 `Query_expression`（unit）层面构建 AccessPath，与 query block 层面（`ConnectJoins`）是两条平行的构建路径。本篇讲 `Query_expression::create_access_paths()` 如何把各 block 的 AccessPath 组装成 set-op 的树；`streaming_allowed` 的完整判定、INTERSECT/EXCEPT 的计数器算法见 [`../07_optimize/19_set_operation.md`](../07_optimize/19_set_operation.md)。

## 目录

- [create_access_paths 完整控制流](#create_access_paths-完整控制流)
- [streaming_allowed 的两个否定条件](#streaming_allowed-的两个否定条件)
- [LIMIT/OFFSET 的预计算](#limitoffset-的预计算)
- [混合 UNION 的两种策略](#混合-union-的两种策略)
- [make_set_op_access_path 与 setup_materialize_set_op](#make_set_op_access_path-与-setup_materialize_set_op)

---

## create_access_paths 完整控制流

> `sql/sql_union.cc` 的 `Query_expression::create_access_paths()`。

### ① 简单查询提前返回

```cpp
void Query_expression::create_access_paths(THD *thd) {
  if (is_simple()) {
    JOIN *join = first_query_block()->join;
    assert(join && join->is_optimized());
    m_root_access_path = join->root_access_path();
    return;
  }
```

纯单查询块（无集合操作）时，直接复用 JOIN 的 `root_access_path()`，不构建额外节点——这就是 `ConnectJoins` 产出的那棵树的直接挂载点。

### ② 根 path 组装

```cpp
if (union_all_sub_paths->size() == 1) {
  m_root_access_path = (*union_all_sub_paths)[0].path;       // 只有一个子块，直接用
} else {
  assert(streaming_allowed);
  m_root_access_path = NewAppendAccessPath(thd, union_all_sub_paths);  // 多个，串联
}
```

只有一个子 path 时直接取用；多个时（流式场景）用 `NewAppendAccessPath` 串联所有 UNION ALL 子块（`APPEND` 节点的 cost 按子节点求和，见 `00_structure.md`）。

### ③ LIMIT/OFFSET 何时直接用 LimitOffset 节点

```cpp
// NOTE: If there's a fake_query_block, its JOIN's iterator already handles
// LIMIT/OFFSET, so we don't do it again here.
if (streaming_allowed && (limit != HA_POS_ERROR || offset != 0) &&
    (is_simple() || set_operation()->m_last_distinct == 0)) {
  m_root_access_path = NewLimitOffsetAccessPath(
      thd, m_root_access_path, limit, offset, calc_found_rows,
      /*reject_multiple_rows=*/false, ...);
}
```

两个关键判据：

- `fake_query_block` 的 JOIN 已经处理了 LIMIT/OFFSET，这里不重复
- `is_simple() || m_last_distinct == 0`：**只有全部是 UNION ALL（没有去重 operand）时，LIMIT 才能直接截断**——一旦有去重，必须先物化去重完再截断（否则截掉需要参与去重的行）

---

## streaming_allowed 的两个否定条件

> 注意：这里是**两个** `if/else if` 分支，不是三个（第三个"否定条件"实际是"非简单 + m_is_materialized"的组合，已并入条件 A）。

```cpp
bool streaming_allowed = true;
```

**条件 A —— 有排序，或整个 set op 被要求物化：**

```cpp
if (global_parameters()->order_list.size() != 0 ||
    (!is_simple() && set_operation()->m_is_materialized)) {
  // If we're sorting, we currently put it in a real table no matter what.
  // This is a legacy decision, because we used to not know whether filesort
  // would want to refer to rows in the table after the sort (sort by row ID).
  // We could probably be more intelligent here now.
  streaming_allowed = false;
}
```

★ 注释自陈是**遗留决定**：排序总是落真实表，因为历史上不确定 filesort 是否在排序后需要按 row ID 引用表行。`m_is_materialized` 为 true（作为派生表被要求物化）时也强制不流式。

**条件 B —— 顶层 INSERT/REPLACE SELECT（Halloween 防护）：**

```cpp
} else if ((thd->lex->sql_command == SQLCOM_INSERT_SELECT ||
            thd->lex->sql_command == SQLCOM_REPLACE_SELECT) &&
           thd->lex->unit == this) {
  // ... we don't want to insert any records before we're done scanning.
  // Otherwise, we would risk incorrect results and/or infinite loops,
  // as we'd be seeing our own records as they get inserted.
  // @todo Figure out if we can check for OPTION_BUFFER_RESULT instead;
  //       see bug #23022426.
  streaming_allowed = false;
}
```

边扫描边插入会读到自己刚插入的行（Halloween Problem），导致错误结果甚至死循环。TODO 指向 bug #23022426，建议未来改用 `OPTION_BUFFER_RESULT` 判断。

---

## LIMIT/OFFSET 的预计算

```cpp
ha_rows offset = global_parameters()->get_offset(thd);
ha_rows limit = global_parameters()->get_limit(thd);
if (limit + offset >= limit)
  limit += offset;
else
  limit = HA_POS_ERROR; /* purecov: inspected */
```

把 limit 与 offset 合并为总扫描上限（`limit += offset`），供物化时的 `max_rows` 下推使用。溢出保护：`limit + offset < limit` 表示溢出，置 `HA_POS_ERROR`（无限）。

---

## 混合 UNION 的两种策略

```cpp
if (!streaming_allowed || !is_simple()) {
  AppendPathParameters param;
  param.path = make_set_op_access_path(
      thd, /*parent*/ nullptr, m_query_term,
      streaming_allowed ? union_all_sub_paths : nullptr, calc_found_rows);
  param.join = nullptr;
  if (!streaming_allowed) union_all_sub_paths->push_back(param);
  // else filled in by make_set_op_access_path
}
```

两种策略：

| 场景 | `union_all_sub_paths` 传入 | 结果 |
|---|---|---|
| **不流式**（`!streaming_allowed`） | `nullptr` | 整个树走物化；`make_set_op_access_path` 返回单一物化 path，手动 push 进数组（只有一个元素） |
| **流式但非简单** | 非空数组 | `make_set_op_access_path` 往里填充：物化部分（UNION DISTINCT 块）+ 流式的 UNION ALL 块 |

★ 关键：**"流式"的本质是"UNION DISTINCT 块先物化，UNION ALL 块用 AppendIterator 追加"**。所以"流式"不是完全不物化，而是"去重部分物化、追加部分不物化"。

---

## make_set_op_access_path 与 setup_materialize_set_op

### make_set_op_access_path

递归遍历 `Query_term` 树，为每个 set-op 节点构造对应的 AccessPath：

- 叶子（`Query_block`）→ 用该 block 的 `root_access_path()`
- `UNION` → 去重部分物化 + `ALL` 部分进 `union_all_sub_paths`
- `INTERSECT` / `EXCEPT` → 物化 + 计数器列

它由 `create_access_paths` 以 `parent=nullptr` 从 `m_query_term` 递归调用。

### setup_materialize_set_op

`Query_term_set_op::setup_materialize_set_op()`（`sql/query_term.cc`）决定**哪些 operand 需要去重**：

```cpp
int64 idx = -1;
for (Query_term *term : m_children) {
  ++idx;
  bool activate_deduplication =
      idx <= m_last_distinct ||
      term_type() != QT_UNION; /* always for INTERSECT and EXCEPT */
  ...
  if (idx == m_last_distinct && idx > 0 && union_distinct_only)
    break;  // The rest will be done by appending.
}
```

判据两条：

1. `idx <= m_last_distinct` —— UNION 的前若干 operand 需要去重
2. `term_type() != QT_UNION` —— **INTERSECT / EXCEPT 永远激活去重**（语义上必须去重/计数）

`union_distinct_only` 参数：为 true 时只物化 UNION DISTINCT 块，`break` 提前退出（剩下的由 AppendIterator 处理）——这正是"流式"策略下 `make_set_op_access_path` 传入该参数的场景。

## 与 19 篇的分工

本篇聚焦"**落成什么 AccessPath 节点**"；`streaming_allowed` 的语义背景、`m_last_distinct` 切分点、INTERSECT/EXCEPT 的 `HalfCounter` 计数器算法、`pushdown_limit_order_by` 下推，见 [`../07_optimize/19_set_operation.md`](../07_optimize/19_set_operation.md)。

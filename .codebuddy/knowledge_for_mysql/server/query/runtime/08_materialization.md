# 08 物化（Materialization）深度解析

> 基于 MySQL 8.0.39 源码。本篇回答：**物化是怎么执行的？共享与重算怎么判断？去重、集合运算、递归各是怎么实现的？为什么 LIMIT 必须下推进物化器？**
>
> **边界**：本篇讲**物化的执行机制**（`MaterializeIterator` 的完整实现）。优化器侧"merge 还是 materialize"的决策见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)；物化用的临时表载体见 [`07_temptable.md`](07_temptable.md)；子查询物化（`subselect_hash_sj_engine`）见 [`02_subquery_runtime.md`](02_subquery_runtime.md)。

## 目录

- [概述](#概述)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
- [共享物化：clone_tmp_table() 与 DELAYED_OPEN](#共享物化clone_tmp_table-与-delayed_open)
- [参数化物化：invalidators 与 rematerialize](#参数化物化invalidators-与-rematerialize)
- [优化器侧：为什么会走到物化](#优化器侧为什么会走到物化)
- [已知限制与坑](#已知限制与坑)
- [参数](#参数)
- [参考](#参考)

---

## 概述

**是什么**：物化是把一个子查询（或派生表、CTE、集合运算的操作数）**先完整执行一遍，把结果写进一张内部临时表**，之后上游就当普通表来读。执行期的载体是 `MaterializeIterator`。

**解决什么问题**：同一个结果被多次读取时，避免重复执行子查询。典型场景：CTE 被引用多次、derived table 作为 join 的内表、UNION/INTERSECT/EXCEPT 的操作数收集、递归 CTE 的迭代。

**在链路的哪个位置**：优化器产出 `AccessPath::MATERIALIZE` → `CreateIteratorFromAccessPath()` 翻译成 `MaterializeIterator` → 执行期 `Init()` 完成全部写入，`Read()` 只从临时表转发。

**一句话概括其工作模式**：

> **`Init()` 把子查询一次跑到底写进临时表；`Read()` 一次一行往外吐。** 这是火山模型里"重活在 `Init()`"规律的典型代表。

---

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 物化的本质：一次全量扫描换多次廉价读取

MySQL 自己在 `MaterializeIterator` 的头注释里讲得很直白：

> Handles materialization; the first call to `Init()` will scan the given iterator to the end, store the results in a temporary table (optionally with deduplication), and then `Read()` will allow you to read that table repeatedly **without the cost of executing the given subquery many times**.

**并且承认一个反直觉的事实**：如果不需要去重、且只读一次，物化在语义上是个 **no-op**。但即便如此也不能直接把算子摘掉——

> …it's not necessarily straightforward to do so by just not inserting the iterator, as the optimizer will have set up everything (e.g., read sets, or what table upstream items will read from) **assuming the materialization will happen**, so the realistic option is setting up everything as if materialization would happen but not actually write to the table; see `StreamingIterator` for details.

**这是理解 MySQL 执行器的一把钥匙**：优化器的"计划"和算子的"存在"是耦合的。read_set、上游 Item 绑定在哪个字段、slice 怎么切，都是**按"会物化"布好的**。要省掉物化，只能造一个 `StreamingIterator` 去"假装物化"（求值写进字段但不落表），而不是删节点。

#### 2. LIMIT 为什么必须下推进物化器（经典的三处都不能放）

这是个非常好的设计权衡案例。`MaterializePathParameters` 里 `limit_rows` 的注释给了完整论证：

> Used for when pushing LIMIT down to `MaterializeIterator`; this is more efficient than having a `LimitOffsetIterator` above the `MaterializeIterator`, since we can stop materializing when there are enough rows. (This is especially important for recursive CTEs.) Note that we cannot have a `LimitOffsetIterator` _below_ the `MaterializeIterator`, as that would count wrong if we have deduplication, and would not work at all for recursive CTEs.

| 放的位置 | 问题 |
|---|---|
| **上面**（`Limit(Materialize(...))`） | 物化器仍要跑完整个子查询——**白写了一大堆行**，只是输出时截断 |
| **下面**（`Materialize(Limit(...))`） | ① 有去重时会**数错**（前 N 行里可能有重复被去掉，实际不足 N 行）；② **递归 CTE 完全不成立**（递归成员看到的是部分结果，LIMIT 会提前截断迭代） |
| **内建参数**（实际做法） | 在写循环里直接 `if (*stored_rows >= m_limit_rows) break;`，**后面的查询块根本不跑** |

所以 `m_limit_rows` 是物化器的内建参数（`HA_POS_ERROR` 表示无限制）。

**还有一个约束**：`assert(m_limit_rows == HA_POS_ERROR || table->is_union_or_table())`——**INTERSECT / EXCEPT 不能下推 LIMIT**。因为它们最终有多少行要等**读的时候看计数器**才知道，物化阶段无法提前停；它们的 LIMIT 由 `TableScanIterator` 在读路径上执行。

#### 3. 去重的两条路径与它们的权衡

物化常常伴随去重（UNION DISTINCT、DISTINCT、DuplicateWeedout）。MySQL 有两条路径：

| 路径 | 机制 | 适用 | 代价 |
|---|---|---|---|
| **唯一索引**（常用） | `ha_write_row()` 撞唯一键返回**可忽略错误**（`HA_ERR_FOUND_DUPP_KEY`），静默丢行 | 绝大多数情况 | 无 |
| **hash 字段**（兜底） | `check_unique_constraint()`：算 hash → 索引查找 → **逐行比对** | B-tree 键太长放不下时 | **hash 冲突会误判重复** |

源码注释明确说 hash 路径**不是常用路径**：

> Note that this is _not_ the common way of deduplicating as we go. The common method is to have a regular index on the table over the right columns, and in that case, `ha_write_row()` will fail with an ignorable error… However, B-tree indexes have limitations, in particular on length, that sometimes require us to do this instead.

**为什么"禁用一个索引"做不到**（`disable_deduplication_by_hash_field` 字段的注释）：UNION DISTINCT 与 UNION ALL 的块写进**同一张表**时，ALL 块不能走 `check_unique_constraint()`（否则会把本该保留的重复行误删）；但也不能用唯一索引去重，因为目标表上可能还有**别的索引要保留**，而实现上无法只禁用其中一个索引。于是只能用一个布尔开关跳过检查——代价是**这类混合 UNION 必须依赖 hash 字段**（`assert(doing_hash_deduplication())`）。

#### 4. 递归的实现"不符合标准描述，但等价且更高效"

`MaterializeRecursive()` 的注释坦承：

> This is not how the SQL standard specifies recursive CTE execution (it assumes building up the new result set from scratch for each iteration, using the previous iteration's results), but it is **equivalent, and more efficient** for the class of queries we support, since we don't need to re-create the same rows over and over again.

标准描述的是"每轮用上一轮结果重建整个结果集"；MySQL 是**边写边读、只追加新增行**，靠 `FollowTailIterator` 读"尾巴"。这避免了行被反复重建。

#### 5. 重算的代价：物化表不能再当 rowid 来源

`rematerialize=true` 时，AccessPath 会被标记 `SAFE_IF_SCANNED_ONCE`：

```
/// The given operation is safe if this access path is scanned once,
/// but not if it's scanned multiple times (e.g. used on the inner side
/// of a nested-loop join). A typical example of this is a derived table
/// or CTE that is rematerialized on each scan, so that references to
/// the old values (such as row IDs) are no longer valid.
```

也就是说：**重算的物化表不能作为 DuplicateWeedout / BKA 的 rowid 来源**——第二次扫到时 rowid 已经指向新数据了。这是正确性约束，不是性能优化。

### 理论溯源

- **物化（materialization）** 是经典查询执行算子：把中间结果显式化，是流式执行（pipelining）的对立面
- **递归 CTE** 的理论是 **最小不动点（least fixed point）**；MySQL 用迭代逼近不动点，收敛判据是"本轮不再新增行"
- **参数化物化**（invalidators）本质是**带依赖的缓存**：外层值变了缓存就失效——这是编译器里"循环不变量外提 + 失效"思路在数据库里的对应物

### 算法与数据结构

**`QueryBlock`（一个待物化的查询块）** 的关键字段：

| 字段 | 作用 |
|---|---|
| `subquery_iterator` | 真正的行来源；物化就是反复调它的 `Read()` 直到 EOF |
| `join` | 单表查询块没有 NLJ 包着，PFS batch mode 无人开启，需要这里代劳；另外用于 `set_executed()`（动态 range 优化依赖）与窗口函数状态复位 |
| `disable_deduplication_by_hash_field` | 混合 UNION ALL/DISTINCT 时跳过 hash 去重检查 |
| `copy_items` | 是否调 `copy_funcs()` 把 Item 求进字段；上游已有 windowing iterator 时置 false（否则会破坏窗口结果） |
| `m_total_operands` / `m_operand_idx` / `m_first_distinct` | INTERSECT / EXCEPT 的"共几个操作数 / 当前第几个 / 第一个带 DISTINCT 的下标" |
| `is_recursive_reference` + `recursive_reader` | 递归引用及其 `FollowTailIterator`（共享 `stored_rows` 指针、溢盘后重定位游标） |

**`MaterializeIterator` 的关键成员**：`m_query_blocks_to_materialize`、`m_table_iterator`（读路径）、`m_cte`（共享判断）、`m_query_expression`（清空相关 CTE）、`m_rematerialize`、`m_limit_rows`、`m_reject_multiple_rows`、`m_invalidators`。

**`HalfCounter`**：INTERSECT ALL 用 64 位整数拆成两个 32 位计数器——`c[0]`=左侧出现次数，`c[1]`=右侧匹配次数，最终取 `min(c[0], c[1])`。

### 他库对比与演进动机

| | MySQL 8.0 | PostgreSQL | SQL Server |
|---|---|---|---|
| 派生表默认 | **merge 优先**（`derived_merge=ON`） | 12 前物化优先，12 起可 `NOT MATERIALIZED` | 通常展开 |
| 物化复用 | CTE 多引用共享一张临时表；相关时通过 invalidator 失效重算 | CTE 一次物化（旧版本不可 merge） | spool / worktable |
| 递归中间态 | 追加写 + `FollowTailIterator` 读尾巴 | tuplestore + WorkTable Scan | 栈式 spool |

演进上，8.0 把物化统一收敛到 `MaterializeIterator` 一个算子（覆盖派生表、CTE、集合运算、递归、排序前落表、窗口前落表、semi-join 物化），这是"去 JOIN 化"重构的一部分。

---

## 核心实现

### 主链路

```
优化期：AccessPath::MATERIALIZE（参数在 MaterializePathParameters）
执行期：CreateIteratorFromAccessPath()
          ├─ 把每个子 AccessPath 翻译成 QueryBlock（并找出 recursive_reader）
          └─ 从 table_path 造出 m_table_iterator（读路径）

MaterializeIterator::Init()
  ├─ 首次：create_materialized_table()（视图/派生表）或 instantiate_tmp_table()
  ├─ 共享 CTE 判断：use_shared_cte_materialization → 只置标记，直接读
  ├─ 已物化：检查 invalidators 的 generation → 决定重扫还是重算
  ├─ 需要重算：set_not_started() + 清空/删除所有行 + 清空相关 CTE
  ├─ hash 去重时打开索引（scope guard 保证关闭）
  ├─ 递归 → MaterializeRecursive()；否则逐块 MaterializeQueryBlock()
  └─ 记录 generation 基线，初始化 m_table_iterator

MaterializeIterator::Read()
  └─（必要时切 ref item slice）→ m_table_iterator->Read()
```

### `Init()`：完整流程

#### 段 1：只有第一次才建表

```cpp
  if (!table()->materialized && table()->pos_in_table_list != nullptr &&
      table()->pos_in_table_list->is_view_or_derived()) {
    // Create the table if it's the very first time.
    if (table()->pos_in_table_list->create_materialized_table(thd())) {
      return true;
    }
  }
```

`create_materialized_table()` 里有两个关键点：

```cpp
    if (!table->is_created()) {
      Derived_refs_iterator it(this);
      while (TABLE *t = it.get_next())
        if (t->is_created()) {
          table->in_use = thd;
          if (open_tmp_table(table)) return true;
          break;
        }
    }
    ...
    /* create tmp table */
    if (instantiate_tmp_table(thd, table)) return true;

    table->file->ha_extra(HA_EXTRA_IGNORE_DUP_KEY);
```

- **CTE 的多个 clone 只要任意一个已经 `open_tmp_table`，本 clone 直接挂上去即可**（共享同一份 SE 数据）。这是 `Derived_refs_iterator` 的第一次登场
- **`ha_extra(HA_EXTRA_IGNORE_DUP_KEY)`**：告诉引擎"重复键不要报到语句级"，这样 `ha_write_row()` 的 dup 错误可由 `is_ignorable_error()` 判定后静默跳过——**这是 UNION DISTINCT 去重的基石**

#### 段 2：共享 CTE 物化的完整判断

```cpp
  const bool use_shared_cte_materialization =
      !table()->materialized && m_cte != nullptr && !m_rematerialize &&
      any_of(m_cte->tmp_tables.begin(), m_cte->tmp_tables.end(),
             [](const Table_ref *table_ref) {
               return table_ref->table != nullptr &&
                      table_ref->table->materialized;
             });

  if (use_shared_cte_materialization) {
    // If using an already materialized shared CTE table, update the
    // invalidators with the latest generation.
    for (Invalidator &invalidator : m_invalidators) {
      invalidator.generation_at_last_materialize =
          invalidator.iterator->generation();
    }
    table()->materialized = true;
  }
```

四个条件缺一不可：

| 条件 | 不满足会怎样 |
|---|---|
| `!table()->materialized`（本 clone 没物化过） | 已物化则走下面的重扫/重算分支 |
| `m_cte != nullptr`（必须是 CTE） | 普通派生表没有"共享"概念 |
| `!m_rematerialize` | 相关 CTE 必须每次重算 |
| **任一 clone 已 materialized** | 没有 → 我就是第一个写者 |

**为什么要同步 invalidator 的 generation**（容易忽略）：本 clone 并没有真的物化，但它的 invalitor 从上次到现在可能已推进了 N 代。若不把 `generation_at_last_materialize` 拉到当前值，下一次 `Init()` 一比较就"不相等"，会**白白触发一次重算**。同步之后语义才正确。

**`materialized` 标记如何传播给所有克隆**：注意这里**只设置本 clone**。传播是**惰性**的——每个克隆的物化器各自执行这段检查、各自置位。（旧路径 `Table_ref::materialize_derived()` 才是显式遍历所有引用。）

#### 段 3：已物化时，重扫还是重算

```cpp
  if (table()->materialized) {
    bool rematerialize = m_rematerialize;

    if (!rematerialize && !use_shared_cte_materialization) {
      // See if any lateral tables that we depend on have changed since
      // last time (which would force a rematerialization).
      //
      // TODO: It would be better, although probably much harder, to check
      // the actual column values instead of just whether we've seen any
      // new rows.
      for (const Invalidator &invalidator : m_invalidators) {
        if (invalidator.iterator->generation() !=
            invalidator.generation_at_last_materialize) {
          rematerialize = true;
          break;
        }
      }
    }

    if (!rematerialize) {
      // Just a rescan of the same table.
      const bool err = m_table_iterator->Init();
      m_table_iter_profiler.StopInit(start_time);
      return err;
    }
  }
```

- `!use_shared_cte_materialization` 这个短路很重要：**如果本次就是"借用别人的结果"，绝不能再因 invalidator 不一致而重算**（刚刚才同步过）
- 检测方式是**比较整数 generation，不是比较列值**。MySQL 自己在 TODO 里承认这是保守的：外层表只要"过了任何一行"（哪怕值相同）就会推进 generation 从而触发重算——**宁可多算，不可算错**
- 不重算时直接 `m_table_iterator->Init()`——这就是"重新扫描同一张表"

#### 段 4：清空重建

```cpp
  table()->set_not_started();

  if (!table()->is_created()) {
    if (instantiate_tmp_table(thd(), table())) return true;
    empty_record(table());
  } else {
    table()->file->ha_index_or_rnd_end();  // @todo likely unneeded => remove
    table()->file->ha_delete_all_rows();
  }
```

**为什么必须先 `set_not_started()`**：源码注释说得很直白——

> `// Mark the table as not started (default is just zero status), or read_system() and read_const() will forget to read the row.`

如果残留 `STATUS_NOT_FOUND`，会被误判为"空表"。

**为什么先 `ha_index_or_rnd_end()` 再 `ha_delete_all_rows()`**：MEMORY/InnoDB 的游标状态在批量删除后可能失效；尤其递归场景里 `FollowTailIterator::Init()` 靠 `file->inited` 判断"是不是首次初始化"（`MaterializeIterator::Init()` 对所有递归成员的读游标做 `ha_index_or_rnd_end()`，把 `inited` 置 false，正好当信号用）。

注意 `ha_delete_all_rows()` **不**清 `materialized` 标志——真正的清空是 `TABLE::empty_result_table()`（它会把 `materialized=false` 并回绕游标）。

**重算时必须连带清空"依赖外部值的嵌套物化"**，否则会读到上一轮的陈旧结果：

```cpp
  if (m_query_expression != nullptr)
    if (m_query_expression->clear_correlated_query_blocks()) return true;

  if (m_cte != nullptr) {
    if (m_cte->clear_all_references()) return true;
  }
```

`Common_table_expr::clear_all_references()` 的注释解释了"多引用游标同步"的两个目的：

> Above, emptying all clones is necessary, to rewind every handler (cursor) to the table's start. Setting `materialized=false` on all is also important or **the writer would skip materialization**… There is one "recursive table" which we don't find here: it's the UNION DISTINCT tmp table. It's reset in `unit::execute()` of the unit which is the body of the CTE.

即：①把每个 handler 的游标回绕到表头；②把 `materialized` 置回 false，**否则写者会以为已经物化过而跳过**。最后一句还坦白有个"漏网之鱼"——递归 CTE 的 UNION DISTINCT 临时表得靠 CTE 体所在 unit 的 `execute()` 去 reset。

`Init()` 里还有一段长注释记录了一个真实 bug 场景：

> `SELECT FROM ot WHERE EXISTS(WITH RECURSIVE cte (...) SELECT * FROM cte)` —— 若 CTE 外层相关，EXISTS 求值时 `ClearForExecution()` 会调 `clear_correlated_query_blocks()` 清掉 CTE（含递归定义里对自身的引用）。**但如果 owning query expression 被 merge 掉了**（如 `FROM ot SEMIJOIN cte ON TRUE`），就没有 Query_expression 了，它的 WITH 子句到不了。这个"lateral CTE"仍需彻底重置——由 `m_cte->clear_all_references()` 兜底。

#### 段 5：hash 去重索引与 scope guard

```cpp
  auto end_unique_index = create_scope_guard([&] {
    if (table()->file->inited == handler::INDEX) table()->file->ha_index_end();
  });

  if (doing_hash_deduplication()) {
    if (table()->file->ha_index_init(0, /*sorted=*/false)) return true;
  } else {
    // We didn't open the index, so we don't need to close it.
    end_unique_index.commit();
  }
```

scope guard 保证**任何返回路径**（含错误返回）都能关掉索引。没开就 `commit()`（取消清理），正常路径是 `rollback()`（执行清理）。注释强调：**这个索引只服务于写入时的去重**，与 `m_table_iterator` 读回数据时用的索引完全无关。

#### 段 6：物化主循环

```cpp
  ha_rows stored_rows = 0;

  if (m_query_expression != nullptr && m_query_expression->is_recursive()) {
    if (MaterializeRecursive()) return true;
  } else {
    for (const materialize_iterator::QueryBlock &query_block :
         m_query_blocks_to_materialize) {
      if (MaterializeQueryBlock(query_block, &stored_rows)) return true;
      if (table()->is_union_or_table()) {
        // For INTERSECT and EXCEPT, this is done in TableScanIterator
        if (m_reject_multiple_rows && stored_rows > 1) {
          my_error(ER_SUBQUERY_NO_1_ROW, MYF(0));
          return true;
        } else if (stored_rows >= m_limit_rows) {
          break;
        }
      }
    }
  }
```

- `stored_rows` **跨块累加**——所以 LIMIT 是全局的，不是每块 N 行
- `m_reject_multiple_rows && stored_rows > 1` → 标量子查询返回多行，报 `ER_SUBQUERY_NO_1_ROW`。注意是 `>1` 而非 `>=1`：**0 行对标量子查询意味着 NULL，是合法的**
- 两个检查都被 `is_union_or_table()` 包着——INTERSECT/EXCEPT 的行数要等读计数器才知道

#### 段 7：收尾

记录当前所有 invalitor 的 generation 作为下次比较的基线；`m_profiler.IncrementNumRows(stored_rows)` 累加本次物化行数。这也是为什么 EXPLAIN ANALYZE 里 "Materialize" 行的 `rows` 是**物化行数**，而它上面读表行的 `rows` 是**读取行数**（可因多次重扫而不同）。

### `MaterializeQueryBlock()`：物化的主循环

先看两个辅助 lambda。

**`read_counter`**——从 record[1] 读计数器（INTERSECT/EXCEPT 用）：

```cpp
  auto read_counter = [t, set_counter_0, set_counter_1]() -> ulonglong {
    assert(t->record[1] - t->record[0] == set_counter_1 - set_counter_0);
    t->set_counter()->set_field_ptr(set_counter_1);
    ulonglong cnt = static_cast<ulonglong>(t->set_counter()->val_int());
    t->set_counter()->set_field_ptr(set_counter_0);
    return cnt;
  };
```

`check_unique_constraint()` 找到匹配行时会把该行读进 **record[1]**，而计数器字段平时指向 record[0]，所以要临时把 `field_ptr` 挪到 record[1] 的对应偏移（`+ rec_buff_length`），读完还原。那个 `assert` 保证这个"指针算术技巧"成立。

**`spill_to_disk_and_retry_update_row`**——溢盘后重试 UPDATE：

```cpp
  auto spill_to_disk_and_retry_update_row = [t, this](THD *thd, int error) -> int {
    bool dummy;
    if (create_ondisk_from_heap(thd, t, error,
                                /*insert_last_record=*/false,
                                /*ignore_last_dup=*/true, &dummy))
      return true;
    // Table's engine changed; index is not initialized anymore.
    if (t->file->ha_index_init(0, /*sorted*/ false)) return true;

    // Inform each reader that the table has changed under their feet,
    // so they'll need to reposition themselves.
    for (const materialize_iterator::QueryBlock &query_b :
         m_query_blocks_to_materialize) {
      if (query_b.is_recursive_reference) {
        query_b.recursive_reader->RepositionCursorAfterSpillToDisk();
      }
    }
    // re-try update: 1. reposition to same row
    error = check_unique_constraint(t);
    assert(error == 0);
    return t->file->ha_update_row(t->record[1], t->record[0]);
  };
```

三步缺一不可：①换引擎并搬数据（`insert_last_record=false`，因为要重试的是 UPDATE 不是 INSERT）；②**重建索引**（换引擎后句柄失效）+ 通知所有递归 reader 重定位游标；③重新定位到同一行再 `ha_update_row(record[1], record[0])`（旧行在前、新行在后）。

#### 主体循环

```cpp
  JOIN *join = query_block.join;
  if (join != nullptr) {
    join->set_executed();  // The dynamic range optimizer expects this.
    if (join->m_windows.elements > 0 && !join->m_windowing_steps) {
      // Initialize state of window functions as window access path
      // will be shortcut.
      for (Window &w : join->m_windows) {
        w.reset_all_wf_state();
      }
    }
  }

  if (query_block.subquery_iterator->Init()) return true;

  PFSBatchMode pfs_batch_mode(query_block.subquery_iterator.get());
  const bool is_union_or_table = table()->is_union_or_table();

  while (true) {
    assert(is_union_or_table || m_limit_rows == HA_POS_ERROR);

    if (*stored_rows >= m_limit_rows) break;

    int error = query_block.subquery_iterator->Read();
    if (error > 0 || thd()->is_error())
      return true;
    else if (error < 0)
      break;
    else if (thd()->killed) {
      thd()->send_kill_message();
      return true;
    }

    // Materialize items for this row.
    if (query_block.copy_items) {
      if (copy_funcs(query_block.temp_table_param, thd())) return true;
    }
```

**要点**：

- `join->set_executed()`：动态 range 优化（运行期还能做 range 优化）依赖这个标记
- 窗口函数状态复位：走物化路径时窗口 access path 会被"抄近路"，必须手动 reset
- `PFSBatchMode` 是 RAII：开 performance schema batch mode 减少每行的 instrumentation 开销
- `Read()` 的**三态返回值**：`0`=OK、`-1`=EOF（正常跳出）、`1`=Error
- **`thd()->killed` 每轮都查**：物化一次要跑完整个子查询，**这是 KILL 最容易卡住的地方**

#### UNION 的去重

```cpp
    if (is_union_or_table) {
      if (query_block.disable_deduplication_by_hash_field) {
        assert(doing_hash_deduplication());
      } else if (!check_unique_constraint(t)) {
        continue;
      }
    }
```

`check_unique_constraint()` 返回 `true`="没找到重复（可以写）"、`false`="找到了（跳过）"。注意它的第一行：

```cpp
bool check_unique_constraint(TABLE *table) {
  ulonglong hash;

  if (!table->hash_field) return true;
  ...
```

**没有 hash 字段时它永远返回 true（不去重）**——去重靠唯一索引 + `ha_write_row()` 的可忽略错误。两条路径是互斥的。

#### INTERSECT / EXCEPT 的计数器算法

这是物化里最精巧的一段。左操作数（operand 0）建立计数，右操作数只更新计数、**从不写行**。

**左侧（operand 0）**：

| 运算 | 新行 | 重复行 |
|---|---|---|
| **EXCEPT** | `counter := 1`，写入 | `counter := counter+1`，`ha_update_row` 后 `continue` |
| **INTERSECT DISTINCT** | `counter := N-1`（N=操作数个数） | 已写过，直接 `continue` |
| **INTERSECT ALL** | `HalfCounter c(0); c[0]=1;` 写入 | `c[0]++`（左计数），更新后 `continue`；超 2³² 报 `ER_INTERSECT_ALL_MAX_DUPLICATES_EXCEEDED` |

**右侧（operand > 0）**首先是这条：

```cpp
      if (check_unique_constraint(t)) {
        // row doesn't have a counter-part in left side, so we can ignore it
        continue;
      }
```

右侧的行如果在左侧不存在，直接忽略——它不可能出现在结果里。

- **EXCEPT 右侧**：`cnt > 0` 时，若当前操作数下标 < `m_first_distinct` 则 `counter-1`（EXCEPT ALL 的逐个抵消），否则直接置 0（EXCEPT DISTINCT 一次性清空）
- **INTERSECT DISTINCT 右侧**：`if (cnt == m_total_operands - m_operand_idx)` 才减 1——这个相等判断的含义是"这个行自上次以来还没被更右的操作数匹配过"，从而保证**每个操作数只贡献一次递减**
- **INTERSECT ALL 右侧**：`if (c[1] + 1 <= left_side) c[1]++`——右计数不超过左计数（因为交集最多只能有左计数那么多）

最后**无条件** `continue;  // right hand side of EXCEPT or INTERSECT, never write`。

**结果的读出**在 `TableScanIterator::Read()` 里按计数器过滤：EXCEPT DISTINCT 是 `cnt >= 1` 出一row；EXCEPT ALL 是 `m_remaining_dups = cnt` 出 cnt 行；INTERSECT DISTINCT 是 `cnt == 0`；INTERSECT ALL 是 `min(c[0], c[1])`。

#### 写入与三种结局

```cpp
    error = t->file->ha_write_row(t->record[0]);
    if (error == 0) {
      ++*stored_rows;
      continue;
    }
    // create_ondisk_from_heap will generate error if needed.
    if (!t->file->is_ignorable_error(error)) {
      bool is_duplicate;
      if (create_ondisk_from_heap(thd(), t, error,
                                  /*insert_last_record=*/true,
                                  /*ignore_last_dup=*/true, &is_duplicate))
        return true;
      // Table's engine changed; index is not initialized anymore.
      if (t->hash_field) t->file->ha_index_init(0, false);
      if (!is_duplicate &&
          (t->is_union_or_table() || query_block.m_operand_idx == 0))
        ++*stored_rows;

      for (const materialize_iterator::QueryBlock &query_b :
           m_query_blocks_to_materialize) {
        if (query_b.is_recursive_reference) {
          query_b.recursive_reader->RepositionCursorAfterSpillToDisk();
        }
      }
    } else {
      // An ignorable error means duplicate key, ie. we deduplicated
      // away the row. This is seemingly separate from
      // check_unique_constraint(), which only checks hash indexes.
    }
```

| 结局 | 条件 | 处理 |
|---|---|---|
| 成功 | `error == 0` | `++*stored_rows` |
| **可忽略错误**（重复键） | `is_ignorable_error(error)` | **什么都不做**——注释说"we deduplicated away the row"，不去重计数也不算已存储行 |
| 真错误（通常是 `HA_ERR_RECORD_FILE_FULL`） | 其余 | 转磁盘表 + 重建索引 + 重定位递归游标 + 按需补计行数 |

`++*stored_rows` 的条件 `!is_duplicate && (is_union_or_table() || m_operand_idx == 0)`：溢盘补写的最后一行若撞了重复键则不计；INTERSECT/EXCEPT 只有左侧块的新增行才计入。

### 递归物化：`MaterializeRecursive()`

#### 强制严格模式（一个由真实 bug 驱动的决策）

源码有一整段注释解释为什么递归物化时**即便 session 非严格模式也要报错**：

> For RECURSIVE, beginners will forget that: the CTE's column types are defined by the non-recursive member… That will cause silent truncation and possibly an **infinite recursion** due to a condition like: `LENGTH(growing_col) < const` … which is always satisfied due to truncation.
>
> If we only raised warnings: it will not interrupt an infinite recursion… So warnings are useless. Instead, we send a truncation error… as WITH RECURSIVE is a new feature we don't have to carry the permissiveness of the past, so we **send an error even if in non-strict mode**.

即：列类型由 anchor 决定 → 递归成员被强转 → 截断 → 终止条件永远成立 → 无限递归。所以用 `Strict_error_handler` 强制报错，并把 `check_for_truncated_fields` 降为 `CHECK_FIELD_WARN` 让 handler 有机会转成 error（而不是被上层吞掉）。

#### 收敛循环

```cpp
  ha_rows stored_rows = 0;

  // Give each recursive iterator access to the stored number of rows
  for (const materialize_iterator::QueryBlock &query_block :
       m_query_blocks_to_materialize) {
    if (query_block.is_recursive_reference) {
      query_block.recursive_reader->set_stored_rows_pointer(&stored_rows);
    }
  }

  // First, materialize all non-recursive query blocks.
  for (const materialize_iterator::QueryBlock &query_block :
       m_query_blocks_to_materialize) {
    if (!query_block.is_recursive_reference) {
      if (MaterializeQueryBlock(query_block, &stored_rows)) return true;
    }
  }

  // Then, materialize all recursive query blocks until we converge.
  ha_rows last_stored_rows;
  do {
    last_stored_rows = stored_rows;
    for (const materialize_iterator::QueryBlock &query_block :
         m_query_blocks_to_materialize) {
      if (query_block.is_recursive_reference) {
        if (MaterializeQueryBlock(query_block, &stored_rows)) return true;
      }
    }
    // 第一次跑完就关掉 trace（REPEATED_SUBSELECT 未开启时）
    if (!disabled_trace &&
        !trace.feature_enabled(Opt_trace_context::REPEATED_SUBSELECT)) {
      trace.disable_I_S_for_this_and_children();
      disabled_trace = true;
    }
  } while (stored_rows > last_stored_rows);
```

- **收敛判据是"本轮有没有新行写进去"**，不是"有没有迭代"——因为一次 `MaterializeQueryBlock()` 内部可能就跑完好几轮（边写边读）
- UNION DISTINCT 时被去重掉的行**不计数**（`stored_rows` 只在真正写入时 `++`）
- **trace 抑制**：递归 N 次会产出 N 份 trace，把 `OPTIMIZER_TRACE` 撑爆，所以第一次跑完就关

#### `FollowTailIterator`：为什么不让它撞 EOF（本篇最有价值的一段）

`Read()` 里那段长注释记录了两个真实的引擎行为问题：

```cpp
int FollowTailIterator::Read() {
  if (m_read_rows == *m_stored_rows) {
    /*
      Return EOF without even checking if there are more rows
      (there isn't), so that we can continue reading when there are.
      There are two underlying reasons why we need to do this,
      depending on the storage engine in use:

      1. For both MEMORY and InnoDB, when they report EOF,
         the scan stays blocked at EOF forever even if new rows
         are inserted later. (InnoDB has a supremum record, and
         MEMORY increments info->current_record unconditionally.)

      2. Specific to MEMORY, inserting records that are deduplicated
         away can corrupt cursors that hit EOF. Consider the following
         scenario:

         - write 'A'
         - write 'A': allocates a record, hits a duplicate key error, leaves
           the allocated place as "deleted record".
         - init scan
         - read: finds 'A' at #0
         - read: finds deleted record at #1, properly skips over it, moves to
           EOF
         - even if we save the read position at this point, it's "after #1"
         - close scan
         - write 'B': takes the place of deleted record, i.e. writes at #1
         - write 'C': writes at #2
         - init scan, reposition at saved position
         - read: still after #1, so misses 'B'.

         In this scenario, the table is formed of real records followed by
         deleted records and then EOF.

       To avoid these problems, we keep track of the number of rows in the
       table by holding the m_stored_rows pointer into the MaterializeIterator,
       and simply avoid hitting EOF.
     */
    return -1;
  }
  ...
```

**这段解释了"为什么要用一个指向栈上 `stored_rows` 的指针"这种看似 hack 的设计**：

1. MEMORY 和 InnoDB 一旦报 EOF，游标**永久卡在 EOF**，后续插入的新行看不到
2. MEMORY 特有：被去重掉的写会留下"deleted record"空洞，后来写入的行复用空洞，导致"保存并恢复读位置"后**漏行**（例子里 'B' 被漏掉）
3. 解法：**永远不撞 EOF**——用 `m_read_rows == *m_stored_rows` 主动返回 -1

溢盘后游标重定位（`RepositionCursorAfterSpillToDisk`）用 InnoDB 的 PK 编码：

```cpp
bool reposition_innodb_cursor(TABLE *table, ha_rows row_num) {
  assert(table->s->db_type() == innodb_hton);
  if (table->file->ha_rnd_init(false)) return true;
  // Per the explanation above, the wanted InnoDB row has PK=row_num.
  uchar rowid_bytes[6];
  encode_innodb_position(rowid_bytes, sizeof(rowid_bytes), row_num);
  /*
    Go to the row, and discard the row. That places the cursor at
    the same row as before the engine conversion, so that rnd_next() will
    read the (row_num+1)th row.
  */
  return table->file->ha_rnd_pos(table->record[0], rowid_bytes);
}
```

（"go to the row, and discard the row"——定位到目标行后丢弃内容，游标停在那里，这样下一次 `rnd_next()` 正好读第 row_num+1 行。）

### `Read()` 与读路径

```cpp
template <typename Profiler>
int MaterializeIterator<Profiler>::Read() {
  const typename Profiler::TimeStamp start_time = Profiler::Now();
  /*
    Enable the items which one should use if one wants to evaluate
    anything (e.g. functions in WHERE, HAVING) involving columns of this
    table.
  */
  if (m_ref_slice != -1) {
    assert(m_join != nullptr);
    if (!m_join->ref_items[m_ref_slice].is_null()) {
      m_join->set_ref_item_slice(m_ref_slice);
    }
  }

  const int err = m_table_iterator->Read();
  m_table_iter_profiler.StopRead(start_time, err == 0);
  return err;
}
```

**为什么 `Read()` 这么薄**：所有行在 `Init()` 时已经在临时表里了。唯一职责是**切 ref item slice**——只有"同 JOIN 内物化"（如排序前落表）才需要；跨 JOIN 的派生表/CTE 里 `m_ref_slice == -1`（上游用外层 JOIN 自己的 slice）。

**`m_table_iterator` 的多种形态**（物化器完全不关心是哪一种）：

| 形态 | 何时 |
|---|---|
| `TableScanIterator` | 默认（物化表无索引时）；集合运算的计数器过滤与 LIMIT 在这里做 |
| `IndexScanIterator` / 各种 ref 迭代器 | 物化表上有派生键（`add_derived_key`）或 UNION DISTINCT 的唯一索引 |
| `FollowTailIterator` | 递归 CTE 的递归引用——**唯一一个"EOF 是暂时状态"的读迭代器** |
| `ConstIterator` / `EQRefIterator` | 优化期已知 0/1 行的 const 派生表 |

创建时有个易被忽略的分支：**如果递归引用被优化掉了**（如 `WHERE FALSE`），`FindSingleIteratorOfType` 找不到 `FOLLOW_TAIL`，就把 `is_recursive_reference` 改回 `false`——降级为普通块。

---

## 共享物化：`clone_tmp_table()` 与 `DELAYED_OPEN`

CTE 被引用 N 次时，**数据是共享的，但每个引用有自己的 `TABLE` 句柄**——多个 `TABLE` 共用一个 `TABLE_SHARE`。`sql/sql_derived.cc` 开头的注释是全仓对共享物化最权威的说明：

```
   (2) Non-recursive CTE referenced more than once:
   - multiple TABLEs, one TABLE_SHARE.
   - The first ref in setup_materialized_derived() calls
   create_tmp_table(); others call open_table_from_share().
   - The first ref in create_derived() calls instantiate_tmp_table()
   (which calls handler::create() then open_tmp_table()); others call
   open_tmp_table(). open_tmp_table() calls handler::open().
   - The first ref in materialize_derived() evaluates the subquery and does
   all writes to the tmp table.
   - Finally all refs set up a read access method (table scan, index scan,
   index lookup, etc) and do reads, possibly interlaced (example: a
   nested-loop join of two references to the CTE).
   - The storage engine (MEMORY or InnoDB) must be informed of the uses above;
   this is done by having TABLE_SHARE::ref_count>=2 for every handler::open()
   call.
```

**关键：克隆必须带 `EXTRA_RECORD | DELAYED_OPEN`，而且 `db_stat` 传 0**（这是最容易忽略的一点）：

```cpp
TABLE *Common_table_expr::clone_tmp_table(THD *thd, Table_ref *tl) {
  TABLE *first = tmp_tables[0]->table;
  // Allocate clone on the memory root of the TABLE_SHARE.
  TABLE *t = static_cast<TABLE *>(first->s->mem_root.Alloc(sizeof(TABLE)));
  if (open_table_from_share(thd, first->s, tl->alias,
                            /*
                              Pass db_stat == 0 to delay opening of table in SE,
                              as table is not instantiated in SE yet.
                            */
                            0,
                            /* We need record[1] for this TABLE instance. */
                            EXTRA_RECORD |
                                /*
                                  Use DELAYED_OPEN to have its own record[0]
                                  (necessary because db_stat is 0).
                                  Otherwise it would be shared with 'first'
                                  and thus a write to tmp table would modify
                                  the row just read by readers.
                                */
                                DELAYED_OPEN,
                            0, t, false, nullptr))
    return nullptr;
  assert(t->s == first->s && t != first && t->file != first->file);
  ...
}
```

**为什么 `DELAYED_OPEN` 不可省？** 注释说得很直白：否则克隆会与 `first` **共享 `record[0]`**，"a write to tmp table would modify the row just read by readers"——**写者写入时会篡改读者刚读到的那一行**。递归 CTE 的"边写边读"尤其依赖这一点。

**没有"数据引用计数"**：共享靠 `TABLE_SHARE::ref_count` / `tmp_handler_count`（告诉引擎有多少个 handler 打开着），数据本身随 `TABLE_SHARE` 在查询结束时释放。遍历所有克隆靠 `Derived_refs_iterator`。

**三个同步点**——共享物化不是"只读共享"，而是**写者 + 多读者并存，任何结构变更必须广播到所有克隆**：

| 同步点 | 函数 | 为什么必须同步 |
|---|---|---|
| 物化标记 | `Table_ref::materialize_derived()` | 任一克隆已物化 ⇒ 全部标记，避免重复写 |
| 引擎切换 | `create_ondisk_from_heap()` | 内存转 InnoDB 时所有克隆的 handler 都要换（见 [`07_temptable.md`](07_temptable.md)） |
| 索引 | `Table_ref::generate_keys()` | 索引要在**每个克隆的 `key_info`** 上都建 |

### 递归 CTE 的克隆：边写边读

递归 CTE 的引用分为**写者**（非递归引用持有的 `TABLE`）和**读者**（递归引用持有的克隆）。两者操作同一张表，写者写的同时读者在读。`MaterializeRecursive()` 的注释明确说这不是 SQL 标准的做法但等价且更高效：

```
  This is not how the SQL standard specifies recursive CTE execution
  (it assumes building up the new result set from scratch for each iteration,
  using the previous iteration's results), but it is equivalent, and more
  efficient for the class of queries we support, since we don't need to
  re-create the same rows over and over again.
```

因此有一个硬约束——**递归表不能有主键**，否则 InnoDB 按 PK 序返回，破坏"插入序"：

```cpp
  /*
    The with-recursive algorithm needs the table scan to return rows in
    insertion order.
    For MEMORY and Temptable it is true.
    For InnoDB: InnoDB's table scan returns rows in PK order. If the PK
    is (not) the autogenerated autoincrement InnoDB ROWID, PK order will (not)
    be the same as insertion order.
    So let's verify that the table has no MySQL-created PK.
  */
  if (unit->is_recursive()) {
    assert(table->s->primary_key == MAX_KEY);
  }
```

---

## 参数化物化：invalidators 与 rematerialize

### 什么时候 `m_rematerialize = true`

| 场景 | 条件 |
|---|---|
| **相关派生表 / LATERAL** | query expression 带 `UNCACHEABLE_DEPENDENT`（体内引用外层列）。**CTE 例外**：`rematerialize=false`，改由 `clear_all_references()` 在合适时机清空（因为 CTE 要支持共享） |
| **依赖表函数**（`JSON_TABLE`） | `tab->dependent` |
| **JOIN 内部物化**（排序前 / 窗口前 / semijoin 物化） | **无条件 true**——这些临时表是算子私有的中间结果，每次重执行都必须重来 |
| **表函数物化** | `is_table_function() && dependent` |

触发 SQL 举例：

```sql
-- 相关派生表（体内引用外层列）
SELECT * FROM t1 WHERE t1.a > (SELECT MAX(x) FROM (SELECT b AS x FROM t2 WHERE t2.c = t1.c) d);

-- LATERAL
SELECT * FROM t1, LATERAL (SELECT * FROM t2 WHERE t2.a = t1.a) d;
```

### `CacheInvalidatorIterator`：只维护一个代数

```cpp
  class CacheInvalidatorIterator final : public RowIterator {
   public:
    bool Init() override { ++m_generation; return m_source_iterator->Init(); }
    int Read() override { ++m_generation; return m_source_iterator->Read(); }
    void SetNullRowFlag(bool is_null_row) override {
      ++m_generation;
      m_source_iterator->SetNullRowFlag(is_null_row);
    }
    int64_t generation() const { return m_generation; }
```

**`Init()` / `Read()` / `SetNullRowFlag()` 三者都自增**。为什么 `SetNullRowFlag()` 也要？因为外连接产生 NULL 补充行时并没有 `Read()` 成功，但**外层行确实变了**，物化内容必须失效——这是个很容易漏的正确性细节。

### 挂在哪个表上：只挂"最后一个依赖"

```cpp
      auto deps = table_ref->derived_query_expression()->m_lateral_deps;
      plan_idx last = NO_PLAN_IDX;
      for (JOIN_TAB **tab2 = join->map2table; deps; tab2++, deps >>= 1) {
        if (deps & 1) last = std::max(last, (*tab2)->idx());
      }
      /*
        We identified the last dependency of table_ref in the plan, and it's
        the table whose reading must trigger rematerialization of table_ref.
      */
```

只挂在**计划中最后一个被依赖的表**上（`std::max`）——挂在靠前的表上会白白发很多次失效。

### `pending_invalidators`：外连接的 NULL 补充行陷阱

如果被依赖的表属于**更外层的 outer join nest**，不能立即挂：

```cpp
    for (plan_idx table_idx :
         BitsSetIn(qep_tab->lateral_derived_tables_depend_on_me)) {
      if (table_idx < last_idx) {
        table_path = NewInvalidatorAccessPathForTable(thd, table_path, qep_tab,
                                                      table_idx);
      } else {
        // The table to invalidate belongs to a higher outer join nest,
        // which means that we cannot emit the invalidator right away --
        // the outer join we are a part of could be emitting NULL-complemented
        // rows that also need to invalidate the cache in question.
        // We'll deal with them in as soon as we get into the same join nest.
        // (But if we deal with them later than that, it might be too late!)
        pending_invalidators->push_back(PendingInvalidator{
            qep_tab, /*table_index_to_attach_to=*/table_idx});
      }
    }
```

**原因**：外层 join 可能正在吐 NULL 补充行，这些行**也必须让缓存失效**。如果挂在错误的位置，NULL 行就不会触发失效 ⇒ 读到陈旧数据。所以先存进 `pending_invalidators`，等递归回到同一个 join nest 再挂。

### 保守失效的代价

检测只比较 generation 而非列值，MySQL 自己在 TODO 里承认：

> TODO: It would be better, although probably much harder, to check the actual column values instead of just whether we've seen any new rows.

**为什么不做成值比较**：派生表可能依赖多个外层列、来自不同表，且物化结果本身可能已跨表聚合；维护"依赖列值 → 缓存有效性"的映射，代价远高于重算一次（尤其外层行数不多时）。

### EXPLAIN 里能看到

- 树形 EXPLAIN：`Materialize ... (invalidate on row from t1)`
- 传统 EXPLAIN 的 Extra 列：`Rematerialize`

### rematerialize 的副作用

```cpp
  if (rematerialize) {
    path->safe_for_rowid = AccessPath::SAFE_IF_SCANNED_ONCE;
  } else {
    path->safe_for_rowid = AccessPath::SAFE;
  }
```

以及 `NewMaterializeAccessPath()` 里：

```cpp
  if (rematerialize) {
    // There's no point in adding invalidators if we're rematerializing
    // every time anyway.
    param->invalidators = nullptr;
  }
```

即：重算的物化表**不能作为 rowid 来源**（DuplicateWeedout / BKA 会拿到失效的 rowid）；既然每次都重算，invalidator 也就没必要挂了。

---

## 优化器侧：为什么会走到物化

决策链（`sql_resolver.cc`）的关键一句是——**"合不了就物化"是默认回退**，不是等价的二选一：

```cpp
    if (tl->is_mergeable() && merge_derived(thd, tl))
      return true;
    if (tl->is_merged()) continue;
    // Prepare remaining derived tables for materialization
    ...
```

**技术可行性** `is_mergeable()`（`Query_expression`）：

```cpp
bool Query_expression::is_mergeable() const {
  if (is_set_operation()) return false;

  Query_block *const select = first_query_block();
  return !select->is_grouped() && select->having_cond() == nullptr &&
         !select->is_distinct() && select->has_tables() &&
         !select->has_limit() && !select->has_windows();
}
```

`Table_ref::is_mergeable()` 另有一条重要限制：

```cpp
  /*
    If the table's content is non-deterministic and the query references it
    multiple times, merging it has the risk of creating different contents.
  */
  Common_table_expr *cte = common_table_expr();
  if (cte != nullptr && cte->references.size() >= 2 &&
      derived->uncacheable & UNCACHEABLE_RAND)
    return false;
```

**"非确定性 + 多引用"禁止 merge**：`RAND()` 每求值一次结果不同，merge 后展开成 N 份会得到不同内容，破坏"同一个 CTE 应一致"的语义。物化一次则天然一致——这是物化**不可替代**的一个场景。

**经验判断** `merge_heuristic()` 的注释给出两条不 merge 的理由：

> A view/derived table is not suggested for merging if it contains subqueries in the SELECT list that depend on columns from itself… we assume they are made derived tables because the user wants them to be materialized, for performance reasons.
>
> Another case is, a query that modifies variables: then try to preserve the original structure of the query.

### 物化的代价估算问题

物化表的行数在物化前是**估算**的，而物化后 optimizer 对临时表也没有真实统计信息——这导致基于物化表的后续操作（join、再排序）代价估算可能明显偏离。这是物化路径的一个固有弱点，也是"能 merge 就 merge"倾向的原因之一。

### 物化划算吗：源码里的显式对算

"物化值不值"在源码里是**显式对算**的，不是拍板。超图优化器（`AddCost()`）：

```cpp
      cost->cost_if_materialized += thd->cost_model()->tmptable_readwrite_cost(
          tmp_table_type, /*write_rows=*/0, /*read_rows=*/num_rows);
      cost->cost_to_materialize +=
          subquery.path->cost +
          kMaterializeOneRowCost * subquery.path->num_output_rows();

      cost->cost_if_not_materialized += num_rows * subquery.path->cost;
```

老优化器的 IN 子查询决策更直观（`compare_costs_of_subquery_strategies()`），trace 字段名可直接对照：

```cpp
  const double subq_executions = calculate_subquery_executions(in_pred, trace);
  const double cost_exists = subq_executions * saved_best_read;
  const double cost_mat_table = sjm.materialization_cost.total_cost();
  const double cost_mat =
      cost_mat_table + subq_executions * sjm.lookup_cost.total_cost();
  const bool mat_chosen = ... (cost_mat < cost_exists) ...;
```

即 **物化划算 ⇔ `C_create + N × C_lookup < N × C_exists`**，其中 N = 预计子查询执行次数（由父查询各层 fanout 连乘，`calculate_subquery_executions()`）。

超图优化器对这条权衡的注释（保留原文拼写错误）：

> Materializiation gives a high up-front cost, but each execution is cheaper, so it will depend on how many times we expect to execute the subquery and how expensive it is to run unmaterialized.

`EstimateMaterializeCost()` 还把成本**显式分成"只付一次"和"每次都付"**——`init_once_cost` 只对 cacheable 的查询块累计（`cost_for_cacheable`）。

**代价常数的精度天花板**——全是 0.1，源码自己都吐槽了：

```cpp
// These are extremely arbitrary cost model constants. We should revise them
// based on actual query times (possibly using linear regression?), and then
// put them into the cost model to make them user-tunable.
constexpr double kApplyOneFilterCost = 0.1;
constexpr double kMaterializeOneRowCost = 0.1;
```

### 物化开始后，索引就加不上了

与"默认无索引"并列的一个坑是：**索引必须在 `instantiate_tmp_table()` 之前建好**，一旦表在引擎里被创建（= 物化开始），键定义即被冻结。`generate_keys()` 开头的注释原话：

```cpp
bool Table_ref::generate_keys() {
  assert(uses_materialization());
  if (!derived_key_list.elements) return false;

  Derived_refs_iterator ref_it(this);
  while (TABLE *t = ref_it.get_next())
    if (t->is_created()) {
      /*
        The table may have been instantiated already, by another query
        block. Consider:
        with qn as (...) select * from qn where a=(select * from qn)
                         union select * from qn where b=3;
        Then the scalar subquery is non-correlated, and cache-able, so the
        optimization phase of the first UNION member evaluates this subquery,
        which instantiates qn, then this phase may want to add an index on 'a'
        (for 'a=') but it's too late. Or the upcoming optimization phase for
        the second UNION member may want to add an index on 'b'.
       */
      return false;
    }
```

机制上有专门为此留的口子：

```cpp
  /**
    true <=> don't actually create table handler when creating the result
    table. This allows range optimizer to add indexes later.
    Used for materialized derived tables/views.
    @see Table_ref::update_derived_keys.
  */
  bool skip_create_table;
```

建索引是三步（四函数）：`Table_ref::update_derived_keys()`（收集候选）→ `JOIN::generate_derived_keys()` / `Table_ref::generate_keys()` / `TABLE::add_tmp_key()`（建）→ `JOIN::finalize_derived_keys()`（删掉没被选中的）。

**后果**：某个引用被判定为常量并在优化期提前求值 ⇒ 物化提前发生 ⇒ **其他 query block 再也不能给它加索引**。这与前面"非确定性 + 多引用禁止 merge"是同一类——**共享带来的便利，要付灵活性的代价**。


---

## 已知限制与坑

| 坑 | 说明 |
|---|---|
| **物化表无统计信息** | 后续代价估算可能严重偏离 |
| **默认无索引** | 除非 `add_derived_key` 自动建键，否则访问退化为全表扫描 |
| **INTERSECT ALL 只支持两个块** | `HalfCounter` 用两个 32 位计数器，源码注释明确"this only works correctly if we only ever have two blocks… so they should not have been merged" |
| **重复计数上限** | INTERSECT ALL 左计数超 2³² 报 `ER_INTERSECT_ALL_MAX_DUPLICATES_EXCEEDED` |
| **重算表不能作 rowid 来源** | `SAFE_IF_SCANNED_ONCE`，影响 DuplicateWeedout / BKA |
| **递归的类型由 anchor 决定** | 可能截断 ⇒ 无限递归，故强制严格模式 |
| **MEMORY 的 deleted record 漏行** | 已由 `FollowTailIterator` 的"不撞 EOF"规避（见前） |
| **保守失效** | 外层只要过一行就重算，哪怕值没变（MySQL 自己的 TODO） |
| **KILL 的响应点** | 物化一次跑完整个子查询，`thd()->killed` 在主循环每轮检查 |

---

## 参数

| 参数 / 开关 | 默认 | 说明 |
|---|---|---|
| `materialization`（`optimizer_switch`） | **ON** | 子查询物化总开关；off ⇒ 强制走 IN→EXISTS |
| `subquery_materialization_cost_based` | **ON** | ON ⇒ 走上面的代价对算；OFF ⇒ 只要能物化就物化（不比较） |
| `derived_merge` | **ON** | derived/view/CTE 是否允许 merge。**关掉即强制物化** |
| `derived_condition_pushdown` | **ON** | 物化 derived 的谓词下推。⚠️ **被引用 ≥2 次的 CTE 与递归 CTE 被排除**——共享物化只有一张临时表，下推会互相干扰 |
| `cte_max_recursion_depth` | **1000** | 递归物化的轮数上限（`FollowTailIterator` 计数） |
| `temptable_max_ram` | 1 GiB（GLOBAL） | 物化载体的**全局**内存预算（所有会话共享） |
| `tmp_table_size` / `max_heap_table_size` | 各 16 MB | 载体降级阈值，详见 [`07_temptable.md`](07_temptable.md) |
| `big_tables` | OFF | ON ⇒ 强制直接落盘，不做内存物化 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Optimizing Derived Tables, View References, and Common Table Expressions*（merge 与 materialize 的行为说明）
- *MySQL 8.0 Reference Manual → MaterializeIterator*（类参考）

**相关文档**
- 物化的载体（内部临时表、TempTable/MEMORY、降级链）见 [`07_temptable.md`](07_temptable.md)
- 优化器侧的 merge / materialize 决策见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)
- 子查询物化（`subselect_hash_sj_engine`、IN2EXISTS）见 [`02_subquery_runtime.md`](02_subquery_runtime.md)
- CTE 与递归（物化的主要使用者）见 [`03_cte.md`](03_cte.md)
- semi-join 的 Materialize 策略见 [`../07_optimize/logical/03_semijoin.md`](../07_optimize/logical/03_semijoin.md)

# 09 执行器：RowIterator 火山模型

> AccessPath 树 → RowIterator 树 → 一行行读出。全部算法级。
>
> 边界：本篇讲**迭代器框架与各类迭代器的算法**；handler 接口语义（`position`/`ref`/`rnd_pos`）见 [`../handler.md`](../handler.md)，InnoDB 侧取行（游标推进）见 [`../../innodb/row_search.md`](../../innodb/row_search.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [零、定位与关键设计](#零定位与关键设计)
- [一、RowIterator 基类与火山契约](#一rowiterator-基类与火山契约)
- [二、迭代器逐个详解](#二迭代器逐个详解)
- [三、CreateIteratorFromAccessPath：显式栈翻译](#三createiteratorfromaccesspath显式栈翻译)
- [四、执行入口 ExecuteIteratorQuery](#四执行入口-executeiteratorquery)
- [五、与 handler / InnoDB 的边界](#五与-handler--innodb-的边界)

---

## 设计思想与理论基础

### 火山模型：来自 Graefe 的 iterator model

`RowIterator` 的 `Init()` / `Read()` 契约，直接来自 Graefe 的经典论文《**Volcano—An Extensible and Parallel Query Evaluation System**》(1990)。这篇论文提出的**迭代器模型（火山模型）**影响了一整代数据库执行器：

- 每个算子实现统一的 `open / next / close` 接口（MySQL 里是 `Init / Read`）
- 数据**自底向上、一次一行**地被父算子"拉"（pull）上来
- 父算子完全不关心子算子内部是排序、扫描还是 join，只要它能 `next()`

这种**统一接口 + 惰性拉取**的设计，让算子可以任意组合——执行计划是一棵树，执行就是递归地 `Read()`。MySQL 8.0 把它落成了 `RowIterator` 抽象类（第一节）。

### 三种执行模型，MySQL 为什么选火山

| 执行模型 | 代表 | 一次处理 | 代价 |
|---|---|---|---|
| **火山（volcano）** | MySQL、PostgreSQL（早期） | 一行 | 每行一次虚函数调用，CPU 开销大 |
| **向量化（vectorized）** | ClickHouse、DuckDB、HeatWave | 一批（列块） | 需要列式存储，SIMD 才有效 |
| **编译执行（codegen）** | PostgreSQL LLVM JIT、Hyper | 一段编译后的代码 | 编译成本高，难调试 |

MySQL 选火山是**被行式存储约束的必然**：

- InnoDB 是行式存储，一行是一个完整 record，**没有列块可供 SIMD 批处理**——向量化无从下手
- 编译执行需要把整个查询编译成机器码，与 MySQL 的"优化器热路径必须快、PS 可复用"理念冲突
- 火山的虚函数开销，对 MySQL 面向的 OLTP 短查询（行数少、latency 敏感）影响小于 OLAP 大查询

所以 8.0 的执行器是**彻底的火山模型**——这也是"社区 MySQL 没有向量执行"这个常见疑问的答案（`Handler` 层、`RowIterator` 层都是逐行）。

### 8.0 从 `Executor` 重构到 `RowIterator` 的意义

5.7 的执行逻辑散落在 `JOIN` 的各个 `exec_*` 方法里，与优化器、与 JOIN 结构强耦合。8.0 重构成独立的 `RowIterator` 体系（`sql/iterators/`），收益：

- **执行计划与执行器解耦**：AccessPath 树（08 篇）能 1:1 翻译成 RowIterator 树
- **EXPLAIN ANALYZE 可观测**：`TimingIterator` 包裹任意算子就能计时
- **算子可独立测试**：每个迭代器是独立类，不再依赖 `JOIN` 全局状态

这是 8.0 里"执行器现代化"的核心一步。

---

## 零、定位与关键设计

**三个反直觉的关键设计**（理解执行器的前提）：

1. **迭代器不拥有行缓冲**。"当前行"隐式存放在各 `TABLE::record[0]` 里，`Read()` 返回的是状态码不是数据。所以 `RowIterator` 基类只有 `m_thd` 一个成员。
2. **读行可能加锁**。`SELECT FOR UPDATE`/`UPDATE`/`DELETE` 下 InnoDB 读到就上锁，所以有 `UnlockRow()` 机制——**谁丢弃行谁负责 unlock**。
3. **排序、物化、聚合都发生在 `Init()` 而非 `Read()`**。`SortingIterator::Init()` 里做完整排序，`MaterializeIterator::Init()` 里物化所有行，`Read()` 只是从结果里取。

---

## 一、RowIterator 基类与火山契约

### 1.1 基类（`sql/iterators/row_iterator.h:82`）

```cpp
class RowIterator {
 public:
  virtual bool Init() = 0;                            // :102
  virtual int Read() = 0;                             // :116
  virtual void SetNullRowFlag(bool is_null_row) = 0;  // :135
  virtual void UnlockRow() = 0;                       // :153
  virtual void StartPSIBatchMode() {}                 // :206
  virtual void EndPSIBatchModeIfStarted() {}          // :217
  virtual RowIterator *real_iterator() { return this; }  // :224
  virtual IteratorProfiler *GetProfiler() { assert(false); return nullptr; }  // :156
 protected:
  THD *thd() const { return m_thd; }
 private:
  THD *const m_thd;                                   // :231
};
```

**用法范式**（类注释 `:72`）：`Init()` 一次，然后 `while (Read() == 0)`。

| 虚函数 | 语义 | 易错点 |
|---|---|---|
| `Init()` | 初始化**或重新初始化**。可重复调用（回绕/重新定位，取决于实现：`SortingIterator` 会**重新排序**，`FollowTailIterator` **不回绕**） | 每次 `Read()` 前必须调过一次 |
| `Read()` | 读一行到 `record[0]` | 返回状态码，不返回数据 |
| `SetNullRowFlag(true)` | 把表缓冲标为"NULL 行"，此后读任何列都得 NULL。**用于 outer join 的 NULL 补全** | `Init()`/`Read()` **不保证**重置它；且**可能在 `Init()` 之前**被调用（外层 EOF 时） |
| `UnlockRow()` | 告诉引擎"上一行没用上"，可释放行锁 | 只对加锁读有意义 |
| `real_iterator()` | 若包装了另一个（如 `TimingIterator`）返回被包装者 | EXPLAIN ANALYZE 时取真实类型必须走它 |

### 1.2 Read() 返回值契约与错误处理

```
 0  OK（一行已就位）
-1  End of records
 1  Error
```

**统一错误处理** `TableRowIterator::HandleError`（`basic_row_iterators.cc:150`）：

```cpp
int TableRowIterator::HandleError(int error) {
  if (thd()->killed) { thd()->send_kill_message(); return 1; }
  if (error == HA_ERR_END_OF_FILE || error == HA_ERR_KEY_NOT_FOUND) {
    m_table->set_no_row();            // ★ "没找到"不是错误，是正常 EOF
    return -1;
  } else {
    PrintError(error);
    return 1;
  }
}
```

所以**所有 TableRowIterator 子类的 `Read()` 都是同一个骨架**：调 `ha_xxx` → 非零就 `return HandleError(err)` → 零就 `++*m_examined_rows; return 0;`。

### 1.3 TableRowIterator（`row_iterator.h:234`）

比基类多：`TABLE *const m_table` + 四个默认实现：

```cpp
void TableRowIterator::UnlockRow() { m_table->file->unlock_row(); }
void TableRowIterator::SetNullRowFlag(bool is_null_row) {
  if (is_null_row) m_table->set_null_row(); else m_table->reset_null_row();
}
void TableRowIterator::StartPSIBatchMode() { m_table->file->start_psi_batch_mode(); }
void TableRowIterator::EndPSIBatchModeIfStarted() {
  m_table->file->end_psi_batch_mode_if_started();
}
```

> ⚠️ **纠错**：`m_expected_rows` **不在** `TableRowIterator` 上。它是每个具体迭代器自己的成员（`basic_row_iterators.h:79/126/472`、`ref_row_iterators.h:66/90`、`index_range_scan.h:102`），**唯一用途**是在 `Init()` 里调 `set_record_buffer()` 分配多行缓冲。

### 1.4 unlock_row 与 record buffer

**为什么需要 unlock**：

1. `Read()` 成功只是把字节填进 `record[0]`，MySQL 层还没判断这行是否真要
2. 加锁读（`FOR UPDATE`/`UPDATE`/`DELETE`）下，InnoDB 的 `row_search_mvcc` **读到就上了锁**
3. READ COMMITTED 及更弱级别下，SQL 标准只要求"返回的行"被保护——**没通过 WHERE 的行其锁可立即释放**

**"谁丢弃行谁 unlock"**：

```cpp
// sql/iterators/composite_iterators.cc:76
int FilterIterator::Read() {
  for (;;) {
    int err = m_source->Read();
    if (err != 0) return err;
    bool matched = m_condition->val_int();
    ...
    if (!matched) { m_source->UnlockRow(); continue; }   // ★
    return 0;
  }
}
```

**三种"故意不 unlock"**（源码注释原文）：

| 迭代器 | 原因 |
|---|---|
| `AggregateIterator` | *"HAVING failed. Ideally we'd backtrack and unlock all rows that went into this aggregate, but we can't"* |
| `ConstIterator` | *"Rows from const tables are read once but potentially used multiple times"* |
| `HashJoinIterator` | *"Since both inputs may have been materialized to disk, we cannot unlock them"* |

**`EQRefIterator` 的单行 cache + 延迟 unlock**（`ref_row_iterators.cc:163`）：

```cpp
int EQRefIterator::Read() {
  if (!m_first_record_since_init) return -1;          // eq_ref 每次 Init 最多一行
  m_first_record_since_init = false;
  bool read_row = !table()->is_started() || table()->file->pushed_cond ||
                  m_ref->disable_cache || m_ref->key_err;
  if (!read_row) memcpy(m_ref->key_buff2, m_ref->key_buff, m_ref->key_length);  // 备份上次 key
  m_ref->key_err = construct_lookup(thd(), table(), m_ref);
  if (!read_row && memcmp(m_ref->key_buff2, m_ref->key_buff, m_ref->key_length) != 0)
    read_row = true;                                   // key 变了 → 必须真读
  if (read_row) {
    if (table()->has_row() && m_ref->use_count == 0)
      table()->file->unlock_row();                     // ★ 上次那行没人用 → 此刻才解锁
    ...
    m_ref->use_count = 1;
  } else if (table()->has_row()) {
    table()->restore_null_flags();                     // 复用 cache：连 NULL 标志也要恢复
    m_ref->use_count++;
  }
  ...
}
```

**算法**：eq_ref 保证每个 key 至多一行。若连续多个外层行给出同一个 key，第二次起**完全跳过引擎调用**，直接复用 `record[0]`。为此维护 `key_buff`/`key_buff2`（本次/上次 key）、`use_count`（被消费次数）、`save_null_flags()`/`restore_null_flags()`。四种情况强制真读：表还没读过一行、有 pushed condition、cache 被禁用、上次构造 key 出错。

`UnlockRow()` 只减 `use_count`（`:246`），真正的 `unlock_row()` 推迟到换 key 时。

### 1.5 PSI batch mode

**做什么**：进入 batch 后**只产生一个 start 事件**，期间所有 table io 不再逐个埋点，只累加 `m_psi_numrows`；退出时一个 end 事件。

**为什么提速**：PFS 埋点本身有计时器开销。N 行全表扫原本 N 次 `start/end_event`，batch 退化成 1 次 + 计数器。对最内层表收益最明显。

**三条规则**（`row_iterator.h:188-204`）：

1. 单子节点迭代器（FilterIterator 等）→ **直接转发**给子节点
2. 自己驱动子节点扫描的（MaterializeIterator / TemptableAggregateIterator / HashJoin build）→ 用 RAII 的 `PFSBatchMode` 包住整个循环
3. 多子节点（join）→ **忽略**转发，`NestedLoopIterator` 只对 inner 开

**哪些能开**由 `ShouldEnableBatchMode()`（`access_path.cc:280`）决定——**只有扫描型单表路径**返回 true：

```cpp
case TABLE_SCAN: case INDEX_SCAN: case REF: case REF_OR_NULL:
case PUSHED_JOIN_REF: case FULL_TEXT_SEARCH: case DYNAMIC_INDEX_RANGE_SCAN:
  return true;
case FILTER:  // 有子查询则 false（子查询内部要单独计时）
  if (path->filter().condition->has_subquery()) return false;
  else return ShouldEnableBatchMode(path->filter().child);
case EQ_REF: case CONST_TABLE:
  // These can read only one row per scan, so batch mode will never be a win
default:
  return false;     // 所有 join 都返回 false
```

---

## 二、迭代器逐个详解

> 所有迭代器通过 `NewIterator<T>()`（`timing_iterator.h:222`）在 MEM_ROOT 上构造；EXPLAIN ANALYZE 时自动包 `TimingIterator<T>`。所有类都是 `final`，便于编译器去虚化。

### 2.1 基础扫描

**TableScanIterator**（`basic_row_iterators.cc:193/217`）

```cpp
bool TableScanIterator::Init() {
  empty_record(table());
  const bool first_init = !table()->file->inited;
  int error = table()->file->ha_rnd_init(true);      // true = 用 rnd_next 顺序扫
  if (first_init && set_record_buffer(table(), m_expected_rows)) return true;
}
int TableScanIterator::Read() {
  if (table()->is_union_or_table()) {                // 普通表/UNION：快路径
    while ((tmp = table()->file->ha_rnd_next(m_record))) {
      if (tmp == HA_ERR_RECORD_DELETED && !thd()->killed) continue;  // MyISAM 并发删
      return HandleError(tmp);
    }
  } else {                                           // EXCEPT/INTERSECT：带计数器
    // 见下
  }
}
```

**集合运算的计数器技巧**：物化阶段对 EXCEPT/INTERSECT **只写一行 + 一个计数器**（`TABLE::m_set_counter`），扫描时再展开：
- `EXCEPT DISTINCT`：cnt ≥ 1 出 1 行，cnt = 0 跳过
- `EXCEPT ALL`：出 cnt 行
- `INTERSECT DISTINCT`：cnt == 0（被减去过）跳过，否则出 1 行
- `INTERSECT ALL`：`HalfCounter`（`basic_row_iterators.h:539`）把 64 位计数器劈成两个 32 位——`[0]`=左侧重复数、`[1]`=右侧已匹配数，出 `min([0],[1])` 行

**IndexScanIterator**（`basic_row_iterators.cc:77/101`）：`m_first` 开关——首次 `ha_index_first()`，之后 `ha_index_next()`。两个优化：覆盖索引时 `set_keyread(true)`；`m_use_order=false` 让分区表用更高效的无序扫描。

**其它零输入迭代器**：

| 迭代器 | 算法 |
|---|---|
| `ConstIterator`（`ref_row_iterators.cc:97`） | 优化期已读出，`read_const()` 查一次后 EOF；`UnlockRow()` 空实现（这行整个查询都要用） |
| `FakeSingleRowIterator` | 出一行后 EOF（计划已判定只有一行且字段已填好） |
| `UnqualifiedCountIterator`（`.cc:846`） | `SELECT COUNT(*)` 无 WHERE 无 join：对所有表 `ha_records()` 求笛卡尔积 |
| `ZeroRowsIterator` / `ZeroRowsAggregatedIterator` | 永远 EOF / 出**一行全 NULL**（`SELECT SUM(x) WHERE 2+2=5`） |
| `TableValueConstructorIterator`（`.cc:920`） | `VALUES ROW(..),ROW(..)`：切换 `Item_values_column` 到下一行 |
| `FullTextSearchIterator`（`ref_row_iterators.cc:648`） | `ft_init()` + `ha_ft_read()`，按相关度取命中 |

### 2.2 ref 族

**RefIterator**（`ref_row_iterators.cc:358`）

```cpp
if (m_first_record_since_init) {
  m_first_record_since_init = false;
  if (m_ref->impossible_null_ref()) { table()->set_no_row(); return -1; }  // ★ Late NULLs Filtering
  construct_lookup(thd(), table(), m_ref);           // 从外层列构造 key
  error = table()->file->ha_index_read_map(..., HA_READ_KEY_EXACT);
} else {
  do { error = table()->file->ha_index_next_same(...); }
  while (error == HA_ERR_KEY_NOT_FOUND && m_is_mvi_unique_filter_enabled);
}
```

- **首次读**：构造 key → `ha_index_read_map(HA_READ_KEY_EXACT)` 精确定位第一条
- **后续读**：`ha_index_next_same()` 逐条返回同 key 的下一条，key 变化返回 `KEY_NOT_FOUND` → EOF
- **Late NULLs Filtering**：key 里出现 SQL NULL 时直接 -1，省一次引擎调用
- MVI（多值索引）的 `do-while`：`HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER` 过滤同一记录被多次命中

**EQRefIterator**：见 1.4（单行 cache）。

**RefOrNullIterator**（`ref_row_iterators.cc:712`）—— `=ref OR IS NULL` 的**两阶段扫描**：

```cpp
if (error == HA_ERR_END_OF_FILE || error == HA_ERR_KEY_NOT_FOUND) {
  if (!*m_ref->null_ref_key) {
    *m_ref->null_ref_key = true;      // 非 NULL 段扫完 → 拨开关
    m_reading_first_row = true;
    return Read();                    // ★ 递归：重来一遍扫 NULL 段
  } else { table()->set_no_row(); return -1; }
}
```

`null_ref_key` 是写进 key buffer 的一个"开关字节"：false 匹配非 NULL 值，true 匹配 NULL。

**DynamicRangeIterator**（`ref_row_iterators.cc:491`）—— EXPLAIN 里的 "Range checked for each record"：

```cpp
bool DynamicRangeIterator::Init() {
  m_mem_root.ClearForReuse();        // 上次的 QUICK 结构整体销毁
  int rc = test_quick_select(thd(), &m_mem_root, &m_mem_root, ...);   // ★ 每次 Init 重跑
  if (range_scan == nullptr) { m_qep_tab->set_type(JT_ALL); }
  else { qck = CreateIteratorFromAccessPath(thd(), &m_mem_root, range_scan, ...); }
  ...
}
```

**每次 `Init()` 都重跑 range 优化器**——因为可用的 const 值/已读表随外层行变化。

**PushedJoinRefIterator**（`.cc:276`）：NDB pushed join 专用。父表行与子表列一起 prefetch，所以 `ha_index_read_pushed()` **不产生网络往返**，只解包。

**AlternativeIterator**（`.cc:784`）：两个候选（ref / 全表扫），**每次 Init 动态二选一**：

```cpp
m_iterator = m_source_iterator.get();
for (bool *cond_guard : m_applicable_cond_guards)
  if (!*cond_guard) { m_iterator = m_table_scan_iterator.get(); break; }
if (m_iterator != m_last_iterator_inited) {
  m_table->file->ha_index_or_rnd_end();     // ★ 切换前必须重置 handler 模式
  m_last_iterator_inited = m_iterator;
}
```

`cond_guard` 是 `NULL IN (...)` 下推产生的标志：出现 NULL 时 ref 不可用，退化全表扫。

### 2.3 index merge 扫描族（RowIDIntersection / RowIDUnion / IndexMerge）

> 对应 [08_range_optimizer.md](07_optimize/physical/08_range_optimizer.md) 4.4/4.5/4.6 的三种 AccessPath（`ROWID_INTERSECTION` / `ROWID_UNION` / `INDEX_MERGE`），在 `access_path.cc:560/602/630` 由 `NewIterator<>` 创建。三者都继承 `TableRowIterator`；其中 `RowIDIntersectionIterator` 额外继承 `RowIDCapableRowIterator`（需对外暴露 `last_rowid()` 供 union 比较）。

**继承链**：

```
RowIterator → TableRowIterator
  ├─ RowIDCapableRowIterator（rowid_capable_row_iterator.h:36，纯虚 last_rowid()）
  │    ├─ IndexRangeScanIterator（叶子）
  │    └─ RowIDIntersectionIterator
  ├─ RowIDUnionIterator
  └─ IndexMergeIterator
```

#### RowIDIntersectionIterator：多路归并取交集（`rowid_ordered_retrieval.cc:322`）

成员 `m_children`（非 CPK 的 range scan）、`m_cpk_child`（聚簇 PK 扫描，可空，**仅过滤不取行**）、`retrieve_full_rows`（取交集后是否回表）、`m_last_rowid`（候选 rowid 缓冲）。

`Read()` 状态机：**确立候选 → 轮转各子扫描"追赶" → 全部确认后输出**：

```cpp
// 1. 确立候选：读第一个 child 一行，取 file->ref 到 m_last_rowid
//    若 m_cpk_child：row_in_ranges() 过滤 CPK 会命中的行
// 2. 内层循环：轮转其余 child，cmp = cmp_ref(child_rowid, m_last_rowid)
//    cmp < 0 → child->UnlockRow() 后继续 Read（追赶）
//    cmp > 0 → 废弃旧候选，该 child 新 rowid 立为新候选，last_rowid_count=1
//    cmp == 0 → last_rowid_count++（该分支确认）
// 3. last_rowid_count == children.size() → 所有分支确认同一 rowid
//    retrieve_full_rows ? ha_rnd_pos 回表 : 直接 return 0
```

关键点：

- **锁语义**：`child_with_last_rowid` 记录"锁住候选行"的 child，被淘汰的行用正确的 child 对象解锁，避免锁泄漏（头注释 `:312` 强调必须用当初锁它的 handler 解锁）。
- **`retrieve_full_rows = !is_covering`**（`rowid_ordered_retrieval_plan.cc:994`）：优化器判定已覆盖则执行期不回表；若本 intersection 是 union 的孩子，union 会把它置 false，由 union 统一回表（避免同一 rowid 回表两次）。
- 各子扫描 `need_rows_in_rowid_order=true` → `HA_MRR_SORTED` + `HA_EXTRA_SECONDARY_SORT_ROWID`，引擎按 rowid 顺序输出，`file->position()` 回填 `file->ref`。ROR 子扫描 `mrr_buf_size=0`（`rowid_ordered_retrieval_plan.cc:690`），排序由引擎内部完成，不占应用层 `HANDLER_BUFFER`。

#### RowIDUnionIterator：优先级队列 k 路归并去重（`rowid_ordered_retrieval.cc:433`）

成员 `Priority_queue<RowIterator*, vector, Quick_ror_union_less> queue`（大顶堆）、`cur_rowid`/`prev_rowid`（连续分配 2×`ref_length`）。

```cpp
// Init：每个 child Init + Read 第一行，有行则 queue.push
// Read：do { 取 queue.top() 最小 rowid 到 cur_rowid；
//           推进该流（下一行 update_top / EOF pop）；
//           dup_row = !cmp_ref(cur_rowid, prev_rowid) } while (dup_row);  // 相邻去重
//       swap(cur, prev); ha_rnd_pos 回表
```

- **比较器 `Quick_ror_union_less`**（`rowid_ordered_retrieval.h:145`）：`cmp_ref(a.last_rowid, b.last_rowid) > 0`，大顶堆 → 堆顶是 rowid 最小者。
- **相邻去重即全局去重**：所有流按 rowid 排序，同一 rowid 的重复必然连续。优化器侧 `roru_total_records -= table_rows * PROD(rows_i/table_rows)` 估的重复量，运行时由 `prev_rowid` 精确剔除。
- 堆归并每次 `update_top`/`push` 约 `log2(n)` 次 `cmp_ref`——正是优化器 `key_compare_cost(total * log2(n))` 估的那部分。

#### IndexMergeIterator：两阶段 Unique 去重（`index_merge.cc:103/220`）

成员 `unique`（`Unique`，树+文件归并）、`read_record`（读 Unique 结果）、`pk_quick_select`（CPK 扫描）。

**Phase 1（`Init()`）**：`set_keyread(true)` → 对每个非 CPK child 临时 `covering_keys.set_bit`（**强制 index-only**，即使该索引本不覆盖全部列）→ `child->Read()` 循环，`row_in_ranges()` 过滤 CPK 会取到的行 → `child_file->position()` 取 rowid → `unique->unique_add()` 去重插入 → `unique->get(table)` 归并成有序结果。

**Phase 2（`Read()`）**：先顺序读 `read_record`（读 rowid → `ha_rnd_pos` 回表）；EOF 后切 `pk_quick_select` 的 CPK 扫描。两阶段输出不相交（Phase 1 已用 `row_in_ranges()` 剔除 CPK 会取到的行）。

> ⚠️ 纠错：8.0.39 的 `IndexMergeIterator` **没有** `m_dedup` / `m_non_cpk_scan_records` 成员（后者是优化器侧 `range_optimizer.cc:1046` 的局部变量），也**没有**独立的 `prepare_unique` 方法（Phase 1 已内联进 `Init()`）；去重**始终**走 `Unique`（8.0 去掉了老版本"不去重"的 sort-union 变体）。

#### 与优化器代价估算的对应

| 优化器代价项（08 篇） | 执行期步骤 |
|---|---|
| `get_sweep_read_cost`（回表） | 三者 `Read()` 里的 `ha_rnd_pos` |
| `key_compare_cost(total*log2(n))`（堆归并） | `RowIDUnionIterator` 的 `Priority_queue` |
| `Unique::get_use_cost`（去重） | `IndexMergeIterator::Init()` 的 `unique_add` + `unique->get` |

### 2.4 过滤与限制

**FilterIterator**：见 1.4（教科书式 σ 算子）。

**LimitOffsetIterator**（`composite_iterators.cc:115`）—— 两个巧妙设计：

```cpp
bool LimitOffsetIterator::Init() {
  if (m_source->Init()) return true;
  if (m_offset > 0) { m_seen_rows = m_limit; m_needs_offset = true; }  // ★ 故意设成 m_limit
  else              { m_seen_rows = 0;        m_needs_offset = false; }
}
```

① 有 OFFSET 时把 `m_seen_rows` 初始化成 `m_limit`，这样 `Read()` 只需一次比较 `m_seen_rows >= m_limit` 就同时覆盖"还有 OFFSET 要跳"和"LIMIT 已满"。

② **OFFSET 的跳过放在 `Read()` 而非 `Init()`**，注释给了两个理由：让被跳过的行也享受 PSI batch mode 收益；避免在 `Init()` 里意外触发 `NestedLoopIterator` 打开 batch mode（此时 executor 还没准备好在出错时关掉它）。

`count_all_rows`（`SQL_CALC_FOUND_ROWS`）时**继续读到底**只为计数，无任何提前终止收益。

### 2.5 连接

**MySQL 8.0 的 join 方式全集**（先有个横向地图，再逐个剖析）：

| join 方式 | 8.0 有没有 | 迭代器 | 复杂度 | 文档位置 |
|-----------|-----------|--------|--------|---------|
| **Nested Loop（NLJ）** | ✅ | `NestedLoopIterator` | O(outer × inner_lookup) | 本节 |
| **BNL**（Block Nested Loop） | ⚠️ 只剩优化器标记 | （改写为 hash join） | O(N+M) | [runtime/05](runtime/05_join_buffer.md) |
| **BKA**（Batched Key Access） | ✅ | `BKAIterator` + MRR | 批量回表，减少随机 IO | 本节 + runtime/05 |
| **Hash Join** | ✅（8.0.18+） | `HashJoinIterator` | O(N+M)（内存内） | 本节 + runtime/05 |
| **Merge Join（sort-merge）** | ❌ **没有** | 无 | — | — |
| semi-join / anti-join | ✅ | `NestedLoopIterator(JoinType::SEMI/ANTI)` 等 | 存在性语义 | 03 篇 + runtime/02 |

> ⚠️ **关键事实：MySQL 8.0 没有 merge join（sort-merge join）**。全网搜源码，"merge join" 只在 `join_optimizer/interesting_orders.cc:823` 的**注释**里以虚拟语气出现（"if we were to use it for merge joins somehow"——"如果我们某天要做 merge join"）。所以别在 MySQL 里找 merge join 的实现，它不存在。join 方式就是 **NLJ 家族 + Hash Join** 两类。

**NLJ 家族的三个变体**（都是"外层每行驱动内层"的循环，差别在内层怎么读）：

| 变体 | 内层读取方式 | 解决什么问题 |
|------|-------------|-------------|
| 普通 NLJ | 每外层行一次 `Init()+Read()` 循环 | 基准（内层有索引时 lookup 便宜） |
| BNL | 外层批量入 buffer，内层扫一遍 | 内层全表扫 O(N×M) → O(N+M)，**但 8.0 执行器已把它改写为 hash join** |
| BKA | 外层批量收集 key → MRR 按 rowid 排序回表 | 批量随机 IO → 顺序 IO（`mrr_batch_size` 控制批大小） |

**Hash Join 与 NLJ 的选择**（优化器侧）：hash join 适合**无索引的等值连接**（内层没索引时 NLJ 要全扫，hash join 建一次表只扫一遍）；有索引时 NLJ 的逐行 lookup 通常更便宜。优化器在 `setup_join_buffering`（`sql_optimizer.cc:3616`）决策，`OPTIMIZER_SWITCH_HASH_JOIN`（默认 ON，8.0.18+）控制。

**NestedLoopIterator**（`composite_iterators.cc:466`）—— **重点：NULL 补全**

```
状态机：NEEDS_OUTER_ROW → READING_FIRST_INNER_ROW → READING_INNER_ROWS
              ↑                    |（内层第一条就 EOF 且 OUTER）     |
              └────────────────────┴──────────────────────────────────┘
```

```cpp
if (m_state == NEEDS_OUTER_ROW) {
  int err = m_source_outer->Read();
  if (err == -1) { m_state = END_OF_ROWS; return -1; }
  if (m_pfs_batch_mode) m_source_inner->StartPSIBatchMode();
  m_source_inner->SetNullRowFlag(false);      // ★ 必须在 Init() 之前
  if (m_source_inner->Init()) return 1;
  m_state = READING_FIRST_INNER_ROW;
}
int err = m_source_inner->Read();
if (err == -1) {
  if ((m_join_type == JoinType::OUTER && m_state == READING_FIRST_INNER_ROW) ||
      m_join_type == JoinType::ANTI) {
    m_source_inner->SetNullRowFlag(true);     // ★ NULL 补全
    m_state = NEEDS_OUTER_ROW;
    return 0;
  } else { m_state = NEEDS_OUTER_ROW; continue; }
}
if (m_join_type == JoinType::ANTI) { m_state = NEEDS_OUTER_ROW; continue; }
if (m_join_type == JoinType::SEMI) m_state = NEEDS_OUTER_ROW;   // 只出第一条
else                               m_state = READING_INNER_ROWS;
return 0;
```

**算法要点**：
- **NULL 补全的判定靠 `m_state == READING_FIRST_INNER_ROW`**——即"**第一条**内层 `Read()` 就 EOF"。若已出过至少一条后再 EOF，说明匹配过，不补 NULL。**这就是状态机必须区分 FIRST / 非 FIRST 的原因**。
- `SetNullRowFlag(false)` 在 `Init()` **之前**：内层若是 HashJoin，其 `Init()`（build）会读外层列，残留的 NULL 标志会建出错误的 hash 表。
- ANTI 是"命中即丢弃换外层行"，SEMI 是"命中即返回第一条"。

**HashJoinIterator**（`hash_join_iterator.cc`）—— 三种形态：

| 形态 | 触发 |
|---|---|
| `IN_MEMORY` | build 侧能全装进 `join_buffer_size` |
| `SPILL_TO_DISK` | 装不下 → 两侧按同一 hash 分区写 chunk，逐对 chunk 做内存 hash join |
| `IN_MEMORY_WITH_HASH_TABLE_REFILL` | 不允许落盘（有 LIMIT）→ 反复"填满 → 扫完 probe → 再填" |

**build 阶段**（`:410`）：

```cpp
const bool reject_duplicate_keys = RejectDuplicateKeys();   // semi/anti 可丢重复 key
PFSBatchMode batch_mode(m_build_input.get());
for (;;) {
  int res = m_build_input->Read();
  if (res == -1) { m_build_iterator_has_more_rows = false; ... return false; }
  RequestRowId(...);
  switch (m_row_buffer.StoreRow(thd(), reject_duplicate_keys)) {
    case ROW_STORED: break;
    case BUFFER_FULL:
      if (!m_allow_spill_to_disk) { /* 转 refill 模式 */ return false; }
      InitializeChunkFiles(...);
      WriteRowsToChunks(...);       // build 剩余行按 hash 分区写盘
      ...
    case FATAL_ERROR: my_error(ER_OUTOFMEMORY, ...); return true;
  }
}
```

**chunk 数估算**（`:377`）：

```cpp
constexpr double kReductionFactor = 0.9;
const double reduced_rows_in_hash_table = std::max(1.0, rows_in_hash_table * 0.9);
const size_t remaining_rows = std::max(rows_in_hash_table, estimated_rows_produced) - rows_in_hash_table;
const size_t chunks_needed = std::max<size_t>(1, std::ceil(remaining_rows / reduced_rows_in_hash_table));
const size_t num_chunks = std::min(max_chunk_files, chunks_needed);
const size_t num_chunks_pow_2 = my_round_up_to_next_power(num_chunks);   // 取 2 的幂（位与代替取模）
```

即"用当前 hash 表能装的行数（打 9 折留余量）估还需要多少 chunk"，上限 `kMaxChunks=128`（防 fd 耗尽）。写 chunk 用**不同的 hash 种子**（`MY_XXH64(key, seed=899339)`），否则重新装载 chunk 时 hash 表会退化。

**probe 阶段**（`:848/944`）：

```cpp
bool null_in_join_key = ConstructJoinKey(...);       // 拼 N 个等值条件的 key
if (null_in_join_key) {
  if (m_join_type == ANTI || m_join_type == OUTER) { /* 走 NULL 补全 */ }
  else { SetReadingProbeRowState(); return; }        // inner/semi 直接跳过
}
m_current_row = m_row_buffer.find(key).value_or(nullptr);   // hash 查找
```

- hash 表的值类型是 `LinkedImmutableString`——同 key 的行串成**链表**
- **extra conditions**（非等值条件）在 hash 查找**之后**、返回**之前**求值；不过就前进到下一个匹配。这也是"有 extra condition 时 build 期不能丢重复 key"的原因
- **落盘时 probe 行也要写盘**：inner/outer 全部写；semi/anti **只写未匹配的**（否则同一 probe 行输出两次）
- **落盘时的 NULL 补全限制**：`on_disk_hash_join() && m_current_chunk == -1`（还没开始读 chunk）时本轮没匹配不代表最终没匹配 → 不补 NULL

**主循环**（`:1046`）是 6 个状态的分派（`LOADING_NEXT_CHUNK_PAIR` / `READING_ROW_FROM_PROBE_ITERATOR` / `READING_ROW_FROM_PROBE_CHUNK_FILE` / `READING_ROW_FROM_PROBE_ROW_SAVING_FILE` / `READING_*_FROM_HASH_TABLE` / `END_OF_ROWS`）。

**BKAIterator + MultiRangeRowIterator**（Batched Key Access）

外层缓存一批行 → 用 MRR 一次性把这批 key 交给引擎 → 引擎按 key 排序后顺序回表。

**MRR 的握手协议**（`bka_iterator.cc:335`）：不给引擎数组，而是给**三个函数指针 + this**：

```cpp
RANGE_SEQ_IF seq_funcs = {MrrInitCallbackThunk, MrrNextCallbackThunk, nullptr};
if (m_join_type == SEMI || m_join_type == ANTI) seq_funcs.skip_record = MrrSkipRecordCallbackThunk;
return m_file->multi_range_read_init(&seq_funcs, this, ..., m_mrr_flags, &m_mrr_buffer);
```

`MrrNextCallback`（`:381`）每次构造一个 range，并把"是哪个外层行"塞进 `range->ptr` 作 cookie：

```cpp
range->ptr = const_cast<char *>(pointer_cast<const char *>(m_current_pos));   // ★ cookie
range->start_key.flag = HA_READ_KEY_EXACT;
range->end_key.flag   = HA_READ_AFTER_KEY;      // 单 key 的闭开区间
```

`Read()`（`:431`）时引擎把 cookie 原样返回，由此知道"这条内层行对应哪个外层行"再 unpack 回外层表 buffer，完成 join。

**NestedLoopSemiJoinWithDuplicateRemovalIterator**（`.cc:2145`）—— loose scan 的"跳过"语义：取一个外层行 → 取**恰好一条**内层行 → 命中后设 `m_deduplicate_against_previous_row`，之后**不扫内层**地跳过所有同 key 外层行；未命中则不设标志。

### 2.6 去重

| 迭代器 | 算法 |
|---|---|
| `WeedoutIterator`（`.cc:2008`） | semijoin 的"inner join + 事后去重"。去重键是**外层表的 row ID**（不是行内容）：`position()` 取 row ID → 临时表查重。流式边产边丢 |
| `RemoveDuplicatesIterator`（`.cc:2056`） | 通用 loose scan：`Cached_item::cmp()` 缓存上一行的 group 值比较。**只在输入已有序时正确** |
| `RemoveDuplicatesOnIndexIterator`（`.cc:2098`） | 同上但比较**索引 keypart 原始字节**（`key_copy`+`key_cmp`），比求值快。仅旧优化器用 |

> confluent weedout（对所有字段去重）在**优化期已被改写成 LIMIT 1**（`sql_executor.cc:1216`）。

### 2.7 排序

**`SortingIterator`：整个排序发生在 `Init()`**（`sorting_iterator.cc:438`）。之后按 2×2 组合选结果迭代器：

|  | addon fields（行数据随排序结果存） | 无 addon（只存 `handler::ref`） |
|---|---|---|
| **内存** | `SortBufferIterator<Packed>` | `SortBufferIndirectIterator` |
| **文件** | `SortFileIterator<Packed>` | `SortFileIndirectIterator` |

- **addon fields**：排序记录里直接带 SELECT 需要的列，**读结果不用回表**
- **indirect**：只存 `handler::ref`（行位置），读结果时 `ha_rnd_pos()` **回表**（8.0.20 后仅 UPDATE/DELETE 两阶段读、FTS、大 BLOB 用）

> 两种载体的本质（addon = 存列值 vs rowid = 存 `handler::ref`）与 8.0.20 演进见 [`runtime/01_filesort_and_temptable.md`](runtime/01_filesort_and_temptable.md) 1.3 节。

六种结果迭代器通过 `IteratorHolder` union（`sorting_iterator.h:139`）**共享同一块存储**，避免堆分配。

`SortBufferIndirectIterator::Read()`（`:362`）每条记录前缀可能有 1 字节 NULL 标志（标识"这行是 NULL 补全行"），之后是各表 row ID，逐表 `ha_rnd_pos()` 回表。

**溢出判定**：由 `filesort()` 决定——`m_sort_result.io_cache` 被真正初始化就说明落盘了。`m_num_rows_estimate` 只用于**是否启用优先队列**（Top-N 堆排序）。

### 2.8 物化与聚合

**MaterializeIterator**（`composite_iterators.cc:824`）—— `Init()` 是全部工作：

1. **建表**（首次）：视图/派生表走 `create_materialized_table()`，JOIN 内部临时表走 `instantiate_tmp_table()`
2. **共享 CTE 复用**：`m_cte->tmp_tables` 中任一张已物化 → 本次不再物化
3. **重扫 vs 重物化**：`m_rematerialize`（依赖外层值）+ `invalidators`（LATERAL：`CacheInvalidatorIterator` 的 generation 变化）
4. **去重**：`doing_hash_deduplication()`（表上有 `hash_field`，B-tree 长度受限时用 hash 列手工比行）；否则依赖唯一索引，`ha_write_row()` 返回**可忽略错误即视为重复**
5. **物化每个 query block**：普通 UNION 走 `MaterializeQueryBlock` 循环；递归 CTE 走 `MaterializeRecursive`

`MaterializeQueryBlock`（`:1122`）主循环：`copy_funcs()` 求值 → `ha_write_row()`；若失败且是"内存临时表满"（`HA_ERR_RECORD_FILE_FULL`）→ `create_ondisk_from_heap()` **转 InnoDB 临时表**并重试（这一转换必须通知所有 `FollowTailIterator` 重新定位游标——递归 CTE 正在同时读同一张表）。

**递归 CTE**（`:1056`）—— 不做标准要求的"每轮重建结果集"，而是**边写边读同一个临时表**：

```cpp
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (qb.is_recursive_reference)
    qb.recursive_reader->set_stored_rows_pointer(&stored_rows);   // ★ 共享行计数
// 1) 先物化所有非递归 block
// 2) 反复物化递归 block 直到收敛
do {
  last_stored_rows = stored_rows;
  for (...) if (qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);
} while (stored_rows > last_stored_rows);     // ★ 收敛条件：本轮没新增行
```

**StreamingIterator**（`.cc:1551`）："不落盘的 MaterializeIterator"——优化器以为会物化（read_set、上游 Item 都按临时表布局设好了），实际只需单趟顺序扫，于是换成纯转发 + `copy_funcs`。`provide_rowid` 时用自增行号伪造 `handler::ref`。

**TemptableAggregateIterator**（`.cc:1725`）—— **输入无需有序的 hash 聚合**：每行先 `ha_index_read_map(HA_READ_KEY_EXACT)` 找组，命中则 `update_tmptable_sum_func()` 累加并 `ha_update_row()`，未命中则 `ha_write_row()` 插新组。

**AggregateIterator**（`.cc:292`）—— **输入已有序的流式聚合**。核心难点：**只有读到下一组的第一行时才知道本组结束**，所以需要两个缓冲：

```
m_first_row_this_group   —— 本组首行（输出时用它的分组列值）
m_first_row_next_group   —— 下一组首行（暂存，下次 Read() 处理）
```

用 `pack_rows::StoreFromTableBuffers`/`LoadIntoTableBuffers` 序列化/反序列化整行，实现"回滚"表缓冲。用 `swap()` 而非 `move()` 以复用已分配的 buffer。

ROLLUP：`first_changed_idx` = 变了的第一个分组列，`m_last_unchanged_group_item_idx = first_changed_idx + 1`；`SetRollupLevel()`（`:443`）遍历 `rollup_group_items`/`rollup_sums` 设置当前层级，让它们把自己变成 NULL 或切换输出哪个 sum。

**零输入行但有聚合**（`SELECT SUM(x) FROM t`）：`READING_FIRST_ROW` 遇 EOF 时调 `item->no_rows_in_result()`，输出一行 NULL。

### 2.9 窗口函数

**WindowIterator**（非缓冲，`:1535`）—— 一行三阶段 `copy_funcs`：

```cpp
SwitchSlice(m_join, m_input_slice);
int err = m_source->Read();
SwitchSlice(m_join, m_output_slice);
if (copy_funcs(m_temp_table_param, thd(), CFT_HAS_NO_WF)) return 1;   // 1. 非 WF 函数
m_window->check_partition_boundary();                                  // 2. 判定分区边界（唯一职责）
if (copy_funcs(m_temp_table_param, thd(), CFT_WF)) return 1;           // 3. 本窗口的 WF
```

**BufferingWindowIterator**（缓冲，`:1586`）—— 有需要 frame 的 WF 时用。**IN / OUT / FB 三张表**：

- **IN** = 输入（父迭代器，已按 PARTITION BY + ORDER BY 排序）
- **OUT** = 输出临时表
- **FB** = frame buffer（`Window::m_frame_buffer`）

流程：`copy_fields` + `copy_funcs(CFT_HAS_NO_WF)` → `buffer_windowing_record()` 把行拷进 FB（`ha_write_row`）并记录"分区第一行"位置供就近定位 → `process_buffered_windowing_record()`（`:654`）根据 `static_aggregate` / `optimizable_row_aggregates`（逆函数滑动）/ `optimizable_range_aggregates` **三条互斥路径**求值。

**一次输入行的读入可能产出 0、1 或多行**——这就是 `m_possibly_buffered_rows` 循环的原因。

分区切换时把该行暂存为 `FBC_FIRST_IN_NEXT_PARTITION`（内存里），本轮先算完旧分区，下一轮 `bring_back_frame_row()`（`:404`）恢复它再真正缓冲。

### 2.10 修改类

**UpdateRowsIterator / DeleteRowsIterator**（`sql_update.cc:2864` / `sql_delete.cc:1186`）—— **两阶段更新**（这正是 [10_dml.md](10_dml.md) "buffer row id 两阶段读"的驱动方）：

1. 扫 join 结果：能"立即更新"的表直接改；不能立即更新的表只把**行 ID（主键）**写进临时文件/缓冲（每表一个 `Unique`，去重 + 可溢出到磁盘）
2. join 扫完后 `DoDelayedUpdates()`/`DoDelayedDeletes()`：对每个缓冲的 row ID 调 `ha_rnd_pos()` 回表再修改
3. 即使出错，只要**不能安全回滚**（改过非事务表）也要把 delayed 更新做完

注意 `Read()` 返回 **-1**——它把整个 join 结果在一次 `Read()` 里消费完。

### 2.11 其它

**TimingIterator**（`timing_iterator.h:165`）—— EXPLAIN ANALYZE 的计时器：

```cpp
void StopInit(TimeStamp start_time) {
  m_elapsed_first_row += Now() - start_time;
  m_num_init_calls++;          // = EXPLAIN ANALYZE 的 loops
  m_first_row = true;
}
void StopRead(TimeStamp start_time, bool read_ok) {
  if (m_first_row) { m_elapsed_first_row += Now() - start_time; m_first_row = false; }
  else             { m_elapsed_other_rows += Now() - start_time; }
  if (read_ok) m_num_rows++;
}
```

`first_row` / `other_rows` 正好对应 EXPLAIN ANALYZE 的 `actual time=first..last` 两个数字。Linux 上绕过慢的 libstdc++ `steady_clock`，直接用 `clock_gettime(CLOCK_MONOTONIC)`。

> **注意**：`MaterializeIterator` / `TemptableAggregateIterator` **不用** TimingIterator 包（`timing_iterator.h:132-156` 长注释）：EXPLAIN 树里 table iterator 在**上**、Materialize 在**下**，而时间要"累积"，所以这两个类内置 profiler 并用 `SetOverrideProfiler()` 让上层显示"物化 + 扫描"总时间。

**FollowTailIterator**（`basic_row_iterators.cc:358`）—— 递归 CTE 的"边写边读"：

```cpp
int FollowTailIterator::Read() {
  if (m_read_rows == *m_stored_rows) {
    /*
      不真的去读（不让自己撞上 EOF）。两个原因：
       1. MEMORY/InnoDB 一旦报 EOF，即使后面插入了新行，扫描也永远停在 EOF
       2. MEMORY 特有问题：写入被去重丢弃的行会留下 deleted record，
          游标撞上 EOF 后再插入的行可能落在已跳过的空洞里，导致漏行
       解决：用指向 MaterializeIterator 的 m_stored_rows 指针数行数，永不撞 EOF。
     */
    return -1;
  }
  if (m_read_rows == m_end_of_current_iteration) {
    if (++m_recursive_iteration_count > thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;
    }
    m_end_of_current_iteration = *m_stored_rows;
  }
  int err = table()->file->ha_rnd_next(m_record);
  ...
}
```

**`Init()` 不回绕游标**——这是与 `TableScanIterator` 的根本差别。

**AppendIterator**（`.cc:2238`）：UNION ALL 顺序拼接。当前 EOF → 关它的 batch mode → 初始化下一个 → 递归。`StartPSIBatchMode()` 只转发**当前**那个，`EndPSIBatchModeIfStarted()` 转发**全部**。

**MaterializeInformationSchemaTableIterator**（`.cc:2209`）：I_S 表查询时才填充——`Init()` 调 `do_fill_information_schema_table()` 灌数据（带 condition 裁剪），再转给表扫描迭代器。

---

## 三、CreateIteratorFromAccessPath：显式栈翻译

`sql/join_optimizer/access_path.cc:379`。

### 3.1 为什么用显式栈而不是递归

源码注释给的理由：

> *"The access path trees can be pretty deep, and the stack frames can be big on certain compilers/setups, so instead of explicit recursion, we push jobs onto a MEM_ROOT-backed stack."*

三个要点：

1. **栈帧太大**：AccessPath 树可以很深（复杂 join / 多层物化），递归有爆栈风险
2. **用 MEM_ROOT 内存换栈空间**：`todo` 是 `Mem_root_array`，`children` 数组也直接分配在 MEM_ROOT 上——**即使 todo 扩容，`job.children` 地址也不变**（子迭代器的 `destination` 指针正指向它们）
3. **两阶段模式**：第一次弹到某 job 时 `children.is_null()` → 分配 children、压回自己、压入所有子 job；等子 job 完成（它们的 `destination` 就是 `job.children[i]`）后再次弹到自己 → 真正构造

```cpp
struct IteratorToBeCreated {
  AccessPath *path;
  JOIN *join;
  bool eligible_for_batch_mode;
  unique_ptr_destroy_only<RowIterator> *destination;    // 结果写到哪里
  Bounds_checked_array<unique_ptr_destroy_only<RowIterator>> children;
};
...
while (!todo.empty()) {
  IteratorToBeCreated job = todo.back();
  todo.pop_back();
  ...
  switch (path->type) { ... }
  path->iterator = iterator.get();          // :1192 反向指针
  *job.destination = std::move(iterator);   // :1193 move 到父节点持有的槽位
}
```

**创建顺序**：`SetupJobsForChildren` 把 **inner 先压、outer 后压**——栈是 LIFO，出栈顺序是 outer → inner，保证 "left before right"（物化路径的 invalidators 依赖此顺序）。

### 3.2 关键 switch 分支

| 类型 | 行号 | 要点 |
|---|---|---|
| `TABLE_SCAN` / `INDEX_SCAN` / `REF` | :423 / :429 / :442 | 按 `reverse` 选模板参数 |
| `EQ_REF` | :462 | **不传 expected_rows**（至多一行，无需 record buffer） |
| `INDEX_RANGE_SCAN` | :504 | geometry / reverse / 正序三种 |
| `INDEX_MERGE` / `ROWID_INTERSECTION` / `ROWID_UNION` | :528 / :565 / :608 | 多子节点循环压栈 |
| `NESTED_LOOP_JOIN` | :703 | `SetupJobsForChildren(outer, inner)` → `NestedLoopIterator(children[0], children[1], join_type, pfs_batch_mode)` |
| `BKA_JOIN` | :728 | **特殊**：先在压子 job **之前**设置 `mrr_path->mrr().bka_path = path`，再从 `mrr_path->iterator->real_iterator()` 拿 `MultiRangeRowIterator*` |
| `HASH_JOIN` | :751 | **build = inner（右）、probe = outer（左）** |
| `FILTER` | :832 | 先 `FinalizeMaterializedSubqueries()` 再建 `FilterIterator` |
| `SORT` | :846 | 创建后把 `SortingIterator*` 回填到 `filesort->tables[0]->sorting_iterator` |
| `TEMPTABLE_AGGREGATE` | :886 | **两个子节点**：`children[0]=subquery`（batch mode=true）、`children[1]=table_path` |
| `MATERIALIZE` | :943 | 见 3.4 |
| `WINDOW` | :1065 | 按 `needs_buffering` 选缓冲/非缓冲 |
| `DELETE_ROWS` / `UPDATE_ROWS` | :1153 / :1170 | **特殊**：压子 job **之前**先调 `SetUpTablesForDelete`/`FinalizeOptimizationForUpdate`（子迭代器构造需要看到最终 read set） |

### 3.3 迭代器与 AccessPath 的 1:1

```cpp
// access_path.h:357
/// If an iterator has been instantiated for this access path, points to the iterator.
/// Used for constructing iterators that need to talk to each other
/// (e.g. for recursive CTEs, or BKA join), and also for locating timing information.
RowIterator *iterator = nullptr;
```

- **非拥有的裸指针**（所有权在父节点 children 数组里的 `unique_ptr_destroy_only`）。迭代器都分配在 MEM_ROOT 上、生命周期到查询结束，所以安全
- **EXPLAIN ANALYZE 时是 `TimingIterator<T>*`**，取真实类型必须走 `real_iterator()`
- **三个用途**：BKA 拿 `MultiRangeRowIterator`、MATERIALIZE 找 `FollowTailIterator`、EXPLAIN 取 profiler

**`IteratorsAreNeeded()`**（`sql_optimizer.cc:11463`）：次级引擎若声明 `USE_EXTERNAL_EXECUTOR`（整体 offload），则**完全不建迭代器树**。

### 3.4 MATERIALIZE 分支的特殊处理

1. **子节点数是 N+1 且异构**：`children[0]` 是"物化后读临时表的迭代器"（必须是单表访问，有 assert 限定类型）；`children[1..N]` 是各 query block 的迭代器，且**每个可能属于不同的 JOIN**
2. **必须等所有子节点就绪**：因为 `QueryBlock::recursive_reader` 需要通过 `path->iterator` **反向查找**子迭代器树里的 `FollowTailIterator`，这只有子迭代器都创建完、反向指针回填完才可能

---

## 四、执行入口 ExecuteIteratorQuery

`sql/sql_union.cc:1672`。骨架：

```cpp
bool Query_expression::ExecuteIteratorQuery(THD *thd) {
  ...
  if (m_root_iterator->Init()) return true;
  for (;;) {
    int error = m_root_iterator->Read();
    if (error > 0) return true;         // Error
    if (error < 0) break;               // EOF
    if (query_result->send_data(thd, *fields)) return true;
    if (thd->killed) { thd->send_kill_message(); return true; }
    ++send_records;
    if (send_records >= query_block->select_limit_cnt && ...) break;   // LIMIT
  }
  ...
}
```

要点：

- 在 `sql_union.cc` 而非 `sql_select.cc`——因为执行单元是 `Query_expression`（集合操作的最外层），`Query_block` 只是它的一个分支
- `send_data` → `Query_result_send::send_data`（`sql_class.cc`）走协议层回包
- **LIMIT 在这里生效**（`send_records >= select_limit_cnt`）——但注意 `LimitOffsetIterator` 也会提前终止，两者不冲突（后者让引擎少读行）
- `EndPSIBatchModeIfStarted()` 用 RAII 保证**即使出错或提前 LIMIT 也关闭 batch mode**

---

## 五、与 handler / InnoDB 的边界

```
RowIterator（本篇）           handler 接口（../handler.md）      InnoDB（../../innodb/）
─────────────────            ──────────────────────────        ────────────────────
Read() 调：
  ha_rnd_next()       ─────▶  handler::ha_rnd_next       ─────▶  row_search.md
  ha_index_next()     ─────▶  handler::ha_index_next     ─────▶    direction / 游标推进
  ha_rnd_pos()        ─────▶  handler::ha_rnd_pos        ─────▶    need_to_process
  ha_multi_range_read_next()   position/ref 语义                 mvcc.md（可见性）
```

**三方分工**：

| 讲什么 | 在哪 |
|---|---|
| 迭代器框架、各类迭代器算法（状态机、NULL 补全、hash join、排序、物化、窗口） | **本篇** |
| `position`/`ref`/`rnd_pos` 的接口语义、ref 与 EXPLAIN ref 的区别 | [`../handler.md`](../handler.md) |
| `row_search_mvcc`、`direction`、`need_to_process`、游标推进 | [`../../innodb/row_search.md`](../../innodb/row_search.md) |
| DML 的 buffer row id 两阶段读 | [`10_dml.md`](10_dml.md) |

---


## 参考

**论文**
- **Graefe《Volcano—An Extensible and Parallel Query Evaluation System》(1990)** —— **火山模型（iterator model）**。`Init()` / `Read()` 契约的直接来源
- **Graefe《Query Evaluation Techniques for Large Databases》(1993)** —— 迭代器族、sort-merge / hash join 的经典综述

**官方文档**
- *MySQL 8.0 Reference Manual → EXPLAIN ANALYZE*（对应 `TimingIterator` 的 `first_row` / `other_rows`）
- *MySQL 8.0 Reference Manual → Optimizing Queries with EXPLAIN*


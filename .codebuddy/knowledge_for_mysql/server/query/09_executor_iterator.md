# 09 执行器：RowIterator 火山模型

> AccessPath 树 → RowIterator 树 → 一行行读出。全部算法级。
>
> 边界：本篇讲**迭代器框架与各类迭代器的算法**；handler 接口语义（`position`/`ref`/`rnd_pos`）见 [`../handler.md`](../handler.md)，InnoDB 侧取行（游标推进）见 [`../../innodb/row_search.md`](../../innodb/row_search.md)。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [零、定位与关键设计](#零定位与关键设计)
- [一、RowIterator 基类与火山契约](#一rowiterator-基类与火山契约)
- [二、迭代器逐个详解](#二迭代器逐个详解)
- [三、从 AccessPath 到 Iterator：翻译入口](#三从-accesspath-到-iterator翻译入口)（翻译机制详见 [`08_access_path.md`](08_access_path.md)「翻译成执行」）
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

### 并行执行：从 Volcano 的 Exchange 到 morsel-driven

上面两节讲的是"**一次处理多少行**"（一行 / 一批 / 一段编译码）。并行是**另一个维度**："**多少线程同时处理**"。本节系统讲清楚它——因为 MySQL 在这件事上的立场，是理解它执行器定位的关键。

#### 先分清并行的三种粒度

| 粒度 | 含义 | 例子 |
|---|---|---|
| ① **inter-query**（查询间） | 多个查询同时跑 | 连接级 thread pool |
| ② **intra-operator**（算子内） | **一个算子**拆成多份并行做 | 并行 scan、并行 hash join build |
| ③ **inter-operator / pipeline**（算子间） | **不同算子**在不同线程同时跑 | scan 出下一批时 join 正处理上一批 |

②③ 合称 **intra-query parallelism（查询内并行）**——这才是"并行查询"真正所指。**MySQL 有 ①，没有 ②③**（证据见最后）。

#### Exchange：Volcano 的并行方案

Graefe 那篇论文标题里就有 "Parallel"——它的并行化方案就是 **Exchange 算子**。

**要解决的问题**：计划树是静态的，如何在树里表达"并行"？**答案**：在计划中插入 Exchange 作为**数据重分布的关节**。三种形态：

| 形态 | 数据流向 | 作用 |
|---|---|---|
| **Gather**（汇聚） | N → 1 | 把多个 worker 的结果收成一个流（计划顶部必然有） |
| **Repartition**（重分区） | N → N | 按 join key / group key 重新分发，保证同一 key 落到同一 worker |
| **Broadcast**（广播） | 1 → N | 把小表复制给每个 worker（小表 join 大表） |

一个并行 hash join 的形状：

```
              Gather                     ← 结果汇总
                │
      ┌─────────┼─────────┐
    HashJoin  HashJoin  HashJoin         ← 3 个 worker 各做一份
        │         │         │
        └──── Repartition(by key) ────┘  ← 按 join key 重分发，保证能 join 上
        ┌─────────┴─────────┐
      Scan R              Scan S         ← 两个输入各自并行扫
```

**关键特征：并行度被"烤进"（baked into）了计划里。** Exchange 下面挂几个 worker，编译期就定死，执行时不能改。这正是后面 morsel-driven 要批评的地方。

#### Pipeline 与 pipeline breaker

| 概念 | 定义 |
|---|---|
| **pipeline** | 一段算子链，数据在其中**不落盘**，一行（或一个批）从链头流到链尾 |
| **pipeline breaker** | **必须吸干全部输入才能吐出第一行**的算子，会打断流水线 |

典型 breaker：

| 算子 | 为什么是 breaker |
|---|---|
| Sort | 没看完全部输入，不知道谁排第一 |
| Hash join 的 **build** 阶段 | 哈希表没建完不能 probe |
| 哈希聚合 / 临时表聚合 | 同理 |
| Exchange | 收不齐数据没法重分布 |

流式的（非 breaker）：`FilterIterator`、`LimitOffsetIterator`、NLJ 的 probe 侧、`AppendIterator`。

MySQL 里**有意识地消除不必要 breaker** 的证据是 `StreamingIterator`——"本要物化但其实不需要"时的去物化版本：

```cpp
// StreamingIterator is a minimal version of MaterializeIterator that does not
// actually materialize; instead, every Read() just forwards the call to the
// subquery iterator...
// It is used for when the optimizer would normally set up a materialization,
// but you don't actually need one
```

#### morsel-driven 在批评什么

HyPer 的 Leis et al.（SIGMOD 2014，《Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age》）针对的就是 Exchange 这套 **plan-driven parallelism**：

1. **并行度静态**——编译期固定，无法适应运行时：别的查询占了核、某 worker 遇到数据倾斜
2. **调度僵化**——线程与计划的绑定固定，一个线程卡住没人接替
3. **NUMA 不友好**——Exchange 跨节点搬数据

**它的解法**：把"计划里写死并行结构"改成"**运行时动态调度数据分片**"。

- **morsel**：输入切成固定大小的水平分片（千行量级），调度单位是 morsel 而**不是整个算子**
- **dispatcher + worker**：worker 数 ≈ 核数，dispatcher 运行时把 morsel 分给空闲 worker
- **弹性并行度**（论文原话）：*"The degree of parallelism is not baked into the plan but can elastically change during query execution"*
- **NUMA-aware**：morsel 的内存分配在处理它的线程的本地 NUMA 节点
- **pipeline 内不落盘**：一个 morsel 在一个线程内流过整段 pipeline

#### MySQL 站在哪：没有 Exchange，没有查询内并行

| | MySQL 8.0.39 社区版 |
|---|---|
| Exchange 算子 | ❌ **不存在** |
| intra-query 并行 | ❌ **没有** |
| 实际有的 | 后台线程（purge/flush/IO）、MTS 并行回放、thread pool（inter-query）、InnoDB `Parallel_reader`（极窄，见下） |

**Exchange 不存在的硬证据**：`AccessPath::Type` 枚举是 MySQL 计划算子的**全集**（TABLE_SCAN / INDEX_SCAN / REF / HASH_JOIN / NESTED_LOOP_JOIN / FILTER / SORT / AGGREGATE / MATERIALIZE / WINDOW / WEEDOUT / APPEND / STREAM / CACHE_INVALIDATOR ...），其中**没有任何 Exchange / Gather / Repartition / Distribute**。另外 `sql/iterators/` 全部 13 个头文件也无 exchange 类；全库搜 `intra-query` 零命中。

> ⚠️ 唯一匹配 `*exchange*` 的是 `sql/sql_exchange.h`，但那是 `SELECT ... INTO OUTFILE` / `LOAD DATA`，与并行无关——别被名字骗了。

**`innodb_parallel_read_threads` 的真实范围**（常被误传为"并行查询"）：只服务 **DDL 建索引 / CHECK TABLE / 无 WHERE 的 `SELECT COUNT(*)`**，不用于任何带谓词、JOIN、GROUP BY 的普通查询。分派逻辑在 `row_scan_index_for_mysql()` 里一目了然：

```cpp
if (!check_keys) {
  return row_mysql_parallel_select_count_star(trx, indexes, n_threads, n_rows);
}
return parallel_check_table(trx, index, n_threads, n_rows);
```

且它 **bypass 了整个迭代器树**（由 `optimize_aggregated_query()` 在优化阶段决定），扫描完线程即解散——**执行器本身仍是单线程 volcano**。

**最有意思的对照：MySQL 其实"懂" morsel，只是不在 SQL 层。**

`Parallel_reader`（`storage/innobase/include/row0pread.h`）的设计注释几乎是 morsel-driven 的 B+tree 变体——**动态分裂 + 工作窃取 run queue**：

```cpp
// To solve the imbalance problem we dynamically split the sub-trees as and
// when required... the first thread that finishes scanning the first set of 4
// partitions will then dynamically split the 5th sub-tree... As the other
// threads complete their sub-tree scans they will pick up more execution
// contexts (Ctx) from the Parallel_reader run queue
```

但它在 `storage/innobase/`，与 SQL 执行器的迭代器树**完全隔离、无任何连接点**。

> 准确表述：**MySQL 的并行是"存储引擎内部的数据并行"，不是"执行器层的算子并行"**；HyPer 恰恰相反（morsel-driven 是执行器层一等公民）。所以 `Parallel_reader` 可以作为"morsel 思想在存储层的独立实现"引用，但**不能**作为"MySQL 有 morsel-driven 查询执行"的证据。

**HeatWave：把并行外包出去**（架构级佐证）

源码树里没有 HeatWave 实现（闭源插件），但留了让二级引擎**整条查询旁路本地执行器**的钩子：

```cpp
/// A hook that secondary storage engines can use to override the executor
/// completely.
using Override_executor_func = bool (*)(JOIN *, Query_result *);
```

配合 `USE_EXTERNAL_EXECUTOR` 标志与 `secondary_engine_cost_threshold` 变量。这说明 Oracle 的战略是：**社区版执行器保持单线程 volcano，把并行/列式/向量化放在闭源 HeatWave 里**——这本身反证了社区版没有查询内并行。

#### 对照表

| 维度 | MySQL 8.0.39 社区版 | HyPer morsel-driven |
|---|---|---|
| 执行模型 | ✅ 经典 Volcano，`Init()` + `Read()` | Volcano 变体（融合） |
| `next()` 粒度 | **1 行** | morsel（千行量级） |
| Exchange 算子 | ❌ 不存在 | ✅ 核心机制 |
| 查询内并行 | ❌ 没有 | ✅ |
| 调度 | 静态树 + 调用栈 | dispatcher 动态 + 工作窃取 |
| NUMA-aware | ❌ | ✅ |
| pipeline | 事实上有流式（含 `StreamingIterator` 去物化），但**无 pipeline 概念/调度器** | ✅ 显式 pipeline 与 breaker 分析 |

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
  int tmp;
  if (table()->is_union_or_table()) {                // 普通表/UNION：快路径
    while ((tmp = table()->file->ha_rnd_next(m_record))) {
      /* ha_rnd_next can return RECORD_DELETED for MyISAM when one thread is
      reading and another deleting without locks. */
      if (tmp == HA_ERR_RECORD_DELETED && !thd()->killed) continue;
      return HandleError(tmp);
    }
    if (m_examined_rows != nullptr) ++*m_examined_rows;

  } else {                                           // EXCEPT/INTERSECT：带计数器
    while (true) {                                   // 外层循环 = 跳过不合格的行
      // ★ m_remaining_dups > 0 ⇒ 同一行还要再出几次
      //   ⇒ 此时**不再调 ha_rnd_next**，只减计数器就 break
      //     引擎侧只扫了一遍，多行是在这里"展开"出来的
      if (m_remaining_dups == 0) {                   // 初始总是 0
        while ((tmp = table()->file->ha_rnd_next(m_record))) {
          if (tmp == HA_ERR_RECORD_DELETED && !thd()->killed) continue;
          return HandleError(tmp);
        }
        if (m_examined_rows != nullptr) ++*m_examined_rows;

        // Filter out rows not qualifying for INTERSECT, EXCEPT by reading the counter.
        const ulonglong cnt =
            static_cast<ulonglong>(table()->set_counter()->val_int());
        if (table()->is_except()) {
          if (table()->is_distinct()) {
            if (cnt >= 1) break;                     // EXCEPT DISTINCT：cnt≥1 出 1 行
          } else {
            m_remaining_dups = cnt;                  // EXCEPT ALL：出 cnt 行
          }
        } else {
          if (table()->is_distinct()) {
            if (cnt == 0) break;                     // INTERSECT DISTINCT：被减过则跳过
          } else {
            HalfCounter c(cnt);
            m_remaining_dups = std::min(c[0], c[1]); // INTERSECT ALL：min(左重复数, 右已匹配数)
          }
        }
      } else {
        --m_remaining_dups;                          // ★ 同一行再返回一次
        break;
      }
      // 走到这里 = 本行不合格，继续外层循环取下一行
    }
    if (++m_stored_rows > m_limit_rows) {            // 限制实际输出行数
      return HandleError(HA_ERR_END_OF_FILE);
    }
  }
  return 0;
}
```

⇒ ★ **最关键的一点**：`m_remaining_dups > 0` 时**不重新调引擎**——只减计数器就返回同一行。物化时用"一行 + 一个计数器"压缩存储，到扫描阶段在这里展开成多行，**引擎只扫了一遍**。

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

#### 短路与零输入迭代器详解

这几个"一行/零行"迭代器看着简单，但每个都藏着**语义难点或极易踩的边界**。

**① `UnqualifiedCountIterator` ★ —— 一行也不读**

```cpp
int UnqualifiedCountIterator::Read() {
  if (!m_has_row) return -1;
  for (Item *item : *m_join->fields) {
    if (item->type() == Item::SUM_FUNC_ITEM &&
        down_cast<Item_sum *>(item)->sum_func() == Item_sum::COUNT_FUNC) {
      ulonglong count = get_exact_record_count(m_join->qep_tab,
                                               m_join->primary_tables, &error);
      down_cast<Item_sum_count *>(item)->make_const(static_cast<longlong>(count));
    }
  }
  ...
}
```

`get_exact_record_count()` 对每个表调 `ha_records()` 然后**相乘**（笛卡尔积）。

⚠️ **它的适用边界极其苛刻**（判定在 `opt_sum_query()`，必须全部满足）：

- WHERE 必须为**空**（`conds == nullptr`）
- 不能有关联/随机依赖：`where_tables & (OUTER_REF_TABLE_BIT | RAND_TABLE_BIT)` → 放弃
- 不能有 semi-join / antijoin nest
- `COUNT` 的参数必须**不可能为 NULL**（`COUNT(可空列)` 直接作废）
- 不能有外连接的 inner table 被 WHERE 引用
- 不能含 schema table / 物化派生表（行数此刻还不可知）
- **SELECT 列表里只要有任何非常量、非 COUNT 的项就作废**
- 只有 `COUNT_FUNC` 被处理（`SUM/AVG/...` 走 `aggr_impossible`）

★ **最反直觉的一条**：走这个迭代器**恰恰要求"至少有一个表不能瞬间给出精确行数"**。如果所有表都能在优化期给出精确行数，就直接 `make_const` 折成常量、走 "Select tables optimized away"，**根本不用这个迭代器**。

另外 **hypergraph 优化器不做这个优化**（测试自证：`--skip_if_hypergraph  # Does not optimize COUNT(*) specially yet.`）——因为它依赖 `m_join->qep_tab` 这套老结构。

**② `ZeroRowsAggregatedIterator` ★ —— 为什么必须出一行全 NULL**

类注释直接给了例子：

```cpp
/**
  Like ZeroRowsIterator, but produces a single output row, since there are
  aggregation functions present and no GROUP BY. E.g.,

    SELECT SUM(f1) FROM t1 WHERE 2+2 = 5;

  should produce a single row, containing only the value NULL.
 */
```

实现是**把所有基表整行标记为 NULL 行，然后对每个字段调 `no_rows_in_result()`**：

```cpp
  // Mark tables as containing only NULL values
  for (Table_ref *table = m_join->query_block->leaf_tables; ...)
    table->table->set_null_row();

  for (Item *item : *m_join->fields) item->no_rows_in_result();
```

⚠️ 注释特别提醒：**必须对 `m_join->fields` 全部调用**（不只是输出列），因为 `join->having` 可能引用不在结果列里的字段。

这就是 SQL 空集语义的落地：`SUM(空集) = NULL` 而非 0，`COUNT(空集) = 0`，且**结果集仍有一行**。

两者的选择点：

```cpp
  /**
    ... send a row even if the join produced no rows if:
     - there is an aggregate function (sum_func_count!=0), and
     - the query is not grouped, and
     - a possible HAVING clause evaluates to TRUE.
  */
  bool send_row_on_empty_set() const {
    return (do_send_rows && tmp_table_param.sum_func_count != 0 &&
            group_list.empty() && !group_optimized_away &&
            query_block->having_value != Item::COND_FALSE);
  }
```

推论：**有 GROUP BY 时空集就是 0 行**（聚合函数不产生"空组"）；**HAVING 恒假时也是 0 行**。聚合版本之后还会挂 HAVING+LIMIT，非聚合版本则不挂。

两个 AccessPath 的基数标注也印证：`ZERO_ROWS` 是 `0.0`，`ZERO_ROWS_AGGREGATED` 是 `1.0`。

**③ `FakeSingleRowIterator` 的两个场景**

- 全是 const 表（`SELECT 1;`、`SELECT * FROM t1 WHERE pk=3;`）
- **顶层就是外连接**时，给左端造一个"虚拟常数行"（`is_top_level_outer_join`）

**④ `TableValueConstructorIterator` —— 靠"重新绑定引用"换行**

`VALUES ROW(1,10), ROW(2,20)` **不物化、不扫表**。`Read()` 里把每个 `Item_values_column` 的 `set_value()` 改成指向下一行：

```cpp
    for (const Item *value : **m_row_it) {
      Item_values_column *ref = down_cast<Item_values_column *>(*output_refs_it);
      ref->set_value(const_cast<Item *>(value));
    }
```

单行时不创建引用对象，直接用 `Query_block::item_list`。另外它的行序全靠 `examined_rows` 计数器，所以 AccessPath 里强制 `count_examined_rows = true`。

**⑤ 这些短路在哪被决定**

- `JOIN::create_root_access_path_for_join()`：按 `select_count` → `is_table_value_constructor` → `const_tables == primary_tables` 的顺序短路
- `JOIN::create_access_paths_for_zero_rows()`：由 `send_row_on_empty_set()` 分叉
- 最终由 `ConnectIterators()` 落到具体迭代器类

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

#### ★ 归并流程总览

> 源码坐标：`[R1] sql/range_optimizer/rowid_ordered_retrieval.h/.cc` · `[R1b] sql/range_optimizer/rowid_capable_row_iterator.h` · `[R2] sql/range_optimizer/index_merge.h/.cc` · `[R3] sql/range_optimizer/range_optimizer.cc` · `[A1] sql/join_optimizer/access_path.h`
> ⚠️ **8.0.39 路径校正**：这三个迭代器在 **`sql/range_optimizer/`** 下（老的 `sql/opt_range.cc` 已拆分）。子迭代器成员叫 **`m_children`**（不是 `m_source_iters`）。

```
【1】RowIDIntersectionIterator —— 轮转"追赶"式归并，k 路全等才输出（AND）
    scan A  5 ─ 7 ─ 9 ─ 11 ─ …     候选 rowid 由"最先读到它的那一路"提出
    scan B  5 ─ 8 ─ 9 ─ 12 ─ …     last_rowid_count = 已确认的路数
    scan C  4 ─ 5 ─ 9 ─ …          cmp<0 ⇒ 跳过并 UnlockRow；cmp>0 ⇒ 换候选、计数归 1
                                   count == k ⇒ 命中 ⇒ 输出 5, 9（天然无重复）

【2】RowIDUnionIterator —— 优先队列 k 路归并，出队后与 prev_rowid 比一次（OR）
    scan A  5 ─ 9 ┐
    scan B  5 ─ 7 ├─► Priority_queue(min-heap) ─► 5,5,7,7,8,9
    scan C  7 ─ 8 ┘                                  └ 相邻相等即丢弃 ⇒ 5,7,8,9

【3】IndexMergeIterator —— Unique 物化排序去重（两阶段），不做在线归并
    Phase 1 (Init)  scan A 5 9 ┐
                    scan B 5 7 ├─► unique_add(rowid) ─► Unique::get(table)
                    scan C 7 8 ┘   (内存 quicksort / 溢出落盘 merge sort)
                                        │ 有序、无重复的 rowid 流
    Phase 2 (Read)  ────────────────────┘
                    SortFile/SortBufferIndirectIterator 按 rowid 回表 ⇒ 5,7,8,9
                    EOF 后 doing_pk_scan = true ⇒ 接着扫 CPK 分支（两集合 rowid 不相交）
```

#### `RowIDIntersectionIterator`：轮转追赶，凑齐 k 路才输出

```cpp
// [R1b] —— 只有"能给出 rowid"的迭代器才能当 ROR 归并的孩子
class RowIDCapableRowIterator : public TableRowIterator {
 public:
  // 上一次 Read() 成功后该迭代器所处的 rowid；只在 ROR 归并中被父节点读取。
  virtual uchar *last_rowid() const = 0;
};
```

`Init()` 只做"准备"：分配 rowid 缓冲、为每路子扫描**克隆独立 handler**（single handler 只能保存一个 key scan 的上下文）并置 `HA_MRR_SORTED` 保证各路 rowid 升序——这是 ROR 归并的前提。

```cpp
// [R1] —— Read() 的核心循环（节选）
int RowIDIntersectionIterator::Read() {
  size_t current_child_idx = 0;   // ★ 非 static：每次 Read() 都从 0 号孩子开始提候选
  for (;;) {
    RowIterator *child = m_children[current_child_idx].get();
    if (int error = child->Read(); error != 0) return error;
    if (m_cpk_child) {            // ★ CPK 分支只当"过滤器"：候选不在 PK range 里就换下一个
      while (!down_cast<IndexRangeScanIterator *>(m_cpk_child->real_iterator())
                  ->row_in_ranges()) {
        child->UnlockRow();       // 只加锁于最终输出的行：读了又不输出必须释放锁
        if (int error = child->Read(); error != 0) return error;
      }
    }
    const uchar *child_rowid =
        down_cast<IndexRangeScanIterator *>(child->real_iterator())->file->ref;
    memcpy(m_last_rowid, child_rowid, table()->file->ref_length);
    RowIterator *child_with_last_rowid = child;   // 记住"最先读到该 rowid 的那一路"
    uint last_rowid_count = 1;

    while (last_rowid_count < m_children.size()) {   // 让其余各路依次"确认"候选
      current_child_idx = (current_child_idx + 1) % m_children.size();
      child = m_children[current_child_idx].get();
      child_rowid = down_cast<IndexRangeScanIterator *>(child->real_iterator())->file->ref;
      int cmp;
      do {
        if (int error = child->Read(); error != 0) {
          if (!thd()->transaction_rollback_request)   // 死锁等已回滚的场景别再解锁
            child_with_last_rowid->UnlockRow();
          return error;
        }
        cmp = table()->file->cmp_ref(child_rowid, m_last_rowid);  // ★ 纯二进制 ref 比较
        if (cmp < 0) child->UnlockRow();   // 该路落后于候选 ⇒ 丢弃这行 ⇒ 解锁
      } while (cmp < 0);

      if (cmp > 0) {                       // 该路超过了候选 ⇒ 候选升级
        memcpy(m_last_rowid, child_rowid, table()->file->ref_length);
        child_with_last_rowid->UnlockRow();   // 旧候选作废
        last_rowid_count = 1;                 // 计数归 1，重新开始确认
        child_with_last_rowid = child;
      } else {
        last_rowid_count++;                   // cmp == 0：这一路也确认了同一个 rowid
      }
    }
    // 所有路都返回了同一个 rowid —— 交集命中
    if (retrieve_full_rows) {
      int error = table()->file->ha_rnd_pos(table()->record[0], m_last_rowid);
      if (error == HA_ERR_RECORD_DELETED) continue;   // 已被并发删除 ⇒ 回循环再取
      if (error == 0) return 0;
      return HandleError(error);
    } else {
      return 0;   // 覆盖索引，或父节点(Union)会负责回表
    }
  }
}
```

★ `cmp_ref` 是 handler 的**二进制 rowid 比较**——不比较字段值、也不产生三值逻辑，所以字段是否为 NULL 完全不参与归并判定。交集只认"rowid 相等"。

#### `RowIDUnionIterator`：优先队列 k 路归并 + 出堆后去重

```cpp
// [R1] —— 比较器：按"各路当前 rowid"建小顶堆
struct Quick_ror_union_less {
  bool operator()(RowIterator *a, RowIterator *b) {
    RowIDCapableRowIterator *real_a =
        down_cast<RowIDCapableRowIterator *>(a->real_iterator());
    RowIDCapableRowIterator *real_b =
        down_cast<RowIDCapableRowIterator *>(b->real_iterator());
    return m_file->cmp_ref(real_a->last_rowid(), real_b->last_rowid()) > 0;
  }
  const handler *m_file;
};
```

`Init()` 一次性完成：预留堆容量、分配 `cur_rowid`/`prev_rowid` **双缓冲**、对每个孩子 `Init()` 并**先读一行入堆**（k 路归并的"初始填充"），最后 `ha_rnd_init`。

```cpp
// [R1] —— Read()：经典 k 路归并 + 输出后去重
int RowIDUnionIterator::Read() {
  for (;;) {
    bool dup_row;
    do {
      if (queue.empty()) return -1;                     // 所有路都 EOF
      RowIterator *quick = queue.top();                 // 当前全局最小 rowid 所在的那一路
      memcpy(cur_rowid,
             down_cast<RowIDCapableRowIterator *>(quick->real_iterator())->last_rowid(),
             rowid_length);
      if (int ret = quick->Read(); ret != 0) {
        if (ret != -1) return ret;
        queue.pop();                                    // 该路耗尽 ⇒ 出堆
      } else {
        queue.update_top();                             // 该路推进一格 ⇒ 下调维持堆序
      }
      if (!have_prev_rowid) { dup_row = false; have_prev_rowid = true; }
      else dup_row = !table()->file->cmp_ref(cur_rowid, prev_rowid);  // 与"上一次输出的"比
    } while (dup_row);                                  // ★ 相同则丢弃 —— 这就是去重

    using std::swap;
    swap(cur_rowid, prev_rowid);                        // ★ 双缓冲轮转，避免再拷一次
    int error = table()->file->ha_rnd_pos(table()->record[0], prev_rowid);
    if (error == HA_ERR_RECORD_DELETED) continue;
    if (error == 0) return 0;
    return HandleError(error);
  }
}
```

**与 Intersection 的差别**：交集是**判定式**——候选必须被全部 k 路确认才输出，所以不需要去重（同一 rowid 不可能输出两次）、也不需要优先队列（轮转即可）；并集是**产出式**——堆顶最小者即输出，但同一 rowid 可能出现在多路（OR 条件重叠），因此必须在**出堆后、回表前**去重。并集**永远自己回表**。

#### `IndexMergeIterator`：离线物化归并（异类）

★ **为什么必须用 `Unique`**：`ROWID_INTERSECTION/UNION` 要求每路**按 rowid 升序产出**（ROR）才能在线归并；`INDEX_MERGE` 用于那些**不保证 rowid 有序**的分支，无法在线 k 路归并，只能先物化全部 rowid、排序、去重，再顺序回表。代价是 `Unique` 可能落盘。

```cpp
// [R2] —— Init() 就是整个 Phase 1
bool IndexMergeIterator::Init() {
  empty_record(table());
  table()->set_keyread(true);          // 阶段1 只读索引项
  table()->prepare_for_position();     // 需要 position() 才能拿到 rowid
  if (unique == nullptr) {
    unique.reset(new (mem_root) Unique(refpos_order_cmp, (void *)file,
                                       file->ref_length, sort_buffer_size));
  }
  ...
  for (unique_ptr_destroy_only<RowIterator> &child : m_children) {
    IndexRangeScanIterator *range_scan_it =
        down_cast<IndexRangeScanIterator *>(child->real_iterator());
    table()->covering_keys.set_bit(range_scan_it->index);   // 强制走 index-only
    if (child->Init()) return true;
    for (;;) {
      int result = child->Read();
      if (result == -1) break;                       // 该分支 EOF
      // ★ 会被 CPK 分支取到的 rowid 不再入 Unique —— 保证两段结果集不相交
      if (pk_quick_select && down_cast<IndexRangeScanIterator *>(
                                 pk_quick_select.get()->real_iterator())->row_in_ranges())
        continue;
      handler *child_file = range_scan_it->file;
      child_file->position(table()->record[0]);      // 由索引项算出 rowid
      if (unique->unique_add(child_file->ref)) return true;
    }
  }
  if (unique->get(table())) return true;             // 排序+去重
  doing_pk_scan = false;
  table()->set_keyread(false);                       // ★ index_merge 不支持 "using index"
  read_record = init_table_iterator(thd(), table(), false, false);
  return false;
}

// Read() 即 Phase 2
int IndexMergeIterator::Read() {
  if (doing_pk_scan) return pk_quick_select->Read();      // 第二段：聚簇 PK 扫描
  int result;
  if ((result = read_record->Read()) == -1) {
    read_record.reset();
    if (pk_quick_select) {                                // 第一段结束 ⇒ 切到 CPK
      doing_pk_scan = true;
      if (pk_quick_select->Init()) return 1;
      return pk_quick_select->Read();
    }
  }
  return result;
}
```

**层次关系（以源码为准）**：`IndexMergeIterator` **不是** Intersection/Union 的父壳，而是**平级的替代方案**——它要求孩子是 `IndexRangeScanIterator`。真正有上下层关系的是 **Union → Intersection**：优化器对每个 disjunct 先尝试 `get_best_ror_intersect()`，命中就把 `ROWID_INTERSECTION` 作为 `ROWID_UNION` 的孩子，并改写为 `need_rows_in_rowid_order=true` + `retrieve_full_rows=false`（让父节点统一回表）。

#### 优化器侧参数与代价分水岭

```cpp
// [A1] access_path.h
struct { TABLE *table; bool forced_by_hint; bool allow_clustered_primary_key_scan;
         Mem_root_array<AccessPath *> *children; } index_merge;
struct { TABLE *table; Mem_root_array<AccessPath *> *children;
         AccessPath *cpk_child;                 // 聚簇 PK 分支：只过滤不取行
         bool forced_by_hint;
         bool retrieve_full_rows;               // 是否由交集自己 ha_rnd_pos 回表
         bool need_rows_in_rowid_order;         // 是否为父(Union)保存 rowid / 建独立 handler
         bool reuse_handler;                    // 首路复用 table->file
         bool is_covering;                      // 各路索引字段并集 ⊇ needed_fields
       } rowid_intersection;
```

★ **`is_covering` 是代价模型的分水岭**：`retrieve_full_rows = !is_covering`。覆盖时总成本 = `SUM(index_scan_cost)`；不覆盖时还要叠一次 `get_sweep_read_cost()`——按"随机 rowid 回表"的磁盘 sweep 模型（含 seek），通常远大于索引扫描本身。所以优化器能靠"是否 covering"直接否决一个交集计划（`ror_intersect_add()` 里 `is_covering` 一旦为真，`disk_sweep_cost` 记 0）。

选择顺序：所有分支都是 ROR 时**优先造 ROR-union**（源码注释：*guaranteed to be cheaper*），失败才回落 `INDEX_MERGE`。

#### `Init()` / `Read()` 职责分工小结

| 迭代器 | `Init()`（一次性） | `Read()`（每趟） | 回表在谁手里 |
|---|---|---|---|
| `RowIDIntersection` | 分配缓冲、克隆 handler、`ha_rnd_init`、逐路 Init | 轮转追赶，凑齐 k 路才返回 1 行 | `retrieve_full_rows==true` 时自己；否则交父 Union |
| `RowIDUnion` | 建堆、双缓冲、预读首行入堆、`ha_rnd_init` | 取堆顶 → 与 `prev_rowid` 去重 → `ha_rnd_pos` | 永远自己（并集是顶层） |
| `IndexMerge` | **Phase 1 全部**：跑干所有分支、`unique_add`、`get()` 排序、建回表迭代器 | **Phase 2**：取一行；EOF 后切 CPK | `read_record` + CPK 分支 |

★ 一句话记忆：**Intersection/Union 是"在线流式归并"（`Init` 轻、`Read` 重），IndexMerge 是"离线物化归并"（`Init` 极重、`Read` 极轻）**——这也是 `IndexMergeIterator` 在火山模型里显得"异类"的原因：它把主要工作放进了 `Init()`（与 2.7 的 `SortingIterator`、2.8 的 `MaterializeIterator` 同构）。
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

#### ★ `FilterIterator`：最小的火山模型中间节点

> 源码坐标：`[I2] sql/iterators/composite_iterators.h/.cc`（注意：`FilterIterator` 与 `LimitOffsetIterator` 都在 **composite** 文件里，不在 `basic_row_iterators`）
> **与 08 篇分工**：参数搬运（`NewFilterAccessPath()` / `NewLimitOffsetAccessPath()`）属于 [`08_access_path.md`](08_access_path.md)。

```cpp
// [I2] sql/iterators/composite_iterators.cc
int FilterIterator::Read() {
  for (;;) {
    int err = m_source->Read();
    if (err != 0) return err;

    bool matched = m_condition->val_int();

    if (thd()->killed) { thd()->send_kill_message(); return 1; }
    /* check for errors evaluating the condition */
    if (thd()->is_error()) return 1;

    if (!matched) {
      m_source->UnlockRow();
      continue;
    }
    return 0;   // Successful row.
  }
}
```

`Init()` 是纯转发（`return m_source->Init();`）。所以 `FilterIterator` **是无状态的**，可安全反复 `Init()`（相关子查询里会大量发生）。全部状态机在 `Read()` 的 `for(;;)` 里。

**三个容易漏掉的细节**：

1. ★ **条件为 NULL 与为 0 走同一条分支**。`val_int()` 返回 `longlong`，NULL 会被求值路径写成 `0`（并通过 `null_value` 标记），所以 `WHERE NULL` / `WHERE 0` / `WHERE <false>` 都进 `if (!matched)`——**条件为 NULL 的行被丢弃**，不需要单独的 NULL 分支。真正要区分 NULL 的语义（`NOT IN`、外连接 `IS NULL`）在 `Item` 层就已化归成整数。
2. **错误检查在求值之后**：`thd()->killed` 与 `thd()->is_error()` 必须放在 `val_int()` 之后——除零、溢出、类型转换错误都是在这一步产生的（如 `WHERE 1/0`）。
3. ★ **被丢弃的行要 `UnlockRow()`**：否则 InnoDB 会在整个扫描期间保持这些行的锁，直到扫描结束——这是批量模式下锁范围被放大的主要来源。

#### ★ `LimitOffsetIterator`：offset 被推迟到第一次 `Read()`

```cpp
// [I2] sql/iterators/composite_iterators.cc
bool LimitOffsetIterator::Init() {
  if (m_source->Init()) return true;
  if (m_offset > 0) {
    m_seen_rows = m_limit;      // ★ 故意设成"已超限"
    m_needs_offset = true;
  } else {
    m_seen_rows = 0;
    m_needs_offset = false;
  }
  return false;                 // ★ 一行 offset 都不吃
}

int LimitOffsetIterator::Read() {
  if (m_seen_rows >= m_limit) {
    if (m_needs_offset) {
      // We skip OFFSET rows here and not in Init(), since performance schema
      // batch mode may not be set up by the executor before the first Read().
      // This makes sure that
      //   a) we get the performance benefits of batch mode even when reading
      //      OFFSET rows, and
      //   b) we don't inadvertedly enable batch mode (e.g. through the
      //      NestedLoopIterator) during Init(), since the executor may not
      //      be ready to _disable_ it if it gets an error before first Read().
      for (ha_rows row_idx = 0; row_idx < m_offset; ++row_idx) {
        int err = m_source->Read();
        if (err != 0) return err;
        if (m_skipped_rows != nullptr) ++*m_skipped_rows;
        m_source->UnlockRow();
      }
      m_seen_rows = m_offset;
      m_needs_offset = false;
    }

    if (m_seen_rows >= m_limit) {
      if (m_count_all_rows) {
        // Count rows until the end or error (ignore the error if any).
        while (m_source->Read() == 0) ++*m_skipped_rows;
      }
      return -1;
    }
  }

  const int result = m_source->Read();
  if (m_reject_multiple_rows) {
    if (result != 0) { ++m_seen_rows; return result; }
    // We read a row. Check for scalar subquery cardinality violation
    if (m_seen_rows - m_offset > 0) {
      my_error(ER_SUBQUERY_NO_1_ROW, MYF(0));
      return 1;
    }
  }
  ++m_seen_rows;
  return result;
}
```

★ **分工最反直觉的一点**：类注释写着 "Init() eats the first offset rows"，实际却一行 offset 都不吃。原因写在上面那段源码注释里——**PSI batch mode 在 `Init()` 阶段还没被 `ExecuteIteratorQuery()` 装好**，若在 `Init()` 里读 offset 行，① 拿不到 batch mode 的性能收益，② 若中途报错，执行器还没准备好去关掉 batch mode。

**`m_seen_rows` 的双语义**：它累加"已看过多少行"，**包含被 offset 跳过的行**。所以 `LIMIT 10 OFFSET 5` 是 `limit=10, offset=5`，`Read()` 最多吐 5 行——`AccessPath` 里 `limit` 存的是"含 offset 的总量"，不是"要吐的行数"。

**`m_limit == 0` 时的早退**：无 offset 时 `Init()` 后 `m_seen_rows == 0`，`LIMIT 0` 则第一次 `Read()` 进 `0 >= 0` 分支直接返回 -1，**整棵树一次 `Read()` 都没下发**。

> ⚠️ 需区分两件事：这是**本迭代器**在 `Read()` 里早退；而"`LIMIT 0` 时连孩子都不 `Init()`"发生在更上层——`JOIN::create_access_paths_for_zero_rows()` 会把整棵子树换成 `ZERO_ROWS` 路径，`LimitOffsetIterator` 根本不会被创建。

#### `send_records` 与 `SQL_CALC_FOUND_ROWS`

构造点（`[I10] sql/join_optimizer/access_path.cc`）决定 `m_skipped_rows` 指向谁：

```cpp
      case AccessPath::LIMIT_OFFSET: {
        ha_rows *send_records = nullptr;
        if (param.send_records_override != nullptr) {
          send_records = param.send_records_override;
        } else if (join != nullptr) {
          send_records = &join->send_records;
        }
        iterator = NewIterator<LimitOffsetIterator>(
            thd, mem_root, std::move(job.children[0]), param.limit,
            param.offset, param.count_all_rows, param.reject_multiple_rows,
            send_records);
        break;
      }
```

| 场景 | `m_skipped_rows` 指向 | 含义 |
|---|---|---|
| 单 query block 普通 LIMIT | `&join->send_records` | 用于 `FOUND_ROWS()` |
| UNION 上的 LIMIT + `SQL_CALC_FOUND_ROWS` | `send_records_override`（即 `Query_expression::send_records`） | UNION 的计数器挂在 `Query_expression` 上，不在单个 JOIN |
| 无 `SQL_CALC_FOUND_ROWS` | 仍指向 `&join->send_records` | 无害的计数 |

⇒ `count_all_rows` 为 true 时那个 `while (m_source->Read() == 0) ++*m_skipped_rows;` 会**把源迭代器读到 EOF**——这就是 `SQL_CALC_FOUND_ROWS` 拿不到 LIMIT 提前结束优化的根本原因，注释写得很直白："you will not get any performance benefits of early end"。

#### `reject_multiple_rows`：标量子查询的运行期基数检查

被优化器从标量子查询变换来的派生表在编译期无法保证只返回一行，所以装一个运行期守卫：**读到第二行（扣掉 offset 后 `m_seen_rows > 0`）就报 `ER_SUBQUERY_NO_1_ROW`**。

同一枚开关有第二处实现——物化路径里是 `MaterializeIterator` 判 `stored_rows > 1`（见 2.8）。两者不重复触发，`AccessPath` 只会把它下发给其中一个。

#### 迭代火山模型的 Read 链

```
        ExecuteIteratorQuery()  while ((error = root_iterator->Read()) == 0)
                          │
        ┌─────────────────────────────────────────┐
        │           LimitOffsetIterator           │  Read(): 判 m_seen_rows / 推进 offset
        └─────────────────────────────────────────┘
                          │ m_source->Read()
        ┌─────────────────────────────────────────┐
        │             FilterIterator              │  Read(): for(;;) 读→val_int()→丢弃
        └─────────────────────────────────────────┘
                          │ m_source->Read()
        ┌─────────────────────────────────────────┐
        │           NestedLoopIterator            │  Read(): 状态机
        └─────────────────────────────────────────┘
                     ╱                    ╲
        ┌───────────────────┐    ┌───────────────────┐
        │  TableScanIterator │    │  EQRefIterator    │  叶子：直接和 handler 打交道
        └───────────────────┘    └───────────────────┘

   返回值语义（本篇 1.2 契约，逐层原样透传）：
     0  → 有一行，record buffer 已填好
     -1 → EOF
     1  → 错误
```

★ **关键观察**：`Filter` 与 `LimitOffset` 都是**拉模型**——上层每调一次 `Read()` 才按需向下拉，绝不预取。整棵树的驱动力完全来自最顶上那个 `while` 循环。
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

**Hash Join 与 NLJ 的选择**（优化器侧）：hash join 适合**无索引的等值连接**（内层没索引时 NLJ 要全扫，hash join 建一次表只扫一遍）；有索引时 NLJ 的逐行 lookup 通常更便宜。决策在 `setup_join_buffering`——它把内层表的 `op_type` 设为 `QEP_TAB::OT_BNL`，执行器侧再由 `UseHashJoin()` 据此转成 hash join。

> ⚠️ **纠正：`hash_join` 开关在 8.0.39 已失效。** `OPTIMIZER_SWITCH_HASH_JOIN` 在整个 `sql/` 下**只有两处出现**——`sql_const.h` 的常量定义（`1ULL << 21`）与 `sys_vars.cc` 里 `optimizer_switch` 默认位掩码——**没有任何消费点**。经典优化器侧的实际判定就是一个结构体成员比较，完全不读开关：
>
> ```cpp
> static bool UseHashJoin(QEP_TAB *qep_tab) {
>   return qep_tab->op_type == QEP_TAB::OT_BNL;
> }
> ```
>
> hypergraph 优化器侧同样不读（直接由 `ProposeHashJoin` 提出候选）。所以"关掉 `hash_join` 退回 BNL"是**旧版本行为**，8.0.39 已不成立。同理 `HASH_JOIN` / `NO_HASH_JOIN` hint 也**无代码消费**（见 [`07_optimize/11_optimizer_hints.md`](07_optimize/11_optimizer_hints.md)）。

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

> ★ 这些迭代器对应的 **AccessPath 形态**（`NewRemoveDuplicatesOnIndexAccessPath` / `NewNestedLoopSemiJoinWithDuplicateRemovalAccessPath` 等在 `ConnectJoins` 里如何组装）见 [`08_access_path.md`](08_access_path.md)；五种 semi-join 策略分别落成哪个节点/迭代器的对照表见 [`07_optimize/logical/03_semijoin.md`](07_optimize/logical/03_semijoin.md) 5.0 节。

### 2.6 去重

| 迭代器 | 算法 |
|---|---|
| `WeedoutIterator`（`.cc:2008`） | semijoin 的"inner join + 事后去重"。去重键是**外层表的 row ID**（不是行内容）：`position()` 取 row ID → 临时表查重。流式边产边丢 |
| `RemoveDuplicatesIterator`（`.cc:2056`） | 通用 loose scan：`Cached_item::cmp()` 缓存上一行的 group 值比较。**只在输入已有序时正确** |
| `RemoveDuplicatesOnIndexIterator`（`.cc:2098`） | 同上但比较**索引 keypart 原始字节**（`key_copy`+`key_cmp`），比求值快。仅旧优化器用 |

> confluent weedout（对所有字段去重）在**优化期已被改写成 LIMIT 1**（`sql_executor.cc:1216`）。

#### ★ 三种去重迭代器：对应两种 semijoin 策略

> 源码坐标：`[I2] sql/iterators/composite_iterators.h/.cc` · `[I8] sql/sql_executor.cc`（`do_sj_dups_weedout`）· `[I11] sql/sql_executor.h`（`SJ_TMP_TABLE`）
> **与 08 篇分工**：semijoin 策略由优化器选定并落成 `AccessPath::REMOVE_DUPLICATES` / `_ON_INDEX` / `WEEDOUT` 属于 [`08_access_path.md`](08_access_path.md)；本段只讲三个迭代器各自的 `Init()`/`Read()` 语义与内存代价。

```
  semijoin: SELECT * FROM t1 WHERE t1.a IN ( SELECT t2.b FROM t2 );

  策略 A  LooseScan（"先去重，再 join"）
      ────────────────────────────────────────────────────────
      Remove duplicates on t2_idx        <- 沿索引跳过重复键
          └─ Index scan on t2            <- 已按 semijoin key 有序
      →  join 到 t1，每个 t2 键值只参与一次
      →  内存：O(1)（只存上一个键值）

  策略 B  Duplicate Weedout（"先 join，再去重"）
      ────────────────────────────────────────────────────────
      Nested loop (t1 ⋈ t2)              <- 当作普通 inner join，可自由重排
          └─ Weedout                     <- 用 rowid 组合写临时表判重
      →  内存：O(已输出行数 × rowid 组合长度)

  共同前提：semijoin 一旦能"去掉重复"，就能当 inner join 重排 join 顺序；
  inner join 可重排、semijoin 不可重排 —— 这是所有 semijoin 策略的出发点。
```

**`RemoveDuplicatesIterator`：按 `group_items` 与上一行比较**

```cpp
// [I2] sql/iterators/composite_iterators.h
/**
  An iterator that removes consecutive rows that are the same according to
  a set of items (typically the join key), so-called "loose scan"
  (not to be confused with "loose index scan", which is made by the
  range optimizer). This is similar in spirit to WeedoutIterator above
  (removing duplicates allows us to treat the semijoin as a normal join),
  but is much cheaper if the data is already ordered/grouped correctly,
  as the removal can happen before the join, and it does not need a
  temporary table.
 */
class RemoveDuplicatesIterator final : public RowIterator {
 public:
  RemoveDuplicatesIterator(THD *thd,
                           unique_ptr_destroy_only<RowIterator> source,
                           JOIN *join, Item **group_items,
                           int group_items_size);
  bool Init() override;
  int Read() override;
 private:
  unique_ptr_destroy_only<RowIterator> m_source;
  Bounds_checked_array<Cached_item *> m_caches;
  bool m_first_row;
};
```

```cpp
// [I2] sql/iterators/composite_iterators.cc
bool RemoveDuplicatesIterator::Init() {
  m_first_row = true;
  return m_source->Init();
}

int RemoveDuplicatesIterator::Read() {
  for (;;) {
    int err = m_source->Read();
    if (err != 0) return err;
    if (thd()->killed) { thd()->send_kill_message(); return 1; }

    bool any_changed = false;
    for (Cached_item *cache : m_caches) {
      any_changed |= cache->cmp();
    }

    if (m_first_row || any_changed) {
      m_first_row = false;
      return 0;
    }
    continue;   // Same as previous row, so keep scanning.
  }
}
```

★ `Init()` 只置 `m_first_row = true`，**故意不重置 `m_caches`**——因为 `Cached_item::cmp()` 的语义是"比当前值与我缓存的值，并把当前值缓存下来"，第一次调用时缓存是脏的，靠 `m_first_row` 兜住"第一行必须输出"。

★ `any_changed |= cache->cmp()` 中的 `|=` **不能写成 `||`**——短路会让后面的 `Cached_item` 拿不到更新，下一行就比错了。这是这段代码最微妙的地方。

**`RemoveDuplicatesOnIndexIterator`：真正的 LooseScan**

```cpp
// [I2] sql/iterators/composite_iterators.cc
int RemoveDuplicatesOnIndexIterator::Read() {
  for (;;) {
    int err = m_source->Read();
    if (err != 0) return err;
    if (thd()->killed) { thd()->send_kill_message(); return 1; }

    // ★ "跳到下一个不同键值"就是这三行
    if (!m_first_row && key_cmp(m_key->key_part, m_key_buf, m_key_len) == 0) {
      continue;                        // 键值相同 → 丢弃，继续向下拉一行
    }
    m_first_row = false;
    key_copy(m_key_buf, m_table->record[0], m_key, m_key_len);
    return 0;
  }
}
```

⇒ **为什么 LooseScan 不需要额外内存**：整个判重状态就是 `m_key_buf` 这一个固定长度（`key_len`）的缓冲区。代价是它依赖孩子按该索引前缀有序，且只认索引前缀（不像 `Cached_item` 能对任意表达式去重）。

`NestedLoopSemiJoinWithDuplicateRemovalIterator` 则把去重塞进 join 里（semijoin 内侧有多张表时用）：

```cpp
// [I2] sql/iterators/composite_iterators.cc
bool NestedLoopSemiJoinWithDuplicateRemovalIterator::Init() {
  if (m_source_outer->Init()) return true;
  m_deduplicate_against_previous_row = false;
  return false;    // ★ 故意不 Init() 内层：内层由 Read() 每次自己 Init()
}

int NestedLoopSemiJoinWithDuplicateRemovalIterator::Read() {
  m_source_inner->SetNullRowFlag(false);
  for (;;) {                                  // Termination condition within loop.
    for (;;) {                                // 找一个与上次不同的外层行
      int err = m_source_outer->Read();
      if (err != 0) return err;
      if (m_deduplicate_against_previous_row &&
          key_cmp(m_key->key_part, m_key_buf, m_key_len) == 0) {
        continue;                             // 同键 → 跳过
      }
      break;
    }
    if (m_source_inner->Init()) return 1;     // 每换一个外层行就重扫一次内层
    int err = m_source_inner->Read();
    if (err == 1) return 1;                   // Error
    if (err == -1) {
      // 本外层行在内层没匹配到 → 继续找同键的下一个外层行
      m_deduplicate_against_previous_row = false;
      continue;
    }
    m_deduplicate_against_previous_row = true;
    key_copy(m_key_buf, m_table_outer->record[0], m_key, m_key_len);
    return 0;
  }
}
```

**`WeedoutIterator`：Duplicate Weedout**

```cpp
// [I2] sql/iterators/composite_iterators.cc
bool WeedoutIterator::Init() {
  if (m_sj->tmp_table->file->ha_delete_all_rows()) return true;   // 清空判重表
  if (m_sj->tmp_table->hash_field != nullptr &&
      !m_sj->tmp_table->file->inited) {
    m_sj->tmp_table->file->ha_index_init(0, false);
  }
  for (SJ_TMP_TABLE_TAB *tab = m_sj->tabs; tab != m_sj->tabs_end; ++tab) {
    TABLE *table = tab->qep_tab->table();
    if (m_tables_to_get_rowid_for & table->pos_in_table_list->map()) {
      table->prepare_for_position();      // 告诉 handler 接下来要 position() 拿 rowid
    }
  }
  return m_source->Init();
}

int WeedoutIterator::Read() {
  for (;;) {
    int ret = m_source->Read();
    if (ret != 0) return ret;

    for (SJ_TMP_TABLE_TAB *tab = m_sj->tabs; tab != m_sj->tabs_end; ++tab) {
      TABLE *table = tab->qep_tab->table();
      if ((m_tables_to_get_rowid_for & table->pos_in_table_list->map()) &&
          can_call_position(table)) {
        table->file->position(table->record[0]);   // ← 抓 rowid 到 file->ref
      }
    }

    ret = do_sj_dups_weedout(thd(), m_sj);
    if (ret == -1) return 1;    // Error
    if (ret == 0) return 0;     // Not a duplicate
    // Duplicate → 继续拉下一行
  }
}
```

注意 `Read()` **没有**像 `FilterIterator` 那样对丢弃的行 `UnlockRow()`——因为丢弃的是"重复的组合"，其中的行仍会被上层以别的组合使用。

**判重的两级结构与行格式**

```cpp
// [I8] sql/sql_executor.cc —— do_sj_dups_weedout() 拼行
  uchar *ptr = sjtbl->tmp_table->visible_field_ptr()[0]->field_ptr();
  // 1. 存长度
  if (sjtbl->tmp_table->visible_field_ptr()[0]->get_length_bytes() == 1) {
    *ptr = (uchar)(sjtbl->rowid_len + sjtbl->null_bytes); ptr++;
  } else {
    int2store(ptr, sjtbl->rowid_len + sjtbl->null_bytes); ptr += 2;
  }
  // 2. 清零 null bytes
  uchar *const nulls_ptr = ptr;
  if (sjtbl->null_bytes) { memset(ptr, 0, sjtbl->null_bytes); ptr += sjtbl->null_bytes; }
  // 3. 逐个拷 rowid
  for (uint i = 0; tab != tab_end; tab++, i++) {
    handler *h = tab->qep_tab->table()->file;
    if (tab->qep_tab->table()->is_nullable() &&
        tab->qep_tab->table()->has_null_row()) {
      *(nulls_ptr + tab->null_byte) |= tab->null_bit;      // NULL 补行
      memset(ptr + tab->rowid_offset, 0, h->ref_length);
    } else {
      memcpy(ptr + tab->rowid_offset, h->ref, h->ref_length);
    }
  }

  if (!check_unique_constraint(sjtbl->tmp_table)) return 1;   // 第一级：hash 判重
  error = sjtbl->tmp_table->file->ha_write_row(sjtbl->tmp_table->record[0]);
  if (error) {
    if (sjtbl->tmp_table->file->is_ignorable_error(error)) return 1;  // 第二级：唯一键冲突
    ...
  }
  return 0;
```

```
  第一级：check_unique_constraint()  —— 只在 hash_field 存在时用
                                       （键太长放不进 B-tree 唯一索引时的退路）
  第二级：ha_write_row() 唯一键冲突 —— 常规路径，ignorable error 即重复

  临时表行格式：[len:1或2字节][null bytes][rowid(t1)][rowid(t2)]...
                                 └── 由 SJ_TMP_TABLE_TAB::rowid_offset 定位
```

⇒ **Weedout 为什么比 LooseScan 贵**：必须在 join **之后**判重，所以 join 本身会产生所有重复组合（放大），且临时表随输出行数增长，行数多时 `create_ondisk_from_heap()` 溢写磁盘。换来的是"去重发生在 t1 侧"以及"join 顺序完全自由"。

⚠️ **8.0.39 符号核对**：老版本叫 `check_unique`，8.0.39 实际是 **`check_unique_constraint()`**（`sql/sql_tmp_table.cc`）。它返回 true 表示"约束满足=**不重复**"，命名容易看反。
### 2.7 排序

**`SortingIterator`：整个排序发生在 `Init()`**（`sorting_iterator.cc:438`）。之后按 2×2 组合选结果迭代器：

|  | addon fields（行数据随排序结果存） | 无 addon（只存 `handler::ref`） |
|---|---|---|
| **内存** | `SortBufferIterator<Packed>` | `SortBufferIndirectIterator` |
| **文件** | `SortFileIterator<Packed>` | `SortFileIndirectIterator` |

- **addon fields**：排序记录里直接带 SELECT 需要的列，**读结果不用回表**
- **indirect**：只存 `handler::ref`（行位置），读结果时 `ha_rnd_pos()` **回表**（8.0.20 后仅 UPDATE/DELETE 两阶段读、FTS、大 BLOB 用）

> 两种载体的本质（addon = 存列值 vs rowid = 存 `handler::ref`）与 8.0.20 演进见 [`runtime/01_filesort.md`](runtime/01_filesort.md) 1.3 节。

六种结果迭代器通过 `IteratorHolder` union（`sorting_iterator.h:139`）**共享同一块存储**，避免堆分配。

`SortBufferIndirectIterator::Read()`（`:362`）每条记录前缀可能有 1 字节 NULL 标志（标识"这行是 NULL 补全行"），之后是各表 row ID，逐表 `ha_rnd_pos()` 回表。

**溢出判定**：由 `filesort()` 决定——`m_sort_result.io_cache` 被真正初始化就说明落盘了。`m_num_rows_estimate` 只用于**是否启用优先队列**（Top-N 堆排序）。

#### ★ `SortingIterator`：`Init()` 排序、`Read()` 纯代理

> 源码坐标：`[I3] sql/iterators/sorting_iterator.h/.cc` · `[I7] sql/filesort.cc` · `[I12] sql/filesort.h`（`Filesort`）· `[I13] sql/table.h`（`TABLE::sorting_iterator` / `duplicate_removal_iterator`）
> **与 08 篇分工**：`NewSortAccessPath()` 如何从 ORDER BY / GROUP BY 造出 `AccessPath::SORT` 属于 [`08_access_path.md`](08_access_path.md)。

**`Init()` / `Read()` 职责分工**——本篇里分工最极端的例子：

```
   Init()
     ├─ ReleaseBuffers()            释放上一次的排序结果（若有）
     ├─ DoSort()                    ★ 把 m_source_iterator 拉到 EOF，交给 ::filesort()
     └─ 根据"结果落在哪"选一个 result iterator（6 选 1），Init() 它

   Read()
     └─ return m_result_iterator->Read()     纯代理（一行函数）

   ★ 再调一次 Init() → 会重新 DoSort()，不是 rewind。
     原因见类注释：依赖的 Index_lookup 可能变了（相关子查询里会发生）。
```

```cpp
// [I3] sql/iterators/sorting_iterator.h
/**
  An adapter that takes in another RowIterator and produces the same rows,
  just in sorted order. (The actual sort happens in Init().) Unfortunately, it
  is still bound to working off a TABLE object, which means that you can't
  use it to e.g. sort the output of a join without materializing into a
  temporary table first ...

  The primary reason for this is that we currently have no way of communicating
  read sets through Iterators, and SortingIterator needs to add fields used in
  ORDER BY to the read set for the appropriate tables. ...
 */
class SortingIterator final : public RowIterator {
 public:
  // Does not take ownership of "filesort" ... It _does_ take ownership of
  // "source" ... "examined_rows", if not nullptr, is incremented for each
  // successful Read().
  //
  // num_rows_estimate is used only for whether we intend to use the priority
  // queue optimization or not; if we estimate fewer rows than we can fit into
  // RAM, we never use the priority queue.
  SortingIterator(THD *thd, Filesort *filesort,
                  unique_ptr_destroy_only<RowIterator> source,
                  ha_rows num_rows_estimate, table_map tables_to_get_rowid_for,
                  ha_rows *examined_rows);

  bool Init() override;
  int Read() override { return m_result_iterator->Read(); }
  void UnlockRow() override { m_result_iterator->UnlockRow(); }
  void CleanupAfterQuery();

 private:
  int DoSort();
  void ReleaseBuffers();

  Filesort *m_filesort;
  unique_ptr_destroy_only<RowIterator> m_source_iterator;
  unique_ptr_destroy_only<RowIterator> m_result_iterator;
  Filesort_info m_fs_info;
  Sort_result m_sort_result;
  const ha_rows m_num_rows_estimate;
  const table_map m_tables_to_get_rowid_for;
  ha_rows *m_examined_rows;

  // ★ 6 种结果迭代器之一，union 避免堆分配
  union IteratorHolder {
    IteratorHolder() {}
    ~IteratorHolder() {}
    SortBufferIterator<true> sort_buffer_packed_addons;
    SortBufferIterator<false> sort_buffer;
    SortBufferIndirectIterator sort_buffer_indirect;
    SortFileIterator<true> sort_file_packed_addons;
    SortFileIterator<false> sort_file;
    SortFileIndirectIterator sort_file_indirect;
  } m_result_iterator_holder;
};
```

`Init()` 结尾那个 **6 选 1 分派表**是本段最值得记的结构：

| 条件 | 结果迭代器 | 含义 |
|---|---|---|
| 内存 + addon + packed | `SortBufferIterator<true>` | 缓冲里直接是完整行（压缩） |
| 内存 + addon + 非 packed | `SortBufferIterator<false>` | 缓冲里是完整行 |
| 内存 + **rowid 排序** | `SortBufferIndirectIterator` | 缓冲里只有 rowid，需回表 |
| 溢写磁盘 + addon + packed | `SortFileIterator<true>` | 从 IO_CACHE 读完整行 |
| 溢写磁盘 + addon + 非 packed | `SortFileIterator<false>` | 同上 |
| 溢写磁盘 + rowid | `SortFileIndirectIterator` | 从 IO_CACHE 读 rowid 再回表 |

```cpp
// [I3] sql/iterators/sorting_iterator.cc
int SortingIterator::DoSort() {
  assert(m_sort_result.io_cache == nullptr);
  m_sort_result.io_cache =
      (IO_CACHE *)my_malloc(key_memory_TABLE_sort_io_cache, sizeof(IO_CACHE),
                            MYF(MY_WME | MY_ZEROFILL));

  ha_rows found_rows;
  bool error = ::filesort(thd(), m_filesort, m_source_iterator.get(),
                          m_tables_to_get_rowid_for, m_num_rows_estimate,
                          &m_fs_info, &m_sort_result, &found_rows);
  for (TABLE *table : m_filesort->tables) {
    table->set_keyread(false);  // Restore if we used indexes
  }
  return error;
}
```

**与 `Filesort` 的分工**

```
   SortingIterator（本文件）                    filesort.cc（真正的排序）
   ───────────────────────────               ─────────────────────────────
   · 驱动：把 source iterator 交出去           · 从 source_iterator 拉所有行
   · 持有 Filesort*（不拥有）                  · 决定 addon fields vs rowid
   · 持有 Filesort_info（排序缓冲）            · 决定 PQ vs 全排 vs 归并
   · 持有 Sort_result（结果落在哪）            · 真正比较、写 chunk、merge
   · Init() 后根据结果形态造 result iterator    · 把结果放进 Sort_result
   · Read() 逐行吐出                           · 排序期间不吐任何行

   一句话：SortingIterator = "驱动 filesort 并逐行吐出"，排序本身全在 filesort.cc。
```

#### ★ 两条路径：普通排序 vs 排序去重

⚠️ **8.0.39 符号核对**：`m_remove_duplicates` **不在** `SortingIterator` 上，而在 **`Filesort`**（`[I12]`）和 `Sort_param`（`[I7]`）上——这是常被记错的一点。路径选择在优化期构造 `Filesort` 时就定了：

```cpp
// [I7] sql/filesort.cc —— Filesort 构造函数
Filesort::Filesort(THD *thd, Mem_root_array<TABLE *> tables_arg,
                   bool keep_buffers_arg, ORDER *order, ha_rows limit_arg,
                   bool remove_duplicates, bool force_sort_rowids,
                   bool unwrap_rollup)
    : m_thd(thd), tables(std::move(tables_arg)),
      keep_buffers(keep_buffers_arg), limit(limit_arg),
      sortorder(nullptr), using_pq(false),
      m_remove_duplicates(remove_duplicates),
      m_force_sort_rowids(force_sort_rowids),
      m_sort_order_length(make_sortorder(order, unwrap_rollup)) {}
```

去重在两个时机生效（**不是只在内存排序时**）：

```cpp
// [I7] —— 内存路径：额外分配"上一个键"的暂存区
    // If deduplicating, we'll need to remember the previous key somehow.
    if (filesort->m_remove_duplicates) {
      param->m_last_key_seen =
          static_cast<uchar *>(thd->mem_root->Alloc(longest_key));
    }

// [I7] —— 归并/落盘路径：靠"当前键必须严格大于上一个已输出的键"判重
        bool is_duplicate = false;
        if (param->m_remove_duplicates) {
          if (seen_any_records &&
              !mcl.key_is_greater_than(merge_chunk->current_key(),
                                       param->m_last_key_seen)) {
            is_duplicate = true;
          }
        }
```

```
   普通排序 (m_remove_duplicates == false)
   ─────────────────────────────────────
   结果行数 == 输入行数

   排序去重 (m_remove_duplicates == true，典型：SELECT DISTINCT / GROUP BY)
   ────────────────────────────────────────────────────────────────────
   排序/归并过程中丢弃相邻同键行
     · 内存路径：param->m_last_key_seen 暂存上一个键
     · 落盘路径：key_is_greater_than(current, last_seen) 判重
   结果行数 <= 输入行数

   ★ SortingIterator::Init() 的 6 选 1 分派对两条路径是同一份代码，
     去重完全发生在 filesort 内部。
```

已知限制（`[I7]`）：**`SELECT DISTINCT ... ORDER BY x LIMIT n` 拿不到优先队列（PQ）优化**——

```cpp
  if (param->m_remove_duplicates) {
    trace_filesort.add("usable", false)
        .add_alnum("cause", "duplicate removal not supported yet");
    return false;
  }
```

#### ★ rowid 排序：`tables_to_get_rowid_for` 非空时

```
   addon fields 排序（默认，快）
   ─────────────────────────────
   排序缓冲里存：[sort key][SELECT 需要的字段值(addon)]
   Read() 时只需把 addon 解包回 record buffer —— 无回表
   → SortBufferIterator / SortFileIterator

   rowid 排序（tables_to_get_rowid_for 非空，或 addon 太长，或 force_sort_rowids）
   ─────────────────────────────────────────────────────────────────────────
   排序缓冲里存：[sort key][rowid]（+ 每表 1 字节 NULL flag，若表可空）
   1. filesort() 调 table->prepare_for_position()（让 handler 准备好 rowid buffer）
   2. result iterator 的 Init() 对每张表 ha_index_or_rnd_end() 再 ha_rnd_init()
   3. Read() 每次：读 rowid → table->file->ha_rnd_pos(record[0], ref_pos)
   → SortBufferIndirectIterator / SortFileIndirectIterator
   代价：每行一次随机回表 I/O（"indirect" 名字的来源）
```

```cpp
// [I3] sql/iterators/sorting_iterator.cc —— 为什么要重新开关扫描
bool SortFileIndirectIterator::Init() {
  m_sum_ref_length = 0;
  for (TABLE *table : m_tables) {
    // The sort's source iterator could have initialized an index
    // read, and it won't call end until it's destroyed (which we
    // can't do before destroying SortingIterator, since we may need
    // to scan/sort multiple times). Thus, as a small hack, we need
    // to reset it here.
    table->file->ha_index_or_rnd_end();
    EndFullTextIndexScan(table);

    int error = table->file->ha_rnd_init(false);
    if (error) { table->file->print_error(error, MYF(0)); return true; }

    if (m_has_null_flags && table->is_nullable()) ++m_sum_ref_length;
    m_sum_ref_length += table->file->ref_length;
  }
  if (m_ref_pos == nullptr) {
    m_ref_pos = thd()->mem_root->ArrayAlloc<uchar>(m_sum_ref_length);
  }
  return false;
}

int SortFileIndirectIterator::Read() {
  for (;;) {
    if (my_b_read(m_io_cache, m_ref_pos, m_sum_ref_length))
      return -1; /* End of file */
    uchar *ref_pos = m_ref_pos;
    bool skip = false;
    for (TABLE *table : m_tables) {
      if (m_has_null_flags && table->is_nullable()) {
        if (*ref_pos++) {                    // NULL 补行
          table->set_null_row();
          ref_pos += table->file->ref_length;
          continue;
        } else {
          table->reset_null_row();
        }
      }
      int tmp = table->file->ha_rnd_pos(table->record[0], ref_pos);
      ref_pos += table->file->ref_length;
      /* The following is extremely unlikely to happen */
      if (tmp == HA_ERR_RECORD_DELETED ||
          (tmp == HA_ERR_KEY_NOT_FOUND && m_ignore_not_found_rows)) {
        skip = true;                          // ★ 排序期间到回表期间行被并发删掉 → 静默跳过
        break;
      } else if (tmp != 0) {
        return HandleError(thd(), table, tmp);
      }
    }
    if (skip) continue;
    if (m_examined_rows != nullptr) ++*m_examined_rows;
    return 0;
  }
}
```

#### 预估行数如何影响 buffer 决策

`m_num_rows_estimate` 只在 `filesort()` 里用于两个决策点。

**决策点一：是否用优先队列 PQ**

```cpp
  if (check_if_pq_applicable(trace, param, fs_info, num_rows_estimate,
                             memory_available)) {
    ...
    filesort->using_pq = true;
    param->using_pq = true;
    param->m_addon_fields_status = Addon_fields_status::using_priority_queue;
  }
```

**决策点二：给排序缓冲预分配多大**（只在不用 PQ 时）

```cpp
    // NOTE: param->max_rows_per_buffer is merely informative (for optimizer
    // trace) in this case, not actually used.
    if (num_rows_estimate < MERGEBUFF2) num_rows_estimate = MERGEBUFF2;
    ha_rows keys =
        memory_available / (param->max_record_length() + sizeof(char *));
    param->max_rows_per_buffer =
        min(num_rows_estimate > 0 ? num_rows_estimate : 1, keys);
    fs_info->set_max_size(memory_available, param->max_record_length());
```

⇒ 缓冲能放多少行 = `min(预估行数, sort_buffer_size / (行长 + 指针长))`，至少按 `MERGEBUFF2`（=15）估。**优化器估得太低会让缓冲预分配偏小**，导致本可一趟内存排完的结果被切成更多 chunk、写盘、再归并。它**不改变**内存上限（仍是 `sort_buffer_size`），只改变"一次性往里塞多少"的预期。

#### 与 `TABLE::sorting_iterator` / `duplicate_removal_iterator` 的反向绑定

```cpp
// [I13] sql/table.h
  /**
    Not owned by the TABLE; used only from filesort_free_buffers().
    See comments on SortingIterator::CleanupAfterQuery().
  */
  SortingIterator *sorting_iterator{nullptr};
  SortingIterator *duplicate_removal_iterator{nullptr};
```

绑定发生在创建 `SortingIterator` **之后**（`[I10]`），是一条**反向指针**（`TABLE` → 曾在它上面排过序的迭代器），用途只有一个：提前释放排序缓冲。

```cpp
// [I7] sql/filesort.cc
void filesort_free_buffers(TABLE *table, bool full) {
  table->unique_result.sorted_result.reset();
  ...
  if (full) {
    if (table->sorting_iterator != nullptr) {
      table->sorting_iterator->CleanupAfterQuery();
      table->sorting_iterator = nullptr;  // The TABLE object can be reused.
    }
    if (table->duplicate_removal_iterator != nullptr) {
      table->duplicate_removal_iterator->CleanupAfterQuery();
      table->duplicate_removal_iterator = nullptr;
    }
  }
}
```

⇒ 为什么要反向绑定？排序缓冲可能很大（`sort_buffer_size` 级）。`JOIN::destroy()` 时会整体释放，这通常够用；但 DELETE/UPDATE、`ALTER TABLE ... ORDER BY` 等一次性 filesort 希望更早释放。TABLE **不拥有**这两个指针，销毁迭代器前必须手动置 nullptr，否则悬空。
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

#### ★ `MaterializePathParameters`：两个关键开关

> 源码坐标：`[I2] sql/iterators/composite_iterators.h/.cc` · `[I14] sql/join_optimizer/materialize_path_parameters.h` · `[I1] sql/iterators/basic_row_iterators.h`（`FollowTailIterator`）
> **与 08 篇分工**：`NewMaterializeAccessPath()` 如何把派生表/CTE/UNION 折成 `AccessPath::MATERIALIZE` 属于 [`08_access_path.md`](08_access_path.md)。

```cpp
// [I14] sql/join_optimizer/materialize_path_parameters.h
  /**
     Used for when pushing LIMIT down to MaterializeIterator; this is
     more efficient than having a LimitOffsetIterator above the
     MaterializeIterator, since we can stop materializing when there are
     enough rows. (This is especially important for recursive CTEs.)
     Note that we cannot have a LimitOffsetIterator _below_ the
     MaterializeIterator, as that would count wrong if we have deduplication,
     and would not work at all for recursive CTEs.
     Set to HA_POS_ERROR for no limit.
  */
  ha_rows limit_rows;

  /**
     True if this is the top level iterator for a
     materialized derived table transformed from a scalar subquery which needs
     run-time cardinality check.
  */
  bool reject_multiple_rows;
```

⇒ **`limit_rows` 为什么必须专门开一个参数**：把 `LIMIT` 放在 MaterializeIterator **上面**只是"少读几行"，省不下物化代价；放在**下面**则在有去重时会数错（去重吃掉的行也算了 limit），对递归 CTE 完全不成立。所以要下推到物化内部。

#### ★ `MaterializeIterator::Init()`：invalidators 如何触发清空

```cpp
// [I2] sql/iterators/composite_iterators.cc
template <typename Profiler>
bool MaterializeIterator<Profiler>::Init() {
  // ① CTE 共享物化：若本 CTE 已被别的引用物化过，直接用
  const bool use_shared_cte_materialization =
      !table()->materialized && m_cte != nullptr && !m_rematerialize &&
      any_of(m_cte->tmp_tables.begin(), m_cte->tmp_tables.end(),
             [](const Table_ref *table_ref) {
               return table_ref->table != nullptr &&
                      table_ref->table->materialized;
             });

  if (table()->materialized) {
    bool rematerialize = m_rematerialize;

    if (!rematerialize && !use_shared_cte_materialization) {
      /*
        See if any lateral tables that we depend on have changed since
        last time (which would force a rematerialization).
        TODO: It would be better ... to check the actual column values
        instead of just whether we've seen any new rows.
      */
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
      return m_table_iterator->Init();
    }
  }
  table()->set_not_started();

  if (!table()->is_created()) {
    if (instantiate_tmp_table(thd(), table())) return true;
    empty_record(table());
  } else {
    table()->file->ha_index_or_rnd_end();
    table()->file->ha_delete_all_rows();        // ★ 清空
  }
  ...
  // ② 逐个 query block 物化
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
          break;                                // ★ limit 下推的生效点
        }
      }
    }
  }

  table()->materialized = true;
  // ③ 刷新 invalidators 的 generation 快照
  for (Invalidator &invalidator : m_invalidators) {
    invalidator.generation_at_last_materialize =
        invalidator.iterator->generation();
  }
  return m_table_iterator->Init();
}
```

**invalidators 触发机制**：

```
   每个 CacheInvalidatorIterator 维持一个自增 generation 计数器：
     CacheInvalidatorIterator::Init() / Read() / SetNullRowFlag() → ++m_generation

   MaterializeIterator 构造时记下"上次物化时各 invalidator 的 generation"（初值 -1）

   每次 Init()：
     for (invalidator : m_invalidators)
        if (invalidator.iterator->generation()
            != invalidator.generation_at_last_materialize)
            → rematerialize = true;

   物化完成后刷新快照。

   ★ 典型场景：LATERAL 派生表。外层每产出一行，内层依赖它的派生表内容就变了
     → generation 涨 → 下一次 Init() 走 rematerialize 分支
       ha_delete_all_rows() 清空 + 重新跑所有 query block

   ⚠️ invalidation 是"粗粒度"的：只看有没有新行流过，不比值（源码 TODO 明说）。
```

单块物化的读循环（`MaterializeQueryBlock()`）：

```cpp
  while (true) {
    if (*stored_rows >= m_limit_rows) break;          // ★ limit 下推

    int error = query_block.subquery_iterator->Read();
    if (error > 0 || thd()->is_error()) return true;
    else if (error < 0) break;
    else if (thd()->killed) { thd()->send_kill_message(); return true; }

    if (query_block.copy_items) {                     // ★ 求值
      if (copy_funcs(query_block.temp_table_param, thd())) return true;
    }

    if (is_union_or_table) {
      if (query_block.disable_deduplication_by_hash_field) {
        assert(doing_hash_deduplication());
      } else if (!check_unique_constraint(t)) {
        continue;                                     // 重复 → 丢弃
      }
    } else if (query_block.m_operand_idx == 0) {
      ...  // INTERSECT / EXCEPT 左侧：维护 set_counter
    }

    error = t->file->ha_write_row(t->record[0]);
    if (error == 0) { ++*stored_rows; continue; }
    if (!t->file->is_ignorable_error(error)) {
      // 内存临时表满 → 转磁盘
      bool is_duplicate;
      if (create_ondisk_from_heap(thd(), t, error, true, true, &is_duplicate))
        return true;
      if (t->hash_field) t->file->ha_index_init(0, false);
      ...
    }
  }
```

#### 多 query block 物化流程（含递归 CTE）

```
                     MaterializeIterator::Init()
                                 │
                 ┌───────────────┴───────────────┐
                 │  已 materialized?              │
                 │  · CTE 共享命中 → 跳过         │
                 │  · rematerialize == true → 重跑│
                 │  · invalidators generation 变 → 重跑
                 │  · 否则只 m_table_iterator->Init() 直接返回（rescan）
                 └───────────────┬───────────────┘
                                 ▼
                     ha_delete_all_rows() / instantiate_tmp_table()
                                 │
                 ┌───────────────┴───────────────┐
                 │ unit->is_recursive()?          │
                 └───┬───────────────────────┬───┘
                    YES                      NO
                     ▼                        ▼
          MaterializeRecursive()      for (query_block : blocks)
                     │                        │ MaterializeQueryBlock()
   ┌─────────────────┴──────────┐             │  Init → while Read → copy_funcs
   │ (1) 非递归块各跑一次        │             │  → check_unique_constraint
   │ (2) 递归块循环到收敛        │             │  → ha_write_row
   │     do {                   │             ▼
   │       last = stored_rows   │      reject_multiple_rows && stored_rows > 1
   │       for (is_recursive_ref)│         → ER_SUBQUERY_NO_1_ROW
   │         MaterializeQueryBlock│
   │     } while (stored_rows > last)
   └────────────────────────────┘
                     ▼
        table()->materialized = true → 刷新 invalidators 快照 → m_table_iterator->Init()
```

★ **递归协作的关键**（`[I1]`，`FollowTailIterator`）：

```cpp
  // Give each recursive iterator access to the stored number of rows
  for (const materialize_iterator::QueryBlock &query_block :
       m_query_blocks_to_materialize) {
    if (query_block.is_recursive_reference) {
      query_block.recursive_reader->set_stored_rows_pointer(&stored_rows);
    }
  }
```

`FollowTailIterator` 的契约（类注释）：它像 `tail -f` 一样**边写边读同一张临时表**；且 **后续 `Init()` 不把游标拨回开头**——这正是递归能推进的原因。落盘溢写时 `MaterializeIterator` 回调 `RepositionCursorAfterSpillToDisk()`。

#### ★ `AggregateIterator`（流式）vs `TemptableAggregateIterator`（临时表）

| | `AggregateIterator` | `TemptableAggregateIterator` |
|---|---|---|
| 前提 | 输入**已按 group key 有序**（来自排序或索引扫描），或无 GROUP BY | 输入无序 |
| `Init()` | 只重置状态机 + `m_source->Init()`；**不跑子查询** | **跑完整个子查询**：逐行 `copy_funcs` → 查组（hash 或 index）→ `ha_update_row`（已有组）/ `ha_write_row`（新组） |
| `Read()` | 真正推进聚合：`m_source->Read()` + `aggregator_add()` | 纯代理 `m_table_iterator->Read()` |
| 输出时机 | 边读边吐（第一个组读完就吐） | `Init()` 跑完后才有第一行 |
| 内存 | O(一行)——`m_first_row_this_group` / `m_first_row_next_group` 两个打包缓冲 | O(组数 × 行宽)，可能落盘 |
| 组判定 | `update_item_cache_if_changed(group_fields)` 与"上一行"比 | `ha_index_read_map(HA_READ_KEY_EXACT)` 或 `check_unique_constraint()` 查表 |

`AggregateIterator::Read()` 里**组切换**那一段是精髓（类注释给了完整例子）：

```cpp
        int first_changed_idx =
            update_item_cache_if_changed(m_join->group_fields);
        if (first_changed_idx >= 0) {
          // The group changed. Store the new row (we can't really use it yet;
          // next Read() will deal with it), then load back the group values
          // so that we can output a row for the current group.
          StoreFromTableBuffers(m_tables, &m_first_row_next_group);
          LoadIntoTableBuffers(m_tables, pointer_cast<const uchar *>(
                                             m_first_row_this_group.ptr()));
          ...
          m_state = LAST_ROW_STARTED_NEW_GROUP;
          return 0;
        }
```

⇒ 它**不知道一组何时结束，直到看到下一组的第一行**，所以必须"把下一组的首行暂存起来，再把本组首行恢复回 record buffer"才能正确输出聚合值。

#### `StreamingIterator`：不真正物化的"物化"

```cpp
// [I2] sql/iterators/composite_iterators.h
/**
  StreamingIterator is a minimal version of MaterializeIterator that does not
  actually materialize; instead, every Read() just forwards the call to the
  subquery iterator and does the required copying from one set of fields to
  another.

  It is used for when the optimizer would normally set up a materialization,
  but you don't actually need one, ie. you don't want to read the rows multiple
  times after writing them, and you don't want to access them by index (only
  a single table scan).
 */
```

```cpp
int StreamingIterator::Read() {
  m_join->set_ref_item_slice(m_input_slice);
  auto switch_to_output_slice = create_scope_guard([&] { ... });

  int error = m_subquery_iterator->Read();
  if (error != 0) return error;

  if (copy_funcs(m_temp_table_param, thd())) return 1;   // ★ 只做字段拷贝

  if (m_provide_rowid) {
    memcpy(table()->file->ref, &m_row_number, sizeof(m_row_number));
    ++m_row_number;
  }
  return 0;
}
```

⇒ 与 `MaterializeIterator` 对比很能说明火山模型：**同一个 `Read()` 契约，两种代价**。流式版每行都要重新执行子查询；物化版只在 `Init()` 里执行一次。

**`Init()` / `Read()` 职责分工总结**（这三个迭代器同构）：

```
   MaterializeIterator::Init()  → 跑完所有子查询，写满临时表
   MaterializeIterator::Read()  → 切 slice + return m_table_iterator->Read()

   ★ 所以 CTE / 派生表不管被引用多少次，子查询只跑一遍（除非 rematerialize）。
```
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

#### ★ `needs_buffering` 决定走哪个迭代器

> 源码坐标：`[I4] sql/iterators/window_iterators.h/.cc` · `[I15] sql/window.h`（`Window::needs_buffering()`、`reset_round()`、`check_partition_boundary()`）· `[I9] sql/join_optimizer/access_path.h`（`AccessPath::WINDOW` 参数）
> **与 08 篇分工**：`NewWindowAccessPath()` 如何按 `needs_buffering` 造出 `AccessPath::WINDOW` 属于 [`08_access_path.md`](08_access_path.md)。

```cpp
// [I15] sql/window.h
  /**
    If we cannot compute one of window functions without looking at succeeding
    rows, return true, else false.
  */
  bool needs_buffering() const { return m_needs_frame_buffering; }
```

两个迭代器在**构造函数里就断言**了各自的前提——这是"二选一"最直接的证据：

```cpp
// [I4] sql/iterators/window_iterators.cc
WindowIterator::WindowIterator(...)
    : RowIterator(thd), m_source(std::move(source)), ... {
  assert(!m_window->needs_buffering());          // ★ 无缓冲
}

BufferingWindowIterator::BufferingWindowIterator(...)
    : RowIterator(thd), m_source(std::move(source)), ... {
  assert(m_window->needs_buffering());           // ★ 需缓冲
}
```

```
   needs_buffering == false（窗口函数只需"当前行及其之前"）
       → WindowIterator：Read() 一行进、一行出，O(1) 内存
       典型：ROW_NUMBER() / RANK() / SUM() OVER (ROWS UNBOUNDED PRECEDING)
             / 任何 frame 上界不超过当前行的

   needs_buffering == true（需要看到后面的行才能算）
       → BufferingWindowIterator：Read() 要把行攒进 frame buffer，攒够才吐
       典型：SUM() OVER (ROWS 2 FOLLOWING)、LAST_VALUE()、
             RANGE 帧（要看到 peer 组结束才知道边界）
```

#### `WindowIterator`：`Read()` 是三步 copy + 一次分区检查

```cpp
// [I4] sql/iterators/window_iterators.cc
bool WindowIterator::Init() {
  if (m_source->Init()) return true;
  m_window->reset_round();                       // ★ 重置窗口函数的"轮"状态
  m_input_slice = m_join->get_ref_item_slice();  // 记下输入 slice
  return false;
}

int WindowIterator::Read() {
  SwitchSlice(m_join, m_input_slice);
  int err = m_source->Read();
  SwitchSlice(m_join, m_output_slice);
  if (err != 0) return err;

  if (copy_funcs(m_temp_table_param, thd(), CFT_HAS_NO_WF)) return 1;   // ①
  m_window->check_partition_boundary();                                 // ②
  if (copy_funcs(m_temp_table_param, thd(), CFT_WF)) return 1;          // ③

  // ④ 只有最后一个窗口才算"依赖窗口函数的表达式"
  if (m_window->is_last() && copy_funcs(m_temp_table_param, thd(), CFT_HAS_WF))
    return 1;

  return 0;
}
```

★ 四步顺序不能乱：

1. **`SwitchSlice(m_input_slice)` → 读源行**：源表达式必须在输入 slice 下求值；读完后立刻切到输出 slice。
2. **`copy_funcs(CFT_HAS_NO_WF)`**：先算**不依赖窗口函数**的表达式（如 `1 + t.a`），把值存进输出表的字段。
3. **`check_partition_boundary()`**：**分区推进**——发现新分区就 `reset_partition_state()`，把 `ROW_NUMBER()` 等归零。这就是"窗口帧随行推进"在无需缓冲时的全部实现：无需 buffer，因为 frame 上界不超过当前行。
4. **`copy_funcs(CFT_WF)`**：算窗口函数本体。最后 `CFT_HAS_WF` 只在 `m_window->is_last()` 时算——多窗口时必须等所有窗口都算完才能算依赖它们的表达式（如 `foo() OVER w1 + bar() OVER w2` 里的 `+`）。

`UnlockRow()` 是空实现并注明 "There's nothing we can do here"——窗口计算已经消费了行，无法安全回退锁。

#### `BufferingWindowIterator`：攒够才吐

```cpp
// [I4] sql/iterators/window_iterators.h（节选）
class BufferingWindowIterator final : public RowIterator {
 public:
  bool Init() override;
  int Read() override;
 private:
  int ReadBufferedRow(bool new_partition_or_eof);
  /// If true, we may have more buffered rows to process that need to be
  /// checked for before reading more rows from the source.
  bool m_possibly_buffered_rows;
  /// Whether the last input row started a new partition, and was tucked away
  /// to finalize the previous partition; if so, we need to bring it back
  /// for processing before we read more rows.
  bool m_last_input_row_started_new_partition;
  bool m_eof;
};
```

```cpp
// [I4] sql/iterators/window_iterators.cc
bool BufferingWindowIterator::Init() {
  if (m_source->Init()) return true;
  m_window->reset_round();
  m_possibly_buffered_rows = false;
  m_last_input_row_started_new_partition = false;
  m_eof = false;
  m_input_slice = m_join->get_ref_item_slice();
  return false;
}
```

★ **`Read()` 的循环语义**（类注释原文要点）：

```
   1. 先把行缓冲进 frame buffer（Window::m_frame_buffer）
   2. 调 process_buffered_windowing_record()
   3. 若 output_row_ready → return 0（吐一行）
      否则 → 继续从源读更多行，推迟求值与返回
   4. 读完一个完整分区（或 EOF）后，把剩余行全部求值输出

   ★ 关键细节：为了判断"上一行确实是本分区的最后一行"，必须多读一行
     来探测分区边界；而那一行在求值时会被反复覆盖，所以处理完当前分区后
     必须重新建立"下一分区第一行"的状态 —— 这就是
     m_last_input_row_started_new_partition 的用途。
```

三个 tmp table 的角色（类注释）：

```
   - 输入表：父迭代器产出，已按 PARTITION BY + ORDER BY 排序
   - 输出表：temp_table_param 指定，写入本步求值后的记录
             （若无最终 ORDER BY / DISTINCT，这次写入可被优化掉）
   - frame buffer：仅 buffering 时需要，由 Window::m_frame_buffer 持有
```

以 `SUM(A+FLOOR(B)) OVER (ROWS 2 FOLLOWING)` 为例的求值顺序：`copy_fields` 拷贝字段到 OUT → `copy_funcs` 算 `FLOOR(B)` 并存进 OUT 的字段 → 把 OUT 的行**缓冲进 frame buffer** → 攒到 frame 完整才求值 SUM。
### 2.10 修改类

**UpdateRowsIterator / DeleteRowsIterator**（`sql_update.cc:2864` / `sql_delete.cc:1186`）—— **两阶段更新**（这正是 [10_dml.md](10_dml.md) "buffer row id 两阶段读"的驱动方）：

1. 扫 join 结果：能"立即更新"的表直接改；不能立即更新的表只把**行 ID（主键）**写进临时文件/缓冲（每表一个 `Unique`，去重 + 可溢出到磁盘）
2. join 扫完后 `DoDelayedUpdates()`/`DoDelayedDeletes()`：对每个缓冲的 row ID 调 `ha_rnd_pos()` 回表再修改
3. 即使出错，只要**不能安全回滚**（改过非事务表）也要把 delayed 更新做完

注意 `Read()` 返回 **-1**——它把整个 join 结果在一次 `Read()` 里消费完。

#### ★ 共同契约：包在源迭代器外面的"修改器"

> 源码坐标：`[I5] sql/iterators/delete_rows_iterator.h`（实现在 `[I17] sql/sql_delete.cc`）· `[I6] sql/iterators/update_rows_iterator.h`（实现在 `[I16] sql/sql_update.cc`）· `[I8] sql/sql_executor.cc` · `[I3] sql/join_optimizer/access_path.{h,cc}`
> ⚠️ 8.0.39 **没有** `sql/iterators/delete_rows_iterator.cc` / `update_rows_iterator.cc`——两个类的实现分别放在 `sql_delete.cc` 与 `sql_update.cc`。

两个类都是**单子节点的装饰者**，挂在 join 根迭代器之上：不产生行、不消费行，**只产生副作用**。`Read()` 仍遵循 0 / -1 / 1 契约，但**实际上永远不会返回 0**——一次 `Read()` 把整个 join 结果消费完，然后返回 -1（成功）或 1（出错）。

```cpp
// [I5] sql/iterators/delete_rows_iterator.h
/// An iterator that deletes all rows returned by its child iterator.
class DeleteRowsIterator final : public RowIterator {
 public:
  DeleteRowsIterator(THD *thd, unique_ptr_destroy_only<RowIterator> source,
                     JOIN *join, table_map tables_to_delete_from,
                     table_map immediate_tables);
  bool Init() override;
  int Read() override;
  // ★ 自身是"一次性"节点，没有 per-row 批次语义，PFS 批次直接透传给源
  void StartPSIBatchMode() override { m_source->StartPSIBatchMode(); }
  void EndPSIBatchModeIfStarted() override { m_source->EndPSIBatchModeIfStarted(); }
  // ★ 修改类节点既不接受 NULL 行补齐，也不自己持行锁 —— 走到这里就是 bug
  void SetNullRowFlag(bool /*is_null_row*/) override { assert(false); }
  void UnlockRow() override { assert(false); }
 private:
  unique_ptr_destroy_only<RowIterator> m_source;
  JOIN *m_join;
  table_map m_tables_to_delete_from;
  /// The tables to delete from immediately while scanning the join result.
  table_map m_immediate_tables;
  /// All the tables that are part of a hash join. For those tables the row ID
  /// will already be available in handler::ref, and calling handler::position()
  /// will overwrite it with an incorrect row ID.
  table_map m_hash_join_tables;
  table_map m_transactional_tables{0};
  table_map m_tables_with_before_triggers{0};
  table_map m_tables_with_after_triggers{0};
  /// Temporary files holding row IDs to delete after the scan is complete.
  Mem_root_array<unique_ptr_destroy_only<Unique>> m_tempfiles;
  Mem_root_array<TABLE *> m_delayed_tables;
  ha_rows m_deleted_rows{0};
  bool m_has_delayed_deletes{false};

  bool DoImmediateDeletesAndBufferRowIds();
  bool DoDelayedDeletes();
  bool DoDelayedDeletesFromTable(TABLE *table);
};
```

**★ 建树顺序的特殊性——本节唯一"反火山"的地方**

`CreateIteratorForAccessPath()` 在子迭代器**还没创建**时先做全局 setup，然后才展开子节点：

```cpp
case AccessPath::DELETE_ROWS: {
  const auto &param = path->delete_rows();
  if (job.children.is_null()) {
    // Setting up tables for delete must be done before the child
    // iterators are created, as some of the child iterators need to see
    // the final read set when they are constructed, so doing it in
    // DeleteRowsIterator's constructor or Init() is too late.
    SetUpTablesForDelete(thd, join);          // ★ 改 read set，必须在建树前
    SetupJobsForChildren(mem_root, param.child, join,
                         eligible_for_batch_mode, &job, &todo);
    continue;                                 // 子建完后再回来 new DeleteRowsIterator
  }
  iterator = NewIterator<DeleteRowsIterator>(
      thd, mem_root, std::move(job.children[0]), join,
      param.tables_to_delete_from, param.immediate_tables);
  break;
}
case AccessPath::UPDATE_ROWS: {
  const auto &param = path->update_rows();
  if (job.children.is_null()) {
    // Do the final setup for UPDATE before the child iterators are created.
    if (FinalizeOptimizationForUpdate(join)) return nullptr;   // ★ 同理
    SetupJobsForChildren(mem_root, param.child, join,
                         eligible_for_batch_mode, &job, &todo);
    continue;
  }
  iterator = CreateUpdateRowsIterator(thd, mem_root, join,
                                      std::move(job.children[0]));
  break;
}
```

⇒ **原因**：扫描类子迭代器（`TableScanIterator` / `IndexScanIterator`）在**构造时**就依据 `TABLE::read_set` 决定是否走 covering index、是否回表。而 setup 会往 read set 追加删除/更新必需的列（主键、binlog 前镜像列等）——若放在本迭代器的构造或 `Init()` 里，子迭代器已按旧 read set 建好了，行为就错了。

```cpp
// [I17] sql/sql_delete.cc
void SetUpTablesForDelete(THD *thd, JOIN *join) {
  for (const Table_ref *tr = join->query_block->leaf_tables; tr != nullptr;
       tr = tr->next_leaf) {
    if (!tr->is_deleted()) continue;
    TABLE *table = tr->table;
    table->covering_keys.clear_all();   // ★ 不再允许 covering index 扫描
    table->prepare_for_position();      // ★ 需要 handler::position() 拿 rowid
    table->mark_columns_needed_for_delete(thd);  // ★ 把删除所需列加进 read set
  }
  THD_STAGE_INFO(thd, stage_deleting_from_main_table);
}
```

#### ★ `m_immediate_tables`：边读边删 vs 延迟删

`GetImmediateDeleteTables()` 的判据——本质是**规避 Halloween problem**（不能在还要继续读的表上边读边删）：

```cpp
// 若被删的表同时在查询里被引用，一律延迟 —— 删了会干扰对它的读取
for (Table_ref *tr = join->query_block->leaf_tables; tr != nullptr; tr = tr->next_leaf) {
  if (!tr->is_deleted()) continue;
  if (unique_table(tr, join->tables_list, false) != nullptr) return 0;
}
table_map candidate_tables = join->const_table_map;      // 1) const 表
if (join->primary_tables > join->const_tables) {
  const Table_ref *first_non_const = ...;
  candidate_tables |= first_non_const->map();            // 2) 第一张非 const 表
}
return delete_tables & candidate_tables;
```

⇒ **"立即删除"只允许发生在 const 表或 join 顺序的第一张非 const 表上**；其余目标表一律延迟。EXPLAIN 里的 `(immediate)` / `(buffered)` 标注就是这个位图的直接呈现。

#### `DeleteRowsIterator::Init()`：一行都不删

```cpp
// [I17] sql/sql_delete.cc
bool DeleteRowsIterator::Init() {
  if (CheckSqlSafeUpdate(thd(), m_join)) return true;   // sql_safe_updates 下拒绝无 key 全表删
  if (m_source->Init()) return true;                    // ★ 唯一硬职责之一

  for (Table_ref *tr = m_join->query_block->leaf_tables; tr != nullptr;
       tr = tr->next_leaf) {
    TABLE *const table = tr->table;
    const uint tableno = tr->tableno();
    if (!IsBitSet(tableno, m_tables_to_delete_from)) continue;

    if (IsBitSet(tableno, m_tables_with_after_triggers)) {
      // AFTER DELETE 触发器可能访问本表，因此必须立即删、不能攒批
      (void)table->file->ha_extra(HA_EXTRA_DELETE_CANNOT_BATCH);
    }
    if (thd()->lex->is_ignore()) table->file->ha_extra(HA_EXTRA_IGNORE_DUP_KEY);
    if (thd()->is_error()) return true;

    // ★ 每个"非 immediate"目标表配一个 Unique：按 rowid 排序 + 去重 + 可溢盘
    if (!IsBitSet(tableno, m_immediate_tables)) {
      auto tempfile = make_unique_destroy_only<Unique>(
          thd()->mem_root, refpos_order_cmp, table->file,
          table->file->ref_length, thd()->variables.sortbuff_size);
      if (tempfile == nullptr || m_tempfiles.push_back(std::move(tempfile)) ||
          m_delayed_tables.push_back(table)) return true;
    }
  }
  assert(!thd()->is_error());
  return false;
}
```

#### ★ 每读到一行做什么

```cpp
for (Table_ref *tr = m_join->query_block->leaf_tables; tr != nullptr; tr = tr->next_leaf) {
  const table_map map = tr->map();
  if (!Overlaps(map, m_tables_to_delete_from)) continue;      // 不是目标表
  const bool immediate = Overlaps(map, m_immediate_tables);   // ★ 唯一的分流开关
  TABLE *const table = tr->table;
  Unique *const tempfile = immediate ? nullptr : m_tempfiles[unique_counter++].get();

  // 外连接没匹配到（NULL 行）或同一行上轮已删 → 跳过
  if (table->has_null_row() || table->has_deleted_row()) continue;

  // ★ hash join 表的 rowid 已在 file->ref 里，再调 position() 会覆盖成"最后读到的行"
  if (!Overlaps(map, m_hash_join_tables)) {
    table->file->position(table->record[0]);
  }

  if (immediate) {
    const ha_rows last_deleted = m_deleted_rows;
    table->set_deleted_row();
    if (DeleteCurrentRowAndProcessTriggers(            // ★ 当场真删
            thd(), table, Overlaps(map, m_tables_with_before_triggers),
            Overlaps(map, m_tables_with_after_triggers), &m_deleted_rows))
      return true;
    if (last_deleted != m_deleted_rows &&
        !Overlaps(map, m_transactional_tables)) {
      thd()->get_transaction()->mark_modified_non_trans_table(Transaction_ctx::STMT);
    }
  } else {
    m_has_delayed_deletes = true;                      // ★ 只记 rowid
    if (tempfile->unique_add(table->file->ref)) return true;
  }
}
```

#### ★ `Read()`：一次调用跑完整条 DELETE

```cpp
// [I17] sql/sql_delete.cc
int DeleteRowsIterator::Read() {
  bool local_error = false;

  // ① 先把 join 吐出的所有行处理完：immediate 表当场删，其余缓冲 rowid
  while (true) {
    if (const int read_error = m_source->Read();
        read_error > 0 || thd()->is_error()) { local_error = true; break; }
    else if (read_error < 0) break;                       // EOF
    else if (thd()->killed) { thd()->send_kill_message(); return 1; }
    else {
      if (DoImmediateDeletesAndBufferRowIds()) { local_error = true; break; }
    }
  }

  // ② 若缓冲了 delayed 删除：即使出错，只要改过非事务表也要删完
  //    （理由：已做的删除回滚不了，删掉其余表的对应行能让各表状态更一致）
  if (m_has_delayed_deletes &&
      (!local_error || thd()->get_transaction()->has_modified_non_trans_table(
                           Transaction_ctx::STMT))) {
    THD_STAGE_INFO(thd(), stage_deleting_from_reference_tables);
    if (DoDelayedDeletes()) local_error = true;
  }

  const THD::killed_state killed_status =
      !local_error ? THD::NOT_KILLED : thd()->killed.load();

  if (!local_error ||
      thd()->get_transaction()->cannot_safely_rollback(Transaction_ctx::STMT)) {
    if (mysql_bin_log.is_open()) {
      int errcode = 0;
      if (!local_error) thd()->clear_error();
      else errcode = query_error_code(thd(), killed_status == THD::NOT_KILLED);
      thd()->thread_specific_used = true;
      // ★ 只有"全部目标表都是事务表"时 binlog 写失败才整体回滚
      if (thd()->binlog_query(THD::ROW_QUERY_TYPE, thd()->query().str,
                              thd()->query().length, m_transactional_tables != 0,
                              false, false, errcode) &&
          m_transactional_tables == m_tables_to_delete_from) {
        local_error = true;
      }
    }
  }

  if (local_error) return 1;
  else {
    thd()->set_row_count_func(m_deleted_rows);   // "Query OK, N rows affected"
    return -1;
  }
}
```

**为什么必须读完才返回 -1**（三个理由）：

1. delayed 表的 rowid 全集只有扫完才知道，且 `Unique` 必须先**排序去重**才能按物理顺序高效回表；
2. 中途出错但已改过非事务表（`cannot_safely_rollback`）时，仍要把 delayed 删完以求各表状态一致；
3. binlog 是**语句级**的，必须在所有删除动作结束后才能定 errcode 并写一次。

#### `UpdateRowsIterator`：`FinalizeOptimizationForUpdate()` 与 immediate 判定

```cpp
// [I16] sql/sql_update.cc —— 只是转发到 Query_result_update::optimize()
bool FinalizeOptimizationForUpdate(JOIN *join) {
  return down_cast<Query_result_update *>(join->query_block->query_result())
      ->optimize();
}
```

它做的事同样是"必须在子迭代器创建前"：同步 `update_table->table`、清 `covering_keys`、AFTER UPDATE trigger 时 `HA_EXTRA_UPDATE_CANNOT_BATCH`、`CheckSqlSafeUpdate()`，然后**从 `AccessPath::UPDATE_ROWS::immediate_tables` 反解出"立即更新表"**：

```cpp
AccessPath *root_path = join->root_access_path();
assert(root_path->type == AccessPath::UPDATE_ROWS);
const table_map immediate_update_tables = root_path->update_rows().immediate_tables;
if (immediate_update_tables != 0) {
  assert(immediate_update_tables == main_table->pos_in_table_list->map());
  table_to_update = main_table;      // ★ 非空 ⇒ "边读边改"的那张表
} else {
  table_to_update = nullptr;
}
```

⇒ **UPDATE 的 immediate 最多只有一张表**（最外层表）：`GetImmediateUpdateTable()` 只考察 `join->qep_tab[0]` 且它必须 `is_updated()`；多目标表时先用 `CollectColumnsReferencedInJoinConditions()` 把"影响后续表行选择的列"收进 `tmp_set`，再用 `safe_update_on_fly()` 判定——**若被更新的字段会影响后面表的行选择，则不能 on-the-fly**。

`Init()` 极简（唯一副作用是开半一致读）：

```cpp
// [I16] sql/sql_update.cc
bool UpdateRowsIterator::Init() {
  if (m_source->Init()) return true;        // ★ 唯一硬职责
  if (m_outermost_table != nullptr &&
      !Overlaps(m_hash_join_tables, m_outermost_table->pos_in_table_list->map())) {
    // 最外层表是 nested loop 的左参数，可以请求半一致读
    m_outermost_table->file->try_semi_consistent_read(true);
  }
  return false;
}
UpdateRowsIterator::~UpdateRowsIterator() {   // 配对关闭
  if (m_outermost_table != nullptr && m_outermost_table->is_created()) {
    m_outermost_table->file->try_semi_consistent_read(false);
  }
}
```

`Read()` 与 DELETE 逐行对称（同样 **从不返回 0**），收尾三件套一致：`DoDelayedUpdates()` → 关掉半一致读 → 写一次语句级 binlog → 返回 -1/1。结果由 `Query_result_update::send_eof()` 取回 `found_rows()` / `updated_rows()`，这就是 `Rows matched: N  Changed: M`。

★ **为什么"一行的更新可能在扫描结束后才执行"**：immediate 最多一张表，其余所有目标表只被缓冲成 `(rowid, 已求值的新值)` 对。注意临时表里存的是**已求值的新值**而不是 Item 树——既避免扫描过程中表内容变化导致 Item 求值不确定（Halloween），也让 delayed 阶段不需要重新求值。

#### 两者对照

| | `DeleteRowsIterator` | `UpdateRowsIterator` |
|---|---|---|
| 目标集合 | `table_map m_tables_to_delete_from` | `Table_ref *m_update_tables` 链 |
| immediate 判据 | `table_map m_immediate_tables`（可多表：const ∪ 首张非 const） | `TABLE *m_immediate_table`（**最多一张**） |
| 延迟缓冲 | 每表一个 `Unique`（只存 rowid，可溢盘） | 每表一个临时表（rowid + **已求值的新值**） |
| 每行动作 | `DeleteCurrentRowAndProcessTriggers()` | immediate: `fill_record_n_invoke_before_triggers()` + `ha_update_row()`；deferred: `StoreRowId()` + `fill_record()` + `ha_write_row()` |
| 收尾 | `DoDelayedDeletes()` → `Unique::get()` → 回表删 | `DoDelayedUpdates()` → 扫临时表 → `PositionScanOnRow()` → `copy_field->invoke_do_copy()` → `ha_update_row()` |
| `Read()` 返回 | -1 / 1 | -1 / 1 |

**三条共同不变式**：① `Init()` 只做"准备"，**任何一行都不改**；② `Read()` 内部自己把源迭代器读到 EOF，一次调用完成整条语句；③ 即使中途出错，只要 `cannot_safely_rollback(STMT)`（改过非事务表），也要把 delayed 阶段跑完，最后再**写一次**语句级 binlog。

> 另注：**EXPLAIN ANALYZE** 下 `DoImmediateDeletesAndBufferRowIds()` / `DoImmediateUpdatesAndBufferRowIds()` 都在开头直接 `return false`，不产生任何副作用。
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

## 三、从 AccessPath 到 Iterator：翻译入口

AccessPath 是"计划"（数据），RowIterator 是"执行"（行为），两者近乎 **1:1**，由 `CreateIteratorFromAccessPath` 桥接：

| AccessPath | RowIterator |
|---|---|
| `TABLE_SCAN` / `INDEX_SCAN` | `TableScanIterator` / `IndexScanIterator` |
| `REF` / `REF_OR_NULL` / `EQ_REF` | `RefIterator` / `RefOrNullIterator` / `EQRefIterator` |
| `INDEX_RANGE_SCAN` | `IndexRangeScanIterator` |
| `NESTED_LOOP_JOIN` | `NestedLoopIterator` |
| `HASH_JOIN` | `HashJoinIterator` |
| `SORT` | `SortingIterator` |
| `FILTER` | `FilterIterator` |
| `MATERIALIZE` | `MaterializeIterator` |
| `WINDOW` | `WindowIterator`（缓冲/非缓冲两型） |
| ... | ... |

> ★ **翻译机制本身详见 [`08_access_path.md`](08_access_path.md)**：显式栈（为何不用递归）、两阶段、`IteratorToBeCreated`、各 AccessPath 类型的 switch 分支要点、batch mode 传递、惰性实例化、MATERIALIZE 的特殊处理、翻译失败路径。**本篇不重复**，只保留"计划↔执行 1:1"这个对照关系与下面两点。

两点与执行模型强相关、放在本篇讲：

**① 翻译发生在 optimize 末尾，不是优化期。** 计划（AccessPath）在优化期完整生成，执行树（RowIterator）到 `Query_expression::optimize` 才实例化——这就是"计划树一次性生成、执行树按需实例化"的分界。

**② 翻译可以被整体跳过。** `IteratorsAreNeeded()`：次级引擎若声明 `USE_EXTERNAL_EXECUTOR`（整体 offload，如 HeatWave），则**完全不建迭代器树**——连 `FinalizePlanForQueryBlock()` 都不会改计划。

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


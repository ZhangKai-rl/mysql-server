# PFS 事件表族：waits / stages / transactions / errors

> **边界**：本篇讲 Performance Schema 的**事件表族**（四类事件各自的表、字段、埋点、开关）。
> - **PFS 框架本身**（PSI 接口、探针、设计模式）见 [`pfs.md`](pfs.md)，此处不重复
> - **statement 一类事件**见 [`pfs_statement.md`](pfs_statement.md)，此处只在对比时提及
> - 观测手段全景与选型决策树见 [`../README.md`](../README.md)

## 目录

- [一、概述](#一概述)
- [二、理论基础：两级开关与三种聚合模型](#二理论基础两级开关与三种聚合模型)
- [三、waits 事件](#三waits-事件)
- [四、stages 事件](#四stages-事件)
- [五、transactions 事件](#五transactions-事件)
- [六、★ errors 事件](#六-errors-事件全库此前完全没写过)
- [七、★ memory 事件](#七-memory-事件pfs-的内存计量)（★ 唯一能定位"内存是谁分配的"；★ 字段分累计/当前/水位三组；★ 默认**开**）
- [八、不属于 `events_*` 的杂项表](#八pfs-里不属于-events_-的杂项表易漏的观测点)（`log_status` / `prepared_statements_instances` / `session_connect_attrs` 等 10 张）
- [九、可观测性](#九可观测性)
- [十、★ 工程实现技法](#十-工程实现技法)
- [十一、Misc（易混淆点）](#十一misc易混淆点)
- [参考](#参考)

---

## 一、概述

### 是什么

PFS 的事件表是**内存表**（表定义里写死 `" ENGINE=PERFORMANCE_SCHEMA"`），由埋点在代码执行时写入，重启即清空。四类事件对应四种"可观测的时间切片"：

| 事件 | 观测什么 | 典型问题 |
|---|---|---|
| **waits** | 一次等待（锁、IO、网络、空闲） | "时间花在等什么上" |
| **stages** | 一个执行阶段（含**进度**） | "现在走到哪一步、做了多少" |
| **transactions** | 一个事务（含 GTID / XA） | "事务状态、读写、隔离级别" |
| **errors** | 服务端错误（按错误号聚合） | "哪些错误在发生、有没有被处理" |

### 用途

- waits/stages/transactions 都有 **current（当前）+ history/history_long（历史环形缓冲）+ summary（聚合）**三层
- **errors 只有 summary**（源码明确："For errors, there are no tables that contains individual event data"）

### 版本演进（源码级）

- **8.0.4**：`setup_timers` 表**被移除**（`storage/perfschema/pfs_dd_version.h` 明确记录 `setup_timers (removed)`，`table_setup_timers.cc` 文件已不存在）⇒ **计时器不再可配置**
- 同批变更：`setup_threads` 新增、`setup_instruments` / `variables_info` 修改
- errors 事件：**源码无直接版本证据**（`pfs_error.cc` 版权起始 2016，但 DD schema 变更历史中从未记载 `events_errors_*`，无 WL 号注释）⇒ 不编造具体版本号

---

## 二、理论基础：两级开关与三种聚合模型

### 2.1 ★ 为什么要 instrument + consumer 两级开关

这是 PFS 最核心的设计权衡。源码给出的理由（`storage/perfschema/pfs.cc` `@subsection PRINCIPLE_PERFORMANCE No performance hit`）：

```
  The instrumentation of the server should be as fast as possible.
  In cases when there are choices between:
  - doing some processing when recording the performance data
    in the instrumentation,
  - doing some processing when retrieving the performance data,
  priority is given in the design to make the instrumentation faster,
  pushing some complexity to data retrieval.
```

即：**优先优化"采集"这条关键路径，把复杂度推给"读取"侧**。

两级各自控制：

| 级别 | 由谁控制 | 粒度 | 关掉后的效果 |
|---|---|---|---|
| **生产端** | `setup_instruments.ENABLED`（`PFS_instr_class::m_enabled`） | **每个 instrument 一个开关**（如 `wait/synch/mutex/sql/LOCK_open`） | **完全不调用任何 PFS 代码**——连 `start_*_wait` 都不进 |
| **消费端** | `setup_consumers.ENABLED`（`flag_events_waits_current` / `_history` / `_history_long`） | **每个表族一个开关** | 仍会调用 PFS、仍聚合到 summary，但**不往 current/history 写行** |

生产端判断在**调用者代码里**（`include/mysql/psi/mysql_mutex.h`）：

```c
  if (that->m_psi != nullptr) {
    if (that->m_psi->m_enabled) {
      /* Instrumentation start */
      locker = PSI_MUTEX_CALL(start_mutex_wait)(...);
      /* Instrumented code */
      result = my_mutex_lock(...);
```

消费端判断在 **PFS 实现里**（`pfs.cc` `pfs_start_mutex_wait_v1()`）：

```c
  /* The caller checks for m_enabled. */
#if 0
  if (!pfs_mutex->m_enabled) { return nullptr; }
#endif
```

（这段被 `#if 0` 注释掉，正是"生产端已经查过了，消费端不再重复查"的证据。）

⇒ **两者是"与"关系**。这解释了一个常见困惑：**为什么表存在却查不到行**——多半是 consumer 没开，而不是 instrument 没开（`pfs.md` 与上层 README 已反复强调这点）。

### 2.2 ★ 三种聚合模型（性能权衡的核心）

`pfs.cc` 定义了三种，并明确说了各自的取舍：

| 模型 | 谁做计算 | 开销 | 源码评价 |
|---|---|---|---|
| **On the fly** | **写线程** | 大 | `This model is to be avoided if possible, due to the overhead caused when instrumenting code.` |
| **Higher level** | **读线程** | 写线程零开销 | 只在 SELECT 时从多个内部 buffer 收集 |
| **Mixed** | 读线程做复杂计算；写线程只在 destroy 时有极小开销 | 折中 | 对象存在时用 higher level，销毁时上浮到父 buffer（on the fly） |
```

  @section MIXED Mixed level aggregates
  The 'Mixed' model is a compromise between 'On the fly' and 'Higher level'
  aggregates, for internal buffers that are not permanent.
  While an object is present in a buffer, the higher level model is used.
  When an object is about to be destroyed, statistics are saved into
  a 'parent' buffer with a longer life cycle, to follow the on the fly model.
```

### 2.3 聚合的形式化定义（理论溯源）

`pfs.cc` `@page PAGE_PFS_AGGREGATES` 给出了严谨定义——这是理解 summary 表的钥匙：

```
  Aggregates tables are tables that can be formally defined as
  SELECT ... from EVENTS_WAITS_HISTORY_INFINITE ... group by 'group clause'.
  Each group clause defines a different kind of aggregate,
  and corresponds to a different table exposed by the performance schema.
  'EVENTS_WAITS_HISTORY_INFINITE' is a table that does not exist,
  the best approximation is EVENTS_WAITS_HISTORY_LONG.
```

**维度（dimension）**——事件是一个数据点 `P(x1, x2, ..., xN)`，每个坐标是一个属性：

```
  Examples of attributes are:
  - the time waited
  - the object waited on
  - the instrument waited on
  - the thread that waited
  - the operation performed

  Computing an aggregate per thread is fundamentally different from
  computing an aggregate by instrument, so the "_BY_THREAD" and
  "_BY_EVENT_NAME" aggregates are different dimensions,
  operating on different x_i and x_j coordinates.
  These aggregates are "orthogonal".
```

**投影（projection）**——每次聚合会**丢掉**依赖该维度的列：

```
  For example, when aggregating wait events by object instances,
  - the wait_time and number_of_bytes can be summed,
    and sum(wait_time) now becomes an object instance attribute.
  - the source, timer_start, timer_end columns are not in the
    _BY_INSTANCE table, because these attributes are only
    meaningful for a wait.
```

⇒ **这就是为什么 `events_waits_summary_by_instance` 没有 `SOURCE` 列**。

**复合（composition）**——多级聚合不是一次算完，而是分解后**复用中间 buffer**：

```
  Functions (or aggregates) such as F_3 are not implemented as is.
  Instead, they are decomposed into F_2_to_3 o F_1_to_2 o F1,
  and each intermediate aggregate is stored into an internal buffer.
```

---

### 2.4 ★ 计时器：为什么等待事件用 CPU cycle，最终却都是皮秒

PFS 所有 `TIMER_WAIT` 列的单位**统一是皮秒（1e-12 秒）**，但**埋点时读的并不是皮秒**。

`storage/perfschema/pfs_timer.h` 里的关键定义：

```c
/** Conversion factor, from micro seconds to pico seconds. */
/** Conversion factor, from nano seconds to pico seconds. */
...
  See my_timer_nanoseconds() in mysys/my_rdtsc.cc
  This logic matches my_timer_nanoseconds(),
  to find out at compile time if a nanosecond
...
#define USED_TIMER_NAME TIMER_NAME_NANOSEC
#define USED_TIMER my_timer_nanoseconds
...
#define USED_TIMER_NAME TIMER_NAME_MICROSEC
#define USED_TIMER my_timer_microseconds
...
ulonglong inline get_wait_timer() { return my_timer_cycles(); }
...
  to normalized values, expressed in picoseconds.
  /** Conversion factor from timer values to pico seconds. */
    Convert a wait from timer units to pico seconds.
    @return the wait, expressed in pico seconds
```

**★ 两层设计**：

| 层 | 用什么 | 为什么 |
|---|---|---|
| **埋点读取** | 等待事件用 **`my_timer_cycles()`（CPU 周期）**；其它按平台选 `my_timer_nanoseconds()` / `my_timer_microseconds()` | cycle 通常**一条 `rdtsc` 指令**就能拿到，是**最快**的时钟源 |
| **对外展示** | 用 `to_pico` 归一化成**皮秒** | 统一单位，用户/工具不需要知道底层用的是哪种时钟 |

⇒ ★ 这正是 2.1 那条设计原则的落地：**"让埋点更快，把复杂度推给读取"** —— 读 cycle 最快，转换延迟到数据被读取时才做（`time_to_pico` 乘一个系数）。

⚠️ 副作用：**cycle 与真实时间的换算依赖 CPU 频率**，在变频/多插槽机器上可能有偏差；这也是为什么可以退化为 nanoseconds / microseconds（编译期按平台选择，`#if` 分支）。

### 2.5 ★ 聚合的实现：`PFS_single_stat::aggregate()`

三种聚合模型（见 2.2）落到代码就是这几个 inline 函数（`storage/perfschema/pfs_stat.h`）：

```cpp
  inline void aggregate(const PFS_single_stat *stat) {
    if (stat->m_count != 0) {
      m_count += stat->m_count;
      m_sum += stat->m_sum;
      if (unlikely(m_min > stat->m_min)) {
        m_min = stat->m_min;
      }
      if (unlikely(m_max < stat->m_max)) {
        m_max = stat->m_max;
      }
    }
  }
```

**逐段解释**：

- **`if (stat->m_count != 0)` 先判空**：从未发生过的统计（`count==0`）不参与聚合。否则它的 `m_min` 初值（通常是 `ULONGLONG_MAX`）会**污染**聚合结果的 min。这是一个必须有的守卫。
- **`m_count +=` / `m_sum +=`**：计数与求和直接累加 —— 这是"**加权平均**"的基础（所以 sys 里 `SUM(sum_timer_wait)/SUM(count_star)` 才是正确的总体平均，见 `sys_schema.md` 4.3 ⑨）。
- **★ `unlikely(...)` 两个分支**：min/max 更新是**罕见路径**（大多数情况下新值落在已有区间内），用 `unlikely()` 提示编译器把这两段放到冷路径，**优化热路径的指令 cache 与分支预测**。这是 PFS 里非常典型的微观优化。
- **`m_min > stat->m_min` 才更新**：取更小的；`m_max < stat->m_max` 取更大的。配合正确的初值（min 初值取最大值、max 初值取 0），第一次聚合就能正确收敛。

**★ 为什么这值得看**：这段十几行的代码就是 PFS 全部 `*_summary_*` 表的数据来源。理解了它，就能理解：
- 为什么**聚合表的 SUM/COUNT/MIN/MAX 是"预聚合"好的**（查询时不再扫描原始事件，代价极低）
- 为什么 **`min` 在某些表上会显示成巨大的数**（`ULONGLONG_MAX` 未被正确初始化/`count==0` 的边界）
- 为什么**历史表（`_history`）是明细**而**汇总表（`_summary`）只有这 4 类值**——因为聚合模型就是这么定的

### 2.6 一个对照：字节统计与"水位"

memory 事件（见第七章）用的是另一套统计（`PFS_memory_stat`），除了 count/sum 还多了**水位（high/low）**：

```
CURRENT_COUNT_USED = COUNT_ALLOC − COUNT_FREE
CURRENT_NUMBER_OF_BYTES_USED = SUM_ALLOC − SUM_FREE
HIGH_NUMBER_OF_BYTES_USED   ← 历史峰值（★ 不因回落而消失）
```

⇒ 与 `PFS_single_stat`（只记 min/max/sum/count）的差别：**memory 需要"当前值"和"峰值"**，因为内存是"存量"而等待时间是"流量"。这也是为什么排查 OOM 要看 `HIGH_*` 而不是 `CURRENT_*`。

---

## 三、waits 事件

### 3.0 ★ PFS 表怎么读：游标 + 行构造 + 乐观锁

**这是理解 PFS 的关键**——`events_waits_current` **不是一张真实的表**：没有文件、没有行存储。你 `SELECT` 它时，PFS 是**游走内部内存结构、临时拼出一行**返回。三件事决定了它怎么工作：

#### ① `rnd_next()` —— 双层游标

```cpp
int table_events_waits_current::rnd_next() {
  PFS_thread *pfs_thread;
  PFS_events_waits *wait;
  bool has_more_thread = true;

  for (m_pos.set_at(&m_next_pos); has_more_thread; m_pos.next_thread()) {
    pfs_thread = global_thread_container.get(m_pos.m_index_1, &has_more_thread);
    if (pfs_thread != nullptr) {
      wait = get_wait(pfs_thread, m_pos.m_index_2);
      if (wait != nullptr) {
        /* Next iteration, look for the next locker in this thread */
        m_next_pos.set_after(&m_pos);
        return make_row(pfs_thread, wait);
      }
    }
  }

  return HA_ERR_END_OF_FILE;
}
```

**逐段解释**：

- **`m_pos` 是二维游标**：`m_index_1` = 线程在 `global_thread_container` 中的下标，`m_index_2` = 该线程**等待栈**的深度（`WAIT_STACK_SIZE`）。所以一次全表扫描是"遍历所有线程 × 每个线程的等待栈"。
- **★ 为什么是等待栈**：一个线程可能嵌套等待（等锁 → 等 IO），所以 `events_waits_current` 一个线程可以贡献**多行**（栈顶是最新的等待）。
- **`get_wait(pfs_thread, m_index_2)` 返回 nullptr 就继续下一个线程** ⇒ 没有等待事件的线程自然被跳过（所以表行数远小于"线程数 × 栈深"）。
- **`m_next_pos.set_after(&m_pos)`**：记住"下一次从这里继续"，支持 MySQL 的分批扫描语义。

**行数上限也是算出来的**：

```cpp
ha_rows table_events_waits_current::get_row_count() {
  return WAIT_STACK_SIZE * global_thread_container.get_row_count();
}
```

⇒ 这是**估算上限**（优化器用于选择执行计划），实际行数扫描时才知道。

#### ② `make_row()` —— ★ 乐观锁防"读到一半线程没了"

```cpp
int table_events_waits_current::make_row(PFS_thread *thread,
                                         PFS_events_waits *wait) {
  pfs_optimistic_state lock;

  /* Protect this reader against a thread termination */
  thread->m_lock.begin_optimistic_lock(&lock);

  if (table_events_waits_common::make_row(wait)) {
    return HA_ERR_RECORD_DELETED;
  }

  if (!thread->m_lock.end_optimistic_lock(&lock)) {
    return HA_ERR_RECORD_DELETED;
  }

  return 0;
}
```

**逐段解释**：

- **★ 这是 PFS 最核心的并发技巧**：正在把某个线程的数据拷成一行时，**这个线程可能正在退出**（内存被回收）。用**乐观锁**（读版本号，事后校验）而不是加互斥锁——因为 PFS 的读路径必须**无锁**（否则观测行为本身会阻塞业务）。
- **`end_optimistic_lock()` 返回 false ⇒ 期间发生了写 ⇒ 直接放弃这一行**（返回 `HA_ERR_RECORD_DELETED`，MySQL 会跳过）。**宁可少一行，也不返回损坏的数据**。
- 这正是 2.2「三种聚合模型」里说的"读取时可能读到不一致数据，但绝不加锁"的具体实现。

#### ③ `read_row_values()` —— ★ 源码注释明说"这些列不可靠"

```cpp
int table_events_waits_common::read_row_values(TABLE *table, unsigned char *buf,
                                               Field **fields, bool read_all) {
  Field *f;
  const LEX_CSTRING *operation;

  /* Set the null bits */
  assert(table->s->null_bytes == 2);
  buf[0] = 0;
  buf[1] = 0;

  /*
    Some columns are unreliable, because they are joined with other buffers,
    which could have changed and been reused for something else.
    These columns are:
    - THREAD_ID (m_thread joins with PFS_thread),
    - SCHEMA_NAME (m_schema_name joins with PFS_table_share)
    - OBJECT_NAME (m_object_name joins with PFS_table_share)
  */
  ...
```

**★ 这段注释是理解 PFS 数据可信度的重要材料**：

- `events_waits_*` 里存的不是字符串副本，而是**指向其它 PFS 缓冲区的指针**（`m_thread`、`m_schema_name`、`m_object_name`）。
- 那些缓冲区是**可复用的**（对象被 drop、线程退出后会被重新分配给新对象）⇒ **读到时可能已经指向了别的东西**。
- ⇒ 所以 **`THREAD_ID` / `SCHEMA_NAME` / `OBJECT_NAME` 这三列是"尽力而为"的**，在高并发/对象频繁变更的场景下可能张冠李戴。**这是 PFS 已知的取舍**：不拷贝字符串（省内存和 CPU），代价是这几列可能不准。

⚠️ 实践含义：用 `events_waits_*` 做精确归因（"这条等待属于哪个表"）时，要意识到 `OBJECT_NAME` 有这个隐患；需要强一致时应用 `OBJECT_INSTANCE_BEGIN`（指针值，不会串）而非名字。

#### ④ 环形缓冲：history / history_long 怎么存

```cpp
struct PFS_events_waits : public PFS_events { ... };

void insert_events_waits_history(PFS_thread *thread, PFS_events_waits *wait);
void insert_events_waits_history_long(PFS_events_waits *wait);

extern PFS_events_waits *events_waits_history_long_array;
```

**解释**：

- **`events_waits_history_long_array` 是一个全局定长数组**（大小由 `performance_schema_events_waits_history_long_size` 决定），**循环覆盖写入**——写满后从头覆盖，所以查到的是"最近 N 条"，**顺序不是数组顺序**（数组里的时间是乱序的）。
- **`events_waits_history`（每线程）** 同理，但数组是 per-thread 的。
- ⇒ ★ 这解释了为什么 `latest_file_io` 之类的视图要 `ORDER BY timer_start`——**不能直接按数组顺序读**（见 `sys_schema.md` 4.5 ⑤）。
- 与 `_current` 的区别：`_current` 是**每线程的等待栈**（当前正在等什么，栈式），`_history*` 是**已结束事件的环形日志**（时间序）。

### 3.1 表族（9 张）

| 表名 | 说明 |
|---|---|
| `events_waits_current` / `_history` / `_history_long` | 逐事件；per-thread 环形缓冲 |
| `events_waits_summary_by_instance` | 按**实例地址** |
| `events_waits_summary_by_thread_by_event_name` | 按线程 × instrument |
| `events_waits_summary_by_user_by_event_name` | 按用户 |
| `events_waits_summary_by_account_by_event_name` | 按 `user@host` |
| `events_waits_summary_by_host_by_event_name` | 按来源 host |
| `events_waits_summary_global_by_event_name` | 全局 |

源码还列出相关的 file/socket 聚合表：`FILE_SUMMARY_BY_EVENT_NAME`、`FILE_SUMMARY_BY_INSTANCE`、`SOCKET_SUMMARY_BY_INSTANCE`、`SOCKET_SUMMARY_BY_EVENT_NAME`、`OBJECTS_SUMMARY_GLOBAL_BY_TYPE`。

被 instrument 的等待源（`pfs.cc`）：

```
  - mutexes (mysql_mutex_t), rwlocks, conditions,
    file I/O (MYSQL_FILE), socket I/O (MYSQL_SOCKET),
    table I/O, table lock, idle
```

### 3.2 ★ instrument 命名分层树

前缀常量集中在 `storage/perfschema/pfs_column_values.cc`（**源码常量，非推测**）：

```c
LEX_CSTRING mutex_instrument_prefix   = {STRING_WITH_LEN("wait/synch/mutex")};
LEX_CSTRING rwlock_instrument_prefix  = {STRING_WITH_LEN("wait/synch/rwlock")};
LEX_CSTRING prlock_instrument_prefix  = {STRING_WITH_LEN("wait/synch/prlock")};
LEX_CSTRING sxlock_instrument_prefix  = {STRING_WITH_LEN("wait/synch/sxlock")};
LEX_CSTRING cond_instrument_prefix    = {STRING_WITH_LEN("wait/synch/cond")};
LEX_CSTRING file_instrument_prefix    = {STRING_WITH_LEN("wait/io/file")};
LEX_CSTRING socket_instrument_prefix  = {STRING_WITH_LEN("wait/io/socket")};
LEX_CSTRING table_io_class_name       = {STRING_WITH_LEN("wait/io/table/sql/handler")};
LEX_CSTRING table_lock_class_name     = {STRING_WITH_LEN("wait/lock/table/sql/handler")};
LEX_CSTRING metadata_lock_class_name  = {STRING_WITH_LEN("wait/lock/metadata/sql/mdl")};
LEX_CSTRING idle_class_name           = {STRING_WITH_LEN("idle")};
LEX_CSTRING error_class_name          = {STRING_WITH_LEN("error")};
```

```
wait/
├── synch/
│   ├── mutex/  <component>/<name>
│   ├── rwlock/ <component>/<name>
│   ├── prlock/ <component>/<name>        (rwlock 的 pr 变体)
│   ├── sxlock/ <component>/<name>        (rwlock 的 sx 变体)
│   └── cond/   <component>/<name>
├── io/
│   ├── file/   <component>/<name>
│   ├── socket/ <component>/<name>
│   └── table/sql/handler                 (全局单例)
└── lock/
    ├── table/sql/handler                 (全局单例)
    └── metadata/sql/mdl                  (全局单例，MDL)
idle                                      (顶层，无 wait/ 前缀)
```

第 4 层是**组件名**，源码说明了它的用途（`pfs.cc` `@section COMPOSITION`）：

```
  The object class is named in such a way
  (for example, "wait/sync/mutex/sql/LOCK_open",
   "wait/io/file/maria/data_file") that the component ("sql", "maria")
   that it belongs to can be inferred.
```

⇒ 所以 `wait/synch/mutex/innodb/...` 就知道是 InnoDB 的锁；`wait/synch/mutex/sql/...` 是 server 层的。

### 3.3 字段定义与关键语义

`storage/perfschema/table_events_waits.cc`：

```c
    "  THREAD_ID BIGINT unsigned not null,\n"
    "  EVENT_ID BIGINT unsigned not null,\n"
    "  END_EVENT_ID BIGINT unsigned,\n"
    "  EVENT_NAME VARCHAR(128) not null,\n"
    "  SOURCE VARCHAR(64),\n"
    "  TIMER_START BIGINT unsigned,\n"
    "  TIMER_END BIGINT unsigned,\n"
    "  TIMER_WAIT BIGINT unsigned,\n"
    "  SPINS INTEGER unsigned,\n"
    "  OBJECT_SCHEMA VARCHAR(64),\n"
    "  OBJECT_NAME VARCHAR(512),\n"
    "  INDEX_NAME VARCHAR(64),\n"
    "  OBJECT_TYPE VARCHAR(64),\n"
    "  OBJECT_INSTANCE_BEGIN BIGINT unsigned not null,\n"
    "  NESTING_EVENT_ID BIGINT unsigned,\n"
    "  NESTING_EVENT_TYPE ENUM('TRANSACTION', 'STATEMENT', 'STAGE', 'WAIT'),\n"
    "  OPERATION VARCHAR(32) not null,\n"
    "  NUMBER_OF_BYTES BIGINT,\n"
    "  FLAGS INTEGER unsigned,\n"
    "  PRIMARY KEY (THREAD_ID, EVENT_ID)\n",
```

**TIMER_* 的单位是皮秒**，由 `time_normalizer` 换算（`pfs_timer.h`）：

```c
/** Conversion factor, from micro seconds to pico seconds. */
#define MICROSEC_TO_PICOSEC 1000000
/** Conversion factor, from nano seconds to pico seconds. */
#define NANOSEC_TO_PICOSEC 1000
```

未结束的事件会用当前时间补 `TIMER_END`（`table_events_waits.cc`）：

```c
  if (m_row.m_end_event_id == 0) {
    if (wait->m_wait_class == WAIT_CLASS_IDLE) { timer_end = get_idle_timer(); }
    else { timer_end = get_wait_timer(); }
  } else {
    timer_end = wait->m_timer_end;
  }
  normalizer->to_pico(wait->m_timer_start, timer_end, &m_row.m_timer_start,
                      &m_row.m_timer_end, &m_row.m_timer_wait);
```

**★ waits 固定用 CYCLE 计时器**（`pfs_timer.cc`）：

```c
time_normalizer *time_normalizer::get_wait() {
  return &to_pico_data[TIMER_NAME_CYCLE];
}
```

而 stages / statements / transactions 用 `USED_TIMER()`，编译期决定为 **NANOSECOND**（退化时 MICROSECOND）：

```c
#ifdef HAVE_NANOSEC_TIMER
#define USED_TIMER_NAME TIMER_NAME_NANOSEC
#define USED_TIMER my_timer_nanoseconds
#else
#define USED_TIMER_NAME TIMER_NAME_MICROSEC
#define USED_TIMER my_timer_microseconds
#endif
```

⇒ **waits 与 stages 的计时器不同源**（CYCLE vs NANOSECOND），这是跨事件比较耗时前必须知道的。

**OBJECT_INSTANCE_BEGIN** = 被等待对象的内存地址（`pfs_events_waits.h`：`/** Address in memory of the object instance waited on. */ const void *m_object_instance_addr;`）。按 wait class 分别填：

| class | OBJECT_TYPE | OBJECT_NAME |
|---|---|---|
| mutex / rwlock / cond | 空 | 空（只用 OBJECT_INSTANCE_BEGIN） |
| file | `FILE` | 文件名 |
| socket | `SOCKET` | `ip:port` |
| table io | `TABLE` / `TEMPORARY TABLE` | schema / table / index |
| MDL | 按 `MDL_key` 映射 | `GLOBAL` / `TABLESPACE` / `SCHEMA` / `TABLE` / `FUNCTION` / `COMMIT` / `USER LEVEL LOCK` / `SRID` / `ACL CACHE` / `COLUMN STATISTICS` / `BACKUP_LOCK` / `RESOURCE_GROUPS` / `FOREIGN KEY` / `CHECK CONSTRAINT` … |

**NESTING_EVENT_ID** 取自 wait 栈中的父事件（`wait - 1`）：

```c
      PFS_events_waits *parent_event = wait - 1;
      wait->m_nesting_event_id = parent_event->m_event_id;
      wait->m_nesting_event_type = parent_event->m_event_type;
```

栈布局（`pfs_instr.h`）——栈底 1 个哑记录专门承载父 stage/statement/transaction：

```c
#define WAIT_STACK_LOGICAL_SIZE 5
/**
  @def WAIT_STACK_BOTTOM
  Maximum number dummy waits records.
  One dummy record is reserved for the parent stage / statement / transaction,
  at the bottom of the wait stack.
*/
#define WAIT_STACK_BOTTOM 1
```

⇒ **等待栈最多 5 层**，嵌套超过的等待无法追溯父事件。

**OPERATION** 的值（`enum_operation_type`）：mutex 是 `lock`/`try_lock`；rwlock 是 `read_lock`/`write_lock`/`try_*`/`unlock`；cond 是 `wait`/`timed_wait`；file 是 `create`/`open`/`close`/`read`/`write`/`seek`/`tell`/`flush`/`stat`/`fstat`…。`table_events_waits.cc` 注释说明了显示名与内部枚举的差异：

```c
/**
  Note: enum_operation_type contains a more precise definition,
  since more details are needed internally by the instrumentation.
  Different similar operations (CLOSE vs STREAMCLOSE) are displayed
  with the same name 'close'.
*/
```

### 3.4 默认开关（★ 高频 instrument 默认关）

注册时**显式覆盖**默认值（`pfs_instr_class.cc`）：

```c
    /* mutex */
    entry->m_enabled = false; /* disabled by default */
    entry->m_timed = false;
```

| 类别 | 默认 ENABLED | 源码注释 |
|---|---|---|
| mutex / rwlock / cond / **socket** | **NO** | `/* disabled by default */` |
| **file** | **YES** | `/* enabled by default */` |
| table io / table lock / idle / metadata | **YES**（用基线，未覆盖） | — |

基线的默认（`init_instr_class()`）：`klass->m_enabled = true; klass->m_timed = true;`

**权衡逻辑**（源码表述 + 推断）：mutex/cond/socket 是**每次加锁都触发**的高频路径，默认关；file I/O、table I/O 频率相对低而价值高，默认开。

⚠️ **重要提醒**：mysql-test 的 `mysql-test/include/default_mysqld.cnf` 里有 `loose-performance-schema-instrument='%=ON'`，所以 **MTR 的 `.result` 文件（如 `start_server_nothing.result` 里 `wait/lock/metadata/sql/mdl YES YES`）不反映出厂默认值**，不要引用它们当默认开关。

**启动参数可覆盖，支持通配符，更长的匹配优先**（`configure_instr_class()`）：

```c
    /**
      Compare class name to all configuration entries. In case of multiple
      matches, the longer specification wins. For example, the pattern
      'ABC/DEF/GHI=ON' has precedence over 'ABC/DEF/%=OFF' regardless of
      position within the configuration file or command line.
    */
```

对应变量 `performance_schema_instrument`（`--performance-schema-instrument=`）。MTR 里有完整功能测试（`setup_instruments_defaults-master.opt`）：

```
--loose-performance-schema-instrument='%=ON'
--loose-performance-schema-instrument=' wait/synch/mutex/sql/% = OFF ''
--loose-performance-schema-instrument='wait/synch/mutex/sql/LOCK% = ON'
--loose-performance-schema-instrument='wait/synch/mutex/sql/LOCK_user_conn    = COUNTED'
```

其中 `COUNTED` = 只计数不计时，result 中体现为 `ENABLED=YES, TIMED=NO`。

### 3.5 典型查询

```sql
-- ① 当前所有连接正在等什么（最常用）
SELECT t.PROCESSLIST_ID, e.EVENT_NAME, e.OBJECT_NAME, e.OPERATION,
       ROUND(e.TIMER_WAIT/1e9, 3) AS wait_ms, e.SOURCE
  FROM performance_schema.events_waits_current e
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE e.END_EVENT_ID IS NULL AND e.EVENT_NAME != 'idle'
 ORDER BY e.TIMER_WAIT DESC LIMIT 10;

-- ② 全实例哪类等待最耗时
SELECT EVENT_NAME, COUNT_STAR,
       ROUND(SUM_TIMER_WAIT/1e12, 2) AS total_sec,
       ROUND(AVG_TIMER_WAIT/1e9, 3)  AS avg_ms
  FROM performance_schema.events_waits_summary_global_by_event_name
 WHERE COUNT_STAR > 0 AND SUM_TIMER_WAIT > 0
 ORDER BY SUM_TIMER_WAIT DESC LIMIT 15;
```

---

## 四、stages 事件

### 4.1 表族与字段（8 张）

`events_stages_current` / `_history` / `_history_long` + `events_stages_summary_by_{thread,user,account,host}_by_event_name` + `events_stages_summary_global_by_event_name`。

字段（比 waits 少很多，多了进度列）：

```c
    "  THREAD_ID BIGINT unsigned not null,\n"
    "  EVENT_ID BIGINT unsigned not null,\n"
    "  END_EVENT_ID BIGINT unsigned,\n"
    "  EVENT_NAME VARCHAR(128) not null,\n"
    "  SOURCE VARCHAR(64),\n"
    "  TIMER_START / TIMER_END / TIMER_WAIT ...\n"
    "  WORK_COMPLETED BIGINT unsigned,\n"
    "  WORK_ESTIMATED BIGINT unsigned,\n"
    "  NESTING_EVENT_ID BIGINT unsigned,\n"
    "  NESTING_EVENT_TYPE ENUM('TRANSACTION', 'STATEMENT', 'STAGE', 'WAIT'),\n"
    "  PRIMARY KEY (THREAD_ID, EVENT_ID) USING HASH\n",
```

### 4.2 ★ `WORK_COMPLETED` / `WORK_ESTIMATED` 怎么埋

**API**（`include/mysql/psi/mysql_stage.h`）——直接写结构体字段，**无锁、无函数调用开销**；非 PFS 构建展开为 `do {} while (0)`：

```c
static inline void inline_mysql_stage_set_work_completed(
    PSI_stage_progress *progress, ulonglong val) {
  if (progress != nullptr) { progress->m_work_completed = val; }
}
static inline void inline_mysql_stage_inc_work_completed(
    PSI_stage_progress *progress, ulonglong val) {
  if (progress != nullptr) { progress->m_work_completed += val; }
}
```

**什么时候才有 progress 句柄**——只有该 stage 声明了 `PSI_FLAG_STAGE_PROGRESS`（`pfs.cc` `pfs_start_stage_v1()`）：

```c
  if (new_klass->is_progress()) {
    pfs_thread->m_stage_progress = &pfs->m_progress;
    pfs->m_progress.m_work_completed = 0;
    pfs->m_progress.m_work_estimated = 0;
  }
  return pfs_thread->m_stage_progress;
```

标记定义（`psi_bits.h`）：`#define PSI_FLAG_STAGE_PROGRESS (1 << 3)`；谓词：`bool is_progress() const { return m_flags & PSI_FLAG_STAGE_PROGRESS; }`

**读表时非 progress stage 的这两列恒为 NULL**（`table_events_stages.cc`）：

```c
  if (klass->is_progress()) {
    m_row.m_work_completed = stage->m_progress.m_work_completed;
    m_row.m_work_estimated = stage->m_progress.m_work_estimated;
  }
  ...
        case 8: /* WORK_COMPLETED */
          if (m_row.m_progress) { set_field_ulonglong(f, m_row.m_work_completed); }
          else { f->set_null(); }
```

**带 progress 的 stage**（源码里显式声明了 `PSI_FLAG_STAGE_PROGRESS` 的）：

- SQL 层：`stage/sql/copy to tmp table`
- 复制：`Applying batch of row changes (write|update|delete)`
- InnoDB：7 个 `alter table (...)`（read PK and internal sort / merge sort / insert / flush / log apply index / log apply table / end）、`alter tablespace (encryption)`、`buffer pool load`
- Clone：`clone (file copy)` / `(redo copy)` / `(page copy)`

⇒ **SQL 层唯一带进度的是 `copy to tmp table`**（COPY 算法拷数据）；INPLACE 的进度靠 InnoDB 那 7 个 stage。

### 4.3 ★ 默认开关：progress stage 默认开

```c
    if (entry->is_progress()) {
      /* Stages with progress information are enabled and timed by default */
      entry->m_enabled = true;
      entry->m_timed = true;
    } else {
      /* Stages without progress information are disabled by default */
      entry->m_enabled = false;
      entry->m_timed = false;
    }
```

而 **consumer 侧默认全关**（`sql/sys_vars.cc`）：

```c
static Sys_var_bool Sys_pfs_consumer_events_stages_current(
    "performance_schema_consumer_events_stages_current",
    "Default startup value for the events_stages_current consumer.",
    READ_ONLY NOT_VISIBLE
        GLOBAL_VAR(pfs_param.m_consumer_events_stages_current_enabled),
    CMD_LINE(OPT_ARG), DEFAULT(false), PFS_TRAILING_PROPERTIES);
```

（`_history` / `_history_long` 同样 `DEFAULT(false)`；且是 `READ_ONLY NOT_VISIBLE`，**只能启动时指定**。）

⇒ **这就是"DDL 看不到进度"的完整答案**：instrument 默认开（`is_progress()`），但 **consumer 默认关且只读**，必须启动时用 `--performance-schema-consumer-events-stages-current=ON`。

### 4.4 典型查询

```sql
-- ALTER 期间看进度（另开会话反复执行）
SELECT t.PROCESSLIST_ID, e.EVENT_NAME,
       e.WORK_COMPLETED, e.WORK_ESTIMATED,
       ROUND(100 * e.WORK_COMPLETED / NULLIF(e.WORK_ESTIMATED, 0), 2) AS pct
  FROM performance_schema.events_stages_current e
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE e.WORK_ESTIMATED > 0;
```

---

## 五、transactions 事件

### 5.1 表族与字段（8 张）

同构：`events_transactions_current` / `_history` / `_history_long` + 5 张 summary。

```c
    "  EVENT_NAME VARCHAR(128) not null,\n"
    "  STATE ENUM('ACTIVE', 'COMMITTED', 'ROLLED BACK'),\n"
    "  TRX_ID BIGINT unsigned,\n"
    "  GTID VARCHAR(64),\n"
    "  XID_FORMAT_ID INTEGER,\n"
    "  XID_GTRID VARCHAR(130),\n"
    "  XID_BQUAL VARCHAR(130),\n"
    "  XA_STATE VARCHAR(64),\n"
    "  TIMER_START / TIMER_END / TIMER_WAIT ...\n"
    "  ACCESS_MODE ENUM('READ ONLY', 'READ WRITE'),\n"
    "  ISOLATION_LEVEL VARCHAR(64),\n"
    "  AUTOCOMMIT ENUM('YES','NO') not null,\n"
    "  NUMBER_OF_SAVEPOINTS BIGINT unsigned,\n"
    "  NUMBER_OF_ROLLBACK_TO_SAVEPOINT BIGINT unsigned,\n"
    "  NUMBER_OF_RELEASE_SAVEPOINT BIGINT unsigned,\n"
    "  OBJECT_INSTANCE_BEGIN BIGINT unsigned,\n"
    "  NESTING_EVENT_ID / NESTING_EVENT_TYPE ...\n",
```

### 5.2 记录什么

内存结构 `pfs_events_transactions.h` 定义了全部字段来源：`m_sid`（source id）、`m_trxid`（**InnoDB 事务 ID**）、`m_state`、`m_gtid_spec`、`m_xa` / `m_xid` / `m_xa_state`、`m_isolation_level`、`m_read_only`、`m_autocommit`、三个 savepoint 计数器。

**XID 与 X/Open XA 规范二进制兼容**：

```c
/**
  struct PSI_xid is binary compatible with the XID structure as
  in the X/Open CAE Specification, Distributed Transaction Processing:
  The XA Specification, X/Open Company Ltd., 1991.
  ...
  A value of -1 in formatID means that the XID is null.
  Max length for bqual and gtrid is 64 bytes each.
*/
```

枚举值（`pfs_column_types.h`）：

```c
enum enum_transaction_state {
  TRANS_STATE_ACTIVE = 1, TRANS_STATE_COMMITTED = 2, TRANS_STATE_ROLLED_BACK = 3
};
enum enum_xa_transaction_state {
  TRANS_STATE_XA_NOTR, TRANS_STATE_XA_ACTIVE, TRANS_STATE_XA_IDLE,
  TRANS_STATE_XA_PREPARED, TRANS_STATE_XA_ROLLBACK_ONLY, TRANS_STATE_XA_COMMITTED
};
enum enum_isolation_level {
  TRANS_LEVEL_READ_UNCOMMITTED, TRANS_LEVEL_READ_COMMITTED,
  TRANS_LEVEL_REPEATABLE_READ, TRANS_LEVEL_SERIALIZABLE
};
```

### 5.3 GTID / XID / XA 的埋点位置

SQL 层的上报点（`include/mysql/psi/mysql_transaction.h` 定义宏）：

| 宏 | 调用点 |
|---|---|
| `MYSQL_SET_TRANSACTION_GTID` | `sql/rpl_gtid_execution.cc` |
| `MYSQL_SET_TRANSACTION_XID` | `sql/xa/sql_xa_start.cc` |
| `MYSQL_SET_TRANSACTION_XA_STATE` | `sql/xa/sql_xa_{start,end,prepare,commit}.cc` |
| `MYSQL_INC_TRANSACTION_SAVEPOINTS` 等 | savepoint 相关 |

⇒ **XA 事务的完整状态机可以直接从 `events_transactions_current` 的 `XA_STATE` 观察**（NOTR → ACTIVE → IDLE → PREPARED → COMMITTED/ROLLBACK_ONLY）。

**NESTING** 取栈顶 statement（事务的父事件是语句）：

```c
      const uint statements_count = pfs_thread->m_events_statements_count;
      if (statements_count > 0) {
        PFS_events_statements *pfs_statement =
            &pfs_thread->m_statement_stack[statements_count - 1];
        pfs->m_nesting_event_id = pfs_statement->m_event_id;
        pfs->m_nesting_event_type = pfs_statement->m_event_type;
      }
```

### 5.4 默认开关：transaction 默认开（与 waits/stages 相反）

`register_global_classes()` 里**未覆盖** `m_enabled`，沿用基线 `true/true`：

```c
  /* Transaction class */
  init_instr_class(&global_transaction_class, transaction_instrument_prefix.str,
                   ..., 0, 0, PSI_DOCUMENT_ME, PFS_CLASS_TRANSACTION);
```

**Consumer 侧也默认开**（`sql/sys_vars.cc`）：

```c
static Sys_var_bool Sys_pfs_consumer_events_transactions_current(... DEFAULT(true) ...);
static Sys_var_bool Sys_pfs_consumer_events_transactions_history(... DEFAULT(true) ...);
static Sys_var_bool Sys_pfs_consumer_events_transactions_history_long(... DEFAULT(false) ...);
```

⇒ **transactions 是四类里唯一 instrument 与 consumer 都默认开的**（waits/stages 的 consumer 默认全是 false）。

### 5.5 ★ summary 表按读写分流（15 列统计）

```c
    "  COUNT_STAR / SUM_TIMER_WAIT / MIN_TIMER_WAIT / AVG_TIMER_WAIT / MAX_TIMER_WAIT,\n"
    "  COUNT_READ_WRITE / SUM_TIMER_READ_WRITE / MIN_ / AVG_ / MAX_TIMER_READ_WRITE,\n"
    "  COUNT_READ_ONLY  / SUM_TIMER_READ_ONLY  / MIN_ / AVG_ / MAX_TIMER_READ_ONLY,\n",
```

聚合时按 `m_read_only` 分流：

```c
    if (state->m_read_only) { stat->m_read_only_stat.aggregate_value(wait_time); }
    else { stat->m_read_write_stat.aggregate_value(wait_time); }
```

⇒ 可以直接读出"只读事务 vs 读写事务各自的耗时分布"。

### 5.6 典型查询

```sql
-- 当前活跃事务（含 XA 状态、GTID、隔离级别）
SELECT t.PROCESSLIST_ID, e.STATE, e.XA_STATE, e.TRX_ID, e.GTID,
       e.ACCESS_MODE, e.ISOLATION_LEVEL, e.AUTOCOMMIT,
       ROUND(e.TIMER_WAIT/1e9, 3) AS elapsed_ms
  FROM performance_schema.events_transactions_current e
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE e.STATE = 'ACTIVE';
```

---

## 六、★ errors 事件（全库此前完全没写过）

### 6.1 只有 summary，没有 current / history（5 张）

| 表名 |
|---|
| `events_errors_summary_by_thread_by_error` |
| `events_errors_summary_by_user_by_error` |
| `events_errors_summary_by_account_by_error` |
| `events_errors_summary_by_host_by_error` |
| `events_errors_summary_global_by_error` |

源码明确（`pfs.cc` `@section IMPL_ERROR`）：

```
  For errors, there are no tables that contains individual event data.

  For errors, the tables that contains aggregated data are:
  - EVENTS_ERRORS_SUMMARY_BY_ACCOUNT_BY_ERROR
  - EVENTS_ERRORS_SUMMARY_BY_HOST_BY_ERROR
  - EVENTS_ERRORS_SUMMARY_BY_THREAD_BY_ERROR
  - EVENTS_ERRORS_SUMMARY_BY_USER_BY_ERROR
  - EVENTS_ERRORS_SUMMARY_GLOBAL_BY_ERROR
```

理由不难理解：**错误不需要"逐条回溯"**（错误日志已经做了），需要的是"按错误号聚合的统计"。

### 6.2 字段定义

```c
    "  ERROR_NUMBER INTEGER,\n"
    "  ERROR_NAME VARCHAR(64),\n"
    "  SQL_STATE VARCHAR(5),\n"
    "  SUM_ERROR_RAISED  BIGINT unsigned not null,\n"
    "  SUM_ERROR_HANDLED BIGINT unsigned not null,\n"
    "  FIRST_SEEN TIMESTAMP(0) null,\n"
    "  LAST_SEEN TIMESTAMP(0) null,\n"
    "  UNIQUE KEY (ERROR_NUMBER) USING HASH\n",
```

（by_thread 加 `THREAD_ID`；by_user 加 `USER`；by_account 加 `USER`+`HOST`；by_host 加 `HOST`。）

⚠️ **TRUNCATE 任意一张会 reset 全部 5 张**（`table_ees_global_by_error::delete_all_rows()`）：

```c
int table_ees_global_by_error::delete_all_rows() {
  reset_events_errors_by_thread();
  reset_events_errors_by_account();
  reset_events_errors_by_user();
  reset_events_errors_by_host();
  reset_events_errors_global();
  return 0;
}
```

### 6.3 埋点在哪（3 处，全库仅此）

**宏** `MYSQL_LOG_ERROR(N, T)`（`include/mysql/psi/mysql_error.h`）：

```c
#ifdef HAVE_PSI_ERROR_INTERFACE
#define MYSQL_LOG_ERROR(N, T) inline_mysql_log_error(N, T)
#else
#define MYSQL_LOG_ERROR(N, T) do {} while (0)
#endif

static inline void inline_mysql_log_error(int error_num,
                                          PSI_error_operation error_operation) {
  PSI_ERROR_CALL(log_error)(error_num, error_operation);
}
```

⚠️ 注意名字：是 `MYSQL_LOG_ERROR` → `log_error` → `pfs_log_error_v1`，**不是** `MYSQL_ERROR_STATEMENT` 或 `PFS_ERROR`。

**操作类型**（`psi_error_bits.h`）：

```c
enum PSI_error_operation {
  PSI_ERROR_OPERATION_RAISED = 0,
  PSI_ERROR_OPERATION_HANDLED
};
```

**三处调用点**：

1. `sql/sql_class.cc` `raise_condition()`——错误进入 Diagnostics Area 前上报 RAISED：
   ```c
     MYSQL_LOG_ERROR(sql_errno, PSI_ERROR_OPERATION_RAISED);
     if (handle_condition(sql_errno, sqlstate, &level, msg)) return nullptr;
   ```
2. `sql/sql_signal.cc`——RESIGNAL 分支：`MYSQL_LOG_ERROR(raised->mysql_errno(), PSI_ERROR_OPERATION_RAISED);`
3. `sql/sp_head.cc`——**存储程序的 SQL handler 捕获了错误** → 上报 HANDLED：
   ```c
       if (!thd->is_fatal_error() && !thd->killed &&
           thd->sp_runtime_ctx->handle_sql_condition(thd, &ip, i)) {
         err_status = false;
         if (error_num) MYSQL_LOG_ERROR(error_num, PSI_ERROR_OPERATION_HANDLED);
       }
   ```

### 6.4 ★ `SUM_ERROR_RAISED` 是"未处理的净值"，不是累计发生次数

这是最反直觉的一点（`pfs_stat.h` `PFS_error_single_stat::aggregate_count`）：

```c
  inline void aggregate_count(int error_operation) {
    m_last_seen = my_micro_time();
    if (m_first_seen == 0) { m_first_seen = m_last_seen; }

    switch (error_operation) {
      case PSI_ERROR_OPERATION_RAISED:
        m_count++;                    // RAISED +1
        break;
      case PSI_ERROR_OPERATION_HANDLED:
        m_handled_count++;
        m_count--;                    // ★ HANDLED +1 的同时 RAISED -1
        break;
```

⇒ **`SUM_ERROR_RAISED` = 发生次数 − 被程序 handler 处理掉的次数 = 未被处理的净错误数**。

MTR 有直接证据（`mysql-test/suite/perfschema/r/error_stats_summary.result`）：

```
ERROR_NUMBER	ERROR_NAME	SQL_STATE	SUM_ERROR_RAISED	SUM_ERROR_HANDLED
1146	ER_NO_SUCH_TABLE	42S02	1	1     <-- 被 CONTINUE HANDLER 处理：RAISED 1、HANDLED 1
```

**NULL 行语义**：错误号超出 `performance_schema_error_size`（如 `SIGNAL ... SET MYSQL_ERRNO = 40000`）时归到 NULL 行（`ERROR_NUMBER/ERROR_NAME/SQL_STATE` 全 NULL）。MTR 用例正是这么构造的：

```sql
CREATE PROCEDURE bigError()
BEGIN
   SIGNAL SQLSTATE "HY000" SET MYSQL_ERRNO = 40000, MESSAGE_TEXT = "A big error number";
END//
```

### 6.5 哪些错误会被 instrument（构建期决定，不是全部）

`utilities/comp_err.cc` 在**构建期**生成分类：

```c
enum PFS_error_stat {
  PFS_NO_STAT,        /** Generate no performance schema statistics. */
  PFS_SESSION_STAT,   /** Generate performance schema session and global statistics. */
  PFS_GLOBAL_STAT     /** Generate performance schema global statistics. */
};
```

- `share/messages_to_clients.txt` 里的错误 → `PFS_SESSION_STAT` → **同时进 by_thread（进而 account/user/host）和 global**
- `share/messages_to_error_log.txt` 里的错误 → `PFS_GLOBAL_STAT` → **只进 global，不进 per-session**
- 已废弃错误、`ER_YES`、`ER_NO` → `PFS_NO_STAT`（完全不 instrument，`error_index = 0`）

```c
  if (new_error->m_is_obsolete) { new_error->m_pfs_stat = PFS_NO_STAT; }
  else if (strcmp(new_error->er_name, "ER_YES") == 0) {
    /* Historical, ER_YES is not a real error, only the error text is used. */
    new_error->m_pfs_stat = PFS_NO_STAT;
  }
```

**记录内容** = 错误码（`mysql_errno`）+ 错误名（`ER_XXX` 符号名）+ SQLSTATE（ODBC state）+ 净计数 + 已处理计数 + 首次/末次时间戳。错误名与 SQLSTATE 来自 `sql/derror.h` 的 `server_error` 结构（`name` / `mysql_errno` / `text` / `odbc_state` / `jdbc_state` / `error_index`）。

### 6.6 默认开关：开，但**不计时**

```c
  /* Error class */
  init_instr_class(&global_error_class, error_class_name.str, ..., PFS_CLASS_ERROR);
  global_error_class.m_event_name_index = GLOBAL_ERROR_INDEX;
  global_error_class.m_enabled = true; /* Enabled by default */
  configure_instr_class(&global_error_class);
  global_error_class.m_timed = false; /* Not applicable */
  error_class_max = 1;                /* only one error class as of now. */
```

三条硬证据：
- **只有一个 error instrument**（`error_class_max = 1`），名字就是 `error`（顶层，无分层）
- **默认 ENABLED=YES**
- **`m_timed = false; /* Not applicable */`** ⇒ `setup_instruments` 里 `error` 行的 **TIMED 列为 NULL**（错误事件不计时）

MTR 反向验证（`start_server_disable_errors-master.opt` 只有 `--loose-performance-schema-instrument='error=OFF'`），此时 5 张表全空。

**容量开关** `performance_schema_error_size`（`sql/sys_vars.cc`）：

```c
static Sys_var_long Sys_pfs_error_size(
    "performance_schema_error_size", "Number of server errors instrumented.",
    READ_ONLY GLOBAL_VAR(pfs_param.m_error_sizing), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, 1024 * 1024), DEFAULT(PFS_MAX_GLOBAL_SERVER_ERRORS), ...);
```

`init_error()` 在 `m_error_sizing == 0` 时整体关闭（`max_global_server_errors = 0; max_session_server_errors = 0;`）。

### 6.7 记录的前置条件（完整链）

`pfs_log_error_v1()` 按顺序检查：

1. `flag_global_instrumentation` = ON
2. `global_error_class.m_enabled` = ON
3. `max_global_server_errors != 0`
4. `flag_thread_instrumentation` 且当前线程 `m_enabled` 且 `error_stat_index < max_session_server_errors` → 记到 per-thread；否则记到 global
5. 错误号不在排除名单

```c
  if (!flag_global_instrumentation) { return; }
  if (!global_error_class.m_enabled) { return; }
  if (!max_global_server_errors) { return; }
  /* Find the index of this particular error in array of error stats. */
  error_stat_index = lookup_error_stat_index(error_num);
  if (error_stat_index >= max_global_server_errors) { error_stat_index = 0; }
```

### 6.8 引入版本：源码无直接证据

`pfs_error.cc` 等文件版权头为 "Copyright (c) 2016, 2024"；但 `pfs_dd_version.h` 的逐版本 schema 变更清单**从未出现 `events_errors_*`**，也无 WL 号注释。⇒ **无法从 8.0.39 源码确定确切引入版本**，不编造。

### 6.9 典型查询（这是此前知识库完全没有的能力）

```sql
-- ① 全实例哪些错误在发生（未被处理的净数）
SELECT ERROR_NUMBER, ERROR_NAME, SQL_STATE,
       SUM_ERROR_RAISED, SUM_ERROR_HANDLED, FIRST_SEEN, LAST_SEEN
  FROM performance_schema.events_errors_summary_global_by_error
 WHERE SUM_ERROR_RAISED > 0
 ORDER BY SUM_ERROR_RAISED DESC;

-- ② 哪个连接错误最多
SELECT t.PROCESSLIST_ID, e.ERROR_NUMBER, e.ERROR_NAME,
       e.SUM_ERROR_RAISED, e.LAST_SEEN
  FROM performance_schema.events_errors_summary_by_thread_by_error e
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE e.SUM_ERROR_RAISED > 0
 ORDER BY e.SUM_ERROR_RAISED DESC LIMIT 10;

-- ③ 哪些错误被存储程序的 handler 吞掉了（HANDLED > 0 但 RAISED 归零）
SELECT ERROR_NUMBER, ERROR_NAME, SUM_ERROR_RAISED, SUM_ERROR_HANDLED
  FROM performance_schema.events_errors_summary_global_by_error
 WHERE SUM_ERROR_HANDLED > 0;
```

### 6.10 与 `error_log` 表不是一回事（易混）

`performance_schema.error_log`（8.0.22，WL#13681）是**错误日志的 SQL 视图**，与 `events_errors_summary_*` **完全不同源**。别把两者当成同一机制。

---

## 七、★ memory 事件（PFS 的内存计量）

> ★ 这是 PFS 里**唯一不叫"事件"的一类**：它没有 start/end、没有嵌套、没有等待时间——它计量的是**内存分配/释放**。
> 之所以仍归入 `PFS_CLASS_MEMORY` 并复用 instrument/consumer 框架，是为了**复用同一套开关与聚合维度**。
> ⚠️ 它是"内存激增 / OOM"类故障的**唯一能定位到分配者**的手段——状态变量只能给总量。

### 7.1 表族（5 张，按聚合维度）

| 表 | 维度 | 用途 |
|---|---|---|
| `memory_summary_global_by_event_name` | 全局 × 事件名 | 谁分配了最多（**首选**） |
| `memory_summary_by_thread_by_event_name` | 线程 × 事件名 | 定位到具体连接 |
| `memory_summary_by_account_by_event_name` | 账号 × 事件名 | 按 `user@host` |
| `memory_summary_by_user_by_event_name` | 用户 × 事件名 | 按 user |
| `memory_summary_by_host_by_event_name` | 主机 × 事件名 | 按 host |

源码：`storage/perfschema/table_mems_global_by_event_name.cc`（另 4 张为 `table_mems_by_*_by_event_name.cc`）。

### 7.2 ★ 字段语义（10 个，分三组——读错组会得出错误结论）

以 `memory_summary_global_by_event_name` 为例（建表语句原文见 `table_mems_global_by_event_name.cc` 的 `m_table_def`）：

| 组 | 字段 | 含义 |
|---|---|---|
| **累计**（只增） | `COUNT_ALLOC` / `COUNT_FREE` | 分配次数 / 释放次数 |
| | `SUM_NUMBER_OF_BYTES_ALLOC` / `SUM_NUMBER_OF_BYTES_FREE` | 累计分配字节 / 累计释放字节 |
| **当前**（差值） | `CURRENT_COUNT_USED` = `COUNT_ALLOC − COUNT_FREE` | ★ **当前未释放的块数** |
| | `CURRENT_NUMBER_OF_BYTES_USED` | ★ **当前在用字节** |
| **水位** | `LOW_COUNT_USED` / `HIGH_COUNT_USED` | 块数的历史低/高水位 |
| | `LOW_NUMBER_OF_BYTES_USED` / `HIGH_NUMBER_OF_BYTES_USED` | ★ 字节的历史低/高水位 |

**★ 三条使用判据**（这是本节最实用的一点）：

| 要回答的问题 | 看哪个字段 | 为什么 |
|---|---|---|
| **是不是泄漏了？** | `CURRENT_COUNT_USED` | 泄漏的特征是**块数只增不减**（分配了没释放），且与该对象的生命周期不符 |
| **现在占多少？** | `CURRENT_NUMBER_OF_BYTES_USED` | 当前真实占用 |
| **OOM 之前峰值到过多少？** | `HIGH_NUMBER_OF_BYTES_USED` | ★ **峰值不会因为回落而消失**——OOM 后进程可能已重启/回落，只有水位能留下证据 |

> ⚠️ 常见误读：拿 `SUM_NUMBER_OF_BYTES_ALLOC`（累计分配）当"占用量"——它是**历史累加值**，会无限增长，与当前占用无关。

### 7.3 ★ 默认开关：memory instrument **默认开**（与 mutex/rwlock/cond 相反）

源码证据（`storage/perfschema/pfs_instr_class.cc`）：

```c
// register_memory_class() 内部调用 init_instr_class()，后者末尾：
klass->m_enabled = true;
klass->m_timed = true;
```

对比另外三类同步原语，它们**显式关闭**：

```c
// register_mutex_class / register_rwlock_class / register_cond_class 中：
entry->m_enabled = false;   /* disabled by default */
```

⇒ **memory 默认开启**（前提是 `performance_schema=ON`）。这解释了为什么"从没配过 PFS"的实例也能查到内存分配。

⚠️ 但有两层约束会让"开了也看不到"：

1. **代码必须注册了 `PSI_memory_key`** —— instrument 不是自动产生的，要靠代码调用 `PSI_MEMORY_CALL(memory_alloc)(key, size, ...)`。没埋点的地方不出现在表里
2. **`performance_schema_max_memory_class` 上限**（启动时变量）——超过上限后新注册的 memory class 不再被计入，表现为"某些事件名缺失"

### 7.4 sys 侧：`x$` 与非 `x$` 的取舍

| 视图 | 说明 |
|---|---|
| `sys.memory_global_by_current_bytes` | 全局 Top，字节数**带单位格式化**（如 1.20 MiB）——人读友好 |
| `sys.x$memory_global_by_current_bytes` | 同一数据，但 `current_alloc` 是**原始字节数**——用于计算/告警 |
| `sys.memory_by_thread_by_current_bytes` / `x$` 版 | 按线程 |
| `sys.memory_by_user_by_current_bytes` / `memory_by_host_by_current_bytes` | 按用户 / 主机 |
| `sys.memory_global_total` / `x$` 版 | 总量一行 |

⚠️ **做监控/告警必须查 `x$` 版**——非 `x$` 版的格式化字符串无法比较大小、无法做阈值判断。

### 7.5 典型查询

```sql
-- ① 全局：谁占得最多（人读）
SELECT event_name, current_alloc
  FROM sys.memory_global_by_current_bytes
 LIMIT 10;

-- ② 同上面的机器消费版（原始字节，可按阈值告警）
SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED, HIGH_NUMBER_OF_BYTES_USED,
       CURRENT_COUNT_USED
  FROM performance_schema.memory_summary_global_by_event_name
 ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC
 LIMIT 10;

-- ③ ★ 查泄漏：块数持续增长而字节不涨（典型"小对象泄漏"）
--    两次采样求差，看 CURRENT_COUNT_USED 是否单调增
SELECT EVENT_NAME, CURRENT_COUNT_USED, CURRENT_NUMBER_OF_BYTES_USED
  FROM performance_schema.memory_summary_global_by_event_name
 WHERE CURRENT_COUNT_USED > 10000
 ORDER BY CURRENT_COUNT_USED DESC
 LIMIT 10;

-- ④ 按线程定位到具体连接
SELECT t.THREAD_ID, t.PROCESSLIST_USER, t.PROCESSLIST_HOST,
       t.PROCESSLIST_COMMAND, m.CURRENT_NUMBER_OF_BYTES_USED
  FROM performance_schema.memory_summary_by_thread_by_event_name m
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE t.PROCESSLIST_ID IS NOT NULL
 ORDER BY m.CURRENT_NUMBER_OF_BYTES_USED DESC
 LIMIT 10;

-- ⑤ ★ OOM 事后取证：看历史峰值（进程可能已回落）
SELECT EVENT_NAME, HIGH_NUMBER_OF_BYTES_USED, CURRENT_NUMBER_OF_BYTES_USED
  FROM performance_schema.memory_summary_global_by_event_name
 ORDER BY HIGH_NUMBER_OF_BYTES_USED DESC
 LIMIT 10;
```

### 7.6 PFS 自己占多少：`SHOW ENGINE PERFORMANCE_SCHEMA STATUS`

PFS 是 MySQL 里少数**观测自身**的子系统。它的内存占用不走 `memory_summary_*`（那是别的模块的），而是单独出口：

```sql
SHOW ENGINE PERFORMANCE_SCHEMA STATUS;
-- 关注 (pfs).memory / performance_schema.memory 之类的行
-- 源码：storage/perfschema/ha_perfschema.cc 的 pfs_status_vars[]
```

⚠️ 排障顺序建议：**先看 PFS 自身占了多少**，再判断"是不是 PFS 把内存吃掉了"——尤其在全开 instrument 的实例上。

---

## 八、PFS 里不属于 `events_*` 的杂项表（易漏的观测点）

> 按"事件表族"的思路（waits / stages / statements / transactions / errors / memory）会**系统性漏掉这一批**——它们没有统一事件模型，但**其中几张是特定故障的唯一解**。

| 表 | 观测什么 | 什么时候需要 | 常用度 |
|---|---|---|---|
| **`log_status`** | redo / binlog / GTID 的**一致性位点** | 备份工具取一致性坐标（`LOCK INSTANCE FOR BACKUP` 配套） | ★★★ |
| **`prepared_statements_instances`** | 每个 prepare 语句实例、执行次数 | ★ 排查 **prep stmt 泄漏**（`max_prepared_stmt_count` 打满、应用忘了 close） | ★★★ |
| **`events_statements_summary_by_program`** | 按**存储程序**（procedure / function / trigger / event）聚合 | 存储程序密集型业务定位热点 routine | ★★★ |
| **`session_connect_attrs`** | 客户端连接属性（`_client_name` / `_pid` / `_os`…） | ★ 溯源"这个连接是哪个应用/驱动/进程发来的" | ★★★ |
| `objects_summary_global_by_type` | 按对象类型的操作汇总 | 各类对象的 DDL / 操作频次 | ★★☆ |
| `socket_summary_by_event_name` / `socket_summary_by_instance` | 网络层 socket 读写字节/次数 | 网络侧瓶颈定位 | ★★☆ |
| `user_variables_by_thread` | 会话 `@var` 的值 | ★ **可跨会话查看别人的用户变量** | ★☆☆ |
| `binary_log_transaction_compression_stats` | binlog 事务压缩率 / 节省字节 | 评估压缩效果 | ★☆☆ |
| `tls_channel_status` | 各连接通道的 TLS 版本 / 密码套件 / 会话复用 | 合规审计 | ★☆☆ |
| `keyring_component_status` | keyring 组件状态 | 组件运维 | ★☆☆ |

### 8.1 几张值得展开的

**`prepared_statements_instances`（★★★ 泄漏排查）**

`max_prepared_stmt_count` 打满时（报 `ER_MAX_PREPARED_STMT_COUNT_REACHED`），先分清是**总量不够**还是**应用没 close**：

```sql
-- 每个连接的 prepared statement 持有数
SELECT OWNER_THREAD_ID, COUNT(*) AS stmt_count
  FROM performance_schema.prepared_statements_instances
 GROUP BY OWNER_THREAD_ID
 ORDER BY stmt_count DESC LIMIT 10;
```

若某连接的持有数随时间单调增长 ⇒ 应用侧泄漏，与 `max_prepared_stmt_count` 调多大无关。

**`session_connect_attrs`（★★★ 客户端溯源）**

`SHOW PROCESSLIST` 只能看到 host，看不到"是哪个进程/哪种驱动"。连接属性由客户端在握手时上报：

```sql
SELECT t.PROCESSLIST_ID, t.PROCESSLIST_USER, t.PROCESSLIST_HOST,
       a.ATTR_NAME, a.ATTR_VALUE
  FROM performance_schema.session_connect_attrs a
  JOIN performance_schema.threads t USING (THREAD_ID)
 WHERE a.ATTR_NAME IN ('_client_name', '_client_version', '_pid', '_os', '_platform')
 ORDER BY t.PROCESSLIST_ID;
```

⚠️ 属性由**客户端主动上报**，客户端不传就没有（不是服务端强制）。

**`user_variables_by_thread`（★☆☆ 但很特殊）**

能**跨会话查看别人的 `@var`**——这是少数能"从外部看另一个会话内部状态"的手段：

```sql
SELECT * FROM performance_schema.user_variables_by_thread;
```

**`log_status`（★★★ 备份一致性位点）**

给备份工具取 redo / binlog / GTID 一致性坐标用（与 `LOCK INSTANCE FOR BACKUP` 配套）。⚠️ 它**不是**"日志的状态"，名字容易误读。

### 8.2 杂项表的共性提醒

- **多数没有 consumer 开关**：不像 `events_*` 需要 `setup_consumers`，这些表多半只要 instrument 开（`setup_instruments`）就有数据
- **`TRUNCATE` 语义各异**：部分表（如 `events_statements_summary_by_program`）会被 `TRUNCATE performance_schema.*` 清零；个别表不可截断
- **命名不统一**：既有 `*_summary_*` 也有 `*_instances` 也有无前缀的（如 `log_status`、`threads`），无法靠前缀推断行为

---
## 九、可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `performance_schema` | ON | Global（只读） | PFS 总开关 |
| `performance_schema_consumer_events_waits_current` | **false** | Global（只读，仅启动） | waits current 表 |
| `performance_schema_consumer_events_waits_history` | **false** | Global（只读） | per-thread 环形缓冲 |
| `performance_schema_consumer_events_waits_history_long` | **false** | Global（只读） | 全局环形缓冲 |
| `performance_schema_consumer_events_stages_current` | **false** | Global（只读） | ★ DDL 进度就看它 |
| `performance_schema_consumer_events_stages_history` / `_long` | **false** | Global（只读） | |
| `performance_schema_consumer_events_transactions_current` | **true** | Global（只读） | ★ 四类里唯一默认开 |
| `performance_schema_consumer_events_transactions_history` | **true** | Global（只读） | |
| `performance_schema_consumer_events_transactions_history_long` | **false** | Global（只读） | |
| `performance_schema_events_waits_history_size` | -1（自动） | Global（只读） | per-thread 行数，0 禁用 |
| `performance_schema_events_waits_history_long_size` | -1（自动） | Global（只读） | 全局行数，0 禁用 |
| `performance_schema_error_size` | `PFS_MAX_GLOBAL_SERVER_ERRORS` | Global（只读） | 被 instrument 的错误数，0 关闭 |
| `performance_schema_instrument` | — | 启动参数 | 通配符覆盖 instrument 开关，长的优先 |

（stages / transactions 的 `*_history_size` / `_history_long_size` 与 waits 对称，均在 `sql/sys_vars.cc`。）

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 现在在等什么 | SQL | `events_waits_current`（排除 `idle`） |
| 哪类等待最耗时 | SQL | `events_waits_summary_global_by_event_name` |
| DDL 进度 | SQL | `events_stages_current` 的 `WORK_COMPLETED/WORK_ESTIMATED`（★ 需先开 stage consumer） |
| 当前事务状态 / XA | SQL | `events_transactions_current`（`STATE`、`XA_STATE`、`GTID`、`XID_*`） |
| 只读 vs 读写事务耗时 | SQL | `events_transactions_summary_*`（`*_READ_ONLY` / `*_READ_WRITE` 两组列） |
| **哪些错误在发生** | SQL | `events_errors_summary_global_by_error` |
| **哪些错误被 handler 吞了** | SQL | `events_errors_summary_*` 的 `SUM_ERROR_HANDLED` |
| instrument 开关状态 | SQL | `setup_instruments`（`ENABLED` / `TIMED` / `PROPERTIES`） |
| consumer 开关状态 | SQL | `setup_consumers` |
| 计时器频率/分辨率/开销 | SQL | `performance_timers`（⚠️ `setup_timers` 已在 8.0.4 移除，不可配置） |

---

## 十、★ 工程实现技法

### 10.1 环形缓冲的"脏读 + 消毒"取舍（省 8 次原子操作）

`table_events_waits.cc` `make_row()` 的注释是极好的设计权衡材料：

```c
  /*
    Design choice:
    We could have used a pfs_lock in PFS_events_waits here,
    to protect the reader from concurrent event generation,
    but this leads to too many pfs_lock atomic operations
    each time an event is recorded:
    - 1 dirty() + 1 allocated() per event start, for EVENTS_WAITS_CURRENT
    - 1 dirty() + 1 allocated() per event end, for EVENTS_WAITS_CURRENT
    - 1 dirty() + 1 allocated() per copy to EVENTS_WAITS_HISTORY
    - 1 dirty() + 1 allocated() per copy to EVENTS_WAITS_HISTORY_LONG
    or 8 atomics per recorded event.
    The problem is that we record a *lot* of events ...

    This code is prepared to accept *dirty* records,
    and sanitizes all the data before returning a row.
  */
```

⇒ **放弃加锁，接受脏读，在返回行前"消毒"（sanitize）**。这是"优化采集路径、复杂度推给读取侧"原则的具体落地。

环形缓冲本身是纯粹的**覆盖最旧**（无丢弃逻辑）：

```c
  index++;
  if (index >= events_waits_history_per_thread) {
    index = 0;
    thread->m_waits_history_full = true;
  }
```

全局缓冲用原子 index 取模（`insert_events_waits_history_long`）：

```c
  uint index = events_waits_history_long_index.m_u32++;
  index = index % events_waits_history_long_size;
  if (index == 0) { events_waits_history_long_full = true; }
```

`*_full` 标志只给**读取侧**判断缓冲是否已绕圈，不阻止写入。空槽位是 `m_wait_class = NO_WAIT_CLASS`，读表时跳过（`return HA_ERR_RECORD_DELETED`）。

### 10.2 TIMED=NO 的"完全短路"

`pfs_start_mutex_wait_v1()`：

```c
    if (pfs_mutex->m_timed) {
      timer_start = get_wait_timer();
      ...
      flags = STATE_FLAG_TIMED;
    } else {
      /*
        Complete shortcut.
      */
      /* Aggregate to EVENTS_WAITS_SUMMARY_BY_INSTANCE (counted) */
      pfs_mutex->m_mutex_stat.m_wait_stat.aggregate_counted();
      return nullptr;
    }
```

⇒ **TIMED=NO 时不读 CPU cycle counter、不返回 locker、结束侧也不执行**——这是最大的开销节省点。代价是 `TIMER_*` 列为 NULL、current/history 表**不会出现该 wait**。

这与 `ENABLED` 的区别：`ENABLED=NO` 是**连 PFS 都不调用**；`ENABLED=YES, TIMED=NO`（MTR 里的 `COUNTED`）是**调用但只计数**。

### 10.3 聚合的中间缓冲复用

见 2.3：多级聚合分解成 `F1 → F_1_to_2 → F_2_to_3`，中间结果存内部 buffer，算 F3 时复用算 F2 已完成的部分。

### 10.4 wait 栈的固定深度与哑记录

```c
#define WAIT_STACK_LOGICAL_SIZE 5
#define WAIT_STACK_BOTTOM 1        // 栈底 1 个哑记录承载父 stage/statement/transaction
#define WAIT_STACK_SIZE (WAIT_STACK_BOTTOM + WAIT_STACK_LOGICAL_SIZE)
```

⇒ 用**栈上数组**而非动态分配（满足"instrumentation 不分配内存"原则，见 `PRINCIPLE_BEHAVIOR`：`None of the instrumentation points allocate memory.`），代价是嵌套深度硬上限 5。

---

## 十一、Misc（易混淆点）

| 混淆 | 澄清 |
|---|---|
| `setup_timers` 能配计时器？ | ❌ **8.0.4 已移除**。waits 固定 CYCLE，stages/statements/transactions 固定 NANOSECOND（退化 MICROSECOND），均不可配 |
| waits 与 stages 的 TIMER_WAIT 能直接比？ | ⚠️ 单位都是皮秒，但**计时器不同源**（CYCLE vs NANOSECOND），精度/开销不同 |
| MTR 的 `.result` 文件能当默认开关？ | ❌ **不能**。MTR 配了 `%=ON`，其 result 不反映出厂默认值 |
| `SUM_ERROR_RAISED` 是累计发生次数？ | ❌ **是"未处理的净值"**，被 handler 处理会 −1 |
| errors 有逐事件表吗？ | ❌ **只有 summary**，源码明确"no tables that contains individual event data" |
| `error` instrument 在 `setup_instruments` 里有 TIMED 列吗？ | ❌ **为 NULL**（`m_timed = false; /* Not applicable */`） |
| `performance_schema.error_log` 与 `events_errors_summary_*` 同源？ | ❌ 完全不同源，前者是错误日志的 SQL 视图（8.0.22 / WL#13681） |
| TIMED=NO 只是不计时？ | ❌ 还导致**该 wait 不进 current/history**（走 Complete shortcut 直接返回 nullptr） |
| 一个错误号超出 `performance_schema_error_size` 会丢？ | ❌ 归到 **NULL 行**（`ERROR_NUMBER/NAME/SQL_STATE` 全 NULL），不丢 |
| TRUNCATE 一张 errors 表只清那张？ | ❌ **会 reset 全部 5 张** |

---

## 参考

- *MySQL 8.0 Reference Manual → Performance Schema*: *Wait Event Tables* / *Stage Event Tables* / *Transaction Event Tables* / *Error Summary Tables*
- 源码：
  - waits：`storage/perfschema/table_events_waits.cc`、`table_ews_by_*_by_event_name.cc`、`pfs_events_waits.cc`、`pfs_timer.cc`、`pfs_column_values.cc`
  - stages：`storage/perfschema/table_events_stages.cc`、`table_esgs_*_by_event_name.cc`、`pfs_instr_class.cc`；API `include/mysql/psi/mysql_stage.h`
  - transactions：`storage/perfschema/table_events_transactions.cc`、`table_ets_*_by_event_name.cc`、`pfs_events_transactions.h`；API `include/mysql/psi/mysql_transaction.h`
  - errors：`storage/perfschema/pfs_error.cc`、`table_ees_*_by_error.cc`、`pfs_stat.h`、`table_helper.cc`；API `include/mysql/psi/mysql_error.h`、`psi_error_bits.h`；埋点 `sql/sql_class.cc`、`sql/sql_signal.cc`、`sql/sp_head.cc`；构建期分类 `utilities/comp_err.cc`
  - 共同机制：`storage/perfschema/pfs.cc`（`PAGE_PFS` / `PAGE_PFS_PSI` / `PAGE_PFS_AGGREGATES` 三份 doxygen 主文档）、`table_setup_instruments.cc`、`table_setup_consumers.cc`、`pfs_dd_version.h`、`sql/sys_vars.cc`
- MTR 用例（默认开关与 errors 语义的直接证据）：`mysql-test/suite/perfschema/t/setup_instruments_defaults-master.opt`、`.../r/error_stats_summary.result`、`.../t/start_server_disable_errors-master.opt`
- [`pfs.md`](pfs.md)（PFS 框架）· [`pfs_statement.md`](pfs_statement.md)（statement 事件）· [`../README.md`](../README.md)（观测手段全景与决策树）

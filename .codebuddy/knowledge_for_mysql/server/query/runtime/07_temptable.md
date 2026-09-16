# 07 内部临时表深度解析

> 基于 MySQL 8.0.39 源码。本篇回答：**MySQL 为什么需要一个"临时表"层？两个内存引擎（TempTable / MEMORY）各是怎么实现的？内存不够时怎么一层层退到磁盘？**
>
> **边界**：本篇讲**内部临时表**这个通用载体（两个引擎的实现、建表设计、阈值与降级链、物化时的使用）。filesort 排序算法见 [`01_filesort.md`](01_filesort.md)；各场景为什么需要物化见对应篇（CTE、窗口、semi-join 等）。

## 目录

- [概述](#概述)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [参考](#参考)

---

## 概述

**是什么**：内部临时表（internal temporary table）是 MySQL 执行期**物化的载体**——当查询需要"先把中间结果存下来再处理"（GROUP BY、DISTINCT、UNION、derived/CTE 物化、窗口函数、semi-join 去重等），就把中间结果写进一张临时表。

**解决什么问题**：执行器是流式的（火山模型一次一行），但很多算子需要**暂存**中间结果——去重需要记住"见过哪些"、分组聚合需要把同组行聚到一起、物化需要存下子结果供多次读取。

**两类临时表**（容易混淆）：

| | 谁创建 | 存储引擎 | 举例 |
|---|---|---|---|
| **内部临时表** | **优化器自动创建**，用户看不到 | TempTable / MEMORY → 降级到 InnoDB | GROUP BY / DISTINCT / UNION / 物化 |
| 用户临时表 | 用户 `CREATE TEMPORARY TABLE` | 用户指定（默认 `default_tmp_storage_engine`） | 显式建表 |

本篇只讲前者。

**在链路的哪个位置**：由优化器在 `make_tmp_tables_info` 阶段决定要不要用、建什么键；执行期由 `MaterializeIterator` / `TemptableAggregateIterator` 等算子建表、写入、扫描。它横跨优化器（决策）与执行器（使用）。

---

## 设计思想与理论基础

### 设计思想与权衡

#### 为什么需要"临时表"这一层，而不是直接用内存数据结构

最自然的想法是：去重用个内存 hash 表、分组用个内存 map 不就行了？MySQL 选择"临时表"作为统一载体，有三个理由：

1. **统一抽象**：去重、分组、物化、半连接去重……本质都是"存一批行 + 按键查找/去重"。用"一张带键的表"统一表达，一套代码（`create_tmp_table` + 索引查找）覆盖所有场景
2. **天然支持溢出**：内存数据结构溢出到磁盘要自己实现序列化/分片；而"表"是存储引擎的接口，**换引擎就等于换存储介质**——内存引擎装不下，换磁盘引擎即可，上层代码不用改
3. **复用引擎能力**：索引查找、范围扫描都由引擎提供

**代价**：表的抽象带来开销——每行要走 record 格式打包/解包、查找走 handler 接口（虚函数）。相比专用内存结构更重。但当数据量超过内存时，这个"重"换来了优雅的降级。

TempTable 引擎针对这个代价做了关键优化：**所有列定长时直接按连续字节存行，零拷贝**（见后面 Row/Cell 一节）。

#### 降级链为什么是"多级"而不是"内存不够就直接落盘"

这是本篇最核心的设计。完整链路是**四级**：

```
① RAM（物理内存）                    temptable_max_ram 默认 1GiB（全局）
     ↓ 超过
② mmap 文件（磁盘-backed，仍按内存访问） temptable_max_mmap 默认 1GiB（全局）
     ↓ 也用尽
③ 抛 RECORD_FILE_FULL
     ↓
④ InnoDB 磁盘临时表（真正的落盘，新建表 + 逐行拷贝）
```

**为什么要插入 mmap 这一级？** 因为 ④ 的代价极高——它不是"把内存表刷到磁盘"，而是**新建一张 InnoDB 表 + 逐行拷贝 + 换掉 handler**。如果内存一满就做这件事，中等规模查询会频繁付出这个代价。mmap 让"内存不够"时先退到"用磁盘但还当内存用"的状态，程序完全无感，只是变慢。

**MEMORY 引擎没有 ①② 的分级**——它只有一个每表上限（`min(tmp_table_size, max_heap_table_size)`），一满就直接跳到 ④。

#### 8.0 为什么用 TempTable 换掉 MEMORY（三个动因，按决定性排序）

**1. MEMORY 引擎不支持 BLOB/TEXT（最硬的限制）**

SQL 层选引擎时有一条直接判断：

```cpp
static bool use_tmp_disk_storage_engine(
    THD *thd, TABLE *table, ulonglong select_options, bool force_disk_table,
    enum_internal_tmp_mem_storage_engine mem_engine) {
  TABLE_SHARE *share = table->s;

  /* Caller needs SE to be disk-based (@see create_tmp_table()). */
  if (force_disk_table) {
    return true;
  }

  /*
    During bootstrap, the heap engine is not available, so we force using
    disk storage engine. This is especially hit when creating a I_S system
    view definition with a UNION in it AND is also when upgrading from
    older DD tables which involves execution of UPDATE queries to adjust
    metadata of DD tables.
  */
  if (opt_initialize || thd->is_dd_system_thread()) {
    return true;
  }

  if (mem_engine == TMP_TABLE_MEMORY) {
    /* MEMORY do not support BLOBs */
    if (share->blob_fields) {
      return true;
    }
  } else {
    assert(mem_engine == TMP_TABLE_TEMPTABLE);
  }
```

**逐段解读**：

- `force_disk_table`：调用方明确要求磁盘（如 `big_tables` 开启、或估计结果太大）
- **bootstrap / DD 系统线程**：此时 heap 引擎还没加载，强制磁盘
- **MEMORY 引擎 + 有 BLOB 字段 ⇒ 直接上磁盘临时表**——注释直白地写着 "MEMORY do not support BLOBs"。注意这个判断**只对 MEMORY 做**，TempTable 分支直接 `assert` 通过（它支持 BLOB/JSON/GEOMETRY）

也就是说：只要查询的中间结果里有 BLOB/TEXT 列（在 JSON 普及后非常常见），用 MEMORY 引擎就**注定要走磁盘**，完全失去了内存临时表的意义。

**2. MEMORY 是定长行，浪费严重**

MEMORY 的行是**定长**的——`VARCHAR(255)` 列不管实际存 `"a"` 还是 255 字节，都按最大长度分配。TempTable 支持**变长行**：按实际长度存，同样的内存能装下多得多的行。

**3. MEMORY 有索引能力限制**

处理 group 列时源码里有这段（注释即为证据）：

```cpp
  for (ORDER *tmp = group; tmp; tmp = tmp->next) {
    /*
      marker == MARKER_BIT means two things:
      - store NULLs in the key, and
      - convert BIT fields to 64-bit long, needed because MEMORY tables
        can't index BIT fields.
    */
    (*tmp->item)->marker = Item::MARKER_BIT;
  }
```

**MEMORY 表不能索引 BIT 字段**，需要转成 64 位长整型才能建键——引擎能力不足打补丁的典型。TempTable 自实现索引，不受此限。

#### 代价要落到具体后果

- **降级是"全表转换"而非"增量溢出"**：一旦触发 ④，整张内存表要**逐行拷贝**到 InnoDB。所以"稍微超一点内存"和"超很多"的代价差不多，都吃一次全量拷贝——这解释了为什么 `Created_tmp_disk_tables` 上升时性能往往是**断崖式**下跌
- **临时表没有统计信息**：优化器对临时表里有多少行只能靠估算，基于临时表的后续操作（join、再排序）代价估算可能严重偏离
- **`temptable_max_ram` 是全局共享的**（不是每表、每会话）：一个会话跑个大查询把 1GiB 吃满，其他会话的临时表也会被迫走 mmap/磁盘。这是"全局预算"设计的必然代价
- **`max_heap_table_size` 对 TempTable 完全不生效**（见阈值一节）——这是很容易记错的一条

### 理论溯源

物化（materialization）是查询执行的基础算子，经典教材里叫 materialize / staging。MySQL 的特色在于把"物化的存储"抽象成**表**，复用存储引擎的多态能力——用存储引擎抽象换介质。

### 算法与数据结构

**TempTable 引擎**（8.0.2 引入）的核心是一套**块式内存分配 + 自实现索引**：

| 类 | 职责 |
|---|---|
| `Table` / `Handler` | 表主体与 handler 子类 |
| `Row` / `Cell` | **行与字段的轻量描述** |
| `Storage` / `Block` / `Chunk` | 块式内存管理，超过阈值改用 mmap |
| `Allocator` | 分配器，`threshold()` 即 `temptable_max_ram` |
| `Tree` / `Hash_unique` / `Hash_duplicates` | 三种索引实现（**C++ 标准容器，非 B+ 树**） |
| `Index` / `kv_store` / `sharded_kv_store` | 索引与分片存储（降低并发争用） |

**MEMORY 引擎**（`ha_heap` 系列，最老的引擎之一）：定长行 + hash/BTREE 索引，`hp_hash` / `hp_rkey` 等文件实现。

### 他库对比与演进动机

| | MySQL 8.0 | PostgreSQL | SQL Server |
|---|---|---|---|
| 物化载体 | **临时表**（可换引擎） | tuplestore（专用内存/文件结构） | worktable |
| 溢出方式 | 换存储引擎（TempTable/MEMORY → InnoDB） | 内部写临时文件 | 内部 spill |
| 内存上限 | `temptable_max_ram`（全局）+ 会话级 | `work_mem`（每算子） | 内存授予（memory grant） |

**演进**：5.x 起用 MEMORY（定长行、无 BLOB、能力受限）→ 8.0.2 引入 TempTable（变长行、支持 BLOB、mmap 中间层）→ 8.0 后期磁盘侧**固定 InnoDB**（`internal_tmp_disk_storage_engine` 参数在 8.0.39 已只剩错误消息文本，代码里 `create_ondisk_from_heap` 直接锁定 `innodb_hton`）。

> 演进动机一句话：**用"更好的内存引擎 + mmap 中间层"把真正的落盘推得尽可能晚。**

---

## 核心实现

### 主链路

```
优化期：make_tmp_tables_info()        决定要不要临时表、建什么键、用什么 op_type
执行期：
  MaterializeIterator / TemptableAggregateIterator 等
    ├─ create_tmp_table()             按 Temp_table_param 建表（字段 + 键）
    │    └─ use_tmp_disk_storage_engine()  先决定内存还是磁盘引擎
    │         └─ 可能直接建 InnoDB 磁盘表（有 BLOB 且 MEMORY / 预估太大）
    ├─ instantiate_tmp_table()        handler->create() + open_tmp_table()
    ├─ 写入：ha_write_row()           去重靠唯一键冲突（FOUND_DUPP_KEY）
    ├─ 若返回 RECORD_FILE_FULL
    │    └─ create_ondisk_from_heap()  ★ 换引擎 + 逐行拷贝到 InnoDB
    └─ 扫描：ha_rnd_next() / ha_index_...()
```

### 两个内存引擎：TempTable 与 MEMORY

#### TempTable：Row / Cell 与"定长行零拷贝"

`Cell` 是**轻量解释器**，只持有 `is_null / data_length / data*` 指针，指向 Row 内部数据区——不拷贝数据。

`Row` 有**两种状态**：

- **轻量**：`m_ptr` 直接指向 MySQL `write_row()` 传入的 record buffer，**不拷贝**
- **自有**：调用 `copy_to_own_memory()` 后，布局为 `[buf_len][Cell 数组][用户数据]`，一次分配整行

`Handler::create()` 会先判断"是否所有列定长"：

- **定长**：`m_rows.element_size(rec_buff_length)`——行就是连续字节，直接按长度切分
- **变长**（VARCHAR/BLOB/JSON/GEOMETRY）：存 `Row` 对象

> **这是 TempTable 快的关键**——定长行**零拷贝**，写入时不需要额外的数据复制。

#### TempTable 的索引：C++ 标准容器，不是 B+ 树

```cpp
// src/table.cc —— 按 MySQL 声明的索引算法选择实现
switch (mysql_index.algorithm) {
  case HA_KEY_ALG_BTREE: append_new_index<Tree>(...); break;
  case HA_KEY_ALG_HASH:
    if (mysql_index.flags & HA_NOSAME) append_new_index<Hash_unique>(...);
    else append_new_index<Hash_duplicates>(...);
}
```

- **默认算法是 hash**
- `Hash_unique::insert` 遇到重复返回 `FOUND_DUPP_KEY`（去重就是靠这个返回值）
- `Tree::insert` 用 `lower_bound` 判重
- **TempTable 的 rowid 是行存储元素的指针**（`ref_length = sizeof(Storage::Element*)`）——定位一行就是解引用一个指针，极快

#### MEMORY：定长行 + hash/BTREE

MEMORY（`ha_heap` 家族）是 MySQL 最老的引擎之一：

- 行**定长**（`hp_create` 里按 `reclength` 计算，注释提到 "so the record length should be at least sizeof(uchar*)"）
- 支持 hash（`hp_hash`）与 BTREE（`hp_rkey`）索引
- **不支持 BLOB/TEXT**（见引擎选择）
- 不支持 BIT 字段索引（见前面的 `MARKER_BIT` 注释）

#### 对比

| | TempTable | MEMORY |
|---|---|---|
| 行格式 | **变长**（变长列按实际长度） | **定长**（varchar 按最大长度） |
| BLOB/JSON/GEOMETRY | ✅ 支持 | ❌ 不支持（有则直接走磁盘） |
| BIT 索引 | ✅ | ❌ 不能索引 BIT |
| 索引实现 | C++ 容器（`Tree`/`Hash_unique`/`Hash_duplicates`） | `hp_hash` / `hp_rkey` |
| 默认索引算法 | hash | 依声明（内部临时表多为 hash） |
| rowid | `Storage::Element*` 指针 | 行在块内的位置 |
| 内存上限 | `temptable_max_ram`（全局）+ `tmp_table_size`（每表） | `min(tmp_table_size, max_heap_table_size)`（每表） |
| 超限后 | mmap → RECORD_FILE_FULL → InnoDB | 直接 RECORD_FILE_FULL → InnoDB |

### create_tmp_table：字段与键的设计

#### 隐藏字段：为什么需要"看不见的列"

临时表里除了用户可见的列，还有**隐藏字段**（`register_hidden_field` 注册，标记为 `HT_HIDDEN_SQL`）：聚合/函数的结果列、去重/计数用的辅助列（如半连接去重的 rowid、ROLLUP 的分组标记）。

建表时为它们预留位置：

```cpp
  const uint extra_fields = 1 + (param->needs_set_counter() ? 1 : 0);
  Field **reg_field =
      own_root.ArrayAlloc<Field *>(field_count + extra_fields + 1, nullptr);
  Field **default_field =
      own_root.ArrayAlloc<Field *>(field_count + extra_fields, nullptr);
  Field **from_field =
      own_root.ArrayAlloc<Field *>(field_count + extra_fields, nullptr);
  Item **from_item =
      own_root.ArrayAlloc<Item *>(field_count + extra_fields, nullptr);
  uint *blob_field = own_root.ArrayAlloc<uint>(field_count + 2);
  ...
  // Leave the first place(s) to be prepared for hash_field (and counter, if
  // needed
  reg_field += extra_fields;
```

**逐段解读**：字段数组**头部**先给 `hash_field`（和 counter）留位，然后是聚合列，最后才是用户可见列。源码注释说明隐藏元素必须排在前面——窗口函数等场景依赖这个顺序来正确求值。

#### hash_field：键太长时怎么办（核心设计）

唯一约束（DISTINCT / 去重）本该直接建唯一键，但**键有长度上限**，而且某些引擎有索引能力限制。于是有了这个设计——改用**隐藏的 hash 字段**：

```cpp
  /**
    When true, enforces unique constraint (by adding a hidden hash_field and
    creating a key over this field) when:
    (1) unique key is too long, or
    (2) number of key parts in distinct key is too big, or
    (3) the caller has requested it.
    (4) we have INTERSECT or EXCEPT, i.e. not UNION.
  */
  bool unique_constraint_via_hash_field =
      param->m_operation != Temp_table_param::TTP_UNION_OR_TABLE;
```

**机制**：把所有要去重的列算成一个 hash 值，存进隐藏字段，然后**只在这个 hash 字段上建唯一键**。

**典型的工程权衡**：

- **收益**：键长度恒定，不受列数/列宽限制；绕开引擎的索引能力限制
- **代价**：**hash 冲突会导致误判重复**——两行内容不同但 hash 相同时，后一行会被当成重复而丢弃。MySQL 接受这个风险（概率极低）换取通用性

注意初始值 `param->m_operation != TTP_UNION_OR_TABLE`——**只要不是 UNION/普通表（即 INTERSECT / EXCEPT）就默认走 hash 字段**，因为集合交集/差集天然需要按整行去重。

#### 各场景的键设计

| 场景 | 键设计 |
|---|---|
| `GROUP BY` | 按 group 列建键（唯一键则实现分组去重），聚合结果存普通列 |
| `DISTINCT` | 按 select 列建唯一键，重复行插入失败（`FOUND_DUPP_KEY`）即丢弃 |
| `UNION`（非 ALL） | **全列唯一键**去重（所以 `UNION` 比 `UNION ALL` 贵就贵在这张表的唯一键维护） |
| DuplicateWeedout（semi-join） | 只有 **rowid** 列 + 唯一键的表，靠 rowid 唯一键去掉重复的外层行（详见 semi-join 篇） |
| CTE / derived 物化 | 通常无键（纯存储），递归 CTE 需特殊处理 |

#### 创建路径

- `create_tmp_table()`：内部临时表（聚合/去重/物化）
- `create_tmp_table_from_fields()`：给定字段列表直接建表（UNION/derived 物化）
- `setup_tmp_table_handler()` + `instantiate_tmp_table()`：真正 `handler->create()` + `open_tmp_table()`

### 内存分配与多级阈值（核心难点，含一处纠错）

> ⚠️ **纠错**：`max_heap_table_size` **只对 MEMORY 引擎生效，对 TempTable 完全不生效**。

每表行数的计算体现了这一点：

| 级别 | 机制 |
|---|---|
| **每表（MEMORY）** | `share->max_rows = min(tmp_table_size, max_heap_table_size) / reclength` |
| **每表（TempTable）** | 只用 `tmp_table_size`；`Prefer_RAM_over_MMAP_policy_obeying_per_table_limit` 在**块粒度**判超限 → `throw Result::RECORD_FILE_FULL` |
| **全局（TempTable）** | `Prefer_RAM_over_MMAP_policy`：RAM 未达 `temptable_max_ram` 用 RAM；超了转 MMAP；再超 `temptable_max_mmap` 就 `RECORD_FILE_FULL` |

TempTable 的分配策略命名就说明了行为——**优先 RAM，超了用 MMAP**：

```cpp
    static size_t threshold() { return temptable_max_ram; }
```

分配块时的注释（块层）：

```
 * we have allocated more than `temptable_max_ram` we start taking memory from
 * the OS disk, using mmap()'ed files.
```

**每表监控**：`TableResourceMonitor` 负责跟踪单表已用量，超限返回 `HA_ERR_RECORD_FILE_FULL`（TempTable 内部是 `throw Result::RECORD_FILE_FULL`，由 handler 层转成 MySQL 错误码）。

### 降级到磁盘：两条路径

#### 路径 ①：建表时就决定用磁盘

`create_tmp_table_with_fallback()`——`instantiate_tmp_table` 时若捕获到 `HA_ERR_RECORD_FILE_FULL`，直接换 `innodb_hton` 重建。适用于"预估就放不下"的场景。

#### 路径 ②：插入途中转磁盘（create_ondisk_from_heap）

这是核心难点。当写入内存表返回"满"时接管：

```cpp
bool create_ondisk_from_heap(THD *thd, TABLE *wtable, int error,
                             bool insert_last_record, bool ignore_last_dup,
                             bool *is_duplicate) {
  int write_err = 0;
  bool table_on_disk = false;
  DBUG_TRACE;

  if (error != HA_ERR_RECORD_FILE_FULL) {
    /*
      We don't want this error to be converted to a warning, e.g. in case of
      INSERT IGNORE ... SELECT.
    */
    wtable->file->print_error(error, MYF(ME_FATALERROR));
    return true;
  }

  if (wtable->s->db_type() != heap_hton) {
    if (wtable->s->db_type() != temptable_hton) {
      /* Do not convert in-memory temporary tables to on-disk
      temporary tables if the storage engine is anything other
      than the temptable engine. */
      wtable->file->print_error(error, MYF(ME_FATALERROR));
      return true;
    }

    /* If we are here, then the in-memory temporary tables need
    to be converted into on-disk temporary tables */
  }
```

**逐段解读**：

1. **只有"文件满"才转换**——其他错误直接上报，不能被转成警告吞掉（注释特别提到 `INSERT IGNORE ... SELECT`）
2. **只对 MEMORY（`heap_hton`）和 TempTable（`temptable_hton`）做转换**

接着是目标引擎与 CTE 克隆的处理：

```cpp
  TABLE_SHARE share = std::move(*old_share);
  assert(share.ha_share == nullptr);

  share.db_plugin = ha_lock_engine(thd, innodb_hton);

  Table_ref *const wtable_list = wtable->pos_in_table_list;
  Derived_refs_iterator ref_it(wtable_list);

  if (wtable_list) {
    Common_table_expr *cte = wtable_list->common_table_expr();
    if (cte) {
      int i = 0, found = -1;
      TABLE *t;
      while ((t = ref_it.get_next())) {
        if (t == wtable) {
          found = i;
          break;
        }
        ++i;
      }
      assert(found >= 0);
      if (found > 0)
        // 'wtable' is at position 'found', move it to 0 to convert it first
        std::swap(cte->tmp_tables[0], cte->tmp_tables[found]);
      ref_it.rewind();
    }
  }
```

**关键点**：

- **磁盘引擎硬编码 InnoDB**——不是参数选择。这解释了为什么错误信息里还留着 `ER_SWITCH_TMP_ENGINE_MSG`（"requires @@internal_tmp_disk_storage_engine=InnoDB"）：那个参数已不存在，只剩这条消息
- **CTE 的共享临时表**：先在 `cte->tmp_tables` 里找到 `wtable` 并**换到首位**，保证它被第一个转换——因为后面要遍历所有引用者

转换与逐行拷贝：

```cpp
    new_table.s = &share;  // New table points to new share

    new_table.file =
        get_new_handler(&share, false, old_share->alloc_for_tmp_file_handler,
                        new_table.s->db_type());
    ...
    /* Fix row type which might have changed with SE change. */
    set_real_row_type(&new_table);

    if (!table_on_disk) {
      if (create_tmp_table_with_fallback(thd, &new_table))
        goto err_after_alloc; /* purecov: inspected */

      table_on_disk = true;
    }

    if (table->is_created()) {
      // Close it, drop it, and open a new one in the disk-based engine.

      if (open_tmp_table(&new_table))
        goto err_after_create; /* purecov: inspected */
      ...
      if (table == wtable) {
        // The table receiving writes; migrate rows before closing/dropping.

        if (unlikely(thd->opt_trace.is_started())) {
          Opt_trace_context *trace = &thd->opt_trace;
          Opt_trace_object wrapper(trace);
          Opt_trace_object convert(trace, "converting_tmp_table_to_ondisk");
          assert(error == HA_ERR_RECORD_FILE_FULL);
          convert.add_alnum("cause", "memory_table_size_exceeded");
          trace_tmp_table(trace, &new_table);
        }

        table->file->ha_index_or_rnd_end();

        if ((write_err = table->file->ha_rnd_init(true))) { ... }

        if (table->no_rows) {
          new_table.file->ha_extra(HA_EXTRA_NO_ROWS);
          new_table.no_rows = true;
        }

        /*
          copy all old rows from heap table to on-disk table
          This is the only code that uses record[1] to read/write but this
          is safe as this is a temporary on-disk table without timestamp/
          autoincrement or partitioning.
        */
        while (!table->file->ha_rnd_next(new_table.record[1])) {
          write_err = new_table.file->ha_write_row(new_table.record[1]);
          DBUG_EXECUTE_IF("raise_error", write_err = HA_ERR_FOUND_DUPP_KEY;);
          if (write_err) goto err_after_open;
        }
        if (insert_last_record) {
          /* copy row that filled in-memory table */
          if ((write_err = new_table.file->ha_write_row(table->record[0]))) {
```

**关键点**：

- **不是"把内存表刷盘"，而是新建一张 InnoDB 表 → 全表扫描内存表 → 逐行写入新表**——一次 O(已插入行数) 的拷贝
- 触发转换的那一行（把表撑满的那行）单独用 `insert_last_record` 补写（`table->record[0]`）
- **CTE 共享表要连所有克隆一起转**：外层 `while (true)` 遍历 `Derived_refs_iterator`，把每个引用者都转到新引擎——否则有的引用指向内存表、有的指向磁盘表就乱了
- 转换会写入 optimizer trace：`converting_tmp_table_to_ondisk` / `cause: memory_table_size_exceeded`——**排查"为什么突然变慢"的关键证据**

### 执行期：物化、聚合与共享

#### 三种 op_type

`make_tmp_tables_info()` 决定建哪种临时表。`QEP_TAB::enum_op_type` 区分：

| op_type | 语义 | 迭代器 |
|---|---|---|
| `OT_MATERIALIZE` | 写入临时表，不聚合，再扫描 | `MaterializeIterator` |
| `OT_AGGREGATE_THEN_MATERIALIZE` | 先流式聚合，再物化结果 | `AggregateIterator` + 物化 |
| `OT_AGGREGATE_INTO_TMP_TABLE` | 直接在临时表上分组聚合 | `TemptableAggregateIterator` |

#### TemptableAggregateIterator：边读边分组

核心是"读源行 → 查重 → 更新聚合或插新行"：

```cpp
for (;;) {
  int read_error = m_subquery_iterator->Read();     // 读源行
  ...
  group_found = ...ha_index_read_map(...HA_READ_KEY_EXACT);  // 查 group
  if (group_found) {
    update_tmptable_sum_func(...);                  // 更新聚合函数
    int error = table()->file->ha_update_row(...);
    if (error != 0 && error != HA_ERR_RECORD_IS_THE_SAME)
      if (move_table_to_disk(error, false)) return true;  // 满 → InnoDB
  } else {
    int error = table()->file->ha_write_row(...);   // 新 group 行
    if (error != 0)
      if (move_table_to_disk(error, true)) return true;
  }
}
```

`Read()` 直接代理给 `m_table_iterator`（扫描结果临时表）。注意每处写操作失败都接 `move_table_to_disk`——降级可能在任意一次写入时触发。

#### MaterializeIterator

`Init()` 里：`rematerialize=false` 且已物化时只重扫；否则 `instantiate_tmp_table` 建表 → 逐个 query block `MaterializeQueryBlock` 物化 → 之后扫描。

这符合火山模型"重活在 `Init()`"的规律（排序、物化都在 `Init`）。

#### 共享物化（CTE / derived 被多次引用）

同一张物化表被多个地方引用时，不是各建一张，而是用 `clone_tmp_table` 造指向同一份数据的"克隆"引用——所以 CTE 只物化一次。降级时也正因如此，才需要把**所有克隆**一起转到新引擎（见上面代码）。

递归 CTE 特殊在：一边写入一边读取（用不同的 TABLE 引用区分读写）。

### 场景全景：哪些算子会用到内部临时表

| 场景 | 用途 |
|---|---|
| `GROUP BY`（无法用索引顺序时） | 分组键 + 聚合结果暂存 |
| `DISTINCT` | 唯一键去重 |
| `ORDER BY` + `GROUP BY` 不同字段 | 先物化分组，再排序 |
| `UNION`（非 ALL） | 全列唯一键去重 |
| derived table / view 物化 | 子查询结果暂存 |
| CTE（含递归） | 物化并可被多引用共享 |
| 窗口函数 | 分区数据暂存 |
| semi-join **DuplicateWeedout** | rowid 唯一键去重 |
| ROLLUP | 多层分组 |
| 子查询物化（`subselect_hash_sj_engine`） | 建 hash 表/临时表 |
| filesort 的 rowid 模式 | 排序结果存 rowid（落盘时也借临时表机制） |

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `internal_tmp_mem_storage_engine` | **TempTable** | SESSION（HINT_UPDATEABLE） | 内存内部临时表引擎，枚举仅 `MEMORY` / `TempTable` |
| `temptable_max_ram` | **1 GiB** | GLOBAL（范围 2MiB ~ 无限） | TempTable **全局**可从 RAM 分配的字节上限；超过后改用 mmap 文件 |
| `temptable_max_mmap` | **1 GiB** | GLOBAL | mmap 分配上限；超过则分配失败（`RECORD_FILE_FULL`） |
| `temptable_use_mmap` | **true** | GLOBAL | 超过 `temptable_max_ram` 后是否用 mmap 继续（关掉则直接降级到 InnoDB） |
| `tmp_table_size` | **16 MB** | SESSION | 内存临时表的**每表**大小上限（两个引擎都看它） |
| `max_heap_table_size` | **16 MB** | SESSION | **仅对 MEMORY 引擎生效**，与 `tmp_table_size` 取较小者 |
| `big_tables` | **false** | SESSION | 开启则强制临时表直接用磁盘（跳过内存） |

> ⚠️ **作用域与生效范围**：`temptable_max_ram` / `temptable_max_mmap` 是**全局共享预算**；`tmp_table_size` / `max_heap_table_size` 是**每会话每表**限制，且 `max_heap_table_size` 只对 MEMORY 生效。

### 状态变量

| 变量名 | 含义 |
|---|---|
| `Created_tmp_tables` | 创建的内部临时表总数 |
| `Created_tmp_disk_tables` | **落到磁盘的**临时表数（性能排查首选） |
| `Created_tmp_files` | 创建的临时文件数 |

排查思路：`Created_tmp_disk_tables` 占比高 ⇒ 调大 `temptable_max_ram` / `tmp_table_size`，或改写查询（加索引避免临时表、减少 DISTINCT/GROUP BY 的列宽）。optimizer trace 里的 `converting_tmp_table_to_ondisk` 能定位是哪一步触发的。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Internal Temporary Table Use in MySQL*（官方列举了哪些场景会用内部临时表）
- *MySQL 8.0 Reference Manual → The TempTable Storage Engine*

**相关文档**
- filesort 排序算法见 [`01_filesort.md`](01_filesort.md)
- 物化执行（子查询惰性物化）见 [`02_subquery_runtime.md`](02_subquery_runtime.md)
- CTE 的共享物化与 `clone_tmp_table` 见 [`03_cte.md`](03_cte.md)
- semi-join 的 DuplicateWeedout 去重表见 [`../07_optimize/logical/03_semijoin.md`](../07_optimize/logical/03_semijoin.md)
- 优化期决定临时表（`make_tmp_tables_info`）见 [`../07_optimize/10_plan_refinement.md`](../07_optimize/10_plan_refinement.md)

**社区文章**
- 阿里内核月报：*[MySQL · 引擎特性 · 临时表改进（weixiang）](http://mysql.taobao.org/monthly/2019/09/01/)*
- 阿里内核月报：*[MySQL · 引擎特性 · 临时表那些事儿（韩逸）](http://mysql.taobao.org/monthly/2019/04/01/)*
- [MySQL 5.7: InnoDB Intrinsic Tables（Krunal Bauskar）](https://dev.mysql.com/blog-archive/mysql-5-7-innodb-intrinsic-tables/) —— intrinsic 表（关闭锁与 redo）的设计动机
- *MySQL 8.0 Reference Manual → [Internal Temporary Table Use in MySQL](https://dev.mysql.com/doc/refman/8.0/en/internal-temporary-tables.html)*

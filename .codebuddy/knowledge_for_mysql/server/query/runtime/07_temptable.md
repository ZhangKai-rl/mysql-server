# 07 内部临时表深度解析

> 基于 MySQL 8.0.39 源码。本篇回答：**MySQL 为什么需要一个"临时表"层？两个内存引擎（TempTable / MEMORY）各是怎么实现的？内存不够时怎么一层层退到磁盘？**
>
> **边界**：本篇讲**内部临时表**这个通用载体（两个引擎的实现、建表设计、阈值与降级链、物化时的使用）。filesort 排序算法见 [`01_filesort.md`](01_filesort.md)；各场景为什么需要物化见对应篇（CTE、窗口、semi-join 等）。

## 目录

- [概述](#概述)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
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

**TempTable 的行存储与索引的分层关系**（理解全篇的主干图）：

```
                        temptable::Table （一张内部临时表）
                                  │
              ┌───────────────────┴───────────────────┐
              │                                       │
      Storage m_rows（行容器）                  m_index_entries（索引）
      按 64 KiB 分页，元素是：                   每个 KEY 一个 Index 子类
      ┌────────────────────────┐              ┌──────────────────────────┐
      │ 全定长：裸 MySQL 行      │              │ HA_KEY_ALG_BTREE → Tree   │
      │   element_size=          │              │   std::multiset<          │
      │   rec_buff_length        │              │     Indexed_cells, ...>    │
      │   memcpy 整行            │              │ HA_KEY_ALG_HASH + NOSAME  │
      ├────────────────────────┤              │   → Hash_unique            │
      │ 含变长：Row 对象         │              │   std::unordered_set      │
      │   element_size=          │              │ HA_KEY_ALG_HASH 非唯一     │
      │   sizeof(Row)            │              │   → Hash_duplicates       │
      │   Row → 自有 buffer      │              │   std::unordered_multiset │
      │   [len][Cell×N][数据]    │              └──────────────────────────┘
      └────────────────────────┘                        │
                                                        │ 索引项 = Indexed_cells
                                                        │ union { mysql_buf | Row* }
                                                        ▼
                                              回指整行（不是 key→value 映射）

      Allocator<uint8_t> —— 所有内存的唯一出口
        Block（1→2→4→...→512 MiB 指数增长）
          ├─ Source::RAM       （≤ temptable_max_ram）
          └─ Source::MMAP_FILE （≤ temptable_max_mmap）
                              └ 都放不下 → throw Result::RECORD_FILE_FULL
```

**降级的状态流转**：

```
   [RAM]  ──RAM 用量 < temptable_max_ram──▶  继续用 RAM
     │
     │ 块分配时 RAM 试算失败
     ▼
   [MMAP] ──MMAP 用量 < temptable_max_mmap──▶ 继续用 mmap
     │
     │ 块分配时 MMAP 也试算失败
     ▼
   throw Result::RECORD_FILE_FULL （= HA_ERR_RECORD_FILE_FULL）
     │
     ├── 发生在 create 阶段 → create_tmp_table_with_fallback() 换 innodb_hton 重建
     │                        ★ Created_tmp_disk_tables++
     └── 发生在 insert/update 阶段 → create_ondisk_from_heap()
                                     新建 InnoDB 表 + ha_rnd_next 逐行拷贝 + 换 handler
                                     ★ 所有 CTE 克隆的 handler 一起换

   注：RAM 用量回落后会退回 RAM（水位回落，非单调）
```

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

**★ 行的物理布局**（`storage/temptable/include/temptable/row.h`，取自源码注释）：

```
一条 Row 的自有 buffer（m_ptr）结构：
  [0, A = sizeof(size_t))                        : buffer 总长 buf_length
  [A, B = A + number_of_cells * sizeof(Cell))    : Cell 数组（头部）
  [B, B + sum(每个 cell 的 user data length))     : 用户数据拼接区（尾部）
```

单个 `Cell` 就是注释里那个 POD：

```cpp
struct Cell {
  bool is_null;      // 1 byte
  uint32_t len;      // 4 bytes
  void *data;        // 8 bytes on 64-bit
};
```

**★ 关键结论：长度与指针在"头部"（Cell 数组），真实数据在"尾部"**。变长列只占真实字节——不补空格、不补 `0xa5`。每行每列固定 16 字节 Cell 开销。

**★ 定长表根本不构造 Row** —— 这是定长/变长的真正分叉，也是"零拷贝"的实质：

```cpp
/* storage/temptable/src/table.cc —— 构造时按是否全定长决定 Storage 元素大小 */
  if (m_all_columns_are_fixed_size) {
    m_rows.element_size(m_mysql_row_length);          // ★ 元素 = 裸 MySQL 行
    assert(m_rows.number_of_elements_per_page() > 0);
  } else {
    m_rows.element_size(sizeof(Row));                 // ★ 元素 = Row 对象
  }
```

```cpp
/* storage/temptable/src/table.cc —— Table::insert()，写入路径 */
Result Table::insert(const unsigned char *mysql_row) {
  Storage::Element *row;
  try {
    row = m_rows.allocate_back();
  } catch (Result ex) {
    return ex;
  }

  Result ret;

  if (m_all_columns_are_fixed_size) {
    assert(m_rows.element_size() == m_mysql_table_share->rec_buff_length);
    assert(m_rows.element_size() == m_mysql_row_length);

    memcpy(row, mysql_row, m_mysql_row_length);       // ★ 整行一次 memcpy，无 Row、无 Cell
  } else {
    assert(m_rows.element_size() == sizeof(Row));

    new (row) Row(mysql_row, &m_allocator);           // ★ 轻量态：只存指针，不拷数据

    ret = static_cast<Row *>(row)->copy_to_own_memory(m_columns,
                                                      m_mysql_row_length);
    if (ret != Result::OK) {                          // ★ 分配失败（RAM/MMAP 都满）
      static_cast<Row *>(row)->~Row();
      m_rows.deallocate_back();
      return ret;                                     //    → RECORD_FILE_FULL 上抛
    }
  }

  if (!indexed()) return Result::OK;

  ret = indexes_insert(row);
  if (ret != Result::OK) {                            // 多索引插入失败要回滚
    if (!m_all_columns_are_fixed_size) static_cast<Row *>(row)->~Row();
    m_rows.deallocate_back();
  }
  return ret;
}
```

**定长行连索引项都不复制 key 数据**——索引项里只存 8 字节指针：

```cpp
/* storage/temptable/src/table.cc —— Table::indexes_insert() */
    Indexed_cells indexed_cells =
        m_all_columns_are_fixed_size
            ? Indexed_cells{static_cast<unsigned char *>(row), *index}   // 裸行指针
            : Indexed_cells{*static_cast<Row *>(row), *index};           // Row 指针
```

**哪些类型算变长**（注意 `CHAR` 是定长，只有 `VARCHAR` 算变长）：

```cpp
/* storage/temptable/include/temptable/handler.h */
inline bool Handler::is_field_type_fixed_size(const Field &mysql_field) const {
  switch (mysql_field.type()) {
    case MYSQL_TYPE_BLOB:
    case MYSQL_TYPE_GEOMETRY:
    case MYSQL_TYPE_JSON:
    case MYSQL_TYPE_LONG_BLOB:
    case MYSQL_TYPE_MEDIUM_BLOB:
    case MYSQL_TYPE_TINY_BLOB:
    case MYSQL_TYPE_VARCHAR:
      return false;
    default:
      return true;                                    // ★ CHAR(MYSQL_TYPE_STRING) 在此算定长
  }
}
```

**变长行落自有内存的全过程**（一次整块分配 + placement new 逐个 Cell，`data_ptr` 顺序推进）：

```cpp
/* storage/temptable/src/row.cc —— Row::copy_to_own_memory() */
Result Row::copy_to_own_memory(const Columns &columns,
                               size_t mysql_row_length) const {
  assert(m_data_is_in_mysql_memory);
  const unsigned char *mysql_row = m_ptr;

  size_t buf_len = sizeof(size_t);                    // 头部先留 buf_length
  for (const auto &column : columns) {
    buf_len += sizeof(Cell) + column.read_user_data_length(mysql_row);
  }

  try {
    m_ptr = m_allocator->allocate(buf_len);           // ★ 一次分配整行（可能抛 RECORD_FILE_FULL）
  } catch (Result ex) {
    return ex;
  }

  *reinterpret_cast<size_t *>(m_ptr) = buf_len;
  m_data_is_in_mysql_memory = false;

  Cell *cell = cells();                               // = m_ptr + sizeof(size_t)
  /* User data begins after the cells array. */       // ★ 数据在 Cell 数组之后
  unsigned char *data_ptr =
      reinterpret_cast<unsigned char *>(cell + columns.size());

  for (const auto &column : columns) {
    const bool is_null = column.read_is_null(mysql_row);
    const uint32_t data_length = column.read_user_data_length(mysql_row);

    if (data_length > 0) {
      column.read_user_data(data_ptr, data_length, mysql_row, mysql_row_length);
    }
    new (cell) Cell{is_null, data_length, data_ptr};
    ++cell;
    data_ptr += data_length;
  }
  return (Result::OK);
}
```

**Row 的两个状态由 `m_data_is_in_mysql_memory` 一个 bool 表达**，见类内注释：

```cpp
/* storage/temptable/include/temptable/row.h —— Row 私有成员注释 */
  /** A pointer to either the mysql row, or our buffer. If
   * - `m_data_is_in_mysql_memory` is true, then this points to a buffer in
   *    mysql write_row() format, not owned by the current Row object;
   * - `m_data_is_in_mysql_memory` is false, then this points a our own buffer
   *    that holds the cells and the user data. Its structure is:
   *    [0, A = sizeof(size_t)): buffer length
   *    [A, B = A + number_of_cells * sizeof(Cell)): cells array
   *    [B, B + sum(user data length for each cell)): user data of the cells */
  mutable unsigned char *m_ptr;
```

轻量态下 `cell()` 用 `Column` 现场解释 MySQL 行格式，产出 `Cell{is_null, len, ptr}`，全程**无 malloc、无 memcpy**：

```cpp
inline Cell Row::cell(const Column &column, size_t i) const {
  if (m_data_is_in_mysql_memory) return cell_in_mysql_memory(column);
  else                           return cell_in_row(i);
}

inline Cell Row::cell_in_mysql_memory(const Column &column) const {
  const bool is_null = column.read_is_null(m_ptr);
  const uint32_t data_length = column.read_user_data_length(m_ptr);
  const unsigned char *data_ptr = column.get_user_data_ptr(m_ptr);
  return Cell{is_null, data_length, data_ptr};
}
```

**`Column` 是 MySQL 行格式的解释器**——变长列的长度字节数可以是 1/2/3/4，`m_length_bytes_size == 0` 即定长：

```cpp
/* storage/temptable/include/temptable/column.h —— Column 成员 */
  /** Bitmask to extract is is-NULL bit from the is-NULL byte. */
  uint8_t m_null_bitmask;
  /** The number of bytes that indicate the length of the user data in the
   * cell, for variable sized cells. If this is 0, then the cell is fixed size. */
  uint8_t m_length_bytes_size;

  union {
    /** 定长列：直接是长度 */
    uint32_t m_length;
    /** 变长列：长度字节在行内的偏移 */
    uint32_t m_offset;
  };
```

```cpp
inline uint32_t Column::read_user_data_length(const unsigned char *mysql_row) const {
  if (m_length_bytes_size == 0) return m_length;      // 定长
  // data_length may contain junk value while particular cell is NULL
  if (read_is_null(mysql_row)) return 0;
  const unsigned char *p = mysql_row + m_offset;
  switch (m_length_bytes_size) {
    case 1: return *p;
    case 2: return *p | (*(p + 1) << 8);
    case 3: return *p | (*(p + 1) << 8) | (*(p + 2) << 16);
    case 4: return *p | (*(p + 1) << 8) | (*(p + 2) << 16) | (*(p + 3) << 24);
  }
  my_abort();
  return 0;
}
```

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

**★ 铁证：容器就是三个 typedef**（`storage/temptable/include/temptable/containers.h`）：

```cpp
#include <set>            /* std::multiset */
#include <unordered_set>  /* std::unordered_set, std::unordered_multiset */

/** The container used by tree unique and non-unique indexes. */
typedef std::multiset<Indexed_cells, Indexed_cells_less,
                      Allocator<Indexed_cells>>
    Tree_container;

/** The container used by hash non-unique indexes. */
typedef std::unordered_multiset<Indexed_cells, Indexed_cells_hash,
                                Indexed_cells_equal_to,
                                Allocator<Indexed_cells>>
    Hash_duplicates_container;

/** The container used by hash unique indexes. */
typedef std::unordered_set<Indexed_cells, Indexed_cells_hash,
                           Indexed_cells_equal_to, Allocator<Indexed_cells>>
    Hash_unique_container;
```

两点值得注意：

1. **是 `set` 不是 `map`**——元素类型就是 `Indexed_cells` 本身。索引项自带回指整行的指针（`union { const unsigned char *m_mysql_buf; const Row *m_row; }`），通过 `Indexed_cells::row()` 取回整行，所以不需要 key→value 映射。
2. **BTREE 就是 libstdc++ 的红黑树**（`std::multiset` 底层 `_Rb_tree`），不是 B+ 树；HASH 就是桶 + 链（`unordered_(multi)set`）。

`Tree::insert` 用 `lower_bound` 判重，唯一性冲突返回 `FOUND_DUPP_KEY`：

```cpp
/* storage/temptable/src/index.cc */
Result Tree::insert(const Indexed_cells &indexed_cells, Cursor *insert_position) {
  bool ok_to_insert;

  if (m_allow_duplicates) {                    // m_allow_duplicates = !(flags & HA_NOSAME)
    ok_to_insert = true;
  } else {
    /* Duplicates not allowed. See if `indexed_cells` is already in the tree. */
    auto pos = m_tree.lower_bound(indexed_cells);
    if (pos == m_tree.end()) {
      ok_to_insert = true;                     // 所有元素都比它小
    } else if (m_tree.key_comp()(indexed_cells, *pos)) {
      ok_to_insert = true;                     // indexed_cells < *pos
    } else {
      ok_to_insert = false;                    // indexed_cells == *pos → 重复
    }
  }

  if (ok_to_insert) {
    Container::iterator it;
    try {
      it = m_tree.emplace(indexed_cells);
    } catch (Result ex) {
      return ex;                               // ★ 内存不足由此上抛
    }
    *insert_position = Cursor(it);
    return Result::OK;
  }
  return Result::FOUND_DUPP_KEY;               // ★ 去重就是靠这个返回值
}
```

`Hash_unique::insert` 更简单——直接靠 `emplace` 返回的 `bool`：

```cpp
Result Hash_unique::insert(const Indexed_cells &indexed_cells,
                           Cursor *insert_position) {
  std::pair<Container::iterator, bool> r;
  try {
    r = m_hash_table.emplace(indexed_cells);
  } catch (Result ex) {
    return ex;
  }
  const bool new_element_inserted = r.second;
  if (!new_element_inserted) return Result::FOUND_DUPP_KEY;
  *insert_position = Cursor(r.first);
  return Result::OK;
}
```

**Hash 索引不支持范围扫描**——`unordered_multiset` 无序，所以 `HA_READ_KEY_OR_NEXT` / `HA_READ_PREFIX_LAST` 直接判 `UNSUPPORTED`：

```cpp
/* storage/temptable/src/handler.cc —— Handler::index_read() */
      case HA_READ_KEY_OR_NEXT:
        if (handler::table->s->key_info[handler::active_index].algorithm !=
            HA_KEY_ALG_BTREE) {
          ret = Result::UNSUPPORTED;           // ★ HASH 索引只能等值查找
          break;
        }
```

引擎能力声明也印证这点（`Handler::index_flags`）：BTREE 给 `HA_READ_NEXT|HA_READ_PREV|HA_READ_ORDER|HA_READ_RANGE`，HASH 只给 `HA_READ_NEXT|HA_ONLY_WHOLE_INDEX`。

**比较与哈希的最终裁决在 `Cell_calculator`**：三种模式（BINARY / CHARSET / CHARSET_AND_CHAR_LENGTH），并且要处理 `PAD_SPACE` 与 `NO_PAD` 排序规则的差异——`'a'` 与 `'a '` 在 `NO_PAD` 下必须判等：

```cpp
/* storage/temptable/include/temptable/cell_calculator.h —— Cell_calculator::compare() 关键片段 */
  if (lhs.is_null()) {
    if (rhs.is_null()) return 0;               // 都是 NULL
    else               return -1;              // NULL < 非 NULL
  } else if (rhs.is_null()) {
    return 1;
  }

  /* 快速路径：长度相同且 memcmp 相等 → 直接判等，不走昂贵的 collation 比较 */
  if ((lhs_data_length == rhs_data_length) &&
      ((lhs_data_length == 0) ||
       (memcmp(lhs.data(), rhs.data(), lhs_data_length) == 0))) {
    return 0;
  }
  ...
  /* If the field is space padded but collation do not want to use
   * the padding it is required to strip the spaces from the end. */
  if (m_is_space_padded && (m_cs->pad_attribute == NO_PAD)) {
    lhs_length = m_cs->cset->lengthsp(m_cs, (const char *)lhs_data, lhs_length);
    rhs_length = m_cs->cset->lengthsp(m_cs, (const char *)rhs_data, rhs_length);
  }
  return m_cs->coll->strnncollsp(m_cs, lhs_data, lhs_length,
                                 rhs_data, rhs_length);
```

哈希函数的选择刻意与 MEMORY 对齐——源码注释直白写着 **"Mimic hp_hashnr() from storage/heap/hp_hash.c"**，即为了让两套引擎的语义一致：

```cpp
inline Cell_calculator::Cell_calculator(const KEY_PART_INFO &mysql_key_part)
    : m_mysql_field(mysql_key_part.field),
      m_cs(field_charset(*m_mysql_field)),
      m_is_space_padded(m_mysql_field->key_type() == HA_KEYTYPE_TEXT),
      m_char_length(0) {
  /* Mimic hp_hashnr() from storage/heap/hp_hash.c. */
  if (m_cs != nullptr) {
    bool use_char_length = (m_cs->mbmaxlen > 1) &&
                           (mysql_key_part.key_part_flag & HA_PART_KEY_SEG);
    if (use_char_length) {                     // 多字节字符集 + 前缀索引 → 按字符数截断
      m_char_length = mysql_key_part.length / m_cs->mbmaxlen;
      m_mode = Mode::CHARSET_AND_CHAR_LENGTH;
    } else {
      m_mode = Mode::CHARSET;
    }
  } else {
    m_mode = Mode::BINARY;
  }
}
```

**索引项的比较器只有一行**（`lhs.compare(rhs) < 0`），前缀索引的处理在 `cell_from_row()` 里把长度截断到 `prefix_length()`：

```cpp
  /* Lower row_cell.data_length() in case we have a prefix index, e.g.:
   * CREATE TABLE t (c CHAR(16), INDEX c(10)); */
  const uint32_t data_length =
      std::min(row_cell.data_length(), indexed_column.prefix_length());
```

`compare()` 只比较 `min(lhs_num, rhs_num)` 个 cell——**这正是前缀搜索（prefix search）能工作的原因**，注释里写得很清楚。

#### MEMORY：定长行 + hash/BTREE

MEMORY（`ha_heap` 家族）是 MySQL 最老的引擎之一：

- 行**定长**（`hp_create` 里按 `reclength` 计算，注释提到 "so the record length should be at least sizeof(uchar*)"）
- 支持 hash（`hp_hash`）与 BTREE（`hp_rkey`）索引
- **不支持 BLOB/TEXT**（见引擎选择）
- 不支持 BIT 字段索引（见前面的 `MARKER_BIT` 注释）

**定长行的铁证**（`storage/heap/hp_write.cc`）——写入就是 `memcpy` 固定 `reclength`，行尾多 1 字节做"未删除"标记：

```cpp
int heap_write(HP_INFO *info, const uchar *record) {
  ...
  if (!(pos = next_free_record_pos(share))) return my_errno();   // ★ 表满 → HA_ERR_RECORD_FILE_FULL
  share->changed = 1;

  for (keydef = share->keydef, end = keydef + share->keys; keydef < end; keydef++) {
    if ((*keydef->write_key)(info, keydef, record, pos)) goto err;
  }

  memcpy(pos, record, (size_t)share->reclength);   // ★ 定长：整行按 reclength 拷贝
  pos[share->reclength] = 1;                       // Mark record as not deleted
  ...
}
```

行存储块在 `hp_create.cc` 里按 `reclength + 1` 初始化，每个槽位大小恒定：

```cpp
/* storage/heap/hp_create.cc */
  recbuffer = (uint)(reclength + sizeof(uchar **) - 1) & ~(sizeof(uchar **) - 1);
  block->recbuffer = recbuffer;                    // ★ 每个记录槽固定 recbuffer 字节
```

**两种索引由两个函数指针分派**（`HP_KEYDEF::write_key`）：BTREE 走红黑树，HASH 走自实现的桶 + 链。

```cpp
/* storage/heap/hp_write.cc —— BTREE 索引插入：就是一棵 TREE（红黑树） */
int hp_rb_write_key(HP_INFO *info, HP_KEYDEF *keyinfo, const uchar *record,
                    uchar *recpos) {
  heap_rb_param custom_arg;
  custom_arg.keyseg = keyinfo->seg;
  custom_arg.key_length = hp_rb_make_key(keyinfo, info->recbuf, record, recpos);
  if (keyinfo->flag & HA_NOSAME) {
    custom_arg.search_flag = SEARCH_FIND | SEARCH_UPDATE;
    keyinfo->rb_tree.flag = TREE_NO_DUPS;          // ★ 唯一键靠 TREE_NO_DUPS，插入失败→DUPP_KEY
  } else {
    custom_arg.search_flag = SEARCH_SAME;
    keyinfo->rb_tree.flag = 0;
  }
  if (!tree_insert(&keyinfo->rb_tree, (void *)info->recbuf,
                   custom_arg.key_length, &custom_arg)) {
    set_my_errno(HA_ERR_FOUND_DUPP_KEY);
    return 1;
  }
  ...
}
```

HASH 索引的结构（`hp_write.cc` 的注释即为权威说明）：

```
Hash index uses HP_BLOCK structure as a 'growable array' of HASH_INFO structs.
hp_mask(hp_rec_hashnr()) maps hash entries values to hash array positions.
If there are several hash entries with the same hash array position P,
they are connected in a linked list via HASH_INFO::next_key.
```

**key 的元数据结构是 `HP_KEYDEF`（含 `HA_KEYSEG *seg`）与 `HA_KEYSEG`**，不是什么 `HEAP_KEY`：

```cpp
/* include/heap.h */
struct HP_KEYDEF /* Key definition with open */
{
  uint flag{0};       /* HA_NOSAME | HA_NULL_PART_KEY */
  uint keysegs{0};    /* Number of key-segment */
  uint length{0};     /* Length of key (automatic) */
  uint8 algorithm{0}; /* HASH / BTREE */
  HA_KEYSEG *seg{nullptr};
  HP_BLOCK block;     /* Where keys are saved */
  ha_rows hash_buckets{0};
  TREE rb_tree;
  int (*write_key)(HP_INFO *info, HP_KEYDEF *keyinfo, const uchar *record, uchar *recpos){nullptr};
  int (*delete_key)(HP_INFO *info, HP_KEYDEF *keyinfo, const uchar *record, uchar *recpos, int flag){nullptr};
  uint (*get_key_length)(HP_KEYDEF *keydef, const uchar *key){nullptr};
};
```

**MEMORY 的"满"是每表上限，直接抛 `HA_ERR_RECORD_FILE_FULL`**（没有 mmap 中间层）：

```cpp
/* storage/heap/hp_write.cc —— next_free_record_pos() */
  if (!(block_pos = (info->records % info->block.records_in_block))) {
    if ((info->records > info->max_records && info->max_records) ||
        (info->data_length + info->index_length >= info->max_table_size)) {
      set_my_errno(HA_ERR_RECORD_FILE_FULL);        // ★ 直接从内存跳到 InnoDB
      return nullptr;
    }
    ...
```

`max_table_size` 在 `ha_heap.cc` 里取自 `max_heap_table_size`，并据此反推 `max_records`：

```cpp
  hp_create_info->max_table_size = current_thd->variables.max_heap_table_size;
  max_rows = (ha_rows)(hp_create_info->max_table_size / mem_per_row);
  if (share->max_rows && share->max_rows < max_rows) max_rows = share->max_rows;
  hp_create_info->max_records = (ulong)max_rows;
```

其中 `mem_per_row` 已把索引开销算进去——HASH 每行 `sizeof(HASH_INFO)`，BTREE 每行 `sizeof(TREE_ELEMENT) + key_length + sizeof(char*)`：

```cpp
      case HA_KEY_ALG_HASH:
        keydef[key].algorithm = HA_KEY_ALG_HASH;
        mem_per_row += sizeof(HASH_INFO);
        break;
      case HA_KEY_ALG_BTREE:
        keydef[key].algorithm = HA_KEY_ALG_BTREE;
        mem_per_row += sizeof(TREE_ELEMENT) + pos->key_length + sizeof(char *);
        break;
```

> ⚠️ **一处常见误传**：说"MEMORY 不支持变长列"。准确说法是——MEMORY **支持 VARCHAR**（`ha_heap.cc` 里 `HA_KEYTYPE_VARTEXT1/2` 有专门分支），但**行存储本身是定长 `reclength`**（VARCHAR 按最大长度占位），且 **BLOB/TEXT 不支持**（`share->blob_fields != 0` 时 SQL 层直接改选磁盘引擎）。

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

> ★ `Field` 的完整剖析（含由 Item 造列的三种路径 `create_tmp_field()` / `create_tmp_field_from_field()` / `create_tmp_field_from_item()`）见 [`../../field.md`](../../field.md) 5.1.6。本节只讲**临时表视角**——字段与键如何设计。

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

**插入隐藏字段的唯一原语**——注意它把所有已有字段的索引 `+1`，然后往 `table->field[-1]` 那个预留前导空位里塞：

```cpp
/* sql/sql_tmp_table.cc —— register_hidden_field() */
static void register_hidden_field(TABLE *table, Field **default_field,
                                  Field **from_field, uint *blob_field,
                                  Field *field) {
  uint i;
  Field **tmp_field = table->field;

  /* Increase all of registered fields index */
  for (i = 0; i < table->s->fields; i++)
    tmp_field[i]->set_field_index(tmp_field[i]->field_index() + 1);

  // Increase the field_index of visible blob field
  for (i = 0; i < table->s->blob_fields; i++) blob_field[i]++;
  // Insert field
  table->field[-1] = field;                     /* ★ 用预留的 extra_fields 前导空位 */
  default_field[-1] = nullptr;
  from_field[-1] = nullptr;
  field->table = table;
  field->set_field_index(0);

  // Keep the field from being expanded by SELECT *.
  field->set_hidden(dd::Column::enum_hidden_type::HT_HIDDEN_SQL);
}
```

最后一行是关键：`HT_HIDDEN_SQL` 使这些列**不被 `SELECT *` 展开**（与 GIPK 用的 `HT_HIDDEN_USER` 不同——后者 `SELECT *` 也看不见但显式引用可见，详见 [`../../../index/gipk.md`](../../../index/gipk.md)）。

**两种隐藏字段的真实创建点**（`<set counter>` 与 `<hash_field>`，都是 `Field_longlong` 且强制 NOT NULL）：

```cpp
  if (unique_constraint_via_hash_field) {
    if (param->needs_set_counter()) {            // ★ 仅 INTERSECT / EXCEPT
      Field_longlong *set_counter = new (&share->mem_root)
          Field_longlong(sizeof(ulonglong), false, "<set counter>", true);
      set_counter->set_flag(NOT_NULL_FLAG);
      register_hidden_field(table, &default_field[0], &from_field[0],
                            share->blob_field, set_counter);
      // Repoint arrays
      table->field--;
      default_field--;
      from_field--;
      from_item--;
      share->reclength += set_counter->pack_length();
      share->fields = ++fieldnr;
      param->hidden_field_count++;
      share->field--;
      table->set_set_counter(
          set_counter, param->m_operation == Temp_table_param::TTP_EXCEPT);
      table->set_distinct(param->m_last_operation_is_distinct);
    }

    Field_longlong *field = new (&share->mem_root)
        Field_longlong(sizeof(ulonglong), false, "<hash_field>", true);
    field->set_flag(NOT_NULL_FLAG);
    register_hidden_field(table, &default_field[0], &from_field[0],
                          share->blob_field, field);
    // Repoint arrays
    table->field--;
    default_field--;
    from_field--;
    from_item--;
    share->reclength += field->pack_length();
    share->fields = ++fieldnr;
    param->hidden_field_count++;
    share->field--;
    table->hash_field = field;                   // ★ 后续 check_unique_constraint 用
  }
```

**"隐藏"的另一层含义**：GROUP BY / HAVING / ORDER BY 里那些不在 SELECT list 的列也算隐藏字段，由 `param->hidden_field_count` 计数，在字段循环里以 `hidden_field_count--` 倒计数分隔——归零那一刻就是"隐藏段结束、可见段开始"的边界：

```cpp
    hidden_field_count--;
    if (hidden_field_count == 0) {
      /* This was the last hidden field; Remember how many hidden fields could have null */
      hidden_null_count = null_count;
      param->hidden_field_count = fieldnr;
      null_count = 0;
      hidden_uneven_bit_length = total_uneven_bit_length;
      total_uneven_bit_length = 0;
    }
  }  // end of for
```

这个边界直接影响 NULL bitmap 的布局——隐藏段与可见段**分开按字节对齐**，所以有 `hidden_null_pack_length` 与 `null_bytes` 两段：

```cpp
  uint hidden_null_pack_length =
      (hidden_null_count + 7 + hidden_uneven_bit_length) / 8;
  share->null_bytes = (hidden_null_pack_length +
                       (null_count + total_uneven_bit_length + 7) / 8);
  share->reclength += share->null_bytes;
  if (share->reclength == 0) share->reclength = 1;  // Dummy select
```

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

**★ 上面说的"hash 冲突会导致误判重复"其实不准确**——MySQL 并没有接受这个风险。冲突时**会逐列比对**。哈希只是用来定位候选行，真正的判等在 `check_unique_constraint()` 里：

```cpp
/* sql/sql_executor.cc —— check_unique_constraint() */
bool check_unique_constraint(TABLE *table) {
  ulonglong hash;

  if (!table->hash_field) return true;
  if (table->no_keyread) return true;

  if (table->group)
    hash = unique_hash_group(table->group);    // ★ 按 GROUP BY 列算 hash
  else
    hash = unique_hash_fields(table);          // ★ 按所有可见列算 hash

  table->hash_field->store(hash, true);
  int res = table->file->ha_index_read_map(table->record[1],
                                           table->hash_field->field_ptr(),
                                           HA_WHOLE_KEY, HA_READ_KEY_EXACT);
  while (res == 0) {                           // ★ 可能有多个候选（真冲突 or 相同 hash）
    // Check whether records are the same.
    if (!(table->group
              ? group_rec_cmp(table->group, table->record[0], table->record[1])
              : table_rec_cmp(table))) {
      return false;                            // ★ 确认是同一行 → 跳过（去重）
    }
    res = table->file->ha_index_next_same(
        table->record[1], table->hash_field->field_ptr(), sizeof(hash));
  }
  return true;
}
```

`table_rec_cmp()` 逐列比（`cmp_field_value`），所以 **hash 冲突只会多一点比较，不会丢行**：

```cpp
static bool table_rec_cmp(TABLE *table) {
  ptrdiff_t diff = table->record[1] - table->record[0];
  Field **fields = table->visible_field_ptr();
  for (uint i = 0; i < table->visible_field_count(); i++) {
    Field *field = fields[i];
    if (cmp_field_value(field, diff)) return true;
  }
  return false;
}
```

**hash 的具体算法**（`unique_hash()`）：按字段类型分三路——JSON 走 `make_hash_key`、字符串走 collation 的 `hash_sort`、其余按字节滚动累加；NULL 用一段特殊变换以免与空串/0 混淆：

```cpp
ulonglong unique_hash(const Field *field, ulonglong *hash_val) {
  uint64_t seed1 = 0, seed2 = 4;
  uint64_t crc = *hash_val;

  if (field->is_null()) {
    /* Change crc in a way different from an empty string or 0. */
    crc = ((crc << 8) + 511 + (crc >> (8 * sizeof(ha_checksum) - 8)));
    goto finish;
  }

  if (field->type() == MYSQL_TYPE_JSON) {
    crc = down_cast<const Field_json *>(field)->make_hash_key(*hash_val);
  } else if (field->key_type() == HA_KEYTYPE_TEXT ||
             field->key_type() == HA_KEYTYPE_VARTEXT1 ||
             field->key_type() == HA_KEYTYPE_VARTEXT2) {
    const uchar *data_ptr = field->data_ptr();
    if (field->data_length() == 0 && data_ptr == nullptr)
      data_ptr = pointer_cast<const uchar *>(const_cast<char *>(""));
    field->charset()->coll->hash_sort(field->charset(), data_ptr,
                                      field->data_length(), &seed1, &seed2);
    my_hash_combine(crc, seed1);
  } else {
    const uchar *pos = field->data_ptr();
    const uchar *end = pos + field->data_length();
    while (pos != end)
      crc = ((crc << 8) + (*pos++)) + (crc >> (8 * sizeof(ha_checksum) - 8));
  }
finish:
  *hash_val = crc;
  return crc;
}
```

> ⚠️ **别把 `hash_field` 与 `Unique` 类混为一谈**。两者都做去重，但是**两条完全独立的机制**：`hash_field` 是临时表里的隐藏 `Field_longlong` + 1-part 键；`Unique`（`sql/uniques.h`，TREE + 溢出到文件再 merge）是 filesort 与 `Item_sum` 用的另一条路径。

**触发 hash_field 的四类条件**（源码注释 + 三处实际判定）：

```cpp
  if (distinct) {
    if (distinct_key_length > max_key_length ||                   // ① 键太长
        (fieldnr - param->hidden_field_count) > max_key_parts ||  // ② 键列数太多
        param->force_hash_field_for_unique) {                     // ③ 调用方强制
      unique_constraint_via_hash_field = true;
    }
  }
```

④ 是 INTERSECT/EXCEPT（初始值）、GROUP BY 键超长 / 非确定性 / BLOB：

```cpp
      /* Use hash key as the unique constraint if the group-by key is
         big or if it is non-deterministic. Group-by items get evaluated
         twice and a non-deterministic function would cause a discrepancy. */
      if ((*cur_group->item)->max_char_length() > CONVERT_IF_BIGGER_TO_BLOB ||
          (*cur_group->item)->is_non_deterministic()) {
        unique_constraint_via_hash_field = true;
      }
```

键长上限取**内存引擎与 InnoDB 的最小值**（因为随时可能降级，必须两边都放得下）：

```cpp
void get_max_key_and_part_length(uint *max_key_length,
                                 uint *max_key_part_length,
                                 uint *max_key_parts) {
  *max_key_length =
      std::min(Cache_temp_engine_properties::HEAP_MAX_KEY_LENGTH,
               Cache_temp_engine_properties::INNODB_MAX_KEY_LENGTH);
  *max_key_part_length =
      std::min(Cache_temp_engine_properties::HEAP_MAX_KEY_PART_LENGTH,
               Cache_temp_engine_properties::INNODB_MAX_KEY_PART_LENGTH);
  *max_key_parts = std::min(Cache_temp_engine_properties::HEAP_MAX_KEY_PARTS,
                            Cache_temp_engine_properties::INNODB_MAX_KEY_PARTS);
}
```

`force_hash_field_for_unique` 的一个真实调用场景——**混合 UNION DISTINCT / UNION ALL** 不能用唯一键去重，因为走到 UNION ALL 段时要能"关掉"去重检查：

```cpp
  } else if (op->has_mixed_distinct_operators()) {
    /* If we have mixed UNION DISTINCT / UNION ALL, we can't use an unique
       index to deduplicate, as we need to be able to turn off deduplication
       checking when we get to the UNION ALL part. ... Thus, we do it manually
       instead. */
    tmp_table_param.force_hash_field_for_unique = true;
  }
```

#### 各场景的键设计

| 场景 | 键设计 |
|---|---|
| `GROUP BY` | 按 group 列建键（唯一键则实现分组去重），聚合结果存普通列 |
| `DISTINCT` | 按 select 列建唯一键，重复行插入失败（`FOUND_DUPP_KEY`）即丢弃 |
| `UNION`（非 ALL） | **全列唯一键**去重（所以 `UNION` 比 `UNION ALL` 贵就贵在这张表的唯一键维护） |
| DuplicateWeedout（semi-join） | 只有 **rowid** 列 + 唯一键的表，靠 rowid 唯一键去掉重复的外层行（详见 semi-join 篇） |
| CTE / derived 物化 | 通常无键（纯存储），递归 CTE 需特殊处理 |

**键名是写死在源码里的**，按场景分别是 `<group_key>` / `<auto_distinct_key>` / `<hash_group_key>` / `<hash_distinct_key>`——**在 optimizer trace 与 `EXPLAIN` 的 `key` 列里能直接看到**，是排查"临时表为什么慢"的第一手证据：

```sql
-- 走普通键：EXPLAIN 的 key 列显示 <group_key> / <auto_distinct_key>
-- 走 hash：  显示 <hash_group_key> / <hash_distinct_key>
```

GROUP BY 键的构造（够短时）：

```cpp
  if (group) {
    table->group = group;                       /* Table is grouped by key */
    share->keys = 1;
    for (ORDER *cur_group = group; cur_group; cur_group = cur_group->next) {
      Field *field = (*cur_group->item)->get_tmp_table_field();
      cur_group->field_in_tmp_table = field;
      ...
    }
    if (!unique_constraint_via_hash_field) {
      param->keyinfo->flags = HA_NOSAME;        // ★ 唯一键 → 分组去重
      param->keyinfo->usable_key_parts = param->group_parts;
      param->keyinfo->name = "<group_key>";
      for (ORDER *cur_group = group; cur_group;
           cur_group = cur_group->next, key_part_info++) {
        Field *field = cur_group->field_in_tmp_table;
        key_part_info->init_from_field(field);

        /* In GROUP BY 'a' and 'a ' are equal for VARCHAR fields */
        key_part_info->key_part_flag |= HA_END_SPACE_ARE_EQUAL;

        if (key_part_info->store_length > max_key_part_length) {
          unique_constraint_via_hash_field = true;   // ★ 中途发现太长 → 改走 hash
          break;
        }
      }
      ...
```

DISTINCT 键的构造——注意 `HA_NULL_ARE_EQUAL`（`NULL == NULL` 判等，这是 SQL 语义要求），且键**跳过隐藏字段**：

```cpp
    if (!unique_constraint_via_hash_field) {
      param->keyinfo->user_defined_key_parts =
          share->fields - param->hidden_field_count;   // ★ 只对可见列建键
      param->keyinfo->flags = HA_NOSAME | HA_NULL_ARE_EQUAL;
      param->keyinfo->name = "<auto_distinct_key>";

      /* Create a distinct key over the columns we are going to return */
      for (unsigned i = param->hidden_field_count; i < share->fields;
           i++, key_part_info++) {
        key_part_info->init_from_field(table->field[i]);
        if (key_part_info->store_length > max_key_part_length) {
          unique_constraint_via_hash_field = true;
          break;
        }
      }
      ...
```

走 hash 时键只有 **1 个 key part**，算法沿用引擎默认（TempTable 是 `HA_KEY_ALG_HASH`）：

```cpp
  // Create a key over hash_field to enforce unique constraint
  if (unique_constraint_via_hash_field) {
    KEY *hash_key;
    KEY_PART_INFO *hash_kpi;
    if (!multi_alloc_root(&share->mem_root, &hash_key, sizeof(*hash_key),
                          &hash_kpi, sizeof(*hash_kpi), NullS))  // Only one key part
      return nullptr;
    table->key_info = share->key_info = hash_key;
    share->key_parts = 1;
    hash_key->actual_flags = hash_key->flags = HA_NULL_ARE_EQUAL;
    hash_key->actual_key_parts = hash_key->usable_key_parts = 1;
    hash_key->user_defined_key_parts = 1;
    hash_key->algorithm = table->file->get_default_index_algorithm();
    if (distinct) hash_key->name = "<hash_distinct_key>";
    else          hash_key->name = "<hash_group_key>";
    hash_kpi->init_from_field(table->hash_field);
    hash_key->key_length = hash_kpi->store_length;
    param->keyinfo = hash_key;
  }
```

**另外两条"隐藏字段 + 键"的特殊设计**：

- **半连接 DuplicateWeedout**：用 `unique_constraint_via_hash_field` 作为 `force_disk_table` 的实参传进去——weedout 表只要 rowid 列，所以直接建成磁盘表更划算：

```cpp
  if (setup_tmp_table_handler(thd, table, 0LL, unique_constraint_via_hash_field,
                              ...
```

- **INTERSECT / EXCEPT**：额外插一个 `<set counter>` 隐藏列计数每行出现次数，`EXCEPT` 时用它做减法（见 `table->set_set_counter(set_counter, m_operation == TTP_EXCEPT)`）。

#### 创建路径

- `create_tmp_table()`：内部临时表（聚合/去重/物化）
- `create_tmp_table_from_fields()`：给定字段列表直接建表（UNION/derived 物化）
- `setup_tmp_table_handler()` + `instantiate_tmp_table()`：真正 `handler->create()` + `open_tmp_table()`

**选引擎的真实位置是 `setup_tmp_table_handler()`**：

```cpp
/* sql/sql_tmp_table.cc */
bool setup_tmp_table_handler(THD *thd, TABLE *table, ulonglong select_options,
                             bool force_disk_table, bool schema_table) {
  TABLE_SHARE *share = table->s;
  assert(table->file == nullptr);

  if (share->db_plugin == nullptr) {
    handlerton *hton;
    enum_internal_tmp_mem_storage_engine mem_engine =
        static_cast<enum_internal_tmp_mem_storage_engine>(
            thd->variables.internal_tmp_mem_storage_engine);

    /* For information_schema tables we use the Heap engine because we do
       not allow user-created TempTable tables and even though information_schema
       tables are not user-created, an ingenious user may execute:
         CREATE TABLE myowntemptabletable LIKE information_schema.some; */
    if (schema_table && (mem_engine == TMP_TABLE_TEMPTABLE)) {
      mem_engine = TMP_TABLE_MEMORY;             // ★ I_S 表强制 MEMORY
    }

    if (use_tmp_disk_storage_engine(thd, table, select_options,
                                    force_disk_table, mem_engine)) {
      hton = innodb_hton;                        // ★ 磁盘侧固定 InnoDB
    } else {
      switch (mem_engine) {
        case TMP_TABLE_TEMPTABLE: hton = temptable_hton; break;
        case TMP_TABLE_MEMORY:    hton = heap_hton;     break;
        default: assert(false);   hton = nullptr;       break;
      }
    }
    share->db_plugin = ha_lock_engine(nullptr, hton);
  }
  ...
  table->file = get_new_handler(share, false, share->alloc_for_tmp_file_handler,
                                share->db_type());
  ...
```

> ⚠️ 注意设的是 `share->db_plugin`（不是 `db_type`），`db_type()` 由 plugin 派生。

`instantiate_tmp_table()` 里 `Created_tmp_tables` 自增，并在此真正 `create`：

```cpp
bool instantiate_tmp_table(THD *thd, TABLE *table) {
  ...
  thd->inc_status_created_tmp_tables();           // ★ Created_tmp_tables++

  if (table->file == nullptr && setup_tmp_table_handler(thd, table, 0)) {
    return true;
  }
  if (share->db_type() == temptable_hton) {
    if (create_tmp_table_with_fallback(thd, table)) return true;
  } else if (share->db_type() == innodb_hton) {
    if (create_tmp_table_with_fallback(thd, table)) return true;
    empty_record(table);                          // Make empty record so random data is not written to disk
  }

  // If a heap table, it's created by open_tmp_table().
  if (open_tmp_table(table)) { ... }
```

**optimizer trace 的 `tmp_table_info` 就是在这里输出的**——排查临时表问题的标准手段（`SET optimizer_trace='enabled=on'`）：

```cpp
/* sql/sql_tmp_table.cc —— trace_tmp_table() */
  trace_tmp.add("columns", s->fields)
      .add("row_length", s->reclength)
      .add("key_length", table->s->keys > 0 ? table->key_info->key_length : 0)
      .add("unique_constraint", table->hash_field ? true : false)
      .add("makes_grouped_rows", table->group != nullptr)
      .add("cannot_insert_duplicates", s->is_distinct);

  if (s->db_type() == innodb_hton) {
    trace_tmp.add_alnum("location", "disk (InnoDB)");
    if (s->db_create_options & HA_OPTION_PACK_RECORD)
      trace_tmp.add_alnum("record_format", "packed");
    else
      trace_tmp.add_alnum("record_format", "fixed");
  } else if (table->s->db_type() == temptable_hton) {
    trace_tmp.add_alnum("location", "TempTable");
  } else {
    trace_tmp.add_alnum("location", "memory (heap)")
        .add("row_limit_estimate", s->max_rows);
  }
```

其中 `unique_constraint: true` 就表示走了 hash_field；`location` 直接告诉你落在哪个引擎。

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

**★ 三个 temptable 系统变量的定义**（默认值一律取自 `sql/sys_vars.cc`，不凭印象）：

```cpp
const char *internal_tmp_mem_storage_engine_names[] = {"MEMORY", "TempTable", nullptr};
static Sys_var_enum Sys_internal_tmp_mem_storage_engine(
    "internal_tmp_mem_storage_engine",
    "The default storage engine for in-memory internal temporary tables.",
    HINT_UPDATEABLE SESSION_VAR(internal_tmp_mem_storage_engine),
    CMD_LINE(REQUIRED_ARG), internal_tmp_mem_storage_engine_names,
    DEFAULT(TMP_TABLE_TEMPTABLE), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(check_session_admin_no_super));

static Sys_var_ulonglong Sys_temptable_max_ram(
    "temptable_max_ram",
    "Maximum amount of memory (in bytes) the TempTable storage engine is "
    "allowed to allocate from the main memory (RAM) before starting to "
    "store data on disk.",
    GLOBAL_VAR(temptable_max_ram), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(2 << 20 /* 2 MiB */, ULLONG_MAX), DEFAULT(1 << 30 /* 1 GiB */),
    BLOCK_SIZE(1));

static Sys_var_ulonglong Sys_temptable_max_mmap(
    "temptable_max_mmap",
    "Maximum amount of memory (in bytes) the TempTable storage engine is "
    "allowed to allocate from MMAP-backed files before starting to "
    "store data on disk.",
    GLOBAL_VAR(temptable_max_mmap), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, ULLONG_MAX), DEFAULT(1 << 30 /* 1 GiB */), BLOCK_SIZE(1));
```

三点值得记：

- `internal_tmp_mem_storage_engine` **默认就是 `TMP_TABLE_TEMPTABLE`**（不是 MEMORY），且只有两个取值
- 两个上限都是 **GLOBAL** 变量（`GLOBAL_VAR`），这正是"全局预算"的由来
- `temptable_use_mmap` 在 8.0.39 **已标记废弃**（`ON_UPDATE(update_deprecated_with_removal_message)`），默认 `true`——意味着 mmap 这一级默认开着

**★ RAM → MMAP 的真正判定点**（`storage/temptable/include/temptable/allocator.h`）——注意**不在 `allocate()` 里**，而在块源策略里，粒度是**整块**：

```cpp
/* Concrete implementation of Block_source_policy, ...
 * In particular, this policy will make TempTable allocator:
 *  1. Use RAM as long as temptable_max_ram threshold is not reached.
 *  2. Start using MMAP when temptable_max_ram threshold is reached.
 *  3. Go back using RAM as soon as RAM consumption drops below the
 *     temptable_max_ram threshold and there is enough space to accommodate the
 *     new block given the size.
 *  4. Not take into account per-table memory limits defined through
 *     tmp_table_size SYSVAR. */
struct Prefer_RAM_over_MMAP_policy {
  static Source block_source(uint32_t block_size,
                             TableResourceMonitor * = nullptr) {
    if (MemoryMonitor::RAM::consumption() < MemoryMonitor::RAM::threshold()) {
      if (MemoryMonitor::RAM::increase(block_size) <=
          MemoryMonitor::RAM::threshold()) {
        return Source::RAM;                       // ★ ① RAM
      } else {
        MemoryMonitor::RAM::decrease(block_size); // 试算失败要还回去
      }
    }
    if (MemoryMonitor::MMAP::consumption() < MemoryMonitor::MMAP::threshold()) {
      if (MemoryMonitor::MMAP::increase(block_size) <=
          MemoryMonitor::MMAP::threshold()) {
        return Source::MMAP_FILE;                 // ★ ② mmap 文件
      } else {
        MemoryMonitor::MMAP::decrease(block_size);
      }
    }
    throw Result::RECORD_FILE_FULL;               // ★ ③ 都放不下 → 上抛
  }
  ...
};
```

注意第 3 条：**RAM 用量的下降会让分配器退回 RAM**——这是"水位回落"设计，不是单调递增。

`MemoryMonitor` 用两个 `std::atomic<size_t>` 全局计数，阈值直接读变量：

```cpp
  struct RAM {
    static size_t increase(size_t bytes) { return ram.fetch_add(bytes) + bytes; }
    static size_t decrease(size_t bytes) { return ram.fetch_sub(bytes) - bytes; }
    static size_t threshold() { return temptable_max_ram; }
    static size_t consumption() { return ram; }
  };
  struct MMAP {
    ...
    static size_t threshold() {
      if (temptable_use_mmap) {
        return temptable_max_mmap;
      } else {
        return 0;                                 // ★ 关掉 mmap 就等于这一级不存在
      }
    }
  };
 private:
  /** Total bytes allocated so far by all threads in RAM/MMAP. */
  static std::atomic<size_t> ram;
  static std::atomic<size_t> mmap;
```

**每表限制是包在外层的另一个策略**——先判 `tmp_table_size`，再委托给全局策略：

```cpp
struct Prefer_RAM_over_MMAP_policy_obeying_per_table_limit {
  static Source block_source(uint32_t block_size,
                             TableResourceMonitor *table_resource_monitor) {
    assert(table_resource_monitor);
    if (table_resource_monitor->consumption() + block_size >
        table_resource_monitor->threshold())
      throw Result::RECORD_FILE_FULL;             // ★ 单表超 tmp_table_size
    return Prefer_RAM_over_MMAP_policy::block_source(block_size);
  }
  ...
};
```

默认组合就是这两个策略：`using Exponential_growth_preferring_RAM_over_MMAP = Allocation_scheme<Exponential_policy, Prefer_RAM_over_MMAP_policy_obeying_per_table_limit>;`

**块大小按已分配块数指数增长，上限 512 MiB**：

```cpp
/* ... this policy will make TempTable allocator to grow the
 * block-size at exponential rate with upper limit of ALLOCATOR_MAX_BLOCK_BYTES,
 * E.g. allocation pattern may look like the following:
 *  1 MiB, 2 MiB, 4 MiB, 8 MiB, 16 MiB, 32 MiB, ..., ALLOCATOR_MAX_BLOCK_BYTES */
struct Exponential_policy {
  static size_t block_size(size_t number_of_blocks, size_t n_bytes_requested) {
    size_t block_size_hint;
    if (number_of_blocks < ALLOCATOR_MAX_BLOCK_MB_EXP) {
      block_size_hint = (1ULL << number_of_blocks) * 1_MiB;
    } else {
      block_size_hint = ALLOCATOR_MAX_BLOCK_BYTES;
    }
    return std::max(block_size_hint, Block::size_hint(n_bytes_requested));
  }
};
```

```cpp
/* storage/temptable/include/temptable/constants.h */
constexpr size_t ALLOCATOR_MAX_BLOCK_MB_EXP = 9;                    // log2(上限 MiB)
constexpr size_t ALLOCATOR_MAX_BLOCK_BYTES = 1_MiB << ALLOCATOR_MAX_BLOCK_MB_EXP;  // = 512 MiB
constexpr size_t STORAGE_PAGE_SIZE = 64_KiB;
constexpr size_t INDEX_DEFAULT_HASH_TABLE_BUCKETS = 1024;
```

块序列：1, 2, 4, 8, 16, 32, 64, 128, 256 MiB，第 9 块起封顶 512 MiB。**为什么是指数增长**——把"向 OS 要内存的次数"从 O(字节数) 压到 O(log 字节数)，同时提升空间局部性；块分配器只维护"已用 chunk 数 + 下一个可用 slot 偏移"，**不维护 chunk 链表**，因此只支持回收最右侧 chunk。

**RAM 与 mmap 两个来源是两个模板特化**（`Source::RAM` / `Source::MMAP_FILE`）：

```cpp
/* storage/temptable/include/temptable/memutils.h */
enum class Source {
  /** Memory is allocated on disk, using mmap()'ed file. */
  MMAP_FILE,
  /** Memory is allocated from RAM, using malloc() for example. */
  RAM,
};
template <> struct Memory<Source::RAM> { ... };         // NUMA 可用走 numa_alloc_local()，否则 malloc()
template <> struct Memory<Source::MMAP_FILE> { ... };   // 失败抛 Result::RECORD_FILE_FULL
```

mmap 文件落在 `mysql_tmpdir`，前缀 `mysql_temptable.`，`UNLINK_FILE` 立即 unlink（所以 `ls` 看不到），并先 `my_fallocator` 写满 `0xa` 字节真正占位：

```cpp
inline void *Memory<Source::MMAP_FILE>::fetch(size_t bytes) {
  char file_path[FN_REFLEN];
  File f = create_temp_file(file_path, mysql_tmpdir, "mysql_temptable.", mode,
                            UNLINK_FILE, MYF(MY_WME));
  if (f < 0) return nullptr;

  /* This will write `bytes` 0xa bytes to the file on disk. */
  if (my_fallocator(f, bytes, 0xa, MYF(MY_WME)) != 0 || ...) { my_close(f, MYF(MY_WME)); return nullptr; }

  void *ptr = my_mmap(nullptr, bytes, PROT_READ | PROT_WRITE, MAP_SHARED, f, 0);
  // NOTE: Closing the file-descriptor __immediately__ after MMAP-ing it does
  // not impact the MMAP functionality at all. ...
  my_close(f, MYF(MY_WME));
  return (ptr == MAP_FAILED) ? nullptr : ptr;
}
```

**server 侧唯一的"取 min"点**在 `alloc_record_buffers()`——**只有 MEMORY 引擎才取 min**：

```cpp
  if (thd->variables.tmp_table_size == ~(ulonglong)0)  // No limit
    share->max_rows = ~(ha_rows)0;
  else
    share->max_rows = (ha_rows)(((share->db_type() == heap_hton)
                                     ? min(thd->variables.tmp_table_size,
                                           thd->variables.max_heap_table_size)
                                     : thd->variables.tmp_table_size) /
                                share->reclength);
```

> 这行代码就是「`max_heap_table_size` 对 TempTable 完全不生效」的最终证据：三元表达式的非 MEMORY 分支根本不看它。

**TempTable 的内存走 PFS 记账**（`memory/temptable/physical_ram` 与 `physical_disk` 两个 key）；而"每次 chunk 分配"的 `logical` 计数默认关闭，源码注释写明开启会有约 4% 性能损失：

```cpp
/* storage/temptable/src/block.cc */
/* Enabling this causes ~ 4% performance drop in sysbench distinct ranges. */
//#define TEMPTABLE_PFS_MEMORY_COUNT_LOGICAL
...
PSI_memory_info pfs_info[] = {
    {&mem_key_physical_disk, "physical_disk", PSI_FLAG_MEM_COLLECT, 0, PSI_DOCUMENT_ME},
    {&mem_key_physical_ram, "physical_ram", PSI_FLAG_MEM_COLLECT, 0, PSI_DOCUMENT_ME},
};
```

### 降级到磁盘：两条路径

#### 路径 ①：建表时就决定用磁盘

`create_tmp_table_with_fallback()`——**建表那一刻**若 TempTable 报 `HA_ERR_RECORD_FILE_FULL`（说明 `temptable_max_ram` 已被别的会话吃光），直接换 `innodb_hton` 重建：

```cpp
/* sql/sql_tmp_table.cc —— create_tmp_table_with_fallback() */
  /*
    INNODB's fixed length column size is restricted to 1024. Exceeding this can
    result in incorrect behavior.
  */
  if (table->s->db_type() == innodb_hton) {
    for (Field **field = table->field; *field; ++field) {
      if ((*field)->type() == MYSQL_TYPE_STRING &&
          (*field)->key_length() > 1024) {
        my_error(ER_TOO_LONG_KEY, MYF(0), 1024);
        return true;
      }
    }
  }

  int error =
      table->file->create(share->table_name.str, table, &create_info, nullptr);
  if (error == HA_ERR_RECORD_FILE_FULL &&
      table->s->db_type() == temptable_hton) {        // ★ 只有 TempTable 有这个 fallback
    table->file = get_new_handler(
        table->s, false, share->alloc_for_tmp_file_handler, innodb_hton);
    error = table->file->create(share->table_name.str, table, &create_info,
                                nullptr);
  }

  if (error) {
    table->file->print_error(error, MYF(0));
    table->db_stat = 0;
    return true;
  } else {
    if (table->s->db_type() != temptable_hton) {
      thd->inc_status_created_tmp_disk_tables();      // ★ Created_tmp_disk_tables++
    }
    return false;
  }
```

三个要点：

- 判据是**错误码 + 引擎是 temptable**，MEMORY 引擎走不到这里（它的 `create` 不会因内存不足失败）
- **换引擎时旧 handler 没被销毁**——直接覆盖 `table->file`。这是"降级"而非"重建"
- `Created_tmp_disk_tables` 只在最终引擎不是 temptable 时自增，所以 TempTable→mmap **不计入**这个状态变量（mmap 仍算"内存表"）

**"建表时就决定用磁盘"的判定是 `use_tmp_disk_storage_engine()`**（前面已贴过主体），四个条件：

| 条件 | 含义 |
|---|---|
| `force_disk_table` | 调用方明确要求（半连接 DuplicateWeedout 传的就是 `unique_constraint_via_hash_field`） |
| `opt_initialize \|\| thd->is_dd_system_thread()` | bootstrap / DD 升级线程，heap 引擎未加载 |
| MEMORY 引擎 + `share->blob_fields` | MEMORY 不支持 BLOB |
| `big_tables` 开启且不是 `SELECT_SMALL_RESULT` | 用户声明"结果会很大" |

`big_tables` 在 `sql/` 下**只有这一处生效点**，是纯粹的"跳过内存直接落盘"开关。

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
> ⚠️ **纠错**：`enum_op_type` **共 8 个值**，且挂在 `QEP_TAB` 上（不是 `Temp_table_param`）。与临时表相关的其实是 **4 个**，另外两个 `OT_AGGREGATE` / `OT_NONE` 恰恰是**不建临时表**的流式路径。

```cpp
/* sql/sql_executor.h —— QEP_TAB */
  // Operation between the previous QEP_TAB and this one.
  enum enum_op_type {
    OT_NONE,                            // Regular nested loop.
    OT_AGGREGATE,                       // Aggregate (GROUP BY).  ★ 不建 tmp 表
    OT_MATERIALIZE,                     // Various temporary table operations
    OT_AGGREGATE_THEN_MATERIALIZE,
    OT_AGGREGATE_INTO_TMP_TABLE,
    OT_WINDOWING_FUNCTION,
    OT_BNL,                             // Block-nested loop (rewritten to hash join)
    OT_BKA                              // Batch key access
  } op_type = OT_NONE;
```

设置点只有一处——`setup_tmptable_write_func()`，三个分支的判据写得很清楚：

```cpp
/* sql/sql_executor.cc */
  if (table->group && tmp_tbl->sum_func_count &&
      !tmp_tbl->precomputed_group_by) {
    /* Note for MyISAM tmp tables: if uniques is true keys won't be created. */
    if (table->s->keys) {
      description = "continuously_update_group_row";
      tab->op_type = QEP_TAB::OT_AGGREGATE_INTO_TMP_TABLE;   // ① 有聚合 + 有唯一键 → 边扫边更新分组行
    }
  } else if (join->streaming_aggregation && !tmp_tbl->precomputed_group_by) {
    description = "write_group_row_when_complete";
    tab->op_type = QEP_TAB::OT_AGGREGATE_THEN_MATERIALIZE;   // ② 流式聚合完再整体物化
    ...
  } else {
    description = "write_all_rows";
    tab->op_type = (phase >= REF_SLICE_WIN_1 ? QEP_TAB::OT_WINDOWING_FUNCTION
                                             : QEP_TAB::OT_MATERIALIZE);  // ③ 窗口 / ④ 普通物化
    ...
  }
  if (description) trace->add_alnum("write_method", description);
```

`description` 会写进 optimizer trace 的 `write_method` 字段——**直接就能看出这次临时表是"边扫边更新"还是"整体物化"**。`OT_AGGREGATE` / `OT_NONE` 在 `JOIN::get_end_select_func()` 里设置，走 `end_send_group` / `end_send`，根本不建临时表。

| op_type | 语义 | 迭代器 |
|---|---|---|
| `OT_MATERIALIZE` | 写入临时表，不聚合，再扫描 | `MaterializeIterator` |
| `OT_AGGREGATE_THEN_MATERIALIZE` | 先流式聚合，再物化结果 | `AggregateIterator` + 物化 |
| `OT_AGGREGATE_INTO_TMP_TABLE` | 直接在临时表上分组聚合 | `TemptableAggregateIterator` |
| `OT_WINDOWING_FUNCTION` | 窗口函数分区暂存 | `MaterializeIterator`（窗口变体） |

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

**写入失败的降级点就在 `MaterializeQueryBlock()` 里**——这是"路径 ②"在普通物化路径上的触发处：

```cpp
/* sql/iterators/composite_iterators.cc */
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
      ...
      // Inform each reader that the table has changed under their feet,
      // so they'll need to reposition themselves.
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

两处分流值得记：

- `is_ignorable_error(error)`（即 `HA_ERR_FOUND_DUPP_KEY`）**不算失败**——那正是"去重生效了"
- 降级后必须**通知递归 CTE 的读取端重新定位游标**（`RepositionCursorAfterSpillToDisk`），因为它正一边读一边写

#### 共享物化（CTE / derived 被多次引用）

同一张物化表被多个地方引用时，不是各建一张，而是用 `clone_tmp_table` 造指向同一份数据的"克隆"引用——所以 CTE 只物化一次。降级时也正因如此，才需要把**所有克隆**一起转到新引擎（见上面代码）。

**"只物化一次"的判定条件**（`MaterializeIterator::Init()`）——源码里真正的标识符是 `use_shared_cte_materialization`，**没有 `m_shared` 这个成员**：

```cpp
  // If this is a CTE, it could be referred to multiple times in the same query.
  // If so, check if we have already been materialized through any of our alias
  // tables.
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
    table()->materialized = true;                 // ★ 直接复用，不再执行子查询
  }
```

四个条件缺一不可：① 本 TABLE 尚未物化 ② 是 CTE（`m_cte != nullptr`，derived table 无此字段）③ 非相关/非 LATERAL（`!m_rematerialize`）④ 该 CTE 的任一引用已经物化。

**克隆共享同一个 `TABLE_SHARE`**，只复制 `hash_field` 指针（因为克隆的 `field[0]` 就是 hash_field）：

```cpp
/* sql/sql_derived.cc —— Common_table_expr::clone_tmp_table() */
  assert(t->s == first->s && t != first && t->file != first->file);
  t->s->increment_ref_count();
  t->s->tmp_handler_count++;

  // In case this clone is used to fill the materialized table:
  bitmap_set_all(t->write_set);
  t->reginfo.lock_type = TL_WRITE;
  t->copy_blobs = true;
  ...
  // If initial CTE table has a hash key, set up a hash key for all clones too.
  if (first->hash_field) {
    t->hash_field = t->field[0];
  }
  t->hidden_field_count = first->hidden_field_count;
  t->set_not_started();
```

> ★ **共享 TABLE_SHARE 但各自有独立 handler**（`t->file != first->file`）——这正是降级时"必须把所有克隆的 handler 一起换掉"的根源：它们指向同一份逻辑数据，引擎必须一致。

递归 CTE 特殊在：一边写入一边读取（用不同的 TABLE 引用区分读写），所以降级后要 `RepositionCursorAfterSpillToDisk()`。

**`TemptableAggregateIterator` 侧的降级入口是 `move_table_to_disk()`**，语义与物化路径一致：

```cpp
template <typename Profiler>
bool TemptableAggregateIterator<Profiler>::move_table_to_disk(int error_code,
                                                              bool was_insert) {
  if (create_ondisk_from_heap(thd(), table(), error_code, was_insert,
                              /*ignore_last_dup=*/false,
                              /*is_duplicate=*/nullptr)) {
    return true;
  }
  int error = table()->file->ha_index_init(0, false);
  if (error != 0) {
    PrintError(error);
    return true;
  }
  return false;
}
```

降级后要**重新定位待更新的那一行**再重试——这是 `TemptableAggregateIterator::Init()` 里最绕的一段（换引擎后 record[0] 里的 BLOB 指针会失效，必须 `restore_record` + 重算聚合）：

```cpp
      if (move_table_to_disk(error, /*insert_operation=*/false)) {
        end_unique_index.commit();
        return true;
      }
      const uchar *key;
      if (using_hash_key()) {
        key = table()->hash_field->field_ptr();
      } else {
        key = m_temp_table_param->group_buff;
      }
      // Read the record to be updated.
      if (table()->file->ha_index_read_map(table()->record[1], key,
                                           HA_WHOLE_KEY, HA_READ_KEY_EXACT)) {
        return true;
      }
      /* As the table has moved to disk, the references to the blobs
         in record[0], if any, would be stale. Copy the record and
         re-evaluate the functions. */
      restore_record(table(), record[1]);
      update_tmptable_sum_func(m_join->sum_funcs, table());
      // Retry the update on the newly created on-disk table.
      error = table()->file->ha_update_row(table()->record[1], table()->record[0]);
```

**其中一个容易忽略的细节**：分组键用 hash 时，`copy_funcs()` 必须在**查重之前**调用一次，否则 hash_field 还是旧值；而用普通键时则要推迟到确认是新分组后才调用，避免同组第 2..N 行重复求值：

```cpp
    if (using_hash_key()) {
      /* We need to call copy_funcs here in order to get correct value for
         hash_field. However, this call isn't needed so early when
         hash_field isn't used as it would cause unnecessary additional
         evaluation of functions to be copied when 2nd and further records
         in group are found. */
      if (copy_funcs(m_temp_table_param, thd())) return true;
      group_found = !check_unique_constraint(table());
    } else {
      for (ORDER *group = table()->group; group; group = group->next) {
        Item *item = *group->item;
        item->save_org_in_field(group->field_in_tmp_table);
        /* Store in the used key if the field was 0 */
        if (item->is_nullable())
          group->buff[-1] = (char)group->field_in_tmp_table->is_null();
      }
      const uchar *key = m_temp_table_param->group_buff;
      group_found = !table()->file->ha_index_read_map(
          table()->record[1], key, HA_WHOLE_KEY, HA_READ_KEY_EXACT);
    }
```

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

**默认值的源码出处**（都读自 `sql/sys_vars.cc`，不凭印象）：

```cpp
static Sys_var_ulonglong Sys_max_heap_table_size(
    "max_heap_table_size",
    "Don't allow creation of heap tables bigger than this",
    HINT_UPDATEABLE SESSION_VAR(max_heap_table_size), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(16384, (ulonglong) ~(intptr)0), DEFAULT(16 * 1024 * 1024),
    BLOCK_SIZE(1024));

static Sys_var_ulonglong Sys_tmp_table_size(
    "tmp_table_size",
    "If an internal in-memory temporary table in the MEMORY or TempTable "
    "storage engine exceeds this size, MySQL will automatically convert it "
    "to an on-disk table ",
    HINT_UPDATEABLE SESSION_VAR(tmp_table_size), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1024, std::numeric_limits<ulonglong>::max()),
    DEFAULT(16 * 1024 * 1024), BLOCK_SIZE(1));

static Sys_var_bool Sys_big_tables(
    "big_tables",
    "Allow big result sets by saving all "
    "temporary sets on file (Solves most 'table full' errors)",
    HINT_UPDATEABLE SESSION_VAR(big_tables), CMD_LINE(OPT_ARG), DEFAULT(false));
```

> ★ 两个"临时表大小"变量的 BLOCK_SIZE 不同：`tmp_table_size` 是 1（可精确到字节），`max_heap_table_size` 是 1024（按 1KB 取整）——这解释了为什么设置后者时偶尔会看到值被向下取整。

`tmpdir` 是只读的全局变量，决定了 mmap 文件与磁盘临时表的落点：

```cpp
static Sys_var_charptr Sys_tmpdir(
    "tmpdir",
    "Path for temporary files. Several paths may "
    "be specified, separated by a "
#if defined(_WIN32)
    "semicolon (;)"
#else
    "colon (:)"
#endif
    ", in this case they are used in a round-robin fashion",
    READ_ONLY NON_PERSIST GLOBAL_VAR(opt_mysql_tmpdir),
    CMD_LINE(REQUIRED_ARG, 't'), IN_FS_CHARSET, DEFAULT(nullptr));
```

**性能_schema 侧可以观测 TempTable 的真实内存占用**——按物理来源分开记账：

```sql
SELECT * FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'memory/temptable/%';
-- memory/temptable/physical_ram    ← RAM 用量
-- memory/temptable/physical_disk   ← mmap 用量（注意：这是"走了 mmap"的信号）
```

> ★ 排查手法：`physical_disk` 从 0 变正，就说明 `temptable_max_ram` 被吃穿了；此时应查是否有大查询（`SELECT ... GROUP BY` 无索引、大 `UNION DISTINCT`）。

### 状态变量

| 变量名 | 含义 |
|---|---|
| `Created_tmp_tables` | 创建的内部临时表总数 |
| `Created_tmp_disk_tables` | **落到磁盘的**临时表数（性能排查首选） |
| `Created_tmp_files` | 创建的临时文件数 |

排查思路：`Created_tmp_disk_tables` 占比高 ⇒ 调大 `temptable_max_ram` / `tmp_table_size`，或改写查询（加索引避免临时表、减少 DISTINCT/GROUP BY 的列宽）。optimizer trace 里的 `converting_tmp_table_to_ondisk` 能定位是哪一步触发的。

---

## Misc

### ★★★ 真名勘误

以下都是"常见说法 / 旧资料说法"与 8.0.39 实际的差异，逐条经 grep 核实：

| 常见说法 | 8.0.39 实际 |
|---|---|
| `store_record_in_tmp_table()` | **不存在**（`sql/` 下 grep 0 命中）。写入是 `MaterializeIterator::MaterializeQueryBlock()` 里直接 `t->file->ha_write_row()` |
| `make_unique_key_by_hash()` | **不存在**。哈希由 `unique_hash()` / `unique_hash_group()` / `unique_hash_fields()` 生成 |
| `Temp_table_param::using_unique_constraint` / `uniques` | **不存在**这两个成员。实际是 `create_tmp_table()` 里的**局部 bool** `unique_constraint_via_hash_field` |
| `Temp_table_param::enum_op_type`，三种 | `op_type` 挂在 **`QEP_TAB`**，且**共 8 个值**；与临时表相关的是 4 个 |
| `precomputed_grouping` | 真名 `precomputed_group_by` |
| `MaterializeIterator::m_shared` | **不存在**。共享物化的标识是局部 `const bool use_shared_cte_materialization` + 成员 `m_cte` |
| `internal_tmp_disk_storage_engine` 变量 | 8.0 已**删除**，磁盘侧固定 InnoDB；仅错误消息文本 `ER_SWITCH_TMP_ENGINE_MSG` 里残留该名字 |
| MEMORY 的 key 结构叫 `HEAP_KEY` | **不存在**。是 `HP_KEYDEF` + `HA_KEYSEG` |
| `ha_temptable::write_row` / `index_read_map` | 类名是 **`temptable::Handler`**；没有 `index_read_map` 覆写（`storage/temptable` 下 grep 0 命中），只有 `Handler::index_read` |
| `Row::data()` | **不存在**。取单元格是 `Row::cell(const Column &, size_t)` |
| `Cell` 在 `column.h` | Cell 在 **`include/temptable/cell.h`**；`column.h` 是 `Column`（MySQL 行格式解释器），两者完全不同 |
| TempTable 索引用 `std::multimap` | 是 **`std::multiset`** / `std::unordered_set` / `std::unordered_multiset`——set 不是 map，key 与 value 是同一对象 |
| `Allocator<RAM>` / `Allocator<MMAP>` | 真身是 `Allocator<T, AllocationScheme>`；RAM/MMAP 是 `Memory<Source::RAM>` / `Memory<Source::MMAP_FILE>` 两个**模板特化** |
| Threshold 判定在 `Allocator::allocate()` | 在 **`Prefer_RAM_over_MMAP_policy::block_source()`**，粒度是整块 |
| `temptable_use_mmap` 是正常开关 | 8.0.39 **已 deprecated**（`ON_UPDATE(update_deprecated_with_removal_message)`） |
| TempTable 头文件在 `src/` | 头文件全在 `storage/temptable/include/temptable/`；`src/` 下只有 `.cc` |
| "定长行零拷贝" | 定长行是**一次整行 memcpy**（无 Cell 数组、无逐列开销）；"零拷贝"更准确地说是指 `Row(mysql_row, allocator)` 轻量态只存指针 |
| MEMORY 不支持变长列 | 不准确：MEMORY **支持 VARCHAR**，不支持的是 **BLOB/TEXT**；只是行存储是定长 `reclength` |
| hash_field 冲突会丢行 | **不会**。`check_unique_constraint()` 冲突时逐列比对（`table_rec_cmp`），hash 只用于定位候选行 |
| `max_heap_table_size` 对 TempTable 生效 | 不生效。`alloc_record_buffers()` 的三元表达式只有 `heap_hton` 分支才取 min |

### 容易踩的坑

- **TempTable 的 `temptable_max_ram` 是全局的**：一个会话吃满，其它会话的临时表立刻转 mmap/磁盘。生产上不要盲目调得太大（那是实打实的常驻内存），也不要用默认值硬扛高并发大查询
- **`Created_tmp_disk_tables` 不包含 mmap**：TempTable→mmap 仍算"内存表"。所以这个状态变量没涨不代表没降级
- **降级是"整表转换"**：超一点点内存就吃一次全表拷贝，性能曲线是断崖不是斜坡
- **临时表上的 BLOB 指针在降级后会失效**：`TemptableAggregateIterator` 里必须 `restore_record()` + 重算聚合才能重试（见上文代码）
- **窗口函数的 frame buffer 也是临时表**：`Temp_table_param::m_window_frame_buffer` 为真时走 `copy_fields` 而非 `items_to_copy`，因为帧缓冲需要"反向拷贝"能力

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

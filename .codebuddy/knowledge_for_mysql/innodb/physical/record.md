# InnoDB 行记录格式与 offsets 数组深度解析

> 基于 MySQL 8.0.39 源码，涵盖行的物理布局（record header / 变长长度 / NULL 位图 / info bits）、**offsets 数组**的设计与编码（`rec_get_offsets` 全链：入口 → 解析 → 字段定位）、REDUNDANT 的 1/2 字节偏移目录、instant ADD/DROP 与行版本在 offsets 层的投影、写方向的 `rec_convert_dtuple_to_rec`、以及 offsets 的消费方。
>
> **边界**：本篇讲**记录本身**（页里的"内容物"）。页的组织（FIL header、page directory、页内查找）见 [`page_structure.md`](page_structure.md)；instant DDL 的语义与决策见 [`ddl.md`](../../feat/ddl/ddl.md)（本篇只讲它在行格式/offsets 层的落地）；二级索引回表取行见 [`row_search.md`](../row_search.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
  - [算法与数据结构](#算法与数据结构)（offsets 布局 / `rec_offs_*` API / `dtuple_t`·`dfield_t`·`dtype_t`）
- [核心实现](#核心实现)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

InnoDB 的"行"（`rec_t`）是 B+ 树叶节点里的一段**自描述字节流**：一个 5（或 6）字节的记录头 + 变长字段长度数组 + NULL 位图 + 字段数据。而 **offsets 数组**是解析这段字节流的中间产物——一个 `ulint` 数组，记录"每个字段在记录里的起始/结束偏移与状态标志"。引擎里几乎每一次取字段、比较记录、写行，都先 `rec_get_offsets()` 再按 offsets 定位。

### 用途

解决的核心问题是：**物理行为了紧凑牺牲了可寻址性**。COMPACT 格式的行里只有"变长字段的长度"和 NULL 位，没有"每个字段的偏移目录"（那是 REDUNDANT 格式的做法，代价是每字段 1~2 字节）。要拿第 n 个字段，必须从头把前 n 个字段的长度全解出来。offsets 数组把这一次解析的结果缓存下来，之后 O(1) 定位任意字段——**一次解析，处处使用**。

它被全引擎复用：`cmp_dtuple_rec`（比较）、`row_build`（record → dtuple）、`row_search_mvcc`（扫描取行）、`lock_rec_*`（锁）、purge、undo 回放、页分裂——全部基于同一份 offsets 协议。

### 版本演进

| 版本 | 变化 |
|---|---|
| 远古（4.x） | REDUNDANT 行格式：6 字节头 + **每字段 1~2 字节的偏移目录**（行自带 offsets，代价是空间） |
| 5.0.3 | COMPACT 行格式：5 字节头，偏移目录被砍掉，换成**变长字段长度数组 + NULL 位图**——字段偏移必须现算（offsets 数组成为运行时必需品） |
| 5.6.9 | 外置（off-page）标志从记录头信息位移入**变长字段长度字节的编码**（2 字节编码的 0x40 位），DYNAMIC 格式伴随 LOB 体系演进 |
| 8.0.29 | instant ADD/DROP 的 V2 行版本号（`REC_INFO_VERSION_FLAG` + 记录头前 1 字节），offsets 解析引入 `REC_OFFS_DEFAULT`/`REC_OFFS_DROP` 标志与"逻辑字段 ≠ 物理字段"的双坐标体系 |

---

## 理论基础

### 设计思想与权衡

**1. 为什么 offsets 不是"行的一部分"，而是运行时解析产物。**

REDUNDANT 把偏移目录存在行里（每字段 1~2 字节，NULL 编码进偏移高位），行天然可随机寻址；COMPACT 把这笔空间省掉，换取更小的行。这个取舍在 COMPACT 时代是对的——大多数行短、字段少，目录开销占比大。代价就是：取字段必须线性解析前缀，而**一次解析的产物必须被复用**，否则每次比较都 O(字段数) 从头解。offsets 数组就是"解析结果复用"的载体。

本来可以那样做、但没有：给 offsets 建真正的对象（带长度、版本、校验和），像 `dtuple_t` 一样管理生命周期。实际做法是**裸 `ulint` 数组 + 约定编码**——因为 offsets 出现在最热的路径上（每行每次扫描都算），对象化意味着堆分配；裸数组允许**栈上预分配**（`ulint offsets_[REC_OFFS_NORMAL_SIZE]`，100 个元素覆盖绝大多数表），热路径零 malloc。代价是编码全靠宏约定，读代码的人必须先背熟布局。

**2. 前缀和设计：offsets 存"结束偏移"而不是"长度"。**

`rec_offs_base(offsets)[1+i]` 存的是**字段 i 的结束偏移**（前缀和），字段长度 = `base[1+n] & MASK - (n==0 ? 0 : base[n] & MASK)`。为什么存前缀和：解析时是顺序累加的（`offs += len`），写回数组的自然形态就是前缀和；而取字段时 `rec_get_nth_field_offs_low` 需要的恰是"起始偏移 = 前一个字段的结束偏移"——前缀和让两个方向的访问都 O(1)，换成存长度则任一方向都要循环累加。

**3. 高 4 位标志编码：把"状态"塞进"偏移"的同一格。**

一个 ulint 同时承载字段结束偏移（低 28 位）与四个状态标志：

```cpp
/* Compact flag ORed to the extra size returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_COMPACT = 1U << 31;   // 0x80000000 —— 只用于 base[0]
/* SQL NULL flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_SQL_NULL = 1U << 31;  // 0x80000000 —— 只用于 base[1+n]
/* External flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_EXTERNAL = 1 << 30;   // 0x40000000 —— 两者都用
/* Default value flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_DEFAULT = 1 << 29;    // 0x20000000 —— instant ADD 默认值
/* An INSTANT DROP flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_DROP = 1 << 28;       // 0x10000000 —— instant DROP
/* Mask for offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_MASK = REC_OFFS_DROP - 1;  // 0x0FFFFFFF —— 低 28 位
```

> ★★ **`REC_OFFS_COMPACT` 与 `REC_OFFS_SQL_NULL` 是同一个位**（都是 `1U << 31`）！不冲突的唯一原因是**一个只用于 `base[0]`、另一个只用于 `base[1+n]`**。这是 offsets 编码里最容易看漏的一处"位复用"。

NULL 字段不占物理空间，但 offsets 里仍占一格（记 `offs | REC_OFFS_SQL_NULL`，偏移与前一个字段相同）——**逻辑字段序与物理字节序由此解耦**。28 位偏移上限 256MB，远超最大页 64KB，安全余量充足。DEFAULT/DROP 两个标志是 8.0.29 instant 的产物：行里没有这个字段的数据，但 offsets 仍按 index 的**全字段数**生成，缺失字段用标志补位——见第 4 点。

**4. 逻辑字段 vs 物理字段：instant 之后的行是"半描述"的。**

instant ADD COLUMN 后，同一张表里老行缺新列。offsets 的约定是：**始终按 index 定义的字段数生成**，行里实际没有的字段不读数据，直接标记：

```cpp
if (col->is_dropped_in_or_before(row_version)) {
  len = offs | REC_OFFS_DROP;             // 该版本已 DROP，行里无数据
} else if (col->is_added_after(row_version)) {
  len = rec_get_instant_offset(index, i, offs);  // 行里没有 → NULL 或 DEFAULT 标志
}
```

`rec_get_instant_offset` 查 `index->get_nth_default(n)`：默认值是 NULL 则标 `REC_OFFS_SQL_NULL`，否则标 `REC_OFFS_DEFAULT`（读方去 DD 的 instant default 里取值）。这带来一个贯穿 8.0.29+ 代码的概念：**逻辑字段号（index 的字段序）与物理字段号（行里的字段序）不再相同**，所有 `rec_offs_nth_*` 入口都先做 `index->get_field_off_pos(n)` 的坐标换算。这是 offsets 体系为 instant 付出的最大代价——每个消费方都要意识到双坐标。

**5. 与 `dtuple_t` 的分工：逻辑行 vs 物理行。**

| | `dtuple_t` | `rec_t` | offsets |
|---|---|---|---|
| 角色 | 逻辑行（内存中可变） | 物理行（页内字节流） | 两者之间的桥 |
| 字段寻址 | 数组直接索引 | 必须解析 | 一次解析缓存 |
| 出现场景 | 搜索元组、插入行、回放行 | 页内记录、undo 记录 | 一切需要"字段"的地方 |

`rec_convert_dtuple_to_rec`（逻辑 → 物理）与 `row_build`/`rec_copy_prefix_to_dtuple`（物理 → 逻辑）是两条转换主链，offsets 是两条链上通用的坐标系统。

### 理论溯源

- **变长记录的紧凑编码**：COMPACT 的"变长长度数组 + NULL 位图 + 定长字段零开销"是关系系统存储层处理变长行的经典方案（System R 时代至今），与 PostgreSQL 的行布局（定长头 + NULL 位图 + 数据）同构，差异在 InnoDB 把变长长度放在数据之前且**逆序**存放。
- **前缀和数组**：offsets 本质是"字段结束偏移的前缀和序列"，配合"起始偏移 = 前一项"的约定，把一个序列同时用作两个方向的索引。

### 算法与数据结构

#### offsets 数组的完整布局

```
         offsets[0]  offsets[1]  offsets[2] offsets[3] │ base[0]      base[1]   base[2]   ... base[n]
         ──────────  ──────────  ────────────────────  │ ───────────  ────────  ────────      ────────
内容：    n_alloc     n_fields    rec指针    index指针   │ extra_size│  字段0末   字段1末   ...  字段n-1末
                                  ├─ 仅 DEBUG 构建 ─┤   │ |COMPACT    偏移      偏移           偏移
                                                       │ |EXTERNAL   (前缀和)  (前缀和)       (前缀和)
         └──────────── REC_OFFS_HEADER_SIZE ──────────┘ │
             release = 2，debug = 4                     └── rec_offs_base(offsets) ──────────────────
```

逐项说明：

| 槽位 | 内容 | 说明 |
|---|---|---|
| `offsets[0]` | `n_alloc` | 数组分配的元素数（`rec_offs_set_n_alloc`）。判断栈上 100 个够不够，不够去 heap 重分配 |
| `offsets[1]` | `n_fields` | 字段数（`rec_offs_set_n_fields`）。注意是**逻辑字段数**，instant 后可能大于行里实际有的 |
| `offsets[2]` | rec 指针 | **仅 DEBUG**：`rec_offs_make_valid` 写入，供 `rec_offs_validate` 校验 offsets 与 rec 匹配 |
| `offsets[3]` | index 指针 | **仅 DEBUG**：同上，且 `offsets[2]==nullptr` 表示"缓存态"（见「固定 offsets 缓存」） |
| `base[0]` | `extra_size \| COMPACT \| EXTERNAL` | 数据区起点到记录头的距离 = 头 5B + NULL 位图 + 变长长度数组；高两位是格式标志 |
| `base[1..n]` | 字段结束偏移前缀和 | 每格低 28 位是偏移，高 4 位可能是 `SQL_NULL`/`EXTERNAL`/`DEFAULT`/`DROP` |

`REC_OFFS_HEADER_SIZE` 的 2 vs 4 是**编译期常量**（`rem/rec.h`）：

```cpp
#ifdef UNIV_DEBUG
constexpr uint32_t REC_OFFS_HEADER_SIZE = 4;   // 多两个指针用于校验
#else
constexpr uint32_t REC_OFFS_HEADER_SIZE = 2;
#endif
```

**一个真实数值例子**：表 `t(a INT NOT NULL, b VARCHAR(10), c INT)`，插入 `(1, 'ab', NULL)`，COMPACT 格式。行内容（从高地址到低地址）：数据区 `a=01 00 00 00` + `b='ab'`，前面是 NULL 位图 1B（c 的位为 1）+ 变长长度 1B（b 长度 2），再前是 5B 记录头。解析出的 offsets：

```
offsets[0] = 103          // n_alloc（栈上 100 不够？这里够，n_fields+1+HEADER=3+1+2=6 ≤ 100）
offsets[1] = 3            // n_fields
base[0]    = 7 | REC_OFFS_COMPACT    // extra_size = 5(头) + 1(NULL位图) + 1(变长长度)
base[1]    = 4                        // a 结束于偏移 4（INT 4B）
base[2]    = 6                        // b 结束于 6（4 + 2）
base[3]    = 6 | REC_OFFS_SQL_NULL    // c 是 NULL：偏移不推进（仍是 6），置第 31 位
```

取字段 c 时 `rec_get_nth_field_offs_low(offsets, 2, &len)` 返回：起始偏移 = `base[2] & MASK = 6`，`base[3]` 有 SQL_NULL 位 → `len = UNIV_SQL_NULL`。**NULL 字段"占槽位不占空间"的编码在这一格上体现得最清楚**。

#### offsets 的读写 API（`rec_offs_*` 宏族）

全部是 `static inline`，零开销抽象，分三层：

| 层 | 函数 | 作用 |
|---|---|---|
| **头部** | `rec_offs_base(offsets)` | `offsets + REC_OFFS_HEADER_SIZE`，base 区起点 |
| | `rec_offs_get_n_alloc` / `rec_offs_set_n_alloc` | 读/写 `offsets[0]` |
| | `rec_offs_n_fields` / `rec_offs_set_n_fields` | 读/写 `offsets[1]` |
| | `rec_offs_make_valid(rec, index, offsets)` | DEBUG：写 `offsets[2]/[3]`；release 下是 `((void)0)` |
| | `rec_offs_validate(rec, index, offsets)` | DEBUG：校验标志递减、`n_fields` 与 index 一致 |
| **整体** | `rec_offs_extra_size(offsets)` | `base[0] & MASK`，数据区起点到记录头的距离 |
| | `rec_offs_data_size(offsets)` | `base[n] & MASK`，数据区总长（= 最后字段的结束偏移） |
| | `rec_offs_size(offsets)` | 整行大小 = extra + data |
| | `rec_offs_comp(offsets)` | `base[0] & REC_OFFS_COMPACT`（是否 COMPACT 格式） |
| | `rec_offs_any_extern(offsets)` | `base[0] & REC_OFFS_EXTERNAL`（行内有外置字段） |
| **字段** | `rec_offs_nth_size(index, offsets, n)` | 字段物理大小（NULL/DEFAULT/DROP 为 0） |
| | `rec_offs_nth_extern(index, offsets, n)` | 是否外置 |
| | `rec_offs_nth_sql_null(index, offsets, n)` | 是否 NULL |
| | `rec_offs_nth_default(index, offsets, n)` | 是否 instant 默认值 |
| | `rec_offs_make_nth_extern(index, offsets, n)` | 标记外置（`rec_offs_base(offsets)[1+n] \|= REC_OFFS_EXTERNAL`） |
| | `rec_get_nth_field_offs(index, offsets, n, &len)` | **消费方主入口**：返回起始偏移 + 长度 |

带 `index` 参数的版本都先做 `index->get_field_off_pos(n)` 的**逻辑→物理槽位换算**（instant 场景），再调 `_low` 后缀的裸函数。这条分层是 8.0 重构后的统一约定：**`_low` = 物理槽位，非 `_low` = 逻辑字段号**。

**★ 四层封装的完整证据**（每层挑代表函数）：

```cpp
// 第 1 层：rem/rec.h —— 直接摸 offsets[0] / offsets[1] / 基址
static inline ulint *rec_offs_base(ulint *offsets) {
  return offsets + REC_OFFS_HEADER_SIZE;    // ★ release 跳 2，debug 跳 4
}

[[nodiscard]] static inline ulint rec_offs_get_n_alloc(const ulint *offsets) {
  n_alloc = offsets[0];
  ut_ad(n_alloc > REC_OFFS_HEADER_SIZE);
  return (n_alloc);
}

static inline void rec_offs_set_n_fields(ulint *offsets, ulint n_fields) {
  ut_ad(n_fields > 0);
  ut_ad(n_fields <= REC_MAX_N_FIELDS);
  ut_ad(n_fields + REC_OFFS_HEADER_SIZE <= rec_offs_get_n_alloc(offsets));
  offsets[1] = n_fields;
}
```

```cpp
// 第 2 层：include/rem0rec.ic —— 作用于整条 offsets
static inline bool rec_offs_comp(const ulint *offsets) {
  return (*rec_offs_base(offsets) & REC_OFFS_COMPACT) != 0;   // base[0] 的 bit31
}

static inline bool rec_offs_any_extern(const ulint *offsets) {
  return (*rec_offs_base(offsets) & REC_OFFS_EXTERNAL);       // ★ O(1) 判断，无需遍历
}

static inline ulint rec_offs_data_size(const ulint *offsets) {
  size = rec_offs_base(offsets)[rec_offs_n_fields(offsets)] & REC_OFFS_MASK;
  return (size);      // ★ O(1)：最后一个字段的结束偏移
}

static inline ulint rec_offs_extra_size(const ulint *offsets) {
  size = *rec_offs_base(offsets) & ~(REC_OFFS_COMPACT | REC_OFFS_EXTERNAL);
  return (size);      // ★ 只剥 bit31/bit30，不剥 DEFAULT/DROP（那两个只出现在 base[1+i]）
}

static inline ulint rec_offs_size(const ulint *offsets) {
  return (rec_offs_data_size(offsets) + rec_offs_extra_size(offsets));
}
```

```cpp
// 第 3 层：include/rem0lrec.h —— 物理槽位，不翻译 instant
static inline ulint rec_offs_nth_extern_low(const ulint *offsets, ulint n) {
  return (rec_offs_base(offsets)[1 + n] & REC_OFFS_EXTERNAL);
}
static inline ulint rec_offs_nth_sql_null_low(const ulint *offsets, ulint n) {
  return (rec_offs_base(offsets)[1 + n] & REC_OFFS_SQL_NULL);
}
static inline ulint rec_offs_nth_default_low(const ulint *offsets, ulint n) {
  return (rec_offs_base(offsets)[1 + n] & REC_OFFS_DEFAULT);
}
static inline ulint rec_offs_nth_size_low(const ulint *offsets, ulint n) {
  if (!n) {
    return (rec_offs_base(offsets)[1 + n] & REC_OFFS_MASK);   // ★ 第 0 个字段起始恒为 0
  }
  return ((rec_offs_base(offsets)[1 + n] - rec_offs_base(offsets)[n]) & REC_OFFS_MASK);
}
```

```cpp
// 第 4 层：rem/rem0wrec.cc —— 逻辑列号 → 物理槽位
ulint rec_offs_nth_extern(const dict_index_t *index, const ulint *offsets, ulint n) {
  if (index && index->has_row_versions()) {
    n = index->get_field_off_pos(n);       // ★★ instant 的坐标换算就在这里
  }
  validate_rec_offset(index, offsets, n, UT_LOCATION_HERE);
  return (rec_offs_nth_extern_low(offsets, n));
}
```

> ⚠️ **`rem0wrec.h` 在 `storage/innobase/include/`（不在 `rem/`）**；实现在 `rem/rem0wrec.cc`。
> ★ `validate_rec_offset()` 越界时不是 `ut_a` 而是先 `dump_metadata_dict_table()` 再 `ib::fatal`——专门为了定位 instant 列错位。

**★ `rec_get_offsets()` 的权威源码**（⚠️ **6 参**，多了 `ut::Location`，且是 `[[nodiscard]]`）：

```cpp
ulint *rec_get_offsets(const rec_t *rec, const dict_index_t *index,
                       ulint *offsets, ulint n_fields, ut::Location location,
                       mem_heap_t **heap) {
  if (dict_table_is_comp(index->table)) {
    switch (UNIV_EXPECT(rec_get_status(rec), REC_STATUS_ORDINARY)) {
      case REC_STATUS_ORDINARY:  n = dict_index_get_n_fields(index); break;
      case REC_STATUS_NODE_PTR:
        n = dict_index_get_n_unique_in_tree_nonleaf(index) + 1; break;
      case REC_STATUS_INFIMUM:
      case REC_STATUS_SUPREMUM:  n = 1; break;
      default: ut_error;
    }
  } else {
    n = rec_get_n_fields_old(rec, index);
  }

  if (UNIV_UNLIKELY(n_fields < n)) n = n_fields;   // ★ 调用方可只算前 n_fields 个

  ulint size = n + (1 + REC_OFFS_HEADER_SIZE);     // ★★ +1 是 base[0] 的 extra size 槽位

  if (UNIV_UNLIKELY(!offsets) ||
      UNIV_UNLIKELY(rec_offs_get_n_alloc(offsets) < size)) {
    if (UNIV_UNLIKELY(!*heap)) *heap = mem_heap_create(size * sizeof(ulint), location);
    offsets = static_cast<ulint *>(mem_heap_alloc(*heap, size * sizeof(ulint)));
    rec_offs_set_n_alloc(offsets, size);           // ★ 只有"新分配"时才写 n_alloc
  }

  rec_offs_set_n_fields(offsets, n);               // ★ 每次调用都必须刷新
  rec_init_offsets(rec, index, offsets);           // ★ 真正填充 base[0] 与 base[1..n]
  return (offsets);
}
```

> ⚠️ **`size = n + (1 + REC_OFFS_HEADER_SIZE)` 不是 `n + REC_OFFS_HEADER_SIZE`**——多出来的 `1` 就是 `base[0]`。debug 下 header=4 → `size = n + 5`。

**★ `rec_init_offsets_comp_ordinary()` 是 COMPACT 主循环**（⚠️ **非 static**，在 `rem/rec.h:1081` 公开声明，带 `temp` 参数）：

```cpp
  /* read the lengths of fields 0..n */
  ulint offs = 0;
  ulint any_ext = 0;
  ulint null_mask = 1;
  uint16_t i = 0;
  do {
    const dict_field_t *field = index->get_physical_field(i);
    const dict_col_t *col = field->col;
    uint64_t len;

    switch (rec_insert_state) {
      case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION: {
        if (col->is_dropped_in_or_before(row_version)) {
          len = offs | REC_OFFS_DROP;              // ★ instant DROP：打 bit28，offs 不推进
          goto resolved;
        } else if (col->is_added_after(row_version)) {
          len = rec_get_instant_offset(index, i, offs);  // ★ instant ADD：打 bit31 或 bit29
          goto resolved;
        }
      } break;
      ...
    }

    if (!(col->prtype & DATA_NOT_NULL)) {
      ut_ad(n_null--);
      if (UNIV_UNLIKELY(!(byte)null_mask)) { nulls--; null_mask = 1; }
      if (*nulls & null_mask) {
        null_mask <<= 1;
        len = offs | REC_OFFS_SQL_NULL;            // ★ NULL：不推进 offs，只打 bit31
        goto resolved;
      }
      null_mask <<= 1;
    }

    if (!field->fixed_len || (temp && !col->get_fixed_size(temp))) {
      len = *lens--;                               // ★ 变长长度数组：向低地址倒着读
      if (DATA_BIG_COL(col)) {
        if (len & 0x80) {                          // ★ 两字节形式
          len <<= 8;
          len |= *lens--;
          offs += len & 0x3fff;                    // ★ 低 14 位 = 本地前缀长度
          if (UNIV_UNLIKELY(len & 0x4000)) {       // ★ bit14 = extern
            ut_ad(index->is_clustered());
            any_ext = REC_OFFS_EXTERNAL;
            len = offs | REC_OFFS_EXTERNAL;
          } else {
            len = offs;
          }
          goto resolved;
        }
      }
      len = offs += len;
    } else {
      len = offs += field->fixed_len;
    }
  resolved:
    rec_offs_base(offsets)[i + 1] = len;
  } while (++i < rec_offs_n_fields(offsets));

  *rec_offs_base(offsets) = (rec - (lens + 1)) | REC_OFFS_COMPACT | any_ext;
  //                        ★★★ (lens+1) = 变长长度数组的末尾边界；rec - 末尾 = extra size
```

> ★★ **`*rec_offs_base(offsets) = (rec - (lens + 1)) | ...` 这一行是整个 offsets 体系的枢纽**——`lens` 在循环里一路向低地址递减，循环结束时它正好指向"最后一个被读的 len 字节"；`lens + 1` 就是变长长度数组的**起始边界**，`rec` 减去它就是 extra_size。

**★ NULL bitmap / lens 的定位**（`rec_init_null_and_len_comp`）：

```cpp
  *nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);       // ★ 默认：origin - 6
  ...
  switch (rec_insert_state) {
    case INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION:
      *n_null = index->n_nullable;                  // 最简情形
      break;
    case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION:
      row_version = (uint8_t)(**nulls);             // ★ version 就存在 nulls 起点那 1 字节
      *nulls -= 1;                                  // ★ nulls 再往前挪 1 字节越过 version
      *n_null = index->get_nullable_in_version(row_version);
      break;
    case INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION:
      non_default_fields = rec_get_n_fields_instant(rec, REC_N_NEW_EXTRA_BYTES, &length);
      *nulls -= length;                             // ★ V1：记录里存了"字段数"（1 或 2 字节）
      *n_null = index->calculate_n_instant_nullable(non_default_fields);
      break;
    ...
  }
  *lens = *nulls - UT_BITS_IN_BYTES(*n_null);       // ★ lens 紧贴 null bitmap 的低地址侧
```

**★ `REC_INSERT_STATE` 六态**（instant ADD/DROP 后的行有 6 种形态，源码给了完整表格注释）：

```cpp
enum REC_INSERT_STATE {
  INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION,
  INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION,
  INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION,
  INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION,
  INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION,
  INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION,
  NONE
};
```

```
  +----------------------------------------------------------------------------+
  |              SCENARIO                         |        STATE               |
  |----------------------------------+------------+---------+------------------|
  |            Row property          | V <= 8.0.28| Bit set | Stored on row    |
  +----------------------------------+------------+---------+------------------+
  | INSERTED before 1st INSTANT ADD  |     Y      | NONE    | N/A              |
  | INSERTED after 1st INSTANT ADD   |     Y      | INSTANT | # of fields      |
  | INSERTED before INSTANT ADD/DROP |     Y      | VERSION | Version = 0      |
  | INSERTED before INSTANT ADD/DROP |     N      | NONE    | N/A              |
  | INSERTED after INSTANT ADD/DROP  |     Y/N    | VERSION | row version      |
  +----------------------------------------------------------------------------+
```

> ★★ **"这两个 bit 互斥"有独立校验**：`get_rec_insert_state()` 里 `ut_ad(!is_versioned || !is_instant)`，且 release 下会打 error 日志提示"table may be corrupt"。
> ★ `rec_get_instant_offset()` 是 instant 列的标记助手：默认值是 NULL → 打 `REC_OFFS_SQL_NULL`；否则打 `REC_OFFS_DEFAULT`。

**★ REDUNDANT 的两种偏移目录**（与 COMPACT 完全对照）：

```cpp
static void rec_init_offset_old_1byte(const rec_t *rec, const dict_index_t *index,
                                      ulint *offsets, uint8_t row_version) {
  ulint offs = REC_N_OLD_EXTRA_BYTES;               // ★ REDUNDANT 固定 6 字节 header
  offs += rec_old_is_versioned(rec) ? 1 : 0;        // ★ V2 多 1 字节 row version
  offs += rec_get_n_fields_old_raw(rec);            // ★ 1 字节/字段的偏移目录
  *rec_offs_base(offsets) = offs;                   // ★★ REDUNDANT 不打 REC_OFFS_COMPACT
  ...
    if (offs & REC_1BYTE_SQL_NULL_MASK) {           // 0x80
      offs &= ~REC_1BYTE_SQL_NULL_MASK;
      offs |= REC_OFFS_SQL_NULL;                    // ★ 翻译成 base 的 bit31
    }
    rec_offs_base(offsets)[1 + i] = offs;
}
```

```cpp
  // 2 字节版多了 extern 标志的翻译
    if (offs & REC_2BYTE_SQL_NULL_MASK) {           // 0x8000
      offs &= ~REC_2BYTE_SQL_NULL_MASK;
      offs |= REC_OFFS_SQL_NULL;
    }
    if (offs & REC_2BYTE_EXTERN_MASK) {             // 0x4000
      offs &= ~REC_2BYTE_EXTERN_MASK;
      offs |= REC_OFFS_EXTERNAL;
      *rec_offs_base(offsets) |= REC_OFFS_EXTERNAL; // ★★ 同时 OR 进 base[0]
    }
```

> ★★ REDUNDANT 的"偏移目录"**存在行里**（每字段 1 或 2 字节），`rec_init_offsets_old` 只是把行内目录**翻译**成统一的 offsets 编码（把 `0x80`/`0x8000`/`0x4000` 这三个行内标志翻译成 `REC_OFFS_SQL_NULL`/`EXTERNAL`）。这正是"COMPACT 省掉目录、换运行时解析"的对照面。

**★ 字段定位的最终入口**：

```cpp
static inline ulint rec_get_nth_field_offs_low(const ulint *offsets, ulint n,
                                               ulint *len) {
  if (n == 0) {
    offs = 0;                                       // ★ 第 0 个字段起始偏移恒为 0
  } else {
    offs = rec_offs_base(offsets)[n] & REC_OFFS_MASK;  // ★ base[n] = 第 n 个字段的起始
  }

  length = rec_offs_base(offsets)[1 + n];           // ★ base[1+n] = 第 n 个字段的结束

  if (length & REC_OFFS_SQL_NULL)       length = UNIV_SQL_NULL;
  else if (length & REC_OFFS_DEFAULT)   length = UNIV_SQL_ADD_COL_DEFAULT;
  else if (length & REC_OFFS_DROP)      length = UNIV_SQL_INSTANT_DROP_COL;
  else { length &= REC_OFFS_MASK; length -= offs; }  // ★ 结束 - 起始 = 真实长度

  *len = length;
  return (offs);
}
```

```cpp
static inline const byte *rec_get_nth_field_instant(const rec_t *rec,
                                                    const ulint *offsets, ulint n,
                                                    const dict_index_t *index,
                                                    ulint *len) {
  if (index->has_row_versions()) {
    n = index->get_field_off_pos(n);
  }
  ulint off = rec_get_nth_field_offs(nullptr, offsets, n, len);

  if (*len != UNIV_SQL_ADD_COL_DEFAULT && *len != UNIV_SQL_INSTANT_DROP_COL) {
    return (rec + off);                             // ★ 正常值 / NULL
  }
  if (*len == UNIV_SQL_INSTANT_DROP_COL) {
    *len = 0;
    return nullptr;                                 // ★ 已 instant DROP
  }
  return (index->get_nth_default(n, len));          // ★★ instant ADD：取 dict 里的默认值
}
```

> ★★ **这是取行路径真正用的函数**——返回的指针**可能不指向 rec**（instant ADD 列指向 dict 缓存的默认值）。任何后续 `mem_heap_dup` / BLOB 处理都必须考虑这一点。

#### 三兄弟结构体：`dtuple_t` / `dfield_t` / `dtype_t`

物理行（`rec_t`）的对偶是内存中的逻辑行（`dtuple_t`）。三个结构层层嵌套：

```cpp
/* ① 类型：位域打包，共 61 bit —— 8 字节内塞进一个字段类型的全部信息 */
struct dtype_t {
  unsigned prtype : 32;        /* precise type：MySQL 类型 + 字符集 + NULLable +
                                  unsigned + binary + true VARCHAR 标志
                                  + DATA_VIRTUAL 等语义位 */
  unsigned mtype : 8;          /* main data type：INT/CHAR/BLOB/FLOAT... */
  /* 以下不影响排序 */
  unsigned len : 16;           /* 长度：通常是 field->pack_length()；
                                  true VARCHAR 是最大字节长度 */
  unsigned mbminmaxlen : 5;    /* 字符集每字符的最小/最大字节数打包：
                                  DATA_MBMINMAXLEN(mbminlen, mbmaxlen)，
                                  mbminlen = DATA_MBMINLEN(mbminmaxlen) */
  bool is_virtual() const { return ((prtype & DATA_VIRTUAL) == DATA_VIRTUAL); }
};

/* ② 一个字段：数据指针 + 长度 + 类型，外加两个状态位 */
struct dfield_t {
  void *data;                      /* 数据指针（可指向页内、堆、外置 BLOB ref） */
  bool ext;                        /* true = 外置存储（行内是 20B BLOB ref） */
  unsigned spatial_status : 2;     /* 外置字段在 undo 里的空间状态（purge 用） */
  unsigned len;                    /* 数据长度；UNIV_SQL_NULL 表示 SQL NULL */
  dtype_t type;                    /* 类型（含 DATA_VIRTUAL 等标志） */

  bool is_virtual() const { return (type.is_virtual()); }
  void reset() { data = nullptr; ext = false; spatial_status = SPATIAL_UNKNOWN; len = 0; }
  dfield_t *clone(mem_heap_t *heap);   /* 深拷贝 */
  byte *blobref() const;               /* 取外置引用 */
  uint32_t lob_version() const;        /* 取 LOB 版本号 */
};

/* ③ 逻辑行：两段 dfield 数组（物理列 + 虚拟列） */
struct dtuple_t {
  uint16_t info_bits;        /* 索引记录的信息位（deleted / version / instant） */
  uint16_t n_fields;         /* 字段总数 */
  uint16_t n_fields_cmp;     /* ★ 参与比较的字段数（默认 = n_fields） */
  dfield_t *fields;          /* ← 物理列数组 */
  uint16_t n_v_fields;       /* 虚拟列数 */
  dfield_t *v_fields;        /* ← 虚拟列数组（dtuple_create_with_vcol 创建） */
  UT_LIST_NODE_T(dtuple_t) tuple_list;  /* 见下：插入时"每索引一个 entry"的链表节点 */
#ifdef UNIV_DEBUG
  mem_heap_t *m_heap{};
  static constexpr size_t MAGIC_N = 614679;   /* 魔数，debug 断言用 */
  size_t magic_n{MAGIC_N};
#endif
  int compare(const rec_t *rec, const dict_index_t *index, const ulint *offsets,
              ulint *matched_fields) const;   /* 与物理行比较 */
  trx_id_t get_trx_id() const;                /* 读 DB_TRX_ID */
  void ignore_trailing_default(const dict_index_t *index);  /* instant 尾部默认值处理 */
};
```

**三个设计点值得单独说**：

1. **`n_fields_cmp` 是"前缀比较"的开关**：搜索元组（search tuple）只比较前 `n_fields_cmp` 个字段，其余忽略——range 优化器构造的"部分键"搜索元组靠它（例如 `WHERE a = 1` 用 `(a, b, c)` 索引的部分前缀）。这是 dtuple 相比 rec 独有的能力，物理行没有这个概念。
2. **`dfield_t::len` 用 `UNIV_SQL_NULL` 表示 NULL**，与 offsets 里的 `REC_OFFS_SQL_NULL` 是**两套表达**（内存态 vs 解析态）——`rec_get_nth_field_offs_low` 负责在两者间翻译。
3. **`dtype_t` 的位域打包**让一个字段类型只占 8 字节且可直接按位比较排序相关部分（`prtype` + `mtype` 影响排序，`len`/`mbminmaxlen` 不影响——注释明说 "the remaining fields do not affect alphabetical ordering"）。

**rec ↔ dtuple 的双向转换**就是两条链（本篇「核心实现」的两条主链）：

| 方向 | 函数 | 用途 |
|---|---|---|
| dtuple → rec | `rec_convert_dtuple_to_rec` | 插入/分裂时把逻辑行编码成物理行 |
| rec → dtuple | `row_build`（`ROW_COPY_DATA` / `ROW_COPY_POINTERS`） | 取行、purge、undo 回放 |
| rec → dtuple（前缀） | `rec_copy_prefix_to_dtuple` | 只拷前 n 个字段（搜索/比较用） |

`ROW_COPY_POINTERS` 是浅拷贝（dfield 只指向页内数据，不复制），`ROW_COPY_DATA` 深拷贝到堆——**绝大多数读路径用 POINTERS 避免拷贝**，只在需要跨 mtr 存活时才 DATA。

#### 配套结构：`tuple_list` 与 `big_rec_t`

**`tuple_list` 的真实用途**（易被误解成"行版本链"）：它是**插入时"每个索引一个 entry"的链表节点**——`ins_node_t` 用它把同一行在所有索引上的 tuple 串起来：

```cpp
/* row0ins.h */
struct ins_node_t {
  dict_index_t *index;                       // 当前正在插入的索引
  dtuple_t *entry;                           // 当前索引的 entry
  UT_LIST_BASE_NODE_T(dtuple_t, tuple_list)
      entry_list;                            // ★ list of entries, one for each index
  ...
};
```

插入主循环两个链表**同步推进**（`row0mysql.cc:1465`、`row0ins.cc:3583`）：

```cpp
for (dict_index_t *index = UT_LIST_GET_FIRST(node->table->indexes);
     index != nullptr;
     index = UT_LIST_GET_NEXT(indexes, index),
     node->entry = UT_LIST_GET_NEXT(tuple_list, node->entry)) {   // 索引链与 entry 链并行走
  row_ins_index_entry_set_vals(node->index, node->entry, node->row);
  ...
}
```

即 `dict_table_t::indexes` 与 `ins_node_t::entry_list` 是**两条平行链表、一一对应**，`tuple_list` 就是后者的链接字段。回滚时（`row_explicit_rollback`）同样按这条链逐个索引撤销。

**`big_rec_t`：外置列的"附属容器"**（`data0data.h:831`）。`dtuple_t` 只装得下**行内**数据；溢出到 LOB 页的列单独用 `big_rec_t` 承载：

```cpp
struct big_rec_t {
  mem_heap_t *heap;          // 分配堆
  const ulint capacity;      // fields 数组容量
  ulint n_fields;            // 实际存了几个外置字段
  big_rec_field_t *fields;   // 外置字段数组（含 field_no / 数据 / 长度）
  static big_rec_t *alloc(mem_heap_t *heap, ulint n_fld);
  void append(const big_rec_field_t &field);
};
```

插入流程因此分两步：先 `dtuple` 写行内部分（外置列留 20B BLOB ref，对应 `dfield_t::ext = true`），再由 `BtrContext::store_big_rec_extern_fields` / `lob::insert` 把 `big_rec_t` 里的外置数据写入 LOB 页。详见 [`lob.md`](lob.md)。

**`dtuple_t::get_trx_id()`**（`data0data.cc:783`）是个小而典型的例子——它用 `dtype_t` 的两个字段联合定位系统列：

```cpp
trx_id_t dtuple_t::get_trx_id() const {
  for (ulint i = 0; i < n_fields; ++i) {
    dfield_t &field = fields[i];
    uint32_t prtype = field.type.prtype & DATA_SYS_PRTYPE_MASK;   // 精确类型（低 16 位）
    if (field.type.mtype == DATA_SYS && prtype == DATA_TRX_ID) {  // 主类型=系统列 且 精确类型=TRX_ID
      return mach_read_from_6((byte *)field.data);                // trx_id 是 6 字节
    }
  }
  return 0;
}
```

注意 `DATA_SYS_PRTYPE_MASK` 屏蔽掉高位标志（NULLable、字符集等）后再比对——这正是 `dtype_t` 把 `prtype` 设计成"类型 + 标志位打包"的直接用法。

复杂度：解析一次 O(字段数 + 变长字段数)；之后字段定位 O(1)；空间 O(字段数) 个 ulint（栈上 100 个免分配）。

---

## 核心实现

### 主链路

**读方向（解析行）**：

```
rec_get_offsets(rec, index, offsets_, ULINT_UNDEFINED, &heap)   // 入口：算 n_fields、判容量
  → rec_init_offsets(rec, index, offsets)                       // 按格式分发
    → rec_init_offsets_new                                     // COMPACT/DYNAMIC
      → rec_init_offsets_comp_ordinary                         // 普通叶记录：解析核心
        → rec_init_null_and_len_comp                           // 定位 nulls/lens、判 instant 状态
    → rec_init_offsets_old                                     // REDUNDANT
  → rec_get_nth_field_offs(rec, offsets, n, &len)              // 消费：字段定位
    → rec_get_nth_field_offs_low                                // 前缀和 → 偏移/长度
```

**写方向（编码行）**：

```
rec_convert_dtuple_to_rec(buf, index, dtuple)
  → rec_convert_dtuple_to_rec_new
    → rec_convert_dtuple_to_rec_comp                            // store_field lambda 逐字段编码
  → rec_set_info_and_status_bits                                // 写头信息位
```

**★ 记录头全部字节偏移常量**（⚠️ **不在 `rem0rec.h`，真身在 `rem/rec.h`**；且全部是 `constexpr`，不是 `#define`）：

```cpp
/* The offset of heap_no in a compact record */
constexpr uint32_t REC_NEW_HEAP_NO = 4;
/* The shift of heap_no in a compact record.
The status is stored in the low-order bits. */
constexpr uint32_t REC_HEAP_NO_SHIFT = 3;

constexpr uint32_t REC_NEXT = 2;                       // ★ 2 字节 next 指针（相对或绝对偏移）
constexpr uint32_t REC_NEXT_MASK = 0xFFFFUL;
constexpr uint32_t REC_NEXT_SHIFT = 0;

constexpr uint32_t REC_OLD_SHORT = 3;                  // ★ REDUNDANT 的 "1 字节 offsets" 标志
constexpr uint32_t REC_OLD_SHORT_MASK = 0x1UL;
constexpr uint32_t REC_OLD_SHORT_SHIFT = 0;

constexpr uint32_t REC_OLD_N_FIELDS = 4;               // ★ REDUNDANT 的字段数（跨字节 11 位）
constexpr uint32_t REC_OLD_N_FIELDS_MASK = 0x7FEUL;
constexpr uint32_t REC_OLD_N_FIELDS_SHIFT = 1;

constexpr uint32_t REC_NEW_STATUS = 3;                 // ★ COMPACT 的 status（低 3 位）
constexpr uint32_t REC_NEW_STATUS_MASK = 0x7UL;
constexpr uint32_t REC_NEW_STATUS_SHIFT = 0;

constexpr uint32_t REC_OLD_HEAP_NO = 5;
constexpr uint32_t REC_HEAP_NO_MASK = 0xFFF8UL;

constexpr uint32_t REC_OLD_N_OWNED = 6;
constexpr uint32_t REC_NEW_N_OWNED = 5;
constexpr uint32_t REC_N_OWNED_MASK = 0xFUL;
constexpr uint32_t REC_N_OWNED_SHIFT = 0;

constexpr uint32_t REC_OLD_INFO_BITS = 6;
constexpr uint32_t REC_NEW_INFO_BITS = 5;
constexpr uint32_t REC_TMP_INFO_BITS = 1;              // ★ temp 文件记录只有 1 字节 extra
constexpr uint32_t REC_INFO_BITS_MASK = 0xF0UL;        // ★ 高 4 位
constexpr uint32_t REC_INFO_BITS_SHIFT = 0;
```

> ⚠️ **不存在 `REC_N_OWNED` / `REC_INFO_BITS` 这两个裸名**——真名是 `REC_OLD_N_OWNED` / `REC_NEW_N_OWNED` 与 `REC_OLD_INFO_BITS` / `REC_NEW_INFO_BITS` / `REC_TMP_INFO_BITS`，只是共用 MASK/SHIFT。
> ⚠️ **`REC_N_NEW_EXTRA_BYTES` 的类型是 `constexpr int32_t`（有符号！）**，其余两个是 `uint32_t`——这是为了让 `nulls -= 1` 这类减法不会整体提升成无符号。

**★ 两条编译期自校验**（把"头布局"钉死在编译期）：

```cpp
static_assert((REC_OLD_SHORT_MASK << (8 * (REC_OLD_SHORT - 3)) ^
               REC_OLD_N_FIELDS_MASK << (8 * (REC_OLD_N_FIELDS - 4)) ^
               REC_HEAP_NO_MASK << (8 * (REC_OLD_HEAP_NO - 4)) ^
               REC_N_OWNED_MASK << (8 * (REC_OLD_N_OWNED - 3)) ^
               REC_INFO_BITS_MASK << (8 * (REC_OLD_INFO_BITS - 3)) ^
               0xFFFFFFFFUL) == 0,
              "sum of old-style masks != 0xFFFFFFFFUL");
//  ★★ REDUNDANT 6 字节头各 mask 异或后必须拼满 0xFFFFFFFF（即各字段不重叠且无缝）

static_assert((REC_NEW_STATUS_MASK << (8 * (REC_NEW_STATUS - 3)) ^
               REC_HEAP_NO_MASK << (8 * (REC_NEW_HEAP_NO - 4)) ^
               REC_N_OWNED_MASK << (8 * (REC_NEW_N_OWNED - 3)) ^
               REC_INFO_BITS_MASK << (8 * (REC_NEW_INFO_BITS - 3)) ^
               0xFFFFFFUL) == 0,
              "sum of new-style masks != 0xFFFFFFUL");
//  ★★ COMPACT 头（不含 REC_NEXT 那 2 字节）拼满 0xFFFFFF
```

**★ 两个"位域布局"的 ASCII 原文**（在 **`include/rem0rec.ic:40`**，不是 `rem0rec.h`；两张图，都含 "downward from origin"）：

```
/* Offsets of the bit-fields in an old-style record. NOTE! In the table the
most significant bytes and bits are written below less significant.

        (1) byte offset         (2) bit usage within byte
        downward from
        origin ->       1       8 bits pointer to next record
                        2       8 bits pointer to next record
                        3       1 bit short flag
                                7 bits number of fields
                        4       3 bits number of fields
                                5 bits heap number
                        5       8 bits heap number
                        6       4 bits n_owned
                                4 bits info bits
*/
```

```
/* Offsets of the bit-fields in a new-style record. ...

        origin ->       1       8 bits relative offset of next record
                        2       8 bits relative offset of next record
                                  the relative offset is an unsigned 16-bit
                                  integer:
                                  (offset_of_next_record
                                   - offset_of_this_record) mod 64Ki, ...
                        3       3 bits status:
                                        000=conventional record
                                        001=node pointer record (inside B-tree)
                                        010=infimum record
                                        011=supremum record
                                        1xx=reserved
                                5 bits heap number
                        4       8 bits heap number
                        5       4 bits n_owned
                                4 bits info bits
*/
```

> ★★ **"downward from origin"是所有 `rec - REC_XXX` 写法的来源**：`origin` 是数据区起点，头部字节在**低地址方向**。

**★ `REC_INFO_*` 四个标志位** + 合法值校验：

```cpp
constexpr uint32_t REC_INFO_MIN_REC_FLAG = 0x10UL;   // bit4：B-tree 最左非叶页的首条用户记录
constexpr uint32_t REC_INFO_DELETED_FLAG = 0x20UL;   // bit5：delete mark
constexpr uint32_t REC_INFO_VERSION_FLAG = 0x40UL;   // bit6：有 row version（instant V2）
constexpr uint32_t REC_INFO_INSTANT_FLAG = 0x80UL;   // bit7：instant ADD COLUMN（V1）
```

```cpp
inline bool rec_info_bits_valid(ulint bits) {
  return (0 == (bits & ~(REC_INFO_DELETED_FLAG | REC_INFO_MIN_REC_FLAG |
                         REC_INFO_INSTANT_FLAG | REC_INFO_VERSION_FLAG)));
}
```

> ★★ **bit6 与 bit7 互斥**（`rec_new_set_instant()` 置 7 清 6，`rec_new_set_versioned()` 置 6 清 7）——源码里甚至有两个同名重载 `rec_new_set_versioned(rec)` / `rec_new_set_versioned(rec, flag)`，靠参数区分。

```cpp
constexpr uint32_t REC_N_OLD_EXTRA_BYTES = 6;   // REDUNDANT
constexpr int32_t  REC_N_NEW_EXTRA_BYTES = 5;   // COMPACT/DYNAMIC  ★ int32_t
constexpr uint32_t REC_N_TMP_EXTRA_BYTES = 1;   // temp 文件记录（只放 info-bits）
```

```cpp
constexpr uint32_t REC_STATUS_ORDINARY  = 0;
constexpr uint32_t REC_STATUS_NODE_PTR  = 1;
constexpr uint32_t REC_STATUS_INFIMUM   = 2;
constexpr uint32_t REC_STATUS_SUPREMUM  = 3;
constexpr uint32_t REC_NODE_PTR_SIZE    = 4;
```

```cpp
constexpr uint32_t REC_1BYTE_SQL_NULL_MASK = 0x80UL;    // REDUNDANT 1 字节偏移的 NULL 位
constexpr uint32_t REC_2BYTE_SQL_NULL_MASK = 0x8000UL;  // REDUNDANT 2 字节偏移的 NULL 位
constexpr uint32_t REC_2BYTE_EXTERN_MASK   = 0x4000UL;  // REDUNDANT 2 字节偏移的 extern 位
constexpr ulint REC_1BYTE_OFFS_LIMIT = 0x7FUL;
constexpr ulint REC_2BYTE_OFFS_LIMIT = 0x7FFFUL;
constexpr ulint REC_MAX_DATA_SIZE   = 16384;
```

**★ next 指针：COMPACT 是相对偏移，REDUNDANT 是绝对偏移**（这是两种格式在"记录链"上的根本差异）：

```cpp
static inline ulint rec_get_next_offs(const rec_t *rec, ulint comp) {
  field_value = mach_read_from_2(rec - REC_NEXT);
  if (comp) {
    return (ut_align_offset(rec + field_value, UNIV_PAGE_SIZE));   // ★ 相对，按页环绕
  } else {
    return (field_value);                                          // ★ 绝对页内偏移
  }
}

static inline void rec_set_next_offs_new(rec_t *rec, ulint next) {
  if (!next) { field_value = 0; }
  else {
    field_value = (ulint)((lint)next - (lint)ut_align_offset(rec, UNIV_PAGE_SIZE));
    field_value &= REC_NEXT_MASK;      // ★ 截断到 16 位 → "mod 64Ki"
  }
  mach_write_to_2(rec - REC_NEXT, field_value);
}
```

> ★★ **页内没有 prev 指针**——`page_rec_get_prev_const()` 必须先 `page_dir_find_owner_slot()` 找到所属 slot，再从**上一个 slot 的 owner** 沿 next 单链表线性扫描。这是"反向扫描慢"的又一个来源。

### 行的物理布局（record header）

COMPACT/DYNAMIC 行的固定头部 5 字节（`REC_N_NEW_EXTRA_BYTES`），REDUNDANT 为 6 字节（多 10 位的字段数）。所有字段位置由 `rem/rec.h` 的位域常量表定义：

```cpp
constexpr uint32_t REC_NEXT = 2;              // next 指针：2B 相对偏移（页内链）
constexpr uint32_t REC_NEXT_MASK = 0xFFFFUL;

constexpr uint32_t REC_NEW_STATUS = 3;        // 记录类型：0 普通 / 1 node_ptr / 2 infimum / 3 supremum
constexpr uint32_t REC_NEW_STATUS_MASK = 0x7UL;

constexpr uint32_t REC_NEW_HEAP_NO = 4;       // heap 序号（插入序），低位 3bit 归 status
constexpr uint32_t REC_HEAP_NO_MASK = 0xFFF8UL;
constexpr uint32_t REC_HEAP_NO_SHIFT = 3;

constexpr uint32_t REC_NEW_N_OWNED = 5;       // 低 4bit：该记录被 page directory 槽"拥有"的记录数
constexpr uint32_t REC_N_OWNED_MASK = 0xFUL;

constexpr uint32_t REC_NEW_INFO_BITS = 5;     // 高 4bit：info bits
constexpr uint32_t REC_INFO_BITS_MASK = 0xF0UL;

/* Info bit 的四个语义位 */
constexpr uint32_t REC_INFO_MIN_REC_FLAG = 0x10UL;   // 最左页的 node_ptr 最小记录
constexpr uint32_t REC_INFO_DELETED_FLAG = 0x20UL;   // 删除标记（purge 前）
constexpr uint32_t REC_INFO_VERSION_FLAG = 0x40UL;   // 该行带 1 字节行版本号（instant V2）
constexpr uint32_t REC_INFO_INSTANT_FLAG = 0x80UL;   // 该行是 instant ADD 后写的（V1 语义）
```

字节序细节：`rec` 指针指向**数据区的起点**（记录 origin），头字段用 `rec - offset` 向下取。信息位的读取统一走 `rec_get_bit_field_1(rec, offs, mask, shift)`。

头部之后（向低地址方向）依次是：

```
低地址
  │  [变长字段长度数组]（逆序：后字段的在前）
  │  [NULL 位图]        （1 bit/可空字段，按字段序）
  │  [行版本号 1B]      （仅 REC_INFO_VERSION_FLAG 置位时）
  │  [记录头 5B]
  ▼ rec（数据区起点：字段 0 的数据）
高地址
```

### offsets 数组的布局与编码（`rec_offs_*` 宏族）

前面讲的是 offsets **怎么生成、怎么用来定位字段**，但**数组本身长什么样**一直没系统说清——而这正是所有 `rec_offs_*` 宏的共同前提。

**整体布局**——一块连续的 `ulint[]`：

```
offsets[0]      = n_fields     （rec_offs_n_fields / rec_offs_set_n_fields）
offsets[1]      = n_alloc      （rec_offs_get_n_alloc / rec_offs_set_n_alloc）
offsets[2]      = rec 的地址   ★ 仅 UNIV_DEBUG 构建存在
offsets[3]      = index 的地址 ★ 仅 UNIV_DEBUG 构建存在
─────────────── REC_OFFS_HEADER_SIZE = 2，★ DEBUG 下为 4（见下），以下即 rec_offs_base(offsets)
base[0]         = extra_size | COMPACT | EXTERNAL      ← 位打包
base[1 + n]     = 第 n 个字段的偏移 | EXTERNAL / NULL / DEFAULT …   ← 位打包
```

⇒ 总容量 = `n + 1 + REC_OFFS_HEADER_SIZE`（恰好对应上一节 `rec_get_offsets` 里的容量判断）。

**★ DEBUG 专属的两个元素**（`rem/rec.h:174-179`）：

```cpp
#ifdef UNIV_DEBUG
constexpr uint32_t REC_OFFS_HEADER_SIZE = 4;
#else
constexpr uint32_t REC_OFFS_HEADER_SIZE = 2;
#endif
```

多出来的 `offsets[2]` / `offsets[3]` 存的是 **`rec` 与 `index` 的地址**，供 `rec_offs_validate()` 断言"这份 offsets 确实是为**这条记录、这个索引**算的"（`rec.h:554/563`）：

```cpp
ut_ad((ulint)rec == offsets[2] || ((ulint)nullptr == offsets[2] &&
                                   offsets == index->rec_cache.offsets));
ut_ad((ulint)index == offsets[3]);
```

⇒ ★ 第一个断言里那个 `||` 分支很关键：**共享的"固定 offsets 缓存"不属于任何一条记录**，所以它的 `offsets[2] == 0`；此时改为校验"这个数组确实就是 `index->rec_cache.offsets` 本身"。这样一套断言同时覆盖了"专属 offsets"与"共享 offsets"两种情形。

**另一个容易漏的点**：`rec_offs_data_size()` 并不额外存一个字段——

```cpp
size = rec_offs_base(offsets)[rec_offs_n_fields(offsets)] & REC_OFFS_MASK;
```

它取的是 `base[n_fields]`，也就是**最后一个字段的结束偏移**——那自然就是整个数据区的大小。⇒ 所以 `base` 部分只有 `1 + n_fields` 个元素，没有专门的 "data_size 槽位"。

**★ 位打包是关键**：每个 ulint 的**低位比特借给标志位**，高位仍存数值。所以取值必须先掩码——同一个 `base[0]` 能被解读出三种语义（`rem0rec.ic`）：

```cpp
rec_offs_comp(offsets)         → (*rec_offs_base(offsets) & REC_OFFS_COMPACT) != 0
rec_offs_any_extern(offsets)   → (*rec_offs_base(offsets) & REC_OFFS_EXTERNAL)
rec_offs_extra_size(offsets)   → *rec_offs_base(offsets) & ~(REC_OFFS_COMPACT | REC_OFFS_EXTERNAL)
```

即：既能问"是不是 COMPACT 格式"、又能问"有没有外置字段"、还能把标志位掩掉取出 `extra_size`。**三个语义挤在一个 ulint 里**。

前面那个真实例子现在就通了——COMPACT 表 `t(a INT NOT NULL, b VARCHAR(10), c INT)` 插入 `(1,'ab',NULL)`：

```
base[0] = 7 | REC_OFFS_COMPACT    // extra_size = 5(头) + 1(NULL位图) + 1(变长长度)
```

**字段偏移同样打包**（`rem0lrec.h:224`）：

```cpp
rec_offs_nth_extern(offsets, n)  → rec_offs_base(offsets)[1 + n] & REC_OFFS_EXTERNAL
```

⇒ `base[1+n]` 里既有第 n 个字段的**偏移**，又用低位标记"是否外置 / 是否 NULL / 是否 default / 是否 instant 列"（`rec_offs_nth_default`、`rec_offs_nth_sql_null`、`rec_offs_nth_instant` 同理——最后这个标记 **instant ADD COLUMN 加进来的、行里可能根本没存值**的列）。

另外 `rec_offs_any_null_extern(offsets)` 是**整条记录级**的快速判断：`base[0]` 里一个比特就能知道"这条记录有没有 NULL 或外置字段"，用来快速跳过慢路径——不必逐字段查。

**为什么这么设计**——三个理由：

1. **省内存**：一个字段只占 1 个 ulint，不额外开标志数组。offsets 是**每条记录都要算**的东西，热路径上少一个数组就少一次分配
2. **一次分配**：`n + 1 + REC_OFFS_HEADER_SIZE` 个 ulint 一次性算完，栈上 100 个够用就完全不碰 heap
3. **可原位重填**：`rec_offs_set_n_alloc` 记下容量后，同一块数组能被下一条记录重写复用（下面「固定 offsets 缓存与 `Rec_offsets`」就靠这个）

**★ 代价**：`offsets[i]` 的**原始值不能当偏移量直接用**——必须先掩掉标志位。这也是为什么代码里到处是 `rec_offs_nth_*` 宏而不是裸下标：裸读会拿到"偏移 + 标志位"的混合值。

### offsets 的生成：rec_get_offsets → rec_init_offsets_comp_ordinary

**入口**先决定字段数与容量：

```cpp
ulint *rec_get_offsets(const rec_t *rec, const dict_index_t *index,
                       ulint *offsets, ulint n_fields, ut::Location location,
                       mem_heap_t **heap) {
  ulint n;

  if (dict_table_is_comp(index->table)) {
    switch (UNIV_EXPECT(rec_get_status(rec), REC_STATUS_ORDINARY)) {
      case REC_STATUS_ORDINARY:
        n = dict_index_get_n_fields(index);      // 普通记录：字段数 = index 字段数
        break;
      case REC_STATUS_NODE_PTR:
        n = dict_index_get_n_unique_in_tree_nonleaf(index) + 1;  // 非叶：唯一键 + 子页号
        break;
      case REC_STATUS_INFIMUM:
      case REC_STATUS_SUPREMUM:
        n = 1;                                   // 伪记录
        break;
      ...
    }
  } else {
    n = rec_get_n_fields_old(rec, index);        // REDUNDANT：字段数存在行头里
  }

  if (UNIV_UNLIKELY(n_fields < n)) n = n_fields;  // 可要求只解析前 n_fields 个

  /* 容量判断：栈上 100 个够不够？不够去 heap */
  ulint size = n + (1 + REC_OFFS_HEADER_SIZE);
  if (UNIV_UNLIKELY(!offsets) ||
      UNIV_UNLIKELY(rec_offs_get_n_alloc(offsets) < size)) {
    if (UNIV_UNLIKELY(!*heap)) {
      *heap = mem_heap_create(size * sizeof(ulint), location);
    }
    offsets = static_cast<ulint *>(mem_heap_alloc(*heap, size * sizeof(ulint)));
    rec_offs_set_n_alloc(offsets, size);
  }

  rec_offs_set_n_fields(offsets, n);
  rec_init_offsets(rec, index, offsets);
  return (offsets);
}
```

一个细节：**COMPACT 记录的行头里根本没有"字段数"**——普通叶记录恒等于 index 的字段数，node_ptr 恒等于"非叶唯一键数 + 1"。字段数从字典拿、不占行空间，这是 COMPACT 比 REDUNDANT 省空间的又一来源。但 instant 之后这条"恒等式"被打破（见后）。

**解析核心** `rec_init_offsets_comp_ordinary`：先按 instant 状态定位 nulls/lens 的起点，再逐字段解长度。第一步是**状态判定**：

```cpp
const byte *nulls = nullptr;
const byte *lens = nullptr;
uint16_t n_null = 0;
uint8_t row_version = UINT8_UNDEFINED;
uint16_t non_default_fields = 0;
enum REC_INSERT_STATE rec_insert_state;

/* COMPACT 常规记录：nulls 紧贴记录头之上 */
*nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);
rec_insert_state = get_rec_insert_state(index, rec, false);

switch (rec_insert_state) {
  case INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION:
    *n_null = index->n_nullable;                 // 无 instant：全字段可空计数
    break;
  case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION:
  case INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION: {
    row_version = (uint8_t)(**nulls);            // 读行版本字节
    *nulls -= 1;                                 // nulls 越过版本字节
    *n_null = index->get_nullable_in_version(row_version);  // 该版本的 NULL 位图宽度
  } break;
  case INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION: {
    non_default_fields = rec_get_n_fields_instant(rec, REC_N_NEW_EXTRA_BYTES, &length);
    *nulls -= length;                            // 越过 V1 的"字段数"1/2 字节
    *n_null = index->calculate_n_instant_nullable(non_default_fields);
  } break;
  case INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION:
    *n_null = index->get_nullable_before_instant_add_drop();
    non_default_fields = index->get_instant_fields();
    break;
  case INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION:
    *n_null = index->get_nullable_before_instant_add_drop();
    break;
}

*lens = *nulls - UT_BITS_IN_BYTES(*n_null);      // 变长长度数组起点
```

要点：

- **NULL 位图宽度随行版本走**：instant ADD 一列后新行的 NULL 位图多 1 位，老行少 1 位——`index->get_nullable_in_version(row_version)` 按版本查字典维护的 `nullables[MAX_ROW_VERSION+1]` 表。这是"行是半描述"的最直接体现：解析行需要字典里的版本化元数据。
- V1（8.0.28 及以前）的 instant 行不存版本号，改存"行里实际字段数"（`rec_get_n_fields_instant`：首字节最高位 0 → 1 字节字段数，1 → 2 字节）；V2 存版本号。两代格式在同一棵 B+ 树里共存，解析器必须同时认识。

**逐字段解析**（第二个循环）：

```cpp
ulint offs = 0;
ulint any_ext = 0;
ulint null_mask = 1;
uint16_t i = 0;
do {
  const dict_field_t *field = index->get_physical_field(i);   // ★ 物理字段序
  const dict_col_t *col = field->col;
  uint64_t len;

  /* instant 状态机：行里没有的字段不读数据，只记标志 */
  switch (rec_insert_state) {
    case INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION: ...
      [[fallthrough]];
    case INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION:
    case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION: {
      if (col->is_dropped_in_or_before(row_version)) {
        len = offs | REC_OFFS_DROP;              // 该版本已删 → 无数据
        goto resolved;
      } else if (col->is_added_after(row_version)) {
        len = rec_get_instant_offset(index, i, offs);  // 行里没有 → DEFAULT/NULL 标志
        goto resolved;
      }
    } break;

    case INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION:
    case INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION: {
      if (i >= non_default_fields) {             // V1：超出实际字段数 → 默认值
        len = rec_get_instant_offset(index, i, offs);
        goto resolved;
      }
    } break;
    ...
  }

  if (!(col->prtype & DATA_NOT_NULL)) {
    /* 可空字段：读 NULL 位（nulls 从高地址向低地址走，null_mask 循环） */
    ut_ad(n_null--);
    if (UNIV_UNLIKELY(!(byte)null_mask)) {
      nulls--;
      null_mask = 1;
    }
    if (*nulls & null_mask) {
      null_mask <<= 1;
      /* NULL 不占空间：offs 不动，只记标志 */
      len = offs | REC_OFFS_SQL_NULL;
      goto resolved;
    }
    null_mask <<= 1;
  }

  if (!field->fixed_len || (temp && !col->get_fixed_size(temp))) {
    /* 变长字段：读长度字节（lens 指针从高地址向低地址递减） */
    len = *lens--;
    if (DATA_BIG_COL(col)) {
      if (len & 0x80) {
        /* 1exxxxxxx xxxxxxxx 两字节编码 */
        len <<= 8;
        len |= *lens--;
        offs += len & 0x3fff;
        if (UNIV_UNLIKELY(len & 0x4000)) {       // bit14 = 外置标志
          ut_ad(index->is_clustered());
          any_ext = REC_OFFS_EXTERNAL;
          len = offs | REC_OFFS_EXTERNAL;
        } else {
          len = offs;
        }
        goto resolved;
      }
    }
    len = offs += len;
  } else {
    len = offs += field->fixed_len;              // 定长：零开销，按字典长度累加
  }
resolved:
  rec_offs_base(offsets)[i + 1] = len;           // 写入前缀和
} while (++i < rec_offs_n_fields(offsets));

*rec_offs_base(offsets) = (rec - (lens + 1)) | REC_OFFS_COMPACT | any_ext;
```

三个编码规则值得背下来：

1. **变长长度编码**：最大长度 ≤ 255 的列永远 1 字节；`DATA_BIG_COL`（BLOB/TEXT 等大列）长度 < 128 用 1 字节，≥ 128 用 2 字节（首字节 `1exxxxxxx`，bit14=1 表示外置）；外置字段长度 ≥ 0xc0 且必须两字节（写侧 `*lens = (byte)(len >> 8) | 0xc0`）。
2. **NULL 与偏移的关系**：NULL 字段不推进 `offs`，其 offsets 槽位 = 前一字段的结束偏移 + SQL_NULL 标志——所以"取字段长度"必须先查标志再减偏移（`rec_get_nth_field_offs_low` 的职责）。
3. **`rec - (lens + 1)`**：解析结束时 lens 已走过所有长度字节，`lens+1` 指向长度数组最低地址字节，`rec - (lens+1)` 即头部总大小——写进 `base[0]`，供 `rec_offs_extra_size` 等直接取"数据区起点到记录头的距离"。

### REDUNDANT 的 1/2 字节偏移目录

REDUNDANT 行自带每字段 1~2 字节的"字段结束偏移"目录（`rec_get_1byte_offs_flag` 判定用 1 还是 2 字节），NULL 用偏移字节的高位（1 字节形式 0x80、2 字节形式 0x8000）表示，外置用 2 字节形式的 0x4000：

```cpp
constexpr uint32_t REC_1BYTE_SQL_NULL_MASK = 0x80UL;
constexpr uint32_t REC_2BYTE_SQL_NULL_MASK = 0x8000UL;
constexpr uint32_t REC_2BYTE_EXTERN_MASK = 0x4000UL;
```

解析（`rec_init_offset_old_2byte` 的字段循环）把行内目录逐项读进 offsets：

```cpp
offs = rec_2_get_field_end_info_low(rec, i);   // 行内 2 字节目录项
if (offs & REC_2BYTE_SQL_NULL_MASK) {
  offs &= ~REC_2BYTE_SQL_NULL_MASK;
  offs |= REC_OFFS_SQL_NULL;                   // 行内标志 → offsets 标志的翻译
}
if (offs & REC_2BYTE_EXTERN_MASK) {
  offs &= ~REC_2BYTE_EXTERN_MASK;
  offs |= REC_OFFS_EXTERNAL;
  *rec_offs_base(offsets) |= REC_OFFS_EXTERNAL;
}
rec_offs_base(offsets)[1 + i] = offs;
```

注意这套翻译的意义：**行内目录的编码与 offsets 的编码是两套协议**，REDUNDANT 解析器把它们归一化到 offsets 协议，上层消费方因此完全感知不到格式差异——这是 offsets 数组作为"统一坐标系统"的又一个证据。

还有一条反向解析链 `rec_get_offsets_reverse`：输入 extra 字节区（逆序），从 index 出发重新算出 offsets——用于**页压缩**（COMPRESSED 格式解压后校验）与恢复路径，同样印证"offsets 可从 index 字典重建"的设计。

### 字段定位：rec_get_nth_field_offs_low

消费方取字段的最终出口：

```cpp
static inline ulint rec_get_nth_field_offs_low(const ulint *offsets, ulint n,
                                               ulint *len) {
  ulint offs;
  ulint length;

  if (n == 0) {
    offs = 0;
  } else {
    offs = rec_offs_base(offsets)[n] & REC_OFFS_MASK;   // 前字段结束偏移 = 本字段起始
  }

  length = rec_offs_base(offsets)[1 + n];               // 本字段结束偏移（可能带标志）

  if (length & REC_OFFS_SQL_NULL) {
    length = UNIV_SQL_NULL;
  } else if (length & REC_OFFS_DEFAULT) {
    length = UNIV_SQL_ADD_COL_DEFAULT;                  // instant：去 DD 取默认值
  } else if (length & REC_OFFS_DROP) {
    length = UNIV_SQL_INSTANT_DROP_COL;                 // instant：已删列
  } else {
    length &= REC_OFFS_MASK;
    length -= offs;                                     // 结束 - 起始 = 长度
  }

  *len = length;
  return (offs);
}
```

这段 20 行函数浓缩了 offsets 协议的**全部语义**：前缀和、四标志、三种特殊长度（`UNIV_SQL_NULL` / `UNIV_SQL_ADD_COL_DEFAULT` / `UNIV_SQL_INSTANT_DROP_COL`）。消费方拿到的 `len` 若是后两种，走 `rec_get_nth_default` 从 DD 的 instant default 缓存取值、或跳过该字段。

**逻辑→物理字段映射**在入口层完成（`rec_get_nth_field_offs` / `rec_offs_nth_*`）：

```cpp
ulint rec_get_nth_field_offs(const dict_index_t *index, const ulint *offsets,
                             ulint n, ulint *len) {
  if (index && index->has_row_versions()) {
    n = index->get_field_off_pos(n);      // 逻辑字段号 → offsets 槽位（考虑 DROP 列的洞）
  }
  return rec_get_nth_field_offs_low(offsets, n, len);
}
```

instant DROP 列虽然行里没数据，但 offsets 里仍为其保留槽位（`REC_OFFS_DROP` 标记），所以逻辑字段号与 offsets 槽位号之间需要 `get_field_off_pos` 换算——**双坐标体系的换算点集中在这几个入口函数**，`_low` 后缀的裸函数永远操作物理槽位。

### 写方向：rec_convert_dtuple_to_rec

编码是解析的镜像。入口按格式分发，写完后统一落 info bits 与 instant 状态：

```cpp
rec_t *rec_convert_dtuple_to_rec(byte *buf, const dict_index_t *index,
                                 const dtuple_t *dtuple) {
  rec_t *rec;
  if (dict_table_is_comp(index->table)) {
    rec = rec_convert_dtuple_to_rec_new(buf, index, dtuple);
  } else {
    rec = rec_convert_dtuple_to_rec_old(buf, index, dtuple);
  }
  ...
  return (rec);
}
```

COMPACT 编码核心（`rec_convert_dtuple_to_rec_comp`）的字段循环：

```cpp
byte *nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);   // NULL 位图起点

/* instant：决定这行带不带版本字节 / V1 字段数字节 */
if (index->has_instant_cols_or_row_versions()) {
  if (is_store_version(index, n_fields)) {
    rec_set_instant_row_version_new(rec, index->table->current_row_version);
    nulls -= 1;                                    // 版本字节挤占一格
    rec_instant_info = Rec_instant_state::REC_IS_VERSIONED;
  } else if (index->is_tuple_instant_format(n_fields)) {
    uint32_t n_fields_len = rec_set_n_fields(rec, n_fields);  // V1：写字段数
    nulls -= n_fields_len;
    rec_instant_info = Rec_instant_state::REC_IS_INSTANT;
  }
}
...
byte *end = rec;                                   // 数据写入游标（从 rec 向高地址）
byte *lens = nulls - UT_BITS_IN_BYTES(n_null);     // 变长长度写入游标（向低地址）

ulint null_mask = 1;
auto store_field = [&](const dfield_t *field, uint32_t pos) {
  const dtype_t *type = dfield_get_type(field);
  uint32_t len = dfield_get_len(field);

  if (!(dtype_get_prtype(type) & DATA_NOT_NULL)) {
    if (!(byte)null_mask) { nulls--; null_mask = 1; }     // 位图跨字节
    if (dfield_is_null(field)) {
      *nulls |= null_mask;                                // 置 NULL 位
      null_mask <<= 1;
      return;                                             // NULL：不写数据、不写长度
    }
    null_mask <<= 1;
  }

  const dict_field_t *ifield = index->get_physical_field(pos);
  uint32_t fixed_len = ifield->fixed_len;
  dict_col_t *col = ifield->col;

  if (fixed_len) {
    ut_ad(len <= fixed_len);                              // 定长：不写长度字节
  } else if (dfield_is_ext(field)) {
    /* 外置：两字节长度，高字节带 0xc0 */
    ut_ad(DATA_BIG_COL(col));
    *lens = (byte)(len >> 8) | 0xc0;
    lens--;
    *lens = (byte)len;
    lens--;
  } else {
    if (len < 128 || !DATA_BIG_LEN_MTYPE(dtype_get_len(type), dtype_get_mtype(type))) {
      *lens = (byte)len;                                  // 短值：1 字节
      lens--;
    } else {
      ut_ad(len < 16384);
      *lens = (byte)(len >> 8) | 0x80;                    // 长值：2 字节 0x80 前缀
      lens--;
      *lens = (byte)len;
      lens--;
    }
  }

  if (len > 0) memcpy(end, dfield_get_data(field), len);  // 数据写到高地址侧
  end += len;
};
```

对称性一览：

- **nulls 向低地址走、数据向高地址走、lens 向低地址走**——三者从记录头两侧对进，整行在页内连续且紧凑；长度数组与 NULL 位图不保存"方向"信息，解析器约定俗成地从同一端开始解。
- 定长字段**完全零开销**（长度从字典 `fixed_len` 拿，行里一个字节都不写）；NULL 字段零开销；短变长 1 字节开销。
- CHAR 列的尾部空格已在 server 层剥掉（`row_mysql_store_col_in_innobase_format`），行里存的是剥空格后的值——读回时按 `mbminlen/mbmaxlen` 补空格（在 `row_sel_field_store_in_mysql_format` 完成，非本篇范围）。

写完之后，`rec_convert_dtuple_to_rec_new` 按 `rec_instant_info` 置 info bits：`REC_IS_VERSIONED` → 置 `REC_INFO_VERSION_FLAG`，`REC_IS_INSTANT` → 置 `REC_INFO_INSTANT_FLAG`，且断言二者不可同时置位（`ut_a(!(rec_get_instant_flag_new(rec) && rec_new_is_versioned(rec)))`）。

### 隐藏系统列与 DB_ROW_ID

#### 三个系统列恒定存在

`dict_table_add_system_columns`（`dict0dict.cc:1131`）在**每张表**的 `dict_table_t` 上追加三个系统列（顺序由数值常量固定）：

```cpp
static_assert(DATA_ROW_ID == 0);        // dict0dict.cc:129
static_assert(DATA_TRX_ID == 1);
static_assert(DATA_ROLL_PTR == 2);
static_assert(DATA_N_SYS_COLS == 3);
static_assert(DATA_TRX_ID_LEN == 6);
```

| 列 | 长度 | 作用 |
|---|---|---|
| `DB_ROW_ID` | 6B | 无主键时的聚簇键（见下） |
| `DB_TRX_ID` | 6B | 最后修改该行的事务 ID（MVCC） |
| `DB_ROLL_PTR` | 7B | 指向 undo 记录（MVCC 历史版本） |

注意：**intrinsic 临时表没有 `DB_ROLL_PTR`**（它关闭了 undo 日志），所以插入时的系统列缓冲区只分配 `ROW_ID + TRX_ID` 两段（`row0ins.cc:153`）。

**★ `dict_table_add_system_columns()` 完整源码**（`dict0dict.cc:1131`）：

```cpp
  /* NOTE: the system columns MUST be added in the following order
  (so that they can be indexed by the numerical value of DATA_ROW_ID,
  etc.) and as the last columns of the table memory object.
  The clustered index will not always physically contain all system
  columns.
  Intrinsic table don't need DB_ROLL_PTR as UNDO logging is turned off
  for these tables. */

  const uint32_t phy_pos = UINT32_UNDEFINED;
  const uint8_t v_added = 0;
  const uint8_t v_dropped = 0;

  dict_mem_table_add_col(table, heap, "DB_ROW_ID", DATA_SYS,
                         DATA_ROW_ID | DATA_NOT_NULL, DATA_ROW_ID_LEN, false,
                         phy_pos, v_added, v_dropped);      // ★ prtype = 0 | 256

  dict_mem_table_add_col(table, heap, "DB_TRX_ID", DATA_SYS,
                         DATA_TRX_ID | DATA_NOT_NULL, DATA_TRX_ID_LEN, false,
                         phy_pos, v_added, v_dropped);      // ★ prtype = 1 | 256

  if (!table->is_intrinsic()) {
    dict_mem_table_add_col(table, heap, "DB_ROLL_PTR", DATA_SYS,
                           DATA_ROLL_PTR | DATA_NOT_NULL, DATA_ROLL_PTR_LEN,
                           false, phy_pos, v_added, v_dropped);  // ★ prtype = 2 | 256
    /* This check reminds that if a new system column is added to
    the program, it should be dealt with here */
  }
```

```cpp
constexpr uint32_t DATA_ROW_ID = 0;              constexpr uint32_t DATA_ROW_ID_LEN   = 6;
constexpr size_t   DATA_TRX_ID = 1;              constexpr size_t   DATA_TRX_ID_LEN   = 6;
constexpr size_t   DATA_ROLL_PTR = 2;            constexpr size_t   DATA_ROLL_PTR_LEN = 7;
constexpr uint32_t DATA_N_SYS_COLS = 3;          // 普通表
constexpr uint32_t DATA_ITT_N_SYS_COLS = 2;      // intrinsic 临时表：只有 ROW_ID + TRX_ID
constexpr uint32_t DATA_FTS_DOC_ID = 3;          // 不计入系统列
constexpr uint32_t DATA_SYS_PRTYPE_MASK = 0xF;   // 从 prtype 低 4 位取系统列编号
```

> ★★ 三个系统列的 `mtype` 恒为 `DATA_SYS`，`prtype` 低 4 位就是 0/1/2，且必带 `DATA_NOT_NULL`。系统列永远是 table 的**最后 N 列**——换算公式 `n_cols - get_n_sys_cols() + sys`（见 `dict_table_get_sys_col_no()`）。
> ⚠️ **不存在 `dict_index_get_sys_col_pos()` 自由函数**（0 命中）——真名是成员函数 `dict_index_t::get_sys_col_pos()`（`dict0mem.cc:716`）。**二级索引调用它返回 `ULINT_UNDEFINED`**（二级索引不含系统列）。

**★ 系统列进 `index->fields` 的落点是 `dict_index_build_internal_clust()`**（`dict0dict.cc:2960`，不是 `dict_index_add_to_cache`）：

```cpp
  /* Add system columns */
  size_t trx_id_pos = new_index->n_def;

  /* Add ROW ID */
  if (!dict_index_is_unique(index)) {
    dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROW_ID), 0, true);
    set_phy_pos(table->get_sys_col(DATA_ROW_ID));
    trx_id_pos++;
  }                                              // ★★ 只有"聚簇索引非唯一"时 DB_ROW_ID 才进记录

  /* Add TRX ID */
  dict_index_add_col(new_index, table, table->get_sys_col(DATA_TRX_ID), 0, true);
  set_phy_pos(table->get_sys_col(DATA_TRX_ID));  // ★ DB_TRX_ID 无条件进聚簇索引

  for (size_t i = 0; i < trx_id_pos; i++) {
    ulint fixed_size = new_index->get_col(i)->get_fixed_size(dict_table_is_comp(table));
    if (fixed_size == 0) { new_index->trx_id_offset = 0; break; }  // ★ 有变长列 → 放弃快路径
    dict_field_t *field = new_index->get_field(i);
    if (field->prefix_len > 0) { new_index->trx_id_offset = 0; break; }  // ★ 有前缀列 → 同样
    fixed_size += new_index->trx_id_offset;
    new_index->trx_id_offset = fixed_size;       // ★★ 累加得到 DB_TRX_ID 的字节偏移
    ...
  }

  /* Add ROLL PTR ... */
  if (!table->is_intrinsic()) {
    dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROLL_PTR), 0, true);
    set_phy_pos(table->get_sys_col(DATA_ROLL_PTR));
  }
```

> ★★ **`trx_id_offset` 是"读 DB_TRX_ID 不用算 offsets"的快路径**——只在前序列全定长且无前缀索引时有效，否则为 0，回退到 `rec_offs` 计算。这正是 `row_get_rec_trx_id()` 里那个 `if (!offset)` 分支的由来：

```cpp
static inline trx_id_t row_get_rec_trx_id(const rec_t *rec, const dict_index_t *index,
                                          const ulint *offsets) {
  ulint offset = index->trx_id_offset;           // ★ 快路径
  if (!offset) {
    offset = row_get_trx_id_offset(index, offsets);   // ★ 慢路径：靠 offsets 换算
  }
  return (trx_read_trx_id(rec + offset));
}

static inline roll_ptr_t row_get_rec_roll_ptr(const rec_t *rec, const dict_index_t *index,
                                              const ulint *offsets) {
  ulint offset = index->trx_id_offset;
  if (!offset) offset = row_get_trx_id_offset(index, offsets);
  return (trx_read_roll_ptr(rec + offset + DATA_TRX_ID_LEN));  // ★★ ROLL_PTR 紧跟 TRX_ID 之后
}
```

> ⚠️ **不存在 `row_get_rec_row_id()`**——要读记录里的 DB_ROW_ID 只能 `rec_get_nth_field(rec, offsets, index->get_sys_col_pos(DATA_ROW_ID), &len)` + `dict_sys_read_row_id(ptr)`。
> ⚠️ **`trx_write_trx_id()` / `trx_read_trx_id()` 在 `include/trx0sys.ic`，不在 `trx0sys.cc`**；`trx_write_roll_ptr()` / `trx_read_roll_ptr()` 在 **`include/trx0undo.ic`**。

**★ node pointer 里只有 `n_uniq` 个前缀字段，不含 DB_TRX_ID / DB_ROLL_PTR**：

```cpp
  n_unique = dict_index_get_n_unique_in_tree_nonleaf(index);
  tuple = dtuple_create(heap, n_unique + 1);     // ★ 多出 1 个放 child page no
  dtuple_set_n_fields_cmp(tuple, n_unique);      // ★ 比较时排除页号字段
  field = dtuple_get_nth_field(tuple, n_unique);
  dfield_set_data(field, buf, 4);
  dtype_set(dfield_get_type(field), DATA_SYS_CHILD, DATA_NOT_NULL, 4);  // ★ 第 4 种 system 类型
  rec_copy_prefix_to_dtuple(tuple, rec, index, n_unique, heap);         // ★ 只拷前 n_unique 个
```

> ★★ **MVCC 两列只存在于叶子层**——非叶层的 node pointer 永远不含 `DB_TRX_ID` / `DB_ROLL_PTR`（无唯一主键时含 `DB_ROW_ID`，因为它属于 `n_uniq`）。

#### row_id 什么时候被分配？

三个系统列在**定义上恒存在**，但 `DB_ROW_ID` **只有在被用作聚簇键时才分配值并写入行**。聚簇索引键的选择顺序：

1. 有显式 `PRIMARY KEY` → 用 PK；
2. 无 PK → 选第一个**全部列非空**的 UNIQUE 索引；
3. 都没有 → **用 `DB_ROW_ID`**：

```cpp
/* dict0dict.cc:3046 —— 聚簇索引无用户列可用时的分支 */
dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROW_ID), 0, true);
set_phy_pos(table->get_sys_col(DATA_ROW_ID));
```

这就是"**什么时候会自动添加 row id**"的答案：**表既无主键也无全非空唯一键时**，InnoDB 隐式生成 6 字节 row_id 作聚簇键，插入时由 `row_ins_alloc_row_id_step`（`row0ins.cc:3475`）分配：

```cpp
row_id = dict_sys_get_new_row_id();
dict_sys_write_row_id(node->row_id_buf, row_id);
```

#### ★ row_id 的持久化：每 256 次才落盘

row_id 来自全局计数器 `dict_sys->row_id`，但**不是每次分配都写盘**（`dict0boot.ic:36`）：

```cpp
static inline row_id_t dict_sys_get_new_row_id(void) {
  dict_sys_mutex_enter();
  id = dict_sys->row_id;
  if (0 == (id % DICT_HDR_ROW_ID_WRITE_MARGIN))     // MARGIN = 256
    dict_hdr_flush_row_id();                          // ★ 每 256 个才写 DICT_HDR 页
  dict_sys->row_id++;
  dict_sys_mutex_exit();
  return (id);
}
```

`DICT_HDR_ROW_ID_WRITE_MARGIN = 256`（`dict0boot.h:334`）——崩溃时最多丢失 255 个"已分配但未落盘"的 row_id。

**启动时补偿**（`dict0boot.cc:228`）：

```cpp
dict_sys->row_id = DICT_HDR_ROW_ID_WRITE_MARGIN +
    ut_uint64_align_up(mach_read_from_8(dict_hdr + DICT_HDR_ROW_ID),
                       DICT_HDR_ROW_ID_WRITE_MARGIN);
```

★ **读回值向上对齐到 256 的倍数、再加 256**——把"可能丢掉的那 255 个"一次性跳过，保证重启后绝不会重用已发出的 row_id。这是典型的"**用跳跃换取免持久化**"设计（与自增计数器"每次写 redo"的取向正好相反，因为 row_id 只要不重复即可，允许空洞）。

`IMPORT TABLESPACE` 时还要额外校正（`row0import.cc:2942` `row_import_set_sys_max_row_id`）：扫描 `SELECT MAX(DB_ROW_ID)`，若 `row_id >= dict_sys->row_id` 则推进并 `dict_hdr_flush_row_id()`——否则导入表的 row_id 可能与现有行冲突。

**★ 完整的分配/落盘/恢复三件套**（⚠️ 三处真名都与常见写法不同）：

```cpp
/** Returns a new row id. —— include/dict0boot.ic:36（不在 dict0dict.cc！） */
static inline row_id_t dict_sys_get_new_row_id(void) {
  row_id_t id;
  dict_sys_mutex_enter();                        // ★ 全局 dict_sys->mutex，唯一串行化点（无原子变量）
  id = dict_sys->row_id;                         // ★ 先取当前值
  if (0 == (id % DICT_HDR_ROW_ID_WRITE_MARGIN)) {
    dict_hdr_flush_row_id();                     // ★ 只在 id % 256 == 0 时落盘
  }
  dict_sys->row_id++;                            // ★ 先返回旧值再自增
  dict_sys_mutex_exit();
  return (id);
}
```

```cpp
/** Writes the current value of the row id counter to the dictionary header
 file page. —— dict/dict0boot.cc:151（真名，不是 dict_write_row_id） */
void dict_hdr_flush_row_id(void) {
  ut_ad(dict_sys_mutex_own());                   // ★ 调用者必须已持锁
  id = dict_sys->row_id;
  mtr_start(&mtr);                               // ★ 独立 mtr，与业务 mtr 无关
  dict_hdr = dict_hdr_get(&mtr);                 // ★ x-latch 住 DICT_HDR_PAGE_NO（不是 S 锁）
  mlog_write_ull(dict_hdr + DICT_HDR_ROW_ID, id, &mtr);
  mtr_commit(&mtr);
}
```

```cpp
/** dict/dict0boot.cc:228 —— 启动时补偿 */
  /* Because we only write new row ids to disk-based data structure
  (dictionary header) when it is divisible by
  DICT_HDR_ROW_ID_WRITE_MARGIN, in recovery we will not recover
  the latest value of the row id counter. Therefore we advance
  the counter at the database startup to avoid overlapping values.
  Note that when a user after database startup first time asks for
  a new row id, then because the counter is now divisible by
  ..._MARGIN, it will immediately be updated to the disk-based
  header. */
  dict_sys->row_id =
      DICT_HDR_ROW_ID_WRITE_MARGIN +
      ut_uint64_align_up(mach_read_from_8(dict_hdr + DICT_HDR_ROW_ID),
                         DICT_HDR_ROW_ID_WRITE_MARGIN);
```

```cpp
/* When a row id which is zero modulo this number (which must be a power of
two) is assigned, the field DICT_HDR_ROW_ID on the dictionary header page is
updated */
constexpr uint32_t DICT_HDR_ROW_ID_WRITE_MARGIN = 256;   // dict0boot.h:334
```

> ★★ 设磁盘上值为 `V`，启动时内存计数器 = `align_up(V, 256) + 256`。这样既跳过"可能已发出但未落盘"的余数部分，又保证启动后立刻是 256 的倍数（第一次分配就触发落盘）。
> ★ 建库时 `DICT_HDR_ROW_ID` 初值写 0（`dict_hdr_create`），配合上面的 +256，**首次实际分配到的 row_id = 256**。
> ⚠️ **`dict_sys_t::row_id` 定义在 `include/dict0dict.h:1017`，不在 `dict0mem.h`**；`dict0mem.h` 里的 `sess_row_id` 是 intrinsic 表专用、完全另一回事。

**★ 分配点有两个，取决于表类型**：

```cpp
  // 普通表：row0ins.cc:3460
static inline void row_ins_alloc_row_id_step(ins_node_t *node) {
  if (dict_index_is_unique(node->table->first_index())) {
    /* No row id is stored if the clustered index is unique */
    return;                                  // ★★ 有显式主键 → 根本不分配，也不写入记录
  }
  row_id = dict_sys_get_new_row_id();
  dict_sys_write_row_id(node->row_id_buf, row_id);
}
```

```cpp
  // intrinsic 临时表：row0mysql.cc:1451 —— 完全绕过全局计数器
  if (!dict_index_is_unique(clust_index)) {
    dict_sys_write_row_id(node->row_id_buf,
                          dict_table_get_next_table_sess_row_id(node->table));
    //  ★ 用 table->sess_row_id 本地计数器，零 mutex 开销
  }
  trx_write_trx_id(node->trx_id_buf,
                   dict_table_get_next_table_sess_trx_id(node->table));
```

```cpp
static inline row_id_t dict_table_get_next_table_sess_row_id(dict_table_t *table) {
  return (++table->sess_row_id);     // ★ 先 ++ 再返回 → intrinsic 表第一个 row_id 是 1 不是 0
}
```

```cpp
dberr_t row_insert_for_mysql(const byte *mysql_rec, row_prebuilt_t *prebuilt) {
  if (prebuilt->table->is_intrinsic()) {
    return (row_insert_for_mysql_using_cursor(mysql_rec, prebuilt));   // ★ 本地 sess_row_id
  } else {
    return (row_insert_for_mysql_using_ins_graph(mysql_rec, prebuilt)); // ★ 全局 dict_sys->row_id
  }
}
```

> ★ intrinsic 表的 UPDATE 还会**重写 row_id**（`row0mysql.cc:2119`）：`dtuple_get_nth_field(node->upd_row, table->get_n_cols() - 2)` —— 倒数第 2 列就是 DB_ROW_ID（intrinsic 只有 2 个系统列）。

**★ 6/7 字节大端读写**（`include/mach0data.ic`，⚠️ **不在 `mach0data.cc`**）：

```cpp
static inline void mach_write_to_7(byte *b, uint64_t n) {
  mach_write_to_3(b, (ulint)(n >> 32));      // ★ 高 3 字节 = n[47:24]
  mach_write_to_4(b + 3, (ulint)n);          // ★ 低 4 字节 = n[23:0]
}
static inline uint64_t mach_read_from_7(const byte *b) {
  return (ut_ull_create(mach_read_from_3(b), mach_read_from_4(b + 3)));
}
static inline void mach_write_to_6(byte *b, uint64_t n) {
  mach_write_to_2(b, (ulint)(n >> 32));      // ★ 高 2 字节 = n[47:32]
  mach_write_to_4(b + 2, (ulint)n);          // ★ 低 4 字节 = n[31:0]
}
static inline uint64_t mach_read_from_6(const byte *b) {
  return (ut_ull_create(mach_read_from_2(b), mach_read_from_4(b + 2)));
}
```

> ★ `row_id_t` 是 64 位，但磁盘/记录里只存 **6 字节（48 位）**。

#### DICT_HDR 页的头部布局

系统表空间页 7 是字典头页（`dict0boot.h:113`）：

| 偏移 | 字段 | 说明 |
|---|---|---|
| 0 | `DICT_HDR_ROW_ID` | 8B，最近分配的 row_id（每 256 更新） |
| 8 | `DICT_HDR_TABLE_ID` | 8B，最近分配的 table id（初值 `DICT_MAX_DD_TABLES`） |
| 16 | `DICT_HDR_INDEX_ID` | 8B，最近分配的 index id |
| 24 | `DICT_HDR_MAX_SPACE_ID` | 4B，最近分配的 space id |
| 28 | `DICT_HDR_MIX_ID_LOW` | 4B（已废弃，仍需初始化） |
| 32 | `DICT_HDR_TABLES` | 4B，SYS_TABLES 聚簇索引 root 页号 |
| 36 | `DICT_HDR_TABLE_IDS` | 4B，SYS_TABLE_IDS 二级索引 root 页号 |

三个 ID 计数器与 row_id 同构：**都是"内存计数器 + 定期刷 DICT_HDR"**，靠启动时向上对齐避免重用。

> **handler 侧的对应物**：`row_prebuilt_t` 里有 `clust_index_was_generated`（本表是否用了 row_id 作聚簇键）与 `byte row_id[DATA_ROW_ID_LEN]`（最后取到的行的 row_id 缓冲）——本篇讲 row_id 如何生成与持久化，那两个字段讲它如何被缓存与回传，见 [`../../server/handler.md`](../../server/handler.md) 的「row_prebuilt_t」一节。

### 固定 offsets 缓存与 Rec_offsets

**★ `Rec_offsets` 是 RAII 包装，内部固定 100 元素的栈缓冲**（⚠️ 不是"语法糖"）：

```cpp
#define rec_offs_init(offsets) \
  rec_offs_set_n_alloc(offsets, (sizeof offsets) / sizeof *offsets)
```

```cpp
class Rec_offsets : private ut::Non_copyable {
 public:
  Rec_offsets() { rec_offs_init(m_preallocated_buffer); }   // ★ 只写一次 offsets[0] = 100

  const ulint *compute(const rec_t *rec, const dict_index_t *index,
                       const ulint n_fields = ULINT_UNDEFINED) {
    m_offsets = rec_get_offsets(rec, index, m_offsets, n_fields,
                                UT_LOCATION_HERE, &m_heap);
    //  ★★ 把上次返回的指针回传 → 复用缓冲区（可能是 heap 上那块）
    return m_offsets;
  }

  ~Rec_offsets() {
    if (m_heap) { mem_heap_free(m_heap); m_heap = nullptr; }
  }

 private:
  mem_heap_t *m_heap{nullptr};                        // ★ 只有超大记录才非空
  ulint m_preallocated_buffer[REC_OFFS_NORMAL_SIZE];  // ★ 100 * 8 = 800 字节栈缓冲
  ulint *m_offsets{m_preallocated_buffer};
};
```

> ⚠️ **`rec_offs_init()` 是宏不是函数**，且用 `sizeof offsets`——**对指针参数会静默算错**（`sizeof(ulint*) = 8` → `n_alloc = 1`），随后 `ut_ad(n_alloc > REC_OFFS_HEADER_SIZE)` 立刻炸（debug）或静默越界（release）。**必须**配 `ulint offsets_[REC_OFFS_NORMAL_SIZE];` 这类真数组使用。

**★★ `dict_index_t::rec_cache` 是"索引级"缓存，不是页级**（这是 offsets 体系的第三个性能层次）：

```cpp
struct rec_cache_t {
  /** Holds reference to cached offsets for record. */
  const ulint *offsets{nullptr};      // ★ const：一经建立不得修改
  /** Number of NULLable columns among those for which offsets are cached */
  size_t nullable_cols{0};            // ★ 用于快速判断"这条记录是否有 NULL → 缓存是否适用"
};
```

```cpp
static void dict_index_try_cache_rec_offsets(dict_index_t *index) {
  const auto n_unique_in_tree = dict_index_get_n_unique_in_tree(index);
  ...
  for (size_t i = 0; i < n_unique_in_tree; i++) {
    if (!index->get_field(i)->fixed_len) {
      return;                         // ★★ 只要有一个变长列 → 放弃缓存
    }
  }

  const auto offsets_len = n_unique_in_tree + (1 + REC_OFFS_HEADER_SIZE);
  auto *const offsets = static_cast<ulint *>(
      mem_heap_alloc(index->heap, sizeof(ulint) * offsets_len));

  index->rec_cache.offsets = offsets;
  rec_offs_set_n_alloc(offsets, offsets_len);
  rec_offs_set_n_fields(offsets, n_unique_in_tree);
  rec_init_fixed_offsets(index, offsets);   // ★★ 只靠 index 的 fixed_len 推算，完全不看 rec

  for (size_t i = 0; i < n_unique_in_tree; i++) {
    if (!(index->get_field(i)->col->prtype & DATA_NOT_NULL)) {
      index->rec_cache.nullable_cols++;
    }
  }
}
```

```cpp
inline void rec_init_fixed_offsets(const dict_index_t *index, ulint *offsets) {
  ut_ad(!index->has_instant_cols_or_row_versions());
  rec_offs_make_valid(nullptr, index, offsets);   // ★★ rec 传 nullptr → offsets[2]=0，标记"共享缓存"
  rec_offs_base(offsets)[0] =
      (REC_N_NEW_EXTRA_BYTES + UT_BITS_IN_BYTES(index->n_nullable)) | REC_OFFS_COMPACT;
  const auto n_fields = rec_offs_n_fields(offsets);
  auto field_end = rec_offs_base(offsets) + 1;
  for (size_t i = 0; i < n_fields; i++) {
    field_end[i] = (i ? field_end[i - 1] : 0) + index->get_field(i)->fixed_len;
  }
}
```

**启用条件（`dict0dict.cc:2567`）**——四个条件全满足才建缓存：

```cpp
  if (dict_table_is_comp(table) && !dict_index_has_virtual(index) &&
      (!table->has_instant_cols() && !table->has_row_versions()) &&
      !dict_index_is_spatial(index)) {
    dict_index_try_cache_rec_offsets(new_index);
```

**★★ 消费点：二分查找直接绕过 `rec_get_offsets()`**（`page0cur.cc:461`）：

```cpp
  auto get_mid_rec_offsets = [&]() -> const auto * {
    if (cached_offsets && !page_cur_has_null(mid_rec, index)) {
      //  ★★ 缓存存在 + 这条记录没有 NULL → 直接用缓存，跳过 rec_get_offsets()
#ifdef UNIV_DEBUG
      { ... 与真算的 offsets 逐字段比对 ut_a(off == off2); ut_a(len == len2); }
#endif
      return cached_offsets;
    }
    ...
  };
```

```cpp
static bool page_cur_has_null(const rec_t *rec, const dict_index_t *index) {
  const auto nullable_cols = index->rec_cache.nullable_cols;
  if (!nullable_cols) return false;
  const byte *nulls = rec - (1 + REC_N_NEW_EXTRA_BYTES);
  size_t n_bytes_to_scan = UT_BITS_IN_BYTES(nullable_cols);
  byte null_mask = 0xff;
  size_t bits_examined = 0;

  for (size_t i = 0; i < n_bytes_to_scan - 1; i++) {
    if (*nulls & null_mask) return true;
    --nulls;
    bits_examined += 8;
  }
  null_mask >>= (8 - (nullable_cols - bits_examined));
  return (*nulls & null_mask);
}
```

> ★★ 三个层次合起来看：**栈缓冲**（`Rec_offsets`，避免 malloc）→ **数组复用**（`rec_get_offsets` 传回旧指针，避免重分配）→ **共享缓存**（`rec_cache`，连解析都省掉，前提是定长且无 NULL）。

两条性能补充：

**`rec_init_fixed_offsets` + `index->rec_cache`**：对**全定长且无 NULL 且无 instant** 的索引，字段偏移与行内容无关，可直接从字典算出并缓存到 `index->rec_cache.offsets`，无数条记录共享一份：

```cpp
inline void rec_init_fixed_offsets(const dict_index_t *index, ulint *offsets) {
  ut_ad(!index->has_instant_cols_or_row_versions());
  rec_offs_make_valid(nullptr, index, offsets);          // 缓存态：rec=nullptr
  rec_offs_base(offsets)[0] =
      (REC_N_NEW_EXTRA_BYTES + UT_BITS_IN_BYTES(index->n_nullable)) |
      REC_OFFS_COMPACT;
  const auto n_fields = rec_offs_n_fields(offsets);
  auto field_end = rec_offs_base(offsets) + 1;
  for (size_t i = 0; i < n_fields; i++) {
    field_end[i] = (i ? field_end[i - 1] : 0) + index->get_field(i)->fixed_len;
  }
}
```

DEBUG 校验里 `rec_offs_validate` 对 `offsets[2] == nullptr`（缓存态）特判放行——"缓存偏移"与"行特定偏移"靠 `offsets[2]` 区分。

**`Rec_offsets` RAII 助手**（`rem0rec.h`）：把"栈数组 + 惰性 heap"的样板封装成对象，`compute(rec, index)` 复用上次分配的内存（`m_offsets` 在多次调用间传递，避免重复 malloc）。`rec_get_offsets` 自身的"复用传入数组"约定（`rec_offs_get_n_alloc(offsets) >= size` 时原位重填）是所有缓存行为的根基。

---

## Misc

### 容易误解的概念

| 概念 | 澄清 |
|---|---|
| `REC_OFFS_COMPACT` | 是"COMPACT 行格式"标志（`base[0]` 高 31 位），**与 COMPRESSED 页格式无关**。COMPRESSED 页解压后的行仍是 COMPACT 行格式 |
| `rec` 指针 | 指向**数据区起点**，不是记录最开头。头部字段一律 `rec - offs` 向下取；"记录大小"= `rec_offs_extra_size + rec_offs_data_size` |
| 变长长度数组的方向 | **逆序**存放：后字段的长度字节在低地址。解析时 lens 从高地址向低地址递减 |
| offsets 的 NULL 槽位 | NULL 字段在 offsets 里**占槽位**（偏移=前一字段末），与 REDUNDANT 行内目录的 NULL 高位编码是两套协议，靠 `rec_init_offsets_old` 翻译归一 |
| `REC_OFFS_DEFAULT` vs `DATA_MISSING` | 前者是 **offsets 协议**的 instant 缺列标志（取值去 DD 默认值）；后者是 **dtuple vfield 段**的"vcol 未计算"哨兵（见 [`../feat/generated_columns.md`](../../feat/generated_columns.md)）。两者无关 |
| `UNIV_SQL_ADD_COL_DEFAULT` / `UNIV_SQL_INSTANT_DROP_COL` | `rec_get_nth_field_offs_low` 返回的特殊长度，消费方必须分支处理，不能当字节数用 |
| `rec_offs_nth_size` | 返回**物理槽位**大小（NULL 为 0、DEFAULT/DROP 也为 0），与"逻辑字段的值大小"不是一回事 |

**★★ 真名勘误全表**：

| 常见写法 | 8.0.39 真相 |
|---|---|
| offsets 数组大小 = `n + REC_OFFS_HEADER_SIZE` | **错**。是 **`n + (1 + REC_OFFS_HEADER_SIZE)`**，多出的 `1` 是 `base[0]`（extra size 槽位） |
| `REC_OFFS_COMPACT` 与 `REC_OFFS_SQL_NULL` 是不同位 | **同一个位 `1U << 31`**！不冲突只因一个用于 `base[0]`、一个用于 `base[1+n]` |
| `REC_OFFS_HEADER_SIZE == 2` | **debug=4 / release=2**。`offsets[2]`(rec)、`offsets[3]`(index) 只在 UNIV_DEBUG 存在 |
| `REC_N_OWNED` / `REC_INFO_BITS` | **都不存在**。真名 `REC_OLD_N_OWNED`(6) / `REC_NEW_N_OWNED`(5) 与 `REC_OLD_INFO_BITS`(6) / `REC_NEW_INFO_BITS`(5) / `REC_TMP_INFO_BITS`(1) |
| 记录头常量用 `#define` | **错**。全是 `constexpr uint32_t`；且 **`REC_N_NEW_EXTRA_BYTES` 是 `constexpr int32_t`（有符号）** |
| 记录头常量在 `include/rem0rec.h` | **错**。8.0.3x 已拆分到 **`rem/rec.h`**；`rem0rec.h` 只留三个长度上限常量 |
| 位域布局 ASCII 图在 `rem0rec.h` | **错**。在 **`include/rem0rec.ic:40`**，是**两张图**（old + new） |
| `rec_get_offsets()` 是 5 参 | **6 参**，多了 `ut::Location location`，且 `[[nodiscard]]` |
| `rec_offs_set_base()` | **不存在**（0 命中）。只有读侧 `rec_offs_base()` |
| `rec_init_offsets_comp_ordinary()` 是 static | **错**。**非 static**，在 `rem/rec.h:1081` 公开声明，带 `temp` 参数 |
| `rec_get_prefix_fields()` | **不存在**。真名 `rec_copy_prefix_to_dtuple()`（rec→dtuple）+ `rec_copy_prefix_to_buf()` / `_old()`（rec→rec） |
| `rec_set_info_bits()` 单一函数 | **不存在**。真名三兄弟 `rec_set_info_bits_old` / `_new` / `_new_temp` |
| `rec_get_deleted_flag(rec, ulint comp)` | comp 类型是 **`bool`**（`rem0rec.ic:353`） |
| `rec_get_instant_row_version()`（裸名） | **不存在** → `rec_get_instant_row_version_new()` / `_old()` |
| `rec_get_converted_size()` 在 `rem0rec.cc` | **错**。在 **`rem0rec.ic:748`**；`rec_get_converted_size_comp()` 才在 `rem0rec.cc:517` |
| `offshift()` 辅助函数 | **不存在**（全 `storage/` 0 命中） |
| `dict_index_get_sys_col_pos()` 自由函数 | **不存在** → 成员函数 `dict_index_t::get_sys_col_pos()`（`dict0mem.cc:716`） |
| `dict_hdr_get_row_id()` / `dict_write_row_id()` | **都不存在** → 写盘 `dict_hdr_flush_row_id()`（`dict0boot.cc:151`）；落记录 `dict_sys_write_row_id()` |
| `row_get_rec_row_id()` | **不存在** → `rec_get_nth_field(...)` + `dict_sys_read_row_id()` |
| `dict_sys_get_new_row_id()` 在 `dict0dict.cc` | **错**。在 **`include/dict0boot.ic:36`**（`static inline`） |
| `dict_sys_t::row_id` 在 `dict0mem.h` | **错**。在 **`include/dict0dict.h:1017`** |
| `dict_index_t::n_core_fields` / `n_core_null_bytes` | **不存在**（无 `n_core_` 前缀符号）→ `n_uniq` + `trx_id_offset` |
| `trx_write_trx_id()` 在 `trx0sys.cc` | **错**。`trx_write/read_trx_id` 在 **`include/trx0sys.ic`**；`trx_write/read_roll_ptr` 在 **`include/trx0undo.ic`** |
| `mach_write_to_6/7` 在 `mach0data.cc` | **错**。四个都在 **`include/mach0data.ic`** |
| `dict_index_is_clustered()` | **不存在**（0 命中）→ `index->is_clustered()` |
| `Rec_offsets` 只是语法糖 | 内部固定 `REC_OFFS_NORMAL_SIZE=100` 的栈缓冲，且 `private ut::Non_copyable` |
| offsets 缓存是"页级缓存" | **错**。是**索引级** `dict_index_t::rec_cache`，只缓存前 `n_unique_in_tree` 个**定长且非 NULL** 列的偏移 |
| `rec_offs_init()` 是函数 | 是**宏**且用 `sizeof offsets`——对指针参数会静默算错 |

**★ 三个"结构性事实"**：

1. **`*rec_offs_base(offsets) = (rec - (lens + 1)) | REC_OFFS_COMPACT | any_ext` 这一行是整个 offsets 体系的枢纽**——`lens` 在循环里一路向低地址递减，循环结束时正好指向"最后一个被读的 len 字节"，`lens + 1` 就是变长长度数组的起始边界，`rec` 减去它就是 extra_size。COMPACT 主循环末尾与 node_ptr 分支末尾都有这一句（node_ptr 版不带 `any_ext`，因为非叶记录不允许 off-page 列）。

2. **`REC_INSERT_STATE` 的六态 + 两个互斥 bit（INSTANT/VERSION）是 8.0.29+ instant ADD/DROP 的完整复杂度**——`get_rec_insert_state()` 里 `ut_ad(!is_versioned || !is_instant)`，release 下两 bit 同时置位会打 "table may be corrupt" 的 error。V1（8.0.12~8.0.28）在记录里存"字段数"，V2（8.0.29+）存"row version"，两者在 `rec_init_null_and_len_comp()` 里走不同分支。

3. **offsets 的三级性能层次**：栈缓冲（`Rec_offsets` 的 800 字节，避免 malloc）→ 数组复用（`rec_get_offsets` 传回旧指针，避免重分配）→ 共享缓存（`rec_cache`，连解析都省掉，前提是定长且无 NULL 且非 instant/spatial/virtual）。第三级的消费点是 `page_cur_search_with_match()` 的二分查找——它直接返回 `cached_offsets` 而**完全跳过** `rec_get_offsets()`。

### 消费方清单

offsets 协议的下游（各篇详述）：比较 `cmp_dtuple_rec`（搜索/插入定位）、`row_build` 与 `rec_copy_prefix_to_dtuple`（record → dtuple）、`row_search_mvcc` 的游标推进（见 [`row_search.md`](../row_search.md)）、行锁的 `lock_rec_*`（见 [`../../lock/transactional/innodb_trx_lock.md`](../../infra/lock/transactional/innodb_trx_lock.md)）、purge 与 undo 回放（见 [`undo_log.md`](../undo_log.md)）、instant DDL 的行迁移（见 [`ddl.md`](../../feat/ddl/ddl.md)）。

---

## 参考

**社区文章 / 博客**
- [InnoDB 物理行中 null 值的存储的推断与验证 — fiona514（阿里内核月报专家投稿）](http://mysql.taobao.org/monthly/2016/08/07/)
- [InnoDB record 格式 — 利维坦](https://www.leviathan.vip/2022/06/08/innodb-record/)
- [littleneko/Notes — Record.md](https://github.com/littleneko/Notes/blob/main/MySQL/MySQL%20%E6%BA%90%E7%A0%81/Record.md)（dtuple/rec 双格式与插入数据结构）
- [InnoDB 行格式转换 — xpchild](https://www.cnblogs.com/xpchild/p/3885640.html)（mysql 行格式 ↔ SE 行格式）
- [insert 三大重要数据结构（row_prebuilt_t / ins_node_t / dtuple_t）— 知乎](https://zhuanlan.zhihu.com/p/103933731)
- 知乎 offsets 数组老文（[164705538](https://zhuanlan.zhihu.com/p/164705538)）：**5.7 口径已过时**——8.0 的 header/双坐标见本篇

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Row Formats（REDUNDANT / COMPACT / DYNAMIC / COMPRESSED）*
- WorkLog: [WL#11491 InnoDB: Support instant add column, using new column default values in redo/undo](https://dev.mysql.com/worklog/task/?id=11491)（instant 行格式）

**内核月报 / 技术文章**
- 阿里内核月报：*[InnoDB instant ADD COLUMN 记录格式](http://mysql.taobao.org/monthly/2023/12/02/)*（源码 `rem/rec.h` 的 `get_rec_insert_state` 注释直接引用了此文的状态表）

**相关文档**
- 页组织与页内查找：[`page_structure.md`](page_structure.md)
- instant DDL 语义与行版本号决策：[`ddl.md`](../../feat/ddl/ddl.md)
- 扫描取行（offsets 的下游）：[`row_search.md`](../row_search.md)
- dtuple 的 vfield 段与 `DATA_MISSING`：[`../feat/generated_columns.md`](../../feat/generated_columns.md)

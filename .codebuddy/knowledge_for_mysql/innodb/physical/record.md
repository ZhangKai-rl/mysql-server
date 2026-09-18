# InnoDB 行记录格式与 offsets 数组深度解析

> 基于 MySQL 8.0.39 源码，涵盖行的物理布局（record header / 变长长度 / NULL 位图 / info bits）、**offsets 数组**的设计与编码（`rec_get_offsets` 全链：入口 → 解析 → 字段定位）、REDUNDANT 的 1/2 字节偏移目录、instant ADD/DROP 与行版本在 offsets 层的投影、写方向的 `rec_convert_dtuple_to_rec`、以及 offsets 的消费方。
>
> **边界**：本篇讲**记录本身**（页里的"内容物"）。页的组织（FIL header、page directory、页内查找）见 [`page_structure.md`](page_structure.md)；instant DDL 的语义与决策见 [`ddl.md`](../ddl.md)（本篇只讲它在行格式/offsets 层的落地）；二级索引回表取行见 [`row_search.md`](../row_search.md)。

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
constexpr uint32_t REC_OFFS_SQL_NULL = 1U << 31;  // SQL NULL
constexpr uint32_t REC_OFFS_EXTERNAL = 1 << 30;   // 外置存储（off-page BLOB）
constexpr uint32_t REC_OFFS_DEFAULT = 1 << 29;    // instant ADD 后行里缺失、用默认值
constexpr uint32_t REC_OFFS_DROP = 1 << 28;       // instant DROP 掉的列
constexpr uint32_t REC_OFFS_MASK = REC_OFFS_DROP - 1;  // 低 28 位 = 真实偏移
```

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

### 消费方清单

offsets 协议的下游（各篇详述）：比较 `cmp_dtuple_rec`（搜索/插入定位）、`row_build` 与 `rec_copy_prefix_to_dtuple`（record → dtuple）、`row_search_mvcc` 的游标推进（见 [`row_search.md`](../row_search.md)）、行锁的 `lock_rec_*`（见 [`../../lock/transactional/innodb_trx_lock.md`](../../lock/transactional/innodb_trx_lock.md)）、purge 与 undo 回放（见 [`undo_log.md`](../undo_log.md)）、instant DDL 的行迁移（见 [`ddl.md`](../ddl.md)）。

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
- instant DDL 语义与行版本号决策：[`ddl.md`](../ddl.md)
- 扫描取行（offsets 的下游）：[`row_search.md`](../row_search.md)
- dtuple 的 vfield 段与 `DATA_MISSING`：[`../feat/generated_columns.md`](../../feat/generated_columns.md)

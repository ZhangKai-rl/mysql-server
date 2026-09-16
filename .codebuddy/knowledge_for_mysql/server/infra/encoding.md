# MySQL/InnoDB 数据编码

> 基于 MySQL 8.0.39 源码。涵盖 InnoDB 的 `mach_*` 序列化族（固定宽度 + 前缀码压缩编码）、端序规则、边界安全解析，以及它在页 / redo / 动态元数据中的应用。
>
> **边界**：本篇讲"字节级编码"这个通用机制；使用它的具体结构（页头、行记录、redo 记录）见 [`../../innodb/physical/`](../../innodb/physical/) 各篇。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`mach0data.h` 是 InnoDB 的**手动序列化工具族**：不依赖 C struct 内存布局，而是一组 `mach_read_from_N` / `mach_write_to_N` 函数，把整数按**明确定义的字节序**写入/读出任意字节缓冲区。

### 为什么需要它

这是 [`tablespace.md`](../../innodb/physical/tablespace.md) 里"`typedef byte` 范式"的必然配套：页上结构是裸字节，不能用 `struct` + 直接赋值（对齐填充破坏布局、端序随机器变化）。于是：

```cpp
/* 页上的 4 字节 = 表空间大小，无论什么 CPU 都写大端 */
mach_write_to_4(header + FSP_SIZE, size);
size = mach_read_from_4(header + FSP_SIZE);
```

**序列化格式 = 磁盘格式**，零转换成本，且跨平台稳定（备份文件可跨端序机器迁移）。

### 版本演进

| 版本 | 变化 |
|---|---|
| 1995（Heikki Tuuri 手写初版） | 固定宽度 `mach_*` 系列 |
| 长期演进 | 引入 `mach_write_compressed`（前缀码）→ `much_compressed`（64 位扩展） |
| 8.0 | 动态元数据 redo 用 `much_compressed`（1..11 字节）编码 table_id/version |

---

## 理论基础

### 设计思想与权衡

#### 〇、三类编码的正交关系（总纲）

`mach_*` 里混着好几类"编码"，它们**不是并列的选项，而是三个正交维度**：

| 维度 | 编码 | 解决的问题 | 例子 |
|---|---|---|---|
| **符号处理** | Offset Binary / Excess-N | 有符号数的**排序正确性**（补码负数高位=1，memcmp 会排错） | 索引键 INT 首字节 XOR 0x80 |
| **字节序** | 大端 / 小端 | 高低字节谁在前 | 整数大端（字典序）、浮点小端（跟 CPU） |
| **长度** | 定长 / VLQ / VLC | 小值省空间的**变长存储** | mach_write_compressed、fts_encode_int |

★ 三个维度**可以自由组合**，但有两个组合几乎不出现：

1. **要排序的字段必须定长**（变长让 memcmp 无法知道边界、无法跳过），而定长大值省不了空间 → **VLQ 不进索引键**；
2. **能用 VLQ 的字段（trx_id/LSN/autoinc）都非负** → 没有"负数排错序"问题 → **Offset Binary 不进 redo**。

所以实际是两套组合：**索引键 = 大端 + Offset Binary（定长）**；**redo/动态元数据 = 大端 + VLQ（变长、无符号）**。下文按这个框架展开：大端（一）、浮点小端（二）、压缩变长（三）、Offset Binary（核心实现·有符号整数）。

#### 一、★ 整数为什么一律大端：字节序 = 字典序

`mach_write_to_2` 的实现里，本仓库有一句中文注释点破了本质：

```cpp
// note: 从这里看出是大端字节序big endian：低地址存储高字节
b[0] = (byte)(n >> 8);
b[1] = (byte)(n);
```

**大端编码的字节串按 `memcmp` 比较的结果，等价于对原数值比较**——高位字节在前，两个字节串从前往后比，第一次不同的字节就决定了数值大小。这个性质让以下操作**免费**：

- 行记录里 `DB_TRX_ID`（6 字节大端）直接参与 MVCC 的版本序比较，不必解码成整数；
- undo / redo 里按 LSN、trx_id 排序的物理块可以**按字节二分**；
- `memcmp` 就能回答"谁的 trx_id 更大"。

**被放弃的方案**：小端 + 每次比较前解码。那样每次版本判断都要 `mach_read_from_6` 再比较，热点路径上平白多一次函数调用与移位。大端把"解码"延后到真正需要数值时（极少数场景），把"比较"留在字节层。

**代价**：任何需要数值的场合都要显式 `mach_read_from_N` 解码（不能直接类型转换），读代码时多一层心智负担。

#### 二、浮点为什么刻意小端（反例，证明"不是所有 mach 都大端"）

```cpp
static inline double mach_double_read(const byte *b);   // little-endian
```

浮点存的是 **IEEE 754 位模式**，与主流小端 CPU 的内存布局一致——`memcpy` 后可直接当 double 用。这里**不追求字典序**（浮点排序本来就不能用整数 memcmp 的规则），所以端序选择是"跟 CPU 对齐"，而不是"跟比较对齐"。

★ 教训：**读代码不能默认 `mach_*` 都是大端**——整数大端（为比较）、浮点小端（为对齐），各有各的理由。

#### 三、压缩编码：前缀码（prefix code）的自同步设计

`mach_write_compressed` 的核心思想是**首字节的高位连续 1 的个数 = 总字节数**：

```
0nnnnnnn                        1 字节，7 位
10nnnnnn nnnnnnnn               2 字节，14 位
110nnnnn nnnnnnnn nnnnnnnn      3 字节，21 位
1110nnnn ...                    4 字节，28 位
11110000 + 4 字节               5 字节，32 位
```

★ 这是**自同步前缀码**：读首字节就能立刻知道后面还有几个字节，不需要长度字段、不需要前向扫描。对比"定长 8 字节 + 1 字节长度"，前缀码把"长度信息"塞进首字节的空闲位，**小值省空间、大值才多花**。

**权衡**：`much_compressed` 编码 64 位大值要 **11 字节**（`0xFF` 分隔 + 高 32 位压缩 5 字节 + 低 32 位压缩 5 字节），比定长 8 字节还长——**压缩的是小值，不是所有值**。InnoDB 赌的是"表 id / autoinc 计数 / LSN 偏移这类高频元数据在绝大多数时刻是小数"。

这个权衡的落点就是动态元数据 redo：autoinc 计数早期值很小，写进 redo 只需 1 字节；等它涨到 2^32 以上，每次多花几字节也无关痛痒——**数据特性（偏小、单调增）驱动编码选择**。

---

## 核心实现

### 固定宽度：逐字节展开（大端）

```cpp
// mach0data.ic:127 —— 4 字节
static inline void mach_write_to_4(byte *b, ulint n) {
  b[0] = static_cast<byte>(n >> 24);   // 最高字节 → 最低地址
  b[1] = static_cast<byte>(n >> 16);
  b[2] = static_cast<byte>(n >> 8);
  b[3] = static_cast<byte>(n);          // 最低字节 → 最高地址
}

// mach0data.ic:140 —— 读回
static inline uint32_t mach_read_from_4(const byte *b) {
  return ((uint32_t(b[0]) << 24) | (uint32_t(b[1]) << 16) |
          (uint32_t(b[2]) << 8)  |  uint32_t(b[3]));
}
```

★ **不依赖机器端序**——显式移位展开，无论跑在大端还是小端 CPU，`b[0]` 永远是最高字节。这就是"跨平台备份"的物理基础。

8 字节 = 两个 4 字节拼：

```cpp
// mach0data.ic:331
static inline void mach_write_to_8(void *b, uint64_t n) {
  mach_write_to_4((byte *)b, (ulint)(n >> 32));        // 高 32 位在前
  mach_write_to_4((byte *)b + 4, (ulint)n);            // 低 32 位在后
}

// mach0data.ic:342
static inline uint64_t mach_read_from_8(const byte *b) {
  uint64_t u64 = mach_read_from_4(b);   // 高 32 位
  u64 <<= 32;
  u64 |= mach_read_from_4(b + 4);       // 低 32 位
  return u64;
}
```

宽度规格：1/2/3/4/6/7/8 字节都有对应函数（`mach_write_to_6` = 2+4，`mach_write_to_7` = 3+4，`mach_write_to_3` 独立实现）。**6 字节就是 trx_id、7 字节就是 ROLL_PTR**——没有为它们单独造格式，就是"少写几个字节"。

### ★★ 有符号整数：Offset Binary（Excess-N）—— 让 memcmp 正确排序负数

前面讲的是**无符号**整数。但索引键里有有符号列（INT / BIGINT），它们的负数怎么办？

**问题**：有符号补码里负数的最高位是 1（如 `-1` = `0xFF...FF`），如果按无符号大端存，`memcmp` 会认为 `-1` > `0`——**负数排到正数后面，索引序全错**。

**解法**（`row0mysql.cc:442`，本仓库有详细中文注释）：**最高字节 XOR 0x80**，把补码数轴平移到无符号空间：

```
补码表示:     -2^31  ...  -1    0    1   ...  2^31-1
最高位:         1          1    0    0          0
XOR 0x80 后:    0          0    1    1          1
无符号值:       0   ...  0x7F  0x80 0x81 ... 0xFF...
对应大小:      最小 ←———————————————→ 最大
```

```cpp
// row0mysql.cc:453 —— 写索引键时
*buf ^= 128;      // 只翻转最高字节的符号位
```

★ **这是 Excess-N 编码（偏移编码）**：原值加一个固定偏移 0x80（针对最高字节），使编码后的字节序与数值序一致。收益是**有符号整数也能被 `memcmp` 直接比较**——索引里正负数混排的键，B-tree 的字节比较天然正确，范围扫描无需解码。

读回时反向（`mach0data.ic:726-736`）：

```cpp
if (unsigned_type || (src[0] & 0x80)) {
  ret = 0;                          // 无符号 或 原符号位为 1（负数）→ 高位补 0
} else {
  ret = 0xFFFFFFFFFFFFFF00ULL;       // 正数 → 符号扩展，高位补 1
}
if (unsigned_type) ret |= src[0];
else ret |= src[0] ^ 0x80;          // ★ 有符号 → 首字节再 XOR 0x80 还原
```

★ **编解码各 XOR 一次 0x80**（写时翻转、读时翻回来），再配合符号扩展（高位补 `0xFF` 还是 `0x00`）。这解释了为什么"索引里的 INT 是 4 字节大端但首字节看着不对"——它被 XOR 过了。

### ★ 压缩编码：`mach_write_compressed` 的 8 个分支

```cpp
// mach0data.ic:156 —— 32 位前缀码压缩
static inline ulint mach_write_compressed(byte *b, ulint n) {
  if (n < 0x80) {                      // 0nnnnnnn → 1 字节（7 位）
    mach_write_to_1(b, n);  return 1;
  } else if (n < 0x4000) {             // 10nnnnnn nnnnnnnn → 2 字节（14 位）
    mach_write_to_2(b, n | 0x8000);    // 最高位置 1 作"2 字节"标记
    return 2;
  } else if (n < 0x200000) {           // 110nnnnn ... → 3 字节（21 位）
    mach_write_to_3(b, n | 0xC00000);
    return 3;
  } else if (n < 0x10000000) {         // 1110nnnn ... → 4 字节（28 位）
    mach_write_to_4(b, n | 0xE0000000);
    return 4;
  } else if (n >= 0xFFFFFC00) {        // 111110nn → 2 字节（extended，10 位）
    mach_write_to_2(b, (n & 0x3FF) | 0xF800);
    return 2;
  } else if (n >= 0xFFFE0000) {        // 1111110n → 3 字节（extended，17 位）
    mach_write_to_3(b, (n & 0x1FFFF) | 0xFC0000);
    return 3;
  } else if (n >= 0xFF000000) {        // 11111110 → 4 字节（extended，24 位）
    mach_write_to_4(b, (n & 0xFFFFFF) | 0xFE000000);
    return 4;
  } else {                             // 11110000 + 4 字节 → 5 字节（32 位）
    mach_write_to_1(b, 0xF0);
    mach_write_to_4(b + 1, n);
    return 5;
  }
}
```

编码表：

| 值范围 | 首字节前缀 | 字节数 | 有效位 |
|---|---|---|---|
| `< 0x80` | `0xxxxxxx` | 1 | 7 |
| `< 0x4000` | `10xxxxxx` | 2 | 14 |
| `< 0x200000` | `110xxxxx` | 3 | 21 |
| `< 0x10000000` | `1110xxxx` | 4 | 28 |
| `>= 0xFFFFFC00` | `111110xx` | 2 | 10（extended） |
| `>= 0xFFFE0000` | `1111110x` | 3 | 17（extended） |
| `>= 0xFF000000` | `11111110` | 4 | 24（extended） |
| 其余 | `11110000` + 4B | 5 | 32 |

★ **几个反直觉点**：

1. **"extended" 分支是给"高位全 1 的大值"的兜底**：普通 4 字节分支只能装 28 位，而 `n >= 0x10000000` 且高位连续的 1（如 `0xFFFFFFFF`）无法用 `1110` 前缀表达——于是用更长的前缀（`111110`/`1111110`/`11111110`）换更少的数据位。所以**同样是 4 字节编码，`0x1FFFFFFF` 用 `1110` 前缀、`0xFFFFFFFF` 用 `11111110` 前缀**，二者不冲突，靠前缀自同步区分。
2. **读首字节即知长度**：`0`→1 字节、`10`→2、`110`→3、`1110`→4、`11110000`→5——**不需要长度字段**，这是前缀码的核心收益。

### ★★ 两个 64 位压缩函数的精确关系（易混！）

`mach0data.h` 里有两个名字很像的 64 位压缩，**编码策略不同**：

```cpp
// ① mach_u64_write_compressed（5..9 字节）—— 高压缩、低定长
// mach0data.ic:396
static inline ulint mach_u64_write_compressed(byte *b, uint64_t n) {
  ulint size = mach_write_compressed(b, (ulint)(n >> 32));  // 高 32 位 VLQ（1..5B）
  mach_write_to_4(b + size, (ulint)n);                      // ★ 低 32 位固定 4 字节，不压缩
  return size + 4;                                          // = 5..9 字节
}

// ② mach_u64_write_much_compressed（1..11 字节）—— 两段全压缩
// mach0data.ic:426
static inline ulint mach_u64_write_much_compressed(byte *b, uint64_t n) {
  if (!(n >> 32)) {
    return mach_write_compressed(b, (ulint)n);   // ★ 高 32 位为 0 → 整个退化为 32 位
  }
  *b = (byte)0xFF;                                // 高 32 位非 0 → 0xFF 分隔标记
  ulint size = 1 + mach_write_compressed(b + 1, (ulint)(n >> 32));   // 高 32 位 VLQ
  size += mach_write_compressed(b + size, (ulint)n & 0xFFFFFFFF);    // 低 32 位也 VLQ
  return size;
}
```

| | ① `write_compressed` | ② `much_compressed` |
|---|---|---|
| 长度范围 | **5..9 字节** | **1..11 字节** |
| 高 32 位 | VLQ（1..5B） | VLQ |
| 低 32 位 | ★ **固定 4 字节（不压缩）** | ★ **也 VLQ** |
| 小值 | 恒 5 字节 | **1 字节** |
| 大值 | 9 字节 | 11 字节 |
| 读侧 | `mach_u64_read_next_compressed`（低 32 位直接 `read_from_4`） | `mach_u64_read_much_compressed` |

★ **为什么并存两个**——**"低 32 位是否压缩"是空间换解析简单性的权衡**：

- ① 低 32 位定长 ⇒ 读时直接 `mach_read_from_4`，解析最简；适合"高 32 位非零、低 32 位随机"的值（典型是 LSN 类）；
- ② 两段全压缩 ⇒ **小值只要 1 字节**，但大值更费且解析要走两次 VLQ；适合"绝大多数值 < 2^32、偶尔很大"的值（典型是 autoinc 计数器、table_id——动态元数据 redo 选它正是为此）。

★ 之前一版文档把两者混为一谈，现在明确了：**动态元数据 redo（`MLOG_TABLE_DYNAMIC_META`）用的是 ② much_compressed**，注释里说的 "each of which would cost 1..11 bytes" 与之对应。

读侧三个变体（`mach0data.h:167-205`）的分工：

| 读函数 | 特点 | 用在哪 |
|---|---|---|
| `mach_u64_read_next_compressed` | 指针前移（顺序解析） | 正向扫 redo |
| `mach_u64_read_much_compressed` | 指针不动 | 随机读 |
| `mach_parse_u64_much_compressed(ptr, end_ptr)` | ★ **带边界检查**，越界置 NULL | 崩溃恢复、解析不可信 redo |

★ **`read` vs `parse` 的分野是安全边界**：`read` 系列假设缓冲区有效（页内解析，长度已被结构保证）；`parse` 系列面对**不可信输入**（redo 日志、损坏页），必须带 `end_ptr` 检查，否则损坏日志会把解析器带出缓冲区。崩溃恢复代码里一律用 parse。

### ★ FTS VLC：另一种变长编码（结束标志在最后字节）

FTS（全文索引）用一套**与 VLQ 完全不同的**变长编码（`fts0vlc.ic`）。关键差异：**VLQ 的"长度"在前（前缀码），VLC 的"结束标志"在后（后缀标志）**：

```cpp
// fts0vlc.ic:64 —— 编码
static inline ulint fts_encode_int(ulint val, byte *buf) {
  ...
  /* High-bit on means "last byte in the encoded integer". */
  *buf |= 0x80;      // ★ 最高位 1 = 这是最后一个字节
  return len;
}

// fts0vlc.ic:114 —— 解码
static inline ulint fts_decode_vlc(byte **ptr) {
  ulint val = 0;
  for (;;) {
    byte b = **ptr;
    ++*ptr;
    val |= (b & 0x7F);     // 低 7 位是数据
    if (b & 0x80) break;   // 最高位 1 = 结束
    val <<= 7;             // 否则左移 7 位继续
  }
  return val;
}
```

长度规格：`≤127`=1B、`≤16383`=2B、`≤2097151`=3B、`≤268435455`=4B、其余 5B。

**两种策略对比**：

| | VLQ（`mach_write_compressed`） | VLC（`fts_encode_int`） |
|---|---|---|
| 长度信息位置 | **首字节前缀**（`0`/`10`/`110`…） | **末字节标志位**（`0x80`） |
| 每字节有效位 | 首字节 7 位、后续 8 位（前缀码按长度给数据位） | 恒定 7 位（末字节也是 7 位） |
| 长度范围 | 1..5 字节 | 1..5 字节 |
| 解码方向 | 看首字节即可定长 | **必须扫到末字节才知道结束** |
| 用在哪 | redo / undo（要能按字节定位边界） | FTS ilist（纯顺序追加、顺序读） |

★ **为什么 FTS 用后缀标志**：FTS 的 doc_id 是**顺序追加、顺序读**的字节流，解码时逐个字节扫即可；而 redo 里的 VLQ 要求"读首字节立即知道整条记录长度"（方便跨过、定位下一条），所以用前缀码。**两种编码各自匹配自己的访问模式**。

### 浮点与字符串

- 浮点：`mach_double_read/write`、`mach_float_read/write`——**小端**（见理论基础二）。
- ★ 字符串的**行内编码不在这里**——行记录里 VARCHAR 的"长度前缀"（1 字节 vs 2 字节、`0x40`/`0xc0` 外置位、长度数组逆序存放、NULL 位图）是**行格式**的专属编码，第一主语是行记录，完整规则见 [`record.md`](../../innodb/physical/record.md) 的「行的物理布局 / 变长长度编码」两节。本文只覆盖"跨结构的字节级通用编码"（端序、变长整数、Offset Binary）。

---

### Server 层：与 InnoDB 相反的另一套

Server 层有自己的编码体系，**端序与 InnoDB 相反（小端为主）**：

| 编码 | 端序 | 用途 |
|---|---|---|
| `int2store/int3store/int4store/int8store`（`my_byteorder.h`） | **小端** | 协议包、binlog event、临时缓冲 |
| Length-Encoded Integer（`mysys/pack.cc:129`） | 变长 | 客户端协议字段长度：`<251`=1B、`<64K`=3B、`<16M`=4B、`>=16M`=9B |
| Row Format lenenc | — | `table::record[0]` 里 VARCHAR：列 ≤255 用 1 字节长度前缀、>255 用 2 字节。发生点：InnoDB 存取行时（`row0mysql.cc` 的 `row_mysql_store_true_varchar` / `row_mysql_read_true_varchar`），详情见 [`record.md`](../../innodb/physical/record.md) |
| Key Format lenenc | — | 索引键缓冲里 VARCHAR：**总是 2 字节**长度前缀（键偏移固定，不省空间）。发生点：构建索引键时（`row0ins.cc` 的 `row_build_index_entry`） |

★ **为什么 InnoDB 大端、Server 小端**：InnoDB 的整数要进索引键参与 `memcmp` 字节比较（所以大端 = 字典序）；Server 层的 `record[0]` / 协议包是"写进内存缓冲、机器直接读"，跟随 CPU 本机端序（x86 小端）最省事——**同一份数据，在"要字节比较"的地方大端、在"机器直接读"的地方小端**，两套各有理由，不是矛盾。

##### ★ Length-Encoded Integer 的完整位布局（`mysys/pack.cc:129`）

```cpp
uchar *net_store_length(uchar *packet, ulonglong length) {
  if (length < 251LL) {           // < 251：1 字节直接存
    *packet = (uchar)length;
    return packet + 1;
  }
  /* 251 is reserved for NULL */  // ★ 0xFB 被保留为 NULL 标记！
  if (length < 65536LL) {         // < 64K：0xFC + 2 字节小端
    *packet++ = 252;
    int2store(packet, (uint)length);
    return packet + 2;
  }
  if (length < 16777216LL) {      // < 16M：0xFD + 3 字节小端
    *packet++ = 253;
    int3store(packet, (ulong)length);
    return packet + 3;
  }
  *packet++ = 254;                // >= 16M：0xFE + 8 字节小端
  int8store(packet, length);
  return packet + 8;
}
```

| 值范围 | 编码 | 字节数 |
|---|---|---|
| `< 251` | 原值直存 | 1 |
| `< 64K` | `0xFC` + int2store 小端 | 3 |
| `< 16M` | `0xFD` + int3store 小端 | 4 |
| `>= 16M` | `0xFE` + int8store 小端 | 9 |

★ **这是第三种变长策略：标记字节 + 负载**——首字节 252/253/254 是"长度档位标记"（与 VLQ 的位前缀、VLC 的末字节标志都不同），负载恒小端。且 **251（`0xFB`）被保留为 NULL**，所以 1 字节档只能装到 250。

★ 三种变长策略至此齐了：**VLQ 前缀码**（位标记在前，redo）/ **VLC 后缀标志**（结束位在后，FTS）/ **lenenc 档位标记**（独立标记字节，协议）。选型的本质还是访问模式：协议包是"顺序写、顺序读"且字段常有 NULL 语义，档位标记最直白。

★ **使用范围澄清（易误解）**：`net_store_length` **不只是"网络通信"**——它是 MySQL"线协议/流式格式"的长度编码，三大使用场景：

| 场景 | 文件 | 是网络吗 |
|---|---|---|
| 客户端协议 | `sql/protocol_classic.cc`、`sql-common/client.cc` | ✅ |
| ★ binlog 事件 | `sql/log_event.cc`、`libbinlogevents/src/codecs/binary.cpp` | ❌ 磁盘文件 |
| ★ 复制 row 格式 | `sql/rpl_record.cc` | 流式 |

即"流式序列化"（协议 / binlog / 复制）共用这一套，与 InnoDB 的**存储**编码（`mach_*`）完全无关——两者分属"线格式"与"磁盘格式"两条线。

**逐调用点**（全库 `net_store_length` 核实）：

| 调用点 | 编码什么 |
|---|---|
| `protocol_classic.cc:510/:531` | 结果集字符串字段的长度 |
| `protocol_classic.cc:588` | 包长度域（先占位、写完回填） |
| `protocol_classic.cc:898/:901` | OK 包的 `affected_rows` / `last_insert_id` |
| `protocol_classic.cc:3096` | 结果集列数（`num_cols`） |
| `protocol_classic.cc:1251-1269` | ★ `net_store_length_fast`：已知 `< 64K` 时跳过档位判断的快速路径 |
| `log_event.cc:10678/:10680/:10687` | ★ Query 事件里的 **db 名 / 表名 / 列数**（`dbuf`/`tbuf`/`cbuf`） |
| `log_event.cc:8103` | ★ Table_map 事件的列类型宽度（`m_width`） |
| `rpl_record.cc` | 复制 row 事件的字段值 |

★ 注意 binlog 用的是**同一套 lenenc**——所以 binlog 事件内部大量出现 `0xFC`/`0xFD` 开头的小端长度，用 `mysqlbinlog --hexdump` 看事件时就是这些档位字节。

### 历史别名已清理：`mach_ull_*` / `mach_dulint`

全库搜索 `mach_ull_*` 与 `mach_dulint` 均 **0 命中**——这两个是 5.x 时代的类型别名（`dulint` = 双 32 位拼 64 位的时代产物），8.0 统一改为 `uint64_t` + `mach_read_from_8` 后已删除。**读老资料时遇到 `dulint` 不要找实现，8.0.39 里没有**。

### Canonical Format 与 Hex Encoding（两个特殊格式）

- **Canonical Format**（`mach_encode_2` / `mach_decode_2`，`mach0data.ic:81`）：把 16-bit 整数转成**本机可 `memcmp` 的规范格式**，只用于**内存快速相等比较**，不落盘——一次转换后多次比较，避免每次比较都做字节序转换。
- **Hex Encoding**（`dict0dd.cc:79`）：DD 的 `se_private_data` 是 TEXT 列，**存不了原始二进制**，于是把字节流拆成两个 ASCII hex 字符（`0xFF` → `'F','F'`），长度翻倍但可打印。`DD_instant_col_val_coder::encode/decode` 是它的使用点——instant ADD COLUMN 的默认值经它转成文本存进 `se_private_data`。

## 应用场景速查

| 场景 | 用的编码 | 见 |
|---|---|---|
| FSP header（FSP_SIZE / FSP_FREE_LIMIT…） | `mach_*_4` 大端 | [`tablespace.md`](../../innodb/physical/tablespace.md) |
| XDES / inode 链表地址 `(page_no, boffset)` | 4+2 共 6 字节 | 同上 |
| 行记录 DB_TRX_ID / DB_ROLL_PTR | 6 / 7 字节大端 | [`record.md`](../../innodb/physical/record.md) |
| ★ 索引键里的有符号列（INT/BIGINT） | **Offset Binary**（大端 + 首字节 XOR 0x80） | 同上 |
| redo 记录（type/space_id/page_no/LSN） | 1/4/4/8 + VLQ compressed | [`../dd/innodb_dict.md`](../dd/innodb_dict.md) 动态元数据节 |
| 动态元数据（autoinc / corrupt 索引） | ★ `much_compressed`（1..11B） | 同上 |
| FTS 辅助索引 ilist（doc_id delta、word position） | ★ **VLC**（末字节 0x80 结束标志） | [`../../feat/fts.md`](../../feat/fts.md) |
| `se_private_data` 里的 instant 默认值 | ★ **Hex Encoding**（字节 → ASCII hex） | [`../dd/dd.md`](../dd/dd.md) 2.4 |
| ★ 客户端协议字段长度 / OK 包 / 列数 | **Length-Encoded Integer** | 本节"使用范围澄清" |
| ★ binlog 事件（Query 的 db/表名、Table_map 宽度）、复制 row 格式 | **Length-Encoded Integer** | 同上 |
| 协议包头 3 字节长度、binlog 事件头、`record[0]` 数值字段 | `intNstore` 小端（`my_byteorder.h`） | — |

---

## Misc

### 本知识库"格式/编码"文档的分工（避免重复）

编码/格式内容散在几篇里，各管一段，互不重复：

| 文档 | 管哪一段编码 | 关键词 |
|---|---|---|
| **本文（encoding.md）** | **跨结构的字节级通用编码** | mach 固定宽度 / VLQ / VLC / Offset Binary / 端序 / Server 协议 |
| [`record.md`](../../innodb/physical/record.md) | **行内字段级编码** | 变长长度前缀（1/2B、`0x40`/`0xc0` 外置位、逆序）、NULL 位图、REDUNDANT 目录、DB_TRX_ID/ROLL_PTR |
| [`tablespace.md`](../../innodb/physical/tablespace.md) / [`page_structure.md`](../../innodb/physical/page_structure.md) | **页头字段的字节布局** | FIL header、FSP header、XDES、inode 的逐字段偏移 |
| [`redo_log.md`](../../innodb/redo_log.md) | **redo 记录格式** | mlog 记录头、物理/逻辑日志类型 |

★ **判断内容该进哪篇**：问"这个编码是**谁专属**的"——行内 VARCHAR 长度前缀是**行格式**专属（进 record.md）；mach 变长整数被页/redo/动态元数据**复用**（进本文）；页头字段布局是**页结构**专属（进 page_structure.md）。同一份编码只在专属的那篇详写，其余交叉引用。

### 容易误解的概念

| 误解 | 正解 |
|---|---|
| "Offset Binary / Excess-N / VLQ 是三种并列编码" | **Offset Binary = Excess-N**（同义词，解决有符号排序）；**VLQ 是另一维度**（变长省空间）。三者正交（见理论基础〇） |
| "mach 系列都是大端" | **浮点是刻意的小端**；还有 `mach_read_from_n_little_endian` 等例外 |
| "压缩编码一定比定长短" | `much_compressed` 大值要 11 字节，比定长 8 字节还长——压缩的是**小值** |
| "read 和 parse 一样" | `mach_parse_*` 带 `end_ptr` 边界检查，不可信输入（redo/损坏页）必须用 parse |
| "4 字节压缩就一种" | 前缀不同：`1110`（28 位）vs `11111110`（extended 24 位），靠前缀自同步 |

### redo 解析里的 `mach_parse_*` 清单（带边界检查的读）

崩溃恢复（`log0recv.cc` 的 `recv_parse_log_recs`）面对的是**可能损坏的 redo 日志**，全程使用带 `end_ptr` 的 parse 族：

| 函数 | 解析对象 |
|---|---|
| `mach_parse_compressed(ptr, end_ptr)` | 32 位 VLQ（redo 里的压缩字段） |
| `mach_parse_u64_much_compressed(ptr, end_ptr)` | 64 位 much_compressed（动态元数据 redo） |
| `mlog_parse_initial_log_record(ptr, end_ptr, &type, &space_id, &page_no)` | ★ 每条 redo 记录的头（type + space + page） |
| `mlog_parse_initial_dict_log_record(ptr, end_ptr, type, &id, &version)` | ★ `MLOG_TABLE_DYNAMIC_META` 的 table_id + version |

★ 解析失败时函数把 `*ptr` 置 NULL，调用方据此判定"记录不完整 / 日志损坏"——**redo 解析器对每一条记录都做边界校验**，这是崩溃恢复不能信任磁盘内容的体现。

### 页压缩（另一条线：数据压缩，不是编码）

`KEY_BLOCK_SIZE` / 透明页压缩（zlib / LZ4，`os/file.cc` 与 `page_zip`）属于**数据压缩**而非本文的"数值编码"——它压的是整页字节流，与端序/变长无关。边界说明：页压缩见 [`../../innodb/physical/page_structure.md`](../../innodb/physical/page_structure.md)，本文不展开。

### 待补清单

（全部完成，暂无待补）

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Page Structure*（页字段的大端约定）

**源码**
- `storage/innobase/include/mach0data.h`（API + 注释）、`mach0data.ic`（实现）

**相关文档**
- `typedef byte` 范式与页结构见 [`../../innodb/physical/tablespace.md`](../../innodb/physical/tablespace.md)
- 动态元数据 redo 使用点见 [`../dd/innodb_dict.md`](../dd/innodb_dict.md)

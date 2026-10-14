# MySQL/InnoDB 字符集与排序规则（charset & collation）深度解析

> 基于 MySQL 8.0.39 源码。涵盖 `CHARSET_INFO` 结构与注册表、8 个通用 collation 算法家族、**UCA 900 权重算法（扫描器 + 权重表 + contraction + implicit weight）**、strxfrm 排序键、**InnoDB 侧的比较回调路径**、DDL/数据字典/协商、协议层转换链。
>
> **边界**：本篇讲字符集与排序规则这个通用机制本身；`mach_*` 字节序列化编码见 [`encoding.md`](encoding.md)；比较操作在索引里的落点见 [`../../innodb/row_search.md`](../../innodb/row_search.md)；**collation 不一致导致索引不可用**的判定见 [`../query/07_optimize/21_collation_index_usability.md`](../query/07_optimize/21_collation_index_usability.md)（本篇不重复展开）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

字符集（charset）回答"字节 ↔ 字符怎么映射"，排序规则（collation）回答"字符怎么比大小"。在 MySQL 里两者打包成一个对象：`CHARSET_INFO`——一个结构体里同时挂着字符集函数表（`cset`：判定字符类别、多字节边界、与 Unicode 互转）和排序函数表（`coll`：比较、生成排序键、LIKE 匹配、哈希）。

### 用途

一个反直觉的开场——MySQL 里 `SELECT 'a' = 'a ';` 的结果**取决于 collation**：

```sql
SELECT 'a' = 'a ' COLLATE utf8mb4_general_ci;   -- 1（PAD SPACE：尾部空格被补齐后相等）
SELECT 'a' = 'a ' COLLATE utf8mb4_0900_ai_ci;   -- 0（NO PAD：尾部空格参与比较）
```

同一个 SQL、同样的两个串，换个 collation 结果相反。这就是 collation 的语义差异：它不仅决定"大小写敏不敏感"，还决定**尾部空格算不算数**、**重音算不算数**（ai/accent insensitive）、**日文假名要不要区分**（ks/kana sensitive）。本篇要回答的核心问题就是：这些语义差异在代码里是怎么实现的、为什么这样设计。

在整条链路上的位置一句话概括：**SQL 文本在 parser 前被字符集解码（`character_set_client`），比较运算走 collation 的 `strnncoll`/`strnncollsp`，排序与索引比较走 `strnxfrm` 生成的权重键，结果返回前按 `character_set_results` 转码**。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.5 前 | collation 只支持单字节 + 8 位表（sort_order 表）；utf8 是 3 字节 |
| 5.5 | utf8mb4（完整 Unicode 4 字节）引入；collation 可动态加载（`--character-sets-dir`） |
| 5.7 | utf8mb4_general_ci/unicode_ci 普及；`default_collation_for_utf8mb4` 变量引入 |
| 8.0 | **utf8mb4_0900_* 系列**（UCA 9.0.0，NO PAD，ai/as/ja_ks 分级比较）；utf8mb4 成为默认字符集；数据字典（dd）接管 collation 元数据（`mysql.collations` 表）；`BINARY` 属性的 deprecated 处理；**`levels_for_order` 字段删除**（比较与排序合并共用一套权重机，5.x 时代的"比较/排序分级分离"设计退役）；**配置式新增 collation 路径删除**（`my_charset_rule.c` 整体移除，Index.xml 里的 `<rules>` 简单字符集规则语义不复存在） |
| 8.0 后期 | utf8mb4_0900_ai_ci 成为 utf8mb4 默认 collation（`default_collation_for_utf8mb4` 默认值从 general_ci 换成它） |
| **8.0.17** | 新增 `utf8mb4_0900_bin`——utf8mb4 成为唯一有**两个 binary collation** 的字符集（`utf8mb4_bin` 是 PAD SPACE，`utf8mb4_0900_bin` 是 NO PAD 且按 UTF-8 字节比，排序结果相同但快得多） |
| **8.0.28** | `ucs2` 字符集 deprecated；`utf8mb3` 开始取代 `utf8` 显示在 I_S 列与 `SHOW` 输出中 |
| **8.0.30** | 新增一批 0900 语言 collation：保加利亚语 `bg`、塞尔维亚拉丁字母 `sr_latn`、波斯尼亚语 `bs`、挪威尼诺斯克语 `nn`、蒙古语西里尔 `mn_cyrl`、加利西亚语 `gl` |

**别名演进的一个细节**（官方文档）：`utf8` 目前是 `utf8mb3` 的别名且该用法已 deprecated，**预期在未来版本 `utf8` 将转而成为 `utf8mb4` 的别名**。所以写 SQL 时显式写 `utf8mb4` 是唯一稳妥的做法——依赖 `utf8` 的语义将来会变。

**为什么演进要做 0900 系列**（官方给出的动机归纳）：旧的 `*unicode_ci`（UCA 4.0.0）只**部分支持** UCA——不支持组合标记（combining marks），影响越南语、约鲁巴语、纳瓦霍语等；一个组合字符与"写成单个 Unicode 字符的同一字符"被视为不同（连 `CHAR_LENGTH()` 都不同）。0900 系列基于 UCA 9.0.0 + CLDR v30，覆盖全 Unicode 范围、支持 NO PAD 语义、并提供 ai/as/ks 分级能力——同时官方称其**比旧 UCA 版本更快**。

---

## 理论基础

### 设计思想与权衡

**权衡一：数据与算法分离——权重表是数据，扫描器是算法。** 8.0 的 UCA collation 不是每个 collation 一份算法代码，而是"一份扫描器 + 一份权重数据"：

- `ctype-uca.cc` 里只有一套 `uca_scanner_900` 扫描器模板
- `strings/uca900_data.h` / `uca900_zh_data.h` / `uca900_ja_data.h` 是**编译期生成的大权重表**（几万行）
- 语言差异（zh 的拼音序、ja 的假名序）通过 `cs->tailoring` 规则串 + 定制数据文件注入，算法不动

代价：权重表巨大（UCA 900 数据文件是仓库里最大的几个头文件之一），编译慢、二进制大；换来的是"加一个语言 collation = 加一行数据 + 一条注册"，算法零改动。

**权衡二：InnoDB 不理解字符集，比较回调 server 层。** 行数据在 InnoDB 里是字节流（BLOB/VARCHAR 的字节 + 长度），索引比较对字符串类型通过 `dtype` 里存的 collation id 回调 server 的 `strnncollsp`。本来可以"把 collation 逻辑下沉进 InnoDB"，但那样引擎就要链接 Unicode 权重表、理解 MySQL 的 charset 语义——引擎与 server 的边界会被破坏，插件化的其他引擎（MyISAM/MEMORY）也要各实现一遍。代价是：**每次比较是一次跨层函数指针回调**，且 InnoDB 侧无法对"纯 ASCII 快路径"这类 server 内优化直接复用（详见核心实现·InnoDB 侧）。

**权衡三：PAD SPACE 是历史债务。** 老 collation 全是 PAD SPACE（SQL 标准要求），8.0 新 0900 系列改成 NO PAD（UCA/ICU 的世界惯例）。这个语义差异造成 `strnncoll` 与 `strnncollsp` 两个函数并存、索引比较与表达式比较行为可能不一致的复杂度——但为了兼容旧数据不能改老 collation，只能新老并存。

**权衡四：单字节表 vs Unicode 权重，两套机制并存。** latin1/simple 系走 256 字节 `sort_order` 查表（快、内存小、只覆盖单字节）；utf8mb4/ucs2 系走 UCA 权重表（慢、内存大、覆盖全 Unicode）。没有"统一成一套"是因为单字节表的比较是 O(1) 查表 + 一次字节比较，UCA 扫描器是逐字符多级权重比较——对 latin1 用户统一成 UCA 是纯亏。

### 理论溯源

把每条理论**落到源码具体位置**（不是列书名）：

| 理论 | 出处 | 源码落点 |
|---|---|---|
| 多级权重比较模型（primary→secondary→tertiary） | Unicode TR10 §4（Main Algorithm） | `my_strnncoll_uca` / `my_strnncoll_uca_900` 的外层 `current_lv` 循环 |
| Collation Element 与 DUCET | TR10 §5；DUCET 4.0 / 5.2 / 9.0.0 三个版本 | `my_uca_v400`(`uca_data.h`) 与 `my_uca_v900`(`uca900_data.h`) 两套权重数据 |
| implicit weight（未赋权码点按码点派发） | TR10 §7.1.3 | `uca_scanner_900::next_implicit` 的 0xFBC0 / 0xFB40 / 0xFB80 区间基值 |
| contraction（多字符序列 → 单一 CE） | TR10 §4.2 | `MY_CONTRACTION` 前缀 trie + `contraction_flags` 位图预筛 |
| 排序键与级别分隔 | TR10 §5（Sort Key） | `my_strnxfrm_uca_900` 输出的 16 位大端权重流，级间 0x0000 |
| 语言定制（tailoring）规则语言 | ICU / CLDR 的 LDML collation rules（`& B < C` 语法） | `cs->tailoring` 规则串 → `my_coll_rule_parse` → `apply_one_rule` 改写权重页副本 |
| 语言排序数据源 | CLDR / `strings/lang_data/*.txt` | `zh_hans.txt` 等 → 脚本生成 → `uca900_zh_data.h` |

注意 MySQL **没有链接 ICU 库**——它借用了 ICU 的 tailoring **规则语言**并自己实现解析，权重表则是从 DUCET/CLDR 数据编译出来的静态大表。

- **UCA（Unicode Collation Algorithm）**：Unicode TR10。四级权重模型（primary 字母本体 / secondary 重音 / tertiary 大小写 / quaternary 其他）直接体现在 `uca_scanner_900` 的 `weight_lv` 分级循环里。
- **DUCET（Default Unicode Collation Element Table）**：`uca900_data.h` 的权重表就是 DUCET 的编译产物——DUCET 里每个字符一个或多个 **Collation Element（CE）**，形如 `[.1C47.0020.0002]` 的三元组（primary.secondary.tertiary），'á' 这种组合字符有多个 CE（分解成 a + 重音符）；**implicit weight 规则**（DUCET 未赋权的码点按码点派发权重）实现于 `next_implicit`——TR10 的 7.1.3 节。
- **contraction（压缩组合）**：TR10 的多字符序列 → 单一 CE 的机制，实现为前缀 trie（`MY_CONTRACTION`）。
- **ICU**：`cs->tailoring` 字段存的就是 ICU collation customization 规则串（如 `de_pb_cldr_30`），`my_coll_rule_parse` 解析后逐条 `apply_one_rule` 改写权重页副本——MySQL 没有链接 ICU 库，而是**把 ICU 的 tailoring 规则语言借来，自己实现解析与应用**。

### 算法与数据结构

核心数据结构是 **UCA 900 权重页**（每页 4×256 个 uint16）：

```
page (uint16[1024]) 布局：
┌───────────────────────────────────────────────┐
│ [0..255]    该页 256 个字符各自的 CE 个数        │  ← UCA900_NUM_OF_CE
│ [256..511]  level 0（primary）权重 ×256          │
│ [512..767]  level 1（secondary）权重 ×256        │
│ [768..1023] level 2（tertiary）权重 ×256         │
└───────────────────────────────────────────────┘
   页号 = 码点 >> 8；权重下标 = 码点 & 0xFF
```

- 查一个字符的 primary 权重 = `page[256 + 0*256 + (ch & 0xFF)]`，两级间接、无任何搜索
- 一个字符有多个 CE（组合字符）时，第二个 CE 在 `+3*256` 偏移处（`UCA900_DISTANCE_BETWEEN_WEIGHTS`）
- 复杂度：比较 O(n) 个字符 × 每字符 O(1) 查表 × 3 级；contraction 额外一次 trie 下钻

### 他库对比与演进动机

| | MySQL 8.0 | PostgreSQL | SQL Server | MariaDB |
|---|---|---|---|---|
| 默认 | utf8mb4_0900_ai_ci（NO PAD） | 依赖 OS locale（glibc）或 ICU | 每库一个 collation | 同 MySQL 5.7 路线 |
| 实现 | 自研权重表 + 扫描器（不链 ICU） | 调系统库 | 自带权重库 | 同 MySQL |
| 排序稳定性 | 同一版本内跨平台一致 | glibc 下跨平台可能不同 | 稳定 | 同 MySQL |

演进动机：MySQL 一直**不依赖系统 locale**（因为数据库排序必须跨平台一致——两台不同 glibc 的服务器比较结果必须相同），所以自研权重表。8.0 引入 0900 系列是向 UCA 新版本对齐（权重表更新到 9.0.0、NO PAD 对齐 ICU 惯例），同时借机提供 ai/as/ja_ks 这些分级比较能力——这是老 `_general_ci`（"猜出来的"简化权重）和 `_unicode_ci`（UCA 4.0 折叠权重）都做不到的。

---

## 核心实现

### 主链路

一条 `SELECT ... ORDER BY name` 从解析到索引比较的全路径：

```
SQL 文本（客户端字节流）
  → Protocol_text 按 character_set_client 解码
  → parser（ident_map 按字符集判定标识符字符）
  → Item 表达式：collation 由 DTCollation::aggregate 推导（EXPLICIT > 列 > 连接默认）
  → filesort / 索引比较：strnxfrm 生成权重键（16 位大端权重流）
  → InnoDB 索引内比较：dtype prtype 里的 collation id
      → get_charset(id) 查注册表
      → 回调 server 层 strnncollsp（逐级权重比较）
  → 结果集按 character_set_results 转码回客户端
```

### 字符集注册表与 ID

#### CHARSET_INFO：一个结构挂两套函数表

`CHARSET_INFO` 同时服务"字符处理"与"排序比较"两个职责，字段分四组：

```c
struct CHARSET_INFO {
  uint number;              /* collation id（全局唯一） */
  uint primary_number;      /* 本 charset 默认 collation 的 id */
  uint binary_number;       /* 本 charset binary collation 的 id */
  uint state;               /* MY_CS_* 位掩码 */
  const char *csname;       /* 字符集名，如 "utf8mb4" */
  const char *m_coll_name;  /* collation 名 */
  const char *comment;
  const char *tailoring;    /* ICU/CLDR 语言定制规则串（如 de_pb_cldr_30） */
  struct Coll_param *coll_param;  /* zh/ja 特殊参数 */

  /* 单字节 256 项查表组（latin1/simple 系用） */
  const uchar *ctype;       /* 字符类别位掩码表 */
  const uchar *to_lower; const uchar *to_upper; const uchar *sort_order;

  /* Unicode 数据组（utf8mb4/ucs2 系用） */
  struct MY_UCA_INFO *uca;  /* 权重数据（my_uca_v400 / my_uca_v900） */
  const uint16 *tab_to_uni; const MY_UNI_IDX *tab_from_uni;  /* 8bit<->Unicode 映射 */
  const MY_UNICASE_INFO *caseinfo;  /* 大小写二级页表 */

  uint strxfrm_multiply;    /* strnxfrmlen 乘数 */
  uint mbminlen; uint mbmaxlen; uint mbmaxlenlen;  /* 1/4/1 for utf8mb4 */
  my_wc_t min_sort_char; my_wc_t max_sort_char;    /* LIKE 优化用 */
  uchar pad_char;
  uchar levels_for_compare;  /* UCA900 用：ai_ci=1, as_ci=2, as_cs=3, ja_ks=4 */

  MY_CHARSET_HANDLER *cset;  /* 字符集函数表 */
  MY_COLLATION_HANDLER *coll;/* collation 函数表 */
  enum Pad_attribute pad_attribute;  /* PAD_SPACE / NO_PAD */
};
```

`cset` 函数表（`MY_CHARSET_HANDLER`）是"字符学"：`ismbchar`（是否多字节字符头）、`mbcharlen`（这个字符几字节）、`numchars`/`charpos`（字符数↔字节偏移）、`well_formed_len`（合法序列长度，安全截断用）、`mb_wc`/`wc_mb`（与 Unicode 互转）、`ctype`（字符类别）、`caseup`/`casedn`、`numcells`（显示宽度）。

`coll` 函数表（`MY_COLLATION_HANDLER`）是"比较学"：

```c
int (*strnncoll)(..., bool is_prefix);  /* 比较，is_prefix 表示后者是前缀 */
int (*strnncollsp)(..., bool diff_if_only_endspace_difference);
int (*strnxfrm)(...);      /* 生成排序键 */
size_t (*strnxfrmlen)(...);/* 预估排序键长度 */
int (*like_range)(...);    /* LIKE 模式的最小/最大键 */
int (*wildcmp)(...);       /* LIKE 通配符匹配 */
uint (*hash_sort)(...);    /* PARTITION KEY 哈希（格式不可变） */
```

**★ 全文 17 位状态位**（⚠️ **不是常见说法里的 1,2,4,8,16…且 `MY_CS_CTYPE_TABLE` / `MY_CS_TOUPPER` / `MY_CS_TOSORT` / `MY_CS_CONFIG` / `MY_CS_INDEX` / `MY_CS_CUSTOM` 在 8.0 已全部删除**；后两个改名成 `*_UNUSED`）：

```cpp
/* CHARSET_INFO::state flags */
static constexpr uint32_t MY_CHARSET_UNDEFINED     = 0;        // for unit testing
static constexpr uint32_t MY_CS_COMPILED           = 1 << 0;   // 0x1     编译型
static constexpr uint32_t MY_CS_CONFIG_UNUSED      = 1 << 1;   // 0x2     旧 MY_CS_CONFIG
static constexpr uint32_t MY_CS_INDEX_UNUSED       = 1 << 2;   // 0x4     旧 MY_CS_INDEX
static constexpr uint32_t MY_CS_LOADED             = 1 << 3;   // 0x8     表数据已在内存
static constexpr uint32_t MY_CS_BINSORT            = 1 << 4;   // 0x10    binary 排序
static constexpr uint32_t MY_CS_PRIMARY            = 1 << 5;   // 0x20    该 charset 的默认 collation
static constexpr uint32_t MY_CS_STRNXFRM           = 1 << 6;   // 0x40    ★ 置位表示"sort_order 查表 ≠ strnxfrm"
static constexpr uint32_t MY_CS_UNICODE            = 1 << 7;   // 0x80    BMP Unicode
static constexpr uint32_t MY_CS_READY              = 1 << 8;   // 0x100   init() 已跑完
static constexpr uint32_t MY_CS_AVAILABLE          = 1 << 9;   // 0x200   编译型 或 已加载
static constexpr uint32_t MY_CS_CSSORT             = 1 << 10;  // 0x400   大小写敏感排序（A<a<B）
static constexpr uint32_t MY_CS_HIDDEN             = 1 << 11;  // 0x800   ★ SHOW 里隐藏
static constexpr uint32_t MY_CS_PUREASCII          = 1 << 12;  // 0x1000
static constexpr uint32_t MY_CS_NONASCII           = 1 << 13;  // 0x2000  ★ 非 ASCII 兼容
static constexpr uint32_t MY_CS_UNICODE_SUPPLEMENT = 1 << 14;  // 0x4000  含非 BMP
static constexpr uint32_t MY_CS_LOWER_SORT         = 1 << 15;  // 0x8000
```

> ★★ **`MY_CS_STRNXFRM` 的注释值得逐字读**：*"if \_not\_ set, sort_order will give same result as strnxfrm -- all new collations should have this flag set, do not check it in new code"*。即这是**历史包袱标记**（老 latin1 的 sort_order 查表与 strnxfrm 等价），新代码不该判它。
> ⚠️ **不存在 `my_charset_is_compiled` 宏**——全代码直接写 `(cs->state & MY_CS_COMPILED)`。只有两个判宏：`my_binary_compare(s)` = `state & MY_CS_BINSORT`、`use_strnxfrm(s)` = `state & MY_CS_STRNXFRM`。
> ★ `my_charset_is_ascii_based(cs)` 是**反着判的**：`(cs->state & MY_CS_NONASCII) ? false : true`。

**★ 两套函数表的完整成员**（`cset` 26 个、`coll` 13 个）：

```cpp
typedef struct MY_CHARSET_HANDLER {
  bool (*init)(CHARSET_INFO *, MY_CHARSET_LOADER *loader);
  uint (*ismbchar)(const CHARSET_INFO *, const char *, const char *);
  uint (*mbcharlen)(const CHARSET_INFO *, uint c);
  size_t (*numchars)(const CHARSET_INFO *, const char *b, const char *e);
  size_t (*charpos)(const CHARSET_INFO *, const char *b, const char *e, size_t pos);
  size_t (*well_formed_len)(const CHARSET_INFO *, const char *b, const char *e,
                            size_t nchars, int *error);
  size_t (*lengthsp)(const CHARSET_INFO *, const char *ptr, size_t length);
  size_t (*numcells)(const CHARSET_INFO *, const char *b, const char *e);
  my_charset_conv_mb_wc mb_wc;      /* 多字节 → Unicode 码点 */
  my_charset_conv_wc_mb wc_mb;      /* Unicode 码点 → 多字节 */
  int (*ctype)(const CHARSET_INFO *cs, int *ctype, const uchar *s, const uchar *e);
  size_t (*caseup_str)(const CHARSET_INFO *, char *);
  size_t (*casedn_str)(const CHARSET_INFO *, char *);
  my_charset_conv_case caseup;
  my_charset_conv_case casedn;
  size_t (*snprintf)(...); size_t (*long10_to_str)(...); size_t (*longlong10_to_str)(...);
  void (*fill)(const CHARSET_INFO *, char *to, size_t len, int fill);
  long (*strntol)(...); ulong (*strntoul)(...);
  longlong (*strntoll)(...); ulonglong (*strntoull)(...);
  double (*strntod)(...); longlong (*strtoll10)(...);
  ulonglong (*strntoull10rnd)(...);
  size_t (*scan)(const CHARSET_INFO *, const char *b, const char *e, int sq);
} MY_CHARSET_HANDLER;
```

> ⚠️ **`copy_fix` 在 8.0 已不存在**，其职责由 `well_formed_len` + `lengthsp` 分担。

```cpp
typedef struct MY_COLLATION_HANDLER {
  bool (*init)(CHARSET_INFO *, MY_CHARSET_LOADER *);
  void (*uninit)(CHARSET_INFO *);          /* ★ 只有 UCA 系真正填了（my_coll_uninit_uca） */
  int (*strnncoll)(const CHARSET_INFO *, const uchar *, size_t, const uchar *, size_t, bool);
  int (*strnncollsp)(const CHARSET_INFO *, const uchar *, size_t, const uchar *, size_t);
  size_t (*strnxfrm)(const CHARSET_INFO *, uchar *dst, size_t dstlen,
                     uint num_codepoints, const uchar *src, size_t srclen, uint flags);
  size_t (*strnxfrmlen)(const CHARSET_INFO *, size_t num_bytes);
  bool (*like_range)(...);
  int (*wildcmp)(...);
  int (*strcasecmp)(const CHARSET_INFO *, const char *, const char *);
  uint (*strstr)(... my_match_t *match, uint nmatch);   /* ★★ "instr" 的声明真名是 strstr！ */
  void (*hash_sort)(const CHARSET_INFO *cs, const uchar *key, size_t len,
                    uint64 *nr1, uint64 *nr2);
  bool (*propagate)(const CHARSET_INFO *cs, const uchar *str, size_t len);
} MY_COLLATION_HANDLER;
```

> ⚠️ **函数表里叫 `strstr` 不叫 `instr`**——实现函数名才叫 `my_instr_simple` / `my_instr_mb`。
> ★ `hash_sort` 的 Doxygen 注释明确要求"**跨版本不可变**"：*"This hash is used for hash-based partitioning ... it needs to be unchanged across releases, so that the on-disk format does not change."*

**★ 代表性 handler 实例**（★ 揭示"binary 与 `*_bin` 是两套东西"）：

```cpp
/* strings/ctype-bin.cc:511 —— my_charset_bin（number=63） */
CHARSET_INFO my_charset_bin = {
    63, 0, 0,                                     /* number / primary / binary */
    MY_CS_COMPILED | MY_CS_BINSORT | MY_CS_PRIMARY,
    "binary", "binary", "Binary pseudo charset",
    nullptr, nullptr,                             /* tailoring / coll_param */
    ctype_bin, bin_char_array, bin_char_array, nullptr,
    /* ★ sort_order = nullptr：binary 不查权重表 */
    nullptr, nullptr, nullptr, &my_unicase_default, nullptr, nullptr,
    1, 1, 1,                                      /* strxfrm/caseup/casedn multiply */
    1, 1, 1,                                      /* mbminlen / mbmaxlen / mbmaxlenlen */
    0, 255, 0,                                    /* min_sort_char / max_sort_char / pad_char */
    false, 1,
    &my_charset_handler, &my_collation_binary_handler,   /* ★ NO PAD 版 */
    NO_PAD};
```

```cpp
/* strings/ctype-latin1.cc:368 —— my_charset_latin1（number=8） */
CHARSET_INFO my_charset_latin1 = {
    8, 0, 0,
    MY_CS_COMPILED | MY_CS_PRIMARY,               /* ★★ 注意：没有 MY_CS_STRNXFRM */
    "latin1", "latin1_swedish_ci", "cp1252 West European",
    nullptr, nullptr,
    ctype_latin1, to_lower_latin1, to_upper_latin1, sort_order_latin1,
    nullptr, cs_to_uni, nullptr, &my_unicase_default, nullptr, nullptr,
    1, 1, 1, 1, 1, 1, 0, 255, ' ', false, 1,
    &my_charset_handler, &my_collation_8bit_simple_ci_handler,
    PAD_SPACE};
```

```cpp
/* strings/ctype-uca.cc:9564 —— 8.0 默认（number=255） */
#define MY_CS_UTF8MB4_UCA_FLAGS \
  (MY_CS_COMPILED | MY_CS_STRNXFRM | MY_CS_UNICODE | MY_CS_UNICODE_SUPPLEMENT)

CHARSET_INFO my_charset_utf8mb4_0900_ai_ci = {
    255, 0, 0,
    MY_CS_UTF8MB4_UCA_FLAGS | MY_CS_PRIMARY,
    "utf8mb4", "utf8mb4_0900_ai_ci", "UTF-8 Unicode",
    nullptr, nullptr,
    ctype_utf8,
    nullptr, nullptr, nullptr,     /* ★★ to_lower/to_upper/sort_order 全 NULL：
                                         大小写与排序都走 uca 指针 */
    &my_uca_v900,                  /* ★ UCA 9.0.0 权重表 */
    nullptr, nullptr, &my_unicase_unicode900, nullptr, nullptr,
    0, 1, 1,                       /* strxfrm_multiply=0：运行时由 my_coll_init_uca 算 */
    1, 4, 1,                       /* mbminlen=1 / mbmaxlen=4 */
    9, 0x10FFFF, ' ', false, 1,    /* ★ max_sort_char = 0x10FFFF（含增补平面） */
    &my_charset_utf8mb4_handler, &my_collation_uca_900_handler,
    NO_PAD};                       /* ★★ 0900 系列全部 NO PAD */
```

> ★★ **`my_charset_bin` 用 `my_collation_binary_handler`（NO PAD），而 `latin1_bin` 之类 8bit `_bin` 用 `my_collation_8bit_bin_handler`（PAD SPACE）**——这就是"binary 伪字符集"与"`xxx_bin` 排序规则"的根本差异。`my_charset_bin` 的 `lengthsp` 是 `my_lengthsp_binary`（**不做尾部空格剥离**），`cset` 的 Doxygen 注释专门点名了这个例外。

#### MY_CS_* 状态位

##### 三套排序数据：sort_order / uca(v400) / uca(v900)

`CHARSET_INFO` 里挂着互斥的排序数据来源，用哪种决定了整个比较算法：

| 字段 | 简单字符集（latin1/gb18030 等） | UCA 字符集（utf8mb4_0900_ai_ci 等） |
|---|---|---|
| `sort_order` | ✅ 指向 256 字节数组（单字节查表） | ❌ nullptr |
| `uca`（v400） | ❌ | ✅ 指向 UCA 4.0 权重表 |
| `uca`（v900） | ❌ | ✅ 指向 UCA 9.0.0 权重表 |

**为什么 `utf8mb4_0900_ai_ci` 不能用 `sort_order`**（四条理由）：① 字符范围 U+0000-U+10FFFF（110 万+），256 字节数组根本装不下；② UTF-8 是变长编码（1-4 字节），无法用"字节值"直接索引；③ 需要支持多级权重、contraction、重音等复杂规则；④ 必须走 UCA 算法。

`strxfrm_multiply` 是"排序键长度相对原串的膨胀倍数"——多数字符集是 1，但像德语 collation 里 `Ä → AE`（一个字符展开成两个字符的权重）就需要更大的倍数（如 2）。

```c
MY_CS_COMPILED = 1<<0    /* 编译进 mysqld */
MY_CS_LOADED   = 1<<3    /* 已从 XML 加载 */
MY_CS_BINSORT  = 1<<4    /* 二进制排序（如 *_bin） */
MY_CS_PRIMARY  = 1<<5    /* 是 charset 的默认 collation */
MY_CS_STRNXFRM = 1<<6    /* 未设置表示 sort_order 表即可替代 strnxfrm */
MY_CS_UNICODE  = 1<<7    /* BMP Unicode */
MY_CS_CSSORT   = 1<<10   /* 大小写敏感排序（A<a<B），供 JDBC isCaseSensitive */
MY_CS_PUREASCII= 1<<12   /* 纯 ASCII */
MY_CS_NONASCII = 1<<13   /* 非 ASCII 兼容（转换走慢速 mb_wc 路径） */
MY_CS_UNICODE_SUPPLEMENT = 1<<14 /* 支持非 BMP */
```

`MY_CS_PRIMARY`/`MY_CS_BINSORT` 在注册时按 `number == primary_number/binary_number` 自动置位——不用手写。

#### 注册表：id 无编码公式，直接下标

与直觉不同，8.0.39 的 collation id **没有"高 8 位 charset、低 8 位序号"的编码公式**——id 是 `share/charsets/Index.xml` 里全局单调分配的编号（注释明说 "keep records sorted by collation number"），注册表 `all_charsets[id]` 直接下标索引：

```c
static void add_compiled_collation(CHARSET_INFO *cs) {
  all_charsets[cs->number] = cs;  // id 即数组下标
  ...
}
```

内置 id 采样：latin1_swedish_ci=8、utf8mb3_general_ci=33、utf8mb4_general_ci=45、utf8mb4_bin=46、binary=63、utf8mb4_unicode_ci=224、utf8mb4_0900_ai_ci=255、utf8mb4_0900_as_ci=305、utf8mb4_0900_bin=309。

初始化顺序：`init_compiled_charsets()`（逐个 `add_compiled_collation(&my_charset_*)`，顺序即 charset-def.cc 里的声明序）→ 读 Index.xml 补充动态加载项 → 每个 collation 首次被 `get_charset(id)` 取用时锁下跑 `cset->init/coll->init`（UCA 的 tailoring 解析就发生在 init 里）。

**utf8 别名机制**：8.0 里 `utf8` 是 `utf8mb3` 的别名——`get_charset_name_alias("utf8")` 返回 `"utf8mb3"`，`get_collation_name_alias` 做 `utf8mb3_*`↔`utf8_*` 双向互换。所以 `CREATE TABLE t (c CHAR(1) CHARACTER SET utf8)` 实际拿到的是 utf8mb3。

**★ 注册表的定义与写入点**：

```cpp
/* include/my_sys.h:261 */
#define MY_ALL_CHARSETS_SIZE 2048
extern MYSQL_PLUGIN_IMPORT CHARSET_INFO *all_charsets[MY_ALL_CHARSETS_SIZE];
extern CHARSET_INFO compiled_charsets[];

/* mysys/charset.cc:410 */
CHARSET_INFO *all_charsets[MY_ALL_CHARSETS_SIZE] = {nullptr};
CHARSET_INFO *default_charset_info = &my_charset_latin1;   /* ★ 初值是 latin1 */

void add_compiled_collation(CHARSET_INFO *cs) {
  assert(cs->number < array_elements(all_charsets));   /* ★ 唯一越界保护 */
  all_charsets[cs->number] = cs;
  map_coll_name_to_number(cs->m_coll_name, cs->number);
  map_cs_name_to_number(cs->csname, cs->number, cs->state);
  cs->state |= MY_CS_AVAILABLE;
}
```

> ⚠️ **`all_charsets` 是指针数组不是结构体数组**；且同一 `number` 同时被当成"排序规则 id"和"字符集的 primary collation id"复用。

**★ 名字表：三个 `unordered_map` 的 key 全部先转小写**（⚠️ 真名不是 `charsets_by_name` 之类）：

```cpp
std::unordered_map<std::string, int> *coll_name_num_map = nullptr;
std::unordered_map<std::string, int> *cs_name_pri_num_map = nullptr;   /* 只收 PRIMARY */
std::unordered_map<std::string, int> *cs_name_bin_num_map = nullptr;   /* 只收 BINSORT */
#define MY_CS_BUFFER_SIZE (MY_CS_NAME_SIZE * 8)

static void map_coll_name_to_number(const char *name, int num) {
  char lower_case_name[MY_CS_BUFFER_SIZE] = {0};
  size_t len = std::min(strlen(name), sizeof(lower_case_name) - 2);
  memcpy(lower_case_name, name, len);
  lower_case_name[len] = '\0';
  my_casedn_str(&my_charset_latin1, lower_case_name);   /* ★★ 用 latin1 转小写 */
  (*coll_name_num_map)[lower_case_name] = num;
}

static void map_cs_name_to_number(const char *name, int num, int state) {
  ... 同样转小写 ...
  if ((state & MY_CS_PRIMARY)) (*cs_name_pri_num_map)[lower_case_name] = num;
  if ((state & MY_CS_BINSORT)) (*cs_name_bin_num_map)[lower_case_name] = num;
}
```

> ★★ **"同一字符集名 + 不同 flag"能取到不同 collation id 的机制就在 `cs_name_pri_num_map` / `cs_name_bin_num_map` 的分收**。`get_charset_number(name, cs_flags)` 的 `cs_flags` **只能**传 `MY_CS_PRIMARY` 或 `MY_CS_BINSORT`，其余走末尾的 `assert(false)`。
> ⚠️ **`get_charset_by_name()` 名不副实**：它其实是"按**排序规则名**查"（内部走 `get_collation_number`）；按字符集名查的真名是 `get_charset_by_csname()`。
> ⚠️ **不存在 `my_charset_get_number_by_name()`** → `get_collation_number()` / `get_charset_number(name, cs_flags)`。

**★ `init_compiled_charsets()` 里一个刻意的顺序技巧**（`charset-def.cc:349`）：

```cpp
  // utf8mb4 is the only character set with more than two binary collations. For
  // backward compatibility, we want the deprecated BINARY type attribute to use
  // utf8mb4_bin, and not the newer utf8mb4_0900_bin collation, for the utf8mb4
  // character set. That is, the following column definition should result in a
  // column with utf8mb4_bin collation:
  //
  //    col_name VARCHAR(10) CHARSET utf8mb4 BINARY
  //
  // Since add_compiled_collation() records the last binary collation added for
  // a given character set as the binary collation of that character set (stored
  // in cs_name_bin_num_map), we add utf8mb4_bin after utf8mb4_0900_bin to make
  // it the preferred binary collation of utf8mb4.
  add_compiled_collation(&my_charset_utf8mb4_0900_bin);
  add_compiled_collation(&my_charset_utf8mb4_bin);
```

> ★★ **`cs_name_bin_num_map` 记录的是"最后一个" BINSORT**，所以必须**先加 0900_bin 再加 utf8mb4_bin**，才能让 `CHARSET utf8mb4 BINARY` 得到 `utf8mb4_bin`。这是"注册顺序影响语义"的罕见例子。
> ⚠️ **`my_charset_filename`（id 17）没有被 `add_compiled_collation` 注册**——它是 `MY_CS_HIDDEN`，不在 SHOW 列表里。

**★ `MY_CHARSET_LOADER::add_collation()` 的三大分支**（★ 这是 `MY_CS_LOADED` / `AVAILABLE` / `CSSORT` / `PUREASCII` / `NONASCII` 的**唯一自动推导点**）：

```cpp
int MY_CHARSET_LOADER::add_collation(CHARSET_INFO *cs) {
  if (cs->m_coll_name && (cs->number || (cs->number = get_collation_number_internal(cs->m_coll_name))) &&
      cs->number < array_elements(all_charsets)) {
    if (!all_charsets[cs->number]) {
      all_charsets[cs->number] = (CHARSET_INFO *)my_once_alloc(sizeof(CHARSET_INFO), MYF(0));
      memset(all_charsets[cs->number], 0, sizeof(CHARSET_INFO));
    } else if (all_charsets[cs->number]->state & MY_CS_COMPILED) {
      // Disallow overwriting compiled character sets
      clear_cs_info(cs);
      return MY_XML_OK;                     /* ★★ 编译型不可被 XML 覆盖，直接忽略 */
    }

    if (cs->primary_number == cs->number) cs->state |= MY_CS_PRIMARY;
    if (cs->binary_number == cs->number)   cs->state |= MY_CS_BINSORT;

    all_charsets[cs->number]->state |= cs->state;
    map_coll_name_to_number(cs->m_coll_name, cs->number);
    map_cs_name_to_number(cs->csname, cs->number, cs->state);

    if (!(all_charsets[cs->number]->state & MY_CS_COMPILED)) {
      CHARSET_INFO *newcs = all_charsets[cs->number];
      if (cs_copy_data(all_charsets[cs->number], cs)) return MY_XML_ERROR;
      newcs->caseup_multiply = newcs->casedn_multiply = 1;
      newcs->levels_for_compare = 1;

      if (!strcmp(cs->csname, "ucs2")) {
        copy_uca_collation(newcs, &my_charset_ucs2_unicode_ci);
        newcs->state |= MY_CS_AVAILABLE | MY_CS_LOADED | MY_CS_NONASCII;
      } else if (!strcmp(cs->csname, "utf8") || !strcmp(cs->csname, "utf8mb3")) {
        copy_uca_collation(newcs, &my_charset_utf8mb3_unicode_ci);
        newcs->ctype = my_charset_utf8mb3_unicode_ci.ctype;
        if (init_state_maps(newcs)) return MY_XML_ERROR;   /* ★ 词法器用的状态图 */
      } else if (!strcmp(cs->csname, "utf8mb4")) {
        copy_uca_collation(newcs, &my_charset_utf8mb4_unicode_ci);
        newcs->ctype = my_charset_utf8mb4_unicode_ci.ctype;
        newcs->state |= MY_CS_AVAILABLE | MY_CS_LOADED;
      } else if (!strcmp(cs->csname, "utf16")) { ... }
      else if (!strcmp(cs->csname, "utf32"))  { ... }
      else {
        const uchar *sort_order = all_charsets[cs->number]->sort_order;
        simple_cs_init_functions(all_charsets[cs->number]);
        newcs->mbminlen = 1; newcs->mbmaxlen = 1;
        if (simple_cs_is_full(all_charsets[cs->number]))
          all_charsets[cs->number]->state |= MY_CS_LOADED;
        all_charsets[cs->number]->state |= MY_CS_AVAILABLE;

        /* Check if case sensitive sort order: A < a < B.
           We need MY_CS_FLAG for regex library, and for case sensitivity flag
           for 5.0 client protocol, to support isCaseSensitive() in JDBC driver */
        if (sort_order && sort_order['A'] < sort_order['a'] && sort_order['a'] < sort_order['B'])
          all_charsets[cs->number]->state |= MY_CS_CSSORT;      /* ★★ 靠 sort_order 三点比较推导 */

        if (my_charset_is_8bit_pure_ascii(all_charsets[cs->number]))
          all_charsets[cs->number]->state |= MY_CS_PUREASCII;
        if (!my_charset_is_ascii_compatible(cs))
          all_charsets[cs->number]->state |= MY_CS_NONASCII;
      }
    }
    clear_cs_info(cs);
  }
  return MY_XML_OK;
}
```

```cpp
static void simple_cs_init_functions(CHARSET_INFO *cs) {
  if (cs->state & MY_CS_BINSORT) cs->coll = &my_collation_8bit_bin_handler;
  else                           cs->coll = &my_collation_8bit_simple_ci_handler;
  cs->cset = &my_charset_8bit_handler;      /* ★ 8bit 共用一套 cset */
}

static bool simple_cs_is_full(CHARSET_INFO *cs) {
  return ((cs->csname && cs->tab_to_uni && cs->ctype && cs->to_upper && cs->to_lower) &&
          (cs->number && cs->m_coll_name && (cs->sort_order || (cs->state & MY_CS_BINSORT))));
}

static void copy_uca_collation(CHARSET_INFO *to, CHARSET_INFO *from) {
  to->cset = from->cset; to->coll = from->coll;
  to->strxfrm_multiply = from->strxfrm_multiply;
  to->min_sort_char = from->min_sort_char; to->max_sort_char = from->max_sort_char;
  to->mbminlen = from->mbminlen; to->mbmaxlen = from->mbmaxlen;
  to->caseup_multiply = from->caseup_multiply; to->casedn_multiply = from->casedn_multiply;
  to->state |= MY_CS_AVAILABLE | MY_CS_LOADED | MY_CS_STRNXFRM | MY_CS_UNICODE;
}
```

> ⚠️ **`my_charset_init()` 不存在**（`my_init()` 里也没有字符集初始化）——真名是 `init_compiled_charsets()`（`mysys/charset-def.cc:271`）＋ static 的 `init_available_charsets()`（`mysys/charset.cc:423`）。后者**只能**经 `std::call_once(charsets_initialized, ...)` 触发，共 6 个触发点，全在 `mysys/charset.cc`。
> ★ 服务端唯一"显式"接点是 `sql/mysqld.cc:8751` 安装 `my_charset_error_reporter = charset_error_reporter`（在 `handle_options` **之前**，让早期解析 `--character-set-server` 的告警能打到服务端通道）。
> ★ `charset_uninit()`（`charset.cc:909`）会跑 `coll->uninit` + 删三个 map + `new (&charsets_initialized) std::once_flag`（placement new 复位，便于单元测试重跑）。

#### 初始化时序：编译型 → Index.xml → 懒加载

完整流程（`init_available_charsets`，由 `std::call_once` 保护只跑一次）：

```
init_available_charsets
  ├─ memset(all_charsets)
  ├─ init_compiled_charsets()      // 逐个 add_compiled_collation(&my_charset_*)
  │     └─ all_charsets[cs->number] = cs；置 MY_CS_AVAILABLE
  └─ my_read_charset_file(<dir>/Index.xml)   // 登记动态/未编译项的名字与编号
```

而 **UCA collation 的 tailoring 解析是懒加载的**——发生在 `get_charset(id)` 首次取用时（`get_internal_charset`）：

```c
if (cs->state & MY_CS_READY) return cs;       // 已初始化直接返回
mysql_mutex_lock(&THR_LOCK_charset);
if (!(cs->state & (MY_CS_COMPILED | MY_CS_LOADED))) {
  my_read_charset_file(&loader, cs->csname ".xml", flags);   // 按需加载单个 xml
}
if (cs->state & MY_CS_AVAILABLE && !(cs->state & MY_CS_READY)) {
  if ((cs->cset->init && cs->cset->init(cs, loader)) ||
      (cs->coll->init && cs->coll->init(cs, loader)))
    cs = nullptr;                              // init 失败
  else
    cs->state |= MY_CS_READY;                  // 幂等：只跑一次
}
mysql_mutex_unlock(&THR_LOCK_charset);
```

**为什么必须懒加载**：`coll->init` 对 UCA collation 就是 `my_coll_init_uca` → `create_tailoring`——要解析 CLDR 规则串、为最多 256 页权重表分配并拷贝（每页数百到数千 uint16）、构建 contraction trie（`std::vector<MY_CONTRACTION>`）。这是一次性重活，放在"第一次真正用到这个 collation"时做（典型如 `CREATE TABLE ... COLLATE utf8mb4_ja_0900_as_cs`），避免启动时为几百个 collation 全部付代价；`READY` 位保证幂等。

##### 名字表：不是"排序数组 + 二分"，是三个 unordered_map

名字 → 编号的查找有**三张表**（`mysys/charset.cc`），全是 hash map：

```c
coll_name_num_map   = new std::unordered_map<std::string, int>;  // "utf8mb4_0900_ai_ci" → 255
cs_name_pri_num_map = new std::unordered_map<std::string, int>;  // charset → 其 primary collation
cs_name_bin_num_map = new std::unordered_map<std::string, int>;  // charset → 其 binary collation
```

插入前**统一小写归一**（`my_casedn_str(&my_charset_latin1, name)`）——所以名字查找大小写不敏感。`get_charset_number(csname, MY_CS_PRIMARY)` 查 pri 表、`MY_CS_BINSORT` 查 bin 表，**两者都没传会 `assert(false)`**（只支持这两种 flag）——这正是"为什么 `get_charset_by_csname` 必须带 `MY_CS_PRIMARY` 或 `MY_CS_BINSORT`"的原因。

##### Index.xml 与 `share/charsets/`

`Index.xml`（627 行）是**目录/元数据**，根元素 `<charsets max-id="99">` 下 **37 个 `<charset>`**，每个字符集列出 `family`/`description`/`alias` 与若干 `<collation>`：

```xml
<charset name="latin1">
  <family>Western</family>
  <collation name="latin1_swedish_ci" id="8" order="Finnish, Swedish">
    <flag>primary</flag>
    <flag>compiled</flag>
  </collation>
  <collation name="latin1_german2_ci" id="31" order="German Phonebook" flag="compiled"/>
  <collation name="latin1_bin" id="47" order="Binary"><flag>binary</flag></collation>
</charset>
```

注意 flag **有两种写法**：子元素 `<flag>binary</flag>` 与属性 `flag="compiled"`——解析器都识别。**没有任何 `<rules>` 元素**（5.x 时代"改 XML 加 collation 规则"的路径已死，见扩展点节）。

解析器是 `my_parse_charset_xml`（`strings/ctype.cc`），三个回调（enter/value/leave）配一张 XPath 前缀表 `sec[]` 驱动：

```c
{_CS_CHARSET,    "charsets/charset"},
{_CS_PRIMARY_ID, "charsets/charset/primary-id"},
{_CS_BINARY_ID,  "charsets/charset/binary-id"},
{_CS_COLNAME,    "charsets/charset/collation/name"},
{_CS_ID,         "charsets/charset/collation/id"},
{_CS_FLAG,       "charsets/charset/collation/flag"},
```

一个细节：解析到 `<charset name="utf8">` 时会把名字**就地改写成 `utf8mb3`**——这就是 utf8 别名机制的落点之一。

`share/charsets/` 下共 **25 个 xml**：`Index.xml` + 24 个**单字符集数据文件**（`latin1.xml`/`cp1251.xml`/`koi8r.xml`…），后者装的是该 8bit 字符集的 `ctype`/`to_lower`/`to_upper`/`sort_order`/`to_uni` **表数据**。即"Index.xml 是目录、`<csname>.xml` 是表数据"。

##### `conf_to_src`：构建期代码生成器（不再是自动步骤）

```c
// strings/conf_to_src.cc  main()
sprintf(filename, "%s/%s", argv[1], "Index.xml");  my_read_charset_file(filename);
for (cs = all_charsets; ...; cs++)
  if (cs->number && !(cs->state & MY_CS_COMPILED))
    { sprintf(filename,"%s/%s.xml",argv[1],cs->csname); my_read_charset_file(filename); }
... fprintf(f, "CHARSET_INFO compiled_charsets[] = {\n");   // 生成字面量数组
```

它从 XML 生成 **`strings/ctype-extra.cc`**（含 `compiled_charsets[]`，armscii8/ascii/cp1251/greek/hebrew/koi8r 等 8bit 字符集都在里面）。**关键现状**：8.0.39 里它是 `MYSQL_ADD_EXECUTABLE(... SKIP_INSTALL)`，但 **CMake 里没有 `add_custom_command` 触发重生成**——`ctype-extra.cc` 已入库，改动 XML 后需**手工重跑**该工具。所以"改 `share/charsets/*.xml` 就生效"是错的，必须重新生成 + 重新编译。

##### 三态位与检查顺序（精确化）

| 位 | 值 | 设置点 |
|---|---|---|
| `MY_CS_COMPILED` | bit0 | 静态 `CHARSET_INFO` 字面量里（`ctype-*.cc` / `compiled_charsets[]`） |
| `MY_CS_LOADED` | bit3 | **仅 XML 路径**设（`add_collation` / `simple_cs_is_full`） |
| `MY_CS_AVAILABLE` | bit9 | 编译期 `add_compiled_collation` 设一次；XML 路径也设 |
| `MY_CS_READY` | bit8 | `get_internal_charset` 跑完 `cset->init`/`coll->init` 后设 |

`get_internal_charset` 的三步：

```c
if (cs->state & MY_CS_READY) return cs;                      // ① 已就绪：快路径（无锁）
mysql_mutex_lock(&THR_LOCK_charset);
if (!(cs->state & (MY_CS_COMPILED | MY_CS_LOADED))) {        // ② 既非编译也非加载
  strxmov(get_charsets_dir(buf), cs->csname, ".xml", NullS);
  my_read_charset_file(&loader, buf, flags);                 //    才尝试按需加载
}
if (cs->state & MY_CS_AVAILABLE) {                           // ③ 可用则跑 init
  if ((cs->cset->init && cs->cset->init(...)) || (cs->coll->init && cs->coll->init(...)))
    cs = nullptr;                                            //    init 失败
  else cs->state |= MY_CS_READY;
} else cs = nullptr;
```

**注意"只有 COMPILED 没有 LOADED"完全正常**：`add_compiled_collation` 已置 `MY_CS_AVAILABLE`，第②步跳过、直接进第③步 init。

##### `<csname>.xml` 运行期加载：8.0 已是死路径

第②步那条"按需加载单个 xml"的分支**在 8.0.39 不会被触发**——因为所有字符集要么在 `ctype-*.cc` 编译、要么由 `conf_to_src` 生成进 `compiled_charsets[]`，**全部带 `MY_CS_COMPILED`**。代码保留是为兼容自定义构建（某些 client 只注册部分字符集）。结论：**不要指望"把 xml 放进 sharedir 就能加字符集"**。

##### 注册表查询 API 全集

| API | 未命中 | `MY_WME` 的作用 |
|---|---|---|
| `get_charset_number(name, cs_flags)` | 返回 0 | 无（不报错） |
| `get_collation_number(name)` | 返回 0 | 无 |
| `get_collation_name(n)` | 返回 **`"?"`**（"mimics find_type()"） | 无 |
| `get_charset(n, flags)` | `nullptr` | → `EE_UNKNOWN_CHARSET`，**消息含 Index.xml 路径** |
| `get_charset_by_name(name, flags)` | `nullptr` | → `EE_UNKNOWN_COLLATION` |
| `get_charset_by_csname(name, cs_flags, flags)` | `nullptr` | → `EE_UNKNOWN_CHARSET` |
| `resolve_charset(name, def, out)` | **回落默认 + 返回 true** | 内部用 `MYF(0)` |
| `resolve_collation(name, def, out)` | 同上 | 内部用 `MYF(0)` |

**`MY_WME` 只决定"要不要 `my_error` 写错误信息"，不改变命中/未命中语义**（未命中恒为 NULL/0）。别名在 API 层再补一道：`get_collation_name_alias` 处理 `utf8mb3_`↔`utf8_`，`get_charset_name_alias` 处理 `utf8`→`utf8mb3`。

##### 启动顺序的三个坑

1. **`system_charset_info` 必须"early 赋值"**——`my_getopt` 在完整初始化前就要用它比较选项名（注释："Must be initialized early for comparison of options name"）。
2. **`fs_character_set()` 不能调 `get_charset_by_name`**——它运行在 `THR_LOCK_charset` 初始化**之前**，而 `get_charset_by_name` 会去 lock 该 mutex；所以改成直接读注册表里的 codepage 编号再 `strcmp` 判 cp932。
3. **Index.xml 缺失/损坏是静默降级**——`my_read_charset_file` 的返回值在 `init_available_charsets` 里**未被检查**；解析错误只 `my_printf_error(EE_UNKNOWN_CHARSET, "Error while parsing '%s': %s")` 后继续。因为 8.0 全字符集已编译，后果近乎无感（只是未编译项的名字/注释缺失）。

##### 注册表的下一站：DD 表 → I_S 视图 → SHOW COLLATION

字符集信息不只活在内存里，还被**固化进数据字典**：

```
all_charsets[]  ──(DD 引导)──►  mysql.character_sets / mysql.collations
                                      │
                                      ▼
                            INFORMATION_SCHEMA.CHARACTER_SETS / COLLATIONS（系统视图）
                                      │
                                      ▼
                            SHOW COLLATION → build_show_collation_query（重写成 I_S 查询）
```

**填充是"直接从 `all_charsets[]` 生成"**的（`Collations::populate`）：外层挑 `MY_CS_PRIMARY && MY_CS_AVAILABLE && !MY_CS_HIDDEN` 的字符集，内层挑 `MY_CS_AVAILABLE && my_charset_same` 的 collation，并写 `set_is_compiled(cl->state & MY_CS_COMPILED)`、`sort_length = strxfrm_multiply`、`pad_attribute`；同时**删除已不存在的旧 id**。触发点是 DD 引导（`bootstrapper.cc` 里 `Collations::instance().populate(thd)` / `Character_sets::instance().populate(thd)`）。

所以"加一个 collation 后 I_S 与 `SHOW COLLATION` 自动就有了"——但前提是**先编译进 `all_charsets[]`**。

`SHOW COLLATION` 在 8.0 被**重写成对 I_S 视图的查询**（`dd/info_schema/show.cc` 的 `build_show_collation_query` + `sql_command_flags[SQLCOM_SHOW_COLLATIONS] |= CF_SHOW_USES_SYSTEM_VIEW`）——这也解释了为什么 `SHOW COLLATION` 的输出可以像 I_S 那样加 `WHERE`/`LIKE`。`character_set_system` 是 READ_ONLY 且绑定 `system_charset_info`（标识符固定 utf8mb3，改不了）；`default_collation_for_utf8mb4` 只允许 `utf8mb4_0900_ai_ci` 或 `utf8mb4_general_ci` 两个值。

### 便利 API 层：server 代码怎么调用字符集

`CHARSET_INFO` 的两个函数表（`cset`/`coll`）是**底层接口**，server 代码日常并不直接 `cs->coll->strnncoll(...)`，而是调**便利层**——一批宏与 inline 函数。这一层的现状（8.0 的重大变化）值得单独讲清，否则会照着 5.7 的写法找不存在的 API。

#### ★ 一批旧便利宏在 8.0 已删除

**8.0.39 全仓库 0 命中**（`#define my_xxx` 模式检索）：`my_strncasecmp`、`my_strnncollsp`、`my_strnxfrmlen`、`my_numchars`、`my_charlen`、`my_well_formed_len`、`my_lengthsp`、`my_hash_sort`、`my_mb_ctype`、`my_ismbhead`。调用方一律直接写 `cs->coll->…` / `cs->cset->…`。

最直接的旁证是一句保留在代码里的注释：

```cpp
// sql/debug_sync.cc
// "The tokens are used in my_strcasecmp(). Unfortunately there is no my_strncasecmp()."
```

**留存的便利层**（全是宏或 inline 转发）：

| 便利名 | 形态 | 转发到 |
|---|---|---|
| `my_strcasecmp(s, a, b)` | 宏 | `cs->coll->strcasecmp` |
| `my_strnncoll(...)` | 宏 | `coll->strnncoll(..., false)` |
| `my_strnxfrm(...)` | 宏 | `coll->strnxfrm(...)` |
| `my_like_range(...)` / `my_wildcmp(...)` | 宏（10/8 参） | `coll->like_range` / `coll->wildcmp` |
| `my_charpos(cs, b, e, num)` | 宏 | `cset->charpos` |
| `my_ismbchar` / `my_ismb1st` / `my_mbcharlen` | inline / 宏 | `cset->ismbchar` / `mbcharlen` |
| `my_caseup_str` / `my_casedn_str` / `my_strntol` 族 | 宏 | `cset->*` |
| `my_isalpha` / `my_toupper` 等 | 宏 | **直接查表**（`cs->ctype[+1]` / `cs->to_upper[]`） |
| `my_convert` / `my_charset_same` / `my_charset_is_ascii_based` | 真函数 / inline | 见各节 |

**调用规模**（8.0.39 grep 统计）：`my_strcasecmp` 在 `sql/sql_table.cc` 138 处、`sql_base.cc` 25、`sql_authorization.cc` 15、`client/mysqldump.cc` 31——是 server 侧使用量最大的字符集 API；`my_strnncoll` 覆盖 17 个 sql 文件；而 `my_wildcmp` 只有 3 处、`my_like_range` 仅 1 处（`range_analysis.cc`）、`my_hash_sort` **0 处**（已无宏，全部直接调句柄）。

#### `my_strcasecmp`：用 `to_upper` 而不是 `sort_order`

它是 3 参数、要求 **NUL 结尾**、**大小写不敏感**的比较，实现按 handler 分四种：

```c
// 8bit（server 最常见）
int my_strcasecmp_8bit(const CHARSET_INFO *cs, const char *s, const char *t) {
  const uchar *map = cs->to_upper;      // ★ 用 to_upper 映射表，不是 sort_order
  while (map[(uchar)*s] == map[(uchar)*t++])
    if (!*s++) return 0;
  return ((int)map[(uchar)s[0]] - (int)map[(uchar)t[-1]]);
}
```

| 实现 | 行为差异 |
|---|---|
| 8bit | 逐字节 `to_upper` 比较 |
| 多字节（mb） | 单字节部分用 `to_upper`；多字节部分 `my_ismbchar` 跳过 + **逐字节 memcmp**（不做大小写折叠）；**失配返回 1（不是负值）** |
| `_bin` collation | 退化为 `strcmp`（**大小写敏感**）——注意 `utf8mb4_0900_bin` 也绑这个 |
| **ucs2 / utf16 / utf32** | `my_strcasecmp_mb2_or_mb4` **直接 `assert(0)`**——在这几个字符集上调用是**编程错误**（release 下恒返回 0） |

两个要点：① 返回值是"差值 / 布尔"混合语义，不同 handler 不一致——**只判 `== 0`，不要判正负**；② 它用 **`to_upper` 映射表**而非 `sort_order`（排序权重）——所以"标识符比较"是"大小写折叠后逐字节比"，与 collation 的排序语义无关。这解释了一件事：`system_charset_info` 的 `to_upper` 就是 `to_upper_utf8mb3`，标识符大小写不敏感**只对该表覆盖的字符生效**。

而 **SQL 关键字连这条路都不走**——用的是 `sql_lex_hash.cc` 里独立的 latin1 关键字表（`lex_casecmp`），注释明确"based on the latin1 character set, and is only used when comparing keywords"。

#### `my_charset_same`：只比 charset 名，且没有"is_known"版本

```c
inline bool my_charset_same(const CHARSET_INFO *cs1, const CHARSET_INFO *cs2) {
  assert(0 != strcmp(cs1->csname, "utf8"));   // 入参必须已规范化为 utf8mb3
  assert(0 != strcmp(cs2->csname, "utf8"));
  return ((cs1 == cs2) || !strcmp(cs1->csname, cs2->csname));
}
```

**只比 `csname`（字符集名），不比 collation 名**——所以 `utf8mb3_general_ci` 与 `utf8mb3_bin` 之间 `my_charset_same` 为真，`String::needs_conversion()` 据此判定"无需转换"（同字符集不同排序规则不重编码）。

⚠️ 更正两个易记错的 API：**`my_charset_same_charset` 与 `my_collation_is_known` 在 8.0.39 不存在**。等价做法是 `get_collation_number()`（未命中返回 0）或 `get_charset_by_name(name, MYF(0))`（返回 nullptr）；DDL/别名场景用 `resolve_charset` / `resolve_collation`（未命中回落默认值并返回 true）。

#### `my_wildcmp` 的四个变体：分派靠 handler 静态绑定

不是运行期按 `MY_CS_UNICODE` 判断，而是**每个 collation 在自己的 handler 结构里静态指好 `wildcmp` 字段**：

| 变体 | 实现要点 |
|---|---|
| 8bit | `likeconv = cs->sort_order[...]`——**按排序权重比较**，天然大小写/重音不敏感 |
| mb | `INC_PTR` 用 `my_ismbchar` 跳多字节；**多字节部分必须逐字节 memcmp**（不做折叠），单字节走 `likeconv` |
| mb_bin | 单字节也直接比较（无 `likeconv`）——`utf8mb3_bin` / `utf8mb4_0900_bin` 用它 |
| **unicode** | 两侧都用 `my_tosort_unicode` **折成排序权重再比**；`_` 的处理是 `result = 1`（不参与权重比较，只推进**一个码点**）；被转义的 `\_` 当字面 `_` 走权重比较；递归有 `my_string_stack_guard` 防栈溢出 |

返回值协议：`0` 匹配 / `-1` 不匹配但用过通配 / `1` 不匹配。

**关键：`like_range` 与 `wildcmp` 是两条独立路径**——前者把 pattern 改写成 `[min, max]` 键区间（供索引 range 扫描），**不调用 wildcmp**；后者是逐字符实际匹配。`Item_func_like::val_int` 最终调的是 `my_wildcmp(...)`：索引区间只是"缩小扫描范围"，真值判定仍靠 wildcmp。

#### ctype 便利宏与多字节判定

`my_isalpha(cs, c)` 等宏是**纯 8bit 表查 + 固定 +1 偏移**：

```c
#define my_isalpha(s, c) (((s)->ctype + 1)[(uchar)(c)] & (_MY_U | _MY_L))
```

**两个使用边界**：① 这组宏**只对单字节字符集有意义**——源码注释明确"对 utf8mb3/utf8mb4，只有 < 0x7F 才有意义"（0x80 以上是变长序列的组成部分，单独看无类别意义）；② 字符集若把 0x80-0xFF 全标 `_MY_L`（如 gbk 标记双字节首字节），会让通用单字节逻辑"看似正确"。

多字节字符的真实类别走 `cset->ctype`（多字节版）：`my_mb_ctype_mb` 先 `mb_wc` 取码点，`wc > 0xFFFF` 直接判类别 0，否则查 `my_uni_ctype[wc >> 8].ctype`（缺失回退 `.pctype`）。

**多字节首字节判定有个 GB18030 特例**：`my_ismb1st` 除了看 `my_mbcharlen` 返回值，还要处理"需要两字节才能定长"的情况——`my_mbmaxlenlen == 2 && mbcharlen == 0` 时也算首字节（GB18030 的 4 字节序列首字节要读第二个字节才能判定）。

#### `hash_sort`：已无便利宏，且契约要求"跨版本不可变"

`my_hash_sort(cs, key, len)` 宏**已删除**，全部直接调 `cs->coll->hash_sort(cs, key, len, &nr1, &nr2)`。契约原文（`m_ctype.h`）：

> "…used for hash-based partitioning (PARTITION KEY), so you cannot change it except when writing a new collation; it needs to be unchanged across releases"

**这是"collation 升级不能随便改"的权威出处**——因为哈希值被两条路径持久化依赖：

| 消费方 | 说明 |
|---|---|
| **PARTITION BY KEY** | 分区路由依赖哈希值——改算法 = 数据分到别的分区 |
| **MEMORY 引擎** | 唯一索引/哈希表用 `hash_sort`（`hp_hash.cc`） |
| `Field::hash` | 等值 join 的哈希表；结果截成 32 位（**保 Windows 平台稳定**） |

实现层：`my_hash_sort_simple` 先 `skip_trailing_space()`（**PAD SPACE 语义由它保证**：`'A '` 与 `'A'` 同哈希）再按 `cs->sort_order` 混合（`tmp2 += 3`，FNV 变体）；`my_hash_sort_mb_bin` 用原始字节而非 sort_order。`partition_handler.cc` 里还有一段 `hash_sort` 的"5.1 兼容"注释（`my_charset_bin.coll->hash_sort` / `my_charset_latin1.coll->hash_sort`）——**哈希格式的磁盘兼容约束**留下的痕迹。

### binary：伪字符集与四个 BINARY 概念

`binary` 是字符集体系里的"**空指针**"——它不表示编码，而是表示"**擦除字符集信息**"。源码文件头的注释就说得很直白：erase the charset information。理解它需要先把"BINARY"这个词的**四种含义**拆开，这是使用中最常见的混淆源。

#### `my_charset_bin` 的定义（number = 63）

```c
// strings/ctype-bin.cc
CHARSET_INFO my_charset_bin = {
    63, 0, 0,                                        /* number */
    MY_CS_COMPILED | MY_CS_BINSORT | MY_CS_PRIMARY,  /* state */
    "binary", "binary", "Binary pseudo charset",
    nullptr, nullptr,          /* sort_order / uca 都没有 */
    ctype_bin, bin_char_array, bin_char_array, nullptr, nullptr, nullptr, nullptr,
    &my_unicase_default, nullptr, nullptr,
    1, 1, 1,                   /* mbminlen = mbmaxlen = mbmaxlenlen = 1 */
    0, 255, 0, false, 1,
    &my_charset_handler,          /* cset */
    &my_collation_binary_handler, /* coll */
    NO_PAD};
```

四个值得注意的点：① **`state` 同时含 `MY_CS_BINSORT | MY_CS_PRIMARY`**——它把自己唯一的 collation（`m_coll_name == "binary"`）标成 primary，所以 `SET NAMES binary`、`get_charset_by_csname("binary", MY_CS_PRIMARY)` 都能正常解析（**不会报 `ER_UNKNOWN_CHARACTER_SET`**）；② `sort_order`/`uca`/`tab_to_uni` 全是 `nullptr`——没有排序表和 Unicode 映射；③ `to_lower`/`to_upper` 都指向 `bin_char_array`（**恒等映射**）；④ `pad_char = 0` 且 `pad_attribute = NO_PAD`——**不是空格**。

#### 函数表：全是"逐字节 + 长度为序"

| 函数 | 实现 | 要点 |
|---|---|---|
| `strnncoll` | `memcmp` 前 `min(slen,tlen)` 字节，**相等则以长度差定序** | `return cmp ? cmp : (int)((t_is_prefix ? len : slen) - tlen)` |
| `strnncollsp` | **直接转调 `strnncoll(...,false)`** | ★ **不跳过也不补齐尾空格**（对照 `my_strnncollsp_8bit_bin` 会补 0x20） |
| `strnxfrm` | `my_strnxfrm_8bit_bin_no_pad` = memcpy 恒等 | `strxfrm_multiply = 1` |
| `hash_sort` | `my_hash_sort_bin` 对整串求哈希 | ★ **不剥离尾空格** |
| `caseup`/`casedn` | `my_case_bin` **返回原长度、不改字节** | `strcasecmp = strcmp` |
| `mb_wc`/`wc_mb` | 一字节 = 一个"码点" | `wc_mb` 只在 `wc < 256` 可写，否则 `MY_CS_ILUNI` |
| `ctype` | `ctype_bin` 对 ASCII 字母/数字/空格**仍置位** | 所以 `my_isalpha(binary, 'a')` 为真（数值解析依赖这个） |

**"不补空格"是最本质的一条**：PAD SPACE 语义完全不存在——`'a'` 与 `'a '` 在 binary 下**不相等**（长度差定序），而在 `utf8mb4_bin` 下相等。

#### 四个 BINARY 概念（★ 最易错的是第②个）

| # | 概念 | 落到哪 |
|---|---|---|
| ① | **`binary` 字符集**（id 63） | 字节串语义，见上 |
| ② | **`BINARY` 列属性**（`VARCHAR(10) BINARY`） | ★ **8.0 转成"该字符集的 `_bin` collation"，不是 binary 字符集** |
| ③ | **`BINARY`/`VARBINARY`/`BLOB` 数据类型** | 走 binary 字符集（`&my_charset_bin`） |
| ④ | **`CAST(x AS BINARY)` / `BINARY x` 操作符** | 把 charset 换成 binary 标签（不改字节） |

**② 的源码证据**（这是最常被误解的一条）：

```c
// sql/sql_table.cc —— 建表时 BINCMP_FLAG 被翻译成 _bin collation
if (sql_field->flags & BINCMP_FLAG) {
  // e.g. CREATE TABLE t1 (a CHAR(1) BINARY);
  sql_field->charset = get_charset_by_csname(sql_field->charset->csname,
                                             MY_CS_BINSORT, MYF(0));
  sql_field->flags &= ~BINCMP_FLAG;
}
```

```c
// sql/sql_yacc.yy —— 语法层先告警，再置 force_binary
void warn_about_deprecated_binary(THD *thd) {
  push_deprecated_warn(thd, "BINARY as attribute of a type",
                       "a CHARACTER SET clause with _bin collation");
}
```

即 **`VARCHAR(10) BINARY` ≡ `VARCHAR(10) CHARACTER SET <原 charset> COLLATE <cs>_bin`**（utf8mb4 → `utf8mb4_bin`；latin1 → `latin1_bin`），**绝不是 binary 字符集**，而且 8.0 会给 deprecated 告警。`SHOW CREATE TABLE` 回显的也是 `CHARACTER SET utf8mb4 COLLATE utf8mb4_bin`。

**④ 的实现**——`BINARY x` 与 `CAST(x AS BINARY)` 走同一个 Item：

```c
// sql/sql_yacc.yy
| BINARY_SYM simple_expr %prec NEG
  { push_deprecated_warn(YYTHD, "BINARY expr", "CAST");
    $$ = create_func_cast(YYTHD, @2, $2, ITEM_CAST_CHAR, &my_charset_bin); }
```

`Item_charset_conversion::val_str` 只做 `m_tmp_value.copy(ptr, len, m_from_cs, m_cast_cs)`（因 `needs_conversion` 对目标 binary 返回 false → **字节原样拷贝**）+ `set_charset(m_cast_cs)`；带长度参数时（`CAST(x AS BINARY(N))`）才 `memset(0)` 补到定长。**所以 `BINARY x` 与 `CONVERT(x USING binary)` 等价，都是"重贴标签、不动字节"**——唯一差别是前者可带定长补零。

#### `utf8mb4_bin` ≠ `my_charset_bin`

一个高频混淆：两者名字都有 bin，但层级完全不同。

| | `utf8mb4_bin` | `my_charset_bin` |
|---|---|---|
| 是什么 | utf8mb4 **charset 的一个 collation** | **独立 charset**（id 63） |
| 解码 | 走 utf8mb4 的 `mb_wc`（按**码点**） | 不解码（按**字节**） |
| 尾空格 | **补 0x20**（PAD SPACE） | **不补**（NO_PAD） |
| 合法性 | 要求 well-formed UTF-8 | **允许任意字节** |
| InnoDB 存储 | prtype 里记 utf8mb4 的 collation id | prtype 里记 **63** |
| 函数能力 | 有 UPPER/LOWER（对 Unicode 有效） | UPPER/LOWER **恒等**（无大小写概念） |

三者（顺序/相等/长度）在 ASCII 范围内常常结果相同（字节序 = 码点序），**差异只在尾空格、非法字节、存储合法性**这三处暴露——所以"看起来一样"但语义不同。

#### binary 在转换路径上的行为：短路，但"存储"例外

三处短路，语义**并不一致**，这是容易踩的坑：

```c
// ① well_formed_copy_nchars —— 目标 binary 时直接 memmove，且「不产生 well_formed 错误」
if ((to_cs == &my_charset_bin) || (from_cs == &my_charset_bin) ||
    (to_cs == from_cs) || my_charset_same(from_cs, to_cs)) {
  if (to_cs == &my_charset_bin) { res = min(min(nchars,to_length),from_length);
                                  memmove(to, from, res);
                                  *well_formed_error_pos = nullptr; ... }
```

```c
// ② String::needs_conversion —— 源 binary 且长度按 mbminlen 对齐 → 不转换（裸拷贝）
if (to_cs == nullptr || to_cs == &my_charset_bin || from_cs == to_cs ||
    my_charset_same(from_cs, to_cs) ||
    ((from_cs == &my_charset_bin) && (0 == (*offset = arg_length % to_cs->mbminlen))))
  return false;
```

```c
// ③ needs_conversion_on_storage —— ★ 反过来「强制」bin → 非 bin 必须转换
return (needs_conversion(arg_length, cs_from, cs_to, &offset) ||
        (cs_from == &my_charset_bin && cs_to != &my_charset_bin &&
         (cs_to->mbminlen != cs_to->mbmaxlen ||   // 变长编码
          cs_to->mbminlen > 2 ||                   // 每字符 > 2 字节
          0 != (arg_length % cs_to->mbmaxlen))));
```

**③ 的动机写在头文件注释里**："Unlike needs_conversion it will require conversion on incoming binary data to ensure the data are verified for validity first"——**落库路径更严**：binary 数据写进 utf8mb4 列必须先过转换器校验良构性，非法字节会报 `ER_TRUNCATED_WRONG_VALUE_FOR_FIELD`，而不是静默裸存。

**还有一条独立警告**：`my_convert` 从多字节 → binary 会"**解码再截成单字节**"（`wc_mb_bin` 对 `wc >= 256` 返回 `ILUNI` → 写 `'?'`），**并不保字节**。真正字节透明的只有上面的 `well_formed_copy_nchars` / `String::copy` 短路路径——**单用 `my_convert` 处理 binary 目标是错的**。

#### binary 在 InnoDB 侧：prtype 63 与"不补 0x20"

```c
// storage/innobase/include/data0type.h
constexpr uint32_t DATA_MYSQL_BINARY_CHARSET_COLL = 63;
```

binary 的 VARCHAR → `DATA_BINARY`、定长 STRING → `DATA_FIXBINARY`；prtype 里记 `charset_no = field_charset->number` = 63。**比较时的分叉**：

```c
// rem/rem0cmp.cc cmp_data
case DATA_FIXBINARY:
case DATA_BINARY:
  if (dtype_get_charset_coll(prtype) != DATA_MYSQL_BINARY_CHARSET_COLL) {
    pad = 0x20;      // 普通字符集：短的一方补空格
    break;
  }
  [[fallthrough]];   // 63（binary）：pad = ULINT_UNDEFINED —— 不补空格
case DATA_INT: ...
```

即 **InnoDB 也认 id 63 这个哨兵**——`DATA_BINARY` 只有在 collation 不是 63 时才补 0x20。这保证了 SQL 层与引擎层对 binary "长度定序、不补空格"的语义完全一致。

#### `Field::binary()` 与"无字符语义"的退化

> ★ `Field` 的完整剖析（类型系统 / 值协议 / 存储布局 / 字符集与排序键 / 与 Item·TABLE·handler·DD 四条接缝）见 [`../field.md`](../field.md)。本节只讲**字符集视角**下的 `Field`——"无字符语义"如何退化。

8.0.39 **只有 `Field::binary()`，没有 `Field::is_binary()`**：

```cpp
virtual bool binary() const { return true; }                                  // 基类默认：数值类都是
bool Field_str::binary() const override { return field_charset == &my_charset_bin; }  // 字符串字段按 charset 判
```

凡是"按字节"的语义在 binary 下都**退化为恒等**：`UPPER(bin_col)` 原样返回（`caseup = my_case_bin` 不改字节）；`test_if_important_data` 对 binary **不剥离尾空格**（`if (cs != &my_charset_bin) str += cs->cset->scan(..., MY_SEQ_SPACES)`），所以 binary 列的尾空格被当作重要数据参与截断判定。

### 字符集处理器：ctype 位掩码与多字节边界

##### ctype 数组的 257 项与 +1 机制

`ctype` 数组长度是 **257** 而不是 256——首元素是占位（语义上是 EOF，值为 0），真正的数据从下标 1 开始。所以所有判定宏都要先 `+1`：

```c
#define my_isupper(s, c) (((s)->ctype + 1)[(uchar)(c)] & _MY_U)
#define my_isgraph(s, c)   (((s)->ctype + 1)[(uchar)(c)] & (_MY_PNT | _MY_U | _MY_L | _MY_NMR))
```

`'A'`（0x41=65）查的是 `ctype[66]`，latin1 下该值 129 = 0b10000001，第 0 位 `_MY_U` 置位。**这个 +1 的位置与直觉相反**（是先挪指针再索引，不是索引后 +1），看宏定义时要留意。

##### 多字节字符集的 ctype 只对 ASCII 有效

单字节字符集（latin1/gbk/gb18030…）的 ctype 表覆盖 0x00-0xFF 全部 256 个值；但多字节字符集不同：

- **utf8mb4**：`ctype` 只有 0x00-0x7F（ASCII）有意义，0x80-0xFF 全为 0（那些字节是多字节序列的组成部分，单独看没有类别意义）
- **gbk**：把 0x80-0xFF 统统标记为 `_MY_L`（小写字母）——因为它们在 GBK 里是**双字节字符的首字节**，这样处理能让通用的单字节逻辑不至于误判

多字节字符的真实类别不能靠 `ctype` 表，要走专门的 `my_mb_ctype_mb`：先 `mb_wc` 转成码点，再查 Unicode 属性表 `my_uni_ctype[wc >> 8]`，优先取动态的 `.ctype`、不存在时回退静态的 `.pctype`（且 `wc > 0xFFFF` 直接判为类别 0）。

##### 数字解析：`strntoull10rnd`（`cset` 表里常被忽略的一员）

**一句话说清它在做什么**：它是"**按字符集规则把一段字符串解析成整数**"的函数，挂在 `cs->cset` 上，**每个字符集一套**。名字拆开：`str` → `n`（按长度而非 `\0` 结尾）→ `toull`（转 unsigned long long）→ `10`（十进制）→ `rnd`（rou**nd**ing，带小数时四舍五入）。

**为什么它必须是字符集相关的**（看似与字符集无关，其实不然）：

- **一个字符占几字节由字符集定**——`_8bit` 版本能直接写 `*str - '0'`，因为它假设 1 字节 = 1 字符；`ucs2` / `utf16` 一个字符占 2 字节，必须换实现
- 不同字符集可能有**不同的数字字符**（如阿拉伯-印度数字 `٠١٢٣`）

**实现要点**（`my_strntoull10rnd_8bit`，`strings/ctype-simple.cc:1235`）：

```cpp
/* Skip leading spaces and tabs */
for (; str < end && (*str == ' ' || *str == '\t'); str++) ;
if ((negative = (*str == '-')) || *str == '+') { if (++str == end) goto ret_edom; }

beg = str;
end9 = (str + 9) > end ? end : (str + 9);
/* Accumulate small number into ulong, for performance purposes */
for (ul = 0; str < end9 && (ch = (uchar)(*str - '0')) < 10; str++) {
  ul = ul * 10 + ch;               // ★ 前 9 位用 ulong 累加——快路径
}
if (str >= end) {                  // 无小数点、无指数的小数字
  *endptr = str;
  if (negative) {
    if (unsigned_flag) {
      *error = ul ? MY_ERRNO_ERANGE : 0;   // ★ 负数存进 UNSIGNED 列 → 报范围错
      return 0;
    }
    return (ulonglong)(longlong) - (long)ul;
  }
  ...
```

1. **`ch = (uchar)(*str - '0') < 10`**——判"是否数字字符"只用**一次减法 + 一次无符号比较**，比 `'0' <= c && c <= '9'` 少一次比较（经典小技巧）
2. ★ **前 9 位走 `ulong` 快路径**——源码注释直说 *"for performance purposes"*：≤9 位十进制必不溢出 32 位，用更窄类型累加更快，超过才切 `ulonglong`
3. ★ **负数写进 UNSIGNED 列是直接报 `MY_ERRNO_ERANGE`，不是截断成 0**（注意 `ul ? ... : 0`——`-0` 不算错）
4. **`*endptr` 输出"解析停在哪"**——供上层判断"字符串有没有被完整消费"（`'12abc'` 停在 `a` ⇒ 报截断警告）
5. **`rnd` 是 rounding**：处理小数点（源码里的 `dot` 变量），把 `'12.6'` **四舍五入**成 13 ⇒ `INSERT INTO t(INT 列) VALUES('12.6')` 得到 13，就是它

> **使用现场**：[`data_repr.md`](../../infra/structure/data_repr.md)「★ `Field::store()` 到底做了什么」——`Field_num::get_int()` 就是转调它，再往上由 `Field_long::store()` 调用。

##### 非法字节：well_formed_len 与 sql_mode 的交互

`well_formed_len` 的契约是"**返回最长合法前缀的字节数**"，非法序列本身不计入，并通过 `*error` 告知：

```c
while (pos) {
  mb_len = my_valid_mbcharlen_utf8mb4(cs, b, e);
  if (mb_len <= 0) { *error = (b < e ? 1 : 0); break; }   // 还有字节却解不出 → 非法
  b += mb_len; pos--;
}
return (size_t)(b - b_start);
```

上层的走向（以 `Field_string::store` 为例）：

- **非法序列** → `check_string_copy_error`：`convert_to_printable` + 警告 `ER_TRUNCATED_WRONG_VALUE_FOR_FIELD`
- **合法但超长** → `report_if_important_data`：**strict 模式（且非 `IGNORE`）下报 `ER_DATA_TOO_LONG`、语句失败**；非 strict 则截断 + `WARN_DATA_TRUNCATED`；若只丢尾空格则降级为 NOTE

即"插入非法 UTF-8 是报错还是变 '?'"取决于 sql_mode——这是字符集行为与 SQL 模式的交界点。

单字节系（latin1/simple）的字符类别是一张 256 项位掩码表：

```c
#define _MY_U 01    /* Upper case 大写 */
#define _MY_L 02    /* Lower case 小写 */
#define _MY_NMR 04  /* Numeral 数字 */
#define _MY_SPC 010 /* Spacing 空白 */
#define _MY_PNT 020 /* Punctuation 标点 */
#define _MY_CTR 040 /* Control 控制符 */
#define _MY_B 0100  /* Blank 空格/制表 */
#define _MY_X 0200  /* heXadecimal digit */
```

例：`ctype_latin1` 里 'A'=0x81（_MY_U|_MY_X）、数字=0x84（_MY_NMR|_MY_X）。`my_isalpha/my_isupper` 等宏直接 `ctype[ch+1] & _MY_*` 查表。`to_lower`/`to_upper`/`sort_order` 同是 256 字节表——`latin1_swedish_ci` 的比较就是逐字节查 `sort_order` 再比。

多字节系的关键在 `mb_wc`（字节序列 → Unicode 码点）与 `well_formed_len`（最长合法前缀长度）：

```c
// utf8mb3 的 mb_wc 骨架（ctype-utf8.cc）
static int my_mb_wc_utf8mb3_no_range(my_wc_t *wc, const uchar *s, const uchar *e) {
  if (s >= e) return MY_CS_TOOSMALL;
  uchar c = s[0];
  if (c < 0x80) { *wc = c; return 1; }          // ASCII 快路径
  if (c < 0xc2) return MY_CS_ILSEQ;             // 非法首字节
  if (s + 1 >= e) return MY_CS_TOOSMALL;
  if (!((s[1] ^ 0x80) < 0x40)) return MY_CS_ILSEQ;  // 续字节必须是 10xxxxxx
  if (c < 0xe0) { *wc = ((c & 0x1f) << 6) | (s[1] & 0x3f); return 2; }
  ... // 三字节同样：先验首字节区间、续字节格式，再合成码点
}
```

这个函数是转换链的地基：`my_convert_internal` 逐码点 `mb_wc` → `wc_mb`，`well_formed_len` 保证截断不会切在字符中间。注意三个返回值的语义分工：`MY_CS_TOOSMALL`（字节不够，非错）、`MY_CS_ILSEQ`（非法序列，错）、正常返回字节数——上层靠这个区分"截断等待更多数据"和"数据坏了"。

##### 编码侧 `my_wc_mb_utf8mb4`：与解码并不对称

`my_wc_mb_utf8mb4` 是"纯写入、无验证"的编码器——只检查目标缓冲是否够、码点是否在上界内，然后用一个自 4→3→2→1 的 fallthrough 链**逆序填字节**（先写尾字节、最后写首字节）：

```c
static int my_wc_mb_utf8mb4(const CHARSET_INFO *cs, my_wc_t wc, uchar *r, uchar *e) {
  if (r >= e) return MY_CS_TOOSMALL;
  if (wc < 0x80)         count = 1;
  else if (wc < 0x800)   count = 2;
  else if (wc < 0x10000) count = 3;
  else if (wc < 0x200000) count = 4;      // ★ 上界是 0x200000，不是 0x10FFFF
  else return MY_CS_ILUNI;
  if (r + count > e) return MY_CS_TOOSMALLN(count);
  switch (count) {
    case 4: r[3] = 0x80 | (wc & 0x3f); wc >>= 6; wc |= 0x10000; [[fallthrough]];
    case 3: r[2] = 0x80 | (wc & 0x3f); wc >>= 6; wc |= 0x800;   [[fallthrough]];
    case 2: r[1] = 0x80 | (wc & 0x3f); wc >>= 6; wc |= 0xc0;    [[fallthrough]];
    case 1: r[0] = (uchar)wc;
  }
  return count;
}
```

与 `my_mb_wc_utf8mb4` 对比，**两侧并不严格对称**：

| | `mb_wc`（解码） | `wc_mb`（编码） |
|---|---|---|
| 合法性校验 | 拒绝 `0x80-0xC1` 开头、overlong、代理区 `U+D800..DFFF`、`> 0x10FFFF` | **不做等价检查**，`0x110000..0x1FFFFF` 也会被编成 4 字节 |
| 上界 | `> 0x10FFFF` 判非法 | 上界 `0x200000` |
| 失败码 | `MY_CS_ILSEQ` | `MY_CS_ILUNI` |

后果是**往返不封闭**：`wc = 0x110000` 能编码成 4 字节，但再解回来是 `MY_CS_ILSEQ`。更要命的是 **`MY_CS_ILUNI == MY_CS_ILSEQ == 0`**——两个失败码数值相同，调用方只能用 `<= 0` 判定失败，**无法区分"源串非法"与"目标装不下"**。这是字符集错误处理里一个持续存在的粗糙点。

单字节字符集的句柄组织是"**一张通用句柄 + 各字符集自己的映射表**"：绝大多数 8bit 字符集共用 `my_charset_8bit_handler`（ctype-simple.cc），各字符集只提供自己的 `tab_to_uni`/`tab_from_uni` 映射表；两个例外——latin1 因为需要 Unicode 双向映射自定义了句柄（`my_mb_wc_latin1` 走映射表），GBK 等多字节字符集各有自己的 handler（`my_mb_wc_gbk` 之类，不属于"简单字符集"）。这也解释了为什么"加一个单字节字符集"成本极低：一张映射表 + 挂通用句柄。

#### 函数表全景：除了比较与转换还有什么

`MY_CHARSET_HANDLER` 完整有 30 个函数指针，按职责分 8 组——前面只讲了比较/转换/大小写，这里把**剩下三组常被忽视的**补齐：

```
① init                         懒加载初始化（tailoring 解析在这）
② 多字节例程  ismbchar / mbcharlen / numchars / charpos / well_formed_len / lengthsp / numcells
③ Unicode 转换 mb_wc / wc_mb      ← 转换链（已详讲）
④ ctype scanner（多字节字符类别）  ← 已讲（my_mb_ctype_mb）
⑤ 大小写      caseup / casedn / caseup_str / casedn_str   ← 已详讲（含 Bug#119463）
⑥ 格式化输出  snprintf / long10_to_str / longlong10_to_str
⑦ fill        按字符集填充
⑧ 数字解析    strntol / strntoul / strntoll / strntoull / strntod / strtoll10 / strntoull10rnd
⑨ scan        扫描特定序列（跳过尾空格 MY_SEQ_SPACES 等）
```

**数字解析族（⑧）——"字符集感知的 `strtol`"**。C 标准库的 `strtol` 只认 ASCII 数字，而 MySQL 要把 `'１２３'`（全角数字）按所在字符集解析成 123。实现（`my_strntol_8bit`）循环里用 `my_isdigit(cs, c)` 判数字、用 `cs->to_lower` 判 a-f 十六进制字符——**一切按字符集的表来，而不是 ASCII 码值**。返回值语义与 `strtol` 不同：错误经 `*err` 出参（`MY_ERRNO_EDOM`/`MY_ERRNO_ERANGE`），`*e` 停在第一个未解析字符。

**格式化输出族（⑥）——"字符集感知的 `snprintf`"**。`my_snprintf_8bit` 是 `snprintf` 的字符集版：它把数字转成**字符集自己的**数字字形。底层原理——`cs->cset->ctype` 表里数字字符被标记 `_MY_NMR`，格式化时用 `'0' + digit` 得到的 ASCII 字符**在 8bit 字符集里恰好与数字字形重合**（单字节字符集的数字字形即 ASCII），所以实现可以共享；但多字节字符集若数字字形不同（如某些东亚字符集的全角数字），就不能走这条路——这是"为什么单字节与多字节的 cset 实现分家"的又一例证。

**`lengthsp` 的 binary 特例（② 里最容易被忽略的注释）**。`lengthsp` 剥尾空格，但 **binary collation 是特例——不剥，返回原长**。原因：`BINARY` 类型内部被映射为 `CHAR(n) COLLATE binary`，而 `Field_string` 读 CHAR 时依赖"剥空格"给出语义长度；binary 若也剥，`BINARY(n)` 读出来就不是完整的 n 字节 padded 宽度了。代价（注释明说 "it's suboptimal"）：**其他调用方也受影响**——比如用户给 enum 值输入时剥尾空格也走 `lengthsp`，此时 binary 语义会不剥。调用 `lengthsp` 前必须意识到这个特例。

**`numcells`——显示宽度**。`numcells` 返回字符串的"显示单元格数"（宽度），不是字节数也不是字符数——东亚全角字符算 2 个 cell、ASCII 算 1 个。它是 `CHAR_LENGTH` 之外的第三个长度维度（另有 `numchars` 算字符数、`mbcharlen` 算单字符字节数）。三者的区别：`mbcharlen('中')=3`（字节）、`numchars("中")=1`（字符）、`numcells("中")=2`（显示宽度）。

**`fill` / `scan`（⑦⑨）**：`fill` 按字符集填充（`fill(cs, to, len, ' ')` 用字符集的空格**字形式**填充）；`scan` 扫描特殊序列（`scan(cs, b, e, MY_SEQ_SPACES)` 跳过尾空格）——`test_if_important_data` 判定"截断丢的是不是重要数据"时就靠它先跳空格再比。

#### 大小写转换：caseup/casedn 与字节膨胀陷阱（Bug#119463）

##### 大小写数据：MY_UNICASE_INFO 二级页表

`cs->caseinfo` 是大小写映射的二级页表（`MY_UNICASE_INFO`），结构与 UCA 权重页同构但独立：

```
my_unicase_default = { 0xFFFF, my_unicase_pages_default }
                              │
                              ↓
my_unicase_pages_default[256]（页指针数组）
  [0x00] → plane00   (U+0000-U+00FF)  基本拉丁，有大小写 A-Z/a-z
  [0x01] → plane01   (U+0100-U+01FF)  拉丁扩展-A
  [0x03] → plane03   (U+0300-U+03FF)  希腊字母 A-W/a-w
  [0x04] → plane04   (U+0400-U+04FF)  西里尔字母
  [0x06] → nullptr   ← 阿拉伯字母：无大小写概念
  [0x4E] → nullptr   ← 汉字：无大小写概念
  ...大量 nullptr...
```

查找就是两次索引：`plane = pages[wc >> 8]`，`plane[wc & 0xFF]` 得到 `{大写, 小写, 标题}` 三元组。**为什么大量页是 nullptr**：这些区段的字符根本没有大小写概念（汉字、阿拉伯字母、符号），页表留空省内存——查到 nullptr 即"该字符不做大小写转换"。

查表函数本体（`my_tolower_utf8mb4`）就是这两次索引，且**页为 nullptr 或 `wc > maxchar` 时原样返回**（不映射）：

```c
static inline void my_tolower_utf8mb4(const MY_UNICASE_INFO *uni_plane, my_wc_t *wc) {
  if (*wc <= uni_plane->maxchar) {
    const MY_UNICASE_CHARACTER *page;
    if ((page = uni_plane->page[(*wc >> 8)]))     // 一级：高 8 位选页
      *wc = page[*wc & 0xFF].tolower;             // 二级：低 8 位选字符
  }
}
```

**`maxchar` 才是"能覆盖到哪些字符"的分水岭**：

| `caseinfo` 实例 | maxchar | 用在 |
|---|---|---|
| `my_unicase_default` / `mysql500` / `turkish` | **0xFFFF** | `utf8mb4_general_ci` 等 |
| `my_unicase_unicode520` | 0x10FFFF | `utf8mb4_unicode_520_ci` |
| `my_unicase_unicode900` | 0x10FFFF | `utf8mb4_0900_*` |

注意 `MY_UNICASE_CHARACTER` 只有 `{toupper, tolower, sort}` 三个 **`uint32` 单码点**字段——**没有"一变多"的槽位**，所以 `ß → SS`、`İ → i̇` 这类特殊映射在大小写转换里根本无法表达（只能在排序权重侧处理）。

**排序权重共用同一张表**：`my_tosort_unicode` 取的是第三字段 `sort`（仅当 `cs->state` 带 `MY_CS_LOWER_SORT` 时才改用 `tolower`），且 **`wc > maxchar` 时一律压成 `MY_CS_REPLACEMENT_CHARACTER`（U+FFFD）**：

```c
static inline void my_tosort_unicode(const MY_UNICASE_INFO *uni_plane, my_wc_t *wc, uint flags) {
  if (*wc <= uni_plane->maxchar) {
    const MY_UNICASE_CHARACTER *page;
    if ((page = uni_plane->page[*wc >> 8]))
      *wc = (flags & MY_CS_LOWER_SORT) ? page[*wc & 0xFF].tolower : page[*wc & 0xFF].sort;
  } else {
    *wc = MY_CS_REPLACEMENT_CHARACTER;   // 0xFFFD
  }
}
```

这就是 `utf8mb4_general_ci`（`caseinfo = my_unicase_default`，maxchar 仅 0xFFFF）下**所有非 BMP 字符权重相同**的**另一条独立路径**——它与 UCA 4.0 扫描器的 `wc > uca->maxchar → 0xFFFD` 是两处不同的代码，但产生同类后果。

##### 两条大小写转换路径

| 路径 | 字符集 | 做法 |
|---|---|---|
| **经 Unicode 码点** | UTF-8 系（utf8mb3/utf8mb4） | `mb_wc` 取码点 → 查 `caseinfo` 页表换码点 → `wc_mb` 编码回去 |
| **编码层面直接转换** | GB18030 / GBK 等 | 不经过 Unicode，直接用 `to_upper`/`to_lower` 数组按编码值映射 |

UTF-8 必须走码点的原因：UTF-8 是变长编码，同一个"字母"的大小写可能落在**不同的 UTF-8 长度档位**（这正是 Bug#119463 里 Ⱦ→⦦ 2→3 字节的来源）；而 GB18030 这类在编码层面就能给出映射表。这也解释了为什么"字节膨胀"问题只出现在 UTF-8 系。

`cset` 表里还有一对常被忽略的函数 `caseup`/`casedn`（及 `caseup_str`/`casedn_str`），配 `CHARSET_INFO` 的 `caseup_multiply`/`casedn_multiply` 字段。`multiply` 的语义是"**转换后最大字节数 / 原字节数**"——等于 1 表示"假设大小写转换不改变字节数"。

这个假设在**绝大多数**字符上成立，但 Unicode 里有一类"跨 UTF-8 编码长度边界"的大小写映射对，会让它破产，进而触发一个真实的截断 bug：

**Bug#119463：`utf8mb4_0900_ai_ci` 下 `SELECT lower('aaaȾbbb')` 返回 `'aaaⱦ'`——后面的 "bbb" 被吞掉了。**

**字符事实**：`Ⱦ`（U+023E，大写 T 带斜杠）UTF-8 编码是 `0xC8 0xBE`（**2 字节**），它的小写 `ⱦ`（U+2C66）编码是 `0xE2 0xB1 0xA6`（**3 字节**）——小写比大写**长 1 字节**。

**根因分两层**：

**第一层：算法缺陷——inplace 转换下的 src/dst 错位**（`my_casedn_utf8mb4`）。大小写转换允许 `src == dst`（原地转换，`caseup_multiply == 1` 时调用方复用 buffer）。旧代码循环体无条件推进两个指针：

```c
while ((src < srcend) &&
       (srcres = my_mb_wc_utf8mb4(&wc, src, srcend)) > 0) {
  my_tolower_utf8mb4(uni_plane, &wc);          // Ⱦ → ⱦ
  dstres = my_wc_mb_utf8mb4(cs, wc, dst, dstend);
  src += srcres;   // Ⱦ 前进 2 字节
  dst += dstres;   // ⱦ 前进 3 字节
}
```

inplace 下 `src` 与 `dst` 是**同一块 buffer**。处理 'Ⱦ' 时 `srcres=2`、`dstres=3`：dst 向前写了 3 字节（ⱦ 的 0xE2 0xB1 0xA6），src 只前进 2 字节，于是 src 现在指向 ⱦ 的第 3 字节 `0xA6` 加上后面的 `bb`：

```
处理前:  [C8 BE] [62 62 62]          src→C8  dst→C8
写 ⱦ :  [E2 B1 A6] [62 62 62]       dst 前进 3 → 指向 62
src+=2:  [E2 B1 A6] [62 62 62]       src 前进 2 → 指向 A6  ← 错位！
下一轮:  mb_wc(A6 62 62) → A6 开头应是双字节，但 0x62 不是合法续字节
         → MY_CS_ILSEQ → 循环 break → 结果只剩 'aaaⱦ'
```

**第二层：数据层——UCA 900 的 caseinfo 映射**。`strings/uca900_data.h` 里的 `MY_UNICASE_CHARACTER` 三元组 `{大写, 小写, 标题}`，U+023E 正确值是 `{0x023E, 0x2C66, 0x023E}`（小写是 0x2C66 不是自身）。若表里被写成 `{0x023E, 0x023E, 0x023E}`，`lower('Ⱦ')` 就不转换——但一旦数据是对的，就会触发第一层的字节膨胀。

**修复链（三处配套）**：

##### 尚未封堵的对称面：NUL 结尾版 `caseup_str` / `casedn_str`

`cset` 表里还有一对 NUL 结尾版本（`my_caseup_str_utf8mb4` / `my_casedn_str_utf8mb4`），它们**有完全相同的字节膨胀隐患，而且更糟**：

```c
static size_t my_casedn_str_utf8mb4(const CHARSET_INFO *cs, char *src) {
  char *dst = src, *dst0 = src;          // ★ dst == src：恒 inplace
  assert(cs->casedn_multiply == 1);      // ★ 唯一防线，release 下被编译掉
  while (*src && (srcres = my_mb_wc_utf8mb4_no_range(cs, &wc, (uchar *)src)) > 0) {
    my_tolower_utf8mb4(uni_plane, &wc);
    if ((dstres = my_wc_mb_utf8mb4_no_range(cs, wc, (uchar *)dst)) <= 0) break;
    src += srcres;
    dst += dstres;                       // ★ dst 可能跑得比 src 快 → 覆盖未读输入
  }
  *dst = '\0';
  return (size_t)(dst - dst0);
}
```

三点差异（都比带长度版更危险）：① 用 `_no_range` 变体，**没有边界检查**；② 以 `*src` 终止；③ 失败只 `break`，**没有 `-1` 失败通道**，返回值还可能大于输入长度。

`assert(cs->casedn_multiply == 1)` 是"inplace 安全"的**前置断言而非运行时保护**——而 Bug#119463 恰恰证明这个声明对 utf8mb4 不成立。

**为什么没爆**：所有调用点处理的都是**标识符**——库名、表名、列名、权限名、文件名（`sql_table.cc`、`sql_trigger.cc`、`sp.cc`、dd 缓存、`auth` 相关），实际内容限于 ASCII/文件系统安全字符，不会出现"映射后变长"的码点。这是**输入域约束下的侥幸，不是代码正确性**，也是 Bug#119463 尚未封堵的对称面。

1. **`ctype-utf8.cc`**：`my_caseup_utf8mb4`/`my_casedn_utf8mb4` 检测到"inplace 且字节数变化"时**返回 -1**（表示"这个转换原地做不了，需要独立/更大 buffer"）：

```c
if (srcres == dstres || src != dst) {
  src += srcres;
  dst += dstres;
} else {
  return -1;   // inplace 且 srcres != dstres：字节膨胀/收缩，放弃原地
}
```

2. **`item_strfunc.cc` 的 `Item_str_conv::val_str`**（LOWER/UPPER 的 Item）：先备份 `orig_res`，收到 `-1` 后 `++multiply` 放大 buffer、`res->swap(orig_res)` 恢复原数据，再走非 inplace 的 `tmp_value.alloc` 路径重试：

```c
orig_res.copy(res->ptr(), res->length(), res->charset());
len = converter(collation.collation, res->ptr(), res->length(),
                res->ptr(), res->length());   // 先试 inplace
if (len == -1) {                              // 原地失败
  ++multiply;                                 // 放大预估倍数
  res->swap(orig_res);                        // 恢复被破坏的数据
  goto multiplyx;                             // 走独立 buffer 重试
}
```

3. **`uca900_data.h`**：恢复 U+023E 的正确映射 `{0x023E, 0x2C66, 0x023E}`。

**这个 bug 揭示的深层教训**：`caseup_multiply`/`casedn_multiply == 1` 是一个"**大小写不改变字节数**"的假设，它只在单字节字符集和多数 Unicode 字符上成立；一旦遇到 2↔3 字节跨编码边界的映射对（Ⱦ↔ⱦ、Ȿ↔ȿ、ɀ↔Ɀ 等都是），inplace 转换就会 src/dst 错位。修复的本质不是"修一个字符"，而是**给转换函数加一个"原地失败返回 -1、上层扩容重试"的协商协议**——让"能原地就原地，不能就换大 buffer"成为显式契约，而不是靠 `multiply` 的静态预估赌它够大。

##### 影响面：不是 0900 独有

- **双向都中招**：`lower('aaaȾbbb')` 截断为 `'aaaⱦ'`（2→3 字节膨胀），`upper('aaaɀbbb')` 同样截断（ɀ U+0240，2 字节 → Ɀ U+2C7F，3 字节）——所以 `caseup` 和 `casedn` 两个函数都要修
- **不止 UCA 900**：`utf8mb4_unicode_520_ci` 下同样复现，`utf8mb4_unicode_ci`（UCA 4.0）亦然——**凡是"支持较完整 Unicode 大小写数据"的 collation 都会触发**
- **`utf8mb4_general_ci` 反而"免疫"**：不是因为它更健壮，而是因为它的大小写数据**不完整**——general_ci 里 `Ⱦ` 根本没有小写映射（大小写都是它自己），`srcres == dstres == 2`，字节数不变，自然不触发。**"数据不全"意外掩盖了算法缺陷**；而把数据补正确（这正是 UCA 900 做的）反而把 bug 暴露出来

##### 修复方案的选型（两案对比）

| 方案 | 做法 | 代价 |
|------|------|------|
| A. 静态放大 multiply | 把 UCA 系列 collation 的 `caseup_multiply`/`casedn_multiply` 改成 2 | 所有 UCA（520/900）定义都要改；**每次大小写转换都申请 2 倍空间**——为极少数特殊字符让所有转换都付出双倍内存 |
| B. 动态协商（采用） | 转换函数检测 `srcres != dstres` 且 inplace → 返回失败；上层 `++multiply` 扩容走独立 buffer 重试 | 要同时改 `casedn` 和 `caseup` 两个函数；但只有真正遇到字节膨胀时才付出扩容代价 |

**选型逻辑**：这是典型的"**静态预估 vs 动态协商**"权衡。方案 A 用空间换简单（一次改定义，永远够用），代价是每次 LOWER/UPPER 都多分配一倍内存；方案 B 用复杂度换空间（多一次失败重试路径），只在罕见字符上付出代价。选 B——因为绝大多数串不包含跨编码边界字符，"为 0.001% 的情况让 100% 的转换多花一倍内存"不划算。

##### 一个补丁演进细节：失败返回值从 0 改成 -1

初版补丁让转换函数返回 `0` 表示"原地失败"，上层判 `if (len == 0)`。但 `0` **有歧义**——它同时是"转换结果长度为 0"的合法返回值：

```sql
mysql> SELECT BINARY lower('Ⱦ');
+-------------------+
| BINARY lower('Ⱦ') |
+-------------------+
| 0x                |   ← 空串 + 1 warning：长度 0 是合法结果
+-------------------+
```

所以最终实现改为返回 **`-1`**（`size_t` 下回绕为最大值），上层判 `if (len == -1)`——用一个不可能是合法长度的哨兵值消除歧义。这类"用哨兵值做带外信号"在 C 接口里常见，也正是容易踩坑之处（返回类型 `size_t` 无符号，`-1` 实际是 `SIZE_MAX`）。

### 排序处理器：比较、排序键、LIKE、哈希

coll 函数表的四个大头：

**strnncoll vs strnncollsp**——PAD 语义的分野。PAD SPACE 的 collation（如 latin1_swedish_ci）里 `strnncollsp` 是"尾部补齐空格再比"：

```c
// my_strnncollsp_uca（旧 UCA 4.0 版，PAD SPACE）
// 一侧扫描完后：
//   取空格的权重 my_space_weight(cs)，与另一侧剩余权重继续比
//   ——即"短串尾部补无限个空格"的语义
```

而 NO PAD 的 0900 系列直接把 `strnncollsp` 别名到 `strnncoll`：

```c
static int my_strnncollsp_uca_900(...) {
  /* We are a NO PAD collation, so this is identical to strnncoll */
  return my_strnncoll_uca_900(...);
}
```

**★★ 三个"补空格"版本的完整源码**（★ 三个函数长得像但语义不同，是最易混的一组）：

```c
/* strings/ctype-bin.cc:117 —— ① my_charset_bin（NO PAD）：纯转发，不补空格 */
static int my_strnncollsp_binary(const CHARSET_INFO *cs, const uchar *s,
                                 size_t slen, const uchar *t, size_t tlen) {
  return my_strnncoll_binary(cs, s, slen, t, tlen, false);
}

static int my_strnncoll_binary(const CHARSET_INFO *cs [[maybe_unused]],
                               const uchar *s, size_t slen, const uchar *t,
                               size_t tlen, bool t_is_prefix) {
  size_t len = std::min(slen, tlen);
  const int cmp = len == 0 ? 0 : memcmp(s, t, len);   // memcmp(a,b,0)==0
  return cmp ? cmp : (int)((t_is_prefix ? len : slen) - tlen);
}
```

```cpp
/* strings/ctype-bin.cc:175 —— ② 8bit 的 *_bin（PAD SPACE）：长串多出的第一个非空格字节决定胜负 */
static int my_strnncollsp_8bit_bin(const CHARSET_INFO *cs, const uchar *a,
                                   size_t a_length, const uchar *b, size_t b_length) {
  const uchar *end; size_t length; int res;
  end = a + (length = std::min(a_length, b_length));
  while (a < end) {
    if (*a++ != *b++) return ((int)a[-1] - (int)b[-1]);
  }
  res = 0;
  if (a_length != b_length) {
    int swap = 1;
    /* Check the next not space character of the longer key. If it's < ' ',
       then it's smaller than the other key. */
    if (a_length < b_length) {       /* ★ 统一把"长串"放到 a，用 swap 翻转符号 */
      a_length = b_length; a = b; swap = -1; res = -res;
    }
    for (end = a + a_length - length; a < end; a++) {
      if (*a != ' ') return (*a < ' ') ? -swap : swap;   /* ★ 与字节 0x20 比 */
    }
  }
  return res;
}
```

```cpp
/* strings/ctype-simple.cc:152 —— ③ simple（latin1 等）：与 ② 同构，但比较的是 sort_order 权重 */
int my_strnncollsp_simple(const CHARSET_INFO *cs, const uchar *a,
                          size_t a_length, const uchar *b, size_t b_length) {
  const uchar *map = cs->sort_order, *end;    /* ★ 256 字节权重表 */
  ...
    if (map[*a++] != map[*b++]) return ((int)map[a[-1]] - (int)map[b[-1]]);
  ...
    for (end = a + a_length - length; a < end; a++) {
      if (map[*a] != map[static_cast<int>(' ')])
        return (map[*a] < map[static_cast<int>(' ')]) ? -swap : swap;
        /* ★★ 与"空格的权重"比，而不是与字节 0x20 比 */
    }
}
```

> ★★ 三条不变式（源码注释原文）：`"a" == "a "` / `"a\0" < "a"` / `"a\0" < "a "`。
> ⚠️ **不存在 `my_strnncollsp_bin()`** —— 真名是 `my_strnncollsp_binary`（binary，NO PAD）与 `my_strnncollsp_8bit_bin`（`*_bin`，PAD SPACE）。

**★ `my_strnncoll_simple()` 有个刻意的写法**（★ 与 `strnncollsp` 不同）：

```cpp
int my_strnncoll_simple(const CHARSET_INFO *cs, const uchar *s, size_t slen,
                        const uchar *t, size_t tlen, bool t_is_prefix) {
  size_t len = (slen > tlen) ? tlen : slen;
  const uchar *map = cs->sort_order;
  if (t_is_prefix && slen > tlen) slen = tlen;
  while (len--) {
    if (map[*s++] != map[*t++]) return ((int)map[s[-1]] - (int)map[t[-1]]);
  }
  /* We can't use (slen - tlen) here as the result may be outside of the
     precision of a signed int */
  return slen > tlen ? 1 : slen < tlen ? -1 : 0;   /* ★★ 只返回 ±1/0，不返回差值 */
}
```

> ★ 注释解释得很直白：`slen - tlen` 是 `size_t`，差值可能溢出 `int`。

**★ `my_strnxfrm_simple()` 与 `my_strxfrm_pad()`**（★ PAD SPACE 在这里统一收尾）：

```cpp
size_t my_strnxfrm_simple(const CHARSET_INFO *cs, uchar *dst, size_t dstlen,
                          uint nweights, const uchar *src, size_t srclen, uint flags) {
  const uchar *map = cs->sort_order;
  uchar *d0 = dst; ...
  // Do the first few bytes.
  remainder = src + (frmlen % 8);
  for (; src < remainder;) *dst++ = map[*src++];      /* ★ 1:1 字节→权重 */

  // Unroll loop for rest of string.
  for (; src < end;) { *dst++ = map[*src++]; ... 共 8 次 ... }  /* ★ 8 路展开 */

  return my_strxfrm_pad(cs, d0, dst, d0 + dstlen, (uint)(nweights - frmlen), flags);
}

size_t my_strxfrm_pad(const CHARSET_INFO *cs, uchar *str, uchar *frmend,
                      uchar *strend, uint nweights, uint flags) {
  if (nweights && frmend < strend) {
    // PAD SPACE behavior.
    uint fill_length = std::min<uint>(strend - frmend, nweights * cs->mbminlen);
    cs->cset->fill(cs, (char *)frmend, fill_length, cs->pad_char);
    frmend += fill_length;
  }
  if ((flags & MY_STRXFRM_PAD_TO_MAXLEN) && frmend < strend) {
    size_t fill_length = strend - frmend;
    cs->cset->fill(cs, (char *)frmend, fill_length, cs->pad_char);
    frmend = strend;
  }
  return frmend - str;
}
```

**★ `my_like_range_*` 的真名勘误**（⚠️ **`my_like_range_bin()` / `my_like_range_utf8mb4()` 都不存在**）：

| 实际函数 | 谁在用 |
|---|---|
| `my_like_range_simple()` | 8bit simple **以及** `binary` / `*_bin`（`ctype-bin.cc:461/475`——因为它们都是单字节） |
| `my_like_range_mb()` | utf8mb4 的 general_ci / bin / 0900_bin **全部复用**（`ctype-utf8.cc:7776/7790`、`ctype-uca.cc:11430`） |
| `my_like_range_generic()` | ucs2 / utf16 / utf32（`ctype-mb.cc:809`） |

```cpp
/* LIKE 优化的核心分支（my_like_range_mb） */
    if ((cs->state & MY_CS_BINSORT) || cs->pad_attribute == NO_PAD) {
      *min_length = static_cast<size_t>(min_str - min_org);
      /* "a" is the smallest possible string for NO PAD. */
      do { *min_str++ = ' '; } while (min_str != min_end);
      /* Pad with spaces, because for CHAR searches, our returned min_length
         is ignored and min_str is put directly into the value to search for. */
    } else {
      *min_length = res_length;
      do { *min_str++ = static_cast<char>(cs->min_sort_char); } while (min_str != min_end);
      /* "a\0\0..." is the smallest possible string for PAD SPACE. */
    }
    *max_length = res_length;
    pad_max_char(cs, max_str, max_end);      /* "a\xff\xff..." */
```

> ★★ **LIKE 范围优化必须避开 contraction 头**——源码里有一段长注释专门解释：捷克语里 `ch` 是一个排在 `h` 和 `i` 之间的独立字母，若 `LIKE 'abc%'` 直接返回 `'abc\0...'` ~ `'abc\xff...'`，会漏掉 `abch` 开头的行。所以碰到 `contraction head + 通配符` 就 `goto fill_max_and_min` 退回成 `LIKE 'ab%'` 的范围。

**★ `my_wildcmp_*` 的四个变体靠宏分派**（★ 三个实现几乎逐行同构，差别只在两个宏）：

```cpp
/* binary */
#define likeconv(s, A) (A)                                  /* ★ 恒等：不过权重表 */
#define INC_PTR(cs, A, B) (A)++

/* 8bit simple */
#define likeconv(s, A) (uchar)(s)->sort_order[(uchar)(A)]   /* ★ 过 sort_order → CI */
#define INC_PTR(cs, A, B) (A)++

/* 多字节 */
#define INC_PTR(cs, A, B) A += (my_ismbchar(cs, A, B) ? my_ismbchar(cs, A, B) : 1)
#define likeconv(s, A) (uchar)(s)->sort_order[(uchar)(A)]
```

> ⚠️ **`my_wildcmp_simple()` 不存在** —— 真名是 **`my_wildcmp_8bit()`**（`ctype-simple.cc:743`）。
> ★ 四者共同点：`my_string_stack_guard(recurse_level)` 防 `'%a%a%a...'` 递归爆栈；经典"锚点字符 + 递归回溯"结构；返回值三态 `0`(匹配) / `-1`(不匹配) / `1`(用了通配符才匹配)。
> ★ `my_wildcmp_utf8mb4()` 只是转发 `my_wildcmp_unicode(cs, ..., cs->caseinfo)`。

**★ `hash_sort` 的三兄弟**（★ 差异就在"去不去尾空格"）：

```cpp
void my_hash_sort_simple(const CHARSET_INFO *cs, const uchar *key, size_t len,
                         uint64 *nr1, uint64 *nr2) {
  const uchar *sort_order = cs->sort_order;
  end = skip_trailing_space(key, len);      /* ★ 去尾空格 */
  for (; key < end; key++) {
    tmp1 ^= (uint64)((((uint)tmp1 & 63) + tmp2) * ((uint)sort_order[(uint)*key])) + (tmp1 << 8);
    /*                                              ★ 喂进哈希的是权重 → 'A' 与 'a' 同哈希 */
    tmp2 += 3;
  }
}

static void my_hash_sort_bin(...) {
  key += len;                               /* ★★ binary：不去尾空格（NO PAD） */
  for (; pos < key; pos++) { tmp1 ^= ... * ((uint)*pos) ...; tmp2 += 3; }
}

static void my_hash_sort_8bit_bin(...) {
  key = skip_trailing_space(key, len);      /* ★ *_bin：去尾空格（PAD SPACE） */
  ...
}
```

```cpp
/* utf8mb4：码点→排序权重→按 2~3 字节喂进哈希 */
static void my_hash_sort_utf8mb4(const CHARSET_INFO *cs, const uchar *s,
                                 size_t slen, uint64 *n1, uint64 *n2) {
  while (e > s && e[-1] == ' ') e--;        /* 去尾空格 */
  while ((res = my_mb_wc_utf8mb4(&wc, s, e)) > 0) {
    my_tosort_unicode(uni_plane, &wc, cs->state);   /* ★ 大小写折叠 */
    ch = (wc & 0xFF);        tmp1 ^= ...; tmp2 += 3;
    ch = (wc >> 8) & 0xFF;   tmp1 ^= ...; tmp2 += 3;
    if (wc > 0xFFFF) {
      /* Put the highest byte only if it is non-zero, to make hash functions
         for utf8mb3 and utf8mb4 compatible for BMP characters. ... */
      ch = (wc >> 16) & 0xFF; tmp1 ^= ...; tmp2 += 3;   /* ★★ 只为兼容 utf8mb3 */
    }
    s += res;
  }
}
```

**★ `instr` 的两个实现**（⚠️ 声明里叫 `strstr`，实现函数名才带 `instr`）：

```cpp
uint my_instr_simple(...) {
  if (cs->sort_order[*str++] == cs->sort_order[*search]) {   /* ★ 权重定位 */
    ... while (j != search_end) if (sort_order[*i++] != sort_order[*j++]) goto skip;
  }
}

uint my_instr_mb(...) {
  if (!cs->coll->strnncoll(cs, b, s_length, s, s_length, false)) {
    /* ★★ 复用 collation 的 strnncoll —— 自动支持 CI / contraction / UCA */
  }
  mb_len = (mb_len = my_ismbchar(cs, b, end)) ? mb_len : 1;   /* ★ 按字符步进 */
  b += mb_len; b_length -= mb_len; res++;
}
```

**strnxfrm 排序键**——把字符串变成可直接 `memcmp` 的字节流：**16 位大端权重序列，级别间嵌 0x0000 分隔**，PAD SPACE 下末尾补空格权重：

```c
// my_strnxfrm_uca 骨架
while (scanner.next() 还有权重) {
  store16be(dst, weight);   // 大端写：保证 memcmp 顺序 == 权重顺序
}
```

用一个完整例子看排序键怎么拼（UCA 的经典流程，串 "aáA"）：

1. 每个字符展开为 Collation Element（DUCET 里的 `[.AAAA.BBBB.CCCC]` 三元组：primary.secondary.tertiary）：
   ```
   a → [.1C47.0020.0002]
   á → [.1C47.0020.0002][.0000.0024.0002]   ← 分解为 a + 重音符，两个 CE
   A → [.1C47.0020.0008]                    ← 大小写差异在 tertiary
   ```
2. 按级别抽取权重：SortKey1 = `1C47 1C47 1C47`（primary 全同）、SortKey2 = `0020 0020 0024 0020`（重音差异）、SortKey3 = `0002 0002 0002 0008`（大小写差异）
3. 级别间以 0x0000 分隔拼接：
   ```
   SortKey = 1C471C471C47 0000 0020002000240020 0000 0002000200020008
   ```

这就是 `WEIGHT_STRING(s)` 返回的字节流（`HEX(WEIGHT_STRING('aáA'))` 可以亲眼看到），也是 filesort/索引比较最终 memcmp 的对象。**级别分隔符 0x0000 的意义**：任何权重的 primary 都 ≥ 0x0100，0x0000 永不与权重混淆，天然成为级别边界。

**排序键长度是"估算"而非精确**——因为调用方（典型是 filesort）要**预分配定长缓冲**。900 版的估算：

```c
const size_t num_codepoints = (len + 3) / 4;          // 注释：We really ought to have len % 4 == 0
const size_t max_num_weights_per_level = num_codepoints * 8;   // U+FDFA 单字符最多截断为 8 个权重
size_t max_num_weights = max_num_weights_per_level * cs->levels_for_compare;
return (max_num_weights + (cs->levels_for_compare - 1)) * sizeof(uint16_t);  // +级分隔符
```

而 **4.0 系 collation（`utf8mb4_unicode_ci` 等）用的是 `my_strnxfrmlen_simple`**（`len * strxfrm_multiply`），只有 900 版有上面这套按 CE 数的估算。filesort 侧用它预分配并把长度写回 `sortorder->length`，超过 10MB 时直接设 `0xFFFFFFFF` 防溢出；实际写键时若 `actual_length` 超出预估即判溢出。

唯一在用的 flag 是 `MY_STRXFRM_PAD_TO_MAXLEN`（"pad tail for filesort"）：把缓冲尾部填满（900 版直接 `memset(..., 0, ...)`），保证定长键可比。（`MY_STRXFRM_DESC`/`MY_STRXFRM_NLEVELS` 这类 5.x 的 flag 在 8.0.39 已不存在。）

**like_range**——LIKE 优化的范围计算。`'abc%'` 的最小键是 'abc' 本身，最大键是"abc 之后所有可能字符的最小值"：

```c
// my_like_range_mb 骨架（ctype-mb.cc）
// 逐字符复制普通字节；
// 遇到 '%'/'_' 时（fill_max_and_min）：
//   BINSORT 或 NO_PAD → min = 已复制前缀 + 空格
//   否则 → min 用 min_sort_char 填满、max 用 max_sort_char 填满
```

这里有个隐蔽的坑（源码注释点名）：**contraction 语言下 'abc%' 的最大键必须扩到 'abc[h]' 之下**——捷克语 'ch' 是一个排序单位，最大键若取 'abc\uffff' 会漏掉 'abch' 开头的串。

**wildcmp**——LIKE 的实际匹配。0900 版是逐码点递归：

```c
// my_wildcmp_uca_impl 骨架
// '%' → 递归 my_wildcmp_uca_impl(..., recurse_level+1)
//      + my_string_stack_guard 栈深度检查
// '_' → 跳过一个码点
// escape 字符 → 下一个字符原样比较
// 等值判定 → my_uca_charcmp（走权重而非字节）
```

> **栈保护的语义要精确**：`my_string_stack_guard` 超限时返回的是 **-1（"不匹配"）**，不是报错、也不是崩溃——即它是一个**静默的正确性降级**。这一点在评估"多 `%` 的病态模式"风险时很关键：查询不会失败，但会**漏掉结果**。
>
> `wildcmp` 在本仓库有 7 份实现（8bit / bin / mb / mb_bin / unicode / uca / gb18030），由 `cs->coll->wildcmp` 分发；`like_range` 同理。**算法骨架与"能否转 range、怎么执行、代价怎么估"的完整链路**见 [`../query/15_like.md`](../query/15_like.md)（本篇只讲字符集侧的回调语义与权重来源）。

**hash_sort**——PARTITION BY KEY 的哈希。注释强调"格式不可变"：分区表哈希值一旦变，已有分区的行归属全乱。

这个"不可变"的权威出处是 `MY_COLLATION_HANDLER::hash_sort` 的契约注释——它要求 **a=b ⇒ H(a)=H(b)**，且"不能更改，除非在写新 collation 时；必须跨 release 保持不变，以免磁盘格式变化"（它还被 MEMORY 引擎用于等值判定）。

900 版实现是 **FNV-1a 变体**，直接消费权重流而不是先生成排序键：

```c
uint64 h = *n1;
h ^= 14695981039346656037ULL;                 // FNV offset basis
scanner.for_each_weight([&](int s_res, bool) {
  h ^= s_res;                                  // 按 16 位权重整体 XOR
  h *= 1099511628211ULL;                       // FNV prime
  return true;
}, ...);
*n1 = h;
```

**与 `strnxfrm` 的关系**：两者复用同一套扫描器与权重数据（因此共享 tailoring），但 hash 不落盘、不受 PAD/级分隔符/缓冲截断影响。共同前提是"相等字符串产生相等权重序列"——所以**DUCET 版本或 tailoring 一变，hash 值就变，分区路由随之改变**。这就是为什么 collation 升级不能随意改动，也是 `hash_sort` 被要求跨版本冻结的原因。

### UCA 权重算法（★ 本篇最重难点）

这是整个字符集系统里最有技术含量的一块。8.0 有两个版本并存：

| | my_uca_v400（unicode_ci 系列） | my_uca_v900（0900 系列） |
|---|---|---|
| 权重版本 | UCA 4.0 | UCA 9.0.0 |
| 比较模型 | **只存 primary 一级**（一个码点的多个 CE 主权重拍平成 0 结尾串） | 三级权重**分 1-4 级并行比较** |
| pad | PAD SPACE | NO PAD |
| levels | 恒为 1（源码里 4.0 路径全部以 `LEVELS_FOR_COMPARE = 1` 实例化） | levels_for_compare：ai=1/as_ci=2/as_cs=3/ja_ks=4 |
| 性能 | 无 | 有 SIMD 化 ASCII 快路径 |

#### 权重表布局

`MY_UCA_INFO` 是权重数据的顶层句柄（`my_uca_v400` / `my_uca_v900` 两个实例）：

```c
struct MY_UCA_INFO {
  enum enum_uca_ver version;     // UCA_V400 / UCA_V900
  my_wc_t maxchar;               // 900: 0x10FFFF（覆盖全 Unicode）
  uchar *lengths;                // 900 版不使用（固定 4×256 布局）；400 版用变长
  uint16 **weights;              // ★ 核心：4352 个页指针
  bool have_contractions;        // 是否存在 contraction
  std::vector<MY_CONTRACTION> *contraction_nodes;
  char *contraction_flags;       // 位图：快速排除不可能参与 contraction 的字符
  my_wc_t first_non_ignorable;   // 0x0009
  my_wc_t last_non_ignorable;    // 0x14646
  uint16 extra_ce_pri_base;      // 0x54A4（额外 CE 的权重基准）
  uint16 extra_ce_sec_base;      // 0x0115
  uint16 extra_ce_ter_base;      // 0x0020
};
```

`weights` 是 **4352 个页指针**（对应 U+0000 到 U+10FFFF 按 256 分组的页数），未分配的页是 `nullptr`——这正是"为什么很多字符要走 implicit weight"的原因。

**对照：UCA 4.0 的页布局完全不同。** 4.0 没有"每级 256 个权重"的固定布局，而是**每个码点一个定长槽位、槽内存一条变长 0 结尾的主权重串**：

```
uca_length[256] = {4, 3, 3, 4, ..., 8, 9, ...}   // 该页每个码点占几个 uint16（0 = 页不存在）
uca_weight[256] = { page000data, page001data, ..., nullptr, ... }  // 页指针

寻址：wbeg = uca->weights[wc >> 8] + (wc & 0xFF) * uca->lengths[wc >> 8]
```

一条槽位内容示例（U+FB00 `ﬀ`，长度 4）：`0x0EB9, 0x0EB9, 0x0000, 0x0000`——两个主权重 + 补 0 结尾。所以 4.0 的 `lengths[]` 是"每字符槽位大小"，而 900 版根本不需要它（固定 4×256）。

**关键点：4.0 表里只有 primary 一级。** 源码里 4.0 相关处留有 `/* W3-TODO */` 注释，且所有 4.0 路径都以 `LEVELS_FOR_COMPARE = 1` 实例化——所谓"折叠"折叠的是**同一码点多个 CE 的主权重**，不是三级权重。这解释了 `utf8mb4_unicode_ci` 为什么同时忽略大小写与重音：它的权重数据里压根没有 secondary/tertiary。

**扫描器的级间行为差异**（这是两版算法的分水岭）：

| | `uca_scanner_any`（4.0） | `uca_scanner_900` |
|---|---|---|
| 一级扫描完 | `++weight_lv; return -1`（**不重启**） | `if (++weight_lv < LEVELS) { sbeg = sbeg_dup; return 0; }` —— **返回 0 作级分隔符，并从头重扫下一级** |
| CE 遍历 | 行内连续取 | `num_of_ce_left` + stride（CE-major 遍历） |
| 额外能力 | 无 | `apply_case_first` / `apply_reorder_param` / Hangul 音节分解 / 日文四级 / ASCII 快路径 |

即：900 的"分级"是靠**扫描器反复从头扫、每次只取一级权重**实现的（所以 900 的 `strnxfrm` 输出里级别间是 0x0000 分隔——那个 0 就是扫描器返回的级分隔符）。

**一次完整的权重查表**（以 'A' 为例）：

```
字节 0x41 (1 字节 UTF-8)
  ↓ mb_wc()
码点 U+0041
  ↓ page = wc >> 8 = 0x00,  code = wc & 0xFF = 0x41
wpage = uca->weights[0x00] = uca900_p000
  ↓ 三个宏分别取三级权重
primary   = UCA900_WEIGHT(wpage, 0, 0x41)   → 页内偏移 256 + 0*256 + 0x41
secondary = UCA900_WEIGHT(wpage, 1, 0x41)   → 页内偏移 256 + 1*256 + 0x41
tertiary  = UCA900_WEIGHT(wpage, 2, 0x41)   → 页内偏移 256 + 2*256 + 0x41
```

三级权重里，'A' 与 'a' 的 primary/secondary 相同、**只有 tertiary 不同**——这就是大小写敏感（as_cs）与否（ai_ci）在数据层的体现。

900 版每页固定 4×256 布局（见理论基础·算法与数据结构的图）。查权重的宏：

```c
#define UCA900_NUM_OF_CE(page, subcode) ((page)[(subcode)])
#define UCA900_WEIGHT(page, level, subcode) (page)[256 + (level)*256 + (subcode)]
```

**★ 页布局宏的完整定义**（★ 前 256 项存 CE 个数，其后每级 256 项）：

```c
#define UCA900_NUM_OF_CE(page, subcode)          ((page)[(subcode)])
#define UCA900_WEIGHT_ADDR(page, level, subcode) ((page) + 256 + (level) * 256 + (subcode))
#define UCA900_WEIGHT(page, level, subcode)      (page)[256 + (level) * 256 + (subcode)]
#define UCA900_DISTANCE_BETWEEN_LEVELS  256
#define UCA900_DISTANCE_BETWEEN_WEIGHTS (MY_UCA_900_CE_SIZE * 256)
```

```cpp
/* strings/str_uca_type.h:76 —— 契约节点 + UCA 总控 */
#define MY_UCA_MAX_CONTRACTION 6
#define MY_UCA_MAX_WEIGHT_SIZE 25
#define MY_UCA_WEIGHT_LEVELS 1

struct MY_CONTRACTION {
  my_wc_t ch;
  std::vector<MY_CONTRACTION> child_nodes;          /* 后续字符（普通契约） */
  std::vector<MY_CONTRACTION> child_nodes_context;  /* 后续字符（前文语境契约） */
  uint16 weight[MY_UCA_MAX_WEIGHT_SIZE];            /* ★ 0 结尾的权重串 */
  bool is_contraction_tail;
  size_t contraction_len;
};

struct MY_UCA_INFO {
  enum enum_uca_ver version;      /* UCA_V400 / UCA_V520 / UCA_V900 */
  my_wc_t maxchar;
  uchar *lengths;                 /* 仅 4.0 用：该页每码点槽位大小 */
  uint16 **weights;
  bool have_contractions;
  std::vector<MY_CONTRACTION> *contraction_nodes;
  char *contraction_flags;        /* ★ 4096 字节位图，快速排除 */
  /* Logical positions：first/last_non_ignorable、*_primary_ignorable、... */
  uint16 extra_ce_pri_base, extra_ce_sec_base, extra_ce_ter_base;
};
```

> ⚠️ **不存在 `struct MY_UCA_WEIGHT` / `weight[2]` / `level` 字段**——契约节点权重是 `MY_CONTRACTION::weight[25]`，UCA900 单字符权重是 `uint16` 页数组，**级间靠 `UCA900_DISTANCE_BETWEEN_LEVELS=256` 偏移**，没有 `level` 成员。
> ⚠️ **不存在 `MY_UCA_LEVELS`**——分级语义靠 `CHARSET_INFO::levels_for_compare`（`uchar`，1..4）+ `uca_scanner_900<..., LEVELS_FOR_COMPARE>` **模板参数**。
> ⚠️ **不存在 `my_uca_get_weight()`** —— 查权重地址的真名是 `my_char_weight_addr()`（4.0）/ `my_char_weight_addr_900()`。
> ⚠️ **不存在 `my_uca_scanner_any()` / `my_uca_scanner_next()`** —— 扫描器是**类**（基类 `my_uca_scanner` + 模板 `uca_scanner_any<Mb_wc>` / `uca_scanner_900<Mb_wc, LEVELS>`），推进方法是成员 `next()`。
> ⚠️ **`my_uca_v900` 不在 `ctype-uca.cc`** —— 在 `strings/uca900_data.h:270943`（`my_uca_v400`/`my_uca_v520` 才在 `ctype-uca.cc`）。

```cpp
/* 两版寻址的对照（ctype-uca.cc:1852） */
static inline uint16 *my_char_weight_addr(MY_UCA_INFO *uca, my_wc_t wc) {
  uint page, ofst;
  return wc > uca->maxchar ? nullptr
       : (uca->weights[page = (wc >> 8)]
            ? uca->weights[page] + (ofst = (wc & 0xFF)) * uca->lengths[page]
            : nullptr);           /* ★ 4.0：页内步长 = lengths[page] */
}
static inline uint16 *my_char_weight_addr_900(MY_UCA_INFO *uca, my_wc_t wc) {
  if (wc > uca->maxchar) return nullptr;
  uint page = wc >> 8, ofst = wc & 0xFF;
  uint16 *weights = uca->weights[page];
  return weights ? UCA900_WEIGHT_ADDR(weights, 0, ofst) : nullptr;   /* ★ 900：固定布局 */
}
```

**★ contraction 的三层实现**（★ 前向 trie，最长匹配）：

```cpp
/* ① 位图快速过滤（只用低 12 位 → 4096 字节） */
#define MY_UCA_CNT_FLAG_SIZE 4096
#define MY_UCA_CNT_FLAG_MASK 4095
#define MY_UCA_CNT_HEAD  1
#define MY_UCA_CNT_TAIL  2
#define MY_UCA_CNT_MID1  4      /* MID2=8 / MID3=16 / MID4=32，注释里说 MID5 不存在
                                   （否则字节位不够） */
#define MY_UCA_PREVIOUS_CONTEXT_HEAD  64
#define MY_UCA_PREVIOUS_CONTEXT_TAIL 128

static inline void my_uca_add_contraction_flag(char *flags, my_wc_t wc, int flag) {
  flags[wc & MY_UCA_CNT_FLAG_MASK] |= flag;
}

/* ② child_nodes 是有序 vector → 二分查找 */
static std::vector<MY_CONTRACTION>::const_iterator find_contraction_part_in_trie(
    const std::vector<MY_CONTRACTION> &cont_nodes, my_wc_t ch) {
  if (cont_nodes.empty()) return cont_nodes.end();
  return std::lower_bound(cont_nodes.begin(), cont_nodes.end(), ch, trie_node_cmp());
}

/* ③ 前向最长匹配（contraction_find） */
const uint16 *my_uca_scanner::contraction_find(my_wc_t wc0, size_t *chars_skipped) {
  const uchar *beg = nullptr;
  auto mb_wc = cs->cset->mb_wc;
  const uchar *s = sbeg;
  const std::vector<MY_CONTRACTION> *cont_nodes = uca->contraction_nodes;
  const MY_CONTRACTION *longest_contraction = nullptr;
  for (;;) {
    node_it = find_contraction_part_in_trie(*cont_nodes, wc0);
    if (node_it == cont_nodes->end() || node_it->ch != wc0) break;
    if (node_it->is_contraction_tail) {
      longest_contraction = &(*node_it);   /* ★★ 遇到 tail 只记录，不停止，继续尝试更长 */
      beg = s;
      *chars_skipped = node_it->contraction_len - 1;
    }
    int mblen;
    if ((mblen = mb_wc(cs, &wc0, s, send)) <= 0) break;
    s += mblen;
    cont_nodes = &node_it->child_nodes;
  }
  if (longest_contraction != nullptr) {
    const uint16 *cweight = longest_contraction->weight;
    if (uca->version == UCA_V900) {
      cweight += weight_lv;                /* ★ 900：先跳到当前级的第一个权重 */
      wbeg = cweight + MY_UCA_900_CE_SIZE;
      wbeg_stride = MY_UCA_900_CE_SIZE;
      num_of_ce_left = 7;
    } else {
      wbeg = cweight + 1;                  /* ★ 4.0：跳过 CE 个数 */
      wbeg_stride = MY_UCA_900_CE_SIZE;
    }
    sbeg = beg;                            /* ★★ 输入指针一次性推进整个契约长度 */
    return cweight;
  }
  return nullptr;
}
```

> ★★ **"最长匹配"是理解 contraction 的关键**：匈牙利语里 `DZ` 和 `DZS` 都是契约，所以遇到 `D`→`Z`(是 tail，记录) 后**不能停**，要继续看 `Z`→`S`；只有走不动了才用最后记录的那个。
> ★ **前文语境契约（previous context）是"逆序存放"的**：`add_contraction_to_trie()` 里先按"当前字符"建节点，再在 `child_nodes_context` 子树里按"前一字符"建节点。CLDR 只支持 2 字符的 previous context。

**★ expansion（一字符贡献多个 CE）**：

```cpp
/* 900 版：CE 个数来自 UCA900_NUM_OF_CE，逐个 CE 的 3 个 uint16 一起拷 */
static size_t my_char_weight_put_900(MY_UCA_INFO *dst, uint16 *to, size_t to_stride,
                                     size_t to_length, uint16 *to_num_ce,
                                     const MY_COLL_RULE *rule, size_t base_len) {
  const my_wc_t *base = rule->base;
  for (count = 0; base_len;) {
    /* 先尝试最长契约 */
    for (size_t chlen = base_len; chlen > 1; chlen--) {
      if ((from = my_uca_contraction_weight(dst->contraction_nodes, base, chlen))) {
        from_stride = 1; base += chlen; base_len -= chlen;
        ce_cnt = *(from + MY_UCA_MAX_WEIGHT_SIZE - 1);
        break;
      }
    }
    if (!from) {
      uint page = *base >> 8, code = *base & 0xFF;
      base++; base_len--;
      if (dst->weights[page]) {
        from = UCA900_WEIGHT_ADDR(dst->weights[page], 0, code);
        from_stride = UCA900_DISTANCE_BETWEEN_LEVELS;
        ce_cnt = UCA900_NUM_OF_CE(dst->weights[page], code);   /* ★★ expansion 长度 */
      }
    }
    for (int weight_ind = 0; weight_ind < ce_cnt * MY_UCA_900_CE_SIZE && count < to_length; weight_ind++) {
      *to = *from; to += to_stride; from += from_stride; count++;
    }
    total_ce_cnt += ce_cnt;
  }
  /* ★ 追加一个"额外 CE" 来容纳 primary shift（僧伽罗语 &ඥ < ඤ 那类规则） */
  if ((rule->diff[0] || rule->diff[1] || rule->diff[2]) && count < to_length) {
    *to = rule->diff[0] ? dst->extra_ce_pri_base : 0; to += to_stride;
    *to = rule->diff[1] ? dst->extra_ce_sec_base : 0; to += to_stride;
    *to = rule->diff[2] ? dst->extra_ce_ter_base : 0; to += to_stride;
    total_ce_cnt++; count += 3;
  }
  ...
}

/* 4.0 版：权重串以 0 结尾，逐个拷到 0 —— expansion 天然支持 */
static size_t my_char_weight_put(...) {
  for (; from && *from && count < to_length;) {
    *to = *from++; to += to_stride; count++;
  }
  *to = 0;
}
```

> ★★ **"额外 CE"的设计动机**（源码长注释）：老实现用"伪 expansion"（把 U+0DA4 展开成 U+0DA5 + `last_non_ignorable`）来做 primary shift，但 900 支持 as_cs 后，多出来的 secondary/tertiary 权重会干扰"从 tertiary 推断大小写"的算法。所以改成**追加一个单独的 CE**，其基准是 `extra_ce_pri_base`（通常 0x54A4 = 最大常规权重 0x54A3 + 1）。中文排序还专门为 reorder 改了这个基准（`change_zh_implicit()`）。

**★ 运行期消费 expansion**（`more_weight()`）：

```cpp
template <class Mb_wc, int LEVELS_FOR_COMPARE>
inline int uca_scanner_900<Mb_wc, LEVELS_FOR_COMPARE>::more_weight() {
  /* Check if the weights for the previous code point have been
     already fully scanned. If no, return the first non-zero weight. */
  while (num_of_ce_left != 0 && *wbeg == 0) {
    wbeg += wbeg_stride; --num_of_ce_left;
  }
  if (num_of_ce_left != 0) {
    uint16 rtn = *wbeg;
    wbeg += wbeg_stride; --num_of_ce_left;
    return rtn;         /* ★ 返回 expansion 里的下一个权重 */
  }
  return -1;
}
```

> ★ 4.0 的对应逻辑只有一行：`if (wbeg[0]) return *wbeg++;`（权重串以 0 结尾）。

#### 扫描器：逐级比较的核心

```c
// my_strnncoll_uca 核心骨架（旧版单级，逻辑与 900 分级版同构）
for (uint current_lv = 0; current_lv < LEVELS_FOR_COMPARE; ++current_lv) {
  do {
    s_res = sscanner.next();
    t_res = tscanner.next();
  } while (s_res == t_res && s_res >= 0 &&
           sscanner.get_weight_level() == current_lv &&
           tscanner.get_weight_level() == current_lv);
  if (sscanner.get_weight_level() == tscanner.get_weight_level()) {
    if (s_res == t_res && s_res >= 0) continue;  // 本级相等，进下一级
    break;                                       // 本级分出大小
  }
  if (tscanner.get_weight_level() > current_lv) {
    // t 在本级没有权重了（较短）：s 剩余权重全是"多出来的"
    if (t_is_prefix) continue;  // 前缀匹配场景：吃掉 s 的剩余
    return 1;                   // 否则 s > t
  }
  if (sscanner.get_weight_level() > current_lv) return -1;
  break;
}
return (s_res - t_res);
```

逐段解释：

- **两级循环**：外层 `current_lv` 走权重级别（primary→secondary→tertiary），内层 `next()` 在同一级内逐个取权重比。这是 TR10 的核心：**只有在 primary 全相等时才看 secondary，secondary 全相等才看 tertiary**。
- **`get_weight_level() > current_lv` 的语义**：扫描器把一侧的某级权重耗尽后，`next()` 返回"我已进入下一级"——于是外层循环发现"一侧在本级没权重了"。若此时另一侧还有本级权重，短的赢了（NO PAD 语义：多出来的字符算数）；PAD SPACE 版本则在这里插入"空格权重 vs 剩余权重"的补齐比较。
- **`is_prefix` 分支**：`strnncoll` 的 `is_prefix=true` 用于索引前缀匹配（"t 是 s 的前缀吗"），此时 s 多出来的尾巴不算数。

`uca_scanner_900::next()` 是扫描器的核心：

```c
// 骨架（含多个定制分支）
int next() {
  my_wc_t wc;
  int res = m_mb_wc(&wc, ...);          // 取下一个码点
  if (res < 0) return res;              // 出错/耗尽

  if (my_uca_have_contractions) {
    if (my_uca_can_be_contraction_tail(m_wc2)) {  // 检查"前一字符+当前字符"组合
      const MY_CONTRACTION *c = previous_context_find(m_wc2, wc);  // ja 专用
      if (c) { ...应用 c 的权重，返回... }
    }
    const MY_CONTRACTION *c = contraction_find(wc, &chars_skipped);  // 头字 trie
    if (c) { ...应用 c 的权重，跳过 chars_skipped 个字符，返回... }
  }

  if (wc 未赋权) return next_implicit(wc);  // implicit weight

  page = wc >> 8;
  wpage = uca->weights[page];              // 页指针表
  wbeg = UCA900_WEIGHT_ADDR(wpage, weight_lv, wc & 0xFF);
  // 跳过 0 权重（ignorable），同一字符有多个 CE 时按 stride 3*256 步进
  ...
  return *wbeg;
}
```

四层决策链：**previous-context contraction（ja）→ 头字 contraction → implicit weight → 常规页查表**。contraction 用 `contraction_flags[ch & 0x1000]` 位图快速排除"不可能构成 contraction 的字符"（HEAD/TAIL/MID 位），命中才走 trie 下钻——**位图是热路径上的第一道筛子**。

##### 扫描器的返回值协议：`>0` / `0` / `-1` 各表示什么

`next()` 的三种返回值是整个比较框架的接缝，读源码前必须先分清：

| 返回值 | 含义 | 比较循环怎么处理 |
|---|---|---|
| `> 0` | 一个权重值 | 正常参与比较 |
| **`0`** | **层分隔符**（"本级已耗完，下一级开始"） | ✅ 关键：**不进比较**，只用于推进级数 |
| `-1` | 字符串读完，或遇到非法多字节序列 | 循环结束条件 |

`0` 的根源在 900 扫描器的层切换代码里：

```c
if (mblen <= 0) {                                   // 本串在本级读完
  if (LEVELS_FOR_COMPARE == 1) { ++weight_lv; return -1; }
  if (++weight_lv < LEVELS_FOR_COMPARE) {
    if (LEVELS_FOR_COMPARE == 4 && cs->coll_param == &ja_coll_param) {
      if (weight_lv == 3 && !has_quaternary_weight) return -1;   // 日文：无假名则跳过第四级
    }
    sbeg = sbeg_dup;                                // 回到串首，重扫下一级
    return 0;                                       // ★ 层分隔符
  }
  return -1;
}
```

两个要点：① **级切换靠"重置 sbeg 从头重扫"实现**——不是记住每级的权重位置，而是重扫一遍取下一级权重（`sbeg_dup` 保存串首）——这正是 `strnxfrm` 输出里那个 `0x0000` 级分隔符的来源；② 比较循环只对 `> 0` 的结果做变换（reorder/case_first），`0` 不会被误改。另外源码注释明确说 `-1` 同时覆盖"读完"与"非法序列"两种情况——**比较层面不区分二者**（都当"这串到此为止"）。

##### expansion：一个字符贡献多个 CE（`ß` = `ss` 的数据基础）

900 权重页的布局不是"一字符三权重"，而是 **`[256 个 CE 计数][CE0 三级][CE1 三级][CE2 三级]…`**：

```
页（256 个字符）
├── 偏移 0..255      : 每个字符的 CE 计数
├── 偏移 256..511    : 所有字符的 primary 权重（第 0 个 CE）
├── 偏移 512..767    : 所有字符的 secondary 权重（第 0 个 CE）
├── 偏移 768..1023   : 所有字符的 tertiary 权重（第 0 个 CE）
├── 偏移 1024..      : 第 1 个 CE 的三级权重（同样三段）
└── …                : 第 N 个 CE
```

```c
#define UCA900_NUM_OF_CE(page, subcode) ((page)[(subcode)])
#define UCA900_WEIGHT_ADDR(page, level, subcode) ((page) + 256 + (level) * 256 + (subcode))
#define UCA900_DISTANCE_BETWEEN_LEVELS 256
#define UCA900_DISTANCE_BETWEEN_WEIGHTS (MY_UCA_900_CE_SIZE * 256)   // CE 间步进 = 768
```

**"一变多"（expansion）就是这个计数 > 1**——没有单独的展开表，多出来的 CE 就排在同一个字符的后续 CE 槽里。扫描器消费方式是"计数 + 迭代器"：

```c
num_of_ce_left = UCA900_NUM_OF_CE(wpage, code);
// 之后每次 next() 沿 wbeg += 768 步进，边吐边跳"本层为 0"的 CE：
while (num_of_ce_left != 0 && *wbeg == 0) { wbeg += wbeg_stride; --num_of_ce_left; }
if (num_of_ce_left != 0) { uint16 rtn = *wbeg; wbeg += wbeg_stride; --num_of_ce_left; return rtn; }
```

**`ß`（U+00DF）的真实数据**（3 个 CE）：

```
CE0: [1E71 . 0020 . 0004]
CE1: [0000 . 0110 . 0004]      ← primary 为 0（不参与一级比较）
CE2: [1E71 . 0020 . 0004]
```

而 `'s'` 的 primary 也是 **`0x1E71`** → 一级层面 `ß` 的权重序列 = `s` 的权重序列 `+ s` 的权重序列 ⇒ **`ß = ss`**。中间那个 CE 的 secondary `0x0110`（属 DUCET 的 "variant secondaries 0110..0114"）让它与两个独立 `s` 在二级/三级层面可分——这就是"一级相等、二级不等"的数据表现。

对照 `general_ci`：它根本不用这张表，`ctype-utf8.cc` 里把 `ß` 的排序权重直接写成 `{0x00DF, 0x00DF, 0x0053}`（第三字段 = `'S'` 的码点）⇒ **`ß = s`**。**同一个字符在两套 collation 下与不同长度的串相等，全部来自数据而非特例代码。**

相关常量：`MY_UCA_MAX_EXPANSION = 6`（规则串里的展开上限）、`MY_UCA_MAX_WEIGHT_SIZE = 25`（= 8 个 CE × 3 权重 + 1 个计数槽）、`MY_UCA_MAX_CONTRACTION = 6`。dump 侧另有 `MY_UCA_MAXCE_TO_DUMP = 8`——因为 DUCET 里最长的 CE 序列属于 `U+FDFA`（阿拉伯语连字，**18 个 CE**），MySQL 只取前 8 个（注释：8 个足以区分 `U+FDFA` 与其他字符）。

**expansion 与 contraction 共用同一个结构** `MY_CONTRACTION`——两者都是"多字符/多 CE 的权重串"，只是一个按"单字符多 CE"索引、一个按"多字符组合"索引。

##### Hangul 音节分解：implicit weight 的一条特殊支路

韩文音节（U+AC00–U+D7AF）**不在 DUCET 表里**。若走通用 implicit weight 分页，它们会落到 `0xFB40/0xFB80/0xFBC0` 区——**权重值远超所有显式字符**，整个韩文会排到 CJK 后面。所以扫描器对这段区间做了算术分解：

```c
#define HANGUL_JAMO_MAX_LENGTH 3
static int my_decompose_hangul_syllable(my_wc_t syllable, my_wc_t *jamo) {
  if (syllable < 0xAC00 || syllable > 0xD7AF) return 0;
  constexpr uint leadingjamo_base = 0x1100;   // 初声 L
  constexpr uint voweljamo_base   = 0x1161;   // 中声 V
  constexpr uint trailingjamo_base= 0x11A7;   // 终声 T
  constexpr uint voweljamo_cnt = 21, trailingjamo_cnt = 28;
  const uint syllable_index = syllable - 0xAC00;
  jamo[0] = leadingjamo_base  + syllable_index / (voweljamo_cnt * trailingjamo_cnt);
  jamo[1] = voweljamo_base    + (syllable_index % (voweljamo_cnt * trailingjamo_cnt)) / trailingjamo_cnt;
  jamo[2] = trailingjamo_index ? (trailingjamo_base + trailingjamo_index) : 0;   // 可能无终声
  return trailingjamo_index ? 3 : 2;
}
```

分解出的每个 jamo **查表拿自己的 CE0 三级权重**（不是纯公式），塞进扫描器的 `implicit[10]` 缓冲（`implicit[9]` 存 jamo 个数），随后按 stride=3 吐出。**混合式**：位置靠算术、权重靠查表。

效果：韩文音节按 jamo 在 DUCET 里的真实位置排序（与单独输入的 jamo 一致），而不是按码点。注意这是 **implicit weight 内部的例外支路**——只在 `uca->weights[page] == nullptr`（整页未加载）时才会走到这里。

##### 扫描器的分派不是函数指针数组

旧印象里"扫描器靠函数指针数组分派"在 8.0 已不是事实。实际是**模板 + 两种取码点策略**：

```c
// 两种 Mb_wc：无状态 functor（可内联）vs 函数指针包装
if (cs->cset->mb_wc == my_mb_wc_utf8mb4_thunk) {
  return my_strnncoll_uca<uca_scanner_900<Mb_wc_utf8mb4, LEVELS>, LEVELS>(cs, Mb_wc_utf8mb4(), ...);
}
Mb_wc_through_function_pointer mb_wc(cs);      // 缓存 cs->cset->mb_wc 的包装类
return my_strnncoll_uca<uca_scanner_900<decltype(mb_wc), LEVELS>, LEVELS>(cs, mb_wc, ...);
```

`Mb_wc` 作为模板参数传进去的作用是**把取码点调用内联进扫描循环**——utf8mb4 走专用实例化（`my_mb_wc_utf8mb4` 直接内联），其他字符集走函数指针包装。外层再按 `cs->levels_for_compare` 用 switch 实例化 1..4 级的版本。设计意图写在 `strings/mb_wc.h` 的注释里："compare `cs->cset->mb_wc` with `my_mb_wc_{utf8mb3,utf8mb4}_thunk`, and if so, instantiate your function with the given class … otherwise use `Mb_wc_through_function_pointer`"。

**ASCII 快路径在 `for_each_weight()` 里**（不在 `next()`）：

```c
if (cs->tailoring || cs->mbminlen != 1 || cs->coll_param) {
  /* slower, generic path */
} else {
  const uint16 *ascii_wpage = UCA900_WEIGHT_ADDR(uca->weights[0], weight_lv, 0);
  ... // 每 4 字节判定是否都在 0x20..0x7E
  if (((four_bytes + 0x01010101u) & 0x80808080) || ((four_bytes - 0x20202020u) & 0x80808080)) break;
  func(s_res0, false); func(s_res1, false); func(s_res2, false); func(s_res3, false);
  if (s_res == 0) ascii_wpage += UCA900_DISTANCE_BETWEEN_LEVELS;   // 层分隔符 → 切到下一级的块
}
```

三个禁用条件（有 tailoring / 非单字节最小字符 / 有 coll_param）——因为这三类 collation 的权重可能被改写，不能直接读 ASCII 页。快路径消费者是 `my_hash_sort_uca_900` 与 `my_strnxfrm_uca_900`（比较走 `next()` 不走这里）。

#### implicit weight：DUCET 没覆盖的码点

```c
// next_implicit 骨架
page = ch >> 15;
implicit[3] = (ch & 0x7FFF) | 0x8000;
if (CJK 扩展区) page += 0xFB80;
else if (U+4E00..U+9FD5 / U+FA0E..) page += 0xFB40;   // CJK 统一表意
else page += 0xFBC0;
implicit[0]=page; implicit[1]=0x0020; implicit[2]=0x0002;
```

规则来自 TR10 7.1.3：未赋权码点按"码点自身"派发权重——primary=按码点区间偏移的页值、secondary=0x0020、tertiary=0x0002。效果是**所有未收录字符自动按码点排序**，新 Unicode 字符不需要等 MySQL 升级权重表就能有确定序。zh 定制经 `change_zh_implicit` 把 0xFBxx 区间压缩到 0xBDCx/0xF62x，**消除权重空洞**——否则 zh 拼音序里未收录汉字的权重会插不进已排好的表意文字序列。

**但这条规则只在 900 系列成立**。UCA 4.0（unicode_ci 系列）的扫描器里有一道硬门槛：

```c
// uca_scanner_any::next() 里的检查
if (wc > uca->maxchar) {
  /* Return 0xFFFD as weight for all characters outside BMP */
  wbeg = nochar;
  wbeg_stride = 0;
  return 0xFFFD;
}
```

`uca->maxchar` 是 U+FFFF——**UCA 4.0 只能为 BMP 字符派发权重，非 BMP（emoji、CJK 扩展、音乐符号等）全部返回同一个 0xFFFD**。后果是一个隐蔽的坑：**utf8mb4_unicode_ci / utf8mb4_general_ci 下，所有辅助平面字符彼此相等**（权重相同 → 比较相等、排序相邻）。900 系列没有这道门槛（next_implicit 对所有码点派发），所以 8.0 升级到 0900 collation 后，非 BMP 字符才第一次有了可区分的顺序。

**官方文档对这条规则的确认**（[Unicode Character Sets](https://dev.mysql.com/doc/mysql-g11n-excerpt/8.0/en/charset-unicode-sets.html)）："For supplementary characters in UCA 4.0.0 collations, their collating weight is `0xfffd`. That is, to MySQL, all supplementary characters are equal to each other, and greater than almost all BMP characters." 文档还给了两个可复现的例子——Deseret 字符（`COUNT(DISTINCT)` 得到 2 而非 3）、楔形文字（`WEIGHT_STRING` 得到 `0E33 FFFD FFFD 0E4A`，两个楔形字符权重都是 FFFD）。

##### 源码与官方文档的一处偏差（CJK 区间上界）

官方给出的 implicit weight 算法是：

```
if (code >= 0x3400 && code <= 0x4DB5)      base = 0xFB80;  /* CJK 扩展 */
else if (code >= 0x4E00 && code <= 0x9FA5) base = 0xFB40;  /* CJK 统一表意 */
else                                        base = 0xFBC0;
```

但 8.0.39 源码里**两个版本的 CJK 上界不同**：

| 实现 | CJK 上界 | 位置 |
|---|---|---|
| `uca_scanner_900::next_implicit` | **0x9FD5**（另含 0xFA0E..0xFA29 兼容表意文字） | `ctype-uca.cc` |
| `uca_scanner_any::next_implicit`（4.0） | **0x9FA5** | 同上 |

即**官方文档描述的是 UCA 4.0 时代的旧上界**；900 实现已随 Unicode 版本把上界推进到 `0x9FD5`（Unicode 9.0 的 CJK 统一表意文字扩展到了 U+9FD5）。这类"文档滞后于源码"在字符集模块并不罕见——**涉及精确区间/版本细节时，以源码为准**。

**★ `next_implicit()` 两版完整源码**：

```cpp
/* strings/ctype-uca.cc:1175 —— 900 版（含 Tangut 与谚文音节分解） */
template <class Mb_wc, int LEVELS_FOR_COMPARE>
ALWAYS_INLINE int uca_scanner_900<Mb_wc, LEVELS_FOR_COMPARE>::next_implicit(my_wc_t ch) {
  my_wc_t hangul_jamo[HANGUL_JAMO_MAX_LENGTH];
  int jamo_cnt;
  if ((jamo_cnt = my_decompose_hangul_syllable(ch, hangul_jamo))) {
    my_put_jamo_weights(hangul_jamo, jamo_cnt);      /* ★★ 谚文音节：算法分解成 2~3 个 jamo */
    num_of_ce_left = jamo_cnt - 1;
    wbeg = implicit + MY_UCA_900_CE_SIZE + weight_lv;
    wbeg_stride = MY_UCA_900_CE_SIZE;
    return *(implicit + weight_lv);
  }

  uint page;
  if (ch >= 0x17000 && ch <= 0x18AFF) {              /* ★ Tangut（西夏文）特例 */
    page = 0xFB00;
    implicit[3] = (ch - 0x17000) | 0x8000;
  } else {
    page = ch >> 15;
    implicit[3] = (ch & 0x7FFF) | 0x8000;            /* ★ BBBB = (CP & 0x7FFF) | 0x8000 */
    if ((ch >= 0x3400 && ch <= 0x4DB5) || (ch >= 0x20000 && ch <= 0x2A6D6) ||
        (ch >= 0x2A700 && ch <= 0x2B734) || (ch >= 0x2B740 && ch <= 0x2B81D) ||
        (ch >= 0x2B820 && ch <= 0x2CEA1))
      page += 0xFB80;                                /* CJK 扩展 B~F */
    else if ((ch >= 0x4E00 && ch <= 0x9FD5) || (ch >= 0xFA0E && ch <= 0xFA29))
      page += 0xFB40;                                /* ★ CJK 基本区 + 兼容表意文字（上界 0x9FD5） */
    else
      page += 0xFBC0;                                /* 其它未分配码点 */
  }
  if (cs->coll_param == &zh_coll_param) page = change_zh_implicit(page);

  implicit[0] = page;      /* primary   */
  implicit[1] = 0x0020;    /* secondary */
  implicit[2] = 0x0002;    /* tertiary  */
  implicit[4] = 0; implicit[5] = 0;
  num_of_ce_left = 1;
  wbeg = implicit + MY_UCA_900_CE_SIZE + weight_lv;
  wbeg_stride = MY_UCA_900_CE_SIZE;
  return *(implicit + weight_lv);
}
```

```cpp
/* strings/ctype-uca.cc:1259 —— 4.0/5.2 版（只有 primary，无谚文分解） */
template <class Mb_wc>
ALWAYS_INLINE int uca_scanner_any<Mb_wc>::next_implicit(my_wc_t ch) {
  implicit[0] = (ch & 0x7FFF) | 0x8000;
  implicit[1] = 0;
  wbeg = implicit;
  wbeg_stride = MY_UCA_900_CE_SIZE;

  uint page = ch >> 15;
  if (ch >= 0x3400 && ch <= 0x4DB5)      page += 0xFB80;
  else if (ch >= 0x4E00 && ch <= 0x9FA5) page += 0xFB40;   /* ★ 上界 0x9FA5 */
  else                                   page += 0xFBC0;
  return page;
}
```

```cpp
/* 谚文音节 → jamo 的纯算法分解（无查表） */
static int my_decompose_hangul_syllable(my_wc_t syllable, my_wc_t *jamo) {
  if (syllable < 0xAC00 || syllable > 0xD7AF) return 0;
  constexpr uint syllable_base = 0xAC00, leadingjamo_base = 0x1100;
  constexpr uint voweljamo_base = 0x1161, trailingjamo_base = 0x11A7;
  constexpr uint voweljamo_cnt = 21, trailingjamo_cnt = 28;
  const uint syllable_index = syllable - syllable_base;
  const uint v_t_combination = voweljamo_cnt * trailingjamo_cnt;
  jamo[0] = leadingjamo_base + syllable_index / v_t_combination;
  jamo[1] = voweljamo_base + (syllable_index % v_t_combination) / trailingjamo_cnt;
  const uint trailingjamo_index = syllable_index % trailingjamo_cnt;
  jamo[2] = trailingjamo_index ? (trailingjamo_base + trailingjamo_index) : 0;
  return trailingjamo_index ? 3 : 2;      /* ★ 无尾音时只返回 2 个 jamo */
}
```

> ★★ **谚文是"算法分解"而非查表**——U+AC00..U+D7AF 的 11172 个音节一个都不进权重表，全部靠 `my_decompose_hangul_syllable()` 拆成 2~3 个 jamo，再各自查真实权重。这是 DUCET 标准规定的做法（Hangul Syllable 的权重 = 其 jamo 权重的串联）。
> ★ `change_zh_implicit()` 把 0xFB00/0xFB40/0xFB41/0xFB80/0xFB84/0xFB85 重映射到 0xF621/0xBDBF/0xBDC0/0xBDC1/0xBDC2/0xBDC3，其余 `weight + 0xF622 - 0xFBC0`——**目的是在 zh 拼音序里腾出连续权重区间**，让 Latin/Cyrillic 等字符组 reorder 后仍能保持"一个字符组一个主权重"。

**★ 扫描器基类与两个模板子类**（★ `next()` 的差异就是两版算法的分水岭）：

```cpp
class my_uca_scanner {          /* 抽象基类，不直接实例化 */
 protected:
  uint weight_lv{0};            /* 0=Primary, 1=Secondary, 2=Tertiary */
  const uint16 *wbeg;           /* 当前权重串起点 */
  uint wbeg_stride{0};          /* 权重间字节步长 */
  const uchar *sbeg, *send;
  const MY_UCA_INFO *uca;
  uint16 implicit[10];
  my_wc_t prev_char{0};         /* ★ 上一个码点，用于 previous-context 契约 */
  uint num_of_ce_left{0};       /* ★ 900 专有：该码点还剩几个 CE */
  const uchar *sbeg_dup;        /* ★ 900 专有：串起点备份，用于"从头重扫下一级" */
 public:
  uint get_weight_level() const { return weight_lv; }
 protected:
  const uint16 *contraction_find(my_wc_t wc0, size_t *chars_skipped);
  inline const uint16 *previous_context_find(my_wc_t wc0, my_wc_t wc1);
};
```

```cpp
/* 4.0/5.2：串结束 → 直接进下一级，不重启、不返回级分隔符 */
template <class Mb_wc>
ALWAYS_INLINE int uca_scanner_any<Mb_wc>::next() {
  if (wbeg[0]) return *wbeg++;              /* ★ 还有 expansion 权重 */
  do {
    int mblen = mb_wc(&wc, sbeg, send);
    if (mblen <= 0) { ++weight_lv; return -1; }   /* ★★ 不重启 */
    sbeg += mblen; char_index++;
    if (wc > uca->maxchar) { wbeg = nochar; wbeg_stride = 0; return 0xFFFD; }
    ... contraction / previous-context ...
    const uint16 *wpage = uca->weights[page];
    if (!wpage) return next_implicit(wc);
    wbeg = wpage + code * uca->lengths[page];
    wbeg_stride = UCA900_DISTANCE_BETWEEN_WEIGHTS;
  } while (!wbeg[0]);
  return *wbeg++;
}
```

```cpp
/* 900：串结束 → 返回 0 作级分隔符，并把 sbeg 拨回起点重扫下一级 */
template <class Mb_wc, int LEVELS_FOR_COMPARE>
ALWAYS_INLINE int uca_scanner_900<Mb_wc, LEVELS_FOR_COMPARE>::next_raw() {
  int remain_weight = more_weight();
  if (remain_weight >= 0) return remain_weight;
  do {
    int mblen = mb_wc(&wc, sbeg, send);
    if (mblen <= 0) {
      if (LEVELS_FOR_COMPARE == 1) { ++weight_lv; return -1; }
      if (++weight_lv < LEVELS_FOR_COMPARE) {
        if (LEVELS_FOR_COMPARE == 4 && cs->coll_param == &ja_coll_param)
          if (weight_lv == 3 && !has_quaternary_weight) return -1;  /* ★ 日文快路径 */
        sbeg = sbeg_dup;                    /* ★★ 拨回起点 */
        return 0;                           /* ★★ 级分隔符 */
      }
      return -1;
    }
    sbeg += mblen;
    ... contraction / previous-context / ja 四级 ...
    const uint16 *wpage = uca->weights[page];
    if (!wpage) return next_implicit(wc);
    wbeg = UCA900_WEIGHT_ADDR(wpage, weight_lv, code);
    wbeg_stride = UCA900_DISTANCE_BETWEEN_WEIGHTS;
    num_of_ce_left = UCA900_NUM_OF_CE(wpage, code);
  } while (!wbeg[0]);
  uint16 rtn = *wbeg; wbeg += wbeg_stride; --num_of_ce_left;
  return rtn;
}

template <class Mb_wc, int LEVELS_FOR_COMPARE>
ALWAYS_INLINE int uca_scanner_900<Mb_wc, LEVELS_FOR_COMPARE>::next() {
  int res = next_raw();
  Coll_param *param = cs->coll_param;
  if (res > 0 && param) {
    if (param->reorder_param && weight_lv == 0) res = apply_reorder_param(res);  /* ★ 只在一级 */
    if (param->case_first != CASE_FIRST_OFF)    res = apply_case_first(res);
  }
  return res;
}
```

**★ `for_each_weight()` 的 ASCII 批量快路径**（★ 一次处理 4 字节）：

```cpp
template <class Mb_wc, int LEVELS_FOR_COMPARE>
template <class T, class U>
ALWAYS_INLINE void uca_scanner_900<Mb_wc, LEVELS_FOR_COMPARE>::for_each_weight(T func, U preaccept_data) {
  if (cs->tailoring || cs->mbminlen != 1 || cs->coll_param) {
    /* Slower, generic path. TODO: See if we can accept some character sets with tailorings. */
    int s_res;
    while ((s_res = next()) >= 0) if (!func(s_res, s_res == 0)) return;
    return;
  }
  ...
  const uchar *send_local = (send - sbeg > 3) ? (send - 3) : sbeg;  /* 留 3 字节安全余量 */
  for (;;) {
    while ((s_res = more_weight()) >= 0) if (!func(s_res, s_res == 0)) return;
    const uchar *sbeg_local = sbeg;
    while (sbeg_local < send_local && preaccept_data(sizeof(uint32))) {
      uint32 four_bytes;
      memcpy(&four_bytes, sbeg_local, sizeof(four_bytes));
      if (((four_bytes + 0x01010101u) & 0x80808080) ||      /* ★ 任一键 > 0x7E */
          ((four_bytes - 0x20202020u) & 0x80808080)) break; /* ★ 任一键 < 0x20 */
      /* Check if all four bytes are in the range 0x20..0x7e, inclusive.
         These have exactly one weight. Note that this unfortunately does not
         include tab and newline, which would otherwise be legal candidates. */
      const int s_res0 = ascii_wpage[sbeg_local[0]];
      ... 4 次 func() ...
      sbeg_local += sizeof(uint32);
    }
    sbeg = sbeg_local;
    s_res = next_raw();
    if (s_res == 0) ascii_wpage += UCA900_DISTANCE_BETWEEN_LEVELS;  /* ★ 级切换要换页指针 */
    if (s_res < 0 || !func(s_res, s_res == 0)) return;
  }
}
```

> ★★ **快路径的三个准入门槛**：无 tailoring、`mbminlen == 1`、无 `coll_param`。所以 **0900_ai_ci（默认）能用，但 `de_pb_0900_ai_ci` 这类带 tailoring 的用不了**。
> ★ 范围检查用两行位运算一次搞定 4 字节：`(x + 0x01010101) & 0x80808080` 判上界（>0x7E）、`(x - 0x20202020) & 0x80808080` 判下界（<0x20）。注释特别提醒**制表符与换行不在范围内**。

**★ `my_strnxfrm_uca_900` 与 4.0 版的关键差异**（★ NO PAD 补的是 0 不是空格权重）：

```cpp
/* 4.0（PAD SPACE）：不足部分用"空格权重"补齐 */
    if (num_codepoints) {
      uint space_count = std::min<uint>((de - dst) / 2, num_codepoints);
      s_res = my_space_weight(cs);
      for (; space_count; space_count--) dst = store16be(dst, s_res);   /* ★ 空格权重 */
    }
  if ((flags & MY_STRXFRM_PAD_TO_MAXLEN) && dst < de) {
    s_res = my_space_weight(cs);
    for (; dst < de;) { *dst++ = s_res >> 8; if (dst < de) *dst++ = s_res & 0xFF; }
  }

/* 900（NO PAD）：★ PAD_TO_MAXLEN 时补的是 0x00 */
  if (flags & MY_STRXFRM_PAD_TO_MAXLEN) {
    memset(dst, 0, dst_end - dst);        /* ★★ 补 0，不是空格权重 */
    dst = dst_end;
  }
```

```cpp
/* 900 的排序键长度估算（ctype-uca.cc:5125） */
static size_t my_strnxfrmlen_uca_900(const CHARSET_INFO *cs, size_t len) {
  /* The character with the most weights is U+FDFA ARABIC LIGATURE SALLALLAHOU
     ALAYHE WASALLAM, which we truncate to eight weights. ...
     We also need to add room for one level separator between each level. */
  const size_t num_codepoints = (len + 3) / 4;
  const size_t max_num_weights_per_level = num_codepoints * 8;
  size_t max_num_weights = max_num_weights_per_level * cs->levels_for_compare;
  if (cs->coll_param && cs->coll_param->reorder_param)
    max_num_weights += max_num_weights_per_level;    /* reorder 可能多加一级权重 */
  return (max_num_weights + (cs->levels_for_compare - 1)) * sizeof(uint16_t);
}
```

> ★ `U+FDFA`（阿拉伯语"愿主福安之"连字）是**权重最多的字符**，被截断到 8 个权重——这就是 `num_codepoints * 8` 这个系数的来源。

#### general_ci 的本质：还原后按码点比

`utf8mb4_general_ci` 常被描述为"unicode_ci 的简化版"，代码层面的本质是：它**根本不查 UCA 权重表**，而是把字符做 Unicode 大小写还原后**直接用码点数值比较**：

```c
// my_strnncoll_utf8mb4_general 的比较循环
while ((res = my_mb_wc_utf8mb4(&wc, s, e)) > 0) {
  my_tosort_unicode(uni_plane, &wc, cs->state);   // 大小写/分解还原
  ... // 直接比还原后的码点数值
}
```

后果：'ß' 被还原成 's'（码点 0x0053），所以 **general_ci 下 'ß' = 's'**；而 unicode_ci 走 UCA 4.0 权重表，'ß' 的权重（0x0FEA0FEA）恰好等于 'ss' 的权重，所以 **unicode_ci 下 'ß' = 'ss'**。同一个字符在两个 collation 下与不同的串相等——这是"简化权重"与"真权重"的分水岭。

#### LIKE 与 = 的分野：一对一 vs 一对多

顺带点破一个反直觉规则：`=`/`<`/`>` 走 Sort Key 比较，允许"一对多相等"（unicode_ci 下 'ß' = 'ss'，两个字符等于一个字符）；但 **LIKE 只允许一对一字符匹配**——所以 unicode_ci 下 `'ß' LIKE 'ss'` 和 `'ß' LIKE 's'` 都返回 0，而 general_ci 下 `'ß' LIKE 's'` 返回 1（字符级一对一相等）。`=` 和 `LIKE` 在同一 collation 下给出不同真值，这是理解 collation 语义时必须分清的两条比较路径。

#### utf8mb4_0900_bin：字节直比的合法优化

`*_bin` 通常被理解为"二进制比较"，但 utf8mb4_0900_bin 的实现有一个值得点出的洞察：它**直接比较 UTF-8 字节，不走 mb_wc 转码点**——因为 UTF-8 编码的字节序与 Unicode 码点序天然一致（多字节首字节数值更大、同长前缀下逐字节递推），字节比较结果等于码点比较结果，但省掉了转换。同时它保留了非 binary 字符集的函数（UPPER/LOWER 可用）。所以它是"binary 的比较语义 + 常规字符集的函数能力"。

官方文档确认了这一点并给出性能结论："For `utf8mb4_0900_bin`, the weight is the `utf8mb4` encoding bytes. The sort order is the same as for `utf8mb4_bin`, but **much faster**."

**对照：`utf16_bin` 不能这么做**。`utf16_bin` 是"按码点"比较而不是"按字节"——因为 UTF-16 用**代理对**表示非 BMP 字符，导致字节序与码点序不一致：

```
码点 0xFF9D（半角片假名 N）  utf16 = FF 9D
码点 0x10384（乌加里特字母）  utf16 = D8 00 DF 84（代理对）
按码点：0xFF9D < 0x10384 ✓
按字节：0xFF > 0xD8 ✗（顺序反了）
```

所以 MySQL 的 `utf16_bin` 遇到代理对先转成码点再比——这也是 SQL:2008 的 UCS_BASIC 要求（"ordering determined entirely by Unicode scalar values"）。而 `ucs2` 不含代理对，可以字节比。**这正说明"字节序 = 码点序"是 UTF-8 特有的性质，不能推广到所有编码**（这也是 0900_bin 那项优化只对 utf8mb4 成立的原因）。

##### 语言定制的数据源：UCA + CLDR v30

官方文档明确：0900 系列 collation **同时基于 UCA 9.0.0 权重与 CLDR v30**（`utf8mb4_la_0900_ai_ci` 是例外——古典拉丁语在 CLDR 里没有定义，故不基于 CLDR）。这解释了源码里 `cs->tailoring` 那些形如 `de_pb_cldr_30` 的规则串——**"cldr_30" 就标明了 CLDR 版本**。

官方还给出两条重要性质：

- 0900 系列覆盖 **U+0 到 U+10FFFF 全范围**
- 非语言特定的 collation 按 DUCET 权重排，DUCET 里没有的码点用 implicit weight；**语言特定的 collation 可能改变 contraction 的排序**（非语言特定的则把 contraction 序列当独立字符处理）

**官方对性能的直接断言**（比任何 benchmark 都有说服力）："Collations based on UCA 9.0.0 and higher are **faster** than collations based on UCA versions prior to 9.0.0."——注意这与"`general_ci` 比 `unicode_ci` 快"是两回事：前者是不同 UCA 版本的比较，后者是"简化权重 vs 完整 UCA"的比较。


#### 端到端：一次 'A' vs '中' 的比较

把前面的零件串起来，看一次真实的比较走完哪些步骤（单字符、ai_ci 只看 primary 级）：

```
"A"                                  "中"
 │                                     │
 ├─ mb_wc: 0x41 → U+0041 (1 字节)      ├─ mb_wc: 0xE4B8AD → U+4E2D (3 字节)
 │                                     │
 ├─ page = 0x00, code = 0x41           ├─ page = 0x4E, code = 0x2D
 ├─ wpage = weights[0x00]  ✔ 存在       ├─ wpage = weights[0x4E]  ? 
 │                                     │   ├─ 页已分配 → 直接取三级权重
 │                                     │   └─ 页为 nullptr → next_implicit()
 │                                     │        CJK 区间基值 0xFB40 + (wc >> 15)
 ├─ primary = 页内 primary 权重         ├─ primary = 隐式权重（高位）
 │                                     │
 └──────────────┬──────────────────────┘
                ↓
         比较 primary：拉丁字母的权重 << CJK 区间的权重
                ↓
             'A' < '中'
```

要点：

- **字节数不参与比较**：'A' 1 字节、'中' 3 字节，但比较的是**权重不是字节**——所以多字节字符与单字节字符在同一套权重空间里排序
- **两条取权路径**（查表 vs implicit）在这里合流：权重页已分配的字符查表，未分配的按码点区间算隐式权重。两者的权重区间被刻意错开（CJK 用 0xFBxx 高位段），保证"未收录字符"也能插到正确的相对位置

#### 900 系列的分级语义（levels_for_compare）

- `utf8mb4_0900_ai_ci`（levels=1）：**只比 primary**——重音、大小写全忽略（ai = accent insensitive）
- `utf8mb4_0900_as_ci`（levels=2）：比 primary+secondary——重音敏感、大小写忽略
- `utf8mb4_0900_as_cs`（levels=3）：比满三级——重音敏感、大小写敏感（tertiary 携带大小写信息）
- `ja_0900_as_cs_ks`（levels=4）：四级——比 as_cs 多一级假名敏感

实现上按 `levels_for_compare` 模板实例化 `uca_scanner_900<Mb_wc, 1|2|3|4>`——**级别数是编译期常量**，比运行时循环少一层判断。这就是"ai 跳过 secondary"的真实含义：不是"比的时候跳过"，是**根本不进入该级别的循环**。

#### 语言 tailoring：ICU 规则串的自家解析器

zh/ja 等语言差异不换算法，而是：

1. `cs->tailoring` 存 ICU 规则串（编译期字符串，如 `de_pb_cldr_30`）
2. init 时 `create_tailoring` → `my_coll_rule_parse` 解析规则 → `my_coll_check_rule_and_inherit` → `apply_one_rule` 逐条改写**权重页的副本**
3. zh/ja 的额外数据（拼音序、假名序）编译在 `uca900_zh_data.h`/`uca900_ja_data.h`，通过 `cs->coll_param` 分支接入扫描器

即：**每个语言 collation 的 CHARSET_INFO 共享同一张权重表模板，init 时各自拷一份 + 按规则改写**。这是"数据驱动"的极致——加语言的成本是一段规则串，不是一份新算法。


顺带澄清"中文排序数据"的来源：`strings/lang_data/` 目录下按语言放着排序定制数据（如 `zh_hans.txt`），`zh` 的拼音序数据由它经脚本生成权重页后编译进 `uca900_zh_data.h`。也就是说 **zh 排序规则的数据源头是 CLDR/语言数据文件，落地形态是编译期权重表**。

##### 规则语言的语法：`&`、四级 shift、`/`、`|`、`[before N]`

上面三步里"解析规则"是最有内容的一环。规则串（如 `"&AE << \u00E4 <<< \u00C4"`）是一门**自定义的小语言**。

**词法：8 个 token，`<` 家族折叠成一个**

```c
typedef enum my_coll_lexem_num_en {
  MY_COLL_LEXEM_EOF = 0, MY_COLL_LEXEM_SHIFT = 1, MY_COLL_LEXEM_RESET = 4,
  MY_COLL_LEXEM_CHAR = 5, MY_COLL_LEXEM_ERROR = 6, MY_COLL_LEXEM_OPTION = 7,
  MY_COLL_LEXEM_EXTEND = 8, MY_COLL_LEXEM_CONTEXT = 9
} my_coll_lexem_num;
```

| 源码字符 | token | 备注 |
|---|---|---|
| `&` | `RESET` | 重置锚点 |
| `=` / `<` / `<<` / `<<<` / `<<<<` | **都是 `SHIFT`** | 用 `lexem->diff`（0/1/2/3/4）区分层级 |
| `/` | `EXTEND` | expansion |
| `\|` | `CONTEXT` | 前文相关收缩 |
| `[...]` | `OPTION` | ★ 整体为一个 token（**带括号配对计数，允许嵌套**） |
| 可打印 ASCII / `\uXXXX` 解出的码点 | `CHAR` | 转义**只支持 `\u`**（无 `\U`/`\x`） |
| 其他 | `ERROR` | |

三个易错点：① **`<`/`<<`/`<<<`/`<<<<`/`=` 不是五个 token**，而是一个 `SHIFT` + `diff`（旧版 `mysys/ctype-uca.c` 里那套 `MY_COLL_LEXEM_LESS2/LESS3` 细粒度枚举在 8.0 已不存在）；② **`*` 没有通配语义**——它只是一个普通 `CHAR`（0x2A）；③ `[...]` 被整体吞掉，所以 `[optimize [a-z]]` 是**一个** token。

**支持集（白名单）**——只有这些语法会被解析：

| 语法 | 语义 | 处理 |
|---|---|---|
| `& X` | 重置锚点（`X` 可为多字符 = expansion 锚） | `my_coll_parser_scan_reset_sequence` |
| `< X` | primary shift：`diff[0]++`，清 `diff[1..3]` | `my_coll_rule_shift_at_level(..., 1)` |
| `<< X` / `<<< X` / `<<<< X` | secondary / tertiary / quaternary shift | 同函数 `case 2/3/4` |
| `= X` | 同层（`diff` 不变，沿用上一条的偏移） | `case 0` |
| `/ X` | expansion：`X` 追加进 `rule.base` | `scan_shift_sequence` |
| `X \| Y` | context：置 `with_context = true`，`Y` 进 `curr[1]` | 同上 |
| `[before 1\|2\|3\|4]` | 前缀插入锚点，填 `before_level` | `my_coll_parser_scan_reset_before` |
| `[first/last non-ignorable]` 等 12 个逻辑位置 | 锚点取 DUCET 边界码点 | `my_coll_parser_scan_logical_position` |
| `[version 4.0.0\|5.2.0]` | 选 UCA 版本 | `my_coll_parser_scan_setting` |
| `[shift-after-method expand\|simple]` | shift 策略 | 同上 |

**★ 不支持集**——以下 ICU 常用指令在 8.0.39 的解析器里**完全没有分支**，出现在 `cs->tailoring` 里会直接**解析失败**：

```
[import] [reorder] [alternate] [backwards] [maxVariable] [normalization]
[caseFirst] [suppressContractions] [optimize] [strength] [variableTop]
[caseLevel] [hiraganaQ] [numeric] [match-boundaries] [match-style]
```

这是最容易照抄 ICU 文档而出错的地方。它们的**功能**在 MySQL 里分两种方式存在：

- **`[caseFirst]` / `[normalization]` / `[reorder]` 的功能 → 用 C 结构体静态实现**（`Coll_param.case_first` / `norm_enabled` / `reorder_param`），**不走规则串**（见后面的 reorder / caseFirst 小节）
- **`[maxVariable]` / `[alternate]` → 8.0 完全未实现**（全仓库无 `MY_COLL_MAX_VARIABLE` / `variable_type[]` 等符号）；源码里 `<family>` 注释出现的 "space, punctuation, symbol, currency" 只是 `CHARGRP_CORE` 的注释，不是变量加权机制。所以"搜索时忽略标点"在 MySQL 里不存在 ICU 那种实现方式

> 注意区分：`strings/ctype.cc`（XML 装载端）确实会把 XML 的 `alternate="..."`/`backwards="..."` 等**转成** `[alternate ...]` 文本追加进 tailoring 串——但那只服务于 8-bit XML 字符集路径，**内置 UCA collation 不走这条路**，对 UCA 规则语言而言这些指令是死语法。

**规则的数据结构 `MY_COLL_RULE`**：

```c
#define MY_UCA_MAX_EXPANSION 6
struct MY_COLL_RULE {
  my_wc_t base[MY_UCA_MAX_EXPANSION];    // reset 锚点 / expansion 源
  my_wc_t curr[MY_UCA_MAX_CONTRACTION];  // 被 shift 的字符（或收缩序列）
  int diff[4];                            // P / S / T / Q 四层差异
  size_t before_level;                    // [before N]，0 = 不启用
  bool with_context;                      // 是否 "X | Y" 形式
};
```

**`before_level` 与 `with_context` 是仅有的两个"标志位"**——不存在 `logical_pos`/`flag` 字段：逻辑位置在**解析期**就被换算成具体码点写进 `base` 了。

**解析主循环**：

```c
static int my_coll_parser_exec(MY_COLL_RULE_PARSER *p) {
  if (!my_coll_parser_scan_settings(p)) return 0;                 // 0 或多个 [setting]
  while (my_coll_parser_curr(p)->term == MY_COLL_LEXEM_RESET) {
    if (!my_coll_parser_scan_rule(p)) return 0;                   // & ... { shift... }
  }
  return my_coll_parser_scan_term(p, MY_COLL_LEXEM_EOF);          // 必须耗尽输入
}
```

采用 **`1 = 成功 / 0 = 失败`** 的反常规约定，以便把文法直接用 `&&`/`||` 翻译成代码。`cs->tailoring` 是**单个 `const char *`**——所谓"多条规则串"其实是编译期**相邻字符串字面量拼接**成的一条；XML 路径则 `realloc` 增量拼成一条。

**失败与容错**（错误码值得记住，报错信息就来自它们）：

| 情况 | 错误码 |
|---|---|
| 无法识别的 token / 非白名单 `[...]` | `EE_COLLATION_PARSER_ERROR` |
| `curr[0] > maxchar` | `EE_SHIFT_CHAR_OUT_OF_RANGE` |
| `base[0] > maxchar` | `EE_RESET_CHAR_OUT_OF_RANGE` |
| `[before N]` 落在 ignorable 字符上 | `EE_FAILED_TO_RESET_BEFORE_{PRIMARY,SECONDARY,TERTIARY}_IGNORABLE_CHAR` |

上限：expansion 6、contraction 6、权重串 25（`MY_UCA_MAX_*`）；`cs->tailoring` 为空则 `create_tailoring` 直接成功返回（纯 DUCET，无定制）。

##### `apply_one_rule`：一条规则怎么"塞进"已有权重序列

这是 tailoring 最需要想清楚的一步。**MySQL 不重排、不重编号已有 DUCET 序列**——权重区间是稠密的，没有空位可插，所以它用"**基准字符权重 −1 + extra CE 递增**"的方式把新字符**塞在基准字符之前**。

两个机制配合完成：

**① 每个被 shift 的字符都会多出一个 "extra CE"**（写进收缩节点或页内的额外 CE 槽）：

```c
// my_char_weight_put_900：复制 base 的 CE 之后，若规则带 diff，追加一个 extra CE
*to = rule->diff[0] ? dst->extra_ce_pri_base : 0;   // extra_ce_pri_base = 0x54A4
```

`0x54A4` 是"DULET 最大常规主权重 + 1"——即**借用了所有真实字符之后的一段编号空间**，用来区分"同一基准字符下的多个 tailor 变体"。

**② 层级差异加在 extra CE 上**：

```c
// apply_shift_900
last_weight_ptr[0]              += r->diff[0];      // primary
last_weight_ptr[to_stride]      += r->diff[1];      // secondary
last_weight_ptr[to_stride * 2]  += r->diff[2];      // tertiary
```

⚠️ **`diff[3]`（quaternary）不参与权重写入**——`<<<<` 只影响解析结构与收缩，日文的第四级由扫描器 `handle_ja_*_quat_wt()` 单独合成（呼应前面"日文四级不在数据表里"那节）。

**`[before N]` 的插入算法**（以 `&[before 1] A < B` 为例，`apply_primary_shift_900`）：

```
1. 先复制 A 的 CE，并追加 extra CE（初值 0x54A4）
2. 在 to 里回溯找「倒数第二个非 ignorable 权重」（最后一个是 extra CE）
   把它减一：        to[...]--          ← "Reset before"：把 A 的主权重降一档
   把 extra CE 加上 diff： last_weight_ptr[...] += diff
3. 结果：B 的 CE = [P(A)-1, S(A), T(A)] + [0x54A4+n, 0, 0]
   ⇒ B 排在 A 之前，且大于任何本来排在 A 之前的字符
```

`[shift-after-method expand]` 时还会额外 +0x1000/+0x100/+0x10 **预留缓冲**，防止"after X"与"before next(X)"交错——这是"插入点两侧都要留空位"的工程处理。

**继承：带重音变体跟着基准字符一起走**——`my_coll_check_rule_and_inherit` → `my_coll_add_inherit_rules`：对每条规则的 `curr[0]`，查它的 Unicode 正则分解记录（`uni_dec`），若它是"基字符 + 组合记号"且与原字符同源，就**自动生成一条新规则** `base = 分解序列, curr = {组合字符}`。这样"移动 e 的权重"会自动带动 é/è/ê 等变体。

**副本 vs 共享页**：`init_weight_level` 先 `memcpy` 整个 `lengths`/`weights` 指针数组，然后**只把将被改写的页 `my_uca_copy_page` 单独分配**，未触及的页仍指向 `my_uca_v900` 的静态共享页。所以 tailoring 的代价与"改了多少页"成正比。`my_coll_uninit_uca` 只 `delete contraction_nodes` 并清 `MY_CS_READY` 位（用于重初始化），页数据交给 loader 生命周期管理。

##### 三个真实规则串逐段读

**德语电话簿 `de_pb_cldr_30`**：

```
&AE << \u00E4 <<< \u00C4     // a-umlaut 排在 "AE" 之后（二级），大写变体再降一级（三级）
&OE << \u00F6 <<< \u00D6
&UE << \u00FC <<< \u00DC
```

读法：`&` RESET → `A`/`E` 两个 CHAR 组成 `base = {A, E}`（**锚点本身是 expansion**）→ `<<` SHIFT(diff=2) → `ä` → `<<<` SHIFT(diff=3) → `Ä`。展示了"reset 可多字符 + 同一锚点下多个 shift"。

**西班牙传统 `es_trad_cldr_30`**（含 contraction）：

```
&N < \u00F1 <<< \u00D1
&C < ch <<< Ch <<< CH        // ★ "ch" 是一个排序单元（contraction）
&l < ll <<< Ll <<< LL
```

`&C` → `ch` 是两个 CHAR ⇒ `curr = {c, h}`、`nshift = 2` ⇒ `apply_one_rule` 走**收缩分支** `add_contraction_to_trie`（权重写进 trie 节点、并打 `MY_UCA_CNT_HEAD/MID/TAIL` 标志位）。这就是经典的"西班牙语 ch/ll 作为单排序单元"。

**日文 `ja_cldr_30`**（四级 + 上下文）：

```
&\u309D <<<< \u30FD                                    // 四级 shift
&[before 3]\u3041 <<<\u3041|\u30FC=\u3042|\u30FC=...   // ★ [before 3] + "|" 上下文
```

第二行读法：`&[before 3]` 设 `before_level = 3`（tertiary 前缀插入），主体 `\u3041|\u30FC` 的 `|` 置 `with_context = true`，`curr = {0x30FC, 0x3041}`——**前文上下文收缩**（"长音符跟在あ之后"作为一个单元），触发点在后一个字符上（对应 `MY_UCA_PREVIOUS_CONTEXT_*` 标志位）。注意 `[before 3]` 的**前缀插入**语义与日文小假名（ぁ<あ）的排序需求正好对上。

##### reorder：把整个字符组挪到别的排序位置

有些语言的排序需要"**某个字符组整体换位**"（克罗地亚语把拉丁字母排到西里尔之前、日语把假名排到汉字之前）。这不是逐个字符改权重，而是**按权重区间整体平移**。

```c
#define UCA_MAX_CHAR_GRP 4
enum enum_char_grp { CHARGRP_NONE, CHARGRP_CORE, CHARGRP_LATIN, CHARGRP_CYRILLIC,
                     CHARGRP_ARAB, CHARGRP_KANA, CHARGRP_OTHERS };
struct Reorder_param {
  enum enum_char_grp reorder_grp[UCA_MAX_CHAR_GRP];   // 组的新顺序
  struct Reorder_wt_rec wt_rec[2 * UCA_MAX_CHAR_GRP]; // 区间换算结果
  int wt_rec_num;
  uint16 max_weight;
};
```

字符组边界是**硬编码的权重区间表**（`char_grp_infos`，如 `CHARGRP_LATIN = {0x1C47, 0x1FB5}`、`CHARGRP_CYRILLIC = {0x2022, 0x21E1}`），`my_calc_char_grp_param()` 把"组的新顺序"换算成 `wt_rec`（旧区间 → 新区间）。

**三条实现路径**（这是 reorder 最容易看晕的地方）：

| 路径 | 时机 | 做法 | 用于 |
|---|---|---|---|
| **运行期平移** | 每次 `next()` 里（**只在 primary 级**） | `weight - old.begin + new.begin` | hr / ru / sr_latn |
| **初始化期组换算** | `my_prepare_reorder()` | 把组区间预计算进 `wt_rec`（避免运行期再算） | 同上 |
| **★ 直接改页** | 初始化期改权重页副本 | `modify_all_zh_pages()` 全量改写 + `change_zh_implicit()` 改隐式首权重 | **zh 专属** |

`apply_reorder_param()` 的第一行就是 `if (cs->coll_param == &zh_coll_param) return weight;`——**中文完全不走运行期平移**，因为拼音序不是"组平移"能表达的（需要整体重排汉字权重），改页更彻底。日文还有个特殊技巧：需要把某些组挪到 Han 之后时，**注入一个 `0xFB86` 权重并借助 `return_origin_weight` 状态机把原权重回退一格再发一次**。

使用者：`utf8mb4_hr_0900_*`（克罗地亚）、`utf8mb4_ru_0900_*`、`utf8mb4_ja_0900_as_cs(_ks)`、`utf8mb4_zh_0900_as_cs`。注意**越南语用的是归一化（norm）不是 reorder**（`vi` 的 `coll_param` 是 `norm_enabled=true` + `reorder_param=nullptr`）；波斯语的 reorder 参数在 `#if 0` 里（未启用）。

##### caseFirst：大小写优先是参数化开关，不是规则串语法

一个常见误解是"`caseFirst` 来自 ICU 规则串"。**8.0.39 没有从规则串解析 `caseFirst` 的语法**（全仓库无该字面量）——它是硬编码在 `Coll_param` 里的枚举：

```c
enum enum_case_first { CASE_FIRST_OFF, CASE_FIRST_UPPER, CASE_FIRST_LOWER };
static Coll_param da_coll_param = {nullptr, false, CASE_FIRST_UPPER};  // 丹麦：大写优先
static Coll_param no_coll_param = {nullptr, false, CASE_FIRST_OFF};    // 挪威：显式关掉
```

实现是**在 tertiary 权重上 OR 一个掩码**：

```c
static bool is_tertiary_weight_upper_case(uint16 weight) {   // TR35 §3.14.1 的大写 tertiary 集合
  if ((weight >= 0x08 && weight <= 0x0C) || weight == 0x0E || weight == 0x11 ||
      weight == 0x12 || weight == 0x1D) return true;
  return false;
}
uint16 apply_case_first(uint16 weight) {
  if (cs->coll_param->case_first == CASE_FIRST_UPPER && weight_lv == 2 && weight < 0x20) {
    if (is_tertiary_weight_upper_case(weight)) weight |= CASE_FIRST_UPPER_MASK;  // 0x0100
    else                                       weight |= CASE_FIRST_LOWER_MASK;  // 0x0300
  }
  return weight;
}
```

掩码值：`CASE_FIRST_UPPER_MASK = 0x0100` / `CASE_FIRST_MIXED_MASK = 0x0200` / `CASE_FIRST_LOWER_MASK = 0x0300`——**用 tertiary 权重的高 2 位**（正常 tertiary 值 < 0x20，高 2 位空闲）"临时征用"来做大小写优先标记。两个落点：运行期 `apply_case_first()`（处理未 tailoring 的字符）、初始化期 `change_weight_if_case_first()` + `spread_case_mask()`（处理 tailoring 生成的权重，把掩码传播到该字符所有 CE 的 tertiary）。

##### 日文四级（`ja_0900_as_cs_ks`）：第四级不在数据表里

`utf8mb4_ja_0900_as_cs_ks` 的 `levels_for_compare = 4`，但**权重页每个 CE 只有三级**——第四级**完全由代码合成**：

```c
static constexpr int JA_KATA_QUAT_WEIGHT = 0x08;   // 片假名
static constexpr int JA_HIRA_QUAT_WEIGHT = 0x02;   // 平假名
...
else if (weight_lv == 3) {                 // 第四级
  wbeg = nochar; num_of_ce_left = 0;       // 明确"不查表"
  if (is_katakana_char(wc) || is_katakana_iteration(wc) || is_ja_length_mark(wc))
    return JA_KATA_QUAT_WEIGHT;
  else if (is_hiragana_char(wc) || is_hiragana_iteration(wc))
    return JA_HIRA_QUAT_WEIGHT;
  return -1;                               // 非假名：本级无权重
}
```

两个配套机制：**① 探测跳过**——primary 扫描时若全程没出现假名/迭代记号/长音符，置 `has_quaternary_weight = false`，进入第四级前直接 `return -1`（整个第四级被跳过，省一遍重扫）；**② 清音浊音之外的 ks 意义**——`ks`（kana sensitive）区分片假名与平假名，正是靠 `0x08` vs `0x02` 这两个常量。

**一个反直觉的实现细节**：规则串里 `<<<<`（四级 shift）能被 lexer 识别（最多 4 个 `<`），`my_coll_rule_shift_at_level()` 也确实累加了 `r->diff[3]`——**但 `apply_shift_900()` 只消费 `diff[0]/[1]/[2]`，第四级的 `diff` 从不参与权重生成**。所以 ks 的第四级与规则串里的四级 shift 无关，纯粹靠上面两个常量合成。

##### contraction 完整实现：前向 trie 与"上文"子树

`MY_CONTRACTION` 是一个**通用节点**（expansion 也用它）：

```c
struct MY_CONTRACTION {
  my_wc_t ch;                                  // 本节点对应的码点
  std::vector<MY_CONTRACTION> child_nodes;         // 前向 trie（后续字符）
  std::vector<MY_CONTRACTION> child_nodes_context; // ★ 上文子树（前一个字符）
  uint16 weight[MY_UCA_MAX_WEIGHT_SIZE];       // 权重串（CE 主序、末槽存 CE 数）
  bool is_contraction_tail;                    // 是否一个完整 contraction 的终点
  size_t contraction_len;
};
```

**两种查找，语义差别很大**：

| | `contraction_find` | `previous_context_find` |
|---|---|---|
| 方向 | **前向**：从当前码点起继续读输入串 | **上文**：用"前一个字符 + 当前字符"查 |
| 长度 | 2..6 字符（受位图 bit 数限制） | **只支持 2 个字符**（源码注释："We don't support context-sensitive contractions longer than two characters at the moment"） |
| 是否消耗输入 | 是（`chars_skipped = contraction_len - 1`，调用方前移 `sbeg`） | **否**（不改输入位置，只看已记录的 `prev_char`） |
| 权重挂在 | 各中间节点 | **第二个字符**的节点上 |

`previous_context_find` 的用途是泰语/老挝语这类"不间断字符串"（前一个字符决定当前字符怎么排），它只认 `child_nodes_context` 子树、不看后文——所以实现上比前向 trie 简单得多（一行 `find_contraction_part_in_trie` 接一个）。

**触发靠 4096 字节位图**（`MY_UCA_CNT_FLAG_SIZE = 4096`，索引 `wc & 0x0FFF`），五种标志：

```c
#define MY_UCA_CNT_HEAD    1     // 可作为 contraction 的头部（前向）
#define MY_UCA_CNT_TAIL    2     // 尾部
#define MY_UCA_CNT_MID1    4     // 中间位置 1..4（4 个 bit）
...
#define MY_UCA_PREVIOUS_CONTEXT_HEAD 64
#define MY_UCA_PREVIOUS_CONTEXT_TAIL 128
```

**`MY_UCA_MAX_CONTRACTION = 6` 的由来正是这个位图**：head + MID1..MID4 + tail 恰好 6 个 bit——再多就没有标志位可用了。

三个容易搞错的点：**① 8.0.39 没有 contraction hash**（`MY_CS_CONTRACTION_HASH_SIZE` 之类全仓 0 命中），查找是 `std::lower_bound` 在按 `ch` 排序的 vector 上二分；**② 默认的 `my_uca_v900` 是 `have_contractions = false, contraction_nodes = nullptr`**——contraction 只在**tailoring 解析时**创建（`&X < Y` 这类规则），所以纯 0900 系列没有 contraction 开销；**③** `contraction_find()` 里的 `num_of_ce_left` 被**硬编码为 7**（= 8 个 CE − 1），不读结构末尾的计数槽——尾部计数只在 tailoring 侧被读。

##### 权重页数据怎么生成、怎么加新 UCA 版本

页命名规则：`uca<版本字母数字>_p<三位十六进制页号>`（`uca900_p000` 覆盖 U+0000–U+00FF、`uca900_p04E` 覆盖 U+4E00–U+4EFF…），未加载的页在索引数组里写 `NULL`：

```c
uint16 *uca900_weight[4352] = { ... };   // 4352 = MY_UCA_NPAGES = 0x110000 / 256
```

单页内容的生成格式（由 dumper 的 `print_one_page()` 输出）：

```
/* Number of CEs for each character. */      ← 256 个值
/* Primary weight 1 for each character. */   ← 第一个 CE 的 primary
/* Secondary weight 1 for each character. */
/* Tertiary weight 1 for each character. */
/* Primary weight 2 for each character. */   ← 第二个 CE（expansion）
...
```

即 `maxnum = maxce × 3 + 1` 个 256 大小块——**这与前面讲的"CE 计数 + 3N 分层块"完全对应**。

**两个版本状况的重要事实**：① 生成工具是 `strings/uca9-dump.cc`（支持 `ducet|ja|zh` 三种模式，ja/zh 的语言页由它生成）；**`uca-dump.cc` 在 8.0.39 已不存在**（4.0/5.2 的 `uca_data.h` 是**冻结遗产**，不再有生成工具）；② 加一个新的 UCA 版本要动六处：`enum_uca_ver` 增项 → 新数据头（页数组 + `*_weight[4352]` + `MY_UCA_INFO`）→ 新扫描器（`uca_scanner_any` 明确 assert 不支持 900；`uca_scanner_900` 用宏写死 3 权重/CE、768 stride）→ 补齐所有 `version` 分支（`my_space_weight`、`copy_ja_han_pages`、`my_prepare_coll_param` 等）→ 新增 `CHARSET_INFO` 条目 → 若引入新层/新 tailoring 语义还要扩展 `apply_shift_900`。

> ⚠️ 注意 `uca900_data.h` 头部注释仍写着 "UCA8.0.0"（陈旧注释，实际是 9.0.0 数据）——这类注释不可当事实依据，以 `my_uca_v900` 定义与官方文档为准。
### InnoDB 侧的比较（★ 引擎与字符集的边界）

**结论先行：InnoDB 不理解字符集，行数据是字节流；比较通过 dtype 里的 collation id 回调 server 层。**

#### dtype：prtype 承载 collation id

`dtype_t` 用 `prtype`（32 位）把 MySQL type code、标志位与 **collation id** 打包在一起。与字符集相关的两点：

```c
constexpr uint32_t MAX_CHAR_COLL_NUM = 32767;       // 注释：We now support 15 bits
constexpr uint32_t CHAR_COLL_MASK = MAX_CHAR_COLL_NUM;
// 解包：注意是右移 16 位——collation 在高 16 位
static inline ulint dtype_get_charset_coll(ulint prtype) {
  return ((prtype >> 16) & CHAR_COLL_MASK);
}
ulint dtype_form_prtype(ulint old_prtype, ulint charset_coll);  // 打包
```

1. **collation id 占 15 位、位于 bit 16-30**：位位置有硬证据——序列化处把 `len` 写在 `buf+2`、**collation 写在 `buf+4`**（`mach_write_to_2(buf + 4, dtype_get_charset_coll(type->prtype))`），并用 `buf[4] |= 128` 单独表示 `DATA_NOT_NULL`。**15 位上限意味着 collation id 不能超过 32767**——这是"为什么 collation id 不能随便往大了编"的硬约束（目前最大 309，余量充足）。
2. **`mbminmaxlen`（5 位，`mbmaxlen * 5 + mbminlen`）**：InnoDB 把"这个字符集一个字符最多/最少几个字节"直接缓存在列元数据里（`get_mbminlen()`/`get_mbmaxlen()`），避免热路径去查 server 的 `CHARSET_INFO`。这是"引擎不碰字符集"原则下**唯一被下沉的字符属性**——因为它只影响字节布局，不影响字符语义。

#### 深入：`mbminmaxlen` 到底是什么、用来干什么

**先澄清一个常见误解**：InnoDB 与字符集打交道**不是只有一个字段**，而是两个、分工明确：

| 字段 | 位置 | 存的是什么 | 服务于 | 怎么用 |
|---|---|---|---|---|
| **collation id** | `prtype` bit 16-30 | 排序规则**编号** | **比较语义** | 用时 `get_charset(id)` 取回 `CHARSET_INFO` 对象，再调 `strnncollsp` |
| **`mbminmaxlen`** | `dict_col_t` 独立 5 位 | 字符宽度**数值** | **存储布局** | 直接用，不查任何字符集对象 |

差别在于：collation id 只是"存了个编号、用时再解析成对象"（语义仍需 server 参与）；而 `mbminmaxlen` 是**把字符属性的值本身缓存下来**（引擎可独立使用）。所以准确说法是：**`mbminmaxlen` 是唯一被"下沉为数值"的字符属性**，而 collation id 是"引用"。

**它存的到底是什么**：`mbminlen`（一个字符最少几个字节）与 `mbmaxlen`（最多几个字节）。打包成 5 位：`mbminmaxlen = mbmaxlen * 5 + mbminlen`。取值示例：

| 字符集 | mbminlen | mbmaxlen | 打包值 |
|---|---|---|---|
| latin1 | 1 | 1 | 1×5+1 = **6** |
| gbk | 1 | 2 | 2×5+1 = **11** |
| utf8mb4 | 1 | 4 | 4×5+1 = **21** |
| ucs2 / utf16 | 2 | 2（ucs2）/ 4（utf16） | 2×5+2 = **12** / 4×5+2 = **22** |

**它的三个实际用途**（都在源码里有硬落点）：

**用途一：判定这一列是"定长"还是"变长"**——`dtype_get_fixed_size_low()`：

```c
case DATA_MYSQL:
  if (prtype & DATA_BINARY_TYPE) {
    return (len);                    // binary 类型：定长
  } else if (!comp) {
    return (len);                    // REDUNDANT 格式：定长
  } else {
#ifdef UNIV_DEBUG
    innobase_get_cset_width(dtype_get_charset_coll(prtype), &i_mbminlen, &i_mbmaxlen);
    ut_ad(DATA_MBMINMAXLEN(i_mbminlen, i_mbmaxlen) == mbminmaxlen);   // ★ 校验缓存
#endif
    if (DATA_MBMINLEN(mbminmaxlen) == DATA_MBMAXLEN(mbminmaxlen)) {
      return (len);                  // ★ min == max → 定长字符集 → 该列定长
    }
  }
  [[fallthrough]];                   // 否则落到下面返回 0 = 变长
...
case DATA_VARCHAR: ...: return (0);  // 变长
```

**核心判据就是 `mbminlen == mbmaxlen`**：一个字符集若每个字符固定占 N 字节（latin1=1、ucs2=2），那 `CHAR(n)` 也就固定占 n×N 字节，是定长列；若字符集变长（utf8mb4 的 1-4 字节、gbk 的 1-2 字节），`CHAR(n)` 在 COMPACT 下就是变长存储。这个判定决定了行格式里字段怎么摆、offsets 怎么算。

**用途二：计算这一列的"最小存储尺寸"**——`dtype_get_min_size_low()`：

```c
case DATA_MYSQL:
  if (prtype & DATA_BINARY_TYPE) {
    return (len);
  } else {
    ulint mbminlen = DATA_MBMINLEN(mbminmaxlen);
    ulint mbmaxlen = DATA_MBMAXLEN(mbminmaxlen);
    if (mbminlen == mbmaxlen) {
      return (len);                              // 定长字符集
    }
    /* this is a variable-length character set */
    ut_a(mbminlen > 0);
    ut_a(mbmaxlen > mbminlen);
    ut_a(len % mbmaxlen == 0);                   // 声明长度必须是 mbmaxlen 的整数倍
    return (len * mbminlen / mbmaxlen);          // ★ 最小尺寸公式
  }
```

**公式 `len × mbminlen / mbmaxlen` 的含义**：`CHAR(10) utf8mb4` 的声明长度 `len = 10 × 4 = 40` 字节（按最坏情况每个字符 4 字节算）；而**全 ASCII 内容时每个字符只占 1 字节**，所以最小存储是 `40 × 1 / 4 = 10` 字节。这个"最小尺寸"用于行长度估算（判断一行能否放进页）、DDL 缓冲区大小计算等场景（`get_min_size()` 的调用点在 `row0ins.cc`、`row0log.cc`、`ddl0buffer.cc`）。

**用途三：供转换路径直接查询宽度**：`get_mbminlen()` / `get_mbmaxlen()` 被 `ha_innodb.cc`（4 处，主要是 `mysql_row_templ_t` 的填充与长度计算）与 `row0ins.cc`（1 处）调用——即"这一列在 MySQL 行格式里有多宽"这类问题，直接读缓存而不是查 `CHARSET_INFO`。

**缓存的一致性怎么保证**：上面代码里那句 debug 断言就是答案——

```c
innobase_get_cset_width(dtype_get_charset_coll(prtype), &i_mbminlen, &i_mbmaxlen);
ut_ad(DATA_MBMINMAXLEN(i_mbminlen, i_mbmaxlen) == mbminmaxlen);
```

`innobase_get_cset_width`（`ha_innodb.cc`）从 collation id 反查 `CHARSET_INFO` 拿到真实的 `mbminlen`/`mbmaxlen`，再断言它与列上缓存的 `mbminmaxlen` 一致。**这是"缓存可能与真实字符集脱节"的防线**——但只在 debug 构建生效。release 下若两者不一致（如跨版本降级、字符集定义变更而字典未更新），引擎会按错误的字符宽度做布局计算而毫无察觉。

设置点在 `dict/mem.cc`（建列时 `column->set_mbminmaxlen(mbminlen, mbmaxlen)`）、`dict0inst.cc`（instant ADD COLUMN 的 dummy 列）、`handler0alter.cc`（新加虚拟列补设）。

> `dtype_t` / `dfield_t` / `dtuple_t` 的完整结构剖析（位布局全貌、`mtype`、`len`、虚拟列数组等）属于"行数据表示"主题，见 [`../../infra/structure/data_repr.md`](../../infra/structure/data_repr.md)，本篇只取字符集相关部分。

建表时 `ha_innodb.cc` 把 `field_charset->number` 经 `dtype_form_prtype` 打包进 prtype；反向由 `dd_get_mysql_charset(column->collation_id())` 从数据字典恢复。

#### mtype 的分层：latin1 遗留 vs 通用路径

`mtype` 里有一批"绑定 latin1_swedish_ci"的历史类型，源码注释写得很直白：

```c
/** character varying of the latin1_swedish_ci charset-collation; note that the
MySQL format for this, DATA_BINARY, DATA_VARMYSQL, is also affected by
whether the 'precise type' contains DATA_MYSQL_TRUE_VARCHAR */
constexpr uint32_t DATA_VARCHAR = 1;
/** fixed length character of the latin1_swedish_ci charset-collation */
constexpr uint32_t DATA_CHAR = 2;
```

即 `DATA_CHAR` / `DATA_VARCHAR` **只服务 latin1_swedish_ci**（5.x REDUNDANT 时代遗留），现代 utf8mb4 列走 `DATA_MYSQL` / `DATA_VARMYSQL` / `DATA_BLOB` 通用路径；`DATA_MYSQL_TRUE_VARCHAR = 15` 是 prtype 里的标志（对应 MySQL 5.0.3+ 的真 VARCHAR，行格式里带 1-2 字节长度前缀）。

#### 比较路径：cmp_data_data → cmp_whole_field → innobase_mysql_cmp

索引页内的字符串比较完整链路（`rem0cmp.cc`）：

```
cmp_data_data(mtype, prtype, is_asc, data1, len1, data2, len2)   ← 对外入口
  → cmp_data(...)
      ├─ 二进制/几何类型：pad = ULINT_UNDEFINED，直接 memcmp 语义
      ├─ DATA_CHAR / DATA_VARCHAR：硬编码 my_charset_latin1.coll->strnncollsp
      └─ 其他 → cmp_whole_field(mtype, prtype, is_asc, ...)
           ├─ DATA_VARMYSQL / DATA_MYSQL → innobase_mysql_cmp(...)
           └─ 其他（POINT 等）各自处理
```

终点 `innobase_mysql_cmp` 的完整实现（注意签名**只有 prtype，没有 mtype**——类型信息全在 prtype 里）：

```c
static inline int innobase_mysql_cmp(ulint prtype, const byte *a,
                                     size_t a_length, const byte *b,
                                     size_t b_length) {
#ifdef UNIV_DEBUG
  switch (prtype & DATA_MYSQL_TYPE_MASK) {       // debug 下校验类型合法
    case MYSQL_TYPE_BIT: case MYSQL_TYPE_STRING: case MYSQL_TYPE_VAR_STRING:
    case MYSQL_TYPE_TINY_BLOB: case MYSQL_TYPE_MEDIUM_BLOB:
    case MYSQL_TYPE_BLOB: case MYSQL_TYPE_LONG_BLOB: case MYSQL_TYPE_VARCHAR:
      break;
    default: ut_error;
  }
#endif
  uint cs_num = (uint)dtype_get_charset_coll(prtype);          // ① 解出 collation id
  if (CHARSET_INFO *cs = get_charset(cs_num, MYF(MY_WME))) {   // ② 查 server 注册表
    if ((prtype & DATA_MYSQL_TYPE_MASK) == MYSQL_TYPE_STRING &&
        cs->pad_attribute == NO_PAD) {
      /* MySQL specifies that CHAR fields are stripped of
      trailing spaces before being returned from the database.
      Normally this is done in Field_string::val_str(),
      but since we don't involve the Field classes for internal
      index comparisons, we need to do the same thing here
      for NO PAD collations. (If not, strnncollsp will ignore
      the spaces for us, so we don't need to do it here.) */
      a_length = cs->cset->lengthsp(cs, (const char *)a, a_length);
      b_length = cs->cset->lengthsp(cs, (const char *)b, b_length);
    }
    return (cs->coll->strnncollsp(cs, a, a_length, b, b_length));  // ③ 跨层回调
  }
  ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_919)                       // 找不到 → 致命
      << "Unable to find charset-collation " << cs_num;
  return (0);
}
```

逐段解释四个关键点：

1. **`lengthsp` 只在 NO PAD 时做，且原因不是"兜底"**：源码注释说得很清楚——CHAR 列返回前通常会由 `Field_string::val_str()` 剥掉尾空格，但**内部索引比较不经过 Field 类**，所以 InnoDB 必须自己剥；而 **PAD SPACE 的 collation 不需要剥**，因为 `strnncollsp` 本身就忽略尾部空格。这条注释把"PAD 语义由谁负责"交代得很精确。
2. **跨层回调是常态**：每一次字符串比较都穿过引擎边界调 server 的 `strnncollsp`——**没有 sort key 缓存、没有 memcmp 快路径**（latin1 硬编码那条除外）。这是"引擎不碰字符集"的直接代价：比较成本 = 一次 `get_charset` + 一次函数指针调用 + 完整的 UCA 权重扫描。
3. **`get_charset` 失败即 `ib::fatal`**：collation id 在字典里存着但注册表查不到（如跨版本降级、字符集目录损坏）会**直接让实例崩溃**——不是返回错误，是 fatal。这是"id 隐含依赖注册表"的刚性。
4. **latin1 那条是唯一的性能特例**：`DATA_CHAR`/`DATA_VARCHAR` 直接调 `my_charset_latin1.coll->strnncollsp`，省掉 `get_charset` 查表与类型判定——因为这两类 mtype 已经保证了字符集就是 latin1_swedish_ci。

#### 为什么不让 InnoDB 自己算（设计权衡）

本来可以把 collation 逻辑下沉进 InnoDB（在引擎内比较、甚至缓存 sort key），但那样：① 引擎要链接 Unicode 权重表，二进制膨胀；② 引擎要理解 MySQL 的 charset 语义，破坏 server↔引擎边界；③ 其他引擎（MyISAM/MEMORY/分区）也要各实现一遍。代价就是上面看到的**每次比较一次跨层函数指针调用**。

反过来，InnoDB 也确实保留了一点"字符感知"：它知道 `mbminlen`/`mbmaxlen`（用于 CHAR 剥空格存储与长度计算）、知道 `pad_attribute`（决定要不要 `lengthsp`）——但**权重计算完全交给 server**。这条边界是清晰的：**引擎管字节布局与长度，server 管字符语义**。

#### 全景：InnoDB 里涉及字符集的八个功能域

比较只是"出口"，字符集信息在 InnoDB 里渗透得更深。完整清单：

```
┌─────────── server: CHARSET_INFO / dd collation_id ───────────┐
└──────────────────────────↕ ha_innodb.cc ─────────────────────┘
① 类型编码   prtype[30:16] = collation id；dict_col_t::mbminmaxlen
② 元数据     DD collation_id ↔ prtype 互转；升级校验；IMPORT 校验
③ 行格式     CHAR 尾空格剥离/回填（填充字节随 mbminlen 变）
④ 索引       prefix_len 存字节；767/3072 键长上限
⑤ 比较       innobase_mysql_cmp → strnncollsp（本节主题）
⑥ 全文检索   fts_is_charset_cjk（9 个硬编码名）；FTS 索引全列同 charset
⑦ 标识符     表名/FK 名在 my_charset_filename ↔ UTF-8 间转换
⑧ 辅助       row_raw_format 输出、InnoDB C API 的 well_formed_len
```

- **① 类型编码**：`prtype` 的 collation id 与列上的 `mbminmaxlen`（见上节）
- **② 元数据**：DD 的 `collation_id` 与 prtype 双向转换（`dd_get_mysql_charset` / `get_innobase_type_from_mysql_dd_type`）；**升级时校验列字符集一致性**（`dict0upgrade.cc`，不一致报 "Column character set mismatch"）；**传输表空间 IMPORT 会校验字符集**——`.cfg` 元数据里写了 8 字节 `collation_id`，导入时与目标列比对，不符则失败
- **③ 行格式**：CHAR 列的尾空格**填充字节随字符集变**——`row_mysql_pad_col` 按 `mbminlen` 分别填 `0x20` / `0x0020` / `0x000020` / `0x00000020`；COMPACT/DYNAMIC 下多字节 CHAR 剥尾空格存储（变长），而 REDUNDANT 格式要求定长，两者冲突时要特殊处理
- **④ 索引**：`dict_field_t::prefix_len` 存的是**字节**（注释明说 UTF-8 下由 server 换算成 `mbmaxlen × 字符数` 后传入）；键长上限 767（ANTELONE）/ 3072（DYNAMIC+）是**纯字节**上限；`innobase_get_at_most_n_mbchars` 用 `my_charpos` 求前 n 字符的真实字节数，防止前缀切断多字节字符
- **⑥ 全文检索**（耦合最重的一块，详见 [`../../feat/fts.md`](../../feat/fts.md)）：`fts_get_charset(prtype)` 从 prtype 取字符集、找不到即 `ib::fatal`；**FTS 索引要求所有列同一字符集**；`fts_is_charset_cjk()` **硬编码 9 个 collation 名**（gb2312/gbk/big5/gb18030/ujis/sjis/cp932/eucjpms/euckr 的 `*_ci`），**不含任何 utf8mb4**——所以 utf8mb4 的中文走不到 CJK 哈希分表分支；ngram 分词只按 parser 插件名判定，不做字符集校验
- **⑦ 标识符**：表名、外键名、表空间路径在 `my_charset_filename` 与 `system_charset_info`(UTF-8) 之间双向转换——这是一条独立于"数据字符集"的链路（文件名有自己的字符集）

**值得注意的一个反例**：InnoDB 的**内部字典表**（SYS_TABLES/SYS_COLUMNS/...）列全是 `DATA_BINARY`/`DATA_INT` 且 `prtype = 0`，所以 `dtype_get_charset_coll()` 得到的是 **0 而不是 63**（`DATA_MYSQL_BINARY_CHARSET_COLL = 63` 只用于判定"是否真 binary 从而不做空格填充"）。别把"内部表"与"binary collation"混为一谈。

#### 行格式与字符集

字符集不改变行格式布局（REC 里存的是字节 + 长度前缀），但有一个省空间优化：COMPACT/DYNAMIC 下，**多字节 CHAR（mbminlen=1 且 mbmaxlen>1）会剥掉尾空格存储**（row0mysql.cc 的 `comp && type==DATA_MYSQL` 分支）——CHAR(10) utf8mb4 存 'abc' 只占 3 字节而不是 40。这也解释了为什么 CHAR 比较前要 `lengthsp` 兜底：存储剥了空格，语义上还要按"补齐"来比。

### DDL、数据字典与协商

#### DDL 路径：从语法到列字符集

语法层（`sql_yacc.yy`）：`default_charset`/`default_collation` 规则产生 `charset_with_opt_binary`；`collation_name` 规则里 `mysqld_collation_get_by_name` **找不到即语法错**（不是运行时错）。

准备阶段（`mysql_prepare_create_table`）做两件事：

```c
save_cs = sql_field->charset = get_sql_field_charset(sql_field, create_info);
// 只有 STRING / VAR_STRING / SET / ENUM 四类才做默认值字符集转换：
if (sql_field->constant_default &&
    save_cs != sql_field->constant_default->collation.collation &&
    (sql_field->sql_type == MYSQL_TYPE_VAR_STRING ||
     sql_field->sql_type == MYSQL_TYPE_STRING || ...)) {
  sql_field->constant_default =
      sql_field->constant_default->safe_charset_converter(thd, save_cs);
  if (sql_field->constant_default == nullptr)
    my_error(ER_INVALID_DEFAULT, MYF(0), sql_field->field_name);   // 转换不了
}
```

注意字面量的字符集来自解析器：`Item_string(... YYTHD->charset())`，而 `THD::charset()` 返回 **`character_set_client`**——所以客户端连接字符集直接决定字面量被贴什么标签（这是转换链上很多问题的源头）。

#### 数据字典：字符集与默认值怎么存

- 排序规则目录表 `mysql.collations`：`id`、`collation_name`、`character_set_id`、`is_compiled`、`sort_length`、**`pad_attribute ENUM('PAD SPACE','NO PAD')`**，`options`；初始化时由 `all_charsets` 填充
- `dd::Table::collation_id` / `dd::Column::collation_id` 外键引用它
- **列默认值存原始字节**：`dd::Column::default_value` 是按**该列自身字符集 pack 好的字节**（SDI 里 `write_binary` 写出），字符集由列的 `collation_id` **隐含**，DD 里没有"默认值字符集"这一列；另有 `default_value_utf8` 是给 I_S/SHOW 用的文本表示
- **打开表时不做任何字符集转换**：`dd_table_share.cc` 把 `col_obj->default_value().data()` 直接 `memcpy` 进 `share->default_values`——从反面证明默认值的字符集是"约定"而非"记录"

#### I_S 与 SHOW

`information_schema.CHARACTER_SETS` / `COLLATIONS` 是 dd 表的视图（`sql/dd/impl/system_views/` 下定义，`add_from("mysql.collations col")` 再 join）；`SHOW COLLATION` / `SHOW CHARACTER SET` 已被**重写为对 I_S 的查询**（`build_show_collation_query`）。列的默认值在 I_S 里读的是 `default_value_utf8`。

#### 协商：表达式 collation 怎么定

**解析**（sql_yacc.yy）：`default_charset`/`default_collation` 规则 → `collation_name` 规则里 `mysqld_collation_get_by_name` **找不到即语法错**（不是运行时错）。

**存储**：dd 里 `mysql.collations` 表（id / collation_name / character_set_id / is_compiled / sort_length / **pad_attribute ENUM('PAD SPACE','NO PAD')** / options）——pad 语义进了数据字典。`dd::Table` 的 `collation_id` 外键引用它。`information_schema.CHARACTER_SETS`/`COLLATIONS` 是 dd 表的视图；`SHOW COLLATION` 被重写为对 I_S 的查询。

**协商**（表达式 collation 推导）：8.0 用 `DTCollation::aggregate`（sql/item.cc），核心是 coercibility 优先级（`COERCIBILITY()` 函数可见，值越低优先级越高）：

| 值 | 来源 | 例 |
|---|---|---|
| 0 | 显式 `COLLATE` 子句 | `expr COLLATE utf8mb4_bin` |
| 1 | 两个不同 collation 字符串的连接结果 | `CONCAT(c1, c2)` |
| 2 | 列 / 存储过程参数 / 局部变量 | 列定义 |
| 3 | 系统常量 | `USER()` / `VERSION()` |
| 4 | 字面量 | `'abc'` |
| 5 | 数值/时间值隐式转字符串 | `1` / `NOW()` |
| 6 | NULL | `NULL` |

推导规则：EXPLICIT 冲突 → 报错（`Illegal mix of collations`）；同 charset 不同 collation → 优先 `MY_CS_BINSORT`（binary 赢了），否则回退 charset 的 bin + DERIVATION_NONE；跨 charset → 依 `MY_COLL_ALLOW_SUPERSET_CONV/COERCIBLE_CONV/NUMERIC_CONV` 决定，同 coercibility 时 Unicode 一方胜出（非 Unicode 自动向 Unicode 转）。DDL 的 charset/collation 匹配校验：`ER_COLLATION_CHARSET_MISMATCH`（"collation 必须属于该 charset"），报错点分散在 SET NAMES、CREATE TABLE 列属性、CONVERT 函数。

**连接默认**：`default_collation_for_utf8mb4` 变量初始化为 `&my_charset_utf8mb4_0900_ai_ci`；`character_set_client` 变更触发 `fix_thd_charset` → `THD::update_charset()` 联动 connection/results。

#### DDL 整表字符集转换：ALTER ... CONVERT TO 走的是什么

`ALTER TABLE ... CONVERT TO CHARACTER SET x` **没有专门的"转换算法"**——它只是把列定义的字符集改掉，随后由拷贝算法逐行用 `Copy_field` 完成转换：

```cpp
// ① 解析阶段：table_charset 是 "CONVERT TO" 专属的开关
const CHARSET_INFO *get_sql_field_charset(...) {
  if (sql_field->is_array || cs == &my_charset_bin) return cs;   // 数组列/BINARY 不改
  if (create_info->table_charset != nullptr) return create_info->table_charset;
  ...
}
// ② 只有 CONVERT TO 会设置它：
if ((create_info->used_fields & HA_CREATE_USED_DEFAULT_CHARSET) && ...)
  if (create_info->used_fields & HA_CREATE_USED_CHARSET)
    create_info->table_charset = create_info->default_table_charset;

// ③ 逐行拷贝（copy_data_between_tables, sql_table.cc）：
thd->check_for_truncated_fields = CHECK_FIELD_WARN;   // ★ 转换期截断降级为 warning
thd->num_truncated_fields = 0;
...
(copy_end++)->set(*ptr, def->field);                  // Copy_field::set —— 差异列才建 Copy_field
```

**三个要点**：① 数据转换复用 `Copy_field` 分派 → `do_field_string` → `Field::store`（即上面那条链），**没有另一套代码**；② 转换期间 `check_for_truncated_fields` 从默认值降为 `CHECK_FIELD_WARN`——**避免一行转不下就中断整表**；③ `table_charset` 的存在使"改列的字符集"与"整表 CONVERT TO"在 `get_sql_field_charset` 里走不同分支（前者改 `cs`，后者改 `table_charset`）。

> 顺带一条边界：DDL 期间**标识符用 `system_charset_info` 比较、列值用列字符集转换**，两条线并行（InnoDB 侧 `dict0dd.cc` 的 `is_renamed`/`is_dropped` 就依赖前者）。

#### 字符串函数怎么定 collation：`Item_str_func` 与参数聚合

⚠️ **名称更正**：`Item_str_ops` 在 8.0.39 **不存在**——字符串函数的基类是 **`Item_str_func`**（`item_strfunc.h`）。

结果的 collation 由 `agg_arg_charsets_for_string_result()` 聚合（内部 `agg_item_charsets` 带 `ALLOW_SUPERSET | ALLOW_COERCIBLE | ALLOW_NUMERIC` 三个开关）：

```cpp
// Item_func_lower::resolve_type —— UPPER/LOWER 的 multiply 来自 collation 字段
if (agg_arg_charsets_for_string_result(collation, args, 1)) return true;
multiply   = collation.collation->casedn_multiply;    // ★ 不是固定值，随 collation
converter  = collation.collation->cset->casedn;
set_data_type_string(args[0]->max_char_length() * multiply);

// Item_func_concat::resolve_type —— CONCAT 的结果 collation = 聚合结果
if (agg_arg_charsets_for_string_result(collation, args, arg_count)) return true;
for (uint i = 0; i < arg_count; i++)
  char_length += args[i]->max_char_length(collation.collation);   // 长度按聚合 collation 算
set_data_type_string(char_length);
```

**`multiply` 的取值链**清楚了：`Item_func_lower/upper` 的缓冲区放大倍数**不是写死的 2**，而是读该 collation 的 `casedn_multiply` / `caseup_multiply`（utf8mb3/utf8mb4 各 collation 均为 1）——这正好与前面 Bug#119463 的修复衔接：**上层扩容时用的就是这个 multiply，`++multiply` 重试意味着突破 collation 的静态声明**。求值时参数先被转成结果 collation 再拼接（`eval_string_arg(collation.collation, ...)` + `tmp_value.set("", 0, collation.collation)`）。

### 字符集在 SQL 语义边缘

前面的章节讲的是"字符集机制本身"，这一节补**它在各 SQL 语义上的落点**——尤其是"同一个 `_ci` 语义在不同语句里由不同函数实现"这件事。

#### 三路比较：ORDER BY / GROUP BY / DISTINCT 走的是不同函数

| 语句 | 比较机制 | 源码入口 |
|---|---|---|
| **ORDER BY** | `strnxfrm` 生成**权重键**，再 memcmp | `Sort_param::make_sortkey_from_item`（`cs->coll->strnxfrm`，定长时 `MY_STRXFRM_PAD_TO_MAXLEN`） |
| **GROUP BY**（内存/temptable） | **`hash_sort` 分组 + `strnncollsp` 比较** | `Cell_calculator::hash()` / `compare()` |
| **GROUP BY**（磁盘/`Unique`） | `strnncollsp` | `unique_key_cmp` → `Field::key_cmp` |
| **DISTINCT** | 与 GROUP BY 共用 `Unique` 机制 | `create_tmp_table(..., distinct=true)` |

**含义**：三者在 `_ci`/`_ai_ci` 下都会把 `'a'`/`'A'`/`'á'` 视为同类（ORDER BY 相邻、GROUP BY 同组、DISTINCT 只留一），在 `_bin`/`_as_cs` 下分开。但**代价不同**：ORDER BY 要生成膨胀的排序键（`strnxfrm`），GROUP BY 走哈希（先 `hash_sort` 再在冲突时 `strnncollsp` 精比）——这也是为什么"GROUP BY 大字符串"比"ORDER BY 大字符串"通常更省内存。

#### collation 聚合：`Illegal mix of collations` 的触发点

集合操作与条件表达式的 collation 都由同一个引擎推导——`agg_arg_charsets` → `agg_item_collations` → **`DTCollation::aggregate`**（前面协商节已给出优先级表）。三个触发错误的场景：

```cpp
// UNION：结果列 collation 无法聚合
if (type->result_type() == STRING_RESULT &&
    type->collation.derivation == DERIVATION_NONE)
  my_error(ER_CANT_AGGREGATE_NCOLLATIONS, MYF(0), "UNION");
```

- **UNION**：两侧对应列 collation 无法聚合 → `ER_CANT_AGGREGATE_*COLLATIONS`（即经典的 `Illegal mix of collations`）
- **CASE / COALESCE / NULLIF**：都走 `aggregate_string_properties` → `agg_item_charsets_for_string_result`，规则与 UNION 一致
- **全数值参数**：回落到 `Item::default_charset()`（= `collation_connection`）

#### `COLLATE` 子句能出现在哪些位置

`COLLATE` 是"把表达式 collation 提升为最高优先级"的唯一手段，语法落点比想象的多：

| 位置 | 规则 |
|---|---|
| 列定义属性 | `pt_collate_column_attr` |
| 类型后 `opt_collate` | `field_def` / SP 参数 `sp_fdparam`/`sp_pdparam`/`sp_decl` / `jt_column` |
| **表达式** `expr COLLATE x` | `Item_func_set_collation`（`%prec NEG`） |
| `ALTER TABLE ... CONVERT TO ... COLLATE x` | 改表默认 collation |
| `SET NAMES ... COLLATE x` | 改会话连接 collation |

表达式形式的关键在 `Item_func_set_collation::resolve_type`：`BINARY` 关键字 → 取**同字符集的 `_bin`**；否则查名字；**跨字符集直接报 `ER_COLLATION_CHARSET_MISMATCH`**；成功后标记 `collation.set(set_collation, DERIVATION_EXPLICIT, ...)`——**`DERIVATION_EXPLICIT` 就是在聚合里"必胜"的那个等级**（对应协商节表里的 coercibility 0）。

#### REGEXP：默认大小写敏感性**来自 collation**

8.0 的正则用 ICU，但"大小写是否敏感"的默认值不是正则自己的，而是**读 collation 的 state**：

```cpp
// Item_func_regexp::set_pattern
bool is_case_sensitive =
    ((collation.collation->state & (MY_CS_CSSORT | MY_CS_BINSORT)) != 0);
```

即 `_ci` 类（无 `CSSORT`/`BINSORT`）→ 默认**不敏感**（ICU 加 `UREGEX_CASE_INSENSITIVE`）；`_bin`/`_as_cs`（含 `MY_CS_BINSORT`/`CSSORT`）→ 默认**敏感**；`match_type` 里的 `i`/`c` 可显式覆盖。注意**它不用 collation 的 `caseup` 表**——`caseup` 只服务于 `UPPER()`；这里 collation 仅决定"默认 flag"。

**`match_type` 的五个字母各是什么**：

| 字母 | ICU flag | 含义 |
|---|---|---|
| `c` | 清除 `UREGEX_CASE_INSENSITIVE` | 强制大小写敏感 |
| `i` | 置 `UREGEX_CASE_INSENSITIVE` | 强制不敏感 |
| `m` | `UREGEX_MULTILINE` | `^`/`$` 匹配行首尾 |
| `n` | `UREGEX_DOTALL` | `.` 匹配换行 |
| **`u`** | `UREGEX_UNIX_LINES` | ★ **UNIX 行尾**（**不是 "unicode"**！） |

ICU 引擎的输入字符集是 **UTF-16LE**（`regexp_lib_charset`），所以非 UTF-16 的输入会先 `my_convert` 转过去；**binary 字符集按 `my_charset_latin1` 仿真处理**（`faux_binary_charset`）。结果字符串的 charset **跟随参数聚合后的 collation**（`buf->set_charset(collation.collation)`）。

#### 分区键的字符集：`PARTITION BY KEY` 的哈希约束

分区是"字符集影响物理落盘"的最直接例子：`PARTITION BY KEY` 用 `Field::hash()` → **该列自身 collation 的 `hash_sort`** 定位分区。所以**换 collation 会改变行的分区归属**——这也是为什么 5.1 → 5.5 的哈希算法变更要靠 `KEY_ALGORITHM_51` 保留旧行为（源码注释：`Force this to my_hash_sort_bin, which was used in 5.1!` / `to keep on-disk stability`）。

`PARTITION BY RANGE/LIST COLUMNS` 含字符串列时，会先对列值做 **`my_strnxfrm` 归一化**再比较（`copy_to_part_field_buffers`）；`ALTER ... CONVERT TO` 改到分区列上会被 `check_part_func_fields` 拒绝（`mbmaxlen > 1 || strxfrm_multiply > 1` 不允许）。**分区机制本身**见 [`../../feat/partitioning.md`](../../feat/partitioning.md) 与 [`../../index/partitioning.md`](../../index/partitioning.md)。

#### 其余落点速查

| 场景 | 字符集规则 |
|---|---|
| **窗口函数** | `PARTITION BY`/`ORDER BY` 的 peer 判定复用普通 ORDER BY（`Arg_comparator` → `strnncoll`），语义一致；细节见窗口函数专题 |
| **内部临时表** | `create_tmp_table` **已无 `default_charset` 参数**——列 collation 一律取表达式的 collation；只有"全数值聚合"才回落 `collation_connection` |
| **生成列 / 函数索引** | collation = **表达式自身的 collation**（函数索引生成隐藏生成列时经 `generate_create_field`）；表达式校验入口是 `pre_validate_value_generator_expr`（**不存在 `check_generated_columns` 这个符号**）；索引可用性见 [`../query/07_optimize/21_collation_index_usability.md`](../query/07_optimize/21_collation_index_usability.md) |
| **`GROUP_CONCAT` / `CONCAT_WS`** | 结果 collation 由 `agg_arg_charsets_for_string_result` 聚合（`GROUP_CONCAT` **只聚合字段参数、不含 ORDER BY 参数**）；`SEPARATOR` 若与结果 collation 不同会被**转换**；`group_concat_max_len` 按 `mbminlen`/`mbmaxlen` 换算字节，并用 `well_formed_len` **在字符边界截断**（不切出半个多字节字符） |
| **`GROUP BY` 的临时表列** | 取分组表达式的 collation，不用 `character_set_connection` |

### 词法分析层的字符集

字符集不止参与"比较与转换"，还直接决定**词法分析怎么切词**——这层常被忽略。

`CHARSET_INFO` 挂着 `state_maps`（`lex_state_maps_st`：256 项 `main_map` + `hint_map`）与 `ident_map`，由 `init_state_maps(cs)` 按**该字符集自己的 ctype** 生成：

```c
if (my_isalpha(cs, i))      state_map[i] = MY_LEX_IDENT;          // 字母 → 标识符
else if (my_isdigit(cs, i)) state_map[i] = MY_LEX_NUMBER_IDENT;   // 数字
else if (my_ismb1st(cs, i)) state_map[i] = MY_LEX_IDENT;          // 多字节首字节也算标识符
else if (my_isspace(cs, i)) state_map[i] = MY_LEX_SKIP;           // 空白跳过
else                        state_map[i] = MY_LEX_CHAR;
// 之后再硬编码：'_' '$' → IDENT、'\'' → STRING、'.' → REAL_OR_POINT、'>' '=' '!' → CMP_OP ...
ident_map[i] = (state_map[i] == MY_LEX_IDENT || state_map[i] == MY_LEX_NUMBER_IDENT);
```

启动时（`lex_init`）对所有"支持作为解析器字符集"的字符集各生成一份，之后只读。`MYSQLlex` 每次进入都**重新取当前字符集**的映射：

```c
const CHARSET_INFO *cs = thd->charset();
const my_lex_states *state_map = cs->state_maps->main_map;
const uchar *ident_map = cs->ident_map;
```

所以 **`SET NAMES` 之后不需要任何"重置词法器"的动作**——下一条语句自动换用新字符集的映射，词法行为立刻变化。这也解释了"为什么 utf8mb4 下某些非 ASCII 字符能作标识符、latin1 下不行"：标识符字符集合是字符集相关的（`ident_map` 由该字符集的 ctype 决定）。

### 标识符与文件名的字符集

**数据值有列字符集，标识符（表名/列名/文件名）走的却是另一套完全独立的规则**——这不只是"用别的 collation"，而是三层各自不同的字符集体系。理解它才能解释 `lower_case_table_names` 的行为差异与"为什么表名有大小写规则但列值没有"。

#### system_charset_info：标识符的规范字符集（编译期硬编码）

```cpp
// sql/mysqld.cc —— "Must be initialized early for comparison of options name"
system_charset_info = &my_charset_utf8mb3_general_ci;
```

它是**编译期硬编码**指向 `my_charset_utf8mb3_general_ci`（id 33）的全局指针，**运行期不可配置**——`--character-set-server` 改的是 `default_charset_info`/`collation_server`（会话/表的默认值），不是它。`character_set_system` 系统变量是 READ_ONLY，绑定的就是它。

用途是"**标识符与元数据文本的规范比较语义**"——大小写不敏感、与连接字符集无关、UTF8MB3 语义。五类典型调用点：

| 场景 | 例子 |
|---|---|
| 标识符合法性 | `check_table_name()` 用 `my_isspace`/`my_ismbchar(system_charset_info, ...)` 计字符数 |
| DD 内存对象列名比较 | `abstract_table_impl.cc` 的 `drop_column`/`get_column`：注释 "Column names are case-insensitive" |
| DD 缓存 / MDL 名字 | `dictionary_client.cc` 比较 `information_schema` |
| InnoDB 表定义对照 | `dict0dd.cc` 的索引名、列重命名、主键名比对 |
| SQL 层约束/键名 | `sql_table.cc`（全文件 80+ 处）：禁止把索引起名成 `PRIMARY`、键名冲突检测 |

**另一个标识符字符集是 `table_alias_charset`**——它**随 `lower_case_table_names` 变化**（见下），用于表**别名**比较。

注意与"标识符相关的还有第三个概念"：SQL **关键字**既不用 `system_charset_info` 也不用列的 collation，而是用**独立的 latin1 关键字表**（`lex_casecmp`，`sql_lex_hash.cc`，注释"based on the latin1 character set, and is only used when comparing keywords"）——三层字符集各有各的用途。

**★ 五个全局字符集指针的编译期硬编码**（`sql/mysqld.cc:10222`，★ 全在同一处赋值）：

```cpp
  /* Character sets */
  system_charset_info   = &my_charset_utf8mb3_general_ci;  /* ★ id 33，mbmaxlen=3 */
  files_charset_info    = &my_charset_utf8mb3_general_ci;  /* ★ 文件名折叠，默认同 system */
  national_charset_info = &my_charset_utf8mb3_general_ci;  /* ★ N'' / NATIONAL */
  table_alias_charset   = &my_charset_bin;                 /* ★★ 表别名用 binary：大小写敏感 */
  character_set_filesystem = &my_charset_bin;              /* ★★ 默认 binary ⇒ 文件系统层不做转换 */
```

```cpp
/* 被指向的对象本体 strings/ctype-utf8.cc:5820 */
CHARSET_INFO my_charset_utf8mb3_general_ci = {
    33, 0, 0,                                          /* ★ number = 33 */
    MY_CS_COMPILED | MY_CS_PRIMARY | MY_CS_STRNXFRM | MY_CS_UNICODE,
    "utf8mb3", "utf8mb3_general_ci", "UTF-8 Unicode",
    ... 1, 3, 1,    /* mbminlen=1, mbmaxlen=3, mbmaxlenlen=1 */
    0, 0xFFFF, ' ', false, 1,
    &my_charset_utf8mb3_handler, &my_collation_utf8mb3_general_ci_handler,
    PAD_SPACE};
```

> ★★ **`system_charset_info` 是 utf8mb3（`mbmaxlen = 3`）不是 utf8mb4**——这是"标识符里放不进 4 字节 emoji"的根本原因，与 `--character-set-server` 无关。

**★ `my_charset_filename` 的两套 handler**（★ `cset` 只换 `mb_wc`/`wc_mb`，`coll` 全盘复用 utf8mb3）：

```cpp
static MY_COLLATION_HANDLER my_collation_filename_handler = {
    nullptr, nullptr,
    my_strnncoll_utf8mb3, my_strnncollsp_utf8mb3,       /* ★ 全盘复用 utf8mb3 */
    my_strnxfrm_unicode, my_strnxfrmlen_utf8mb3,
    my_like_range_mb, my_wildcmp_utf8mb3, my_strcasecmp_utf8mb3,
    my_instr_mb, my_hash_sort_utf8mb3, my_propagate_complex};

static MY_CHARSET_HANDLER my_charset_filename_handler = {
    nullptr,
    my_ismbchar_utf8mb3, my_mbcharlen_utf8mb3, my_numchars_mb, my_charpos_mb3,
    my_well_formed_len_mb, my_lengthsp_8bit, my_numcells_mb,
    my_mb_wc_filename,          /* ★★ 关键：filename 的编解码是"特制"的 */
    my_wc_mb_filename,
    my_mb_ctype_mb, my_caseup_str_utf8mb3, my_casedn_str_utf8mb3,
    my_caseup_utf8mb3, my_casedn_utf8mb3,
    my_snprintf_8bit, ... my_scan_8bit};

CHARSET_INFO my_charset_filename = {
    17, 0, 0,
    MY_CS_COMPILED | MY_CS_PRIMARY | MY_CS_STRNXFRM | MY_CS_UNICODE |
        MY_CS_HIDDEN | MY_CS_NONASCII,      /* ★★ HIDDEN + NONASCII */
    "filename", "filename", "",
    nullptr, nullptr,
    ctype_utf8mb3, to_lower_utf8mb3, to_upper_utf8mb3, to_upper_utf8mb3,
    nullptr, nullptr, nullptr, &my_unicase_default,
    nullptr, nullptr,                        /* ★★ state_map / ident_map 全 nullptr */
    1, 1, 1,
    1, 5, 1,                                 /* ★★ mbminlen=1, mbmaxlen=5 */
    0, 0xFFFF, ' ', false, 1,
    &my_charset_filename_handler, &my_collation_filename_handler,
    PAD_SPACE};
```

> ★★ **`mbmaxlen = 5` 不是 3**——`filename` 把 0x80–0xFF 的"原始字节"编码成多字节序列，最坏膨胀到 5 字节。
> ★★ **`MY_CS_NONASCII` 的连锁后果**：`my_convert()` 开头就判 `(to_cs->state | from_cs->state) & MY_CS_NONASCII`，所以任何涉及 `filename` 的转换**一个字节都不走 ASCII 快路径**，全量走 `my_convert_internal()` 慢路径。
> ⚠️ **`state_map` / `ident_map` 都是 nullptr** —— `filename` 字符集**不能用于词法解析**。
> ⚠️ **`my_charset_filename` 没有被 `add_compiled_collation()` 注册**（`charset-def.cc` 里没有它），所以 `SHOW CHARACTER SET` / `SHOW COLLATION` 都看不到它。它是唯一带 `MY_CS_HIDDEN` 的编译型字符集。

**★ `lower_case_table_names` 的三层折叠——每层用的字符集都不同**：

```cpp
/* 第 1 层：SQL/DD 层标识符规范化 → system_charset_info 或 utf8mb3_tolower_ci */
    my_casedn_str(&my_charset_utf8mb3_tolower_ci, name_buf);      /* dd_schema.cc:95 / sql_db.cc:374 */
    my_casedn_str(system_charset_info, db_name_buff);             /* dd_table.cc:2489 / sp.cc:1201 */

/* 第 2 层：表名 → 文件名（★ 换字符集，不是折叠大小写） */
size_t tablename_to_filename(const char *from, char *to, size_t to_length) {
  uint errors;
  length = strconvert(system_charset_info, from, &my_charset_filename, to,
                      to_length, &errors);        /* ★★ utf8mb3 → filename(17) */
  if (check_if_legal_tablename(to) && length + 4 < to_length) {
    memcpy(to + length, "@@@", 4);                /* ★ 含非法字符（'/' '.'）⇒ 追加 "@@@" */
    length += 3;
  }
  return length;
}

/* 第 3 层：磁盘路径层（★ 只有 lctn==2 才折叠，且跳过 data_home / tmpdir） */
const char *get_canonical_filename(handler *file, const char *path, char *tmp_path) {
  if (lower_case_table_names != 2 || (file->ha_table_flags() & HA_FILE_BASED))
    return path;                                  /* ★★ 只有 lctn==2 才走 */
  ...
  if (tmp_path != path) my_stpcpy(tmp_path, path);
  /* we only should turn into lowercase database/table part
     so start the process after homedirectory */
  my_casedn_str(files_charset_info, tmp_path + mysql_data_home_len);
  /*  ★★ 只对 data_home 之后的 db/table 部分折叠 */
  return tmp_path;
}
```

```cpp
/* 临时表名（sql_table.cc:926） */
  if (lower_case_table_names) {
    /* Convert all except tmpdir to lower case */
    my_casedn_str(files_charset_info, p);         /* ★ 同样跳过 tmpdir */
  }

/* COM_FIELD_LIST 路径（sql_parse.cc:2193）—— ★ 返回值必须回写 length */
      if (lower_case_table_names && !is_infoschema_db(db.str, db.length))
        table_name.length = my_casedn_str(files_charset_info, table_name.str);
      /*  ★★ my_casedn_str 返回新长度（可能变短），必须回写 */
```

> ⚠️ **`my_stpcpy_dn()` 不存在**（全仓 0 命中）——折叠序列固定为 `my_stpcpy`/`my_stpnmov` 拷路径 + `my_casedn_str(files_charset_info, 偏移)`。
> ★★ **`my_caseup_str()` / `my_casedn_str()` 是宏不是函数**：`#define my_casedn_str(s, a) ((s)->cset->casedn_str((s), (a)))`。真函数叫 `my_casedn_str_8bit` / `my_casedn_str_utf8mb4`。

**★ 大小写转换的"字节膨胀陷阱"**（★ `casedn_utf8mb4` 在原地转换且字节数变了时**直接返回 `(size_t)-1`**）：

```cpp
static size_t my_casedn_8bit(const CHARSET_INFO *cs, char *src, size_t srclen,
                             char *dst, size_t dstlen) {
  const uchar *map = cs->to_lower;
  assert(src == dst && srclen == dstlen);    /* ★★ 强制原地：8bit 大小写不改变字节数 */
  for (; src != end; src++) *src = (char)map[(uchar)*src];
  return srclen;
}

static size_t my_casedn_utf8mb4(const CHARSET_INFO *cs, char *src, size_t srclen,
                                char *dst, size_t dstlen) {
  assert(src != dst || cs->casedn_multiply == 1);
  bool inplace = (src == dst);
  while ((src < srcend) && (srcres = my_mb_wc_utf8mb4(&wc, src, srcend)) > 0) {
    my_tolower_utf8mb4(uni_plane, &wc);
    if ((dstres = my_wc_mb_utf8mb4(cs, wc, dst, dstend)) <= 0 && !inplace) break;
    if (srcres == dstres || !inplace) { src += srcres; dst += dstres; }
    else {
      return -1;      /* ★★ inplace 且字节数变了 ⇒ 返回 (size_t)-1 */
    }
  }
  return (size_t)(dst - dst0);
}
```

> ★ 注释里的例子：`U+0130`（İ，utf8 两字节 `0xC4B0`）小写成 `U+0069`（i，一字节 `0x69`）——**变小**；反过来 `Ⱦ`(2B) → `ⱦ`(3B) **变大**。所以 `my_casedn_str_utf8mb4()` 最后必须补 `*dst = '\0'`。

**★ utf8mb4 的 cset 回调四个**（★ 真名勘误密集区）：

```cpp
/* ① well_formed_len：区分"不完整"与"非法" */
static size_t my_well_formed_len_utf8mb4(const CHARSET_INFO *cs, const char *b,
                                         const char *e, size_t pos, int *error) {
  while (pos) {
    int mb_len;
    if ((mb_len = my_valid_mbcharlen_utf8mb4(cs, b, e)) <= 0) {
      *error = b < e ? 1 : 0;      /* ★★ 还有剩余字节才算真非法 */
      break;
    }
    b += mb_len; pos--;
  }
  return (size_t)(b - b_start);
}

/* ② ismbchar：★ 返回"字符长度"不是 bool！ */
static uint ALWAYS_INLINE my_ismbchar_utf8mb4_inl(const CHARSET_INFO *cs, const char *b, const char *e) {
  int res = my_valid_mbcharlen_utf8mb4(cs, (const uchar *)b, (const uchar *)e);
  return (res > 1) ? res : 0;      /* ★★ 单字节/非法 → 返回 0 */
}

/* ③ charpos：★★ 真名 my_charpos_mb4，且全仓唯一的 8 字节 ASCII 位图快路径在这里 */
size_t my_charpos_mb4(const CHARSET_INFO *cs, const char *pos, const char *end, size_t length) {
  size_t min_length = std::min<size_t>(end - pos, length);
  const char *safe_end = std::min(end, pos + min_length) - std::min<size_t>(7, min_length);
  const char *start = pos;
  while (pos < safe_end) {
    uint64_t data;
    memcpy(&data, pos, sizeof(data));
    if (data & 0x8080808080808080ULL) break;    /* ★★ 8 字节一组位图预筛 */
    pos += sizeof(data); length -= sizeof(data);
  }
  while (length && pos < end) {
    uint mb_len;
    pos += (mb_len = my_ismbchar_utf8mb4_inl(cs, pos, end)) ? mb_len : 1;
    length--;
  }
  return (size_t)(length ? end + 2 - start : pos - start);   /* ★ 越界哨兵 */
}

/* ④ mbcharlen：★ 0x80–0xC1 与 0xF8–0xFF 都是非法首字节 */
static uint my_mbcharlen_utf8mb4(const CHARSET_INFO *cs, uint c) {
  if (c < 0x80) return 1;
  if (c < 0xc2) return 0;   /* Illegal mb head */
  if (c < 0xe0) return 2;
  if (c < 0xf0) return 3;
  if (c < 0xf8) return 4;
  return 0;                 /* Illegal mb head */
}
```

> ⚠️ **不存在 `my_charpos_utf8mb4()`** → 真名 **`my_charpos_mb4()`**（`ctype-utf8.cc:7734`）。utf8mb3/gbk/big5/sjis 用 `my_charpos_mb3()`，8bit 用 `my_charpos_8bit()`。
> ⚠️ **`my_ismbchar_utf8mb4()` 本身没有位图预筛**——它只是对 `my_valid_mbcharlen_utf8mb4()` 的三行薄封装；位图快路径**只存在于 `my_charpos_mb4()`**，全仓仅此一处。

**★ 解码与编码的不对称**（★ 两处硬证据）：

```cpp
/* 解码侧（strings/mb_wc.h:115，模板化、在头文件里） */
  if (c < 0xf0) {
    /* 三字节：一次 memcmp 两个 continuation 字节（大小端无关） */
    uint16 two_bytes; memcpy(&two_bytes, s + 1, sizeof(two_bytes));
    if ((two_bytes & 0xc0c0) != 0x8080) return MY_CS_ILSEQ;
    *pwc = ((c & 0x0f) << 12) + ((s[1] & 0x3f) << 6) + (s[2] & 0x3f);
    if (*pwc < 0x800) return MY_CS_ILSEQ;                        /* ★ 拒绝 overlong */
    if (*pwc >= 0xd800 && *pwc <= 0xdfff) return MY_CS_ILSEQ;    /* ★ 拒绝代理对区间 */
    return 3;
  }
  if (SUPPORT_MB4) {
    uint32 four_bytes; memcpy(&four_bytes, s, sizeof(four_bytes));
#ifdef WORDS_BIGENDIAN
    if ((four_bytes & 0xf8c0c0c0) != 0xf0808080) return MY_CS_ILSEQ;
#else
    if ((four_bytes & 0xc0c0c0f8) != 0x808080f0) return MY_CS_ILSEQ;
#endif
    /*  ★★ 三字节用"大小端无关"写法，四字节却被大小端分叉 —— 不对称之一 */
    *pwc = ((c & 0x07) << 18) + ...;
    if (*pwc < 0x10000 || *pwc > 0x10ffff) return MY_CS_ILSEQ;   /* ★ 上界 U+10FFFF */
    return 4;
  }
```

```cpp
/* 编码侧（strings/ctype-utf8.cc:7238，非模板、在 .cc 里） */
static int my_wc_mb_utf8mb4(const CHARSET_INFO *cs, my_wc_t wc, uchar *r, uchar *e) {
  if (wc < 0x80) count = 1; else if (wc < 0x800) count = 2;
  else if (wc < 0x10000) count = 3;
  else if (wc < 0x200000) count = 4;      /* ★★ 上界 0x200000 ≠ 解码侧 0x10FFFF */
  else return MY_CS_ILUNI;
  if (r + count > e) return MY_CS_TOOSMALLN(count);
  switch (count) {                        /* ★★ 倒序 + fallthrough：从 r[3] 写到 r[0] */
    case 4: r[3] = 0x80 | (wc & 0x3f); wc >>= 6; wc |= 0x10000; [[fallthrough]];
    case 3: r[2] = 0x80 | (wc & 0x3f); wc >>= 6; wc |= 0x800;   [[fallthrough]];
    case 2: r[1] = 0x80 | (wc & 0x3f); wc >>= 6; wc |= 0xc0;    [[fallthrough]];
    case 1: r[0] = (uchar)wc;
  }
  return count;
}
```

> ★★ **`wc ∈ (0x10FFFF, 0x200000]` 可以被"编"出来却无法被"解"回来**——这是编码侧上界比解码侧宽的直接后果。
> ★ `mb_wc` 注册的是 **`my_mb_wc_utf8mb4_thunk`**（因为模板函数签名里没有 `CHARSET_INFO*`），而 `wc_mb` **直接注册**无需 thunk——**不对称之二**。
> ★ `my_valid_mbcharlen_utf8mb4()` 没有独立实现，就是"试着解码一次看返回几个字节"。

#### my_charset_filename：一个独立的隐藏字符集（number = 17）

这是最容易被误解的一处：**磁盘文件名不是 `utf8mb3_bin`，也不是普通 utf8mb3**——MySQL 定义了一个**专用隐藏字符集** `my_charset_filename`：

```c
CHARSET_INFO my_charset_filename = {
    17, 0, 0,          /* number = 17 */
    MY_CS_COMPILED | MY_CS_PRIMARY | MY_CS_STRNXFRM | MY_CS_UNICODE |
        MY_CS_HIDDEN | MY_CS_NONASCII,      /* ★ MY_CS_HIDDEN */
    "filename", "filename", "", ...
    /* collation handler 直接复用 utf8mb3 的（strnncoll/strnxfrm/strcasecmp） */
    /* cset handler 只换掉 mb_wc/wc_mb —— 编码规则是 filename 专属的 */
    nullptr, nullptr,   /* state_map / ident_map 都是 nullptr：不能用于解析 */
    ...
    1, 5, 1,            /* mbminlen=1, mbmaxlen=5（转义后最长 5 字节）, mbmaxlenlen=1 */
};
```

它的编码规则（`filename_safe_char[128]`）只允许 `\0 A-Z a-z 0-9 _` 原样输出，**其他字符转义**：

- 转义符是 **`'@'`**（`#define MY_FILENAME_ESCAPE '@'`）
- 字母类不安全字符 → `'@'` + 2 字节 base-80 编码（3 字节总长，故 `mbmaxlen=5` 对应 hex 形式）
- 其他 → `'@'` + **4 位十六进制**（如 `$` → `@0024`）

两个往返安全机制的细节：

1. **Windows 保留名**：名字若命中保留名列表（`CLOCK$` 之类），`tablename_to_filename` **额外追加 `"@@@"`**——因为 `CLOCK$` 本身已被转义成 `CLOCK@0024`，再加 `@@@` 后缀才能与"真的叫 `CLOCK@0024` 的表"区分开。
2. **临时表前缀**：`#sql...` 前缀的名字在 `filename_to_tablename` 里**直接拷贝不转换**（临时表名不经 filename 编码）。

转换对：`tablename_to_filename`（UTF8MB3 → filename）/ `filename_to_tablename`（反向），底层是 `strconvert()`。**错误处理有历史遗留痕迹**：反向转换 `errors != 0` 意味着这是 5.0 时代未转义的旧名字，报 `ER_INVALID_OR_OLD_TABLE_OR_DB_NAME`。

InnoDB 侧包装成 `dict_name::file_to_table` / `table_to_file`；`dict0dict.cc` 注释明确："所有表名内部以 my_charset_filename 存储（临时表例外），外键名以 UTF-8 存储"——**同一张表的"表名"与"外键名"用了两种不同编码**。

#### DD 里的标识符 collation 矩阵

数据字典的**名字列 collation 不是统一的**，而是按对象类型在代码里显式声明（`name_collation()` 虚函数覆盖）：

| DD 表 | 名字列 collation | 备注 |
|---|---|---|
| `mysql.tables` | `fs_name_collation()`（随 lctn：0=bin / 1,2=tolower_ci） | 表名，随 `lower_case_table_names` 变 |
| `mysql.columns` / `indexes` / `check_constraints` / `table_partitions` | **`utf8mb3_tolower_ci`**（固定） | 列名/索引名/约束名/分区名 |
| `mysql.foreign_keys.name` | **`utf8mb3_general_ci`** | FK 名（SQL 层也是 `my_strcasecmp` 比较） |
| `mysql.foreign_keys.referenced_table_*` | `fs_name_collation()` | 引用表名，随 lctn |
| （非标识符）`mysql.tables.engine` | `utf8mb3_general_ci` | 同一张表内多种 collation 并存 |

```cpp
// object_table_definition_impl.h
static const CHARSET_INFO *name_collation() { return &my_charset_utf8mb3_bin; }   // 默认
static const CHARSET_INFO *fs_name_collation() {                                  // 文件系统名
  if (lower_case_table_names == 0) return &my_charset_utf8mb3_bin;
  return &my_charset_utf8mb3_tolower_ci;
}
```

而 DD **内存对象**（`dd::String_type` = `std::basic_string`，按字节语义）之间的比较**不用 collation**，靠显式调 `my_strcasecmp(system_charset_info, ...)` 或键对象的 collation 比较。

#### lower_case_table_names：三层折叠，都靠 `casedn` 而不是 `sort_order`

lctn 的 0/1/2 差别不在"是否折叠"，而在**谁在哪一层折叠**：

| 层 | 折叠用的字符集 | 触发条件 |
|---|---|---|
| SQL / MDL 层 | `files_charset_info`（= system_charset_info）或 `table_alias_charset`（= `utf8mb3_tolower_ci` 当 lctn≠0） | lctn≠0 时 DB 名/别名折叠 |
| DD 层 | `fs_name_collation()`（= `utf8mb3_tolower_ci` 当 lctn≠0） | 构造 MDL key / 查找键时折叠（**lctn=2 只折查找键，不改存储**） |
| InnoDB 文件名 | **`my_charset_filename`** | `innobase_casedn_path()` 折路径 |

**关键澄清：折叠调用的是 `cset->casedn`（大小写映射表），不是 `sort_order`**——`my_charset_utf8mb3_tolower_ci` / `general_ci` / `filename` 的 `casedn` 都是 `my_casedn_utf8mb3`，且这些 collation 的 `casedn_multiply = 1`，所以折叠不改变字节长度。

三种模式的行为：

- **lctn=0**：不折叠，DD/文件名 collation 是 `utf8mb3_bin`（区分大小写）
- **lctn=1**：**名字以小写为规范形式**——由升级/导入路径强制（`sdi_api.cc` 直接 `casedn` 后存 SDI；升级路径校验"表名必须全小写"，否则报 `ER_TABLE_NAME_IN_UPPER_CASE_NOT_ALLOWED`）
- **lctn=2**：输入名保持原样存储，但所有**查找键/MDL/文件名**折叠成小写（`fs_name_case()`）

```cpp
// sql/mysqld.cc：lctn 定下来后重置别名字符集
lower_case_file_system = (test_if_case_insensitive(mysql_real_data_home) == 1);
table_alias_charset = (lower_case_table_names ? &my_charset_utf8mb3_tolower_ci : &my_charset_bin);
```

#### 标识符的字符集校验：词法层管"能切分"，语义层管"合法"

两条独立的规则：

1. **能切分成标识符**——由**连接字符集**的 `ident_map` 决定（词法层）。所以"非 ASCII 名"的接受度随 `SET NAMES` 变化。
2. **切出来之后是否合法**——由 `system_charset_info` 判定（语义层）：

```cpp
// sql/table.cc check_table_name()
if (!length || length > NAME_LEN) return Ident_name_check::WRONG;      // 64 字节上限
// 用 system_charset_info 逐字符计数（my_isspace / my_ismbchar）
if (name_length > NAME_CHAR_LEN) return Ident_name_check::TOO_LONG;    // 64 字符上限
if (末尾是空格) return Ident_name_check::WRONG;
```

语法层在 `IDENT_sys` 规则里就把标识符**从连接字符集转成 `system_charset_info`**（`thd->convert_string(&$$, system_charset_info, ...)`）；另有一个 `TEXT_STRING_filesystem` 规则转成 `character_set_filesystem`（文件名的字符集入口）。

#### 对象类型 × 字符集对照

| 对象 | SQL 层比较 | DD 存储 collation | 磁盘/内部编码 |
|---|---|---|---|
| 表名 / 库名 / 分区名 | `check_table_name()`（system_charset_info） | `fs_name_collation()`（随 lctn） | `my_charset_filename`（转义编码） |
| 列名 / 索引名 | `my_strcasecmp(system_charset_info)` | `utf8mb3_tolower_ci` | UTF8MB3 |
| 外键名 | `my_strcasecmp(system_charset_info)`（"Names of foreign keys in InnoDB are case-insensitive."） | `utf8mb3_general_ci` | **UTF-8**（与表名不同！） |
| 检查约束名 | `my_strcasecmp(system_charset_info)` | `utf8mb3_tolower_ci` | UTF8MB3 |
| 索引名（InnoDB 内部） | 名字长度按 `MAX_TABLE_UTF8MB3_LEN` 计算 | — | 分区表名拼成 `table<PART_SEPARATOR>partition<SUB_PART_SEPARATOR>subpartition` |
| `PRIMARY` | 保留名特例（`ER_WRONG_NAME_FOR_INDEX`） | — | — |
### 协议层转换链

**为什么以 Unicode 为 pivot**：任意两个字符集互转若都实现直接转换函数，N 个字符集需要 **N×(N-1)** 个转换函数；改为"源 → Unicode → 目标"后，每个字符集只需实现 `mb_wc` 与 `wc_mb` 两个函数，**总量降到 2N**。这是字符集转换层最根本的一个设计决策。代价是每次转换都要经过"解码 + 编码"两趟，无法做字符集间的字节级捷径。

转换的完整上层入口：`String::copy`（sql-common/sql_string.cc）先 `needs_conversion()` 判断是否要转，binary 源直接 `copy_aligned()` 字节级拷贝，其余走 `copy_and_convert` → `my_convert`（strings/ctype.cc）。发送端 `Protocol_classic` 判断 `!my_charset_same(fromcs, result_cs) && 双方非 binary` 才转换（character_set_results）。

#### 结果集里两套字符集：元数据名字 与 列值

一个常见的过度简化是"元数据一律 utf8mb4，不受 `character_set_results` 影响"。**源码不是这样**——元数据的名字字段在发送时也会转换：

```c
// Protocol_classic::send_field_metadata
const CHARSET_INFO *cs = system_charset_info;      // ★ 源字符集：utf8mb3_general_ci
...
store_string(STRING_WITH_LEN("def"), cs) ||
store_string(field->db_name,    strlen(field->db_name),    cs) ||
store_string(field->table_name, strlen(field->table_name), cs) ||
store_string(field->col_name,   strlen(field->col_name),   cs) ...
```

名字（catalog/db/table/col/org_*）的**源字符集是 `system_charset_info`（utf8mb3）**，但每个都过 `store_string(..., cs)`——而 `store_string` 会按 `result_cs`（= `character_set_results`）转换：

```c
// Protocol_classic::store_string —— 唯一的字符串转换点
if (result_cs != nullptr && !my_charset_same(fromcs, result_cs) &&
    fromcs != &my_charset_bin && result_cs != &my_charset_bin)
  return net_store_data_with_conversion(...);   // 转换
return net_store_data(...);                      // 原样发
```

**所以元数据名字的线上编码确实受 `character_set_results` 影响**。而列定义里的 **`charsetnr`（collation 号）字段**另有规则：

```c
if (item_charset == &my_charset_bin || thd_charset == nullptr) {
  int2store(pos, item_charset->number);    // 不转换：发列自身的 charset 号
} else {
  int2store(pos, thd_charset->number);     // 有转换：发 character_set_results 的号
}
```

**值侧的规则更窄**：只有字符串类型走 `store_string` 做转换；`store_longlong`/`store_double`/`store_decimal` **全部经 `longlong10_to_str` 之类的纯 ASCII 格式化**，与字符集无关。`SET character_set_results = NULL` 时 `result_cs == nullptr` → **一律原样发送源字节**（客户端自理），此时 `String::needs_conversion` 的 `to_cs == nullptr` 短路也一致。

**★ `store_string` / `net_store_data_with_conversion` 完整源码**（⚠️ 真名 `Protocol_classic`，不是 `Protocol_text`）：

```cpp
bool Protocol_classic::store_string(const char *from, size_t length,
                                    const CHARSET_INFO *fromcs) {
  field_pos++;
  // result_cs is nullptr when client issues SET character_set_results=NULL
  if (result_cs != nullptr && !my_charset_same(fromcs, result_cs) &&
      fromcs != &my_charset_bin && result_cs != &my_charset_bin) {
    /*  ★★ 四个条件全满足才转：①results 非 NULL ②两者不"同字符集" ③源非 binary ④目标非 binary */
    return net_store_data_with_conversion(pointer_cast<const uchar *>(from),
                                          length, fromcs, result_cs);
  }
  return net_store_data(pointer_cast<const uchar *>(from), length, packet);
}

bool Protocol_classic::net_store_data_with_conversion(const uchar *from, size_t length,
                                                      const CHARSET_INFO *from_cs,
                                                      const CHARSET_INFO *to_cs) {
  uint dummy_errors;
  size_t conv_length = to_cs->mbmaxlen * length / from_cs->mbminlen;
  if (conv_length > 250) {
    /* For strings with conv_length greater than 250 bytes we don't know how
       many bytes we will need to store length: one or two, because we don't
       know result length until conversion is done. For example, when
       converting from utf8 (mbmaxlen=3) to latin1, conv_length=300 means that
       the result length can vary between 100 to 300. ... Thus conversion
       directly to "packet" is not worthy. Let's use "convert" as a temporary
       buffer. */
    return (convert.copy((const char *)from, length, from_cs, to_cs, &dummy_errors) ||
            net_store_data((const uchar *)convert.ptr(), convert.length(), packet));
  }
  /*  ★ ≤250：直接转进 packet，"原地"转换，先占 1 字节长度位再回填 */
  size_t packet_length = packet->length();
  size_t new_length = packet_length + conv_length + 1;
  if (new_length > packet->alloced_length() && packet->mem_realloc(new_length)) return true;
  char *length_pos = packet->ptr() + packet_length;
  char *to = length_pos + 1;                    /* ★ 先占 1 字节长度位 */
  to += copy_and_convert(to, conv_length, to_cs, (const char *)from, length,
                         from_cs, &dummy_errors);
  net_store_length((uchar *)length_pos, to - length_pos - 1);
  packet->length((uint)(to - packet->ptr()));
  return false;
}
```

> ★★ **`>250` 这个阈值的真正原因**：长度前缀是 1 还是 2 字节取决于转换**后**的长度，而转换前不知道 → 无法预留。所以大于 250 就先转进临时 `String convert`。

**★ `THD::update_charset()`：用 `needs_conversion(0, ...)` 做"两字符集是否等价"的缓存**：

```cpp
void THD::update_charset() {
  size_t not_used;
  charset_is_system_charset = !String::needs_conversion(
      0, variables.character_set_client, system_charset_info, &not_used);
  charset_is_collation_connection = !String::needs_conversion(
      0, variables.character_set_client, variables.collation_connection, &not_used);
  charset_is_character_set_filesystem = !String::needs_conversion(
      0, variables.character_set_client, variables.character_set_filesystem, &not_used);
  /*  ★★ 传 arg_length=0 ⇒ binary 分支 offset 必为 0，退化为纯粹的"两 cs 是否等价"判定 */
}
```

```cpp
/* sql/sql_class.h:3307 —— ★ THD::charset() 返回的是 character_set_client */
  const CHARSET_INFO *charset() const { return variables.character_set_client; }
```

```cpp
/* sql/system_variables.h:319 —— ★ 六个变量分两组 */
  /* Only charset part of these variables is sensible */
  const CHARSET_INFO *character_set_filesystem;
  const CHARSET_INFO *character_set_client;
  const CHARSET_INFO *character_set_results;
  /* Both charset and collation parts of these variables are important */
  const CHARSET_INFO *collation_server;
  const CHARSET_INFO *collation_database;
  const CHARSET_INFO *collation_connection;
```

> ⚠️ **没有独立的 `character_set_server` / `character_set_database` 变量**——它们由 `collation_server` / `collation_database` 兼任。
> ⚠️ **`charsets_dir` 不带 `my_` 前缀**（`mysys/charset.cc:356`）；服务端侧缓冲名是 `mysql_charsets_dir`（`mysqld.cc:1490`）。**不存在 `my_charset_dir`**。
> ⚠️ **`LC_ALL` 与字符集加载无关**——全仓仅在 `client/mysql.cc:2733` 出现一次 `setlocale(LC_ALL, "")`；**服务端 mysqld 内部没有任何 `setlocale`**。字符集完全由 `Index.xml` + 编译进来的结构体决定，不受 locale 环境变量影响。
> ⚠️ **`sql/field_common_properties.h` 与字符集完全无关**——全文只有 9 个 `enum_field_types` 判定 inline 函数（is_integer_type / is_string_type / is_temporal_* 等）。文件头注释说明它是给 mysqlbinlog 这类工具共用的。
> ⚠️ **`sql/sql_string.cc` 不存在** —— 真文件是 **`sql-common/sql_string.cc`**；且 `needs_conversion()` 与 `copy_and_convert()` 都定义在**头文件 `include/sql_string.h`**（inline）。

```c
// my_convert 骨架
if (任一侧 MY_CS_NONASCII) return my_convert_internal(...);  // 直接慢路径
// 快路径：批量复制 ASCII（4 字节步进检查 0x80808080），
// 遇非 ASCII 字符才转入 internal 逐码点处理

// my_convert_internal
for (每个码点) {
  mb_wc(...) → wc;          // 源字节 → Unicode
  if (MY_CS_ILSEQ 或 无 Unicode 映射) {
    error_count++;
    写入 '?';               // 非法序列替换为问号
    continue;
  }
  wc_mb(...);               // Unicode → 目标字节
  if (目标无法编码，MY_CS_ILUNI) { error_count++; 写入 '?'; }
}
```

两层设计：ASCII 快路径（大部分数据是 ASCII，4 字节一批检查）包着逐码点慢路径；错误用 '?' 替换并计数，由上层决定告警还是报错。三个错误码的分工：`MY_CS_ILSEQ`（非法字节序列）、`MY_CS_TOOSMALL`（字节不够，非错）、`MY_CS_ILUNI`（合法码点但目标字符集没有）。

**一个隐蔽风险**：`needs_conversion()` 对 binary 源走 `copy_aligned()` **逐字节拷贝、无合法性检查**——把 utf8mb4 编码的字节（如 `X'E5BCA0'`）直接塞进 latin1 列不报错也不转码，读出来就是乱码。binary 是"绕过一切字符语义"的旁路，用时要清楚自己在搬字节不是搬字符。

#### 案例：mysqldump --default-character-set=binary 与列默认值（转换链上的诡异 bug）

第二个真实案例，正好打在"binary 源不转换"这条规则上——而且现象极具迷惑性。

**现象**：`mysqldump --default-character-set=binary` 导出的库，导入时中文默认值建表失败，**但只对奇数个汉字失败**：

```sql
SET NAMES binary;
CREATE TABLE t3 (name VARCHAR(256) NOT NULL DEFAULT '中文')   DEFAULT CHARSET=gbk;  -- 失败
CREATE TABLE t4 (name VARCHAR(256) NOT NULL DEFAULT '没签名') DEFAULT CHARSET=gbk;  -- 失败
-- 而偶数个汉字的默认值却能建成功
```

**根因链条**：

1. **mysqldump 用 binary 导出**：binary 字符集不涉及任何字符集转换，可避免"某字符在目标字符集没有对应编码"的数据丢失——这是它被选作备份编码的理由
2. **DD 不记录"默认值的字符集"**：8.0 的数据字典里 `dd::Column::default_value` 存的是**按列自身字符集 pack 好的原始字节**（SDI 里以二进制写出），字符集由该列的 `collation_id` 隐含（列未显式指定时才继承表默认）；另有 `default_value_utf8` 是给 I_S/SHOW 用的文本表示。而 **`SHOW CREATE TABLE`（mysqldump 的数据源）输出的 `DEFAULT` 文本是 `system_charset_info`（默认 utf8mb3）**，与列字符集（gbk）解耦——这才是问题的关键前提
3. **导入时连接字符集是 binary**：字面量 `Item_string` 的 collation 取自解析器（`YYTHD->charset()` 即 `character_set_client`），所以默认值字面量被标成 **binary**；随后建表准备阶段要把它转成列字符集，走 `safe_charset_converter` → `String::copy` → **`needs_conversion(binary → gbk)` 恒为 false**（见下节），于是"只改标签、不改字节"
4. **但不转换不等于不校验**：复制后仍要用目标字符集（gbk）的 `well_formed_len` 做合法性检测。此时缓冲区里躺的是 **utf8mb4 编码的汉字（每字 3 字节）**，却拿 **gbk 的规则（汉字 2 字节）** 去判别 → 长度对不上 → 报错

**奇偶性的来源**（这个现象最迷惑人的地方）：utf8mb4 汉字 3 字节，奇数个汉字 = 奇数 × 3 = **奇数**，按 gbk 双字节规则切分后必然剩 1 个孤立字节 → 判定非法；偶数个汉字 = 偶数 × 3 = **偶数**，刚好能被 gbk 的 2 字节规则整除 → 误判为合法。**偶数成功不是因为逻辑正确，纯粹是字节数凑巧整除**——逻辑一直是错的，只是没暴露。

**把链路逐步摊开**（从 dump 到建表失败）：

```
① dump 侧：SHOW CREATE TABLE 把 gbk 默认值转成 system_charset_info(utf8mb3) 文本
   → mysqldump 强制 character_set_results=binary（原样取字节，不做结果集转码）
   → dump 文件里 "DEFAULT '中文'" 是 UTF-8 字节，而文件头却是 SET NAMES binary

② restore：连接字符集 binary
   → 字面量 Item_string 的 collation = YYTHD->charset() = character_set_client = binary

③ 建表准备（sql_table.cc mysql_prepare_create_table）：
   列字符集(gbk) != 字面量字符集(binary) 且类型是 STRING → safe_charset_converter
   → Item_string::charset_converter → String::copy(binary → gbk)
   → needs_conversion 返回 false（gbk mbminlen==1，len % 1 == 0 恒成立）
   → 只把标签改成 gbk，字节仍是 UTF-8，conv_errors = 0 —— "安全转换"判定通过

④ 默认值落 dd（default_values.cc prepare_default_value）：
   save_in_field(regfield) → Field_string::store(from, len, gbk)

⑤ field_well_formed_copy_nchars(gbk → gbk) 走分支 A：
   gbk 的 well_formed_len 在 UTF-8 字节上判定 → well_formed_error_pos != nullptr
   → check_string_copy_error 返回 TYPE_WARN_INVALID_STRING

⑥ prepare_default_value 看到 res != TYPE_OK → my_error(ER_INVALID_DEFAULT)
   → "Invalid default value for 'xxx'"，建表失败
```

第 ③ 步是整条链的**开关**：`needs_conversion` 只看了 `mbminlen` 对齐就判定"不需要转换"，于是 UTF-8 字节被贴上 gbk 标签蒙混过关，直到第 ⑤ 步才被 `well_formed_len` 抓出来。若 UTF-8 字节恰好也能拼成合法 gbk 序列，则会**静默落错码**（更隐蔽，不报错但数据乱码）。

**修复选择**：两个方案——改内核（`well_formed_copy_nchars` 语义）或改 mysqldump。最终选后者（尽量不动内核）：**mysqldump 在输出表定义前把 `character_set_connection` 设为 utf8mb4，表定义输出完再设回**。这样导入时走 utf8mb4→gbk 的正常转换路径，默认值被正确转码后通过校验。

**这个案例的教训**：binary 字符集是"**绕过转换**"而非"绕过校验"。源字符集为 binary 时系统逐字节复制、不做转码，但后续仍会按目标字符集做合法性检测——**"不转换"与"不检查"是两件事**。当缓冲区里的字节编码与目标字符集声明不一致时（utf8mb4 字节 + gbk 列），检测就会给出错误结论，而且错得有奇偶这样的迷惑性。

#### 两个 bug 的共同温床：把"字符集声明"当成事实

Bug#119463 与这个 mysqldump 案例看起来毫不相干，但根因同构——**都把字符集自己声明的属性当成了事实**，而真正的校验被推迟到别处：

| | 被信任的"声明" | 真实情况 | 校验实际发生在 |
|---|---|---|---|
| Bug#119463 | `caseup/casedn_multiply == 1`（"大小写不改字节数"） | 存在 2↔3 字节跨边界映射对 | 转换函数内部 `srcres != dstres`，但旧代码没检查 |
| mysqldump 案例 | `needs_conversion == false`（"binary 源不需要转换"） | 字节编码与目标字符集不符 | 推迟到 `Field::store` 的 `well_formed_len` |

两个 bug 都是"**声明失效但没人验证**"，且失效后果都表现为**静默的数据损坏**（截断 / 乱码）而非崩溃。

再叠加一个共同粗糙点：**`MY_CS_ILUNI == MY_CS_ILSEQ == 0`**——"目标装不下"与"源串非法"数值相同，调用方只能判 `<= 0`，无法区分原因，也就无法给出精确诊断。这类"错误码信息量不足"是字符集代码里反复出现的模式。

#### 复制链路中的字符集

binlog 也要记录字符集，否则从库回放时语义会漂：

- **Query 事件**：`Q_CHARSET_CODE` 携带 **6 字节打包的三个 uint16**（`character_set_client` / `collation_connection` / `collation_server`），另有 `Q_CHARSET_DATABASE_CODE`（库字符集）与 `Q_DEFAULT_COLLATION_FOR_UTF8MB4`（为跨版本复制而加）
- **Rows 事件**：**本身不带字符集**，列字符集走 Table_map 事件的 optional metadata（`DEFAULT_CHARSET` / `COLUMN_CHARSET`，存的是 **collation number**）
- **回放**：只有与上次缓存不同时才切换 thd 的字符集变量（`cached_charset` 比对）；**默认不做字符集转换**——只有打开 `slave_type_conversions` 时才经 `Copy_field` 转换，否则直接按从库表定义存储

即：主库 A 字符集、从库表 B 字符集时，**行数据不转换**（可能乱码或报错），而语句的字符集上下文由 Query 事件还原。

---

### 落盘路径：needs_conversion 与 well_formed_copy_nchars

前面讲的 `String::copy` / `my_convert` 是"**宽松转换**"——转不动就替换成 `?`、把错误计数累加返回。但数据**落盘**（`Field::store`）走的是另一条更严格的路，两者语义相反，必须分开理解。

#### needs_conversion：一个只判"要不要转"的廉价谓词

```c
inline bool String::needs_conversion(size_t arg_length, const CHARSET_INFO *from_cs,
                                     const CHARSET_INFO *to_cs, size_t *offset) {
  *offset = 0;
  if (to_cs == nullptr ||                       // character_set_results=NULL
      (to_cs == &my_charset_bin) ||             // 目标是 binary
      from_cs == to_cs ||                       // 同一 CHARSET_INFO 指针
      my_charset_same(from_cs, to_cs) ||        // 同一字符集名（不比排序规则）
      ((from_cs == &my_charset_bin) &&
       (0 == (*offset = (arg_length % to_cs->mbminlen)))))   // ★ 源 binary 且长度对齐
    return false;
  return true;
}
```

六条"不转换"组合里，**最后一条是坑**：源是 binary 且 `arg_length % mbminlen == 0` 就不转换。对 gbk（`mbminlen == 1`）而言 `len % 1 == 0` **恒成立**——所以 `binary → gbk` **永远不转换**。

`needs_conversion_on_storage` 是为补这个洞而存在的"更严格版"：存盘时只要源是 binary、目标非 binary，且目标是**变长编码**（`mbminlen != mbmaxlen`，gbk/utf8mb4 都满足）就**强制走转换**。注意 CREATE TABLE 的默认值路径**没有用它**（走的是 `safe_charset_converter` → `String::copy` → `needs_conversion`）——这正是 mysqldump 那个 bug 能成立的关键。

`copy_aligned` 则负责"binary → 定宽多字节（ucs2/utf32）"时补前导 0（把残片凑成一个完整字符），注释明确"只对大端 UCS-2 安全"。

`String::copy` 的完整分支：① 不转换 → `copy(str, len, to_cs)`，**只改 `m_charset` 标签、字节原封不动**；② binary 源且有残片 → `copy_aligned` 左补 0；③ 真转换 → `copy_and_convert`。

##### `String` 类：字节与字符的双长度、借用语义

`String` 是字符集代码里出现最多的载体，它的三个特性必须清楚：

```cpp
char *m_ptr;  size_t m_length;  const CHARSET_INFO *m_charset;
uint32 m_alloced_length;  bool m_is_alloced;
// ★ 默认构造的 charset 是 &my_charset_bin ——「未声明即字节串」
```

| 方法 | 语义 | 坑 |
|---|---|---|
| `length()` | **字节数**（`m_length` 恒为字节） | 当"字符数"用是错的 |
| `numchars()` | **字符数**（走 `cset->numchars` 逐字符 `mb_wc`） | utf8mb4 下 `"中".length()==3` 但 `numchars()==1` |
| `charpos(i)` | 第 i 个字符的**字节位置** | 需要字符定位时用它，别用 `ptr()+i` |
| `ptr()` | 内部指针，**注释明确"可能不含结束 NUL"** | 要 NUL 结尾得用 `c_ptr()`（必要时才 realloc） |
| `set(const char*, len, cs)` | **借用**外部缓冲（`m_is_alloced=false`） | 生命周期由调用方负责 |
| `copy(str, len, cs)` | 拷贝 + 设 charset，**不做任何合法性校验** | 要校验得走 `well_formed_copy_nchars` / 带转换的 `copy` |

**"借用"语义是理解 `copy_if_not_alloced` 的钥匙**：

```cpp
String *copy_if_not_alloced(String *to, String *from, size_t from_length) {
  if (from->m_is_alloced && from->m_alloced_length >= from_length) return from;  // 自有缓冲 → 就地改
  ... // 否则拷到 to，返回 to
}
```

用途：某 `Item` 想**就地修改**参数返回的字符串（如 `Item_func_insert`、`Item_str_conv`），但参数常是 `Field::val_str` **借出的记录缓冲**（`Field_string::val_str` 用 `set((const char*)ptr, length, field_charset)`，`m_is_alloced=false`）或 join buffer 里的记录——**直接改会写穿只读记录**，所以必须先隔离。对照：`Item_param::val_str` 返回的是**它自己拥有的** `str_value_ptr`（ps 参数），与 `Field::val_str` 的借用语义相反——这正是 `copy_if_not_alloced` 要判"是否已分配"的根源。

**三个陷阱**：

1. **`chop()` 只砍最后一个字节**——源码注释明确"字符串以多字节字符结尾时不安全"（UTF-8 下会砍掉半个字符）。
2. **跨字符集复用同一个 `String`**：字节与 charset 分开存，若只改 `ptr/length` 而忘 `set_charset`，`numchars()`/转换会按错的字符集解释 → 越界或乱码。安全做法是每次填充都显式带 cs 的 `set`/`copy`。
3. **`String` 的堆缓冲与 `THR_MALLOC`（mem_root）是两套**：`String` 对象本身可以走 mem_root 分配（`operator new(MEM_ROOT*)`），但**它内部的 `m_ptr` 缓冲是堆**（`my_malloc`，`STRING_PSI_MEMORY_KEY`）——误以为随 mem_root 释放会泄漏。

#### well_formed_copy_nchars：落盘前的守门人

这是 Field 层唯一的"带转换 + 带合法性校验 + 带字符数上限"的拷贝原语，被 `Field_string::store`（CHAR）、`Field_varstring::store`（VARCHAR）、`Field_blob::store`、`print_default_clause`（SHOW CREATE TABLE）调用。它有两支：

**分支 A（不转换，但仍校验）**——当两字符集相同 / 涉及 binary / `my_charset_same` 时：

```c
res = to_cs->cset->well_formed_len(to_cs, from, from + min_length, nchars, &well_formed_error);
if (res > 0) memmove(to, from, res);      // 只拷"合法前缀"
*from_end_pos = from + res;
*well_formed_error_pos = well_formed_error ? *from_end_pos : nullptr;
*cannot_convert_error_pos = nullptr;      // A 分支永远不会设它
```

**分支 B（真转换）**——逐字符 `mb_wc` → `wc_mb`：

```c
for (; nchars; nchars--) {
  if ((cnvres = (*mb_wc)(from_cs, &wc, from, from_end)) > 0) from += cnvres;
  else if (cnvres == MY_CS_ILSEQ) {                  // 源端非法序列
    if (!*well_formed_error_pos) *well_formed_error_pos = from;
    from++; wc = '?';
  } else if (cnvres > MY_CS_TOOSMALL) {              // 合法但无 Unicode 映射
    if (!*cannot_convert_error_pos) *cannot_convert_error_pos = from;
    from += (-cnvres); wc = '?';
  } else break;                                      // TOOSMALL：被截在字符中间
outp:
  if ((cnvres = (*wc_mb)(to_cs, wc, (uchar *)to, to_end)) > 0) to += cnvres;
  else if (cnvres == MY_CS_ILUNI && wc != '?') {     // 目标装不下
    if (!*cannot_convert_error_pos) *cannot_convert_error_pos = from_prev;
    wc = '?'; goto outp;
  } else { from = from_prev; break; }
}
```

三个出参的语义要分清：

| 出参 | 含义 | 只在 |
|---|---|---|
| `well_formed_error_pos` | 源串里**第一个非法序列**的起点 | A、B 都有 |
| `cannot_convert_error_pos` | 第一个"源合法但转不了"的字符起点（无 Unicode 映射 / 目标装不下） | **仅 B** |
| `from_end_pos` | 扫描停止点（用于判断是否被 `to_length`/`nchars` 截断） | A、B 都有 |

`nchars` 是**字符数不是字节数**（`Field_string::store` 传 `field_length / mbmaxlen`）。另外 `my_charset_same` 只比**字符集名不比排序规则**——`gbk_chinese_ci` 与 `gbk_bin` 走 A 分支。

**与 `String::copy` 的语义相反**：`String::copy` 是"尽量转换、转不动替换成 `?`、不回报位置"（用于协议输出、表达式求值）；`well_formed_copy_nchars` 是"不替换、遇到第一个问题就停下并回报位置"（用于严格落盘）。这解释了为什么同一个坏字节在 `SELECT` 里变成 `?`、在 `INSERT` 里却报错。

**★ A 分支的完整源码**（⚠️ **校验 + 拷贝一体**，`memmove` 就在函数体内，不是"只校验"）：

```cpp
/* sql-common/sql_string.cc:757 */
size_t well_formed_copy_nchars(const CHARSET_INFO *to_cs, char *to, size_t to_length,
                               const CHARSET_INFO *from_cs, const char *from,
                               size_t from_length, size_t nchars,
                               const char **well_formed_error_pos,
                               const char **cannot_convert_error_pos,
                               const char **from_end_pos) {
  size_t res;
  if ((to_cs == &my_charset_bin) || (from_cs == &my_charset_bin) ||
      (to_cs == from_cs) || my_charset_same(from_cs, to_cs)) {
    /* ★ 分支 A：同构路径 —— 不做 Unicode 往返，只做字节搬运 + 合法性扫描。
       这是落盘主路径（utf8mb4→utf8mb4 走这里） */

    if (to_cs == &my_charset_bin) {
      res = min(min(nchars, to_length), from_length);
      memmove(to, from, res);        /* ★ 目标是 binary：纯截断，不校验 */
      *from_end_pos = from + res;
      *well_formed_error_pos = nullptr;
      *cannot_convert_error_pos = nullptr;
    } else {
      int well_formed_error; uint from_offset;
      if ((from_offset = (from_length % to_cs->mbminlen)) &&
          (from_cs == &my_charset_bin)) {
        /* Copying from BINARY to UCS2 needs to prepend zeros sometimes:
           INSERT INTO t1 (ucs2_column) VALUES (0x01);  0x01 -> 0x0001 */
        uint pad_length = to_cs->mbminlen - from_offset;
        memset(to, 0, pad_length);
        memmove(to + pad_length, from, from_offset);
        /* In some cases left zero-padding can create an incorrect character.
           For example: INSERT INTO t1 (utf32_column) VALUES (0x110000);
           We'll pad the value to 0x00110000, which is a wrong UTF32 sequence! */
        if (to_cs->cset->well_formed_len(to_cs, to, to + to_cs->mbminlen, 1,
                                         &well_formed_error) != to_cs->mbminlen) {
          *from_end_pos = *well_formed_error_pos = from;
          *cannot_convert_error_pos = nullptr;
          return 0;
        }
        nchars--; from += from_offset; from_length -= from_offset;
        to += to_cs->mbminlen; to_length -= to_cs->mbminlen;
      }

      size_t min_length = min(from_length, to_length);
      /* If we operate on a multi-byte fixed-width character set, make sure the
         string wasn't truncated in the middle of a character. */
      if (to_cs->mbmaxlen > 1 && to_cs->mbmaxlen == to_cs->mbminlen)  // gbk is not fixed-width
        min_length -= min_length % to_cs->mbmaxlen;

      res = to_cs->cset->well_formed_len(to_cs, from, from + min_length, nchars,
                                         &well_formed_error);
      if (res > 0) memmove(to, from, res);     /* ★ 实际拷贝 */
      *from_end_pos = from + res;

      /* ★★ "截断" 与 "真非法" 的区分：再往后探一个字符 */
      if (well_formed_error && to_cs->mbmaxlen > 1 &&
          res > min_length - to_cs->mbmaxlen) {
        const char *from_end = from + min(res + to_cs->mbmaxlen, from_length);
        size_t extra = to_cs->cset->well_formed_len(
            to_cs, *from_end_pos, from_end, 1, &well_formed_error);
        well_formed_error = (res + extra < min_length);
      }
      /* ★ 解释：300 字节的合法串，第 255 字节处有个 3 字节字符跨过截断边界。
         第一次 well_formed_len 返回 254，看着像"非法"；再探一次发现 254..257
         构成一个完整字符 ⇒ 其实是"截断"而非"非法"。 */

      *well_formed_error_pos = well_formed_error ? *from_end_pos : nullptr;
      *cannot_convert_error_pos = nullptr;
      if (from_offset) res += to_cs->mbminlen;
    }
  } else { /* 分支 B：真转换 —— 见上文骨架 */ }
  return res;
}
```

> ★★ **`my_well_formed_len_utf8mb4()` 里有个关键的"不完整 vs 非法"区分**：`*error = b < e ? 1 : 0;`——**还有剩余字节才算真非法**，刚好到缓冲区末尾只是"不完整"，`error = 0`。这是 `TOOSMALL` 与 `ILSEQ` 的分界。

**★ 三个 Field 的 `nchars` 与 `count_spaces` 各不相同**（这是 CHAR/VARCHAR/BLOB 行为差异的根源）：

| Field | `nchars` 实参 | `count_spaces` | 尾部空格处理 |
|---|---|---|---|
| `Field_string`（CHAR） | `field_length / mbmaxlen` | **false** | 补 `pad_char` 到 `field_length`；丢尾部空格不算截断 |
| `Field_varstring`（VARCHAR） | `field_length / mbmaxlen` | **true** | 先写 1/2 字节长度前缀；丢尾部空格也报 NOTE |
| `Field_blob` | `length`（**不限制字符数**） | true | 先按 `mbmaxlen * length` 申请内存，只靠 `new_length` 字节上限截断 |

```cpp
/* Field_string::store —— CHAR 补空格 */
  copy_length = field_well_formed_copy_nchars(field_charset, (char *)ptr, field_length,
      cs, from, length, field_length / field_charset->mbmaxlen, ...);
  /* Append spaces if the string was shorter than the field. */
  if (copy_length < field_length)
    field_charset->cset->fill(field_charset, (char *)ptr + copy_length,
                              field_length - copy_length, field_charset->pad_char);
  return check_string_copy_error(..., from + length, false, cs);   /* ★ count_spaces=false */

/* Field_varstring::store —— VARCHAR 写长度前缀 */
  copy_length = field_well_formed_copy_nchars(field_charset, (char *)ptr + length_bytes,
      field_length, cs, from, length, field_length / field_charset->mbmaxlen, ...);
  if (length_bytes == 1) *ptr = (uchar)copy_length;
  else int2store(ptr, static_cast<uint16>(copy_length));   /* ★ 先写长度再回落盘数据 */
  return check_string_copy_error(..., from + length, true, cs);    /* ★ count_spaces=true */
```

> ⚠️ **`Field_blob::store()` 只做分派**（`store_to_mem` vs `store_internal`），well_formed 的调用点在 **`Field_blob::store_internal()`**。
> ⚠️ **ENUM / SET 完全不走 well_formed**——它们只用 `String::copy()` 做"能转就转"，`dummy_errors` 直接丢弃：
> ```cpp
>   if (String::needs_conversion_on_storage(length, cs, field_charset)) {
>     uint dummy_errors;
>     tmpstr.copy(from, length, cs, field_charset, &dummy_errors);   /* ★ 错误计数丢弃 */
>     from = tmpstr.ptr(); length = tmpstr.length();
>   }
> ```

**★ `my_convert()` 的 ASCII 快路径**（★ 与 `well_formed_copy_nchars` 是**两条不同的转换链**）：

```cpp
size_t my_convert(char *to, size_t to_length, const CHARSET_INFO *to_cs,
                  const char *from, size_t from_length,
                  const CHARSET_INFO *from_cs, uint *errors) {
  /* If any of the character sets is not ASCII compatible,
     immediately switch to slow mb_wc->wc_mb method. */
  if ((to_cs->state | from_cs->state) & MY_CS_NONASCII)
    return my_convert_internal(...);      /* ★★ 任一非 ASCII 兼容 → 一个字节都不快拷 */

  length = length2 = std::min(to_length, from_length);
#if defined(__i386__) || defined(_WIN32) || defined(__x86_64__)
  for (; length >= 4; length -= 4, from += 4, to += 4) {
    if (uint4korr(from) & 0x80808080) break;      /* ★ 4 字节一组：任一高位为 1 就退出 */
    int4store(to, uint4korr(from));
  }
#endif
  for (;; *to++ = *from++, length--) {
    if (!length) { *errors = 0; return length2; }
    if ((static_cast<uchar>(*from)) > 0x7F) {     /* 首个非 ASCII 字节处"切一刀" */
      size_t copied_length = length2 - length;
      to_length -= copied_length; from_length -= copied_length;
      return copied_length + my_convert_internal(to, to_length, to_cs, from,
                                                 from_length, from_cs, errors);
    }
  }
}
```

> ★★ **`copy_and_convert()` 只是 `my_convert()` 的一行 inline 壳**（`include/sql_string.h:140`），没有独立实现。
> ★ `my_convert_internal()` 的三种错误处理：`MY_CS_ILSEQ` → 吞 1 字节换 `'?'`；`cnvres > MY_CS_TOOSMALL`（合法但无 Unicode 映射）→ 吞整字符换 `'?'`；`MY_CS_ILUNI` → **降级成 `'?'` 再 `goto outp` 重试一次**，仍失败才 break。

#### check_string_copy_error：把位置翻译成 SQL 状态

```c
if (!(pos = well_formed_error_pos) && !(pos = cannot_convert_error_pos))
  return report_if_important_data(from_end_pos, end, count_spaces);  // ① 都为空 → 按截断处理

convert_to_printable(tmp, sizeof(tmp), pos, (end - pos), cs, 6);     // 只打 6 字节
push_warning_printf(thd, ..., ER_TRUNCATED_WRONG_VALUE_FOR_FIELD, "string", tmp, field_name, ...);
if (well_formed_error_pos != nullptr) return TYPE_WARN_INVALID_STRING;  // ② 非法串
return TYPE_WARN_TRUNCATED;                                             // ③ 不可转换
```

三分法：**非法位置 → `TYPE_WARN_INVALID_STRING`**；**不可转换位置 → `TYPE_WARN_TRUNCATED`**；**都为空 → 按 `from_end_pos` 判是否截断**。截断路径再走 `report_if_important_data`：

- `test_if_important_data` 先跳过尾随空格再判——CHAR 语义下尾随空格不算"重要数据"
- strict 模式且非 `IGNORE` → `ER_DATA_TOO_LONG`；否则 `WARN_DATA_TRUNCATED`
- `count_spaces` 由调用方决定：CHAR 传 **false**（尾随空格不重要），VARCHAR/BLOB 传 **true**

##### `convert_to_printable`：把坏字节变成可读的 `\xXX`

上面报错消息里只截 6 字节——那 6 字节就是它转换出来的：

```cpp
size_t convert_to_printable(char *to, size_t to_len, const char *from,
                            size_t from_len, const CHARSET_INFO *from_cs,
                            size_t nbytes) {
  assert(to_len >= 8);                       // 至少容纳 "\xXX..." + '\0'
  char *t = to, *t_end = to + to_len - 1;    // -1 留给 '\0'
  const char *f = from, *f_end = from + (nbytes ? min(from_len, nbytes) : from_len);
  char *dots = to;                           // 最后一个能塞 "..." 的落点
  for (; t < t_end && f < f_end; f++) {
    if ((unsigned char)*f >= 0x20 && (unsigned char)*f <= 0x7F &&
        from_cs->mbminlen == 1) {            // ASCII 可打印 且 源是 ASCII 兼容
      *t++ = *f;
    } else {
      if (t_end - t < 4) break;              // 放不下 "\xXX" 就停
      *t++ = '\\'; *t++ = 'x';
      *t++ = _dig_vec_upper[((unsigned char)*f) >> 4];
      *t++ = _dig_vec_upper[((unsigned char)*f) & 0x0F];
    }
    if (t_end - t >= 3) dots = t;
  }
  if (f < from + from_len) memcpy(dots, STRING_WITH_LEN("...\0"));  // 截断 → 追加 "..."
  else *t = '\0';
  return t - to;
}
```

四个设计点：① **转 `\xXX` 的条件包含 `from_cs->mbminlen != 1`**——对 UCS2 这类字符集，即使字节落在可打印区间也一律转义（逐字节打印会破坏字符边界）；② **`"..."` 的位置**是"最后一次还能放 3 字节的地方"（`dots`），只有"输入没读完"（`f < from + from_len`）时才追加——用于提示"错误内容在中间被截断"；③ **`assert(to_len >= 8)`** 的来源：`"\xXX..."` 5 字节 + `'\0'`；④ 它是**错误消息专用**工具（调用点：`Field_longstr::check_string_copy_error`、`Query_result_send::send_data`、`thd`/`Item_func` 的错误构造）。**`convert_to_printable_ascii` 在 8.0.39 不存在**（只有它与 `bin_to_hex_str`）。

#### 两条落盘入口：string/varstring 走 well_formed，blob/enum/set 走 String::copy

上一节讲了"落盘守门人"，但**不是所有 Field 都走它**——实际有两条入口，判据是列的存储形态：

| 列类型 | 入口 | 调用链 |
|---|---|---|
| `Field_string`（CHAR）/ `Field_varstring`（VARCHAR） | **`well_formed_copy_nchars`** | `store` → `field_well_formed_copy_nchars`（薄包装）→ `well_formed_copy_nchars` |
| `Field_blob` / `Field_enum` / `Field_set` | **`needs_conversion_on_storage` + `String::copy`** | `store` → 不转换则直接存指针+长度；转换则 `String::copy` → `my_convert` |

```cpp
// Field_string::store（sql/field.cc）——无条件走 well_formed 路径
copy_length = field_well_formed_copy_nchars(field_charset, (char *)ptr, field_length,
                                            cs, from, length,
                                            field_length / field_charset->mbmaxlen,
                                            &well_formed_error_pos, ...);
if (copy_length < field_length)   // 不足部分用 pad_char 填充
  field_charset->cset->fill(field_charset, (char *)ptr + copy_length,
                            field_length - copy_length, field_charset->pad_char);
return check_string_copy_error(well_formed_error_pos, cannot_convert_error_pos,
                               from_end_pos, from + length, false, cs);

// Field_blob::store —— 另一条路
if (!String::needs_conversion_on_storage(length, cs, field_charset)) {
  store_ptr_and_length(...); return TYPE_OK;      // 不转换：只存指针 + 长度
}
tmpstr.copy(from, length, cs, field_charset, &dummy_errors);   // 转换：走 String::copy
```

注意 `Field_string::store` 里两个细节：**① `nchars = field_length / mbmaxlen`**（按字符数而非字节数限制，正是 `well_formed_copy_nchars` 的 `nchars` 参数）；**② 不足部分用 `cset->fill` + `pad_char` 填充**——CHAR 的"补空格"在这里发生（而不是在 InnoDB 侧）。

#### `Copy_field` 只做分派，不做转换

一个常见误解是"`Copy_field` 负责字符集转换"。实际它**只选择函数指针**，转换发生在被调函数内部的 `Field::store`：

```cpp
void Copy_field::set(Field *to, Field *from) {
  if (to->type() == MYSQL_TYPE_NULL) { m_do_copy = do_skip; return; }
  m_from_field = from; m_to_field = to;
  m_do_copy2 = get_copy_func();          // 按类型组合选转换函数
  /* 再按 null 语义包一层：do_copy_null / do_copy_not_null / do_copy_maybe_null … */
}

// get_copy_func() 里与字符集相关的两个关键分支：
//   目标是 BLOB 且 charset 不同（或 JSON 与非 JSON 混） → do_conv_blob
//   普通字符串且 m_to_field->charset() != m_from_field->charset() → do_field_string
```

```cpp
// do_field_string —— "转换"就这两行
void do_field_string(Copy_field *copy, const Field *from_field, Field *to_field) {
  String res;
  from_field->val_str(&res);                                     // res 带「源列」字符集
  to_field->store(res.ptr(), res.length(), res.charset());       // 转换在 store 内发生
}
```

**零转换的快路径**：`fields_are_memcpyable()` 要求 `to->charset() == from->charset()`（以及类型/pack_length 等条件）成立才允许纯 memcpy——即"同字符集同长度"的列间拷贝完全不碰字符集代码。

⚠️ **更正**：`Copy_field::fix_length_and_charset` **在 8.0.39 不存在**（全仓 0 命中）——`Copy_field` 的公开接口只有 `set` / `invoke_do_copy` / `invoke_do_copy2` / `from_field` / `to_field`。

### 性能维度：字符集的成本结构

字符集对性能的影响不是常量，而是三个环节各自成本的叠加：**比较**、**转换**、**存储**。三个环节都在热路径上。

#### 环节一：比较与排序（最大头）

每次字符串比较的成本链 = `mb_wc` 解码 + UCA 查页 + 多级权重比较 +（InnoDB 侧）跨层回调 `strnncollsp`。其中**最直接的成本开关是 collation 比几级**：

| collation | 比较级数（`levels_for_compare`） | 相对成本 |
|---|---|---|
| `utf8mb4_bin` / latin1 系 | 0（`memcmp` / 单字节 `sort_order` 查表） | 最低 |
| `utf8mb4_0900_ai_ci` | 1 级（只比 primary） | 低 |
| `utf8mb4_0900_as_ci` | 2 级 | 中 |
| `utf8mb4_0900_as_cs` | 3 级 | 高 |
| `utf8mb4_unicode_ci`（UCA 4.0） | 1 级但权重串更长 | 中 |

级数的成本是**编译期固化**的（模板实例化），`ai_ci` 根本不生成 secondary/tertiary 的比较代码。两个叠加项：**NO PAD 的 `lengthsp`**（比较前剥尾空格，多一趟扫描）与 **InnoDB 侧无排序键缓存**（每次索引比较实时调 `strnncollsp`，latin1 硬编码特例除外）。

#### 环节二：排序键膨胀（filesort / GROUP BY）

排序不是比原串，而是**先生成权重键再 memcmp**。膨胀倍数见 `my_strnxfrmlen_uca_900` 的估算公式（详见排序处理器节）：

```
max_num_weights = 字符数 × 8 个 CE/字符 × levels_for_compare
排序键字节数   = (max_num_weights + 级分隔符数) × 2
```

即 `utf8mb4_0900_as_cs` 的排序键最长可到 `字符数 × 8 × 3 × 2 = 48×`；`utf8mb4_unicode_ci` 的 `strxfrm_multiply = 8`。filesort 用这些膨胀后的键做 memcmp——**内存占用与比较字节数都成倍涨**。纯 ASCII 短值还好，长文本排序列是重灾区。

#### 环节三：转换（连接字符集 ≠ 存储字符集）

`my_convert` 是"ASCII 快路径 + 逐码点慢路径"：纯 ASCII 4 字节一批检查（`0x80808080` 掩码），接近 memcpy；**出现非 ASCII 字符则每个字符走 `mb_wc` → `wc_mb` 两趟**。连接字符集与表不一致时，每行每列都付这个成本（内核月报 2017 亦点明"client/server 字符集不一致导致 CPU 升高"）。

#### 环节四：存储与键长（间接影响）

- **键长上限被 `mbmaxlen` 吃掉**：3072 字节是字节数，`utf8mb4` 的索引前缀最多 768 字符，latin1 能到 3072 字符
- **CHAR 空间**：`CHAR(100)` 在 utf8mb4 下最长 400 字节（COMPACT 剥尾空格能省，REDUNDANT 是实打实定长）
- **尾空格剥离/回填**：多字节 CHAR 写时剥、读时补，各多一趟

#### 量化手段

1. `HEX(WEIGHT_STRING(col))` —— 直接看排序键长度，膨胀一目了然
2. `levels_for_compare` / `strxfrm_multiply` —— 查 collation 的静态成本参数
3. 压测对比：纯 ORDER BY / GROUP BY 负载下对比 `0900_ai_ci` vs `0900_as_cs` vs `latin1`，差异可观

#### 表大小估算：从表定义与行数到容量

前面所有字节级规则的直接应用就是**估算表大小**。先明确两件事：

1. **估算的对象是 `rec_t`**——即聚簇索引叶子页里**一条物理记录**的字节数（不是 server 的 `TABLE::record`，也不是 `dtuple_t`）。算清一条 `rec_t` 多大，就能推出页数与总容量。
2. **`CHAR(n)` 的 `len` 是 `n × mbmaxlen`**（不是 `mbminlen`）——因为要能容纳最坏情况（每个字符都取最长编码）。源码里的证据是 `dtype_get_min_size_low` 的 `ut_a(len % mbmaxlen == 0)`，以及最小尺寸 `len × mbminlen / mbmaxlen`（全用最短字符时的下限 = `n × mbminlen`）。

##### 那不足的部分补什么？——补"空格字符"，按 `mbminlen` 为单位

`row_mysql_pad_col(mbminlen, pad, len)` 给出答案：填的是**空格字符 U+0020**，但**填充单位是"一个最短字符"（`mbminlen` 字节），不是单个 0x20 字节**：

```c
void row_mysql_pad_col(ulint mbminlen, byte *pad, ulint len) {
  switch (UNIV_EXPECT(mbminlen, 1)) {
    case 1:                       /* space = 0x20 */
      memset(pad, 0x20, len);
      break;
    case 2:                       /* space = 0x0020（大端） */
      pad_end = pad + len;
      ut_a(!(len % 2));
      while (pad < pad_end) { *pad++ = 0x00; *pad++ = 0x20; }
      break;
    case 4:                       /* space = 0x00000020（大端） */
      pad_end = pad + len;
      ut_a(!(len % 4));
      while (pad < pad_end) { *pad++ = 0x00; *pad++ = 0x00; *pad++ = 0x00; *pad++ = 0x20; }
      break;
  }
}
```

三个要点：

1. **填的是空格字符而非零字节**：`0x20` 是 ASCII/UCS 的空格。所以 `CHAR(10) utf8mb4` 存 `'abc'`（3 字节）时，补 **7 个空格字符**凑到 10 字节——读回来是 `'abc       '`（然后由 server 按 CHAR 语义剥掉）。
2. **按 `mbminlen` 为单位而不是 `mbmaxlen`**：补的是"最短字符"，因为**填充一个字符的最小代价就是 mbminlen 字节**。`ucs2`/`utf16`（mbminlen=2）补的是大端 `0x00 0x20` 两个字节一组；`utf32`（mbminlen=4）补 `0x00 0x00 0x00 0x20`；并且有 `ut_a(!(len % mbminlen))` 保证整除。
3. **没有 `case 3`**——因为 MySQL 里不存在"最短字符 3 字节"的字符集（字符集的最小字符宽度只可能是 1、2、4）。这个缺失本身就是字符集性质的一个侧面证据。

**什么时候真的会填？** COMPACT/DYNAMIC 下多字节 CHAR **先剥尾空格再存**（变长，填充不落盘），所以填充主要出现在：

- **REDUNDANT 格式**：CHAR 定长分配 `n × mbmaxlen`，不足部分由 `row_mysql_pad_col` 填空格
- **读回 MySQL 格式时**：`row_sel_field_store_in_mysql_format` 按 `templ->mbminlen` 把空格补回去，让 server 看到"完整宽度"的 CHAR 值
- **旧格式 → 新格式转换**（`data0data.cc`）时同样走一遍

这也解释了"为什么 COMPACT 比 REDUNDANT 省空间"在 CHAR 列上体现得最明显：同样的 `CHAR(255) utf8mb4`，REDUNDANT 每行列都要占 1020 字节，而 COMPACT 只存实际内容。

**一条 `rec_t` 的构成**：

| 组成 | 字节 |
|---|---|
| 各列数据 | 见下 |
| 记录头（COMPACT） | 5 |
| 变长字段长度列表 | 每个变长列 1 或 2 字节 |
| NULL 位图 | ⌈可空列数 / 8⌉ |
| `DB_TRX_ID` + `DB_ROLL_PTR` | 6 + 7（无主键时再加 `DB_ROW_ID` 6） |

**各列数据怎么算**（按内容、不按字符数）：

| 类型 | 字节 |
|---|---|
| 定长数值（`TINYINT`1 / `INT`4 / `BIGINT`8 / `DATETIME`5 或 8…） | 固定值 |
| `VARCHAR(n)` | 实际内容字节 + 长度前缀（≤255 字节用 1，否则 2） |
| `CHAR(n)`，**COMPACT/DYNAMIC** | 剥尾空格后按**实际内容**字节（变长） |
| `CHAR(n)`，**REDUNDANT** | 定长 **`n × mbmaxlen`**（不剥、按最坏分配） |
| `TEXT`/`BLOB` | 页内前缀 + 20 字节指针，其余页外另算 |

**字符 → 字节**：`utf8mb4` 的 ASCII 1 / 中文 3 / emoji 4；`gbk` 的 ASCII 1 / 中文 2；`latin1` 恒 1。

⚠️ **最常见的估算错误**：以为"utf8mb4 就是 4 倍"。`mbmaxlen` 只决定**声明上限与定长分配**，实际占用按内容算——`CHAR(10) utf8mb4` 存 `'abc'` 在 COMPACT 下只占 3 字节，而非 40。

**从 `rec_t` 到表大小**：

```
每页可用     ≈ 16KB × 15/16 ≈ 15232 字节     （默认留 1/16 给更新）
每页行数     = 15232 / 每条 rec_t 字节
叶子页数     = 总行数 / 每页行数
总大小       ≈ 叶子页数 × 16KB × 系数(1.1~1.3，非叶节点 + 碎片)
```

**二级索引单独算**：每个二级索引是一棵独立 B+树，其记录 = 索引列字节 + 主键字节 + 记录头 + 系统列。同时注意 **3072 字节键长上限**——`utf8mb4` 的索引前缀最多覆盖 768 个字符，`VARCHAR(2000) utf8mb4` 建全列索引会直接失败（`ER_INDEX_COLUMN_TOO_LONG`）。

**算一个实例**：

```sql
CREATE TABLE t (
  id BIGINT PRIMARY KEY,
  name VARCHAR(64) CHARACTER SET utf8mb4,   -- 平均存 10 个汉字 = 30 字节
  status TINYINT
) ENGINE=InnoDB;
```

| 组成 | 字节 |
|---|---|
| id | 8 |
| name 长度前缀 | 1（30 < 255） |
| name 内容 | 30 |
| status | 1 |
| 记录头 | 5 |
| NULL 位图 | 1 |
| DB_TRX_ID + DB_ROLL_PTR | 13 |
| **rec_t 合计** | **59** |

```
每页行数 = 15232 / 59 ≈ 258
100 万行 → 3876 页 → ≈ 60.6 MB（未计非叶与碎片）
乘 1.2 → ≈ 73 MB
```

同样的表若 `name` 是 latin1 且只存 ASCII：内容从 30 降到 10 字节 → `rec_t` 39 字节 → 表约 **48 MB**。字符集选型对容量的直接影响由此可见。

**必须用实测校准**（估算只给数量级，真实值受页填充率、碎片、页外 BLOB、前缀压缩、instant ADD COLUMN 影响）：

```sql
SELECT AVG_ROW_LENGTH, DATA_LENGTH, INDEX_LENGTH, TABLE_ROWS
FROM information_schema.TABLES WHERE TABLE_NAME = 't';
```

`AVG_ROW_LENGTH` 就是真实的"每条 rec_t 平均字节"——**把它代回公式比任何理论估算都准**。

#### 选型建议（按性价比排序）

1. **按需降级**：不需要重音区分就不用 `as_*`——`0900_ai_ci`（1 级）是多数业务的默认；纯标识符（UUID、code）用 `ascii`/`binary` 最省
2. **保持三层一致**：`character_set_client = connection = 表字符集`，消灭转换
3. **索引前缀列用窄字符集**：能 ascii 就不用 utf8mb4——键长上限与每字节比较成本都省
4. **慎用 `unicode_ci` 做高频排序列**：它的排序键膨胀最大，要精确排序才付这个价

---

## ★ 本机制里的工程实现技法

### 一、C 函数指针表：结构化的"穷人版多态"

一个 collation 的完整行为 = **两组函数指针 + 一组数据表**，全部挂在 `CHARSET_INFO` 上：

```c
struct CHARSET_INFO {
  ...
  MY_CHARSET_HANDLER *cset;   // 字符学：mb_wc / wc_mb / ctype / caseup / well_formed_len ...
  MY_COLLATION_HANDLER *coll; // 比较学：strnncoll / strnncollsp / strnxfrm / like_range / wildcmp / hash_sort
};
```

**为什么不用 C++ 虚函数**：`CHARSET_INFO` 是 **C ABI 结构**——libmysql（C 客户端库）、mysys、各存储引擎都要用它，动态加载的字符集也靠填充这个结构接入；换成 C++ 类会破坏 ABI 与 C 侧可用性。代价是没有编译期的接口一致性检查（函数签名靠 discipline），也没有继承复用——于是用"**共享 handler 实例 + 数据表差异**"来替代继承：

```c
// 绝大多数 8bit 字符集共用同一个 handler 实例，各自的差异在数据表里
&my_charset_8bit_handler,
&my_collation_8bit_simple_ci_handler,
// UCA 900 系列也共用一个 handler，差异在 cs->uca 权重表与 tailoring 规则
&my_charset_utf8mb4_handler,
&my_collation_uca_900_handler,
```

| 技法 | 用在哪 | 收益 | 代价 / 反直觉处 |
|---|---|---|---|
| 函数指针表 | `MY_CHARSET_HANDLER` / `MY_COLLATION_HANDLER` | 运行时可选、C ABI 兼容、动态加载可用 | 无编译期接口检查 |
| 共享 handler 实例 | `my_charset_8bit_handler` / `my_collation_uca_900_handler` | 加字符集只需换数据表 | 覆盖错了不报错 |
| 数据与行为分离 | `sort_order` / `ctype` / `uca` 权重表 | 加 collation 基本零代码 | 权重表编译进二进制（数 MB） |

### 二、数据驱动的 tailoring：拷贝权重页而非原地改

语言定制（`zh` 拼音序、`de` 电话簿序…）的实现是 `create_tailoring`：解析 `cs->tailoring` 规则串（ICU/CLDR 的 LDML 语法，如 `& B < C`），把规则**应用到权重表的副本**上：

```
create_tailoring(cs, loader)
  ├─ my_coll_rule_parse(&rules, cs->tailoring, ...)   // 解析 LDML 规则
  ├─ 按 UCA 版本选源权重数据（my_uca_v400 / my_uca_v900）
  ├─ init_weight_level / my_uca_copy_page             // ★ 逐页拷贝：256 * lengths[page] * 2 字节
  ├─ apply_one_rule(...)                              // 逐条规则改写拷出来的页
  └─ add_contraction_to_trie(...)                     // 建 contraction trie
```

**关键设计：必须拷贝权重页，不能原地改**。因为多个 collation 共享同一份基础权重表（`my_uca_v900`），若原地改写，`utf8mb4_0900_ai_ci` 的定制会污染 `utf8mb4_0900_as_cs`。所以每个带 tailoring 的 collation 在 init 时各拷一份私有副本——这也是前面说"懒加载很贵"的原因（要拷 256 页）。

| 维度 | ICU 原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 依赖 | 链接 ICU 库 | 自研解析器（只借规则语言） | 不引外部依赖、跨版本行为稳定；规则语言只支持子集 |
| 权重数据 | ICU 运行时加载 | 编译期头文件 + init 时拷贝 | 加载零 IO；代价是二进制大、init 慢 |
| 定制时机 | 运行时 | 首次使用时（`coll->init`，`THR_LOCK_charset` 保护） | 惰性；多线程首次并发需加锁，`READY` 位保证幂等 |

### 三、模板实例化做编译期分级派发

`levels_for_compare` 是运行时字段（存在 `CHARSET_INFO` 里），但算法把**级别数提升为编译期常量**——用模板实例化 + 一次 switch 分派：

```c
// 运行期只做一次分派，之后全是编译期特化代码
switch (cs->levels_for_compare) {
  case 1: cmp = my_strnncoll_uca_900<Mb_wc, 1>(...); break;   // ai_ci
  case 2: cmp = my_strnncoll_uca_900<Mb_wc, 2>(...); break;   // as_ci
  case 3: cmp = my_strnncoll_uca_900<Mb_wc, 3>(...); break;   // as_cs
  case 4: cmp = my_strnncoll_uca_900<Mb_wc, 4>(...); break;   // ja_ks
}
```

```c
template <class Mb_wc, int LEVELS_FOR_COMPARE>
class uca_scanner_900 : public my_uca_scanner { ... };
```

**两个模板参数各自的意义**：

- `LEVELS_FOR_COMPARE`：把"要比较几级"变成编译期常量，于是扫描器内部的级数循环可以被完全展开/优化掉，且"ai_ci 只看 primary"不是运行时判断而是**根本不生成那部分代码**
- `Mb_wc`：**把字符集解码函数作为模板参数传进去**，让 `mb_wc` 调用被内联进扫描循环——避免热路径上每字符一次函数指针调用（UTF-8/UTF-16/UTF-32/utf8mb3 各自实例化一份）

代价是代码膨胀（级别数 × 字符集 的组合各一份机器码），换来的是比较热路径上没有运行时分支与间接调用。

### 四、热路径的位图预筛 + 批量 ASCII 快路径

字符集代码里反复出现同一手法：**先花 O(1) 判断"能不能走快路径"**。

**① contraction 位图**：contraction（多字符组合排序）需要下钻 trie，代价高。于是先用位图排除绝大多数字符：

```c
// MY_UCA_CNT_HEAD / TAIL / MID 等标志位；只有置位的字符才可能参与 contraction
if (my_uca_can_be_contraction_tail(m_wc2)) { ... previous_context_find(...) ... }
const MY_CONTRACTION *c = contraction_find(wc, &chars_skipped);
```

一次位测试替代一次 trie 下钻——因为绝大多数字符（ASCII、汉字）根本不参与 contraction。

**② `my_convert` 的 4 字节批量 ASCII 检查**：转换时先按机器字长批量判断"这段是不是纯 ASCII"：

```c
// 一次检查 4 个字节的高位（0x80808080 掩码），全 0 说明都是 ASCII，可整块复制
while (...) { if ((four_bytes & 0x80808080) == 0) { /* 批量复制 */ } else { /* 转逐字符路径 */ } }
```

真实数据多为 ASCII，于是转换退化到接近 `memcpy` 的速度。

**③ `for_each_weight` 的 ASCII 快路径**：UCA 扫描器里同样对 0x20..0x7E 范围做批量检查，直接取权重而跳过 contraction/implicit 等分支。

三处的共同思想：**慢路径的判别成本必须 O(1) 且极低，快路径要能覆盖绝大多数实际数据**。这是字符串处理这类"每字符都跑一遍"的代码唯一可行的优化方向。

共同思想：**先花 O(1) 判断"要不要走慢路径"，绝大多数情况命中快路径**。

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `character_set_server` | utf8mb4 | G/S | 服务端默认（建库未指定时） |
| `character_set_client/connection/results` | utf8mb4 | G/S | 协议三态 |
| `character_set_database` | utf8mb4 | G/S | 当前库默认（USE 时联动） |
| `character_set_filesystem` | binary | G/S | 文件名编码 |
| `collation_server` | utf8mb4_0900_ai_ci | G/S | 服务端默认 collation |
| `default_collation_for_utf8mb4` | utf8mb4_0900_ai_ci | G/S | utf8mb4 缺省 collation 的兜底 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 某 collation 的 pad 属性 | SQL | `information_schema.COLLATIONS` 的 `PAD_ATTRIBUTE` 列 |
| 所有可用 charset/collation | SQL | `SHOW CHARACTER SET` / `SHOW COLLATION` |
| 列实际 collation | SQL | `information_schema.COLUMNS` 的 `COLLATION_NAME` |
| 表达式推导出的 collation | SQL | `EXPLAIN FORMAT=JSON` 的 `attached_condition` / 报错信息 |
| 字符串比较的排序键 | SQL | `HEX(WEIGHT_STRING(s))`（直接看权重字节） |
| 转换错误 | SQL | `SHOW WARNINGS`（'?' 替换的告警） |

`WEIGHT_STRING('a')` 是对 strnxfrm 的直接观测：`HEX(WEIGHT_STRING('A' COLLATE utf8mb4_0900_as_cs))` 能看到 tertiary 级里大小写的权重差。

---

## Misc

### 术语：MySQL 字符串命名的缩写与函数名解码

MySQL 的字符集代码满是缩写，函数名像 `my_strnncollsp_uca_900` 这种"一长串"其实是可解码的。先记缩写：

| 缩写 | 完整 | 含义 |
|---|---|---|
| `mb` | multi-byte | 多字节（本字符集编码下的字节序列），如 `mb_wc`、`mbcharlen` |
| `wc` | wide character | 宽字符 = **Unicode 码点**，如 `wc_mb` |
| `cs` / `cset` | character set | 字符集（/`cset` 特指字符集函数表） |
| `coll` | collation | 排序规则（比较） |
| `ctype` | character type | 字符类别判定 |
| `uni` | unicode | 与 Unicode 相关的数据表 |
| `xfrm` | transform | 变换——特指**生成排序键**（`strnxfrm`） |
| `tab` | table | 查找表（`tab_to_uni`/`tab_from_uni`） |
| `nn` | n, n | **两串都按长度处理**（非 `\0` 终止） |
| `sp` | space | 处理**尾部空格**语义（`strnncollsp`） |

于是函数名可以逐段解码：

```
my_strnncollsp_uca_900
│   │   │   │  │   └── 权重版本：UCA 9.0.0
│   │   │   │  └────── 算法族：UCA 权重（对比 _simple / _mb / _binary）
│   │   │   └───────── sp：按尾部空格语义比较（PAD SPACE / NO PAD）
│   │   └───────────── coll：比较
│   └───────────────── nn：两个串都按长度而非 \0 终止
└───────────────────── my：MySQL 内部前缀
```

同理 `strnxfrm` = "按长度处理的串 → 排序键变换"，`mb_wc` = "多字节字节序列 → Unicode 码点"。**记住这张表，strings/ 下几百个函数名的含义就都能读出来了。**

### utf8 / utf8mb3 / utf8mb4 三兄弟

- `utf8` 在 8.0 里是 `utf8mb3` 的**别名**（`get_charset_name_alias` 互换），历史上"utf8"只支持 3 字节，放不进 emoji（4 字节）——这就是 utf8mb4 存在的原因
- utf8mb4 在 8.0.39 编译了 **89 个 collation**：`utf8mb4_0900_ai_ci`(255) + 28 个 `*_0900_ai_ci` 语言变体、`utf8mb4_0900_as_cs` + 20 个 as_cs 语言 + `ja_0900_as_cs`/`ja_0900_as_cs_ks`、`utf8mb4_0900_as_ci`(305) + 若干语言、`utf8mb4_0900_bin`(309)；老 `utf8mb4_general_ci`(45)/`utf8mb4_bin`(46)/`utf8mb4_unicode_ci`(224) + 23 个 UCA 4.0 语言变体
- 注册顺序有讲究：`utf8mb4_bin` 在 `utf8mb4_0900_bin` **之后**注册——为了让 deprecated 的 `BINARY` 属性仍映射到 `utf8mb4_bin`（覆盖 `cs_name_bin_num_map`）

**其他 Unicode 字符集家族**（同一套码点、不同编码方式）：

| 字符集 | mbminlen/mbmaxlen | 说明 |
|---|---|---|
| utf8mb3 | 1/3 | 仅 BMP，deprecated |
| utf8mb4 | 1/4 | 全 Unicode |
| ucs2 | 2/2 | 仅 BMP，定长 2 字节，**无代理对** |
| utf16 | 2/4 | 支持代理对（变长） |
| utf16le | 2/4 | UTF-16 小端 |
| utf32 | 4/4 | 定长 4 字节 |

三点值得记住：①**没有 BOM 处理**——`my_utf16_uni`/`my_utf32_uni` 都是裸解码，不跳过也不生成 BOM；②`well_formed_len` 对定长家族要求**按码点对齐**（utf32 长度非 4 倍数直接判非法）；③ debug 构建下 `my_strnncollsp_utf32` 仍有 `assert((slen % 4) == 0)`——"调用方必须按码点边界传长度"的契约断言（同族 ucs2 版已改成容错写法 `slen &= ~1`）。遇到 utf32/utf16 相关断言崩溃，通常意味着上游传了未对齐的长度。

### 坑

1. **`strnncoll` 与 `strnncollsp` 语义不同但常被混用**：PAD SPACE collation 下索引比较（sp）和 `WHERE` 表达式（nn）对尾空格的判断可能不一致——SQL 标准规定字符串比较应 PAD，但表达式 `=` 走 `strnncoll` 时 NO PAD。这正是开场例子里 0900 系列结果反直觉的根因。
2. **换 collation 改变比较结果但不触发重建**：`ALTER TABLE ... CONVERT TO CHARACTER SET` 才重写数据；`ALTER ... COLLATE`（8.0 只改元数据）后已有索引的排序键还是旧的——老版本里这是著名的"索引排序错乱"坑。
3. **contraction 语言的 LIKE 范围**：`like_range` 的最大键必须考虑 contraction 展开（'ch' 场景），实现是显式特殊处理，语言变体多时容易漏。
4. **`_general_ci` 是"猜的"**：它按简化的 primary 权重比，多语言场景下与 UCA 标准序有出入——新项目一律用 `_0900_ai_ci` 或 `_unicode_ci`。
5. **转换的 '?' 替换是静默的**：`my_convert` 的错误以警告形式返回，`SET sql_mode` 不含严格转换开关时数据损坏无声无息。
6. **UCA 4.0 下非 BMP 字符全等**：`uca_scanner_any` 对 `wc > maxchar`（U+FFFF）统一返回 0xFFFD——utf8mb4_unicode_ci/general_ci 下所有 emoji/CJK 扩展字符彼此相等，排序相邻且 `=` 恒真。升级 0900 系列才有可区分的顺序。
7. **`=` 与 `LIKE` 在同一 collation 下可能不同真值**：`=` 走 Sort Key（一对多相等），LIKE 走逐字符（一对一）——unicode_ci 下 `'ß' = 'ss'` 为真但 `'ß' LIKE 'ss'` 为假。
8. **大小写转换的字节膨胀**（Bug#119463）：`caseup_multiply == 1` 假设大小写不改变字节数，但 Ⱦ(U+023E,2B)↔ⱦ(U+2C66,3B) 这类跨编码边界的映射对会让 inplace 转换 src/dst 错位、截断字符串；UCA 520/900 系列都中招，general_ci 因数据不全反而"免疫"。
9. **binary 源是"不转换"不是"不校验"**：源字符集为 binary 时逐字节复制、不做转码，但后续仍按目标字符集做合法性检测——utf8mb4 字节塞进 gbk 列时，"奇数个汉字报错、偶数个成功"是字节数凑巧整除的假象（见转换链的 mysqldump 案例）。注意"落库路径反而更严"：`needs_conversion_on_storage` 专门对 bin→非bin 强制转换以校验良构性。
10. **`BINARY` 一词有四种含义**（最容易搞错的是第二种）：① `binary` 字符集（id 63）② **`BINARY` 列属性**——8.0 把它解析成"该字符集的 `_bin` collation"（`varchar(10) BINARY` ≡ `CHARACTER SET utf8mb4 COLLATE utf8mb4_bin`）**并给 deprecated 告警**，绝不是 binary 字符集 ③ `BINARY`/`VARBINARY`/`BLOB` 数据类型 → 走 binary 字符集 ④ `CAST(x AS BINARY)`/`BINARY x` → 只重贴 charset 标签、不改字节（等价 `CONVERT(x USING binary)`）。
11. **`needs_conversion` 对 binary 源的"对齐短路"**：源是 binary 且 `arg_length % to_cs->mbminlen == 0` 就**不转换**——对 gbk（mbminlen=1）而言该条件**恒成立**，于是 `binary → gbk` **永不走转换**。这条捷径是 mysqldump bug 成立的机制之一；存储路径用 `needs_conversion_on_storage` 才能补上。
12. **结果集元数据名字的字符集也受 `character_set_results` 影响**：名字以 `system_charset_info`（utf8mb3）为**源**，但经 `store_string` 会转到 `character_set_results`——"元数据一律 utf8mb4、不受 results 影响"是错的。另有 `charsetnr` 字段的两种写法（有转换时发 results 的号，否则发列自身的号）。
13. **`String::length()` 是字节，`numchars()` 才是字符**：utf8mb4 下 `"中".length() == 3` 而 `numchars() == 1`；要字符级定位用 `charpos(i)`（返回字节位置）。另外 `String` 默认构造的 charset 是 `&my_charset_bin`——**未显式声明即被当作字节串**。
14. **`Field::binary()` 才是判定函数，没有 `Field::is_binary()`**：基类默认返回 `true`（数值类型也算），字符串字段用 `field_charset == &my_charset_bin` 覆盖。凡"按字节"的语义在 binary 下退化为恒等（`UPPER(bin_col)` 原样返回、尾空格在截断判定中算"重要数据"）。
15. **tailoring 的支持集远窄于 ICU 文档**：写自定义规则串时若用了 `[import]`/`[reorder]`/`[maxVariable]`/`[alternate]`/`[backwards]`/`[normalization]`/`[caseFirst]`/`[suppressContractions]`，会直接 `EE_COLLATION_PARSER_ERROR` 导致 collation 初始化失败（`READY` 位设不上，后续取用返回 nullptr）——这些指令的功能要么由 `Coll_param` C 结构体提供，要么（maxVariable/alternate）根本没有。
16. **`[before N]` 靠"基准权重 −1 + extra CE 递增"塞空隙，不重编号**：DUCET 权重区间是稠密的，tailor 变体被塞进基准字符前的编号空间（`extra_ce_*_base` = 0x54A4 段）。含义：① 同一基准字符下能区分的变体数受该编号空间限制；② ★ **`<<<<`（quaternary）的 `diff[3]` 不参与权重写入**——四级只在日文里由扫描器合成，"四级 shift"只影响解析结构。
17. **分区键的 collation 决定行落在哪个分区**：`PARTITION BY KEY` 用列自身 collation 的 `hash_sort`——**改 collation 会导致同一条记录换分区**，所以跨版本哈希算法不敢变（5.1 兼容分支硬编码 `my_charset_bin`/`my_charset_latin1`）。`PARTITION BY RANGE/LIST COLUMNS` 含字符串列时先 `my_strnxfrm` 归一化。
18. **`ALTER ... CONVERT TO` 改到分区列上会被拒绝**：`check_part_func_fields` 对需要字符集处理的 CHAR/VARCHAR 列（`mbmaxlen > 1 || strxfrm_multiply > 1`）直接不允许——"改字符集"与"分区函数"在语义上冲突，只能重建分区表。

### 易混淆

- **`character_set_connection` vs `character_set_client`**：client 是"进来的字节"，connection 是"解析后内部使用"（两者通常一起变），results 是"出去的字节"
- **collation 的 bin 与 binary charset**：`utf8mb4_bin`（比较按码点，但仍是 utf8mb4 家族）vs `binary`（charset=binary，比较纯字节 memcmp，含长度敏感）——`CHAR(10) BINARY` 的 pad 行为完全不同
- **`sort_order` 表 vs `strnxfrm`**：单字节老 collation 用 sort_order 表直接查表比较（不生成排序键）；`MY_CS_STRNXFRM` 置位的才走 strnxfrm——两者并存是历史分层
- **`levels_for_compare` 的 ai/as 与 `_ci` 的 ci**：老 `_ci` 的"case insensitive"靠权重表里大小写同权重实现；900 系列的 ai/as 靠"比几级"实现——机制完全不同
- **`[import]`/`[reorder]`/`[maxVariable]` 不是 MySQL 的 UCA 规则语言**：8.0 解析器只认 `&`/四级 shift/`=`/`/`/`|`/`[before N]`/逻辑位置/`[version]`/`[shift-after-method]`；ICU 文档里的 `[import]`/`[alternate]`/`[backwards]`/`[maxVariable]`/`[caseFirst]`/`[normalization]`/`[suppressContractions]` **全无解析分支**（出现即报错）。其中 `case_first`/`norm`/`reorder` 的功能由 **C 结构体 `Coll_param` 静态实现**，`maxVariable`/`alternate` **8.0 完全未实现**
- **REGEXP 的 `match_type` 里 `u` 不是 unicode**：是 **UNIX 行尾**（`UREGEX_UNIX_LINES`）；默认大小写敏感性**来自 collation** 的 `MY_CS_CSSORT | MY_CS_BINSORT`（`_ci` → 默认不敏感），`i`/`c` 只是显式覆盖——**不走 collation 的 `caseup` 表**
- **同一个 `_ci` 语义在不同语句里由不同函数实现**：ORDER BY 走 `strnxfrm` 生成权重键；GROUP BY 走 `hash_sort`（冲突时 `strnncollsp` 精比）；DISTINCT 走 `Unique` 的 `strnncollsp`——结果一致但代价与内存特征不同

### 扩展点

加一个新语言 collation（如 `utf8mb4_0900_xx_ai_ci`）要动：

1. 生成权重定制数据（脚本生成 `uca900_xx_data.h` 风格的权重页）
2. `ctype-uca.cc` 里加一个 `CHARSET_INFO my_charset_utf8mb4_0900_xx_ai_ci` 定义（挂 handler + tailoring 规则串）
3. `mysys/charset-def.cc` 的 `init_compiled_charsets` 注册它
4. `share/charsets/Index.xml` 加 `<collation>` 条目（供名字/编号注册与 I_S 元数据）
5. dd 初始化时 `mysql.collations` 表自动从 all_charsets 填充，无需手改

**★ 历史提醒（内核月报时代的路径已死）**：5.6/5.7 时代可以"改 Index.xml 加 `<rules><reset>...</reset><i>...</i></rules>` 重启即得新 collation"（月报 2017/03 的 utf8_phone_ci 实例），**8.0 这条路径整体删除**——`my_charset_rule.c` 文件与 `my_coll_rule_parse` 已不存在，Index.xml 里也没有任何 `<rules>` 元素；XML 解析器残留的 `rules/reset/i` 标签语义已变成 **UCA tailoring 文本拼接**（`reset` → `&`，`i` → 忽略级规则），只对 UCA 类 collation 生效。8.0 新增 collation 必须改源码（`share/charsets/` 定义 + `strings/conf_to_src.cc` 重新生成 `ctype-*.cc`）后重新编译。运行期仅保留按需加载 `<csname>.xml` 的机制。

**★ Index.xml 的 `cs_value` 里那些"拼接 tailoring 文本"的分支**（★ 这就是上面"残留标签语义"的具体落点）：

```cpp
    case _CS_RESET:        rc = tailoring_append(st, "%.*s", len, attr); break;
    case _CS_DIFF1 ... _CS_IDENTICAL:
                           rc = tailoring_append(st, diff_fmt[state - _CS_DIFF1], len, attr); break;
    case _CS_EXP_EXTEND:   rc = tailoring_append(st, " / %.*s", len, attr); break;   /* ★ expansion */
    case _CS_RESET_BEFORE: rc = tailoring_append(st, "[before %.*s]", len, attr); break;
    /* Logical Reset Positions */
    case _CS_RESET_FIRST_NON_IGNORABLE:  rc = tailoring_append(st, "[first non-ignorable]", 0, nullptr); break;
    case _CS_RESET_LAST_PRIMARY_IGNORABLE: ... "[last primary ignorable]" ...
    /* Collation Settings */
    case _CS_ST_STRENGTH:     rc = tailoring_append(st, "[strength %.*s]", len, attr); break;
    case _CS_ST_ALTERNATE:    rc = tailoring_append(st, "[alternate %.*s]", len, attr); break;
    case _CS_ST_BACKWARDS:    rc = tailoring_append(st, "[backwards %.*s]", len, attr); break;
    case _CS_ST_CASE_LEVEL:   rc = tailoring_append(st, "[caseLevel %.*s]", len, attr); break;
    case _CS_ST_CASE_FIRST:   rc = tailoring_append(st, "[caseFirst %.*s]", len, attr); break;
    case _CS_ST_NUMERIC:      rc = tailoring_append(st, "[numeric %.*s]", len, attr); break;
    /* Special purpose commands */
    case _CS_UCA_VERSION:     rc = tailoring_append(st, "[version %.*s]", len, attr); break;
    case _CS_CL_OPTIMIZE:     rc = tailoring_append(st, "[optimize %.*s]", len, attr); break;
    case _CS_CL_SUPPRESS_CONTRACTIONS: rc = tailoring_append(st, "[suppress contractions %.*s]", ...); break;
```

> ★★ 这些 `[...]` 形式正是 ICU/CLDR 的规则语法（对应"规则语言的语法"那节）。`</collation>` 闭合时 `cs_leave()` 调 `i->loader->add_collation(&i->cs)`——**这是 Index.xml / `<csname>.xml` 落库到 `all_charsets` 的唯一出口**。
> ★ `cs_value` 里还有一条硬约束：`_CS_CSNAME` 分支把 `"utf8"` 强制替换成 `"utf8mb3"`，并 `assert(0 != strcmp(i->cs.csname, "utf8"))`——**外部字符集不允许叫 utf8**。

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `include/mysql/strings/m_ctype.h` | **不存在** → 只有 `include/m_ctype.h` |
| `strings/ctype.c` | **不存在** → 真名 **`strings/ctype.cc`**（C++） |
| `strings/uctype.h` | **不存在** → 拆成 `include/my_uctype.h` + `strings/str_uca_type.h` |
| `strings/uca-dump.cc` | **不存在** → **`strings/uca9-dump.cc`** |
| `sql/sql_string.cc` | **不存在** → **`sql-common/sql_string.cc`**；且 `needs_conversion` / `copy_and_convert` 定义在头文件 `include/sql_string.h`（inline） |
| `MY_CS_CTYPE_TABLE` / `MY_CS_TOUPPER` / `MY_CS_TOSORT` / `MY_CS_CUSTOM` | **全部不存在**（只剩 `MY_CS_CTYPE_TABLE_SIZE=257` 这类**尺寸**宏） |
| `MY_CS_CONFIG` / `MY_CS_INDEX` | **不存在** → 改名 **`MY_CS_CONFIG_UNUSED`** / **`MY_CS_INDEX_UNUSED`**（"unused bitmask"） |
| `MY_CS_*` 位是 1,2,4,8,16… | **连续 `1 << 0 … 1 << 15`**，共 17 个（含 `MY_CHARSET_UNDEFINED=0`、`MY_CS_READY`、`MY_CS_CSSORT`） |
| `charsets_by_name` / `collations_by_name` / `charsets_by_csname` | **均不存在** → **`coll_name_num_map`** / **`cs_name_pri_num_map`** / **`cs_name_bin_num_map`**（`std::unordered_map<std::string,int>*`） |
| `my_charset_init()` | **不存在** → `init_compiled_charsets()`（`charset-def.cc:271`）+ static 的 `init_available_charsets()`（`charset.cc:423`） |
| `my_collation_utf8mb4_uca_handler` | **不存在** → **`my_collation_any_uca_handler`**（600 系列）+ **`my_collation_uca_900_handler`**（0900 系列） |
| `my_charset_get_number_by_name` | **不存在** → `get_collation_number()` / `get_charset_number(name, cs_flags)` |
| `my_charset_is_compiled` 宏 | **不存在** → 代码一律写 `(cs->state & MY_CS_COMPILED)`；只有 `my_binary_compare` / `use_strnxfrm` 两个判宏 |
| `my_strcasecmp()` 是函数 | 是**宏**（`m_ctype.h:763`）→ `cs->coll->strcasecmp` |
| `my_charset_same()` 是宏 | 是 **inline 函数**（`m_ctype.h:692`），且带 `assert` 禁止名字为 `"utf8"` |
| `my_caseup_str()` / `my_casedn_str()` 是函数 | 是**宏**（`m_ctype.h:814-815`）；真函数叫 `my_casedn_str_8bit` / `_utf8mb4` |
| coll 函数表里有 `instr` 成员 | 声明真名是 **`strstr`**；实现函数名才带 `instr` |
| `my_strnncollsp_bin()` | **不存在** → `my_strnncollsp_binary`（binary，NO PAD）+ `my_strnncollsp_8bit_bin`（`*_bin`，PAD SPACE） |
| `my_like_range_bin()` / `my_like_range_utf8mb4()` | **都不存在** → 二者都用 `my_like_range_simple()` / `my_like_range_mb()` |
| `my_wildcmp_simple()` | **不存在** → **`my_wildcmp_8bit()`** |
| `my_instr_utf8mb4()` | **不存在** → 复用 `my_instr_mb()` |
| `my_uca_get_weight()` | **不存在** → `my_char_weight_addr()` / `my_char_weight_addr_900()` |
| `my_uca_scanner_any()` / `my_uca_scanner_next()` | **不是函数** → 是**类**（基类 `my_uca_scanner` + 模板 `uca_scanner_any<Mb_wc>` / `uca_scanner_900<Mb_wc,LEVELS>`），推进方法是成员 `next()` |
| `MY_UCA_LEVELS` | **不存在** → 靠 `CHARSET_INFO::levels_for_compare` + 模板参数 |
| `struct MY_UCA_WEIGHT` / `weight[2]` / `level` | **不存在** → 契约节点用 `MY_CONTRACTION::weight[25]`；900 单字符权重是 `uint16` 页数组，级间靠 `UCA900_DISTANCE_BETWEEN_LEVELS=256` |
| `MY_TAILORING` | **不存在** → `CHARSET_INFO::tailoring`（`const char*`）+ 解析产物 `MY_COLL_RULES` / `MY_COLL_RULE` |
| `my_uca_load_open()` | **不存在** → `my_coll_init_uca()` → `create_tailoring()` |
| `my_uca_v900` 在 `ctype-uca.cc` | **不在** → 在 **`strings/uca900_data.h:270943`**（`my_uca_v400`/`v520` 才在 ctype-uca.cc） |
| `my_charpos_utf8mb4()` | **不存在** → **`my_charpos_mb4()`** |
| `my_ismbchar_utf8mb4()` 有位图预筛 | **没有** → 它只是三行薄封装；8 字节 ASCII 位图快路径**只存在于 `my_charpos_mb4()`**，全仓仅此一处 |
| `my_stpcpy_dn()` | **不存在**（0 命中）→ `my_stpcpy()` + `my_casedn_str(files_charset_info, …)` |
| `check_column_grant_for_charset()` | **不存在**（0 命中） |
| `my_charset_dir` | **不存在** → **`charsets_dir`**（`mysys/charset.cc:356`）/ 服务端 `mysql_charsets_dir` |
| `LC_ALL` 影响字符集加载 | **无关** —— 服务端 mysqld 内部**没有任何 `setlocale`**；仅 `client/mysql.cc:2733` 有一次 |
| `system_charset_info` 是 utf8mb4 | **是 `my_charset_utf8mb3_general_ci`（id 33，mbmaxlen=3）** |
| `Protocol_text::store_string()` | **不存在** → **`Protocol_classic::store_string()`** |
| `THD::charset()` 返回连接 collation | 返回 **`variables.character_set_client`** |
| 有独立的 `character_set_server` 变量 | **没有** → 由 `collation_server` 兼任 |
| `sql/field_common_properties.h` 与字符集有关 | **完全无关** → 只有 9 个 `enum_field_types` 判定 inline 函数 |
| `Field_blob::store()` 里做 well_formed | `store()` 只分派 → well_formed 在 **`store_internal()`** |
| `Copy_field::set()` 做字符集转换 | **只选函数指针**；转换在 `do_field_string` / `do_conv_blob` 里 |
| `well_formed_copy_nchars()` 只校验 | **校验 + 拷贝一体**（`memmove` 在函数体内） |
| `copy_and_convert()` 有独立实现 | **只有一行** → `return my_convert(...)` |
| 8.0 不再需要 boost 之外的"新 collation 可热加" | 5.6/5.7 的"改 Index.xml 加 rules"路径**整体删除**，必须改源码重编译 |

**★ 三个"结构性事实"**

1. **`ext::` 式的"两套函数表 + 一个 id 直接下标"是字符集体系能横向扩展的根本**——`CHARSET_INFO` 里只有 `cset` / `coll` 两个指针，加一个新字符集就是"填两张表 + 一个 id"；`all_charsets[cs->number] = cs` 让 id 与下标等同，**没有编码公式可冲突**。代价是 id 必须全局单调分配（Index.xml 里注释要求 "keep records sorted by collation number"）。

2. **`MY_CS_LOADED` / `AVAILABLE` / `CSSORT` / `PUREASCII` / `NONASCII` 这五个状态位是"推导"出来的，不是声明的**——`MY_CHARSET_LOADER::add_collation()` 是它们的**唯一自动推导点**：Unicode 家族走 `copy_uca_collation()`，其余 8bit 走 `simple_cs_init_functions()` 后用 `sort_order['A'] < sort_order['a'] < sort_order['B']` 三点比较推 `CSSORT`、用 `my_charset_is_8bit_pure_ascii()` 推 `PUREASCII`。且**编译型不可被 XML 覆盖**（命中 `MY_CS_COMPILED` 就 `clear_cs_info` + 直接 return）。

3. **"宽松转换"与"严格落盘"是两条方向相反的链**——`String::copy` / `my_convert` / `Protocol_classic::store_string` 是"尽量转、转不动换 `'?'`、不回报位置"（服务读路径与协议输出）；`well_formed_copy_nchars` / `check_string_copy_error` 是"不替换、遇到第一个问题就停并回报位置"（服务写路径）。同一个坏字节在 `SELECT` 里变成 `?`、在 `INSERT` 里报错，**根源就是这两条链语义相反**。协议侧还有个 250 字节阈值：超过就先转进临时缓冲，因为长度前缀的字节数取决于转换后的长度。

> 注：`include/m_ctype.h`、`strings/ctype-utf8.cc`、`strings/uca900_data.h` 等文件中的部分 `// 中文` 注释为本工作区人工批注，非 Oracle 上游原文。

---

## 参考

**官方文档**
- [MySQL Globalization → 1.10.1 Unicode Character Sets](https://dev.mysql.com/doc/mysql-g11n-excerpt/8.0/en/charset-unicode-sets.html)——★ 本节的权威基准：UCA 版本对应关系（`unicode_ci`=4.0.0 / `_520_ci`=5.2.0 / `_0900_*`=9.0.0，且 0900 系列同时基于 **CLDR v30**）、pad 属性清单、implicit weight 官方算法、`general_ci` vs `unicode_ci` 的权威结论（`ß=s` vs `ß=ss`）、非 BMP 字符权重、语言 specifier 表
- *Unicode Technical Standard #10: Unicode Collation Algorithm* —— 权重级别模型（§4）、implicit weight（§7.1.3）、contraction（§4.2）；DUCET 权重数据见 unicode.org 的 allkeys.txt

**WorkLog**
- WL#4013：utf8mb4（完整 Unicode）引入
- WL#10461 / WL#12357 系列：UCA 9.0.0 collation（utf8mb4_0900_*）

**Bug 论坛**
- [Bug#119463](https://bugs.mysql.com/bug.php?id=119463)：`utf8mb4_0900_ai_ci` 下 `lower('aaaȾbbb')` 截断为 `'aaaⱦ'`——大小写转换 inplace 下字节膨胀导致 src/dst 错位（本篇「大小写转换」节完整剖析根因与三层修复链；本地仓库含修复 commit 与测试 `mysql-test/suite/collations/t/bug_119463.test`）
- Bug#37086818：`my_strnncollsp_utf32` 的 `assert((slen % 4) == 0)` 断言失败（该断言在 8.0.39 仍存活，见本篇「其他 Unicode 字符集家族」节）

**内核月报 / 技术文章**
- [数据库内核月报 2017/03：《MySQL · 实现分析 · 对字符集和字符序支持的实现》](https://www.bookstack.cn/read/aliyun-rds-core/5fa32471f61eeddf.md)——句柄虚函数表组织、Unicode 媒介间接转换、Index.xml 配置式新增 collation（★ 该路径 8.0 已删除，本文档扩展点节已按源码修正）
- [阿里云开发者社区：《详解 MySQL 字符集和 Collation》](https://developer.aliyun.com/article/1462909)（张熙哲，RDS 内核研发）——DUCET CE 三元组示例、Sort Key 拼接流程、UCA 4.0 非 BMP 0xFFFD 局限、COERCIBILITY 表（均已回源码核实后入文）

**相关文档**
- ★ 本篇讲的 `wildcmp` / `like_range` 回调被谁消费、LIKE 的完整链路见 [`../query/15_like.md`](../query/15_like.md)
- 字节级编码（`mach_*` 序列化）见 [`encoding.md`](encoding.md)
- 索引内比较的整体流程见 [`../../innodb/row_search.md`](../../innodb/row_search.md)
- 排序（filesort 用排序键）见 [`../query/runtime/01_filesort.md`](../query/runtime/01_filesort.md)
- 数据字典的字符集元数据存储见 [`../dd/dd.md`](../dd/dd.md)
- ★ collation 对索引可用性的影响（"为什么换了 collation 就全表扫描"）见 [`../query/07_optimize/21_collation_index_usability.md`](../query/07_optimize/21_collation_index_usability.md)

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
//      + my_string_stack_guard 防栈溢出（超深 '%' 模式炸栈）
// '_' → 跳过一个码点
// escape 字符 → 下一个字符原样比较
// 等值判定 → my_uca_charcmp（走权重而非字节）
```

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
### 协议层转换链

**为什么以 Unicode 为 pivot**：任意两个字符集互转若都实现直接转换函数，N 个字符集需要 **N×(N-1)** 个转换函数；改为"源 → Unicode → 目标"后，每个字符集只需实现 `mb_wc` 与 `wc_mb` 两个函数，**总量降到 2N**。这是字符集转换层最根本的一个设计决策。代价是每次转换都要经过"解码 + 编码"两趟，无法做字符集间的字节级捷径。

转换的完整上层入口：`String::copy`（sql-common/sql_string.cc）先 `needs_conversion()` 判断是否要转，binary 源直接 `copy_aligned()` 字节级拷贝，其余走 `copy_and_convert` → `my_convert`（strings/ctype.cc）。发送端 `Protocol_classic` 判断 `!my_charset_same(fromcs, result_cs) && 双方非 binary` 才转换（character_set_results）。

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
9. **binary 源是"不转换"不是"不校验"**：源字符集为 binary 时逐字节复制、不做转码，但后续仍按目标字符集做合法性检测——utf8mb4 字节塞进 gbk 列时，"奇数个汉字报错、偶数个成功"是字节数凑巧整除的假象（见转换链的 mysqldump 案例）。：`caseup_multiply == 1` 假设大小写不改变字节数，但 Ⱦ(U+023E,2B)↔ⱦ(U+2C66,3B) 这类跨编码边界的映射对会让 inplace 转换 src/dst 错位、截断字符串。修复后转换函数对"原地失败"返回 -1，上层扩容重试。

### 易混淆

- **`character_set_connection` vs `character_set_client`**：client 是"进来的字节"，connection 是"解析后内部使用"（两者通常一起变），results 是"出去的字节"
- **collation 的 bin 与 binary charset**：`utf8mb4_bin`（比较按码点，但仍是 utf8mb4 家族）vs `binary`（charset=binary，比较纯字节 memcmp，含长度敏感）——`CHAR(10) BINARY` 的 pad 行为完全不同
- **`sort_order` 表 vs `strnxfrm`**：单字节老 collation 用 sort_order 表直接查表比较（不生成排序键）；`MY_CS_STRNXFRM` 置位的才走 strnxfrm——两者并存是历史分层
- **`levels_for_compare` 的 ai/as 与 `_ci` 的 ci**：老 `_ci` 的"case insensitive"靠权重表里大小写同权重实现；900 系列的 ai/as 靠"比几级"实现——机制完全不同

### 扩展点

加一个新语言 collation（如 `utf8mb4_0900_xx_ai_ci`）要动：

1. 生成权重定制数据（脚本生成 `uca900_xx_data.h` 风格的权重页）
2. `ctype-uca.cc` 里加一个 `CHARSET_INFO my_charset_utf8mb4_0900_xx_ai_ci` 定义（挂 handler + tailoring 规则串）
3. `mysys/charset-def.cc` 的 `init_compiled_charsets` 注册它
4. `share/charsets/Index.xml` 加 `<collation>` 条目（供名字/编号注册与 I_S 元数据）
5. dd 初始化时 `mysql.collations` 表自动从 all_charsets 填充，无需手改

**★ 历史提醒（内核月报时代的路径已死）**：5.6/5.7 时代可以"改 Index.xml 加 `<rules><reset>...</reset><i>...</i></rules>` 重启即得新 collation"（月报 2017/03 的 utf8_phone_ci 实例），**8.0 这条路径整体删除**——`my_charset_rule.c` 文件与 `my_coll_rule_parse` 已不存在，Index.xml 里也没有任何 `<rules>` 元素；XML 解析器残留的 `rules/reset/i` 标签语义已变成 **UCA tailoring 文本拼接**（`reset` → `&`，`i` → 忽略级规则），只对 UCA 类 collation 生效。8.0 新增 collation 必须改源码（`share/charsets/` 定义 + `strings/conf_to_src.cc` 重新生成 `ctype-*.cc`）后重新编译。运行期仅保留按需加载 `<csname>.xml` 的机制。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Character Sets, Collations, Unicode*（pad 属性、ai/as/ks 语义的基准）
- *Unicode Technical Standard #10: Unicode Collation Algorithm*（权重级别模型、implicit weight 规则 7.1.3、contraction）

**WorkLog**
- WL#4013：utf8mb4（完整 Unicode）引入
- WL#10461 / WL#12357 系列：UCA 9.0.0 collation（utf8mb4_0900_*）

**Bug 论坛**
- Bug#119463：`utf8mb4_0900_ai_ci` 下 `lower('aaaȾbbb')` 截断为 `'aaaⱦ'`——大小写转换 inplace 下字节膨胀导致 src/dst 错位（本篇「大小写转换」节完整剖析根因与三层修复链）
- 索引排序键与 ALTER COLLATE 的错乱类 bug（换 collation 不重建索引的语义坑出处）

**内核月报 / 技术文章**
- [数据库内核月报 2017/03：《MySQL · 实现分析 · 对字符集和字符序支持的实现》](https://www.bookstack.cn/read/aliyun-rds-core/5fa32471f61eeddf.md)——句柄虚函数表组织、Unicode 媒介间接转换、Index.xml 配置式新增 collation（★ 该路径 8.0 已删除，本文档扩展点节已按源码修正）
- [阿里云开发者社区：《详解 MySQL 字符集和 Collation》](https://developer.aliyun.com/article/1462909)（张熙哲，RDS 内核研发）——DUCET CE 三元组示例、Sort Key 拼接流程、UCA 4.0 非 BMP 0xFFFD 局限、COERCIBILITY 表（均已回源码核实后入文）

**相关文档**
- 字节级编码（`mach_*` 序列化）见 [`encoding.md`](encoding.md)
- 索引内比较的整体流程见 [`../../innodb/row_search.md`](../../innodb/row_search.md)
- 排序（filesort 用排序键）见 [`../query/runtime/01_filesort.md`](../query/runtime/01_filesort.md)
- 数据字典的字符集元数据存储见 [`../dd/dd.md`](../dd/dd.md)
- ★ collation 对索引可用性的影响（"为什么换了 collation 就全表扫描"）见 [`../query/07_optimize/21_collation_index_usability.md`](../query/07_optimize/21_collation_index_usability.md)

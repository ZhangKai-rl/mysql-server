# MySQL DBUG 调试框架深度解析

> 基于 MySQL 8.0.39 源码，涵盖 DBUG 宏体系、trace 输出、调试模式控制、与 InnoDB trace 的关系。

## 目录

- [概述](#概述)
  - [是什么 / 用途 / 版本演进](#是什么)
- [理论基础](#理论基础)
  - [历史背景（Fred Fish 1987）/ 设计模式 / 数据结构 / 类似实现对比](#历史背景一份-1987-年的公共领域代码)
- [DBUG 宏体系](#dbug-宏体系)
- [运行时实现（mysys/dbug.cc）](#运行时实现mysysdbugcc)
- [trace 输出与格式](#trace-输出与格式)
- [调试模式控制](#调试模式控制)
  - [控制字符串解析 / 全部控制标志 / 三个入口](#控制字符串解析dbugparse)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [debug_sync](#debug_sync与-dbug-协作的另一套机制)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)
- [★★ 真名勘误全表](#-真名勘误全表)

---

## 概述

### 是什么

DBUG 是 MySQL 从 **Fred Fish 1987 年公共领域的 dbug 包**继承来的运行时调试框架：一套 C 宏 + 一个运行时库（`mysys/dbug.cc`），提供**函数进入/退出追踪**、**关键字过滤的条件打印**、**内存 dump**、以及**故障注入点**四种能力。

它的核心设计是**"宏在编译期消失，运行时库只在开 debug 时才被调用"**——release 构建（`NDEBUG`）下所有 DBUG 宏展开为空语句，零开销。

**★ 开关不是 `DBUG_OFF` 而是标准 `NDEBUG`**（⚠️ 这是本篇第一条勘误）：

```cpp
/* include/my_dbug.h:57 */
#if !defined(NDEBUG)
#include <assert.h>  // IWYU pragma: keep
#include <stdio.h>
...                    /* ★ 全部 debug 版宏定义 */
#else /* No debugger */ /* my_dbug.h:231 */
...                    /* ★ 全部空宏 */
#endif
```

```cmake
# CMakeLists.txt:2651-2655 —— 构建类型的初始值注释原文
#      DEBUG          -g                  ← 无 -DNDEBUG → DBUG 生效
#      RELWITHDEBINFO -O2 -g -DNDEBUG     ← NDEBUG → DBUG 全空
#      RELEASE        -O3 -DNDEBUG
#      MINSIZEREL     -Os -DNDEBUG
```

> ⚠️ **`DBUG_OFF` 在 8.0.39 里没有任何定义**——全仓库仅 `storage/perfschema/pfs.cc:6324` 一处 `#ifndef DBUG_OFF`（遗留死代码，该分支恒编译）。真正的开关链是 **`WITH_DEBUG` → `CMAKE_BUILD_TYPE=Debug` → 不加 `-DNDEBUG`**。

### 用途

| 能力 | 宏 | 典型场景 |
|---|---|---|
| 函数追踪 | `DBUG_TRACE` / `DBUG_ENTER` + `DBUG_RETURN` | 看调用栈与嵌套层级 |
| 条件打印 | `DBUG_PRINT(keyword, ("fmt", args))` | 按关键字筛选输出 |
| 内存 dump | `DBUG_DUMP(keyword, ptr, len)` | 看损坏的内存/主键值 |
| 故障注入 | `DBUG_EXECUTE_IF(keyword, code)` | MTR 里模拟磁盘满、崩溃、随机失败 |
| 崩溃注入 | `DBUG_SUICIDE()` | 在指定点 SIGKILL（**不产生 core**） |
| 同步点 | `DEBUG_SYNC(thd, "point")` | 多线程时序编排（**另一套机制**，见下） |

### 版本演进

| 版本 | 变化 |
|------|------|
| 1987 | Fred Fish 发布 dbug 包并放弃版权（文件头注释原文 "Copyright Abandoned, 1987, Fred Fish"） |
| 1989 | SCCS 版本 1.25（`mysys/dbug.cc` 里仍保留 `@(#)dbug.c 1.25 7/25/89`） |
| MySQL 早期 | Michael Widenius 加 `DBUG_DUMP`、`O`/`A` 标志、`S` 选项 |
| MySQL 5.x | Sergei Golubchik 加 `DBUG_EXECUTE_IF`、增量模式（`-#+t:-d,info`）、`DBUG_SET`、thread-local settings、负向列表、`function/` 层次语法 |
| 8.0 | ★ **`DBUG_TRACE`（RAII）取代 `DBUG_ENTER`/`DBUG_RETURN` 成为主流写法**；`DBUG_ASSERT` 废弃改用原生 `assert()`；新增 `DBUG_LOG` / `DBUG_VAR` / `dbug()` lambda 接口 |

> ★ `DBUG_TRACE` 命中 **695+ 文件**，而 `DBUG_RETURN` 只剩 **13 个文件**（`mysys/my_compress.cc`、group_replication 插件、`libmysql/authentication_win` 等遗留代码）。

---

## 理论基础

### 历史背景：一份 1987 年的公共领域代码

`mysys/dbug.cc` 的文件头注释值得一字不漏地读——它解释了为什么这个模块的代码风格与 MySQL 其他部分完全不同：

```cpp
/******************************************************************************
 *                                 N O T I C E                                *
 *                    Copyright Abandoned, 1987, Fred Fish                    *
 *      This previously copyrighted work has been placed into the  public     *
 *      domain  by  the  author  and  may be freely used for any purpose,     *
 *      private or commercial.                                                *
 *      Because of the number of inquiries I was receiving about the  use     *
 *      of this product in commercially developed works I have decided to     *
 *      simply make it public domain to further its unrestricted use.   I     *
 *      specifically  would  be most happy to see this material become a      *
 *      part of the standard Unix distributions by AT&T and the  Berkeley     *
 *      Computer  Science  Research Group, and a standard part of the GNU     *
 *      system from the Free Software Foundation.                             *
 ******************************************************************************
 */

/*
 *  FILE
 *      dbug.c   runtime support routines for dbug package
 *  SCCS
 *      @(#)dbug.c      1.25    7/25/89
 *
 *  AUTHOR(S)
 *      Fred Fish               (base code)
 *      Binayak Banerjee        (profiling enhancements)
 *
 *      Michael Widenius:
 *        DBUG_DUMP       - To dump a block of memory.
 *        PUSH_FLAG "O"   - To be used instead of "o" if we want flushing after each write
 *        PUSH_FLAG "A"   - as 'O', but we will append to the out file instead
 *        Check of malloc on entry/exit (option "S")
 *
 *      Sergei Golubchik:
 *        DBUG_EXECUTE_IF
 *        incremental mode (-#+t:-d,info ...)
 *        DBUG_SET, _db_explain_
 *        thread-local settings
 *        negative lists (-#-d,info => everything but "info")
 *        function/ syntax
 *        (the logic is - think of a call stack as of a path.
 *        "function" means only this function, "function/" means the hierarchy.
 *        following this logic glob(7) wildcards are supported.)
 */
```

> ⚠️ **文件头注释里提到的 `S` 选项（malloc 检查）在 8.0.39 的 `DbugParse()` 里根本没有 `case 'S'`**——它只存在于这段历史说明中。

### 设计模式

三个正交的设计：

1. **编译期消除（Conditional Compilation）**：`NDEBUG` 下宏展开为空，debug 代码不进 release 二进制。这是"零开销调试"的基础。
2. **栈帧寄生（Caller-Allocated Frame）**：`_db_stack_frame_` 分配在**被跟踪函数自己的 C 栈**上，dbug 自己**不维护函数栈**。源码注释原文：*"this is why the ENTER macro must be the first 'executable' code in a function"*。
3. **脏读快路径（Dirty Read Fast Path）**：所有热路径函数开头都是 `if (!DEBUGGING) return;`——**不加锁地读** `flags` 位。注释原文 *"Dirty read, for DBUG_PRINT() performance."* 出现了 3 次（`_db_doprnt_`、`_db_dump_`、`_db_keyword_`）。

### 算法与数据结构

```
每线程一份（懒分配，TLS 只存一个指针）
  THR_mysys (thread_local)  →  st_my_thread_var { my_thread_id id; CODE_STATE *dbug; }
                                                      │
  CODE_STATE ─────────────────────────────────────────┘
    ├─ process / func / func_len / file     当前"位置"
    ├─ framep  ──> _db_stack_frame_ ──> prev ──> ... ──> nullptr   ★ 寄生在各函数 C 栈上
    ├─ stack   ──> settings(会话级) ──> settings ──> &init_settings  ★ 全局，链尾恒定
    ├─ level / lineno                       嵌套深度 / 输出行号
    └─ u_line / u_keyword / locked          ★ DBUG_PRINT 的两段式中转
```

**★ 四个真实结构体**（⚠️ **不存在 `class Debug` / `Indenter` / `WriteBuffer` / `Path` 等类**——8.0.39 仍是纯 C 风格）：

```cpp
/* include/my_dbug.h:59 —— 栈帧，寄生在调用者 C 栈上 */
struct _db_stack_frame_ {
  const char *func;   /* function name of the previous stack frame       */
  int func_len;       /* how much to print from func */
  const char *file;   /* filename of the function of previous frame      */
  unsigned int level; /* this nesting level, highest bit enables tracing */
  struct _db_stack_frame_ *prev; /* pointer to the previous frame */
};
```

```cpp
/* mysys/dbug.cc:156 —— 关键字/函数名/进程名链表节点（struct hack 柔性数组） */
struct link {
  struct link *next_link; /* Pointer to the next link */
  char flags;
  char str[1]; /* Pointer to link's contents */   /* ★ sizeof(link)+len 一次分配 */
};

/* flags for struct link and return flags of InList */
#define SUBDIR 1        /* this MUST be 1 —— 对应控制串里的 "func/" */
#define INCLUDE 2
#define EXCLUDE 4
#define MATCHED 65536   /* ★ "列表为空/只有 EXCLUDE 项"时的默认返回值 = 通配接受 */
#define NOT_MATCHED 0
```

```cpp
/* mysys/dbug.cc:186 —— 一份"控制串"的解析结果，可压栈 */
struct settings {
  uint flags;               /* Current settings flags               */
  uint maxdepth;            /* Current maximum trace depth          */
  uint delay;               /* Delay after each output line         */
  uint sub_level;           /* Sub this from code_state->level      */
  FILE *out_file;           /* Current output stream                */  /* ★ 唯一出口 */
  FILE *prof_file;          /* Current profiling stream             */
  char name[FN_REFLEN];     /* Name of output file                  */
  struct link *functions;   /* List of functions                    */
  struct link *p_functions; /* List of profiled functions           */
  struct link *keywords;    /* List of debug keywords               */  /* ★ -d,xxx */
  struct link *processes;   /* List of process names                */
  struct settings *next;    /* Next settings in the list            */  /* ★ 栈顶在头部 */
};

#define is_shared(S, V) ((S)->next && (S)->next->V == (S)->V)  /* 共享就不能单独 free */
```

```cpp
/* mysys/dbug.cc:217 —— 每线程运行时状态（my_dbug.h:67 只前向声明） */
struct CODE_STATE {
  const char *process;   /* Pointer to process name; usually argv[0] */
  const char *func;      /* Name of current user function            */
  int func_len;          /* How many bytes to print from func        */
  const char *file;      /* Name of current user file                */
  struct _db_stack_frame_ *framep;  /* Pointer to current frame      */
  struct settings *stack;           /* debugging settings             */
  int lineno;            /* Current debugger output line number      */
  uint level;            /* Current function nesting level           */

  /*
   *      The following variables are used to hold the state information
   *      between the call to _db_pargs_() and _db_doprnt_(), during
   *      expansion of the DBUG_PRINT macro.  This is the only macro
   *      that currently uses these variables.
   */
  uint u_line;           /* User source code line number */
  int locked;            /* If locked with _db_lock_file_ */
  const char *u_keyword; /* Keyword for current macro */
  uint m_read_lock_count;                              /* ★ 可重入读锁计数 */
};

#define get_code_state_or_return if (!((cs = code_state()))) return
```

### 类似实现对比

| 机制 | 开关 | 与 DBUG 的关系 |
|---|---|---|
| **DBUG** | `NDEBUG` + 运行时 `DEBUG_ON` 标志 | 本模块 |
| **InnoDB `ut_a` / `ut_ad` / `ut_d`** | **`UNIV_DEBUG`**（不是 NDEBUG！） | **完全解耦**。`ut_a` 在 release 下**也成立**（永远断言）；`ut_ad`/`ut_d` 由 `UNIV_DEBUG` 控制 |
| **debug_sync** | `ENABLED_DEBUG_SYNC` + `opt_debug_sync_timeout` | **独立机制**，只借道 `DBUG_PRINT` 写 trace |
| **`assert()`** | `NDEBUG` | 8.0 用它取代了已废弃的 `DBUG_ASSERT` |
| **栈保护 canary** | 编译器 `-fstack-protector` | 与 DBUG 无关，编译器注入 |

---

## DBUG 宏体系

`DBUG_TRACE` / `DBUG_PRINT` / `DBUG_RETURN` / `DBUG_ENTER` / `DBUG_DUMP` 等宏的展开与语义。

- `DBUG_DUMP(keyword, ptr, len)`（`my_dbug.h:195`）→ `_db_dump_(__LINE__, keyword, ptr, len)`：debug 模式下把 `ptr` 起始的 `len` 字节按十六进制 dump 到 trace 输出；release 版是空宏 `do{}while(0)`（`my_dbug.h:273`），零开销。
- 用法示例：`ha_innobase::rnd_pos`（`ha_innodb.cc:10870`）里 `DBUG_DUMP("key", pos, ref_length)` 用于 dump 主键值（ref）调试。

**★ 完整宏表（debug 版，`my_dbug.h:95-229`）**：

```cpp
/* ---- ① 函数追踪：RAII 版（8.0 主流） ---- */
#ifdef __cplusplus                          /* ★★ DBUG_TRACE 只在 C++ 下存在 */
#if defined(__GNUC__)
#define DBUG_PRETTY_FUNCTION __PRETTY_FUNCTION__
#elif defined(__FUNCSIG__)                  /* MSVC：跳过 __cdecl */
#define DBUG_PRETTY_FUNCTION strchr(__FUNCSIG__, ' ') + 1
#else
#define DBUG_PRETTY_FUNCTION __func__
#endif

class AutoDebugTrace {
 public:
  AutoDebugTrace(const char *function, const char *filename, int line) {
    // Remove the return type, if it's there.
    const char *begin = strchr(function, ' ');
    if (begin != nullptr) function = begin + 1;
    // Cut it off at the first parenthesis; the argument list is
    // often too long to be interesting.
    const char *end = strchr(function, '(');
    if (end == nullptr) _db_enter_(function, strlen(function), filename, line, &m_stack_frame);
    else                _db_enter_(function, end - function,  filename, line, &m_stack_frame);
  }
  ~AutoDebugTrace() { _db_return_(0, &m_stack_frame); }   /* ★★ _line_ 传 0 */
 private:
  _db_stack_frame_ m_stack_frame;           /* ★ 内嵌在 RAII 对象里 */
};

#define DBUG_TRACE \
  const AutoDebugTrace _db_trace(DBUG_PRETTY_FUNCTION, __FILE__, __LINE__)
#endif /* __cplusplus */
```

```cpp
/* ---- ② 函数追踪：经典版（遗留） ---- */
#define DBUG_ENTER(a)                        \
  struct _db_stack_frame_ _db_stack_frame_;  /* ★ 裸栈变量，名字被 DBUG_RETURN 硬编码引用 */ \
  _db_enter_(a, ::strlen(a), __FILE__, __LINE__, &_db_stack_frame_)
#define DBUG_RETURN(a1)                       \
  do {                                        \
    _db_return_(__LINE__, &_db_stack_frame_); \
    return (a1);                              \
  } while (0)
#define DBUG_VOID_RETURN                      \
  do {                                        \
    _db_return_(__LINE__, &_db_stack_frame_); \
    return;                                   \
  } while (0)
```

> ★★ **`DBUG_TRACE` 与 `DBUG_ENTER` 是完全不同的机制**：前者是 C++ RAII 对象（析构自动退出，函数里直接 `return 42;` 即可），后者在调用者栈上声明裸 `_db_stack_frame_` 且**必须**配对 `DBUG_RETURN`。
> ⚠️ **`DBUG_LEAVE` 不存在**——退出侧只有 `DBUG_RETURN` / `DBUG_VOID_RETURN` / `~AutoDebugTrace()`。
> ⚠️ **宏里没有 `__builtin_expect`**——快路径靠运行时库里的无锁脏读。

```cpp
/* ---- ③ 条件打印（两段式） ---- */
#define DBUG_PRINT(keyword, arglist) \
  do {                               \
    _db_pargs_(__LINE__, keyword);   /* ★ 先暂存 __LINE__ 与 keyword 到 CODE_STATE */ \
    if (_db_enabled_()) {            /* ★ 再问"这个 keyword 该不该输出" */ \
      _db_doprnt_ arglist;           \
    }                                \
  } while (0)
#define DBUG_DUMP(keyword, a1, a2) _db_dump_(__LINE__, keyword, a1, a2)
#define DBUG_FILE _db_fp_()
#define DBUG_LOCK_FILE _db_lock_file_()
#define DBUG_UNLOCK_FILE _db_unlock_file_()
#define DBUG_END() _db_end_()
#define DBUG_EXPLAIN(buf, len) _db_explain_(0, (buf), (len))
#define DBUG_EXPLAIN_INITIAL(buf, len) _db_explain_init_((buf), (len))
```

```cpp
/* ---- ④ 控制串 ---- */
#define DBUG_PUSH(a1) _db_push_(a1)
#define DBUG_POP() _db_pop_()
#define DBUG_SET(a1) _db_set_(a1)
#define DBUG_SET_INITIAL(a1) _db_set_init_(a1)
#define DBUG_PROCESS(a1) _db_process_(a1)
```

```cpp
/* ---- ⑤ 故障注入（★ 测试的核心） ---- */
#define DBUG_EXECUTE(keyword, a1)        \
  do {                                   \
    if (_db_keyword_(0, (keyword), 0)) { \   /* strict=0 */
      a1                                 \
    }                                    \
  } while (0)
#define DBUG_EXECUTE_IF(keyword, a1)     \
  do {                                   \
    if (_db_keyword_(0, (keyword), 1)) { \   /* ★ strict=1 */
      a1                                 \
    }                                    \
  } while (0)
#define DBUG_EVALUATE(keyword, a1, a2)   (_db_keyword_(0, (keyword), 0) ? (a1) : (a2))
#define DBUG_EVALUATE_IF(keyword, a1, a2)(_db_keyword_(0, (keyword), 1) ? (a1) : (a2))
```

> ★★★ **`DBUG_EXECUTE` 与 `DBUG_EXECUTE_IF` 的唯一区别就是第三个参数 `strict`**：
> ```cpp
>   strict = strict ? INCLUDE : INCLUDE | MATCHED;
> ```
> - `strict=0`：**关键字列表为空也算命中**（裸 `-d` 全开即可触发）。
> - `strict=1`：**必须被 `-d,my_keyword` 显式包含**才命中。这才是"注入点"语义——默认永不触发。

```cpp
/* ---- ⑥ 崩溃 / 退出 ---- */
#ifndef _WIN32
#define DBUG_ABORT() (_db_flush_(), my_abort())
#define DBUG_EXIT() (_db_flush_(), exit(2))
#else
#define DBUG_ABORT() (_db_flush_(), (void)_CrtSetReportMode(...), my_abort())
#define DBUG_EXIT()  (_db_flush_(), (void)_CrtSetReportMode(...), _exit(2))
#endif

/*
  Make the program fail, without creating a core file.
  abort() will send SIGABRT which (most likely) generates core.
  Use SIGKILL instead, which cannot be caught.
  We also pause the current thread, until the signal is actually delivered.
  An alternative would be to use _exit(EXIT_FAILURE),
  but then valgrind would report lots of memory leaks.
 */
#ifdef _WIN32
#define DBUG_SUICIDE() DBUG_EXIT()
#else
[[noreturn]] extern void _db_suicide_();
extern void _db_flush_gcov_();
#define DBUG_SUICIDE() (_db_flush_(), _db_suicide_())
#endif
```

> ⚠️ **`DBUG_ASSERT` / `DBUG_CHECK_CRASHED` 在 8.0.39 都不存在**（前者只在 mysql-test 的注释文字里遗留，后者 0 命中）。8.0 的断言统一用原生 `assert()`（`my_dbug.h:40` 在 `!NDEBUG` 时才引入 `<assert.h>`）。

**★ C++ 层的三个便利接口**（`my_dbug.h:297-364`）：

```cpp
/* ① DBUG_LOG —— C++ 流式输出 */
/*
  Note: DBUG_PRINT() has a limitation of 1024 bytes for a single
  print out.  So, this limitation is there for DBUG_LOG macro also.
*/
#define DBUG_LOG(keyword, v)                           \
  do {                                                 \
    _db_pargs_(__LINE__, keyword);                     \
    if (_db_enabled_()) {                              \
      std::ostringstream sout;                         \
      sout << v;                                       \
      DBUG_PRINT(keyword, ("%s", sout.str().c_str())); \
    }                                                  \
  } while (0)

/* ② DBUG_VAR —— 变量名 + 值，形如 i=[4711] */
#define DBUG_VAR(v) #v << "=[" << (v) << "]"

/* ③ dbug() —— lambda 版 DBUG_EXECUTE_IF */
template <class DBGCLOS>
inline void dbug(const char *keyword [[maybe_unused]],
                 DBGCLOS &&clos [[maybe_unused]]) {
  DBUG_EXECUTE_IF(keyword, clos(););
}
```

> ★ `dbug()` 的 `@note` 值得注意：*"The body of the callable will be checked by the compiler even in optimized mode."*——release 下 lambda **仍会被编译期检查**（语法完整），只是不被调用。

**★ release 版（`NDEBUG`）的全部空宏**（`my_dbug.h:231-295`，★ 零开销的证明）：

```cpp
#else /* No debugger */

#ifdef __cplusplus
#define DBUG_TRACE do { } while (false)
#endif
#define DBUG_ENTER(a1)                              /* ★ 完全空，连 do{}while(0) 都没有 */
#define DBUG_RETURN(a1) do { return (a1); } while (0)
#define DBUG_VOID_RETURN do { return; } while (0)
#define DBUG_EXECUTE(keyword, a1) do { } while (0)
#define DBUG_EXECUTE_IF(keyword, a1) do { } while (0)
#define DBUG_EVALUATE(keyword, a1, a2) (a2)         /* ★★ 折叠为"永不注入"那一支 */
#define DBUG_EVALUATE_IF(keyword, a1, a2) (a2)
#define DBUG_PRINT(keyword, arglist) do { } while (0)
#define DBUG_PUSH(a1) do { } while (0)
#define DBUG_SET(a1) do { } while (0)
#define DBUG_SET_INITIAL(a1) do { } while (0)
#define DBUG_POP() do { } while (0)
#define DBUG_PROCESS(a1) do { } while (0)
#define DBUG_DUMP(keyword, a1, a2) do { } while (0)
#define DBUG_END() do { } while (0)
#define DBUG_LOCK_FILE do { } while (0)
#define DBUG_FILE (stderr)                          /* ★★ 唯一"非空"的一个 */
#define DBUG_UNLOCK_FILE do { } while (0)
#define DBUG_EXPLAIN(buf, len)                      /* 完全空 */
#define DBUG_EXPLAIN_INITIAL(buf, len)              /* 完全空 */
#define DBUG_ABORT() do { } while (0)
#define DBUG_SUICIDE() do { } while (0)

#endif
```

> ★ 三个细节：① `DBUG_ENTER` / `DBUG_EXPLAIN` 展开为**完全空**，所以 `DBUG_ENTER("x");` 的分号变成空语句；② `DBUG_EVALUATE*` 折叠为 `(a2)`，编译器完全消除 `a1`；③ release 下**没有** `DBUG_EXIT()` 定义，`DBUG_FILE` 直接是 `stderr`。

**★ 底层函数声明**（`my_dbug.h:69-93`，★ 全部以 `_db_xxx_` 命名避免与用户符号冲突）：

```cpp
struct CODE_STATE;                          /* ★ 只前向声明，真身在 dbug.cc:217 */

extern int _db_keyword_(struct CODE_STATE *, const char *, int);
extern int _db_explain_(struct CODE_STATE *cs, char *buf, size_t len);
extern int _db_explain_init_(char *buf, size_t len);
extern int _db_is_pushed_(void);
extern void _db_process_(const char *name);
extern void _db_push_(const char *control);
extern void _db_pop_(void);
extern void _db_set_(const char *control);
extern void _db_set_init_(const char *control);
extern void _db_enter_(const char *_func_, int func_len, const char *_file_,
                       unsigned int _line_, struct _db_stack_frame_ *_stack_frame_);
extern void _db_return_(unsigned int _line_, struct _db_stack_frame_ *_stack_frame_);
extern void _db_pargs_(unsigned int _line_, const char *keyword);
extern int _db_enabled_();
extern void _db_doprnt_(const char *format, ...)
    MY_ATTRIBUTE((format(printf, 1, 2)));   /* ★ 编译器帮你校验格式串 */
extern void _db_dump_(unsigned int _line_, const char *keyword,
                      const unsigned char *memory, size_t length);
extern void _db_end_(void);
extern void _db_lock_file_(void);
extern void _db_unlock_file_(void);
extern FILE *_db_fp_(void);
extern void _db_flush_();
```

---

## 运行时实现（mysys/dbug.cc）

**★ `_db_enter_()`：函数进入**（`mysys/dbug.cc:1124`）：

```cpp
void _db_enter_(const char *_func_, int func_len, const char *_file_,
                uint _line_, struct _db_stack_frame_ *_stack_frame_) {
  int save_errno;
  CODE_STATE *cs;
  if (!((cs = code_state()))) {
    /* 线程未 init（my_thread_init 之前调用 DBUG_ENTER）时只清帧就返回，保证 valgrind 干净 */
    _stack_frame_->level = 0; _stack_frame_->prev = nullptr;
    return;
  }
  save_errno = errno;                    /* ★ DBUG 全程不破坏调用者的 errno */

  read_lock_stack(cs);

  _stack_frame_->func = cs->func;        /* ★ 把"上一个"函数/文件存进新帧（前驱语义） */
  _stack_frame_->func_len = cs->func_len;
  _stack_frame_->file = cs->file;
  cs->func = _func_; cs->func_len = func_len; cs->file = _file_;
  _stack_frame_->prev = cs->framep;      /* ★ 串成调用栈链表 */
  _stack_frame_->level = ++cs->level | framep_trace_flag(cs, cs->framep);
  /* ★★ level 低 31 位 = 嵌套深度，最高位 = 从父帧继承的 TRACE_ON */
  cs->framep = _stack_frame_;

  switch (DoTrace(cs)) {
    case ENABLE_TRACE:                   /* 命中 "func/"（SUBDIR）→ 从本帧起递归开启 */
      cs->framep->level |= TRACE_ON;
      if (!TRACING) break;
      [[fallthrough]];
    case DO_TRACE:
      if (TRACING) {
        if (!cs->locked) native_mutex_lock(&THR_LOCK_dbug);
        DoPrefix(cs, _line_);
        Indent(cs, cs->level);
        (void)fprintf(cs->stack->out_file, ">%.*s\n", cs->func_len, cs->func);
        /* ★★ trace 里的 ">func_name" 就是这一行 */
        DbugFlush(cs); /* This does a unlock */
      }
      break;
    case DISABLE_TRACE:                  /* 命中 "-func/" → 从本帧起关闭 */
      cs->framep->level &= ~TRACE_ON;
      [[fallthrough]];
    case DONT_TRACE: break;
  }
  errno = save_errno;
  unlock_stack(cs);
}
```

**★ `_db_return_()`：函数退出**（`mysys/dbug.cc:1193`）：

```cpp
void _db_return_(uint _line_, struct _db_stack_frame_ *_stack_frame_) {
  int save_errno = errno;
  uint _slevel_ = _stack_frame_->level & ~TRACE_ON;   /* ★ 剥掉 TRACE_ON 取纯深度 */
  CODE_STATE *cs;
  get_code_state_or_return;

  if (cs->framep != _stack_frame_) {                  /* ★ 漏写 DBUG_RETURN → 直接 abort */
    char buf[512];
    snprintf(buf, sizeof(buf), ERR_MISSING_RETURN, cs->func);
    DbugExit(buf);
  }

  read_lock_stack(cs);
  if (DoTrace(cs) & DO_TRACE) {                       /* ★ 位与：ENABLE_TRACE(3) 也算 */
    if (TRACING) {
      if (!cs->locked) native_mutex_lock(&THR_LOCK_dbug);
      DoPrefix(cs, _line_);
      Indent(cs, cs->level);
      if (_line_ == 0) {
        (void)fprintf(cs->stack->out_file, "<%.*s\n", cs->func_len, cs->func);
        /* ★★ DBUG_TRACE 的 RAII 析构传 0 → "<func"（无行号） */
      } else {
        (void)fprintf(cs->stack->out_file, "<%.*s %u\n", cs->func_len, cs->func, _line_);
        /* ★★ DBUG_RETURN/VOID_RETURN 传 __LINE__ → "<func 行号>" */
      }
      DbugFlush(cs);
    }
  }
  /* Check to not set level < 0. This can happen if DBUG was disabled when
     function was entered and enabled in function. */
  cs->level = _slevel_ != 0 ? _slevel_ - 1 : 0;
  cs->func = _stack_frame_->func; cs->func_len = _stack_frame_->func_len;
  cs->file = _stack_frame_->file;
  if (cs->framep != nullptr) cs->framep = cs->framep->prev;   /* 弹帧 */
  errno = save_errno;
  unlock_stack(cs);
}
```

**★ `_db_pargs_()` + `_db_enabled_()` + `_db_doprnt_()`：DBUG_PRINT 的三段**：

```cpp
void _db_pargs_(uint _line_, const char *keyword) {
  CODE_STATE *cs;
  get_code_state_or_return;
  cs->u_line = _line_;            /* ★ 暂存 __LINE__ */
  cs->u_keyword = keyword;        /* ★ 暂存 keyword */
}

int _db_enabled_() {
  CODE_STATE *cs;
  get_code_state_or_return 0;
  if (!DEBUGGING) return 0;                      /* ★ 脏读快路径 */
  if (_db_keyword_(cs, cs->u_keyword, 0)) return 1;   /* strict=0 */
  return 0;
}

/*
 *      Note that the format string SHOULD NOT include a terminating
 *      newline, this is supplied automatically.     ★★ 格式化约束原文
 */
void _db_doprnt_(const char *format, ...) {
  va_list args; CODE_STATE *cs; int save_errno;
  get_code_state_or_return;
  /* Dirty read, for DBUG_PRINT() performance. */
  if (!DEBUGGING) return;

  va_start(args, format);
  read_lock_stack(cs);
  save_errno = errno;
  if (!cs->locked) native_mutex_lock(&THR_LOCK_dbug);
  DoPrefix(cs, cs->u_line);
  if (TRACING) Indent(cs, cs->level + 1);        /* ★★ 开 trace：比 '>' 行再深一级 */
  else (void)fprintf(cs->stack->out_file, "%.*s: ", cs->func_len, cs->func);
  (void)fprintf(cs->stack->out_file, "%s: ", cs->u_keyword);
  DbugVfprintf(cs->stack->out_file, format, args);
  DbugFlush(cs);
  errno = save_errno;
  unlock_stack(cs);
  va_end(args);
}

MY_ATTRIBUTE((format(printf, 2, 0)))
static void DbugVfprintf(FILE *stream, const char *format, va_list args) {
  vfprintf(stream, format, args);
  fprintf(stream, "\n");          /* ★ 自动补换行 */
}
```

> ★ **`DBUG_PRINT` 的两条真实约束**：① 单次输出上限 **1024 字节**（`my_dbug.h:311`）；② 格式串**不要自带 `\n`**（`dbug.cc:1308`，由 `DbugVfprintf` 补）。**不存在**"只支持 `%s`"之类的白名单——`_db_doprnt_` 直接 `vfprintf`。

**★ `_db_dump_()`：十六进制 dump**：

```cpp
void _db_dump_(uint _line_, const char *keyword, const unsigned char *memory,
               size_t length) {
  int pos; CODE_STATE *cs;
  get_code_state_or_return;
  /* Dirty read, for DBUG_DUMP() performance. */
  if (!DEBUGGING) return;
  read_lock_stack(cs);
  if (_db_keyword_(cs, keyword, 0)) {
    if (!cs->locked) native_mutex_lock(&THR_LOCK_dbug);
    DoPrefix(cs, _line_);
    if (TRACING) Indent(cs, cs->level + 1);
    else fprintf(cs->stack->out_file, "%.*s: ", cs->func_len, cs->func);
    (void)fprintf(cs->stack->out_file, "%s: Memory: %p  Bytes: (%ld)\n",
                  keyword, memory, (long)length);
    pos = 0;
    while (length-- > 0) {
      uint tmp = *(memory++);
      if ((pos += 3) >= 80) { fputc('\n', cs->stack->out_file); pos = 3; }
      /* ★ 每字节 3 字符（2 hex + 1 空格），满 80 列换行 */
      fputc(_dig_vec_upper[((tmp >> 4) & 15)], cs->stack->out_file);
      fputc(_dig_vec_upper[tmp & 15], cs->stack->out_file);
      fputc(' ', cs->stack->out_file);
    }
    (void)fputc('\n', cs->stack->out_file);
    DbugFlush(cs);
  }
  unlock_stack(cs);
}
```

**★ 关键字判定：`_db_keyword_()`**（★ 注入点的总闸）：

```cpp
int _db_keyword_(CODE_STATE *cs, const char *keyword, int strict) {
  bool result;
  get_code_state_if_not_set_or_return false;   /* ★ cs 允许为 NULL（宏里直接传 0） */
  /* Dirty read, for DBUG_EXECUTE(), DBUG_EXECUTE_IF() ... performance. */
  if (!DEBUGGING) return false;
  read_lock_stack(cs);
  strict = strict ? INCLUDE : INCLUDE | MATCHED;
  /* ★★ strict=1 要求"显式 INCLUDE"；strict=0 接受 MATCHED（空列表=全通过） */
  result = DoTrace(cs) & DO_TRACE && InList(cs->stack->keywords, keyword) & strict;
  /* ★ 两道闸：① 当前函数/进程/深度是否 trace；② keyword 是否在列表里 */
  unlock_stack(cs);
  return result;
}
```

**★ `DoTrace()`：函数/进程/深度三道闸**：

```cpp
static int DoTrace(CODE_STATE *cs) {
  if ((cs->stack->maxdepth == 0 || cs->level <= cs->stack->maxdepth) &&
      InList(cs->stack->processes, cs->process) & (MATCHED | INCLUDE))
    switch (InList(cs->stack->functions, cs->func)) {
      case INCLUDE | SUBDIR:  return ENABLE_TRACE;   /* "func/"：本帧起向下全开 */
      case INCLUDE:           return DO_TRACE;       /* "func"：只本帧 */
      case MATCHED | SUBDIR:
      case NOT_MATCHED | SUBDIR:
      case MATCHED:
        return framep_trace_flag(cs, cs->framep) ? DO_TRACE : DONT_TRACE;
        /* ★★ 未显式列出 → 继承父帧的 TRACE_ON 位（"父开了我就开"） */
      case EXCLUDE:
      case NOT_MATCHED:       return DONT_TRACE;
      case EXCLUDE | SUBDIR:  return DISABLE_TRACE;  /* "-func/"：本帧起向下全关 */
    }
  return DONT_TRACE;
}

/* 返回值常量（注意可作为位掩码用） */
#define DO_TRACE 1
#define DONT_TRACE 2
#define ENABLE_TRACE 3     /* ★ 3 = 1|2，所以 "DoTrace(cs) & DO_TRACE" 对它也真 */
#define DISABLE_TRACE 4

/* 根帧规则（继承的源头） */
#define framep_trace_flag(cs, frp) \
  (frp ? frp->level & TRACE_ON \
       : (ListFlags(cs->stack->functions) & INCLUDE) ? 0 : (uint)TRACE_ON)
/* ★ f 列表里只要有 INCLUDE 项 → 默认不 trace；否则默认全 trace */
```

**★ `InList()`：glob 匹配**（★ 关键字/函数名都支持通配符）：

```cpp
static int InList(struct link *linkp, const char *cp) {
  int result;
  for (result = MATCHED; linkp != nullptr; linkp = linkp->next_link) {
    if (!fnmatch(linkp->str, cp, 0)) return linkp->flags;   /* ★★ glob(7) 通配 */
    if (!(linkp->flags & EXCLUDE)) result = NOT_MATCHED;
    if (linkp->flags & SUBDIR) result |= SUBDIR;
  }
  return result;
}
```

> ★ 注释原文解释"空列表 = 全匹配"的用意：*"If linkp is NULL or contains only EXCLUDE elements then the string is treated as if it is in the list. This may seem rather strange at first but leads to the desired operation if no list is given."*

**★ 线程安全：三把锁 + 可重入读锁**：

```cpp
static native_mutex_t THR_LOCK_dbug;          /* 保护"一次完整的输出行" */
static native_mutex_t THR_LOCK_gcov;          /* ★ 单独一把，原因见下注释 */
static native_rw_lock_t THR_LOCK_init_settings;  /* 保护全局 init_settings */

/**
  Lock the stack debugging settings.
  Only the shared (global) settings are locked if necessary,
  per thread settings are local and safe to use.
  This lock is re entrant.
*/
static void read_lock_stack(CODE_STATE *cs) {
  if (cs->stack == &init_settings) {           /* ★★ 只有"当前生效的是全局"才加锁 */
    if (++(cs->m_read_lock_count) == 1)
      native_rw_rdlock(&THR_LOCK_init_settings);
  }
}
static void unlock_stack(CODE_STATE *cs) {
  if (cs->stack == &init_settings) {
    if (--(cs->m_read_lock_count) == 0) native_rw_unlock(&THR_LOCK_init_settings);
  }
}
```

```cpp
/* DbugFlush —— 每条输出路径都以它结尾（内部承担解锁） */
static void DbugFlush(CODE_STATE *cs) {
  if (cs->stack->flags & FLUSH_ON_WRITE) {     /* ★ 只有 O/A（或 stdout）才真 fflush */
    (void)fflush(cs->stack->out_file);
    if (cs->stack->delay) (void)Delay(cs->stack->delay);
  }
  if (!cs->locked) native_mutex_unlock(&THR_LOCK_dbug);
}
```

> ★ `THR_LOCK_gcov` 单独存在的原因（`dbug.cc:309` 注释原文）：*"We don't re-use THR_LOCK_dbug, because that would disallow: DBUG_LOCK_FILE; ..... DBUG_SUICIDE(); .... DBUG_UNLOCK_FILE;"*
> ★ 改全局 settings 时升级为**写锁**（`DbugParse` 里 `if (stack == &init_settings) native_rw_wrlock(...)`），且 `assert(!cs->locked)` 禁止在 `DBUG_LOCK_FILE` 区间内改控制串。

**★ `_db_suicide_()`：不产生 core 的崩溃**：

```cpp
void _db_suicide_() {
  int retval;
  sigset_t new_mask;
  sigfillset(&new_mask);
#ifdef HAVE_GCOV
  fprintf(stderr, "Flushing gcov data\n"); fflush(stderr); _db_flush_gcov_();
#endif
  fprintf(stderr, "SIGKILL myself\n"); fflush(stderr);
  retval = kill(getpid(), SIGKILL);            /* ★ 用 SIGKILL 而非 abort() */
  assert(retval == 0);
  retval = sigsuspend(&new_mask);              /* ★ 填满信号掩码，保证信号先投递 */
  fprintf(stderr, "sigsuspend returned %d errno %d \n", retval, errno);
  assert(false); /* With full signal mask, we should never return here. */
  DBUG_ABORT();
}
```

**★ 两个真实用例**：

```cpp
/* 现代三件套（sql/hostname_cache.cc:446）—— 函数末尾无需任何 DBUG_* */
int ip_to_hostname(...) {
  DBUG_TRACE;                                  /* ★ RAII 进入 */
  DBUG_PRINT("info", ("IP address: '%s'; family: %d.", ip_string, (int)ip->sa_family));
  ...
  if (is_ip_loopback(ip)) {
    DBUG_PRINT("info", ("Loopback address detected."));
    *hostname = const_cast<char *>(my_localhost);
    return 0;                                  /* ★ 裸 return，析构自动打印 "<ip_to_hostname" */
  }
```

```cpp
/* 经典三件套（mysys/my_compress.cc:280）—— 8.0 里仅存的"活化石" */
bool my_compress(...) {
  DBUG_ENTER("my_compress");                   /* ★ 手写函数名字符串 */
  if (*len < MIN_COMPRESS_LENGTH) {
    *complen = 0;
    DBUG_PRINT("note", ("Packet too short: Not compressed"));
  } else {
    uchar *compbuf = my_compress_alloc(...);
    if (!compbuf) DBUG_RETURN(*complen ? 0 : 1);   /* ★ 早退也必须走 DBUG_RETURN */
    ...
  }
  DBUG_RETURN(0);
}
```

---

## trace 输出与格式

**★ 行前缀 `DoPrefix()`（`dbug.cc:1840`）—— 所有输出行格式的唯一来源**：

```cpp
static void DoPrefix(CODE_STATE *cs, uint _line_) {
  cs->lineno++;                                          /* 永远递增，供 'N' 编号 */
  if (cs->stack->flags & PID_ON)
    (void)fprintf(cs->stack->out_file, "T@%u: ", my_thread_var_id());   /* 'i' */
  if (cs->stack->flags & NUMBER_ON)
    (void)fprintf(cs->stack->out_file, "%5d: ", cs->lineno);            /* 'N' */
  if (cs->stack->flags & TIMESTAMP_ON) {
    struct timeval tv; struct tm *tm_p;
    if (gettimeofday(&tv, nullptr) != -1) {
      if ((tm_p = localtime((const time_t *)&tv.tv_sec)))
        (void)fprintf(cs->stack->out_file,
                      "%02d:%02d:%02d.%06d ",          /* 'T'：注意不带日期 */
                      tm_p->tm_hour, tm_p->tm_min, tm_p->tm_sec, (int)(tv.tv_usec));
    }
  }
  if (cs->stack->flags & PROCESS_ON)
    (void)fprintf(cs->stack->out_file, "%s: ", cs->process);            /* 'P' */
  if (cs->stack->flags & FILE_ON)
    (void)fprintf(cs->stack->out_file, "%14s: ", BaseName(cs->file));   /* 'F' */
  if (_line_ != 0 && (cs->stack->flags & LINE_ON))
    (void)fprintf(cs->stack->out_file, "%5d: ", _line_);                /* 'L' */
  if (cs->stack->flags & DEPTH_ON)
    (void)fprintf(cs->stack->out_file, "%4d: ", cs->level);             /* 'n' */
}
```

**★ 缩进 `Indent()`（⚠️ 不存在 `Indenter` 类）**：

```cpp
#define INDENT 2        /* Indentation per trace level */
static void Indent(CODE_STATE *cs, int indent) {
  int count;
  indent = std::max<int>(indent - 1 - cs->stack->sub_level, 0) * INDENT;
  /* ★ sub_level 是 'r' 选项记下的基准深度，可把深栈"拉平"回 0 */
  for (count = 0; count < indent; count++) {
    if ((count % INDENT) == 0) fputc('|', cs->stack->out_file);   /* ★ 竖线不是空格 */
    else                       fputc(' ', cs->stack->out_file);
  }
}
```

> ★ 所以缩进是 `| | | ` 这种**竖线台阶**，每级 2 字符。

**★ 四条输出路径的格式总结**：

| 宏 | 输出函数 | 缩进 | 正文 |
|---|---|---|---|
| `DBUG_ENTER` / `DBUG_TRACE` | `_db_enter_` | `Indent(cs, cs->level)` | `>func\n` |
| `DBUG_RETURN` / `DBUG_VOID_RETURN` | `_db_return_` | `Indent(cs, cs->level)` | `<func <行号>\n` |
| `DBUG_TRACE`（RAII 析构） | `_db_return_(_line_=0)` | `Indent(cs, cs->level)` | `<func\n`（**无行号**） |
| `DBUG_PRINT`（开 `t`） | `_db_doprnt_` | `Indent(cs, cs->level + 1)` | `keyword: <正文>\n` |
| `DBUG_PRINT`（未开 `t`） | `_db_doprnt_` | **无缩进** | `func: keyword: <正文>\n` |
| `DBUG_DUMP` | `_db_dump_` | 同 DBUG_PRINT | `keyword: Memory: 0x... Bytes: (N)\n` + 每行 80 列大写 hex |

**★ 输出示例（由上述格式串严格推导）**：

```
# --debug="d:t"（仅 trace，无前缀选项）
>main
| >do_command
| | info: command: SELECT 1
| | >dispatch_command
| | | >mysql_execute_command
| | | | info: table: t1
| | | <mysql_execute_command 4210
| | <dispatch_command 1980
<main 999
```

```
# --debug="d:t:i:F:L:N:P:T"（全前缀打开，level=3 的 >do_command）
T@4:    12: 14:22:33.123456: mysqld:   sql_parse.cc:  1234:    3: | | >do_command
└─'i'  └─'N'  └─────'T'─────┘  └─'P'─┘  └──'F'───┘  └─'L'─┘  └'n'┘  └缩进┘
```

```
# DBUG_DUMP("blob", ptr, 8)
| | | blob: Memory: 0x7f8e2c004000  Bytes: (8)
| | | 41 42 43 44 45 46 47 48
```

**★ 输出到 stderr 还是文件的分派**（★ 全局只有一个出口 `cs->stack->out_file`）：

```cpp
/* ① 默认 stderr（code_state() 里） */
    memset(&init_settings, 0, sizeof(init_settings));
    init_settings.out_file = stderr;      /* ★ 分派点 1 */
    init_settings.flags = OPEN_APPEND;

/* ② "o" 不带文件名 → "-" → stdout */
        if (control != end) DBUGOpenFile(cs, control, end, stack->flags & OPEN_APPEND);
        else                DBUGOpenFile(cs, "-", nullptr, 0);

/* ③ 真正打开文件 */
static void DBUGOpenFile(CODE_STATE *cs, const char *name, const char *end, int append) {
  ...
    if (strcmp(name, "-") == 0) {
      cs->stack->out_file = stdout;       /* ★ 分派点 2 */
      cs->stack->flags |= FLUSH_ON_WRITE;
      cs->stack->name[0] = 0;
    } else {
      if (!Writable(name)) { ... }        /* ★ suid 安全：用真实 uid 查写权限 */
      else if (!(fp = fopen(name, append ? "a+" : "w"))) { ... }
      else cs->stack->out_file = fp;      /* ★ 分派点 3 */
    }
}
```

> ★ **`o`/`O` 用 `"w"`（截断）、`a`/`A` 用 `"a+"`（追加）**——这就是 MTR 用 `A` 而不是 `O` 的原因：多次重启 trace 能累积。
> ★ `DBUGCloseFile()` 永不关 stderr/stdout。
> ★ `Writable()` 的注释解释得很清楚：*"Because the debugger might be linked in with a program that runs with the set-uid-bit (suid) set, we have to be careful about opening a user named file"*——防提权写文件。

---

## 调试模式控制

### 控制字符串解析：`DbugParse()`

**★ `DbugParse()` 是 static 的**（⚠️ **不存在 `dbug_parse()`**），它是控制串语法的"事实文档"（源码注释明写 *"See the manual for the list of supported signs, flags, and modifiers"*，语法只隐含在它的 switch 里）：

```cpp
/*
 *      The debug control string is a sequence of colon separated fields
 *      as follows:
 *
 *              [+]<field_1>:<field_2>:...:<field_N>
 *
 *      Each field consists of a mandatory flag character followed by
 *      an optional "," and comma separated list of modifiers:
 *
 *              [sign]flag[,modifier,modifier,...,modifier]
 *
 *      For convenience, any leading "-#" is stripped off.
 *
 *  RETURN
 *      1 - a list of functions ("f" flag) was possibly changed
 *      0 - a list of functions was not changed
 */
static int DbugParse(CODE_STATE *cs, const char *control) {
  ...
  assert(!cs->locked);                    /* ★ 禁止在 DBUG_LOCK_FILE 区间内改控制串 */
  if (stack == &init_settings) native_rw_wrlock(&THR_LOCK_init_settings);
  /* ★★ 改全局 → 写锁 */

  if (control[0] == '-' && control[1] == '#') control += 2;   /* 剥掉历史前缀 "-#" */

  rel = control[0] == '+' || control[0] == '-';
  /* ★ rel = 增量模式：以 +/- 开头表示"在现有 settings 上叠加/删减" */
  if ((!rel || (!stack->out_file && !stack->next))) {
    FreeState(cs, stack, 0);              /* 非增量：彻底重置 */
    stack->flags = 0; ... stack->out_file = stderr; ...
  } else if (!stack->out_file) {
    stack->flags = stack->next->flags;    /* 增量 + 新 settings 未初始化 → 从父层继承 */
    ...
    if (stack->next == &init_settings) {
      native_rw_rdlock(&THR_LOCK_init_settings);
      /*
        Never share with the global parent - it can change under your feet.
        Reset out_file to stderr to prevent sharing of trace files between
        global and session settings.
      */
      stack->out_file = stderr;           /* ★★ 绝不与全局共享 FILE* */
      stack->functions = ListCopy(init_settings.functions);   /* ★ 深拷贝 */
      ...
    }
  }

  end = DbugStrTok(control);              /* ★ 按 ':' 切字段 */
  while (control < end) {
    int c, sign = (*control == '+') ? 1 : (*control == '-') ? -1 : 0;
    if (sign) control++;
    c = *control++;
    if (*control == ',') control++;
    /* XXX when adding new cases here, don't forget _db_explain_ ! */
    switch (c) { ... 见下表 ... }
    if (!*end) break;
    control = end + 1;
    end = DbugStrTok(control);
  }
  ...
  return !rel || f_used;                  /* ★ 非增量 或 改过 f → 触发 FixTraceFlags */
}
```

**★ `DbugStrTok()`：为什么 `o,/tmp/a:b` 里的冒号不会被切断**：

```cpp
/*
 *     strtok lookalike - splits on ':', magically handles ::, :\ and :/
 */
static const char *DbugStrTok(const char *s) {
  while (s[0] &&
         (s[0] != ':' || (s[1] == '\\' || s[1] == '/' || (s[1] == ':' && s++))))
    s++;
  return s;
}
/* ★ ':' 后跟 '\' '/' 或另一个 ':'（路径/盘符/双冒号）时不视为分隔符 */
```

### ★ 全部控制标志（`DbugParse` 的 switch，逐个含义）

| 字符 | 作用对象 | 含义 |
|---|---|---|
| **`d`** | `keywords` + `DEBUG_ON` | 打开 debug。`d` 或 `d,k1,k2` = **只**输出这些关键字；**`+d`**（带 `+` 无修饰符）= 打开**全部**关键字（列表清空 = MATCHED = 全通过）；`-d,k` = 排除 k（负向列表）；`-d` 单独且到底 = 清空并关 `DEBUG_ON` |
| **`D`** | `delay` | 每行输出后延迟（`atoi` 值 / 10 秒） |
| **`f`** | `functions` | 需要 trace 的函数名；**以 `/` 结尾 = 整棵子调用树（`SUBDIR`）**；支持 glob |
| **`F`** | `FILE_ON` | 每行打印源文件名（basename） |
| **`i`** | `PID_ON` | 每行打印线程 id（`T@N:`） |
| **`L`** | `LINE_ON` | 每行打印源码行号 |
| **`n`** | `DEPTH_ON` | 每行打印嵌套层级 |
| **`N`** | `NUMBER_ON` | 每行打印自增序号 |
| **`o`** | `out_file` | 打开 trace 文件（**截断** `"w"`），不每次 flush |
| **`O`** | `out_file` + `FLUSH_ON_WRITE` | 同 `o` 但**每行后 fflush**（crash 时 trace 不丢） |
| **`a`** | `out_file` + `OPEN_APPEND` | **追加**打开（`"a+"`），不 flush |
| **`A`** | `out_file` + 两者 | 追加 + 每行 flush（**MTR 默认用它**） |
| **`p`** | `processes` | 只对指定进程名生效 |
| **`P`** | `PROCESS_ON` | 每行打印进程名 |
| **`r`** | `sub_level` | 把当前深度记为缩进基准（`sub_level = cs->level`） |
| **`t`** | `maxdepth` + `TRACE_ON` | 打开 trace；裸 `t` = `MAXDEPTH`(**200**)；`t,N` = 深度 N（相对模式是 `+=`/`-=`） |
| **`T`** | `TIMESTAMP_ON` | 每行打印 `HH:MM:SS.uuuuuu` |

> ⚠️ **不存在的标志**：**`S`**（malloc 检查，只在文件头历史注释里被提到，switch 中**无 case**）、**`g`**（只出现在 `_db_explain_` 的输出侧回显 `p_functions`，**解析器不认**）。

**★ 标志位定义**（`dbug.cc:130`）：

```cpp
#define DEBUG_ON (1 << 1)        /* Debug enabled */
#define FILE_ON (1 << 2)         /* File name print enabled */
#define LINE_ON (1 << 3)         /* Line number print enabled */
#define DEPTH_ON (1 << 4)        /* Function nest level print enabled */
#define PROCESS_ON (1 << 5)      /* Process name print enabled */
#define NUMBER_ON (1 << 6)       /* Number each line of output */
#define PROFILE_ON (1 << 7)      /* Print out profiling code */
#define PID_ON (1 << 8)          /* Identify each line with process id */
#define TIMESTAMP_ON (1 << 9)    /* timestamp every line of output */
#define FLUSH_ON_WRITE (1 << 10) /* Flush on every write */
#define OPEN_APPEND (1 << 11)    /* Open for append      */
#define TRACE_ON ((uint)1 << 31) /* Trace enabled. MUST be the highest bit!*/

#define TRACING (cs->stack->flags & TRACE_ON)
#define DEBUGGING (cs->stack->flags & DEBUG_ON)
#define PROFILING (cs->stack->flags & PROFILE_ON)
```

> ★★ **`TRACE_ON` 是最高的 bit31**，因为它被**复用**存进 `_db_stack_frame_::level` 的最高位，用来表示"这一帧要不要 trace"。

### 三个入口：`_db_set_` / `_db_push_` / `_db_set_init_`

```cpp
void _db_set_(const char *control) {        /* ★ SET SESSION debug 走这里 */
  ...
  if (cs->stack == &init_settings) PushState(cs);
  /* ★★ 若还指向全局，先压出一层线程私有 settings，避免污染全局 */
  if (DbugParse(cs, control)) {
    read_lock_stack(cs); FixTraceFlags(old_fflags, cs); unlock_stack(cs);
    /* ★ 中途改 -#d,... 时，已铺开的调用栈上每帧的 TRACE_ON 位要重算 */
  }
}

void _db_push_(const char *control) {       /* ★ 客户端/工具走这里：无条件压栈 */
  ... PushState(cs); ...
}

void _db_set_init_(const char *control) {   /* ★ SET GLOBAL debug / mysqld --debug */
  CODE_STATE tmp_cs;
  memset(&tmp_cs, 0, sizeof(tmp_cs));
  tmp_cs.stack = &init_settings;            /* ★ 伪造"当前就是全局"的临时 CODE_STATE */
  tmp_cs.process = db_process ? db_process : "dbug";
  DbugParse(&tmp_cs, control);
}

int _db_is_pushed_() {
  return (cs->stack != &init_settings);     /* ★ 栈顶不是全局 = 本会话 push 过 */
}
```

```cpp
/* settings 栈：头插法，链尾恒为全局 init_settings */
static void PushState(CODE_STATE *cs) {
  struct settings *new_malloc = (struct settings *)DbugMalloc(sizeof(struct settings));
  memset(new_malloc, 0, sizeof(struct settings));   /* ★ out_file=NULL 是"未初始化"哨兵 */
  new_malloc->next = cs->stack;
  cs->stack = new_malloc;
}
```

```cpp
void _db_pop_() {
  discard = cs->stack;
  if (discard != &init_settings) {          /* ★★ 弹到只剩全局就拒绝再弹 */
    ...
    cs->stack = discard->next;
    FreeState(cs, discard, 1);
    ...
  }
}
```

> ★ `_db_pop_()` 的注释解释了这个保护：*"Note that the pop will fail if it would remove the last valid settings from the stack. This prevents user errors in the push/pop sequence from screwing up the debugger."*
> ★ `FreeState()` 里的 `is_shared()` 判断：与下一层共享的列表/FILE* **不许释放**。

---

## 核心调用栈

```
DBUG_PRINT("info", ("fmt", x))
  ├─ _db_pargs_(__LINE__, "info")            → cs->u_line / cs->u_keyword
  └─ _db_enabled_()                           → if (!DEBUGGING) return 0;   ★脏读
       └─ _db_keyword_(cs, kw, 0)
            ├─ DoTrace(cs)                    → 深度 / 进程 / 函数 三闸
            └─ InList(keywords, kw)           → fnmatch(glob)
  └─ _db_doprnt_("fmt", x)                    → if (!DEBUGGING) return;      ★脏读
       ├─ read_lock_stack(cs)                 → 仅全局 settings 才加锁
       ├─ native_mutex_lock(&THR_LOCK_dbug)   → 若未在 DBUG_LOCK_FILE 区间
       ├─ DoPrefix(cs, cs->u_line)            → i/N/T/P/F/L/n 前缀
       ├─ Indent(cs, cs->level + 1)           → "| | " 竖线台阶
       ├─ fprintf(out_file, "%s: ", u_keyword)
       ├─ DbugVfprintf(out_file, fmt, args)   → vfprintf + 自动补 '\n'
       └─ DbugFlush(cs)                       → FLUSH_ON_WRITE 才 fflush；然后 unlock
```

```
DBUG_TRACE  →  AutoDebugTrace ctor  →  _db_enter_(PRETTY_FUNCTION, __FILE__, __LINE__, &frame)
                                          ├─ frame.level = ++cs->level | framep_trace_flag(...)
                                          ├─ DoTrace → ENABLE/DO/DISABLE/DONT
                                          └─ fprintf(">%.*s\n")  + DbugFlush
              ... 函数体 ...
              AutoDebugTrace dtor   →  _db_return_(0, &frame)   ★ _line_ == 0
                                          └─ fprintf("<%.*s\n")   ★ 无行号
```

```
SET SESSION debug='d:t:o,/tmp/x'
  sql_parse.cc → sql_set_variables() → set_var::check() → sys_var::update()
    → Sys_var_dbug::session_update      （sql/sys_vars.h:1254）
      → DBUG_SET(val) → _db_set_ → PushState + DbugParse → DBUGOpenFile / ListAdd
      → 之后每个 DBUG_PRINT 就写进 /tmp/x

SET GLOBAL debug='...'  → Sys_var_dbug::global_update → DBUG_SET_INITIAL → _db_set_init_
                                                                          （改全局 init_settings）
SET debug = DEFAULT     → DBUG_POP()
```

> ⚠️ **不存在 `Debug::print()` 这一层**——`my_dbug.h:181` 的宏直接调用 `_db_pargs_` / `_db_enabled_` / `_db_doprnt_` 三个自由函数。

---

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `debug` | 空 | **Session**（可 `SET GLOBAL`） | 控制 DBUG 开关与模式 |
| `debug_sync` | NULL | **ONLY_SESSION**（无 GLOBAL、无命令行） | debug sync 动作串（见下） |

**★ `debug` 系统变量的真名是 `Sys_var_dbug`**（⚠️ **不是 `Sys_debug`，也不在 `set_var.cc`**）：

```cpp
/* sql/sys_vars.h:1216 —— 整块包在 #ifndef NDEBUG 里 */
#ifndef NDEBUG
/**
  @@session.dbug and @@global.dbug variables.

  @@dbug variable differs from other variables in one aspect:
  if its value is not assigned in the session, it "points" to the global
  value, and so when the global value is changed, the change
  immediately takes effect in the session.

  This semantics is intentional, to be able to debug one session from
  another.
*/
class Sys_var_dbug : public sys_var {
  bool session_update(THD *, set_var *var) override {
    const char *val = var->save_result.string_value.str;
    if (!var->value) DBUG_POP();          /* ★ SET debug = DEFAULT → 弹掉会话私有层 */
    else             DBUG_SET(val);       /* ★ SET SESSION debug → _db_set_ */
    return false;
  }
  bool global_update(THD *, set_var *var) override {
    DBUG_SET_INITIAL(var->save_result.string_value.str);   /* ★ 改全局 init_settings */
    return false;
  }
  const uchar *session_value_ptr(THD *running_thd, THD *, std::string_view) override {
    char buf[512];
    DBUG_EXPLAIN(buf, sizeof(buf));       /* ★★ SELECT @@debug 读到的是"反解析"结果 */
    return (uchar *)running_thd->mem_strdup(buf);
  }
  const uchar *global_value_ptr(THD *thd, std::string_view) override {
    char buf[512];
    DBUG_EXPLAIN_INITIAL(buf, sizeof(buf));
    return (uchar *)thd->mem_strdup(buf);
  }
};
#endif
```

```cpp
/* sql/sys_vars.cc:2087 —— 实例定义 */
#ifndef NDEBUG
static Sys_var_dbug Sys_dbug("debug", "Debug log", sys_var::SESSION,
                             CMD_LINE(OPT_ARG, '#'), DEFAULT(""),
                             NO_MUTEX_GUARD, NOT_IN_BINLOG,
                             ON_CHECK(check_session_admin_for_debug));
#endif
```

> ★★★ **release 下 `debug` 变量根本不存在**——类与实例都整块包在 `#ifndef NDEBUG` 里，`SET debug` 会报 `ERROR 1193 Unknown system variable 'debug'`。
> ★ 需要 **`SESSION_VARIABLES_ADMIN` 或 `SUPER`** 权限（`ON_CHECK(check_session_admin)`）。
> ★★ **`SELECT @@debug` 读到的不是你 SET 进去的原串**，而是 `_db_explain_()` 把当前 settings **反向序列化**成的控制串（比如 `SET debug='+d,foo'` 之后可能回显成 `d,foo:t:...`）。

**★ `--debug` 命令行选项是"自动注册"的，不是手写的**（⚠️ `mysqld.cc` 里**没有** `"debug"` 选项条目）：

```cpp
/* sql/set_var.h:334 —— sys_var 自动进命令行 */
  bool is_settable_at_command_line() { return option.id != -1; }
  bool register_option(std::vector<my_option> *array, int parse_flags) {
    return is_settable_at_command_line() && (m_parse_flag & parse_flags) &&
           (array->push_back(option), false);
  }

/* sql/set_var.cc:166 */
int sys_var_add_options(std::vector<my_option> *long_options, int parse_flags) {
  for (sys_var *var = all_sys_vars.first; var; var = var->next) {
    if (var->register_option(long_options, parse_flags)) goto error;
  }
  return 0;
}

/* sql/mysqld.cc:11045 —— "debug" 就是在这里被拼进 all_options 的 */
  sys_var_add_options(&all_options, sys_var::PARSE_NORMAL);
```

```cpp
/* sql/mysqld.cc:10417 —— '#' 就是 --debug 的 short id，启动时唯一的 DBUG 初始化点 */
  switch (optid) {
    case '#':
#ifndef NDEBUG
      DBUG_SET_INITIAL(argument ? argument : default_dbug_option);
#endif
      opt_endinfo = true;
      break;
```

```cpp
/* sql/mysqld.cc:10267 —— 裸 --debug 的默认控制串 */
  default_dbug_option =
      IF_WIN("d:t:i:O,\\mysqld.trace", "d:t:i:o,/tmp/mysqld.trace");
```

> ⚠️ **`MYSQL_DEBUG` 环境变量对 mysqld 无效**——全树唯一的 `getenv("MYSQL_DEBUG")` 在 **`libmysql/libmysql.cc:244`**（客户端库），`sql/mysqld.cc` 里没有。

**★ `DBUG_PUSH` / `DBUG_SET_INITIAL` 的调用点分布**：

| 宏 | 调用点 |
|---|---|
| `DBUG_SET_INITIAL` | **仅 2 处**：`sql/mysqld.cc:10420`（启动 `--debug`）、`sql/sys_vars.h:1264`（`SET GLOBAL debug`） |
| `DBUG_PUSH` | **`sql/` 下没有**；只在客户端/工具：`comp_err.cc`、`my_print_defaults.cc`、`innochecksum.cc`、`ibd2sdi.cc`、`mysqlslap.cc`、`mysqldump.cc`、`sql-common/client.cc`、`libmysql/libmysql.cc` |
| `DBUG_POP` | **仅 1 处**：`sql/sys_vars.h:1257`（`SET debug = DEFAULT`） |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| 无 | DBUG 不暴露状态变量；观测靠 trace 文件本身 |

---

## debug_sync：与 DBUG 协作的另一套机制

**★★ 两套独立机制**（⚠️ `debug_sync` **不是** DBUG 的子功能——`sql/debug_sync.cc:324` 的小标题原文就是 *"=== **Co-work with** the DBUG facility ==="*）：

| | DBUG | debug_sync |
|---|---|---|
| 开关 | `NDEBUG` + 运行时 `DEBUG_ON` | 编译期 `ENABLED_DEBUG_SYNC` + 运行时 `opt_debug_sync_timeout` |
| 用途 | 打印 / 追踪 / 注入 | 多线程**时序编排**（signal / wait_for） |
| 耦合 | 借道 `DBUG_PRINT` 写 trace | 是 |

```cpp
/* sql/debug_sync.h:46 —— ★ 不存在 include/mysql/service_debug_sync.h */
#if defined(ENABLED_DEBUG_SYNC)
#define DEBUG_SYNC(_thd_, _sync_point_name_)                 \
  do {                                                       \
    if (unlikely(opt_debug_sync_timeout))                    \
      debug_sync(_thd_, STRING_WITH_LEN(_sync_point_name_)); \
  } while (0)
/* ★★ 全局开关是 opt_debug_sync_timeout(uint)，为 0 时整个 facility 关闭 ⇒ 线上零开销 */

extern MYSQL_PLUGIN_IMPORT uint opt_debug_sync_timeout;
#define DEBUG_SYNC_DEFAULT_WAIT_TIMEOUT 300
#else
#define DEBUG_SYNC(_thd_, _sync_point_name_) /* disabled DEBUG_SYNC */
#endif
```

```cmake
/* CMakeLists.txt:1505 —— 只在 Debug 构建加 */
FOREACH(LANG C CXX)
  STRING_PREPEND(CMAKE_${LANG}_FLAGS_DEBUG "-DENABLED_DEBUG_SYNC ")
  STRING_PREPEND(CMAKE_${LANG}_FLAGS_DEBUG "-DSAFE_MUTEX ")
ENDFOREACH()

/* sql/CMakeLists.txt:638 —— 非 debug 直接从源列表剔除 debug_sync.cc */
  IF(NOT CMAKE_CXX_FLAGS MATCHES "DENABLED_DEBUG_SYNC" AND ...)
    LIST(REMOVE_ITEM SQL_SHARED_SOURCES debug_sync.cc)
  ENDIF()
```

**★ 形式化语法（`sql/debug_sync.cc:176` 的权威注释）**：

```
  === Formal Syntax ===
      {RESET |
       <sync point name> TEST |
       <sync point name> CLEAR |
       <sync point name> {{SIGNAL <signal name>[, <signal name>]* |
                           WAIT_FOR <signal name> [TIMEOUT <seconds>]
                           [NO_CLEAR_EVENT]}
                          [EXECUTE <count>] &| HIT_LIMIT <count>}}
```

> ★ **语法是"顺序敏感"的**：`SIGNAL` → `WAIT_FOR`[→`TIMEOUT`] → `EXECUTE` → `NO_CLEAR_EVENT` → `HIT_LIMIT`，顺序错了报 `Illegal or out of order stuff`。
> ★ `SIGNAL` 隐含 `EXECUTE 1`；`WAIT_FOR` 隐含 `EXECUTE 1` + `TIMEOUT opt_debug_sync_timeout` + `clear_event=true`。
> ★ 特殊同步点 **`now`**：`SET DEBUG_SYNC='now SIGNAL s'` 会在 SET 语句**本身**立刻执行。
> ★ `WAIT_FOR` 超时只发 **WARNING**（`ER_DEBUG_SYNC_TIMEOUT`），不会让语句失败——但会污染 MTR 的 `.result`。

```cpp
struct st_debug_sync_action {
  ulong activation_count = 0; /* max(hit_limit, execute) */
  ulong hit_limit = 0;        /* hits before kill query */
  ulong execute = 0;          /* executes before self-clear */
  ulong timeout = 0;          /* wait_for timeout */
  String signal;              /* signal to emit */
  String wait_for;            /* signal to wait for */
  String sync_point;          /* sync point name */
  bool need_sort = false;
  bool clear_event = false;   /* ★ NO_CLEAR_EVENT → false，信号不被消费，可广播给多等待者 */
};
```

**★ 命中判定是二分查找**（`debug_sync_find()`，按 `(name_len, name)` 排序）：

```cpp
void debug_sync(THD *thd, const char *sync_point_name, size_t name_len) {
  ...
  DBUG_PRINT("debug_sync_point", ("hit: '%s'", sync_point_name));
  /* ★★ 这就是 grep trace 时的 "|debug_sync_point:" 关键字 */
  ds_control->dsp_hits++;
  if (ds_control->ds_active &&
      (action = debug_sync_find(ds_control->ds_action, ds_control->ds_active,
                                sync_point_name, name_len)) &&
      action->activation_count) {
    debug_sync_execute(thd, action);
    ds_control->dsp_executed++;
    if (!action->activation_count) debug_sync_remove_action(ds_control, action);
  }
}
```

**★ 两套机制真正接上的地方**：

```cpp
/* sql/debug_sync.h:75 —— 把 DBUG_EXECUTE_IF 与 debug_sync_set_action 粘一起 */
#define DBUG_SIGNAL_WAIT_FOR(T, A, B, C)                     \
  DBUG_EXECUTE_IF(A, {                                       \
    const char act[] = "now SIGNAL " B " WAIT_FOR " C;       \
    assert(!debug_sync_set_action(T, STRING_WITH_LEN(act))); \
  };)

/* sql/debug_sync.cc:1985 —— 由 @@debug='d,syncpoint_NAME' 激活 */
void conditional_sync_point(std::string name) {
  const std::string debug_symbol = "syncpoint_" + name;
  DBUG_EXECUTE_IF(debug_symbol.c_str(), {
    std::string act = "now SIGNAL reached_" + name + " WAIT_FOR continue_" + name;
    bool ret = debug_sync_set_action(current_thd, act.c_str(), act.length());
    assert(!ret);
  });
}
```

```cpp
/* sql/sql_parse.cc:3751 —— 崩溃注入的服务端接应点 */
      const bool force_server_crash_dbug = false;
      DBUG_EXECUTE_IF("crash_now", assert(force_server_crash_dbug););
      /* ★ 用 assert(常量false) 制造 core；release 下被 NDEBUG 吃掉 */
      DBUG_EXECUTE_IF("crash_now_safe", DBUG_SUICIDE(););
      /* ★ SIGKILL 自杀，不产生 core */
```

**★ `DBUG_EXECUTE_IF` 的八类真实用例**（★ 展示"注入点"的多样性）：

```cpp
/* ① 联动开启另一个关键字（sql/log.cc:835） */
    DBUG_EXECUTE_IF("simulate_slow_log_write_error",
                    { DBUG_SET("+d,simulate_file_write_error"); });

/* ② 覆盖局部变量触发边界/溢出（sql/log.cc:1051） */
    DBUG_EXECUTE_IF("slow_log_table_max_rows_examined", {
      query_utime = (longlong)1555826389LL * 1000000 + 1;
      lock_utime = query_utime;
    });
    DBUG_EXECUTE_IF("slow_log_table_max_rows_examined",
                    { rows_examined = 4294967294LL; });  // overflow 4-byte int

/* ③ 随机概率失败（sql/binlog.cc:2448） */
    DBUG_EXECUTE_IF("simulate_binlog_flush_error", {
      if (rand() % 3 == 0) { thd->commit_error = THD::CE_FLUSH_ERROR; }
    };);

/* ④ 崩溃注入（sql/binlog.cc:5013 / rpl_replica.cc:5770） */
    DBUG_EXECUTE_IF("crash_create_non_critical_before_update_index", DBUG_SUICIDE(););
    DBUG_EXECUTE_IF("crash_before_rotate_relaylog", DBUG_SUICIDE(););

/* ⑤ DBUG_EVALUATE_IF 做条件短路（sql/binlog.cc:1988） */
    if (DBUG_EVALUATE_IF("fail_to_open_log_file", true, false) ||
        DBUG_EVALUATE_IF("fail_to_read_index_file", true, false)) {

/* ⑥ 触发一次后自我摘除关键字（sql/mdl.cc:3381）—— ★ 防重复/递归触发 */
  DBUG_EXECUTE_IF("bug34594035_fail_acl_cache_lock",
                  debug_sync(get_thd(), "123", 3);
                  mysql_prlock_wrlock(&m_LOCK_waiting_for);
                  mysql_prlock_unlock(&m_LOCK_waiting_for);
                  DBUG_SET("-d,bug34594035_fail_acl_cache_lock"); return true;);

/* ⑦ 模拟外部库整个返回值族（sql/hostname_cache.cc:523） */
  DBUG_EXECUTE_IF("getnameinfo_error_noname",  { strcpy(hostname_buffer, "<garbage>"); err_code = EAI_NONAME; });
  DBUG_EXECUTE_IF("getnameinfo_fake_ipv4",     { strcpy(hostname_buffer, "santa.claus.ipv4.example.com"); err_code = 0; });
  DBUG_EXECUTE_IF("getnameinfo_fake_max_length_plus_1", { std::string s(HOSTNAME_LENGTH + 1, 'a'); ... });

/* ⑧ 内嵌断言 + 内嵌 DBUG_PRINT（sql/binlog.cc:2549） */
  DBUG_EXECUTE_IF("simulate_commit_failure", {
    XID_STATE *xs = thd->get_transaction()->xid_state();
    assert((thd->is_error() && xs->get_state() == XID_STATE::XA_IDLE) ||
           xs->get_state() == XID_STATE::XA_NOTR);
  });
```

> ★ 用例 ⑥ 的 `DBUG_SET("-d,xxx")` 自我摘除是很常见的模式——避免注入点在循环/递归里被反复触发。
> ★ MTR 里的用法（`mysql-test/t/dd_debug.test`）：`SET GLOBAL DEBUG='+d,weak_object_impl_store_fail_before_store_children';`，用 `-d` 关掉。

### InnoDB 的 `ut_a` / `ut_ad` / `ut_d`

```cpp
/* storage/innobase/include/ut0dbg.h:44 */
#define ut_a(EXPR)                                        \
  do {                                                    \
    if (unlikely(false == (bool)(EXPR))) {                \
      ut_dbg_assertion_failed(#EXPR, __FILE__, __LINE__); \
    }                                                     \
  } while (0)
/* ★★ ut_a 与 NDEBUG 无关，release 版也成立（InnoDB 的"永远断言"） */

#ifdef UNIV_DEBUG
#define ut_ad(EXPR) ut_a(EXPR)      /* Debug assertion */
#define ut_d(EXPR)  EXPR            /* Debug statement */
#define ut_o(EXPR)                  /* Opposite of ut_d */
#else
#define ut_ad(EXPR)
#define ut_d(EXPR)
#define ut_o(EXPR)  EXPR
#endif
/* ★★ 开关是 UNIV_DEBUG，不是 NDEBUG / 不是 DBUG —— 完全解耦 */

#ifdef UNIV_DEBUG
#define DBUG_INJECT_CRASH(prefix, count)            \
  do {                                              \
    char buf[64];                                   \
    snprintf(buf, sizeof buf, prefix "_%u", count); \
    DBUG_EXECUTE_IF(buf, DBUG_SUICIDE(););          \
  } while (0)
/* ★★ 唯一真正搭上 DBUG 的地方：关键字是运行期拼的 "prefix_N"，
        测试用 SET GLOBAL debug='+d,prefix_3' 精准注入第 N 次崩溃 */
#endif
```

### MTR 侧的 `--debug`

```perl
# mysql-test-run.pl:185 —— 默认只开 "d"，不开 t，所以默认没有 trace 文件
my $debug_d = "d";

# :2311 —— --debug-common 把常用关键字一次性打开
  if ($opt_debug_common) {
    $opt_debug = 1;
    $debug_d = "d,query,info,error,enter,exit";
  }

# :6489 —— mysqld 的 trace 落在 var/log/mysqld.1.trace
  if ($opt_debug) {
    mtr_add_arg($args, "--debug=$debug_d:t:i:A,%s/log/%s.trace",
                $path_vardir_trace, $mysqld->name());
  }                             /* ★ 用 A（追加+每行 flush），不用 O */

# :4235 —— bootstrap 单独一份
    mtr_add_arg($args, "--debug=$debug_d:t:i:A,%s/log/bootstrap.trace", $path_vardir_trace);

# :2796 —— 客户端用 --loose-debug（release 版也能启动）
    mtr_add_arg($args, "--loose-debug=$debug_d:t:A,%s/log/%s.trace", ...);

# :6342 —— debug sync 的超时
  mtr_add_arg($args, "--loose-debug-sync-timeout=%s", $opt_debug_sync_timeout)
    unless $opt_user_args;      /* ★ 默认 600；valgrind ×10、sanitize ×2 */
```

> ★ `debug_sync.cc:331` 给出的实用 grep：
> ```sh
> egrep 'query:|debug_sync_exec:' mysql-test/var/log/mysqld.1.trace
> # 想看"命中了哪些同步点"则再加 "|debug_sync_point:"
> ```

---

## Misc

### InnoDB 的 trace

- `UNIV_DEBUG` / `ut_d` / `ut_a`：InnoDB 自己的调试断言编译开关，非 DBUG
- `innodb_monitor` / `INNODB_METRICS` 表：InnoDB 运行状态输出
- 注意区分：DBUG trace（函数调用追踪） vs InnoDB 断言（`ut_a`） vs InnoDB 状态输出（`SHOW ENGINE INNODB STATUS`）

### 内存破坏哨兵（canary 思想）

MySQL 源码字面没有 `canary` 标识符，但用 magic 值实现"内存破坏检测哨兵"（canary 思想）：

- `THD_SENTRY_MAGIC 0xfeedd1ff` / `THD_SENTRY_GONE 0xdeadbeef`（`sql/sql_class.h:338`）：`THD::dbug_sentry`（注释 "watch out for memory corruption"，`sql_class.h:1499`）在 debug 版构造函数置 MAGIC（`sql_class.cc:759`）、析构置 GONE（`sql_class.cc:1449`）；`THD_CHECK_SENTRY` 断言校验，检测 THD 结构体被越界写坏。
- InnoDB 内存块 `magic_n`（`mem0mem.h:407`）：`MEM_BLOCK_MAGIC_N 0x445566778899AABB` / `MEM_FREED_BLOCK_MAGIC_N 0xBBAA998877665544`；`mem_block_validate`（`mem0mem.ic:110`）校验失败抛 fatal "Memory block is invalid"；释放时改写（`memory.cc:430`）以抓 double-free / use-after-free。
- 动态数组块 `DYN_BLOCK_MAGIC_N 375767`（`dyn0types.h:41`）。
- 区分：`BINLOG_MAGIC`（`\xfe\x62\x69\x6e`）、`tc_log_magic`（`tc_log.cc:311`）是"格式标识魔数"，用于识别文件类型，非"被篡改即告警"的检测哨兵，语义不同于 canary。
- 栈保护（stack canary）由编译器 `-fstack-protector` 注入，主代码不显式写。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| **`include/my_dbug.h`**（365 行） | **全部 DBUG 宏**（debug 版 + release 空宏两套） |
| **`mysys/dbug.cc`**（2191 行） | 运行时库；整个实体代码包在 `#ifndef NDEBUG` 里 |
| **`sql/sys_vars.h:1216`** | `Sys_var_dbug` 类（`debug` 系统变量） |
| `sql/sys_vars.cc:2087` | `Sys_dbug` 实例 |
| `sql/mysqld.cc:10417` | `--debug` 的 `case '#'` → `DBUG_SET_INITIAL` |
| `sql/debug_sync.h:46` | `DEBUG_SYNC` 宏（**不是** `include/mysql/service_debug_sync.h`） |
| `sql/debug_sync.cc:176` | debug sync 的形式化语法注释 |
| `storage/innobase/include/ut0dbg.h:44` | `ut_a` / `ut_ad` / `ut_d` / `DBUG_INJECT_CRASH` |
| `mysys/my_thr_init.cc:65` | `THR_mysys`（唯一的 TLS 指针） |

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| DBUG 由 `DBUG_OFF` 控制 | **错**。真正的开关是标准 **`NDEBUG`**（`my_dbug.h:57`）。`DBUG_OFF` 全仓库仅 `storage/perfschema/pfs.cc:6324` 一处 `#ifndef`（遗留死代码，无任何定义） |
| `DBUG_ASSERT` 是 DBUG 家族成员 | **不存在**（仅 mysql-test 注释文字里遗留）。8.0 用原生 `assert()` |
| `DBUG_CHECK_CRASHED` / `DBUG_CRASH_ENTER` | **不存在**（0 命中）。崩溃类只有 `DBUG_SUICIDE()` |
| `DBUG_LEAVE` | **不存在**。退出侧只有 `DBUG_RETURN` / `DBUG_VOID_RETURN` / `~AutoDebugTrace()` |
| `DBUG_ENTER`/`DBUG_RETURN` 是主流写法 | **已被 `DBUG_TRACE` 取代**（695+ 文件 vs 13 个文件） |
| `DBUG_TRACE` 是 `DBUG_ENTER` 的别名 | **完全不同机制**。前者是 C++ **RAII 对象**（`const AutoDebugTrace _db_trace(...)`），**只在 `__cplusplus` 下存在**，C 代码不可用 |
| 宏里用了 `__builtin_expect` | **没有**。快路径靠运行时库里的**无锁脏读** `if (!DEBUGGING) return;` |
| `DBUG_PRINT` 有格式串白名单约束 | **没有**。真实约束只有两条：① 单次输出上限 **1024 字节**；② 格式串**不要自带 `\n`** |
| `CODE_STATE` 定义在 `my_dbug.h` | **只有前向声明**（`my_dbug.h:67`），真身在 `mysys/dbug.cc:217` |
| `command_names` 与枚举有 `static_assert` | **没有任何校验**，唯一约束是注释 `Must match the "command_names" array` |
| `include/dbug.h` / `mysys/dbug.h` | **都不存在**。只有 `include/my_dbug.h`（宏）+ `mysys/dbug.cc`（实现） |
| `dbug_parse()` | **不存在** → **`static int DbugParse()`**（`dbug.cc:440`） |
| 8.0 已重构成 `class Debug`（`Debug::enter/exit/print/setup`） | **不存在**（0 命中）。仍是 Fred Fish 的 C 风格：`_db_enter_` / `_db_return_` / `_db_doprnt_` |
| 有 `Indenter` 类（`append_indent` 等） | **不存在** → `static void Indent(CODE_STATE *, int)`（11 行，直接 `fputc`） |
| 有 `WriteBuffer` / `Path` / `Flush` / `Frame` / `State` 类 | **均不存在** → `Flush`=`DbugFlush()`；`Frame`=`_db_stack_frame_`；`State`=`CODE_STATE`+`settings`；`WriteBuffer`/`Path` 无任何对应物（直接 `fprintf`） |
| `class Debug` 有 `FILE *out_file` 成员 | 真名是 `struct settings::out_file`；`CODE_STATE` 只持有 `settings *stack` 指针 |
| 文件末尾有 `dbug_help` 讲控制串语法 | **不存在**（0 命中）。`dbug.cc` 到 2191 行 `#endif` 就结束；注释明写 `See the manual` |
| `DBUG_EXECUTE_IF` 由 `_db_execute_` 实现 | **不存在 `_db_execute_`** → 就是 `_db_keyword_(0, kw, **1**)`；与 `DBUG_EXECUTE` 唯一区别是第 3 参 `strict` |
| `_db_explain_` 是 EXECUTE_IF 的实现 | **不是**。它是把 settings **反向序列化成控制串**（给 `SELECT @@debug` 用） |
| 每线程用 `thread_local CODE_STATE` | 机制对、形式错。TLS 只有 `st_my_thread_var *THR_mysys` 一个**指针**，其中 `CODE_STATE *dbug` 槽位**首次使用时才 `DbugMalloc`** |
| `DBUG_PRINT` 一定调 `_db_doprnt_` | **不一定**。宏先 `_db_pargs_`，再用 `_db_enabled_()` 短路 |
| trace 深度无上限 | 有。默认 **`MAXDEPTH 200`**（`t` 不带数字即 200） |
| 缩进每级是 2 个空格 | 是 2 字符但形如 **`"| "`**（`count % INDENT == 0` 时输出 `\|`） |
| `debug` 系统变量的类是 `Sys_debug`、在 `set_var.cc` | **真名 `Sys_var_dbug`**，在 **`sql/sys_vars.h:1228`**；实例 `Sys_dbug` 在 `sql/sys_vars.cc:2087` |
| `mysqld --debug` 是 `mysqld.cc` 里手写的 `{"debug", '#', ...}` | **不是**。`mysqld.cc` 里**没有**该条目，由 `Sys_var_dbug` 的 `CMD_LINE(OPT_ARG,'#')` 经 `sys_var_add_options()` **自动注册** |
| `MYSQL_DEBUG` 环境变量能被 mysqld 读 | **不能**。唯一 `getenv("MYSQL_DEBUG")` 在 **`libmysql/libmysql.cc:244`**（客户端库） |
| 控制标志有 `S`（malloc 检查）和 `g`（profiling） | **`DbugParse` 里根本无 `case 'S'`**（只在文件头历史注释里）；**`g` 解析器不认**（只在 `_db_explain_` 输出侧） |
| `debug_sync` 是 DBUG 的子功能 | **两套独立机制**。源码小标题原文是 "**Co-work with** the DBUG facility"。开关是 `opt_debug_sync_timeout` |
| `DEBUG_SYNC` 宏在 `include/mysql/service_debug_sync.h` | **不存在该文件** → 在 **`sql/debug_sync.h:46`** |
| `Debug_sync_point` 类 | **不存在** → C 结构体 `st_debug_sync_action`；引擎真名 `static void debug_sync_execute()` |
| release 版也能 `SET debug` | **不能**。类与实例都整块包在 `#ifndef NDEBUG`，release 下报 `Unknown system variable 'debug'` |
| InnoDB `ut_ad`/`ut_d` 走 DBUG | 走的是 **`UNIV_DEBUG`**，不是 `NDEBUG`/DBUG。只有 `DBUG_INJECT_CRASH` 真调 `DBUG_EXECUTE_IF` |
| `SET debug='d:t:o,/tmp/x'` 是全局的 | `debug` 是 **`SESSION`** 作用域；session 走 `DBUG_SET`，global 走 `DBUG_SET_INITIAL` |
| `SET GLOBAL debug_sync` / `--debug-sync` | **不成立**。`ONLY_SESSION` + `NO_CMD_LINE` |
| `safe_process` 的 `Retry ... EAGAIN` | （MTR 侧）8.0.39 是无限重试不判 errno，该注释是老版本残留 |
| 命令总数"约 100 条" | `command_names` 中真实命令名 **99 个**；`enum enum_commands` 共 **103 个**值 |

**★ 三个"结构性事实"**

1. **栈帧寄生是 DBUG 最反直觉也最精巧的设计**——`_db_stack_frame_` 分配在**被跟踪函数自己的 C 栈**上，dbug 自己**不维护函数栈**。源码注释说得很清楚：*"this is why the ENTER macro must be the first 'executable' code in a function"*。代价是：漏写 `DBUG_RETURN`（或用了裸 `return`）会导致 `cs->framep != _stack_frame_`，直接 `DbugExit(ERR_MISSING_RETURN)` 中止；收益是：零额外内存分配、零内部栈维护、天然线程安全。`DBUG_TRACE`（RAII）把这个约束自动化了——这也是它取代 `DBUG_ENTER` 的根本原因。

2. **`strict` 参数的一个比特，划分了"打印"与"注入"两种语义**——`strict = strict ? INCLUDE : INCLUDE | MATCHED`。`DBUG_PRINT`/`DBUG_EXECUTE`/`DBUG_DUMP`（strict=0）在**关键字列表为空时也命中**，所以裸 `-d` 全开就能看到一切；`DBUG_EXECUTE_IF`/`DBUG_EVALUATE_IF`（strict=1）**必须被 `-d,kw` 显式包含**，所以默认永不触发。这正是一个比特撑起"MTR 故障注入点默认沉睡"的语义。

3. **debug_sync 与 DBUG 是"协作"而非"包含"**——二者开关完全独立（`opt_debug_sync_timeout` vs `DEBUG_ON`），唯一的耦合是 debug_sync 借道 `DBUG_PRINT("debug_sync*"...)` 写 trace，以及 `DBUG_SIGNAL_WAIT_FOR()` / `conditional_sync_point()` 把 `DBUG_EXECUTE_IF` 与 `debug_sync_set_action` 粘在一起。理解这一点才能明白为什么 `SET DEBUG_SYNC` 不需要先开 `debug`，以及为什么 `debug_sync` 超时产生的 WARNING 会污染 MTR 的 `.result`（因为它走的是 SQL warning 通道，不是 DBUG 通道）。

> 注：本工作副本含少量非上游改动（`mysys/dbug.cc:326/340`、`sql/sys_vars.cc:2076`、`sql/debug_sync.h:44`、`sql/debug_sync.cc:411` 的中文批注，以及 `sql/sys_vars.cc` 里 `SET debug` 时额外打一条 error log 的本地增强），已在文中标注。

---

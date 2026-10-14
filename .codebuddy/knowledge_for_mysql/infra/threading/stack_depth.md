# 栈深度治理深度剖析（thread_stack / C 栈 / 递归上限）

> 基于 MySQL **8.0.39** 源码，覆盖：操作系统层栈的建立机制（`execve` / `mmap` / `mprotect` / guard page）、`thread_stack` 系统变量的完整语义与生效链路、MySQL 的**软件栈检查**（`sql/check_stack.cc`）、**递归路径的检查点地图**、**栈与 MEM_ROOT 的分工**、**JSON 嵌套的容量分析**。
>
> **★ 边界**：本篇讲**栈这个横切主题本身**——栈从哪来、多大、怎么防溢出、哪些递归吃栈。
> 注：**软件栈检查（`check_stack_overrun`）是 server 层专属机制**，`storage/innobase/` 下 **0 处**——InnoDB 不做栈检查，它靠 `std::thread` 默认 ~8MB 栈 + 内部递归天然有界（B-tree 层数、dict 递归都有限）来规避。详见 [§四](#四-软件栈检查的作用范围server-层专属)。
> 线程本身的创建/停止见 [`thread_lifecycle.md`](thread_lifecycle.md)；THD 对象见 [`thd.md`](thd.md)；连接线程见 [`connection_threading.md`](connection_threading.md)。
> 本篇不深剖"某个 SQL 语义为什么那样做"，只深剖**它们共同踩的那块栈**。

## 目录

- [概述](#概述)
- [理论基础：栈是从哪来的](#理论基础栈是从哪来的)
- [核心实现：thread_stack 的完整语义](#核心实现thread_stack-的完整语义)
- [★ 软件栈检查：sql/check_stack.cc](#-软件栈检查sqlcheck_stackcc)
- [★ 检查点地图：哪些递归路径有保护](#-检查点地图哪些递归路径有保护)
- [★ 栈 vs MEM_ROOT：数据放哪 vs 调用多深](#-栈-vs-mem_root数据放哪-vs-调用多深)
- [★ JSON 嵌套的容量分析](#-json-嵌套的容量分析)
- [可观测性与调优](#可观测性与调优)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [易误判清单](#易误判清单)
- [Misc](#misc)

---

## 概述

### 一句话定位

MySQL **没有任何"表达式深度"的硬性限制**。一条 SQL 能嵌套多深，唯一的上限是**连接线程的 C 栈有多大**。这个上限由 `thread_stack` 决定，由 `sql/check_stack.cc` 的软件检查兜底，而**软件检查只在部分递归路径上布置了检查点**——没布置的路径会一路递归到 `pthread` 栈尾的 `PROT_NONE` 守护页，以 **SIGSEGV** 的形式崩掉。

### 三个必须先纠正的直觉

**直觉一：「`thread_stack` 默认很小，MySQL 靠 MEM_ROOT 所以不吃栈」** —— 错。默认其实是 **1MB**（`DEFAULT_THREAD_STACK`），而 MEM_ROOT 管的是**数据放哪**（堆），栈管的是**调用能钻多深**，两者同时消耗、互不替代。

**直觉二：「栈溢出 MySQL 会优雅报错」** —— 只对一半。有检查点的路径会报 `ER_STACK_OVERRUN_NEED_MORE`（"Thread stack overrun"）；**没检查点的路径直接 SIGSEGV**。`item_json_func.cc` 里 JSON 全部函数的求值递归，**一处检查都没有**。

**直觉三：「`thread_stack` 调小一点省内存，反正有保护」** —— 危险。保护是**按路径**布置的，不是全局的；而且 `thread_stack` 是 `READ_ONLY`，改错一次要重启才生效。

### 全景图

```
┌─ OS 层 ────────────────────────────────────────────────────┐
│ execve → 主线程栈（可增长，RLIMIT_STACK）                   │
│ mmap   → pthread 线程栈（固定，尾部 mprotect PROT_NONE）    │
└────────────────────────────────────────────────────────────┘
                          ↑
┌─ MySQL 层 ─────────────────────────────────────────────────┐
│ thread_stack（默认 1M，下限 128K，READ_ONLY）                │
│   → connection_attrib → pthread_attr_setstacksize           │
│   → 只作用于连接线程（InnoDB 用 std::thread，不受它管）      │
└────────────────────────────────────────────────────────────┘
                          ↑
┌─ 防护层 ───────────────────────────────────────────────────┐
│ check_stack_overrun（软件检查，提前 margin 拦住）            │
│   ✅ contextualize 递归：基类级全覆盖                        │
│   ❌ JSON 求值递归 / Item 基类递归：无检查点 → SIGSEGV       │
└────────────────────────────────────────────────────────────┘
```

---

## 理论基础：栈是从哪来的

本节是纯操作系统内容。MySQL 的 `thread_stack` 只是"给 pthread 传了个参数"，不理解 OS 层的机制，就无法判断一次 SIGSEGV 到底意味着什么。

### 一、`execve`：主线程栈的建立

进程启动时，内核执行 `execve()` 系统调用，加载 ELF 可执行文件并建立全新的用户态地址空间。其中栈是这样建立的：

```
execve() 内核动作（简化）：
  1. 解析 ELF 头，映射代码段/数据段
  2. 建立「用户栈」VMA（虚拟内存区域）
  3. 在栈顶（高地址）依次压入：
       - 命令行参数 argv[]
       - 环境变量 envp[]
       - 辅助向量 auxv（AT_PAGESZ / AT_RANDOM / AT_SECURE …）
  4. 设置 %rsp 指向栈顶，跳到入口
```

关键点是第 2 步：这块栈 VMA 被内核标记为**可向下扩展**（Linux 内部 `VM_GROWSDOWN` / `expand_downwards`）。这意味着主线程栈**不是一开始就分配好的固定大小**，而是一个"起点 + 可扩展区间"。

它的上限由 `RLIMIT_STACK` 资源限制决定（`ulimit -s`，Linux 默认 **8MB**）。

### 二、`mmap`：按需兑现的匿名映射

`mmap` 是"在虚拟地址空间里划一段区域"的系统调用：

```c
void *addr = mmap(NULL, length, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
```

| 参数 | 含义 |
|---|---|
| `MAP_ANONYMOUS` | **匿名映射**——不对应任何文件，内容全零（栈、堆都用它） |
| `MAP_PRIVATE` | 私有，写时复制，改动不共享 |
| `MAP_STACK` | 提示内核"这是栈"（Linux 2.6.34+，主要对部分架构有意义） |
| `PROT_READ \| PROT_WRITE` | 可读可写（**没有 PROT_EXEC**，栈默认不可执行） |

**核心概念：虚拟内存 vs 物理内存。** `mmap` 返回成功**只代表虚拟地址空间被划走了**，物理页一个都没分配。真正访问时触发**缺页异常**，内核才分配物理页并建立页表映射。

这就是"配 1MB 栈不意味着吃 1MB 内存"的原因——`memory_architecture.md` 里记的"线程栈：虚拟内存 1MB+，物理按需"，说的正是这个。**栈只有被实际用到的那部分才占物理内存。**

### 三、`pthread_create`：线程栈是 mmap 出来的

`pthread_create()` 创建线程时，glibc 内部（ `allocate_stack()`）做的事：

```c
// 简化自 glibc
void *stack = mmap(NULL, size + guardsize, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
mprotect(stack, guardsize, PROT_NONE);   // ← 把最低那一段设为不可访问
// 然后把 (stack + guardsize) 作为栈顶传给 clone()
```

所以线程栈**不在主线程栈的旁边**，而是在进程地址空间的 **mmap 区域**里单独划出来的一块。一个进程有 N 个线程，就有 N 块这样独立的 mmap 区域。

### 四、guard page 与 `mprotect(PROT_NONE)`

`mprotect(addr, len, prot)` 用来**修改一段已有映射的访问权限**。`prot` 可选：

| 标志 | 含义 |
|---|---|
| `PROT_READ` / `PROT_WRITE` / `PROT_EXEC` | 可读 / 可写 / 可执行 |
| **`PROT_NONE`** | **完全不可访问**——读、写、执行都会触发 SIGSEGV |

栈是**向下增长**的（x86 上 `%rsp` 递减），所以 glibc 在栈区的**最低地址端**预留一段（默认 **1 个页 = 4096 字节**，可用 `pthread_attr_setguardsize` 调整），`mprotect` 成 `PROT_NONE`。

**这块守护页的作用：让栈溢出立即、确定地暴露。** 栈指针一旦越过可用区间踩进守护页，立刻 SIGSEGV——而不是静默写进相邻的某个映射（另一个线程栈、堆、某个库的 data 段），那种情况会造成**难以定位的内存破坏**，而不是一个干净的 crash。

> 这是操作系统设计里"**快速失败优于静默破坏**"的典型应用。对数据库这种长跑进程，一次干净的 SIGSEGV + core dump 远比内存被悄悄踩坏要好诊断得多。

### 五、★ 为什么主线程栈能增长，线程栈不能

这是最容易被忽略、也最关键的一点：

| | 主线程栈 | pthread 线程栈 |
|---|---|---|
| 建立者 | `execve` 内核建立 | glibc `mmap` |
| VMA 标记 | **可向下扩展** | 普通匿名映射 |
| 能否自动增长 | ✅ 能（受 `RLIMIT_STACK` 约束） | ❌ **不能，固定大小** |
| 溢出后果 | 先自动扩展；超过 `RLIMIT_STACK` 才 SIGSEGV | **踩 guard page → 立即 SIGSEGV** |

**为什么线程栈不能扩展？** 因为线程栈是一块普通 mmap 区域，它的**紧邻下方是别的映射**（另一个线程的栈、堆、共享库……）。内核无法"就地扩展"——扩展就意味着踩到邻居。而主线程栈下方是内核专门保留的、尚未分配的空间，所以内核可以安全地向下推。

**后果**：主线程上一个 8MB 深度的递归可能"跑着跑着就成功了"（栈自动扩展），而同样的递归在连接线程上会**稳定 SIGSEGV**。这也是为什么 MySQL 必须自己实现软件栈检查——它不能指望 OS 兜底。

### 六、一张图总结

```
进程虚拟地址空间（x86_64，向下增长）

高地址
┌──────────────────────────────┐
│  内核空间（用户不可访问）      │
├──────────────────────────────┤
│  主线程栈                     │ ← execve 建立
│     已用部分                  │    VM_GROWSDOWN，可向下扩展
│        ↓                     │    上限 RLIMIT_STACK（默认 8MB）
│   （可扩展区间）              │    溢出先扩展，超了才 SIGSEGV
├──────────────────────────────┤
│  mmap 区域                    │
│  ┌────────────────────────┐  │
│  │ 线程 A 栈              │  │ ← mmap 固定大小
│  │   已用 ↓               │  │
│  │   可用                 │  │   = thread_stack + guardsize
│  ├────────────────────────┤  │
│  │ guard page (PROT_NONE) │  │ ← mprotect，4KB
│  └────────────────────────┘  │   踩到 → SIGSEGV，不扩展
│  ┌────────────────────────┐  │
│  │ 线程 B 栈 …            │  │
├──────────────────────────────┤
│  堆（brk / malloc）          │
├──────────────────────────────┤
│  共享库 / 数据段 / 代码段     │
低地址
```

---

## 核心实现：thread_stack 的完整语义

### 一、变量定义

`thread_stack` 对应的 C 变量是 `my_thread_stack_size`，定义在 `sql/sys_vars.cc`：

```cpp
static Sys_var_ulong Sys_thread_stack(
    "thread_stack", "The stack size for each thread",
    READ_ONLY GLOBAL_VAR(my_thread_stack_size), CMD_LINE(REQUIRED_ARG),
#if defined(__clang__) && defined(HAVE_UBSAN)
    // Clang with DEBUG needs more stack, esp. with UBSAN.
    VALID_RANGE(DEFAULT_THREAD_STACK, ULONG_MAX),
#else
    VALID_RANGE(128 * 1024, ULONG_MAX),
#endif
    DEFAULT(DEFAULT_THREAD_STACK), BLOCK_SIZE(1024));
```

| 属性 | 值 | 含义 |
|---|---|---|
| **默认值** | `DEFAULT_THREAD_STACK` = **1MB** | `include/my_thread.h`：`#define DEFAULT_THREAD_STACK (1024UL * 1024UL)` |
| **下限** | **128KB**（正常构建） | 低于此值启动即拒绝 |
| **下限（debug）** | **1MB**（clang + UBSAN） | 注释原话："Clang with DEBUG needs more stack, esp. with UBSAN." |
| **只读** | `READ_ONLY` | **运行时不能改，必须重启** |
| **粒度** | `BLOCK_SIZE(1024)` | 必须是 1024 的整数倍 |

**两个反直觉点：**

1. **下限只有 128KB**——比默认 1MB 低 8 倍。所以"把 `thread_stack` 调到 256K"是完全合法的配置，启动时不会有任何警告。这是很多线上事故的伏笔：配置者以为只是"省点内存"，实际是把递归深度上限砍到了 1/4。

2. **debug 构建禁止调小**——Clang + UBSAN 下 `VALID_RANGE` 下限就是 1MB。因为插桩版本每个栈帧大得多（UBSAN 会插入大量检查代码），栈吃得更凶。这也侧面说明：**栈的实际消耗随构建方式剧烈变化**，`thread_stack` 的"安全值"不是固定数字。

### 二、生效链路

```
thread_stack（配置文件）
   ↓  sys_vars.cc  Sys_thread_stack
my_thread_stack_size（全局变量）
   ↓  mysqld.cc  my_thread_attr_setstacksize(&connection_attrib, my_thread_stack_size + guardize)
connection_attrib（全局线程属性）
   ↓  传给 my_thread_create / mysql_thread_create
pthread_attr_setstacksize → pthread_create → glibc mmap + mprotect
```

`sql/mysqld.cc` 里的实际代码：

```cpp
size_t guardize = 0;
#ifndef _WIN32
  int retval = pthread_attr_getguardsize(&connection_attrib, &guardize);
  assert(retval == 0);
  if (retval == 0) { /* guardize 取到系统默认，通常 4096 */ }
  if (retval != 0) guardize = my_thread_stack_size;
#endif

#if defined(__ia64__) || defined(__ia64)
  /*
    Peculiar things with ia64 platforms - it seems we only have half the
    stack size in reality, so we have to double it here
  */
  guardize = my_thread_stack_size;
#endif

  if (0 != my_thread_attr_setstacksize(&connection_attrib,
                                       my_thread_stack_size + guardize)) {
    assert(false);
  }
```

**★ 关键辨析：`guardize` 到底是什么？**

`guardize` 这个名字有极强的误导性——它**不是** glibc 的 guard page 大小（尽管初值取自 `pthread_attr_getguardsize`）。它的真实语义是：**在申请栈大小时额外多申请的一块余量**。

| 平台 | `guardize` 的值 | 实际申请的 pthread 栈 |
|---|---|---|
| **x86_64（绝大多数生产环境）** | `pthread_attr_getguardsize` 返回值，通常 **4096** | **`thread_stack` + 4KB** |
| **ia64（安腾，已基本绝迹）** | `= my_thread_stack_size`（即翻倍） | **2 × `thread_stack`** |
| `getguardsize` 失败的兜底 | `= my_thread_stack_size` | 2 × `thread_stack` |

ia64 那段注释很坦白："it seems we only have half the stack size in reality, so we have to double it here"——那个平台上实际可用栈只有申请值的一半，所以直接翻倍补回来。

> **易错点**：看到 `my_thread_stack_size + guardize` 就以为"实际栈是配置值的两倍"，这是错的。在 x86_64 上多出来的只有 **4KB**。这个区别在分析事故时很关键——差一倍会导致"按理不该爆"的错误判断。

### 三、配套防御：实际栈不足时自动下调

申请完还会回读一次实际值：

```cpp
{
  /* Retrieve used stack size;  Needed for checking stack overflows */
  size_t stack_size = 0;
  my_thread_attr_getstacksize(&connection_attrib, &stack_size);

  /* We must check if stack_size = 0 as Solaris 2.9 can return 0 here */
  if (stack_size && stack_size < (my_thread_stack_size + guardize)) {
    LogErr(WARNING_LEVEL, ER_STACKSIZE_UNEXPECTED,
           my_thread_stack_size + guardize, (long)stack_size);
#if defined(__ia64__) || defined(__ia64)
    my_thread_stack_size = stack_size / 2;
#else
    my_thread_stack_size = static_cast<ulong>(stack_size - guardize);
#endif
  }
}
```

**为什么必须回读并下调？** 因为后面 `check_stack_overrun` 的判定基准是 `my_thread_stack_size`。如果系统实际给的栈**小于**请求值（某些平台/容器环境会 clamp），而 `my_thread_stack_size` 仍维持请求值，就会出现**致命的错位**：软件检查认为"还有空间"，实际已经踩到边界——检查结果失效，SIGSEGV 照旧。

所以这里一旦检测到实际值偏小，就**把 `my_thread_stack_size` 下调到实际值**，并打 `ER_STACKSIZE_UNEXPECTED` 警告，让软件检查阈值与实际栈对齐。

> 这是"**防御性编程**"的好例子：不信任系统调用的返回值，用回读校验 + 自我修正，而不是 assert 了事。

### 四、★ `thread_stack` 的作用范围：只管连接线程

这是本篇最重要的范围界定。`connection_attrib` 这个全局线程属性**只用于连接线程**：

```cpp
my_thread_attr_t connection_attrib;          // mysqld.cc 全局

// 初始化：连接线程是 DETACHED
(void)my_thread_attr_init(&connection_attrib);
my_thread_attr_setdetachstate(&connection_attrib, MY_THREAD_CREATE_DETACHED);
pthread_attr_setscope(&connection_attrib, PTHREAD_SCOPE_SYSTEM);

// 使用：三种连接接纳线程
key_thread_handle_con_namedpipes, &hThread, &connection_attrib, ...
key_thread_handle_con_sockets,    &hThread, &connection_attrib, ...
key_thread_handle_con_sharedmem,  &hThread, &connection_attrib, ...
```

而 **InnoDB 的线程走的是 `std::thread`**（见 `os0thread-create.h` 的 `create_detached_thread`）：

```cpp
std::thread t(std::move(detached_thread), f, args...);
```

`std::thread` 不传任何 `pthread_attr`，用的是 **glibc 默认栈大小**（x86_64 Linux 上通常是 **8MB**）。

**由此得出的判据：**

| 线程类别 | 栈大小来源 | 典型值 | 能否用 `thread_stack` 调整 |
|---|---|---|---|
| **连接线程**（Per_thread / 线程池 worker） | `connection_attrib` ← `thread_stack` | 1MB（默认）/ 256K（若调小） | ✅ 能 |
| **InnoDB 后台线程** | `std::thread` 默认 | **8MB** | ❌ **不能** |
| server 层其他后台线程 | 各自 attr 或默认 | 各异 | 部分能 |

**两条推论：**

1. **栈溢出几乎是连接线程的专利。** InnoDB 后台线程有着 8 倍大的栈（8MB vs 1MB），且无法被 `thread_stack` 调小——所以"后台线程栈溢出"极为罕见。
2. **调小 `thread_stack` 省不了多少内存**。它只影响连接线程，而栈是虚拟内存、物理按需兑现（见理论基础第二节），`max_connections=1000` 配 1MB 栈也只是 1GB **虚拟**地址空间，实际物理占用取决于每连接真正用了多少栈。为省内存把它调到 256K 是**性价比极低的优化，风险却很高**。

---

## ★ 软件栈检查：sql/check_stack.cc

既然线程栈不能自动增长，MySQL 就必须自己判断"还能不能继续递归"。这就是 `sql/check_stack.cc`（153 行，一个职责极纯的独立文件）。

### 一、核心实现

```cpp
bool check_stack_overrun(const THD *thd, long margin, unsigned char *buf) {
  assert(thd == current_thd);
  assert(stack_direction == -1 || stack_direction == 1);

#if defined(HAVE_ASAN)
  // Stack grows upward, but our address computations do not work with
  // the "fake stack" of ASAN. Just return OK.
  if (stack_direction == 1) {
    return false;
  }
#endif

  // 计算已用栈大小
  long stack_used =
      used_stack(thd->thread_stack, reinterpret_cast<char *>(&stack_used));
  if (stack_used >= static_cast<long>(my_thread_stack_size - margin) ||
      DBUG_EVALUATE_IF("simulate_stack_overrun", true, false)) {
    // Touch the buffer, so that it is not optimized away by -flto.
    if (buf != nullptr) buf[0] = '\0';

    /*
      Do not use stack for the message buffer to ensure correct
      behaviour in cases we have close to no stack left.
    */
    char *ebuff = new (std::nothrow) char[MYSQL_ERRMSG_SIZE];
    if (ebuff) {
      snprintf(ebuff, MYSQL_ERRMSG_SIZE,
               ER_THD(thd, ER_STACK_OVERRUN_NEED_MORE), stack_used,
               my_thread_stack_size, margin);
      my_message(ER_STACK_OVERRUN_NEED_MORE, ebuff, MYF(ME_FATALERROR));
      delete[] ebuff;
    }
    return true;
  }
#ifndef NDEBUG
  max_stack_used = std::max(max_stack_used.load(), stack_used);
#endif
  return false;
}
```

**判定式**：`已用栈 >= thread_stack − margin` → 报 `ER_STACK_OVERRUN_NEED_MORE`（"Thread stack overrun"）并返回 true，调用方逐层返回错误。

### 二、三个精妙细节

**① 基准从哪来：`thd->thread_stack` 是"局部变量的地址"**

```cpp
// mysqld.cc，连接线程入口附近
thd->thread_stack = (char *)&thd;
```

用**入口函数第一个局部变量（THD 指针本身）的地址**当作"栈底近似值"。这不是精确的栈底（栈底在更高地址），但**它是保守的**——用它算出的"已用栈"比实际偏大，等于提前一点触发保护。

> 这个技巧的细节与权衡，见 [`thd.md`](thd.md) 与 [`internal_thd.md`](internal_thd.md)（`internal_thd.md` 有专节《`thread_stack` 为什么能用局部变量地址》）。

**② 栈增长方向是运行期探测的，且要防编译器优化**

```cpp
namespace {
int stack_direction = 0;
}

NO_INLINE
int stack_direction_f(int *a) {
  int b;
  if (&b > a) {
    return 1;      // 栈向上增长
  }
  return 0;        // 栈向下增长
}

/*
 Prevent compiler optimizations by calling function
 through pointer.
*/
int (*ptr_f)(int *) = stack_direction_f;

NO_INLINE
int initialize_stack_direction() {
  int a;
  int retval = ptr_f(&a);
  if (retval == 0) {
    stack_direction = -1;
  } else {
    stack_direction = 1;
  }
  return retval;
}
```

为什么要这么绕？

- 栈增长方向是**平台相关**的（x86 向下，PA-RISC 等向上），不能编译期写死
- 用 `NO_INLINE` 防止内联导致 `b` 和 `a` 在同一帧、地址比较失去意义
- 用**函数指针 `ptr_f`** 间接调用——注释原话："Prevent compiler optimizations by calling function through pointer"。如果直接调用，编译器可能做内联或跨函数分析，把地址比较常量折叠掉

**③ 错误消息故意用堆分配**

```cpp
/*
  Do not use stack for the message buffer to ensure correct
  behaviour in cases we have close to no stack left.
*/
char *ebuff = new (std::nothrow) char[MYSQL_ERRMSG_SIZE];
```

栈快满的时候，**连构造错误消息的 buffer 都放不下**。所以这里用 `new`（堆）+ `std::nothrow`（分配失败返回 null 而不抛异常——抛异常也要用栈展开）。注释把理由写得很清楚。

> 这是"**在资源耗尽的边界上，连错误处理本身都要考虑资源**"的典型。类似的思路在 OOM 处理里也常见。

### 三、margin 的含义

```cpp
// sql/sql_const.h
constexpr const long STACK_MIN_SIZE{30000};  // Abort if less stack during eval.
constexpr const long STACK_MIN_SIZE{20000};  // （另一分支，Windows 等）
```

`margin` 是**保留给后续执行的余量**，不是"剩余多少就报警"。语义是：

> "如果剩余栈少于 `margin`，就别再往下走了——因为接下来真正的执行（eval）还要吃这么多。"

调用方传 `STACK_MIN_SIZE`（约 30KB）的含义就是：**留 30KB 给求值阶段**。这解释了一个重要现象——软件检查会在真正爆栈之前相当早就拦下来（30KB 相当于几十层递归的深度）。

### 四、软件栈检查的作用范围：server 层专属

**★ `check_stack_overrun` 是 server 层机制——`storage/innobase/` 下 0 处调用。** InnoDB 完全不做软件栈检查，原因有二：

| 理由 | 说明 |
|---|---|
| **栈足够大** | InnoDB 线程走 `std::thread`（`os0thread-create.h`），用 glibc 默认栈（x86_64 约 **8MB**），是连接线程默认值的 8 倍 |
| **递归天然有界** | B-tree 层数、dict 递归、锁递归等都有明确上界，不像"用户写出来的表达式"那样可以任意嵌套 |

> 唯一在 InnoDB 侧命中的 `stacksize` 是 `fts/fts0pars.cc` 里的 `yystacksize`——但那是 **FTS 的 bison 解析器栈**（堆上动态增长的数组，配 `YYSTACK_GAP_MAXIMUM`），**与线程栈无关**，不要混为一谈。

`check_stack_overrun` 只在**被调用的地方**生效。它不像 OS 的 guard page 那样是**兜底的最后一道防线**——恰恰相反：

| 防护层 | 触发条件 | 后果 | 覆盖面 |
|---|---|---|---|
| **软件检查**（`check_stack_overrun`） | 递归路径上**有检查点** | 优雅报 `ER_STACK_OVERRUN_NEED_MORE` | **部分路径** |
| **硬件/内核**（guard page `PROT_NONE`） | 真的踩到栈尾 | **SIGSEGV，进程崩溃** | **全部路径** |

**所以：有没有检查点，决定了同一个栈溢出表现为"报错"还是"崩溃"。**

---

## ★ 检查点地图：哪些递归路径有保护

这是本篇最实用的部分——只有源码级盘点才能画出来，而且它直接决定故障排查方向。

### 一、✅ contextualize 递归：基类级全覆盖

**这是最有价值的一处检查**，位于 parse tree 节点模板的基类：

```cpp
// sql/parse_tree_node_base.h
virtual bool contextualize(Context *pc) {
    uchar dummy;
    if (check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy)) return true;
    ...
    return false;
}
```

**为什么这一处顶得上很多处？** 因为 `contextualize` 是 **parse tree → Item 树**的递归遍历入口，而每个具体节点的 `contextualize` 都会（直接或间接）经过基类版本。许多派生节点还不放心，在自己实现的开头**再查一次**：

```cpp
// sql/parse_tree_nodes.h，PT_option_value_list
bool contextualize(Parse_context *pc) override {
    uchar dummy;
    if (check_stack_overrun(pc->thd, STACK_MIN_SIZE, &dummy)) return true;
    return head->contextualize(pc) || super::contextualize(pc);
}
```

**含义**：一条 SQL 的 parse tree 有多深，`contextualize` 就递归多少层，而**每层进入时都查一次栈**。所以**语法/语义分析阶段（contextualize）的深递归是被完整保护的**——达到阈值会报 "Thread stack overrun" 而不是崩。

> 这是"**把检查放进基类虚函数**"的设计价值：一处代码覆盖整个继承体系的所有递归。相比在每个派生类里手写，既不会漏，也不会重复。

### 二、各文件检查点统计（8.0.39 实测）

对 `sql/item*.cc` 全族统计 `check_stack_overrun` 出现次数：

| 文件 | 检查点数 | 说明 |
|---|---|---|
| `sql/item_func.cc` | 3 | 部分通用函数 |
| `sql/item_cmpfunc.cc` | 3 | 比较/逻辑函数（`AND`/`OR`/`CASE` 等） |
| `sql/item_xmlfunc.cc` | **2** | ★ **XPath 函数——有防护** |
| `sql/item_subselect.cc` | 1 | 子查询 |
| **`sql/item_json_func.cc`** | **0** | ★ **JSON 全部函数——零防护** |
| **`sql/item.cc`** | **0** | ★ **Item 基类（walk / copy / split_sum_func 等递归）——零防护** |
| `sql/item_strfunc.cc`、`item_timefunc.cc` 等其余 | 0 | 无 |

**★ 最扎眼的对照：`item_xmlfunc.cc` 有 2 处，`item_json_func.cc` 有 0 处。**

XML 的 XPath 求值和 JSON 的函数求值在结构上是**同一类问题**——都是对嵌套结构做递归求值。一边加了栈检查，另一边一处没加。这不是设计取舍，更像是**演进过程中的遗漏**：XML 函数年代久远，经历过栈溢出问题后补了检查；JSON 函数（5.7 引入）是新代码，没有继承这个教训。

### 三、由此得出的诊断判据

| 观测到的现象 | 说明什么 | 排查方向 |
|---|---|---|
| 报 **`ER_STACK_OVERRUN_NEED_MORE`**（"Thread stack overrun: N bytes used of a 262144 byte stack..."） | 崩在**有检查点的路径**：contextualize 递归、或 `item_func`/`item_cmpfunc`/XML 的求值 | 错误本身带诊断信息（已用/总量/margin），直接看 SQL 嵌套深度 |
| **SIGSEGV**（无 MySQL 错误信息，只有 core / signal 11） | 崩在**无检查点的路径**：JSON 求值递归、Item 基类递归 | 必须靠 core dump 看调用栈，错误信息里**没有任何 MySQL 层面的线索** |

> **这条判据是本篇最实用的产出**：同样是"栈不够了"，**有无 MySQL 错误信息**直接定位到是哪一类路径。看到 SIGSEGV 且调用栈里是 `Item_func_json_*` 的一层层 `val_json`，就是零检查点路径的典型形态。

### 四、为什么"有保护"却仍然会崩

有人会问：既然 contextualize 全程有检查，一条深嵌套的 SQL 在 contextualize 阶段不就该被拦住吗？

**不一定**，因为**不同阶段的每层栈消耗不同**：

| 阶段 | 每层做什么 | 每层栈消耗 |
|---|---|---|
| contextualize | 遍历 PT 节点、建 Item | 相对较小 |
| **执行期 `val_*` 递归** | 真正求值：构造 Json_dom、COALESCE 参数传递、IF 分支…… | **明显更大** |

contextualize 阶段数百层可能还**没到**检查线（于是放行），但到了执行期，每层更大的栈帧让同样的深度**冲过**了实际栈尾——而这条路径**一个检查点都没有**，于是 SIGSEGV。

**这就是"有保护"和"保护得住"的差距**：防护布置在了 A 阶段，真正的溢出发生在 B 阶段。

---

## ★ 栈 vs MEM_ROOT：数据放哪 vs 调用多深

这是最容易混淆的一组概念，值得单列一节。

### 一、两者是什么

| | **MEM_ROOT** | **C 栈（`thread_stack`）** |
|---|---|---|
| 本质 | **堆**上的内存池（arena allocator） | 线程的调用栈 |
| 分配方式 | `alloc_root()`，指针碰撞式的批量分配 | 函数调用自动压帧， `%rsp` 移动 |
| 谁在用 | Item 对象、LEX 结构、字符串缓冲、各类临时结构 | 函数调用、局部变量、返回地址 |
| 生命周期 | 绑定 THD / LEX / 语句，**整块释放**（`free_root`） | 函数返回即回收 |
| 容量 | 动态增长，理论上到进程内存上限 | **固定**，启动时定死 |
| 耗尽后果 | 分配失败 → 报 OOM 类错误（可捕获） | **SIGSEGV 或优雅报错**（取决于有无检查点） |
| 调整参数 | 无（跟随 `tmp_table_size` 等间接影响） | **`thread_stack`** |

### 二、源码证据：Item 对象在堆上（MEM_ROOT）

```cpp
// sql/item.h
static void *operator new(size_t size) noexcept { ... }
static void *operator new(size_t size, MEM_ROOT *mem_root, ...)
```

Item 用 **placement new 分配在 MEM_ROOT 上**。也就是说——**深嵌套表达式里那数百个 Item 对象，全部在堆上**，由 MEM_ROOT 管理，随语句结束整块释放。

### 三、★ 那为什么还会爆栈？

因为**创建对象**和**遍历/求值对象**是两件事：

```
深嵌套的 JSON_SET(JSON_EXTRACT(COALESCE(IF(...))))

堆（MEM_ROOT）：                    栈（C stack）：
┌─────────────────────────┐        ┌─────────────────────────┐
│ Item_func_json_set      │        │ val_json()  帧 1        │
│ Item_func_json_extract  │        │ val_json()  帧 2        │
│ Item_func_coalesce      │   ←──→ │ val_json()  帧 3        │
│ Item_func_if            │  遍历  │   ...                   │
│ ... （数百个对象）      │        │ val_json()  帧 N        │
│                         │        │                         │
│ 占用：几十 KB（很便宜）   │        │ 占用：数百层×数百B≈200KB│
└─────────────────────────┘        └─────────────────────────┘
     数据放哪 ✅                        调用钻多深 ⚠️
```

**一句话区分：**

> **MEM_ROOT 管"数据放哪"（堆，池化，生命周期可控）；`thread_stack` 管"调用能钻多深"（栈，逐层消耗）。**

### 四、两条推论

1. **把 MEM_ROOT 调得再大也救不了栈溢出**——MEM_ROOT 只影响对象能存多少，不影响递归能钻多深。
2. **把栈调大也救不了内存暴涨**——深嵌套 SQL 的内存问题（大量 Item、大 JSON）走的是堆。

所以"MySQL 用 MEM_ROOT，所以不吃栈"是个**把两个正交维度混为一谈**的错误结论。正确的说法是：**数据在堆上（MEM_ROOT），遍历它们的过程在栈上**。

### 五、还有一类"栈堆配合"的中间形态

值得顺带一提：有些代码会**先在栈上开个小 buffer，不够了才转到堆/MEM_ROOT**。典型是 `String` 类的行为——构造时可能带一块栈上或内嵌的小缓冲，长度超出后再向 MEM_ROOT 申请。

这类设计让"小值零分配、大值才动用堆"，但也意味着**栈消耗和堆消耗在同一段代码里混合出现**，分析时不能只看一头。

---

## ★ JSON 嵌套的容量分析

按你指定的位置，JSON 的容量问题放在本篇统一分析（而不是散在 `server/datatype/json.md`）。

### 一、两类"深度"必须分开

| 类型 | 例子 | 谁在递归 | 有防护吗 |
|---|---|---|---|
| **JSON 值/文档的深度** | 一个嵌套 50 层的 JSON 文档作为**字面量或字段值** | JSON 解析器（递归下降） | 解析路径本身的处理方式不同于表达式求值 |
| **表达式的嵌套深度** | `JSON_SET(JSON_EXTRACT(COALESCE(IF(...))))` 嵌套数百层 | **Item 树的 `val_json()` 递归** | ❌ **`item_json_func.cc` 0 处检查** |

**这是两个完全不同的量**，却经常被混淆：

- 前者是**数据**的深度——一个 JSON 文档嵌了多少层
- 后者是**代码**的深度——一条 SQL 表达式嵌了多少层函数

**本篇关注的是后者。** 前者虽然也吃栈（解析 JSON 字面量也是递归下降），但它受的是另一套约束；后者直接由 Item 求值递归决定，而这条路径**零检查点**——所以栈深度治理真正要盯的是表达式深度。

### 二、JSON 值深度的计算本身是递归的

```cpp
// sql-common/json_dom.cc，depth() 的实现片段
deepest_child = std::max(deepest_child, iter->second->depth());   // object
deepest_child = std::max(deepest_child, child->depth());          // array
```

`Json_dom::depth()` 自己就是一个**递归函数**——计算一个 JSON 文档的嵌套深度要递归遍历它。这意味着**查询深度这个动作本身也吃栈**（尽管单层消耗很小）。

### 三、深嵌套表达式的成本推算

以典型的深嵌套形态为例（`JSON_SET` / `JSON_EXTRACT` / `COALESCE` / `IF` 交替嵌套）：

| 成本项 | 位置 | 量级 |
|---|---|---|
| **Item 对象** | MEM_ROOT（堆） | 数百个对象，几十 KB——**很便宜** |
| **contextualize 递归** | 栈 | 数百层，有检查点，**可能未触发**（每层较小） |
| **执行期 `val_json()` 递归** | 栈 | 数百层 × 每层数百字节 ≈ **200KB 量级**——**致命** |
| 入口已有调用链 | 栈 | 十余 KB |

**关键点**：同样的嵌套深度，**在堆上很便宜，在栈上很贵**。这正是上一节"栈 vs MEM_ROOT"的现实版本。

### 四、为什么这条路径特别危险

三个因素叠加：

1. **零检查点**——`item_json_func.cc` 一处 `check_stack_overrun` 都没有，所以不会优雅报错，只会 SIGSEGV
2. **每层栈消耗不小**——JSON 求值涉及 `Json_dom` 构造、路径表达式处理、临时对象，栈帧比简单的算术函数大
3. **触发成本极低**——一条 SQL 就足以让线程踩过守护页，而语句本身往往**由程序自动生成**（ORM、报表工具、ETL 拼 SQL），人工审查很难提前发现

**补充一个复制相关的推论**：深嵌套 SQL 若以 **SBR** 记录，从库重放原始语句时**同样会递归**（因而同样危险）；若是 **RBR**，行事件里**不含原始表达式**，从库不会重现该递归。这是判断"故障会不会随复制扩散"的依据。

### 五、与 XML 的对照（再次强调）

`item_xmlfunc.cc` 有 2 处检查、`item_json_func.cc` 有 0 处。**同样深度的嵌套表达式，打在 XPath 上会收到一条 `ER_STACK_OVERRUN_NEED_MORE`，打在 JSON 上则直接 SIGSEGV。**

---

## 可观测性与调优

### 一、怎么知道栈用了多少

| 手段 | 说明 |
|---|---|
| 错误信息 | `ER_STACK_OVERRUN_NEED_MORE` 会打印 `stack_used` / `my_thread_stack_size` / `margin`——**只有有检查点的路径才有** |
| debug 构建 | `check_stack.cc` 里有 `std::atomic<long> max_stack_used`，记录历史最大栈使用 |
| `DBUG_EVALUATE_IF("simulate_stack_overrun", ...)` | 用 debug 开关**模拟**栈溢出，用于测试优雅报错路径 |
| core dump | SIGSEGV 时的调用栈深度是唯一线索（无检查点路径） |

### 二、调优建议

| 场景 | 建议 |
|---|---|
| 默认情况 | **保持默认 1MB**，不要为省内存调小 |
| 深嵌套 SQL、复杂视图、深度递归的存储过程 | **调大**（如 2MB、4MB） |
| 已经调到 256K/512K 的实例 | **强烈建议评估恢复**——这是把递归深度上限砍到 1/4~1/2 |
| ASAN / UBSAN 构建 | 栈帧显著变大，下限本身就是 1MB，别再调小 |
| 想通过调小栈省内存 | **收益极低**：栈是虚拟内存、物理按需，且只作用于连接线程 |

### 三、★ 一个重要的认知

**`thread_stack` 是"递归深度上限"的间接控制杆，而不是内存优化参数。** 由于栈物理按需兑现，调小它几乎省不到内存，却实实在在地砍掉了能承受的递归深度——**风险收益完全不对等**。

---

## ★ 本机制里的工程实现技法

### 一、把检查放进基类虚函数：一处覆盖整个递归体系

`Parse_tree_node_tmpl::contextualize()` 里那一行 `check_stack_overrun`，保护了**整个 parse tree 继承体系的所有 contextualize 递归**。相比在每个派生节点手写，既不会漏写，也不会重复。

> 可迁移的启发：**当防护逻辑对"所有子类"都成立时，放进基类的模板方法，而不是指望每个实现者记得写。**

### 二、`NO_INLINE` + 函数指针：对抗编译器优化的双重保险

探测栈增长方向时，`NO_INLINE` 防止内联破坏地址比较的语义，函数指针 `ptr_f` 防止编译器跨函数分析后常量折叠。注释原话："Prevent compiler optimizations by calling function through pointer."

> 可迁移的启发：**依赖"运行时地址关系"的代码，必须主动对抗优化器**，否则 `-O2` 下行为可能完全改变。

### 三、在资源耗尽的边界上，错误处理本身也要考虑资源

```cpp
/*
  Do not use stack for the message buffer to ensure correct
  behaviour in cases we have close to no stack left.
*/
char *ebuff = new (std::nothrow) char[MYSQL_ERRMSG_SIZE];
```

栈快满时，连错误消息的 buffer 都放不下——所以用堆 + `std::nothrow`（避免抛异常时的栈展开）。

### 四、回读校验 + 自我修正，而不是 assert 了事

申请栈大小后回读实际值，若偏小则**把 `my_thread_stack_size` 下调到实际值**并打警告。这保证了软件检查的判定基准**始终与真实栈一致**——否则检查会失效（以为还有空间，实际已越界）。

> 可迁移的启发：**凡"配置值"与"系统实际给予的值"可能不一致的地方，都要做回读校正**，尤其当这个配置值还被用作安全判定的基准时。

### 五、平台差异用 `guardize` 变量吸收

ia64 上"实际只有一半栈可用"这种平台怪癖，被 `guardize` 一个变量吸收掉，主逻辑 `my_thread_stack_size + guardize` 保持不变。

---

## 易误判清单

| # | 常见误判 | 实际 |
|---|---|---|
| 1 | "`thread_stack` 默认很小（几十 KB）" | **默认 1MB**；下限才是 128K |
| 2 | "MySQL 用 MEM_ROOT，所以不吃栈" | 数据在堆（MEM_ROOT），**遍历/求值在栈**，两者正交 |
| 3 | "栈溢出 MySQL 会优雅报错" | 只在**有检查点的路径**；`item_json_func.cc` 零检查 → SIGSEGV |
| 4 | "`my_thread_stack_size + guardize` 说明实际栈是 2 倍" | **x86_64 上 guardize 只有 4KB**；翻倍仅在 ia64 |
| 5 | "`thread_stack` 管所有线程" | **只作用于 `connection_attrib`（连接线程）**；InnoDB 用 `std::thread` 默认 8MB |
| 6 | "调小栈能省很多内存" | 栈是虚拟内存、物理按需，且只影响连接线程——**收益极低、风险很高** |
| 7 | "JSON 值和表达式的深度是同一回事" | 前者是**数据**深度（文档嵌多少层），后者是**代码**深度（表达式嵌多少层函数）；栈深度治理盯的是后者 |
| 8 | "深嵌套 SQL 的栈风险会随 binlog 复制到从库" | **RBR 不会**（行事件不含原始表达式），**SBR 会**（重放原始语句时同样递归） |
| 9 | "`check_stack_overrun` 只在解析期用" | contextualize 是**基类级**覆盖（每个节点进入都查），远不止"解析期一处" |
| 10 | "运行时能改 `thread_stack`" | `READ_ONLY`，**必须重启** |

---

## Misc

### 交叉链接

| 主题 | 位置 |
|---|---|
| 线程创建原语与生命周期 | [`thread_lifecycle.md`](thread_lifecycle.md)（含 `DETACHED`/`JOINABLE`、`my_thread_create`） |
| THD 对象、`thread_stack` 基准的由来 | [`thd.md`](thd.md) |
| 内部 THD 的 `thread_stack` 权衡（专节） | [`internal_thd.md`](internal_thd.md) |
| 连接线程模型 | [`connection_threading.md`](connection_threading.md)（含 `thd_set_thread_stack(thd, (char *)&thd)` 技巧） |
| 线程全景与 InnoDB 后台线程 | [`threading_architecture.md`](threading_architecture.md) |
| 内存架构（栈作为虚拟内存的定位） | [`../memory/memory_architecture.md`](../memory/memory_architecture.md) |
| JSON 数据类型（值语义、函数语义） | [`../../server/datatype/json.md`](../../server/datatype/json.md) |
| Item 表达式（栈消耗视角） | [`../../server/query/13_item_expression.md`](../../server/query/13_item_expression.md) |
| contextualize 流程 | [`../../server/query/05_contextualize.md`](../../server/query/05_contextualize.md)（★ 该篇目前**未写** `check_stack_overrun`，本篇是其栈侧补充） |

### 关键源码坐标速查

| 主题 | 位置 |
|---|---|
| `DEFAULT_THREAD_STACK` | `include/my_thread.h`（`(1024UL * 1024UL)`） |
| `thread_stack` 变量定义 | `sql/sys_vars.cc` → `Sys_thread_stack` |
| 栈大小设置与回读校正 | `sql/mysqld.cc`（`connection_attrib` / `guardize` / `ER_STACKSIZE_UNEXPECTED`） |
| 软件栈检查 | `sql/check_stack.cc` / `sql/check_stack.h` |
| `STACK_MIN_SIZE` | `sql/sql_const.h`（30000） |
| contextualize 基类检查 | `sql/parse_tree_node_base.h` → `Parse_tree_node_tmpl::contextualize` |
| Item 分配于 MEM_ROOT | `sql/item.h` → `operator new(size_t, MEM_ROOT *)` |
| InnoDB 线程栈来源 | `storage/innobase/include/os0thread-create.h`（`std::thread`） |

### 待补充 / 未确证项

- **JSON 值深度是否存在硬限制**：本次未在源码中找到明确的深度上限常量（曾尝试在 `sql-common/json_dom.*` 中检索未果）。官方语义上有 JSON 文档深度限制的说法，但**本篇未从源码确证，故不列为事实**。若后续确证，应补在此节。
- **逐层栈消耗的绝对值**：依赖编译器、优化级别与构建选项（debug / UBSAN 插桩会显著放大栈帧），**不是源码常量**；本篇只给量级（数百字节/层），不给出未实测的具体数字。
- 各递归路径的**逐层栈消耗实测值**：需要特定构建下实测（可用 `DBUG` + `max_stack_used` 或 perf 采样），本次未做。

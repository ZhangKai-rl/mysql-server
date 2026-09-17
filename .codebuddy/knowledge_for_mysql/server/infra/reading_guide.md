# MySQL 源码阅读知识地图

> 基于 MySQL 8.0.39 源码。回答一个问题：**读 MySQL 源码读不懂时，缺的是什么、去哪补**。
> 涵盖七个知识域、C 预处理器与 C++ 惯用法清单、设计模式与源码的完整对照、分阶段学习路线、
> "读不懂"的自诊断流程、参考资料与学术索引。

> **边界**：本篇是**阅读源码的前置知识与方法论**（通用能力），不是某个机制的实现剖析；
> 具体机制见 [`pfs.md`](pfs.md)、[`memory.md`](memory.md)、[`list.md`](list.md) 等同目录各篇，
> 以及 [`variables.md`](variables.md)、[`encoding.md`](encoding.md)。
> 本目录的文件索引见 [`README.md`](README.md)。

## 目录

- [1. 总览：七个知识域](#1-总览你必须掌握的七个知识域)
- [2. 第一域：领域理论基础](#2-第一域领域理论基础最优先决定了你为什么读)
- [3. 第二域：C 预处理器 + C++ 惯用法](#3-第二域c-预处理器--c-惯用法决定了你看不看得懂代码字面意思)
- [4. 第三域：设计模式](#4-第三域设计模式决定了你看不看得懂代码组织)
- [5. 第四域：操作系统基础](#5-第四域操作系统基础决定了你看得懂底层行为)
- [6. 第五域：并发与内存模型](#6-第五域并发与内存模型决定了你能理解多线程正确性)
- [7. 第六域：计算机体系结构](#7-第六域计算机体系结构决定了你理解为什么这么设计)
- [8. 第七域：工程工具链](#8-第七域工程工具链决定了你怎么高效读)
- [9. 推荐学习路线](#9-推荐学习路线分阶段执行不要一次性补完)
- [10. "读不懂"的自诊断流程](#10-读不懂的自诊断流程)
- [11. 必读参考资料清单](#11-必读参考资料清单按需取用)
- [设计模式与 MySQL 源码完整对照表](#设计模式与-mysql-源码完整对照表)
- [推荐学习资源与路线](#推荐学习资源与路线)
- [核心术语中英文对照](#核心术语中英文对照设计模式)
- [学术论文索引](#学术论文索引软件工程与设计模式)
- [计算机科学主要领域与顶级会议](#计算机科学主要领域与顶级会议)
- [共享设计哲学](#共享设计哲学跨模块通用)
- [参考](#参考)

---

## 阅读 MySQL 源码需要补什么？—— 完整知识地图

这一节的目标不是"列出所有可能需要的东西"，而是**从源码出发，告诉你某某行代码为什么读不懂、你要补什么才能读懂**。按遇到问题的频率和阻塞程度排列。

---

## 1. 总览：你必须掌握的七个知识域

```
                              ┌─────────────────────┐
                              │   7. 工程工具链       │
                              │   CMake, gdb, git    │
                              │   ctags, perf, ASAN  │
                              ├─────────────────────┤
                              │   6. 计算机体系结构    │
                              │   cache line, NUMA   │
                              │   TLB, branch pred   │
                              ├─────────────────────┤
                              │   5. 并发与内存模型    │
                              │   futex, spinlock    │
                              │   atomic, memory_order│
                              ├─────────────────────┤
                              │   4. 操作系统基础      │
                              │   mmap, huge page    │
                              │   virtual memory     │
               ┌──────────────┴─────────────────────┴──────────────┐
               │                    3. 设计模式                    │
               │    Strategy / Template Method / Observer / Facade │
               │    Bridge / Adapter / Decorator / Factory / RAII │
               ├──────────────────────────────────────────────────┤
               │              2. 语言：C 预处理器 + C++ 惯用法     │
               │    宏展开 ## # 条件编译  /  template RAII atomic │
               ├──────────────────────────────────────────────────┤
               │              1. 领域理论基础                      │
               │    Event Tracing, AOP, DTrace, Buddy, Slab,      │
               │    Bump/Arena, Segregated-fit, Size Class        │
               └──────────────────────────────────────────────────┘
```

**读源码不是先学完所有知识再开始**，而是**遇到看不懂的地方 → 定位是哪个知识域 → 去补 → 再回来看**。下面按这个逻辑展开。

---

## 2. 第一域：领域理论基础（最优先，决定了你"为什么读"）

**如果不知道 buf_buddy 是 buddy allocator，你会把它当成普通的 linked-list 分配器来读，然后就觉得"这段代码好奇怪"然后放弃。**

| 读到哪里 | 会遇到什么困惑 | 需要补充的知识 | 补什么资料 |
|---------|--------------|--------------|----------|
| `mem0mem.h:86-91` 看到三种 type | "DYNAMIC 和 BUFFER 到底有什么区别？为什么还有 BTR_SEARCH 这个奇怪的组合值？" | Bump/Arena allocator 的设计取舍：从堆分配 vs 从预分配池分配，不可失败 vs 可返回 NULL | [`memory.md`](memory.md)；Knuth TAOCP Vol.1 §2.5 |
| `buf0buddy.cc:buf_buddy_alloc_low` | "为什么按 `1 << i` 分裂？释放时怎么找到伙伴？" | Buddy allocator 的分裂/合并算法，伙伴地址公式 `buddy = addr XOR (1 << order)` | Knowlton 1965 (CACM)；[`memory.md`](memory.md)「buddy allocator」 |
| `my_alloc.cc:103` 看到 `m_block_size += m_block_size / 2` | "为什么每次增加 50%？固定大小不好吗？" | Arena allocator 的指数增长策略 vs 固定块策略的适用场景 | mem_heap 作为对比：InnoDB 场景可预测 vs Server 层不可预测 |
| `memory.cc:278` 看到 `type == MEM_HEAP_DYNAMIC \|\| len < UNIV_PAGE_SIZE / 2` | "为什么 BUFFER 类型的小块也走 malloc？" | Buffer Pool 页面的机会成本：16KB 的 BP 页面换几 KB 临时数据是赔本买卖 | Buffer Pool 管理基础知识 |
| `jemalloc` 的设计 | "size class 和 buddy 的 2^n 级别有什么区别？" | Segregated-fit：对象按大小分类，每类独立 free list，释放后可**立即**被同大小请求复用 | Bonwick 1994 Slab Allocator 论文；[`memory.md`](memory.md)「jemalloc / size class」 |
| PFS `events_waits_summary_*` 表 | "event tracing 和 sampling 哪个好？为什么 PFS 选前者？" | Event-based Tracing vs Sampling 的理论基础：精确度、因果关系、开销模型 | Anderson SOSP 1997；DTrace USENIX ATC 2004 |
| `mysql_mutex.h` 中嵌套的 `#define` | "为什么 lock 一个 mutex 要通过这么多层宏？" | AOP 的 weaving 思想：横切关注点分离 + 编译期零开销注入 | Kiczales ECOOP 1997；[`pfs.md`](pfs.md)「AOP 编织」 |

---

## 3. 第二域：C 预处理器 + C++ 惯用法（决定了你"看不看得懂代码字面意思"）

**这是最大的阅读障碍来源**。MySQL 大量依赖 C 预处理器宏生成代码，不掌握预处理器就读不懂任何 PFS 代码。

**C 预处理器——必学清单：**

| 你必须理解的 | 为什么必须 | MySQL 中的典型例子 | 检验方法 |
|-------------|----------|-------------------|---------|
| **宏展开顺序**：参数先展开，然后替换 | 不懂就看不懂多层嵌套宏 | `#define mysql_mutex_lock(M) mysql_mutex_lock_with_src(M, __FILE__, __LINE__)` 继续展开到 `inline_mysql_mutex_lock(M, "file.cc", 320)` | 用 `gcc -E` 展开一个简单的 `mysql_mutex_lock(&foo)` 看输出 |
| **`#` 和 `##` 操作符** | PFS 中 `__FILE__`、自动拼接标识符都靠它们 | `#define PSI_MUTEX_CALL(M) psi_mutex_service->M` 中的函数名拼接 | 写一个简单宏测试 `#`(stringify) 和 `##`(token paste) |
| **条件编译 `#ifdef`** | PFS 的零开销关闭完全依赖它 | `#ifdef HAVE_PSI_MUTEX_INTERFACE` → 决定整段 instrumentation 代码是否存在 | 理解 `-DHAVE_PSI_MUTEX_INTERFACE` 编译选项如何改变生成的二进制 |
| **可变参数宏 `__VA_ARGS__`** | `ib::fatal()` 等日志宏使用 | `#define ut_a(EXPR) (void)((EXPR) || (ut_a_print(#EXPR, ...), 0))` | 找到 `ut_a` 的定义，手动展开 |
| **X-Macro 技巧** | PFS 大量使用 "用数据驱动代码生成" 的模式 | PFS 的 instrument class 注册表：一个宏定义数据表，另一个宏定义遍历逻辑 | 搜索 `PFS_engine_table_share` 相关宏 |

**C++ 必须掌握的惯用法——按在 MySQL 源码中出现频率排序：**

| 频率 | 你必须懂的 C++ 特性 | 不懂的后果 | MySQL 中哪里出现 | 推荐阅读 |
|------|-------------------|----------|----------------|---------|
| ⭐⭐⭐⭐⭐ | **RAII + `unique_ptr` + 自定义 deleter** | 看不懂 `Scoped_heap`、`Mem_root_allocator` 为什么"没有手动 free" | `mem0mem.h:440-507` Scoped_heap；`my_alloc.h` Mem_root_allocator | Stroustrup C++ 4th Ed. §5.2, §34.3 |
| ⭐⭐⭐⭐⭐ | **`std::atomic` + `memory_order`** | 看不懂 `free_block_ptr` 的 `compare_exchange_strong` | `mem0mem.h` free_block_for_heap；`rpl_handler.cc` Delegate spin lock | C++ Concurrency in Action §5 |
| ⭐⭐⭐⭐ | **模板特化 + SFINAE** | 看不懂 `ut::malloc_withkey` 为什么用 `select_malloc_impl_t` | `ut0new.h:619` 模板选择：`WITH_PFS_MEMORY == true` 时走 PFS 包装 | cppreference: SFINAE |
| ⭐⭐⭐⭐ | **`constexpr` / `static_assert`** | 看不懂编译期常量判断（如 `MEM_NO_MANS_LAND` 在 Debug/Release 差异） | `mem0mem.h:99-108` `constexpr int MEM_NO_MANS_LAND` | cppreference: constexpr |
| ⭐⭐⭐ | **`placement new`** | 看不懂 Buffer Pool page 如何在已有内存上构造对象 | `buf0buf.cc` 中的 page 初始化 | Stroustrup §11.2.4 |
| ⭐⭐⭐ | **`explicit` / `noexcept` / `[[maybe_unused]]`** | 不是阻塞项，但不认识会减慢阅读 | 遍布全项目 | cppreference 逐个查 |
| ⭐⭐ | **variadic template / `std::forward`** | `ut::new_<T>(args...)` 的完美转发 | `ut0new.h` 中的 `new_` 函数模板 | cppreference: perfect forwarding |

---

## 4. 第三域：设计模式（决定了你"看不看得懂代码组织"）

详见 [`pfs.md`](pfs.md) 的设计模式各节。最重要的三个：**Strategy**（PSI vtable）、**Template Method**（start/do/end）、**Observer**（Delegate）。

---

## 5. 第四域：操作系统基础（决定了你"看得懂底层行为"）

| 你必须理解的 OS 概念 | 为什么必须 | MySQL 中的典型例子 | 推荐资料 |
|---------------------|----------|-------------------|---------|
| **虚拟内存：`mmap` / `munmap` / `mprotect`** | 不懂就看不懂 Buffer Pool 为什么用 `mmap` 而不是 `malloc`，以及 `large_page_alloc` | `large_page_alloc-linux.h:mmap(MAP_HUGETLB)` | APUE §14.8; Linux man mmap |
| **Huge Page（大页）**：2MB/1GB 页 vs 4KB 页 | 不懂就看不懂为什么 BP 要单独走 huge page 路径 | `large_page_alloc` 强制 2MB 对齐 | Linux kernel doc: hugetlbpage |
| **TLB（Translation Lookaside Buffer）** | 不懂 huge page 就不知道为什么"减少 TLB miss" | BP 使用大页后 TLB 条目减少 512 倍 | CSAPP §9.6.2 |
| **Cache Line（缓存行）**：64 字节 | 不懂就看不懂 `ut::cacheline_aligned`、`PFS_cacheline_atomic_uint32` | PFS event 结构体中 `alignas(64)` 消除 false sharing | CSAPP §6.4 |
| **False Sharing（伪共享）** | 不懂就看不懂为什么结构体里插 `char padding[60]` | PFS 的 `PFS_cacheline_atomic_uint32` 保证每个计数器独占一个 cache line | 搜索 "false sharing cache line padding" |
| **Futex vs Pthread Mutex** | 不懂就看不懂 `mysql_mutex_t` 的底层实现选择 | `include/my_mutex.h` 中的 `native_mutex_t` 在不同平台的实现 | Linux man futex(7); pthread_mutex_lock(3) |

---

## 6. 第五域：并发与内存模型（决定了你"能理解多线程正确性"）

| 你必须理解的 | 为什么必须 | MySQL 中的典型例子 | 推荐资料 |
|-------------|----------|-------------------|---------|
| **mutex vs rwlock 的语义差异** | PFS 单独为两类 lock 设计了不同的 event 类型 | `PFS_events_waits` 中的 `WAIT_CLASS_MUTEX` vs `WAIT_CLASS_RWLOCK` | 任何并发编程教材 |
| **spinlock 与 blocking lock 的选择** | 不懂就看不懂 `replication_optimize_for_static_plugin_config` 为什么切换锁类型 | `Delegate` 中根据配置选择 `mysql_rwlock_t` 或 `Shared_spin_lock` | `rpl_handler.h:175-226` |
| **死锁的四个必要条件 + 锁顺序** | 不懂就看不懂 `MEM_HEAP_BTR_SEARCH` 的设计 | 持有 AHI latch 时不能从 BP 分配页面（防止递归锁）→ 预留在先 | [`memory.md`](memory.md)「死锁与锁顺序」；Coffman 1971 "deadlock conditions" |
| **CAS（Compare-And-Swap）语义** | 不懂就看不懂 `compare_exchange_strong` | `free_block_for_heap.compare_exchange_strong(expected, block)` | cppreference: atomic::compare_exchange |
| **C++11 Memory Order（内存序）** | 不懂就看不懂 PFS `m_enabled` 为什么有时用 `relaxed` 有时用 `acquire/release` | PFS consumer flags：`m_enabled.load(std::memory_order_relaxed)` | C++ Concurrency in Action §5.3 |
| **Lock-free 编程基础** | `free_block_ptr` 的原子操作设计 | `heap->free_block_ptr->load()` + `store(nullptr)` 的无锁模式 | Herlihy & Shavit, "The Art of Multiprocessor Programming" |

---

## 7. 第六域：计算机体系结构（决定了你"理解为什么这么设计"）

| 你必须理解的 | 为什么必须 | MySQL 中的典型例子 | 推荐资料 |
|-------------|----------|-------------------|---------|
| **Branch Predictor（分支预测器）** | PFS 零开销关闭的核心机制 | `if (m_psi->m_enabled)` 在关闭时分支预测 100% 命中 | CSAPP §4.5; Smith ISCA 1981 |
| **Cache Hierarchy（L1/L2/L3/主存）** | 解释为什么 cache line padding 有效 | PFS 聚合表中的 `PFS_cacheline_atomic_uint32` | CSAPP §6 |
| **Pipeline（指令流水线）** | 解释为什么分支预测错误代价大 | PFS 开启时 branch miss 的开销来源 | CSAPP §4.4 |
| **NUMA（非一致性内存访问）** | 解释 Buffer Pool 多实例设置 | `innodb_buffer_pool_instances`：每个 NUMA 节点一个 BP 实例 | CSAPP §9.5 |

---

## 8. 第七域：工程工具链（决定了你"怎么高效读"）

**不掌握这些工具，读源码效率除以 10。**

| 工具 | 用途 | 必须掌握的命令/操作 | MySQL 源码场景 |
|------|------|-------------------|--------------|
| **grep / ripgrep** | 搜索符号定义和引用 | `rg "mem_heap_create\(" --type cpp` | 找一个函数的所有调用点 |
| **ctags / cscope** | 跳转到符号定义 | `ctags -R storage/innobase/` 后 vim 中 `Ctrl-]` 跳转 | 从 `mem_heap_alloc` 跳转到其定义 |
| **gcc -E** | 查看宏展开结果 | `gcc -E -Iinclude file.c` | **理解 PFS 编织最关键的命令**：展开 `mysql_mutex_lock(&foo)` 看三层宏 |
| **gdb** | 运行时调试 | `b mem_heap_create_block`, `p type`, `p len`, `bt` | 在三种 type 分支设断点，观察不同调用路径 |
| **perf** | 性能采样 | `perf record -g`, `perf report` | 了解热点在哪，决定先读哪段代码 |
| **git blame / log** | 追溯代码历史 | `git blame mem0mem.h \| grep "BTR_SEARCH"` | 找到 `MEM_HEAP_BTR_SEARCH` 是谁在什么 commit 中引入的、commit message 说了什么 |
| **ASAN / Valgrind** | 运行时内存错误检测 | `-DWITH_ASAN=ON` cmake 选项 | Debug 模式下 `MEM_NO_MANS_LAND` + ASAN 双重检测越界 |
| **CMake 选项** | 理解编译配置 | `cmake -LH \| grep -i psi` | 看看哪些 PFS 接口被编译进去了 |
| **clang-format** | 理解代码风格 | `.clang-format` 文件 | MySQL 使用 Allman 风格（大括号独立一行），知道这点就不会觉得格式奇怪 |

**一个具体的效率技巧：用 gcc -E 展开宏的完整示例**

```bash
# 创建最小测试文件
cat > /tmp/test_mutex.c << 'EOF'
#include "mysql/psi/mysql_mutex.h"
void test() {
    mysql_mutex_t m;
    mysql_mutex_lock(&m);
}
EOF

# 展开宏（仅预处理，不编译）
gcc -E -I include -I include/mysql -I build/include \
    /tmp/test_mutex.c 2>/dev/null | grep -A 30 "mysql_mutex_lock"

# 你会看到 mysql_mutex_lock(&m) 展开为 inline_mysql_mutex_lock(&m, "/tmp/test.c", 3)
# inline_mysql_mutex_lock 又包含 start_mutex_wait → my_mutex_lock → end_mutex_wait
```

**这就是理解 PFS 编织的终极技巧。** 任何嵌套宏，gcc -E 一下全清楚。

---

## 9. 推荐学习路线（分阶段执行，不要一次性补完）

**阶段一：能跑通一个调用链（第 1 周）**
```
1. gcc -E 展开 mysql_mutex_lock → 理解三层宏
2. gdb 在 mem_heap_create_block 设断点 → 看三种 type 分别走哪个分支
3. ctags 生成索引 → 能在 vim/vscode 里跳转定义
```
**检验标准**：能从 `mem_heap_alloc(heap, 100)` 一路追踪到 `ut::malloc_withkey` 或 `buf_block_alloc`。

**阶段二：理解设计层面（第 2-3 周）**
```
4. 学 Strategy 模式 → 去读 PSI vtable
5. 学 Template Method → 去读 start/end 三部曲
6. 学 RAII → 去读 Scoped_heap
7. 学 Observer → 去读 Delegate+FOREACH_OBSERVER
```
**检验标准**：看到任意一个 PFS 埋点，能说出用了哪几个设计模式。

**阶段三：理解底层原理（第 4-5 周）**
```
8. 学虚拟内存/mmap → 去读 large_page_alloc
9. 学 CAS/memory_order → 去读 free_block_ptr 的原子操作
10. 学分支预测/cache line → 理解 m_enabled 为什么零开销
```
**检验标准**：能解释 PFS 关闭时 `if (m_psi->m_enabled)` 为什么只需 ~1 cycle。

**阶段四：使用工具验证理解（持续）**
```
11. git blame 追溯关键设计引入的 commit
12. perf 采样看真正的热点路径
13. gdb 在 event 记录点设断点，手工验证 ring buffer 写入
```
**检验标准**：能写一个简单的 SQL 查询 PFS 表，并解释背后发生了什么。

---

## 10. "读不懂"的自诊断流程

当你遇到一段看不懂的代码时，按以下顺序排查：

```
看不懂代码
    │
    ├─ 是语法层面（不认识某个 C/C++ 关键字/运算符）？
    │   → 查 cppreference.com
    │
    ├─ 是宏展开后看不懂（一堆嵌套的 #define）？
    │   → gcc -E 展开看实际代码
    │
    ├─ 是不知道这段代码"为什么存在"（看懂了语法但不理解设计）？
    │   → 查 [`pfs.md`](pfs.md) 的设计模式各节，看是哪种设计模式
    │
    ├─ 是不知道底层 OS/CPU 行为？
    │   → 查本篇第五~七域，补对应 OS/并发/体系结构知识
    │
    ├─ 是不知道业务动机（为什么需要这个功能）？
    │   → git log + git blame 看 commit message
    │   → 在 [`pfs.md`](pfs.md) / [`memory.md`](memory.md) 中找对应理论解释
    │
    └─ 都不是？
        → 很可能是一段历史遗留代码或为了兼容性的 workaround
        → git blame 看是谁写的、什么时间写的、commit message 怎么说的
```

---

## 11. 必读参考资料清单（按需取用）

| 类别 | 书籍/资料 | 关联章节 | 优先级 |
|------|---------|---------|-------|
| 设计模式入门 | 《Head First Design Patterns》(Freeman, 2004) | 4.1-4.4 | ⭐⭐⭐⭐⭐ 最推荐入门 |
| 设计模式参考 | 《Design Patterns》(GoF, 1994) | 4.1-4.4 | ⭐⭐⭐ 参考手册 |
| C 预处理器 | GCC Manual §3 (Macros) | 4.0.3 | ⭐⭐⭐⭐⭐ 必读 |
| C++ 核心 | 《The C++ Programming Language》4th Ed. (Stroustrup) §5 RAII, §34 unique_ptr, §28 template | 4.0.3 | ⭐⭐⭐⭐ |
| C++ 并发 | 《C++ Concurrency in Action》(Williams, 2nd Ed.) §5 内存模型, §7 无锁 | 4.0.6 | ⭐⭐⭐⭐ |
| 体系+OS | 《Computer Systems: A Programmer's Perspective》(CSAPP) §4 处理器, §6 存储器, §9 虚拟内存 | 4.0.5, 4.0.7 | ⭐⭐⭐⭐⭐ 一本覆盖 50% |
| OS 深入 | 《Advanced Programming in the UNIX Environment》(APUE) §14 mmap | 4.0.5 | ⭐⭐⭐ |
| 分配器理论 | Knuth TAOCP Vol.1 §2.5; Knowlton CACM 1965; Bonwick USENIX 1994 | 第二章 | ⭐⭐⭐ |
| MySQL 内幕 | 《MySQL 技术内幕: InnoDB 存储引擎》(姜承尧, 第2版) | 全局 | ⭐⭐⭐⭐ 中文首选 |
| MySQL 源码 | 官方 Developer Guide: storage/perfschema/README（另见 [`pfs.md`](pfs.md)） | 第一、三章 | ⭐⭐⭐⭐ |

---

## 设计模式与 MySQL 源码完整对照表

| 设计模式 | 中文 | GoF 分类 | MySQL 中的位置 | 一句话说明 |
|---------|------|---------|---------------|-----------|
| **Strategy** | 策略 | 行为型 | PSI vtable、mem_heap type 分支 | 同一接口，多种算法可替换 |
| **Template Method** | 模板方法 | 行为型 | `start_x → do_x → end_x` 宏展开 | 固定骨架，可变步骤 |
| **Observer** | 观察者 | 行为型 | `Delegate` + `xxxx_observer` 回调 | 解耦事件源与处理器 |
| **Factory Method** | 工厂方法 | 创建型 | `PSI_bootstrap::get_interface()` | 根据版本号创建对应 ABI |
| **Facade** | 外观 | 结构型 | `mysql_mutex_t` 统一接口 | 隐藏多子系统的复杂性 |
| **Bridge** | 桥接 | 结构型 | PSI 抽象 ↔ PFS 具体实现 | 抽象与实现独立演化 |
| **Adapter** | 适配器 | 结构型 | `Mem_root_allocator<T>` | Arena 分配器 → STL 接口 |
| **Decorator** | 装饰器 | 结构型 | `ut::malloc_withkey` | malloc + PFS 统计 |
| **Microkernel** | 微内核 | 架构¹ | MySQL Plugin 体系 | 核心最小化，功能插件化 |
| **RAII** | RAII | C++ 惯用法² | `Scoped_heap` | 自动管理资源生命周期 |

> ¹ Microkernel 不属于 GoF 23，出自 POSA（Pattern-Oriented Software Architecture）系列。  
> ² RAII 不属于 GoF 23，是 C++ 最重要的资源管理惯用法（idiom），重要性不亚于任何 GoF 模式。

## 推荐学习资源与路线

| 顺序 | 资源 | 说明 |
|------|------|------|
| 1 | 《Head First Design Patterns》(Freeman, 2004) | **最推荐的入门书**——图文并茂，用 Java 示例但思想跨语言。先看 Strategy、Observer、Template Method、Factory、Facade、Adapter、Bridge 这几章 |
| 2 | 中文博客：搜索 "C 语言实现设计模式" | Strategy 在 C 中用函数指针实现、Observer 在 C 中用回调函数实现——MySQL 正是这样做的 |
| 3 | GoF《Design Patterns》(1994) | 经典但抽象，适合作为参考手册而非通读 |
| 4 | POSA Vol.1 (Buschmann, 1996) | 架构模式：Microkernel 的权威定义 |
| 5 | 《API Design for C++》(Reddy, 2011) | Pimpl、Handle-Body、Factory 在 C++ 中的工业实践 |

**针对 MySQL 源码的阅读建议：**
- 先学 Strategy + Template Method → 立刻去读 `mysql_mutex.h`，你会发现所有埋点宏都是这两个模式的组合
- 再学 RAII → 去读 `Scoped_heap`，理解为什么 C++ 代码中看不到 `mem_heap_free` 调用
- 然后学 Observer → 去读 `rpl_handler.cc` 的 `FOREACH_OBSERVER` 宏
- 最后学 Microkernel → 理解 MySQL 为什么是"插件海洋"

---

## 核心术语中英文对照（设计模式）

| 英文 | 中文标准译法 | GoF 分类 | 说明 |
|------|-------------|---------|------|
| **Design Pattern** | 设计模式 | — | 对软件设计中反复出现的问题的通用可复用解决方案。GoF 1994 定义了 23 种经典模式 |
| **Strategy Pattern** | 策略模式 | 行为型 | 定义一系列算法并使其可互换。MySQL 中 PSI vtable / mem_heap type 分支是其 C 语言版本 |
| **Template Method** | 模板方法模式 | 行为型 | 定义算法骨架，延迟步骤到子类。PFS 的 start→do→end 三部曲是宏展开的编译期模板方法 |
| **Observer / Publish-Subscribe** | 观察者模式 / 发布-订阅模式 | 行为型 | 定义一对多依赖，当对象状态变化时通知所有依赖者。MySQL 复制 Delegate 是其实现 |
| **Factory Method** | 工厂方法模式 | 创建型 | 定义创建对象的接口，让子类决定实例化哪个类。PSI ABI `get_interface(version)` 是其实现 |
| **Facade** | 外观模式 / 门面模式 | 结构型 | 为子系统提供统一高层接口。`mysql_mutex_t` 隐藏了平台锁 + PFS + debug 三个子系统 |
| **Bridge** | 桥接模式 | 结构型 | 分离抽象与实现，让两者独立变化。PSI 接口定义与 PFS 实现的分层 |
| **Adapter** | 适配器模式 | 结构型 | 将一个接口转换为另一个接口。`Mem_root_allocator<T>` 适配 arena 分配器为 STL allocator |
| **Decorator** | 装饰器模式 | 结构型 | 动态地为对象添加额外职责。`ut::malloc_withkey` 对 malloc 透明增加 PFS 统计 |
| **RAII** (Resource Acquisition Is Initialization) | 资源获取即初始化 | C++ 惯用法¹ | 将资源生命周期绑定到对象作用域。`Scoped_heap` 通过 `unique_ptr` + 自定义 deleter 实现 |
| **vtable** (Virtual Function Table) | 虚函数表 | 底层机制 | C++ 虚函数 / C 语言策略模式的底层实现机制。函数指针数组 |
| **Deleter / Custom Deleter** | 自定义删除器 | C++ 惯用法 | `std::unique_ptr<T, Deleter>` 的第二个模板参数，用于非标准的资源释放方式 |
| **Inheritance** (白盒复用) | 继承（白盒复用） | 面向对象基础 | 子类继承父类的接口和实现。需了解父类内部细节 |
| **Composition / Delegation** (黑盒复用) | 组合 / 委托（黑盒复用） | 面向对象基础 | 对象持有另一个对象的引用并委托调用。只需知道接口，不需知道内部实现。GoF 提倡"优先使用组合而非继承" |

> ¹ RAII 不属于 GoF 23，由 Bjarne Stroustrup 在《The C++ Programming Language》中提出，是 C++ 独有的、最重要的资源管理惯用法。

#### 设计模式中文检索关键词

| 你想了解 | 用这些关键词搜索 |
|---------|---------------|
| 设计模式入门 | "Head First 设计模式" / "设计模式 通俗易懂" |
| C 语言如何实现设计模式 | "C 语言 设计模式" / "函数指针 策略模式" / "C语言 观察者模式" |
| MySQL 中的设计模式 | "MySQL 源码 设计模式" / "mysql plugin 设计模式" |
| 策略模式详解 | "策略模式 C++" / "Strategy Pattern vtable" |
| C++ RAII | "RAII C++" / "资源获取即初始化" / "unique_ptr 自定义删除器" |
| GoF 23 种 | "GoF 23种设计模式" / "Design Patterns Gamma" |

---

## 学术论文索引（软件工程与设计模式）

| 论文/著作 | 作者 | 会议/出版社 | 年份 | MySQL 关联 |
|---------|------|----------|------|-----------|
| *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF) | Gamma, Helm, Johnson, Vlissides | Addison-Wesley | 1994 | **设计模式圣经**：Strategy、Observer、Template Method、Factory、Facade、Adapter、Bridge、Decorator——MySQL PFS 和内存管理模块使用了其中至少 8 种。Strategy（PSI vtable）、Observer（Delegate）、Template Method（start/do/end）是最核心的三种 |
| *Pattern-Oriented Software Architecture (POSA) Vol.1* | Buschmann, Meunier, Rohnert, Sommerlad, Stal | Wiley | 1996 | **Microkernel（微内核）模式**正式命名：核心系统 + 插件扩展。MySQL Server 为微核心，存储引擎/PFS 为插件 |
| *The C++ Programming Language, 4th Ed.* | Bjarne Stroustrup | Addison-Wesley | 2013 | **RAII 惯用法**的权威出处：资源获取即初始化，C++ 最重要的资源管理思想。Scoped_heap 的 `unique_ptr<T, custom_deleter>` 是其应用 |
| *Refactoring: Improving the Design of Existing Code* | Martin Fowler | Addison-Wesley | 1999 | 重构与设计模式的关系：从"代码坏味道"到设计模式的演进路径 |
| *Head First Design Patterns* | Freeman, Freeman | O'Reilly | 2004 | **最佳入门书**：用 Java 示例图文并茂讲解 GoF 模式，适合首次接触设计模式的读者 |

## 计算机科学主要领域与顶级会议

#### 系统（Systems）——PFS 与内存管理的核心领域

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **SOSP** | Symposium on Operating Systems Principles | 操作系统最高殿堂，两年一届 | Anderson 1997 持续 profiling |
| **OSDI** | Operating Systems Design and Implementation | 与 SOSP 同级，交替举办 | 系统设计与实现 |
| **USENIX ATC** | USENIX Annual Technical Conference | 通用系统顶会 | **DTrace 2004** 零开销探针 |
| **EuroSys** | European Conference on Computer Systems | 欧洲系统顶会 | jemalloc 2011 |
| **ASPLOS** | Architectural Support for Programming Languages and Operating Systems | 体系+语言+OS交叉 | **Hoard 2000** 多线程 malloc |

#### 编程语言（Programming Languages）——AOP 的学术根基

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **ECOOP** | European Conference on Object-Oriented Programming | 面向对象 | **AOP 1997** Kiczales et al. |
| **PLDI** | Programming Language Design and Implementation | PL 实现顶会 | **Pin 2005** 动态二进制插桩 |
| **OOPSLA** | Object-Oriented Programming, Systems, Languages & Applications | 面向对象系统 | AspectJ 相关论文 |

#### 数据库（Databases）——MySQL 本身的归属

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **SIGMOD** | ACM SIGMOD Conference | 数据库第一会 | 查询引擎、存储引擎论文 |
| **VLDB** | Very Large Data Bases | 与 SIGMOD 同级 | InnoDB、B+tree、AHI 论文 |
| **ICDE** | IEEE International Conference on Data Engineering | 数据库三大会之一 | 数据工程 |

#### 体系结构（Architecture）

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **ISCA** | International Symposium on Computer Architecture | 体系结构第一会 | Smith 1981 / Yeh & Patt 1991 分支预测 |

#### 软件工程（Software Engineering）

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **ICSE** | International Conference on Software Engineering | 软工第一会 | 可观测性框架设计、profiling 方法学 |
| **FSE/ESEC** | Foundations of Software Engineering | 与 ICSE 同级 | 软件诊断、性能分析 |

#### 经典期刊

| 期刊 | 全称 | 本文关联 |
|------|------|---------|
| **CACM** | Communications of the ACM | Knowlton 1965 buddy system、Parnas 1972 信息隐藏 |
| **TAOCP** | Knuth, *The Art of Computer Programming* | Vol.1 (1968) buddy/bump allocator 形式化分析 |

#### 工业会议

| 会议 | 本文关联 |
|------|---------|
| **BSDCan** | jemalloc 2006 |

#### 领域交叉关系图

```
         Systems（SOSP/OSDI/ATC/EuroSys）← DTrace, jemalloc
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
PL(ECOOP/PLDI)  Databases(SIGMOD/VLDB)  Architecture(ISCA)
AOP 1997        InnoDB/B+tree/AHI        分支预测理论
    │                    │
    └────────┬───────────┘
             │ Hoard (ASPLOS 2000)
             ▼
    Software Engineering (ICSE/FSE)
    Observability, Profiling
```

---

> **说明**：以上仅列出与我们讨论相关的领域。不涉及的领域包括：网络（SIGCOMM）、安全（CCS/S&P/USENIX Sec/NDSS）、理论（STOC/FOCS）、AI/ML（NeurIPS/ICML/ICLR）、CV（CVPR/ICCV）、NLP（ACL/EMNLP）、HCI（CHI/UIST）、分布式（PODC/DISC）、存储（FAST/MSST）。

---

## 共享设计哲学（跨模块通用）

| 设计原则 | 思想/理论来源 | MySQL 中的实现 |
|---------|-------------|---------------|
| **Observer（观察者）** | GoF 1994 行为型模式 | Replication Delegate：`add_observer()`/`remove_observer()`/`FOREACH_OBSERVER` 通知循环 |
| **Microkernel（微内核）** | POSA 1996 架构模式 | MySQL Server 核心 + 插件海洋：PFS、InnoDB、复制等均以统一插件接口集成 |

---

*本目录文档均基于 MySQL 8.0 源码分析撰写，力求在数据结构、分配算法、学术理论、工程实现四个层面提供完备的参考。全部代码引用来自项目中实际存在的文件，所有论文引用均可通过会议名称和年份检索验证。*

---

## 参考

> 本文结论基于 MySQL 8.0.39 源码；设计模式与论文索引用于回溯思想来源，具体机制以源码为准。

**相关文档**

- 本目录文件索引：[`README.md`](README.md)
- Performance Schema（AOP 编织、Strategy/Template Method 的实际落地）：[`pfs.md`](pfs.md)
- 内存分配器矩阵（Bump/Arena/Buddy/Slab 理论与实现）：[`memory.md`](memory.md)
- 变量体系：[`variables.md`](variables.md)　|　数据编码：[`encoding.md`](encoding.md)

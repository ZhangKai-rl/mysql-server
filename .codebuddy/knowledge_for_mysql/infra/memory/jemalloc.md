# jemalloc 与 MySQL：生产分配器（接入、账本、调优与 Profiling）

> ★ **本篇是内存主题的生产重点篇**：Oracle 官方二进制、Percona Server、MariaDB 及各主流发行版包**默认链接 jemalloc**，生产 MySQL 的进程级分配器基本就是它。本篇回答生产 DBA/内核研发最关心的四个问题：
> 1. **它是怎么进到 mysqld 里的**（接入机制，含 `--malloc-lib` 的今昔）；
> 2. **RSS 为什么涨上去降不下来**（allocated/active/mapped/resident/retained 四层次账本 + decay 机制）；
> 3. **怎么观测与调优**（mallctl/stats、MALLOC_CONF、OOM 诊断流程）；
> 4. **PFS 查不到时怎么定位**（jeprof profiling 实操）。
>
> jemalloc 作为分配器本身的**算法设计**（segregated-fit、size class、tcache、arena）见 [allocator.md](allocator.md) 2.5 节；
> PFS 内存埋点机制见 [allocator.md](allocator.md)「可观测性交汇」；
> 全局内存架构与预算见 [memory_architecture.md](memory_architecture.md)。
>
> 基于本仓库 MySQL 8.0.39 源码（`mysys`、`sql`、`storage/innobase`、`cmake` 等模块）+ jemalloc 5.x 官方文档 + 生产案例。

## 目录

- [摘要](#摘要)
- [一、为什么生产是 jemalloc](#一为什么生产是-jemalloc)
- [二、接入机制：jemalloc 怎么进入 mysqld](#二接入机制jemalloc-怎么进入-mysqld)
- [三、RSS 四层次账本：为什么内存"降不下来"](#三rss-四层次账本为什么内存降不下来)
- [四、生产观测：mallctl / stats 与 OOM 诊断流程](#四生产观测mallctl--stats-与-oom-诊断流程)
- [五、MALLOC_CONF 调优](#五malloc_conf-调优)
- [六、内存 Profiling 实操：PFS 查不到时怎么办](#六内存-profiling-实操pfs-查不到时怎么办)
- [七、THP 与大页：显式大页 vs 透明大页](#七thp-与大页显式大页-vs-透明大页)
- [八、生产最佳实践清单](#八生产最佳实践清单)
- [九、常见误区澄清](#九常见误区澄清)
- [参考](#参考)

---

## 摘要

生产 MySQL 的进程级分配器几乎都是 jemalloc：官方二进制、Percona、MariaDB 默认链接，发行版包（Debian/Ubuntu/RHEL 的 mysql 包）亦然。选它不是因为它"更快"，而是三点工程价值：**多核可扩展的 arena 模型**（高并发分配不塌）、**成熟的 decay 归还机制**（dirty → muzzy → 归还 OS，碎片与 RSS 可控）、**`prof:true` 采样 profiling + `mallctl` 账本接口**（内存问题可观测、可定位——这是 glibc malloc 给不了的）。理解生产内存现象的关键是 jemalloc 的**四层次账本**：`allocated`（应用实际持有）≤ `active`（分配器占用的页）≤ `mapped`（虚拟映射），RSS（`resident`）在这之上还叠加 decay 延迟——"RSS 涨上去降不下来"绝大多数不是泄漏，而是 dirty page 还没过 `dirty_decay_ms` 的归还窗口。

---

## 一、为什么生产是 jemalloc

### 1.1 事实：主流发行都默认 jemalloc

- **Oracle 官方二进制**：默认链接 jemalloc（发行工程侧行为，源码树只留下 `-DWITH_JEMALLOC` 自编译路径，见 2.4）；
- **Percona Server / MariaDB**：默认 jemalloc；
- **Debian/Ubuntu/RHEL 的 `mysql` 包**：链接 `libjemalloc`。

自编译且不指定分配器时，落到 glibc 的 ptmalloc2——这是"性能测试与生产不一致"的常见来源之一：**你的生产在 jemalloc 上，你的压测可能在 ptmalloc 上**。

### 1.2 MySQL 分配特征与 jemalloc 的匹配

MySQL 的分配是**双峰分布**，恰好卡在 jemalloc 的设计甜区：

| 分配形态 | MySQL 里的典型 | jemalloc 对应机制 |
|----------|----------------|-------------------|
| **海量短命小对象**（KB 级，语句内分配释放） | THD `main_mem_root`、`mem_heap` 块、解析器 AST、`Item` 树 | size class + **thread cache（tcache）无锁分配**——每个线程独立缓存，不碰全局锁 |
| **中块**（KB~MB） | sort/join buffer、binlog cache、row buffer | arena → bin → slab，锁竞争按 arena 分片 |
| **巨型常驻**（GB 级） | BP chunk（mmap 大页）、TempTable MMAP | `oversize_threshold`（默认 8MB）以上直接 mmap，绕过 size class，free 即 munmap |

glibc ptmalloc2 的问题在于：多线程下用单一主 arena + 多个动态 arena，锁竞争与"内存粘滞"（arena 间不均衡导致 RSS 虚高）在高并发时放大。tcmalloc 虽快，但**RSS 阶梯式增长**（per-thread cache 激进缓存、归还机制不如 decay 精细）、**大页支持弱**（MAP_HUGETLB 路径不成熟）。jemalloc 的三段机制（tcache/arena/mmap）与 decay 归还正好覆盖 MySQL 三种形态，且碎片可控——这就是官方选它的工程理由。

### 1.3 何时考虑换 tcmalloc / glibc

- **换 tcmalloc**：极端高并发小对象吞吐基准里 tcmalloc 有时略优；但 MySQL 场景收益不明确，且失去 `prof` 生态，不推荐作为默认。
- **换 glibc**：ASAN/LSAN 调试构建必须用系统 malloc（CMakeLists.txt:1207-1210 明确互斥）；极小内存（<512MB）容器里 jemalloc 的元数据开销占比略高。生产调试构建以外不建议。

---

## 二、接入机制：jemalloc 怎么进入 mysqld

### 2.1 Linux：链接期替换（无任何运行时代码）

**先纠正一个流传已久的认知**：8.0 的 mysqld 在 Linux 上**没有** dlopen 加载 malloc 库的代码——全库搜 `dlopen`/`RTLD` 在 mysys 下 0 匹配，jemalloc 版本符号校验（`mallctl`/`jemalloc_version`）也 0 匹配。接入就是最朴素的**链接期替换**：

```
mysqld 可执行文件的 malloc/free/calloc/realloc 符号
  └─ 链接时被 jemalloc 库定义（-ljemalloc 放在链接行最后）
       └─ 所有 my_malloc / ut::malloc 最终落到 je_malloc
```

cmake 侧的机制（`cmake/malloc_utils.cmake`）：

```cmake
# CMakeLists.txt:860-880
OPTION(WITH_TCMALLOC "Use tcmalloc rather than builtin malloc/free etc." OFF)
OPTION(WITH_JEMALLOC "Use jemalloc rather than builtin malloc/free etc." OFF)
IF(WITH_JEMALLOC)
  FIND_MALLOC_LIBRARY(jemalloc)
ENDIF()
IF(WITH_JEMALLOC AND (WITH_TCMALLOC OR WITH_TCMALLOC_DEBUG))
  MESSAGE(FATAL_ERROR ...)   # 两者互斥
```

```cmake
# cmake/malloc_utils.cmake:42-77（FIND_MALLOC_LIBRARY 要点）
FIND_LIBRARY(MALLOC_LIBRARY ${library_name})
# 找不到 → WARN_MISSING_SYSTEM_JEMALLOC()（提示 apt install libjemalloc-dev）
# 最后改写 CMAKE_CXX_LINK_EXECUTABLE 追加 -ljemalloc
# 注释（62-69 行）强调：必须保证是链接行最后一个库
```

要点：

1. **不 vendor**：全库 `*jemalloc*` 只有 4 个文件（`cmake/win_jemalloc.cmake`、`include/jemalloc_win.h`、`mysys/my_nojemalloc.cc`、`sql/enable_win_jemalloc.cc`），没有 `extra/jemalloc`。对照：tcmalloc 有 bundled 版本（`extra/gperftools/gperftools-2.15/`，CMakeLists.txt:2028）。
2. **ASAN 互斥**：CMakeLists.txt:1207-1210 明确 ASAN/LSAN 与 jemalloc/tcmalloc 不能同用（sanitizer 自带分配器）。

### 2.2 Windows：LoadLibraryEx + 链接期符号覆盖

Windows 是运行时加载模型（8.0.28+ 引入）：

```cpp
// mysys/my_malloc.cc:200-260（init_malloc_pointers，仅 _WIN32）
LoadLibraryEx(jemalloc_dll_name, NULL, LOAD_LIBRARY_SEARCH_APPLICATION_DIR);
GetProcAddress 解析 je_malloc / je_calloc / je_realloc / je_free
→ pfn_malloc = tmp_je_malloc; ... using_jemalloc = true;

// my_malloc.cc:264-267
#define malloc(size) mysys::detail::pfn_malloc((size))   // 宏重定向
```

- DLL 名与导出符号：`include/jemalloc_win.h:38-42`（`jemalloc.dll`，`je_malloc/je_calloc/je_realloc/je_free`）。
- **开关的链接技巧**：`sql/enable_win_jemalloc.cc:43` 的 `use_jemalloc_allocations() { return true; }`（mysqld 链接它）vs `mysys/my_nojemalloc.cc:41` 返回 false（mysys 库默认）——linker 优先取目标文件里的符号，所以 mysqld 链接了 enable 版就启用，其他 mysys 消费方默认不启用。

### 2.3 --malloc-lib 的今昔：只剩 mysqld_safe 的 LD_PRELOAD

`--malloc-lib` 是 5.x/早期 8.0 的 mysqld 选项，8.0.39 中 mysqld 本体已无此选项（`sql/mysqld.cc` 全文件 0 匹配，`my_malloc_lib`/`my_use_mmap` 全局变量也已删除）。现存的只有 shell 层：

```sh
# scripts/mysqld_safe.sh
82: --malloc-lib=LIB           Preload shared library LIB if available
255: --malloc-lib=*) set_malloc_lib "$val" ;;
361-395: set_malloc_lib():
    # 校验必须为绝对路径且位于 /usr/lib | /usr/lib64 |
    #   /usr/lib/i386-linux-gnu | /usr/lib/x86_64-linux-gnu
    # 然后 add_mysqld_ld_preload "$malloc_lib"
```

即：`--malloc-lib` 现在的语义是 **mysqld_safe 在 exec mysqld 前把指定 .so 塞进 LD_PRELOAD**。这意味着：

- 直接以 `mysqld` 启动（不用 mysqld_safe）时，**这个选项不存在**；换分配器要自己在环境/服务单里设 `LD_PRELOAD`。
- 官方文档也把 `--malloc-lib` 归在 mysqld_safe 章节（6.3.2），而非 mysqld 选项章节——与本仓库源码一致。

**生产验证分配器是否生效**（不用迷信配置，直接看进程）：

```bash
ldd $(which mysqld) | grep -E 'jemalloc|tcmalloc'
# 或
grep -E 'jemalloc|tcmalloc' /proc/$(pidof mysqld)/maps | head -1
# 或查 PFS：SELECT * FROM sys.memory_global_by_current_bytes; 里 memory/innodb 条目数正常即插桩正常
```

### 2.4 my_malloc 的包装层：PSI 头 + 两条分配器路径

jemalloc 接管后，Server 层所有分配仍走 `my_malloc` 包装（`mysys/my_malloc.cc`；旧的 `include/my_malloc.h` 已删除）：

```cpp
// include/mysql/psi/mysql_memory.h:77-93（PSI 头结构）
struct my_memory_header {
  PSI_memory_key m_key;   // 插桩 key
  uint m_magic;           // PSI_MEMORY_MAGIC = 1234（越界检测）
  size_t m_size;          // 用户请求大小
  PSI_thread *m_owner;    // 归属线程
};
#define PSI_HEADER_SIZE 32
#define USER_TO_HEADER(P) ((my_memory_header *)(((char *)P) - PSI_HEADER_SIZE))
```

```cpp
// my_malloc.cc:56-58、364-392（有 PSI 编译时）
#define USE_MALLOC_WRAPPER
my_internal_malloc: 分配 PSI_HEADER_SIZE + size
  → 写 m_magic/m_size/m_key
  → PSI_MEMORY_CALL(memory_alloc)(key, raw_size, &mh->m_owner)
  → 返回 HEADER_TO_USER(mh)

my_malloc     → my_internal_malloc<redirecting_allocator>   // 走 jemalloc
my_std_malloc → my_internal_malloc<std_allocator>           // 走 glibc malloc
// 注释（64-67 行）：std:: 版给 OpenSSL 等不兼容 jemalloc 的场景
```

失败语义：`my_raw_malloc`（my_malloc.cc:310-346）——`size==0 → 1`；`MY_ZEROFILL → calloc`；失败时 `MY_WME` → `my_error(EE_OUTOFMEMORY)`、`MY_FAE` → `exit(1)`。

---

## 三、RSS 四层次账本：为什么内存"降不下来"

这是生产最常被问的问题：**大查询结束后 RSS 为什么不回落？是不是泄漏？** 答案在 jemalloc 的内存账本里——RSS 不是一层，是四层叠出来的。

### 3.1 四层次语义

```
stats.allocated  ≤  stats.active  ≤  stats.mapped
（应用持有的字节）  （分配器占用的页）  （虚拟映射总量）

stats.resident（≈RSS，物理驻留）介于 active 与 mapped 之间，
stats.retained = 分配器保留、既没用也暂未还给系统的虚拟空间
```

| 层次 | mallctl 名字 | 含义 | 生产意义 |
|------|--------------|------|----------|
| **allocated** | `stats.allocated` | 应用 `malloc` 实际拿到的字节合计 | 判断"应用是否真在涨"的基准线 |
| **active** | `stats.active` | 分配器为满足分配从 OS 拿的页（含 size class 的内部碎片） | `active - allocated` = 内部碎片；`active` 才是分配器真实占用 |
| **mapped** | `stats.mapped` | 虚拟地址空间映射总量（含 retained） | 虚拟内存上限 |
| **resident** | `stats.resident` | 物理驻留（RSS 的 jemalloc 口径） | 与 `/proc/PID/status` 的 VmRSS 对照 |
| **retained** | `stats.retained` | 已归还物理页但保留虚拟地址的空间 | 非物理占用，可忽略 |

**核心结论**：`active` 与 `allocated` 的差是碎片，`resident` 与 `active` 的差是 **decay 窗口里还没归还的 dirty/muzzy 页**——"RSS 降不下来"几乎都是后者。

### 3.2 decay 机制：dirty → muzzy → 归还 OS

释放的内存不立即还 OS，而是经历两段延迟（这就是 `dirty_decay_ms`/`muzzy_decay_ms` 两个参数）：

```
free 的页
  ↓
dirty page（内容还在，可被同 size class 立即复用）
  ↓ 经过 dirty_decay_ms（默认 10s）未被复用
muzzy page（madvise(MADV_FREE)：标记可回收，OS 内存压力大时先收它）
  ↓ 经过 muzzy_decay_ms（默认 10s）未被复用
归还 OS（MADV_DONTNEED / munmap）→ 进入 retained（虚拟地址保留）
```

所以一次大查询释放 2GB 后，RSS 最多拖 `dirty_decay_ms + muzzy_decay_ms`（默认约 20s）才回落——**这是设计行为，不是泄漏**。真正要警惕的是：

1. **`background_thread:false`**（jemalloc 默认关闭）：decay 扫描在**应用线程的分配/释放路径上同步做**，高负载下 decay 可能长期不推进，dirty 页堆积 → RSS 持续虚高（这是社区反复遇到的坑，很多发行版为 MySQL 默认开启 background_thread）；
2. **`dirty_decay_ms` 被调得过大**：换取"复用命中率"，代价是 RSS 回落更慢；
3. **真的泄漏**：`allocated` 持续单调上涨且 free 不掉——这时才轮到第四章的 jeprof。

### 3.3 碎片：active - allocated

jemalloc 的 size class 设计把内部碎片控制在 ~25% 内（见 allocator.md 2.5.2），但**多 arena + tcache 的缓存会放大 active**：每个线程 tcache 里的空闲块、每个 arena 的 slab 余量都不在 allocated 里。`active/allocated` 比值长期 > 1.5 说明分配模式与 size class 错配（比如大量恰好跨档位的固定大小对象），可通过 `lg_tcache_max`、`narenas` 调整或业务侧减少"伪大对象"。

---

## 四、生产观测：mallctl / stats 与 OOM 诊断流程

### 4.1 怎么把账本读出来

方式一：**mallctl 接口**（C 程序 / `gdb` attach 后 `call`）：

```c
size_t allocated, active, resident, retained;
size_t sz = sizeof(allocated);
mallctl("stats.allocated", &allocated, &sz, NULL, 0);
mallctl("stats.active",     &active,     &sz, NULL, 0);
mallctl("stats.resident",   &resident,   &sz, NULL, 0);
mallctl("stats.retained",   &retained,   &sz, NULL, 0);
```

方式二：**`malloc_stats_print` 输出**（`MALLOC_CONF=stats_print:true` 或在 gdb 里调）：

```
Allocated: 3.1 GiB, Active: 3.6 GiB, Resident: 4.2 GiB, Mapped: 4.5 GiB
```

方式三：**per-arena 明细**（定位是哪个 arena 的 dirty 堆积）：

```c
for (unsigned i = 0; i < narenas; ++i) {
  char buf[64];
  snprintf(buf, sizeof buf, "stats.arenas.%u.dirty_decay_ms", i);
  // 或读 stats.arenas.<i>.purged / nmadvise 判断 decay 是否在推进
}
```

### 4.2 OOM 诊断流程（生产分流表）

mysqld 被 OOM killer 打死 / RSS 告警时，按此分流：

```
vmRSS 与 jemalloc 账本对比
├─ vmRSS ≈ stats.allocated（且 allocated 在涨）
│   → 应用真的在涨内存：查 PFS memory_summary（memory_architecture.md 五章）
│     → PFS 解释不了 → jeprof（本篇六章）
├─ vmRSS >> allocated，active 正常（≈ allocated × 1.2 内）
│   → dirty 页堆积：查 background_thread 是否开启、dirty_decay_ms 是否过大
│     → 临时手段：mallctl("arena.<i>.purge") 强制归还
├─ active >> allocated（碎片）
│   → size class 错配 / arena 过多：调 narenas、lg_tcache_max，或业务侧对象大小
├─ vmRSS 远大于 resident 口径（虚拟内存爆炸）
│   → mapped/retained 大：多为线程栈 + BP 延迟映射（见 memory_architecture.md 四章要点 5），
│     非物理占用，先看 VmRSS 而不是 VmSize
└─ 以上都不符合
    → 检查 TempTable mmap（memory/temptable/physical_disk）、第三方库（SSL/zlib）
      以及 THP 碎片化（本篇七章）
```

---

## 五、MALLOC_CONF 调优

jemalloc 的运行参数不走配置文件，走 **`MALLOC_CONF` 环境变量**（逗号分隔的 `key:value` 串，也可用 `mallctl` 运行时改）。MySQL 生产场景的配置：

| 选项 | 默认 | **生产建议** | 说明 |
|------|------|--------------|------|
| `background_thread` | **false** | **true** | ★ 最重要的一项。开启后 decay 由后台线程做，不再占用应用线程的分配/释放路径；不开的坑见 3.2 |
| `dirty_decay_ms` | 10000 | 10000~60000 | dirty 页复用窗口。调大提升复用率、减少 page fault；代价是 RSS 回落慢。内存紧张的环境调小（如 1000） |
| `muzzy_decay_ms` | 10000 | 同 dirty | muzzy → 归还 OS 的窗口 |
| `narenas` | 4×CPU（cap 1024） | 默认即可 | arena 过少锁竞争、过多内存占用上升；CPU > 64 时可按 NUMA 节点数微调 |
| `percpu_arena` | disabled | NUMA 可试 `percpu` | CPU 亲和绑定 arena，减少跨 NUMA 访存 |
| `oversize_threshold` | 8MB | 默认 | 超过直接 mmap 分配（绕过 size class）。MySQL 大块（BP 之外的大 buffer）走这里 |
| `metadata_thp` | disabled | `auto` | jemalloc 元数据是否用 THP。老内核有卡顿风险（见七章），5.x 新版本 `auto` 安全 |
| `lg_tcache_max` | 15（32KB） | 默认 | tcache 缓存上限，调大可减少 arena 锁竞争、代价是线程本地缓存占用 |

设置方式（生产三选一）：

```bash
# 1. systemd 服务单（推荐）
Environment=MALLOC_CONF=background_thread:true,dirty_decay_ms:10000,muzzy_decay_ms:10000

# 2. mysqld_safe 场景
export MALLOC_CONF="background_thread:true"

# 3. 验证生效（启动后）
cat /proc/$(pidof mysqld)/environ | tr '\0' '\n' | grep MALLOC_CONF
```

注意事项：

1. **启动前设置才稳**：`MALLOC_CONF` 在进程第一次分配前读取；运行期改要走 mallctl。
2. **版本敏感**：`percpu_arena`、`metadata_thp` 是 jemalloc 5.x 选项。确认版本：`strings /usr/lib/x86_64-linux-gnu/libjemalloc.so.2 | grep -E '^5\.[0-9]'`。
3. **先量后调**：用第四章的账本对比 allocated/active/resident 找出瓶颈层，再动对应参数；盲调 `narenas`/`lg_tcache_max` 的收益通常小于开 `background_thread`。

### 5.1 容器场景：MySQL 为什么同样会被 OOM-kill（与 DuckDB 主动归还方案对比）

容器/cgroup 里"数据库内部统计已回落、内核仍按 RSS 杀进程"是通用现象。DuckDB 在 2026-10 的 PR #26246 给出了**主动方案**：在**释放路径**（不是分配路径）上装 RSS 探针，当物理 RSS 接近 `memory_limit` 阈值（默认 10% 冗余安全带）时触发 `Allocator::Flush()`，直接 `madvise(..., MADV_DONTNEED)` 立即解除物理页映射，RSS 当刻下降——代价是这些页下次被访问时要重新缺页（零页填充），所以它只在逼近阈值时才付这个代价。

**MySQL 的对照事实（本仓库 8.0.39 源码核实）**：

| 对照项 | DuckDB（PR #26246） | MySQL 8.0.39 |
|--------|---------------------|--------------|
| RSS 水位感知 | ✅ 释放路径上的 RSS 探针 | ❌ **全库无 `VmRSS` / `/proc/self/statm` 读取**（0 匹配） |
| cgroup 限制感知 | ✅ | ❌ **全库无 `cgroup` / `memory.max` 读取**（仅 ndb test 与 bundled gperftools 里有） |
| `MADV_DONTNEED` 主动归还 | ✅ 触发式 | ❌ 全库 0 匹配 |
| madvise 的实际用途 | 归还物理内存 | **只用于 core dump 控制**：`buf0buf.cc:840-875` 的 `MADV_DONTDUMP`/`MADV_DODUMP`，配合 `innodb_buffer_pool_in_core_file` 决定 BP 是否进 core 文件（注释明言 "Dump core without large memory buffers"） |
| 大块真归还路径 | Flush | 只有 **BP 在线缩容**：`buf_pool_resize` → `deallocate_chunk` → `ut::free_large_page`（munmap） |
| 瞬态内存归还 | Flush | **完全交给 jemalloc decay**（dirty/muzzy 两段窗口）+ `background_thread` |

**两种哲学的差异**：DuckDB 是 OLAP——内存特征是"查询期峰值 → free → RSS 滞留"，痛点是**滞留**，所以要在释放路径上主动、即时归还；MySQL 是 OLTP——BP 是**故意常驻**的（大小由 `innodb_buffer_pool_size` 显式声明，运维知情），真正的滞留来源只是瞬态内存（mem_heap、sort/binlog/TempTable 释放后留在 jemalloc dirty 页），量级远小于 BP，用 decay 时间窗（默认 ~20s）消化足够，不需要改内核代码。

**因此 MySQL 在容器里的 OOM 主因与 DuckDB 不同**：不是"RSS 滞留在窗口内来不及跌"，而是 **BP 基线设得离 cgroup limit 太近 + 每连接内存 × 并发数叠加尖峰**（BP 默认 128MB 且不自动感知容器内存，容器部署必须显式估算）。应对是第四章 4.2 的分流 + 本章参数（开 `background_thread`、内存紧张时调小 `dirty_decay_ms`）；极端需要"立即归还"时可外部调 `mallctl("arena.<i>.purge")`——这就是 DuckDB 把它自动化之前的等价手工手段。

---

## 六、内存 Profiling 实操：PFS 查不到时怎么办

### 6.1 为什么需要 jemalloc profiling

PFS 的内存统计**只覆盖插桩内存**（走 `my_malloc` / `ut::malloc_withkey` 且 key 已注册的），还存在未插桩部分：第三方库（SSL、zlib）、TempTable 的 MMAP 文件映射、线程栈等。实际排查中常见"RSS 涨 5GB、`memory_summary_*` 只解释出 3GB"——差的 2GB 就需要进程级分配器 profile。

### 6.2 准备带 prof 的 jemalloc

发行版 jemalloc 通常**不带 profiling**（`--enable-prof` 是编译选项）。要编译一个：

```bash
git clone https://github.com/jemalloc/jemalloc && cd jemalloc
./autogen.sh --enable-prof --enable-stats
make -j && sudo make install   # 或只把 lib/libjemalloc.so 拷到服务器
```

### 6.3 启动 mysqld 时开启

```bash
export LD_PRELOAD=/path/to/libjemalloc.so
export MALLOC_CONF="prof:true,prof_accum:true,lg_prof_sample:20,prof_prefix:/tmp/jeprof.out"
mysqld_safe --malloc-lib=/path/to/libjemalloc.so &   # 或直接 LD_PRELOAD
```

参数含义：

| 参数 | 说明 |
|------|------|
| `prof:true` | 开启 heap profiling |
| `prof_accum:true` | dump 出的 heap 文件是**累计**分配而非当前存活——找"谁分配得多"（内存膨胀场景）比"当前存活"（泄漏场景）更常用 |
| `lg_prof_sample:20` | 每 `2^20`（1MB）采样一次调用栈。默认 19（512KB）；采样越密开销越大 |
| `prof_prefix` | heap dump 文件前缀 |
| `prof_gdump:true` | 进程退出时自动 dump 最终 heap |

### 6.4 抓取与分析

```bash
# 方法一：手动触发 dump（用 jeprof 或 mallctl）
# 方法二：依赖 prof_gdump 在退出时生成

jeprof --text /usr/sbin/mysqld /tmp/jeprof.out.*.heap | head -30
# 典型输出：按累计分配字节排序的函数调用栈
#   Total: 12.5 GB
#      8.2 GB  my_malloc
#      3.1 GB  ut::malloc_withkey
#      ...

# 对比两个时间点的 dump 找增量（内存持续增长场景）
jeprof --text --base=heap.t0.heap mysqld heap.t1.heap
```

公开案例（知乎《告别内存 OOM，解决 MySQL 内存增长问题》）：客户大事务下 RSS 异常增长，PFS 定位不出，用 jeprof 抓到是某路径的 `my_malloc` 大块分配——这正是"PFS 插桩覆盖不到/粒度不够"时 profile 的价值。

### 6.5 局限与边界

- **TempTable 的 RAM 部分走 `malloc`**（`temptable/memutils.h:151-157`，NUMA 编译时可能走 `numa_alloc_local`），链接 jemalloc 后被接管、prof 可见；**落到 MMAP 文件的部分绕过 malloc**，只能靠 PFS 的 `memory/temptable/physical_disk` 单独统计。
- **BP chunk 是 mmap 大页/匿名映射**（`ut::malloc_large_page_withkey`），不经 jemalloc 的 size class 路径，prof 里看不到——BP 本来就由 `innodb_buffer_pool_size` 精确控制，不需要 prof。
- 采样式 prof 对小对象高频分配有统计噪声；精确排查配合 `lg_prof_sample:17`（128KB）加大采样密度。

---

## 七、THP 与大页：显式大页 vs 透明大页

### 7.1 MySQL 自己的大页：显式 MAP_HUGETLB（与 THP 无关）

`--large-pages` 的完整链路（Linux）：

```
--large-pages[=ON]（mysqld 选项，sys_vars.cc:2404-2408，READ_ONLY）
  → opt_large_pages（sql/mysqld.cc:1265）
  → 启动时 my_get_large_page_size()（mysys/my_largepage.cc:63-80）
      └─ 读 /proc/meminfo 的 Hugepagesize（探测系统大页大小）
  → ha_innodb.cc:4923-4928：os_use_large_pages = opt_large_pages
  → buf_pool_t::allocate_chunk（buf0buf.cc:883-887）
      └─ ut::malloc_large_page_withkey(..., fallback_to_normal_page_t{})
           └─ large_page_aligned_alloc（detail/ut/large_page_alloc-linux.h:55-59）
                mmap(NULL, n_bytes, PROT_READ|PROT_WRITE,
                     MAP_PRIVATE | MAP_ANON | MAP_HUGETLB, -1, 0)
```

要点：

1. **走的是 hugetlbfs 显式大页**（`MAP_HUGETLB`），不是 THP——两者互不相干。用 `--large-pages` 前要 `sysctl vm.nr_hugepages` 预留好大页，否则 mmap 失败并**回退普通页**（ut0new.h:1357-1365 的回退逻辑）。
2. **无 memfd/shmget**（全库 0 匹配），就是最朴素的 `mmap + MAP_HUGETLB`。
3. **chunk 大小不强制大页倍数**：`ha_innodb.cc:4611-4620` 只有 `ut_ad` 断言（页大小/块大小对齐）；对齐由分配器兜底——`large_page_alloc-linux.h:53-54` 注释"mmap 会把 n_bytes 内部向上取整到大页倍数"。
4. macOS 固定 2MB superpage（`large_page_alloc-osx.h`），Windows 用 `VirtualAlloc + MEM_LARGE_PAGES`。
5. **源码没有 `MADV_NOHUGEPAGE` 处理**——MySQL 对 THP 没有任何主动防御（见下）。

### 7.2 THP：为什么社区共识是"数据库关闭它"

THP（Transparent Huge Pages）由内核在缺页时自动合并 4KB 页为 2MB 大页。对数据库的伤害链条（腾讯云《我们为什么要禁用 THP》，PingCAP 专栏）：

1. **数据库访存离散** → 吃不到 THP 的 TLB 收益；
2. THP 分配需**连续 2MB 物理内存**，找不到时缺页路径上同步做**直接内存规整（compact）**——perf 火焰图上表现为 `do_huge_pmd_anonymous_page` 的 Sys CPU 飙升；
3. 高阶内存占用**加速碎片化** → 触发直接内存回收 → 秒级延迟抖动。

排查手段：`sar -B` 看 `pgscand/s`、`/sys/kernel/debug/extfrag/extfrag_index`（order≥3 接近 1.000 = 碎片严重）、BPF 的 `compactsnoop`。

**配置建议**：

```bash
# 数据库场景直接关（内核 < 4.6，如 CentOS 7）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 内核 >= 4.6 折中方案：应用显式 madvise 才用（defer+madvise 避免分配路径上的同步规整）
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag
```

### 7.3 jemalloc 与 THP 的交互：一个已知 bug

jemalloc 有自己的 THP 策略（`opt.thp`，值可为 `default/always/never`，作用于 `mmap` 时的 `MADV_HUGEPAGE`）。已知问题：**jemalloc 的 `opt.thp:never` 并不能完全关掉 THP**（`base_map` 路径仍可能拿到 THP），修复见 jemalloc PR #1704。另有 `metadata_thp`（jemalloc 元数据是否用 THP）：老内核上 `always` 曾引发分配卡顿，5.x 新版 `auto` 才安全。对 MySQL 的意义：

- MySQL 的 BP 走 `MAP_HUGETLB` 显式大页，不经过 jemalloc，不受 `opt.thp` 影响；
- 其余 malloc 内存（THD、buffer、cache）都在 jemalloc 里，**最终防线还是 OS 层关 THP**——只靠 `MALLOC_CONF=opt.thp:never` 不够。

---

## 八、生产最佳实践清单

| # | 项 | 做法 | 依据 |
|---|----|------|------|
| 1 | **OS 层关 THP** | `enabled=never`（或新内核 `madvise`+`defer+madvise`） | 七章 7.2 |
| 2 | **开 background_thread** | `MALLOC_CONF=background_thread:true`（systemd `Environment=` 或 mysqld_safe 前 export） | 三章 3.2——防 dirty 堆积 |
| 3 | **启动前设 MALLOC_CONF** | 在服务单/脚本里，不要指望运行期改 | 五章 |
| 4 | **大页只给 BP** | `--large-pages=ON` + `vm.nr_hugepages` 预留；其他分配不掺和 | 七章 7.1 |
| 5 | **BP 之外靠 jemalloc 默认档** | 不调 `narenas`/`lg_tcache_max` 除非账本显示碎片/锁问题 | 五章 |
| 6 | **自编译必链 jemalloc** | `-DWITH_JEMALLOC=ON`，避免"压测 ptmalloc、生产 jemalloc"的不一致 | 一章 1.1 |
| 7 | **debug 构建用系统 malloc** | ASAN/LSAN 与 jemalloc 互斥（CMakeLists.txt:1207-1210） | 二章 2.1 |
| 8 | **RSS 告警先看账本再下结论** | vmRSS vs allocated/active/resident 分流（四章 4.2），别直接甩锅"泄漏" | 三章 |
| 9 | **PFS 解释不了的 RSS 用 jeprof** | `--enable-prof` 编译 + `prof:true,prof_accum:true` | 六章 |
| 10 | **升级 jemalloc 版本** | 用 5.2+（`metadata_thp:auto`、`percpu_arena` 稳定），避开 PR #1704 之前的 opt.thp 坑 | 七章 7.3 |

---

## 九、常见误区澄清

| 误区 | 事实 |
|------|------|
| "mysqld 有 `--malloc-lib` 选项" | **没有**。只剩 `mysqld_safe` 脚本的 LD_PRELOAD 机制（scripts/mysqld_safe.sh:361-395） |
| "mysqld 运行时 dlopen 加载 jemalloc 并校验版本" | Linux 上是**链接期替换**（`-ljemalloc`）；运行时加载只在 Windows（`LoadLibraryEx`）；版本符号校验全库 0 匹配 |
| "MySQL 源码里 vendor 了 jemalloc" | 没有。vendor 的是 tcmalloc（`extra/gperftools`）；jemalloc 纯外部 |
| "`--large-pages` 用的是 THP" | 不是。是 hugetlbfs 显式大页（`mmap + MAP_HUGETLB`），与 THP 无关 |
| "MySQL 源码会主动避开 THP" | 不会。全库无 `MADV_NOHUGEPAGE`；THP 要在 OS 层关 |
| "RSS 涨上去降不下来 = 泄漏" | 绝大多数是 decay 窗口 + background_thread 未开 + PFS/thread cache 保留的设计行为；先对比 allocated（三章 3.1）再下结论 |
| "TempTable 内存都在 PFS 里、prof 看不到" | RAM 部分走 `malloc`（jemalloc 可接管、prof 可见）；只有落到 MMAP 文件的部分绕过 malloc |
| "PFS 内存统计能覆盖进程全部内存" | 不能。只覆盖插桩分配；BP mmap、线程栈、第三方库、TempTable MMAP 都不在里面 |
| "jemalloc 默认配置就适合 MySQL" | 不适合。`background_thread` 默认关闭是社区反复踩的坑，生产应显式开启 |

---

## 参考

**源码（本仓库 8.0.39）**
- `cmake/malloc_utils.cmake`（FIND_MALLOC_LIBRARY / WARN_MISSING_SYSTEM_JEMALLOC）、`CMakeLists.txt`（WITH_JEMALLOC / ASAN 互斥）
- `cmake/win_jemalloc.cmake`、`include/jemalloc_win.h`、`mysys/my_nojemalloc.cc`、`sql/enable_win_jemalloc.cc`
- `mysys/my_malloc.cc`（my_internal_malloc / init_malloc_pointers / my_raw_malloc）、`include/mysql/psi/mysql_memory.h`（my_memory_header）
- `scripts/mysqld_safe.sh`（set_malloc_lib / LD_PRELOAD）
- `storage/innobase/include/detail/ut/large_page_alloc-linux.h`（mmap + MAP_HUGETLB）、`-osx.h`、`-win.h`
- `mysys/my_largepage.cc`（/proc/meminfo 探测）、`sql/sys_vars.cc`（large_pages sysvar）、`storage/innobase/handler/ha_innodb.cc`（os_use_large_pages 传递）
- `storage/temptable/include/temptable/memutils.h`（RAM 走 malloc / MMAP 文件）、`storage/innobase/buf/buf0buf.cc`（madvise_dont_dump）

**外部材料**
- MySQL 8.0 Reference Manual: [7.1.7 Server Command Options](https://dev.mysql.com/doc/refman/8.0/en/server-options.html)（--large-pages）、[6.3.2 mysqld_safe](https://dev.mysql.com/doc/refman/8.0/en/mysqld-safe.html)（--malloc-lib）
- 知乎《告别内存 OOM，解决 MySQL 内存增长问题》（jeprof 定位 MySQL 内存增长实操案例），<https://zhuanlan.zhihu.com/p/701482304>
- 腾讯云开发者社区（PingCAP 专栏）《我们为什么要禁用 THP》，<https://cloud.tencent.com/developer/article/1727260>
- jemalloc 官方文档（opt.prof / opt.thp / opt.dirty_decay_ms / mallctl stats.* 语义）、jemalloc PR #1704（opt.thp:never 仍使用 THP 的修复）
- 菜鸟笔记《jemalloc 内存的四种状态》（decay 流转与 MADV_FREE/MADV_DONTNEED），<https://www.coonote.com/linux-performance-opt/four-states-jemalloc-memory.html>

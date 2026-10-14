# 系统层与外部工具：MySQL 看不见的地方怎么看

> **边界**：本篇讲**进程之外的观测手段**（栈、CPU、内核、IO、网络、监控平台）。
> - MySQL 内部的手段见 [`sys_schema.md`](sys_schema.md)、[`pfs/pfs_events.md`](pfs/pfs_events.md)、[`processlist.md`](processlist.md)、[`show_commands.md`](show_commands.md)
> - ★ 这些工具与 MySQL 的**唯一桥梁**是 `performance_schema.threads.THREAD_OS_ID`（见 [`processlist.md`](processlist.md) 5.4）
>
> **一句话**：MySQL 的 SQL 层手段回答"数据库在等什么"，系统层工具回答"**进程在 CPU/内核/IO 上到底在干什么**"——后者是前者的盲区。

## 目录

- [一、概述](#一概述)
- [二、★ 桥梁：把 MySQL 线程映射到 OS 线程](#二-桥梁把-mysql-线程映射到-os-线程)
- [三、栈分析：`pstack` / `gdb` / `pt-pmp`](#三栈分析pstack--gdb--pt-pmp)
- [四、CPU 采样：`perf` / 火焰图 / `eBPF`](#四cpu-采样perf--火焰图--ebpf)
- [五、系统资源：`iostat` / `vmstat` / `strace` / `lsof` / `tcpdump`](#五系统资源iostat--vmstat--strace--lsof--tcpdump)
- [六、监控平台：PMM](#六监控平台pmm)
- [七、★ 代价对照（哪些会暂停进程）](#七-代价对照哪些会暂停进程)
- [八、决策树：什么时候用哪个](#八决策树什么时候用哪个)
- [九、可观测性](#九可观测性)
- [十、Misc（易混淆点）](#十misc易混淆点)
- [参考](#参考)

---

## 一、概述

### 是什么

系统层工具在 **MySQL 进程之外**观测——不依赖任何 PFS instrument、不受 instrument 开关影响、不需要 MySQL 配合。

### 为什么需要（MySQL 内部手段的三个盲区）

| 盲区 | 说明 | 谁补 |
|---|---|---|
| **CPU 热点在哪个函数** | PFS 只能告诉你"等了多久"，不能告诉你"CPU 烧在哪一行" | `perf` / 火焰图 |
| **内核态 / 系统调用** | PFS 不覆盖内核（如 `futex`、页错误、网络栈） | `perf` / eBPF / `strace` |
| **进程已 hang 住** | SQL 连不上、PFS 打不开时，SQL 层手段全部失效 | `gdb` / core dump / `pstack` |

### 版本演进

- 8.0.28+：`performance_schema.threads` 增加了 `THREAD_OS_ID` 之外的**内存列**（`CONTROLLED_MEMORY` 等），让"MySQL 内部看内存"与"OS 层看 RSS"能对账
- 8.0：`SHOW ENGINE INNODB STATUS` 的 **spin 计数已恒为 0**（插桩移除），⇒ 自旋类观测**只能靠系统层**

---

## 二、★ 桥梁：把 MySQL 线程映射到 OS 线程

这是本篇最关键的一节——**没有这座桥，系统层工具看到的是一堆无从归属的 tid**。

### 2.1 三个 ID 的关系

| ID | 含义 | 在哪看 |
|---|---|---|
| `THREAD_ID` | PFS 内部单调递增，**连接断开后不复用** | `performance_schema.threads.THREAD_ID` |
| `PROCESSLIST_ID` | `SHOW PROCESSLIST` 的 `Id`，**`KILL` 用的这个** | `threads.PROCESSLIST_ID` |
| **`THREAD_OS_ID`** | **OS 层的 tid**（Linux `gettid()`） | `threads.THREAD_OS_ID` |

源码（`storage/perfschema/pfs.cc`）：

```c
void pfs_set_thread_os_id_vc(PSI_thread *thread) {
  auto *pfs = reinterpret_cast<PFS_thread *>(thread);
  if (unlikely(pfs == nullptr)) return;
  pfs->m_thread_os_id = my_thread_os_id();
}
```

### 2.2 标准操作

```sql
-- ① 从 PROCESSLIST_ID（你看到的 Id）拿到 OS tid
SELECT thread_id, processlist_id, thread_os_id, name, type
  FROM performance_schema.threads
 WHERE processlist_id = 123;

-- ② 反过来：知道 tid 想找是哪个连接
SELECT processlist_id, thread_os_id, processlist_user, processlist_host,
       processlist_state, processlist_info
  FROM performance_schema.threads
 WHERE thread_os_id = 45678;

-- ③ 一次导出全部映射（压测/排障时留档）
SELECT processlist_id, thread_os_id, name
  FROM performance_schema.threads
 WHERE type = 'FOREGROUND' AND processlist_id IS NOT NULL
 ORDER BY processlist_id;
```

### 2.3 拿到 tid 之后能做什么

| 工具 | 用法 | 看什么 |
|---|---|---|
| `perf` | `perf top -t <tid>` / `perf record -t <tid> -g -- sleep 30` | 该连接的 CPU 热点 |
| `pidstat` | `pidstat -t -p <mysqld_pid> 1` | 各线程 CPU / IO |
| `ps` | `ps -L -p <mysqld_pid>`（LWP 列 = tid） | 线程 CPU 时间 |
| `eBPF` | `offcputime -t <tid>`、`biolatency` 等按 tid 过滤 | off-CPU 时间、IO 延迟 |
| `/proc` | `/proc/<pid>/task/<tid>/stack`、`/task/<tid>/status` | 内核栈、状态（无需额外工具） |
| `top` | `top -H -p <mysqld_pid>` | 线程级 CPU 实时视图 |

---

## 三、栈分析：`pstack` / `gdb` / `pt-pmp`

### 3.1 `pstack` —— 一次全量栈快照

**是什么**：`gdb` 的封装，attach 到进程打印**所有线程**的栈。

**适用**：看"**现在卡在哪个函数**"。

```bash
pstack <mysqld_pid> > /tmp/stack.txt          # 一次快照
# 或（pstack 不可用时）
gdb -p <mysqld_pid> -batch -ex "thread apply all bt"
```

★ **核心价值（来自本知识库的实战案例）** —— [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md) 的《Rename table 死锁分析》：

> `ALTER TABLE ... RENAME` 卡住，错误日志刷 `too many files stay open` / `problems renaming`，**pstack 停在 `fil_rename_tablespace`（InnoDB 文件层）而非 MDL 等待点**。真实死锁是 InnoDB `fil_system` 的 IO 同步缺陷。

**教训**：**卡住时先看 pstack 停在哪个函数**——停在 MDL 等待点才是 MDL 问题，停在 `fil_rename_tablespace` 是文件层问题。**持有 MDL 不代表死锁环里有 MDL。**

### 3.2 `gdb` —— 看锁的持有者

**适用**：实例已 hang / 有 core dump 时，**唯一现场**。

四类锁的排查法（详见 [`../lock/primitives/innodb_sync.md`](../lock/primitives/innodb_sync.md)「core dump 排查锁持有者」）：

| 锁 | gdb 看什么 | 关键点 |
|---|---|---|
| server `pthread_mutex_t` | `p LOCK_status` → `__data.__owner` | `__owner` = 持有者线程的 **LWP 号**，对照 `info threads` 定位 |
| server `pthread_rwlock_t` | `__nr_readers` / `__nr_writers_queued` / `__writer` | 写锁优先；读锁持有者可能多个，需排除法 |
| InnoDB `rw_lock_t` | `lock_word` + `writer_thread` + `last_*_file_name` | ★ **`last_s_file_name:last_s_line` / `last_x_file_name:last_x_line`** = 最近一次加 S/X 锁的源码位置 |
| InnoDB `PolicyMutex` | `m_lock_word` | 0 空闲 / 1 持有 / 2 有等待者 |

⚠️ **`m_waiters=0` 不代表无等待线程**——线程都在自旋（未进等待队列）时 waiters 仍为 0；自旋时长由 `innodb_sync_spin_loops` / `innodb_spin_wait_delay` 控制。

### 3.3 ★ `pt-pmp` —— 采样聚合找热点与持有者

**是什么**：Percona Toolkit 的 Poor Man's Profiler。本质是**循环执行 pstack（多次采样）后按栈聚合**，输出按出现次数排序的结果。

```bash
pt-pmp --pid <mysqld_pid> --iterations 30 --interval 0.5
# 或先存栈再聚合
pt-pmp /tmp/stack.txt
```

**★ 排除法（核心思想：持有者堆栈只出现 1 次，等锁者大量重复）** —— 详见 [`../lock/primitives/innodb_sync.md`](../lock/primitives/innodb_sync.md)：

1. 导出全部线程堆栈 → `pt-pmp` 聚合
2. 排除频次高于 `__nr_readers` 的高频堆栈（大概率是等锁者）
3. 排除阻塞在获取该锁的线程
4. 排除带 `poll()` / `epoll_wait` 的堆栈
5. 排除带 `pthread_cond_wait` / `__lll_lock_wait` 的堆栈（持有锁期间 sleep 概率极低）
6. **剩下的"只出现 1 次"的堆栈即持有者**

**参数联动**：若所有拿锁线程都在自旋（`waiters=0` 且大量 CPU 空转），减小 `innodb_sync_spin_loops` 或 `innodb_spin_wait_delay` 可让竞争显形——**排查不出来时的最后一招**。

### 3.4 三者的选择

| 场景 | 工具 |
|---|---|
| "现在卡在哪"（一次性） | `pstack` |
| "反复卡在同一处吗"（趋势） | `pt-pmp`（多次采样聚合） |
| 已 hang / 有 core | `gdb`（还能直接看锁结构体） |
| 找锁持有者 | `gdb` + `pt-pmp` 排除法 |

---

## 四、CPU 采样：`perf` / 火焰图 / `eBPF`

### 4.1 `perf` —— CPU 采样（几乎不暂停进程）

```bash
perf top -p <mysqld_pid>                       # 实时热点
perf record -p <mysqld_pid> -g -- sleep 30     # 采样 30 秒（含调用图）
perf report                                    # 查看
perf record -t <tid> -g -- sleep 30            # 只采某个 MySQL 连接
```

**关键点**：

- **采样式**（按频率中断取 PC），不是插桩 ⇒ 开销通常很低
- `-g` 采调用图，才能生成火焰图
- 需要 mysqld 的**符号表**（否则看到一堆地址）。建议部署时保留 `mysqld.debug` 或确保二进制未被 strip
- 配合 `THREAD_OS_ID` 可只采目标连接（见 2.3）

### 4.2 火焰图

```bash
# perf 采完后生成
perf script | stackcollapse-perf.pl | flamegraph.pl > mysql.svg
```

**怎么看**：

- **横轴 = 样本占比**（不是时间顺序），越宽 = 出现越多 = 热点
- **纵轴 = 调用栈深度**，越高的方块是被调用者
- 看**顶部宽的平顶**——那是真正在消耗 CPU 的函数
- ⚠️ 火焰图是 **on-CPU** 的；**off-CPU（等待）不在里面**——这正好是 MySQL 场景的常见盲区，需要 eBPF 的 off-CPU 分析补

### 4.3 `eBPF` —— 内核态、几乎零开销

**是什么**：在内核中运行沙箱程序，做**内核态聚合**后再吐给用户态 ⇒ 开销远低于把每个事件传给用户态的方案。

常用工具（bcc / bpftrace 生态）：

| 工具 | 看什么 |
|---|---|
| `offcputime` | **off-CPU 时间**（在等什么、等多久） |
| `biolatency` | 块设备 IO 延迟分布 |
| `tcplife` / `tcpretrans` | TCP 连接寿命 / 重传 |
| `funclatency` | 某内核/用户函数的延迟分布 |
| `syscount` | 系统调用频次 |
| `profile` | CPU 采样（perf 的 eBPF 版） |

**与 MySQL 联动**：

```bash
offcputime -p <mysqld_pid> -f 30      # 看 mysqld 在 off-CPU 上花在哪
offcputime -t <tid>                   # 只看某个 MySQL 连接
```

⇒ 这补上了火焰图看不到的"**等待**"维度，对数据库这种大量等待的工作负载尤其重要。

⚠️ 需要较新内核 + root/CAP_BPF 权限；容器环境可能受限。

---

## 五、系统资源：`iostat` / `vmstat` / `strace` / `lsof` / `tcpdump`

| 工具 | 看什么 | 典型用法 | 注意 |
|---|---|---|---|
| `iostat` | 磁盘 IO（吞吐、await、%util） | `iostat -x 1` | `await` 高 / `%util` 接近 100 = IO 瓶颈 |
| `vmstat` | CPU / 内存 / 上下文切换 / IO 等待 | `vmstat 1` | `r` 列（运行队列）持续 > CPU 核数 = CPU 饱和；`cs` 高 = 上下文切换频繁 |
| `strace` | 系统调用 | `strace -p <pid> -c`（统计） / `-e trace=file` | ⚠️ **代价很高**（每个 syscall 都拦截），生产慎用；`-c` 先做统计 |
| `lsof` | 打开文件 / 句柄 | `lsof -p <mysqld_pid>` | 查表数量、句柄泄漏、临时文件 |
| `tcpdump` | 网络包 | `tcpdump -i any port 3306 -w /tmp.pcap` | ⚠️ 高流量下代价大，且**抓到的是明文/敏感数据**，注意合规 |

**`iostat` 与 MySQL 的对账**：PFS 的 `wait/io/file/*` 给的是 MySQL 视角的 IO 等待；`iostat` 给的是设备视角。**两者都看**才能区分"是 MySQL 发起了太多 IO"还是"设备本身慢"。

---

## 六、监控平台：PMM

**是什么**：Percona Monitoring and Management——Prometheus + Grafana + 一堆 exporters 的打包方案，专为 MySQL 设计。

**与前述手段的关系**：

| 层级 | 手段 | 时效 |
|---|---|---|
| 实时排查 | `sys.session` / `PFS.threads` / `pstack` | 秒级 / 一次性 |
| **常驻监控** | **PMM**（Prometheus 抓取 + Grafana 展示 + 告警） | 长期 / 趋势 |

**它采什么**：

- `mysqld_exporter`：状态变量（QPS、连接、InnoDB 行操作、缓冲池…）
- PFS 相关：语句摘要（digest）、等待事件
- `node_exporter`：OS 层（CPU、内存、磁盘、网络）
- Query Analytics：基于 PFS `events_statements_summary_by_digest` 的慢查询分析

⇒ **PMM 是把本篇与前面几篇的"手工查询"自动化、长期化的方案**。

⚠️ 采集频率通常是 1s~5s，⇒ **抓不到秒级以下的抖动**；短时尖刺仍需 `perf` / pstack 现场抓。

---

## 七、★ 代价对照（哪些会暂停进程）

这是本篇最需要被记住的一张表——**选错工具可能把生产实例搞挂**。

| 工具 | 机制 | 会暂停进程吗 | 开销 | 生产可用 |
|---|---|---|---|---|
| **`pstack`** | attach（gdb 封装） | ✅ **会**（attach 期间目标进程被暂停） | 单次通常很短，但**连接数越多、栈越深，暂停越久** | ⚠️ 谨慎，低峰用 |
| **`gdb`** | attach | ✅ **会**（且默认停在断点/attach 点） | 同上，且交互式操作会长时间暂停 | ❌ 仅 hang/core 场景 |
| **`pt-pmp`** | 循环 `pstack` N 次 | ✅ **会，且是 N 次累加** | ⚠️ **多次暂停的累计时间需评估** | ⚠️ 控制 iterations |
| **`perf`** | 采样式（性能计数器中断） | ❌ 基本不暂停 | 低（通常 1%~5%） | ✅ 推荐 |
| **火焰图** | 基于 perf 数据离线生成 | ❌ 不涉及 | 无（生成是离线动作） | ✅ |
| **`eBPF`** | 内核态程序 | ❌ | **最低**（内核态聚合） | ✅（需权限与内核支持） |
| **`strace`** | 拦截每个 syscall | ❌ 但**极大拖慢** | ⚠️ **很高**，可能让实例变慢数倍 | ❌ 仅短时统计（`-c`） |
| **`tcpdump`** | 抓包 | ❌ | 高流量下显著 | ⚠️ 短时、限端口 |
| **`iostat`/`vmstat`** | 读 `/proc`、`/sys` | ❌ | 极低 | ✅ |
| **`lsof`** | 读 `/proc/<pid>/fd` | ❌ | 低（但句柄极多时慢） | ✅ |
| **PMM** | 定期抓取 | ❌ | 低（采集频率决定） | ✅（常驻） |

**核心结论**：

- ★ **`pstack` 与 `pt-pmp` 会暂停进程**——这是它们与 `perf`/eBPF 最本质的区别
- **`pt-pmp` 是多次暂停的累加**——`--iterations` 越大代价越大
- 高频/长期观测请用 **`perf` / eBPF / PMM**，不要用 pstack 家族轮询

---

## 八、决策树：什么时候用哪个

```
┌─ MySQL 能连上、能查 SQL
│   ├─ 看谁在跑 / 在等什么     → sys.session / PFS.threads     [processlist.md]
│   ├─ 看聚合热点（哪类语句慢） → sys.statements_with_*          [sys_schema.md]
│   ├─ 看状态量趋势            → PFS.global_status / PMM        [status_variables.md]
│   └─ 看 InnoDB 内部          → SHOW ENGINE INNODB STATUS      [show_commands.md]
│
├─ 想知道"CPU 到底烧在哪"
│   ├─ 常规 / 长期             → perf + 火焰图
│   └─ 要更低开销 / 要 off-CPU → eBPF（profile / offcputime）
│       └─ 只想看某个连接       → 先查 THREAD_OS_ID，再 perf/eBPF -t
│
├─ 实例卡住 / 无响应
│   ├─ 还能 attach             → pstack（看停在哪个函数）
│   │                             pt-pmp（采样聚合，找持有者）
│   └─ 已 crash / 有 core       → gdb（看锁结构体 + 线程栈）
│
├─ 怀疑 OS 资源瓶颈
│   ├─ IO                       → iostat -x 1
│   ├─ CPU / 上下文切换          → vmstat 1
│   └─ 句柄 / 文件               → lsof -p
│
└─ 要长期趋势 + 告警            → PMM
```

---

## 九、可观测性

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 现在卡在哪个函数 | `pstack` | `pstack <pid>` |
| 反复卡在同一处吗 | `pt-pmp` | `pt-pmp --pid <pid> --iterations 30` |
| CPU 热点（on-CPU） | `perf` + 火焰图 | `perf record -g` → `flamegraph.pl` |
| **等待时间（off-CPU）** | eBPF | `offcputime -p <pid>` |
| 某个连接的 CPU/等待 | 先取 tid | `threads.THREAD_OS_ID` → `perf -t` / `offcputime -t` |
| 磁盘瓶颈 | `iostat` | `iostat -x 1` |
| CPU / 上下文切换 | `vmstat` | `vmstat 1` |
| 长事务/大查询（现场） | 见 MySQL 侧 | [`processlist.md`](processlist.md) |
| 长期趋势 + 告警 | PMM | — |

### 典型组合（一个完整的排障流程）

```bash
# ① MySQL 侧：找到问题连接和它的 OS tid
mysql -e "SELECT processlist_id, thread_os_id, processlist_state, processlist_info
          FROM performance_schema.threads
          WHERE processlist_command NOT IN ('Sleep','Daemon')
          ORDER BY processlist_time DESC LIMIT 5;"

# ② 系统侧：看这个连接在 CPU 上干什么（不暂停进程）
perf record -t <tid> -g -- sleep 30 && perf report

# ③ 若怀疑是等待而非 CPU：看 off-CPU
offcputime -t <tid>

# ④ 若实例已卡住：抓栈（会暂停，低峰做）
pstack <mysqld_pid> > /tmp/stack.txt
# 或采样聚合找持有者
pt-pmp --pid <mysqld_pid> --iterations 30 --interval 0.5

# ⑤ 宏观：确认是不是 OS 资源问题
iostat -x 1 &  vmstat 1
```

---

## 十、Misc（易混淆点）

| 混淆 | 澄清 |
|---|---|
| `pstack` 只是"打印一下"，没代价？ | ❌ **会暂停进程**（attach 机制） |
| `pt-pmp` 比 pstack 更安全？ | ❌ 它是**循环 pstack**，是**多次暂停的累加**，`--iterations` 越大越久 |
| `perf` 会暂停进程吗？ | ❌ 采样式，基本不暂停，开销通常 1%~5% |
| 火焰图能看到"等待"吗？ | ❌ **只能看 on-CPU**；等待（off-CPU）需 eBPF 的 `offcputime` |
| strace 可以长期开着？ | ❌ **代价极高**（拦截每个 syscall），仅短时统计（`-c`） |
| `THREAD_OS_ID` 是 pid？ | ❌ 是 **tid**（Linux `gettid()`）；用 `perf -t <tid>`、`ps -L` 的 LWP 列 |
| 能直接用 `THREAD_ID` 去 KILL？ | ❌ `KILL` 用 **`PROCESSLIST_ID`**；`THREAD_ID` 只对 PFS 内部有意义 |
| PFS 全关了系统层工具就没用了？ | ❌ 正好相反——系统层**不依赖任何 instrument**，PFS 关了照样能用 |
| PMM 能抓到秒级抖动？ | ⚠️ 采集间隔通常 1s~5s，**抓不到秒级以下尖刺**，需现场 perf/pstack |
| `SHOW ENGINE INNODB STATUS` 的 spin 数能看自旋？ | ❌ **恒为 0**（8.0 已移除插桩）；自旋只能靠系统层 |
| 容器里能用 eBPF/perf 吗？ | ⚠️ 常受权限（CAP_BPF/CAP_SYS_ADMIN）与内核版本限制，需确认 |

---

### ★ 各工具的实战命令与输出形态

> 本节补"怎么用、输出长什么样"。工具本身的源码不在 MySQL 仓库内，故贴的是**命令与真实输出形态**，并标明工具归属。

#### ① `pstack` —— 本质是 gdb 的批处理包装

`pstack` 不是独立工具，它是 **gdb 的 shell 脚本包装**（RHEL 系称为 `pstack`，本质是）：

```bash
#!/bin/sh
# pstack 的实质（简化）：对目标 pid 跑 gdb batch，逐线程打印栈
gdb -quiet -nx -p $pid \
    -batch \
    -ex 'set pagination off' \
    -ex 'thread apply all bt' \
    2>/dev/null
```

**要点**：
- **★ 它会 attach 到目标进程 ⇒ 暂停进程**（gdb attach 时进程被 `SIGSTOP`）。持续时间通常很短（秒级），但**不可高频执行**。
- 输出是"**所有线程的栈**"，一次全量快照。
- ⇒ 判断"现在卡在哪"用一次快照即可；判断"热点在哪"要多次采样（见 `pt-pmp`）。

典型输出（截取一个线程）：

```
Thread 37 (Thread 0x7f8e4c0e9700 (LWP 12345)):
#0  0x00007f8e5a2b3f0d in poll () from /lib64/libc.so.6
#1  0x0000000000f2a3c1 in vio_io_wait (vio=0x7f8e3c0012a0, ...)
#2  0x0000000000f2a5b8 in vio_read (vio=0x7f8e3c0012a0, ...)
#3  0x00000000009d2e41 in net_read_raw_loop (net=0x7f8e3c0009a0, ...)
#4  0x00000000009d31a2 in net_read_packet (net=0x7f8e3c0009a0, ...)
#5  0x0000000000a1f8c3 in do_command (thd=0x7f8e3c000820)
#6  0x0000000000b4d2e1 in handle_connection (arg=0x7f8e4c001230)
```

⇒ 读法：**从下往上**是调用链，`#0` 是当前停的位置。上例是"连接线程在等客户端发数据"（`poll` → `vio_io_wait`），即**空闲**。

#### ② ★ `pt-pmp` —— 多次采样 + 聚合，用来找热点

`pt-pmp`（Percona Toolkit 的 Poor Man's Profiler）的核心逻辑是**循环调用 `pstack`/`gdb` 若干次，把每次的栈聚合计数**：

```bash
# pt-pmp 的实质（简化）
for i in $(seq 1 $SAMPLES); do
    pstack $pid 2>/dev/null >> $tmpfile
    sleep $INTERVAL
done

# 聚合：把每个线程的栈块做"相同栈归并为一组 + 计数"
awk '
  /^Thread/ { if (length(stack) > 0) { count[stack]++; stack = "" } ; next }
  { stack = stack "\n" $0 }
  END { if (length(stack) > 0) count[stack]++;
        for (s in count) print count[s], s }
' $tmpfile | sort -rn
```

**★ 为什么这样能定位问题——排除法**：

| 现象 | 含义 |
|---|---|
| 某个栈只出现 **1 次** | 大概率是**持有者**（只有一个线程持有这把锁） |
| 某个栈出现**几十上百次** | 是**等待者**（大量线程卡在同一个地方） |

⇒ 所以排查锁竞争时：**先找出"大量重复的那个栈"确定等什么，再从少数派里找持有者**。

典型输出：

```
    142 __lll_lock_wait,pthread_mutex_lock,lock_rec_lock_slow,lock_rec_lock,row_search_mvcc,ha_innobase::index_read,handler::index_read_map,...
      1 __lll_lock_wait,pthread_mutex_lock,lock_rec_lock_slow,lock_rec_lock,row_upd,ha_innobase::update_row,...
```

⇒ 142 个线程在等同一个行锁，1 个线程在做 `update_row` ⇒ **后者就是持有者**（或最接近它）。

⚠️ **代价**：`pt-pmp` 默认采样 30 次左右 ⇒ **进程被暂停 30 次**（累加）。生产慎用，低峰做。

#### ③ `perf` —— 不暂停进程的 CPU 采样

```bash
# 采样 30 秒，记录调用栈（-g）
perf record -F 99 -g -p $(pidof mysqld) -- sleep 30

# 查看热点
perf report --stdio

# 生成火焰图（需 FlameGraph 工具）
perf record -F 99 -g -p $(pidof mysqld) -- sleep 30
perf script > out.perf
stackcollapse-perf.pl out.perf > out.folded
flamegraph.pl out.folded > mysqld.svg
```

**解释**：
- **`-F 99`** 采样 99Hz（避开 100Hz 的整数倍，减少与系统定时器共振导致的采样偏差）
- **`-g`** 记录调用栈（没有它只能看到函数名，看不到调用链）
- **★ `perf` 不暂停进程**（内核采样），代价约 1%~5%，可较长时间运行
- ⇒ 与 `pstack`/`pt-pmp` 的分工：**CPU 热点用 `perf`；"卡住不动"用 `pstack`**

#### ④ eBPF —— 几乎零开销的内核态追踪

```bash
# 统计 mysqld 内某个内核函数的延迟分布
bpftrace -e 'kprobe:__blkdev_issue_flush /comm == "mysqld"/ { @start[tid] = nsecs; }
             kretprobe:__blkdev_issue_flush /@start[tid]/ {
                 @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'

# 追踪 MySQL 的用户态函数（需要 uprobe + 符号表）
bpftrace -e 'uprobe:/usr/sbin/mysqld:dispatch_command { @count[comm] = count(); }'
```

**解释**：
- **★ 在内核态聚合** ⇒ 只把结果传回用户态，开销远低于 perf 的全量采样
- **`uprobe`** 能追踪 MySQL 的**用户态函数**（如 `dispatch_command`），前提是二进制**带符号表**（生产版本常 strip 掉 ⇒ 需要安装 debuginfo）
- ⇒ 适合"不想暂停、又要看细节"的场景

#### ⑤ 端到端：从 MySQL 找到 OS 线程再下钻

这是本目录 `system_tools.md` 最有价值的一条链路（★ `THREAD_OS_ID` 是桥）：

```sql
-- ① MySQL 侧：找到问题连接和它的 OS 线程 id
SELECT PROCESSLIST_ID AS conn_id,
       THREAD_OS_ID  AS os_tid,
       PROCESSLIST_STATE, PROCESSLIST_INFO
  FROM performance_schema.threads
 WHERE PROCESSLIST_COMMAND != 'Sleep'
 ORDER BY PROCESSLIST_TIME DESC LIMIT 5;
```

```bash
# ② 系统侧：看这个线程在 CPU 上干什么（不暂停进程）
perf top -t <os_tid>

# ③ 若怀疑是等待而非 CPU：看 off-CPU
#    （需 eBPF，或 perf sched）

# ④ 若实例已卡住：抓栈（会暂停，低峰做）
pstack <os_tid>
# 或采样聚合找持有者
pt-pmp --pid <os_tid>

# ⑤ 宏观：确认是不是 OS 资源问题
iostat -x 1
sar -n DEV 1
```

⇒ ★ **这套流程的价值**：把"MySQL 里看到的现象"和"OS 上看到的开销"对应起来。`performance_schema.threads.THREAD_OS_ID` 是唯一可靠的桥梁（不要靠 `PROCESSLIST_ID` 猜）。

---

## 参考

- 工具文档：
  - `perf` — *Linux perf Examples*（Brendan Gregg）、`perf record/report/top` man page
  - 火焰图 — *Flame Graphs*（Brendan Gregg），`FlameGraph` 仓库的 `stackcollapse-perf.pl` / `flamegraph.pl`
  - eBPF — *BPF Performance Tools*（Brendan Gregg）；bcc / bpftrace 工具集
  - `pt-pmp` — Percona Toolkit 文档（Poor Man's Profiler）
  - PMM — Percona Monitoring and Management 文档
- 本知识库内的相关材料（避免重复，此处只做指向）：
  - [`processlist.md`](processlist.md) 5.4 —— ★ **`THREAD_OS_ID` 的来源与联动**（本篇的桥梁）
  - [`../lock/primitives/innodb_sync.md`](../lock/primitives/innodb_sync.md)「core dump 排查锁持有者：gdb + pt-pmp 排除法」—— ★ **六步排除法的完整方法论**、各类锁的 gdb 查看法
  - [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md)「排查案例：卡住≠MDL」—— ★ **pstack 停在 `fil_rename_tablespace`** 的经典反例
  - [`show_commands.md`](show_commands.md) 4.6/4.7 —— `SHOW ENGINE INNODB STATUS` 的锁代价与 spin 计数恒为 0
  - [`../memory/jemalloc.md`](../memory/jemalloc.md) —— 内存侧的 Profiling 与 RSS 对账
  - [`../README.md`](../README.md)（观测手段全景与决策树）

# memory：内存（分配器矩阵 + 全局内存架构 + jemalloc）

> 本目录收纳 MySQL 的**内存**主题。内存与锁、线程**同构**——横跨 server 与 InnoDB，是跨层基础设施：InnoDB 的 mem_heap / buf_buddy、server 的 MEM_ROOT / jemalloc 属于分配器层；Buffer Pool 页面、redo buffer、各 cache、PFS 内存统计属于架构层。三篇文档分工如下：
>
> - `allocator.md` —— **分配器**（怎么分配）：四个分配器的算法、数据结构、主链路、PFS 埋点机制、设计模式、论文索引
> - `memory_architecture.md` —— **架构**（谁在分配、分配多少）：MySQL 进程内存地图、各内存区的用途与控制参数、内存预算、OOM 排查路线图
> - `jemalloc.md` —— **★ 生产重点篇**（生产 MySQL 基本都是 jemalloc）：接入机制、RSS 四层次账本（allocated/active/mapped/resident/retained）、mallctl 观测与 OOM 诊断、MALLOC_CONF 调优、jeprof profiling、THP 交互、最佳实践清单

## 文件索引

| 文件 | 内容 |
|------|------|
| [allocator.md](allocator.md) | 分层内存分配器矩阵：mem_heap / MEM_ROOT / buf_buddy / jemalloc（从 `server/infra/` 迁入，2026-09-29）+ ut::allocator 族 + PFS 埋点机制 |
| [memory_architecture.md](memory_architecture.md) | 全局内存架构总览：InnoDB + server 各内存区、预算视角、PFS memory_summary 观测、OOM 排查 |
| [jemalloc.md](jemalloc.md) | ★ 生产重点：jemalloc 与 MySQL——接入、RSS 账本与 decay、观测/OOM 诊断、调优、Profiling、THP、最佳实践 |

## 与相关文档的边界

- **Buffer Pool 机制**（页状态机、LRU、刷盘、并发协议）→ [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)，内存视角的 BP（chunk、内存分配、压缩页 frame）在 `memory_architecture.md` 总览
- **PFS 框架本身**（PSI 接口、AOP 编织、零开销探针）→ [`../../server/infra/pfs.md`](../observability/pfs/pfs.md)，内存观测（memory_summary_* 表族）在 `memory_architecture.md`
- **执行期内存消费**（sort buffer / join buffer / TempTable）→ [`../../server/query/runtime/`](../../server/query/runtime/) 各篇，内存预算视角的汇总在 `memory_architecture.md`
- **AHI**（btr_search 内存区）→ [`../../index/ahi.md`](../../index/ahi.md)
- **redo log buffer** → [`../../innodb/redo_log.md`](../../innodb/redo_log.md)
- **数据结构**（哈希、队列等容器本身）→ [`../structure/`](../structure/)

## 分工判据

`memory_architecture.md` 只写各内存区的**"内存身份"四元组**：谁控制 / 多大 / 哪条分配路径 / 占预算多少。一个事实的**第一主语**决定归属：

| 主语 | 例子 | 归属 |
|------|------|------|
| 内存身份 | "BP 用 mmap 大页分配、chunk 128MB 粒度、resize 时 page_hash 双份" | `memory_architecture.md` |
| 功能机制 | "chunk 头部放 buf_block_t 数组、frame 16KB 对齐" | 各专题（buffer_pool.md 等） |

机制细节一律**指向专题 + 交叉引用**，不在内存文档复制（如 chunk 布局 → buffer_pool.md「slot-based 布局细节」、log buffer resize 三锁 → redo_log.md「log buffer 的在线变更」）。`memory_architecture.md` 保留并强化它**独有、任何专题都讲不全**的部分：进程总图、三分类、内存预算、OOM 排查、分配路径落点总表、版本误区。

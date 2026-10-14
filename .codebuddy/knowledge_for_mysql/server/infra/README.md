# infra：基础设施与通用机制

> 本目录是 MySQL **通用基础设施**的知识库：与具体查询/事务逻辑无关、但被全系统复用的机制层。
>
> **★ 线程三篇已迁出**（2026-09-28）：`connection_threading.md` / `thread_pool.md` / `thread_pool_mariadb_percona.md` 按**主题**归位到 [`../../infra/threading/`](../../infra/threading/)——线程与锁同构地横跨 server 与 InnoDB，属于跨层基础设施，不留在 server 层专题。
>
> **★ 内存篇已迁出**（2026-09-29）：`memory.md`（分配器矩阵，mem_heap/buf_buddy 属 InnoDB、MEM_ROOT 属 server，同样跨层）迁至 [`../../infra/memory/allocator.md`](../../infra/memory/allocator.md)，与全局内存架构总览同目录。

## 文件索引

| 文件 | 内容 |
|------|------|
| [diagnostics.md](diagnostics.md) | 诊断体系：诊断区（=语句完成状态记录区，含 affected_rows）、诊断区栈、DECLARE HANDLER（UNDO 从不支持）、DA 状态→OK/EOF/ERR 包 |
| [variables.md](variables.md) | 变量体系：系统变量 / 状态变量 / 用户变量 |
| [encoding.md](encoding.md) | 数据编码：mach 读写族、压缩编码、端序规则 |
| [charset.md](charset.md) | 字符集与排序规则：UCA 900 权重算法、InnoDB 比较回调 |
| [dbug.md](dbug.md) | DBUG 调试框架 |
| [mtr.md](mtr.md) | mysql-test 框架（MTR）原理剖析 |
| [build.md](build.md) | CMake 编译构建系统剖析 |
| [vio.md](vio.md) | VIO 通信抽象 |
| [io_cache.md](io_cache.md) | server 层 I/O 抽象：IO_CACHE 与 ostream 装饰者链 |
| **[reading_guide.md](reading_guide.md)** | 源码阅读知识地图：七个知识域、C++ 惯用法、学习路线 |

> **阅读顺序建议**：先扫一遍 [`reading_guide.md`](reading_guide.md) 的「七个知识域」知道自己缺什么，
> 再按需读各机制篇；遇到看不懂的代码回到它的「自诊断流程」。

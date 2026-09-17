# infra：基础设施与通用机制

> 本目录是 MySQL **通用基础设施**的知识库：与具体查询/事务逻辑无关、但被全系统复用的机制层。

## 文件索引

| 文件 | 内容 |
|------|------|
| [pfs.md](pfs.md) | **Performance Schema**：内建可观测性框架（event-based tracing、PSI 接口、AOP 编织、零开销探针）+ PFS 设计模式 |
| [memory.md](memory.md) | **内存分配器矩阵**：mem_heap（Bump）/ MEM_ROOT（Arena）/ buf_buddy（Buddy）/ jemalloc + 与 PFS 的交汇 |
| [variables.md](variables.md) | 变量体系：系统变量 / 状态变量 / 用户变量 / 配置文件来源链路 |
| [list.md](list.md) | 侵入式链表 |
| [encoding.md](encoding.md) | **数据编码**：`mach_read/write_from/to_N` 固定宽度族、★ 两种压缩编码（`compressed` 5..9B / `much_compressed` 1..11B，动态元数据 redo 用它）、★ 端序规则（整数大端=字典序，浮点刻意小端）、★ `read` vs `parse`（后者带边界检查，不可信输入必须用 parse）、应用场景速查 |
| [dbug.md](dbug.md) | DBUG 调试框架 |
| [vio.md](vio.md) | VIO 通信抽象 |
| [io_cache.md](io_cache.md) | **server 层 I/O 抽象**：`IO_CACHE` 数据结构（双缓冲指针）、函数指针"穷人版多态"、延迟建文件（`file == -1`）、`READ_NET` 把网络流伪装成文件、透明加解密、**`Basic_ostream`→`Truncatable_ostream`→`IO_CACHE_ostream`/`Binlog_encryption_ostream` 装饰者链与 `Binlog_ofile` 门面**、★ 这套体系里的 C++ 运用 |
| **[reading_guide.md](reading_guide.md)** | **★ 源码阅读知识地图**（不属于 infra 机制，是通用阅读能力）：七个知识域、C 预处理器与 C++ 惯用法清单、设计模式与 MySQL 源码完整对照表、分阶段学习路线、"读不懂"的自诊断流程、参考资料与学术索引 |

> **阅读顺序建议**：先扫一遍 [`reading_guide.md`](reading_guide.md) 的「七个知识域」知道自己缺什么，
> 再按需读各机制篇；遇到看不懂的代码回到它的「自诊断流程」。

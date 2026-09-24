# infra：基础设施与通用机制

> 本目录是 MySQL **通用基础设施**的知识库：与具体查询/事务逻辑无关、但被全系统复用的机制层。

## 文件索引

| 文件 | 内容 |
|------|------|
| [pfs.md](pfs.md) | **Performance Schema**：内建可观测性框架（event-based tracing、PSI 接口、AOP 编织、零开销探针）+ PFS 设计模式 |
| [pfs_statement.md](pfs_statement.md) | **PFS 语句统计**（1077 行）：`events_statements_*` 四张表族（current 是栈/history 环形/汇总矩阵）、`PFS_statement_stat` 统计字段与「诊断字段是 SQL 层 push 进来」、**450 桶指数延迟直方图**与 `QUANTILE_95/99/999`、`QUERY_SAMPLE` 三选一采样策略（最慢 / 老化 60s 重采）、sys 视图 p95 与 PFS 分位数的语义差异。★起点在**网络层收包回调**而非 `dispatch_command`，靠两次 refine 收窄事件名。闭环补充：**三层开关**（instrument/consumer/actor）与"过滤在 start 时一次算完"的零开销机理、PFS 表是**内存数组+位置游标**（`get_row_count` 返回容量上限、`UNIQUE KEY USING HASH` 实为 O(n) 扫描、TRUNCATE=清零内存）、事务/stage/wait 统一事件模型、**autosize 按 `max_connections` 等三 hint 而非内存**、与慢日志完全独立、复制线程走 `statement/abstract/relay_log` 且豁免 `setup_actors` |
| [memory.md](memory.md) | **内存分配器矩阵**：mem_heap（Bump）/ MEM_ROOT（Arena）/ buf_buddy（Buddy）/ jemalloc + 与 PFS 的交汇 |
| [variables.md](variables.md) | 变量体系：系统变量 / 状态变量 / 用户变量 / 配置文件来源链路 |
| [encoding.md](encoding.md) | **数据编码**：`mach_read/write_from/to_N` 固定宽度族、★ 两种压缩编码（`compressed` 5..9B / `much_compressed` 1..11B，动态元数据 redo 用它）、★ 端序规则（整数大端=字典序，浮点刻意小端）、★ `read` vs `parse`（后者带边界检查，不可信输入必须用 parse）、应用场景速查 |
| [charset.md](charset.md) | **字符集与排序规则**：`CHARSET_INFO` 双函数表结构 + 注册表（id 直索引、utf8=mb3 别名）、8 个 collation 家族、★ **UCA 900 权重算法**（扫描器逐级比较 / 权重页布局 / contraction trie+位图 / implicit weight / levels_for_compare 编译期分级 / ICU 规则串自研解析）、★ **InnoDB 侧比较回调**（dtype prtype 存 id → innobase_mysql_cmp 跨层调 strnncollsp、NO PAD 的 lengthsp 兜底、多字节 CHAR 剥空格存储）、strxfrm 排序键、PAD SPACE vs NO PAD、like_range/wildcmp、协议转换链、协商（DTCollation::aggregate） |
| [dbug.md](dbug.md) | DBUG 调试框架 |
| [mtr.md](mtr.md) | **mysql-test 框架（MTR）原理剖析**：★ 双层架构（Perl 调度层 `mysql-test-run.pl` + C++ 执行层 `client/mysqltest.cc`）、`My::*`/`mtr_*` 模块职责、目录与文件类型（suite/include/collections、`.test`/`.result`/`.opt`/`.reject`）、**服务器 bootstrap**（`mysqld --initialize-insecure` + `--init-file=bootstrap.sql`，`mtr` 库的由来）、★ **mysqltest 完整命令表**（`command_names[]` 复用服务器 `TYPELIB`）、golden file 比对与 `--record`、flaky 的来源、`My::SafeProcess` 的孤儿进程方案 |
| [build.md](build.md) | **CMake 编译构建系统剖析**：顶层 `CMakeLists.txt` 的结构（★ `INCLUDE` 顺序即依赖图、`ADD_SUBDIRECTORY` 从底层到高层）、`cmake/` 60+ 模块按职责分类、**bundled vs system 依赖策略**（`WITH_SYSTEM_LIBS`）、★ **bison 与 macOS 的坑**（最低 3.0.4、系统 bison 是 2.3 且 homebrew 不符号链接）、★ **`WITH_HYPERGRAPH_OPTIMIZER` 只在 Debug 构建默认 ON**、sanitizer/LTO/gcov 选项、插件 vs 组件两套宏、编译期 vs 运行期配置的三层对照 |
| [vio.md](vio.md) | VIO 通信抽象 |
| [io_cache.md](io_cache.md) | **server 层 I/O 抽象**：`IO_CACHE` 数据结构（双缓冲指针）、函数指针"穷人版多态"、延迟建文件（`file == -1`）、`READ_NET` 把网络流伪装成文件、透明加解密、**`Basic_ostream`→`Truncatable_ostream`→`IO_CACHE_ostream`/`Binlog_encryption_ostream` 装饰者链与 `Binlog_ofile` 门面**、★ 这套体系里的 C++ 运用 |
| **[reading_guide.md](reading_guide.md)** | **★ 源码阅读知识地图**（不属于 infra 机制，是通用阅读能力）：七个知识域、C 预处理器与 C++ 惯用法清单、设计模式与 MySQL 源码完整对照表、分阶段学习路线、"读不懂"的自诊断流程、参考资料与学术索引 |

> **阅读顺序建议**：先扫一遍 [`reading_guide.md`](reading_guide.md) 的「七个知识域」知道自己缺什么，
> 再按需读各机制篇；遇到看不懂的代码回到它的「自诊断流程」。

# infra：跨层基础设施（数据结构 + 锁）

> 本目录收纳 MySQL 的**跨层基础设施**——server 与 InnoDB 两层共有/共用、且不属于任何单一功能模块的"基座"内容。当前含两大子域：
>
> - `structure/` —— **数据结构**：按**类型**一篇一主题，**不分层**。跨层的同主题实现（如两个无锁哈希、两个 MPMC 队列）在**一篇文档里写清对比**（2026-09-20 用户拍板的组织原则）
> - `lock/` —— **锁与同步**：锁横跨 include/mysys/sql/innodb，是最重要的横切主题之一，整体从顶层迁入本目录（2026-09-20）
>
> **与 `server/infra/` 的区别**：本目录是**跨层**基础设施之家；`server/infra/` 是 **server 层基础设施专题**（PFS/内存/编码/IO_CACHE 等，**暂不归入**——它是 server 层专属设施，与跨层定位不同，2026-09-20 用户拍板先不动）。

## 目录树

```
infra/
├── README.md                             ← 本篇：定位 + 归属判据 + 全量清单
├── lock/                                 ← 锁与同步（详见 lock/README.md）
│   ├── primitives/                       ← 同步原语（mutex/rwlock/event/latch/RCU/Seq_lock…）
│   └── transactional/                    ← 事务锁（MDL/行锁/表锁/AUTOINC/全局锁…）
└── structure/                            ← 数据结构：按类型一篇一主题，不分层
    ├── list.md                           ✅ 链表三实现对比（ut_list_base / SQL_I_List / C LIST）
    ├── hash.md                           ✅ 哈希全盘点（LF_HASH / ut_lock_free_hash_t / hash_table_t 三实现）
    ├── queue.md                          ✅ 有界 MPMC 队列（Integrals_lockfree_queue / mpmc_bq 同族异路）
    ├── counter.md                        ✅ 分片计数器（ib_counter_t / Counter::Shards 两代对比）
    ├── link_buf.md                       ✅ Link_buf：无锁环形"链缓冲"（redo recent_written / recent_closed）
    └── data_repr.md                      ✅ 行数据表示（跨层：server Field/TABLE::record ↔ InnoDB dtype/dfield/dtuple ↔ rec_t）
```

## 归属判据

- **数据结构本身**（链表、哈希、队列、计数器、链缓冲等"结构"）→ `structure/`，**按类型一篇一主题**：同一主题跨层实现合一篇对比写清（如 `hash.md` 含三个哈希、`queue.md` 含两个队列），单层独有也单篇（如 `link_buf.md`）。**哈希/队列/计数器这类"主题"要写全量盘点**，不是只写某个实现（2026-09-20 用户拍板）
- **用无锁结构实现的机制**（第一主语是功能）→ 各功能模块目录：`Bgc_ticket_manager`（binlog 组提交 ticket 机制）→ `server/replication/binlog.md`；redo 对 `Link_buf` 的使用 → `innodb/redo_log.md`（容器本身的剖析在 structure/）
- **锁原语/同步原语**（mutex / rwlock / cond / event / latch / RCU 锁模式 / seqlock）→ `lock/primitives/`
- **事务锁**（MDL / 行锁 / 表锁 / AUTOINC / 全局锁）→ `lock/transactional/`

## 全量数据结构清单（与 `lock/README.md` 盘点同步）

| 结构 | 层 | 类型 | 用户/用途 | 文档 |
|---|---|---|---|---|
| 链表（`ut_list_base` / `SQL_I_List` / `base_list`+`List` / `I_List` / C `LIST`） | 跨层 | 五实现：侵入式双向偏移量 / 侵入式单向二级指针 / 非侵入式哨兵 / 侵入式双向自解链 / C 双向 | trx_sys / buf_pool / lock / item 链 / 执行计划链等 | ✅ [list.md](structure/list.md) |
| `LF_HASH` | server | 无锁哈希（split-ordered list + hazard pointer，**有论文原型**） | MDL_map / Acl_cache / PFS 对象表 | ✅ [hash.md](structure/hash.md) |
| `ut_lock_free_hash_t`（+ `ut_lock_free_cnt_t`） | innodb | 无锁哈希（开放寻址 + 链表数组，**无论文原型**） | `buf_stat_per_index`（bp 每索引页数） | ✅（同上 hash.md） |
| `hash_table_t` | innodb | **有锁**经典哈希（链地址法 + 分片 rw_lock） | lock_sys `rec_hash`/`prdt_hash`、bp `page_hash`、dict、AHI | ✅（同上 hash.md，三实现糅合一篇） |
| `Integrals_lockfree_queue` | server | 无锁队列（仅整型，虚拟索引 + MSB 占用位） | `Bgc_ticket_manager`（binlog 组提交 ticket） | ✅ [queue.md](structure/queue.md) |
| `mpmc_bq` | innodb | 无锁有界 MPMC（Vyukov 算法，引 1024cores.net） | dblwr `Segments` / FTS `Docq` | ✅（同上 queue.md，同族异路合一篇） |
| `ib_counter_t` | innodb | 分片计数器（RDTSC 散槽，读 fuzzy） | `srv0srv.h` 5 个 typedef + 全局统计 | ✅ [counter.md](structure/counter.md) |
| `Counter::Shards` | innodb | 分片计数器第二代（真原子，读精确） | bp `m_n_page_gets`、no-logging mtr、Parallel_reader | ✅（同上 counter.md） |
| `Link_buf<Position>` | innodb | 无锁环形"链缓冲"（乱序报告 + tail 沿链推进） | redo `recent_written` / `recent_closed` | ✅ [link_buf.md](structure/link_buf.md) |
| `Seq_lock<data_t>` | innodb | 序号锁（seqlock，回调式，引 HPL-2012-68） | `mt_fast_modulo_t`（hash 表快速取模） | 属锁原语，✅ [lock/primitives/seq_lock.md](lock/primitives/seq_lock.md) |
| `MyRcuLock<T>`（RCU） | server | RCU 锁模式 | `ssl_acceptor_context_data` | 属锁原语，✅ [lock/primitives/rcu.md](lock/primitives/rcu.md) |
| `dtype_t` / `dfield_t` / `dtuple_t`（+ `big_rec_t`/`upd_t`/`multi_value_data`/`row_ext_t`） | 跨层 | **行数据表示**（类型位域打包 / 字段 / 元组），对偶于 `rec_t` 字节流 | server↔InnoDB 插入检索全链路 | ✅ [data_repr.md](structure/data_repr.md) |
| `Field`（68 子类）/ `TABLE::record[2]` / `String` / `Copy_field` | server | server 行表示（类型视图 + 字节缓冲） | 执行器、handler 接口、字符集转换 | ✅（同上 data_repr.md，与引擎侧合一篇对比） |

## 两个无锁 hash 的对比入口

server 侧的 `LF_HASH` 与 innodb 侧的 `ut_lock_free_hash_t` 是**两条完全不同的路线**（split-ordered list + hazard pointer vs 开放寻址 + 链表数组 + 引用计数；有论文原型 vs 无论文原型）。三实现对比、论文不对称的诚实说明与"通用性越强、理论原型越重"的结论见 [hash.md](structure/hash.md)「总览」与「选择指南」。

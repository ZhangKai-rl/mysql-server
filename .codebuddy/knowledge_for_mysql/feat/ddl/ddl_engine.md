# 8.0 新版 DDL 引擎与并行构建深度解析

> 基于 MySQL 8.0.39 源码，剖析 8.0 引入的新版 DDL 执行引擎 `storage/innobase/ddl/`（约 200 KB）：`Context` / `Loader` / `Builder` / `Btree_load` 的分层与职责、扫描—排序—建树三期的真实算法、临时文件与缓冲区管理，以及并发调度（线程池构成、任务队列的同步与终止检测、`Parallel_reader` 的动态分片）。
>
> **边界**：本篇只写**引擎内部的执行引擎**——索引和表究竟是怎么被构建出来的。**并行 DDL 是这套引擎的并发维度，故并入本篇，不单独成篇。** server 层的 DDL 编排、算法选择策略与 MDL 三段式见 [`ddl.md`](ddl.md)；各算法自身的完整跨层链路见 [`inplace_ddl.md`](inplace_ddl.md)、[`copy_ddl.md`](copy_ddl.md)、[`instant_ddl.md`](instant_ddl.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [新版 DDL 引擎全景](#80-新版-ddl-引擎全景storageinnobaseddl)
  - [为什么单独一个目录（`row0merge.cc` 消失）](#独立目录的由来row0mergecc-的移除)
  - [老架构剖析：`row0merge.cc`](#老架构剖析row0mergecc3975-行--130-kb)
  - [★★ 新旧对比：到底改了什么](#-新旧实现对比)
  - [文件 / 类职责总表](#文件--类职责总表) / [协作图：从 server 到 B+Tree](#协作图从-server-到-btree)
  - [★ `ddl::Builder` 的状态机](#-ddlbuilder-的状态机)
  - [扫描期](#扫描期行进入-builder-的路径) / [★ 排序期：多趟 2 路而非一次 N 路](#-排序期是多趟-2-路不是一次-n-路) / [★ 建树期：归并并行、建树串行](#-建树期归并并行建树串行)
  - [★ `ddl::Loader`：线程数、队列终止、错误处理](#-ddlloader线程数队列终止错误处理)
  - [并行扫描与 `ddl::Context`](#并行扫描parallel_cursor退化单线程的三个条件) / [临时文件与缓冲区](#临时文件与缓冲区)
  - [FTS 与 RTree 两个特例](#fts-构建的模型差异) / [对外入口 `ddl0ddl.cc`](#ddl0ddlcc对外入口)
  - [★ 关键函数逐行源码剖析](#-关键函数逐行源码剖析)
- [并行 DDL：这套引擎的并发维度](#并行-ddl社区版-8027-确有)
  - [★ 任务队列：同步原语与终止判定](#)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`storage/innobase/ddl/` 是 8.0 引入的 **DDL 执行引擎**：索引与表究竟"怎么被构建出来"由它负责——扫描旧数据、排序、自底向上 bulk load 成 B-tree。它取代了旧的 `row0merge.cc`，把构建过程拆成 `Context`（状态与内存预算）/ `Loader`（线程与任务调度）/ `Builder`（单索引构建）/ `Btree_load`（建树）四层。

### 用途

它是 INPLACE 算法（重建表、加索引）底下的**执行层**：[`inplace_ddl.md`](inplace_ddl.md) 讲的是并发协议（MDL 升降级 + row log），本篇讲的是那条协议底下真正干活的引擎。COPY 不走这套（它逐行 INSERT），INSTANT 也不走（它压根不构建）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 / 5.7 | 索引构建在 `row0merge.cc`（约 130 KB）：单线程、merge sort 写临时文件、再逐条插入 B-tree；5.7.5 引入 **bulk load**（sorted index build）替代逐条插入 |
| 8.0 | 新增 `storage/innobase/ddl/`（约 200 KB），构建过程按"扫描—排序—建树"三期分层重写，**`row0merge.cc` 从源码树消失** |
| 8.0.14 | InnoDB **并行扫描**（`innodb_parallel_read_threads`），为并行 DDL 提供扫描能力 |
| 8.0.27 | **并行创建索引**（`innodb_ddl_threads`），社区版可用；本篇的「并行 DDL」章即此 |

---

## 理论基础

### 设计思想与权衡

**要解决的问题**：索引构建是 DDL 里最耗时的部分——O(n) 扫描 + O(n log n) 排序 + 建树。在大表上它决定了整个 DDL 的时长，而老的 `row0merge.cc` 是**全程单线程**的。

**新引擎的做法**：把构建切成三期，逐期决定能否并行：

```
扫描（可并行）  →  排序（可并行：各线程归并自己那份）  →  建树（串行）
```

**为什么建树仍然串行**：B-tree 的 bulk load 是自底向上按序填充，并行建树需要跨线程协调页分裂与父子关系，实现复杂度远高于收益。源码里那句 `// todo: parallel sort and build` 说明官方也承认这里还有并行化空间——**不是不能，是尚未做**。

**代价与约束**：

- **并行度受内存预算限制**：`innodb_ddl_buffer_size` 是总量，线程越多每线程分到的排序缓冲越小，缓冲区不足会退化成更多趟归并、更多临时文件 I/O——并行不是无条件加速。
- **临时文件成为新瓶颈**：排序要写临时文件，`innodb_disable_sort_file_cache` 决定是否绕过 OS page cache（绕过可减少双份缓存，但每次读写都落盘）。
- **失效场景**：小表上并行开销大于收益，会退化成单线程（见「并行扫描：退化单线程的三个条件」）；内存预算不足时线程数也会被折算下调。

### 算法与数据结构

| 阶段 | 算法 | 并行度 | 关键类 |
|---|---|---|---|
| 扫描 | `Parallel_reader` 按叶子页范围**动态**分片（不是静态均分） | 并行 | `Parallel_reader` / `Parallel_cursor` |
| 排序 | **多趟 2 路归并**（不是一次 N 路）——受缓冲区大小约束 | 并行（各线程归并自己那份） | `Builder` / `Merge_cursor` |
| 建树 | 自底向上 **bulk load**（sorted index build） | **串行** | `Btree_load` |
| 调度 | 任务队列 + **quiescence 终止检测** | 并行 | `Loader` / `Loader::Task_queue` |

★ "多趟 2 路"这个选择常被误解为偷懒：一次 N 路归并需要同时持有 N 份输入缓冲，内存预算不允许；多趟 2 路只需要 2 份，用临时文件轮转换取内存——**是用 I/O 换内存的典型取舍**。

---

## 8.0 新版 DDL 引擎全景（`storage/innobase/ddl/`）

> **这是本篇最大的一块此前完全没覆盖的源码**。8.0.27 并行 DDL 重构后，InnoDB 的索引构建从旧的 `row0merge.cc` 迁到了**新增的 `storage/innobase/ddl/` 目录**——**10 个 .cc、约 200 KB**。本章按"架构 → 各组件 → 各阶段"讲透。

### 独立目录的由来：`row0merge.cc` 的移除

- **8.0.39 里 `row0merge.cc` 已被移除**（`storage/innobase/` 下再无 `row0merge.*`），只剩 3 处**注释中的历史引用**（`srv0srv.h` 的 `row_merge_insert_index_tuples()` / `row_merge_sort()`、`ut0stage.h` 的 `row_merge_buf_sort()`、`lob0lob.h` 的 `row_merge_copy_blobs()`）——全是注释，不是代码
- 新目录所有文件头注释统一写着 **`Created 2020-11-01 by Sunny Bains.`**（`ddl0impl.h`、`ddl0impl-builder.h`、`ddl0impl-loader.h`、`ddl0impl-merge.h`、`ddl0impl-buffer.h`、`ddl0builder.cc`、`ddl0loader.cc`、`ddl0merge.cc`、`ddl0par-scan.cc`、`ddl0ctx.cc`、`ddl0ddl.cc`、`ddl0file-reader.cc`、`ddl0rtree.cc`）
  - 唯一例外：`include/ddl0fts.h` 是 `Created 10/13/2010 Jimmy Yang` —— FTS 那块是从更老的代码搬来的
- ⚠️ **WL 号：源码无直接证据**（`storage/innobase/ddl/` 与 `include/ddl0*.h` 里 grep `WL #` / `Work Log` 均 0 命中），不要编

### 老架构剖析：`row0merge.cc`（3975 行 / 130 KB）

> 老代码在 8.0.39 已被删除，但 **git 里取得到**：`git show e701897fdd1^:storage/innobase/row/row0merge.cc`（删除提交 `e701897fdd1`，2021-05-04）⇒ **3975 行、130791 字节**。本节的结论全部来自这份真实代码。

#### 总体流程与调用链

主入口 **`row_merge_build_indexes()`**（对外唯一入口，被 ALTER/CREATE 路径调用）：

```
row_merge_build_indexes(trx, old_table, new_table, online, indexes[], ...)
│
├─ stage->begin_phase_read_pk(...)                        PFS：进入"读 PK"阶段
├─ block = alloc.allocate_large(3 * srv_sort_buf_size)    3 个 block：in0 / in1 / out
├─ FlushObserver(new_table->space, trx, stage)
├─ 每个 DICT_FTS 索引：row_merge_create_fts_sort_index() → row_fts_psort_info_init()
│
├─ ★ row_merge_read_clustered_index(...)        ← 扫描 + 建 run（单线程，约 940 行）
│     ├─ 每个索引一个 row_merge_buf_create()
│     ├─ btr_pcur_open_at_index_side(true, clust_index, BTR_SEARCH_LEAF, ...)
│     └─ 主循环：每扫一行 → for (i = 0; i < n_index; i++)
│            row_merge_buf_add()  满则 row_merge_buf_sort() → row_merge_buf_write()
│                                 → row_merge_write(file->fd, file->offset++, block)
│
└─ for (i = 0; i < n_indexes; i++):            ← 排序 + 装载（串行，索引 0 完全建完才轮到 1）
      if (dict_index_is_spatial) continue;     空间索引扫描期已插入
      if (DICT_FTS) row_fts_start_parallel_merge() / row_fts_merge_insert()
      else if (merge_files[i].fd >= 0):
          row_merge_sort(...)                  多趟 2 路归并
          BtrBulk btr_bulk(sort_idx, trx->id, flush_observer);
          btr_bulk.init();
          row_merge_insert_index_tuples(..., &btr_bulk, stage);
          btr_bulk.finish(error);
      row_merge_file_destroy(&merge_files[i]);
      if (online && old_table == new_table) { flush_observer->flush(); row_merge_write_redo(); row_log_apply(); }
```

⇒ **关键结构性事实**：扫描是"一行 → 所有索引各填一份"的**耦合循环**；排序/归并/装载是"一个索引走完再下一个"的**串行 for**。

#### 核心数据结构（由使用点反推，定义在被一并删除的 `row0merge.h`）

```c
/* row_merge_buf_create_low() 中的初始化 */
  buf->heap = heap;  buf->index = index;  buf->max_tuples = max_tuples;
  buf->tuples = ut_malloc_nokey(2 * max_tuples * sizeof *buf->tuples);
  buf->tmp_tuples = buf->tuples + max_tuples;
```

| 结构 | 字段 | 要点 |
|---|---|---|
| `row_merge_buf_t` | `heap` / `index` / `max_tuples` / `tuples` / `tmp_tuples` / `n_tuples` / `total_size` | `tuples` 分配 **2×max_tuples**（后半当归并 aux）；`tuples` 用 `ut_malloc_nokey` 而非 heap ⇒ `buf_empty()` 可 `mem_heap_empty(heap)` 后把数组原样挂回复用。`max_tuples = srv_sort_buf_size / index->get_min_size()` |
| `merge_file_t` | `int fd` / `ulint offset` / `ulint n_rec` | ⚠️ **`offset` 的单位是 block 数不是字节**：`os_offset_t ofs = ((os_offset_t)offset) * srv_sort_buf_size;` |
| `row_merge_dup_t` | `{index, table, col_map, n_dup}` | `row_merge_dup_report()` 里 `if (!dup->n_dup++)` ⇒ **只报第一条但计全部** |

#### 缓冲区：一个 buffer 满 = 一个 block = 一个 run

`row_merge_buf_add()` 的满判据两条，缺一不可：

```c
  if (buf->n_tuples >= buf->max_tuples) return 0;
  /* Reserve one byte for the end marker of row_merge_block_t. */
  if (buf->total_size + data_size >= srv_sort_buf_size - 1) {
    return (index->is_multi_value() ? n_row_added : 0);
  }
```

写盘发生在扫描循环的 `write_buffer_and_retry:` 标签处。有个重要分支——**数据能全放内存就不落盘**：

```c
        if (row == nullptr && file->fd == -1 && !clust_temp_file) {
          /* 全部行都在内存里且扫描已结束 → 直接 BtrBulk 插入，不用临时文件 */
          BtrBulk btr_bulk(index[i], trx->id, observer); ...
        } else {
          row_merge_file_create_if_needed(file, tmpfd, buf->n_tuples, path);
          row_merge_buf_write(buf, file, block);
          if (!row_merge_write(file->fd, file->offset++, block)) err = DB_TEMP_FILE_WRITE_FAIL;
        }
```

#### 临时文件 IO：裸 `int fd` + 整块读写

```c
ibool row_merge_read(int fd, ulint offset, row_merge_block_t *buf) {
  os_offset_t ofs = ((os_offset_t)offset) * srv_sort_buf_size;
  IORequest request;
  /* Merge sort pages are never compressed. */
  request.disable_compression();
  err = os_file_read_no_error_handling_int_fd(request, nullptr, fd, buf, ofs, srv_sort_buf_size, nullptr);
  posix_fadvise(fd, ofs, srv_sort_buf_size, POSIX_FADV_DONTNEED);   /* Each block is read exactly once. */
  return err == DB_SUCCESS;
}
```

- 固定整块 `srv_sort_buf_size` 读写（老版**没有**按 `IO_BLOCK_SIZE` 对齐的细粒度写入）
- PFS 是**手工**注册：`PSI_FILE_CALL(get_thread_file_name_locker)(...)`
- 跨 block 的记录用 `mrec_buf_t` 拼：`memcpy(*buf, b, avail_size); row_merge_read(fd, ++(*foffs), block); memcpy(*buf + avail_size, b, extra_size - avail_size);`

**记录格式（与新版逐字节相同 —— 磁盘格式向后兼容）**：

```c
/* 写 */
  if (e < 0x80) *b++ = (byte)e;
  else { *b++ = (byte)(0x80 | (e >> 8)); *b++ = (byte)e; }
  memcpy(b, mrec - rec_offs_extra_size(offsets), rec_offs_size(offsets));
/* 读 */
  extra_size = *b++;
  if (!extra_size) { *mrec = nullptr; return nullptr; }      /* End of list */
  if (extra_size >= 0x80) { extra_size = (extra_size & 0x7f) << 8; extra_size |= *b++; }
  extra_size--;                                               /* Normalize: 0 = EOF */
```

前缀值 = `extra_size + 1`（+1 是为了让 0 专用于 EOF）。`row_merge_read_rec()` 的返回语义很讲究：**返回 `nullptr` 且 `*mrec == nullptr` = 正常 EOF；返回 `nullptr` 且 `*mrec != nullptr` = IO 错误**（调用点用 `if (!b) { if (mrec) goto corrupt; }` 区分）。

#### 排序与归并：老版本**同样**是多趟 2 路 + ping-pong

内存内：`UT_SORT_FUNCTION_BODY` 归并排序，`row_merge_tuple_cmp` 比较。

磁盘上（`row_merge_sort` → `row_merge`）：

```c
  const ulint ihalf = run_offset[*num_run / 2];   /* half the input file */
  foffs0 = 0;  foffs1 = ihalf;
  for (; foffs0 < ihalf && foffs1 < file->offset; foffs0++, foffs1++) {
    if (trx_is_interrupted(trx)) return DB_INTERRUPTED;
    run_offset[n_run++] = of.offset;
    error = row_merge_blocks(dup, file, block, &foffs0, &foffs1, &of, stage);   // 2 路合并
  }
  while (foffs0 < ihalf)  row_merge_blocks_copy(...);     // 落单的左边
  while (foffs1 < file->offset) row_merge_blocks_copy(...);
  if (of.n_rec != file->n_rec) return DB_CORRUPTION;
  /* Swap file descriptors for the next pass. */
  *tmpfd = file->fd;  *file = of;
```

`row_merge_blocks()` 用 3 个 block（in0/in1/out）+ 宏 `ROW_MERGE_WRITE_GET_NEXT_LOW`，唯一索引去重在合并里当场做：

```c
  while (mrec0 && mrec1) {
    int cmp = cmp_rec_rec_simple(mrec0, mrec1, offsets0, offsets1, dup->index, dup->table);
    if (cmp < 0)  ROW_MERGE_WRITE_GET_NEXT(0, dup->index, goto merged);
    else if (cmp) ROW_MERGE_WRITE_GET_NEXT(1, dup->index, goto merged);
    else { mem_heap_free(heap); return DB_DUPLICATE_KEY; }
  }
```

#### 装载 B-tree：老版本**已经是 bulk load**（`BtrBulk`）

```c
      BtrBulk btr_bulk(sort_idx, trx->id, flush_observer);
      error = btr_bulk.init();
      if (error == DB_SUCCESS) {
        error = row_merge_insert_index_tuples(trx, sort_idx, old_table,
                                              merge_files[i].fd, block, nullptr, &btr_bulk, stage);
        error = btr_bulk.finish(error);
      }
```

`row_merge_insert_index_tuples()` 内部：`row_merge_read_rec` → `row_rec_to_index_entry_low` → `btr_bulk->insert(dtuple)`。

⇒ ⚠️ **所以新版不是"从逐条 insert 进化到 bulk load"**——老版本早就是 bulk load，变化是 **`BtrBulk` → `Btree_load`**，且新版把"读记录"从装载器里剥离（装载器持有 cursor 自己拉）。老版本**唯一**走逐条 `btr_cur_optimistic_insert` 的是**空间索引**（`index_tuple_info_t::insert()`）。

#### 并行能力：**完全无**

- 扫描：`btr_pcur_open_at_index_side(...)` + 单线程 while，一个 `mtr`
- 全文件 grep `parallel|std::thread|pthread|os_thread`：命中的**全是 FTS 相关**（`fts_parallel_sort_event` / `row_fts_start_parallel_merge`，由 `row0ftsort.cc` 管），没有任何线程创建或 `Parallel_reader`
- 排序/归并/装载：在 `for (i = 0; i < n_indexes; i++)` 里**串行**

#### 空间索引与 FTS：完全混在主流程

**空间索引**：扫描期只缓存不排序（`sp_tuples[s_idx_cnt]->add(row, ext)`），真正插入在 `skip_sort` 分支，且与 mtr 提交时机纠缠：

```c
          /* We have to insert the cached spatial index rows, since after the
             mtr_commit, the cluster index page could be updated, then the data
             in cached rows become invalid. */
          err = row_merge_spatial_rows(trx->id, sp_tuples, num_spatial, ..., &pcur, &mtr, &mtr_committed);
          if (!mtr_committed) { btr_pcur_move_to_prev_on_page(&pcur); btr_pcur_store_position(&pcur, &mtr); mtr_commit(&mtr); }
```

`row_merge_spatial_rows()` 还会返回 `DB_CACHE_RECORDS` 触发"先把指针深拷贝成数据再提交 mtr"的二次尝试。

**FTS**：先造假 `fts_sort_idx`，主循环里 `buf->index->type & DICT_FTS` 的分支散落 4 处，且明确不写临时文件：

```c
      /* Secondary index and clustered index which is not in sorted order can use
         the temporary file. Fulltext index should not use the temporary file. */
      if (!skip_sort && !(buf->index->type & DICT_FTS)) { ... }
```

装载阶段用 `row_fts_start_parallel_merge()` + `os_event_wait_time_low` **轮询 10000 次**等子线程退出。

### ★★ 新旧实现对比

| 维度 | 老（`row0merge.cc`） | 新（`ddl/`） |
|---|---|---|
| **代码组织** | 单文件 3975 行，混装：排序缓冲 + 临时文件 IO + 归并 + B-tree 装载 + 空间索引 + FTS | 10 个 `.cc` + 13 个 `ddl0*.h`，全在 `namespace ddl`，按职责切分 |
| **核心抽象** | `row_merge_buf_t` + 一堆自由函数，状态靠参数传递 | 类层次：`Context` → `Loader`（线程池+任务队列）→ `Builder`（每索引状态机）→ `Thread_ctx`（每线程 buffer+file） |
| **并行** | **无**。扫描/排序/归并/装载全串行 | 三层并行：①`Parallel_reader` 并行扫描 ②`std::thread` 线程池 ③每 `Thread_ctx` 独立排序/归并 |
| **归并方式** | 多趟 2 路 + ping-pong，**单线程**；`offset` 单位是 **block 数** | 同样多趟 2 路 + ping-pong，但 **N 个线程各自归并**；`m_offsets` 存的是**字节偏移** |
| **B-tree 装载** | `BtrBulk`，装载器**自带**读记录循环 | `Btree_load`，**装载器持有 cursor 自己拉**（读与装载解耦，装载器可内部并行） |
| **临时文件 IO** | 裸 `int fd`；整块 `srv_sort_buf_size` 读写；PFS 手工 locker | RAII 句柄 `os_fd_t`；`ddl::pread/pwrite` 字节偏移；IO buffer 按 `UNIV_SECTOR_SIZE` 对齐，长度按 `IO_BLOCK_SIZE` 对齐 |
| **内存管理** | 单一旋钮 `srv_sort_buf_size`，一次分配 3 个 block | `Context` 三个预算函数按线程数动态分配；PSI key `mem_key_ddl`；排序完主动释放 key buffer |
| **错误处理** | C 风格 `dberr_t` + `goto func_exit/corrupt`；手工 `trx->error_key_num = i` | 面向对象 `set_error()/get_error()`；状态机有显式 `State::ERROR`；`Task_queue` 全局错误快速失败 |
| **可观测性** | 一个共享 `ut_stage_alter_t`，多索引共用一个计数 | **每 Builder 一个 `Alter_stage`**，最后 `aggregate()` 汇总；`ib_ddl` 线程也进 PFS |
| **空间索引 / FTS** | 混在主流程（分支散落、与 mtr 时机纠缠） | 完全分离：`ddl0rtree.cc`（`RTree_inserter`）+ `ddl0fts.cc`（`FTS` 类）；主流程只留两个谓词 + 一个 `FTS_SORT_AND_BUILD` 状态 |

#### 重构动机

1. **并行化是主因，而且是唯一有硬证据的动因**。老的 `row_merge_read_clustered_index()` 是"扫一行 → 遍历所有索引各填一份"的巨型函数，所有索引的 buffer 生命周期被绑死在同一个 `mtr` 上。新版拆成 `Parallel_cursor`（只管扫描）+ `Builder`（只管一个索引）+ `Loader`（只管调度），`Builder` 的状态机允许每个索引独立推进，这才使按索引/按线程投放任务成为可能。
   **证据**：`Parallel_cursor::scan()` 直接复用 8.0 已有的 `Parallel_reader::available_threads()` + `reader.run(n_threads)`——老 `row0merge.cc` 完全没有接入点。
2. **多路输入是原结构撑不住的**。新版每线程有 `Thread_ctx{m_key_buffer, m_file, m_offsets}`，"一个文件里有多段 run"这层语义老版 `merge_file_t{fd, offset, n_rec}` 表达不了（老版用单独的 `run_offset` 数组，且只服务于一趟内的对半切分）。
3. **装载器解耦**：老版装载器自带读循环；新版 `btree_build()` 只做 `add_file()`×N → `cursor.open()` → `btr_load.build(cursor)`。
4. **可观测性粒度**：多索引并行推进时，共用一个 stage 无法区分进度 ⇒ 必须每 Builder 一个 `Alter_stage` 再聚合。

⚠️ **源码无直接证据的部分**：为什么从 `BtrBulk` 换成 `Btree_load`、为什么 `N_WAY_MERGE` 保留 2、为什么 FTS 独立成 50 KB 文件——`storage/innobase/ddl/` 的注释没有说明动机。唯一相关的是 `ddl0impl-merge.h` 那句 *"The design is generalized as an N way merge, however we stick with 2 for now."* ⇒ **N 路是预留而非当前实现**。

### 文件 / 类职责总表

| 文件 | 大小 | 角色 | 主要类 |
|---|---|---|---|
| `ddl0builder.cc` | **60 KB** | **索引构建核心**：每索引一个 Builder，含状态机、行→tuple、落盘、归并调度、B+Tree 装载 | `ddl::Builder`、`Builder::Thread_ctx`、`Load_cursor`、`Merge_cursor`、`File_cursor` |
| `ddl0fts.cc` | **50 KB** | 全文索引构建（分词 + aux 表装载） | `ddl::FTS`、`FTS::Parser`、`FTS::Inserter` |
| `ddl0ddl.cc` | 17 KB | 对外 API + 临时文件 IO + 失败回滚 | `ddl::create_index` / `drop_indexes` / `drop_table` / `lock_table` / `file_create` / `pread` / `pwrite` / `Dup::report` |
| `ddl0ctx.cc` | 16 KB | DDL 执行期上下文 | `ddl::Context`、`ddl::Sequence`、`Context::FTS` |
| `ddl0merge.cc` | 16 KB | 临时文件的外部归并排序 | `ddl::Merge_file_sort`、`Merge_file_sort::Cursor`、`Output_file` |
| `ddl0loader.cc` | 13 KB | 编排：扫描 → 排序 → 建树的任务调度 | `ddl::Loader`、`Loader::Task`、`Loader::Task_queue` |
| `ddl0par-scan.cc` | 12.6 KB | 聚簇索引（并行）扫描 | `ddl::Parallel_cursor`（`Cursor::create_cursor` **只返回它**） |
| `ddl0buffer.cc` | 7.7 KB | 内存排序缓冲 | `ddl::Key_sort_buffer`（⚠️ **没有 `ddl::Buffer` 这个类**） |
| `ddl0file-reader.cc` | 7 KB | 临时文件按块预读 / 跨块拼接 | `ddl::File_reader` |
| `ddl0rtree.cc` | 6.4 KB | 空间索引构建（逐行 rtree 插入） | `ddl::RTree_inserter` |

头文件：`ddl0ddl.h`（Context/Sequence/Dup/Index_defn/IO 常量）、`ddl0impl.h`（file_t/IO_buffer/Row/pread/pwrite）、`ddl0impl-builder.h` / `-buffer.h` / `-file-reader.h` / `-merge.h` / `-loader.h` / `-cursor.h` / `-rtree.h` / `-compare.h`（`Compare_key`，排序比较器 + dup 检测）、`ddl0fts.h`。

### 协作图：从 server 到 B+Tree

```
server 层
  ha_innobase::prepare_inplace_alter_table()
      └─ ddl::create_index(trx, new_table, &index_defs[a], add_v)   [ddl0ddl.cc]
             → dict_mem_index_create → dict_build_index_def
             → dict_index_add_to_cache_w_vcol → dict_create_index_tree_in_mem
             → ctx->add_index[a] = index

  ha_innobase::inplace_alter_table() → inplace_alter_table_impl()
      └─ ddl::Context ddl(trx, old_table, new_table, online, add_index, ...,
                          thd_ddl_buffer_size(thd),   /* innodb_ddl_buffer_size */
                          thd_ddl_threads(thd));      /* innodb_ddl_threads     */
         └─ clean_up(ddl.build())

ddl::Context::build()                                        [ddl0ctx.cc]   ★ 真正的总入口
  Loader loader{*this};  return cleanup(loader.build_all());

Loader::build_all()                                          [ddl0loader.cc]
  ├─ prepare()      : 每个待建索引 new 一个 ddl::Builder(ctx, *this, i)
  └─ scan_and_build_indexes()
       ├─ Cursor::create_cursor(ctx) → Parallel_cursor      [ddl0par-scan.cc]
       ├─ ctx.read_init(cursor); cursor->open()
       ├─ cursor->scan(m_builders)        /* 扫描期：每行 → builder->add_row() */
       ├─ cursor->finish(err)
       └─ load()                          /* 排序 + 建树期：任务队列驱动 */

Builder: 每索引一个实例，内部 m_thread_ctxs（每扫描线程一份）
   Thread_ctx = { Key_sort_buffer*, n_recs, 临时文件 file_t,
                  对齐 IO buffer, Merge_offsets(run 起点), RTree_inserter* }
```

### ★ `ddl::Builder` 的状态机

源码枚举注释：`note: Build phase/states`

```
INIT → ADD → SETUP_SORT → SORT → BTREE_BUILD → FINISH → STOP
                    ↘ FTS_SORT_AND_BUILD ↗          （异常随时 → ERROR）
```

`set_next_state()` 的迁移规则（真实逻辑）：

- 任何时刻 `get_error() != DB_SUCCESS` → 直接 `set_state(ERROR)`
- `INIT` → `ADD`
- `ADD` → 若 `is_fts_index()` → `FTS_SORT_AND_BUILD`；否则若 `!is_skip_file_sort()` → `SETUP_SORT`；否则（**主键且已知有序**）→ `FINISH`
- `SETUP_SORT` → `SORT` → `BTREE_BUILD` → `FINISH` → `STOP`

`is_skip_file_sort()` = `m_ctx.m_skip_pk_sort && m_sort_index->is_clustered()`，即"主键按既有顺序，跳过排序"。

#### ★ 不存在 `Builder::build()`：主入口在 Loader 侧

Builder 只有阶段方法，**没有总入口**。真正的驱动链是：

```
Loader::build_all()            ddl0loader.cc:484
  ├─ prepare()                 :370   为每个索引 new 一个 Builder
  ├─ scan_and_build_indexes()  :402
  │    └─ cursor->scan(m_builders)   :455  → Parallel_cursor::scan  ddl0par-scan.cc:96
  │         └─ builder->add_row(...) / builder->batch_insert(...)
  └─ load()                    :468 → 290   推进状态、建任务队列
```

★ 一个容易读错的点：`Builder::init()`（`ddl0builder.cc:630`）结尾调用了 `set_next_state()`，把 INIT 推进到 **ADD**——所以 `Loader::load()` 里才敢直接断言 `builder->get_state() == State::ADD`（`ddl0loader.cc:303`）。状态是被 `init` 推过一格的，不是停在上一个阶段。

#### 状态机的迁移规则

```cpp
// ddl0builder.cc:2085
void Builder::set_next_state() {
  if (get_error() != DB_SUCCESS) { set_state(State::ERROR); return; }   // ★ 一步短路
  switch (get_state()) {
    case State::ADD:
      if (is_fts_index())            set_state(State::FTS_SORT_AND_BUILD);
      else if (!is_skip_file_sort()) set_state(State::SETUP_SORT);
      else { set_state(State::FINISH); break; }        // ★ 跳过排序+建树
    case State::SORT:   set_state(State::BTREE_BUILD); break;
    case State::FINISH: set_state(State::STOP); break;
  }
}
```

三条：

1. ★ **任何错误一步短路到 `ERROR`**，不再迁移（`STOP`/`ERROR` 分支里是 `ut_error`）。所以失败后状态不会"继续往前走"，而是停住等清理。
2. ★ `is_skip_file_sort()` 时 **ADD 直接跳到 FINISH**——即数据小到不值得落临时文件时，跳过整个排序+建树阶段。
3. 头文件（`ddl0impl-builder.h:51`）的注释特别说明这个枚举 **"看似 state，实则 phase"**——它标记的是"进行到哪一阶段"，不是传统意义上可自由迁移的状态机。调度器 `Loader::Task::operator()`（`ddl0builder.cc:2135`）就是按它分派到 `setup_sort` / `merge_sort` / `btree_build` / `fts_sort_and_build` / `finish`。

#### ★ 三种索引走三条不同的路

`add_row()`（`ddl0builder.cc:1575`）是分派点：

```cpp
  auto err = m_ctx.check_state_of_online_build_log();
  if (err != DB_SUCCESS)            err = online_build_handle_error(err);
  else if (is_spatial_index()) { if (!cursor.eof()) err = batch_add_row(row, thread_id); }
  else                              err = bulk_add_row(cursor, row, thread_id, ...);
```

| 索引类型 | 路径 | 特点 |
|---|---|---|
| **B-tree** | `bulk_add_row` → `add_to_key_buffer` → `copy_row` | 写 `Key_sort_buffer`，满则落临时文件 |
| **R-tree（空间）** | `batch_add_row`（`:1329`）**只做浅拷贝加入批次** | ★ **不写 key_buffer、不落临时文件**；真正插入由扫描线程在 **PAGE 边界**触发 `batch_insert`（`ddl0par-scan.cc:283`） |
| **FTS（全文）** | 走 `bulk_add_row`，但 `:1513` `if (key_buffer->is_fts()) return DB_SUCCESS;` | ★ **不写临时文件**，交给 `fts_sort_and_build`（`:1924`）一步完成 |

★ R-tree 最特殊：`Loader::load()` 里显式排除它——

```cpp
  /* RTrees are built during the scan phase, using row by row insert. */
  if (!builder->is_spatial_index()) { builder->set_next_state(); add_task(Task{builder}); }
```

⇒ 也就是说**空间索引在扫描阶段就逐行建好了**，根本不参与后面的排序/归并/装载流水线。这是三种索引里唯一"边扫边建"的。

#### ★ `DB_OVERFLOW` 被当作"放不下"而非错误

`copy_row()`（`:1037`、`:1056`、`:1122`）在 buffer 装不下时返回 `DB_OVERFLOW`——**这是控制流信号，不是故障**。调用方据此腾空间后重试：

```cpp
// ddl0builder.cc:1442
  err = add_to_key_buffer(ctx, mv_rows_added);
  if (err != DB_OVERFLOW) return err;          /* Need to make room ... and retry. */
  ...
  if (!key_buffer->empty()) { err = key_buffer_sort(thread_id); ... }   // 先内存内排序
  if (is_skip_file_sort()) { err = insert_direct(cursor, thread_id); key_buffer->clear(); }
  else err = key_buffer->serialize(io_buffer, persistor);               // 否则落临时文件
```

（多值索引还允许"半行入 buffer"——`:1363` 注释说明 *partial row due to size limit*。）

### 扫描期：行进入 Builder 的路径

**`init(Cursor&, size_t n_threads)`** —— 扫描开始前对每个 builder 调用：

- 建 per-builder 的 `Alter_stage`（PFS 进度），注释 `Each builder is responsible for building a single index.`
- `buffer_size = m_ctx.scan_buffer_size(n_threads)`（pair{sort buffer, io buffer}），为每个线程建 `Thread_ctx`：`Key_sort_buffer(index, buffer_size.first)` + `ut::make_unique_aligned<byte[]>(UNIV_SECTOR_SIZE, buffer_size.second)`；空间索引额外建 `RTree_inserter`
- FTS 分支：按 `DICT_TF2_FLAG_IS_SET(new_table, DICT_TF2_FTS_ADD_DOC_ID)` 决定用 `Gen_sequence`（生成 doc id）还是 `Fetch_sequence`（从行里取），然后 `fts.m_ptr->start_parse_threads(this)`
- `is_skip_file_sort()` 时预建 `Btree_load`

**`add_row()`** → 空间索引走 `batch_add_row()`，其余走 `bulk_add_row()`：

```c
do {
  err = add_to_key_buffer(ctx, mv_rows_added);   // 返回 DB_OVERFLOW = key buffer 满
  key_buffer_sort(thread_id);                     // 排序
  if (is_skip_file_sort()) {
    insert_direct(cursor, thread_id);             // 直接灌 m_btr_load，不落盘
    key_buffer->clear();  m_ctx.note_max_trx_id();
  } else {
    create_file();                                // 首次打开临时文件
    thread_ctx->m_offsets.push_back(file.m_size); // ★ 该 run 的起始偏移
    key_buffer->serialize(io_buffer, persistor);  // 追加写盘
    clear();
  }
} while (!cursor.eof());
```

⇒ **key buffer 满 → 排序 → 追加写临时文件**，所以"每个扫描线程 × 每个索引"产出一个临时文件，文件内含**多个已排序 run**（`Merge_offsets` 记每个 run 起点）。

### ★ 排序期：是"多趟 2 路"，**不是**一次 N 路

头文件注释原文：*\"The design is generalized as an N way merge, however we stick with 2 for now.\"* → `static constexpr size_t N_WAY_MERGE = 2;`

> ⚠️ **纠正一个容易写错的点**：**"多趟 2 路 + 两临时文件 ping-pong"不是新版引入的**——老 `row0merge.cc` 就是这么做的（`row_merge()` 每趟把文件按 `run_offset[num_run/2]` 对半切、逐 block 两两合并，趟末 `*tmpfd = file->fd; *file = of;` 交换 fd；`row_merge_sort()` 就是 `do { row_merge() } while (num_runs > 1)`）。
> **真正的差别是"谁并行"**：老版本一趟里所有 block 对的合并都在**单线程**里串行做完；新版每个扫描线程各自归并自己的文件，且这些归并是**并行的 Task**（见 8.7）。算法层面新旧同构。

```c
while (offsets.size() > 1) {
    Output_file output_file(ctx, tmpfd, io_buffer);
    Cursor cursor{builder, file, m_dup, m_stage};
    err = merge_ranges(cursor, offsets, output_file, io_buffer_size);
    /* Swap the input file with the output file and repeat. */
    tmpfd.swap(file->m_file);
    std::swap(offsets, m_next_offsets);
    file->m_size = output_file.get_size();
}
```

⇒ **两个临时文件 ping-pong 互换**，每趟把所有 run 两两合并成一半，直到只剩一个。

- 缓冲区：`n_buffers = (m_n_threads * N_WAY_MERGE) + 1`，`io_buffer_size = ctx.merge_io_buffer_size(n_buffers)`
- 输入侧：`Merge_file_sort::Cursor::prepare()` 对**同一个文件**建两个 `File_reader`（两个起始偏移），再 `Merge_cursor::open()`
- `Merge_cursor` = `std::priority_queue<File_cursor*, File_cursors, Compare>` —— 真正的"多路"在这里（归并阶段恒 2 路，**建树阶段才是 N 路**）
- 输出侧 `Output_file::flush()` 写 1 字节 `0` 作为 **end-of-list 标记**（注释：*\"There must always be room to write the end of list marker.\"*）
- 中断检查节流：`TRX_INTERRUPTED_CHECK = 64`，注释 *\"The transaction interrupted check is expensive, we check after this many page writes.\"*

**重复键检查的巧思**：

```c
auto dup = cursor.size() == 0 ? m_merge_ctx->m_dup : nullptr;
```

⇒ **只有在"单路 append"（实际没归并）时才把 dup 传给写路径**；两路归并过程中不查。归并中的重复检测改由 `Output_file::duplicate_check()` 顺序比较相邻记录（`cmp_rec_rec_simple`，`cmp <= 0` → `dup->report()`）。

`setup_sort()` 里还有一条：某线程**只有 1 个 run 且是唯一索引** → 先 `check_duplicates()`（`Merge_cursor` + `Compare_key`）扫一遍查重。

### `ddl::Context`：它持有什么，不持有什么

`ddl::Context`（`ddl0ddl.h:321`）是**栈上对象**，一次 inplace_alter 一份：

```cpp
// handler0alter.cc:6348
ddl::Context ddl(trx, m_prebuilt->table, ctx->new_table, ctx->online, ...,
                 thd_ddl_buffer_size(...), thd_ddl_threads(...));
const auto err = clean_up(ddl.build());
```

| 类别 | 字段 |
|---|---|
| 事务/表 | `m_trx`、`m_old_table` / `m_new_table`、`m_online`、`m_stage` |
| 索引 | `m_indexes`、`m_key_numbers`、`m_n_uniq`、`m_cursor` |
| 内存 | `m_dtuple_heap`（仅 add_cols 时建）、**`m_max_buffer_size`**、`m_max_threads` |
| 统计/错误 | `std::atomic<dberr_t> m_err`、`m_err_key_number`、`m_fts`、`m_nonnull` |
| 并发 | `m_autoinc_mutex`、`m_sequence` |

★ 关键在于**它不持有什么**：**临时文件不在 ctx 里**（在 `Builder::Thread_ctx::m_file`），**线程也不在**（在 `Loader::Task_queue`）。ctx 只是"参数与错误汇总中心"。

生命周期：

- 构造（`ddl0ctx.cc:46`）：`trx_start_if_not_started_xa`、按需新建 `Flush_observer`、`mutex_create`；
- 析构（`:126`）**只做两件事**：`mem_heap_free(m_dtuple_heap)` + `mutex_destroy`；
- ★ 异常清理走 `Context::cleanup()`（`:300`）：给未完成的索引打 **`DICT_CORRUPT`** 并置 `drop_aborted = true`（`:323`），然后 `observer->interrupted()/flush()/delete_`；成功路径才 `Builder::write_redo`（`:378`）。

**两个上限**（`ha_innodb.cc:1114`）：

```cpp
MYSQL_THDVAR_ULONG(ddl_buffer_size, ..., 1048576, 65536, 4294967295, 0);  // 默认 1MB
MYSQL_THDVAR_ULONG(ddl_threads,     ..., 4, 1, 64, 0);                    // 默认 4
```

★ `scan_buffer_size()`（`ddl0ctx.cc:149`）按 `m_max_buffer_size / n_threads` 切分，且最后 `size.first -= size.second`（`:195`）——**排序缓冲与 IO 缓冲共享同一份预算**，不是各自独立分配。

（一处值得留意的死代码：`allocate()` 与 `m_n_allocated`（`ddl0ddl.h:486/646`）全仓库**无定义、无调用**——内存预算实际靠上面的切分，而非运行时记账。）

### 缓冲与临时文件：buffer 满 = block = run

```cpp
// ddl0impl-buffer.h:89
bool full() const noexcept { return size() >= m_max_tuples; }
bool will_fit(size_t n) const noexcept {   /* 预留 1 字节 end marker + 元数据开销 */
  return m_total_size + n + (sizeof(...) * (m_n_tuples + 1)) <= m_buffer_size - 1; }
```

buffer 满之后的三步**顺序不能换**（`ddl0builder.cc:1442`）：

```cpp
  err = key_buffer_sort(thread_id);                     // ① 先内存内排序（+去重）
  thread_ctx->m_offsets.push_back(file.m_size);          // ② ★ 记下该 block 起始偏移 = 一个 run
  err = key_buffer->serialize(io_buffer, persistor);     // ③ 再落临时文件
```

⇒ `m_offsets` 的**个数就是 run 数**——后续 `create_merge_sort_tasks`（`:1842`）靠它决定归并任务。

#### ★ 临时文件：无命名、无复用、无显式删除

```cpp
// ddl0ddl.cc:129
auto fd = innobase_mysql_tmpfile(path);        // NULL → innobase_mysql_tmpdir()
if (srv_disable_sort_file_cache) os_file_set_nocache(file.m_file.get(), ...);
```

★ 三条都反直觉：

1. **创建即 unlink**——`create_temp_file(..., UNLINK_FILE)`（`mysys/mf_tempfile.cc:271`）创建后立刻取消链接，文件系统里**看不到这个临时文件**；
2. **回收只靠 fd 析构**——`Unique_os_file_descriptor` 析构时 `close`（`ddl0ddl.h:171`），没有显式 `unlink`/删除代码；
3. **没有复用或缓存**——每次都是新文件，唯一的"缓存"开关是 `srv_disable_sort_file_cache`（控制是否 `O_DIRECT`）。

⇒ 这也是为什么 DDL 期间 `lsof` 能看到大量已删除的临时文件句柄。

#### 2 路归并的 ping-pong：交换 fd 与偏移表

`N_WAY_MERGE = 2`（`ddl0impl-merge.h:50`），多趟归并靠**两两互换**实现：

```cpp
// ddl0merge.cc:502
auto tmpfd = ddl::file_create_low(builder->tmpdir());
while (offsets.size() > 1) {
  Output_file output_file(ctx, tmpfd, io_buffer);
  Cursor cursor{builder, file, m_merge_ctx->m_dup, ...};
  err = merge_ranges(cursor, offsets, output_file, io_buffer_size);
  tmpfd.swap(file->m_file);              // ★ 输入/输出文件描述符互换
  std::swap(offsets, m_next_offsets);    // ★ run 偏移表互换，成为下一趟的输入
  if (!offsets.empty()) file->m_size = output_file.get_size();
}
```

★ 只分配**一份** IO 缓冲（`ddl0merge.cc:491`），大小 `merge_io_buffer_size(n_buffers)`，`n_buffers = m_n_threads * 2 + 1`。

IO 层的三条约定：

- `constexpr size_t IO_BLOCK_SIZE = 4 * 1024`（`ddl0ddl.h:65`）；`write_buffer` 用 `ut_uint64_align_down` 落盘整块、残余 `memmove` 到头部（`ddl0buffer.cc:126`）；
- 结尾写 0 填充到 `align_up(buf_filled + 1)`，即 **end-of-chunk 标记**（`:218`）；
- `pwrite`/`pread` 后都调 `posix_fadvise(..., POSIX_FADV_DONTNEED)` **主动丢弃页缓存**（`ddl0ddl.cc:103`）——临时文件读一次就不用了，不值得占 OS 缓存。

### ★ 建树期：**归并并行、建树串行**

`btree_build()`（单线程）：

- 把每个 `thread_ctx->m_file` 作为一路 `Merge_cursor::add_file()`（**N 路**）
- 建 `Btree_load btr_load(m_index, trx->id, observer)`，`cursor.open()` → `btr_load.build(cursor)` → `btr_load.finish(...)`
- 完成后 `set_next_state()` 并 `m_loader.add_task(Loader::Task{this})` → `FINISH`

**并行/串行的落点在 `merge_sort()`**：

```c
m_n_sort_tasks.fetch_sub(1);
// 只有当 n == 1 且状态仍是 SORT 时才 set_next_state() 并投一个（单个）新任务
```

⇒ 每个线程各自归并自己那份临时文件（**并行**），全部归并完才投**一个**建树任务（**串行**）。源码里还有一处 `// todo: parallel sort and build` 注释——官方也认为还有并行化空间。

**`finish()` / `finalize()`** 三分支：

| 条件 | 做什么 |
|---|---|
| 出错 或 `!m_ctx.m_online` | 什么都不做（`Do not apply any online log.`） |
| `m_old_table != m_new_table`（rebuild） | 只 `observer->flush()`（row log 由 server 后续 `row_log_table_apply`） |
| 否则 | `finalize()`：`observer->flush()` → `Clone_notify(SPACE_ALTER_INPLACE_BULK)` → `write_redo(m_index)` → `row_log_apply()` |

`write_redo()` 写一条 **`MLOG_INDEX_LOAD`** redo（`11 + 8` 字节，含 `index->id`），标记"该索引的页级 redo 被禁用且已刷盘"。

#### 批量建树到底怎么"长"出这棵树：`Btree_load::insert`

归并产出的是**有序记录流**，把它变成 B+Tree 靠的是 `Btree_load`（`btr/btr0load.cc`）。★ 关键认知是：**它是自底向上长的**——先填满叶子层，满了才往上加一层，而不是先建根再往下分裂：

```cpp
dberr_t Btree_load::insert(dtuple_t *tuple, size_t level) noexcept {
  bool is_left_most{};
  dberr_t err{DB_SUCCESS};

  /* Check if we need to create a Page_load for the level. */
  if (level + 1 > m_page_loaders.size()) {        // ① 出现了新的更高层
    auto page_loader = ut::new_withkey<Page_load>(
        UT_NEW_THIS_FILE_PSI_KEY, m_index, m_trx_id, FIL_NULL, level,
        m_flush_observer);
    ...
    m_page_loaders.push_back(page_loader);        // 每层一个 Page_load
    ut_a(level + 1 == m_page_loaders.size());
    m_root_level = level;                         // ② ★ 树高随插入动态增长
    is_left_most = true;
    if (level > 0) page_loader->release();        // 非叶层用完即放
  }

  auto page_loader = m_page_loaders[level];
  if (level > 0) page_loader->latch();

  if (is_left_most && level > 0 && page_loader->get_rec_no() == 0) {
    /* The node pointer must be marked as the predefined minimum
    record, as there is no lower alphabetical limit to records in
    the leftmost node of a level: */
    const auto info_bits = dtuple_get_info_bits(tuple) | REC_INFO_MIN_REC_FLAG;
    dtuple_set_info_bits(tuple, info_bits);       // ③ ★ 最左节点打标记
  }

  big_rec_t *big_rec{};
  auto rec_size = rec_get_converted_size(m_index, tuple);

  if (page_loader->need_ext(tuple, rec_size)) {   // ④ 溢出列
    /* The record is so big that we have to store some fields
    externally on separate database pages */
    big_rec = dtuple_convert_big_rec(m_index, nullptr, tuple);
    ...
```

四点解释了这个设计的由来：

1. **`m_page_loaders[]` 是"每层一个 Page_load"的数组**——`m_page_loaders[level]` 就是当前该层正在填充的页加载器；`level + 1 > size()` 意味着**第一次出现这个层级**，于是新建。非叶层（level > 0）只在需要写 node pointer 时才 latch/使用，写完 `release()`。
2. ★ **`m_root_level` 是动态增长的**——插入过程中它不断被抬高。这正是"自底向上"的直接证据：树不是预先分配好高度再填，而是**叶子填不下就分裂并向上一层追加节点指针**，层数是被"顶"出来的。
3. ★ **`REC_INFO_MIN_REC_FLAG`**：某层最左节点的第一条 node pointer 必须打上"预定义最小记录"标记——源码注释解释得清楚：*"there is no lower alphabetical limit to records in the leftmost node of a level"*。最左节点**没有下界**，如果不显式标记为最小记录，范围扫描/定位到该层最左时会找不到合法入口。这是 B+Tree 里那条著名的"infimum"在 bulk load 路径上的体现。
4. **溢出列在插入时就地处理**：`need_ext()` 判断记录太大（超出页内可容纳），立刻 `dtuple_convert_big_rec` 把部分字段转到外部页。注意第一条断言：只有**聚簇索引且 level == 0** 才允许 big_rec——溢出只发生在叶子层的聚簇记录上。

⇒ 由此也能理解"建树串行"的必然性：整棵树由**一个 `Btree_load` 实例**持有各层的 `Page_load`，记录必须**严格有序**地依次插入（否则 `REC_INFO_MIN_REC_FLAG`、层高增长、页分裂的判据全都不成立）。所以 8.0 可以并行**归并**（产出有序流），但把有序流装进树这一步只能串行——这正好对应上节"归并并行、建树串行"。

#### ★★ 反直觉：`Loader::load()` 本身不做装载

看名字会以为它是装载主循环，其实**它只建队列、拉线程、把每个 builder 推进一态并投递一个任务**（`ddl0loader.cc:290`）：

```cpp
for (auto builder : m_builders) {
  ut_a(builder->get_state() == Builder::State::ADD);
  /* RTrees are built during the scan phase, using row by row insert. */
  if (!builder->is_spatial_index()) {
    builder->set_next_state();
    add_task(Task{builder});
```

真正的"逐条读有序记录 → 插入 B-tree"在 `Builder::btree_build()`（`ddl0builder.cc:1752`）：把各归并临时文件挂到 `Merge_cursor`，交给 `btr_load.build(cursor)`。而 `Btree_load::build()`（`btr0load.cc:1304`）才是那个主循环：

```cpp
while ((err = cursor.fetch(dtuple)) == DB_SUCCESS) { ... insert(dtuple, 0); ... cursor.next(); }
```

#### 线程模型：并行发生在"每个索引一个任务"，不是"读并行"

```cpp
// ddl0loader.cc:452
/* Read clustered index of the table and create files for secondary
index entries for merge sort and bulk build of the indexes. */
// todo: parallel read
err = cursor->scan(m_builders);
```

★ **装载阶段没有独立的读线程**——读聚簇索引由调用线程串行完成，源码里还留着 `// todo: parallel read`。并行只发生在"**每个二级索引一个 build 任务**"这一层（以及归并阶段的多线程）。

线程构成：前台用户线程 1 个 + `N-1` 个 `std::thread`，**全部跑同一个消费者** `Task_queue::execute`（`ddl0loader.cc:332`）。同步原语是 `ib_mutex_t m_mutex` + `os_event_t m_consumer_event`，任务容器是**无界** `std::deque`。

线程数 = `m_ctx.m_max_threads` ← `thd_ddl_threads()` ← sysvar **`innodb_ddl_threads`，默认 4，范围 1..64**（`ha_innodb.cc:1120`）。`<= 1` 时退化为同步 `st_execute()`，**连锁和 event 都不创建**。

#### ★★ 队列终止：不用毒丸，靠"空闲线程数 == 活跃线程数"

```cpp
// ddl0loader.cc:105
 1. Some initial tasks are added before threads come here to execute tasks.
 2. While executing, a task can generate more tasks. That is the only way
    a task can be added.
 [1] & [2] implies that when all threads are idle, all tasks are completed
    and no more tasks can be added.
  ...
      while (m_tasks.empty()) {
        const auto sig_count = os_event_reset(m_consumer_event);
        ++m_n_idle;
        if (m_n_idle >= m_n_threads) {
          --m_n_threads; --m_n_idle;
          mutex_exit(&m_mutex);
          os_event_set(m_consumer_event);      // ★ 唤醒别人，形成级联退出
          return DB_SUCCESS;
        }
```

★ 这个终止条件是**靠前置约束推导出来的**，不是靠显式的"结束任务"：因为任务只能由任务产生（[2]），所以当所有线程都闲着时，必然不可能再有新任务。⇒ 不需要毒丸（poison pill）。

★ 另一个细节：退出的线程必须 `os_event_set` **再唤醒别人**，否则剩下的线程会一直等在 event 上——这形成**级联退出**。

#### 错误处理：只保留第一个错误

- 出错线程退出循环后 `--m_n_threads` 并 `os_event_set`；
- 被唤醒的线程**主动查全局错误**（`ddl0loader.cc:146`）：`err = m_ctx.get_error(); if (err != DB_SUCCESS) { ... return err; }`；
- ★ 汇总用 CAS **只保留第一个**（`ddl0ddl.h:427`）：`dberr_t expected{DB_SUCCESS}; m_err.compare_exchange_strong(expected, err);`——后续错误被丢弃，不会覆盖第一条。

#### `Load_cursor` 为什么存在

三个 Cursor 各司其职，容易混淆：

| | 位置 | 作用 |
|---|---|---|
| `ddl::Cursor` | `ddl0impl-cursor.h:41` | **扫描期**数据源抽象（`scan(Builders&)`），与装载无关 |
| `Btree_load::Cursor` | `btr0load.h:54` | **装载器消费端**纯虚接口，仅 `fetch` / `next` / `duplicates_detected` |
| ★ `Load_cursor` | `ddl0impl-builder.h:458` | 实现侧公共基类——让 `Btree_load` 只依赖接口，而实现需要 builder 上下文 + 重复键上报 + 行级临时内存 |

```cpp
struct Load_cursor : Btree_load::Cursor {
  [[nodiscard]] bool duplicates_detected() const noexcept override;
  Dup *m_dup{};            // 重复键收集
  dberr_t m_err{DB_SUCCESS};
  Builder *m_builder{};
  Scoped_heap m_tuple_heap{};   // raw row → dtuple_t 转换的临时内存，每次 fetch 前 clear()
```

`Merge_cursor : Load_cursor`（`:486`）用**优先队列**做多路归并，产出全局有序流；且每次 `fetch` 前还会 `m_builder->check_state_of_online_build_log()`（在线构建日志的状态检查）。

#### 收尾

- `Btree_load::finish`（`btr0load.cc:1273`）：`m_page_loaders` 为空 ⇒ 表是空的，直接返回；否则 `finalize_page_loads()` + `load_root_page()`。
- `Builder::btree_build` 尾部有 ★ `ut_a(err != DB_SUCCESS || total_rows == cursor.get_n_rows());`——装载行数必须与游标报告的行数一致。
- ★ **统计信息不在 loader 里**：在 commit 阶段由 `alter_stats_norebuild` / `alter_stats_rebuild` → `dict_stats_update_for_index`（`handler0alter.cc:7350`）完成。

### ★ `ddl::Loader`：线程数、队列终止、错误处理

**线程数怎么决定**：

- `Loader::load()` 里 `const bool sync = m_ctx.m_max_threads <= 1;`
- `m_max_threads` 来自 `thd_ddl_threads(thd)`（session 变量 **`innodb_ddl_threads`**，默认 4）
- 内存预算来自 `thd_ddl_buffer_size(thd)`（`innodb_ddl_buffer_size`）
- 真正**额外**创建的线程数是 **N-1**：`for (size_t i = 1; i < m_ctx.m_max_threads; ++i) threads.push_back(std::thread{fn, i});` —— 第 0 个"线程"就是用户线程自己
- 同步模式走 `st_execute()`：当前线程串行把队列跑完
- ⚠️ **扫描阶段的线程数是另一套**：`thd_parallel_read_threads(thd)`（`innodb_parallel_read_threads`），经 `Parallel_reader::available_threads()` 折算——**两套线程数独立**

**队列终止条件**（源码注释原文，值得直接引用）：

```
1. Some initial tasks are added before threads come here to execute tasks.
2. While executing, a task can generate more tasks. That is the only way
   a task can be added.

[1] & [2] implies that when all threads are idle, all tasks are completed
and no more tasks can be added.

We exit here when all running threads are idle.
```

实现用 `m_n_idle >= m_n_threads` 判定，完整循环如下（含三个容易写错的并发细节）：

```cpp
dberr_t mt_execute() {
  ut_a(!m_sync);
  dberr_t err{DB_SUCCESS};

  do {
    mutex_enter(&m_mutex);

    while (m_tasks.empty()) {
      // ★ 防丢唤醒：先取当前 signal 计数作基线，再把自己计入空闲
      const auto sig_count = os_event_reset(m_consumer_event);

      ++m_n_idle;

      if (m_n_idle >= m_n_threads) {    // ★ quiescence：全员空闲 ⇒ 不可能再有新任务
        ut_a(m_n_threads > 0);
        --m_n_threads;                  // 退出连锁：分母减一
        ut_a(m_n_idle > 0);
        --m_n_idle;
        mutex_exit(&m_mutex);

        os_event_set(m_consumer_event); // ★ 必须唤醒下一个，否则它永远睡在 wait 上

        return DB_SUCCESS;
      }

      mutex_exit(&m_mutex);

      os_event_wait_low(m_consumer_event, sig_count);  // ★ 带基线等：不吃掉期间的信号

      err = m_ctx.get_error();          // 被唤醒后先查全局错误位

      if (err != DB_SUCCESS) {
        mutex_enter(&m_mutex);
        ut_a(m_n_threads > 0);
        --m_n_threads;
        ut_a(m_n_idle > 0);
        --m_n_idle;
        mutex_exit(&m_mutex);

        os_event_set(m_consumer_event);  // 把"出错"这件事传播下去

        return err;
      }

      os_event_reset(m_consumer_event);
      mutex_enter(&m_mutex);
      --m_n_idle;                        // 有新任务了，退出空闲态
    }

    auto task = m_tasks.front();
    m_tasks.pop_front();

    mutex_exit(&m_mutex);

    err = task();                        // ★ 任务在锁外执行

  } while (err == DB_SUCCESS);

  mutex_enter(&m_mutex);
  ut_a(m_n_threads > 0);
  --m_n_threads;
  mutex_exit(&m_mutex);

  os_event_set(m_consumer_event);

  return err;
}
```

四个要点（每一个都是并发代码里典型的坑）：

1. **防丢唤醒**：`os_event_reset()` 返回的 signal 计数要原样传给 `os_event_wait_low()`。少了这个配合，reset 与 wait 之间到达的信号会被吃掉，线程可能永久睡眠。
2. **退出连锁**：准备退出的线程除了 `--m_n_threads`，**还必须 `os_event_set()`**——分母变小后要唤醒别人重新判定，否则被唤醒者看到的还是旧分母，全员睡死。
3. **quiescence 为什么成立**：完全依赖注释 [1][2] 那条不变量——**任务只能在任务执行过程中产生**。所以"所有线程同时空闲"等价于"不会再有新任务"。这比引用计数简洁，代价是必须严守"任务外不得 add_task"。
4. **锁外执行**：`mutex_exit()` 之后才调 `task()`——排序和建树是耗时操作，占着队列锁会把整个队列串行化掉。

同步模式（单线程）走的是另一个函数 `st_execute()`，直接 `while (!m_tasks.empty())` 串行跑完，构造时连锁和事件都不创建。

**错误处理（三处）**：

1. **线程创建失败**：`catch (...)` → `ib::warn(ER_DDL_MSG_1)` → `m_taskq->thread_create_failed()` → `break`
2. **等待中的线程被唤醒后先查全局错误**：`err = m_ctx.get_error(); if (err != DB_SUCCESS) { ...; return err; }` —— "一个线程报错，全体快速退出"
3. **`task()` 返回错误** → 该线程跳出，主线程 `join()` 后 `if (err == DB_SUCCESS) err = m_ctx.get_error();` —— **因为并发任务的错误只落在 `Context` 上，必须最后再取一次**

**其他**：

- `load()` 开头 `ut_a(builder->get_state() == Builder::State::ADD)`，且：
  ```
  /* RTrees are built during the scan phase, using row by row insert. */
  if (!builder->is_spatial_index()) { builder->set_next_state(); add_task(Task{builder}); }
  ```
  ⇒ 空间索引**根本不进排序/建树任务队列**
- `Builder::set_error(err)` → `m_ctx.set_error(err, m_id)`（**CAS，只记第一个错误**，同时记 `m_err_key_number`）

### 并行扫描（`Parallel_cursor`）：退化单线程的三个条件

⚠️ `ddl0par-scan.cc` 里**只有 `Parallel_cursor` 一个类**——8.0.39 已经没有独立的"串行扫描 cursor"了，串行只是 `n_threads == 0` 的同步读模式。

退化为单线程的**三个条件**：

1. `m_single_threaded_mode = m_ctx.has_virtual_columns() || m_ctx.has_fts_indexes()`（构造时定）。对应注释：*\"We don't do a parallel scan of the clustered index when FTS and/or virtual columns are involved. The build phase is parallel though.\"*
2. 任一 builder `is_skip_file_sort() || is_spatial_index()` → 也退单线程。注释原文：
   *\"Parallel scan will break the order. If in the future we decide to force a parallel scan then we will need to force a file sort later. Or, figure out how to \\\"stitch\\\" the lists together after the dumping the rows from the scan.\"*
3. `Parallel_reader::available_threads()` 只分到 1 个 → `n_threads = 0`

★ **分片方式的关键澄清**：**不是按行数切**，而是 `Parallel_reader::Scan_range FULL_SCAN`（全表范围），由 `Parallel_reader`（`row0pread.h/cc`）内部按页/子树切成子范围分发给各线程 ⇒ **记录顺序不保证**，这正是条件 2 存在的原因。

容错：`reader.run(n_threads)` 返回 `DB_OUT_OF_RESOURCES` → `ib::warn(ER_INNODB_OUT_OF_RESOURCES) << "Resource not available to create threads for parallel scan." << " Falling back to single thread mode."` → 对每个 builder `fallback_to_single_thread()` → `reader.run(0)`。

### `ddl::Context`：状态与内存预算

**数据成员**：`trx`、`m_fts`、`m_old_table`/`m_new_table`、`m_online`、`m_indexes`、`m_key_numbers`、`m_table`、`m_add_cols`、`m_col_map`、`m_add_autoinc`、`m_sequence`、`m_stage`、`m_add_v`、`m_eval_table`、`m_skip_pk_sort`、`m_nonnull`、`m_n_uniq`、`m_need_observer`、`m_cursor`、`m_n_allocated`、`m_max_buffer_size`、`m_max_threads`、`m_autoinc_mutex`、`m_dtuple_heap`，以及原子 `m_err` + `m_err_key_number`。

**三个值得单列的点**：

**① flush observer 规则**（构造函数注释，非常重要）：

> Since we disable redo logging in bulk load, so we should flush dirty pages before online log apply... 1. Online add index: flush dirty pages right before row_log_apply(). 2. Table rebuild: flush dirty pages before row_log_table_apply(). We use bulk load to create all types of indexes **except spatial index**, for which redo logging is enabled. If we create only spatial indexes, we don't need to flush dirty pages at all.

代码：`m_need_observer = m_old_table != m_new_table;`，然后**只要有一个非空间索引就置 true**。

**② 错误只记第一个**：`set_error()` 用 `m_err.compare_exchange_strong(expected, err)`，成功后把 `m_key_numbers[id]` 写进 `m_err_key_number`（供 `trx->error_key_num`）。

**③ 内存预算三函数**：

| 函数 | 逻辑 |
|---|---|
| `scan_buffer_size(n_threads)` | `n_buffers = n_threads * (m_fts.m_ptr ? FTS_NUM_AUX_INDEX : m_indexes.size())`；`min_io_size = srv_page_size/2 + IO_BLOCK_SIZE`（注释：一个 B-tree 节点至少能放 2 条记录 + IO 对齐）；io buffer 按阶梯选（`>=2M→1M`、`>=1M→512K`、`>=512K→128K`、`>=256K→64K`、否则 `32K`）；最后 `size.first -= size.second` |
| `merge_io_buffer_size(n)` | `max(max(srv_page_size, 2*IO_BLOCK_SIZE), load_io_buffer_size(n))` |
| `load_io_buffer_size(n)` | `max(max(srv_page_size, IO_BLOCK_SIZE), (m_max_buffer_size/n) 向下对齐到 IO_BLOCK_SIZE)` |

`handle_autoinc()` 是**唯一需要加锁**的地方（`m_autoinc_mutex` 保护 `m_sequence++`，序列耗尽 → `DB_AUTOINC_READ_ERROR`）。

`build()` 本体就三行：`Loader loader{*this}; const auto err = cleanup(loader.build_all()); ut_ad(err != DB_SUCCESS || loader.validate_indexes());`（注释：*\"Validate the indexes after the pages have been flushed to disk. Otherwise we can deadlock between flushing and is_free page check.\"*）

### 临时文件与缓冲区

**`Key_sort_buffer`（不是字节缓冲，是 `dfield_t*` 数组 + mem_heap）**：

- 判满两套：`full()` 按 `m_max_tuples`；`will_fit(n)` 按字节预算
- `sort()`：`Compare_key(m_index, dup, true)` + **自实现的 `merge_sort()`**（不是 `std::sort`），注释 *\"Compare all the columns of the key to preserve order in the index.\"*
- `serialize()`：逐条 `rec_get_serialize_size` + `rec_serialize_dtuple` 写进 IO buffer；写不下 → `write_buffer()`（按 `IO_BLOCK_SIZE` 向下对齐写盘，余下 `memmove` 到头部）；最后 `memset` 补 0 对齐作为 **end-of-chunk 标记**；单条记录 `>= io_buffer.second` → `DB_TOO_BIG_RECORD`
- 临时文件目录来自 `innobase_mysql_tmpdir()`；可用 `srv_disable_sort_file_cache`（`innodb_disable_sort_file_cache`）走 `os_file_set_nocache` 绕开 OS page cache；计数 `MONITOR_ALTER_TABLE_SORT_FILES`

**构建大索引会落盘吗？——会。**（见 8.5）

**`File_reader`**：`prepare()` 建 IO buffer + `m_aux_buf`（`UNIV_PAGE_SIZE_MAX / 2`）；`next()` 读前缀 `extra_size`（`== 0` → `DB_END_OF_INDEX`；`>= 0x80` 再读一字节），**跨块记录**用 `m_aux_buf` 把两段拼起来；构造函数做 `posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL | POSIX_FADV_NOREUSE)`。

**`ddl::pread` / `pwrite`**：都先 `request.disable_compression()`（注释 *\"Merge sort pages are never compressed.\"*），读写后 `posix_fadvise(..., POSIX_FADV_DONTNEED)` 主动释放 page cache。

### FTS 构建的模型差异

不是"排序记录 → 批量建 B+Tree"，而是"**分词 → `(word, doc_id, position)` → 写入 6 张 aux 表**"。`ddl0fts.h` 顶部注释：

> **Phase I:** 起若干 parsing/tokenization 线程，从队列取文档、解析分词、填 buffer、排序、写临时文件。**每个 parser 实例 × 每个 aux 表一个文件**，所以 2 个 parse 线程 → `2 × FTS_NUM_AUX_INDEX` 个文件。
> **Phase II:** Phase I 的临时文件**不关闭**直接交给写阶段，创建 `FTS_NUM_AUX_INDEX` 个线程，每个线程归并 2 个文件并插入新 FTS 索引。

**与 Builder 的 4 个接触点**：

1. `Builder::init()`：建 doc id 序列（`Gen_sequence` / `Fetch_sequence`）+ `start_parse_threads(this)`
2. 扫描期 `copy_fts_column()` 把文档包成 `FTS::Doc_item`，`FTS::enqueue()` 按 `doc_id % m_parsers.size()` **分片给 parser**
3. `Builder::fts_sort_and_build()`：`fts.m_ptr->insert(this)` → 关临时文件 → `ut::delete_(fts.m_ptr)`
4. `add_to_key_buffer()` 每加一行就 `fts.m_ptr->check_for_errors()`（把 parser 线程的 `Diagnostics_area` 错误回传 `thd`）

它自己的复杂度：`FTS::Parser`（每个 parser 有独立 `Diagnostics_area` 和独立 `create_internal_thd()`）；`FTS::Inserter` 自己 `trx_allocate_for_background()`、自己开 aux 表、自己建 `Btree_load` 灌数据、`fts_sql_commit(trx)` 收尾；`FTS::insert()` 起 `FTS_NUM_AUX_INDEX - 1` 个线程，最后一个留在**当前线程**（和 Loader 同样的"N-1 + 自己"模式）。

⇒ FTS 的"索引"不是单个 `dict_index_t` 而是一组 aux 表，doc id 要生成/获取，分词插件挂在 `Context` 上，并行维度从"扫描线程"变成"parser 线程 × `FTS_NUM_AUX_INDEX`"。这些与 Builder 通用路径正交，所以独立成文件（且是唯一保留 2010 年版权的头）。

### RTree：不走文件排序，且 **redo 是开着的**

- 只有一个类 `RTree_inserter`
- 扫描期 `batch_add_row()` 把 tuple **浅拷**进 `m_dtuples`；扫完一页由 `Parallel_cursor` 的 finish 回调触发 `batch_insert()`
- 每条 tuple：`rtr_init_rtr_info` → `rtr_info_update_btr` → `btr_cur_search_to_nth_level(..., PAGE_CUR_RTREE_INSERT, BTR_MODIFY_LEAF, ...)` → `btr_cur_optimistic_insert`；`DB_FAIL` 则 `mtr.commit()` 重开走 `btr_cur_pessimistic_insert`；若 `rtr_info.mbr_adj` 还要 `rtr_ins_enlarge_mbr` 上溯更新父节点 MBR
- `static constexpr ulint flag = BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG | BTR_KEEP_SYS_FLAG | BTR_CREATE_FLAG;`（不记 undo、不加锁）
- `deep_copy_tuples()`：跳过第 0 个 MBR 字段，其余 `dfield_dup`，注释 *\"Since the data of the tuple pk fields are pointers to cluster index rows. After mtr commit, these pointers could point to invalid data.\"*
- ⇒ 因为是逐行走正常 B-tree 插入（**redo 开着**），所以 `Context` 里"只有空间索引时 `m_need_observer` 可以为 false"（见 8.10 ①）

### `ddl0ddl.cc`：对外入口

| 函数 | 作用 | 调用点 |
|---|---|---|
| `create_index(trx, table, index_def, add_v)` | 建索引原型 + 建 B-tree 根页 | `handler0alter.cc` prepare 阶段：`ctx->add_index[a] = ddl::create_index(...)` |
| `drop_indexes(trx, table, locked)` | 回滚/失败时清掉在建索引 | `innobase_rollback_sec_index()`、prepare 的 drop 分支 |
| `drop_table(trx, table)` | `row_drop_table_for_mysql` | commit 阶段，注释 *\"Drop the copy of the old table, which was renamed to ctx->tmp_name at the atomic DDL transaction commit.\"* |
| `lock_table(trx, table, LOCK_X/LOCK_S)` | `lock_table_for_trx`，且 `trx->in_innodb \|= TRX_FORCE_ROLLBACK_DISABLE` | `online == false` 时 `LOCK_S`；commit 时 `LOCK_X` |
| `file_create` / `pread` / `pwrite` | 临时文件 IO | Builder / Merge / File_reader |
| `Dup::report()` | 报重复：**只报告第一条，但统计全部**（`if (m_n_dup == 1) ...`） | Builder / Output_file |

失败回滚辅助：`index_build_failed()`（标 `ONLINE_INDEX_ABORTED_DROPPED` + `table->drop_aborted = true`，虚拟列索引还要 `index->type |= DICT_CORRUPT`）、`mark_secondary_indexes()`（按 `ONLINE_INDEX_COMPLETE / ABORTED_DROPPED / CREATION / ABORTED` 四态分支）。

---

### ★ 关键函数逐行源码剖析

> 前面 8.4–8.16 是"架构与流程"，本节是**函数级代码**。每个函数给完整代码 + 逐分支讲解。

#### `Builder::bulk_add_row()`：扫描期核心循环

**签名**：`dberr_t bulk_add_row(Cursor &cursor, Row &row, size_t thread_id, Latch_release &&latch_release)`
**返回**：`DB_SUCCESS`（继续）/ `DB_END_OF_INDEX`（该 builder 取完）/ 错误码

```cpp
  size_t mv_rows_added{};                    // multi-value index 的"断点续传"计数
  auto thread_ctx = m_thread_ctxs[thread_id];
  auto key_buffer = thread_ctx->m_key_buffer;

  do {
    dberr_t err{DB_SUCCESS};
    Copy_ctx ctx{row, m_ctx.m_eval_table, thread_id};

    if (likely(!cursor.eof())) {
      err = add_to_key_buffer(ctx, mv_rows_added);
      if (err != DB_OVERFLOW) return err;    // 成功 / 真错误 → 直接返回
      /* Need to make room, flush the current key buffer to disk and retry. */
    } else if (unlikely(thread_ctx->m_n_recs == 0 && key_buffer->empty())) {
      return DB_END_OF_INDEX;                // 表为空
    }
```

三个分支：① 正常填入 → 返回；② `DB_OVERFLOW`（buffer 满）→ 往下走"排序 + 落盘"；③ EOF 且**历史上从未写过任何东西**（`m_n_recs == 0 && key_buffer->empty()`）→ 整表为空。

`mv_rows_added`：一行 JSON 数组（multi-value index）可能产生 N 条索引项，buffer 可能在中间填满 ⇒ 下次要从断点继续。

```cpp
    if (unlikely(is_fts_index() && (cursor.eof() || !m_ctx.m_fts.m_doc_id->is_generated()))) {
      return DB_SUCCESS;                     // FTS 不在这里落盘，走 ddl0fts.cc
    }
    ut_ad(m_ctx.m_old_table == m_ctx.m_new_table
              ? !key_buffer->is_clustered()
              : (m_id == 0) == key_buffer->is_clustered());
```

```cpp
    if (!key_buffer->empty()) {
      err = key_buffer_sort(thread_id);      // 排序（唯一索引顺便查重）
      if (err != DB_SUCCESS) { set_error(err); return get_error(); }

      if (is_skip_file_sort()) {             // 主键已有序 → 不落盘，直接装载
        if (!cursor.eof()) {
          /* Copy the row data and release any latches held by the parallel
          scan thread. Required for the log_free_check() during mtr.commit(). */
          err = cursor.copy_row(thread_id, row);      // ★ 深拷贝
          if (err != DB_SUCCESS) { set_error(err); return get_error(); }
          err = latch_release();                       // ★ 放 latch
          if (err != DB_SUCCESS) { set_error(err); return get_error(); }
        }
        err = insert_direct(cursor, thread_id);        // Btree_load 直接装载本批
        key_buffer->clear();
        ...
        m_ctx.note_max_trx_id(key_buffer->m_index);
        if (!cursor.eof()) continue;
        return DB_END_OF_INDEX;
      }
    }
```

★ **`is_skip_file_sort()` 分支的关键细节**：并行扫描线程持有 page latch，而 `log_free_check()`（在 `mtr.commit()` 里）要求调用者**不能持 latch**。所以必须先 `copy_row()` **深拷贝**行数据（latch 一放指针就失效），再 `latch_release()`，才能安全 `insert_direct()`。

```cpp
    /* Fulltext index read threads should not write to the temporary file directly. */
    if (unlikely(key_buffer->is_fts())) return DB_SUCCESS;

    IF_ENABLED("ddl_write_failure", set_error(DB_TEMP_FILE_WRITE_FAIL); return get_error();)

    if (!thread_ctx->m_file.m_file.is_open()) {        // ★ 懒创建临时文件
      if (!create_file(thread_ctx->m_file)) { set_error(DB_IO_ERROR); return get_error(); }
    }

    auto persistor = [&](IO_buffer io_buffer) -> dberr_t {
      auto &file = thread_ctx->m_file;
      ut_a(!(file.m_size % IO_BLOCK_SIZE));            // 文件大小始终 IO_BLOCK_SIZE 对齐
      os_offset_t n = io_buffer.second;
      ut_a(n % IO_BLOCK_SIZE == 0);                    // 写入长度也必须对齐
      auto err = ddl::pwrite(file.m_file.get(), io_buffer.first, n, file.m_size);
      if (err != DB_SUCCESS) { set_error(DB_TEMP_FILE_WRITE_FAIL); return get_error(); }
      file.m_size += n;
      return DB_SUCCESS;
    };

    auto &file = thread_ctx->m_file;
    thread_ctx->m_offsets.push_back(file.m_size);      // ★ 记录本 run 的起始【字节】偏移
    auto io_buffer = thread_ctx->m_io_buffer;
    err = key_buffer->serialize(io_buffer, persistor);
    if (err != DB_SUCCESS) return err;
    key_buffer->clear();
    m_ctx.note_max_trx_id(key_buffer->m_index);

  } while (!cursor.eof());
  return DB_END_OF_INDEX;
```

三个要点：

1. **`persistor` 是回调式 IO**——`serialize()` 不知道怎么落盘，只负责"攒够一块就调你"
2. ★ **`m_offsets.push_back(file.m_size)` 是相对老版的核心新增**：把每个 run 在文件里的**字节偏移**记下来（老版是 block 计数 + 单独 `run_offset` 数组）。后续 `Merge_file_sort::sort()` 就靠这个列表切分
3. **`IF_ENABLED`** 是故障注入宏（对应老版 `DBUG_EXECUTE_IF`）

#### `create_merge_sort_tasks()`：归并的并行化

```cpp
  for (auto thread_ctx : m_thread_ctxs) {
    thread_ctx->m_file.m_n_recs = thread_ctx->m_n_recs;
    n_runs_to_merge += thread_ctx->m_offsets.size();

    /* If there is a single file then there is nothing to merge and the
       file must already be sorted. */
    if (thread_ctx->m_offsets.size() < 2 && is_unique_index()) {
      dupcheck.push_back(thread_ctx);       // 没走过归并 ⇒ 唯一性没查过，要补查
    }
  }

  if (m_local_stage != nullptr) m_local_stage->begin_phase_sort(log2(n_runs_to_merge));

  /* Set the next state so that the tasks are executed in the context
     of the next state. */
  set_next_state();                         // SETUP_SORT → SORT ★ 必须先改状态

  m_n_sort_tasks.fetch_add(m_thread_ctxs.size(), std::memory_order_relaxed);

  for (auto thread_ctx : m_thread_ctxs) {
    if (thread_ctx->m_key_buffer != nullptr) {
      ut::delete_(thread_ctx->m_key_buffer);       // 主动还内存
      thread_ctx->m_key_buffer = nullptr;
    }
    m_loader.add_task(Loader::Task{this, thread_ctx->m_id});   // ★ 每线程一个 Task
  }
```

**为什么要 `dupcheck`**：每线程的文件内部已归并完，但**线程之间是分开的**。若某线程只写了 0 或 1 个 run（文件"天生有序"，没走归并路径），唯一性检查在归并期没做过 ⇒ 要用 `check_duplicates()` 补一次。

**三个设计点**：

1. **先 `set_next_state()` 再投任务**——`Loader::Task::operator()` 是按 `get_state()` 分派的，必须先把状态推到 `SORT`，任务被线程取到时才会走 `merge_sort()`
2. **立刻释放 `m_key_buffer`**——排序期不再需要它（老版 `row_merge_buf_t` 生命周期跟扫描绑定，做不到）
3. `m_n_sort_tasks` 原子计数 = 线程数

#### `Merge_file_sort::sort()` + 一趟内部

```cpp
  while (offsets.size() > 1) {                  // 等价老版 while (num_runs > 1)
    Output_file output_file(ctx, tmpfd, io_buffer);
    Cursor cursor{builder, file, m_merge_ctx->m_dup, m_merge_ctx->m_stage};
    err = merge_ranges(cursor, offsets, output_file, io_buffer_size);
    m_n_rows = output_file.get_n_rows();
    if (unlikely(err != DB_SUCCESS)) break;

    /* Swap the input file with the output file and repeat. */
    tmpfd.swap(file->m_file);                   // ★ ping-pong（RAII 句柄，无需 close/reopen）
    std::swap(offsets, m_next_offsets);
  }
  ut_a(err != DB_SUCCESS || file->m_n_recs == m_n_rows);   // 行数守恒（老版: of.n_rec != file->n_rec → DB_CORRUPTION）
```

`n_buffers = (m_n_threads * N_WAY_MERGE) + 1`：N 路输入各一个读 buffer + 1 个写 buffer。

`next_range()` 每次取**两个**偏移组成一个 range（对应老版"两两合并"）：

```cpp
Merge_file_sort::Range Merge_file_sort::next_range(Merge_offsets &offsets) noexcept {
  const auto size = m_merge_ctx->m_file->m_size;
  Range range{size, size};                    // 默认：到文件尾
  if (!offsets.empty()) { range.first = offsets.front(); offsets.pop_front(); }
  if (!offsets.empty()) { range.second = offsets.front(); offsets.pop_front(); }
  return range;
}
```

`merge_rows()` 是纯粹的"取最小 → 写 → 推进"：

```cpp
  while ((err = cursor.fetch(mrec, offsets)) == DB_SUCCESS) {
    /* If we are simply appending from a single partition then enable
       duplicate key checking for the write phase. */
    auto dup = cursor.size() == 0 ? m_merge_ctx->m_dup : nullptr;
    err = output_file.write(mrec, offsets, dup);
    if (unlikely(err != DB_SUCCESS)) break;
    err = cursor.next();
    if (unlikely(err != DB_SUCCESS)) break;
  }
```

⚠️ `cursor.size() == 0` 才开查重——只有单路追加（没有真正归并竞争）时才检查，多路归并时游标内部已比较过。

**`Output_file::write()`**：buffer 满时把 `IO_BLOCK_SIZE` 向下对齐的整块 `pwrite` 出去，尾巴 `memmove` 到头部继续攒；第二次还装不下 ⇒ 单条记录过大（debug 下 `ut_error`，release 下 `DB_TOO_BIG_RECORD`）。

**`Output_file::flush()`**：

```cpp
  /* There must always be room to write the end of list marker. */
  *m_ptr++ = 0;                              // end-of-chunk 标记（与老版 row_merge_write_eof 一致）
  /* Reset the duplicate checks because we are going to start merging
     a new range after the flush. */
  m_last_mrec = nullptr;
  const auto len = ut_uint64_align_up(m_ptr - m_buffer.first, IO_BLOCK_SIZE);
  ...
  if (err == DB_SUCCESS && !(m_interrupt_check++ % TRX_INTERRUPTED_CHECK) && m_ctx.is_interrupted())
    return DB_INTERRUPTED;
```

★ **中断检查按频率做**：`TRX_INTERRUPTED_CHECK = 64`——不是每次 flush 都查 `trx_is_interrupted()`（有锁开销），每 64 次查一次。老版是在 `row_merge()` 循环里每轮查，粒度更粗（一轮 = 一对 block）。

#### `Key_sort_buffer::serialize()`

```cpp
  auto write_buffer = [io_buffer, &ptr, persist]() -> dberr_t {
    persist_buffer.second = ut_uint64_align_down(ptr - io_buffer.first, IO_BLOCK_SIZE);
    auto err = persist(persist_buffer);
    if (err != DB_SUCCESS) return err;
    const auto bytes_remaining = (ptr - io_buffer.first) - persist_buffer.second;
    ptr = io_buffer.first;
    memmove(ptr, ptr + persist_buffer.second, bytes_remaining);   // 尾巴挪到头部
    ptr += bytes_remaining;
    ut_ad(bytes_remaining < IO_BLOCK_SIZE);
    return DB_SUCCESS;
  };
```

逐条序列化：算 `rec_get_serialize_size` → 编 1/2 字节 `extra_size+1` 前缀 → 装不下先 `write_buffer()` 腾地方 → 写前缀 → `rec_serialize_dtuple` 写体。

★ **EOF 标记比老版严谨**：

```cpp
  /* At this point there is some data remaining in buffer. It needs to be
     persisted, followed by zero-filled region at least 1 byte in length and
     aligned to IO_BLOCK_SIZE ("end-of-chunk" marker) */
  size_t aligned_size = ut_uint64_align_up(buf_filled + 1, IO_BLOCK_SIZE);
  if (aligned_size > io_buffer.second) {
    const auto err = write_buffer();          // 先腾出至少一整块
    ut_a(size_t(ptr - io_buffer.first) < IO_BLOCK_SIZE);
    /* After writing buffer contains [0, IO_BLOCK_SIZE) bytes, so aligning it
       to IO_BLOCK_SIZE guarantees space for end-of-chunk marker */
    aligned_size = IO_BLOCK_SIZE;
  }
  auto pad_end = io_buffer.first + aligned_size;
  memset(ptr, 0, pad_end - ptr);              // 零填充到对齐
  return persist({io_buffer.first, aligned_size});
```

老版只写 1 个 `0` 字节（因为整块写不需要对齐）；新版要求"EOF 之后零填充到 `IO_BLOCK_SIZE` 对齐"——因为写入长度必须对齐，而"加 EOF 会溢出"时先腾出 `< IO_BLOCK_SIZE` 的空间，对齐后必然是 `IO_BLOCK_SIZE`，**保证 EOF 一定有地方写**。

#### `Loader::Task::operator()`：任务分派状态机

```cpp
dberr_t Loader::Task::operator()() noexcept {
  switch (m_builder->get_state()) {
    case Builder::State::SETUP_SORT:          err = m_builder->setup_sort();         break;
    case Builder::State::SORT:                err = m_builder->merge_sort(m_thread_id); break;
    case Builder::State::BTREE_BUILD:         err = m_builder->btree_build();        break;
    case Builder::State::FTS_SORT_AND_BUILD:  err = m_builder->fts_sort_and_build(); break;
    case Builder::State::FINISH:              err = m_builder->finish();             break;
    case Builder::State::ERROR:               err = m_builder->get_error();          break;
    case Builder::State::ADD:
    case Builder::State::INIT:
    case Builder::State::STOP:
    default:                                  ut_error;
  }
  return err;
}
```

| 状态 | 动作 | 谁投的 |
|---|---|---|
| `SETUP_SORT` | 拆出 N 个 SORT 任务 | `Loader::load()` 初始化 |
| `SORT` | `merge_sort(m_thread_id)`——**需要 thread_id** | `create_merge_sort_tasks()` |
| `BTREE_BUILD` | `btree_build()`——无 thread_id（串行，内部 N 路 Merge_cursor） | 最后完成归并的线程 |
| `FTS_SORT_AND_BUILD` | → `ddl0fts.cc` 的 `FTS::insert()` | 状态机从 `ADD` 转来 |
| `FINISH` | 关临时文件 / `finalize()` | `btree_build()` 末尾 |
| `ERROR` | 直接返回已存错误，不再做工作 | `set_error()` 后 |

**任务可自我繁殖**：`setup_sort()` 投 N 个 SORT → `merge_sort()` 最后一棒投 1 个 BTREE_BUILD → `btree_build()` 末尾投 1 个 FINISH。这正是 `Task_queue::mt_execute()` 注释描述的不变量（见 8.10）。

#### `Parallel_cursor::scan()`：三个回调

**回调 1 `set_start_callback`**（仅 `IF_DEBUG`）：把工作线程的 `current_thd` 指向用户线程的 THD，使 `DEBUG_SYNC` 在并行线程里也能触发。

**回调 2 `set_finish_callback`**（页/线程结束）：

```cpp
      case Parallel_reader::State::PAGE:
        if (!batch_insert.empty()) err = batch_inserter(thread_ctx);  // 空间索引批量插入
        /* Reset the heap. Note: row.m_offsets and row.m_ptr are invalid now. */
        mem_heap_empty(m_heaps[thread_id]);
        nr += n_rows[thread_id];
        for (auto &builder : builders)
          if (builder->stage() != nullptr) {
            builder->stage()->n_pk_recs_inc(n_rows[thread_id]);
            builder->stage()->inc(1);
          }
        n_rows[thread_id] = 0;
        return err;

      case Parallel_reader::State::THREAD: {
        ut_a(n_rows[thread_id] == 0);
        auto &row = rows[thread_id];
        row.m_ptr = nullptr;  m_eof = true;
        return bulk_inserter(thread_ctx, row);     // 通知各 builder 扫描结束（flush 残 buffer）
      }
```

⚠️ `mem_heap_empty()` 之后 `row.m_offsets` / `row.m_ptr` **立刻失效**——这正是 `bulk_add_row()` 里要 `cursor.copy_row()` 深拷贝的原因。

**回调 3 `add_scan` 的逐行回调**：

```cpp
        row.m_rec = read_ctx->m_rec;
        row.m_offsets = rec_get_offsets(row.m_rec, index(), nullptr, ULINT_UNDEFINED, UT_LOCATION_HERE, &heap);
        /* We only need to copy the data when the heap is emptied. */
        auto err = row.build(m_ctx, m_index, heap, ROW_COPY_POINTERS);   // ★ 默认只拷指针
        err = bulk_inserter(read_ctx->thread_ctx(), row);
        ++n_rows[thread_id];
```

★ **`ROW_COPY_POINTERS` = 默认只拷指针不拷数据**（零拷贝），因为 heap 要到 `PAGE` 边界才清空。这是并行扫描高效的关键。

`bulk_inserter` 里的 latch 处理：

```cpp
  /* Handle the case when the bulk loader wants to do an mtr commit. The
     log_free_checks require that the caller can't be holding any latches. */
  auto bulk_inserter = [&](Thread_ctx *thread_ctx, Row &row) {
    bool latches_released{};
    for (auto builder : builders) {
      const auto err = builder->add_row(*this, row, thread_id, [&]() {
        if (!latches_released && thread_ctx->get_state() != Parallel_reader::State::THREAD) {
          thread_ctx->savepoint();  latches_released = true;
        }
        return DB_SUCCESS;
      });
      ...
    }
    if (latches_released) {
      ut_a(row.m_ptr != nullptr);
      return thread_ctx->restore_from_savepoint();    // 恢复位置
    }
    return DB_SUCCESS;
  };
```

**资源不足降级**（已在 8.11 覆盖，此处补代码）：

```cpp
    if (err == DB_OUT_OF_RESOURCES) {
      ib::warn(ER_INNODB_OUT_OF_RESOURCES) << "Resource not available to create "
        "threads for parallel scan." << " Falling back to single thread mode.";
      reader.release_threads(n_threads);
      for (auto builder : builders) builder->fallback_to_single_thread();
      err = reader.run(0);
    }
```

`fallback_to_single_thread()` 把 `m_thread_ctxs` 缩到 1（此时 builder 已 `init(n_threads)` 过了，多份 `Thread_ctx` 必须收敛）。

#### `Builder::btree_build()`：N 路归并 → Btree_load

```cpp
  Merge_cursor cursor(this, &dup, m_local_stage);
  const auto io_buffer_size = m_ctx.load_io_buffer_size(m_thread_ctxs.size());

  for (auto thread_ctx : m_thread_ctxs) {
    if (!thread_ctx->m_file.m_file.is_open()) continue;    // 该线程没落过盘（数据全在内存）
    err = cursor.add_file(thread_ctx->m_file, io_buffer_size);
    if (err != DB_SUCCESS) { set_error(err); return get_error(); }
    ut_a(thread_ctx->m_n_recs == thread_ctx->m_file.m_n_recs);   // 行数守恒
    total_rows += thread_ctx->m_n_recs;
  }
```

★ **"归并并行、建树串行"的落点在这里**：`Merge_cursor` 内部是 `std::priority_queue<File_cursor*, ..., Compare>`，`add_file()` 每调一次加一路 ⇒ N 个扫描线程 = N 路输入。

```cpp
  Btree_load btr_load(m_index, m_ctx.m_trx->id, observer);
  if (total_rows > 0) {
    err = cursor.open();
    if (err == DB_SUCCESS) {
      err = btr_load.build(cursor);           // ★ 装载器持有游标自己拉
    } else if (err == DB_END_OF_INDEX) err = DB_SUCCESS;
    cursor_err = cursor.get_err();
    if (cursor_err == DB_END_OF_INDEX) cursor_err = DB_SUCCESS;
    ut_a(err != DB_SUCCESS || total_rows == cursor.get_n_rows());
  }
```

★ **这是新老最本质的接口差异**：老版 `row_merge_insert_index_tuples()` 是外面 `for(;;) { ... btr_bulk->insert(dtuple); }`（外面 push）；新版是 `btr_load.build(cursor)`——**装载器持有游标自己拉**，这是 `Btree_load` 能内部并行的前提。

`Merge_cursor` 取数（优先队列 + 重复键上报）：

```cpp
dberr_t Merge_cursor::fetch(const mrec_t *&rec, ulint *&offsets) noexcept {
  const auto err = m_builder->check_state_of_online_build_log();
  if (err != DB_SUCCESS) return err;
  m_cursor = pop();                          // 取堆顶（当前最小）
  return m_cursor->fetch(rec, offsets);
}
dberr_t Merge_cursor::next() noexcept {
  m_err = m_cursor->next();                  // 该路前进一条
  if (likely(m_err == DB_SUCCESS)) { m_pq.push(m_cursor); ... }
  m_err = m_pq.empty() ? m_err : DB_SUCCESS;
  return m_err;
}
// 比较器里顺便查重（与老版同一个比较函数 cmp_rec_rec_simple）
  auto cmp = cmp_rec_rec_simple(...);
  if (cmp == 0) m_dup->report(l.m_mrec, &l.m_offsets[0]);
  return cmp < 0;
```

收尾：

```cpp
  /* First we check if the Btree loader returned an internal error.
     If loader succeeded then we check if the cursor returned an error. */
  err = btr_load.finish(err != DB_SUCCESS ? err : cursor_err);   // ★ 装载器错误优先
  if (err != DB_SUCCESS) set_error(err);
  set_next_state();                          // BTREE_BUILD → FINISH
  if (err == DB_SUCCESS) m_loader.add_task(Loader::Task{this});  // 只有成功才投下一个
  return get_error();
```

两个细节：① **`Btree_load` 的内部错误优先于 cursor 错误**（更能说明问题）；② **无论成败都要 `finish()`**（否则资源泄漏），但**只有成功才投下一个任务**——失败时任务链到此终止。

## 并行 DDL（社区版 8.0.27+ 确有）


月报把"并行 DDL"作为 PolarDB 能力介绍，但社区版 8.0.27+ 就引入了（`ha_innodb.cc`）：

```cpp
// 并行读（扫描聚簇索引）线程数
static MYSQL_THDVAR_ULONG(parallel_read_threads, ..., "Number of threads to do parallel read.",
                          4 /* Default */, 1 /* Min */, Parallel_reader::MAX_THREADS /* Max */, 0);

// DDL 可用内存
static MYSQL_THDVAR_ULONG(ddl_buffer_size, ..., "Maximum size of memory to use (in bytes) for DDL.",
                          1048576 /* Default: 1MB */, 65536 /* Min: 64KB */, 4294967295 /* Max */, 0);

// DDL 最大线程数
static MYSQL_THDVAR_ULONG(ddl_threads, ..., "Maximum number of threads to use for DDL.",
                          4 /* Default */, 1 /* Min */, 64 /* Max */, 0);
```

即 `innodb_ddl_threads`（默认 4，范围 1-64）、`innodb_ddl_buffer_size`（默认 1MB）、`innodb_parallel_read_threads`（默认 4）。使用点见 `handler0alter.cc`（`thd_parallel_read_threads`）、`6353`（`thd_ddl_threads`）。

**差异**：PolarDB 是调优到 15-20 倍加速；社区版有机制但加速比没那么夸张。


社区版 8.0.39 **支持**并行 DDL（`innodb_ddl_threads` 于 8.0.27 引入），但并行**仅限 InnoDB 层的部分阶段**，不是全链路并行。

### 三个参数（ha_innodb.cc）

```cpp
innodb_parallel_read_threads  // 并行扫描线程数，默认 4，范围 1~MAX_THREADS(256)
innodb_ddl_buffer_size        // DDL 排序总内存，默认 1048576(1MB)，范围 64KB~4GB-1
innodb_ddl_threads            // DDL 并行线程数，默认 4，范围 1~64
```

每个并行扫描线程分到的排序 buffer = `innodb_ddl_buffer_size / innodb_parallel_read_threads`，所以调大扫描线程数时应同步调大 buffer。

相关第四个参数 **`innodb_disable_sort_file_cache`**（默认 OFF，ha_innodb.cc）：控制 DDL 排序临时文件是否**绕过 OS page cache**。排序临时文件创建后（`ddl::file_create`，ddl0ddl.cc），若开启则对临时文件 `fcntl(fd, F_SETFL, O_DIRECT)`（`os_file_set_nocache`，os0file.cc）。默认 OFF 走 OS 缓存（归并排序反复读临时文件能命中、更快），但大排序会把热数据页挤出 OS cache；设 ON 用 O_DIRECT 避免污染，代价是排序变慢，且 tmpfs 不支持 O_DIRECT（会告警后回退）。

### 三阶段与并行粒度（关键差异）

| 阶段 | 创建二级索引（CREATE INDEX / ADD INDEX） | 重建表（ALTER ENGINE / OPTIMIZE） |
|------|----------------------------------------|--------------------------------|
| ① 扫描 | **并行**（`parallel_read_threads`，B+树子树粒度） | **单线程**（主键有序，顺序插入新树即可；该参数对重建表**不生效**） |
| ② 排序 | **并行**（`ddl_threads`，粒度 = **临时文件数 = 扫描线程数**） | **并行**（`ddl_threads`，粒度 = **索引级**，每个二级索引一个临时文件） |
| ③ 构建 B+树 | **单线程** | **并行**（`ddl_threads`，**索引级**，每个索引一个构建任务） |

**核心区别**：建索引是"**单个索引内部**并行"（多临时文件归并），重建表是"**多个索引之间**并行"。

### 源码：Loader 任务状态机

`ddl0loader.cc`：`Loader::build_all`（484）→ `prepare()` → `scan_and_build_indexes`（402）。每个索引是一个 `Builder`，由状态机驱动（`Loader::Task::operator()`，2135）：

```cpp
switch (m_builder->get_state()) {
  case Builder::State::SETUP_SORT:          err = m_builder->setup_sort();           break;
  case Builder::State::SORT:               err = m_builder->merge_sort(m_thread_id); break;
  case Builder::State::BTREE_BUILD:        err = m_builder->btree_build();           break;
  case Builder::State::FTS_SORT_AND_BUILD: err = m_builder->fts_sort_and_build();    break;
  case Builder::State::FINISH:             err = m_builder->finish();                break;
  ...
}
```

状态推进 `set_next_state()`（2108）：`SORT → BTREE_BUILD → FINISH`。work 线程从任务队列取任务，**按 Builder 当前状态**决定做排序还是构建——排序任务完成后自动追加该索引的构建任务（`ddl0loader.cc` 在扫描后为每个 builder `set_next_state()` + `add_task()`）。

**RTree（空间索引）是特例**：`ddl0loader.cc` 注释 "RTrees are built during the scan phase, using row by row insert" —— 不走 merge sort，扫描阶段逐行插入。

### 并行扫描如何分片

`Parallel_reader`（`row0pread.cc`）把 B+树按**子树**切成多个分片，每个 work 线程领一个：

1. 用户线程按 **ROOT PAGE 的子树数量预分片**（全程持 INDEX + ROOT 的 S 锁，确保不新增子树）；
2. 分片数对线程数取余的"余量分片"标记为需再分，由 work 线程**二次分片**——此时 INDEX/ROOT 锁已释放、子树结构可能变化，所以按 **RANGE**（而非子树数量）划分；
3. 每个 work 持**一个临时文件 + 一个排序 buffer**；buffer 满则内存排序后写文件 → 得到 N 个"局部有序"文件；
4. 第二阶段对这 N 个文件并行归并排序。

### Parallel_reader 的深层设计（row0pread.h）

`row0pread.h` 的类注释完整交代了设计意图，值得逐条读：

**三级执行上下文**：一个 `Parallel_reader` 可同时扫多个索引（分区表就是多个索引），每个 `add_scan(trx, config, f)`（312）注册一个 `Scan_ctx`；`Scan_ctx` 又包含多个 `Ctx`，最终被线程执行的是 `Ctx`（一个 Ctx = 一个分片范围）：

```
Parallel_reader                 // 一次并行读的总调度
  ├─ Scan_ctx[]                 // 每个 add_scan 一个（一个索引/分区一个）
  │    └─ Ctx[]                 // 该扫描切出的分片，进 run queue 被线程领取
  └─ m_parallel_read_threads[]  // 工作线程池（IB_thread）
```

**动态分片（work stealing 思想）**：类注释原文（85-91 行）——"If you have 5 sub-trees to scan and 4 threads then it will tag the 5th sub-tree as 'to_be_split'... the first thread that finishes scanning the first set of 4 partitions will then dynamically split the 5th sub-tree and add the newly created sub-trees to the execution context (Ctx) run queue"。即**余量分片由最先空出来的线程动态切分**，而不是等所有线程都扫完——这是解决"分片数 ≠ 线程数"负载不均的关键。

**回调机制**：三个 `std::function`（row0pread.h）——`Start`（线程启动时初始化局部状态，如分配排序 buffer）、`F`（每扫一行调一次，COUNT* 是 `counter++`、建索引是构建索引 tuple）、`Finish`（线程收尾，flush 剩余 buffer 到临时文件）。`Thread_ctx` 持有线程局部状态（`m_callback_ctx` + `m_blob_heap`）。

**全局线程上限**：`MAX_THREADS = 256`（106）是**所有连接之和**（`s_active_threads` 静态原子计数，`release_threads` 递减），达到后新连接回退单线程扫描。所以 256 不是单条 SQL 的上限，是实例级并行读总预算。

**明确限制**：类注释 102 行 "Secondary index scans are not supported currently"——**并行扫描只支持聚簇索引**，二级索引扫描回退串行。

### Loader 任务队列：同步原语与终止判定（ddl0loader.cc）

**先把 "quiescence" 这个词讲清楚**（MySQL 里有两个同名但毫不相干的东西，极易混淆）：

- **本篇说的 quiescence ＝ 终止检测**（quiescence detection / distributed termination detection），是分布式与并发算法里的经典问题：判断一个并发计算是否已经彻底结束。经典表述见 Chandy & Lamport 1979 的 termination detection 工作——系统处于**静默（quiescent）**状态的定义是"没有任何进程在执行、也没有消息在传输"；只要再满足一条不变量"**活动只能由活动引发**"，那么静默就意味着不可能再生出新活动，可以安全宣告计算结束。它不需要知道任务总数，也不维护全局引用计数。
- **MySQL 里的另一个 `quiesce` ＝ 表静默**（`row_quiesce_*`，见 `row0quiesce.h` 与 `ha_innodb.cc` 的 `ha_innobase::external_lock` / `FLUSH TABLES ... FOR EXPORT` 路径）：把一张表的修改停下来、脏页刷干净，让 .ibd 处于可安全拷贝的静止状态，服务于物理备份。**它与终止检测毫无关系**，只是英文名撞车。在源码里 grep `quiesce` 命中的绝大多数是这一个。

两者唯一的共同点是字面义"静下来"：一个是"计算没有活动了"，一个是"表没有写入了"。

**DDL 为什么需要终止检测**：Loader 的并行有一个容易被忽略的性质——**任务可以在执行过程中产生新任务**（排序任务跑完会追加该索引的建树任务，见 9.3）。于是"什么时候算全部做完"变成非平凡问题：任务总数事先未知，无法用固定计数，也无法用简单的"队列空"判定（队列空可能只是暂时的，某个正在跑的任务马上会投新的）。这正是终止检测的典型场景，InnoDB 用的就是上面那条静默条件。

InnoDB 的解法不是引用计数，而是靠 **"所有线程同时空闲"** 这个静默条件（quiescence detection）判定终止。源码注释把前提和推论写得非常明确：

```cpp
/** Execute function when there is more than one thread. The general idea
is as follows:
 1. Some initial tasks are added before threads come here to execute tasks.
 2. While executing, a task can generate more tasks. That is the only way
    a task can be added.

 [1] & [2] implies that when all threads are idle, all tasks are completed
 and no more tasks can be added.

 We exit here when all running threads are idle. */
```

多线程序列主循环 `mt_execute()`：

```cpp
do {
  mutex_enter(&m_mutex);

  while (m_tasks.empty()) {
    const auto sig_count = os_event_reset(m_consumer_event);   // ★ 先 reset 取当前 signal 计数

    ++m_n_idle;

    if (m_n_idle >= m_n_threads) {   // ★ 终止条件：所有线程都空闲 ⇒ 不可能再有新任务
      --m_n_threads;                 // 本线程退出，活跃线程数减一
      --m_n_idle;
      mutex_exit(&m_mutex);
      os_event_set(m_consumer_event);  // ★ 必须唤醒别人，否则它们永久阻塞在 wait
      return DB_SUCCESS;
    }

    mutex_exit(&m_mutex);
    os_event_wait_low(m_consumer_event, sig_count);   // ★ 带基线计数等待，防丢失唤醒

    err = m_ctx.get_error();
    if (err != DB_SUCCESS) {         // ★ 别的线程出错 → 自己也退出（共享错误状态）
      mutex_enter(&m_mutex);
      --m_n_threads;
      --m_n_idle;
      mutex_exit(&m_mutex);
      os_event_set(m_consumer_event);
      return err;
    }

    os_event_reset(m_consumer_event);
    mutex_enter(&m_mutex);
    --m_n_idle;
  }

  auto task = m_tasks.front();
  m_tasks.pop_front();

  mutex_exit(&m_mutex);

  err = task();      // ★ 任务在锁外执行：排序/建树是毫秒~秒级，绝不能持锁跑

} while (err == DB_SUCCESS);
```

状态流转：

```
        ┌──────────── 队列非空 ────────────┐
        │                                  │
        ↓                                  │
  [取任务 + m_tasks.pop_front()]           │
        ↓                                  │
  mutex_exit → task()（锁外）─── 产出新任务 ┘
        ↓（队列空）
  ++m_n_idle → os_event_wait_low(event, sig_count)
        ↓
   m_n_idle >= m_n_threads ?
        ├─ 否 → 继续等（被 add_task 的 set 唤醒）
        └─ 是 → --m_n_threads（退出）+ os_event_set（唤醒他人连锁退出）
```

六个必须讲清的设计点：

1. **为什么"所有线程空闲"就能判终止**：任务只能在**某个任务执行期间**产生（注释 [2]）。所以只要还有一个线程在跑任务，它就可能产出新任务；反过来，所有线程都空转在队列上 ⇒ 没有线程在执行 ⇒ 不可能再产生新任务 ⇒ 队列空且永远空 ⇒ 安全终止。这比引用计数简洁，代价是必须严守"加任务只能在任务内"这条不变量——一旦有人在任务外 `add_task`，这个判定就失效。
2. **退出是连锁的**：发现可以退出的线程除了 `--m_n_threads`，还必须 `os_event_set`，因为其他线程正阻塞在 `os_event_wait_low` 上。分母变小后，被唤醒的线程重新检查 `m_n_idle >= m_n_threads` 依然成立，于是依次退出——像多米诺一样收干净。
3. **`os_event_reset` + `os_event_wait_low(sig_count)` 是 InnoDB 的标准防丢唤醒套路**：先 reset 拿到当前 signal 计数作为基线，再以该基线等待；期间若有人 `os_event_set`，计数变化会使等待立即返回。若 reset 与 wait 之间不做这个配合，那期间的 set 会被吃掉，线程永久挂死。
4. **任务在锁外执行**：`mutex_exit` 之后才调 `task()`。否则排序、归并、建树这些重活会串行化整个队列，并行度归零。
5. **错误传播靠共享状态**：`m_ctx.get_error()` 是全局错误位，任一线程出错后，其他线程被唤醒时读到错误即退出——不需要额外的取消标志或中断机制。
6. **单线程退化**：`sync = (m_ctx.m_max_threads <= 1)`，此时走 `st_execute()`，且构造时**不创建锁与事件**（`if (!m_sync) { os_event_create(); mutex_create(...); }`），直接串行消费队列——省掉单线程场景的全部同步开销。

线程构成上还有一个反直觉点：实际参与的任务线程数是 `m_max_threads`，其中 **1 个是用户连接线程自己**，额外 `m_max_threads - 1` 个才是裸 `std::thread`。也就是说调 `innodb_ddl_threads=4` 时，真正新建的线程只有 3 个。

### 使用建议

- **建索引**：`innodb_parallel_read_threads` 与 `innodb_ddl_threads` **设为相同值**（排序任务数由扫描线程数决定，设更大无效）；
- **重建表**：只调 `innodb_ddl_threads`（按二级索引数量设置），`innodb_parallel_read_threads` 不生效；
- 加大扫描线程时同步加大 `innodb_ddl_buffer_size`。

---

## 可观测性

### 系统变量

| 变量名 | 说明 |
|--------|------|
| `innodb_ddl_threads` | 建索引的并行线程数；★ **实际额外新建的是 N-1 个线程**（第 0 个"线程"是用户线程自己），容量规划别按 N 算满 |
| `innodb_ddl_buffer_size` | 并行 DDL 的排序缓冲**总预算**；线程越多每线程分到的越少，会退化成更多趟归并、更多临时文件 I/O |
| `innodb_parallel_read_threads` | 扫描并行度；★ 与 `innodb_ddl_threads` 是**两套独立**的线程数 |
| `innodb_disable_sort_file_cache` | 临时文件是否绕过 OS page cache；绕开可避免双份缓存，但每次读写都真实落盘 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 说明 |
|--------|------|------|
| DDL 处在哪个阶段 | PFS | `performance_schema.events_stages_current`：扫描 / 排序 / 建树各有对应 stage |
| 并行工作线程 | PFS | `thread/innodb/ib_ddl`——★ **`SHOW PROCESSLIST` 看不到**这些线程 |
| 排序临时文件 | 文件系统 / PFS | 临时文件落在 `--tmpdir`（真实名 `ib` + 6 位随机），PFS 里可读名 "Innodb Merge Temp File" |
| 并行是否退化 | 变量 + 源码条件 | 对比 `innodb_ddl_threads` 与实际新建线程数；退化条件见「并行扫描（`Parallel_cursor`）：退化单线程的三个条件」 |

---

## Misc

### 面向二次开发

**扩展点**：新增一种索引类型（如特殊空间索引）需要实现对应的 `Builder` / `Btree_load` 特化，并在 `Loader` 的任务生成处接入。**FTS 与 RTree 是当前仅有的两个"不走标准三期"的特例**（见「为什么 FTS 单独 50 KB：模型根本不同」）——它们是最好的参照样本。

### 坑与已知缺陷

| 坑 | 说明 |
|---|---|
| 线程数含用户线程 | 设 4 只新建 3 个线程；并行度规划时要按 N-1 算 |
| 临时文件吃 `tmpdir` | 排序溢出必写临时文件；`innodb_disable_sort_file_cache` 只影响**是否缓存**，不影响**是否写盘** |
| 并行不是总更快 | 小表、内存预算不足、以及退化单线程的三个条件，都会让并行空转 |
| 建树仍串行 | 源码 `// todo: parallel sort and build`——这是**已知未做的优化**，不是设计限制 |

### 术语对照

| 术语 | 含义 |
|---|---|
| sorted index build / bulk load | 自底向上按序构建 B-tree，与逐条插入相对 |
| quiescence | 本篇指任务队列的"全员空闲即终止"判定；★ 与 InnoDB 的 `row_quiesce_*`（**表静默**，给 `FLUSH TABLES FOR EXPORT` 用）**不是一回事** |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Online DDL*（`innodb_ddl_threads` / `innodb_ddl_buffer_size` / `innodb_parallel_read_threads` / `innodb_disable_sort_file_cache` 的语义基准）

**相关文档**
- 上层：server 侧 DDL 编排、算法选择策略、MDL 三段式、原子 DDL 见 [`ddl.md`](ddl.md)
- 调用本引擎的算法链路（INPLACE 的 MDL 升降级与 row log）见 [`inplace_ddl.md`](inplace_ddl.md)
- 另两条路：COPY 见 [`copy_ddl.md`](copy_ddl.md)、INSTANT 见 [`instant_ddl.md`](instant_ddl.md)

---


---


# MySQL 压缩全景深度解析

> 基于 MySQL 8.0.39 源码。**"压缩"在 MySQL 里不是单一特性，而是 9 个独立机制共用 3 个算法库的合集**。本篇是全景地图：每套机制讲清"压什么、用什么算法、在哪一层、什么时候压/解压"，深度剖析落在各归属文档（互指）。
>
> **边界**：插件体系（GR XCom、X Protocol）与 NDB 不收录（见 [范围判定](#范围判定)）；MariaDB/Percona 的分支特性只做存在性判定、不做源码级剖析（见 [不存在清单](#不存在清单)）。

## 是什么

按压缩对象分四类：

| # | 机制 | 层 | 算法 | 用户入口 | 压缩对象 | 解压时机 | 深挖归属 |
|---|---|---|---|---|---|---|---|
| 1 | page_zip 页压缩 | InnoDB | zlib | `ROW_FORMAT=COMPRESSED` / `KEY_BLOCK_SIZE` | B-tree 页（16K 逻辑压成 8K/4K） | 读盘进 buffer pool 时 | 本篇页压缩节（B-tree 操作的特殊处理见 [`index/btr.md`](../index/btr.md) 压缩页专节） |
| 2 | 透明页压缩 TPC | InnoDB | zlib / LZ4 | `CREATE/ALTER TABLE ... COMPRESSION=` | 数据文件页（压后打洞成稀疏文件） | os 层读盘时 | [`innodb/fil.md`](../innodb/fil.md) TPC 专节 |
| 3 | ZBLOB LOB 页压缩 | InnoDB | zlib | 自动（大字段 ≤128K 走单流） | BLOB/TEXT/SDI 数据页 | 读 LOB 时顺序 inflate | [`innodb/physical/lob.md`](../innodb/physical/lob.md) ZBLOB 专节 |
| 4 | FTS optimize 单词流 | InnoDB | zlib(level 9) | OPTIMIZE 内部自动 | 倒排重编码的内存缓冲 | 用完即弃（不落盘） | [`feat/fts.md`](fts.md) |
| 5 | binlog 事务压缩 | server | zstd | `binlog_transaction_compression` | 单事务的 binlog 事件流 | slave apply / mysqlbinlog | [`server/replication/binlog.md`](../server/replication/binlog.md) 专节 |
| 6 | 连接协议压缩 | server | zlib / zstd | `protocol_compression_algorithms` | 网络包（≥50B 才压） | 收包时 | [`server/auth/authentication.md`](../server/auth/authentication.md) |
| 7 | COMPRESS()/UNCOMPRESS() | server SQL 函数 | zlib | 内建函数 | 用户传入的字符串值 | 调用 UNCOMPRESS() | 本篇（无归属文档） |
| 8 | clone 网络压缩 | clone 插件 | zlib/zstd（复用协议） | `clone_enable_compression` | clone 网络包 | 收包时 | [`server/plugin/clone.md`](../server/plugin/clone.md) |
| 9 | clone 文件重压缩/打洞 | InnoDB clone | zlib/LZ4（复用 TPC） | 表压缩属性自动 | 传输中的压缩页 | recipient 落盘前 | 同上 |

**三个跨机制事实**：

1. `innodb_compression_level`（默认 6）一个变量管 **page_zip + TPC + ZBLOB** 三处（全局 `page_zip_level`，`page0zip.h:60`）；
2. zlib 是全库**唯一**的页级真压缩算法，LZ4 只在 TPC 和 GR 出现，zstd 只在 binlog/协议/clone 网络层出现；
3. **压缩页的"页"仍是 `FIL_PAGE_INDEX`**——压缩是存储形态，不是新页类型（page_zip 是内存双份结构；TPC 页磁盘上类型改为 `FIL_PAGE_COMPRESSED` 但解压后原样恢复）。

## 范围判定：哪些算、哪些不算

| 机制 | 判定 | 理由 |
|---|---|---|
| 以上 9 项 | 收录 | server + InnoDB + clone（InnoDB 数据移动核心链路） |
| GR XCom 消息压缩（`group_replication_compression_threshold`，LZ4） | **排除** | 插件体系（`plugin/group_replication/libmysqlgcs/.../gcs_message_stage_lz4.cc`），用户裁定不管 |
| X Protocol 消息压缩（`mysqlx_compression_algorithms`，deflate/lz4/zstd 三选） | **排除** | 插件体系（`plugin/x/.../protocol_flusher_compression.cc`） |
| NDB binlog 压缩（`--ndb-log-transaction-compression`） | **排除** | NDB 集群（复用 #5 的 zstd 路径） |
| mysqlpump `--compress-output=`（LZ4F/zlib） | **排除** | 客户端工具，非 server 运行时机制 |
| myisampack 表打包（Huffman） | **排除** | MyISAM 离线只读工具 |
| 字段/记录压缩、redo/undo 压缩等 | **不存在** | 见 [不存在清单](#不存在清单) |

## 三个算法库的接入

| 库 | bundled 版本 | 构建选项 | 压缩特性 | 全库接入点 |
|---|---|---|---|---|
| zlib | 1.2.13 | `WITH_ZLIB`（必选） | deflate 流（windowBits=15/memLevel 可调） | page_zip（`deflateInit2` page0zip.cc:1060）、TPC（`compress2` os0file.cc:1387）、ZBLOB（`deflateInit2` zlob0ins.cc:282）、FTS（`deflateInit` fts0opt.cc:824）、协议压缩（`compress2` my_compress.cc:215）、COMPRESS()（`compress` item_strfunc.cc:4114） |
| zstd | 1.5.5 | `WITH_ZSTD` | 流式（CStream 一套）/ 单次（CCtx） | binlog（`ZSTD_createCStream` zstd_comp.cpp:94）、协议压缩（`ZSTD_compressCCtx` my_compress.cc:124） |
| lz4 | 1.9.4 | `WITH_LZ4` | block 单次（无 level 参数） | TPC（`LZ4_compress_default` os0file.cc:1402）、GR XCom（排除）、X Protocol（排除） |

三库源码自带于 `extra/`，均可切 `system`。

## 页压缩：page_zip（`ROW_FORMAT=COMPRESSED`）

**压什么**：每个 B-tree 页（聚簇 + 二级 + node pointer 页），16K 逻辑页经 zlib 压成 8K/4K（`KEY_BLOCK_SIZE`，物理上塞在同一个 16K 槽位里）。**内存双份**：buffer pool 里 frame（未压缩，所有操作在上面做）+ `page_zip`（压缩镜像 + 修改日志）——磁盘一份压缩镜像，内存反而更贵（16K+8K）。**读盘解压**：`buf_zip_decompress` → `page_zip_decompress`（`buf0buf.cc:3409`）。B-tree 操作对压缩页的特殊处理（插入 NULL/二次分裂/合并判定）→ [`index/btr.md`](../index/btr.md) 压缩页专节。

### 双份结构：frame 是唯一操作对象，page_zip 只承载压缩镜像 + 修改日志

`page_zip_des_t`（`page0types.h:200-224`）只有 6 个成员，**没有 filename**：

```c
struct page_zip_des_t {
  page_zip_t *data;     // 压缩镜像（指向 buf 块的压缩缓冲区）
#ifdef UNIV_DEBUG
  uint16_t m_start;     // 修改日志起始偏移（仅 DEBUG 版存在）
  bool m_external;
#endif
  uint16_t m_end;       // 修改日志结束偏移
  uint16_t n_blobs;     // 外部存储列数（16KiB 页最大 744）
  bool m_nonempty;      // 修改日志非空
  uint8_t ssize;        // 压缩页移位大小：(UNIV_ZIP_SIZE_MIN >> 1) << ssize
};
```

内嵌于 `buf_block_t::page.zip`。压缩镜像的物理布局（`page0zip.ic:40-105` 权威注释）：未压缩页头 `PAGE_DATA` 字节原样复制 → 压缩索引字段信息 → 压缩数据流 → **修改日志**（`m_start..m_end`）→ 空闲零区 → BLOB 指针区 → 每记录不压缩列（`trx_id+roll_ptr` / node_ptr）→ **dense 目录**（自页尾向前，每用户记录 2B：`PAGE_ZIP_DIR_SLOT_MASK=0x3fff` / `OWNED=0x4000` / `DEL=0x8000`）。

**同步模型**：B-tree 遍历/比较**全在 frame 上**（`page_cur_search_with_match` 无 zip 分支——压缩页进 buffer pool 时已由 `buf_zip_decompress` 解压）；page_zip 通过四条通道同步：

| 操作 | 通道 | 行为 |
|---|---|---|
| 页头区（<PAGE_DATA） | `page_zip_write_header` | 直接 memcpy 到镜像同偏移 |
| 整条记录写 | `page_zip_write_rec`（`page0zip.cc:1659`） | 追加修改日志，`m_end` 前移 |
| 删除 | `page_mem_free` → `page_zip_dir_delete` | 往修改日志追加 clear 记录，**不产生可复用空洞** |
| 整页重压 | `page_zip_compress`（`page0zip.cc:921`） | 成功：`m_end = PAGE_DATA + total_out`；失败返回 false，**page_zip 保持原状**（叶子页调 `dict_index_zip_failure` 计入 zip_pad） |

**空间语义**：压缩页的空闲账本不是 `PAGE_GARBAGE`，而是 `page_zip_available` / `page_zip_max_ins_size`（`page0zip.ic:190-234`）：剩余 = 页大小 − `m_end` − trailer（dense 目录 + 每记录不压缩列 + BLOB 指针 + 新目录槽）。`page_zip_empty_size`（`page0zip.cc:110`）= 空压缩页**最坏情况保证容量**（页头 + `compressBound` 最坏压缩率都扣掉）——SMO 预测与分裂点选择全用它。

### zip_pad 自适应闭环：压缩失败率驱动"预期可压缩页大小"

压缩页有独立于 B-tree 逻辑的**填充率自适应**：每个索引持 `zip_pad_info_t`（`dict0mem.h:980-988`）——`mutex` + `std::atomic<ulint> pad`（当前 padding 字节数）+ `success`/`failure`（本 round 计数）+ `n_rounds`（连续成功轮数）。

**计数点**：每次页压缩结束调 `dict_index_zip_success`（`dict0dict.cc:4907`，++success）或 `dict_index_zip_failure`（:4927，++failure）；两者都汇入 `dict_index_zip_pad_update`（:4855-4895）做回合判定：

```cpp
    if (info->failure >= ((info->success + info->failure) * zip_threshold) / 100) {
      /* Only do increment if it won't increase padding beyond max pad size. */
      if (info->pad + ZIP_PAD_INCR < (UNIV_PAGE_SIZE * zip_pad_max) / 100) {
        info->pad.fetch_add(ZIP_PAD_INCR);      // +128B（原子读保证无锁读一致）
        MONITOR_INC(MONITOR_PAD_INCREMENTS);
      }
      info->n_rounds = 0;
    } else {
      ++info->n_rounds;
      /* If enough successful rounds are completed with compression failure
      rate in control, decrease the padding. */
      if (info->n_rounds >= ZIP_PAD_SUCCESSFUL_ROUND_LIMIT && info->pad > 0) {
        info->pad.fetch_sub(ZIP_PAD_INCR);      // 连续 5 轮达标才减，防抖动
        info->n_rounds = 0;
        MONITOR_INC(MONITOR_PAD_DECREMENTS);
      }
    }
```

参数：`ZIP_PAD_INCR = 128`（dict0mem.h:967）、`ZIP_PAD_SUCCESSFUL_ROUND_LIMIT = 5`（:964）、阈值 `zip_failure_threshold_pct`（=`innodb_compression_failure_threshold_pct`，默认 5%）、上限 `zip_pad_max`（=`innodb_compression_pad_pct_max`，默认 50%）。**加急减缓**：失败一轮立即 +128B，连减要 5 轮无失败——避免抖动。两个变量置 0 即整体禁用（`dict_index_zip_success/failure` 开头 "Disabled by user" 提前返回，`optimal_page_size` 直接返回 `UNIV_PAGE_SIZE`）。

**消费点 `dict_index_zip_pad_optimal_page_size`**（:4942-4960）：返回 `max(UNIV_PAGE_SIZE - pad, (100 - zip_pad_max)%)`——"预期能压缩成功的页大小上限"，四处使用：① 乐观插入放弃判定（`btr0cur.cc:2916-2922`）；② SMO 预测 `btr_cur_will_modify_tree`（:580-585）用的是更保守的 `page_zip_empty_size`（最坏压缩率），**不是** pad；③ merge 拒绝闸 `btr_can_merge_with_page`（`btr0btr.cc:4625-4632`）；④ bulk load 的 `Page_load` padding（`btr0load.cc:405-406`）。**闭环**：压缩失败率 ↑ → pad ↑ → optimal 变小 → 乐观插入/合并更早放弃、提前分裂 → 页内数据量变小 → 压缩成功率 ↑ → 失败率回落 → 5 轮后 pad 回落。

### redo 差异：record 级全适用 + 5 个 zip 专属类型

- **record 级 MLOG 全部适用于压缩页**：`recv_parse_or_apply_log_rec_body` 在 applying 时取 `page_zip = buf_block_get_page_zip(block)`（`log0recv.cc:1705-1723`），所有解析函数带 `page_zip` 参数**双写**（frame + 镜像）；
- zip 专属 5 类（`log0recv.cc:2301-2352`）：`MLOG_ZIP_WRITE_NODE_PTR` / `WRITE_BLOB_PTR` / `WRITE_HEADER` / `PAGE_COMPRESS` / `PAGE_COMPRESS_NO_DATA`；
- `MLOG_ZIP_PAGE_REORGANIZE`（=73）payload 是 **1 字节压缩级别**——经 `mlog_open_and_write_index(mtr, page, index, type, page_zip ? 1 : 0, log_ptr)` 预留 1 字节再 `mach_write_to_1` 写入（`btr0btr.cc:1296-1320`，**不是** `mlog_write_ulint`）；
- `MLOG_PAGE_CREATE/COMP_PAGE_CREATE` 对压缩页**不适用**（`ut_a(!page_zip)`，`log0recv.cc:2156`）——压缩页建页走 `MLOG_ZIP_PAGE_COMPRESS` 全镜像（防 zlib 版本差异导致恢复不一致）；
- 恢复时 `log0recv.cc` 内**没有** `page_zip_decompress` 调用——解压在 buffer pool IO 层（`buf_zip_decompress`），redo 解析始终在已解压的 frame + page_zip 双份上同步。

**现状定位**：`ROW_FORMAT=COMPRESSED`（KEY_BLOCK_SIZE）在 8.0.39 仍是完整一等路径，与 transparent page compression（`COMPRESSION=`，页打洞，**不产生** `page_zip_des_t`）是两套独立机制。**bulk load 支持压缩页**（`Btree_load`/`Page_load` 有完整 zip 路径：`page_create_zip` 建页、`compress()` 整页压、失败 `page_split` 再切、padding 用 `dict_index_zip_pad_optimal_page_size`）。

## 透明页压缩：TPC（`COMPRESSION=`）

**压什么**：file-per-table 数据文件的每个页（`page_no>0`），只压 `FIL_PAGE_DATA(38)` 之后的页体，38B 页头保留并改写；压缩长度对齐 FS block 后**打洞**（`fallocate(FALLOC_FL_PUNCH_HOLE|FALLOC_FL_KEEP_SIZE)`）释放尾部空间 → 稀疏文件。**关键不变量**：压缩/解压只发生在 os 文件 I/O 边界，**buffer pool 里永远是未压缩页**——上层零感知、redo 格式零改动（IO 层注入点见 [`innodb/fil.md`](../innodb/fil.md) 主链路节）。

### 压缩页头布局与 `struct Compression`

`FIL_PAGE_TYPE` 改为 `FIL_PAGE_COMPRESSED`，元数据借用 `FIL_PAGE_FILE_FLUSH_LSN(26)` 起的 8 字节（`fil0types.h:78-121`）：

```cpp
constexpr uint32_t FIL_PAGE_VERSION = FIL_PAGE_FILE_FLUSH_LSN;             // 26, u8
constexpr uint32_t FIL_PAGE_ALGORITHM_V1 = FIL_PAGE_VERSION + 1;           // 27, u8
constexpr uint32_t FIL_PAGE_ORIGINAL_TYPE_V1 = FIL_PAGE_ALGORITHM_V1 + 1;  // 28, u16
constexpr uint32_t FIL_PAGE_ORIGINAL_SIZE_V1 = FIL_PAGE_ORIGINAL_TYPE_V1 + 2; // 30, u16
constexpr uint32_t FIL_PAGE_COMPRESS_SIZE_V1 = FIL_PAGE_ORIGINAL_SIZE_V1 + 2; // 32, u16
```

`struct Compression`（`os/file.h:51-66`）：`enum Type { NONE=0, ZLIB=1, LZ4=2 }`——**无 zstd**；枚举注释原文（恢复期无压缩类型信息，按不压缩回写）解释了恢复行为。`is_compressed_page` = `FIL_PAGE_TYPE == FIL_PAGE_COMPRESSED`。

### 压缩本体 `os_file_compress_page`（os0file.cc:1337-1457）

前置排除：R-Tree 页（复用这 8 字节存 split sequence number）、`block_size==UNDEFINED`、`NONE`、页 < 2 个 FS block——直接整页原样返回。核心：

```cpp
  ulint out_len = src_len - (FIL_PAGE_DATA + block_size);   // 必须省出至少 1 个 FS block
  ulint content_len = src_len - FIL_PAGE_DATA;
  switch (compression.m_type) {
    case Compression::ZLIB: {
      uLongf zlen = out_len;
      if (compress2(dst + FIL_PAGE_DATA, &zlen, src + FIL_PAGE_DATA,
                    content_len, page_zip_level) != Z_OK) {
        *dst_len = src_len; return (src);                    // 失败 → 整页不压缩
      }
      len = zlen; break;
    }
    case Compression::LZ4:
      len = LZ4_compress_default(src + FIL_PAGE_DATA, dst + FIL_PAGE_DATA,
                                 content_len, out_len);
      if (len == 0 || len >= out_len) { *dst_len = src_len; return (src); } // 收益不足
      break;
  }
```

成功后：页头原样 `memmove` → 写 6 字段元数据（`FIL_PAGE_VERSION_2`/算法/原类型/原长度/压后长度）→ `*dst_len = ut_calc_align(len + FIL_PAGE_DATA, block_size)` 对齐 FS block → 尾部清零（防信息泄漏）。**压失败/收益不足一律整页不压缩、写盘后不打洞**——压缩是尽力而为。

### 写路径：决策 → 分流 → 打洞

- **决策**（`Fil_shard::do_io`，fil0fil.cc:7853）：写 + 非 zip 页 + `page_no>0` + 打洞可用 + `file->punch_hole` → 注入 `space->compression_type`；否则 `clear_compressed`；
- **包装重载 `os_file_compress_page(IORequest,...)`**（os0file.cc:1887-1960）：

```cpp
file::Block *os_file_compress_page(IORequest &type, void *&buf, ulint *n) {
  ulint n_alloc = *n * 2;                    // 最坏情况缓冲（LZ4 上界校验）
  auto block = os_alloc_block();
  ulint compressed_len = *n;
  ulint old_compressed_len = mach_read_from_2((byte *)buf + FIL_PAGE_COMPRESS_SIZE_V1);
  if (old_compressed_len > 0)
    old_compressed_len = ut_calc_align(old_compressed_len + FIL_PAGE_DATA, type.block_size());
  else
    old_compressed_len = *n;                 // 页内旧压缩长度（可能来自恢复）
  byte *compressed_page = (byte *)ut_align(block->m_ptr, os_io_ptr_align);
  byte *buf_ptr = os_file_compress_page(type.compression_algorithm(), type.block_size(),
      (byte *)buf, *n, compressed_page, &compressed_len);
  if (buf_ptr != buf) {                      // 真的压缩了（压不动返回原页）
    memcpy((byte *)buf + FIL_PAGE_COMPRESS_SIZE_V1, buf_ptr + FIL_PAGE_COMPRESS_SIZE_V1, 2);
    buf = buf_ptr; *n = compressed_len;      // 把新压缩长度回写原页头
    if (compressed_len >= old_compressed_len && !type.is_punch_hole_optimisation_disabled()) {
      type.clear_punch_hole();               // 没比旧页更小 → 本次不打洞
    }
  }
  return (block);
}
```

压缩结果放独立 `file::Block`（不破坏原 buf）；"页边扩大"场景清 `PUNCH_HOLE` 只做完整写——避免反复打洞。**同步写**（`os_file_io` :4820）在提交点调它；**AIO 写**（`os_aio_func` :6582）提交时压缩、`slot->len` 缩小为压缩长度。

- **打洞**（`os_file_io_complete` :1675-1703）：`punch_hole` 且确实压缩（`len<src_len`）→ `os_file_punch_hole(fh, offset+len, src_len-len)`；POSIX 实现 `fallocate(FALLOC_FL_PUNCH_HOLE|FALLOC_FL_KEEP_SIZE)`（:2062），`ENOTSUP` → `DB_IO_NO_PUNCH_HOLE` → `fil_no_punch_hole`（fil0fil.cc:8861）**永久关闭**该文件的打洞；AIO 完成路径打洞条件含 `!slot->type.is_dblwr()`——双写文件是裸页拷贝不打洞。

### 读路径：os 层解压（transparent 的关键）

读满整页后（`os_file_read_page` :5067），若 `Compression::is_compressed_page` 为真，交给 `os_file_io_complete` 读分支（:1664-1674）：**先解密、再解压**（与写侧"先压缩后加密"对称）。`Compression::deserialize`（os/file.cc:136-265，核心段）：

```cpp
dberr_t Compression::deserialize(bool dblwr_read, byte *src, byte *dst, ulint dst_len) {
  if (!is_compressed_page(src)) return (DB_SUCCESS);      // 未压缩页原样
  meta_t header;
  deserialize_header(src, &header);                       // 读 6 字段元数据
  byte *ptr = src + FIL_PAGE_DATA;
  /* 校验页版本 / original_size 范围（UNIV_PAGE_SIZE_MIN..MAX）*/
  if (!is_valid_page_version(header.m_version) ||
      header.m_original_size < UNIV_PAGE_SIZE_MIN - (FIL_PAGE_DATA + 8) ||
      header.m_original_size > UNIV_PAGE_SIZE_MAX - FIL_PAGE_DATA) {
    return DB_CORRUPTION;
  }
  if (dst == nullptr) {   // 调用方不知道期望大小：按 original_size + 50% 余量分配
    ulint n_bytes = header.m_original_size + (header.m_original_size / 2);
    dst = ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, n_bytes);
    allocated = true;
  }
  switch (static_cast<Compression::Type>(header.m_algorithm)) {
    case Compression::ZLIB: {
      uLongf zlen = header.m_original_size;
      if (uncompress(dst, &zlen, ptr, header.m_compressed_size) != Z_OK) {
        return (DB_IO_DECOMPRESS_FAIL);
      }
      len = zlen; break;
    }
    case Compression::LZ4:
      if (dblwr_read) {
        ret = LZ4_decompress_safe(ptr, dst, header.m_compressed_size, header.m_original_size);
      } else {
        /* LZ4_decompress_fast is a little faster but can potentially read
        beyond the input buffer if the data is malformed. When recovering
        from the double write buffer we can afford to use the slower
        function above. */
        ret = LZ4_decompress_fast(ptr, dst, header.m_original_size);
      }
      if (ret < 0) return (DB_IO_DECOMPRESS_FAIL);
      break;
    default:
      ib::error(ER_IB_MSG_741) << "Compression algorithm support missing: ...";
      return (DB_UNSUPPORTED);
  }
  /* Leave the header alone：解压数据 memmove 回原位（覆盖压缩数据）*/
  memmove(src + FIL_PAGE_DATA, dst, len);
  mach_write_to_2(src + FIL_PAGE_TYPE, header.m_original_type);  // 还原原始页类型
  if (allocated) ut::free(dst);
  return (DB_SUCCESS);
}
```

关键点：**LZ4 双函数**——常规读用 `LZ4_decompress_fast`（快但可能越界读），dblwr 恢复路径用 `LZ4_decompress_safe`（注释原文，安全优先）；解压数据就地 `memmove` 覆盖压缩字节；页头除 `FIL_PAGE_TYPE` 外保持压缩时拷贝的原始头。上层拿到的永远是未压缩页。

### 与 dblwr / 恢复的交互

- dblwr 写帧：`dblwr::get_encrypted_frame`（buf0dblwr.cc:2413）先 `os_file_compress_page` 再加密（注释原文 "Compression needs to be done before encryption"）——**dblwr 文件存的是压缩（可能加密）后的帧**；
- dblwr 恢复：`recv::Pages::check`（:2948）按 `FIL_PAGE_COMPRESSED_AND_ENCRYPTED` / `FIL_PAGE_COMPRESSED` 两分支，先解密再 `os_file_decompress_page(true, ...)`；
- redo 恢复回写：走 `fil_io → do_io` 同一路径重新压缩+打洞；**恢复早期 DD 未读、算法未知 → `compression_type==NONE` 整页不压缩**（os/file.h:54-56 注释），fil0fil.cc:2630-2638 读表空间头时顺带从 1 号页压缩头反推算法恢复能力（0 号页永不压缩，1 号页元数据足以反推）。

### 属性落地与拒绝清单

`fil_set_compression`（fil0fil.cc:8863）：`Compression::check`（"zlib"/"lz4" 字符串校验）→ 拒绝共享/系统/临时/zip 表空间（`DB_IO_NO_PUNCH_HOLE_TABLESPACE`）→ 写 `space->compression_type` → 文件系统不支持打洞返回 `DB_IO_NO_PUNCH_HOLE_FS`。DDL 侧 `create_option_compression_is_valid`（ha_innodb.cc:12630）额外拒绝：与 `KEY_BLOCK_SIZE`/`ROW_FORMAT=COMPRESSED` 同用、临时表、general/系统表空间。

## LOB 页压缩：ZBLOB

**压什么**：大字段（BLOB/TEXT/SDI）≤ 128K（`Z_CHUNK_SIZE`）时走**单 zlib 流**：一条 LOB 压成一段**跨页连续的 zlib 流**，铺满 `FIL_PAGE_TYPE_ZBLOB(11)`（首页）/`ZBLOB2(12)`/`FIL_PAGE_SDI_ZBLOB(19)` 页链（`fil0fil.h:1248-1277`）。>128K 走 `lob::z_insert` 分块格式（每 128K 一个独立流 + LOB 索引）。LOB 结构与页类型体系见 [`innodb/physical/lob.md`](../innodb/physical/lob.md)。

**触发阈值 `Z_CHUNK_SIZE = 128K`**（`lob0lob.h:86-91`）：`ref_t::use_single_z_stream(lob_len)` = `length() <= 128K`。调用点 `btr_store_big_rec_extern_fields`（`lob0lob.cc:514-525`）：≤128K 走 `zInserter` 单流；该单流分支只在 `page_zip != nullptr`（COMPRESSED 行格式）或 SDI 索引时进入。

### 写入 `zInserter`（zlob0ins.cc）

`prepare()`（:266-289）完整：

```cpp
dberr_t zInserter::prepare() {
  /* Zlib deflate needs 128 kilobytes for the default window size, plus
  512 << memLevel, plus a few kilobytes for small objects.  We use reduced
  memLevel to limit the memory consumption, and preallocate the heap,
  hoping to avoid memory fragmentation. */
  m_heap = mem_heap_create(250000, UT_LOCATION_HERE);
  if (m_heap == nullptr) return (DB_OUT_OF_MEMORY);

  page_zip_set_alloc(&m_stream, m_heap);
  int ret = deflateInit2(&m_stream, page_zip_level, Z_DEFLATED, 15, 7,
                         Z_DEFAULT_STRATEGY);
  if (ret != Z_OK) return (DB_FAIL);
  return (DB_SUCCESS);
}
```

级别 `page_zip_level`（与 page_zip/TPC 共用 `innodb_compression_level`）、windowBits=15、memLevel=7（降内存）、zlib 的 malloc 全部重定向到 250KB 的 InnoDB mem_heap。

逐页写 `write_into_single_page`（:161-231）核心：

```cpp
int zInserter::write_into_single_page() {
  const uint in_before = m_stream.avail_in;
  page_t *blob_page = buf_block_get_frame(m_cur_blob_block);

  /* Space available in compressed page to carry blob data */
  const uint payload_size_zip = page_size.physical() - FIL_PAGE_DATA;
  m_stream.next_out = blob_page + FIL_PAGE_DATA;      // 输出直指页体
  m_stream.avail_out = payload_size_zip;
  int err = deflate(&m_stream, Z_FINISH);             // 一次填满一页
  ut_a(err == Z_OK || err == Z_STREAM_END);
  ut_a(err == Z_STREAM_END || m_stream.avail_out == 0);

  const blob_page_info_t page_info(m_cur_blob_page_no,
                                   in_before - m_stream.avail_in,       // 本页吃掉多少输入
                                   payload_size_zip - m_stream.avail_out); // 本页写出多少
  add_to_blob_dir(page_info);

  mlog_write_ulint(blob_page + FIL_PAGE_NEXT, FIL_NULL, MLOG_4BYTES, mtr);
  mlog_write_ulint(blob_page + FIL_PAGE_PREV, FIL_NULL, MLOG_4BYTES, mtr);
  /* Write a back pointer to the record into the otherwise unused area. */
  mlog_write_ulint(blob_page + FIL_PAGE_FILE_FLUSH_LSN, m_ctx->space(), MLOG_4BYTES, mtr);
  mlog_write_ulint(blob_page + FIL_PAGE_FILE_FLUSH_LSN + 4,
                   m_ctx->get_page_no(), MLOG_4BYTES, mtr);
  if (m_stream.avail_out > 0) {
    /* Zero out the unused part of the page. */           // 剩余清零保证 redo 可复现
    memset(blob_page + page_zip_get_size(...) - m_stream.avail_out, 0, m_stream.avail_out);
  }
  mlog_log_string(blob_page + FIL_PAGE_FILE_FLUSH_LSN, ...);   // 整页 redo
  memcpy(blob_page_zip->data, blob_page, page_zip_get_size(page_zip)); // frame → zip 镜像
  return (err);
}
```

`deflate(Z_FINISH)` 一页填满返回 `Z_OK`（写下一页继续）、流结束返回 `Z_STREAM_END`（`update_length_in_blobref` 把总长写回行内 extern ref）；每 4 页 `m_ctx->check_redolog()` 释放重拿 B-tree 锁（跨 mtr 写大 LOB）。页布局：ZBLOB 页从 `FIL_PAGE_DATA(38)` 铺满压缩字节流到页尾（剩余清零）、**无页内目录/分片元数据**（长度、首页码都在行内 ref 里）。

### 读取 `zReader`（lob0lob.cc:157-275）

`setup_zstream`（:157-174）：`inflateInit` + 40KB mem_heap。`fetch()` 主循环（节选 :189-234）：

```cpp
  setup_zstream();
  m_remaining = m_rctx.m_blobref.length();
  while (m_rctx.m_page_no != FIL_NULL) {
    err = fetch_page();
    m_stream.next_in = m_bpage->zip.data + m_rctx.m_offset;
    m_stream.avail_in = m_rctx.m_page_size.physical() - m_rctx.m_offset;

    int zlib_err = inflate(&m_stream, Z_NO_FLUSH);
    switch (zlib_err) {
      case Z_OK:
        if (m_stream.avail_out == 0) goto end_of_blob;   // 输出缓冲已满：够了
        break;
      case Z_STREAM_END:
        if (m_rctx.m_page_no == FIL_NULL) goto end_of_blob;   // 恰好覆盖整个页链
        [[fallthrough]];
      default:
        err = DB_FAIL;
        ib::error(ER_IB_MSG_630) << "inflate() of compressed BLOB page ... returned "
                                 << zlib_err << " (" << m_stream.msg << ")";
        ut_error;
    }
    buf_page_release_zip(m_bpage);
    m_rctx.m_offset = FIL_PAGE_NEXT;                     // 后续页从 FIL_PAGE_DATA 起喂流
  }
end_of_blob:
  buf_page_release_zip(m_bpage);
  inflateEnd(&m_stream);
  mem_heap_free(m_heap);
```

**必须从头顺序解压，无法随机访问中间页**：`z_stream m_stream` 是 `zReader` 成员（`lob0lob.h:1337`），跨页持续 `inflate`；`Z_STREAM_END` 出现在非末页（页链尚未走完）即 `ut_error`——流必须恰好覆盖整个页链。入口 `zlob0read.cc:85-92` 按首帧类型分派：`ZBLOB`/`SDI_ZBLOB` → `zReader`，`ZLOB_FIRST` → 分块格式。

**限制**：单流 LOB **不支持 partial update**（`lob0lob.cc:1212-1220`，`use_single_z_stream()` 直接禁用 `lob::z_update`）。

## FTS optimize 单词流

zlib 只出现在 `fts0opt.cc`（`deflateInit(level=9)`，`:824`），是 OPTIMIZE 流程里**纯内存**的 word 列表缓冲（1024B block 链表），用完即释放——**不落盘、无开关**。把"InnoDB FTS 用 zlib 压缩倒排索引"写进文档是错的（真正的倒排压缩是 ilist 的 delta 编码）→ [`feat/fts.md`](fts.md)、[`index/inverted.md`](../index/inverted.md)。

## binlog 事务压缩（zstd）

**压什么**：以**事务为单位**，一个事务的所有事件打包成 `Transaction_payload_log_event`（type 40，payload 是原始事件流的 zstd 压缩结果）。压缩发生在 BGC 流水线的 finalize cache 阶段（`Binlog_cache_compressor`，`binlog.cc:2055`）；`shall_compress` 四排除（incident/非事务表/SBR）；GTID event 单独写出不受压。**解压**：slave `do_apply_event` + `Decompressing_event_object_istream` 逐事件解出、mysqlbinlog 同款。**位置语义不变**：按事务回放、按 GTID 定位不受影响。P_S 表 `binary_log_transaction_compression_stats` 统计压缩率。

事件格式（TLV 编码、post-header=0）→ [`server/replication/binlog.md`](../server/replication/binlog.md) 专节。

## 连接协议压缩（zlib/zstd）

**压什么**：连接层网络包（≥ `MIN_COMPRESS_LENGTH=50` 字节才压）。**协商**：服务端握手按 `protocol_compression_algorithms` 设能力位（`CLIENT_COMPRESS` / `CLIENT_ZSTD_COMPRESSION_ALGORITHM`），zlib 优先、zstd 由客户端带 level（1-22）——详见 [`server/auth/authentication.md`](../server/auth/authentication.md)。**实现**：每连接 `mysql_compress_context`（algorithm + union{zlib level / zstd cctx+dctx+level}）→ `my_compress`/`my_uncompress` 按算法分派；压缩包 7 字节头（4B 包长 + 1B 序号 + 3B 未压缩长度，**0 表示未压缩**）；压缩反而变大就整包原样发。默认 level：zlib=6、zstd=3。复制连接：`replica_compressed_protocol`（弃用别名 `slave_compressed_protocol`）强制 zlib，新版走 `CHANGE REPLICATION SOURCE ... COMPRESSION_ALGORITHM`。

实现细节（my_compress.cc 贴码、包格式）→ [`server/auth/authentication.md`](../server/auth/authentication.md)。

## COMPRESS()/UNCOMPRESS()（SQL 内建函数）

```sql
SELECT COMPRESS(any_string);    -- zlib compress()，前 4 字节存原始长度
SELECT UNCOMPRESS(x);           -- 解压
SELECT UNCOMPRESSED_LENGTH(x);  -- 只读前 4 字节
```

`Item_func_compress::val_str`（`item_strfunc.cc:4088-4134`）核心：

```cpp
String *Item_func_compress::val_str(String *str) {
  ...
  String *res = args[0]->val_str(str);
  ...
  ulong newlen = compressBound(res->length()) + 4;   // zlib 最坏上界 + 4 字节长度头
  String *str2 = new (mem_root) String();
  str2->alloc(newlen);
  // ★ 尾空格处理：COMPRESS 会吞掉尾随空格（uncompress 无法恢复），
  //   为了能完整恢复，末尾额外存一个 '.' 标记
  ...
  unsigned long tmp = (unsigned long)str2->alloced_length();
  int err = compress((Bytef *)str2->ptr() + 4, &tmp,
                     (const Bytef *)res->ptr(), res->length());
  ...
  int4store(str2->ptr(), res->length());   // ★ 前 4 字节 = 原始长度
  str2->length(tmp + 4);
  ...
}
```

- 结果格式：`[4B 原始长度][zlib 压缩数据]`——`UNCOMPRESSED_LENGTH()` 直接读前 4 字节，`UNCOMPRESS()` 用它分配解压缓冲；
- `compressBound` 预分配最坏上界，`compress()` 实际压缩；
- **尾空格陷阱**：MySQL 的 `Field_string` 比较会忽略尾空格，压缩结果存入 CHAR 列再取出会丢字节——源码注释记录了用额外 '.' 标记规避（这是历史兼容行为，MySQL 手册明确 COMPRESS/UNCOMPRESS 结果**必须存 BLOB/TEXT**）。

**注意**：它是**函数不是存储特性**——结果就是普通字符串值（存进 BLOB 列），索引/比较都对"压缩后的字节"生效，与"字段压缩"（不存在，见不存在清单）是两回事。

## clone 的两个独立压缩（InnoDB 数据移动）

1. **网络压缩**：`clone_enable_compression`（默认 false）——clone 客户端把自带 `NET_SERVER`（含 `compress_ctx`）挂进连接，**复用经典协议压缩**（zlib/zstd）；
2. **文件重压缩/打洞**：表启用 TPC 时，donor 用 `os_file_compress_page` 把整页重压缩后传出（磁盘稀疏空洞无法经网络传输），recipient 写压缩字节 + `os_file_punch_hole` 还原稀疏文件——与网络压缩**正交叠加**（内层页压缩、外层网络压缩）。

实现细节 → [`server/plugin/clone.md`](../server/plugin/clone.md)。

## 变量与监控总表

| 变量 | 默认 | 作用机制 |
|---|---|---|
| `innodb_compression_level` | 6（0-9） | page_zip + TPC(zlib) + ZBLOB 共用 |
| `innodb_compression_failure_threshold_pct` | 5（0-100） | zip_pad 自适应触发阈值 |
| `innodb_compression_pad_pct_max` | 50（0-75） | 压缩页 padding 上限 |
| `innodb_log_compressed_pages` | ON | page_zip 整页镜像是否进 redo（**不是 redo 压缩**） |
| `innodb_cmp_per_index_enabled` | OFF | 启用 `I_S.INNODB_CMP_PER_INDEX` |
| `binlog_transaction_compression` | OFF | binlog zstd 压缩开关 |
| `binlog_transaction_compression_level_zstd` | 3（1-22） | binlog zstd 级别 |
| `protocol_compression_algorithms` | zlib,zstd,uncompressed | 协议压缩算法协商 |
| `clone_enable_compression` | false | clone 网络压缩 |

监控：`MONITOR_PAGE_COMPRESS/DECOMPRESS/PAD_INCREMENTS/PAD_DECREMENTS`（`compression` 模块，page_zip 计数点 page0zip.cc:1038/1302）；**TPC 无专用计数器**；binlog 压缩有 P_S `binary_log_transaction_compression_stats`。

## 不存在清单（MariaDB/Percona 分支特性，非源码级剖析）

| 特性 | 来源 | Oracle MySQL 8.0.39 证据 |
|---|---|---|
| 字段/列级压缩（`innodb_compressed_columns`） | Percona | 无变量（sys_vars.cc 零命中）、无语法（sql_yacc.yy 的 COMPRESSED_SYM 仅 ROW_FORMAT）、无 handler 列压缩 API |
| 记录压缩（column compression） | MariaDB | 同上；唯一命中是 myisampack 离线工具 |
| redo log 压缩 | Percona/MariaDB | log 目录 grep compress 仅 MLOG_ZIP_* 重放与变长整数编码 |
| undo 压缩 | — | trx0undo.cc 的 "compress" 全是 `mlog_catenate_ulint_compressed` 变长编码 |
| 压缩 dblwr | Percona | 8.0.39 的 Reduced_double_write 只存 `(space,page,LSN)` 条目 |
| 临时表压缩 | — | `ER_UNSUPPORT_COMPRESSED_TEMPORARY_TABLE` 直接报错 |
| `CREATE TABLESPACE ... COMPRESSION=` | — | 语法只在 CREATE/ALTER TABLE（sql_yacc.yy:6758），tablespace_option 无此规则 |
| zstd 页压缩 / `innodb_page_compression` | Percona | 无（zstd 只出现在 binlog/协议/clone 网络层） |

## 常见误解

1. **"压缩省内存"**：page_zip 在 buffer pool 里 frame + zip 双份，比普通页更费内存；TPC 才是内存单份（但省磁盘靠打洞）。
2. **`innodb_log_compressed_pages` 不是 redo 压缩**：它决定 page_zip 的整页镜像要不要写进 redo，MySQL 官方没有 redo 压缩。
3. **`COMPRESSION=` 与 `KEY_BLOCK_SIZE` 是互斥的两套**：前者打洞（页不真小）、后者真压缩（页 16K→8K/4K），`create_option_compression_is_valid` 明确拒绝同用。
4. **`COMPRESS()` 函数 ≠ 字段压缩**：函数结果存进列是"压缩过的数据"，列本身没有压缩语义，索引/比较都无效。
5. **InnoDB FTS 没有 zlib 压缩倒排索引**：fts0opt.cc 的 zlib 是纯内存缓冲，落盘的倒排压缩是 ilist delta 编码。
6. **clone 有两套压缩别混**：`clone_enable_compression` 管网络带宽，TPC 重压缩管 recipient 磁盘稀疏性，可叠加。

## 参考

- [`index/btr.md`](../index/btr.md) 压缩页专节（page_zip 操作算法）
- [`innodb/fil.md`](../innodb/fil.md)（TPC 实现链）
- [`innodb/physical/lob.md`](../innodb/physical/lob.md)（ZBLOB）
- [`feat/fts.md`](fts.md)、[`server/replication/binlog.md`](../server/replication/binlog.md)、[`server/auth/authentication.md`](../server/auth/authentication.md)、[`server/plugin/clone.md`](../server/plugin/clone.md)

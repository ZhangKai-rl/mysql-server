# MySQL Binlog 机制深度解析

> 基于 MySQL 8.0.39 源码，涵盖 binlog 物理结构与写入路径（magic/FD/IO_CACHE/binlog cache/rotate/purge）、Ordered Commit 三阶段流水线与源码级剖析、BGC Ticket 系统、内部 2PC、**崩溃后 binlog 如何裁决 prepared 事务**、binlog_order_commits 参数、commit 阶段与 trx->no、Anonymous_Gtid、**GTID 与 binlog 的持久化交互**、事件字节级布局、Row Image 记录过程、**读侧 dump 线程**、**半同步与 binlog 的衔接**。
>
> **边界**：本篇讲 binlog 侧机制（物理结构、BGC 流水线、2PC 协调与裁决、事件字节布局、GTID 持久化、读侧 dump、半同步）。**外部 XA**（由外部 TM 裁决的分布式事务、`XA_prepare_log_event`、`Xa_state_list` 六态）已独立成篇，见 [`../xa.md`](../xa.md)；2PC 的引擎侧执行（trx prepare 逐行剖析、undo 状态）与事务生命周期见 [`../../innodb/trx.md`](../../innodb/trx.md)；崩溃恢复的引擎侧（redo 前滚、扫描解析状态机）见 [`../../innodb/recovery.md`](../../innodb/recovery.md)。

## 目录

- [binlog 物理结构与写入路径](#binlog-物理结构与写入路径)
  - [文件布局：magic + FD + 顺序事件流](#文件布局magic--fd--顺序事件流)
  - [IO_CACHE：唯一的落盘通道](#io_cache唯一的落盘通道)
  - [binlog cache：两个 cache 与临时文件溢出](#binlog-cache两个-cache-与临时文件溢出)
  - [事件序列化：三段式 + 流式 CRC32](#事件序列化三段式--流式-crc32)
  - [文件管理：rotate / purge / 出错策略](#文件管理rotate--purge--出错策略)
- [Ordered Commit 三阶段流水线](#ordered-commit-三阶段流水线)
  - [提交流程全景图](#提交流程全景图)
  - [ordered_commit 源码级剖析](#ordered_commit-源码级剖析)
- [BGC Ticket 系统](#bgc-ticket-系统)
- [内部 2PC（两阶段提交）](#内部-2pc两阶段提交)
- [binlog 与崩溃恢复：2PC 裁决](#binlog-与崩溃恢复2pc-裁决)
- [binlog_order_commits 参数](#binlog_order_commits-参数)
- [Commit 阶段与 trx_no](#commit-阶段与-trx_no)
- [Anonymous_Gtid](#anonymous_gtid)
- [GTID 与 binlog 的持久化交互](#gtid-与-binlog-的持久化交互)
- [Event 格式与 mysqlbinlog 解读](#event-格式与-mysqlbinlog-解读)
- [事件二进制布局（字节级）](#事件二进制布局字节级)
- [Row Image 记录过程](#row-image-记录过程)
- [binlog 读侧：dump 线程（Binlog_sender）](#binlog-读侧dump-线程binlog_sender)
- [半同步复制与 binlog 的衔接](#半同步复制与-binlog-的衔接)
- [参考](#参考)

---

## binlog 物理结构与写入路径

### 文件布局：magic + FD + 顺序事件流

```c
#define BINLOG_MAGIC "\xfe\x62\x69\x6e"   // \xfe 'b' 'i' 'n'
#define BIN_LOG_HEADER_SIZE 4U
#define LOG_EVENT_HEADER_LEN 19U          // 每个事件的固定头长度
#define BINLOG_CHECKSUM_LEN CHECKSUM_CRC32_SIGNATURE_LEN   // 4
#define LOG_EVENT_BINLOG_IN_USE_F 0x1
```

magic 只在**新建空文件**时写一次（`open_binlog()`），紧接着写 FD：

```c
  if (m_binlog_file->is_empty()) {
    m_binlog_file->write(pointer_cast<const uchar *>(BINLOG_MAGIC), BIN_LOG_HEADER_SIZE);
    write_file_name_to_index_file = true;
  }
  if (!is_relay_log) s.common_header->flags |= LOG_EVENT_BINLOG_IN_USE_F;
  if (write_event_to_binlog(&s)) goto err;        // FD (FORMAT_DESCRIPTION_EVENT)
  ... Previous_gtids_log_event                    // 仅 GTID 模式 / relay log
  if (m_binlog_file->flush_and_sync()) goto err;
```

- magic 是"这个文件是 binlog"的唯一自证标识；读端 `read_binlog_magic()` 用 `memcmp` 判定，失败即 `BAD_BINLOG_MAGIC`（加密文件以 `ENCRYPTION_MAGIC` 占位）
- **FD 必须是文件里第一个事件**——它携带 `binlog_version`、server version、以及各事件类型的 post-header length 数组，读者必须先解析 FD 才能解析后续任何事件
- FD 头打 `LOG_EVENT_BINLOG_IN_USE_F`，正常关闭时清 0；重启后该位仍为 1 即判定为 **crash binlog**，进入恢复流程（截断不完整事件 + 清标志）

```
┌──────────────────── binlog.000007 ────────────────────┐
│ 0   │ fe 62 69 6e                magic（4B）           │
│ 4   │ FORMAT_DESCRIPTION_EVENT   flags |= IN_USE_F    │
│     │ PREVIOUS_GTIDS_LOG_EVENT   （GTID 开启时）        │
│     ├─────────────────────────────────────────────────┤
│     │ GTID / ANONYMOUS_GTID                           │
│     │ QUERY("BEGIN")                                  │
│     │ TABLE_MAP → WRITE_ROWS → ... → XID   ← 一个事务  │
│ ... │ ROTATE("binlog.000008")     ← 轮转时追加在旧文件尾│
└───────────────────────────────────────────────────────┘
  每个 event = [19B 通用头][私有 post-header + body][4B CRC32?]
  通用头：timestamp(4) type(1) server_id(4) event_size(4) end_log_pos(4) flags(2)
```

★ **事件之间没有"下一个偏移"指针**：靠物理紧邻（写完 N 字节就停在那）+ 头里的 `end_log_pos`（本事件结束后的文件绝对偏移）。所以 binlog **只能顺序读**；`end_log_pos` 的价值是自校验与提供复制位点，不是跳转。这也是"出现半个事件必须截断到最后一个合法边界"的原因。

`.index` 索引文件内容极简（每行一个 binlog 路径），但写入是 **crash-safe 三步**：

```c
int MYSQL_BIN_LOG::add_log_to_index(...) {
  if (open_crash_safe_index_file()) goto err;
  if (copy_file(&index_file, &crash_safe_index_file, 0)) goto err;   // ① 整份拷贝
  my_b_write(&crash_safe_index_file, log_name, log_name_len);
  my_b_write(&crash_safe_index_file, pointer_cast<const uchar *>("\n"), 1);
  flush_io_cache(&crash_safe_index_file);                            // ② append + flush
  mysql_file_sync(crash_safe_index_file.file, MYF(MY_WME));          //    + fsync
  if (close_crash_safe_index_file()) goto err;
  if (move_crash_safe_index_file_to_index_file(need_lock_index)) goto err;   // ③ rename
}
```

任意时刻崩溃，index 要么是旧版、要么是完整新版，不会出现半行。文件编号由 `generate_new_name()` 扫目录取最大 N + 1。

### IO_CACHE：唯一的落盘通道

```c
struct IO_CACHE {
  my_off_t pos_in_file{0};      // buffer 首字节对应的文件偏移
  my_off_t end_of_file{0};      // 写缓存：容量上限（binlog cache 复用为阈值）
  uchar *write_buffer{nullptr}; // 写 buffer
  uchar *write_pos{nullptr};    // 当前写点
  uchar *write_end{nullptr};    // 写区上界
  int (*write_function)(IO_CACHE *, const uchar *, size_t){nullptr};
  ulong disk_writes{0};         // 真正落盘次数 → binlog_cache_disk_use
  bool disk_sync{false};        // flush 时是否顺带 fsync
};

inline int my_b_write(IO_CACHE *info, const uchar *buffer, size_t count) {
  if (info->write_pos + count <= info->write_end) {   // 快路径：纯 memcpy
    memcpy(info->write_pos, buffer, count);
    info->write_pos += count;
    return 0;
  }
  return (*info->write_function)(info, buffer, count); // 慢路径：_my_b_write
}
```

★ 8.0.39 **已无 `write_io_cache` 符号**（老版本名），现在是 `my_b_write` / `my_b_safe_write` / `flush_io_cache` 三件套。

- `my_b_flush_io_cache()` 只做 `write(2)`（经 `mysql_encryption_file_write`），**不等于 fsync**；fsync 只在 `disk_sync` 为真时顺带做，binlog 主文件不走这条路而是显式 `sync()`
- `_my_b_write()` 的 `pos_in_file + buffer_length > end_of_file` 是**容量超限的唯一判定点**，返回 `EFBIG`

```
 事务线程                     FLUSH 阶段(leader)         SYNC 阶段(leader)
 ─────────                     ────────────────           ────────────────
 Log_event::write()
   └─ IO_CACHE_ostream::write
        └─ my_b_safe_write ──► IO_CACHE.write_buffer（用户态内存）
                                    │ write_pos 越过 write_end
                                    ▼
                          my_b_flush_io_cache()
                          mysql_encryption_file_write()   ← write(2)
                                    ▼
                              OS page cache
                                    │ sync_binlog_file(): mysql_file_sync()
                                    ▼
                                 磁盘 (fsync)
```

**`sync_binlog` 语义**（由 leader 在 SYNC 阶段执行，`sync_counter` 按**组**计数，不是按事务/事件）：

| 值 | fsync 次数 | 说明 |
|---|---|---|
| 0 | 从不 fsync | 只 `flush()` 到 OS page cache，靠 OS 刷盘 |
| 1 | 每个 group commit 一次 | 最安全；`update_binlog_end_pos_after_sync = true`，fsync 后才对外公布 end_pos |
| N | 每 N 个 group 一次（`++sync_counter >= N` 后清零） | 崩溃最多丢最近 N-1 个 group |

relay log 是**另一套计数器**：`after_write_to_relay_log()` 每个事件都 `flush_and_sync`，周期取 `sync_relay_log`。

### binlog cache：两个 cache 与临时文件溢出

```c
class binlog_cache_mngr {
  binlog_stmt_cache_data stmt_cache;   // 非事务表
  binlog_trx_cache_data  trx_cache;    // 事务表
  binlog_cache_data *get_binlog_cache_data(bool is_transactional) {
    if (is_transactional) return &trx_cache; else return &stmt_cache;
  }
  int flush(THD *thd, ...) {           // 顺序固定：先 stmt 后 trx
    stmt_cache.flush(...); trx_cache.flush(...);
  }
};
```

**为什么必须两个**：非事务表的修改无法回滚，其 binlog 也不能丢；事务表的 binlog 必须能被 `ROLLBACK` 整段丢弃（trx_cache 有 `truncate()` / `cache_state_map` / `before_stmt_pos` 支持语句级与 savepoint 级回退，stmt_cache 没有）。混在一个 cache 里就无法实现"回滚事务表部分、保留非事务表部分"。

```
        binlog_cache_data::write_event(ev) → binary_event_serialize → my_b_safe_write
                  │
        ┌───────────────────────┐   write_pos + len <= write_end ?
        │ 内存 buffer            │──是──► memcpy 返回（0 次磁盘 IO）
        │ binlog_cache_size      │
        └───────────────────────┘
                  │ 否 → _my_b_write()
                  ▼
   pos_in_file + buffer_length > end_of_file(=max_binlog_cache_size) ?
        │是                              │否
        ▼                                ▼
   errno=EFBIG → ER_TRANS_CACHE_FULL   my_b_flush_io_cache()
   （或 ER_STMT_CACHE_FULL）            if (file==-1) real_open_cached_file()
                                          → 惰性创建 tmp file（tmpdir/MLxxxx）
                                        ++disk_writes → binlog_cache_disk_use++
```

`binlog_cache_size` 只是**初始内存 buffer 大小**，`max_binlog_cache_size` 才是溢出阈值。溢出到临时文件不算错误，只是性能下降；**超限**（超过 max）才报 `ER_TRANS_CACHE_FULL`。

`finalize()` 负责"event → IO_CACHE"（补 pending rows event、写 XID/COMMIT），`flush()` 负责"IO_CACHE → binlog 文件"。`do_write_cache()` 不重新解析事件，而是 `stream_copy()` 把 cache 字节流按页喂给 `Binlog_event_writer`，因此**一个事件跨多个 cache 页也能正确重组**。

### 事件序列化：三段式 + 流式 CRC32

```c
// 每个 event::write() 都是这三段
return (write_header(ostream, sizeof(xid)) ||
        wrapper_my_b_safe_write(ostream, (uchar *)&xid, sizeof(xid)) ||
        write_footer(ostream));
```

- `write_header`：写 19B 头；若需要 checksum 则初始化 crc 并把 `BINLOG_CHECKSUM_LEN` 计入 `event_size`
- `wrapper_my_b_safe_write`（body）：`crc = checksum_crc32(crc, buf, size)` **增量累加**
- `write_footer`：尾部追加 4B crc32

★ 关键细节：**FD 事件算 crc 前先临时清除 `LOG_EVENT_BINLOG_IN_USE_F`**，因为正常关闭会清该位，否则校验永远对不上。

**进 cache 时 `end_log_pos=0`**（此时不知道最终文件偏移），真正的 `end_log_pos` 与 checksum 由 `Binlog_event_writer::update_header()` 在 flush 时回填——所以它必须逐字节重组事件头，跨页处理复杂。

### 文件管理：rotate / purge / 出错策略

```c
int MYSQL_BIN_LOG::new_file_impl(...) {
  mysql_mutex_lock(&LOCK_xids);
  while (get_prep_xids() > 0)                       // ① 等 in-flight 原子 DDL/XA 提交完
    mysql_cond_wait(&m_prep_xids_cond, &LOCK_xids);
  ...
  if ((error = ha_flush_logs(true))) goto end;       // ② 先让引擎 redo 落盘
  if ((error = generate_new_name(new_name, name))) goto end;
  if (m_binlog_file->is_open()) {
    Rotate_log_event r(new_name + dirname_length(new_name), ...);
    if ((error = write_event_to_binlog(&r))) goto end;   // ③ ROTATE 写在旧文件末尾
    if ((error = m_binlog_file->flush())) goto end;
  }
  close(LOG_CLOSE_TO_BE_OPENED | LOG_CLOSE_INDEX, false, false);
  error = open_binlog(old_name, new_name_ptr, ...);      // ④ 开新文件（magic+FD+Prev_gtids）
}
```

轮转 = "写 ROTATE 到旧文件尾 → flush → 关旧 → 开新 → 追加 index 行"。两处屏障的意义：**等 `prep_xids==0`** 防止原子 DDL 的 binlog 与引擎状态在文件边界撕裂；**`ha_flush_logs`** 保证轮转时引擎侧已持久化。

**purge 的优先级**：`binlog_expire_logs_seconds` **优先于** `expire_logs_days`——只要前者 > 0 后者被完全忽略（8.0 里设 seconds 时 days 会被置 0）。`purge_logs()` 按文件名边界删，`purge_logs_before_date()` 按 mtime 删；都跳过当前活跃文件与被 dump 线程占用的文件。

**写失败的两种策略**（`binlog_error_action`）：

| 值 | 行为 |
|---|---|
| `ABORT_SERVER`（默认） | `exec_binlog_error_action_abort()` → `ER_BINLOG_LOGGING_IMPOSSIBLE` + `my_abort()`，宁可停服也不让 binlog 与引擎分叉 |
| `IGNORE_ERROR` | `close(LOG_CLOSE_STOP_EVENT)` 关闭 binlog 并写 Stop 事件，让引擎继续提交（复制中断需人工介入） |

## Ordered Commit 三阶段流水线

### 提交流程全景图

下图是事务从发起 COMMIT 到完成的完整流程（对应 `ha_commit_trans` → `ordered_commit`），按准备/FLUSH/SYNC/COMMIT 四段着色，并标注崩溃恢复分界：

```mermaid
flowchart TD
    Start([用户线程发起 COMMIT]) --> P1

    subgraph S0[" 准备阶段（进入 ordered_commit 前） "]
        P1["① 获取 MDL COMMIT 锁（防 FTWRL）<br/>handler.cc:1751"]
        P2["② 引擎 prepare：InnoDB undo 标记 PREPARED + 写 redo<br/>（HA_IGNORE_DURABILITY，暂不 fsync）<br/>trx_prepare_low · trx0trx.cc:2976"]
        P3["③ binlog cache finalize，追加 Xid_log_event<br/>binlog.cc:8243"]
        P1 --> P2 --> P3
    end

    subgraph S1[" FLUSH 阶段（Leader 持 LOCK_log） "]
        F1["④ 加入 flush 队列（同组事务排队）"]
        F2["⑤ Leader 获取 LOCK_log"]
        F3["⑥ fetch 整个 flush 队列 = group 快照<br/>fetch_and_process_flush_stage_queue · binlog.cc:8412"]
        F4["⑦ ha_flush_logs：InnoDB redo 批量 fsync<br/>innobase_flush_logs → log_buffer_flush_to_disk"]
        F5["⑧ 分配 sequence_number（MTS 逻辑时钟）<br/>binlog_cache_data::flush · binlog.cc:2419"]
        F6["⑨ binlog cache 顺序 write → binlog 文件（仅到 OS cache）<br/>process_flush_stage_queue · binlog.cc:8456"]
        F1 --> F2 --> F3 --> F4 --> F5 --> F6
    end

    subgraph S2[" SYNC 阶段（Leader 持 LOCK_sync） "]
        Y1["⑩ 加入 sync 队列，释放 LOCK_log"]
        Y2["⑪ Leader 获取 LOCK_sync，fetch sync 队列"]
        Y3["⑫ sync_binlog_file → fsync(binlog)<br/>★ 提交点（崩溃恢复分界线）<br/>binlog.cc:8659"]
        Y1 --> Y2 --> Y3
    end

    subgraph S3[" COMMIT 阶段（Leader 持 LOCK_commit） "]
        C1["⑬ 加入 commit 队列，释放 LOCK_sync"]
        C2["⑭ Leader 获取 LOCK_commit"]
        C3["⑮ InnoDB 引擎层 commit：undo COMMITTED、分配 trx->no、<br/>释放行锁、更新 MVCC<br/>trx_commit_in_memory"]
        C4["⑯ Leader 有序批量写 gtid_executed<br/>update_commit_group · binlog.cc:8562"]
        C5["⑰ 释放 LOCK_commit"]
        C1 --> C2 --> C3 --> C4 --> C5
    end

    P3 --> F1
    F6 --> Y1
    Y3 --> C1

    C5 --> POST["⑱ after_commit hook（半同步 ACK 时机）<br/>回复客户端 OK"]
    POST --> Done([提交完成])

    Y3 -.->|"此前崩溃 → 恢复 ROLLBACK"| RB["Xid_log_event 未落盘"]
    Y3 -.->|"此后崩溃 → 恢复 COMMIT"| CM["Xid_log_event 已落盘"]

    classDef prep fill:#d6eaf8,stroke:#2e86c1,color:#1b4f72
    classDef flush fill:#fadbd8,stroke:#c0392b,color:#7b241c
    classDef sync fill:#e8daef,stroke:#8e44ad,color:#4a235a
    classDef commit fill:#fcf3cf,stroke:#b7950b,color:#7d6608
    classDef point fill:#f5b7b1,stroke:#c0392b,stroke-width:3px,color:#7b241c
    class P1,P2,P3 prep
    class F1,F2,F3,F4,F5,F6 flush
    class Y1,Y2 sync
    class Y3 point
    class C1,C2,C3,C4,C5 commit
```

> 关于右下"写盘层次"：⑨ 的 `write` 只把 binlog cache 写入 **OS page cache**（sync_binlog=1 时此处仍可能因崩溃丢失）；⑫ 的 `fsync` 才把数据刷到 **binlog 物理文件**，这才是持久化与崩溃恢复的分界。InnoDB 侧同理——⑦ 的 redo fsync 保证 prepare 记录落盘，但引擎自身的 commit（undo COMMITTED / trx->no）在 ⑮ 才发生，且不再强制 fsync（`trx->flush_log_later`）。

事务提交进入 `MYSQL_BIN_LOG::ordered_commit`（binlog.cc:8853）后，经历三个阶段的流水线。Leader 线程代表整个 group 穿越所有 stage，Follower 线程从进入 flush stage 起沉睡，直到 `signal_done` 才被唤醒。

### Flush Stage（持 LOCK_log）

Leader 串行处理 group 内每个事务的 binlog cache，将其写入 binlog 文件。这一阶段是提交的真正瓶颈所在——`process_flush_stage_queue`（binlog.cc:8456）串行 flush 每个事务的 binlog cache。

关键操作包括：调用 `ha_flush_logs(true)` 将整组事务的 InnoDB redo prepare 记录批量 fsync（redo 持久化），然后将 binlog cache（含 `Xid_log_event`）写入 binlog 文件。

崩溃注入点 `crash_commit_before_log`（binlog.cc:8867）位于写 binlog 之前。

### Sync Stage（持 LOCK_sync）

对 binlog 文件执行 `fsync`。`sync_binlog_file`（binlog.cc:8659）只是普通的 fsync 调用，没有 group 概念——group 的合并在 stage queue 层面完成，sync 阶段只是把整组事务的落盘合并成一次 fsync。

**这里是提交点（Point of No Return）**。崩溃注入点 `crash_commit_after_log`（binlog.cc:8964）位于 fsync 成功之后。在此之后崩溃，事务视为已提交。

### Commit Stage（持 LOCK_commit）

Leader 按 binlog 顺序逐个调用引擎层 commit（`finish_transaction_in_engines`），释放行锁、更新 MVCC 可见性。详见后文 [Commit 阶段与 trx_no](#commit-阶段与-trx_no)。

### 分组控制参数

| 参数 | 作用 |
|------|------|
| `binlog_group_commit_sync_delay` | 事务延迟提交的微秒数（最大 1 秒），增加组内事务数量，减少 sync 次数 |
| `binlog_group_commit_sync_no_delay_count` | 组内事务数达到此值时立即提交，无需等待 delay |

### Group 的形成

Group 的形成完全由 `LOCK_log` 和 stage queue 控制，与 ticket 无关。Leader 持有 `LOCK_log` 期间，新到达的事务进入 flush stage 队列成为 Follower。Leader 释放 `LOCK_log` 后，下一个事务成为新 group 的 Leader。

---

### ordered_commit 源码级剖析

> 上面是流水线概览，本节给代码级实现：谁当 leader、队列怎么组织、三把 mutex 怎么交接、follower 什么时候醒。

**leader / follower 的判定**——`change_stage` → `enroll_for`，判据是"**入队前队列是否为空**"，没有任何选举协议：

```cpp
bool Commit_stage_manager::enroll_for(StageID stage, THD *thd,
                                      mysql_mutex_t *stage_mutex,
                                      mysql_mutex_t *enter_mutex) {
  bool leader = this->append_to(stage, thd);        // 队列空 ⇒ 我是 leader
  ...
  if (stage_mutex) mysql_mutex_unlock(stage_mutex);  // ★ 先释放上一阶段的 stage mutex

  if (!leader) {                                    // ===== follower：睡 =====
    mysql_mutex_lock(&m_lock_done);
    while (thd->tx_commit_pending)                  // ★ while：共享 cond，防虚假唤醒
      mysql_cond_wait(&m_stage_cond_binlog, &m_lock_done);
    mysql_mutex_unlock(&m_lock_done);
    return false;
  }
  if (leader && enter_mutex != nullptr) mysql_mutex_lock(enter_mutex);  // ===== leader：加锁干活 =====
  return leader;
}
```

★ **"先解锁旧的、再加锁新的"** 是流水线能重叠的关键：本组在 sync 时，下一组已经可以拿 `LOCK_log` 开始 flush。

**队列**：`Commit_stage_manager` 单例持有 5 个 `Mutex_queue`（FLUSH / SYNC / COMMIT / AFTER_COMMIT / COMMIT_ORDER_FLUSH），每个队列是**用 `THD::next_to_commit` 串起来的侵入式单链表**，`m_last` 是指针的指针：

```
        m_queue[SYNC_STAGE]
        ┌──────────────────────────────────────────────┐
        │ m_first ─► THD1 ──next_to_commit──► THD2 ──► THD3 ──► nullptr │
        │ m_last ──────────────────────────────────────────▲ (&THD3->next)│
        └────────────────────────────────────────────────────────────────┘
  append():  *m_last = first; 走到链尾; m_last = &tail->next_to_commit;
             返回「入队前是否为空」⇒ leader 判据
  fetch_and_empty(): result = m_first; 清空（链表本身不拆，仍串着整组）
```

注意 **队列锁 `m_queue_lock[]` 与 stage mutex（`LOCK_log`/`LOCK_sync`/`LOCK_commit`）是两把不同的锁**：入队（短操作）与阶段工作（长，含 I/O）因此不互斥。

**三阶段主干**（leader 视角，错误处理已略）：

```cpp
int MYSQL_BIN_LOG::ordered_commit(THD *thd, bool all, bool skip_commit) {
  init_thd_variables(thd, all, skip_commit);     // tx_commit_pending = true（follower 的谓词）

  /* Stage #1 FLUSH：进入时拿 LOCK_log */
  if (change_stage(thd, BINLOG_FLUSH_STAGE, thd, nullptr, &LOCK_log))
    return finish_commit(thd);                   // 我是 follower → 直接收尾
  flush_error = process_flush_stage_queue(&total_bytes, &wait_queue);
  if (flush_error == 0 && total_bytes > 0) flush_error = flush_cache_to_file(&flush_end_pos);

  /* Stage #2 SYNC：交出 LOCK_log，拿 LOCK_sync；整批 wait_queue 入 sync 队列 */
  if (change_stage(thd, SYNC_STAGE, wait_queue, &LOCK_log, &LOCK_sync))
    return finish_commit(thd);
  if (!flush_error && (sync_counter + 1 >= get_sync_period()))
    Commit_stage_manager::get_instance().wait_count_or_timeout(
        opt_binlog_group_commit_sync_no_delay_count,
        opt_binlog_group_commit_sync_delay, SYNC_STAGE);      // ★ throttle
  final_queue = Commit_stage_manager::get_instance().fetch_queue_acquire_lock(SYNC_STAGE);
  std::pair<bool, bool> result = sync_binlog_file(false);     // ★ fsync

  /* Stage #3 COMMIT（+ AFTER_COMMIT） */
  if (change_stage(thd, COMMIT_STAGE, final_queue, &LOCK_sync, &LOCK_commit))
    return finish_commit(thd);
  process_commit_stage_queue(thd, commit_queue);              // ★ 引擎提交在这里
  if (change_stage(thd, AFTER_COMMIT_STAGE, commit_queue, &LOCK_commit, &LOCK_after_commit))
    return finish_commit(thd);
  process_after_commit_stage_queue(thd, after_commit_queue);

  Commit_stage_manager::get_instance().signal_done(final_queue);  // ★ 唤醒全部 follower
  (void)finish_commit(thd);
  ... rotate ...
}
```

**FLUSH 阶段做三件事**（leader 一个人替整组干完）：

```cpp
  THD *first_seen = fetch_and_process_flush_stage_queue();   // ① ha_flush_logs：整组 prepared redo 落盘
  assign_automatic_gtids_to_flush_group(first_seen);         // ② 按入队顺序分配 GTID
  for (THD *head = first_seen; head; head = head->next_to_commit)
    flush_thread_caches(head);                               // ③ 逐个把 binlog cache 拷进共享 IO_CACHE
  m_binlog_file->flush();                                    //    最后一次 write(2)
```

**SYNC 阶段的 throttle**（`binlog_group_commit_sync_delay` / `sync_no_delay_count`）：

```cpp
void Commit_stage_manager::wait_count_or_timeout(ulong count, long usec, StageID stage) {
  long delta = std::max<long>(1, (to_wait * 0.1));           // 按 10% 切片
  while (to_wait > 0 && (count == 0 || m_queue[stage].get_size() < count)) {
    my_sleep(delta);                                         // ★ 分段 sleep 轮询队列长度，不是 cond_timedwait
    to_wait -= delta;
  }
}
```

- 位置：sync stage、**持 `LOCK_sync`**、且在 `fetch_queue_acquire_lock` **之前**——晚到的会话能继续入队，达到 `no_delay_count` 就提前结束等待
- ★ leader **持锁睡眠**，会阻塞其它组进入 sync
- `sync_binlog=0` 时 `get_sync_period()==0`，条件恒真 ⇒ **每组都延迟**

**COMMIT 阶段**：`process_commit_stage_queue` 遍历链表做两件大事——`m_dependency_tracker.update_max_committed(head)`（并行复制依赖跟踪，见 prpl.md）与 `finish_transaction_in_engines()`（InnoDB 真正提交），然后 `gtid_state->update_commit_group(first)` **按链表顺序**把 GTID 入 `gtid_executed`（乱序会产生临时空洞，导致 Gtid_set 反复增删 interval）。AFTER_COMMIT 独立成第四阶段，因为 `after_commit` hook 可能很慢（如 Group Replication），不让它占 `LOCK_commit`。

**follower 何时醒**：只有最后一次 `signal_done(final_queue)`（commit/after-commit 全部完成、锁都释放之后）才把 `tx_commit_pending` 置 false 并 broadcast。中间阶段切换 follower 一直是睡着的 ⇒ **三阶段流水线对 follower 完全透明**。

**与 2PC 的衔接**（提交点在 flush 之后、引擎 commit 之前）：

```
ha_flush_logs（prepared redo 落盘）
  → binlog write + flush（flush_cache_to_file）
  → after_flush hook
  → sync_binlog_file（sync_binlog=1 时 fsync）      ← ★ 提交点：此后崩溃也要重做
  → call_after_sync_hook（半同步在此回 ACK）
  → process_commit_stage_queue → ha_commit_low    ← 引擎提交
```

`ordered_commit` 返回后**不再有** `ha_commit_low`：引擎提交只发生在 (a) leader 的 `process_commit_stage_queue`，或 (b) `binlog_order_commits=0` / follower 路径下的 `finish_commit`。

| 决策 | 收益 | 代价 / 约束 |
|---|---|---|
| leader 判定 = "入队前队列为空" | 无选举协议、无额外原子操作 | 谁是 leader 不确定；leader 慢会拖整组 |
| 队列锁与 stage mutex 分离 | 入队（短）与阶段工作（长，含 I/O）不互斥 | 须严格遵守"先解锁旧再加锁新"以免死锁 |
| 先解锁旧锁再加新锁 | 相邻阶段可重叠（组 A sync 时组 B flush） | 存在极短无 stage 锁窗口 |
| follower 睡到组末 | follower 零 I/O；顺序由 leader 保证 | 尾延迟 = 整组最慢成员 |
| GTID 由 leader 批量 `update_commit_group` | `gtid_executed` 保持单区间 | 必须在有序提交语义下 |
| throttle 持 `LOCK_sync` 用 `my_sleep` 轮询 | 简单，能随时响应 `no_delay_count` | 阻塞其它组进入 sync |
| `sync_counter` 按"组"计数 | 语义清晰、实现极简 | `sync_binlog=N` 的粒度是 group 而非事务数 |
| AFTER_COMMIT 独立成第四阶段 | 慢 hook（GR）不占 `LOCK_commit` | 多一次入队/交接 |
| `after_sync` 早于引擎 commit | 半同步可在主库可见前回 ACK | sync 失败连带影响提交路径 |

## BGC Ticket 系统

### Ticket 的基本机制

`Bgc_ticket_manager`（bgc_ticket_manager.h）维护两个单调递增的 ticket 值：`front_ticket`（当前正在放行的号）和 `back_ticket`（下次分配的号）。

三个核心操作用 ticket 原子变量的最高位（MSB）当锁位做 CAS 来串行化：

- `assign_session_to_ticket`：给事务分配当前 back_ticket（幂等由 THD 侧 `Binlog_group_commit_ctx::assign_ticket()` 保证——`m_session_ticket.is_set()` 则跳过，manager 侧函数本身不幂等）
- `push_new_ticket`：递增 back_ticket，开启新的一轮
- `pop_front_ticket`：推进 front_ticket 到下一个号

线程进入 flush stage 队列前，先调 `wait_for_ticket_turn`（rpl_commit_stage_manager.cc:214），**只有当自己的 ticket == front_ticket 时才被放行进队列**。

### 一个 group 内 ticket 一定相同

因为 `append_to` 先 `wait_for_ticket_turn`（只放行 ticket==front）再进队列，而 group 本质是 leader 用 `fetch_queue` 对队列做的一次快照。能同时待在队列里的成员都满足 ticket==front，所以 group 内所有成员的 ticket 必然等于同一个 front 值。

反过来不成立：ticket 相同不代表是同一个 group。在普通 BGC 中所有事务 ticket 都是 1，但系统仍按 `LOCK_log` + stage queue 切出多个不同的 group。

### 普通BGC中 ticket 不激活

`push_new_ticket` 只在 GR（Group Replication）视图变更和 debug 测试注入中调用。普通 BGC 中 `back_ticket = front_ticket = 1` 永不变化，`wait_for_ticket_turn` 直接短路通过（零开销存在）。Group 形成完全由 `LOCK_log` + stage queue 控制。

### Ticket 的底层实现：63 位值 + 1 位 MSB 锁

`BgcTicket` 把"值"和"临界区状态"编码进同一个 64 位原子：低 63 位是 ticket 值（`max_ticket_value = 2^63-1`，超界回绕到 1），最高位是 in-use 锁位。`AtomicBgcTicket::set_in_use` 用 CAS 自旋拿锁：

```cpp
std::pair<BgcTicket, BgcTicket> AtomicBgcTicket::set_in_use(
    bool inc_next_before_lock, bool inc_next_before_release) {
  BgcTicket prev_ticket, next_ticket;
  while (true) {
    auto current_value =
        m_ticket->load(std::memory_order_acquire) & BgcTicket::clear_bit;
    prev_ticket = next_ticket = BgcTicket(current_value);
    BgcTicket in_use_ticket = prev_ticket;
    if (inc_next_before_lock) {
      in_use_ticket.set_next();
      next_ticket.set_next();
    } else if (inc_next_before_release) {
      next_ticket.set_next();
    }
    in_use_ticket.set_in_use();
    if (m_ticket->compare_exchange_strong(current_value, in_use_ticket.get(),
                                          std::memory_order_release)) {
      return std::make_pair(prev_ticket, next_ticket);
    }
    std::this_thread::yield();
  }
}
```

`AtomicBgcTicketGuard`（RAII）构造时 `set_in_use`（拿锁 = MSB 置 1），析构时 `set_used`（写入新值，`assert` 校验 MSB 必为 0 即释放）。**与 server 层 `Integrals_lockfree_queue`（`sql/containers/`）head/tail 的 MSB 占用位是同构手法**——一个原子变量同时承载"值"与"临界区状态"。

### 会话计数队列与"计数对齐"放行协议

`m_sessions_per_ticket` 是 `Integrals_lockfree_queue<uint64_t>`（无锁队列，见 infra/structure 篇）实例，记录每个"已关闭分配"的 ticket 的会话总数。**容量硬编码 1024（`max_concurrent_tickets`），无 sysvar 控制**。叫号的唯一放行条件在 `pop_front_ticket`：

```cpp
BgcTicket::ValueType front_ticket_sessions =
    this->m_sessions_per_ticket.front();
if (prev_front != this->m_back_ticket.load() &&
    this->m_front_ticket_processed_sessions_count == front_ticket_sessions) {
  this->m_front_ticket_processed_sessions_count = 0;
  this->m_sessions_per_ticket.pop();
  front_ticket_guard.set_next(prev_front.next());  /* 放行下一个号 */
}
```

即：**当前 ticket 的"已处理会话数" == 入队时记录的"该 ticket 会话总数"才推进 front**——计数对齐即放行，叫号无需遍历等待者。会话数来自 `assign_session_to_ticket` 的 `++m_back_ticket_sessions_count`，`push_new_ticket` 关闭旧号时把总数入队。

### 等待与放行：1 秒 timedwait

等待侧 `Commit_stage_manager::wait_for_ticket_turn`：`ticket != front_ticket && ticket > coalesced_ticket && !thd->killed` 时在 `m_cond_wait_for_ticket_turn` 上 **1 秒 `cond_timedwait` 循环**（防丢失 broadcast）。放行后经 `update_session_ticket_state` → `add_processed_sessions_to_front_ticket(1, ticket)` 计入已处理，并尝试 `signal_end_of_ticket` → `pop_front_ticket`——前后值不同则 broadcast 唤醒下一号所有等待者。

### manual_ticket_setting 开关与 GR 调用链

普通 BGC 零开销的机制是 `Binlog_group_commit_ctx::manual_ticket_setting()`（`Aligned_atomic<bool>`，默认 false）：**只由 GR 插件**通过 `Commit_stage_manager::enable_manual_session_tickets()`（恢复完成后）/ `disable_manual_session_tickets()`（GR 停止）翻转，**不是 sysvar**。

GR 视图变更时 `Certification_handler::generate_view_change_bgc_ticket()`：

```cpp
ticket_manager.push_new_ticket();            /* 关旧号、旧号会话数入队 */
ticket_manager.pop_front_ticket();           /* 旧号处理完则推进 front */
auto [ticket, _] = ticket_manager.push_new_ticket(
    binlog::BgcTmOptions::inc_session_count);/* View Change Event 独占新号 */
```

返回的 ticket 随 View Change Log Event 传输，applier 经 `set_session_ticket()` 赋给应用线程（仅 manual 模式生效）。`coalesce()`（GR 停止时）清空计数队列、置 `m_coalesced_ticket`，此后所有 ticket 比较直接通过——**零开销逃生舱**，避免屏障泄漏 hang。

### 版本与关系澄清

- **版本**：文件 copyright 2022（**8.0.28 开发周期**产物），不是 8.0.27。
- **与 Commit_order_manager 的关系**：`Commit_order_manager` 在 `sql/rpl_replica_commit_order_manager.h`，是 **slave 侧 MTS 保序提交**机制（worker 按主库顺序提交），**Bgc_ticket_manager 不是为解决它而引入**——两者无替代关系、可共存。ticket 系统为 **GR 视图变更**提供屏障（官方注释原文：View Change Event 被授予单独 ticket 值，屏障前的事务都提交完、之后的才开始）。
- **无 sysvar**：容量硬编码 1024，开关是 `manual_ticket_setting()` 原子 bool；不存在 `binlog_group_commit_ticket_capacity`、`use_bgc_ticket_manager` 这类变量。

### Ticket 的目的

为 Group Replication（MGR）的**认证线程（certification）**与**binlog 提交流水线**之间提供可插入的有序屏障（barrier synchronization）。MGR 下事务先经过 Paxos（XCom）广播 + 认证才能提交，认证是异步进行的。视图变更（成员加入/退出）时，必须保证屏障之前的所有事务都提交完，之后的才开始。Ticket 的"发号-叫号"机制精确地实现了这个屏障。

### 工业界的类似实现

Ticket lock 算法最早由 Mellor-Crummey 和 Scott 在 1991 年 TOCS 论文中系统化提出。工业界最著名的实现是 Linux 内核的 ticket spinlock（2.6.25 引入），后来在 4.16 前后 ARM64 等架构转向 qspinlock（基于 MCS lock 的变体）。

需要强调的是，MySQL 的 `Bgc_ticket_manager` **不是自旋锁**，而是基于条件变量 `cond_timedwait` 阻塞的高层批次调度器。它只借了"发号-叫号、可按序放行、可插入屏障"的思想，用途和实现都与内核 ticket spinlock 完全不同。

### 关于代码中的 "fackbook" 注释

`bgc_ticket_manager.h:217-218` 的文字是用户手写学习笔记（引用了 CSDN 博客，且 "fackbook" 是 "facebook" 的笔误），**不是 MySQL 官方注释**。官方源码未声称 `Bgc_ticket_manager` 源自 Facebook 的 ticket lock，该关联是笔记作者的联想，无官方依据。

### MGR 与 XCom（简述）

MGR 完全开源（GPLv2），代码位于 `plugin/group_replication/`。上层 `src/` 负责认证、成员管理、恢复、流控；下层 `libmysqlgcs/` 封装组通信，其内部核心是 **XCom**——MySQL 自研的 Multi-Paxos 实现，主体在 `libmysqlgcs/src/bindings/xcom/xcom/xcom_base.cc`（307KB），负责在成员间对事务顺序达成一致（total order broadcast）。Ticket 屏障正是服务于 `src/` 认证线程与 binlog 提交之间的同步。

---

## 内部 2PC（两阶段提交）

### 为什么需要 2PC

一个事务的持久化数据分散在两个独立的日志系统：InnoDB redo/undo log（保证引擎内部原子性和持久性）和 binlog（保证复制和 PITR）。两者必须原子地一起提交，否则会导致主从不一致。

### 协调者与参与者

`tc_log` 是全局抽象基类指针，运行时指向具体实现：

- 开启 binlog 时，`tc_log = &mysql_bin_LOG`，binlog 文件充当事务协调日志
- 关闭 binlog 时，`tc_log` 指向 `TC_LOG_DUMMY`，退化为引擎自己 1PC

存储引擎若实现了 `handlerton::prepare` 回调，就是 2PC 参与者。事务注册参与者时，若引擎没有 prepare 方法，直接标记为 `no_2pc`（handler.cc:1333）。

### 2PC 的完整调用链

入口是 `ha_commit_trans`（handler.cc:1615）：

```
第一阶段 PREPARE（handler.cc:1772）:
  仅当 rw_ha_count > 1 且非 no_2pc
  tc_log->prepare(thd, all)
    → ha_prepare_low → 各引擎 prepare
    → InnoDB: 写 prepare 记录到 undo header

第二阶段 COMMIT（handler.cc:1788）:
  tc_log->commit(thd, all)
    → ordered_commit 三阶段流水线
    → flush: ha_flush_logs 持久化 redo prepare + 写 binlog(含 Xid_log_event)
    → sync:  fsync(binlog) ← 提交点
    → commit: ha_commit_low → InnoDB 引擎层真正 commit
```

### binlog 侧的核心环节：finalize cache 与结束事件

> 2PC 的引擎侧剖析（prepare 五层逐行、undo 状态、`HA_IGNORE_DURABILITY` 的协同、触发条件 `rw_ha_count > 1`、外部 XA 的 detach / by_xid）见 [`../../innodb/trx.md`](../../innodb/trx.md)「事务与 binlog：2PC」。本篇只讲 binlog 自己的环节。

`MYSQL_BIN_LOG::commit` 在进 `ordered_commit` 之前，先 finalize binlog cache：

| 动作 | 位置 | 说明 |
|------|------|------|
| 记录 MTS last_committed | `store_commit_parent` | 取当前 `max_committed_timestamp` 作为 commit parent |
| 追加结束事件 | binlog.cc:8243 | 普通 2PC → `Xid_log_event`；XA → `XA_prepare_log_event`；1PC → `Query_log_event("COMMIT")` |
| finalize cache | `cache_mngr->trx_cache.finalize` | 将结束事件写入 trx cache，cache 冻结不再追加 |
| before_commit hook | binlog.cc:8278 | 插件钩子（如半同步复制） |

### 提交点与崩溃恢复

提交点是 sync stage 里 **binlog fsync 成功那一刻**。携带 XID 的 `Xid_log_event` 一旦落盘，事务就算提交。

崩溃恢复入口 `binlog::Binlog_recovery::recover`（binlog.cc:7916）：

1. 扫描最后一个 binlog 文件，收集所有 `Xid_log_event` 中的 XID → 组成 commit_list
2. 让每个存储引擎 recover → InnoDB 扫描 undo，返回所有处于 PREPARED 状态的 XID 列表
3. 对每个 PREPARED 的 XID 做仲裁：在 commit_list 中 → COMMIT；不在 → ROLLBACK

| 崩溃时机 | InnoDB 状态 | binlog 状态 | 恢复决定 |
|---------|------------|-------------|---------|
| `crash_commit_after_prepare` | prepared | 无 XID | rollback |
| `crash_commit_before_log` | prepared | 无 XID | rollback |
| `crash_commit_after_log` | prepared | 有 XID（已 fsync） | commit |
| `crash_commit_after` | committed | 有 XID | 已提交，无需处理 |

### 内部 2PC 与外部 XA

MySQL 的"2PC"有两副面孔，共用同一套 prepare/commit 引擎接口：

- **内部 2PC**：对用户透明，普通 `COMMIT` 内核自动完成，目的是 binlog/redo 一致性。
- **外部 XA**：用户显式 `XA START` / `XA PREPARE` / `XA COMMIT`，跨多个 MySQL 实例，额外写 `XA_prepare_log_event`。引擎侧细节（detach、by_xid 接口、XA 返回码、崩溃恢复状态机）见 [`../../innodb/trx.md`](../../innodb/trx.md)「外部 XA：与内部 2PC 的差异」。

---

## binlog 与崩溃恢复：2PC 裁决

> 「内部 2PC」节讲**正常提交**时 binlog 与引擎怎么配合；本节讲**崩溃后** binlog 凭什么裁决引擎里的 prepared 事务——这是 binlog 作为协调者的核心职责。

> ⚠️ **勘误（8.0.39 实测）**：`xid_cache` / `XID_cache` / `xid_cache_insert` 在 8.0 分支**已不存在**（5.6/5.7 的全局哈希缓存被移除）；`MYSQL_BIN_LOG::recover` 也**不是函数**，它只是 PSI 内存 key 的名字字符串。真实链路是 `open_binlog` → `binlog::Binlog_recovery::recover()` → `ha_recover()` → `recover_one_internal_trx()`。

### 启动恢复主流程

```
                    mysqld 启动
                        │
        MYSQL_BIN_LOG::open_binlog(opt_name)
        assert(!is_relay_log)        ← 只裁决 binlog，不管 relay log
                        │
        find_log_pos() → find_next_log()  定位最后一个 binlog 文件
                        │
        read_binlog_in_use_flag(binlog_file_reader)
                        │
        ┌───────────────┴────────────────┐
   flag 未置位                       flag 置位（崩溃）
   = 干净关闭                            │
   → ha_recover() 空跑               ER_BINLOG_RECOVERING_AFTER_CRASH_USING
   → return 0                            │
                        binlog::Binlog_recovery::recover()
                        顺序扫描事件，边扫边记账：
                          m_internal_xids ← XID_EVENT / atomic DDL ddl_xid
                          m_external_xids ← XA_PREPARE / XA COMMIT / XA ROLLBACK
                          m_valid_pos     ← 仅在事务边界推进
                                         │
                        ha_recover(&m_internal_xids, &xa_list)
                        对每个引擎 prepared 事务：
                          xid ∈ commit_list → commit_by_xid()
                          xid ∉ commit_list → rollback_by_xid()
                                         │
                        truncate_update_log_file(valid_pos)
                        截断尾部半个事务 + 清除 LOG_EVENT_BINLOG_IN_USE_F
```

### 崩溃分支：`open_binlog`

```cpp
  /* If the binary log was not properly closed it means that the server
     may have crashed. ... collect logged XIDs; complete the 2PC; collect
     the last valid position. */
  if (!read_binlog_in_use_flag(binlog_file_reader)) {
    error = ha_recover();          // ① 干净关闭：空跑兜底即可
    return error;
  }
  LogErr(INFORMATION_LEVEL, ER_BINLOG_RECOVERING_AFTER_CRASH_USING, opt_name);

  binlog::Binlog_recovery bl_recovery{binlog_file_reader};
  bl_recovery.recover();           // ② 扫描收集 XID 与最后一个合法位点

  my_off_t valid_pos = bl_recovery.get_valid_pos();
  my_off_t binlog_size = binlog_file_reader.ifile()->length();

  if (bl_recovery.is_binlog_malformed()) { ...return 1; }
  if (bl_recovery.has_engine_recovery_failed()) {
    /* truncate log file but do NOT clear LOG_EVENT_IN_USE_F flag */
    truncate_update_log_file(log_name, valid_pos, binlog_size, false);
    return 1;                      // ③ ★ 引擎侧失败要保留"犯罪现场"
  }
  if (valid_pos > 0) {
    truncate_update_log_file(log_name, valid_pos, binlog_size, true);  // ④ 截断 + 清 flag
  }
```

- ③ 用 `update=false`：引擎裁决失败时**保留** in-use 标记，下次启动再试，避免把证据抹掉

### 扫描阶段：收集 XID 与合法位点

```cpp
binlog::Binlog_recovery &binlog::Binlog_recovery::recover() {
  while (istream >> ev) {
    switch (ev->get_type_code()) {
      case binary_log::QUERY_EVENT:  this->process_query_event(...); break;
      case binary_log::XID_EVENT:    this->process_xid_event(...);   break;
      case binary_log::XA_PREPARE_LOG_EVENT: this->process_xa_prepare_event(...); break;
      default: break;
    }
    // Whenever the current position is at a transaction boundary, save it
    if (!this->m_is_malformed && !this->m_in_transaction &&
        !is_gtid_event(ev.get()) && !is_session_control_event(ev.get()))
      this->m_valid_pos = this->m_reader.position();     // ★ 只在事务边界推进
    if (this->m_is_malformed) break;
  }
  if (!this->m_is_malformed && total_ha_2pc > 1) {
    Xa_state_list xa_list{this->m_external_xids};
    this->m_no_engine_recovery = ha_recover(&this->m_internal_xids, &xa_list);
  }
  return (*this);
}
void binlog::Binlog_recovery::process_xid_event(Xid_log_event const &ev) {
  this->m_is_malformed = !this->m_in_transaction;        // 事务外出现 XID = 损坏
  if (this->m_is_malformed) return;
  this->m_in_transaction = false;
  if (!this->m_internal_xids.insert(ev.xid).second) {    // XID 重复 = 损坏
    this->m_is_malformed = true;
  }
}
```

★ `m_valid_pos` 只在**不在事务中、且非 GTID/会话控制事件**时推进——保证**永远截断在完整事务边界**，不会留下半个事务。

### 唯一的判决表达式

```cpp
void recover_one_internal_trx(xarecover_st const &info, handlerton &ht,
                              XA_recover_txn const &xa_trx, my_xid xid, ...) {
  if (info.commit_list ? info.commit_list->count(xid) != 0
                       : tc_heuristic_recover == TC_HEURISTIC_RECOVER_COMMIT) {
    exec_status = ht.commit_by_xid(&ht, const_cast<XID *>(&xa_trx.id));
  } else {
    exec_status = ht.rollback_by_xid(&ht, const_cast<XID *>(&xa_trx.id));
  }
}
```

**这就是全部裁决逻辑**：`commit_list->count(xid) != 0` → 提交，否则回滚。`commit_list` 就是扫描 binlog 得到的 `m_internal_xids`。引擎侧的 prepared 事务是"待决"状态，**binlog 是唯一裁决者**。

内/外部 XA 的区分靠 `xid_t::get_my_xid()`：能解出内部 XID（`"MySQLXid"` 前缀 + server_id + xid）即内部 2PC，返回 0 则是外部 XA（走 `Xa_state_list` 六态决策，详见 [`../xa.md`](../xa.md)）。

### XID 的分配与落盘

- **何时分配**：无全局缓存，直接复用 `thd->query_id`（全局单调）。第一个支持 2PC 的引擎注册时 `trn_ctx->xid_state()->set_query_id(thd->query_id)`，`is_null()` 保护只分配一次
- **写进哪个事件**：条件是 `rw_ha_count(trx_scope) > 1`——binlog handlerton 也算一个参与者，所以"InnoDB + binlog"就 >1 ⇒ 写 `Xid_log_event`；单引擎无 binlog 则退化成写 `COMMIT` 的 `Query_log_event`（**后者不进** `m_internal_xids`）
- `Xid_log_event` 的负载只有 `sizeof(xid)` 一个 8 字节整数，就是"内部 2PC 的 commit 记录"，与 InnoDB redo 里的 prepare 记录一一对应

### in-use 标记与截断

```cpp
  // 新建文件时置位
  if (!is_relay_log) s.common_header->flags |= LOG_EVENT_BINLOG_IN_USE_F;
  // 正常 close() 时清零
  if (!is_relay_log) {
    my_off_t offset = BIN_LOG_HEADER_SIZE + FLAGS_OFFSET;
    uchar flags = 0;
    (void)m_binlog_file->update(&flags, 1, offset);
  }
```

★ 两处都排除 relay log —— **relay log 永远不设 in-use 标记**，因此它不参与这套裁决。

截断由 `truncate_update_log_file()` 完成（`ofile->truncate(valid_pos)` + 清 flag）。index 文件则走 crash-safe 三步（临时文件 + fsync + rename）。一处例外：**index 文件损坏时无条件 abort**，不尊重 `binlog_error_action`（注释：危害远大于停机，会导致 purge 误删）。

### 持久化顺序：sync_binlog × innodb_flush_log_at_trx_commit

```
                    innodb_flush_log_at_trx_commit = 1   |  = 2        |  = 0
                  ┌─────────────────────────────────────┼─────────────┼──────────────
 sync_binlog = 1  │ 双 1：真正持久                        │ OS 崩溃丢    │ OS 崩溃丢
（真·crash safe） │ redo+binlog 均已落盘，2PC 裁决总正确  │ InnoDB 数据  │ InnoDB 数据
                  ├─────────────────────────────────────┼─────────────┼──────────────
 sync_binlog = N  │ 引擎已 commit、binlog 丢失            │ 两边都可能丢 │ 两边都可能丢
      (>1)        │ → **主从不一致**，最危险              │ 不一致       │ 不一致
                  ├─────────────────────────────────────┼─────────────┼──────────────
 sync_binlog = 0  │ binlog 有、引擎没有 → 提交被回滚       │ 同上         │ 两边都丢
                  │ 丢事务但不产生主从不一致              │             │ 仅靠 2PC 兜底
```

**为什么必须"双 1"**：2PC 的正确性前提是"prepare 落盘"与"binlog 落盘"两个动作各自持久化，崩溃后才能靠对账收敛。

★ **`sync_binlog=N>1` 为什么最危险**：XID 已写入 binlog 页缓存但尚未 fsync，COMMIT 阶段继续推进、InnoDB 已完成 `commit_by_xid`。此时**掉电/OS 崩溃** ⇒ binlog 尾部丢失；重启后 redo 里的事务处于 prepared 而 binlog 里找不到对应 XID ⇒ 被 `rollback_by_xid()` 回滚。**但客户端已收到 commit 成功**，且若 binlog 已发给从库则**从库有、主库无** ⇒ 主从永久不一致。

反向组合（`sync_binlog=0` + 引擎未落盘）是"安全降级"：binlog 有 XID → 提交，但 InnoDB 根本没有该 prepared 事务（无害），结果是**丢事务但不产生主从分歧**。

### relay log 不参与裁决

```cpp
  /* This function is used for 2pc transaction coordination. Hence, it
     is never used for relay logs. */
  assert(!is_relay_log);
```

从库新建 relay log 走多参版 `open_binlog`。relay log 的"恢复"不是截断修复，而是**丢弃重拉**：`init_recovery()` 先 `mts_recovery_groups()`，若有 MTS gap 先补 gap（见 prpl.md），无 gap 才 `recover_relay_log()`——以 applier 位点为准重建 receiver 位点 + 启用全新 relay log 文件，并清空 Retrieved_Gtid_Set 保证半截事务重新拉取。由 `relay_log_recovery=ON`（READ_ONLY）触发。

| 决策 | 实现 | 理由 / 代价 |
|---|---|---|
| 裁决方向 | binlog 单向裁决引擎 | 引擎 prepared 事务无自决能力；binlog 是唯一"已对外承诺"的记录 |
| 扫描范围 | 只扫**最后一个** binlog 文件 | 更早文件的 XID 早已收敛；重启时间不随历史增长 |
| XID 来源 | 复用 `thd->query_id`，无 cache | 去掉 5.7 全局哈希 + 锁，消除 cache 与 binlog 不一致的风险 |
| `m_valid_pos` 推进 | 仅事务边界 + 非 GTID/会话控制事件 | 宁可多截一点，也不留半截事务 |
| 引擎恢复失败 | 截断但**保留** in-use flag | 保住现场，下次重试 |
| relay log 不走 2PC 裁决 | `assert(!is_relay_log)` + 不设 in-use | 从库可重新索取，丢弃重拉比截断更简单可靠 |
| index 损坏 | 忽略 `binlog_error_action`，强制 abort | 会导致 purge 误删 |
| `sync_binlog` 默认 1 | 双 1 | `N>1` 会产生永久主从分歧，不可接受 |

## binlog_order_commits 参数

### 参数定义

全局变量，默认 `true`（sys_vars.cc:1741）。控制 BGC 第三阶段 commit stage 里引擎层 commit 调用的顺序——具体是 `ha_commit_low`（`finish_transaction_in_engines`）这一步。

**它不影响 flush 和 sync 两个阶段**，binlog 写入顺序和 fsync 永远是有序的。它只控制"最后让 InnoDB 真正 commit"这一步是有序还是各自为政。

### ON（默认）

Leader 把整个 group 拉进 commit stage，按事务写入 binlog 的同一顺序，逐个调用引擎 commit。GTID 也由 leader 有序批量写入 `gtid_executed`（binlog.cc:9075 注释：这样 `gtid_executed` 始终是一个连续区间，避免产生临时空洞、避免频繁加锁 `Gtid_set`）。

### OFF

不进 commit stage，各线程回到 `finish_commit` 里各自调用 `finish_transaction_in_engines` 自己做引擎 commit——顺序不保证。`finish_commit` 里 8728 的注释明确点破这个语义。

### Clone 强制有序

即使 `opt_binlog_order_commits` 为 false，只要 `Clone_handler::need_commit_order()` 为真，仍强制走有序分支（binlog.cc:9054）。Clone、一致性备份依赖"引擎 commit 顺序 == binlog 顺序"。

### 为什么只有 commit 阶段有开关

三阶段本质不同：

- **flush 没得选**：写 binlog 文件字节流就是复制顺序，必须串行，一旦乱序就是复制正确性灾难。
- **sync 没得选**：对整个 binlog 文件做一次 fsync，BGC 的核心收益就是把整组落盘合并成一次 fsync。让多个线程各自 fsync 反而破坏了 group commit 的意义。
- **commit 有得选**：引擎层 commit（改自己事务状态、释放自己的行锁、写自己的 commit 标记）各事务相互独立，天然可并行。正因可并行，才出现"保持有序（leader 串行）还是放开并行（各线程自己干）"的权衡。

默认选有序，因为有序保证 GTID 连续性、支持 Clone/备份一致性、且整组一次加锁比每线程各加锁开销更小。关闭仅在极高并发下省 stage 切换 / `LOCK_commit` 争用，榨微弱吞吐，代价大不推荐。

### 开启 binlog 后都走 BGC

`binlog_order_commits` 和"是否走 BGC"是两件不同的事。只要开启 binlog，所有事务提交都进入 `ordered_commit` 的三阶段流水线，这就是 BGC 机制本身。BGC 不是可开关的，它是 binlog 提交的唯一路径。`binlog_order_commits` 只决定 BGC 内部 commit stage 是否保持有序。

---

## Commit 阶段与 trx_no

### commit order 的核心载体：trx->no

`binlog_order_commits` 控制的"引擎 commit 顺序"，本质就是内存态的提交顺序，核心体现是 `trx->no`（serialisation number）的分配顺序。

> `trx->no` 的分配细节（`trx_add_to_serialisation_list` / `trx_serialisation_number_get` 的 purge queue 优化）、`trx->id` vs `trx->no` 的对比、commit 阶段的内存收尾（undo COMMITTED / 放锁 / MVCC / GTID 落盘）见 [`../../innodb/trx.md`](../../innodb/trx.md)「事务提交」。commit 阶段可并发的理由见下文「为什么只有 commit 阶段有开关」。

`trx->no` 在 `finish_transaction_in_engines` 里、serialisation mutex 保护下按调用先后单调递增分配。谁先被调用 commit，谁的 `trx->no` 就更小。

`binlog_order_commits` ON → `trx->no` 分配顺序 == binlog 写入顺序；OFF → 顺序不保证。Clone、一致性备份、`START TRANSACTION WITH CONSISTENT SNAPSHOT` 依赖"引擎提交序 == binlog 序"来取一致快照。

---

## Anonymous_Gtid

### 与普通 Gtid 的区别

`Gtid_log_event`（type=33）和 `Anonymous_gtid_log_event`（type=34）共用同一个 C++ 类 `Gtid_log_event`，区别在于 `spec.type`：

| | Gtid_log_event | Anonymous_gtid_log_event |
|---|---|---|
| type code | 33 | 34 |
| 携带 SID:GNO | 有（全局唯一事务标识） | 无（`spec.set_anonymous()`，sid/gno 清空） |
| sequence_number/last_committed | 有 | 有（完全相同） |
| IGNORABLE 标志 | 无 | 有（`LOG_EVENT_IGNORABLE_F`） |
| gtid_next 类型 | `ASSIGNED_GTID` | `ANONYMOUS_GTID` |
| 生成条件 | gtid_mode = ON | gtid_mode = OFF |

区分逻辑在 `Gtid_log_event` 构造函数（log_event.cc:12996-13016）：根据 `gtid_next.type` 决定 event 类型和标志。

### 为什么引入 Anonymous_Gtid

5.7 引入逻辑时钟 MTS 后，需要每个事务前有一个统一的事务起始事件来携带 `sequence_number` / `last_committed`。在 gtid_mode=ON 时，`Gtid_log_event` 天然承担这个角色。但 gtid_mode=OFF 时不生成 Gtid_log_event，MTS 就没有地方放依赖信息。

解决方案是引入 `Anonymous_gtid_log_event`——即使 gtid_mode=OFF，也为每个事务写一个 Anonymous_Gtid 事件。它不带 GTID（匿名），但携带 sequence_number / last_committed，让 MTS 在不开 GTID 的情况下也能运行。同时标记 `LOG_EVENT_IGNORABLE_F`，表示对开 GTID 的从库来说这个事件可忽略。

log_event.cc:2935 的注释直接点明了这个必要性。

### 多重作用

1. **MTS 事务边界标记**：coordinator 据此识别事务边界、提取 sequence_number/last_committed 做依赖调度
2. **匿名 GTID 所有权管理**：从库收到时设置 `gtid_next=ANONYMOUS` 并获取匿名所有权（sql_class.h:3565）
3. **兼容 5.6 master**：5.6 master（gtid_mode=off）不生成任何 GTID 事件，从库在执行 Query_log_event 时通过 `gtid_reacquire_ownership_if_anonymous` 设置匿名所有权（rpl_rli_pdb.h:749）
4. **GTID 模式平滑升级**：OFF → OFF_PERMISSIVE → ON_PERMISSIVE → ON 的过渡期，Anonymous_Gtid 与 Gtid 混合存在，MTS 始终能工作

---

## GTID 与 binlog 的持久化交互

> ⚠️ **命名勘误（8.0.39）**：`GTID_GROUP` / `ANONYMOUS_GROUP` / `GTID_NOT_YET_DETERMINED_GROUP` **已不存在**，源码注释记录了重命名（`BUG#18089914`）：现行 `enum_gtid_type` 是 `AUTOMATIC_GTID` / `ASSIGNED_GTID`（旧 `GTID_GROUP`）/ `ANONYMOUS_GTID` / `UNDEFINED_GTID` / `NOT_YET_DETERMINED_GTID` / `PRE_GENERATE_GTID`。另外 `fetch_gtids_from_binlog`、`update_gtids_purged` 均不存在（对应物见下文）。

### 三处持久化：表 + 两类事件

`mysql.gtid_executed` 是 **InnoDB 表**，主键 `(source_uuid, interval_start)`，一行存一个 GNO 区间。

```
        ┌─────────────────────── 事务提交路径 ───────────────────────┐
        │  ordered_commit FLUSH 阶段: generate_automatic_gtid()      │
        │  write_transaction(): Gtid_log_event 写进 binlog 文件       │
        │  COMMIT 阶段: update_commit_group() → executed_gtids       │
        └───────┬──────────────────────────────┬────────────────────┘
                │ (1) 事务级                    │ (2) 落表：8.0.23+ 由
                v                               v     InnoDB 后台批量写
   ┌────────────────────────┐          ┌──────────────────────────┐
   │ binlog: Gtid_log_event │          │ mysql.gtid_executed 表    │
   │ / Anonymous_gtid_event │          │ (InnoDB, 区间行)          │
   └───────────┬────────────┘          └────────────┬─────────────┘
               │ rotate 时把"本文件之前的集合"       │ 启动时 read_gtid_
               │ 快照出来                            │ executed_from_table
               v                                     │
   ┌──────────────────────────────┐ <---- 合并 ----→┘
   │ binlog: Previous_gtids_log_event（每个文件头部）│
   └──────────────────────────────┘
```

- **(1) 事务级** `Gtid_log_event`：复制/恢复时"逐事务"重放的凭证
- **(2) 文件级** `Previous_gtids_log_event`：打开该文件时服务器已拥有的 GTID 集合快照，用于启动时 O(1~2 文件) 收敛
- **(3) 表级** `mysql.gtid_executed`：**与 binlog 生命周期解耦**的权威副本——binlog 会被 purge，表不会

**为什么必须双写**：只有表则无事件内容无法复制；只有 binlog 则 purge 后重启 GTID 永久丢失。

### 落表：8.0.23+ 已交给 InnoDB 后台

```cpp
int Gtid_state::save(THD *thd) {
  assert(thd->owned_gtid.sidno > 0);
  int ret = gtid_table_persistor->save(thd, &thd->owned_gtid);
  if (1 == ret) {
    /* Gtid table is not ready to be used, so failed to open it. Ignore. */
    thd->clear_error();
  } else if (-1 == ret) return -1;
  return 0;
}
```

- `ret == 1`（表还没准备好）**被静默容忍**——DDL 初始化/`mysql.gtid_executed` 尚未创建时不能让事务失败
- ★ 8.0.39 里 `Gtid_table_persistor::save(THD*, const Gtid*)` **几乎不写行**（只有 debug 分支写），真正落表的是 `thd->request_persist_gtid_by_se()` → InnoDB `Clone_persist_gtid` 后台累积后批量 `save(&set, compress=false)`。所以 `handler.cc` 里有 `if (thd->owned_gtid.sidno > 0 && !thd->se_persists_gtid()) error = gtid_state->save(thd);` —— **InnoDB 事务不再同步插表**，去掉了每事务一次表 IO

**五个触发时机**：

| 时机 | 调用点 |
|---|---|
| rotate | `new_file_impl()` → `save_gtids_of_last_binlog_into_table()` |
| shutdown | `if (opt_bin_log) gtid_state->save_gtids_of_last_binlog_into_table()`（失败只告警） |
| 启动收敛 | `mysqld_main()`：`gtid_state->save(&gtids_in_binlog_not_in_table)` |
| 压缩 | `Gtid_table_persistor::compress()` ← `compress_gtid_table` 线程 |
| `SET @@GLOBAL.GTID_PURGED` | `Gtid_state::add_lost_gtids()` |

rotate 的核心是**差集公式**（`executed_gtids - previous_gtids_logged - gtids_only_in_table`）：

```cpp
  /* Use local Sid_map, so that we don't need a lock while inserting into table. */
  Sid_map sid_map(nullptr);
  Gtid_set logged_gtids_last_binlog(&sid_map, nullptr);
  global_sid_lock->wrlock();
  ret = (logged_gtids_last_binlog.add_gtid_set(&executed_gtids) != RETURN_STATUS_OK);
  if (!ret) {
    logged_gtids_last_binlog.remove_gtid_set(&previous_gtids_logged);
    logged_gtids_last_binlog.remove_gtid_set(&gtids_only_in_table);
    if (!logged_gtids_last_binlog.is_empty()) {
      /* Prepare previous_gtids_logged for next binlog always. Need it
      even during shutdown to synchronize with innodb GTID persister. */
      if (previous_gtids_logged.add_gtid_set(&logged_gtids_last_binlog))
        ret = ER_OOM_SAVE_GTIDS;
      global_sid_lock->unlock();                       // ★ 先放锁
      if (!ret) if (save(&logged_gtids_last_binlog))   //   再做表 IO
        ret = ER_RPL_GTID_TABLE_CANNOT_OPEN;
    } else global_sid_lock->unlock();
  } else global_sid_lock->unlock();
```

★ 用**局部 `Sid_map`** 拷出集合、释放 `global_sid_lock` 后才做表 IO——避免持全局锁做 IO。

### `Gtid_state` 内存态

```cpp
  Gtid_set lost_gtids;             // 曾被 purge 过的 GTID ⊆ executed_gtids
  Gtid_set executed_gtids;         // @@GLOBAL.GTID_EXECUTED
  Gtid_set gtids_only_in_table;    // 只在表里、不在任何现存 binlog 里
  Gtid_set previous_gtids_logged;  // 已写入 PREVIOUS_GTIDS 的部分
  Owned_gtids owned_gtids;         // 正在被某线程持有（gno → owner thread_id）
  rpl_sidno server_sidno;          // 本实例 server_uuid 的 sidno
```

```
executed_gtids = gtids_only_in_table ∪ (现存 binlog 中的 GTID)
lost_gtids     = gtids_only_in_table ∪ purged_gtids_from_binlog
owned_gtids ∩ executed_gtids = ∅   （owned 只在 in-flight 期间）
```

★ `gtids_only_in_table` 是**双写去重的枢纽**：rotate 算"要补进表的新 GTID"和 `open_binlog` 生成 PREVIOUS_GTIDS 时都必须减掉它。

**为什么用 sidno 而不是 uuid**：`Sid_map` 维护 `_sidno_to_sid`（数组，O(1)）与 `_sid_to_sidno`（hash）。GTID 集合内部按 sidno 做**稠密数组下标**（`Gtid_set::m_intervals[sidno]`），集合运算全是整数操作；`Mutex_cond_array sid_locks` 也按 sidno 索引，`WAIT_FOR_EXECUTED_GTID_SET` 就等在这个 per-sidno 条件变量上。**sidno 只增不复用**（`add_sid` 分配 `max_sidno+1`），故 `ensure_sidno` 必须处处调用。

### GTID 分配：FLUSH 阶段，先于事件落盘

```cpp
  THD *first_seen = fetch_and_process_flush_stage_queue();
  assign_automatic_gtids_to_flush_group(first_seen);      // ★ 先给整组分配 GTID
  for (THD *head = first_seen; head; head = head->next_to_commit) {
    const auto [error, flushed_bytes] = flush_thread_caches(head);   // 再刷 cache
  }
```

顺序是确定的——`write_transaction()` 写 `Gtid_log_event` 时直接读 `thd->owned_gtid`，所以分配必须前置。

```cpp
  if (global_gtid_mode.get() >= Gtid_mode::ON_PERMISSIVE) {
    Gtid automatic_gtid = {specified_sidno, specified_gno};
    if (automatic_gtid.sidno == 0) automatic_gtid.sidno = get_server_sidno();
    if (automatic_gtid.gno == 0) {
      automatic_gtid.gno = get_automatic_gno(automatic_gtid.sidno);
      if (automatic_gtid.sidno == get_server_sidno() && automatic_gtid.gno != -1)
        next_free_gno = automatic_gtid.gno + 1;          // ★ 组提交优化：从上次+1 找空洞
    }
    if (automatic_gtid.gno == -1 || acquire_ownership(thd, automatic_gtid))
      ret = RETURN_STATUS_REPORTED_ERROR;
  } else {
    // GTID_MODE = OFF / OFF_PERMISSIVE：只标记匿名事务
    thd->owned_gtid.sidno = THD::OWNED_SIDNO_ANONYMOUS;
    thd->owned_gtid.gno = 0;
    acquire_anonymous_ownership();
  }
```

- `GTID_MODE = OFF/OFF_PERMISSIVE` 时**不分配 GTID**，后续写出 `Anonymous_gtid_log_event`
- `next_free_gno` 是组提交下的优化：避免整组都从 1 扫；回滚时会回退，防止永久留洞
- ★ 分配只是 `acquire_ownership()`（进 `owned_gtids`），**此时还没进 `executed_gtids`**——要等 COMMIT 阶段的 `update_commit_group()`。这正是"**崩溃后必须扫 binlog 补表**"的根因

### `Previous_gtids_log_event` 的生成

rotate → `new_file_impl()` → `open_binlog()`；注意**先补表再开新文件**。

```cpp
    if (!is_relay_log) {
      const Gtid_set *executed_gtids = gtid_state->get_executed_gtids();
      const Gtid_set *gtids_only_in_table = gtid_state->get_gtids_only_in_table();
      /* logged_gtids_binlog = executed_gtids - gtids_only_in_table */
      if (logged_gtids_binlog.add_gtid_set(executed_gtids) != RETURN_STATUS_OK) goto err;
      logged_gtids_binlog.remove_gtid_set(gtids_only_in_table);
    }
    Previous_gtids_log_event prev_gtids_ev(previous_logged_gtids);
    if (write_event_to_binlog(&prev_gtids_ev)) goto err;
```

★ 来源是**内存态差集** `executed_gtids - gtids_only_in_table`，不是"上一个文件的 PREVIOUS_GTIDS + 本文件 GTID"。relay log 走 `previous_gtid_set_relaylog`（即 Retrieved_Gtid_Set）。`current_thd == nullptr`（启动早期）时不写，由 `mysqld_main()` 在 GTID 初始化后补写（源码注释直言是历史包袱）。

**从库为什么靠它**：`read_gtids_from_binlog()` 读到 `PREVIOUS_GTIDS_LOG_EVENT` 就 `add_to_set(all_gtids)` 并停止回溯（`can_stop_reading = true`）——读任一个文件的头部就知道"该文件起点之前源端已有什么"，与自己的 `gtid_executed` 做差集即"我缺哪些"。

### 崩溃后的 GTID 收敛

```
  mysqld_main()
     ├─ gtid_state->init()                     // server_uuid → sidno, next_free_gno=1
     ├─ gtid_state->read_gtid_executed_from_table()
     │      └─ Gtid_table_persistor::fetch_gtids(&executed_gtids)   // 全表扫描
     └─ if (opt_bin_log)
          mysql_bin_log.init_gtid_sets(&gtids_in_binlog, &purged_gtids_from_binlog, ...)
             ├─ 反向遍历 index：找第一个含 PREVIOUS_GTIDS 的文件 → all_gtids
             └─ 正向遍历 index：找第一个"PREVIOUS_GTIDS + GTID"的文件 → lost_gtids
          │
          ├─ if (!gtids_in_binlog.is_subset(executed_gtids))    ← ★ 补表
          │     gtids_in_binlog_not_in_table = gtids_in_binlog - executed_gtids
          │     gtid_state->save(&gtids_in_binlog_not_in_table); executed_gtids += ...
          ├─ gtids_only_in_table = executed_gtids - gtids_in_binlog
          ├─ lost_gtids = gtids_only_in_table + purged_gtids_from_binlog
          └─ write_event_to_binlog_and_sync(Previous_gtids_log_event)
```

注释列明了四种必须补表的场景：① upgrade ② 从备份 provision 的从库 ③ 上次 `RESET MASTER` 后未 rotate 就崩溃 ④ **最后一个 binlog 的 GTID 未落表就崩溃**（最常见）。

`binlog_gtid_simple_recovery`（默认 ON，只读）开启时**最多读 2 个 binlog**；代价是若 binlog 序列不规则（5.6 升上来、`RESET MASTER` 后遗留空文件），算出的 `GTID_PURGED` 可能**偏小且不可逆**。

### purge 与 GTID 的约束

`MYSQL_BIN_LOG::purge_logs()` 在更新 index 之后、删文件之前重算 `lost_gtids`：

```cpp
  if (!is_relay_log) {
    global_sid_lock->wrlock();
    error = init_gtid_sets(nullptr,
                           const_cast<Gtid_set *>(gtid_state->get_lost_gtids()),
                           opt_source_verify_checksum, false, nullptr, nullptr);
    global_sid_lock->unlock();
  }
```

★ **purge 本身不校验 GTID 是否已落表**，它依赖一条**前置不变式**：rotate 时已把上一个文件的 GTID 全部写进表，且 purge 只能从 index 头部**顺序删除**不能跳删 ⇒ 被删文件的 GTID 必已落表。

因此 rotate 失败（`ER_RPL_GTID_TABLE_CANNOT_OPEN`）时 `new_file_impl` 会**拒绝轮换**并在文件满时关闭 binlog——宁可停写也不能让未落表的 GTID 被后续 purge 带走。违反的后果：重启后该 GTID 在表和 binlog 里都没有 ⇒ 从 `GTID_EXECUTED` 消失 ⇒ 从库重连时被判"未执行"而**重放，造成重复执行/主键冲突**。

`RESET MASTER` 是另一条路径：`reset_logs()` → `gtid_state->clear(thd)` 清空四个集合 + `gtid_table_persistor->reset(thd)`（`delete_all` 而非 truncate，因为 DDL 非事务）。

| 决策 | 选择 | 代价 / 收益 |
|---|---|---|
| GTID 双写 | 表是 purge 后仍存活的权威副本 | 每事务多一条持久化路径；换 purge/崩溃安全 |
| 落表延迟化（8.0.23+） | 由 InnoDB `Clone_persist_gtid` 后台批量写 | 去掉每事务表 IO；换来 SE/非 SE 写路径分离 |
| 表不可用静默容忍 | `ret==1` 清错继续 | 启动/DDL 期不误伤；靠启动补表兜底 |
| rotate 用内存差集 | `executed - previous_logged - only_in_table` | O(内存集合)；强依赖三集合严格维护 |
| FLUSH 分配、COMMIT 才入 `executed_gtids` | leader 批量分配 | 少锁竞争；产生"已落 binlog 未入内存"窗口 ⇒ 必须启动扫描补表 |
| purge 不校验落表 | 依赖"rotate 必先落表"不变式 | purge 路径简单；rotate 失败必须硬失败 |

## Event 格式与 mysqlbinlog 解读

### Event 边界

mysqlbinlog 输出中，每个 event 由三部分组成：

```
# at <offset>                    ← event 起始：字节偏移
#<时间> server id <id>  end_log_pos <pos> CRC32 <crc>  <EventType> <详情>  ← event 头
<event 内容>                     ← event 体（SQL/SET 语句等）
```

`# at` 就是 event 的分隔符。遇到下一个 `# at` 意味着上一个 event 结束。两个 `# at` 之间的所有内容属于同一个 event。

### 关键字段

| 字段 | 含义 |
|------|------|
| `# at 157` | event 在 binlog 文件中的起始字节偏移 |
| `260703 15:17:04` | event 的时间戳 |
| `server id 1` | 生成该 event 的服务器 id |
| `end_log_pos 236` | event 结束位置 = 下一个 event 的起始位置（236-157=79 字节是 event 长度） |
| `CRC32 0x...` | 校验和 |
| `GTID` / `Query` / `Start` | event 类型 |
| `last_committed=0 sequence_number=1` | MTS 逻辑时钟依赖信息（仅 GTID event 有） |

### mysqlbinlog 显示名与内部类型对照

| mysqlbinlog 显示 | 内部 event 类型 | type code |
|---|---|---|
| `Start:` | `FORMAT_DESCRIPTION_EVENT` | 15 |
| `Previous-GTIDs` | `PREVIOUS_GTIDS_LOG_EVENT` | 35 |
| `GTID` | `GTID_LOG_EVENT` | 33 |
| `Anonymous_GTID` | `ANONYMOUS_GTID_LOG_EVENT` | 34 |
| `Query` | `QUERY_EVENT` | 2 |
| `Table_map` | `TABLE_MAP_EVENT` | 19 |
| `Update_rows` / `Write_rows` / `Delete_rows` | `ROWS_EVENT` | 23/24/25 |
| `Xid` | `XID_EVENT` | 16 |
| `Rotate` | `ROTATE_EVENT` | 4 |

### 常用参数

```bash
# 解码行事件，显示每个字段的具体值（调试最有用）
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog_file

# 显示十六进制 dump（调试 event 头部）
mysqlbinlog --hexdump binlog_file
```

### 一个事务的 event 组成

每个事务在 binlog 中由以下 event 序列组成：

```
Gtid_log_event (或 Anonymous_gtid_log_event)   ← 携带 GTID + sequence_number/last_committed
Query_log_event ("BEGIN")                       ← 事务开始（可选，autocommit DDL 可能没有）
... 行事件或语句事件 ...
Xid_log_event 或 Query_log_event ("COMMIT")    ← 事务结束
```

---

## 事件二进制布局（字节级）

> 上面「Event 格式」节讲的是 mysqlbinlog 怎么读、字段叫什么；本节给**字节级布局**——每个字段在事件里的偏移、长度、字节序。所有多字节整数均为**小端**（`int2store/int4store/int8store`）。

### 通用头 19 字节

```
偏移(相对事件首字节)  长度  字段                      源码常量
  0                   4    timestamp (when)         —（4 字节无符号秒）
  4                   1    event_type               EVENT_TYPE_OFFSET
  5                   4    server_id                SERVER_ID_OFFSET
  9                   4    event_size               EVENT_LEN_OFFSET
 13                   4    end_log_pos (log_pos)    LOG_POS_OFFSET
 17                   2    flags                    FLAGS_OFFSET
 19                  ...    post-header + body [+ 4 字节 crc32]
```

- `LOG_EVENT_HEADER_LEN 19U`（`LOG_EVENT_MINIMAL_HEADER_LEN` 同为 19：FD 与 ROTATE 强制只用这 19 字节）
- ★ `event_size` = 19 + post-header + body **+ checksum**，**包含**尾部 4 字节 crc32
- `end_log_pos` = 本事件结束后的文件偏移（同样含 checksum）；在 relay log 中保留的是**主库 binlog** 的偏移
- flags 低位：`LOG_EVENT_BINLOG_IN_USE_F 0x1`、`LOG_EVENT_IGNORABLE_F 0x80`
- 三段式：通用头（固定 19）→ post-header（**长度不固定，由 FD 的数组给出**）→ body（变长）→ [crc32 4]

### FORMAT_DESCRIPTION_EVENT（type 15）

`FORMAT_DESCRIPTION_HEADER_LEN = (2+50+4) + 1 + 41 = 98`

```
偏移(相对事件首字节)  长度  字段                       源码常量
 19                   2    binlog_version (=4)        ST_BINLOG_VER_OFFSET + 19
 21                  50    server_version[50],0补齐    ST_SERVER_VER_OFFSET(2)
 71                   4    create_timestamp           ST_CREATED_OFFSET(52)
 75                   1    header_length (=19)        ST_COMMON_HEADER_LEN_OFFSET(56)
 76                  41    post_header_len[41]        （下标 i 对应 event_type = i+1）
117                   1    checksum alg desc (A)
118                   4    crc32 (V)
---  事件总长 = 19+98+1+4 = 122 字节
```

**post-header 长度数组**（`post_header_len[event_type - 1]`）：

| event_type | 值 | 常量 |
|---|---|---|
| QUERY_EVENT(2) | 13 | `QUERY_HEADER_LEN = 4+4+1+2+2` |
| ROTATE_EVENT(4) | 8 | `ROTATE_HEADER_LEN` |
| FORMAT_DESCRIPTION_EVENT(15) | 98 | `FORMAT_DESCRIPTION_HEADER_LEN` |
| XID_EVENT(16) | 0 | `XID_HEADER_LEN` |
| TABLE_MAP_EVENT(19) | 8 | `TABLE_MAP_HEADER_LEN` |
| WRITE/UPDATE/DELETE_ROWS_V1(23/24/25) | 8 | `ROWS_HEADER_LEN_V1` |
| INCIDENT_EVENT(26) | 2 | `INCIDENT_HEADER_LEN` |
| ROWS_QUERY_LOG_EVENT(29) | 0 | `IGNORABLE_HEADER_LEN` |
| WRITE/UPDATE/DELETE_ROWS(30/31/32) | 10 | `ROWS_HEADER_LEN_V2` |
| **GTID_LOG_EVENT(33) / ANONYMOUS(34)** | **42** | `Gtid_event::POST_HEADER_LENGTH` |
| PREVIOUS_GTIDS_LOG_EVENT(35) | 0 | `IGNORABLE_HEADER_LEN` |
| PARTIAL_UPDATE_ROWS_EVENT(39) | 10 | `ROWS_HEADER_LEN_V2` |
| TRANSACTION_PAYLOAD_EVENT(40) | 声明为 40 | `TRANSACTION_PAYLOAD_HEADER_LEN = 0` |

**为什么 FD 必须是第一个**：解析任何事件都要先知道"post-header 有多长"，而这个长度本身写在 FD 里——鸡生蛋。解法是 FD（及可能先到的 ROTATE）**保证只用 19 字节固定头**，且 `binlog_version` 位于永不变动的偏移；读出 FD 后拿到 `common_header_len` 与 41 项 `post_header_len[]`，后续事件才能定位边界。FD 还给出 `server_version`（决定有无 checksum）与尾部 1 字节校验算法描述符。

### Gtid_log_event(33) / Anonymous_gtid_log_event(34)

post-header **固定 42 字节**：

```
post-header（偏移相对 19）
  0   1   GTID flags            FLAG_MAY_HAVE_SBR = 1（是否可能含 SBR）
  1  16   uuid / SID            Uuid::BYTE_LENGTH
 17   8   gno (int64)           GTID >=1；Anonymous 必须为 0
 25   1   lt_type (=2)          LOGICAL_TIMESTAMP_TYPECODE
 26   8   last_committed        int64
 34   8   sequence_number       int64
body（变长，5.7/8.0 逐步追加）
 42   7   immediate_commit_timestamp（bit55 为标志位，1<<55）
 49   7   original_commit_timestamp     ← 仅当 bit55 置位时存在
 --  1~9  transaction_length（net_store_length 变长编码）
 --   4   immediate_server_version（bit31 为标志位，1<<31）
 --   4   original_server_version       ← 仅当 bit31 置位时存在
 --   8   commit_group_ticket           ← 可选（BgcTicket）
```

★ 时间戳是 **7 字节**不是 8 字节——最高位（bit55）兼作"是否跟随 original 时间戳"的标志；`server_version` 同理用 bit31。这是"省字节 + 兼容旧版本"的取舍。

### Query_log_event(2)

post-header 13 字节（5.0 之前为 `QUERY_HEADER_MINIMAL_LEN = 11`，无 status_vars_length）：

```
  0   4   slave_proxy_id        Q_THREAD_ID_OFFSET（源码字段名仍保留 slave 字样）
  4   4   execution_time        Q_EXEC_TIME_OFFSET
  8   1   schema_length         Q_DB_LEN_OFFSET
  9   2   error_code            Q_ERR_CODE_OFFSET
 11   2   status_vars_length    Q_STATUS_VARS_LEN_OFFSET
 13   n   status_vars           Q_DATA_OFFSET = QUERY_HEADER_LEN
 13+n sl  schema（schema_length 字节 + 1 字节 0x00）
 ...  余  query（到事件尾，不含 crc）
```

`status_vars` 是 (1 字节 code + 值) 序列，按 code 递增写（`Q_FLAGS2_CODE`、`Q_SQL_MODE_CODE`、`Q_CATALOG_NZ_CODE`、`Q_AUTO_INCREMENT`、`Q_CHARSET_CODE`、`Q_TIME_ZONE_CODE`、`Q_MICROSECONDS`、`Q_XID`…）。

### Table_map_log_event(19)

```
post-header（8）
  0   6   table_id              TM_MAPID_OFFSET
  6   2   flags                 TM_FLAGS_OFFSET
body
  var      schema_length（packed） + schema + 0x00
  var      table_length（packed） + table + 0x00
  var      column_count（packed）
  var cc   column_types[column_count]（每列 1 字节）
  var      metadata_length（packed） + metadata
  var      null_bits（(cc+7)/8 字节）
  var  余  optional_metadata（binlog_row_metadata，TLV：(type,length,value) 序列）
```

optional_metadata 类型含 `SIGNEDNESS=1`、`DEFAULT_CHARSET=2`、`COLUMN_CHARSET=3`、`COLUMN_NAME=4`、`SET_STR_VALUE=5`、`ENUM_STR_VALUE=6`、`GEOMETRY_TYPE=7`、`SIMPLE_PRIMARY_KEY=8`、`PRIMARY_KEY_WITH_PREFIX=9`、`ENUM_AND_SET_DEFAULT_CHARSET=10`、`ENUM_AND_SET_COLUMN_CHARSET=11`、`COLUMN_VISIBILITY=12`。

### Rows_log_event（WRITE 30 / UPDATE 31 / DELETE 32 / PARTIAL_UPDATE 39）

```
post-header
  0   6   table_id              ROWS_MAPID_OFFSET
  6   2   flags                 ROWS_FLAGS_OFFSET
  8   2   extra_data_len        ROWS_VHLEN_OFFSET   ← 仅 V2（post_header_len==10），含自身 2 字节
 10   …   extra_data（extra_data_len-2 字节，(typecode,值) 块；NDB=0、PART=1）
body
  var      columns_count（packed，m_width）
  var      columns-present bitmap1（BI，ceil(n/8) 字节）
  var      columns-present bitmap2（AI）← 仅 UPDATE / UPDATE_V1 / PARTIAL_UPDATE
  var      行数据（重复到事件尾）
```

- 每行 = `null bitmap（ceil(该镜像列数/8) 字节）` + 逐列值（**NULL 列不占值空间**）
- UPDATE：先按 bitmap1 解 BI，再按 bitmap2 解 AI
- `PARTIAL_UPDATE_ROWS_EVENT` 的 AI 前另有 `value_options（packed）+ partial bits（ceil(json列数/8)）`，用于 `binlog_row_value_options=PARTIAL_JSON`

### 其余几个关键事件

**Xid_log_event(16)**：post-header 为 0，body 就是 **8 字节 xid**（小端），随后 crc32。它是内部 2PC 的"commit 记录"。

**Rotate_log_event(4)**：

```
post-header（8，已冻结）
  0   8   first_log_file_position   R_POS_OFFSET
body
  8   余   new_log_ident（文件名，无长度前缀）      R_IDENT_OFFSET = 8
```

不需要长度字段：post-header 之后到 `event_size - checksum` 的**全部剩余字节**即文件名。

**Previous_gtids_log_event(35)**：post-header = 0，body 是 `Gtid_set::encode()` 的输出：

```
  0   8   n_sids（uint8korr）
  8   ─   重复 n_sids 次：
          uuid(16) + n_intervals(8) + 重复 n_intervals 次：start(8) + end(8)
                                      （int8store 定长，非 packed）
```

**Incident_log_event(26)**：post-header 2 字节 `incident_number`，body = `message_length(1) + message`（≤255）。

### 校验和

- `BINLOG_CHECKSUM_LEN 4`，位置是事件的**最后 4 字节**，且**在 `event_size` 之内**
- 覆盖范围：`crc32(event_buf[0 .. event_size-4))` —— **含 19 字节通用头在内**的除自身外全部字节；边写边算，算法固定 zlib crc32
- **FD 特例**：尾部恒为 `(A) 1 字节算法描述符 + (V) 4 字节 crc`，**即使 `binlog_checksum=OFF` 也占这 5 字节**（A=0 表示本文件其余事件无 checksum）

### 压缩事件：Transaction_payload_event(40)

`binlog_transaction_compression` 开启后，一个事务的所有事件被打成一个 payload：

```
 19   var   payload data header：(typecode, length, value) 三元组，全部 net_store_length 变长编码
            typecode: 1 = OTW_PAYLOAD_SIZE_FIELD
                      2 = OTW_PAYLOAD_COMPRESSION_TYPE_FIELD
                      3 = OTW_PAYLOAD_UNCOMPRESSED_SIZE_FIELD
                      0 = OTW_PAYLOAD_HEADER_END_MARK（仅 1 字节）
  …   var   payload（压缩后的完整事件字节流，zstd frame）
```

compression type：`ZSTD = 0` / `NONE = 255`；`max_log_event_size = 1GB`。注意 FD 的 post-header 数组里该位填的是常量 40 而非 0，但解码器直接从 `common_header_len` 之后开始读三元组，**不使用** post_header_len，因此不影响编解码。

### 加密：每文件一把数据密钥

```
  0    4   ENCRYPTION_MAGIC = 0xFD62696E   （对比 BINLOG_MAGIC = 0xFE62696E）
  4    1   header version (=1)
  5    ─   TLV 字段（总长 HEADER_SIZE = 512，尾部补 0）
          type=1 KEY_ID
          type=2 ENCRYPTED_FILE_PASSWORD（32 字节值）
          type=3 IV_FOR_FILE_PASSWORD（16 字节值）
512   ─  加密数据区（含被加密的 BINLOG_MAGIC），整文件要么全加密要么全不加密
```

**两级密钥**：`file password`（每文件 32 字节随机数）用 keyring 中的 *Replication Encryption Key* 以 **AES-256-CBC** 加密后存文件头；真正加密数据区用的是由 file password 生成的密钥，算法 **AES-256-CTR**。所以**每个 binlog/relay log 文件一把独立数据密钥**，轮换主密钥只需重新加密各文件头里的密码。

## Row Image 记录过程

> 基于 MySQL 8.0.39 源码，ROW format binlog 中 DML 操作如何转化为行镜像（before-image/after-image）写入 binlog 的完整流程。

### DML 触发入口

所有 DML 操作在存储引擎执行成功后，通过 handler 层的 `ha_write_row` / `ha_update_row` / `ha_delete_row` 调用 `binlog_log_row()` 进入 binlog 记录流程：

| DML 类型 | 前镜像 (BI) | 后镜像 (AI) | 入口 |
|----------|-------------|-------------|------|
| INSERT | 无 | 新插入行 | `ha_write_row` → `binlog_log_row(table, nullptr, buf, ...)` |
| UPDATE | 修改前行 | 修改后行 | `ha_update_row` → `binlog_log_row(table, old_data, new_data, ...)` |
| DELETE | 被删除行 | 无 | `ha_delete_row` → `binlog_log_row(table, buf, nullptr, ...)` |

### 核心调度：binlog_log_row()

`binlog_log_row()`（handler.cc:7860）完成以下工作：

1. 检查当前表是否需要行格式 binlog（`check_table_binlog_row_based`）
2. 如果启用了 `transaction_write_set_extraction`，计算主键等价（PKE）用于写集合提取
3. 若是语句的第一行，先写入 `Table_map_event`（`write_locked_table_maps`）
4. 调用对应的 `Log_func` 函数指针（`binlog_write_row` / `binlog_update_row` / `binlog_delete_row`）

### 列筛选：binlog_row_image 参数

在打包前，根据 `binlog_row_image` 参数决定哪些列参与记录：

| 取值 | read_set（前镜像） | write_set（后镜像） |
|------|-------------------|-------------------|
| **FULL**（默认） | 所有列 | 所有列 |
| **NOBLOB** | PK 列 + 非 BLOB 列 | 非 BLOB 列 |
| **MINIMAL** | 仅主键列 | 仅被修改的列 |

`mark_columns_per_binlog_row_image()`（table.cc:5682）在语句准备阶段设置 bitmap；`binlog_prepare_row_images()`（binlog.cc:11249）在实际打包前进一步调整 read_set。若无主键，read_set 全部置位（无法用主键唯一标识行）。

### 行数据打包：pack_row()

`pack_row()`（rpl_record.cc:232）将内部记录格式转换为 binlog 传输格式：

```
+-----------+----------+----------+     +----------+
| null_bits | column_1 | column_2 | ... | column_N |
+-----------+----------+----------+     +----------+
```

`null_bits` 占 `ceil(N/8)` 字节，N 为 image 中包含的列数。对于 `PARTIAL_UPDATE_ROWS_EVENT`（JSON 部分更新优化），布局还包含 `value_options` 和 `partial_bits`。

### 行镜像类型

```cpp
enum class enum_row_image_type { WRITE_AI, UPDATE_BI, UPDATE_AI, DELETE_BI };
```

- `WRITE_AI`：INSERT 后镜像，用 `table->write_set` 打包
- `UPDATE_BI`：UPDATE 前镜像，用 `table->read_set` 打包
- `UPDATE_AI`：UPDATE 后镜像，用 `table->write_set` 打包，支持 JSON 部分更新
- `DELETE_BI`：DELETE 前镜像，用 `table->read_set` 打包

### UPDATE 完整调用链（最复杂的情况）

```
1. handler::ha_update_row(old_data, new_data)
   └─ 2. binlog_log_row(table, old_data, new_data, Update_rows_log_event::binlog_row_logging_function)
       ├─ 3. check_table_binlog_row_based()
       ├─ 4. add_pke() — 写集合提取
       ├─ 5. write_locked_table_maps() — 写入 Table_map_event
       └─ 6. THD::binlog_update_row(table, is_trans, before_record, after_record, extra_row_info)
           ├─ 7. binlog_prepare_row_images(thd, table) — 调整 read_set
           ├─ 8. pack_row(table, table->read_set, before_row, before_record, UPDATE_BI)
           ├─ 9. pack_row(table, table->write_set, after_row, after_record, UPDATE_AI, value_options)
           ├─ 10. binlog_prepare_pending_rows_event<Update_rows_log_event>() — 获取/创建事件
           ├─ 11. ev->add_row_data(before_row, before_size)
           ├─ 12. ev->add_row_data(after_row, after_size)
           └─ 13. 恢复原 read_set/write_set
```

### Rows_event 二进制格式

```
Post-Header:
  table_id     (6 bytes)
  flags        (2 bytes)

Body:
  width                    (packed integer) — 表的列数
  cols                     (bitfield) — 前镜像列位图
  cols_ai                  (bitfield, UPDATE only) — 后镜像列位图
  extra_row_info           (optional)
  row data: 序列化的行数据
    对于每行: null_bits + column_values
    UPDATE 每行包含 BI 和 AI 两部分
```

---

## binlog 读侧：dump 线程（Binlog_sender）

> 前面都是写侧。本节讲 binlog 怎么被读出来发给从库——8.0 已完成 slave→replica 改名，但 dump 线程核心类仍叫 `Binlog_sender`。

> ⚠️ **命名勘误（8.0.39）**：`send_fake_rotate_event` → 实为 `fake_rotate_event`；`get_gtid_set_from_binlog` → 实为 `MYSQL_BIN_LOG::find_first_log_not_in_gtid_set()` + `read_gtids_from_binlog()`；`wait_for_update_binlog` → 实为 `MYSQL_BIN_LOG::wait_for_update()`；`Binlog_sender::send_event` → 实为 `send_packet()` / `send_packet_and_flush()`；**`LOG_EVENT_SKIP_REPLICATION_F` 全仓库不存在**，dump 线程也不做 server_id 过滤。

### 两个命令入口与僵尸连接踢除

`COM_BINLOG_DUMP`(0x12) 与 `COM_BINLOG_DUMP_GTID`(0x1E) 都落在 `sql/rpl_source.cc`：

- `com_binlog_dump()`：报文 `uint4 pos | uint2 flags | uint4 server_id | filename`。★ **pos 只有 4 字节**（注释："4 bytes is too little ... fixed in the new protocol"），以 `nullptr` 作为 exclude_gtid 调用 `mysql_binlog_send()`
- `com_binlog_dump_gtid()`：pos 扩到 8 字节，并构造 `Gtid_set slave_gtid_executed` 传入 ⇒ `m_using_gtid_protocol = (exclude_gtids != nullptr)`

两者都先 `check_global_access(REPL_SLAVE_ACL)`、申请 dump 资源，然后 **`kill_zombie_dump_threads()`**：用 `Find_zombie_dump_thread` 扫描所有 dump 连接，按 `@replica_uuid`（5.6+）匹配，无 UUID 时退化成 `server_id` 相等（5.5 老从库）。命中后**不调 `kill_one_thread()`**（避免二次遍历），而是 `duplicate_slave_id = true; awake(THD::KILL_QUERY)`，让新从库收到"已有同 UUID/server_id 的副本连接"**不要再重连**。

### 主循环

```
        com_binlog_dump / com_binlog_dump_gtid
          ACL → dump 资源 → kill_zombie_dump_threads
                    │
        mysql_binlog_send() → Binlog_sender::run()
   [init] register_log_info / check_start_file / transmit_start hook
                    │
   ┌────────────────▼───────────────────────────────────────┐
   │ run() 外层「文件循环」 while(!error && !killed)         │
   │   fake_rotate_event()      ← 假 ROTATE（总是发）        │
   │   reader.open(log_file)    ← magic + FD 校验            │
   │   send_binlog(reader, start_pos) ──┐                    │
   │        ┌───────────────────────────┘                    │
   │        │  send_format_description_event()               │
   │        │  has_previous_gtid_log_event()                 │
   │        │  ┌─ 内层「读-等循环」──────────────────────┐   │
   │        │  │ get_binlog_end_pos()                      │   │
   │        │  │   ├ wait_new_events()（update_cond）      │   │
   │        │  │   └ 冷文件 → end_pos = 0                  │   │
   │        │  │ send_events() → read / skip / send_packet │   │
   │        │  └───────────────────────────────────────────┘   │
   │   LOCK_index → find_next_log() → start_pos = 4            │
   └────────────────┬───────────────────────────────────────┘
              cleanup()：transmit_stop hook / unregister_log_info / my_eof
```

```cpp
void Binlog_sender::run() {
  init();
  reader.allocator()->set_sender(this);      // ★ 事件字节直接读进 m_packet，零拷贝
  while (!has_error() && !m_thd->killed) {
    /* 即使上一个 binlog 里已有真 ROTATE，这里仍无条件先发一个假 ROTATE */
    if (unlikely(fake_rotate_event(log_file, start_pos))) break;
    if (reader.open(log_file)) { set_fatal_error(...); break; }
    if (send_binlog(reader, start_pos)) break;
    mysql_bin_log.lock_index();
    if (!mysql_bin_log.is_open()) { /* binlog 被关：重开 index 找下一个 */ }
    int error = mysql_bin_log.find_next_log(&m_linfo, false);
    mysql_bin_log.unlock_index();
    if (unlikely(error)) { set_fatal_error("could not find next log"); break; }
    start_pos = BIN_LOG_HEADER_SIZE;         // 下一文件跳过 4 字节 magic
    reader.close();
  }
  if (was_killed_by_duplicate_slave_id)
    set_fatal_error("A replica with the same server_uuid/server_id ... has connected");
  cleanup();
}
```

`send_binlog()` 是"取终点 → 送到终点 → 再取终点"的循环：

```cpp
int Binlog_sender::send_binlog(File_reader &reader, my_off_t start_pos) {
  if (unlikely(send_format_description_event(reader, start_pos))) return 1;
  if (m_check_previous_gtid_event) {
    if (has_previous_gtid_log_event(reader, &has_prev_gtid_ev)) return 1;
    if (!has_prev_gtid_ev) return 0;         // ★ 无 Previous_gtids → 整个文件不发给从库
  }
  if (reader.position() != start_pos && reader.seek(start_pos)) return 1;
  while (!m_thd->killed) {
    auto [end_pos, code] = get_binlog_end_pos(reader);
    if (code) return 1;
    if (send_events(reader, end_pos)) return 1;
    if (end_pos == 0) return 0;              // 冷文件：读完即止
  }
  return 1;
}
```

只有**活跃文件**（`is_active()`）才有非零 `end_pos`。`has_previous_gtid_log_event()` 返回 false 时 `return 0`，外层会 `find_next_log()` 跳到下一个文件——这正是"从库用 GTID 协议连上、但主库有一批 GTID-free 老 binlog"时**静默跳过**这些文件的机制。

### 定位起点：`check_start_file()`

- **GTID auto-positioning**：`MYSQL_BIN_LOG::find_first_log_not_in_gtid_set()` **逆序**逐个只读各文件的 `Previous_gtids_log_event`，找到第一个 `previous_gtid_set.is_subset(gtid_set)` 的文件即起点
- 此前两道"从库超前"裁决：`m_exclude_gtid` 必须是 `executed_gtids ∪ owned_gtids` 的子集（否则 `ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE`）；`gtid_state->get_lost_gtids()` 必须是 `m_exclude_gtid` 的子集（否则 `ER_SOURCE_HAS_PURGED_REQUIRED_GTIDS`）——**这是"要的 binlog 已被 purge"的唯一判定点**
- **pos 合法性**：`m_start_pos < BIN_LOG_HEADER_SIZE(4)` 报错；`> file length` 报错。magic/FD 的合法性由 `File_reader::open()` 与 `send_format_description_event()` 兜住

### 等待新事件与心跳

```cpp
int Binlog_sender::wait_new_events(my_off_t log_pos) {
  if (stop_waiting_for_update(log_pos)) return 0;   // 快路径：原子量已推进，不进锁不进 cond
  if (flush_net()) return 1;                        // ★ 等之前必须把 net buffer 冲干净
  mysql_bin_log.lock_binlog_end_pos();
  m_thd->ENTER_COND(mysql_bin_log.get_log_cond(),   // update_cond
                    mysql_bin_log.get_binlog_end_pos_lock(),
                    &stage_source_has_sent_all_binlog_to_replica, &old_stage);
  if (m_heartbeat_period.count() > 0) ret = wait_with_heartbeat(log_pos);
  else                                 ret = wait_without_heartbeat(log_pos);
  mysql_bin_log.unlock_binlog_end_pos();
  m_thd->EXIT_COND(&old_stage);
}
```

- 唤醒源是 commit 路径写完 binlog 后 `update_binlog_end_pos()` → `signal_update()` → `mysql_cond_broadcast(&update_cond)`
- ★ **心跳不是定时器线程**：`wait_with_heartbeat()` 是"条件变量带超时醒来 → 临时**释放** `LOCK_binlog_end_pos`（`Scope_guard` 负责重新加锁）→ `send_heartbeat_event()`"
- 心跳周期来自从库建连时 `SET @source_heartbeat_period`（旧名 `@master_heartbeat_period`）这个**用户变量**
- `get_binlog_end_pos()` 用原子量 `atomic_binlog_end_pos` 做**无锁快路径**，只在真要睡时才加锁

### 事件读取、过滤与改写

```cpp
  if (m_exclude_gtid && (in_exclude_group = skip_event(event_ptr, in_exclude_group))) {
    if (m_heartbeat_period > 0ns && (now - m_last_event_sent_ts) >= m_heartbeat_period) {
      if (send_heartbeat_event(log_pos)) return 1;   // 跳过太久也要发心跳
    } else exclude_group_end_pos = log_pos;          // 攒着，等下个要发的事件前补发
  } else {
    if (exclude_group_end_pos) { /* 先补一个心跳再发真事件（含 packet 现场保护/恢复） */ }
    if (before_send_hook(log_file, log_pos)) return 1;        // ← 半同步埋点
    if (unlikely(send_packet())) return 1;
    if (unlikely(after_send_hook(log_file, in_exclude_group ? log_pos : 0))) return 1;
  }
```

- **读侧不反序列化**：`read_event()` 只拿裸字节，除 FD / GTID / QUERY(DDL 判定) 外原样转发
- **过滤只有三类**：`check_event_type()`（GTID_MODE 与事件类型冲突 → 致命错）、`skip_event()`（GTID 在 exclude 集内则整组跳过，**ROTATE 永不跳过**）、合成事件
- **三个合成/改写事件**：
  - `fake_rotate_event()`：手工拼 ROTATE，`timestamp=0`（让从库能区分真假）、`LOG_EVENT_ARTIFICIAL_F`，物理上不存在于任何 binlog
  - `send_format_description_event()`：真 FD 会被**改写**——清 `LOG_EVENT_BINLOG_IN_USE_F`；GTID 协议且 `m_gtid_clear_fd_created_flag` 时把 `created` 置 0（防从库清临时表）；非 GTID 协议且 `start_pos > 4` 时把 `log_pos` 置 0（防从库推进 `group_master_log_pos`）；改完重算 checksum
  - `send_heartbeat_event()`：v1 的 log_pos 用 `int4store`（**>4GB 会截断**），v2 用 `binary_log::codecs` 支持 8 字节
- `send_packet()` 只 `my_net_write()` 进 net buffer；`flush_net()` 才 `net_flush()` 真正发出。★ `get_binlog_end_pos()` 与 `wait_new_events()` 在"追平要等"之前**必须** `flush_net()`，否则从库会误判主库无数据

### 传输层两个开关

- **checksum 协商**：从库建连后 `SET @source_binlog_checksum = @@global.binlog_checksum`；若从库 `UNDEF` 而主库开启 checksum → 直接终止 dump
- **压缩**：服务端变量是 `replica_compressed_protocol`（`slave_compressed_protocol` 只是废弃别名），压缩发生在 **net 层**（`net_write_packet()`），对 `Binlog_sender` 完全透明

| 决策 | 实现 | 代价 / 收益 |
|---|---|---|
| 文件循环 + 读-等循环双层 | `run()` 管切换，`send_binlog()` 管"取终点-送到终点" | 切换文件要重发 FD、重算 Previous_gtids；换 rotate 期间不丢事件 |
| 冷/热文件区分 | `is_active()` + `end_pos==0` | 冷文件读 EOF 就切下一个，热文件才进 `update_cond` 等 |
| 原子量快路径 | `get_binlog_end_pos()` 无锁读 | 避免每事件抢锁 |
| 心跳靠 cond 超时 | 非独立定时器 | 实现简单；精度受 broadcast 影响 |
| 假 ROTATE 无条件发 | `run()` 每次循环首行 | 从库多记一次 rotate（源码注释承认）；换 mysqlbinlog 兼容 |
| 跳过 GTID 补发心跳 | `exclude_group_end_pos` | 从库 `master_log_pos` 不被"静默跳过"卡住 |
| 事件零拷贝 | `Event_allocator` 直读进 `m_packet` | 省一次 memcpy；代价是 packet 需 4KB~4GB 自适应 |
| hook 默认关闭 | `m_observe_transmission` 由 `transmit_start` 出参决定 | 装了半同步/GR 才付 hook 成本 |

## 半同步复制与 binlog 的衔接

> ⚠️ **命名勘误（8.0.39）**：类名**仍是旧名** `ReplSemiSyncBase` / `ReplSemiSyncMaster` / `ReplSemiSyncSlave`，**不存在** `ReplSemiSyncSource` 类；改名只发生在文件名/插件名/sysvar 名层面（CMake 同时产出 4 个插件，后两者由 `_old.cc` 在 `USE_OLD_SEMI_SYNC_TERMINOLOGY` 下 include 新文件编译而来，两组 sysvar 互斥）。另外 ★ `waitAfterSync` / `waitAfterCommit` **全仓 grep 0 命中**——两个埋点已合并进同一个 `commitTrx()`。

### 插件架构：四个 Delegate

| Delegate | semisync 回调 | 服务器侧触发点 |
|---|---|---|
| `Trans_delegate` | `repl_semi_report_commit` / `_rollback` | `process_after_commit_stage_queue`（AFTER_COMMIT 埋点） |
| `Binlog_storage_delegate` | `repl_semi_report_binlog_update`（after_flush）/ `_binlog_sync`（after_sync） | FLUSH 后；`call_after_sync_hook` |
| `Binlog_transmit_delegate` | `repl_semi_binlog_dump_start/stop`、`reserve_header`、`before/after_send_event` | `Binlog_sender` 各处 |
| `Binlog_relay_IO_delegate` | `repl_semi_slave_io_start/io_end/request_dump/read_event/queue_event` | IO 线程 |

### 两种 wait point 的精确埋点

```cpp
static int repl_semi_report_binlog_sync(..., const char *log_file, my_off_t log_pos) {
  if (rpl_semi_sync_source_wait_point == WAIT_AFTER_SYNC)
    return repl_semisync->commitTrx(log_file, log_pos);
  return 0;
}
static int repl_semi_report_commit(Trans_param *param) {
  bool is_real_trans = param->flags & TRANS_IS_REAL_TRANS;
  if (rpl_semi_sync_source_wait_point == WAIT_AFTER_COMMIT && is_real_trans && param->log_pos)
    return repl_semisync->commitTrx(param->log_file, param->log_pos);
  return 0;
}
```

```
FLUSH   process_flush_stage_queue → flush_cache_to_file
        └─ after_flush → writeTranxInBinlog()   ← 登记位点进 active_tranxs_
SYNC    sync_binlog_file(false)
        change_stage(COMMIT_STAGE)   ← 拿到 LOCK_commit，尚未做引擎提交
        ├─ ★ AFTER_SYNC 埋点：call_after_sync_hook → commitTrx()      【默认】
        ├─ process_commit_stage_queue() → 引擎 ha_commit_low()，事务对其他会话可见
        change_stage(AFTER_COMMIT_STAGE)
        └─ ★ AFTER_COMMIT 埋点：process_after_commit_stage_queue → commitTrx()
        → signal_done() → finish_commit() → 客户端收到 OK
```

**崩溃语义差别**：

- **AFTER_SYNC**（默认）：等待在引擎提交**之前**。主库在等待后、引擎提交前崩溃 ⇒ 事务在主库不存在（会被回滚），但 binlog 已落盘、从库已收到 ⇒ 需要人工处理，但**客户端从未收到过成功**
- **AFTER_COMMIT**：等待在引擎提交**之后**、回 OK **之前**。此时事务已在主库可见、其他会话能读到；此刻崩溃且从库未收到 ⇒ 数据永久丢失，而**并发会话可能已读到这份即将丢失的数据并据此决策**（"幻读式"不一致）

### `commitTrx`：等待与降级

```cpp
  lock();                                    // LOCK_binlog_
  TranxNode *entry = nullptr;
  if (active_tranxs_ != nullptr && trx_wait_binlog_name) {
    entry = active_tranxs_->find_active_tranx_node(trx_wait_binlog_name, trx_wait_binlog_pos);
    if (entry) thd_cond = &entry->cond;      // ★ 事务槽位自带的 cond
  }
  THD_ENTER_COND(nullptr, thd_cond, &LOCK_binlog_,
                 &stage_waiting_for_semi_sync_ack_from_replica, &old_stage);
  if (getMasterEnabled() && trx_wait_binlog_name) {
    /* 计算绝对超时 wait_timeout_ 毫秒 */
    while (is_on()) {
      if (reply_file_name_inited_) {
        if (ActiveTranx::compare(reply_file_name_, reply_file_pos_,
                                 trx_wait_binlog_name, trx_wait_binlog_pos) >= 0)
          break;                             // 从库已追上，无需等待
      }
      if (!entry) { is_semi_sync_trans = false; goto l_end; }  // 未开 semi → 视为异步
      /* 维护 wait_file_* = 所有等待者中最小的位点 */
      if (connection_events_loop_aborted() && 
          (rpl_semi_sync_source_clients == rpl_semi_sync_source_wait_for_replica_count - 1) && is_on()) {
        LogErr(WARNING_LEVEL, ER_SEMISYNC_FORCED_SHUTDOWN);
        switch_off();                        // 强制降级，避免 shutdown 挂死
        break;
      }
      rpl_semi_sync_source_wait_sessions++;
      entry->n_waiters++;
      wait_result = mysql_cond_timedwait(&entry->cond, &LOCK_binlog_, &abstime);
      entry->n_waiters--;
      if (wait_result != 0) {                // 真超时
        rpl_semi_sync_source_wait_timeouts++;
        switch_off();                        // ★ 降级为异步
      } else { /* 统计 wait_time */ }
    }
  l_end:
    if (is_on() && is_semi_sync_trans) rpl_semi_sync_source_yes_transactions++;  // yes_tx
    else                              rpl_semi_sync_source_no_transactions++;    // no_tx
  }
  if (trx_wait_binlog_name && active_tranxs_ && entry && entry->n_waiters == 0)
    active_tranxs_->clear_active_tranx_nodes(...);   // 最后一个等待者清理槽位
```

要点：

- 等待对象是**事务自己的 `TranxNode::cond`**（不是全局 cond），`active_tranxs_` 是按 (file,pos) 有序的链表+哈希表；唤醒由 `signal_waiting_sessions_up_to(reply_file_pos_)` 沿链表**广播**所有位点 ≤ 已 ACK 位点的 node
- ★ **超时不回滚**：`commitTrx` 不返回错误，事务照常提交，只是 `switch_off()` 把 `state_ = false` 并 `signal_waiting_sessions_all()`，本次计入 `no_tx`，`Rpl_semi_sync_source_status` 翻 OFF
- 恢复 ON 靠 `try_switch_on()`：dump 线程发下一事件时若其位点 ≥ `commit_file_*` 说明从库已追上

### 从库何时回 ACK

```cpp
static int repl_semi_slave_queue_event(Binlog_relay_IO_param *param, ...) {
  if (rpl_semi_sync_replica_status && semi_sync_need_reply)
    (void)repl_semisync->slaveReply(param->mysql, param->master_log_name, param->master_log_pos);
  return 0;
}
```

★ ACK 在 **`after_queue_event`** 里发，即**事件已写入 relay log 之后**，而非收到即回。`slaveReadSyncHeader` 剥掉 2 字节头（`kPacketMagicNum = 0xef`，`kPacketFlagSync = 0x01`）判断是否需要回复。

源侧收 ACK 的是 **`Ack_receiver` 后台线程**（不是 dump 线程）：`Socket_listener` 轮询所有 semisync dump 的 vio → `reportReplyPacket` → `handleAck`。`wait_for_replica_count > 1` 时先塞进 `AckContainer`，凑够 N 个不同从库才上报**其中最小的**位点。

```
 客户端      ordered_commit        dump 线程        副本 IO 线程     Ack_receiver
   │  COMMIT      │                   │                  │                │
   ├─────────────►│ FLUSH: after_flush → writeTranxInBinlog()             │
   │              │ SYNC              │                  │                │
   │              │ ★ AFTER_SYNC: commitTrx() cond_timedwait ┐            │
   │              │           send XID（before_send → sync 位置 1）◄┘      │
   │              │                   ├─────────────────►│ queue_event     │
   │              │                   │                  │ 写 relay log    │
   │              │                   │◄─────────────────┤ slaveReply()    │
   │              │                   │            net_flush ────────────►│
   │              │      reportReplyBinlog() → signal_waiting_sessions_up_to()
   │              │ 引擎 commit（AFTER_SYNC 语义）                         │
   │◄─── OK ──────┤                   │                  │                │
```

### 与 binlog 发送的交互

- `transmit_start` 读会话变量 `rpl_semi_sync_replica` 判断对端是否 semisync，是则 `ack_receiver->add_slave()`、`clients++`、置 `set_observe_flag()`，并**乐观假设**该从库已收到其请求位点之前的所有事件、立即 `handleAck(server_id, log_file, log_pos)`
- `reserve_header` 预留 2 字节 `{0xef, 0}`；`send_heartbeat_event_v2` 也走它，所以 HB 包同样带 semisync 头（sync 位为 0）
- ★ `before_send_event` → `updateSyncHeader`：**只有该事件位点确实是一个活跃事务的结束位点**（`is_tranx_end_pos`）才置 sync 位——这就是"只对事务边界要 ACK"
- `replication_sender_observe_commit_only=ON` 时 `Observe_transmission_guard` 只在 XID / XA_PREPARE / TRANSACTION_PAYLOAD / DDL-QUERY 上打开 observe，中间行事件完全不过 observer
- `after_send_event` 中被跳过的区间走 `skipSlaveReply` → **源端单方面把跳过的位置算作已 ACK**，防止历史 GTID 拖住 `reply_file_pos_`

### 边界：不保证 apply

- 半同步只保证"至少一个从库的 **IO 线程收到并写入 relay log**"，**不保证 applier 已 apply**
- ★ `writeTranxInBinlog` 在 **after_flush**（`sync_binlog` 之前）就登记位点。若 `sync_binlog != 1`，binlog 可能只在 OS cache，此时从库已 ACK 而主库掉电仍会丢 ⇒ **`sync_binlog=1` 是半同步"不丢"的前提之一**
- `rpl_semi_sync_source_wait_no_replica` 默认 1：从库不足时**仍等满 timeout** 才降级；置 0 则立即 `switch_off()`

| 决策 | 实现 | 代价 / 收益 |
|---|---|---|
| 插件 + Delegate observer | 4 个 `*_delegate` + `RUN_HOOK` | 解耦；每事件多一次 observer 遍历，故有 `observe_commit_only` |
| 两埋点合并进单一 `commitTrx` | 由 `wait_point` 在 hook 入口分流 | 逻辑集中，语义靠调用方保证 |
| 每事务一个 `TranxNode::cond` | `TranxNodeAllocator` 池化 | 唤醒精确、避免惊群；需 `n_waiters` 引用计数做清理 |
| ACK 读取独立成 `Ack_receiver` 线程 | dump 线程 `readSlaveReply` 只 flush | dump 线程不被单从库阻塞，支持多从库并发 ACK |
| 超时只降级不回滚 | `switch_off()` + `signal_waiting_sessions_all()` | 可用性优先；计入 `no_tx`，status 翻 OFF |
| 恢复 ON 靠"推断从库追上" | `updateSyncHeader` 比对 `commit_file_*` | 无需额外协议；时机是启发式的，可能滞后一个事件 |
| 跳过区间视为已 ACK | `skipSlaveReply` → `handleAck` | 防止旧 GTID 拖住 `reply_file_pos_` |

## 参考

**官方文档**

- *MySQL 8.0 Reference Manual → Binary Log*
- *MySQL 8.0 Reference Manual → XA Transactions*

**相关文档**

- 2PC 引擎侧执行（prepare 五层逐行、undo 状态、外部 XA、崩溃恢复三幕）见 [`../../innodb/trx.md`](../../innodb/trx.md)
- GTID 的三条持久化路径见 [`gtid.md`](gtid.md)
- **MTS 依赖追踪**（`sequence_number` / `last_committed` 怎么算、三种模式、从库如何消费）见 [`prpl.md`](prpl.md)（本篇只覆盖"写进 binlog"这一段）
- 复制拓扑与故障转移见 [`replication.md`](replication.md)；GTID 见 [`gtid.md`](gtid.md)


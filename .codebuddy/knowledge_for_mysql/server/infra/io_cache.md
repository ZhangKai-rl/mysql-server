# IO_CACHE 深度剖析：server 层的通用带缓冲 I/O

> 基于 MySQL 8.0.39 源码。涵盖 `IO_CACHE` 的数据结构与状态机、**"穷人版多态"（函数指针）**、**延迟创建文件**、**把网络流伪装成文件**、以及围绕它建立的 **ostream / istream 类继承体系**（`Basic_ostream` → `Truncatable_ostream` → `IO_CACHE_ostream` / `Binlog_encryption_ostream` / `IO_CACHE_binlog_cache_storage`）。

> **边界**：本篇讲 **server 层通用的带缓冲 I/O 抽象**。InnoDB 自己的 I/O 栈（`fil_io` / `os_file` / AIO / doublewrite）见 [`../../innodb/io.md`](../../innodb/io.md)，其中「server 层 I/O 全景」一节是本篇的**上游场景清单**；binlog 的组提交与复制语义见 [`../replication/binlog.md`](../replication/binlog.md)；MyISAM 的 key cache 是另一套独立的引擎私有缓存，见 `../../innodb/io.md` 的相关小节。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现一：IO_CACHE 的数据结构](#核心实现一io_cache-的数据结构)
- [核心实现二：穷人版多态（函数指针）](#核心实现二穷人版多态函数指针)
- [核心实现三：生命周期与状态机](#核心实现三生命周期与状态机)
- [核心实现四：关键算法逐段剖析](#核心实现四关键算法逐段剖析)
- [核心实现五：延迟创建文件（file == -1 的妙用）](#核心实现五延迟创建文件file--1-的妙用)
- [核心实现六：不是文件的 IO_CACHE（NET / FIFO / 内存）](#核心实现六不是文件的-io_cachenet--fifo--内存)
- [核心实现七：透明加解密](#核心实现七透明加解密)
- [核心实现八：★ server 层 I/O 的类继承体系](#核心实现八-server-层-io-的类继承体系)
- [核心实现九：★ 这套体系里的 C++ 运用](#核心实现九-这套体系里的-c-运用)
- [谁在用 IO_CACHE](#谁在用-io_cache)
- [相关的系统变量](#相关的系统变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`IO_CACHE` 是 MySQL server 层的**通用带缓冲 I/O** 结构。它包装一个文件描述符（或压根不包装），提供 `my_b_read` / `my_b_write` / `my_b_printf` / `my_b_get` 这类接口，把"多次小写"攒成"一次大写"。

它不是某一处的专用机制，而是**被十几个子系统复用**的底座：binlog、relay log、slow/general log、filesort、Unique、hash join chunk、LOAD DATA、SELECT INTO OUTFILE、复制元数据文件、mysqlbinlog 的事件打印……

### 为什么复杂

三个原因叠加：

1. **一套结构，六种语义**：`READ_CACHE` / `WRITE_CACHE` / `SEQ_READ_APPEND` / `READ_FIFO` / `READ_NET` / `WRITE_NET`，同一组字段在不同 type 下含义完全不同。
2. **它要能不指向文件**：网络流（`LOAD DATA LOCAL INFILE`）、FIFO、纯内存缓冲都能用同一套接口。
3. **它上面还叠了一层 C++ 继承体系**（ostream / istream），而这一层与 mysys 的 C 风格结构是"两套范式缝合"的。

> **一句话**：`IO_CACHE` 是 C 语言写的、带 fast-path 的、可延迟落盘的、能伪装网络的"穷人多态"缓冲结构；外面又套了一层 C++ 装饰器/门面体系。

### 版本演进

| 版本 | 变化 |
|------|------|
| 早期 | `IO_CACHE` 与 `IO_CACHE_SHARE`（多 dump 线程共享读同一个 binlog） |
| 5.7 | `IO_CACHE_ostream` / `IO_CACHE_istream` 引入，开始把 C 结构包装成 C++ 流 |
| 8.0 | ① binlog 写路径全面改用 `IO_CACHE_ostream` + `Binlog_encryption_ostream` 装饰器链；② **`SEQ_READ_APPEND` 在 `sql/` 层已无调用点**（循环缓冲成为遗留能力）；③ **`IO_CACHE_SHARE` 彻底废弃**（`sql/` 下 0 匹配，代码未删）；④ 新增 binlog cache 临时文件加密（`m_encryptor` / `m_decryptor`） |

---

## 理论基础

### 设计思想与权衡

#### 一、★ 为什么用函数指针而不是 C++ 虚函数

`IO_CACHE` 里有 `read_function` / `write_function` 两个函数指针，这是**手工实现的 vtable**，只装 1~2 个 slot。为什么不用 C++ 虚类？

看 `my_b_write` 的定义（`include/my_sys.h:495-511`）：

```c
inline int my_b_write(IO_CACHE *info, const uchar *buffer, size_t count) {
  if (info->write_pos + count <= info->write_end) {
    memcpy(info->write_pos, buffer, count);
    info->write_pos += count;
    return 0;
  }
  return (*info->write_function)(info, buffer, count);
}
```

**这是"手工 devirtualization"（手工去虚化）**：

```
热路径（99% 的情况）：数据在缓冲区里 → 纯 memcpy，零函数调用
冷路径（1%）：缓冲区满了 → 才走函数指针间接调用
```

如果改成纯 C++ 虚类，每次 `my_b_write` 都是一次**间接调用 + 无法内联**。而这里是"能内联的分支在前，间接调用在后"。这是 C 风格性能工程对 C++ 多态的经典取舍：

| | C++ 虚函数 | IO_CACHE 的函数指针 |
|---|---|---|
| 调用开销 | 每次都是间接调用 | **热路径直接 memcpy，零调用** |
| 可扩展性 | 编译期固定 | 运行期可替换（READ_NET 就是运行期填的） |
| 类型安全 | 编译期检查 | 无（填错就是段错误） |

**这个取舍的代价直接体现在代码里**：`SEQ_READ_APPEND` 故意把 `write_function` 置为 `nullptr`，注释写的是 "Force a core if used"——因为类型系统帮不了忙，只能用崩溃来兜底。

#### 二、为什么 IO_CACHE 要能"不是文件"

这是理解 `READ_NET` / `READ_FIFO` / `open_cached_file` 三者的统一视角：

> **只要有一处数据源（文件 / 网络 / 管道 / 内存），上层的"读一行、解析一个字段"逻辑就不该关心它来自哪里。**

| 数据源 | 机制 | 收益 |
|--------|------|------|
| 文件 | 普通 fd | — |
| 网络 | `READ_NET` + `_my_b_net_read` | `LOAD DATA LOCAL INFILE` 的解析逻辑与本地文件**完全复用** |
| FIFO | `READ_FIFO` | 同上 |
| 内存（可能溢出到临时文件） | `open_cached_file`（`file == -1`） | **小数据完全不落盘** |

#### 三、★ 分层妥协：`READ_NET` 打破了 mysys / sql 的边界

这是最有教学价值的一处设计。`init_functions()` 里对 `READ_NET` **什么都不做**（`mysys/mf_iocache.cc:128-150`）：

```c
static void init_functions(IO_CACHE *info) {
  enum cache_type type = info->type;
  switch (type) {
    case READ_NET:
      /*
        Must be initialized by the caller. The problem is that
        _my_b_net_read has to be defined in sql directory because of
        the dependency on THD, and therefore cannot be visible to
        programs that link against mysys but know nothing about THD, such
        as myisamchk
      */
      break;
    case SEQ_READ_APPEND:
      info->read_function = _my_b_seq_read;
      info->write_function = nullptr; /* Force a core if used */
      break;
    default:
      info->read_function = info->share ? _my_b_read_r : _my_b_read;
      info->write_function = _my_b_write;
  }

  setup_io_cache(info);
}
```

**问题**：`_my_b_net_read` 需要 `NET*`，而 `NET*` 从 `current_thd` 拿 ⇒ 它必须住在 `sql/`；但 `mysys` 库要能被 `myisamchk` 这类**不认识 THD** 的程序链接。

**解法**：mysys 层**故意留空槽**，由 sql 层在运行时填（`sql/sql_load.cc:1406`）：

```c
      if (get_it_from_net) cache.read_function = _my_b_net_read;
```

> 这是"依赖倒置"的 C 语言版：底层定义接口（函数指针），上层注入实现，从而避免底层反向依赖上层。**代价是类型系统保护失效，且这个约定只有注释守护。**

#### 四、设计模式

| 模式 | 体现 |
|------|------|
| **手工 vtable（函数指针）** | `read_function` / `write_function` + fast-path 内联 |
| **延迟初始化（Lazy）** | `open_cached_file` 时 `file == -1`，首次 flush 才真建文件 |
| **装饰者（Decorator）** | `Binlog_encryption_ostream` 包裹 `IO_CACHE_ostream`，做偏移翻译 |
| **门面（Facade）** | `MYSQL_BIN_LOG::Binlog_ofile` 组装 pipeline 并维护位点 |
| **回调 / 钩子** | `pre_read` / `pre_close` / `arg`（LOAD DATA 的 binlog 记录） |
| **二级指针做多态** | `current_pos` / `current_end` 指向 `&read_pos` 或 `&write_pos` |

#### 五、理论溯源

| 概念 | 落点 |
|------|------|
| **stdio 缓冲 I/O** | `IO_CACHE` 本质是可控版本的 stdio 缓冲，但暴露了 flush / spill / 上限等控制点 |
| **手工去虚化（devirtualization）** | `my_b_read` / `my_b_write` 的 fast-path |
| **依赖倒置原则（DIP）** | `READ_NET` 的函数指针由上层注入 |
| **AOP（面向切面）** | `mysql_encryption_file_read/write/seek` 是所有 I/O 的统一拦截点 |
| **CTR 流密码** | 使"加密 + 可 seek"同时成立（CBC 做不到） |

#### 六、与 InnoDB Buffer Pool 的对比（双缓存问题的根源）

| | `IO_CACHE` | InnoDB Buffer Pool |
|---|---|---|
| 缓存对象 | **字节流**（顺序） | **页**（随机，按 space_id+page_no 索引） |
| 替换策略 | 无（顺序消费/顺序追加） | LRU + 中点插入 |
| 是否绕过 OS page cache | 否（**与 page cache 叠加**） | 是（`O_DIRECT`） |
| 落盘控制 | `flush_io_cache`（write(2)）+ 可选 `my_sync` | 刷脏 + `fsync` + doublewrite |
| 并发 | 单线程写（或已废弃的 share 机制） | 高并发，latch/io_fix 全套 |

> **★ 这是 server 层"双缓存"的根源**：数据从 OS page cache → `IO_CACHE` 缓冲 → 用户，中间多了一份。对 binlog 这类顺序写影响不大（OS 的 write-back 也能合并），但对"每行一次 flush"的 general log 就很痛。

---

## 核心实现一：IO_CACHE 的数据结构

### 完整定义（`include/my_sys.h:340-466`）

```c
struct IO_CACHE /* Used when caching files */
{
  /* Offset in file corresponding to the first byte of uchar* buffer. */
  my_off_t pos_in_file{0};
  my_off_t end_of_file{0};
  /* Points to current read position in the buffer */
  uchar *read_pos{nullptr};
  /* the non-inclusive boundary in the buffer for the currently valid read */
  uchar *read_end{nullptr};
  uchar *buffer{nullptr}; /* The read buffer */
  /* Used in ASYNC_IO */
  uchar *request_pos{nullptr};

  /* Only used in WRITE caches and in SEQ_READ_APPEND to buffer writes */
  uchar *write_buffer{nullptr};
  uchar *append_read_pos{nullptr};
  /* Points to current write position in the write buffer */
  uchar *write_pos{nullptr};
  /* The non-inclusive boundary of the valid write area */
  uchar *write_end{nullptr};

  uchar **current_pos{nullptr}, **current_end{nullptr};

  mysql_mutex_t append_buffer_lock;
  IO_CACHE_SHARE *share{nullptr};

  int (*read_function)(IO_CACHE *, uchar *, size_t){nullptr};
  int (*write_function)(IO_CACHE *, const uchar *, size_t){nullptr};

  cache_type type{TYPE_NOT_SET};

  IO_CACHE_CALLBACK pre_read{nullptr};
  IO_CACHE_CALLBACK post_read{nullptr};
  IO_CACHE_CALLBACK pre_close{nullptr};

  ulong disk_writes{0};
  void *arg{nullptr};       /* for use by pre/post_read */
  char *file_name{nullptr}; /* if used with 'open_cached_file' */
  char *dir{nullptr}, *prefix{nullptr};
  File file{-1};                               /* file descriptor */
  PSI_file_key file_key{PSI_NOT_INSTRUMENTED}; /* instrumented file key */

  bool seek_not_done{false};
  int error{0};
  /* buffer_length is memory size allocated for buffer or write_buffer */
  size_t buffer_length{0};
  size_t read_length{0};
  myf myflags{0};
  bool alloced_buffer{false};
  // This is an encryptor for encrypting the temporary file of the IO cache.
  Stream_cipher *m_encryptor = nullptr;
  // This is a decryptor for decrypting the temporary file of the IO cache.
  Stream_cipher *m_decryptor = nullptr;
  // Synchronize flushed buffer with disk.
  bool disk_sync{false};
  // Delay in milliseconds after disk synchronization of the flushed buffer.
  uint disk_sync_delay{0};
};
```

### ★ 字段语义分组解读

#### A. 双缓冲指针组（理解 IO_CACHE 的关键）

**同一组字段在不同 `type` 下含义完全不同**：

| 字段 | READ_CACHE | WRITE_CACHE | SEQ_READ_APPEND |
|---|---|---|---|
| `buffer` | 读缓冲区 | == `write_buffer` | 读缓冲区 |
| `write_buffer` | == `buffer` | 写缓冲区 | `buffer + cachesize`（**独立的第二块**） |
| `read_pos` / `read_end` | 当前读位置 / 有效边界 | 未用 | 读 buffer 用 |
| `write_pos` / `write_end` | 未用 | 当前写位置 / 可写边界 | 写 write_buffer 用 |
| `append_read_pos` | 未用 | 未用 | 在 **write_buffer** 中的读位置 |

关键一行（`mysys/mf_iocache.cc:257-259`）：

```c
      info->write_buffer = info->buffer;
      if (type == SEQ_READ_APPEND)
        info->write_buffer = info->buffer + cachesize;
```

（分配时 `buffer_block *= 2`，所以 SEQ_READ_APPEND 有两块缓冲区。）

#### A'. ★ 内存布局图（三种 type 下的缓冲区布局）

这是理解 `IO_CACHE` 最直观的一张图。注意 `buffer` / `write_buffer` / 各指针在不同 `type` 下的关系：

```
┌─ case 1: READ_CACHE ─────────────────────────────────────────────────┐
│                                                                       │
│  buffer ──▶ ┌─────────────────────────────────────────┐              │
│             │ 已从文件读入、但尚未被消费的数据          │  cachesize   │
│             └─────────────────────────────────────────┘              │
│              ▲                ▲                    ▲                 │
│              │                │                    │                 │
│           buffer         read_pos             read_end               │
│              (= request_pos)  (当前读位置)      (有效数据边界)         │
│                                                                       │
│  write_buffer == buffer（同一块）                                     │
│  current_pos = &read_pos,  current_end = &read_end                    │
│  my_b_tell() = pos_in_file + (read_pos - request_pos)                 │
└───────────────────────────────────────────────────────────────────────┘

┌─ case 2: WRITE_CACHE ────────────────────────────────────────────────┐
│                                                                       │
│  buffer ──▶ ┌─────────────────────────────────────────┐              │
│  (=write_buffer)│ 待落盘的累积数据                      │              │
│             └─────────────────────────────────────────┘              │
│              ▲                ▲                    ▲                 │
│           buffer         write_pos            write_end              │
│                          (当前写位置)       (可写边界，可能被主动缩小) │
│                                                                       │
│  ★ write_end = buffer + buffer_length - ((pos_in_file+len) & (IO_SIZE-1))
│     → 主动缩小，保证下次 flush 的落盘长度块对齐（见 4.1 ⑥）            │
│  current_pos = &write_pos, current_end = &write_end                   │
└───────────────────────────────────────────────────────────────────────┘

┌─ case 3: SEQ_READ_APPEND（循环双缓冲，8.0 已无业务调用点）────────────┐
│                                                                       │
│  buffer ──▶ ┌──────────────┬──────────────┐  分配时 buffer_block *= 2 │
│             │  读缓冲区     │  写缓冲区     │                          │
│             │  (buffer)    │ (write_buffer │                          │
│             │              │  = buffer +   │                          │
│             │              │    cachesize) │                          │
│             └──────────────┴──────────────┘                          │
│               ▲         ▲    ▲         ▲                             │
│            read_pos  read_end│      write_end                        │
│                              │                                       │
│                        append_read_pos（在 write_buffer 里的读位置）   │
│                              write_pos（当前写位置）                  │
│                                                                       │
│  读可以发生在两个 buffer 中的任意一个 → 所以需要 append_buffer_lock    │
└───────────────────────────────────────────────────────────────────────┘
```

> **为什么要记这张图**：IO_CACHE 的全部复杂性都源于"**同一组字段在不同 type 下含义不同**"。看懂这张图，`_my_b_write` / `_my_b_read` / `reinit_io_cache` 里的指针操作就都不是魔法了。

#### B. 二级指针 `current_pos` / `current_end`

`setup_io_cache()`（`mysys/mf_iocache.cc:117-126`）建立绑定：

```c
void setup_io_cache(IO_CACHE *info) {
  /* Ensure that my_b_tell() and my_b_bytes_in_cache works */
  if (info->type == WRITE_CACHE) {
    info->current_pos = &info->write_pos;
    info->current_end = &info->write_end;
  } else {
    info->current_pos = &info->read_pos;
    info->current_end = &info->read_end;
  }
}
```

于是 `my_b_tell(info)` = `pos_in_file + *current_pos - request_pos`，**调用方无需知道自己是读缓存还是写缓存**。

> **⚠️ 代价**：因为 `current_pos` 是**指向自身成员的指针**，**移动或拷贝 IO_CACHE 对象后必须重新调 `setup_io_cache()`**，否则它指向旧对象的成员。头文件注释明确警告了这点（`mysys/mf_iocache.cc:104-116`）。

#### C. `pos_in_file` vs `end_of_file`

- `pos_in_file`：缓冲区**首字节**在文件中的绝对偏移（不是当前位置！）。
- `end_of_file`：名义上的文件末尾。**这是全结构最大的坑，见 [Misc](#misc)。**

#### D. `seek_not_done`：延迟 seek

任何读写前若发现它为 true，先 `lseek`。这样连续的 `my_b_write` 只在必要时 seek 一次。

对**不可 seek 的对象**（pipe/FIFO/socket）有特判（`mysys/mf_iocache.cc:199-215`）：

```c
    pos = mysql_file_tell(file, MYF(0));
    if ((pos == (my_off_t)-1) && (my_errno() == ESPIPE)) {
      /*
         This kind of object doesn't support seek() or tell(). Don't set a
         flag that will make us again try to seek() later and fail.
      */
      info->seek_not_done = false;
      assert(seek_offset == 0);
    } else
      info->seek_not_done = (seek_offset != pos);
```

#### E. `error` 的三态语义

- `0` = 成功
- `-1` = 硬错误
- `>0` = **部分读写**，值为实际 I/O 的字节数

#### F. `disk_writes`

**每次 flush 计 1（不是每字节）**，这就是 `binlog_cache_disk_use` / `binlog_stmt_cache_disk_use` 状态变量的数据源。字段注释（`include/my_sys.h:424-428`）明说了这个用途。

---

## 核心实现二：穷人版多态（函数指针）

### 绑定表

| `type` | `read_function` | `write_function` |
|---|---|---|
| `TYPE_NOT_SET` | `_my_b_read` | `_my_b_write` |
| `READ_CACHE` | `_my_b_read_r`（若 `share≠NULL`）／ `_my_b_read` | `_my_b_write` |
| `WRITE_CACHE` | `_my_b_read` | `_my_b_write` |
| `READ_FIFO` | `_my_b_read` | `_my_b_write` |
| **`READ_NET`** | **NULL**（上层手工填 `_my_b_net_read`） | `_my_b_write` |
| `WRITE_NET` | `_my_b_read` | `_my_b_write` |
| **`SEQ_READ_APPEND`** | `_my_b_seq_read` | **NULL**（故意置空，误用即 core） |

### 三个精妙之处

**① `SEQ_READ_APPEND` 的 `write_function = nullptr`**

注释是 "Force a core if used"。因为 SEQ_READ_APPEND **必须**用 `my_b_append`（要加 `append_buffer_lock`），用 `my_b_write` 是 bug。这就是 `my_b_safe_write` 存在的原因（`mysys/mf_iocache.cc:1358-1367`）：

```c
int my_b_safe_write(IO_CACHE *info, const uchar *Buffer, size_t Count) {
  if (info->type == SEQ_READ_APPEND) return my_b_append(info, Buffer, Count);
  return my_b_write(info, Buffer, Count);
}
```

（原注释还留了个 2001 年的编译器 bug 考古：*Sasha: We are not writing this with the ? operator to avoid hitting a possible compiler bug... gcc 2.95 cannot deal with several layers of ternary operators*。）

**② `READ_NET` 的 `break`** —— 见[理论基础](#三-分层妥协read_net-打破了-mysys--sql-的边界)。

**③ `my_b_read` / `my_b_write` 的 fast-path**（上文已贴）——手工去虚化。

### `cache_type` 常量与实际使用

```c
enum cache_type {
  TYPE_NOT_SET = 0,
  READ_CACHE,
  WRITE_CACHE,
  SEQ_READ_APPEND /* sequential read or append */,
  READ_FIFO,
  READ_NET,
  WRITE_NET
};
```

| 类型 | 场景 | 8.0.39 的实际使用点 |
|---|---|---|
| `READ_CACHE` | 顺序读 | binlog index（`binlog.cc:4008,5361`）、`IO_CACHE_istream`、`Stdin_istream` |
| `WRITE_CACHE` | 顺序写 | general/slow log（`log.cc:545`）、`IO_CACHE_ostream`、`SELECT INTO OUTFILE`、filesort |
| `SEQ_READ_APPEND` | 边写边读 | **已无业务调用点**，仅存于 `mf_iocache.cc:1576` 的 `#ifdef MAIN` 自测试 |
| `READ_FIFO` | 命名管道 | `LOAD DATA` 从 FIFO 读（`sql_load.cc:1393`） |
| `READ_NET` | 网络伪装成文件 | `LOAD DATA LOCAL INFILE`（`sql_load.cc:1393`） |
| `WRITE_NET` | 写网络 | **无任何使用点**（只在 `init_functions` 的 default 分支与 assert 里出现） |

---

## 核心实现三：生命周期与状态机

```
                    ┌─────────────────────────┐
                    │  open_cached_file()     │  mysys/mf_cache.cc:53
                    │  只设 dir/prefix,       │  → init_io_cache(file=-1,
                    │  file 仍是 -1           │     WRITE_CACHE)
                    └───────────┬─────────────┘
                                │
   ┌────────────────────────────▼──────────────────────────────┐
   │            init_io_cache_ext()   mysys/mf_iocache.cc:176   │
   │  · 探测文件大小（READ_CACHE/SEQ）并据此裁剪 cachesize        │
   │  · 分配 buffer（失败则按 3/4 递减重试）                     │
   │  · end_of_file = 探测值 或 ~(my_off_t)0                    │
   │  · init_functions()  ← 绑定函数指针                        │
   └────────────────────────────┬──────────────────────────────┘
                                │
                    ╔═══════════▼═══════════╗
                    ║    WRITE_CACHE        ║◄──── reinit_io_cache ────┐
                    ║  my_b_write 累积      ║                          │
                    ╚═══════════┬═══════════╝                          │
                                │ 缓冲满 → my_b_flush_io_cache         │
                                │ （file==-1 时 real_open_cached_file）│
                                │                                      │
                                │ reinit_io_cache(READ_CACHE,0,...)    │
                    ╔═══════════▼═══════════╗                          │
                    ║     READ_CACHE        ║──────────────────────────┘
                    ║  my_b_read 消费       ║   （再切回 WRITE_CACHE）
                    ╚═══════════┬═══════════╝
                                │
                    ┌───────────▼───────────┐
                    │    end_io_cache()     │  mysys/mf_iocache.cc:1519
                    │  pre_close 回调 → flush → my_free → 删 cipher     │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │  close_cached_file()  │  先把 file 置 -1 跳过 flush
                    └───────────────────────┘
```

### `init_io_cache_ext` 的六个关键点

```c
  info->type = TYPE_NOT_SET; /* Don't set it until mutex are created */
```
① **先置 `TYPE_NOT_SET`**：中途失败返回时 `type` 保持 NOT_SET，避免 `end_io_cache` 去销毁未初始化的 mutex。

```c
      end_of_file = mysql_encryption_file_seek(info, 0L, MY_SEEK_END, MYF(0));
      ...
      if ((my_off_t)cachesize > end_of_file - seek_offset + IO_SIZE * 2 - 1) {
        cachesize = (size_t)(end_of_file - seek_offset) + IO_SIZE * 2 - 1;
        use_async_io = false; /* No need to use async */
      }
```
② **小文件裁剪**：为 10 字节的文件分配 1 MB 缓冲是浪费。`MY_DONT_CHECK_FILESIZE` 可关闭此优化。

```c
      if (cachesize == min_cache) return 2; /* Can't alloc cache */
      /* Try with less memory */
      cachesize = (cachesize * 3 / 4 & ~(min_cache - 1));
```
③ **内存分配递减重试**：失败就降到 3/4 再试，直到 `min_cache`（8 KB / 16 KB）还失败才放弃。

```c
  info->myflags = cache_myflags & ~(MY_NABP | MY_FNABP);
```
④ **剥离 `MY_NABP` / `MY_FNABP`**：这两个"必须全写完"的标志由 IO_CACHE 在具体调用点追加，不存进 `myflags`。

```c
  if (type == WRITE_CACHE)
    info->write_end =
        info->buffer + info->buffer_length - (seek_offset & (IO_SIZE - 1));
```
⑤ **首块对齐技巧**：让第一次 flush 的落盘长度恰好对齐到 `IO_SIZE`，后续 flush 自然保持对齐。代价是首块略小。

```c
  my_off_t end_of_file = ~(my_off_t)0;
```
⑥ **`end_of_file` 默认是无符号最大值**（"无上限"）——这决定了 `_my_b_write` 的 EFBIG 检查对普通 WRITE_CACHE 永不触发。

---

## 核心实现四：关键算法逐段剖析

### 4.0 先看清全局：写路径与读路径的完整流程

下面两张图是后面四节剖析的地图。先看图，再看代码。

**写路径**（`my_b_write` → `_my_b_write`）：

```
my_b_write(info, buf, N)
   │
   ├──── fast path ────────────────────────────────────────────────┐
   │     write_pos + N <= write_end ?                              │
   │        是 → memcpy(write_pos, buf, N); write_pos += N;        │
   │              return 0        （零函数调用，纯内存操作）         │
   └───────────────────────────────────────────────────────────────┘
   │
   否（缓冲区装不下）→ _my_b_write(info, buf, N)
         │
   ① EFBIG 检查    pos_in_file + buffer_length > end_of_file ?
         │             是 → errno=EFBIG; return -1
         │             ★ 只有 binlog cache 把 end_of_file 设成上限才会触发
         ▼
   ② 填满剩余      rest = write_end - write_pos
                   memcpy(write_pos, buf, rest);  buf += rest;  N -= rest;
                   write_pos = write_end（缓冲满）
         ▼
   ③ flush         my_b_flush_io_cache(info, 1)
                     ├─ file == -1 ? → real_open_cached_file()（延迟建文件！）
                     ├─ 条件 seek
                     ├─ write(2) 落盘  ← 可能经 mysql_encryption_file_write 加密
                     ├─ 可选 fsync（disk_sync）
                     ├─ ++disk_writes
                     └─ 重算 write_end（主动缩小以保证下次块对齐）
         ▼
   ④ 大块直写      剩余 N >= IO_SIZE ?
                     length = N & ~(IO_SIZE-1)      ← 取整到 4096 倍数
                     write(2) 直写，绕过缓冲区
                     N -= length
         ▼
   ⑤ 余料入缓冲     memcpy(write_pos, 剩余 N);  write_pos += N
         ▼
        return 0
```

**读路径**（`my_b_read` → `_my_b_read`，与写路径完全对称）：

```
my_b_read(info, buf, N)
   │
   ├──── fast path ────────────────────────────────────────────────┐
   │     read_pos + N <= read_end ?                                │
   │        是 → memcpy(buf, read_pos, N); read_pos += N;          │
   │              return 0                                          │
   └───────────────────────────────────────────────────────────────┘
   │
   否 → _my_b_read(info, buf, N)
         │
   ① 搬走残留      left = read_end - read_pos
                   memcpy(buf, read_pos, left);  N -= left
         ▼
   ② 定位          pos_in_file + (read_end - buffer)
         ▼
   ③ 延迟 seek     seek_not_done ? → lseek
                   （ESPIPE 断言：管道不该走到这）
         ▼
   ④ 块内偏移      diff = pos_in_file & (IO_SIZE-1)
         ▼
   ⑤ 大块直读      N >= IO_SIZE + (IO_SIZE - diff) ?
                     length = (N & ~(IO_SIZE-1)) - diff   ← 取整再减块内偏移
                     read(2) 直读，绕过缓冲区
         ▼
   ⑥ 填充缓冲      max_length = read_length - diff
                   ★ READ_FIFO 不按 end_of_file 裁剪（管道长度未知）
                   read(2) 读入 info->buffer
         ▼
   ⑦ 交付          read_pos = buffer + N      ← 用户拿走 N 字节
                   read_end = buffer + length ← 多读的 length-N 留给下次 fast path
         ▼
        return 0
```

> **对称性**：写是"填满 → flush → 大块直写 → 余料入缓冲"，读是"搬残留 → 定位 → 大块直读 → 填充缓冲 → 交付"。两者都在**大块时绕过缓冲区**，都在**小块时利用缓冲区**，都用 `IO_SIZE` 对齐。



### 4.1 `my_b_flush_io_cache`（`mysys/mf_iocache.cc:1430-1500`）

这是最核心的函数。完整代码：

```c
int my_b_flush_io_cache(IO_CACHE *info, int need_append_buffer_lock) {
  size_t length;
  my_off_t pos_in_file;
  bool append_cache = (info->type == SEQ_READ_APPEND);

  if (!append_cache) need_append_buffer_lock = 0;

  if (info->type == WRITE_CACHE || append_cache) {
    if (info->file == -1) {                                     /* ① */
      if (real_open_cached_file(info)) return (info->error = -1);
    }
    LOCK_APPEND_BUFFER;                                         /* ② */

    if ((length = (size_t)(info->write_pos - info->write_buffer))) {  /* ③ */
      if (info->share) copy_to_read_buffer(info, info->write_buffer, length); /* ④ */

      pos_in_file = info->pos_in_file;
      if (!append_cache && info->seek_not_done) {                /* ⑤ */
        if (mysql_encryption_file_seek(info, pos_in_file, MY_SEEK_SET,
                                       MYF(0)) == MY_FILEPOS_ERROR) {
          UNLOCK_APPEND_BUFFER;
          return (info->error = -1);
        }
        if (!append_cache) info->seek_not_done = false;
      }
      if (!append_cache) info->pos_in_file += length;
      info->write_end = (info->write_buffer + info->buffer_length -
                         ((pos_in_file + length) & (IO_SIZE - 1)));   /* ⑥ */

      if (mysql_encryption_file_write(info, info->write_buffer, length,
                                      info->myflags | MY_NABP))       /* ⑦ */
        info->error = -1;
      else
        info->error = 0;
      if (!append_cache) {
        info->end_of_file = std::max(info->end_of_file, (pos_in_file + length)); /* ⑧ */
      } else {
        info->end_of_file += (info->write_pos - info->append_read_pos);
        assert(info->end_of_file == mysql_file_tell(info->file, MYF(0)));
      }

      info->append_read_pos = info->write_pos = info->write_buffer;   /* ⑨ */
      if (info->disk_sync) {                                          /* ⑩ */
        if (mysql_file_sync(info->file, MYF(MY_WME))) {
          UNLOCK_APPEND_BUFFER;
          return -1;
        }
        if (info->disk_sync_delay) my_sleep(info->disk_sync_delay * 1000);
      }
      ++info->disk_writes;                                            /* ⑪ */
      UNLOCK_APPEND_BUFFER;
      return info->error;
    }
  }
  UNLOCK_APPEND_BUFFER;
  return 0;
}
```

**逐段解释**：

**① spill 分支（延迟创建文件）**
```c
    if (info->file == -1) {
      if (real_open_cached_file(info)) return (info->error = -1);
    }
```
`open_cached_file` 机制在这里兑现：**直到第一次真正需要落盘才建临时文件**。

**② 条件加锁**：只有 `SEQ_READ_APPEND` 才需要（读写并发访问 `write_buffer`），普通 WRITE_CACHE 单线程写。

**③ 空缓冲短路**：`write_pos == write_buffer` 时直接返回 0。

**④ 共享缓存直传（死代码）**：`share` 恒为 NULL（见 [Misc](#misc)），这条永不执行。注释说放在 write **之前**是为了让读者的处理与本次写并行。

**⑤ 条件 seek**：append 缓存用 `O_APPEND` 打开，内核自动定位到 EOF，不需要 seek。

**⑥ ★ 重新计算 `write_end`（最微妙的一行）**
```c
      info->write_end = (info->write_buffer + info->buffer_length -
                         ((pos_in_file + length) & (IO_SIZE - 1)));
```
`write_end` **主动缩小** `(pos_in_file + length) % IO_SIZE` 的量。目的：让**下一次** flush 时 `pos_in_file + length` 重新对齐到 IO_SIZE。

举例：`buffer_length = 4096`，写完 `pos_in_file + length = 4100`，则 `4100 & 4095 = 4`，`write_end = write_buffer + 4092`。下次最多写 4092 字节，`4100 + 4092 = 8192` = 2×IO_SIZE。**这样每次 flush 的长度都是块对齐的**，代价是有效载荷略小。

**⑦ 写盘 + 加密钩子**：注意**追加了 `MY_NABP`**（not all bytes = error）。

**⑧ `end_of_file` 更新用 `max`**：因为 `end_of_file` 可能被外部设成上限值（binlog cache），不能被正常写入拉低。

**⑨ 指针复位**：写缓冲清空。

**⑩ 可选 fsync**：`disk_sync_delay` 单位是毫秒，所以 `* 1000` 转微秒给 `my_sleep`（测试模拟慢盘用）。

> **一个不一致**：这里 `return -1` **没有设置 `info->error`**，且提前返回时**没有** `++disk_writes`，与成功路径行为不同。

**⑪ `disk_writes` 计数**：这就是 `binlog_cache_disk_use` 的数据源。注意**即使只写了几个字节也计 1**。

### 4.2 `_my_b_write`（`mysys/mf_iocache.cc:1238-1311`）

```c
int _my_b_write(IO_CACHE *info, const uchar *Buffer, size_t Count) {
  size_t rest_length, length;
  my_off_t pos_in_file = info->pos_in_file;

  if (pos_in_file + info->buffer_length > info->end_of_file) {   /* ① */
    errno = EFBIG;
    set_my_errno(EFBIG);
    return info->error = -1;
  }

  rest_length = (size_t)(info->write_end - info->write_pos);      /* ② */
  memcpy(info->write_pos, Buffer, (size_t)rest_length);
  Buffer += rest_length;
  Count -= rest_length;
  info->write_pos += rest_length;

  if (my_b_flush_io_cache(info, 1)) return 1;                     /* ③ */
  if (Count >= IO_SIZE) {                                          /* ④ */
    length = Count & (size_t) ~(IO_SIZE - 1);
    if (info->seek_not_done) { ... }
    if (mysql_encryption_file_write(info, Buffer, length,
                                    info->myflags | MY_NABP))
      return info->error = -1;
    if (info->share) copy_to_read_buffer(info, Buffer, length);
    Count -= length;
    Buffer += length;
    info->pos_in_file += length;
  }
  memcpy(info->write_pos, Buffer, (size_t)Count);                  /* ⑤ */
  info->write_pos += Count;
  return 0;
}
```

**① ★ EFBIG 检查（最关键）**

只在 `_my_b_write` 被调用时检查（即缓冲区满时），是"提前一个缓冲区"的预警。对普通 `WRITE_CACHE`，`end_of_file` 是 `~(my_off_t)0`，**永不触发**。

**只有 `IO_CACHE_binlog_cache_storage` 把 `end_of_file` 设成 `max_cache_size`**（`binlog_ostream.cc:55`），于是这里就变成 `max_binlog_cache_size` 的实现！

**② 填满剩余（无条件 memcpy）**：没有检查 `Count >= rest_length`，因为能进 `_my_b_write` 就说明 fast-path 失败了（`Count > rest_length`），这是调用契约不变式。

**④ 大块直写（bypass 缓冲）**：剩余 ≥ IO_SIZE 时取整到 IO_SIZE 倍数直接写文件，**完全绕过缓冲区**。避免大块数据经由小缓冲来回 memcpy。

**⑤ 余料入缓冲**：不足 IO_SIZE 的尾巴放进刚清空的缓冲区。

**完整的"三明治"写入策略**：
```
[填满当前缓冲] → [flush 落盘] → [大块直写] → [余料入缓冲]
```

### 4.3 `_my_b_read`（`mysys/mf_iocache.cc:424-550`）的对称设计

与 `_my_b_write` 完全对称：

```c
  if ((left_length = (size_t)(info->read_end - info->read_pos))) {   /* ① */
    memcpy(Buffer, info->read_pos, left_length);
    Buffer += left_length;
    Count -= left_length;
  }

  pos_in_file = info->pos_in_file + (size_t)(info->read_end - info->buffer);  /* ② */
  ...
  diff_length = (size_t)(pos_in_file & (IO_SIZE - 1));                /* ③ */

  if (Count >= (size_t)(IO_SIZE + (IO_SIZE - diff_length))) {         /* ④ 大块直读 */
    length = (Count & (size_t) ~(IO_SIZE - 1)) - diff_length;
    ...
    mysql_encryption_file_read(info, Buffer, length, info->myflags);
    ...
  }

  max_length = info->read_length - diff_length;                       /* ⑤ 填充缓冲 */
  if (info->type != READ_FIFO && max_length > (info->end_of_file - pos_in_file))
    max_length = (size_t)(info->end_of_file - pos_in_file);
  ...
  info->read_pos = info->buffer + Count;                              /* ⑥ 交付 */
  info->read_end = info->buffer + length;
```

**关键点**：

- **④ 大块直读**：条件 `Count >= IO_SIZE + (IO_SIZE - diff_length)`；长度 `(Count & ~(IO_SIZE-1)) - diff_length`，**向下取整再减块内偏移**，保证读完后 `pos_in_file` 重新落回块边界。
- **⑤ `READ_FIFO` 特判**：管道长度未知/在增长，**不检查 `end_of_file` 上界**。
- **⑥ 交付时多读的数据被保留**：`read_pos = buffer + Count`（用户拿走 Count），`read_end = buffer + length`（还剩 `length - Count` 给下次 fast-path）。**这才是缓存的意义**。
- ESPIPE 断言引用了历史 bug（Bugs#25807 和 #22828：对 FIFO 不该尝试 seek）。

### 4.4 `reinit_io_cache`（`mysys/mf_iocache.cc:328-392`）：READ ↔ WRITE 切换

**两条路径**：

**路径 A：快路径（复用缓冲区）**，条件 `!clear_cache && seek_offset >= pos_in_file && seek_offset <= my_b_tell(info)`：

- **W → R**：`read_end = write_pos`（刚写的就是有效数据）、`end_of_file = my_b_tell(info)`（**截断**：写入点之后的丢弃）、`seek_not_done = (file != -1)`。
- **R → W**：`write_end = write_buffer + buffer_length`、`end_of_file = ~(my_off_t)0`（**解除上限**）。

**路径 B：慢路径**：先 flush（除非 `clear_cache`），再重置所有指针。

**典型用例 1**：`IO_CACHE_binlog_cache_storage::begin()`（`binlog_ostream.cc:169`）—— `reinit_io_cache(&m_io_cache, READ_CACHE, 0, false, false)`，把写态 cache 切成读态。**数据在内存里就不用落盘**。

**典型用例 2（很巧）**：`truncate()`（`binlog_ostream.cc:100`）：

```cpp
  if (reinit_io_cache(&m_io_cache, WRITE_CACHE, offset, false,
                      offset < m_io_cache.pos_in_file /*clear_cache*/))
    return true;
  m_io_cache.end_of_file = m_max_cache_size;
```

注释解释了 `clear_cache` 的取法：

> *It is not really necessary to flush the data will be truncated into temporary file before truncating. And it may cause write failure. So set clear_cache to true if all data in cache will be truncated.*

即：**若要丢的数据已在盘上，就跳过 flush**——既省一次写，也避免 tmpdir 满时 truncate 反而失败。

**注意**：`truncate` 后必须**重新**设 `end_of_file = m_max_cache_size`，因为慢路径把它改成了 `~(my_off_t)0`。忘了这行就等于取消了 `max_binlog_cache_size` 限制。

---

## 核心实现五：延迟创建文件（file == -1 的妙用）

`mysys/mf_cache.cc` 的三个函数：

```c
bool open_cached_file(IO_CACHE *cache, const char *dir, const char *prefix,
                      size_t cache_size, myf cache_myflags) {
  cache->dir = dir ? my_strdup(...) : (char *)nullptr;
  cache->prefix = prefix ? my_strdup(...) : (char *)nullptr;
  cache->file_name = nullptr;
  cache->buffer = nullptr; /* Mark that not open */
  if (!init_io_cache(cache, -1, cache_size, WRITE_CACHE, 0L, false,
                     MYF(cache_myflags | MY_NABP))) {
    return false;
  }
  ...
}

bool real_open_cached_file(IO_CACHE *cache) {
  char name_buff[FN_REFLEN];
  int error = 1;
  if ((cache->file = mysql_file_create_temp(
           cache->file_key, name_buff, cache->dir, cache->prefix,
           (O_RDWR | O_TRUNC), UNLINK_FILE, MYF(MY_WME))) >= 0) {
    error = 0;
  }
  return error;
}

void close_cached_file(IO_CACHE *cache) {
  if (my_b_inited(cache)) {
    File file = cache->file;
    cache->file = -1; /* Don't flush data */
    (void)end_io_cache(cache);
    if (file >= 0) {
      (void)mysql_file_close(file, MYF(0));
    }
    ...
  }
}
```

**三个巧妙之处**：

**① `open_cached_file` 传 `file = -1`**：此时 IO_CACHE 是**纯内存写缓冲**，没有 fd。小事务 / 小排序 / 小 join **完全不产生临时文件**，零 open/write/unlink 系统调用。

**② `real_open_cached_file` 用 `UNLINK_FILE`**：文件创建后**立即 unlink**，只保留 fd：
- 目录里不可见（不污染 tmpdir）
- 进程崩溃不留下垃圾（内核在最后一个 fd 关闭时自动回收）
- `close_cached_file` 里不需要 `my_delete`

**③ `close_cached_file` 先把 `file` 置 -1**：因为 `end_io_cache` 内部有

```c
    if (info->file != -1) /* File doesn't exist */
      error = my_b_flush_io_cache(info, 1);
```

置 -1 就**跳过最后一次 flush**。对临时文件来说数据马上丢弃，flush 毫无意义，且可能因 tmpdir 满而报错。**这是"修改字段来短路下游逻辑"的技巧**（略 hacky 但有效）。

**`end_io_cache` 的另外两个细节**：

```c
  if ((pre_close = info->pre_close)) {
    (*pre_close)(info);
    info->pre_close = nullptr;
  }
  if (info->alloced_buffer) {
    info->alloced_buffer = false;
    if (info->file != -1)
      error = my_b_flush_io_cache(info, 1);
    ...
```

- **`pre_close` 回调在 flush 之前** —— 这样 `log_loaded_block` 能写出最后一个 `Append_block_log_event`。
- `pre_close` 置 nullptr 保证幂等（注释：*"This function is also safe to call twice with the same handle"*）。

---

## 核心实现六：不是文件的 IO_CACHE（NET / FIFO / 内存）

### 6.1 ★ `READ_NET`：把网络流伪装成文件

`sql/mf_iocache.cc:61-93`（全函数）：

```cpp
int _my_b_net_read(IO_CACHE *info, uchar *Buffer,
                   size_t Count [[maybe_unused]]) {
  ulong read_length;
  NET *net = current_thd->get_protocol_classic()->get_net();

  if (!info->end_of_file)
    return 1; /* because my_b_get (no _) takes 1 byte at a time */
  read_length = my_net_read(net);
  if (read_length == packet_error) {
    info->error = -1;
    return 1;
  }
  if (read_length == 0) {
    info->end_of_file = 0; /* End of file from client */
    return 1;
  }
  /* to set up stuff for my_b_get (no _) */
  info->read_end = (info->read_pos = net->read_pos) + read_length;
  Buffer[0] = info->read_pos[0]; /* length is always 1 */

  /*
    info->request_pos is used by log_loaded_block() to know the size
    of the current block.
    info->pos_in_file is used by log_loaded_block() too.
  */
  info->pos_in_file += read_length;
  info->request_pos = info->read_pos;

  info->read_pos++;

  return 0;
}
```

**四个技巧**：

**① `NET*` 每次从 `current_thd` 取**，不存进 IO_CACHE —— 这正是它必须住在 `sql/` 的原因。

**② 零拷贝借用 NET 缓冲**：
```cpp
  info->read_end = (info->read_pos = net->read_pos) + read_length;
```
IO_CACHE 的 `read_pos`/`read_end` **直接指向 `net->read_pos`**，不 memcpy。所以 `init_io_cache_ext` 对 READ_NET **不分配 buffer**（`if (type != READ_NET && type != WRITE_NET)`），且 `alloced_buffer` 保持 false。

**③ ★ `end_of_file` 被复用成"流还活着吗"标志**：非 0 = 有数据，0 = 客户端结束。这是 `end_of_file` 字段的**第三种语义**（见 [Misc](#misc)）。

**④ `Count` 被忽略，每次只读 1 字节**：因为 `READ_INFO` 用 `my_b_get()` 逐字节扫描（要做字符集/分隔符解析）。

**接线**（`sql/sql_load.cc:1389-1410`）：

```c
    if (init_io_cache(
            &cache, (get_it_from_net) ? -1 : file, 0,
            (get_it_from_net) ? READ_NET : (is_fifo ? READ_FIFO : READ_CACHE),
            0L, true, MYF(MY_WME))) { ... }
    else {
      /*
        init_io_cache() will not initialize read_function member
        if the cache is READ_NET. So we work around the problem with a
        manual assignment
      */
      need_end_io_cache = true;
      if (get_it_from_net) cache.read_function = _my_b_net_read;
      if (mysql_bin_log.is_open())
        cache.pre_read = cache.pre_close = (IO_CACHE_CALLBACK)log_loaded_block;
    }
```

### 6.2 `READ_FIFO`

只有两处特殊处理：**不探测文件大小**、**读时不检查 `end_of_file` 上界**（管道长度未知/在增长）。

### 6.3 纯内存用法

`PRINT_EVENT_INFO` 的三个 cache（`sql/log_event.cc:14366-14369`）：

```c
  myf const flags = MYF(MY_WME | MY_NABP);
  open_cached_file(&head_cache, nullptr, nullptr, 0, flags);
  open_cached_file(&body_cache, nullptr, nullptr, 0, flags);
  open_cached_file(&footer_cache, nullptr, nullptr, 0, flags);
```

`dir` 和 `prefix` 都传 `nullptr`，用途是 mysqlbinlog 打印事件时**分别缓冲 head / body / footer** 再按顺序拼接。因为内容很小，**几乎从不 spill 到磁盘**——是事实上的纯内存缓冲区。

### 6.4 回调钩子：`pre_read` / `pre_close` / `arg`

字段注释说明了唯一的生产用途：

> *Callbacks when the actual read I/O happens. These were added and are currently used for binary logging of LOAD DATA INFILE - when a block is read from the file, we create a block create/append event, and when IO_CACHE is closed, we create an end event.*

`log_loaded_block`（`sql/binlog.cc:3442`）挂在 `pre_read`/`pre_close` 上，每读到一个块就写 `Append_block_log_event`，结束时写 `End_load_query_log_event`。它用 `my_b_get_buffer_start(file)`（即 `request_pos`）和 `my_b_get_bytes_in_buffer(file)` 定位刚读进来的整块——这正是 `_my_b_net_read` 要维护 `request_pos` 的原因。

> **注意**：`pre_read` **只在 `_my_b_get` 里被调用**（`mf_iocache.cc:1219-1226`）。`_my_b_read` / `_my_b_read_r` / `_my_b_seq_read` / `_my_b_net_read` 都不调它。所以 **`post_read` 在 8.0 中从未被赋值**（grep 只有字段声明和那处调用）。

---

## 核心实现七：透明加解密

### 拦截点（AOP 式）

`mysys/mf_iocache.cc:1640-1693`：

```c
my_off_t mysql_encryption_file_seek(IO_CACHE *cache, my_off_t pos, int whence,
                                    myf flags) {
  if (cache->m_encryptor != nullptr) cache->m_encryptor->set_stream_offset(pos);
  if (cache->m_decryptor != nullptr) cache->m_decryptor->set_stream_offset(pos);
  return mysql_file_seek(cache->file, pos, whence, flags);
}

size_t mysql_encryption_file_read(IO_CACHE *cache, uchar *buffer, size_t count,
                                  myf flags) {
  size_t ret = mysql_file_read(cache->file, buffer, count, flags);
  if (ret != MY_FILE_ERROR && cache->m_decryptor != nullptr)
    cache->m_decryptor->decrypt(buffer, buffer, ret ? ret : count);
  return ret;
}
```

**三个机制**：

1. **统一拦截**：IO_CACHE 里所有真正的 I/O 都走这三个函数（22 处调用点），**没有一处直接调 `mysql_file_read/write`**。
2. **CTR 流密码 + `set_stream_offset`**：seek 时同步重置加密器/解密器的流偏移，使"文件偏移 N 的密文"永远对应"逻辑偏移 N 的明文"。**这让随机访问成为可能——CBC 之类分组链接模式做不到。**
3. **原地加解密**：`decrypt(buffer, buffer, ret)`，输入输出同一块内存。

### ★ 返回值的陷阱

```c
      if (!(flags & (MY_NABP | MY_FNABP))) {
        written = written + ret;
      }
      ...
    ret = written;
```

- 若 `flags` **带** `MY_NABP`（IO_CACHE 正是这么调的），`written` 始终是 **0**，函数最终 `return 0`。
- 为什么可以？因为带 `MY_NABP` 时 `mysql_file_write` 保证要么全写要么返回 `MY_FILE_ERROR`，调用方只关心 0 vs 非 0：
  ```c
  if (mysql_encryption_file_write(info, info->write_buffer, length,
                                  info->myflags | MY_NABP))
    info->error = -1;
  ```
  **返回 0 恰好被解读为"成功"** —— 脆弱但有效的约定。

### 临时文件加密是**每事务换密码**

`sql/binlog_ostream.cc:234-251`：

```cpp
bool IO_CACHE_binlog_cache_storage::setup_ciphers_password() {
  unsigned char password[Aes_ctr_encryptor::PASSWORD_LENGTH];
  ...
  /* Generate password, it is a random string. */
  if (my_rand_buffer(password, sizeof(password))) return true;
  ...
}
```

头文件注释解释：

> *This function is called by reset() that is called every time a transaction commits... The file password shall vary not only per temporary file, but also per transaction being committed within a single client connection.*

还有一个**动态开关约束**（`binlog_ostream.cc:61-91`）：只有当临时文件为空时才能切换加密状态，否则已写入的密文/明文会与新 cipher 不匹配。

### ★ 两套加密机制要分清

| | `IO_CACHE::m_encryptor` | `Binlog_encryption_ostream` |
|---|---|---|
| 保护对象 | binlog cache 的**临时文件** | **binlog 文件本身** |
| 层次 | mysys 层，IO_CACHE 内部 | sql 层，ostream 装饰器 |
| 密码生命周期 | 每事务重新生成 | 每文件一个，存在 header 里 |

---

## 核心实现八：★ server 层 I/O 的类继承体系

### 8.1 完整继承树

```
【输出侧 / ostream】
────────────────────────────────────────────────────────────────────
                        ┌───────────────────────────┐
                        │      Basic_ostream        │  (纯虚: write)
                        │   basic_ostream.h:37      │
                        └─────────────┬─────────────┘
              ┌───────────────────────┼──────────────────────┬─────────────────┐
              │                       │                      │                 │
   ┌──────────▼──────────┐  ┌─────────▼──────────┐  ┌────────▼─────────┐  ┌────▼─────────────────┐
   │ Truncatable_ostream │  │ Compressed_ostream │  │ Binlog_cache_    │  │ StringBuffer_ostream │
   │ (抽象) +truncate    │  │ (终结点,不落盘)    │  │ storage          │  │ basic_ostream.h:160  │
   │        +seek        │  │ basic_ostream.h:173│  │ :Basic_ostream   │  └─────────────────────┘
   │        +flush +sync │  └────────────────────┘  │ binlog_ostream   │
   │ basic_ostream.h:58  │                          │ .h:174           │
   └──────┬──────────────┘                          └──────┬───────────┘
          │                                                │ 组合(!) 非继承:
          ├───────────────────────┐                        │ m_file: IO_CACHE_binlog_
          │                       │                        │         cache_storage
   ┌──────▼────────────┐  ┌───────▼──────────────┐         │ m_pipeline_head:
   │ IO_CACHE_ostream  │  │Binlog_encryption_    │         │   Truncatable_ostream*
   │ (真正落盘终点)    │  │ostream (装饰器)      │         │
   │ basic_ostream.h:98│  │binlog_ostream.h:246  │         │
   │ m_io_cache:       │  │m_down_ostream:       │         │
   │   IO_CACHE        │  │  unique_ptr<         │         │
   │                   │  │  Truncatable_ostream>│         │
   └───────────────────┘  └──────────────────────┘         │
                                                            │
                                            ┌───────────────▼──────────────────┐
                                            │ IO_CACHE_binlog_cache_storage   │
                                            │ :Truncatable_ostream            │
                                            │ binlog_ostream.h:70             │
                                            │ m_io_cache: IO_CACHE(内存+临时文件)│
                                            └─────────────────────────────────┘

【门面】
────────────────────────────────────────────────────────────────────
   ┌────────────────────────────────────────────────────┐
   │ MYSQL_BIN_LOG::Binlog_ofile : Basic_ostream        │  sql/binlog.cc:375
   │  m_pipeline_head: unique_ptr<Truncatable_ostream>  │
   │  m_position / m_encrypted_header_size / m_encrypted│
   │  ★ 门面：组装 pipeline + 维护 binlog 位点          │
   └────────────────────────────────────────────────────┘

【输入侧 / istream】
────────────────────────────────────────────────────────────────────
                    ┌──────────────────────┐
                    │    Basic_istream     │  (纯虚: read)  :33
                    └──────────┬───────────┘
                 ┌─────────────┴────────────┐
                 │                          │
      ┌──────────▼───────────┐   ┌──────────▼──────────┐
      │Basic_seekable_istream│   │   Stdin_istream     │  :132
      │ +seek()=0,length()=0 │   │ m_io_cache:IO_CACHE │
      │  basic_istream.h:58  │   └─────────────────────┘
      └──────────┬───────────┘
      ┌──────────┴───────────────┬─────────────────────────┐
      │                          │                         │
┌─────▼──────────────┐  ┌────────▼────────────┐  ┌─────────▼──────────┐
│  IO_CACHE_istream  │  │Binlog_encryption_   │  │ Basic_binlog_ifile │
│        :88         │  │istream (装饰器):118 │  │       :157         │
└────────────────────┘  └─────────────────────┘  └────────────────────┘
```

### 8.2 `Basic_ostream`：只有一个虚函数

```cpp
class Basic_ostream {
 public:
  virtual bool write(const unsigned char *buffer, my_off_t length) = 0;
  virtual ~Basic_ostream() = default;
};
```

契约里最关键的一句（注释）：**"It will never returns false when partial data is written"** —— 要么全写进去返回 false，要么出错返回 true，**不允许部分成功**。

### 8.3 `Truncatable_ostream`：加上 seek/truncate/flush/sync

```cpp
class Truncatable_ostream : public Basic_ostream {
 public:
  virtual bool truncate(my_off_t offset) = 0;
  virtual bool seek(my_off_t offset) = 0;
  virtual bool flush() = 0;
  virtual bool sync() = 0;
  ~Truncatable_ostream() override = default;
};
```

> **设计不对称**：输出侧**没有** `Basic_seekable_ostream`——MySQL 认为输出流天然分两类：可 seek/truncate 的（文件-like）和不可的（网络-like、压缩器）。输入侧才做了拆分（`Basic_istream` / `Basic_seekable_istream`）。

### 8.4 `IO_CACHE_ostream`：落盘终点

```cpp
class IO_CACHE_ostream : public Truncatable_ostream {
 public:
  bool open(PSI_file_key log_file_key, const char *file_name, myf flags);
  bool close();
  bool write(const unsigned char *buffer, my_off_t length) override;
  bool seek(my_off_t offset) override;
  bool truncate(my_off_t offset) override;
  bool flush() override;
  bool sync() override;
 private:
  IO_CACHE m_io_cache;
};
```

实现要点（`sql/basic_ostream.cc:31-90`）：

```cpp
bool IO_CACHE_ostream::open(...) {
  File file = -1;
  if ((file = mysql_file_open(log_file_key, file_name, O_CREAT | O_WRONLY,
                              MYF(MY_WME))) < 0)
    return true;
  if (init_io_cache(&m_io_cache, file, IO_SIZE, WRITE_CACHE, 0, false, flags)) {
    mysql_file_close(file, MYF(0));
    return true;
  }
  return false;
}

bool IO_CACHE_ostream::seek(my_off_t offset) {
  return reinit_io_cache(&m_io_cache, WRITE_CACHE, offset, false, true);
}

bool IO_CACHE_ostream::truncate(my_off_t offset) {
  if (my_chsize(m_io_cache.file, offset, 0, MYF(MY_WME))) return true;
  reinit_io_cache(&m_io_cache, WRITE_CACHE, offset, false, true);
  return false;
}

bool IO_CACHE_ostream::sync() {
  return mysql_file_sync(m_io_cache.file, MYF(MY_WME)) != 0;
}
```

- `seek` / `truncate` **都用 `reinit_io_cache`**，且 `clear_cache=true`（强制丢弃缓冲）。
- `truncate` 的 `my_chsize` + `reinit` **两步必须配对**，否则 IO_CACHE 内部位置与文件大小脱节。
- **★ `sync()` 不会帮你 `flush()`**。头文件注释明说：*"It doesn't check and flush any remaining data still left in IO_CACHE's buffer. So a call to flush() is necessary"*。所以上层有：
  ```cpp
  bool flush_and_sync() { return flush() || sync(); }
  ```

### 8.5 `Binlog_encryption_ostream`：装饰者 + 偏移翻译

```cpp
class Binlog_encryption_ostream : public Truncatable_ostream {
 public:
  bool open(std::unique_ptr<Truncatable_ostream> down_ostream);
  bool open(std::unique_ptr<Truncatable_ostream> down_ostream,
            std::unique_ptr<Rpl_encryption_header> header);
  bool write(const unsigned char *buffer, my_off_t length) override;
  bool truncate(my_off_t offset) override;
  bool seek(my_off_t offset) override;
  bool flush() override;
  bool sync() override;
 private:
  std::unique_ptr<Truncatable_ostream> m_down_ostream;
  std::unique_ptr<Rpl_encryption_header> m_header;
  std::unique_ptr<Stream_cipher> m_encryptor;
};
```

**写**：分块加密（栈上 2 KB 缓冲），逐块转发：

```cpp
bool Binlog_encryption_ostream::write(const unsigned char *buffer,
                                      my_off_t length) {
  const int ENCRYPT_BUFFER_SIZE = 2048;
  unsigned char encrypt_buffer[ENCRYPT_BUFFER_SIZE];
  const unsigned char *ptr = buffer;

  while (length > 0) {
    int encrypt_len =
        std::min(length, static_cast<my_off_t>(ENCRYPT_BUFFER_SIZE));
    if (m_encryptor->encrypt(encrypt_buffer, ptr, encrypt_len)) { ... return true; }
    if (m_down_ostream->write(encrypt_buffer, encrypt_len)) return true;
    ptr += encrypt_len;
    length -= encrypt_len;
  }
  return false;
}
```

**★ 偏移翻译是装饰者的价值增量**：

```cpp
bool Binlog_encryption_ostream::seek(my_off_t offset) {
  if (m_down_ostream->seek(m_header->get_header_size() + offset)) return true;
  return m_encryptor->set_stream_offset(offset);
}
```

因为加密文件前面多了一段 `Rpl_encryption_header`，**逻辑偏移 ≠ 物理偏移**。装饰者负责换算，同时重置 cipher 的流偏移。

> **为什么必须用 CTR**：`set_stream_offset()` 能直接跳转 counter，所以才能支持 seek/truncate。CBC 之类分组链接模式做不到。

### 8.6 `Binlog_ofile`：门面（Facade）

```cpp
class MYSQL_BIN_LOG::Binlog_ofile : public Basic_ostream {
 public:
  bool open(PSI_file_key log_file_key, const char *binlog_name, myf flags,
            bool existing = false) {
    std::unique_ptr<IO_CACHE_ostream> file_ostream(new IO_CACHE_ostream);
    if (file_ostream->open(log_file_key, binlog_name, flags)) return true;
    m_pipeline_head = std::move(file_ostream);

    /* Setup encryption for new files if needed */
    if (!existing && rpl_encryption.is_enabled()) {
      std::unique_ptr<Binlog_encryption_ostream> encrypted_ostream(
          new Binlog_encryption_ostream());
      if (encrypted_ostream->open(std::move(m_pipeline_head))) return true;
      m_encrypted_header_size = encrypted_ostream->get_header_size();
      m_pipeline_head = std::move(encrypted_ostream);
    }
    return false;
  }
 private:
  my_off_t m_position = 0;
  int m_encrypted_header_size = 0;
  std::unique_ptr<Truncatable_ostream> m_pipeline_head;
  bool m_encrypted = false;
};
```

**它是门面的四点证据**：

1. `open()` 是**工厂 + 组装器**：先造 `IO_CACHE_ostream`，再按需把 `Binlog_encryption_ostream` 套上去。
2. 它只继承 `Basic_ostream`，但通过 `m_pipeline_head` 转发 `truncate`/`seek`/`flush`/`sync`。
3. 它维护**门面专属状态** `m_position`（逻辑位点，不含加密头）。
4. `get_real_file_size()` 体现翻译职责：
   ```cpp
   my_off_t get_real_file_size() { return m_position + m_encrypted_header_size; }
   ```
   注释说：`position()` 是明文事件流视角，`get_real_file_size()` 是物理文件视角。

**流水线形态**：

```
不加密:  Binlog_ofile ──> IO_CACHE_ostream ──> IO_CACHE ──> binlog 文件
加密:    Binlog_ofile ──> Binlog_encryption_ostream ──> IO_CACHE_ostream ──> IO_CACHE ──> 文件
                          (seek/truncate 做偏移翻译 + CTR counter 重置)
```

### 8.7 `IO_CACHE_binlog_cache_storage`：接口的"部分实现"

```cpp
class IO_CACHE_binlog_cache_storage : public Truncatable_ostream {
 public:
  bool write(const unsigned char *buffer, my_off_t length) override;
  bool truncate(my_off_t offset) override;
  /* binlog cache doesn't need seek operation. Setting true to return error */
  bool seek(my_off_t offset [[maybe_unused]]) override { return true; }
  bool flush() override { return false; }
  bool sync() override { return false; }
  ...
 private:
  IO_CACHE m_io_cache;
  my_off_t m_max_cache_size = 0;
};
```

**三个"故意不实现"**：

- `seek()` **恒返回 true（错误）** —— binlog cache 是纯顺序的。
- `flush()` / `sync()` **恒返回 false（成功）** —— 是 no-op，因为临时文件不需要持久化。
- `end_of_file` 被当成容量上限：
  ```cpp
  bool IO_CACHE_binlog_cache_storage::open(const char *dir, const char *prefix,
                                           my_off_t cache_size,
                                           my_off_t max_cache_size) {
    if (open_cached_file(&m_io_cache, dir, prefix, cache_size, MYF(MY_WME)))
      return true;
    if (rpl_encryption.is_enabled()) enable_encryption();
    m_max_cache_size = max_cache_size;
    /* Set the max cache size for IO_CACHE */
    m_io_cache.end_of_file = max_cache_size;
    return false;
  }
  ```

`begin` / `next` 是**读侧的零拷贝接口**：

```cpp
bool IO_CACHE_binlog_cache_storage::next(unsigned char **buffer,
                                         my_off_t *length) {
  my_b_fill(&m_io_cache);
  *buffer = m_io_cache.read_pos;
  *length = my_b_bytes_in_cache(&m_io_cache);
  m_io_cache.read_pos = m_io_cache.read_end;   // ★ 直接吞掉整块
  return m_io_cache.error;
}
```

### 8.8 `Binlog_cache_storage`：继承 + 组合双用

```cpp
class Binlog_cache_storage : public Basic_ostream {
 public:
  bool write(const unsigned char *buffer, my_off_t length) override {
    assert(m_pipeline_head != nullptr);
    return m_pipeline_head->write(buffer, length);
  }
  bool copy_to(Basic_ostream *ostream, bool *ostream_error = nullptr) {
    return stream_copy(&m_file, ostream, ostream_error);
  }
 private:
  Truncatable_ostream *m_pipeline_head = nullptr;
  IO_CACHE_binlog_cache_storage m_file;
};
```

- **它继承 `Basic_ostream`（提供 write），同时组合 `IO_CACHE_binlog_cache_storage m_file`**。
- `m_pipeline_head` 目前**永远指向 `&m_file`**：
  ```cpp
  bool Binlog_cache_storage::open(my_off_t cache_size, my_off_t max_cache_size) {
    const char *LOG_PREFIX = "ML";
    if (m_file.open(mysql_tmpdir, LOG_PREFIX, cache_size, max_cache_size))
      return true;
    m_pipeline_head = &m_file;
    return false;
  }
  ```
  存在的唯一理由是**向前兼容**（将来加压缩/加密装饰器时只改 `open()` 一处）。头文件注释明说这个意图。

### 8.9 全套类清单

| 类 | 基类 | 位置 |
|---|---|---|
| `Basic_ostream` | — (抽象) | `sql/basic_ostream.h:37` |
| `Truncatable_ostream` | `Basic_ostream` | `sql/basic_ostream.h:58` |
| `IO_CACHE_ostream` | `Truncatable_ostream` | `sql/basic_ostream.h:98` |
| `StringBuffer_ostream<N>` | `Basic_ostream` + `StringBuffer<N>` | `sql/basic_ostream.h:160` |
| `Compressed_ostream` | `Basic_ostream` | `sql/basic_ostream.h:173` |
| `IO_CACHE_binlog_cache_storage` | `Truncatable_ostream` | `sql/binlog_ostream.h:70` |
| `Binlog_cache_storage` | `Basic_ostream` | `sql/binlog_ostream.h:174` |
| `Binlog_encryption_ostream` | `Truncatable_ostream` | `sql/binlog_ostream.h:246` |
| `MYSQL_BIN_LOG::Binlog_ofile` | `Basic_ostream` | `sql/binlog.cc:375` |
| `Basic_istream` | — (抽象) | `sql/basic_istream.h:33` |
| `Basic_seekable_istream` | `Basic_istream` | `sql/basic_istream.h:58` |
| `IO_CACHE_istream` | `Basic_seekable_istream` | `sql/basic_istream.h:88` |
| `Stdin_istream` | `Basic_istream` | `sql/basic_istream.h:132` |
| `Binlog_encryption_istream` | `Basic_seekable_istream` | `sql/binlog_istream.h:118` |
| `Basic_binlog_ifile` | `Basic_seekable_istream` | `sql/binlog_istream.h:157` |

> **`Compressed_ostream` 不是装饰器**：它只继承 `Basic_ostream`，输出目的地是 `Managed_buffer_sequence`（内存 buffer 序列），是**终结点**。它由 `Binlog_cache_compressor`（`binlog.cc:2055`，**不在继承体系里**，是 RAII 驱动器）使用。

---

## 核心实现九：★ 这套体系里的 C++ 运用

> `IO_CACHE` 本身是 C 风格结构，但它外面那层 ostream/istream 体系用了不少**并不初级**的 C++：极窄的纯虚接口、`unique_ptr` 所有权转移链、`= delete` 的深层理由、非类型模板参数、duck-typing 模板、NVI 模板方法、以及"C 结构如何被安全地包进 C++ 类"。这一章逐项讲清**用了什么、为什么、代价是什么**。

### 9.1 接口设计：极窄的抽象基类

```cpp
class Basic_ostream {
 public:
  virtual bool write(const unsigned char *buffer, my_off_t length) = 0;
  virtual ~Basic_ostream() = default;
};
```

**只有一个虚函数**。这个极窄接口是整个体系能被"装饰器无限嵌套"的前提——任何"能被写入的字节汇"都能接入。

**虚析构是刚需**：装饰器链靠 `unique_ptr<Truncatable_ostream>` 持有下一级，通过基类指针析构时必须正确调用派生类析构。

**★ 为什么输出侧没有 `Basic_seekable_ostream`（输入侧却有 `Basic_seekable_istream`）**

源码里没有写明设计意图，但从代码可以推断（标注为推断）：

1. **输出侧的 seek 从不单独使用**。binlog 需要随机写只有两种场景：① 回改文件头（清 `LOG_EVENT_BINLOG_IN_USE_F`、重加密 header）；② 回滚写坏的事务（`truncate`）。**两者必然同时需要 truncate**，所以 `seek + truncate + flush + sync` 被打包成一个 `Truncatable_ostream`，不拆成两层。
2. **输入侧的 seek 与 `length()` 成对出现**（`Basic_seekable_istream` 同时有这两个，注释明说"调用者应该用 `length()` 检查越界"），而输入侧没有 truncate 概念，故单独成层。
3. **门面刻意隐藏 seekable**：`Binlog_ofile : public Basic_ostream` 只对外承诺 `write`，`truncate`/`flush`/`sync` 都是**非虚的新方法**——seekable 只是 pipeline 内部实现细节。

**代价（接口污染的直接证据）**：

```cpp
// sql/binlog_ostream.h:96-98
  /* binlog cache doesn't need seek operation. Setting true to return error */
  bool seek(my_off_t offset [[maybe_unused]]) override { return true; }
```

binlog cache 是纯顺序的，却被迫实现一个返回错误的 `seek()`。这就是"把 seek 塞进 `Truncatable_ostream`"的代价。

### 9.2 所有权与生命周期：`unique_ptr` 的装饰器链

`Binlog_ofile::open()` 是整条链的组装现场（`sql/binlog.cc:414-428`）：

```cpp
    std::unique_ptr<IO_CACHE_ostream> file_ostream(new IO_CACHE_ostream);
    if (file_ostream->open(log_file_key, binlog_name, flags)) return true;

    m_pipeline_head = std::move(file_ostream);          // ① 派生→基的转换

    if (!existing && rpl_encryption.is_enabled()) {
      std::unique_ptr<Binlog_encryption_ostream> encrypted_ostream(
          new Binlog_encryption_ostream());
      if (encrypted_ostream->open(std::move(m_pipeline_head))) return true;  // ② 所有权移交
      m_encrypted_header_size = encrypted_ostream->get_header_size();
      m_pipeline_head = std::move(encrypted_ostream);   // ③ 装饰器接管到门面
    }
```

**① 派生→基的 `unique_ptr` 转换**：`unique_ptr<IO_CACHE_ostream>` 移动赋给 `unique_ptr<Truncatable_ostream>`。这靠 `unique_ptr` 的转换构造函数完成，其**安全性前提是基类有虚析构**——正是 9.1 里的 `virtual ~Basic_ostream() = default`。

**② 按值传参表达"接管所有权"**（sink 惯用法）：

```cpp
  bool open(std::unique_ptr<Truncatable_ostream> down_ostream);
```

实参 `m_pipeline_head` 被 `std::move` 移走后**变成 nullptr**，所有权短暂"悬在形参里"。`unique_ptr` 不可拷贝，所以 `std::move` 是**类型层面强制的**——一个下游流不可能被两个装饰器同时持有。

**③ 析构顺序如何保证**：

```cpp
void Binlog_encryption_ostream::close() {
  m_encryptor.reset(nullptr);
  m_header.reset(nullptr);
  m_down_ostream.reset(nullptr);
}
```

显式 `close()` 被调用在成员逆序析构之前，所以实际顺序是 **先关加密器、再关下游流**——这是正确顺序（不能先关文件再关 cipher）。

**保证**：无论正常析构还是错误返回路径，`unique_ptr` 成员都保证下游流被关闭、fd 被释放。

**代价**：`open()` 失败时，形参 `down_ostream` 被销毁 ⇒ 原来的 `IO_CACHE_ostream` 被析构、文件被关闭，而 `m_pipeline_head` 已为空。调用方需靠 `is_open()`（`return m_pipeline_head != nullptr;`）兜底。

**★ `shared_ptr` 只有一处**——`Compressed_ostream` 的压缩器：

```cpp
  using Compressor_ptr_t = std::shared_ptr<Compressor_t>;
```

为什么用 shared：`Compressor` 是**不可拷贝不可移动**的抽象类，且 zstd 的 `ZSTD_CStream` 上下文很贵，**需要跨事务复用**；而 `Compressed_ostream` 是**栈上的短命对象**（`Compressed_ostream stream{m_compressor, m_managed_buffer_sequence};`），不能"拿走"压缩器，只能共享。

**★ 同一概念在两个类里的形态不同**（很有意思的对比）：

| 类 | `m_pipeline_head` 的类型 | 原因 |
|---|---|---|
| `Binlog_ofile` | `std::unique_ptr<Truncatable_ostream>` | 真的有多级装饰器，需要动态分配与转移 |
| `Binlog_cache_storage` | `Truncatable_ostream *`（裸指针） | 目前只有一级，真实对象是**按值成员** `IO_CACHE_binlog_cache_storage m_file;`，用 `unique_ptr` 会 double-free 且引入无谓堆分配。名字 `m_pipeline_head` 是为将来预留的扩展点 |

### 9.3 `= delete` 的真实理由（不是随手写的）

这批类里 16 处 `= delete`，两个真实原因：

**① `IO_CACHE` 是被 C 风格管理的资源句柄，拷贝即灾难**

```c
  uchar **current_pos{nullptr}, **current_end{nullptr};   // ← 二级指针
  mysql_mutex_t append_buffer_lock;                        // ← 内嵌 mutex，本身不可拷贝
  uchar *buffer{nullptr};                                  // ← my_malloc 出来的
  Stream_cipher *m_encryptor = nullptr;                    // ← 裸指针，end_io_cache 会 delete
```

**最致命的是二级指针**：`current_pos` 指向 `IO_CACHE` **自身**的 `write_pos`。按位拷贝一个对象，副本的 `current_pos` 会指向**原对象**的 `write_pos`——两个对象互相踩。

**② 防止对象切片**：这些都是多态类，按值拷贝到基类形参会丢掉派生部分。

**副作用（容易忽略）**：声明了拷贝构造/赋值（即使是 deleted），**移动构造/移动赋值也不会被隐式生成**。所以这些类"既不能拷也不能移"。需要明确表达时得写满 4 条：

```cpp
  Compressed_ostream() = delete;
  Compressed_ostream(const Compressed_ostream &) = delete;
  Compressed_ostream &operator=(const Compressed_ostream &) = delete;
```

**没有自定义移动操作**——全族 grep 无 `&&)`，所有移动都作用在 `unique_ptr`/`shared_ptr` 上，即**依赖标准库智能指针的移动语义**。

### 9.4 模板与泛型

**① `StringBuffer_ostream<N>`：非类型模板参数 + 多重继承**

```cpp
template <int BUFFER_SIZE>
class StringBuffer_ostream : public Basic_ostream,
                             public StringBuffer<BUFFER_SIZE> {
 public:
  bool write(const unsigned char *buffer, my_off_t length) override {
    return StringBuffer<BUFFER_SIZE>::append(
        reinterpret_cast<const char *>(buffer), length);
  }
};
```

- `int BUFFER_SIZE` 是**非类型模板参数**：栈上缓冲大小编译期确定，小缓冲场景零堆分配（对比 `IO_CACHE_ostream` 用堆缓存）。
- 多重继承 = 接口继承（`Basic_ostream`）+ 实现继承（`StringBuffer<N>`）。
- `StringBuffer<BUFFER_SIZE>::append(...)` 的**限定名是必需的**——模板里查找依赖基类的名字需要显式限定。

**② `stream_copy`：duck-typing 模板（零约束、零开销）**

```cpp
template <class ISTREAM, class OSTREAM>
bool stream_copy(ISTREAM *istream, OSTREAM *ostream,
                 bool *ostream_error = nullptr) {
  bool ret = istream->begin(&buffer, &length);
  while (!ret && length > 0) {
    if (ostream->write(buffer, length)) { ... return true; }
    ret = istream->next(&buffer, &length);
  }
  return ret;
}
```

**它不靠继承约束类型，而是靠结构化类型（duck typing）**：

- `ISTREAM` 只要有 `begin()` / `next()`——而这两个是 `IO_CACHE_binlog_cache_storage` 的**非虚公有方法**，根本不在 `Basic_istream` 接口里。这是**刻意的零拷贝设计**：`begin`/`next` 直接把 `m_io_cache.read_pos` 交出去。
- `OSTREAM` 只要有 `write()`。所以 `Truncatable_ostream*` 和 `Basic_ostream*` 都能传（压缩场景传的就是 `Compressed_ostream*`，它只有 `Basic_ostream` 一个基类）。

**保证**：零虚调用开销（编译期绑定）+ 零内存拷贝（直接暴露内部缓冲指针）。
**代价**：**没有任何约束表达**（C++20 Concepts 之前的标准做法）。传错类型会得到一屏模板实例化错误；契约只写在注释里。

**③ `Aes_ctr_cipher<TYPE>`：用模板参数代替运行期分支**

```cpp
enum class Cipher_type : int { ENCRYPT = 0, DECRYPT = 1 };

template <Cipher_type TYPE>
class Aes_ctr_cipher : public Stream_cipher { ... };

typedef class Aes_ctr_cipher<Cipher_type::ENCRYPT> Aes_ctr_encryptor;
typedef class Aes_ctr_cipher<Cipher_type::DECRYPT> Aes_ctr_decryptor;
```

加/解密两套代码路径编译期分离，**零运行期分支开销**。

**④ 模板模板参数（policy-based 设计）**

```cpp
template <class IFILE, class EVENT_DATA_ISTREAM,
          template <class> class EVENT_OBJECT_ISTREAM, class ALLOCATOR>
class Basic_binlog_file_reader : public IBasic_binlog_file_reader { ... };
```

`template <class> class EVENT_OBJECT_ISTREAM` 是模板模板参数。为了让模板之间还能共享多态，另配一个纯抽象接口 `IBasic_binlog_file_reader`——**编译期 policy + 运行期接口双轨**。

**⑤ `Managed_buffer_sequence_t` 的 using 链**（类型别名一路传递）

```
Compressed_ostream::Managed_buffer_sequence_t
  = Compressor::Managed_buffer_sequence_t
  = mysqlns::buffer::Managed_buffer_sequence<>          // 默认 <unsigned char, std::vector>
  → 继承 Rw_buffer_sequence<unsigned char, std::vector>
```

注意 `using typename Base::X;` 语法（C++11 起允许引入依赖基类的类型成员），以及源码里那句跨编译器兼容的痕迹：

```cpp
  // Would prefer to use:
  // using typename Rw_buffer_sequence_t::Buffer_sequence_t;
  // But that doesn't compile on Windows (maybe a compiler bug).
```

### 9.5 属性与现代惯用法

**① `[[NODISCARD]]` 是宏，不是原生 attribute**

```cpp
  [[NODISCARD]] bool write(const unsigned char *buffer, my_off_t length) override;
```

```cpp
// libbinlogevents/include/nodiscard.h:44-48
#ifdef __GNUC__
#define NODISCARD nodiscard, gnu::warn_unused_result
#else
#define NODISCARD nodiscard
#endif
```

**为什么不用原生 `[[nodiscard]]`**：gcc bug 84476 会让 `[[nodiscard]]` 在某些场景失效，而 `gnu::warn_unused_result` 不受影响；MSVC 不认识后者会告警，所以只在 `__GNUC__` 下加。这是"**标准 attribute + 编译器 attribute 双写**"的实战技巧。

> 全类族中原生小写 `[[nodiscard]]`：**0 处**。

**② `[[maybe_unused]]` 主要用来对抗 `#ifdef HAVE_PSI_INTERFACE`**

```cpp
    PSI_file_key log_file_key [[maybe_unused]],
```

这些形参在 PSI 未编译进来时不存在（被 `#ifdef` 包住），但函数体仍会用到。比旧式 `(void)var;` 更清晰，且编译器保证不告警。

**③ `noexcept` 一处都没有——这是有理由的**

MySQL 服务端**基本不使用 C++ 异常**（错误一律 `bool` 返回值 + `my_error()`）；而且这批类的析构会调 `close()` → `end_io_cache()` → 可能失败的 I/O，标 `noexcept` 反而是撒谎。

**④ `enum class` vs unscoped enum**

| 用法 | 例子 |
|---|---|
| `enum class` | `enum class Cipher_type : int`（**带底层类型**，用作模板实参）、`Keyring_status`、`Key_rotation_step` |
| unscoped | **`cache_type` 是普通 enum**（`include/my_sys.h:282`）——IO_CACHE 保持了 C 风格 |
| **带 TODO 的** | `binary_log::transaction::compression::type` 源码里明写：`// Todo: use enum class and a more specific name.` |

> 这一条很能说明 MySQL 8.0 的现状：**是 C++17 代码库，但仍保留大量 C 风格遗留，改造是渐进的**。

**⑤ `std::pair<bool, std::string>` —— 无异常下的"值 + 错误串"**

```cpp
  std::pair<bool, std::string> reencrypt();
```

只有这一处。为什么不用异常：服务端不用异常，而"失败 + 原因字符串"需要两个返回值。代价：`.first`/`.second` 可读性差，且只有 `first==true` 时 `second` 才有意义（契约写在注释里）。

### 9.6 多态的另外三处运用

**① `Stream_cipher` + `Aes_ctr_cipher<TYPE>` + 工厂**

```cpp
class Stream_cipher {
 public:
  virtual ~Stream_cipher() = default;
  virtual bool open(const Key_string &password, int header_size) = 0;
  virtual bool encrypt(unsigned char *dest, const unsigned char *src, int length) = 0;
  virtual bool decrypt(unsigned char *dest, const unsigned char *src, int length) = 0;
  virtual bool set_stream_offset(uint64_t offset) = 0;   // ← 关键：CTR 才能这么做
};
```

`Aes_ctr::get_encryptor()` / `get_decryptor()` 返回 `unique_ptr<Stream_cipher>`（工厂）。

> **`set_stream_offset` 是 CTR 流密码的专属能力**——所以它出现在抽象基类里，等于把"必须用流密码"这个约束编码进了接口。

**② `Compressor`：NVI（Non-Virtual Interface）/ 模板方法模式**

```cpp
class Compressor {
 public:
  [[NODISCARD]] Compress_status compress(Managed_buffer_sequence_t &out);   // 公开非虚
  template <class Input_char_t>
  void feed(const Input_char_t *input_data, Size_t input_size) {
    feed_char_t(reinterpret_cast<const Char_t *>(input_data), input_size);   // 模板包装
  }
 private:
  [[NODISCARD]] virtual Compress_status do_compress(Managed_buffer_sequence_t &out) = 0;  // 私有纯虚
  virtual void do_feed(const Char_t *input_data, Size_t input_size) = 0;
};
```

- **公开接口定死算法骨架，可变部分下沉到私有虚函数**——标准 NVI。
- 模板 `feed` 接受任意字符类型（`char` / `unsigned char`），内部统一 `reinterpret_cast` 后调私有虚函数。
- 实现只有 **zstd** 和 **none**（8.0.39 **没有 lz4**），工厂用 `std::make_unique` 构造。

**③ `Rpl_encryption_header`：抽象工厂 + 版本化二进制格式**

8 个纯虚函数（`serialize` / `deserialize` / `get_encryptor` / `get_decryptor` / ...），两个静态工厂：

```cpp
  static std::unique_ptr<Rpl_encryption_header> get_header(Basic_istream *istream);
  static std::unique_ptr<Rpl_encryption_header> get_new_default_header();
```

`get_header()` 从流里读版本号 → 实例化对应版本子类 → 委托反序列化。**把 `switch(version)` 关在工厂里**，是版本化格式 + 多态的经典组合。

### 9.7 ★ C 与 C++ 的边界：C 结构如何被安全地包进 C++ 类

这是整章最有价值的一节——它解释了"为什么一个 C 结构能被 C++ 类当值成员持有"。

**① 组合而非继承 + NSDMI**

```cpp
class IO_CACHE_ostream : public Truncatable_ostream {
 private:
  IO_CACHE m_io_cache;          // ← 值成员，不是指针
};
```

**这个包装能成立，是因为 `IO_CACHE` 已经被"半 C++ 化"了**——它的每个成员都带 NSDMI（C++11 in-class member initializer）：

```c
  my_off_t pos_in_file{0};
  uchar *buffer{nullptr};
  File file{-1};
  cache_type type{TYPE_NOT_SET};
  int (*read_function)(IO_CACHE *, uchar *, size_t){nullptr};
```

于是：

- `IO_CACHE_ostream() = default;` 能生成可用的默认构造（**所有指针/字段被初始化为 0/nullptr**，而不是未定义的垃圾值）；
- `my_b_inited()` 才能把 "buffer != nullptr" 当作"是否已经 init"的判据。

> **这是"用 C++ 的 NSDMI 给 C 结构赋予可平凡默认构造能力"的教科书例子。** 没有 NSDMI，C 结构放进 C++ 类就必须手写构造函数逐字段初始化。

**② `my_b_inited()` 充当"构造是否成功"的运行时状态位**

```c
inline void my_b_clear(IO_CACHE *info) { info->buffer = nullptr; }
inline bool my_b_inited(const IO_CACHE *info) { return info->buffer != nullptr; }
```

- `init_io_cache()` 成功 ⇒ `buffer != nullptr` ⇒ "已构造"
- `end_io_cache()` 后（它把 `buffer` 置空）⇒ "已析构"

于是 C++ 的析构函数可以写成：

```cpp
bool IO_CACHE_ostream::close() {
  if (my_b_inited(&m_io_cache)) {        // ← 模拟"只在构造成功时才析构"
    int ret = end_io_cache(&m_io_cache);
    ret |= mysql_file_close(m_io_cache.file, MYF(MY_WME));
    return ret != 0;
  }
  return false;
}
```

而每个成员方法开头用 `assert(my_b_inited(&m_io_cache));` 表达前置条件"对象已 open"。

**代价（两阶段构造的固有问题）**：

- 构造 + `open()` 分离，中间态对象是"活着但不可用"的，**C++ 无法强制调用 `open()`**；
- **NDEBUG 下 `assert` 消失** ⇒ release 版本中"未 open 就 write"是 UB。

> 现代 C++ 的做法会是 `std::optional<IO_CACHE>` 或"构造里做完初始化 + 失败抛异常"。MySQL 保留 C 风格是因为 `IO_CACHE` 要被 mysys / MyISAM 这些**纯 C 消费者**共用。

### 9.8 RAII 的破损处（诚实记录）

| 破损 | 位置与说明 |
|------|-----------|
| **delete 后不置空** | `end_io_cache()` 里 `delete info->m_encryptor;` 后**没有置 nullptr**（对比同一函数里 `info->buffer = nullptr` 是置了的）。而 `IO_CACHE_binlog_cache_storage::disable_encryption()` **是置空的**——**同一资源两种清理方式，一种正确一种不正确** |
| **`release()` 退回裸指针** | `binlog_ostream.cc:217-218`：`m_io_cache.m_encryptor = encryptor.release();` 为了塞进 C 结构的裸指针成员，放弃 `unique_ptr` 所有权，**RAII 在此断裂** |
| **忽略 `open()` 返回值** | `binlog.cc:485` 的 `encrypted_ostream->open(...)` 没检查返回值，而同文件 `:424` 是检查的（且兄弟方法用了 `[[NODISCARD]]` 说明团队知道该用） |
| **`new` 而非 `make_unique`** | `std::unique_ptr<T> p(new T)` 遍布（libbinlogevents 里已用 `make_unique`，`sql/` 侧没跟上） |
| **析构吞掉错误** | `~IO_CACHE_ostream() { close(); }` —— `close()` 的 `bool` 返回值被丢弃，这是 RAII 的固有限制，只能靠事后检查 `write_error` 标志 |

### 9.9 C++ 特性速查表

| 特性 | 位置 | 一句话 |
|---|---|---|
| 纯虚 / 抽象基类 | `basic_ostream.h:50`、`58-90` | 接口极窄（1~5 个纯虚） |
| `virtual ~X() = default` | `basic_ostream.h:51` | 根基类 |
| `~X() override = default` | `basic_ostream.h:92,188` | 中间/叶子类，让编译器校验确实覆盖 |
| `unique_ptr` 派生→基转换 | `binlog.cc:418,426,487` | 依赖虚析构才安全 |
| 按值传 `unique_ptr`（sink） | `binlog_ostream.h:259,272` | 表达"接管所有权" |
| `unique_ptr` 移动出去 | `binlog.cc:578-580` + `down_cast` | 所有权让渡 |
| `shared_ptr` | `basic_ostream.h:176`、`rpl_context.h:278` | 仅压缩器（跨事务复用） |
| 裸指针（非拥有） | `binlog_ostream.h:237` | `m_pipeline_head = &m_file`，预留扩展点 |
| `= delete` 拷贝 | 16 处 | 防切片 + 防二级指针/裸 buffer 误拷 |
| 无自定义移动 | 全族 grep 无 `&&)` | 只依赖智能指针移动 |
| 非类型模板参数 | `basic_ostream.h:159`、`stream_cipher.h:190` | 栈缓冲大小 / 加解密方向 |
| 模板模板参数 | `binlog_reader.h:290` | policy-based 设计 |
| 多重继承 | `basic_ostream.h:160-161` | 接口 + 实现 |
| duck-typing 模板 | `binlog_ostream.h:48-65` `stream_copy` | 零拷贝零虚开销，但零约束 |
| NVI / 模板方法 | `compressor.h:150,230` | 公开非虚 + 私有纯虚 |
| `[[NODISCARD]]` 宏 | `nodiscard.h:44-48` | 绕 gcc bug 84476 |
| `[[maybe_unused]]` | 5 处 | 对抗 `#ifdef HAVE_PSI_INTERFACE` |
| `noexcept` | **0 处** | 服务端不用异常 |
| `enum class` | `stream_cipher.h:182` 等 | 有，但 `cache_type` 仍是普通 enum |
| CRTP / SFINAE | **无** | |
| type traits | `template_utils.h:95-103` `down_cast` | 仅此一处（`static_assert` + `is_base_of`） |
| lambda + RAII 守卫 | `binlog.cc:8092`、`raii/sentry.h:37` | `THD_STAGE_GUARD` 就在 `Compressed_ostream` 旁边 |
| **C 结构包装** | `basic_ostream.h:151` | **组合 + NSDMI 使 `= default` 可用** |
| **"构造成功"检查** | `my_sys.h:487-491` `my_b_inited` | 两阶段构造的运行时状态位 |

---

## 谁在用 IO_CACHE

| 类别 | 位置 | 说明 |
|------|------|------|
| **binlog index** | `binlog.cc:4008, 5361` | `READ_CACHE` |
| **crash-safe index** | `binlog.cc:5933` | `WRITE_CACHE` |
| **purge index** | `binlog.cc:6197, 6266` | 写后 `reinit` 成读 |
| **binlog cache（事务/语句）** | `binlog_ostream.cc:48` | `open_cached_file`，前缀 `ML` |
| **binlog 文件写** | `basic_ostream.cc:45` | 经 `IO_CACHE_ostream` |
| **relay log index** | `rpl_source.cc:1362` | reinit |
| **general / slow log** | `log.cc:545` | `WRITE_CACHE`，**每行末尾 flush** |
| **filesort** | `filesort.cc:546, 1097, 1102` | `tempfile` / `chunk_file`，前缀 `MY` |
| **Unique（去重）** | `uniques.cc:372, 923` | 前缀 `MY` |
| **hash join chunk** | `hash_join_chunk.cc:78` | 前缀 `MY` |
| **LOAD DATA** | `sql_load.cc:1391` | `READ_CACHE` / `READ_FIFO` / `READ_NET` |
| **SELECT INTO OUTFILE** | `query_result.cc:238` | 可选 `disk_sync` |
| **多表 UPDATE 的 rowid 缓冲** | `sql_update.cc:753` | 前缀 `MY` |
| **mysqlbinlog 事件打印** | `log_event.cc:14367-14369` | head/body/footer 三个，**几乎不落盘** |
| **复制元数据** | `rpl_info_file.cc:108, 155` | `READ_CACHE` / `WRITE_CACHE` |
| **auto.cnf（server-uuid）** | `mysqld.cc:5554` | `my_b_printf` + flush + `my_sync` |
| **5.7→8.0 升级读 schema** | `dd/upgrade_57/schema.cc:84` | `READ_CACHE` |

---

## 相关的系统变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `binlog_cache_size` | **32768** | 事务 binlog cache 的内存缓冲大小，也是 spill 阈值 |
| `binlog_stmt_cache_size` | **32768** | 非事务语句的 cache |
| `max_binlog_cache_size` | 极大（≈16 EB） | 写进 `IO_CACHE::end_of_file`，超限 → `EFBIG` |
| `max_binlog_stmt_cache_size` | 同上 | |
| `read_buff_size` | 128 KB | `my_default_record_cache_size` 的来源：IO_CACHE 传 0 时的回落值 |
| `sort_buffer_size` | — | filesort 的内存缓冲（不是 IO_CACHE，但决定 spill 频率） |
| `select_into_buffer_size` | — | `SELECT INTO OUTFILE` 的 IO_CACHE 大小 |
| `select_into_disk_sync` | `OFF` | 打开后每次 flush 都 `mysql_file_sync` |
| `select_into_disk_sync_delay` | `0` | 毫秒，测试模拟慢盘用 |
| `binlog_encryption` | `OFF` | 同时影响 binlog 文件加密与 cache 临时文件加密（**两套机制**） |
| `tmpdir` | 系统临时目录 | IO_CACHE 临时文件的落点 |

---

## Misc

### ★ `end_of_file` 的三种语义（最大的坑）

同一个字段在三种场景下含义完全不同：

| 场景 | `end_of_file` 的含义 |
|------|---------------------|
| **普通 `READ_CACHE`** | 文件的真实末尾（用于裁剪读取长度） |
| **普通 `WRITE_CACHE`** | `~(my_off_t)0`（无上限），使 `_my_b_write` 的 EFBIG 检查永不触发 |
| **`IO_CACHE_binlog_cache_storage`** | **`max_binlog_cache_size`**——容量上限，超限返回 `EFBIG` |
| **`READ_NET`** | **"流还活着吗"**：非 0 = 有数据，0 = 客户端结束 |

> 一个字段承载四种语义，是这个 C 风格结构"复用到底"的典型体现，也是理解 binlog cache 大小限制实现的关键。

### 反直觉点

| 现象 | 原因 |
|------|------|
| **general log 每行一次 `write(2)`** | `File_query_log::write_general` 在行尾调 `flush_io_cache`（`log.cc:670`），**没有 fsync 但有系统调用开销** |
| **`max_binlog_cache_size` 是"提前一个缓冲区"预警** | 检查在 `_my_b_write` 里（`pos_in_file + buffer_length > end_of_file`），不是精确字节计数 |
| **`disk_writes` 不是 I/O 次数而是 flush 次数** | 每次 flush 计 1，即使只写了几个字节 |
| **`sync()` 不帮你 `flush()`** | `IO_CACHE_ostream` 注释明说，所以上层用 `flush_and_sync()` |
| **移动 IO_CACHE 对象会踩坑** | `current_pos` 是指向自身成员的二级指针，移动后必须重新 `setup_io_cache()` |
| **`mysql_encryption_file_write` 在带 `MY_NABP` 时返回 0 表示成功** | 依赖"调用方只关心 0 vs 非 0"的脆弱约定 |
| **binlog cache 临时文件每事务换一次密码** | `setup_ciphers_password()` 在每次 `reset()`（事务提交）时重新生成随机数 |

### 已废弃但仍存在的代码

| 机制 | 状态 |
|------|------|
| **`IO_CACHE_SHARE` / `_my_b_read_r`** | 5.x 时代多 dump 线程共享读同一个 binlog 的遗产。**8.0 的 `sql/` 下 0 匹配**，`init_io_cache_share` / `remove_io_thread` / `IO_CACHE_SHARE` 均无调用点，`_my_b_read_r` 因此成为**死代码**（唯一的绑定条件是 `share != nullptr`） |
| **`SEQ_READ_APPEND`** | 循环双缓冲实现完整，但 `sql/` 层已无调用点，只剩 `mf_iocache.cc:1576` 的 `#ifdef MAIN` 自测试 |
| **`WRITE_NET`** | 8.0 无任何使用点 |
| **`post_read` 回调** | 从未被赋值（只有 `_my_b_get` 会调用它） |

---

## 参考

**源码**（MySQL 8.0.39，本文所有字段名、函数名、行号均已核实）

- `include/my_sys.h` —— `struct IO_CACHE`（:340）、`IO_CACHE_SHARE`（:325）、`cache_type`（:282）、`my_b_read`/`my_b_write` 内联（:495-511）
- `mysys/mf_iocache.cc` —— `init_io_cache_ext`(:176)、`init_io_cache`(:312)、`reinit_io_cache`(:328)、`_my_b_read`(:424)、`_my_b_get`(:1219)、`_my_b_write`(:1238)、`my_b_append`(:1319)、`my_b_safe_write`(:1358)、`my_b_flush_io_cache`(:1430)、`end_io_cache`(:1519)、`init_io_cache_share`(:620)、`lock_io_cache`(:742)、`unlock_io_cache`(:873)、`copy_to_read_buffer`(:1039)、`_my_b_read_r`(:923)、`mysql_encryption_file_*`(:1640-1693)
- `mysys/mf_cache.cc` —— `open_cached_file`(:53)、`real_open_cached_file`(:75)、`close_cached_file`(:87)
- `mysys/mf_tempfile.cc` —— `create_temp_file`(:219)
- `sql/basic_ostream.h` / `.cc` —— `Basic_ostream`(:37)、`Truncatable_ostream`(:58)、`IO_CACHE_ostream`(:98 / cc:31-90)、`Compressed_ostream`(:173)
- `sql/binlog_ostream.h` / `.cc` —— `IO_CACHE_binlog_cache_storage`(:70)、`Binlog_cache_storage`(:174)、`Binlog_encryption_ostream`(:246)
- `sql/binlog.cc` —— `MYSQL_BIN_LOG::Binlog_ofile`(:375)、`log_loaded_block`(:3442)
- `sql/mf_iocache.cc` —— `_my_b_net_read`(:61)
- `sql/sql_load.cc` —— `READ_INFO` 与 READ_NET/READ_FIFO 接线(:1332, :1389-1410)
- `sql/basic_istream.h` / `sql/binlog_istream.h` —— 输入侧体系

**内核月报 / 技术文章**

- 数据库内核月报（阿里云 PolarDB 内核团队）：http://mysql.taobao.org/monthly/
- *MySQL · 引擎特性 · InnoDB 文件系统之 IO 系统和内存管理*（2016/02，基于 5.7.11）——server 层与 InnoDB I/O 的对照视角（**注意：版本差异较大，本文已对其中"模拟 AIO 合并被禁用"等说法做了纠正**）

**相关文档**

- **MySQL 侧 I/O 栈与 server 层场景清单**：[`../../innodb/io.md`](../../innodb/io.md)（其「核心实现六：server 层 I/O 全景」是本篇的上游）
- binlog 的组提交与复制：[`../replication/binlog.md`](../replication/binlog.md)
- 变量体系：[`variables.md`](variables.md)
- VIO 通信抽象（与 IO_CACHE 的 READ_NET 互补）：[`vio.md`](vio.md)

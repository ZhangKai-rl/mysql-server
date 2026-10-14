# 01 从收包到命令分发

> 本篇覆盖：一个数据包如何在**协议层（VIO / NET / Protocol）**被读进来、如何被读成命令、`COM_QUERY` 如何走到解析入口、结果集如何写回，以及 LEX / mem_root 的生命周期。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、协议层：包格式 / VIO / NET / Protocol](#一协议层包格式--vio--net--protocol)
- [二、连接与 THD](#二连接与-thd)
- [三、线程模型](#三线程模型)
- [四、do_command：读一个包](#四do_command读一个包)
- [五、dispatch_command：按命令码分派](#五dispatch_command按命令码分派)
- [六、dispatch_sql_command（原 mysql_parse）](#六dispatch_sql_command原-mysql_parse)
- [七、LEX 的生命周期](#七lex-的生命周期)
- [八、mem_root：为什么 Parse Tree 从不 delete](#八mem_root为什么-parse-tree-从不-delete)
- [九、Prepared Statement 的差异](#九prepared-statement-的差异)
- [十、插件介入点：审计与查询改写](#十插件介入点审计与查询改写)
- [十一、权限检查在哪个阶段](#十一权限检查在哪个阶段)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [核心调用栈](#核心调用栈)

---

## 设计思想与理论基础

### 协议分三层：VIO / NET / Protocol

收发包栈被刻意分成三层（第一节详述），这是**分层抽象**思想的直接体现——每一层只关心自己的抽象，屏蔽下层的差异：

| 层 | 抽象 | 屏蔽的差异 |
|---|---|---|
| VIO | "能读写字节流的通道" | socket / SSL / 命名管道 / 共享内存 |
| NET | "有序的分包字节流" | 4B 包头、16MB 分片、压缩 |
| Protocol | "命令与结果集" | text / binary 两种编码、OK/ERR/EOF 结构 |

**为什么需要 VIO 这一层**：没有它，NET 层就得写四套读写逻辑（SSL 的 `SSL_read`、socket 的 `recv`、管道、共享内存），而且每加一种传输方式就要改一遍 NET。VIO 用 vtable 把"读/写"抽象成函数指针，新增传输方式只需实现一组函数（第一节 1.4）。

```cpp
/* sql/protocol.h:43 —— Protocol 抽象基类的两个纯虚 */
  virtual int read_packet() = 0;
  virtual int get_command(COM_DATA *com_data, enum_server_command *cmd) = 0;
  enum enum_protocol_type {
    PROTOCOL_TEXT = 0,    // text Protocol type used mostly
                          // for the old (MySQL 4.0 protocol)
    PROTOCOL_BINARY = 1,  // binary protocol type
    PROTOCOL_LOCAL = 2,   // local protocol type(intercepts communication)
    PROTOCOL_ERROR = 3,   // error protocol instance
    PROTOCOL_PLUGIN = 4   // pluggable protocol type     ★ X Plugin 走这里
  };
```
> ★ `Protocol` 基类只关心两件事：**读一个包**、**把包解释成一个命令**。其余全是 `store_xxx` 的结果集编码虚函数。

**★ `Protocol_classic`：三层在这里合流**（`sql/protocol_classic.h:52`）：

```cpp
class Protocol_classic : public Protocol {
 private:
  ulong m_client_capabilities;      /* ★ 客户端能力位：EOF/压缩/41 协议全靠它分叉 */
 protected:
  THD *m_thd;
  String *packet;        /* ★ 一级缓冲：指向 &m_thd->packet（String），行数据先攒在这里 */
  String convert;
  uint field_pos;        /* 当前写到第几列，NDEBUG 下与 field_types 对账 */
  bool send_metadata;    /* ★ true = 正在发元数据，此时走文本协议编码 */
  uint field_count;
  uint sending_flags;
  const CHARSET_INFO *result_cs;
```
```cpp
/* :3038 —— Protocol 层的 flush 就是 NET 层的 net_flush */
bool Protocol_classic::flush() { return net_flush(&m_thd->net); }
/* :3025 —— SHOW PROCESSLIST 状态读的就是它 */
uint Protocol_classic::get_rw_status() { return m_thd->net.reading_or_writing; }
/* :3363 —— 一级 → 二级：一行攒完打成完整协议包交给 NET */
bool Protocol_classic::end_row() {
  return my_net_write(&m_thd->net, pointer_cast<uchar *>(packet->ptr()),
                      packet->length());
}
```

### 协议层的设计取向：简单、兼容优先

MySQL 客户端/服务器协议有几个刻意的取舍，贯穿本节：

- **文本为主的结果集**：默认把数字转成十进制字符串发出去，而不是二进制——牺牲带宽换调试友好（`tcpdump` 能直接看懂），binary 协议只给 Prepared Statement 用
- **length-coded integer**：变长整数（1/3/4/9 字节），小值省字节——因为绝大多数长度值很小，这是典型的"为常见情况优化"
- **16MB 分片**：单包上限 `MAX_PACKET_LENGTH`，用"尾片 < 16MB 即结束"表达多包边界，不引入额外 EOF 标记——协议极简

**★ "文本为主"的源码证据**：整数也先转十进制字符串（`protocol_classic.cc:3455`，见 1.8）：

```cpp
  const char *end = longlong10_to_str(value, pos + 1, unsigned_flag ? 10 : -10);
  *pos = end - (pos + 1);  // Set the length byte.
```
> ★ 代价是带宽与 CPU（每行每列都要做一次 `longlong10_to_str`），收益是 `tcpdump` 直接可读、客户端解析简单。**binary 协议只给 PS 用。

**★ "兼容优先"的源码证据**：老客户端的分支塞满各处（`protocol_classic.cc:3319`）：

```cpp
  /* Keep things compatible for old clients */
  if (field->type == MYSQL_TYPE_VARCHAR) field->type = MYSQL_TYPE_VAR_STRING;
```
```cpp
  } else {   /* ★ 4.0 老客户端的 ColumnDefinition320：只 10 字节 */
    pos[0] = 3;
    int3store(pos + 1, field->length);
    pos[4] = 1;
    pos[5] = field->type;
    pos[6] = 3;
    int2store(pos + 7, field->flags);
    pos[9] = (char)field->decimals;
  }
```
> ★ 还有 `NET::fd`（仅为 Perl DBI/DBD 保留）、`NET::return_status`（4.0 老协议回写 server_status 的指针）——**字段都还在，只是没人用了**。这就是"二十年兼容性锁死"的物证。

**★ length-coded integer 的官方动机**（`mysys/pack.cc:112`，★ "为常见情况优化"）：

```cpp
/**
  Faster net_store_length when we know that length is less than 65536.
  We keep a separate version for that range because it's widely used in
  libmysql.

  uint is used as argument type because of MySQL type conventions:
    - uint for 0..65536
    - ulong for 0..4294967296
    - ulonglong for bigger numbers.
*/
static uchar *net_store_length_fast(uchar *packet, size_t length) {
  if (length < 251) { *packet = (uchar)length; return packet + 1; }
  *packet++ = 252;                     /* 252 = 后跟 2 字节 */
  int2store(packet, (uint)length);
  return packet + 2;
}
```

### one-thread-per-connection：历史包袱，但简单可靠

线程模型（第三节）用的是最朴素的**一连接一线程**，而不是线程池/协程。这在今天看来低效（每连接一个 256KB~MB 级栈），但：

- 实现极其简单，每个连接的 THD 状态天然隔离，**没有共享状态竞争**
- 历史包袱：MySQL 诞生于"连接数不大"的年代，且一直没遇到非改不可的压力（连接风暴是运维层面用连接池解决的问题）

这个取舍与后面各篇形成对照——MySQL 的很多设计都是"**简单 > 极致性能**"。

**★ 官方对这个模型的描述**（`sql/conn_handler/connection_handler_impl.h:38`）：

```cpp
/**
  This class represents the connection handling functionality
  that each connection is being handled in a single thread
*/
class Per_thread_connection_handler : public Connection_handler {
  /**
    Check if idle threads to handle connection in
    thread cache. If so enqueue the new connection
    to be picked by the idle thread in thread cache.
  */
  bool check_idle_thread_and_enqueue_connection(Channel_info *channel_info);

  /**
    List of pending channel info objects to be picked by idle
    threads. Protected by LOCK_thread_cache.
  */
  static std::list<Channel_info *> *waiting_channel_info_list;
  ...
 public:
  // Status variables related to Per_thread_connection_handler
  static ulong blocked_pthread_count;  // Protected by LOCK_thread_cache.  ★ = Threads_cached
  static ulong slow_launch_threads;
  static bool shrink_cache;
  // System variable
  static ulong max_blocked_pthreads;                        /* ★ 就是 thread_cache_size */
  static Channel_info *block_until_new_connection();
 protected:
  bool add_connection(Channel_info *channel_info) override;
  uint get_max_threads() const override;   /* ★ 返回 max_connections */
};

/**
  This class represents the connection handling functionality
  of all connections being handled in a single worker thread.
*/
class One_thread_connection_handler : public Connection_handler {  /* ★ --thread-handling=no-threads */
 protected:
  bool add_connection(Channel_info *channel_info) override;
  uint get_max_threads() const override { return 1; }
};
```

**★ 一个有趣的细节："慢启动"统计**（`connection_handler_per_thread.cc:200`）：

```cpp
  if (channel_info->get_prior_thr_create_utime() != 0) {
    /*
      A pthread was created to handle this connection:
      increment slow_launch_threads counter if it took more than
      slow_launch_time seconds to create the pthread.
    */
    ulonglong launch_time =
        thd->start_utime - channel_info->get_prior_thr_create_utime();
    if (launch_time >= slow_launch_time * 1000000ULL)
      Per_thread_connection_handler::slow_launch_threads++;
  }
  ...
  /*
    handle_one_connection() is normally the only way a thread would
    start and would always be on the very high end of the stack ,
    therefore, the thread stack always starts at the address of the
    first local variable of handle_one_connection, which is thd. We
    need to know the start of the stack so that we could check for
    stack overruns.
  */
  thd_set_thread_stack(thd, (char *)&thd);   /* ★★ 用局部变量地址锚定栈底，用于栈溢出检测 */
  thd->store_globals();                      /* 把 THD 绑到 current_thd 等线程局部变量 */
```
> ★ 状态变量 `Slow_launch_threads` 就是这么来的——它衡量的是**建 OS 线程**花的时间，不是认证时间。

**★ 线程池不是"没有"，而是"插件化"**（`connection_handler_manager.cc:287`）：

```cpp
int my_connection_handler_set(Connection_handler_functions *chf,
                              THD_event_functions *tef) {
  assert(chf != nullptr && tef != nullptr);
  if (chf == nullptr || tef == nullptr) return 1;
  Plugin_connection_handler *conn_handler =
      new (std::nothrow) Plugin_connection_handler(chf);
  if (conn_handler == nullptr) return 1;
  Connection_handler_manager::get_instance()->load_connection_handler(conn_handler);
  Connection_handler_manager::saved_event_functions =
      Connection_handler_manager::event_functions;
  Connection_handler_manager::event_functions = tef;
  return 0;
}
```
> ★ 官方（Percona）线程池正是通过 `my_connection_handler_set()` 接管 `Connection_handler`。所以严格说：**8.0.39 的社区版默认不用线程池，但架构上留了口子**。

---

## 先有个整体印象

一句话概括这段做的事：**把一个网络字节流，变成"一条待执行的 SQL 语句"这个内部对象（LEX + Sql_cmd），执行完再把结果集写回网络字节流**。

```
网络字节流
   │  VIO（socket / SSL 抽象）读取
   ▼
NET 层：4B 包头（3B 长度 + 1B 序号）+ payload，>16MB 自动分片
   │  Protocol 层：首字节是命令码，解出 command + payload
   ▼
command (COM_QUERY / COM_STMT_PREPARE / ...)
   │  switch 分派
   ▼
COM_QUERY → 准备 LEX → 改写插件 → 解析 → 得到 LEX + Sql_cmd → 交给执行器
   │
   └─ 执行结果 → Protocol 编码（OK/ERR/EOF/结果集）→ NET → VIO → 回客户端
```

**这段不碰"SQL 是什么意思"** —— 那是 parser 和 contextualize 的事（见 02、03）。

---

## 一、协议层：包格式 / VIO / NET / Protocol

> 收包与回包是一个整体：包在进来（读）和出去（写）两个方向都经过同一套 VIO → NET → Protocol 分层，所以放在一起讲。

### 1.1 分层总览

```
应用层    Protocol_classic        命令解码 / 结果集编码（text & binary）
             │
网络层    NET                      4B 包头、缓冲、>16MB 分片、压缩、超限判定
             │
传输层    VIO                      虚抽象：socket / SSL / 命名管道统一接口
             │
        OS socket
```

### 1.2 包格式：4 字节包头与 16MB 分片

**包头**（`include/mysql_com.h:108`）：

```cpp
#define MAX_PACKET_LENGTH (256L * 256L * 256L - 1)  // 0xFFFFFF = 16777215 = 16MB-1
#define NET_HEADER_SIZE 4  // 3B length + 1B sequence id
```

```
int<3>  payload_length   不含这 4 字节头的 payload 字节数（小端）
int<1>  sequence_id      包序号
string<var> payload
```

**sequence_id**（`net->pkt_nr`）：

- 写路径 `net->pkt_nr++`（`my_net_write` / `net_write_command`），写 header 时取低 8 位强转 `(uchar)`，所以会回绕
- 读路径 `net_read_packet_header` 校验后 `net->pkt_nr++`
- 复位：`net_new_transaction(net)`（`include/mysql_com.h:1090` 展开为 `net->pkt_nr = 0`），`do_command` 收新命令前调用；`net_clear()` 则同时清 `pkt_nr` 和 `compress_pkt_nr`

**写 header 与大包拆片**（`sql-common/net_serv.cc:438` `my_net_write`）：

```cpp
while (len >= MAX_PACKET_LENGTH) {          // 满 16MB 就发一个满长片段
  const ulong z_size = MAX_PACKET_LENGTH;
  int3store(buff, z_size);                  // 低 3 字节 = 0xFFFFFF
  buff[3] = (uchar)net->pkt_nr++;           // 第 4 字节 = seq
  if (net_write_buff(net, buff, NET_HEADER_SIZE) ||
      net_write_buff(net, packet, z_size)) return true;
  packet += z_size; len -= z_size;
}
int3store(buff, static_cast<uint>(len));    // 最后一片 < 16MB（可为 0）
buff[3] = (uchar)net->pkt_nr++;
return net_write_buff(net, packet, len);
```

> **EOF 语义**：最后一个片段长度 `< 16MB`（可为 0）本身即表示"多包结束"，协议层不需要额外的 EOF 标记。注意条件是 `<` 而非 `<=`，所以恰为 `MAX_PACKET_LENGTH` 时也会拆。

**读侧合并**（`net_serv.cc:2170` `net_read_uncompressed_packet`）：

```cpp
len = net_read_packet(net, &complen);
if (len == MAX_PACKET_LENGTH) {             // 首片段满长 => 还有续片
  ulong save_pos = net->where_b;
  size_t total_length = 0;
  do {
    net->where_b += len;                    // 后移写入位置，把片段拼进 buff
    total_length += len;
    len = net_read_packet(net, &complen);
  } while (len == MAX_PACKET_LENGTH);       // 读到 < 16M 的尾片为止
  if (len != packet_error) len += total_length;
  net->where_b = save_pos;
}
```

### 1.3 NET 结构（`include/mysql_com.h:914`）

关键成员：

| 成员 | 语义 |
|------|------|
| `buff` / `buff_end` / `write_pos` / `read_pos` | 可增长缓冲区。`write_pos` 是发送缓冲当前写指针，`read_pos` 是 `my_net_read` 返回的 payload 起点 |
| `remain_in_buf` / `buf_length` / `where_b` | 压缩协议下跟踪"一个压缩包内含多个 MySQL 包"的游标 |
| `max_packet` | 当前缓冲区容量（`net_realloc` 按 `IO_SIZE` 对齐后更新） |
| `max_packet_size` | 协议允许的 payload 上限，据此抛 `ER_NET_PACKET_TOO_LARGE` |
| `pkt_nr` / `compress_pkt_nr` | 内层 MySQL 包序号 / 压缩层包序号 |
| `compress` | 是否启用压缩协议（默认 false） |
| `error` / `last_errno` / `last_error` / `sqlstate` | NET 层错误状态 |

**★ `NET` 是纯 POD 结构体，27 个字段，零成员函数**（⚠️ 常见说法"NET 是 class"是错的）：

```cpp
/* include/mysql_com.h:908 —— error 字段的 5 个取值 */
#define NET_ERROR_UNSET 0               /**< No error has occurred yet */
#define NET_ERROR_SOCKET_RECOVERABLE 1  /**< Socket still usable */
#define NET_ERROR_SOCKET_UNUSABLE 2     /**< Do not use the socket */
#define NET_ERROR_SOCKET_NOT_READABLE 3 /**< Try write and close socket */
#define NET_ERROR_SOCKET_NOT_WRITABLE 4 /**< Try read and close socket */

/* include/mysql_com.h:914 */
typedef struct NET {
  MYSQL_VIO vio;                                          /* ★ VIO 层句柄：NET 与 VIO 的唯一耦合点 */
  unsigned char *buff, *buff_end, *write_pos, *read_pos;  /* ★ 三级缓冲的核心四指针 */
  my_socket fd; /* For Perl DBI/dbd */                   /* ★ 仅为 Perl DBI/DBD 保留的冗余 fd */
  /**
    Set if we are doing several queries in one
    command ( as in LOAD TABLE ... FROM MASTER ),
    and do not want to confuse the client with OK at the wrong time
  */
  unsigned long remain_in_buf, length, buf_length, where_b;
  unsigned long max_packet, max_packet_size;              /* 容量 / 硬上限 */
  unsigned int pkt_nr, compress_pkt_nr;                   /* ★ 两套独立序号 */
  unsigned int write_timeout, read_timeout, retry_count;  /* 单位秒 */
  int fcntl;
  unsigned int *return_status;                            /* 4.0 老协议回写 server_status 的指针 */
  unsigned char reading_or_writing;                       /* ★★ SHOW PROCESSLIST State 的数据源：1=读 2=写 0=其它 */
  unsigned char save_char;                                /* 读包时做 \0 截断的临时保存字节 */
  bool compress;
  unsigned int last_errno;
  unsigned char error;                                    /* 取值即上面 5 个 NET_ERROR_* */
  /** Client library error message buffer. Actually belongs to struct MYSQL. */
  char last_error[MYSQL_ERRMSG_SIZE];
  /** Client library sqlstate buffer. Set along with the error message. */
  char sqlstate[SQLSTATE_LENGTH + 1];
  /**
    Extension pointer, for the caller private use.
    ...
    The mysqld server process uses this pointer internally,
    to maintain the server internal instrumentation for the connection.
  */
  void *extension;                                        /* ★★ 服务端指向 NET_SERVER */
} NET;
#define packet_error (~(unsigned long)0)
```
```cpp
/* include/mysql_com_server.h:59 —— 服务端扩展：压缩上下文在这里 */
typedef struct NET_SERVER {
  before_header_callback_fn m_before_header;   /* 读包头前的钩子（PFS / 审计埋点） */
  after_header_callback_fn m_after_header;
  void *m_user_data;
  struct compression_attributes compression;   /* 协商出的算法名 + level，供 SHOW 显示 */
  mysql_compress_context compress_ctx;         /* ★ zlib / zstd 的运行时上下文 */
  bool timeout_on_full_packet;                 /* ★ net_read_timeout 是"每包"还是"每次 recv" */
} NET_SERVER;
```
> ⚠️ **真名核查**：`compress_buf` / `net_skip_rest` / `net_wait_data` **在 8.0.39 全部不存在**（`compress_buf` 被 `NET_SERVER::compress_ctx` 取代）。

**★ `MAX_PACKET_LENGTH` 的官方注释直接点出了 0xFE 歧义**（`include/mysql_com.h:100`）：

```cpp
/**
  Maximum length of protocol packet.
  @ref page_protocol_basic_ok_packet length limit also restricted to this value
  as any length greater than this value will have first byte of
  @ref page_protocol_basic_ok_packet to be 254 thus does not
  provide a means to identify if this is @ref page_protocol_basic_ok_packet or
  @ref page_protocol_basic_eof_packet.
*/
#define MAX_PACKET_LENGTH (256L * 256L * 256L - 1)
```
```cpp
#define NET_HEADER_SIZE 4  /**< standard header size. 3B length + 1B sequence id.  */
#define COMP_HEADER_SIZE 3 /**< compression header extra size */
```
> ★ 这就是"OK 包长度也被限制在 16MB"的原因：**超过 16MB 时 OK 包首字节会变成 254(0xFE)，与 EOF 无法区分**。

**★ 缓冲区容量从哪来**（`sql/sql_client.cc:39`，★ 两个"max"的区别）：

```cpp
void my_net_local_init(NET *net) {
  net->max_packet = (uint)global_system_variables.net_buffer_length;
                                          /* ★ 二级缓冲初始容量 = @@net_buffer_length（默认 16KB） */
  my_net_set_read_timeout(net, (uint)global_system_variables.net_read_timeout);
  my_net_set_write_timeout(net, (uint)global_system_variables.net_write_timeout);
  net->retry_count = (uint)global_system_variables.net_retry_count;
  net->max_packet_size =
      max<size_t>(global_system_variables.net_buffer_length,
                  global_system_variables.max_allowed_packet);
                                          /* ★ 硬上限 = max(net_buffer_length, max_allowed_packet) */
}
```
```cpp
/* sql-common/net_serv.cc:149 —— 缓冲区建立：多分配 4+3 字节给包头和解压留余量 */
  if (!(net->buff = (uchar *)my_malloc(
            key_memory_NET_buff,
            (size_t)net->max_packet + NET_HEADER_SIZE + COMP_HEADER_SIZE,
            MYF(MY_WME))))
    return true;
  net->buff_end = net->buff + net->max_packet;
  ...
  if (vio) {
    net->fd = vio_fd(vio);
    vio_fastsend(vio);                    /* ★ 关 Nagle：TCP_NODELAY */
  }
```

### 1.4 VIO 层：为什么需要抽象

`struct Vio`（`include/violite.h:318`）的核心是**函数指针 vtable**（`:384-433`）：

```cpp
size_t (*read)(MYSQL_VIO, uchar *, size_t);
size_t (*write)(MYSQL_VIO, const uchar *, size_t);
void (*viodelete)(MYSQL_VIO);
int (*vioerrno)(MYSQL_VIO);
int (*timeout)(MYSQL_VIO, uint, bool);
...
```

`enum_vio_type`（`violite.h:79`）：`TCPIP` / `SOCKET`（Unix domain）/ `NAMEDPIPE` / `SSL` / `SHARED_MEMORY` / `LOCAL` / `PLUGIN`。

`vio_init`（`vio/vio.cc:211`）按 type 绑 vtable：

```cpp
switch (type) {
  case VIO_TYPE_SSL:
    vio->read = vio_ssl_read;  vio->write = vio_ssl_write;  break;
  default:  // TCPIP / Unix socket
    vio->read = vio->read_buffer ? vio_read_buff : vio_read;
    vio->write = vio_write;  ...
}
```

分派宏（`violite.h:284`）：`#define vio_read(vio,buf,size) ((vio)->read)(vio,buf,size)`。

- **viosocket**（`vio/viosocket.cc:136`）：`vio_read` 在有 `read_timeout` 时用 `MSG_DONTWAIT` 发 `mysql_socket_recv`，遇 `EAGAIN` 且阻塞模式则 `vio_socket_io_wait`（poll/select）等待可读再重试。`vio_read_buff`（`:179`）小读（<2048）时一次 recv 16KB 到 `read_buffer` 再 memcpy，省系统调用
- **viossl**（`vio/viossl.cc:263`）：`vio_ssl_read` 包装 `SSL_read`，把 `SSL_ERROR_WANT_READ/WRITE` 经 `ssl_should_retry` 转成 `VIO_IO_EVENT_*`；非阻塞时返回 `VIO_SOCKET_WANT_READ/WRITE` 让上层知道该等哪个方向

### 1.5 NET 层：读一个包

`my_net_read`（`net_serv.cc:2241`）按 `net->compress` 分派到 `net_read_compressed_packet` / `net_read_uncompressed_packet`。

单包 `net_read_packet`（`net_serv.cc:2080`）：

```cpp
static size_t net_read_packet(NET *net, size_t *complen) {
  net->compress_pkt_nr = net->pkt_nr;             // 读失败也要保证序号一致
  if (net_read_packet_header(net)) goto error;    // 先读 4B（压缩 7B）头
  ...
  pkt_len = uint3korr(net->buff + net->where_b);  // 3 字节 payload 长度
  if (!pkt_len) goto end;                         // 长度 0 => 多包结尾
  pkt_data_len = max(pkt_len, *complen) + net->where_b;
  if ((pkt_data_len >= net->max_packet) && net_realloc(net, pkt_data_len))
    goto error;                                   // 扩容 + 上限校验
  if (net_read_raw_loop(net, pkt_len)) goto error;// 精确读 payload
  ...
}
```

读头 `net_read_packet_header`（`net_serv.cc:1453`）校验序号：`pkt_nr != (uchar)net->pkt_nr` 时 server 侧 `my_error(ER_NET_PACKETS_OUT_OF_ORDER)`。

底层 `net_read_raw_loop`（`net_serv.cc:1350`）循环 `vio_read` 直到读满 `count` 字节；`recvcnt == 0` 表示对端关闭。

**超限判定**（`net_serv.cc:210` `net_realloc`）：

```cpp
if (length >= net->max_packet_size) {
  net->error = NET_ERROR_SOCKET_RECOVERABLE;
  net->last_errno = ER_NET_PACKET_TOO_LARGE;      // 客户端映射 CR_NET_PACKET_TOO_LARGE
  return true;
}
```

`max_packet_size` 在 `my_net_local_init`（`sql/sql_client.cc:39`）取 `max(net_buffer_length, max_allowed_packet)`。

### 1.6 NET 层：写一个包

`net_write_buff`（`net_serv.cc:944`）的缓冲策略：先 memcpy 到 `net->buff[write_pos]`，放不下才 `net_write_packet`；大块数据（`len > max_packet`）直接写出省一次 memcpy。

`net_write_packet`（`net_serv.cc:1296`）经 `compress_packet`（可选压缩）→ `net_write_raw_loop`（`:997`）→ `vio_write`，循环直到写完。

`net_flush`（`net_serv.cc:285`）：

```cpp
if (net->buff != net->write_pos) {               // 有积压就刷
  net_write_packet(net, net->buff, (size_t)(net->write_pos - net->buff));
  net->write_pos = net->buff;
}
if (net->compress) net->pkt_nr = net->compress_pkt_nr;  // 同步压缩序号
```

### 1.7 压缩协议

- 压缩头 7B：`[comp_len(3)][comp_seq(1)][uncomp_len(3)]`（`compress_packet`，`net_serv.cc:1248`）
- `uncomp_len == 0` 表示 payload **未压缩**（压缩后反而更大时就发原文）
- `pkt_nr`（内层）与 `compress_pkt_nr`（压缩层）分开计数，`net_flush` / 读头时对齐
- 默认**不压缩**（`my_net_init` 里 `net->compress = false`），握手阶段按能力协商

**★ 7 字节头的逐字节写入**（`sql-common/net_serv.cc:1248`，⚠️ 真实路径是 `sql-common/net_serv.cc`，**不是** `sql/net_serv.cc`）：

```cpp
static uchar *compress_packet(NET *net, const uchar *packet, size_t *length) {
  uchar *compr_packet;
  size_t compr_length = 0;
  const uint header_length = NET_HEADER_SIZE + COMP_HEADER_SIZE;  /* ★ 4 + 3 = 7 */

  compr_packet = (uchar *)my_malloc(key_memory_NET_compress_packet,
                                    *length + header_length, MYF(MY_WME));
  if (compr_packet == nullptr) return nullptr;

  memcpy(compr_packet + header_length, packet, *length);   /* 原包整体搬到 7B 头之后 */

  mysql_compress_context *compress_ctx = compress_context(net);

  /* Compress the encapsulated packet. */
  if (my_compress(compress_ctx, compr_packet + header_length, length,
                  &compr_length)) {
    /*
      If the length of the compressed packet is larger than the
      original packet, the original packet is sent uncompressed.
    */
    compr_length = 0;      /* ★ 压缩失败/反而变大 → uncomp_len 置 0，原样发 */
  }

  /* Length of the compressed (original) packet. */
  int3store(&compr_packet[NET_HEADER_SIZE], static_cast<uint>(compr_length)); /* 字节 4..6 */
  /* Length of this packet. */
  int3store(compr_packet, static_cast<uint>(*length));                        /* 字节 0..2 */
  /* Packet number. */
  compr_packet[3] = (uchar)(net->compress_pkt_nr++);                          /* 字节 3 */

  *length += header_length;
  return compr_packet;
}
```

**★ 官方抓包实例**（`net_serv.cc:1112`，★ `SELECT 1` 未压缩时的样子）：

```
  Sending a `SELECT 1` query as uncompressed payload:
  0d 00 00 00 00 00 00 09    00 00 00 03 53 45 4c 45    ............SELE
  43 54 20 31                                           CT 1

  decodes into:
  raw packet length                      -> 20
  compressed payload length   = 0d 00 00 -> 13 (20 - 7)
  sequence id                 = 00       ->  0
  uncompressed payload length = 00 00 00 -> uncompressed
```
> ★★ **`uncomp_len == 0` 就是"本包未压缩"的信号**——不是"压缩后长度为 0"。
> ⚠️ 常见说法"3B comp + 3B pkt_nr + 1B uncomp"是错的顺序，真相是 **3B comp(0..2) + 1B seq(3) + 3B uncomp(4..6)**。

**★ 小包不压缩的阈值**（`mysys/my_compress.cc:280`）：

```cpp
bool my_compress(mysql_compress_context *comp_ctx, uchar *packet, size_t *len,
                 size_t *complen) {
  if (*len < MIN_COMPRESS_LENGTH) {          /* ★ 小于 50 字节不压缩 */
    *complen = 0;
    DBUG_PRINT("note", ("Packet too short: Not compressed"));
  } else {
    uchar *compbuf = my_compress_alloc(comp_ctx, packet, len, complen);
    ...
```
```cpp
/* include/my_sys.h:173 —— 常量不在 mysql_com.h */
#define MIN_COMPRESS_LENGTH 50
```
> ★ 官方注释（`net_serv.cc:1176`）：*"Usually payloads less than 50 bytes (::MIN_COMPRESS_LENGTH) aren't compressed."*

**★★ 8.0 新增 zstd，不只是 zlib**（`include/my_compress.h:28`）：

```cpp
/* List of valid values for compression_algorithm */
enum enum_compression_algorithm {   /* ★ 8.0 新增 MYSQL_ZSTD */
  MYSQL_UNCOMPRESSED = 1,
  MYSQL_ZLIB,
  MYSQL_ZSTD,
  MYSQL_INVALID
};

typedef struct mysql_compress_context {
  enum enum_compression_algorithm algorithm;  ///< Compression algorithm name.
  union {
    mysql_zlib_compress_context zlib_ctx;
    mysql_zstd_compress_context zstd_ctx;
  } u;
} mysql_compress_context;
```
```cpp
/* mysys/my_compress.cc:295 —— 算法分发 */
uchar *my_compress_alloc(mysql_compress_context *comp_ctx, const uchar *packet,
                         size_t *len, size_t *complen) {
  if (comp_ctx->algorithm == enum_compression_algorithm::MYSQL_ZSTD)
    return zstd_compress_alloc(&comp_ctx->u.zstd_ctx, packet, len, complen);
  if (comp_ctx->algorithm == enum_compression_algorithm::MYSQL_UNCOMPRESSED) {
    // If compression algorithm is set to none do not compress, even if compress
    // flag was set.
    *complen = 0;
    return nullptr;
  }
  assert(comp_ctx->algorithm == enum_compression_algorithm::MYSQL_ZLIB);
  return zlib_compress_alloc(&comp_ctx->u.zlib_ctx, packet, len, complen);
}
```
```cpp
/* mysys/my_compress.cc:351 —— 默认压缩级别：zlib=6, zstd=3 */
unsigned int mysql_default_compression_level(
    enum enum_compression_algorithm algorithm) {
  switch (algorithm) {
    case MYSQL_ZLIB: return 6;
    case MYSQL_ZSTD: return 3;
    default: assert(0); return 0;
  }
}
```

**★ 压缩的启用条件与压缩层的"透明性"**（`net_serv.cc:1042`，官方 Doxygen）：

```cpp
/**
  @page page_protocol_basic_compression Compression

  Compression is:
    - its own protocol layer
    - transparent to the other MySQL protocol layers
    - compressing a string of bytes (which may even be a part of
      @ref sect_protocol_basic_packets_packet)

  It is enabled if:
    - the server announces ::CLIENT_COMPRESS or
      ::CLIENT_ZSTD_COMPRESSION_ALGORITHM ... based on
      variable protocol_compression_algorithms and
    - the client does following: ...
      - if client flags do not match then connection fallsback to uncompressed mode.
*/
```

### 1.8 Protocol 层：结果集编码

**length-coded integer**（`mysys/pack.cc:129` `net_store_length`）：

```cpp
if (length < 251)         { *packet = (uchar)length; return packet + 1; }   // 1 字节
if (length < 65536)       { *packet++ = 252; int2store(packet, length); return packet + 2; }  // 3 字节
if (length < 16777216)    { *packet++ = 253; int3store(packet, length); return packet + 3; }  // 4 字节
*packet++ = 254; int8store(packet, length); return packet + 8;              // 9 字节
```

规则：**1 字节(0–250) / 3 字节(252 前缀) / 4 字节(253 前缀) / 9 字节(254 前缀)**；`251(0xFB)` 保留给 NULL，`255(0xFF)` 保留给 ERR 头。

**text 协议行编码**（`Protocol_text::store_*`，`protocol_classic.cc:3401`）：每列 = `net_store_length`（变长长度）+ 文本字节。`store_string`（`:3423`）必要时做 charset 转换；`store_integer`（`:3455`）用 `longlong10_to_str` 转十进制文本；`store_double` 走 `my_gcvt`；`store_datetime` 走 `my_datetime_to_str`。

**OK / EOF / ERR 包**：

| 包 | 头字节 | 结构 |
|----|--------|------|
| OK（`net_send_ok`，`:862`） | `0x00`（`CLIENT_DEPRECATE_EOF` 时可为 `0xFE`） | lenenc affected_rows + last_insert_id + server_status(2B) + warnings(2B) + info 串 |
| EOF（`write_eof_packet`，`:1086`） | `0xFE` | warnings(2B) + status_flags(2B) |
| ERR（`net_send_error_packet`，`:1199`） | `0xFF` | error_code(2B) + `'#'` + sqlstate(5B) + message |

**binary 协议**（prepared statement 结果集，`Protocol_binary::start_row`，`:3786`）：行 = `0x00` 头 + NULL bitmap + 各列**定长二进制**（无 lenenc 长度前缀）。`store_null`（`:3793`）只置 bitmap 位；`store_tiny/short/long/longlong` 写 1/2/4/8 字节小端。

**★★ 继承链：`Protocol_binary → Protocol_text → Protocol_classic → Protocol`**（⚠️ binary **不是**直接继承 classic）：

```cpp
/* sql/protocol_classic.h:216 */
/** Class used for the old (MySQL 4.0 protocol). */
class Protocol_text : public Protocol_classic { ... };

/* :240 —— ★ 注意：binary 继承 text */
class Protocol_binary final : public Protocol_text {
 private:
  uint bit_fields;       /* ★ NULL-bitmap 字节数 = (num_cols + 9) / 8 */
  ...
  // Decimals are sent as text, also over the binary protocol.
  using Protocol_text::store_decimal;      /* ★★ DECIMAL 在二进制协议里也走文本 */
  ...
};
```
> ★ `+9` 而非 `+7`：因为 bitmap 从 offset 2 开始算列（前 2 字节被 `0x00` 头占用）。

**★★ 官方的"结果集发送流程"注释**（`sql/protocol.h:191`，★ 这是理解整个回包链路的骨架）：

```cpp
  /**
    Result set sending functions

    @details Server uses following schema to send result:
                  ... sending metadata ...
                             | start_result_metadata(...)
                             | start_row()
                             | send_field_metadata(...)
                             | end_row()
              ... same for each field sent ...
                             | end_result_metadata(...)
                             |
                  ... sending result ...
                             | start_row(...)
                             | store_xxx(...)
           ... store_xxx(..) is called for each field ...
                             | end_row(...)
        ... same for each row, until all rows are sent ...
                             | send_ok/eof/error(...)
    However, a protocol implementation might use different schema. For
    example, Protocol_callback ignores start/end_row when metadata is being
    sent.
   */
```
> ⚠️ **8.0.39 里 `Protocol` 没有 `send_result_metadata`**，它被拆成上面 5 步。`THD::send_result_metadata()` 是驱动编排。

**★★ `send_field_metadata()`：ColumnDefinition41 的逐字段编码**（`protocol_classic.cc:3269`）：

```cpp
  /* Keep things compatible for old clients */
  if (field->type == MYSQL_TYPE_VARCHAR) field->type = MYSQL_TYPE_VAR_STRING;
                                        /* ★ 兼容优先：VARCHAR 降级成 VAR_STRING */

  if (has_client_capability(CLIENT_PROTOCOL_41)) {
    if (store_string(STRING_WITH_LEN("def"), cs) ||                 /* ① catalog：永远 "def" */
        store_string(field->db_name, strlen(field->db_name), cs) || /* ② schema */
        store_string(field->table_name, ...) ||                     /* ③ table（别名） */
        store_string(field->org_table_name, ...) ||                 /* ④ org_table（真名） */
        store_string(field->col_name, ...) ||                       /* ⑤ name（别名） */
        store_string(field->org_col_name, ...) ||                   /* ⑥ org_name（真名） */
        packet->mem_realloc(packet->length() + 12)) { ... }
    pos = packet->ptr() + packet->length();
    *pos++ = 12;  // Length of packed fields    /* ⑦ 固定 0x0c：后面 12 字节定长区 */
    ...
      int2store(pos, item_charset->number);     /* ⑧ 字符集（2B） */
      int4store(pos + 2, field->length);        /* ⑨ column_length（4B） */
    ...
    pos[6] = field->type;             /* ⑩ 列类型（1B，enum_field_types） */
    int2store(pos + 7, field->flags); /* ⑪ flags（2B） */
    pos[9] = (char)field->decimals;   /* ⑫ decimals（1B） */
    pos[10] = 0;  // For the future
    pos[11] = 0;  // For the future
  } else {
    /* ★ 4.0 老客户端的 ColumnDefinition320：只 10 字节 */
    pos[0] = 3;
    int3store(pos + 1, field->length);
    pos[4] = 1;
    pos[5] = field->type;
    pos[6] = 3;
    int2store(pos + 7, field->flags);
    pos[9] = (char)field->decimals;
  }
```
```cpp
/* include/mysql_com.h:149 —— ⑪ 里那些 flag */
#define NOT_NULL_FLAG 1     /**< Field can't be NULL */
#define PRI_KEY_FLAG 2      /**< Field is part of a primary key */
#define UNIQUE_KEY_FLAG 4   /**< Field is part of a unique key */
#define MULTIPLE_KEY_FLAG 8 /**< Field is part of a key */
#define BLOB_FLAG 16        /**< Field is a blob */
#define UNSIGNED_FLAG 32    /**< Field is unsigned */
#define ZEROFILL_FLAG 64    /**< Field is zerofill */
#define BINARY_FLAG 128     /**< Field is binary   */
/* The following are only sent to new clients */
#define ENUM_FLAG 256
#define AUTO_INCREMENT_FLAG 512
#define TIMESTAMP_FLAG 1024
#define SET_FLAG 2048
#define NO_DEFAULT_VALUE_FLAG 4096
#define ON_UPDATE_NOW_FLAG 8192
#define NUM_FLAG 32768
```

**★ text 协议：整数也先转十进制字符串**（`protocol_classic.cc:3455`）：

```cpp
static bool store_integer(int64 value, bool unsigned_flag, uint32 zerofill,
                          String *packet) {
  ...
  char *pos = packet->prep_append(MY_INT64_NUM_DECIMAL_DIGITS + 2,
                                  PACKET_BUFFER_EXTRA_ALLOC);
  if (pos == nullptr) return true;
  const char *end = longlong10_to_str(value, pos + 1, unsigned_flag ? 10 : -10);
                                   /* ★ 先转十进制文本，写在长度字节之后 */
  *pos = end - (pos + 1);          /* ★ 再回填 1B 长度前缀 */
  packet->length(end - packet->ptr());
  return false;
}
```
```cpp
/* :3406 —— NULL 是 0xFB，不是 0x00 也不是 0xFE */
bool Protocol_text::store_null() {
  field_pos++;
  char buff[1];
  buff[0] = (char)251;
  return packet->append(buff, sizeof(buff), PACKET_BUFFER_EXTRA_ALLOC);
}
```
```cpp
/* :3401 —— start_row 只把 length 归零，不释放内存（复用同一个 String） */
void Protocol_text::start_row() {
  field_pos = 0;
  packet->length(0);
}
```

**★★ `Protocol_binary`：元数据阶段"退化"成文本**（`protocol_classic.cc:3786`）：

```cpp
void Protocol_binary::start_row() {
  if (send_metadata) return Protocol_text::start_row();   /* ★ 元数据阶段走文本 */
  packet->length(bit_fields + 1);
  memset(packet->ptr(), 0, 1 + bit_fields);               /* ① 首字节 0x00 ② bitmap 清零 */
  field_pos = 0;
}

bool Protocol_binary::store_null() {
  if (send_metadata) return Protocol_text::store_null();
  uint offset = (field_pos + 2) / 8 + 1, bit = (1 << ((field_pos + 2) & 7));
  char *to = packet->ptr() + offset;
  *to = (char)((uchar)*to | (uchar)bit);   /* ★ 只置 bitmap 位，payload 不占字节 */
  field_pos++;
  return false;
}

bool Protocol_binary::store_long(longlong from, uint32 zerofill) {
  if (send_metadata) return Protocol_text::store_long(from, zerofill);
  field_pos++;
  char *to = packet->prep_append(4, PACKET_BUFFER_EXTRA_ALLOC);  /* ★ 4B 定长小端 */
  if (!to) return true;
  int4store(to, static_cast<uint32>(from));
  return false;
}
```

**★ `Protocol` 是个"栈"**（`sql/protocol.h:43`，★ 少有人注意的设计）：

```cpp
class Protocol {
 private:
  /// Pointer to the Protocol below on the stack.
  Protocol *m_previous_protocol = nullptr;
  ...
  Protocol *pop_protocol() { ... }
  void push_protocol(Protocol *protocol) {
    assert(!protocol->m_previous_protocol);
    protocol->m_previous_protocol = this;
  }
  enum { SEND_NUM_ROWS = 1, SEND_DEFAULTS = 2, SEND_EOF = 4 };  /* ★ 三个发送开关位 */
```
> ★ `COM_STMT_EXECUTE` 等可临时压入一个新 Protocol（`Protocol_binary`），执行完再弹出。
> ⚠️ `sql/protocol.cc` **不存在**——真实文件是 `sql/protocol.h`（基类）+ `sql/protocol_classic.h/.cc`。

### 1.9 回包链路

```
send_result_set_row → start_row()/store_*() → end_row()
end_row: my_net_write(packet)                              protocol_classic.cc:3365
my_net_write → net_write_buff（缓冲）→ net_write_packet（可选压缩）
             → net_write_raw_loop → vio_write →（socket send / SSL_write）
```

关键行号：`Protocol_classic::write` `:1380`、`flush` `:3038`、`end_row` `:3363-3367`、`net_send_ok` 的 `my_net_write + net_flush` `:966-967`。

**★ 三个"写"函数的分工**（⚠️ 三者并存，不是互相取代）：

| 函数 | 文件:行 | 职责 |
|---|---|---|
| `my_net_write()` | `sql-common/net_serv.cc:438` | 写**裸 payload** 包（加 4B 头） |
| `net_write_command()` | `:878` | 写**命令包**（header + 1B command + payload），且**自带 `net_flush`** |
| `net_write_packet()` | `:1296` | 真正落 socket（可选压缩） |

```cpp
/* net_serv.cc:438 —— my_net_write：16MB 分片循环 */
  while (len >= MAX_PACKET_LENGTH) {          /* ★ 条件是 >= 不是 > */
    const ulong z_size = MAX_PACKET_LENGTH;
    int3store(buff, z_size);                  /* 3B payload length，小端 */
    buff[3] = (uchar)net->pkt_nr++;           /* ★ 1B sequence id，每片自增并回绕 */
    if (net_write_buff(net, buff, NET_HEADER_SIZE) ||
        net_write_buff(net, packet, z_size)) return true;
    packet += z_size;
    len -= z_size;
  }
  int3store(buff, static_cast<uint>(len));    /* 写最后一片 */
  buff[3] = (uchar)net->pkt_nr++;
```
```cpp
/* net_serv.cc:878 —— net_write_command：command 只占第一个分片 */
  buff[4] = command; /* For first packet */
  ...
      head_len = 0;                           /* ★ 后续分片不再带 header/command */
      header_size = NET_HEADER_SIZE;
  ...
  bool rc = net_write_buff(net, buff, header_size) ||
            (head_len && net_write_buff(net, header, head_len)) ||
            net_write_buff(net, packet, len) || net_flush(net);
                                              /* ★★ 命令包必须立即发出 */
```

**★★ `net_write_buff()`：flush 阈值的真相**（`net_serv.cc:944`，★ 三级缓冲的核心）：

```cpp
static bool net_write_buff(NET *net, const uchar *packet, size_t len) {
  ulong left_length;
  if (net->compress && net->max_packet > MAX_PACKET_LENGTH)
    left_length = (ulong)(MAX_PACKET_LENGTH - (net->write_pos - net->buff));
  else
    left_length = (ulong)(net->buff_end - net->write_pos);  /* ★ 阈值 = 缓冲区剩余空间 */

  if (len > left_length) {                    /* ★ 装不下 → 触发真正的发送 */
    if (net->write_pos != net->buff) {
      /* ① 先把缓冲区填满凑成一个满包发出 */
      memcpy(net->write_pos, packet, left_length);
      if (net_write_packet(net, net->buff,
                           (size_t)(net->write_pos - net->buff) + left_length))
        return true;
      net->write_pos = net->buff;
      packet += left_length;
      len -= left_length;
    }
    if (net->compress) {
      /* ② 压缩模式：按 16MB 切片直发（uncomp-len 只 3B，装不下更大） */
      left_length = MAX_PACKET_LENGTH;
      while (len > left_length) {
        if (net_write_packet(net, packet, left_length)) return true;
        packet += left_length;
        len -= left_length;
      }
    }
    if (len > net->max_packet) return net_write_packet(net, packet, len);
                                              /* ③ 大块直发，绕过拷贝 */
  }
  if (len > 0) memcpy(net->write_pos, packet, len);  /* ④ 装得下 → 只 memcpy，不发送 */
  net->write_pos += len;
  return false;
}
```

**★ `net_flush()`：两套序号的同步点**（`net_serv.cc:285`）：

```cpp
bool net_flush(NET *net) {
  bool error = false;
  if (net->buff != net->write_pos) {          /* ★ 空缓冲 flush 是 no-op */
    error = net_write_packet(net, net->buff, (size_t)(net->write_pos - net->buff));
    net->write_pos = net->buff;
  }
  /* Sync packet number if using compression */
  if (net->compress) net->pkt_nr = net->compress_pkt_nr;   /* ★★ 压缩模式下在此同步 */
  return error;
}
```
```cpp
/* net_serv.cc:264 —— 每个新命令序号归零 */
void net_clear(NET *net, bool check_buffer [[maybe_unused]]) {
  net->pkt_nr = net->compress_pkt_nr = 0;
  net->write_pos = net->buff;
}
```

**★ `net_realloc()`：扩容与硬上限**（`net_serv.cc:210`）：

```cpp
bool net_realloc(NET *net, size_t length) {
  if (length >= net->max_packet_size) {       /* ★ 硬上限：>= max_allowed_packet 直接报错 */
    net->error = NET_ERROR_SOCKET_RECOVERABLE;
    net->last_errno = ER_NET_PACKET_TOO_LARGE;
#ifdef MYSQL_SERVER
    my_error(ER_NET_PACKET_TOO_LARGE, MYF(0));
#endif
    return true;
  }
  pkt_length = (length + IO_SIZE - 1) & ~(IO_SIZE - 1);  /* ★ 按 IO_SIZE(4096) 向上取整 */
  ...
#ifdef MYSQL_SERVER
  net->buff = net->write_pos = buff;          /* ★★ 扩容后 write_pos 被复位到 buff */
#endif
  net->buff_end = buff + (net->max_packet = (ulong)pkt_length);
  return false;
}
```

**★★ 第三级：`net_write_raw_loop()` 与 `Bytes_sent`**（`net_serv.cc:997`）：

```cpp
static bool net_write_raw_loop(NET *net, const uchar *buf, size_t count) {
  unsigned int retry_count = 0;
  while (count) {
    size_t sentcnt = vio_write(net->vio, buf, count);   /* ★ VIO 层唯一出口 */
    if (sentcnt == VIO_SOCKET_ERROR) {
      if (net_should_retry(net, &retry_count)) continue;
      else break;
    }
    count -= sentcnt;
    buf += sentcnt;
#ifdef MYSQL_SERVER
    thd_increment_bytes_sent(sentcnt);        /* ★ Bytes_sent 状态变量累加 */
#endif
  }
  ...
}
```
> ⚠️ `net_should_retry()` 是 **static 内部函数**（`net_serv.cc:308`）；服务端受 `net->retry_count` 限制重试次数，客户端则"永远重试"。

### 1.10 结果集发送的三级缓冲、背压与 SHOW PROCESSLIST 状态

1.9 讲了**调用链**，本节讲**物理缓冲**——数据真正到客户端要过三道缓冲：

```
① MySQL 用户态：net_buffer（NET::buff，初始 = net_buffer_length，默认 16KB）
      ↓ my_net_write / net_flush → send()
② 服务端内核：socket send buffer（Linux /proc/sys/net/core/wmem_default ≈ 208KB）
      ↓ 网络（TCP 滑动窗口 + 拥塞控制）
③ 客户端内核：socket receive buffer → 应用读取
```

**背压（为什么卡在 Sending to client）**：客户端不消费 → ③ receive buffer 满 → TCP 窗口通告为 0 → ② send buffer 无法清空 → send buffer 满 → `send()` 阻塞 → MySQL 停在 1.9 的 `vio_write`。**此时 SQL 已经执行完**，只是结果发不出去。

**State 的判定是两个互斥分支**（`sql/sql_show.cc:2787`）：

```cpp
static const char *thread_state_info(THD *invoking_thd, THD *inspected_thd) {
  if (inspected_thd->get_protocol()->get_rw_status()) {
    if (...get_rw_status() == 2) return "Sending to client";   // 正在写网络
    if (inspected_thd->get_command() == COM_SLEEP) return "";
    return "Receiving from client";                             // 正在读
  } else {
    const char *proc_info = inspected_thd->proc_info_session(...);
    if (proc_info) return proc_info;                            // stage，如 "executing"
    if (inspected_thd->current_cond.load()) return "Waiting on cond";
  }
}
```

- `rw_status == 2` → **`Sending to client`**
- `rw_status == 0` → 显示当前 **stage**（`executing`；**8.0.17 之前叫 `Sending data`**，8.0.17 起合并为 `executing`）

**两个状态的实践判别**（排查方向完全不同）：

| State | 含义 | 排查方向 |
|-------|------|---------|
| `executing` / `Sending data` | 语句**正在执行**（含等锁、排序、聚合） | **服务端**：锁等待、执行计划、扫描行数 |
| `Sending to client` | **SQL 已执行完**，在往网络写 | **客户端**：消费慢、网络、结果集过大 |

**★★ `rw_status` 的赋值点与清零点**（★ 状态是"进出成对"的）：

```cpp
/* net_serv.cc:1304 —— 写方向：进入 net_write_packet 时置 2 */
  net->reading_or_writing = 2;      /* → "Sending to client" */
  ...
  res = net_write_raw_loop(net, packet, length);
  if (do_compress) my_free(const_cast<uchar *>(packet));
  net->reading_or_writing = 0;      /* ★ 退出时清零 */
```
> ★★ 关键：**`net_write_raw_loop()` 阻塞在 `vio_write()` 期间，状态一直是 2** —— 这就是背压时 PROCESSLIST 停在那里不动的机制。

```cpp
/* sql/protocol_classic.cc:3025 —— 读状态 */
uint Protocol_classic::get_rw_status() { return m_thd->net.reading_or_writing; }
```
```cpp
/* sql/sql_show.cc:2787 —— ★ 完整实现（8.0.39 没有 "Sending data" 字符串） */
static const char *thread_state_info(THD *invoking_thd, THD *inspected_thd) {
  if (inspected_thd->get_protocol()->get_rw_status()) {
    if (inspected_thd->get_protocol()->get_rw_status() == 2)
      return "Sending to client";                     /* ★ 2 = 正在 vio_write */
    if (inspected_thd->get_command() == COM_SLEEP) return "";
    return "Receiving from client";                   /* 1 = 正在读 */
  } else {
    MUTEX_LOCK(lock, &inspected_thd->LOCK_current_cond);
    const char *proc_info = inspected_thd->proc_info_session(invoking_thd);
    if (proc_info) return proc_info;                  /* PSI stage，如 "executing" */
    if (inspected_thd->current_cond.load()) return "Waiting on cond";
    return nullptr;
  }
}
```
> ★ PSI stage 常量在 `sql/mysqld.cc:12199+`（如 `stage_executing = {0, "executing", ...}`）。

**★ 超时：`net_read_timeout` 的两种语义**（`net_serv.cc:1350`，★ 少有人知）：

```cpp
#ifdef MYSQL_SERVER
  NET_SERVER *server_ext = static_cast<NET_SERVER *>(net->extension);
  if (server_ext) timeout_on_full_packet = server_ext->timeout_on_full_packet;
#endif
  ...
    if (timeout_on_full_packet) {
      time_t current_time = time(&current_time);
      if (static_cast<unsigned int>(current_time - start_time) > net->read_timeout) {
        is_packet_timeout = true;
        break;                    /* ★ 整包超时：从第一个字节起算总时长 */
      }
    }
  ...
    /* First packet always wait for net_wait_timeout */
    if (net->pkt_nr == 0 && (vio_was_timeout(net->vio) || is_packet_timeout)) {
      net->last_errno = ER_CLIENT_INTERACTION_TIMEOUT;   /* ★★ 等命令阶段用 net_wait_timeout */
      LogErr(INFORMATION_LEVEL, ER_NET_WAIT_ERROR2, timeout.c_str(),
             auth_id.auth_str().c_str());
    }
```
> ★★ `pkt_nr == 0` 意味着"这是这个命令的第一个包" —— 此时的超时按 **`wait_timeout`**（默认 28800）算而不是 `net_read_timeout`（30），并写一条 INFO 日志。这是"空闲连接被 wait_timeout 断掉"的实现点。

**★ 结果集发送的每列都走栈上缓冲，绕开 mem_root**（`sql/sql_class.cc:2863`）：

```cpp
bool THD::send_result_set_row(const mem_root_deque<Item *> &row_items) {
  char buffer[MAX_FIELD_WIDTH];                    /* ★ 栈上，有意避开 mem_root */
  String str_buffer(buffer, sizeof(buffer), &my_charset_bin);
  for (Item *item : VisibleFields(row_items)) {
    if (item->send(m_protocol, &str_buffer) || is_error()) return true;
    /*
      Reset str_buffer to its original state, as it may have been altered in
      Item::send().
    */
    str_buffer.set(buffer, sizeof(buffer), &my_charset_bin);  /* ★ 每列重置 */
  }
  return false;
}
```
> ★ 大结果集发送时**不产生任何 mem_root 分配**——否则每行分配会把 mem_root 撑爆（也解释了为什么 `kPreallocSz` 判定通常走 `ClearForReuse`）。

**为什么等锁也显示 `executing`**：stage 是**粗粒度**标记——`ExecuteIteratorQuery`（`sql/sql_union.cc:1673`）入口设置后**整个执行期保持**，不区分内部在做什么；锁等待发生在 handler 层（`ha_innobase::index_read`），不会触发状态切换。所以 `executing` 时间长 **≠** 在计算，要查 `performance_schema.data_lock_waits` / `SHOW ENGINE INNODB STATUS` 才能确认是否在等锁。

**纠偏：`net_buffer_length` 只是初始值，调大它不能消除背压**

- `NET::max_packet` 初始 = `net_buffer_length`（`sql/sql_client.cc:40`）
- 不够时通过 `net_realloc` **动态增长**（`sql-common/net_serv.cc:210`、`:1549`、`:1748`）

所以调大 `net_buffer_length` 只减少 realloc 次数。**瓶颈在客户端消费速度（TCP 窗口），不在用户态 buffer 大小**——即使 net buffer 装下整个结果集，数据仍要推到 ② send buffer，满了照样阻塞。

正确缓解方向：

| 方向 | 做法 |
|------|------|
| 根本 | 客户端及时消费：`mysql_store_result`（一次读完）优于 `mysql_use_result`（流式，客户端慢处理即卡服务端） |
| 减小压力 | 加 `LIMIT`、只取需要的列 |
| 排查 | 客户端 GC / 网络延迟 / 应用线程阻塞 |
| 治标 | 增大 OS 层 `net.core.wmem_default`——只缓冲更多，不根治 |

---

## 二、连接与 THD

**一个客户端连接 = 一个 `THD` 对象**（会话描述符），里面挂着 `LEX *lex;`（`sql/sql_class.h:983`）。

THD 的创建不在 `sql_connect.cc`，而在连接处理层：

```cpp
// sql/conn_handler/channel_info.cc:40
THD *Channel_info::create_thd() {
  Vio *vio_tmp = create_and_init_vio();
  if (vio_tmp == nullptr) return nullptr;
  THD *thd = new (std::nothrow) THD;                      // :46
  if (thd == nullptr) { vio_delete(vio_tmp); return nullptr; }
  thd->get_protocol_classic()->init_net(vio_tmp);         // :52  绑定 NET/Vio
  return thd;
}
```

`sql/sql_connect.cc` 里负责的是认证与连接状态：

| 函数 | 行号 | 作用 |
|------|------|------|
| `check_connection` | 441 | host / ACL 校验 |
| `login_connection` | 697 | → `acl_authenticate` |
| `prepare_new_connection_state` | 773 | 810 行调 `thd->init_query_mem_roots()` |
| **`thd_prepare_connection`** | **889** | 组合：893 第一次 `lex_start(thd)` + 认证 + 初始化 |
| `end_connection` | 732 | |
| `thd_connection_alive` | 932 | |

**★ thread_id 的分配**（`sql/sql_class.cc:1140`）：

```cpp
void THD::set_new_thread_id() {
  m_thread_id = Global_THD_manager::get_instance()->get_new_thread_id();
  variables.pseudo_thread_id = m_thread_id;
  thr_lock_info_init(&lock_info, m_thread_id, &COND_thr_lock);
}
```
```cpp
/* sql/mysqld_thd_manager.cc:249 —— 自增 + 撞号重试 */
my_thread_id Global_THD_manager::get_new_thread_id() {
  my_thread_id new_id;
  MUTEX_LOCK(lock, &LOCK_thread_ids);
  do {
    new_id = thread_id_counter++;
  } while (!thread_ids.insert_unique(new_id).second);   /* ★ 回绕/测试时撞号就继续自增 */
  return new_id;
}
```
> ★ `Global_THD_manager` 的 THD 列表**按 thread_id 分区**（`thd_partition()` + 每区一把锁），降低锁竞争：
> ```cpp
>   const int partition = thd_partition(thd->thread_id());
>   MUTEX_LOCK(lock_list, &LOCK_thd_list[partition]);
>   std::pair<THD_array::iterator, bool> insert_result =
>       thd_list[partition].insert_unique(thd);
> ```

**★★ `THD::release_resources()` 的清理顺序**（`sql/sql_class.cc:1338`，★ 十步）：

```cpp
void THD::release_resources() {
  Global_THD_manager::get_instance()->release_thread_id(m_thread_id);   /* ① 归还 thread_id */
  ...
  start_disposal();                                                     /* 生命周期标记 */
  if (is_classic_protocol() && get_protocol_classic()->get_vio()) {
    vio_delete(get_protocol_classic()->get_vio());                      /* ② 关 Vio */
    get_protocol_classic()->end_net();                                  /* ③ 关 NET */
  }
  mysql_audit_free_thd(this);                                           /* ④ 审计 per-THD 资源 */
  stmt_map.reset(); /* close all prepared statements */                 /* ⑤ 关所有 PS */
  if (!is_cleanup_done()) cleanup();
  mdl_context.destroy();                                                /* ⑥ MDL */
  ha_close_connection(this);                                            /* ⑦ 通知存储引擎 */
  plugin_thdvar_cleanup(this, m_enable_plugins);                        /* ⑧ 插件 THD 变量 */
  if (!external_store_.empty())
    (void)free_thd_store_resource(this, external_store_);               /* ⑨ component 私有数据 */
  ...
  mysql_mutex_lock(&LOCK_status);
  add_to_status(&global_status_var, &status_var);                       /* ⑩★ 会话状态汇入全局 */
#ifdef HAVE_PSI_THREAD_INTERFACE
  if (m_psi != nullptr) PSI_THREAD_CALL(aggregate_thread_status)(m_psi);
#endif
  status_var_aggregated = true;      /* ★★ 保证不重复汇入 */
  mysql_mutex_unlock(&LOCK_status);
  ...
}
```
> ★★ 第 ⑩ 步解释了为什么 `SHOW GLOBAL STATUS` 里 `Bytes_sent` **不含当前断开中的会话的最后一截**——汇入发生在 `release_resources()`，之后线程状态就 freeze 了。
> ★ `~THD()`（`sql_class.cc:1417`）有幂等保护：`if (!release_resources_done()) release_resources();`

**★ THD 构造时的协议对象**（`sql/sql_class.cc:626`）：

```cpp
THD::THD(bool enable_plugins)
    : Query_arena(&main_mem_root, STMT_REGULAR_EXECUTION),  /* ★ THD 自己就是个 Query_arena */
      main_lex(new LEX),                                    /* ★ 堆上 new，非 mem_root */
      lex(main_lex.get()),
      m_main_security_ctx(this),                            /* ★ Security_context 是 THD 的成员不是指针 */
      m_security_ctx(&m_main_security_ctx),
      protocol_text(new Protocol_text),                     /* ★ COM_QUERY 用 */
      protocol_binary(new Protocol_binary),                 /* ★ COM_STMT_* 用 */
      ...
      main_mem_root(key_memory_thd_main_mem_root,
                    global_system_variables.query_alloc_block_size),
```

### 握手与认证（`sql/auth/sql_authentication.cc`）

> 本节是主链视角（握手在整个链路上的位置）。握手包的字节布局、认证插件协商状态机、凭据校验算法、失败防御等**协议细节**见 [`../auth/authentication.md`](../auth/authentication.md)。

```
客户端 TCP 连接
   │
   ▼
服务端先发 HandshakeV10 初始握手包   send_server_handshake_packet :1602
   │   protocol_version(0x0A) + server_version + thread_id
   │   + scramble 前半 + server 能力 + charset + auth plugin name
   ▼
客户端回 HandshakeResponse41（含用户名、密码 scramble、client 能力）
   │
   ▼
login_connection（sql_connect.cc:697）
   → check_connection（:707）
   → acl_authenticate(thd, COM_CONNECT)   sql_authentication.cc:3834
   │   认证成功 → 回 OK 包；失败 → 回 ERR 包
   ▼
进入命令循环
```

> 认证阶段用 `connect_timeout` 作为读写超时，完成后恢复为 `net_read_timeout` / `net_write_timeout`（`sql_connect.cc:703-721`）。

---

## 三、线程模型

**one-thread-per-connection**（每个连接一个 OS 线程）：

| 环节 | 位置 |
|------|------|
| 调度器枚举 `SCHEDULER_ONE_THREAD_PER_CONNECTION` | `conn_handler/connection_handler_manager.h:111` |
| 选中该调度器 | `connection_handler_manager.cc:155-156` |
| 创建 OS 线程 | `connection_handler_per_thread.cc:417` `mysql_thread_create(..., handle_connection, ...)` |
| 线程主函数 | `connection_handler_per_thread.cc:246 handle_connection()` |
| 构造 THD | 同文件 `194 init_new_thd()` → `channel_info->create_thd()`（195）、`store_globals()`（226） |
| 认证 + 命令循环 | 同文件 `301 thd_prepare_connection()`；`304-306 while (thd_connection_alive(thd)) { if (do_command(thd)) break; }` |
| 销毁 | `308 end_connection`、`313 release_resources`、`328 delete thd`；线程回缓存 `338 block_until_new_connection()` |

> 关键点：**命令循环就在这里** —— `while (thd_connection_alive(thd)) { do_command(thd); }`。一个连接的所有 SQL 都在这个循环里串行处理。

**★★★ thread cache 缓存的是 OS 线程，不是 THD**（⚠️ 这是最容易搞错的一点）：

```cpp
/* sql/conn_handler/connection_handler_per_thread.cc:256 */
  for (;;) {
    // note: create THD for each connection!        /* ★★ 官方注释：每条连接新建 THD */
    THD *thd = init_new_thd(channel_info);
    ...
    /* 命令循环 */
    while (thd_connection_alive(thd)) {
      if (do_command(thd)) break;                   /* ★ do_command 返回 1 → 跳出，结束连接 */
    }
    end_connection(thd);
    close_connection(thd, 0, false, false);
    thd->get_stmt_da()->reset_diagnostics_area();
    thd->release_resources();
    thd_manager->remove_thd(thd);
    Connection_handler_manager::dec_connection_count();   /* ★ Threads_connected -1 */
    delete thd;                                           /* ★★★ THD 销毁 —— 缓存的不是 THD */

    // Server is shutting down so end the pthread.
    if (connection_events_loop_aborted()) break;

    channel_info = Per_thread_connection_handler::block_until_new_connection();
                                                    /* ★★★ 线程不退出，睡眠等下一个连接 */
    if (channel_info == nullptr) break;             /* 缓存已满/缩容 → 线程真的退出 */
    pthread_reused = true;
    ...
  }
  my_thread_end();
  my_thread_exit(nullptr);
```
> ★★ 所以"thread cache"的准确说法是：**OS 线程在连接结束后不退出，阻塞在 `block_until_new_connection()` 上等待复用**；THD 则每次都 `new` / `delete`。

**★ `block_until_new_connection()`：缓存的物理形态**（`connection_handler_per_thread.cc:136`）：

```cpp
Channel_info *Per_thread_connection_handler::block_until_new_connection() {
  Channel_info *new_conn = nullptr;
  mysql_mutex_lock(&LOCK_thread_cache);
  if (blocked_pthread_count < max_blocked_pthreads && !shrink_cache) {
                                       /* ★ 缓存未满（< thread_cache_size）才允许入缓存 */
    DBUG_POP();
    assert(!_db_is_pushed_());
    blocked_pthread_count++;           /* ★ = 状态变量 Threads_cached +1 */
    while (!connection_events_loop_aborted() && !wake_pthread && !shrink_cache)
      mysql_cond_wait(&COND_thread_cache, &LOCK_thread_cache);
                                       /* ★★ 线程在这里睡着 */
    blocked_pthread_count--;
    if (shrink_cache && blocked_pthread_count <= max_blocked_pthreads)
      mysql_cond_signal(&COND_flush_thread_cache);
    if (wake_pthread) {
      wake_pthread--;
      if (!waiting_channel_info_list->empty()) {
        new_conn = waiting_channel_info_list->front();   /* 从队列头取走一个待处理连接 */
        waiting_channel_info_list->pop_front();
      } else { assert(0); }
    }
  }
  mysql_mutex_unlock(&LOCK_thread_cache);
  return new_conn;                     /* nullptr = 该退出了 */
}
```

**★ `add_connection()`：先复用，再新建**（`connection_handler_per_thread.cc:401`）：

```cpp
bool Per_thread_connection_handler::add_connection(Channel_info *channel_info) {
  int error = 0;
  my_thread_handle id;
  if (!check_idle_thread_and_enqueue_connection(channel_info)) return false;
                                       /* ★ 先试复用；成功直接返回，不建线程 */
  channel_info->set_prior_thr_create_utime();
  error = mysql_thread_create(key_thread_one_connection, &id, &connection_attrib,
                              handle_connection, (void *)channel_info);
  if (error) {
    connection_errors_internal++;
    if (!create_thd_err_log_throttle.log())     /* ★ 建线程失败的错误日志要限速 */
      LogErr(ERROR_LEVEL, ER_CONN_PER_THREAD_NO_THREAD, error);
    channel_info->send_error_and_close_channel(ER_CANT_CREATE_THREAD, error, true);
    Connection_handler_manager::dec_connection_count();
    return true;
  }
  Global_THD_manager::get_instance()->inc_thread_created();  /* ★ Threads_created +1 */
  return false;
}
uint Per_thread_connection_handler::get_max_threads() const { return max_connections; }
```
```cpp
/* :384 —— 复用：精确唤醒一个 */
bool Per_thread_connection_handler::check_idle_thread_and_enqueue_connection(
    Channel_info *channel_info) {
  bool res = true;
  mysql_mutex_lock(&LOCK_thread_cache);
  if (Per_thread_connection_handler::blocked_pthread_count > wake_pthread) {
                                       /* ★ 有"真的闲着"的线程（> 已被唤醒的）才复用 */
    waiting_channel_info_list->push_back(channel_info);
    wake_pthread++;
    mysql_cond_signal(&COND_thread_cache);   /* ★ 只 signal 一个 */
    res = false;
  }
  mysql_mutex_unlock(&LOCK_thread_cache);
  return res;
}
```
> ⚠️ **真名核查**：`Per_thread_connection_handler` 类定义在 **`sql/conn_handler/connection_handler_impl.h:42`**（不存在 `connection_handler_per_thread.h`）。

**★ `max_connections` 之后还留 1 个名额给 SUPER**（`connection_handler_manager.cc:113`）：

```cpp
bool Connection_handler_manager::check_and_incr_conn_count(
    bool is_admin_connection) {
  bool connection_accepted = true;
  mysql_mutex_lock(&LOCK_connection_count);
  /*
    Here we allow max_connections + 1 clients to connect
    (by checking before we increment by 1).
    The last connection is reserved for SUPER users. ...
  */
  if (connection_count > max_connections && !is_admin_connection) {
    connection_accepted = false;
    m_connection_errors_max_connection++;
  } else {
    ++connection_count;
    if (connection_count > max_used_connections) {
      max_used_connections = connection_count;      /* ★ Max_used_connections 在此刷新 */
      max_used_connections_time = time(nullptr);
    }
  }
  mysql_mutex_unlock(&LOCK_connection_count);
  return connection_accepted;
}
```
```cpp
/* :247 —— 新连接入口：两道闸 */
void Connection_handler_manager::process_new_connection(Channel_info *channel_info) {
  if (connection_events_loop_aborted() ||
      !check_and_incr_conn_count(channel_info->is_admin_connection())) {
    channel_info->send_error_and_close_channel(ER_CON_COUNT_ERROR, 0, true);
                                       /* ★ "Too many connections" */
    delete channel_info;
    return;
  }
  if (m_connection_handler->add_connection(channel_info)) {   /* 复用 or 新建 */
    inc_aborted_connects();
    delete channel_info;
  }
}
```

**★ 三种调度器与线程池插件接入点**（`connection_handler_manager.h:111`）：

```cpp
  enum scheduler_types {
    SCHEDULER_ONE_THREAD_PER_CONNECTION = 0,   /* "one-thread-per-connection"（默认） */
    SCHEDULER_NO_THREADS,                      /* "no-threads" */
    SCHEDULER_TYPES_COUNT                      /* ★ = 2，兼作 "loaded-dynamically" */
  };
```
```cpp
/* connection_handler_manager.cc:264 —— 线程池插件接进来后 thread_handling 变成 2 */
void Connection_handler_manager::load_connection_handler(
    Connection_handler *conn_handler) {
  assert(Connection_handler_manager::thread_handling != SCHEDULER_TYPES_COUNT);
  m_saved_connection_handler = m_connection_handler;
  m_saved_thread_handling = Connection_handler_manager::thread_handling;
  m_connection_handler = conn_handler;
  Connection_handler_manager::thread_handling = SCHEDULER_TYPES_COUNT;
  max_threads = m_connection_handler->get_max_threads();
}
```
> ★★ 这就是 `SHOW VARIABLES LIKE 'thread_handling'` 会显示 **`loaded-dynamically`** 的原因——装了线程池插件后 `thread_handling` 被置为 `SCHEDULER_TYPES_COUNT(2)`。
> ★ 三个取值：`"one-thread-per-connection", "no-threads", "loaded-dynamically"`（`sql/sys_vars.cc:4047`），默认下标 0。

**★ `Channel_info`：连接通道抽象**（`sql/conn_handler/channel_info.h:39`）：

```cpp
/**
  This abstract base class represents connection channel information
  about a new connection. Its subclasses encapsulate differences
  between different connection channel types.

  Currently we support local and TCP/IP sockets (all platforms),
  named pipes and shared memory (Windows only).
*/
class Channel_info {
 protected:
  virtual Vio *create_and_init_vio() const = 0;   /* ★ 纯虚：各通道各自实现 */
 public:
  virtual THD *create_thd();
  virtual void send_error_and_close_channel(uint errorcode, int error, bool senderror);
  virtual bool is_admin_connection() const { return false; }  /* 管理连接覆写为 true */
};
```
```cpp
/* channel_info.cc:40 */
THD *Channel_info::create_thd() {
  Vio *vio_tmp = create_and_init_vio();
  if (vio_tmp == nullptr) return nullptr;
  THD *thd = new (std::nothrow) THD;              /* ★★ THD 在这里被 new 出来 */
  if (thd == nullptr) { vio_delete(vio_tmp); return nullptr; }
  thd->get_protocol_classic()->init_net(vio_tmp);  /* ★ Vio 挂到 NET */
  return thd;
}
```

**★ `thread_cache_size` 的动态覆盖**（`sql/mysqld.cc:5055`，⚠️ 注册默认与实际默认不同）：

```cpp
  /* 注册时是 DEFAULT(0)（sys_vars.cc:5165），但运行期被改成： */
  min(8 + max_connections / 100, 100)
```
```cpp
/* connection_handler_per_thread.cc:357 —— 缩容时只唤醒"多出来"的那部分 */
void Per_thread_connection_handler::modify_thread_cache_size(
    const ulong thread_cache_size) {
  mysql_mutex_lock(&LOCK_thread_cache);
  if (thread_cache_size >= blocked_pthread_count) { ...; return; }
  shrink_cache = true;
  if (thread_cache_size == 0) {
    mysql_cond_broadcast(&COND_thread_cache);     /* 归零 → 唤醒全部让它们退出 */
  } else {
    ulong num_threads = blocked_pthread_count - thread_cache_size;
    for (ulong i = 0; i < num_threads; i++)
      mysql_cond_signal(&COND_thread_cache);      /* 只唤醒多出来的部分 */
  }
  while (blocked_pthread_count > thread_cache_size)
    mysql_cond_wait(&COND_flush_thread_cache, &LOCK_thread_cache);
  shrink_cache = false;
  mysql_mutex_unlock(&LOCK_thread_cache);
}
void Per_thread_connection_handler::kill_blocked_pthreads() {
  modify_thread_cache_size(0);                    /* shutdown 时清空缓存 */
}
```

---

## 四、do_command：读一个包

`sql/sql_parse.cc:1308`。核心三步：拿 NET、阻塞读、交给分派：

```cpp
// sql/sql_parse.cc:1338-1341
net = thd->get_protocol_classic()->get_net();
my_net_set_read_timeout(net, thd->variables.net_wait_timeout);
net_new_transaction(net);
...
// sql/sql_parse.cc:1381
rc = thd->get_protocol()->get_command(&com_data, &command);   // 阻塞读
...
// sql/sql_parse.cc:1440
return_value = dispatch_command(thd, &com_data, command);
```

**命令码就是包的首字节**：

```cpp
// sql/protocol_classic.cc:2991
int Protocol_classic::get_command(COM_DATA *com_data, enum_server_command *cmd) {
  if (int rc = read_packet()) return rc;                    // :2994 → read_packet() :1410 → my_net_read
  if (input_packet_length == 0) { input_raw_packet[0] = (uchar)COM_SLEEP; input_packet_length = 1; }
  input_raw_packet[input_packet_length] = '\0';
  *cmd = (enum enum_server_command)(uchar)input_raw_packet[0];   // :3013  首字节
  if (*cmd >= COM_END) *cmd = COM_END;
  input_packet_length--; input_raw_packet++;                     // :3019  跳过命令字节
  return parse_packet(com_data, *cmd);                           // :3022
}
```

`COM_*` 枚举在 `include/my_command.h:48-103`（`COM_QUERY = 4`、`COM_STMT_PREPARE = 75`、`COM_STMT_EXECUTE = 76`，哨兵 `COM_END = 102`）。

**为什么叫 `MYSQLlex` 而不是 `yylex`**：`sql/CMakeLists.txt:1336` 用 `--name-prefix=MYSQL` 编译 bison，所以 `yyparse → MYSQLparse`、`yylex → MYSQLlex`。

---

## 五、dispatch_command：按命令码分派

`sql/sql_parse.cc:1688`，`switch (command)` 在 `:1814`。进入 switch 前的公共动作：

```cpp
// sql/sql_parse.cc:1714-1720, 1809
thd->set_command(command);
thd->enable_slow_log = true;
thd->lex->sql_command = SQLCOM_END;
thd->set_query_id(next_query_id());                            // :1768
...
if (mysql_audit_notify(thd, AUDIT_EVENT(MYSQL_AUDIT_COMMAND_START), command, ...))  // :1809
    goto done;
```

主要 case：

| 命令 | 行号 | 动作 |
|------|------|------|
| `COM_INIT_DB` | 1815 | `mysql_change_db` |
| `COM_STMT_EXECUTE` | 1932 | `mysqld_stmt_execute` |
| `COM_STMT_PREPARE` | 1973 | `mysqld_stmt_prepare` |
| `COM_STMT_CLOSE` / `RESET` / `FETCH` / `SEND_LONG_DATA` | 1992 / 2001 / 1950 / 1961 | |
| **`COM_QUERY`** | **2011** | 见下 |
| `COM_QUIT` / `COM_PING` / `COM_SET_OPTION` | 2257 / 2369 / 2406 | |
| `default` | 2443 | `ER_UNKNOWN_COM_ERROR` |

`COM_QUERY` 分支（`:2011`）：

```cpp
    case COM_QUERY: {
      ...
      alloc_query(thd, com_data->com_query.query, com_data->com_query.length);  // :2016
      ...
      Parser_state parser_state;                       // :2034
      if (parser_state.init(thd, thd->query().str, thd->query().length)) break; // :2035
      ...
      dispatch_sql_command(thd, &parser_state);        // :2055
```

### 命令码全景（`include/my_command.h`）

★ **一个常见错误**：`enum enum_server_command` **不在 `include/mysql_com.h`**——8.0 已迁到 **`include/my_command.h`**。枚举无显式赋值，值就是 0x00 起的声明序号。

已废弃/拒绝的命令（落到 `dispatch_command` 的 default 分支，回 `ER_UNKNOWN_COM_ERROR`，**不断连**）：

```cpp
    case COM_SLEEP:
    case COM_CONNECT:         // Impossible here
    case COM_TIME:            // Impossible from client
    case COM_DELAYED_INSERT:  // INSERT DELAYED has been removed.
    case COM_END:
    default:
      my_error(ER_UNKNOWN_COM_ERROR, MYF(0));
      break;
```

| 类别 | 命令 |
|---|---|
| 日常命令 | `COM_QUERY(0x03)` / `COM_INIT_DB(0x02)` / `COM_PING(0x0E)` / `COM_QUIT(0x01)` / `COM_RESET_CONNECTION(0x1F)` |
| 预编译 | `COM_STMT_PREPARE(0x16)` / `EXECUTE(0x17)` / `SEND_LONG_DATA(0x18)` / `CLOSE(0x19)` / `RESET(0x1A)` / `FETCH(0x1C)`（详见 [`12_prepared_statement.md`](12_prepared_statement.md)） |
| 复制 | `COM_BINLOG_DUMP(0x12)` / `COM_BINLOG_DUMP_GTID(0x1E)` / `COM_REGISTER_SLAVE(0x15)` |
| 其他 | `COM_STATISTICS(0x09)` / `COM_DEBUG(0x0D)` / `COM_CHANGE_USER(0x11)` / `COM_SET_OPTION(0x1B)` / `COM_CLONE(0x20)` / `COM_SUBSCRIBE_GROUP_REPLICATION_STREAM(0x21)` |
| Deprecated 但仍可用 | `COM_FIELD_LIST(0x04)` / `COM_REFRESH(0x07)` / `COM_PROCESS_INFO(0x0A)` / `COM_PROCESS_KILL(0x0C)` |
| 拒绝/废弃 | `COM_SLEEP(0x00)` / `COM_CREATE_DB(0x05)` / `COM_DROP_DB(0x06)` / `COM_DEPRECATED_1(0x08，原 COM_SHUTDOWN)` / `COM_CONNECT(0x0B)` / `COM_TIME(0x0F)` / `COM_DELAYED_INSERT(0x10)` / `COM_END(0x22)` |

★ 命令名对照表在 `sql_parse.cc` 的 `Command_names::m_names[]`；越界命令码在 `get_command()` 里被**钳到 `COM_END`**（`if (*cmd >= COM_END) *cmd = COM_END;`），空包兜底成 `COM_SLEEP` 后被拒。

### EOF_Packet 与 Deprecate EOF

`write_eof_packet()`（`sql/protocol_classic.cc`）——载荷 5 字节：

```
[0xFE:1][warnings:int2][status_flags:int2]     （pre-4.1 客户端只有 0xFE 一个字节）
```

出现位置：**列定义块之后**、**行流结束后**、PREPARE 响应的参数/列元数据块之后、`COM_STMT_FETCH` 每批之后。行生成中途失败时，结尾 EOF 由 **ERR 包**替代。

`CLIENT_DEPRECATE_EOF`（`1UL << 24`）：开启后 EOF 被**首字节 0xFE 的 OK 包**替代：

```cpp
// Protocol_classic::send_eof()
if (has_client_capability(CLIENT_DEPRECATE_EOF) &&
    (m_thd->get_command() != COM_BINLOG_DUMP &&
     m_thd->get_command() != COM_BINLOG_DUMP_GTID))
  retval = net_send_ok(thd, ..., /*eof_identifier=*/true);
else
  retval = net_send_eof(thd, ...);

// net_send_ok 里：
if (eof_identifier && protocol->has_client_capability(CLIENT_DEPRECATE_EOF))
  buff[0] = 254;   // OK with 0xFE header
else
  buff[0] = 0;
```

★ **`COM_BINLOG_DUMP` / `COM_BINLOG_DUMP_GTID` 例外**——仍用真 EOF，因为复制线程对 OK 包里的 session state 段没有意义。

### ★ 0xFE 首字节的三重歧义

同一个 `0xFE` 至少有三种含义：**EOF_Packet**、**Deprecate-EOF 的 OK 包**、连接阶段的 **AuthSwitchRequest**；行数据里 lenenc 整数的 8 字节长度形式也以 `0xFE` 开头。协议靠**包长度 + 阶段上下文**区分，没有专门的类型字段：

| 场景 | 区分手段 |
|---|---|
| EOF vs 0xFE-OK | **载荷长度**：EOF 恒 5 字节（pre-4.1 恒 1），0xFE-OK 恒 ≥ 7（含 affected_rows/last_insert_id/status/warnings）。连接器普遍以"payload ≤ 8 视为 EOF"判定 |
| AuthSwitchRequest | **阶段互斥**：它只出现在握手后、命令前的认证阶段（见 [`../auth/authentication.md`](../auth/authentication.md)），命令阶段不可能出现 |
| 行内 lenenc 0xFE | **元数据驱动**：行解析按列定义声明的类型/长度读取，不靠首字节猜包型 |

这是"协议无显式类型字段、全靠位置语义"设计的代价——MySQL 用二十年兼容性锁死了这套约定。

**★★ 官方的消歧规则原文**（`sql/protocol_classic.cc:988`，★ `@warning` 级别的提醒）：

```cpp
  @warning
  The @ref page_protocol_basic_eof_packet packet may appear in places where
  a @ref sect_protocol_basic_dt_int_le "Protocol::LengthEncodedInteger"
  may appear. You must check whether the packet length is less than 9 to
  make sure that it is a @ref page_protocol_basic_eof_packet packet.
```
> ★★ **"包长 < 9 才是 EOF"** —— 这是官方给出的唯一可靠判据，比"≤ 8"更准确地说是 `< 9`。

**★ EOF_Packet 的两种形态**（`protocol_classic.cc:1086`）：

```cpp
static bool write_eof_packet(THD *thd, NET *net, uint server_status,
                             uint statement_warn_count) {
  bool error;
  Protocol *protocol = thd->get_protocol();
  if (protocol->has_client_capability(CLIENT_PROTOCOL_41)) {
    uchar buff[5];                       /* ★ 41 协议：0xFE + warnings(2B) + status(2B) */
    uint tmp = min(statement_warn_count, 65535U);
    buff[0] = 254;                       /* ★ 0xFE */
    int2store(buff + 1, tmp);
    if (thd->is_fatal_error()) server_status &= ~SERVER_MORE_RESULTS_EXISTS;
    int2store(buff + 3, server_status);
    error = my_net_write(net, buff, 5);
  } else
    error = my_net_write(net, eof_buff, 1);   /* ★ 4.0 老客户端：就 1 个字节 0xFE */
  return error;
}
```
```cpp
/* protocol_classic.cc:975 */
static uchar eof_buff[1] = {(uchar)254}; /* Marker for end of fields */
```

**★★ OK_Packet 的 `0x00` / `0xFE` 双形态**（`protocol_classic.cc:862`，★ 参数名就叫 `eof_identifier`）：

```cpp
/**
  @param eof_identifier          when true [FE] will be set in OK header
                                 else [00] will be used
*/
static bool net_send_ok(THD *thd, uint server_status, uint statement_warn_count,
                        ulonglong affected_rows, ulonglong id,
                        const char *message, bool eof_identifier) {
  ...
  /*
    Use 0xFE packet header if eof_identifier is true
    unless we are talking to old client
  */
  if (eof_identifier && (protocol->has_client_capability(CLIENT_DEPRECATE_EOF)))
    buff[0] = 254;      /* ★★ 0xFE 形态：EOF 位置的 OK 包 */
  else
    buff[0] = 0;        /* ★★ 0x00 形态：普通 OK 包 */

  pos = net_store_length(buff + 1, affected_rows);   /* length-encoded integer */
  pos = net_store_length(pos, id);
```
```cpp
/* protocol_classic.cc:1299 / :1317 —— 两个调用点，eof_identifier 一 false 一 true */
bool Protocol_classic::send_ok(...) {
  const bool retval = net_send_ok(m_thd, server_status, statement_warn_count,
                                  affected_rows, last_insert_id, message, false);
}
bool Protocol_classic::send_eof(uint server_status, uint statement_warn_count) {
  /*
    Normally end of statement reply is signaled by OK packet, but in case
    of binlog dump request an EOF packet is sent instead. Also, old clients
    expect EOF packet instead of OK
  */
  if (has_client_capability(CLIENT_DEPRECATE_EOF) &&
      (m_thd->get_command() != COM_BINLOG_DUMP &&          /* ★★ binlog dump 永远发真 EOF */
       m_thd->get_command() != COM_BINLOG_DUMP_GTID))
    retval = net_send_ok(m_thd, server_status, statement_warn_count, 0, 0,
                         nullptr, true);                    /* ★ 0xFE 形态的 OK */
  else
    retval = net_send_eof(m_thd, server_status, statement_warn_count);
}
```
```cpp
/* include/mysql_com.h:691 —— 那个开关 */
#define CLIENT_DEPRECATE_EOF (1UL << 24)
```
> ★★ 有趣的一点：`CLIENT_DEPRECATE_EOF` 开启时，虽然发的是"OK 包"，但**首字节是 `0xFE`**——所以从抓包看它长得跟 EOF 一模一样，只是更长。这就是"三重歧义"的根源。

**★ lenenc 整数的 254 分支**（`mysys/pack.cc:89`，第三重歧义）：

```cpp
uint64_t net_field_length_ll(const uchar **packet) {
  const uchar *pos = *packet;
  if (*pos < 251) { ... (*packet)++; return (uint64_t)*pos; }
  if (*pos == 251) { ... return (uint64_t)NULL_LENGTH; }    /* ★ 251 = NULL */
  if (*pos == 252) { ... }
  if (*pos == 253) { ... }
  assert(*pos == 254);                                       /* ★ 254 = 8 字节 */
  *packet += 9;
  return (uint64_t)uint8korr(pos + 1);
}
```

### LOCAL INFILE：一条服务端主动发起的文件读取

完整时序（全部源码直证）：

```
C→S: COM_QUERY "LOAD DATA LOCAL INFILE 'f' INTO TABLE t"
S:   能力双检（sql_load.cc）：
       has_client_capability(CLIENT_LOCAL_FILES) && opt_local_infile
       否则 ER_CLIENT_LOCAL_FILES_DISABLED
S→C: 0xFB + filename            ← ★ 服务端主动请求客户端读文件！
C→S: 文件内容分片（16MB 分片规则同前）
C→S: 空包（长度 0）              ← 结束标记；客户端拒绝时也必须发空包
S→C: OK / ERR
```

★ **请求包的真实字节是 `0xFB (251) + filename`**，由 `net_request_file()` 发出——而这个函数的实现**不在** `net_serv.cc`，在 **`sql/rpl_replica.cc`**（复制代码遗留位置，被 `sql_load.cc` 复用）：

```cpp
bool net_request_file(NET *net, const char *fname) {
  return net_write_command(net, 251, pointer_cast<const uchar *>(fname),
                           strlen(fname), pointer_cast<const uchar *>(""), 0);
}
```

服务端收数据时走 `READ_NET` io_cache（`sql_load.cc`：`cache.read_function = _my_b_net_read`）。

**开关**：服务端 `local_infile` 系统变量**默认 `false`**；客户端 `MYSQL_OPT_LOCAL_INFILE` 选项等价于设置 `CLIENT_LOCAL_FILES` 能力位。**两端都允许才生效**——服务端 `&&` 双检，客户端 `handle_local_infile()` 检查自己的 client_flag，否则自己拒绝并发空包报 `CR_LOAD_DATA_LOCAL_INFILE_REJECTED`。

★ **为什么默认全关**：文件请求是**服务端在命令响应流里主动下发**的，文件名由服务端任意指定、客户端无确认 UI——一个恶意/被劫持的服务端可以让任何连上来的客户端库读取**客户端机器上**任意可读文件并上传（经典 "MySQL protocol file read" 攻击）。防护即三件：`local_infile` 默认 false、必须协商 `CLIENT_LOCAL_FILES`、`check_one_table_access` 权限检查（LOCAL 时**不需要** `FILE_ACL`，`m_is_local_file ? 0 : FILE_ACL`）。客户端侧还有 `mysql_set_local_infile_handler()` 可整体替换读文件回调。

---

### 5.x 多语句（multi-statement）处理

一条 `COM_QUERY` 里带 `;` 分隔的多条语句，是协议层最容易出正确性问题的一处。

#### 切分方式：不是预先 split，而是语法器打点 + 循环再解析

"刀"在语法器里——`sql_yacc.yy` 的 `sql_statement` 产生式，归约完一条语句、看到 `;` 时决定要不要继续：

```
sql_statement:
          simple_statement_or_begin
          {
            Lex_input_stream *lip = YYLIP;

            if (YYTHD->get_protocol()->has_client_capability(CLIENT_MULTI_QUERIES) &&
                lip->multi_statements &&
                ! lip->eof())
            {
              /*
                We found a well formed query, and multi queries are allowed:
                - force the parser to stop after the ';'
                - mark the start of the next query for the next invocation
                  of the parser.
              */
              lip->next_state= MY_LEX_END;
              lip->found_semicolon= lip->get_ptr();
            }
            else
              lip->found_semicolon= NULL;   // Single query, terminated.
          }
          ';'
          opt_end_of_input
```

所以模型是「**解析 → 打点 → 执行 → 再解析剩余 → 再打点 → …**」，而不是先切成 N 条字符串。

`dispatch_sql_command()` 读取该点并把 THD 的 query 长度截到 `;` **之前**（去掉 `;`）：

```cpp
    qlen = found_semicolon ? (found_semicolon - thd->query().str)
                           : thd->query().length;
    /*
      We set thd->query_length correctly to not log several queries, when we
      execute only first. We set it to not see the ';' otherwise it would get
      into binlog and Query_log_event::print() would give ';;' output.
    */
```

循环体在 `dispatch_command()` 的 COM_QUERY 分支：

```cpp
      while (!thd->killed && (parser_state.m_lip.found_semicolon != nullptr) &&
             !thd->is_error()) {
        ...
        thd->update_slow_query_status();
        thd->send_statement_status();     // 本条的结果集结束
        ...
        thd->set_query(beginning_of_next_stmt, length);
        thd->set_query_id(next_query_id());
        thd->status_var.questions++;
        thd->set_time();                  // 重置本条语句的起始时间
        parser_state.reset(beginning_of_next_stmt, length);
        dispatch_sql_command(thd, &parser_state);
```

#### 结果集序列：`SERVER_MORE_RESULTS_EXISTS`

```cpp
#define SERVER_MORE_RESULTS_EXISTS 8   /**< Multi query - next query exists */
```

**置位发生在执行当前语句之前**（只要解析出后面还有）：

```cpp
        if (found_semicolon) {
          lex->safe_to_cache_query = false;
          thd->server_status |= SERVER_MORE_RESULTS_EXISTS;
        }
```

于是本条的 OK/EOF 包带该标志，客户端据此继续读下一个结果集。

**清位**：每条语句开头 `reset_for_next_command()` 清 `SERVER_STATUS_CLEAR_SET`（含该位）；**出错时主动掐断**：

```cpp
  /* Abort multi-result sets */
  thd->server_status &= ~SERVER_MORE_RESULTS_EXISTS;
```

> 顺带：`CALL` 复用同一个标志位（存储过程可能返回多个结果集），因此要求客户端有 `CLIENT_MULTI_RESULTS` 能力，否则报 `ER_SP_BADSELECT`。

#### 两道闸：客户端能力位 + 服务端内部开关

**闸一**——`CLIENT_MULTI_STATEMENTS`（旧名 `CLIENT_MULTI_QUERIES`）。不声明就只执行第一条（多余语句在同一次解析里报语法错误）。可运行时开关（`COM_SET_OPTION` 的 `MYSQL_OPTION_MULTI_STATEMENTS_ON/OFF`）。

**闸二**——`Lex_input_stream::multi_statements`，默认 true，但在**所有"内部/预编译"解析路径被显式关掉**：

```cpp
  parser_state.m_lip.multi_statements = false;   // PREPARE 路径
  parser_state.m_lip.multi_statements = false;   // reprepare（注释：A safety measure.）
  ps.m_lip.multi_statements = false;             // 动态 SQL / EXECUTE IMMEDIATE
```

⇒ **预处理语句与存储程序内部的文本永远不会被拆成多语句。**

#### 一条失败后，后续不再执行

循环条件直接给出答案：`!thd->is_error()` 为假即退出；同时发错误包前会清掉 `SERVER_MORE_RESULTS_EXISTS`，客户端不会再等。`thd->killed` 同理。

#### 每条语句各自的诊断区与权限

每条语句进入 `dispatch_sql_command()` 第一件事是 `mysql_reset_thd_for_next_command(thd)` → `reset_for_next_command()`：

```cpp
  thd->server_status &= ~SERVER_STATUS_CLEAR_SET;
  thd->clear_error();
  thd->get_stmt_da()->reset_diagnostics_area();
  thd->get_stmt_da()->reset_statement_cond_count();
  ...
  /*
    In case we're processing multiple statements we need to checkout a new
    acl access map here as the global acl version might have increased due to
    a grant/revoke or flush.
  */
  thd->security_context()->checkout_access_maps();
```

⇒ **诊断区每条语句各自清空**（不是整批共享）；**ACL map 每条重新 checkout**——这点容易被忽略：同一批里前一条 GRANT，后一条应当立刻看到新权限（详见 [`../infra/diagnostics.md`](../infra/diagnostics.md)）。

> ⚠️ **关于注入防护**：源码里**没有**针对 multi-statement 注入的专门注释或代码。能证实的防护只有"opt-in 本身"（三个条件同时成立才切分）。真正的防注入落在客户端库与连接器。唯一沾边的注释在解析失败分支，讲的是**日志取舍**而非防护——并且明确说明为了便于定位 SQL 注入，选择把坏查询文本记进日志。

---

## 六、dispatch_sql_command（原 mysql_parse）

> ⚠️ **版本差异**：`mysql_parse()` 在 8.0.3x 已**重命名**为 `dispatch_sql_command()`（`sql/sql_parse.h:104` 声明，`sql/sql_parse.cc:5242` 定义）。网上大量资料仍写 `mysql_parse`，对不上时别慌。

主流程：

```cpp
// sql/sql_parse.cc:5242
void dispatch_sql_command(THD *thd, Parser_state *parser_state) {
  statement_id_to_session(thd);
  mysql_reset_thd_for_next_command(thd);                 // :5248
  thd->reset_rewritten_query();                          // :5251
  lex_start(thd);                                        // :5253  LEX 初始化
  thd->m_parser_state = parser_state;
  invoke_pre_parse_rewrite_plugins(thd);                 // :5256  改写插件（解析前）
  thd->m_parser_state = nullptr;
  ...
  if (!err) {
    err = parse_sql(thd, parser_state, nullptr);         // :5271  真正解析
    if (!err) err = invoke_post_parse_rewrite_plugins(thd, false);  // :5272
  }
  ...
  if (thd->rewritten_query().length() == 0) mysql_rewrite_query(thd);   // :5302 脱敏
  ...
  error = mysql_execute_command(thd, true);              // :5374  进入执行器
```

`parse_sql()` 在 `sql/sql_parse.cc:7067`，内部三个要点：

1. `:7135` 给 mem_root 设 `parser_max_mem_size` 上限（防单条 SQL 撑爆内存）+ OOM handler
2. `:7142` 调 `thd->sql_parser()`
3. `:7213` 清 `m_parser_state`

真正的解析入口：

```cpp
// sql/sql_class.cc:3048
bool THD::sql_parser() {
  ...
  Parse_tree_root *root = nullptr;
  if (MYSQLparse(this, &root) || is_error()) { cleanup_after_parse_error(); return true; }  // :3067
  if (root != nullptr && lex->make_sql_cmd(root)) return true;                              // :3077
```

**注意 `root` 是局部变量** —— Parse Tree 的根只在 `sql_parser()` 里存在，转成 `Sql_cmd*` 后就没人引用它了（详见 03）。

`mysql_execute_command()` 定义在 `sql/sql_parse.cc:2946`，是执行器的总入口（后续篇章）。

---

## 七、LEX 的生命周期

| 事项 | 位置 |
|------|------|
| `lex_start()` | **`sql/sql_lex.cc:511`**（`lex->reset()` :517、`new_top_level_query()` :521） |
| `LEX::reset()` | `sql/sql_lex.cc:418` |
| `LEX::new_top_level_query()` | `sql/sql_lex.cc:780` |
| 挂在 THD 上 | `sql/sql_class.h:983 LEX *lex;` |
| `lex_end()` | `sql/sql_lex.cc:535` |
| `LEX::destroy()` | `sql/sql_lex.h:4379` |
| `LEX::~LEX()` | `sql/sql_lex.cc:403` |
| 首次创建 | `sql/sql_connect.cc:893`（`thd_prepare_connection` 里第一次 `lex_start`） |
| 语句末重置 | `sql/sql_parse.cc:5248`（下一条开头）+ `5419-5421`（本条结尾） |

**语句结束的收尾三连**：

```cpp
// sql/sql_parse.cc:5419-5421
thd->lex->destroy();          // 销毁 LEX 自有对象（临时表等）
thd->end_statement();         // → lex_end()，注释明确写 "Don't free mem_root"
thd->cleanup_after_query();   // 清理 Item 列表，不碰 mem_root
```

**常规语句复用同一个 `thd->lex`**（`sql_class.h:977` 注释），不是每条语句新建一个。

```cpp
/* sql/sql_class.h:972 —— 权威注释：复用 vs 新建的分界线 */
 private:
  /**
    The lex to hold the parsed tree of conventional (non-prepared) queries.
    Whereas for prepared and stored procedure statements we use an own lex
    instance for each new query, for conventional statements we reuse
    the same lex. (@see dispatch_sql_command for details).
  */
  std::unique_ptr<LEX> main_lex;
 public:
  LEX *lex;                                        // parse tree descriptor
```
> ⚠️ **真名核查**：是 `LEX *lex`（`:983`）+ `std::unique_ptr<LEX> main_lex`（`:979`），**不存在 `THD::m_lex`**。

**★★ `LEX` 是 `struct`，继承 `Query_tables_list`，不是 `Query_arena`**（⚠️ 两处常见错误）：

```cpp
/* sql/sql_lex.h:3876 */
struct LEX : public Query_tables_list {
  friend bool lex_start(THD *thd);

  Query_expression *unit;  ///< Outer-most query expression
  /// @todo: query_block can be replaced with unit->first-select()
  Query_block *query_block;            ///< First query block.
  Query_block *all_query_blocks_list;  ///< List of all query blocks
 private:
  /* current Query_block in parsing */
  Query_block *m_current_query_block;
 public:
  bool is_explain() const { return explain_format != nullptr; }
```
```cpp
/* 代表性成员（sql_lex.h:3926） */
  Query_result *result;                 /* ★ 结果集落点（Query_result_send / INTO OUTFILE） */
  THD *thd;                             /* ★ LEX 反向持有 THD */
  Opt_hints_global *opt_hints_global;   /* Optimizer hints */
  Plugins_array plugins;                /* 本语句用到的插件 ref 列表 */
  Table_ref *insert_table;
  partition_info *part_info;
  LEX_USER *definer;
  List<set_var_base> var_list;
  /**
    List of placeholders ('?') for parameters of a prepared statement. Because
    we append to this list during parsing, it is naturally sorted by
    position of the '?' in the query string. ...
  */
  List<Item_param> param_list;          /* ★ PS 的 '?'，按出现顺序天然有序 */
  Sql_cmd *m_sql_cmd;                   /* ★ make_sql_cmd(Parse_tree_root*) 的产物 */
  bool safe_to_cache_query;             /* ★ query cache 已删，现用于分区表达式可用性判断 */
  sp_head *sphead;
```
> ⚠️ `sql_command` **不在 LEX 里**，它在基类 `Query_tables_list` 中；LEX 里也没有 `option_bits`，对应物是 `m_statement_options`。

**★★ `LEX::~LEX()` 什么都不 delete**（`sql/sql_lex.cc:403`，★ 这是"从不 delete"最直白的证据）：

```cpp
LEX::~LEX() {
  destroy_query_tables_list();                                  /* 基类：表/例程/权限链表 */
  plugin_unlock_list(nullptr, plugins.begin(), plugins.size());  /* 归还 plugin ref */
  unit = nullptr;  // Created in mem_root - no destructor        /* ★★ 关键自注 */
  query_block = nullptr;
  m_current_query_block = nullptr;
}
```
> ★★ `// Created in mem_root - no destructor` —— parse tree 在 mem_root 里，**没有析构函数可调，也不需要调**。

**★ `LEX::LEX()` 也完全不碰 mem_root**（`sql/sql_lex.cc:3585`）：

```cpp
LEX::LEX()
    : unit(nullptr), query_block(nullptr), all_query_blocks_list(nullptr),
      m_current_query_block(nullptr), result(nullptr), thd(nullptr),
      opt_hints_global(nullptr),
      plugins(PSI_NOT_INSTRUMENTED),
      ...
      is_lex_started(false),
      will_contextualize(true) {
  reset_query_tables_list(true);        /* ★ 只把基类表/例程链表初始化好 */
}
```
> ★ mem_root 绑定发生在 **`st_lex_local::operator new`**（见下），不在构造函数里。

**★★ `st_lex_local`：LEX 与 mem_root 的真正绑定点**（`sql/sql_lex.h:4856`）：

```cpp
struct st_lex_local : public LEX {
  static void *operator new(size_t size) noexcept {
    return (*THR_MALLOC)->Alloc(size);   /* ★ THR_MALLOC = 当前 arena 的 mem_root */
  }
  static void *operator new(size_t size, MEM_ROOT *mem_root,
                            const std::nothrow_t &arg
                            [[maybe_unused]] = std::nothrow) noexcept {
    return mem_root->Alloc(size);        /* ★ PS 用它：new (m_arena.mem_root) st_lex_local */
  }
  static void operator delete(void *ptr [[maybe_unused]],
                              size_t size [[maybe_unused]]) {
    TRASH(ptr, size);                    /* ★★ 只 TRASH（填毒），不 free */
  }
  static void operator delete(void *, MEM_ROOT *, const std::nothrow_t &)
      noexcept { /* Never called */ }
};
```

**★ `lex_start()`：复用而非新建**（`sql/sql_lex.cc:502`）：

```cpp
bool lex_start(THD *thd) {
  LEX *lex = thd->lex;        /* ★ 复用 thd->lex 指向的那个对象 */
  lex->thd = thd;
  lex->reset();               /* ★ 复位全部字段；旧 parse tree 指针直接丢弃 */
  thd->init_cost_model();
  const bool status = lex->new_top_level_query();   /* 建出 unit + query_block 骨架 */
  assert(lex->current_query_block() == nullptr);
  lex->m_current_query_block = lex->query_block;
  return status;
}
```
```cpp
/* sql/sql_lex.cc:775 */
bool LEX::new_top_level_query() {
  assert(unit == nullptr && query_block == nullptr);
  assert(result == nullptr);
  query_block = new_query(nullptr);     /* ★ new_query 内部走 new(thd->mem_root) */
  if (query_block == nullptr) return true;
  unit = query_block->master_query_expression();
  return false;
}
```
```cpp
/* sql/sql_lex.cc:531 —— lex_end 也不 free parse tree */
void lex_end(LEX *lex) {
  lex->release_plugins();
  sp_head::destroy(lex->sphead);   /* ★ 只销毁 SP 头 */
  lex->sphead = nullptr;
}
```

---

## 八、mem_root：为什么 Parse Tree 从不 delete

这是理解 MySQL 内存模型的关键。

**THD 的 mem_root 就是 `main_mem_root`**：

```cpp
// sql/sql_class.cc:626-629
THD::THD(bool enable_plugins)
    : Query_arena(&main_mem_root, STMT_REGULAR_EXECUTION), ...
      main_mem_root(key_memory_thd_main_mem_root,
                    global_system_variables.query_alloc_block_size),
```

（`Query_arena::mem_root` 指向 `&main_mem_root`，成员声明 `sql/sql_class.cc:706`）

**Parse Tree 节点全部分配在它上面** —— `NEW_PTN` 就是 `new(mem_root)`（详见 02）。

**每命令末一次性回收**（在 `dispatch_command` 尾部，不是 `do_command`）：

```cpp
// sql/sql_parse.cc:2507-2524
thd->work_part_info = nullptr;
constexpr size_t kPreallocSz = 40960;
if (thd->mem_root->allocated_size() < kPreallocSz)
  thd->mem_root->ClearForReuse();   // :2522  保留最后一块复用
else
  thd->mem_root->Clear();           // :2524  大查询整块释放
```

**结论**：

- PT 节点、Item、QE/QB 全是 mem_root 分配 → **永不逐个 `delete`**
- `ClearForReuse()` 把整块 arena 标记为空，整棵树一次性"蒸发"
- Item 另外有 `THD::item_list` 串起来，在 `cleanup_after_query()`（`sql_class.cc:1747`）里 `cleanup_items()` + `free_items()`（`:1807-1808`）
- 解析期还有独立上限：`sql_parse.cc:7135` 的 `set_max_capacity(parser_max_mem_size)`
- 连接销毁：`main_mem_root.Clear()`（`sql/sql_class.cc:1470`）

**★★ `MEM_ROOT` 的官方 Doxygen**（`include/my_alloc.h:52`，★ 讲清了为什么这么做）：

```cpp
/**
 * The MEM_ROOT is a simple arena, where allocations are carved out of
 * larger blocks. Using an arena over plain malloc gives you two main
 * advantages:
 *
 *   * Allocation is very cheap (only a few CPU cycles on the fast path).
 *   * You do not need to keep track of which memory you have allocated,
 *     as it will all be freed when the arena is destroyed.
 *
 * Thus, if you need to do many small allocations that all are to have
 * roughly the same lifetime, the MEM_ROOT is probably a good choice.
 * The flip side is that _no_ memory is freed until the arena is destroyed,
 * and no destructors are run (although you can run them manually yourself).
 *
 * This specific implementation works by allocating exponentially larger blocks
 * each time it needs more memory (generally increasing them by 50%), which
 * guarantees O(1) total calls to malloc and free. Only one free block is
 * ever used; as soon as there's an allocation that comes in that doesn't fit,
 * that block is stored away and never allocated from again. ...
 *
 * For C compatibility reasons, MEM_ROOT is a struct, even though it is
 * logically a class and follows the style guide for classes.
 */
struct MEM_ROOT {
 private:
  struct Block {
    Block *prev{nullptr}; /** Previous block; used for freeing. */
    char *end{nullptr};   /** One byte past the end; used for Contains(). */
  };
```
> ⚠️ **真名核查**：8.0.39 用的是私有嵌套 `MEM_ROOT::Block`，**不再使用 `USED_MEM`**（后者虽仍定义在 `include/my_sys.h:556` 但已不被 MEM_ROOT 用）。

**★ 快路径：纯 bump 分配**（`include/my_alloc.h:133`）：

```cpp
  void *Alloc(size_t length) MY_ATTRIBUTE((malloc)) {
    length = ALIGN_SIZE(length);
    // note: bump allocator
    if (static_cast<size_t>(m_current_free_end - m_current_free_start) >= length) {
      void *ret = m_current_free_start;
      m_current_free_start += length;   /* ★ 指针加法，无锁无 free list */
      return ret;
    }
    return AllocSlow(length);
  }
```

**★★ 1.5 倍指数增长：O(1) 次 malloc 的来源**（`mysys/my_alloc.cc:104`）：

```cpp
  const size_t bytes_to_alloc = length + ALIGN_SIZE(sizeof(Block));
  Block *new_block = static_cast<Block *>(
      my_malloc(m_psi_key, bytes_to_alloc, MYF(MY_WME | ME_FATALERROR)));
  if (new_block == nullptr) {
    if (m_error_handler) (m_error_handler)();
    return nullptr;
  }
  TRASH(new_block, bytes_to_alloc);
  new_block->end = pointer_cast<char *>(new_block) + bytes_to_alloc;
  m_allocated_size += length;

  // Make the default block size 50% larger next time.
  // This ensures O(1) total mallocs (assuming Clear() is not called).
  if (!MEM_ROOT_SINGLE_CHUNKS) {
    m_block_size += m_block_size / 2;      /* ★★ 1.5 倍 */
  }
  return new_block;
```

**★ `Clear()` vs `ClearForReuse()`**（`mysys/my_alloc.cc:171`，★ 差别只有一个 block）：

```cpp
void MEM_ROOT::Clear() {
  if (m_current_block == nullptr) return;
  Block *start = m_current_block;
  m_current_block = nullptr;
  m_block_size = m_orig_block_size;      /* ★ 块大小也复位 */
  m_current_free_start = &s_dummy_target;
  m_current_free_end = &s_dummy_target;
  m_allocated_size = 0;
  FreeBlocks(start);                     /* ★ 整链 free */
}

void MEM_ROOT::ClearForReuse() {
  if (MEM_ROOT_SINGLE_CHUNKS) { Clear(); return; }
  if (m_current_block == nullptr) return;
  // Keep the last block, which is usually the biggest one.
  m_current_free_start = pointer_cast<char *>(m_current_block) +
                         ALIGN_SIZE(sizeof(*m_current_block));
  Block *start = m_current_block->prev;  /* ★ 保留最后一个（通常最大的）block */
  m_current_block->prev = nullptr;
  m_allocated_size = m_current_free_end - m_current_free_start;
  TRASH(m_current_free_start, m_allocated_size);
  FreeBlocks(start);
}

void MEM_ROOT::FreeBlocks(Block *start) {
  // The MEM_ROOT might be allocated on itself, so make sure we don't
  // touch it after we've started freeing.
  for (Block *block = start; block != nullptr;) {
    Block *prev = block->prev;
    TRASH(block, std::distance(pointer_cast<char *>(block), block->end));
    my_free(block);
    block = prev;
  }
}
```
> ⚠️ **`init_alloc_root()` / `free_root()` / `alloc_root()` 在 8.0.39 的 `mysys/my_alloc.cc` 中已不存在**——替代品就是 `Alloc()` / `Clear()` / `ClearForReuse()`。

**★★ `Query_arena` = mem_root + Item 链表**（`sql/sql_class.h:343`）：

```cpp
class Query_arena {
 private:
  /*
    List of items created for this query. Every item adds itself to the list
    on creation (see Item::Item() for details)
  */
  Item *m_item_list;                     /* ★ 单向侵入式链表：Item::next_free */
 public:
  MEM_ROOT *mem_root;  // Pointer to current memroot   /* ★ "当前"二字：可被 swap */
  /*
    The states reflects three different life cycles for three
    different types of statements:
    Prepared statement: STMT_INITIALIZED -> STMT_PREPARED -> STMT_EXECUTED.
    Stored procedure:   STMT_INITIALIZED_FOR_SP -> STMT_EXECUTED.
    Other statements:   STMT_REGULAR_EXECUTION never changes.
  */
  enum enum_state {
    STMT_INITIALIZED = 0, STMT_INITIALIZED_FOR_SP = 1, STMT_PREPARED = 2,
    STMT_REGULAR_EXECUTION = 3, STMT_EXECUTED = 4, STMT_ERROR = -1
  };
```
```cpp
/* sql/sql_class.cc:2009 —— ★ free_items 只跑析构，不 free 内存 */
void Query_arena::add_item(Item *item) {
  item->next_free = m_item_list;      /* 头插法 */
  m_item_list = item;
}
void Query_arena::free_items() {
  Item *next;
  /* This works because items are allocated with (*THR_MALLOC)->Alloc() */
  for (; m_item_list; m_item_list = next) {
    next = m_item_list->next_free;
    m_item_list->delete_self();       /* ★ 只跑 Item 析构，不 free mem_root 内存 */
  }
}
void Query_arena::set_query_arena(const Query_arena &set) {
  mem_root = set.mem_root;            /* ★★ 8.0 没有 set_arena()，这就是"换 arena" */
  set_item_list(set.item_list());
  state = set.state;
}
void Query_arena::swap_query_arena(const Query_arena &source, Query_arena *backup) {
  backup->set_query_arena(*this);     /* 先存旧的 */
  set_query_arena(source);            /* 再切成 source（PS 的 m_arena） */
}
```
> ⚠️ `thd->set_arena()` **不存在**。

**★★ 其实是四级 mem_root，不是三级**（`sql/sql_class.h:4339` / `:2102`）：

```cpp
  /**
    This memory root is used for two purposes:
    - for conventional queries, to allocate structures stored in main_lex
    during parsing, and allocate runtime data (execution plan, etc.)
    during execution.
    - for prepared queries, only to allocate runtime data. The parsed
    tree itself is reused between executions and thus is stored elsewhere.
  */
  MEM_ROOT main_mem_root;
```
```cpp
  /*
    A permanent memory area of the statement. For conventional
    execution, the parsed tree and execution runtime reside in the same
    memory root. In this case stmt_arena points to THD. In case of
    a prepared statement or a stored procedure statement, thd->mem_root
    conventionally points to runtime memory, and thd->stmt_arena
    points to the memory of the PS/SP, where the parsed tree of the
    statement resides. ...
    Note: in the parser, stmt_arena == thd, even for PS/SP.
  */
  Query_arena *stmt_arena;
```
| 级别 | 载体 | 生命周期 |
|---|---|---|
| ① `THD::main_mem_root` | 常规语句的 parse tree + 运行时数据 | 每条命令末 `ClearForReuse`/`Clear` |
| ② `Prepared_statement::m_mem_root` | PS 的 parse tree | 跨语句，PS 销毁时才释放 |
| ③ `sp_lex_instr::m_lex_mem_root` | 存储程序指令级 | SP 指令生命周期 |
| ④ transaction mem_root | 事务内数据 | `get_transaction()->init_mem_root_defaults` |

```cpp
/* sql/sql_class.cc:1134 */
void THD::init_query_mem_roots() {
  mem_root->set_block_size(variables.query_alloc_block_size);
  get_transaction()->init_mem_root_defaults(variables.trans_alloc_block_size,
                                            variables.trans_prealloc_size);
}
```
> ★ `THD::end_statement()` 的注释明确写为什么不在那里 free（`sql_class.cc:2037`）：
> ```cpp
>   /*
>     Don't free mem_root, as mem_root is freed in the end of dispatch_command
>     (once for any command).
>   */
> ```

**★ `placement new` on MEM_ROOT**（`include/my_alloc.h:424`，★ 全 server `new (thd->mem_root) Xxx` 的落点）：

```cpp
/**
 * Allocate an object of the given type. Use like this:
 *   Foo *foo = new (mem_root) Foo();
 * Note that unlike regular operator new, this will not throw exceptions. ...
 */
inline void *operator new(size_t size, MEM_ROOT *mem_root,
                          const std::nothrow_t &arg
                          [[maybe_unused]] = std::nothrow) noexcept {
  return mem_root->Alloc(size);
}
inline void operator delete(void *, MEM_ROOT *, const std::nothrow_t &) noexcept {
  /* never called */
}
```
```cpp
/* sql/mem_root_allocator.h:135 —— STL 容器绑 mem_root 时 deallocate 是空实现 */
  void deallocate(pointer, size_type) {}      /* ★★ 再次印证"从不 delete" */
```

---

## 九、Prepared Statement 的差异

路径在 `sql/sql_prepare.cc`：

```
COM_STMT_PREPARE  → mysqld_stmt_prepare()               sql_prepare.cc:1541
                     → Prepared_statement::prepare()    sql_prepare.cc:2452
                          m_lex = new (m_arena.mem_root) st_lex_local;   // :2472
                          lex_start(thd);                                // :2512
                          parse_sql(...);                                // :2536  解析一次

COM_STMT_EXECUTE  → mysqld_stmt_execute()               sql_prepare.cc:1875
                     → Prepared_statement::execute_loop() :2988
                          → execute() :3383
                               mysql_execute_command(thd, true);         // :3662  不再解析
```

与 `COM_QUERY` 的对比：

| | COM_QUERY | Prepared Statement |
|---|-----------|-------------------|
| 解析次数 | 每条 1 次 | PREPARE 1 次；**EXECUTE 0 次**（除非 reprepare，见 `sql_prepare.cc:3203`） |
| LEX 归属 | 复用 `thd->lex` | 每个 PS 自带 `m_lex`，在 `m_arena.mem_root`（`sql_prepare.h:349`） |
| LEX 存活 | 语句结束即 `destroy()` | 存活到 `DEALLOCATE` / 连接断开（析构 `sql_prepare.cc:2357`） |
| 权限检查 | 每次执行 | **每次 EXECUTE 都重做**（走同一个 `mysql_execute_command`） |

**★★ PS 的 LEX 借壳 THD：`Statement_backup`**（`sql/sql_prepare.cc:376`，★ 这是 PS 最精妙的一处）：

```cpp
class Statement_backup {
  LEX *m_lex;
  LEX_CSTRING m_query_string;
  String m_rewritten_query;
  bool m_safe_to_display;
 public:
  /**
    Prepared the THD to execute the prepared statement.
    Save the current THD statement state.
  */
  void set_thd_to_ps(THD *thd, Prepared_statement *stmt) {
    mysql_mutex_lock(&thd->LOCK_thd_data);
    m_lex = thd->lex;
    thd->lex = stmt->m_lex;          /* ★★ THD 的 lex 被"借"给 PS */
    mysql_mutex_unlock(&thd->LOCK_thd_data);
    m_query_string = thd->query();
    thd->set_query(stmt->m_query_string);
    ...
  }
  void restore_thd(THD *thd, Prepared_statement *stmt) {
    mysql_mutex_lock(&thd->LOCK_thd_data);
    stmt->m_lex = thd->lex;
    thd->lex = m_lex;                /* ★ 换回原来的（普通语句的 main_lex） */
    mysql_mutex_unlock(&thd->LOCK_thd_data);
    ...
  }
```
> ★★ 所以：**执行期 `thd->lex` 指向 PS 的 `m_lex`，语句结束再换回**。这就是"同一个执行器（mysql_execute_command）能同时服务普通语句和 PS"的关键。

**★ `Prepared_statement` 类定义**（`sql/sql_prepare.h:342`）：

```cpp
class Prepared_statement final {
 public:
  /// Memory allocation arena, for permanent allocations to statement.
  Query_arena m_arena;               /* ★ PS 级 arena：mem_root 指向下面的 m_mem_root */
  Item_param **m_param_array{nullptr};
  Server_side_cursor *m_cursor{nullptr};
  const Protocol *m_active_protocol{nullptr};   /* ★ 用于检查执行期协议是否变了 */
  uint m_param_count{0};
  const ulong m_id;
  LEX *m_lex{nullptr};               /* ★ PS 独占的 LEX，跨语句长期存活 */
  LEX_CSTRING m_query_string{NULL_CSTR};
  PSI_prepared_stmt *m_prepared_stmt{nullptr};
 private:
  /// The memory root to allocate parsed tree elements (instances of Item,
  /// Query_block and other classes).
  MEM_ROOT m_mem_root;               /* ★★ 跨语句存活的证据 */
```
```cpp
/* sql_prepare.cc:2291 —— arena 直接绑定 m_mem_root，状态 STMT_INITIALIZED */
Prepared_statement::Prepared_statement(THD *thd_arg)
    : m_arena(&m_mem_root, Query_arena::STMT_INITIALIZED),
      m_id(++thd_arg->statement_id_counter),
      m_mem_root(key_memory_prepared_statement_main_mem_root,
                 thd_arg->variables.query_alloc_block_size) {
  *m_last_error = '\0';
}
```

**★★ `prepare()`：arena 的两次 swap**（`sql/sql_prepare.cc:2440`）：

```cpp
  m_lex = new (m_arena.mem_root) st_lex_local;    /* ★ PS 的 LEX 建在 PS 自己的 mem_root 上 */
  ...
  Statement_backup stmt_backup;
  stmt_backup.set_thd_to_ps(thd, this);           /* ① thd->lex 临时改指向 PS 的 m_lex */
  stmt_backup.save_rlb(thd);
  thd->swap_query_arena(m_arena, &arena_backup);  /* ②★★ thd->mem_root 切成 PS 的 m_mem_root */
  ...
  old_stmt_arena = thd->stmt_arena;
  thd->stmt_arena = &m_arena;                     /* ③ stmt_arena 也指向 PS */

  parser_state.m_lip.stmt_prepare_mode = true;    /* ★ '?' 变成 Item_param 而不是报错 */
  parser_state.m_lip.multi_statements = false;
  lex_start(thd);                                 /* ★ 同一个 lex_start()，但此刻 mem_root 已是 PS 的 */
  ...
  error = parse_sql(thd, &parser_state, nullptr); /* ★ 与 dispatch_sql_command 同一入口 */
  ...
  thd->swap_query_arena(arena_backup, &m_arena);  /* ④★ 还原：mem_root 回到 main_mem_root，
                                                       PS 的 mem_root 原封不动留下 */
```

**★★ `execute()` 的分工：只有 `stmt_arena` 切，mem_root 不切**（`sql_prepare.cc:3465`）：

```cpp
  /*
    Set up THD for execution of prepared statement:
    - THD::lex becomes statement's LEX
    - THD::m_query_string becomes statement's query string
    - THD::m_rewritten_query becomes statement's rewritten query
    - THD::stmt_arena becomes statement's arena (m_item_list, state)
    - THD::mem_root remains the current execution mem_root      /* ★★ 关键 */
  */
  Statement_backup stmt_backup;
  stmt_backup.set_thd_to_ps(thd, this);
  ...
  old_stmt_arena = thd->stmt_arena;
  thd->stmt_arena = &m_arena;        /* ★ 只有 stmt_arena 切到 PS；mem_root 仍是 THD 的 */
  ...
  m_lex->clear_execution();          /* ★★ PS 复用 LEX 的关键：不重新解析，只清空上次执行状态 */
```
> ★★ **PS 执行期新分配的 Item 落在 THD 的运行时 mem_root（会被回收），而 parse tree 留在 PS 的 m_mem_root（不回收）** —— 这就是"两级 mem_root"的分工。

**★ `execute_loop()`：reprepare 循环**（`sql_prepare.cc:2988`）：

```cpp
  if (m_arena.get_state() == Query_arena::STMT_ERROR) {   /* ★ 出错的 PS 不可再执行 */
    my_message(m_last_errno, m_last_error, MYF(0));
    return true;
  }
  ...
reexecute:
  if (!check_parameter_types()) {
    assert(!reprepared_for_types);
    reprepared_for_types = true;
    if (reprepare(thd)) return true;
    goto reexecute;      /* ★ 参数类型变了 → 重新 prepare 后再执行 */
  }
  ...
  if (sql_command_flags[m_lex->sql_command] & CF_REEXECUTION_FRAGILE) {
    reprepare_observer.reset_reprepare_observer();
    stmt_reprepare_observer = &reprepare_observer;
  }
  thd->push_reprepare_observer(stmt_reprepare_observer);
  error = execute(thd, expanded_query, open_cursor) || thd->is_error();
  thd->pop_reprepare_observer();
  ...
    if (reprepare_observer.is_invalidated()) {
      assert(thd->get_stmt_da()->mysql_errno() == ER_NEED_REPREPARE);
      if (reprepare_observer.can_retry()) {
        thd->clear_error();
        error = reprepare(thd);      /* ★ 表结构变了 → reprepare */
```

**★ `COM_STMT_EXECUTE` 的参数绑定**（`sql/sql_prepare.cc:526`）：

```cpp
static bool set_parameter_value(Item_param *param, const uchar **pos, ulong len) {
  switch (param->data_type_source()) {
    case MYSQL_TYPE_TINY:   { int8 value = (int8)**pos; param->set_int(...); break; }
    case MYSQL_TYPE_SHORT:  { int16 value = sint2korr(*pos); ... }
    case MYSQL_TYPE_LONG:   { int32 value = sint4korr(*pos); ... }
    case MYSQL_TYPE_LONGLONG: { longlong value = sint8korr(*pos); ... }
    case MYSQL_TYPE_FLOAT:  { float data = float4get(*pos); param->set_double(...); }
    case MYSQL_TYPE_DOUBLE: { double data = float8get(*pos); ... }
    ...
```
```cpp
/* sql_prepare.cc:1488 —— '?' 个数来自 LEX::param_list */
static bool init_param_array(THD *thd, Prepared_statement *stmt) {
  LEX *lex = stmt->m_lex;
  if ((stmt->m_param_count = lex->param_list.elements)) {
    if (stmt->m_param_count > static_cast<uint>(UINT_MAX16)) {
      my_error(ER_PS_MANY_PARAM, MYF(0));
      return true;
    }
    /* Use thd->mem_root as it points at statement mem_root */
    stmt->m_param_array = thd->mem_root->ArrayAlloc<Item_param *>(stmt->m_param_count);
                                 /* ★ 此刻 thd->mem_root == PS 的 m_mem_root */
```
> ⚠️ THD 里的 PS 容器成员名是 **`stmt_map`**（`sql_class.h:1268`）。

---

## 十、插件介入点：审计与查询改写

审计钩子定义在 `sql/sql_audit.cc`：`mysql_audit_notify` 各重载在 364 / **471**（parse 子类）/ **854**（command 子类）/ 883。

在 `dispatch_sql_command` 里有两个精确的插入点：

| 时机 | 行号 | 函数 |
|------|------|------|
| 解析**前** | `sql_parse.cc:5256` | `invoke_pre_parse_rewrite_plugins()` → `sql_query_rewrite.cc:52` |
| 解析**后**、执行前 | `sql_parse.cc:5272` | `invoke_post_parse_rewrite_plugins()` → `sql_query_rewrite.cc:89` |

两者能力不同：

```cpp
// sql/sql_query_rewrite.cc:63-75（pre-parse）
mysql_audit_notify(thd, AUDIT_EVENT(MYSQL_AUDIT_PARSE_PREPARSE), &flags, &rewritten_query);
if (flags & MYSQL_AUDIT_PARSE_REWRITE_PLUGIN_QUERY_REWRITTEN) {
  alloc_query(thd, rewritten_query.str, rewritten_query.length);   // :73  整条 SQL 被替换
  thd->m_parser_state->init(thd, thd->query().str, thd->query().length);
```

- **pre-parse 插件**只能"整串替换 SQL 文本"
- **post-parse 插件**能改写 Parse Tree —— 官方 Rewriter 插件（`plugin/rewriter/rewriter_plugin.cc:471`）就只注册了 `MYSQL_AUDIT_PARSE_POSTPARSE`

另有一个**与插件无关**的脱敏改写：`mysql_rewrite_query()`（`sql_parse.cc:5302` → `sql/sql_rewrite.cc:352`），只在语句含明文密码时才动，用于 general log / PFS。

**★★ 一个必须澄清的命名陷阱**（⚠️ 极易混淆）：

| 名字 | 真实身份 |
|---|---|
| `sql/sql_rewrite.h` / `.cc` | **不是**查询改写插件 —— 是把密码等敏感信息改写到**日志**的一组 `Rewriter_*` 类（`sql_rewrite.h:149` `Rewriter_user` …） |
| `plugin/rewriter/` | **才是**查询改写插件（官方 Rewriter），以 **audit plugin** 身份挂在 `MYSQL_AUDIT_PARSE_POSTPARSE` 上 |
| `sql/sql_query_rewrite.cc` | 是把上面两者**接进执行链**的胶水（`invoke_pre_parse_rewrite_plugins` / `invoke_post_parse_rewrite_plugins`） |

**★ `mysql_audit_notify` 是一族重载，不是一个函数**（`sql/sql_audit.h:84-332`，★ 11 个事件类各一个）：

| 事件 | 时机 | 能否中止 |
|---|---|---|
| `MYSQL_AUDIT_COMMAND_START` | `dispatch_command` 大 switch **之前** | ✅ **返回值非 0 → `goto done`** |
| `MYSQL_AUDIT_COMMAND_END` | `done:` 里 | ❌ 注释明说 "informational only" |
| `MYSQL_AUDIT_GENERAL_RESULT` | 发完结果后 | ❌ |
| `MYSQL_AUDIT_GENERAL_STATUS` | 每条语句（含多语句的子句） | ❌ |
| `MYSQL_AUDIT_PARSE_PREPARSE` | 解析前 | ✅ 可改写整串 SQL |
| `MYSQL_AUDIT_PARSE_POSTPARSE` | 解析后、执行前 | ✅ 可改写 Parse Tree |
| `MYSQL_AUDIT_CONNECTION_CHANGE_USER` | `COM_CHANGE_USER` | 参与 auth_rc |
| `MYSQL_AUDIT_CONNECTION_DISCONNECT` | `end_connection()` | ❌ |

```cpp
/* sql_parse.cc:2455 —— COMMAND_END 的官方说明 */
  /* command_end is informational only. The plugin cannot abort
     execution of the command at this point. */
  mysql_audit_notify(thd, AUDIT_EVENT(MYSQL_AUDIT_COMMAND_END), command,
                     cn.c_str());
```

**★ `COM_CHANGE_USER` 的审计与回滚**（`sql_parse.cc:1882`，★ 一个完整的失败回滚样例）：

```cpp
      Security_context save_security_ctx(*(thd->security_context()));
      MUTEX_LOCK(grd_secctx, &thd->LOCK_thd_security_ctx);

      auth_rc = acl_authenticate(thd, COM_CHANGE_USER);
      auth_rc |= mysql_audit_notify(
          thd, AUDIT_EVENT(MYSQL_AUDIT_CONNECTION_CHANGE_USER));
      if (auth_rc) {
        *thd->security_context() = save_security_ctx;   /* ★ 整体回滚 Security_context */
        thd->set_user_connect(save_user_connect);
        thd->reset_db(save_db);
        my_error(ER_ACCESS_DENIED_CHANGE_USER_ERROR, MYF(0), ...);
        thd->killed = THD::KILL_CONNECTION;             /* ★ CHANGE USER 失败直接断连接 */
        error = true;
      }
```
> ★ 注意 `LOCK_thd_security_ctx` 的作用：防止 `SHOW PROCESSLIST` 在更新过程中读到半截的 Security_context。

**★ 几个权限检查点的真身**（`dispatch_command` 各 case 里）：

```cpp
/* COM_FIELD_LIST —— 需要 SELECT_ACL */
      if (check_table_access(thd, SELECT_ACL, &table_list, true, UINT_MAX, false))
        break;
/* COM_REFRESH —— 需要 RELOAD_ACL */
      if (check_global_access(thd, RELOAD_ACL)) break;
/* COM_DEBUG —— 需要 SUPER_ACL */
      if (check_global_access(thd, SUPER_ACL)) break;
/* COM_PROCESS_INFO —— 有 PROCESS_ACL 才能看别人的 */
      global_access = (check_global_access(thd, PROCESS_ACL) == 0);
      mysqld_list_processes(thd, global_access ? NullS : thd->security_context()->priv_user().str, ...);
/* COM_SUBSCRIBE_GROUP_REPLICATION_STREAM —— ★ 动态全局授权（不是老 ACL 位） */
      if (!sctx->has_global_grant(STRING_WITH_LEN("GROUP_REPLICATION_STREAM")).first) {
        my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "");
        error = true;
        break;
      }
```
> ★★ 两种权限模型并存：老的 ACL 位（`check_global_access(thd, XXX_ACL)`）与 8.0 的动态授权（`has_global_grant(STRING_WITH_LEN("XXX"))`）。新功能一律用后者。

**★ `COM_STATISTICS` 是"裸写"一行文本，不走结果集协议**（`sql_parse.cc:2330`）：

```cpp
      length = snprintf(buff, buff_len - 1,
                        "Uptime: %lu  Threads: %d  Questions: %lu  "
                        "Slow queries: %llu  Opens: %llu  Flush tables: %lu  "
                        "Open tables: %u  Queries per second avg: %u.%03u", ...);
      // TODO: access of protocol_classic should be removed.
      // should be rewritten using store functions
      if (thd->get_protocol_classic()->write(pointer_cast<const uchar *>(buff), length))
        break;
      if (thd->get_protocol()->flush()) break;
      thd->get_stmt_da()->disable_status();      /* ★ 不回 OK/EOF */
```
> ★ 这就是 `mysqladmin status` 拿到的那一行文本。源码自己留了 TODO 说"应该改写成 store 函数"。

**★ `SERVER_MORE_RESULTS_EXISTS` 的设置与清除**：

```cpp
/* sql_parse.cc:2159 —— 多语句还有下文时置位（在 thd->server_status 里） */
        thd->send_statement_status();   /* ★ 第一条的 OK/EOF 带着这个标志发给客户端 */
```
```cpp
/* protocol_classic.cc:1106 —— 致命错误时必须清掉，否则客户端会继续等 */
    if (thd->is_fatal_error()) server_status &= ~SERVER_MORE_RESULTS_EXISTS;
```
> ★ 客户端（libmysql）靠 `SERVER_MORE_RESULTS_EXISTS`（0x0008）决定是否继续 `mysql_next_result()`。

---

## 十一、权限检查在哪个阶段

（`sql_acl.cc` 在 8.0 已拆到 `sql/auth/`）

| 函数 | 位置 | 层次 |
|------|------|------|
| `check_access` | `sql/auth/sql_authorization.cc:2119` | 全局 / 库级 |
| `check_table_access` | 同文件 2334 | 先 `check_access`（2380）再 `check_grant`（2388） |
| `check_grant` | 同文件 3762 | 表级 |
| `check_grant_column` | 同文件 3947 | 列级 |
| `check_global_access` | 同文件 5895 | 全局 |

**DML / SELECT 的检查时机**（在 `Sql_cmd_dml::prepare` 里，`sql/sql_select.cc:495`）：

```
:531  precheck(thd)                    → Sql_cmd_select::precheck :1110（粗粒度，如 FILE_ACL）
:542  open_tables_for_query()          开表
:568  prepare_inner()                  resolve 阶段：细粒度列权限在此
:736  check_privileges()（已 prepared 分支）→ :1151 → :1173 check_all_table_privileges
                                                    → :1168 check_column_privileges
```

DDL / 管理语句直接在 `mysql_execute_command` 的 case 里调（如 `sql_parse.cc:3668`、`3867`、`5075`）。

**★ 粗粒度 vs 细粒度的分工**（`sql/auth/sql_authorization.cc:2119`）：

```cpp
/**
  Check access rights to a database or the whole server.

  @param thd          Thread handler
  @param want_access  Bit mask of privileges the user needs to have to
                      execute the query successfully.
  ...
*/
bool check_access(THD *thd, ulong want_access, const char *db, ...)
```
> ★ `check_access()` 只看**全局位**（`sctx->master_access()`）与**库级**；表/列级必须走 `check_grant()`。

**★ `check_table_access()` 的两段式**（`sql_authorization.cc:2334`）：

```cpp
  // 先用粗粒度快速判定
  if (check_access(thd, want_access, db, ...))    /* :2380 */
    goto err;
  // 粗粒度不够，再逐表查 grant table
  if (check_grant(thd, want_access, tables, ...)) /* :2388 */
    goto err;
```
> ★ 这个"先粗后细"的顺序是性能关键：**有全局权限的用户不会去查 `mysql.tables_priv`**。

**★ 权限检查在链路上的三个落点**（★ 不同语句类型落点不同）：

| 语句类型 | 检查落点 |
|---|---|
| DML / SELECT | `Sql_cmd_dml::prepare` → `precheck()`（粗）→ `open_tables_for_query()` → `check_privileges()`（细） |
| DDL / 管理语句 | 直接在 `mysql_execute_command` 的对应 case 里 |
| 协议命令（`COM_*`） | 直接在 `dispatch_command` 的 case 里（见上文 `check_global_access` 那组） |

**★ 认证阶段就把 `Security_context` 定好了**（`sql/sql_class.cc:626`）：

```cpp
      m_main_security_ctx(this),                  /* ★ Security_context 是 THD 的成员（值语义） */
      m_security_ctx(&m_main_security_ctx),       /* ★ thd->security_context() 默认指向它 */
```
> ★ 所以**命令分发阶段只读 `Security_context`，不写**——唯一会写的是 `COM_CHANGE_USER`（且要拿 `LOCK_thd_security_ctx` 并整体回滚）。

**★ 两种权限模型并存**（★ 8.0 的关键演进）：

```cpp
/* 老模型：ACL 位 */
  ulong master_access = thd->security_context()->master_access();
  bool ok = sctx->check_access(SUPER_ACL);
/* 新模型：动态授权（字符串名） */
  bool ok = sctx->has_global_grant(STRING_WITH_LEN("SYSTEM_VARIABLES_ADMIN")).first;
```
> ★★ 8.0 新增的权限一律用动态授权（存 `mysql.global_grants`），老的 ACL 位保留兼容。**新功能不该再往 `master_access` 里加位**。

---

## 相关的系统变量/状态变量

### 系统变量（默认值已核实）

| 变量 | 默认值 | 作用域 | 说明 |
|------|--------|--------|------|
| `max_allowed_packet` | **64MB** | Global/Session | 单包 payload 上限，超出报 `ER_NET_PACKET_TOO_LARGE`（`sys_vars.cc:2918`） |
| `net_buffer_length` | **16384** | Global/Session | NET 缓冲区初始大小（`sys_vars.cc:3205`） |
| `net_read_timeout` | **30 秒** | Session | 读超时（`include/mysql_com.h:882` `NET_READ_TIMEOUT`） |
| `net_write_timeout` | **60 秒** | Session | 写超时（`NET_WRITE_TIMEOUT`，`mysql_com.h:883`） |
| `net_retry_count` | 10 | Global/Session | 可重试错误的重试次数 |
| `compress` | **off** | 客户端选项 | 客户端 `-C` 开启压缩连接 |
| `protocol_compression_algorithms` | **zlib,zstd,uncompressed** | Global/Session | 8.0.18+ 协议压缩算法列表（`compression.h:45`） |

> `max_allowed_packet` 若低于全局 `net_buffer_length` 会发 warning（`WARN_OPTION_BELOW_LIMIT`，`sys_vars.cc:2905`）。
> `net_read_timeout` / `net_write_timeout` 更新时经 `fix_net_read_timeout` / `fix_net_write_timeout` 实时下发到 VIO（`sys_vars.cc:3211-3251`）。

### 状态变量

| 变量 | 说明 |
|------|------|
| `Aborted_clients` | 客户端未正常关闭连接即断开 |
| `Aborted_connects` | 认证失败的连接尝试 |
| `Threads_connected` / `Threads_running` | 当前连接数 / 正在执行命令的线程数 |
| `Bytes_received` / `Bytes_sent` | 累计收 / 发字节数 |

**★ 状态变量的注册原文**（`SHOW_VAR status_vars[]`，`sql/mysqld.cc:9690`）：

```cpp
SHOW_VAR status_vars[] = {
    {"Aborted_clients", (char *)&aborted_threads, SHOW_LONG, SHOW_SCOPE_GLOBAL},
    {"Aborted_connects", (char *)&show_aborted_connects, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
    ...
    {"Bytes_received", (char *)offsetof(System_status_var, bytes_received),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},                       /* mysqld.cc:9717 */
    {"Bytes_sent", (char *)offsetof(System_status_var, bytes_sent),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},                       /* :9719 */
    ...
    {"Connections", (char *)&show_thread_id_count, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
    ...
    {"Max_used_connections", (char *)&Connection_handler_manager::max_used_connections,
     SHOW_LONG, SHOW_SCOPE_GLOBAL},                               /* :9848 */
    ...
    {"Threads_cached", (char *)&Per_thread_connection_handler::blocked_pthread_count,
     SHOW_LONG_NOFLUSH, SHOW_SCOPE_GLOBAL},                       /* :10027 */
    {"Threads_connected", (char *)&Connection_handler_manager::connection_count,
     SHOW_INT, SHOW_SCOPE_GLOBAL},                                /* :10030 */
    {"Threads_created", (char *)&show_num_thread_created, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
    {"Threads_running", (char *)&show_num_thread_running, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
```
> ★ 注意 `Threads_cached` 用的是 `SHOW_LONG_NOFLUSH`——**`FLUSH STATUS` 不清它**。
> ★ `Threads_running` 的 +1/-1 就在 `dispatch_command` 里（`inc_thread_running()` / `dec_thread_running()`）。

**★ 连接相关系统变量补齐（逐个核实）**：

| 变量 | 文件:行号 | 默认值 | 说明 |
|---|---|---|---|
| `max_connections` | `sql/sys_vars.cc:2979` | **151**（`MAX_CONNECTIONS_DEFAULT`，`sys_vars.h:109`） | `The number of simultaneous clients allowed` |
| `max_connect_errors` | `sys_vars.cc:2987` | **100** | 来自某 host 的中断连接超此数即 block |
| `connect_timeout` | `sys_vars.cc:2056` | **10**（`CONNECT_TIMEOUT`，`:184`） | 等 connect packet 的秒数 |
| `thread_cache_size` | `sys_vars.cc:5161` | 注册 `DEFAULT(0)`，运行期被 `mysqld.cc:5055` 覆盖为 `min(8 + max_connections/100, 100)` | `How many threads we should keep in a cache for reuse` |
| `thread_handling` | `sys_vars.cc:4049` | 下标 **0** = `one-thread-per-connection` | 三个取值：见下 |
| `local_infile` | `sys_vars.cc:2416` | **false**（8.0 起） | `Enable LOAD DATA LOCAL INFILE` |
| `skip_networking` | `sys_vars.cc:3980` | false | 禁止 TCP/IP |

```cpp
/* sql/sys_vars.cc:4047 —— 三个取值，默认取第一个 */
static const char *thread_handling_names[] = {
    "one-thread-per-connection", "no-threads", "loaded-dynamically", NullS};
```
```cpp
/* sql/sys_vars.cc:2418 —— ★ local_infile 默认 false */
static Sys_var_bool Sys_local_infile("local_infile",
                                     "Enable LOAD DATA LOCAL INFILE",
                                     GLOBAL_VAR(opt_local_infile),
                                     CMD_LINE(OPT_ARG), DEFAULT(false));
```
```cpp
/* sql/sys_vars.cc:213 / :184 —— 两个常量的真身 */
static constexpr int MYSQLD_NET_RETRY_COUNT = 10;
static constexpr uint CONNECT_TIMEOUT = 10;
```

**★ `dispatch_command` 的公共前缀与唯一出口**（★ 五节核心）：

```cpp
/* sql/sql_parse.cc:1670 —— 返回值语义 */
/**
  @retval
    0   ok
  @retval
    1   request of thread shutdown, i. e. if command is COM_QUIT
*/
bool dispatch_command(THD *thd, const COM_DATA *com_data,
                      enum enum_server_command command) {
```
```cpp
  bool error = false;                        /* ★ 唯一返回值载体，全程只写不读直到 2531 行 */
  ...
  thd->set_command(command);                 /* ★ SHOW PROCESSLIST 的 Command 列读这里 */
  thd->lex->sql_command = SQLCOM_END;        /* to avoid confusing VIEW detectors */
  if (thd->killed == THD::KILL_QUERY) thd->killed = THD::NOT_KILLED;
  thd->set_query_id(next_query_id());
  thd_manager->inc_thread_running();         /* ★ Threads_running +1 */

  if (!(server_command_flags[command] & CF_SKIP_QUESTIONS))
    thd->status_var.questions++;             /* ★ Questions（PING/STATISTICS/PREPARE 等不计） */

  thd->server_status &= ~SERVER_STATUS_CLEAR_SET;

  if (thd->get_protocol()->type() == Protocol::PROTOCOL_PLUGIN &&
      !(server_command_flags[command] & CF_ALLOW_PROTOCOL_PLUGIN)) {
    my_error(ER_PLUGGABLE_PROTOCOL_COMMAND_NOT_SUPPORTED, MYF(0));
    thd->killed = THD::KILL_CONNECTION;
    error = true;
    goto done;
  }

  /* ★ 密码过期：除这 7 个命令外一律拒绝 */
  if (unlikely(thd->security_context()->password_expired() &&
               command != COM_QUERY && command != COM_STMT_CLOSE &&
               command != COM_STMT_SEND_LONG_DATA && command != COM_PING &&
               command != COM_QUIT && command != COM_STMT_PREPARE &&
               command != COM_STMT_EXECUTE)) {
    my_error(ER_MUST_CHANGE_PASSWORD, MYF(0));
    goto done;
  }

  /* ★★ 审计插件可在命令开始前直接否决 */
  if (mysql_audit_notify(thd, AUDIT_EVENT(MYSQL_AUDIT_COMMAND_START), command,
                         Command_names::str_global(command).c_str())) {
    goto done;
  }

  switch (command) {
```
```cpp
/* sql/sql_parse.cc:2448 —— done: 收尾，唯一出口 */
done:
  thd->update_slow_query_status();
  if (thd->killed) thd->send_kill_message();
  thd->send_statement_status();              /* ★ 统一在这里发最后的状态包 */

  /* command_end is informational only. The plugin cannot abort
     execution of the command at this point. */
  mysql_audit_notify(thd, AUDIT_EVENT(MYSQL_AUDIT_COMMAND_END), command, cn.c_str());

  log_slow_statement(thd);
  thd->set_command(COM_SLEEP);               /* ★ 回到空闲态 */
  thd->lex->sql_command = SQLCOM_END;
  thd_manager->dec_thread_running();         /* ★ Threads_running -1，与 1770 行配对 */
  ...
  return error;                              /* ★★★ 唯一出口 */
```
> ★★ `dispatch_command` **只有一个 `return`**（`:2531`），所有异常路径走 `goto done`。
> ★ `server_command_flags[]` 是"协议命令级"标志（`sql_parse.cc:497`），与 `sql_command_flags[]`（SQL 语句级）并存，长度 `COM_END + 1 = 35`。

**★ `COM_QUERY` 的多语句循环**（`sql/sql_parse.cc:2070`，★ 三个退出条件）：

```cpp
      while (!thd->killed && (parser_state.m_lip.found_semicolon != nullptr) &&
             !thd->is_error()) {
                       /* ★★★ 被 kill / 没有分号 / 上一条出错 */
        const char *beginning_of_next_stmt = parser_state.m_lip.found_semicolon;
        thd->update_slow_query_status();
        thd->send_statement_status();   /* ★ 先发第一条的 OK/EOF（带 SERVER_MORE_RESULTS_EXISTS） */
        ...
        thd->status_var.questions++;    /* ★ 每条子语句各计一次 Questions */
        thd->set_query(beginning_of_next_stmt, length);
                                        /* ★ 把 thd->query() 切成下一条 */
        thd->set_query_id(next_query_id());
        parser_state.reset(beginning_of_next_stmt, length);
        dispatch_sql_command(thd, &parser_state);
      }
```
```cpp
/* sql/sql_parse.cc:2406 —— COM_SET_OPTION：运行时切换多语句开关 */
      switch (com_data->com_set_option.opt_command) {
        case (int)MYSQL_OPTION_MULTI_STATEMENTS_ON:
          thd->get_protocol_classic()->add_client_capability(CLIENT_MULTI_STATEMENTS);
          my_eof(thd);
          break;
        case (int)MYSQL_OPTION_MULTI_STATEMENTS_OFF:
          thd->get_protocol_classic()->remove_client_capability(CLIENT_MULTI_STATEMENTS);
          my_eof(thd);
          break;
        default:
          my_error(ER_UNKNOWN_COM_ERROR, MYF(0));   /* ★ 除 MULTI_STATEMENTS 外一律拒绝 */
      }
```

**★★ LOCAL INFILE：服务端主动发起**（`sql/rpl_replica.cc:2385`）：

```cpp
bool net_request_file(NET *net, const char *fname) {
  return net_write_command(net, 251, pointer_cast<const uchar *>(fname),
                           strlen(fname), pointer_cast<const uchar *>(""), 0);
                           /* ★★★ 251 = 0xFB —— 这就是 LOCAL INFILE 请求包 */
}
```
```cpp
/* sql/sql_load.cc:2126 —— 两道安全闸 */
  if (m_is_local_file) {
    if (!thd->get_protocol()->has_client_capability(CLIENT_LOCAL_FILES) ||
        !opt_local_infile) {                 /* ★ 闸一：客户端能力位；闸二：服务端开关 */
      my_error(ER_CLIENT_LOCAL_FILES_DISABLED, MYF(0));
      return true;
    }
```
```cpp
/* sql/sql_load.cc:466 —— 服务端"反向"向客户端要文件 */
  if (m_is_local_file) {
    (void)net_request_file(thd->get_protocol_classic()->get_net(),
                           m_exchange.file_name);
    file = -1;                               /* ★ 后面走 net 读而不是 fd */
  }
```
> ⚠️ **真名核查**：`handle_local_infile()` 是**客户端**函数（`libmysql/libmysql.cc:407`）；`mysql_set_local_infile_handler` 也是客户端 API（`include/mysql.h:513`）。**服务端侧只有 `net_request_file()`**。
> ★ 服务端读**本地**文件还要受 `secure_file_priv` 限制，且防 symlink 攻击（`sql_load.cc:498`：非符号链接 + 普通文件或 FIFO）。

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| `sql/net_serv.cc` | **不存在** → `sql-common/net_serv.cc`（client/server 共用） |
| `sql/protocol.cc` | **不存在** → `sql/protocol.h` + `sql/protocol_classic.h/.cc` |
| `NET` 是 class / 有成员函数 | **纯 POD struct**，27 字段，零成员函数 |
| `NET::compress_buf` / `net_skip_rest` / `net_wait_data` | **三个都不存在**（`compress_buf` → `NET_SERVER::compress_ctx`） |
| `my_net_write` 已被取代 | **三者并存分工不同**：`my_net_write`=裸 payload / `net_write_command`=命令包(自带 flush) / `net_write_packet`=落 socket |
| 压缩头 3B+3B+1B | **错** → 3B comp(0..2) + 1B seq(3) + 3B uncomp(4..6) |
| `MIN_COMPRESS_LENGTH` 在 `mysql_com.h` | 在 **`include/my_sys.h:173`** = 50 |
| 压缩只有 zlib | 8.0 有 **zstd**：`MYSQL_UNCOMPRESSED/ZLIB/ZSTD/INVALID`，默认 level zlib=6 / zstd=3 |
| `Protocol_binary` 直接继承 `Protocol_classic` | **错** → `binary → text → classic → Protocol` |
| `Protocol` 有 `send_result_metadata` | **没有** → 拆成 start_result_metadata / start_row / send_field_metadata / end_row / end_result_metadata 五步 |
| `Query_result_send` 在 `sql_class.h` | 在 **`sql/query_result.h:185`** |
| SHOW PROCESSLIST 有 "Sending data" | **8.0.39 全库 grep 0 命中** → 真相是 **`"Sending to client"`**（`sql_show.cc:2791`） |
| `thd->set_sent_row_count()` 用于发结果集 | 发结果集用 **`inc_sent_row_count(1)`**（`query_result.cc:106`） |
| `Per_thread_connection_handler` 在 `connection_handler_per_thread.h` | **不存在该文件** → 类在 `connection_handler_impl.h:42` |
| 命令码约 40 个 | **35 个枚举值**：`COM_SLEEP=0 … COM_SUBSCRIBE_GROUP_REPLICATION_STREAM=33`，`COM_END=34` |
| 有 `COM_SHUTDOWN` | **不存在** → 位置 8 是 `COM_DEPRECATED_1`，注释 `Deprecated, used to be COM_SHUTDOWN`；字符串仍叫 "Shutdown" |
| `handle_local_infile()` 是服务端函数 | **是客户端函数** → 服务端只有 `net_request_file()`（发 0xFB） |
| thread cache 缓存 THD | **缓存 OS 线程不是 THD** → 官方注释 `// note: create THD for each connection!` + `delete thd` |
| `dispatch_command()` 多处 return | **只有一个出口** `return error`（`:2531`），异常走 `goto done` |
| `local_infile` 默认 ON | **`DEFAULT(false)`**（8.0 起） |
| `thread_cache_size` 默认 0 | 注册 `DEFAULT(0)`，但 `mysqld.cc:5055` 覆盖为 `min(8 + max_connections/100, 100)` |
| `thread_handling` 两个取值 | **三个**：`one-thread-per-connection` / `no-threads` / `loaded-dynamically` |
| `sql/sql_rewrite.{h,cc}` 是查询改写插件 | **不是** → 那是日志脱敏（`Rewriter_user` 等）；真正的改写插件是 `plugin/rewriter/`，以 **audit plugin** 身份挂在 `MYSQL_AUDIT_PARSE_POSTPARSE` |
| `mysql_parse()` 还存在 | **不存在** → 改名 `dispatch_sql_command()`（`sql_parse.cc:5244`） |
| `class LEX` 继承 `Query_arena` | **`struct LEX : public Query_tables_list`**（不是 class，不继承 Query_arena） |
| `LEX::LEX()` 绑定 mem_root | **完全不接触 mem_root** → 绑定在 `st_lex_local::operator new` |
| `LEX::~LEX()` delete parse tree | **什么都不 delete** → 自注 `// Created in mem_root - no destructor` |
| `init_alloc_root()` / `free_root()` / `alloc_root()` | 在 8.0.39 的 `mysys/my_alloc.cc` 中**全部不存在** |
| MEM_ROOT 用 `USED_MEM` 块 | 改用私有 `MEM_ROOT::Block`（`USED_MEM` 已不被使用） |
| `THD::free_items()` 释放内存 | 只跑 `Item::delete_self()` 析构，**不释放 mem_root 内存** |
| `thd->set_arena()` | **不存在** → `Query_arena::set_query_arena()` / `swap_query_arena()` |
| `Statement_area` / `Traditional_statement_area` / `Execution_state` | **三个都不存在** |
| `THD::m_lex` | **不存在** → `LEX *lex` + `std::unique_ptr<LEX> main_lex` |
| PS 的 LEX 挂在 THD 上 | PS 有**自己的** `m_lex`，执行时由 `Statement_backup::set_thd_to_ps()` 临时替换 `thd->lex` |
| mem_root 只有 THD 一个 | 至少**四级**：main_mem_root / PS m_mem_root / sp_lex_instr / transaction |

---

## 核心调用栈

```
connection_handler_per_thread.cc:246  handle_connection()
 └─ :304  while (thd_connection_alive(thd)) { do_command(thd); }
     │
     ├─ sql_parse.cc:1308  do_command()
     │    ├─ :1381  get_command()  → protocol_classic.cc:2991
     │    │    └─ read_packet() :1410 → my_net_read net_serv.cc:2241
     │    │         └─ net_read_packet → vio_read →（viosocket recv / viossl SSL_read）
     │    └─ :1440  dispatch_command()
     │
     └─ sql_parse.cc:1688  dispatch_command()
          ├─ :1809  audit: COMMAND_START
          └─ :2011  case COM_QUERY
               ├─ :2016  alloc_query()
               └─ :2055  dispatch_sql_command()               ← 原 mysql_parse
                    ├─ :5248  mysql_reset_thd_for_next_command()
                    ├─ :5253  lex_start()          → sql_lex.cc:511
                    ├─ :5256  pre-parse 改写插件    → sql_query_rewrite.cc:52
                    ├─ :5271  parse_sql()          → sql_parse.cc:7067
                    │    └─ :7142  THD::sql_parser() → sql_class.cc:3048
                    │         ├─ :3067  MYSQLparse()        → ① Parse Tree
                    │         └─ :3077  lex->make_sql_cmd() → ② 逻辑查询树（见 03）
                    ├─ :5272  post-parse 改写插件
                    ├─ :5302  mysql_rewrite_query()（脱敏）
                    └─ :5374  mysql_execute_command()  → sql_parse.cc:2946
                         └─ :4724  lex->m_sql_cmd->execute()  → sql_select.cc:675

【收尾】sql_parse.cc:5419-5421  lex->destroy() / end_statement() / cleanup_after_query()
       sql_parse.cc:2522       mem_root->ClearForReuse()   ← Parse Tree 消失
```

---


## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Connectors and APIs*（客户端/服务端协议）
- *MySQL 8.0 Reference Manual → Connection Management and Thread Handling*（one-thread-per-connection 模型）

**书籍**
- 《高性能 MySQL》第 1 章（MySQL 逻辑架构与连接/线程管理）

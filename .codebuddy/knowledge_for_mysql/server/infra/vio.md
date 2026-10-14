# VIO 通信抽象层深度解析

> 基于 MySQL 8.0.39 源码，涵盖 Vio 结构体、vtable 多态、各类连接实现（socket/SSL/命名管道/共享内存）、与 NET 及认证插件的调用关系。
>
> **边界**：本篇讲 VIO 通信抽象层本身；认证插件的 VIO 视角见 [`mfa.md`](../auth/mfa.md) 与认证主题；上层网络包读写（`NET`）不在本篇展开。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

VIO 是 **Virtual I/O（虚拟 IO）** 的缩写，是 MySQL 的**通信抽象层**。它把底层各种传输通道——TCP socket、Unix domain socket、SSL、Windows 命名管道、Windows 共享内存——统一成一组 `read`/`write`/`shutdown` 等函数指针，让上层（协议层 `NET`、认证插件）完全不关心底层走的是哪种连接。

### 用途

MySQL 同一套协议逻辑要跑在多种传输上（`-h127.0.0.1` 走 TCP、`-hlocalhost` 走 socket、`--ssl` 走 SSL、Windows 上还能走命名管道和共享内存）。VIO 就是这层"多态"——上层只调用 `vio_read(vio, buf, size)`，具体行为由 `vio` 对象背后的 vtable 决定。

### 版本演进

| 版本 | 变化 |
|------|------|
| 早期 | VIO 只有 socket 一种 |
| 5.x | 加入 SSL、Windows 命名管道/共享内存 |
| 8.0 | 引入 `USE_PPOLL_IN_VIO`/`HAVE_KQUEUE`，`vio_io_wait` 支持 poll/kqueue 事件驱动；`MYSQL_SOCKET` 结合 PSI 插桩 |

---

## 理论基础

### 设计思想与权衡

**核心问题是"C 接口如何做多态"**。MySQL 的 VIO 是纯 C 接口（`violite.h`），不能像 C++ 那样用虚函数，于是采用了**手写 vtable（函数指针表）**：

```384:404:include/violite.h
  void (*viodelete)(MYSQL_VIO) = {nullptr};
  int (*vioerrno)(MYSQL_VIO) = {nullptr};
  size_t (*read)(MYSQL_VIO, uchar *, size_t) = {nullptr};
  size_t (*write)(MYSQL_VIO, const uchar *, size_t) = {nullptr};
  int (*timeout)(MYSQL_VIO, uint, bool) = {nullptr};
  int (*viokeepalive)(MYSQL_VIO, bool) = {nullptr};
  int (*fastsend)(MYSQL_VIO) = {nullptr};
  bool (*peer_addr)(MYSQL_VIO, char *, uint16 *, size_t) = {nullptr};
  void (*in_addr)(MYSQL_VIO, struct sockaddr_storage *) = {nullptr};
  bool (*should_retry)(MYSQL_VIO) = {nullptr};
  bool (*was_timeout)(MYSQL_VIO) = {nullptr};
  int (*vioshutdown)(MYSQL_VIO) = {nullptr};
  bool (*is_connected)(MYSQL_VIO) = {nullptr};
  bool (*has_data)(MYSQL_VIO) = {nullptr};
  int (*io_wait)(MYSQL_VIO, enum enum_vio_io_event, int) = {nullptr};
  bool (*connect)(MYSQL_VIO, struct sockaddr *, socklen_t, int) = {nullptr};
```

这是典型的 C 风格多态：一个 `Vio` 对象 = 通用字段（socket、type、timeout、地址、读缓冲）+ 一组函数指针。创建时按连接类型填充不同的实现函数。

**权衡与代价**：
- 放弃了 C++ 虚函数的类型安全与编译期检查，换来的是 C ABI 稳定（`Vio` 可被插件、Connector/C 等纯 C 代码共享）。
- 手写 vtable 意味着"新增一种 VIO 类型"必须记得填充全部十几个函数指针，遗漏一个就是空指针调用——这在 `enum_vio_type` 注释里被显式警告（新增类型要同步更新 `LAST_VIO_TYPE` 和 `get_vio_type_name`）。

### 理论溯源

VIO 的 vtable 手写多态，与 COM（Component Object Model）的 IUnknown/vtable、以及 Linux 内核 VFS 的 `struct file_operations` 是同一套思想：**面向对象接口 + 函数指针表**，用结构体把"数据"和"操作"绑定，运行时按对象的实际类型分发。这套手法在 C 生态里被广泛使用（glibc 的 `FILE`、内核的 `file_operations`）。

### 算法与数据结构

⚠️ **勘误**：
- **不存在 `vio/vio.h` / `vio/viopriv.h`**。真实头文件是 **`include/violite.h`**（`struct Vio` 就在这里，`:318`）+ `vio/vio_priv.h`（只有内部声明）。
- **不存在 `vio_ops` / `VioSocketOps` vtable 结构体**。vtable 是 `struct Vio` 里的 **17 个裸函数指针成员**。
- **不存在 `Vio::sd`**（被 `MYSQL_SOCKET mysql_socket` 取代）、**不存在 `HANDLE handle`**、**不存在 `write_buffer`/`write_pos`/`write_end`**（只有**读**缓冲）、**不存在 `func_name`**、**不存在 `timeout`**（是 `read_timeout`/`write_timeout`）。

**`struct Vio` 前半**（`include/violite.h:318`）：

```cpp
/*
 This structure is for every connection on both sides.
 Note that it has a non-default move assignment operator, so if adding more
 members, you'll need to update operator=.
*/
struct Vio {
  MYSQL_SOCKET mysql_socket;          /* Instrumented socket */   // ★ 取代旧版裸 my_socket sd
  bool localhost = {false};           /* Are we from localhost? */
  enum_vio_type type = {NO_VIO_TYPE}; /* Type of connection */

  int read_timeout = {-1};  /* Timeout value (ms) for read ops. */   // ★ 毫秒，-1 = 无限
  int write_timeout = {-1}; /* Timeout value (ms) for write ops. */
  int retry_count = {1};    /* Retry count */
  bool inactive = {false};  /* Connection has been shutdown */

  struct sockaddr_storage local;  /* Local internet address */
  struct sockaddr_storage remote; /* Remote internet address */
  size_t addrLen = {0};           /* Length of remote address */
  char *read_buffer = {nullptr};  /* buffer for vio_read_buff */
  char *read_pos = {nullptr};     /* start of unfetched data in the read buffer */
  char *read_end = {nullptr};     /* end of unfetched data */

#ifdef USE_PPOLL_IN_VIO
  /**
    Thread PID which is to be sent SIGALRM to terminate ppoll
    wait when shutting down vio. It is made an std::optional so
    that server code has the ability to set this to an illegal value
    and thereby ensure that it is set before shutting down vio. In the server
    the THD and thereby the Vio can switch between OS threads, so it does not
    make sense to assign the thread id when creating the THD/Vio.
    ...
  */
  std::optional<my_thread_t> thread_id = 0;      // ★ 取消机制核心
  sigset_t signal_mask;  // Signal mask
  /*
    Flag to indicate whether we are in poll or shutdown.
    A true value of flag indicates either the socket
    has called  shutdown or it is sleeping on a poll call.
    ... If the other thread is waiting on poll
    sleep, it wakes up the thread by sending a signal via
    pthread_kill. Also it ensures that no other thread enters in
    to a poll call if it's socket has undergone shutdown.
  */
  std::atomic_flag poll_shutdown_flag = ATOMIC_FLAG_INIT;
#elif defined HAVE_KQUEUE
  int kq_fd = {-1};
  std::atomic_flag kevent_wakeup_flag = ATOMIC_FLAG_INIT;
#endif

#ifdef HAVE_SETNS
  /** Socket network namespace. */
  char network_namespace[256];
#endif
```

**vtable 段**（`:375`）——★ 注意 `read` 成员就叫 `read`，不是 `vio_read`：

```cpp
  /*
     VIO vtable interface to be implemented by VIO's like SSL, Socket,
     Named Pipe, etc.
  */
  /*
     viodelete is responsible for cleaning up the VIO object by freeing
     internal buffers, closing descriptors, handles.
  */
  void (*viodelete)(MYSQL_VIO) = {nullptr};
  int (*vioerrno)(MYSQL_VIO) = {nullptr};
  size_t (*read)(MYSQL_VIO, uchar *, size_t) = {nullptr};         // ★ 名字就叫 read
  size_t (*write)(MYSQL_VIO, const uchar *, size_t) = {nullptr};
  int (*timeout)(MYSQL_VIO, uint, bool) = {nullptr};
  int (*viokeepalive)(MYSQL_VIO, bool) = {nullptr};
  int (*fastsend)(MYSQL_VIO) = {nullptr};
  bool (*peer_addr)(MYSQL_VIO, char *, uint16 *, size_t) = {nullptr};
  void (*in_addr)(MYSQL_VIO, struct sockaddr_storage *) = {nullptr};  // ★ 死槽：从不赋值
  bool (*should_retry)(MYSQL_VIO) = {nullptr};
  bool (*was_timeout)(MYSQL_VIO) = {nullptr};
  /*
     vioshutdown is resposnible to shutdown/close the channel, so that no
     further communications can take place, however any related buffers,
     descriptors, handles can remain valid after a shutdown.
  */
  int (*vioshutdown)(MYSQL_VIO) = {nullptr};
  bool (*is_connected)(MYSQL_VIO) = {nullptr};
  bool (*has_data)(MYSQL_VIO) = {nullptr};
  int (*io_wait)(MYSQL_VIO, enum enum_vio_io_event, int) = {nullptr};
  bool (*connect)(MYSQL_VIO, struct sockaddr *, socklen_t, int) = {nullptr};  // ★ 同样死槽
#ifdef _WIN32
  OVERLAPPED overlapped = {0};
  HANDLE hPipe{nullptr};                       // ★ 旧版 "HANDLE handle" 在这里
#endif
  void *ssl_arg = {nullptr};                   // ★ SSL* 存这里
  struct PSI_socket_locker *m_psi_read_locker = {nullptr};
  PSI_socket_locker_state m_psi_read_state;
  struct PSI_socket_locker *m_psi_write_locker = {nullptr};
  PSI_socket_locker_state m_psi_write_state;
#if defined(_WIN32)
  HANDLE handle_file_map = {nullptr};          // 以下 7 项是 Windows 共享内存专用
  char *handle_map = {nullptr};
  HANDLE event_server_wrote = {nullptr};
  HANDLE event_server_read = {nullptr};
  HANDLE event_client_wrote = {nullptr};
  HANDLE event_client_read = {nullptr};
  HANDLE event_conn_closed = {nullptr};
  size_t shared_memory_remain = {0};
  char *shared_memory_pos = {nullptr};
#endif
  bool (*is_blocking)(Vio *vio) = {nullptr};   // ★ 唯一三个用 Vio* 而非 MYSQL_VIO 的槽
  int (*set_blocking)(Vio *vio, bool val) = {nullptr};
  int (*set_blocking_flag)(Vio *vio, bool val) = {nullptr};
  /* Indicates whether socket or SSL based communication is blocking or not. */
  bool is_blocking_flag = {true};

 private:
  friend Vio *internal_vio_create(uint flags);
  friend void internal_vio_delete(Vio *vio);
  friend bool vio_reset(Vio *vio, enum_vio_type type, my_socket sd, void *ssl,
                        uint flags);

  explicit Vio(uint flags);      // ★ 构造/析构私有：只能 internal_vio_create / mysql_socket_vio_new
  ~Vio();

 public:
  Vio(const Vio &) = delete;                 // ★ 禁止拷贝，只支持移动
  Vio &operator=(const Vio &) = delete;
  Vio &operator=(Vio &&vio);
};

#define SSL_handle SSL *
```

> ★★ **两个死槽**：`in_addr` 与 `connect` 在 `vio_init()` 的**所有分支里都没有赋值**，恒为 `nullptr`；全源码唯一触碰它们的地方是 `Vio::operator=` 的搬运（`:166`、`:180`）。所以 **不存在 `vio_in_addr()` / `vio_local_addr()`**，`Vio::local` 成员保留但无人填充。

**`enum enum_vio_type`**（`:79`）——8.0 已显式指定底层类型 `int`：

```cpp
enum enum_vio_type : int {
  NO_VIO_TYPE = 0,           /** Type of the connection is unknown. */
  VIO_TYPE_TCPIP = 1,        /** Used in case of TCP/IP connections. */
  VIO_TYPE_SOCKET = 2,       /** Used for Unix Domain socket connections. Unix only. */
  VIO_TYPE_NAMEDPIPE = 3,    /** Used for named pipe connections. Windows only. */
  VIO_TYPE_SSL = 4,          /** Used in case of SSL connections. */
  VIO_TYPE_SHARED_MEMORY = 5,/** Used for shared memory connections. Windows only. */
  VIO_TYPE_LOCAL = 6,        /** Used internally by the prepared statements */
  VIO_TYPE_PLUGIN = 7,       /** Implicitly used by plugins that doesn't support any other VIO_TYPE. */

  FIRST_VIO_TYPE = VIO_TYPE_TCPIP,
  /*
    If a new type is added, please update LAST_VIO_TYPE. In addition, please
    change get_vio_type_name() in vio/vio.c to return correct name for it.
  */
  LAST_VIO_TYPE = VIO_TYPE_PLUGIN
};
```

```cpp
// :121
enum enum_vio_io_event { VIO_IO_EVENT_READ, VIO_IO_EVENT_WRITE, VIO_IO_EVENT_CONNECT };

#define VIO_SOCKET_ERROR     ((size_t)-1)     // ★ 注意是 size_t：vio_read 返回 -1 实为 SIZE_MAX
#define VIO_SOCKET_WANT_READ ((size_t)-2)     // ★ SSL 非阻塞专用
#define VIO_SOCKET_WANT_WRITE ((size_t)-3)

#define VIO_LOCALHOST 1            /* a localhost connection */
#define VIO_BUFFERED_READ 2        /* use buffered read */
#define VIO_READ_BUFFER_SIZE 16384 /* size of read buffer */   // ★ 唯一的缓冲常量
```

**`MYSQL_SOCKET` 插桩 socket**（`include/mysql/components/services/bits/mysql_socket_bits.h:35`）：

```cpp
/**
  An instrumented socket.
  @c MYSQL_SOCKET is a replacement for @c my_socket.
*/
struct MYSQL_SOCKET {
  /** The real socket descriptor. */
  my_socket fd;                      // Unix: int；Windows: SOCKET

  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c MYSQL_SOCKET interface.
  */
  struct PSI_socket *m_psi;          // ★ 无条件存在以保证 ABI 兼容
};

#define MYSQL_INVALID_SOCKET mysql_socket_invalid()   // ★ 是函数调用，不是常量
```

```cpp
static inline my_socket mysql_socket_getfd(MYSQL_SOCKET mysql_socket) {
  return mysql_socket.fd;            // ★ 等价于旧版的 vio->sd，纯取字段零开销
}
static inline void mysql_socket_setfd(MYSQL_SOCKET *mysql_socket, my_socket fd) {
  if (likely(mysql_socket != nullptr)) {
    mysql_socket->fd = fd;           // ★ 只改 fd，不动 m_psi（所以 vio_reset 要单独拷 m_psi）
  }
}
```

> ⚠️ **不存在 `MySocket`（驼峰类）**。`mysql_socket` 只是 `struct Vio` 的**成员名**。
>
> ★ 两个小函数（`:545`）：`enum enum_vio_type vio_type(const Vio *vio) { return vio->type; }`、`my_socket vio_fd(Vio *vio) { return mysql_socket_getfd(vio->mysql_socket); }` —— 后者是 `vio->sd` 的官方替代。

`Vio` 结构体（`violite.h:318`）分两部分：

```318:334:include/violite.h
struct Vio {
  MYSQL_SOCKET mysql_socket;          /* Instrumented socket */
  bool localhost = {false};           /* Are we from localhost? */
  enum_vio_type type = {NO_VIO_TYPE}; /* Type of connection */

  int read_timeout = {-1};  /* Timeout value (ms) for read ops. */
  int write_timeout = {-1}; /* Timeout value (ms) for write ops. */
  int retry_count = {1};    /* Retry count */
  bool inactive = {false};  /* Connection has been shutdown */

  struct sockaddr_storage local;  /* Local internet address */
  struct sockaddr_storage remote; /* Remote internet address */
  size_t addrLen = {0};           /* Length of remote address */
  char *read_buffer = {nullptr};  /* buffer for vio_read_buff */
  char *read_pos = {nullptr};     /* start of unfetched data */
  char *read_end = {nullptr};     /* end of unfetched data */
```

- `type` 是判别标志，决定 vtable 该填哪套实现
- `read_buffer`/`read_pos`/`read_end` 支持缓冲读（`VIO_BUFFERED_READ`），避免频繁 syscall
- `local`/`remote` 存两端地址，供 `peer_addr`/日志用

七种连接类型（`violite.h:79-119`）：

```79:119:include/violite.h
enum enum_vio_type : int {
  NO_VIO_TYPE = 0,        // 未知
  VIO_TYPE_TCPIP = 1,     // TCP/IP
  VIO_TYPE_SOCKET = 2,    // Unix domain socket
  VIO_TYPE_NAMEDPIPE = 3, // 命名管道（仅 Windows）
  VIO_TYPE_SSL = 4,       // SSL
  VIO_TYPE_SHARED_MEMORY = 5, // 共享内存（仅 Windows）
  VIO_TYPE_LOCAL = 6,     // 预处理语句内部使用
  VIO_TYPE_PLUGIN = 7,    // 不支持其他类型的插件隐式使用
  ...
};
```

---

## 核心实现

### 主链路

```
客户端/服务器建立连接
  → vio_new(sd, type, flags)         分配 Vio 对象
    → internal_vio_create(flags)      填默认字段 + 按 type 填 vtable
  → NET 层（net.cc/net_serv.cc）通过 vio_read/vio_write 收发协议包
  → 认证插件通过 MYSQL_PLUGIN_VIO 读写握手数据（封装 Vio）
  → 连接关闭 vio_delete / vio_shutdown
```

### vtable 填充（关键环节）

**8.0 的 vtable 没有"实例"，只有"宏改写"**（`include/violite.h:284`）：

```cpp
#if !defined(DONT_MAP_VIO)
#define vio_delete(vio) (vio)->viodelete(vio)
#define vio_errno(vio) (vio)->vioerrno(vio)
#define vio_read(vio, buf, size) ((vio)->read)(vio, buf, size)
#define vio_write(vio, buf, size) ((vio)->write)(vio, buf, size)
#define vio_fastsend(vio) (vio)->fastsend(vio)
#define vio_keepalive(vio, set_keep_alive) \
  (vio)->viokeepalive(vio, set_keep_alive)
#define vio_should_retry(vio) (vio)->should_retry(vio)
#define vio_was_timeout(vio) (vio)->was_timeout(vio)
#define vio_shutdown(vio) ((vio)->vioshutdown)(vio)
#define vio_peer_addr(vio, buf, prt, buflen) \
  (vio)->peer_addr(vio, buf, prt, buflen)
#define vio_io_wait(vio, event, timeout) (vio)->io_wait(vio, event, timeout)
#define vio_is_connected(vio) (vio)->is_connected(vio)
#define vio_is_blocking(vio) (vio)->is_blocking(vio)
#define vio_set_blocking(vio, val) (vio)->set_blocking(vio, val)
#define vio_set_blocking_flag(vio, val) (vio)->set_blocking_flag(vio, val)
#endif /* !defined(DONT_MAP_VIO) */
```

> ★★ 而 `vio/vio_priv.h:32` 里 `#define DONT_MAP_VIO` —— 所以 **vio 包内（`viosocket.cc` / `viossl.cc`）看到的是真实函数，包外看到的是宏**。这正是 `vio_socket_timeout()` 要在 `vio_priv.h` 里单独声明的原因（`timeout` 槽没有对应宏）。

**`vio_init()` —— vtable 的唯一装配点，4 个分支**（`vio/vio.cc:211`，⚠️ 是 **`static`**，不是公开 API）：

```cpp
static bool vio_init(Vio *vio, enum enum_vio_type type, my_socket sd, uint flags) {
  mysql_socket_setfd(&vio->mysql_socket, sd);
  vio->localhost = flags & VIO_LOCALHOST;
  vio->type = type;

  switch (type) {
#ifdef _WIN32
    case VIO_TYPE_NAMEDPIPE:
      vio->viodelete = vio_delete;
      vio->vioerrno = vio_errno;
      vio->read = vio_read_pipe;
      vio->write = vio_write_pipe;
      vio->fastsend = vio_fastsend;
      vio->viokeepalive = vio_keepalive;
      vio->should_retry = vio_should_retry;
      vio->was_timeout = vio_was_timeout;
      vio->vioshutdown = vio_shutdown_pipe;
      vio->peer_addr = vio_peer_addr;
      vio->io_wait = no_io_wait;                    // ★ 管道无 poll，恒返回"就绪"
      vio->is_connected = vio_is_connected_pipe;
      vio->has_data = has_no_data;
      vio->is_blocking = vio_is_blocking;
      vio->set_blocking = vio_set_blocking;
      vio->set_blocking_flag = vio_set_blocking_flag;
      vio->is_blocking_flag = true;
      break;                                        // ★ 不填 vio->timeout

    case VIO_TYPE_SHARED_MEMORY:
      vio->viodelete = vio_delete_shared_memory;
      vio->read = vio_read_shared_memory;
      vio->write = vio_write_shared_memory;
      vio->vioshutdown = vio_shutdown_shared_memory;
      vio->io_wait = no_io_wait;
      vio->is_connected = vio_is_connected_shared_memory;
      vio->has_data = has_no_data;
      ...  // 其余同管道
      break;                                        // ★ 同样不填 vio->timeout
#endif /* _WIN32 */

    case VIO_TYPE_SSL:
      vio->viodelete = vio_ssl_delete;
      vio->vioerrno = vio_errno;
      vio->read = vio_ssl_read;                     // ★ 1/4
      vio->write = vio_ssl_write;                   // ★ 2/4
      vio->fastsend = vio_fastsend;
      vio->viokeepalive = vio_keepalive;
      vio->should_retry = vio_should_retry;
      vio->was_timeout = vio_was_timeout;
      vio->vioshutdown = vio_ssl_shutdown;          // ★ 3/4
      vio->peer_addr = vio_peer_addr;
      vio->io_wait = vio_io_wait;                   // ← 复用 socket 版
      vio->is_connected = vio_is_connected;
      vio->has_data = vio_ssl_has_data;             // ★ 4/4（SSL_pending）
      vio->timeout = vio_socket_timeout;            // ← 复用 socket 版
      vio->is_blocking = vio_is_blocking;
      vio->set_blocking = vio_set_blocking;
      vio->set_blocking_flag = vio_set_blocking_flag;
      vio->is_blocking_flag = true;
      break;

    default:                                        // ★ TCPIP / SOCKET / LOCAL / PLUGIN
      vio->viodelete = vio_delete;
      vio->vioerrno = vio_errno;
      vio->read = vio->read_buffer ? vio_read_buff : vio_read;   // ★★ 只有带 VIO_BUFFERED_READ 才用带缓冲读
      vio->write = vio_write;
      vio->fastsend = vio_fastsend;
      vio->viokeepalive = vio_keepalive;
      vio->should_retry = vio_should_retry;
      vio->was_timeout = vio_was_timeout;
      vio->vioshutdown = vio_shutdown;
      vio->peer_addr = vio_peer_addr;
      vio->io_wait = vio_io_wait;
      vio->is_connected = vio_is_connected;
      vio->timeout = vio_socket_timeout;
      vio->has_data = vio->read_buffer ? vio_buff_has_data : has_no_data;
      vio->is_blocking = vio_is_blocking;
      vio->set_blocking = vio_set_blocking;
      vio->set_blocking_flag = vio_set_blocking_flag;
      vio->is_blocking_flag = true;
      break;
  }                                                 // ★ in_addr / connect 两个槽全部分支都没赋值

#ifdef HAVE_KQUEUE
  vio->kq_fd = kqueue();                            // ★ BSD：每个 Vio 自带一个 kqueue fd
  if (vio->kq_fd == -1) return true;
#endif
  return false;
}
```

> ★★ **SSL 只 SSL 化了 4 个槽**（`read`/`write`/`vioshutdown`/`has_data`），其余全部复用 socket 实现——这就是"**不存在 `vio_ssl_timeout()` / `vio_ssl_keepalive()` / `vio_ssl_io_wait()` / `vio_ssl_blocking()`**"的铁证（全仓 0 命中）。
>
> ★ 缓冲区只在 `VIO_BUFFERED_READ` 时分配（`Vio::Vio(uint flags)`，`:101`）：
> ```cpp
>   if (flags & VIO_BUFFERED_READ)
>     read_buffer = (char *)my_malloc(key_memory_vio_read_buffer,
>                                     VIO_READ_BUFFER_SIZE, MYF(MY_WME));
> ```

**`vio_reset()` —— VIO 抽象的"原地变身"**（`:351`）：

```cpp
/**
  Reinitialize an existing Vio object.

  @remark Used to rebind an initialized socket-based Vio object
          to another socket-based transport type. For example,
          rebind a TCP/IP transport to SSL.

  @remark If new socket handle passed to vio_reset() is not equal
          to the socket handle stored in Vio then socket handle will
          be closed before storing new value. If handles are equal
          then old socket is not closed. This is important for
          vio_reset() usage in ssl_do().
  ...
*/
bool vio_reset(Vio *vio, enum enum_vio_type type, my_socket sd,
               void *ssl [[maybe_unused]], uint flags) {
  int ret = false;
  Vio new_vio(flags);                    // ★ 先在栈上建一个新 Vio，成功才整体搬 → 强异常安全

  /* The only supported rebind is from a socket-based transport type. */
  assert(vio->type == VIO_TYPE_TCPIP || vio->type == VIO_TYPE_SOCKET);

  if (vio_init(&new_vio, type, sd, flags)) return true;

#ifdef USE_PPOLL_IN_VIO
  /* Preserve thread_id & signal_mask for this connection */
  new_vio.thread_id = vio->thread_id;
  new_vio.signal_mask = vio->signal_mask;
#endif
  /* Preserve perfschema info for this connection */
  new_vio.mysql_socket.m_psi = vio->mysql_socket.m_psi;   // ★ PSI 句柄不能丢
  new_vio.ssl_arg = ssl;

  /*
    Propagate the timeout values. Necessary to also propagate
    the underlying proprieties associated with the timeout,
    such as the socket blocking mode.
  */
  if (vio->read_timeout >= 0)
    ret |= vio_timeout(&new_vio, 0, vio->read_timeout / 1000);   // ★ 毫秒 → 秒
  if (vio->write_timeout >= 0)
    ret |= vio_timeout(&new_vio, 1, vio->write_timeout / 1000);

  if (!ret) {
    /* Close socket only when it is not equal to the new one. */
    if (sd != mysql_socket_getfd(vio->mysql_socket)) {
      if (vio->inactive == false) vio->vioshutdown(vio);   // ★ fd 不同才关旧的
    }
#ifdef HAVE_KQUEUE
    else { close(vio->kq_fd); vio->kq_fd = -1; }
#endif
    *vio = std::move(new_vio);    // ★★★ 同一地址上的对象被整体替换
  }
  return ret;
}
```

> ★★ **上层持有的 `Vio*` / `NET*` / `THD` 指针全部不变**——这就是"TCP 连接原地升级成 SSL 而对上层透明"的全部实现基础。

**Windows 非 socket 传输的 stub 槽**（`:74`）：

```cpp
/**
  Stub io_wait method that defaults to indicate that
  requested I/O event is ready.
  Used for named pipe and shared memory VIO types.
  @retval 1       The requested I/O event has occurred.
*/
static int no_io_wait(Vio *vio [[maybe_unused]],
                      enum enum_vio_io_event event [[maybe_unused]],
                      int timeout [[maybe_unused]]) {
  return 1;
}
extern "C" {
static bool has_no_data(Vio *vio [[maybe_unused]]) { return false; }
}
```

`vio.cc` 里 `internal_vio_create` 根据 `type` 分发，填充不同的函数指针实现（`vio.cc:224-296`）：

```267:296:vio/vio.cc
    case VIO_TYPE_SSL:
      vio->viodelete = vio_ssl_delete;
      vio->vioerrno = vio_errno;
      vio->read = vio_ssl_read;
      vio->write = vio_ssl_write;
      ...
      break;

    default:                              // TCP/IP、Unix socket
      vio->viodelete = vio_delete;
      vio->vioerrno = vio_errno;
      vio->read = vio->read_buffer ? vio_read_buff : vio_read;
      vio->write = vio_write;
      ...
```

关键点：`VIO_TYPE_SSL` 的 `read` 指向 `vio_ssl_read`（`viossl.cc:263`），内部 `SSL_read`；而默认（TCP/socket）的 `read` 指向 `vio_read`（`viosocket.cc`），内部 `recv`/`read`。**上层调用 `vio->read(...)` 这一个入口，底层却走完全不同的系统调用**——这就是 vtable 多态的价值。

SSL 的 `vio_ssl_read` 展示差异：

```263:266:vio/viossl.cc
size_t vio_ssl_read(Vio *vio, uchar *buf, size_t size) {
  int ret;
  SSL *ssl = static_cast<SSL *>(vio->ssl_arg);
```

它从 `vio->ssl_arg` 取出 `SSL*` 上下文调 `SSL_read`；而普通 socket 版则直接 `recv(vio_fd(vio), ...)`。数据被 SSL 层加密/解密后，对上层完全透明。

### 各实现文件

`ls vio/` 实测 11 项：`vio.cc`、`vio_priv.h`、`viopipe.cc`（**不是 `vionp.cc`**）、`vioshm.cc`、`viosocket.cc`、`viossl.cc`、`viosslfactories.cc`、`viotest*.cc`。⚠️ **共享内存与命名管道并未被删除**，只是都在 `IF(WIN32)` 下才参与编译。

**读写：不存在 `vio_socket_read()` / `vio_socket_write()`**，裸 socket 实现就叫 `vio_read()` / `vio_write()`（`vio/viosocket.cc:136` / `:213`）：

```cpp
// :124
#if defined(__linux__)
#define VIO_USE_DONTWAIT 1
#define VIO_DONTWAIT MSG_DONTWAIT      // ★ Linux：超时靠"非阻塞 + ppoll 等待"，不切 O_NONBLOCK
#else
#define VIO_DONTWAIT 0                 // ★ 其他平台：改为切换 O_NONBLOCK
#endif

size_t vio_read(Vio *vio, uchar *buf, size_t size) {
  ssize_t ret;
  int flags = 0;
  /* Ensure nobody uses vio_read_buff and vio_read simultaneously. */
  assert(vio->read_end == vio->read_pos);

  /* If timeout is enabled, do not block if data is unavailable. */
  if (vio->read_timeout >= 0) flags = VIO_DONTWAIT;

  while ((ret = mysql_socket_recv(vio->mysql_socket, (SOCKBUF_T *)buf, size,
                                  flags)) == -1) {
    int error = socket_errno;
    /* Error encountered that is unrelated to blocking; percolate it up. */
    if (error != SOCKET_EAGAIN /* && != SOCKET_EWOULDBLOCK */) break;

    assert(error == SOCKET_EAGAIN || error == SOCKET_EWOULDBLOCK);
    if (!vio_is_blocking(vio)) {         // ★ 非阻塞：不等待，直接返回 -1
      return -1;
    }
    /* Wait for input data to become available. */
    if ((ret = vio_socket_io_wait(vio, VIO_IO_EVENT_READ))) break;
  }
  return ret;
}
```

**`vio_read_buff()`（`:179`）——三分支，注意 `vio_write_buff()` 不存在**：

```cpp
size_t vio_read_buff(Vio *vio, uchar *buf, size_t size) {
  size_t rc;
#define VIO_UNBUFFERED_READ_MIN_SIZE 2048     // ★ 阈值
  if (vio->read_pos < vio->read_end) {
    /* 分支 1：缓冲里有存货 → 纯 memcpy，零 syscall */
    rc = std::min<size_t>(vio->read_end - vio->read_pos, size);
    memcpy(buf, vio->read_pos, rc);
    vio->read_pos += rc;
  } else if (size < VIO_UNBUFFERED_READ_MIN_SIZE) {
    /* 分支 2：小请求 → 一次读满 16KB 缓冲 */
    rc = vio_read(vio, (uchar *)vio->read_buffer, VIO_READ_BUFFER_SIZE);
    if (rc != 0 && rc != (size_t)-1) {
      if (rc > size) {
        vio->read_pos = vio->read_buffer + size;
        vio->read_end = vio->read_buffer + rc;   // ★ 余量留在缓冲里
        rc = size;
      }
      memcpy(buf, vio->read_buffer, rc);
    }
  } else
    rc = vio_read(vio, buf, size);              /* 分支 3：大请求 → 绕过缓冲，直接读进用户 buf */
  return rc;
}

bool vio_buff_has_data(Vio *vio) { return (vio->read_pos != vio->read_end); }
```

**`vio_socket_io_wait()`（`:94`）——把三态 `-1/0/1` 归一成 `0/-1`**：

```cpp
int vio_socket_io_wait(Vio *vio, enum enum_vio_io_event event) {
  int timeout, ret;
  assert(event == VIO_IO_EVENT_READ || event == VIO_IO_EVENT_WRITE);
  if (event == VIO_IO_EVENT_READ) timeout = vio->read_timeout;
  else                            timeout = vio->write_timeout;

  switch (vio_io_wait(vio, event, timeout)) {
    case -1: ret = -1; break;   /* Upon failure, vio_read/write() shall return -1. */
    case 0:  ret = -1; break;   /* The wait timed out. */
    default: ret =  0; break;   /* A positive value indicates an I/O event. */
  }
  return ret;
}
```

**超时：`vio_set_timeout()` 不存在**。真实层次是 `vio_timeout()`（**秒**）→ vtable → `vio_socket_timeout()`（**毫秒**）：

```cpp
// vio/vio.cc:520
int vio_timeout(Vio *vio, uint which, int timeout_sec) {
  int timeout_ms;
  bool old_mode;
  /* Vio timeouts are measured in milliseconds. Check for a possible
     overflow. In case of overflow, set to infinite. */
  if (timeout_sec > INT_MAX / 1000) timeout_ms = -1;
  else                              timeout_ms = (int)(timeout_sec * 1000);  // ★ 秒 → 毫秒

  /* Deduce the current timeout status mode. */
  old_mode = vio->write_timeout < 0 && vio->read_timeout < 0;   // ★ 全无限 = 阻塞

  if (which) vio->write_timeout = timeout_ms;
  else       vio->read_timeout  = timeout_ms;

  /* VIO-specific timeout handling. Might change the blocking mode. */
  return vio->timeout ? vio->timeout(vio, which, old_mode) : 0;
  //                    ★ 管道/共享内存 timeout 槽为 nullptr → 直接返回 0
}
```

```cpp
// vio/viosocket.cc:310
int vio_socket_timeout(Vio *vio, uint which [[maybe_unused]], bool old_mode) {
#if defined(_WIN32)
  {
    int optname; DWORD timeout = 0;
    /* ... VIO behavior for zero timeout ... is not supported on WIN32 ... */
    if (which) { optname = SO_SNDTIMEO; if (vio->write_timeout > 0) timeout = vio->write_timeout; }
    else       { optname = SO_RCVTIMEO; if (vio->read_timeout  > 0) timeout = vio->read_timeout;  }
    ret = mysql_socket_setsockopt(vio->mysql_socket, SOL_SOCKET, optname,
                                  (const char *)&timeout, sizeof(timeout));
    //  ★ Windows 走内核级 socket 超时
  }
#else
  /*
    The MSG_DONTWAIT trick is not used with SSL sockets as the send and
    receive I/O operations are wrapped through SSL-specific functions
    (SSL_read and SSL_write) which are not equivalent to the standard
    recv(2) and send(2) used in vio_read() and vio_write(). Hence, the
    socket blocking mode is changed and vio_io_wait() is used to wait
    for I/O or timeout.
  */
#ifdef VIO_USE_DONTWAIT
  if (vio->type == VIO_TYPE_SSL)     // ★★ Linux 上只有 SSL 才动阻塞模式；裸 TCP 靠 MSG_DONTWAIT
#endif
  {
    bool new_mode = vio->write_timeout < 0 && vio->read_timeout < 0;
    if (new_mode != old_mode) ret = vio_set_blocking(vio, new_mode);  // ★ 只在跨界时才 syscall
  }
#endif
  return ret;
}
```

**阻塞模式：`vio_blocking()` / `vio_fcntl()` 不存在**，是 `vio_set_blocking()` / `vio_is_blocking()` / `vio_set_blocking_flag()`：

```cpp
// :247
int vio_set_blocking(Vio *vio, bool status) {      // status=true → 阻塞
#ifdef _WIN32
  u_long arg = status ? 0 : 1;                     // ★ FIONBIO：arg=1 非阻塞（与 POSIX 相反）
  return ioctlsocket(mysql_socket_getfd(vio->mysql_socket), FIONBIO, &arg);
#else
  int flags = fcntl(mysql_socket_getfd(vio->mysql_socket), F_GETFL, NULL);
  if (flags < 0) return -1;
  /*
    Always set/clear the flag to avoid inheritance issues. This is
    a issue mainly on Mac OS X Tiger (version 10.4) where although
    the O_NONBLOCK flag is inherited from the parent socket, the
    actual non-blocking behavior is not inherited.
  */
  if (status) flags &= ~O_NONBLOCK;
  else        flags |=  O_NONBLOCK;
  return fcntl(mysql_socket_getfd(vio->mysql_socket), F_SETFL, flags);
#endif
}

// :285
int vio_set_blocking_flag(Vio *vio, bool status) {
  /* Asynchronous communication in client is allowed only for below
     types of connections. */
  if (VIO_TYPE_TCPIP == vio->type || VIO_TYPE_SOCKET == vio->type ||
      VIO_TYPE_SSL == vio->type) {                 // ★ 只有这三种支持异步
    vio->is_blocking_flag = status;                // ★ 先改逻辑标记
    return vio_set_blocking(vio, status);          // ★ 再改真实 socket
  }
  return -1;                                       // ★ 管道/共享内存直接拒绝
}

// :305 —— 只读成员，不做 syscall（vio_read/write 每次都调它，性能关键）
bool vio_is_blocking(Vio *vio) { return vio->is_blocking_flag; }
```

**关闭：`vio_close()` 不存在**，是 `vio_shutdown()` + `vio_delete()`：

```cpp
// :462 —— 关闭 + 取消机制都在这里
int vio_shutdown(Vio *vio) {
  int r = 0;
  if (vio->inactive == false) {                    // ★ 幂等
    assert(vio->type == VIO_TYPE_TCPIP || vio->type == VIO_TYPE_SOCKET ||
           vio->type == VIO_TYPE_SSL);
    if (mysql_socket_shutdown(vio->mysql_socket, SHUT_RDWR)) r = -1;   // ① 先 shutdown

#ifdef USE_PPOLL_IN_VIO
    assert(vio->thread_id.has_value());
    if (vio->thread_id.value() != 0 && vio->poll_shutdown_flag.test_and_set()) {
      int en = pthread_kill(vio->thread_id.value(), SIGALRM);           // ② 唤醒阻塞在 ppoll 的线程
      if (en == 0) vio_wait_until_woken(vio);
      else { my_message_local(WARNING_LEVEL, EE_PTHREAD_KILL_FAILED, ...); }
    }
#elif defined HAVE_KQUEUE
    if (vio->kq_fd != -1 && vio->kevent_wakeup_flag.test_and_set())
      vio_wait_until_woken(vio);                                        // ② BSD：EVFILT_USER
#endif

    if (mysql_socket_close(vio->mysql_socket)) r = -1;                  // ③ 再 close
#ifdef HAVE_KQUEUE
    if (vio->kq_fd == -1 || close(vio->kq_fd)) r = -1;
    vio->kq_fd = -1;
#endif
  }
  vio->inactive = true;                            // ★ 置位：后续 vio_delete 不再 shutdown
  vio->mysql_socket = MYSQL_INVALID_SOCKET;        // ★ 防 use-after-close
  return r;
}
```

```cpp
// vio/vio.cc:545
void internal_vio_delete(Vio *vio) {
  if (!vio) return; /* It must be safe to delete null pointers. */
  if (vio->inactive == false) vio->vioshutdown(vio);   // ★ 走 vtable：SSL 走 vio_ssl_shutdown
  vio->~Vio();                                          // ★ placement new 分配，不能 delete
  my_free(vio);
}
void vio_delete(Vio *vio) { internal_vio_delete(vio); }
```

**错误判定族**（`vio_should_report()` / `vio_was_interrupted()` 均不存在）：

```cpp
// :74
int vio_errno(Vio *vio) {
#ifdef _WIN32
  if (vio->type == VIO_TYPE_NAMEDPIPE || vio->type == VIO_TYPE_SHARED_MEMORY)
    return GetLastError();      // ★ 管道/共享内存不走 Winsock
#endif
  return socket_errno;          // ★ Unix = errno；Windows = WSAGetLastError()
}

// :424 —— 这就是 8.0 里"是否被中断"的判定
bool vio_should_retry(Vio *vio) { return (vio_errno(vio) == SOCKET_EINTR); }

// :436
bool vio_was_timeout(Vio *vio) { return (vio_errno(vio) == SOCKET_ETIMEDOUT); }
//  ★ ETIMEDOUT 由 vio_io_wait() 在超时分支手动种下：errno = SOCKET_ETIMEDOUT
```

**`vio_keepalive()` / `vio_fastsend()`**（`:399` / `:369`）：

```cpp
int vio_keepalive(Vio *vio, bool set_keep_alive) {
  uint opt = 0;
  if (vio->type != VIO_TYPE_NAMEDPIPE) {            // ★ 只有管道被排除
    if (set_keep_alive) opt = 1;
    r = mysql_socket_setsockopt(vio->mysql_socket, SOL_SOCKET, SO_KEEPALIVE,
                                (char *)&opt, sizeof(opt));
  }
  return r;
}

int vio_fastsend(Vio *vio) {
#if defined(IPTOS_THROUGHPUT)
  { int tos = IPTOS_THROUGHPUT;
    r = mysql_socket_setsockopt(vio->mysql_socket, IPPROTO_IP, IP_TOS, &tos, sizeof(tos)); }
#endif
  if (!r) {
    int nodelay = 1;
    r = mysql_socket_setsockopt(vio->mysql_socket, IPPROTO_TCP, TCP_NODELAY,
                                &nodelay, sizeof(nodelay));   // ★ 关 Nagle
  }
  return r;
}
```

**地址相关**（`vio_in_addr()` / `vio_local_addr()` 不存在）：

```cpp
// :669
bool vio_peer_addr(Vio *vio, char *ip_buffer, uint16 *port, size_t ip_buffer_size) {
  if (vio->localhost) {
    /* Unix socket / 管道 / 共享内存：没有真实 peer，伪造 127.0.0.1:0 */
    struct in_addr *ip4 = &((struct sockaddr_in *)&(vio->remote))->sin_addr;
    vio->remote.ss_family = AF_INET;
    vio->addrLen = sizeof(struct sockaddr_in);
    ip4->s_addr = htonl(INADDR_LOOPBACK);
    my_stpcpy(ip_buffer, "127.0.0.1");
    *port = 0;
  } else {
    err_code = mysql_socket_getpeername(vio->mysql_socket, addr, &addr_length);
    if (err_code) return true;
    vio_get_normalized_ip(addr, addr_length, (struct sockaddr *)&vio->remote, &vio->addrLen);
    err_code = vio_getnameinfo((struct sockaddr *)&vio->remote, ip_buffer,
                               ip_buffer_size, port_buffer, NI_MAXSERV,
                               NI_NUMERICHOST | NI_NUMERICSERV);   // ★ 纯数字，不反解 DNS
    ...
  }
}
```

> ★ IPv6 归一化（`vio_get_normalized_ip()`，`:572`）：IPv4-mapped（`::ffff:a.b.c.d`）与 IPv4-compatible（`::a.b.c.d`）都塌回 IPv4 形式，取 `src_ip6_int32[3]`（最后 4 字节）——同地址三种写法才能正确比较。
>
> ★ `vio_getnameinfo()`（`:1257`）存在的唯一理由是**填 macOS/BSD 的 `sin_len` / `sin6_len`** 和满足 Solaris 对 salen 的要求。

**`vio_is_connected()`（`:1164`）——EOF 探测**：

```cpp
bool vio_is_connected(Vio *vio) {
  uint bytes = 0;
  if (!vio_io_wait(vio, VIO_IO_EVENT_READ, 0)) return true;   // ★ timeout=0 纯探测，不阻塞
  while (socket_peek_read(vio, &bytes)) {
    if (socket_errno != SOCKET_EINTR) return false;
  }
  /* There might be buffered data at the SSL layer. */
  if (!bytes && vio->type == VIO_TYPE_SSL)
    bytes = SSL_pending((SSL *)vio->ssl_arg);                 // ★ SSL 内部未解密数据也算"活着"
  return bytes != 0;
}

static bool socket_peek_read(Vio *vio, uint *bytes) {
#if defined(_WIN32)
  u_long len; if (ioctlsocket(sd, FIONREAD, &len)) return true; *bytes = len;
#elif defined(FIONREAD_IN_SYS_IOCTL) || defined(FIONREAD_IN_SYS_FILIO)
  int len; if (ioctl(sd, FIONREAD, &len) < 0) return true; *bytes = len;
#else
  char buf[1024];
  ssize_t res = recv(sd, &buf, sizeof(buf), MSG_PEEK);   // ★ 兜底：真 recv 但 MSG_PEEK 不取走
  if (res < 0) return true; *bytes = res;
#endif
}
```

> ⚠️ **`vio_read_available()` / `vio_write_available()` 不存在**（`socket_peek_read` 是 static，对外只有 `#ifndef NDEBUG` 的 `vio_pending()`）。
>
> **构造族**（`vio_new_win32socket()` / `vio_new_socket()` 不存在）：`vio_new()`（`vio/vio.cc:443`）、`mysql_socket_vio_new()`（`:424`，真正核心）、`vio_new_win32pipe()`（`:457`）、`vio_new_win32shared_memory()`（`:477`）。
>
> ★ `internal_vio_create()` 用 **placement new**（构造函数是 private），`mysql_socket_vio_new()` 最后才 `vio->mysql_socket = mysql_socket;`（保留 `m_psi`，所以是整体覆盖而不是只 `setfd`）。

| 文件 | 实现的 VIO 类型 |
|------|------|
| `vio.cc` | vtable 分发、`vio_read_buff` 缓冲读、通用逻辑 |
| `viosocket.cc` | TCP/IP + Unix socket（`vio_read`/`vio_write` 走 `recv`/`send`） |
| `viossl.cc` | SSL（`vio_ssl_read`/`vio_ssl_write` 走 `SSL_read`/`SSL_write`） |
| `viopipe.cc` | Windows 命名管道 |
| `vioshm.cc` | Windows 共享内存 |

### 事件等待（io_wait）

⚠️ **`vio_cancel()` 不存在**（全仓 0 命中）。8.0 的取消机制由三块拼成：`poll_shutdown_flag` + `thread_id` + `signal_mask`，`vio_shutdown()` 发 `pthread_kill(SIGALRM)`，`vio_io_wait()` 检查 flag 提前返回 `-1`。

`vio_io_wait()` 有 **三份完整实现**（`#if !defined(_WIN32) && !defined(HAVE_KQUEUE)` / `#elif defined(_WIN32)` / `#elif defined(HAVE_KQUEUE)`）。

**① Unix ppoll 版（Linux 默认，`vio/viosocket.cc:805`）**：

```cpp
/* Linux specific flag used to detect connection shutdown. The flag is
   also used for half-closed notification, which here is interpreted as
   if there is data available to be read from the socket. */
#ifndef POLLRDHUP
#define POLLRDHUP 0
#endif
#define MY_POLL_SET_IN  (POLLIN | POLLPRI)
#define MY_POLL_SET_OUT (POLLOUT)
#define MY_POLL_SET_ERR (POLLERR | POLLHUP | POLLNVAL)

int vio_io_wait(Vio *vio, enum enum_vio_io_event event, int timeout) {
  struct pollfd pfd;
  my_socket sd = mysql_socket_getfd(vio->mysql_socket);
  MYSQL_SOCKET_WAIT_VARIABLES(locker, state)     // ★ PSI socket 等待插桩

  memset(&pfd, 0, sizeof(pfd));
  pfd.fd = sd;
  switch (event) {
    case VIO_IO_EVENT_READ:
      pfd.events = MY_POLL_SET_IN;
      revents = MY_POLL_SET_IN | MY_POLL_SET_ERR | POLLRDHUP;
      break;
    case VIO_IO_EVENT_WRITE:
    case VIO_IO_EVENT_CONNECT:                   // ★ CONNECT 与 WRITE 同样等"可写"
      pfd.events = MY_POLL_SET_OUT;
      revents = MY_POLL_SET_OUT | MY_POLL_SET_ERR;
      break;
  }

  MYSQL_START_SOCKET_WAIT(locker, &state, vio->mysql_socket, PSI_SOCKET_SELECT, 0);

#ifdef USE_PPOLL_IN_VIO
  // Check if shutdown is in progress, if so return -1
  if (vio->poll_shutdown_flag.test_and_set()) {   // ★ 取消点：已在 shutdown / in-poll → 立即放弃
    MYSQL_END_SOCKET_WAIT(locker, 0);
    return -1;
  }
  timespec ts; timespec *ts_ptr = nullptr;
  if (timeout >= 0) {
    ts = {timeout / 1000, (timeout % 1000) * 1000000};   // ★ 毫秒 → {秒, 纳秒}
    ts_ptr = &ts;                                        // ★ timeout<0 → nullptr = 无限
  }
#endif

  do {
#ifdef USE_PPOLL_IN_VIO
    ret = ppoll(&pfd, 1, ts_ptr,
                !vio->thread_id.has_value() || (vio->thread_id.value() != 0)
                    ? &vio->signal_mask
                    : nullptr);
    //        ★★★ ppoll 而非 poll：原子地解除信号屏蔽，让 SIGALRM 能打断等待
    //        这就是"KILL 能立刻打断阻塞读"的实现基础。客户端库传 nullptr
#else
    ret = poll(&pfd, 1, timeout);
#endif
  } while (ret < 0 && vio_should_retry(vio) && (retry_count++ < vio->retry_count));

#ifdef USE_PPOLL_IN_VIO
  vio->poll_shutdown_flag.clear();   // ★ 退出前清 flag，让 shutdown 侧的忙等自旋结束
#endif

  switch (ret) {
    case -1: break;                  /* On error, -1 is returned. */
    case 0:
      errno = SOCKET_ETIMEDOUT;      // ★★ 超时标志是这样种下的，vio_was_timeout() 才能读到
      break;
    default:
      assert(pfd.revents & revents);
      break;
  }
  MYSQL_END_SOCKET_WAIT(locker, 0);
  return ret;
}
```

**② Windows select 版（`:906`）**：

```cpp
int vio_io_wait(Vio *vio, enum enum_vio_io_event event, int timeout) {
  struct timeval tm;
  fd_set readfds, writefds, exceptfds;
  fd = mysql_socket_getfd(vio->mysql_socket);
  if (fd == INVALID_SOCKET) return -1;

  if (timeout >= 0) {
    tm.tv_sec = timeout / 1000;
    tm.tv_usec = (timeout % 1000) * 1000;
  }
  FD_ZERO(&readfds); FD_ZERO(&writefds); FD_ZERO(&exceptfds);
  FD_SET(fd, &exceptfds);                    // ★ 异常集总是监听
  switch (event) {
    case VIO_IO_EVENT_READ:  FD_SET(fd, &readfds);  break;
    case VIO_IO_EVENT_WRITE:
    case VIO_IO_EVENT_CONNECT: FD_SET(fd, &writefds); break;
  }
  MYSQL_START_SOCKET_WAIT(locker, &state, vio->mysql_socket, PSI_SOCKET_SELECT, 0);
  /* The first argument is ignored on Windows. */
  do {
    ret = select((int)(fd + 1), &readfds, &writefds, &exceptfds,
                 (timeout >= 0) ? &tm : NULL);   // ★ timeout<0 → NULL = 无限
  } while (ret < 0 && vio_should_retry(vio) && (retry_count++ < vio->retry_count));
  MYSQL_END_SOCKET_WAIT(locker, 0);

  if (ret == 0) WSASetLastError(SOCKET_ETIMEDOUT);   // ★ Win32 用 WSASetLastError 而非 errno
  if (ret <= 0) return ret;
  switch (event) {
    case VIO_IO_EVENT_READ:  ret = (FD_ISSET(fd, &readfds) ? 1 : 0); break;
    case VIO_IO_EVENT_WRITE:
    case VIO_IO_EVENT_CONNECT: ret = (FD_ISSET(fd, &writefds) ? 1 : 0); break;
  }
  ret |= (FD_ISSET(fd, &exceptfds) ? 1 : 0);
  assert(ret);
  return ret;
}
```

**③ BSD/macOS kqueue 版（`:982`）**：

```cpp
int vio_io_wait(Vio *vio, enum enum_vio_io_event event, int timeout) {
  static const int MAX_EVENT = 2;
  struct kevent kev_set[MAX_EVENT], kev_event[MAX_EVENT];
  if (vio->kq_fd == -1) return -1;

  EV_SET(&kev_set[1], WAKEUP_EVENT_ID, EVFILT_USER,
         EV_ADD | EV_ENABLE | EV_DISPATCH | EV_CLEAR, 0, 0, nullptr);
  //     ★ 槽位 1 常驻一个用户事件（0xFACEFEED），专供 vio_shutdown 唤醒
  switch (event) {
    case VIO_IO_EVENT_READ:
      EV_SET(&kev_set[0], fd, EVFILT_READ, EV_ADD|EV_ENABLE|EV_DISPATCH|EV_CLEAR, 0,0,nullptr); break;
    case VIO_IO_EVENT_WRITE:
    case VIO_IO_EVENT_CONNECT:
      EV_SET(&kev_set[0], fd, EVFILT_WRITE, EV_ADD|EV_ENABLE|EV_DISPATCH|EV_CLEAR, 0,0,nullptr); break;
  }
  timespec ts = {timeout / 1000, (timeout % 1000) * 1000000};

  if (vio->kevent_wakeup_flag.test_and_set()) { MYSQL_END_SOCKET_WAIT(locker,0); return -1; }
  do {
    nev = kevent(vio->kq_fd, kev_set, MAX_EVENT, kev_event, MAX_EVENT,
                 timeout >= 0 ? &ts : nullptr);
  } while (nev < 0 && vio_should_retry(vio) && (retry_count++ < vio->retry_count));
  vio->kevent_wakeup_flag.clear();

  if (nev == -1) { DBUG_PRINT("error", ("kevent returned error %d\n", errno)); }
  else if (nev == 0) { errno = SOCKET_ETIMEDOUT; }
  else {
    for (int i = 0; i < nev; i++) {
      if (!(kev_event[i].flags & (EV_ERROR | EV_EOF))) { assert(...); }
      if (kev_event[i].filter == EVFILT_USER) { nev = -1; break; }  // ★ 收到唤醒 → 视为被取消
    }
  }
  return nev;
}
```

**三平台对照表**：

| 维度 | Unix（含 Linux，`USE_PPOLL_IN_VIO`） | Windows | BSD（`HAVE_KQUEUE`） |
|---|---|---|---|
| 等待原语 | `ppoll()`（无 ppoll 时退 `poll()`） | `select()` | `kevent()` |
| 超时表达 | `timespec{s, ns}`，`>=0` 才传指针 | `timeval`，`<0` 传 `NULL` | `timespec`，`<0` 传 `nullptr` |
| 超时错误 | `errno = SOCKET_ETIMEDOUT` | `WSASetLastError` | `errno = SOCKET_ETIMEDOUT` |
| 事件位 | `POLLIN\|POLLPRI` / `POLLOUT` / `POLLERR\|POLLHUP\|POLLNVAL\|POLLRDHUP` | 三个 `fd_set`，`exceptfds` 恒设置 | `EVFILT_READ` / `EVFILT_WRITE` + 常驻 `EVFILT_USER` |
| KILL 打断 | `ppoll` 传 `&vio->signal_mask` + `pthread_kill(SIGALRM)` | **无**（只有 `shutdown` + `close`） | `EVFILT_USER(0xFACEFEED)` + `NOTE_TRIGGER` |

**唤醒握手的两个版本**（`:438`）：

```cpp
#ifdef USE_PPOLL_IN_VIO
static void vio_wait_until_woken(Vio *vio) {
  while (vio->poll_shutdown_flag.test_and_set()) {
    // Wait until the vio is woken up from poll.
  }                                  // ★ 忙等自旋：等 vio_io_wait 退出时 clear()
}
#elif defined HAVE_KQUEUE
static const int WAKEUP_EVENT_ID = 0xFACEFEED;
static void vio_wait_until_woken(Vio *vio) {
  if (vio->kq_fd != -1) {
    struct kevent kev;
    EV_SET(&kev, WAKEUP_EVENT_ID, EVFILT_USER, 0, NOTE_TRIGGER, 0, NULL);
    int nev = kevent(vio->kq_fd, &kev, 1, nullptr, 0, nullptr);
    if (nev != -1) {
      while (vio->kevent_wakeup_flag.test_and_set()) { /* Wait... */ }
    }
  }
}
#endif
```

**发起取消的真正入口在 THD 侧**（`sql/sql_class.cc:1932`）：

```cpp
void THD::shutdown_active_vio() {
  mysql_mutex_assert_owner(&LOCK_thd_data);        // ★ 必须在 LOCK_thd_data 保护下
  if (active_vio) {
#ifdef USE_PPOLL_IN_VIO
    // Vio::thread_id may not be correct if the THD has been
    // associated with a different OS thread since the Vio object was
    // created. So we store the current THD::real_id here so that
    // vio_shutdown() will not try to send SIGALRM to an incorrect, or
    // invalid thread id.
    active_vio->thread_id = real_id;               // ★★ KILL 时才"现填"线程 id
#endif
    vio_shutdown(active_vio);                      // ★ 宏展开为 active_vio->vioshutdown(...)
    active_vio = nullptr;
    m_SSL = nullptr;
  }
}
```

> ★ `signal_mask` 的注入点在连接建立处（`sql/conn_handler/socket_connection.cc:150`）：`vio->thread_id.reset(); vio->signal_mask = mysqld_signal_mask;` —— `mysqld_signal_mask` 是"只放行 SIGALRM"的掩码，交给 `ppoll` 原子地解除阻塞。
>
> ★ `USE_PPOLL_IN_VIO` 的定义处：`include/violite.h:55`，`#if !defined(_WIN32) && !defined(HAVE_KQUEUE)`。

`Vio` 里的 `io_wait` 函数指针支持 poll/kqueue 事件驱动（`USE_PPOLL_IN_VIO` / `HAVE_KQUEUE` 条件编译），用于 `vio_io_wait` 在指定超时内等待可读/可写事件。这是 8.0 对阻塞模型的一个补充——上层（如半同步复制、连接管理）可以非阻塞地等 I/O 就绪。

---

### SSL 层（`vio/viossl.cc`）

⚠️ **勘误**：**不存在 `vio/viossl.h`**（`vio/` 下除 `vio_priv.h` 无任何 `.h`）；**不存在 `struct sslinfo`**（SSL 状态由 `Vio::ssl_arg` 承载，取出用 `static_cast<SSL *>(vio->ssl_arg)`）；**不存在 `vio_ssl_new()` / `vio_ssl_upgrade()` / `vio_new_ssl()`**。

`struct st_VioSSLFd` 也**不在** `viossl.cc`，而在 `include/violite.h:255`，且**只有一个成员**：

```cpp
struct st_VioSSLFd {
  SSL_CTX *ssl_context;   // ★ 整个"SSL 文件描述符"抽象 = 一个 SSL_CTX* 的包装壳
};
```

> ⚠️ **MySQL 侧没有自己的引用计数**（`SSL_CTX_up_ref` 全仓 0 命中）。生命周期由 `Ssl_acceptor_context_data` 独占持有，靠"**先 `SSL_free(acceptor_)` 再 `free_vio_ssl_acceptor_fd()`**"的固定顺序保证安全（`sql/ssl_acceptor_context_data.cc:181`）；热重载则用 `Lock_and_access_ssl_acceptor_context` 的 RAII 读写锁。

**`ssl_should_retry()`（`:224`）——WANT_READ/WRITE 语义的唯一收敛点**：

```cpp
/**
  Check if an operation should be retried and handle errors
  ...
  Note that this is not done for SSL_ERROR_WANT_READ/SSL_ERROR_WANT_WRITE
  since these are not treated as errors and a call to the function is retried.
  ...
  @return Whether a SSL I/O operation should be deferred.
  @retval true    Temporary failure, retry operation.
  @retval false   Indeterminate failure.
*/
static bool ssl_should_retry(Vio *vio, int ret, enum enum_vio_io_event *event,
                             unsigned long *ssl_errno_holder) {
  int ssl_error, err_error;
  SSL *ssl = static_cast<SSL *>(vio->ssl_arg);
  bool should_retry = true;

  ssl_error = SSL_get_error(ssl, ret);
  switch (ssl_error) {
    case SSL_ERROR_WANT_READ:
      err_error = ssl_error;  // for backward compatibility.
      *event = VIO_IO_EVENT_READ;      // ★ 不是说"读就绪"，而是"OpenSSL 需要底层 socket 可读才能推进"
      break;
    case SSL_ERROR_WANT_WRITE:
      err_error = ssl_error;
      *event = VIO_IO_EVENT_WRITE;
      break;
    default:
      /* first save the top ERR error */
      err_error = ERR_get_error();
      /* now report all remaining errors on and/or clear the error stack */
#ifndef NDEBUG
      /* Note: the OpenSSL error queue gets cleared in report_errors(). */
      report_errors(ssl);              // ★ 顺带清空错误队列（这是它的副作用，也是真正目的之一）
#else
      ERR_clear_error();
#endif
      should_retry = false;
      ssl_set_sys_error(ssl_error);
      break;
  }
  *ssl_errno_holder = err_error;
  return should_retry;
}
```

```cpp
// :157 —— SSL 错误 → 系统 errno 的翻译表
static void ssl_set_sys_error(int ssl_error) {
  int error = 0;
  switch (ssl_error) {
    case SSL_ERROR_ZERO_RETURN: error = SOCKET_ECONNRESET; break;
    case SSL_ERROR_WANT_READ:
    case SSL_ERROR_WANT_WRITE:
      error = SOCKET_EWOULDBLOCK;      // ★ 让上层 vio_should_retry/was_timeout 语义一致
      break;
    case SSL_ERROR_SSL:
#ifdef EPROTO
      error = EPROTO;
#else
      error = SOCKET_ECONNRESET;
#endif
      break;
    case SSL_ERROR_SYSCALL:
    case SSL_ERROR_NONE:
    default: break;
  };
  if (error) { IF_WIN(WSASetLastError, (void))(error); /* else */ errno = error; }
}
```

**`vio_ssl_read()`（`:263`）——非阻塞与阻塞两套语义**：

```cpp
size_t vio_ssl_read(Vio *vio, uchar *buf, size_t size) {
  int ret;
  SSL *ssl = static_cast<SSL *>(vio->ssl_arg);
  unsigned long ssl_errno_not_used;

  while (true) {
    enum enum_vio_io_event event;
    /*
      Check that the SSL thread's error queue is cleared. Otherwise
      SSL_read() returns an error from the error queue, when SSL_read() failed
      because it would block.
    */
    assert(ERR_peek_error() == 0);

    ret = SSL_read(ssl, buf, (int)size);
    if (ret > 0) break;

    /* Process the SSL I/O error. */
    if (!ssl_should_retry(vio, ret, &event, &ssl_errno_not_used)) break;

    if (!vio->is_blocking_flag) {          // ★★ 非阻塞：绝不睡眠，翻译成 size_t 负常量上抛
      switch (event) {
        case VIO_IO_EVENT_READ:  return VIO_SOCKET_WANT_READ;    // == (size_t)-2
        case VIO_IO_EVENT_WRITE: return VIO_SOCKET_WANT_WRITE;   // == (size_t)-3
        default:                 return VIO_SOCKET_ERROR;        // == (size_t)-1
      }
    }

    /* Attempt to wait for an I/O event. */
    if (vio_socket_io_wait(vio, event)) break;   // ★ 阻塞：poll/kqueue 等就绪后回到循环重试
  }
  return ret < 0 ? -1 : ret;
}
```

> ★★ **反直觉点（`vio_ssl_write()` 同样）**：`SSL_write` 也可能返回 `WANT_READ`——TLS 握手 / 重协商期间写操作需要先读对端数据。所以**两个方向都必须处理两个 event**。
>
> ★ `vio_ssl_write()`（`:305`）与之完全同构，唯一的差别是 `SSL_write(ssl, buf, (int)size)`。

**`vio_ssl_shutdown()`（`:347`）——为什么不做 close_notify**：

```cpp
int vio_ssl_shutdown(Vio *vio) {
  int r = 0;
  SSL *ssl = (SSL *)vio->ssl_arg;
  if (ssl) {
    /*
    THE SSL standard says that SSL sockets must send and receive a close_notify
    alert on socket shutdown to avoid truncation attacks. However, this can
    cause problems since we often hold a lock during shutdown and this IO can
    take an unbounded amount of time to complete. Since our packets are self
    describing with length, we aren't vulnerable to these attacks. Therefore,
    we just shutdown by closing the socket (quiet shutdown).
    */
    SSL_set_quiet_shutdown(ssl, 1);      // ★★ 不发 close_notify：MySQL 报文自带长度头

    switch ((r = SSL_shutdown(ssl))) {
      case 1: break;                     /* Shutdown successful */
      case 0:
        /*
          Shutdown not yet finished - since the socket is going to
          be closed there is no need to call SSL_shutdown() a second
          time to wait for the other side to respond
        */
        break;
      default:                           /* Shutdown failed */
        report_errors(ssl); /* or ERR_clear_error() */
        break;
    }
  }
  return vio_shutdown(vio);              // ★ 落到 socket 的 shutdown()+close()
}
```

**`vio_ssl_delete()`（`:389`）/ `vio_ssl_has_data()`（`:787`）**：

```cpp
void vio_ssl_delete(Vio *vio) {
  if (!vio) return; /* It must be safe to delete null pointer */
  if (vio->inactive == false)
    vio_ssl_shutdown(vio); /* Still open, close connection first */
  if (vio->ssl_arg) {
    SSL_free((SSL *)vio->ssl_arg);
    vio->ssl_arg = nullptr;
  }
  vio_delete(vio);
}

bool vio_ssl_has_data(Vio *vio) {
  return SSL_pending(static_cast<SSL *>(vio->ssl_arg)) > 0 ? true : false;
  //     ★ 只检查 OpenSSL 内部"已解密但用户态没读走"的明文缓冲，不检查内核 socket 缓冲
}
```

**★ `ssl_do()`（`:634`）——普通 socket → SSL 的真正升级入口**：

```cpp
static int ssl_do(struct st_VioSSLFd *ptr, Vio *vio, long timeout,
                  SSL_SESSION *ssl_session, ssl_handshake_func_t func,
                  unsigned long *ssl_errno_holder, SSL **sslptr) {
  SSL *ssl = nullptr;
  my_socket sd = mysql_socket_getfd(vio->mysql_socket);
  //                                ★★ 复用已有 Vio 里那个"已经连好的" fd —— 不新建连接

  if (*sslptr == nullptr) {
    if (!(ssl = SSL_new(ptr->ssl_context))) { ... return 1; }
    if (ssl_session != nullptr) { SSL_set_session(ssl, ssl_session); /* 会话复用 */ }
    SSL_clear(ssl);
    SSL_SESSION_set_timeout(SSL_get_session(ssl), timeout);
    SSL_set_fd(ssl, sd);                 // ★★ 把裸 fd 交给 OpenSSL，从此 BIO 拥有同一个 fd
#if defined(SSL_OP_NO_COMPRESSION)
    SSL_set_options(ssl, SSL_OP_NO_COMPRESSION);
#endif
    *sslptr = ssl;
    pfs_ssl_setup_instrumentation(vio, ssl);
  } else {
    ssl = *sslptr;
  }
  ERR_clear_error();

  size_t loop_ret;
  if ((loop_ret = ssl_handshake_loop(vio, ssl, func, ssl_errno_holder))) {
    if (loop_ret != VIO_SOCKET_ERROR) {
      return (int)loop_ret;  // Don't free SSL     ★ 非阻塞下返回 WANT_* 时保留 SSL* 续握手
    }
    SSL_free(ssl);
    *sslptr = nullptr;
    return (int)VIO_SOCKET_ERROR;
  }

  /*
    Connection succeeded. Install new function handlers,
    change type, set sd to the fd used when connecting
    and set pointer to the SSL structure
  */
  if (vio_reset(vio, VIO_TYPE_SSL, SSL_get_fd(ssl), ssl, 0)) return 1;
  //  ★★★ 就这一行：同一个 Vio 对象原地"变身"成 SSL

  /* ... DBUG：打印 cipher / peer certificate / shared ciphers ... */
  return 0;
}

int sslaccept(struct st_VioSSLFd *ptr, Vio *vio, long timeout,
              unsigned long *ssl_errno_holder) {
  return ssl_do(ptr, vio, timeout, nullptr, SSL_accept, ssl_errno_holder, nullptr);
}

int sslconnect(struct st_VioSSLFd *ptr, Vio *vio, long timeout,
               SSL_SESSION *session, unsigned long *ssl_errno_holder, SSL **ssl) {
  return ssl_do(ptr, vio, timeout, session, SSL_connect, ssl_errno_holder, ssl);
}
```

> ★ `ssl_handshake_loop()`（`:407`）与 read/write **完全同构**：阻塞时 `vio_socket_io_wait()`，非阻塞时返回 `VIO_SOCKET_WANT_READ/WRITE`。

**工厂函数（`vio/viosslfactories.cc`）**：

```cpp
// :659 客户端
struct st_VioSSLFd *new_VioSSLConnectorFd(...) {
  int verify = SSL_VERIFY_PEER;
  /* Turn off verification of servers certificate if both
     ca_file and ca_path is set to NULL */
  if (ca_file == nullptr && ca_path == nullptr) verify = SSL_VERIFY_NONE;

  if (!(ssl_fd = new_VioSSLFd(..., /*is_client=*/true, ...))) return nullptr;
  SSL_CTX_set_verify(ssl_fd->ssl_context, verify, nullptr);
  //                                                    ★ 回调为 nullptr
  return ssl_fd;
}

// :687 服务端
struct st_VioSSLFd *new_VioSSLAcceptorFd(...) {
  int verify = SSL_VERIFY_PEER | SSL_VERIFY_CLIENT_ONCE;
  if (!(ssl_fd = new_VioSSLFd(..., /*is_client=*/false, ...))) return nullptr;
  /* Set max number of cached sessions, returns the previous size */
  SSL_CTX_sess_set_cache_size(ssl_fd->ssl_context, 128);
  SSL_CTX_set_verify(ssl_fd->ssl_context, verify, nullptr);
  /*
    Set session_id - an identifier for this server session
    Use the ssl_fd pointer
  */
  SSL_CTX_set_session_id_context(ssl_fd->ssl_context,
                                 (const unsigned char *)ssl_fd, sizeof(ssl_fd));
  return ssl_fd;
}

// :716 —— CTX 生命周期终点（唯一释放点）
void free_vio_ssl_acceptor_fd(struct st_VioSSLFd *fd) {
  SSL_CTX_free(fd->ssl_context);
  my_free(fd);
}
```

> ⚠️ **不存在 `vio_ssl_verify_callback`**。8.0.39 用 `SSL_CTX_set_verify(ctx, verify, nullptr)` 把校验交给 OpenSSL 内建；客户端额外校验在 `sql-common/client.cc:3642 ssl_verify_server_cert()`，服务端自检在 `sql/ssl_acceptor_context_data.cc:40 verify_store_cert()`。

**连接层如何建 VIO、SSL 何时升级**（`sql/conn_handler/socket_connection.cc:150` / `:216`）：

```cpp
// UDS：出生即 VIO_TYPE_SOCKET，永不会升 SSL
  Vio *create_and_init_vio() const override {
    Vio *vio = mysql_socket_vio_new(m_connect_sock, VIO_TYPE_SOCKET, VIO_LOCALHOST);
#ifdef USE_PPOLL_IN_VIO
    if (vio != nullptr) {
      vio->thread_id.reset();
      vio->signal_mask = mysqld_signal_mask;
    }
#endif
    return vio;
  }

// TCP：出生是 VIO_TYPE_TCPIP，稍后可能被 ssl_do()/vio_reset() 改成 VIO_TYPE_SSL
  Vio *create_and_init_vio() const override {
    Vio *vio = mysql_socket_vio_new(m_connect_sock, VIO_TYPE_TCPIP, 0);
    ...
  }
```

```cpp
// sql/auth/sql_authentication.cc:2885 —— 唯一的升级入口
  if (protocol->has_client_capability(CLIENT_SSL)) {
    std::unique_ptr<Lock_and_access_ssl_acceptor_context> context;
    //  ★★ 这就是"引用计数"的真实形态：RAII + 读写锁，而非原子计数
    if (thd->is_admin_connection() && g_admin_ssl_configured == true)
      context = std::make_unique<Lock_and_access_ssl_acceptor_context>(mysql_admin);
    else
      context = std::make_unique<Lock_and_access_ssl_acceptor_context>(mysql_main);
    if (!context.get()->have_ssl()) return packet_error;
    DBUG_PRINT("info", ("IO layer change in progress..."));
    if (sslaccept(*(context.get()), protocol->get_vio(),
                  protocol->get_net()->read_timeout, &errptr)) {
      return packet_error;
    }
    int rc = protocol->read_packet();
    //  ★★ 升级后立刻在同一个 Vio 上读第二个握手包 —— 上层代码毫无感知
    mpvio->vio_is_encrypted = 1;
```

## 相关的系统变量/状态变量

VIO 本身几乎没有直接暴露的系统变量，相关参数集中在连接层（`connect_timeout`、`net_read_timeout`、`net_write_timeout`、`interactive_timeout`、`wait_timeout`），这些由 `NET`/`THD` 层消费，通过 `vio->read_timeout`/`write_timeout` 生效，不直接属于 VIO 变量。

---

## Misc

### 易混淆概念

- **VIO vs NET**：VIO 是"怎么收发字节"（socket/SSL/管道），NET 是"字节流怎么拆成 MySQL 协议包"（4 字节包头 + payload）。`NET` 结构体里持有 `Vio *vio`（`mysql.h.pp:140`），是 NET 的上游。
- **VIO vs MYSQL_PLUGIN_VIO**：`MYSQL_PLUGIN_VIO` 是认证插件看到的 VIO 接口（`include/mysql/plugin_auth.h`），`MPVIO_EXT` 是它的服务器端内部扩展（`sql_authentication.h:61`）。前者是"插件能调用的 VIO 函数"，后者是"认证会话上下文 + VIO"。
- **VIO vs vio/ 目录**：`vio/` 目录是 VIO 的实现；`include/violite.h` 是对外头文件。

---

## 参考

**官方文档**
- MySQL 8.0 Reference Manual → Connection Transport Protocols

**相关文档**
- 认证插件如何通过 VIO 收发握手数据见 [`mfa.md`](../auth/mfa.md)
- 网络协议包（NET 层）拆分与组装（待补链接）

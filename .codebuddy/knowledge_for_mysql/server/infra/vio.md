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

| 文件 | 实现的 VIO 类型 |
|------|------|
| `vio.cc` | vtable 分发、`vio_read_buff` 缓冲读、通用逻辑 |
| `viosocket.cc` | TCP/IP + Unix socket（`vio_read`/`vio_write` 走 `recv`/`send`） |
| `viossl.cc` | SSL（`vio_ssl_read`/`vio_ssl_write` 走 `SSL_read`/`SSL_write`） |
| `viopipe.cc` | Windows 命名管道 |
| `vioshm.cc` | Windows 共享内存 |

### 事件等待（io_wait）

`Vio` 里的 `io_wait` 函数指针支持 poll/kqueue 事件驱动（`USE_PPOLL_IN_VIO` / `HAVE_KQUEUE` 条件编译），用于 `vio_io_wait` 在指定超时内等待可读/可写事件。这是 8.0 对阻塞模型的一个补充——上层（如半同步复制、连接管理）可以非阻塞地等 I/O 就绪。

---

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

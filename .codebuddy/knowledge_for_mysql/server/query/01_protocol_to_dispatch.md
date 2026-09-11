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

### 协议层的设计取向：简单、兼容优先

MySQL 客户端/服务器协议有几个刻意的取舍，贯穿本节：

- **文本为主的结果集**：默认把数字转成十进制字符串发出去，而不是二进制——牺牲带宽换调试友好（`tcpdump` 能直接看懂），binary 协议只给 Prepared Statement 用
- **length-coded integer**：变长整数（1/3/4/9 字节），小值省字节——因为绝大多数长度值很小，这是典型的"为常见情况优化"
- **16MB 分片**：单包上限 `MAX_PACKET_LENGTH`，用"尾片 < 16MB 即结束"表达多包边界，不引入额外 EOF 标记——协议极简

### one-thread-per-connection：历史包袱，但简单可靠

线程模型（第三节）用的是最朴素的**一连接一线程**，而不是线程池/协程。这在今天看来低效（每连接一个 256KB~MB 级栈），但：

- 实现极其简单，每个连接的 THD 状态天然隔离，**没有共享状态竞争**
- 历史包袱：MySQL 诞生于"连接数不大"的年代，且一直没遇到非改不可的压力（连接风暴是运维层面用连接池解决的问题）

这个取舍与后面各篇形成对照——MySQL 的很多设计都是"**简单 > 极致性能**"。

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

### 1.9 回包链路

```
send_result_set_row → start_row()/store_*() → end_row()
end_row: my_net_write(packet)                              protocol_classic.cc:3365
my_net_write → net_write_buff（缓冲）→ net_write_packet（可选压缩）
             → net_write_raw_loop → vio_write →（socket send / SSL_write）
```

关键行号：`Protocol_classic::write` `:1380`、`flush` `:3038`、`end_row` `:3363-3367`、`net_send_ok` 的 `my_net_write + net_flush` `:966-967`。

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

### 握手与认证（`sql/auth/sql_authentication.cc`）

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

# mysql-test 框架（MTR）原理剖析

> 基于 MySQL 8.0.39。本篇剖析 MTR 的**实现原理**：Perl 调度层（`mysql-test-run.pl`）与 C++ 执行层（`client/mysqltest.cc`）的分工、服务器 bootstrap 流程、`mysqltest` 测试语言的命令表与解释器结构、结果比对与 `--record`、用例组织与调度。
>
> **边界**：本篇讲"MTR 这套代码是怎么工作的"，不讲"如何写测试用例"的方法论。DBUG/`SET debug` 通道见 [`dbug.md`](dbug.md)；编译构建见 [`build.md`](build.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、双层架构：调度器与执行器](#一双层架构调度器与执行器)
  - [二、目录与文件类型](#二目录与文件类型)
  - [三、服务器生命周期：bootstrap 与启动](#三服务器生命周期bootstrap-与启动)
  - [四、mysqltest 测试语言与命令表](#四mysqltest-测试语言与命令表)
  - [五、结果比对与 --record](#五结果比对与---record)
  - [六、用例发现与调度](#六用例发现与调度)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MTR 是 MySQL 的回归测试框架。它的核心是一个**双层结构**：

| 层 | 语言 | 程序 | 职责 |
|---|---|---|---|
| **调度层** | Perl | `mysql-test/mysql-test-run.pl` | 发现用例、启停 mysqld、管理复制拓扑、并行调度、报告 |
| **执行层** | C++ | `client/mysqltest.cc`（编译出 `mysqltest` 二进制） | 解释 `.test` 脚本、连上服务器执行、产生输出、与 `.result` 比对 |

★ 这个双层是理解 MTR 一切行为的前提：**`.test` 文件不是 Perl 脚本，也不是 shell**，而是一种由 `mysqltest` 解释的专用语言（本篇第四章列出完整命令表）。

调度层的 Perl 模块（`mysql-test-run.pl` 的 `use` 列表，源码顶部）：

```
My::ConfigFactory   生成 my.cnf
My::CoreDump        core dump 检测与处理
My::File::Path      File::Path 的补丁版
My::Find            定位 mysqld / mysqltest 等二进制
My::Options         命令行选项解析
My::Platform        平台差异
My::SafeProcess     安全进程（保证子进程被清理）
My::SysInfo          CPU/内存信息（决定并发数）
mtr_cases           用例收集
mtr_cases_from_list 从 list 文件读用例
mtr_match           字符串匹配工具
mtr_report          报告输出
mtr_results         结果收集
```

### 一个反直觉的开场

`mysqltest` 是**用真实客户端协议连接真实 mysqld 的**，不是模拟器。所以：

- `.test` 里的每一行 SQL 都会真正走一遍解析 → 优化 → 执行；
- 测试之间靠"重启 mysqld"或"重建表"来隔离，而不是靠 mock；
- 一个测试慢，往往不是 MTR 慢，而是**服务器真的在做那些事**。

由此带来一个重要的工程后果：**MTR 用例的"结果文件"（`.result`）记录的是真实输出**，任何优化器输出格式的改动都会导致成百上千个 `.result` 失效。这也是为什么 MySQL 社区里优化器改动必须同步更新 result 文件。

### 版本演进

| 版本 | 变化 |
|---|---|
| 4.x | MTR 早期形态（shell + mysqltest） |
| 5.x | 稳定为 Perl 调度器 + mysqltest 客户端双层 |
| 5.6+ | 引入 `My::SafeProcess`（解决孤儿进程）、`--parallel` |
| 8.0 | bootstrap 改用 `mysqld --initialize-insecure` + `--init-file`（此前是 `mysql_install_db` / `--bootstrap` 直灌 SQL） |
| 8.0.x | `collections/` 用于 CI 分组；`skip_if_hypergraph` 命令（因 hypergraph 与经典优化器结果不同） |

---

## 理论基础

### Golden file 测试范式

MTR 采用的是经典的 **golden file（基线文件）** 范式：

```
.test    输入脚本
.result  期望输出（"golden"）
         ↓ 实际输出 == 期望输出？ → pass / fail
```

它的优点是"任何行为变化都会被捕获"，缺点是**对输出的稳定性要求极高**——任何非确定性的输出（时间、路径、PID、线程 id、行数估算）都会让用例 flaky。

MTR 解决这个问题靠两套机制（本篇第四章、第五章）：

1. **结果修饰命令**：`replace_result` / `replace_column` / `replace_regex` / `sorted_result` 等，把非确定性输出规范化
2. **日志抑制**：`disable_query_log` / `disable_result_log` / `disable_warnings`，干脆不记录某些输出

### 为什么是 Perl + C++ 双语

| 职责 | 为什么用这门语言 |
|---|---|
| 调度（Perl） | 进程/文件/配置管理是 Perl 的强项；跨平台（含 Windows）；无需编译 |
| 执行（C++） | 需要用 MySQL 客户端协议（`libmysql`）；要精确控制连接、异步查询（`send`/`reap`）、会话跟踪 |

★ 关键点是 `send` / `reap` 这对命令：它们要求客户端能**异步**发出查询后再回收结果，这是测试并发/锁等待的基础。用 C++ 直接调 `mysql_send_query` / `mysql_read_query_result` 才能做到，Perl DBI 做不到这个粒度。

### 他库对比

| 库 | 测试框架 | 差异 |
|---|---|---|
| **MySQL** | MTR（Perl + mysqltest，golden file） | 双层、真实服务器、基线文件 |
| **PostgreSQL** | `pg_regress`（shell + psql，golden file）+ TAP 测试（Perl） | 同为 golden file，但更轻量；PG 还有大量单元测试 |
| **SQLite** | TCL 脚本 + 单一 C 文件 | 极简 |
| **Oracle** | 闭源 | — |

MySQL 的特色是 **`.test` 的 DSL 与真实服务器的组合**——它让"测试"与"真实执行"之间没有缝隙，代价是测试运行很重。

---

## 核心实现

### 一、双层架构：调度器与执行器

```
mysql-test-run.pl（Perl）
  │
  ├─ 解析命令行（My::Options）
  ├─ 定位二进制（My::Find：mysqld / mysqltest / mysqladmin …）
  ├─ 收集用例（mtr_cases / mtr_cases_from_list）
  ├─ 生成配置（My::ConfigFactory → my.cnf）
  ├─ bootstrap 数据目录（mysqld --initialize-insecure）
  ├─ 对每个用例：
  │     ├─ 起/复用 mysqld（My::SafeProcess）
  │     ├─ 调 mysqltest 执行 .test
  │     │      └─ mysqltest 自己比对 .result
  │     └─ 收集结果（mtr_results）
  └─ 报告（mtr_report）

client/mysqltest.cc（C++）
  │
  ├─ 解析 .test → 命令序列（command_names[] typelib）
  ├─ 逐条解释执行
  ├─ 连服务器（可多连接）
  ├─ 输出写进 result 文件
  └─ 与 .result 比对（或 --record 时重写）
```

★ 注意职责边界：**比对发生在 mysqltest 里，不在 Perl 里**。Perl 只看 mysqltest 的退出码。

### 二、目录与文件类型

> `mysql-test/` 下的主要目录。

| 目录 | 用途 |
|---|---|
| `suite/` | 各个测试套件（`main`、`innodb`、`rpl`、`opt_trace`、`json` …），每个套件下是 `t/`（脚本）与 `r/`（结果） |
| `include/` | 可复用的 `.inc` 片段（被 `--source` 引入） |
| `extra/` | 辅助脚本（如 `binlog_tests/`） |
| `collections/` | CI 用的分组清单（`.push` / `.list` / `.def`），指定某次 CI 跑哪些套件 |
| `lib/` | Perl 模块（`My/*.pm`、`mtr_*.pm`）与少量测试辅助 C++ |

**用例相关的文件类型**（同名、不同后缀构成一个用例）：

| 后缀 | 作用 |
|---|---|
| `.test` | 测试脚本 |
| `.result` | 期望输出（基线） |
| `.opt` | 启动 mysqld 时的**额外选项**（例如 `--optimizer_switch=...`） |
| `.cnf` | 额外的 my.cnf 片段 |
| `.inc` | 被 `--source` 引入的片段 |
| `.reject` | **比对失败时**生成的实际输出（用于排查与 `--record`） |
| `.rdiff` | reject 与 result 的差异 |

★ `.opt` 文件是理解"为什么某些用例的服务器配置不同"的关键：每个用例可以自带 mysqld 启动参数，MTR 会为它单独起服务器。

### 三、服务器生命周期：bootstrap 与启动

> `mysql-test-run.pl`。

#### 3.1 bootstrap：创建初始数据目录

8.0 用 `mysqld --initialize-insecure` 而不是旧的 `--bootstrap` 直灌：

```perl
mtr_add_arg($args, "--initialize-insecure");
...
mtr_add_arg($args, "--init-file=$bootstrap_sql_file");
```

`--init-file` 指向 MTR 生成的 `bootstrap.sql`，其内容由 MTR 逐段拼出（源码里一段段 `mtr_tofile`）：

```
use mysql;
（视情况）DROP DATABASE sys;
DELETE FROM mysql.user where user= '';      ← 清掉匿名用户
CREATE DATABASE test;
CREATE DATABASE mtr;                        ← MTR 自己用的库
include/mtr_warnings.sql                    ← 安装 MTR 的告警过滤
include/mtr_check.sql                       ← 安装 MTR 的检查存储过程
（视情况）use test; + 追加 init-file 内容
```

★ 两个细节：

- **`mtr` 这个库是 MTR 专用的**，存放告警过滤规则与检查过程（所以你在 `.test` 里看到的 `call mtr.check_testcase()` 之类来自这里）。
- **`MYSQLD_BOOTSTRAP_CMD` / `MYSQLD_INSTALL_CMD` 环境变量**会被导出，供测试内部需要重新 bootstrap 的场景使用（源码注释明说"Save the value of MYSQLD_BOOTSTRAP_CMD before a test with bootstrap"）。

#### 3.2 启动参数来自三个来源

| 来源 | 说明 |
|---|---|
| `My::ConfigFactory` 生成的 `my.cnf` | 通用配置（端口、socket、datadir、log） |
| 用例的 `.opt` 文件 | 该用例专属的 mysqld 选项 |
| 命令行 `--mysqld=...` | 全局附加 |

★ `My::ConfigFactory` 是一个"配置工厂"：它按模板生成多个 mysqld 实例的 `my.cnf`（主从复制场景下会有多个）。

#### 3.3 进程管理：`My::SafeProcess`

MySQL 测试里最麻烦的问题是**孤儿 mysqld 进程**——测试崩溃后 mysqld 还在跑，占用端口导致后续测试失败。

`My::SafeProcess` 的做法是：所有子进程由一个**守护进程**（`safe_process`）启动并监护，父进程死了守护进程会确保子进程也被清理。这是一个典型的"用监护进程解决孤儿进程"的方案。

### 四、mysqltest 测试语言与命令表

> `client/mysqltest.cc`。

#### 4.1 命令是怎么被识别的

```cpp
const char *command_names[] = {
    "connection", "query", "connect", "sleep", "inc", "dec", "source",
    "disconnect", "let", "echo", "expr", "while", "end", "save_master_pos",
    "sync_with_master", "sync_slave_with_master", "error", "send", "reap",
    "dirty_close", "replace_result", "replace_column", "ping", "eval",
    /* Enable/disable that the _query_ is logged to result file */
    "enable_query_log", "disable_query_log",
    /* Enable/disable that the _result_ from a query is logged to result file */
    "enable_result_log", "disable_result_log", "enable_connect_log",
    "disable_connect_log", "wait_for_slave_to_stop", "enable_warnings",
    "disable_warnings", "enable_info", "disable_info",
    "enable_session_track_info", "disable_session_track_info",
    "enable_metadata", "disable_metadata", "enable_async_client",
    "disable_async_client", "exec", "execw", "exec_in_background", "delimiter",
    "disable_abort_on_error", "enable_abort_on_error", "vertical_results",
    "horizontal_results", "query_vertical", "query_horizontal", "sorted_result",
    "partially_sorted_result", "lowercase_result", "skip_if_hypergraph",
    "start_timer", "end_timer", "character_set", "disable_ps_protocol",
    "enable_ps_protocol", "disable_reconnect", "enable_reconnect", "if",
    "disable_testcase", "enable_testcase", "replace_regex",
    "replace_numeric_round", "remove_file", "file_exists", "write_file",
    "copy_file", "perl", "die", "assert",

    /* Don't execute any more commands, compare result */
    "exit", "skip", "chmod", "append_file", "cat_file", "diff_files",
    "send_quit", "change_user", "mkdir", "rmdir", "force-rmdir", "force-cpdir",
    "list_files", "list_files_write_file", "list_files_append_file",
    "send_shutdown", "shutdown_server", "result_format", "move_file",
    "remove_files_wildcard", "copy_files_wildcard", "send_eval", "output",
    "reset_connection", "query_attributes",

    nullptr};

TYPELIB command_typelib = {array_elements(command_names), "", command_names, nullptr};
```

★ 命令名被做成一个 **`TYPELIB`**——这正是 MySQL 服务器内部解析 `ENUM` / 系统变量时用的同一个结构。用 `find_type()` 做命令查找，**哈希 + 排序数组查找**。这是"复用服务器基础设施"的一个例子。

#### 4.2 命令分类

| 类别 | 命令 |
|---|---|
| **连接管理** | `connect` / `connection` / `disconnect` / `ping` / `dirty_close` / `send_quit` / `change_user` / `reset_connection` |
| **查询执行** | `query` / `send` / `reap` / `send_eval` / `eval` / `error`（声明下一条预期报错） |
| **变量与流程** | `let` / `inc` / `dec` / `expr` / `if` / `while` / `end` / `die` / `assert` / `skip` / `exit` |
| **结果修饰** | `replace_result` / `replace_column` / `replace_regex` / `replace_numeric_round` / `sorted_result` / `partially_sorted_result` / `lowercase_result` / `result_format` / `vertical_results` / `horizontal_results` |
| **日志开关** | `enable/disable_query_log` / `enable/disable_result_log` / `enable/disable_warnings` / `enable/disable_info` / `enable/disable_metadata` / `enable/disable_abort_on_error` / `enable/disable_ps_protocol` / `enable/disable_reconnect` / `enable/disable_async_client` |
| **文件与进程** | `write_file` / `append_file` / `cat_file` / `diff_files` / `remove_file` / `file_exists` / `copy_file` / `move_file` / `chmod` / `mkdir` / `rmdir` / `list_files` / `exec` / `execw` / `exec_in_background` / `perl` |
| **复制同步** | `save_master_pos` / `sync_with_master` / `sync_slave_with_master` / `wait_for_slave_to_stop` |
| **服务器控制** | `shutdown_server` / `send_shutdown` |
| **其它** | `source`（引入 .inc）/ `sleep` / `echo` / `delimiter` / `character_set` / `start_timer` / `end_timer` / `output` / `query_attributes` / `skip_if_hypergraph` |

#### 4.3 几个值得注意的命令

| 命令 | 为什么重要 |
|---|---|
| **`send` / `reap`** | 异步查询：`send` 发出后不等待，`reap` 才收结果。测试锁等待、并发、复制延迟全靠它 |
| **`error`** | 声明下一条语句预期返回某个错误码；不写则任何错误都算测试失败 |
| **`sorted_result`** | 让结果按行排序后再比对——用于"顺序不确定但集合确定"的查询（如无 ORDER BY 的 JOIN） |
| **`replace_result` / `replace_column`** | 把非确定性字符串（临时路径、PID）替换成固定值 |
| **`disable_ps_protocol` / `enable_ps_protocol`** | 控制是否走 prepared statement 协议——同一个测试能同时覆盖两种协议路径 |
| **`skip_if_hypergraph`** | 8.0 新增：hypergraph 与经典优化器计划不同时，用它跳过（这本身就是"两套优化器并存"带来的测试负担） |
| **`source`** | 引入 `.inc`，是 MTR 的"函数库"机制 |
| **`assert` / `die`** | 流程断言 |

#### 4.4 变量替换

`let $var = <value>` 定义变量，之后 `$var` 在后续语句中被替换。`eval` 用于"把变量展开后再执行"（普通 `query` 不做变量展开）。

这是一个**两级执行**设计：`query` 直接发送原文，`eval` 先做变量替换再发送。

### 五、结果比对与 --record

#### 5.1 比对流程

```
mysqltest 执行 .test
   ├─ 每条命令的输出写入内存 result 缓冲
   └─ 执行结束（exit 命令 / 文件末尾）
         ├─ 有 --result-file：与 .result 逐行比对
         │     ├─ 相同 → 退出码 0
         │     └─ 不同 → 生成 .reject + .rdiff，退出码非 0
         └─ 有 --record：直接重写 .result（"录制基线"）
```

★ `--record` 的本质是**重新录制 golden file**。这是 MTR 工作流里最常用也最危险的选项：它会把当前行为（包括 bug 行为）写成基线。

#### 5.2 为什么会有 flaky

非确定性输出的典型来源：

| 来源 | 处理手段 |
|---|---|
| 行数估算 / 代价数字 | `replace_column` / 只测计划形态不测数字 |
| 文件绝对路径 | `replace_result` |
| 时间 | `replace_regex` |
| 无 ORDER BY 的行序 | `sorted_result` |
| 线程 id / PID | `replace_regex` |
| 告警消息 | `disable_warnings` |

理解这一点后就能明白：MTR 用例里大量 `replace_*` 命令不是为了"隐藏问题"，而是**golden file 范式对确定性的必然要求**。

### 六、用例发现与调度

> `mtr_cases.pm`。

#### 6.1 发现

- 指定 suite 时扫描该 suite 的 `t/*.test`
- 指定单个用例时精确匹配
- `--suite` 可多指定；`collections/` 下的清单文件用于 CI 一次性指定一批

#### 6.2 调度

| 选项 | 作用 |
|---|---|
| `--parallel=N` | 并行跑 N 个用例（每个用例可能起自己的 mysqld） |
| `--repeat=N` | 重复跑 N 次（查 flaky 的常用手段） |
| `--force` | 失败不停（默认失败即停） |
| `--big-test` | 跑标记为 big 的用例 |
| `--record` | 录制基线 |
| `--start` / `--start-and-exit` | 起服务器但不跑用例（**调试时极有用**：起好环境后手动连接） |
| `--gdb` / `--ddd` / `--dbx` | 在调试器下跑 |

★ `--start-and-exit` 是内核开发最常用的选项：它起好一个按 MTR 配置启动的 mysqld 并退出，然后你可以用 `mysql` 客户端连上去手工复现——**复现环境与你写用例时的环境完全一致**。

#### 6.3 与构建系统的衔接

`mysql-test/CMakeLists.txt` 把 MTR 接入 CMake：编译完成后可以直接 `ctest` 或 `make test` 触发（但 MTR 的主要用法仍是直接调 `mysql-test-run.pl`，因为选项更丰富）。

---

## ★ 本机制里的工程实现技法

### 1. 复用服务器的 `TYPELIB` 做命令解析

`command_typelib` 用的是 MySQL 服务器内部同一个 `TYPELIB` 结构与 `find_type()` 查找函数。这是一个"**基础设施复用**"的例子：客户端工具与服务器共享 mysys 的字符串表机制。

### 2. `--init-file` 注入初始化 SQL

8.0 的 bootstrap 不再直灌 SQL，而是用 `mysqld --initialize-insecure --init-file=bootstrap.sql`。这样初始化逻辑走的是**标准的服务器初始化路径**，而不是 MTR 私有的引导路径——减少了"测试环境与真实环境不一致"的风险。

### 3. `My::SafeProcess`：用守护进程解决孤儿进程

测试框架最头疼的"崩溃后残留进程"问题，MySQL 的解法是引入一个监护进程（`safe_process`），由它启动并看护目标进程，父进程异常退出时由监护进程负责清理。

### 4. `send` / `reap` 暴露协议层的异步能力

MTR 没有自己实现"并发测试"的抽象，而是**直接把客户端协议的异步能力暴露成两个命令**。这让测试能表达"发出查询但不等待"这类时序，代价是测试脚本要自己管理时序（容易写出 flaky 用例）。

### 5. 用环境变量传递 bootstrap 命令

`MYSQLD_BOOTSTRAP_CMD` / `MYSQLD_INSTALL_CMD` 被导出到环境，供需要重新初始化数据目录的测试使用。这是"**跨进程传递复杂命令**"的朴素做法（在 Perl 里拼好命令字符串，子进程从环境读）。

### 6. golden file + 结果修饰命令的组合

MTR 没有为"非确定性输出"设计专门机制，而是给了 `replace_*` / `sorted_result` / `disable_*` 一组命令，把规范化责任交给用例作者。这是"**机制简单、约定靠人**"的设计取向。

---

## 可观测性

### 跑测试时的产物位置

| 路径 | 内容 |
|---|---|
| `mysql-test/var/log/` | 各 mysqld 的错误日志、`bootstrap.log` |
| `mysql-test/var/tmp/` | `bootstrap.sql` 等临时文件 |
| `mysql-test/var/run/` | socket 与 pid |
| `mysql-test/var/log/mysqltest.log` | mysqltest 的输出 |
| `suite/<x>/r/<name>.reject` | 比对失败时的实际输出 |
| `suite/<x>/r/<name>.rdiff` | reject 与 result 的差异 |

### 常用排查手段

| 场景 | 做法 |
|---|---|
| 用例失败看差异 | 看 `.reject` 与 `.rdiff` |
| 想手工复现 | `--start-and-exit` 起服务器，再手工连 |
| 想看服务器日志 | `mysql-test/var/log/mysqld.1.err`（编号按实例） |
| 想确认 bootstrap 干了什么 | 看 `var/tmp/bootstrap.sql` 与 `var/log/bootstrap.log` |
| 怀疑 flaky | `--repeat=N` |

### 调试服务器与 MTR 结合

| 需求 | 选项 |
|---|---|
| 在 gdb 下跑 | `--gdb` |
| 起好环境不跑用例 | `--start-and-exit` |
| 让服务器输出 DBUG | `--debug=...`（见 [`dbug.md`](dbug.md)） |
| 单跑一个用例 | `./mtr <suite>.<name>` |

★ `--start-and-exit` + 手工 `mysql` 客户端，是"把 MTR 的环境配置能力与交互式调试结合"的标准做法——比在 `.test` 里反复试错高效得多。

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 加一个新命令 | `client/mysqltest.cc` 的 `command_names[]` + 对应的 `do_*` 函数 + 命令分发 |
| 加一个新 suite | `mysql-test/suite/<name>/`（`t/` + `r/`）+ 如需进 CI 则加到 `collections/` |
| 改 bootstrap 内容 | `mysql-test-run.pl` 里拼 `bootstrap.sql` 的那段，或 `include/mtr_*.sql` |
| 给某类测试统一加服务器选项 | 用 `.opt` 文件（按用例）或修改 `My::ConfigFactory`（全局） |

### 已知缺陷

- **golden file 范式对输出稳定性要求极高**，任何优化器输出格式改动都会波及大量 `.result`。
- **`--record` 会把 bug 行为固化成基线**，误用会掩盖问题。
- **测试运行重**：每个用例可能重启 mysqld，套件全跑耗时很长。
- **flaky 用例排查成本高**：需要 `--repeat` 反复跑。
- **`skip_if_hypergraph` 的存在本身说明两套优化器的输出差异是系统性的**，而不是个别用例问题。

### 社区边界澄清

- **MTR 是社区版自带的完整测试框架**，与 Oracle 内部的测试套件不同——社区版源码树里看到的只是公开的部分。
- **CI 用的 `collections/` 清单**是 Oracle 内部的 CI 配置，社区开发者本地通常不用。
- **没有"单元测试"层面的优化器测试**：优化器改动几乎只能通过 MTR 的 SQL 级用例验证（这也是为什么 result 文件如此重要）。

---

## 参考

**官方文档**

- MySQL 8.0 Reference Manual, "The MySQL Test Suite"
- MySQL Internals / Source Documentation: *MySQL Test framework manual*（`mysql-test-run.pl` 文件头注释里给出的链接：`https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN.html`）
- `mysql-test/README`、`mysql-test/README.stress`、`mysql-test/README.gcov`

**源码出处**

- `mysql-test/mysql-test-run.pl` —— 调度层主程序；模块 `use` 列表在文件顶部
- `mysql-test/lib/mtr_*.pm`、`mysql-test/lib/My/*.pm` —— Perl 模块
- `client/mysqltest.cc` —— 执行层；`command_names[]`（本篇第四章完整引用）
- `mysql-test/include/mtr_warnings.sql`、`mtr_check.sql` —— bootstrap 注入的告警过滤与检查过程
- `mysql-test/collections/` —— CI 分组清单

**相关文档**

- [`dbug.md`](dbug.md) —— `SET debug` / DBUG 通道（MTR 里用 `--debug` 打开）
- [`build.md`](build.md) —— 编译构建（产出 mysqld / mysqltest 二进制）
- [`../../server/query/11_explain_and_trace.md`](../../server/query/11_explain_and_trace.md) —— optimizer trace（MTR 用例里最常用的观察手段）

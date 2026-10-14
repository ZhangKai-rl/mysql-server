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

⚠️ **勘误**：**不存在 `mysql-test/mysql-test-run.cc`**。全仓库只有 `mysql-test/mysql-test-run.pl`（268 KB）和 `mysql-test/mysql-test-run.dox`（276 KB Doxygen 手册）。`mysql-test/` 下递归搜 `*.cc` 只有 3 个：`lib/My/SafeProcess/safe_kill_win.cc`、`safe_process.cc`、`safe_process_win.cc`——**都是守护/看门狗进程，不是调度器**。

真正的分工：

| 层 | 程序 | 语言 | 职责 |
|---|---|---|---|
| 调度器 | `mysql-test-run.pl` | Perl | 发现用例、fork worker、启停 mysqld、收结果、报报告 |
| 执行器 | `mysqltest`（`client/mysqltest.cc`） | C++ | 解释 `.test` 脚本、发 SQL、**比对结果并给出退出码** |
| 守护 | `mysqltest_safe_process` | C++ | 包裹 mysqld/mysqltest，防孤儿进程 |

**两层之间唯一的接口是 5 个退出码**（`mysql-test-run.pl:362`）：

```perl
use constant { MYSQLTEST_PASS        => 0,
               MYSQLTEST_FAIL        => 1,
               MYSQLTEST_SKIPPED     => 62,
               MYSQLTEST_NOSKIP_PASS => 63,
               MYSQLTEST_NOSKIP_FAIL => 64 };
```

> ★★ **Perl 侧不做 diff**。它只删旧的 `.reject`（`:4461`），真正的比对与 `.reject` 生成在 `client/mysqltest.cc:2219-2234`。

**调度器 `main()` 骨架**（`mysql-test-run.pl:442`，节选关键 9 步）：

```perl
sub main {
  command_line_setup();                          # ① 参数解析（Getopt::Long）
  create_unique_id_dir();                        # ② /tmp/mysql-unique-ids 锁目录
  My::SafeProcess::find_bin($bindir, $path_client_bindir);   # ③ 定位 mysqld/mysqltest/safe_process

  find(\&get_all_suites, "$glob_mysql_test_dir");# ④ 扫 suite 目录（标准库 File::Find）

  my $sys_info = My::SysInfo->new();
  $ENV{NUMBER_OF_CPUS} = $sys_info->num_cpus();
  if ($opt_parallel eq "auto") {                 # ⑤ worker 数 = CPU 数，可被 MTR_MAX_PARALLEL 压
    $opt_parallel = $ENV{NUMBER_OF_CPUS};
    if (defined $ENV{MTR_MAX_PARALLEL}) {
      my $max_par = $ENV{MTR_MAX_PARALLEL};
      $opt_parallel = $max_par if ($opt_parallel > $max_par);
    }
    $opt_parallel = 1 if ($opt_parallel < 1);
  }

  my $tests = collect_test_cases($opt_reorder, $opt_suites, \@opt_cases,
                                 $opt_skip_test_list);        # ⑥ 收集用例
  initialize_servers();                                       # ⑦ bootstrap 数据目录

  $opt_parallel = $num_tests if $opt_parallel > $num_tests;   # ⑧ 不超过用例总数
  if ($opt_parallel > 1 && ($opt_start_exit || $opt_stress
                            || $is_option_mysqlx_port_set)) {
    $opt_parallel = 1;      # ★ 与 --start-and-exit / --stress / --mysqlx_port 互斥
  }

  my $server = new IO::Socket::INET(Listen    => $opt_parallel,  # ⑨ master 监听，backlog = worker 数
                                    LocalAddr => 'localhost',
                                    Proto     => 'tcp',);
  my $server_port = $server->sockport();

  for my $child_num (1 .. $opt_parallel) {                    # ⑩ fork worker
    my $child_pid = My::SafeProcess::Base::_safe_fork();
    if ($child_pid == 0) {
      $server = undef;    # Close the server port in child
      set_vardir("$opt_vardir/$child_num");   # ★ 每个 worker 独立 var/N 与 tmpdir/N
      $opt_tmpdir = "$opt_tmpdir/$child_num";
      run_worker($server_port, $child_num);   # 永不返回
      exit(1);
    }
    $children{$child_pid} = 1;
  }

  my $completed = run_test_server($server, $tests, $opt_parallel);   # ⑪ 调度主循环
  ...
  mtr_report_stats("Completed", $completed, $opt_accept_fail);       # ⑫ 统计报告
}
```

> ★ 并行是 **fork 多进程 + 本机 loopback TCP**，**不是** Perl `threads`、`My::ExecFD`（后者全仓 0 命中）。`threads` 只用于**收集用例阶段**（`mtr_cases.pm:463-497`）。

**worker 主循环**（`:1314`）——注意它连回 master 的握手顺序：

```perl
sub run_worker ($) {
  my ($server_port, $thread_num) = @_;

  my $server = new IO::Socket::INET(PeerAddr => 'localhost',
                                    PeerPort => $server_port,
                                    Proto    => 'tcp');
  report_option('name', "worker[$thread_num]");     # ★ 输出里的 w1/w2/... 前缀
  set_build_thread_ports($thread_num);              # ★ 每个 worker 一段独立端口
  environment_setup();

  my $hello = <$server>;    # ★ 等 master 的 "HELLO\n"
  setup_vardir();
  check_running_as_root();

  print $server "START\n";  # ★ 向 master 要第一个用例

  while (my $line = <$server>) {
    chomp($line);
    if ($line eq 'TESTCASE') {
      my $test = My::Test::read_test($server);      # 反序列化用例对象
      $test->{worker} = $thread_num if $opt_parallel > 1;
      run_testcase($test);                          # ★ 起 mysqld / 起 mysqltest / 判结果
      $test->write_test($server, 'TESTRESULT');     # ★ 结果回传
    } elsif ($line eq 'BYE') {
      my $ret = stop_all_servers($opt_shutdown_timeout);
      print $server "SRV_CRASH\n" if $shutdown_report;
      print $server "VALGREP\n" if $valgrind_reports;
      print_times_used($server, $thread_num);       # ★ 上报 'SPENT'
      exit($valgrind_reports);
    }
  }
}
```

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

**真实目录**（`ls mysql-test/` 核实）：

```
mysql-test/
  mysql-test-run.pl          <-- 268 KB，调度器（Perl）
  mysql-test-run.dox         <-- 276 KB，Doxygen 手册
  mysql-test-run.pl 的符号链接 mtr 由 CMake 生成（不是自定义 target）
  collections/  extra/  include/  r/  std_data/  suite/  t/   <-- main 套件
  lib/                       <-- 15 项
    My/      mtr_cases.pm  mtr_cases_from_list.pm  mtr_gcov.pl  mtr_gprof.pl
             mtr_io.pl  mtr_lock_order.pl  mtr_match.pm  mtr_misc.pl
             mtr_process.pl  mtr_report.pm  mtr_results.pm  mtr_stress.pl
             mtr_unique.pm  t/
    My/  Config.pm ConfigFactory.pm CoreDump.pm Exec.pm File/ Find.pm Handles.pm
         Memcache.pm Options.pm Platform.pm SafeProcess/ SafeProcess.pm SysInfo.pm Test.pm
```

**每个套件（suite）的内部约定**——`collect_one_suite()`（`lib/mtr_cases.pm:776`）决定：

```perl
  my $suitedir = "$::glob_mysql_test_dir";     # main 套件直接用 mysql-test/ 根
  if ($suite ne "main") { ... $suitedir = get_suite_dir($suite); ... }

  my $resdir  = "$suitedir/r";
  my $testdir = "$suitedir/t";

  if (-d $testdir) {
    if (-d $resdir) { } else { $resdir = $testdir; }   # 没有 r/ 就用 t/
  } else {
    mtr_error("Can't have r/ dir without t/") if -d $resdir;
    $resdir = $testdir = $suitedir;                    # 都没有就用 suite 根
  }
```

**文件类型全表**（在 `collect_one_test_case()` 里逐个被识别，`mtr_cases.pm:1144`）：

| 文件 | 位置 | 作用 | 识别处 |
|---|---|---|---|
| `<t>.test` | `t/` | 测试脚本 | `opendir(TESTDIR)` + `mtr_match_extension($elem,'test')` |
| `<t>.result` | `r/` | golden file | `$tinfo->{result_file} = "$resdir/$tname.result"` |
| `<t>.cnf` | `t/` | 该用例专属 my.cnf 模板 | `$tinfo->{'template_path'} = $test_cnf_file` |
| `my.cnf` | `<suite>/` | 套件级 my.cnf | 无 `<t>.cnf` 时回落 |
| `suite.opt` | `t/`（注意不是 suite 根） | 套件级 mysqld 选项 | `opts_from_file($suite_opt_file)` |
| `<t>-master.opt` / `-slave.opt` / `-client.opt` | `t/` | 单用例选项 | `process_opts_file()` |
| `<t>-master.sh` / `-slave.sh` | `t/` | 启动前后跑 shell | Windows 上直接 skip |
| `combinations` / `<t>.combinations` | `<suite>/` 或 `t/` | 同用例多组开关展开 | `combinations_from_file()` |
| `disabled.def` | `collections/` | 禁用清单 | `create_disabled_test_list()` |
| `include/*.inc` | `include/` | 可 source 的片段 + **tag 标记** | `tags_from_test_file()` |

**优先级链**（`mtr_cases.pm`）：`<t>.cnf` > `<suite>/my.cnf` > `include/default_my.cnf`（rpl 套件例外强制用 `suite/rpl/my.cnf`）。

**`process_opts_file()`（`:1067`）——`.opt` 里有一部分是"给 MTR 自己看的指令"，不是 mysqld 选项**：

```perl
sub process_opts_file {
  my ($tinfo, $opt_file, $opt_name) = @_;
  if (-f $opt_file) {
    my $opts = opts_from_file($opt_file);
    foreach my $opt (@$opts) {
      my $value;
      $value = mtr_match_prefix($opt, "--timezone=");
      if (defined $value) { $tinfo->{'timezone'} = $value; next; }

      $value = mtr_match_prefix($opt, "--result-file=");
      if (defined $value) {
        $tinfo->{'result_file'} = "r/$value.result";   # ★ 换一个 golden file
        next;
      }

      $value = mtr_match_prefix($opt, "--config-file-template=");
      if (defined $value) {
        $tinfo->{'template_path'} = dirname($tinfo->{path}) . "/$value"; next;
      }

      if ($opt eq "--force-restart") {
        # ★ 强制重启：即使下一个用例配置相同也不复用 mysqld
        $tinfo->{'force_restart'} = 1; next;
      }
      $value = mtr_match_prefix($opt, "--testcase-timeout=");
      if (defined $value) { $tinfo->{'case-timeout'} = $value; next; }
      if ($opt eq "--nowarnings") { $tinfo->{'skip_check_warnings'} = 1; next; }

      push(@{ $tinfo->{$opt_name} }, $opt);   # ★ 其余原样传给 mysqld
    }
  }
}
```

> ★ `--force-restart` **不是 mysqld 选项**，它是 MTR 的调度指令；同理 `--nowarnings`、`--testcase-timeout=`。

**combinations：一个用例展开成多个**（`mtr_cases.pm:675`）——文件用 **my.cnf 语法**，每个 `[组名]` 是一组开关：

```perl
sub combinations_from_file($) {
  my $config = My::Config->new($combination_file);
  foreach my $group ($config->groups()) {
    my $comb = {};
    $comb->{name} = $group->name();                 # ★ 组名 = 组合名（出现在报告里）
    foreach my $option ($group->options()) {
      push(@{ $comb->{comb_opt} }, $option->option());
    }
    push(@combinations, $comb);
  }
  return @combinations;
}

sub create_test_combinations($$) {
  foreach my $comb (@{$combinations}) {
    # ★ 去重：命令行 / .opt 里已显式设过的组合跳过
    if (My::Options::is_set($test->{master_opt}, $comb->{comb_opt}) ||
        My::Options::is_set($test->{slave_opt},  $comb->{comb_opt}) ||
        My::Options::is_set(\@::opt_extra_bootstrap_opt, $comb->{comb_opt}) ||
        My::Options::is_set(\@::opt_extra_mysqld_opt,    $comb->{comb_opt})) {
      next;
    }
    my $new_test = My::Test->new();
    while (my ($key, $value) = each(%$test)) {      # ★ 深拷贝原用例
      if (ref $value eq "ARRAY") { push(@{ $new_test->{$key} }, @$value); }
      else { $new_test->{$key} = $value; }
    }
    push(@{ $new_test->{master_opt} }, @{ $comb->{comb_opt} });
    push(@{ $new_test->{slave_opt} },  @{ $comb->{comb_opt} });
    $new_test->{combination} = $comb->{name};
    push(@new_cases, $new_test);
  }
  return @new_cases;
}
```

> ★ 两个层级：`suite/combinations`（对整个套件生效，`collect_one_suite:895`）与 `t/<name>.combinations`（只对该用例，`collect_one_test_case:1455`）。

**tag 表：靠扫 `.test` 里的 include 打标记**（`mtr_cases.pm:1476`）——这是"为什么加一行 `source include/have_ndb.inc;` 就变成 NDB 测试"的原理：

```perl
my @tags = (
  [ "include/have_binlog_format_row.inc", "binlog_formats", ["row"] ],
  [ "include/have_log_bin.inc", "need_binlog", 1 ],
  [ "include/force_myisam_default.inc", "myisam_test", 1 ],
  [ "include/big_test.inc",       "big_test",   1 ],
  [ "include/asan_have_debug.inc","asan_need_debug", 1 ],
  [ "include/have_debug.inc",     "need_debug", 1 ],
  [ "include/have_ndb.inc",       "ndb_test",   1 ],
  [ "include/full_result_diff.inc", "full_result_diff", 1 ],
  [ "run_ndbapitest.inc", "ndbt_test",  1 ],
  [ "include/rpl_init.inc",    "rpl_test", 1 ],
  [ "include/ndb_master-slave.inc", "ndb_test",       1 ],
  [ "federated.inc",                "federated_test", 1 ],
  [ "include/not_windows.inc",      "not_windows",    1 ],
  [ "include/not_parallel.inc",     "not_parallel",   1 ],
  # ★ 打上这个 tag 的用例会被 run_test_server 挑出来，最后由最后一个 worker 串行跑
  [ "have_group_replication_plugin.inc", "grp_rpl_test", 1 ],
  [ "include/no_valgrind_without_big.inc", "no_valgrind_without_big", 1 ]);
```

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

⚠️ **勘误**：8.0.39 **不用 `mysqld --bootstrap`**，用的是 **`--initialize-insecure`**。"bootstrap" 只残留在函数名 `mysql_install_db()`、`sql_to_bootstrap()`、变量名 `$bootstrap_sql_file` 和注释里。

```perl
# mysql-test-run.pl:4118
sub initialize_servers {
  if (using_extern()) {
    if (!-d $opt_vardir) { setup_vardir(); }
  } else {
    kill_leftovers("$opt_vardir/run");
    if (!$opt_start_dirty) {              # ★ --start-dirty 跳过整个初始化
      remove_stale_vardir();
      setup_vardir();
      mysql_install_db(default_mysqld(), "$opt_vardir/data/");
      $initial_bootstrap_cmd = $ENV{'MYSQLD_BOOTSTRAP_CMD'};
    }
  }
}
```

**`mysql_install_db()` 核心**（`:4205`，节选）——★ 注意环境变量传递技法：

```perl
sub mysql_install_db {
  my ($mysqld, $datadir, $bootstrap_opts) = @_;

  my $args;
  mtr_init_args(\$args);
  mtr_add_arg($args, "--no-defaults");
  mtr_add_arg($args, "--initialize-insecure");   # ★★ 8.0 的 bootstrap 就是这个
  mtr_add_arg($args, "--loose-skip-ndbcluster");
  mtr_add_arg($args, "--tmpdir=%s", "$opt_vardir/tmp/");
  mtr_add_arg($args, "--core-file");
  mtr_add_arg($args, "--datadir=%s", "$install_datadir");
  mtr_add_arg($args, "--secure-file-priv=%s", "$opt_vardir");
  mtr_add_arg($args, "--innodb_buffer_pool_size=24M");
  mtr_add_arg($args, "--innodb-redo-log-capacity=10M");
  mtr_add_arg($args, "--innodb_autoextend_increment=8");
  mtr_add_arg($args, "--loose-auto_generate_certs=OFF");
  ...
  # The user can set MYSQLD_BOOTSTRAP to the full path to a mysqld
  # to run a different mysqld during --initialize.
  my $exe_mysqld_bootstrap = $ENV{'MYSQLD_BOOTSTRAP'} || find_mysqld($install_basedir);

  # Export MYSQLD_BOOTSTRAP_CMD variable containing <path>/mysqld <args>
  $ENV{'MYSQLD_BOOTSTRAP_CMD'} = "$exe_mysqld_bootstrap " . join(" ", @$args);
  $ENV{'MYSQLD_INSTALL_CMD'}   = "$exe_mysqld_bootstrap " . join(" ", @$args);
  #   ★★★ "用环境变量传递 bootstrap 命令"就在这里：
  #   .test 文件可以写 --exec $MYSQLD_BOOTSTRAP_CMD，.opt 文件可用 ${MYSQLD_BOOTSTRAP_CMD}

  # Create the bootstrap.sql file
  my $bootstrap_sql_file = "$opt_vardir/tmp/bootstrap.sql";
  mtr_add_arg($args, "--init-file=$bootstrap_sql_file");   # ★★ 拼接好的 SQL 这样喂进去

  mtr_tofile($bootstrap_sql_file, "use mysql;\n");
  mtr_appendfile_to_file("include/mtr_test_data_timezone.sql", $bootstrap_sql_file);
  mtr_tofile($bootstrap_sql_file, "DELETE FROM mysql.user where user= '';\n");
  mtr_tofile($bootstrap_sql_file, "CREATE DATABASE test;\n");
  mtr_tofile($bootstrap_sql_file, "CREATE DATABASE mtr;\n");
  mtr_tofile($bootstrap_sql_file, mtr_grab_file("include/mtr_warnings.sql"));
  mtr_tofile($bootstrap_sql_file, mtr_grab_file("include/mtr_check.sql"));
  if (defined $init_file) {                       # 用户 --initialize=--init-file=X
    mtr_tofile($bootstrap_sql_file, "use test;\n");
    mtr_appendfile_to_file($init_file, $bootstrap_sql_file);
  }

  my $path_bootstrap_log = "$opt_vardir/log/bootstrap.log";
  mtr_tofile($path_bootstrap_log, "$exe_mysqld_bootstrap " . join(" ", @$args) . "\n");

  my $res = My::SafeProcess->run(name   => "initialize",
                                 path   => $exe_mysqld_bootstrap,
                                 args   => \$args,
                                 input  => $bootstrap_sql_file,
                                 output => $path_bootstrap_log,
                                 error  => $path_bootstrap_log,
                                 append => 1,
                                 verbose => $opt_verbose,);
  if ($res != 0) {
    mtr_error("Error executing mysqld --initialize\n" .
              "Could not install system database from $bootstrap_sql_file\n" .
              "see $path_bootstrap_log for errors");
  }
  # Remove the auto.cnf so that a new auto.cnf is generated for master
  # and slaves when the server is restarted
  if (-f "$datadir/auto.cnf") { unlink "$datadir/auto.cnf"; }
}
```

> ★ 产物：`var/log/bootstrap.log`、`var/tmp/bootstrap.sql`、`var/data/`。**完整 bootstrap 命令行只在 `bootstrap.log` 第一行**——这是排查"改了 my.cnf 后起不来"的第一现场。

**`sql_to_bootstrap()`**（`:4145`）——把多行 SQL 压成一行以适配 `--init-file`：

```perl
# Remove all newline characters expect after semicolon
sub sql_to_bootstrap {
  my ($sql) = @_;
  my @lines = split(/\n/, $sql);
  my $result = "\n";
  my $delimiter = ';';
  foreach my $line (@lines) {
    if ($line =~ /^delimiter (.*)/) { ... $delimiter = $new; next; }
    if ($line =~ /\Q$delimiter\E$/) {
      $line =~ s/\Q$delimiter\E$/;/;
      $result .= "$line\n";
      next;
    }
    next if ($line =~ /^\s*--/);            # 去掉 -- 注释
    $line =~ s/\'\@HOSTNAME\@\'/localhost/;
    $result .= "$line ";                    # 其余行只加空格，不加换行
  }
  return $result;
}
```

**启动参数的优先级**（`mysqld_arguments()`，`:6286`）：

```perl
  # Options specified in .opt files should be added last so they can
  # override defaults above.
  ...
  foreach my $arg (@$extra_opts) {          # suite.opt + master.opt + --mysqld= + restart_opts
    next if (grep { $arg =~ $_ } @options); # 选项文件类参数已先处理
    if ($arg =~ /--init[-_]file=(.*)/) {
      $arg = "--init-file=" . get_bld_path($1);   # out-of-source 修正
    } elsif ($arg eq "--daemonize") {
      $mysqld->{'daemonize'} = 1;
    }
    mtr_add_arg($args, "%s", $arg);
  }
```

> ★ `--defaults-group-suffix`（`:6463`）是 my.cnf 里 `[mysqld.1]` / `[mysqld.2]` 分组能命中的原因。
>
> ★ `.opt` 文件里可以写 `${MYSQLD}` 之类环境变量，由 `get_extra_opts()`（`:6809`）展开：
> ```perl
>   foreach my $opt (@$opts) {
>     $opt =~ s/\$\{(\??\w+)\}/envsubst($1)/ge;
>     $opt =~ s/\$(\??\w+)/envsubst($1)/ge;
>   }
> ```

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

⚠️ **勘误**：`SafeProcess.pm` **没有 POD**（grep `=head1` 0 命中）。"为什么需要守护进程"的权威解释是 `package` 之后的**普通 `#` 注释块**（`:27-55`）：

```perl
# Class that encapsulates process creation, monitoring and cleanup.
#
# Spawns a monitor process which spawns a new process locally or remote
# using subclasses My::Process::Local or My::Process::Remote etc.
#
# The monitor process runs a simple event loop more or less just
# waiting for a reason to zap the process it monitors. Thus the user
# of this class does not need to care about process cleanup, it's
# handled automatically.
#
# The monitor process wait for:
#  - the parent process to close the pipe, in that case it
#    will zap the "monitored process" and exit
#  - the "monitored process" to exit, in which case it will exit
#    itself with same exit code as the "monitored process"
#  - the parent process to send the "shutdown" signal in wich case
#    monitor will kill the "monitored process" hard and exit
#
# When used it will look something like this:
# $> ps
#  [script.pl]
#   - [monitor for `mysqld`]
#     - [mysqld]
#   - [monitor for `mysqld`]
#     - [mysqld]
#
use base 'My::SafeProcess::Base';   # ★ 真正的 spawn 能力来自基类
```

> ★ 进程树是 **3 层**：`mysql-test-run.pl` →(fork+exec)→ `mysqltest_safe_process` →(fork+exec)→ `mysqld`。
> ⚠️ 第 30 行提到的 `My::Process::Local` / `My::Process::Remote` **在 8.0.39 不存在**（`lib/My/` 下无 `Process.pm`），是历史遗留注释。

**第 1 层：Perl 的 fork + exec**（`lib/My/SafeProcess/Base.pm:87`）：

```perl
sub create_process {
  ...
  my $pipe = IO::Pipe->new();
  my $pid  = _safe_fork();          # ★ 裸 fork()，不是 threads
  if ($pid) {
    $pipe->reader();
    my $line = <$pipe>;             # ★ 阻塞等子进程说 "ready\n"，确保 exec 成功后才返回
    if (defined $pid_file) {
      sleep 1 until -e $pid_file;   # ★ --daemonize 场景：mysqld 还会再 fork，真 pid 从文件读
      open FILE, $pid_file; chomp(my $pid_val = <FILE>); close FILE;
      return $pid_val;
    }
    return $pid;
  }
  $SIG{INT} = 'DEFAULT';
  setpgrp(0, 0) if $opts{setpgrp};  # ★ 自成进程组，便于整体 kill
  if ($output and !open(STDOUT, $open_mode, $output)) { ... }
  if (!exec($path, @$args)) {       # ★ exec mysqltest_safe_process -- <mysqld args>
    croak("Failed to exec '$path': $!");
  }
  $pipe->writer();
  print $pipe "ready\n";
}

# Retry a couple of times if fork returns EAGAIN
sub _safe_fork {
  my $retries = 5;
  my $pid;
FORK: {
    $pid = fork;
    if (not defined($pid)) {
      croak("fork failed after: $!") if (!$retries--);
      warn("fork failed sleep 1 second and redo: $!");
      sleep(1);
      redo FORK;
    }
  }
  return $pid;
}

sub exit_status {
  my $self = shift;
  my $raw  = $self->{EXIT_STATUS};
  if ($raw & 127) { return 1; }      # 被信号杀死 → 当作失败
  else { return $raw >> 8; }         # ★ 还原成进程 exit code（mysqltest 的 0/1/62/63/64）
}
```

**第 2 层：`safe_process.cc` 守护进程**（`lib/My/SafeProcess/safe_process.cc:175`，Unix 版完整关键段）：

```cpp
/*
  Utility program that encapsulates process creation, monitoring
  and bulletproof process cleanup

  Usage:
    safe_process [options to safe_process] -- progname arg1 ... argn
  ...
  The safe_process then checks the following things:
  1. Child exits, propagate the child's return code to the parent
     by exiting with the same return code as the child.
  2. Parent dies, immediately kill the child and exit, thus the
     parent does not need to properly cleanup any child, it is handled
     automatically.
  3. Signal's recieced by the process will trigger same action as 2)
*/

int main(int argc, char *const argv[]) {
  pid_t parent_pid = getppid();
  struct sigaction sa, sa_abort;
  sa.sa_handler = handle_signal;
  sa.sa_flags = SA_NOCLDSTOP;        // 子进程 stop 不触发 SIGCHLD
  sigaction(SIGTERM, &sa, nullptr);  // ★ Perl 的 start_kill() 发 TERM → 这里
  sigaction(SIGSEGV, &sa, nullptr);
  sigaction(SIGINT,  &sa, nullptr);
  sigaction(SIGCHLD, &sa, nullptr);
  sigaction(SIGABRT, &sa_abort, nullptr);   // dump_core() 发 ABRT → abort_child()

  /* Parse arguments */
  for (int i = 1; i < argc; i++) {
    if (strcmp(arg, "--") == 0 && strlen(arg) == 2) {
      child_argv = &argv[i + 1];     // ★ "--" 之后全部属于 mysqld
      break;
    } else if (strcmp(arg, "--verbose") == 0) { verbose++; }
    else if (strncmp(arg, "--parent-pid", 12) == 0) { ... }
    else if (strcmp(arg, "--nocore") == 0) { nocore = true; }
    else if (strncmp(arg, "--env ", 6) == 0) { putenv(strdup(arg + 6)); }
  }

  char buf; int pfd[2];
  if (pipe(pfd) == -1) die("Failed to create pipe");   // ★ 唯一的 pipe，只用于 ready 握手

  while ((child_pid = fork()) == -1) { message("fork failed"); sleep(1); }

  if (child_pid == 0) {
    close(pfd[0]);
    signal(SIGTERM, SIG_DFL);        // ★ 恢复默认 handler，MTR 的 TERM 才能正常关 mysqld
    signal(SIGINT, SIG_DFL);
    signal(SIGCHLD, SIG_DFL);
    signal(SIGABRT, SIG_DFL);
    setpgid(0, 0);                   // ★ 自成进程组 → kill(-child_pid, SIGKILL) 能清干净
    if (nocore) { struct rlimit corelim = {0,0}; setrlimit(RLIMIT_CORE, &corelim); }
    buf = 37;                        // ★★ 魔数 37：只表示"我要 exec 了"，不是退出码
    if ((write(pfd[1], &buf, 1)) < 1) die("Failed to signal that child is ready");
    close(pfd[1]);
    if (execvp(child_argv[0], child_argv) < 0) die("Failed to exec child");
  }

  close(pfd[1]);
  while ((read(pfd[0], &buf, 1)) < 1) {
    if (errno != EINTR) die("Failed to read signal from child %d", errno);
  }
  if (buf != 37) die("Didn't get 37 from pipe");
  close(pfd[0]);

  /* Monitor loop */
  while (true) {
    if (kill(parent_pid, 0) != 0) {  // ★★ 信号 0 = 仅探测存活：父进程死了 → 立刻杀 mysqld
      print_message("Parent is not alive anymore, parent pid %d:", parent_pid);
      kill_child();
    }
    if (terminated) { kill_child(); }
    if (aborted)    { abort_child(); }
    sleep(1);                        // ★ 1 秒轮询
  }
}
```

```cpp
static void wait_pid(void) {
  int status = 0;
  pid_t ret_pid;
  while ((ret_pid = waitpid(child_pid, &status, 0)) < 0) {
    if (errno != EINTR) die("Failure to wait for child %d, errno %d", child_pid, errno);
  }
  if (ret_pid == child_pid) {
    int exit_code = 1;
    if (WIFEXITED(status)) {
      exit_code = WEXITSTATUS(status);
      if (exit_code != 0 && exit_code != 62)     // ★ 62 = skip，不打印错误噪音
        print_message("Child process: %d, exit: %d", child_pid, exit_code);
      exit(exit_code);        // ★★★ 退出码回传就这一行：以子进程的 exit code 自己退出
    }
    if (WIFSIGNALED(status))
      print_message("Child process: %d, killed by signal: %d", child_pid, WTERMSIG(status));
    exit(exit_code);
  }
}

static void abort_child(void) {
  kill(-child_pid, SIGABRT);   // ★ 负号 = 杀整个进程组
  wait_pid();
}
static void kill_child(void) {
  kill(-child_pid, SIGKILL);   // ★ SIGKILL 进程组，保证 mysqld 的子进程也被带走
  wait_pid();
}

extern "C" void handle_signal(int sig) { terminated = sig; }  // ★ 只置标志位，async-signal-safe
extern "C" void handle_abort(int sig)  { aborted = sig; }
```

> ⚠️ **不存在 `--safe-parent-pid`**（Unix 真名 `--parent-pid`）；**不存在 `parent_fd`** 回传退出码——pipe 只传 1 个字节 `37`。

**优雅停机优先于硬杀**（`SafeProcess.pm:213`）：

```perl
sub shutdown {
  my $shutdown_timeout = shift;
  foreach my $proc (@processes) {
    my $shutdown = $proc->{SAFE_SHUTDOWN};   # ★ mysqld 传的是 sub { mysqld_stop($mysqld) }
    if ($shutdown_timeout > 0 and defined $shutdown) {
      $shutdown->();                         # ★ 先试 mysqladmin shutdown
      $proc->{WAS_SHUTDOWN} = 1;
    } else { $proc->start_kill(); }
  }
  ...
  foreach my $proc (@kill_processes) {
    warn "ERROR: A process did not stop gracefully, killing it forcefully. ...";
    $proc->start_kill();
  }
}

sub start_kill {
  ...
  $ret = kill("TERM", $pid);   # Unix：给 safe_process 发 SIGTERM，由它 kill_child()
}
```

> ⚠️ **不存在 `poll()`**——等价功能是 `check_any()`（`:547`，非阻塞 `wait_one(0)`）。
> ⚠️ `wait_one()` 的超时是用 `alarm($timeout)` + `local $SIG{ALRM} = sub { die "waitpid timeout"; }` 打断阻塞 `waitpid` 实现的（`:425`）。
>
> ★ `END{}` 兜底（`:73`）：Perl 退出时把 `%running` 里还活着的进程全部 `kill()`，这是最后一道防孤儿闸门。

**Windows 分支是独立编译单元** `safe_process_win.cc`（不是 `#ifdef _WIN32`），由 `lib/My/SafeProcess/CMakeLists.txt:24` 的 `IF(WIN32)` 选择。它用三样 Unix 没有的东西：
- `get_parent_pid()` 靠 `CreateToolhelp32Snapshot` 反查（Windows 无 `getppid()`）
- `CreateEvent` 命名事件 `"safe_process[pid]"` + `mysqltest_safe_kill.exe` 远程触发
- **Job Object**：`CREATE_SUSPENDED` 建进程 → `AssignProcessToJobObject` → `ResumeThread`，最后 `TerminateJobObject` 一次性清干净
- ★ 反向注入：若目标是 `mysqltest.exe`，会追加 `--safe-process-pid=<pid>`，让客户端能自己通知守护进程

MySQL 测试里最麻烦的问题是**孤儿 mysqld 进程**——测试崩溃后 mysqld 还在跑，占用端口导致后续测试失败。

`My::SafeProcess` 的做法是：所有子进程由一个**守护进程**（`safe_process`）启动并监护，父进程死了守护进程会确保子进程也被清理。这是一个典型的"用监护进程解决孤儿进程"的方案。

### 四、mysqltest 测试语言与命令表

> `client/mysqltest.cc`。

#### 4.1 命令是怎么被识别的

⚠️ **勘误**：命令表**不叫 `commands[]`**，且**不存在 `client/mysqltest.h`**（头文件已拆到 `client/mysqltest/` 下 7 个）。真身是**三件套**：`enum enum_commands`（`:437`）+ `const char *command_names[]`（`:543`）+ `TYPELIB command_typelib`（`:587`）。

```cpp
// client/mysqltest.cc:576
struct st_command {
  char *query, *query_buf, *first_argument, *last_argument, *end;
  DYNAMIC_STRING content;
  size_t first_word_len, query_len;
  bool abort_on_error, used_replace;
  char output_file[FN_REFLEN];
  enum enum_commands type;      // ★ 没有函数指针！分派全靠 main() 的 switch
  uint lineno;
};

TYPELIB command_typelib = {array_elements(command_names), "", command_names,
                           nullptr};
//  ★★ 这就是「复用服务器 TYPELIB」的实证：99 个命令名塞进 TYPELIB，
//     然后直接调服务器侧通用的 find_type() 做大小写不敏感全名匹配
```

```cpp
// :543 —— 完整命令名数组（99 个真实命令 + nullptr 哨兵）
const char *command_names[] = {
    "connection", "query", "connect", "sleep", "inc", "dec", "source",
    "disconnect", "let", "echo", "expr", "while", "end", "save_master_pos",
    "sync_with_master", "sync_slave_with_master", "error", "send", "reap",
    "dirty_close", "replace_result", "replace_column", "ping", "eval",
    "enable_query_log", "disable_query_log",
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
```

> ★ 顺序与 `enum enum_commands` **严格一一对应**。`Q_CONNECTION = 1` 是刻意的——`find_type()` 返回「数组下标 + 1」，0 表示未找到。

**查找函数真名是 `get_command_type()`**（`:9187`），**不存在 `find_command()`**：

```cpp
static void get_command_type(struct st_command *command) {
  if (*command->query == '}') { command->type = Q_END_BLOCK; return; }

  save = command->query[command->first_word_len];
  command->query[command->first_word_len] = 0;   // ★ 临时截断首单词，避免复制字符串
  type = find_type(command->query, &command_typelib, FIND_TYPE_NO_PREFIX);
  //                                               ★ 不允许缩写，只大小写不敏感全名
  command->query[command->first_word_len] = save;

  if (type > 0) {
    command->type = (enum enum_commands)type;
    if (type == Q_QUERY) { command->query = command->first_argument; }
  } else {
    /* No mysqltest command matched */
    if (command->type != Q_COMMENT_WITH_COMMAND) {
      command->type = Q_QUERY;   // ★★★ 未识别的行 = 直接发给 mysqld 的 SQL
    } else {
      die("Found line '%s' beginning with -- that didn't contain "
          "a valid mysqltest command, ...");
    }
  }
}
```

> ★★ **这就是"mysqltest 文件里裸写 SQL 也能跑"的全部原理**：命令表未命中 → 降级为 `Q_QUERY`。也解释了为什么 `-- foo` 开头的注释若首词恰好像命令会报错。

**分派是 `main()` 里两个 `switch`，不是函数指针表**（`:9742` 与 `:10208`）：

```cpp
  while (!read_command(&command) && !abort_flag) {
    if (command->type == Q_UNKNOWN || command->type == Q_COMMENT_WITH_COMMAND)
      get_command_type(command);
    ...
    switch (command->type) {          // ★ 第一 switch：真正干活的命令
      case Q_CONNECT:      do_connect(command); break;
      case Q_CONNECTION:   select_connection(command); break;
      case Q_LET:          do_let(command); break;
      case Q_SORTED_RESULT:
        display_result_sorted = true; start_sort_column = 0; break;
      case Q_QUERY: case Q_REAP: {
        int flags = QUERY_REAP_FLAG | QUERY_SEND_FLAG;
        if (q_send_flag) { flags = QUERY_SEND_FLAG; ... }
        else if (command->type == Q_REAP) { flags = QUERY_REAP_FLAG; }
        run_query(cur_con, command, flags);
        break;
      }
      case Q_SEND: case Q_SEND_EVAL:
        run_query(cur_con, command, QUERY_SEND_FLAG);   // ★ 只发不收
        break;
      ...
      case Q_EXIT: abort_flag = true; break;   // ★ 不再处理后续命令，但仍走结果比对
    }
    if (!processed) {
      switch (command->type) {        // ★ 第二 switch：只处理块结构
        case Q_WHILE:  do_block(cmd_while, command); break;
        case Q_IF:     do_block(cmd_if, command); break;
        case Q_ASSERT: do_block(cmd_assert, command); break;
        case Q_END_BLOCK: do_done(command); break;
      }
    }
    if (command_executed != last_command_executed || command->used_replace) {
      free_all_replace();             // ★★ replace_result/regex/column 都是「一次性」的
      display_result_sorted = false;
      display_result_lower  = false;
    }
    flush_ds_res();                   // ★ 每条命令产出立即 flush 到 .log
  }
```

> ★★ `sorted_result` / `lowercase_result` / `replace_*` **只对紧跟着的下一条命令生效**——因为它们在每个命令执行后被无条件复位。这是新手最常踩的坑。

**`read_command()` 缓存**（`:7616`）——`while` 循环回跳靠它：

```cpp
static int read_command(struct st_command **command_ptr) {
  if (parser.current_line < parser.read_lines) {
    *command_ptr = q_lines->at(parser.current_line);   // ★ 复用缓存，不重读文件
    start_lineno = (*command_ptr)->lineno;
    return 0;
  }
  ...
  if (*p == '#')                  command->type = Q_COMMENT;
  else if (p[0]=='-' && p[1]=='-') command->type = Q_COMMENT_WITH_COMMAND;
  else if (*p == '\n')             command->type = Q_EMPTY_LINE;
  // 首单词长度：由 ' ' / '(' / delimiter 终止
  while (*p && !my_isspace(charset_info,*p) && *p != '(' && !is_delimiter(p)) p++;
  command->first_word_len = (uint)(p - command->query);
}
```

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

**★ 命令系统的真实结构：只有一张名字表 + 两个 `switch`**（⚠️ **不存在 `commands[]` 分发表，也没有 `static_assert` 校验**）：

```cpp
/* client/mysqltest.cc:432 —— 枚举与名字表的唯一约束就是这两行注释 */
/*
  List of commands in mysqltest
  Must match the "command_names" array
  Add new commands before Q_UNKNOWN!
*/
enum enum_commands {
  Q_CONNECTION = 1,      /* ★★ 从 1 起：find_type 返回 pos+1，0 表示未匹配 */
  Q_QUERY, Q_CONNECT, Q_SLEEP, Q_INC, Q_DEC, Q_SOURCE, Q_DISCONNECT,
  Q_LET, Q_ECHO, Q_EXPR, Q_WHILE, Q_END_BLOCK, Q_SAVE_MASTER_POS,
  Q_SYNC_WITH_MASTER, Q_SYNC_SLAVE_WITH_MASTER, Q_ERROR, Q_SEND, Q_REAP,
  Q_DIRTY_CLOSE, Q_REPLACE, Q_REPLACE_COLUMN, Q_PING, Q_EVAL,
  Q_ENABLE_QUERY_LOG, Q_DISABLE_QUERY_LOG, Q_ENABLE_RESULT_LOG,
  Q_DISABLE_RESULT_LOG, Q_ENABLE_CONNECT_LOG, Q_DISABLE_CONNECT_LOG,
  Q_WAIT_FOR_SLAVE_TO_STOP, Q_ENABLE_WARNINGS, Q_DISABLE_WARNINGS,
  Q_ENABLE_INFO, Q_DISABLE_INFO, Q_ENABLE_SESSION_TRACK_INFO,
  Q_DISABLE_SESSION_TRACK_INFO, Q_ENABLE_METADATA, Q_DISABLE_METADATA,
  Q_ENABLE_ASYNC_CLIENT, Q_DISABLE_ASYNC_CLIENT, Q_EXEC, Q_EXECW,
  Q_EXEC_BACKGROUND, Q_DELIMITER, Q_DISABLE_ABORT_ON_ERROR,
  Q_ENABLE_ABORT_ON_ERROR, Q_DISPLAY_VERTICAL_RESULTS,
  Q_DISPLAY_HORIZONTAL_RESULTS, Q_QUERY_VERTICAL, Q_QUERY_HORIZONTAL,
  Q_SORTED_RESULT, Q_PARTIALLY_SORTED_RESULT, Q_LOWERCASE,
  Q_SKIP_IF_HYPERGRAPH, Q_START_TIMER, Q_END_TIMER, Q_CHARACTER_SET,
  Q_DISABLE_PS_PROTOCOL, Q_ENABLE_PS_PROTOCOL, Q_DISABLE_RECONNECT,
  Q_ENABLE_RECONNECT, Q_IF, Q_DISABLE_TESTCASE, Q_ENABLE_TESTCASE,
  Q_REPLACE_REGEX, Q_REPLACE_NUMERIC_ROUND, Q_REMOVE_FILE, Q_FILE_EXIST,
  Q_WRITE_FILE, Q_COPY_FILE, Q_PERL, Q_DIE, Q_ASSERT, Q_EXIT, Q_SKIP,
  Q_CHMOD_FILE, Q_APPEND_FILE, Q_CAT_FILE, Q_DIFF_FILES, Q_SEND_QUIT,
  Q_CHANGE_USER, Q_MKDIR, Q_RMDIR, Q_FORCE_RMDIR, Q_FORCE_CPDIR,
  Q_LIST_FILES, Q_LIST_FILES_WRITE_FILE, Q_LIST_FILES_APPEND_FILE,
  Q_SEND_SHUTDOWN, Q_SHUTDOWN_SERVER, Q_RESULT_FORMAT_VERSION, Q_MOVE_FILE,
  Q_REMOVE_FILES_WILDCARD, Q_COPY_FILES_WILDCARD, Q_SEND_EVAL, Q_OUTPUT,
  Q_RESET_CONNECTION, Q_QUERY_ATTRIBUTES,
  Q_UNKNOWN,                /* 100 —— 以下 4 个在 command_names 里没有名字 */
  Q_COMMENT,                /* 101 */
  Q_COMMENT_WITH_COMMAND,   /* 102 */
  Q_EMPTY_LINE              /* 103 */
};
```

```cpp
/* client/mysqltest.cc:543 —— 99 个真实命令名 + nullptr 哨兵 */
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

/* :587 —— TYPELIB 初始化（★ 注意没有常见的 -1）*/
TYPELIB command_typelib = {array_elements(command_names), "", command_names,
                           nullptr};
```

> ★★ **`array_elements(command_names)` = 100（含 nullptr 哨兵），没有 `-1`**。能工作是因为服务器侧 `find_type()` 的循环终止条件是 `(j = typelib->type_names[pos])`（遇 nullptr 停），`count` 只用于 `if (!typelib->count) return 0`。这是"复用服务器 TYPELIB 做命令解析"能成立的细节基础。
> ★ **`Q_CONNECTION = 1` 是刻意的**——`find_type` 返回 `pos+1`，`0` 表示未匹配，所以枚举必须从 1 起。
> ★ `Q_UNKNOWN` / `Q_COMMENT` / `Q_COMMENT_WITH_COMMAND` / `Q_EMPTY_LINE` 在 `command_names` 里**没有对应名字**，由词法层直接赋值，不经过 TYPELIB。

**★ 命令识别的三步**（★ 真名不是 `find_command`）：

```cpp
/* ① read_command() —— 读一行、切首词、判定注释/空行 */
  if (*p == '#')                        command->type = Q_COMMENT;
  else if (p[0] == '-' && p[1] == '-')  command->type = Q_COMMENT_WITH_COMMAND; p += 2;
  else if (*p == '\n')                  command->type = Q_EMPTY_LINE;
  ...
  // Calculate first word length(the command), terminated
  // by 'space' , '(' or 'delimiter'
  p = command->query;
  while (*p && !my_isspace(charset_info, *p) && *p != '(' && !is_delimiter(p)) p++;
  command->first_word_len = (uint)(p - command->query);

/* ② get_command_type() —— 临时截断成 C 串后查 TYPELIB */
static void get_command_type(struct st_command *command) {
  if (*command->query == '}') { command->type = Q_END_BLOCK; return; }
  save = command->query[command->first_word_len];
  command->query[command->first_word_len] = 0;                 /* ★ 临时置 0 */
  type = find_type(command->query, &command_typelib, FIND_TYPE_NO_PREFIX);
  command->query[command->first_word_len] = save;              /* ★ 立刻还原 */
  if (type > 0) {
    command->type = (enum enum_commands)type;
    if (type == Q_QUERY) command->query = command->first_argument;  /* 跳过 "query" 本身 */
  } else {
    /* No mysqltest command matched */
    if (command->type != Q_COMMENT_WITH_COMMAND) {
      command->type = Q_QUERY;        /* ★★ 匹配不到就当作 SQL 发给 mysqld */
    } else {
      die("Found line '%s' beginning with -- that didn't contain "
          "a valid mysqltest command, check your syntax or "
          "use # if you intended to write a comment", command->query);
    }
  }
}
```

> ★★ **`FIND_TYPE_NO_PREFIX` 表示不允许前缀缩写，必须全名匹配**。
> ★ **"裸 SQL 无需前缀"的实现点就在 `type == 0 → Q_QUERY` 这一行**；反过来，以 `--` 开头的行若首词不是合法命令则**直接报错**（因为已经声明"这是注释"却又解析不出命令）。
> ★ 首词终止符是**空白 / `(` / delimiter** 三者之一——所以 `connect(...)` 的 `(` 也能正确切词。

**★ 分发：主 switch（9742–10203）+ 块命令 switch（10208–10224）**（⚠️ 没有 `commands[]` 数组）：

```cpp
    switch (command->type) {
      case Q_CONNECTION: select_connection(command); break;   /* ★ 真名不是 do_connection */
      case Q_QUERY: case Q_REAP: { ... }                       /* ★ 共用分支，靠 flags 区分 */
      case Q_SEND: case Q_SEND_EVAL: { ... }
      case Q_SORTED_RESULT:            display_result_sorted = true; start_sort_column = 0; break;
      case Q_PARTIALLY_SORTED_RESULT:  display_result_sorted = true;
                                       start_sort_column = atoi(command->first_argument); break;
      case Q_LOWERCASE:                display_result_lower = true; break;
      case Q_SKIP_IF_HYPERGRAPH:       skip_if_hypergraph = true; break;
      case Q_DIE:                      die("%s", command->first_argument); break;
      ...
    }
    /* 块命令（10208 起）*/
    switch (command->type) {
      case Q_IF:     do_block(cmd_if,    command); break;
      case Q_WHILE:  do_block(cmd_while, command); break;
      case Q_ASSERT: do_block(cmd_assert, command); break;
      case Q_END_BLOCK: do_done(command); break;              /* ★ 真名不是 do_end */
    }
```

```cpp
/* 一次性开关的统一复位（10235）*/
    if (command_executed != last_command_executed || command->used_replace) {
      /* As soon as any command has been executed, the replace structures should be cleared */
      free_all_replace();
      /* Also reset "sorted_result", "lowercase" and "skip_if_hypergraph"*/
      display_result_sorted = false;
      display_result_lower = false;
      skip_if_hypergraph = false;
    }
```

> ★★ **`sorted_result` / `lowercase_result` / `skip_if_hypergraph` / 所有 `replace_*` 都是"只影响下一条语句"的一次性开关**，靠主循环末尾统一复位——这解释了为什么它们必须写在目标语句**紧前面**。

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

**★ 逐个的实现要点**（⚠️ **大量 `do_xxx()` 名字是误解——真正存在的很少**）：

```cpp
/* send/reap 的本质：两个 flag + 一个 pending 标志位 */
#define QUERY_SEND_FLAG 1
#define QUERY_REAP_FLAG 2
struct st_connection { ...
  /* Set after send to disallow other queries before reap */
  bool pending;
};
```

```cpp
/* run_query_normal() —— 异步两半在这里被干净切开（8615-8618）*/
  if (flags & QUERY_SEND_FLAG) {
    if (mysql_send_query_wrapper(&cn->mysql, query, (ulong)query_len)) { ...handle_error... goto end; }
  }
  if (!(flags & QUERY_REAP_FLAG)) {
    cn->pending = true;
    return;              /* ★★ 只有 SEND → 发完直接返回，不等结果 */
  }
  do {
    if ((counter == 0) && mysql_read_query_result_wrapper(&cn->mysql)) { ... }
    if (mysql_field_count(mysql) && ((res = mysql_store_result_wrapper(mysql)) == nullptr)) { ... }
    ...
  } while (!(error = mysql_next_result_wrapper(mysql)));
end:
  cn->pending = false;
  var_set_errno(mysql_errno(mysql));    /* ★ 顺带把错误码写进内建变量 $mysql_errno */
```

```cpp
/* reap 与 query 共用同一个 case，只是 flags 不同 —— 这就是没有 do_send/do_reap 的原因 */
        case Q_QUERY:
        case Q_REAP: {
          int flags = QUERY_REAP_FLAG | QUERY_SEND_FLAG;   /* 默认：发 + 收 */
          if (q_send_flag) { flags = QUERY_SEND_FLAG; ... q_send_flag = 0; }
          else if (command->type == Q_REAP) { flags = QUERY_REAP_FLAG; }   /* ★ 只收 */
          run_query(cur_con, command, flags);
        }
        case Q_SEND:
        case Q_SEND_EVAL:
          if (!*command->first_argument) {
            /* 不带参数的 send：影响的是"下一条"语句 */
            if (command->type == Q_SEND) q_send_flag = 1; else q_send_flag = 2;
            break;
          }
          run_query(cur_con, command, QUERY_SEND_FLAG);
```

> ★★ **`send` 不带参数时置 `q_send_flag`，影响的是下一条语句**——这是 `send` 两种形态（带参/不带参）语义不同的地方。
> ★ `run_query()` 开头的两道守卫：`if (cn->pending && (flags & QUERY_SEND_FLAG)) die("Cannot run query on connection between send and reap")` 与 `if (!(flags & QUERY_SEND_FLAG) && !cn->pending) die("Cannot reap on a connection without pending send")`。

```cpp
/* do_error() —— 四种写法：S42000(SQLSTATE) / ER_XXX、CR_XXX(符号名) / 纯数字 / $var */
  if (error.front() == 'S') {
    if (error.length() != SQLSTATE_LENGTH) die("The sqlstate must be exactly %d chars long.", SQLSTATE_LENGTH);
    for (...) if (!my_isdigit(...) && !my_isupper(...))
      die("The sqlstate may only consist of digits[0-9] and _uppercase_ letters.");
    expected_errors->add_error(error.c_str(), ERR_SQLSTATE);
  } else if (error.front() == 's')  die("The sqlstate definition must start with an uppercase S.");
  else if (error.front() == 'C' || error.front() == 'E') {
    int error_code = get_errcode_from_name(error);   /* ★ 查 mysqld 的 errmsg 表 */
    if (error_code == -1) die("Unknown SQL error name '%s'.", error.c_str());
    expected_errors->add_error(error_code, ERR_ERRNO);
  } else if (error.front() == 'c' || error.front() == 'e') die("...must start with an uppercase C or E.");
  else {
    int error_code = get_int_val(error.c_str());
    if (error_code == -1) die("Invalid argument '%s' to 'error' command, ...");
    expected_errors->add_error((std::uint32_t)error_code, ERR_ERRNO);
  }
  if (expected_errors->count() >= MAX_ERROR_COUNT) die("Too many errorcodes specified.");
```

> ★ **`C` 前缀 = 客户端错误（`CR_xxx`）、`E` 前缀 = 服务端错误（`ER_xxx`）**；小写 `s`/`c`/`e` 会报专门的错（提示必须大写）。

```cpp
/* replace_result 的实现：一棵 256 路 trie，多模式串一次性并行替换 */
struct REPLACE {
  int found;
  REPLACE *next[256];        /* ★★ 每字节一个 next[] 槽（Aho-Corasick 风格状态机） */
};
struct REPLACE_STRING { int found; const char *replace_string; uint to_offset; int from_offset; };

void do_get_replace(struct st_command *command) {
  ...
  for (i = 1, pos = word_end_chars; i < 256; i++)
    if (my_isspace(charset_info, i)) *pos++ = i;      /* ★ 用于"整词匹配"边界判断 */
  *pos = 0;
  if (!(glob_replace = init_replace(from_array.typelib.type_names,
                                    to_array.typelib.type_names,
                                    (uint)from_array.typelib.count, word_end_chars)))
    die("Can't initialize replace from '%s'", command->query);
}
```

> ⚠️ **真名带 `get`**：`do_get_replace` / `do_get_replace_column` / `do_get_replace_regex` / `do_get_replace_numeric_round`。且这个结构叫 `REPLACE`（`st_replace` 不存在）。

```cpp
/* replace_column：列号 → 替换串的稀疏数组，列号从 1 起 */
static char *replace_column[MAX_COLUMNS];
static uint max_replace_column = 0;
  my_free(replace_column[column_number - 1]);
  replace_column[column_number - 1] = my_strdup(..., to, ...);
  max_replace_column = std::max(max_replace_column, column_number);

/* replace_regex：乒乓双缓冲，避免反复 malloc/free */
struct st_replace_regex {
  Prealloced_array<st_regex, 128> regex_arr;   /* ★ 预分配 128 个槽 */
  char *buf{nullptr};
  char *even_buf{nullptr};  char *odd_buf{nullptr};   /* ★ 每条规则替换后切换角色 */
};

/* replace_numeric_round：浮点抖动归一化，精度 0..16 */
  if (str2int(ds_round.str, 10, 0, REPLACE_ROUND_MAX, &v) == NullS)
    die("A number between 0 and %d is required for the precision in replace_numeric_round", REPLACE_ROUND_MAX);
  /* 关键细节：若实际小数位 < 请求精度，则按实际位数取，避免 1.2 变成 1.2000000 */
  if (size1 < (size_t)r) r = size1;
```

```cpp
/* ps_protocol：复用通用 set_property，disable 时额外关掉 stmt */
        case Q_DISABLE_PS_PROTOCOL:
          set_property(command, P_PS, false);
          close_statements();      /* ★★ 必须显式关闭已 prepare 的 statement */
          break;
```

```cpp
/* skip_if_hypergraph：真正的"跳过"在 run_query 里提前 return */
  if (skip_if_hypergraph && opt_hypergraph) {
    constexpr char message[] =
        "<ignored hypergraph optimizer error: statement skipped by test>\n";
    dynstr_append_mem(&ds_res, message, strlen(message));   /* ★ 写固定占位串保证可比对 */
    return;
  }
```

```cpp
/* source：★ 它本身不执行任何语句，只是把新文件压入文件栈 */
static void do_source(struct st_command *command) {
  ...
  /* If this file has already been sourced, don't source it again.
     It's already available in the q_lines cache. */
  if (parser.current_line < (parser.read_lines - 1))
    ; /* Do nothing */
  else {
    open_file(ds_filename.str);     /* ★ 压栈 */
  }
}
static struct st_test_file file_stack[16];   /* ★★ .inc 嵌套最多 16 层 */
```

> ★ `open_file()` 的查找顺序：**当前文件所在目录 → `--basedir` 相对路径**。

```cpp
/* if / while / assert 共用 do_block()，差异只在末尾 */
  if (cmd == cmd_assert) {
    if (!v_val_bool) die("Assertion failed: %s", command->query);   /* ★ assert 不入栈 */
  } else {
    cur_block++; cur_block->cmd = cmd; cur_block->ok = v_val_bool;
  }

/* while 的"倒带"就一行（do_done）*/
  if (cur_block->ok && cur_block->cmd == cmd_while) {
    cur_block--;
    parser.current_line = cur_block->line;    /* ★★ 拨回 while 所在行 */
  } else {
    cur_block--; parser.current_line++;
  }
```

> ★★ **`while` 循环靠 `q_lines` 缓存 + `parser.current_line` 拨回实现"重放"，无需重新读文件**。块栈 `static struct st_block block_stack[32]`（最多 32 层）。
> ★ `assert` 与 `if`/`while` 共用求值逻辑，但**不进块栈**，失败直接 `die`。
> ⚠️ **不存在 `do_if()` / `do_while()` / `do_die()`** —— 前两者是 `do_block()`，`die` 是内联 `die("%s", command->first_argument)`，`end` 的真名是 `do_done()`。

#### 4.4 变量替换

`let $var = <value>` 定义变量，之后 `$var` 在后续语句中被替换。`eval` 用于"把变量展开后再执行"（普通 `query` 不做变量展开）。

这是一个**两级执行**设计：`query` 直接发送原文，`eval` 先做变量替换再发送。

**★ 这个"两级"在源码里就是 `run_query()` 开头的一个 if**：

```cpp
static void run_query(struct st_connection *cn, struct st_command *command, int flags) {
  ...
  bool complete_query = ((flags & QUERY_SEND_FLAG) && (flags & QUERY_REAP_FLAG));
  ...
  /*
    Evaluate query if this is an eval command
  */
  if (command->type == Q_EVAL || command->type == Q_SEND_EVAL) {
    init_dynamic_string(&eval_query, "", command->query_len + 256);
    do_eval(&eval_query, command->query, command->end, false);   /* ★★ eval：先展开 */
    query = eval_query.str; query_len = eval_query.length;
  } else {
    query = command->query; query_len = std::strlen(query);      /* ★★ query：直接用原文 */
  }
```

**★ `do_eval()` 是单遍扫描的替换引擎**（★ 有个 8.0 特有的坑）：

```cpp
void do_eval(DYNAMIC_STRING *query_eval, const char *query,
             const char *query_end, bool pass_through_escape_chars) {
  for (p = query; (c = *p) && p < query_end; ++p) {
    next_c = *(p + 1);
    switch (c) {
      case '$':
        if (escaped ||
            // a JSON path expression
            next_c == '.' || next_c == '[' || next_c == '\'' || next_c == '"') {
          escaped = 0;
          dynstr_append_mem(query_eval, p, 1);       /* ★★ JSON 路径 $.x / $[0] 原样保留 */
        } else {
          if (!(v = var_get(p, &p, false, false))) die("Bad variable in eval");
          dynstr_append_mem(query_eval, v->str_val, v->str_val_len);
        }
        break;
      case '\\':
        if (escaped) { escaped = 0; dynstr_append_mem(query_eval, p, 1); }
        else if (next_c == '\\' || next_c == '$' || next_c == '"') {
          escaped = 1;                                /* ★ \$ 转义为字面 $ */
          if (pass_through_escape_chars) dynstr_append_mem(query_eval, p, 1);
        } else dynstr_append_mem(query_eval, p, 1);
        break;
      default: escaped = 0; dynstr_append_mem(query_eval, p, 1); break;
    }
  }
}
```

> ★★ **`$` 后面跟 `.` `[` `'` `"` 时被视为 JSON 路径表达式，原样保留不替换**——写 `SELECT JSON_EXTRACT(x, '$[0]')` 时不会误替换，但反过来也意味着**紧跟 JSON 符号的 `$var` 不会被展开**。
> ★ `pass_through_escape_chars` 由 `do_exec` 传 `!is_windows`——Windows 上丢掉反斜杠、非 Windows 上保留给 shell。

**★ 变量的二层存储与惰性同步**（⚠️ **`var_fill()` 不存在**）：

```cpp
typedef struct {
  char *name; size_t name_len;
  char *str_val; size_t str_val_len;
  int int_val; size_t alloced_len;
  bool int_dirty;   /* do not update string if int is updated until first read */
  bool is_int;
  bool alloced;
} VAR;

/*Perl/shell-like variable registers */
VAR var_reg[10];                                        /* ★ $0..$9 */
collation_unordered_map<std::string, std::unique_ptr<VAR, var_free>> *var_hash;  /* ★ $name */
```

> ★ `var_reg[10]` 是 Perl 风格的 `$0`..`$9`（也用于正则捕获），`var_hash` 是具名变量。
> ★★ **`int_dirty` 实现 int 与 str 的惰性同步**：`inc`/`dec` 只改 `int_val` 并置 `int_dirty`，字符串形式**等到下次 `var_get` 才 `sprintf` 回写**。
> ★ **未定义的变量自动回落到环境变量**：`var_get()` 里 `find_or_nullptr(*var_hash, name)` 失败就 `var_from_env(buff, "")`（调 `getenv`）——这就是 `$MYSQLTEST_VARDIR` 之类能用的原因。
> ★ `var_set()` 里**不带 `$` 前缀**的名字会走 `setenv()`——mysqltest 变量与环境变量双向互通。

**★ `eval_expr()` 是右值的"万能求值器"**（四种形态）：

```cpp
void eval_expr(VAR *v, const char *p, const char **p_end, bool open_end, bool do_eval) {
  if (!do_eval) goto NO_EVAL;
  if (*p == '$') { if ((vp = var_get(p, p_end, false, false))) var_copy(v, vp); ... return; }  /* ① 拷贝变量 */
  if (*p == '`') { var_query_set(v, p, p_end); return; }                                        /* ② 执行 SQL 取值 */
  {
    if (parse_function("query_get_value", var_set_query_get_value)) return;
    if (parse_function("convert_error",   var_set_convert_error))   return;
    if (parse_function("escape",          var_set_escape))          return;                     /* ③ 三个内建函数 */
  }
NO_EVAL : { ...原样当字符串... }                                                                 /* ④ 纯字符串 */
}
```

> ⚠️ **`expr` 只支持二元运算**（一个运算符、两个操作数），不是通用表达式求值器；全部按 `double` 计算，位运算强转 `int`。支持的运算符：`+ - * / % && || & | ^ >> <<`。
> ★ `let` 的右值**总是**先 `do_eval` 展开，再 `var_set`（内部还会再调一次 `eval_expr`）——所以 `let $a = $b` 是"展开后赋值"。

### 五、结果比对与 --record

#### 5.1 比对流程

⚠️ **勘误**：**Perl 侧不做 diff**。它只删旧 `.reject`；比对、`--record` 落盘、`.reject` 生成**全在 `client/mysqltest.cc`**。

**两条出口**（`main()` 尾部，`:10255`）：

```cpp
  if (log_file.bytes_written()) {
    if (result_file_name) {
      if (record) {
        /* Recording */
        if (my_copy(log_file.file_name(), result_file_name, MYF(0)) != 0)
          die("Failed to copy '%s' to '%s', errno: %d", ...);
        //  ★★ --record 的真相：把 .log 直接 copy 成 .result
      } else {
        check_result();     // ★ 非 record 模式：比对
      }
    }
  } else {
    /* Empty output is an error *unless* we also have an empty result file */
    if (!result_file_name || record ||
        compare_files(log_file.file_name(), result_file_name)) {
      die("The test didn't produce any output");
    } else { empty_result = true; }
  }
  ...
  cleanup_and_exit(0);    // ★ 只有这一条成功出口；任何 die() 都是非 0
```

**`check_result()`（`:2193`）——`.reject` 的唯一来源**：

```cpp
static void check_result() {
  const char *mess = "Result content mismatch\n";
  switch (compare_files(log_file.file_name(), result_file_name)) {
    case RESULT_OK:
      break;
    case RESULT_LENGTH_MISMATCH:
      mess = "Result length mismatch\n";
      [[fallthrough]];
    case RESULT_CONTENT_MISMATCH: {
      char reject_file[FN_REFLEN];
      fn_format(reject_file, result_file_name, opt_logdir, ".reject",
                MY_REPLACE_DIR | MY_REPLACE_EXT);
      if (my_copy(log_file.file_name(), reject_file, MYF(0)) != 0)
        die("Failed to copy '%s' to '%s', errno: %d", ...);
      //  ★★ .reject = 实际输出 .log 的拷贝，放在 --logdir 下
      bool ignored_diff = show_diff(nullptr, result_file_name, reject_file);
      if (ignored_diff) {
        abort_not_supported_test("Hypergraph optimizer did not support all queries.");
      }
      die("%s", mess);
      break;
    }
  }
}
```

**`compare_files2()`（`:2118`）——比对是整文件 memcmp，不是逐行 diff**：

```cpp
static int compare_files2(File fd, const char *filename2) {
  char buff[512], buff2[512];
  while ((len = my_read(fd, (uchar *)&buff, sizeof(buff), MYF(0))) > 0) {
    if ((len2 = my_read(fd2, (uchar *)&buff2, sizeof(buff2), MYF(0))) < len)
      { error = RESULT_LENGTH_MISMATCH; break; }
    if (len2 > len) { error = RESULT_LENGTH_MISMATCH; break; }
    if ((memcmp(buff, buff2, len))) { error = RESULT_CONTENT_MISMATCH; break; }
  }
  if (!error && my_read(fd2, (uchar *)&buff2, sizeof(buff2), MYF(0)) > 0)
    error = RESULT_LENGTH_MISMATCH;
  return error;
}
```

> ★★ **512 字节一块的二进制块比对**。所谓"逐行"其实是错觉——真正的按行处理发生在**生成阶段**（`dynstr_append_sorted` 按行排序、`append_result` 按行拼 `\t`），**比对阶段**纯二进制。所以 result 文件里一个换行差异就会 FAIL。

**四种结果修饰的优先级**（都在**写入时**应用，不在比对时）：

| 顺位 | 命令 | 落点 |
|---|---|---|
| 1（最高） | `--replace_column N str` | `append_field()` `:8218`，直接换掉整列值（连 NULL 也盖掉） |
| 2 | `--lowercase_result` | `replace_dynstr_append_mem()` 最先生效 |
| 3 | `--replace_regex` | `multi_reg_replace()` |
| 4 | `--replace_result` / `--replace_numeric_round` | 256 叉 Trie 一次扫描 |

```cpp
// :11334 —— 统一入口
void replace_dynstr_append_mem(DYNAMIC_STRING *ds, const char *val, size_t len) {
  if (display_result_lower) { my_casedn_str(charset_info, const_cast<char*>(val)); }
  if (glob_replace_regex) {
    if (!multi_reg_replace(glob_replace_regex, const_cast<char*>(val), &len))
      { val = glob_replace_regex->buf; } else { len = orig_len; }
  }
  if (glob_replace) { replace_strings_append(glob_replace, &ds_temp, val, len); }
  if (glob_replace_numeric_round >= 0) { ... }
}

// :8218 —— replace_column 优先级最高
static void append_field(DYNAMIC_STRING *ds, uint col_idx, MYSQL_FIELD *field,
                         char *val, size_t len, bool is_null) {
  if (col_idx < max_replace_column && replace_column[col_idx]) {
    val = replace_column[col_idx];     // ★ 先换列值
    len = std::strlen(val);
  } else if (is_null) { val = null; len = 4; }
  ...
  replace_dynstr_append_mem(ds, val, len);   // ★ 换完仍会再过一遍 regex / replace_result
}
```

**`--sorted_result` 的真身是 `dynstr_append_sorted()`**（`:11440`），**不存在 `reorder_result()` / `sort_result()`**：

```cpp
void dynstr_append_sorted(DYNAMIC_STRING *ds, DYNAMIC_STRING *ds_input,
                          int start_sort_column) {
  char *start = ds_input->str;
  char *end = ds_input->str + ds_input->length;

  /* First line is result header, skip past it */
  while (*start && *start != '\n') start++;
  start++;
  dynstr_append_mem(ds, ds_input->str, start - ds_input->str);  // ★ 表头不参与排序

  while (start < end) {
    char *line_end = (char *)start;
    while (*line_end != '\n') line_end++;
    *line_end = 0;
    std::string result_row = std::string(start, line_end - start);
    if (!sorted.empty() && start_sort_column > 0) {
      // --partially_sorted_result：按前 N 列分组，组内排序
      size_t prev_line_prefix_len =
          length_of_n_first_columns(sorted.back(), start_sort_column);
      if (sorted.back().compare(0, prev_line_prefix_len, result_row, 0,
                                prev_line_prefix_len) != 0) {
        std::sort(sorted.begin() + first_unsorted_row, sorted.end());
        first_unsorted_row = sorted.size();
      }
    }
    sorted.push_back(result_row);
    start = line_end + 1;
  }
  std::stable_sort(sorted.begin() + first_unsorted_row, sorted.end());
  //  ★ 稳定排序：相同行保持原相对顺序
  for (auto i : sorted) { dynstr_append_mem(ds, i.c_str(), i.length());
                          dynstr_append(ds, "\n"); }
}
```

**`--replace_regex` 用的是 `std::regex`，不是 `class Reg`**（`client/mysqltest/regular_expressions.cc:152`）：

```cpp
int multi_reg_replace(struct st_replace_regex *r, char *val, size_t *len) {
  ...
  for (i = 0; i < r->regex_arr.size(); i++) {
    std::regex rpat(re.pattern, (re.icase == 0)
                                    ? std::regex_constants::ECMAScript
                                    : std::regex_constants::icase);
    std::string sin = std::string(in_buf, *len);
    std::string sout;
    std::regex_replace(std::back_inserter(sout), sin.begin(), sin.end(),
                       rpat, re.replace, std::regex_constants::format_sed);
    if (sout.compare(sin) != 0) {
      ...
      std::swap(in_buf, out_buf);   // ★ 乒乓缓冲：奇数轮 even→odd，偶数轮 odd→even
      buf_len_p = (out_buf == r->even_buf) ? &r->even_buf_len : &r->odd_buf_len;
    }
  }
  return (r->buf == nullptr);
}
```

> ★ `/from/to/i` 语法解析见 `init_replace_regex()`（`:10665`）：`/` 分隔，尾随 `i` = 大小写不敏感，`\/` 转义。

**`send` / `reap` 不是独立函数，是 flags 位**（`:121`）：

```cpp
#define QUERY_SEND_FLAG 1
#define QUERY_REAP_FLAG 2
```

```cpp
// :8596 run_query_normal() —— 协议层异步能力的暴露点
  if (flags & QUERY_SEND_FLAG) {
    if (mysql_send_query_wrapper(&cn->mysql, query, (ulong)query_len)) { ... }
  }
  if (!(flags & QUERY_REAP_FLAG)) {
    cn->pending = true;      // ★ send：发完就返回
    return;
  }
  do {
    if ((counter == 0) && mysql_read_query_result_wrapper(&cn->mysql)) { ... }
    //  ★ reap：这里才真正读结果
    ...
  } while (!(error = mysql_next_result_wrapper(mysql)));
```

```cpp
// :8949 run_query() 开头的状态机守卫
  if (cn->pending && (flags & QUERY_SEND_FLAG))
    die("Cannot run query on connection between send and reap");
  if (!(flags & QUERY_SEND_FLAG) && !cn->pending)
    die("Cannot reap on a connection without pending send");
```

> ★★ **这就是"暴露协议层异步能力"的实质**：`mysql_send_query` + `mysql_read_query_result` = 把 `mysql_real_query` 劈成两半。中间的等待窗口由测试作者用 `sleep` 或另一个连接的操作填充 —— **复现锁竞争、死锁、并发读写**的基础设施。
>
> ⚠️ **`--md5` / `init_md5` 在 8.0.39 已彻底移除**（全仓 0 命中）。
>
> ★ `send_quit` ≠ `disconnect`：前者发 `COM_QUIT` 让**服务端**关连接；后者是**客户端**侧关 socket（`dirty_close` 是它的粗暴变体）。

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

**★ 归一化发生在"输出生成期"而不是"比对期"**（★ 这是最容易被误解的一点）：

```cpp
/* client/mysqltest.cc:11337 —— 所有进日志的输出都过这个函数 */
void replace_dynstr_append_mem(DYNAMIC_STRING *ds, const char *val, size_t len) {
#ifdef _WIN32
  fix_win_paths(val, len);                          /* ① 平台路径统一 */
#endif
  if (display_result_lower)
    my_casedn_str(charset_info, const_cast<char *>(val));   /* ② lowercase_result */
  if (glob_replace_regex)
    if (!multi_reg_replace(glob_replace_regex, const_cast<char *>(val), &len))
      val = glob_replace_regex->buf;                 /* ③ replace_regex（时间/线程 id/行号） */
  if (glob_replace)
    replace_strings_append(glob_replace, &ds_temp, val, len);   /* ④ replace_result */
  if (glob_replace_numeric_round >= 0)
    replace_numeric_round_append(...);               /* ⑤ replace_numeric_round */
}
```

而**比对本身是纯 `memcmp` 分块比较，零归一化**：

```cpp
static int compare_files2(File fd, const char *filename2) {
  while ((len = my_read(fd, (uchar *)&buff, sizeof(buff), MYF(0))) > 0) {
    if ((len2 = my_read(fd2, (uchar *)&buff2, sizeof(buff2), MYF(0))) < len) { error = RESULT_LENGTH_MISMATCH; break; }
    if (len2 > len)                                                        { error = RESULT_LENGTH_MISMATCH; break; }
    if ((memcmp(buff, buff2, len)))                                        { error = RESULT_CONTENT_MISMATCH; break; }
  }
}
```

> ★★ 三层要分清：**生成期归一化**（mysqltest 的 `replace_dynstr_append_mem`）→ **比对期逐字节 memcmp** → **失败期产出 `.reject` + diff**。MTR 的 Perl 侧**不参与任何归一化**。

**★ `.reject` 由 mysqltest 自己生成**（⚠️ 不是 Perl 侧），路径受 `--logdir` 控制：

```cpp
      /* Put reject file in opt_logdir */
      fn_format(reject_file, result_file_name, opt_logdir, ".reject",
                MY_REPLACE_DIR | MY_REPLACE_EXT);      /* ★ 同名换扩展名，落到 --logdir */
      if (my_copy(log_file.file_name(), reject_file, MYF(0)) != 0)
        die("Failed to copy '%s' to '%s', errno: %d", log_file.file_name(), reject_file, errno);
      bool ignored_diff = show_diff(nullptr, result_file_name, reject_file);
```

**★ `show_diff()` 的 diff 工具降级链**（★ 三级降级 + 一个 hypergraph 特例）：

```cpp
      exit_code = run_tool(diff_name, &ds_diff, "-u", filename1, filename2, "2>&1", nullptr);
      if (exit_code > 1) {
        exit_code = run_tool(diff_name, &ds_diff, "-c", ...);          /* 降级 2：context diff */
        if (exit_code > 1) {
          exit_code = run_tool(diff_name, &ds_diff, filename1, filename2, ...);  /* 降级 3：plain diff */
          if (exit_code > 1) diff_name = nullptr;
        }
      } else if (exit_code == 1 && opt_hypergraph &&
                 is_diff_clean_except_hypergraph(&ds_diff)) {
        dynstr_free(&ds_diff);
        return true;      /* ★★ hypergraph 下"只有 hypergraph 差异"视为可忽略 */
      }
  if (!diff_name) {
    /* Fallback to dump both files to result file and inform about installing "diff". */
    ...cat_file(&ds_diff, filename1); cat_file(&ds_diff, filename2);
  }
```

> ⚠️ **`dyn_string_cmp` 已废弃**——只剩 `check_result()` 末尾 `default:` 分支里的一句历史错误文案 `"Unknown error code from dyn_string_cmp()"`。

**★★ 重试机制（`--retry`）是 MTR 对抗 flaky 的唯一内建手段**，但它**完全在主进程**（不在 worker）：

```perl
# 默认值（mysql-test-run.pl:152-153）
my $opt_retry         = 3;                                    # 账面"最多尝试次数"
my $opt_retry_failure = env_or_val(MTR_RETRY_FAILURE => 2);   # 失败次数达到即停

# worker 侧只负责累加失败计数（:6185）
  $tinfo->{'result'} = 'MTR_RES_FAILED';
  my $test_failures = $tinfo->{'failures'} || 0;
  $tinfo->{'failures'} = $test_failures + 1;   /* ★ tinfo 随 TESTRESULT 回传并被原样重发，故计数累加 */

# 主进程侧（run_test_server 内，:1126）
          my $retries         = $result->{retries}  || 2;
          my $test_has_failed = $result->{failures} || 0;
          if ($test_has_failed and $retries <= $opt_retry) {
            if ($opt_retry > 1 and $failures >= $opt_retry_failure) {
              mtr_report("Test $tname has failed $failures times, no more retries.\n");
            } else {
              mtr_report("Retrying test $tname, attempt($retries/$opt_retry).\n");
              delete($result->{result});            /* ★ 清掉结果让 worker 重跑 */
              $result->{retries} = $retries + 1;
              $result->write_test($sock, 'TESTCASE');   /* ★★ 重试 = 同一 tinfo 再发一次 */
              next;
            }
          }
```

> ★★★ **实推演（默认 `retry=3` / `retry_failure=2`）**：第 1 次失败 → `failures=1`、retries 未定义取 2 → `2<=3` 且 `1<2` → 重发（打印 `attempt(2/3)`），retries=3。第 2 次失败 → `failures=2`、retries=3 → `3<=3` 成立，但 `failures(2) >= retry_failure(2)` → 打印 "no more retries" 停止。**所以默认实际最多跑 2 次，不是 3 次**——想跑满 3 次需显式 `--retry-failure=3`。

**★ unstable 判定只在收尾且只在 `--report-unstable-tests` 下**（`mtr_report.pm:575`）：

```perl
      if ($::opt_report_unstable_tests and defined $tinfo->{retries}) {
        my $num_passed = $tinfo->{retries} - $tinfo->{failures} - 1;   /* ★ 成功次数 = 总尝试 - 失败 - 1 */
        /* Tests which exhibit both passing and failing behaviour with the same
           code are unstable tests. ... */
        if ($num_passed > 0) {
          $tot_unstable++;
          $tinfo->{'result'} = 'MTR_RES_UNSTABLE';      /* ★★ MTR_RES_UNSTABLE 的唯一产生点 */
        }
      }
```

> ⚠️ **`MTR_RES_*` 在 8.0.39 里不是常量，是散落各处的裸字符串**，且只有 **4 个值**：`PASSED` / `FAILED` / `SKIPPED` / `UNSTABLE`。`lib/mtr_results.pm` 里**没有任何** `MTR_RES_*`——它只导出 `resfile_*` 系列（结果文件读写）。
> ★ **真正的五个 `use constant` 是 mysqltest 客户端的进程退出码**（`mysql-test-run.pl:362`）：`MYSQLTEST_PASS=0 / FAIL=1 / SKIPPED=62 / NOSKIP_PASS=63 / NOSKIP_FAIL=64`。
> ★ 全是 unstable 时只警告不算硬失败：`if ($tot_failed == $tot_unstable) mtr_warning("There are failures due to unstable test cases...")`。

### 六、用例发现与调度

> `mtr_cases.pm`。

#### 6.1 发现

⚠️ **勘误**：`collect_test_cases()` **不在 `mysql-test-run.pl`**，定义在 **`lib/mtr_cases.pm:412`**（`@EXPORT` 导出，`main()` 的 `:635` 调用）。

```perl
sub collect_test_cases ($$$$) {
  ...
  $do_test_reg   = init_pattern($do_test,   "--do-test");   # ★ 纯字面量时自动补 ^ 前缀
  $skip_test_reg = init_pattern($skip_test, "--skip-test");

  create_disabled_test_list(\%disabled, $opt_skip_test_list);   # ★ 读 disabled.def

  if ($parallel == 1 or !$threads_support or !$threads_shared_support) {
    foreach my $suite (split(",", $suites)) {
      push(@$cases, collect_one_suite($suite, $opt_cases, $opt_skip_test_list, \%disabled));
      push(@$cases, collect_one_suite("i_" . $suite, ...));    # ★ i_<suite> 是"扩展套件"
    }
  } else {
    foreach my $suite (split(",", $suites)) {
      push(@collect_test_cases_thrds,
           threads->create("collect_one_suite", $suite, ...)); # ★★ 唯一用 Perl threads 的地方
      ...
    }
    foreach my $collect_thrd (threads->list()) {
      push(@$cases, $collect_thrd->join());
    }
  }

  if ($opt_reorder && !$quick_collect) {
    foreach my $tinfo (@$cases) {
      my @criteria = ();
      push(@criteria, "ndb=" . ($tinfo->{'ndb_test'} ? "A" : "B"));
      push(@criteria, $tinfo->{template_path});              # ★ my.cnf 模板
      my $opts = $tinfo->{'master_opt'} ? $tinfo->{'master_opt'} : [];
      push(@criteria, join("!", sort @{$opts}) . "~");       # ★ mysqld 选项串
      push(@criteria, join("!", sort @{ $tinfo->{'slave_opt'} })) if $tinfo->{'slave_opt'};
      push(@criteria, $tinfo->{force_restart} ? "force-restart" : "no-restart");
      $tinfo->{criteria} = join(" ", @criteria);
    }
    @$cases = sort { $a->{criteria} cmp $b->{criteria}; } @$cases;
    #  ★★ 排序后同配置用例相邻 → 配合下面的 reserved 亲和调度，减少 mysqld 重启
  }
  return $cases;
}
```

**扫 `t/` 目录用的是 `opendir/readdir`，不是 File::Find**（`mtr_cases.pm:874`）：

```perl
    opendir(TESTDIR, $testdir) or mtr_error("Can't open dir \"$testdir\": $!");
    foreach my $elem (sort readdir(TESTDIR)) {
      my $tname = mtr_match_extension($elem, 'test');
      next unless defined $tname;
      next if ($do_test_reg and not $tname =~ /$do_test_reg/o);   # --do-test 过滤
      push(@cases, collect_one_test_case($suitedir, $testdir, $resdir,
                                         $suite, $tname, $elem,
                                         $disabled, $suite_opts));
    }
    closedir TESTDIR;
```

> ⚠️ **勘误：`My::Find` 不是 `File::Find` 的重写**。它**没有 `find()`**，只导出 `my_find_bin` / `my_find_dir` / `my_find_file`，做的是「候选路径 × 构建子目录 × 候选文件名」**笛卡尔积 + glob**，用于解决"源码树 / 二进制发布包 / VS-Xcode 多配置构建（Release/Debug/RelWithDebInfo）下可执行文件路径不确定"：
> ```perl
>   push(@extra_dirs, ("Release", "Relwithdebinfo", "Debug")) if @extra_dirs == 0;
>   push(@paths, map { my $p=$_; map { "$p/$_" } @extra_dirs } @paths);   # paths × build_dirs
>   @paths = map { my $p=$_; map { "$p/$_" } @names } @paths;             # × names
>   @paths = map { "$base/$_" } @paths;
>   @paths = map { glob("$_") } @paths;      # ★ 不递归，靠 glob 展开通配
> ```
> 真正的目录树扫描在 `mysql-test-run.pl:536` 用标准库 `File::Find::find(\&get_all_suites, ...)`。

**`disabled.def` 格式**（`mtr_cases.pm:126` 校验）：

```
<suitename>.<testcasename> [@platform|@!platform] : <BUG|WL>#<XXXX> [<comment>]
```

读取时会叠加多个来源：`collections/disabled.def`、`disabled_ndb.def`（ndb 开启时）、`disabled-asan.list` / `disabled-ubsan.list`（sanitize）、`disabled-valgrind.list`（valgrind），以及 internal / cloud 目录下的同名文件。

#### 6.2 调度

**`run_test_server()`（`:939`）是 master 主循环**——注释里明说行协议：

```perl
# The message starts with a codeword, which can be 'TESTRESULT',
# 'START', 'SPENT' or 'VALGREP'.
# ... When a test has been identified, it's sent to the worker
# with a message tagged 'TESTCASE'.
# When all tests are completed ... a message 'BYE' is sent to the worker.

sub run_test_server ($$$) {
  my $s = IO::Select->new();
  $s->add($server);

  while (1) {
    my @ready = $s->can_read(1);          # ★ 1 秒超时 select，主循环心跳
    foreach my $sock (@ready) {
      if ($sock == $server) {
        my $child = $sock->accept();
        $s->add($child);
        print $child "HELLO\n";
      } else {
        my $line = <$sock>;
        if (!defined $line) {             # worker 断开
          $s->remove($sock);
          if (--$childs == 0) { return $completed; }
          next;
        }
        chomp($line);

        if ($line eq 'TESTRESULT') {
          $result = My::Test::read_test($sock);     # ★ 反序列化整个 My::Test 对象
          mtr_report_test($result);
          if ($result->is_failed()) {
            ...  # 保存 var/log/<test>/、找 core、My::CoreDump->show()
            if (!$opt_force) { push(@$completed, $result);
                               return $completed unless $result->{'dont_kill_server'}; }
            elsif ($opt_max_test_fail > 0 and $num_failed_test >= $opt_max_test_fail) {
              mtr_report("Too many tests($num_failed_test) failed! Terminating...");
              return undef;
            }
          }
          # ★ 失败重试：把同一个用例对象再发回同一个 worker
          if ($test_has_failed and $retries <= $opt_retry) {
            $result->{retries} = $retries + 1;
            $result->write_test($sock, 'TESTCASE');
            next;
          }
          push(@$completed, $result);
        } elsif ($line =~ /^SPENT/)   { add_total_times($line); }
        elsif ($line eq 'VALGREP')    { $valgrind_reports = 1; }
        elsif ($line eq 'SRV_CRASH')  { $shutdown_report = 1; }
```

**★ 亲和性调度（减少 mysqld 重启的关键）**：

```perl
        for (my $i = 0 ; $i <= @$tests ; $i++) {
          my $t = $tests->[$i];
          last unless defined $t;

          if (run_testcase_check_skip_test($t)) {
            push(@$completed, splice(@$tests, $i, 1)); redo;
          }
          # ★ 挑出 not_parallel 用例，最后由最后一个 worker 串行跑
          if ($t->{'not_parallel'}) {
            push(@$non_parallel_tests, splice(@$tests, $i, 1)); redo;
          }
          if ($t->{ndb_test} and $num_ndb_tests >= $max_ndb) { next; }
          if (!defined $second_best) { $second_best = $i; }

          if ($opt_reorder and $opt_parallel > 1 and defined $result) {
            my $wid = $result->{worker};
            next if (defined $t->{reserved} and $t->{reserved} != $wid);
            if (!defined $t->{reserved} && defined $t->{criteria}) {
              $t->{criteria} =~ s/force-restart$/no-restart/;
              my $criteria = $t->{criteria};
              my $maxres = (@$tests - $i) / $opt_parallel + 1;
              for (my $j = $i + 1 ; $j <= $i + $maxres ; $j++) {
                last if $tests->[$j]->{criteria} ne $criteria;
                last if $tests->[$j]->{'not_parallel'};
                $tests->[$j]->{reserved} = $wid;
                #  ★★★ 把 criteria 相同（同 my.cnf / 同 mysqld 选项）的用例
                #      "预订"给同一个 worker，这样它就不需要重启 mysqld
              }
            }
          }
          $next = splice(@$tests, $i, 1);
          last;
        }

        if ($next) {
          $next->write_test($sock, 'TESTCASE');    # ★ 下发用例
          $running{ $next->key() } = $next;
        } else {
          if (int(@$non_parallel_tests) > 0 and $completed_wid_count == $opt_parallel) {
            $next = shift @$non_parallel_tests;    # ★ 最后一个 worker 跑串行用例
            $next->write_test($sock, 'TESTCASE');
          } else {
            print $sock "BYE\n";                   # ★ 没活了，让 worker 收尾
            $closed_sock{$sock} = 1;
          }
        }
      }
    }
    if (has_expired($suite_timeout)) { ... return undef; }   # 全局超时
  }
}
```

> ★★ **criteria 字符串 + `reserved` 预订** 是 MTR 能把 `--parallel=8` 跑出接近串行速度的原因：同配置的用例被排在一起并钉在同一个 worker 上，mysqld 只在配置切换时才重启。报告末尾那句 `The servers were restarted N times` 就是这套调度的成绩单。

**报告**（`lib/mtr_report.pm:195`）——失败原因正文来自 `logfile`：

```perl
  if ($result eq 'MTR_RES_FAILED') {
    my $fail = $skip_ignored ? "noskip-${retry}fail" : "${retry}fail";
    ...
    if ($logfile) {
      mtr_report("\n$logfile\n");   # ★★ mysqltest 写的 .reject diff 摘要就在这里打印
    }
    if ($comment) { mtr_report("\n$comment\n"); }
    if (!$logfile and !$comment) {
      mtr_report("\nUnknown result, neither 'comment' or 'logfile' set");
    }
  }
```

> ⚠️ **不存在 `print_test_result()`**——真名 `mtr_report_test()`，另有 `mtr_report_test_passed()`（`:170`）/ `_skipped()`（`:164`）。

**★★ 通信介质是 TCP socket，不是 pipe**（⚠️ 这点常被想错）：

```perl
# :715 —— 主进程建监听 socket（并发度 = --parallel）
  my $server = new IO::Socket::INET(Listen    => $opt_parallel,
                                    LocalAddr => 'localhost',
                                    Proto     => 'tcp',);
# :1320 —— worker 回连
  my $server = new IO::Socket::INET(PeerAddr => 'localhost',
                                    PeerPort => $server_port,
                                    Proto    => 'tcp');
```

```perl
# :772 —— worker 是真 fork 出来的进程（Windows 上另有分支）
  for my $child_num (1 .. $opt_parallel) {
    my $child_pid = My::SafeProcess::Base::_safe_fork();
    if ($child_pid == 0) {
      $server = undef;            # Close the server port in child
      $tests  = {};               # Don't need the tests list in child
      if ($opt_parallel > 1) {
        set_vardir("$opt_vardir/$child_num");     # ★ 每个 worker 独立 var/N、tmp/N
        $opt_tmpdir = "$opt_tmpdir/$child_num";
      }
      run_worker($server_port, $child_num);       # ★ 子进程永不返回
      exit(1);
    }
    $children{$child_pid} = 1;
  }
  my $completed = run_test_server($server, $tests, $opt_parallel);
  exit(0) if $opt_start_exit;                     # ★ --start-and-exit：不等 worker 收尾
  kill("INT", keys(%children));                   # Send Ctrl-C to any children still running
```

**★ 完整协议是 8 个 codeword + 一个 `key= value` 块**（★ 注释里只列了 4 个，实际还有 4 个）：

| codeword | 方向 | 含义 |
|---|---|---|
| `HELLO` | 主 → worker | 握手/就绪信号；worker 在 `setup_vardir()` **之前**阻塞等它 |
| `START` | worker → 主 | 新 worker 就绪，请求第一个用例（**不带负载**） |
| `TESTCASE` | 主 → worker | 派发一个用例（负载 = 序列化后的 `My::Test`） |
| `TESTRESULT` | worker → 主 | 回传结果（负载 = 同一个 `My::Test` + result/failures/logfile） |
| `BYE` | 主 → worker | 用例发完 / 失败即停，令其退出 |
| `SPENT` | worker → 主 | 各阶段耗时（`--report-times` 用） |
| `VALGREP` | worker → 主 | 发现 valgrind/sanitizer 报告 |
| `SRV_CRASH` | worker → 主 | 关闭期检测到 mysqld 崩溃 |

```perl
# lib/My/Test.pm:53 —— 单行协议的关键：转义 | \ LF CR
sub _encode {
  my ($value) = @_;
  $value =~ s/([|\\\x{0a}\x{0d}])/sprintf('\%02X', ord($1))/eg;
  return $value;
}
sub write_test {
  my ($test, $sock, $header) = @_;
  print $sock $header, "\n";                       # ① codeword 行
  while ((my ($key, $value)) = each(%$test)) {
    print $sock $key, "= ";
    if (ref $value eq "ARRAY") { print $sock "[", _encode(join(", ", @$value)), "]"; }
    else                       { print $sock _encode($value); }
    print $sock "\n";
  }
  print $sock "\n";                                # ② 空行 = 消息结束符
}
```

**★ worker 主循环 `run_worker()`**（⚠️ 真名，不在 `My::Test` 里）：

```perl
sub run_worker ($) {
  my ($server_port, $thread_num) = @_;
  $SIG{INT} = sub { exit(1); };
  set_build_thread_ports($thread_num);      # ★ 每个 worker 独立端口段
  my $hello = <$server>;                    # ★★ 阻塞等主进程的 HELLO
  setup_vardir();
  print $server "START\n";                  # → 主
  while (my $line = <$server>) {
    chomp($line);
    if ($line eq 'TESTCASE') {
      my $test = My::Test::read_test($server);
      delete($test->{'comment'}); delete($test->{'logfile'});   # ★ 清掉上次残留
      $test->{worker} = $thread_num if $opt_parallel > 1;
      run_testcase($test);
      $test->write_test($server, 'TESTRESULT');                 # → 主
    } elsif ($line eq 'BYE') {
      mtr_report("Server said BYE");
      my $ret = stop_all_servers($opt_shutdown_timeout);
      print $server "SRV_CRASH\n" if $shutdown_report;
      my $valgrind_reports = valgrind_exit_reports() if (...);
      print $server "VALGREP\n" if $valgrind_reports;
      print_times_used($server, $thread_num);                   # → 主（SPENT）
      exit($valgrind_reports);
    } else { mtr_error("Could not understand server, '$line'"); }
  }
}
```

**★ 调度器的核心：criteria 排序 + reserved 预留**（★ "哪些用例能复用同一个 mysqld"的真相）：

```perl
# ① 收集期算 criteria（mtr_cases.pm:565）
      push(@criteria, "ndb=" . ($tinfo->{'ndb_test'} ? "A" : "B"));
      push(@criteria, $tinfo->{template_path});                        # 同为 my.cnf 模板的排一起
      my $opts = $tinfo->{'master_opt'} ? $tinfo->{'master_opt'} : [];
      push(@criteria, join("!", sort @{$opts}) . "~");                 # master_opt 完全相同
      push(@criteria, join("!", sort @{ $tinfo->{'slave_opt'} })) if $tinfo->{'slave_opt'};
      push(@criteria, $tinfo->{force_restart} ? "force-restart" : "no-restart");
      $tinfo->{criteria} = join(" ", @criteria);
    @$cases = sort { $a->{criteria} cmp $b->{criteria}; } @$cases;

# ② 调度期把"criteria 相同"的后续用例预留给同一 worker（mysql-test-run.pl:1221）
            if (!defined $t->{reserved} && defined $t->{criteria}) {
              $t->{criteria} =~ s/force-restart$/no-restart/;   # 比"下一个"时忽略 force-restart
              my $maxres = (@$tests - $i) / $opt_parallel + 1;  # ★ 别预留太多，防饿死
              for (my $j = $i + 1 ; $j <= $i + $maxres ; $j++) {
                my $tt = $tests->[$j];
                last unless defined $tt;
                last if $tt->{criteria} ne $criteria;
                last if $tt->{'not_parallel'};
                $tt->{reserved} = $wid;
              }
            }
```

> ★★ **criteria 本身不是"能否复用 mysqld"的判据**——真正判定在 `servers_need_restart($tinfo)`（返回需要重启的服务器列表，空列表 = 全部复用）。criteria 只是**把可能复用的用例在列表里排到相邻位置**，再由 `reserved` 机制钉给同一个 worker。
> ★ `not_parallel.inc` 的用例被单独摘出来，**等到所有其他 worker 都完成后由最后一个 worker 串行跑**。

**★ `--repeat=N` 是"收集期克隆"而不是"循环跑 N 遍"**：

```perl
sub duplicate_test_cases($) {
  my $tests = shift;
  my $new_tests = [];
  foreach my $test (@$tests) {
    if ($test->{'skip'}) { push(@{$new_tests}, $test); }    # ★ skip 的不重复
    else {
      for (my $i = 1 ; $i <= $::opt_repeat ; $i++) {
        push(@{$new_tests}, create_duplicate_test($test));   # ★ 深拷贝（数组字段逐个拷）
      }
    }
  }
  return $new_tests;
}
```

> ★★ 因此 N 份是**独立 tinfo**，可能被分到**不同 worker**——查 flaky 时不能假设它们跑在同一个 mysqld 上。

**★ 并行收集：suite 级用 Perl ithreads**（⚠️ 不是串行）：

```perl
      my $parallel = $ENV{NUMBER_OF_CPUS};
      $parallel = $::opt_parallel if ($::opt_parallel < $parallel);
      $parallel = 1               if $quick_collect;      # ★ --start* 时退化成 1
      ...
        push(@collect_test_cases_thrds,
             threads->create("collect_one_suite", $suite, $opt_cases,
                             $opt_skip_test_list, \%disabled));
        while ($parallel <= scalar @collect_test_cases_thrds) {
          mtr_milli_sleep(100);
          @collect_test_cases_thrds = threads->list(threads::running());
        }
```

**★ `--do-test` 的正则语义**（⚠️ `--do-test=foo` 不是"包含匹配"）：

```perl
sub init_pattern {
  my ($from, $what) = @_;
  return undef unless defined $from;
  if ($from =~ /^[a-z0-9\.]*$/) {
    # Does not contain any regex (except '.' that we allow as
    # separator betwen suite and testname), make the pattern match
    # beginning of string.
    $from = "^$from";        # ★★ 纯字面量 → 补 ^ 变"前缀匹配"
  }
  eval { "" =~ /$from/; 1 } or mtr_error("Invalid regex '$from' passed to $what\nPerl says: $@");
  return $from;             # ★ 含元字符则原样当正则用，不加 ^
}
```

> ★ 另外 `--do-test` 只匹配 **basename**（`next if ($do_test_reg and not $tname =~ /$do_test_reg/o)`），不含 suite 名。
> ★ `--skip-test` 命中的用例**仍进列表**，只是标 `skip`（所以统计里算 skipped 而不消失）；`disabled.def` 命中的则标 `skip` **且** `disable`（统计里单列）。

**`mtr_report_stats()`（`:553`）如何识别 unstable**：

```perl
    if ($tinfo->{failures}) {
      $tot_tests++; $tot_failed++;
      if ($::opt_report_unstable_tests and defined $tinfo->{retries}) {
        my $num_passed = $tinfo->{retries} - $tinfo->{failures} - 1;
        if ($num_passed > 0) {                    # ★ 有失败也有成功 = unstable
          $tot_unstable++;
          $tinfo->{'result'} = 'MTR_RES_UNSTABLE';
        }
      }
    }
    ...
  print "The servers were restarted $tot_restarts times\n";
  print "The servers were reinitialized $tot_reinits times\n";
```

#### 6.3 与构建系统的衔接

⚠️ **勘误**：CMake 里**不存在 `ADD_CUSTOM_TARGET(... mtr)`**。真实内容是：

```cmake
# mysql-test/CMakeLists.txt:80-81 —— mtr 只是符号链接
ADD_CUSTOM_COMMAND(
  OUTPUT ${CMAKE_BINARY_DIR}/mysql-test/mtr
  COMMAND ${CMAKE_COMMAND} -E create_symlink ./mysql-test-run.pl mtr
  ...)

# :117
ADD_CUSTOM_TARGET(test-mtr ...)     # ← 真名

# :143
ADD_CUSTOM_TARGET(run-profile-suite ...)
```

```cmake
# mysql-test/lib/My/SafeProcess/CMakeLists.txt:24
IF (WIN32)
  MYSQL_ADD_EXECUTABLE(mysqltest_safe_process safe_process_win.cc COMPONENT Test)
  MYSQL_ADD_EXECUTABLE(mysqltest_safe_kill safe_kill_win.cc COMPONENT Test)
ELSE()
  MYSQL_ADD_EXECUTABLE(mysqltest_safe_process safe_process.cc COMPONENT Test)
ENDIF()
#  ★ Windows 是完全独立的编译单元，不是 #ifdef _WIN32
```

> ★ 一个 ASAN 相关的坑（`CMakeLists.txt:33-45`）：glibc 把 Sun RPC/XDR 拆到 `libtirpc` 后与 libasan 不兼容，所以 ASAN 构建下会生成 `asan_library_name.h`，由 `safe_process.cc` 在 `execvp` 前 `putenv("LD_PRELOAD=...")` 预加载。
> ⚠️ **条件是 `#if defined(HAVE_ASAN) && defined(HAVE_TIRPC)`，且同时预载 libasan + libtirpc 两个库**。`asan_library_name.h` **不是仓库里的静态头文件**，是 CMake 构建期生成的：`ldd` 一个探针程序 `asan_test` → 正则抽路径 → 写 `.h`（`read_ldd_output.cmake`）。

`mysql-test/CMakeLists.txt` 把 MTR 接入 CMake：编译完成后可以直接 `ctest` 或 `make test` 触发（但 MTR 的主要用法仍是直接调 `mysql-test-run.pl`，因为选项更丰富）。

#### 6.4 常用选项速查

| 选项 | 作用 |
|---|---|
| `--parallel=N` / `auto` | 并行跑 N 个用例；`auto` = `$ENV{NUMBER_OF_CPUS}`，且**不超过用例数**。与 `--start-and-exit`/`--stress`/`--mysqlx_port` 冲突时强制降为 1 |
| `--repeat=N` | 收集期克隆 N 份（查 flaky 的常用手段） |
| `--retry=N` / `--retry-failure=N` | 默认 3 / 2，**实际最多跑 2 次** |
| `--force` | 失败不停（默认失败即停）。`--accept-test-fail` 隐含它 |
| `--max-test-fail=N` | 默认 10（`MTR_MAX_TEST_FAIL`） |
| `--big-test` / `--only-big-test` | 跑 big 用例；两者同时给会关掉后者。**命令行点名用例时自动开 `--big-test`** |
| `--record` | 录制基线 |
| `--start` / `--start-and-exit` | 起服务器但不跑用例（**调试时极有用**） |
| `--gdb` / `--ddd` / `--dbx` / `--manual-*` | 在调试器下跑 |

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

**★ `--start*` 的实现：全局 `$start_only` + `quick-collect`**：

```perl
# :2155
  if ($opt_start or $opt_start_dirty or $opt_start_exit) {
    collect_option('quick-collect', 1);   # ★ 只收集到第一个用例就停，不用扫全 suite
    $start_only = 1;
  }
```

```perl
# run_testcase() 里（:5042）—— 起完服务器就打印端口并分叉
  if ($start_only) {
    mtr_print("\nStarted",    started(all_servers()));
    mtr_print("Using config", $tinfo->{template_path});
    mtr_print("Port and socket path for server(s):");
    foreach my $mysqld (mysqlds()) {
      mtr_print($mysqld->name() . "  " . $mysqld->value('port') . "  " . $mysqld->value('socket'));
    }
    if ($opt_start_exit) {
      mtr_print("Server(s) started, not waiting for them to finish");
      if (IS_WINDOWS) { POSIX::_exit(0); }    # exit hangs here in ActiveState Perl
      else            { exit(0); }
    }
    ...
    mtr_print("Waiting for server(s) to exit...");   # ★ 裸 --start：挂在这里等服务器退出
    if ($opt_wait_all) { My::SafeProcess->wait_all(); ... }
    else { my $proc = My::SafeProcess->wait_any(); ... }
  }
```

> ★★ **`--start-and-exit` 还有个反向操作**（`start_servers()` 里）：`My::SafeProcess->start_exit()` —— 设全局开关后 `new()` **不再包 `safe_process`**，直接启动真实程序。否则 mysqld 会被守护进程看护、随父进程退出而被杀。**这是 `--start-and-exit` 能留下一个可连接 mysqld 的关键**。

**★ `--gdb` / `--ddd` / `--dbx` 的实现是"包装命令行"**（★ 通过引用改写 `$exe` 与 `$args`）：

```perl
  if ($opt_gdb || $opt_manual_gdb)        { gdb_arguments(\$args, \$exe, $mysqld->name()); }
  elsif ($opt_lldb || $opt_manual_lldb)   { lldb_arguments(\$args, \$exe, $mysqld->name()); }
  elsif ($opt_ddd || $opt_manual_ddd)     { ddd_arguments(\$args, \$exe, $mysqld->name()); }
  elsif ($opt_dbx || $opt_manual_dbx)     { dbx_arguments(\$args, \$exe, $mysqld->name()); }
  elsif ($opt_debugger)                   { debugger_arguments(\$args, \$exe, $mysqld->name()); }
  elsif ($opt_manual_debug) {
    print "\nStart " . $mysqld->name() . " in your debugger\n" .
      "dir: $glob_mysql_test_dir\n" . "exe: $exe\n" . "args:  " . join(" ", @$args) . "\n\n" . "Waiting ....\n";
    $exe = undef;      /* ★★ --manual-debug：干脆不启动，只把命令行打给用户 */
  }
```

```perl
# ★ 用调试器时把所有超时放宽到"一周"（:2124）
    $glob_debugger = 1;
    if (using_extern()) { mtr_error("Can't use --extern when using debugger"); }
    $opt_testcase_timeout = 7 * 24 * 60;      /* Set one week timeout */
    $opt_suite_timeout    = 7 * 24 * 60;
    $opt_shutdown_timeout = 24 * 60;          /* One day to shutdown */
    $opt_start_timeout    = 24 * 60 * 60;     /* One day for PID file creation */

# 并给 mysqld 传 --gdb
  if ($glob_debugger) { mtr_add_arg($args, "--gdb"); }
```

> ★ **`--gdb` 会同时给 mysqld 传 `--gdb`**（让服务器侧配合，如禁用 stack trace 之类）。
> ★ 用调试器时**不能**配 `--extern`（源码直接 `mtr_error`）。

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

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `client/mysqltest.h` | **不存在**（只有 `client/mysqltest.cc`，364 KB）；相关头在 `client/mysqltest/` 子目录（`regular_expressions.h`、`logfile.h`） |
| 有 `commands[]` 名字→处理函数映射数组 | **不存在**。分发是 `main()` 里的**两个 `switch (command->type)`**（9742–10203 主 switch、10208–10224 块命令 switch） |
| `do_send()` / `do_reap()` | **都不存在**。`send` 是 `case Q_SEND:` 直接 `run_query(..., QUERY_SEND_FLAG)`；`reap` 与 `query` **共用** `case Q_QUERY/Q_REAP`，靠 `flags` 区分 |
| `do_sorted_result()` / `do_partially_sorted_result()` / `do_lowercase_result()` | **都不存在** —— 各是主 switch 的一个 `case`，设 1–2 个全局标志 |
| `do_skip_if_hypergraph()` | **不存在**（只有 `skip_if_hypergraph = true;`）；真正的跳过在 `run_query()` 里提前 return |
| `do_query()` | **不存在** → 核心是 `run_query()`（8933），再分派 `run_query_normal()` / `run_query_stmt()` |
| `do_enable_ps_protocol()` / `do_disable_ps_protocol()` | **不存在** → 复用通用 `set_property(command, P_PS, ...)`，disable 时额外 `close_statements()` |
| `find_command()` / `get_command()` | **不存在** → **`read_command()`**（7616，读行切首词）+ **`get_command_type()`**（9187，查 TYPELIB） |
| `do_inc()` / `do_dec()` | **不存在** → `do_modify_var(command, DO_INC / DO_DEC)` |
| `do_if()` / `do_while()` / `do_assert()` | **不存在** → 统一 `do_block(cmd, command)`；`assert` 不入块栈，失败直接 `die` |
| `do_end()` | 真名 **`do_done()`**（6913） |
| `do_die()` | **不存在** → 内联 `die("%s", command->first_argument)` |
| `do_replace_result()` 等 | 真名带 **`get`**：`do_get_replace` / `do_get_replace_column` / `do_get_replace_regex` / `do_get_replace_numeric_round` |
| `do_connection()` / `do_disconnect()` | **不存在** → **`select_connection()`** / **`do_close_connection()`**（`dirty_close` 与之共用，差异只是 `vio_delete`+`end_server` 的"脏关闭"分支） |
| `var_fill()` | **不存在** → `var_init` → `var_get` → `eval_expr` → `var_set`，另有 `var_obtain` / `var_from_env` |
| `struct REPLACE` 叫 `st_replace` | 真名就是 **`REPLACE`**（`st_replace` 0 命中）；是一棵 **256 路 trie** |
| `command_names` 与枚举有 `static_assert` 校验 | **没有任何 `static_assert`**。唯一约束是枚举上方的注释 `Must match the "command_names" array / Add new commands before Q_UNKNOWN!` |
| TYPELIB 初始化是 `array_elements(...) - 1` | **没有 `-1`**，是 `{array_elements(command_names), "", command_names, nullptr}`（=100，含哨兵） |
| 命令总数"约 100 条" | `command_names` 中**真实命令名 99 个**；`enum enum_commands` 共 **103 个**值（+ `Q_UNKNOWN`/`Q_COMMENT`/`Q_COMMENT_WITH_COMMAND`/`Q_EMPTY_LINE`） |
| `compare_results` / `dyn_string_cmp` | 真名 **`check_result()`**（2193）→ `compare_files()` → `compare_files2()`。`dyn_string_cmp` **已废弃**，只剩 `default:` 分支里一句历史错误文案 |
| Perl 侧生成 `.reject` / 做 diff | **错**。全在 `client/mysqltest.cc` 的 `check_result()`；路径受 `--logdir` 控制（`opt_logdir`） |
| 比对时会归一化尾部空格/换行 | **错**。比对是纯 `memcmp` 分块比较，**零归一化**；归一化只在**输出生成期**（`replace_dynstr_append_mem`） |
| `--record` 会整理/格式化后落盘 | **错**。就是一行 `my_copy(log_file, result_file_name, MYF(0))`，**逐字节相同** |
| `safe_process.cc` 有 "Retry a couple of times if fork returns EAGAIN" | **不存在**（老版本残留说法）。8.0.39 是 `while ((child_pid = fork()) == -1) { message("fork failed"); sleep(1); }`——**无限重试、不判 errno** |
| LD_PRELOAD 段条件是 `HAVE_ASAN` | 实际是 `#if defined(HAVE_ASAN) && defined(HAVE_TIRPC)`，且**同时预载 libasan + libtirpc** |
| `asan_library_name.h` 是仓库静态头文件 | **不是**。CMake 构建期生成：`ldd asan_test` → 正则抽路径 → `read_ldd_output.cmake` 写 `.h` |
| 跨平台 `safe_kill.c` | 真名 **`safe_kill_win.cc`**（仅 Windows，目标 `mysqltest_safe_kill`）；**Unix 侧没有 safe_kill**，直接 `kill("TERM", $pid)` |
| SafeProcess 只用 `kill()` | 三层：`start_kill()`（只发信号不等）、`kill()`（= start_kill + wait_one）、`shutdown($timeout, @procs)`（先优雅回调，超时再硬杀） |
| bootstrap 用 `mysqld --bootstrap` | **错**。是 `mysqld --initialize-insecure` + `--init-file=var/tmp/bootstrap.sql`；函数名 `mysql_install_db()` |
| 启动就绪靠 "ping 端口" | **错**。默认**不等**（`$wait_for_pid_file = 0`）；就绪判定是等 **pid 文件出现**（`sleep_until_pid_file_created`）。`mtr_ping_port` 只在启动**前**检查端口是否空闲 |
| `My::ConfigFactory->new()` | **不存在** → 类方法 **`new_config()`**（725），另有 `run_rules_for_group()` / `run_section_rules()` + 几十个 `fix_*` |
| worker 主循环在 `My::Test` 里 | **错**。是 **`run_worker()`**（1314）；`My/Test.pm` 只做测试对象 + 编解码 |
| 主进程与 worker 用 pipe/共享内存 | **错**。是 **TCP socket**（`IO::Socket::INET`，localhost）；协议是**纯文本行** |
| `collect_test_cases()` 串行扫 suite | **错**。用 **Perl `threads` 并行收集**，并发度 `min(NUMBER_OF_CPUS, --parallel)`；`quick_collect` 时退化成 1 |
| `--do-test=foo` 是"包含匹配" | **错**。纯字面量（只含 `[a-z0-9.]`）才补 `^` 变前缀匹配；含元字符则**原样当正则不加 `^`** |
| `disabled.def` 只有一个文件 | **错**。按优先级收集最多 7 类：internal/cloud、internal、`collections/disabled.def`、`disabled_ndb.def`、`disabled-asan.list`、`disabled-ubsan.list`、`disabled-valgrind.list`、+ `--skip-test-list` |
| `--retry=3` = 最多再跑 3 次（共 4 次） | **错**。默认 `retry=3`/`retry_failure=2`，**实际最多跑 2 次**（详见上文推演） |
| 重试在 worker 里做 | **错**。完全在**主进程** `run_test_server()`：清 `result`、递增 `retries`、同一 tinfo 再发 `TESTCASE` |
| `MTR_RES_UNSTABLE` 由 worker 判定 | **错**。由**主进程收尾** `mtr_report_stats()` 判定，且**仅当 `--report-unstable-tests` 开启** |
| `--repeat=N` 是循环跑 N 遍 | **错**。是 `duplicate_test_cases()` 在收集阶段**克隆 N 个独立 tinfo**（可能被分到不同 worker） |
| "`MTR_RES_*` 五个退出码" | **`MTR_RES_*` 只有 4 个且是裸字符串**（PASSED/FAILED/SKIPPED/UNSTABLE），无任何 `use constant` 定义。真正的五个常量是 **mysqltest 进程退出码**：`MYSQLTEST_PASS=0 / FAIL=1 / SKIPPED=62 / NOSKIP_PASS=63 / NOSKIP_FAIL=64` |
| `lib/mtr_results.pm` 定义 `MTR_RES_*` | **错**。它只导出 `resfile_*` 系列（结果文件读写），**不含任何 `MTR_RES_*`** |
| `print_test_result()` | **不存在** → `mtr_report_test()`，另有 `mtr_report_test_passed()` / `_skipped()` |
| CMake 里有 `ADD_CUSTOM_TARGET(... mtr)` | **不存在** → `mtr` 只是 **符号链接**（`create_symlink`）；target 真名 `test-mtr` / `run-profile-suite` |

**★ 三个"结构性事实"**

1. **复用服务器 `TYPELIB` 做命令解析，是"零成本拿到前缀匹配 + 大小写不敏感 + 未匹配回退"的关键**——`command_names[]` 之外没有第二张表，`find_type()` 用 `FIND_TYPE_NO_PREFIX` 保证全名匹配，`type == 0` 时回退成 `Q_QUERY`（发给 mysqld）。代价是**枚举与名字表必须严格同序**，而 8.0.39 **没有任何编译期校验**，只有一行注释约束——加新命令插错位置会静默错位。

2. **"输出确定性"是三层分工，且归一化与比对彻底分离**——生成期 `replace_dynstr_append_mem()` 按 `lowercase → replace_regex → replace_result → replace_numeric_round` 顺序改写；比对期纯 `memcmp` 分块比较；失败期产出 `.reject` + 三级降级的 diff。**Perl 侧不参与任何归一化**。所以 `.result` 里看到的是"已归一化后的输出"，`replace_*` 不是"隐藏问题"而是 golden file 范式对确定性的必然要求。

3. **`safe_process` 用"独立进程组 + 父进程探活"解决孤儿进程**——子进程 `setpgid(0, 0)` 独立成组，守护循环每秒 `kill(parent_pid, 0)` 探活，MTR 主进程一消失就 `kill(-child_pid, SIGKILL)` **带走整棵进程树**；守护进程用子进程的退出码退出（`exit(exit_code)`），所以 MTR 看到的退出码即真实退出码。这是"MTR 被 Ctrl-C 后不留 mysqld 孤儿"的根本机制，也是 `--start-and-exit` 必须反向调用 `My::SafeProcess->start_exit()` 绕过它的原因。

> 注：本工作区副本含少量非上游改动（`mysql-test/lib/My/SafeProcess.pm` 中的 `[MORPHY]` 调试打印、`mysql-test-run.pl:4955` 与部分源码文件中的中文批注），撰写时已剔除或标注。

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
- [`../../server/query/11_explain_and_trace.md`](../../infra/observability/explain_trace.md) —— optimizer trace（MTR 用例里最常用的观察手段）

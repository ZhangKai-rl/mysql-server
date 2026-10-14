# sys schema：MySQL 自带的运维便捷层

> **边界**：本篇讲 **sys schema 本身**（它是什么、源码怎么组织、视图/函数/过程体系、设计取舍）。
> - **PFS 框架**见 [`pfs/pfs.md`](pfs/pfs.md)；**事件表族**见 [`pfs/pfs_events.md`](pfs/pfs_events.md)
> - **I_S**见 [`i_s.md`](i_s.md)
> - 观测手段全景与选型决策树见 [`../README.md`](../README.md)
>
> **一句话**：sys 不是新的观测数据源，而是**架在 PFS / I_S 之上的一层 SQL 视图与工具**，把"机器友好的窄表"加工成"人能直接看的结果"。

## 目录

- [一、概述](#一概述)
- [二、理论基础：为什么需要 sys，代价是什么](#二理论基础为什么需要-sys代价是什么)
- [三、源码组织与安装机制](#三源码组织与安装机制)
- [四、视图体系（102 个）](#四视图体系102-个)
- [五、函数体系（22 个）](#五函数体系22-个)
- [六、存储过程体系（26 个）](#六存储过程体系26-个)
- [七、可观测性](#七可观测性)
- [八、★ 工程实现技法](#八-工程实现技法)
- [九、Misc（易混淆点）](#九misc易混淆点)
- [参考](#参考)

---

## 一、概述

### 是什么

sys schema 是 MySQL **自带**的一层纯 SQL 对象（视图 + 存储函数 + 存储过程），全部基于 `performance_schema` 与 `information_schema` 构建，**不存任何自己的数据**。

规模（源码实测，`scripts/sys_schema/`）：

| 类别 | 数量 |
|---|---|
| 视图 | **102**（`views/p_s/` 91 + `views/i_s/` 10 + `version` 1） |
| 存储函数 | **22** |
| 存储过程 | **26**（另有 2 个非过程片段文件） |
| 表 | 1（`sys_config`，外加 2 个升级/数据文件） |
| 触发器 | 2（`sys_config` 的 insert/update 设 user） |
| .sql 文件合计 | **161** |

### 用途

- **给人看**：把皮秒转 `18.45 s`、字节转 `664.06 KiB`、长 SQL 截断成一行
- **替掉老的观测入口**：`sys.session` 替代 `SHOW PROCESSLIST`（且**锁更少**）
- **跨表组合**：把 PFS 多张窄表 JOIN 成一张能回答问题的宽表
- **直接产出可执行语句**：`innodb_lock_waits` 直接给 `KILL` 语句

### 版本演进（源码级）

- **8.0**：格式化函数从"存储函数"**下沉为服务器内建函数**（`FORMAT_BYTES` / `FORMAT_PICO_TIME` / `PS_THREAD_ID` / `PS_CURRENT_THREAD_ID` 注册在 `sql/item_create.cc`），sys 侧只留薄封装
- **8.0 新增专属文件**（在 `CMakeLists.txt` 而不在 5.7 的 `sys_57.txt` 中）：
  - `tables/sys_config_alter.sql`（`ALTER TABLE sys_config CHARACTER SET utf8mb4;`）
  - `views/p_s/innodb_lock_waits.sql` + `x$`（基于 `performance_schema.data_lock_waits` / `data_locks`，注释标 `Versions: 8.0+`）
- **8.0 取消了 sys schema 版本化升级**（`sql/dd/impl/upgrade/server.cc` 注释：*"With the SYS schema versioning removed"*），`sys.version` 视图已标注 deprecated

---

## 二、理论基础：为什么需要 sys，代价是什么

### 2.1 为什么需要它（源码注释直接给的答案）

**① 可读性** —— `functions/format_time.sql` 的 COMMENT：

```
Takes a raw picoseconds value, and converts it to a human readable form.

Picoseconds are the precision that all latency values are printed in
within Performance Schema, however are not user friendly when wanting
to scan output from the command line.
```

⇒ PFS 给你 `342342342342345`，sys 给你 `00:05:42`。

**② 更少的锁（非阻塞）** —— `views/p_s/processlist.sql`：

```
-- A detailed non-blocking processlist view to replace
-- [INFORMATION_SCHEMA. | SHOW FULL] PROCESSLIST
-- Performs less locking than the legacy sources, whilst giving extra information.
```

⇒ `SHOW PROCESSLIST` / `I_S.PROCESSLIST` 要持全局锁遍历 THD 列表；`sys.session` 只读 PFS 表。

**③ 信息量** —— `session` 比 `SHOW PROCESSLIST` 多出 20+ 列（`thd_id`、`rows_examined`、`tmp_disk_tables`、`full_scan`、`current_memory`、`last_wait` + `source` 文件行号、`trx_state`、客户端 `pid`/`program_name`、ALTER 的 `progress` 百分比）。

**④ 跨表组合** —— PFS 是"按维度打散的多张窄表"，真实问题往往要跨表：

| 视图 | 由什么组合 |
|---|---|
| `schema_table_statistics` | `table_io_waits_summary_by_table`（逻辑行操作）+ `file_summary_by_instance`（物理 IO，靠路径反解） |
| `user_summary` | `accounts`（连接）+ `x$user_summary_by_statement_latency`（语句）+ `x$user_summary_by_file_io`（IO）+ `x$memory_by_user_by_current_bytes`（内存） |
| `innodb_lock_waits` | `data_lock_waits` + `data_locks`×2 + `I_S.INNODB_TRX`×2 |

**⑤ 直接产出可执行结果** —— `innodb_lock_waits` 生成 `KILL QUERY 5`；`schema_redundant_indexes` 生成 `ALTER TABLE ... DROP INDEX ...`。

### 2.2 ★ 代价（三条，都有源码证据）

#### ① 视图每次查询现算，不物化

MySQL 视图无物化。sys 作者是靠**把中间结果拆成独立 `ALGORITHM = TEMPTABLE` 视图**来给优化器创造物化机会——辅助视图的注释直接说明：

```sql
-- Helper view for schema_table_statistics
-- Having this view with ALGORITHM = TEMPTABLE means MySQL can use the optimizations for
-- materialized views to improve the overall performance.
```

而且视图**层层嵌套**：`user_summary` 依赖 3 个 `x$` 视图，每个又各扫一遍 PFS 汇总表；`statements_with_runtimes_in_95th_percentile` 依赖的百分位视图还**自连接两次**并含一个标量子查询。

`processlist` 是 `ALGORITHM = TEMPTABLE` 且 JOIN 了 7 张表（含 2 次 `session_connect_attrs` 自连）+ `ORDER BY`。

⇒ **查 sys 视图 = 触发多轮 PFS 表扫描 + 可能的派生表物化（写临时表）**。

⚠️ **"sys 视图查询代价的具体量级"源码无直接证据**（无 benchmark 注释）。可实证的说法是：纯 SQL 视图、无缓存、无物化、层层嵌套，代价 ≈ 底层 PFS 表扫描 × 嵌套层数 + 派生表物化。

#### ② ★ 格式化 → 字符串 → 不可排序比较（有意取舍，补偿机制是 `x$`）

三个格式化函数的返回类型：

| 函数 | 返回类型 | 源码说明 |
|---|---|---|
| `format_bytes` | **TEXT** | *"We feed in and return TEXT here, as aggregates of bytes can return numbers larger than BIGINT UNSIGNED"* |
| `format_time` | TEXT CHARSET UTF8MB4 | — |
| `format_statement` | **LONGTEXT** | — |

**代价清单**：

1. **不可排序**：`ORDER BY total_latency` 按字符串排（`1.02 ms` < `999.00 us` 会排错）
2. **不可比较/聚合**：无法 `WHERE total_latency > 1000000`，也无法 `SUM()`
3. **精度丢失**：`format_pico_time` 只保留 2 位小数 + 单位

⇒ 这正是 `user_summary` 必须 JOIN `x$user_summary_by_statement_latency` 而不是常规版的**唯一原因**。

**设计意图**：

> **无前缀视图 = 人读（默认）；`x$` 视图 = 机器读（监控采集 / 二次聚合 / JOIN）。**

#### ③ 存储过程需要 SUPER 且会关 binlog

多数 `ps_setup_*` / `diagnostics` / `statement_performance_analyzer` 的注释都写着：

```
Requires the SUPER privilege for "SET sql_log_bin = 0;".
```

代码里都有 `SET @log_bin := @@sql_log_bin; SET sql_log_bin = 0; ... SET sql_log_bin = @log_bin;`（因为要建临时表/改配置，不该复制到从库）。

### 2.3 sys 与 PFS / I_S 的关系

**目录本身就是分类**：`views/i_s/`（10 个，**纯 I_S**）vs `views/p_s/`（91 个，多数纯 PFS）。

| 类型 | 例子 | 证据 |
|---|---|---|
| **纯 I_S** | `innodb_buffer_stats_by_schema` / `_by_table`、`schema_auto_increment_columns`、`schema_object_overview`、`schema_redundant_indexes` | 在 `views/i_s/` 全目录搜索 `performance_schema.` → **0 匹配** |
| **纯 PFS** | `wait_classes_global_by_latency`、`memory_by_thread_by_current_bytes`、`io_by_thread_by_latency` | — |
| **PFS + I_S 混用** | `p_s/innodb_lock_waits`（PFS 锁 + I_S 事务）、`latest_file_io`（PFS 历史 + I_S.PROCESSLIST）、`schema_unused_indexes`（PFS + I_S.STATISTICS）、`metrics`（**四路 UNION**） | 见定义 |

`metrics` 是四路 UNION ALL 的典型（`global_status` + `INNODB_METRICS` + PFS 内存 + 当前时间）：

```sql
(SELECT LOWER(VARIABLE_NAME), VARIABLE_VALUE, 'Global Status', 'YES'
   FROM performance_schema.global_status) UNION ALL (
SELECT NAME, COUNT, CONCAT('InnoDB Metrics - ', SUBSYSTEM),
       IF(STATUS = 'enabled','YES','NO')
  FROM information_schema.INNODB_METRICS
) ...
```

（`Type` 列的 `Enabled` 有第三态 `'PARTIAL'`，注释解释：*"PARTIAL ... means some but not all of the memory/% instruments are enabled."*）

### 2.4 用户定位：**主要面向 DBA（人），次面向监控工具**

**面向 DBA 的证据**：格式化函数的理由注释都是 *"useful ... from the command line"*；`x$` 是补充而非主视图；`diagnostics()` 用法示例是 `TEE diag.out` + `NOTEE`；`ps_statement_avg_latency_histogram()` 输出 ASCII 柱状图。

**面向程序的证据**：`x$` 视图族（数值原值）；`table_exists()` 有 `OUT` 参数；`ps_thread_trx_info()` 返回 **JSON**；`init_priv.sql` 给 `mysql.sys` 授了 `SYSTEM_USER` / `AUDIT_ABORT_EXEMPT` / `FIREWALL_EXEMPT`（说明会被特权工具调用）。

---

## 三、源码组织与安装机制

### 3.1 目录结构

```
scripts/sys_schema/
├── functions/*.sql          22 个存储函数
├── procedures/*.sql         28 文件 → 26 过程 + 2 片段
├── views/
│   ├── version.sql          sys.version（已 deprecated）
│   ├── i_s/*.sql            10 个，基于 INFORMATION_SCHEMA
│   └── p_s/*.sql            91 个，基于 Performance Schema
├── tables/                  sys_config.sql / _alter.sql / _data.sql
├── triggers/                sys_config_insert_set_user / _update_set_user
├── before_setup.sql / after_setup.sql / init_priv.sql / sys_57.sql
└── CMakeLists.txt / run_gensysschema.cmake / generate_sql_file_57.sh / *.awk / sys_57.txt
```

⚠️ **文件名与视图名不一致的例子**：`views/p_s/sessions.sql` 建的是 **`session`** 视图，`x_sessions.sql` 建的是 **`x$session`**。

### 3.2 ★ 安装机制：构建期拼接 → 编译进二进制 → 初始化执行 → 升级重放

**① 构建期**：`CMakeLists.txt` 调 `run_gensysschema.cmake` 把所有 .sql 拼成 `ALL_SYS_SCHEMA.sql`，再用 `comp_sql` 工具（`scripts/comp_sql.cc`）转成 C 字符串数组 `mysql_sys_schema[]`，输出到 `sql_commands.h`。

**② 初始化期**：`sql/sql_initialize.cc` 把它列进命令表：

```cpp
static const char **cmds[] = {initialization_cmds, mysql_system_tables,
                              initialization_data, mysql_system_data,
                              fill_help_tables,    mysql_system_users,
                              mysql_sys_schema,    nullptr};
static const char *cmd_descs[] = {..., "Creating the sys schema", nullptr};
```

⇒ **sys schema 是 `mysqld --initialize` 时自动创建的，不需要手工安装。**

**③ 升级期**：`sql/dd/impl/upgrade/server.cc` 的 `fix_sys_schema()` 整段重放。

**关键：所有 SQL 都是幂等的**——`CREATE DATABASE IF NOT EXISTS` / `CREATE OR REPLACE VIEW` / `DROP FUNCTION IF EXISTS` + `CREATE` / `INSERT IGNORE`。所以同一段脚本可反复执行（初始化 + 每次升级）。

### 3.3 `before_setup.sql` / `after_setup.sql` 为什么需要

```sql
-- before_setup.sql
SET NAMES utf8mb4;
SET @sql_log_bin = @@sql_log_bin;
SET sql_log_bin = 0;
CREATE DATABASE IF NOT EXISTS sys DEFAULT CHARACTER SET utf8mb4;
ALTER DATABASE sys CHARACTER SET utf8mb4;
USE sys;
-- after_setup.sql
SET @@sql_log_bin = @sql_log_bin;
```

两个作用：

1. **建库期间关 binlog**（`after_setup` 还原）——sys 是每台实例自带的元数据层，不该复制到从库
2. **`USE sys;` 建立默认库上下文**——后续所有 `CREATE VIEW/FUNCTION/PROCEDURE` 都写裸名（如 `VIEW processlist`），靠这一句落到 `sys` 库

⇒ before/after 是一对**作用域括号**。

### 3.4 版本管理

`views/version.sql`：

```sql
-- NOTE: This view is deprecated and will be removed in a future release.
CREATE OR REPLACE ... VIEW version (sys_version, mysql_version) AS
SELECT '2.1.2' AS sys_version, version() AS mysql_version;
```

⇒ **sys 版本是硬编码常量 `2.1.2`**，不是从表读的；**没有 `sys_version` 表**。8.0 已取消版本比对升级，改为"升级就整段重放"。

### 3.5 ★ 权限模型（安全设计）

`init_priv.sql` 创建一个**密码被写成"永不合法"、全部权限位为 N 的锁定账号**：

```sql
INSERT IGNORE INTO mysql.user VALUES ('localhost','mysql.sys','N',...,'caching_sha2_password',
  '$A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED',...);
```

所有对象统一：

```sql
DEFINER = 'mysql.sys'@'localhost' SQL SECURITY INVOKER
```

- definer 是锁定账号 ⇒ 保证 definer 永远存在（视图不会因 definer 丢失报 1449），且自身无权限
- **`SQL SECURITY INVOKER`** ⇒ 实际以**调用者**权限执行，不越权

---

### 3.6 ★ `sys_config`：sys 的配置机制

表定义（`scripts/sys_schema/tables/sys_config.sql`）：

```sql
CREATE TABLE IF NOT EXISTS sys_config (
    variable VARCHAR(128) PRIMARY KEY,
    value VARCHAR(128),
    set_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    set_by VARCHAR(128)
) ENGINE = InnoDB;
```

- **`variable` 做主键**：保证配置项唯一；`INSERT IGNORE` 初始化时不会重复插入
- **`set_time` 自动维护**（`DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`）⇒ **自带审计**
- **`set_by`** 由触发器填 `USER()`
- **显式 `ENGINE = InnoDB`**（避免默认引擎被改后失去事务性）；这是 `sys` 库里**唯一的一张实体表**

默认值（`tables/sys_config_data.sql`）：

```sql
-- NOTE: This needs to be replicated within the sys_config_clean.inc file

INSERT IGNORE INTO sys.sys_config (variable, value) VALUES
    ('statement_truncate_len', 64),
    ('statement_performance_analyzer.limit', 100),
    ('statement_performance_analyzer.view', NULL),
    ('diagnostics.allow_i_s_tables', 'OFF'),
    ('diagnostics.include_raw', 'OFF'),
    ('ps_thread_trx_info.max_length', 65535);
```

| 配置项 | 默认 | 含义 |
|---|---|---|
| `statement_truncate_len` | `64` | `format_statement()` 截断长度；`0` = 不截断 |
| `statement_performance_analyzer.limit` | `100` | SPA 报表行数上限 |
| `statement_performance_analyzer.view` | `NULL` | 自定义视图；NULL = 不用 |
| `diagnostics.allow_i_s_tables` | **`OFF`** | ★ 默认关：**不扫描 INFORMATION_SCHEMA 表**（表多时全表扫 I_S 可能拖垮实例） |
| `diagnostics.include_raw` | **`OFF`** | ★ 默认关：诊断输出不含原始 P_S 数据（否则输出爆炸） |
| `ps_thread_trx_info.max_length` | `65535` | 事务信息 JSON 最大长度 |

⇒ **两个 `OFF` 是刻意的"安全默认"**：`diagnostics()` 是最重的存储过程（55 KB），默认只做轻量采集。

**触发器**（`triggers/sys_config_insert_set_user.sql`，update 版同构）：

```sql
CREATE DEFINER='mysql.sys'@'localhost' TRIGGER sys_config_insert_set_user
    BEFORE INSERT on sys_config
    FOR EACH ROW
BEGIN
    IF @sys.ignore_sys_config_triggers != true AND NEW.set_by IS NULL THEN
        SET NEW.set_by = USER();
    END IF;
END$$
```

**`@sys.ignore_sys_config_triggers` 是逃生舱**：升级脚本批量灌数据时设这个变量，触发器跳过，不会把升级账号写进 `set_by`。

**★ 读取范式**（以 `statement_performance_analyzer.sql` 为例，全库统一）：

```sql
    -- Set configuration options
    IF (@sys.statement_performance_analyzer.limit IS NULL) THEN
        SET @sys.statement_performance_analyzer.limit = sys.sys_get_config('statement_performance_analyzer.limit', '100');
    END IF;
    IF (@sys.debug IS NULL) THEN
        SET @sys.debug                                = sys.sys_get_config('debug'                               , 'OFF');
    END IF;
```

★ **为什么用 `IF...THEN` 而不是 `SET @x = IFNULL(@x, sys_get_config(...))`**——这不是风格问题是**性能问题**：`sys_get_config()` 内部有一次 `SELECT ... FROM sys.sys_config`，`IFNULL` 写法**每次执行都跑一次子查询**（在结果集循环里会跑 N 次）；`IF...THEN` 只在**会话首次**跑一次，之后短路。`sys_get_config.sql` 的 COMMENT 里明确写了这条警告。

其它约定：
- 位置固定在**过程体最开头**，统一 `-- Set configuration options` 注释段
- **默认值硬编码在第二个参数**，与 `sys_config_data.sql` 保持一致但**互相独立**——即使表被清空代码仍能跑
- 布尔配置用字符串 `'ON'`/`'OFF'`，判断写 `IF (@sys.debug = 'ON')`
- 调用方 `SET @sys.xxx = 10;` 即可**会话级覆盖**，无需改表

### 3.7 ★ 打包脚本 `generate_sql_file_57.sh`

⚠️ 本仓库是 **`generate_sql_file_57.sh`**（不是 `generate_sql_56.sql`），旁边还有 `sys_57.txt`、`sys_57.sql`。

```bash
OUTPUTFILE="../mysql_sys_schema.sql"

sed -e "s/^/-- /" LICENSE > ${OUTPUTFILE}
echo "" >> ${OUTPUTFILE}

cat >> ${OUTPUTFILE} << HERE_DOC
--
-- WARNING: THIS IS A MANUALLY GENERATED FILE,
-- CHANGES NEED TO BE MADE IN THE scripts/sys_schema/*.sql FILES.
--
-- SEE scripts/sys_schema/generate_sql_file_57.sh TO GENERATE THIS FILE
--
HERE_DOC

echo "" >> ${OUTPUTFILE}

#
# Concatenate all files listed in file sys_57.txt
# Remove individual file copyrights
# Replace newlines in COMMENTs with literal \n
# Drop added trailing \n closing the COMMENT string
# Remove leading spaces
# Remove -- line comments *after removing leading spaces*
# Collapse code limited by $$
# Replace '$$' delimiter with ;
# Collapse code limited by ;
# Remove leading spaces (again), to clean up after the collapses.
# Append to the resulting file
# #

cat `grep -v "#" sys_57.txt` | \
sed -e "/Copyright/,/51 Franklin St/d" | \
sed -e "/^ *COMMENT/,/^ *'/{G;s/\n/\\\n/g;}" | \
sed -e "s/^'\\\n$/'/g" | \
sed -e "s/^ *//g" | \
sed -e "/^--/d" | \
awk -f collapse-delimiter.awk | \
sed -e "s/\\$\\$/;\n/g" | \
awk -f collapse-semicolon.awk | \
sed -e "s/^ *//g" \
>> ${OUTPUTFILE}

echo "$OUTPUTFILE updated"
```

**它做的是"发布打包"，不是"派生 `x$` 版"**：

1. `LICENSE` 注释化前置
2. 按 `sys_57.txt` 的文件顺序 `cat` 拼接（`grep -v "#"` 跳过注释行 ⇒ `sys_57.txt` 本身是**带注释的文件清单**）
3. **剥掉每个文件各自 15 行的 GPL 版权头**（`/Copyright/,/51 Franklin St/d`）——避免 160 个文件 × 15 行重复版权
4. **把多行 `COMMENT '...'` 折成单行**，内部换行换字面 `\n`：因为 `mysql` 客户端非交互模式下 `DELIMITER` 无效，必须先把 SP 体压成单行
5. **`sed "/^--/d"` 删掉所有 `--` 注释** ⇒ 单文件版 `mysql_sys_schema.sql` 里**看不到源码注释**
6. `collapse-delimiter.awk` + `s/$$/;\n/g` + `collapse-semicolon.awk`：把 `DELIMITER $$ ... $$` 还原成普通 `;` 结尾的单行语句，**消除 `DELIMITER` 依赖**，使生成的 `.sql` 能被 `mysql <` 直接执行
7. 输出 `../mysql_sys_schema.sql`，供 `mysqld --bootstrap` / `mysql_upgrade` 一次性装载

⇒ **`x$` 与非 `x$` 是两份独立源码，不存在代码生成关系**（见 5.4）。维护成本双倍，是 sys 已知的取舍。

### 3.8 安装顺序的约束（三点）

`sys_57.sql` 是纯 `SOURCE` 清单，但有**三处顺序约束**不能颠倒：

**① `before_setup.sql` 先关 binlog**：

```sql
SET NAMES utf8mb4;
SET @sql_log_bin = @@sql_log_bin;
SET sql_log_bin = 0;
CREATE DATABASE IF NOT EXISTS sys DEFAULT CHARACTER SET utf8mb4;
ALTER DATABASE sys CHARACTER SET utf8mb4;
USE sys;
```

`after_setup.sql` 再 `SET @@sql_log_bin = @sql_log_bin;` 恢复。

**为什么关 binlog**：sys 是每台机器都一样的元数据，不关会把几十 KB DDL 复制到所有从库并污染 GTID。

**② helper 视图必须在依赖它的视图之前**：

```sql
SOURCE ./views/p_s/x_ps_digest_avg_latency_distribution.sql
SOURCE ./views/p_s/x_ps_digest_95th_percentile_by_avg_us.sql
SOURCE ./views/p_s/statements_with_runtimes_in_95th_percentile.sql
```

这三段是**严格依赖链**（见 4.3 ④）。同理 `processlist` 必须在 `sessions` 之前（`session` 引用它）。

**③ `execute_prepared_stmt.sql` 在 `diagnostics.sql` 之前**（后者调用前者这个动态 SQL 公共工具）。

**④ 每个 `p_s` 视图都是 `x$` 紧随非 `x$` 成对 SOURCE**：

```sql
SOURCE ./views/p_s/io_by_thread_by_latency.sql
SOURCE ./views/p_s/x_io_by_thread_by_latency.sql
```

⇒ 注意 `i_s/` 目录下的前缀是 **`x_`** 而非 `x$`（如 `x_innodb_lock_waits.sql`）——因为 5.x 时代标识符里 `$` 有兼容问题。这是 `p_s/` 用 `x$`、`i_s/` 用 `x_` 的历史原因。

---

## 四、视图体系（102 个）

### 4.1 ★ 命名规律

#### ① `x$` 前缀 = 数值原值版（机器消费）

⚠️ **源码无直接注释定义 `x$`**（`scripts/sys_schema/` 下无 README）。以下结论由**成对视图定义差异** + **内部 JOIN 一律引用 `x$` 版**两点反推。

`x$` 与非 `x$` 视图**列集合、JOIN、ORDER BY 逐字相同**，唯一区别是**不格式化**：

| 列 | `processlist` | `x$processlist` |
|---|---|---|
| `current_statement` | `sys.format_statement(pps.processlist_info)` | `pps.processlist_info` |
| `statement_latency` | `format_pico_time(esc.timer_wait)` | `esc.timer_wait`（裸皮秒） |
| `current_memory` | `format_bytes(mem.current_allocated)` | `mem.current_allocated`（裸字节） |

**硬证据：`x$` 视图被其它对象内部消费**（要数值比较/聚合）：

- `statements_with_runtimes_in_95th_percentile` JOIN `x$ps_digest_95th_percentile_by_avg_us ... ON ROUND(stmts.avg_timer_wait/1000000) >= top_percentile.avg_us`
- `user_summary` JOIN `x$user_summary_by_statement_latency` 并 `SUM(stmt.total_latency)`
- `processlist` JOIN `x$memory_by_thread_by_current_bytes`
- `ps_statement_avg_latency_histogram()` 内部 SELECT `sys.x$ps_digest_avg_latency_distribution`

**纯辅助（只有 `x$` 版，无人读版）**：`x$ps_digest_avg_latency_distribution`、`x$ps_digest_95th_percentile_by_avg_us`、`x$ps_schema_table_statistics_io`、`x$schema_flattened_keys`。

⚠️ **`_raw` 后缀在 8.0.39 源码中不存在任何视图**——只有配置变量 `diagnostics.include_raw`。**源码无直接证据支持 `_raw` 视图命名约定。**

#### ② `_by_<维度>` / `_global` / 前缀

| 片段 | 语义 | 例 |
|---|---|---|
| `_by_user` / `_by_host` / `_by_thread` / `_by_account` | 按该实体分组 | `memory_by_thread_by_current_bytes` |
| `_by_file` / `_by_wait` / `_by_stages` / `_by_statement_type` | 按该客体/阶段分组 | `io_global_by_file_by_bytes` |
| `_by_latency` / `_by_bytes` / `_by_avg_latency` | **排序/度量口径** | `wait_classes_global_by_avg_latency` |
| `_global` | 不做 per-user/host 拆分，全局聚合 | `waits_global_by_latency`、`memory_global_total` |
| `schema_` 前缀 | 按库/表维度 | `schema_table_statistics` |
| `statements_with_` 前缀 | 语句摘要维度的"筛选型"视图 | `statements_with_full_table_scans` |
| `ps_` 前缀 | 直接对应 PFS 原表的辅助/校验视图 | `ps_check_lost_instrumentation` |

读作："`<客体> _by_ <分组维度> _by_ <排序度量>`"。

### 4.2 视图清单（按用途）

| 用途 | 视图（每个通常都有对应 `x$` 版） |
|---|---|
| **会话/连接** (5) | `session`+`x$session`、`processlist`+`x$processlist`、`session_ssl_status`（仅 1） |
| **语句分析** (14) | `statement_analysis`、`statements_with_errors_or_warnings`、`statements_with_full_table_scans`、`statements_with_runtimes_in_95th_percentile`、`statements_with_sorting`、`statements_with_temp_tables` + 辅助 `x$ps_digest_*` |
| **IO** (12) | `io_by_thread_by_latency`、`io_global_by_file_by_bytes`、`io_global_by_file_by_latency`、`io_global_by_wait_by_bytes`、`io_global_by_wait_by_latency`、`latest_file_io` |
| **锁** (4) | `innodb_lock_waits`、`schema_table_lock_waits` |
| **内存** (10) | `memory_by_host_by_current_bytes`、`memory_by_thread_by_current_bytes`、`memory_by_user_by_current_bytes`、`memory_global_by_current_bytes`、`memory_global_total` |
| **Schema/表** (10) | `schema_index_statistics`、`schema_table_statistics`、`schema_table_statistics_with_buffer`、`schema_tables_with_full_table_scans`、`schema_unused_indexes`（仅 1）+ 辅助 |
| **用户汇总** (12) | `user_summary`、`user_summary_by_file_io(_type)`、`user_summary_by_stages`、`user_summary_by_statement_latency`、`user_summary_by_statement_type` |
| **主机汇总** (12) | `host_summary` 及同构的 5 个 `host_summary_by_*` |
| **等待** (10) | `wait_classes_global_by_avg_latency`、`wait_classes_global_by_latency`、`waits_by_host_by_latency`、`waits_by_user_by_latency`、`waits_global_by_latency` |
| **杂项** (2) | `metrics`、`ps_check_lost_instrumentation` |

### 4.3 逐个深剖（10 个）

#### ① `sys.session` —— 只是 `processlist` 的一层过滤

源文件 `scripts/sys_schema/views/p_s/sessions.sql`（**文件名是复数 `sessions`，视图名是单数 `session`**）：

```sql
CREATE OR REPLACE
  DEFINER = 'mysql.sys'@'localhost'
  SQL SECURITY INVOKER 
VIEW session
 AS
SELECT * FROM sys.processlist
WHERE conn_id IS NOT NULL AND command != 'Daemon';
```

`x$session`（`x_sessions.sql`）完全对称，只是 `SELECT * FROM sys.x$processlist`。

**为什么这么写**：
- **`session` 自己不做任何表关联**，它是 `processlist` 的纯过滤层。这是 sys 的分层设计：**重逻辑只写一遍**，`session` 只负责"去掉后台线程"。
- `conn_id IS NOT NULL`：`threads.processlist_id` 只有前台连接才有值，后台线程（main、page_cleaner…）是 NULL。
- `command != 'Daemon'`：8.0 里被 kill 但尚未清理的连接会显示 `Daemon`，二次兜底。
- 它没有 `ALGORITHM = TEMPTABLE`（`processlist` 有）⇒ 走 MERGE，**继承 `processlist` 的排序**。

#### ② `sys.processlist` —— 五表 LEFT JOIN，全篇最复杂

```sql
CREATE OR REPLACE
  ALGORITHM = TEMPTABLE
  DEFINER = 'mysql.sys'@'localhost'
  SQL SECURITY INVOKER
VIEW processlist (
  thd_id, conn_id, user, db, command, state, time,
  current_statement, execution_engine, statement_latency, progress,
  lock_latency, cpu_latency, rows_examined, rows_sent, rows_affected,
  tmp_tables, tmp_disk_tables, full_scan,
  last_statement, last_statement_latency, current_memory,
  last_wait, last_wait_latency, source,
  trx_latency, trx_state, trx_autocommit, pid, program_name
) AS
SELECT pps.thread_id AS thd_id,
       pps.processlist_id AS conn_id,
       IF(pps.name IN ('thread/sql/one_connection', 'thread/thread_pool/tp_one_connection'),
          CONCAT(pps.processlist_user, '@', pps.processlist_host),
          REPLACE(pps.name, 'thread/', '')) user,
       pps.processlist_db AS db,
       pps.processlist_command AS command,
       pps.processlist_state AS state,
       pps.processlist_time AS time,
       sys.format_statement(pps.processlist_info) AS current_statement,
       pps.execution_engine AS execution_engine,
       IF(esc.end_event_id IS NULL,
          format_pico_time(esc.timer_wait),
          NULL) AS statement_latency,
       IF(esc.end_event_id IS NULL,
          ROUND(100 * (estc.work_completed / estc.work_estimated), 2),
          NULL) AS progress,
       format_pico_time(esc.lock_time) AS lock_latency,
       format_pico_time(esc.cpu_time) AS cpu_latency,
       esc.rows_examined AS rows_examined,
       esc.rows_sent AS rows_sent,
       esc.rows_affected AS rows_affected,
       esc.created_tmp_tables AS tmp_tables,
       esc.created_tmp_disk_tables AS tmp_disk_tables,
       IF(esc.no_good_index_used > 0 OR esc.no_index_used > 0, 'YES', 'NO') AS full_scan,
       IF(esc.end_event_id IS NOT NULL,
          sys.format_statement(esc.sql_text),
          NULL) AS last_statement,
       IF(esc.end_event_id IS NOT NULL,
          format_pico_time(esc.timer_wait),
          NULL) AS last_statement_latency,
       format_bytes(mem.current_allocated) AS current_memory,
       ewc.event_name AS last_wait,
       IF(ewc.end_event_id IS NULL AND ewc.event_name IS NOT NULL,
          'Still Waiting',
          format_pico_time(ewc.timer_wait)) last_wait_latency,
       ewc.source,
       format_pico_time(etc.timer_wait) AS trx_latency,
       etc.state AS trx_state,
       etc.autocommit AS trx_autocommit,
       conattr_pid.attr_value as pid,
       conattr_progname.attr_value as program_name
  FROM performance_schema.threads AS pps
  LEFT JOIN performance_schema.events_waits_current        AS ewc USING (thread_id)
  LEFT JOIN performance_schema.events_stages_current      AS estc USING (thread_id)
  LEFT JOIN performance_schema.events_statements_current  AS esc  USING (thread_id)
  LEFT JOIN performance_schema.events_transactions_current AS etc USING (thread_id)
  LEFT JOIN sys.x$memory_by_thread_by_current_bytes AS mem USING (thread_id)
  LEFT JOIN performance_schema.session_connect_attrs AS conattr_pid
    ON conattr_pid.processlist_id=pps.processlist_id and conattr_pid.attr_name='_pid'
  LEFT JOIN performance_schema.session_connect_attrs AS conattr_progname
    ON conattr_progname.processlist_id=pps.processlist_id and conattr_progname.attr_name='program_name'
 ORDER BY pps.processlist_time DESC, last_wait_latency DESC;
```

**逐段解释**：

- **五个 `LEFT JOIN` 而非 INNER**：`threads` 是驱动表，每行必须保留（否则空闲连接就消失了）。用 `USING (thread_id)` 是因为五张 `events_*_current` 的主键都是 `thread_id`，天然 0 或 1 行匹配，**不会产生行放大**。这正是它**比 `SHOW PROCESSLIST` / `I_S.PROCESSLIST` 更轻**的原因：不需要持有 `LOCK_thd_data`、不遍历 THD 链表。
- **user 列的 IF**：前台连接显示 `user@host`；后台线程没有 user/host，把 `thread/innodb/page_cleaner_thread` 剥成 `page_cleaner_thread`。
- **★ `IF(esc.end_event_id IS NULL, ...)` 这个"NULL 语义开关"最精妙**：`end_event_id IS NULL` 表示**事件尚未结束**（正在跑）。所以正在跑 → `statement_latency` 有值而 `last_statement` 为 NULL；已结束（空闲连接）→ 反过来。**一个 IF 同时表达"当前语句"和"上一条语句"，永远只有一边非空**。
- **`progress`** 来自 `events_stages_current` 的 `work_completed/work_estimated`（online DDL 进度），同样挂在"正在进行"上。
- **`last_wait_latency` 返回 `'Still Waiting'`** 字符串：此刻正卡在某个等待上。返回字符串而非数字，是为了一眼看出"还在等"。
- **★ `LEFT JOIN sys.x$memory_by_thread_by_current_bytes`（引用 `x$` 版）**再自己套 `format_bytes()`。这是 sys 通例：**内部互引用一律取 `x$` 拿原始值，只在最外层格式化一次**，避免"格式化后的字符串再参与聚合"。
- **`ALGORITHM = TEMPTABLE`**：强制物化，避免多个 `*_current` 内存表被反复扫描，也防 ORDER BY 与 MERGE 冲突。

#### ③ `sys.innodb_lock_waits` —— "谁等谁"的四表十字关联

```sql
SELECT r.trx_wait_started AS wait_started,
       TIMEDIFF(NOW(), r.trx_wait_started) AS wait_age,
       TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_age_secs,
       CONCAT(sys.quote_identifier(rl.object_schema), '.',
              sys.quote_identifier(rl.object_name)) AS locked_table,
       rl.object_schema AS locked_table_schema,
       rl.object_name AS locked_table_name,
       rl.index_name AS locked_index,
       rl.lock_type AS locked_type,
       r.trx_id AS waiting_trx_id,
       r.trx_mysql_thread_id AS waiting_pid,
       sys.format_statement(r.trx_query) AS waiting_query,
       rl.engine_lock_id AS waiting_lock_id,
       rl.lock_mode AS waiting_lock_mode,
       b.trx_id AS blocking_trx_id,
       b.trx_mysql_thread_id AS blocking_pid,
       sys.format_statement(b.trx_query) AS blocking_query,
       bl.engine_lock_id AS blocking_lock_id,
       bl.lock_mode AS blocking_lock_mode,
       CONCAT('KILL QUERY ', b.trx_mysql_thread_id) AS sql_kill_blocking_query,
       CONCAT('KILL ', b.trx_mysql_thread_id) AS sql_kill_blocking_connection
    FROM performance_schema.data_lock_waits w
 INNER JOIN information_schema.innodb_trx b  ON b.trx_id = CAST(w.blocking_engine_transaction_id AS CHAR)
 INNER JOIN information_schema.innodb_trx r  ON r.trx_id = CAST(w.requesting_engine_transaction_id AS CHAR)
 INNER JOIN performance_schema.data_locks bl ON bl.engine_lock_id = w.blocking_engine_lock_id
 INNER JOIN performance_schema.data_locks rl ON rl.engine_lock_id = w.requesting_engine_lock_id
 ORDER BY r.trx_wait_started;
```

**关联结构**（核心是 `data_lock_waits` 这张"边表"，8.0 起取代 5.7 的 `INNODB_LOCK_WAITS`）：

```
data_lock_waits w  (一行 = 一条等待边)
   ├─ requesting_engine_transaction_id ─CAST AS CHAR─> innodb_trx r  (等待方事务)
   ├─ blocking_engine_transaction_id  ─CAST AS CHAR─> innodb_trx b  (阻塞方事务)
   ├─ requesting_engine_lock_id       ──────────────> data_locks rl (想拿的锁)
   └─ blocking_engine_lock_id         ──────────────> data_locks bl (持有的锁)
```

**逐段解释**：
- **★ `CAST(... AS CHAR)` 必须**：`data_lock_waits` 的事务 id 是数字型，而 `INNODB_TRX.TRX_ID` 是字符串（InnoDB 内部 trx_id 可能是 128 位大数），不 CAST 会类型不匹配。
- **全部 INNER JOIN**（与 `processlist` 相反）：一条等待边缺任意一端就是残缺数据，宁可不显示。
- **`locked_*` 全取 `rl`**（requesting lock，即"想要的锁"）而非 `bl`：语义上"被争抢的资源"由请求方描述更准确。
- **`quote_identifier()`** 把 `locked_table` 拼成 `` `test`.`t1` ``，可直接复制执行。
- **`wait_age_secs` 怎么算**：`TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW())` —— **纯 SQL 计算，不是 P_S 计时器**。因为 P_S 的 lock 表**不带时间戳**，只能借 `INNODB_TRX.TRX_WAIT_STARTED`。这也是它要额外 JOIN `information_schema` 的原因（代价略高）。
- **`sql_kill_blocking_query` / `sql_kill_blocking_connection`**：直接生成 `KILL QUERY 5` / `KILL 5` 字符串——sys 的"**可执行输出**"设计。
- **`ORDER BY r.trx_wait_started` 升序**：等得最久的排最前。

#### ④ ★ `statements_with_runtimes_in_95th_percentile` —— 纯 SQL 算百分位

主视图：

```sql
CREATE OR REPLACE
  ALGORITHM = MERGE
  DEFINER = 'mysql.sys'@'localhost'
  SQL SECURITY INVOKER 
VIEW statements_with_runtimes_in_95th_percentile (
  query, db, full_scan, exec_count, err_count, warn_count,
  total_latency, max_latency, avg_latency,
  rows_sent, rows_sent_avg, rows_examined, rows_examined_avg,
  first_seen, last_seen, digest
) AS
SELECT sys.format_statement(DIGEST_TEXT) AS query,
       SCHEMA_NAME as db,
       IF(SUM_NO_GOOD_INDEX_USED > 0 OR SUM_NO_INDEX_USED > 0, '*', '') AS full_scan,
       COUNT_STAR AS exec_count,
       SUM_ERRORS AS err_count,
       SUM_WARNINGS AS warn_count,
       format_pico_time(SUM_TIMER_WAIT) AS total_latency,
       format_pico_time(MAX_TIMER_WAIT) AS max_latency,
       format_pico_time(AVG_TIMER_WAIT) AS avg_latency,
       SUM_ROWS_SENT AS rows_sent,
       ROUND(IFNULL(SUM_ROWS_SENT / NULLIF(COUNT_STAR, 0), 0)) AS rows_sent_avg,
       SUM_ROWS_EXAMINED AS rows_examined,
       ROUND(IFNULL(SUM_ROWS_EXAMINED / NULLIF(COUNT_STAR, 0), 0)) AS rows_examined_avg,
       FIRST_SEEN AS first_seen,
       LAST_SEEN AS last_seen,
       DIGEST AS digest
  FROM performance_schema.events_statements_summary_by_digest stmts
  JOIN sys.x$ps_digest_95th_percentile_by_avg_us AS top_percentile
    ON ROUND(stmts.avg_timer_wait/1000000) >= top_percentile.avg_us
 ORDER BY AVG_TIMER_WAIT DESC;
```

**`avg_latency` 直接用 P_S 的 `AVG_TIMER_WAIT` 列**，不是手写 `SUM_TIMER_WAIT/COUNT_STAR`——二者数学等价（P_S 内部维护），但直接用省一次除法且精度由 P_S 保证。

对照：下面 `wait_classes_global_by_latency` 里**必须手写** `SUM(sum_timer_wait)/SUM(COUNT_STAR)`，因为做了 `GROUP BY`（见 ⑨）。

**★ 95 百分位是怎么算的（两层 helper，纯 SQL 技巧）**

第一层——做**直方图**（把 N 条语句压成 M 个桶）：

```sql
-- x$ps_digest_avg_latency_distribution
SELECT COUNT(*) cnt, 
       ROUND(avg_timer_wait/1000000) AS avg_us
  FROM performance_schema.events_statements_summary_by_digest
 GROUP BY avg_us;
```

第二层——**累积分布自连接**（不用窗口函数，兼容老版本）：

```sql
-- x$ps_digest_95th_percentile_by_avg_us
SELECT s2.avg_us avg_us,
       IFNULL(SUM(s1.cnt)/NULLIF((SELECT COUNT(*) FROM performance_schema.events_statements_summary_by_digest), 0), 0) percentile
  FROM sys.x$ps_digest_avg_latency_distribution AS s1
  JOIN sys.x$ps_digest_avg_latency_distribution AS s2
    ON s1.avg_us <= s2.avg_us
 GROUP BY s2.avg_us
HAVING IFNULL(SUM(s1.cnt)/NULLIF((SELECT COUNT(*) FROM performance_schema.events_statements_summary_by_digest), 0), 0) > 0.95
 ORDER BY percentile
 LIMIT 1;
```

**技巧拆解**：
- **`s1 JOIN s2 ON s1.avg_us <= s2.avg_us`**：对每个候选阈值 `s2.avg_us`，把所有 `avg_us ≤ 它` 的桶计数 `s1.cnt` **累加** → 这就是**经验累积分布函数（ECDF）**的值。
- 分母是**原始表的总行数**（子查询），不是 `SUM(cnt)`——避免 helper 视图自身过滤导致偏差。
- **`HAVING ... > 0.95` + `ORDER BY percentile` + `LIMIT 1`**：取"累积比例刚超过 95% 的最小阈值"，正是百分位定义。用 `HAVING` 而非 `WHERE`，因为过滤的是聚合结果。
- **`NULLIF(...,0)` + `IFNULL(...,0)` 双重保险**：digest 表为空时不除零。

回到主视图：`ON ROUND(stmts.avg_timer_wait/1000000) >= top_percentile.avg_us` —— 用同一"微秒取整"口径对齐，**只有一条 `>=`**，所以主视图能用 `ALGORITHM = MERGE`。

#### ⑤ `sys.io_by_thread_by_latency`

```sql
SELECT IF(processlist_id IS NULL, 
             SUBSTRING_INDEX(name, '/', -1), 
             CONCAT(processlist_user, '@', processlist_host)
          ) user, 
       SUM(count_star) total,
       format_pico_time(SUM(sum_timer_wait)) total_latency,
       format_pico_time(MIN(min_timer_wait)) min_latency,
       format_pico_time(AVG(avg_timer_wait)) avg_latency,
       format_pico_time(MAX(max_timer_wait)) max_latency,
       thread_id,
       processlist_id
  FROM performance_schema.events_waits_summary_by_thread_by_event_name 
  LEFT JOIN performance_schema.threads USING (thread_id)
 WHERE event_name LIKE 'wait/io/file/%'
   AND sum_timer_wait > 0
 GROUP BY thread_id, processlist_id, user
 ORDER BY SUM(sum_timer_wait) DESC;
```

**解释**：
- **数据源是 `events_waits_summary_by_thread_by_event_name`**（按 `thread × event_name` 二维聚合），能回答"哪个线程在 IO 上最慢"；而 `file_summary_by_instance` 按文件聚合，只能回答"哪个文件最热"。
- **`event_name LIKE 'wait/io/file/%'`** 是 sys 的惯用法：P_S 的 `event_name` 是 `/` 分隔的层级路径，前缀过滤 = 取整个子树。
- **★ `AVG(avg_timer_wait)` 是"未加权平均"** —— 先按 event_name 求 avg，再对所有 event_name 求 AVG。与"总时长/总次数"的**加权平均**不同。sys 这里选未加权，语义是"一个典型 IO 事件的耗时"，避免被超高频小 IO 拉低。**这是容易被忽略的语义陷阱**。
- **`SUBSTRING_INDEX(name,'/',-1)`**：后台线程取路径最后一段 → `page_cleaner_thread`。

#### ⑥ `sys.io_global_by_file_by_bytes`

```sql
SELECT sys.format_path(file_name) AS file, 
       count_read, 
       format_bytes(sum_number_of_bytes_read) AS total_read,
       format_bytes(IFNULL(sum_number_of_bytes_read / NULLIF(count_read, 0), 0)) AS avg_read,
       count_write, 
       format_bytes(sum_number_of_bytes_write) AS total_written,
       format_bytes(IFNULL(sum_number_of_bytes_write / NULLIF(count_write, 0), 0.00)) AS avg_write,
       format_bytes(sum_number_of_bytes_read + sum_number_of_bytes_write) AS total, 
       IFNULL(ROUND(100-((sum_number_of_bytes_read/ NULLIF((sum_number_of_bytes_read+sum_number_of_bytes_write), 0))*100), 2), 0.00) AS write_pct 
  FROM performance_schema.file_summary_by_instance
 ORDER BY sum_number_of_bytes_read + sum_number_of_bytes_write DESC;
```

**解释**：
- **`ALGORITHM = MERGE`**（与 ⑤ 的 TEMPTABLE 对比）：无 JOIN、无 GROUP BY，MERGE 可把外层 WHERE 下推到 P_S 表。
- **★ `IFNULL(x / NULLIF(y,0), 0)` 是 sys 的标志性防御写法**，全库出现几十次。只读或只写的文件很常见，除零会被兜成 0 而非报错/NULL 列。
- **★ `ORDER BY` 用未格式化的原始表达式**，绝不能写 `ORDER BY total`——因为 `total` 已是 `"4.31 MiB"` 字符串，会按**字典序**排！这是 sys 的铁律：**排序一律用原始数值列**。

#### ⑦ `memory_global_by_current_bytes` 与 `x$` 版：★ 逐行 diff

非 `x$` 版：

```sql
SELECT event_name,
       current_count_used AS current_count,
       format_bytes(current_number_of_bytes_used) AS current_alloc,
       format_bytes(IFNULL(current_number_of_bytes_used / NULLIF(current_count_used, 0), 0)) AS current_avg_alloc,
       high_count_used AS high_count,
       format_bytes(high_number_of_bytes_used) AS high_alloc,
       format_bytes(IFNULL(high_number_of_bytes_used / NULLIF(high_count_used, 0), 0)) AS high_avg_alloc
  FROM performance_schema.memory_summary_global_by_event_name
 WHERE current_number_of_bytes_used > 0
 ORDER BY current_number_of_bytes_used DESC;
```

`x$` 版：

```sql
SELECT event_name,
       current_count_used AS current_count,
       current_number_of_bytes_used AS current_alloc,
       IFNULL(current_number_of_bytes_used / NULLIF(current_count_used, 0), 0) AS current_avg_alloc,
       high_count_used AS high_count,
       high_number_of_bytes_used AS high_alloc,
       IFNULL(high_number_of_bytes_used / NULLIF(high_count_used, 0), 0) AS high_avg_alloc
  FROM performance_schema.memory_summary_global_by_event_name
 WHERE current_number_of_bytes_used > 0
 ORDER BY current_number_of_bytes_used DESC;
```

★ **唯一差异**：`x$` 版裸列，非 `x$` 版在 4 个字节列外裹一层 `format_bytes()`。**列名、列序、FROM、WHERE、ORDER BY、ALGORITHM 完全相同** ⇒ `SELECT *` 列序一致，脚本切换 `x$` 前缀可无痛迁移。

⇒ 且**两版 ORDER BY 都用原始字节列**，排序结果完全一致（格式化不改变顺序）——这是"x$ 与非 x$ 同源"的关键设计。

#### ⑧ `sys.schema_table_statistics` —— 逻辑行操作 + 物理 IO 合一

```sql
SELECT pst.object_schema AS table_schema,
       pst.object_name AS table_name,
       format_pico_time(pst.sum_timer_wait) AS total_latency,
       pst.count_fetch AS rows_fetched,
       format_pico_time(pst.sum_timer_fetch) AS fetch_latency,
       pst.count_insert AS rows_inserted,
       format_pico_time(pst.sum_timer_insert) AS insert_latency,
       pst.count_update AS rows_updated,
       format_pico_time(pst.sum_timer_update) AS update_latency,
       pst.count_delete AS rows_deleted,
       format_pico_time(pst.sum_timer_delete) AS delete_latency,
       fsbi.count_read AS io_read_requests,
       format_bytes(fsbi.sum_number_of_bytes_read) AS io_read,
       format_pico_time(fsbi.sum_timer_read) AS io_read_latency,
       fsbi.count_write AS io_write_requests,
       format_bytes(fsbi.sum_number_of_bytes_write) AS io_write,
       format_pico_time(fsbi.sum_timer_write) AS io_write_latency,
       fsbi.count_misc AS io_misc_requests,
       format_pico_time(fsbi.sum_timer_misc) AS io_misc_latency
  FROM performance_schema.table_io_waits_summary_by_table AS pst
  LEFT JOIN x$ps_schema_table_statistics_io AS fsbi
    ON pst.object_schema = fsbi.table_schema
   AND pst.object_name = fsbi.table_name
 ORDER BY pst.sum_timer_wait DESC;
```

**解释**：
- **两个维度拼一张表**：`table_io_waits_summary_by_table` 给**次数维度**（fetch/insert/update/delete 各多少次、耗时），`x$ps_schema_table_statistics_io`（sys 自己按文件聚合的 helper）给**字节维度**。一次 JOIN 把"逻辑行操作"和"物理 IO"合到一行。
- **又见 `x$` 引用**（`LEFT JOIN x$ps_schema_table_statistics_io`），取原始字节后再格式化。
- **`LEFT JOIN` 而非 INNER**：某些表（内存临时表等）在 IO 侧可能无对应行，INNER 会整个丢掉。
- **`io_misc`**（`count_misc`）是 open/close/seek/stat 等其他文件操作，常被忽略但其实是"表打开开销"的信号。

#### ⑨ `sys.wait_classes_global_by_latency` —— 上卷到类别

```sql
SELECT SUBSTRING_INDEX(event_name,'/', 3) AS event_class, 
       SUM(COUNT_STAR) AS total,
       format_pico_time(SUM(sum_timer_wait)) AS total_latency,
       format_pico_time(MIN(min_timer_wait)) min_latency,
       format_pico_time(IFNULL(SUM(sum_timer_wait) / NULLIF(SUM(COUNT_STAR), 0), 0)) AS avg_latency,
       format_pico_time(MAX(max_timer_wait)) AS max_latency
  FROM performance_schema.events_waits_summary_global_by_event_name
 WHERE sum_timer_wait > 0
   AND event_name != 'idle'
 GROUP BY SUBSTRING_INDEX(event_name,'/', 3) 
 ORDER BY SUM(sum_timer_wait) DESC;
```

**解释**：
- **`SUBSTRING_INDEX(event_name,'/',3)`**：`wait/io/file/innodb/innodb_data_file` 取前 3 段 → `wait/io/file`。这是把 P_S 的细粒度 instrument 树**上卷到类别层**（改数字换粒度：2→`wait/io`，4→`wait/io/file/innodb`）。"wait_classes" 这个名字就来自这里。
- **★ `AND event_name != 'idle'` 必写**：`idle` 的 `sum_timer_wait` 是启动以来所有空闲时间的累计（量级是"小时 × 线程数"），不排除会**永远占据 Top1**，整个视图作废。
- **★ `avg_latency = SUM(sum_timer_wait)/SUM(COUNT_STAR)` 是加权平均**，与 ⑤ 的 `AVG(avg_timer_wait)`（未加权）形成对比——二者都对但语义不同。

#### ⑩ 视图体系的两条通则（从上面 9 个归纳）

| 通则 | 说明 |
|---|---|
| **内部互引用一律取 `x$`** | 格式化后的 TEXT 无法参与 JOIN/聚合；只在最外层格式化一次 |
| **ORDER BY 永远用原始数值列** | 否则会按格式化后的字符串字典序排（经典 bug） |
| **`NULLIF` + `IFNULL` 防御除零** | P_S 的计数器在"从未发生"时是 0 |
| **`ALGORITHM` 按复杂度选** | 纯列变换用 MERGE（可下推）；JOIN+ORDER BY 用 TEMPTABLE（物化更稳） |
| **可执行输出** | 直接生成 `KILL 5` / `` `db`.`tbl` `` 这类能复制就跑的字符串 |

### 4.5 再补 6 个常用视图（代码密集）

#### ① `innodb_buffer_stats_by_schema` —— 按库看缓冲池占用

```sql
SELECT IF(LOCATE('.', ibp.table_name) = 0, 'InnoDB System', REPLACE(SUBSTRING_INDEX(ibp.table_name, '.', 1), '`', '')) AS object_schema,
       format_bytes(SUM(IF(ibp.compressed_size = 0, 16384, compressed_size))) AS allocated,
       format_bytes(SUM(ibp.data_size)) AS data,
       COUNT(ibp.page_number) AS pages,
       COUNT(IF(ibp.is_hashed = 'YES', 1, NULL)) AS pages_hashed,
       COUNT(IF(ibp.is_old = 'YES', 1, NULL)) AS pages_old,
       ROUND(SUM(ibp.number_records)/COUNT(DISTINCT ibp.index_name)) AS rows_cached
  FROM information_schema.innodb_buffer_page ibp
 WHERE table_name IS NOT NULL
 GROUP BY object_schema
 ORDER BY SUM(IF(ibp.compressed_size = 0, 16384, compressed_size)) DESC;
```

**要点**：
- ★ **未压缩页用硬编码 `16384`** 而不是 `@@innodb_page_size` —— `compressed_size` 为 0 表示未压缩，此时按默认 16KB 估算。**若你的实例改过页大小，这一列会算错**（已知近似）。
- **`COUNT(IF(cond, 1, NULL))`** 是条件计数的惯用写法（NULL 不计入 COUNT），比 `SUM(IF(cond,1,0))` 更直观。
- **`LOCATE('.', table_name) = 0` → `'InnoDB System'`**：系统表空间的页没有 `库.表` 结构，归为一类。
- ⚠️ 扫描 `INFORMATION_SCHEMA.INNODB_BUFFER_PAGE` **代价很高**（要遍历整个缓冲池的页元数据），大缓冲池上慎用。

#### ② `user_summary` —— 三层嵌套引用的典型

```sql
SELECT IF(accounts.user IS NULL, 'background', accounts.user) AS user,
       SUM(stmt.total) AS statements,
       format_pico_time(SUM(stmt.total_latency)) AS statement_latency,
       format_pico_time(IFNULL(SUM(stmt.total_latency) / NULLIF(SUM(stmt.total), 0), 0)) AS statement_avg_latency,
       SUM(stmt.full_scans) AS table_scans,
       SUM(io.ios) AS file_ios,
       format_pico_time(SUM(io.io_latency)) AS file_io_latency,
       SUM(accounts.current_connections) AS current_connections,
       SUM(accounts.total_connections) AS total_connections,
       COUNT(DISTINCT host) AS unique_hosts,
       format_bytes(SUM(mem.current_allocated)) AS current_memory,
       format_bytes(SUM(mem.total_allocated)) AS total_memory_allocated
  FROM performance_schema.accounts
  LEFT JOIN sys.x$user_summary_by_statement_latency AS stmt ON IF(accounts.user IS NULL, 'background', accounts.user) = stmt.user
  /* ... 继续 LEFT JOIN x$user_summary_by_file_io、x$memory_by_user_by_current_bytes ... */
 GROUP BY user;
```

**要点**：
- **驱动表是 `performance_schema.accounts`**（账号维度），再 LEFT JOIN 三个 `x$` helper（语句 / IO / 内存）。这就是 sys 的**"维度表 + 多个 x$ 指标表"**装配模式。
- **`IF(user IS NULL, 'background', user)`**：PFS 的 `accounts` 表里后台线程是 `user = NULL`，归为 `background` 一行。
- 注意 `statement_avg_latency` 这里是**加权平均**（`SUM(total_latency)/SUM(total)`），因为做了 GROUP BY。

#### ③ `waits_by_user_by_latency`

```sql
SELECT IF(user IS NULL, 'background', user) AS user,
       event_name AS event,
       count_star AS total,
       format_pico_time(sum_timer_wait) AS total_latency,
       format_pico_time(avg_timer_wait) AS avg_latency,
       format_pico_time(max_timer_wait) AS max_latency
  FROM performance_schema.events_waits_summary_by_user_by_event_name
 WHERE event_name != 'idle'
   AND user IS NOT NULL
   AND sum_timer_wait > 0
 ORDER BY user, sum_timer_wait DESC;
```

**要点**：两个"排噪"条件必写——**`event_name != 'idle'`**（否则永远 Top1）、**`user IS NOT NULL`**（后台线程）；再加 `sum_timer_wait > 0` 去掉从未发生的组合。

#### ④ `host_summary_by_file_io`

```sql
SELECT IF(host IS NULL, 'background', host) AS host,
       SUM(count_star) AS ios,
       format_pico_time(SUM(sum_timer_wait)) AS io_latency
  FROM performance_schema.events_waits_summary_by_host_by_event_name
 WHERE event_name LIKE 'wait/io/file/%'
 GROUP BY IF(host IS NULL, 'background', host)
 ORDER BY SUM(sum_timer_wait) DESC;
```

#### ⑤ `latest_file_io` —— 最近发生的 IO 事件

```sql
SELECT IF(id IS NULL,
             CONCAT(SUBSTRING_INDEX(name, '/', -1), ':', thread_id),
             CONCAT(user, '@', host, ':', id)
          ) thread,
       sys.format_path(object_name) file,
       format_pico_time(timer_wait) AS latency,
       operation,
       format_bytes(number_of_bytes) AS requested
  FROM performance_schema.events_waits_history_long
  JOIN performance_schema.threads USING (thread_id)
  LEFT JOIN information_schema.processlist ON processlist_id = id
 WHERE object_name IS NOT NULL
   AND event_name LIKE 'wait/io/file/%'
 ORDER BY timer_start;
```

**要点**：
- 数据源是 **`events_waits_history_long`**（历史环形缓冲），所以能看"最近发生过什么"——这是 `*_current` 表做不到的。
- ⚠️ 但它 JOIN 了 `INFORMATION_SCHEMA.PROCESSLIST`（为了拿 `user@host`），**这是 sys 里少数会碰到 I_S 的地方**，代价高于纯 PFS 视图。
- 后台线程用 `SUBSTRING_INDEX(name,'/',-1)` 拼 `page_cleaner:12` 这种"线程名:thread_id"。

#### ⑥ `ps_check_lost_instrumentation` —— 最短但极有用

```sql
SELECT variable_name, variable_value
  FROM performance_schema.global_status
 WHERE variable_name LIKE 'perf%lost'
   AND variable_value > 0;
```

**要点**：查所有 `Performance_schema_*_lost` 状态变量，**只要非零就说明 PFS 因为内存上限丢了事件/实例**。这是"PFS 数据不完整"的第一排查点——**在做任何 PFS 分析前应该先看它**（对应 `performance_schema_max_*` 系列启动变量需要调大）。

### 4.4 为什么很多视图要配一个同名格式化函数

1. **逻辑复用 + 唯一真相源**：几十个视图都要把皮秒转 `18.45 s`，内联无法维护
2. **可集中配置**：`format_statement` 的截断长度从 `sys_config` 读（见 5.2），改一行配置影响**所有**视图
3. **历史演进**：最早是纯 SQL 存储过程实现，8.0 把热路径下沉为内建函数，sys 同名函数退化为薄封装以保兼容

---

## 五、函数体系（22 个）

### 5.1 完整清单

| 分类 | 函数 |
|---|---|
| **格式化/展示** (4) | `format_bytes`、`format_time`、`format_statement`、`format_path` |
| **PFS 探测/映射** (9) | `ps_thread_id`、`ps_thread_account`、`ps_thread_stack`、`ps_thread_trx_info`、`ps_is_account_enabled`、`ps_is_consumer_enabled`、`ps_is_instrument_default_enabled`、`ps_is_instrument_default_timed`、`ps_is_thread_instrumented` |
| **路径解析** (2) | `extract_schema_from_file_name`、`extract_table_from_file_name` |
| **标识符** (1) | `quote_identifier` |
| **配置** (1) | `sys_get_config` |
| **版本** (3) | `version_major`、`version_minor`、`version_patch` |
| **列表操作** (2) | `list_add`、`list_drop` |

⚠️ 两个"容易以为有、实际没有"的：**`format_number` 不存在**；**`ps_current_thread_id` 不是 sys 的存储函数**（是 8.0 服务器内建函数）。

### 5.2 ★ 格式化函数族：8.0 已下沉为服务器内建函数

`functions/format_bytes.sql`——sys 侧只剩薄封装：

```sql
CREATE DEFINER='mysql.sys'@'localhost' FUNCTION sys.format_bytes (
        -- We feed in and return TEXT here, as aggregates of
        -- bytes can return numbers larger than BIGINT UNSIGNED
        bytes TEXT
    ) RETURNS TEXT
BEGIN
  IF (bytes IS NULL) THEN RETURN NULL;
  ELSE RETURN format_bytes(bytes);      -- ← 调服务器内建函数，不是自己实现
  END IF;
END$$
```

真正实现在 `sql/item_create.cc`：

```cpp
{"FORMAT_BYTES",     SQL_FN(Item_func_pfs_format_bytes, 1)},
{"FORMAT_PICO_TIME", SQL_FN(Item_func_pfs_format_pico_time, 1)},
{"PS_CURRENT_THREAD_ID", SQL_FN(Item_func_pfs_current_thread_id, 0)},
{"PS_THREAD_ID",         SQL_FN(Item_func_pfs_thread_id, 1)},
```

⇒ **性能原因**：存储函数每行都要走 SP 解释器，而 sys 视图会扫出成百上千行。

**唯一还保留完整 SQL 实现的是 `format_time`**（8.0 视图已改用 `format_pico_time`，它只为向后兼容）：

```sql
  ELSEIF picoseconds >= 604800000000000000 THEN RETURN CONCAT(ROUND(picoseconds/604800000000000000,2),' w');
  ELSEIF picoseconds >= 86400000000000000  THEN RETURN CONCAT(ROUND(picoseconds/86400000000000000,2), ' d');
  ...
  ELSEIF picoseconds >= 1000               THEN RETURN CONCAT(ROUND(picoseconds/1000,2), ' ns');
  ELSE RETURN CONCAT(picoseconds, ' ps');
```

### 5.3 关键函数定义（6 例，贴完整实现）

#### ① `sys.format_bytes()` —— ★ 8.0 起只是个 NULL 转发壳

```sql
CREATE DEFINER='mysql.sys'@'localhost' FUNCTION sys.format_bytes (
        -- We feed in and return TEXT here, as aggregates of
        -- bytes can return numbers larger than BIGINT UNSIGNED
        bytes TEXT
    )
    RETURNS TEXT
    SQL SECURITY INVOKER
    DETERMINISTIC
    NO SQL
BEGIN
  IF (bytes IS NULL) THEN
    RETURN NULL;
  ELSE
    RETURN format_bytes(bytes);
  END IF;
END$$
```

**只做一件事：NULL 透传**，其余交给内置 `format_bytes()`。

★ 注释解释了为什么参数是 `TEXT` 而不是 `BIGINT UNSIGNED`：**`SUM()` 聚合字节数可能超过 BIGINT UNSIGNED 上界**（P_S 字节计数器在多实例累加时可能溢出整数域），用 TEXT 走字符串/浮点路径规避。

**真正的阈值逻辑在 C++**（`sql/item_pfs_func.cc`）：

```cpp
bool Item_func_pfs_format_bytes::resolve_type(THD *) {
  /* Format is 'AAAA.BB UUU' = 11 characters or 'AAAA bytes' = 10 characters. */
  set_data_type_string(11U, &my_charset_utf8mb3_general_ci);
  return false;
}

String *Item_func_pfs_format_bytes::val_str(String *) {
  double bytes = args[0]->val_real();

  null_value = args[0]->null_value;
  if (null_value) {
    return error_str();
  }

  /* Declaring 'volatile' as workaround for 32-bit optimization bug. */
  volatile double bytes_abs = std::abs(bytes);

  constexpr uint64_t kib{1024};
  constexpr uint64_t mib{1024 * kib};
  constexpr uint64_t gib{1024 * mib};
  constexpr uint64_t tib{1024 * gib};
  constexpr uint64_t pib{1024 * tib};
  constexpr uint64_t eib{1024 * pib};

  uint64_t divisor;
  int len;
  const char *unit;

  if (bytes_abs >= eib)      { divisor = eib; unit = "EiB"; }
  else if (bytes_abs >= pib) { divisor = pib; unit = "PiB"; }
  else if (bytes_abs >= tib) { divisor = tib; unit = "TiB"; }
  else if (bytes_abs >= gib) { divisor = gib; unit = "GiB"; }
  else if (bytes_abs >= mib) { divisor = mib; unit = "MiB"; }
  else if (bytes_abs >= kib) { divisor = kib; unit = "KiB"; }
  else                       { divisor = 1;    unit = "bytes"; }

  if (divisor == 1) {
    len = sprintf(m_value_buffer, "%4d %s", (int)bytes, unit);
  } else {
    double value = bytes / divisor;
    if (std::abs(value) >= 100000.0) {
      len = sprintf(m_value_buffer, "%4.2e %s", value, unit);
    } else {
      len = sprintf(m_value_buffer, "%4.2f %s", value, unit);
    }
  }

  m_value.set(m_value_buffer, len, &my_charset_utf8mb3_general_ci);
  return &m_value;
}
```

**★ 回答"1024 还是 1000"：是 1024（二进制）**，且单位名写 `KiB/MiB/GiB` 而非 `KB/MB` —— 这是刻意的，IEC 标准里 `KiB`=1024 字节、`KB`=1000 字节，sys 用 IEC 记名避免歧义。

其它要点：
- **阈值"从大到小逐档 `>=`"** ⇒ 1023 字节显示 `1023 bytes`，1024 字节显示 `1.00 KiB`
- **双重 `abs`**：保证负数（P_S 计数器理论上可能下溢）也能正确分档
- **★ `volatile` 是 32 位编译器的 workaround**（注释明说）：防 32 位 x87 浮点寄存器 80 位扩展精度导致比较结果不稳定
- **三分支格式化**：`<1KiB` 用整数 `%4d`；`≥1KiB 且 <100000` 用 `%4.2f`；`≥100000` 用**科学计数 `%4.2e`** 防输出宽度爆掉
- **`resolve_type` 预留 11 字符**：`'AAAA.BB UUU'` 最长 11，`'AAAA bytes'` 10 → 固定 11，避免客户端缓冲区抖动

#### ② `sys.format_time()` —— 皮秒换算，纯 SQL 版保留

```sql
CREATE DEFINER='mysql.sys'@'localhost' FUNCTION sys.format_time (
-- We feed in and return TEXT here, as aggregates of
-- picoseconds can return numbers larger than BIGINT UNSIGNED
        picoseconds TEXT
    )
    RETURNS TEXT CHARSET UTF8MB4
    SQL SECURITY INVOKER
    DETERMINISTIC
    NO SQL
BEGIN
  IF picoseconds IS NULL THEN RETURN NULL;
  ELSEIF picoseconds >= 604800000000000000 THEN RETURN CONCAT(ROUND(picoseconds / 604800000000000000, 2), ' w');
  ELSEIF picoseconds >= 86400000000000000  THEN RETURN CONCAT(ROUND(picoseconds / 86400000000000000, 2),  ' d');
  ELSEIF picoseconds >= 3600000000000000   THEN RETURN CONCAT(ROUND(picoseconds / 3600000000000000, 2),   ' h');
  ELSEIF picoseconds >= 60000000000000     THEN RETURN CONCAT(ROUND(picoseconds / 60000000000000, 2),     ' m');
  ELSEIF picoseconds >= 1000000000000      THEN RETURN CONCAT(ROUND(picoseconds / 1000000000000, 2),      ' s');
  ELSEIF picoseconds >= 1000000000         THEN RETURN CONCAT(ROUND(picoseconds / 1000000000, 2),         ' ms');
  ELSEIF picoseconds >= 1000000            THEN RETURN CONCAT(ROUND(picoseconds / 1000000, 2),            ' us');
  ELSEIF picoseconds >= 1000               THEN RETURN CONCAT(ROUND(picoseconds / 1000, 2),               ' ns');
  ELSE RETURN CONCAT(picoseconds, ' ps');
  END IF;
END$$
```

**解释**：阈值全写成**字面量**而非 `60*1000000000000` 表达式——因为这是 `NO SQL DETERMINISTIC` 函数，每次调用都要跑，预计算常量省掉 9 次乘法。档位：周/天/时/分/秒/毫秒/微秒/纳秒/皮秒，保留 2 位小数。

**P_S 所有 `TIMER_WAIT` 列的单位统一是皮秒（1e-12 s）**，换算链 `1ns=1000ps`、`1us=1000ns`… **全 1000 进制**（时间用 SI，字节用 1024 —— **两者进制不同，这是 sys 里最容易记混的一点**）。

内置版（`format_pico_time()`）**没有"周"这一档**，且分钟写作 `min`（与 `ms` 区分）：

```cpp
  constexpr uint64_t nano{1000};
  constexpr uint64_t micro{1000 * nano};
  constexpr uint64_t milli{1000 * micro};
  constexpr uint64_t sec{1000 * milli};
  constexpr uint64_t min{60 * sec};
  constexpr uint64_t hour{60 * min};
  constexpr uint64_t day{24 * hour};

  /* Declaring 'volatile' as workaround for 32-bit optimization bug. */
  volatile double time_abs = std::abs(time_val);

  if (time_abs >= day)       { divisor = day;   unit = "d"; }
  else if (time_abs >= hour) { divisor = hour;  unit = "h"; }
  else if (time_abs >= min)  { divisor = min;   unit = "min"; }
  else if (time_abs >= sec)  { divisor = sec;   unit = "s"; }
  else if (time_abs >= milli){ divisor = milli; unit = "ms"; }
  else if (time_abs >= micro){ divisor = micro; unit = "us"; }
  else if (time_abs >= nano) { divisor = nano;  unit = "ns"; }
  else                       { divisor = 1;     unit = "ps"; }
```

#### ③ `format_statement()` —— ★ 保留头尾、砍中间

```sql
BEGIN
  -- Check if we have the configured length, if not, init it
  IF @sys.statement_truncate_len IS NULL THEN
      SET @sys.statement_truncate_len = sys_get_config('statement_truncate_len', 64);
  END IF;

  IF CHAR_LENGTH(statement) > @sys.statement_truncate_len THEN
      RETURN REPLACE(CONCAT(LEFT(statement, (@sys.statement_truncate_len/2)-2), ' ... ',
                            RIGHT(statement, (@sys.statement_truncate_len/2)-2)), '\n', ' ');
  ELSE 
      RETURN REPLACE(statement, '\n', ' ');
  END IF;
END$$
```

**截断规则**：
- **★ 保留头尾、砍中间**：`LEFT(x, len/2-2)` + `' ... '` + `RIGHT(x, len/2-2)`。默认 len=64 → 头 30 + ` ... `(5) + 尾 30。为什么留两头？SQL 的**头部**是动词和表名（`SELECT ... FROM`），**尾部**常带 `WHERE`/`ORDER BY` 过滤条件——中间往往是长长的列清单，最可牺牲。这比 `LEFT(x,64)` 信息量大得多。
- **`-2`** 是给 `' ... '` 两侧空格留位置，使总长接近 `len`。
- **`REPLACE(..., '\n', ' ')`** 把换行压成空格，保证 `\G` 纵向输出时一行显示完整。
- **`CHAR_LENGTH()` 而非 `LENGTH()`**：按**字符**计数，多字节 UTF-8 下不会截断出半个字符。
- **★ 会话级缓存配置**：只在**第一次调用**时查一次 `sys_config`，之后复用 `@sys.*`。注释明确说明这比每行 `IFNULL(@x, sys_get_config(...))` **快得多**。

#### ④ `format_path()` —— 路径脱敏（7 个目录）

```sql
BEGIN
  DECLARE v_path VARCHAR(512);
  DECLARE v_undo_dir VARCHAR(1024);
  DECLARE path_separator CHAR(1) DEFAULT '/';

  IF @@global.version_compile_os LIKE 'win%' THEN
    SET path_separator = '\\';
  END IF;

  -- OSX hides /private/ in variables, but Performance Schema does not
  IF in_path LIKE '/private/%' THEN
    SET v_path = REPLACE(in_path, '/private', '');
  ELSE
    SET v_path = in_path;
  END IF;

  SET v_undo_dir = IFNULL((SELECT VARIABLE_VALUE FROM performance_schema.global_variables
                            WHERE VARIABLE_NAME = 'innodb_undo_directory'), '');

  IF v_path IS NULL THEN
    RETURN NULL;
  ELSEIF v_path LIKE CONCAT(@@global.datadir, IF(SUBSTRING(@@global.datadir, -1) = path_separator, '%', CONCAT(path_separator, '%'))) ESCAPE '|' THEN
    SET v_path = REPLACE(v_path, @@global.datadir, CONCAT('@@datadir', IF(SUBSTRING(@@global.datadir, -1) = path_separator, path_separator, '')));
  ELSEIF v_path LIKE CONCAT(@@global.tmpdir, ...) ESCAPE '|' THEN
    SET v_path = REPLACE(v_path, @@global.tmpdir, CONCAT('@@tmpdir', ...));
  /* ... replica_load_tmpdir / innodb_data_home_dir / innodb_log_group_home_dir /
         innodb_undo_directory / basedir 同构 ... */
  END IF;

  RETURN v_path;
END$$
```

**解释**：
- **两个目的**：① 脱敏/缩短（`/data/3306/data/ibdata1` → `@@datadir/ibdata1`，**不泄露部署路径**，便于贴工单）；② **跨实例可比**（不同机器 datadir 不同，脱敏后可对比）。
- **★ `ESCAPE '|'`**：路径可能含 `_` 和 `%`（LIKE 通配符），指定路径里几乎不可能出现的 `|` 作转义符，保证按**字面量**匹配前缀。容易忽略但必须有。
- **`IF(SUBSTRING(dir,-1) = path_separator, '%', CONCAT(path_separator,'%'))`**：datadir 可能带也可能不带尾部分隔符（`/data/` vs `/data`），统一处理避免匹配到 `/dataXxx`。**7 个分支全部重复此模式**，故代码冗长但结构一致。
- **★ macOS 特例**（注释原话 "OSX hides /private/ in variables, but Performance Schema does not"）：macOS 上 `@@datadir` 报 `/var/...` 而 P_S 存真实路径 `/private/var/...`，**两者对不上**，先剥 `/private` 才能匹配。
- **`innodb_undo_directory` 特殊处理**：只在启用独立 undo 表空间时才设置，故用 `IFNULL((SELECT ...), '')` 查（不能直接 `@@global.`）。

#### ⑤ ★ `sys_get_config()` —— sys 的配置机制全貌

```sql
CREATE DEFINER='mysql.sys'@'localhost' FUNCTION sys_get_config (
        in_variable_name VARCHAR(128),
        in_default_value VARCHAR(128)
    )
    RETURNS VARCHAR(128)
    SQL SECURITY INVOKER
    DETERMINISTIC
    READS SQL DATA
BEGIN
    DECLARE v_value VARCHAR(128) DEFAULT NULL;

    -- Check if we have the variable in the sys.sys_config table
    SET v_value = (SELECT value FROM sys.sys_config WHERE variable = in_variable_name);
  
    -- Protection against the variable not existing in sys_config
    IF (v_value IS NULL) THEN
        SET v_value = in_default_value;
    END IF;

    RETURN v_value;
END$$
```

**★ 注意：函数本身不读 `@sys.*`**。真正的优先级链是**调用方**拼出来的三段式：

```
1. @sys.<name>          会话变量（最高优先级，调用方 IF 判断）
2. sys.sys_config 表     全局持久配置
3. 硬编码 default        代码兜底
```

调用方标准写法：

```sql
IF (@sys.xxx IS NULL) THEN
    SET @sys.xxx = sys.sys_get_config('xxx', '默认值');
END IF;
```

**其它要点**：
- **★ 用标量子查询而非 `SELECT INTO`**：变量不存在时标量子查询返回 NULL（而非 1329 NO_DATA_FOUND 异常），`IF v_value IS NULL` 直接兜底。**这是故意不用 `INTO` 的原因**。
- **命名约定 `@sys.<name>`**：注释明确——"约定是把用户变量命名成 `@sys.<配置名>`，而 `<配置名>` 才是存进 `sys_config` 表、传给函数的参数"。既避免污染用户变量命名空间，又能一眼看出来源。
- **`v_value` 是 VARCHAR(128)** ⇒ **所有 sys 配置都是字符串**（布尔用 `'ON'`/`'OFF'`，数字用 `'100'`），调用方自行转换。

#### ⑥ `sys.ps_thread_id()` —— 两套 ID 之间的桥

```sql
CREATE DEFINER='mysql.sys'@'localhost' FUNCTION sys.ps_thread_id (
        in_connection_id BIGINT UNSIGNED
    ) RETURNS BIGINT UNSIGNED
    SQL SECURITY INVOKER
    NOT DETERMINISTIC
    READS SQL DATA
BEGIN
  IF (in_connection_id IS NULL) THEN
    RETURN ps_current_thread_id();
  ELSE
    RETURN ps_thread_id(in_connection_id);
  END IF;
END$$
```

**解释**：
- **纯转发**（8.0 下沉为内置函数后的形态）。
- **"传 NULL = 查自己"** 是 sys 的 API 惯例，让一个函数覆盖两个用例。
- **`NOT DETERMINISTIC` + `READS SQL DATA`**（与 `format_*` 的 `DETERMINISTIC NO SQL` 对比）：它要读 P_S 的 `threads` 表，结果随连接状态变化，这个声明影响优化器是否敢缓存结果。
- ★ **为什么需要它**：P_S 内部用 `THREAD_ID`（单调递增、P_S 专用），而 `SHOW PROCESSLIST` / `KILL` 用的是 `PROCESSLIST_ID`（即 `CONNECTION_ID()`）。**两套 ID 并存是 8.0 最常见的困惑点**，这个函数就是那座桥。

## 六、存储过程体系（26 个）

### 6.1 完整清单

| 族 | 过程 |
|---|---|
| **PFS 开关控制 `ps_setup_*`** (17) | `ps_setup_enable/disable_instrument`、`_consumer`、`_thread`、`_background_threads`、`ps_setup_reset_to_default`、`ps_setup_save`、`ps_setup_reload_saved`、`ps_setup_show_enabled/disabled`（+ `_consumers` / `_instruments` 变体） |
| **追踪/诊断** (3) | `ps_trace_thread`、`ps_trace_statement_digest`、`ps_statement_avg_latency_histogram` |
| **报告生成** (2) | `diagnostics`、`statement_performance_analyzer` |
| **工具** (3) | `create_synonym_db`、`execute_prepared_stmt`、`table_exists` |
| **维护** (1) | `ps_truncate_all_tables` |

### 6.2 ★ `ps_setup_*` 与"直接 UPDATE setup_instruments"的真实区别

`ps_setup_enable_instrument.sql`：

```sql
BEGIN
    UPDATE performance_schema.setup_instruments
       SET enabled = 'YES', timed = 'YES'
     WHERE name LIKE CONCAT('%', in_pattern, '%');
    SELECT CONCAT('Enabled ', @rows := ROW_COUNT(), ' instrument', IF(@rows != 1, 's', '')) AS summary;
END$$
```

五点区别：

1. **模式匹配**（`LIKE '%pattern%'`）——一次开一类（如 `wait/synch/mutex` 开上百个）
2. **同时设 `enabled` 和 `timed`**——手工 UPDATE 很容易只改 `enabled` 忘 `timed`，结果"开了但不计时"
3. **返回摘要**（`ROW_COUNT()` + 单复数）——交互式运维要知道"刚才改了多少行"
4. **★ 事务性的保存/回滚**（手工 UPDATE 做不到）：
   - `ps_setup_save()` 取**命名锁** `GET_LOCK('sys.ps_setup_save', ...)`，把 `setup_actors` / `setup_consumers` / `setup_instruments` / `threads` 四张表快照到临时表
   - `ps_setup_reload_saved()` 写回
   - 注释：*"Saves the current configuration of Performance Schema, so that you can alter the setup for debugging purposes, but restore it to a previous state."*
5. **★ `ps_setup_reset_to_default()` 处理"出厂默认值"这个手算不出来的东西**：
   ```sql
   SET @query = 'UPDATE performance_schema.setup_instruments
                    SET ENABLED = sys.ps_is_instrument_default_enabled(NAME),
                        TIMED   = sys.ps_is_instrument_default_timed(NAME)';
   ```
   —— **每个 instrument 的出厂默认开关逐条不同**（存在代码里，不在表里），必须靠 `ps_is_instrument_default_enabled()` / `_timed()` 逐行问。它还依次重置 `setup_actors`、`setup_instruments`、`setup_consumers`、`setup_objects`、`threads` 五张表。

⚠️ 这些过程都 `SET sql_log_bin = 0`（需要 SUPER），因为调试动作不该复制到从库。

### 6.3 `sys.diagnostics()` —— 一次采集什么

（55 KB，最大的过程）

```
   * The GLOBAL VARIABLES
   * Several sys schema views including metrics or equivalent
   * Queries in the 95th percentile
   * Several ndbinfo views for MySQL Cluster（游标显式排除了 blocks / config_params /
     dict_obj_types / disk_write_speed_base / memory_per_fragment / memoryusage /
     operations_per_fragment / threadblocks）
   * Replication (both master and slave) information.
```

- **三种视图时态**：initial（开始时的快照，需 `@sys.diagnostics.include_raw='ON'`）/ overall（结束时，**总是包含**）/ delta（差值）
- **三档自动配置** `in_auto_config`：`current`（不动配置）/ `medium`（开部分，需 SUPER）/ `full`（全开，注释警告 *"This will have a big impact on the performance - be careful using this option"*），非 `current` 时结束后自动还原
- **自我保护**：开头就把自己排除在采集之外
  ```sql
  SELECT INSTRUMENTED INTO v_this_thread_enabled FROM performance_schema.threads
   WHERE PROCESSLIST_ID = CONNECTION_ID();
  IF (v_this_thread_enabled = 'YES') THEN CALL sys.ps_setup_disable_thread(CONNECTION_ID()); END IF;
  ```
- 用法（注释原文）：`mysql> TEE diag.out; CALL sys.diagnostics(120, 30, 'current'); NOTEE;`

### 6.4 其它值得一提的

- **`ps_statement_avg_latency_histogram()`** —— 纯 SQL 画 ASCII 直方图（`.`=1、`*`=2、`#`=3 单位），内部依赖 `x$ps_digest_avg_latency_distribution`
- **`ps_trace_thread(...)`** —— 输出 **DOT 图文件**，注释直接给渲染命令：`dot -Tpdf -o /tmp/stack_25.pdf ...`
- **`ps_trace_statement_digest(...)`** —— 追踪某 DIGEST 并对最长样例尝试 EXPLAIN；注释列出 EXPLAIN 会失败的三种情况（PFS 截断长 SQL_TEXT / 默认 schema 是 sys / SHOW 不支持 EXPLAIN），失败静默忽略
- **`statement_performance_analyzer(...)`** —— 7 种动作（`snapshot`/`overall`/`delta`/`create_table`/`create_tmp`/`save`/`cleanup`），做**基线 vs 当前**的语句性能 diff
- **`ps_truncate_all_tables()`** —— 扫 PFS 下 `LIKE '%summary%' OR LIKE '%history%'` 的表逐个 TRUNCATE（重置所有聚合）
- **`create_synonym_db()`** —— 给 PFS/I_S 建短名别名库（`CALL sys.create_synonym_db('performance_schema','ps')`）

---

### 6.5 ★ `ps_is_consumer_enabled()`：consumer 的父子依赖（理解"为什么看不到数据"的关键）

这个函数把 PFS consumer 的**层级依赖**硬编码成了 CASE 表达式，是理解"开了 instrument 还是没数据"的钥匙：

```sql
BEGIN
    DECLARE v_is_enabled ENUM('YES', 'NO') DEFAULT NULL;
    DECLARE v_error_msg VARCHAR(128);

    -- Return NULL for a NULL argument.
    IF (in_consumer IS NULL) THEN
        RETURN NULL;
    END IF;

    SET v_is_enabled = (
        SELECT (CASE
                   WHEN c.NAME = 'global_instrumentation' THEN c.ENABLED
                   WHEN c.NAME = 'thread_instrumentation' THEN IF(cg.ENABLED = 'YES' AND c.ENABLED = 'YES', 'YES', 'NO')
                   WHEN c.NAME LIKE '%\_digest'           THEN IF(cg.ENABLED = 'YES' AND c.ENABLED = 'YES', 'YES', 'NO')
                   WHEN c.NAME LIKE '%\_current'          THEN IF(cg.ENABLED = 'YES' AND ct.ENABLED = 'YES' AND c.ENABLED = 'YES', 'YES', 'NO')
                   ELSE IF(cg.ENABLED = 'YES' AND ct.ENABLED = 'YES' AND c.ENABLED = 'YES'
                           AND ( SELECT cc.ENABLED FROM performance_schema.setup_consumers cc WHERE NAME = CONCAT(SUBSTRING_INDEX(c.NAME, '_', 2), '_current')
                               ) = 'YES', 'YES', 'NO')
                END) AS IsEnabled
          FROM performance_schema.setup_consumers c
               INNER JOIN performance_schema.setup_consumers cg
               INNER JOIN performance_schema.setup_consumers ct
         WHERE cg.NAME       = 'global_instrumentation'
               AND ct.NAME   = 'thread_instrumentation'
               AND c.NAME    = in_consumer
        );

    IF (v_is_enabled IS NOT NULL) THEN
        RETURN v_is_enabled;
    ELSE
        -- A value of NULL here means it is an unknown consumer name that was passed as an argument.
        -- Only an input value of NULL is allowed to return a NULL result value, to throw a signal instead.
        SET v_error_msg = CONCAT('Invalid argument error: ', in_consumer, ' in function sys.ps_is_consumer_enabled.');
        SIGNAL SQLSTATE 'HY000'
           SET MESSAGE_TEXT = v_error_msg,
               MYSQL_ERRNO  = 3047;
    END IF;
END$$
```

**★ consumer 的依赖层次**（从 CASE 反推出的完整结构）：

```
global_instrumentation                       ← 总开关，无依赖
   ├── thread_instrumentation                ← 依赖 global
   │      ├── *_current                       ← 依赖 global + thread
   │      │      └── *_history / *_history_long
   │      │              ← 依赖 global + thread + 对应的 _current
   └── *_digest                               ← 依赖 global（不依赖 thread！）
```

| consumer 类别 | 生效条件（全部为 YES 才算 YES） |
|---|---|
| `global_instrumentation` | 自己 |
| `thread_instrumentation` | global + 自己 |
| `*_digest` | global + 自己 |
| `*_current` | global + thread + 自己 |
| `*_history` / `*_history_long` | global + thread + **对应的 `_current`** + 自己 |

**逐段解释**：

- **★ 最后那个 ELSE 分支的"父 consumer 名"是怎么算出来的**：
  ```sql
  CONCAT(SUBSTRING_INDEX(c.NAME, '_', 2), '_current')
  ```
  以 `events_waits_history_long` 为例：`SUBSTRING_INDEX(...,'_',2)` 取前两段 → `events_waits`，再拼 `_current` → `events_waits_current`。**这就是"history 依赖同名 current"的由来**。

- **为什么用两个 `INNER JOIN` 交叉连接再在 WHERE 里限定 `cg.NAME`/`ct.NAME`**：这是把一个"常量查找"（取总开关状态）塞进同一条 SELECT 的技巧——相当于对 `setup_consumers` 表做两次自连接取常量行，避免写三个独立子查询。

- **★ NULL 语义的严格区分**（注释说得很清楚）：
  - 输入 NULL → 返回 NULL（合法）
  - 输入是**未知 consumer 名** → 结果 NULL → **抛 `SIGNAL SQLSTATE 'HY000'` + errno 3047**（`ER_INVALID_ARGUMENT`）
  
  即"只有输入 NULL 才允许返回 NULL"，其余 NULL 一律视为错误。这是防御式设计的好例子。

⇒ **实践意义**：当你"开了 instrument 却查不到数据"时，用这个函数逐个确认 consumer 链路上每一环：

```sql
SELECT sys.ps_is_consumer_enabled('global_instrumentation'),
       sys.ps_is_consumer_enabled('thread_instrumentation'),
       sys.ps_is_consumer_enabled('events_statements_current');
```

### 6.6 `ps_setup_save` 的命名锁

```sql
    DECLARE v_lock_result INT;
    ...
    SELECT GET_LOCK('sys.ps_setup_save', in_timeout) INTO v_lock_result;
```

**解释**：`ps_setup_save()` 会把当前 PFS 配置快照存到临时表，如果多个会话同时做"保存→修改→还原"，会互相覆盖 ⇒ 用 MySQL 的**命名锁** `GET_LOCK()` 串行化。锁名直接用函数名（`sys.ps_setup_save`），与 sys 的 `@sys.*` 命名约定一致。超时由 `in_timeout` 参数控制。

⇒ 这是 sys 里少数用到**显式锁**的地方——因为跨会话共享的是 PFS 的全局配置状态。

### 6.7 `quote_identifier()` —— 生成可执行 SQL 的基础

```sql
CREATE DEFINER='mysql.sys'@'localhost' FUNCTION quote_identifier (
        in_identifier TEXT
    )
    RETURNS TEXT
    SQL SECURITY INVOKER
    DETERMINISTIC
    NO SQL
BEGIN
    RETURN CONCAT('`', REPLACE(in_identifier, '`', '``'), '`');
END$$
```

**解释**：给标识符加反引号并把内部反引号**翻倍**转义（SQL 标准写法）。所以 `innodb_lock_waits` 的 `locked_table` 能输出 `` `test`.`t1` `` 这样可直接执行的字符串。

⇒ 这是 sys"**可执行输出**"设计的基石：视图不只给你看，还给你能复制粘贴就跑的 SQL。

---

### 6.8 ★ `ps_statement_avg_latency_histogram()`：纯 SQL 画 ASCII 直方图

它不返回表，而是**用字符串拼出一个 ASCII 直方图**。核心技巧是**在 SELECT 列表里用 `:=` 做链式赋值**——这是 PFS/sys 里最"极端"的一段 SQL：

```sql
SELECT CONCAT('\n',
       /* ... 前 14 个桶同构，此处省略 ... */
       @label := CONCAT(@label_inner := CONCAT('\n(', ROUND(@bucket_size / @unit_div), ' - ',
                                               ROUND((@bucket_size * 2) / @unit_div), @unit, ')'),
                        REPEAT(' ', @max_label_size - LENGTH(@label_inner)),
                        @count_in_bucket := IFNULL((SELECT SUM(cnt)
                                                      FROM sys.x$ps_digest_avg_latency_distribution
                                                     WHERE avg_us > 0 AND avg_us <= @bucket_size), 0)),
       REPEAT(' ', @max_label_len - LENGTH(@label)), '| ',
       IFNULL(REPEAT(IF(@count_in_bucket < (@one_unit := 40), '.',
                        IF(@count_in_bucket < (@two_unit := 80), '*', '#')),
                     IF(@count_in_bucket < @one_unit,
                        @count_in_bucket,
                        IF(@count_in_bucket < @two_unit,
                           @count_in_bucket / 2,
                           @count_in_bucket / 3))), ''),
       @label := CONCAT(@label_inner := CONCAT('\n(', ROUND((@bucket_size * 2) / @unit_div), ' - ',
                                               ROUND((@bucket_size * 3) / @unit_div), @unit, ')'),
                        /* ... 下一个桶 ... */),
       /* ... 共 16 个桶，逐个手写展开 ... */
       '');
```

桶宽与总量：

```sql
ROUND((@bucket_size := (SELECT ROUND((MAX(avg_us) - MIN(avg_us)) / (@buckets := 16)) AS size
                           FROM sys.x$ps_digest_avg_latency_distribution)), 0) ...
/* @total_queries := (SELECT SUM(cnt) FROM sys.x$ps_digest_avg_latency_distribution) */
```

**逐段解释**：

- **★ 桶宽怎么定**：`(MAX(avg_us) - MIN(avg_us)) / 16` —— 固定 **16 个桶**，极差等分。所以用 `x$ps_digest_avg_latency_distribution`（见 4.3 ④ 的直方图 helper）做输入，而不是原始 digest 表。
- **★ 三种字符表示三个量级**：`.` = 每单位 1 条、`*` = 每单位 2 条、`#` = 每单位 3 条。
  ```sql
  IF(@count_in_bucket < (@one_unit := 40), '.', IF(@count_in_bucket < (@two_unit := 80), '*', '#'))
  ```
  配合 `REPEAT(..., count/1 | count/2 | count/3)` —— **把条长压缩到 40 字符以内**，否则高频桶会把终端刷爆。这是"对数式"的朴素替代。
- **★ `@x := ...` 在 SELECT 列表里链式赋值**：MySQL 不保证 SELECT 列表中表达式的求值顺序，但**从左到右**实际是可靠的，sys 依赖这一点把 `@label` / `@count_in_bucket` / `@bucket_size` 一路传下去。**这是脆弱但有效的技巧**（官方也承认依赖实现细节）。
- **★ 16 个桶是手写展开的**——因为这是在**构造一个字符串**而不是结果集，SQL 里没有循环可用；存储过程也不能动态生成"列"。所以这段代码有 16 段几乎相同的重复。这是"SQL 表达力边界"的直观例子。
- **`REPEAT(' ', @max_label_size - LENGTH(@label_inner))`**：手工对齐标签宽度，让 `|` 竖线在一条直线上。

⇒ **为什么值得看**：它展示了纯 SQL 做数据可视化的极限——能用，但代价是 16 段重复代码 + 依赖求值顺序。这类"技巧"在 sys 里不少，理解它们有助于判断**哪些视图能放心用、哪些只是炫技**。

### 6.9 4.3 ⑧ 引用的 helper：`x$ps_schema_table_statistics_io`

`schema_table_statistics` 的"字节维度"来自这个 helper（`schema_table_statistics` 自己只提供次数维度）：

```sql
SELECT extract_schema_from_file_name(file_name) AS table_schema,
       extract_table_from_file_name(file_name) AS table_name,
       SUM(count_read) AS count_read,
       SUM(sum_number_of_bytes_read) AS sum_number_of_bytes_read,
       SUM(sum_timer_read) AS sum_timer_read,
       SUM(count_write) AS count_write,
       SUM(sum_number_of_bytes_write) AS sum_number_of_bytes_write,
       SUM(sum_timer_write) AS sum_timer_write,
       SUM(count_misc) AS count_misc,
       SUM(sum_timer_misc) AS sum_timer_misc
  FROM performance_schema.file_summary_by_instance
 GROUP BY table_schema, table_name;
```

**解释**：

- **★ 关键在 `extract_schema_from_file_name()` / `extract_table_from_file_name()`** 这两个函数：`file_summary_by_instance` 只有**文件**维度（`./test/t1.ibd`），没有 schema/table 列。这两个函数从路径里反解出库名和表名，才能与 `table_io_waits_summary_by_table` 的 `object_schema`/`object_name` JOIN 上。
- **这是 sys 的一条"粘合"范式**：PFS 的**文件级**表和**表级**表之间缺一个关联键，sys 用"路径解析函数"补上。
- ⚠️ 也因此它有**已知局限**：分区表、共享表空间（`ibdata1` 里多张表）无法从文件名反解出唯一的表，会聚合到一起。

---

### 6.10 ★ `sys.diagnostics()`：诊断 = 两次快照求差

它是 sys 里最大的存储过程（55 KB），核心范式是**"对一组 sys 视图做前后两次快照，再求差"**。

**第一步：对每张视图拍"开始快照"**（游标遍历 + 动态 SQL 建临时表）：

```sql
        c_sysviews_w_delta_loop: LOOP
            FETCH c_sysviews_w_delta INTO v_table_name;
            IF v_done THEN
                LEAVE c_sysviews_w_delta_loop;
            END IF;

            IF (@sys.debug = 'ON') THEN
                SELECT CONCAT('The following queries are for storing the initial content of ', v_table_name) AS 'Debug';
            END IF;

            CALL sys.execute_prepared_stmt(CONCAT('DROP TEMPORARY TABLE IF EXISTS `tmp_', v_table_name, '_start`'));
            CALL sys.execute_prepared_stmt(CONCAT('CREATE TEMPORARY TABLE `tmp_', v_table_name, '_start` SELECT * FROM `sys`.`x$', v_table_name, '`'));

            IF (@sys.diagnostics.include_raw = 'ON') THEN
                SET @sys.diagnostics.table_name = CONCAT('x$', v_table_name);
                EXECUTE stmt_gen_query USING @sys.diagnostics.table_name;
                -- If necessary add ORDER BY and LIMIT
                SELECT CONCAT(@sys.diagnostics.sql_select,
                              IF(order_by IS NOT NULL, CONCAT('\n ORDER BY ', REPLACE(order_by, '%{TABLE}', CONCAT('tmp_', v_table_name, '_start'))), ''),
                              IF(limit_rows IS NOT NULL, CONCAT('\n LIMIT ', limit_rows), '')
                       )
                  INTO @sys.diagnostics.sql_select
                  FROM tmp_sys_views_delta
                 WHERE TABLE_NAME COLLATE utf8mb4_0900_as_ci = v_table_name;
                SELECT CONCAT('Initial ', v_table_name) AS 'The following output is:';
                CALL sys.execute_prepared_stmt(@sys.diagnostics.sql_select);
            END IF;
        END LOOP;
        CLOSE c_sysviews_w_delta;
```

**逐段解释**：

- **★ `CREATE TEMPORARY TABLE tmp_<name>_start SELECT * FROM sys.x$<name>`** —— 这就是"快照"：把某个 sys 视图的当前内容**整表复制进临时表**。用的是 **`x$` 版**（原始值），因为后面要做**差值计算**，格式化后的字符串没法相减。
- **为什么必须用动态 SQL（`execute_prepared_stmt`）**：视图名来自游标变量 `v_table_name`，SQL 里**表名不能是变量**，只能拼字符串再 PREPARE/EXECUTE。这也是 `execute_prepared_stmt.sql` 存在的理由（sys 的动态 SQL 公共工具）。
- **★ `%{TABLE}` 占位符替换**：`order_by` 存在 `tmp_sys_views_delta` 表里，值是类似 `%{TABLE}.total_latency DESC` 的模板，执行时把 `%{TABLE}` 换成实际的临时表名。这让"每张视图自己的排序规则"变成**数据**而不是代码。
- **★ `include_raw` 门控原始数据输出**：默认 `OFF` ⇒ 只输出 delta；打开才输出"初始快照"原始行。这是防止输出爆炸的安全默认（见 3.6）。
- **采样间隔**：两次快照之间是 `SLEEP`（默认 60 秒），之后建 `tmp_<name>_end` 再求差 ⇒ **诊断报告里的数字都是"这段时间内的增量"**。

**第二步：求差并输出**（`tmp_sys_views_delta` 表驱动）：

```sql
    FROM tmp_sys_views_delta diag
```

⇒ `tmp_sys_views_delta` 是一张**元数据表**，每行描述一张要采集的视图（表名、SELECT 列表、order_by、limit_rows）。**"采集什么"是配置数据，不是硬编码**——这就是为什么 `diagnostics()` 能覆盖几十张视图而代码仍然可维护。

**为什么值得剖析**：
1. **"两次快照求差"是 PFS 分析的基本范式**（`SHOW STATUS`/PFS 的聚合表都是累计值，看增量才有意义）——`diagnostics()` 把它做成了自动化流程
2. **`x$` 视图在这里是刚需**：差值计算必须用原始数值
3. **元数据驱动**：把"采集哪些视图、怎么排序"放进表，代码只写一个循环

---

## 七、可观测性

### 系统变量与状态变量

sys schema **没有自己的系统变量**——配置走 `sys.sys_config` 表：

| 配置项 | 默认 | 作用 |
|---|---|---|
| `statement_truncate_len` | 64 | `format_statement` 截断长度 |
| `statement_performance_analyzer.limit` | 100 | 分析器返回行数 |
| `statement_performance_analyzer.view` | — | 分析器默认视图 |
| `diagnostics.include_raw` | OFF | `diagnostics()` 是否含 initial 视图 |
| `diagnostics.allow_i_s_tables` | OFF | 是否扫 I_S 表 |
| `ps_thread_trx_info.max_length` | 65535 | `ps_thread_trx_info()` 截断长度 |

改法：`UPDATE sys.sys_config SET value='128' WHERE variable='statement_truncate_len';`

⚠️ `sys_config` 的 insert/update 触发器会自动记 `set_by`（谁改的）。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 现在谁在跑、在等什么 | SQL | `sys.session`（★ 比 `SHOW PROCESSLIST` 锁更少、信息更多） |
| 含后台线程 | SQL | `sys.processlist`、`sys.session_ssl_status` |
| 谁阻塞了谁 | SQL | `sys.innodb_lock_waits`（★ 直接给 KILL 语句） |
| 最慢的语句 | SQL | `sys.statements_with_runtimes_in_95th_percentile` |
| 全表扫描的语句 | SQL | `sys.statements_with_full_table_scans` |
| IO 热点（按线程） | SQL | `sys.io_by_thread_by_latency` |
| 表级读写热点 | SQL | `sys.schema_table_statistics` |
| 哪类等待最久 | SQL | `sys.wait_classes_global_by_latency` |
| 内存占用 | SQL | `sys.memory_by_thread_by_current_bytes`（⚠️ memory instrument 默认关） |
| 用户/主机维度汇总 | SQL | `sys.user_summary` / `sys.host_summary` |
| 一次性全面体检 | 过程 | `CALL sys.diagnostics(120, 30, 'current')` |
| 临时打开某类 instrument | 过程 | `CALL sys.ps_setup_enable_instrument('wait/synch/mutex')` |
| 改完想还原 | 过程 | `CALL sys.ps_setup_save(10)` → ... → `CALL sys.ps_setup_reload_saved()` |
| **要数值（监控采集）** | SQL | **用 `x$` 视图**（如 `x$processlist`、`x$user_summary_by_statement_latency`） |

---

## 八、★ 工程实现技法

### 8.1 用 `ALGORITHM = TEMPTABLE` 换"物化优化"

辅助视图注释直接说明动机：

```sql
-- Having this view with ALGORITHM = TEMPTABLE means MySQL can use the optimizations for
-- materialized views to improve the overall performance.
```

⇒ **把中间结果拆成独立视图**，给优化器创造派生表物化 + 复用的机会。（代价：写临时表。）

### 8.2 用"锁定 definer + INVOKER"解决视图权限与存活

definer 是一个密码永不合法、权限位全 N 的账号 ⇒ 既保证 definer 永存（不报 1449），又不给视图任何额外权限；`SQL SECURITY INVOKER` 保证以调用者权限执行。

### 8.3 用"可配置函数"而非内联表达式

`format_statement` 从 `sys_config` 读截断长度并缓存到会话变量 ⇒ 一处配置影响所有视图；且用 `IF ... THEN ... END IF` 而非 `IFNULL()` 避免循环内反复查询（注释明确警告了性能）。

### 8.4 用 `x$` 视图族补偿"格式化"的代价

格式化把数值变成 TEXT（不可排序/比较/聚合），sys 的补偿是**每个视图都配一个 `x$` 数值版**，且内部消费一律用 `x$`。这是"面向人 + 面向机器"双轨设计。

### 8.5 幂等 DDL，让同一脚本可用于初始化与升级

全部 `CREATE DATABASE IF NOT EXISTS` / `CREATE OR REPLACE VIEW` / `DROP ... IF EXISTS` + `CREATE` / `INSERT IGNORE` ⇒ 一段脚本反复执行都安全。

### 8.6 `ps_setup_save` 的命名锁

`GET_LOCK('sys.ps_setup_save', timeout)` 保证同一时刻只有一个会话在保存配置；拿不到就明确报错（而不是静默覆盖）。

---

## 九、Misc（易混淆点）

| 混淆 | 澄清 |
|---|---|
| sys 是独立数据源？ | ❌ **不是**。它不存任何自己的数据，全是 PFS / I_S 之上的视图 |
| sys 需要单独安装？ | ❌ **8.0 自带**，`mysqld --initialize` 时自动创建，升级时若不存在会重建 |
| `sys.version` 能读到真实版本？ | ⚠️ 是**硬编码常量 `2.1.2`**；该视图已 deprecated；8.0 已取消版本化升级 |
| 有 `_raw` 后缀视图？ | ❌ **8.0.39 源码中不存在**（只有配置变量 `diagnostics.include_raw`）。源码无直接证据支持该命名 |
| 有 `format_number` 函数？ | ❌ **不存在**（22 个函数里没有） |
| `ps_current_thread_id` 是 sys 函数？ | ❌ 是 **8.0 服务器内建函数**（`sql/item_create.cc`） |
| sys 视图能排序/聚合数值列吗？ | ❌ 默认视图是**格式化字符串**，不可排序比较 ⇒ **要数值请用 `x$` 版** |
| 两个视图的 avg 口径一样吗？ | ⚠️ **不一样**：`io_by_thread_by_latency` 是 `AVG(avg_timer_wait)`（未加权）；`wait_classes_global_by_latency` 是 `SUM/SUM`（加权） |
| `schema_table_statistics` 的物理 IO 对分区表准吗？ | ❌ **不准**（`extract_*_from_file_name` 注释明说不支持分区表 / `DATA_DIRECTORY`） |
| 改了 sys 视图会被覆盖吗？ | ✅ **会**。下次升级整段重放，手工修改静默丢失 ⇒ 要自定义请另建库指向 `sys.x$*` |
| 调用 `ps_setup_*` 需要什么权限？ | **SUPER**（因为内部 `SET sql_log_bin = 0`） |

---

## 参考

- *MySQL 8.0 Reference Manual → sys Schema*（视图/函数/过程逐个说明）
- 源码：
  - 目录根 `/Users/morphyzhang/Documents/Code/mysql-server/scripts/sys_schema/`（161 个 .sql）
  - 视图 `views/p_s/*.sql`（91）、`views/i_s/*.sql`（10）、`views/version.sql`
  - 函数 `functions/*.sql`（22）、过程 `procedures/*.sql`（28 文件）
  - 安装 `CMakeLists.txt` / `run_gensysschema.cmake` / `scripts/comp_sql.cc` / `sql/sql_initialize.cc` / `sql/dd/impl/upgrade/server.cc`
  - 内建格式化函数注册 `sql/item_create.cc`（`FORMAT_BYTES` / `FORMAT_PICO_TIME` / `PS_THREAD_ID` / `PS_CURRENT_THREAD_ID`）
  - 权限 `scripts/sys_schema/init_priv.sql`；配置 `tables/sys_config*.sql` + `triggers/*.sql`
  - 5.7 兼容 `sys_57.sql` / `sys_57.txt` / `generate_sql_file_57.sh`
- [`pfs/pfs.md`](pfs/pfs.md)（PFS 框架）· [`pfs/pfs_events.md`](pfs/pfs_events.md)（事件表族）· [`i_s.md`](i_s.md)（I_S）· [`../README.md`](../README.md)（观测手段全景与决策树）
- [`logging/`](logging/)（日志类手段，与 sys 互补：sys 看实时、日志看事后）

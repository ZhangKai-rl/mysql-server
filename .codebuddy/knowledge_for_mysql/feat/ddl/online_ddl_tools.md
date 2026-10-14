# gh-ost / pt-osc 在线 DDL 工具深度解析

> 剖析 GitHub gh-ost 与 Percona pt-online-schema-change 两个在线 DDL 工具的核心机制：存量拷贝、增量同步（binlog 订阅 vs 触发器）、cut-over 切换、限流与故障处理。
>
> **边界**：本篇讲的是**外部工具**（事实来源为 gh-ost 与 Percona Toolkit 的源码与官方文档，非本仓库 mysql-server 源码）；MySQL 原生 DDL 算法（INSTANT/INPLACE/COPY 的锁行为、MDL 三段式、row log）见 [`ddl.md`](ddl.md)，本篇只留选型结论与指针。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [存量拷贝与 chunk 分片](#存量拷贝与-chunk-分片)
  - [增量同步：binlog 订阅 vs 触发器](#增量同步binlog-订阅-vs-触发器)
  - [cut-over：切换的艺术](#cut-over切换的艺术)
  - [统计信息：cut-over 前的强制 ANALYZE](#统计信息cut-over-前的强制-analyze)
  - [限流、暂停与外部协作](#限流暂停与外部协作)
  - [故障与回滚](#故障与回滚)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

gh-ost（GitHub 2016 开源，Go 实现）与 pt-osc（pt-online-schema-change，Percona Toolkit，Perl 实现）是第三方 MySQL 在线 schema 变更工具。两者的骨架相同——**影子表 + 存量拷贝 + 增量同步 + 原子切换**：建一张新结构的影子表，把存量数据拷进去，同步拷贝期间的增量变更，最后原子地交换表名。区别只在增量同步的手段：pt-osc 用**触发器**，gh-ost 用 **binlog 订阅**。

### 用途

解决原生 DDL 在大表上的可用性问题。原生算法做不到的几类场景由工具补位：改列类型/字符集这类只能 COPY（全程阻塞）的操作、需要限流/暂停/可中断重跑的迁移、希望在 replica 上先灰度验证再上生产、主从延迟敏感的大表 rebuild。一个反直觉的切入点是：`ALTER TABLE t MODIFY c INT` 在 10 亿行表上会全程锁表数小时，而 gh-ost 可以让整个迁移期间业务完全无感知，切换点阻塞只有几十毫秒。

### 版本演进

| 版本/时间 | 变化 |
|------|------|
| 2010 | Facebook online-schema-change 开源：触发器方案鼻祖（影子表 + AFTER 触发器同步增量） |
| 2011+ | Percona pt-online-schema-change：触发器方案工程化（chunk 分片、外键处理、限流、`--no-drop-old-table`） |
| 2016 | GitHub gh-ost：去触发器，改用 binlog 订阅增量（伪装 replica），新增可暂停/限流/`--test-on-replica` 灰度 |
| MySQL 5.6 | 原生 online DDL：INPLACE 支持并发 DML（row log 机制），工具使用场景开始收窄 |
| MySQL 8.0 | 原子 DDL + INSTANT（8.0.12 表尾加列，8.0.29 扩展），gh-ost 相应支持 `--attempt-instant-ddl` 优先走 INSTANT |

---

## 理论基础

### 设计思想与权衡

**三个核心决策，各自都有被放弃的替代方案：**

**决策一：影子表 + 原子切换，而不是就地修改。** 工具无法修改引擎内部结构（不能像 InnoDB 的 INPLACE 那样原地 rebuild），只能走"外部重建"：影子表隔离变更过程，最后一步原子换名。代价是**需要一整张表的额外磁盘空间**和一次全量拷贝——这正是原生 INPLACE 想省掉的，但工具换来的是与引擎解耦：任何引擎、任何 MySQL 版本、任何变更类型都能用。

**决策二（gh-ost 的立身之本）：用 binlog 订阅替代触发器。** pt-osc 的触发器方案有一个结构性代价：**写放大**——业务对原表的每条 DML 都要在触发器里多写一次影子表，事务变大、延迟上升、主从放大。gh-ost 换了一条路：伪装成 MySQL replica，用 `COM_BINLOG_DUMP` 协议订阅 binlog 流，把对原表的变更**在 gh-ost 进程内**重放到影子表。代价是：要求 `binlog_format=ROW`、需要一个持续运行的独立进程、实现复杂度高得多（要解析 binlog 事件、处理 UPDATE 改主键、处理与存量拷贝的竞态）。但收益是原表 DML **零额外开销**。

**决策三：单向同步（原表 → 影子表），而不是双向。** 影子表在切换之前不对外可见，业务只会写原表，所以只需要单向同步——这是工具正确性的关键简化。双向同步（影子表也往回同步）是很多自研工具掉进去的坑：双向要处理冲突、死循环（触发器触发触发器）、顺序依赖，复杂度爆炸。单向 + "切换前影子表无人可见" 这两个前提互相支撑，任何一个不成立整个设计都要重来。

**切换窗口的权衡（gh-ost magic lock）：** 最朴素的切换是"锁原表 → RENAME → 解锁"，但 RENAME 在锁内执行意味着阻塞时间包含 RENAME 自身的时间（受元数据锁排队影响，可能被前面的查询拖长）。gh-ost 的 magic lock 把顺序倒过来：**先让 RENAME 卡在元数据锁上排队，再放锁让它瞬间完成**——RENAME 语句在锁外预先发起并阻塞等待，放锁的瞬间它就拿到锁执行。这样切换点 = "释放锁"一个动作，阻塞时间被压到最小，而且 RENAME 本身是单条语句的原子双表换名，不存在中间状态。代价是引入了 magic sentry 表和一个"阻塞后确认"的复杂状态机（代码上用了四个 channel 协调两个会话）。

### 理论溯源

- **CDC（Change Data Capture）**：gh-ost 的 binlog 订阅是教科书式 CDC——把源表的变更流（MySQL binlog ROW 事件）捕获后重放到目标。落点在 `ApplyDMLEventQueries`：把一批 `BinlogDMLEvent` 转成对 ghost 表的 DML 批量执行。
- **元数据锁排队**：cut-over 的核心技巧依赖 MySQL MDL 的公平排队语义——一个语句持锁时，后续的 RENAME 会进入 PENDING 队列等待，锁释放后按序获得。gh-ost 通过 `performance_schema.metadata_locks` 确认 RENAME 已处于 PENDING 状态（`ExpectMetadataLock` 查 `lock_status='PENDING'` 且 `lock_type='EXCLUSIVE'`）。

### 算法与数据结构

- **chunk 分片**：存量拷贝按唯一键（主键优先）的范围分片，每片一个事务。pt-osc 用 `WHERE pk >= ? ORDER BY pk LIMIT N` 推进游标；gh-ost 的 `CalculateNextIterationRangeEndValues` 用 `LIMIT N,1`（offset 方式）或临时表方式定位每片的结束键值，再 `INSERT ... SELECT` 整片搬移。复杂度：全表扫一遍 O(N)，每片事务保证回滚粒度。
- **双队列单消费者（gh-ost）**：`copyRowsQueue`（存量拷贝）与 `applyEventsQueue`（binlog 增量）由同一个 `executeWriteFuncs` 消费，**增量队列优先**——存量拷贝可以慢慢跑，增量必须及时追上，否则切换时积压越大锁表越久。
- **magic sentry 表**：一张被 `LOCK TABLES ... WRITE` 锁住的占位表，占着旧表最终的名字（`_del`）。作用有二：让 RENAME 的目标名已被占用从而**必然阻塞**（而不是可能因目标名冲突直接报错）；作为"锁还持有"的锚点——gh-ost 通过检查这张表确保锁没丢。

### 他库对比与演进动机

Facebook OSC（2010）首创触发器方案，但存在几个痛点：触发器带来的写放大、表上已有触发器时冲突、外键场景复杂。pt-osc 继承并工程化了它（chunk 分片、限流、外键方法），成为触发器方案的事实标准。gh-ost 的演进动机明确写在它的设计文档里：**彻底消除触发器**——GitHub 当时的表规模下触发器开销不可接受，于是选择伪装 replica 订阅 binlog。三者关系是一脉相承的：触发器方案 → 触发器方案工程化 → 去触发器方案。PostgreSQL 没有等价的第三方工具生态（它的 ALTER TABLE 多数操作不重写全表，且 `pg_repack` 走的是物理重写 + 视图切换的路线），Oracle 有 Online Redefinition（DBMS_REDEFINITION，与影子表+增量同步同构，但由数据库内核提供）。

---

## 核心实现

### 主链路

gh-ost 的 `Migrate()` 是迁移总控，骨架如下：

```
Migrate
  → 解析 ALTER 语句、建 inspector（检查表结构/唯一键/外键）
  → 启动 streaming：伪装 replica 订阅 binlog（先于 applier，见下）
  → 启动 applier：建 changelog 表(_ghc) → 建 ghost 表(_gho) → 在 ghost 上执行 ALTER
  → （可选）--attempt-instant-ddl：8.0 下先试 ALGORITHM=INSTANT，成功直接收工
  → 读迁移范围（min/max 键值），启动双 goroutine：
      executeWriteFuncs（消费两个队列）+ iterateChunks（产出拷贝任务）
  → 存量拷贝完成 → 增量追平 → cutOver → 清理
```

pt-osc 的主链路同构但更线性：

```
main
  → 检查约束（主键/唯一键、外键、已有触发器冲突）
  → 建影子表 _new（新结构）
  → 建三个 AFTER 触发器（把原表 DML 镜像到 _new）
  → chunk 循环拷贝存量
  → RENAME TABLE 原子交换 → 删触发器 → 删旧表
```

一个容易被忽略的顺序细节：gh-ost 的 streaming **必须先于** applier 启动——因为"ghost 表已建好"这个状态本身是通过 changelog 表上的 binlog 事件（`GhostTableMigrated`）从 applier 传回主进程的，streaming 没启动就收不到这个事件，流程会卡死。

### 存量拷贝与 chunk 分片

gh-ost 的拷贝分两步：先算出本片的结束键值，再整片 `INSERT ... SELECT`：

```go
// 定位下一片的结束键值（两种策略：offset 法 / 临时表法）
buildFunc := sql.BuildUniqueKeyRangeEndPreparedQueryViaOffset
if i == 1 {
    buildFunc = sql.BuildUniqueKeyRangeEndPreparedQueryViaTemptable
}
// 查询形如：SELECT col1..coln FROM tbl
//           WHERE (col1..coln) > (min 值) ORDER BY col1..coln LIMIT chunk_size,1
```

为什么用 `LIMIT N,1` 取"第 N+1 行"做边界：直接读第 N+1 行的键值，比让 MySQL 估算更精确，且下一片的 `WHERE (keys) > (边界)` 可以走索引。临时表法是 offset 法的兜底——大 offset 时 LIMIT 偏移量扫行代价高，先把范围物化到临时表再定位。

chunk 大小是动态的：gh-ost 的 `--chunk-size` 默认 1000 行，但受 `--max-load` 限流影响实际会等；pt-osc 用 `--chunk-time`（默认 0.5s）让每片拷贝耗时自适应——片大就减小，片小就加大。

### 增量同步：binlog 订阅 vs 触发器

**gh-ost 侧**：订阅到的 ROW 事件被重放为对 ghost 表的 DML。UPDATE 有一个必须处理的特殊 case——**UPDATE 修改了唯一键列**：

```go
// 把修改唯一键的 UPDATE 拆成 DELETE（按旧键）+ INSERT（按新键）
if _, isModified := apl.updateModifiesUniqueKeyColumns(dmlEvent); isModified {
    results := make([]*dmlBuildResult, 0, 2)
    dmlEvent.DML = binlog.DeleteDML
    results = append(results, apl.buildDMLEventQuery(dmlEvent)...)   // 按旧键值删
    dmlEvent.DML = binlog.InsertDML
    results = append(results, apl.buildDMLEventQuery(dmlEvent)...)   // 按新键值插
    return results
}
```

为什么必须拆：UPDATE 的 WHERE 用的旧键值在 ghost 表里可能已经被存量拷贝改了（拷贝与增量并发），直接 `UPDATE ... SET pk=新 WHERE pk=旧` 在 ghost 表上语义不对。拆成删+插后两条语句各自独立成立。

重放时的"重复键是预期行为"：

```go
// Duplicate entry on migration unique key is expected during binlog replay
// (row was already copied during bulk copy phase)
if strings.Contains(message, "Duplicate entry") && migrationKeyRegex.MatchString(message) {
    continue    // 该行在存量拷贝阶段已写入，重放的 INSERT 撞键是正常的
}
```

这是存量拷贝与增量同步竞态的直接体现：binlog 里的一行可能已被 chunk 拷贝写过一次，重放时按主键写自然会撞——这不是错误而是"幂等收敛"。

**pt-osc 侧**：三个 AFTER 触发器，核心是**幂等设计**——触发器与 chunk 拷贝会写同一行，所有写入都带 IGNORE：

```sql
-- AFTER INSERT：新行镜像到影子表（INSERT IGNORE 容忍与 chunk 拷贝的撞键）
CREATE TRIGGER `pt_osc_db_t_ins` AFTER INSERT ON `db`.`t`
FOR EACH ROW INSERT IGNORE INTO `db`.`_t_new` (...) VALUES (NEW.col1, ...);

-- AFTER DELETE：删除影子表对应行（按主键，DELETE IGNORE 幂等）
CREATE TRIGGER `pt_osc_db_t_del` AFTER DELETE ON `db`.`t`
FOR EACH ROW DELETE IGNORE FROM `db`.`_t_new` WHERE `_t_new`.pk <=> OLD.pk;

-- AFTER UPDATE：先删旧行再插新行（UPDATE 不能 IGNORE，所以拆两步）
CREATE TRIGGER `pt_osc_db_t_upd` AFTER UPDATE ON `db`.`t`
FOR EACH ROW BEGIN
  DELETE IGNORE FROM `db`.`_t_new` WHERE !(OLD.pk <=> NEW.pk) AND `_t_new`.pk <=> OLD.pk;
  INSERT IGNORE INTO `db`.`_t_new` (...) VALUES (NEW.col1, ...);
END
```

三个设计点：`<=>` 是 NULL 安全的等值比较（主键含 NULL 时 `=` 失效）；UPDATE 拆成删+插两步且删的条件带上"主键确实变了"（`!(OLD.pk <=> NEW.pk)`，主键没变时删插同一行会白忙）；全部 IGNORE 化让"chunk 已拷过该行"与"触发器再写一次"的竞争退化为无害的警告。

两种增量同步的代价对比非常清晰：binlog 订阅对**原表零开销**（事件流本来就存在，gh-ost 只是旁路读取），触发器是**每条 DML 的写放大**（事务内多一次影子表写）。这就是 gh-ost 相对 pt-osc 的核心优势，也是它选择 CDC 路线的全部理由。

### cut-over：切换的艺术

cut-over 是两工具正确性最集中的地方。两种风格：

**pt-osc / gh-ost two-step 的朴素切换**：锁原表 → 改名（原表→`_old`，影子表→原名）→ 解锁。gh-ost 的 two-step 用两条 ALTER 语句改名，两条语句之间有一个"原表名不存在"的窗口：

```go
// 第一步：原表改名 _old
query := fmt.Sprintf(`alter table %s.%s rename %s`, db, OriginalTableName, GetOldTableName())
// 第二步：ghost 表顶上去 —— 两条语句之间的瞬间，业务查原表名会报"表不存在"
query = fmt.Sprintf(`alter table %s.%s rename %s`, db, GetGhostTableName(), OriginalTableName)
```

**gh-ost atomic（默认）的 magic lock 切换**：一条 RENAME 语句原子地完成双表换名（单语句多表 RENAME 在 MySQL 里是原子的），配合 sentry 表把切换点压到最小。流程分四个阶段：

```go
// 阶段一：另一个会话建 magic sentry 表并锁定（RENAME 的"目标名 _del"被 sentry 占用）
// sentry 表：create table _del (id int auto_increment primary key) comment='ghost-cut-over-sentry'
query = `lock tables db.t write, db._del write`     // 锁原表 + sentry 表
tableLocked <- nil                                   // 通知主流程"已锁住"

// 阶段二：主流程等 binlog 事件全部重放完（waitForEventsUpToLock）

// 阶段三：第三个会话发起原子 RENAME，预期阻塞在元数据锁上
query = `rename table db.t to db._del, db._gho to db.t`
// 此 RENAME 因 db.t 被锁而进入 MDL PENDING 队列

// 阶段四：确认 RENAME 已阻塞（PROCESSLIST / metadata_locks 查 PENDING 状态）
//         → DROP magic sentry 表 → UNLOCK TABLES → RENAME 瞬间完成
```

这段设计的精髓在注释里直接点破了："We know the original table is locked. We know any newly incoming DML on original table is blocked"——锁原表的瞬间，新 DML 全部阻塞，之后只需把锁前积压的 binlog 事件追平，切换点就安全了。RENAME **先卡住排队、后放锁执行**的顺序倒置，让切换阻塞时间 = 锁释放到 RENAME 拿到锁的时间，而不是 RENAME 自身的执行时间。magic sentry 表则是"锁还持有"的物理证据：sentry 被锁着且 RENAME 的目标名被它占着，RENAME 只能等——如果锁意外丢失，RENAME 会因目标名冲突直接失败而不是静默切错。

### 统计信息：cut-over 前的强制 ANALYZE

gh-ost 在 cut-over 前会强制对 ghost 表执行 `ANALYZE TABLE`，源码注释把动机写得非常清楚——这正是"重建表后统计信息为空"问题的直接答案：

```
// AnalyzeGhostTable runs an explicit ANALYZE TABLE on the ghost table, forcing a
// synchronous InnoDB persistent-statistics recompute before cut-over. Without it the
// freshly swapped table can serve traffic with a near-zero row estimate, which the
// optimizer costs as a free full scan — the failure mode motivating upstream #1419.
```

不 ANALYZE 的后果链条：ghost 表是新建的，统计信息是 0 行（或近 0）→ 切换后优化器认为这张"大表"几乎是空的 → 全表扫描被代价成免费 → 切完立刻大面积慢查询。这与原生 DDL 的规律一致（COPY 重建后统计不迁移、INPLACE rebuild 才重算），但工具把这个坑**显式处理**了：切换前同步重算一次统计，且 ANALYZE 失败视为致命错误拒绝切换（fail-closed，见 `classifyAnalyzeTableResult` 对 "status OK" 行的严格检查）。这一点上工具比原生 COPY 算法做得更完整。

### 限流、暂停与外部协作

**gh-ost 的限流体系**（围绕"迁移可随时暂停"设计）：

- `--max-load`：负载阈值（如 `Threads_running=30`），超过即暂停拷贝
- `--throttle-flag-file`：外部创建文件即暂停、删除即恢复——这是与切换窗口协作的关键：DBA 在低峰窗口删文件放行，高峰创建文件卡住
- `--throttle-query` + `--throttle-control-replicas`：轮询指定从库上某查询的结果（如 `SELECT max(Seconds_Behind_Master) ...`），超过阈值暂停——把"从库延迟"变成可编程的限流信号
- 心跳：gh-ost 周期性向 changelog 表写 heartbeat 行（`InitiateHeartbeat`），用"心跳写入时间 vs 被订阅到的延迟"来测复制延迟（`TimeSinceLastHeartbeatOnChangelog`）。cut-over 前检查这个延迟：心跳延迟超过 `--max-lag-millis` 就拒绝切换（见 `cutOver` 开头的 sleepWhileTrue 循环）
- `--postpone-cut-over-flag-file`：拷贝完成后不立即切换，等外部信号——"拷贝完成、人工确认后再切"的灰度和声模式

还有一层防并发：`AcquireMigrationLock` 用 `GET_LOCK('gh-ost::db.table')` 用户级锁，防止两个 gh-ost 进程同时迁同一张表。

**pt-osc 的限流**：`--max-load` / `--critical-load`（后者达到直接中止而非暂停）、`--chunk-time`（动态调整片大小）。

### 故障与回滚

**cut-over 之前失败**：原表完全无损。gh-ost 重跑时 `--initially-drop-ghost-table` 清掉残留的 ghost 表；pt-osc 同理清理残留影子表/触发器。

**cut-over 中崩溃**：atomic 模式下 RENAME 是单条语句，要么切完（旧表在 `_del`）要么没切（原表还在），没有中间态；two-step 模式若崩在两步之间，原表名不存在——此时旧表在 `_old`，需人工改回。这是 two-step 的固有风险，也是默认用 atomic 的原因。

**半同步下两阶段提交的丢数据防护**（gh-ost 的一个精巧 case）。`ReadMigrationRangeValues` 读拷贝范围前，先往 changelog 写一条状态，源码注释完整记述了动机：

```
// When using semi-sync and setting rpl_semi_sync_master_wait_point=AFTER_SYNC,
// if an INSERT statement is being committed but blocks due to an unmet ack count,
// the data inserted by the transaction is not visible to ReadMigrationRangeValues,
// so the copy of the existing data in the table does not include the new row.
// However, the binlog event for the transaction is already written to the binlog,
// so the addDMLEventsListener only captures the binlog event after the transaction,
// and thus the transaction's binlog event is not captured, resulting in data loss.
```

即：事务在半同步 ack 等待中时，行数据对拷贝不可见（未提交），但 binlog 已写出且 gh-ost 的监听游标已越过它——拷贝漏掉这行、增量也漏掉，数据丢失。解法：先写 changelog（在 binlog 流中形成一道栅栏），再读拷贝范围——changelog 写入成功意味着流中的并发事务都已提交可见，读到的范围就不会漏行。

**从库延迟**：pt-osc 的触发器写放大在 ROW 格式下会随复制重放到从库，主从延迟被放大；gh-ost 的增量重放只发生在 gh-ost 所在实例，从库只收到正常 DML——但切换瞬间 ghost 表全量数据变更会作为 RENAME + 数据以 binlog 形式从执行实例流向从库，是一次性的大流量。

---

## 可观测性

### 系统变量与状态变量（MySQL 侧前置条件）

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `binlog_format` | ROW | Both | gh-ost 硬性前提，非 ROW 直接启动失败 |
| `log_slave_updates` | ON | Global | 在 replica 上跑 gh-ost 时 replica 必须写 binlog |
| `lock_wait_timeout` | 31536000 | Both | cut-over 的短锁等待受此影响，gh-ost 会话内单独设短超时 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 迁移进度 | gh-ost 状态输出 | `--verbose` 或交互式状态屏（拷了多少行/多少 DML 事件待追平） |
| 当前是否被限流 | gh-ost 状态 | `throttle` 状态行；changelog 表的 throttle hint |
| 切换是否安全 | 心跳延迟 | gh-ost cut-over 前日志（heartbeat lag 与 `--max-lag-millis` 对比） |
| 影子表统计是否正确 | SQL | 切换后 `SHOW TABLE STATUS` 或 `information_schema.tables` 看 ghost 表行数 |

### 工具参数（关键项）

- gh-ost：`--alter`、`--execute`（不加则 dry-run）、`--chunk-size`、`--max-load`、`--throttle-flag-file`、`--test-on-replica`、`--cut-over`(atomic|two-step)、`--postpone-cut-over-flag-file`、`--initially-drop-ghost-table`、`--attempt-instant-ddl`
- pt-osc：`--alter`、`--execute`（不加则 dry-run）、`--chunk-time`、`--max-load`、`--critical-load`、`--alter-foreign-keys-method`(auto|rebuild_constraints|drop_swap|none)、`--no-drop-old-table`、`--check-alter`

---

## Misc

### 工具切换与原子的边界

gh-ost/pt-osc 的 atomic cut-over 依赖"单条 RENAME 语句内多表改名原子"这一 MySQL 语义（8.0 原子 DDL 之前就已存在）。8.0 之后 RENAME 本身也纳入原子 DDL（见 [`ddl.md`](ddl.md)），但对工具而言这只是锦上添花——工具的原子性不依赖 8.0。

### 工具的"原子"与引擎原子 DDL 的区别

工具整体流程（拷贝+同步）的一致性由工具自身状态机保证，不是数据库事务；引擎原子 DDL 保证的是每条 DDL 语句（如 cut-over 那条 RENAME）崩溃安全。两者正交。

### 外键的两种命运

pt-osc 默认把外键重建到影子表（`rebuild_constraints`），因为影子表是"新表"，旧表的外键不会自动跟随；gh-ost 对外键的处理依赖 `--skip-foreign-key-checks` 期间的窗口配合。外键是触发器方案和 binlog 方案共同的麻烦点，也是选型时最常被忽略的坑。

---

## 参考

**官方文档**
- [gh-ost 官方仓库与文档](https://github.com/github/gh-ost)（cut-over 设计、`--test-on-replica`、throttle 机制；本文代码引自 master 分支 `go/logic/migrator.go` 与 `go/logic/applier.go`）
- [Percona Toolkit 文档：pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)（触发器机制、`--alter-foreign-keys-method`、`--max-load`）

**技术文章**
- [gh-ost: GitHub's online schema migration tool for MySQL](https://github.blog/2016-08-01-gh-ost-github-s-online-schema-migration-tool-for-mysql/)（GitHub Engineering 博客，去触发器动机的原始论述）
- gh-ost issue [#1419](https://github.com/github/gh-ost/issues/1419)：切换后统计信息近零导致全表扫描（AnalyzeGhostTable 的动机来源）

**相关文档**
- 原生 DDL 算法（INSTANT/INPLACE/COPY 锁行为、MDL 三段式、row log、原子 DDL、重建后统计处理）见 [`ddl.md`](ddl.md)

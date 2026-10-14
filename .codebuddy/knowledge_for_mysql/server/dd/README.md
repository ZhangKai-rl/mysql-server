# dd/ —— 数据字典

> 一句话定位：**MySQL 里"元数据存在哪、怎么被加载、两份字典如何同步"**的唯一出处。
> 8.0 有**两套并存的字典**（server 层权威 DD + InnoDB 内部运行时字典），辨析见 [`dd.md`](dd.md) 文首的「两套并存的字典」表。

## 文件索引

| 文件 | 内容 |
|---|---|
| [dd.md](dd.md) | server 层 8.0 数据字典：DD 表体系、对象模型、升级三路径（**SDI 不在本篇**——见下） |
| ⭐ [`../../feat/sdi.md`](../../feat/sdi.md) | **SDI（Serialized Dictionary Information）已独立成篇**，放 `feat/` 因为它**横跨 server ↔ InnoDB 两层**且是 8.0 引入的跨层特性（与 `fts.md` 同性质）：server 定序列化格式（`sdi.cc`）、InnoDB 提供存储（`dict0sdi.cc` 的 `SDI_<space_id>` B-tree、`ibd2sdi`）。★ 服务器**只写不读**（`dict_sdi_get`/`_drop` 被 `#if 0`）；★ `.sdi` 文件按 **`hton->sdi_set` 能力**分派而非 `innodb_read_only`；★ `sdi_version` 全等 ⇒ 跨 minor 导入基本不可行 |
| [innodb_dict.md](innodb_dict.md) | InnoDB 内部字典：内部表、dict cache、动态元数据 |
| [statistics.md](statistics.md) | InnoDB 统计信息：采样算法、统计线程、ANALYZE |
| [stored_program_dd.md](stored_program_dd.md) | **存储程序与视图的 DD 侧持久化**：`sp_head`/`Table_ref`/`Event_parse_data`/`::Trigger` → `mysql.routines`/`triggers`/`events`/`tables` 的字段映射。★ 视图**没有** `mysql.views` 表（与表共用 `mysql.tables`）；★ 视图依赖是**名字级**而触发器是 `table_id` 外键；★ 时区 UTC 化不在 `dd_event.cc`；★ `originator` + `SLAVESIDE_DISABLED` 是从库不重复执行的保障；三类写入器（被动填充/主动写入/子对象挂载）与 `dd::create_table` 的分工。与 [`../stored_program.md`](../stored_program.md)（SQL 层语义）是姊妹篇 |
| [system_views.md](system_views.md) | **I_S 的 DD 半边**：8.0 绝大多数 I_S 表（`COLUMNS`/`TABLES`/`STATISTICS`…）是**真视图**（`CREATE VIEW` 落进 DD），视图定义在 C++ 里用 builder 拼。43 张清单 + `SHOW` 重写 + `Table_statistics`（Cardinality 过期）。★ 与 [`../../feat/i_s.md`](../../infra/observability/i_s.md)（插件机制 + InnoDB 26 张表）是**姊妹篇**，合起来才是完整的 I_S |

## 与周边文档的边界

| 主题 | 不在本目录，见 |
|---|---|
| 表对象的生命周期（`TABLE_SHARE` / `TABLE` / 开表 / 锁表 / 表 DDL 操作） | [`../table.md`](../table.md) |
| DDL 的三种算法与执行（COPY / INPLACE / INSTANT、row log、DDL log） | [`../../feat/ddl/ddl.md`](../../feat/ddl/ddl.md) |
| 分区表的 DD 往返与裁剪 | [`partitioning.md`](../../feat/partitioning.md) |
| 生成列表达式的 DD 文本往返 | [`generated_columns.md`](../../feat/generated_columns.md) |
| MDL（元数据**锁**，与元数据本身不同） | [`../../lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md) |
| `row_prebuilt_t` / `mysql_row_templ_t`（运行期行转换模板） | [`../handler.md`](../handler.md) |
| `DB_ROW_ID` 与 DICT_HDR 的 row_id 计数器 | [`../../innodb/physical/record.md`](../../innodb/physical/record.md) |

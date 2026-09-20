# dd/ —— 数据字典

> 一句话定位：**MySQL 里"元数据存在哪、怎么被加载、两份字典如何同步"**的唯一出处。
> 8.0 有**两套并存的字典**（server 层权威 DD + InnoDB 内部运行时字典），辨析见 [`dd.md`](dd.md) 文首的「两套并存的字典」表。

## 文件索引

| 文件 | 内容 |
|---|---|
| [dd.md](dd.md) | **server 层 8.0 数据字典**：★ 全景图 + DD 表体系（30 张逐张用途）与 `mysql.ibd`、★★ `dd::tables::Tables` vs `dd::Table` + 完整继承多态（双轨/虚继承/多态实例/Collection 所有权）、★★ **Composite 描述树与 attribute/children/store/restore 机制**（加载=依赖序、drop=逆序、叶子用空实现、根 vs 叶子）、★ `options` vs `se_private_data` 与全键名表、★ `Dictionary_client` + RAII + COW + MVCC 私有写集、SDI、★★ **初始化执行链**（create_target_table/execute_direct/dd_properties 先建/Upgrade_status 阶段文件）、★★ **升级三路径**（DD_INITIALIZE/_SERVER/RESTART + 5.7 迁移 + 崩溃可回退）、原子 DDL 中的地位、★ DD 里的 C++ 运用 |
| [innodb_dict.md](innodb_dict.md) | **InnoDB 内部字典**：DICT_HDR 页（系统表空间页 7）、`SYS_TABLES`/`SYS_COLUMNS`/`SYS_INDEXES`/`SYS_FIELDS` 四张内部表、`dict_table_t`（★ instant 三组计数器 + `dd_table_has_row_versions` 与 ALTER 继承）/ `dict_index_t`（★ instant 三件套、★ `is_usable()`）/ `dict_col_t` 对象模型、dict cache（哈希 + ★ LRU/non_LRU 与"谁不可淘汰" + `dict_sys_mutex` + ★ **intrinsic 会话缓存**）、从 DD 加载链路、★ `mysql.ibd` 硬编码创建与页布局、★★ **动态元数据全链**（三态机 + `MLOG_TABLE_DYNAMIC_META` 压缩 redo + DD Buffer Table 快照与 checkpoint 分工 + recovery 四步）、★ **index corrupt 闭环**、★ `mysql.innodb_ddl_log` 与两种原子性分工、分区表一对多 |
| [statistics.md](statistics.md) | **InnoDB 统计信息**：★ 四套"统计"辨析（`innodb_*_stats` / DD 的 `table_stats` / 内存统计 / 直方图）、★ 表级 `stats_auto_recalc`+`stats_sample_pages` 存在 `dd::Table::options()`、`dict_stats_update()` 四种模式逐行解析（★ 采前唤醒 purge / ★ 用 dummy clone 承接读取）、采样算法与 `N_SAMPLE_PAGES`、★ **专属线程 `dict_stats_thread`**（★ 周期性唤醒防事件丢失 / `BG_STAT_IN_PROGRESS` / ★ recalc_pool latch 层级的三场景推导）、`ANALYZE TABLE` 链路、参数表 |

## 与周边文档的边界

| 主题 | 不在本目录，见 |
|---|---|
| 表对象的生命周期（`TABLE_SHARE` / `TABLE` / 开表 / 锁表 / 表 DDL 操作） | [`../table.md`](../table.md) |
| DDL 的三种算法与执行（COPY / INPLACE / INSTANT、row log、DDL log） | [`../../innodb/ddl.md`](../../innodb/ddl.md) |
| 分区表的 DD 往返与裁剪 | [`partitioning.md`](../../feat/partitioning.md) |
| 生成列表达式的 DD 文本往返 | [`generated_columns.md`](../../feat/generated_columns.md) |
| MDL（元数据**锁**，与元数据本身不同） | [`../../lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md) |
| `row_prebuilt_t` / `mysql_row_templ_t`（运行期行转换模板） | [`../handler.md`](../handler.md) |
| `DB_ROW_ID` 与 DICT_HDR 的 row_id 计数器 | [`../../innodb/physical/record.md`](../../innodb/physical/record.md) |

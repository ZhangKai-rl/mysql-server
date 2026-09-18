# 云存储深度解析（EBS / 云盘 / 分布式块存储）

> 综合 AWS EBS、腾讯云 CBS、阿里云 ESSD 官方文档与 MySQL 8.0.39 源码整理。讲清三件事：**云盘是什么（以及它不是什么）**、**为什么云一定要把存储做成网络块设备**、**MySQL 跑在云盘上时内核侧究竟发生了什么**。

> **边界**：本篇讲**云块存储本身**与其上的 MySQL I/O 行为。云数据库整体架构（Proxy/HA/切换）见 [`cloud_db.md`](cloud_db.md)；网络（VPC/VIP/网关）见 [`cloud_networking.md`](cloud_networking.md)；`innodb_flush_method` 与 Buffered/O_DIRECT 的完整语义见 [`../innodb/io.md`](../innodb/io.md)；脏页刷盘的调度侧见 [`../innodb/buffer_pool.md`](../innodb/buffer_pool.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [主链路：一次 16KB 页写从 mysqld 到物理盘](#主链路一次-16kb-页写从-mysqld-到物理盘)
- [attach/detach 与 mount/umount：控制面 vs 数据面](#attachdetach-与-mountumount控制面-vs-数据面)
- [卷类型与性能模型](#卷类型与性能模型)
- [快照、克隆与 lazy load](#快照克隆与-lazy-load)
- [云盘上跑 MySQL（衔接）](#云盘上跑-mysql衔接)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

**云盘 = 网络块存储（Network Block Storage）**：一块通过数据中心网络连接到计算实例、在 OS 里表现为普通块设备（`/dev/nvme1n1`、`/dev/vdb`）的虚拟硬盘。数据在存储集群内部以**多副本**方式跨物理盘/机架存放，对上层只暴露**线性块地址（LBA）**。

AWS 叫 **EBS（Elastic Block Store）**，腾讯云叫 **CBS（Cloud Block Storage）**，阿里云叫**云盘（ESSD/SSD/高效云盘）**——**同一个东西的三种品牌名**，不是三种技术。

### 用途

- 让存储的生命周期**独立于**计算实例（实例可以销毁重建，数据不动）
- 让存储可以在实例之间**迁移**（这是云数据库 HA 的物理基础）
- 用多副本替代 RAID，把"硬盘损坏"从运维事件降级为系统内部自愈

### 三个最容易搞混的问题

> 这三个问题看似基础，却是云上数据库几乎所有"灵异故障"的认知源头。

#### 一、EBS 就是云盘吗？

**是，也不完全是**——区别在于专名与通名：

| 说法 | 性质 | 范围 |
|------|------|------|
| **网络块存储 / 分布式块存储** | 技术通名 | 通用概念 |
| **EBS** | AWS 的产品专名 | 只在 AWS 语境下成立 |
| **云盘** | 国内云厂商的产品通名（阿里叫"云盘"、腾讯叫"云硬盘/CBS"） | 中文语境下的通名 |

所以在中文技术讨论里说"这块库用的云盘"，等价于 AWS 说"this instance uses EBS"。**但严格说 EBS ≠ 云盘**：EBS 是 AWS 那一个具体实现（有自己的卷类型谱系 gp3/io2、自己的 multi-attach 语义），而"云盘"是国内各家实现的统称（阿里云 ESSD 有 PL0~PL3 分级、腾讯云 CBS 有五级盘型）。**说"云盘"时指类别，说"EBS"时指 AWS 那一个产品。**

> ⚠️ 同类坑：**Aurora / PolarDB / TDSQL-C 的存储层不是云盘**（虽然常被这么叫）。它们是自研的**日志结构化分布式存储**，只接收 redo log，不是通用块设备。详见 [三条技术路线](#三条技术路线云盘版--本地盘版--存算分离)。

#### 二、detach/attach 就是 unmount/mount 吗？

**不是。它们差了两个层次，而且顺序错了会丢数据。**

```
┌──────────────────────────────────────────────────────────┐
│ 云控制面（云厂商 API）                                     │
│   AttachVolume / DetachVolume                            │
│   → 决定"这块卷插在哪台实例上"                              │
│   → 结果：实例的 PCI/NVMe 总线上多出（少掉）一个设备          │
└───────────────────────┬──────────────────────────────────┘
                        │ 这一步成功后 OS 才看得见 /dev/nvme1n1
┌───────────────────────▼──────────────────────────────────┐
│ Guest OS 内核（文件系统层）                                 │
│   mount /dev/nvme1n1 /data                               │
│   → 把块设备挂进目录树，建立 superblock/dentry 缓存          │
└──────────────────────────────────────────────────────────┘
```

| | attach / detach | mount / umount |
|---|---|---|
| **层次** | **云控制面**（虚拟化/存储平面） | **Guest OS 内**（文件系统层） |
| **主语** | 卷 ↔ 实例 | 块设备 ↔ 目录 |
| **操作者** | 云 API / 控制台 | OS 系统调用 |
| **可见性变化** | `lsblk` 里出现/消失设备 | `df` 里出现/消失挂载点 |
| **能否跨实例** | ✅ 能把卷搬到另一台机器 | ❌ 只在本机内有效 |
| **数据落盘语义** | 无（不涉及 page cache） | 涉及 page cache 与脏数据回写 |

**唯一正确的顺序**：

```
上线：  AttachVolume  →  lsblk 确认设备  →  mount  →  起 mysqld
下线：  停 mysqld  →  umount（刷脏数据）  →  DetachVolume
```

**顺序错的后果**：

- **先 detach 后 umount**（或干脆不 umount 就强制 detach）：文件系统 superblock 没卸载、page cache 里的脏数据没回写，卷被抽走。轻则下次挂载触发 `fsck`/journal replay，重则 **ext4 journal 与数据不一致，MySQL 数据文件物理损坏**。
- **先 mount 后 attach**：设备还不存在，mount 直接失败（`No such file or directory`）。

> **对数据库的直接意义**：`detach/attach` 是云数据库 HA 的**物理基础**。主库所在宿主机宕机时，管控不需要"把数据复制一份到新机器"，只需要把同一块云盘 **detach → attach** 到健康的备机，再 mount、拉起 mysqld、跑崩溃恢复。**数据一字节都没搬**——这是本地盘版 RDS 做不到的（本地盘跟着机器走，机器死则数据不可访问，只能靠 binlog 在别的机器上重放出一个副本）。

#### 三、云托管 RDS 现在都是云盘吗？"云盘版"是什么？

**主流 RDS 确实基本都是云盘，但"云盘版"是一个有对立面的产品名词，不是默认事实。**

以阿里云 RDS MySQL 为例，官方存储类型列了四种：**高性能本地盘、高性能云盘、ESSD 云盘、SSD 云盘（部分实例已停售）**——也就是说"云盘版"是相对于"本地盘版"而言的一个**可选形态**。

| 形态 | 存储在哪 | HA 怎么做 | 扩缩容 | 现状 |
|------|---------|----------|--------|------|
| **本地盘版** | 独占物理机的本地 NVMe/SSD | 只能靠 **binlog 逻辑复制** 在另一台机器上重建副本 | 受单机盘位限制，扩容=迁机 | 逐步淘汰 |
| **云盘版** | 分布式块存储（多副本） | **detach/attach 换机** + binlog 备机 | 在线扩盘，几乎无上限 | **当前主流** |
| **存算分离版**（Aurora/PolarDB/TDSQL-C） | 自研日志结构化存储 | 共享存储 + 多计算节点 | 存储自动增长 | 云原生新形态 |

**为什么云盘版赢了本地盘版**——关键不是性能（本地盘更快），而是**控制权**：

1. **HA 从"数据复制"降级为"设备重挂"**：本地盘版主机宕机，副本得靠 binlog 追平，RTO 分钟级且可能丢数据（异步复制）；云盘版只是把盘挂到新机器，RTO 秒级、RPO≈0。
2. **备份从"逻辑导出"降级为"盘级快照"**：云盘快照是块级增量的，秒级完成、对实例几乎无影响。
3. **规格与存储解耦**：CPU/内存换规格时数据不用动。

代价是**性能让渡**：本地盘 4K 随机读 ~100µs 级，云盘要 0.1~3ms（见[卷类型](#卷类型与性能模型)的官方时延表），且延迟受网络与其他租户影响。

> 一句话：**"云盘版"是用 1~2 个数量级的 I/O 延迟，换来了"存储独立于计算"这个云计算最核心的弹性前提。**

### 形态演进

| 阶段 | 形态 | 特征 |
|------|------|------|
| 2010 前 | 本地盘 + 手工 RAID | 存储=机器的一部分 |
| 2010-2015 | **网络块存储普及**（EBS/CBS/云盘） | 三副本、AZ 内冗余、快照；数据库还是"单机 + 主备复制" |
| 2015-2018 | **存算分离**（Aurora 2014 发布 / PolarDB 2017 / TDSQL-C） | 存储层开始理解 redo log，不再是哑块设备 |
| 2018 至今 | **性能分层 + 弹性配置**（gp3 的 IOPS/吞吐/容量三者解耦、ESSD PL 分级、增强型/极速型 SSD） | 从"盘型固定"到"性能按需购买" |

> **演进主线**：存储从"一块更可靠的硬盘"→"可编排的资源"→"理解数据库语义的服务"。每一阶段的动机写在[他库对比与演进动机](#他库对比与演进动机)。

---

## 理论基础

> 本章回答"云为什么要把存储做成网络块设备"，以及这个选择**付出了什么代价**。

### 设计思想与权衡

#### 一、为什么必须是"网络"的？——被否决的三个方案

云的弹性有个死要求：**实例可以在任意时刻消失，数据不能跟着消失**。为此存储必须与计算解耦。当时可选的路有三条，云盘选了看起来最笨的那条：

| 方案 | 做法 | 为什么没这么做（或只做了一部分） |
|------|------|--------------------------------|
| **① 本地盘 + 上层复制** | 每副本独立本地盘，靠数据库 binlog/raft 同步 | **数据库负担太重**：云厂商要卖的是"任何数据库都能跑"，不能要求 MySQL/Postgres/Oracle 各自实现分布式复制；且 RTO 长（要重建数据） |
| **② 网络文件存储（NFS）** | 共享 NAS，多机挂同一目录 | **锁语义与延迟都不对**：数据库要的是"独占 + 强一致 + 低延迟块写"，NFS 的缓存一致性模型（close-to-open）会让两个 mysqld 同时写同一个 ibd 而互不知情；且文件级抽象让数据库无法控制刷盘语义 |
| **③ 对象存储（S3）** | HTTP PUT/GET 存数据 | **不能原地改写**：对象是整体的、不可变的，数据库的 16KB 页级原地更新完全无法映射；延迟是毫秒到几十毫秒级 |
| **✅ ④ 网络块存储** | 暴露 LBA，对 OS 伪装成本地盘 | **兼容性最好**：上层一切不用改（文件系统、数据库、内核 I/O 栈全部原样工作），云厂商只需在底层保证"这个 LBA 读出来是上次写进去的" |

**这就是云盘全部设计选择的根源**：它卖的是**最小契约的兼容性**，代价是**放弃了对上层语义的理解**——存储层不知道自己存的是 InnoDB 的页还是猫的图片。Aurora 后来的革命，正是把这条契约撕开重新谈（见[理论溯源](#理论溯源)）。

#### 二、多副本为什么是三份？——可用性与成本的拐点

云盘普遍采用 **AZ 内三副本**（腾讯云文档明写"采用三副本的分布式机制"）。为什么是 3：

| 副本数 | 可容忍故障 | 写放大 | 成本 |
|--------|-----------|--------|------|
| 2 | 1 盘坏就只剩 1 份，**无法判断谁对**（脑裂风险） | 2× | 低 |
| **3** | **可容忍 1 份损坏且能自愈**（多数派裁决） | 3× | **中（拐点）** |
| 5 | 容忍 2 份 | 5× | 高 |

配合 **Quorum 写**（W=2 of 3，多数派返回即 ACK），既保证持久又不要求三份全写成功。AWS 给出的对外承诺是 **gp3/gp2/io1 年故障率 0.1%~0.2%（持久性 99.8%~99.9%），io2 Block Express 达 99.999%**——后者贵，买的就是更严的副本策略与更快的自愈。

> ⚠️ **必须说清的边界**：三副本是 **AZ 内** 的。**AZ 整体故障（机房断电、水淹、网络分区）时云盘一样不可用**。跨 AZ 容灾必须靠：快照（存对象存储，跨 AZ 甚至跨地域）+ 数据库层的跨 AZ 复制。把"云盘三副本"等同于"高可用"是云上最危险的误解之一。

#### 三、性能为什么不能给满？——超售、burst 与 credit 桶

这是**云盘与数据库冲突最尖锐**的地方。

物理盘的能力是固定的，但租户的 I/O 需求是突发的。如果按峰值给每个卷预留性能，成本会失控。于是云盘普遍采用**超售 + 突发（burst）**：

```
gp2 模型（老）：
  基线 IOPS = 3 × 容量(GiB)，下限 100，上限 16000
  空闲时攒 I/O credit，忙时透支 credit 突发到 3000 IOPS
  credit 耗尽 → 立刻掉回基线
```

**这个设计的失效场景（必须连着讲）**：

1. **小容量卷跑高 IOPS 业务**：100 GiB 的 gp2 卷基线只有 300 IOPS。MySQL 跑起来靠 burst 撑到 3000，看起来一切正常；持续写入几小时后 credit 耗尽，**IOPS 断崖式跌回 300**。
2. **表现**：QPS 突降、慢查询暴涨、`Innodb_buffer_pool_wait_free` 上升——而 CPU、内存、连接数全正常。这是云上 MySQL 最经典的"查不出原因的抖动"。
3. **为什么难发现**：credit 余额是**云监控指标**（`BurstBalance`），MySQL 和 OS 里都看不到。

**gp3 的修正（2020）**：把 IOPS、吞吐、容量**三者解耦**——基线直接给 3000 IOPS / 125 MiB/s，超出部分单独买。这本质上是把"超售的突发能力"改成"明码标价的预留能力"，从赌 credit 变成买确定性。

> **对数据库的直接推论**：云上 MySQL **不要**用绑容量的老盘型（gp2）。要么用性能解耦的新盘型（gp3/ESSD PL1+），要么明确知道自己买了多少 IOPS。

#### 四、块语义的最小契约，以及它逼数据库重造轮子

云盘对上层的承诺只有一句：**"写进去的字节，读出来还是那些字节"**。它**不承诺**：

- ❌ 多块写是原子的（写 16KB 页时断电，可能一半新一半旧 → **torn page / 半页写**）
- ❌ 写返回时数据已持久化（除非设备支持并正确实现 FUA/flush）
- ❌ 延迟稳定（受网络、邻居租户、副本自愈影响）

于是数据库只能自己补：

| 缺口 | 数据库的补救 | 源码落点 |
|------|------------|---------|
| 半页写 | **doublewrite buffer**：页先整页写到 dblwr 文件并 fsync，再写数据文件；恢复时用 dblwr 副本修复 | [`buf0dblwr.cc`](../innodb/io.md#doublewrite) |
| 刷盘持久化 | **fsync / fdatasync** 或 `O_DIRECT` | [`os_file_fsync_posix`](../innodb/io.md#os_file-与-io-方式o_direct--o_sync--fsync) |
| 延迟不稳定 | WAL（redo 顺序写）+ 后台异步刷脏，把随机写延迟从前端事务路径上摘掉 | [`buf_flush_write_block_low`](../innodb/io.md#云盘上的-mysql-io) |

> **这就是"重复的原子性"**：存储层如果提供 **16KB 原子写（atomic write）**，doublewrite 就可以完全关掉，写放大直接减半。部分厂商做了（Fusion-io 的 atomic write、部分云盘的 16K 原子写），MySQL 也留了口子（`innodb_doublewrite=OFF` + 设备保证），但**社区版默认不敢依赖**，因为块存储契约里没有这一条。这是"最小契约"设计的长期成本。

#### 五、控制权让渡：你看不到物理盘

上云之后，DBA 失去了几项能力，这些损失是**结构性**的：

| 失去的 | 后果 | 应对 |
|--------|------|------|
| 不知道数据在哪块物理盘、什么介质 | `iostat` 的 `%util`/`await` 含义被网络层模糊 | 看云监控的卷级指标，而不是只信 `iostat` |
| 看不到邻居租户 | **noisy neighbor**：延迟毛刺与你无关 | 用 p99/p999 而不是平均值看延迟；买更高盘型换隔离 |
| 无法控制副本放置 | 三副本可能落在同机架（概率低但存在） | 不赌，跨 AZ 靠复制 |
| 故障模式从"报错"变成"卡住" | 见下 | 见下 |

**最后一条最危险**：本地盘坏是**返回 EIO**（MySQL 会 `ib::fatal` 崩溃，由 HA 接管，行为明确）；云盘底层故障时更常见的表现是**长时间 hang 而不是报错**——因为网络超时的默认容忍远大于磁盘超时。而 MySQL 的 `pwrite`/`fsync`/`io_getevents` **全部没有超时机制**，于是整实例挂起，HA 的心跳探测可能也跟着卡住。

> **这是"MySQL 对云盘无感知"最直接的运维代价**，见 [云盘上跑 MySQL（衔接）](#云盘上跑-mysql衔接)。

### 理论溯源

| 理论/论文 | 核心主张 | 在云存储里的体现 |
|-----------|---------|-----------------|
| **Amazon Aurora (SIGMOD 2017)** *Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases* | "**The log is the database**"：网络是瓶颈，所以只传 redo log，不传数据页；存储层自己异步回放日志 | 直接宣告了"通用块存储不适合云数据库"。Aurora 的存储层**不是 EBS**，是理解 redo 语义的六副本 quorum 服务。这是云存储演进的分水岭 |
| **PolarFS (VLDB 2018)** *PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database* | 用 **RDMA + 用户态 I/O 栈** 把分布式文件系统的延迟压到接近本地 SSD | PolarDB 的存储底座。证明"网络存储 ≠ 必然慢"——**协议栈开销才是主因**，不是网络本身 |
| **Gray & Reuter《Transaction Processing》/ ARIES** | WAL 与 "**steal/no-force**" 缓冲管理；**partial write** 需要额外保护 | 云盘把 partial write 问题原封不动留给了数据库，doublewrite 是 ARIES 体系在云时代的遗留补丁 |
| **Quorum 系统（Gifford 1979）/ Raft** | W + R > N 保证读写一致 | 云盘三副本写多数派返回即 ACK |
| **Chain Replication（van Renesse & Schneider, OSDI 2004）** | 写沿链传递、尾部返回，读走尾节点 | 许多分布式块存储（含 Azure 早期的 Stream Layer）的副本模型 |
| **PACELC（Abadi 2012）** | 分区时 A/C 权衡，否则 L（延迟）/C 权衡 | 云盘选择了 **PC/EL**：一致优先，接受网络带来的延迟 |

> **源码哪一体现**：MySQL 侧没有 Aurora 那套"日志即数据库"，它仍然把存储当哑块设备——**整个 `storage/innobase` 目录里检索不到任何与云相关的代码**（grep 结论见后）。Aurora 的思想落在**云厂商的存储层**里，不在 MySQL 源码里。

### 算法与数据结构

| 机制 | 算法 | 为什么选它 |
|------|------|-----------|
| **地址映射** | 虚拟 LBA → (chunk_id, offset) 的**大页表/映射表**，chunk 通常 几 MB~几十 MB | 支持**秒级快照**与**在线扩盘**：扩盘只是加映射项，不搬数据；快照只是冻结映射表 |
| **快照** | **增量 + COW/ROW**：打快照时冻结映射表，之后被覆盖的 chunk 才复制到新 chunk | 第一次快照可以秒级完成，且不占额外空间；多次快照形成增量链 |
| **多副本一致性** | Quorum（W=2,R=1 of 3）或**链式复制** | Quorum 写延迟取决于最快的两副本；链式复制吞吐更好但尾延迟更高 |
| **故障自愈** | **副本重平衡**：检测到副本不可用时，在健康节点上按映射表重建数据 | 把"换盘"变成后台任务，对用户透明 |
| **条带化** | 一个卷的 LBA 空间打散到多个存储节点/chunk | 单卷 IOPS 可以超过单盘能力（这也是"单盘 100 万 IOPS"的来源：它不是一块盘，是一堆盘） |

### 他库对比与演进动机

#### 三家云盘规格对照

| 维度 | AWS **gp3** | AWS **io2 Block Express** | 腾讯云**增强型 SSD** | 腾讯云**极速型 SSD** | 阿里云 **ESSD** |
|------|------------|--------------------------|---------------------|---------------------|----------------|
| 容量 | 1 GiB–64 TiB | 4 GiB–64 TiB | ≤32000 GiB | ≤32000 GiB | 按 PL 分级 |
| 最大 IOPS | 80,000（25.6 KiB I/O） | **256,000**（16 KiB） | 100,000（叠加额外性能） | **1,000,000**（叠加额外性能） | 单盘最高 100 万 |
| 最大吞吐 | 2,000 MiB/s | 4,000 MiB/s | 1,000 MB/s | 4,000 MB/s | — |
| 时延（官方口径） | — | 16 KiB I/O 平均 **<500 µs** | 4K 单路随机 **0.2–1 ms** | 4K 单路随机 **0.1–0.5 ms** | 25 GE + RDMA，单路低时延 |
| 持久性 | 99.8%–99.9% | **99.999%** | 三副本 | 三副本 | 多副本 |
| 性能与容量 | **解耦**（IOPS/吞吐单独买） | 解耦 | 基准 + 额外性能 | 基准 + 额外性能 | PL0–PL3 分级 |

腾讯云其余盘型（官方公式）：

| 盘型 | 随机 IOPS 公式 | 上限 | 吞吐上限 | 4K 单路时延 |
|------|---------------|------|---------|------------|
| 高性能云硬盘 | `min{1800 + 容量×8, 6000}` | 6,000 | 150 MB/s | 0.8–5 ms |
| 通用型 SSD | `min{1800 + 容量×15, 10000}` | 10,000 | 190 MB/s | 0.5–3 ms |
| SSD 云硬盘 | `min{1800 + 容量×30, 26000}` | 26,000 | 260 MB/s | 0.5–3 ms |
| 增强型 SSD | 基准 `min{1800 + 容量×50, 50000}` + 额外 | 100,000 | 1,000 MB/s | 0.2–1 ms |
| 极速型 SSD | 基准 `min{4000 + 容量×100, 50000}` + 额外 | 1,000,000 | 4,000 MB/s | 0.1–0.5 ms |

> **读法**：IOPS 与容量挂钩的盘型（前三种）本质上还是"买容量送性能"的老模型，与 gp2 同构；增强型/极速型、gp3、ESSD 才是"性能独立配置"的新模型。**跑数据库一律选后者**，否则就会遇到 gp2 那种 credit 耗尽掉速（虽然腾讯云这几档没有显式的 credit 桶，但"性能随容量"意味着小容量卷照样喂不饱数据库）。

#### 三条技术路线：云盘版 / 本地盘版 / 存算分离

| 维度 | 本地盘版 | **云盘版（本篇）** | 存算分离（Aurora/PolarDB/TDSQL-C） |
|------|---------|------------------|-----------------------------------|
| 存储 | 独占物理机本地 NVMe | 分布式块存储（哑块设备） | 自研日志结构化存储 |
| 传给存储的是 | 数据页 | **数据页（16KB）** | **redo log** |
| 网络放大 | 无 | 1×（页大小即传输量） | 小得多（只传变更的日志） |
| 半页写防护 | 设备通常支持原子写 | **必须 doublewrite** | 存储层自己做，数据库不需要 |
| HA | binlog 重建，分钟级 | **detach/attach 换机，秒级** | 共享存储 + 秒级切计算节点 |
| 只读扩展 | 独立副本 | 独立副本 | **共享同一份存储，加节点不加存储** |
| 延迟 | 最低（~100 µs） | 中（0.1–3 ms） | 网络一跳（RDMA 优化后接近本地） |
| 代表 | 阿里云"高性能本地盘"、部分独占机型 | **AWS RDS、阿里云 RDS 云盘版、腾讯云 CDB** | Aurora / PolarDB / TDSQL-C |

**演进动机**（为什么会有第三条路）：

云盘把"网络是瓶颈"的矛盾暴露到了极致——**数据库写放大 × 页大小 × 三副本** 会让网络带宽成为瓶颈（Aurora 论文开篇就说：在云上，网络而不是 CPU/存储成为限制）。既然如此，为什么不干脆只传最小必要的东西？于是有了"日志即数据库"。**但这条路要求存储层理解数据库语义，代价是存储不再通用**（只能服务特定引擎）。所以三条路线会长期并存：通用云盘服务一切，存算分离服务头部数据库。

---

## 主链路：一次 16KB 页写从 mysqld 到物理盘

```
① mysqld（用户态）
   buf_flush_write_block_low        先 log_write_up_to（WAL 屏障）
      └─ dblwr::write               整页写进 doublewrite 文件 + fsync
           └─ write_to_datafile
                └─ fil_io → Fil_shard::do_io    page_no → 文件 offset
② OS 内核（VFS → 块层）
   os_aio_func
     ├─ AIO_mode::SYNC → pwrite()
     └─ async → io_submit()          （libaio，槽位数受 innodb_write_io_threads 限制）
③ 虚拟化层（CVM）
   virtio-blk / virtio-scsi / NVMe 前端驱动 → 后端（宿主机或硬件卸载）
④ 宿主机 → 网卡 → 数据中心网络
   （SR-IOV / 硬件卸载可绕过宿主机内核，Nitro 卡即为此设计）
⑤ 云盘存储集群
   LBA → chunk 映射表 → 定位 3 个副本所在存储节点
   → 写多数派（W=2/3）→ 返回 ACK
⑥ 物理 SSD / NVMe（可能跨机架）
```

**每层引入的代价**：

| 层 | 引入什么 | 量级 |
|----|---------|------|
| ① → ② | 系统调用 + VFS（O_DIRECT 可跳过 page cache） | ~µs |
| ② → ④ | **虚拟化 + 网络**（云盘的全部额外开销都在这） | **0.1–3 ms（主要来源）** |
| ④ → ⑥ | 副本复制 + 寻址 | 已包含在盘型时延里 |

> 关键认知：**云盘比本地盘慢，慢在网络与协议栈，不是慢在 SSD**。这也是 PolarFS 用 RDMA + 用户态栈把延迟压回去的理论依据。

---

## attach/detach 与 mount/umount：控制面 vs 数据面

这一节把概述里的问题二展开到工程细节，因为**它是云数据库 HA 的操作核心**。

### 完整状态机

```
                    ┌──────────────┐
     创建卷  ──────▶│  available   │◀──── DetachVolume ────┐
                    │（未挂任何实例）│                        │
                    └──────┬───────┘                        │
                           │ AttachVolume(instance)          │
                           ▼                                │
                    ┌──────────────┐                        │
                    │  attaching   │                        │
                    └──────┬───────┘                        │
                           ▼                                │
                    ┌──────────────┐        umount          │
                    │   attached   │────────────────────────┘
                    │（OS 可见设备）│
                    └──────┬───────┘
                           │ mount /dev/nvme1n1 /data
                           ▼
                    ┌──────────────┐
                    │   mounted    │◀── MySQL 数据目录
                    └──────────────┘
```

**注意两个状态是正交的**：`attached` 只是"线插上了"，`mounted` 才是"卷里的数据可用"。一块卷可以 `attached` 但没 `mounted`（刚挂上还没挂文件系统），也可以因为挂载了但文件系统错误而不可用。

### 云数据库 HA 的标准动作序列

```
主库宿主机宕机
  ↓
① 管控判定主库不可达（拨测/心跳，通常 5–15s）
  ↓
② 【安全屏障】确认旧主已让出写权限
    - 云盘版：旧主机已死，不可能再写（这是云盘相对本地盘的关键优势：
      detach 后旧实例物理上无法再访问这块盘）
    - 存算分离版：需要显式的 DEMOTE/lease 机制（见 cloud_db.md）
  ↓
③ DetachVolume（旧实例，可能失败需强制）
  ↓
④ AttachVolume（新实例，同一块卷，同一份数据）
  ↓
⑤ 新实例上：lsblk 确认设备 → fsck（如需）→ mount → chown mysql
  ↓
⑥ 启动 mysqld → InnoDB 崩溃恢复（redo 前滚 + undo 回滚）
  ↓
⑦ 网络层把 VIP 指向新实例（见 cloud_networking.md）
```

**第 ⑥ 步是云盘版 HA 的隐藏成本**：虽然数据没搬，但 **mysqld 起来要跑崩溃恢复**，恢复时间取决于 checkpoint 之后的 redo 量。这也是为什么云数据库会调优 `innodb_max_dirty_pages_pct` 与 checkpoint 频率——**控制 RTO 就是控制崩溃恢复要回放多少 redo**。

### 顺序错误的四种典型事故

| 错误操作 | 后果 | 严重度 |
|---------|------|--------|
| 不 umount 直接 detach | 文件系统未卸载，脏数据丢失；journal 未完成 | 🔴 数据损坏 |
| **MySQL 还活着就 detach** | mysqld 持有的 fd 指向已消失的设备，后续 I/O 全部失败；`fsync` 返回 EIO → `ib::fatal` 崩溃 | 🔴 实例崩溃 |
| attach 后立即 mount（不等设备就绪） | `mount: /dev/nvme1n1: No such file or directory` | 🟡 重试即可 |
| **同一卷 attach 到两台实例并同时 mount ext4** | **两块 OS 各自缓存元数据，必然文件系统损坏**。只有集群文件系统（OCFS2/GFS2）或云盘 multi-attach + 集群感知才行 | 🔴 灾难 |

> 最后一条值得强调：AWS 的 **io1/io2 Multi-Attach** 允许一块卷挂到多台实例，但它只解决"能同时看到这块盘"，**不解决并发写的缓存一致性**。要用它跑数据库集群，必须在上面再跑一套集群文件系统或分布式锁；MySQL 社区版没有这个能力。

---

## 卷类型与性能模型

### 选型的四个真实维度

不要按"SSD/HDD"选，按这四个维度选：

1. **IOPS**（16KB 页随机读写能力）→ 决定 OLTP 吞吐
2. **吞吐**（MB/s，大块顺序 I/O）→ 决定备份、批量导入、大表扫描、redo 大写入
3. **时延**（尤其是 **p99**，不是平均）→ 决定单条 SQL 的尾部延迟
4. **性能是否与容量绑定** → 决定"小容量高 IOPS"能不能成立

### AWS EBS 谱系（官方规格）

| 类型 | 介质 | 容量 | 最大 IOPS | 最大吞吐 | Multi-Attach | 定位 |
|------|------|------|----------|---------|--------------|------|
| **gp3** | SSD | 1 GiB–64 TiB | 80,000（25.6 KiB I/O） | 2,000 MiB/s | ❌ | **默认首选**，IOPS/吞吐/容量解耦 |
| **gp2** | SSD | 1 GiB–16 TiB | 16,000（16 KiB） | 250 MiB/s | ❌ | 老默认，IOPS 绑容量（3/GiB），靠 burst |
| **io2 Block Express** | SSD | 4 GiB–64 TiB | **256,000**（16 KiB） | 4,000 MiB/s | ✅ | 大型 OLTP，16 KiB 平均 <500 µs，持久性 **99.999%** |
| **io1** | SSD | 4 GiB–16 TiB | 64,000（16 KiB） | 1,000 MiB/s | ✅ | io2 前代 |
| **st1** | HDD | 125 GiB–16 TiB | 500（1 MiB I/O） | 500 MiB/s | ❌ | 吞吐型，不能做数据库数据盘 |
| **sc1** | HDD | 125 GiB–16 TiB | 250（1 MiB I/O） | 250 MiB/s | ❌ | 冷数据 |
| `standard`（磁性） | HDD | 1 GiB–1 TiB | 40–200 | 40–90 MiB/s | ❌ | 上一代，勿用 |

**持久性**：gp2/gp3/io1 为 **99.8%–99.9%（年故障率 0.1%–0.2%）**；io2 Block Express 为 **99.999%（0.001%）**。

> **读官方表格的一个坑**："最大 IOPS 80,000（25.6 KiB I/O）"括号里的 I/O 大小是**达到该 IOPS 所需的前提**——IOPS × I/O size = 吞吐上限。gp3 的 80,000 × 25.6 KiB = 2,000 MiB/s，正好等于它的吞吐上限。**你不可能同时拿到最大 IOPS 和最大吞吐**，它们是同一块带宽的两面。数据库（16KB 小 I/O）关心前者，备份/导入关心后者。

### gp2 的 burst 机制（为什么要彻底弃用它）

```
                    3000 IOPS ─────────┐
                                       │  突发区（透支 credit）
                                       │╲
                                       │ ╲____╲____╲___  credit 耗尽
                                       │
    基线 = min(3 × 容量GiB, ...)  ─────┼──────────────────────
      100 GiB 卷 → 300 IOPS            │
                                       └──────────────────── 时间
```

- 卷空闲时累积 I/O credit，忙碌时以 3000 IOPS 透支；
- **credit 耗尽后立刻掉回基线**，没有任何渐变；
- 掉速后只有停止 I/O 才能重新累积 credit——**而数据库停不下来**，所以一旦掉下去就很难爬回来（恶性循环）。

**判定方法**：看云监控的 `BurstBalance`（EBS 卷指标）与 `VolumeQueueLength`。前者归零 + 后者飙升 = 正在被 credit 卡住。

### 数据库选型决策

```
这个卷要跑 MySQL 数据目录吗？
├─ 否（备份/归档/日志）
│   └─ 吞吐 HDD（st1/sc1）即可
└─ 是
   ├─ 核心生产 OLTP，能接受溢价
   │   └─ io2 Block Express / 极速型 SSD / ESSD PL2-PL3
   ├─ 通用生产
   │   └─ gp3 / 增强型 SSD / ESSD PL1
   │      ★ IOPS 与容量解耦，按实测 IOPS 需求配，不要"按容量估算"
   └─ 开发/测试
       └─ gp3 最低配 / 通用型 SSD

★ 共同前提：绝不用 IOPS 绑容量的老盘型跑生产数据库
```

---

## 快照、克隆与 lazy load

### 快照是块级增量的，但恢复有冷启动代价

```
打快照：
  ① 冻结 LBA→chunk 映射表（秒级，几乎无性能影响）
  ② 之后的写入触发 COW：被覆盖的旧 chunk 复制到新 chunk 后保存
  ③ 快照 = 冻结的映射表 + 那些被 COW 出来的旧 chunk

从快照恢复：
  ① 新建空卷，映射表指向快照
  ② 读某个 LBA 时，若数据还在对象存储里 → 先拉回来（lazy load）
```

**lazy load 是云上数据库恢复最大的坑**：

| 阶段 | 表现 |
|------|------|
| 从快照建卷后立刻压测 | 首次访问某块要现去对象存储拉，**延迟从 0.5ms 恶化到几十~上百 ms** |
| 新拉的只读节点 | 缓冲池冷 + 存储层也冷，双重冷启动，性能可能只有稳态的 1/10 |
| 恢复的实例直接切线上流量 | **雪崩** |

**应对**：

1. **预热（pre-warm）**：新卷上线先全量读一遍。MySQL 侧可以做一次全表扫描把 BP 灌满，存储侧同时被 lazy load 填满。
   ```bash
   # 存储层预热（读整个块设备）
   fio --filename=/dev/nvme1n1 --rw=read --bs=1M --iodepth=32 --name=warm
   ```
2. **付费免除**：AWS 的 **Fast Snapshot Restore (FSR)** 可预置完全初始化的快照。
3. **永不"恢复即上线"**：先恢复 → 预热 → 追 binlog → 灰度切流。

### 快照 ≠ 数据库一致性备份

**这是云上数据丢失的头号来源。**

云盘快照是**崩溃一致（crash-consistent）**的：它相当于给机器断电后拍的磁盘镜像。对 InnoDB 来说这**通常够用**（有 redo + doublewrite，能崩溃恢复），但：

| 风险 | 说明 |
|------|------|
| **redo 与数据页不在同一块卷** | 若把 redo log 单独放一块盘，两块卷的快照时间点不同 → 恢复后 redo 与页不匹配 |
| **binlog 与数据不在同一块卷** | 同理，恢复出来的实例 binlog 位点与数据不一致，作为主库会让备库错乱 |
| **非事务引擎** | MyISAM 表没有崩溃恢复，崩溃一致的快照可能直接损坏 |
| **跨卷原子性无法保证** | 多块卷的快照不是原子的（除非用云厂商的**快照一致性组**） |

**正确做法**：要么用一致性组快照，要么在打快照前 `FLUSH TABLES WITH READ LOCK` + 记录 binlog 位点（云数据库管控就是这么做的），要么用数据库层的备份工具（xtrabackup / clone plugin）。

---
## 云盘上跑 MySQL（衔接）

> 前面讲的都是云侧。本节只给**结论级**的衔接：**MySQL 对云盘完全无感知**，以及由此必须手工完成的配置。
>
> **MySQL 侧的完整 I/O 栈**——WAL 屏障、doublewrite、`pwrite` vs `io_submit`、fsync 失败处理、`innodb_io_capacity` 自适应算法、邻接刷盘源码——见 **[`../innodb/io.md`](../innodb/io.md)**，尤其其中的 [「★ 云盘上的 MySQL I/O」](../innodb/io.md#云盘上的-mysql-io) 一节。

### ★ MySQL 对云盘完全无感知

**这是一个可以给出证据的事实，不是观点。**

### 证据一：源码里一个相关词都没有

对 `storage/innobase`（InnoDB 全部源码）检索 `EBS`、`cloud`、`network storage`、`SAN`、`iSCSI`、`SSD`、`NVMe`：

```
结果：0 matches
```

对整个 MySQL Server 源码树检索 `cloud`：

```
唯一命中：storage/ndb/src/kernel/blocks/thrman.cpp:2132 的一条注释
  "Even more in a cloud environment we could easily be affected by
   other activities in other cloud apps"
（NDB Cluster 线程管理器，与存储 I/O 无关）
```

### 证据二：没有相关系统变量

唯一的"设备自适应"是 `innodb_dedicated_server`——但它自适应的是**内存**（按物理内存推算 buffer pool），不是存储。

### 证据三：与块设备的契约面只有 6 个系统调用

`open` / `pread` / `pwrite` / `io_submit`+`io_getevents` / `fsync`+`fdatasync` / `fcntl(F_SETFL, O_DIRECT)`（外加 `fallocate` 做 punch hole）。**MySQL 无法区分自己跑在本地 NVMe、EBS、还是网络文件系统上。**

### 五条隐含假设（这才是要点）

MySQL 对底层设备做了五个假设，其中**每一条在云盘上都比本地盘更脆弱**：

| # | 假设 | 源码落点 | 云盘上的风险 |
|---|------|---------|------------|
| 1 | **512B 扇区写是原子的** | `OS_FILE_LOG_BLOCK_SIZE = 512`，redo 按 512 对齐写 | 云盘普遍保证，但**16KB 页写仍不原子** → 必须 doublewrite |
| 2 | **`fsync` 返回 = 数据已持久化** | 所有持久性保证的基础 | 若云盘/虚拟化层的 flush 语义不完整（某些 O_DIRECT 实现有问题），持久性承诺就失效 |
| 3 | **`O_DIRECT` 的 write 返回 = 已持久化** | `dblwr::is_fsync_required()` 在 O_DIRECT 下跳过 fsync | **最高风险**：见下 |
| 4 | **设备能力 = `innodb_io_capacity`** | 默认 **200**，无自动探测 | 与实际云盘差 1~2 个数量级 → 刷脏跟不上 |
| 5 | **I/O 要么成功要么报错** | `os_file_fsync_posix` 遇 EIO 即 fatal | 云盘更可能 **hang 而非报错** → 整实例挂起，无超时保护 |

### 最危险的一条：`innodb_dedicated_server` 的静默改写

`ha_innodb.cc:4993`：

```cpp
  if (srv_dedicated_server && sysvar_source_svc != nullptr &&
      os_is_o_direct_supported()) {
    ...
      if (source == COMPILED) {
        innodb_flush_method = static_cast<ulong>(SRV_UNIX_O_DIRECT_NO_FSYNC);
      }
    ...
  }
```

**当你开了 `innodb_dedicated_server=ON` 且没显式指定 `innodb_flush_method` 时，MySQL 会静默把它改成 `O_DIRECT_NO_FSYNC`**——即"用 O_DIRECT 写，且不 fsync"（仅在文件大小变化时才 flush）。

在本地 NVMe 上这通常没问题。**在云盘上，这个改动把数据安全完全押在"设备的 O_DIRECT 写返回即持久化"这一条上**——而这恰恰是云盘/虚拟化层最容易出偏差的语义（尤其是某些虚拟化后端、缓存层、或开了写缓存的场景）。

> **结论：云上部署 MySQL，永远显式设置 `innodb_flush_method=O_DIRECT`，不要依赖 `innodb_dedicated_server` 的推断。**

### 云上必须手工改的配置

| 参数 | 云上建议 | 理由 |
|------|---------|------|
| `innodb_flush_method` | **显式 `O_DIRECT`** | 不被 `innodb_dedicated_server` 静默改成 `O_DIRECT_NO_FSYNC` |
| `innodb_io_capacity` | 按卷稳态 IOPS × 0.7~0.8 | 默认 **200** 是机械盘数字，会让云盘 95% 能力闲置、脏页锯齿式堆积 |
| `innodb_flush_neighbors` | **0** | 邻接刷盘是"把随机写顺序化"的机械盘设计，云盘上收益为零却带来写放大 |
| `innodb_write_io_threads` | 4 → 8~16 | 云盘队列深，默认 4 个线程发不出足够并发请求 |

> 一句话：**云上 MySQL 的性能问题，80% 不是"盘不够快"，而是"MySQL 不知道盘有多快"。**

**参数详解与源码级解释见 [`../innodb/io.md`](../innodb/io.md) 的「相关的系统变量」与「★ 云盘上的 MySQL I/O」两节。**

---
## Misc

### RDS 三种存储形态速查

| | 本地盘版 | **云盘版** | 存算分离版 |
|---|---|---|---|
| 存储类型 | 独占物理机本地 NVMe/SSD | 分布式块存储（ESSD/云盘/EBS） | 自研日志结构化存储 |
| 传给存储 | 数据页 | **数据页** | **redo log** |
| HA 机制 | binlog 重建副本 | **detach/attach 换机** | 共享存储切计算节点 |
| RTO | 分钟级 | 秒级（+ 崩溃恢复） | 秒级 |
| doublewrite | 设备可能支持原子写，可关 | **必须开** | 存储层负责，不需要 |
| 备份 | 逻辑/物理备份 | **盘级快照（秒级）** | 存储层连续快照 |
| 只读扩展 | 每副本一份数据 | 每副本一份数据 | **共享一份存储** |
| 代表 | 阿里云"高性能本地盘"（历史形态） | **阿里云 RDS 云盘版、AWS RDS、腾讯云 CDB** | Aurora / PolarDB / TDSQL-C |

### 术语对照

| 术语 | 全称 / 含义 |
|------|------------|
| **EBS** | Elastic Block Store（AWS 的网络块存储产品名） |
| **云盘 / 云硬盘** | 国内云厂商的网络块存储通名 |
| **CBS** | Cloud Block Storage（腾讯云云硬盘） |
| **ESSD** | Enterprise SSD（阿里云企业级 SSD 云盘，PL0–PL3 分级） |
| **LBA** | Logical Block Address，线性块地址 |
| **IOPS** | I/O Operations Per Second |
| **burst / credit** | 突发性能与 I/O 信用额度（gp2 类盘型机制） |
| **Multi-Attach** | 一块卷同时挂到多台实例（需集群文件系统配合） |
| **lazy load** | 从快照恢复的卷首次读某块时才从对象存储拉取 |
| **crash-consistent** | 崩溃一致快照（相当于断电后的磁盘镜像） |
| **FSR** | Fast Snapshot Restore（AWS 预初始化快照，免除冷启动） |
| **torn page / partial write** | 半页写：多块写的原子性缺失导致的页损坏 |

### 常见误解纠正

| 误解 | 正确 |
|------|------|
| "EBS 和云盘是两个东西" | 是同一类技术（网络块存储），EBS 是 AWS 专名、云盘是国内通名。**但 Aurora 存储不是云盘** |
| "detach/attach 就是 unmount/mount" | **差两个层次**：前者是云控制面（卷↔实例），后者是 Guest OS（块设备↔目录）。顺序必须是 attach→mount / umount→detach |
| "RDS 都是云盘版" | 主流是，但**"云盘版"是相对于"本地盘版"的可选形态**（阿里云 RDS 就同时列了高性能本地盘/高性能云盘/ESSD 云盘） |
| "三副本 = 高可用" | 三副本是 **AZ 内**的，AZ 故障照样挂。跨 AZ 必须靠快照 + 数据库复制 |
| "云盘快照 = 数据库备份" | 快照是**崩溃一致**的，不是应用一致的。多卷需一致性组，或打快照前 `FLUSH TABLES WITH READ LOCK` |
| "从快照恢复的实例能直接上线" | 有 **lazy load 冷启动**代价，必须先预热 |
| "IOPS 和吞吐能同时拿满" | 不能，二者是同一块带宽的两面：IOPS × I/O size ≈ 吞吐上限 |
| "gp2 便宜够用" | 小容量 gp2 基线只有 3 IOPS/GiB，credit 耗尽后断崖掉速，是数据库抖动的经典来源 |
| "本地盘一定比云盘快所以更好" | 更快，但失去了 detach/attach、秒级快照、在线扩盘——**云的弹性建立在存储与计算解耦上** |
| "MySQL 会自适应云盘" | **完全不会**。`storage/innobase` 里 `cloud`/`EBS`/`NVMe`/`SSD` 全部 0 命中，与设备的唯一契约是 6 个系统调用 |
| "开了 `innodb_dedicated_server` 就优化了" | 它会**静默把 flush_method 改成 `O_DIRECT_NO_FSYNC`**，云上这是数据安全风险 |
| "`iostat %util` 100% 说明盘满了" | 云盘是分布式设备，`%util` 无意义。**看云监控的卷级 IOPS/吞吐/队列长度** |
| "Multi-Attach 能让两台 MySQL 同时读写一块盘" | 只解决了"能看到"，**不解决缓存一致性**，必须配集群文件系统；社区版 MySQL 没有这个能力 |

### 一句话总结

> **云盘的本质是一笔交易：用 0.1~3ms 的网络延迟和可预期的性能上限，换来"存储独立于计算"这个云计算一切弹性的前提。MySQL 对这笔交易毫不知情——它仍然按本地盘的假设在运行，所有适配都要由 DBA 手工完成（主要就是 `innodb_io_capacity`、`innodb_flush_neighbors`、`innodb_flush_method` 这三个）。**

---

## 参考

**论文**

- Verbitski et al. *Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases*. **SIGMOD 2017**. （"The log is the database"；云存储演进的分水岭，直接指出通用块存储在云数据库场景的瓶颈）
- Cao et al. *PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database*. **VLDB 2018**. （RDMA + 用户态 I/O 栈，证明网络存储的瓶颈在协议栈而非网络本身）
- van Renesse & Schneider. *Chain Replication for Supporting High Throughput and Availability*. **OSDI 2004**.
- Gifford. *Weighted Voting for Replicated Data*. **SOSP 1979**. （Quorum 理论）
- Gray & Reuter. *Transaction Processing: Concepts and Techniques*. （WAL、steal/no-force、partial write）
- Abadi. *Consistency Tradeoffs in Modern Distributed Database System Design*. **IEEE Computer 2012**. （PACELC）

**官方文档**

- *Amazon EBS User Guide → Amazon EBS volume types*（gp3/gp2/io1/io2 Block Express/st1/sc1 的容量、IOPS、吞吐、持久性、Multi-Attach 官方规格）
- *Amazon EBS User Guide → General Purpose SSD volumes / Provisioned IOPS SSD volumes*（gp2 的 baseline 3 IOPS/GiB 与 burst 机制）
- *Amazon EBS User Guide → Amazon EBS snapshots*（增量快照与 Fast Snapshot Restore）
- 腾讯云 *云硬盘 → 云硬盘类型*（高性能/通用型 SSD/SSD/增强型 SSD/极速型 SSD 的 IOPS 公式、吞吐、4K 单路时延、三副本机制）
- 阿里云 *云服务器 ECS → ESSD 云盘*（PL0–PL3 分级、25 GE + RDMA、单盘 100 万 IOPS）
- 阿里云 *云数据库 RDS MySQL → 存储类型*（高性能本地盘 / 高性能云盘 / ESSD 云盘 / SSD 云盘 的对比与选购）

**源码**（MySQL 8.0.39，本文所有代码与行号均已核实）

- `storage/innobase/buf/buf0flu.cc` —— `buf_flush_write_block_low`（WAL 屏障）、`set_flush_target_by_lsn`（自适应刷脏）、`buf_flush_try_neighbors`、`buf_flush_fsync`
- `storage/innobase/buf/buf0dblwr.cc` —— `dblwr::write`、`Double_write::sync_page_flush`、`write_dblwr_pages`、`is_fsync_required`
- `storage/innobase/fil/fil0fil.cc` —— `fil_io`、`Fil_shard::do_io`、`fil_flush`、`fil_flush_file_spaces`
- `storage/innobase/os/os0file.cc` —— `os_aio_func`、`SyncFileIO::execute`、`os_file_fsync_posix`、`os_file_create_func`、`AIO::start`、`AIO::linux_dispatch`、`LinuxAIOHandler::collect`、`os_file_handle_error_cond_exit`
- `storage/innobase/log/log0write.cc` / `log0files_io.cc` —— `log_writer_write_buffer`、`log_flush_low`、`Log_file_handle::fsync`
- `storage/innobase/handler/ha_innodb.cc` —— 各 I/O 相关 sysvar 定义与 `innodb_dedicated_server` 的 flush_method 改写
- `storage/innobase/include/srv0srv.h` —— `srv_unix_flush_t` 枚举

**相关文档**

- 云数据库整体架构（Proxy/HA/切换/透明切换分级）见 [`cloud_db.md`](cloud_db.md)
- 云上网络（VPC/VIP/网关/PrivateLink）见 [`cloud_networking.md`](cloud_networking.md)
- InnoDB 的 Buffered I/O 与 O_DIRECT 完整语义见 [`../innodb/io.md`](../innodb/io.md)
- Buffer Pool 与脏页刷盘调度见 [`../innodb/buffer_pool.md`](../innodb/buffer_pool.md)
- redo log 格式与 checkpoint 见 [`../innodb/redo_log.md`](../innodb/redo_log.md)
- 崩溃恢复（云盘版 HA 第 ⑥ 步）见 [`../innodb/recovery.md`](../innodb/recovery.md)

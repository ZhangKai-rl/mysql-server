# 云数据库架构深度解析（以 TDSQL-C/NCDB 为例）

> 综合知识库与源码研究整理，涵盖数据面/支撑环境分层、存算分离、网络体系（VPC/VIP/TGW/Proxy）、HA 切换、透明切换（L0-L6）、运维排查。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [总体架构：数据面 + 支撑环境](#总体架构数据面--支撑环境)
- [网络体系](#网络体系)
- [HA 系统](#ha-系统)
- [内核侧角色切换](#内核侧角色切换)
- [透明切换分级](#透明切换分级)
- [关键流程链路](#关键流程链路)
- [关键组件速查](#关键组件速查)
- [Misc](#misc)

---

## 概述

### 是什么

云数据库 = **MySQL 内核（数据面）** + **支撑环境（控制面）**。用户购买的不是一台装了 MySQL 的虚机，而是一个由内核、网络网关、HA 编排、共享存储、管控系统共同构成的**服务**。

```
用户视角:    一个 VIP + 端口 + 账号密码
真实构成:    VIP → TGW/Proxy → RW/RO 节点 → 共享存储
             + 管控/HA/拨测/备份/监控 在背后维持其可用
```

### 用途

- 免运维：HA、备份、监控、扩缩容由云厂商承担
- 高可用：节点故障自动切换，RPO≈0 / RTO 秒级
- 弹性：存算分离架构下存储按需扩展，只读节点快速挂载

### 演进

| 阶段 | 形态 | 特征 |
|------|------|------|
| 自建 | 物理机 + MySQL | 自己管一切 |
| 云主机托管 | CVM 自装 MySQL | 只是换了机器，HA 仍自理 |
| RDS（主备架构） | 主备半同步 + keepalived/VIP | 管控接管 HA/备份，但存储每副本独立 |
| 云原生（存算分离） | Aurora/PolarDB/TDSQL-C | 一份共享存储多计算节点，redo 物理复制 |

### 服务模型定位（IaaS/PaaS/SaaS）

| 模型 | 提供物 | 用户管什么 |
|------|--------|-----------|
| IaaS | 虚机/网络/存储（"毛坯房"） | OS+运行时+数据+应用 |
| **PaaS（云数据库属此类）** | 数据库服务 | 数据+应用 |
| SaaS | 完整应用 | 只用 |

---

## 理论基础

### 设计模式

| 模式 | 体现 |
|------|------|
| **数据面/控制面分离**（SDN 思想） | 数据面跑 SQL；支撑环境（管控/HA/拨测）独立成面 |
| **代理模式** | Proxy/L7 网关持有 VIP，客户端连的是代理而非物理节点 |
| **观察者模式** | 拨测系统观察节点健康，事件驱动 HA 编排 |
| **责任链** | 透明切换 L0→L6 能力逐级叠加 |
| **共享存储单写者** | RW 独占写；RO 提主需先确保旧 RW 让出写权（DEMOTE 日志/CDP 补全） |

### 相关论文

- **Amazon Aurora (SIGMOD 2017)**：存算分离鼻祖，"The log is the database"，redo 物理复制到存储层
- **PolarFS (VLDB 2018)**：阿里 PolarDB 的分布式文件系统
- **Spanner/Calvin**：与多写集群（MGR）路线对比——云原生选择"共享存储 + 单写者"而非"无共享 + 多数派"

### 算法与数据结构

- **Quorum 仲裁**：DBStore 层多副本（3 副本写多数派成功即 ACK）
- **日志回放**：RO 回放 redo（物理复制 apply_log），与 binlog 逻辑复制对比
- **TCP 四元组**：理解 VIP 切换后旧连接为何必断的根基

### 类似实现对比

| 维度 | AWS Aurora | 阿里 PolarDB | 腾讯 TDSQL-C |
|------|-----------|--------------|--------------|
| 存储 | Aurora Storage（6 副本 3AZ） | PolarFS/PolarStore | DBStore/DBFS（自研） |
| 复制 | redo→存储层 | redo 物理复制 | redo→DBStore + RO 共享 log |
| 代理 | RDS Proxy | Proxy | Magpie/Proxy（会话恢复） |
| 内核 | MySQL 分叉 | MySQL 分叉 | TXSQL/NCDB（含 GSI 等扩展） |

### 历史背景

传统 RDS 主备架构痛点：一份数据写三份（主数据+备 binlog+备数据），存储成本 3 倍、复制延迟大。Aurora 提出"存储即多副本"，redo 只落一次存储层，RO 直接从共享存储读——TDSQL-C 沿此路线。

---

## 总体架构：数据面 + 支撑环境

### 数据面（业务直接感知）

```
客户端 → VIP → [TGW(L4) / Proxy(L7)] → RW 节点 ⇄ 共享存储(DBStore)
                                  └──→ RO 节点 ─┘
```

### 支撑环境（业务不感知，缺了就不可用）

| 组件 | 职责 | 关键点 |
|------|------|--------|
| **管控（OSS）** | 实例生命周期：创建/销毁/变配/参数 | cdb_worker、管控 DB |
| **HA 系统** | 故障检测 + 主从切换 + VIP 重绑 | ha_agent/ha_scheduler、switch_slave_*.cgi |
| **拨测** | 写拨测探测节点真实可用性 | pingsvr，15s 不通即触发 HA |
| **存储管控** | DBStore 卷管理、IO 限速 | dbmaster/pagestore/dbclient |
| **备份恢复** | 物理备份/PITR/快照 | xtrabackup 系 |
| **监控** | 指标采集 + 告警 | metricbeat/filebeat/Grafana |
| **CDC** | binlog 变更捕获供下游 | 数据订阅 |
| **Proxy** | 连接保持、读写分离、会话恢复 | Magpie |

**判别法**：断掉后"业务立刻不可用"的是数据面；"当下没事，出故障时没人兜底"的是支撑环境。

---

## 网络体系

### 三层概念辨析

| 概念 | 本质 | 类比 |
|------|------|------|
| **VPC** | 私有网络空间（IP 段/子网/路由表/安全组） | 你租的楼 |
| **VIP** | 不绑网卡的虚拟 IP，由路由规则动态指向 | 前台总机号 |
| **TGW** | L4 网关（TCP/UDP 转发+健康检查） | 前台接待员 |

```
客户端 → VIP(10.0.1.100) → TGW/VPCgw → RW 物理IP(10.0.1.10)
                              ↑ HA 切换改这里的路由指向
```

### 持有 VIP 的三种组件（层次不同）

| 组件 | 层次 | HA 时前端 TCP |
|------|------|---------------|
| VPC Gateway | L3 路由 | **断**（不保留连接上下文） |
| TGW | L4 转发 | **断** |
| Proxy | L7（理解 MySQL 协议） | **保持**（后端重建，前端不动） |

### 接入方式：线上 vs 线下

| 方式 | 场景 | 路径 | 特点 |
|------|------|------|------|
| **PrivateLink** | 线上（云上 VPC→云数据库 VPC） | VPC↔VPC 内网直连 | 不走公网、<1ms、免公网费 |
| **VPC Gateway/公网** | 线下（本地/自建 IDC→云数据库） | 公网/VPN/专线→VPC | 需白名单+SSL、受公网延迟 |

### TDSQL-C VIP 切换依赖（因实例类型）

| 实例类型 | VIP 切换依赖 |
|----------|-------------|
| 同园区 TGW 实例 | TGW 接口 + cdb_worker/ajs |
| 同园区非 TGW | VPC 接口 |
| 跨园区 | VPC 接口 |
| 开了 Proxy | Proxy（无论上表哪种） |

### K8s 网络补充（业务侧访问数据库时）

| 概念 | 要点 |
|------|------|
| Service/ClusterIP | K8s 访问抽象；ClusterIP 是 Service 的一种类型（集群内虚拟 IP，kube-proxy iptables/ipvs 转发到 Pod） |
| Overlay | VXLAN 封装，Pod IP 独立段（如 10.244.x.x），与 VPC 不直接通，需 NAT/Service 桥接 |
| Underlay | Pod IP 即 VPC 子网 IP，无封装、性能优，可直接访问 VPC 内云数据库 |
| jnsgw | 基础网络网关（逐步淘汰）；vpcgw=VPC 网关（子网间/跨 VPC/NAT/专线） |

---

## HA 系统

### 故障发现（三条途径）

| 途径 | 机制 | 触发条件 |
|------|------|----------|
| **拨测** | pingsvr 短连写拨测 | 连续不通 > 15s |
| **磁盘监控** | RS 上 dd 写测 | 磁盘不可写 |
| **错误日志监控** | 扫 err log | terribly wrong / system error 等关键字 |

拨测 SQL：
```sql
set @@binlog_format=ROW;
create temporary table mysql._CDB_PING_ (id int);
```

拨测发起 HA 的错误码：`10/2003/2006/2013/1896/3198/4504/4505/1114/3675`

### 切换四阶段

```
阶段1 检查 → 是否真需要切换（写拨测不通/负载高/频繁重启）
阶段2 选主 → 指定或自动（健康优先 + 数据最多 + 同园区优先）
阶段3 同步 → ★先置空 VIP（旧主不再收新连接）→ 追数据
阶段4 切VIP → VIP 绑到新主 + 连通性检查（幂等）
```

**核心原则：先同步数据，再切换 VIP**。阶段 3 置空 VIP 是关键——避免切换窗口内写入丢失。

### VIP 切换对客户端的影响

**传统切换（无 Proxy）：TCP 必断**

TCP 连接是四元组 `(客户端IP, port) ↔ (VIP, port)`。VIP 重绑后：
- 旧主上内核 socket 失效（不再持有 VIP）
- 新主 mysqld 是新进程，无该连接的 socket 上下文
- 客户端必须重连

影响：所有连接断（含空闲）+ 未提交事务回滚 + **COMMIT 中事务结果不确定** + 新主 BP 冷启动性能抖动。

**有 Proxy：前端 TCP 保持**（见透明切换 L2/L3）。

---

## 内核侧角色切换

与管控侧 VIP 切换并行，内核执行角色翻转：

| 场景 | 流程 | 关键操作 |
|------|------|----------|
| **计划内切换** | RW 降级 → RO 升级 | `ALTER CYNOS TO REPLICA`（写 DEMOTE 日志保证一致性）→ `ALTER CYNOS TO PRIMARY` |
| **计划外（旧 RW 不可达）** | RO 强制提主 | `ALTER CYNOS FORCE TO PRIMARY`；清空 BP+cache（日志可能不完整） |
| **新流程** | 一步提主 | CDP 补全日志，不再"降主再升主" |

### RO 升级为新 RW 的内部步骤

1. 关闭 rpl sys（停止从旧主拉日志）
2. 以 RW 角色重连存储（DBStore）
3. 初始化 log sys / trx sys
4. **reopen binlog，递增全局 binlog index**（HA 后第一条 binlog）
5. 启动 RW 独有后台线程：purge / fts / gtid persistent

### tencentroot 账号

- TDSQL-C 内部超级账号，运维操作必须在 **RW 节点**上执行
- 已知坑：①线程池未适配 admin port 时 tencentroot 连接计数为负→连不上；②启动后首个非白名单 IP 的 tencentroot 连接会污染认证方式→后续白名单连接失败（修复：白名单用户强制 native 认证）

---

## 透明切换分级

### 能力分级（L0~L6）

| 级别 | 能力 | 实现手段 |
|------|------|----------|
| L0 | 数据不丢 | 已有（同步后切换） |
| L1 | 快速切主 | **在线提主**：RO 不重启完成角色升级 |
| **L2** | **连接保持** | **Proxy 保留前端 TCP**，后端重建到新主 |
| **L3** | **会话恢复** | Proxy 同步通道：schema/变量/PS 状态恢复到新主 |
| L4 | 性能连续 | 在线 BP 预热（切换前新主预加载热页） |
| L5 | 提交结果确定 | LTXID + lpos（客户端可查询事务是否提交） |
| L6 | 未提交事务续传 | Attach + 语句级 checkpoint |

### 三条状态恢复通道

| 通道 | 内容 | 特点 |
|------|------|------|
| 存储共享 | redo/undo/binlog 在 DBFS | 天然共享，无需同步 |
| RO 同步 | 内存状态（BP/dict cache/lock） | 内存到内存，异步或强同步 |
| Proxy 同步 | 协议层可见状态（session vars/PS/临时表） | Proxy 缓存，切换时下发到新主 |

---

## 关键流程链路

### HA 切换完整链路（管控+内核+网络）

```
拨测失败15s / 磁盘故障 / errlog 关键字
  → ha_scheduler 判定
  → 选主（健康+数据最多+同园区）
  → 内核: 旧 RW 写 DEMOTE 日志(计划内) 或 RO FORCE 提主(计划外)
  → 置空 VIP（旧主解绑）
  → 等数据同步追平
  → VIP 重绑新主（TGW/VPC/Proxy 接口）
  → 新主 reopen binlog(递增 index) + 启动 purge/fts 线程
  → 拨测恢复确认
```

### 透明切换链路（有 Proxy）

```
HA_PLANNED_SWITCH_START 信号 → Proxy
  → Proxy 冻结新事务（短暂）
  → 保留前端 TCP（L2 核心）
  → 预建新主连接
  → 空闲连接做 Session 恢复（L3：变量/PS/schema 下发新主）
  → 切换完成，恢复流量
```

---

## 关键组件速查

| 组件/接口 | 位置 | 说明 |
|-----------|------|------|
| `pingsvr` | 拨测服务 | 短连写拨测，15s 不通触发 HA |
| `switch_slave_rw_inst.cgi` | 切换接口 | RW 切换 |
| `switch_slave_by_ping.cgi` | 切换接口 | 拨测触发 |
| `switch_slave_cause_by.cgi` | 切换接口 | 磁盘故障触发 |
| `switch_slave_by_ha_scheduler.cgi` | 切换接口 | RO 踢除 |
| `ALTER CYNOS TO PRIMARY/REPLICA` | 内核 | 角色 DDL（NCDB 扩展） |
| core 文件 | `/data/coredump/back/` | `core_mysqld_<pid>`，用 base_phony 对应版本 mysqld 打开 |
| err log | `/data1/mysql_root/log/<port>/NCDB_TXSQL.err` | 含 crash 前上下文 |
| `cynos_version` | errlog 首行 | 定位内核版本（base_phony 被替换时用） |
| HA 后旧节点日志 | `/data/cdb_log/` | HA 发生后被挪到这里 |

---

## Misc

### 运维排查思路（ping 通 ≠ 服务通）

| 层次 | 验证手段 | 说明 |
|------|----------|------|
| DNS | `dig` / DoH (`curl https://223.5.5.5/resolve?name=...`) | 公司内网 DNS 可能 NXDOMAIN，DoH 是旁路验证 |
| ICMP | `ping` | 只证明入口 IP 活着 |
| TCP | `nc -zv host port` | 只证明三次握手 |
| 应用协议 | `nc < /dev/null` 抓 banner / `curl` | MySQL 应主动发 banner；无响应=后端挂 |
| 完整链路 | `mysql -h ... -e "SELECT 1"` | 到认证层才算通 |

典型误判链：ping 通 → 以为服务正常 → 实际是 SLB 入口活着、后端挂了（返回 HTTP 502 或无 banner）。

### cgroup 容器监控（节点侧）

| 指标 | 来源文件 | 含义 |
|------|----------|------|
| `memory_limit_in_bytes` | cgroup 伪文件 | 容器内存上限 |
| `memory_usage_in_bytes` | cgroup 伪文件 | 总占用（**含 page cache**） |
| `memory_stat_rss` | `memory.stat` | 用户态进程实际内存 |

`usage ≥ rss` 恒成立，差值是 page cache。K8s 默认经 cAdvisor（kubelet 内置）采集暴露给 Prometheus。cgroup OOM 会直接 SIGKILL mysqld，无 stack trace。

### 术语对照

| 术语 | 含义 |
|------|------|
| RS | Real Server，网关后端真实节点 |
| RW | 读写节点（主） |
| RO | 只读节点（从） |
| admin port | 节点管理端口（如 20145），运维/拨测用 |
| GTW | gateway 统称（vpcgw/jnsgw/TGW） |
| DBFS | TDSQL-C 的数据库文件系统（对接 DBStore） |
| CDP | 切换时补全日志的机制（新流程提主用） |

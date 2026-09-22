# binlog 加密（binlog_encryption）

> 基于 MySQL 8.0.39 源码，剖析 binlog/relay log **文件级加密**（8.0.14 引入）：两级密钥（File Password + Replication Encryption Key）、512 字节加密文件头（TLV）、`Aes_ctr_cipher` 流密码、`Binlog_encryption_ostream/istream` 加解密流、keyring 主密钥轮换 + `reencrypt_logs` 重加密、以及读侧（dump/恢复/mysqlbinlog）的透明解密。
>
> **边界**：本篇讲"文件如何被加密/解密、主密钥如何轮换"。加密文件头的字节布局已在 [`binlog.md`](binlog.md)「binlog 文件加密」有一小节，本篇挖深到代码级；事件序列化/checksum 见 [`binlog_event.md`](binlog_event.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`binlog_encryption` 是 MySQL 8.0.14 引入的**文件级**加密：binlog 文件和 relay log 文件（统称 replication logs）在落盘前用流密码加密。官方头文件 `sql/rpl_log_encryption.h` 把边界定得很死：

> "A replication log file is either encrypted or not ... It is not possible that part of a log file is encrypted and part of it is non-encrypted."

即：**整个文件要么全加密、要么全不加密**，不存在"部分加密"。加密发生在**文件头之后的整个数据区**（含 `BINLOG_MAGIC` 本身）。

### 加密对象与位置

- 对象：binlog 文件 + relay log 文件；
- 加密粒度：文件级，非 event 级（与 [`binlog_event.md`](binlog_event.md) 的 event 无关，也与事务压缩 `Transaction_payload_event` 无关）；
- 层级位置：在写入 pipeline 里，加密是最外层（`压缩 event → 写 IO_CACHE → 加密流 → 落盘`），读侧反向（`读文件 → 解密流 → 读 IO_CACHE`）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0.14 | 引入 `binlog_encryption`，支持 binlog 文件加密 |
| 8.0.17 | 扩展支持 relay log 加密；主密钥轮换（`ALTER INSTANCE ROTATE BINLOG MASTER KEY`） |
| 8.0.28 | 引入 `binlog_rotate_encryption_master_key_at_startup` 启动自动轮换 |

---

## 理论基础

### 设计思想：为什么是两级密钥

加密文件的一个经典难题是**主密钥轮换**：如果所有文件直接用一把 master key 加密，轮换 master key 就要重写所有文件的全部密文（O(文件总量)），代价极高。MySQL 用**两级密钥**解耦：

```
Replication Encryption Key（主密钥，存 keyring，32 字节）
        │  AES-256-CBC 加密（IV 也存文件头）
        ▼
File Password（每文件 32 字节随机数）──存文件头（密文）
        │  EVP_BytesToKey(SHA-512, count=1, 无 salt) 派生
        ▼
File Key（32 字节）── AES-256-CTR 加密
        ▼
文件数据区（含 BINLOG_MAGIC 的整个 binlog 内容）
```

- **第一级 File Password**：每个文件一把独立的 32 字节随机数，用主密钥加密后存文件头。真正加密数据区的密钥（File Key）由它派生。
- **第二级 Replication Encryption Key**：全局主密钥，由 keyring 生成并存储，只用来加密/解密各文件的 File Password。

**轮换主密钥的成本因此从 O(文件总量) 降到 O(文件数量)**：只需用新主密钥重新加密每个文件头的 32 字节 File Password（`reencrypt_logs` 做的就是这个），数据区密文完全不用动。官方注释把动机写得极清楚（`rpl_log_encryption.h` 的 "Two Tier Keys"）。

### 算法取舍：AES-CBC 加密口令 vs AES-CTR 加密数据

- **AES-256-CBC** 加密 File Password：口令是固定 32 字节、一次性加密，CBC 需要 IV（16 字节，也存文件头 `IV_FOR_FILE_PASSWORD`），没有随机访问需求，用 CBC 合适。
- **AES-256-CTR** 加密数据区：binlog 是**流式追加**，且读侧要**随机 seek**（dump 线程按位点跳转、恢复扫描随机读）。CTR 模式的 counter 由字节偏移决定，天然支持"从任意 offset 开始解密"，这是选 CTR 而非 CBC 的关键——CBC 解密依赖前一个密文块，无法随机访问。

`Aes_ctr`（`sql/stream_cipher.h`）把这个取舍固化：

```cpp
class Aes_ctr {
  static const EVP_MD *get_evp_md() { return EVP_sha512(); }      // 口令派生摘要
  static const EVP_CIPHER *get_evp_cipher() { return EVP_aes_256_ctr(); }
};
```

### 数据结构：TLV 文件头

加密文件头用 TLV（type-length-value）编码，总长固定 512 字节（`Rpl_encryption_header_v1::HEADER_SIZE`），尾部补 0。三种字段类型（`enum Encryption_type`）：`KEY_ID=1`、`ENCRYPTED_FILE_PASSWORD=2`（32 字节）、`IV_FOR_FILE_PASSWORD=3`（16 字节）。

---

## 核心实现

### 两级密钥的载体：Rpl_encryption

`Rpl_encryption`（`sql/rpl_log_encryption.h`，全局单例 `rpl_encryption`）是加密功能的容器类，关键接口：

- `initialize()`：启动时初始化，读取 keyring 里的主密钥；
- `recover_master_key()`：恢复主密钥（处理上一次未完成的轮换）；
- `get_master_key()`：返回当前主密钥（`Rpl_encryption_key{m_id, m_value}`）；
- `get_key(key_id, key_type, key_size)`：从 keyring 取指定 key（带缓存）；
- `enable(THD*)` / `disable(THD*)`：开启（生成主密钥 + rotate）/ 关闭（仅 rotate）；
- `purge_unused_master_keys()`：轮换后清理旧主密钥。

主密钥的存储与获取都通过 keyring，`Keyring_status` 枚举定义了 9 种 keyring 错误（`KEYRING_ERROR_FETCHING`、`KEY_NOT_FOUND`、`UNEXPECTED_KEY_SIZE`、`UNEXPECTED_KEY_TYPE`、`KEY_EXISTS_UNEXPECTED`、`KEYRING_ERROR_GENERATING`、`KEYRING_ERROR_STORING`、`KEYRING_ERROR_REMOVING`），任何一步失败都要能精确上报。

### 加密文件头：Rpl_encryption_header_v1

`Rpl_encryption_header` 是基类（负责序列化/反序列化文件头），`Rpl_encryption_header_v1` 是 v1 实现（`sql/rpl_log_encryption.h`）：

```
+----------------+---------------------------------------------------+
| 固定头 (5B)     | ENCRYPTION_MAGIC (4B, 0xFD62696E) + version (1B)   |
+----------------+---------------------------------------------------+
| TLV 字段        | type=1 KEY_ID                                   |
|                | type=2 ENCRYPTED_FILE_PASSWORD（32B）            |
|                | type=3 IV_FOR_FILE_PASSWORD（16B）               |
+----------------+---------------------------------------------------+
| 填充            | 补 0 到 HEADER_SIZE = 512                        |
+----------------+---------------------------------------------------+
| 加密数据区       | （含 BINLOG_MAGIC 的整个 binlog 内容）            |
+----------------+---------------------------------------------------+
```

`ENCRYPTION_MAGIC` 是 `0xFD62696E`，与普通 binlog 的 `BINLOG_MAGIC = 0xFE62696E` 只差一个字节——读侧靠这 1 字节之差判断"这个文件是否加密"。关键方法：

- `encrypt_file_password()`：用主密钥（AES-256-CBC）加密 File Password；
- `decrypt_file_password()`：反向解密；
- `get_encryptor()` / `get_decryptor()`：用解密出的 File Password 派生 File Key，创建 `Stream_cipher`。

### 流密码：Stream_cipher 与 Aes_ctr_cipher

`Stream_cipher`（`sql/stream_cipher.h`）是加解密流密码的抽象接口，支持**顺序**和**随机**两种访问模式：

```cpp
class Stream_cipher {
  virtual bool open(const Key_string &password, int header_size) = 0;  // 用口令初始化
  virtual bool encrypt(unsigned char *dest, const unsigned char *src, int length) = 0;
  virtual bool decrypt(unsigned char *dest, const unsigned char *src, int length) = 0;
  virtual bool set_stream_offset(uint64_t offset) = 0;  // ★ 随机访问：设当前偏移
};
```

`Aes_ctr_cipher<TYPE>`（模板，`TYPE` 是 `Cipher_type::ENCRYPT`/`DECRYPT`）是 AES-256-CTR 实现：

```cpp
template <Cipher_type TYPE>
class Aes_ctr_cipher : public Stream_cipher {
  EVP_CIPHER_CTX *m_ctx;                      // OpenSSL 上下文
  unsigned char m_file_key[FILE_KEY_LENGTH];  // 32 字节 File Key
  unsigned char m_iv[AES_BLOCK_SIZE];         // 16 字节 IV
  bool init_cipher(uint64_t offset);          // 由 offset 计算 CTR counter 写入 IV
};
```

**关键点**：`set_stream_offset(offset)` 配合 `init_cipher(offset)`——AES-CTR 的 counter 由字节偏移计算，所以从任意 offset 都能正确恢复 keystream，无需从头解密。这是 dump 线程"从指定位点发 binlog"、恢复扫描"随机读事件"能对加密文件工作的根本原因。

**`open(password, header_size)` 的完整逻辑**（`sql/stream_cipher.cc`）：

```cpp
bool Aes_ctr_cipher<TYPE>::open(const Key_string &password, int header_size) {
  m_header_size = header_size;                       // 记录数据区起点偏移
  if (EVP_BytesToKey(Aes_ctr::get_evp_cipher(),     // AES-256-CTR
                     Aes_ctr::get_evp_md(),          // SHA-512
                     nullptr,                        // 无 salt
                     password.data(), password.length(),
                     1,                              // count=1（单次摘要，无迭代拉伸）
                     m_file_key, m_iv) == 0)
    return true;                                     // KDF 失败
  return init_cipher(0);                             // 初始化 CTR，counter=0
}
```

**一个重要的实现细节（纠正）**：File Key 的派生**不是 PBKDF2，也不是直接 hash**，而是 OpenSSL 的 `EVP_BytesToKey`（传统 KDF）——用 SHA-512 对 file password 做 `count=1`（等价单次摘要、无迭代拉伸）得到 64 字节输出，前 32 字节作 `m_file_key`、后 16 字节作 `m_iv`。为什么敢用无 salt、无迭代的弱 KDF？因为 **file password 本身是每文件随机生成的 32 字节高熵数**，不需要 salt 和迭代来对抗低熵口令——这是「密钥源足够随机，KDF 只需拉伸长度」的典型取舍。

**`init_cipher(offset)` 的逐行逻辑**（CTR 随机访问的核心）：

```cpp
bool Aes_ctr_cipher<TYPE>::init_cipher(uint64_t offset) {
  uint64_t counter = offset / AES_BLOCK_SIZE;         // ★ 字节偏移 → 块计数器
  ...
  memcpy(m_iv + AES_BLOCK_SIZE - 8, &counter, 8);     // 把 counter 写入 IV 低 8 字节
  EVP_CipherInit_ex(m_ctx, ..., m_file_key, m_iv, ...);
  EVP_CIPHER_CTX_set_padding(m_ctx, 0);               // CTR 无需 padding
}
```

AES-CTR 的 keystream 由「IV + counter」决定，而 counter = 字节偏移 / 16，所以从任意 offset 都能精确重建该位置的 keystream——这就是 `seek` 能工作的底层数学。`m_header_size` 记录了数据区起点，`seek(offset)` 时下游要 seek 到 `offset + header_size`（跳过加密文件头）。

### 加密流：Binlog_encryption_ostream

`Binlog_encryption_ostream`（`sql/binlog_ostream.h`）是写入 pipeline 的一层，继承 `Truncatable_ostream`：

```cpp
class Binlog_encryption_ostream : public Truncatable_ostream {
  bool open(unique_ptr<Truncatable_ostream> down_ostream);              // 新建：生成新 password
  bool open(..., unique_ptr<Rpl_encryption_header> header);             // 已加密：复用 header
  std::pair<bool,std::string> reencrypt();                              // 重加密文件头
  bool write(...) override; bool truncate(...); bool seek(...);
  unique_ptr<Truncatable_ostream> m_down_ostream;   // 下游（IO_CACHE_ostream）
  unique_ptr<Rpl_encryption_header> m_header;
  unique_ptr<Stream_cipher> m_encryptor;            // AES-CTR 加密器
};
```

`write` 时，数据先经 `m_encryptor->encrypt()` 再写下游。两个 `open` 重载区分"新建文件"（生成新 File Password + 写文件头）和"重开已加密文件"（读文件头复用 File Password）。

### 解密流：Binlog_encryption_istream 与透明判定

读侧的解密同样是一层流，`Binlog_encryption_istream`（`sql/binlog_istream.h`）继承 `Basic_seekable_istream`：

```cpp
ssize_t Binlog_encryption_istream::read(unsigned char *buffer, size_t length) {
  ssize_t ret = m_down_istream->read(buffer, length);       // 读密文
  if (ret > 0 && m_decryptor->decrypt(buffer, buffer, ret)) ret = -1;  // 原地解密
  return ret;
}
bool Binlog_encryption_istream::seek(my_off_t offset) {
  bool res = m_decryptor->set_stream_offset(offset);        // 设 CTR counter
  if (!res) res = m_down_istream->seek(offset + m_decryptor->get_header_size());
  return res;
}
```

**透明判定的入口**是 `Basic_binlog_ifile::read_binlog_magic()`（`sql/binlog_istream.cc`）：读文件头 4 字节 magic，若是 `ENCRYPTION_MAGIC` 就套一层 `Binlog_encryption_istream`，否则按普通 binlog 读。因此**上层（dump、恢复、mysqlbinlog）完全不感知加密**——它们拿到的一律是解密后的字节流。这也是"文件级加密对读侧透明"的实现点。

### keyring 轮换 + reencrypt_logs

主密钥轮换入口是 SQL 命令 `ALTER INSTANCE ROTATE BINLOG MASTER KEY`（`Rotate_binlog_master_key::execute`）。轮换被建模成一个**可恢复的 8 步状态机**（`Rpl_encryption::Key_rotation_step`）：

```
START → DETERMINE_NEXT_SEQNO → GENERATE_NEW_MASTER_KEY → REMOVE_MASTER_KEY_INDEX
      → STORE_MASTER_KEY_INDEX → ROTATE_LOGS → PURGE_UNUSED_ENCRYPTION_KEYS
      → REMOVE_KEY_ROTATION_TAG
```

官方注释强调"rotation process is recoverable"——每一步写入 keyring 的中间状态（sequence number、rotation tag），崩溃后 `recover_master_key()` 能从 keyring 读出"上次轮换停在哪一步"继续，保证轮换的原子性。

轮换的核心动作是 `MYSQL_BIN_LOG::reencrypt_logs()`（`sql/binlog.cc`）：

1. 从 index 文件读所有 binlog 文件名；
2. **跳过最后一个**（当前正在写的文件）；
3. **逆序**遍历（从最新到最旧）；
4. 每个文件 `open_existing` → `is_encrypted()` 判断 → 取出 pipeline 头的 `Binlog_encryption_ostream` → 用新主密钥重新加密 File Password。

之所以逆序 + 跳过最后：正在写的文件由后续 rotate 自然切换，旧文件按序重写文件头即可。relay log 的重加密走 `rpl_replica.cc` 里的 `relay_log.reencrypt_logs()`，由 `flush_relay_logs_cmd` 触发。注意 reencrypt 是**同步**执行（在 ALTER INSTANCE 命令内），不是后台线程——期间 dump 线程仍可读，因为重加密只改文件头、不动数据区。

### 写入链路层级

加密在写入链路的**最外层**（相对 checksum 和事务压缩）：

```
事务提交 → 事件序列化（含 checksum CRC32）
  → 事务压缩（Transaction_payload_event，可选）
  → binlog cache 刷写 → IO_CACHE
  → Binlog_encryption_ostream（加密）   ← 最外层
  → 文件
```

checksum 在事件内（event 级），压缩在事务内（transaction 级），加密在文件级——三层正交，互不干扰。

**读写两侧的装饰器链完全对称**（类层次的加密层，详见 [`io_cache.md`](../infra/io_cache.md) 8.5/8.10）：

```
写侧:  Binlog_ofile ──> Binlog_encryption_ostream ──> IO_CACHE_ostream ──> IO_CACHE ──> 文件
       (门面 Facade)     (分块加密 + 偏移翻译)         (WRITE_CACHE)

读侧:  Basic_binlog_ifile ──> Binlog_encryption_istream ──> IO_CACHE_istream ──> IO_CACHE ──> 文件
       (门面 Facade,           (读后原地解密 + 偏移翻译)      (READ_CACHE)
        read_binlog_magic
        按 magic 动态套解密层)
```

两侧的 `seek` 都做 `offset ± header_size` 偏移翻译、都靠 `set_stream_offset` 重置 CTR counter——这是「装饰器链 + 偏移翻译」设计在读写两侧的镜像。

---

## ★ 本机制里的工程实现技法

### 一、模板分派加解密（Cipher_type）

`Aes_ctr_cipher<Cipher_type::ENCRYPT>` 和 `Aes_ctr_cipher<Cipher_type::DECRYPT>` 是**同一份代码**，用模板参数区分加密/解密方向（`Cipher_type` 枚举），`typedef` 出 `Aes_ctr_encryptor`/`Aes_ctr_decryptor` 两个别名。这样加解密逻辑只写一份，OpenSSL 的 `EVP_EncryptUpdate`/`EVP_DecryptUpdate` 由模板特化分派，避免复制粘贴两份几乎相同的代码。

### 二、流式 pipeline 的分层封装

加密/解密都实现为 `ostream/istream` 的**可叠加层**：

```
写侧：IO_CACHE_ostream ← Binlog_encryption_ostream（加密层）
读侧：文件 ifile ← Binlog_encryption_istream（解密层）← 上层 reader
```

每一层只关心自己的职责（加密层管加解密，下游层管 IO），通过 `unique_ptr` 持有下游，天然支持"动态决定是否加密"——`read_binlog_magic` 里判断 magic 后选择是否插入解密层，上层代码零改动。

### 三、可恢复的轮换状态机

主密钥轮换被拆成 8 个有名字的步骤，每步的状态（sequence number、rotation tag）落 keyring，`recover_master_key` 启动时读取并续跑。这是「长事务/多步操作的可恢复性」的经典解法：不追求单步原子，而是保证每一步幂等 + 状态可查，崩溃后从断点继续。

### 四、CTR 的随机访问能力

`set_stream_offset(offset)` 让加密流支持随机 seek——这是 CTR 模式相对 CBC 的独特优势（counter = f(offset)）。binlog 的 dump 线程、恢复扫描都依赖随机访问，选 CTR 是"算法能力匹配业务需求"的典型决策。

| 特性 | 用在哪 | 收益 | 代价 |
|---|---|---|---|
| 模板分派加解密 | `Aes_ctr_cipher<TYPE>` | 一份代码两方向 | 模板实例化两份 |
| 两级密钥 | File Password + 主密钥 | 轮换成本 O(文件数) | 多一层派生 |
| TLV 文件头 | `Rpl_encryption_header_v1` | 可扩展字段 | 512 字节固定开销/文件 |
| 可恢复轮换状态机 | `Key_rotation_step` | 崩溃可续跑 | 8 步状态要落 keyring |
| CTR 随机访问 | `set_stream_offset` | dump/恢复可随机读 | CTR counter 需正确计算 |

---

## 可观测性

### 系统变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `binlog_encryption` | OFF | 动态全局，需 `SUPER_ACL` 或 `BINLOG_ENCRYPTION_ADMIN` 权限，`NOT_IN_BINLOG` |
| `binlog_rotate_encryption_master_key_at_startup` | OFF | 只读，启动时自动轮换主密钥 |

### 观测手段

| 我想看 | 手段 |
|--------|------|
| 文件是否加密 | 看文件头 4 字节：`0xFD62696E` 是加密，`0xFE62696E` 是普通 |
| 主密钥轮换 | `ALTER INSTANCE ROTATE BINLOG MASTER KEY`，keyring 里 sequence number 递增 |
| 加密配置 | `SHOW VARIABLES LIKE 'binlog_encryption'` |

---

## Misc

### 扩展点

- **新文件头版本**：`Rpl_encryption_header` 是基类，加 `Rpl_encryption_header_v2` 可扩展新字段（TLV 天然向后兼容，旧版本读到未知 type 可跳过）。
- **新加密算法**：`Stream_cipher` 抽象接口下加新实现即可，但数据区算法受制于"已写文件要能解"，切换需配合文件头版本。

### 坑与已知缺陷

- **文件级而非 event 级**：加密不感知 event，`binlog_transaction_compression`、checksum 都在加密之前完成，不要混淆三层（event/transaction/file）。
- **keyring 依赖**：主密钥在 keyring，keyring 丢失则所有加密 binlog 无法解密（数据永久不可读）——这是用 keyring 的固有风险。
- **`0xFD` vs `0xFE` 一个字节之差**：读侧判定加密就靠这一个字节，若文件头损坏可能误判。
- **reencrypt 是同步的**：`ALTER INSTANCE ROTATE BINLOG MASTER KEY` 会同步重加密所有旧文件头，binlog 文件多时可能阻塞较久，不是后台异步。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Encrypting Binary Log Files and Relay Log Files*
- *MySQL 8.0 Reference Manual → `ALTER INSTANCE ROTATE BINLOG MASTER KEY`*

**Worklog / Bug**
- WL#12371 / WL#13560 系列：binlog 加密与 relay log 加密

**相关文档**
- 加密文件头字节布局见 [`binlog.md`](binlog.md)「binlog 文件加密」
- 事件序列化与 checksum 见 [`binlog_event.md`](binlog_event.md)
- 事务压缩（加密的前一层）见 [`binlog.md`](binlog.md)「binlog 事务压缩」

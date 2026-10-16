# Keystore —— `.sconekey` 密钥文件格式

> 规范性文档。本文档与
> `crates/scone-keystore/src/lib.rs` 之间的任何分歧都是
> bug（确定事实来源并修复）。实现参考：crate
> `scone-keystore`（v2）。

## 目的

将一个 Ed25519 私钥（*seed*，32 字节）以口令
加密的形式存储在磁盘上。该密钥**绝不**以
明文接触磁盘；所有秘密（口令、派生物料）在使用
后都在内存中被清零。

## 概览

- **KDF**：Argon2id（v0x13），默认参数 `m_cost` = 64 MiB、
  `t_cost` = 3、`p_cost` = 1（`argon2` crate 的默认值）。
- **加密**：XChaCha20-Poly1305（AEAD），32 字节密钥由
  KDF 派生，每文件 24 字节随机 nonce。
- **加密内容**：恰好是 Ed25519 seed（32 字节）；
  Poly1305 标签（16 字节）作为密文的
  后缀存储（AEAD `encrypt` API 的标准输出）。

## 字节封装（`bytes` 模块，web）

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` 在
不触碰文件系统的情况下产生并消费
相同的格式：web 客户端在浏览器中生成身份
（`scone-crypto` + bip39）并封装精确的密钥文件字节
（可下载、可被 CLI 原样打开 —— 测试套件中有
互操作测试）。原生路径
（`create`/`create_with_phrase`）委托给同样的
封装：一种格式编码，三种去向。

## 二进制格式（v2）—— 52 字节头 + 公开元数据 + 密文

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII — unchanged since v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      32          public_key: Ed25519 public key (32 bytes, IN THE CLEAR)
84      8           created_at: u64 big-endian (Unix, seconds)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (readable without the passphrase, may be empty)
—       48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

`public_key`/`created_at`/`label` 元数据是公开的：**无需
口令**即可读取（`public_metadata_from_bytes`）。公钥
天生公开；标签只是一个显示名 —— 隐藏它
毫无收益，却让无口令列出变得不可能。总长
取决于标签：标签为空时 141 字节
（KEYFILE_LEN + 32 + 8 + 1），再加上标签的长度。

**v1（已被取代，恰好 100 字节）**：magic ‖ version=1 ‖
m ‖ t ‖ salt ‖ nonce ‖ 密文 —— 不存储公钥（上锁的
文件是匿名的）。v1 文件仍然**可读**
（元数据为 `None`，`identity show` 需要口令；
通过 `scone identity restore` 重新封装即可迁移到
v2）。

### 设计选择

- **KDF 参数写在文件里**：`m_cost_mib` 与 `t_cost`
  从头部读取（读取时不硬编码），使未来版本
  可以提高它们而不破坏现有文件。
- **读取时的安全边界**：声明
  `m_cost_mib` > 64 或 `t_cost` = 0 或 > 10 的文件会被拒绝
  （`Error::InvalidParams`），*在任何分配之前* —— 恶意
  文件无法强迫节点分配数 GB 内存或无限
  循环。写入时的默认值为 `m_cost_mib` = 64、
  `t_cost` = 3。
- **显式 magic + 版本**：不是密钥文件的文件
  （`Error::InvalidFile`）或不受支持版本的文件
  （`Error::UnsupportedVersion`）被干净地拒绝，
  不会 panic。
- **错误口令**：Poly1305 认证失败不会指明
  失败*原因*；因此错误的口令与损坏的
  密文无法区分，以同一个独立错误
  `Error::WrongPassphrase` 呈现。

## 加密密钥派生

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

然后：

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

v1 中没有关联数据（AAD）。每文件随机的
24 字节 nonce 使 nonce 重用风险可忽略
（密钥文件只写入一次）。

## API（`scone-keystore`）

| 函数 | 职责 |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | 生成密钥，写入加密的密钥文件（创建父目录）；**拒绝覆盖**已存在的文件（`Error::KeyfileExists`）；在内存中返回密钥 |
| `create_overwriting(path, passphrase) -> GeneratedKey` | 同上，但显式替换已存在的密钥文件 |
| `open(path, passphrase) -> SigningKey` | 解密并返回签名密钥 |
| `list(dir) -> Vec<KeyEntry>` | 列出目录中的 `*.sconekey` 文件（按名称排序）；目录缺失 = 空列表 |

`GeneratedKey.signing_key` 在 drop 时由
`ed25519-dalek` 清零（默认 `zeroize` 特性）。
派生的 AEAD 密钥、seed 以及
解密后的明文都包在 `zeroize::Zeroizing` 中：
Drop 覆盖所有退出路径，包括由 `?` 传播的错误。

## 文件权限

在 Unix 上，密钥文件以权限 `0600`（仅所有者
可读写）创建，通过 `OpenOptions` +
`set_permissions` —— 与进程 umask 无关。存在性
检查与创建是原子的（`create_new`、`O_EXCL`）：
两次并发的生成不可能静默互相覆盖。

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- 默认目录：`$HOME/.scone/keys`。
- 身份名称被严格验证：空、`.`、`..`、`/`、
  `\\\\`、NUL、前导点以及 `.sconekey` 后缀都被拒绝
  （名称会成为密钥目录内的文件名 —— 任何
  路径组成部分都无法逃出 keystore）。
- 不带 `--passphrase-env` 时，口令在终端上
  **以掩码输入**请求（`rpassword`：不回显；
  `generate` 时输入两次并确认）。
- `identity generate` **恰好一次显示 BIP39 恢复短语
  （24 个词）** —— 全新 256 位熵，仅凭这些词
  即可重新生成同一密钥（`scone identity restore`）；
  它们从不存储在任何地方。
- `identity generate` **拒绝覆盖**已存在的
  密钥文件；`--force` 显式替换（新的 salt/nonce、新
  密钥、新短语）。
- `identity list` 对 v2 文件**无需口令**即显示
  公钥与标签（公开元数据）。
- `identity show` 读取 v2 密钥文件的公开元数据，
  **无需口令**即显示公钥、小写十六进制的
  `OwnerId`（64 个字符，与 `DomainId` 相同的约定）、
  标签及创建日期；v1 文件（无
  元数据）仍会索要口令。

## 此格式有意不做的事

- 不支持一文件多密钥（一文件一 seed）。
- 不支持更改口令（如果将来支持：以
  新的 salt/nonce 重写文件）。


## 默认身份（cli-simplify-v1）

需要签名身份的命令（`domain register`、
`domain set`、`domain update`、`domain renew`）在
keystore 恰好持有单一身份时无需 `--identity`
即使用它。有多个身份时命令失败并列出
它们（`scone identity list`）；请显式传
`--identity <name>`。

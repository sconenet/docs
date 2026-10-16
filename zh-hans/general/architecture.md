# 架构

## 先用大白话说

五个概念足以理解其余一切：

1. **一个名字** —— 互联网上的地址，例如 `mysite.scone`。
2. **你的 24 词助记短语** —— 你的身份。它既是证明名字属于你的
   地契，也为你的每一次变更签名。
3. **公共记录账本**（区块链）—— 谁拥有什么的共享
   公开账本。没有人控制它，也没有人
   能改写它。
4. **挖矿** —— 你保留（注册）一个名字的方式：你的计算机做一些
   对所有人公平一致的工作。没有排队，没有中间人。
5. **中继（relay）** —— 通过保存账本副本来
   运行网络并回答请求的计算机。任何人都可以运行
   中继；数量越多，网络越强健。

本页其余部分会更深入 —— 你可以到此为止，
这已经足够理解这个项目了。

## 指导原则

Scone 将权限与可用性分离：

- **区块链**是所有权及变更顺序的事实来源；
- **DHT** 是一个分发/复制系统，它**不是**权威；
- **本地存储**保存节点的状态、索引和缓存。

节点必须将来自 DHT 的任何数据视为潜在
恶意数据：只有在通过密码学验证
（所有者签名、与链上的所有权 / 序列号 /
record_hash 匹配）之后才会被接受。

## 职责分离

```text
BLOCKCHAIN
──────────
Ownership
Registration
Authorization
Ordering
Sequence
Record hash
Consensus information

DHT
───
DNS records
Signed records
Replication
Availability

LOCAL STORAGE
─────────────
Local state
Indexes
Cached DHT records
Blockchain state
```

## crate 与依赖关系

```text
scone-crypto     ← lowest layer: pure primitives over bytes
    ▲
scone-core       ← protocol types (depends on scone-crypto)
    ▲
scone-protocol  ← wire/blockchain binary format
    ▲
scone-blockchain← blockchain logic, canonical state (in memory)
    ▲               └ scone-topology (pure engine, zero network/fs/
                     async; fed by scone-network) — scone-genesis
                     (one-shot mainnet genesis tool binary)
scone-storage   ← local persistence: redb backend + verified snapshots
    ▲
scone-network   ← p2p relay: libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← encrypted `.sconekey` keyfiles (depends on scone-crypto)
    ▲
scone           ← binary / application
```

### 决策：`scone-core` 依赖 `scone-crypto`

`DomainId` 必须能从最底层开始计算
（`BLAKE3-256(prefix || canonical name)`），但具体的密码学
实现不能污染核心。解决方式：

- `scone-crypto` 是最底层：纯粹的字节上的确定性函数
  （外加用于密钥生成的操作系统 RNG），零核心类型，
  无网络/文件系统/异步依赖 → 不可能出现循环依赖；
- `scone-core` 仅将其用于哈希；
- Ed25519 签名和密钥位于 `scone-crypto` 中，
  作用于字节（由 `scone-protocol` 产生的规范载荷）。

### 各 crate 的职责

**scone-core** —— 名称（`TldName`、`DomainName`）、标识符
（`DomainId`、`OwnerId`、`PublicKeyRef`）、DNS 数据（`DnsRecord`、
`SignedDnsRecord`）、交易（`RegisterDomain`、`UpdateDomain`）。
确定且可移植。此 crate 中禁止出现：libp2p、redb、
tokio/异步、网络、文件、操作系统、DNS 服务器、区块链逻辑。

**scone-crypto** —— 密码学原语。当前包括：`hash256`
（BLAKE3）、Ed25519 签名与密钥（`SigningKey`、
`PublicKey`、`Signature`、严格批量验证）、
记录组的 AEAD 信封（XChaCha20-Poly1305）。计划中：Merkle 树。
规则：不使用自造算法，只使用经过审计的依赖。

**scone-protocol** —— 协议的二进制格式：交易、DNS 记录、
区块（带有不透明共识字段的 `Block`/`BlockHeader`）以及
P2P 消息的规范编码/解码（最小 LEB128 变长整数、裸 32 字节
标识符），在分配内存前检查大小限制、拒绝非规范形式，
并进行版本管理（`PROTOCOL_VERSION`）。不含共识、网络或
存储逻辑。格式参考：`/docs/en/technical/protocol.md`。

**scone-blockchain** —— 区块链逻辑与规范状态：
交易标识（`TxId`）、有序 Merkle 树、
`BlockHash`、确定性创世、区块与交易验证、
状态应用规则（`DomainState`/`ChainState`，位于内存中，
可按 `DomainId` 直接访问）、分叉检测与分支选择
（`fork_choice`：终局性下限 + 最长链 + 最低哈希
平局决胜）、检查点终局性以及共识抽象
（`trait Consensus`，默认为宽松实现）。
注册 PoW（挖矿/验证、按网络的难度）位于
`scone-core`。此 crate 中禁止出现：网络、DHT、存储（redb）、
DNS 服务器。参考：`/docs/en/technical/blockchain.md`。

**scone-storage** —— 本地持久化（已交付）。具体后端是
`redb`（`NodeStore`），隐藏在一个最小 trait 之后，
使区块链/DHT/DNS 永远不会与 redb 直接耦合；
也可以接入另一个后端（分布式 KV）。包括分页
KV 状态后端、区块日志、经校验的快照
（带哈希的清单 + 页）以及重启时的恢复/重放路径。参考：
`/docs/en/technical/storage.md`。

**scone-network** —— 节点的 p2p 中继：libp2p/QUIC swarm、
交易与区块中继（系统性的本地验证）、
有界内存池、有界区块同步（每请求 128 个区块）、用于
`SignedDnsRecord` 的 Kademlia DHT、UDP+TCP DNS 服务器
（GeoDNS 常开 —— 内嵌 GeoIP 表、递归回退）、锚节点的
检查点/心跳循环（CLI 通过直接的 p2p QUIC 与
中继通信，不再有本地 RPC）。可达性与发现：AutoNAT
v2（可达/处于 NAT 后的判定）、面向 NAT 后节点的
circuit-relay v2 + DCUtR、mDNS（零配置局域网发现，
在 devnet 上默认开启）和 UPnP（由路由器做 QUIC
端口映射，可选启用）。通过 `.env` 文件进行配置（见 `.env.example`）。
参考：`/docs/en/technical/relay.md`。

**scone-keystore** —— Ed25519 签名密钥的加密存储
（`.sconekey` 文件：Argon2id + XChaCha20-Poly1305）。这个 crate
拥有文件系统，从而使 `scone-crypto` 保持纯粹。参考：
`/docs/en/technical/keystore.md`。

**scone-topology** —— 用于本地 p2p 拓扑优化的纯引擎：
零网络、零文件系统、零异步。它既不了解 libp2p
也不了解 swarm：`scone-network` 向其输入观测数据（ping RTT、
连接情况），并接收邻域管理决策。

**scone-genesis** —— 用于主网
创世准备的一次性工具二进制（kickstarter）：带版本的 JSON 清单、
分配方案、确定性区块。

**scone** —— 二进制/CLI。只有实际使用某个 crate 时才
依赖它（当前包括：用于中继的 scone-network、scone-blockchain、
scone-storage、scone-keystore、scone-protocol、scone-core、
scone-crypto、scone-topology）。

## 节点

一个 Scone 节点组合了：

1. 一条区块链（共识验证、所有权状态）；
2. DHT 路由（`SignedDnsRecord` 的发布/检索）；
3. 本地存储（状态、索引、缓存）；
4. 前端的解析器 / DNS 服务器。

### 启动新节点（状态引导，P0.3/P0.4）

新节点不会重放创世：它从一个经过校验的
已终局状态启动。

```text
finalized checkpoint (committee, signature quorum — validated by
        │              the chain layer before any use)
        ▼
verified snapshot (manifest + paginated pages; the importer
        │              RECOMPUTES the SMT root from the entries
        │              and compares it to the signed state_root —
        │              a falsified/incomplete snapshot is rejected
        │              before any write)
        ▼
anchor block H (requested from the network BY HASH)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height)
        │
        ▼
synced — at boot, replay of ONLY the suffix above H
```

两类节点画像：

- **归档节点**（运营者标志）：保留所有历史区块、
  向网络提供这些区块、通过现有路径启动；
- **全状态节点**：持有 H 时刻的完整状态及其后缀，
  但不保留 H 以下的历史 —— 它验证未来的区块并
  提供状态，不提供历史。

该路径为可选（OPT-IN）：导入经校验的快照是一个显式的
运营者动作（`import_verified_snapshot`），绝不自动进行 ——
现有节点的默认行为不会改变。
规范性细节：`/docs/en/technical/storage.md` 中
"VERIFIED state snapshot" 与 "State bootstrap" 两节。

## DNS 解析流程

```text
DNS query (example.uip)
    │
    ▼
Node: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── authoritative DNS snapshot (RAM) ─► unknown/expired → NXDOMAIN
    │                                          (without touching the core)
    ├── sharded DNS cache (qname,qtype) ───► hit: shared entry,
    │                                          lazy verification
    ├── local cache (SignedDnsRecord) ─────► verification (signature, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ─────────────────► same verification
    │                                          (miss → 30 s negative cache)
    ▼
DNS answer (if and only if valid)
```

DNS 快照（每个活跃域名的存在性 + 过期时间，
在每个已应用区块之后发布）保证存在性
决策永远不会经过中继的运行循环：只有真正的
记录获取才会触及核心。

## 可演进性

- 带版本的前像（`SCONE-DOMAIN-V1`、`SCONE-PUBKEY-V1`、
  `SCONE-OWNER-V1` ……）：修改一条派生规则会创建一个
  新命名空间，而不是破坏现有的命名空间；
- `RecordData` 通过变体 + 针对读取时尚未知类型的
  `Unknown { type_code, data }` 进行扩展；
- 用于线上格式重大变更的 `PROTOCOL_VERSION`（`scone-protocol`）；
- 链上只存储哈希：大块数据留在 DHT 中。

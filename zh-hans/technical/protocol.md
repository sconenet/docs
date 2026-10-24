# 协议

Scone 协议的参考文档：二进制格式、规范
编码、交易、DNS 记录、区块、P2P 消息、限制及
版本兼容性。参考实现是
[`scone-protocol`](../../../crates/scone-protocol) crate。

标记为**（暂定）**的条目尚未定案。

## 协议版本

- 常量：`PROTOCOL_VERSION: u32 = 1`（`scone-protocol`）；
- 由 `Hello.version`（握手）和 `BlockHeader.version` 携带；
- 兼容性规则：
  - 收到的版本**高于**本地版本 → 拒绝
    （`ProtocolError::UnsupportedVersion`）；节点不猜测
    它不认识的版本的格式；
  - 版本 `0` → 无效；
  - 只存在一套格式：交易版本字节
    （`0x01`）与区块版本必须与本地值**完全**
    一致；
- 此阶段没有 major/minor 拆分：单一整数，任何
  线上格式破坏时递增。

## 二进制格式原则

- 二进制，协议数据绝不使用 JSON；
- 确定且规范：两个逻辑相等的对象产生
  **完全相同的字节**；
- 顺序解析，尽可能无分配（字节借用）；
- 所有网络输入都不可信：长度与计数在任何
  分配**之前**检查；解码绝不 panic。

### 字节序

- 所有整数都是 LEB128 变长整数（7 位一组，最低有效
  在前）：不存在本地字节序问题；
- IPv4/IPv6 地址与 32 字节标识符：裸字节数组，无
  长度前缀；
- 格式中没有定宽整数：唯一的整数
  编码规则。

### 变长整数（最小 LEB128）

每个字节携带 7 位数据；除最后一个字节外
每一字节的最高位都是 1。一个 `u64` 占 1 到 10 字节。

| 值 | 字节（十六进制） |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

编码必须是**最小的**：超长形式在
解码时被拒绝（`InvalidVarint`）。这是
一个值只有一种有效字节序列（规范编码）的
必要条件。拒绝示例：`80 00`（非最小）、`ff×9 02`（u64 溢出）、`ff×10`
（超长）。

### 字符串与字节

- `bytes`：变长整数长度 + 裸字节；
- `str`：同上，并强制 UTF-8 解码；
- 每个长度都有上限（见[限制](#limits)）并在
  读取前检查。

### 标识符

| 类型 | 线上表示 |
|---|---|
| `DomainId` | 32 裸字节 |
| `TldId` | 32 裸字节 |
| `OwnerId` | 32 裸字节 |
| `PublicKeyRef` | 32 裸字节 |
| `RecordHash` | 32 裸字节 |
| `BlockHash` | 32 裸字节 |
| `MerkleRoot` | 32 裸字节 |
| `DomainName` | `str`（≤ 253 字节，由 `scone-core` 重新验证） |

派生（`BLAKE3-256` + `SCONE-DOMAIN-V1` 前缀等）仍由
`scone-core` 定义（见 `/docs/en/general/naming.md`）。

## 交易

```text
Transaction = disc u8 || version u8 || payload
```

| 判别值 | 类型 |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

每个判别值都是任意的不透明常量（没有逻辑
顺序；`0x00` 永远无效）—— 完整规范性表格见
`/docs/en/technical/transactions.md`。

`version` 字节（当前值 `0x01`）紧跟判别值：
任何其他值都被显式拒绝
（`UnsupportedVersion(value)`）。大多数交易都是签名的
（Ed25519）—— 除了四种无签名的 pow-to-owner 形式
（`0x2A`、`0x57`、`0x84`、`0xC3`）：挖出的工作量证明
就是授权，其挑战绑定了受益者密钥与网络。完整规范、
签名载荷与验证规则见
**`/docs/en/technical/transactions.md`**（规范性）。

自 2026-09 线上格式变更起，**交易不再携带
`network` 字段**：`network_id`（`str` ≤ 16）注入
**签名载荷**（已签名形式）或 **PoW 挑战**
（pow-to-owner 形式）—— 为一个网络制作的交易绝不会
转移到另一个网络（见 `/docs/en/technical/blockchain.md`）。

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` 以明文携带**规范名称**，后跟
其 `DomainId` —— 一致性 `domain_id == DomainId(name)`
在解码时重新检查（重新计算，绝不信任）。`proof` 携带
注册 PoW（20 字节：nonce + 纪元 + 难度 —— 纪元
加盐 `epoch = height/144` 取自应用区块，在应用时
对照网络常量与区块纪元严格验证；不匹配则
类型化拒绝 `ProofEpochMismatch` —— M8b，见
`transactions.md`）。`public_key` 是签名者的 Ed25519 密钥（恰好
32 字节）；`owner` 必须是该密钥的 BLAKE3 派生
（重新计算，绝不信任）；`signature` 恰好 64 字节。

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

TLD 的声明（与域名注册表分离的注册表，M7）：
与域名一样，TLD **以明文携带**（M8b —— PoW
挑战由名称派生），`tld_id` 是该名称的
`SCONE-TLD-V1` 派生，在构造上与每个 `DomainId`
都不相交。状态规则（M8b）：TLD 空闲且非 ICANN
保留（`IcannTldReserved` —— ICANN 根 TLD 任何人都不可
声明）且 PoW 达到标签**按长度**的难度
（`pow_length` 曲线 —— 见 `blockchain.md`）；声明后
即封闭（仅限 assign）。

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

解码时重新检查的不变量：`sequence > 0` 且
`owner` == `public_key` 的派生。无 `timestamp`：顺序
经由 `sequence`。该交易只携带紧凑引用 —— 完整
DNS 内容留在 DHT（见下文）。

## DNS 记录（DHT 侧）

完整 DNS 记录绝不进入区块链：它们通过
DHT（`GetRecord`/`Record` 消息）交换，在链上只以
其 `RecordHash` 提交。

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

类型码是**真实的 DNS 码**（IANA）。TTL（秒、
变长整数）跟在类型码之后并覆盖整条记录：边界
`MIN_TTL = 1` .. `MAX_TTL = 2,592,000`（30 天；默认
`DEFAULT_TTL = 86,400`）。

| 码 | 类型 | 载荷 |
|---|---|---|
| 1 | A | 4 字节 |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`、`name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 字节 |
| 33 | SRV | `priority.v`、`weight.v`、`port.v`、`target(name)` |
| 257 | CAA | `flags u8`、`tag`（1–15 B）、`value`（`str` ≤ 4096） |
| 32768 | GEOROUTE | GeoRoute（见下文） |
| 其他 | Unknown | code.v + `bytes` ≤ 4096 |

示例：`A 192.0.2.1`（TTL 86,400）→ `01 80 a3 05 c0 00 02 01`
（类型、TTL 变长整数、地址）。

携带已知码的 `Unknown` 在编码时被拒绝（解码
绝不可能产生它）。

#### GEOROUTE（GeoDNS）

Scone 私有类型（非 IANA 范围）。一条地理路由规则：

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- world
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numeric
          | 0x03 region.v           -- region code
          | 0x04 asn.v              -- AS number
          | 0x05 prefix.addr[16] || prefix.len u8   -- IP prefix (normalized)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- binary IPv4
          | 0x01 octets[16]         -- binary IPv6
```

**规范规则**：同一作用域的端点以**排序后**
的形式编码（priority 升序，然后 weight 降序，然后
target 升序）；排序在编码时施加，未排序的线上
数据在解码时被拒绝（`NonCanonical`），与记录集
一致。完全重复的端点被拒绝。

**验证**：`priority > 0`、`weight ≤ 100`（0 = 不活跃
端点）、每作用域 ≤ 32 个端点（`GEO_MAX_ENDPOINTS`）。
本地表（`continent_of_country`）不认识的
国家/大洲码在线上级别**被接受**：解析器的
表可能比发布者的更新。TTL 取自
父级 `RecordData`（不变）。

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**规范规则**：记录集是无序集合。它以
**按每条记录各自编码的字典序排序**的形式
编码，严格递增（拒绝重复）。未排序的
线上数据在解码时被拒绝（`NonCanonical`）。原始
`Vec` 的顺序不被传输。

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

哈希只覆盖**记录内容**；所有者的签名在
同样的规范字节上单独验证。*（决策：承诺
不覆盖签名 —— 本文档的早期版本提到
"完整 SignedDnsRecord"；把哈希与
签名方案耦合没有任何收益。）*

## 区块

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`：协议版本（上述规则）；
- `height`：高度（创世 = 0）；
- `prev_hash`：前一区块的哈希（创世区块
  为全零）；
- `tx_root`：对交易列表的承诺（由
  `scone-blockchain` 定义的 Merkle 树，见
  `/docs/en/technical/blockchain.md`）；
- `timestamp`：顺序信息（Unix，秒）；
- `consensus`：出块者载荷（`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`，见 `blockchain.md`）—— 一个网络的创世
  携带原始 `network_id`，而分配创世（主网
  kickstarter）可携带 `ALLOC-V1` 清单（≤ 60 KiB）；受
  `MAX_CONSENSUS_LEN`（64 KiB）约束。

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

顺序解析；计数器在分配**之前**设上限；任何
截断都被拒绝。与记录集不同，交易顺序是
**有意义的**（共识顺序）并被精确保留。

`BlockHash`（规范头的哈希）与 `tx_root`由
`scone-blockchain` 在此处定义的编码上计算（精确
公式见 `/docs/en/technical/blockchain.md`）。

## P2P 消息

```text
Message = disc u8 || payload
```

传输层（分帧、libp2p……）不在范围内；一个传输帧
不应超过 `MAX_MESSAGE_LEN`（1 MiB）。

| 判别值 | 消息 | 方向 | 载荷 | 限制 | 期望的回复 |
|---|---|---|---|---|---|
| `0x01` | `Hello` | 双向 | `version.v` | — | `Hello` |
| `0x02` | `Ping` | 双向 | — | — | `Pong` |
| `0x03` | `Pong` | 双向 | — | — | — |
| `0x04` | `GetBlock` | 双向 | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | 双向 | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | 双向 | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | 双向 | `Transaction` | — | — |
| `0x08` | `GetRecord` | 双向 | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | 双向 | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | 双向 | `Checkpoint` | 签名者 ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | 双向 | — | — | 检查点窗口 |
| `0x0c` | `GetStatus` | 双向 | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | 双向 | 节点状态（网络、tip、高度、NAT、域名、内存池、对等节点、终局性、新鲜运营者） | — | — |
| `0x0e` | `LookupName` | 双向 | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | 双向 | 名称 + TLD 的链上状态 | — | — |
| `0x10` | `GetOwnedNames` | 双向 | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | 双向 | 所有者的存活域名 + TLD | — | — |
| `0x12` | `Error` | 回复 | 应用级错误消息（`str`） | — | — |

### Checkpoint（M5）

一个聚合检查点以如下形式上线：标签 `SCONE-CKPT-V1` ||
主体 `CheckpointData`（116 B，LE —— 与签名
前像的末尾相同：`epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32) recovery(4)`）|| `count.v` ||
每个签名者的 `pk[32] ‖ sig[64]`，**按 pk 升序排序、无重复**、
`count ≤ 64`（`MAX_CHECKPOINT_SIGNERS`）。严格解码：截断、错误
标签、非规范 pk、重复或未排序 = 拒绝（`None`），绝不
panic（损坏模糊测试已覆盖）。

`GetCheckpoints` 请求终局性窗口；回复是通过
一条或多条 `Checkpoint` 消息给出的
`checkpoint_window()` 窗口（至多 `CHECKPOINT_KEEP` 个
检查点，从旧到新）。

`TxId` 现由 `scone-blockchain` 定义
（`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`，见
`/docs/en/technical/blockchain.md`）。内存池与窗口化
防重放（`TxReplay`、`REPLAY_WINDOW_BLOCKS = 256`）已随
中继（M4）交付；只有 `GetTransaction` 仍被推迟（其
定义取决于余下的共识决策）。

## 区块链 / DHT 分工

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

链说明*谁*拥有以及*当前状态是什么*（sequence +
record_hash）。DHT 提供*内容*。一条 DHT 记录当且仅当
如下条件满足才被接受：

1. 其签名对链上注册的所有者有效；
2. 其 `sequence` 与链上一致（或更高）；
3. 在其规范编码上重新计算的 `RecordHash` 与链上的
   `record_hash` 匹配。

## 限制

| 常量 | 值 | 作用 |
|---|---|---|
| `MAX_NAME_LEN` | 253 | 名称长度 |
| `MAX_TLD_LEN` | 63 | TLD 长度 |
| `MAX_TXT_LEN` | 4096 | TXT 字符串 |
| `MAX_PROOF_LEN` | 256 | 注册证明 |
| `MAX_SIGNATURE_LEN` | 128 | 签名（Ed25519 = 64） |
| `MAX_RECORDS_PER_SET` | 256 | 每集合记录数 |
| `MAX_DHT_RECORD_LEN` | 64 KiB | 编码后的已签名 DNS 记录（DHT 载荷：线上、存储、gossip）—— 总边界在编码时和解码前都检查；比任何现实的记录集（SPF+DKIM+DMARC ≈ 2–5 KiB）高 15–30 倍，比传输上限低 16 倍。 |
| `GEO_MAX_ENDPOINTS` | 32 | 每个 GeoRoute 作用域的端点数 |
| `MAX_UNKNOWN_DATA` | 4096 | 未知类型的数据 |
| `MAX_TXS_PER_BLOCK` | 4096 | 每区块交易数 |
| `MAX_CONSENSUS_LEN` | 64 KiB | 头部的共识载荷（为 `ALLOC-V1` 创世清单加宽 —— 主网 kickstarter） |
| `MAX_BLOCKS_PER_REQUEST` | 128 | 每次 `GetBlocks` 的区块数 |
| `MAX_MESSAGE_LEN` | 1 MiB | 传输帧（建议值） |

任何超限 → `LimitExceeded`，在分配前检测：恶意
对等节点无法在解码时造成失衡的分配。

## 规范编码

基本规则：

```text
canonical object → canonical binary encoding → hash / signature
```

- 只用最小变长整数；
- 记录集严格递增排序；
- 逻辑相等的对象恰好有**一种**有效编码；
- 解码拒绝非规范形式（`NonCanonical`），因此
  同一对象不存在多种有效编码。

## 错误处理

`ProtocolError`（`scone-protocol::error`）：

| 变体 | 含义 |
|---|---|
| `Truncated` | 输入在一个值中间结束 |
| `InvalidVarint` | 非最小、溢出或超长的变长整数 |
| `IntegerOutOfRange` | 整数放不进目标类型 |
| `UnsupportedVersion` | 零或高于本地的版本 |
| `UnknownDiscriminant` | 未知的交易/消息判别值 |
| `LimitExceeded` | 长度或计数超限 |
| `InvalidUtf8` | 非 UTF-8 字符串 |
| `TrailingBytes` | 完整值之后还有剩余字节 |
| `Validation` | 违反 `scone-core` 不变量 |
| `NonCanonical` | 逐字段有效但非规范的编码 |

解码在网络数据上绝不 panic。

## 签名

- 方案：**Ed25519**（RFC 8032），经 `scone-crypto`
  （`ed25519-dalek`、`verify_strict` 验证 —— 拒绝
  可延展签名与弱密钥）；
- **链上交易**：每个 `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` 内嵌 `public_key`（32 B）+ `signature`
  （64 B），覆盖载荷
  `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`；
  完整规范见：`/docs/en/technical/transactions.md`；
- **DHT 记录**（`SignedDnsRecord`）：被签名的载荷是
  `DnsRecord` 的规范编码；验证将
  `record + owner + signature` 与区块链中注册的
  身份交叉核验；
- 横切规则：`owner` 与所有哈希/承诺都**重新
  计算**，绝不取自提供的值。

## 共识规则

已交付（描述见 `/docs/en/technical/blockchain.md`）：签名
出块者（`SCONE-BLOCK-V1` 载荷、检查点选举的
委员会）、聚合检查点终局性（`SCONE-CKPT-V1`、
终局性下限处的分叉选择）、按名称长度的注册
PoW。余下主题：细粒度反垃圾费用、所有者密钥
轮换。

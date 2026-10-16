# 已签名交易（v1 格式）

已签名交易格式的**规范性**参考文档。
它规定了字段、精确的线上顺序、大小、签名
载荷与验证规则，以便无需阅读 Rust 即可实现
独立实现。参考实现为
[`scone-protocol`](../../../crates/scone-protocol)（编码）与
[`scone-blockchain`](../../../crates/scone-blockchain)（验证）。

`PROTOCOL_VERSION` = **1**。只有一套格式：任何
非 `0x01` 的版本字节（交易中）或与本地版本
不同（区块中）都会以 `UnsupportedVersion` 拒绝。

## 概览

```text
Transaction = disc u8 || version u8 || payload
```

| 字段 | 大小 | 作用 |
|---|---|---|
| `disc` | 1 B | 交易类型（见下表） |
| `version` | 1 B | **恰好 `0x01`**（交易格式版本） |

**交易在线上不再携带 `network` 字段**
（2026-09 变更）：`network_id`（`str` ≤ 16、ASCII
`[a-z0-9-]` —— 例如 `scone-testnet`，13 字节）**只存在于
签名载荷中** —— 由签名者作为第一个字段注入其中
（见[签名载荷](#签名载荷)）—— 并在 PoW 摘要中保持
按网络区分。因此，为一个网络**签名**的交易
绝不能在另一个网络上验证：其签名对链
重新计算的载荷（注入链自己的网络）验证失败
（`InvalidSignature`）（见
`/docs/en/technical/blockchain.md` —— 网络与创世）。

### 判别值（规范性表格）

每个判别值都是**固定的任意不透明常量**：
这些值被刻意选得互不相同，使线上格式**绝不**
暗示类型之间的逻辑顺序（没有 0x01/0x02/0x03……
序列）。`0x00` **永远不是**有效类型（典型的损坏
字节）。未来的类型将取任何未被占用的空闲值，
并一次性记录在下表中。

| 判别值 | 类型 | 作用 |
|---|---|---|
| `0x21` | `RegisterDomain` | 域名声明 |
| `0x52` | `UpdateDomain` | 域名 DNS 数据的新版本 |
| `0x93` | `RegisterTld` | 顶级域声明（M7） |
| `0x47` | `TransferTld` | 将 TLD 转移给新所有者（M8a） |
| `0x6B` | `RevokeTld` | 放弃一个 TLD（M8a） |
| `0xB8` | `SetTldOpen` | 向自助注册**不可逆地**开放一个 TLD（单向、所有者选项；`open=false` 是类型化拒绝） |
| `0xD4` | `AssignDomain` | 由 TLD 所有者直接分配域名（M8a） |
| `0x3C` | `RenewDomain` | 延长域名的注册期（M8a；因所有者选项而携带规范名称 —— 开放 TLD 跟随到期时间） |
| `0x8F` | `RenewTld` | 私有 TLD 3 年期限的续期（与创建等量的 PoW） |
| `0x15` | `SlashTx` | 锚节点歧义行为的证据 + 封禁（M9） |
| `0xE2` | `TransferDomain` | 将域名转移给新所有者（M8c） |
| `0x71` | `Heartbeat` | 运营者的存活证明（服务证明 RFC） |
| `0xA9` | `SetTldOperator` | 私有 TLD 运营的委托（服务证明 RFC） |
| `0x5E` | `SetDomainOperator` | 域名运营的委托（服务证明 RFC） |

任何非 `0x01` 的版本字节都被显式拒绝
（`UnsupportedVersion(收到的值)`）：未知或过时的格式
绝不会被意外解析。

## SLASH_TX（0x15，M9）

某锚节点在**同一纪元签署两个冲突检查点**
（歧义行为）的链上证据。任何人都可以提交 —— 证据
是密码学且自含的：

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` 检查结构：同一纪元、不同的
  `signing_hash`、两个不同的签名、报告者对
  规范载荷的签名；
- 状态规则（记入日志、重组时可逆）：被指控者的
  两份签名都必须对同一纪元的两个不同签名哈希
  验证通过；被指控者必须在资格池中
  （否则 `SlashOffenderNotInPool`）；然后**封禁** —— 该密钥
  被持久地从 PoS 资格池中移除（状态中的黑名单）；
- 最小制裁：被指控者的域名/TLD 不被没收
  （已记录的 M9 决策）；
- 同一检查点两次 = 非歧义行为（拒绝）；不同
  纪元 = 拒绝；伪造签名 = 拒绝。

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `name` | `str` ≤ 253 | 所声明域名的规范名称，**以明文携带**（见下文） |
| `domain_id` | 32 裸字节 | **必须**等于 `DomainId(name)`（重新计算，绝不信任） |
| `owner` | 32 裸字节 | **必须**是 `public_key` 的派生（见下文） |
| `timestamp` | 变长整数 | 顺序信息（Unix，秒） |
| `proof` | `bytes` ≤ 256 | 注册 PoW 载荷（见[下文](#注册工作量证明-pow)） |
| `public_key` | 32 裸字节 | 签名者的 Ed25519 公钥（RFC 8032） |
| `signature` | **恰好 64 字节** | 签名载荷的 Ed25519 签名 |

### 名称由交易携带

`RegisterDomain` 以明文传输规范名称，后跟
其 `DomainId`。一致性 `domain_id == DomainId(name)`
在编码、解码与验证时重新检查（纵深防御）：
id 与所携带名称不匹配的
`RegisterDomain` 无效。`DomainId` 仍是共识身份（链与
DHT 键）；所携带的名称使链自描述，并让
签名覆盖整个声明。

## REGISTER_TLD (0x93)

顶级域的声明（TLD 注册表，M7）：

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `tld` | `str` ≤ 63 | TLD 的规范名称**以明文携带**（M8b：PoW 挑战由名称派生，正如 `RegisterDomain` 携带自己的名称） |
| `tld_id` | 32 裸字节 | TLD 的身份：`BLAKE3-256("SCONE-TLD-V1" || tld)` —— **必须**是 `tld` 的派生（重新计算） |
| `owner` | 32 裸字节 | `public_key` 的派生 |
| `timestamp` | 变长整数 | 顺序信息（Unix，秒） |
| `proof` | `bytes` ≤ 256 | 注册 PoW 载荷（见[下文](#注册工作量证明-pow)） |
| `open` | `0x00\|0x01` | **开放**诞生：TLD 直接落入 `TldState::Open`（无主、自助 + PoW、按 `open_tld_pow_difficulty` 费率的一次 PoW —— 比私有便宜：公共资源、无 PoS 席位也无排他性）；**私有**诞生：历史行为（所有者、PoS 席位、`tld_pow_difficulty`） |
| `public_key` | 32 裸字节 | 签名者的 Ed25519 公钥 |
| `signature` | **恰好 64 字节** | 签名载荷的 Ed25519 签名 |

`TldId` 空间与 `DomainId` 空间**不相交**（不同的
派生前缀 —— 见 `/docs/en/general/naming.md`）：注册
TLD `uip`（对 `*.uip` 命名空间的声明）永远
不可能侵占域名的身份，反之亦然。应用时的
状态规则：TLD 必须空闲（与域名注册表分离的
注册表）。

## M8a 家族 —— 线上格式

五种类型补全 TLD 注册表与域名生命周期。
线上格式自 M8a 起为规范性；**完整状态规则自
M8b 起**（见 `/docs/en/technical/blockchain.md`）：需要
TLD 所有者、凡自助声明处必有 PoW、域名侧的
过期与宽限。没有任何载荷在线上携带 `network`
字段（它只存在于签名载荷中）。

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `tld_id` | 32 裸字节 | 被转移的 TLD |
| `owner` | 32 裸字节 | `public_key` 的派生（当前所有者 / 签名者） |
| `new_owner` | 32 裸字节 | 接收方身份（不透明：任何值在语法上都有效；与可花费密钥的绑定是 M8b 状态规则） |
| `public_key` | 32 裸字节 | 签名者的 Ed25519 公钥 |
| `signature` | **恰好 64 字节** | 签名载荷的 Ed25519 签名 |

无时间戳或序列号：一个 TLD 同一时刻只有一个
所有者；第一个被应用的转移获胜（仅凭链上
顺序）。

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

自愿放弃一个 TLD（之后可通过一次新的
`RegisterTld` 重新声明）。由当前所有者签名；
刻意不带时间戳或序列号 —— 放弃是针对当前
状态的一次性动作。

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `open` | 1 字节 | **严格为 `0x00` 或 `0x01`**（规范形式；任何其他值 → 解码错误，绝不强制转换） |
| `open_proof` | `proof`（长度变长整数 + 字节） | **被签名的**字段；当 `open = 0x01` 时强制存在并被验证（开放 PoW、网络的 `open_tld_pow_difficulty`、挑战 `SCONE-TLD-OPEN-V1 ‖ tld_id`）；当 `open = 0x00` 时为空（关闭免费） |

**封闭的** TLD = 仅限 assign（域名只能由
TLD 所有者通过 `AssignDomain` 创建）；**开放的**
TLD = 任何人都可以做注册 PoW 并声明一个空闲
域名（`RegisterDomain`）。**开放**需要一份额外的
工作量证明（比 TLD 声明轻：主网上约 3 天
对比约 1 周），由所有者在 `SetTldOpen` 中一次性
支付 —— 证明在已签名的 `open_proof` 字段中
传输，签名后无法被替换。

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

`RegisterDomain` 的镜像：规范名称以明文
携带，且 `domain_id` 必须等于 `DomainId(name)`
（在解码、编码与验证时重新计算）。由
**该名称所属 TLD 的所有者**签名（在应用时
检查，M8b）；`assignee` 成为域名的第一
所有者（不透明，如同 `new_owner`）。

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `name` | `str` ≤ 253 | **以明文携带**的规范名称（所有者选项：续期推迟其下开放 TLD 的死亡，链必须知道名称）；`domain_id` 必须等于 `DomainId(name)`（重新计算） |
| `valid_until` | 变长整数 | 新的注册到期时间（Unix，秒） |
| `proof` | 有界 | **续期授权**（见下文） |

由域名当前所有者签名；`valid_until` 是否
真的延长了到期时间（且至多延长一个续期
期限）是状态规则（M8b）。

#### 续期授权 —— 依 TLD 而定的 PoW 或 PoS

续期的成本取决于其下 TLD 的性质
（在应用时、区块状态上求值）：

- **开放 TLD**：续期花费**与创建相同的 PoW** ——
  相同的挑战（`SCONE-DOMAIN-V1 ‖ canonical name`）、
  相同的按标签长度与按网络的难度、相同的纪元
  有效窗口。域名绝不免费"被收割"：生命的
  每一段期限都以工作量支付。
- **私有 TLD**：命名空间是 TLD 所有者的质押
  （PoS）。交易必须**由 TLD 所有者（或其委托的
  运营者）签名** —— 签名即授权，`proof`
  字段必须为空（20 个零字节）。TLD 所有者
  按自己的规则续期其命名空间下的域名。

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `tld` | `str` ≤ 63 | 规范 TLD 名称；`tld_id` 必须等于 `TldId(tld)`（重新计算） |
| `valid_until` | 变长整数 | 必须恰好为 `now + 3 年`（固定期限 —— 在应用时重新推导，任何其他值都被拒绝） |
| `proof` | 有界 | 续期 PoW（见下文） |

由 TLD 当前的**私有**所有者签名。应用规则：

- 只适用于私有 TLD（开放 TLD 通过其
  域名存活 —— 类型化拒绝 `TldIsOpen`）；
- 在期限内以及到期后 **60 天宽限期**内
  有效；再往后 TLD 已死（GC），拒绝为
  `TldExpiredFrozen`；
- **与创建等量的 PoW**：挑战
  `SCONE-TLD-V1 ‖ name ‖ current_expiry(BE64)`
  （防重用后缀把每个 PoW 绑定到唯一一个
  期限），与初始声明相同的按长度难度；
- 宽限期内续期不被冻结 —— 它是过期的
  所有者仅存的可用管理操作。

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `domain_id` | 32 裸字节 | 目标域名 |
| `owner` | 32 裸字节 | `public_key` 的派生 |
| `sequence` | 变长整数 | **> 0**，应用时必须等于 `current + 1` |
| `record_hash` | 32 裸字节 | 对 DHT 记录集的承诺 |
| `public_key` | 32 裸字节 | 签名者的 Ed25519 公钥 |
| `signature` | **恰好 64 字节** | 签名载荷的 Ed25519 签名 |

注意：`UpdateDomain` **不携带** `timestamp` —— 顺序
经由 `sequence`。它也不携带名称：域名已经
注册，id 足矣。

### 严格边界

- `public_key`：**恰好 32 裸字节**，无长度前缀。
  无法解码为曲线上点的 pk → 解码时
  `InvalidTransactionKey` 错误。
- `signature`：**恰好 64 裸字节**，无长度前缀、
  无余量。任何其他大小都是硬解码错误
  （拒绝截断）。
- `proof`：长度变长整数 + 字节，上限 256
  （`MAX_PROOF_LEN`），分配前检查。
- `name`：长度变长整数 + UTF-8 字节，上限 253
  （`MAX_NAME_LEN`），由 `scone-core` 命名规则重新
  验证。

## 所有者派生

硬性规则：**所有者永远被重新计算，绝不信任**。

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

`owner` 与其自身 `public_key` 的派生不匹配的
交易无效（核心侧 `InvalidOwner`、
链侧 `OwnerKeyMismatch`），在编码、解码
**以及**链验证时拒绝（纵深防御）。

## 签名载荷

被签名的载荷是交易（**不含签名字段**）的规范
编码，加以域分离标签为前缀 —— 并在头部
**注入目标链的 `network_id`**：

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

其中 `canonical_encode(tx_without_signature, network)` 遵循
上述线上结构，包含判别值与版本字节，在
`signature` 之前停止，**并以 `network` 作为第一个
字段** —— 这是网络出现的唯一位置。链用
它自己的网络重新计算载荷：为另一个网络
签名的交易签名验证失败（`InvalidSignature`）；
这就是防跨链重放机制。

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] public_key[32]
```

属性：

- 确定且纯粹：同一交易 ⇒ 处处同一载荷；
- ASCII 前缀 `SCONE-TX-SIG-V1`（15 字节、
  十六进制 `53434f4e452d54582d5349472d5631`）防止
  签名在协议的另一个哈希域中被重用；
- 除 `signature` 外的所有字段都被覆盖 ——
  **包括 `RegisterDomain` 携带的名称**：修改
  其中任何一个都会使签名无效；
- 被注入的 `network_id` 也被覆盖：同一笔未
  签名交易为两个网络签名会产生两个不同的
  载荷（因此两个不同的签名）。

## 注册工作量证明（PoW）

`RegisterDomain` 与 `RegisterTld` 的 `proof` 字段携带
注册工作量证明（`scone-core::pow` 模块）。

### 摘要（纪元加盐、防预挖）

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocks)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` 是规范的 `network_id`（M8b）：为一个
网络挖出的证明在另一个网络上不是证明。

`SCONE-POW-V1-ARGON2` 还固定了按网络的 Argon2id
参数（`pow_params`：主网 m = 64 MiB、t = 2、
p = 1；测试网/开发网 m = 8 KiB、t = 1、p = 1）：
Argon2id 核心（RFC 9106）使每次哈希的成本由
内存主导（抗 GPU/ASIC），且在一组参数下
挖出的证明绝不在另一组下验证通过。

`challenge` 是所声明对象的派生输入 ——
TLD 为 `"SCONE-TLD-V1" || tld`，域名为
`"SCONE-DOMAIN-V1" || name`：正是为 `TldId` /
`DomainId` 哈希的字节，因此证明绝不能跨
命名空间或跨对象种类重放。`epoch_le64` 与
`nonce_le64` 是纪元与 nonce 的小端 8 字节
编码。

当 `digest` 至少有 `difficulty` 个前导零
比特时证明有效。

#### 纪元加盐 —— 防预挖

无盐的摘要既不绑定时间也不绑定高度：任何
人都可以今天为成千上万个名称预挖证明，
然后在主网上线时一拥而上注册（抢注）——
证明自含，对最先到来的任何区块都有效。
摘要以**应用该交易的区块的纪元**加盐：

- `epoch = block_height / POW_WINDOW`、
  `POW_WINDOW = 144` 个区块（65 秒/区块约
  2.6 小时）：给诚实用户的宽限窗口（挖矿、
  然后在窗口内提交），超出则必须重挖；
- 证明在线上声明其纪元；应用时，声明的
  纪元必须落在应用区块的**有效窗口**内 ——
  只向后容忍（迟交可被吸收，"为未来挖矿"
  不可能），窗口大小由
  `epoch_validity_window(difficulty)` = `1 << max(0,
  difficulty − 16)` 个纪元决定（加宽的窗口：
  主网高难度要挖数天到数周，长于一个纪元；
  囤积毫无收益，因为窗口 ≈ 挖矿成本 ——
  突发中的每个证明都要在窗口内支付其
  完整的 `2^difficulty` 次哈希）。在难度 ≤ 16 位
  （测试网、长名称）时严格相等规则
  不变；
- 内存池准入（尽力而为）：未来的区块高度
  未知 —— 中继对照 `当前高度 + 1` 的纪元
  验证；严格的应用时验证仍是权威；
- 确定性重放：证明携带其纪元且窗口是
  （难度、区块纪元）的纯函数 —— 历史区块
  在任何节点上都以相同方式重新验证。

主网前的破坏性变更：旧的紧凑格式（12 字节）
被拒绝，无向后兼容。

### 线上载荷（固定 20 字节）

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

难度与纪元随证明一起传输，但它们是
**生产者声明的、因此绝不被信任**：验证者
将难度与注册种类的**协议常量**比较（声明
难度与常量不同 —— 更低*或*更高 —— 的
载荷被拒绝，`InvalidProof`），将纪元与应用
区块的纪元比较（不匹配则
`ProofEpochMismatch`）。然后按常量难度重新
验证 nonce、从头重算摘要。

### 难度（**按长度**的曲线，M8b + 实测校准）

难度不是一个扁平参数：它是
**所创建标签长度**的确定性函数
（`scone-core::pow_length` —— `RegisterTld` 取
TLD，`RegisterDomain` 取最左标签），按网络
区分：

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| 网络 | 私有 TLD（apex/floor） | 生而开放的 TLD / 域名（apex/floor） |
|---|---|---|
| testnet | 8 位 / 2 位 | 6 位 / 2 位 |
| mainnet | 42 位 / 30 位 | 40 位 / 30 位 |
| devnet | 2 位 / 1 位 | 1 位 / 1 位 |

**生而开放**的 TLD（`open = 0x01` 的
`RegisterTld`）支付"开放"曲线（apex − 2、
如同 `domain_difficulty`）；**开放一个既有的
私有 TLD**（`SetTldOpen`）保持扁平难度
（`open_tld_pow_difficulty`：测试网 6 位 /
主网 22 位 / 开发网 2 位 —— 该交易
只携带 `tld_id`，名称在创建时已按其长度
付费）。主网基准（私有 TLD，约 6 M 哈希/秒）：
1 字符 = 42 位（≥ 7 天）、5 字符 = 40 位
（约 2 天）、19 字符 = 35 位（约 1.6 小时）、
29+ 字符 = 30 位保底（约 3 分钟）。曲线位于
`scone-core::pow_length`（常量 `MAINNET_APEX = 42`、
`MAINNET_FLOOR = 30`、`TESTNET_APEX = 8`、
`TESTNET_FLOOR = 2`、`DEVNET_APEX = 2`、
`DEVNET_FLOOR = 1`）；扁平参数
（`tld_pow_difficulty` 等）仍在
`scone-core::network::NetworkParams`。修改一个值
= **该网络**的一次共识变更。不变量：开发网是
完整的第三个 `NetworkParams`，且测试网
与其开发网前的常量（Argon2id 参数、曲线、
窗口）保持严格一致 —— 除代码本身外
开发网不与其他网络共享任何常量。

扁平上限的实测校准（2026-10，`cargo run
--release -p scone-core --example pow_bench` —— 参照
x86-64 上单线程吞吐约 7.7 M H/s）：

| 种类（主网） | 难度 | 平均实际时间 |
|---|---|---|
| `RegisterTld` | 24 位 | ~7 s |
| `RegisterDomain` | 20 位 | ~54 ms |

按 ~65 秒/区块：TLD 声明花费约 0.1 个区块的
单核 CPU，域名约 1 毫秒区块时间 —— 对用户
是象征性的，规模化工业化则代价高昂
（防抢注）。若硬件目标下降（移动 ARM：
慢约 4–8 倍）需重新校准。

状态分支（M8b，已交付）：`RegisterTld` 总是
需要其 PoW；`RegisterDomain` 在**开放** TLD 上
需要 PoW（封闭路径仅限 assign ——
`AssignDomain`、无 PoW、由 TLD 所有者签名）。

## 验证规则

一个交易当且仅当如下条件满足才有效：

1. **严格解码**：遵守上述结构与边界（已知
   判别值、版本 `0x01`、pk 32 B、签名 64 B、
   核心不变量在解码时重新检查）；
2. **所有者/公钥绑定**：`owner ==
   OwnerId::from(public_key)`（重新计算）；
3. **名称/id 一致性**（RegisterDomain）：
   `domain_id == DomainId(name)`（重新计算）；
4. **签名**：以链的网络**从头重新计算**的签名
   载荷对 `signature` 做 Ed25519 `verify_strict`
   （绝不取自对等节点或按原样取自交易）——
   为另一个网络签名的交易在此失败
   （`InvalidSignature`、防跨链重放）；
5. **状态规则**（在区块中应用时 —— M8b，见
   `/docs/en/technical/blockchain.md`）：
   RegisterTld → TLD 空闲 + 非 ICANN + PoW
   （按长度难度）；RegisterDomain → TLD 已注册
   **且开放** + 域名空闲（含宽限期）+ PoW
   （按长度）；UpdateDomain → 域名已存在、
   当前 `owner`、`sequence == current + 1`；
   TransferTld / RevokeTld / SetTldOpen /
   AssignDomain → TLD 存在**且私有**、签名者 =
   该 TLD 的所有者；TransferDomain → 域名
   存在、所有者；RenewDomain → 域名存在、
   严格延长 ≤ 3 年、PoW 授权（开放 TLD：与
   创建相同的 PoW、**绑定到当前到期时间** ——
   一个不同于注册以及此前每次续期的挑战）
   或 PoS（私有 TLD：TLD 所有者 —— 或其
   有效运营者 —— 的签名、空 `proof`）。

验证使用 `verify_strict`（拒绝可延展签名
与弱/非规范密钥），这是不可信数据的
必然要求。

## `push_block` 中的操作顺序

密码学验证（2 到 4）在状态应用**之前**
对每笔交易进行，基于每个区块的原子工作
状态；任何错误都使链保持严格不变。

## 区块组装

`BlockBuilder`（`scone-blockchain`）从交易队列
组装区块：在有序列表上重新计算 `tx_root`、
头部链接到父哈希（`prev_hash`）、`version` =
`PROTOCOL_VERSION`。共识载荷（已签名的出块者
载荷）与时间戳由调用者提供；现有的
`Consensus` trait 与验证对称使用。

## 十六进制示例（真实、可复现的向量 —— M8b）

### RegisterDomain `example.uip`、时间戳 42、测试网

已签名载荷（固定向量；历史上由已移除的
离线工具生成 —— 这些值仍是有效向量）。
`network` 出现在这里是因为这是签名载荷
（它唯一出现的地方）；被签名的线上交易
本身不含网络：

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
21                                    disc REGISTER_DOMAIN (0x21)
01                                    format version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, length 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (fixed build key)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + start of sig
```

### RegisterTld `uip`、时间戳 42

已签名载荷（固定向量）—— `TldId` 是
`/docs/en/general/naming.md` 中固定的派生
向量：

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, 3-char label)
00                                    open (0x00 = private birth)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

（完整值曾可通过已移除的离线工具复现；
下列向量仍被固定且有效。PoW 按网络
难度自动挖取 —— nonce 每次运行都不同；
只有 network/name/id/owner/pk 是稳定的。自
2026-09 线上变更起，已签名交易的十六进制
不再包含字节
`0d 73636f6e652d746573746e6574`：网络只在
待签名载荷中传输。）

## 十六进制如今在哪里

离线命令 `tx build|sign|verify` 已被移除
（cli-simplify-v1）。已签名十六进制格式本身未变，
仍是 **web/API 流程**的线上格式：注册
web 应用构建、签名（浏览器内 keystore）并
以规范十六进制通过 API 提交交易。在 CLI
侧，用户路径是一条命令的形式：

```bash
scone domain register example.uip        # mines, signs, submits
scone renew example.uip                  # renewal proof, signs, submits
scone domains                            # what you own
```

在所有代码路径中，所有者与密钥都从签名
身份**重新计算**，绝不取自任何输入。

## 版本历史

| 版本 | 内容 | 状态 |
|---|---|---|
| 1 | 内嵌 `public_key` + `signature`、载荷 `SCONE-TX-SIG-V1`、版本字节 `0x01`、`RegisterDomain` 携带名称、`RegisterTld` (0x93)、M8a 家族：`TransferTld` (0x47)、`RevokeTld` (0x6B)、`SetTldOpen` (0xB8)、`AssignDomain` (0xD4)、`RenewDomain` (0x3C) —— 规范线上格式；M8b：`network` 字段签入每个载荷的头部、`RegisterTld` 携带名称、完整状态规则、按网络 PoW 难度 | 已被取代 |
| 1（2026-09，线上无 network） | **从所有交易的线上格式中移除 `network` 字段**：network_id 只存在于签名载荷中（`signing_payload(tx, network)` 将其注入头部）；网络分离依赖签名（外部载荷 `InvalidSignature`，`WrongNetwork` 错误消失）；`TX_FORMAT_VERSION` 保持 `0x01`（可重启的测试网、无共存）；**所有 TxId 改变**（TxId 是已签名线上编码的哈希，减少 14–15 字节）；PoW 摘要、创世与按网络的 data-dir 不变 | 已被取代 |
| 1（2026-10，PoW 纪元加盐） | **防预挖 PoW 摘要**：盐 `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + 基于 `nonce_le` 的 Argon2id 核心，`epoch = block_height / 144` 取自应用该交易的区块；证明线上载荷 12 → **20 字节**（`nonce_le ‖ epoch_le ‖ difficulty_le`），声明的纪元对照应用区块的纪元有效窗口验证（类型化拒绝 `ProofEpochMismatch`）、内存池准入时尽力的 `(height+1)/144`、离线路径上的 `--epoch` CLI 标志；主网前的破坏性变更：旧的 12 字节载荷被拒绝，**所有 TxId 改变**（`proof` 字段增长 8 字节） | 当前 |

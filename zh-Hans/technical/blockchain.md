# 区块链

区块链层的参考文档：身份、哈希、创世、
规范状态、交易排序与区块验证。
参考实现是 crate
[`scone-blockchain`](../../../crates/scone-blockchain)，其下是
`scone-protocol`（规范编码）、`scone-core`（业务规则）与
`scone-crypto`（BLAKE3）。

当前范围：具备真实共识（签名出块者、
检查点、终局性）的区块链逻辑、KV 后端之上的
**内存状态**（具体存储位于 `scone-storage`）—— 见
[推迟到共识](#推迟到共识)。

## 身份与哈希域分离

四种身份、四种角色、四个不同的 BLAKE3 前缀 —— 一个
前缀绝不在两种用途间复用：

| 类型 | 角色 | 前缀 | 公式 |
|---|---|---|---|
| `DomainId` | 域名身份 | `SCONE-DOMAIN-V1` | 见 `/docs/en/general/naming.md` |
| `RecordHash` | DNS 内容身份 | `SCONE-RECORD-V1` | 见 `/docs/en/technical/protocol.md` |
| `TxId` | 交易身份 | `SCONE-TX-V1` | 见下文 |
| `BlockHash` | 区块身份 | `SCONE-BLOCK-V1` | 见下文 |

Merkle 树使用自己的前缀 `SCONE-MERKLE-V1`。这些前缀
之外还有 `scone-core` 的
（`SCONE-PUBKEY-V1`、`SCONE-OWNER-V1`）。

完整依赖链：

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash of the next block
    ↓
canonical chain
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- 确定性：同一交易（同一规范编码）⇒ 同一
  `TxId`，在所有节点上，永远如此；
- **按内容**标识：任何被修改的字段都改变
  `TxId`（域名、所有者、时间戳、证明、序列号、
  record_hash）；
- 没有任何本地时间戳或非规范值进入计算；
- 不依赖存储或网络；
- 该身份不保证应用上的唯一性：重放一笔
  完全相同的交易会因状态规则失败（双重
  `RegisterDomain`、序列号重放），而不是因
  `TxId` 失败。

API：`scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`。

## Merkle 树（tx_root）

对一个区块交易**有序列表**的承诺：

```text
0 transactions  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
internal node   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
odd level       the last hash is duplicated to complete the pair
```

规则：

- **绝不排序**：区块中交易的顺序是
  有意义的；包含相同交易但顺序不同的
  两个区块拥有不同的根；
- `0x00` / `0x01` 标签防止叶节点与
  内部节点之间的任何混淆（第二原像）；
- `SCONE-MERKLE-V1` 前缀将该树与其他哈希
  域隔离（`TxId` 不能被解释为节点）；
- 自研实现（约 30 行）、确定且经过测试；没有
  外部 Merkle 库。

API：`merkle_root(&[TxId]) -> MerkleRoot` 与
`tx_root(&[Transaction]) -> Result<MerkleRoot>`。

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

哈希**只**覆盖规范头部：交易已由
`tx_root` 承诺，因此不再被二次哈希：

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

修改一笔交易、它们的顺序、高度、时间戳、
`prev_hash` 或共识载荷都会改变 `BlockHash`。

API：`block_hash(&BlockHeader) -> Result<BlockHash>`。

## 网络与创世（M8b）

Scone 定义三个互不相交的网络，以规范的
`network_id`（1..=16 个 ASCII 字节 `[a-z0-9-]`）标识：

| 网络 | `network_id` | 私有 TLD apex | 开放 TLD apex | 域名 apex | TLD 保底 | 域名保底 |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 位 | 7 位 | 6 位 | 2 位 | 1 位 |
| mainnet | `scone-mainnet` | 28 位 | 27 位 | 26 位 | 25 位 | 22 位 |
| devnet | `scone-devnet` | 2 位 | 1 位 | 1 位 | 1 位 | 1 位 |

参数位于 `scone-core::network::NetworkParams`
（命名实例 `TESTNET` / `MAINNET` / `DEVNET`，经
`NetworkParams::by_name` 查找）。修改一个难度 = **该
网络**的一次共识变更。

**结构性不变量**：开发网是完整的第三个
`NetworkParams` —— 绝不是测试网标志的子集。测试网
与其加入开发网之前的状态**严格一致**
（相同的 Argon2id 常量、相同的曲线、相同的纪元
窗口）；测试套件固定了这一不变量。

**按长度变化的难度**（`scone-core::pow_length`）：创建
的难度取决于**所创建标签的长度**
（`RegisterTld` 取 TLD —— 私有或新近开放 ——，
`RegisterDomain` 取最左标签）。公式为：

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), clamped at the floor
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), clamped at the floor
```

两条不同的斜率：TLD 为 `log₂(1.22) ≈ 0.287` 位/字符
（每多一个字符使成本除以约 1.22），域名为
`log₂(1.40) ≈ 0.480` 位/字符 —— 域名衰减
更快：它们的价值在于承载它们的 TLD。生而
开放的 TLD 位于私有 apex − 1；域名位于私有
apex − 2（主网：私有 TLD 25、开放 TLD 24、域名
23）。主网基准（内存困难的 Argon2id、单线程
约 11.4 H/s）：私有 TLD 1 字符 = 28 位 ≈ 30 天、
9+ 字符 = 25 位保底 ≈ 3.8 天；开放 TLD 1 字符 =
27 位 ≈ 15 天；域名 1 字符 = 26 位 ≈ 3.8 天、
9+ 字符 = 22 位保底 ≈ 11.5 小时。**8 核参照机**
（2026-10 重新校准 —— CLI 多核挖矿），每核
约 11.4 H/s。**开放**一个既有私有 TLD
（`SetTldOpen`）保持扁平难度
（`open_tld_pow_difficulty`：测试网 6 位 /
主网 22 位）：该交易只携带 `tld_id`，名称在
创建时已按其长度付费。纯函数（确定性：
两个节点对同一名称算出同一难度）。

创世是**按网络**的协议常量，仅凭常量即可
复现（无随机性、无本地时钟、无网络）：

| 字段 | 值 |
|---|---|
| `version` | `PROTOCOL_VERSION`（`1`） |
| `height` | `0` |
| `prev_hash` | 32 个零字节（无前一区块） |
| `tx_root` | 空列表的 Merkle 根 |
| `timestamp` | `GENESIS_TIMESTAMP = 0`（纯结构性） |
| `consensus` | `network_id` 的规范字节 |
| transactions | 空 |

**分配创世**变体（主网 kickstarter，M-next）：
`consensus` 字段携带 `network_id ‖ ALLOC-V1 ‖ count.v ‖
entries`（预支付的域名/TLD、按名称规范排序、
清单 ≤ 60 KiB、≤ 2048 项分配）—— 正是这个加宽的
格式证明了 `MAX_CONSENSUS_LEN` = 64 KiB 的合理性
（见 `scone-blockchain::genesis_alloc`）。

共识载荷中的 `network_id` 使创世哈希 —— 从而
整条区块链 —— **跨网络不相交**：测试网
区块永远无法接到主网链上
（`UnknownParent`），不重新签名一切就无从
重放。

API：`genesis_of(&NetworkParams) -> Block`、
`genesis_hash_of(NetworkId) -> BlockHash`；别名
`genesis()` / `genesis_hash()` 指测试网（开发的
默认网络）。

### 开发网 —— 可重置的本地网络

开发网（`scone-devnet`）是第三个网络，专用于
快速本地迭代：

- **自己的创世**：共识载荷中的 `scone-devnet`
  网络 id 产生一个与测试网和主网都不同的
  创世哈希 —— 任何创世、区块、签名或 PoW
  证明都不跨越边界（重放不可能）。
- **近零 PoW**：曲线固定在 0..=2 位区间
  （`DEVNET_APEX = 2`、`DEVNET_FLOOR = 1` —— 长度
  衰减仍适用）：TLD 或域名声明毫秒级挖出，
  走的与其他网络完全相同的代码路径。
- **快速 PoS 共识**：与测试网相同的参数
  （委员会 4 人、10 秒纪元），本地网格上
  快速终局。
- **重置**：链位于自己的数据目录
  （默认 `~/.scone/devnet/`）。**清空该目录并
  重启中继即重置链**（全新创世、干净的
  存储）—— 无需额外工具。绝不要把开发网
  数据目录指向另一个网络的数据目录：持有
  另一条链的非空存储会被拒绝
  （`Corrupted`）。

## 规范状态

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- enum (owner-option): the open/private invariant is
               -- structural; the death deadline lives outside the entry
               -- (tld_floors table + journaled death schedule)
ChainState   = NetworkParams                  (bound network, M8b)
               + DomainId -> DomainState      (direct access, in memory)
               + TldId -> TldState            (TLD registry, M7b)
               + DomainId -> (expired_at, OwnerId)   (grace windows, M8b)
```

* `registered_at` / `valid_until`（M8b）：注册到期，
  对照**父区块的时间戳**求值（确定性、
  承诺在头部 —— 绝不是本地时钟）；
* `open`：私有 TLD 可以**生而开放**（`open = 0x01`
  的 `RegisterTld` —— 无主、自助）或在声明时
  **封闭**（封闭 = 仅限 assign），之后经
  `SetTldOpen`（带 PoW 的自助注册）变得开放 ——
  **不可逆**；

- 按 `DomainId`（32 字节）直接访问：绝不用
  `String` 作键、绝不全量扫描 —— 这是面向
  数千亿域名的必要条件；
- 只要还没有 `UpdateDomain` 被应用，
  `record_hash == None`；
- TLD 与域名的 id 空间**在构造上不相交**
  （不同的派生前缀，见 `/docs/en/general/naming.md`）：
  注册 TLD `uip` 绝不能侵占域名的身份，反之
  亦然。

### 应用规则

`state.apply(transaction)` 是确定且原子的（出错时
状态不变）：

**所有交易（M8b，线上格式 2026-09）**：交易不再
携带 `network` 字段 —— 分离依赖**签名**：
被签名的载荷注入链的 `network_id`
（`validate_transaction(tx, network_of_the_chain)`），
因此为另一个网络签名的交易签名验证失败
（`InvalidSignature`、发生在应用之前 —— 测试网
签名的交易绝不被应用、绝不被主网节点放入
内存池，反之亦然）。

**REGISTER_TLD**：

- TLD 必须**空闲**（不在 TLD 注册表中）且**非
  ICANN 保留**（`IcannTldReserved`：ICANN 根 TLD 在
  Scone 上不属于任何人）；
- **强制的 PoW**：对挑战
  `SCONE-TLD-V1 || tld`（由交易携带的名称派生）
  执行 `pow::verify`，难度为标签的**按长度**
  难度（`pow_length`）：私有诞生 →
  `tld_difficulty`（网络 apex）、开放诞生
  （`open = 0x01`）→ `open_tld_difficulty`
  （apex − 2）—— 摘要从头重算
  （盐 `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖
  epoch`、Argon2id 核心），自声明的难度必须
  等于期望难度，且证明**声明的纪元**必须等于
  应用区块的 `height/POW_WINDOW`（防预挖的
  纪元加盐 —— 类型化拒绝 `ProofEpochMismatch`，
  见 `transactions.md`）；
- 应用时：`open = 0x01` → `{ Open }`（无主 ——
  自助、PoS 席位释放、死亡保底置于
  `now + grace`）；`open = 0x00` →
  `{ Private { owner } }` —— **默认封闭**
  （仅限 assign）；开放须经显式的
  `SetTldOpen`。

**REGISTER（RegisterDomain）**：

- **所携带名称的 TLD 必须已注册且开放**：链
  自行派生 `TldId(name.tld())`（绝不是提供的
  值）；不存在 → `UnknownTld`（D1、M7c）、存在
  但封闭 → `TldClosed`（M8b：仅限 assign 的
  路径是排他的）；
- 域名必须**空闲**：不在状态中且不在任何
  已过期注册的宽限窗口内（见下文 GC）；
- **强制的 PoW**：挑战 `SCONE-DOMAIN-V1 || name`、
  难度为**最左标签的按长度难度**
  （`pow_length::domain_difficulty` —— TLD 既然
  开放，这就是自助入场价）；
- 应用时：`{ owner, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 年 }` ——
  **注册持续 1 年**（365 天）。

**区块内顺序**：有意义。同一区块内
`[RegisterTld, SetTldOpen, RegisterDomain]` 有效；
任何在前置条件存在之前应用规则的排列都会
导致整个区块被拒绝（undo-log）。

**UPDATE（UpdateDomain）**：

- 域名必须存在（未过期）；
- `tx.owner` 必须是当前所有者；
- `tx.sequence` 必须**恰好**为 `current_sequence + 1`
  （无间隙、无重放）；
- 应用时：`sequence` 前进、`record_hash` 被替换。

**TRANSFER_TLD (0x47)**：

- TLD 必须存在且**私有**（否则
  `UnknownTld` / `TldIsOpen` —— 开放 TLD 不属于
  任何人）；
- 签名者必须是当前所有者（否则
  `NotTldOwner`）；
- 应用时：`owner = new_owner`（接收方身份是
  不透明的；与可花费密钥的绑定由后续
  交易完成）。无序列号：第一个被应用的
  转移获胜（链上顺序）。

**REVOKE_TLD (0x6B)**：

- TLD 必须存在且**私有**（开放 TLD 不能被
  放弃 —— `TldIsOpen`：它在 GC 处自然死亡）、
  签名者 = 当前所有者；
- 应用时：TLD 从注册表移除，可通过一次新的
  `RegisterTld`（含 PoW）重新声明。

**SET_TLD_OPEN (0xB8)**：

- TLD 必须存在且**私有**（已开放 →
  `TldIsOpen`：先前的所有者已被抹除，不存在
  任何可以为其签名的合法密钥）；`open = 0x00`
  （关闭）→ 在纯验证阶段即类型化拒绝
  `TldCloseUnsupported`：**开放是不可逆的**
  （所有者选项）；
- 签名者 = 当前所有者（`NotTldOwner`）；
- **开放 = 强制 PoW**：对挑战
  `SCONE-TLD-OPEN-V1 || tld_id` 按网络的扁平
  `open_tld_pow_difficulty`（测试网 6 位 /
  主网 22 位）执行 `pow::verify`，含纪元加盐
  （证明在交易的已签名 `open_proof` 字段中
  传输）；
- 应用时：`TldState::Open` —— **所有者被抹除**
  （不可逆、PoS 席位释放）、死亡保底置于
  `now + grace`。

**ASSIGN_DOMAIN (0xD4)**：

- 所携带名称的 TLD 必须存在且**私有**
  （否则 `TldIsOpen`：开放 TLD 仅限自助）、
  签名者 = TLD 的所有者（`NotTldOwner`）；
- 域名必须空闲（与 REGISTER 相同的宽限
  规则）；
- 应用时：`{ owner: assignee, sequence: 0,
  record_hash: None, registered_at: now,
  valid_until: now + 1 年 }`。**无 PoW**：为它
  背书的是命名空间的所有者。

**TRANSFER_DOMAIN (0xE2, M8c)**：

- 域名必须存在（否则 `UnknownDomain`）、
  签名者 = 当前所有者（`NotOwner`）；
- 应用时：`owner = new_owner`（PoS 席位在
  所有者池中跟随域名，与 `TransferTld`
  对称）。无序列号：第一个被应用的转移
  获胜（链上顺序）。

**RENEW_DOMAIN (0x3C)**：

- 域名必须存在；**TLD 种类首先被求值**
  （M8c 加固）：
  - **私有 TLD**：**PoS** 授权 —— 签名者就是
    TLD 所有者，或 TLD 所有者的席位委托给了
    签名者的公钥
    （`effective_operator(tld_owner) ==
    renew.public_key` —— 委托总是从 TLD 所有者
    读出，绝不反向）。`proof` 字段必须为空
    （非空证明 = 拒绝）。
  - **开放 TLD**：**PoW** 授权 —— 签名者是
    当前域名所有者（否则 `NotOwner`），证明
    挖出续期挑战：与创建相同的难度
    （按标签长度）、**绑定到当前到期时间**
    （`challenge = DOMAIN_ID_VERSION ‖ canonical
    name ‖ current valid_until`）—— 续期的
    证明不同于注册以及此前每一次续期的
    证明（防重用）；
- `valid_until` 必须**严格延长**当前期限
  （否则 `RenewalNotExtending`）；
- `valid_until ≤ now + 3 年`（`RenewalExceedsTerm`
  —— 防囤积边界：**1 个续期期限 = 1 年、最多
  预付 3 年**）；
- 应用时：`valid_until = tx.valid_until`；续期
  还推迟其下开放 TLD 的死亡保底（交易为此
  携带规范名称）。

**确定性 GC（M8b）**：在应用一个区块的交易
之前，链对照**父区块的时间戳**求值过期：
任何 `valid_until ≤ now` 的域名都被从活跃
状态移除并停入 **30 天宽限窗口**
（`valid_until + 30 天`）。宽限期内只有原
所有者可以重新声明该名称；之后它对所有人
空闲。GC 记入日志：被拒绝的区块逐位
恢复 GC 前的状态。

**TLD 生命周期（固定期限）**：TLD 携带自己的
时钟，独立于域名。

- **私有 TLD**：声明开启 **3 年期限**
  （`TLD_TERM_SECS`）。续期（`RenewTld`、0x8F）
  支付**与创建等量的 PoW** 并精确重置 3 年
  （固定期限在应用时重新推导）。到期时
  **60 天宽限**（`TLD_GRACE_SECS`）开启：续期
  仍可能，但**其他所有管理操作都被冻结** ——
  尤其是 `AssignDomain`（类型化拒绝
  `TldExpiredFrozen`）：过期的所有者在续期
  之前失去管理权。其下已分配的域名过各自
  的 `valid_until` —— TLD 冻结，但其子域名
  不受影响。宽限期过而无续期：TLD 在 GC
  处死亡（PoS 席位释放、可重新声明）。
  `TransferTld` 携带剩余期限。
- **开放 TLD**：生而（或被开放时）带 **1 年
  保底**（`OPEN_TLD_FLOOR_SECS` —— 防瞬间
  创建销毁）。其下每个存活的域名都把死亡
  时钟往后推（跟随最新域名的到期）。无
  域名且保底已过：GC 处死亡、可重新声明。
- **不朽开放 TLD**（仅创世：`.scone`、`.tor`
  —— 经 CNAME 在洋葱地址之上提供可验证
  命名 ——、`.p2p`）：`OpenImmortalTld` 分配
  设置 `until = u64::MAX` 哨兵 —— GC 永远
  无法越过它，单调重设也永远无法调低它：
  该 TLD 永不过期，即使其所有域名都被
  清空。没有运行时交易能创建不朽
  （`RegisterTld` 保持 1 年保底）；其下域名
  过正常的周期。

TLD 的死亡与域名共享确定性的死亡队列
（`tld_deaths`），记入日志且可逐位恢复。

**高度 1 例外**：创世携带一个结构上为空的
时间戳（`GENESIS_TIMESTAMP` = 0，协议约定 ——
不是时钟）。因此第一个区块的链上时间是
它自己的时间戳（承诺在其头部、确定性），
而非其父区块的：把区块 1 的过期锚定在 0
会把生而开放的 TLD 的死亡保底（以及区块 1
出生的域名的 `valid_until`）钉在 Unix
纪元 —— 区块 2（真实时间戳）的应用前 GC
会在应用之前移除该资源（"dropped
conflicting tx from block production: unknown
TLD"、测试网回归 60）。

**重放**：跨网络重放在三个层面上不可能 ——
不相交的创世（`UnknownParent`）、注入每个
签名载荷的 `network_id`（在另一网络上签名
无效 —— 这是自线上格式 2026-09 起网络出现的
唯一位置）、摘要中按网络分离的 PoW。同一
网络内完全相同交易的重放因状态规则失败
（双重声明、序列号），绝不因 `TxId` 失败。

同一网络中从同一状态出发、以相同顺序应用
相同区块的两个节点产出完全相同的最终
状态。

## 交易排序

顺序是每一层都承诺的基本属性：

```text
transaction order in the block
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
canonical chain
```

两笔交易不可互换：同一区块内
`[REGISTER, UPDATE]` 可应用，`[UPDATE, REGISTER]`
被拒绝（`UPDATE` 时刻域名不存在）。全局
平局裁决（为内存池排序、产出区块）属于
共识。

## 区块验证（push_block）

一切都**重新计算**，没有什么是凭信任接受的 ——
对等节点提供的 `tx_root`、哈希、签名或字段
绝不被按面值接受：

1. `prev_hash` == 规范 tip 的哈希（否则父
   未知时 `UnknownParent`、父已知但非 tip 时
   `ParentNotTip`：最小分叉检测）；
2. `height` == tip 高度 + 1；
3. `version` **恰好** == `PROTOCOL_VERSION`
   （任何其他版本 —— 更低或更高 —— 都被
   拒绝，而不是被重新解释）；
4. 交易数 ≤ `MAX_TXS_PER_BLOCK`（4096）；
5. 在区块顺序上对交易重算 `tx_root`；
6. 共识钩子（`validate_header`、`validate_tx`）；
6ter. **窗口化 TXID 防重放（.bak 移植）**：
   `TxId` 出现在最近
   `REPLAY_WINDOW_BLOCKS = 256` 个区块的包含
   索引中的交易被拒绝（`TxReplay`）。索引
   （TxId → 包含高度）是规范链的纯函数、
   以相同节奏修剪；索引在启动时由存储
   重建（重扫最近 `min(256, height)` 个持久化
   区块 —— M8，见 `/docs/en/technical/storage.md`）：
   重启的节点与在线节点拒绝相同的重放。
   窗口之外，重新包含仍受状态规则约束
   （双重声明、序列号重放）。
6bis. **签名出块者（M5、.bak 移植）**：`consensus`
   载荷必须能解码为出块者载荷
   （`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`、有界）；
   签名覆盖*出块者签名哈希* = **清空
   `consensus` 字段后**头部的 `BlockHash`
   （载荷就在那个字段里；签完整哈希会
   循环）；出块者必须属于区块 `timestamp`
   处被授权的集合（检查点选举的委员会、
   加上解锁的恢复抽签；引导期：存活域名
   的所有者 —— 空池 = 开放生产，否则第一
   次注册将不可能）。创世（高度 0、空
   载荷）是结构性的、永不涉及。类型化
   错误 `InvalidProducer`；
7. 对每笔交易，按顺序：**密码学**验证
   （`validate_transaction`：重算所有者/密钥
   绑定 + 对从头重算的签名载荷
   `verify_strict`，见
   `/docs/en/technical/transactions.md`），然后在
   **undo 日志**之下应用到规范状态
   （undo-log：每笔已应用交易一条记录、
   成本 O(区块交易数) 而非 O(域名数)）；
8. 原子提交：一切通过 ⇒ 区块加入、日志
   丢弃；哪怕最轻微的错误 ⇒ 日志回卷（逆序）、
   链严格不变、逐位一致。

没有任何验证函数 panic；任何畸形或敌意
输入都产生类型化的 `BlockchainError`
（`OwnerKeyMismatch`、`InvalidSignature` ……）。

## 区块组装（BlockBuilder）

`BlockBuilder`（`scone-blockchain::builder`）从已
验证交易的队列组装区块：

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- 在排队交易的有序列表上**重算** `tx_root`
  （绝不被提供）；
- `prev_hash`/`height` 链到父区块（父哈希由
  调用者经 `block_hash` 计算，绝不取自对等
  节点）；
- `version` = `PROTOCOL_VERSION`；
- `MAX_TXS_PER_BLOCK` 上限在插入时强制
  （`TooManyTransactions`）；
- `consensus` 载荷（已签名出块者载荷）与
  `timestamp` 由调用者提供：具体共识经
  现有的 [`Consensus`](#共识抽象) trait 插入，
  与验证对称；
- M5：`.with_producer(&SigningKey)` 以出块者
  身份签名区块 —— `consensus` 载荷成为
  已签名出块者载荷（覆盖上述出块者签名
  哈希）并取代任何先前的 `.with_consensus`。

组装好的区块无需重新编码即通过
`push_block`（已测试）。

## 分叉与分叉选择（M3c、.bak 移植）

`push_block` 仍是线性入口（父 == tip，否则
`UnknownParent`/`ParentNotTip`）。`try_attach`
（M3c）评估竞争区块：

- 父未知 → `UnknownParent`（区块等待）；
- 血统有效但落后 → 分支评估：最长链
  获胜；完全平局 → 确定性平局裁决
  （字典序最小的 tip 哈希 —— 持有相同区块的
  两个节点总是选出同一胜者、已测试）；
- **终局性下限**：某个祖先与已终局检查点
  矛盾（包含在被接受检查点的
  `prev_checkpoint_hash` 中）的分支被拒绝
  （类型化错误
  `Consensus("branch contradicts a finalized
  checkpoint")`）；
- 采纳 = 从创世重放获胜分支的区块
  （逐位确定性已测试：以不同顺序收到相同
  区块的两个节点最终处于同一状态）；
- 没有任何收到的区块在完整验证之前成为
  规范。

## 有界 RAM 窗口（M7a、.bak 移植）

链只在 RAM 中保留**最近 512 个区块**
（`RAM_WINDOW_BLOCKS`，.bak 的
`HISTORY_KEEP = 512` 的移植），位于**终局性
下限**（最后一个已终局检查点）之上：

- 区块只有**同时**在窗口外
  （`height < tip − 511`）**且**严格低于终局性
  下限才离开 RAM —— 下限之上的重组始终
  可能，因此这些区块绝不被驱逐；
- 在第一个已终局检查点之前（引导期），
  **什么都不被驱逐**：有界内存随终局性
  开始；
- 驱逐是真实的：区块、哈希索引
  （`known_hashes`、自身有界、按高度索引）
  与交易离开内存 —— 存储
  （`NodeStore`、`scone-storage`）持有持久
  副本；
- 窗口之下的 `block(h)` → `None`；
  `block_result(h)` → 类型化错误
  `BlockPruned { height }` —— 重组路径
  （`push_block`、`try_attach`、`adopt_branch`、
  前缀重放）传播它：中继必须从存储重新加载
  该段或回退到全量同步（已记录的限制：
  完整中继移植之前无分段采纳）；
- 建立在已被驱逐父区块之上的区块归类为
  `BlockPruned`（而非 `UnknownParent`）：父
  高度 < `base_height` ⇒ 存储可以提供它。

## 共识抽象

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus`（默认）：接受一切 ——
  在真实规则定义之前使用；
- 钩子由 `push_block` 调用：具体共识
  （PoW、难度、费用、时间戳/防重放、分叉
  选择）将无需重写链即可插入；
- 单一抽象，无 trait 层级。

## 可扩展状态（KV 后端、SMT 支点、过期队列、有界所有者池）

目标：2500 亿以上的潜在域名 —— 绝不把一切
装进 RAM。四个部件：

### KV 后端（`state_backend.rs`）

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = `DomainId`/`TldId` 的 32 个裸字节 ——
  一个有序键空间（不相交的派生前缀），值为
  `SCONE-ENTRY-DOM-V1`（固定 88 字节）/
  `SCONE-ENTRY-TLD-V1`（可变：私有 33 字节 /
  开放 1 字节、所有者选项枚举 —— 标志 +
  条件所有者、不再有 `until` 字段）编码；
- `MemoryBackend`（BTreeMap）= 参考实现
  （开发、测试）；`scone-blockchain` 中无具体
  依赖（该 crate 禁止 redb）—— 协议/共识仅
  依赖 trait 本身；
- `CachedBackend<B>`（`state_cache.rs`）= 内层
  后端之上的有界 LRU 缓存：直写（内层
  始终最新）、未命中绝不缓存、`range` 对
  内层分页 —— 与单独的内层后端回答相同
  （确定性契约、已测试）；
- `RedbBackend`（`scone-storage::redb_backend`）=
  持久化后端：`state_kv` 表（32 字节裸键、
  字典序 = 所需的升序）、计数器维护在
  `state_kv_meta` 元表中（绝不用 COUNT）；
  `CachedBackend<RedbBackend>` = RAM 占用
  有界的状态 —— "轻量层"；
- `smt_persist`：消费 SMT 的脏集合
  （`take_dirty_sets`）并将其写入后端中
  保留前缀的键下（0x01 节点、0x02 叶）——
  分页 SMT 的基础；
- trait 不再要求 `Default`（磁盘数据库没有
  合理的默认值）；
- `range` 按键升序迭代（确定性契约）——
  仅归档路径、绝不是共识规则；
- `ChainState` 仅按键经后端访问域名/TLD
  （点读、记日志的写）。

### 规范 SMT 根（支点）

- `state_root_smt()`（O(1)、缓存根、
  `SCONE-STATE-SMT-V1`）是**规范的**
  state_root：它是 `checkpoint_data` 所承诺、
  `accept_checkpoint` 在 tip 处验证的。计算
  绝不遍历域名集合；
- 旧的直接折叠 `SCONE-STATE-V1` 以
  `state_root_archive()` 存续：O(N)、归档/
  验证路径（从存储全量重折叠以交叉核对
  历史根）—— 不是共识产物。两者承诺
  相同的叶：逻辑状态相等 ⇒ 根相等
  （对两者都是）；
- 带 SMT 根的被接受检查点在恢复
  （reload）之后重新验证通过：根是恢复
  状态（后端 + SMT + 索引）的纯函数。

### 过期队列（GC O(expired) 而非 O(N)）

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>`
  —— 过期时刻 → 届时过期的 id（每槽位内
  id 有序：状态相等与清空顺序与到达顺序
  无关）；
- `gc_expired_journaled(now)` 清空
  `range(..=now)` 而不是扫描整个状态：成本
  O(该时刻过期数)；
- RAM 边界：每个存活域名一条记录（仅
  待定过期 —— 续期重新为槽位配键、GC
  清空它），绝不是扫描结构中每域名一个
  节点；
- 由 undo 日志维护（逐位回滚：恢复的槽位
  被放回）。

### 有界所有者池（密钥而非域名）

- `owner_refcounts: HashMap<OwnerId, u64>` ——
  每个**不同的**存活所有者（域名 + TLD）
  一个计数器，由日志在每次
  register/transfer/revoke/assign/GC 时维护
  （acquire/release）；
- `eligible_validators(now)` 读取该索引 ——
  O(owners)、绝不对域名迭代。时间上的
  存活性已有保证：确定性 GC（父时间戳）
  已在调用前移除每个过期域名及其引用
  计数。输出有序（确定性）；
- RAM 边界：O(不同所有者)、与域名数量
  无关；
- `owner_pool_root`（归档折叠）折叠这些有序
  的所有者 —— 同样不扫描域名。

稳态 RAM 预算（磁盘/网络后端）：
O(不同所有者 + 待定过期 + 进行中的宽限窗口 +
undo 日志) —— 全部与域名数量无关地有界、
全部由日志逐位恢复。

## 席位与终局性委员会（精确的 PoS 机制）

本节描述实现中**精确**的资格与委员会选举
机制（`finality.rs`）—— 运维层面的补充（锚
循环、线上格式）位于
`/docs/en/technical/relay.md`，安全分析位于
`/docs/en/technical/security-model.md`。

### 席位 = 持有（无注册）

PoS **席位**不是注册：它是对一个存活域名
或 TLD 的持有。`eligible_validators(now)` 计算
资格池：

- 经有界 `owner_refcounts` 索引取存活所有者
  的密钥（每个存活域名/TLD 一个**不同的**
  所有者 —— 一把密钥下的一百个域名 =
  一个席位：经济上的抗女巫按身份
  线性）；
- TLD 算数（其声明花费了 PoW —— 存活 TLD
  的所有者是利益相关者），存活域名也算
  （`valid_until > now` —— 由确定性 GC 保证、
  调用前已清空）；
- **减去被封禁的密钥**（M9：被罚没的歧义
  行为者终身出局）；
- **开放** TLD（`owner: None`）不持有席位：
  其先前的所有者在开放时被释放
  （`SetTldOpen` 应用中的 `owner_release`），
  它在 GC 处死亡、不再投票。

输出有序（确定性顺序、与索引的迭代顺序
无关）。

结构性后果：**席位的死亡 = 域名/TLD 的
死亡**。过期 + GC、`RevokeTld`（席位释放）、
`TransferTld`/`TransferDomain`（席位跟随域名到
新所有者）、`SetTldOpen` 开放（席位释放）——
每一次所有权转换都对称于
`owner_refcounts` 日志（acquire/release）地在
池中移动或释放席位。

### 资格池冻结在最后一个已终局检查点（`FinalizedBase.pool`）

接受检查点时（`accept_checkpoint`），终局性
基座捕获**已终局区块处**状态的资格池：

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(已终局区块的
  时间戳)` —— 终局性此后**绝不**重读活跃
  状态：检查点之后的任何注册（声明、转移、
  封禁）只对更晚的纪元生效；
- `seed` = `next_seed(signing_hash(checkpoint))`
  —— **下一个**纪元的种子派生自检查点的
  **内容**哈希（带键、不含签名集 —— 持有
  同一检查点不同法定人数子集的两个视图
  选出同一个委员会）。

### 委员会抽签（引导、新鲜度、降级）

`committee(recovery)`：

1. **引导**（无已终局检查点）：整个资格池
   签名 —— 从本地视图抽出的委员会会在
   落后的节点间产生分歧（终局性冻结的
   根因）；整个池按链是确定性的。严格的
   选择在第一次终局性时恢复。
2. **已终局政体**：从冻结资格池的**新鲜**
   子集抽签（RFC 服务证明 §5.2）—— 每个
   席位（所有者密钥）由 tip 处的状态解析到
   其**有效**运营者
   （`SetTldOperator`/`SetDomainOperator` 委托 =
   最近的链上事实，或自运营时的所有者
   密钥）；席位**以其有效密钥**参与（签名
   的正是那把密钥），前提是它有**新鲜的
   心跳**（滑动窗口）。被封禁密钥保持
   排除。
3. **安全降级**：新鲜运营者少于
   `MIN_FINALITY_COMMITTEE_SIZE`（4）→ 从
   整个资格池抽签（冷启动引导、普遍的
   运营者故障 —— 构造上终局性绝不丢失）。

`allowed_producers(block_ts)`（区块生产）遵循
同一方案：引导期为整个池；已终局政体为
当选委员会 + 恢复抽签（在共识时间的
k × `epoch_secs` 之后解锁抽签 k）。

检查点法定人数为 `quorum_for(委员会规模)`
（floor(2n/3)+1）；低于
`MIN_FINALITY_COMMITTEE_SIZE` 的委员会
绝不终局化（BFT 防护）。

## 推迟到共识

在此 crate 中刻意未定义：

- **内存池**：细粒度准入/重排序（费用、
  替换）留在中继（M4 带来了按域名上限 +
  窗口化 TXID 防重放 —— 
  `REPLAY_WINDOW_BLOCKS = 256`）；
- **全局排序**：内存池排序是局部的确定性
  排序（中继），不是全局权威；
- GC 之外的**时间戳权威**：过期（M8b）使用
  父区块的时间戳；其他时间规则（细粒度
  防重放）留在共识中；
- 分布式存储：状态在内存中；后端
  （`scone-storage`、redb）应用这些规则而不
  重复它们。

自 M8b 起已交付（此前被推迟）：按网络的注册
PoW（`RegisterTld` 总是、开放 TLD 上的
`RegisterDomain`）、过期/续期 1 年 + 30 天宽限 +
3 年视界、父区块时间戳处的确定性
GC、跨网络重放不可能（创世 + 已签名字段 +
PoW 摘要）。

## 确定性分布式仿真（P1.6）

`crates/scone-blockchain/src/sim/` 包含一个
**确定性的**网络仿真装置（`SimNet`）：N 个
真实的 `Blockchain`（无 mock），由**虚拟时钟**
与按 `(tick, 插入顺序)` 排序的**事件队列**
驱动。无套接字、无异步、无墙上时钟 —— 每次
抽取（丢失、延迟、重复）都是 `(seed, 发送
序号)` 的纯函数，因此固定 seed 的场景可以
逐字节复现。

### 模型

- **传输**：每次发送都经历概率丢失
  （按 ‰ 可配置）、有界人工延迟
  （`latency_ticks`）、可能的重复（一份延迟的
  副本）以及当前分区图（跨组消息丢失）。
  重复绝不推进逻辑时钟也不触发新的
  发送：有重复与无重复的真实事件流相同
  （确定性契约、由 `duplicate_and_delay` 测试
  证明）。
- **流行病 gossip**：节点仅在首次见到
  区块/交易/检查点时向其连接的对等节点
  （同一分区）转发 —— 中继规则 H2（切断
  循环）。停泊的（孤儿）区块绝不被
  重新广播；接收空隙由同步路径
  （`GetBlocks`）填补，如同中继。
- **生产**：每个间隔一次全局生产步；每个
  分区组每轮至多产出一个区块，由持有最高
  高度的授权节点产出。引导期（第一次
  终局性之前）出块者固定为编号最低的
  节点：授权出块者集合从状态（存活域名
  池）派生，两个略有差异的视图会选举
  不同的委员会，一个本地合法的区块在
  别处会是 `InvalidProducer` —— 永久分歧。
  第一次终局性之后，委员会冻结在终局性
  基座中（任何已收敛节点上都相同），
  生产自由。
- **终局性**：在 `sign_interval` 高度的每个
  倍数处，委员会的指定锚节点经真实的
  崩溃安全 `SignerGuard` 签署其 tip 的确定性
  内容（每纪元一把密钥 = 一票、扩散前
  状态 fsync）；其他委员会成员在收到后
  签名（在验证所引用区块确为该高度处
  他们的规范区块之后 —— 中继级防护）并
  重新广播丰富后的聚合。达到法定人数时
  `accept_checkpoint` 终局化。每个节点承诺
  每纪元见到的第一个提案（引导期防双重
  终局防护：同一纪元签名者不相交的两个
  竞争提案无法都累积）。
- **崩溃/重启**：崩溃的节点丢失全部 RAM
  状态（链、内存池、停泊区、视图）但在
  磁盘上保留其 signer-guard 目录，与真实
  进程一样；它通过重放对等节点提供的
  规范区块重新同步
  （`GetBlocks`/`GetCheckpoints`）。

### 已覆盖的场景（`sim::tests`）

| 测试 | 证明了什么 |
|---|---|
| `convergence_simple` | 8 节点、5 ‰ 丢失、延迟 1–5 tick、500 区块：处处相同 tip、相同 `state_root_smt`；活跃终局性；每纪元绝无两个不兼容检查点。 |
| `partition_then_heal` | 4a/4b 分区约 300 区块：两侧都生产，愈合 → 输掉分支重组后收敛；交叉检查点窗口绝不与共享纪元矛盾（愈合前后皆是）。 |
| `crash_restart` | h≈200 崩溃、h≈400 重启：通过重放网络区块重新同步 —— 相同高度、tip、状态；崩溃后的交易对复活节点可见。 |
| `duplicate_and_delay` | 10 ‰ 重复：最终高度、状态根与已终局检查点数与无重复运行相同（幂等：TxReplay/seen-sets 吸收重复）。 |
| `concurrent_tx` | 同一域名上的两个并发所有者：恰好一个更新获胜（所有者 —— 入侵者 `NotOwner`、干净类型化拒绝），绝无双重状态应用，输掉的交易不进入任何链。 |

### 参数与时限

默认值：`produce_interval` 7 tick、
`sign_interval` 16 高度、`sync_interval` 29–60 tick
（默认 41、某些测试中调低）、`horizon` 200k–500k
虚拟 tick。每个测试内嵌 debug 模式下
< 60 秒的墙上时钟防护
（`assert_under_60s`）。主导成本是验证内核
（debug 下每个空 `push_block_with_gc` 约
9 ms、Ed25519 未优化）：场景保持在约
8 节点 × 500 区块以下。

### 限制

- **没有真实的传输层**：本装置在恶劣投递
  下演练链逻辑，而非 swarm（libp2p、
  request-response、背压、发现）。真实的 e2e
  测试（`crates/scone-network/tests/`）仍是
  传输层的证明。
- **RAM 内批量同步**：被服务的区块来自 RAM
  窗口（场景保持在 `RAM_WINDOW_BLOCKS`
  之下）；`BlockPruned` + 存储重载路径未被
  仿真（由链的单元测试覆盖）。
- **诚实的锚节点模型**：锚节点只签署可对
  其自身规范链验证的内容；拜占庭歧义
  行为（蓄意双签）被 `SignerGuard` 阻止但
  未作为主动攻击演练。
- **一次只有一个分区**：分组图是全局的、
  由驱动器顺序更改；持续重叠/演化的
  分区未被建模。

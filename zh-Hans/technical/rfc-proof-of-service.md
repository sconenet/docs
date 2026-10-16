# RFC —— 运营证明与终局性委员会（proof-of-service）

> 状态：规范性（NORMATIVE）—— 已端到端实现
> （第 1 至 5 步：core/protocol/blockchain/network/CLI；
> 本文档是合并形态）。下文的每条规则都已对照代码
> 验证；引用的常量就是编码中的值。
> 日期：2026 年 10 月（规范性合并）。触及：
> `scone-core`、`scone-protocol`、`scone-blockchain`、
> `scone-network`、`scone`（CLI）。

## 1. 动机

终局性资格池就是所有者池（存活域名/TLD 的
所有者）。持有即得席位 —— **运营**本身什么也
不给。本 RFC 把签署检查点的权利与在线中继的
运营证明绑定、而绝不以所有权为条件：

- **保留自由**：私有 TLD 只要续期其期限
  （`RenewTld`、3 年期限）就保持可分配、无任何
  活动条件。只有*检查点签署*是通过在线表现
  赢得的。
- **运营激励**：运营一个私有 TLD（或任何席位）
  = 进入委员会。运营者参与维持网络的运转、
  协议对此予以承认。
- **可度量的去中心化**：委员会成为*实际在线*
  的运营者集合、而非沉睡的密钥。

## 2. 概览

一笔轻量的 `Heartbeat` 交易（判别值 `0x71`）、
由**运营者**密钥签名、每纪元发出一次。状态
持有近期心跳的滑动窗口。一个纪元的委员会
从**其有效运营者持有新鲜心跳的**资格池子集中
选举。没有心跳：不能签名、但不丧失任何所有权
利。

## 3. Heartbeat 交易

```
Heartbeat {
    operator: PublicKey,     // Ed25519 key of the operator (the signer)
    owner: OwnerId,          // identity RE-DERIVED from operator (never trusted)
    tip_seen: [u8; 32],      // last tip seen by the node
    height_seen: u64,        // corresponding height
    timestamp: u64,          // Unix seconds
} + Ed25519 signature over the canonical payload (network injected)
```

被签名的主体（线上、`scone-protocol`）：
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖
height_seen.v ‖ timestamp.v ‖ operator[32]`（签名
者的密钥出现在主体尾部、如同每笔交易）
`‖ signature[64]`、即编码后约 230 字节 —— 
**无 PoW**：成本是持续的在线（每纪元一次
心跳、持续数月）。

签名者就是运营者：`owner` 总是从 `operator`
重新派生（绝不取自输入）、解码以类型化错误
（`InvalidOwner`）拒绝与头部 `operator` 字段不同
的签名密钥。

- `tip_seen`/`height_seen` 服务于可观测性（检测
  有 bug 或滞后的运营者）；它们**不**影响有效
  性（轻微滞后的节点仍是活节点）。

### 3.1 纯验证（scone-core）

1. 对网络的规范载荷有有效的 Ed25519 签名。
2. `owner` == 从 `operator` 派生的身份（否则
   `InvalidOwner`）—— 在编码、解码**以及**
   `validate()` 处检查。

### 3.2 状态规则（应用、scone-blockchain）

在高度为 `block_height` 的承载区块中应用一笔
心跳时（每次拒绝都先于任何变更）：

1. **跨纪元防重放**：每运营者 `timestamp` 严格
   递增 —— 若
   `timestamp <= operator_last_hb[operator]` 则
   类型化拒绝 `StaleHeartbeat`。
2. **每纪元唯一性**：每（`operator`、承载区块
   的纪元）只接受一次心跳 —— 重复以类型化错误
   拒绝（`HeartbeatDuplicate`）。纪元是
   `block_height / epoch_min_blocks`（与终局性
   相同的定义、绝不是新常量）。
3. 记日志地插入窗口 + 更新
   `operator_last_hb`（重组中可逆、逐位 undo）。

### 3.3 防垃圾

- 每运营者每纪元 1 笔被接受的交易 —— 最大
  流量是 `pool_size / epoch_secs`、可忽略。
- 无费用（经济层之前）：速率在结构上有界。

## 4. 运营委托

席位 → 运营者的链接经两笔专门交易：

- `SetTldOperator { tld_id, operator }`（判别值
  `0xA9`）、由**私有** TLD 的**当前**所有者签名
  （开放 TLD 没有可合法签名的人 —— 类型化拒绝
  `TldIsOpen`）。`operator: None` = 撤销、回到
  所有者密钥的自运营；可即时撤销。
- `SetDomainOperator { domain_id, operator }`
  （判别值 `0x5E`）、由域名当前所有者签名。相同
  语义。
- `RegisterTld` / `RegisterDomain` /
  `AssignDomain` 不携带任何 `operator` 字段：
  席位 → 运营者的链接始终是最近的链上事实、
  在 tip 处解析。

线上的运营者选项是规范的：`0x00` = `None`
（自运营）、`0x01` = 裸 32 字节密钥；任何其他
值在解码时被拒绝。

状态维护：

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, u64>       // bounded counter
operator_last_hb: HashMap<OperatorKey, u64>     // last accepted ts
```

应用规则（`apply_seat_operator`、记日志）：

- **幂等性**：向同一运营者重新委托（或撤销不
  存在的委托）是不记日志的无操作（逐位 undo
  要求如此）。
- **防出租边界**：一个运营者同一时刻只能被
  **唯一一个席位**委托
  （`MAX_SEATS_PER_OPERATOR = 1`）。第二个席位向
  已被占用的运营者委托 → 类型化拒绝
  `OperatorAlreadyDelegated`。理由：没有边界、
  大运营者会批量出售签名并再造中心化 —— 恰是
  当选委员会要避免的。

## 5. 新鲜度窗口

### 5.1 滑动窗口

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- 条目在有效心跳被应用时插入（§3.2）。
- 在**每个**区块应用时确定性清除
  （`gc_heartbeats_journaled`）：纪元 `e` 满足
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` 的任何
  条目被移除（记日志 —— 重组中可逆、如同过期
  队列）。
- `HEARTBEAT_WINDOW_EPOCHS = 2`：当前纪元 + 前
  一个 —— 运营者可以错过**一个**纪元而不失去
  新鲜度、第二个使其过期。
- **结构性内存上限**：窗口至多持有
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` 条
  （每运营者每纪元一条、元组键）；
  `operator_last_hb` 以运营者数量为界且绝不清除
  （防重放规则是单调的）。

### 5.2 新鲜度

"新鲜" = 在
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`
中的某个 `e` 存在条目 `(e, operator)`。由于状态
在每个区块清除、`recent_heartbeats` 中的存在**
就是**新鲜度（`operator_is_fresh`）。

### 5.3 有效运营者

席位的**有效运营者**是存在活委托时的
`seat_operators[seat]`、否则是所有者的密钥
（`effective_operator`）。席位以该有效密钥
签名 —— 委托移动的是签名权、绝非所有权。

## 6. 委员会选举

```
eligible = { effective operators of the frozen pool's seats
             (banned key excluded) having a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

规范性细节（在 `finality.rs` 中验证、规定于
`/docs/en/technical/blockchain.md` 的"席位与终局
性委员会"一节）：

- `pool` 是**冻结在最后一个已终局检查点处**的
  资格池（`FinalizedBase.pool` —— 绝非活跃
  状态）：本地视图会在节点间分歧、法定人数
  将永远无法形成；
- `seed` 派生自前一检查点的内容哈希
  （`signing_hash`）；
- 冻结资格池的每个席位由 tip 处的状态解析到其
  **有效**运营者（委托与心跳是最近的链上
  事实：同一 tip 处的两个节点算出相同的集
  合）；被封禁密钥保持排除；
- 抽取在有效密钥的排序/去重集合上进行
  （`select_committee` 不变）。

**显式且安全的降级**：引导中的网络（无心跳）、
或运营者的普遍故障、回退到整个资格池 —— 构造
上终局性绝不丢失。这是项目的"绝不死锁"规则
在此处的应用。

### 6.1 冷启动引导

纪元 0（以及只要新鲜运营者少于
`MIN_FINALITY_COMMITTEE_SIZE`）：整个资格池。
随着运营者发出心跳、约束自然收紧。没有需要
配置的特殊过渡期。

## 7. 自动发出（scone-network）

在每个生产周期（`Wake::Produce` —— 无新定时
器）、中继检查是否跨越了**纪元边界**
（`height / epoch_min_blocks`、与终局性相同的
定义）。在新纪元的第一个周期、若满足如下
条件则发出心跳：

1. 配备了锚密钥（`--anchor-key` 或自动）、
   且
2. 该密钥自身持有一个活席位（本地检查：该
   密钥在链上资格池中 —— 存活域名/TLD 的
   所有者、被封禁密钥除外）。

本地防垃圾：每节点每纪元至多一次发出
（`last_emitted_epoch` 记忆、本地内存、绝非链
状态）；只有在内存池注入成功后才取该记忆 ——
失败则在下一周期重试。这里没有任何致命的
东西：最坏情况下心跳等到下一周期。

**被委托的运营者**（自身不是所有者）不被自动
发出覆盖：他们经 §8 的 CLI 构建器发出（手动或
air-gapped 运营）。心跳不一定由发出者自己产
出到区块：它与任何交易一样（内存池、由任何
人生产）。

## 8. CLI（`scone`）

离线心跳构建器随 `tx` 命令面被移除
（cli-simplify-v1）。中继自动发出自己的心跳
（§7）；若将来再次需要手动/air-gapped 发出、
会以专门命令的形式回归。默认值曾是：
`tip_seen` = 32 个零字节（仅可观测性）、
`height_seen` = 0、`timestamp` = now。
- 如同每笔交易、`tx sign` 把 operator 与 owner
  **重新绑定**到真实的签名密钥（绝不信任输入
  值）；`tx verify` 本地验证。

## 9. 可观测性

- `NodeStatus`（线上 p2p 消息 `0x0d`）携带
  `fresh_operators: u32`（变长整数）：持有新鲜
  心跳的运营者数 —— 资格池中有委员会资格的
  活跃子集（资格池席位的有效运营者、被封禁
  密钥除外、排序/去重）。由中继从
  `state().fresh_operators(height)` 填充；由
  `scone status` 显示为 `fresh operators: N`。
- 活动论据："运营一个私有 TLD = 进入去中心化
  DNS 的终局性委员会" —— 可在链上验证。
- 未来表面（未指定）：`operator_status` —— 每
  席位的被委托运营者、新鲜度（fresh / expired /
  never seen）、最后的 `tip_seen`。本地控制
  RPC 已移除（纯 p2p 节点）；表面尚待定义。

## 10. 本 RFC 不做的事

- 不因不活跃而**丧失持有**（选项 A —— 未运营
  TLD 的衰减 —— 被显式拒绝）。
- **无奖励**（选项 C 等待经济层）。
- 心跳不证明服务的*质量*（产出的区块、服务
  的 DHT 记录）—— 只证明在线。计量真实贡献
  是可能的未来扩展、也更易被博弈。

## 11. 受保护的不变量

- 逐位确定性：窗口与选举是已应用状态的纯
  函数（记日志、逐位 undo）。
- 网络数据上绝不 panic：畸形的心跳或委托 →
  类型化拒绝（`InvalidOwner`、`StaleHeartbeat`、
  `HeartbeatDuplicate`、`OperatorAlreadyDelegated`、
  `TldIsOpen`、`NotTldOwner`、`NotOwner`）。
- 持有无条件；只有检查点签署是有条件的。

## 12. 常量与参数（编码值）

| 名称 | 值 | 位置 |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| `Heartbeat` 判别值 | `0x71` | `scone-protocol/src/transaction.rs` |
| `SetTldOperator` 判别值 | `0xA9` | 同上 |
| `SetDomainOperator` 判别值 | `0x5E` | 同上 |

这些值是共识的一部分：修改它们会破坏链的
可重放性（窗口/清除）或选举（委员会）。

## 13. 实现计划 —— 最终状态

| 步骤 | crate | 内容 | 状态 |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`、`SetTldOperator`、`SetDomainOperator`、线上格式、验证 | 完成 |
| 2 | blockchain | 滑动窗口（记日志）、过滤资格池、选举、约 15 个测试 | 完成 |
| 3 | network | 自动发出、交易中继 | 完成 |
| 4 | CLI | `tx build heartbeat`、`scone status` 中的 `fresh operators` 显示 | 完成 |
| 5 | docs | 本 RFC 合并为规范形态、fr/en 镜像 | 完成 |

# 网络中继（M4）—— 架构、流程与 CLI 控制

> Scone 网络中继（`scone-network`）的规范规格。
> 任何文档/代码分歧 = bug（修复事实来源）。

## 概览

**中继**是 Scone 的完整 P2P 节点：它维护规范
区块链（RAM + `NodeStore` redb）、中继交易与区块、
（devnet 模式下）产出区块、提供同步协议、在
Kademlia DHT 中发布并解析 `SignedDnsRecord`，并暴露
CLI 直接查询的 **p2p 协议**（`/scone/reqres/1`）
（RPC→p2p 统一：JSON TCP 通道已消失）。

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  ephemeral port    │  ├─ identify  (address discovery)      │
  by default) ◄─────┤  ├─ ping      (liveness)               │
                    │  ├─ kad       (record DHT)             │
                    │  └─ reqres    (scone-prot. messages)   │
                    │                                        │
                    │  Relay core (single task)              │
                    │  ├─ Blockchain (full validation)       │
                    │  ├─ Mempool (bounded, dedup by TxId)   │
                    │  ├─ BlockBuilder (devnet production)   │
                    │  ├─ Fork parking (bounded, 2048)       │
                    │  └─ Sync driver (batched catch-up)     │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, same messages)      │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` 是工作区中**唯一的异步
crate**：其下的一切（crypto → core → protocol →
blockchain → storage）保持同步且纯粹。

## 连接准入（P1.5）

swarm 层面的硬性限制（libp2p `connection_limits`
行为、`SconeBehaviour` 的第一个条目）：

- `MAX_TOTAL_CONNECTIONS = 128` —— 每个中继已建立
  连接（入 + 出）的上限；超出的入站连接被拒绝、
  绝不排队；
- `MAX_PER_PEER_CONNECTIONS = 4` —— 单个对等节点
  不能通过开许多连接垄断总配额；
- `MAX_PENDING_INCOMING = 32` —— slowloris 防护：
  未认证的半连接绝不无限等待。

这些限制补充既有的按消息限制
（`MAX_MESSAGE_LEN` 1 MiB）与按预算限制（同步
`max ≤ 128` 区块、DNS TCP 64 连接/512 B/5 秒）。
连接洪泛攻击者得到的是拒绝、不是内存增长。

## 对等节点发现

三种互补机制、按此优先顺序：

1. **显式种子**（`--bootstrap`、完整 multiaddr）——
   总是优先；提供非空列表**取代**其他一切。
   `--no-builtin-seeds` 清空内嵌种子以实现 100% 私有
   部署。
2. **内嵌种子**（`scone-network/src/config.rs` 中的
   `builtin_seeds(network)`）—— 在每次官方二进制
   发布时冻结、按网络。未配置的中继独自加入
   网络。（只要公共测试网未部署、测试网上为
   空。）
3. **习得目录**（`<data-dir>/peers.json`）—— 每个经
   `identify` 习得的地址都被保留（最后所见；
   上限 **2000 条** —— 是地址而非连接、最大约
   400 KiB RAM —— 以及 **90 天年龄截断**：最后
   所见超过 90 天的条目在写入时被丢弃、在加载时
   被忽略、使 2000 个槽位保持充满可信的对等
   节点、而不是只剩下启动时拨号超时的早 churn
   化石）并在生产 tick 以约 1 次/分钟保存。无
   种子且无 bootstrap 启动时、目录被重新组合：
   中继无需配置就找到它惯常的对等节点。它是
   缓存：文件缺失/损坏 = 空目录、绝不是错误。
4. **UPnP**（`--upnp`、可选）—— 由家用路由器做
   QUIC 监听端口映射：节点无需 circuit relay
   即可从外部到达。在 `NewExternalAddr` 时、外部
   地址被注册为 swarm 的真实外部地址
   （`add_external_address` —— 被 identify/autonat
   看到、是 transport-relay 候选）。无 IGD 网关
   （`GatewayNotFound`）或网关不可路由
   （`NonRoutableGateway`）：一条普通 info 日志、
   跳过映射 —— 家用 LAN 上的正常情形。过期
   的映射（续期失败）是警告。

UPnP 保持**默认禁用**（`config.rs` 中
`enable_upnp` = `false`）：公共部署未经同意不与
路由器对话。**mDNS 发现已被移除**
（remove-mdns-discovery、2026-10-22）：对等连接
仅来自显式 bootstrap、内建种子与 Kademlia ——
旧版 `--mdns`/`SCONE_MDNS` 标志仍被接受但无
效果（脚本兼容、与 `--rpc` 同样处理）。

### 本地拓扑优化（常开）

中继使用纯粹的 `scone-topology` 引擎主动优化其
邻域（见该 crate 的文档）。**始终启用**；唯一暴露
的调节项是 **profile**（`--topology-profile`、环境
变量 `SCONE_TOPOLOGY_PROFILE`、默认 `default`）：

| profile | min / target / max | 评估 | 适用对象 |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | 树莓派、家用小主机、1 vCPU VPS |
| `small` | 4 / 8 / 12 | 90 s | 小型服务器、家用 NAS |
| `default` | 8 / 16 / 32 | 60 s | 标准服务器（默认） |
| `high` | 16 / 32 / 64 | 45 s | 主权节点、骨干 |
| `very-high` | 32 / 64 / 128 | 30 s | 创始人枢纽 |
| `devtest` | 2 / 4 / 6 | 5 s | **仅实验室/测试**（设计上不稳定） |

```
scone relay --topology-profile tiny
```

保护性参数（滞回 0.15、冷却 600 s、宽限
120 s、测量 15 s）处处相同：它们守护稳定性、
没有理由随节点规模缩放。未知名称是类型化启动
错误。

不一致的配置（`min > target > max`、阈值在
`[0,1]` 之外 —— 含 `NaN` ——、零间隔）是
**类型化启动错误**、绝不静默。

**视图（`TopologyView`）**—— 由 swarm 事件喂给：

- 成功的 `ping` → RTT 观测（8 样本的中位数
  窗口）；
- 失败的 `ping` → 可靠性惩罚；
- 收到 `identify` → 地址（每节点 ≤ 4）+ 宣告的
  协议（kad/reqres/identify/relay 位掩码）；
- 连接建立 → 连接状态、方向（入/出）、年龄、
  检测到的 circuit-relay；
- 连接关闭 → 惩罚 + churn 计数、仅在非自愿时
  （主动驱逐不计）；
- 出站拨号失败 → 可靠性惩罚。

视图有界（500 个对等节点；超出后只记住已连接
的对等节点）。

**来自 kbuckets 的 `kad_required`** —— 每次评估
之前、中继遍历 Kademlia 路由表：在某个桶中为
`Connected` 的对等节点、或在**饱和**桶中为
`Disconnected` 的对等节点（k = 20）受保护。受
保护的对等节点绝不被选为驱逐对象。

**优化循环** —— 按 profile 的评估间隔
（`default` 下 60 s；见上表）：

1. 重算 `kad_required`（kbuckets）；
2. 引擎评估（滞回：120 s 宽限期、最低 +15% 的
   分数改善、600 s 驱逐冷却）；
3. 应用决策：`AddCandidate` → 拨号候选；
   `Replace` → 关闭被驱逐的对等节点（只一次 ——
   渐进、不循环）然后拨号替代者；
4. 记录一份**指标**快照（INFO 级别）：
   已知/已连接/入/出/circuit/受保护的计数、
   RTT 均值/中位数、不同 IP 桶、最大集中度、
   churn（替换/断开）。

邻域分数绝不是信任度量（见 `scone-topology`
文档）：没有任何共识决策由它派生。

### 可达性（AutoNAT）与传输中继（circuit-relay v2）

每个中继都运行 AutoNAT（客户端 + 服务器）：节点
请求其对等节点尝试**回拨**它；判定（`reachable`：
是 = 从外部可达、否 = 在 NAT 后、unknown = 尚未
测试）每 5 分钟刷新并暴露于 `NodeStatus`（字段
`nat_reachable`、线上变长整数 0/1/2）—— CLI 显示
它（`scone status`）。私有地址计入
（`only_global_ips = false`）：局域网网格测试其
局域网可达性。

可达的中继启用 circuit-relay v2 **hop**（它可以为
NAT 后的对等节点中继流量）与 **DCUtR**（打洞：
经 circuit 连接的两个节点尝试直连并释放
中继）。被判定在 NAT 后的中继自动向第一个已知
可达对等节点请求保留并经其监听（地址
`/p2p/<relay>/p2p-circuit`）。

生产规则：经中继的 NAT 后节点不应成为区块
生产者（延迟/可用性劣化）—— PoS 规则（席位 =
存活域名）尚未禁止它；将在公共测试网之前作为
运维要求记录。

## 传输与 libp2p 协议

| 协议 | 名称 | 角色 |
|---|---|---|
| QUIC（quic-v1） | — | 主传输（udp） |
| identify | `/scone/id/1` | 对等节点地址发现（填充 Kademlia 表） |
| ping | libp2p 标准 | 存活性 |
| Kademlia | `/scone/kad/1` | DHT：键 = `DomainId`（32 B）、值 = `SignedDnsRecord` 的规范编码 |
| request-response | `/scone/reqres/1` | scone-protocol 应用消息 |

### 应用帧（`/scone/reqres/1`）

每一帧是 `len: u32 BE || bytes`、其中字节是
`scone-protocol` 中 `Message` 的规范编码（现有
编解码器 —— 无第二种格式）。长度在任何读取或
分配**之前**受 `limits::MAX_MESSAGE_LEN` 约束：
宣告更大帧的对等节点被零内存成本拒绝。

交换的消息（每次请求/响应往返一条）：

| 消息 | 方向 | 回复 |
|---|---|---|
| `Hello { version }` | 握手 | `Hello`（本地版本） |
| `Ping` / `Pong` | 存活性 | `Pong` |
| `GetBlock { hash }` | 查询 | `Block`（RAM 窗口）或未知时 `Pong` |
| `GetBlocks { start, max }` | 同步 | **一条** `Block`（高度 `start`）、或没有时 `Pong` |
| `Block (b)` | 区块中继 | `Pong`（确认） |
| `Transaction (tx)` | 交易中继 | `Pong`（确认） |
| `GetRecord { domain_id }` | 本地 DHT | `Record`（本地缓存）或 `Pong` |
| `Record (r)` | 发布 | 接受则 `Pong`、否则错误 |

规则：**每个请求恰好一条回复**；确认消息是
`Pong`。同步回复一次携带一个区块 —— 追赶由
请求方迭代（见"同步"）。这是刻意有界的：每个
对等节点在途区块绝不多于一个。

## 中继行为

### 启动

0. 网络解析（`--network testnet|mainnet|devnet`、
   项目开发期间默认 testnet —— M8b）。data-dir
   **按网络**（`~/.scone/<network>/`）：持有另一
   网络链的非空存储被拒绝（`Corrupted`）；
1. 打开/创建 redb 存储（`<data-dir>/chain.redb`）；
2. `load_chain(store, network)`：O(1) tip + 分页
   域名状态（RAM 窗口：创世 + tip）；
3. swarm 构造（每会话临时的 Ed25519 身份 ——
   是否持久化网络身份是未决事项）；
4. 组合启动列表（见[对等节点发现](#对等节点发现)）：
   显式 `--bootstrap`（必须以 `/p2p/<peer-id>`
   结尾）否则网络的内嵌种子否则习得目录
   （`peers.json`）；
5. 拨号每个 multiaddr 并注册到 Kademlia 表
   （死拨号被忽略 —— 其余节点组成网络）。

### 接收交易（P2P 或 CLI 提交）

1. 严格有界解码（`MAX_MESSAGE_LEN`）；
2. 完整密码学验证（`validate_transaction`：
   所有者/密钥绑定、对规范载荷
   `verify_strict`）；
3. **防重放优先**（廉价、独立于状态、终局），
   然后状态预检（`push_block` 规则的廉价镜像 ——
   昂贵的 PoW 只在 push 时验证）：重放的
   `RenewDomain` 回答 `TxReplay`、而非撒谎的
   PoW 错误（其证明绑定的是第一次应用已经越过
   的到期时间）。网络分离在第 2 步强制：签名
   载荷注入**中继**的网络、在别处签名的交易以
   `InvalidSignature` 失败（线上格式 2026-09：
   不再有 `network` 字段）；
   `RegisterDomain` → 被注册名称的 TLD
   （否则 `UnknownTld`、D1/M7c）**且开放**
   （否则 `TldClosed`、M8b）然后名称空闲；
   `UpdateDomain` → 已知域名、精确序列
   `current + 1`、所有者；`RegisterTld` → 空闲
   TLD；`TransferTld`/`RevokeTld`/`SetTldOpen` →
   已知 TLD + 所有者签名；`AssignDomain` → TLD
   所有者 + 空闲名称；`RenewTld` → 已知私有
   TLD + 所有者、固定期限重推导、宽限窗口
   预检；`RenewDomain` → 已知域名 + 所有者、
   下一高度处（绑定当前到期时间的）续期 PoW
   预检；
4. 内存池插入（固定容量、按 `TxId` 去重 ——
   重新提交池中已有的交易是无操作）；
5. 向对等节点广播（发送方除外）。

### 接收区块（P2P、同步或自产）

1. `push_block`：既有的完整验证（父 = 规范 tip、
   高度、重算 Merkle 根、每笔交易先验证再
   应用 —— 每区块原子）；
2. 若接受：中继读取应用结果
   （`push_block_with_gc` → 哈希 + 被确定性 GC
   移除的域名）并经 `store_block_with_removals`
   持久化（原子增量追加：区块 + tip + 被修改
   状态 —— upsert **与**移除：过期域名、被放弃
   的 TLD）。因此重启绝不能复活已过期的
   注册；
3. 内存池对账（被包含的交易被移除）；
4. 向对等节点广播（发送方除外）。

无效区块以类型化错误被拒绝；中继继续
运行。网络数据上无 panic。重活应用
（Argon2 + redb 提交）经
`tokio::task::block_in_place`（同步驱动、p2p
接受、生产）：运行时把就绪任务移到其他
worker —— 应用区块不再拖住 DNS 服务或指标。

### DNS 快照（发布）

在每个已应用的区块之后（p2p 接受 —— 含重组
——、同步驱动、生产）**以及**启动时（重启在
下一个区块之前绝不能对存活名称返回
NXDOMAIN），中继把**权威 DNS 快照**发布到 DNS
面读取的 `watch` 馈送上（完整规格见
`dns.md`）：

- tip 延长（压倒性路径）：**O(touched) 增量**、
  写时复制 —— 区块的交易点名被触及的域名、
  加上确定性 GC 的移除
  （`AppliedBlock.gc_removed_domains`、在 push 时
  求值）；
- 重组 / 启动：**全量重建**（一次分页的
  `ChainState::dns_facts` 遍历）。

### 区块生产（M5：签名出块者）

每 `produce_interval` 秒（默认 2 s）：若内存池
非空、由 `BlockBuilder` 组装区块
（`timestamp = now`）**由授权出块者签名**（载荷
`SCONE-BLOCK-V1` 位于 `header.consensus`、见
`/docs/en/technical/blockchain.md`）并走与收到区块
完全相同的接受路径。无交易的中继不生产
任何东西。

生产密钥：中继配备锚密钥（`--anchor-key`）时
为锚密钥 —— 锚是授权出块者 —— 否则为固定的
devnet 密钥、在生产者池为空期间有效（开放
引导：创世）。一旦存在存活的域名/TLD、devnet
密钥签名的区块被拒绝（`InvalidProducer`）。

### 锚循环（终局性、M5）

带 `--anchor-key`（`.sconekey` 密钥文件、口令经
`--anchor-passphrase-env`、默认
`SCONE_ANCHOR_PASS` —— `scone-network` 库默认的
`SCONE_ANCHOR_PASSPHRASE` 经二进制永远触不到、
clap 总是覆盖它）：

- **提议者轮换**：每个纪元恰有**一名**指定的
  委员会成员自提议（`committee[epoch % size]`、
  确定性）；其他成员绝不签署自己的 tip ——
  检查点的内容取决于提议者的 tip，而不加
  门控的自提议曾把纪元撕裂成相互竞争的内容、
  每个守卫都正确地将其作为歧义行为拒绝
  （testnet、2026-10-21）；
- 在每个被接受/产出的区块之后、若本节点是
  当前委员会成员**且**委员会达到
  `MIN_FINALITY_COMMITTEE_SIZE`、中继经其
  `SignerGuard`（数据目录中的
  `signer-state.dat` —— 一把密钥 = 一票/纪元、
  崩溃安全 fail-closed）签署提议的检查点并
  gossip 它；
- **收到即投票**：内容在我们的链上有效
  （被引用的区块存在、且哈希相同）并带有
  委员会签名的提议也会得到我们的签名 ——
  没有它，每个锚只签署自己 tip 的内容、法定
  人数永远凑不齐；
- 收到的签名（`Message::Checkpoint`）按
  `CheckpointData` 哈希累积（上界 256 条）；在
  **同一**数据上出现 `quorum_for(委员会规模)`
  个成员签名时 → `accept_checkpoint` + 广播已
  终局的检查点；
- **终局性持久化**：检查点窗口与冻结基座在
  每次终局化时写入存储、并在启动时重新
  加载。检查点是 gossip 状态（绝不由区块
  携带）—— 没有该持久化的重启会从空基座
  开始、再也无法链接对等节点提供的窗口
  （网络窗口之下的纪元锚已消失）；
- **终局性追赶**：基座为空的中继（全新节点的
  启动）每个 tick 向对等节点请求
  `GetCheckpoints`、直到基座恢复；
- 无锚密钥的中继保持**被动**：它累积、
  终局化、中继、但绝不签名；
- `GetCheckpoints` → 提供 `checkpoint_window()`
  窗口；
- 任何循环错误被记录并吞掉 —— 绝不致命；
- 状态（`GetStatus`）暴露 `checkpoints` 与
  `finalized_epoch`。

### 分叉停泊与重组（粘性少数派修复）

同高度的姊妹区块、或父区块不是当前 tip 的
区块（`ParentNotTip`/`UnknownParent`）绝不被
丢弃：它被**停泊**（有界 map、
`PARKED_CAP = 2048` 个区块、每个约 KiB —— 有界
的最坏情况 RAM；满载的停泊区静默忽略新来
者）。已被提供给 `accept_block` 的哈希被去重
（`seen_blocks`）：被重新 gossip 的输掉区块不被
重新评估。

在每个 `Produce` tick、`reconsider_parked`：

1. 线性清空父区块已成为 tip 的停泊区块；
2. 若某个停泊分支强于当前 tip（分叉选择规则
   **不变**：终局性下限、最长链、最低哈希）、
   采纳 —— 被置换的后缀本身被停泊（分叉的
   **两侧**都保持可评估）。

这解开了采纳了等高分叉输掉一侧的节点：
一次被提供然后被拒绝的获胜区块被周期性重新
评估、而不是被永远遗忘（chaos-30 发现：相隔
15 ms 的两个姊妹区块）。

**重组持久化**：采纳另一分支无法通过单调追加
（`NonMonotonicHeight`）。中继应用
`rewind_to_height(分叉点)` 然后
`persist_state_snapshot`（从链导出重写活跃状态）
然后获胜后缀的单调重新追加（见
`storage.md`）。持久化失败被记录并降级（启动时
经重放愈合）—— 绝不致命。

## 同步（追赶）

在每个到对等节点的连接上、中继请求
`GetBlocks(start = 本地高度 + 1)`。每个恰好补齐
期望高度的收到 `Block` 触发对下一高度的请求、
直到耗尽（`Pong` = 没有更多）或 `MAX_SYNC_ROUNDS`
（10,000）批次 —— 即使面对撒谎对等节点的绝对
上界。收到的区块走完整验证：恶意对等节点无法
让任何无效的东西被接受。

### 记录 DHT

- **put**（API/web 流程 —— CLI 十六进制通道已
  移除）：严格有界解码
  （`MAX_DHT_CACHE_ENTRY`）、对照链上状态验证
  （所有者、序列号、重算的 `record_hash` —— 见
  "记录验证"）、然后 `kad.put_record`（本地存储 +
  到最近对等节点的复制、复制因子的
  `Quorum::Majority`）与持久 redb 缓存。
- **get**（`domain resolve`）：`kad.get_record`；
  等待者绑定到其 `QueryId` 与所请求的
  `DomainId` —— 一条回复只解析匹配的查询、且
  记录键必须恰好是所请求的域名（在链上验证
  **之前**检查）。结果随后在返回前对照链验
  证。同时请求的上限：`MAX_DHT_WAITERS`（256）；
  超出则立即报错。成功验证的解析被写入持久
  缓存（M5）：本地读取（`domain_info`）此后无
  需新的 DHT 查询即可服务。
- Kademlia 以服务器模式运行、无记录 TTL（链
  而非时间主宰有效性）。

### 记录验证（"链是权威"规则）

被解析或收到的 `SignedDnsRecord` 只有在如下
条件满足时才被接受/返回：

1. 域名在链上已注册；
2. `record.owner` = 当前链上所有者；
3. `record.sequence` = 当前链上序列号；
4. `record_hash(record.record)`（重算 BLAKE3）=
   链上承诺的 `record_hash`。

否则：P2P 侧静默拒绝、CLI 侧显式错误。记录的
Ed25519 签名由中继**在收到时验证**
（`ResolveLocal`）：签名必须对照所有者已学习的
公钥**或**该域名的任何记录委托密钥通过
（KSK/ZSK-lite —— 委托以所有者的名义发布；由
无关密钥签名的记录被拒绝、`record rejected`）；
它也被传输给未来的验证者 ——
链上承诺仍是首要的完整性控制。

## Prometheus 指标

由专职任务服务的 HTTP 端点（Prometheus 文本格式
0.0.4、手写的 exposition —— 无新依赖）：

- 默认在 `127.0.0.1:9101` **公开**（CLI 应用该
  默认；库默认为 `None` —— 单进程中的多个中继
  绝不碰撞）。
- `--metrics-addr ADDR:PORT`：将端点绑定到该地址（如本地 `127.0.0.1:9101`，对外 `0.0.0.0:9101`）。
- `--metrics-token TOKEN`：切换到**私有**模式 ——
  `Authorization: Bearer *** required`（常数时间
  比较、否则 `401`；令牌绝不出现在日志或
  exposition 中）。
- 绑定失败**禁用端点**（非致命警告）—— 绝不
  禁用节点。

生效的绑定在 stdout 上宣告（机器可读契约、如同
`p2p:`）：监听一建立就输出一条
`metrics: <addr>` 行。

**19 个计数器**（`counter`、单调 —— 热路径：
共享 `Arc<Metrics>` 上的 `fetch_add(Relaxed)`、
绝不是共识输入）：

| 组 | 序列 |
|---|---|
| 链 | `scone_blocks_applied_total`、`scone_blocks_rejected_total`、`scone_checkpoints_signed_total`、`scone_checkpoint_sigs_received_total` |
| 内存池 | `scone_txs_admitted_total`、`scone_txs_rejected_total`、`scone_txs_evicted_grace_total`、`scone_txs_included_total` |
| DNS | `scone_dns_queries_total`、`scone_dns_cache_hits_total`、`scone_dns_cache_misses_total`、`scone_dns_local_chain_total`、`scone_dns_dht_answers_total`、`scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`、`scone_dht_negative_hits_total`、`scone_dht_negative_expired_total`、`scone_dht_negative_capped_total`、`scone_dht_cache_evictions_total` |
| 对等网络 | `scone_peer_connections_established_total`、`scone_peer_connections_closed_total`、`scone_peer_connections_incoming_total`、`scone_peer_dial_failures_total`、`scone_peer_blocks_received_total`、`scone_peer_transactions_received_total` |

**6 个 gauge**（`scone_height`、`scone_mempool`、
`scone_peers`、`scone_domains`、
`scone_finalized_epoch`、`scone_addr_book_entries`）
—— 前五个每 15 s 经 `RelayCommand` 通道刷新
（与 CLI `status` 相同的路径 —— 单一事实来源）；
`scone_addr_book_entries` 由注册表直接维护
（每次地址簿插入时更新、事实来源）。

E2E 测试：`metrics_endpoint_public_and_private`
（200 + exposition、无/错令牌 401、带 Bearer
200）。

## CLI 控制（RPC→p2p 统一）

JSON TCP RPC 服务器已被**移除**：CLI（以及注册商
的 axum 桥）**直接**说 p2p 协议 —— 相同线上格式
（`/scone/reqres/1`）、中继侧相同的处理器、零
并行通道。每条命令打开到节点 multiaddr（`--peer`）
的 QUIC 连接、交换**一条** `Message`、关闭。

中继服务的控制消息：

- `GetStatus` → `NodeStatus { network,
  nat_reachable, tip, height, domain_count,
  mempool, peers, finalized_epoch, checkpoints,
  fresh_operators }`（由 `scone status` 显示 ——
  `fresh_operators` = 有新鲜心跳的运营者、RFC
  服务证明）；
- `LookupName { name }` → `NameState`（链上状态、
  如同 `lookup`）；
- `GetOwnedNames { owner }` → `OwnedNames`（某
  `OwnerId` 的存活域名 + TLD）；
- `Transaction (tx)` → 带自由 gossip 的提交
  （中继向对等节点重新广播）；
- `GetRecord { domain_id }` / `Record (r)` —— DHT
  解析与发布（如同 CLI 侧的
  `scone domain resolve`）；
- 应用拒绝 → `Message::Error(msg)`（作为可读
  消息到达客户端、而非 EOF）。

客户端拉起一个最小的 libp2p swarm
（request-response + identify）、30 s 调用超时
（20 s 往返）。默认 `--peer`：
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…`
（本地开发节点；带真实 peer id 的完整
multiaddr 由中继经 stdout 的 `p2p:` 行宣告）。

内部 `RelayCommand` 通道为内嵌 DNS 面保留
（`ResolveLocal`、一次进程内往返、绝非 DHT
等待者 —— 见 `dns.md`）。

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR]
            [--bootstrap MULTIADDR]... [--upnp]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]...
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # express single record
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer defaults to the local relay (SCONE_PEER env overrides);
# --identity defaults to the single keystore identity when there
# is exactly one. Removed (stubs suggest replacements): show,
# submit, lookup, owned.
```

每条控制命令直接对中继的 multiaddr 说 QUIC
（`--peer`、默认本地节点 —— RPC→p2p 统一、不再有
JSON TCP 通道）。中继缺席 → 清晰消息
"p2p node unreachable — is a relay running?"、非零
退出。`--rpc` 标志保留但**过时且被忽略**（RPC
服务器已移除；仅为脚本兼容）。

### 环境配置（`.env`）

启动时、`scone`（经 dotenvy、在 clap 解析之前）
从当前目录加载 `.env` 文件：文件缺失 = 静默
无操作、绝不是错误。加载的值成为声明了变量的
选项（`cli.rs` 中的 `env = "…"`）的**环境基线**；
**显式 CLI 标志总是胜过** `.env`、`.env` 胜过内建
默认值：

```text
explicit CLI flag  >  .env variable  >  built-in default
```

仓库根目录的 `.env.example` 文件记录每个变量
（复制为 `.env` 并调整；`.env` 已被 **gitignore**
—— 绝不提交真实 `.env`）。变量：

| 变量 | 镜像标志 | 语义 |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet`（默认）、`mainnet`、`devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | 数据目录（默认 `~/.scone/<network>/`） |
| `SCONE_LISTEN` | `--listen` | P2P 监听 multiaddr（默认：所有接口、临时端口） |
| `SCONE_BOOTSTRAP` | `--bootstrap` | bootstrap 种子、**可重复**（每值一行 —— dotenvy 累积）；取代内建种子 |
| `SCONE_GENESIS` | `--genesis` | 创世文件（创始人节点） |
| `SCONE_MDNS` | `--mdns` | （已废弃 —— mDNS 发现已移除；被接受、被忽略） |
| `SCONE_UPNP` | `--upnp` | UPnP 端口映射（`true` = 启用） |
| `SCONE_DNS_ADDR` | `--dns` | UDP DNS 服务器绑定地址 |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | 递归回退、**可重复** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | DNS 缓存条目上限 |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | DNS 缓存内存上限（如 `64MB`） |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | 并发 DNS 查询上限 |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | 指标端点 bearer 令牌 |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | 拓扑 profile（`tiny`/`small`/`default`/`high`/`very-high`/`devtest`；默认 `default`） |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | 锚密钥文件（检查点 + 生产） |

`SCONE_ANCHOR_PASS` 不是某个选项的变量：它是
**持有锚口令的变量**的默认名称
（`--anchor-passphrase-env`、CLI 层默认
`SCONE_ANCHOR_PASS`）—— 中继从该标志命名的变量
读取口令。

**锚自动（默认开）**：不带 `--anchor-key` 时、锚
密钥文件在数据目录中自动生成/持久化
（`anchor.sconekey`、若定义了 `SCONE_ANCHOR_PASS`
则用它作口令、否则随机生成并写入
`anchor.pass`）—— 一旦其密钥拥有存活域名/TLD、
中继即成为生产者（PoS 资格从不免费）。
`--no-anchor-auto` 保持中继严格被动。
`--anchor-passphrase-env` 的默认环境变量是
`SCONE_ANCHOR_PASS`。

`--network`：默认跟随**构建**（启用主网的二进制
为 mainnet、否则 testnet）；`devnet` 是可重置的
本地网络（自己的 `scone-devnet` 创世、近零的
0..=2 位 PoW —— 清空 `~/.scone/devnet/` 并重启即
重置链）。设置创世、PoW 难度与默认 data-dir
（`~/.scone/<network>/`）；来自另一网络的交易被
类型化拒绝（`WrongNetwork`）。

离线十六进制命令面（`tx` 与 `submit` 子命令）已
消失（remove-tx-surface-v2）。CLI 覆盖用户路径
（`domain register`、`domain renew`、`domains`）
**以及**作为一等命令的 M8b 家族：`tld open`、
`tld transfer`、`tld revoke`、`domain assign`、
`domain transfer`。`register` 类声明按网络难度
挖取其 PoW（devnet 上毫秒级）。

**不存在任何 geo 或健康选项**：地理区**由构建
强制**（内嵌 DB-IP Lite 表 —— 完整语义见
[`dns.md`](dns.md#geodns-geographic-routing)）；
地理路由无需任何选项**常开**，且解析器不做
任何健康检查 —— 它按原样服务签名记录集
（端点可用性是端点运营者的事）。任何
`--geo-*`/`--health-*` 标志都被 clap 硬性拒绝。

### `domain register|update`（M5）—— 一条命令完成签名交易

这些命令在单次调用中串联 构建 → 签名
（keystore）→ 提交 →（确认）。

- **register**：构建已签名的 `RegisterDomain`
  （timestamp = now、devnet 下空证明）、提交、
  等待链上确认（序列 ≥ 0、30 s 超时）。已注册
  的名称在客户端**和**中继侧都被拒绝。
- **update**：记录文件是唯一事实来源。序列号
  （链上 + 1）与 `record_hash`（文件记录的规范
  BLAKE3）被**派生**、绝不手工输入：签名者与
  验证者不可能分歧。UpdateDomain 确认之后、
  `SignedDnsRecord`（来自文件的记录 + 所有者 +
  对规范编码的 Ed25519 签名）经 `put_record`
  发布到 DHT（它会对照链重新验证）。未注册的
  域名、非所有者身份或文件错误干净地失败、
  不触碰链。

#### 记录文件

文本、每行一条记录、`#` = 注释、空行忽略；
排序的规范编码使文件顺序对被承诺的哈希没有
影响。文件是完整集合（更新是替换、不是合
并）。类型：

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

类型化错误：未知类型、无效 IP/名称（带行号）、
空集合、重复、超过 `MAX_RECORDS_PER_SET`。

## `--genesis`（创始人）

`scone relay --genesis genesis.bin` —— **仅创始人
节点**（kickstarter 活动；文件由 `scone-genesis`
二进制产生）。规范性规则：

1. 文件超过 1 MiB 在**读取之前**被拒绝、然后
   被严格解码（`decode_complete`：无尾部字节）。
   区块 0 的哈希被**重算**、绝不从文件读取。
2. 语义验证：高度 0、`consensus` 以所请求网络
   （`--network`）的 network id 开头、无交易、
   规范的分配清单。任何违反 = 启动时类型化
   错误（绝不 panic、守护进程不启动）。
3. **处女** data-dir：链诞生在被分配的创世上 ——
   分配在任何磁盘写入**之前**在 RAM 中应用、
   然后被持久化（种子状态 + meta 键下的创世
   区块）。因此第一个产出的区块只能扩展被
   分配的创世。
4. **已投产**的 data-dir：存储的诞生创世是
   区块 1 的 `prev_hash`（自证）。相等的文件
   哈希 = 静默 OK（该选项**在首次启动后是多余
   的**）；不同 = 类型化拒绝 —— 节点经同步
   加入网络、绝不经 `--genesis`。
5. **不带**该选项重启：创始人诞生的 data-dir
   经 meta 键恢复其被分配的创世（每次启动
   RAM 重绑定）。

固定的创世哈希 pin（防网络分歧）是独立的
工作流。

### `scone-genesis` 二进制清单（kickstarter）

`scone-genesis` 二进制（crate `scone-genesis`）
准备 `genesis-manifest.json` 清单（`ALLOC-V1`
格式）然后产出 `genesis.bin`。规范性规则：

- `scone-genesis new --network <network>` 创建空
  清单；`add`/`remove`/`list` 管理分配、`build`
  产出 `genesis.bin`（+ 哈希）、`verify` 重新
  验证它。
- `add domain <name>` 要求该域名的 **TLD 已经
  在清单中被分配**（否则立即拒绝 —— 链在创世
  应用时会拒绝它）；其 `valid_until` 是**自动
  的**：`born_at + 1 年`（清单的
  default_validity）、绝不手工输入。
- `add tld <name>`：**默认开放**（无主、自助、
  1 年保底、线上 owner = 零 —— kind `0x03`）；
  `--private` = 私有（所有者、创世期限
  `born_at + TLD_TERM_SECS` —— 3 年、标准生命
  周期：`RenewTld`、60 天宽限）。
- 按名称规范排序（相等内容 → 相同区块 → 相同
  哈希、与插入顺序无关）；拒绝重复。

## 限制与内存保证

| 资源 | 边界 |
|---|---|
| P2P 帧 | 解析前 `MAX_MESSAGE_LEN`（1 MiB） |
| 内存池 | 4096 tx（超出类型化拒绝）；每域名 `MAX_PENDING_PER_DOMAIN = 4` 笔待定 tx |
| 内存池宽限 | `MAX_GRACE_ATTEMPTS = 10` 次因非终局原因被冲刷的 tx 重新插入（按 40 s/周期约 7 分钟）、然后驱逐 |
| 分叉停泊 | `PARKED_CAP = 2048` 个竞争区块 |
| 累积检查点 | `MAX_PENDING_CHECKPOINTS = 256`（驱逐：最小数据哈希） |
| 同时 DHT 等待者 | `MAX_DHT_WAITERS`（256）；DHT 查找 30 秒有界（`DHT_LOOKUP_TIMEOUT`） |
| 每同步回复区块数 | 每往返 1 个；每对等节点 ≤ 10,000 批次（`MAX_SYNC_ROUNDS`）；请求上的协议边界 `MAX_BLOCKS_PER_REQUEST = 128` |
| DHT 记录 | `MAX_DHT_CACHE_ENTRY`（64 KiB） |
| 持久 DHT 缓存（磁盘） | 双重 LRU 边界：`DHT_CACHE_MAX_ENTRIES`（100,000 条）与 `DHT_CACHE_MAX_BYTES`（2 GiB）—— 先到者按键序驱逐；GC 驱逐在与区块相同的事务中（被链 GC 杀死的域名绝不留下孤儿条目） |
| DHT 负缓存 | `DHT_NEGATIVE_TTL_SECS`（30 s）、FIFO 上限 `DHT_NEGATIVE_CAP`（4,096） |
| RAM 中的链 | 创世 + tip 窗口（历史由存储提供） |

防循环规则（H2）：交易只有在其被**实际插入**
内存池时（首次见到）才被重新广播；重复项
（已在池中或已在链中）绝不被重新中继。已规范
化的区块在 `push_block` 之前被检测并忽略（无
重新存储、无重新广播）。

未决事项：中继的持久网络身份、广播 gossip
（当前：向每个已知对等节点的单播
request-response）。自初稿以来已解决：真实的
分叉选择（停泊 + reconsider_parked、见上文）、
PoW 共识（签名出块者 + 锚委员会）、DNS 解析器
（`--dns` 面、见 dns.md）、控制通道（RPC→p2p
统一：不再有 JSON TCP）。

## 优雅停机（SIGTERM / SIGINT）

`scone relay` 守护进程（前台）经 `tokio::signal`
捕获 SIGTERM 与 SIGINT：收到信号时、
`Relay::run` future 被取消（二进制中的
`tokio::select!`）、从而 drop libp2p Swarm 并关闭
QUIC 套接字。tokio 运行时随后以有界停机
（≤ 2 s）停止、drop 分离的 DNS 任务（UDP/TCP）
并关闭其套接字 —— 目标是一次 `kill` 不留下
孤儿 UDP 套接字（测试网 60 上观察到的案例）。
信号驱动的停机记录
`relay shutting down (signal)`（INFO、stderr）并以
退出码 **0** 退出：被请求的停机不是失败。若
处理器安装失败、信号的默认行为生效（降级、
绝不 panic）。


## Tor 传输（未决问题）

QUIC/UDP 不能穿透普通 SOCKS 代理、因此中继
今天无法直接在 Tor 上运行。原生 libp2p Tor
传输（洋葱地址的中继）是被跟踪的前进路径 ——
见网站的抗审查指南了解当前状态与 `.tor`
CNAME 桥（隐藏服务的可读名称、今天已支持）。

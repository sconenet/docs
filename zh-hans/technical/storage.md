# 存储 —— 节点本地持久化（redb）

> 规范性文档。本文档与
> `crates/scone-storage/src/` 之间的任何分歧都是
> bug（确定事实来源并修复）。实现参考：crate
> `scone-storage`（格式 v1）、`redb` 4.2 后端。

## 目的

节点的嵌入式本地持久化：规范区块、域名
状态、DHT 缓存、元数据。存储是**持久层**，绝不是
权威：RAM 状态（[`ChainState`]、共识的事实
来源）在启动时从存储重建，每次
写入都是**增量**（被一个区块修改的域名），绝不是
状态的完整拷贝。

横切约束（面向用户、每个里程碑）：**最小
内存占用**。大容量数据（区块、域名状态、DHT
缓存）位于 redb 中并经游标/范围访问 —— 绝不
全量载入 RAM、绝无用重复。

## 边界：`NodeStore` trait

`crates/scone-storage/src/lib.rs` 定义对其余代码
（区块链、未来的 M4 中继）可见的唯一存储面：

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` —— 原子
  追加（见下文）；
- `block_at_height(h)` / `block_by_hash(hash)` ——
  点读；
- `tip() -> (height, hash)` —— O(1)，从 `meta`
  读取；
- `put_domain_state` / `domain_state` /
  `domain_count` —— **被维护的**计数器
  （`meta["domain_count"]`），绝不用
  `COUNT(*)`；
- `iterate_domains(after, max) -> DomainPage` ——
  基于游标的分页（默认每批 100）、`DomainId`
  的字节升序；
- `put_tld_state` / `tld_state` / `tld_count` /
  `iterate_tlds` —— TLD 注册表的相同契约
  （`tld_count` 被维护、游标有界）（M7d）；
- `put_dht_cache` / `dht_cache` —— 不透明的
  `SignedDnsRecord` 字节；
- `resolve_name(canonical)` / `resolve_tld(tld)`
  —— 持久的 Name→Id 索引（P0.2）、O(1) 点查、
  不迭代域名（默认 `None` = 无索引的后端、调用
  者回退到 `iterate_domains` 或链状态）；
- `tld_floors_all` / `snapshot_tld_floors_all` ——
  读取开放 TLD 的寿命保底（所有者选项：专用
  表、活跃的 `tld_floors_v1` 与冻结的
  `snapshot_v1_floors`）；
- `rewind_to_height(h)` —— 重组：在一个事务中
  移除 `h` 之上的区块并把 `tip`/`tip_height`
  带回 `h`（`persist_state_snapshot` 的前置、
  见下文）；
- `meta_get` / `meta_set`；
- `snapshot_meta` / `snapshot_domains` /
  `snapshot_tlds` —— 启动状态快照读取
  （M6b；空默认值 = 无快照的后端、
  `load_chain` 回退到全量加载）。

规则：

- **API 中不出现 redb 类型**：只用字节
  （`&[u8]`、`[u8; 32]`）与核心类型。后端可
  替换。
- **无异步、无内部可变性**：写取 `&mut self`
  （redb 串行化写者）、读取 `&self`（每次调用
  一个短读事务）。
- 该 API 绝不重新验证区块：传给
  `append_block*` 的哈希是调用者
  （`Blockchain::push_block`）**重新计算**后
  提供的。

## redb 表结构

| 表 | 键（排序） | 值 |
|---|---|---|
| `blocks_by_height` | `u64` 高度（有序 b-tree） | `hash(32) ‖ height(8 BE) ‖ canonical block` |
| `blocks_by_hash` | 区块哈希（32 B） | 区块的规范编码 |
| `domains` | `DomainId`（32 B、字节序） | `DomainStateBytes`（见下文） |
| `tlds` | `TldId`（32 B、字节序） | `TldStateBytes`（见下文、M7d） |
| `dht_cache` | `DomainId`（32 B） | `SignedDnsRecord` 字节（不透明） |
| `names_v3`（P0.2） | 索引键（32 B、字节序） | 目标 id（32 B） |
| `name_reverse_v3`（P0.2） | 目标 id（32 B、字节序） | `tag(1) ‖ canonical name` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta`（M6b） | `u64` = 0（单槽） | `height(8 BE) ‖ tip_hash(32)` = 恰好 40 B |
| `snapshot_v1_domains`（M6b） | `DomainId`（32 B、字节序） | `DomainStateBytes`（与 `domains` 相同格式） |
| `snapshot_v1_tlds`（M6b） | `TldId`（32 B、字节序） | `TldStateBytes`（与 `tlds` 相同格式） |
| `tld_floors_v1`（所有者选项） | `TldId`（32 B、字节序） | `u64`（到期保底、redb 隐式 BE） |
| `snapshot_v1_floors`（M6b） | `TldId`（32 B、字节序） | `u64` —— 保底的冻结拷贝（快照 AT 状态、而非 tip） |

额外的 `meta` 键（P0.3/P0.4）：

| 键 | 值 |
|---|---|
| `verified_snapshot_v1`（P0.3） | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B —— 已验证导入快照的标记（保留、如同 `tip`） |

`meta` 键：

| 键 | 值 |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION`（1） |
| `tip` | tip 区块的哈希（32 B） |
| `tip_height` | `u64 BE`（使 `tip()` 为 O(1)） |
| `domain_count` | `u64 BE`（被维护的计数器） |
| `tld_count` | `u64 BE`（被维护的计数器、M7d） |

`blocks_by_height` 的 `hash ‖ height` 头使每个索引
自含（无需读区块即可验证）；两个索引在同一
事务中更新。

### 创世约定

创世区块是确定性的（协议常量）且**绝不
按字节存储**。全新存储的 `tip == (0,
genesis_hash)`；第一个被追加的区块高度为 1。
创世哈希总是重新派生
（`scone_blockchain::genesis_hash()`），绝不读取。

## `DomainStateBytes` 编码格式（M8b）

依据前导版本标签为 65 或 97 字节、定宽
大端整数（规范、可逐字节比较）：

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) ‖ mutations(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash present → exactly 97 bytes
tag = 0x02 : record_hash absent  → exactly 65 bytes
```

`mutations` 是命名空间的单调变异计数器（CAS
防重放世代 —— 见 `transactions.md`，变异计数器）。
M8b 编码（57/89 字节、无计数器）是
**非法**的：严格解码 → `Corrupted`（测试网
重置；计数器前的存储经重放再生成）。

- `registered_at` / `valid_until`（M8b）：注册的
  过期（声明时 1 年、可续期、见
  `/docs/en/technical/blockchain.md`）。

M7 编码（41/73 字节、无过期）自 M8b 起
**非法**：严格解码 → `Corrupted`。（M7 存储
经重放再生成：`load_chain_replay`。）

- `owner`：`OwnerId`（32 B）；
- `sequence`：最后一个被应用的 `UpdateDomain`
  序列号（`RegisterDomain` 之后立即为 0）；
- `record_hash`：DNS 记录集当前的链上承诺
  （只要没有 `UpdateDomain` 被应用则为
  `None`）。

**严格**解码（`DomainStateBytes::decode`）：未知
标签、错误长度或多余字节 → 
`StorageError::Corrupted`，绝不 panic。

## `TldStateBytes` 编码格式（所有者选项）

长度依枚举变体而定 —— **私有** TLD 为 50 字节、
**开放** TLD 为 10 字节（既无序列号也无记录
哈希）：

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
                [‖ mutations(8)]
```

- `flag`（所有者选项、`TldState` 枚举）：`0x00`
  = `TldState::Open`（无主 —— 自助 + PoW、无
  所有者字节）、`0x01` = `TldState::Private`
  （后跟 32 个所有者字节 + 8 个计数器字节）。
  严格规范 ——
  任何其他字节 → `Corrupted`。不变量"开放 ⟺
  无所有者"是结构性的：两种状态不可能被
  混淆。
- **`until`（8 字节、BE64）**：TLD 自己的
  时钟 —— 私有期限终点（声明/续期 + 3 年）
  或开放死亡时刻表（出生时 1 年保底、被每个
  存活域名推迟）。它重新成为条目的一个
  属性（TLD 生命周期变更）。派生的
  `tld_floors_v1` 表（+ 其冻结拷贝
  `snapshot_v1_floors`）仍然跟踪保底、在重放时
  重建并在启动时恢复 —— 重启的节点杀死的
  TLD 与连续运行的节点相同。

M7d（33 字节、标签 `0x01`）、M8b（34 字节、
标签 `0x02`、布尔 `open`）、旧标签 `0x03`
（内嵌 `until`）以及计数器前的标签 `0x04`
布局（42/2 字节、无计数器）的编码都是**非法**的：
严格
解码 → `Corrupted`（主网前、经重放再生成即
迁移路径）。

标签只为格式演进而存在：未来的布局递增
标签、旧读取者以 `Corrupted` 失败而不是去
猜测。**严格**解码（`TldStateBytes::decode`）：
未知标签、错误长度、多余字节或非规范
标志 → `StorageError::Corrupted`，绝不 panic。

## 原子性保证

`append_block_with_state` 在**单个** redb
`WriteTransaction` 中写入：

1. 区块到 `blocks_by_height`（前置检查：该
   高度不得存在内容不同的区块、否则
   `StorageError::Conflict`）；
2. 区块到 `blocks_by_hash`；
3. 每个域名状态增量（`domains`）；
4. `domain_count` 计数器（仅按**新**域名的数量
   递增；已存在域名的 `UpdateDomain` 不使它
   递增）；
5. 每个 TLD 注册表增量（`tlds`、M8b：声明、
   转移、开放/封闭）—— **在同一事务中**：
   写了区块却没写其 TLD 变更在结构上不可
   能；
6. `tld_count` 计数器（仅新 TLD）；
7. 每项**移除**（M8b）：被确定性 GC 过期的
   域名（`removed_domains`）与被放弃的 TLD
   （`removed_tlds`）在同一事务中离开存储、
   计数器递减 —— 重启绝不能复活一个已过期
   的注册；
8. P0.2 的**名称索引**增量（upsert 与移除、
   见下文）—— 相同保证：崩溃绝不留下指向
   不存在域名的索引条目；
9. `tip` 与 `tip_height`。

redb 以 WAL 方式提交：写中途崩溃留下**之前
的**一致状态。因此**没有状态的区块写入在
结构上不可能** —— 这是该里程碑的关键
不变量。

额外保证：

- **单调性**：在 ≠ `tip_height + 1` 的高度追加
  → `StorageError::NonMonotonicHeight`、什么也
  不写（在打开写事务*之前*检查）；
- **幂等性**：以相同 `(height, hash)` 对重新
  追加当前 tip 区块是无操作（快速路径比较
  高度与哈希；高度相同时、相同的区块字节
  也是静默无操作、不同字节是 `Conflict`）；
- **损坏的数据库**：截断/被污染的文件 →
  打开或读取时类型化错误
  （`StorageError::Database`/`Corrupted`）、绝不
  panic（已测试）；
- **格式版本**：`format_version` ≠ 1 →
  `StorageError::UnsupportedFormat`（需要显式
  迁移路径、绝不猜测）。

## 区块链集成（`integration.rs`）

角色分离：RAM 中的 [`ChainState`] = 共识权威；
存储 = 持久层。所选策略（有据可查的最简
策略）：**在每个区块持久化状态** —— 
`store_block` 写入区块 + 被**该区块触及**的
域名状态（增量）。全量重放仍然可能并作为
控制/修复路径。

- `store_block(store, chain, block, hash)`：规范
  编码区块、从 RAM 状态（已被 `push_block`
  更新）读取被触及域名与被声明 TLD 的最终
  状态、原子追加。
- `load_chain(store)`：**不重放** —— 读取 tip +
  tip 区块（O(1) 区块读取）、以每页 100
  （`DOMAIN_PAGE`）分页恢复域名状态**并以
  每页 100 恢复 TLD 注册表**（`TLD_PAGE`、
  M7d —— 关闭重启窗口：没有它、重启的
  节点会忘记每个已声明的 TLD、并以
  `UnknownTld` 拒绝该 TLD 下的每个
  `RegisterDomain`、尽管其磁盘持有完整
  状态）。RAM 链只保留创世 + tip；历史区块
  留在存储中并从中提供。自 M8 起
  `load_chain` 随后重建 **TXID 防重放索引**
  （见下文）。
- `load_chain_replay(store)`：逐个重放所有已
  存储区块（绝不全在 RAM 中）、完整验证；
  得到的 tip 必须等于存储的 tip、否则
  `Corrupted`。
- `rebuild_replay_index_from_store(store, chain,
  tip_height)`（M8）：重扫最近
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` 个
  持久化区块（一次一个区块在 RAM 中）并经
  公共 API `Blockchain::rebuild_replay_index`
  把每个 TxId 重新插入链的索引**到其原始
  包含高度**（不触碰私有字段）。tip 之下
  缺失区块或窗口内不可解码 → `Corrupted`
  （那些字节曾用于构建链：此处的失败表明
  存储已损坏）。
- `touched_domains(block)`：被区块交易修改的
  域名 id、去重、区块顺序。
- `touched_tlds(block)`：被区块改变状态的 TLD
  id（`RegisterTld`、`TransferTld`、`RevokeTld`、
  `SetTldOpen`）、去重、区块顺序（M8b）。
- `store_block_with_removals(...)`：
  `store_block` 的变体、额外接收被区块 GC
  移除的域名（M8b）以进行上述原子写。中继
  从 `Blockchain::push_block_with_gc`（应用
  结果暴露 `gc_removed_domains`）取得该
  列表。**DHT 缓存的 GC 驱逐**：被移除域名
  的 `dht_cache` 条目在**同一**事务中与区块
  一起删除（追加增量携带 `dht_cache` 表）——
  DHT 记录绝不超过它所服务的域名的寿命、
  过期条目绝不永久堆积（测试：
  `gc_evicts_the_dht_cache_entry`）。

中继（M4）将使用：启动时 `load_chain`、之后
对每个被接受的区块 `push_block`（RAM）再
`store_block`（磁盘）。

## 重组持久化（`rewind_to_height` + `persist_state_snapshot`）

重组（切换到更强的姊妹分支）无法通过单调
追加进行 —— 写入获胜后缀会以
`NonMonotonicHeight` 失败。专用路径
（粘性少数派修复、中继 + 存储）：

1. `rewind_to_height(fork_height)` —— 一个
   redb 事务移除分叉点之上的区块并把
   `tip`/`tip_height` 带回（名称索引与活跃
   状态在下一步修正）；
2. `persist_state_snapshot(store, chain)` ——
   从重组后 RAM 状态导出重写活跃表
   （`domains`、`tlds`、名称索引、TLD 保底）：
   活跃表在启动时绝描述**输掉**的分支；
3. 被采纳后缀的单调重新追加（普通增量）。

此路径上的持久化失败被记录并降级（存储
在启动时经重放愈合）—— 绝不致命。

## 启动状态快照（M6b）

如上所述的 `load_chain` 仍是 O(状态大小)：
不重放区块、但整个 `domains` 表被重读。是
每区块持久化状态的路径使重放变得不必
要。M6b 快照针对另一种成本：RAM 状态必须
**经重放**重建的场景（修复、控制、或没有
按区块持久化状态的后端）。它提供
**O(tip − H)** 的启动 —— 只重放快照之上的
后缀。

### 写入

- **间隔**：`SNAPSHOT_INTERVAL = 64`
  （`scone-storage` 的公共常量）。触发点是
  区块 `k·SNAPSHOT_INTERVAL + 1` 的追加：该
  事务把活跃表状态冻结为**应用该区块之前
  的样子**、即区块 `k·SNAPSHOT_INTERVAL` 之后
  的规范状态。
- **绝不在 tip**：快照因此最近位于
  `tip − 1`、写入时永远严格低于 tip。紧接着
  的崩溃绝不能留下"高于"已持久化链的
  快照。
- **原子**：冻结（清空 + 把 `domains`/`tlds`
  表拷贝到 `snapshot_v1_*`、写
  `snapshot_v1_meta`）发生在与触发区块追加
  **同一个** redb 事务中 —— 崩溃留下的要么
  是完好的旧快照、要么是随其区块完整的新
  快照（redb WAL）。
- **单槽**：每个快照**替换**前一个（表清空
  后重拷）—— 无累积、无需管理修剪、磁盘
  占用以约 2× 状态为界。
- 冻结状态的格式严格是
  `DomainStateBytes`/`TldStateBytes` 的（复用、
  无新编码）；只有 `snapshot_v1_meta` 行是新
  格式（`height(8 BE) ‖ tip_hash(32)`、严格
  解码：任何其他长度 → `Corrupted`）。

### 启动时读取（`load_chain`）

1. tip 一如既往被验证（从 tip 区块头部重算
   哈希对照 `meta["tip"]`）。
2. 若 `snapshot_meta()` 存在、其高度 `H` 满足
   `0 < H < tip_height`、且存储在 `H` 处区块
   的**重算**哈希等于快照的 `tip_hash`
   （**锚** —— 与已持久化链不相干的、被篡改
   的、或来自另一文件的快照在此被发现），
   冻结状态以每页 100 恢复、然后只重放区块
   `H+1..=tip`（完整验证、与活跃区块
   相同）。
3. 重放到达的 tip 必须等于存储的 tip。
4. **此路径上的任何失败（快照缺失、高度不可
   用、锚无效、字节不可解码、重放失败）静默
   降级到 M6b 之前的全量加载**（读活跃表、
   无重放）—— 坏快照绝不使启动失败。向后
   兼容是"无快照"这一特例：M6b 之前写入的
   存储（没有 `snapshot_v1_*` 表）加载方式
   相同、`snapshot_meta()` 为 `None`。

被重放的后缀以 `SNAPSHOT_INTERVAL` 为界
（触发时至多 64 个区块 + 此后挖出的区块）：
启动为 O(tip − H) 且 H 保证是近期的。

## 启动时重建 TXID 防重放索引（M8）

防重放索引（`included`、TxId → 包含高度、
窗口 `REPLAY_WINDOW_BLOCKS = 256` —— 见
`/docs/en/technical/blockchain.md`）是**规范链的
纯函数**：它绝不持久化。由于 `load_chain`
不重放历史，M8 之前恢复的链以**空**索引
启动 —— 重启的节点接受任何仍在重放窗口内
的交易的重新包含（已记录的限制；状态规则
仍是唯一防线）。

M8 关闭了这一窗口：恢复之后（全量路径或
快照 + 后缀重放），`load_chain` 调用
`rebuild_replay_index_from_store`，后者重读
最近 `min(REPLAY_WINDOW_BLOCKS, tip_height)`
个持久化区块 —— 一次一个区块在 RAM 中、
一如既往地内存有界 —— 并把每个 TxId 重新
插入到其**原始高度**。少于 256 个区块的
存储因此重建其全部索引。

属性：

- **与在线节点决策相同**：相同条目、相同
  修剪规则（`h > tip − 256`）—— 重启的节点
  与从未停止的节点拒绝相同的重放（已测试：
  精确索引计数且与修剪后的活跃索引相等）。
- **修剪语义不变**：窗口之外的旧交易仍被
  索引接纳（之后若适用则因状态规则失败）——
  与之前完全一样。
- **快照不冻结索引**：快照的后缀重放重建
  H 之上的条目、M8 重扫补上 H 之下的窗口
  部分（与 owner_keys/grace 相同的契约：
  重建、绝不持久化）。
- **成本**：启动时 ≤ 256 次区块点读、
  O(window) —— 与链龄无关。
- **窗口内**缺失或不可解码的区块使启动以
  `Corrupted` 失败（那些字节曾用于构建
  链）；窗口之外的历史区块不受影响。

`load_chain_replay`（零信任）通过其全量重放
以相同方式重建索引 —— 两条路径收敛。

## 持久 Name → Id 索引（P0.2）

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ canonical name)`
不可逆：解析名称（中继/DNS）需要持久
索引。P0.2 把它加入存储、与其余部分保持
相同的事务纪律。

### 结构

两张伴生表、一起写入与擦除：

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ canonical name)  (32 B)
                val  = target id (32 B : DomainId or TldId)

name_reverse_v3 key  = target id (32 B)
                val  = tag(0x01 domain | 0x02 TLD) ‖ canonical name
```

- `names_v3` 的键是**名称的哈希、而非明文
  名称**（紧凑的索引表、不经 dump 泄露名称
  gossip）。键是**名称的纯函数**：索引随时可
  从链重建（`rebuild_name_index`）、不为其
  持久化任何额外元数据。
- `name_reverse_v3` 存储名称（恰好一次）、
  因为移除路径（过期域名的 GC、`RevokeTld`）
  只知道 **id**：要擦除的哈希键总得有办法
  找到。标签区分两个命名空间。
- 严格分离：`resolve_name` 绝不回答 TLD 条
  目、`resolve_tld` 绝不回答域名条目（由反向
  表的标签决定；`DomainName` 有 ≥ 2 个标签、
  TLD 有 1 个 —— 两种键派生共享 `names_v3`
  空间但不共享结果）。
- `DomainId` 保持 32 字节、绝不缩减（项目
  硬规则：共识身份、抗碰撞）。

### 事务一致性

upsert（`RegisterDomain`、`AssignDomain`、
`RegisterTld` —— 名称由交易以明文携带）与
移除（GC、放弃 —— 裸 id）由区块的
`StateDelta`（`name_upserts` / `name_removals`）
携带、并在与区块及状态写入**同一个** redb
事务中应用。因此崩溃既不能留下指向不存在
域名的索引条目（无幽灵）、也不能留下没有
条目的已注册域名。失败的
追加什么也不写（已测试）。命名变体
`put_domain_state_with_name` /
`put_tld_state_with_name` / `remove_domain_state`
为修复工具提供相同的原子性。

`store_block` / `store_block_with_removals` 从
区块的**同一批**交易派生名称增量 —— 中继
接口无需额外参数。

### 重建（`RedbStore::rebuild_name_index`）

索引是规范链的纯函数、如同 M8 防重放
索引。`rebuild_name_index` 从**已持久化的
区块**重建它（`DomainStateBytes` 不含名称
—— 只有 owner/sequence/expiration —— 因此
事实来源是交易流、其中 `RegisterDomain` 与
`AssignDomain` 以明文携带规范名称）：

1. 第一遍：扫描 `blocks_by_height` 创世→tip、
   一次一个区块在 RAM 中；收集在
   `RegisterDomain`/`AssignDomain`/`RegisterTld`
   中见到的 `name → id` 绑定；
2. 第二遍：只过滤到**存活的**目标（持久化
   状态是权威的）—— 注册后又被 GC/过期的
   域名不被重新索引；
3. 第三遍：在一个事务中擦除 + 全量重写两
   张表（崩溃 → 旧的或新的完整索引）。

缺失或不可解码的区块 → `Corrupted`（与
构建链的相同字节）。成本：RAM 中 O(见到的
不同名称)（每名称 ≤ 253 B）、读取 O(区块) ——
这是修复工具、不是启动路径。

### 该索引不是什么

- 不是权威：RAM 链状态仍是唯一权威；索引
  是可重建的缓存（分歧是事务性 bug、可由
  `rebuild_name_index` 与其对照发现）。
- 不冻结在 M6b 快照中：`names_v3`/
  `name_reverse_v3` 表是活跃的、跟随每个区块
  的增量；状态快照不内嵌它们（恢复时经
  增量重建、或以 `rebuild_name_index` 修复）。

### 快照不承诺什么

RAM 状态 `ChainState` 携带未被共识承诺的
内部结构（所有者密钥、宽限窗口）。它们
**不**被冻结：后缀重放通过应用交易重建
它们（确定性、相关窗口内与全量重放内容
相同）。TXID 防重放索引遵循相同契约
（M8、见上一节）：启动时重建、绝不冻结。
与 `load_chain_replay` 的逐位相等已由测试
在规范状态上验证（state_root + SMT）。

## 已验证（VERIFIED）状态快照 —— 分块 + 清单 + 密码学验证（P0.3）

上面的 M6b 快照加速信任自己磁盘的节点的
启动。P0.3 针对不同的需求：在节点之间传输
状态（引导新节点、修复）而不信任传输。
格式与验证位于
`crates/scone-storage/src/verified_snapshot.rs` +
根重建位于
`crates/scone-blockchain/src/snapshot_verify.rs`。

### 信任模型

快照携带一个**已终局**区块的状态 —— 由
委员会法定人数签名的检查点
（`CheckpointData.state_root`、O(1) 的
`state_root_smt` 根）。检查点的真实性
（Ed25519 签名法定人数）由链层在导入
**之前**验证。导入本身通过重算验证快照 ↔
检查点的一致性：

1. **每一页的哈希**：`blake3("SCONE-SNAP-PAGE-HASH" ‖
   完整页编码)` 必须等于清单宣告的哈希
   （一个伪造字节 → 拒绝）；
2. **条目数之和 == 清单计数器**（缺页或
   错误计数器 → 拒绝）；
3. **从导入条目重算的 SMT 根 ==
   `checkpoint.data.state_root`** —— 那项
   密码学检查：整个状态（域名 + TLD）被
   重新折叠进一个全新的 SMT
   （`snapshot_verify::recompute_state_root`、
   叶 `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1`
   与活跃路径完全一致）且根必须等于委员会
   签署的根。伪造、截断或不完整的快照
   在任何写入**之前**被拒绝；
4. **结构一致性**：清单的
   `height`/`tip_hash`/`state_root` == 检查点的；
   严格递增的 id（无重复）；连续的页索引；
   DoS 边界（≤ 10,000 条/页、≤ 2^20 页、总
   预算）。

验证失败 → `StorageError::SnapshotRejected`、
存储保持严格不变（一切都在写事务之前
验证）。

### 序列化格式（严格、大端、毫不留情的解码）

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← every page is
        ‖ entry_count(4) ‖ entries                   self-identifying

entry   = kind(0x01 domain | 0x02 TLD | 0x03 TLD floor) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B or TldStateBytes 2/34 B —
           the existing canonical storage formats, reused; kind
           0x03 (lifetime floor of an open TLD, owner-option) has
           a dedicated format WITHOUT a length prefix:
           id(32) ‖ floor(8 BE), fixed size)
```

页顺序：所有域名条目（id 递增）、然后所有
TLD 条目（id 递增）、然后冻结在快照高度的
TLD 寿命保底（`snapshot_tld_floors_all` ——
而非活跃表）—— 存储游标分页的顺序。
每一页携带 `(index, height, 检查点的
state_root, entries)`：单独一页知道自己属于
哪个快照。导出时每页 256 条
（`VERIFIED_SNAPSHOT_PAGE_ENTRIES`）；导入时
DoS 边界。TLD 保底（`tld_floors_v1`）是可移植
格式的一部分：kind 0x03 携带它们、且导入
要求它们严格递增并指向快照中**存在的**
TLD —— 否则导入者永远不会杀死导出者会
杀死的东西（静默分叉、被拒绝）。

### 导出（`export_verified_snapshot(store, checkpoint)`）

序列化**最后一个已终局检查点**的状态 ——
而非 tip。先输出域名、然后 TLD、再然后冻结
在快照高度的 TLD 寿命保底（导出/导入对称：
导出历史上遗漏了保底 —— 已修复、回归测试
`export_carries_tld_lifetime_floors`）。前提：
存储在检查点高度恰好持有 M6b 持久化快照
（`snapshot_v1_*`）（`CHECKPOINT_KEEP`/
`SNAPSHOT_INTERVAL` 窗口保证这样的状态
定期可得）；区块的锚从存储字节**重算**且
必须等于 `checkpoint.data.block_hash`。否则 →
`StorageError::SnapshotUnavailable`（导出是
显式动作：等待下一个间隔/检查点）。

### 导入（`import_verified_snapshot(store, checkpoint, manifest, pages)`）

验证一切（见上）、然后在一个 redb 事务中
写入：活跃表 `domains`/`tlds` + 计数器 +
`tip = (H, checkpoint hash)`、冻结表
`snapshot_v1_*` 锚定在同一区块、以及保留
标记 `meta["verified_snapshot_v1"]`
（`height ‖ tip_hash ‖ manifest_hash`）。目标：
仅限空存储（否则 `SnapshotUnavailable`）。

### 验证成本（已记录）

经有序插入的 SMT 重建：N 条 × O(40) 哈希
≈ O(N·40)。100,000 个域名约 210 万次 blake3
哈希（几十毫秒）。一次性 —— 引导/修复导入、
绝不是按区块路径：可接受。有序插入（id
递增）只创建每个分支点一次；无论如何根
只取决于内容（SMT 保证、`smt.rs` 中的差分
模糊测试）。

## 状态引导 —— 无创世重放（P0.4）

新节点不重放自创世的历史而加入网络：

```text
finalized checkpoint (relay, validated by the chain layer)
        │
        ▼
verified snapshot (manifest + pages, crypto verification above)
        │  import → state at H + tip (H, hash) + marker
        ▼
anchor block H (requested from the network BY HASH — append_bootstrap_anchor)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height, ordinary atomic
        │               append — the storage API requires only this)
        ▼
synced — boot replays ONLY the suffix above H
```

- **暴露给中继的存储 API**：
  `append_bootstrap_anchor(H, hash, bytes)`
  （铺设锚区块：必需的标记、与标记的高度/
  哈希一致性、幂等）然后按递增高度的普通
  原子追加。（按哈希的）网络检索是中继的
  事务、不是存储的。
- **`load_chain`** 识别该标记：H 之下的历史
  合法缺失（非归档节点）。具体地：M8 防重放
  窗口重扫以 H 为下限（H 之下的区块不存
  在也不缺失）；导入之后（锚尚未铺设）无
  已存储区块的 tip → 类型化诊断
  `SnapshotUnavailable`（"锚必须来自网络"）、
  不是损坏。
- **归档节点 vs 全状态节点**：归档节点
  （运营者标志）保留一切 —— 向网络提供
  历史区块并经现有路径启动。全状态节点
  （以此方式配置的新节点的 P0.4 默认）在 H
  启动：验证未来的区块（后缀重放）但不能
  提供 H 之下的历史或重放创世。**默认行为
  不变**：只有运营者显式触发导入时该路径
  才活跃（`import_verified_snapshot` 是专门
  动作、绝不自动）；每个现有节点启动方式
  相同。
- **与全节点的相等性**：加入后缀 H+1..H+k
  之后、状态、tip 与 `state_root_smt` 与同一
  tip 处的全节点逐位一致（已测试）；防重放
  索引恰好覆盖 H 及以上的交易 —— 非归档
  节点无法知道 H 之下的窗口、状态规则仍是
  防线（H 之下的重复交易因状态失败、绝不
  被接受两次）。

## 内存策略

- **处处游标**：`iterate_domains` 经 redb 的
  惰性 `range` 把页限制到 `max` 条（默认
  100）；整个域名表绝不被物化。大列表
  （TLD、域名）强制走此游标
  （`iterate_tlds`、相同契约、M7d）。
- **只有增量**：`append_block_with_state` 只写
  被区块修改的域名与 TLD —— 绝不写完整
  状态；
- **无同步重复**：RAM 状态不按区块重新
  持久化；磁盘状态不按区块重新加载；
- **点读**：每次读取打开一个短读事务、
  返回值、关闭；redb `AccessGuard` 只在有界
  拷贝期间存活；
- **历史区块绝不进 RAM**：`load_chain` 只加载
  tip；`load_chain_replay` 逐高度流式处理；
- **被维护的计数器**：`domain_count` 是
  O(1)、不是扫描。

## DoS 边界

存储限定恶意对等节点能迫使它持久化或
分配什么：

- **`MAX_DHT_CACHE_ENTRY`（64 KiB）**：更大
  的 DHT 记录在 `put_dht_cache` 的任何写事务
  之前被拒绝 → `StorageError::TooLarge`
  （什么也不写）；
- **`MAX_DOMAIN_PAGE`（10,000）**：
  `iterate_domains` *内部*把请求的页限制到
  `min(max, 10_000)` —— 过大的 `max` 绝不在
  RAM 中物化超过 10,000 个状态；
  `iterate_tlds` 有相同边界（`MAX_TLD_PAGE`、
  M7d）；
- **保留的 `meta` 键**：`meta_set` 以
  `StorageError::ReservedKey` 拒绝 `tip`、
  `tip_height`、`format_version`、
  `domain_count`、`tld_count` 与
  `verified_snapshot_v1`（P0.3）—— 覆写存储
  的内部记账会静默腐蚀它。
- **双重有界的 DHT 缓存（LRU）**：
  `DHT_CACHE_MAX_ENTRIES`（100,000 条）与
  `DHT_CACHE_MAX_BYTES`（2 GiB —— 敌意场景
  防护；每条已由 `MAX_DHT_CACHE_ENTRY` 限制
  在 64 KiB）、先到者按键序驱逐。计数器
  （条数 + 字节）在写入时维护 —— 绝不为
  计数而扫描；精确重建只发生在打开时
  （唯一的全量扫描、在启动时）。访问器
  `dht_cache_entries`/`dht_cache_bytes`、边界
  可经 `set_dht_cache_bounds` 调整（测试：
  `dht_cache_respects_the_double_bound`）。

## 信任模型

两条加载路径、两个层级的保证：

- **`load_chain` —— 本地磁盘信任**：tip 的
  哈希从 tip 区块头部**重算**并与存储的
  `meta["tip"]` 比较（分歧 →
  `StorageError::Corrupted`、绝不在伪造哈希上
  恢复链）、域名状态被严格解码。然而
  历史区块**不**被重放：持久化状态被按面值
  接受。这是信任自己磁盘（只由同一节点
  原子地写入）的节点的快速启动。自 M6b
  起、该路径在可用时先使用**状态快照**
  （见下文）：快照的锚被重算、H 之上的
  后缀像活跃区块一样被重新验证 —— 重放
  部分因此是零信任、冻结的基座是磁盘
  信任。
- **`load_chain_replay` —— 零信任**：所有区块
  被逐一重读并重新验证（完整验证、与活跃
  区块相同）；得到的 tip 必须等于存储的
  tip。这是控制/修复工具。

**建议**：严格的节点（或存储文件可能被篡
改的节点 —— 共享磁盘、可疑备份的恢复）
必须经 `load_chain_replay` 而非 `load_chain`
启动。

## 并发

redb 4.x 允许**每个文件每个进程单一
`Database` 对象**（同时双重 `open` 以
`DatabaseAlreadyOpen` 失败）。受支持的模型：
克隆的 `RedbStore`（`Arc<Database>`）跨线程
共享 —— 写者由 `&mut self` + redb 内部锁
串行化、读者并发（快照读事务）。所有句柄
完全 drop 之后、文件可重新打开。

## 测试（`tests/redb_store.rs` + `#[cfg(test)]` 模块）

每个测试使用自己的 redb 临时文件
（`tempfile`）。覆盖：

- 区块往返创世→N、重启、重读、重放；
- M7d：TLD 注册表经原子增量持久化、重启时
  恢复（重启前声明的 TLD 下的
  `RegisterDomain` 仍可接纳 —— 窗口已关
  闭）、重开时精确计数器、升序 `tlds` 游标、
  失败的追加不留下 TLD、保留键 `tld_count`
  被拒绝；
- 原子性：失败的追加不留任何东西（tip、
  域名、索引完好）；重复增量只计一次；
- 域名游标：分页 100/7、升序 `DomainId`
  顺序、严格在存在或缺失 id 之后开始、
  `max = 0`；
- 精确的 `domain_count` 计数器（插入/更新/
  重开）；
- DHT 缓存往返 + 覆写；
- 并发：两个线程在克隆句柄上读取；
- 损坏的数据库：截断/垃圾文件 → 类型化
  错误、无 panic（`catch_unwind`）；
- M6b：快照按间隔写入、严格低于 tip；
  从快照启动 == 完整的逐位重放
  （state_root + SMT、完整逻辑状态、tip）；
  被篡改的快照（损坏的 tip_hash、不可解码
  状态、height > tip、错误长度 meta）→ 静默
  回退全量加载、相同最终链；重放后缀 <
  间隔；跨间隔时快照替换（无累积）；无
  快照表的 M6b 前存储加载方式相同；
- M8：启动时重建防重放索引 —— 窗口内被
  重新包含的交易在恢复后被拒绝
  （`TxReplay`）；窗口之外的旧交易被接纳
  （修剪不变、状态规则裁决）；对比窗口
  更短与更长的存储、精确计数 == 活跃索引
  （== `load_chain_replay`）；快照启动：窗口
  中 H 之下的部分由重扫重建、H 之上的
  部分由后缀重放重建；
- P0.2：名称索引 —— 120 个域名的
  `resolve_name == DomainId::from_name`（活跃、
  重启后、擦除 + 重建后）、已声明 TLD 的
  `resolve_tld`；过期域名的 GC 移除条目
  （无幽灵）、重建前后皆然；域名/TLD 命名
  空间的严格分离；失败的追加不留下索引
  条目；索引键是文档记载的 BLAKE3 派生；
- P0.3/P0.4（`tests/verified_snapshot.rs`）：
  导出 → 导入 → 启动 == 完整重放（状态、
  tip、逐位 `state_root_smt`；导入的状态是
  已终局区块的、而非 tip 的）；被篡改一个
  字节的页 → 任何写入之前拒绝（存储严格
  不变）；缺失页 → 拒绝；错误的
  `state_root`（清单 ↔ 检查点、然后重算根 ≠
  已签名根）→ 拒绝；错误计数器/高度/不一
  致 tip/重复 id → 拒绝；导入要求空存储；
  导出要求检查点高度处有持久化快照；锚
  契约（`append_bootstrap_anchor`：必需标
  记、一致性、幂等）；严格的序列化格式
  （多余字节、标签、DoS 边界）；H 处快照
  然后加入区块 H+1..H+k → 链与全节点相同
  （状态/tip/根）、无 H 之下历史的启动、
  防重放索引 = 恰好 H 及之上的交易。

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

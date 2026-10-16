# RFC — 營運證明與最終性委員會（proof-of-service）

> 狀態：規範性 — 端到端已實作（步驟 1 至 5：
> core/protocol/blockchain/network/CLI；本文件是
> 整併形式）。以下每條規則都對程式碼驗證過；
> 引用的常數是程式碼中的值。
> 日期：2026 年 10 月（規範整併）。觸及：
> `scone-core`、`scone-protocol`、`scone-blockchain`、
> `scone-network`、`scone`（CLI）。

## 1. 動機

最終性池是擁有者池（活躍網域/TLD 的
擁有者）。持有給予席位 — **營運**不給予
任何東西。本 RFC 把簽署檢查點的權利繫於
線上中繼站的營運證明，而絕不以持有權為
條件：

- **保留的自由**：私有 TLD 只要持續續期其
  任期（`RenewTld`、3 年任期）即保持可
  指派、無任何活動條件。只有*檢查點簽署*
  是以在場賺取的。
- **營運的誘因**：營運一個私有 TLD（或任何
  席位）= 進入委員會。營運者參與維持
  網路運作，協定承認它。
- **可衡量的去中心化**：委員會成為*實際
  線上*的營運者集合，而非休眠金鑰。

## 2. 總覽

一個輕量的 `Heartbeat` 交易（判別碼
`0x71`），由**營運者**金鑰簽署、每 epoch
發射一次。狀態保有近期心跳的滑動窗口。
一個 epoch 的委員會**從池中有效營運者
持有新鮮心跳的子集**選出。無心跳：不
簽署，但不喪失擁有權。

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

簽署本體（線上格式、`scone-protocol`）：
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖
height_seen.v ‖ timestamp.v ‖ operator[32]`
（簽署者的金鑰出現在本體尾部、與每筆
交易相同）`‖ signature[64]`，即編碼後約
230 位元組 — **無 PoW**：成本是持續在場
（每 epoch 一次心跳、持續數月）。

簽署者就是營運者：`owner` 恆由 `operator`
重新衍生（絕不取自輸入），解碼以型別化
錯誤（`InvalidOwner`）拒絕與開頭
`operator` 欄位不同的簽章金鑰。

- `tip_seen`／`height_seen` 供可觀測性
 （偵測有 bug 或落後的營運者）；它們
  **不**影響有效性（輕微落後的節點仍是
  活著的節點）。

### 3.1 純驗證（scone-core）

1. 對網路的正典酬載有有效的 Ed25519
   簽章。
2. `owner` == 由 `operator` 衍生的身分
  （否則 `InvalidOwner`）— 在編碼、解碼
   **以及** `validate()` 時檢查。

### 3.2 狀態規則（套用、scone-blockchain）

在高度 `block_height` 的承載區塊中套用
心跳時（每次拒絕都先於任何突變）：

1. **跨 epoch 防重播**：每個營運者的
   `timestamp` 嚴格遞增 — 若
   `timestamp <= operator_last_hb[operator]`
   則型別化拒絕 `StaleHeartbeat`。
2. **每 epoch 唯一性**：每（`operator`、
   承載區塊的 epoch）只接受一次心跳 —
   重複以型別化錯誤（`HeartbeatDuplicate`）
   拒絕。epoch 是
   `block_height / epoch_min_blocks`（與
   最終性相同的定義、絕非新常數）。
3. 有日誌地插入窗口 + 更新
   `operator_last_hb`（reorg 中可逆、
   逐位元 undo）。

### 3.3 反垃圾

- 每營運者每 epoch 1 筆被接受的 tx —
  最大流量是 `pool_size / epoch_secs`、
  可忽略。
- 無費用（經濟層之前）：速率在結構上
  有界。

## 4. 營運委派

席位 → 營運者的連結經由兩筆專屬交易：

- `SetTldOperator { tld_id, operator }`
 （判別碼 `0xA9`），由**私有** TLD 的
  **當前**擁有者簽署（開放 TLD 沒有可
  合法簽署的人 — 型別化拒絕
  `TldIsOpen`）。`operator: None` = 撤銷、
  回到擁有者金鑰的自營；即刻可撤銷。
- `SetDomainOperator { domain_id, operator }`
 （判別碼 `0x5E`），由網域的當前擁有者
  簽署。相同語義。
- `RegisterTld`／`RegisterDomain`／
  `AssignDomain` 不攜帶 `operator` 欄位：
  席位 → 營運者的連結恆為最近的鏈上
  事實、於尖端解析。

線上格式上的營運者選項是正典的：
`0x00` = `None`（自營）、`0x01` = 原始
32 位元組金鑰；任何其他值在解碼時被拒。

狀態維護：

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, u64>       // bounded counter
operator_last_hb: HashMap<OperatorKey, u64>     // last accepted ts
```

套用規則（`apply_seat_operator`、有
日誌）：

- **冪等性**：重新委派給同一營運者（或
  撤銷不存在的委派）是無日誌的 no-op
 （逐位元 undo 需要它）。
- **反出租界限**：一個營運者同時只能被
  **單一席位**委派
 （`MAX_SEATS_PER_OPERATOR = 1`）。第二個
  席位委派給已被佔用的營運者 → 型別化
  拒絕 `OperatorAlreadyDelegated`。理由：
  若無界限，大型營運者會批發簽署權並
  重現中心化 — 這正是選出的委員會要
  避免的。

## 5. 新鮮度窗口

### 5.1 滑動窗口

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- 條目在有效心跳套用時插入（§3.2）。
- 在**每個**區塊套用時決定性清除
  （`gc_heartbeats_journaled`）：任何
  滿足 `e + HEARTBEAT_WINDOW_EPOCHS <=
  epoch` 的 epoch `e` 條目被移除（有
  日誌 — reorg 中可逆、如到期佇列）。
- `HEARTBEAT_WINDOW_EPOCHS = 2`：當前
  epoch + 前一個 — 營運者可以錯過**一個**
  epoch 而不失去新鮮度，第二個使它
  過期。
- **結構性記憶體上限**：窗口至多持有
  `HEARTBEAT_WINDOW_EPOCHS ×
  operator_count` 條（每營運者每 epoch
  一條、元組金鑰）；`operator_last_hb`
  受營運者數量約束且絕不清除（防重播
  規則是單調的）。

### 5.2 新鮮度

「新鮮」= 對 `e` 在
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1,
epoch]` 內存在條目 `(e, operator)`。由於
狀態在每個區塊清除，存在於
`recent_heartbeats` 中**即是**新鮮度
（`operator_is_fresh`）。

### 5.3 有效營運者

席位的**有效營運者**是存在活躍委派時的
`seat_operators[seat]`、否則是擁有者的
金鑰（`effective_operator`）。席位以該
有效金鑰簽署 — 委派移動的是簽署權、
絕不是所有權。

## 6. 委員會選舉

```
eligible = { effective operators of the frozen pool's seats
             (banned key excluded) having a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

規範細節（在 `finality.rs` 驗證、於
`/docs/en/technical/blockchain.md` §「席位
與最終性委員會」規定）：

- `pool` 是**凍結於最後已最終化檢查點**
  的池（`FinalizedBase.pool` — 絕非活躍
  狀態）：本機視圖會在節點間分歧、法定
  數永遠無法形成；
- `seed` 由前一檢查點的內容雜湊衍生
  （`signing_hash`）；
- 凍結池的每個席位由尖端的狀態解析至其
  **有效**營運者（委派與心跳是最近的
  鏈上事實：同一尖端的兩個節點算出相同
  集合）；被禁金鑰保持排除；
- 抽取在排序／去重的有效金鑰集合上執行
  （`select_committee` 不變）。

**明確且安全的降級**：引導中的網路（無
心跳）、或營運者的全面故障，退回整個
池 — 建構上最終性絕不喪失。這是專案的
「絕不死鎖」規則在此處的應用。

### 6.1 冷引導

Epoch 0（以及只要新鮮營運者少於
`MIN_FINALITY_COMMITTEE_SIZE`）：整個池。
隨營運者發射，約束自然收緊。無需組態
任何特殊的過渡期。

## 7. 自動發射（scone-network）

在每個生產循環（`Wake::Produce` — 無新
計時器），中繼站檢查是否跨越了 EPOCH
邊界（`height / epoch_min_blocks`、與
最終性相同的定義）。在新 epoch 的第一個
循環，若以下條件成立它發射心跳：

1. 已武裝錨點金鑰（`--anchor-key` 或
   自動），**且**
2. 該金鑰本身持有活躍席位（本機檢查：
   金鑰在鏈上資格池中 — 活躍網域/TLD 的
   擁有者、被禁金鑰排除）。

本機反垃圾：每節點每 epoch 至多一次
發射（`last_emitted_epoch` 備忘、本機
記憶體、絕非鏈上狀態）；備忘只在成功
注入記憶體池後採取 — 失敗在下一循環
重試。這裡沒有任何致命的東西：最壞情況
心跳等到下一循環。

**受委派的營運者**（自身不是擁有者）
不在自動發射的涵蓋範圍：他們經 §8 的
CLI 建構器發射（手動或氣隙營運）。心跳
**不一定**由發射者自己產出進區塊：它
如任何其他交易（記憶體池、由任何人
生產）。

## 8. CLI（`scone`）

離線心跳建構器（隨 `tx` 表面一起）已隨
離線 tx 表面移除（cli-simplify-v1）。中繼站
自動發射自己的心跳（§7）；若再需要
手動／氣隙發射，將以專屬指令回歸。原本
的預設是：`tip_seen` = 32 個零位元組
（僅供可觀測性）、`height_seen` = 0、
`timestamp` = now。
- 如每筆 tx，`tx sign` 把 operator 與
  owner **重新綁定**到真實的簽署金鑰
 （輸入值絕不被信任）；`tx verify` 在
  本機驗證。

## 9. 可觀測性

- `NodeStatus`（線上 p2p 訊息 `0x0d`）
  攜帶 `fresh_operators: u32`（varint）：
  有新鮮心跳的營運者數量 — 池中符合
  委員會資格的活躍子集（池席位的有效
  營運者、被禁金鑰排除、排序／去重）。
  由中繼站從
  `state().fresh_operators(height)` 填充；
  由 `scone status` 顯示為
  `fresh operators: N`。
- 活動論述：「營運一個私有 TLD = 進入
  去中心化 DNS 的最終性委員會」— 可在
  鏈上驗證。
- 未來表面（未指定）：`operator_status` —
  每席位：受委派的營運者、新鮮度
 （新鮮／過期／從未見到）、最後的
  `tip_seen`。本機控制 RPC 已移除（純
  p2p 節點）；表面仍有待定義。

## 10. 本 RFC 不做的事

- 不因不活躍而**喪失持有**（選項 A —
  未營運 TLD 的衰減 — 被明確拒絕）。
- **無獎勵**（選項 C 等待經濟層）。
- 心跳不證明服務的*品質*（產出的區塊、
  服務的 DHT 記錄）— 只證明在場。計算
  真實貢獻是可能的未來擴充、但更可被
  博弈。

## 11. 受保護的不變量

- 逐位元決定性：窗口與選舉是已套用
  狀態的純函數（有日誌、逐位元 undo）。
- 絕不因網路資料 panic：畸形的心跳或
  委派 → 型別化拒絕（`InvalidOwner`、
  `StaleHeartbeat`、`HeartbeatDuplicate`、
  `OperatorAlreadyDelegated`、`TldIsOpen`、
  `NotTldOwner`、`NotOwner`）。
- 持有是無條件的；只有檢查點簽署是
  有條件的。

## 12. 常數與參數（程式碼值）

| 名稱 | 值 | 位置 |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| `Heartbeat` 判別碼 | `0x71` | `scone-protocol/src/transaction.rs` |
| `SetTldOperator` 判別碼 | `0xA9` | 同上 |
| `SetDomainOperator` 判別碼 | `0x5E` | 同上 |

這些值是共識的一部分：改變它們會破壞
鏈的重播能力（窗口／清除）或選舉
（委員會）。

## 13. 實作計畫 — 最終狀態

| 步驟 | Crate | 內容 | 狀態 |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`、`SetTldOperator`、`SetDomainOperator`、線上格式、驗證 | 完成 |
| 2 | blockchain | 滑動窗口（有日誌）、過濾的池、選舉、約 15 個測試 | 完成 |
| 3 | network | 自動發射、tx 轉發 | 完成 |
| 4 | CLI | `tx build heartbeat`、`scone status` 中的 `fresh operators` 顯示 | 完成 |
| 5 | docs | 本 RFC 整併為規範形式、fr/en 鏡像 | 完成 |

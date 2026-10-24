# 網路中繼（M4）— 架構、流程與 CLI 控制

> Scone 網路中繼站（`scone-network`）的規範規格。
> 任何文件／程式碼分歧 = bug（修正事實來源）。

## 總覽

**中繼站**是 Scone 的完整 P2P 節點：它維護正典
區塊鏈（RAM + `NodeStore` redb）、轉發交易與區塊、
產出區塊（devnet 模式）、服務同步協定、在
Kademlia DHT 中發布並解析 `SignedDnsRecord`，並
暴露 CLI 直接查詢的 **p2p 協定**
（`/scone/reqres/1`）（RPC→p2p 統一：JSON TCP
通道已移除）。

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

`crates/scone-network` 是工作區中**唯一的
async crate**：其下的一切（crypto → core →
protocol → blockchain → storage）保持同步且
純淨。

## 連線准入（P1.5）

swarm 層級的硬性限制（libp2p
`connection_limits` 行為、`SconeBehaviour` 的
第一個條目）：

- `MAX_TOTAL_CONNECTIONS = 128` — 每個中繼站
  已建立連線（傳入 + 傳出）的上限；超額的
  傳入被拒絕、絕不排隊；
- `MAX_PER_PEER_CONNECTIONS = 4` — 單一 peer
  無法以開啟多條連線壟斷總配額；
- `MAX_PENDING_INCOMING = 32` — slowloris
  防護：未認證的半連線絕不無限期等待。

這些限制補充既有的每訊息限制
（`MAX_MESSAGE_LEN` 1 MiB）與每預算限制
（同步 `max ≤ 128` 區塊、DNS TCP 64 連線／
512 B／5 秒）。連線洪氾攻擊者得到的是
拒絕，而非記憶體成長。

## Peer 探索

三個互補機制，依此優先順序：

1. **明確種子**（`--bootstrap`、完整
   multiaddr）— 恆定優先；提供非空清單
   即取代其他一切。`--no-builtin-seeds`
   清空內嵌種子以實現 100% 私有部署。
2. **內嵌種子**（`scone-network/src/config.rs`
   的 `builtin_seeds(network)`）— 在每次官方
   二進位檔發布時凍結、每網路一份。未
   組態的中繼站獨自加入網路。（公開
   testnet 尚未部署前在 testnet 上為空。）
3. **習得的目錄**（`<data-dir>/peers.json`）—
   每個經 `identify` 習得的位址都被保留
  （最後所見；上限 **2000 條** — 是位址而
   非連線、最大時約 400 KiB RAM — 以及
   **90 天年齡切斷**：最後所見超過 90 天的
   條目在寫入時丟棄、載入時忽略，讓
   2000 個槽位填滿可信的 peer 而非
   churn 前的化石 — 其唯一貢獻只會是
   開機時的撥號逾時），並在生產 tick
   約每分鐘儲存一次。無種子無 bootstrap
   開機時重組目錄：中繼站無需組態就
   找到它慣常的 peer。它是快取：檔案
   缺失／損毀 = 空目錄，絕不是錯誤。
4. **UPnP**（`--upnp`、opt-in）— 由家用
   路由器映射 QUIC 監聽埠：節點無需
   circuit relay 即可從外部觸達。在
   `NewExternalAddr` 時，外部位址註冊為
   swarm 的真實外部位址
  （`add_external_address` — identify／
   autonat 可見、transport-relay 候選）。
   無 IGD 閘道（`GatewayNotFound`）或
   不可路由閘道（`NonRoutableGateway`）：
   一條普通 info 日誌、跳過映射 — 家用
   LAN 上的正常情況。過期的映射（續約
   失敗）是警告。

UPnP 保持**預設停用**（`config.rs` 中
`enable_upnp` = `false`）：公開部署未經同意
不與任何路由器交談。**mDNS 探索已被移除**
（remove-mdns-discovery、2026-10-22）：對等
連線僅來自明確 bootstrap、內建種子與
Kademlia — 舊版 `--mdns`／`SCONE_MDNS` 旗標
仍被接受但無效果（腳本相容、與 `--rpc`
同樣處理）。

### 本機拓撲最佳化（恆開）

中繼站使用純 `scone-topology` 引擎主動
最佳化其鄰里（見該 crate 的文件）。**恆定
啟用**；唯一暴露的調整是**設定檔**
（`--topology-profile`、環境變數
`SCONE_TOPOLOGY_PROFILE`、預設 `default`）：

| 設定檔 | min / 目標 / max | 評估 | 適合誰 |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 秒 | Raspberry Pi、家用盒、1-vCPU VPS |
| `small` | 4 / 8 / 12 | 90 秒 | 小型伺服器、家用 NAS |
| `default` | 8 / 16 / 32 | 60 秒 | 標準伺服器（預設） |
| `high` | 16 / 32 / 64 | 45 秒 | 主權節點、骨幹 |
| `very-high` | 32 / 64 / 128 | 30 秒 | 創始人樞紐 |
| `devtest` | 2 / 4 / 6 | 5 秒 | **僅實驗室／測試**（設計上不穩定） |

```
scone relay --topology-profile tiny
```

防護參數（遲滯 0.15、冷卻 600 秒、寬限
120 秒、測量 15 秒）到處相同：它們守護
穩定性、沒有理由隨節點規模縮放。未知
名稱是型別化啟動錯誤。

不一致的組態（`min > target > max`、門檻在
`[0,1]` 之外 — 含 `NaN` —、零區間）是
**型別化啟動錯誤**，絕非沉默行為。

**視圖（`TopologyView`）** — 由 swarm 事件
餵入：

- 成功的 `ping` → RTT 觀測（8 個樣本的中位
  數窗口）；
- 失敗的 `ping` → 可靠性懲罰；
- 收到 `identify` → 位址（每 peer ≤ 4）+
  宣告的協定（kad/reqres/identify/relay
  位元遮罩）；
- 連線建立 → 連線狀態、方向
  （傳入／傳出）、年齡、偵測到的
  circuit-relay；
- 連線關閉 → 只有非自願時才懲罰 +
  churn 計數（自願逐出不計）；
- 撥號失敗 → 可靠性懲罰。

視圖有界（500 peer；超過後只記住已連線
的 peer）。

**來自 kbuckets 的 `kad_required`** — 每次
評估前，中繼站走訪 Kademlia 路由表：
在某 bucket 中 `Connected`、或在
SATURATED bucket 中 `Disconnected`（k = 20）
的 peer 受保護。受保護的 peer 絕不被選為
逐出。

**最佳化迴圈** — 以設定檔的評估區間
（`default` 下 60 秒；見上表）：

1. 重新計算 `kad_required`（kbuckets）；
2. 引擎評估（遲滯：120 秒寬限期、最低
   分數改善 +15%、逐出冷卻 600 秒）；
3. 套用決策：`AddCandidate` → 撥號候選；
   `Replace` → 關閉被逐出的 peer（一次 —
   漸進、不迴圈）然後撥號替換者；
4. 記錄一份**指標**快照（INFO 級）：
   已知／已連線／傳入／傳出／circuit／
   受保護計數、RTT 平均／中位數、互異
   IP bucket、最大集中度、churn
  （替換／斷線）。

鄰里分數絕不是信任度量（見
`scone-topology` 文件）：沒有共識決策由它
衍生。

### 可達性（AutoNAT）與傳輸中繼（circuit-relay v2）

每個中繼站執行 AutoNAT（客戶端 + 伺服
器）：節點請求其 peer 嘗試**回撥**它；
判定（`reachable`：是 = 可從外部觸達、
否 = 在 NAT 之後、未知 = 尚未測試）每
5 分鐘刷新並暴露於 `NodeStatus`（欄位
`nat_reachable`、線上 varint 0/1/2）— CLI
顯示它（`scone status`）。私有位址也算
（`only_global_ips = false`）：LAN 網格
測試其 LAN 可達性。

可觸達的中繼站啟用 circuit-relay v2
**hop**（可為 NATed peer 中繼流量）與
**DCUtR**（打洞：兩個經 circuit 連線的
節點嘗試直接連線並釋放中繼站）。被判定
NATed 的中繼站自動向第一個已知可觸達
peer 請求預約並經它監聽（位址
`/p2p/<relay>/p2p-circuit`）。

生產規則：經中繼的 NATed 節點不應成為
區塊生產者（延遲／可用性退化）— PoS
規則（席位 = 活躍網域）尚未禁止它；
將在公開 testnet 前作為營運要求載明。

## 傳輸與 libp2p 協定

| 協定 | 名稱 | 角色 |
|---|---|---|
| QUIC（quic-v1） | — | 主要傳輸（udp） |
| identify | `/scone/id/1` | Peer 位址探索（填充 Kademlia 表） |
| ping | libp2p 標準 | 活躍性 |
| Kademlia | `/scone/kad/1` | DHT：金鑰 = `DomainId`（32 B）、值 = `SignedDnsRecord` 的正典編碼 |
| request-response | `/scone/reqres/1` | scone-protocol 應用訊息 |

### 應用框（`/scone/reqres/1`）

每個框是 `len: u32 BE || bytes`，其中
bytes 是 `scone-protocol` 中 `Message` 的
正典編碼（既有編解碼器 — 無第二種
格式）。長度在任何讀取或配置**之前**由
`limits::MAX_MESSAGE_LEN` 約束：宣告更大
框的 peer 被拒絕而不花記憶體。

交換的訊息（每次請求／回應往返一個）：

| 訊息 | 方向 | 回覆 |
|---|---|---|
| `Hello { version }` | 交握 | `Hello`（本機版本） |
| `Ping`／`Pong` | 活躍性 | `Pong` |
| `GetBlock { hash }` | 查詢 | `Block`（RAM 視窗）或未知時 `Pong` |
| `GetBlocks { start, max }` | 同步 | **一個** `Block`（高度 `start`），或無時 `Pong` |
| `Block (b)` | 區塊轉發 | `Pong`（ack） |
| `Transaction (tx)` | 交易轉發 | `Pong`（ack） |
| `GetRecord { domain_id }` | 本機 DHT | `Record`（本機快取）或 `Pong` |
| `Record (r)` | 發布 | 接受時 `Pong`，否則錯誤 |

規則：**每個請求恰好一個回覆**；確認
訊息是 `Pong`。同步回覆一次攜帶一個
區塊 — 補趕在請求方迭代（見「同步」）。
這是刻意有界的：每 peer 絕不超過一個
在途區塊。

## 中繼站行為

### 啟動

0. 網路解析（`--network testnet|mainnet|devnet`、
   專案開發期間預設 testnet — M8b）。
   data-dir **每網路一份**
  （`~/.scone/<network>/`）：持有另一網路
   鏈的非空儲存庫被拒（`Corrupted`）；
1. 開啟／建立 redb 儲存庫
  （`<data-dir>/chain.redb`）；
2. `load_chain(store, network)`：O(1) 尖端 +
   分頁的網域狀態（RAM 視窗：創世 +
   尖端）；
3. swarm 建構（每會話短暫的 Ed25519 身分 —
   持久化網路身分是開放決策）；
4. 組成開機清單（見
   [Peer 探索](#peer-探索)）：明確
   `--bootstrap`（必須以 `/p2p/<peer-id>`
   結尾）否則網路的內嵌種子否則習得
   目錄（`peers.json`）；
5. 撥號每個 multiaddr 並註冊進 Kademlia 表
  （死撥號被忽略 — 其餘組成網路）。

### 接收交易（P2P 或 CLI `submit tx` 提交）

1. 嚴格有界解碼（`MAX_MESSAGE_LEN`）；
2. 完整密碼學驗證（`validate_transaction`：
   擁有者／金鑰綁定、對正典酬載執行
   `verify_strict`）；
3. **防重播優先**（便宜、與狀態無關、
   確定性），然後狀態預檢（`push_block`
   規則的便宜鏡像 — 昂貴的 PoW 只在
   push 時驗證）：重播的 `RenewDomain`
   回答 `TxReplay` 而非謊報的 PoW
   錯誤（其證明綁定第一次套用已越過的
   到期日）。網路分離在步驟 2 強制：
   簽章酬載注入**中繼站**的網路、在
   他處簽署的交易以 `InvalidSignature`
   失敗（2026-09 線上格式：不再有
   `network` 欄位）；
   `RegisterDomain` → 被註冊名稱的
   TLD（否則 `UnknownTld`、D1/M7c）
   **且開放**（否則 `TldClosed`、
   M8b）然後名稱空閒；`UpdateDomain` →
   已知網域、精確序號 `current + 1`、
   擁有者；`RegisterTld` → 空閒 TLD；
   `TransferTld`／`RevokeTld`／`SetTldOpen`
   → 已知 TLD + 擁有者簽署；
   `AssignDomain` → TLD 擁有者 + 空閒
   名稱；`RenewTld` → 已知私有 TLD +
   擁有者、固定任期重新衍生、寬限窗口
   預檢；`RenewDomain` → 已知網域 +
   擁有者、下一高度時的續期 PoW 預檢
  （綁定當前到期日）；
4. 記憶體池插入（固定容量、依 `TxId`
   去重 — 重新提交已在池中的交易是
   no-op）；
5. 廣播給 peer（發送者除外）。

### 接收區塊（P2P、同步或自產）

1. `push_block`：既有的完整驗證（父 =
   正典尖端、高度、重新計算的 Merkle
   根、每筆交易先驗證再套用 — 每區塊
   原子）；
2. 若接受：中繼站讀取套用結果
  （`push_block_with_gc` → 雜湊 + 決定性
   GC 移除的網域）並經 `store_block_with_removals`
   持久化（原子差量附加：區塊 + 尖端 +
   修改的狀態 — upsert **及**移除：過期
   網域、被撤銷的 TLD）。因此重啟絕不
   能復活過期的註冊；
3. 記憶體池調解（已收錄的交易被移除）；
4. 廣播給 peer（發送者除外）。

無效區塊以型別化錯誤被拒；中繼站繼續
運轉。絕不因網路資料 panic。重型套用
（Argon2 + redb 提交）經過
`tokio::task::block_in_place`（同步驅動、
p2p 接受、生產）：執行時把就緒任務移到
其他工作者 — 套用區塊不再拖住 DNS 服務
或指標。

### DNS 快照（發布）

在**每個**已套用區塊之後（p2p 接受 —
含 reorg —、同步驅動、生產）**以及**開機
時（重啟不得讓活躍名稱 NXDOMAIN 到下一個
區塊），中繼站把**權威 DNS 快照**發布到
DNS 表面讀取的 `watch` 饋送上（完整規格
見 `dns.md`）：

- 尖端延伸（壓倒性的路徑）：**O(被觸及)
  差量**、寫時複製 — 區塊的交易點名被
  觸及的網域，加上決定性 GC 移除
  （`AppliedBlock.gc_removed_domains`、
  push 時評估）；
- reorg／開機：**完整重建**（一次分頁的
  `ChainState::dns_facts` 遍歷）。

### 區塊生產（M5：簽署生產者）

每 `produce_interval` 秒（預設 2 秒）：若
記憶體池非空，由 `BlockBuilder` 組裝區塊
（`timestamp = now`），**由獲授權的生產者
簽署**（`header.consensus` 中的酬載
`SCONE-BLOCK-V1`、見
`/docs/en/technical/blockchain.md`）並走
與接收區塊完全相同的接受路徑。無交易
的中繼站不產出任何東西。

生產金鑰：中繼站配備錨點金鑰
（`--anchor-key`）時用之 — 錨點是獲授權
的生產者 — 否則用固定的 devnet 金鑰，
在生產者池為空時有效（開放引導：
創世）。一旦存在活躍的網域/TLD，以
devnet 金鑰簽署的區塊被拒
（`InvalidProducer`）。

### 錨點迴圈（最終性、M5）

帶 `--anchor-key`（`.sconekey` 金鑰檔、
通關密語經 `--anchor-passphrase-env`、
預設 `SCONE_ANCHOR_PASS` — `scone-network`
函式庫預設 `SCONE_ANCHOR_PASSPHRASE` 經由
二進位檔永不被觸及，clap 恆覆寫它）：

- **提議者輪替**：每個 epoch 恰有**一位**
  被指定的委員會成員自我提議
 （`committee[epoch % size]`、決定性）；
  其他成員絕不簽署自己的尖端 — 檢查點的
  內容取決於提議者的尖端，而未加閘門的
  自我提議曾把 epoch 撕裂成相互競爭的
  內容、每個防護都正確地將其作為
  equivocation 拒絕（testnet、2026-10-21）；
- 在每個已接受／已產出的區塊後，若本
  節點是當前委員會成員**且**委員會達到
  `MIN_FINALITY_COMMITTEE_SIZE`，中繼站
  經其 `SignerGuard` 簽署被提議的檢查點
  （data-dir 中的 `signer-state.dat` —
  一金鑰 = 每 epoch 一票、防當機
  fail-closed）並 gossip 它；
- **收到即投票**：內容在我們的鏈上有效
  （被引用的區塊存在、且雜湊相同）並
  帶有委員會簽章的提議也會得到我們的
  簽章 — 沒有它，每個錨點只簽署自己
  尖端的內容、法定人數永遠湊不齊；
- 收到的簽章（`Message::Checkpoint`）依
  `CheckpointData` 雜湊累積（上界 256
  條）；同一資料上有 `quorum_for(委員會
  規模)` 個成員簽章 → `accept_checkpoint` +
  廣播已最終化的檢查點；
- **最終性持久化**：檢查點窗口與凍結
  基座在每次最終化時寫入儲存、並在啟動
  時重新載入。檢查點是 gossip 狀態
 （絕不由區塊攜帶）— 沒有此持久化的
  重啟會從空基座開始、再也無法鏈接
  peer 提供的窗口（網路窗口之下的
  epoch 錨點已消失）；
- **最終性追趕**：基座為空的中繼站
 （全新節點的啟動）每個 tick 向 peer
  請求 `GetCheckpoints`、直到基座恢復；
- 無錨點金鑰的中繼站保持**被動**：它
  累積、最終化、轉發，但絕不簽署；
- `GetCheckpoints` → 服務
  `checkpoint_window()` 窗口；
- 迴圈的任何錯誤被記錄並吞下 — 絕不
  致命；
- 狀態（`GetStatus`）暴露 `checkpoints` 與
  `finalized_epoch`。

### 分叉停等與 reorg（黏性少數修正）

同一高度的姊妹區塊、或父非當前尖端的
區塊（`ParentNotTip`／`UnknownParent`）
絕不遺失：它被**停等**（有界映射、
`PARKED_CAP = 2048` 區塊、每個約 KiB —
有界的最壞情況 RAM；滿載的停等區靜靜
忽略新來者）。已提供給 `accept_block`
的雜湊被去重（`seen_blocks`）：重新
gossip 的輸家區塊不被重新評估。

在**每個** `Produce` tick，`reconsider_parked`：

1. 線性清空父已成為尖端的停等區塊；
2. 若某停等分支強於當前尖端（分叉選擇
   規則不變：最終性下限、最長鏈、最低
   雜湊），採納 — 被擠下的字尾本身被
   停等（分叉的**兩**側都保持可評估）。

這解黏了採納等高分叉輸側的節點：
曾被提供而被拒的贏家區塊被週期性
重新評估，而非永遠被遺忘（chaos-30
發現：相距 15 ms 的兩個姊妹區塊）。

**Reorg 持久化**：採納另一分支無法經過
單調附加（`NonMonotonicHeight`）。中繼站
套用 `rewind_to_height(分叉點)` 然後
`persist_state_snapshot`（從鏈的匯出重寫
活躍狀態）然後獲勝字尾的單調重附加
（見 `storage.md`）。持久化失敗被記錄並
降級（開機時經重播痊癒）— 絕不致命。

## 同步（補趕）

每次連上 peer，中繼站請求
`GetBlocks(start = 本機高度 + 1)`。每個
恰好完成預期高度的收到 `Block` 觸發下一
高度的請求，直到耗盡（`Pong` = 沒有
更多）或 `MAX_SYNC_ROUNDS`（10,000）批次 —
即使面對說謊的 peer 也有絕對界限。收到
的區塊經完整驗證：惡意 peer 無法讓任何
無效的東西被接受。

### 記錄 DHT

- **put**（API/web 流程 — CLI 十六進位
  通道已移除）：嚴格有界解碼
  （`MAX_DHT_CACHE_ENTRY`）、對鏈上狀態
  驗證（擁有者、序號、重新計算的
  `record_hash` — 見「記錄驗證」），然後
  `kad.put_record`（本機儲存 + 對最近
  peer 的複製、複製因子的
  `Quorum::Majority`）與持久的 redb 快取。
- **get**（`domain resolve`）：
  `kad.get_record`；等待者綁定於其
  `QueryId` 與請求的 `DomainId` — 回覆
  只解析相符的查詢，且記錄金鑰必須
  恰好是請求的網域（在鏈上驗證**之前**
  檢查）。結果隨後對鏈驗證才回傳。同時
  請求上限：`MAX_DHT_WAITERS`（256）；
  超過即立即錯誤。成功驗證的解析寫入
  持久快取（M5）：本機讀取
  （`domain_info`）隨後無需新的 DHT 查詢
  即服務。
- Kademlia 以伺服器模式執行、無記錄
  TTL（鏈治理有效性，而非時間）。

### 記錄驗證（「鏈是權威」規則）

解析或收到的 `SignedDnsRecord` 只有在
以下條件才被接受／回傳：

1. 網域在鏈上已註冊；
2. `record.owner` = 當前鏈上擁有者；
3. `record.sequence` = 當前鏈上序號；
4. `record_hash(record.record)`（重新計算
   的 BLAKE3）= 鏈上承諾的
   `record_hash`。

否則：P2P 側靜靜拒絕、CLI 側明確錯誤。
記錄的 Ed25519 簽章由中繼站**在收到時
驗證**（`ResolveLocal`：擁有者的公鑰
鏈上已知、簽章必須通過 — 以另一把
金鑰簽署的記錄被拒、`record rejected`）；
它也被傳輸給未來的驗證者 — 鏈上承諾
仍是主要的完整性控制。

## Prometheus 指標

由專屬任務服務的 HTTP 端點（Prometheus
文字格式 0.0.4、手寫展示 — 無新相依）：

- 預設**公開**於 `127.0.0.1:9101`（CLI
  套用該預設；函式庫預設為 `None` — 一個
  行程中的多個中繼站絕不碰撞）。
- `--metrics-addr ADDR:PORT`：將端點綁定到該位址（如本機 `127.0.0.1:9101`，對外 `0.0.0.0:9101`）。
- `--metrics-token TOKEN`：切換為**私有**
  模式 — 需要 `Authorization: Bearer ***`
 （常數時間比較、否則 `401`；token 絕不
  出現在日誌或展示中）。
- 綁定失敗**停用端點**（非致命警告）—
  絕不是節點。

有效綁定在 stdout 宣告（機器可讀契約、
如 `p2p:`）：一聽即出現 `metrics: <addr>`
行。

**19 個計數器**（`counter`、單調 — 熱
路徑：共享 `Arc<Metrics>` 上的
`fetch_add(Relaxed)`、絕非共識輸入）：

| 群組 | 序列 |
|---|---|
| 鏈 | `scone_blocks_applied_total`、`scone_blocks_rejected_total`、`scone_checkpoints_signed_total`、`scone_checkpoint_sigs_received_total` |
| 記憶體池 | `scone_txs_admitted_total`、`scone_txs_rejected_total`、`scone_txs_evicted_grace_total`、`scone_txs_included_total` |
| DNS | `scone_dns_queries_total`、`scone_dns_cache_hits_total`、`scone_dns_cache_misses_total`、`scone_dns_local_chain_total`、`scone_dns_dht_answers_total`、`scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`、`scone_dht_negative_hits_total`、`scone_dht_negative_expired_total`、`scone_dht_negative_capped_total`、`scone_dht_cache_evictions_total` |
| Peer 網路 | `scone_peer_connections_established_total`、`scone_peer_connections_closed_total`、`scone_peer_connections_incoming_total`、`scone_peer_dial_failures_total`、`scone_peer_blocks_received_total`、`scone_peer_transactions_received_total` |

**6 個量規**（`scone_height`、
`scone_mempool`、`scone_peers`、
`scone_domains`、`scone_finalized_epoch`、
`scone_addr_book_entries`）— 前五個每
15 秒經 `RelayCommand` 通道刷新（與 CLI
`status` 同一路徑 — 單一事實來源）；
`scone_addr_book_entries` 由登記冊直接
維護（每次位址簿插入時更新、事實
來源）。

E2E 測試：`metrics_endpoint_public_and_private`
（200 + 展示、無／錯 token 時 401、帶
Bearer 時 200）。

## CLI 控制（RPC→p2p 統一）

JSON TCP RPC 伺服器已被**移除**：CLI
（與註冊服務的 axum 橋）**直接**說 p2p
協定 — 相同線上格式（`/scone/reqres/1`）、
中繼站側相同的處理器、零平行通道。每個
指令對節點的 multiaddr（`--peer`）開啟
QUIC 連線、交換**一個** `Message`、關閉。

中繼站服務的控制訊息：

- `GetStatus` → `NodeStatus { network,
  nat_reachable, tip, height, domain_count,
  mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }`（由
  `scone status` 顯示 — `fresh_operators` =
  有新鮮心跳的營運者、RFC
  proof-of-service）；
- `LookupName { name }` → `NameState`（鏈上
  狀態、如 `lookup`）；
- `GetOwnedNames { owner }` → `OwnedNames`
 （某 `OwnerId` 的活躍網域 + TLD）；
- `Transaction (tx)` → 附帶免費 gossip 的
  提交（中繼站向 peer 重新廣播）；
- `GetRecord { domain_id }`／`Record (r)` —
  DHT 解析與發布（如 CLI 側的
  `scone domain resolve`）；
- 應用層拒絕 → `Message::Error(msg)`
 （以可讀訊息抵達客戶端、而非 EOF）。

客戶端帶起最小的 libp2p swarm
（request-response + identify）、30 秒呼叫
逾時（20 秒往返）。預設 `--peer`：
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…`
（本機開發節點；帶真實 peer id 的完整
multiaddr 由中繼站在 stdout 經 `p2p:` 行
宣告）。

內部 `RelayCommand` 通道為內嵌 DNS 表面
保留（`ResolveLocal`、一次行程內往返、
絕非 DHT 等待者 — 見 `dns.md`）。

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

每個控制指令直接對中繼站的 multiaddr 說
QUIC（`--peer`、預設本機節點 — RPC→p2p
統一、不再有 JSON TCP 通道）。中繼站不
存在 → 清楚訊息「p2p node unreachable —
is a relay running?」、非零退出。`--rpc`
旗標保留但**已過時且被忽略**（RPC 伺服器
已移除；僅為腳本相容）。

### 環境組態（`.env`）

啟動時，`scone`（經 dotenvy、在 clap 解析
之前）從當前目錄載入 `.env` 檔案：檔案
不存在 = 靜靜跳過、絕不是錯誤。載入的值
成為宣告變數（`cli.rs` 中的 `env = "…"`)
的選項之**環境基線**；**明確的 CLI 旗標
恆勝**於 `.env`，`.env` 勝於內建預設：

```text
explicit CLI flag  >  .env variable  >  built-in default
```

儲存庫根目錄的 `.env.example` 檔案記錄每個
變數（複製為 `.env` 並調整；`.env` 已被
**gitignore** — 絕不提交真實的 `.env`）。
變數：

| 變數 | 對應旗標 | 語義 |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet`（預設）、`mainnet`、`devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | 資料目錄（預設 `~/.scone/<network>/`） |
| `SCONE_LISTEN` | `--listen` | P2P 監聽 multiaddr（預設：所有介面、短暫埠） |
| `SCONE_BOOTSTRAP` | `--bootstrap` | bootstrap 種子、**可重複**（每值一行 — dotenvy 累加）；取代內建種子 |
| `SCONE_GENESIS` | `--genesis` | 創世檔（創始節點） |
| `SCONE_MDNS` | `--mdns` | （已過時 — mDNS 探索已移除；被接受、被忽略） |
| `SCONE_UPNP` | `--upnp` | UPnP 埠映射（`true` = 啟用） |
| `SCONE_DNS_ADDR` | `--dns` | UDP DNS 伺服器綁定位址 |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | 遞迴備援、**可重複** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | DNS 快取條目上限 |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | DNS 快取記憶體上限（如 `64MB`） |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | 並發 DNS 查詢上限 |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | 指標端點 bearer token |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | 拓撲設定檔（`tiny`／`small`／`default`／`high`／`very-high`／`devtest`；預設 `default`） |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | 錨點金鑰檔（檢查點 + 生產） |

`SCONE_ANCHOR_PASS` 不是某選項的變數：
它是**持有錨點通關密語之變數**的預設
**名稱**（`--anchor-passphrase-env`、CLI
層級預設 `SCONE_ANCHOR_PASS`）— 中繼站
從該旗標命名的變數讀取通關密語。

**錨點自動（預設開）**：無 `--anchor-key`
時，錨點金鑰檔自動產生／持久化於資料
目錄（`anchor.sconekey`、若有定義則通關
密語為 `SCONE_ANCHOR_PASS`、否則隨機產生
並寫入 `anchor.pass`）— 只要其金鑰擁有
活躍網域/TLD，中繼站即成為生產者（PoS
資格絕非免費）。`--no-anchor-auto` 使
中繼站嚴格保持被動。`--anchor-passphrase-env`
的預設環境變數是 `SCONE_ANCHOR_PASS`。

`--network`：預設跟隨**建置**（啟用主網的
二進位檔為 mainnet、否則 testnet）；
`devnet` 是可重置的本機網路（自己的
`scone-devnet` 創世、近零 0..=2 bit PoW —
抹除 `~/.scone/devnet/` 並重新啟動即重置
鏈）。設定創世、PoW 難度與預設
data-dir（`~/.scone/<network>/`）；來自
另一網路的交易被型別化拒絕
（`WrongNetwork`）。

離線十六進位表面（`tx` 與 `submit` 子
指令）已移除（remove-tx-surface-v2）。
CLI 涵蓋使用者路徑（`domain register`、
`domain renew`、`domains`）**以及**作為
一級指令的 M8b 家族：`tld open`、
`tld transfer`、`tld revoke`、
`domain assign`、`domain transfer`。
`register` 聲明以網路難度挖其 PoW
（devnet 上毫秒級）。

**不存在任何 geo 或健康選項**：地理區**由建置
強制**（內嵌 DB-IP Lite 表 — 完整語義見
[`dns.md`](dns.md#geodns-geographic-routing)）；
地理路由無需任何選項即**恆開**，且解析器
不做任何健康檢查 — 它按原樣服務簽署記錄集
（端點可用性是端點營運者的事）。任何
`--geo-*`／`--health-*` 旗標都被 clap 硬性拒絕。

### `domain register|update`（M5）— 一條指令完成簽署交易

這些指令在單次呼叫中鏈結 build → 簽署
（keystore）→ 提交 →（確認）。

- **register**：組建簽署的 `RegisterDomain`
 （timestamp = now、devnet 中空證明）、
  提交它、等待鏈上確認（序號 ≥ 0、
  30 秒逾時）。已註冊的名稱在客戶端
  **及**中繼站側都被拒。
- **update**：記錄檔是唯一事實來源。序號
  （鏈上 + 1）與 `record_hash`（檔案記錄
  的正典 BLAKE3）是**衍生的**、絕非輸入：
  簽署者與驗證者不可能分歧。
  UpdateDomain 確認後，`SignedDnsRecord`
 （檔案的記錄 + 擁有者 + 對正典編碼的
  Ed25519 簽章）經 `put_record` 發布到
  DHT（它會對鏈重新驗證）。未註冊的
  網域、非擁有者身分或檔案錯誤乾淨地
  失敗、不觸碰鏈。

#### 記錄檔

文字、每行一筆記錄、`#` = 註解、空行
忽略；排序的正典編碼使檔案的順序對
承諾的雜湊沒有影響。檔案是**完整**
集合（update 是替換、不是合併）。型別：

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

型別化錯誤：未知型別、無效 IP／名稱
（附行號）、空集合、重複、>
`MAX_RECORDS_PER_SET`。

## `--genesis`（創始人）

`scone relay --genesis genesis.bin` —
**僅創始節點**（kickstarter 活動；檔案由
`scone-genesis` 二進位檔產生）。規範規則：

1. 檔案超過 1 MiB 在**讀取前**被拒，
   然後嚴格解碼（`decode_complete`：無
   尾隨位元組）。區塊 0 的雜湊**重新
   計算**、絕不從檔案讀取。
2. 語義驗證：高度 0、`consensus` 以請求
   網路的網路 id 開頭（`--network`）、
   無交易、正典配置清單。任何違反 =
   啟動時型別化錯誤（絕不 panic、守護
   行程不啟動）。
3. **全新**data-dir：鏈誕生於配置的
   創世 — 配置在**任何磁碟寫入之前**
   於 RAM 中套用，然後持久化（種子
   狀態 + 中繼鍵下的創世區塊）。因此
   第一個產出的區塊只能延伸配置的
   創世。
4. **已有生產力**的 data-dir：儲存庫的
   誕生創世是區塊 1 的 `prev_hash`
  （自我認證）。檔案雜湊相等 = 靜靜
   OK（該選項在**首次啟動後是多餘
   的**）；不同 = 型別化拒絕 — 節點經
   同步加入網路、絕不經 `--genesis`。
5. **不帶**該選項重啟：創始誕生的
   data-dir 經中繼鍵恢復其配置創世
  （每次開機重新綁定 RAM）。

固定的創世雜湊釘（反網路分歧）是
獨立的工作流。

### `scone-genesis` 二進位清單（kickstarter）

`scone-genesis` 二進位檔（crate
`scone-genesis`）準備 `genesis-manifest.json`
清單（`ALLOC-V1` 格式）然後產出
`genesis.bin`。規範規則：

- `scone-genesis new --network <network>`
  建立空清單；`add`／`remove`／`list`
  管理配置、`build` 產出 `genesis.bin`
 （+ 雜湊）、`verify` 重新驗證它。
- `add domain <name>` 要求該網域的
  **TLD 已在清單中配置**（否則立即
  拒絕 — 鏈在創世套用時本就會拒絕
  它）；其 `valid_until` 是**自動的**：
  `born_at + 1 year`（清單的
  default_validity）、絕不人工輸入。
- `add tld <name>`：**預設開放**（無主、
  自助、1 年下限、線上格式擁有者為零 —
  類 `0x03`）；`--private` = 私有
 （擁有者、創世任期 `born_at +
  TLD_TERM_SECS` — 3 年、標準生命週期：
  `RenewTld`、60 天寬限）。
- 依名稱正典排序（相等內容 → 相同區塊
  → 相同雜湊、無論插入順序）；重複
  被拒。

## 限制與記憶體保證

| 資源 | 界限 |
|---|---|
| P2P 框 | 解析前的 `MAX_MESSAGE_LEN`（1 MiB） |
| 記憶體池 | 4096 tx（超過型別化拒絕）；每網域 `MAX_PENDING_PER_DOMAIN = 4` 筆待決 tx |
| 記憶體池寬限 | `MAX_GRACE_ATTEMPTS = 10` 次因非確定原因被沖銷 tx 的重新插入（40 秒/循環約 7 分鐘），然後逐出 |
| 分叉停等 | `PARKED_CAP = 2048` 個競爭區塊 |
| 累積檢查點 | `MAX_PENDING_CHECKPOINTS = 256`（逐出：最小資料雜湊） |
| 同時 DHT 等待者 | `MAX_DHT_WAITERS`（256）；DHT 查找有界 30 秒（`DHT_LOOKUP_TIMEOUT`） |
| 每同步回覆區塊數 | 每往返 1 個；每 peer ≤ 10,000 批次（`MAX_SYNC_ROUNDS`）；請求上的協定界限 `MAX_BLOCKS_PER_REQUEST = 128` |
| DHT 記錄 | `MAX_DHT_CACHE_ENTRY`（64 KiB） |
| 持久 DHT 快取（磁碟） | 雙重 LRU 界限：`DHT_CACHE_MAX_ENTRIES`（100,000 條）**及** `DHT_CACHE_MAX_BYTES`（2 GiB）— 先觸及者依金鑰序逐出；GC 逐出在與區塊**同一**交易中（被鏈 GC 殺死的網域絕不留孤兒條目） |
| DHT 負快取 | `DHT_NEGATIVE_TTL_SECS`（30 秒）、FIFO 上限 `DHT_NEGATIVE_CAP`（4,096） |
| RAM 中的鏈 | 創世 + 尖端視窗（歷史由儲存庫服務） |

防迴圈規則（H2）：交易只有在**確實
插入**記憶體池（首次見到）時才被重新
廣播；重複（已在池中或已在鏈中）絕不
重新轉發。已是正典的區塊在 `push_block`
之前被偵測並忽略（不重新儲存、不重新
廣播）。

開放決策：中繼站的持久網路身分、廣播
gossip（目前：對每個已知 peer 的單播
request-response）。自初稿以來已解決：
真實分叉選擇（停等 + reconsider_parked、
見上）、PoW 共識（簽署生產者 + 錨點
委員會）、DNS 解析器（`--dns` 表面、見
dns.md）、控制通道（RPC→p2p 統一：不再
有 JSON TCP）。

## 優雅關閉（SIGTERM / SIGINT）

`scone relay` 守護行程（前景）經
`tokio::signal` 捕捉 SIGTERM 與 SIGINT：
收到訊號時，`Relay::run` future 被取消
（二進位檔中的 `tokio::select!`），這
drop libp2p Swarm 並關閉 QUIC socket。
tokio 執行時隨後以有界關閉（≤ 2 秒）
停止，drop 分離的 DNS 任務（UDP/TCP）
並關閉其 socket — 目標是 `kill` 不留
孤兒 UDP socket（testnet 60 觀察到的
案例）。訊號驅動的關閉記錄
`relay shutting down (signal)`（INFO、
stderr）並以退出碼 **0** 離開：被請求的
關閉不是失敗。若處理器安裝失敗，訊號
的預設行為適用（降級、絕不 panic）。


## Tor 傳輸（開放議題）

QUIC/UDP 無法穿越普通 SOCKS 代理，因此
中繼站今天無法輕易跑在 Tor 之上。原生
libp2p Tor 傳輸（洋蔥位址的中繼站）是
被追蹤的前進路徑 — 見網站的抗審查指南
了解現況與 `.tor` CNAME 橋（隱藏服務的
可讀名稱、今日即支援）。

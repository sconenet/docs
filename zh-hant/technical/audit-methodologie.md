# 安全稽核方法論 — Scone（主網前）

工作文件。目標：對生產部署的信心。
跨 crate 參照：AGENTS.md 的硬性規則（不自行發明
密碼學、嚴格解碼、重新計算雜湊、決定性）是
本稽核的需求規格。

## 0. 範圍與威脅（在重讀任何一行程式碼之前）

對手模型：一個任意敵對的 p2p 節點（可透過 QUIC/UDP
傳送任何位元組）、一個敵對的交易提交者、一個好奇的
中繼站營運者、一個被竊的 data-dir。設計上排除：
終端客戶端（DNS 本質上是公開的）、整體可用性
（混沌測試已做過）。

資產表，依關鍵性排序：
1. **所有權**（狀態鏈：Register/Update/Transfer/Assign）—
   偽陽性 = 網域被竊。
2. **簽章／金鑰持有**（keystore、Ed25519、OwnerId
   衍生）。
3. **不可信輸入驗證**（線上解碼、限制、
   panic）。
4. **資源 DoS**（DNS 快取、記憶體池、DHT、同步）。
5. **決定性／共識**（reorg、GC、狀態復活）。

## 1. 自動化靜態稽核（第 1 天 — 機械性）

- `cargo audit`（相依套件 CVE：libp2p/quinn/rustls/redb/ed25519）。
- `cargo geiger` + `unsafe` 檢視（預期 ≈ 0）。
- `cargo clippy --workspace --all-targets -- -D warnings`
  （已綠，保持為 CI 閘門）。
- 對解碼邊界做模糊測試（第 2-3 天）：
  對 `scone_protocol::decode_complete`（Message、
  Transaction、Block、SignedDnsRecord、Checkpoint）跑
  `cargo-fuzz` — 這是「對不可信資料絕不 panic」必須
  成立的表面。
  預算：連續 24 小時模糊測試，語料來自既有
  測試。退出標準：零崩潰、零逾時、解碼
  分支覆蓋率 > 95%。
- 對 DNS 線上解析器（dns.rs 的 `parse_query`、
  `extract_answers`）與記錄檔解析器（domain.rs）做
  模糊測試。

## 2. 依元件的目標性人工審查（第 4-8 天）

依資產關鍵性排序，1 個元件 = 1 份檢查清單：

### 2.1 scone-crypto + 身分衍生（第 4 天）
- OwnerId = BLAKE3(pubkey)：檢查位元組正典化（沒有
  歧義表示 → 2 把金鑰、1 個擁有者）。
- Ed25519：嚴格的反序列化（非正典金鑰被拒？）、
  帶情境分離的驗證（每個用途有自己的領域
  標籤：SCONE-TX-SIG-V1、SCONE-BLOCK-V1、DHT 記錄 —
  跨協定重播不可能）。
- Keystore：通關密語 → 金鑰（KDF？迭代次數？）、歸零
  （zeroization）、檔案權限、密文上的 padding oracle。

### 2.2 交易套用（第 5 天）— 核心
對每個 Transaction 變體（目前 13 個）：
- 交易在重播下是否冪等（唯一 txid、nonce/序號）？
- 所有權檢查是否先於任何效果？（簽署者 == 鏈上
  學得的擁有者，絕不信任交易本身聲稱的。）
- UpdateDomain 序號：精確的 current+1（而非 ≥）— 檢查
  對間隙的拒絕與對重播的拒絕。
- Renew：時間界限（≤ 3 年、> now）— 時間來自
  區塊，絕非本機時鐘（規則已定，檢查每個
  呼叫點）。
- AssignDomain/TransferTld：受指派者的公鑰是否在所有
  路徑上都被學得（無公鑰池陷阱 — 參見 skill）。
- GC／到期：不可能復活（precheck-OK/lookup-OK/push-fails
  悖論）— 檢查 gc_expired_journaled。

### 2.3 網路層（第 6 天）
- 每個遠端位元組在配置記憶體之前都先經過嚴格的
  有界解碼器（limits.rs 的 MAX_*）— 找出無界的
  `Vec::with_capacity(input_len)`。
- requery/sync GetBlocks：每個 peer 與每個循環的界限
  （已有：每請求 128 區塊）— 檢查沒有重複下載
  迴圈。
- Gossip Record：在快取／gossip 之前先鏈上重新驗證
  （由我的 record_dht 測試驗證）— 也檢查 fanout 上限。
- kad DHT：敵對 peer 能否強制儲存 N 筆記錄（每 peer
  配額？）或針對性地逐出誠實記錄？

### 2.4 DNS 表面（第 7 天）
- 快取：注入不可能（正典 (qname,qtype) 金鑰 — 沒有
  大小寫／編碼碰撞）、硬性記憶體預算（已測試）、
  備援路徑無快取中毒（TXID + 問題回顯 — 驗證
  上游無法被遠端偽造）。
- 回答：誠實的 AA、誠實的 TC、無放大（有界的
  回應／請求比 — MAX_INFLIGHT_UDP 已在，檢查
  TCP 複製）。

### 2.5 儲存與創世（第 8 天）
- redb：磁碟損毀 → 型別化錯誤，不 panic（簽署端已有
  diskfail 測試 — 中繼端的對應呢？）。
- 創世：雜湊涵蓋整份清單；兩個不同的
  創世 = data-dir 拒絕（已測試）；驗證不可能
  在事後對已同步的 data-dir 注入創世。

## 3. 對手測試（第 9-12 天）— 補寫缺失的

對人工審查的每個發現，一個證明它的測試：
- 模擬中的敵對節點（SimNet 加一個傳送畸形／
  不一致訊息的節點）— 為 SimNet 擴充「被動
  拜占庭」角色；
- 真實的 30 節點網格含 3 個敵對節點（隨機位元組、
  無效交易爆發、偽造的 DHT 記錄）— 混沌電池
  已在，加上攻擊者角色；
- 交叉檢查：解碼的兩個實作（protocol.md 文件是
  規範性的 — 稽核者僅依文件實作線上
  格式並比對位元組；任何分歧 = 文件或
  程式碼的 bug）。

## 4. 外部稽核（若有預算）— 在 1-3 之後

對廠商的最小有利範圍：scone-crypto、
scone-protocol（線上格式）、交易套用（state.rs），約 8-10 k LOC。
形式：2 週，1 位資深密碼學稽核員 + 1 位審查者。交付物：
附嚴重度、PoC、已審修復的報告。依賴既有的
規範性文件（docs/technical）作為需求
規格的基礎 — 這是實際的成本優勢。

## 5. 退出標準（主網 go/no-go）

- [ ] cargo-fuzz 對 5 個解碼器 24 小時無崩潰
- [ ] 審查 2.2 完成：每個交易變體都有審查者
      簽核的表單（所有權／重播／時間／效果）
- [ ] 含敵對節點的 30 節點網格：48 小時無尖端
      分歧或快取損毀
- [ ] 文件／程式碼分歧：零（由雙線上實作
      驗證）
- [ ] 阻斷性 CVE：零（cargo audit）
- [ ] 密碼學／線上／交易範圍的外部稽核完成（若有
      預算）

## 建議的執行順序

1→2.1→2.2（核心，60% 的風險）→2.3→2.4→2.5→3 與
各審查並行（對手測試在發現當天撰寫，
而非之後）。

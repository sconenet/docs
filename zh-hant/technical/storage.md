# 儲存 — 本機節點持久化（redb）

> 規範性文件。本文件與
> `crates/scone-storage/src/` 之間的任何分歧都是
> bug（找出事實來源、修正之）。實作參考：crate
> `scone-storage`（格式 v1）、`redb` 4.2 後端。

## 目的

節點的內嵌本機持久化：正典區塊、網域
狀態、DHT 快取、中繼資料。儲存庫是
**持久化**、絕非權威：RAM 狀態
（[`ChainState`]、共識的事實來源）在啟動時
由儲存庫重建，而每次寫入都是**差量**
（被區塊修改的網域）、絕非狀態的完整
副本。

橫切約束（面向使用者、每個里程碑）：
**最小記憶體佔用**。大容量（區塊、網域
狀態、DHT 快取）住在 redb 中並經由游標／
範圍存取 — 絕不完全載入 RAM、絕不
無謂地複製。

## 邊界：`NodeStore` trait

`crates/scone-storage/src/lib.rs` 定義其餘
程式碼（區塊鏈、未來的 M4 中繼站）可見的
唯一儲存面：

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` —
  原子附加（見下）；
- `block_at_height(h)` / `block_by_hash(hash)` —
  點讀；
- `tip() -> (height, hash)` — O(1)、從
  `meta` 讀取；
- `put_domain_state` / `domain_state` /
  `domain_count` — **維護的**計數器
  （`meta["domain_count"]`）、絕非
  `COUNT(*)`；
- `iterate_domains(after, max) -> DomainPage` —
  基於游標的分頁（預設每批 100）、
  `DomainId` 的位元組升序；
- `put_tld_state` / `tld_state` / `tld_count` /
  `iterate_tlds` — TLD 登記冊的相同契約
  （`tld_count` 維護、有界游標）（M7d）；
- `put_dht_cache` / `dht_cache` — 不透明的
  `SignedDnsRecord` 位元組；
- `resolve_name(canonical)` /
  `resolve_tld(tld)` — 持久的 Name→Id
  索引（P0.2）、O(1) 點查找、不迭代
  網域（預設 `None` = 無索引的後端、呼叫者
  退回 `iterate_domains` 或鏈狀態）；
- `tld_floors_all` / `snapshot_tld_floors_all` —
  讀取開放 TLD 的生命下限（owner-option：
  專屬資料表、活躍的 `tld_floors_v1` 與
  凍結的 `snapshot_v1_floors`）；
- `rewind_to_height(h)` — reorg：在**一個**
  交易中移除 `h` 以上的區塊並把
  `tip`／`tip_height` 帶回 `h`
  （`persist_state_snapshot` 的前置、見下）；
- `meta_get` / `meta_set`；
- `snapshot_meta` / `snapshot_domains` /
  `snapshot_tlds` — 開機狀態快照讀取
  （M6b；空預設 = 無快照的後端、
  `load_chain` 隨後退回完整載入）。

規則：

- **API 中沒有 redb 型別**：只有位元組
 （`&[u8]`、`[u8; 32]`）與核心型別。後端可
  替換。
- **無 async、無內部可變性**：寫入取
  `&mut self`（redb 序列化寫入者）、讀取取
  `&self`（每次呼叫一個短讀交易）。
- API 絕不重新驗證區塊：傳給
  `append_block*` 的雜湊是呼叫者
  （`Blockchain::push_block`）提供的**重新
  計算**雜湊。

## redb 資料表架構

| 資料表 | 金鑰（排序） | 值 |
|---|---|---|
| `blocks_by_height` | `u64` 高度（有序 b-tree） | `hash(32) ‖ height(8 BE) ‖ canonical block` |
| `blocks_by_hash` | 區塊雜湊（32 B） | 區塊的正典編碼 |
| `domains` | `DomainId`（32 B、位元組序） | `DomainStateBytes`（見下） |
| `tlds` | `TldId`（32 B、位元組序） | `TldStateBytes`（見下、M7d） |
| `dht_cache` | `DomainId`（32 B） | `SignedDnsRecord` 位元組（不透明） |
| `names_v3`（P0.2） | 索引鍵（32 B、位元組序） | 目標 id（32 B） |
| `name_reverse_v3`（P0.2） | 目標 id（32 B、位元組序） | `tag(1) ‖ canonical name` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta`（M6b） | `u64` = 0（單槽） | `height(8 BE) ‖ tip_hash(32)` = 恰好 40 B |
| `snapshot_v1_domains`（M6b） | `DomainId`（32 B、位元組序） | `DomainStateBytes`（與 `domains` 相同格式） |
| `snapshot_v1_tlds`（M6b） | `TldId`（32 B、位元組序） | `TldStateBytes`（與 `tlds` 相同格式） |
| `tld_floors_v1`（owner-option） | `TldId`（32 B、位元組序） | `u64`（到期下限、redb 隱式 BE） |
| `snapshot_v1_floors`（M6b） | `TldId`（32 B、位元組序） | `u64` — 下限的凍結副本（快照**時**的狀態、非尖端） |

額外的 `meta` 金鑰（P0.3/P0.4）：

| 金鑰 | 值 |
|---|---|
| `verified_snapshot_v1`（P0.3） | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — 已驗證匯入快照的標記（保留、如 `tip`） |

`meta` 金鑰：

| 金鑰 | 值 |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION`（1） |
| `tip` | 尖端區塊的雜湊（32 B） |
| `tip_height` | `u64 BE`（使 `tip()` 為 O(1)） |
| `domain_count` | `u64 BE`（維護的計數器） |
| `tld_count` | `u64 BE`（維護的計數器、M7d） |

`blocks_by_height` 的 `hash ‖ height` 標頭使
每個索引自含（無需讀區塊即可驗證）；兩個
索引在同一交易中更新。

### 創世慣例

創世區塊是決定性的（協定常數）且**絕不
以位元組儲存**。全新的儲存庫有
`tip == (0, genesis_hash)`；第一個附加的
區塊高度為 1。創世雜湊恆重新衍生
（`scone_blockchain::genesis_hash()`）、絕不
讀取。

## `DomainStateBytes` 編碼格式（M8b）

依開頭版本標籤為 65 或 97 位元組、固定寬度
大端整數（正典、可逐位元組比較）：

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) ‖ mutations(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash present → exactly 97 bytes
tag = 0x02 : record_hash absent  → exactly 65 bytes
```

`mutations` 是命名空間單調的變更計數器
（CAS 反重播世代 — 見 `transactions.md`、
變更計數器）。M8b 的編碼（57/89 位元組、
無計數器）**不合法**：嚴格解碼 →
`Corrupted`（testnet 重置；計數器前的
儲存庫經重播再生）。

- `registered_at`／`valid_until`（M8b）：
  註冊的到期（聲明時 1 年、可續期、見
  `/docs/en/technical/blockchain.md`）。

M7 編碼（41/73 位元組、無到期）自 M8b 起
**不合法**：嚴格解碼 → `Corrupted`。（M7
儲存庫經重播再生：`load_chain_replay`。）

- `owner`：`OwnerId`（32 B）；
- `sequence`：最後套用的 `UpdateDomain`
  序號（`RegisterDomain` 之後立刻為 0）；
- `record_hash`：DNS 記錄集當前的鏈上承諾
  （只要尚未套用 `UpdateDomain` 則為
  `None`）。

**嚴格**解碼（`DomainStateBytes::decode`）：
未知標籤、錯誤長度或多餘位元組 →
`StorageError::Corrupted`、絕不 panic。

## `TldStateBytes` 編碼格式（owner-option）

依枚舉變體而定的可變長度 — **私有** TLD
50 位元組、**開放** TLD 10 位元組（兩者皆
無序號也無記錄雜湊）：

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
                [‖ mutations(8)]
```

- `flag`（owner-option、`TldState` 枚舉）：
  `0x00` = `TldState::Open`（無主 — 自助 +
  PoW、無擁有者位元組）、`0x01` =
  `TldState::Private`（後接 32 個擁有者
  位元組 + 8 個計數器位元組）。嚴格正典 —
  任何其他位元組 → `Corrupted`。「開放 ⟺
  無擁有者」的不變量是**結構性的**：兩種
  狀態不可能混淆。
- **`until`（8 位元組、BE64）**：TLD 自己的
  時鐘 — 私有任期結束（聲明／續期 + 3 年）
  或開放死亡時程（誕生時 1 年下限、被每個
  活躍網域推遲）。它再次成為條目的屬性
 （TLD 生命週期變更）。衍生的
  `tld_floors_v1` 資料表（+ 其凍結副本
  `snapshot_v1_floors`）仍追蹤下限、在重播
  時重建並於開機時還原 — 重啟的節點殺死
  與連續節點相同的 TLD。

M7d（33 位元組、標籤 `0x01`）、M8b（34
位元組、標籤 `0x02`、布林 `open`）、舊標籤
`0x03`（內嵌 `until`）與計數器前的標籤
`0x04` 佈局（42/2 位元組、無計數器）的
編碼都**不合法**：嚴格解碼 → `Corrupted`
（主網前、經重播再生即為遷移路徑）。

標籤只是為了格式演進：未來的佈局提高
標籤、舊讀取器以 `Corrupted` 失敗而非
猜測。**嚴格**解碼
（`TldStateBytes::decode`）：未知標籤、錯誤
長度、多餘位元組或非正典旗標 →
`StorageError::Corrupted`、絕不 panic。

## 原子性保證

`append_block_with_state` 在**單一** redb
`WriteTransaction` 中寫入：

1. `blocks_by_height` 中的區塊（前置檢查：
   該高度不得以不同內容存在、否則
   `StorageError::Conflict`）；
2. `blocks_by_hash` 中的區塊；
3. 每個網域狀態差量（`domains`）；
4. `domain_count` 計數器（僅以**新**網域數
   遞增；既有網域的 `UpdateDomain` 不
   遞增它）；
5. 每個 TLD 登記冊差量（`tlds`、M8b：
   聲明、轉移、開放／封閉）— **在同一
   交易中**：寫了區塊卻沒寫其 TLD 變更在
   結構上不可能；
6. `tld_count` 計數器（僅新 TLD）；
7. 每個**移除**（M8b）：被決定性 GC 到期的
   網域（`removed_domains`）與被撤銷的
  TLD（`removed_tlds`）在同一交易中離開
   儲存庫、計數器遞減 — 重啟絕不能復活
   過期的註冊；
8. P0.2 的**名稱索引**差量（upsert 與移除、
   見下）— 相同保證：當機絕不留下一個
   指向不存在網域的索引條目；
9. `tip` 與 `tip_height`。

redb 以 WAL 風格提交：寫入中途當機留下
**前一個**一致狀態。**因此寫了區塊卻沒寫
其狀態在結構上不可能** — 這是本里程碑的
關鍵不變量。

額外保證：

- **單調性**：以 ≠ `tip_height + 1` 的高度
  附加 → `StorageError::NonMonotonicHeight`、
  什麼都不寫（在開啟寫交易*之前*檢查）；
- **冪等性**：以相同 `(height, hash)` 對重新
  附加當前尖端區塊是 no-op（快路徑比較
  高度與雜湊；高度相同時、相同的區塊位元組
  也是靜靜的 no-op、不同的位元組則是
  `Conflict`）；
- **損毀的資料庫**：截斷／污染的檔案 →
  開啟或讀取時的型別化錯誤
  （`StorageError::Database`／`Corrupted`）、
  絕不 panic（已測試）；
- **格式版本**：`format_version` ≠ 1 →
  `StorageError::UnsupportedFormat`（需要
  明確的遷移路徑、絕不猜測）。

## 區塊鏈整合（`integration.rs`）

角色分離：RAM 中的 [`ChainState`] = 共識
權威；儲存庫 = 持久化。所選策略（如文件
所載、最簡單者）：**每個區塊持久化狀態** —
`store_block` 寫入區塊 + 被**該區塊觸及**之
網域的狀態（差量）。完整重播仍可能並
作為控制／修復路徑。

- `store_block(store, chain, block, hash)`：
  正典編碼區塊、從 RAM 狀態（已被
  `push_block` 更新）讀取被觸及網域**及**
  已聲明 TLD 的最終狀態、原子附加。
- `load_chain(store)`：**不重播** — 讀取
  尖端 + 尖端區塊（O(1) 區塊讀取）、以
  100 頁（`DOMAIN_PAGE`）還原網域狀態
  **並以 100 頁**（`TLD_PAGE`、M7d — 關閉
  重啟窗口：沒有它、重啟的節點忘記每個
  已聲明的 TLD、以 `UnknownTld` 拒絕該
  TLD 下的每個 `RegisterDomain`、儘管其
  磁碟持有完整狀態）還原 TLD 登記冊。
  RAM 鏈只保留創世 + 尖端；歷史區塊留在
  儲存庫並由它服務。自 M8 起、
  `load_chain` 隨後重建 **TXID 防重播
  索引**（見下）。
- `load_chain_replay(store)`：逐一再重播
  所有儲存的區塊（絕不全部在 RAM）、
  完整驗證；所得尖端必須等於儲存的
  尖端、否則 `Corrupted`。
- `rebuild_replay_index_from_store(store,
  chain, tip_height)`（M8）：重掃最後
  `min(REPLAY_WINDOW_BLOCKS, tip_height)`
  個持久化區塊（一次只在 RAM 一個區塊）
  並經公用 API
  `Blockchain::rebuild_replay_index`把每個
  TxId 以**其原始收錄高度**重新插入鏈的
  索引（不存取私有欄位）。尖端下缺少
  區塊或窗口內無法解碼 → `Corrupted`
  （那些位元組已用於建鏈：此處的失敗
  表示儲存庫受損）。
- `touched_domains(block)`：被區塊交易
  修改的網域 id、去重、區塊順序。
- `touched_tlds(block)`：被區塊突變的 TLD
  id（`RegisterTld`、`TransferTld`、
  `RevokeTld`、`SetTldOpen`）、去重、區塊
  順序（M8b）。
- `store_block_with_removals(...)`：
  `store_block` 的變體、另接收被區塊 GC
  移除的網域（M8b）以進行上述原子寫入。
  中繼站從
  `Blockchain::push_block_with_gc` 餵入
  此清單（套用結果暴露
  `gc_removed_domains`）。**DHT 快取的
  GC 逐出**：被移除網域的 `dht_cache`
  條目在與區塊**同一**交易中刪除（附加
  差量攜帶 `dht_cache` 資料表）— DHT
  記錄絕不活得比它服務的網域久、過期
  條目絕不永久累積（測試：
  `gc_evicts_the_dht_cache_entry`）。

中繼站（M4）將使用：啟動時
`load_chain`、然後對每個接受的區塊
`push_block`（RAM）再 `store_block`
（磁碟）。

## Reorg 持久化（`rewind_to_height` +
`persist_state_snapshot`）

reorg（切換到更強的姊妹分支）無法經過
單調附加 — 寫入獲勝字尾會以
`NonMonotonicHeight` 失敗。專屬路徑
（黏性少數修正、中繼站 + 儲存庫）：

1. `rewind_to_height(fork_height)` — 一個
   redb 交易移除分叉點以上的區塊並把
   `tip`／`tip_height` 帶回（名稱索引與
   活躍狀態在下一步修正）；
2. `persist_state_snapshot(store, chain)` —
   從 reorg 後的 RAM 狀態匯出重寫活躍
   資料表（`domains`、`tlds`、名稱索引、
   TLD 下限）：活躍資料表在開機時**絕不**
   描述**輸**分支；
3. 被採納字尾的單調重附加（普通差量）。

此路徑上的持久化失敗被記錄並降級
（儲存庫在開機時經重播痊癒）— 絕不
致命。

## 開機狀態快照（M6b）

如上所述的 `load_chain` 仍是 O(狀態大小)：
不重播任何區塊、但整個 `domains` 資料表
被重讀。是「每區塊持久化狀態」的路徑使
重播不必要。M6b 快照處理另一種成本：
RAM 狀態必須**經重播**重建的情況（修復、
控制、或無每區塊持久化狀態的後端）。
它提供 **O(tip − H)** 的開機 — 只重播
快照之上的字尾。

### 寫入

- **區間**：`SNAPSHOT_INTERVAL = 64`
 （`scone-storage` 的公用常數）。觸發是
  區塊 `k·SNAPSHOT_INTERVAL + 1` 的附加：
  該交易凍結活躍資料表**在套用該區塊
  之前**的狀態、即區塊
  `k·SNAPSHOT_INTERVAL` 之後的正典狀態。
- **絕不在尖端**：快照因此在寫入當下至多
  位於 `tip − 1`、恆嚴格低於尖端。隨後的
  當機絕不能留下「高於」已持久化鏈的
  快照。
- **原子**：凍結（清除 + 把 `domains`／
  `tlds` 資料表複製到 `snapshot_v1_*`、寫入
  `snapshot_v1_meta`）發生在與觸發區塊附加
  **相同**的 redb 交易中 — 當機留下的是
  完整的舊快照、或是連同其區塊完整的新
  快照（redb WAL）。
- **單槽**：每個快照**取代**前一個（資料表
  清空再重抄）— 無累積、無需管理的
  修剪、磁碟佔用受 ~2× 狀態約束。
- 凍結狀態的格式嚴格是
  `DomainStateBytes`／`TldStateBytes` 的
  （重用、無新編碼）；只有
  `snapshot_v1_meta` 列是新格式
  （`height(8 BE) ‖ tip_hash(32)`、嚴格
  解碼：任何其他長度 → `Corrupted`）。

### 開機時讀取（`load_chain`）

1. 尖端如常驗證（從尖端區塊標頭重新計算
   的雜湊 vs `meta["tip"]`）。
2. 若 `snapshot_meta()` 存在、其高度 `H`
   滿足 `0 < H < tip_height`、且儲存在
   `H` 的區塊**重新計算**的雜湊等於快照的
   `tip_hash`（**錨** — 與持久化鏈無關、
   被竄改、或來自另一檔案的快照在此被
   偵測）、凍結狀態以 100 頁還原、然後
   只重播區塊 `H+1..=tip`（完整驗證、與
   活躍區塊相同）。
3. 重播到達的尖端必須等於儲存的尖端。
4. **此路徑上的任何失敗（快照缺失、不可用
   的高度、無效錨、無法解碼的位元組、
   重播失敗）靜靜降級為 M6b 之前的完整
   載入**（讀活躍資料表、不重播）— 壞
   快照絕不使開機失敗。向後相容是「無
   快照」的特例：M6b 之前寫入的儲存庫
  （沒有 `snapshot_v1_*` 資料表）相同地
   載入、`snapshot_meta()` 為 `None`。

重播的字尾受 `SNAPSHOT_INTERVAL` 約束
（觸發時至多 64 個區塊 + 之後挖出的
區塊）：開機是 O(tip − H)、且 H 保證
新近。

## 開機時重建 TXID 防重播索引（M8）

防重播索引（`included`、TxId → 收錄高度、
窗口 `REPLAY_WINDOW_BLOCKS = 256` — 見
`/docs/en/technical/blockchain.md`）是**正典
鏈的純函數**：它絕不持久化。由於
`load_chain` 不重播歷史、M8 之前還原的
鏈以**空**索引啟動 — 重啟的節點接受仍在
其重播窗口內的任何交易重新收錄（已載明
的限制；狀態規則仍是唯一防線）。

M8 關閉此窗口：還原之後（完整路徑**或**
快照 + 字尾重播）、`load_chain` 呼叫
`rebuild_replay_index_from_store`，它重讀
最後
`min(REPLAY_WINDOW_BLOCKS, tip_height)`
個持久化區塊 — 一次只在 RAM 一個區塊、
如同處處一樣地記憶體有界 — 並把每個
TxId 以**其原始高度**重新插入。因此少於
256 個區塊的儲存庫重建其整個索引。

性質：

- **與活躍節點相同的決策**：相同條目、
  相同修剪規則（`h > tip − 256`）— 重啟
  的節點與從未停止的節點拒絕相同的
  重播（已測試：精確的索引計數及與修剪
  後活躍索引的相等）。
- **修剪語義不變**：比窗口舊的交易仍被
  索引接受（然後在適用時失敗於狀態
  規則）— 與之前完全相同。
- **快照不凍結索引**：快照的字尾重播
  重建 H 之上的條目、M8 重掃補上窗口中
  H **之下**的部分（與 owner_keys／grace
  相同契約：重建、絕不持久化）。
- **成本**：開機時 ≤ 256 次點區塊讀、
  O(窗口) — 與鏈齡無關。
- 窗口**內**缺失或無法解碼的區塊使開機
  以 `Corrupted` 失敗（那些位元組已用於
  建鏈）；窗口之外、歷史區塊不被觸碰。

`load_chain_replay`（零信任）經其完整重播
以相同方式重建索引 — 兩條路徑收斂。

## 持久的 Name → Id 索引（P0.2）

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖
canonical name)` 不可逆：解析名稱
（中繼站/DNS）需要持久索引。P0.2 把它
加進儲存庫、紀律與其餘部分相同。

### 架構

兩個伴隨資料表、一起寫入與抹除：

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ canonical name)  (32 B)
                val  = target id (32 B : DomainId or TldId)

name_reverse_v3 key  = target id (32 B)
                val  = tag(0x01 domain | 0x02 TLD) ‖ canonical name
```

- `names_v3` 的金鑰是**名稱的雜湊、而非
  明文名稱**（緊湊的索引資料表、傾印
  不產生名稱 gossip）。金鑰是**名稱的純
  函數**：索引隨時可從鏈重建
  （`rebuild_name_index`）、不為它持久化
  額外的中繼資料。
- `name_reverse_v3` 儲存名稱（恰好一次）、
  因為移除路徑（過期網域的 GC、
  `RevokeTld`）只知道 **id**：要抹除的
  雜湊金鑰總得有辦法找到。標籤區分兩個
  命名空間。
- 嚴格分離：`resolve_name` 絕不回答 TLD
  條目、`resolve_tld` 絕不回答網域條目
  （由反向資料表的標籤決定；`DomainName`
  有 ≥ 2 個標籤、TLD 有 1 個 — 兩個金鑰
  衍生共享 `names_v3` 空間但不共享結果）。
- `DomainId` 維持 32 位元組、絕不縮減
  （專案硬規則：共識身分、抗碰撞）。

### 交易一致性

upsert（`RegisterDomain`、`AssignDomain`、
`RegisterTld` — 交易以明文攜帶名稱）與
移除（GC、撤銷 — 裸 id）由區塊的
`StateDelta`（`name_upserts` /
`name_removals`）攜帶、在與區塊及狀態
寫入**相同的 redb 交易**中套用。因此
當機既不能留下指向不存在網域的索引
條目（無幽靈）、也不能留下沒有條目的
已註冊網域。失敗的附加什麼都不寫
（已測試）。具名變體
`put_domain_state_with_name` /
`put_tld_state_with_name` /
`remove_domain_state` 為修復工具提供相同
原子性。

`store_block` / `store_block_with_removals`
從區塊的**相同**交易衍生名稱差量 —
中繼站介面無額外參數。

### 重建（`RedbStore::rebuild_name_index`）

索引是正典鏈的純函數、如 M8 防重播
索引。`rebuild_name_index` 從**持久化區塊**
重建它（`DomainStateBytes` **不**含名稱 —
只有擁有者/序號/到期 — 因此事實來源是
交易流、`RegisterDomain` 與 `AssignDomain`
在那裡以明文攜帶正典名稱）：

1. 第一遍：掃描 `blocks_by_height`
   創世→尖端、一次只在 RAM 一個區塊；
   收集在
   `RegisterDomain`／`AssignDomain`／
   `RegisterTld` 中見到的 `name → id`
   綁定；
2. 第二遍：只過濾到**活躍**目標（持久化
   狀態是權威）— 註冊後被 GC／過期的
   網域不被重新索引；
3. 第三遍：在**一個**交易中抹除 + 完整
   重寫兩個資料表（當機 → 舊或新的完整
   索引）。

缺失或無法解碼的區塊 → `Corrupted`
（與建鏈相同的位元組）。成本：RAM 中
O(見過的互異名稱)（每名稱 ≤ 253 B）、
讀取 O(區塊) — 是修復工具、不是開機
路徑。

### 索引不是什麼

- 不是權威：RAM 鏈狀態仍是唯一權威；
  索引是可重建的快取（分歧是交易性
  bug、可由 `rebuild_name_index` 與它
  比較來偵測）。
- 不凍結於 M6b 快照：
  `names_v3`／`name_reverse_v3` 資料表是
  活躍的、跟隨每個區塊的差量；狀態
  快照不內嵌它們（恢復時經差量重建、
  或以 `rebuild_name_index` 作為修復）。

### 快照不承諾什麼

RAM 狀態 `ChainState` 攜帶不由共識承諾的
內部結構（擁有者金鑰、寬限窗口）。它們
**不**被凍結：字尾重播經套用交易重建
它們（決定性、相關窗口的內容與完整重播
相同）。TXID 防重播索引遵循相同契約
（M8、見前節）：開機時重建、絕不凍結。
與 `load_chain_replay` 的逐位元相等由測試
在正典狀態（state_root + SMT）上驗證。

## VERIFIED 狀態快照 — 分塊 + 清單 + 密碼學驗證（P0.3）

上面的 M6b 快照加速信任自己磁碟之節點的
開機。P0.3 處理不同需求：在節點之間運輸
狀態（引導新節點、修復）而不信任運輸。
格式與驗證位於
`crates/scone-storage/src/verified_snapshot.rs`
+ 根重建於
`crates/scone-blockchain/src/snapshot_verify.rs`。

### 信任模型

快照攜帶**已最終化**區塊的狀態 — 由
委員會法定簽署的檢查點
（`CheckpointData.state_root`、O(1) 的
`state_root_smt` 根）。檢查點的真實性
（Ed25519 簽章法定）由鏈層在匯入**之前**
驗證。匯入本身以重新計算驗證快點 ↔
檢查點的一致性：

1. **每頁的雜湊**：`blake3("SCONE-SNAP-PAGE-HASH"
   ‖ 完整頁編碼)` 必須等於清單宣告的雜湊
  （一個偽造位元組 → 拒絕）；
2. **條目總和 == 清單計數器**（缺頁或
   錯誤計數器 → 拒絕）；
3. **從匯入條目重新計算的 SMT 根 ==
   `checkpoint.data.state_root`** — 關鍵的
   密碼學檢查：整個狀態（網域 + TLD）
   被重新折疊進一個新鮮的 SMT
  （`snapshot_verify::recompute_state_root`、
   葉 `SCONE-LEAF-DOM-V1`／
   `SCONE-LEAF-TLD-V1` 與活躍路徑完全
   相同）、且根必須等於委員會簽署的。
   偽造、截斷或不完整的快照在任何
   寫入**之前**被拒；
4. **結構一致性**：清單的
   `height`／`tip_hash`／`state_root` ==
   檢查點的那些；嚴格遞增的 id（無
   重複）；連續的頁索引；DoS 界限
  （每頁 ≤ 10,000 條、≤ 2^20 頁、總預算）。

驗證失敗 → `StorageError::SnapshotRejected`、
儲存庫保持嚴格不變（一切都在寫交易
之前驗證）。

### 序列化格式（嚴格、大端、無情解碼）

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

頁序：所有網域條目（id 遞增）、然後所有
TLD 條目（id 遞增）、然後凍結於快照高度的
TLD 生命下限（`snapshot_tld_floors_all` —
不是活躍資料表）— 即儲存庫游標分頁的
順序。每頁攜帶
`(index, height, 檢查點的 state_root,
entries)`：孤零零一頁也知道它屬於哪個
快照。匯出時每頁 256 條
（`VERIFIED_SNAPSHOT_PAGE_ENTRIES`）；
匯入時 DoS 界限。TLD 下限
（`tld_floors_v1`）是可攜格式的一部分：
kind 0x03 攜帶它們、且匯入要求它們嚴格
遞增並指向快照中**存在**的 TLD — 否則
匯入者永遠不會殺掉匯出者會殺掉的東西
（沉默分叉、被拒）。

### 匯出（`export_verified_snapshot(store, checkpoint)`）

序列化**最後已最終化檢查點**的狀態 —
不是尖端。先發出網域、TLD、然後凍結於
快照高度的 TLD 生命下限（匯出／匯入
對稱：匯出歷史上遺漏下限 — 已修正、
回歸測試 `export_carries_tld_lifetime_floors`）。
前提：儲存庫持有恰在該檢查點高度的
M6b 持久化快照（`snapshot_v1_*`）
（`CHECKPOINT_KEEP`／`SNAPSHOT_INTERVAL`
窗口保證這樣的狀態規律地可用）；區塊的
錨從儲存的位元組**重新計算**、且必須等於
`checkpoint.data.block_hash`。否則 →
`StorageError::SnapshotUnavailable`（匯出是
明確的動作：等待下一個區間／檢查點）。

### 匯入（`import_verified_snapshot(store, checkpoint, manifest, pages)`）

驗證一切（見上）、然後在**一個** redb 交易
中寫入：活躍資料表 `domains`／`tlds` +
計數器 + `tip = (H, 檢查點雜湊)`、凍結
資料表 `snapshot_v1_*` 錨定於同一區塊、
以及保留標記
`meta["verified_snapshot_v1"]`（`height ‖
tip_hash ‖ manifest_hash`）。目標：僅限
空儲存庫（否則 `SnapshotUnavailable`）。

### 驗證成本（已載明）

經有序插入的 SMT 重建：N 條 × O(40) 雜湊
≈ O(N·40)。100,000 個網域約 2.1 M 個
blake3 雜湊（數十 ms）。一次性 — 引導／
修復匯入、絕非每區塊路徑：可接受。
有序插入（id 遞增）只建立每個分支點
一次；無論如何根只取決於內容（SMT
保證、`smt.rs` 中的差分模糊）。

## 狀態引導 — 不重播創世（P0.4）

新節點不從創世重播歷史即加入網路：

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

- **暴露給中繼站的儲存 API**：
  `append_bootstrap_anchor(H, hash, bytes)`
 （鋪設錨區塊：必要的標記、與標記的
   高度／雜湊一致性、冪等）然後依遞增
   高度的普通原子附加。網路擷取（依
   清單中的雜湊）是中繼站的事、不是
   儲存庫的。
- **`load_chain`** 認得標記：H 以下的歷史
  合理地缺席（非封存節點）。具體而言：
  M8 防重播窗口重掃以 H 為下限（H 以下
  的區塊不存在、也不缺失）；匯入**之後**
  沒有儲存區塊的尖端（錨尚未鋪設）→
  型別化診斷 `SnapshotUnavailable`（「錨
  必須來自網路」）、不是損毀。
- **封存 vs 全狀態節點**：封存節點
  （operator 旗標）保留一切 — 它向網路
  服務歷史區塊、並經既有路徑開機。全
  狀態節點（以此方式**組態**的新節點之
  P0.4 預設）在 H 開機：它驗證未來區塊
  （字尾重播）、但不能服務 H 以下的
  歷史或重播創世。**預設行為不變**：
  只有營運者明確觸發匯入時此路徑才
  活躍（`import_verified_snapshot` 是專屬
  動作、絕不自動）；每個既有節點相同地
  開機。
- **與完整節點的相等性**：加入字尾
  H+1..H+k 之後、狀態、尖端與
  `state_root_smt` 與同一尖端的完整節點
  逐位元相同（已測試）；防重播索引恰好
  涵蓋 H 之上（含）的 tx — 非封存節點
  無法知道 H 之下的窗口、狀態規則仍是
  防線（H 之下重複的 tx 在狀態上失敗、
  絕不被接受兩次）。

## 記憶體策略

- **處處游標**：`iterate_domains` 經 redb 的
  惰性 `range` 把頁限制在 `max` 條（預設
  100）；整個網域資料表絕不被物化。大
  清單（TLD、網域）必須經由此游標
 （`iterate_tlds`、相同契約、M7d）。
- **只有差量**：`append_block_with_state`
  只寫被區塊修改的網域與 TLD — 絕非
  完整狀態；
- **無同步重複**：RAM 狀態不每區塊重新
  持久化；磁碟狀態不每區塊重新載入；
- **點讀**：每次讀取開啟一個短讀交易、
  回傳值、關閉；redb `AccessGuard` 只活
  在有界複製期間；
- **歷史區塊絕不進 RAM**：`load_chain`
  只載入尖端；`load_chain_replay` 逐高度
  串流；
- **維護的計數器**：`domain_count` 是
  O(1)、不是掃描。

## DoS 界限

儲存庫約束敵對 peer 能讓它持久化或配置
的東西：

- **`MAX_DHT_CACHE_ENTRY`（64 KiB）**：更大
  的 DHT 記錄被 `put_dht_cache` 在任何寫
  交易**之前**拒絕 →
  `StorageError::TooLarge`（什麼都不寫）；
- **`MAX_DOMAIN_PAGE`（10,000）**：
  `iterate_domains` *內部*把請求的頁限制在
  `min(max, 10_000)` — 過大的 `max` 絕不
  在 RAM 物化超過 10,000 個狀態；
  `iterate_tlds` 有相同界限（`MAX_TLD_PAGE`、
  M7d）；
- **保留的 `meta` 金鑰**：`meta_set` 以
  `StorageError::ReservedKey` 拒絕 `tip`、
  `tip_height`、`format_version`、
  `domain_count`、`tld_count` 與
  `verified_snapshot_v1`（P0.3）— 覆寫
  儲存庫的內部記帳會靜靜地腐蝕它。
- **雙重界限的 DHT 快取（LRU）**：
  `DHT_CACHE_MAX_ENTRIES`（100,000 條）
  **及** `DHT_CACHE_MAX_BYTES`（2 GiB —
  敵對情況防護；每條已被
  `MAX_DHT_CACHE_ENTRY` 鉗制在 64 KiB）、
  先觸及者依金鑰序逐出。計數器（條目 +
  位元組）在寫入時維護 — 絕不為了計數
  而掃描；精確重建只發生在開啟時（唯一
  的完整掃描、在開機時）。存取器
  `dht_cache_entries`／`dht_cache_bytes`、
  可調界限經 `set_dht_cache_bounds`（測試：
  `dht_cache_respects_the_double_bound`）。

## 信任模型

兩條載入路徑、兩級保證：

- **`load_chain` — 本機磁碟信任**：尖端的
  雜湊從尖端區塊標頭**重新計算**並與
  儲存的 `meta["tip"]` 比較（分歧 →
  `StorageError::Corrupted`、絕不在偽造
  雜湊上還原鏈）、網域狀態被嚴格解碼。
  然而、歷史區塊**不**被重播：持久化的
  狀態被照單全收。這是信任自己磁碟
  （只由同一節點原子地寫入）之節點的
  快速啟動。自 M6b 起、此路徑在可用時
  先使用**狀態快照**（見下）：快照的錨
  被重新計算、H 之上的字尾如活躍區塊
  般重新驗證 — 重播的部分因此是零
  信任、凍結的基礎是磁碟信任。
- **`load_chain_replay` — 零信任**：所有
  區塊被逐一重讀並重新驗證（完整驗證、
  與活躍區塊相同）；所得尖端必須等於
  儲存的尖端。這是控制／修復工具。

**建議**：嚴格的節點（或儲存檔案可能被
竄改的節點 — 共享磁碟、可疑備份的還原）
必須經 `load_chain_replay` 啟動、而非
`load_chain`。

## 並發

redb 4.x 允許**每檔每行程單一
`Database` 物件**（同時雙重 `open` 以
`DatabaseAlreadyOpen` 失敗）。受支援的
模型：跨執行緒共享的克隆 `RedbStore`
（`Arc<Database>`）— 寫入者由 `&mut self` +
redb 內部鎖序列化、讀取者並發（快照讀
交易）。所有處理完全 drop 之後、檔案可
以重新開啟。

## 測試（`tests/redb_store.rs` + `#[cfg(test)]`
模組）

每個測試使用自己的 redb 暫存檔
（`tempfile`）。涵蓋：

- 區塊往返 創世→N、重啟、重讀、重播；
- M7d：TLD 登記冊經原子差量持久化、
  重啟時還原（重啟前聲明的 TLD 之下、
  `RegisterDomain` 保持可受理 — 窗口
  關閉）、重新開啟時計數器精確、`tlds`
  游標升序、失敗的附加不留 TLD、保留
  金鑰 `tld_count` 被拒；
- 原子性：失敗的附加什麼都不留（尖端、
  網域、索引完好）；重複差量只計一次；
- 網域游標：分頁 100/7、`DomainId` 升序、
  從存在或缺失的 id 之後嚴格開始、
  `max = 0`；
- 精確的 `domain_count` 計數器（插入／
  更新／重新開啟）；
- DHT 快取往返 + 覆寫；
- 並發：兩執行緒在克隆處理上讀取；
- 損毀的資料庫：截斷／垃圾檔案 →
  型別化錯誤、不 panic（`catch_unwind`）；
- M6b：快照在區間時寫入、嚴格低於
  尖端；從快照開機 = 完整的逐位元重播
  （state_root + SMT、完整邏輯狀態、
  尖端）；被竄改的快照（損毀的
  tip_hash、無法解碼的狀態、height >
  tip、錯誤長度的 meta）→ 靜靜退回完整
  載入、相同的最終鏈；重播的字尾 <
  區間；跨越區間時快照替換（無累積）；
  沒有快照資料表的 M6b 前儲存庫相同地
  載入；
- M8：開機時重建防重播索引 — 還原後、
  窗口內重新收錄的 tx 被拒（`TxReplay`）；
  比窗口舊的 tx 被接受（修剪不變、狀態
  規則決定）；對比窗口短與長的儲存庫、
  精確計數 == 活躍索引（==
  `load_chain_replay`）；快照開機：窗口中
  H 之下的部分由重掃重建、H 之上的由
  字尾重播；
- P0.2：名稱索引 — 120 個網域的
  `resolve_name == DomainId::from_name`
  （活躍、重啟後、抹除 + 重建後）、
  已聲明 TLD 的 `resolve_tld`；過期網域的
  GC 移除條目（無幽靈）、重建前後；
  網域/TLD 命名空間的嚴格分離；失敗的
  附加不留索引條目；索引金鑰是載明的
  BLAKE3 衍生；
- P0.3/P0.4（`tests/verified_snapshot.rs`）：
  匯出 → 匯入 → 開機 == 完整重播（狀態、
  尖端、逐位元 `state_root_smt`；匯入的
  狀態是已最終化區塊的、不是尖端的）；
  被改一位元組的頁 → 任何寫入前拒絕
 （儲存庫嚴格不變）；缺頁 → 拒絕；錯誤
  的 `state_root`（清單 ↔ 檢查點、然後
  重新計算的根 ≠ 簽署的根）→ 拒絕；錯誤
  的計數器／高度／不一致的尖端／重複
  id → 拒絕；匯入要求空儲存庫；匯出
  要求檢查點高度的持久化快照；錨契約
 （`append_bootstrap_anchor`：必要的
  標記、一致性、冪等）；嚴格的序列化
  格式（多餘位元組、標籤、DoS 界限）；
  H 的快照然後加入區塊 H+1..H+k → 鏈
  與完整節點相同（狀態/尖端/根）、無
  H 以下歷史的開機、防重播索引 = 恰好
  H 之上（含）的 tx。

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

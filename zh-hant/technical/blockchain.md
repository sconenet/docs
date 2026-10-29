# 區塊鏈

區塊鏈層的參考文件：身分、雜湊、創世、
正典狀態、交易排序與區塊驗證。
參考實作是 crate
[`scone-blockchain`](../../../crates/scone-blockchain)，建立在
`scone-protocol`（正典編碼）、`scone-core`（業務規則）與
`scone-crypto`（BLAKE3）之上。

目前範圍：帶真實共識的區塊鏈邏輯（簽署生產者、
檢查點、最終性）、KV 後端之上的**記憶體內狀態**
（具體儲存位於 `scone-storage`）— 見
[延後至共識](#延後至共識)。

## 身分與雜湊領域分離

四個身分、四個角色、四個互異的 BLAKE3 前綴 — 一個前綴
絕不在兩個用途間重複使用：

| 型別 | 角色 | 前綴 | 公式 |
|---|---|---|---|
| `DomainId` | 網域名稱身分 | `SCONE-DOMAIN-V1` | 見 `/docs/en/general/naming.md` |
| `RecordHash` | DNS 內容身分 | `SCONE-RECORD-V1` | 見 `/docs/en/technical/protocol.md` |
| `TxId` | 交易身分 | `SCONE-TX-V1` | 見下 |
| `BlockHash` | 區塊身分 | `SCONE-BLOCK-V1` | 見下 |

Merkle 樹使用自己的前綴 `SCONE-MERKLE-V1`。這些前綴
是 `scone-core` 前綴（`SCONE-PUBKEY-V1`、
`SCONE-OWNER-V1`）之外的補充。

完整相依鏈：

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

- 決定性：同一筆交易（同一正典編碼）⇒ 同一
  `TxId`，在所有節點上，永遠；
- **依內容**的身分：任何被修改的欄位都改變 `TxId`
  （網域、擁有者、時間戳、證明、序號、record_hash）；
- 沒有本機時間戳或非正典的值進入計算；
- 不相依於儲存或網路；
- 此身分不保證套用的唯一性：重播
  一筆相同的交易會在狀態規則上失敗（重複 `RegisterDomain`、
  序號重播），而非在 `TxId` 上。

API：`scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`。

## Merkle 樹（tx_root）

對一個區塊交易的**有序清單**承諾：

```text
0 transactions  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
internal node   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
odd level       the last hash is duplicated to complete the pair
```

規則：

- **絕不排序**：區塊中交易的順序
  具有意義；兩個包含相同交易但順序
  不同的區塊有不同的根；
- `0x00`／`0x01` 標籤防止葉與
  內部節點之間的任何混淆（第二原像）；
- `SCONE-MERKLE-V1` 前綴將樹與其他雜湊
  領域隔離（`TxId` 不能被解讀為節點）；
- 自制實作（約 30 行），決定性且經測試；不
  使用外部 Merkle 程式庫。

API：`merkle_root(&[TxId]) -> MerkleRoot` 與
`tx_root(&[Transaction]) -> Result<MerkleRoot>`。

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

雜湊**僅**涵蓋正典標頭：交易已
由 `tx_root` 承諾，因此不再雜湊第二次：

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

修改一筆交易、它們的順序、高度、時間戳、
`prev_hash` 或共識酬載都會改變 `BlockHash`。

API：`block_hash(&BlockHeader) -> Result<BlockHash>`。

## 網路與創世（M8b）

Scone 定義三個互斥的網路，由正典的
`network_id`（1..=16 個 ASCII 位元組 `[a-z0-9-]`）識別：

| 網路 | `network_id` | 私有 TLD 頂點 | 開放 TLD 頂點 | 網域頂點 | TLD 下限 | 網域下限 |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bits | 7 bits | 6 bits | 2 bits | 1 bit |
| mainnet | `scone-mainnet` | 28 bits | 27 bits | 26 bits | 25 bits | 22 bits |
| devnet | `scone-devnet` | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit |

參數位於 `scone-core::network::NetworkParams`
（具名實例 `TESTNET`／`MAINNET`／`DEVNET`，透過
`NetworkParams::by_name` 查詢）。改變難度 = 該網路
的共識變更。

**結構不變量**：devnet 是完整的第三個
`NetworkParams` — 絕非 testnet 旗標的子集。Testnet
與加入 devnet 之前**嚴格相同**
（相同的 Argon2id 常數、相同的曲線、相同的 epoch 窗口）；
測試套件釘死此不變量。

**長度可變難度**（`scone-core::pow_length`）：建立
的難度取決於**被建立標籤**的長度
（`RegisterTld` 的 TLD — 私有或新開放 —，
`RegisterDomain` 的最左標籤）。公式給出：

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), clamped at the floor
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), clamped at the floor
```

兩條互異的斜率：TLD 為 `log₂(1.22) ≈ 0.287` 位元/字元
（每多一個字元將成本除以 ~1.22），網域為
`log₂(1.40) ≈ 0.480` 位元/字元 — 網域衰減
較快：其價值在於承載它的 TLD。天生開放的 TLD
位於私有頂點 − 1；網域位於私有頂點 − 2
（主網：私有 TLD 25、開放 TLD 24、網域 23）。主網
基準（記憶體困難 Argon2id，單執行緒約 11.4 H/s）：
私有 TLD 1 字元 = 28 bits ≈ 30 天，9+ 字元 = 25-bit
下限 ≈ 3.8 天；開放 TLD 1 字元 = 27 bits ≈ 15 天；
網域 1 字元 = 26 bits ≈ 3.8 天，9+ 字元 = 22-bit
下限 ≈ 11.5 小時。**8 核參考機**
（2026-10 重新校準 — CLI 多核
挖礦），約 11.4 H/s/核。**開放**既有
私有 TLD（`SetTldOpen`）維持平難度
（`open_tld_pow_difficulty`：testnet 6 bits／mainnet
22 bits）：交易只攜帶 `tld_id`，名稱在建立時
已為其長度付費。**另外兩種平式證明**補齊了
這個家族（共識層級的反濫發、與註冊相同的
epoch 鹽和窗口）：`transfer_pow_difficulty`
用於讓與（`TransferTld`／`TransferDomain`、挑戰
= 命名空間**當前**的計數器綁定 — 見
`transactions.md`、變更計數器），
`assign_pow_difficulty` 用於私有 TLD 之下的建立
（`AssignDomain`、挑戰 `SCONE-DOMAIN-ASSIGN-V1 ‖
name` — 刻意為平：命名空間的排他性已在 TLD
建立時付費、無按長度的重複課稅）。指示性
數值：16-bit mainnet（bench-pow 2026-10-23
校準：在實測 10.6 H/s 的單核上、每次讓與／
指派約 1.7 小時、參考 11.4 H/s；4 核約
25 分鐘 — 對工業化而言昂貴、對誠實的
擁有者則是平常的工作）、4-bit testnet、
1-bit devnet。純函數（決定性）。

創世是**每網路**的協定常數，僅由
常數即可重現（無隨機性、無本機時鐘、無網路）：

| 欄位 | 值 |
|---|---|
| `version` | `PROTOCOL_VERSION`（`1`） |
| `height` | `0` |
| `prev_hash` | 32 個零位元組（無前一區塊） |
| `tx_root` | 空清單的 Merkle 根 |
| `timestamp` | `GENESIS_TIMESTAMP = 0`（純結構性） |
| `consensus` | `network_id` 的正典位元組 |
| 交易 | 空 |

**配置創世**變體（主網 kickstarter，M-next）：`consensus`
欄位攜帶 `network_id ‖ ALLOC-V1 ‖ count.v ‖ entries`
（預付的網域/TLD、依名稱正典排序、清單 ≤ 60 KiB、
≤ 2048 項配置）— 此加寬格式正是
`MAX_CONSENSUS_LEN` = 64 KiB 的理由（見
`scone-blockchain::genesis_alloc`）。

**創世配置產物的散布**：任何創世產物
（`genesis.bin`、清單）都絕不存放在原始碼
儲存庫或程式碼發行中、無論網路或專案階段
— 兩份創世檔就是兩個網路。mainnet 配置創世
僅在上線時誕生（切換為預設啟用的
`mainnet` feature）：由 `scone-genesis` 從
發布於儲存庫之外的已簽署清單產生、其雜湊
固定在上線公告中、且只有創始節點消費它
（`--genesis`、鏈的誕生）；所有其他節點以
同步加入網路。該工具拒絕任何寫入 Rust
workspace 目錄內部的行為（存在 `Cargo.toml`）：
創世絕不能落入原始碼樹。

共識酬載中的 `network_id` 使創世雜湊 —
乃至整條區塊鏈 — **跨網路互斥**：testnet
區塊永遠無法接到 mainnet 鏈
（`UnknownParent`），不重新簽署
一切就不可能重播。

API：`genesis_of(&NetworkParams) -> Block`、
`genesis_hash_of(NetworkId) -> BlockHash`；別名 `genesis()`／
`genesis_hash()` 指向 testnet（開發預設）。

### Devnet — 可重置的本機網路

devnet（`scone-devnet`）是第三個網路，專為快速
本機迭代：

- **自己的創世**：共識酬載中的 `scone-devnet`
  網路 id 產生與 testnet 和 mainnet 兩者皆
  不同的創世雜湊 — 沒有任何創世、區塊、簽章或
  PoW 證明跨越邊界（重播不可能）。
- **近零 PoW**：曲線釘在 0..=2 bit 範圍
  （`DEVNET_APEX = 2`、`DEVNET_FLOOR = 1` — 長度衰減
  仍適用）：TLD 或網域聲明在毫秒內挖成，
  走的與其他網路完全相同的程式碼路徑。
- **快速 PoS 共識**：與 testnet 相同的參數（委員會
  4 人、10 s epoch），在本機網格上快速最終化。
- **重置**：鏈住在自己的資料目錄
  （預設 `~/.scone/devnet/`）。**抹除該目錄並
  重新啟動中繼站即重置整條鏈**（新鮮創世、
  全新儲存庫）— 無需額外工具。絕不可將 devnet
  資料目錄指向另一網路的資料目錄：持有另一
  條鏈的非空儲存庫會被拒絕（`Corrupted`）。

## 正典狀態

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64, mutations: u64 }
TldState     = enum { Private { owner: OwnerId, mutations: u64 } | Open }
               -- enum (owner-option): the open/private invariant is
               -- structural; the death deadline lives outside the entry
               -- (tld_floors table + journaled death schedule)
ChainState   = NetworkParams                  (bound network, M8b)
               + DomainId -> DomainState      (direct access, in memory)
               + TldId -> TldState            (TLD registry, M7b)
               + DomainId -> (expired_at, OwnerId)   (grace windows, M8b)
```

`mutations` 是命名空間**單調**的變更計數器
— 其受防護操作（`SetRecordDelegates`、
`SetTldOperator`、`SetDomainOperator`、
`TransferTld`、`TransferDomain` 先驗證後
遞增；續期保留它、轉移提高它）的 CAS 反重播
世代。它被提交在命名空間的 SMT 葉中 —
網域葉 `SCONE-LEAF-DOM-V4`（計數器位於
生命週期欄位與記錄承諾之間）、TLD 葉
`SCONE-LEAF-TLD-V4`（單射的變體位元組；
`Private` 攜帶計數器）— 並在 name-state
回應中暴露。客戶端先讀取再綁定
（fetch-then-bind）：對過時世代構建的交易是
一次型別化拒絕（`StaleStateBinding`）、不
變更任何東西。完整衍生與理由：
`transactions.md`（變更計數器）；安全性
分析：`security-model.md`。

* `registered_at`／`valid_until`（M8b）：註冊到期，
  以**父區塊的時間戳**評估（決定性，
  承諾於標頭 — 絕非本機時鐘）；
* `open`：私有 TLD 可以**天生開放**（`RegisterTld`
  攜帶 `open = 0x01` — 無主、自助）或在聲明時
  **封閉**（封閉 = 僅限指派），之後透過 `SetTldOpen`
  （帶 PoW 的自助註冊）變為開放 — **不可逆**；

- 依 `DomainId`（32 位元組）直接存取：金鑰絕不是
  `String`、絕不全文掃描 — 這是邁向數千億
  網域的必要條件；
- 只要尚未套用任何 `UpdateDomain`，`record_hash == None`；
- TLD 與網域的識別碼空間**在建構上互斥**
  （互異衍生前綴，見 `/docs/en/general/naming.md`）：
  註冊 TLD `uip` 絕不能霸佔網域的身分，反向
  亦然。

### 套用規則

`state.apply(transaction)` 是決定性且原子的（出錯時
狀態不變）：

**所有交易（M8b，2026-09 線上格式）**：交易不再
攜帶 `network` 欄位 — 分離依賴**簽章**：簽署
酬載注入鏈的 `network_id`
（`validate_transaction(tx, network_of_the_chain)`），
因此為另一網路簽署的交易會簽章驗證失敗
（`InvalidSignature`，在套用之前 — testnet 簽署的
交易永不被 mainnet 節點套用、入池，反向亦然）。

**REGISTER_TLD**：

- TLD 必須**空閒**（不在 TLD 登記冊中）且**非
  ICANN 保留**（`IcannTldReserved`：ICANN 根 TLD
  在 Scone 上不屬於任何人）；
- **強制 PoW**：對挑戰
  `SCONE-TLD-V1 || tld`（由交易攜帶的名稱衍生）
  執行 `pow::verify`，難度為該標籤的**依長度**
  難度（`pow_length`）：私有誕生 → `tld_difficulty`
  （網路頂點），開放誕生（`open = 0x01`）→
  `open_tld_difficulty`（頂點 − 2）— 摘要
  從零重新計算
  （鹽 `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`、
  Argon2id 核心），自報的
  難度必須等於預期難度，且證明**宣告的 epoch**
  必須等於套用區塊的 `height/POW_WINDOW`
  （對預挖的 epoch 鹽 — 型別化拒絕
  `ProofEpochMismatch`，見 `transactions.md`）；
- 套用時：`open = 0x01` → `{ Open }`（無主 — 自助、
  PoS 席位釋放、死亡下限設於 `now + grace`）；
  `open = 0x00` → `{ Private { owner } }` — **預設封閉**
  （僅限指派）；開放須透過 `SetTldOpen` 明示。

**REGISTER（RegisterDomain）**：

- **攜帶名稱的 TLD 必須已註冊且開放**：鏈
  自行衍生 `TldId(name.tld())`（絕非外部供值）；
  不存在 → `UnknownTld`（D1、M7c），存在但封閉 →
  `TldClosed`（M8b：僅限指派的路徑為獨佔）；
- 網域必須**空閒**：不在狀態中且不處於任何
  過期註冊的寬限窗口（見下文 GC）；
- **強制 PoW**：挑戰 `SCONE-DOMAIN-V1 || name`，
  難度為**最左標籤的依長度難度**
  （`pow_length::domain_difficulty` — TLD 既已開放，
  這是自助入場價）；
- 套用時：`{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 year }` —
  **註冊為期 1 年**（365 天）。

**區塊內順序**：有意義。同一區塊內的
`[RegisterTld, SetTldOpen, RegisterDomain]` 有效；
任何在前置條件成立前套用規則的排列都使整個
區塊被拒（undo-log）。

**UPDATE（UpdateDomain）**：

- 網域必須存在（未過期）；
- `tx.owner` 必須是當前擁有者；
- `tx.sequence` 必須**恰好**是 `current_sequence + 1`
  （無間隙、無重播）；
- 套用時：`sequence` 前進、`record_hash` 被替換。

**TRANSFER_TLD（0x47）**：

- TLD 必須存在且為**私有**（否則 `UnknownTld`／
  `TldIsOpen` — 開放 TLD 不屬於任何人）；
- 簽署者必須是當前擁有者（否則 `NotTldOwner`）；
- 套用時：`owner = new_owner`（受讓者的身分
  不透明；與可花費金鑰的綁定由後續
  交易完成）。沒有序號：第一筆被套用的轉移獲勝
  （鏈序）。

**REVOKE_TLD（0x6B）**：

- TLD 必須存在且為**私有**（開放 TLD 不能被
  撤銷 — `TldIsOpen`：它在 GC 自然死亡），簽署者 =
  當前擁有者；
- 套用時：TLD 自登記冊移除，可被新的
  `RegisterTld`（含 PoW）重新聲明。

**SET_TLD_OPEN（0xB8）**：

- TLD 必須存在且為**私有**（已開放 → `TldIsOpen`：
  前擁有者已被抹除，不再存在可為其
  簽章的合法金鑰）；`open = 0x00`（關閉）→ 在
  純驗證階段即型別化拒絕
  `TldCloseUnsupported`：**開放不可逆**
  （owner-option）；
- 簽署者 = 當前擁有者（`NotTldOwner`）；
- **開放 = 強制 PoW**：對挑戰
  `SCONE-TLD-OPEN-V1 || tld_id` 以網路的平
  `open_tld_pow_difficulty`（testnet 6 bits／
  mainnet 22 bits）執行 `pow::verify`，
  含 epoch 鹽（證明存放於交易簽署的
  `open_proof` 欄位）；
- 套用時：`TldState::Open` — **擁有者被抹除**
  （不可逆、PoS 席位釋放），死亡下限設於
  `now + grace`。

**ASSIGN_DOMAIN（0xD4）**：

- 攜帶名稱的 TLD 必須存在且為**私有**
  （否則 `TldIsOpen`：開放 TLD 僅限自助），簽署者 =
  TLD 的擁有者（`NotTldOwner`）；
- 網域必須空閒（與 REGISTER 相同的寬限規則）；
- 套用時：`{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 year }`。
  **無 PoW**：由命名空間的擁有者作保。

**TRANSFER_DOMAIN（0xE2，M8c）**：

- 網域必須存在（否則 `UnknownDomain`），簽署者 =
  當前擁有者（`NotOwner`）；
- 套用時：`owner = new_owner`（PoS 席位在擁有者
  池中跟隨網域，與 `TransferTld` 對稱）。沒有
  序號：第一筆被套用的轉移獲勝（鏈序）。

**RENEW_DOMAIN（0x3C）**：

- 網域必須存在；**TLD 種類最先被評估**
  （M8c 強化）：
  - **私有 TLD**：**PoS** 授權 — 簽署者就是 TLD
    擁有者，或 TLD 擁有者的席位委派給簽署者的
    公鑰
    （`effective_operator(tld_owner) == renew.public_key` —
    委派永遠從 TLD 擁有者讀取，絕不
    反向）。`proof` 欄位必須為空（非空
    證明 = 拒絕）。
  - **開放 TLD**：**PoW** 授權 — 簽署者是
    當前網域擁有者（否則 `NotOwner`），證明
    挖出續期挑戰：與建立相同的難度
    （依標籤長度），**綁定於當前到期日**
    （`challenge = DOMAIN_ID_VERSION ‖ canonical name ‖ current
    valid_until`）— 續期的證明有別於
    註冊與先前每次續期
    （防重用）；
- `valid_until` 必須**嚴格延長**當前期限
  （否則 `RenewalNotExtending`）；
- `valid_until ≤ now + 3 years`（`RenewalExceedsTerm` —
  反囤積界限：**1 個續期任期 = 1 年，最多
  預先 3 年**）；
- 套用時：`valid_until = tx.valid_until`；續期同時
  延後其下開放 TLD 的死亡下限（交易為此
  攜帶正典名稱）。

**決定性 GC（M8b）**：套用區塊交易之前，
鏈以**父區塊的時間戳**評估到期：任何
`valid_until ≤ now` 的網域都自活躍狀態移除並
停入 **30 天寬限窗口**
（`valid_until + 30 天`）。寬限期間只有前
擁有者能重新聲明該名稱；其後對所有人
自由。GC 有日誌：被拒的區塊逐位元
還原 GC 前的狀態。

**TLD 生命週期（固定任期）**：TLD 帶有自己的
時鐘，獨立於網域。

- **私有 TLD**：聲明開啟 **3 年任期**
  （`TLD_TERM_SECS`）。續期（`RenewTld`、0x8F）支付
  **與建立等量的 PoW** 並精確重置 3 年（套用時
  重新衍生固定任期）。到期時開啟 **60 天寬限**
  （`TLD_GRACE_SECS`）：續期仍然可能，但**其他
  每項管理操作都凍結** — 尤其是 `AssignDomain`
  （型別化拒絕 `TldExpiredFrozen`）：過期的
  擁有者失去管理權直到續期。其下已指派的
  網域過各自的 `valid_until` — 凍結的是 TLD，
  不是它的子網域。寬限過後未續期：TLD 在
  GC 死亡（PoS 席位釋放、可重新聲明）。
  `TransferTld` 攜帶剩餘任期。
- **開放 TLD**：誕生（或開放）時帶 **1 年下限**
  （`OPEN_TLD_FLOOR_SECS` — 防瞬間建立又摧毀）。
  其下每個活著的網域都把死亡時鐘往後推（跟隨
  最新網域的到期）。沒有網域且下限
  已過：GC 死亡、可重新聲明。
- **不朽開放 TLD**（僅創世：`.scone`、`.tor` —
  透過 CNAME 在洋蔥位址之上提供可驗證命名 —、
  `.p2p`）：`OpenImmortalTld` 配置設定
  `until = u64::MAX` 哨兵 — GC 永遠無法跨越它，
  單調重新武裝永遠無法降低它：TLD 永不到期，
  即使所有網域都被清空。沒有任何執行時
  交易能建立不朽（`RegisterTld` 保持 1 年
  下限）；其下網域過正常的週期。

TLD 死亡與網域共享決定性死亡佇列
（`tld_deaths`），有日誌且可逐位元還原。

**高度 1 例外**：創世帶有結構上為零的時間戳
（`GENESIS_TIMESTAMP` = 0，協定慣例 — 不是時鐘）。
第一個區塊的鏈時間因此是它自己的時間戳
（承諾於其標頭、決定性），而非父區塊的：把
區塊 1 的到期錨在 0 會把天生開放 TLD 的死亡
下限（以及區塊 1 誕生網域的 `valid_until`）釘在
Unix 紀元 — 區塊 2（真實時間戳）的套用前
GC 會在套用前移除該資源（「dropped conflicting
tx from block production: unknown TLD」，testnet
回歸 60）。

**重播**：跨網路重播在三個層次上不可能 —
互斥的創世（`UnknownParent`）、注入每個簽章
酬載的 `network_id`（在另一網路上簽章無效 —
自 2026-09 線上格式起網路僅出現於此）、PoW
在摘要中依網路分離。同網路內相同交易的
重播在狀態規則上失敗（重複聲明、序號），
絕不在 `TxId` 上。

同一網路的兩個節點從相同狀態出發、
以相同順序套用相同區塊，會產出完全
相同的最終狀態。

## 交易排序

順序是根本屬性，在每一層都被承諾：

```text
transaction order in the block
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
canonical chain
```

兩筆交易不可互換：同一區塊內
`[REGISTER, UPDATE]` 可套用、`[UPDATE, REGISTER]` 被拒
（`UPDATE` 時網域不存在）。全域平手決勝（排序
記憶體池、產出區塊者）屬於共識。

## 區塊驗證（push_block）

一切皆**重新計算**，無一憑信念 — peer 提供的
`tx_root`、雜湊、簽章或欄位絕不照單全收：

1. `prev_hash` == 正典尖端的雜湊（父未知 →
   `UnknownParent`；已知但非尖端 → `ParentNotTip`：
   最小分叉偵測）；
2. `height` == 尖端高度 + 1；
3. `version` **恰好** == `PROTOCOL_VERSION`（任何其他
   版本 — 較低或較高 — 都被拒絕，不被重新解釋）；
4. 交易數 ≤ `MAX_TXS_PER_BLOCK`（4096）；
5. 對區塊順序中的交易重新計算 `tx_root`；
6. 共識掛鉤（`validate_header`、`validate_tx`）；
6ter. **開窗 TXID 防重播（.bak 移植）**：`TxId`
   出現在最近 `REPLAY_WINDOW_BLOCKS = 256` 個
   區塊之收錄索引中的交易被拒
   （`TxReplay`）。該索引（TxId → 收錄高度）是
   正典鏈的純函數，以相同步調修剪；索引
   在開機時由儲存層重建（重掃最後
   `min(256, height)` 個持久化區塊 — M8，見
   `/docs/en/technical/storage.md`）：重啟的節點
   與活著的節點拒絕相同的
   重播。窗口之外，重新收錄仍受
   狀態規則約束（重複聲明、序號重播）。
6bis. **簽署生產者（M5、.bak 移植）**：`consensus`
   酬載必須解碼為生產者酬載
   （`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`，有界）；
   簽章涵蓋*生產者簽章雜湊* = 標頭的 `BlockHash`
   **且 `consensus` 欄位清空**（酬載住在該
   欄位；簽完整雜湊會循環）；生產者必須
   屬於區塊 `timestamp` 時獲授權的集合（由
   檢查點選出的委員會，加上解鎖的復原抽籤；
   引導：活躍網域的擁有者們 — 空池 = 開放
   生產，否則第一次註冊將不可能）。創世
   （高度 0、空酬載）是結構性的、永不受此
   約束。型別化錯誤 `InvalidProducer`；
7. 對每筆交易依序：**密碼學**驗證
   （`validate_transaction`：擁有者／金鑰綁定重新計算 +
   對從零重新計算的簽署酬載執行 `verify_strict`，見
   `/docs/en/technical/transactions.md`），然後在
   **undo 日誌**下套用到正典狀態（undo-log：每筆
   已套用交易一條目，成本 O(區塊交易數)，
   而非 O(網域數)）；
8. 原子提交：全部通過 ⇒ 區塊加入、日誌丟棄；
   一有錯誤 ⇒ 日誌倒轉（逆序），鏈
   嚴格不變，逐位元一致。

沒有驗證函數 panic；任何畸形或敵對輸入
都產生型別化的 `BlockchainError`
（`OwnerKeyMismatch`、`InvalidSignature`、…）。

## 區塊組裝（BlockBuilder）

`BlockBuilder`（`scone-blockchain::builder`）從一列
已驗證的交易組裝區塊：

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` 對佇列中有序的交易清單**重新
  計算**（絕非外部供值）；
- `prev_hash`／`height` 鏈結到父（父雜湊由
  呼叫者透過 `block_hash` 計算，絕不取自 peer）；
- `version` = `PROTOCOL_VERSION`；
- `MAX_TXS_PER_BLOCK` 界限在插入時強制
  （`TooManyTransactions`）；
- `consensus` 酬載（簽署的生產者酬載）與
  `timestamp` 由呼叫者提供：具體共識透過既有的
  [`Consensus`](#共識抽象) trait 插入，與
  驗證對稱；
- M5：`.with_producer(&SigningKey)` 以生產者身分簽署
  區塊 — `consensus` 酬載成為（對上述生產者
  簽章雜湊的）簽署生產者酬載，並取代任何
  先前的 `.with_consensus`。

組裝好的區塊無需重新編碼即通過 `push_block`
（已測試）。

## 分叉與分叉選擇（M3c、.bak 移植）

`push_block` 仍是線性入口（父 == 尖端，否則
`UnknownParent`／`ParentNotTip`）。`try_attach`（M3c）評估
競爭區塊：

- 父未知 → `UnknownParent`（區塊等待）；
- 血統有效但落後 → 分支評估：最長鏈
  獲勝；完全平手 → 決定性平手決勝（字典序
  中最小的尖端雜湊 — 持有相同區塊的兩個
  節點總是選出相同贏家，已測試）；
- **最終性下限**：某祖先與已最終化檢查點
  （收錄於某已接受檢查點的 `prev_checkpoint_hash`）
  矛盾的分支被拒（型別化錯誤
  `Consensus("branch contradicts a finalized checkpoint")`）；
- 採納 = 從創世重播獲勝分支的區塊
  （逐位元決定性已測：兩個以不同順序收到
  相同區塊的節點最終狀態一致）；
- 沒有收到的區塊在完整驗證前成為正典。

## 有界 RAM 視窗（M7a、.bak 移植）

鏈在 RAM 中僅保留**最後 512 個區塊**
（`RAM_WINDOW_BLOCKS`，.bak 的 `HISTORY_KEEP = 512`
移植），位於**最終性下限**（最後已最終化的
檢查點）之上：

- 區塊只有在視窗外（`height < tip − 511`）
  **且**嚴格低於最終性下限時才離開 RAM —
  下限之上的 reorg 永遠可能，因此那些區塊
  絕不被逐出；
- 在第一個已最終化檢查點之前（引導），**沒有
  任何東西被逐出**：有界記憶體隨最終性
  開始；
- 逐出是真實的：區塊、雜湊索引（`known_hashes`，
  本身有界、依高度索引）與交易離開記憶體 —
  儲存庫（`NodeStore`、`scone-storage`）持有
  持久副本；
- 視窗之下的 `block(h)` → `None`；
  `block_result(h)` → 型別化錯誤 `BlockPruned { height }`
  — reorg 路徑（`push_block`、`try_attach`、
  `adopt_branch`、前綴重播）傳播它：中繼站必須
  從儲存重載該段或退回完整同步（已載明的
  限制：完整中繼移植前無分段採納）；
- 建在被逐出父之上的區塊分類為 `BlockPruned`
  （而非 `UnknownParent`）：父高度 < `base_height`
  ⇒ 儲存庫可提供它。

## 共識抽象

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus`（預設）：接受一切 — 在
  真實規則定義之前使用；
- 掛鉤由 `push_block` 呼叫：具體共識（PoW、
  難度、費用、時間戳／防重播、分叉選擇）將
  插入而無需改寫鏈；
- 單一抽象，無 trait 階層。

## 可擴展狀態（KV 後端、SMT 樞軸、到期佇列、有界擁有者池）

目標：2500 億以上的潛在網域 — 絕不把一切載入
RAM。四個部件：

### KV 後端（`state_backend.rs`）

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = `DomainId`／`TldId` 的 32 個原始位元組 — 單一
  有序金鑰空間（互異衍生前綴），值 =
  `SCONE-ENTRY-DOM-V1`（88 固定位元組）／
  `SCONE-ENTRY-TLD-V1`（可變：私有 33 位元組／
  開放 1 位元組、owner-option 枚舉 — 旗標 + 條件
  擁有者，不再有 `until` 欄位）編碼；
- `MemoryBackend`（BTreeMap）= 參考實作（開發、
  測試）；`scone-blockchain` 中沒有具體相依
  （crate 內禁止 redb）— 協定／共識僅相依於
  TRAIT；
- `CachedBackend<B>`（`state_cache.rs`）= 內層後端
  之上的有界 LRU 快取：write-through（內層
  永遠最新）、miss 絕不快取、`range` 對內層
  分頁 — 與內層後端單獨給出相同答案
  （決定性契約，已測試）；
- `RedbBackend`（`scone-storage::redb_backend`）=
  持久化後端：`state_kv` 資料表（32 位元組原始
  金鑰、字典序 = 所需升序）、計數器維護於
  `state_kv_meta` 中繼資料表（絕不用 COUNT）；
  `CachedBackend<RedbBackend>` = 有界 RAM 佔用的
  狀態 — 「輕量層」；
- `smt_persist`：消費 SMT 的髒集合
  （`take_dirty_sets`）並在保留前綴金鑰
  （0x01 節點、0x02 葉）下寫入後端 — 分頁
  SMT 的基礎；
- trait 不再要求 `Default`（磁碟資料庫沒有
  合理的預設值）；
- `range` 以金鑰升序迭代（決定性契約）—
  僅封存路徑，絕非共識規則；
- `ChainState` 僅透過後端以金鑰存取網域/TLD
  （點讀、有日誌的寫入）。

### 正典 SMT 根（樞軸）

- `state_root_smt()`（O(1)、快取根、
  `SCONE-STATE-SMT-V1`）是**正典**的 state_root：
  它是 `checkpoint_data` 所承諾、`accept_checkpoint`
  在尖端所驗證的。計算絕不
  遍歷網域集合；
- 舊的直接折疊 `SCONE-STATE-V1` 以
  `state_root_archive()` 存續：O(N)、封存／驗證
  路徑（從儲存完整重新折疊以交叉驗證歷史
  根）— 不是共識產物。兩者承諾相同葉：
  相等的邏輯狀態 ⇒ 相等的
  根（對兩者皆然）；
- 帶 SMT 根的已接受檢查點在還原
  （reload）後重新驗證：根是還原狀態
  （後端 + SMT + 索引）的純函數。

### 到期佇列（GC O(已到期)，而非 O(N)）

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` —
  到期時刻 → 屆時到期的識別碼（識別碼在
  每槽內排序：狀態相等與傾倒順序與到達
  順序無關）；
- `gc_expired_journaled(now)` 傾倒 `range(..=now)` 而非
  掃描整個狀態：成本 O(該時刻的已到期數)；
- RAM 界限：每個活著的網域一條目（僅待決
  到期 — 續期重新編排金鑰槽、GC 傾倒它），
  絕非掃描結構中每網域一節點；
- 由 undo 日誌維護（逐位元回滾：還原的
  槽被放回）。

### 有界擁有者池（金鑰，而非網域）

- `owner_refcounts: HashMap<OwnerId, u64>` — 每個
  互異活躍擁有者（網域 + TLD）一計數器，
  由日誌在每次 register/transfer/revoke/assign/GC
  維護（acquire/release）；
- `eligible_validators(now)` 讀取此索引 — O(擁有者
  數)，絕不迭代網域。時間上的活躍性已
  保證：決定性 GC（父時間戳）在呼叫前
  已移除每個過期網域及其引用計數。排序
  輸出（決定性）；
- RAM 界限：O(互異擁有者數)，與網域
  數無關；
- `owner_pool_root`（封存折疊）折疊這些排序
  的擁有者 — 也不掃描網域。

穩態 RAM 預算（磁碟／網路後端）：
O(互異擁有者 + 待決到期 + 進行中寬限窗口 +
undo 日誌) — 全部與網域數無關地有界，
全部由日誌逐位元還原。

## 席位與最終性委員會（精確 PoS 機制）

本節描述如實作（`finality.rs`）的**精確**資格與
委員會選舉機制 — 營運互補（錨點迴圈、
線上格式）位於 `/docs/en/technical/relay.md`，安全
分析位於 `/docs/en/technical/security-model.md`。

### 席位 = 持有（無註冊）

PoS **席位**不是註冊：它是對活躍網域或
TLD 的持有。`eligible_validators(now)` 計算池：

- 活躍擁有者的金鑰，經由有界的
  `owner_refcounts` 索引（每個活躍網域/TLD 一個
  互異擁有者 — 一把金鑰下的一百個
  網域 = 一個席位：經濟反女巫攻擊以
  身分計、線性）；
- TLD 也算（其聲明成本是 PoW — 活躍 TLD 的
  擁有者是權益者），活躍網域亦然
  （`valid_until > now` — 由決定性 GC 保證、
  呼叫前已傾倒）；
- **扣除被禁金鑰**（M9：被罰沒的 equivocoder
  終身出局）；
- **開放** TLD（`owner: None`）不持有席位：其
  前擁有者在開放時釋放（`SetTldOpen` 套用中的
  `owner_release`），且它在 GC 死亡前不再
  投票。

排序輸出（決定性順序，與索引迭代
順序無關）。

結構性後果：**席位的死亡 = 網域/TLD 的
死亡**。到期 + GC、`RevokeTld`（席位釋放）、
`TransferTld`／`TransferDomain`（席位跟隨網域到
新擁有者）、`SetTldOpen` 開放（席位釋放）—
每次所有權轉移都在池中移動或釋放席位，
與 `owner_refcounts` 日誌（acquire/release）對稱。

### 池凍結於最後已最終化檢查點（`FinalizedBase.pool`）

接受檢查點時（`accept_checkpoint`），最終性基礎
捕捉**已最終化區塊當時**狀態的池：

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(已最終化區塊的時間戳)`
  — 之後最終性絕不重讀活躍狀態：檢查點後的
  任何註冊（聲明、轉移、禁權）只對
  之後的 epoch 有效；
- `seed` = `next_seed(signing_hash(checkpoint))` — 下個
  epoch 的種子由檢查點的**內容**雜湊衍生
  （帶鍵、簽章集合除外 — 對同一檢查點持有
  不同法定子集的兩個視圖選出相同的
  委員會）。

### 委員會抽籤（引導、新鮮度、降級）

`committee(recovery)`：

1. **引導**（無已最終化檢查點）：池的
   **新鮮營運者**簽署（`fresh_operators` —
   最近的心跳、與已最終化體制相同的窗口）。
   該過濾器是鏈狀態（心跳在鏈上），因此
   依尖端決定性：位於同一尖端的兩個節點
   算出同一集合。新鮮營運者低於
   `MIN_FINALITY_COMMITTEE_SIZE` 時安全降級到
   整個池（冷引導、全面故障 — 建構上
   最終性絕不喪失）。原始池不能充當簽署
   集合：每次自助註冊都會增大法定數的
   分母、而那些擁有者從不簽署 — 約
   10 次註冊後最終性變得不可達（testnet、
   2026-10-21）。
2. **已最終化體制**：從凍結池的**新鮮**子集抽籤
   （RFC proof-of-service §5.2）— 每個席位（擁有者
   金鑰）由尖端的狀態解析至其有效
   營運者（`SetTldOperator`／`SetDomainOperator`
   委派 = 最近的鏈上事實，或自營時的擁有者
   金鑰）；席位以**其有效金鑰**參與（即
   簽署的那把），前提是有**新鮮心跳**（滑動
   窗口）。被禁金鑰保持排除。
3. **安全降級**：新鮮營運者少於
   `MIN_FINALITY_COMMITTEE_SIZE`（4）→ 從整個
   池抽籤（冷引導、全面營運者故障 — 建構上
   最終性絕不喪失）。

`allowed_producers(block_ts)`（區塊生產）遵循一條
刻意更寬的規則：引導時整個池；已最終化
體制中為**整個凍結池**。選出的委員會（+
復原抽籤）仍然只是**檢查點簽署**集合。將
生產收窄至委員會曾被先規範、後實作、隨後
移除：視圖落後一個檢查點的生產者會按引導
規則評估候選區塊、其產出的每個區塊都被
已最終化的 peer 拒絕（`InvalidProducer`）、且
跨轉換期拼出的任何鏈歷史永遠無法通過
`pool_at_height` 重播 — 最終性永遠凍結在
第一個檢查點（sim 重現器 `duplicate_and_delay`、
2026-10-21）。僅憑池凍結就承載了安全性：
成員變更（註冊、讓與、封禁、到期）只在
下一個檢查點生效，因此生產者集合保持為
已最終化鏈的純函數。

檢查點法定數為 `quorum_for(委員會規模)`
（floor(2n/3)+1）；低於 `MIN_FINALITY_COMMITTEE_SIZE`
的委員會絕不最終化（BFT 防護）。

## 延後至共識

本 crate 中刻意不定義：

- **記憶體池**：細粒度准入／重排（費用、替換）
  留在中繼站（M4 帶來每網域上限 + 開窗
  TXID 防重播 — `REPLAY_WINDOW_BLOCKS = 256`）；
- **全域排序**：記憶體池排序是本機決定性排序
  （中繼站），不是全域權威；
- GC 之外的**時間戳權威**：到期（M8b）使用父
  區塊的時間戳；其他時間規則（細粒度
  防重播）保留在共識；
- 分散式儲存：狀態在記憶體中；後端
  （`scone-storage`、redb）套用這些規則而不重複
  它們。

自 M8b 起交付（先前延後）：每網路註冊
PoW（`RegisterTld` 恆為、開放 TLD 上的
`RegisterDomain`）、到期／續期 1 年 + 30 天寬限 +
3 年視野、父區塊時間戳的決定性 GC、
跨網路重播不可能（創世 + 簽署欄位 + PoW 摘要）。

## 決定性分散式模擬（P1.6）

`crates/scone-blockchain/src/sim/` 包含**決定性**
網路模擬框架（`SimNet`）：N 個真實的
`Blockchain`（無 mock），由**虛擬時鐘**與
以 `(tick, 插入順序)` 排序的**事件佇列**驅動。
沒有 socket、沒有 async、沒有壁鐘 — 每次
抽取（遺失、延遲、重複）都是
`(seed, 發送序號)` 的純函數，因此固定種子的
情境可逐位元組重現。

### 模型

- **傳輸**：每次發射經歷概率性遺失
  （以 ‰ 可配置）、有界人工延遲
  （`latency_ticks`）、可能的重複（一份延遲副本），
  以及當前的分區映射（跨群訊息遺失）。
  重複絕不推進邏輯時鐘、不觸發新的
  發送：有無重複的真實事件流相同
  （決定性契約，由 `duplicate_and_delay`
  測試證明）。
- **疫病 gossip**：節點只在首次
  見到時向其連線的 peer（同分區）重傳
  區塊／交易／檢查點 — 中繼規則 H2（切斷
  迴圈）。停等（孤兒）區塊絕不
  重播；接收缺口由同步路徑（`GetBlocks`）
  填補，與中繼站相同。
- **生產**：每區間一次全域生產步；每個
  分區群每輪至多產出一個區塊，由持有
  最佳高度的獲授權節點產出。引導期間
  （第一次最終性之前）生產者釘在最低
  節點：獲授權生產者集合由狀態衍生
  （活躍網域池），兩個略有差異的視圖選出
  不同委員會，本機合法的區塊在別處會是
  `InvalidProducer` — 永久分歧。第一次
  最終性後委員會凍結於最終性基礎（任何
  收斂節點上相同），生產自由。
- **最終性**：在 `sign_interval` 高度的每個
  倍數，委員會的指定錨點透過真實
  防當機的 `SignerGuard` 簽署其尖端的決定性
  內容（一把金鑰 = 每 epoch 一票，擴散前
  狀態 fsync）；委員會其他成員
  在收到後簽署（先驗證所引用的區塊確實
  是他們在該高度的 正典區塊 — 中繼層防護）
  並重播豐富化的聚合。達到法定數時
  `accept_checkpoint` 最終化。每節點對每
  epoch 承諾所見的第一個提案（引導防
  雙重最終化防護：同一 epoch 兩個帶互斥
  簽署者的競爭提案無法同時累積）。
- **當機／重啟**：當機的節點失去所有
  RAM 狀態（鏈、記憶體池、停等、視圖）
  但保留磁碟上的 signer-guard 目錄，
  如真實行程；它透過重播 peer 提供的
  正典區塊重新同步
  （`GetBlocks`／`GetCheckpoints`）。

### 涵蓋情境（`sim::tests`）

| 測試 | 證明什麼 |
|---|---|
| `convergence_simple` | 8 節點、5 ‰ 遺失、延遲 1–5 ticks、500 區塊：處處相同尖端、相同 `state_root_smt`；活躍最終性；每 epoch 絕無兩個不相容的檢查點。 |
| `partition_then_heal` | 4a/4b 分區約 300 區塊：兩側皆產出，癒合 → 隨輸分支的 reorg 收斂；交叉的檢查點窗口絕不與共享 epoch 矛盾（癒合前後皆然）。 |
| `crash_restart` | h≈200 當機、h≈400 重啟：透過重播網路區塊重新同步 — 相同高度、尖端、狀態；當機後交易對復活的節點可見。 |
| `duplicate_and_delay` | 10 ‰ 重複：最終高度、狀態根與已最終化檢查點數與無重複的執行相同（冪等：TxReplay／seen 集合吸收重複）。 |
| `concurrent_tx` | 兩個並發的擁有者於同一網域：恰好一次更新獲勝（擁有者 — 入侵者 `NotOwner`、乾淨的型別化拒絕），絕無雙重狀態套用，輸的交易不進任何鏈。 |

### 參數與時長界限

預設值：`produce_interval` 7 ticks、`sign_interval` 16
高度、`sync_interval` 29–60 ticks（預設 41，某些
測試調低）、`horizon` 200k–500k 虛擬 ticks。每個
測試內嵌 debug 模式下 < 60 秒的壁鐘防護
（`assert_under_60s`）。主導成本是驗證核心
（debug 下每個空的 `push_block_with_gc` 約 9 ms、
Ed25519 未最佳化）：情境維持在約 8 節點 ×
500 區塊以下。

### 限制

- **無真實傳輸層**：本框架演練惡劣交付下的
  鏈邏輯，而非 swarm（libp2p、
  request-response、背壓、探索）。真實 e2e
  測試（`crates/scone-network/tests/`）仍是
  傳輸證明。
- **RAM 內批次同步**：提供的區塊來自 RAM
  視窗（情境維持在 `RAM_WINDOW_BLOCKS`
  之下）；未模擬 `BlockPruned` + 儲存重載
  路徑（由鏈的單元測試涵蓋）。
- **誠實錨點模型**：錨點只簽署可對其
  自身正典鏈驗證的內容；拜占庭
  equivocation（蓄意雙簽）由 `SignerGuard`
  防止，但未作為主動攻擊演練。
- **一次一個分區**：群映射是全域的、由
  驅動器循序變更；持續重疊／演變的
  分區未被建模。

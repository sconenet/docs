# 簽署交易（格式 v1）

簽署交易格式的**規範**參考文件。
它規定欄位、精確的線上順序、大小、簽署
酬載與驗證規則，以便在不閱讀 Rust 的
情況下完成獨立實作。參考實作是
[`scone-protocol`](../../../crates/scone-protocol)（編碼）與
[`scone-blockchain`](../../../crates/scone-blockchain)（驗證）。

`PROTOCOL_VERSION` = **1**。只有一套
格式：任何非 `0x01` 的版本位元組（交易
中）或與本機版本不同者（區塊中）都以
`UnsupportedVersion` 拒絕。

## 總覽

```text
Transaction = disc u8 || version u8 || payload
```

| 欄位 | 大小 | 角色 |
|---|---|---|
| `disc` | 1 B | 交易型別（見下表） |
| `version` | 1 B | **恰好 `0x01`**（tx 格式版本） |

**交易不再於線上格式攜帶 `network`
欄位**（2026-09 變更）：`network_id`
（`str` ≤ 16、ASCII `[a-z0-9-]` — 例如
`scone-testnet`、13 位元組）只住在**簽署
酬載**中 — 由簽署者作為第一個欄位注入
（見[簽署酬載](#簽署酬載)）— 並在 PoW 摘要
中保持每網路一份。因此為一個網路
**簽署**的交易絕不在另一個網路上驗證：
其簽章對鏈重新計算（注入自己的網路）的
酬載失敗（`InvalidSignature`，見
`/docs/en/technical/blockchain.md` — 網路與
創世）。

### 判別碼（規範表）

每個判別碼都是**固定的任意不透明常數**：
值被刻意選成互異，使線上格式**絕不**暗示
型別間的邏輯順序（無 0x01/0x02/0x03…
序列）。`0x00` **絕非**有效型別（典型的
損毀位元組）。未來型別取任何未用的自由
值、並一次性載明於本表。

| 判別碼 | 型別 | 角色 |
|---|---|---|
| `0x21` | `RegisterDomain` | 網域聲明 |
| `0x52` | `UpdateDomain` | 網域 DNS 資料的新版本 |
| `0x93` | `RegisterTld` | 頂級網域聲明（M7） |
| `0x47` | `TransferTld` | TLD 轉移給新擁有者（M8a） |
| `0x6B` | `RevokeTld` | TLD 的放棄（M8a） |
| `0xB8` | `SetTldOpen` | TLD 對自助註冊**不可逆**的開放（單向、owner-option；`open=false` 是型別化拒絕） |
| `0xD4` | `AssignDomain` | 由 TLD 擁有者直接指派網域（M8a） |
| `0x3C` | `RenewDomain` | 延長網域註冊（M8a；自 owner-option 起攜帶正典名稱 — 開放 TLD 跟隨到期日） |
| `0x8F` | `RenewTld` | 私有 TLD 3 年任期的續期（與建立等量的 PoW） |
| `0x15` | `SlashTx` | 錨點 equivocation 的證據 + 禁權（M9） |
| `0xE2` | `TransferDomain` | 網域轉移給新擁有者（M8c） |
| `0x71` | `Heartbeat` | 營運者的活躍性證明（proof-of-service RFC） |
| `0xA9` | `SetTldOperator` | 私有 TLD 營運的委派（proof-of-service RFC） |
| `0x5E` | `SetDomainOperator` | 網域營運的委派（proof-of-service RFC） |

任何非 `0x01` 的版本位元組都被明確拒絕
（`UnsupportedVersion(收到的值)`）：未知
或過時的格式絕不被意外解析。

## SLASH_TX（0x15、M9）

錨點在同一 epoch 簽署兩個**衝突檢查點**
（equivocation）的鏈上證據。任何人都可以
提交 — 證據是密碼學的且自含：

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` 檢查結構：同一 epoch、互異的
  `signing_hash`、兩個互異的簽章、舉報者
  對正典酬載的簽章；
- 狀態規則（有日誌、reorg 時可逆）：被指控
  者的**兩個**簽章都必須對同一 epoch 的
  兩個互異簽署雜湊通過；被指控者必須在
  資格池中（否則 `SlashOffenderNotInPool`）；
  然後**禁權** — 金鑰持久地自 PoS 池移除
  （狀態中的黑名單）；
- 最小制裁：被指控者的網域/TLD **不被**
  沒收（已載明的 M9 決策）；
- 同一檢查點兩次 = 非 equivocation（被拒）；
  不同 epoch = 被拒；偽造簽章 = 被拒。

## REGISTER_DOMAIN（0x21）

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `name` | `str` ≤ 253 | 被聲明網域的正典名稱、**以明文攜帶**（見下） |
| `domain_id` | 32 個原始位元組 | **必須**等於 `DomainId(name)`（重新計算、絕不信任） |
| `owner` | 32 個原始位元組 | **必須**是 `public_key` 的衍生（見下） |
| `timestamp` | varint | 排序資訊（Unix、秒） |
| `proof` | `bytes` ≤ 256 | 註冊 PoW 酬載（見[下](#註冊工作量證明pow)） |
| `public_key` | 32 個原始位元組 | 簽署者的 Ed25519 公鑰（RFC 8032） |
| `signature` | **恰好 64 位元組** | 簽署酬載的 Ed25519 簽章 |

### 名稱由交易攜帶

`RegisterDomain` 以明文傳輸正典名稱、後接
其 `DomainId`。一致性
`domain_id == DomainId(name)` 在編碼、解碼
與驗證時重新檢查（縱深防禦）：id 與攜帶
名稱不符的 `RegisterDomain` 無效。
`DomainId` 仍是共識身分（鏈與 DHT 金鑰）；
攜帶的名稱使鏈自我描述、且簽章涵蓋整個
聲明。

## REGISTER_TLD（0x93）

頂級網域的聲明（TLD 登記冊、M7）：

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `tld` | `str` ≤ 63 | TLD 的正典名稱**以明文攜帶**（M8b：PoW 挑戰由名稱衍生、正如 `RegisterDomain` 自帶其名） |
| `tld_id` | 32 個原始位元組 | TLD 的身分：`BLAKE3-256("SCONE-TLD-V1" || tld)` — **必須**是 `tld` 的衍生（重新計算） |
| `owner` | 32 個原始位元組 | `public_key` 的衍生 |
| `timestamp` | varint | 排序資訊（Unix、秒） |
| `proof` | `bytes` ≤ 256 | 註冊 PoW 酬載（見[下](#註冊工作量證明pow)） |
| `open` | `0x00\|0x01` | **開放**誕生：TLD 直接落在 `TldState::Open`（無主、自助 + PoW、以 `open_tld_pow_difficulty` 費率的一次 PoW — 比私有便宜：公共資源、無 PoS 席位亦無排他性）；**私有**誕生：歷史行為（擁有者、PoS 席位、`tld_pow_difficulty`） |
| `public_key` | 32 個原始位元組 | 簽署者的 Ed25519 公鑰 |
| `signature` | **恰好 64 位元組** | 簽署酬載的 Ed25519 簽章 |

`TldId` 空間與 `DomainId` 空間**互斥**
（互異衍生前綴 — 見
`/docs/en/general/naming.md`）：註冊 TLD
`uip`（對 `*.uip` 命名空間的聲明）絕不能
霸佔網域的身分、反向亦然。套用時的狀態
規則：TLD 必須空閒（登記冊與網域登記冊
分離）。

## M8a 家族 — 線上格式

五個型別補全 TLD 登記冊與網域生命週期。
自 M8a 起為規範線上格式；自 M8b 起有
**完整狀態規則**（見
`/docs/en/technical/blockchain.md`）：需要
TLD 擁有者、自助聲明處皆有 PoW、網域側
的到期與寬限。沒有任何酬載在線上格式
攜帶 `network` 欄位（它只住在簽署酬載
中）。

### TRANSFER_TLD（0x47）

```text
tld_id[32] owner[32] new_owner[32] proof.v public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `tld_id` | 32 個原始位元組 | 被轉移的 TLD |
| `owner` | 32 個原始位元組 | `public_key` 的衍生（當前擁有者／簽署者） |
| `new_owner` | 32 個原始位元組 | 受讓者的身分（不透明：任何值在語法上都有效；與可花費金鑰的綁定是 M8b 狀態規則） |
| `proof` | 有界（varint ‖ 位元組） | 反頻繁倒手的讓與證明、挑戰 `SCONE-TLD-XFER-V1 ‖ tld_id`、平式難度 `transfer_pow_difficulty`、epoch 鹽 |
| `public_key` | 32 個原始位元組 | 簽署者的 Ed25519 公鑰 |
| `signature` | **恰好 64 位元組** | 簽署酬載的 Ed25519 簽章 |

無時間戳或序號：一個 TLD 同時恰好一個
擁有者；第一筆被套用的轉移獲勝（僅憑
鏈序）。

### REVOKE_TLD（0x6B）

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

TLD 的自願放棄（之後可由新的
`RegisterTld` 重新聲明）。由當前擁有者
簽署；刻意無時間戳或序號 — 撤銷是對當前
狀態的一次性動作。

### SET_TLD_OPEN（0xB8）

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `open` | 1 位元組 | **嚴格為 `0x00` 或 `0x01`**（正典形式；任何其他值 → 解碼錯誤、絕不強制轉換） |
| `open_proof` | `proof`（長度 varint + 位元組） | **已簽署**欄位；`open = 0x01` 時必填並驗證（開放 PoW、網路的 `open_tld_pow_difficulty`、挑戰 `SCONE-TLD-OPEN-V1 ‖ tld_id`）；`open = 0x00` 時為空（封閉是免費的） |

**封閉**TLD = 僅限指派（網域只能由 TLD
擁有者經 `AssignDomain` 建立）；**開放**
TLD = 任何人都能做註冊 PoW 並聲明空閒
網域（`RegisterDomain`）。**開放**花費
額外的工作量證明（比 TLD 聲明輕：主網
約 3 天 vs 約 1 週）、由擁有者在
`SetTldOpen` 中一次支付 — 證明存於已簽署的
`open_proof` 欄位、簽署後無法被替換。

### ASSIGN_DOMAIN（0xD4）

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] proof.v public_key[32] signature[64]
```

`RegisterDomain` 的鏡像：正典名稱以明文
攜帶、且 `domain_id` 必須等於
`DomainId(name)`（在解碼、編碼與驗證時
重新計算）。由**名稱之 TLD 的擁有者**
簽署（套用時檢查、M8b）；`assignee` 成為
網域的第一個擁有者（不透明、如
`new_owner`）。攜帶自己的平式反濫發證明
（`assign_pow_difficulty`、挑戰
`SCONE-DOMAIN-ASSIGN-V1 ‖ name` — 刻意為平：
命名空間的排他性已在 TLD 建立時付費）。

### RENEW_DOMAIN（0x3C）

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `name` | `str` ≤ 253 | 正典名稱**以明文攜帶**（owner-option：續期延後其下開放 TLD 的死亡、鏈必須知道名稱）；`domain_id` 必須等於 `DomainId(name)`（重新計算） |
| `valid_until` | varint | 新的註冊到期（Unix、秒） |
| `proof` | 有界 | **續期授權**（見下） |

由網域的當前擁有者簽署；`valid_until`
是否真的延長到期（且至多一個續期任期）
是狀態規則（M8b）。

#### 續期授權 — 依 TLD 而定的 PoW 或 PoS

續期的成本取決於其下 TLD 的性質
（套用時、在區塊狀態上評估）：

- **開放 TLD**：續期花費**與建立相同的
  PoW** — 相同挑戰
 （`SCONE-DOMAIN-V1 ‖ 正典名稱`）、相同的
  每標籤長度與每網路難度、相同的 epoch
  有效窗口。網域絕不被免費「撿走」：
  生命的每一任期都以工作支付。
- **私有 TLD**：命名空間是 TLD 擁有者的
  權益（PoS）。交易必須**由 TLD 擁有者**
 （或其受委派的營運者）簽署 — 簽章就是
  授權、`proof` 欄位必須為空（20 個零
  位元組）。TLD 擁有者按自己的規則續期
  其命名空間的網域。

### RENEW_TLD（0x8F）

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `tld` | `str` ≤ 63 | 正典 TLD 名稱；`tld_id` 必須等於 `TldId(tld)`（重新計算） |
| `valid_until` | varint | 必須**恰好**是 `now + 3 年`（固定任期 — 套用時重新衍生、任何其他值被拒） |
| `proof` | 有界 | 續期 PoW（見下） |

由 TLD 當前的**私有**擁有者簽署。套用
規則：

- 僅適用於私有 TLD（開放 TLD 靠其網域
  活著 — 型別化拒絕 `TldIsOpen`）；
- 任期內**及**到期後的 **60 天寬限**內
  有效；超過則 TLD 已死（GC）、拒絕為
  `TldExpiredFrozen`；
- **與建立等量的 PoW**：挑戰
  `SCONE-TLD-V1 ‖ 名稱 ‖ 當前到期(BE64)`
 （防重用後綴把每個 PoW 綁到**一個**
  任期）、與初始聲明相同的依長度難度；
- 寬限期間續期**不**凍結 — 它是過期擁有者
  僅存的可用管理操作。

## UPDATE_DOMAIN（0x52）

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| 欄位 | 型別 | 約束 |
|---|---|---|
| `domain_id` | 32 個原始位元組 | 目標網域 |
| `owner` | 32 個原始位元組 | `public_key` 的衍生 |
| `sequence` | varint | **> 0**、套用時必須等於 `current + 1` |
| `record_hash` | 32 個原始位元組 | 對 DHT 記錄集的承諾 |
| `public_key` | 32 個原始位元組 | 簽署者的 Ed25519 公鑰 |
| `signature` | **恰好 64 位元組** | 簽署酬載的 Ed25519 簽章 |

註：`UpdateDomain` **不**攜帶 `timestamp` —
排序經由 `sequence`。它也不攜帶名稱：
網域已註冊、id 即足夠。

### 嚴格界限

- `public_key`：**恰好 32 個原始位元組**、
  無長度前綴。無法解碼為曲線點的 pk →
  解碼時 `InvalidTransactionKey` 錯誤。
- `signature`：**恰好 64 個原始位元組**、
  無長度前綴、無餘裕。任何其他大小都是
  硬解碼錯誤（截斷被拒）。
- `proof`：長度 varint + 位元組、上限 256
  （`MAX_PROOF_LEN`）、配置前檢查。
- `name`：長度 varint + UTF-8 位元組、
  上限 253（`MAX_NAME_LEN`）、由
  `scone-core` 命名規則重新驗證。

## POW-TO-OWNER — 四種無簽名形式（0x2A、0x57、0x84、0xC3）

與簽署形式並列的四種**加法**變體（絕不是
取代 — D1）：`REGISTER_TLD_POW`（0x2A）、
`RENEW_TLD_POW`（0x57）、`REGISTER_DOMAIN_POW`
（0x84）、`RENEW_DOMAIN_POW`（0xC3）。沒有
一個攜帶簽署者金鑰**或**簽章：**挖出的
工作量證明就是全部授權**。

### 挑戰綁定受益者與網路（D2）

```
challenge = kind_tag ‖ name ‖ pubkey_dest[32] ‖ len(network_id) ‖ network_id
```

- `kind_tag`：家族標籤（TLD 為 SCONE-TLD-V1、
  網域為 SCONE-DOMAIN-V1）— 無跨種類重播；
- `pubkey_dest`：**受益者**的金鑰 — 被交出的
  證明只能由該金鑰消費（無證明竊取、安全的
  委託挖礦）；
- `network_id`：devnet 證明絕不在 testnet 或
  mainnet 上重播。

續期形式附加了簽署家族的反重用後綴：
`‖ current_valid_until`（BE、8 位元組）— 一份
挖出的證明恰好支付一個任期。

### 主體結構

簽署主體的精確鏡像、`pubkey_dest` 頂替
簽署者金鑰、沒有尾隨簽章：

- `REGISTER_TLD_POW`：name ‖ tld_id ‖ owner ‖
  timestamp.v ‖ proof ‖ open(0x00|0x01) ‖
  pubkey_dest — `owner` = hash(pubkey_dest)、
  重新計算、絕不被信任（D4）；open = OPEN
  誕生（較低費率）；private = 嚴格的 3 年
  任期、受益者成為 PoS 席位；
- `REGISTER_DOMAIN_POW`：name ‖ domain_id ‖
  owner ‖ timestamp.v ‖ proof ‖ pubkey_dest —
  僅限 OPEN TLD（私有 → `TldClosed`：私有
  命名空間保留擁有者簽署路徑）；
- `RENEW_TLD_POW` / `RENEW_DOMAIN_POW`：name ‖
  id ‖ owner ‖ valid_until.v ‖ proof ‖
  pubkey_dest。

### 套用防護（D3）

- 鏈上擁有者**還必須是**挑戰的受益者：
  續期絕不能換手 — 只有註冊建立
  新財產；
- 難度、epoch 窗口、寬限與任期是簽署形式
  自己的（D5：進入成本不變）；
- 受益者的金鑰被解析（PoS 池解析），
  恰如簽署者的金鑰。

### txid 與共存

txid 保持內容的正典雜湊；0x2A/0x57/0x84/0xC3
判別碼就是分叉標記 — 舊節點將它們作為
`UnknownDiscriminant` 拒絕、新節點按原樣解碼
v1 形式。`signing_payload` 對全部四種均為
N/A（型別化 `PowNotSignable` 錯誤）：它們
絕不被簽署。

CLI：`scone domain register <name> --to
<pubkey-hex>`、`scone domain mine <name>
--mine-for <pubkey>`、`scone domain renew <name>
--to`、`scone tld register/renew --to`。

## 擁有者衍生

硬規則：**擁有者恆重新計算、絕不信任**。

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

`owner` 與其自身 `public_key` 衍生不符的
交易無效（核心側 `InvalidOwner`、鏈側
`OwnerKeyMismatch`）、在編碼、解碼**及**
鏈驗證時被拒（縱深防禦）。

## 簽署酬載

簽署酬載是交易**去掉簽章欄位**的正典
編碼、前加領域分離標籤 — **並在開頭注入
目標鏈的 `network_id`**：

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

其中 `canonical_encode(tx_without_signature,
network)` 遵循上述線上結構、含判別碼與
版本位元組、在 `signature` 之前停止、
**以 `network` 為第一個欄位** — 這是網路
出現的唯一位置。鏈以**它自己的**網路
重新計算酬載：為另一網路簽署的交易簽章
驗證失敗（`InvalidSignature`）；這是反跨
鏈重播機制。

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] proof public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] proof public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] proof public_key[32]
```

性質：

- 決定性且純粹：同一交易 ⇒ 到處相同的
  酬載；
- ASCII 前綴 `SCONE-TX-SIG-V1`（15 位元組、
  hex `53434f4e452d54582d5349472d5631`）
  防止簽章在協定的其他雜湊領域被重用；
- `signature` 以外的所有欄位都被涵蓋 —
  **包括 `RegisterDomain` 攜帶的名稱**：
  修改其中任何一個都使簽章無效；
- 被注入的 `network_id` 被涵蓋：同一未簽署
  tx 為兩個網路簽署產生兩個互異的酬載
 （因此兩個互異的簽章）。

## 註冊工作量證明（PoW）

`RegisterDomain` 與 `RegisterTld` 的
`proof` 欄位攜帶註冊工作量證明
（`scone-core::pow` 模組）。

### 摘要（epoch 鹽、反預挖）

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocks)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` 是正典 `network_id`（M8b）：為一
個網路挖的證明不是另一個網路上的證明。

`SCONE-POW-V1-ARGON2` 也釘住每網路的
Argon2id 參數（`pow_params`：mainnet
m = 64 MiB、t = 2、p = 1；testnet/devnet
m = 8 KiB、t = 1、p = 1）：Argon2id 核心
（RFC 9106）使每次雜湊的成本由記憶體主導
（抗 GPU/ASIC）、且在一組參數下挖的證明
絕不在另一組下驗證。

`challenge` 是被聲明物件的衍生輸入 —
TLD 為 `"SCONE-TLD-V1" || tld`、網域為
`"SCONE-DOMAIN-V1" || name`：恰好是被
雜湊以得出 `TldId`／`DomainId` 的位元組、
因此證明絕不可跨命名空間或跨物件種類
重播。`epoch_le64` 與 `nonce_le64` 是
epoch 與 nonce 的小端 8 位元組編碼。

當 `digest` 至少有 `difficulty` 個前導零
位元時證明有效。

#### Epoch 鹽 — 反預挖

無鹽的摘要不綁定時間也不綁定高度：任何
人都能今天為數千個名稱預挖證明、然後在
主網上線時爆量註冊（囤積）— 證明曾經是
自含的、在先到的任何區塊上都有效。摘要
以**套用交易之區塊的 epoch** 加鹽：

- `epoch = block_height / POW_WINDOW`、
  `POW_WINDOW = 144` 區塊（65 秒/區塊時
  約 2.6 小時）：給誠實使用者的寬限窗口
  （挖、然後在窗口內提交）、超過則必須
  重挖；
- 證明在線上格式**宣告**其 epoch；套用時、
  宣告的 epoch 必須落在套用區塊的**有效
  窗口**內 — 僅向後容忍（遲交被吸收、
  「為未來」挖礦不可能）、窗口大小由
  `epoch_validity_window(difficulty)` =
  `1 << max(0, difficulty − 16)` 個 epoch
  決定（加寬的窗口：主網的高難度以數天
  到數週開挖、比一個 epoch 長；囤積毫無
  增益、因為窗口 ≈ 挖礦成本 — 爆量的
  每個證明都在窗口**之內**花費其完整的
  `2^difficulty` 次雜湊）。難度 ≤ 16 bits
  時（testnet、長名稱）、嚴格相等規則
  不變；
- 記憶體池准入（盡力而為）：未來區塊高度
  未知 — 中繼站對 `當前高度 + 1` 的
  epoch 驗證；嚴格的套用時驗證仍是
  權威；
- 決定性重播：證明**攜帶**其 epoch、且
  窗口是（難度、區塊 epoch）的純函數 —
  歷史區塊在任何節點上都相同地重新
  驗證。

主網前的破壞性變更：舊的緊湊格式
（12 位元組）被拒、無向後相容。

### 線上酬載（20 個固定位元組）

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

難度與 epoch 隨證明傳輸、但它們是**生產
者宣告的、因此絕不被信任**：驗證者把
難度與註冊種類的**協定常數**比較（宣告
難度與常數不同 — 更低*或*更高 — 的酬載
被拒、`InvalidProof`）、把 epoch 與套用
區塊的比較（不符時 `ProofEpochMismatch`）。
然後 nonce 以常數難度重新驗證、摘要從零
重新計算。

### 難度（**依長度**曲線、M8b + 實測校準）

難度不是平坦參數：它是**被建立標籤長度**
的決定性函數（`scone-core::pow_length` —
`RegisterTld` 為 TLD、`RegisterDomain` 為
最左標籤）、每網路一份：

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| 網路 | 私有 TLD（頂點/下限） | 天生開放 TLD／網域（頂點/下限） |
|---|---|---|
| testnet | 8 bits / 2 bits | 6 bits / 2 bits |
| mainnet | 42 bits / 30 bits | 40 bits / 30 bits |
| devnet | 2 bits / 1 bit | 1 bit / 1 bit |

**天生開放**的 TLD（`RegisterTld` 攜帶
`open = 0x01`）支付「開放」曲線（頂點 −
2、如 `domain_difficulty`）；**開放既有
私有 TLD**（`SetTldOpen`）維持平難度
（`open_tld_pow_difficulty`：testnet
6 bits／mainnet 22 bits／devnet 2 bits —
交易只攜帶 `tld_id`、名稱已於建立時為其
長度付費）。主網基準（私有 TLD、約
6 M 雜湊/s）：1 字元 = 42 bits（≥ 7 天）、
5 字元 = 40 bits（約 2 天）、19 字元 =
35 bits（約 1.6 小時）、29+ 字元 = 30-bit
下限（約 3 分鐘）。曲線住在
`scone-core::pow_length`（常數
`MAINNET_APEX = 42`、`MAINNET_FLOOR =
30`、`TESTNET_APEX = 8`、
`TESTNET_FLOOR = 2`、`DEVNET_APEX = 2`、
`DEVNET_FLOOR = 1`）；平坦參數
（`tld_pow_difficulty` 等）仍在
`scone-core::network::NetworkParams`。改變
一個值 = 該網路的共識變更。不變量：devnet
是完整的第三個 `NetworkParams`、且
testnet 與其 devnet 前的常數（Argon2id
參數、曲線、窗口）嚴格相同 — devnet 與
其他網路除程式碼本身外不共享任何常數。

平坦頂點的實測校準（2026-10、`cargo run
--release -p scone-core --example pow_bench`
— 參考 x86-64 上單執行緒吞吐約 7.7 M
H/s）：

| 種類（mainnet） | 難度 | 平均實際時間 |
|---|---|---|
| `RegisterTld` | 24 bits | 約 7 秒 |
| `RegisterDomain` | 20 bits | 約 54 毫秒 |

以約 65 秒/區塊：TLD 聲明花費約 0.1 個
單核 CPU 區塊、網域約 1 毫秒個區塊 —
對使用者是象徵性的、對規模化工業化則
昂貴（反囤積）。若硬體目標下降（行動
ARM：慢約 4–8 倍）需重新校準。

狀態分支（M8b、已交付）：`RegisterTld`
**恆**需要其 PoW；`RegisterDomain` 在
**開放** TLD 上需要它（封閉路徑是僅限
指派 — `AssignDomain`、由 TLD 擁有者
簽署、帶自己的平式反濫發證明）。

**讓與與指派證明（共識層級的反濫發）**：
`TransferTld`、`TransferDomain` 與 `AssignDomain`
各自攜帶**平式**的工作量證明 — 兩種讓與用
`transfer_pow_difficulty`（挑戰
`SCONE-TLD-XFER-V1 ‖ tld_id`／
`SCONE-DOMAIN-XFER-V1 ‖ domain_id`）、指派用
`assign_pow_difficulty`（挑戰
`SCONE-DOMAIN-ASSIGN-V1 ‖ name`）。與註冊相同
的 epoch 鹽與反預挖窗口；在記憶體池准入時
驗證（估計 epoch、盡力而為）、然後在套用時
驗證（嚴格）。目標：讓與倒手與私有 TLD 下的
大量建立的代價是共識層級的工作、而非政策。
無按長度課稅：對指派而言，命名空間的排他性
已在私有 TLD 建立時付清。

**被動讓與**：這些取得路徑（`TransferTld`、
`TransferDomain`、`AssignDomain`）僅由讓與方
簽署 — 絕不徵詢受讓方。這是有意為之的
選擇（贈與、死投、遺贈）、沒有可強加的
PoS 債務：見 `security-model.md`（被動取得與
資格）。

## 驗證規則

一筆交易若且唯若以下條件才有效：

1. **嚴格解碼**：遵守上述結構與界限
  （已知判別碼、版本 `0x01`、pk 32 B、
   簽章 64 B、核心不變量在解碼時重新
   檢查）；
2. **擁有者/pk 綁定**：`owner ==
   OwnerId::from(public_key)`
  （重新計算）；
3. **名稱/id 一致性**（RegisterDomain）：
   `domain_id == DomainId(name)`
  （重新計算）；
4. **簽章**：對以鏈的網路**從零重新計算**的
   簽署酬載執行 `signature` 的 Ed25519
   `verify_strict`（絕不取自 peer 或按
   交易原樣）— 為另一網路簽署的 tx 在此
   失敗（`InvalidSignature`、反跨鏈
   重播）；
5. **狀態規則**（在區塊中套用時 — M8b、見
   `/docs/en/technical/blockchain.md`）：
   RegisterTld → TLD 空閒 + 非 ICANN +
   PoW（依長度難度）；RegisterDomain →
   TLD 已註冊**且開放** + 網域空閒
  （含寬限）+ PoW（依長度）；
   UpdateDomain → 網域存在、當前
  `owner`、`sequence == current + 1`；
   TransferTld / RevokeTld / SetTldOpen /
   AssignDomain → 存在**且私有**的 TLD、
   簽署者 = TLD 的擁有者；
   TransferDomain → 網域存在、擁有者；
   RenewDomain → 網域存在、嚴格延長
   ≤ 3 年、PoW 授權（開放 TLD：與建立
   相同的 PoW、**綁定當前到期日** —
   有別於註冊與先前每次續期的挑戰）
   或 PoS（私有 TLD：TLD 擁有者 — 或其
   有效營運者 — 的簽章、`proof` 為空）。

驗證使用 `verify_strict`（拒絕可延展簽章
與弱式／非正典金鑰）、如不可信資料所
要求。

## `push_block` 中的操作順序

密碼學驗證（2 到 4）對每筆交易在狀態
套用**之前**進行、在每區塊的原子工作
狀態上；任何錯誤都使鏈嚴格不變。

## 區塊組裝

`BlockBuilder`（`scone-blockchain`）從
交易佇列組裝區塊：`tx_root` 對有序清單
重新計算、標頭鏈結到父雜湊
（`prev_hash`）、`version` =
`PROTOCOL_VERSION`。共識酬載（簽署的
生產者酬載）與時間戳由呼叫者提供；既有
的 `Consensus` trait 與驗證對稱使用。

## 十六進位範例（真實、可重現的向量 — M8b）

### RegisterDomain `example.uip`、時間戳 42、testnet

簽署酬載（釘取向量；歷史上由已移除的
離線工具產生 — 這些值仍是有效向量）。
`network` 出現在這裡、因為這是**簽署
酬載**（它唯一的位置）；已簽署的線上
交易本身不含它：

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

### RegisterTld `uip`、時間戳 42

簽署酬載（釘取向量）— `TldId` 是
`/docs/en/general/naming.md` 中釘死的衍生
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

（完整的值曾可經已移除的離線工具重現；
以下的向量保持釘死且有效。PoW 以網路
難度自動開挖 — nonce 因執行而異；只有
網路/名稱/id/擁有者/pk 是穩定的。自
2026-09 線上格式變更起、已簽署交易的
十六進位**不再**包含位元組
`0d 73636f6e652d746573746e6574`：網路只在
被簽署的酬載中旅行。）

## 十六進位現在住在哪裡

離線的 `tx build|sign|verify` 指令已移除
（cli-simplify-v1）。已簽署的十六進位格式
本身不變、仍是 **web/API 流程**的線上
格式：註冊服務的 web 應用經 API 以正典
十六進位組建、簽署（瀏覽器內 keystore）
並提交交易。CLI 側、使用者路徑是一條
指令的形式：

```bash
scone domain register example.uip        # mines, signs, submits
scone domain renew example.uip                      # renewal proof, signs, submits
scone domains                            # what you own
```

在所有程式碼路徑中、擁有者與金鑰都從
簽署身分**重新計算**、絕不取自任何輸入。

## 版本歷史

| 版本 | 內容 | 狀態 |
|---|---|---|
| 1 | 內嵌 `public_key` + `signature`、酬載 `SCONE-TX-SIG-V1`、版本位元組 `0x01`、`RegisterDomain` 攜帶名稱、`RegisterTld`（0x93）、M8a 家族：`TransferTld`（0x47）、`RevokeTld`（0x6B）、`SetTldOpen`（0xB8）、`AssignDomain`（0xD4）、`RenewDomain`（0x3C）— 規範線上格式；M8b：每個酬載開頭簽署 `network` 欄位、`RegisterTld` 攜帶名稱、完整狀態規則、每網路 PoW 難度 | 已被取代 |
| 1（2026-09、無 network 的線上格式） | **自所有交易的線上格式移除 `network` 欄位**：network_id 只住在簽署酬載中（`signing_payload(tx, network)` 在開頭注入）；網路分離依賴簽章（外來酬載上 `InvalidSignature`、`WrongNetwork` 錯誤消失）；`TX_FORMAT_VERSION` 維持 `0x01`（可重啟的 testnet、無共存）；**所有 TxId 改變**（TxId 是已簽署線上編碼的雜湊、少了 14–15 位元組）；PoW 摘要、創世與每網路 data-dir 不變 | 已被取代 |
| 1（2026-10、PoW epoch 鹽） | **反預挖 PoW 摘要**：鹽 `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + 對 `nonce_le` 的 Argon2id 核心、`epoch = 套用 tx 之區塊的 block_height / 144`；證明線上酬載 12 → **20 位元組**（`nonce_le ‖ epoch_le ‖ difficulty_le`）、宣告的 epoch 對套用區塊的 epoch 有效窗口驗證（型別化拒絕 `ProofEpochMismatch`）、記憶體池准入時盡力而為的 `(height+1)/144`、離線路徑上的 `--epoch` CLI 旗標；主網前的破壞性變更：舊的 12 位元組酬載被拒、**所有 TxId 改變**（`proof` 欄位多 8 位元組） | 現行 |

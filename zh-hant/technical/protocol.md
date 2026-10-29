# 協定

Scone 協定的參考文件：二進位格式、正典
編碼、交易、DNS 記錄、區塊、P2P 訊息、限制與
版本相容性。參考實作是
[`scone-protocol`](../../../crates/scone-protocol) crate。

標記**（暫定）**的項目尚未定案。

## 協定版本

- 常數：`PROTOCOL_VERSION: u32 = 1`
  （`scone-protocol`）；
- 由 `Hello.version`（交握）與
  `BlockHeader.version` 攜帶；
- 相容性規則：
  - 收到的版本**高於**本機版本 → 拒絕
    （`ProtocolError::UnsupportedVersion`）；節點
    不猜測它不認識的版本之格式；
  - 版本 `0` → 無效；
  - 只有一套格式：交易版本位元組
    （`0x01`）與區塊版本必須與本機值**完全**
    相符；
- 此階段無 major/minor 拆分：單一整數，
  任何線上格式破壞性變更時遞增。

## 二進位格式原則

- 二進位，協定資料絕不用 JSON；
- 決定性且正典：兩個邏輯相等的物件產生
  **完全相同的位元組**；
- 循序解析、盡可能無配置（位元組借用）；
- 所有網路輸入皆不可信：長度與計數器
  在任何配置**之前**檢查；解碼絕不 panic。

### 字節序

- 所有整數皆 LEB128 varint（7 位元群組、
  最低有效在前）：不適用原生子節序；
- IPv4/IPv6 位址與 32 位元組識別碼：原始
  位元組陣列、無長度前綴；
- 格式中沒有固定大小的寬整數：單一的
  整數編碼規則。

### Varint（最小 LEB128）

每個位元組攜帶 7 個資料位元；除了最後
一個位元組外高位元皆為 1。`u64` 佔 1 到
10 個位元組。

| 值 | 位元組（hex） |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16,384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

編碼必須**最小**：超長形式在解碼時被拒
（`InvalidVarint`）。這是一個值只有唯一
有效位元組序列（正典編碼）的必要條件。
拒絕：`80 00`（非最小）、`ff×9 02`（u64
溢位）、`ff×10`（過長）。

### 字串與位元組

- `bytes`：varint 長度 + 原始位元組；
- `str`：相同，但強制 UTF-8 解碼；
- 每個長度都有上限（見[限制](#限制)）並在
  讀取前檢查。

### 識別碼

| 型別 | 線上表示 |
|---|---|
| `DomainId` | 32 個原始位元組 |
| `TldId` | 32 個原始位元組 |
| `OwnerId` | 32 個原始位元組 |
| `PublicKeyRef` | 32 個原始位元組 |
| `RecordHash` | 32 個原始位元組 |
| `BlockHash` | 32 個原始位元組 |
| `MerkleRoot` | 32 個原始位元組 |
| `DomainName` | `str`（≤ 253 位元組、由 `scone-core` 重新驗證） |

衍生（`BLAKE3-256` + `SCONE-DOMAIN-V1` 前綴
等）仍由 `scone-core` 定義（見
`/docs/en/general/naming.md`）。

## 交易

```text
Transaction = disc u8 || version u8 || payload
```

| 判別碼 | 型別 |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

每個判別碼都是任意的不透明常數（無邏輯
順序；`0x00` 永遠無效）— 完整規範表在
`/docs/en/technical/transactions.md`。

`version` 位元組（當前值 `0x01`）跟在判別碼
之後：任何其他值都被明確拒絕
（`UnsupportedVersion(value)`）。大多數交易
皆經簽署（Ed25519）— 除了四種無簽名的
pow-to-owner 形式（`0x2A`、`0x57`、`0x84`、
`0xC3`）：挖出的工作量證明即是授權，其挑戰
綁定受益者金鑰與網路。完整規格、簽署酬載
與驗證規則見
**`/docs/en/technical/transactions.md`**（規範）。

自 2026-09 線上格式變更起，**交易不再攜帶
`network` 欄位**：`network_id`（`str` ≤ 16）
注入**簽署酬載**（簽署形式）或 **PoW 挑戰**
（pow-to-owner 形式）— 為一個網路製作的
交易永遠不會轉移到另一個網路（見
`/docs/en/technical/blockchain.md`）。

### REGISTER_DOMAIN（0x21）

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` 以**明文攜帶正典名稱**，
後接其 `DomainId` — 一致性
`domain_id == DomainId(name)` 在解碼時重新
檢查（重新計算、絕不信任）。`proof` 攜帶
註冊 PoW（20 位元組：nonce + epoch + 難度 —
套用區塊的 epoch 鹽 `epoch = height/144`、
在套用時對網路常數與區塊 epoch **嚴格**
驗證；不符時型別化拒絕
`ProofEpochMismatch` — M8b、見
`transactions.md`）。`public_key` 是簽署者
的 Ed25519 金鑰（恰好 32 位元組）；`owner`
必須是該金鑰的 BLAKE3 衍生（重新計算、
絕不信任）；`signature` 恰好 64 位元組。

### REGISTER_TLD（0x93）

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

TLD 的聲明（登記冊與網域登記冊分離、
M7）：TLD 與網域名稱一樣**以明文攜帶**
（M8b — PoW 挑戰由名稱衍生），`tld_id` 是
該名稱的 `SCONE-TLD-V1` 衍生、與每個
`DomainId` 在建構上互斥。狀態規則（M8b）：
TLD 空閒 + 非 ICANN 保留
（`IcannTldReserved` — ICANN 根 TLD 無人可
聲明）+ 以標籤**依長度**難度的 PoW
（`pow_length` 曲線 — 見 `blockchain.md`）；
聲明後為封閉（僅限指派）。

### UPDATE_DOMAIN（0x52）

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

解碼時重新檢查的不變量：`sequence > 0` 且
`owner` == `public_key` 的衍生。無
`timestamp`：排序經由 `sequence`。交易只攜帶
緊湊的參照 — 完整 DNS 內容留在 DHT（見
下文）。

## DNS 記錄（DHT 側）

完整 DNS 記錄絕不進入區塊鏈：它們經由
DHT（`GetRecord`／`Record` 訊息）交換，並
在鏈上僅以 `RecordHash` 承諾。

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

型別碼是**真實的 DNS 碼**（IANA）。TTL
（秒、varint）跟在型別碼之後並涵蓋整筆
記錄：界限 `MIN_TTL = 1` .. `MAX_TTL =
2,592,000`（30 天；預設
`DEFAULT_TTL = 86,400`）。

| 碼 | 型別 | 酬載 |
|---|---|---|
| 1 | A | 4 位元組 |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`、`name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 位元組 |
| 33 | SRV | `priority.v`、`weight.v`、`port.v`、`target(name)` |
| 257 | CAA | `flags u8`、`tag`（1–15 B）、`value`（`str` ≤ 4096） |
| 32768 | GEOROUTE | GeoRoute（見下） |
| 其他 | Unknown | code.v + `bytes` ≤ 4096 |

範例：`A 192.0.2.1`（TTL 86,400）→
`01 80 a3 05 c0 00 02 01`
（型別、TTL varint、位址）。

攜帶已知碼的 `Unknown` 在編碼時被拒
（解碼絕不會產生一個）。

#### GEOROUTE（GeoDNS）

Scone 私有型別（非 IANA 範圍）。一條地理
路由規則：

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- world
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numeric
          | 0x03 region.v           -- region code
          | 0x04 asn.v              -- AS number
          | 0x05 prefix.addr[16] || prefix.len u8   -- IP prefix (normalized)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- binary IPv4
          | 0x01 octets[16]         -- binary IPv6
```

**正典規則**：給定範疇的端點以**排序**
編碼（優先權升序、權重降序、目標升序）；
排序在編碼時套用，未排序的線上格式在
解碼時被拒（`NonCanonical`），與記錄集
相同。完整的端點重複被拒。

**驗證**：`priority > 0`、`weight ≤ 100`（0 =
不活躍端點）、每範疇 ≤ 32 個端點
（`GEO_MAX_ENDPOINTS`）。本機表
（`continent_of_country`）不認識的
國家／大陸碼在線上層級**被接受**：解析器
的表可能比發布者的更新。TTL 是父
`RecordData` 的（不變）。

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**正典規則**：記錄集是無序集合。它以
**依每筆記錄個別編碼的字典序排序**
編碼、嚴格遞增（重複被拒）。未排序的
線上格式在解碼時被拒（`NonCanonical`）。
原始 `Vec` 的順序不被傳送。

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

雜湊僅涵蓋**記錄內容**；擁有者的簽章
在相同的正典位元組上另行驗證。*（決策：
承諾不涵蓋簽章 — 本文件的早期版本提到
「完整的 SignedDnsRecord」；把雜湊耦合於
簽章一無所獲，且把承諾綁死於簽章
方案。）*

## 區塊

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`：協定版本（上述規則）；
- `height`：高度（創世 = 0）；
- `prev_hash`：前一區塊的雜湊（創世區塊
  為零）；
- `tx_root`：對交易清單的承諾（由
  `scone-blockchain` 定義的 Merkle 樹，見
  `/docs/en/technical/blockchain.md`）；
- `timestamp`：排序資訊（Unix、秒）；
- `consensus`：生產者酬載（`SCONE-BLOCK-V1 ‖
  pk[32] ‖ sig[64]`、見 `blockchain.md`）—
  網路的創世攜帶原始 `network_id`，而配置
  創世（主網 kickstarter）可攜帶
  `ALLOC-V1` 清單（≤ 60 KiB）；由
  `MAX_CONSENSUS_LEN`（64 KiB）約束。

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

循序解析；計數器在配置**之前**封頂；
任何截斷都被拒。與記錄集不同，交易順序
**有意義**（共識順序）且被精確保留。

`BlockHash`（正典標頭的雜湊）與 `tx_root`
由 `scone-blockchain` 對此處定義的編碼計算
（精確公式見 `/docs/en/technical/blockchain.md`）。

## P2P 訊息

```text
Message = disc u8 || payload
```

傳輸（框架、libp2p…）不在範圍內；傳輸框
不應超過 `MAX_MESSAGE_LEN`（1 MiB）。

| Disc | 訊息 | 方向 | 酬載 | 限制 | 預期回覆 |
|---|---|---|---|---|---|
| `0x01` | `Hello` | 雙向 | `version.v` | — | `Hello` |
| `0x02` | `Ping` | 雙向 | — | — | `Pong` |
| `0x03` | `Pong` | 雙向 | — | — | — |
| `0x04` | `GetBlock` | 雙向 | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | 雙向 | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | 雙向 | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | 雙向 | `Transaction` | — | — |
| `0x08` | `GetRecord` | 雙向 | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | 雙向 | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | 雙向 | `Checkpoint` | 簽署者 ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | 雙向 | — | — | 檢查點窗口 |
| `0x0c` | `GetStatus` | 雙向 | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | 雙向 | 節點狀態（網路、尖端、高度、NAT、網域、記憶體池、peer、最終性、新鮮營運者） | — | — |
| `0x0e` | `LookupName` | 雙向 | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | 雙向 | 名稱 + TLD 的鏈上狀態、含 CAS 世代（變更計數器、registered_at） | — | — |
| `0x10` | `GetOwnedNames` | 雙向 | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | 雙向 | 擁有者的活躍網域 + TLD | — | — |
| `0x12` | `Error` | 回覆 | 應用層錯誤訊息（`str`） | — | — |

### Checkpoint（M5）

聚合檢查點的線上格式為：標籤
`SCONE-CKPT-V1` || 本體 `CheckpointData`
（116 B、LE — 與簽署原像的結尾相同：
`epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32)
recovery(4)`）|| `count.v` || 每個簽署者
`pk[32] ‖ sig[64]`，**依 pk 升序排序、無
重複**、`count ≤ 64`（`MAX_CHECKPOINT_SIGNERS`）。
嚴格解碼：截斷、錯誤標籤、非正典 pk、
重複或未排序 = 拒絕（`None`），絕不
panic（損毀模糊測試已測）。

`GetCheckpoints` 請求最終性窗口；回覆是
`checkpoint_window()` 窗口（至多
`CHECKPOINT_KEEP` 個檢查點、由舊到新），
經一或多個 `Checkpoint` 訊息送達。

`TxId` 現由 `scone-blockchain` 定義
（`BLAKE3-256("SCONE-TX-V1" ||
canonical(Transaction))`、見
`/docs/en/technical/blockchain.md`）。記憶體池
與開窗防重播（`TxReplay`、
`REPLAY_WINDOW_BLOCKS = 256`）已在中繼站
交付（M4）；僅 `GetTransaction` 仍延後
（其定義取決於其餘共識決策）。

## 區塊鏈／DHT 拆分

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

鏈說明*誰*擁有與*當前狀態為何*（序號 +
record_hash）。DHT 提供*內容*。一筆 DHT
記錄若且唯若以下條件才被接受：

1. 其簽章對鏈上註冊的擁有者有效；
2. 其 `sequence` 與鏈上的一致（或更高）；
3. 對其正典編碼重新計算的 `RecordHash` 與
   鏈上的 `record_hash` 相符。

## 限制

| 常數 | 值 | 角色 |
|---|---|---|
| `MAX_NAME_LEN` | 253 | 名稱長度 |
| `MAX_TLD_LEN` | 63 | TLD 長度 |
| `MAX_TXT_LEN` | 4096 | TXT 字串 |
| `MAX_PROOF_LEN` | 256 | 註冊證明 |
| `MAX_SIGNATURE_LEN` | 128 | 簽章（Ed25519 = 64） |
| `MAX_RECORDS_PER_SET` | 256 | 每集合記錄數 |
| `MAX_DHT_RECORD_LEN` | 64 KiB | 編碼後的已簽署 DNS 記錄（DHT 酬載：線上、儲存、gossip）— 總界限在編碼時與任何解碼前檢查；高於任何現實集合（SPF+DKIM+DMARC ≈ 2–5 KiB）15–30 倍、低於傳輸上限 16 倍。 |
| `GEO_MAX_ENDPOINTS` | 32 | 每個 GeoRoute 範疇的端點數 |
| `MAX_UNKNOWN_DATA` | 4096 | 未知型別的資料 |
| `MAX_TXS_PER_BLOCK` | 4096 | 每區塊交易數 |
| `MAX_CONSENSUS_LEN` | 64 KiB | 標頭的共識酬載（為 `ALLOC-V1` 創世清單加寬 — 主網 kickstarter） |
| `MAX_BLOCKS_PER_REQUEST` | 128 | 每次 `GetBlocks` 的區塊數 |
| `MAX_MESSAGE_LEN` | 1 MiB | 傳輸框（建議） |

任何超限 → `LimitExceeded`、在配置前
偵測：惡意 peer 無法在解碼時造成不相稱的
配置。

## 正典編碼

根本規則：

```text
canonical object → canonical binary encoding → hash / signature
```

- 僅最小 varint；
- 記錄集嚴格遞增排序；
- 邏輯相等的物件恰好有**一**種有效編碼；
- 解碼拒絕非正典形式（`NonCanonical`），
  因此同一物件沒有多種有效編碼。

## 錯誤處理

`ProtocolError`（`scone-protocol::error`）：

| 變體 | 意義 |
|---|---|
| `Truncated` | 輸入在某個值的中間結束 |
| `InvalidVarint` | 非最小、溢位或過長的 varint |
| `IntegerOutOfRange` | 整數放不進目標型別 |
| `UnsupportedVersion` | 零或高於本機的版本 |
| `UnknownDiscriminant` | 未知的交易／訊息判別碼 |
| `LimitExceeded` | 長度或計數超過限制 |
| `InvalidUtf8` | 非 UTF-8 字串 |
| `TrailingBytes` | 完整值之後剩餘的位元組 |
| `Validation` | 違反 `scone-core` 不變量 |
| `NonCanonical` | 逐欄位有效但非正典的編碼 |

解碼絕不因網路資料 panic。

## 簽章

- 方案：**Ed25519**（RFC 8032），經
  `scone-crypto`（`ed25519-dalek`、
  `verify_strict` 驗證 — 可延展簽章與弱金鑰
  被拒）；
- **鏈上交易**：每個 `RegisterDomain`／
  `RegisterTld`／`UpdateDomain` 內嵌
  `public_key`（32 B）+ `signature`（64 B），
  對酬載 `"SCONE-TX-SIG-V1" ||
  canonical_encode(tx_without_signature)`；
  完整規範規格：
  `/docs/en/technical/transactions.md`；
- **DHT 記錄**（`SignedDnsRecord`）：簽署
  酬載是 `DnsRecord` 的正典編碼；驗證以
  區塊鏈中註冊的身分交叉比對
  `record + owner + signature`；
- 橫切規則：`owner` 與所有雜湊／承諾皆
  **重新計算**，絕不取自提供的值。

## 共識規則

已交付（描述於 `/docs/en/technical/
blockchain.md`）：簽署的區塊生產者
（`SCONE-BLOCK-V1` 酬載、檢查點選出的
委員會）、以聚合檢查點實現的最終性
（`SCONE-CKPT-V1`、最終性下限處的分叉
選擇）、依名稱長度的註冊 PoW。
剩餘主題：細粒度反垃圾費用、擁有者金鑰
輪換。

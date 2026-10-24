# 架構

## 先用白話說明

五個概念就足以理解其餘的一切：

1. **名稱** — 網際網路上的一個位址，例如 `mysite.scone`。
2. **你的 24 個字詞助記詞** — 你的身分。它是證明名稱屬於你的
   權狀，並為你所做的每項變更簽名。
3. **公開的登記簿**（區塊鏈）— 共享、公開的帳本，記錄誰擁有
   什麼。沒有人控制它，也沒有人
   能改寫它。
4. **挖礦** — 你保留名稱的方式：你的電腦做一些
   對每個人都公平且相等的工作。沒有排隊，沒有中間人。
5. **中繼站** — 一台透過保存帳本副本並回答問題來運作網路的
   電腦。任何人都可以運行
   一台；數量越多，網路越強壯。

本頁其餘部分更深入 — 你可以在此停住，
卻已經理解了這個專案。

## 指導原則

Scone 將權威與可用性分離：

- **區塊鏈**是所有權與變更順序的事實來源；
- **DHT** 是一套散布／複製系統，它**不是**權威；
- **本機儲存**保存節點的狀態、索引與快取。

節點必須將來自 DHT 的任何資料視為潛在
惡意：只有在密碼學驗證通過後才會被接受
（擁有者簽章、與鏈上的所有權／序號／
record_hash 比對）。

## 權責分離

```text
BLOCKCHAIN
──────────
Ownership
Registration
Authorization
Ordering
Sequence
Record hash
Consensus information

DHT
───
DNS records
Signed records
Replication
Availability

LOCAL STORAGE
─────────────
Local state
Indexes
Cached DHT records
Blockchain state
```

## Crates 與相依關係

```text
scone-crypto     ← lowest layer: pure primitives over bytes
    ▲
scone-core       ← protocol types (depends on scone-crypto)
    ▲
scone-protocol  ← wire/blockchain binary format
    ▲
scone-blockchain← blockchain logic, canonical state (in memory)
    ▲               └ scone-topology (pure engine, zero network/fs/
                     async; fed by scone-network) — scone-genesis
                     (one-shot mainnet genesis tool binary)
scone-storage   ← local persistence: redb backend + verified snapshots
    ▲
scone-network   ← p2p relay: libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← encrypted `.sconekey` keyfiles (depends on scone-crypto)
    ▲
scone           ← binary / application
```

### 決策：`scone-core` 相依於 `scone-crypto`

`DomainId` 必須能從最底層開始計算
（`BLAKE3-256(prefix || canonical name)`），但具體的密碼學
不得污染核心。解法：

- `scone-crypto` 是最底層：對位元組的純決定性函數
  （加上用於金鑰產生的 OS RNG）、零核心型別、
  沒有網路／檔案系統／async 相依 → 不可能出現循環；
- `scone-core` 僅將它用於雜湊；
- Ed25519 簽章與金鑰位於 `scone-crypto`，並作用於
  位元組（由 `scone-protocol` 產生的正典酬載）。

### 各 crate 的角色

**scone-core** — 名稱（`TldName`、`DomainName`）、識別碼
（`DomainId`、`OwnerId`、`PublicKeyRef`）、DNS 資料（`DnsRecord`、
`SignedDnsRecord`）、交易（`RegisterDomain`、`UpdateDomain`）。
決定性且可移植。此 crate 中禁止：libp2p、redb、
tokio/async、網路、檔案、OS、DNS 伺服器、區塊鏈邏輯。

**scone-crypto** — 密碼學原語。目前：`hash256`
（BLAKE3）、Ed25519 簽章與金鑰（`SigningKey`、
`PublicKey`、`Signature`、嚴格批次驗證）、記錄群組的
AEAD 信封（XChaCha20-Poly1305）。計畫中：Merkle 樹。
規則：不自行發明演算法，僅使用經稽核的相依套件。

**scone-protocol** — 協定的二進位格式：交易、DNS 記錄、
區塊（`Block`／`BlockHeader`，帶不透明的共識欄位）與
P2P 訊息的正典編碼／解碼（最小 LEB128 varint、裸 32 位元組
識別碼），會在配置記憶體前檢查大小限制、拒絕非正典形式，
並提供版本控制（`PROTOCOL_VERSION`）。沒有共識、網路或
儲存邏輯。格式參考：`/docs/en/technical/protocol.md`。

**scone-blockchain** — 區塊鏈邏輯與正典狀態：
交易識別（`TxId`）、有序 Merkle 樹、
`BlockHash`、決定性創世、區塊與交易驗證、
狀態套用規則（`DomainState`／`ChainState`，在記憶體中、
依 `DomainId` 直接存取）、分叉偵測與分支選擇
（`fork_choice`：最終性下限 + 最長鏈 + 最低雜湊
平手決勝）、檢查點最終性，以及共識抽象
（`trait Consensus`，預設為寬鬆實作）。
註冊 PoW（挖礦／驗證、依網路不同的難度）位於
`scone-core`。此 crate 中禁止：網路、DHT、儲存（redb）、
DNS 伺服器。參考：`/docs/en/technical/blockchain.md`。

**scone-storage** — 本機持久化（已交付）。具體後端為
`redb`（`NodeStore`），隱藏在一個極小的 trait 之後，
讓區塊鏈／DHT／DNS 永不直接耦合於 redb；可以
插入另一個後端（分散式 KV）。包含分頁
KV 狀態後端、區塊日誌、經驗證的快照（含雜湊的清單
+ 分頁）以及重啟還原／重播路徑。參考：
`/docs/en/technical/storage.md`。

**scone-network** — 節點的 p2p 中繼：libp2p／QUIC swarm、
交易與區塊轉發（系統性本機驗證）、有界
記憶體池、有界區塊同步（每請求 128 個區塊）、用於
`SignedDnsRecord` 的 Kademlia DHT、UDP+TCP DNS 伺服器
（GeoDNS 永遠開啟 — 內嵌 GeoIP 表、遞迴備援）、錨點的
檢查點／心跳迴圈（CLI 對中繼站直接說 p2p QUIC，
不再有本機 RPC）。可達性與探索：AutoNAT
v2（可達／NATed 判定）、circuit-relay v2 + DCUtR 給 NATed
節點與 UPnP（由路由器映射 QUIC 埠，採 opt-in）。透過 `.env` 檔案組態（見 `.env.example`）。
參考：`/docs/en/technical/relay.md`。

**scone-keystore** — Ed25519 簽署金鑰的加密儲存
（`.sconekey` 檔案：Argon2id + XChaCha20-Poly1305）。這個 crate
擁有檔案系統，讓 `scone-crypto` 得以保持純淨。參考：
`/docs/en/technical/keystore.md`。

**scone-topology** — 用於本機 p2p 拓撲最佳化的純（PURE）引擎：
零網路、零檔案系統、零 async。它既不認識 libp2p
也不認識 swarm：`scone-network` 餵給它觀測資料（ping RTT、
連線），並收到鄰里管理決策。

**scone-genesis** — 用於主網創世準備的一次性工具二進位檔
（kickstarter）：版本化 JSON 清單、配置
（allocations）、決定性區塊。該工具產出成品（artifacts），儲存庫
永不將其納入版本控制（拒絕寫入任何 Rust workspace 內部；散布
保留至主網上線，見 `blockchain.md`）。

**scone** — 二進位檔／CLI。只有在實際使用某 crate 時才
相依於它（目前：relay 用 scone-network、scone-blockchain、
scone-storage、scone-keystore、scone-protocol、scone-core、
scone-crypto、scone-topology）。

## 節點

一個 Scone 節點結合了：

1. 區塊鏈（共識驗證、所有權狀態）；
2. DHT 路由（`SignedDnsRecord` 的發布／擷取）；
3. 本機儲存（狀態、索引、快取）；
4. 前端的解析器／DNS 伺服器。

### 啟動新節點（狀態引導，P0.3/P0.4）

新節點不重播創世：它從一個經驗證的
已最終化狀態出發。

```text
finalized checkpoint (committee, signature quorum — validated by
        │              the chain layer before any use)
        ▼
verified snapshot (manifest + paginated pages; the importer
        │              RECOMPUTES the SMT root from the entries
        │              and compares it to the signed state_root —
        │              a falsified/incomplete snapshot is rejected
        │              before any write)
        ▼
anchor block H (requested from the network BY HASH)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height)
        │
        ▼
synced — at boot, replay of ONLY the suffix above H
```

兩種節點設定檔：

- **封存節點**（operator 旗標）：保存所有歷史區塊、
  將它們提供給網路，透過既有路徑啟動；
- **全狀態節點**：持有 H 的完整狀態與字尾，
  但不含 H 以下的歷史 — 它驗證未來的區塊並
  提供狀態，不提供歷史。

此路徑為 OPT-IN：匯入經驗證的快照是明確的
營運者操作（`import_verified_snapshot`），絕不自動 —
既有節點的預設行為不變。
規範性細節：`/docs/en/technical/storage.md`，
「VERIFIED state snapshot」與「State bootstrap」各節。

## DNS 解析流程

```text
DNS query (example.uip)
    │
    ▼
Node: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── authoritative DNS snapshot (RAM) ─► unknown/expired → NXDOMAIN
    │                                          (without touching the core)
    ├── sharded DNS cache (qname,qtype) ───► hit: shared entry,
    │                                          lazy verification
    ├── local cache (SignedDnsRecord) ─────► verification (signature, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ─────────────────► same verification
    │                                          (miss → 30 s negative cache)
    ▼
DNS answer (if and only if valid)
```

DNS 快照（每個活躍網域的存在與到期，
在每個已套用區塊之後發布）保證存在性
決策永遠不經過中繼站的執行迴圈：核心
只有在實際擷取記錄時才被觸碰。

## 可演化性

- 版本化的前像（`SCONE-DOMAIN-V1`、`SCONE-PUBKEY-V1`、
  `SCONE-OWNER-V1`、…）：改變衍生規則會建立新的
  命名空間，而非破壞既有的；
- `RecordData` 透過變體 + `Unknown { type_code, data }`
  擴充，以容納讀取時尚未知道的型別；
- `PROTOCOL_VERSION`（`scone-protocol`）用於線上格式破壞性變更；
- 鏈只儲存雜湊：大資料留在 DHT。

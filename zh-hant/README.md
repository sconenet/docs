# Scone 文件

> `docs/` 下的所有文件皆為**規範性（NORMATIVE）**：它必須能讓人在不閱讀
> Rust 程式碼的情況下完成獨立實作。任何文件與程式碼之間的分歧
> 都視為 bug（找出事實來源並修正之）。

分為兩部分：

- **`general/`** — 運作原理：協定的概念與規則，
  無需閱讀格式細節即可理解；
- **`technical/`** — 給開發者的詳細規格：精確的
  二進位格式、雜湊公式、界限、API，以及各元件的
  行為。

## `general/` — 運作原理

| 文件 | 內容 |
|---|---|
| [architecture.md](general/architecture.md) | 權責分離（區塊鏈）／可用性（DHT）／本機儲存；各 crate 的角色；DNS 解析流程；可演化性 |
| [naming.md](general/naming.md) | 命名規則：LDH 字母集、TLD、網域與子網域、正規化、`DomainId` 衍生與十六進位表示 |

## `technical/` — 開發者規格

| 文件 | 內容 |
|---|---|
| [protocol.md](technical/protocol.md) | 標準線上二進位格式：LEB128 varint、識別碼、DNS 記錄、區塊、P2P 訊息、限制、版本控制 |
| [transactions.md](technical/transactions.md) | 簽署交易格式 v1：欄位、線上順序（RegisterDomain 攜帶名稱、任意判別碼）、擁有者衍生、簽署酬載 `SCONE-TX-SIG-V1`、驗證規則 |
| [blockchain.md](technical/blockchain.md) | 區塊鏈層：`TxId`、Merkle 樹、`BlockHash`、創世區塊、正典狀態、區塊驗證、共識抽象 |
| [keystore.md](technical/keystore.md) | `.sconekey` 金鑰檔格式（Argon2id + XChaCha20-Poly1305）、`scone-keystore` API、`identity` 指令 |
| [storage.md](technical/storage.md) | 本機 redb 持久化：資料表架構、原子差量、記憶體策略、信任模型、DoS 界限 |
| [relay.md](technical/relay.md) | 網路中繼（`scone-network`）：libp2p 協定、節點探索（種子、目錄、mDNS、UPnP）、可達性（AutoNAT、circuit-relay、DCUtR）、記憶體池、同步、Kademlia DHT、本機拓撲、Prometheus 指標、DNS 快照、完整 CLI、`.env` 組態 |
| [dns.md](technical/dns.md) | UDP + TCP DNS 伺服器（M6/M7b）：權威 DNS 快照、16 路分片的熱路徑快取（共享條目、無配置查找、緊湊記錄、名稱壓縮）、RFC 1035 編解碼器、TCP RFC 7766（512 位元組 UDP／64 KiB TCP 預算、TC 位元）、DHT 負快取、遞迴備援、界限 |
| [cli.md](technical/cli.md) | `scone` 二進位檔：日誌與詳細程度（stdout/stderr、`-v` 等級、`RUST_LOG`）、透過 `.env` 的環境組態（優先順序 旗標 > `.env` > 預設） |
| [security-model.md](technical/security-model.md) | 安全模型：委員會選取、種子反研磨（分析：不需要 VRF）、equivocation 與罰沒 |
| [audit-methodologie.md](technical/audit-methodologie.md) | 安全稽核方法論：分歧、事實來源、修正階段 |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | 服務證明 RFC：營運者心跳、操作委派、在新鮮席位中選舉委員會 |

## 交叉參照慣例

內部文件與 Rust 程式碼註解以相對於儲存庫根目錄的絕對路徑
引用這些頁面
（例如 `/docs/en/technical/protocol.md`）。

# Keystore — `.sconekey` 金鑰檔格式規格

> 規範性文件。本文件與
> `crates/scone-keystore/src/lib.rs` 之間的任何分歧都是
> bug（找出事實來源、修正之）。實作參考：crate
> `scone-keystore`（v2）。

## 目的

把 Ed25519 私鑰（*seed*、32 位元組）以通關密語
加密儲存於磁碟。金鑰**絕不**以明文觸及
磁碟；所有機密（通關密語、衍生材料）在使用後
於記憶體中歸零。

## 總覽

- **KDF**：Argon2id（v0x13）、預設參數 `m_cost`
  = 64 MiB、`t_cost` = 3、`p_cost` = 1（`argon2`
  crate 的預設值）。
- **加密**：XChaCha20-Poly1305（AEAD）、由
  KDF 衍生的 32 位元組金鑰、每檔 24 位元組
  隨機 nonce。
- **加密內容**：恰好是 Ed25519 seed（32
  位元組）；Poly1305 標籤（16 位元組）作為
  密文的後綴儲存（AEAD `encrypt` API 的
  標準輸出）。

## 位元組封裝（`bytes` 模組、web）

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}`
在不觸碰檔案系統的情況下產生並消費相同
格式：web 客戶端在瀏覽器中產生身分
（`scone-crypto` + bip39）並封裝精確的金鑰檔
位元組（可下載、CLI 可原樣開啟 — 套件中的
互通性測試）。原生的
路徑（`create`／`create_with_phrase`）委派給
相同的封裝：一種格式編碼、三個目的地。

## 二進位格式（v2）— 52 位元組標頭 + 公開中繼資料 + 密文

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII — unchanged since v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      32          public_key: Ed25519 public key (32 bytes, IN THE CLEAR)
84      8           created_at: u64 big-endian (Unix, seconds)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (readable without the passphrase, may be empty)
—       48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

`public_key`／`created_at`／`label` 中繼資料是
公開的：**無需通關密語**即可讀取
（`public_metadata_from_bytes`）。公鑰本質
公開；標籤只是顯示名稱 — 隱藏它一無所獲
卻犧牲無密碼列表。總長取決於標籤：空標籤
時 141 位元組（KEYFILE_LEN + 32 + 8 + 1），
加上標籤長度。

**v1（已被取代、恰好 100 位元組）**：magic ‖
version=1 ‖ m ‖ t ‖ salt ‖ nonce ‖ 密文 —
公鑰未被儲存（上鎖的檔案是匿名的）。v1
檔案保持**可讀**（中繼資料 `None`、
`identity show` 需要通關密語；經
`scone identity restore` 重新封裝可將其遷移至
v2）。

### 設計選擇

- **KDF 參數存在檔案裡**：`m_cost_mib` 與
  `t_cost` 自標頭讀取（讀取時不寫死），
  使未來版本得以提高它們而不破壞既有
  檔案。
- **讀取時的安全界限**：宣告
  `m_cost_mib` > 64 或 `t_cost` = 0 或 > 10 的
  檔案在任何配置*之前*被拒
  （`Error::InvalidParams`）— 惡意
  檔案無法強迫節點配置數 GB 或無限
  迴圈。寫入時的預設為 `m_cost_mib` = 64、
  `t_cost` = 3。
- **明確的 magic + 版本**：不是金鑰檔的
  檔案（`Error::InvalidFile`）或不支援版本
  （`Error::UnsupportedVersion`）的檔案被
  乾淨地拒絕、不 panic。
- **錯誤的通關密語**：Poly1305 認證失敗
  不會指出*為何*失敗；錯誤的通關密語與
  損毀的密文因此無法區分，並以同一個
  明確錯誤 `Error::WrongPassphrase` 浮現。

## 加密金鑰衍生

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

然後：

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

v1 中無關聯資料（AAD）。每檔隨機的 24
位元組 nonce 使 nonce 重用風險可以忽略
（金鑰檔只寫一次）。

## API（`scone-keystore`）

| 函數 | 角色 |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | 產生金鑰、寫入加密金鑰檔（建立父目錄）；**拒絕覆寫**既有檔案（`Error::KeyfileExists`）；在記憶體中回傳金鑰 |
| `create_overwriting(path, passphrase) -> GeneratedKey` | 相同，但明確替換既有金鑰檔 |
| `open(path, passphrase) -> SigningKey` | 解密並回傳簽署金鑰 |
| `list(dir) -> Vec<KeyEntry>` | 列出目錄中的 `*.sconekey` 檔案（依名稱排序）；目錄不存在 = 空清單 |

`GeneratedKey.signing_key` 在 drop 時由
`ed25519-dalek` 歸零（預設 `zeroize`
特性）。衍生的 AEAD 金鑰、seed 與
解密後的明文都包在 `zeroize::Zeroizing`
中：Drop 涵蓋所有退出路徑，包括 `?`
傳播的錯誤。

## 檔案權限

在 Unix 上，金鑰檔以權限 `0600`（僅擁有者
可讀寫）建立，經由 `OpenOptions` +
`set_permissions` — 與行程 umask 無關。存在性
與建立是原子的（`create_new`、`O_EXCL`）：
兩個並行的產生無法靜靜互相覆寫。

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- 預設目錄：`$HOME/.scone/keys`。
- 身分名稱被嚴格驗證：空、`.`、`..`、`/`、
  `\\\\`、NUL、前導點與 `.sconekey` 後綴都被
  拒絕（名稱會成為金鑰目錄內的檔名 —
  沒有任何路徑元件能逃出 keystore）。
- 不帶 `--passphrase-env` 時，通關密語在
  終端機上**以遮罩輸入**要求（`rpassword`：
  不回顯；`generate` 兩次並確認）。
- `identity generate` **恰好顯示一次 BIP39
  復原短語（24 個字詞）** — 全新 256 位元
  熵，僅憑字詞即可重現同一把金鑰
  （`scone identity restore`）；它們絕不
  儲存於任何地方。
- `identity generate` **拒絕覆寫**既有的
  金鑰檔；`--force` 明確替換（新 salt/nonce、
  新金鑰、新短語）。
- `identity list` 對 v2 檔案**無需通關密語**
  即顯示公鑰與標籤（公開中繼資料）。
- `identity show` 讀取 v2 金鑰檔的公開
  中繼資料，並**無需通關密語**顯示公鑰、
  小寫十六進位的 `OwnerId`（64 字元、與
  `DomainId` 相同慣例）、標籤與建立
  日期；v1 檔案（無中繼資料）仍會要求
  密語。

## 此格式刻意不做的事

- 沒有每檔多金鑰（每檔一個 seed）。
- 沒有通關密語變更（若將來有：以新
  salt/nonce 重寫檔案）。


## 預設身分（cli-simplify-v1）

需要簽署身分的指令（`domain register`、
`domain set`、`domain update`、`domain renew`）
在 keystore 恰有一個身分時無需 `--identity`
即使用該**單一**身分。有多個身分時指令
失敗並列出它們（`scone identity list`）；
請明確傳入 `--identity <名稱>`。

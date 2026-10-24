# CLI（`scone`）— 日誌與詳細程度

> `scone` 二進位檔日誌行為的規範。任何文件／程式碼
> 分歧 = bug（修正事實來源）。

## 原則

- **stdout 保留給指令輸出**（`scone status`、
  `identity list`、`status`、…）。腳本可以重定向
  stdout 而永遠不會在其中發現日誌 — 管線保持
  可靠。接受的例外：`scone relay` 在監聽器
  綁定後立即於 stdout 發出資料行
  `p2p: <multiaddr>`、`dns: <addr>` 與
  `metrics: <addr>`
  （供 `--bootstrap`／組態使用的資料，不是日誌；
  各恰好發出一次、立即沖刷）。
- **所有日誌走向 stderr**（緊湊的 `tracing` 格式），
  包括中繼站的 — 遷移前中繼站就已
  記錄到 stderr。
- CLI 面向使用者的錯誤訊息（`scone: <error>`）留在
  stderr：這是 UX，不是日誌。

## 等級與 `--verbose`／`-v`

| 呼叫 | 有效等級 |
|---|---|
| `scone <cmd>`（不帶 `-v`） | `warn` |
| `scone relay`（不帶 `-v`） | `info` |
| `scone -v <cmd>`／`scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …`（及更多） | `trace` |

`-v`／`--verbose` 是**全域且可重複**的旗標：
子指令前後皆可（`scone -vv identity list` 與
`scone identity list -vv` 等價）。

**為什麼 `scone relay` 預設停在 INFO**：中繼站是
守護行程 — 它必須記錄自己的活動（RPC/p2p 監聽、
接受與產出的區塊、連線的 peer、拒絕）才可
營運，而不需要旗標。相反地，一次性指令
保持安靜（WARN），以免污染終端機或腳本。

## `RUST_LOG`（覆寫）

若設定了 `RUST_LOG` 環境變數，它**完全取代**
由 `-v` 導出的等級（`env-filter` 語法）：

```bash
RUST_LOG=debug scone identity list        # debug without -v
RUST_LOG=error scone -vv relay            # error wins over -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## 環境組態（`.env`）

解析參數之前，`scone` 從當前目錄載入 `.env` 檔案
（`main.rs` 中的 `dotenvy::dotenv()`）。檔案
不存在 = 靜靜跳過，絕不是錯誤。載入的值成為
宣告了環境變數（`cli.rs` 中的 `env = "…"`) 的中繼站
選項的基線。

優先順序（一個選項、三個來源）：

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- 命令列傳入的旗標永遠獲勝；
- 否則為變數（無論從 `.env` 載入或設於
  真實行程環境 — 同一變數、同一效果）；
- 否則為 clap 的內建預設。

可重複的變數（`SCONE_BOOTSTRAP`、`SCONE_DNS_UPSTREAM`）
累加：`.env` 中每個值一行（dotenvy 附加）。

`SCONE_*` 變數的完整表（網路、P2P 監聽、
bootstrap、創世、UPnP、DNS 表面、指標、拓撲
設定檔、錨點）及各自的語義位於
[`relay.md`](relay.md#environment-configuration-env)；
帶註解的範本是儲存庫根目錄的 `.env.example`
（複製為 `.env`，絕不提交 — `.env` 已被
gitignore）。

## 等級語義（程式碼慣例）

- `error!`：失敗（今日無呼叫點 — 網路失敗
  被吞為 `warn!`，只有致命的本機失敗以
  來自 `run()` 的型別化錯誤浮現）；
- `warn!`：被擋下的敵對或惡意網路條目、
  自記憶體池逐出的過期交易、生產期間
  捨棄的衝突交易、出站／入站失敗、DHT 金鑰
  不符；
- `info!`：區塊接受／產出、RPC/p2p 監聽、peer
  連線、中繼站啟動（狀態檢查點）；
- `debug!`：內部細節（交易接受、peer 斷線、RPC
  往返、keystore 列表、`SCONE_KAD_DEBUG` 下的
  kad 事件）；
- `trace!`：保留給酬載（今日無呼叫點）。

函式庫 crate（`scone-network`、…）發出 `tracing`
巨集但**不初始化任何 subscriber**：初始化只
存在於 `scone` 二進位檔（子指令分派之前 —
中繼站從開機就需要它的
日誌）。


## 簡單用法（cli-simplify-v1）

- **不需要 `--peer`**：指令預設與本機中繼站
  交談；`SCONE_PEER` 環境變數或 `--peer` 可覆寫。
- 當 keystore 只持有一個身分時**不需要
  `--identity`**（靜靜使用；多個 → 錯誤把它們
  列出）。
- 已移除（stub 建議替代）：`show`、`submit`、
  `lookup`、`owned`。
- 新增：`scone domain renew <name>`、`scone domains`、
  `scone domain get|set`（單筆記錄、無檔案）。

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## TLD 生命週期（cli-unified-lifecycle）

`scone tld register <tld>` 是一條指令的 TLD 入口，
與 `scone domain register` 對稱 — 先讀取
鏈上狀態，再執行正確的動作：

- **空閒 TLD** → 註冊，預設**天生開放**（無旗標：
  自助命名空間、1/3 的 PoW）。`--private` 以
  完整 PoW 價格選擇自有命名空間，並帶明確
  確認（經濟決策）。`--yes` 跳過確認
  （腳本）。
- **你的、90 天內到期** → 提議續期
  （PoW 綁定當前到期日 — 防重用），確認後
  執行。
- **你的、離到期尚遠** → 無事可做（`tld renew
  --force` 仍可強制續期）。
- **他人的** → 帶擁有者的乾淨型別化錯誤。

```bash
scone tld register mytld            # open birth (default)
scone tld register mytld --private  # owned namespace (confirm)
scone tld renew mytld               # explicit renewal
```

同樣地，對你擁有且 90 天內到期的名稱執行
`scone domain register <name>` 會提議續期而非
失敗。

## 進階指令

上述預設涵蓋日常使用。供腳本與 web/API
互通的完整表面仍可用：

每個操作只有一條規範路徑；以前的根層級別名（`show`、`lookup`、`owned`、`renew`）已被移除（remove-cli-aliases），從未真實存在的 `domain info` 提法也一併移除。

離線十六進位表面（`tx build|sign|verify`、
`submit tx` 子指令）已移除
（remove-tx-surface-v2）：它所服務的 M8b
操作如今是一級指令 —
`scone tld open <tld>`（單向自助開放、為你
挖出開放 PoW）、`scone tld transfer <tld> <owner>`、
`scone tld revoke <tld>`、`scone domain assign <name> <owner>`
（TLD 擁有者、私有 TLD）與 `scone domain transfer <name>
<owner>`。每一個：狀態讀取、簽章、提交、txid。DNS
記錄解析移至 `scone domain resolve <name>`（原
`record get`）；十六進位發布通道（`record put`）已
移除 — 編輯一律透過 `domain set`／
`domain update`。
- `scone domain mine <name>` — 只挖註冊證明
  （無身分、無 keystore、不提交）：將
  `scone-domain-pow-v1` JSON `{type, name, network_id,
  epoch, proof}` 印到 stdout。委託挖礦：另一台
  機器為你挖礦，你的金鑰絕不離開你的機器。
  可以管線或重新導向：`scone domain mine n.tld |
  scone domain register n.tld --proof -`（或
  `--proof FILE`）。證明綁定名稱與網路；新鮮度
  由節點在套用時強制執行（epoch 有效窗口）。
  使用 `--mine-for <pubkey>` 時，改為為該受益者
  挖一個 POW-TO-OWNER 證明，並印出
  `scone-domain-pow-to-v1` JSON `{type, name,
  network_id, epoch, pow_to, proof}` — 只能由該
  確切金鑰的 `register --to` / `renew --to` 消費
  （挑戰內嵌該金鑰；提交時不分大小寫檢查）。
- Pow-to-owner 提交（無需 keystore — `--to` 與
  `--identity` 互斥）：`scone domain register <name>
  --to <pubkey>` / `scone domain renew <name> --to
  <pubkey>` 與 `scone tld register <tld> --to
  <pubkey>` / `scone tld renew <tld> --to <pubkey>`
  提交挑戰綁定受益者金鑰的證明；鏈上
  `owner` 重新計算為 `hash(pubkey_dest)`
  （續期絕不換手）。不帶 `--proof` 時，指令自己
  挖證明；帶 `--proof -`/FILE 時消費
  `scone-domain-pow-to-v1` 酬載，並拒絕為其他
  受益者挖出的證明。管線風格：`scone domain
  mine n.tld --mine-for <pubkey> | scone domain
  register n.tld --to <pubkey> --proof -`。
- `scone domain export <name> [--out FILE]` — 以
  精確的 `update --file` 格式寫出
  經鏈驗證的記錄集
 （保證往返：匯出 → 編輯 → update 重新提交
  相同集合）。不帶 `--out`：stdout。
  `domain resolve` 使用相同的可讀格式，因此
  `resolve > zone.txt` 可直接
  匯入。無文字語法的種類（GEOROUTE、未知）
  以註解警告匯出 — 絕不靜靜丟棄。

`domain get` 也印出 `domain_id` 與 `valid_until`。

## PoW 基準測試（`scone bench-pow`）

測量機器的 Argon2id 雜湊率（與共識驗證使用的
是同一個雜湊），並把網路的難度換算成具體的
等待時間：

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

輸出：總雜湊率與每個 worker 的雜湊率，以及每項
操作的預期平均時間 — 轉讓／指派（固定費率）、
私有 TLD 開放，以及 TLD／網域註冊下限（長度
曲線）。所選網路決定 Argon2id 參數（mainnet：
64 MiB、t=2 — 真實成本；testnet/devnet：輕量
設定檔，毫秒級證明）。每個 worker 配置自己的
64 MiB Argon2id 通道 — 請按可用記憶體調整
`--cores`。

## 建置：開發 vs 啟用主網（mainnet-compile-gate）

預設建置**不含主網**：`scone relay --network
mainnet`（以及帶 `SCONE_NETWORK=mainnet` 的
註冊服務）立即失敗並附重建提示。官方
發行的二進位檔以如下方式建置：

```bash
cargo build --release --features mainnet
```

`scone --version` 在此類建置上顯示 `+mainnet`，
且中繼站啟動日誌顯示 `build: mainnet-enabled`。

預設的 `--network` 也跟隨建置：啟用主網的
建置為 `mainnet`（官方二進位檔讓使用者落在
生產網路），否則 `testnet`。明確的
`--network` 旗標或 `SCONE_NETWORK` 環境變數
永遠獲勝（clap 順序：旗標 > 環境 > 預設）。
這是營運護欄（意外執行、開發／CI 二進位
檔）— 不是密碼學邊界；共識規則仍是對抗
敵對節點的防線。

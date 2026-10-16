# UDP + TCP DNS 伺服器（M6 + M7b）— `scone dns`

> Scone DNS 表面的規範規格。
> 任何文件／程式碼分歧 = bug（修正事實來源）。

## 總覽

**DNS 伺服器**是中繼站的可選表面：它以
**UDP 與 TCP**（RFC 7766、同埠）回答 Scone 名稱的
DNS 查詢，且**僅使用經鏈驗證的資料**，並為
其他一切提供**可選的遞迴備援**（`addr:port`）。
它不擁有鏈、儲存庫或 DHT：存在／到期
決策經過**權威 DNS 快照**（每個已套用區塊
之後發布的 RAM 視圖 — 見下文），記錄
擷取經過中繼站（行程），透過內部
`RelayCommand`／`ResolveLocal` 通道（行程內、一次
往返）— 與 CLI 相同的已驗證路徑。

```text
              UDP+TCP :5353 (default 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Sharded cache, 16 ways (qname)  │
                    │  └────────────────────────────────┘
                    │  Scone-shaped name?
                    │  ├─ YES → authoritative DNS snapshot:
                    │  │        unknown/expired → NXDOMAIN (RAM,
                    │  │        zero round trips into the core)
                    │  │        → apex search by suffix
                    │  │        → internal `resolve_local` (relay)
                    │  │           chain + verified record (hash,
                    │  │           owner, sequence) → answers
                    │  │        apex without a published record → NODATA
                    │  └─ NO → upstream fallback (if configured)
                    │           otherwise REFUSED
                    │
                    │  response budget:
                    │    UDP → 512 bytes (no EDNS); overflow =
                    │          cut at the last fitting record + TC=1 →
                    │          the client retries over TCP
                    │    TCP → 64 KiB (2-byte length prefix,
                    │          RFC 1035 §4.2.2): full set
```

## 啟動

```bash
scone relay --dns 127.0.0.1:5353            # DNS surface only
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + recursive fallback
```

中繼站在 stdout 宣告有效位址
（機器可讀契約，如 `p2p:`）：socket 綁定後
立即出現一行 `dns: <addr>`。

驗證用戶端：`dig <name> --dns 127.0.0.1:5353
--qtype A`（A、AAAA、TXT、MX、NS、CNAME、ANY）。

## 解析路徑（權威 = 鏈，恆定）

0. **權威 DNS 快照**：由中繼站核心在**每個
   已套用區塊之後**（推送而非輪詢）與開機時
   發布的不可變視圖
   （`Arc<HashMap<DomainId, {valid_until, has_record}>>`、
   `tokio::sync::watch` 饋送）。
   存在／到期決策 — NXDOMAIN、過期網域 — 透過
   此快照自 RAM 服務：一次 `Arc` 複製 + 一次
   映射查找，**絕不進入中繼站的單一擁有者
   執行迴圈**。完整路徑
   （`RelayCommand`／`ResolveLocal`）仍保留給實際的
   記錄擷取（DHT 拉取），因快取而罕見。快照
   絕不攜帶記錄資料：僅存在與到期。
   - **O(被觸及) 的寫時複製差量**：映射是
     `Arc<HashMap>`；尖端延伸發布該區塊的差量
     （區塊自身的交易點名每個它們能改變的
     網域 — Register/Assign/Update/Renew/TransferDomain — 加上
     推送時評估的決定性到期 GC 移除），透過
     `Arc::make_mut`：只有在讀者仍持有
     前一視圖時才複製映射。開機與 reorg 重建
     完整快照（一次分頁後端遍歷）。
   - 新鮮度：區塊套用期間讀者可能看到
     落後數百 ms 的快照 — 與狀態 RPC 相同的
     陳舊度；DNS 快取 TTL 不受影響（回答
     依其來源保持權威）。
1. **分片快取**：[`ShardedCache`] — 16 個獨立的
   `Cache` 分片，以 `hash(qname)` 為鍵（qtype 共用
   分片：每名稱局部性）。熱門名稱只與
   1/16 的流量競爭，而非在數千 qps 下排隊於
   單一 mutex 之後。全域界限（預設 100,000 條目／
   64 MiB，`--dns-cache-limit`／`--dns-cache-max-mem`）
   逐分片分攤（已載明的最壞情況：1 + N/分片數
   因子）。每個條目的**真實** rdata 大小都被
   計入（每條目約 128 位元組開銷），FIFO 逐出
   在兩個界限之一首次觸及時觸發 — 記憶體
   上限即使面對完整尺寸條目
   （`MAX_DHT_RECORD_LEN`）也成立。TTL 60 s
   （權威）／鉗制為 600 s（備援）。負
   快取（NXDOMAIN/NODATA）遵循相同規則；每個
   條目攜帶其來源（權威或備援），因此命中時
   AA 位元保持誠實。**絕不快取 SERVFAIL**
   （暫時的本機故障）**也不 REFUSED**（組態
   狀態）：兩者每次查詢都重新計算。容量滿時
   FIFO 逐出。**熱路徑（受 Cloudflare/1.1.1.1
   啟發）**：
   - 共享的 `Arc<CacheEntry>` 條目：命中是
     引用計數遞增，絕非 rdata 的深層複製
     （`put` 取代 Arc — 沒有共享突變；TTL 仍在
     分片鎖下檢查）；
   - **無配置查找**：兩層映射
     （`Box<str>` qname → qtype → 條目），查詢
     借用的 `&str` 直接探測（`Borrow<str>`）—
     舊的 `(String, u16)` 元組金鑰在**每次**
     查找都配置新的 `String`；
   - **緊湊連續記錄**：每條目一個 `Box<[u8]>`
     緩衝區（每記錄 `u16 rdlen ‖ u16 rtype ‖
     u32 ttl ‖ rdata`）而非每記錄一個 `Vec` —
     記憶體預算精確追蹤已配置的量；
   - **輸出時的 RFC 1035 名稱壓縮**：權威
     回答的擁有者是指向被回顯問題的 2 位元組
     壓縮指標（`0xC00C`）— 問題絕不
     每記錄重新完整編碼。
2. **Scone 形狀？** 嚴格前置過濾：標籤
   `[a-z0-9-_]`（1..63）、≥ 2 個標籤、TLD 1..63
   位元組 `[a-z0-9-]`（見 `TldName::MAX_LEN`）、
   總長 ≤ 253 — **且 TLD 在 ICANN 根之外**
   （`is_icann_tld`、凍結的 IANA 清單 + RFC 2606：
   `com`、`fr`、`org`… 永不屬於 Scone）。
   不可能是 Scone 名稱的名稱（如 `www.foo_bar`、
   TLD 含底線）走向備援；可能的 Scone 名稱
   （`absent.uip`）是鏈的事：未註冊 → 權威
   NXDOMAIN，即使沒有上游。ICANN TLD
   （`example.com`）在已組態時轉送上游，否則
   REFUSED — 絕不由 Scone 權威解析。
3. **Apex 搜尋**：後綴由長至短（最少
   2 個標籤）。中繼站已知其鏈上狀態的
   第一個後綴 = apex；記錄集服務整個
   qname（含子名 — 隱式萬用字元行為，
   開放決策）。
4. **`resolve_local`（內部通道）**：單次往返，絕非
   DHT 等待者。鏈上狀態（`registered`、擁有者、
   序號）+ 本機 DHT 快取記錄**重新驗證**
   （擁有者、序號、record_hash == 鏈上承諾）。
   不合格的記錄絕不作答 — 可用性 ≠ 權威。
   - 未註冊 → NXDOMAIN；
   - 已註冊但無有效記錄 → NODATA（NOERROR、
     0 個回答）；
   - 已驗證記錄 → 回答按 qtype 過濾。

## 編解碼器（RFC 1035、嚴格子集）

- 查詢：12 位元組標頭、**一個**問題、QCLASS =
  IN、**無壓縮**（指標 = 拒絕）。標籤轉小寫並
  驗證。QR=1、QDCOUNT≠1、無效標籤 → **沉默**
  （垃圾不花成本）。
- 回應：QR|RD（回顯）|RA、**AA 僅在權威回答**
  （鏈、新鮮或快取 — 絕不在備援或錯誤
  RCODE 上）、問題逐字複製、權威回答帶
  **RFC 1035 壓縮的擁有者**（指向被回顯問題的
  2 位元組指標 `0xC00C`）；備援上游回覆原樣
  服務（絕不重新編碼）。
- 服務的型別：A、AAAA、CNAME、NS、MX、TXT、
  **PTR、SRV、CAA**（+ Unknown 逐字重新編碼；
  共 9 種原生型別，rdata 依 RFC 編碼 —
  1035/2782/8659；嚴格 CAA：僅臨界位元 0x80、
  小寫標籤）。A/AAAA 查詢包含集合的
  CNAME（別名追逐）。ANY 查詢（255）：整個
  集合由編解碼器過濾。
- 限制（M7b）：**回應預算取決於傳輸** —
  UDP 上 512 位元組（經典 RFC 1035 §2.3.4
  承載、無 EDNS）、TCP 上 65,535。超出預算的
  集合在最後一個放得下的記錄處切斷，帶
  **TC=1**（誠實截斷、RFC 1035 §4.2.1）：客戶端
  經 TCP 重試（RFC 7766），在那裡服務完整
  集合。大於 UDP 預算的上游回覆被
  重新編碼截斷（TC=1）而非原樣交付；
  在 TCP 上逐位元組通過。截斷也適用於
  快取命中（預算在回應時施加，而非在
  填充快取時）。
- RCODE：0 NOERROR（有無回答皆可）、
  2 SERVFAIL（本機故障）、3 NXDOMAIN（權威）、
  5 REFUSED（無上游的非 Scone）。

## 遞迴備援

- 上游 `addr:port`（UDP）、按序嘗試、每個
  上游 2 秒逾時、socket **connect** 到上游
  （核心過濾：來自任何其他來源的封包絕不
  被接受）。上游回應只有在**與查詢相符**時
  才被接受：相同 TXID、QR=1、問題
  （QNAME+QTYPE+QCLASS）逐位元組複製。
  不合格的回應 → SERVFAIL，且
  不快取任何東西。
- 已驗證的上游回應原樣通過；其
  回答以來源 `fallback` 快取（絕不設
  AA，即使在快取命中上），TTL 鉗制為
  600 s。上游錯誤 RCODE（NOERROR/NXDOMAIN
  以外）不快取。
- **預設私有**：無上游 = Scone 形狀之外的
  名稱 REFUSED。

## DHT 負快取（反放大）

**未獲回應的 DHT 拉取**（記錄預期在鏈上 —
`record_hash` 已承諾 — 但不在本機快取且不在
被查詢的 peer 上）將該名稱停放
`DHT_NEGATIVE_TTL_SECS`（30 秒）於有界負
快取中（FIFO、上限 `DHT_NEGATIVE_CAP` =
4,096 個名稱）：TTL 內對同名稱的重複解析被
**抑制**（計數、絕不重新拉取）。掃描器 —
或擁有者從不發布的熱門名稱 — 不再能把
一次 DNS 查詢放大為每次查詢對每個網格
peer 的 `GetRecord`。負
條目在 TTL 到期、記錄到達（成功拉取寫入
真實快取）或被 FIFO 上限推出時死亡。

揭露的計數器（`scone_dht_negative_hits_total`、
`_expired_total`、`_capped_total` — 見
`relay.md` 的「Metrics」）。

## TCP（M7b、RFC 7766）

與 UDP 同埠（在 `--dns` 位址上的第二次 TCP
綁定）。傳輸遵循 RFC 1035 §4.2.2：每個訊息
以 big-endian 的 **2 位元組**長度為前綴。

- **查詢**：前綴長度 ≥ 1 且 ≤ 512 位元組
  （DNS 查詢放得進經典承載 — RFC 7766 §6；
  不服務 EDNS）。越界 → 立即關閉連線。
- **回應**：65,535 位元組預算（2 位元組前綴的
  極大值）— 對 UDP 太大的集合在此**完整**
  服務，TC=0。
- **管線化**：每連線多個查詢按序
  串列服務（一次寫入 = 一個帶前綴的回應）。
- **讀取逾時**：每次讀取（前綴或本體）5 秒 —
  沉默的客戶端被斷線、其槽位釋放。
- **並發連線**：最多 64（號誌）；第 65 個
  立即拒絕（關閉連線）、絕不排隊 — 任務或
  記憶體無無界成長。
- 讀／寫／逾時錯誤：debug 日誌 + 乾淨關閉。
  **絕不因網路輸入 panic。**
- 垃圾（無法理解的查詢）得到沉默但不
  關閉連線：下一個（有效的）查詢被服務。

```text
client                     server
  │ [len u16][query]         │
  │ ─────────────────────►   │ cache → authoritative → fallback
  │ ◄─────────────────────   │ [len u16][response ≤ 64 KiB]
  │ (UDP had answered TC=1 → the client retries here)
```

## 界限與護欄

| 護欄 | 值 |
|---|---|
| UDP 資料包（查詢／回應） | 4096 位元組（超過即拒） |
| UDP 回應預算（超過即 TC=1） | 512 位元組（無 EDNS） |
| TCP 查詢上限 | 512 位元組（超過即關閉） |
| TCP 回應上限 | 65,535 位元組 |
| 並發 TCP 連線 | 64（號誌、立即拒絕） |
| TCP 讀取逾時 | 5 秒 |
| 快取 | 全域界限預設 100,000 條目／64 MiB，分攤於 16 個分片（`--dns-cache-limit`／`--dns-cache-max-mem`） |
| 權威 TTL | **每記錄**（`record.ttl`、界限 1..=2,592,000 s；歷史常數 60 s 僅保留給服務回應） |
| 備援 TTL（鉗制） | 600 秒 |
| 並行 UDP 查詢 | 4,096（`MAX_INFLIGHT_UDP`、號誌；超過：丟棄、客戶端重試）— 可調 `--dns-max-inflight`（最小 1） |
| DHT 負快取 | 30 秒（`DHT_NEGATIVE_TTL_SECS`）、FIFO 上限 4,096 個名稱 |
| 每個上游的逾時 | 2 秒 |

遵守的專案規則：絕不因網路資料 panic；
處處嚴格解碼；伺服器不觸碰鏈、儲存庫
或 DHT（中繼站仍是唯一的驗證權威）。

## 測試

- 單元（`scone-network::dns`）：嚴格編解碼器
 （QR、壓縮、標籤拒絕）、權威回答、NODATA、
  NXDOMAIN、無上游 REFUSED、對 UDP mock 的
  備援、有界快取、TXT 分塊、上游解析；M6
  修正：AA 位元誠實（權威 vs 備援 vs 錯誤、
  含快取命中）、超大集合的 TC=1 截斷、上游
  回應拒絕（TXID 或問題不符）、不快取
  SERVFAIL/REFUSED；分片快取：分片散布、
  拆分的全域界限、4 個並發工作者於熱門
  名稱；熱路徑：無配置查找、緊湊條目、
  擁有者壓縮。
- M7b：依傳輸的預算（UDP 512 位元組 → TC=1、
  TCP → 完整集合）、快取命中的截斷、真實
  `run_tcp`（> 512 的集合完整服務、2 個查詢的
  管線化、超大小查詢 → 關閉、沉默客戶端 →
  5 秒逾時、同一連線上垃圾後接有效查詢）、
  上游回覆 > 512 在 UDP 上重新編碼 TC=1／
  在 TCP 上逐位元組。
- 真實整合（`tests/dns_server.rs`）：完整中繼站
  （含生產 devnet）+ 註冊 + UpdateDomain +
  PutRecord，然後真實 UDP 查詢：已驗證的 A
 （AA 位元）、TXT、NODATA AAAA、NXDOMAIN、
  經 mock 備援、再查詢、經 apex 的子名。
- 二進位 E2E（`crates/scone/tests/dns_cli.rs`）：
  真實的 `scone` 二進位檔 — `relay --dns`、
  `identity generate`、`domain register`、
  `domain update --file`，然後（dnsutils 的）
  `dig`（已驗證的 A、TXT、NXDOMAIN、REFUSED）
  於真實 UDP 埠。

## 開放決策

- 無 EDNS（UDP 預算維持 512 位元組；TC=1
  截斷 + TCP 涵蓋 devnet 需求）。
- 記錄集不攜帶每記錄名稱：整個集合服務
  apex 的每個子名（隱式萬用字元）。每記錄
  定址需要擴充 `DnsRecord`（協定）。
- 備援轉發原始查詢：惡意上游可以回答
  任何東西（未驗證的回應 — 這是標準
  遞迴模型；Scone 名稱絕不經過它）。

## GeoDNS（地理路由）

> 規範規格。任何文件／程式碼分歧 = bug。

### 內嵌 GeoIP 表（自動）

每個二進位檔內嵌一張編譯進中繼站的
**IP → 國家映射表**：**無需組態**、地理路由
到處自動啟用。

- **資料來源**：**RIR 委派檔**（RIPE NCC、
  ARIN、APNIC、LACNIC、AFRINIC），可自由下載
  與再散布：
  `https://ftp.ripe.net/pub/stats/<rir>/delegated-<rir>-extended-latest`
  （每個被委派 IPv4/IPv6 區塊的註冊國家）。
  產生日期在表頭
  （`assets/embedded-geo.table`）。
- **重新產生**：`scripts/gen-geo-table.py`
 （下載 5 個檔案、轉換、正規化前綴 — 約
  176,000 條規則）。
- **粒度**：國家（數值 ISO 3166-1）+ 大陸
  （UN M49）衍生。無 ASN 也無區域（委派檔
  中不存在）。
- **效能**：對排序區間的**二分搜尋**查找
  （首次使用時每 v4/v6 族各建一次索引）—
  實測熱路徑約 95 ns/查找、全表隨機存取約
  50 µs（基準：`cargo test -p scone-network
  --release --test geo_bench -- --ignored`）。
- **中繼站自我定位**：路由跟隨**每次查詢的
  來源 IP** — LAN 中繼站自動把其本機客戶端
  路由到正確區域。
- **優先順序**：明確的營運者表
  （`--geo-ip-table`／geo-config）**覆蓋**內嵌
  表。

### 總覽

網域可以在其簽署記錄集中發布**地理路由**
規則：`GEOROUTE` 記錄（Scone 私有型別
`32,768`、IANA 範圍之外）。解析器以
**規則所選端點的位址**（離客戶端位置最近
的範疇、經表由其來源 IP 衍生）回答這些
網域的 A/AAAA 問題，而非靜態 A/AAAA
記錄。其他一切（無 GEOROUTE 的網域、其他
qtype）遵循經典 DNS 路徑，**逐位元組**相同。

### GeoDNS 指標

Prometheus 端點揭露地理回答分佈：

- `scone_dns_geo_answers_total` — 已服務的
  GEOROUTE 回答總數；
- `scone_dns_geo_by_continent_total{continent="<M49>"}` —
  依大陸；
- `scone_dns_geo_by_country_total{country="<ISO>"}` —
  依國家（受標準本身限制：≤ 9 個大陸、
  ≤ 250 個國家）。

### GEOROUTE 記錄（線上格式）

一個 GEOROUTE = 一個**範疇** + ≤ 32 個
**端點**（解碼時正典排序：優先權升序、
權重降序、目標升序；重複 = 拒絕）：

```text
GeoScope   = 0x00 (world)
           | 0x01 continent.v (UN M49 code: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (ISO 3166-1 numeric code: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (OPAQUE ISO 3166-2 subdivision, u32:
                                publisher-defined number, compared
                                as is — no local table)
           | 0x04 asn.v       (AS number, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (normalized 16-byte IP prefix: IPv4 = address in the
              first 4 bytes + ZERO tail, len ≤ 32;
              IPv6 = 16 free bytes, len ≤ 128; host bits ALWAYS
              zero — a non-normalized prefix is rejected)
GeoTarget  = 0x00 octets[4]  (binary IPv4 address)
           | 0x01 octets[16] (binary IPv6 address)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

驗證界限（解碼）：`priority > 0`、
`weight ≤ 100`（`weight = 0` = 不活躍端點、
絕不被選）、TTL 如任何記錄在
1..=2,592,000 s 內。本機表不認識的
國家／大陸在線上**被允許**：路由就退回
較高範疇。

**線上格式向後相容（v1 → v2）**：v1 位元組
不變 — v1 範疇（世界／大陸／國家）保持
判別碼 `0x00..=0x02` 與其精確編碼；三個
新範疇取隨後的判別碼（`0x03..=0x05`），
順序與正典排序的全序一致
（`World < Continent < Country < Region < Asn <
IpPrefix`）。遇見判別碼 ≥ `0x03` 的 v1
解碼器本就嚴格失敗（記錄被拒）；v1 產生的
位元組仍被相同解碼（見凍結的
`geo_known_bytes` 測試）。

### 選擇（解析）

1. **範疇** — 遞減特異性、**6 個層級**：
   **最長相符 IP 前綴 → 精確 AS → 精確國家 →
   精確區域 → 大陸 → 世界**。
   - *IpPrefix*：在前綴與客戶端 IP（定位器
     所知：`client_ip`，否則已知的來源
     前綴）相符的路由中，**最長**（`len`
     最大）者勝；`len` 相等時，正典序中
     第一個（`routes` 切片的順序、編碼時
     已排序）。細分不如前綴/AS（它們
     識別精確網路）也不如國家（不透明
     編號、只在發布者與定位器共享同一
     編號時可靠）特異，但大於大陸
    （它包含於其國家、故更包含於其
     大陸）。
   - *Asn*：客戶端的精確 AS 號碼。
   - *Country*：精確國家。
   - *Region*：精確細分（不透明識別碼）。
   - *Continent*：由定位器提供**或由國家
     經本機表衍生**（`continent_of_country`）；
     表不認識的國家退回明確的
     大陸或世界。
   - *World*。
   v1 的 [`GeoLocation`]（region/asn/prefixes/
   client_ip 未設）精確產生 v1 行為：國家 →
   大陸 → 世界。無相符路由（含無 World
   範疇）→ 經典 DNS 行為，如同 GEOROUTE
   不存在。
2. **優先權** — 類 SRV 語義：只有**最小
   優先權**群組符合資格；較高優先權是
   **故障轉移**層級。
3. **權重** — 在選出的群組內，**決定性
   加權**抽取：對 nonce（解析器的查詢
   計數器）的穩定雜湊（SplitMix64）對累積
   權重。無 RNG：相同 nonce 的兩次查詢給出
   相同結果、遞增的 nonce 依權重比例
   分佈。`weight = 0` = 不活躍、絕不被選
  （即使孤身 — 移到下一層級）。
4. **健康故障轉移** — **Unhealthy** 的端點被
   放在一邊；若整個最小層級不可用，移到
   下一優先權層級。**Unknown 視為可用**
  （不懲罰未知者）。全部不可用 → 經典
   DNS 行為。

### 回應

- 單一位址：**IPv4 給 A、IPv6 給 AAAA** —
  依 qtype 對族的嚴格過濾（A 的問題絕不
  回答 IPv6，反之亦然）。
- TTL = 來源 GEOROUTE 記錄的 TTL（解碼時
  受一般 TTL 規則約束）。
- 設定 AA（權威回應 — GEOROUTE 如任何
  其他記錄是已驗證的簽署記錄）。
- 過濾後的空回應（無正確族的候選、
  全部 Unhealthy、無相符路由）→ 解析器
  繼續其經典路徑（通常是 NODATA：
  NOERROR、0 個回答）；錯誤 rcode 遵循
  一般規則（絕不快取）。
- 集合中共存的經典 A/AAAA 記錄對 geo
  **優先**（geo 僅在 qtype 無線上格式回答
  時適用）。

### 快取：明確決策

地理路由決策**絕不**以
`(qname, qtype)` 快取：它取決於解析器的
客戶端範疇，這樣的金鑰會把美國路由
送給台灣客戶端。Geo 回答每次查詢重新
計算（成本：一次 select_scope + 加權
抽取、µs 級）並繞過快取。經典的
(qname, qtype) 快取對其他一切保持不變 —
包括 geo 未決定任何東西時產生的
NODATA。錯誤 rcode 保持絕不快取。

### 健康檢查（`--health-check`）

- 每個目標（`GeoTarget`）一個 `HealthCache`：
  `probe` = 非阻塞讀取（新鮮狀態、或
  Unknown），**絕不**從解析路徑觸發
  檢查。
- 刷新：若選出的候選是 Unknown／不在
  快取中，排程**一次**刷新作為背景任務
  （`tokio::spawn_blocking`：TCP connect 是
  阻塞的、在 UDP 迴圈中被禁止）。防濫發：
  每目標一次一個刷新（HashSet）、在前一個
  刷新結束後可重啟。
- `TcpChecker`：對 `target:port` 的有界 TCP
  connect（2 秒逾時）。**埠無法從 DNS
  記錄衍生**：`--health-port` 設定（預設
  443）。檢查器**刻意天真**（成功的
  connect ≠ 健康的服務）— 可改進（HTTP、
  TLS、應用層）而無格式變更。
- 條目 TTL：`--health-ttl-secs`（預設 5 秒）；
  過期條目 = Unknown = 可用。

### 安全

- GEOROUTE 記錄如任何其他記錄是簽署
  記錄：任何使用前對鏈驗證（擁有者、
  序號、record_hash）。**無驗證即無網路
  信任** — 未驗證的 DHT GEOROUTE 絕不
  回答。
- 客戶端的位置來自**內嵌 GeoIP 表**
  （RIR 委派、見上）由其來源 IP 衍生 —
  或來自解析器明確的本機組態
  （`--geo-ip-table` 或
  `--geo-country`／`--geo-continent`／
  `--geo-asn`／`--geo-region`，後者優先），
  絕不來自網路 GeoIP API（無查詢洩漏、
  無相依）。
- 抽取是決定性且本機的：無秘密、無
  不可預測性需求（安全在於記錄
  簽章，而非抽取）。

### 完整範例（參考規格）

`www.example.scone` 發布（apex
`example.scone`、整個集合服務每個子名）：

| 範疇 | 端點 | 優先權 | 權重 |
|---|---|---|---|
| US（840） | 10.1.0.1 | 10 | 70 |
| US（840） | 10.1.0.2 | 10 | 30 |
| US（840） | 10.2.0.1 | 20 | 100 |
| TW（158） | 10.3.0.1 | 10 | 80 |
| TW（158） | 10.4.0.1 | 10 | 20 |
| TW（158） | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

預期行為：

- **US** 客戶端 → 10.1.0.1（≈70%）或
  10.1.0.2（≈30%）；若兩者皆 Unhealthy →
  故障轉移 p20 → **10.2.0.1**；
- **TW** 客戶端 → 10.3.0.1（≈80%）或
  10.4.0.1（≈20%）；
- **FR** 客戶端（250）→ 集合中無 FR 或
  EU 路由 → WORLD → **10.0.0.1**；
- 全部 Unhealthy → 無 geo 決定 → NODATA
 （NOERROR、0 個回答）；
- 對僅 IPv6 路由的 **A** 問題 → 空回答
  （A 中絕無 IPv6）；
- 無適用規則的網域 → 相同問題、經典
  行為（NODATA：GEOROUTE 無線上格式
  A rdata）。

### CLI 選項（`scone relay`）

| 選項 | 預設 | 角色 |
|---|---|---|
| （無 — GeoDNS 恆開、內嵌表） | 開 | 依來源 IP 自動路由 |
| `--geo-ip-table <path>` | — | **本機 GeoIP 表**（見下）。優先於所有靜態 `--geo-*` 選項 |
| `--geo-country <u16>` | — | 客戶端國家、ISO 3166-1 數值（840=US…）— 靜態 |
| `--geo-continent <u16>` | — | 客戶端大陸、UN M49（150=EU…）；省略時由國家衍生 — 靜態 |
| `--geo-asn <u32>` | — | 客戶端 AS（64500…）— 靜態、v2 新增 |
| `--geo-region <u32>` | — | 客戶端細分（不透明的發布者識別碼）— 靜態、v2 新增 |
| `--health-check` | 關 | 啟用檢查（健康故障轉移） |
| `--no-health-check` | 關 | **明確停用**檢查 — 勝過 `--health-check`（見優先順序） |
| `--health-ttl-secs <u64>` | 5 | 健康條目的 TTL |
| `--health-port <u16>` | 443 | 檢查用 TCP 埠（無法從 DNS 記錄衍生） |

#### 本機 GeoIP 表（`--geo-ip-table`）

前綴的本機 TEXT 檔案、啟動時讀取**一次**
（最多 100,000 條規則、`MAX_GEOIP_RULES`）。
每行一條規則、`#` = 註解、空行忽略：

```text
# prefix/mask        country continent asn     region
10.0.0.0/8           840   3          .       .
2001:db8::/32        250   150        64500   11
192.0.2.0/24         158   9
```

- `prefix/mask`：IPv4（`len ≤ 32`、轉為
  16 位元組形式、零尾 — 與線上格式相同的
  慣例）或 IPv6（`len ≤ 128`）。**主機位元
  必須為零**（`10.1.2.3/8` 被拒）；
- `country`：**數值** ISO 3166-1 `u16`、必填；
- `continent`：UN M49 `u16`、可選（`.` 或
  省略 = 不存在、大陸可由國家衍生）；
- `asn`：`u32`、可選（`.` = 不存在）；
- `region`：不透明 `u32`、可選（`.` = 不存在）。

解析：**DNS 查詢的來源 IP**（每客戶端：
UDP `recv_from`／TCP `peer_addr`；未知 →
解析器的參考 IP）以 16 位元組比較；
**最長相符前綴**的規則獲勝（平手時檔案
順序中第一個）；無相符 → 空位置
（World）。

**嚴格性**：檔案由營運者組態 — 無效行
（畸形前綴、非數值國家、未知大陸、
過多欄位…）是攜帶**出錯行號**的**啟動
錯誤**；絕不靜靜忽略一行、絕不部分
載入表。

#### 選項優先順序

- **定位器**：`--geo-ip-table` **優先於**靜態
  選項
  （`--geo-country`／`--geo-continent`／`--geo-asn`／`--geo-region`），
  提供表時靜態選項被忽略。無表時
  套用靜態選項。
- **健康**：`--no-health-check` 是**專用的
  獲勝旗標**，勝過 `--health-check` — 組合
  `--health-check --no-health-check` 使檢查
  保持停用、無輸入順序歧義。無
  `--health-check` 時檢查預設關閉：
  **Unknown = 可用**，即使目標已死 geo
  回應仍送出。（`NoopChecker` — 恆為
  Healthy — 也存在，用於掛載健康結構
  而不擱置任何端點：測試／偏執。）

範例：

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-ip-table /etc/scone/geoip.txt \
            --no-health-check
```

靜態 v2 範例：

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-country 250 --geo-asn 64500 --geo-region 11 \
            --health-check --health-port 443
```


## DoH / DoT（加密傳輸）— 經外部前端

Scone DNS 伺服器說**純 UDP 與 TCP**（RFC 1035 /
7766）— 這是原生範圍、完整且經測試。到
*終端客戶端*的加密傳輸（DoH —
DNS-over-HTTPS、RFC 8484；DoT — DNS-over-TLS、
RFC 7858）**委派給標準前端**：Scone 服務
資料、前端服務傳輸。

為何此設計：原生 DoH 伺服器需要在節點內
完整的 HTTPS 技術棧（憑證、續期、HTTP/2）—
在出現瀏覽器端終端使用者之前，一個巨大
的營運表面換來零效益。既有前端做得更好
更久；節點保持最小。

### 範例：unbound 在 Scone 中繼站前做 DoH

`/etc/unbound/unbound.conf`：

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) on 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # the Scone relay (--dns 127.0.0.1:5353)
```

憑證（`scone.pem`／`scone.key`）如任何 TLS
服務般取得 — 例如解析器網域上的
certbot/acme.sh。DoH 客戶端然後指向
`https://dns.example.tld/dns-query`。

### 範例：knot-resolver 做 DoT

`/etc/knot-resolver/kresd.conf`：

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### 前端增加什麼

- **客戶端 → 前端通道的加密**（敵對
  區段：使用者的網路）；前端 → 中繼站
  區段保持本機（同機或受信網路）。
- **一個額外的前端側快取**（unbound/knot
  也快取 — 無害、Scone TTL 仍是上界）。
- **一個穩定端點**（IPv4/IPv6、網域），而
  中繼站移動。

### 前端不做什麼

- 它不*驗證*任何東西：中繼站服務的每個
  回答在進入快取前已對鏈驗證（見
  「解析路徑」）。前端轉發已權威的
  回答。
- 它不取代去中心化：DoH 前端是*被營運的*
  服務 — 解析器營運者的選擇，而非協定
  的。每個 Scone 中繼站保持可經原生
  UDP/TCP 獨立使用。

### 未來的原生實作（已載明、未承諾）

原生 DoH 傳輸（`--doh ADDR`）仍可想像：
線上格式相同（RFC 8484 重用二進位 DNS
訊息），它只是 POST/GET 解碼 + TLS proxy
（caddy/nginx 處理憑證）後的 HTTP/2，節點
本身保持無 TLS。無進行中工作 — 需求
將隨終端使用者到來。

# Máy chủ DNS UDP + TCP (M6 + M7b) — `scone dns`

> Đặc tả chuẩn của bề mặt DNS Scone.
> Bất kỳ sự khác biệt tài liệu/mã = bug (sửa nguồn chân lý).

## Tổng quan

**Máy chủ DNS** là một bề mặt tùy chọn của relay: nó trả lời các
truy vấn DNS **UDP và TCP** (RFC 7766, cùng cổng) cho các tên
Scone **chỉ từ dữ liệu đã xác minh đối với chuỗi**, và cung cấp
một **fallback đệ quy tùy chọn** (`addr:port`) cho mọi thứ còn
lại. Nó không sở hữu chuỗi, kho, hay DHT: các quyết định tồn
tại/hết hạn đi qua **snapshot DNS có thẩm quyền** (một góc nhìn
RAM được xuất bản sau mỗi khối được áp dụng — xem dưới), và các
lần lấy bản ghi đi qua relay (tiến trình), qua kênh nội bộ
`RelayCommand`/`ResolveLocal` (trong tiến trình, một vòng) —
cùng đường đã xác minh như CLI.

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

## Khởi động

```bash
scone relay --dns 127.0.0.1:5353            # DNS surface only
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + recursive fallback
```

Relay loan báo địa chỉ hiệu lực trên stdout (hợp đồng đọc-được-
bởi-máy, như `p2p:`): một dòng `dns: <addr>` ngay khi socket
được buộc.

Client kiểm chứng: `dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Đường phân giải (THẨM QUYỀN = chuỗi, luôn luôn)

0. **Snapshot DNS có thẩm quyền**: một góc nhìn bất biến
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`,
   nguồn cấp `tokio::sync::watch`) được xuất bản bởi lõi relay
   **sau mỗi khối được áp dụng** (push, không poll) và lúc khởi
   động. Các quyết định tồn tại/hết hạn — NXDOMAIN, tên miền hết
   hạn — được phục vụ từ RAM qua snapshot này: một clone `Arc` +
   một tra cứu bản đồ, **không bao giờ đi vào vòng chạy đơn-chủ
   của relay**. Đường đầy đủ
   (`RelayCommand`/`ResolveLocal`) vẫn dành cho các lần lấy bản
   ghi thực sự (kéo DHT), hiếm nhờ cache. Snapshot KHÔNG BAO GIỜ
   mang dữ liệu bản ghi: chỉ tồn tại và hết hạn.
   - **Delta copy-on-write O(chạm tới)**: bản đồ là một
     `Arc<HashMap>`; một mở rộng đỉnh xuất bản delta của khối
     (các giao dịch riêng của khối nêu tên mọi tên miền chúng có
     thể thay đổi — Register/Assign/Update/Renew/TransferDomain —
     cộng các lần gỡ hết hạn-GC tất định đánh giá lúc push)
     qua `Arc::make_mut`: bản đồ chỉ được clone NẾU một trình
     đọc còn giữ góc nhìn trước. Khởi động và reorg dựng lại
     snapshot đầy đủ (một lần đi backend phân trang).
   - Độ mới: một trình đọc có thể thấy một snapshot cũ vài trăm
     ms trong lúc áp dụng khối — cùng độ cũ mà một RPC trạng
     thái sẽ có; các TTL cache DNS không bị ảnh hưởng (các câu
     trả lời vẫn có thẩm quyền theo nguồn gốc của chúng).
1. **Cache chia shard**: [`ShardedCache`] — 16 shard `Cache` độc
   lập khóa theo `hash(qname)` (qtype chia sẻ shard: tính cục bộ
   theo-tên). Một tên nóng cạnh tranh với 1/16 lưu lượng thay vì
   xếp hàng sau một mutex duy nhất ở hàng nghìn qps. Biên TOÀN
   CỤC (100.000 mục / 64 MiB mặc định,
   `--dns-cache-limit` / `--dns-cache-max-mem`) chia theo shard
   (trường hợp xấu được ghi nhận: một hệ số 1 + N/shards). Kích
   thước rdata THẬT của từng mục được tính (~128 byte chi phí
   trên mỗi mục) và trục xuất FIFO kích hoạt ở biên đầu tiên
   trong hai biên — nắp bộ nhớ giữ ngay cả với các mục kích
   thước đầy (`MAX_DHT_RECORD_LEN`). TTL 60 s
   (có thẩm quyền) / kẹp 600 s (fallback). Cache âm tính
   (NXDOMAIN/NODATA) theo cùng quy tắc; mỗi mục mang nguồn gốc
   (có thẩm quyền hay fallback) của nó để bit AA giữ trung thực
   khi trúng. **Không bao giờ cache SERVFAIL** (thất bại cục bộ
   thoáng qua) **hay REFUSED** (trạng thái cấu hình): cả hai
   được tính lại ở mỗi truy vấn. Trục xuất FIFO khi đầy. **Đường
   nóng (lấy cảm hứng Cloudflare/1.1.1.1)**:
   - các mục dùng chung `Arc<CacheEntry>`: một lần trúng là một
     bump refcount, KHÔNG BAO GIỜ một clone sâu rdata (`put`
     THAY THẾ Arc — không đột biến chia sẻ; TTL vẫn được kiểm
     dưới khóa shard);
   - **tra cứu không cấp phát**: bản đồ hai tầng
     (`Box<str>` qname → qtype → mục), `&str` mượn của truy vấn
     thăm trực tiếp (`Borrow<str>`) — khóa tuple `(String, u16)`
     cũ cấp phát một `String` mới trên MỌI lần tra cứu;
   - **các bản ghi đóng gói liền khối**: một đệm `Box<[u8]>` cho
     mỗi mục (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` cho mỗi
     bản ghi) thay vì một `Vec` cho mỗi bản ghi — ngân sách bộ
     nhớ theo dõi chính xác những gì được cấp phát;
   - **nén tên RFC 1035 lúc xuất**: owner của các câu trả lời có
     thẩm quyền là một con trỏ nén 2 byte (`0xC00C`) vào câu hỏi
     được vang lại — câu hỏi không bao giờ được mã hóa lại đầy
     đủ cho mỗi bản ghi.
2. **Hình-Scone?** Tiền lọc nghiêm ngặt: các nhãn `[a-z0-9-_]`
   (1..63), ≥ 2 nhãn, TLD 1..63 byte `[a-z0-9-]` (xem
   `TldName::MAX_LEN`), tổng chiều dài ≤ 253 — **và TLD ngoài
   root ICANN** (`is_icann_tld`, danh sách IANA đóng băng +
   RFC 2606: `com`, `fr`, `org`… không bao giờ thuộc về Scone).
   Một tên KHÔNG THỂ là tên Scone (ví dụ `www.foo_bar`, gạch
   dưới trong TLD) đi sang fallback; một tên có thể là Scone
   (`absent.uip`) là việc của chuỗi: chưa đăng ký → NXDOMAIN có
   thẩm quyền, kể cả khi không có upstream. Một TLD ICANN
   (`example.com`) được chuyển tiếp tới các upstream nếu cấu
   hình, REFUSED nếu không — không bao giờ được Scone phân giải
   một cách có thẩm quyền.
3. **Tìm apex**: các hậu tố từ dài nhất đến ngắn nhất (tối thiểu
   2 nhãn). Hậu tố đầu tiên mà trạng thái on-chain được relay
   biết = apex; bộ bản ghi phục vụ toàn bộ qname (các tên con
   kể cả — hành vi wildcard ngầm, quyết định mở).
4. **`resolve_local` (kênh nội bộ)**: một vòng duy nhất, không
   bao giờ một trình chờ DHT. Trạng thái on-chain (`registered`,
   owner, sequence) + bản ghi cache DHT cục bộ **được xác minh
   lại** (owner, sequence, record_hash == cam kết on-chain).
   Một bản ghi không phù hợp không bao giờ trả lời — sẵn sàng ≠
   thẩm quyền.
   - chưa đăng ký → NXDOMAIN;
   - đăng ký mà không có bản ghi hợp lệ → NODATA (NOERROR, 0
     câu trả lời);
   - bản ghi đã xác minh → các câu trả lời lọc theo qtype.

## Codec (RFC 1035, tập con nghiêm ngặt)

- Truy vấn: header 12 byte, **một** câu hỏi, QCLASS = IN,
  **không nén** (con trỏ = từ chối). Nhãn chuyển chữ thường và
  được xác thực. QR=1, QDCOUNT≠1, nhãn vô hiệu → **im lặng**
  (rác không tốn gì).
- Phản hồi: QR|RD(vang lại)|RA, **AA chỉ trên các câu trả lời
  có thẩm quyền** (chuỗi, mới hay cache — không bao giờ trên
  fallback hay các RCODE lỗi), câu hỏi chép nguyên văn, các câu
  trả lời có thẩm quyền với **owner nén RFC 1035** (con trỏ 2
  byte `0xC00C` vào câu hỏi được vang lại); các phản hồi
  upstream fallback được phục vụ nguyên trạng (không bao giờ
  mã hóa lại).
- Các kiểu phục vụ: A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV,
  CAA** (+ Unknown mã hóa lại nguyên văn; tổng 9 kiểu bản địa,
  rdata mã hóa theo RFC — 1035/2782/8659; CAA nghiêm ngặt: bit
  tới hạn 0x80 duy nhất, tag chữ thường). Một truy vấn A/AAAA
  gồm các CNAME của bộ (đuổi alias). Truy vấn ANY (255): toàn
  bộ bộ được lọc bởi codec.
- Giới hạn (M7b): **ngân sách phản hồi phụ thuộc vận chuyển** —
  512 byte qua UDP (payload cổ điển RFC 1035 §2.3.4, không
  EDNS), 65.535 qua TCP. Một bộ vượt ngân sách bị cắt tại bản
  ghi cuối vừa, với **TC=1** (cắt cụt trung thực, RFC 1035
  §4.2.1): client thử lại qua TCP (RFC 7766), nơi bộ đầy đủ
  được phục vụ. Một phản hồi upstream lớn hơn ngân sách UDP
  được mã hóa lại cụt (TC=1) thay vì giao nguyên trạng; qua TCP
  nó đi qua byte cho byte. Sự cắt cụt cũng áp dụng cho các lần
  trúng cache (ngân sách áp dụng lúc phản hồi, không phải lúc
  lấp cache).
- RCODE: 0 NOERROR (có hay không có câu trả lời), 2 SERVFAIL
  (thất bại cục bộ), 3 NXDOMAIN (có thẩm quyền), 5 REFUSED
  (không-Scone không có upstream).

## Fallback đệ quy

- Các upstream `addr:port` (UDP), thử theo thứ tự, hết hạn 2 s
  mỗi upstream, socket **connect** với upstream (lọc hạt nhân:
  một gói từ bất kỳ nguồn khác không bao giờ được chấp nhận).
  Phản hồi upstream chỉ được chấp nhận nếu nó **khớp truy
  vấn**: cùng TXID, QR=1, câu hỏi (QNAME+QTYPE+QCLASS) chép
  byte cho byte. Một phản hồi không phù hợp → SERVFAIL, và
  không có gì được cache.
- Phản hồi upstream đã xác minh được truyền nguyên trạng; các
  câu trả lời của nó được cache với nguồn gốc `fallback` (AA
  không bao giờ đặt, kể cả khi trúng cache) và TTL kẹp 600 s.
  Các RCODE lỗi upstream (khác NOERROR/NXDOMAIN) không được
  cache.
- **Riêng tư mặc định**: không upstream = REFUSED cho các tên
  ngoài hình Scone.

## Cache âm tính DHT (chống khuếch đại)

Một **lần kéo DHT không trả lời** (bản ghi kỳ vọng on-chain —
`record_hash` được cam kết — nhưng vắng trong cache cục bộ và
các peer được hỏi) đỗ tên vào `DHT_NEGATIVE_TTL_SECS` (30 s)
trong một cache âm tính bị chặn (FIFO, nắp
`DHT_NEGATIVE_CAP` = 4.096 tên): các lần phân giải lặp lại của
cùng tên trong TTL bị **dập tắt** (đếm, không bao giờ kéo lại).
Một máy quét — hay một tên nóng mà chủ chưa từng xuất bản —
không thể còn khuếch đại một truy vấn DNS thành một `GetRecord`
cho mỗi peer lưới cho mỗi truy vấn. Mục âm tính chết tại TTL,
khi bản ghi đến (lần kéo thành công ghi vào cache thật), hay
bị đẩy ra nắp FIFO.

Các bộ đếm phơi bày (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — xem "Metrics" trong
`relay.md`).

## TCP (M7b, RFC 7766)

Cùng cổng với UDP (buộc TCP thứ hai trên địa chỉ `--dns`). Vận
chuyển theo RFC 1035 §4.2.2: mọi thông điệp được tiền tố chiều
dài trên **2 byte** big-endian.

- **Truy vấn**: tiền tố chiều dài ≥ 1 và ≤ 512 byte (một truy
  vấn DNS vừa payload cổ điển — RFC 7766 §6; không phục vụ
  EDNS). Ngoài biên → đóng kết nối tức thì.
- **Phản hồi**: ngân sách 65.535 byte (tối đa tiền tố 2 byte)
  — một bộ quá lớn cho UDP được phục vụ **đầy đủ** ở đây,
  TC=0.
- **Pipelining**: nhiều truy vấn mỗi kết nối được phục vụ tuần
  tự, theo thứ tự (một lần ghi = một phản hồi có tiền tố).
- **Hết hạn đọc**: 5 s mỗi lần đọc (tiền tố hay thân) — một
  client im lặng bị ngắt, khe của nó được giải phóng.
- **Kết nối đồng thời**: tối đa 64 (semaphore); cái thứ 65 bị
  từ chối tức thì (đóng kết nối), không bao giờ xếp hàng —
  không tăng trưởng không bị chặn của tác vụ hay bộ nhớ.
- Lỗi đọc/ghi/hết hạn: nhật ký debug + đóng sạch. **Không bao
  giờ panic trên đầu vào mạng.**
- Rác (truy vấn không hiểu được) nhận im lặng nhưng không đóng
  kết nối: truy vấn (hợp lệ) tiếp theo được phục vụ.

```text
client                     server
  │ [len u16][query]         │
  │ ─────────────────────►   │ cache → authoritative → fallback
  │ ◄─────────────────────   │ [len u16][response ≤ 64 KiB]
  │ (UDP had answered TC=1 → the client retries here)
```

## Biên và rào chắn

| Rào chắn | Giá trị |
|---|---|
| Datagram UDP (truy vấn/phản hồi) | 4096 byte (từ chối khi vượt) |
| Ngân sách phản hồi UDP (TC=1 khi vượt) | 512 byte (không EDNS) |
| Truy vấn TCP tối đa | 512 byte (đóng khi vượt) |
| Phản hồi TCP tối đa | 65.535 byte |
| Kết nối TCP đồng thời | 64 (semaphore, từ chối tức thì) |
| Hết hạn đọc TCP | 5 s |
| Cache | biên TOÀN CỤC 100.000 mục / 64 MiB mặc định, chia trên 16 shard (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| TTL có thẩm quyền | **theo từng bản ghi** (`record.ttl`, biên 1..=2.592.000 s; hằng số lịch sử 60 s chỉ còn cho các phản hồi dịch vụ) |
| TTL fallback (kẹp) | 600 s |
| Truy vấn UDP song song | 4.096 (`MAX_INFLIGHT_UDP`, semaphore; vượt: bỏ, client thử lại) — chỉnh được `--dns-max-inflight` (tối thiểu 1) |
| Cache âm tính DHT | 30 s (`DHT_NEGATIVE_TTL_SECS`), nắp FIFO 4.096 tên |
| Hết hạn mỗi upstream | 2 s |

Các quy tắc dự án được tôn trọng: không bao giờ panic trên dữ
liệu mạng; giải mã nghiêm ngặt mọi nơi; máy chủ không chạm
chuỗi, kho, hay DHT (relay vẫn là thẩm quyền xác minh duy
nhất).

## Kiểm thử

- Đơn vị (`scone-network::dns`): codec nghiêm ngặt (QR, nén,
  từ chối nhãn), các câu trả lời có thẩm quyền, NODATA,
  NXDOMAIN, REFUSED không upstream, fallback tới một mock UDP,
  cache bị chặn, cắt khúc TXT, phân tích upstream; sửa M6:
  trung thực bit-AA (có thẩm quyền vs fallback vs lỗi, gồm cả
  trúng cache), cắt cụt TC=1 của một bộ quá cỡ, từ chối phản
  hồi upstream (lệch TXID hay câu hỏi), không cache
  SERVFAIL/REFUSED; cache chia shard: trải shard, chia biên
  toàn cục, 4 worker đồng thời trên một tên nóng; đường nóng:
  tra cứu không cấp phát, mục đóng gói, nén owner.
- M7b: ngân sách theo vận chuyển (UDP 512 byte → TC=1, TCP →
  bộ đầy đủ), cắt cụt một lần trúng cache, `run_tcp` thật (bộ
  > 512 phục vụ đầy đủ, pipelining 2 truy vấn, truy vấn quá
  cỡ → đóng, client im lặng → hết hạn 5 s, rác rồi truy vấn
  hợp lệ trên cùng kết nối), phản hồi upstream > 512 mã hóa
  lại TC=1 qua UDP / byte cho byte qua TCP.
- Tích hợp thật (`tests/dns_server.rs`): một relay đầy đủ (gồm
  tạo khối devnet) + đăng ký + UpdateDomain + PutRecord, rồi
  các truy vấn UDP thật: A đã xác minh (bit AA), TXT, NODATA
  AAAA, NXDOMAIN, fallback qua mock, truy vấn lại, tên con qua
  apex.
- E2E nhị phân (`crates/scone/tests/dns_cli.rs`): tệp nhị
  phân `scone` thật — `relay --dns`, `identity generate`,
  `domain register`, `domain update --file`, rồi `dig`
  (dnsutils) (A đã xác minh, TXT, NXDOMAIN, REFUSED) trên một
  cổng UDP thật.

## Các quyết định mở

- Không EDNS (ngân sách UDP giữ 512 byte; cắt cụt TC=1 + TCP
  phủ nhu cầu devnet).
- Bộ bản ghi không mang tên theo từng bản ghi: cả bộ phục vụ
  mọi tên con của apex (wildcard ngầm). Địa chỉ hóa theo-từng-
  bản-ghi sẽ cần mở rộng `DnsRecord` (protocol).
- Fallback chuyển tiếp truy vấn thô: một upstream độc hại có
  thể trả lời bất cứ gì (phản hồi chưa xác minh — đây là mô
  hình đệ quy chuẩn; các tên Scone KHÔNG BAO GIỜ đi qua nó).

## GeoDNS (định tuyến địa lý)

> Đặc tả chuẩn. Bất kỳ khác biệt tài liệu/mã = bug.

### Bảng GeoIP nhúng (tự động)

Mọi tệp nhị phân nhúng một **bảng ánh xạ IP → quốc gia** biên
dịch vào relay: **không cần cấu hình**, định tuyến địa lý hoạt
động mọi nơi, tự động.

- **Nguồn dữ liệu**: các **tệp ủy quyền RIR** (RIPE NCC, ARIN,
  APNIC, LACNIC, AFRINIC), tải và phân phối lại tự do:
  `https://ftp.ripe.net/pub/stats/<rir>/delegated-<rir>-extended-latest`
  (quốc gia đăng ký của mỗi khối IPv4/IPv6 được ủy quyền).
  Ngày tạo nằm trong header bảng
  (`assets/embedded-geo.table`).
- **Tái tạo**: `scripts/gen-geo-table.py` (tải 5 tệp, chuyển
  đổi, chuẩn hóa tiền tố — ~176.000 quy tắc).
- **Độ mịn**: quốc gia (ISO 3166-1 số) + châu lục (UN M49) dẫn
  xuất. Không ASN hay vùng (vắng trong các ủy quyền).
- **Hiệu năng**: tra cứu **tìm kiếm nhị phân** trên các dải đã
  sắp (chỉ mục dựng một lần khi dùng lần đầu, theo họ v4/v6)
  — đo được ~95 ns/tra cứu nóng, ~50 µs truy cập ngẫu nhiên
  trên bảng đầy đủ (bench: `cargo test -p scone-network
  --release --test geo_bench -- --ignored`).
- **Tự-định-vị relay**: định tuyến theo **IP nguồn của mỗi
  truy vấn** — một relay LAN định tuyến các client cục bộ của
  mình sang vùng đúng tự động.
- **Ưu tiên**: một bảng vận hành viên tường minh
  (`--geo-ip-table`/geo-config) **ghi đè** bảng nhúng.

### Tổng quan

Một tên miền có thể xuất bản các quy tắc **định tuyến địa lý**
trong bộ bản ghi có ký của mình: các bản ghi `GEOROUTE` (kiểu
riêng Scone `32.768`, ngoài dải IANA). Bộ phân giải trả lời các
câu hỏi A/AAAA cho các tên miền này bằng **địa chỉ của endpoint
được chọn bởi quy tắc** (scope gần nhất với vị trí CLIENT, dẫn
xuất từ IP nguồn qua bảng), thay vì một bản ghi A/AAAA tĩnh.
Mọi thứ khác (tên miền không GEOROUTE, qtype khác) theo đường
DNS cổ điển, **byte cho byte**.

### Các chỉ số GeoDNS

Điểm cuối Prometheus phơi bày phân tách câu trả lời geo:

- `scone_dns_geo_answers_total` — tổng các câu trả lời
  GEOROUTE được phục vụ;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — theo
  châu lục;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — theo
  quốc gia (bị chặn bởi chính các tiêu chuẩn: ≤ 9 châu lục,
  ≤ 250 quốc gia).

### Bản ghi GEOROUTE (wire)

Một GEOROUTE = một **scope** + ≤ 32 **endpoint** (sắp chuẩn lúc
giải mã: priority tăng, weight giảm, target tăng; trùng = từ
chối):

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

Biên xác thực (giải mã): `priority > 0`, `weight ≤ 100`
(`weight = 0` = endpoint không hoạt động, không bao giờ được
chọn), TTL trong 1..=2.592.000 s như mọi bản ghi. Một
quốc gia/châu lục không rõ với các bảng cục bộ vẫn **được phép**
trên wire: định tuyến đơn giản quay về một scope cao hơn.

**Tương thích ngược wire (v1 → v2)**: các byte v1 KHÔNG ĐỔI —
các scope v1 (thế giới/châu lục/quốc gia) giữ discriminant
`0x00..=0x02` và mã hóa chính xác của chúng; ba scope mới lấy
các discriminant kế tiếp (`0x03..=0x05`) trong một thứ tự toàn
phần nhất quán với sắp chuẩn
(`World < Continent < Country < Region < Asn < IpPrefix`). Một
bộ giải mã v1 gặp discriminant ≥ `0x03` đã thất bại nghiêm
ngặt từ trước (bản ghi bị từ chối); các byte do v1 tạo ra vẫn
được giải mã giống hệt (xem kiểm thử `geo_known_bytes`, đóng
băng).

### Lựa chọn (phân giải)

1. **Scope** — tính cụ thể giảm dần, **6 tầng**: **tiền tố IP
   khớp dài nhất → AS chính xác → quốc gia chính xác → vùng
   chính xác → châu lục → thế giới**.
   - *IpPrefix*: trong các tuyến mà tiền tố khớp IP của client
     (như locator biết: `client_ip`, nếu không các tiền tố
     nguồn đã biết), cái **dài nhất** (`len` max) thắng; khi
     bằng `len`, cái đầu theo thứ tự chuẩn (thứ tự lát cắt
     `routes`, sắp lúc mã hóa). Một vùng kém cụ thể hơn
     tiền tố/AS (những cái định danh chính xác mạng) và kém
     hơn quốc gia (đánh số mờ, chỉ đáng tin nếu bên xuất bản
     và locator chia sẻ cùng một cái), nhưng hơn châu lục
     (nó nằm trong quốc gia của nó, do đó a fortiori trong
     châu lục của nó).
   - *Asn*: số AS chính xác của client.
   - *Country*: quốc gia chính xác.
   - *Region*: vùng chính xác (định danh mờ).
   - *Continent*: do locator cung cấp **hoặc dẫn xuất từ quốc
     gia** qua bảng cục bộ (`continent_of_country`); một quốc
     gia không rõ bảng quay về châu lục tường minh hay thế
     giới.
   - *World*.
   Một [`GeoLocation`] v1 (region/asn/prefixes/client_ip không
   đặt) tạo ra đúng hành vi v1: quốc gia → châu lục → thế
   giới. Không tuyến khớp (kể cả khi không có scope World) →
   hành vi DNS cổ điển, như thể GEOROUTE không tồn tại.
2. **Priority** — ngữ nghĩa kiểu SRV: chỉ nhóm **priority tối
   thiểu** đủ điều kiện; các priority cao hơn là các tầng
   **failover**.
3. **Weight** — trong nhóm được bầu, rút **trọng số tất định**:
   băm ổn định (SplitMix64) của một nonce (bộ đếm truy vấn của
   bộ phân giải) đối với các trọng số cộng dồn. Không RNG: hai
   truy vấn cùng nonce cho cùng kết quả, các nonce tăng phân
   phối tỷ lệ với trọng số. `weight = 0` = không hoạt động,
   không bao giờ được chọn (kể cả đứng một mình — sang tầng
   kế).
4. **Failover sức khỏe** — một endpoint **Unhealthy** bị để
   riêng; nếu toàn bộ tầng tối thiểu không khả dụng, sang tầng
   priority kế. **Không-clear coi như khả dụng** (cái không rõ
   không bị phạt). Mọi thứ không khả dụng → hành vi DNS cổ
   điển.

### Phản hồi

- Một địa chỉ duy nhất: **A nếu IPv4, AAAA nếu IPv6** — lọc
  nghiêm ngặt theo họ theo qtype (một câu hỏi A KHÔNG BAO GIỜ
  trả lời một IPv6, và ngược lại).
- TTL = của bản ghi GEOROUTE nguồn (bị chặn lúc giải mã bởi
  các quy tắc TTL chung).
- AA đặt (phản hồi có thẩm quyền — GEOROUTE là một bản ghi có
  ký đã xác minh như mọi bản ghi khác).
- Phản hồi rỗng sau lọc (không ứng viên đúng họ, mọi thứ
  Unhealthy, không tuyến khớp) → bộ phân giải tiếp tục đường
  cổ điển của mình (thường là NODATA: NOERROR, 0 câu trả lời);
  các rcode lỗi theo quy tắc chung (không bao giờ cache).
- Một bản ghi A/AAAA cổ điển cùng tồn tại trong bộ được ƯU
  TIÊN over geo (geo chỉ áp dụng khi vắng câu trả lời wire cho
  qtype).

### Cache: quyết định tường minh

Quyết định định tuyến geo **KHÔNG BAO GIỜ được cache** dưới
`(qname, qtype)`: nó phụ thuộc scope client của bộ phân giải,
và một khóa như vậy sẽ phục vụ tuyến US cho một client TW. Các
câu trả lời geo được tính lại ở mỗi truy vấn (chi phí: một
select_scope + rút trọng số, µs) và bypass cache. Cache cổ
điển (qname, qtype) không đổi cho mọi thứ khác — kể cả NODATA
được tạo khi geo không quyết gì. Các rcode lỗi vẫn không bao
giờ cache.

### Kiểm tra sức khỏe (`--health-check`)

- `HealthCache` theo từng đích (`GeoTarget`): `probe` = đọc
  không chặn (trạng thái mới, hay Unknown), **không bao giờ**
  một lần kiểm được kích hoạt từ đường phân giải.
- Làm mới: nếu một ứng viên được bầu là Unknown/vắng trong
  cache, MỘT lần làm mới được lên lịch như một tác vụ nền
  (`tokio::spawn_blocking`: connect TCP là chặn, bị cấm trong
  vòng UDP). Chống spam: một lần làm mới mỗi lúc cho mỗi đích
  (HashSet), khởi động lại được sau khi lần làm mới trước kết
  thúc.
- `TcpChecker`: connect TCP bị chặn (hết hạn 2 s) tới
  `target:port`. **Cổng không thể dẫn xuất từ một bản ghi
  DNS**: `--health-port` đặt nó (mặc định 443). Trình kiểm
  **cố ý ngây thơ** (connect thành công ≠ dịch vụ khỏe) — có
  thể cải thiện (HTTP, TLS, mức ứng dụng) mà không đổi định
  dạng.
- TTL mục: `--health-ttl-secs` (mặc định 5 s); mục hết hạn =
  Unknown = khả dụng.

### Bảo mật

- Các bản ghi GEOROUTE là các bản ghi có ký như mọi bản ghi
  khác: được xác minh đối với chuỗi (owner, sequence,
  record_hash) trước mọi sử dụng. **Không tin mạng nếu chưa
  xác minh** — một GEOROUTE DHT chưa xác minh không bao giờ
  trả lời.
- Vị trí client đến từ **bảng GeoIP nhúng** (ủy quyền RIR, xem
  trên) dẫn xuất từ IP nguồn của nó — hay từ cấu hình cục bộ
  tường minh của bộ phân giải (`--geo-ip-table` hay
  `--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`,
  cái thắng), không bao giờ từ một API GeoIP mạng (không rò
  truy vấn, không phụ thuộc).
- Phép rút là tất định và cục bộ: không bí mật, không cần tính
  khó đoán (an ninh là của chữ ký bản ghi, không phải của phép
  rút).

### Ví dụ hoàn chỉnh (đặc tả tham chiếu)

`www.example.scone` xuất bản (apex `example.scone`, cả bộ phục
vụ mọi tên con):

| Scope | Endpoint | Priority | Weight |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Hành vi kỳ vọng:

- client **US** → 10.1.0.1 (≈70%) hay 10.1.0.2 (≈30%); nếu cả
  hai Unhealthy → failover p20 → **10.2.0.1**;
- client **TW** → 10.3.0.1 (≈80%) hay 10.4.0.1 (≈20%);
- client **FR** (250) → không tuyến FR hay EU trong bộ → WORLD
  → **10.0.0.1**;
- mọi thứ Unhealthy → không quyết geo → NODATA (NOERROR, 0 câu
  trả lời);
- câu hỏi **A** trên một tuyến chỉ-IPv6 → câu trả lời rỗng
  (không bao giờ một IPv6 trong A);
- tên miền không có quy tắc áp dụng được → cùng câu hỏi, hành
  vi cổ điển (NODATA: GEOROUTE không có rdata A wire).

### Các tùy chọn CLI (`scone relay`)

| Tùy chọn | Mặc định | Vai trò |
|---|---|---|
| (không — GeoDNS luôn bật, bảng nhúng) | bật | Định tuyến tự động theo IP nguồn |
| `--geo-ip-table <path>` | — | **Bảng GeoIP cục bộ** (xem dưới). Chiếm ƯU TIÊN over mọi tùy chọn tĩnh `--geo-*` |
| `--geo-country <u16>` | — | Quốc gia client, ISO 3166-1 số (840=US…) — tĩnh |
| `--geo-continent <u16>` | — | Châu lục client, UN M49 (150=EU…); dẫn xuất từ quốc gia nếu bỏ qua — tĩnh |
| `--geo-asn <u32>` | — | AS client (64500…) — tĩnh, mới trong v2 |
| `--geo-region <u32>` | — | Vùng client (định danh mờ của bên xuất bản) — tĩnh, mới trong v2 |
| `--health-check` | tắt | Bật kiểm tra (failover sức khỏe) |
| `--no-health-check` | tắt | **Tắt tường minh** kiểm tra — THẮNG `--health-check` (xem ưu tiên) |
| `--health-ttl-secs <u64>` | 5 | TTL của một mục sức khỏe |
| `--health-port <u16>` | 443 | Cổng TCP cho kiểm tra (không dẫn xuất được từ bản ghi DNS) |

#### Bảng GeoIP cục bộ (`--geo-ip-table`)

Tệp VĂN BẢN cục bộ của các tiền tố, đọc **một lần** lúc khởi
động (tối đa 100.000 quy tắc, `MAX_GEOIP_RULES`). Một quy tắc
mỗi dòng, `#` = chú thích, dòng trống bỏ qua:

```text
# prefix/mask        country continent asn     region
10.0.0.0/8           840   3          .       .
2001:db8::/32        250   150        64500   11
192.0.2.0/24         158   9
```

- `prefix/mask`: IPv4 (`len ≤ 32`, chuyển sang dạng 16 byte,
  đuôi 0 — cùng quy ước wire) hay IPv6 (`len ≤ 128`). **Các
  bit host phải 0** (`10.1.2.3/8` bị từ chối);
- `country`: ISO 3166-1 **số** `u16`, bắt buộc;
- `continent`: UN M49 `u16`, tùy chọn (`.` hay bỏ = vắng,
  châu lục khi đó dẫn xuất được từ quốc gia);
- `asn`: `u32`, tùy chọn (`.` = vắng);
- `region`: `u32` mờ, tùy chọn (`.` = vắng).

Phân giải: **IP nguồn của truy vấn DNS** (theo-từng-client:
UDP `recv_from` / TCP `peer_addr`; không rõ → IP tham chiếu
của bộ phân giải) được so trong 16 byte; quy tắc có **tiền tố
khớp dài nhất** thắng (khi hòa, cái đầu theo thứ tự tệp);
không khớp → vị trí rỗng (World).

**Nghiêm ngặt**: tệp do vận hành viên cấu hình — một dòng vô
hiệu (tiền tố sai dạng, quốc gia không số, châu lục không rõ,
trường thừa…) là một **lỗi khởi động** mang **số dòng vi
phạm**; không bao giờ một dòng bị bỏ âm thầm, không bao giờ
một bảng cục bộ.

#### Ưu tiên tùy chọn

- **Locator**: `--geo-ip-table` **chiếm ưu tiên** over các
  tùy chọn tĩnh
  (`--geo-country`/`--geo-continent`/`--geo-asn`/`--geo-region`),
  chúng bị bỏ qua khi bảng được cung cấp. Không bảng, các
  tùy chọn tĩnh áp dụng.
- **Sức khỏe**: `--no-health-check` là một **cờ riêng thắng**
  over `--health-check` — tổ hợp
  `--health-check --no-health-check` để kiểm tra tắt, không
  mơ hồ thứ tự nhập. Không `--health-check`, kiểm tra tắt mặc
  định: **Không-clear = khả dụng**, câu trả lời geo đi ra kể
  cả khi đích đã chết. (Một `NoopChecker` — luôn Healthy —
  cũng tồn tại để dựng cấu trúc sức khỏe mà không để riêng
  endpoint nào: kiểm thử/hoang tưởng.)

Ví dụ:

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-ip-table /etc/scone/geoip.txt \
            --no-health-check
```

Ví dụ tĩnh v2:

```bash
scone relay --dns 127.0.0.1:5353 \
            --geo-country 250 --geo-asn 64500 --geo-region 11 \
            --health-check --health-port 443
```


## DoH / DoT (vận chuyển mã hóa) — qua frontend ngoài

Máy chủ DNS Scone nói **UDP và TCP thuần** (RFC 1035 / 7766)
— đó là phạm vi bản địa, hoàn chỉnh và đã kiểm thử. Các vận
chuyển mã hóa tới *client cuối* (DoH — DNS-over-HTTPS, RFC
8484; DoT — DNS-over-TLS, RFC 7858) được **ủy quyền cho một
frontend chuẩn**: Scone phục vụ dữ liệu, frontend phục vụ vận
chuyển.

Vì sao thiết kế này: một máy chủ DoH bản địa đòi hỏi một chồng
HTTPS đầy đủ trong nút (chứng chỉ, gia hạn, HTTP/2) — một bề
mặt vận hành lớn mà không lợi ích gì trước khi có người dùng
cuối phía trình duyệt. Các frontend sẵn có làm tốt hơn và lâu
hơn; nút giữ tối thiểu.

### Ví dụ: unbound làm DoH trước một relay Scone

`/etc/unbound/unbound.conf`:

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

Chứng chỉ (`scone.pem`/`scone.key`) lấy như mọi dịch vụ TLS —
ví dụ certbot/acme.sh trên tên miền của bộ phân giải. Client
DoH khi đó trỏ tới `https://dns.example.tld/dns-query`.

### Ví dụ: knot-resolver làm DoT

`/etc/knot-resolver/kresd.conf`:

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### Frontend thêm gì

- **Mã hóa kênh client→frontend** (đoạn thù địch: mạng của
  người dùng); đoạn frontend→relay giữ cục bộ (cùng máy hay
  mạng tin cậy).
- **Một cache thêm phía frontend** (cache unbound/knot cũng —
  vô hại, các TTL Scone vẫn là cận trên).
- **Một điểm cuối ổn định** (IPv4/IPv6, tên miền) trong khi
  các relay di chuyển.

### Frontend KHÔNG làm gì

- Nó không *xác minh* gì: mọi câu trả lời do relay phục vụ đã
  được xác minh đối với chuỗi trước khi cache (xem "Đường
  phân giải"). Frontend chuyển tiếp các câu trả lời đã-có-thẩm
  quyền.
- Nó không thay thế sự phi tập trung: một frontend DoH là một
  dịch vụ *được vận hành* — lựa chọn của vận hành viên bộ phân
  giải, không phải của giao thức. Mọi relay Scone giữ độc lập
  khả dụng qua UDP/TCP bản địa.

### Triển khai bản địa tương lai (được ghi nhận, chưa cam kết)

Một vận chuyển DoH bản địa (`--doh ADDR`) vẫn khả dĩ: định dạng
wire giống hệt (RFC 8484 tái sử dụng các thông điệp DNS nhị
phân), nó sẽ là một giải mã POST/GET + HTTP/2 sau một proxy
TLS (caddy/nginx lo chứng chỉ), nút tự thân giữ không-TLS.
Không có công việc đang tiến — nhu cầu sẽ đến cùng người dùng
cuối.

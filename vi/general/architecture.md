# Kiến trúc

## Trước hết, bằng lời giản dị

Năm khái niệm là đủ để hiểu mọi thứ còn lại:

1. **Một tên** — một địa chỉ trên internet, ví dụ `mysite.scone`.
2. **Cụm từ 24 từ của bạn** — danh tính của bạn. Nó là văn bằng
   chứng minh một tên là của bạn, và ký mọi thay đổi bạn thực hiện.
3. **Sổ ghi chép công cộng** (blockchain) — sổ cái chung, công khai
   về ai sở hữu cái gì. Không ai kiểm soát nó, không ai có thể viết
   lại nó.
4. **Đào (mining)** — cách bạn dành giữ một tên: máy tính của bạn làm
   một ít công việc, công bằng và như nhau cho mọi người. Không xếp
   hàng, không trung gian.
5. **Một relay** — một máy tính vận hành mạng bằng cách giữ một bản
   sao của cuốn sổ và trả lời các câu hỏi. Bất kỳ ai cũng có thể vận
   hành một relay; càng nhiều relay, mạng càng mạnh.

Phần còn lại của trang này đi sâu hơn — bạn có thể dừng ở đây và
vẫn đã hiểu dự án.

## Nguyên tắc dẫn đường

Scone tách biệt thẩm quyền khỏi tính sẵn sàng:

- **blockchain** là nguồn chân lý về quyền sở hữu và thứ tự các
  thay đổi;
- **DHT** là một hệ thống phân phối/sao chép, nó **không phải** một
  thẩm quyền;
- **lưu trữ cục bộ** giữ trạng thái của nút, các chỉ mục và cache.

Một nút phải coi mọi dữ liệu đến từ DHT là có khả năng độc hại:
chỉ được chấp nhận sau khi xác minh bằng mật mã (chữ ký của chủ sở
hữu, đối chiếu với ownership / sequence / record_hash của chuỗi).

## Tách biệt trách nhiệm

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

## Các crate và phụ thuộc

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

### Quyết định: `scone-core` phụ thuộc vào `scone-crypto`

`DomainId` phải tính được từ nền móng lên
(`BLAKE3-256(prefix || canonical name)`), nhưng mật mã cụ thể không
được làm ô nhiễm core. Giải pháp:

- `scone-crypto` là tầng thấp nhất: các hàm thuần túy tất định trên
  byte (cộng thêm RNG của hệ điều hành để sinh khóa), không kiểu
  core nào, không phụ thuộc mạng/hệ thống tệp/async → không thể có
  tính vòng tròn;
- `scone-core` chỉ dùng nó để băm;
- Chữ ký và khóa Ed25519 sống trong `scone-crypto` và thao tác trên
  byte (payload chuẩn do `scone-protocol` tạo ra).

### Vai trò của từng crate

**scone-core** — tên (`TldName`, `DomainName`), định danh
(`DomainId`, `OwnerId`, `PublicKeyRef`), dữ liệu DNS (`DnsRecord`,
`SignedDnsRecord`), giao dịch (`RegisterDomain`, `UpdateDomain`).
Tất định và khả chuyển. Bị cấm trong crate này: libp2p, redb,
tokio/async, mạng, tệp, hệ điều hành, máy chủ DNS, logic
blockchain.

**scone-crypto** — nguyên thủy mật mã. Hiện nay: `hash256`
(BLAKE3), chữ ký và khóa Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, xác thực batch nghiêm ngặt), phong bì
AEAD của các nhóm bản ghi (XChaCha20-Poly1305). Dự kiến: cây
Merkle. Quy tắc: không thuật toán tự chế, chỉ dùng các phụ thuộc đã
được kiểm toán.

**scone-protocol** — định dạng nhị phân của giao thức: mã hóa/giải
mã chuẩn (varint LEB128 tối giản, định danh trần 32 byte) của các
giao dịch, bản ghi DNS, khối (`Block`/`BlockHeader` với một trường
đồng thuận mờ) và thông điệp P2P, với giới hạn kích thước được kiểm
tra trước khi cấp phát, từ chối các dạng không chuẩn, và phiên bản
hóa (`PROTOCOL_VERSION`). Không có logic đồng thuận, mạng hay lưu
trữ. Tài liệu tham chiếu định dạng: `/docs/en/technical/protocol.md`.

**scone-blockchain** — logic blockchain và trạng thái chuẩn: định
danh giao dịch (`TxId`), cây Merkle có thứ tự,
`BlockHash`, genesis tất định, xác thực khối và giao dịch, quy tắc
áp dụng trạng thái (`DomainState`/`ChainState`, trong bộ nhớ, truy
cập trực tiếp theo `DomainId`), phát hiện fork và lựa chọn nhánh
(`fork_choice`: sàn tính cuối + chuỗi dài nhất + phá vỡ hòa bằng
hash thấp nhất), tính cuối của checkpoint, và trừu tượng hóa đồng
thuận (`trait Consensus`, mặc định là triển khai dễ dãi). PoW đăng
ký (đào/xác minh, độ khó theo từng mạng) sống trong `scone-core`.
Bị cấm trong crate này: mạng, DHT, lưu trữ (redb), máy chủ DNS.
Tham khảo: `/docs/en/technical/blockchain.md`.

**scone-storage** — lưu trữ cục bộ (đã bàn giao). Backend cụ thể là
`redb` (`NodeStore`), ẩn sau một trait tối thiểu để blockchain/DHT/DNS
không bao giờ ghép trực tiếp với redb; một backend khác (KV phân
tán) có thể được cắm vào. Bao gồm backend trạng thái KV phân trang,
nhật ký khối, snapshot đã xác minh (manifest có băm + các trang) và
đường dẫn khôi phục/phát lại khi khởi động lại. Tham khảo:
`/docs/en/technical/storage.md`.

**scone-network** — relay p2p của nút: swarm libp2p/QUIC, chuyển
tiếp giao dịch và khối (xác minh cục bộ có hệ thống), mempool có
giới hạn, đồng bộ khối có giới hạn (128 khối/yêu cầu), DHT Kademlia
cho `SignedDnsRecord`, máy chủ DNS UDP+TCP (GeoDNS luôn bật — bảng
GeoIP nhúng, chuyển tiếp đệ quy), vòng lặp checkpoint/heartbeat
của các anchor (CLI nói p2p QUIC trực tiếp với relay, không còn
RPC cục bộ nữa). Khả năng kết nối và khám phá: AutoNAT v2 (phán
quyết kết nối được/đứng sau NAT), circuit-relay v2 + DCUtR cho các
nút NATed, mDNS (khám phá LAN không cần cấu hình, BẬT mặc định trên
devnet) và UPnP (ánh xạ cổng QUIC bởi router, tự nguyện). Cấu hình
qua tệp `.env` (xem `.env.example`).
Tham khảo: `/docs/en/technical/relay.md`.

**scone-keystore** — lưu trữ mã hóa của các khóa ký Ed25519 (các
tệp `.sconekey`: Argon2id + XChaCha20-Poly1305). Crate này sở hữu
hệ thống tệp để `scone-crypto` giữ được sự thuần túy. Tham khảo:
`/docs/en/technical/keystore.md`.

**scone-topology** — động cơ THUẦN TÚY cho tối ưu hóa cấu trúc liên
kết p2p cục bộ: không mạng, không hệ thống tệp, không async. Nó
không biết libp2p lẫn swarm: `scone-network` cung cấp cho nó các
quan sát (RTT ping, kết nối) và nhận về các quyết định quản lý hàng
xóm.

**scone-genesis** — tệp nhị phân công cụ dùng một lần (kickstarter)
cho việc chuẩn bị genesis mainnet: manifest JSON có phiên bản, các
phân bổ, khối tất định.

**scone** — tệp nhị phân/CLI. Chỉ phụ thuộc vào một crate khi thực
sự dùng nó (hiện nay: scone-network cho relay, scone-blockchain,
scone-storage, scone-keystore, scone-protocol, scone-core,
scone-crypto, scone-topology).

## Nút mạng

Một nút Scone kết hợp:

1. một blockchain (xác thực đồng thuận, trạng thái sở hữu);
2. định tuyến DHT (xuất bản/truy xuất `SignedDnsRecord`);
3. lưu trữ cục bộ (trạng thái, chỉ mục, cache);
4. một bộ phân giải / máy chủ DNS ở phía trước.

### Khởi động một nút mới (bootstrap trạng thái, P0.3/P0.4)

Một nút mới KHÔNG phát lại genesis: nó khởi động từ một trạng thái
đã hoàn tất được xác minh.

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

Hai hồ sơ nút:

- **nút lưu trữ (archive)** (cờ vận hành viên): giữ mọi khối lịch
  sử, phục vụ chúng cho mạng, khởi động qua các đường dẫn hiện có;
- **nút trạng thái-đầy-đủ (full-state)**: giữ toàn bộ trạng thái
  tại H và phần đuôi, nhưng không giữ lịch sử dưới H — nó xác thực
  các khối tương lai và phục vụ trạng thái, nó không phục vụ lịch
  sử.

Đường dẫn này là TỰ NGUYỆN: nhập vào một snapshot đã xác minh là
một thao tác tường minh của vận hành viên
(`import_verified_snapshot`), không bao giờ tự động — hành vi mặc
định của một nút hiện có không thay đổi. Chi tiết chuẩn:
`/docs/en/technical/storage.md`, các mục "VERIFIED state snapshot"
và "State bootstrap".

## Luồng phân giải DNS

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

Snapshot DNS (tồn tại + hết hạn của mọi tên miền còn sống, được xuất
bản sau mỗi khối được áp dụng) bảo đảm rằng một quyết định tồn tại
không bao giờ đi qua vòng chạy của relay: core chỉ được chạm đến
cho các lần lấy bản ghi thực sự.

## Khả năng tiến hóa

- preimage có phiên bản (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): thay đổi một quy tắc dẫn xuất tạo ra một
  không gian tên mới thay vì phá vỡ cái hiện có;
- `RecordData` mở rộng được qua các biến thể + `Unknown { type_code, data }`
  cho các kiểu chưa biết tại thời điểm đọc;
- `PROTOCOL_VERSION` (`scone-protocol`) cho các thay đổi gãy định
  dạng wire;
- chuỗi chỉ lưu các băm: dữ liệu lớn ở lại trong DHT.

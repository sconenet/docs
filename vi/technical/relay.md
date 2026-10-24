# Relay mạng (M4) — kiến trúc, các luồng và điều khiển CLI

> Đặc tả chuẩn của relay mạng Scone (`scone-network`).
> Bất kỳ khác biệt tài liệu/mã = bug (sửa nguồn chân lý).

## Tổng quan

**Relay** là nút P2P đầy đủ của Scone: nó duy trì blockchain chuẩn
(RAM + `NodeStore` redb), chuyển tiếp giao dịch và khối, tạo
khối (chế độ devnet), phục vụ giao thức đồng bộ, xuất bản và
phân giải các `SignedDnsRecord` trong một DHT Kademlia, và phơi
bày **giao thức p2p** (`/scone/reqres/1`) mà CLI truy vấn trực
tiếp (thống nhất RPC→p2p: kênh TCP JSON đã biến mất).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  ephemeral port    │  ├─ identify  (address discovery)      │
  by default) ◄─────┤  ├─ ping      (liveness)               │
                    │  ├─ kad       (record DHT)             │
                    │  └─ reqres    (scone-prot. messages)   │
                    │                                        │
                    │  Relay core (single task)              │
                    │  ├─ Blockchain (full validation)       │
                    │  ├─ Mempool (bounded, dedup by TxId)   │
                    │  ├─ BlockBuilder (devnet production)   │
                    │  ├─ Fork parking (bounded, 2048)       │
                    │  └─ Sync driver (batched catch-up)     │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, same messages)      │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` là **crate async duy nhất** trong
workspace: mọi thứ dưới nó (crypto → core → protocol →
blockchain → storage) giữ đồng bộ và thuần túy.

## Nhận kết nối (P1.5)

Các giới hạn cứng ở tầng swarm (hành vi `connection_limits` của
libp2p, mục đầu của `SconeBehaviour`):

- `MAX_TOTAL_CONNECTIONS = 128` — nắp trên các kết nối thiết lập
  (vào + ra) mỗi relay; phần vào vượt bị TỪ CHỐI, không bao giờ
  xếp hàng;
- `MAX_PER_PEER_CONNECTIONS = 4` — một peer duy nhất không thể
  độc quyền hạn ngạch tổng bằng cách mở nhiều kết nối;
- `MAX_PENDING_INCOMING = 32` — canh slowloris: các nửa-kết nối
  chưa xác thực không bao giờ chờ vô hạn.

Các giới hạn này bổ sung các giới hạn theo-thông-điệp hiện có
(`MAX_MESSAGE_LEN` 1 MiB) và theo-ngân-sách (đồng bộ `max ≤ 128`
khối, DNS TCP 64 kết nối/512 B/5 s). Kẻ tấn công lũ lụt kết nối
nhận các lời từ chối, không phải sự tăng trưởng bộ nhớ.

## Khám phá peer

Ba cơ chế bổ trợ, theo thứ tự ưu tiên này:

1. **Seed tường minh** (`--bootstrap`, multiaddr đầy đủ) — luôn
   ưu tiên; cung cấp một danh sách không rỗng THAY THẾ mọi thứ
   còn lại. `--no-builtin-seeds` rỗng hóa các seed nhúng cho
   triển khai 100% riêng tư.
2. **Seed nhúng** (`builtin_seeds(network)` trong
   `scone-network/src/config.rs`) — đóng băng tại mỗi bản phát
   hành nhị phân chính thức, theo từng mạng. Một relay không cấu
   hình gia nhập mạng một mình. (Rỗng trên testnet chừng nào
   testnet công cộng chưa được triển khai.)
3. **Thư mục học được** (`<data-dir>/peers.json`) — mọi địa chỉ
   học qua `identify` được giữ (lần thấy cuối; nắp **2000 mục**
   — địa chỉ, không phải kết nối, ~400 KiB RAM tối đa — và một
   **mốc tuổi 90 ngày**: một mục lần cuối thấy hơn 90 ngày trước
   bị gỡ lúc ghi và bỏ qua lúc nạp, giữ 2000 khe đầy các peer
   hợp lý thay vì hóa thạch trước-churn mà đóng góp duy nhất
   sẽ là một timeout quay số lúc khởi động) và lưu ~1 lần/phút
   tại nhịp sản xuất. Lúc khởi động không seed không bootstrap,
   thư mục được tái lập: relay tìm lại các peer thường lệ của
   mình không cần cấu hình. Nó là một cache: tệp thiếu/hỏng =
   thư mục rỗng, không bao giờ lỗi.
4. **UPnP** (`--upnp`, tự nguyện) — ánh xạ cổng lắng nghe QUIC
   bởi router gia đình: nút trở nên reachable từ bên ngoài
   KHÔNG cần circuit relay. Khi `NewExternalAddr`, địa chỉ ngoài
   được đăng ký làm địa chỉ ngoài thật của swarm
   (`add_external_address` — thấy bởi identify/autonat, một ứng
   viên transport-relay). Không cổng IGD (`GatewayNotFound`) hay
   cổng không-định-tuyến được (`NonRoutableGateway`): một dòng
   nhật ký info đơn giản, ánh xạ bị bỏ qua — các trường hợp bình
   thường trên LAN gia đình. Ánh xạ hết hạn (gia hạn thất bại)
   là một cảnh báo.

UPnP giữ **tắt mặc định** (`enable_upnp` = `false` trong
`config.rs`): một triển khai công cộng không nói chuyện với
router nào khi chưa có sự đồng ý. **Khám phá mDNS đã bị gỡ bỏ**
(remove-mdns-discovery, 2026-10-22): peering đến độc quyền từ
bootstrap tường minh, các seed dựng sẵn và Kademlia — cờ cũ
`--mdns`/`SCONE_MDNS` vẫn được chấp nhận mà không có hiệu lực
(tương thích script, đối xử giống như `--rpc`).

### Tối ưu cấu trúc liên kết cục bộ (luôn bật)

Relay chủ động tối ưu hàng xóm của mình dùng động cơ thuần túy
`scone-topology` (xem tài liệu crate). **Luôn bật**; chỉnh duy
nhất được phơi bày là một **hồ sơ** (`--topology-profile`, env
`SCONE_TOPOLOGY_PROFILE`, mặc định `default`):

| Hồ sơ | min / mục tiêu / max | đánh giá | cho ai |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 s | Raspberry Pi, hộp gia đình, VPS 1-vCPU |
| `small` | 4 / 8 / 12 | 90 s | máy chủ nhỏ, NAS gia đình |
| `default` | 8 / 16 / 32 | 60 s | máy chủ chuẩn (mặc định) |
| `high` | 16 / 32 / 64 | 45 s | nút chủ quyền, xương sống |
| `very-high` | 32 / 64 / 128 | 30 s | các hub sáng lập |
| `devtest` | 2 / 4 / 6 | 5 s | **CHỈ PHÒNG THÍ NGHIỆM/KIỂM THỬ** (bất ổn theo thiết kế) |

```
scone relay --topology-profile tiny
```

Các tham số bảo vệ (trễ 0,15, hồi lạnh 600 s, hoãn 120 s, đo 15
s) giống nhau mọi nơi: chúng canh sự ổn định và không có lý do
scale theo kích thước nút. Một tên lạ là một lỗi khởi động có
kiểu.

Một cấu hình không nhất quán (`min > mục tiêu > max`, ngưỡng
ngoài `[0,1]` — gồm `NaN` —, khoảng không) là một **lỗi khởi
động có kiểu**, không bao giờ hành vi âm thầm.

**Góc nhìn (`TopologyView`)** — được nuôi bởi các sự kiện swarm:

- `ping` thành công → quan sát RTT (cửa sổ trung vị 8 mẫu);
- `ping` thất bại → hình phạt độ tin cậy;
- `identify` nhận → địa chỉ (≤ 4/peer) + giao thức loan báo
  (bitmask kad/reqres/identify/relay);
- kết nối thiết lập → trạng thái kết nối, hướng
  (vào/ra), tuổi, circuit-relay phát hiện;
- kết nối đóng → hình phạt + bộ đếm churn CHỈ KHI không được
  yêu cầu (một trục xuất tự nguyện không được tính);
- thất bại quay số ra → hình phạt độ tin cậy.

Góc nhìn bị chặn (500 peer; vượt mức đó, chỉ các peer kết nối
được ghi nhớ).

**`kad_required` từ các kbucket** — trước mỗi đánh giá, relay đi
qua bảng định tuyến Kademlia: một peer được bảo vệ nếu nó
`Connected` trong một bucket, hay `Disconnected` trong một
bucket BÃO HÒA (k = 20). Một peer được bảo vệ KHÔNG BAO GIỜ
được chọn để trục xuất.

**Vòng tối ưu** — tại khoảng đánh giá của hồ sơ (60 s dưới
`default`; xem bảng trên):

1. tính lại `kad_required` (kbucket);
2. đánh giá động cơ (trễ: thời kỳ hoãn 120 s, mức cải thiện điểm
   tối thiểu +15%, hồi lạnh trục xuất 600 s);
3. quyết định áp dụng: `AddCandidate` → quay số ứng viên;
   `Replace` → đóng peer bị trục xuất (MỘT LẦN — tiệm tiến,
   không lặp) rồi quay dụng sự thay thế;
4. một snapshot **chỉ số** được ghi nhật ký (mức INFO): các số
   đếm đã-biết/đã-kết-nối/vào/ra/circuit/được-bảo-vệ, RTT trung
   bình/trung vị, bucket IP riêng biệt, nồng độ tối đa, churn
   (thay thế/ngắt kết nối).

Điểm hàng xóm KHÔNG BAO GIỜ là một thước đo tin cậy (xem tài
liệu `scone-topology`): không quyết định đồng thuận nào dẫn xuất
từ nó.

### Khả năng kết nối (AutoNAT) và relay vận chuyển (circuit-relay v2)

Mọi relay chạy AutoNAT (client + server): nút nhờ các peer của
mình thử quay số NGƯỢC lại nó; phán quyết (`reachable`: có =
reachable từ bên ngoài, không = sau NAT, không rõ = chưa thử)
được làm mới mỗi 5 phút và phơi bày trong `NodeStatus` (trường
`nat_reachable`, wire varint 0/1/2) — CLI hiển thị nó
(`scone status`). Địa chỉ riêng tư được tính
(`only_global_ips = false`): một mạng lưới LAN thử tính
reachable LAN của mình.

Một relay reachable bật circuit-relay v2 **hop** (nó có thể
chuyển tiếp lưu lượng cho các peer NATed) và **DCUtR** (đục lỗ:
hai nút nối qua circuit thử một kết nối trực tiếp và giải
phóng relay). Một relay bị coi là NATed tự động xin một đặt
chỗ từ peer reachable đầu tiên biết đến và lắng nghe qua nó
(địa chỉ `/p2p/<relay>/p2p-circuit`).

Quy tắc sản xuất: một nút NATed qua relay không nên trở thành
người produce khối (độ trễ/tính sẵn sàng suy giảm) — quy tắc
PoS (ghế = tên miền còn sống) chưa cấm nó; sẽ được ghi nhận
như một yêu cầu vận hành trước testnet công cộng.

## Vận chuyển và các giao thức libp2p

| Giao thức | Tên | Vai trò |
|---|---|---|
| QUIC (quic-v1) | — | Vận chuyển chính (udp) |
| identify | `/scone/id/1` | Khám phá địa chỉ peer (lấp bảng Kademlia) |
| ping | chuẩn libp2p | Sống còn |
| Kademlia | `/scone/kad/1` | DHT: khóa = `DomainId` (32 B), giá trị = mã hóa chuẩn của `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | các thông điệp ứng dụng scone-protocol |

### Khung ứng dụng (`/scone/reqres/1`)

Mỗi khung là `len: u32 BE || byte` trong đó byte là mã hóa chuẩn
của một `Message` từ `scone-protocol` (codec hiện có — không
định dạng thứ hai). Chiều dài bị chặn bởi
`limits::MAX_MESSAGE_LEN` **trước** mọi lần đọc hay cấp phát:
một peer loan báo khung lớn hơn bị từ chối mà không tốn bộ nhớ.

Các thông điệp trao đổi (một mỗi vòng yêu cầu/phản hồi):

| Thông điệp | Chiều | Trả lời |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (phiên bản cục bộ) |
| `Ping` / `Pong` | sống còn | `Pong` |
| `GetBlock { hash }` | truy vấn | `Block` (cửa sổ RAM) hay `Pong` nếu không rõ |
| `GetBlocks { start, max }` | đồng bộ | **một** `Block` (chiều cao `start`), hay `Pong` nếu không có |
| `Block (b)` | chuyển tiếp khối | `Pong` (ack) |
| `Transaction (tx)` | chuyển tiếp tx | `Pong` (ack) |
| `GetRecord { domain_id }` | DHT cục bộ | `Record` (cache cục bộ) hay `Pong` |
| `Record (r)` | xuất bản | `Pong` nếu nhận, ngược lại lỗi |

Quy tắc: **đúng một trả lời mỗi yêu cầu**; các thông điệp xác
nhận là `Pong`. Trả lời đồng bộ mang một khối mỗi lần — bắt kịp
lặp phía bên yêu cầu (xem "Đồng bộ"). Điều này bị chặn một cách
cố ý: không bao giờ hơn một khối đang bay mỗi peer.

## Hành vi relay

### Khởi động

0. phân giải mạng (`--network testnet|mainnet|devnet`, mặc định
   testnet trong khi dự án đang phát triển — M8b). Thư mục dữ
   liệu là **theo từng mạng** (`~/.scone/<network>/`): một kho
   khác rỗng giữ chuỗi từ mạng khác bị từ chối (`Corrupted`);
1. mở/tạo kho redb (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: đỉnh O(1) + trạng thái tên
   miền phân trang (cửa sổ RAM: genesis + đỉnh);
3. dựng swarm (danh tính Ed25519 phù-du theo phiên — lưu danh
   tính mạng là một quyết định mở);
4. dựng danh sách khởi động (xem
   [Khám phá peer](#khám-pháy-peer)): `--bootstrap` tường minh
   (phải kết thúc bằng `/p2p/<peer-id>`) HOẶC seed nhúng của
   mạng HOẶC thư mục học được (`peers.json`);
5. quay số mỗi multiaddr và đăng ký nó vào bảng Kademlia
   (một lần quay số chết bị bỏ qua — những cái khác dựng nên
   mạng).

### Nhận một giao dịch (P2P hay nộp qua CLI `submit tx`)

1. giải mã bị chặn nghiêm ngặt (`MAX_MESSAGE_LEN`);
2. xác thực mật mã đầy đủ (`validate_transaction`: ràng buộc
   owner/khóa, `verify_strict` trên payload chuẩn);
3. **chống phát lại TRƯỚC** (rẻ, độc lập trạng thái, tất định),
   rồi tiền kiểm tra trạng thái (gương rẻ của các quy tắc
   `push_block` — PoW, vốn đắt, chỉ được xác minh lúc push):
   một `RenewDomain` phát lại trả lời `TxReplay`, không phải
   lỗi PoW nói dối (proof của nó trói hạn mà lần áp dụng đầu
   đã vượt qua). Sự tách mạng được thực thi ở bước 2: payload
   chữ ký tiêm mạng của RELAY, một tx ký nơi khác thất bại với
   `InvalidSignature` (wire 2026-09: hết trường `network`);
   `RegisterDomain` → TLD của tên đăng ký
   (nếu không `UnknownTld`, D1/M7c) **và mở** (nếu không
   `TldClosed`, M8b) rồi tên tự do; `UpdateDomain` → tên miền
   biết, sequence đúng `hiện_tại + 1`, chủ; `RegisterTld` → TLD
   tự do; `TransferTld`/`RevokeTld`/`SetTldOpen` → TLD biết +
   người ký là chủ; `AssignDomain` → chủ TLD + tên tự do;
   `RenewTld` → TLD riêng tư biết + chủ, hạn cố định dẫn xuất
   lại, tiền kiểm tra cửa sổ hoãn; `RenewDomain` → tên miền
   biết + chủ, tiền kiểm tra PoW gia hạn (trói vào hạn hiện
   tại) tại chiều cao kế;
4. chèn mempool (dung lượng cố định, khử trùng theo `TxId` —
   nộp lại một tx đã trong pool là no-op);
5. phát tới các peer (trừ người gửi).

### Nhận một khối (P2P, đồng bộ, hay tự tạo)

1. `push_block`: xác thực đầy đủ hiện có (cha = đỉnh chuẩn,
   chiều cao, gốc Merkle tính lại, mọi giao dịch xác thực rồi
   áp dụng — nguyên tử mỗi khối);
2. nếu chấp nhận: relay đọc kết quả áp dụng
   (`push_block_with_gc` → băm + các tên miền bị gỡ bởi GC tất
   định) và lưu bền vững qua `store_block_with_removals`
   (nối delta nguyên tử: khối + đỉnh + trạng thái thay đổi —
   upsert VÀ gỡ: tên miền hết hạn, TLD thu hồi). Một lần khởi
   động lại do đó không bao giờ hồi sinh một đăng ký hết hạn;
3. đối chiếu mempool (các tx được bao hàm bị gỡ);
4. phát tới các peer (trừ người gửi).

Một khối vô hiệu bị từ chối với một lỗi có kiểu; relay tiếp tục
chạy. Không panic trên dữ liệu mạng. Các lần áp dụng nặng
(Argon2 + commit redb) đi qua `tokio::task::block_in_place`
(trình lái đồng bộ, chấp nhận p2p, sản xuất): runtime chuyển các
tác vụ sẵn sàng sang worker khác — áp dụng một khối không còn
làm đình trệ dịch vụ DNS hay các chỉ số.

### Snapshot DNS (xuất bản)

Sau MỌI khối được áp dụng (chấp nhận p2p — reorg kể cả —, trình
lái đồng bộ, sản xuất) VÀ lúc khởi động (một lần khởi động lại
không được NXDOMAIN các tên sống đến khối kế), relay xuất bản
**snapshot DNS có thẩm quyền** lên nguồn `watch` do bề mặt DNS
đọc (đặc tả đầy đủ trong `dns.md`):

- mở rộng đỉnh (đường áp đảo): **delta O(chạm tới)**,
  copy-on-write — các giao dịch của khối nêu tên các tên miền
  bị chạm, cộng các lần gỡ GC tất định
  (`AppliedBlock.gc_removed_domains`, đánh giá lúc push);
- reorg / khởi động: **dựng lại đầy đủ** (một lần đi
  `ChainState::dns_facts` phân trang).

### Tạo khối (M5: người produce có chữ ký)

Cứ mỗi `produce_interval` giây (2 s mặc định): nếu mempool
không rỗng, một khối được lắp bởi `BlockBuilder`
(`timestamp = bây giờ`) **ký bởi một người produce được ủy
quyền** (payload `SCONE-BLOCK-V1` trong `header.consensus`,
xem `/docs/en/technical/blockchain.md`) và theo đúng đường chấp
nhận của một khối nhận được. Một relay không tx không tạo gì.

Khóa sản xuất: khóa anchor (`--anchor-key`) khi relay được vũ
trang một cái — một anchor là một người produce được ủy quyền —
ngược lại khóa devnet cố định, hợp lệ chừng nào tập người
produce còn rỗng (bootstrap mở: genesis). Ngay khi một tên
miền/TLD còn sống tồn tại, một khối ký bởi khóa devnet bị từ
chối (`InvalidProducer`).

### Vòng anchor (tính cuối, M5)

Với `--anchor-key` (tệp khóa `.sconekey`, cụm mật khẩu qua
`--anchor-passphrase-env`, mặc định `SCONE_ANCHOR_PASS` — mặc
định thư viện `SCONE_ANCHOR_PASSPHRASE` của `scone-network`
không bao giờ đạt tới qua tệp nhị phân, clap luôn ghi đè nó):

- **xoay vòng proposer**: chính xác MỘT thành viên ủy ban được
  chỉ định tự đề xuất mỗi epoch (`committee[epoch % size]`,
  tất định); các thành viên khác không bao giờ ký đỉnh của chính
  mình — nội dung của một checkpoint phụ thuộc vào đỉnh của
  proposer, và các tự đề xuất không có cổng đã chia rẽ epoch
  thành các nội dung đối đầu mà mỗi guard từ chối đúng đắn như
  equivocation (testnet, 2026-10-21);
- sau mỗi khối được chấp nhận/tạo, nếu nút này là thành viên của
  ủy ban hiện hành **và** ủy ban đạt
  `MIN_FINALITY_COMMITTEE_SIZE`, relay ký checkpoint đề xuất
  qua `SignerGuard` của nó (`signer-state.dat` trong thư mục dữ
  liệu — một khóa = một phiếu/epoch, chống-crash fail-closed)
  và gossip nó;
- **vote-on-receipt**: một đề xuất mà nội dung hợp lệ trên chuỗi
  CỦA CHÚNG TA (khối được tham chiếu có mặt, ở cùng băm) và mang
  một chữ ký ủy ban cũng nhận được chữ ký CỦA CHÚNG TA — không có
  nó, mỗi anchor chỉ ký nội dung đỉnh của chính mình và quorum
  chưa bao giờ tập hợp;
- các chữ ký nhận được (`Message::Checkpoint`) được tích lũy
  theo băm `CheckpointData` (chặn 256 mục); tại
  `quorum_for(kích thước ủy ban)` chữ ký thành viên trên CÙNG
  dữ liệu → `accept_checkpoint` + phát checkpoint hoàn tất;
- **lưu bền tính cuối**: cửa sổ checkpoint và nền đóng băng
  được ghi vào store tại mỗi lần hoàn tất và được nạp lại lúc
  khởi động. Các checkpoint là trạng thái GOSSIP (không bao giờ
  được khối mang theo) — một lần khởi động lại không có sự lưu
  bền này bắt đầu từ một nền rỗng và không bao giờ có thể nối lại
  một cửa sổ do peer phục vụ (các anchor epoch dưới cửa sổ của
  mạng đã mất);
- **đuổi kịp tính cuối**: một relay có nền rỗng (khởi động của
  một nút mới) yêu cầu `GetCheckpoints` từ một peer mỗi nhịp cho
  đến khi nền được phục hồi;
- một relay không khóa anchor giữ **bị động**: nó tích lũy, hoàn
  tất, và chuyển tiếp, nhưng không bao giờ ký;
- `GetCheckpoints` → cửa sổ `checkpoint_window()` được phục vụ;
- mọi lỗi vòng được ghi và nuốt — không bao giờ nghiêm trọng;
- trạng thái (`GetStatus`) phơi bày `checkpoints` và
  `finalized_epoch`.

### Đỗ fork và reorg (sửa thiểu số-dính)

Một khối chị em cùng chiều cao, hay một khối mà cha không phải
đỉnh hiện tại (`ParentNotTip`/`UnknownParent`), KHÔNG BAO GIỜ
bị mất: nó được **đỗ** (bản đồ chặn, `PARKED_CAP = 2048` khối,
~KiB mỗi cái — RAM trường-hợp-xấu bị chặn; một bãi đỗ đầy âm
thầm bỏ qua người mới đến). Các băm đã được đưa cho
`accept_block` được khử trùng (`seen_blocks`): một khối thua
gossip lại không được đánh giá lại.

Tại MỌI nhịp `Produce`, `reconsider_parked`:

1. xả tuyến tính các khối đỗ mà cha đã trở thành đỉnh;
2. nếu một nhánh đỗ mạnh hơn đỉnh hiện tại (quy tắc chọn-fork
   KHÔNG ĐỔI: sàn tính cuối, chuỗi dài nhất, băm thấp nhất),
   nhận nuôi — hậu tố bị truất chính nó được đỗ (CẢ HAI phía
   của fork giữ khả năng đánh giá).

Điều này gỡ kẹt một nút đã nhận phía thua của một fork cùng
chiều cao: khối thắng, được đưa một lần rồi từ chối, được đánh
giá lại định kỳ thay vì bị quên mãi mãi (phát hiện chaos-30:
hai khối chị em cách nhau 15 ms).

**Lưu bền vững reorg**: nhận nuôi nhánh khác không thể đi qua
phép nối đơn điệu (`NonMonotonicHeight`). Relay áp dụng
`rewind_to_height(điểm rẽ)` rồi `persist_state_snapshot` (ghi
lại trạng thái sống từ bản xuất của chuỗi) rồi nối lại đơn điệu
hậu tố thắng (xem `storage.md`). Một thất bại lưu bền vững được
ghi và suy giảm (tự chữa lúc khởi động qua phát lại) — không
bao giờ nghiêm trọng.

## Đồng bộ (bắt kịp)

Trên mỗi kết nối tới một peer, relay yêu cầu
`GetBlocks(start = chiều cao cục bộ + 1)`. Mỗi `Block` nhận được
hoàn tất đúng chiều cao kỳ vọng kích hoạt yêu cầu chiều cao kế,
cho đến cạn (`Pong` = hết) hay `MAX_SYNC_ROUNDS` (10.000) lô —
một biên tuyệt đối ngay cả chống một peer nói dối. Các khối nhận
đi qua xác thực đầy đủ: một peer độc hại không thể khiến cái gì
vô hiệu được chấp nhận.

### DHT bản ghi

- **put** (luồng API/web — kênh hex CLI đã gỡ): giải mã bị chặn
  nghiêm ngặt (`MAX_DHT_CACHE_ENTRY`), xác minh đối với trạng
  thái on-chain (chủ, sequence, `record_hash` tính lại — xem
  "Xác minh bản ghi"), rồi `kad.put_record` (lưu cục bộ +
  sao chép tới các peer gần nhất, `Quorum::Majority` của hệ số
  sao chép) và cache redb bền vững.
- **get** (`domain resolve`): `kad.get_record`; trình chờ bị
  trói vào `QueryId` và `DomainId` yêu cầu — một trả lời chỉ
  giải truy vấn khớp, và khóa bản ghi phải đúng tên miền yêu cầu
  (kiểm tra **trước** xác minh on-chain). Kết quả sau đó được
  xác minh đối với chuỗi trước khi trả về. Nắp yêu cầu đồng
  thời: `MAX_DHT_WAITERS` (256); vượt, lỗi tức thì. Một lần
  phân giải xác minh thành công được ghi vào cache bền vững
  (M5): các lần đọc cục bộ (`domain_info`) khi đó phục vụ nó
  không cần truy vấn DHT mới.
- Kademlia chạy chế độ máy chủ, không TTL bản ghi (chuỗi chi
  phối tính hợp lệ, không phải thời gian).

### Xác minh bản ghi (quy tắc "chuỗi là thẩm quyền")

Một `SignedDnsRecord` phân giải hay nhận được chỉ được
chấp nhận/trả về nếu:

1. tên miền đăng ký on-chain;
2. `record.owner` = chủ on-chain hiện tại;
3. `record.sequence` = sequence on-chain hiện tại;
4. `record_hash(record.record)` (BLAKE3 tính lại) =
   `record_hash` cam kết on-chain.

Ngược lại: từ chối âm thầm phía P2P, lỗi tường minh phía CLI.
Chữ ký Ed25519 của bản ghi được **xác minh lúc nhận** bởi relay
(`ResolveLocal`: khóa công khai của chủ biết on-chain, chữ ký
phải mở khóa — một bản ghi ký bởi khóa khác bị từ chối,
`record rejected`); nó cũng được vận chuyển cho các bên xác
minh tương lai — cam kết on-chain vẫn là kiểm soát toàn vẹn
chính.

## Chỉ số Prometheus

Một điểm cuối HTTP (định dạng văn bản Prometheus 0.0.4, phơi bày
tự viết — không phụ thuộc mới) phục vụ bởi một tác vụ riêng:

- **Tắt mặc định**: không có máy chủ HTTP nào chạy khi thiếu `--metrics-addr` (metrics là opt-in).
- `--metrics-addr ADDR:PORT`: gắn endpoint vào địa chỉ đó (vd. `127.0.0.1:9101` cục bộ, `0.0.0.0:9101` để mở ra ngoài).
- `--metrics-token TOKEN`: chuyển sang chế độ **riêng tư** —
  `Authorization: Bearer *** required` (so sánh thời-gian-
  không-đổi, `401` ngược lại; token không bao giờ xuất hiện
  trong nhật ký hay trong phơi bày).
- Một thất bại buộc **tắt điểm cuối** (cảnh báo không nghiêm
  trọng) — không bao giờ nút.

Buộc hiệu lực được loan báo trên stdout (hợp đồng đọc-được-bởi-
máy, như `p2p:`): một dòng `metrics: <addr>` ngay khi lắng
nghe.

**19 bộ đếm** (`counter`, đơn điệu — đường nóng:
`fetch_add(Relaxed)` trên một `Arc<Metrics>` chung, không bao
giờ một đầu vào đồng thuận):

| Nhóm | Chuỗi |
|---|---|
| Chuỗi | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Mạng peer | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 gauge** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`, `scone_addr_book_entries`)
— năm cái đầu làm mới mỗi 15 s qua kênh `RelayCommand` (cùng
đường với CLI `status` — nguồn chân lý duy nhất);
`scone_addr_book_entries` được duy trì trực tiếp bởi registry
(cập nhật ở mỗi lần chèn sổ địa chỉ — nguồn chân lý).

Kiểm thử E2E: `metrics_endpoint_public_and_private` (200 +
phơi bày, 401 không/kèm token sai, 200 với Bearer).

## Điều khiển CLI (thống nhất RPC→p2p)

Máy chủ RPC TCP JSON đã bị **gỡ**: CLI (và cầu axum của
registrar) nói giao thức p2p **trực tiếp** — cùng wire
(`/scone/reqres/1`), cùng các trình xử lý phía relay, không kênh
song song. Mỗi lệnh mở một kết nối QUIC tới multiaddr của nút
(`--peer`), trao đổi MỘT `Message`, đóng.

Các thông điệp điều khiển do relay phục vụ:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (hiển thị bởi `scone status`
  — `fresh_operators` = các vận hành viên có heartbeat còn
  mới, RFC proof-of-service);
- `LookupName { name }` → `NameState` (trạng thái on-chain, như
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (tên miền + TLD còn
  sống của một `OwnerId`);
- `Transaction (tx)` → nộp kèm gossip miễn phí (relay phát lại
  tới peer);
- `GetRecord { domain_id }` / `Record (r)` — phân giải và xuất
  bản DHT (như `scone domain resolve` phía CLI);
- từ chối ứng dụng → `Message::Error(msg)` (đạt client như một
  thông điệp dễ đọc, không phải EOF).

Client dựng một swarm libp2p tối thiểu (request-response +
identify), hết hạn gọi 30 s (vòng 20 s). Mặc định `--peer`:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (nút dev cục
bộ; multiaddr đầy đủ với peer id thật được relay loan báo trên
stdout qua dòng `p2p:`).

Kênh `RelayCommand` nội bộ sống sót cho bề mặt DNS nhúng
(`ResolveLocal`, một vòng trong tiến trình, không bao giờ một
trình chờ DHT — xem `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR]
            [--bootstrap MULTIADDR]... [--upnp]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]...
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # express single record
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer defaults to the local relay (SCONE_PEER env overrides);
# --identity defaults to the single keystore identity when there
# is exactly one. Removed (stubs suggest replacements): show,
# submit, lookup, owned.
```

Mọi lệnh điều khiển nói QUIC trực tiếp tới multiaddr của relay
(`--peer`, mặc định nút cục bộ — thống nhất RPC→p2p, hết kênh
TCP JSON). Relay vắng → thông điệp rõ "p2p node unreachable — is
a relay running?", thoát khác không. Cờ `--rpc` được giữ nhưng
**lỗi thời và bị bỏ qua** (máy chủ RPC đã gỡ; chỉ tương thích
script).

### Cấu hình môi trường (`.env`)

Lúc khởi động, `scone` nạp (qua `dotenvy`, trước khi phân tích
clap) một tệp `.env` từ thư mục hiện tại: tệp thiếu = no-op âm
thầm, không bao giờ lỗi. Các giá trị nạp thành **chuẩn cơ sở
môi trường** của các tùy chọn khai báo một biến
(`env = "…"` trong `cli.rs`); **một cờ CLI tường minh luôn
thắng** `.env`, và `.env` thắng các mặc định dựng sẵn:

```text
explicit CLI flag  >  .env variable  >  built-in default
```

Tệp `.env.example` ở thư mục gốc kho ghi tài liệu mọi biến (sao
chép sang `.env` và chỉnh; `.env` bị **gitignore** — không bao
giờ commit một `.env` thật). Các biến:

| Biến | Cờ soi | Ngữ nghĩa |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (mặc định), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | thư mục dữ liệu (mặc định `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | multiaddr lắng nghe P2P (mặc định: mọi giao diện, cổng phù du) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | seed bootstrap, **lặp được** (một dòng mỗi giá trị — dotenvy tích lũy); thay các seed dựng sẵn |
| `SCONE_GENESIS` | `--genesis` | tệp genesis (nút sáng lập) |
| `SCONE_MDNS` | `--mdns` | (lỗi thời — khám phá mDNS đã bị gỡ bỏ; vẫn chấp nhận, bỏ qua) |
| `SCONE_UPNP` | `--upnp` | ánh xạ cổng UPnP (`true` = bật) |
| `SCONE_DNS_ADDR` | `--dns` | địa chỉ buộc máy chủ DNS UDP |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | fallback đệ quy, **lặp được** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | nắp mục cache DNS |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | nắp bộ nhớ cache DNS (ví dụ `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | nắp truy vấn DNS đồng thời |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | token bearer điểm cuối chỉ số |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | hồ sơ cấu trúc liên kết (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; mặc định `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | tệp khóa anchor (checkpoint + sản xuất) |

`SCONE_ANCHOR_PASS` KHÔNG PHẢI tùy chọn của một biến: nó là TÊN
MẶC ĐỊNH của **biến giữ cụm mật khẩu anchor**
(`--anchor-passphrase-env`, mặc định `SCONE_ANCHOR_PASS` ở cấp
CLI) — relay đọc cụm mật khẩu từ biến mà cờ này nêu tên.

**Anchor tự động (mặc định BẬT)**: không `--anchor-key`, một
tệp khóa anchor được sinh/lưu tự động trong thư mục dữ liệu
(`anchor.sconekey`, cụm mật khẩu `SCONE_ANCHOR_PASS` nếu định
nghĩa, ngược lại sinh ngẫu nhiên và ghi vào `anchor.pass`) —
relay trở thành người produce ngay khi khóa của nó sở hữu tên
miền/TLD còn sống (đủ điều kiện PoS không bao giờ miễn phí).
`--no-anchor-auto` giữ relay hoàn toàn bị động. Biến env mặc
định của `--anchor-passphrase-env` là `SCONE_ANCHOR_PASS`.

`--network`: mặc định theo BUILD (mainnet trên các tệp nhị phân
bật-mainnet, testnet ngược lại); `devnet` là mạng cục bộ đặt-
lại-được (genesis `scone-devnet` riêng, PoW gần-bằng-không
0..=2 bit — xóa `~/.scone/devnet/` và khởi động lại đặt lại
chuỗi). Đặt genesis, các độ khó PoW, và thư mục dữ liệu mặc
định (`~/.scone/<network>/`); một tx từ mạng khác bị từ chối có
kiểu (`WrongNetwork`).

Bề mặt hex ngoại tuyến (các lệnh con `tx` và `submit`) đã biến
mất (remove-tx-surface-v2). CLI phủ các đường người dùng
(`domain register`, `domain renew`, `domains`) VÀ gia đình M8b
như các lệnh hạng nhất: `tld open`, `tld transfer`, `tld
revoke`, `domain assign`, `domain transfer`. Các lời tuyên bố
`register` đào PoW của mình ở độ khó mạng (mili-giây trên
devnet).

**Không có tùy chọn geo hay sức khỏe nào tồn tại**: geozone
**được build áp đặt** (bảng DB-IP Lite nhúng — ngữ nghĩa đầy đủ
trong
[`dns.md`](dns.md#geodns-geographic-routing)); định tuyến địa lý
**luôn bật** mà không cần tùy chọn nào, và bộ phân giải không
thực hiện kiểm tra sức khỏe nào — nó phục vụ tập bản ghi có chữ
ký nguyên trạng (khả dụng của endpoint là việc của các vận hành
viên endpoint). Bất kỳ cờ `--geo-*`/`--health-*` nào cũng là một
lời từ chối clap dứt khoát.

### `domain register|update` (M5) — giao dịch có chữ ký trong một lệnh

Các lệnh này dây dựng → ký (keystore) → nộp → (xác nhận)
trong một lời gọi duy nhất.

- **register**: dựng một `RegisterDomain` có ký (timestamp =
  bây giờ, proof rỗng trên devnet), nộp nó, chờ xác nhận
  on-chain (sequence ≥ 0, hết hạn 30 s). Một tên đã đăng ký bị
  từ chối phía client VÀ phía relay.
- **update**: tệp bản ghi là nguồn chân lý duy nhất. Sequence
  (on-chain + 1) và `record_hash` (BLAKE3 chuẩn của các bản ghi
  tệp) được **dẫn xuất**, không bao giờ nhập: người ký và các
  bên xác minh không thể lệch nhau. Sau xác nhận UpdateDomain,
  `SignedDnsRecord` (bản ghi từ tệp + chủ + chữ ký Ed25519 trên
  mã hóa chuẩn) được xuất bản lên DHT qua `put_record` (tái
  xác minh đối với chuỗi). Một tên miền chưa đăng ký, một danh
  tính không-phải-chủ, hay lỗi tệp thất bại sạch, không chạm
  chuỗi.

#### Tệp bản ghi

Văn bản, một bản ghi mỗi dòng, `#` = chú thích, dòng trống bỏ
qua; mã hóa chuẩn đã sắp làm thứ tự tệp không có hiệu lực trên
băm cam kết. Tệp là bộ HOÀN CHỈNH (một lần cập nhật thay thế,
không trộn). Các kiểu:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spaces normalized)
```

Lỗi có kiểu: kiểu lạ, IP/tên vô hiệu (kèm số dòng), bộ rỗng,
trùng, > `MAX_RECORDS_PER_SET`.

## `--genesis` (sáng lập)

`scone relay --genesis genesis.bin` — **chỉ nút sáng lập**
(chiến dịch kickstarter; tệp do tệp nhị phân `scone-genesis`
tạo). Quy tắc chuẩn:

1. Tệp bị từ chối **trước khi đọc** nếu vượt 1 MiB, rồi giải mã
   nghiêm ngặt (`decode_complete`: không byte thừa). Băm khối 0
   được **tính lại**, không bao giờ đọc từ tệp.
2. Xác thực ngữ nghĩa: chiều cao 0, `consensus` bắt đầu bằng id
   mạng của mạng yêu cầu (`--network`), không giao dịch,
   manifest phân bổ chuẩn. Mọi vi phạm = lỗi có kiểu lúc khởi
   động (không bao giờ panic, daemon không khởi động).
3. Thư mục dữ liệu **trinh nữ**: chuỗi sinh trên genesis được
   phân bổ — các phân bổ được áp dụng trong RAM **trước mọi lần
   ghi đĩa**, rồi lưu (trạng thái gieo + khối genesis dưới một
   khóa meta). Khối đầu tiên tạo ra do đó chỉ có thể mở rộng
   genesis được phân bổ.
4. Thư mục dữ liệu **đã sản xuất**: genesis khi sinh của kho là
   `prev_hash` của khối 1 (tự-chứng-thực). Băm tệp bằng = OK
   âm thầm (tùy chọn **dư thừa sau lần khởi động đầu**); khác =
   từ chối có kiểu — một nút gia nhập mạng qua đồng bộ, không
   bao giờ qua `--genesis`.
5. Khởi động lại **không** tùy chọn: một thư mục dữ liệu sinh-
   bởi-sáng-lập hồi phục genesis phân bổ của nó qua khóa meta
   (gắn lại RAM ở mọi lần khởi động).

Ghim băm genesis hằng số (chống lệch-mạng) là một dòng công
việc riêng.

### Manifest tệp nhị phân `scone-genesis` (kickstarter)

Tệp nhị phân `scone-genesis` (crate `scone-genesis`) chuẩn bị
manifest `genesis-manifest.json` (định dạng `ALLOC-V1`) rồi tạo
`genesis.bin`. Quy tắc chuẩn:

- `scone-genesis new --network <mạng>` tạo manifest rỗng;
  `add`/`remove`/`list` quản lý phân bổ, `build` tạo
  `genesis.bin` (+ băm), `verify` xác thực lại nó.
- `add domain <tên>` đòi **TLD của tên miền đã được phân bổ
  trong manifest** (từ chối tức thì ngược lại — chuỗi sẽ đã từ
  chối nó lúc áp genesis); `valid_until` của nó là **tự động**:
  `born_at + 1 năm` (default_validity của manifest), không bao
  giờ nhập tay.
- `add tld <tên>`: **mở mặc định** (không ai, tự phục vụ, sàn
  1 năm, chủ = 0 trên wire — kind `0x03`); `--private` = riêng
  tư (chủ, hạn genesis `born_at + TLD_TERM_SECS` — 3 năm,
  vòng đời chuẩn: `RenewTld`, hoãn 60 ngày).
- Sắp chuẩn theo tên (nội dung bằng → cùng khối → cùng băm,
  bất kể thứ tự chèn); trùng bị từ chối.

## Giới hạn và bảo đảm bộ nhớ

| Tài nguyên | Biên |
|---|---|
| Khung P2P | `MAX_MESSAGE_LEN` (1 MiB) trước khi phân tích |
| Mempool | 4096 tx (từ chối có kiểu khi vượt); `MAX_PENDING_PER_DOMAIN = 4` tx treo mỗi tên miền |
| Hoãn mempool | `MAX_GRACE_ATTEMPTS = 10` lần chèn lại một tx bị xả vì lý do không-tất-định (≈ 7 phút ở 40 s/vòng), rồi trục xuất |
| Đỗ fork | `PARKED_CAP = 2048` khối cạnh tranh |
| Checkpoint tích lũy | `MAX_PENDING_CHECKPOINTS = 256` (trục xuất: băm dữ liệu nhỏ nhất) |
| Trình chờ DHT đồng thời | `MAX_DHT_WAITERS` (256); tra cứu DHT chặn 30 s (`DHT_LOOKUP_TIMEOUT`) |
| Khối mỗi trả lời đồng bộ | 1 mỗi vòng; ≤ 10.000 lô mỗi peer (`MAX_SYNC_ROUNDS`); biên giao thức `MAX_BLOCKS_PER_REQUEST = 128` trên yêu cầu |
| Bản ghi DHT | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Cache DHT bền vững (đĩa) | biên LRU kép: `DHT_CACHE_MAX_ENTRIES` (100.000 mục) VÀ `DHT_CACHE_MAX_BYTES` (2 GiB) — cái nào tới trước trục xuất theo thứ tự khóa; trục xuất GC trong CÙNG giao dịch với khối (một tên miền bị GC của chuỗi giết không bao giờ để lại mục mồ côi) |
| Cache âm tính DHT | `DHT_NEGATIVE_TTL_SECS` (30 s), nắp FIFO `DHT_NEGATIVE_CAP` (4.096) |
| Chuỗi trong RAM | genesis + cửa sổ đỉnh (lịch sử phục vụ từ kho) |

Quy tắc chống vòng (H2): một giao dịch chỉ được phát lại nếu nó
**thực sự được chèn** vào mempool (thấy lần đầu); một bản trùng
(đã trong pool hay đã trong chuỗi) không bao giờ được chuyển
tiếp lại. Một khối đã-chuẩn được phát hiện và bỏ qua trước
`push_block` (không lưu lại, không phát lại).

Các quyết định mở: danh tính mạng bền vững của relay, gossip
phát (hiện tại: unicast request-response tới mỗi peer biết).
Đã giải quyết từ lúc viết ban đầu: chọn fork thật (đỗ +
reconsider_parked, xem trên), đồng thuận PoW (người produce có
chữ ký + ủy ban anchor), bộ phân giải DNS (bề mặt `--dns`, xem
dns.md), kênh điều khiển (thống nhất RPC→p2p: hết TCP JSON).

## Graceful shutdown (SIGTERM / SIGINT) — tắtGraceful shutdown êm dịu

Daemon `scone relay` (tiền cảnh) bắt SIGTERM và SIGINT qua
`tokio::signal`: khi có tín hiệu, future `Relay::run` bị hủy
(`tokio::select!` trong tệp nhị phân), thả Swarm libp2p và đóng
các socket QUIC. Runtime tokio sau đó dừng với một lần tắt bị
chặn (≤ 2 s) thả các tác vụ DNS tách rời (UDP/TCP) và đóng
socket của chúng — mục tiêu là một `kill` không để lại socket
UDP mồ côi (trường hợp thấy trên testnet 60). Tắt theo tín hiệu
ghi `relay shutting down (signal)` (INFO, stderr) và thoát với
mã **0**: tắt được yêu cầu không phải thất bại. Nếu cài đặt
trình xử lý thất bại, hành vi mặc định của tín hiệu áp dụng
(suy giảm, không bao giờ panic).


## Vận chuyển Tor (vấn đề mở)

QUIC/UDP không xuyên qua proxy SOCKS thuần, nên các relay
không thể chạy trên Tor một cách tầm thường hôm nay. Một vận
chuyển Tor libp2p bản địa (relay địa chỉ-onion) là đường theo
dõi phía trước — xem hướng dẫn kháng kiểm duyệt của website cho
trạng thái hiện tại và cầu CNAME `.tor` (tên dễ đọc cho hidden
service, hỗ trợ hôm nay).

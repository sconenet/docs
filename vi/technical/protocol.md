# Giao thức

Tài liệu tham chiếu cho giao thức Scone: định dạng nhị phân, mã
hóa chuẩn, giao dịch, bản ghi DNS, khối, thông điệp P2P, giới hạn
và tương thích phiên bản. Triển khai tham chiếu là crate
[`scone-protocol`](../../../crates/scone-protocol).

Các mục đánh dấu **(tạm thời)** chưa phải là bản cuối.

## Phiên bản giao thức

- hằng số: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- được mang bởi `Hello.version` (handshake) và
  `BlockHeader.version`;
- quy tắc tương thích:
  - phiên bản nhận được **cao hơn** phiên bản cục bộ → từ chối
    (`ProtocolError::UnsupportedVersion`); nút không đoán định
    dạng của một phiên bản nó không biết;
  - phiên bản `0` → vô hiệu;
  - chỉ có một bộ định dạng: các byte phiên bản giao dịch
    (`0x01`) và phiên bản khối phải khớp với giá trị cục bộ
    **chính xác**;
- không tách lớn/nhỏ ở giai đoạn này: một số nguyên duy nhất,
  tăng khi có bất kỳ thay đổi gãy nào của định dạng wire.

## Nguyên tắc định dạng nhị phân

- nhị phân, không bao giờ JSON cho dữ liệu giao thức;
- tất định và chuẩn: hai đối tượng bằng nhau về logic tạo ra
  **đúng cùng các byte**;
- phân tích tuần tự, không cấp phát khi có thể (mượn byte);
- mọi đầu vào mạng là không tin cậy: chiều dài và bộ đếm được
  kiểm tra **trước** bất kỳ cấp phát nào; giải mã không bao giờ
  panic.

### Thứ tự byte

- mọi số nguyên là varint LEB128 (nhóm 7 bit, trọng số thấp nhất
  đứng trước): thứ tự byte của máy không áp dụng;
- địa chỉ IPv4/IPv6 và định danh 32 byte: mảng byte thô, không
  tiền tố chiều dài;
- không có số nguyên kích thước cố định rộng trong định dạng:
  một quy tắc mã hóa số nguyên duy nhất.

### Varint (LEB128 tối giản)

Mỗi byte mang 7 bit dữ liệu; bit cao là 1 trên mọi byte trừ byte
cuối. Một `u64` vừa trong 1 đến 10 byte.

| giá trị | byte (hex) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16.384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

Việc mã hóa phải là **tối giản**: các dạng quá khổ bị từ chối
khi giải mã (`InvalidVarint`). Đây là điều kiện cần để một giá
trị chỉ có đúng một chuỗi byte hợp lệ (mã hóa chuẩn). Các lần
từ chối: `80 00` (không tối giản), `ff×9 02` (tràn u64),
`ff×10` (quá dài).

### Chuỗi và byte

- `bytes`: chiều dài varint + byte thô;
- `str`: như trên, kèm giải mã UTF-8 bắt buộc;
- mọi chiều dài đều bị nắp (xem [Giới hạn](#giới-hạn)) và được
  kiểm tra trước khi đọc.

### Định danh

| Kiểu | Biểu diễn wire |
|---|---|
| `DomainId` | 32 byte thô |
| `TldId` | 32 byte thô |
| `OwnerId` | 32 byte thô |
| `PublicKeyRef` | 32 byte thô |
| `RecordHash` | 32 byte thô |
| `BlockHash` | 32 byte thô |
| `MerkleRoot` | 32 byte thô |
| `DomainName` | `str` (≤ 253 byte, được xác thực lại bởi `scone-core`) |

Các phép dẫn xuất (`BLAKE3-256` + tiền tố `SCONE-DOMAIN-V1`,
v.v.) vẫn do `scone-core` định nghĩa (xem
`/docs/en/general/naming.md`).

## Giao dịch

```text
Transaction = disc u8 || version u8 || payload
```

| Discriminant | Kiểu |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x2A` | `RegisterTldPow` |
| `0x52` | `UpdateDomain` |
| `0x57` | `RenewTldPow` |
| `0x84` | `RegisterDomainPow` |
| `0x93` | `RegisterTld` |
| `0xC3` | `RenewDomainPow` |

Mỗi discriminant là một hằng số mờ tùy ý (không có trình tự logic
nào; `0x00` không bao giờ hợp lệ) — bảng chuẩn đầy đủ trong
`/docs/en/technical/transactions.md`.

Byte `version` (giá trị hiện tại `0x01`) đi sau discriminant:
mọi giá trị khác bị từ chối tường minh
(`UnsupportedVersion(value)`). Hầu hết các giao dịch đều được ký
(Ed25519) — NGOẠI TRỪ bốn dạng pow-to-owner không chữ ký
(`0x2A`, `0x57`, `0x84`, `0xC3`): bằng chứng đã đào CHÍNH LÀ sự
ủy quyền, và challenge của nó gắn khóa của người thụ hưởng và
mạng. Đặc tả đầy đủ, payload ký và quy tắc xác thực trong
**`/docs/en/technical/transactions.md`** (chuẩn).

Kể từ thay đổi wire tháng 9-2026, **giao dịch không còn mang
trường `network`**: `network_id` (`str` ≤ 16) được tiêm vào
**payload ký** (các dạng có chữ ký) hoặc **challenge PoW**
(các dạng pow-to-owner) — một tx tạo cho mạng này không bao giờ
chuyển sang mạng khác (xem `/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` mang **tên chuẩn dưới dạng văn bản rõ**, theo sau
bởi `DomainId` của nó — tính nhất quán `domain_id == DomainId(name)`
được kiểm tra lại lúc giải mã (tính lại, không bao giờ tin).
`proof` mang PoW đăng ký (20 byte: nonce + epoch + độ khó — muối
epoch `epoch = height/144` của khối áp dụng, xác minh NGHIÊM
NGẶT lúc áp dụng đối với hằng số mạng và epoch của khối; từ chối
có kiểu `ProofEpochMismatch` khi lệch — M8b, xem
`transactions.md`). `public_key` là khóa Ed25519 của người ký
(đúng 32 byte); `owner` phải là dẫn xuất BLAKE3 của khóa đó (tính
lại, không bao giờ tin); `signature` đúng 64 byte.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Tuyên bố một TLD (registry tách khỏi registry tên miền, M7): TLD
được **mang dưới dạng văn bản rõ** như tên của một tên miền (M8b
— challenge PoW dẫn xuất từ tên), và `tld_id` là dẫn xuất
`SCONE-TLD-V1` của tên đó, tách rời khỏi mọi `DomainId` ngay từ
cấu trúc. Quy tắc trạng thái (M8b): TLD tự do + không thuộc
danh mục dự trữ ICANN (`IcannTldReserved` — các TLD gốc ICANN
không ai tuyên bố được) + PoW ở độ khó **theo chiều dài** của
nhãn (đường cong `pow_length` — xem `blockchain.md`); sau khi
tuyên bố là đóng (chỉ-assign).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Bất biến kiểm tra lại lúc giải mã: `sequence > 0` và `owner` ==
dẫn xuất của `public_key`. Không có `timestamp`: thứ tự đi qua
`sequence`. Giao dịch chỉ mang các tham chiếu gọn — toàn bộ nội
dung DNS ở lại trong DHT (xem dưới).

## Bản ghi DNS (phía DHT)

Các bản ghi DNS đầy đủ không bao giờ vào blockchain: chúng được
trao đổi qua DHT (các thông điệp `GetRecord`/`Record`) và cam
kết on-chain chỉ bằng `RecordHash` của chúng.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

Các mã kiểu là **các mã DNS thật** (IANA). TTL (giây, varint)
đi sau mã kiểu và phủ toàn bộ bản ghi: biên `MIN_TTL = 1` ..
`MAX_TTL = 2.592.000` (30 ngày; mặc định
`DEFAULT_TTL = 86.400`).

| Mã | Kiểu | Payload |
|---|---|---|
| 1 | A | 4 byte |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 byte |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (xem dưới) |
| khác | Unknown | code.v + `bytes` ≤ 4096 |

Ví dụ: `A 192.0.2.1` (TTL 86.400) → `01 80 a3 05 c0 00 02 01`
(kiểu, TTL varint, địa chỉ).

Một `Unknown` mang mã đã biết bị từ chối lúc mã hóa (giải mã
không bao giờ tạo ra được).

#### GEOROUTE (GeoDNS)

Kiểu riêng của Scone (ngoài dải IANA). Một quy tắc định tuyến
địa lý:

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

**Quy tắc chuẩn**: các endpoint của một scope cho trước được mã
hóa **đã sắp xếp** (priority tăng dần, rồi weight giảm dần, rồi
target tăng dần); việc sắp xếp được áp dụng lúc mã hóa và một
wire chưa sắp xếp bị từ chối lúc giải mã (`NonCanonical`), như
với các bộ bản ghi. Các endpoint trùng lặp hoàn toàn bị từ chối.

**Xác thực**: `priority > 0`, `weight ≤ 100` (0 = endpoint không
hoạt động), ≤ 32 endpoint mỗi scope (`GEO_MAX_ENDPOINTS`). Một
mã nước/châu lục không có trong bảng cục bộ
(`continent_of_country`) vẫn **được chấp nhận** ở mức wire:
bảng của bộ phân giải có thể mới hơn bảng của bên xuất bản. TTL
là của `RecordData` cha (không đổi).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Quy tắc chuẩn**: một bộ bản ghi là một tập hợp không có thứ
tự. Nó được mã hóa **sắp xếp theo thứ tự từ điển của mã hóa riêng
của từng bản ghi**, tăng nghiêm ngặt (trùng lặp bị từ chối).
Một wire chưa sắp xếp bị từ chối lúc giải mã (`NonCanonical`).
Thứ tự của `Vec` gốc không được vận chuyển.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

Băm chỉ phủ **nội dung bản ghi**; chữ ký của chủ sở hữu được xác
minh riêng trên cùng các byte chuẩn đó. *(Quyết định: cam kết
không phủ chữ ký — một phiên bản trước của tài liệu này nhắc đến
"SignedDnsRecord đầy đủ"; ghép băm với chữ ký không mang lại gì
và trói cam kết vào lược đồ chữ ký.)*

## Khối

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: phiên bản giao thức (các quy tắc trên);
- `height`: chiều cao (genesis = 0);
- `prev_hash`: băm của khối trước (toàn số 0 với khối
  genesis);
- `tx_root`: cam kết trên danh sách giao dịch (cây Merkle do
  `scone-blockchain` định nghĩa, xem
  `/docs/en/technical/blockchain.md`);
- `timestamp`: thông tin thứ tự (Unix, giây);
- `consensus`: payload của người produce (`SCONE-BLOCK-V1 ‖
  pk[32] ‖ sig[64]`, xem `blockchain.md`) — genesis của một
  mạng mang `network_id` thô, và một genesis phân bổ (kickstarter
  mainnet) có thể mang một manifest `ALLOC-V1` (≤ 60 KiB); bị
  chặn bởi `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Phân tích tuần tự; bộ đếm bị nắp **trước** khi cấp phát; mọi sự
cắt cụt bị từ chối. Khác với các bộ bản ghi, thứ tự giao dịch là
**có ý nghĩa** (thứ tự đồng thuận) và được bảo tồn chính xác.

`BlockHash` (băm của header chuẩn) và `tx_root` được tính bởi
`scone-blockchain` trên các mã hóa định nghĩa ở đây (công thức
chính xác trong `/docs/en/technical/blockchain.md`).

## Thông điệp P2P

```text
Message = disc u8 || payload
```

Vận chuyển (đóng khung, libp2p…) nằm ngoài phạm vi; một khung
vận chuyển không nên vượt `MAX_MESSAGE_LEN` (1 MiB).

| Disc | Thông điệp | Chiều | Payload | Giới hạn | Trả lời kỳ vọng |
|---|---|---|---|---|---|
| `0x01` | `Hello` | hai chiều | `version.v` | — | `Hello` |
| `0x02` | `Ping` | hai chiều | — | — | `Pong` |
| `0x03` | `Pong` | hai chiều | — | — | — |
| `0x04` | `GetBlock` | hai chiều | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | hai chiều | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | hai chiều | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | hai chiều | `Transaction` | — | — |
| `0x08` | `GetRecord` | hai chiều | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | hai chiều | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | hai chiều | `Checkpoint` | người ký ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | hai chiều | — | — | cửa sổ checkpoint |
| `0x0c` | `GetStatus` | hai chiều | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | hai chiều | trạng thái nút (mạng, đỉnh, chiều cao, NAT, tên miền, mempool, peer, tính cuối, vận hành viên còn mới) | — | — |
| `0x0e` | `LookupName` | hai chiều | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | hai chiều | trạng thái on-chain của tên + TLD | — | — |
| `0x10` | `GetOwnedNames` | hai chiều | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | hai chiều | tên miền + TLD còn sống của chủ | — | — |
| `0x12` | `Error` | trả lời | thông báo lỗi mức ứng dụng (`str`) | — | — |

### Checkpoint (M5)

Một checkpoint tổng hợp lên wire dưới dạng: thẻ `SCONE-CKPT-V1` ||
thân `CheckpointData` (116 B, LE — giống hệt phần cuối của
preimage ký: `epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32) recovery(4)`) ||
`count.v` || `pk[32] ‖ sig[64]` cho mỗi người ký, **sắp theo pk
tăng dần, không trùng lặp**, `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`). Giải mã nghiêm ngặt: cắt cụt, sai
thẻ, pk không chuẩn, trùng lặp hoặc chưa sắp = từ chối
(`None`), không bao giờ panic (đã fuzz thử hỏng).

`GetCheckpoints` yêu cầu cửa sổ tính cuối; trả lời là cửa sổ
`checkpoint_window()` (tối đa `CHECKPOINT_KEEP` checkpoint, cũ
đến mới) qua một hoặc nhiều thông điệp `Checkpoint`.

`TxId` nay được định nghĩa bởi `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, xem
`/docs/en/technical/blockchain.md`). Mempool và chống phát lại
cửa sổ (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) đã bàn giao
trong relay (M4); chỉ còn `GetTransaction` bị hoãn (định nghĩa
của nó phụ thuộc các quyết định đồng thuận còn lại).

## Tách blockchain / DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (full, signed)
sequence
record_hash
```

Chuỗi nói *ai* sở hữu và *trạng thái hiện tại là gì* (sequence +
record_hash). DHT cung cấp *nội dung*. Một bản ghi DHT được chấp
nhận khi và chỉ khi:

1. chữ ký của nó hợp lệ cho chủ đã đăng ký trên chuỗi;
2. `sequence` của nó nhất quán với (hoặc cao hơn) của chuỗi;
3. `RecordHash` tính lại trên mã hóa chuẩn của nó khớp với
   `record_hash` của chuỗi.

## Giới hạn

| Hằng số | Giá trị | Vai trò |
|---|---|---|
| `MAX_NAME_LEN` | 253 | chiều dài một tên |
| `MAX_TLD_LEN` | 63 | chiều dài một TLD |
| `MAX_TXT_LEN` | 4096 | chuỗi TXT |
| `MAX_PROOF_LEN` | 256 | bằng chứng đăng ký |
| `MAX_SIGNATURE_LEN` | 128 | chữ ký (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | bản ghi mỗi bộ |
| `MAX_DHT_RECORD_LEN` | 64 KiB | bản ghi DNS có ký mã hóa (payload DHT: wire, lưu trữ, gossip) — nắp tổng kiểm tra lúc mã hóa VÀ trước mọi giải mã; cao hơn 15–30× mọi bộ thực tế (SPF+DKIM+DMARC ≈ 2–5 KiB), thấp hơn 16× nắp vận chuyển. |
| `GEO_MAX_ENDPOINTS` | 32 | endpoint mỗi scope GeoRoute |
| `MAX_UNKNOWN_DATA` | 4096 | dữ liệu kiểu không rõ |
| `MAX_TXS_PER_BLOCK` | 4096 | giao dịch mỗi khối |
| `MAX_CONSENSUS_LEN` | 64 KiB | payload đồng thuận của header (mở rộng cho manifest genesis `ALLOC-V1` — kickstarter mainnet) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | khối mỗi `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | khung vận chuyển (khuyến nghị) |

Mọi vượt quá → `LimitExceeded`, phát hiện trước khi cấp phát:
một peer độc hại không thể gây một cấp phát mất cân đối lúc giải
mã.

## Mã hóa chuẩn

Quy tắc nền tảng:

```text
canonical object → canonical binary encoding → hash / signature
```

- chỉ varint tối giản;
- các bộ bản ghi sắp xếp tăng nghiêm ngặt;
- một đối tượng bằng nhau về logic có đúng **một** mã hóa hợp
  lệ;
- giải mã từ chối các dạng không chuẩn (`NonCanonical`), nên
  không có nhiều mã hóa hợp lệ cho cùng một đối tượng.

## Xử lý lỗi

`ProtocolError` (`scone-protocol::error`):

| Biến thể | Ý nghĩa |
|---|---|
| `Truncated` | đầu vào kết thúc giữa chừng một giá trị |
| `InvalidVarint` | varint không tối giản, tràn hoặc quá dài |
| `IntegerOutOfRange` | số nguyên không vừa kiểu đích |
| `UnsupportedVersion` | phiên bản 0 hoặc cao hơn cục bộ |
| `UnknownDiscriminant` | discriminant giao dịch/thông điệp không rõ |
| `LimitExceeded` | chiều dài hoặc số lượng vượt một giới hạn |
| `InvalidUtf8` | chuỗi không phải UTF-8 |
| `TrailingBytes` | byte còn sót sau một giá trị hoàn chỉnh |
| `Validation` | vi phạm một bất biến của `scone-core` |
| `NonCanonical` | mã hóa hợp lệ từng trường nhưng không chuẩn |

Giải mã không bao giờ panic trên dữ liệu mạng.

## Chữ ký

- lược đồ: **Ed25519** (RFC 8032) qua `scone-crypto`
  (`ed25519-dalek`, xác minh `verify_strict` — chữ ký có thể uốn
  nắn và khóa yếu bị từ chối);
- **giao dịch chuỗi**: mọi `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` nhúng `public_key` (32 B) + `signature` (64 B)
  trên payload
  `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`;
  đặc tả chuẩn hoàn chỉnh: `/docs/en/technical/transactions.md`;
- **bản ghi DHT** (`SignedDnsRecord`): payload ký là mã hóa
  chuẩn của `DnsRecord`; việc xác minh bắt chéo
  `record + owner + signature` với danh tính đã đăng ký trong
  blockchain;
- quy tắc xuyên suốt: `owner` và mọi băm/cam kết đều được **tính
  lại**, không bao giờ lấy từ một giá trị được cung cấp.

## Quy tắc đồng thuận

Đã bàn giao (mô tả trong `/docs/en/technical/blockchain.md`):
người produce khối có chữ ký (payload `SCONE-BLOCK-V1`, ủy ban
được bầu bởi checkpoint), tính cuối bằng checkpoint tổng hợp
(`SCONE-CKPT-V1`, lựa chọn fork tại sàn tính cuối), PoW đăng ký
theo chiều dài tên. Các chủ đề còn lại: phí chống spam tinh tế,
xoay khóa chủ sở hữu.

# Giao dịch có chữ ký (định dạng v1)

Tài liệu tham chiếu **chuẩn** cho định dạng giao dịch có chữ ký.
Nó đặc tả các trường, thứ tự wire chính xác, kích thước, payload
ký và các quy tắc xác thực, để cho phép một triển khai độc lập
mà không cần đọc Rust. Triển khai tham chiếu là
[`scone-protocol`](../../../crates/scone-protocol) (mã hóa) và
[`scone-blockchain`](../../../crates/scone-blockchain) (xác
thực).

`PROTOCOL_VERSION` = **1**. Chỉ có một bộ định dạng: mọi byte
phiên bản khác `0x01` (trong một giao dịch) hay khác phiên bản
cục bộ (trong một khối) đều bị từ chối với `UnsupportedVersion`.

## Tổng quan

```text
Transaction = disc u8 || version u8 || payload
```

| Trường | Kích thước | Vai trò |
|---|---|---|
| `disc` | 1 B | kiểu giao dịch (bảng dưới) |
| `version` | 1 B | **đúng `0x01`** (phiên bản định dạng tx) |

**Giao dịch không còn mang trường `network` trên wire** (thay đổi
2026-09): `network_id` (`str` ≤ 16, ASCII `[a-z0-9-]` — ví dụ
`scone-testnet`, 13 byte) sống **chỉ trong payload ký** — người
ký tiêm nó ở đó như trường đầu tiên (xem [Payload ký](#payload-ký))
— và giữ tính theo-từng-mạng trong digest PoW. Một giao dịch
**được ký** cho một mạng do đó không bao giờ xác minh được trên
mạng khác: chữ ký của nó thất bại (`InvalidSignature`) đối với
payload được chuỗi tính lại, vốn tiêm mạng của chính nó (xem
`/docs/en/technical/blockchain.md` — mạng và genesis).

### Các discriminant (bảng chuẩn)

Mỗi discriminant là một **hằng số mờ cố định, tùy ý**: các giá
trị được chọn khác nhau một cách cố ý, để wire **không bao giờ**
gợi ý một thứ tự logic giữa các kiểu (không có trình tự
0x01/0x02/0x03…). `0x00` **không bao giờ** là một kiểu hợp lệ
(byte hỏng điển hình). Các kiểu tương lai sẽ lấy một giá trị tự
do chưa dùng nào đó, được ghi tài liệu một lần cho tất cả trong
bảng này.

| Discriminant | Kiểu | Vai trò |
|---|---|---|
| `0x21` | `RegisterDomain` | tuyên bố một tên miền |
| `0x52` | `UpdateDomain` | phiên bản mới của dữ liệu DNS một tên miền |
| `0x93` | `RegisterTld` | tuyên bố một tên miền cấp cao (M7) |
| `0x47` | `TransferTld` | chuyển nhượng một TLD cho chủ mới (M8a) |
| `0x6B` | `RevokeTld` | từ bỏ một TLD (M8a) |
| `0xB8` | `SetTldOpen` | mở **không thể đảo ngược** một TLD cho tự đăng ký (một chiều, tùy chọn của chủ; `open=false` là một từ chối có kiểu) |
| `0xD4` | `AssignDomain` | gán trực tiếp một tên miền bởi chủ của TLD (M8a) |
| `0x3C` | `RenewDomain` | gia hạn đăng ký một tên miền (M8a; mang tên chuẩn từ khi có tùy chọn chủ — một TLD mở bám theo hạn) |
| `0x8F` | `RenewTld` | gia hạn hạn 3 năm của một TLD RIÊNG TƯ (PoW bằng lúc tạo) |
| `0x15` | `SlashTx` | bằng chứng equivocation của anchor + cấm (M9) |
| `0xE2` | `TransferDomain` | chuyển nhượng một tên miền cho chủ mới (M8c) |
| `0x71` | `Heartbeat` | bằng chứng sống của một vận hành viên (RFC proof-of-service) |
| `0xA9` | `SetTldOperator` | ủy quyền vận hành một TLD riêng tư (RFC proof-of-service) |
| `0x5E` | `SetDomainOperator` | ủy quyền vận hành một tên miền (RFC proof-of-service) |

Mọi byte phiên bản khác `0x01` bị từ chối tường minh
(`UnsupportedVersion(giá trị nhận được)`): một định dạng không
rõ hay lỗi thời không bao giờ bị phân tích do vô ý.

## SLASH_TX (0x15, M9)

Bằng chứng on-chain rằng một anchor đã ký hai **checkpoint xung
đột ở cùng epoch** (equivocation). Bất kỳ ai cũng có thể nộp —
bằng chứng là mật mã và tự chứa:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` kiểm tra cấu trúc: cùng epoch, `signing_hash` khác
  nhau, hai chữ ký khác nhau, chữ ký của người báo trên payload
  chuẩn;
- quy tắc trạng thái (ghi nhật ký, đảo ngược được khi reorg): cả
  HAI chữ ký của bị cáo phải xác minh được đối với HAI băm ký
  khác nhau ở cùng epoch; bị cáo phải nằm trong tập hợp đủ điều
  kiện (nếu không `SlashOffenderNotInPool`); sau đó là **cấm** —
  khóa bị gỡ khỏi tập PoS một cách bền vững (danh sách đen
  trong trạng thái);
- trừng phạt tối thiểu: các tên miền/TLD của bị cáo KHÔNG bị
  tịch thu (quyết định đã ghi nhận của M9);
- cùng checkpoint hai lần = không phải equivocation (từ chối);
  khác epoch = từ chối; chữ ký giả mạo = từ chối.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `name` | `str` ≤ 253 | tên chuẩn của tên miền được tuyên bố, **mang dưới dạng văn bản rõ** (xem dưới) |
| `domain_id` | 32 byte thô | **phải** bằng `DomainId(name)` (tính lại, không bao giờ tin) |
| `owner` | 32 byte thô | **phải** là dẫn xuất của `public_key` (xem dưới) |
| `timestamp` | varint | thông tin thứ tự (Unix, giây) |
| `proof` | `bytes` ≤ 256 | payload PoW đăng ký (xem [dưới](#bằng-chứng-công-việc-đăng-ký-pow)) |
| `public_key` | 32 byte thô | khóa công khai Ed25519 của người ký (RFC 8032) |
| `signature` | **đúng 64 byte** | chữ ký Ed25519 của payload ký |

### Tên được mang bởi giao dịch

`RegisterDomain` vận chuyển tên chuẩn dưới dạng văn bản rõ, theo
sau bởi `DomainId` của nó. Tính nhất quán `domain_id == DomainId(name)`
được kiểm tra lại lúc mã hóa, giải mã và xác thực (phòng thủ
nhiều lớp): một `RegisterDomain` mà id không khớp tên mang theo
là vô hiệu. `DomainId` vẫn là danh tính đồng thuận (khóa chuỗi
và DHT); tên mang theo làm chuỗi tự-mô-tả và phủ toàn bộ lời
tuyên bố trong chữ ký.

## REGISTER_TLD (0x93)

Tuyên bố một tên miền cấp cao (registry TLD, M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `tld` | `str` ≤ 63 | tên chuẩn của TLD **mang dưới dạng văn bản rõ** (M8b: challenge PoW dẫn xuất từ tên, như `RegisterDomain` mang tên của mình) |
| `tld_id` | 32 byte thô | danh tính của TLD: `BLAKE3-256("SCONE-TLD-V1" || tld)` — **phải** là dẫn xuất của `tld` (tính lại) |
| `owner` | 32 byte thô | dẫn xuất của `public_key` |
| `timestamp` | varint | thông tin thứ tự (Unix, giây) |
| `proof` | `bytes` ≤ 256 | payload PoW đăng ký (xem [dưới](#bằng-chứng-công-việc-đăng-ký-pow)) |
| `open` | `0x00\|0x01` | sinh **mở**: TLD đáp trực tiếp vào `TldState::Open` (không ai cả, tự phục vụ + PoW, một PoW duy nhất ở mức `open_tld_pow_difficulty` — rẻ hơn riêng tư: tài nguyên chung, không ghế PoS mà cũng không độc quyền); sinh **riêng tư**: hành vi lịch sử (chủ sở hữu, ghế PoS, `tld_pow_difficulty`) |
| `public_key` | 32 byte thô | khóa công khai Ed25519 của người ký |
| `signature` | **đúng 64 byte** | chữ ký Ed25519 của payload ký |

Không gian `TldId` **tách rời** khỏi không gian `DomainId`
(tiền tố dẫn xuất khác nhau — xem `/docs/en/general/naming.md`):
đăng ký TLD `uip` (một tuyên bố chủ quyền trên không gian tên
`*.uip`) không bao giờ có thể chiếm dụng danh tính của một tên
miền, và ngược lại. Quy tắc trạng thái lúc áp dụng: TLD phải tự
do (registry tách khỏi registry tên miền).

## Gia đình M8a — các định dạng wire

Năm kiểu hoàn tất registry TLD và vòng đời tên miền. Định dạng
wire chuẩn từ M8a; **quy tắc trạng thái đầy đủ từ M8b** (xem
`/docs/en/technical/blockchain.md`): yêu cầu chủ TLD, PoW ở mọi
nơi lời tuyên bố là tự phục vụ, hết hạn và thời gian hoãn phía
tên miền. Không payload nào mang trường `network` trên wire
(nó chỉ sống trong payload ký).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `tld_id` | 32 byte thô | TLD được chuyển nhượng |
| `owner` | 32 byte thô | dẫn xuất của `public_key` (chủ hiện tại / người ký) |
| `new_owner` | 32 byte thô | danh tính người nhận (mờ: mọi giá trị đều hợp lệ về cú pháp; ràng buộc vào các khóa có thể chi tiêu nó là quy tắc trạng thái M8b) |
| `public_key` | 32 byte thô | khóa công khai Ed25519 của người ký |
| `signature` | **đúng 64 byte** | chữ ký Ed25519 của payload ký |

Không có timestamp hay sequence: một TLD có đúng một chủ tại một
thời điểm; lần chuyển được áp dụng đầu tiên thắng (chỉ thứ tự
chuỗi).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Từ bỏ tự nguyện một TLD (sau đó có thể tuyên bố lại bằng một
`RegisterTld` mới). Ký bởi chủ hiện tại; cố ý không có timestamp
hay sequence — một revoke là một lần duy nhất đối với trạng thái
hiện tại.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `open` | 1 byte | **nghiêm ngặt `0x00` hoặc `0x01`** (dạng chuẩn; mọi giá trị khác → lỗi giải mã, không bao giờ ép buộc) |
| `open_proof` | `proof` (varint chiều dài + byte) | trường **được ký**; bắt buộc và được xác minh khi `open = 0x01` (PoW mở, `open_tld_pow_difficulty` của mạng, challenge `SCONE-TLD-OPEN-V1 ‖ tld_id`); rỗng khi `open = 0x00` (đóng là miễn phí) |

Một TLD **đóng** = chỉ-assign (các tên miền chỉ được tạo bởi chủ
TLD qua `AssignDomain`); một TLD **mở** = bất kỳ ai cũng có thể
làm PoW đăng ký và tuyên bố một tên miền tự do
(`RegisterDomain`). **Mở** tốn thêm một bằng chứng công việc
(nhẹ hơn tuyên bố TLD: ~3 ngày trên mainnet so với ~1 tuần),
trả một lần bởi chủ trong `SetTldOpen` — bằng chứng đi trong
trường `open_proof` được ký và không thể bị thay thế sau khi
ký.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] public_key[32] signature[64]
```

Ảnh phản chiếu của `RegisterDomain`: tên chuẩn được mang dưới
dạng văn bản rõ và `domain_id` phải bằng `DomainId(name)` (tính
lại lúc giải mã, mã hóa và xác thực). Ký bởi **chủ của TLD của
tên** (kiểm tra lúc áp dụng, M8b); `assignee` trở thành chủ đầu
tiên của tên miền (mờ, như `new_owner`).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `name` | `str` ≤ 253 | tên chuẩn **mang dưới dạng văn bản rõ** (tùy chọn chủ: việc gia hạn trì hoãn cái chết của TLD mở bên dưới, chuỗi phải biết tên); `domain_id` phải bằng `DomainId(name)` (tính lại) |
| `valid_until` | varint | hạn đăng ký mới (Unix, giây) |
| `proof` | bị chặn | **ủy quyền gia hạn** (xem dưới) |

Ký bởi chủ hiện tại của tên miền; việc `valid_until` có thực sự
kéo dài hạn (và tối đa một hạn gia hạn) là quy tắc trạng thái
(M8b).

#### Ủy quyền gia hạn — PoW hay PoS tùy TLD

Chi phí của một lần gia hạn phụ thuộc bản chất của TLD bên dưới
(đánh giá lúc áp dụng, trên trạng thái khối):

- **TLD mở**: gia hạn tốn **cùng PoW như lúc tạo** — cùng
  challenge (`SCONE-DOMAIN-V1 ‖ canonical name`), cùng độ khó
  theo chiều dài nhãn và theo mạng, cùng cửa sổ hiệu lực
  epoch. Một tên miền không bao giờ bị "thu hoạch" miễn phí:
  mọi hạn sống đều được trả bằng công việc.
- **TLD riêng tư**: không gian tên là phần đặt cược PoS của chủ
  TLD. Giao dịch phải được **ký bởi chủ TLD** (hoặc vận hành
  viên được ủy quyền của họ) — chữ ký CHÍNH LÀ ủy quyền,
  trường `proof` phải rỗng (20 byte 0). Chủ TLD gia hạn các
  tên miền của không gian tên mình theo quy tắc riêng.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `tld` | `str` ≤ 63 | tên chuẩn của TLD; `tld_id` phải bằng `TldId(tld)` (tính lại) |
| `valid_until` | varint | phải ĐÚNG là `bây giờ + 3 năm` (hạn cố định — dẫn xuất lại lúc áp dụng, mọi giá trị khác bị từ chối) |
| `proof` | bị chặn | PoW gia hạn (xem dưới) |

Ký bởi chủ **riêng tư** hiện tại của TLD. Quy tắc áp dụng:

- chỉ áp dụng cho một TLD RIÊNG TƯ (một TLD mở sống qua các tên
  miền của nó — từ chối có kiểu `TldIsOpen`);
- hợp lệ trong hạn VÀ xuyên qua **thời gian hoãn 60 ngày** theo
  sau hạn; vượt quá, TLD chết (GC) và sự từ chối là
  `TldExpiredFrozen`;
- **PoW bằng lúc tạo**: challenge
  `SCONE-TLD-V1 ‖ name ‖ current_expiry(BE64)` (hậu tố chống tái
  sử dụng trói mỗi PoW vào MỘT hạn), cùng độ khó theo chiều dài
  như lời tuyên bố ban đầu;
- gia hạn KHÔNG bị đóng băng trong thời gian hoãn — nó là thao
  tác quản lý duy nhất còn lại cho một chủ đã hết hạn.

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Trường | Kiểu | Ràng buộc |
|---|---|---|
| `domain_id` | 32 byte thô | tên miền đích |
| `owner` | 32 byte thô | dẫn xuất của `public_key` |
| `sequence` | varint | **> 0**, phải bằng `hiện_tại + 1` lúc áp dụng |
| `record_hash` | 32 byte thô | cam kết với bộ bản ghi DHT |
| `public_key` | 32 byte thô | khóa công khai Ed25519 của người ký |
| `signature` | **đúng 64 byte** | chữ ký Ed25519 của payload ký |

Ghi chú: `UpdateDomain` **không** mang `timestamp` — thứ tự đi
qua `sequence`. Nó cũng không mang tên: tên miền đã đăng ký rồi,
id là đủ.

### Biên nghiêm ngặt

- `public_key`: **đúng 32 byte thô**, không tiền tố chiều dài.
  Một pk không giải mã được như một điểm trên đường cong → lỗi
  `InvalidTransactionKey` lúc giải mã.
- `signature`: **đúng 64 byte thô**, không tiền tố chiều dài,
  không dư địa. Mọi kích thước khác là lỗi giải mã cứng (cắt cụt
  bị từ chối).
- `proof`: varint chiều dài + byte, nắp 256
  (`MAX_PROOF_LEN`), kiểm tra trước khi cấp phát.
- `name`: varint chiều dài + byte UTF-8, nắp 253
  (`MAX_NAME_LEN`), xác thực lại bởi các quy tắc đặt tên của
  `scone-core`.

## Dẫn xuất owner

Quy tắc cứng: **owner luôn được tính lại, không bao giờ tin**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Một giao dịch mà `owner` không khớp dẫn xuất của chính
`public_key` của nó là vô hiệu (`InvalidOwner` phía core,
`OwnerKeyMismatch` phía chuỗi), bị từ chối lúc mã hóa, giải mã
**và** xác thực chuỗi (phòng thủ nhiều lớp).

## Payload ký

Payload ký là mã hóa chuẩn của giao dịch **không có trường chữ
ký**, tiền tố bằng một thẻ tách biệt miền — **với `network_id`
của chuỗi đích được tiêm ở đầu**:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

trong đó `canonical_encode(tx_without_signature, network)` theo
cấu trúc wire trên, gồm discriminant và byte phiên bản, dừng
trước `signature`, **với `network` là trường đầu tiên** — đây là
nơi DUY NHẤT mạng xuất hiện. Chuỗi tính lại payload với mạng
CỦA CHÍNH NÓ: một giao dịch ký cho mạng khác thất bại xác minh
chữ ký (`InvalidSignature`); đây là cơ chế chống phát lại chéo
chuỗi.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] public_key[32]
```

Tính chất:

- tất định và thuần túy: cùng giao dịch ⇒ cùng payload ở mọi
  nơi;
- tiền tố ASCII `SCONE-TX-SIG-V1` (15 byte,
  hex `53434f4e452d54582d5349472d5631`) ngăn mọi tái sử dụng
  chữ ký trong một miền băm khác của giao thức;
- mọi trường trừ `signature` đều được phủ — **kể cả tên mang
  bởi một `RegisterDomain`**: sửa bất kỳ trường nào làm vô hiệu
  chữ ký;
- `network_id` tiêm được phủ: cùng tx chưa ký ký cho hai mạng
  tạo ra hai payload khác nhau (do đó hai chữ ký khác nhau).

## Bằng chứng công việc đăng ký (PoW)

Trường `proof` của `RegisterDomain` và `RegisterTld` mang bằng
chứng công việc đăng ký (mô-đun `scone-core::pow`).

### Digest (muối epoch, chống tiền-đào)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blocks)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` là `network_id` chuẩn (M8b): một bằng chứng đào cho
mạng này không phải bằng chứng trên mạng khác.

`SCONE-POW-V1-ARGON2` cũng ghim các tham số Argon2id theo từng
mạng (`pow_params`: mainnet m = 64 MiB, t = 2, p = 1;
testnet/devnet m = 8 KiB, t = 1, p = 1): lõi Argon2id (RFC 9106)
làm cho chi phí mỗi băm bị chi phối bởi bộ nhớ (chống GPU/ASIC),
và một bằng chứng đào dưới một bộ tham số không bao giờ xác minh
được dưới bộ khác.

`challenge` là đầu vào dẫn xuất của đối tượng được tuyên bố —
`"SCONE-TLD-V1" || tld` cho một TLD, `"SCONE-DOMAIN-V1" || name`
cho một tên miền: đúng các byte được băm cho `TldId` /
`DomainId`, để một bằng chứng không bao giờ phát lại được chéo
giữa các không gian tên hay giữa các loại đối tượng. `epoch_le64`
và `nonce_le64` là các mã hóa 8 byte little-endian của epoch và
nonce.

Bằng chứng hợp lệ khi `digest` có ít nhất `difficulty` bit 0
dẫn đầu.

#### Muối epoch — chống tiền-đào

Một digest không muối không ràng buộc thời gian hay chiều cao:
bất kỳ ai cũng có thể tiền-đào hôm nay các bằng chứng cho hàng
nghìn tên, rồi đăng ký ồ ạt lúc mainnet khởi động (chiếm dụng) —
bằng chứng vốn tự chứa và hợp lệ ở khối nào đến trước. Digest
được muối bằng **epoch của khối ÁP DỤNG giao dịch**:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` khối
  (~2,6 h ở 65 s/khối): một cửa sổ khoan nhượng cho người dùng
  trung thực (đào, rồi nộp trong cửa sổ), buộc đào lại khi vượt
  qua;
- bằng chứng TUYÊN BỐ epoch của nó trên wire; lúc áp dụng,
  epoch khai báo phải rơi vào **cửa sổ hiệu lực** của khối áp
  dụng — chỉ dung sai LÙI (nộp muộn được hấp thụ, đào "cho
  tương lai" là bất khả), cửa sổ có kích thước
  `epoch_validity_window(difficulty)` = `1 << max(0, difficulty −
  16)` epoch (cửa sổ mở rộng: các độ khó mainnet cao đào trong
  nhiều-ngày-đến-nhiều-tuần, dài hơn một epoch; tích trữ không
  thu được gì vì cửa sổ ≈ chi phí đào — mỗi bằng chứng của một
  đợt ồ ạt tốn đủ `2^difficulty` băm BÊN TRONG cửa sổ). Ở độ
  khó ≤ 16 bit (testnet, tên dài) quy tắc bằng-nghiêm-ngặt không
  đổi;
- vào mempool (best effort): chiều cao khối tương lai chưa
  biết — relay xác minh đối với epoch của `chiều cao hiện tại +
  1`; việc xác minh nghiêm ngặt lúc áp dụng vẫn là thẩm quyền;
- phát lại tất định: bằng chứng MANG epoch của nó và cửa sổ là
  hàm thuần túy của (độ khó, epoch khối) — một khối lịch sử
  xác minh lại giống hệt trên bất kỳ nút nào.

Thay đổi gãy trước-mainnet: định dạng compact cũ (12 byte) bị từ
chối không tương thích ngược.

### Payload wire (20 byte cố định)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

Độ khó và epoch đi cùng bằng chứng, nhưng chúng là **do người
produce khai báo và do đó không bao giờ tin**: bên xác minh so
sánh độ khó với **hằng số giao thức** cho loại đăng ký (một
payload khai báo độ khó khác hằng số — thấp hơn *hoặc* cao hơn
— bị từ chối, `InvalidProof`), và epoch với epoch của khối áp
dụng (`ProofEpochMismatch` khi lệch). Nonce sau đó được xác minh
lại ở độ khó hằng số, digest tính lại từ đầu.

### Các độ khó (đường cong theo **chiều dài**, M8b + hiệu chuẩn đo được)

Độ khó KHÔNG phải một tham số phẳng: nó là một hàm tất định của
**chiều dài nhãn được tạo** (`scone-core::pow_length` — TLD cho
`RegisterTld`, nhãn ngoài cùng bên trái cho `RegisterDomain`),
theo từng mạng:

```text
difficulty(len) = apex − round(0.415 × (len − 1)), clamped at the floor
```

| Mạng | TLD riêng tư (apex/floor) | TLD sinh-mở / tên miền (apex/floor) |
|---|---|---|
| testnet | 8 bit / 2 bit | 6 bit / 2 bit |
| mainnet | 42 bit / 30 bit | 40 bit / 30 bit |
| devnet | 2 bit / 1 bit | 1 bit / 1 bit |

Một TLD **sinh-mở** (`RegisterTld` với `open = 0x01`) trả đường
cong "mở" (apex − 2, như `domain_difficulty`); **mở một TLD
riêng tư sẵn có** (`SetTldOpen`) giữ độ khó phẳng
(`open_tld_pow_difficulty`: 6 bit testnet / 22 bit mainnet /
2 bit devnet — tx chỉ mang `tld_id`, tên đã trả theo chiều dài
của nó lúc tạo). Benchmark mainnet (TLD riêng tư, ~6 M băm/s):
1 ký tự = 42 bit (≥ 7 ngày), 5 ký tự = 40 bit (~2 ngày),
19 ký tự = 35 bit (~1,6 h), 29+ ký tự = sàn 30 bit (~3 phút).
Các đường cong sống trong `scone-core::pow_length` (hằng số
`MAINNET_APEX = 42`, `MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`,
`TESTNET_FLOOR = 2`, `DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`);
các tham số phẳng (`tld_pow_difficulty`, v.v.) vẫn ở
`scone-core::network::NetworkParams`. Đổi một giá trị = một
thay đổi đồng thuận **cho mạng đó**. Bất biến: devnet là một
`NetworkParams` thứ ba hoàn chỉnh và testnet giữ nguyên tuyệt
đối các hằng số trước-devnet của nó (tham số Argon2id, đường
cong, cửa sổ) — devnet không chia sẻ hằng số nào với các mạng
khác ngoài chính mã nguồn.

Hiệu chuẩn đo được của các đỉnh phẳng (2026-10, `cargo run
--release -p scone-core --example pow_bench` — thông lượng
đơn luồng ~7,7 M H/s trên một x86-64 tham chiếu):

| Loại (mainnet) | độ khó | thời gian thực trung bình |
|---|---|---|
| `RegisterTld` | 24 bit | ~7 s |
| `RegisterDomain` | 20 bit | ~54 ms |

Ở ~65 s/khối: một lời tuyên bố TLD tốn ~0,1 khối CPU đơn nhân,
một tên miền ~1 ms khối — mang tính biểu tượng với người dùng,
tốn kém để công nghiệp hóa ở quy mô lớn (chống chiếm dụng). Cần
hiệu chuẩn lại nếu đối tượng phần cứng giảm (ARM di động: chậm
hơn ~4–8×).

Các nhánh trạng thái (M8b, đã bàn giao): `RegisterTld` LUÔN LUÔN
yêu cầu PoW của nó; `RegisterDomain` yêu cầu nó trên một TLD
**mở** (đường đóng là chỉ-assign — `AssignDomain`, không PoW,
ký bởi chủ TLD).

## Quy tắc xác thực

Một giao dịch hợp lệ khi và chỉ khi:

1. **giải mã nghiêm ngặt**: cấu trúc và biên trên được tôn trọng
   (discriminant biết, phiên bản `0x01`, pk 32 B, chữ ký 64 B,
   bất biến core kiểm tra lại lúc giải mã);
2. **ràng buộc owner/pk**: `owner == OwnerId::from(public_key)`
   (tính lại);
3. **nhất quán tên/id** (RegisterDomain): `domain_id ==
   DomainId(name)` (tính lại);
4. **chữ ký**: Ed25519 `verify_strict` của `signature` đối với
   payload ký **tính lại từ đầu** với mạng của chuỗi (không bao
   giờ lấy từ một peer hay từ giao dịch như được cung cấp) —
   một tx ký cho mạng khác thất bại ở đây (`InvalidSignature`,
   chống phát lại chéo chuỗi);
5. **quy tắc trạng thái** (lúc áp dụng trong khối — M8b, xem
   `/docs/en/technical/blockchain.md`):
   RegisterTld → TLD tự do + không ICANN + PoW (độ khó theo
   chiều dài); RegisterDomain → TLD đã đăng ký **và mở** + tên
   miền tự do (gồm thời gian hoãn) + PoW (theo chiều dài);
   UpdateDomain → tên miền tồn tại, `owner` hiện tại,
   `sequence == hiện_tại + 1`;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → TLD
   tồn tại **và riêng tư**, người ký = chủ TLD;
   TransferDomain → tên miền tồn tại, chủ;
   RenewDomain → tên miền tồn tại, kéo dài nghiêm ngặt ≤ 3 năm,
   ủy quyền PoW (TLD mở: cùng PoW như lúc tạo, **ràng buộc vào
   hạn hiện tại** — một challenge khác với đăng ký và mọi lần
   gia hạn trước) hay PoS (TLD riêng tư: chữ ký của chủ TLD —
   hay vận hành viên hiệu lực — `proof` rỗng).

Việc xác minh dùng `verify_strict` (từ chối chữ ký uốn nắn được
và khóa yếu/không chuẩn), như yêu cầu cho dữ liệu không tin
cậy.

## Thứ tự thao tác trong `push_block`

Xác thực mật mã (2 đến 4) diễn ra cho mỗi giao dịch **trước**
khi áp dụng trạng thái, trên một trạng thái làm việc nguyên tử
cho mỗi khối; mọi lỗi để lại chuỗi thay đổi một cách chặt chẽ
hay không đổi gì.

## Lắp ráp khối

`BlockBuilder` (`scone-blockchain`) lắp ráp một khối từ một
hàng đợi giao dịch: `tx_root` tính lại trên danh sách có thứ tự,
header móc vào băm cha (`prev_hash`), `version` =
`PROTOCOL_VERSION`. Payload đồng thuận (payload người produce có
chữ ký) và timestamp do bên gọi cung cấp; trait `Consensus` hiện
có được dùng đối xứng với xác thực.

## Ví dụ hex (vectơ thật, tái lập được — M8b)

### RegisterDomain `example.uip`, timestamp 42, testnet

Payload ký (vectơ ghim; được sinh lịch sử bởi công cụ ngoại
tuyến đã gỡ — các giá trị vẫn là vectơ hợp lệ). `network` xuất
hiện ở đây vì đây là PAYLOAD KÝ (nơi duy nhất của nó); giao
dịch wire đã ký tự thân không chứa network:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
21                                    disc REGISTER_DOMAIN (0x21)
01                                    format version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, length 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (fixed build key)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + start of sig
```

### RegisterTld `uip`, timestamp 42

Payload ký (vectơ ghim) — `TldId` là vectơ dẫn xuất được ghim
trong `/docs/en/general/naming.md`:

```text
53434f4e452d54582d5349472d5631       SCONE-TX-SIG-V1 prefix
93 01                                 disc REGISTER_TLD + version
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, 3-char label)
00                                    open (0x00 = private birth)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(Các giá trị đầy đủ từng tái lập được qua công cụ ngoại tuyến đã
gỡ; các vectơ dưới vẫn được ghim và hợp lệ. PoW được đào tự động
ở độ khó mạng — nonce thay đổi theo mỗi lần chạy; chỉ
network/name/id/owner/pk là ổn định. Kể từ thay đổi wire
2026-09, hex của giao dịch đã ký KHÔNG CÒN chứa các byte
`0d 73636f6e652d746573746e6574`: network chỉ đi trong payload
cần ký.)

## Hex giờ sống ở đâu

Các lệnh ngoại tuyến `tx build|sign|verify` đã bị gỡ
(cli-simplify-v1). Chính định dạng hex-đã-ký không đổi và vẫn là
định dạng wire của **luồng web/API**: ứng dụng web registrar
dựng, ký (keystore trong trình duyệt) và nộp giao dịch dưới dạng
hex chuẩn qua API. Phía CLI, các đường người dùng là các dạng
một-lệnh:

```bash
scone domain register example.uip        # đào, ký, nộp
scone domain renew example.uip                      # bằng chứng gia hạn, ký, nộp
scone domains                            # những gì bạn sở hữu
```

Trong mọi đường mã, owner và khóa đều được **tính lại** từ danh
tính ký, không bao giờ lấy từ bất kỳ đầu vào nào.

## Lịch sử phiên bản

| Phiên bản | Nội dung | Trạng thái |
|---|---|---|
| 1 | `public_key` + `signature` nhúng, payload `SCONE-TX-SIG-V1`, byte phiên bản `0x01`, `RegisterDomain` mang tên, `RegisterTld` (0x93), gia đình M8a: `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — định dạng wire chuẩn; M8b: trường `network` ký ở đầu mọi payload, `RegisterTld` mang tên, quy tắc trạng thái đầy đủ, độ khó PoW theo mạng | bị thay thế |
| 1 (2026-09, wire không network) | **gỡ trường `network` khỏi wire của mọi giao dịch**: network_id chỉ sống trong payload ký (`signing_payload(tx, network)` tiêm nó ở đầu); sự tách mạng dựa vào chữ ký (`InvalidSignature` trên payload lạ, lỗi `WrongNetwork` biến mất); `TX_FORMAT_VERSION` giữ `0x01` (testnet tái khởi động, không cùng tồn tại); **mọi TxId đổi** (TxId là băm của mã hóa wire đã ký, mất 14–15 byte); digest PoW, genesis và thư mục dữ liệu theo mạng không đổi | bị thay thế |
| 1 (2026-10, muối epoch PoW) | **digest PoW chống tiền-đào**: muối `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + lõi Argon2id trên `nonce_le`, với `epoch = block_height / 144` của khối ÁP DỤNG tx; payload wire bằng chứng 12 → **20 byte** (`nonce_le ‖ epoch_le ‖ difficulty_le`), epoch khai báo được xác minh đối với cửa sổ hiệu lực epoch của khối áp dụng (từ chối có kiểu `ProofEpochMismatch`), best-effort `(height+1)/144` lúc vào mempool, cờ CLI `--epoch` trên các đường ngoại tuyến; thay đổi gãy trước-mainnet: payload 12 byte cũ bị từ chối, **mọi TxId đổi** (trường `proof` dài thêm 8 byte) | hiện hành |

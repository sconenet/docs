# Blockchain

Tài liệu tham chiếu cho tầng blockchain: danh tính, băm, genesis,
trạng thái chuẩn, thứ tự giao dịch và xác thực khối. Triển khai
tham chiếu là crate
[`scone-blockchain`](../../../crates/scone-blockchain), trên nền
`scone-protocol` (mã hóa chuẩn), `scone-core` (quy tắc nghiệp
vụ) và `scone-crypto` (BLAKE3).

Phạm vi hiện tại: logic blockchain với đồng thuận thật (người
produce có chữ ký, checkpoint, tính cuối), **trạng thái trong bộ
nhớ** trên một backend KV (lưu trữ cụ thể sống trong
`scone-storage`) — xem [Hoãn cho đồng thuận](#hoãn-cho-đồng-thuận).

## Danh tính và tách biệt miền băm

Bốn danh tính, bốn vai trò, bốn tiền tố BLAKE3 khác nhau — một
tiền tố không bao giờ được tái sử dụng qua hai cách dùng:

| Kiểu | Vai trò | Tiền tố | Công thức |
|---|---|---|---|
| `DomainId` | danh tính tên miền | `SCONE-DOMAIN-V1` | xem `/docs/en/general/naming.md` |
| `RecordHash` | danh tính nội dung DNS | `SCONE-RECORD-V1` | xem `/docs/en/technical/protocol.md` |
| `TxId` | danh tính giao dịch | `SCONE-TX-V1` | dưới |
| `BlockHash` | danh tính khối | `SCONE-BLOCK-V1` | dưới |

Cây Merkle dùng tiền tố riêng `SCONE-MERKLE-V1`. Các tiền tố này
cộng thêm vào các tiền tố của `scone-core` (`SCONE-PUBKEY-V1`,
`SCONE-OWNER-V1`).

Chuỗi phụ thuộc đầy đủ:

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash of the next block
    ↓
canonical chain
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- tất định: cùng một giao dịch (cùng mã hóa chuẩn) ⇒ cùng
  `TxId`, trên mọi nút, mãi mãi;
- danh tính **theo nội dung**: mọi trường bị sửa đổi đổi `TxId`
  (tên miền, owner, timestamp, proof, sequence, record_hash);
- không timestamp cục bộ hay giá trị không chuẩn nào vào phép
  tính;
- không phụ thuộc lưu trữ hay mạng;
- danh tính không bảo đảm tính duy nhất của việc áp dụng: phát
  lại một giao dịch giống hệt thất bại trên quy tắc trạng thái
  (`RegisterDomain` kép, phát lại sequence), không phải trên
  `TxId`.

API: `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Cây Merkle (tx_root)

Cam kết trên **danh sách có thứ tự** các giao dịch của một khối:

```text
0 transactions  root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf            BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
internal node   BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
odd level       the last hash is duplicated to complete the pair
```

Quy tắc:

- **không bao giờ sắp xếp**: thứ tự giao dịch trong một khối là có
  ý nghĩa; hai khối chứa cùng các giao dịch theo thứ tự khác
  nhau có gốc khác nhau;
- các thẻ `0x00` / `0x01` ngăn mọi nhầm lẫn giữa một lá và một
  nút nội bộ (second-preimage);
- tiền tố `SCONE-MERKLE-V1` cô lập cây khỏi các miền băm khác
  (một `TxId` không thể được diễn giải như một nút);
- triển khai tự nhà (~30 dòng), tất định và được kiểm thử; không
  thư viện Merkle ngoài.

API: `merkle_root(&[TxId]) -> MerkleRoot` và
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

Băm chỉ phủ **header chuẩn**: các giao dịch đã được cam kết bởi
`tx_root`, nên chúng không bị băm lần thứ hai:

```text
transactions -> TxIds -> MerkleRoot -> BlockHeader -> BlockHash
```

Sửa một giao dịch, thứ tự của chúng, chiều cao, timestamp,
`prev_hash`, hay payload đồng thuận đổi `BlockHash`.

API: `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Mạng và genesis (M8b)

Scone định nghĩa ba mạng tách rời, định danh bằng một `network_id`
chuẩn (1..=16 byte ASCII `[a-z0-9-]`):

| Mạng | `network_id` | apex TLD riêng tư | apex TLD mở | apex tên miền | sàn TLD | sàn tên miền |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bit | 7 bit | 6 bit | 2 bit | 1 bit |
| mainnet | `scone-mainnet` | 28 bit | 27 bit | 26 bit | 25 bit | 22 bit |
| devnet | `scone-devnet` | 2 bit | 1 bit | 1 bit | 1 bit | 1 bit |

Các tham số sống trong `scone-core::network::NetworkParams`
(các thực thể đặt tên `TESTNET` / `MAINNET` / `DEVNET`, tra qua
`NetworkParams::by_name`). Đổi một độ khó = một thay đổi đồng
thuận **cho mạng đó**.

**Bất biến cấu trúc**: devnet là một `NetworkParams` thứ ba hoàn
chỉnh — không bao giờ là một tập con của các cờ testnet.
Testnet giữ **nguyên tuyệt đối** những gì nó đã có trước khi
devnet được thêm (cùng hằng số Argon2id, cùng đường cong, cùng
cửa sổ epoch); bộ kiểm thử ghim bất biến này.

**Độ khó thay đổi theo chiều dài** (`scone-core::pow_length`):
độ khó của một lần tạo phụ thuộc chiều dài của **nhãn được tạo**
(TLD cho `RegisterTld` — riêng tư hay mới mở —, nhãn ngoài cùng
bên trái cho `RegisterDomain`). Công thức cho:

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), clamped at the floor
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), clamped at the floor
```

Hai độ dốc khác nhau: `log₂(1.22) ≈ 0.287` bit/ký tự cho TLD
(mỗi ký tự thêm chia chi phí cho ~1.22), và
`log₂(1.40) ≈ 0.480` bit/ký tự cho tên miền — tên miền suy giảm
nhanh hơn: giá trị của chúng nằm ở TLD mang chúng. Một TLD
sinh-mở nằm ở apex riêng tư − 1; một tên miền ở apex riêng tư − 2
(mainnet: TLD riêng tư 25, TLD mở 24, tên miền 23). Benchmark
mainnet (Argon2id nặng-bộ-nhớ, ~11,4 H/s đơn luồng): TLD riêng tư
1 ký tự = 28 bit ≈ 30 ngày, 9+ ký tự = sàn 25 bit ≈ 3,8 ngày;
TLD mở 1 ký tự = 27 bit ≈ 15 ngày; tên miền 1 ký tự = 26 bit ≈
3,8 ngày, 9+ ký tự = sàn 22 bit ≈ 11,5 h. **Máy tham chiếu
8 nhân** (tái hiệu chuẩn 2026-10 — CLI đào đa nhân), ~11,4 H/s/nhân.
**Mở** một TLD riêng tư sẵn có (`SetTldOpen`) giữ độ khó phẳng
(`open_tld_pow_difficulty`: 6 bit testnet / 22 bit mainnet): tx
chỉ mang `tld_id`, tên đã trả theo chiều dài của nó lúc tạo.
Hàm thuần túy (tất định: hai nút tính cùng một độ khó cho cùng
một tên).

Genesis là một hằng số giao thức **theo từng mạng**, tái lập được
chỉ từ các hằng số (không ngẫu nhiên, không đồng hồ cục bộ,
không mạng):

| Trường | Giá trị |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 byte 0 (không có khối trước) |
| `tx_root` | Gốc Merkle của danh sách rỗng |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (thuần cấu trúc) |
| `consensus` | các byte chuẩn của `network_id` |
| giao dịch | rỗng |

Biến thể **genesis phân bổ** (kickstarter mainnet, M-next):
trường `consensus` mang `network_id ‖ ALLOC-V1 ‖ count.v ‖
entries` (các tên miền/TLD trả trước, sắp chuẩn theo tên,
manifest ≤ 60 KiB, ≤ 2048 phân bổ) — chính định dạng mở rộng này
biện minh cho `MAX_CONSENSUS_LEN` = 64 KiB (xem
`scone-blockchain::genesis_alloc`).

`network_id` trong payload đồng thuận làm cho các băm genesis —
và do đó toàn bộ blockchain — **tách rời giữa các mạng**: một
khối testnet không bao giờ gắn được vào một chuỗi mainnet
(`UnknownParent`), không phát lại được nếu không ký lại tất cả.

API: `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash`; các bí danh
`genesis()` / `genesis_hash()` chỉ testnet (mặc định phát
triển).

### Devnet — mạng cục bộ đặt lại được

Devnet (`scone-devnet`) là mạng thứ ba, dành cho lặp cục bộ
nhanh:

- **Genesis riêng**: id mạng `scone-devnet` trong payload đồng
  thuận cho một băm genesis khác với CẢ testnet lẫn mainnet —
  không genesis, khối, chữ ký hay bằng chứng PoW nào bao giờ
  vượt qua biên giới (phát lại là bất khả).
- **PoW gần-bằng-không**: đường cong ghim trong dải 0..=2 bit
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — suy giảm theo chiều
  dài vẫn áp dụng): một lời tuyên bố TLD hay tên miền đào trong
  vài mili-giây, luyện đúng đường mã giống các mạng khác.
- **Đồng thuận PoS nhanh**: cùng tham số như testnet (ủy ban 4,
  epoch 10 s) cho tính cuối nhanh trên một mạng cục bộ.
- **Đặt lại**: chuỗi sống trong thư mục dữ liệu riêng của nó
  (`~/.scone/devnet/` mặc định). **Xóa thư mục đó và khởi động
  lại relay đặt lại chuỗi** (genesis mới, kho còn nguyên) —
  không cần công cụ thêm. KHÔNG BAO GIỜ trỏ một thư mục dữ liệu
  devnet vào thư mục dữ liệu của mạng khác: một kho khác rỗng
  giữ một chuỗi khác bị từ chối (`Corrupted`).

## Trạng thái chuẩn

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- enum (owner-option): the open/private invariant is
               -- structural; the death deadline lives outside the entry
               -- (tld_floors table + journaled death schedule)
ChainState   = NetworkParams                  (bound network, M8b)
               + DomainId -> DomainState      (direct access, in memory)
               + TldId -> TldState            (TLD registry, M7b)
               + DomainId -> (expired_at, OwnerId)   (grace windows, M8b)
```

* `registered_at` / `valid_until` (M8b): hết hạn đăng ký, đánh
  giá đối với **timestamp của khối cha** (tất định, được cam kết
  trong header — không bao giờ đồng hồ cục bộ);
* `open`: một TLD riêng tư có thể **sinh ra đã mở**
  (`RegisterTld` với `open = 0x01` — không ai cả, tự phục vụ)
  hay **đóng** lúc tuyên bố (đóng = chỉ-assign), rồi trở thành
  mở qua `SetTldOpen` (tự đăng ký với PoW) — **không thể đảo
  ngược**;

- truy cập trực tiếp theo `DomainId` (32 byte): không bao giờ
  một `String` làm khóa, không bao giờ quét toàn bộ — một điều
  kiện cần để nhắm tới hàng trăm tỷ tên miền;
- `record_hash == None` chừng nào chưa có `UpdateDomain` nào
  được áp dụng;
- các không gian id TLD và tên miền **tách rời ngay từ cấu trúc**
  (tiền tố dẫn xuất khác nhau, xem `/docs/en/general/naming.md`):
  đăng ký TLD `uip` không bao giờ chiếm dụng danh tính một tên
  miền, và ngược lại.

### Quy tắc áp dụng

`state.apply(transaction)` là tất định và nguyên tử (khi lỗi,
trạng thái không đổi):

**Mọi giao dịch (M8b, wire 2026-09)**: giao dịch không còn mang
trường `network` — sự tách biệt dựa vào **chữ ký**: payload ký
tiêm `network_id` của chuỗi
(`validate_transaction(tx, network_of_the_chain)`), nên một tx
ký cho mạng khác thất bại xác minh chữ ký
(`InvalidSignature`, TRƯỚC khi áp dụng — một tx ký testnet
không bao giờ được áp dụng, không bao giờ vào mempool bởi một nút
mainnet, và ngược lại).

**REGISTER_TLD**:

- TLD phải **tự do** (vắng trong registry TLD) và **không thuộc
  danh mục dự trữ ICANN** (`IcannTldReserved`: các TLD gốc ICANN
  không thuộc về ai trên Scone);
- **PoW bắt buộc**: `pow::verify` trên challenge
  `SCONE-TLD-V1 || tld` (dẫn xuất từ tên mang bởi tx), ở độ khó
  **theo chiều dài** của nhãn (`pow_length`): sinh riêng tư →
  `tld_difficulty` (apex mạng), sinh mở (`open = 0x01`) →
  `open_tld_difficulty` (apex − 2) — digest được tính lại từ đầu
  (muối `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`,
  lõi Argon2id), độ khó tự-khai-báo phải bằng độ khó kỳ vọng, và
  **epoch khai báo** bởi bằng chứng phải bằng
  `height/POW_WINDOW` của khối áp dụng (muối epoch chống tiền-
  đào — từ chối có kiểu `ProofEpochMismatch`, xem
  `transactions.md`);
- khi áp dụng: `open = 0x01` → `{ Open }` (không ai — tự phục vụ,
  ghế PoS được giải phóng, sàn chết đặt tại `now + grace`);
  `open = 0x00` → `{ Private { owner } }` — **đóng mặc định**
  (chỉ-assign); việc mở là tường minh qua `SetTldOpen`.

**REGISTER (RegisterDomain)**:

- **TLD của tên mang theo phải được đăng ký VÀ mở**: chuỗi tự
  dẫn xuất `TldId(name.tld())` (không bao giờ một giá trị được
  cung cấp); vắng → `UnknownTld` (D1, M7c), có nhưng đóng →
  `TldClosed` (M8b: đường chỉ-assign là độc quyền);
- tên miền phải **tự do**: vắng trong trạng thái VÀ ngoài mọi
  cửa sổ hoãn của một đăng ký hết hạn (xem GC dưới);
- **PoW bắt buộc**: challenge `SCONE-DOMAIN-V1 || name` ở **độ
  khó theo chiều dài của nhãn ngoài cùng bên trái**
  (`pow_length::domain_difficulty` — TLD đang mở, đây là giá
  vào tự phục vụ);
- khi áp dụng: `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 năm }` — **đăng ký
  kéo dài 1 năm** (365 ngày).

**Thứ tự nội khối**: có ý nghĩa. `[RegisterTld, SetTldOpen,
RegisterDomain]` trong cùng khối là hợp lệ; mọi hoán vị áp dụng
một quy tắc trước khi tiền điều kiện của nó tồn tại bị từ chối
cả khối (undo-log).

**UPDATE (UpdateDomain)**:

- tên miền phải tồn tại (chưa hết hạn);
- `tx.owner` phải là chủ hiện tại;
- `tx.sequence` phải là **đúng bằng** `current_sequence + 1`
  (không khoảng trống, không phát lại);
- khi áp dụng: `sequence` tăng, `record_hash` được thay.

**TRANSFER_TLD (0x47)**:

- TLD phải tồn tại và **riêng tư** (nếu không `UnknownTld` /
  `TldIsOpen` — một TLD mở không thuộc về ai);
- người ký phải là chủ hiện tại (nếu không `NotTldOwner`);
- khi áp dụng: `owner = new_owner` (danh tính người nhận là mờ;
  việc ràng buộc vào các khóa có thể chi tiêu nó do các tx sau
  đó làm). Không sequence: lần chuyển được áp dụng đầu tiên thắng
  (thứ tự chuỗi).

**REVOKE_TLD (0x6B)**:

- TLD phải tồn tại và **riêng tư** (một TLD mở không thể bị thu
  hồi — `TldIsOpen`: nó chết tự nhiên tại GC), người ký = chủ
  hiện tại;
- khi áp dụng: TLD bị gỡ khỏi registry và trở nên có thể tuyên
  bố lại bằng một `RegisterTld` mới (kèm PoW).

**SET_TLD_OPEN (0xB8)**:

- TLD phải tồn tại và **riêng tư** (đã mở → `TldIsOpen`: chủ
  cũ đã bị xóa, không còn khóa chính đáng nào để ký cho nó);
  `open = 0x00` (đóng) → từ chối có kiểu `TldCloseUnsupported`
  ngay từ xác thực thuần: **việc mở là không thể đảo ngược**
  (tùy chọn chủ);
- người ký = chủ hiện tại (`NotTldOwner`);
- **mở = PoW bắt buộc**: `pow::verify` trên challenge
  `SCONE-TLD-OPEN-V1 || tld_id` ở độ khó phẳng
  `open_tld_pow_difficulty` của mạng (6 bit testnet / 22 bit
  mainnet), kèm muối epoch (bằng chứng đi trong trường
  `open_proof` được ký của tx);
- khi áp dụng: `TldState::Open` — **chủ bị xóa** (không thể đảo
  ngược, ghế PoS được giải phóng), một sàn chết được đặt tại
  `now + grace`.

**ASSIGN_DOMAIN (0xD4)**:

- TLD của tên mang theo phải tồn tại và **riêng tư** (nếu không
  `TldIsOpen`: một TLD mở chỉ tự phục vụ), người ký = chủ của
  TLD (`NotTldOwner`);
- tên miền phải tự do (cùng quy tắc hoãn như REGISTER);
- khi áp dụng: `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 năm }`.
  **Không PoW**: chính chủ không gian tên bảo lãnh.

**TRANSFER_DOMAIN (0xE2, M8c)**:

- tên miền phải tồn tại (nếu không `UnknownDomain`), người ký =
  chủ hiện tại (`NotOwner`);
- khi áp dụng: `owner = new_owner` (ghế PoS đi theo tên miền
  trong tập chủ, đối xứng với `TransferTld`). Không sequence:
  lần chuyển được áp dụng đầu tiên thắng (thứ tự chuỗi).

**RENEW_DOMAIN (0x3C)**:

- tên miền phải tồn tại; **loại TLD được đánh giá TRƯỚC**
  (củng cố M8c):
  - **TLD riêng tư**: ủy quyền **PoS** — người ký CHÍNH LÀ chủ
    TLD, hoặc ghế của chủ TLD ủy quyền cho khóa công khai của
    người ký
    (`effective_operator(tld_owner) == renew.public_key` —
    ủy quyền luôn được đọc TỪ chủ TLD, không bao giờ ngược
    lại). Trường `proof` phải rỗng (proof không rỗng = từ
    chối).
  - **TLD mở**: ủy quyền **PoW** — người ký là chủ hiện tại của
    tên miền (nếu không `NotOwner`) và bằng chứng đào challenge
    gia hạn: cùng độ khó như lúc tạo (theo chiều dài nhãn),
    **ràng buộc vào hạn hiện tại**
    (`challenge = DOMAIN_ID_VERSION ‖ canonical name ‖ current
    valid_until`) — proof của một lần gia hạn khác với của đăng
    ký và của mọi lần gia hạn trước (chống tái sử dụng);
- `valid_until` phải **kéo dài nghiêm ngặt** hạn hiện tại (nếu
  không `RenewalNotExtending`);
- `valid_until ≤ now + 3 năm` (`RenewalExceedsTerm` — biên
  chống tích trữ: **1 hạn gia hạn = 1 năm, tối đa 3 năm trước**);
- khi áp dụng: `valid_until = tx.valid_until`; việc gia hạn cũng
  trì hoãn sàn chết của TLD mở bên dưới (tx mang tên chuẩn cho
  mục đích đó).

**GC tất định (M8b)**: trước khi áp dụng các giao dịch của một
khối, chuỗi đánh giá các hết hạn tại **timestamp của khối cha**:
mọi tên miền có `valid_until ≤ now` bị gỡ khỏi trạng thái sống
và đỗ vào một **cửa sổ hoãn 30 ngày**
(`valid_until + 30 ngày`). Trong thời gian hoãn, chỉ chủ cũ mới
có thể tuyên bố lại tên; sau đó, tên tự do cho mọi người. GC
được ghi nhật ký: một khối bị từ chối khôi phục trạng thái
trước-GC bit cho bit.

**Vòng đời TLD (hạn cố định)**: các TLD mang đồng hồ riêng của
chúng, độc lập với tên miền.

- **TLD riêng tư**: lời tuyên bố mở một **hạn 3 năm**
  (`TLD_TERM_SECS`). Gia hạn (`RenewTld`, 0x8F) trả một **PoW
  bằng lúc tạo** và đặt lại đúng 3 năm (hạn cố định dẫn xuất
  lại lúc áp dụng). Khi hết hạn một **thời gian hoãn 60 ngày**
  (`TLD_GRACE_SECS`) mở ra: gia hạn vẫn khả năng, nhưng **mọi
  thao tác quản lý khác bị đóng băng** — đáng chú ý
  `AssignDomain` (từ chối có kiểu `TldExpiredFrozen`): một chủ
  hết hạn mất quyền quản lý cho đến khi gia hạn. Các tên miền
  đã cấp bên dưới sống `valid_until` riêng của chúng — TLD đóng
  băng, chứ không phải các con của nó. Hết hoãn mà không gia
  hạn: TLD chết tại GC (ghế PoS được giải phóng, có thể tuyên bố
  lại). Một `TransferTld` mang theo hạn còn lại.
- **TLD mở**: sinh ra (hoặc được mở) với một **sàn 1 năm**
  (`OPEN_TLD_FLOOR_SECS` — chống tạo-hủy tức thời). Mọi tên
  miền còn sống bên dưới đẩy đồng hồ chết lùi lại (nó bám theo
  hạn của tên miền mới nhất). Không còn tên miền + đã quá sàn:
  chết tại GC, có thể tuyên bố lại.
- **TLD mở bất tử** (chỉ genesis: `.scone`, `.tor` — đặt tên
  kiểm chứng được trên các địa chỉ onion qua CNAME —, `.p2p`):
  phân bổ `OpenImmortalTld` đặt cột mốc `until = u64::MAX` —
  GC không bao giờ vượt qua được và việc tái vũ trang đơn điệu
  không bao giờ hạ nó được: TLD không bao giờ hết hạn, kể cả khi
  trống rỗng mọi tên miền của nó. Không giao dịch runtime nào
  tạo được một TLD bất tử (`RegisterTld` giữ sàn 1 năm của
  mình); các tên miền bên dưới sống chu kỳ bình thường.

Cái chết của các TLD chia hàng đợi chết tất định của tên miền
(`tld_deaths`), ghi nhật ký và khôi phục được bit-chính-xác.

**Ngoại lệ chiều cao-1**: genesis mang một timestamp vô hiệu VỀ
MẶT CẤU TRÚC (`GENESIS_TIMESTAMP` = 0, một quy ước giao thức —
không phải một đồng hồ). Thời gian chuỗi của khối THỨ NHẤT do
đó là chính timestamp của nó (được cam kết trong header, tất
định), không phải của khối cha: neo các hết hạn của khối 1 vào
0 sẽ ghim sàn chết của một TLD sinh mở (và `valid_until` của
một tên miền sinh tại khối 1) vào epoch Unix — GC trước-áp-dụng
của khối 2 (timestamp thật) sẽ gỡ tài nguyên trước khi áp dụng
("dropped conflicting tx from block production: unknown TLD",
hồi quy testnet 60).

**Phát lại**: phát lại chéo mạng là bất khả ở ba tầng — genesis
tách rời (`UnknownParent`), `network_id` tiêm vào mọi payload
chữ ký (chữ ký vô hiệu trên mạng khác — nơi duy nhất mạng xuất
hiện kể từ wire 2026-09), PoW tách theo từng mạng trong
digest. Phát lại nội mạng của một tx giống hệt thất bại trên
quy tắc trạng thái (tuyên bố kép, sequence), không bao giờ trên
`TxId`.

Hai nút của cùng mạng khởi động từ cùng trạng thái và áp dụng
cùng các khối theo cùng thứ tự tạo ra đúng cùng trạng thái cuối.

## Thứ tự giao dịch

Thứ tự là một tính chất nền tảng, được cam kết ở mọi tầng:

```text
transaction order in the block
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
canonical chain
```

Hai giao dịch không thể hoán đổi cho nhau: trong cùng khối,
`[REGISTER, UPDATE]` được áp dụng, `[UPDATE, REGISTER]` bị từ
chối (tên miền chưa tồn tại tại thời điểm `UPDATE`). Phá vỡ hòa
toàn cục (sắp mempool, produce khối) thuộc về đồng thuận.

## Xác thực khối (push_block)

Mọi thứ đều được **tính lại**, không có gì được tin — một
`tx_root`, băm, chữ ký hay trường do peer cung cấp không bao
giờ được lấy theo mặt chữ:

1. `prev_hash` == băm của đỉnh chuẩn (nếu không `UnknownParent`
   khi cha không rõ, `ParentNotTip` khi rõ nhưng không phải
   đỉnh: phát hiện fork tối thiểu);
2. `height` == chiều cao đỉnh + 1;
3. `version` == `PROTOCOL_VERSION` **chính xác** (mọi phiên bản
   khác — thấp hơn hay cao hơn — đều bị từ chối, không diễn giải
   lại);
4. số giao dịch ≤ `MAX_TXS_PER_BLOCK` (4096);
5. `tx_root` tính lại trên các giao dịch theo thứ tự khối;
6. các móc đồng thuận (`validate_header`, `validate_tx`);
6ter. **chống phát lại TXID cửa sổ (cổng .bak)**: một giao dịch
   mà `TxId` xuất hiện trong chỉ mục bao hàm của
   `REPLAY_WINDOW_BLOCKS = 256` khối gần nhất bị từ chối
   (`TxReplay`). Chỉ mục (TxId → chiều cao bao hàm) là hàm
   thuần túy của chuỗi chuẩn, được xén cùng nhịp; chỉ mục được
   dựng lại lúc khởi động bởi storage (quét lại
   `min(256, height)` khối bền vững cuối — M8, xem
   `/docs/en/technical/storage.md`): một nút khởi động lại từ
   chối cùng các phát lại như một nút sống. Vượt cửa sổ, việc
   bao hàm lại vẫn phục tùng quy tắc trạng thái (tuyên bố kép,
   phát lại sequence).
6bis. **người produce có chữ ký (M5, cổng .bak)**: payload
   `consensus` phải giải mã được như một payload người produce
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, bị chặn); chữ ký phủ
   *băm chữ ký người produce* = `BlockHash` của header **với
   trường `consensus` rỗng** (payload sống trong trường đó; ký
   băm đầy đủ sẽ vòng tròn); người produce phải thuộc tập được
   ủy quyền tại `timestamp` của khối (ủy ban được bầu bởi
   checkpoint, cộng các lần rút khôi phục đã mở khóa;
   bootstrap: chủ của các tên miền còn sống — một tập rỗng =
   produce mở, nếu không lần đăng ký đầu tiên sẽ bất khả).
   Genesis (chiều cao 0, payload rỗng) là cấu trúc và không
   bao giờ liên quan. Lỗi có kiểu `InvalidProducer`;
7. cho mỗi giao dịch, theo thứ tự: xác thực **mật mã**
   (`validate_transaction`: ràng buộc owner/khóa tính lại +
   `verify_strict` trên payload ký tính lại từ đầu, xem
   `/docs/en/technical/transactions.md`), rồi áp dụng lên trạng
   thái chuẩn dưới một **nhật ký hoàn tác** (undo-log: một mục
   cho mỗi giao dịch được áp dụng, chi phí O(giao dịch của
   khối), không O(tên miền));
8. commit nguyên tử: mọi thứ qua ⇒ khối được thêm, nhật ký bị
   vứt; lỗi nhỏ nhất ⇒ quay ngược nhật ký (thứ tự đảo) và chuỗi
   chặt chẽ không đổi, bit cho bit.

Không hàm xác thực nào panic; mọi đầu vào sai dạng hay thù địch
tạo một `BlockchainError` có kiểu
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Lắp ráp khối (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) lắp ráp một khối từ
một hàng đợi giao dịch đã xác thực:

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **tính lại** trên danh sách có thứ tự của các giao
  dịch trong hàng đợi (không bao giờ được cung cấp);
- `prev_hash`/`height` móc vào cha (băm cha do bên gọi tính qua
  `block_hash`, không bao giờ lấy từ peer);
- `version` = `PROTOCOL_VERSION`;
- biên `MAX_TXS_PER_BLOCK` được áp dụng lúc chèn
  (`TooManyTransactions`);
- payload `consensus` (payload người produce có chữ ký) và
  `timestamp` do bên gọi cung cấp: đồng thuận cụ thể cắm vào qua
  trait [`Consensus`](#trừu-tượng-đồng-thuận) hiện có, đối xứng
  với xác thực;
- M5: `.with_producer(&SigningKey)` ký khối với tư cách người
  produce — payload `consensus` trở thành payload người produce
  có chữ ký (trên băm chữ ký người produce định nghĩa trên) và
  thay thế mọi `.with_consensus` trước đó.

Một khối lắp ráp xong đi qua `push_block` không cần mã hóa lại
(đã kiểm thử).

## Fork và lựa chọn fork (M3c, cổng .bak)

`push_block` vẫn là điểm vào tuyến tính (cha == đỉnh, nếu không
`UnknownParent`/`ParentNotTip`). `try_attach` (M3c) đánh giá
các khối cạnh tranh:

- cha không rõ → `UnknownParent` (khối chờ);
- dòng dõi hợp lệ nhưng trễ → đánh giá nhánh: chuỗi dài nhất
  thắng; hòa hoàn hảo → phá vỡ hòa tất định (băm đỉnh nhỏ nhất
  theo thứ tự từ điển — hai nút giữ cùng các khối luôn chọn
  cùng người thắng, đã kiểm thử);
- **sàn tính cuối**: một nhánh mà một tổ tiên nào đó mâu thuẫn
  với một checkpoint hoàn tất (bao gồm trong
  `prev_checkpoint_hash` của một checkpoint được chấp nhận) bị
  từ chối (lỗi có kiểu
  `Consensus("branch contradicts a finalized checkpoint")`);
- nhận nuôi = phát lại từ genesis các khối của nhánh thắng (tính
  tất định bit-chính-xác được kiểm thử: hai nút nhận cùng các
  khối theo thứ tự khác nhau kết thúc trong cùng trạng thái);
- không khối nhận được nào là chuẩn trước khi xác thực đầy đủ.

## Cửa sổ RAM bị chặn (M7a, cổng .bak)

Chuỗi chỉ giữ **512 khối cuối** trong RAM
(`RAM_WINDOW_BLOCKS`, cổng của `HISTORY_KEEP = 512` bản .bak),
trên **sàn tính cuối** (checkpoint hoàn tức cuối cùng):

- một khối chỉ rời RAM nếu nó **vừa** ngoài cửa sổ
  (`height < tip − 511`) **vừa** dưới nghiêm ngặt sàn tính cuối
  — một reorg trên sàn luôn còn khả năng, nên các khối đó
  KHÔNG BAO GIỜ bị trục xuất;
- trước checkpoint hoàn tất đầu tiên (bootstrap), **không gì bị
  trục xuất**: bộ nhớ bị chặn bắt đầu cùng với tính cuối;
- việc trục xuất là thật: các khối, chỉ mục băm (`known_hashes`,
  chính nó bị chặn, lập chỉ mục theo chiều cao), và các tx rời
  bộ nhớ — kho (`NodeStore`, `scone-storage`) giữ các bản bền
  vững;
- `block(h)` dưới cửa sổ → `None`; `block_result(h)` → lỗi có
  kiểu `BlockPruned { height }` — các đường reorg
  (`push_block`, `try_attach`, `adopt_branch`, phát lại tiền
  tố) truyền nó: relay phải nạp lại đoạn từ storage hoặc quay về
  đồng bộ đầy đủ (hạn chế được ghi nhận: không có nhận nuôi
  phân đoạn trước khi cổng relay đầy);
- một khối dựng trên một cha đã bị trục xuất phân loại là
  `BlockPruned` (không phải `UnknownParent`): chiều cao cha <
  `base_height` ⇒ kho có thể phục vụ nó.

## Trừu tượng hóa đồng thuận

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (mặc định): chấp nhận mọi thứ — dùng cho
  đến khi các quy tắc thật được định nghĩa;
- các móc được gọi bởi `push_block`: đồng thuận cụ thể (PoW, độ
  khó, phí, timestamp/chống phát lại, lựa chọn fork) sẽ cắm vào
  mà không viết lại chuỗi;
- một trừu tượng duy nhất, không có phân tầng trait.

## Trạng thái khả mở rộng (backend KV, trục SMT, hàng đợi hết hạn, tập chủ bị chặn)

Mục tiêu: hơn 250 tỷ tên miền tiềm năng — KHÔNG BAO GIỜ nạp tất
cả vào RAM. Bốn mảnh:

### Backend KV (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = 32 byte thô của `DomainId`/`TldId` — MỘT không gian
  khóa có thứ tự (tiền tố dẫn xuất tách rời), giá trị = các mã
  hóa `SCONE-ENTRY-DOM-V1` (88 byte cố định) /
  `SCONE-ENTRY-TLD-V1` (biến thiên: 33 byte riêng tư / 1 byte
  mở, enum tùy chọn chủ — cờ + chủ có điều kiện, không còn
  trường `until`);
- `MemoryBackend` (BTreeMap) = triển khai tham chiếu (dev,
  kiểm thử); KHÔNG phụ thuộc cụ thể nào trong `scone-blockchain`
  (redb bị cấm trong crate) — protocol/consensus chỉ phụ thuộc
  TRAIT;
- `CachedBackend<B>` (`state_cache.rs`) = cache LRU bị chặn
  trên một backend bên trong: write-through (backend bên trong
  luôn cập nhật), miss không bao giờ được cache, `range` phân
  trang backend bên trong — cùng câu trả lời như backend riêng
  (hợp đồng tất định, đã kiểm thử);
- `RedbBackend` (`scone-storage::redb_backend`) = backend bền
  vững: bảng `state_kv` (khóa thô 32 byte, thứ tự từ điển =
  thứ tự tăng dần yêu cầu), bộ đếm duy trì trong bảng meta
  `state_kv_meta` (không bao giờ COUNT);
  `CachedBackend<RedbBackend>` = trạng thái với dấu chân RAM
  bị chặn — "tầng nhẹ";
- `smt_persist`: tiêu thụ các tập bẩn của SMT
  (`take_dirty_sets`) và ghi chúng vào một backend dưới các
  khóa tiền tố-dành-riêng (0x01 nút, 0x02 lá) — nền của SMT phân
  trang;
- trait không còn yêu cầu `Default` (một cơ sở dữ liệu đĩa
  không có giá trị mặc định hợp lý);
- `range` lặp theo thứ tự khóa tăng dần (hợp đồng tất định) —
  chỉ đường lưu trữ, không bao giờ là quy tắc đồng thuận;
- `ChainState` truy cập tên miền/TLD CHỈ theo khóa qua backend
  (đọc điểm, ghi có nhật ký).

### Gốc SMT chuẩn (trục)

- `state_root_smt()` (O(1), gốc được cache, `SCONE-STATE-SMT-V1`)
  là state_root **chuẩn**: nó là cái `checkpoint_data` cam kết
  và cái `accept_checkpoint` xác minh tại đỉnh. Phép tính
  KHÔNG BAO GIỜ duyệt tập các tên miền;
- phép gấp trực tiếp cũ `SCONE-STATE-V1` sống sót như
  `state_root_archive()`: O(N), đường lưu trữ/xác minh (gấp lại
  đầy đủ từ storage để đối chiếu một gốc lịch sử) — không phải
  một tạo tác đồng thuận. Cả hai cam kết cùng các lá: trạng
  thái logic bằng nhau ⇒ gốc bằng nhau (cho cả hai);
- một checkpoint được chấp nhận với gốc SMT xác minh lại được
  sau một khôi phục (tải lại): gốc là hàm thuần của trạng thái
  khôi phục (backend + SMT + chỉ mục).

### Hàng đợi hết hạn (GC O(hết hạn), không O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — thời điểm
  hết hạn → các id hết hạn lúc đó (id sắp theo từng khe: đẳng
  thức trạng thái và thứ tự xả độc lập với thứ tự đến);
- `gc_expired_journaled(now)` xả `range(..=now)` thay vì quét
  toàn trạng thái: chi phí O(hết hạn tại thời điểm đó);
- nắp RAM: một mục cho mỗi tên miền CÒN SỐNG (chỉ hết hạn treo
  — một lần gia hạn khóa lại khe, GC xả nó), không bao giờ một
  nút cho mỗi tên miền trong một cấu trúc quét;
- duy trì bởi nhật ký hoàn tác (rollback bit-chính-xác: khe
  khôi phục được đặt lại vào).

### Tập chủ bị chặn (khóa, không phải tên miền)

- `owner_refcounts: HashMap<OwnerId, u64>` — một bộ đếm cho mỗi
  chủ còn sống RIÊNG BIỆT (tên miền + TLD), duy trì bởi nhật ký
  ở mọi register/transfer/revoke/assign/GC (acquire/release);
- `eligible_validators(now)` đọc chỉ mục này — O(chủ), KHÔNG
  BAO GIỜ một lặp qua tên miền. Sống còn thời gian đã được bảo
  đảm: GC tất định (timestamp cha) đã gỡ mọi tên miền hết hạn
  VÀ refcount của nó trước lời gọi. Đầu ra sắp xếp (tất
  định);
- nắp RAM: O(chú riêng biệt), độc lập với số tên miền;
- `owner_pool_root` (gấp lưu trữ) gấp các chủ đã sắp — cũng
  không quét tên miền.

Ngân sách RAM trạng thái-ổn-định (backend đĩa/mạng):
O(chú riêng biệt + hết hạn treo + cửa sổ hoãn đang bay + nhật ký
hoàn tác) — tất cả bị chặn độc lập với số tên miền, tất cả được
khôi phục bit-chính-xác bởi nhật ký.

## Ghế và ủy ban tính cuối (cơ chế PoS chính xác)

Phần này mô tả cơ chế đủ điều kiện và bầu-ủy-ban CHÍNH XÁC như
triển khai (`finality.rs`) — phần bổ sung vận hành (vòng anchor,
wire) sống trong `/docs/en/technical/relay.md`, phân tích bảo
mật trong `/docs/en/technical/security-model.md`.

### Một ghế = sự sở hữu (không đăng ký)

Một **ghế** PoS KHÔNG PHẢI một đăng ký: nó là sự sở hữu một tên
miền hay TLD còn sống. `eligible_validators(now)` tính tập hợp:

- các khóa của các chủ còn sống qua chỉ mục `owner_refcounts`
  bị chặn (một chủ RIÊNG BIỆT cho mỗi tên miền/TLD còn sống —
  một trăm tên miền dưới một khóa = MỘT ghế: chống Sybil kinh tế
  là theo từng danh tính, tuyến tính);
- một TLD tính (lời tuyên bố của nó tốn một PoW — chủ của một
  TLD còn sống là một bên có cổ phần), một tên miền còn sống
  cũng vậy (`valid_until > now` — được bảo đảm bởi GC tất
  định, đã xả trước lời gọi);
- **trừ các khóa bị cấm** (M9: một kẻ equivocation bị slash
  khỏi tập hợp trọn đời);
- một TLD **mở** (`owner: None`) không giữ GHẾ nào: chủ cũ của
  nó được giải phóng lúc mở (`owner_release` trong phép áp
  dụng `SetTldOpen`), và nó chết tại GC mà không bao giờ bỏ
  phiếu nữa.

Đầu ra sắp xếp (thứ tự tất định, độc lập với thứ tự lặp của chỉ
mục).

Hệ quả cấu trúc: **cái chết của ghế = cái chết của tên
miền/TLD**. Hết hạn + GC, `RevokeTld` (ghế được giải phóng),
`TransferTld`/`TransferDomain` (ghế đi theo tên miền sang chủ
mới), mở bằng `SetTldOpen` (ghế được giải phóng) — mọi chuyển
tiếp sở hữu di chuyển hay giải phóng ghế trong tập hợp, đối
xứng với nhật ký `owner_refcounts` (acquire/release).

### Tập hợp ĐÓNG BĂNG tại checkpoint hoàn tất cuối (`FinalizedBase.pool`)

Khi chấp nhận một checkpoint (`accept_checkpoint`), cơ sở tính
cuối bắt giữ tập hợp của trạng thái TẠI KHỐI ĐƯỢC HOÀN TẤT:

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(timestamp của khối được hoàn
  tất)` — tính cuối KHÔNG BAO GIỜ đọc lại trạng thái sống sau
  đó: mọi đăng ký (tuyên bố, chuyển, cấm) sau checkpoint chỉ
  tính cho các epoch sau;
- `seed` = `next_seed(signing_hash(checkpoint))` — seed của
  epoch KẾ Tiếp dẫn xuất từ băm **nội dung** của checkpoint
  (có khóa, loại trừ bộ chữ ký — hai góc nhìn giữ các tập con
  quorum khác nhau của cùng checkpoint bầu ra cùng ủy ban).

### Rút ủy ban (bootstrap, độ mới, suy giảm)

`committee(recovery)`:

1. **Bootstrap** (chưa có checkpoint hoàn tất): TOÀN BỘ tập hợp
   ký — một ủy ban rút từ một góc nhìn cục bộ sẽ lệch giữa các
   nút trễ (nguyên nhân gốc của các đơ tính cuối); toàn bộ tập
   hợp là tất định theo chuỗi. Lựa chọn nghiêm ngặt tiếp tục
   ở lần tính cuối đầu tiên.
2. **Chế độ hoàn tất**: rút từ tập con **còn mới** của tập hợp
   đóng băng (RFC proof-of-service §5.2) — mỗi ghế (khóa chủ)
   được phân giải tới vận hành viên HIỆU LỰC của nó bởi trạng
   thái tại đỉnh (ủy quyền `SetTldOperator`/`SetDomainOperator`
   = một sự kiện on-chain gần đây, hay khóa chủ khi tự vận
   hành); ghế tham gia DƯỚI KHÓA HIỆU LỰC của nó (đó là cái ký),
   với điều kiện nó có một **heartbeat còn mới** (cửa sổ
   trượt). Khóa bị cấm vẫn bị loại.
3. **Suy giảm an toàn**: ít hơn `MIN_FINALITY_COMMITTEE_SIZE`
   (4) vận hành viên còn mới → rút từ TOÀN BỘ tập hợp
   (bootstrap nguội, sự cố vận hành viên tổng thể — tính cuối
   không bao giờ mất về mặt cấu trúc).

`allowed_producers(block_ts)` (tạo khối) theo cùng lược đồ:
toàn bộ tập hợp lúc bootstrap; ủy ban được bầu + các lần rút
khôi phục (rút k mở khóa sau k × `epoch_secs` thời gian đồng
thuận) trong chế độ hoàn tất.

Quorum checkpoint là `quorum_for(kích thước ủy ban)`
(floor(2n/3)+1); một ủy ban dưới `MIN_FINALITY_COMMITTEE_SIZE`
không bao giờ hoàn tất (canh BFT).

## Hoãn cho đồng thuận

Cố ý không định nghĩa trong crate này:

- **mempool**: vào/sắp xếp tinh tế (phí, thay thế) ở lại relay
  (M4 mang nắp theo-tên-miền + chống phát lại TXID cửa sổ —
  `REPLAY_WINDOW_BLOCKS = 256`);
- **sắp xếp toàn cục**: thứ tự mempool là một sắp cục bộ tất
  định (relay), không phải một thẩm quyền toàn cục;
- **thẩm quyền timestamp** ngoài GC: các hết hạn (M8b) dùng
  timestamp của khối cha; các quy tắc thời gian khác (chống
  phát lại tinh tế) còn ở đồng thuận;
- lưu trữ phân tán: trạng thái trong bộ nhớ; backend
  (`scone-storage`, redb) áp dụng các quy tắc này mà không nhân
  đôi chúng.

Đã bàn giao từ M8b (trước đây bị hoãn): PoW đăng ký theo từng
mạng (`RegisterTld` luôn, `RegisterDomain` trên TLD mở),
hết hạn/gia hạn 1 năm + hoãn 30 ngày + chân trời 3 năm, GC tất
định tại timestamp khối cha, phát lại chéo mạng bất khả
(genesis + trường ký + digest PoW).

## Mô phỏng phân tán tất định (P1.6)

`crates/scone-blockchain/src/sim/` chứa một bộ mô phỏng mạng
**tất định** (`SimNet`): N `Blockchain` thật (không mock) được
lái bởi một **đồng hồ ảo** và một **hàng đợi sự kiện** ưu tiên
theo `(tick, thứ tự chèn)`. Không socket, không async, không
đồng hồ tường — mọi lần rút (mất, độ trễ, nhân đôi) là hàm
thuần của `(seed, số lần gửi)`, nên một kịch bản với seed cố
định tái lập được byte cho byte.

### Mô hình

- **Vận chuyển**: mọi lần phát trải qua mất xác suất (cấu hình
  được theo ‰), độ trễ nhân tạo bị chặn (`latency_ticks`), nhân
  đôi khả dĩ (một bản sao trễ), và bản đồ phân vùng hiện hành
  (thông điệp liên nhóm bị mất). Các nhân đôi không bao giờ
  đẩy đồng hồ logic và không kích hoạt gửi mới: dòng sự kiện
  thật giống hệt có hay không nhân đôi (hợp đồng tất định,
  chứng minh bởi kiểm thử `duplicate_and_delay`).
- **Gossip lan truyền**: một nút phát lại tới các peer kết nối
  (cùng phân vùng) một khối/giao dịch/checkpoint chỉ khi thấy
  lần đầu — quy tắc relay H2 (cắt vòng). Một khối đỗ (orphan)
  KHÔNG BAO GIỜ được phát lại; các khoảng trống nhận được được
  lấp bởi đường đồng bộ (`GetBlocks`), như relay.
- **Tạo khối**: một bước tạo toàn cục mỗi khoảng; mỗi nhóm
  phân vùng tạo tối đa MỘT khối mỗi vòng, bởi nút được ủy quyền
  giữ chiều cao tốt nhất. Trong bootstrap (trước lần tính cuối
  đầu tiên) người produce được ghim vào nút thấp nhất: tập
  người produce được ủy quyền dẫn xuất từ trạng thái (tập các
  tên miền còn sống), hai góc nhìn hơi khác nhau bầu các ủy ban
  khác nhau và một khối chính đáng cục bộ sẽ là
  `InvalidProducer` nơi khác — một lệch vĩnh viễn. Sau lần tính
  cuối đầu tiên ủy ban được đóng băng trong cơ sở tính cuối
  (giống hệt trên mọi nút hội tụ) và việc tạo khối tự do.
- **Tính cuối**: tại mọi bội số của `sign_interval` chiều cao,
  anchor được chỉ định của ủy ban ký nội dung tất định của đỉnh
  của nó qua một `SignerGuard` chống-crash thật (một khóa = một
  phiếu mỗi epoch, trạng thái fsync trước khi khuếch tán); các
  thành viên ủy ban khác ký khi nhận (sau khi xác minh khối
  được tham chiếu đúng là khối chuẩn của họ ở chiều cao đó —
  canh cấp relay) và phát lại tổng hợp làm giàu. Đạt quorum,
  `accept_checkpoint` hoàn tất. Mỗi nút cam kết với đề xuất đầu
  tiên thấy theo từng epoch (canh bootstrap chống hoàn-tất-kép:
  hai đề xuất cạnh tranh cùng epoch với các người ký tách rời
  không thể cùng tích lũy).
- **Crash/khởi động lại**: một nút crash mất toàn bộ trạng thái
  RAM (chuỗi, mempool, đỗ, góc nhìn) nhưng giữ thư mục
  signer-guard trên đĩa, đúng như một tiến trình thật; nó tái
  đồng bộ bằng cách phát lại các khối chuẩn do peer phục vụ
  (`GetBlocks`/`GetCheckpoints`).

### Các kịch bản được phủ (`sim::tests`)

| Kiểm thử | Chứng minh điều gì |
|---|---|
| `convergence_simple` | 8 nút, mất 5 ‰, độ trễ 1–5 tick, 500 khối: cùng đỉnh, cùng `state_root_smt` mọi nơi; tính cuối hoạt động; không bao giờ hai checkpoint không tương thích mỗi epoch. |
| `partition_then_heal` | phân vùng 4a/4b ~300 khối: cả hai bên produce, chữa lành → hội tụ với reorg của nhánh thua; các cửa sổ checkpoint giao nhau không bao giờ mâu thuẫn một epoch chung (trước và sau chữa lành). |
| `crash_restart` | Crash ở h≈200, khởi động lại ở h≈400: tái đồng bộ bằng cách phát lại các khối của mạng — cùng chiều cao, đỉnh, trạng thái; giao dịch sau-crash hiển thị với nút hồi sinh. |
| `duplicate_and_delay` | nhân đôi 10 ‰: chiều cao cuối, gốc trạng thái, và số checkpoint hoàn tất giống hệt chạy không nhân đôi (lũy tính: TxReplay/các tập-đã-thấy hấp thụ bản trùng). |
| `concurrent_tx` | Hai chủ đồng thời trên cùng tên miền: đúng một bản cập nhật thắng (chủ — `NotOwner` cho kẻ xâm nhập, từ chối có kiểu rõ), không bao giờ áp dụng trạng thái kép, tx thua không vào chuỗi nào. |

### Tham số và biên thời lượng

Giá trị mặc định: `produce_interval` 7 tick, `sign_interval` 16
chiều cao, `sync_interval` 29–60 tick (41 mặc định, hạ trong
một số kiểm thử), `horizon` 200k–500k tick ảo. Mọi kiểm thử nhúng
một canh đồng hồ-tường < 60 s ở chế độ debug
(`assert_under_60s`). Chi phí trội là hạt nhân xác thực (~9 ms
mỗi `push_block_with_gc` rỗng ở debug, Ed25519 chưa tối ưu):
các kịch bản giữ dưới ~8 nút × 500 khối.

### Các giới hạn

- **Không tầng vận chuyển thật**: bộ này luyện logic chuỗi dưới
  tình trạng giao delivering bất lợi, không phải swarm (libp2p,
  request-response, backpressure, khám phá). Các kiểm thử e2e
  thật (`crates/scone-network/tests/`) vẫn là bằng chứng vận
  chuyển.
- **Đồng bộ theo lô trong RAM**: các khối phục vụ đến từ cửa sổ
  RAM (các kịch bản giữ dưới `RAM_WINDOW_BLOCKS`); đường
  `BlockPruned` + nạp-lại-từ-kho không được mô phỏng (do các
  kiểm thử đơn vị của chuỗi phủ).
- **Mô hình anchor trung thực**: các anchor chỉ ký nội dung kiểm
  chứng được đối với chuỗi chuẩn riêng của chúng; equivocation
  byzantine (ký đôi cố ý) bị `SignerGuard` ngăn nhưng không
  được luyện như một tấn công chủ động.
- **Một phân vùng tại một thời điểm**: bản đồ nhóm là toàn cục
  và được thay đổi tuần tự bởi bộ lái; các phân vùng chồng lấn/
  tiến hóa liên tục không được mô hình hóa.

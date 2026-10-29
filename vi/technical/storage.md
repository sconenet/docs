# Storage — lưu trữ cục bộ của nút (redb)

> Tài liệu chuẩn. Bất kỳ khác biệt nào giữa tài liệu này và
> `crates/scone-storage/src/` là một bug (xác định nguồn chân lý,
> sửa nó). Tham chiếu triển khai: crate `scone-storage`
> (định dạng v1), backend `redb` 4.2.

## Mục đích

Lưu trữ cục bộ nhúng của nút: các khối chuẩn, trạng thái tên
miền, cache DHT, siêu dữ liệu. Kho là một **sự lưu bền vững**,
không bao giờ một thẩm quyền: trạng thái RAM ([`ChainState`],
nguồn của đồng thuận) được dựng lại lúc khởi động từ kho, và mỗi
lần ghi là một **delta** (các tên miền bị thay đổi bởi một khối),
không bao giờ một bản sao đầy đủ của trạng thái.

Ràng buộc xuyên suốt (hướng người dùng, mọi mốc): **dấu chân bộ
nhớ tối thiểu**. Các khối lượng lớn (khối, trạng thái tên miền,
cache DHT) sống trong redb và được truy cập qua con trỏ/phạm vi
— không bao giờ nạp đầy vào RAM, không bao giờ nhân bản vô ích.

## Biên giới: trait `NodeStore`

`crates/scone-storage/src/lib.rs` định nghĩa bề mặt lưu trữ duy
nhất nhìn thấy được bởi phần còn lại của mã (blockchain, relay
M4 tương lai):

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — nối thêm nguyên tử
  (xem dưới);
- `block_at_height(h)` / `block_by_hash(hash)` — đọc điểm;
- `tip() -> (height, hash)` — O(1), đọc từ `meta`;
- `put_domain_state` / `domain_state` / `domain_count` — bộ đếm
  **được duy trì** (`meta["domain_count"]`), không bao giờ một
  `COUNT(*)`;
- `iterate_domains(after, max) -> DomainPage` — phân trang theo
  con trỏ (lô 100 mặc định), thứ tự byte tăng của `DomainId`;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  cùng hợp đồng cho registry TLD (`tld_count` được duy trì,
  con trỏ bị chặn) (M7d);
- `put_dht_cache` / `dht_cache` — byte `SignedDnsRecord` mờ;
- `resolve_name(canonical)` / `resolve_tld(tld)` — chỉ mục
  Name→Id bền vững (P0.2), tra cứu điểm O(1), không lặp qua tên
  miền (mặc định `None` = backend không chỉ mục, bên gọi quay về
  `iterate_domains` hay trạng thái chuỗi);
- `tld_floors_all` / `snapshot_tld_floors_all` — đọc các sàn đời
  sống của các TLD mở (tùy chọn chủ: các bảng riêng, sống
  `tld_floors_v1` và đóng băng `snapshot_v1_floors`);
- `rewind_to_height(h)` — reorg: gỡ các khối trên `h` và đưa
  `tip`/`tip_height` về `h` trong MỘT giao dịch (tiền đề của
  `persist_state_snapshot`, xem dưới);
- `meta_get` / `meta_set`;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` — đọc
  snapshot trạng thái lúc khởi động (M6b; mặc định rỗng =
  backend không snapshot, `load_chain` khi đó quay về nạp đầy).

Quy tắc:

- **Không kiểu redb trong API**: chỉ byte (`&[u8]`, `[u8; 32]`)
  và các kiểu core. Backend thay thế được.
- **Không async, không khả biến nội**: ghi lấy `&mut self`
  (redb tuần tự hóa các trình ghi), đọc lấy `&self` (một giao
  dịch đọc ngắn mỗi lời gọi).
- API không bao giờ xác thực lại khối: băm truyền cho
  `append_block*` là băm **được tính lại** do bên gọi cung cấp
  (`Blockchain::push_block`).

## Lược đồ bảng redb

| Bảng | Khóa (sắp) | Giá trị |
|---|---|---|
| `blocks_by_height` | chiều cao `u64` (b-tree có thứ tự) | `hash(32) ‖ height(8 BE) ‖ canonical block` |
| `blocks_by_hash` | băm khối (32 B) | mã hóa chuẩn của khối |
| `domains` | `DomainId` (32 B, thứ tự byte) | `DomainStateBytes` (dưới) |
| `tlds` | `TldId` (32 B, thứ tự byte) | `TldStateBytes` (dưới, M7d) |
| `dht_cache` | `DomainId` (32 B) | byte `SignedDnsRecord` (mờ) |
| `names_v3` (P0.2) | khóa chỉ mục (32 B, thứ tự byte) | id đích (32 B) |
| `name_reverse_v3` (P0.2) | id đích (32 B, thứ tự byte) | `tag(1) ‖ canonical name` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta` (M6b) | `u64` = 0 (khe đơn) | `height(8 BE) ‖ tip_hash(32)` = đúng 40 B |
| `snapshot_v1_domains` (M6b) | `DomainId` (32 B, thứ tự byte) | `DomainStateBytes` (cùng định dạng `domains`) |
| `snapshot_v1_tlds` (M6b) | `TldId` (32 B, thứ tự byte) | `TldStateBytes` (cùng định dạng `tlds`) |
| `tld_floors_v1` (tùy chọn chủ) | `TldId` (32 B, thứ tự byte) | `u64` (sàn hết hạn, BE ngầm của redb) |
| `snapshot_v1_floors` (M6b) | `TldId` (32 B, thứ tự byte) | `u64` — bản đóng băng của các sàn (trạng thái TẠI snapshot, không phải đỉnh) |

Khóa `meta` thêm (P0.3/P0.4):

| Khóa | Giá trị |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — dấu hiệu của một snapshot ĐÃ XÁC MINH được nhập (dành riêng, như `tip`) |

Các khóa `meta`:

| Khóa | Giá trị |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | băm (32 B) của khối đỉnh |
| `tip_height` | `u64 BE` (làm `tip()` O(1)) |
| `domain_count` | `u64 BE` (bộ đếm được duy trì) |
| `tld_count` | `u64 BE` (bộ đếm được duy trì, M7d) |

Header `hash ‖ height` của `blocks_by_height` làm mỗi chỉ mục
tự chứa (kiểm chứng được mà không đọc khối); cả hai chỉ mục được
cập nhật trong cùng giao dịch.

### Quy ước genesis

Khối genesis là tất định (hằng số giao thức) và **không bao giờ
được lưu** dưới dạng byte. Một kho mới có `tip == (0,
genesis_hash)`; khối được nối đầu tiên có chiều cao 1. Băm
genesis luôn được dẫn xuất lại
(`scone_blockchain::genesis_hash()`), không bao giờ đọc.

## Định dạng mã hóa `DomainStateBytes` (M8b)

65 hay 97 byte tùy thẻ phiên bản dẫn đầu, số nguyên big-endian
độ-cố-định (chuẩn, so sánh được byte-cho-byte):

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) ‖ mutations(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash present → exactly 97 bytes
tag = 0x02 : record_hash absent  → exactly 65 bytes
```

`mutations` là bộ đếm biến đổi monotonic của namespace (thế hệ
chống phát lại CAS — xem `transactions.md`, Bộ đếm biến đổi). Các
mã hóa M8b (57/89 byte, không bộ đếm) **vô hiệu**: giải mã nghiêm
ngặt → `Corrupted` (reset testnet; một kho tiền-bộ-đếm tái tạo qua
phát lại).

- `registered_at` / `valid_until` (M8b): hết hạn của đăng ký
  (1 năm lúc tuyên bố, gia hạn được, xem
  `/docs/en/technical/blockchain.md`).

Các mã hóa M7 (41/73 byte, không hết hạn) **vô hiệu** từ M8b:
giải mã nghiêm ngặt → `Corrupted`. (Một kho M7 tái tạo qua phát
lại: `load_chain_replay`.)

- `owner`: `OwnerId` (32 B);
- `sequence`: sequence `UpdateDomain` áp dụng cuối (0 ngay sau
  một `RegisterDomain`);
- `record_hash`: cam kết on-chain hiện tại của bộ bản ghi DNS
  (`None` chừng nào chưa có `UpdateDomain` nào được áp dụng).

Giải mã **nghiêm ngặt** (`DomainStateBytes::decode`): thẻ lạ,
sai chiều dài, hay byte thừa → `StorageError::Corrupted`, không
bao giờ panic.

## Định dạng mã hóa `TldStateBytes` (tùy chọn chủ)

Chiều dài biến thiên theo biến thể enum — 50 byte cho một TLD
**riêng tư**, 10 cho một TLD **mở** (không sequence mà cũng
không băm bản ghi):

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
                [‖ mutations(8)]
```

- `flag` (tùy chọn chủ, enum `TldState`): `0x00` =
  `TldState::Open` (không ai — tự phục vụ + PoW, không byte
  chủ), `0x01` = `TldState::Private` (32 byte chủ + 8 byte bộ
  đếm theo sau).
  Chuẩn nghiêm ngặt — mọi byte khác → `Corrupted`. Bất biến
  "mở ⟺ không chủ" là CẤU TRÚC: hai trạng thái không thể bị
  nhầm lẫn.
- **`until` (8 byte, BE64)**: đồng hồ riêng của TLD — cuối hạn
  riêng tư (tuyên bố/gia hạn + 3 năm) hay lịch chết mở (sàn
  1 năm lúc sinh, được đẩy bởi mọi tên miền còn sống). Nó lại
  là một thuộc tính của mục (thay đổi vòng đời TLD). Bảng dẫn
  xuất `tld_floors_v1` (+ bản đóng băng `snapshot_v1_floors`)
  vẫn theo dõi các sàn, dựng lại khi phát lại và khôi phục lúc
  khởi động — một nút khởi động lại giết cùng các TLD như một
  nút liên tục.

Các mã hóa M7d (33 byte, thẻ `0x01`), M8b (34 byte, thẻ
`0x02`, boolean `open`), thẻ cũ `0x03` (`until` nhúng) và các
bố cục thẻ `0x04` tiền-bộ-đếm (42/2 byte, không bộ đếm)
**vô hiệu**: giải mã nghiêm ngặt → `Corrupted` (trước-mainnet,
tái tạo qua phát lại là đường di trú).

Thẻ chỉ tồn tại cho sự tiến hóa định dạng: một bố cục tương lai
nâng thẻ và các trình đọc cũ thất bại với `Corrupted` thay vì
đoán. Giải mã **nghiêm ngặt** (`TldStateBytes::decode`): thẻ
lạ, sai chiều dài, byte thừa, hay cờ không chuẩn →
`StorageError::Corrupted`, không bao giờ panic.

## Bảo đảm nguyên tử

`append_block_with_state` ghi trong **một** `WriteTransaction`
redb duy nhất:

1. khối trong `blocks_by_height` (kiểm tra trước: chiều cao
   không được tồn tại với nội dung khác, nếu không
   `StorageError::Conflict`);
2. khối trong `blocks_by_hash`;
3. mọi delta trạng thái tên miền (`domains`);
4. bộ đếm `domain_count` (tăng theo số tên miền **mới** duy
   nhất; một `UpdateDomain` của một tên miền sẵn có không tăng
   nó);
5. mọi delta registry TLD (`tlds`, M8b: tuyên bố, chuyển,
   mở/đóng) — **trong cùng giao dịch**: một khối được ghi không
   kèm các thay đổi TLD của nó là bất khả về cấu trúc;
6. bộ đếm `tld_count` (chỉ TLD mới);
7. mọi **lần gỡ** (M8b): các tên miền hết hạn bởi GC tất định
   (`removed_domains`) và các TLD bị thu hồi
   (`removed_tlds`) rời kho trong cùng giao dịch, bộ đếm giảm —
   một lần khởi động lại không bao giờ hồi sinh một đăng ký hết
   hạn;
8. các delta **chỉ mục tên** của P0.2 (upsert và gỡ, xem dưới)
   — cùng bảo đảm: một crash không bao giờ để lại một mục chỉ
   mục trỏ tới một tên miền thiếu;
9. `tip` và `tip_height`.

redb commit kiểu-WAL: một crash giữa chừng ghi để lại trạng
thái **trước** mạch lạc. **Một khối được ghi không kèm trạng
thái của nó do đó là bất khả về cấu trúc** — đây là bất biến
chìa khóa của mốc.

Các bảo đảm thêm:

- **Đơn điệu**: nối ở chiều cao ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, không có gì được ghi
  (kiểm tra *trước* khi mở giao dịch ghi);
- **Lũy tính**: nối lại khối đỉnh hiện tại với cùng cặp
  `(height, hash)` là một no-op (đường nhanh so chiều cao và
  băm; khi bằng chiều cao, byte khối giống nhau cũng là no-op
  âm thầm, byte khác là `Conflict`);
- **Cơ sở dữ liệu hỏng**: tệp cắt cụt/nhiễm → lỗi có kiểu
  (`StorageError::Database`/`Corrupted`) lúc mở hay khi đọc,
  không bao giờ panic (đã kiểm thử);
- **Phiên bản định dạng**: `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (một đường di trú tường
  minh là bắt buộc, không bao giờ đoán).

## Tích hợp blockchain (`integration.rs`)

Phân vai: [`ChainState`] trong RAM = thẩm quyền đồng thuận; kho
= lưu bền vững. Chiến lược chọn (đơn giản nhất, như được ghi
tài liệu): **lưu trạng thái ở mỗi khối** — `store_block` ghi
khối + trạng thái của các tên miền **bị chạm bởi khối đó**
(delta). Phát lại đầy đủ vẫn khả năng và phục vụ như đường
kiểm soát/sửa chữa.

- `store_block(store, chain, block, hash)`: mã hóa khối chuẩn,
  đọc trạng thái cuối của các tên miền bị chạm VÀ các TLD được
  tuyên bố từ trạng thái RAM (đã được `push_block` cập nhật),
  nối thêm nguyên tử.
- `load_chain(store)`: **không phát lại** — đọc đỉnh + khối
  đỉnh (đọc khối O(1)), khôi phục trạng thái tên miền theo trang
  100 (`DOMAIN_PAGE`) **và registry TLD theo trang 100**
  (`TLD_PAGE`, M7d — đóng cửa sổ khởi động lại: thiếu nó, một
  nút khởi động lại quên mọi TLD đã tuyên bố và từ chối mọi
  `RegisterDomain` dưới TLD đó với `UnknownTld`, dù đĩa của nó
  giữ trạng thái đầy đủ). Chuỗi RAM chỉ giữ genesis + đỉnh; các
  khối lịch sử ở lại kho và được phục vụ từ đó. Từ M8,
  `load_chain` sau đó dựng lại **chỉ mục chống phát lại TXID**
  (xem dưới).
- `load_chain_replay(store)`: phát lại mọi khối được lưu từng
  cái một (không bao giờ tất cả trong RAM) với xác thực đầy đủ;
  đỉnh thu được phải bằng đỉnh đã lưu, nếu không `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8): quét lại
  `min(REPLAY_WINDOW_BLOCKS, tip_height)` khối bền vững cuối
  (một khối trong RAM mỗi lần) và chèn lại mọi TxId vào chỉ mục
  chuỗi **tại chiều cao bao hàm gốc**, qua API công cộng
  `Blockchain::rebuild_replay_index` (không truy cập trường
  riêng). Khối thiếu dưới đỉnh hay không giải mã được trong
  cửa sổ → `Corrupted` (những byte đó đã phục vụ dựng chuỗi:
  thất bại ở đây báo hiệu một kho hư).
- `touched_domains(block)`: id của các tên miền bị thay đổi bởi
  các giao dịch của khối, khử trùng, thứ tự khối.
- `touched_tlds(block)`: id của các TLD bị đột biến bởi khối
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  khử trùng, thứ tự khối (M8b).
- `store_block_with_removals(...)`: biến thể của `store_block`
  nhận thêm các tên miền bị gỡ bởi GC của khối (M8b) cho lần
  ghi nguyên tử mô tả trên. Relay cấp danh sách này từ
  `Blockchain::push_block_with_gc` (kết quả áp dụng phơi bày
  `gc_removed_domains`). **Trục xuất GC của cache DHT**: các
  mục `dht_cache` của các tên miền bị gỡ bị xóa trong CÙNG giao
  dịch với khối (các delta nối thêm mang bảng `dht_cache`) —
  một bản ghi DHT không bao giờ sống lâu hơn tên miền nó phục
  vụ, các mục hết hạn không bao giờ tích lũy vĩnh viễn (kiểm
  thử: `gc_evicts_the_dht_cache_entry`).

Relay (M4) sẽ dùng: `load_chain` lúc khởi động, rồi cho mỗi
khối được chấp nhận `push_block` (RAM) rồi `store_block`
(đĩa).

## Lưu bền vững reorg (`rewind_to_height` + `persist_state_snapshot`)

Một reorg (chuyển sang nhánh chị em mạnh hơn) không thể đi qua
phép nối đơn điệu — ghi hậu tố thắng sẽ thất bại với
`NonMonotonicHeight`. Đường riêng (sửa thiểu số-dính, relay +
kho):

1. `rewind_to_height(fork_height)` — MỘT giao dịch redb gỡ các
   khối trên điểm rẽ và đưa `tip`/`tip_height` về (chỉ mục tên
   và trạng thái sống được sửa ở bước kế);
2. `persist_state_snapshot(store, chain)` — ghi lại các bảng
   sống (`domains`, `tlds`, chỉ mục tên, sàn TLD) từ bản xuất
   trạng thái RAM sau-reorg: các bảng sống KHÔNG BAO GIỜ mô tả
   nhánh THUA lúc khởi động;
3. nối lại đơn điệu của hậu tố được nhận nuôi (delta thường).

Một thất bại lưu bền vững trên đường này được ghi nhật ký và suy
giảm (kho tự chữa lúc khởi động qua phát lại) — không bao giờ
nghiêm trọng.

## Snapshot trạng thái lúc khởi động (M6b)

`load_chain` như mô tả trên vẫn là O(kích thước trạng thái):
không khối nào được phát lại, nhưng toàn bộ bảng `domains` được
đọc lại. Chính đường trạng thái-lưu-theo-từng-khối làm cho phát
lại trở nên không cần thiết. Snapshot M6b giải quyết một chi
phí khác: trường hợp trạng thái RAM phải được dựng lại **bằng
phát lại** (sửa chữa, kiểm soát, hay một backend không có trạng
thức lưu theo khối). Nó cung cấp khởi động trong **O(tip − H)**
— chỉ phát lại hậu tố trên snapshot.

### Ghi

- **Khoảng**: `SNAPSHOT_INTERVAL = 64` (hằng số công cộng của
  `scone-storage`). Bộ kích hoạt là phép nối khối
  `k·SNAPSHOT_INTERVAL + 1`: giao dịch đó đóng băng trạng thái
  các bảng sống **như nó có trước khi áp dụng khối đó**, tức
  trạng thái chuẩn sau khối `k·SNAPSHOT_INTERVAL`.
- **Không bao giờ tại đỉnh**: snapshot do đó nằm tại `tip − 1`
  gần nhất, luôn dưới nghiêm ngặt đỉnh tại thời điểm được ghi.
  Một crash ngay sau không bao giờ để lại một snapshot "trên"
  chuỗi được lưu.
- **Nguyên tử**: sự đóng băng (xóa sạch + sao chép các bảng
  `domains`/`tlds` sang `snapshot_v1_*`, ghi `snapshot_v1_meta`)
  diễn ra trong CÙNG giao dịch redb với phép nối của khối kích
  hoạt — một crash để lại hoặc snapshot cũ nguyên vẹn, hoặc cái
  mới đầy đủ với khối của nó (WAL redb).
- **Khe đơn**: mọi snapshot THAY THẾ cái trước (bảng rỗng rồi
  sao chép lại) — không tích lũy, không xén để quản lý, dấu
  chân đĩa bị chặn bởi ~2× trạng thái.
- Định dạng các trạng thái đóng băng đúng là của
  `DomainStateBytes`/`TldStateBytes` (tái dùng, không mã hóa
  mới); chỉ dòng `snapshot_v1_meta` là định dạng mới
  (`height(8 BE) ‖ tip_hash(32)`, giải mã nghiêm ngặt: mọi
  chiều dài khác → `Corrupted`).

### Đọc lúc khởi động (`load_chain`)

1. Đỉnh được xác thực như luôn (băm tính lại từ header khối đỉnh
   so với `meta["tip"]`).
2. Nếu `snapshot_meta()` tồn tại, chiều cao `H` của nó thỏa
   `0 < H < tip_height`, và băm TÍNH LẠI của khối lưu tại `H`
   bằng `tip_hash` của snapshot (**neo** — một snapshot lạ với
   chuỗi lưu, bị sửa, hay từ tệp khác bị phát hiện ở đây),
   trạng thái đóng băng được khôi phục theo trang 100 rồi chỉ
   các khối `H+1..=tip` được phát lại (xác thực đầy đủ, giống
   khối sống).
3. Đỉnh đạt được bởi phát lại phải bằng đỉnh đã lưu.
4. **Mọi thất bại trên đường này (snapshot thiếu, chiều cao
   vô dụng, neo vô hiệu, byte không giải mã được, phát lại
   thất bại) âm thầm suy giảm về nạp đầy pre-M6b** (đọc các
   bảng sống, không phát lại) — một snapshot xấu không bao giờ
   làm hỏng khởi động. Tương thích ngược là trường hợp riêng
   "không snapshot": một kho ghi trước M6b (không có các bảng
   `snapshot_v1_*`) nạp giống hệt, `snapshot_meta()` là
   `None`.

Hậu tố được phát lại bị chặn bởi `SNAPSHOT_INTERVAL` (tối đa 64
khối lúc kích hoạt + các khối đào từ đó): khởi động là
O(tip − H) với H bảo đảm gần.

## Chỉ mục chống phát lại TXID dựng lại lúc khởi động (M8)

Chỉ mục chống phát lại (`included`, TxId → chiều cao bao hàm,
cửa sổ `REPLAY_WINDOW_BLOCKS = 256` — xem
`/docs/en/technical/blockchain.md`) là một **hàm thuần của
chuỗi chuẩn**: nó không bao giờ được lưu. Vì `load_chain`
không phát lại lịch sử, một chuỗi khôi phục khởi động trước M8
với chỉ mục **rỗng** — một nút khởi động lại chấp nhận bao hàm
lại bất kỳ giao dịch nào còn trong cửa sổ phát lại của nó
(hạn chế được ghi nhận; quy tắc trạng thái vẫn là rào chắn duy
nhất).

M8 đóng cửa sổ này: sau khôi phục (đường đầy ĐO snapshot + phát
lại hậu tố), `load_chain` gọi
`rebuild_replay_index_from_store`, đọc lại
`min(REPLAY_WINDOW_BLOCKS, tip_height)` khối bền vững cuối —
một khối trong RAM mỗi lần, bộ nhớ bị chặn như mọi nơi — và
chèn lại mọi TxId tại **chiều cao gốc** của nó. Một kho với
ít hơn 256 khối do đó dựng lại toàn bộ chỉ mục.

Tính chất:

- **Quyết định giống nút sống**: cùng các mục, cùng quy tắc xén
  (`h > tip − 256`) — một nút khởi động lại và một nút không
  bao giờ dừng từ chối cùng các phát lại (đã kiểm thử: số đếm
  chỉ mục chính xác và bằng với chỉ mục sống đã xén).
- **Ngữ nghĩa xén không đổi**: một giao dịch cũ hơn cửa sổ vẫn
  được chỉ mục nhận (rồi thất bại quy tắc trạng thái nếu áp
  dụng) — y như trước.
- **Snapshot KHÔNG đóng băng chỉ mục**: phát lại hậu tố của
  snapshot dựng lại các mục trên H, quét lại M8 thêm phần cửa
  sổ DƯỚI H (cùng hợp đồng như owner_keys/grace: dựng lại,
  không bao giờ lưu).
- **Chi phí**: ≤ 256 lần đọc khối điểm lúc khởi động, O(cửa
  sổ) — độc lập với tuổi chuỗi.
- Một khối thiếu hay không giải mã được **trong cửa sổ** làm
  khởi động thất bại với `Corrupted` (những byte đó đã phục vụ
  dựng chuỗi); ngoài cửa sổ, các khối lịch sử không bị chạm.

`load_chain_replay` (không tin) dựng lại chỉ mục cùng cách qua
phát lại đầy đủ của nó — hai đường hội tụ.

## Chỉ mục Name → Id bền vững (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ canonical name)`
không khả nghịch: phân giải một tên (relay/DNS) đòi hỏi một chỉ
mục bền vững. P0.2 thêm nó vào kho, với cùng kỷ luật giao dịch
như phần còn lại.

### Lược đồ

Hai bảng đồng hành, ghi và xóa cùng nhau:

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ canonical name)  (32 B)
                val  = target id (32 B : DomainId or TldId)

name_reverse_v3 key  = target id (32 B)
                val  = tag(0x01 domain | 0x02 TLD) ‖ canonical name
```

- Khóa `names_v3` là một **băm của tên, không phải tên văn bản
  rõ** (bảng chỉ mục gọn, không gossip tên qua dump). Khóa là
  **hàm thuần của tên**: chỉ mục dựng lại được bất kỳ lúc nào
  từ chuỗi (`rebuild_name_index`), không siêu dữ liệu thêm nào
  được lưu cho nó.
- `name_reverse_v3` lưu tên (đúng một lần) vì các đường gỡ (GC
  tên miền hết hạn, `RevokeTld`) chỉ biết **id**: khóa đã băm
  cần xóa phải được tìm bằng cách nào đó. Thẻ phân biệt hai
  không gian tên.
- Tách nghiêm ngặt: `resolve_name` KHÔNG BAO GIỜ trả lời một
  mục TLD và `resolve_tld` không bao giờ một mục tên miền (thẻ
  của bảng đảo quyết định; một `DomainName` có ≥ 2 nhãn, một
  TLD có 1 — hai phép dẫn xuất khóa chia sẻ không gian
  `names_v3` nhưng không chia sẻ kết quả).
- `DomainId` giữ 32 byte, không bao giờ giảm (quy tắc cứng dự
  án: danh tính đồng thuận, kháng va chạm).

### Nhất quán giao dịch

Các upsert (`RegisterDomain`, `AssignDomain`, `RegisterTld` —
tên mang văn bản rõ bởi các giao dịch) và các lần gỡ (GC, thu
hồi — id trần) được mang bởi `StateDelta` của khối
(`name_upserts` / `name_removals`) và áp dụng trong **cùng giao
dịch redb** với các lần ghi khối và trạng thái. Một crash do đó
không thể để lại một mục chỉ mục trỏ tới một tên miền thiếu
(không ma), cũng không một tên miền đăng ký thiếu mục của nó.
Một lần nối thất bại không ghi gì (đã kiểm thử). Các biến thể có
tên `put_domain_state_with_name` / `put_tld_state_with_name` /
`remove_domain_state` cung cấp cùng tính nguyên tử cho công cụ
sửa chữa.

`store_block` / `store_block_with_removals` dẫn xuất các delta
tên từ CÙNG các giao dịch của khối — không tham số thêm tại giao
diện relay.

### Dựng lại (`RedbStore::rebuild_name_index`)

Chỉ mục là hàm thuần của chuỗi chuẩn, như chỉ mục chống phát
lại M8. `rebuild_name_index` dựng lại nó từ **các khối được
lưu** (`DomainStateBytes` KHÔNG chứa tên — chỉ
chủ/sequence/hết hạn — nên nguồn chân lý là dòng giao dịch,
nơi `RegisterDomain` và `AssignDomain` mang tên chuẩn văn bản
rõ):

1. lượt 1: quét `blocks_by_height` genesis→đỉnh, một khối trong
   RAM mỗi lần; thu thập các ràng buộc `tên → id` thấy trong
   `RegisterDomain`/`AssignDomain`/`RegisterTld`;
2. lượt 2: lọc chỉ các đích **còn sống** (trạng thái được lưu
   là thẩm quyền) — một tên miền đăng ký rồi GC/hết hạn không
   được lập chỉ mục lại;
3. lượt 3: xóa + ghi lại đầy đủ cả hai bảng trong MỘT giao dịch
   (crash → chỉ mục cũ hay mới đầy đủ).

Một khối thiếu hay không giải mã được → `Corrupted` (cùng byte
đã dựng chuỗi). Chi phí là O(số tên riêng biệt thấy) trong RAM
(≤ 253 B mỗi tên) và O(số khối) đọc — một công cụ sửa chữa,
không phải đường khởi động.

### Chỉ mục KHÔNG phải là gì

- Không phải thẩm quyền: trạng thái chuỗi RAM vẫn là thẩm quyền
  duy nhất; chỉ mục là một cache dựng lại được (một lệch là một
  bug tính giao dịch, phát hiện được bằng
  `rebuild_name_index` so sánh với nó).
- Không đóng băng trong snapshot M6b: các bảng
  `names_v3`/`name_reverse_v3` là sống và theo các delta của
  từng khối; snapshot trạng thái không nhúng chúng (dựng lại
  bởi delta khi tiếp tục, hay `rebuild_name_index` để sửa).

### Snapshot không cam kết gì

Trạng thái RAM `ChainState` mang các cấu trúc nội bộ không được
đồng thuận cam kết (khóa chủ, cửa sổ hoãn). Chúng KHÔNG được
đóng băng: phát lại hậu tố dựng lại chúng bằng cách áp dụng các
giao dịch (tất định, cùng nội dung với phát lại đầy đủ cho
cửa sổ liên quan). Chỉ mục chống phát lại TXID theo cùng hợp
đồng (M8, xem mục trước): dựng lại lúc khởi động, không bao giờ
đóng băng. Đẳng thức bit-chính-xác với `load_chain_replay` được
kiểm thử xác minh trên trạng thái chuẩn (state_root + SMT).

## Snapshot trạng thái ĐÃ XÁC MINH — các khúc + manifest + xác minh mật mã (P0.3)

Snapshot M6b ở trên tăng tốc khởi động của một nút tin đĩa của
mình. P0.3 giải quyết một nhu cầu khác: vận chuyển trạng thái
giữa các nút (bootstrap một nút mới, sửa chữa) mà không tin vận
chuyển. Định dạng và xác minh sống trong
`crates/scone-storage/src/verified_snapshot.rs` + dựng lại gốc
trong `crates/scone-blockchain/src/snapshot_verify.rs`.

### Mô hình tin cậy

Snapshot mang trạng thái của một khối **được hoàn tất** — một
checkpoint ký bởi quorum ủy ban (`CheckpointData.state_root`,
gốc O(1) `state_root_smt`). Tính chân thực của checkpoint
(quorum chữ ký Ed25519) được tầng chuỗi xác thực TRƯỚC khi
nhập. Bản thân phép nhập xác minh sự khớp snapshot ↔ checkpoint
bằng TÍNH LẠI:

1. **băm mọi trang**: `blake3("SCONE-SNAP-PAGE-HASH" ‖
   mã hóa trang đầy đủ)` phải bằng băm do manifest loan báo
   (một byte giả → từ chối);
2. **tổng các mục == bộ đếm manifest** (một trang thiếu hay một
   bộ đếm sai → từ chối);
3. **gốc SMT tính lại từ các mục nhập == `checkpoint.data.state_root`**
   — kiểm tra mật mã CHÍNH: toàn bộ trạng thái (tên miền + TLD)
   được gấp lại vào một SMT mới
   (`snapshot_verify::recompute_state_root`, các lá
   `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1` y như đường sống)
   và gốc phải bằng cái ủy ban đã ký. Một snapshot giả mạo,
   cắt cụt, hay không đầy đủ bị từ chối TRƯỚC MỌI LẦN GHI;
4. **các nhất quán cấu trúc**: `height`/`tip_hash`/`state_root`
   của manifest == của checkpoint; id tăng nghiêm ngặt (không
   trùng); chỉ mục trang tuần tự; biên DoS (≤ 10.000 mục/trang,
   ≤ 2^20 trang, ngân sách tổng).

Thất bại xác minh → `StorageError::SnapshotRejected`, kho giữ
nguyên không đổi nghiêm ngặt (mọi thứ được xác minh trước giao
dịch ghi).

### Định dạng tuần tự (nghiêm ngặt, big-endian, giải mã không
khoan nhượng)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← every page is
        ‖ entry_count(4) ‖ entries                   self-identifying

entry   = kind(0x01 domain | 0x02 TLD | 0x03 TLD floor) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B or TldStateBytes 2/34 B —
           the existing canonical storage formats, reused; kind
           0x03 (lifetime floor of an open TLD, owner-option) has
           a dedicated format WITHOUT a length prefix:
           id(32) ‖ floor(8 BE), fixed size)
```

Thứ tự trang: mọi mục tên miền (id tăng), rồi mọi mục TLD (id
tăng), rồi các sàn đời sống TLD ĐÓNG BĂNG tại chiều cao
snapshot (`snapshot_tld_floors_all` — không phải bảng sống) —
thứ tự phân trang con trỏ của kho. Mỗi trang mang `(index,
height, state_root của checkpoint, entries)`: một trang lẻ biết
nó thuộc snapshot nào. Trang 256 mục lúc xuất
(`VERIFIED_SNAPSHOT_PAGE_ENTRIES`); biên DoS lúc nhập. Các sàn
TLD (`tld_floors_v1`) là một phần của định dạng khả chuyển: kind
0x03 mang chúng, và phép nhập đòi chúng tăng nghiêm ngặt và nhắm
một TLD CÓ MẶT trong snapshot — nếu không bên nhập sẽ không
bao giờ giết cái mà bên xuất sẽ giết (fork âm thầm, bị từ
chối).

### Xuất (`export_verified_snapshot(store, checkpoint)`)

Tuần tự hóa trạng thái của CHECKPOINT HOÀN TẤT CUỐI — không
phải đỉnh. Phát các tên miền, TLD, rồi các sàn đời sống TLD
đóng băng tại chiều cao snapshot (đối xứng xuất/nhập: bản xuất
lịch sử bỏ sót sàn — đã sửa, kiểm thử hồi quy
`export_carries_tld_lifetime_floors`). Tiền đề: kho giữ một
snapshot M6b bền vững (`snapshot_v1_*`) đúng tại chiều cao
checkpoint (cửa sổ `CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL` bảo
đảm một trạng thái như vậy sẵn có đều đặn); neo của khối được
TÍNH LẠI từ byte đã lưu và phải bằng
`checkpoint.data.block_hash`. Nếu không →
`StorageError::SnapshotUnavailable` (xuất là một hành động
tường minh: chờ khoảng/checkpoint kế tiếp).

### Nhập (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Xác minh mọi thứ (xem trên), rồi ghi trong MỘT giao dịch redb:
các bảng sống `domains`/`tlds` + bộ đếm + `tip = (H, checkpoint
hash)`, các bảng đóng băng `snapshot_v1_*` neo tại cùng khối,
và dấu hiệu dành riêng
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Đích: chỉ kho rỗng (nếu không
`SnapshotUnavailable`).

### Chi phí xác minh (được ghi nhận)

Dựng lại SMT qua các phép chèn có thứ tự: N mục × O(40) băm ≈
O(N·40). ~2,1 triệu băm blake3 cho 100.000 tên miền (vài chục
ms). MỘT LẦN — một nhập bootstrap/sửa chữa, không bao giờ
đường theo-khối: chấp nhận được. Chèn có thứ tự (id tăng) tạo
mỗi điểm nhánh đúng một lần; dù sao gốc cũng chỉ phụ thuộc nội
dung (bảo đảm SMT, fuzz vi phân trong `smt.rs`).

## Bootstrap trạng thái — không phát lại genesis (P0.4)

Một nút mới gia nhập mạng không phát lại lịch sử từ genesis:

```text
finalized checkpoint (relay, validated by the chain layer)
        │
        ▼
verified snapshot (manifest + pages, crypto verification above)
        │  import → state at H + tip (H, hash) + marker
        ▼
anchor block H (requested from the network BY HASH — append_bootstrap_anchor)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height, ordinary atomic
        │               append — the storage API requires only this)
        ▼
synced — boot replays ONLY the suffix above H
```

- **API lưu trữ phơi bày cho relay**: `append_bootstrap_anchor(H,
  hash, bytes)` (đặt khối neo: dấu hiệu bắt buộc, nhất quán
  chiều cao/băm với dấu hiệu, lũy tính) rồi phép nối nguyên tử
  thường theo chiều cao tăng. Truy xuất mạng (theo băm từ
  manifest) là mối quan tâm của relay, không phải của kho.
- **`load_chain`** nhận ra dấu hiệu: lịch sử dưới H vắng một
  cách chính đáng (nút không-lưu-trữ). Cụ thể: quét lại cửa sổ
  chống phát lại M8 được sàn tại H (các khối dưới H không tồn
  tại và không thiếu); một đỉnh không có khối được lưu SAU một
  phép nhập (neo chưa đặt) → chẩn đoán có kiểu
  `SnapshotUnavailable` ("neo phải đến từ mạng"), không phải hỏng.
- **Nút lưu trữ vs nút trạng thái-đầy-đủ**: một nút lưu trữ
  (cờ vận hành viên) giữ mọi thứ — nó phục vụ các khối lịch sử
  cho mạng và khởi động qua các đường hiện có. Một nút
  trạng thái-đầy-đủ (mặc định P0.4 cho một nút mới ĐƯỢC CẤU
  HÌNH thế) khởi động tại H: nó xác thực các khối tương lai
  (phát lại hậu tố) nhưng không thể phục vụ lịch sử dưới H hay
  phát lại genesis. **Hành vi mặc định KHÔNG ĐỔI**: đường này
  chỉ hoạt động nếu vận hành viên kích hoạt tường minh phép
  nhập (`import_verified_snapshot` là một hành động riêng,
  không bao giờ tự động); mọi nút hiện có khởi động giống hệt.
- **Bằng với nút đầy**: sau khi gia nhập hậu tố H+1..H+k,
  trạng thái, đỉnh, và `state_root_smt` bit-chính-xác với một
  nút đầy tại cùng đỉnh (đã kiểm thử); chỉ mục chống phát lại
  phủ đúng các tx tại/trên H — một nút không-lưu-trữ không thể
  biết cửa sổ dưới H, quy tắc trạng thái vẫn là rào chắn (một
  tx trùng dưới H thất bại trên trạng thái, không bao giờ được
  chấp nhận hai lần).

## Chính sách bộ nhớ

- **Con trỏ mọi nơi**: `iterate_domains` chặn trang tới `max`
  mục (100 mặc định) qua `range` lười của redb; toàn bộ bảng
  tên miền không bao giờ được vật chất hóa. Các danh sách lớn
  (TLD, tên miền) bắt buộc đi qua con trỏ này
  (`iterate_tlds`, cùng hợp đồng, M7d).
- **Chỉ delta**: `append_block_with_state` chỉ ghi các tên miền
  và TLD bị thay đổi bởi khối — không bao giờ trạng thái đầy
  đủ;
- **Không nhân bản đồng bộ**: trạng thái RAM không được lưu lại
  theo khối; trạng thái đĩa không được nạp lại theo khối;
- **Đọc điểm**: mỗi lần đọc mở một giao dịch đọc ngắn, trả giá
  trị, đóng; các `AccessGuard` redb chỉ sống trong thời gian
  của bản sao bị chặn;
- **Khối lịch sử không bao giờ trong RAM**: `load_chain` chỉ
  nạp đỉnh; `load_chain_replay` phát trực tiếp theo chiều cao;
- **Bộ đếm được duy trì**: `domain_count` là O(1), không phải
  quét.

## Biên DoS

Kho chặn những gì một peer thù địch có thể khiến nó lưu hay
cấp phát:

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)**: một bản ghi DHT lớn hơn
  bị từ chối trước mọi giao dịch ghi bởi `put_dht_cache` →
  `StorageError::TooLarge` (không có gì được ghi);
- **`MAX_DOMAIN_PAGE` (10.000)**: `iterate_domains` *nội bộ*
  chặn trang yêu cầu tới `min(max, 10_000)` — một `max` quá
  mức không bao giờ vật chất hóa hơn 10.000 trạng thái trong
  RAM; `iterate_tlds` cùng biên (`MAX_TLD_PAGE`, M7d);
- **Các khóa `meta` dành riêng**: `meta_set` từ chối `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count`,
  và `verified_snapshot_v1` (P0.3) với
  `StorageError::ReservedKey` — ghi đè lên sổ kế toán nội bộ
  của kho sẽ âm thầm làm hỏng nó.
- **Cache DHT chặn-kép (LRU)**: `DHT_CACHE_MAX_ENTRIES`
  (100.000 mục) VÀ `DHT_CACHE_MAX_BYTES` (2 GiB — canh trường
  hợp thù địch; mỗi mục đã bị nắp 64 KiB bởi
  `MAX_DHT_CACHE_ENTRY`), biên nào tới trước trục xuất theo thứ
  tự khóa. Các bộ đếm (mục + byte) được duy trì lúc ghi —
  không bao giờ một quét để đếm; dựng lại chính xác chỉ xảy ra
  lúc mở (quét đầy duy nhất, lúc khởi động). Bộ truy cập
  `dht_cache_entries`/`dht_cache_bytes`, biên chỉnh được qua
  `set_dht_cache_bounds` (kiểm thử:
  `dht_cache_respects_the_double_bound`).

## Mô hình tin cậy

Hai đường nạp, hai mức bảo đảm:

- **`load_chain` — tin đĩa cục bộ**: băm của đỉnh được **tính
  lại** từ header khối đỉnh và so với `meta["tip"]` lưu (lệch
  → `StorageError::Corrupted`, không bao giờ một chuỗi khôi
  phục trên một băm giả), và các trạng thái tên miền được giải
  mã nghiêm ngặt. Tuy nhiên các khối lịch sử **không được phát
  lại**: các trạng thái lưu được lấy theo mặt chữ. Đây là khởi
  động nhanh của một nút tin đĩa riêng của mình (chỉ được ghi
  bởi chính nút đó, nguyên tử). Từ M6b, đường này trước hết
  dùng **snapshot trạng thái** khi khả dụng (xem trên): neo
  snapshot được tính lại và hậu tố trên H được xác thực lại như
  khối sống — phần phát lại do đó không-tin, cơ sở đóng băng
  tin-đĩa.
- **`load_chain_replay` — không tin**: mọi khối được đọc lại
  và xác thực lại từng cái một (xác thực đầy đủ, giống khối
  sống); đỉnh thu được phải bằng đỉnh lưu. Đây là công cụ
  kiểm soát/sửa chữa.

**Khuyến nghị**: một nút nghiêm ngặt (hay một nút mà tệp lưu
trữ của nó có thể đã bị can thiệp — đĩa chia sẻ, khôi phục một
bản sao dự phòng đáng ngờ) phải khởi động qua `load_chain_replay`,
không phải `load_chain`.

## Đồng thời

redb 4.x cho phép **một đối tượng `Database` duy nhất mỗi tệp
và mỗi tiến trình** (mở kép đồng thời thất bại với
`DatabaseAlreadyOpen`). Mô hình được hỗ trợ: một `RedbStore`
nhân bản (`Arc<Database>`) chia sẻ giữa các luồng — trình ghi
tuần tự bởi `&mut self` + khóa nội bộ redb, trình đọc đồng thời
(giao dịch đọc snapshot). Sau khi mọi handle được thả hoàn
toàn, tệp có thể được mở lại.

## Kiểm thử (`tests/redb_store.rs` + các mô-đun `#[cfg(test)]`)

Mọi kiểm thử dùng tệp tạm redb riêng (`tempfile`). Phủ:

- khứ hồi khối genesis→N, khởi động lại, đọc lại, phát lại;
- M7d: registry TLD lưu bởi delta nguyên tử, khôi phục khi
  khởi động lại (một `RegisterDomain` dưới TLD tuyên bố trước
  khởi động lại vẫn được nhận — cửa sổ đóng), bộ đếm chính xác
  khi mở lại, con trỏ `tlds` tăng, nối thất bại không để lại
  TLD, khóa dành riêng `tld_count` bị từ chối;
- nguyên tử: nối thất bại không để lại gì (đỉnh, tên miền, chỉ
  mục nguyên vẹn); delta trùng đếm một lần;
- con trỏ tên miền: phân trang 100/7, thứ tự `DomainId` tăng,
  bắt đầu sau nghiêm ngặt một id có mặt hay thiếu, `max = 0`;
- bộ đếm `domain_count` chính xác (chèn/cập nhật/mở lại);
- khứ hồi cache DHT + ghi đè;
- đồng thời: đọc từ hai luồng trên handle nhân bản;
- cơ sở dữ liệu hỏng: tệp cắt cụt / rác → lỗi có kiểu, không
  panic (`catch_unwind`);
- M6b: snapshot ghi tại khoảng, dưới nghiêm ngặt đỉnh; khởi
  động từ snapshot = phát lại đầy đủ bit-chính-xác (state_root
  + SMT, trạng thái logic hoàn chỉnh, đỉnh); snapshot bị sửa
  (tip_hash hỏng, trạng thái không giải mã được, chiều cao >
  đỉnh, meta sai chiều dài) → quay về âm thầm nạp đầy, cùng
  chuỗi cuối; hậu tố phát lại < khoảng; thay thế snapshot
  (không tích lũy) khi vượt khoảng; kho pre-M6b không bảng
  snapshot nạp giống hệt;
- M8: chỉ mục chống phát lại dựng lại lúc khởi động — một tx
  bao hàm lại trong cửa sổ bị từ chối (`TxReplay`) sau khôi
  phục; một tx cũ hơn cửa sổ được nhận (xén không đổi, quy tắc
  trạng thái quyết định); số đếm chính xác == chỉ mục sống (==
  `load_chain_replay`) cho một kho ngắn hơn và dài hơn cửa sổ;
  khởi động snapshot: phần cửa sổ dưới H được dựng bởi quét lại,
  phần trên bởi phát lại hậu tố;
- P0.2: chỉ mục tên — `resolve_name == DomainId::from_name`
  cho 120 tên miền (sống, sau khởi động lại, sau xóa + dựng
  lại), `resolve_tld` cho TLD được tuyên bố; GC một tên miền
  hết hạn gỡ mục (không ma), trước và sau dựng lại; tách nghiêm
  ngặt không gian tên miền/TLD; nối thất bại không để lại mục
  chỉ mục; khóa chỉ mục là phép dẫn xuất BLAKE3 được ghi tài
  liệu;
- P0.3/P0.4 (`tests/verified_snapshot.rs`): xuất → nhập →
  khởi động == phát lại đầy đủ (trạng thái, đỉnh,
  `state_root_smt` bit-chính-xác; trạng thái nhập là của khối
  hoàn tất, không phải đỉnh); trang giả bởi một byte → từ chối
  trước mọi ghi (kho giữ nguyên nghiêm ngặt); trang thiếu → từ
  chối; `state_root` sai (manifest ↔ checkpoint, rồi gốc tính
  lại ≠ gốc ký) → từ chối; bộ đếm/chiều cao sai / đỉnh không
  nhất quán / id trùng → từ chối; nhập đòi kho rỗng; xuất đòi
  một snapshot bền vững tại chiều cao checkpoint; hợp đồng neo
  (`append_bootstrap_anchor`: dấu hiệu bắt buộc, nhất quán, lũy
  tính); định dạng tuần tự nghiêm ngặt (byte thừa, thẻ, biên
  DoS); snapshot tại H rồi gia nhập các khối H+1..H+k → chuỗi
  giống hệt nút đầy (trạng thái/đỉnh/gốc), khởi động không lịch
  sử dưới H, chỉ mục chống phát lại = đúng các tx tại/trên H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

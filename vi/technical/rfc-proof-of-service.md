# RFC — Bằng chứng vận hành và ủy ban tính cuối (proof-of-service)

> Trạng thái: CHUẨN — đã triển khai trọn vẹn (bước 1 đến 5:
> core/protocol/blockchain/network/CLI; tài liệu này là dạng hợp
> nhất). Mọi quy tắc dưới đây được đối chiếu với mã; các hằng số
> được trích dẫn là các giá trị trong mã.
> Ngày: tháng 10 năm 2026 (hợp nhất chuẩn). Liên quan đến:
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Động cơ

Tập hợp tính cuối là tập chủ sở hữu (chủ của các tên miền/TLD còn
sống). Sở hữu trao một ghế — **vận hành** không trao gì. RFC này
ràng buộc quyền ký checkpoint vào bằng chứng vận hành của một
relay trực tuyến, mà không bao giờ đặt điều kiện lên quyền sở
hữu:

- **Tự do được bảo toàn**: một TLD tư nhân vẫn có thể được cấp
  không kèm điều kiện hoạt động chừng nào hạn của nó còn được
  gia hạn (`RenewTld`, hạn 3 năm). Chỉ *việc ký checkpoint* mới
  được kiếm qua sự hiện diện.
- **Khuyến khích vận hành**: vận hành một TLD tư nhân (hay bất kỳ
  ghế nào) = ngồi trong ủy ban. Vận hành viên tham gia giữ mạng
  hoạt động và giao thức thừa nhận điều đó.
- **Phi tập trung đo lường được**: ủy ban trở thành tập các vận
  hành viên *thực sự trực tuyến*, không phải các khóa ngủ đông.

## 2. Tổng quan

Một giao dịch `Heartbeat` nhẹ nhàng (discriminant `0x71`), ký bởi
một khóa **vận hành viên**, phát một lần mỗi epoch. Trạng thái giữ
một cửa sổ trượt của các heartbeat gần đây. Ủy ban của một epoch
được bầu **từ tập con của tập hợp mà vận hành viên hiệu lực có
heartbeat còn mới**. Không có heartbeat: không ký, nhưng không
mất quyền sở hữu.

## 3. Giao dịch Heartbeat

```
Heartbeat {
    operator: PublicKey,     // Ed25519 key of the operator (the signer)
    owner: OwnerId,          // identity RE-DERIVED from operator (never trusted)
    tip_seen: [u8; 32],      // last tip seen by the node
    height_seen: u64,        // corresponding height
    timestamp: u64,          // Unix seconds
} + Ed25519 signature over the canonical payload (network injected)
```

Thân ký (wire, `scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (khóa của người ký xuất hiện ở đuôi
thân, như mọi tx) `‖ signature[64]`, tức ~230 byte khi mã hóa —
**không PoW**: chi phí là sự hiện diện liên tục (một heartbeat
mỗi epoch, trong nhiều tháng).

Người ký CHÍNH LÀ vận hành viên: `owner` luôn được dẫn xuất lại
từ `operator` (không bao giờ lấy từ đầu vào), và việc giải mã từ
chối với một lỗi có kiểu (`InvalidOwner`) một khóa ký khác với
trường `operator` ở đầu.

- `tip_seen`/`height_seen` phục vụ khả năng quan sát (phát hiện
  vận hành viên lỗi hoặc trễ); chúng KHÔNG ảnh hưởng tính hợp lệ
  (một nút trễ nhẹ vẫn là một nút sống).

### 3.1 Xác thực thuần túy (scone-core)

1. Chữ ký Ed25519 hợp lệ trên payload chuẩn của mạng.
2. `owner` == danh tính dẫn xuất từ `operator` (nếu không
   `InvalidOwner`) — kiểm tra lúc mã hóa, lúc giải mã VÀ bởi
   `validate()`.

### 3.2 Quy tắc trạng thái (áp dụng, scone-blockchain)

Khi áp dụng một heartbeat trong một khối mang ở chiều cao
`block_height` (mọi lần từ chối đều trước mọi đột biến):

1. **Chống phát lại chéo epoch**: `timestamp` tăng nghiêm ngặt
   theo từng vận hành viên — từ chối có kiểu `StaleHeartbeat`
   nếu `timestamp <= operator_last_hb[operator]`.
2. **Duy nhất mỗi epoch**: chỉ một heartbeat được chấp nhận cho
   mỗi (`operator`, epoch của khối mang) — bản sao bị từ chối
   với một lỗi có kiểu (`HeartbeatDuplicate`). Epoch là
   `block_height / epoch_min_blocks` (cùng định nghĩa với tính
   cuối, không bao giờ là hằng số mới).
3. Chèn vào cửa sổ có ghi nhật ký + cập nhật
   `operator_last_hb` (đảo ngược được khi reorg, hoàn tác bit-
   chính-xác).

### 3.3 Chống spam

- 1 tx được chấp nhận mỗi vận hành viên mỗi epoch — lưu lượng
  tối đa là `pool_size / epoch_secs`, không đáng kể.
- Không phí (trước tầng kinh tế): tốc độ bị chặn về mặt cấu
  trúc.

## 4. Ủy quyền vận hành

Liên kết ghế → vận hành viên đi qua hai giao dịch chuyên dụng:

- `SetTldOperator { tld_id, operator }` (discriminant `0xA9`),
  ký bởi chủ **hiện tại** của TLD **tư nhân** (một TLD mở không
  có ai ký một cách chính đáng — từ chối có kiểu
  `TldIsOpen`). `operator: None` = thu hồi, quay lại tự vận
  hành bằng khóa của chủ; thu hồi được tức thì.
- `SetDomainOperator { domain_id, operator }` (discriminant
  `0x5E`), ký bởi chủ hiện tại của tên miền. Cùng ngữ nghĩa.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` không mang
  trường `operator` nào: liên kết ghế → vận hành viên luôn là
  một sự kiện on-chain gần đây, được phân giải tại đỉnh.

Tùy chọn vận hành viên trên wire là chuẩn: `0x00` = `None`
(tự vận hành), `0x01` = khóa thô 32 byte; giá trị nào khác bị từ
chối lúc giải mã.

Trạng thái duy trì:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per seat
operator_seats: HashMap<OperatorKey, u64>       // bounded counter
operator_last_hb: HashMap<OperatorKey, u64>     // last accepted ts
```

Quy tắc áp dụng (`apply_seat_operator`, có ghi nhật ký):

- **Lũy tính**: ủy quyền lại cho cùng vận hành viên (hoặc thu
  hồi một ủy quyền không tồn tại) là một no-op không ghi nhật
  ký (hoàn tác bit-chính-xác đòi hỏi vậy).
- **Giới hạn chống cho thuê**: một vận hành viên chỉ có thể được
  ủy quyền bởi **đúng một ghế** tại một thời điểm
  (`MAX_SEATS_PER_OPERATOR = 1`). Một ghế thứ hai ủy quyền cho
  một vận hành viên đã bị chiếm → từ chối có kiểu
  `OperatorAlreadyDelegated`. Lý do: nếu không giới hạn, một
  vận hành viên lớn bán chữ ký theo số lượng và tái tạo sự tập
  trung — chính xác điều ủy ban được bầu nhằm tránh.

## 5. Cửa sổ độ mới

### 5.1 Cửa sổ trượt

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Mục được chèn khi áp dụng một heartbeat hợp lệ (§3.2).
- Dọn tất định tại MỌI lần áp dụng khối
  (`gc_heartbeats_journaled`): mọi mục của epoch `e` với
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` bị gỡ (có ghi nhật ký —
  đảo ngược được khi reorg, như các hàng đợi hết hạn).
- `HEARTBEAT_WINDOW_EPOCHS = 2`: epoch hiện tại + epoch trước —
  một vận hành viên có thể bỏ LỠ MỘT epoch mà không mất độ mới,
  epoch thứ hai khiến nó hết hạn.
- **Nắp bộ nhớ cấu trúc**: cửa sổ giữ tối đa
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` mục (một mỗi vận
  hành viên mỗi epoch, khóa tuple); `operator_last_hb` bị chặn
  bởi số vận hành viên và không bao giờ được dọn (quy tắc chống
  phát lại là đơn điệu).

### 5.2 Độ mới

"Còn mới" = sự hiện diện của một mục `(e, operator)` với `e`
trong `[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Vì trạng
thái dọn tại mọi khối, sự hiện diện trong `recent_heartbeats`
CHÍNH LÀ độ mới (`operator_is_fresh`).

### 5.3 Vận hành viên hiệu lực

**Vận hành viên hiệu lực** của một ghế là `seat_operators[seat]`
nếu một ủy quyền sống tồn tại, nếu không thì khóa của chủ sở
hữu (`effective_operator`). Ghế ký dưới khóa hiệu lực đó — một
ủy quyền di chuyển quyền ký, không bao giờ quyền sở hữu.

## 6. Bầu chọn ủy ban

```
eligible = { effective operators of the frozen pool's seats
             (banned key excluded) having a fresh heartbeat }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // existing draw
else :
    committee = select_committee(seed, pool)       // safe degradation
```

Chi tiết chuẩn (được xác minh trong `finality.rs`, đặc tả trong
`/docs/en/technical/blockchain.md` § "Seats and the finality
committee"):

- `pool` là tập hợp **bị đóng băng tại checkpoint hoàn tất cuối
  cùng** (`FinalizedBase.pool` — không bao giờ trạng thái
  sống): một góc nhìn cục bộ sẽ lệch giữa các nút và quorum sẽ
  không bao giờ hình thành;
- `seed` dẫn xuất từ băm nội dung của checkpoint trước
  (`signing_hash`);
- mọi ghế của tập hợp đóng băng được phân giải tới vận hành
  viên HIỆU LỰC của nó bởi trạng thái tại đỉnh (ủy quyền và
  heartbeat là các sự kiện on-chain gần đây: hai nút ở cùng
  đỉnh tính ra cùng một tập); khóa bị cấm vẫn bị loại;
- cuộc rút thăm chạy trên tập khóa hiệu lực đã sắp/xóa trùng
  lặp (`select_committee` không đổi).

**Suy giảm tường minh và an toàn**: một mạng đang bootstrap
(không heartbeat), hay một sự cố tổng thể của các vận hành
viên, quay về toàn bộ tập hợp — tính cuối không bao giờ mất về
mặt cấu trúc. Đây là quy tắc "không bao giờ dead-lock" của dự án
áp dụng tại đây.

### 6.1 Bootstrap nguội

Epoch 0 (và chừng nào còn ít hơn
`MIN_FINALITY_COMMITTEE_SIZE` vận hành viên còn mới): toàn bộ
tập hợp. Ràng buộc tự siết lại khi các vận hành viên phát.
Không có giai đoạn chuyển tiếp đặc biệt nào phải cấu hình.

## 7. Phát tự động (scone-network)

Tại mỗi chu kỳ tạo khối (`Wake::Produce` — không có timer mới),
relay kiểm tra xem một RANH GIỚI EPOCH có bị vượt qua không
(`height / epoch_min_blocks`, cùng định nghĩa với tính cuối).
Tại chu kỳ đầu tiên của một epoch mới, nó phát một heartbeat
nếu:

1. một khóa anchor được vũ trang (`--anchor-key` hoặc tự động),
   VÀ
2. chính khóa đó giữ một ghế sống (kiểm tra CỤC BỘ: khóa nằm
   trong tập hợp đủ điều kiện on-chain — chủ của các tên
   miền/TLD còn sống, khóa bị cấm bị loại trừ).

Chống spam cục bộ: tối đa MỘT lần phát mỗi epoch mỗi nút
(`last_emitted_epoch` ghi nhớ, bộ nhớ cục bộ, không bao giờ
trạng thái chuỗi); ghi nhớ chỉ được lấy sau một lần tiêm vào
mempool thành công — thất bại sẽ thử lại ở chu kỳ tiếp. Không
gì ở đây là nghiêm trọng: cùng lắm heartbeat chờ chu kỳ tiếp
theo.

Một **vận hành viên được ủy quyền** (bản thân không phải chủ sở
hữu) không được phủ bởi phát tự động: họ phát qua trình dựng
CLI của §8 (vận hành thủ công hoặc cách ly không khí). Heartbeat
KHÔNG NHẤT THIẾT phải được người phát đưa vào khối: nó là một tx
như mọi tx khác (mempool, tạo khối bởi bất kỳ ai).

## 8. CLI (`scone`)

Trình dựng heartbeat ngoại tuyến (đã gỡ cùng bề mặt `tx`) đã
bị gỡ bỏ cùng bề mặt tx ngoại tuyến (cli-simplify-v1). Các
relay tự phát heartbeat của mình tự động (§7); phát thủ
công/cách ly không khí, nếu bao giờ cần lại, sẽ trở về dưới
dạng một lệnh chuyên dụng. Mặc định trước đây là: `tip_seen` =
32 byte 0 (chỉ để quan sát), `height_seen` = 0, `timestamp` =
bây giờ.
- Như mọi tx, `tx sign` **ràng buộc lại** operator và owner vào
  khóa ký THẬT (giá trị đầu vào không bao giờ được tin); `tx
  verify` xác minh cục bộ.

## 9. Khả năng quan sát

- `NodeStatus` (thông điệp p2p wire `0x0d`) mang
  `fresh_operators: u32` (varint): số vận hành viên có heartbeat
  còn mới — tập con sống của tập hợp đủ điều kiện cho ủy ban
  (vận hành viên hiệu lực của các ghế trong tập hợp, khóa bị cấm
  loại trừ, đã sắp/xóa trùng lặp). Relay điền từ
  `state().fresh_operators(height)`; `scone status` hiển thị
  dưới dạng `fresh operators: N`.
- Lập luận chiến dịch: "vận hành một TLD tư nhân = ngồi trong
  ủy ban tính cuối của DNS phi tập trung" — kiểm chứng được
  on-chain.
- Bề mặt tương lai (chưa đặc tả): `operator_status` — theo từng
  ghế, vận hành viên được ủy quyền, độ mới (mới / hết hạn /
  chưa từng thấy), `tip_seen` cuối. RPC điều khiển cục bộ đã bị
  gỡ (nút chỉ-p2p); bề mặt vẫn phải được định nghĩa.

## 10. Những gì RFC này KHÔNG làm

- **Không mất quyền sở hữu** vì không hoạt động (phương án A —
  suy giảm TLD không vận hành — bị loại trừ tường minh).
- **Không thưởng** (phương án C chờ một tầng kinh tế).
- Heartbeat không chứng minh *chất lượng* dịch vụ (khối được
  tạo, bản ghi DHT được phục vụ) — chỉ sự hiện diện. Đếm đóng
  góp thật là một phần mở rộng tương lai khả dĩ, dễ bị lạm dụng
  hơn.

## 11. Các bất biến được bảo vệ

- Tính tất định bit-chính-xác: cửa sổ và cuộc bầu là hàm thuần
  túy của trạng thái được áp dụng (ghi nhật ký, hoàn tác bit-
  chính-xác).
- Không bao giờ panic trên dữ liệu mạng: heartbeat hay ủy quyền
  sai dạng → từ chối có kiểu (`InvalidOwner`, `StaleHeartbeat`,
  `HeartbeatDuplicate`, `OperatorAlreadyDelegated`, `TldIsOpen`,
  `NotTldOwner`, `NotOwner`).
- Quyền sở hữu là vô điều kiện; chỉ việc ký checkpoint là có
  điều kiện.

## 12. Hằng số và tham số (giá trị trong mã)

| Tên | Giá trị | Vị trí |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| `Heartbeat` discriminant | `0x71` | `scone-protocol/src/transaction.rs` |
| `SetTldOperator` discriminant | `0xA9` | như trên |
| `SetDomainOperator` discriminant | `0x5E` | như trên |

Các giá trị này là một phần của đồng thuận: thay đổi chúng làm
gãy khả năng phát lại chuỗi (cửa sổ/dọn) hoặc cuộc bầu (ủy
ban).

## 13. Kế hoạch triển khai — trạng thái cuối

| Bước | Crate | Nội dung | Trạng thái |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, xác thực | xong |
| 2 | blockchain | cửa sổ trượt (ghi nhật ký), tập hợp lọc, bầu chọn, ~15 kiểm thử | xong |
| 3 | network | phát tự động, chuyển tiếp tx | xong |
| 4 | CLI | `tx build heartbeat`, hiển thị `fresh operators` trong `scone status` | xong |
| 5 | docs | RFC này hợp nhất thành dạng chuẩn, bản song song fr/en | xong |

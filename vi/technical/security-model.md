# Mô hình bảo mật — lựa chọn ủy ban và chống grinding

> Tài liệu chuẩn. Bất kỳ sự khác biệt nào giữa tài liệu và mã = bug.

## Tổng quan

Ủy ban của epoch N+1 được rút ra từ seed
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`,
trên **tập hợp đủ điều kiện bị đóng băng tại khối được hoàn tất bởi
checkpoint N**. `signing_hash` là băm của **nội dung** checkpoint
(epoch, height, block, prev, state root — loại trừ bộ chữ ký): seed
là một hàm thuần túy của CÁI GÌ đã được hoàn tất, không bao giờ
của BỘ PHẬN nào của quorum đã ký nó — hai góc nhìn giữ các tập con
khác nhau (cùng hợp lệ) của cùng một checkpoint bầu ra cùng một ủy
ban.

## Mối đe dọa: grinding seed của ủy ban

Kẻ tấn công muốn ảnh hưởng đến thành phần ủy ban. Ba cửa sổ tấn
công tồn tại; cả ba đều bị đóng về mặt cấu trúc:

### 1. Trước seed — không thể grind vị trí của mình

Seed của epoch N+1 dẫn xuất từ **băm của checkpoint N**, ký bởi
quorum (floor(2n/3)+1 của ủy ban N — `quorum_for`). Checkpoint N
cam kết `state_root` của một trạng thái **ghim tập hợp** (cơ sở
PoS = trạng thái chính xác tại khối được hoàn tất,
`FinalizedBase.pool`). Tạo ra một seed khác đòi hỏi một checkpoint
khác được ký bởi quorum — tức là làm hỏng ≥ 1/3 ủy ban N, chính là
giả thuyết lỗi bị loại trừ bởi mô hình BFT.

### 2. Sau seed — quá muộn để tham gia

Mọi đăng ký (tuyên bố tên miền/TLD, xuất hiện khóa chủ mới) chỉ
tính cho các epoch sau, mà seed của chúng chưa được biết. Tập hợp
bị đóng băng trong `FinalizedBase` TRƯỚC khi bất kỳ ai (kể cả
những người ký checkpoint N) có thể tính seed của N+1.

### 3. Chọn "best-of-m" không có lợi thế

Vị trí của một khóa trong cuộc rút thăm =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (đều nhau), bậc tự do
duy nhất của nó là `pk`. Kẻ grinding sinh m khóa và chỉ đăng ký
khóa có vị trí tốt nhất trước hết phải **biết seed** — nhưng lúc
đó tập hợp đã bị đóng băng rồi. Đăng ký m ứng viên "đề phòng" tốn
m bằng chứng công việc (một cho mỗi tên miền/TLD) và cho đúng m/|pool|
hiện diện kỳ vọng: **không có lợi thế gì so với m lần đăng ký
trực tiếp**. Chiến lược thích ứng (xem seed, rồi chọn đăng ký gì)
là bất khả về mặt thời gian.

### 4. Thiên lệch người-produce-cuối (tấn công giữ khối)

Người produce khối cuối trước một checkpoint có thể giữ lại khối
của mình để ảnh hưởng `signing_hash(checkpoint N)` và do đó seed.
Hai lớp phòng thủ:

- checkpoint ký một trạng thái ở chiều cao H, và **tập hợp được
  đánh giá tại dấu thời gian của khối H** — giữ lại một khối làm
  trễ checkpoint nhưng không đổi tập hợp của một epoch đã bị
  checkpoint N−1 đóng băng (các tập hợp chồng lấn ít từ epoch
  sang epoch);
- việc giữ khối khiến kẻ tấn công mất chính phần thưởng produce
  của mình và chỉ ảnh hưởng LIVENESS (checkpoint bị trễ), không
  bao giờ SAFETY: hai checkpoint cuối không tương thích đòi hỏi
  ≥ 1/3 chữ ký kép, chứng minh được bằng `SlashTx` và bị phạt
  bằng cấm.

Kết luận: một VRF hay commit-reveal sẽ không thêm gì ở đây — seed
vốn đã là hàm của một sự kiện được quorum ký, khó đoán-có-thể-trễ-
nhưng-không-thể-giả-mạo, và việc đóng băng tập hợp gỡ bỏ đòn bẩy
thích ứng duy nhất. Đừng phức tạp hóa giao thức (ràng buộc P1.7:
"nếu cần" — phân tích: không cần).

## Equivocation và slashing

Một anchor ký hai checkpoint xung đột ở cùng epoch có thể bị bất
kỳ ai giữ cả hai chữ ký chứng minh (`SlashTx`,
discriminant `0x15`): trục xuất khỏi tập hợp PoS (được ghi nhật
ký, có thể đảo ngược khi reorg). Các tên miền/TLD của bị cáo
không bị tịch thu (trừng phạt tối thiểu — quyết định M9).

## Các tính chất được kiểm thử xác minh

- `checkpoint_hash_engages_signatures`: băm checkpoint bao phủ
  các chữ ký (không giả mạo hậu nghiệm);
- `seed_chain_and_recovery`: chuỗi seed dẫn xuất chính xác, gồm
  cả khôi phục k;
- `small_committee_never_finalizes`: canh BFT — một ủy ban dưới
  `MIN_FINALITY_COMMITTEE_SIZE` không bao giờ hoàn tất;
- e2e 4 relay: một checkpoint hoàn tất và lan truyền, cùng epoch
  trên mọi nút.

## Giới hạn đã biết (được ghi nhận, chưa sửa)

- Tập hợp hợp lệ bao gồm các chủ sở hữu TLD còn sống (tuyên bố =
  PoW đã bỏ ra) — bề mặt Sybil bị chặn bởi chi phí PoW trên mỗi
  khóa;
- `state_root` của checkpoint chỉ được xác minh trong RAM khi
  checkpoint nhắm vào đỉnh; phát lại hồi tố là một đường dẫn
  storage/relay (xem `docs/technical/blockchain.md`).

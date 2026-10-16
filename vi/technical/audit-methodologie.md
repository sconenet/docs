# Phương pháp luận kiểm toán bảo mật — Scone (trước mainnet)

Tài liệu làm việc. Mục tiêu: sự tự tin cho triển khai sản xuất.
Tham chiếu xuyên crate: các quy tắc cứng của AGENTS.md (không mật
mã tự chế, giải mã nghiêm ngặt, băm được tính lại, tính tất định)
là đặc tả yêu cầu của cuộc kiểm toán.

## 0. Phạm vi và mối đe dọa (trước khi đọc lại bất kỳ dòng mã nào)

Mô hình đối thủ: một nút p2p thù địch tùy ý (có thể gửi bất kỳ byte
nào qua QUIC/UDP), một người gửi tx thù địch, một vận hành viên
relay tò mò, một thư mục dữ liệu bị đánh cắp. BỊ LOẠI TRỪ theo
thiết kế: client cuối (DNS vốn công khai theo bản chất), tính
sẵn sàng toàn cục (chaos đã kiểm thử).

Bảng tài sản, theo thứ tự tới hạn:
1. **Quyền sở hữu** (chuỗi trạng thái: Register/Update/Transfer/Assign) —
   một dương tính giả = đánh cắp tên miền.
2. **Nắm giữ chữ ký/khóa** (keystore, Ed25519, dẫn xuất
   OwnerId).
3. **Xác thực đầu vào không tin cậy** (giải mã wire, giới hạn,
   panic).
4. **DoS tài nguyên** (cache DNS, mempool, DHT, sync).
5. **Tính tất định/đồng thuận** (reorg, GC, hồi sinh trạng
   thái).

## 1. Kiểm toán tĩnh tự động (ngày 1 — cơ học)

- `cargo audit` (CVE phụ thuộc: libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + rà soát `unsafe` (kỳ vọng ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (đã xanh, giữ làm rào CI).
- Fuzz các ranh giới giải mã (ngày 2-3):
  `cargo-fuzz` trên `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — đây CHÍNH LÀ
  bề mặt nơi "không bao giờ panic trên dữ liệu không tin cậy"
  phải đứng vững. Ngân sách: 24 h fuzz liên tục, corpus gieo từ
  các kiểm thử hiện có. Tiêu chí thoát: không crash, không
  timeout, độ phủ các nhánh giải mã > 95%.
- Fuzz bộ phân tích wire DNS (dns.rs `parse_query`,
  `extract_answers`) và bộ phân tích tệp bản ghi (domain.rs).

## 2. Rà soát thủ công có chủ đích theo thành phần (ngày 4-8)

Sắp theo mức tới hạn của tài sản, 1 thành phần = 1 danh sách
kiểm tra:

### 2.1 scone-crypto + dẫn xuất danh tính (ngày 4)
- OwnerId = BLAKE3(pubkey): kiểm tra chuẩn hóa byte (không có
  biểu diễn mơ hồ → 2 khóa, 1 chủ).
- Ed25519: GIẢI MÃ nghiêm ngặt (khóa không chuẩn bị từ chối?),
  xác minh với tách biệt ngữ cảnh (mỗi lần dùng có thẻ miền riêng:
  SCONE-TX-SIG-V1, SCONE-BLOCK-V1, bản ghi DHT — không thể phát
  lại chéo giao thức).
- Keystore: cụm mật khẩu → khóa (KDF? số vòng?), zeroization,
  quyền tệp, oracle đệm trên bản mã.

### 2.2 Áp dụng giao dịch (ngày 5) — trái tim
Với TỪNG biến thể Transaction (hiện 13):
- Tx có lũy tính khi phát lại không (txid duy nhất, nonce/sequence)?
- Kiểm tra sở hữu có đi trước mọi hiệu lực không? (người ký ==
  chủ được học on-chain, không bao giờ tin từ chính tx.)
- Sequence UpdateDomain: đúng hiện-tại+1 (không phải ≥) — kiểm
  tra việc từ chối khoảng trống và từ chối phát lại.
- Renew: biên thời gian (≤ 3 năm, > bây giờ) — thời gian đến từ
  KHỐI, không bao giờ đồng hồ cục bộ (quy tắc đã đặt, kiểm tra
  mọi call site).
- AssignDomain/TransferTld: pubkey của bên nhận có ĐƯỢC HỌC
  (bẫy tập hợp không-pubkey — xem skill) trên mọi đường dẫn.
- GC/hết hạn: không thể hồi sinh (nghịch lý
  precheck-OK/lookup-OK/push-thất-bại) — kiểm tra
  gc_expired_journaled.

### 2.3 Tầng mạng (ngày 6)
- Mọi byte từ xa đi qua một bộ giải mã chặt có giới hạn TRƯỚC khi
  cấp phát (MAX_* của limits.rs) — tìm `Vec::with_capacity(input_len)`
  không giới hạn.
- requery/sync GetBlocks: giới hạn trên mỗi peer và mỗi chu kỳ
  (đã có: 128 khối/yêu cầu) — kiểm tra không có vòng tải-lại.
- Gossip Record: xác minh lại chuỗi TRƯỚC cache/gossip
  (được xác minh bởi các kiểm thử record_dht của tôi) — kiểm cả
  nắp fanout.
- kad DHT: một peer thù địch có thể ép lưu N bản ghi (hạn ngạch
  trên mỗi peer?) hoặc trục xuất có chủ đích các bản ghi trung
  thực?

### 2.4 Bề mặt DNS (ngày 7)
- Cache: không thể tiêm (khóa chuẩn (qname,qtype) — không va
  chạm hoa/thường/mã hóa), ngân sách bộ nhớ cứng (đã kiểm thử),
  không đầu độc cache qua đường fallback (TXID + echo câu hỏi —
  xác minh upstream không thể bị giả mạo từ xa).
- Câu trả lời: AA trung thực, TC trung thực, không khuếch đại
  (tỷ lệ phản hồi/yêu cầu bị chặn — MAX_INFLIGHT_UDP đã có, kiểm
  tra sao chép TCP).

### 2.5 Lưu trữ & genesis (ngày 8)
- redb: hỏng đĩa → lỗi có kiểu, không panic (kiểm thử diskfail
  đã có phía signer — tương đương relay?).
- Genesis: băm phủ toàn bộ manifest; hai genesis khác nhau = từ
  chối thư mục dữ liệu (đã kiểm thử); xác minh tính bất khả xâm
  nhập một genesis HẬU QUẢ vào một thư mục dữ liệu đã đồng bộ.

## 3. Kiểm thử đối thủ (ngày 9-12) — viết những gì còn thiếu

Với mỗi phát hiện của rà soát thủ công, một kiểm thử chứng minh
nó:
- nút thù địch trong mô phỏng (SimNet với một nút gửi thông điệp
  sai dạng/không đồng ý) — mở rộng SimNet với vai trò "byzantine
  bị động";
- mạng lưới 30 nút thật với 3 nút thù địch (byte ngẫu nhiên,
  loạt tx vô hiệu, bản ghi DHT giả mạo) — bộ chaos đã có, thêm
  vai trò kẻ tấn công;
- đối chiếu chéo: hai triển khai của việc giải mã (tài liệu
  protocol.md mang tính CHUẨN — một kiểm toán viên triển khai
  wire chỉ từ tài liệu và so sánh byte; bất kỳ khác biệt nào =
  bug tài liệu HOẶC mã).

## 4. Kiểm toán bên ngoài (nếu có ngân sách) — sau 1-3

Phạm vi tối thiểu có lời cho nhà cung cấp: scone-crypto,
scone-protocol (wire), áp dụng tx (state.rs), ~8-10 k LOC.
Định dạng: 2 tuần, 1 kiểm toán viên mật mã senior + 1 người rà
soát. Sản phẩm: báo cáo với mức nghiêm trọng, PoC, các sửa chữa
được rà. Dựa vào tài liệu chuẩn hiện có (docs/technical) làm nền
tảng của đặc tả yêu cầu — đây là một lợi thế chi phí thực sự.

## 5. Tiêu chí thoát (go/no-go mainnet)

- [ ] cargo-fuzz 24 h không crash trên 5 bộ giải mã
- [ ] Rà soát 2.2 hoàn tất: mỗi biến thể tx có phiếu riêng của
      nó (sở hữu/phát lại/thời gian/hiệu lực) được người rà ký
      duyệt
- [ ] Mạng 30 nút gồm cả nút thù địch: 48 h không lệch đỉnh
      hay hỏng cache
- [ ] Khác biệt tài liệu/mã: bằng không (xác minh bằng triển
      khai wire kép)
- [ ] CVE chặn: bằng không (cargo audit)
- [ ] Kiểm toán bên ngoài phạm vi mật mã/wire/tx đã xong (nếu
      được cấp ngân sách)

## Thứ tự thực thi khuyến nghị

1→2.1→2.2 (trái tim, 60% rủi ro)→2.3→2.4→2.5→3 song song với
từng rà soát (kiểm thử đối thủ được viết ngay trong ngày phát
hiện, không phải sau).

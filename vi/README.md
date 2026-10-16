# Tài liệu Scone

> Toàn bộ tài liệu trong `docs/` mang tính **CHUẨN NHẤ (NORMATIVE)**: chúng
phải cho phép một triển khai độc lập mà không cần đọc mã Rust. Bất kỳ
sự khác biệt nào giữa tài liệu và mã đều là một bug (xác định nguồn
chân lý và sửa nó).

Hai phần:

- **`general/`** — cách hoạt động: các khái niệm và quy tắc của giao
  thức, tiếp cận được mà không cần đọc chi tiết các định dạng;
- **`technical/`** — đặc tả chi tiết dành cho nhà phát triển: định dạng
  nhị phân chính xác, các công thức băm, giới hạn, API và hành vi của
  từng thành phần.

## `general/` — cách hoạt động

| Tài liệu | Nội dung |
|---|---|
| [architecture.md](general/architecture.md) | Sự tách biệt giữa thẩm quyền (blockchain) / tính sẵn sàng (DHT) / lưu trữ cục bộ; vai trò của các crate; luồng phân giải DNS; khả năng tiến hóa |
| [naming.md](general/naming.md) | Quy tắc đặt tên: bảng chữ cái LDH, TLD, tên miền và tên miền con, chuẩn hóa, dẫn xuất `DomainId` và biểu diễn thập lục phân |

## `technical/` — đặc tả dành cho nhà phát triển

| Tài liệu | Nội dung |
|---|---|
| [protocol.md](technical/protocol.md) | Định dạng nhị phân wire chuẩn: varint LEB128, định danh, bản ghi DNS, khối, thông điệp P2P, giới hạn, phiên bản |
| [transactions.md](technical/transactions.md) | Định dạng giao dịch có chữ ký v1: các trường, thứ tự wire (RegisterDomain mang tên, discriminant tùy ý), dẫn xuất owner, payload ký `SCONE-TX-SIG-V1`, quy tắc xác thực |
| [blockchain.md](technical/blockchain.md) | Tầng blockchain: `TxId`, cây Merkle, `BlockHash`, khối genesis, trạng thái chuẩn, xác thực khối, trừu tượng hóa đồng thuận |
| [keystore.md](technical/keystore.md) | Định dạng tệp khóa `.sconekey` (Argon2id + XChaCha20-Poly1305), API `scone-keystore`, các lệnh `identity` |
| [storage.md](technical/storage.md) | Lưu trữ cục bộ bằng redb: lược đồ bảng, delta nguyên tử, chính sách bộ nhớ, mô hình tin cậy, giới hạn chống DoS |
| [relay.md](technical/relay.md) | Relay mạng (`scone-network`): các giao thức libp2p, khám phá peer (seed, thư mục, mDNS, UPnP), khả năng kết nối (AutoNAT, circuit-relay, DCUtR), mempool, đồng bộ, DHT Kademlia, cấu trúc liên kết cục bộ, chỉ số Prometheus, snapshot DNS, CLI đầy đủ, cấu hình `.env` |
| [dns.md](technical/dns.md) | Máy chủ DNS UDP + TCP (M6/M7b): snapshot DNS có thẩm quyền, cache đường nóng chia 16 shard (mục dùng chung, tra cứu không cấp phát, bản ghi đóng gói, nén tên), codec RFC 1035, TCP RFC 7766 (ngân sách 512 byte UDP / 64 KiB TCP, bit TC), cache âm tính DHT, chuyển tiếp đệ quy, giới hạn |
| [cli.md](technical/cli.md) | Tệp nhị phân `scone`: ghi nhật ký và mức chi tiết (stdout/stderr, các cấp `-v`, `RUST_LOG`), cấu hình môi trường qua `.env` (thứ tự ưu tiên cờ > `.env` > mặc định) |
| [security-model.md](technical/security-model.md) | Mô hình bảo mật: lựa chọn ủy ban, chống grinding seed (phân tích: VRF không cần thiết), equivocation và slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Phương pháp luận kiểm toán bảo mật: sự khác biệt, nguồn chân lý, các vòng sửa chữa |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC proof-of-service: heartbeat của vận hành viên, ủy quyền thao tác, bầu chọn ủy ban trong số các ghế còn mới |

## Quy ước tham chiếu chéo

Các tài liệu nội bộ và chú thích mã Rust tham chiếu các trang này bằng
đường dẫn tuyệt đối từ thư mục gốc của kho
(ví dụ. `/docs/en/technical/protocol.md`).

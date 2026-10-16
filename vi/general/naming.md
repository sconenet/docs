# Đặt tên

## Bảng chữ cái

Chữ ASCII thường, chữ số và dấu gạch nối (LDH, RFC 1123):
`[a-z0-9-]`, với dấu gạch nối không đứng ở vị trí đầu tiên cũng không
ở vị trí cuối cùng của một nhãn. Không chữ in hoa, không dấu gạch dưới.

Việc xác thực là **nghiêm ngặt** đối với ASCII: một tên không chuẩn sẽ
bị từ chối, không bao giờ tự động sửa. `"ABC"` → lỗi (không tự động
chuyển thành chữ thường). Lý do: một ranh giới giao thức phải tất
định; việc chuẩn hóa ngầm tạo ra sự mơ hồ về danh tính.

## Tên quốc tế hóa (IDNA)

Tên không dựa trên chữ Latinh được chấp nhận làm **đầu vào** và mã hóa
thành punycode (IDNA, RFC 5890/5891, UTS#46): `例子.uip` có dạng
chuẩn là `xn--fsqu00a.uip`.

- dạng chuẩn (on-chain, id BLAKE3, DNS, lưu trữ) luôn là **A-label**
  ASCII: mọi quy tắc LDH ở trên áp dụng cho dạng đó;
- dạng unicode gốc (**U-label**) được giữ trong bộ nhớ để hiển thị
  (`DomainName::unicode()` / `TldName::unicode()`);
- giới hạn 63 byte trên mỗi nhãn và giới hạn tổng 253 byte áp dụng
  cho dạng A-label chuẩn, chứ không phải UTF-8 thô;
- ASCII vẫn đi trên đường nghiêm ngặt: IDNA chỉ áp dụng cho đầu vào
  không phải ASCII (`"ABC"` vẫn không hợp lệ);
- chính sách chống đồng hình (chữ viết hỗn hợp, ký tự dễ nhầm):
  chưa lọc ở giai đoạn này, chỉ dùng UTS#46.

## TLD

Một TLD là một nhãn DNS như mọi nhãn khác: giới hạn nhãn chung
(RFC 1035: 63 byte) được áp dụng, không có giới hạn riêng nào ngắn
hơn cho TLD.

- mẫu: `[a-z0-9-]{1,63}` (LDH, không dấu gạch nối ở đầu hoặc cuối),
  trên dạng A-label chuẩn
- giới hạn trong mã: `TldName::MAX_LEN = 63` byte
- hợp lệ: `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, TLD dài đúng
  63 byte, `éxemple` (→ `xn--exemple-2ua`), `例子`
  (→ `xn--fsqu00a`)
- không hợp lệ: TLD có A-label vượt quá 63 byte, `ABC`, `-ab`,
  `ab-`, chuỗi rỗng

## Tên miền

- hình dạng: `label[.label]*.tld` — ít nhất hai nhãn (`name.tld`);
- mỗi nhãn: LDH `[a-z0-9-]{1,63}`, dấu gạch nối không ở đầu cũng
  không ở cuối;
- tổng chiều dài: ≤ 253 byte;
- không có nhãn rỗng (`a..b`, `.a`, `a.` không hợp lệ);
- nhãn cuối cùng phải là một TLD hợp lệ.

### Tên miền con

Kiểu `DomainName` chấp nhận toàn bộ hệ thống phân cấp ngay từ đầu:

```text
example.uip
shop.example.uip
api.shop.example.uip
```

Một tên miền con có `DomainId` **khác biệt** với tên miền cha của nó.
Các quy tắc phân bổ tên miền con (quyền của chủ sở hữu cha, v.v.) là
công việc tương lai.

## Chuẩn hóa

Một tên đã được xác thực vốn đã là chuẩn: `canonical() == as_str()`.
Do đó ngày nay bị từ chối: chữ in hoa, dấu câu khác dấu phân tách `.`,
khoảng trắng, unicode, ký tự vô hình, v.v.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || canonical_name)
```

- tất định: cùng một tên chuẩn ⇒ cùng một id, ở mọi nơi, mãi mãi;
- đơn ánh trên thực tế: hai tên khác nhau ⇒ hai id khác nhau;
- tên thô **không bao giờ** được dùng làm định danh nội bộ
  (khóa blockchain, khóa DHT) — chỉ để nhập và hiển thị;
- tiền tố `SCONE-DOMAIN-V1` cô lập các miền dẫn xuất: thay đổi quy
  tắc (bảng chữ cái, chuẩn hóa) tương đương với việc tạo một không
  gian tên có phiên bản mới (một tiền tố dẫn xuất mới), mà không cần
  viết lại cái hiện có.

### Biểu diễn văn bản

Mã hóa thập lục phân **chữ thường** (`0-9a-f`) của 32 byte thô:
đúng 64 ký tự, không tiền tố (`0x`), không bao giờ cắt ngắn.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

Dạng này được tạo ra bởi `Display` (`Debug` của Rust in
`DomainId(<hex>)`, cùng một chuỗi). Bất kỳ đầu ra nào được con người
hay máy ngoài định dạng nhị phân chuẩn tiêu thụ — nhật ký, thông báo
lỗi, đầu ra CLI — PHẢI dùng biểu diễn này và không biểu diễn nào
khác (không cắt ngắn, không tiền tố, không chữ in hoa).

## Giới hạn và quy tắc tương lai (chưa định xong)

- tên quốc tế hóa (IDN) qua punycode;
- chiều dài tối thiểu của nhãn cấp hai;
- hạn/kéo dài cho việc claimed tên miền.

## TldId (M7a)

Một TLD có thể đăng ký mang danh tính riêng của nó, khác biệt với
danh tính của bất kỳ tên miền nào:

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- tiền tố `SCONE-TLD-V1` **khác biệt** với `SCONE-DOMAIN-V1`:
  không gian id của TLD và của tên miền tách rời ngay từ đầu —
  đăng ký TLD `uip` (một tuyên bố chủ quyền trên không gian tên
  `*.uip`) không bao giờ có thể chiếm dụng danh tính của một tên
  miền, và ngược lại;
- cùng quy tắc hiển thị như `DomainId`: thập lục phân chữ thường,
  64 ký tự, không bao giờ cắt ngắn;
- vectơ ghim (dẫn xuất được ghim bằng kiểm thử):
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

Giao dịch `RegisterTld` (tuyên bố có chữ ký đối với một TLD, `owner`
luôn được tính lại từ khóa nhúng) là một phần của enum
`Transaction` và định dạng wire kể từ M7b: discriminant
`0x93` (hằng số mờ tùy ý), payload ký
`SCONE-TX-SIG-V1`, quy tắc đầy đủ trong
`/docs/en/technical/transactions.md`.
Trong khi đó, `RegisterDomain` của một tên miền mang tên chuẩn dưới
dạng văn bản rõ.

## Dự trữ / quản trị TLD

Được mở bởi M7a: `RegisterTld` tuyên bố một TLD còn tự do vì lợi ích
của danh tính được dẫn xuất từ khóa ký. Kể từ M8b việc quản trị đã
hoàn chỉnh: tuyên bố bằng PoW (độ khó theo từng mạng) mở một hạn
3 năm có thể gia hạn (`RenewTld`), không gian tên **đóng** theo mặc
định (chỉ-assign), mở công khai tường minh (`SetTldOpen`), chuyển
nhượng (`TransferTld`), từ bỏ (`RevokeTld`, có thể tuyên bố lại).
TLD tư nhân sống 3 năm (thời gian hoãn 60 ngày, việc quản lý bị
đóng băng trong thời gian hoãn); TLD mở mang mức sàn 1 năm được duy
trì bởi các tên miền của chúng; tên miền hết hạn sau 1 năm (gia hạn
được, thời gian hoãn 30 ngày — xem
`/docs/en/technical/blockchain.md`).

**TLD gốc của ICANN**: bị dự trữ. `RegisterTld` trên một TLD thuộc
danh sách IANA được ghim (`is_icann_tld`, ~1.440 mục + RFC 2606) bị
từ chối với một lỗi có kiểu (`IcannTldReserved`) — root di sản
thuộc về DNS hiện có. Tại thời điểm phân giải, các tên này được
chuyển tiếp đến các upstream đã cấu hình (`--dns-upstream`) hoặc
REFUSED nếu không có upstream. Các TLD mà ICANN bổ sung sau đó vẫn
có thể được tuyên bố (danh sách ghim: tính nhất quán > tính mới).

## Kinh tế vòng đời — ai trả gì, lúc nào

Mô hình ủy quyền theo theo bản chất của tài nguyên: công việc (PoW)
trả cho thứ đi vào tài sản chung, một chữ ký (PoS) chi phối thứ
thuộc về ai đó. Tóm tắt chuẩn:

### TLD tư nhân

- **Sinh — PoW**: được tuyên bố bằng bằng chứng công việc, độ khó
  theo chiều dài (mainnet: 1 ký tự ≈ 34 ngày tính toán; sàn ≈ 4,3
  ngày ở 10+ ký tự). Người tuyên bố trở thành chủ sở hữu và nắm một
  **ghế PoS** (và đào tạo trình xác thực — yêu cầu heartbeat để
  giữ sự mới, có ủy quyền cho vận hành viên).
- **Sống — PoS**: chữ ký của chủ sở hữu ủy quyền mọi thứ trong không
  gian tên: `assign` (cấp một tên miền), `set-tld-open`
  (không thể đảo ngược — **giải phóng ghế**), `transfer`, `revoke`,
  và việc gia hạn các tên miền sống dưới nó.
- **Hạn — 3 năm**: lời tuyên bố mở một hạn 3 năm CỐ ĐỊNH
  (`TLD_TERM_SECS`). **Gia hạn — PoW bằng đúng lúc tạo**
  (`RenewTld`, cùng đường cong độ khó theo chiều dài; challenge bị
  ràng buộc vào hạn hiện tại — chống tái sử dụng): mỗi hạn sống
  đều được trả bằng công việc.
- **Hết hạn — thời gian hoãn 60 ngày, quản lý đóng băng**: quá thời
  hạn, một thời gian hoãn 60 ngày (`TLD_GRACE_SECS`) vẫn cho phép
  gia hạn, nhưng mọi thao tác quản lý ngoài gia hạn đều bị đóng
  băng — đáng chú ý là `assign`: một chủ sở hữu hết hạn mất quyền
  quản lý cho đến khi họ gia hạn. Các tên miền đã được cấp bên dưới
  sống cuộc đời riêng của chúng (`valid_until` riêng) — TLD đóng
  băng, chứ không phải các con của nó.
- **Chết — GC**: hạn + thời gian hoãn trôi qua mà không gia hạn →
  TLD biến mất (ghế PoS được giải phóng, bất kỳ ai cũng có thể
  tuyên bố lại bằng một PoW mới). Một `TransferTld` mang theo hạn
  còn lại (không đặt lại).

### TLD mở (sinh ra đã mở)

- **Sinh — PoW giảm** (rẻ hơn một phần ba so với tư nhân: một tài
  nguyên chung không ai chi phối). **Không có ghế PoS**.
- **Sàn — 1 năm** (`OPEN_TLD_FLOOR_SECS`): lúc sinh (hoặc lúc mở
  bằng `SetTldOpen`), một mức sàn 1 năm ngăn việc tạo-hủy tức thời
  một TLD không có tên miền nào.
- **Sống**: không ai ký bất cứ thứ gì — các tên miền tự đăng ký
  phục vụ bản thân. Mỗi tên miền còn sống bên dưới đẩy đồng hồ cái
  chết lùi lại (nó bám theo hạn của tên miền mới nhất, bị chặn bởi
  mức sàn).
- **Chết — GC** khi không còn tên miền nào sống bên dưới và đã quá
  sàn/hạn; tên lại trở nên có thể tuyên bố.

### Tên miền dưới một TLD mở

- **Sinh — PoW** (đường cong giảm so với TLD: mainnet 1 ký tự ≈ 8,5
  ngày, sàn ≈ nửa ngày).
- **Gia hạn — PoW giống hệt lúc sinh** (cùng đường cong độ khó;
  challenge bị ràng buộc vào hạn hiện tại, do đó khác với lúc đăng
  ký và mọi lần gia hạn trước đó): mỗi hạn sống đều được trả bằng
  công việc, một tên miền không thể bị "thu hoạch" miễn phí (≤ 3
  năm trước).
- **Chết**: hết hạn → thời gian hoãn 30 ngày (có thể gia hạn) → GC
  → tên trở nên có thể tuyên bố bởi **bất kỳ ai** bằng một PoW mới.

### Tên miền dưới một TLD tư nhân

- **Sinh — PoS của chủ TLD**: chỉ chủ sở hữu (hoặc vận hành viên
  được ủy quyền của họ) mới cấp (`assign`) — chữ ký là thẩm quyền,
  không có PoW.
- **Gia hạn — PoS của chủ TLD**: ký bởi chủ TLD, trường `proof`
  rỗng.
- **Chết**: thời gian hoãn rồi GC; việc tuyên bố lại phải đi qua
  `assign` của chủ sở hữu một lần nữa.

### Bất biến xuyên suốt

- Mọi PoW đều được muối bằng **epoch**, với một cửa sổ hiệu lực
  **hồi tố** tỷ lệ thuận với độ khó (một cuộc đào nhiều tuần vẫn
  được tính ngay cả khi epoch đã đổi).
- Sự equivocation của người ký PoS bị **slash** (cấm vĩnh viễn — M9).
- Mọi thao tác đều xác minh lại toàn bộ tại thời điểm áp dụng:
  không có gì được tin theo lòng tin (băm, bằng chứng và chữ ký đều
  được tính lại).

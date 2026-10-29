# CLI (`scone`) — ghi nhật ký và mức chi tiết

> Hành vi chuẩn của việc ghi nhật ký của tệp nhị phân `scone`. Bất kỳ
> sự khác biệt nào giữa tài liệu và mã = bug (sửa nguồn chân lý).

## Nguyên tắc

- **stdout được dành cho đầu ra lệnh** (`scone status`,
  `identity list`, `status`, …). Một script có thể chuyển hướng
  stdout mà không bao giờ tìm thấy nhật ký trong đó — pipe vẫn đáng
  tin cậy. Ngoại lệ được chấp nhận: `scone relay` phát các dòng dữ
  liệu `p2p: <multiaddr>`, `dns: <addr>` và `metrics: <addr>` trên
  stdout ngay khi các listener được buộc
  (dữ liệu dùng được cho `--bootstrap` / cấu hình, không phải nhật
  ký; mỗi dòng phát đúng một lần, flush ngay lập tức).
- **Mọi nhật ký đều đi vào stderr** (định dạng `tracing` gọn),
  kể cả của relay — relay đã ghi vào stderr trước khi di trú.
- Các thông báo lỗi hướng người dùng của CLI (`scone: <error>`) ở
  lại stderr: đây là UX, không phải nhật ký.

## Các cấp và `--verbose` / `-v`

| Lời gọi | Cấp hiệu lực |
|---|---|
| `scone <cmd>` (không `-v`) | `warn` |
| `scone relay` (không `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (trở lên) | `trace` |

`-v`/`--verbose` là một cờ **toàn cục và lặp lại được**: được chấp
nhận trước hoặc sau lệnh con (`scone -vv identity list` và
`scone identity list -vv` là tương đương).

**Vì sao `scone relay` giữ INFO mặc định**: relay là một daemon —
nó phải ghi lại hoạt động của mình (lắng nghe RPC/p2p, các khối
được chấp nhận và tạo ra, peer kết nối, các lần từ chối) để vận
hành được, mà không cần một cờ nào. Ngược lại, các lệnh dùng một
lần giữ im lặng (WARN) để không làm bẩn một terminal hay một
script.

## `RUST_LOG` (ghi đè)

Nếu biến môi trường `RUST_LOG` được đặt, nó **thay thế hoàn toàn**
cấp dẫn xuất từ `-v` (cú pháp `env-filter`):

```bash
RUST_LOG=debug scone identity list        # debug không cần -v
RUST_LOG=error scone -vv relay            # error thắng -vv
RUST_LOG=scone_network=trace,info scone relay   # theo target
```

## Cấu hình môi trường (`.env`)

Trước khi phân tích tham số, `scone` nạp một tệp `.env` từ thư mục
hiện tại (`dotenvy::dotenv()` trong `main.rs`). Tệp thiếu =
no-op im lặng, không bao giờ là lỗi. Các giá trị nạp được thành
chuẩn cơ sở của các tùy chọn relay khai báo một biến môi trường
(`env = "…"` trong `cli.rs`).

Thứ tự ưu tiên (một tùy chọn, ba nguồn):

```text
explicit CLI flag  >  .env variable  >  built-in default
```

- một cờ được truyền trên dòng lệnh LUÔN thắng;
- nếu không, biến (dù nạp từ `.env` hay đặt trong môi trường tiến
  trình thực — cùng một biến, cùng hiệu ứng);
- nếu không, mặc định dựng sẵn của clap.

Các biến lặp được (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`) tích
lũy: một dòng cho mỗi giá trị trong `.env` (dotenvy nối thêm).

Bảng đầy đủ các biến `SCONE_*` (mạng, lắng nghe P2P, bootstrap,
genesis, UPnP, bề mặt DNS, chỉ số, hồ sơ cấu trúc liên kết,
anchor) và ngữ nghĩa của từng biến nằm trong
[`relay.md`](relay.md#environment-configuration-env); mẫu có chú
thích là `.env.example` ở thư mục gốc của kho (sao chép sang
`.env`, không bao giờ commit — `.env` bị gitignore).

## Ngữ nghĩa các cấp (quy ước mã)

- `error!`: thất bại (hiện không có call site — thất bại mạng bị
  nuốt thành `warn!` và chỉ thất bại cục bộ nghiêm trọng nổi lên
  thành một lỗi có kiểu từ `run()`);
- `warn!`: các mục mạng thù địch hoặc độc hại bị đẩy lùi, tx cũ
  bị đẩy ra khỏi mempool, tx xung đột bị bỏ trong quá trình tạo
  khối, thất bại outbound/inbound, lệch khóa DHT;
- `info!`: khối được chấp nhận/tạo ra, lắng nghe RPC/p2p, peer
  kết nối, khởi động relay (checkpoint trạng thái);
- `debug!`: chi tiết nội bộ (tx được chấp nhận, peer ngắt kết nối,
  vòng RPC, liệt kê keystore, sự kiện kad dưới
  `SCONE_KAD_DEBUG`);
- `trace!`: dành cho payload (hiện không có call site).

Các crate thư viện (`scone-network`, …) phát các macro `tracing`
nhưng **khởi tạo không subscriber nào**: việc khởi tạo chỉ sống
trong tệp nhị phân `scone` (trước khi phân phát lệnh con — relay
cần nhật ký của nó ngay từ lúc khởi động).


## Sử dụng đơn giản (cli-simplify-v1)

- **Không cần `--peer`**: mặc định các lệnh nói chuyện với relay
  cục bộ; biến môi trường `SCONE_PEER` hoặc `--peer` ghi đè.
- **Không cần `--identity`** khi keystore giữ một danh tính duy
  nhất (nó được dùng âm thầm; nhiều hơn → lỗi liệt kê chúng).
- Đã gỡ bỏ (các stub gợi ý thay thế): `show`, `submit`,
  `lookup`, `owned`.
- Mới: `scone domain renew <name>`, `scone domains`, `scone domain
  get|set` (bản ghi đơn, không tệp).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## Vòng đời TLD (cli-unified-lifecycle)

`scone tld register <tld>` là điểm vào TLD bằng một lệnh duy nhất,
đối xứng với `scone domain register` — trạng thái on-chain được
đọc TRƯỚC, rồi hành động đúng mới chạy:

- **TLD tự do** → được đăng ký, sinh ra **MỞ mặc định** (không cờ:
  không gian tên tự phục vụ, 1/3 PoW). `--private` chọn một không
  gian tên sở hữu riêng với giá PoW đầy đủ, kèm xác nhận tường
  minh (quyết định kinh tế). `--yes` bỏ qua xác nhận (script).
- **của bạn, hết hạn trong vòng 90 ngày** → GIA HẠN được đề xuất
  (PoW ràng buộc vào hạn hiện tại — chống tái sử dụng) và chạy sau
  khi xác nhận.
- **của bạn, còn xa hạn** → không có gì để làm (`tld renew --force`
  để gia hạn dù sao).
- **của người khác** → lỗi có kiểu rõ ràng kèm chủ sở hữu.

```bash
scone tld register mytld            # sinh mở (mặc định)
scone tld register mytld --private  # không gian tên sở hữu (xác nhận)
scone tld renew mytld               # gia hạn tường minh
```

Tương tự, `scone domain register <name>` trên một tên bạn sở hữu
mà hết hạn trong vòng 90 ngày sẽ đề xuất gia hạn thay vì thất bại.

## Ủy nhiệm bản ghi (KSK/ZSK-lite)

Chủ của một tên miền có thể cấp cho CÁC KHÁC quyền ký
`domain update` nhân danh mình — tập ủy nhiệm bản ghi (tối đa 3
khóa, thay thế trọn vẹn ở mỗi lần `set`):

```bash
# cấp (chủ-signed; CLI lấy thế hệ CAS của namespace
# và gắn giao dịch vào nó):
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# kiểm tra (không cần identity):
scone domain delegates list example.uip

# thu hồi tất cả:
scone domain delegates clear example.uip --identity alice
```

Một ủy nhiệm sau đó ký `domain update` đúng như chủ sẽ làm
(`--identity deleg`) — trên chuỗi, trường `owner` của tx là chủ
của TÊN MIỀN trong khi chữ ký đến từ ủy nhiệm. Việc cấp là độc
quyền của chủ (một ủy nhiệm không thể quản lý tập), bị xóa bởi
một chuyển nhượng, và mỗi lần `set`/`clear` tăng bộ đếm biến đổi
của namespace (chống phát lại CAS — xem `transactions.md`): một
lần `set` bị phát lại không thể hồi sinh các khóa đã thu hồi.
Cập nhật của một ủy nhiệm đã bị thu hồi bị từ chối cục bộ ("không
phải chủ cũng không phải ủy nhiệm bản ghi").

## Các lệnh nâng cao

Các mặc định ở trên phủ việc dùng hằng ngày. Cho script và tương
tác web/API, toàn bộ bề mặt vẫn khả dụng:

mỗi thao tác chỉ có đúng MỘT đường dẫn chuẩn; các bí danh gốc cũ
(`show`, `lookup`, `owned`, `renew`) đã bị xóa
(remove-cli-aliases), cùng với đề cập `domain info` — vốn chưa
bao giờ tương ứng với lệnh thật.

Bề mặt hex ngoại tuyến (`tx build|sign|verify`, các lệnh con
`submit tx`) đã BỊ GỠ BỎ (remove-tx-surface-v2): các thao tác M8b
mà nó phục vụ nay là các lệnh hạng nhất —
`scone tld open <tld>` (mở tự phục vụ một chiều, PoW mở được đào
hộ bạn), `scone tld transfer <tld> <owner>`,
`scone tld revoke <tld>`, `scone domain assign <name> <owner>`
(chủ TLD, TLD tư nhân) và `scone domain transfer <name>
<owner>`. Mỗi lệnh: đọc trạng thái, chữ ký, gửi, txid. Việc phân
giải bản ghi DNS chuyển sang `scone domain resolve <name>` (trước
là `record get`); kênh xuất bản hex (`record put`) đã biến mất —
chỉnh sửa đi qua `domain set` / `domain update`.
- `scone domain mine <name>` — chỉ đào bằng chứng đăng ký
  (không cần danh tính, không keystore, không gửi): in JSON
  `scone-domain-pow-v1` `{type, name, network_id, epoch,
  proof}` ra stdout. Đào ủy nhiệm: một máy khác đào hộ bạn,
  khóa của bạn không bao giờ rời khỏi bạn. Pipe hay
  redirect: `scone domain mine n.tld | scone domain register
  n.tld --proof -` (hay `--proof FILE`). Bằng chứng gắn với
  tên và mạng; độ tươi được nút áp đặt lúc áp dụng (cửa sổ
  hiệu lực epoch). Với `--mine-for <pubkey>`, nó thay vào đó
  đào một bằng chứng POW-TO-OWNER cho người thụ hưởng đó và
  in JSON `scone-domain-pow-to-v1` `{type, name, network_id,
  epoch, pow_to, proof}` — chỉ `register --to` / `renew
  --to` cho đúng khóa đó mới tiêu thụ được (challenge nhúng
  nó; kiểm tra không phân biệt hoa thường khi gửi).
- Gửi pow-to-owner (không cần keystore — `--to` loại trừ lẫn
  với `--identity`): `scone domain register <name> --to
  <pubkey>` / `scone domain renew <name> --to <pubkey>` và
  `scone tld register <tld> --to <pubkey>` / `scone tld renew
  <tld> --to <pubkey>` gửi một bằng chứng mà challenge gắn
  khóa NGƯỜI THỤ HƯỞNG; `owner` trên chuỗi được tính lại
  thành `hash(pubkey_dest)` (một renew KHÔNG BAO GIỜ đổi
  chủ). Không có `--proof`, lệnh tự đào bằng chứng; với
  `--proof -`/FILE nó tiêu thụ một payload
  `scone-domain-pow-to-v1` và từ chối bằng chứng đào cho
  người thụ hưởng KHÁC. Kiểu pipe: `scone domain mine n.tld
  --mine-for <pubkey> | scone domain register n.tld --to
  <pubkey> --proof -`.
- `scone domain export <name> [--out FILE]` — ghi bộ bản ghi đã
  được chuỗi xác minh đúng định dạng `update --file` (bảo đảm khứ
  hồi: export → chỉnh sửa → update gửi lại đúng bộ đó). Không có
  `--out`: stdout. `domain resolve` dùng cùng định dạng dễ đọc,
  nên `resolve > zone.txt` nhập trực tiếp được. Các kiểu không có
  cú pháp văn bản (GEOROUTE, không rõ) được xuất thành cảnh báo
  chú thích — không bao giờ bị bỏ âm thầm.

`domain get` cũng in `domain_id` và `valid_until`.

## Benchmark PoW (`scone bench-pow`)

Đo tốc độ băm Argon2id của máy (chính loại băm mà việc xác
minh đồng thuận dùng) và dịch các độ khó của mạng thành các
thời gian chờ cụ thể:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

Kết quả: H/s tổng và theo từng worker, rồi thời gian chờ trung
bình dự kiến mỗi thao tác — chuyển nhượng/gán (giá phẳng), mở
TLD riêng tư, và các sàn đăng ký TLD/tên miền (đường cong chiều
dài). Mạng được chọn quyết định tham số Argon2id (mainnet:
64 MiB, t=2 — chi phí thật; testnet/devnet: hồ sơ nhẹ, bằng
chứng mili-giây). Mỗi worker cấp phát lane Argon2id 64 MiB của
riêng nó — chỉnh `--cores` theo RAM khả dụng.

## Build: dev vs bật-mainnet (mainnet-compile-gate)

Build mặc định **không mang mainnet**: `scone relay --network
mainnet` (và các dịch vụ registrar với `SCONE_NETWORK=mainnet`)
thất bại ngay lập tức kèm gợi ý build lại. Tệp nhị phân phân phối
chính thức được build bằng:

```bash
cargo build --release --features mainnet
```

`scone --version` hiển thị `+mainnet` trên các build như vậy, và
nhật ký khởi động relay ghi `build: mainnet-enabled`.

`--network` mặc định cũng theo build: `mainnet` trên build bật
mainnet (tệp nhị phân chính thức đặt người dùng lên mạng sản
xuất), `testnet` nếu không. Một cờ `--network` tường minh hay
biến môi trường `SCONE_NETWORK` luôn thắng
(thứ tự clap: cờ > env > mặc định). Đây là một rào chắn vận hành
(chạy vô tình, tệp nhị phân dev/CI) — không phải một ranh giới mật
mã; các quy tắc đồng thuận vẫn là phòng tuyến chống nút thù địch.

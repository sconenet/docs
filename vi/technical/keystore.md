# Keystore — đặc tả định dạng tệp khóa `.sconekey`

> Tài liệu chuẩn. Bất kỳ sự khác biệt nào giữa tài liệu này và
> `crates/scone-keystore/src/lib.rs` là một bug (xác định nguồn
> chân lý, sửa nó). Tham chiếu triển khai: crate
> `scone-keystore` (v2).

## Mục đích

Lưu một khóa riêng Ed25519 (*seed*, 32 byte) trên đĩa, mã hóa bằng
một cụm mật khẩu. Khóa **không bao giờ** chạm đĩa dưới dạng văn
bản rõ; mọi bí mật (cụm mật khẩu, vật liệu dẫn xuất) đều được
zeroize trong bộ nhớ sau khi dùng.

## Tổng quan

- **KDF**: Argon2id (v0x13), tham số mặc định `m_cost` = 64 MiB,
  `t_cost` = 3, `p_cost` = 1 (mặc định của crate `argon2`).
- **Mã hóa**: XChaCha20-Poly1305 (AEAD), khóa 32 byte dẫn xuất từ
  KDF, nonce ngẫu nhiên 24 byte cho mỗi tệp.
- **Nội dung được mã hóa**: đúng seed Ed25519 (32 byte); thẻ
  Poly1305 (16 byte) được lưu dưới dạng hậu tố của bản mã (đầu ra
  chuẩn của API `encrypt` của AEAD).

## Niêm phong byte (mô-đun `bytes`, web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}` tạo ra
và tiêu thụ CÙNG định dạng mà không chạm hệ thống tệp: client web
sinh danh tính trong trình duyệt (`scone-crypto` +
bip39) và niêm phong đúng các byte của tệp khóa (tải xuống được,
mở được bởi CLI nguyên trạng — kiểm tra tương tác trong bộ kiểm
thử). Đường dẫn native (`create`/`create_with_phrase`) ủy quyền
cho cùng một phép niêm phong: một kiểu mã hóa duy nhất của định
dạng, ba đích đến.

## Định dạng nhị phân (v2) — header 52 byte + siêu dữ liệu công khai + bản mã

```text
Offset  Size        Field
0       9           magic: "SCONEKEY1" (ASCII — unchanged since v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost in MiB; m_cost_kib = value × 1024)
11      1           t_cost: u8 (Argon2id iterations)
12      16          Argon2id salt (random per file)
28      24          XChaCha20-Poly1305 nonce (random per file)
52      32          public_key: Ed25519 public key (32 bytes, IN THE CLEAR)
84      8           created_at: u64 big-endian (Unix, seconds)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (readable without the passphrase, may be empty)
—       48          ciphertext (32 bytes encrypted seed) || Poly1305 tag (16 bytes)
```

Siêu dữ liệu `public_key`/`created_at`/`label` là công khai: đọc
được **không cần cụm mật khẩu** (`public_metadata_from_bytes`).
Khóa công khai vốn là công khai; nhãn chỉ là tên hiển thị — che
nó không mua được gì mà lại làm mất việc liệt kê không-cần-mật-
khẩu. Tổng chiều dài tùy thuộc vào nhãn: 141 byte với nhãn rỗng
(KEYFILE_LEN + 32 + 8 + 1), cộng thêm chiều dài của nhãn.

**v1 (bị thay thế, đúng 100 byte)**: magic ‖ version=1 ‖ m ‖
t ‖ salt ‖ nonce ‖ ciphertext — khóa công khai KHÔNG được lưu
(một tệp khóa bị khóa vốn nặc danh). Tệp v1 vẫn **đọc được**
(siêu dữ liệu `None`, cần cụm mật khẩu cho `identity show`;
niêm phong lại qua `scone identity restore` sẽ di trú chúng
sang v2).

### Các lựa chọn thiết kế

- **Tham số KDF nằm trong tệp**: `m_cost_mib` và `t_cost` được
  đọc từ header (không cứng khi đọc), cho phép nâng chúng trong
  các phiên bản tương lai mà không làm gãy các tệp hiện có.
- **Giới hạn bảo mật lúc đọc**: một tệp khai báo
  `m_cost_mib` > 64 hoặc `t_cost` = 0 hoặc > 10 bị từ chối
  (`Error::InvalidParams`) *trước* bất kỳ cấp phát nào — một tệp
  độc hại không thể ép một nút cấp phát hàng gigabyte hay lặp vô
  hạn. Mặc định lúc ghi là `m_cost_mib` = 64,
  `t_cost` = 3.
- **Magic + phiên bản tường minh**: một tệp không phải tệp khóa
  (`Error::InvalidFile`) hoặc phiên bản không được hỗ trợ
  (`Error::UnsupportedVersion`) bị từ chối gọn gàng, không
  panic.
- **Sai cụm mật khẩu**: thất bại xác thực Poly1305 không cho biết
  *vì sao* nó thất bại; do đó một cụm mật khẩu sai và một bản mã
  bị hỏng là không phân biệt được và nổi lên cùng một lỗi riêng
  `Error::WrongPassphrase`.

## Dẫn xuất khóa mã hóa

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

Sau đó:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

Không có dữ liệu liên kết (AAD) trong v1. Nonce 24 byte, ngẫu
nhiên theo từng tệp, khiến rủi ro tái sử dụng nonce không đáng
kể (tệp khóa chỉ được ghi một lần).

## API (`scone-keystore`)

| Hàm | Vai trò |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Sinh khóa, ghi tệp khóa mã hóa (tạo thư mục cha); **từ chối ghi đè** tệp đã tồn tại (`Error::KeyfileExists`); trả về khóa trong bộ nhớ |
| `create_overwriting(path, passphrase) -> GeneratedKey` | Như trên, nhưng thay thế tường minh một tệp khóa đã có |
| `open(path, passphrase) -> SigningKey` | Giải mã và trả về khóa ký |
| `list(dir) -> Vec<KeyEntry>` | Liệt kê các tệp `*.sconekey` trong một thư mục (sắp theo tên); thư mục thiếu = danh sách rỗng |

`GeneratedKey.signing_key` bị zeroize khi drop bởi `ed25519-dalek`
(tính năng `zeroize` mặc định). Khóa AEAD dẫn xuất, seed, và văn
bản rõ đã giải mã được bọc trong `zeroize::Zeroizing`:
Drop phủ mọi đường thoát, kể cả lỗi truyền qua `?`.

## Quyền tệp

Trên Unix, tệp khóa được tạo với quyền `0600`
(chỉ chủ sở hữu được đọc/ghi), qua `OpenOptions` +
`set_permissions` — bất kể umask của tiến trình. Sự tồn tại và
tạo là nguyên tử (`create_new`, `O_EXCL`): hai lần sinh đồng
thời không thể âm thầm ghi đè lẫn nhau.

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- Thư mục mặc định: `$HOME/.scone/keys`.
- Tên danh tính được xác thực nghiêm ngặt: rỗng, `.`, `..`, `/`,
  `\\\\`, NUL, dấu chấm ở đầu, và hậu tố `.sconekey` đều bị từ
  chối (tên trở thành tên tệp bên trong thư mục khóa — không
  thành phần đường dẫn nào thoát khỏi keystore).
- Không có `--passphrase-env`, cụm mật khẩu được yêu cầu trên
  terminal **với đầu vào che mờ** (`rpassword`: không echo; hai
  lần với xác nhận cho `generate`).
- `identity generate` **hiển thị cụm từ khôi phục BIP39 (24
  từ)** đúng một lần — entropy 256 bit mới, riêng các từ đã tái
  tạo lại cùng một khóa (`scone identity restore`); chúng không
  bao giờ được lưu ở bất kỳ đâu.
- `identity generate` **từ chối ghi đè** một tệp khóa đã có;
  `--force` thay thế nó tường minh (salt/nonce mới, khóa mới,
  cụm từ mới).
- `identity list` hiển thị, với tệp v2, khóa công khai và nhãn
  **không cần cụm mật khẩu** (siêu dữ liệu công khai).
- `identity show` đọc siêu dữ liệu công khai của tệp khóa v2 và
  hiển thị khóa công khai, `OwnerId` dạng thập lục phân chữ
  thường (64 ký tự, cùng quy ước như `DomainId`), nhãn, và ngày
  tạo **không cần cụm mật khẩu**; tệp v1 (không siêu dữ liệu)
  vẫn yêu cầu nó.

## Những gì định dạng này cố ý không làm

- Không nhiều khóa trong một tệp (một seed mỗi tệp).
- Không đổi cụm mật khẩu (nếu bao giờ có: ghi lại tệp với
  salt/nonce mới).


## Danh tính mặc định (cli-simplify-v1)

Các lệnh cần một danh tính ký (`domain register`,
`domain set`, `domain update`, `domain renew`) dùng danh tính DUY
NHẤT của keystore không cần `--identity` khi chỉ có đúng một.
Với nhiều danh tính, lệnh thất bại và liệt kê chúng
(`scone identity list`); hãy truyền `--identity <name>` tường
minh.

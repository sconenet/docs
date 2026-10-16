# Keystore — spesifikasi format keyfile `.sconekey`

> Dokumen normatif. Setiap perbedaan antara dokumen ini dan
> `crates/scone-keystore/src/lib.rs` adalah bug (identifikasi sumber
> kebenarannya, lalu perbaiki). Referensi implementasi: crate
> `scone-keystore` (v2).

## Tujuan

Menyimpan kunci privat Ed25519 (*seed*, 32 byte) di disk,
terenkripsi dengan passphrase. Kunci **tidak pernah** menyentuh disk
dalam teks polos; semua rahasia (passphrase, material turunan)
di-zeroize di memori setelah dipakai.

## Ringkasan

- **KDF**: Argon2id (v0x13), parameter default `m_cost` = 64 MiB,
  `t_cost` = 3, `p_cost` = 1 (default crate `argon2`).
- **Enkripsi**: XChaCha20-Poly1305 (AEAD), kunci 32-byte yang
  diturunkan dari KDF, nonce acak 24-byte per berkas.
- **Konten terenkripsi**: tepat seed Ed25519 (32 byte); tag
  Poly1305 (16 byte) disimpan sebagai akhiran ciphertext (keluaran
  standar API `encrypt` AEAD).

## Penyegelan byte (modul `bytes`, web)

`scone_keystore::bytes::{seal_to_bytes, open_from_bytes}`
menghasilkan dan mengonsumsi format yang SAMA tanpa menyentuh
filesystem: klien web menghasilkan identitas di browser
(`scone-crypto` + bip39) dan menyegel byte keyfile yang persis
(bisa diunduh, bisa dibuka oleh CLI apa adanya — test
interoperabilitas ada di suite). Jalur native
(`create`/`create_with_phrase`) mendelegasikan ke penyegelan yang
sama: satu enkode format, tiga tujuan.

## Format biner (v2) — header 52-byte + metadata publik + ciphertext

```text
Offset  Ukuran      Field
0       9           magic: "SCONEKEY1" (ASCII — tak berubah sejak v1)
9       1           version: u8 = 2
10      1           m_cost_mib: u8 (m_cost dalam MiB; m_cost_kib = nilai × 1024)
11      1           t_cost: u8 (iterasi Argon2id)
12      16          salt Argon2id (acak per berkas)
28      24          nonce XChaCha20-Poly1305 (acak per berkas)
52      32          public_key: kunci publik Ed25519 (32 byte, TEKS POLOS)
84      8           created_at: u64 big-endian (Unix, detik)
92      1           label_len: u8 (0..=255)
93      label_len   label: UTF-8 (terbaca tanpa passphrase, boleh kosong)
—       48          ciphertext (seed terenkripsi 32 byte) || tag Poly1305 (16 byte)
```

Metadata `public_key`/`created_at`/`label` bersifat publik: terbaca
**tanpa passphrase** (`public_metadata_from_bytes`). Kunci publik
memang publik; label hanyalah nama tampilan — menyembunyikannya
tidak memberi apa pun dan mengorbankan daftar tanpa-passphrase.
Panjang total bergantung pada label: 141 byte dengan label kosong
(KEYFILE_LEN + 32 + 8 + 1), ditambah panjang label.

**v1 (usang, tepat 100 byte)**: magic ‖ version=1 ‖ m ‖
t ‖ salt ‖ nonce ‖ ciphertext — kunci publik TIDAK disimpan (berkas
terkunci bersifat anonim). Berkas v1 tetap **terbaca** (metadata
`None`, passphrase wajib untuk `identity show`; penyegelan ulang
via `scone identity restore` memigrasikannya ke v2).

### Pilihan desain

- **Parameter KDF dalam berkas**: `m_cost_mib` dan `t_cost`
  dibaca dari header (bukan di-hardcode saat membaca),
  memungkinkan kenaikannya di versi mendatang tanpa merusak berkas
  yang ada.
- **Batas keamanan saat membaca**: berkas yang mendeklarasikan
  `m_cost_mib` > 64 atau `t_cost` = 0 atau > 10 ditolak
  (`Error::InvalidParams`) *sebelum* alokasi apa pun — berkas
  berbahaya tidak bisa memaksa node mengalokasikan gigabyte atau
  berputar tanpa batas. Default saat menulis: `m_cost_mib` = 64,
  `t_cost` = 3.
- **Magic + versi eksplisit**: berkas yang bukan keyfile
  (`Error::InvalidFile`) atau berversi tak didukung
  (`Error::UnsupportedVersion`) ditolak dengan bersih, tanpa
  panic.
- **Passphrase salah**: kegagalan autentikasi Poly1305 tidak
  menunjukkan *mengapa* gagal; passphrase salah dan ciphertext
  rusak karena itu tak terbedakan dan muncul sebagai error berbeda
  yang sama `Error::WrongPassphrase`.

## Derivasi kunci enkripsi

```text
key = Argon2id(passphrase, salt, m_cost = m_cost_mib × 1024 KiB,
               t_cost = t_cost, p_cost = 1, output = 32 bytes)
```

Kemudian:

```text
ciphertext || tag = XChaCha20-Poly1305-Seal(key, nonce, plaintext = seed)
```

Tanpa associated data (AAD) di v1. Nonce 24-byte, acak per berkas,
membuat risiko reuse nonce dapat diabaikan (keyfile hanya
ditulis sekali).

## API (`scone-keystore`)

| Fungsi | Peran |
|---|---|
| `create(path, passphrase) -> GeneratedKey` | Menghasilkan kunci, menulis keyfile terenkripsi (membuat direktori induk); **menolak menimpa** berkas yang ada (`Error::KeyfileExists`); mengembalikan kunci di memori |
| `create_overwriting(path, passphrase) -> GeneratedKey` | Sama, tetapi secara eksplisit mengganti keyfile yang ada |
| `open(path, passphrase) -> SigningKey` | Mendekripsi dan mengembalikan kunci penandatangan |
| `list(dir) -> Vec<KeyEntry>` | Mendaftar berkas `*.sconekey` dalam sebuah direktori (diurutkan per nama); direktori hilang = daftar kosong |

`GeneratedKey.signing_key` di-zeroize saat drop oleh
`ed25519-dalek` (fitur `zeroize` default). Kunci AEAD turunan,
seed, dan plaintext terdekripsi dibungkus
`zeroize::Zeroizing`: Drop mencakup semua jalur keluar, termasuk
error yang diteruskan `?`.

## Izin berkas

Di Unix, keyfile dibuat dengan izin `0600` (baca/tulis hanya oleh
owner), via `OpenOptions` + `set_permissions` — terlepas dari
umask proses. Eksistensi dan penciptaan bersifat atomik
(`create_new`, `O_EXCL`): dua generasi bersamaan tidak bisa saling
menimpa diam-diam.

## CLI

```bash
scone identity generate <n> [--dir <directory>] [--passphrase-env VAR] [--force]
scone identity list [--dir <directory>]
scone identity show <n> [--dir <directory>] [--passphrase-env VAR]
```

- Direktori default: `$HOME/.scone/keys`.
- Nama identitas divalidasi ketat: kosong, `.`, `..`, `/`,
  `\\\\`, NUL, titik di awal, dan akhiran `.sconekey` ditolak
  (nama menjadi nama berkas di dalam direktori kunci — tidak ada
  komponen path yang bisa lolos dari keystore).
- Tanpa `--passphrase-env`, passphrase diminta di terminal
  **dengan input tersamar** (`rpassword`: tanpa gema; dua kali
  dengan konfirmasi untuk `generate`).
- `identity generate` **menampilkan frasa pemulihan BIP39 (24
  kata)** tepat sekali — entropi 256-bit segar, kata-kata itu saja
  meregenerasi kunci yang sama (`scone identity restore`); ia
  tidak pernah disimpan di mana pun.
- `identity generate` **menolak menimpa** keyfile yang ada;
  `--force` secara eksplisit menggantinya (salt/nonce baru, kunci
  baru, frasa baru).
- `identity list` menampilkan, untuk berkas v2, kunci publik dan
  label **tanpa passphrase** (metadata publik).
- `identity show` membaca metadata publik keyfile v2 dan
  menampilkan kunci publik, `OwnerId` dalam heks huruf kecil (64
  karakter, konvensi sama dengan `DomainId`), label, dan tanggal
  penciptaan **tanpa passphrase**; berkas v1 (tanpa metadata)
  tetap memintanya.

## Yang sengaja tidak dilakukan format ini

- Tanpa banyak kunci per berkas (satu seed per berkas).
- Tanpa perubahan passphrase (jika suatu saat ada: tulis ulang
  berkas dengan salt/nonce baru).


## Identitas default (cli-simplify-v1)

Perintah yang membutuhkan identitas penandatangan (`domain
register`, `domain set`, `domain update`, `domain renew`)
menggunakan identitas TUNGGAL keystore tanpa `--identity` saat
tepat ada satu. Dengan beberapa identitas, perintah gagal dan
mendaftarkannya (`scone identity list`); berikan
`--identity <name>` secara eksplisit.

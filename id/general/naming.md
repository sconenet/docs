# Penamaan

## Alfabet

Huruf ASCII kecil, digit, dan tanda hubung (LDH, RFC 1123):
`[a-z0-9-]`, dengan tanda hubung tidak boleh di posisi pertama
maupun terakhir sebuah label. Tanpa huruf besar, tanpa garis bawah.

Validasi **ketat** untuk ASCII: nama non-kanonik ditolak,
tidak pernah dikoreksi. `"ABC"` → error (tanpa penurunan huruf
diam-diam). Alasan: batas protokol harus deterministik;
kanonisasi implisit menciptakan ambiguitas identitas.

## Nama internasionalisasi (IDNA)

Nama non-Latin diterima sebagai **input** dan dienkode ke punycode
(IDNA, RFC 5890/5891, UTS#46): `例子.uip` memiliki bentuk kanonik
`xn--fsqu00a.uip`.

- bentuk kanonik (on-chain, id BLAKE3, DNS, penyimpanan) selalu
  **A-label** ASCII: setiap aturan LDH di atas berlaku untuk bentuk
  itu;
- bentuk unicode asli (**U-label**) disimpan di memori untuk
  tampilan (`DomainName::unicode()` / `TldName::unicode()`);
- batas 63 byte per label dan batas total 253 byte berlaku pada
  bentuk kanonik A-label, bukan pada UTF-8 mentah;
- ASCII tetap pada jalur ketat: IDNA hanya berlaku untuk input
  non-ASCII (`"ABC"` tetap tidak valid);
- kebijakan anti-homograf (skrip campuran, karakter yang mudah
  tertukar): belum difilter untuk saat ini, hanya UTS#46.

## TLD

TLD adalah label DNS seperti label lainnya: batas label generik
(RFC 1035: 63 byte) yang berlaku, bukan batas khusus TLD yang lebih
pendek.

- pola: `[a-z0-9-]{1,63}` (LDH, tanpa tanda hubung di awal/akhir),
  pada bentuk kanonik A-label
- batas dalam kode: `TldName::MAX_LEN = 63` byte
- valid: `uip`, `com`, `test`, `abc12`, `0x`, `a-b`, TLD dengan
  tepat 63 byte, `éxemple` (→ `xn--exemple-2ua`), `例子`
  (→ `xn--fsqu00a`)
- tidak valid: TLD yang A-label-nya melebihi 63 byte, `ABC`, `-ab`,
  `ab-`, string kosong

## Domain

- bentuk: `label[.label]*.tld` — minimal dua label (`name.tld`);
- tiap label: LDH `[a-z0-9-]{1,63}`, tanda hubung tidak di awal
  maupun akhir;
- panjang total: ≤ 253 byte;
- tanpa label kosong (`a..b`, `.a`, `a.` tidak valid);
- label terakhir harus merupakan TLD yang valid.

### Subdomain

Tipe `DomainName` menerima hierarki penuh sejak awal:

```text
example.uip
shop.example.uip
api.shop.example.uip
```

Subdomain memiliki `DomainId` yang **berbeda** dari induknya.
Aturan alokasi subdomain (hak owner induk, dsb.) adalah pekerjaan
mendatang.

## Kanonisasi

Nama yang telah divalidasi sudah kanonik:
`canonical() == as_str()`. Karena itu yang ditolak hari ini:
huruf besar, tanda baca selain pemisah `.`, spasi, unicode,
karakter tak terlihat, dll.

## DomainId

```text
DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || canonical_name)
```

- deterministik: nama kanonik yang sama ⇒ id yang sama, di mana
  pun, selamanya;
- injektif dalam praktik: dua nama berbeda ⇒ dua id berbeda;
- nama mentah **tidak boleh pernah** menjadi identifier internal
  (kunci blockchain, kunci DHT) — hanya untuk input dan tampilan;
- prefiks `SCONE-DOMAIN-V1` mengisolasi domain derivasi: mengubah
  aturan (alfabet, normalisasi) sama dengan menciptakan namespace
  berversi baru (prefiks derivasi baru), tanpa menulis ulang yang
  sudah ada.

### Representasi tekstual

Enkode heksadesimal **huruf kecil** (`0-9a-f`) dari 32 byte mentah:
tepat 64 karakter, tanpa prefiks (`0x`), tidak pernah dipotong.

```text
DomainId([0xab; 32]) → "ababababababababababababababababababababababababababababababababab"
DomainId([0x00; 32]) → "0000000000000000000000000000000000000000000000000000000000000000"
```

Bentuk ini dihasilkan oleh `Display` (`Debug` Rust mencetak
`DomainId(<hex>)`, string yang sama). Semua keluaran yang
dikonsumsi manusia atau mesin di luar format biner kanonik — log,
pesan error, keluaran CLI — HARUS menggunakan representasi ini dan
bukan yang lain (tanpa pemotongan, tanpa prefiks, tanpa huruf
besar).

## Batasan dan aturan mendatang (belum ditetapkan)

- nama internasionalisasi (IDN) via punycode;
- panjang minimum label tingkat kedua;
- kedaluwarsa/perpanjangan klaim domain.

## TLDId (M7a)

TLD yang dapat didaftarkan membawa identitasnya sendiri, berbeda
dari identitas domain mana pun:

```text
TldId = BLAKE3-256("SCONE-TLD-V1" || tld)
```

- prefiks `SCONE-TLD-V1` **berbeda** dari `SCONE-DOMAIN-V1`:
  ruang id TLD dan domain terpisah secara konstruksi —
  mendaftarkan TLD `uip` (klaim atas namespace `*.uip`) tidak
  pernah bisa menyerobot identitas sebuah domain, begitu pula
  sebaliknya;
- aturan tampilan yang sama dengan `DomainId`: heks huruf kecil,
  64 karakter, tidak pernah dipotong;
- vektor terpin (derivasi dipin oleh test):
  `TldId("uip") = ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e`.

Transaksi `RegisterTld` (klaim bertanda tangan atas TLD, `owner`
selalu dihitung ulang dari kunci yang tertanam) adalah bagian dari
enum `Transaction` dan format wire sejak M7b: diskriminan
`0x93` (konstanta opaque arbitrer), payload bertanda tangan
`SCONE-TX-SIG-V1`, aturan lengkap di
`/docs/en/technical/transactions.md`.
Sementara itu, `RegisterDomain` sebuah domain membawa nama kanonik
dalam teks polos.

## Reservasi/tata kelola TLD

Dibuka oleh M7a: `RegisterTld` mengklaim TLD bebas demi identitas
yang diturunkan dari kunci penandatangan. Sejak M8b tata kelola
lengkap: klaim dengan PoW (kesulitan per jaringan) membuka masa
3 tahun yang dapat diperpanjang (`RenewTld`), namespace
**tertutup** secara default (hanya assign), pembukaan eksplisit
(`SetTldOpen`), transfer (`TransferTld`), pelepasan (`RevokeTld`,
dapat diklaim ulang). TLD privat hidup 3 tahun (masa tenggang 60
hari, manajemen dibekukan selama masa tenggang); TLD terbuka
membawa lantai 1 tahun yang dipelihara oleh domain-domainnya;
domain kedaluwarsa setelah 1 tahun (dapat diperpanjang, masa
tenggang 30 hari — lihat `/docs/en/technical/blockchain.md`).

**TLD root ICANN**: dicadangkan. `RegisterTld` atas TLD dari daftar
IANA terpin (`is_icann_tld`, ~1.440 entri + RFC 2606) ditolak
dengan error bertipe (`IcannTldReserved`) — root warisan milik DNS
yang sudah ada. Saat resolusi, nama-nama ini diteruskan ke
upstream yang dikonfigurasi (`--dns-upstream`) atau REFUSED tanpa
upstream. TLD yang ditambahkan ICANN kemudian tetap dapat
diklaim (daftar terpin: konsistensi > kesegaran).


## Ekonomi siklus hidup — siapa membayar apa, kapan

Model otorisasi mengikuti sifat sumber daya: kerja (PoW) membayar
apa yang masuk ke ranah publik, tanda tangan (PoS) mengatur apa
yang dimiliki seseorang. Ringkasan normatif:

### TLD privat

- **Lahir — PoW**: diklaim dengan proof of work, kesulitan
  menurut panjang (mainnet: 1 karakter ≈ 34 hari komputasi; lantai
  ≈ 4,3 hari pada 10+ karakter). Pengklaim menjadi owner dan
  memegang **kursi PoS** (kumpulan kelayakan validator — heartbeat
  wajib agar tetap segar, delegasi operator tersedia).
- **Hidup — PoS**: tanda tangan owner mengotorisasi segalanya di
  namespace: `assign` (memberi domain), `set-tld-open`
  (tak dapat dibatalkan — **melepaskan kursi**), `transfer`,
  `revoke`, dan perpanjangan domain yang hidup di bawahnya.
- **Masa — 3 tahun**: klaim membuka masa 3 tahun TETAP
  (`TLD_TERM_SECS`). **Perpanjangan — PoW setara
  penciptaan** (`RenewTld`, kurva kesulitan per panjang yang sama;
  tantangan terikat pada kedaluwarsa saat ini — anti-reuse):
  setiap masa hidup dibayar dengan kerja.
- **Kedaluwarsa — masa tenggang 60 hari, manajemen dibekukan**:
  lewat tenggat, masa tenggang 60 hari (`TLD_GRACE_SECS`)
  memungkinkan perpanjangan, tetapi setiap operasi manajemen selain
  perpanjangan dibekukan — terutama `assign`: owner yang
  kedaluwarsa kehilangan hak manajemen sampai memperpanjang.
  Domain yang sudah diberikan di bawahnya menjalani hidupnya
  sendiri (`valid_until`-nya sendiri) — TLD-nya yang membeku,
  bukan anak-anaknya.
- **Mati — GC**: masa + masa tenggang berlalu tanpa perpanjangan →
  TLD hilang (kursi PoS dilepaskan, dapat diklaim ulang oleh siapa
  pun dengan PoW segar). `TransferTld` membawa sisa masa (tanpa
  reset).

### TLD terbuka (lahir terbuka)

- **Lahir — PoW dikurangi** (sepertiga lebih murah dari privat:
  sumber daya bersama yang tidak digover siapa pun). **Tanpa kursi
  PoS**.
- **Lantai — 1 tahun** (`OPEN_TLD_FLOOR_SECS`): saat lahir (atau
  saat dibuka via `SetTldOpen`), lantai 1 tahun mencegah
  penciptaan-penghancuran instan TLD tanpa domain.
- **Hidup**: tidak ada yang menandatangani apa pun — domain
  mendaftar mandiri. Setiap domain hidup di bawahnya mendorong
  mundur jam kematian (mengikuti kedaluwarsa domain terbaru,
  dibatasi oleh lantai).
- **Mati — GC** saat tidak ada domain hidup di bawahnya lagi dan
  lantai/kedaluwarsa telah berlalu; nama dapat diklaim lagi.

### Domain di bawah TLD terbuka

- **Lahir — PoW** (kurva dikurangi vs TLD: mainnet 1 karakter ≈
  8,5 hari, lantai ≈ setengah hari).
- **Perpanjangan — PoW identik dengan kelahiran** (kurva kesulitan
  yang sama; tantangan terikat pada kedaluwarsa saat ini, sehingga
  berbeda dari registrasi dan setiap perpanjangan sebelumnya):
  setiap masa hidup dibayar dengan kerja, domain tidak bisa
  "dipanen" gratis (≤ 3 tahun di muka).
- **Mati**: kedaluwarsa → masa tenggang 30 hari (dapat
  diperpanjang) → GC → nama dapat diklaim oleh **siapa pun**
  dengan PoW segar.

### Domain di bawah TLD privat

- **Lahir — PoS owner TLD**: hanya owner (atau operator
  yang didelegasikan) yang memberi (`assign`) — tanda tangan adalah
  otoritasnya, tanpa PoW.
- **Perpanjangan — PoS owner TLD**: ditandatangani oleh owner TLD,
  field `proof` kosong.
- **Mati**: masa tenggang lalu GC; klaim ulang kembali melalui
  `assign` owner.

### Invarian lintas bagian

- Setiap PoW di-salt per **epoch**, dengan jendela validitas
  **mundur** proporsional kesulitan (penambangan berpekan-minggu
  tetap mendarat meski epoch berganti).
- Ekuivokasi penandatangan PoS di-**slash** (larangan seumur
  hidup — M9).
- Setiap operasi memverifikasi ulang segalanya saat apply: tidak
  ada yang diambil atas kepercayaan (hash, bukti dan tanda tangan
  dihitung ulang).

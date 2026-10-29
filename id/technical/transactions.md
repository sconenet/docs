# Transaksi bertanda tangan (format v1)

Dokumen referensi **normatif** untuk format transaksi bertanda
tangan. Ia menspesifikasikan field, urutan wire eksak, ukuran,
payload bertanda tangan dan aturan validasi, agar implementasi
independen dimungkinkan tanpa membaca Rust. Implementasi
referensinya adalah [`scone-protocol`](../../../crates/scone-protocol)
(enkode) dan [`scone-blockchain`](../../../crates/scone-blockchain)
(validasi).

`PROTOCOL_VERSION` = **1**. Hanya ada satu himpunan format: byte
versi selain `0x01` (dalam transaksi) atau berbeda dari versi
lokal (dalam blok) ditolak dengan `UnsupportedVersion`.

## Ringkasan

```text
Transaction = disc u8 || version u8 || payload
```

| Field | Ukuran | Peran |
|---|---|---|
| `disc` | 1 B | tipe transaksi (tabel di bawah) |
| `version` | 1 B | **tepat `0x01`** (versi format tx) |

**Transaksi tidak lagi membawa field `network` di wire**
(perubahan 2026-09): `network_id` (`str` ≤ 16, ASCII
`[a-z0-9-]` — mis. `scone-testnet`, 13 byte) hidup **hanya dalam
payload penandatanganan** — ia disuntikkan di sana sebagai field
pertama oleh penandatangan (lihat [Payload bertanda
tangan](#payload-bertanda-tangan)) — dan tetap per-jaringan dalam
digest PoW. Transaksi yang **ditandatangani** untuk satu jaringan
karena itu tidak pernah terverifikasi di jaringan lain:
tandatangannya gagal (`InvalidSignature`) terhadap payload yang
dihitung ulang oleh chain, yang menyuntikkan jaringannya sendiri
(lihat `/docs/en/technical/blockchain.md` — jaringan dan genesis).

### Diskriminan (tabel normatif)

Setiap diskriminan adalah **konstanta opaque arbitrer yang
tetap**: nilainya sengaja dipilih berbeda, agar wire **tidak
pernah** mengesankan urutan logis antar tipe (tanpa urutan
0x01/0x02/0x03…). `0x00` **tidak pernah** tipe yang valid (byte
korupsi tipikal). Tipe mendatang akan mengambil nilai bebas yang
belum terpakai, didokumentasikan sekali untuk selamanya dalam
tabel ini.

| Diskriminan | Tipe | Peran |
|---|---|---|
| `0x21` | `RegisterDomain` | klaim sebuah domain |
| `0x52` | `UpdateDomain` | versi baru data DNS sebuah domain |
| `0x93` | `RegisterTld` | klaim domain tingkat atas (M7) |
| `0x47` | `TransferTld` | transfer TLD ke owner baru (M8a; PoW transfer terikat-penghitung) |
| `0x6B` | `RevokeTld` | pelepasan TLD (M8a) |
| `0xB8` | `SetTldOpen` | pembukaan **tak dapat dibatalkan** TLD ke registrasi mandiri (satu arah, opsi owner; `open=false` adalah penolakan bertipe) |
| `0xD4` | `AssignDomain` | penugasan langsung domain oleh owner TLD (M8a) |
| `0x3C` | `RenewDomain` | perpanjangan registrasi domain (M8a; membawa nama kanonik sejak opsi owner — TLD terbuka mengikuti kedaluwarsa) |
| `0x8F` | `RenewTld` | perpanjangan masa 3 tahun TLD PRIVAT (PoW setara penciptaan) |
| `0x15` | `SlashTx` | bukti ekuivokasi anchor + banning (M9) |
| `0xE2` | `TransferDomain` | transfer domain ke owner baru (M8c; PoW transfer terikat-penghitung) |
| `0x71` | `Heartbeat` | bukti kehidupan operator (RFC proof-of-service) |
| `0xA9` | `SetTldOperator` | delegasi operasi TLD privat (RFC proof-of-service; terikat-penghitung) |
| `0x5E` | `SetDomainOperator` | delegasi operasi domain (RFC proof-of-service; terikat-penghitung) |
| `0x66` | `SetRecordDelegates` | kunci-kunci delegasi record sebuah domain — KSK/ZSK-lite: tepat kunci-kunci ini, di samping owner, boleh menandatangani `UpdateDomain` (terikat-penghitung) |

Byte versi selain `0x01` ditolak secara eksplisit
(`UnsupportedVersion(nilai yang diterima)`): format tak dikenal
atau usang tidak pernah diparse tanpa sengaja.

## SLASH_TX (0x15, M9)

Bukti on-chain bahwa sebuah anchor menandatangani dua
**checkpoint bertentangan pada epoch yang sama** (ekuivokasi).
Siapa pun bisa mengirimkannya — buktinya kriptografis dan
mandiri:

```text
offender[32] | evidence_a (CheckpointData 116 B)
  | sig_a[64] | evidence_b (CheckpointData) | sig_b[64]
  | reporter_pk[32] | reporter_signature[64]
```

- `validate()` memeriksa strukturnya: epoch sama,
  `signing_hash` berbeda, dua tanda tangan berbeda, tanda tangan
  reporter atas payload kanonik;
- aturan state (terjurnal, dapat dibatalkan dalam reorg): KEDUA
  tanda tangan terdakwa harus terverifikasi terhadap DUA hash
  penandatanganan berbeda pada epoch yang sama; terdakwa harus
  berada dalam kumpulan layak (`SlashOffenderNotInPool` jika
  tidak); lalu **banning** — kunci dikeluarkan dari kumpulan PoS
  secara persisten (blacklist dalam state);
- sanksi minimal: domain/TLD terdakwa TIDAK disita (keputusan M9
  terdokumentasi);
- checkpoint yang sama dua kali = bukan ekuivokasi (ditolak);
  epoch berbeda = ditolak; tanda tangan palsu = ditolak.

## REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `name` | `str` ≤ 253 | nama kanonik domain yang diklaim, **dibawa dalam teks polos** (lihat bawah) |
| `domain_id` | 32 byte mentah | **harus** sama dengan `DomainId(name)` (dihitung ulang, tidak pernah dipercaya) |
| `owner` | 32 byte mentah | **harus** derivasi dari `public_key` (lihat bawah) |
| `timestamp` | varint | informasi pengurutan (Unix, detik) |
| `proof` | `bytes` ≤ 256 | payload PoW registrasi (lihat [bawah](#bukti-proof-of-work-registrasi-pow)) |
| `public_key` | 32 byte mentah | kunci publik Ed25519 penandatangan (RFC 8032) |
| `signature` | **tepat 64 byte** | tanda tangan Ed25519 atas payload bertanda tangan |

### Nama dibawa oleh transaksi

`RegisterDomain` mengangkut nama kanonik dalam teks polos,
diikuti `DomainId`-nya. Konsistensi `domain_id == DomainId(name)`
diperiksa ulang saat enkode, dekode dan validasi (pertahanan
berlapis): `RegisterDomain` yang id-nya tidak cocok dengan nama
yang dibawa tidak valid. `DomainId` tetap identitas konsensus
(kunci chain dan DHT); nama yang dibawa membuat chain
menjelaskan dirinya sendiri dan mencakup seluruh klaim dalam
tanda tangan.

## REGISTER_TLD (0x93)

Klaim atas domain tingkat atas (registri TLD, M7):

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) open(0x00|0x01) public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `tld` | `str` ≤ 63 | nama kanonik TLD **dibawa dalam teks polos** (M8b: tantangan PoW diturunkan dari nama, sebagaimana `RegisterDomain` membawa miliknya) |
| `tld_id` | 32 byte mentah | identitas TLD: `BLAKE3-256("SCONE-TLD-V1" || tld)` — **harus** derivasi dari `tld` (dihitung ulang) |
| `owner` | 32 byte mentah | derivasi dari `public_key` |
| `timestamp` | varint | informasi pengurutan (Unix, detik) |
| `proof` | `bytes` ≤ 256 | payload PoW registrasi (lihat [bawah](#bukti-proof-of-work-registrasi-pow)) |
| `open` | `0x00\|0x01` | kelahiran **terbuka**: TLD mendarat langsung di `TldState::Open` (tanpa pemilik, mandiri + PoW, satu PoW pada tarif `open_tld_pow_difficulty` — lebih murah dari privat: sumber daya bersama, tanpa kursi PoS maupun eksklusivitas); kelahiran **privat**: perilaku historis (owner, kursi PoS, `tld_pow_difficulty`) |
| `public_key` | 32 byte mentah | kunci publik Ed25519 penandatangan |
| `signature` | **tepat 64 byte** | tanda tangan Ed25519 atas payload bertanda tangan |

Ruang `TldId` **terpisah** dari ruang `DomainId` (prefiks derivasi
berbeda — lihat `/docs/en/general/naming.md`): mendaftarkan TLD
`uip` (klaim atas namespace `*.uip`) tidak pernah bisa menyerobot
identitas sebuah domain, begitu pula sebaliknya. Aturan state saat
penerapan: TLD harus bebas (registri terpisah dari registri
domain).

## Keluarga M8a — format wire

Lima tipe melengkapi registri TLD dan siklus hidup domain. Format
wire normatif sejak M8a; **aturan state lengkap sejak M8b**
(lihat `/docs/en/technical/blockchain.md`): owner TLD wajib, PoW
di mana pun klaim bersifat mandiri, kedaluwarsa dan masa tenggang
di sisi domain. Tidak ada payload yang membawa field `network` di
wire (ia hidup hanya dalam payload penandatanganan).

### TRANSFER_TLD (0x47)

```text
tld_id[32] owner[32] new_owner[32] proof.v public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `tld_id` | 32 byte mentah | TLD yang ditransfer |
| `owner` | 32 byte mentah | derivasi dari `public_key` (owner saat ini / penandatangan) |
| `new_owner` | 32 byte mentah | identitas penerima (opaque: nilai apa pun valid secara sintaks; pengikatan ke kunci yang bisa membelanjakannya adalah aturan state M8b) |
| `proof` | berbatas (varint ‖ byte) | bukti transfer anti-churn yang ditambang untuk ikatan penghitung SAAT INI dari namespace (lihat [Penghitung mutasi](#penghitung-mutasi)), kesulitan datar `transfer_pow_difficulty`, salt epoch |
| `public_key` | 32 byte mentah | kunci publik Ed25519 penandatangan |
| `signature` | **tepat 64 byte** | tanda tangan Ed25519 atas payload bertanda tangan |

Tanpa timestamp atau sequence: TLD memiliki tepat satu owner pada
satu waktu; transfer pertama yang diterapkan menang (urutan chain
saja).

### REVOKE_TLD (0x6B)

```text
tld_id[32] owner[32] public_key[32] signature[64]
```

Pelepasan TLD secara sukarela (dapat diklaim ulang setelahnya
dengan `RegisterTld` segar). Ditandatangani oleh owner saat ini;
sengaja tanpa timestamp atau sequence — revoke bersifat sekali
jalan terhadap state saat ini.

### SET_TLD_OPEN (0xB8)

```text
tld_id[32] owner[32] open u8 open_proof(proof) public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `open` | 1 byte | **secara ketat `0x00` atau `0x01`** (bentuk kanonik; nilai lain → error dekode, bukan koersi) |
| `open_proof` | `proof` (varint-panjang + byte) | field **bertanda tangan**; wajib dan diverifikasi saat `open = 0x01` (PoW pembukaan, `open_tld_pow_difficulty` jaringan, tantangan `SCONE-TLD-OPEN-V1 ‖ tld_id`); kosong saat `open = 0x00` (penutupan gratis) |

TLD **tertutup** = assign-only (domain dibuat secara eksklusif
oleh owner TLD via `AssignDomain`); TLD **terbuka** = siapa pun
bisa melakukan PoW registrasi dan mengklaim domain bebas
(`RegisterDomain`). **Pembukaan** berbiaya proof of work tambahan
(lebih ringan dari klaim TLD: ~3 hari di mainnet versus ~1
pekan), dibayar sekali oleh owner dalam `SetTldOpen` — buktinya
bepergian dalam field `open_proof` bertanda tangan dan tidak bisa
diganti setelah penandatanganan.

### ASSIGN_DOMAIN (0xD4)

```text
name(str ≤ 253) domain_id[32] owner[32] assignee[32] proof.v public_key[32] signature[64]
```

Cermin dari `RegisterDomain`: nama kanonik dibawa dalam teks
polos dan `domain_id` harus sama dengan `DomainId(name)` (dihitung
ulang saat dekode, enkode dan validasi). Ditandatangani oleh
**owner TLD dari nama itu** (diperiksa saat penerapan, M8b);
`assignee` menjadi owner pertama domain (opaque, seperti
`new_owner`). Membawa bukti anti-spam datarnya sendiri
(`assign_pow_difficulty`, tantangan `SCONE-DOMAIN-ASSIGN-V1 ‖
name` — datar memang disengaja: eksklusivitas namespace sudah
dibayar saat penciptaan TLD).

### RENEW_DOMAIN (0x3C)

```text
name(str ≤ 253) domain_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `name` | `str` ≤ 253 | nama kanonik **dibawa dalam teks polos** (opsi owner: perpanjangan menunda kematian TLD terbuka di bawahnya, chain harus mengenal nama); `domain_id` harus sama dengan `DomainId(name)` (dihitung ulang) |
| `valid_until` | varint | kedaluwarsa registrasi baru (Unix, detik) |
| `proof` | berbatas | **otorisasi perpanjangan** (lihat bawah) |

Ditandatangani oleh owner domain saat ini; apakah `valid_until`
benar-benar memperpanjang kedaluwarsa (dan paling banyak satu
masa perpanjangan) adalah aturan state (M8b).

#### Otorisasi perpanjangan — PoW atau PoS menurut TLD

Biaya perpanjangan bergantung pada sifat TLD di bawahnya
(dievaluasi saat apply, atas state blok):

- **TLD terbuka**: perpanjangan berbiaya **PoW yang sama dengan
  penciptaan** — tantangan sama
  (`SCONE-DOMAIN-V1 ‖ canonical name`), kesulitan per-panjang-label
  dan per-jaringan yang sama, jendela validitas epoch yang sama.
  Domain tidak pernah "dipanen" gratis: setiap masa hidup dibayar
  dengan kerja.
- **TLD privat**: namespace adalah stake owner TLD (PoS).
  Transaksi harus **ditandatangani oleh owner TLD** (atau operator
  yang didelegasikan) — tanda tangan ADALAH otorisasinya, field
  `proof` harus kosong (20 byte nol). Owner TLD memperpanjang
  domain namespacenya di bawah aturannya sendiri.

### RENEW_TLD (0x8F)

```text
tld(str ≤ 63) tld_id[32] owner[32] valid_until.v proof(bounded) public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `tld` | `str` ≤ 63 | nama kanonik TLD; `tld_id` harus sama dengan `TldId(tld)` (dihitung ulang) |
| `valid_until` | varint | harus TEPAT `sekarang + 3 tahun` (masa tetap — diturunkan ulang saat apply, nilai lain ditolak) |
| `proof` | berbatas | PoW perpanjangan (lihat bawah) |

Ditandatangani oleh owner **privat** TLD saat ini. Aturan apply:

- berlaku hanya untuk TLD PRIVAT (TLD terbuka hidup melalui
  domainnya — penolakan bertipe `TldIsOpen`);
- valid selama masa DAN seluruh **masa tenggang 60 hari** yang
  mengikuti kedaluwarsa; setelahnya TLD mati (GC) dan
  penolakannya `TldExpiredFrozen`;
- **PoW setara penciptaan**: tantangan
  `SCONE-TLD-V1 ‖ name ‖ current_expiry(BE64)` (akhiran anti-reuse
  mengikat tiap PoW pada SATU masa), kesulitan per-panjang yang
  sama dengan klaim awal;
- perpanjangan TIDAK dibekukan selama masa tenggang — ia satu-
  satunya operasi manajemen yang masih tersedia bagi owner
  kedaluwarsa.

## Keluarga otoritas-terdelegasi — terikat-penghitung (0xE2, 0xA9, 0x5E, 0x66)

Empat transaksi mendelegasikan atau menugaskan ulang otoritas atas
sebuah namespace. Semuanya berbagi satu disiplin anti-replay: tiap
transaksi membawa atau menambang sebuah **ikatan penghitung** —
`BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ‖ id[32] ‖
generation_be64 ‖ counter_be64)` — yang dihitung atas penghitung
mutasi SAAT INI dari namespace (lihat
[Penghitung mutasi](#penghitung-mutasi)). Transaksi yang dibangun
atas generasi usang ditolak meskipun tanda tangan atau buktinya
masih valid secara kriptografis.

### TRANSFER_DOMAIN (0xE2)

```text
domain_id[32] owner[32] new_owner[32] proof.v public_key[32] signature[64]
```

Cermin `TransferTld` untuk domain: ditandatangani owner saat ini,
`new_owner` opaque. `proof` adalah PoW transfer anti-churn (datar
`transfer_pow_difficulty`, salt epoch) yang ditambang untuk
`mutations_binding(domain_id, counter)` — tantangan generasi state
yang ditransfer. Aturan penerapan: pemeriksaan owner dulu
(`NotOwner`), lalu PoW terhadap ikatan penghitung SAAT INI
(transfer yang di-replay memecahkan tantangan yang lebih lama —
penolakan bertipe), lalu penghitung bertambah. Himpunan delegasi
record adalah pemberian owner LAMA: sebuah transfer
menghapusnya. `TransferTld` (0x47) adalah bentuk TLD yang
simetris: bukti transfernya ditambang untuk
`mutations_binding(tld_id, counter)`.

### SET_TLD_OPERATOR (0xA9) / SET_DOMAIN_OPERATOR (0x5E)

```text
id[32] owner[32] operator.option state_binding[32] public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `id` | 32 byte mentah | `tld_id` (0xA9) atau `domain_id` (0x5E) |
| `owner` | 32 byte mentah | derivasi dari `public_key` (owner saat ini / penandatangan) |
| `operator.option` | `0x00` atau `0x01 ‖ key[32]` | operator yang didelegasikan, atau `0x00` untuk kembali mengoperasikan sendiri |
| `state_binding` | **tepat 32 byte mentah** | hash kanonik ikatan penghitung SAAT INI: `mutations_binding(id, generation, counter)` (generation = `valid_until` untuk TLD, `registered_at` untuk domain) — ikatan usang adalah penolakan bertipe (`StaleStateBinding`) meski tanda tangan valid |
| `public_key` | 32 byte mentah | kunci publik Ed25519 penandatangan (owner namespace) |
| `signature` | **tepat 64 byte** | tanda tangan Ed25519 atas payload bertanda tangan |

Ditandatangani oleh owner saat ini (TLD privat untuk 0xA9; domain
terdaftar untuk 0x5E). Terikat anti-sewa: satu operator yang
didelegasikan paling banyak melayani satu kursi
(`OperatorAlreadyDelegated`). Penerapan memverifikasi ikatan, lalu
menaikkan penghitung — delegasi ulang kepada operator YANG SAMA
adalah perubahan generasi, bukan no-op.

### SET_RECORD_DELEGATES (0x66)

```text
domain_id[32] owner[32] count(1) keys[32×count] state_binding[32] public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `domain_id` | 32 byte mentah | domain target |
| `owner` | 32 byte mentah | derivasi dari `public_key` — owner SAAT INI dari DOMAIN |
| `count` | 1 byte | paling banyak 3 (`MAX_RECORD_DELEGATES`) |
| `keys` | 32 × count byte mentah | kunci-kunci publik delegasi, **menaik ketat** (urutan kanonik; kunci duplikat atau tak terurut adalah error dekode) |
| `state_binding` | **tepat 32 byte mentah** | hash kanonik penghitung SAAT INI: `mutations_binding(domain_id, registered_at, mutations)` |
| `public_key` | 32 byte mentah | kunci publik Ed25519 penandatangan — owner DOMAIN |
| `signature` | **tepat 64 byte** | tanda tangan Ed25519 atas payload bertanda tangan |

KSK/ZSK-lite: himpunan yang dibawa MENGGANTIKAN penuh himpunan
sebelumnya (tanpa penggabungan) dan — di samping owner domain —
tepat kunci-kunci inilah yang boleh menandatangani `UpdateDomain`
untuk domain tersebut. Eksklusif owner: seorang delegasi tidak
pernah bisa mengelola himpunannya. Himpunan mati bersama domain
dan dihapus oleh sebuah transfer. Penerapan memverifikasi ikatan
terhadap penghitung saat ini dan menaikkannya: siklus
`∅ → {9,10} → ∅ → replay` tidak bisa menghidupkan kembali kunci
yang dicabut (lihat `security-model.md`, serangan replay).

## UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

| Field | Tipe | Batasan |
|---|---|---|
| `domain_id` | 32 byte mentah | domain target |
| `owner` | 32 byte mentah | derivasi dari `public_key` |
| `sequence` | varint | **> 0**, harus sama dengan `current + 1` saat penerapan |
| `record_hash` | 32 byte mentah | komitmen atas himpunan record DHT |
| `public_key` | 32 byte mentah | kunci publik Ed25519 penandatangan |
| `signature` | **tepat 64 byte** | tanda tangan Ed25519 atas payload bertanda tangan |

Catatan: `UpdateDomain` membawa **tanpa** `timestamp` — pengurutan
lewat `sequence`. Ia juga tidak membawa nama: domain sudah
terdaftar, id saja cukup.

#### Aturan otoritas — owner atau delegasi record

Field `owner` adalah owner on-chain dari DOMAIN (pembaruan itu
UNTUK namespace tersebut); `public_key` adalah penandatangan
SEBENARNYA — owner ATAU salah satu kunci delegasi record domain
(KSK/ZSK-lite, lihat `SetRecordDelegates`). Berbeda dari semua
transaksi bertanda tangan lain, derivasi owner↔kunci sengaja TIDAK
dipaksakan untuk varian ini (`validate_transaction` mengecualikannya):
aturan otoritas lapisan state — penandatangan ∈ {owner} ∪ delegasi,
`sequence == current + 1` — adalah satu-satunya pengikat. Tanda
tangan tetap mencakup field `owner` melalui payload penandatanganan,
sehingga seorang delegasi menandatangani "untuk" namespace tertentu.
Catatan: peta owner→kunci dipelajari atas dasar kemunculan pertama
(registrasi/penugasan), sehingga pembaruan oleh delegasi tidak
pernah menimpanya.

### Batas ketat

- `public_key`: **tepat 32 byte mentah**, tanpa prefiks panjang.
  pk yang tidak terdekode sebagai titik kurva → error
  `InvalidTransactionKey` saat dekode.
- `signature`: **tepat 64 byte mentah**, tanpa prefiks panjang,
  tanpa margin. Ukuran lain adalah error dekode keras (pemotongan
  ditolak).
- `proof`: varint-panjang + byte, dibatasi 256
  (`MAX_PROOF_LEN`), diperiksa sebelum alokasi.
- `name`: varint-panjang + byte UTF-8, dibatasi 253
  (`MAX_NAME_LEN`), divalidasi ulang oleh aturan penamaan
  `scone-core`.

## Derivasi owner

Aturan keras: **owner selalu dihitung ulang, tidak pernah
dipercaya**.

```text
PublicKeyRef = BLAKE3-256("SCONE-PUBKEY-V1" || public_key)
OwnerId      = BLAKE3-256("SCONE-OWNER-V1" || PublicKeyRef)
```

Transaksi yang `owner`-nya tidak cocok dengan derivasi
`public_key`-nya sendiri tidak valid (`InvalidOwner` di sisi
core, `OwnerKeyMismatch` di sisi chain), ditolak saat enkode,
dekode **dan** validasi chain (pertahanan berlapis). SATU
pengecualian: `UpdateDomain` — di bawah model delegasi record,
field `owner`-nya adalah owner on-chain dari DOMAIN sementara
penandatangannya bisa berupa delegasi record; aturan otoritas
lapisan state mengikat penandatangan ke namespace.

## Penghitung mutasi

Setiap namespace membawa penghitung mutasi MONOTON
(`mutations: u64`) — generasi anti-replay CAS dari operasi-
operasi yang dijaganya.

```text
mutations_binding = BLAKE3-256("SCONE-BINDING-MUTATIONS-V1" ||
                               id[32] || generation_be64 || counter_be64)
```

- **Transaksi terjaga** (memverifikasi ikatan, lalu menaikkan
  penghitung): `SetRecordDelegates`, `SetTldOperator`,
  `SetDomainOperator`, `TransferTld`, `TransferDomain`.
- **Penanda generasi**: `registered_at` untuk domain (tetap untuk
  seluruh masa hidup namespace), `valid_until` untuk TLD (secara
  struktural berbeda di setiap masa hidup — klaim ulang hanya
  terjadi setelah masa multi-tahun penuh plus masa tenggang).
  Tanpanya, namespace yang mati dan didaftarkan ulang (penghitung
  di-reset) bisa menyiapkan ulang transaksi dari masa hidup
  sebelumnya begitu penghitung mencapai nilai yang sama.
- **Verifikasi**: penerapan menurunkan ulang ikatan dari state
  SAAT INI dan membandingkannya dengan nilai yang dibawa
  (set-types) atau memverifikasi PoW terhadapnya (transfer).
  Ketidakcocokan adalah penolakan bertipe (`StaleStateBinding` /
  bukti tak terpecahkan) yang tidak mengubah apa pun.
- **Mengapa penghitung**: fungsi murni dari state saat ini
  kembali ke nilai lama ketika state secara sah berputar balik —
  transfer A→B→A, himpunan delegasi `∅ → {9,10} → ∅`. Replay akan
  menyiapkan ulang dirinya sendiri. Sebuah penghitung tidak pernah
  kembali. Penghitung dikomitmen di daun SMT namespace (V4) dan
  diekspos dalam respons name-state agar klien bisa
  fetch-then-bind tanpa kepercayaan.

## POW-TO-OWNER — empat bentuk tanpa tanda tangan (0x2A, 0x57, 0x84, 0xC3)

Empat varian TAMBAHAN di samping bentuk-bentuk bertanda tangan
(tidak pernah pengganti — D1): `REGISTER_TLD_POW` (0x2A),
`RENEW_TLD_POW` (0x57), `REGISTER_DOMAIN_POW` (0x84),
`RENEW_DOMAIN_POW` (0xC3). Tak satu pun membawa kunci
penandatangan ATAU tanda tangan: **bukti proof of work yang
ditambang adalah seluruh otorisasinya**.

### Tantangan mengikat penerima manfaat dan jaringan (D2)

```
challenge = kind_tag ‖ name ‖ pubkey_dest[32] ‖ len(network_id) ‖ network_id
```

- `kind_tag`: tag keluarga (SCONE-TLD-V1 untuk TLD,
  SCONE-DOMAIN-V1 untuk domain) — tidak ada replay
  lintas-kind;
- `pubkey_dest`: kunci PENERIMA MANFAAT — bukti yang
  diserahkan hanya dapat dikonsumsi oleh kunci itu (tidak ada
  pencurian bukti, penambangan terdelegasi yang aman);
- `network_id`: bukti devnet tidak pernah replay di testnet
  atau mainnet.

Bentuk PERPANJANGAN menambahkan sufiks anti-pakai-ulang
keluarga bertanda tangan: `‖ current_valid_until` (BE,
8 byte) — satu bukti yang ditambang membayar tepat satu masa.

### Struktur body

Cermin eksak dari body bertanda tangan, `pubkey_dest`
menggantikan kunci penandatangan, TANPA tanda tangan di ekor:

- `REGISTER_TLD_POW`: name ‖ tld_id ‖ owner ‖ timestamp.v ‖
  proof ‖ open(0x00|0x01) ‖ pubkey_dest — `owner` =
  hash(pubkey_dest), dihitung ulang, tidak pernah dipercaya
  (D4); open = kelahiran TERBUKA (tarif lebih ringan); privat
  = masa 3 tahun ketat, penerima manfaat menjadi kursi PoS;
- `REGISTER_DOMAIN_POW`: name ‖ domain_id ‖ owner ‖
  timestamp.v ‖ proof ‖ pubkey_dest — hanya TLD TERBUKA
  (privat → TldClosed: namespace privat mempertahankan jalur
  tanda tangan owner);
- `RENEW_TLD_POW` / `RENEW_DOMAIN_POW`: name ‖ id ‖ owner ‖
  valid_until.v ‖ proof ‖ pubkey_dest.

### Penjaga penerapan (D3)

- Owner on-chain JUGA HARUS penerima manfaat tantangan: sebuah
  perpanjangan TIDAK PERNAH berganti tangan — hanya register
  yang menciptakan properti baru;
- kesulitan, jendela epoch, tenggang, dan masa adalah milik
  bentuk BERTANDA TANGAN sendiri (D5: biaya masuk tidak
  berubah);
- kunci penerima manfaat dipelajari (resolusi kumpulan PoS)
  persis seperti kunci penandatangan.

### txid dan koeksistensi

Txid tetap hash kanonik dari konten; diskriminan
0x2A/0x57/0x84/0xC3 adalah penanda fork — node lama menolaknya
sebagai UnknownDiscriminant, node baru mendekode bentuk v1
tanpa perubahan. `signing_payload` N/A untuk keempatnya
(error bertipe `PowNotSignable`): tidak pernah ditandatangani.

CLI: `scone domain register <name> --to <pubkey-hex>`,
`scone domain mine <name> --mine-for <pubkey>`,
`scone domain renew <name> --to`, `scone tld register/renew
--to`.

## Payload bertanda tangan

Payload bertanda tangan adalah enkode kanonik transaksi **tanpa
field signature**, berprefiks tag pemisahan domain — **dengan
`network_id` chain target yang disuntikkan di kepala**:

```text
signing_payload(tx, network) = "SCONE-TX-SIG-V1"
                              || canonical_encode(tx_without_signature, network)
```

di mana `canonical_encode(tx_without_signature, network)`
mengikuti struktur wire di atas, diskriminan dan byte versi
termasuk, berhenti sebelum `signature`, **dengan `network`
sebagai field pertama** — inilah SATU-SATUNYA tempat jaringan
muncul. Chain menghitung ulang payload dengan jaringannya SENDIRI:
transaksi yang ditandatangani untuk jaringan lain gagal verifikasi
tanda tangan (`InvalidSignature`); inilah mekanisme anti replay
lintas-chain.

```text
REGISTER_DOMAIN_no_sig = disc(0x21) version(0x01) network name domain_id[32] owner[32]
                            timestamp.v proof public_key[32]
REGISTER_TLD_no_sig    = disc(0x93) version(0x01) network tld tld_id[32] owner[32]
                            timestamp.v proof public_key[32]
UPDATE_DOMAIN_no_sig   = disc(0x52) version(0x01) network domain_id[32] owner[32]
                            sequence.v record_hash[32] public_key[32]
TRANSFER_TLD_no_sig    = disc(0x47) version(0x01) network tld_id[32] owner[32]
                            new_owner[32] proof public_key[32]
REVOKE_TLD_no_sig      = disc(0x6B) version(0x01) network tld_id[32] owner[32]
                            public_key[32]
SET_TLD_OPEN_no_sig    = disc(0xB8) version(0x01) network tld_id[32] owner[32]
                            open(0x00|0x01) open_proof(proof) public_key[32]
ASSIGN_DOMAIN_no_sig   = disc(0xD4) version(0x01) network name domain_id[32] owner[32]
                            assignee[32] proof public_key[32]
RENEW_DOMAIN_no_sig    = disc(0x3C) version(0x01) network name domain_id[32] owner[32]
                            valid_until.v proof public_key[32]
TRANSFER_DOMAIN_no_sig = disc(0xE2) version(0x01) network domain_id[32] owner[32]
                            new_owner[32] proof public_key[32]
```

Properti:

- deterministik dan murni: transaksi sama ⇒ payload sama di mana
  pun;
- prefiks ASCII `SCONE-TX-SIG-V1` (15 byte,
  heks `53434f4e452d54582d5349472d5631`) mencegah reuse tanda
  tangan di domain hash lain protokol;
- semua field kecuali `signature` tercakup — **termasuk nama yang
  dibawa `RegisterDomain`**: mengubah satu saja di antaranya
  membatalkan tanda tangan;
- `network_id` yang disuntikkan tercakup: tx belum bertanda yang
  sama ditandatangani untuk dua jaringan menghasilkan dua payload
  berbeda (maka dua tanda tangan berbeda).

## Bukti proof of work registrasi (PoW)

Field `proof` dari `RegisterDomain` dan `RegisterTld` membawa
bukti proof of work registrasi (modul `scone-core::pow`).

### Digest (salt epoch, anti pre-mining)

```text
epoch  = block_height / POW_WINDOW        (POW_WINDOW = 144 blok)
salt   = BLAKE3-256("SCONE-POW-V1-ARGON2" || network || challenge || epoch_le64)
digest = Argon2id(m, t, p = 1; password = nonce_le64, salt = salt[0..16])
```

`network` adalah `network_id` kanonik (M8b): bukti yang
ditambang untuk satu jaringan bukan bukti di jaringan lain.

`SCONE-POW-V1-ARGON2` juga mem-pin parameter Argon2id
per-jaringan (`pow_params`: mainnet m = 64 MiB, t = 2, p = 1;
testnet/devnet m = 8 KiB, t = 1, p = 1): inti Argon2id
(RFC 9106) membuat biaya per hash didominasi memori (resistensi
GPU/ASIC), dan bukti yang ditambang di bawah satu set parameter
tidak pernah terverifikasi di bawah yang lain.

`challenge` adalah masukan derivasi objek yang diklaim —
`"SCONE-TLD-V1" || tld` untuk TLD, `"SCONE-DOMAIN-V1" || name`
untuk domain: tepat byte yang di-hash untuk `TldId` /
`DomainId`, sehingga bukti tidak pernah bisa direplay lintas
namespace maupun lintas jenis objek. `epoch_le64` dan
`nonce_le64` adalah enkode 8-byte little-endian dari epoch dan
nonce.

Bukti valid ketika `digest` memiliki setidaknya `difficulty` bit
nol di depan.

#### Salt epoch — anti pre-mining

Digest tanpa salt tidak mengikat waktu maupun tinggi: siapa pun
bisa menambang bukti hari ini untuk ribuan nama, lalu
mendaftarkannya beruntun saat peluncuran mainnet (squatting) —
buktinya mandiri dan valid di blok mana pun yang datang lebih
dulu. Digest di-salt dengan **epoch blok yang MENERAPKAN
transaksi**:

- `epoch = block_height / POW_WINDOW`, `POW_WINDOW = 144` blok
  (~2,6 jam pada 65 dtk/blok): jendela tenggang bagi pengguna
  jujur (tambang, lalu kirim dalam jendela), penambangan ulang
  wajib setelahnya;
- bukti MENDEKLARASIKAN epoch-nya di wire; saat penerapan, epoch
  yang dideklarasikan harus jatuh dalam **jendela validitas**
  blok penerap — tolerasi MUNDUR saja (pengiriman terlambat
  diserap, menambang "untuk masa depan" mustahil), jendela
  berukuran `epoch_validity_window(difficulty)` =
  `1 << max(0, difficulty − 16)` epoch (jendela dilebarkan:
  kesulitan mainnet tinggi ditambang dalam hitungan
  hari-ke-pekan, lebih lama dari satu epoch; menimbun tidak
  memberi apa-apa karena jendela ≈ biaya penambangan — setiap
  bukti dari sebuah ledakan berbiaya penuh `2^difficulty`
  hash-nya DI DALAM jendela). Pada kesulitan ≤ 16 bit (testnet,
  nama panjang) aturan kesamaan-ketat tidak berubah;
- admisi mempool (best effort): tinggi blok mendatang tidak
  dikenal — relay memverifikasi terhadap epoch dari
  `tinggi saat ini + 1`; verifikasi ketat saat penerapan tetap
  otoritasnya;
- replay deterministik: bukti MEMBAWA epoch-nya dan jendela
  adalah fungsi murni dari (kesulitan, epoch blok) — blok
  historis terverifikasi ulang secara identik di node mana pun.

Perubahan merusak pra-mainnet: format kompak lama (12 byte)
ditolak tanpa kompatibilitas mundur.

### Payload wire (20 byte tetap)

```text
nonce_le64 || epoch_le64 || difficulty_le32
```

Kesulitan dan epoch bepergian bersama bukti, tetapi keduanya
**dideklarasikan produsen sehingga tidak pernah dipercaya**:
verifier membandingkan kesulitan terhadap **konstanta protokol**
untuk jenis registrasi (payload yang kesulitannya berbeda dari
konstanta — lebih rendah *atau* lebih tinggi — ditolak,
`InvalidProof`), dan epoch terhadap milik blok penerap
(`ProofEpochMismatch` saat ketidakcocokan). Nonce lalu
diverifikasi ulang pada kesulitan konstanta, digest dihitung
ulang dari nol.

### Kesulitan (kurva **berdasarkan panjang**, M8b + kalibrasi terukur)

Kesulitan BUKAN parameter datar: ia fungsi deterministik dari
**panjang label yang dibuat** (`scone-core::pow_length` — TLD
untuk `RegisterTld`, label paling kiri untuk `RegisterDomain`),
per jaringan:

```text
difficulty(len) = apex − round(0.415 × (len − 1)), di-clamp di lantai
```

| Jaringan | TLD privat (apex/lantai) | TLD lahir-terbuka / domain (apex/lantai) |
|---|---|---|
| testnet | 8 bit / 2 bit | 6 bit / 2 bit |
| mainnet | 42 bit / 30 bit | 40 bit / 30 bit |
| devnet | 2 bit / 1 bit | 1 bit / 1 bit |

TLD yang **lahir terbuka** (`RegisterTld` dengan `open = 0x01`)
membayar kurva "pembukaan" (apex − 2, seperti
`domain_difficulty`); **membuka TLD privat yang sudah ada**
(`SetTldOpen`) tetap pada kesulitan datar
(`open_tld_pow_difficulty`: 6 bit testnet / 22 bit mainnet /
2 bit devnet — tx hanya membawa `tld_id`, namanya sudah dibayar
panjangnya saat penciptaan). Benchmark mainnet (TLD privat,
~6 M hash/dtk): 1 karakter = 42 bit (≥ 7 hari), 5 karakter =
40 bit (~2 hari), 19 karakter = 35 bit (~1,6 jam), 29+ karakter
= lantai 30-bit (~3 menit). Kurvanya hidup di
`scone-core::pow_length` (konstanta `MAINNET_APEX = 42`,
`MAINNET_FLOOR = 30`, `TESTNET_APEX = 8`, `TESTNET_FLOOR = 2`,
`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1`); parameter datar
(`tld_pow_difficulty`, dsb.) tetap di
`scone-core::network::NetworkParams`. Mengubah satu nilai =
perubahan konsensus **untuk jaringan itu**. Invarian: devnet
adalah `NetworkParams` ketiga yang lengkap dan testnet tetap
identik ketat dengan konstanta pra-devnet-nya (parameter
Argon2id, kurva, jendela) — devnet TIDAK berbagi konstanta apa
pun dengan jaringan lain melampaui kodenya sendiri.

Kalibrasi terukur puncak datar (2026-10, `cargo run
--release -p scone-core --example pow_bench` — throughput
satu-utas ~7,7 M H/s pada x86-64 referensi):

| Jenis (mainnet) | kesulitan | waktu nyata rata-rata |
|---|---|---|
| `RegisterTld` | 24 bit | ~7 dtk |
| `RegisterDomain` | 20 bit | ~54 ms |

Pada ~65 dtk/blok: klaim TLD berbiaya ~0,1 blok CPU satu-inti,
domain ~1 ms blok — simbolis bagi pengguna, mahal untuk
diindustrialisasi skala besar (anti-squat). Kalibrasi ulang jika
target perangkat keras turun (ARM mobile: ~4–8× lebih lambat).

Cabang state (M8b, dirilis): `RegisterTld` SELALU membutuhkan
PoW-nya; `RegisterDomain` membutuhkannya pada TLD **terbuka**
(jalur tertutup assign-only — `AssignDomain`, ditandatangani
oleh owner TLD, dengan bukti anti-spam datarnya sendiri).

**Bukti transfer dan penugasan (anti-spam tingkat konsensus)**:
`TransferTld`, `TransferDomain` dan `AssignDomain` masing-masing
membawa bukti kerja **datar** — `transfer_pow_difficulty` untuk
dua transfer (tantangan = ikatan penghitung SAAT INI dari
namespace, [Penghitung mutasi](#penghitung-mutasi) — tag datar
`SCONE-*-XFER-V1` dipensiunkan), `assign_pow_difficulty`
untuk penugasan (tantangan `SCONE-DOMAIN-ASSIGN-V1 ‖ name`).
Salt epoch dan jendela anti-premining yang sama dengan
registrasi; diverifikasi saat admisi mempool (estimasi epoch,
best-effort) lalu saat penerapan (ketat). Tujuannya: perputaran
transfer dan penciptaan massal di bawah TLD privat membebani
kerja tingkat konsensus, bukan kebijakan. Tanpa pajak
per-panjang: untuk penugasan, eksklusivitas namespace sudah
dibayar saat TLD privat diciptakan.

**Transfer pasif**: jalur akuisisi ini (`TransferTld`,
`TransferDomain`, `AssignDomain`) ditandatangani oleh pemindah
seorang diri — penerima tidak pernah dikonsultasi. Pilihan yang
dianggap disengaja (hadiah, dead drop, warisan), tanpa utang
PoS yang bisa dipaksakan: lihat `security-model.md` (akuisisi
pasif dan kelayakan).

## Aturan validasi

Sebuah transaksi valid jika dan hanya jika:

1. **dekode ketat**: struktur dan batas di atas dihormati
   (diskriminan dikenal, versi `0x01`, pk 32 B, tanda tangan
   64 B, invarian core diperiksa ulang saat dekode);
2. **ikatan owner/pk**: `owner == OwnerId::from(public_key)`
   (dihitung ulang);
3. **konsistensi nama/id** (RegisterDomain): `domain_id ==
   DomainId(name)` (dihitung ulang);
4. **tanda tangan**: Ed25519 `verify_strict` atas `signature`
   terhadap payload bertanda tangan **yang dihitung ulang dari
   nol** dengan jaringan chain (tidak pernah diambil dari peer
   maupun dari transaksi sebagaimana diberikan) — tx yang
   ditandatangani untuk jaringan lain gagal di sini
   (`InvalidSignature`, anti replay lintas-chain);
5. **aturan state** (saat penerapan dalam blok — M8b, lihat
   `/docs/en/technical/blockchain.md`):
   RegisterTld → TLD bebas + bukan ICANN + PoW (kesulitan
   berdasarkan panjang); RegisterDomain → TLD terdaftar **dan
   terbuka** + domain bebas (masa tenggang termasuk) + PoW
   (berdasarkan panjang);
   UpdateDomain → domain ada, `owner` saat ini,
   `sequence == current + 1`;
   TransferTld / RevokeTld / SetTldOpen / AssignDomain → TLD ada
   **dan privat**, penandatangan = owner TLD;
   TransferDomain → domain ada, owner;
   RenewDomain → domain ada, perpanjangan ketat ≤ 3 tahun,
   otorisasi PoW (TLD terbuka: PoW sama dengan penciptaan,
   **terikat pada kedaluwarsa saat ini** — tantangan berbeda dari
   registrasi dan setiap perpanjangan sebelumnya) atau PoS
   (TLD privat: tanda tangan owner TLD — atau operator
   efektifnya — `proof` kosong).

Verifikasi memakai `verify_strict` (penolakan tanda tangan
malleable dan kunci lemah/non-kanonik), sebagaimana disyaratkan
untuk data tidak dipercaya.

## Urutan operasi dalam `push_block`

Validasi kriptografis (2 sampai 4) berlangsung untuk setiap
transaksi **sebelum** penerapan state, pada state kerja atomik
per blok; error apa pun meninggalkan chain tanpa perubahan yang
ketat.

## Perakitan blok

`BlockBuilder` (`scone-blockchain`) merakit blok dari antrean
transaksi: `tx_root` dihitung ulang atas daftar terurut, header
dirantai ke hash induk (`prev_hash`), `version` =
`PROTOCOL_VERSION`. Payload konsensus (payload produsen bertanda
tangan) dan timestamp disediakan pemanggil; trait `Consensus` yang
ada dipakai simetris dengan validasi.

## Contoh heks (vektor nyata, dapat direproduksi — M8b)

### RegisterDomain `example.uip`, timestamp 42, testnet

Payload bertanda tangan (vektor terpin; secara historis dihasilkan
oleh perkakas offline yang dihapus — nilainya tetap vektor yang
valid). `network` muncul di sini karena ini PAYLOAD
PENANDATANGANAN (satu-satunya tempatnya); transaksi wire yang
ditandatangani sendiri tidak memuatnya:

```text
53434f4e452d54582d5349472d5631       prefiks SCONE-TX-SIG-V1
21                                    disc REGISTER_DOMAIN (0x21)
01                                    versi format
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
0b 6578616d706c652e756970             name (str, panjang 11)
e6a2cbe264f81ed7da4d901c4deec6a3579224b8080fad01cb7461b8c264f181  domain_id
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner (kunci build tetap)
2a                                    timestamp (varint, 42)
14 0200000000000000 0000000000000000 04000000
                                      proof (20 B: nonce 2, epoch 0, difficulty 4 — testnet)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk + awal sig
```

### RegisterTld `uip`, timestamp 42

Payload bertanda tangan (vektor terpin) — `TldId` adalah vektor
derivasi yang di-pin di `/docs/en/general/naming.md`:

```text
53434f4e452d54582d5349472d5631       prefiks SCONE-TX-SIG-V1
93 01                                 disc REGISTER_TLD + versi
0d 73636f6e652d746573746e6574         network (str, "scone-testnet")
03 756970                             tld (str, "uip")
ad5a86d68643d5c22d6a959bb1a315530c77dfda241f1ac1a8107450f3fab25e  tld_id ("uip")
7b2446dadddf640244ac8677a401f148b2b23d1c836d5ad3dd340f8ef2edbed8   owner
2a                                    timestamp (varint, 42)
14 8e00000000000000 0000000000000000 07000000
                                      proof (20 B: nonce 0x8e, epoch 0, difficulty 7 — testnet, label 3 karakter)
00                                    open (0x00 = kelahiran privat)
2a002152f8d19b791d24453242e15f2eab6cb7cffa7b6a5ed30097960e069881db12  pk
```

(Nilai lengkap dulu dapat direproduksi via perkakas offline yang
dihapus; vektor di atas tetap ter-pin dan valid. PoW ditambang
otomatis pada kesulitan jaringan — nonce bervariasi antar
eksekusi; hanya network/name/id/owner/pk yang stabil. Sejak
perubahan wire 2026-09, heks transaksi bertanda tangan TIDAK LAGI
memuat byte
`0d 73636f6e652d746573746e6574`: jaringan bepergian hanya dalam
payload yang akan ditandatangani.)

## Di mana heks hidup sekarang

Perintah offline `tx build|sign|verify` dihapus
(cli-simplify-v1). Format heks-bertanda-tangan sendiri tidak
berubah dan tetap format wire dari **alur web/API**: aplikasi web
registrar membangun, menandatangani (keystore dalam-browser) dan
mengirim transaksi sebagai heks kanonik melalui API. Di sisi CLI,
jalur pengguna adalah bentuk satu-perintah:

```bash
scone domain register example.uip        # menambang, menandatangani, mengirim
scone domain renew example.uip                      # bukti perpanjangan, menandatangani, mengirim
scone domains                            # apa yang Anda miliki
```

Di semua jalur kode, owner dan kunci **dihitung ulang** dari
identitas penandatangan, tidak pernah diambil dari input mana
pun.

## Riwayat versi

| Versi | Konten | Status |
|---|---|---|
| 1 | `public_key` + `signature` tertanam, payload `SCONE-TX-SIG-V1`, byte versi `0x01`, `RegisterDomain` membawa nama, `RegisterTld` (0x93), keluarga M8a: `TransferTld` (0x47), `RevokeTld` (0x6B), `SetTldOpen` (0xB8), `AssignDomain` (0xD4), `RenewDomain` (0x3C) — format wire normatif; M8b: field `network` ditandatangani di kepala tiap payload, `RegisterTld` membawa nama, aturan state lengkap, kesulitan PoW per-jaringan | digantikan |
| 1 (2026-09, wire tanpa network) | **penghapusan field `network` dari wire semua transaksi**: network_id hidup hanya dalam payload penandatanganan (`signing_payload(tx, network)` menyuntikkannya di kepala); pemisahan jaringan bersandar pada tanda tangan (`InvalidSignature` pada payload asing, error `WrongNetwork` hilang); `TX_FORMAT_VERSION` tetap `0x01` (testnet relaunchable, tanpa koeksistensi); **semua TxId berubah** (TxId adalah hash enkode wire bertanda tangan, yang kehilangan 14–15 byte); digest PoW, genesis dan data-dir per-jaringan tak berubah | digantikan |
| 1 (2026-10, salt epoch PoW) | **digest PoW anti pre-mining**: salt `BLAKE3("SCONE-POW-V1-ARGON2" ‖ network ‖ challenge ‖ epoch_le)` + inti Argon2id atas `nonce_le`, dengan `epoch = block_height / 144` dari blok yang MENERAPKAN tx; payload wire bukti 12 → **20 byte** (`nonce_le ‖ epoch_le ‖ difficulty_le`), epoch yang dideklarasikan diverifikasi terhadap jendela validitas epoch blok penerap (penolakan bertipe `ProofEpochMismatch`), best-effort `(height+1)/144` saat admisi mempool, flag CLI `--epoch` pada jalur offline; perubahan merusak pra-mainnet: payload 12-byte lama ditolak, **semua TxId berubah** (field `proof` membesar 8 byte) | digantikan |
| 1 (2026-10, penghitung mutasi) | **ikatan anti-replay CAS**: penghitung mutasi monoton per-namespace (`DomainState`/`TldState::Private`), lima transaksi terjaga verify-then-increment (`SetRecordDelegates` 0x66 BARU, `SetTldOperator`, `SetDomainOperator`, `TransferTld`, `TransferDomain`); PoW transfer ditambang untuk ikatan penghitung (tantangan statis `SCONE-TLD-XFER-V1`/`SCONE-DOMAIN-XFER-V1` dipensiunkan); set-types membawa field wire `state_binding` 32 byte bertanda tangan; daun SMT V4 mengomitmenkan penghitung (V2/V3 dipensiunkan — V3 tidak pernah dirilis); kodek storage +8 byte per domain / +8 per TLD privat; `NameState` mengekspos penghitung dan `registered_at`; perubahan merusak pra-mainnet: format state + wire berubah, **reset testnet**, snapshot lama ditolak saat import | kini |

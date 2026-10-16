# Protokol

Dokumen referensi untuk protokol Scone: format biner, enkode
kanonik, transaksi, record DNS, blok, pesan P2P, batasan dan
kompatibilitas versi. Implementasi referensinya adalah crate
[`scone-protocol`](../../../crates/scone-protocol).

Butir yang ditandai **(provisional)** belum final.

## Versi protokol

- konstanta: `PROTOCOL_VERSION: u32 = 1` (`scone-protocol`);
- dibawa oleh `Hello.version` (handshake) dan `BlockHeader.version`;
- aturan kompatibilitas:
  - versi yang diterima **lebih tinggi** dari versi lokal → tolak
    (`ProtocolError::UnsupportedVersion`); node tidak menebak
    format versi yang tidak dikenalnya;
  - versi `0` → tidak valid;
  - hanya ada satu himpunan format: byte versi transaksi
    (`0x01`) dan versi blok harus cocok dengan nilai lokal
    secara **eksak**;
- tanpa pemecahan mayor/minor pada tahap ini: satu integer,
  dinaikkan pada setiap pemutusan format wire.

## Prinsip format biner

- biner, tidak pernah JSON untuk data protokol;
- deterministik dan kanonik: dua objek yang setara secara logis
  menghasilkan **byte yang persis sama**;
- parsing sekuensial, tanpa alokasi bila mungkin (peminjaman byte);
- semua masukan jaringan tidak dipercaya: panjang dan penghitung
  diperiksa **sebelum** alokasi apa pun; dekode tidak pernah
  panic.

### Endianness

- semua integer adalah varint LEB128 (grup 7-bit, least
  significant first): endianness native tidak berlaku;
- alamat IPv4/IPv6 dan identifier 32-byte: array byte mentah,
  tanpa prefiks panjang;
- tidak ada integer lebar berukuran tetap dalam format: satu
  aturan enkode integer saja.

### Varint (LEB128 minimal)

Setiap byte membawa 7 bit data; bit tinggi bernilai 1 pada setiap
byte kecuali yang terakhir. `u64` muat dalam 1 sampai 10 byte.

| nilai | byte (heks) |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 255 | `ff 01` |
| 16.384 | `80 80 01` |
| 2^32 | `80 80 80 80 10` |
| u64::MAX | `ff ff ff ff ff ff ff ff ff 01` |

Enkode harus **minimal**: bentuk terlalu besar ditolak saat dekode
(`InvalidVarint`). Ini syarat perlu agar sebuah nilai hanya
memiliki satu urutan byte yang valid (enkode kanonik). Penolakan:
`80 00` (non-minimal), `ff×9 02` (overflow u64), `ff×10`
(terlalu panjang).

### String dan byte

- `bytes`: varint panjang + byte mentah;
- `str`: sama, dengan dekode UTF-8 wajib;
- setiap panjang dibatasi (lihat [Batasan](#batasan)) dan
  diperiksa sebelum membaca.

### Identifier

| Tipe | Representasi wire |
|---|---|
| `DomainId` | 32 byte mentah |
| `TldId` | 32 byte mentah |
| `OwnerId` | 32 byte mentah |
| `PublicKeyRef` | 32 byte mentah |
| `RecordHash` | 32 byte mentah |
| `BlockHash` | 32 byte mentah |
| `MerkleRoot` | 32 byte mentah |
| `DomainName` | `str` (≤ 253 byte, divalidasi ulang oleh `scone-core`) |

Derivasi (`BLAKE3-256` + prefiks `SCONE-DOMAIN-V1`, dsb.) tetap
didefinisikan oleh `scone-core` (lihat
`/docs/en/general/naming.md`).

## Transaksi

```text
Transaction = disc u8 || version u8 || payload
```

| Diskriminan | Tipe |
|---|---|
| `0x21` | `RegisterDomain` |
| `0x52` | `UpdateDomain` |
| `0x93` | `RegisterTld` |

Setiap diskriminan adalah konstanta opaque arbitrer (tanpa urutan
logis; `0x00` tidak pernah valid) — tabel normatif lengkap di
`/docs/en/technical/transactions.md`.

Byte `version` (nilai saat ini `0x01`) mengikuti diskriminan:
nilai lain ditolak secara eksplisit
(`UnsupportedVersion(value)`). **Semua transaksi
bertanda tangan** (Ed25519): spesifikasi lengkap, payload
bertanda tangan dan aturan validasi di
**`/docs/en/technical/transactions.md`** (normatif).

Sejak perubahan wire 2026-09, **transaksi tidak lagi membawa field
`network`**: `network_id` (`str` ≤ 16) disuntikkan **hanya ke
dalam payload penandatanganan** — tx yang ditandatangani untuk
satu jaringan tidak pernah terverifikasi di jaringan lain (lihat
`/docs/en/technical/blockchain.md`).

### REGISTER_DOMAIN (0x21)

```text
name(str ≤ 253) domain_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

`RegisterDomain` membawa **nama kanonik dalam teks polos**,
diikuti `DomainId`-nya — konsistensi `domain_id == DomainId(name)`
diperiksa ulang saat dekode (dihitung ulang, tidak pernah
dipercaya). `proof` membawa PoW registrasi (20 byte: nonce + epoch
+ difficulty — salt epoch `epoch = height/144` dari blok yang
diterapkan, diverifikasi SECARA KETAT saat penerapan terhadap
konstanta jaringan dan epoch blok; penolakan bertipe
`ProofEpochMismatch` saat ketidakcocokan — M8b, lihat
`transactions.md`). `public_key` adalah kunci Ed25519 penandatangan
(tepat 32 byte); `owner` harus derivasi BLAKE3 dari kunci itu
(dihitung ulang, tidak pernah dipercaya); `signature` tepat 64
byte.

### REGISTER_TLD (0x93)

```text
tld(str ≤ 63) tld_id[32] owner[32] timestamp.v proof(bytes ≤ 256) public_key[32] signature[64]
```

Klaim atas TLD (registri terpisah dari registri domain, M7): TLD
**dibawa dalam teks polos** seperti nama domain (M8b — tantangan
PoW diturunkan dari nama), dan `tld_id` adalah derivasi
`SCONE-TLD-V1` dari nama itu, terpisah dari setiap `DomainId`
secara konstruksi. Aturan state (M8b): TLD bebas + tidak
ICANN-reserved (`IcannTldReserved` — TLD root ICANN tidak bisa
diklaim siapa pun) + PoW pada kesulitan label **berdasarkan
panjang** (kurva `pow_length` — lihat `blockchain.md`); yang
diklaim tertutup (assign-only).

### UPDATE_DOMAIN (0x52)

```text
domain_id[32] owner[32] sequence.v record_hash[32] public_key[32] signature[64]
```

Invarian diperiksa ulang saat dekode: `sequence > 0` dan `owner`
== derivasi dari `public_key`. Tanpa `timestamp`: pengurutan
lewat `sequence`. Transaksi hanya membawa referensi ringkas —
konten DNS penuh tetap di DHT (lihat bawah).

## Record DNS (sisi DHT)

Record DNS penuh tidak pernah masuk blockchain: ia
dipertukarkan via DHT (pesan `GetRecord`/`Record`) dan
dikomitmenkan on-chain hanya lewat `RecordHash`-nya.

### RecordData

```text
RecordData = type.v || ttl.v || payload(type)
```

Kode tipe adalah **kode DNS yang sebenarnya** (IANA). TTL
(detik, varint) mengikuti kode tipe dan mencakup seluruh record:
batas `MIN_TTL = 1` .. `MAX_TTL = 2.592.000` (30 hari; default
`DEFAULT_TTL = 86.400`).

| Kode | Tipe | Payload |
|---|---|---|
| 1 | A | 4 byte |
| 2 | NS | `name` |
| 5 | CNAME | `name` |
| 12 | PTR | `name` |
| 15 | MX | `preference.v`, `name` |
| 16 | TXT | `str` ≤ 4096 |
| 28 | AAAA | 16 byte |
| 33 | SRV | `priority.v`, `weight.v`, `port.v`, `target(name)` |
| 257 | CAA | `flags u8`, `tag` (1–15 B), `value` (`str` ≤ 4096) |
| 32768 | GEOROUTE | GeoRoute (lihat bawah) |
| lainnya | Unknown | code.v + `bytes` ≤ 4096 |

Contoh: `A 192.0.2.1` (TTL 86.400) → `01 80 a3 05 c0 00 02 01`
(tipe, TTL varint, alamat).

`Unknown` yang membawa kode yang sudah dikenal ditolak saat
enkode (dekode tidak pernah menghasilkannya).

#### GEOROUTE (GeoDNS)

Tipe privat Scone (rentang non-IANA). Satu aturan routing
geografis:

```text
GeoRoute  = scope || count.v(≤ 32) || endpoint…
scope     = 0x00                    -- dunia
          | 0x01 continent.v        -- UN M49
          | 0x02 country.v          -- ISO 3166-1 numerik
          | 0x03 region.v           -- kode region
          | 0x04 asn.v              -- nomor AS
          | 0x05 prefix.addr[16] || prefix.len u8   -- prefiks IP (dinormalisasi)
endpoint  = target || priority.v || weight.v
target    = 0x00 octets[4]          -- IPv4 biner
          | 0x01 octets[16]         -- IPv6 biner
```

**Aturan kanonik**: endpoint dari sebuah scope tertentu dienkode
**terurut** (priority menaik, lalu weight menurun, lalu target
menaik); pengurutan diterapkan saat enkode dan wire tak terurut
ditolak saat dekode (`NonCanonical`), seperti himpunan record.
Duplikat endpoint lengkap ditolak.

**Validasi**: `priority > 0`, `weight ≤ 100` (0 = endpoint tidak
aktif), ≤ 32 endpoint per scope (`GEO_MAX_ENDPOINTS`). Kode
negara/benua yang tidak dikenal tabel lokal
(`continent_of_country`) **diterima** di level wire: tabel
resolver bisa lebih baru daripada milik penerbit. TTL adalah TTL
`RecordData` induknya (tak berubah).

### DnsRecord

```text
domain_id[32] sequence.v expiration.v count.v(≤ 256) records…
```

**Aturan kanonik**: himpunan record adalah koleksi tanpa urutan.
Ia dienkode **terurut menurut urutan leksikografis enkode
individual tiap record**, meningkat ketat (duplikat ditolak).
Wire tak terurut ditolak saat dekode (`NonCanonical`). Urutan
`Vec` asli tidak diangkut.

### SignedDnsRecord

```text
DnsRecord owner[32] signature(bytes ≤ 128)
```

### RecordHash

```text
RecordHash = BLAKE3-256("SCONE-RECORD-V1" || canonical(DnsRecord))
```

Hash mencakup **konten record saja**; tanda tangan owner
diverifikasi terpisah atas byte kanonik yang sama. *(Keputusan:
komitmen tidak mencakup tanda tangan — versi awal dokumen ini
menyebut "SignedDnsRecord lengkap"; mengikat hash ke tanda
tangan tidak membawa apa-apa dan mengikat komitmen pada skema
tanda tangan.)*

## Blok

### BlockHeader

```text
version.v height.v prev_hash[32] tx_root[32] timestamp.v consensus(bytes ≤ 65,536)
```

- `version`: versi protokol (aturan di atas);
- `height`: tinggi (genesis = 0);
- `prev_hash`: hash blok sebelumnya (nol untuk blok genesis);
- `tx_root`: komitmen atas daftar transaksi (pohon Merkle yang
  didefinisikan `scone-blockchain`, lihat
  `/docs/en/technical/blockchain.md`);
- `timestamp`: informasi pengurutan (Unix, detik);
- `consensus`: payload produsen (`SCONE-BLOCK-V1 ‖ pk[32] ‖
  sig[64]`, lihat `blockchain.md`) — genesis sebuah jaringan
  membawa `network_id` mentah, dan genesis alokasi (kickstarter
  mainnet) boleh membawa manifest `ALLOC-V1` (≤ 60 KiB);
  dibatasi `MAX_CONSENSUS_LEN` (64 KiB).

### Block

```text
BlockHeader tx_count.v(≤ 4096) transactions…
```

Parsing sekuensial; penghitung dibatasi **sebelum** alokasi;
setiap pemotongan ditolak. Berbeda dengan himpunan record, urutan
transaksi **signifikan** (urutan konsensus) dan dipertahankan
secara eksak.

`BlockHash` (hash header kanonik) dan `tx_root` dihitung oleh
`scone-blockchain` atas enkode yang didefinisikan di sini (rumus
eksak di `/docs/en/technical/blockchain.md`).

## Pesan P2P

```text
Message = disc u8 || payload
```

Transport (framing, libp2p…) di luar lingkup; frame transport
sebaiknya tidak melebihi `MAX_MESSAGE_LEN` (1 MiB).

| Disc | Pesan | Arah | Payload | Batas | Balasan yang diharapkan |
|---|---|---|---|---|---|
| `0x01` | `Hello` | dua arah | `version.v` | — | `Hello` |
| `0x02` | `Ping` | dua arah | — | — | `Pong` |
| `0x03` | `Pong` | dua arah | — | — | — |
| `0x04` | `GetBlock` | dua arah | `hash[32]` | — | `Block` |
| `0x05` | `GetBlocks` | dua arah | `start.v` `max.v` | `max ≤ 128` | ≤ `max` × `Block` |
| `0x06` | `Block` | dua arah | `Block` | tx ≤ 4096 | — |
| `0x07` | `Transaction` | dua arah | `Transaction` | — | — |
| `0x08` | `GetRecord` | dua arah | `domain_id[32]` | — | `Record` |
| `0x09` | `Record` | dua arah | `SignedDnsRecord` | — | — |
| `0x0a` | `Checkpoint` | dua arah | `Checkpoint` | penandatangan ≤ 64 | — |
| `0x0b` | `GetCheckpoints` | dua arah | — | — | jendela checkpoint |
| `0x0c` | `GetStatus` | dua arah | — | — | `NodeStatus` |
| `0x0d` | `NodeStatus` | dua arah | state node (jaringan, tip, tinggi, NAT, domain, mempool, peer, finalitas, operator segar) | — | — |
| `0x0e` | `LookupName` | dua arah | `name(str)` | — | `NameState` |
| `0x0f` | `NameState` | dua arah | state on-chain nama + TLD | — | — |
| `0x10` | `GetOwnedNames` | dua arah | `owner[32]` | — | `OwnedNames` |
| `0x11` | `OwnedNames` | dua arah | domain + TLD hidup milik owner | — | — |
| `0x12` | `Error` | balasan | pesan error level aplikasi (`str`) | — | — |

### Checkpoint (M5)

Checkpoint teragregasi naik ke wire sebagai: tag `SCONE-CKPT-V1`
|| body `CheckpointData` (116 B, LE — identik dengan akhir
preimage penandatanganan: `epoch(8) height(8) block_hash(32)
prev_checkpoint_hash(32) state_root(32) recovery(4)`) ||
`count.v` || `pk[32] ‖ sig[64]` per penandatangan, **terurut
menaik per pk, tanpa duplikat**, `count ≤ 64`
(`MAX_CHECKPOINT_SIGNERS`). Dekode ketat: pemotongan, tag salah,
pk non-kanonik, duplikat atau tak terurut = penolakan (`None`),
tidak pernah panic (fuzz korupsi teruji).

`GetCheckpoints` meminta jendela finalitas; balasannya adalah
jendela `checkpoint_window()` (paling banyak `CHECKPOINT_KEEP`
checkpoint, terlama ke terbaru) via satu atau lebih pesan
`Checkpoint`.

`TxId` kini didefinisikan oleh `scone-blockchain`
(`BLAKE3-256("SCONE-TX-V1" || canonical(Transaction))`, lihat
`/docs/en/technical/blockchain.md`). Mempool dan anti-replay
berjendela (`TxReplay`, `REPLAY_WINDOW_BLOCKS = 256`) sudah
dirilis di relay (M4); hanya `GetTransaction` yang masih
ditunda (definisinya bergantung pada keputusan konsensus yang
tersisa).

## Pembagian Blockchain / DHT

```text
BLOCKCHAIN                      DHT
──────────                      ───
domain_id                       domain_id
owner                           SignedDnsRecord (lengkap, bertanda tangan)
sequence
record_hash
```

Chain mengatakan *siapa* yang memiliki dan *apa state saat ini*
(sequence + record_hash). DHT menyediakan *kontennya*. Record DHT
diterima jika dan hanya jika:

1. tanda tangannya valid untuk owner yang terdaftar di chain;
2. `sequence`-nya konsisten dengan (atau lebih tinggi dari)
   milik chain;
3. `RecordHash` yang dihitung ulang atas enkode kanoniknya cocok
   dengan `record_hash` chain.

## Batasan

| Konstanta | Nilai | Peran |
|---|---|---|
| `MAX_NAME_LEN` | 253 | panjang sebuah nama |
| `MAX_TLD_LEN` | 63 | panjang sebuah TLD |
| `MAX_TXT_LEN` | 4096 | string TXT |
| `MAX_PROOF_LEN` | 256 | bukti registrasi |
| `MAX_SIGNATURE_LEN` | 128 | tanda tangan (Ed25519 = 64) |
| `MAX_RECORDS_PER_SET` | 256 | record per himpunan |
| `MAX_DHT_RECORD_LEN` | 64 KiB | record DNS bertanda tangan terenkode (payload DHT: wire, penyimpanan, gossip) — batas total diperiksa saat enkode DAN sebelum dekode apa pun; 15–30× di atas himpunan realistis mana pun (SPF+DKIM+DMARC ≈ 2–5 KiB), 16× di bawah tutup transport. |
| `GEO_MAX_ENDPOINTS` | 32 | endpoint per scope GeoRoute |
| `MAX_UNKNOWN_DATA` | 4096 | data tipe tak dikenal |
| `MAX_TXS_PER_BLOCK` | 4096 | transaksi per blok |
| `MAX_CONSENSUS_LEN` | 64 KiB | payload konsensus header (dilebarkan untuk manifest genesis `ALLOC-V1` — kickstarter mainnet) |
| `MAX_BLOCKS_PER_REQUEST` | 128 | blok per `GetBlocks` |
| `MAX_MESSAGE_LEN` | 1 MiB | frame transport (disarankan) |

Setiap kelebihan → `LimitExceeded`, terdeteksi sebelum alokasi:
peer berbahaya tidak bisa memicu alokasi yang tidak sebanding
saat dekode.

## Enkode kanonik

Aturan fundamental:

```text
objek kanonik → enkode biner kanonik → hash / tanda tangan
```

- hanya varint minimal;
- himpunan record terurut meningkat ketat;
- objek yang setara secara logis memiliki tepat **satu** enkode
  valid;
- dekode menolak bentuk non-kanonik (`NonCanonical`), sehingga
  tidak ada beberapa enkode valid untuk objek yang sama.

## Penanganan error

`ProtocolError` (`scone-protocol::error`):

| Varian | Arti |
|---|---|
| `Truncated` | input berakhir di tengah nilai |
| `InvalidVarint` | varint non-minimal, overflow atau terlalu panjang |
| `IntegerOutOfRange` | integer tidak muat tipe targetnya |
| `UnsupportedVersion` | versi nol atau lebih tinggi dari lokal |
| `UnknownDiscriminant` | diskriminan transaksi/pesan tak dikenal |
| `LimitExceeded` | panjang atau jumlah melampaui batas |
| `InvalidUtf8` | string non-UTF-8 |
| `TrailingBytes` | byte tersisa setelah nilai lengkap |
| `Validation` | pelanggaran invarian `scone-core` |
| `NonCanonical` | enkode valid-per-field tetapi non-kanonik |

Dekode tidak pernah panic atas data jaringan.

## Tanda tangan

- skema: **Ed25519** (RFC 8032) via `scone-crypto`
  (`ed25519-dalek`, verifikasi `verify_strict` — tanda tangan
  malleable dan kunci lemah ditolak);
- **transaksi chain**: setiap `RegisterDomain` / `RegisterTld` /
  `UpdateDomain` menanam `public_key` (32 B) + `signature` (64 B)
  atas payload
  `"SCONE-TX-SIG-V1" || canonical_encode(tx_without_signature)`;
  spesifikasi normatif lengkap:
  `/docs/en/technical/transactions.md`;
- **record DHT** (`SignedDnsRecord`): payload bertanda tangan
  adalah enkode kanonik `DnsRecord`; verifikasi menyilangkan
  `record + owner + signature` dengan identitas yang terdaftar di
  blockchain;
- aturan lintas bagian: `owner` dan semua hash/komitmen **dihitung
  ulang**, tidak pernah diambil dari nilai yang diberikan.

## Aturan konsensus

Sudah dirilis (dijelaskan di
`/docs/en/technical/blockchain.md`): produsen blok bertanda
tangan (payload `SCONE-BLOCK-V1`, komite terpilih checkpoint),
finalitas via checkpoint teragregasi (`SCONE-CKPT-V1`, fork
choice di lantai finalitas), PoW registrasi menurut panjang nama.
Topik tersisa: biaya anti-spam terperinci, rotasi kunci owner.

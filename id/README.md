# Dokumentasi Scone

> Semua dokumentasi di bawah `docs/` bersifat **NORMATIF**: harus memungkinkan
> implementasi independen tanpa membaca Rust. Setiap perbedaan
> dokumentasi/kode adalah bug (identifikasi sumber kebenarannya,
> lalu perbaiki).

Dua bagian:

- **`general/`** — cara kerjanya: konsep dan aturan
  protokol, dapat diakses tanpa membaca detail format;
- **`technical/`** — spesifikasi terperinci untuk pengembang: format
  biner eksak, rumus hash, batasan, API, dan perilaku
  setiap komponen.

## `general/` — cara kerjanya

| Dokumen | Konten |
|---|---|
| [architecture.md](general/architecture.md) | Pemisahan otoritas (blockchain) / ketersediaan (DHT) / penyimpanan lokal; peran tiap crate; alur resolusi DNS; evolusibilitas |
| [naming.md](general/naming.md) | Aturan penamaan: alfabet LDH, TLD, domain dan subdomain, kanonisasi, derivasi `DomainId` dan representasi heksadesimal |

## `technical/` — spesifikasi untuk pengembang

| Dokumen | Konten |
|---|---|
| [protocol.md](technical/protocol.md) | Format biner wire kanonik: varint LEB128, identifier, record DNS, blok, pesan P2P, batasan, versioning |
| [transactions.md](technical/transactions.md) | Format transaksi bertanda tangan v1: field, urutan wire (RegisterDomain membawa nama, diskriminan arbitrer), derivasi owner, payload bertanda tangan `SCONE-TX-SIG-V1`, aturan validasi |
| [blockchain.md](technical/blockchain.md) | Lapisan blockchain: `TxId`, pohon Merkle, `BlockHash`, genesis, state kanonik, validasi blok, abstraksi konsensus |
| [keystore.md](technical/keystore.md) | Format keyfile `.sconekey` (Argon2id + XChaCha20-Poly1305), API `scone-keystore`, perintah `identity` |
| [storage.md](technical/storage.md) | Persistensi lokal redb: skema tabel, delta atomik, kebijakan memori, model kepercayaan, batasan DoS |
| [relay.md](technical/relay.md) | Relay jaringan (`scone-network`): protokol libp2p, penemuan peer (seed, direktori, mDNS, UPnP), jangkauan (AutoNAT, circuit-relay, DCUtR), mempool, sinkronisasi, DHT Kademlia, topologi lokal, metrik Prometheus, snapshot DNS, CLI lengkap, konfigurasi `.env` |
| [dns.md](technical/dns.md) | Server DNS UDP + TCP (M6/M7b): snapshot DNS otoritatif, cache hot-path ter-shard 16 arah (entri bersama, pencarian tanpa alokasi, record terpadat, kompresi nama), kodek RFC 1035, TCP RFC 7766 (anggaran 512-byte UDP / 64 KiB TCP, bit TC), cache negatif DHT, fallback rekursif, batasan |
| [cli.md](technical/cli.md) | Biner `scone`: logging dan verbositas (stdout/stderr, level `-v`, `RUST_LOG`), konfigurasi lingkungan via `.env` (prioritas flag > `.env` > default) |
| [security-model.md](technical/security-model.md) | Model keamanan: pemilihan komite, anti-grinding seed (analisis: VRF tidak diperlukan), ekuivokasi dan slashing |
| [audit-methodologie.md](technical/audit-methodologie.md) | Metodologi audit keamanan: perbedaan, sumber kebenaran, putaran koreksi |
| [rfc-proof-of-service.md](technical/rfc-proof-of-service.md) | RFC proof-of-service: heartbeat operator, delegasi operasi, pemilihan komite di antara kursi segar |

## Konvensi rujukan silang

Dokumen internal dan komentar kode Rust merujuk halaman-halaman ini
berdasarkan path absolut dari akar repositori
(mis. `/docs/en/technical/protocol.md`).

# Arsitektur

## Pertama, dalam kata-kata sederhana

Lima konsep cukup untuk memahami segala hal lainnya:

1. **Sebuah nama** — alamat di internet, seperti `mysite.scone`.
2. **Frasa 24 kata Anda** — identitas Anda. Itu adalah sertifikat yang
   membuktikan sebuah nama milik Anda, dan menandatangani setiap
   perubahan yang Anda buat.
3. **Buku catatan publik** (blockchain) — buku besar bersama dan
   publik tentang siapa memiliki apa. Tidak ada yang
   mengendalikannya, tidak ada yang bisa menulis ulang.
4. **Mining** — cara Anda memesan sebuah nama: komputer Anda
   melakukan sejumlah pekerjaan, adil dan setara bagi semua orang.
   Tanpa antrean, tanpa perantara.
5. **Sebuah relay** — komputer yang menjalankan jaringan dengan
   menyimpan salinan buku dan menjawab pertanyaan. Siapa pun bisa
   menjalankannya; semakin banyak, semakin kuat jaringannya.

Sisa halaman ini membahas lebih dalam — Anda boleh berhenti di sini
dan sudah memahami proyek ini.

## Prinsip panduan

Scone memisahkan otoritas dari ketersediaan:

- **blockchain** adalah sumber kebenaran atas kepemilikan dan urutan perubahan;
- **DHT** adalah sistem distribusi/replikasi, ia **bukan** otoritas;
- **penyimpanan lokal** menampung state node, indeks, dan cache.

Sebuah node harus memperlakukan semua data yang datang dari DHT
sebagai berpotensi berbahaya: data hanya diterima setelah verifikasi
kriptografis (tanda tangan owner, kecocokan dengan ownership /
sequence / record_hash pada chain).

## Pemisahan tanggung jawab

```text
BLOCKCHAIN
──────────
Ownership
Registrasi
Otorisasi
Pengurutan
Sequence
Record hash
Informasi konsensus

DHT
───
Record DNS
Record bertanda tangan
Replikasi
Ketersediaan

PENYIMPANAN LOKAL
─────────────────
State lokal
Indeks
Record DHT ter-cache
State blockchain
```

## Crate dan dependensi

```text
scone-crypto     ← lapisan terendah: primitif murni atas byte
    ▲
scone-core       ← tipe protokol (bergantung pada scone-crypto)
    ▲
scone-protocol  ← format biner wire/blockchain
    ▲
scone-blockchain← logika blockchain, state kanonik (di memori)
    ▲               └ scone-topology (engine murni, tanpa jaringan/fs/
                     async; diberi makan oleh scone-network) — scone-genesis
                     (biner alat genesis mainnet sekali jalan)
scone-storage   ← persistensi lokal: backend redb + snapshot terverifikasi
    ▲
scone-network   ← relay p2p: libp2p/QUIC, mempool, sinkronisasi, DHT, DNS
    ▲
scone-keystore  ← keyfile `.sconekey` terenkripsi (bergantung pada scone-crypto)
    ▲
scone           ← biner / aplikasi
```

### Keputusan: `scone-core` bergantung pada `scone-crypto`

`DomainId` harus dapat dihitung dari fondasi
(`BLAKE3-256(prefix || canonical name)`), tetapi kriptografi konkret
tidak boleh mencemari core. Resolusinya:

- `scone-crypto` adalah lapisan terendah: fungsi deterministik murni
  atas byte (plus RNG OS untuk pembuatan kunci), tanpa tipe core,
  tanpa dependensi jaringan/filesystem/async → tidak mungkin terjadi
  siklus;
- `scone-core` menggunakannya hanya untuk hashing;
- Tanda tangan dan kunci Ed25519 berada di `scone-crypto` dan bekerja
  atas byte (payload kanonik yang dihasilkan oleh `scone-protocol`).

### Peran tiap crate

**scone-core** — nama (`TldName`, `DomainName`), identifier
(`DomainId`, `OwnerId`, `PublicKeyRef`), data DNS (`DnsRecord`,
`SignedDnsRecord`), transaksi (`RegisterDomain`, `UpdateDomain`).
Deterministik dan portabel. Dilarang dalam crate ini: libp2p, redb,
tokio/async, jaringan, berkas, OS, server DNS, logika blockchain.

**scone-crypto** — primitif kriptografis. Hari ini: `hash256`
(BLAKE3), tanda tangan dan kunci Ed25519 (`SigningKey`,
`PublicKey`, `Signature`, verifikasi batch ketat), amplop AEAD
grup record (XChaCha20-Poly1305). Direncanakan: pohon Merkle.
Aturan: tidak ada algoritma buatan sendiri, hanya dependensi yang
telah diaudit.

**scone-protocol** — format biner protokol: enkode/dekode kanonik
(varint LEB128 minimal, identifier 32-byte polos) atas transaksi,
record DNS, blok (`Block`/`BlockHeader` dengan field konsensus
opaque) dan pesan P2P, dengan batasan ukuran yang diperiksa
sebelum alokasi, penolakan bentuk non-kanonik, dan versioning
(`PROTOCOL_VERSION`). Tanpa logika konsensus, jaringan, atau
penyimpanan. Referensi format:
`/docs/en/technical/protocol.md`.

**scone-blockchain** — logika blockchain dan state kanonik:
identifikasi transaksi (`TxId`), pohon Merkle terurut,
`BlockHash`, genesis deterministik, validasi blok dan transaksi,
aturan penerapan state (`DomainState`/`ChainState`, di memori,
akses langsung per `DomainId`), deteksi fork dan pemilihan cabang
(`fork_choice`: lantai finalitas + rantai terpanjang + tie-break
hash terendah), finalitas checkpoint, dan abstraksi konsensus
(`trait Consensus`, implementasi permisif secara default). PoW
registrasi (mine/verify, kesulitan per jaringan) berada di
`scone-core`. Dilarang dalam crate ini: jaringan, DHT, penyimpanan
(redb), server DNS. Referensi:
`/docs/en/technical/blockchain.md`.

**scone-storage** — persistensi lokal (sudah dirilis). Backend konkret
adalah `redb` (`NodeStore`), disembunyikan di balik trait minimal agar
blockchain/DHT/DNS tidak pernah terkopel langsung ke redb; backend
lain (KV terdistribusi) bisa dipasang. Termasuk backend state KV
berhalaman, log blok, snapshot terverifikasi (manifest ter-hash
+ halaman) dan jalur pemulihan/replay saat restart. Referensi:
`/docs/en/technical/storage.md`.

**scone-network** — relay p2p node: swarm libp2p/QUIC, relay
transaksi dan blok (validasi lokal sistematis), mempool
berbatas, sinkronisasi blok berbatas (128 blok/permintaan), DHT
Kademlia untuk `SignedDnsRecord`, server DNS UDP+TCP (GeoDNS selalu
aktif — tabel GeoIP tertanam, fallback rekursif), loop
checkpoint/heartbeat anchor (CLI berbicara p2p QUIC langsung ke
relay, tanpa RPC lokal lagi). Jangkauan dan penemuan: AutoNAT
v2 (vonis reachable/NATed), circuit-relay v2 + DCUtR untuk node
ter-NAT, mDNS (penemuan LAN tanpa konfigurasi, AKTIF secara default
di devnet) dan UPnP (pemetaan port QUIC oleh router, opt-in).
Konfigurasi via berkas `.env` (lihat `.env.example`).
Referensi: `/docs/en/technical/relay.md`.

**scone-keystore** — penyimpanan terenkripsi kunci tanda tangan Ed25519
(berkas `.sconekey`: Argon2id + XChaCha20-Poly1305). Crate ini
memiliki filesystem agar `scone-crypto` bisa tetap murni. Referensi:
`/docs/en/technical/keystore.md`.

**scone-topology** — engine MURNI untuk optimasi topologi p2p lokal:
tanpa jaringan, tanpa filesystem, tanpa async. Ia tidak mengenal
libp2p maupun swarm: `scone-network` memberinya observasi (RTT ping,
koneksi) dan menerima keputusan pengelolaan lingkungan sekitar.

**scone-genesis** — biner alat sekali jalan (kickstarter) untuk
persiapan genesis mainnet: manifest JSON berversi, alokasi,
blok deterministik.

**scone** — biner/CLI. Bergantung pada sebuah crate hanya jika
benar-benar menggunakannya (hari ini: scone-network untuk relay,
scone-blockchain, scone-storage, scone-keystore, scone-protocol,
scone-core, scone-crypto, scone-topology).

## Node

Sebuah node Scone menggabungkan:

1. blockchain (validasi konsensus, state kepemilikan);
2. routing DHT (publikasi/pengambilan `SignedDnsRecord`);
3. penyimpanan lokal (state, indeks, cache);
4. resolver / server DNS di lini depan.

### Memulai node baru (bootstrap state, P0.3/P0.4)

Node baru TIDAK me-replay genesis: ia mulai dari state ter-finalisasi
yang terverifikasi.

```text
checkpoint ter-finalisasi (komite, kuorum tanda tangan — divalidasi oleh
        │              lapisan chain sebelum digunakan)
        ▼
snapshot terverifikasi (manifest + halaman berhalaman; importer
        │              MENGHITUNG ULANG akar SMT dari entri
        │              dan membandingkannya dengan state_root yang
        │              ditandatangani — snapshot palsu/tidak lengkap
        │              ditolak sebelum penulisan apa pun)
        ▼
blok anchor H (diminta dari jaringan BERDASARKAN HASH)
        │
        ▼
blok terbaru H+1..H+k (relay, tinggi meningkat)
        │
        ▼
tersinkron — saat boot, replay HANYA akhiran di atas H
```

Dua profil node:

- **node arsip** (flag operator): menyimpan semua blok historis,
  melayaninya ke jaringan, mulai lewat jalur yang sudah ada;
- **node full-state**: memegang state penuh pada H dan akhiran,
  tetapi bukan riwayat di bawah H — ia memvalidasi blok mendatang dan
  melayani state, ia tidak melayani riwayat.

Jalur ini OPT-IN: mengimpor snapshot terverifikasi adalah tindakan
operator eksplisit (`import_verified_snapshot`), tidak pernah
otomatis — perilaku default node yang sudah ada tidak berubah.
Detail normatif: `/docs/en/technical/storage.md`, bagian
"VERIFIED state snapshot" dan "State bootstrap".

## Alur resolusi DNS

```text
Kueri DNS (example.uip)
    │
    ▼
Node: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── snapshot DNS otoritatif (RAM) ─► tidak dikenal/kedaluwarsa → NXDOMAIN
    │                                          (tanpa menyentuh core)
    ├── cache DNS ter-shard (qname,qtype) ─► hit: entri bersama,
    │                                          verifikasi malas
    ├── cache lokal (SignedDnsRecord) ────► verifikasi (tanda tangan, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ───────────────► verifikasi yang sama
    │                                          (miss → cache negatif 30 dtk)
    ▼
Jawaban DNS (jika dan hanya jika valid)
```

Snapshot DNS (eksistensi + kedaluwarsa setiap domain hidup,
dipublikasikan setelah tiap blok yang diterapkan) menjamin bahwa
keputusan eksistensi tidak pernah melewati run loop relay: core
hanya disentuh untuk pengambilan record yang sebenarnya.

## Evolusibilitas

- preimage berversi (`SCONE-DOMAIN-V1`, `SCONE-PUBKEY-V1`,
  `SCONE-OWNER-V1`, …): mengubah aturan derivasi menciptakan
  namespace baru alih-alih merusak yang sudah ada;
- `RecordData` dapat diperluas via varian + `Unknown { type_code, data }`
  untuk tipe yang belum dikenal saat pembacaan;
- `PROTOCOL_VERSION` (`scone-protocol`) untuk pemutusan format wire;
- chain hanya menyimpan hash: data besar tetap di DHT.

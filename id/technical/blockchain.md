# Blockchain

Dokumen referensi untuk lapisan blockchain: identitas, hashing,
genesis, state kanonik, pengurutan transaksi, dan validasi blok.
Implementasi referensinya adalah crate
[`scone-blockchain`](../../../crates/scone-blockchain), di atas
`scone-protocol` (enkode kanonik), `scone-core` (aturan bisnis),
dan `scone-crypto` (BLAKE3).

Lingkup saat ini: logika blockchain dengan konsensus nyata
(produsen bertanda tangan, checkpoint, finalitas), **state di
memori** di atas backend KV (penyimpanan konkret ada di
`scone-storage`) — lihat
[Ditunda ke konsensus](#ditunda-ke-konsensus).

## Identitas dan pemisahan domain hash

Empat identitas, empat peran, empat prefiks BLAKE3 berbeda —
sebuah prefiks tidak pernah dipakai ulang di dua penggunaan:

| Tipe | Peran | Prefiks | Rumus |
|---|---|---|---|
| `DomainId` | identitas nama domain | `SCONE-DOMAIN-V1` | lihat `/docs/en/general/naming.md` |
| `RecordHash` | identitas konten DNS | `SCONE-RECORD-V1` | lihat `/docs/en/technical/protocol.md` |
| `TxId` | identitas transaksi | `SCONE-TX-V1` | di bawah |
| `BlockHash` | identitas blok | `SCONE-BLOCK-V1` | di bawah |

Pohon Merkle memakai prefiksnya sendiri `SCONE-MERKLE-V1`.
Prefiks-prefiks ini di samping milik `scone-core`
(`SCONE-PUBKEY-V1`, `SCONE-OWNER-V1`).

Rantai dependensi lengkap:

```text
DomainId
    ↓
RegisterDomain / UpdateDomain
    ↓
Transaction
    ↓
TxId
    ↓
MerkleRoot
    ↓
BlockHeader
    ↓
BlockHash
    ↓
prev_hash blok berikutnya
    ↓
chain kanonik
```

## TxId

```text
TxId = BLAKE3-256("SCONE-TX-V1" || canonical_encode(Transaction))
```

- deterministik: transaksi yang sama (enkode kanonik sama) ⇒
  `TxId` sama, di semua node, selamanya;
- identitas **menurut konten**: field yang dimodifikasi apa pun
  mengubah `TxId` (domain, owner, timestamp, proof, sequence,
  record_hash);
- tidak ada timestamp lokal atau nilai non-kanonik yang masuk
  perhitungan;
- tanpa dependensi pada penyimpanan atau jaringan;
- identitas tidak menjamin keunikan penerapan: me-replay
  transaksi identik gagal pada aturan state (`RegisterDomain`
  ganda, replay sequence), bukan pada `TxId`.

API: `scone_blockchain::transaction_id(&Transaction) -> Result<TxId>`.

## Pohon Merkle (tx_root)

Komitmen atas **daftar terurut** transaksi sebuah blok:

```text
0 transaksi   root = BLAKE3-256("SCONE-MERKLE-V1" || "EMPTY")
leaf          BLAKE3-256("SCONE-MERKLE-V1" || 0x00 || TxId)
node internal BLAKE3-256("SCONE-MERKLE-V1" || 0x01 || left || right)
level ganjil  hash terakhir diduplikasi untuk melengkapi pasangan
```

Aturan:

- **tidak pernah diurutkan**: urutan transaksi dalam sebuah blok
  signifikan; dua blok berisi transaksi yang sama dalam urutan
  berbeda memiliki akar berbeda;
- tag `0x00` / `0x01` mencegah kebingungan antara leaf dan node
  internal (preimage-kedua);
- prefiks `SCONE-MERKLE-V1` mengisolasi pohon dari domain hash
  lain (sebuah `TxId` tidak bisa diinterpretasikan sebagai node);
- implementasi in-house (~30 baris), deterministik dan teruji;
  tanpa pustaka Merkle eksternal.

API: `merkle_root(&[TxId]) -> MerkleRoot` dan
`tx_root(&[Transaction]) -> Result<MerkleRoot>`.

## BlockHash

```text
BlockHash = BLAKE3-256("SCONE-BLOCK-V1" || canonical_encode(BlockHeader))
```

Hash mencakup **hanya** header kanonik: transaksi sudah
dikomitmenkan oleh `tx_root`, jadi tidak di-hash kedua kalinya:

```text
transaksi -> TxId -> MerkleRoot -> BlockHeader -> BlockHash
```

Mengubah sebuah transaksi, urutannya, tinggi, timestamp,
`prev_hash`, atau payload konsensus mengubah `BlockHash`.

API: `block_hash(&BlockHeader) -> Result<BlockHash>`.

## Jaringan dan genesis (M8b)

Scone mendefinisikan tiga jaringan terpisah, diidentifikasi oleh
`network_id` kanonik (1..=16 byte ASCII `[a-z0-9-]`):

| Jaringan | `network_id` | apex TLD privat | apex TLD terbuka | apex domain | lantai TLD | lantai domain |
|---|---|---|---|---|---|---|
| testnet | `scone-testnet` | 8 bit | 7 bit | 6 bit | 2 bit | 1 bit |
| mainnet | `scone-mainnet` | 28 bit | 27 bit | 26 bit | 25 bit | 22 bit |
| devnet | `scone-devnet` | 2 bit | 1 bit | 1 bit | 1 bit | 1 bit |

Parameter hidup di `scone-core::network::NetworkParams`
(instance bernama `TESTNET` / `MAINNET` / `DEVNET`, pencarian via
`NetworkParams::by_name`). Mengubah satu kesulitan = perubahan
konsensus **untuk jaringan itu**.

**Invarian struktural**: devnet adalah `NetworkParams` ketiga yang
lengkap — bukan subkumpulan flag testnet. Testnet tetap
**identik secara ketat** dengan keadaannya sebelum devnet
ditambahkan (konstanta Argon2id, kurva, jendela epoch yang sama);
suite test mem-pin invarian ini.

**Kesulitan bervariasi-panjang** (`scone-core::pow_length`):
kesulitan penciptaan bergantung pada panjang **label yang
dibuat** (TLD untuk `RegisterTld` — privat atau baru dibuka —,
label paling kiri untuk `RegisterDomain`). Rumusnya:

```text
TLD     : difficulty(len) = apex − round(0.287 × (len − 1)), di-clamp di lantai
domain  : difficulty(len) = apex − round(0.480 × (len − 1)), di-clamp di lantai
```

Dua kemiringan berbeda: `log₂(1.22) ≈ 0.287` bit/karakter untuk
TLD (tiap karakter ekstra membagi biaya ~1,22), dan
`log₂(1.40) ≈ 0.480` bit/karakter untuk domain — domain meluruh
lebih cepat: nilainya ada pada TLD yang membawanya. TLD
lahir-terbuka berada di apex privat − 1; domain di apex privat −
2 (mainnet: TLD privat 25, TLD terbuka 24, domain 23). Benchmark
mainnet (Argon2id memory-hard, ~11,4 H/s satu-utas): TLD privat
1 karakter = 28 bit ≈ 30 hari, 9+ karakter = lantai 25-bit ≈ 3,8
hari; TLD terbuka 1 karakter = 27 bit ≈ 15 hari; domain 1
karakter = 26 bit ≈ 3,8 hari, 9+ karakter = lantai 22-bit ≈
11,5 jam. **Mesin referensi 8-inti** (kalibrasi ulang 2026-10 —
CLI menambang multi-inti), ~11,4 H/s/inti. **Membuka** TLD privat
yang ada (`SetTldOpen`) tetap pada kesulitan datar
(`open_tld_pow_difficulty`: 6 bit testnet / 22 bit mainnet): tx
hanya membawa `tld_id`, namanya sudah membayar panjangnya saat
penciptaan. Fungsi murni (determinisme: dua node menghitung
kesulitan yang sama untuk nama yang sama).

Genesis adalah konstanta protokol **per jaringan**, dapat
direproduksi dari konstanta saja (tanpa keacakan, tanpa jam
lokal, tanpa jaringan):

| Field | Nilai |
|---|---|
| `version` | `PROTOCOL_VERSION` (`1`) |
| `height` | `0` |
| `prev_hash` | 32 byte nol (tanpa blok sebelumnya) |
| `tx_root` | akar Merkle daftar kosong |
| `timestamp` | `GENESIS_TIMESTAMP = 0` (murni struktural) |
| `consensus` | byte kanonik `network_id` |
| transaksi | kosong |

Varian **genesis alokasi** (kickstarter mainnet, M-next): field
`consensus` membawa `network_id ‖ ALLOC-V1 ‖ count.v ‖ entries`
(domain/TLD prabayar, urut kanonik per nama, manifest ≤ 60 KiB,
≤ 2048 alokasi) — format yang dilebarkan inilah yang
membenarkan `MAX_CONSENSUS_LEN` = 64 KiB (lihat
`scone-blockchain::genesis_alloc`).

`network_id` dalam payload konsensus membuat hash genesis — dan
karena itu seluruh blockchain — **terpisah antar jaringan**: blok
testnet tidak pernah bisa menempel ke chain mainnet
(`UnknownParent`), replay mustahil tanpa menandatangani ulang
segalanya.

API: `genesis_of(&NetworkParams) -> Block`,
`genesis_hash_of(NetworkId) -> BlockHash`; alias `genesis()` /
`genesis_hash()` merujuk testnet (default pengembangan).

### Devnet — jaringan lokal yang dapat di-reset

Devnet (`scone-devnet`) adalah jaringan ketiga, dikhususkan untuk
iterasi lokal cepat:

- **Genesis sendiri**: id jaringan `scone-devnet` dalam payload
  konsensus menghasilkan hash genesis berbeda dari KEDUA testnet
  dan mainnet — tidak ada genesis, blok, tanda tangan atau bukti
  PoW yang pernah melintasi batas (replay mustahil).
- **PoW mendekati nol**: kurva di-pin pada rentang 0..=2 bit
  (`DEVNET_APEX = 2`, `DEVNET_FLOOR = 1` — peluruhan panjang
  tetap berlaku): klaim TLD atau domain ditambang dalam
  milidetik, menempuh jalur kode yang persis sama dengan jaringan
  lain.
- **Konsensus PoS cepat**: parameter sama dengan testnet (komite
  4, epoch 10 dtk) untuk finalitas cepat pada mesh lokal.
- **Reset**: chain hidup di direktori datanya sendiri
  (`~/.scone/devnet/` secara default). **Menghapus direktori itu
  dan menjalankan ulang relay me-reset chain** (genesis segar,
  store perawan) — tanpa perkakas tambahan. JANGAN PERNAH
  mengarahkan data dir devnet ke data dir jaringan lain: store
  tak-kosong yang memegang chain lain ditolak (`Corrupted`).

## State kanonik

```text
DomainState  = { owner: OwnerId, sequence: u64, record_hash: Option<RecordHash>,
                 registered_at: u64, valid_until: u64 }
TldState     = enum { Private { owner: OwnerId } | Open }
               -- enum (opsi owner): invarian terbuka/privat bersifat
               -- struktural; tenggat kematian hidup di luar entri
               -- (tabel tld_floors + jadwal kematian terjurnal)
ChainState   = NetworkParams                  (jaringan terikat, M8b)
               + DomainId -> DomainState      (akses langsung, di memori)
               + TldId -> TldState            (registri TLD, M7b)
               + DomainId -> (expired_at, OwnerId)   (jendela masa tenggang, M8b)
```

* `registered_at` / `valid_until` (M8b): kedaluwarsa registrasi,
  dievaluasi terhadap **timestamp blok induk** (deterministik,
  dikomitmen dalam header — bukan jam lokal);
* `open`: TLD privat bisa **lahir terbuka** (`RegisterTld`
  dengan `open = 0x01` — tanpa pemilik, mandiri) atau
  **tertutup** saat klaim (tertutup = assign-only), lalu menjadi
  terbuka via `SetTldOpen` (registrasi mandiri dengan PoW) —
  **secara tak dapat dibatalkan**;

- akses langsung per `DomainId` (32 byte): tidak pernah `String`
  sebagai kunci, tidak pernah pemindaian penuh — syarat perlu
  untuk menargetkan ratusan miliar domain;
- `record_hash == None` selama belum ada `UpdateDomain` yang
  diterapkan;
- ruang id TLD dan domain **terpisah secara konstruksi** (prefiks
  derivasi berbeda, lihat `/docs/en/general/naming.md`):
  mendaftarkan TLD `uip` tidak pernah bisa menyerobot identitas
  sebuah domain, begitu pula sebaliknya.

### Aturan penerapan

`state.apply(transaction)` deterministik dan atomik (saat error,
state tidak berubah):

**Semua transaksi (M8b, wire 2026-09)**: transaksi tidak lagi
membawa field `network` — pemisahan bersandar pada **tanda
tangan**: payload bertanda tangan menyuntikkan `network_id`
chain (`validate_transaction(tx, network_of_the_chain)`), maka tx
yang ditandatangani untuk jaringan lain gagal verifikasi tanda
tangan (`InvalidSignature`, SEBELUM penerapan — tx bertanda
testnet tidak pernah diterapkan, tidak pernah di-pool oleh node
mainnet, dan sebaliknya).

**REGISTER_TLD**:

- TLD harus **bebas** (tidak ada di registri TLD) dan **bukan
  ICANN-reserved** (`IcannTldReserved`: TLD root ICANN milik tak
  seorang pun di Scone);
- **PoW wajib**: `pow::verify` pada tantangan
  `SCONE-TLD-V1 || tld` (diturunkan dari nama yang dibawa tx),
  pada kesulitan label **berdasarkan panjang** (`pow_length`):
  kelahiran privat → `tld_difficulty` (apex jaringan),
  kelahiran terbuka (`open = 0x01`) → `open_tld_difficulty`
  (apex − 2) — digest dihitung ulang dari nol
  (salt `SCONE-POW-V1-ARGON2 ‖ network ‖ challenge ‖ epoch`,
  inti Argon2id), kesulitan yang dideklarasikan sendiri harus
  sama dengan kesulitan yang diharapkan, dan **epoch yang
  dideklarasikan** bukti harus sama dengan `height/POW_WINDOW`
  blok penerap (salt epoch melawan pre-mining — penolakan bertipe
  `ProofEpochMismatch`, lihat `transactions.md`);
- saat penerapan: `open = 0x01` → `{ Open }` (tanpa pemilik —
  mandiri, kursi PoS dilepaskan, lantai kematian diletakkan pada
  `now + grace`);
  `open = 0x00` → `{ Private { owner } }` — **tertutup secara
  default** (assign-only); pembukaan eksplisit via `SetTldOpen`.

**REGISTER (RegisterDomain)**:

- **TLD dari nama yang dibawa harus terdaftar DAN terbuka**:
  chain menurunkan `TldId(name.tld())` sendiri (bukan nilai yang
  diberikan); absen → `UnknownTld` (D1, M7c), ada tetapi tertutup
  → `TldClosed` (M8b: jalur assign-only eksklusif);
- domain harus **bebas**: absen dari state DAN di luar jendela
  masa tenggang registrasi kedaluwarsa mana pun (lihat GC di
  bawah);
- **PoW wajib**: tantangan `SCONE-DOMAIN-V1 || name` pada
  **kesulitan berdasarkan panjang label paling kiri**
  (`pow_length::domain_difficulty` — TLD terbuka, inilah harga
  masuk mandiri);
- saat penerapan: `{ owner, sequence: 0, record_hash: None,
  registered_at: now, valid_until: now + 1 tahun }` —
  **registrasi berlangsung 1 tahun** (365 hari).

**Urutan intra-blok**: signifikan. `[RegisterTld, SetTldOpen,
RegisterDomain]` dalam blok yang sama valid; permutasi apa pun
yang menerapkan aturan sebelum prasyaratnya ada ditolak
sebagai blok keseluruhan (undo-log).

**UPDATE (UpdateDomain)**:

- domain harus ada (belum kedaluwarsa);
- `tx.owner` harus owner saat ini;
- `tx.sequence` harus **tepat** `current_sequence + 1`
  (tanpa celah, tanpa replay);
- saat penerapan: `sequence` maju, `record_hash` diganti.

**TRANSFER_TLD (0x47)**:

- TLD harus ada dan **privat** (`UnknownTld` / `TldIsOpen` jika
  tidak — TLD terbuka milik tak seorang pun);
- penandatangan harus owner saat ini (`NotTldOwner` jika tidak);
- saat penerapan: `owner = new_owner` (identitas penerima
  opaque; pengikatan ke kunci yang bisa membelanjakannya
  dilakukan tx berikutnya). Tanpa sequence: transfer pertama yang
  diterapkan menang (urutan chain).

**REVOKE_TLD (0x6B)**:

- TLD harus ada dan **privat** (TLD terbuka tidak bisa dicabut —
  `TldIsOpen`: ia mati secara alami di GC), penandatangan = owner
  saat ini;
- saat penerapan: TLD dikeluarkan dari registri dan dapat
  diklaim ulang oleh `RegisterTld` segar (PoW termasuk).

**SET_TLD_OPEN (0xB8)**:

- TLD harus ada dan **privat** (sudah terbuka → `TldIsOpen`:
  owner lama terhapus, tak ada lagi kunci sah untuk
  menandatanganinya); `open = 0x00` (menutup) → penolakan bertipe
  `TldCloseUnsupported` sedini validasi murni: **pembukaan tak
  dapat dibatalkan** (opsi owner);
- penandatangan = owner saat ini (`NotTldOwner`);
- **pembukaan = PoW wajib**: `pow::verify` pada tantangan
  `SCONE-TLD-OPEN-V1 || tld_id` pada `open_tld_pow_difficulty`
  datar jaringan (6 bit testnet / 22 bit mainnet), salt epoch
  termasuk (bukti bepergian dalam field `open_proof` tx yang
  bertanda tangan);
- saat penerapan: `TldState::Open` — **owner terhapus** (tak
  dapat dibatalkan, kursi PoS dilepaskan), lantai kematian
  diletakkan pada `now + grace`.

**ASSIGN_DOMAIN (0xD4)**:

- TLD dari nama yang dibawa harus ada dan **privat**
  (`TldIsOpen` jika tidak: TLD terbuka hanya mandiri),
  penandatangan = owner TLD (`NotTldOwner`);
- domain harus bebas (aturan masa tenggang sama dengan REGISTER);
- saat penerapan: `{ owner: assignee, sequence: 0, record_hash:
  None, registered_at: now, valid_until: now + 1 tahun }`.
  **Tanpa PoW**: owner namespacelah yang menanggung.

**TRANSFER_DOMAIN (0xE2, M8c)**:

- domain harus ada (`UnknownDomain` jika tidak), penandatangan =
  owner saat ini (`NotOwner`);
- saat penerapan: `owner = new_owner` (kursi PoS mengikuti domain
  dalam kumpulan owner, simetris dengan `TransferTld`). Tanpa
  sequence: transfer pertama yang diterapkan menang (urutan
  chain).

**RENEW_DOMAIN (0x3C)**:

- domain harus ada; **jenis TLD dievaluasi PERTAMA**
  (pengerasan M8c):
  - **TLD privat**: otorisasi **PoS** — penandatangan ADALAH
    owner TLD, atau kursi owner TLD mendelegasikan ke kunci
    publik penandatangan
    (`effective_operator(tld_owner) == renew.public_key` —
    delegasi selalu dibaca DARI owner TLD, tidak pernah
    sebaliknya). Field `proof` harus kosong (bukti tidak kosong =
    tolak).
  - **TLD terbuka**: otorisasi **PoW** — penandatangan adalah
    owner domain saat ini (`NotOwner` jika tidak) dan buktinya
    menambang tantangan perpanjangan: kesulitan sama dengan
    penciptaan (menurut panjang label), **terikat pada
    kedaluwarsa saat ini**
    (`challenge = DOMAIN_ID_VERSION ‖ canonical name ‖ current
    valid_until`) — bukti perpanjangan berbeda dari milik
    registrasi dan dari setiap perpanjangan sebelumnya
    (anti-reuse);
- `valid_until` harus **memperpanjang secara ketat** tenggat
  saat ini (`RenewalNotExtending` jika tidak);
- `valid_until ≤ now + 3 tahun` (`RenewalExceedsTerm` — batas
  anti-hoarding: **1 masa perpanjangan = 1 tahun, maksimum 3
  tahun di muka**);
- saat penerapan: `valid_until = tx.valid_until`; perpanjangan
  juga menunda lantai kematian TLD terbuka di bawahnya (tx
  membawa nama kanonik untuk itu).

**GC deterministik (M8b)**: sebelum menerapkan transaksi sebuah
blok, chain mengevaluasi kedaluwarsa pada **timestamp blok
induk**: setiap domain yang `valid_until ≤ now` dikeluarkan dari
state hidup dan diparkir dalam **jendela masa tenggang 30 hari**
(`valid_until + 30 hari`). Selama masa tenggang, hanya owner
lamanya yang bisa mengklaim ulang namanya; setelahnya, bebas
untuk semua. GC terjurnal: blok yang ditolak memulihkan state
pra-GC bit demi bit.

**Siklus hidup TLD (masa tetap)**: TLD membawa jamnya sendiri,
independen dari domain.

- **TLD privat**: klaim membuka **masa 3 tahun**
  (`TLD_TERM_SECS`). Perpanjangan (`RenewTld`, 0x8F) membayar
  **PoW setara penciptaan** dan me-reset tepat 3 tahun (masa
  tetap diturunkan ulang saat apply). Pada kedaluwarsa, **masa
  tenggang 60 hari** (`TLD_GRACE_SECS`) terbuka: perpanjangan
  tetap mungkin, tetapi **setiap operasi manajemen lain
  dibekukan** — terutama `AssignDomain` (penolakan bertipe
  `TldExpiredFrozen`): owner kedaluwarsa kehilangan hak
  manajemen sampai memperpanjang. Domain yang sudah diberikan di
  bawahnya menjalani `valid_until` mereka sendiri — TLD-nya yang
  membeku, bukan anak-anaknya. Masa tenggang berlalu tanpa
  perpanjangan: TLD mati di GC (kursi PoS dilepaskan, dapat
  diklaim ulang). `TransferTld` membawa sisa masa.
- **TLD terbuka**: lahir (atau dibuka) dengan **lantai 1 tahun**
  (`OPEN_TLD_FLOOR_SECS` — anti cipta-hancur instan). Setiap
  domain hidup di bawahnya mendorong mundur jam kematian
  (mengikuti kedaluwarsa domain terbaru). Tanpa domain + lantai
  berlalu: mati di GC, dapat diklaim ulang.
- **TLD terbuka abadi** (hanya genesis: `.scone`, `.tor` —
  penamaan terverifikasi di atas alamat onion via CNAME —,
  `.p2p`): alokasi `OpenImmortalTld` menyetel sentinel
  `until = u64::MAX` — GC tidak pernah bisa melintasinya dan
  pemasangan-ulang monotone tidak pernah bisa menurunkannya: TLD
  tidak pernah kedaluwarsa, bahkan dikosongkan dari semua
  domainnya. Tidak ada transaksi runtime yang bisa menciptakan
  yang abadi (`RegisterTld` tetap pada lantai 1 tahunannya);
  domain di bawahnya menjalani siklus normalnya.

Kematian TLD berbagi antrean kematian deterministik domain
(`tld_deaths`), terjurnal dan dapat dipulihkan bit-eksak.

**Pengecualian tinggi-1**: genesis membawa timestamp yang NOL
SECARA STRUKTURAL (`GENESIS_TIMESTAMP` = 0, konvensi protokol —
bukan jam). Waktu chain blok PERTAMA karena itu adalah
timestamp-nya sendiri (dikomitmen dalam header, deterministik),
bukan induknya: menjangkar kedaluwarsa blok 1 pada 0 akan
mem-pin lantai kematian TLD yang lahir terbuka (dan `valid_until`
domain yang lahir di blok 1) ke epoch Unix — GC pra-apply blok 2
(timestamp nyata) akan menghapus sumber daya itu sebelum
penerapan ("dropped conflicting tx from block production:
unknown TLD", regresi testnet 60).

**Replay**: replay lintas-jaringan mustahil pada tiga level —
genesis terpisah (`UnknownParent`), `network_id` disuntikkan ke
setiap payload tanda tangan (tanda tangan tidak valid di jaringan
lain — satu-satunya tempat jaringan muncul sejak wire 2026-09),
PoW terpisah per jaringan dalam digest. Replay intra-jaringan tx
identik gagal pada aturan state (klaim ganda, sequence), bukan
pada `TxId`.

Dua node jaringan yang sama mulai dari state yang sama dan
menerapkan blok yang sama dalam urutan yang sama menghasilkan
state akhir yang persis sama.

## Pengurutan transaksi

Urutan adalah properti fundamental, dikomitmen di setiap level:

```text
urutan transaksi dalam blok
    ↓
MerkleRoot
    ↓
BlockHash
    ↓
chain kanonik
```

Dua transaksi tidak dapat dipertukarkan: dalam blok yang sama,
`[REGISTER, UPDATE]` diterapkan, `[UPDATE, REGISTER]` ditolak
(domain belum ada saat `UPDATE`). Tie-break global (yang
mengurutkan mempool, yang memproduksi blok) milik konsensus.

## Validasi blok (push_block)

Semuanya **dihitung ulang**, tidak ada yang diambil atas
kepercayaan — `tx_root`, hash, tanda tangan, atau field yang
diberikan peer tidak pernah diterima apa adanya:

1. `prev_hash` == hash tip kanonik (jika tidak `UnknownParent`
   bila induk tak dikenal, `ParentNotTip` bila dikenal tetapi
   bukan tip: deteksi fork minimal);
2. `height` == tinggi tip + 1;
3. `version` == `PROTOCOL_VERSION` **secara eksak** (versi lain
   — lebih rendah atau lebih tinggi — ditolak, tidak
   diinterpretasikan ulang);
4. jumlah transaksi ≤ `MAX_TXS_PER_BLOCK` (4096);
5. `tx_root` dihitung ulang atas transaksi dalam urutan blok;
6. hook konsensus (`validate_header`, `validate_tx`);
6ter. **anti-replay TXID berjendela (port .bak)**: transaksi yang
   `TxId`-nya muncul dalam indeks inklusi dari
   `REPLAY_WINDOW_BLOCKS = 256` blok terbaru ditolak
   (`TxReplay`). Indeks (TxId → tinggi inklusi) adalah fungsi
   murni dari chain kanonik, dipangkas pada laju yang sama;
   indeks dibangun ulang saat boot oleh storage (rescan
   `min(256, height)` blok terakhir yang dipersist — M8, lihat
   `/docs/en/technical/storage.md`): node yang direstart menolak
   replay yang sama seperti node hidup. Melampaui jendela,
   re-inklusi tetap tunduk pada aturan state (klaim ganda,
   replay sequence).
6bis. **produsen bertanda tangan (M5, port .bak)**: payload
   `consensus` harus terdekode sebagai payload produsen
   (`SCONE-BLOCK-V1 ‖ pk[32] ‖ sig[64]`, berbatas); tanda
   tangan mencakup *hash tanda tangan produsen* = `BlockHash`
   header **dengan field `consensus` dikosongkan** (payload hidup
   di field itu; menandatangani hash penuh akan melingkar);
   produsen harus termasuk himpunan yang diotorisasi pada
   `timestamp` blok (komite terpilih checkpoint, plus undian
   pemulihan yang terbuka; bootstrap: owner domain hidup —
   kumpulan kosong = produksi terbuka, jika tidak registrasi
   pertama mustahil). Genesis (tinggi 0, payload kosong)
   struktural dan tidak pernah terkait. Error bertipe
   `InvalidProducer`;
7. untuk setiap transaksi, berurutan: validasi **kriptografis**
   (`validate_transaction`: ikatan owner/kunci dihitung ulang +
   `verify_strict` atas payload bertanda tangan yang dihitung
   ulang dari nol, lihat
   `/docs/en/technical/transactions.md`), lalu penerapan ke
   state kanonik di bawah **jurnal undo** (undo-log: satu entri
   per transaksi yang diterapkan, biaya O(transaksi blok), bukan
   O(domain));
8. komit atomik: semuanya lolos ⇒ blok ditambahkan, jurnal
   dibuang; sedikit error ⇒ jurnal digulung mundur (urutan
   terbalik) dan chain tidak berubah secara ketat, bit demi bit.

Tidak ada fungsi validasi yang panic; masukan salah-bentuk atau
permusuhan apa pun menghasilkan `BlockchainError` bertipe
(`OwnerKeyMismatch`, `InvalidSignature`, …).

## Perakitan blok (BlockBuilder)

`BlockBuilder` (`scone-blockchain::builder`) merakit blok dari
antrean transaksi tervalidasi:

```text
BlockBuilder::after(parent_height, prev_hash)
    .with_timestamp(t) .with_consensus(payload)
    .push_tx(tx)… .build() -> Block
```

- `tx_root` **dihitung ulang** atas daftar terurut transaksi
  antrean (tidak pernah diberikan);
- `prev_hash`/`height` dirantai ke induk (hash induk dihitung
  pemanggil via `block_hash`, tidak pernah diambil dari peer);
- `version` = `PROTOCOL_VERSION`;
- batas `MAX_TXS_PER_BLOCK` ditegakkan saat penyisipan
  (`TooManyTransactions`);
- payload `consensus` (payload produsen bertanda tangan) dan
  `timestamp` disediakan pemanggil: konsensus konkret dipasang
  via trait [`Consensus`](#abstraksi-konsensus) yang ada,
  simetris dengan validasi;
- M5: `.with_producer(&SigningKey)` menandatangani blok sebagai
  produsen — payload `consensus` menjadi payload produsen
  bertanda tangan (atas hash tanda tangan produsen yang
  didefinisikan di atas) dan menggantikan `.with_consensus`
  sebelumnya.

Blok yang dirakit lolos `push_block` tanpa enkode ulang (teruji).

## Fork dan fork choice (M3c, port .bak)

`push_block` tetap titik masuk linear (induk == tip, jika tidak
`UnknownParent`/`ParentNotTip`). `try_attach` (M3c) mengevaluasi
blok yang bersaing:

- induk tak dikenal → `UnknownParent` (blok menunggu);
- silsilah valid tetapi tertinggal → evaluasi cabang: rantai
  terpanjang menang; seri sempurna → tie-break deterministik
  (hash tip terkecil dalam urutan leksikografis — dua node yang
  memegang blok yang sama selalu memilih pemenang yang sama,
  teruji);
- **lantai finalitas**: cabang yang salah satu leluhurnya
  bertentangan dengan checkpoint terfinalisasi (termasuk dalam
  `prev_checkpoint_hash` checkpoint yang diterima) ditolak
  (error bertipe
  `Consensus("branch contradicts a finalized checkpoint")`);
- adopsi = replay dari genesis blok cabang pemenang (determinisme
  bit-eksak teruji: dua node yang menerima blok yang sama dalam
  urutan berbeda berakhir pada state yang sama);
- tidak ada blok yang diterima yang kanonik sebelum validasi
  penuh.

## Jendela RAM berbatas (M7a, port .bak)

Chain hanya menyimpan **512 blok terakhir** di RAM
(`RAM_WINDOW_BLOCKS`, port dari `HISTORY_KEEP = 512` milik .bak),
di atas **lantai finalitas** (checkpoint terakhir terfinalisasi):

- blok keluar dari RAM hanya jika **keduanya** di luar jendela
  (`height < tip − 511`) **dan** di bawah ketat lantai
  finalitas — reorg di atas lantai selalu tetap mungkin, maka
  blok-blok itu TIDAK PERNAH diusir;
- sebelum checkpoint pertama terfinalisasi (bootstrap), **tidak
  ada yang diusir**: memori berbatas dimulai dengan finalitas;
- pengusiran nyata: blok, indeks hash (`known_hashes`, juga
  berbatas, diindeks per tinggi), dan tx keluar dari memori —
  store (`NodeStore`, `scone-storage`) memegang salinan tahan
  lama;
- `block(h)` di bawah jendela → `None`; `block_result(h)` →
  error bertipe `BlockPruned { height }` — jalur reorg
  (`push_block`, `try_attach`, `adopt_branch`, replay prefiks)
  menyebarkannya: relay harus memuat ulang segmen dari storage
  atau jatuh ke sinkronisasi penuh (keterbatasan terdokumentasi:
  tanpa adopsi tersegmen sebelum port relay penuh);
- blok yang dibangun di atas induk terusir terklasifikasi
  `BlockPruned` (bukan `UnknownParent`): tinggi induk <
  `base_height` ⇒ store bisa melayaninya.

## Abstraksi konsensus

```rust
pub trait Consensus {
    fn validate_header(&self, header: &BlockHeader) -> Result<()>;
    fn validate_tx(&self, tx: &Transaction) -> Result<()>;
}
```

- `PermissiveConsensus` (default): menerima semuanya — dipakai
  sampai aturan nyata didefinisikan;
- hook dipanggil oleh `push_block`: konsensus konkret (PoW,
  kesulitan, biaya, timestamp/anti-replay, fork choice) akan
  terpasang tanpa menulis ulang chain;
- satu abstraksi, tanpa hierarki trait.

## State skalabel (backend KV, pivot SMT, antrean kedaluwarsa, kumpulan owner berbatas)

Tujuan: 250+ miliar domain potensial — JANGAN PERNAH memuat
semuanya ke RAM. Empat bagian:

### Backend KV (`state_backend.rs`)

```rust
pub trait StateBackend: Default + Send {
    fn get(&self, key: &Pkey) -> Option<Vec<u8>>;
    fn put(&mut self, key: Pkey, value: Vec<u8>);
    fn delete(&mut self, key: &Pkey);
    fn range(&self, from: Option<&Pkey>, limit: usize)
        -> (Vec<(Pkey, Vec<u8>)>, Option<Pkey>);
}
```

- `Pkey` = 32 byte mentah `DomainId`/`TldId` — SATU ruang kunci
  terurut (prefiks derivasi terpisah), nilai = enkode
  `SCONE-ENTRY-DOM-V1` (88 byte tetap) /
  `SCONE-ENTRY-TLD-V1` (variabel: 33 byte privat / 1 byte
  terbuka, enum opsi owner — flag + owner bersyarat, tidak ada
  lagi field `until`);
- `MemoryBackend` (BTreeMap) = implementasi referensi (dev,
  test); TANPA dependensi konkret dalam `scone-blockchain`
  (redb dilarang dalam crate) — protokol/konsensus bergantung
  pada TRAIT saja;
- `CachedBackend<B>` (`state_cache.rs`) = cache LRU berbatas di
  atas backend dalam: write-through (yang dalam selalu mutakhir),
  miss tidak pernah di-cache, `range` memberi halaman pada yang
  dalam — jawaban sama dengan backend dalam sendirian (kontrak
  determinisme, teruji);
- `RedbBackend` (`scone-storage::redb_backend`) = backend
  terpersist: tabel `state_kv` (kunci mentah 32-byte, urutan
  leksikografis = urutan menaik yang disyaratkan), penghitung
  dipelihara dalam tabel meta `state_kv_meta` (jangan pernah
  COUNT); `CachedBackend<RedbBackend>` = state dengan jejak RAM
  berbatas — "tingkat ringan";
- `smt_persist`: mengonsumsi himpunan kotor SMT
  (`take_dirty_sets`) dan menuliskannya ke backend di bawah
  kunci prefiks-cadangan (0x01 node, 0x02 leaf) — fondasi SMT
  berhalaman;
- trait tidak lagi mensyaratkan `Default` (basis data disk tidak
  punya nilai default yang masuk akal);
- `range` beriterasi dalam urutan kunci menaik (kontrak
  determinisme) — hanya jalur arsip, bukan aturan konsensus;
- `ChainState` mengakses domain/TLD HANYA per kunci melalui
  backend (baca titik, tulis terjurnal).

### Akar SMT kanonik (pivot)

- `state_root_smt()` (O(1), akar ter-cache, `SCONE-STATE-SMT-V1`)
  adalah state_root **kanonik**: inilah yang dikomitmenkan
  `checkpoint_data` dan diverifikasi `accept_checkpoint` di tip.
  Perhitungan TIDAK PERNAH melintasi himpunan domain;
- fold langsung lama `SCONE-STATE-V1` bertahan sebagai
  `state_root_archive()`: O(N), jalur arsip/verifikasi (fold
  ulang penuh dari storage untuk mengecek silang akar
  historis) — bukan artefak konsensus. Keduanya mengomitmenkan
  leaf yang sama: state logis setara ⇒ akar setara (untuk
  keduanya);
- checkpoint yang diterima dengan akar SMT terverifikasi ulang
  setelah pemulihan (reload): akar adalah fungsi murni dari state
  yang dipulihkan (backend + SMT + indeks).

### Antrean kedaluwarsa (GC O(kedaluwarsa), bukan O(N))

- `expirations: BTreeMap<u64, BTreeSet<DomainId>>` — instan
  kedaluwarsa → id yang kedaluwarsa saat itu (id terurut per
  slot: kesetaraan state dan urutan drain tak bergantung urutan
  kedatangan);
- `gc_expired_journaled(now)` menguras `range(..=now)`
  alih-alih memindai seluruh state: biaya O(yang kedaluwarsa
  pada instan itu);
- batas RAM: satu entri per domain HIDUP (kedaluwarsa
  tertunda saja — perpanjangan mengubah kunci slot, GC
  mengurasnya), tidak pernah satu node per domain dalam struktur
  pemindaian;
- dipelihara oleh jurnal undo (rollback bit-eksak: slot yang
  dipulihkan dikembalikan).

### Kumpulan owner berbatas (kunci, bukan domain)

- `owner_refcounts: HashMap<OwnerId, u64>` — satu penghitung per
  owner hidup BERBEDA (domain + TLD), dipelihara jurnal pada
  setiap register/transfer/revoke/assign/GC (akuisisi/lepas);
- `eligible_validators(now)` membaca indeks ini — O(owner),
  TIDAK PERNAH iterasi atas domain. Kebaruan temporal sudah
  terjamin: GC deterministik (timestamp induk) telah menghapus
  setiap domain kedaluwarsa DAN refcount-nya sebelum panggilan.
  Keluaran terurut (deterministik);
- batas RAM: O(owner berbeda), independen dari jumlah domain;
- `owner_pool_root` (fold arsip) melipat owner terurut ini —
  juga tanpa pemindaian domain.

Anggaran RAM tunak (backend disk/jaringan):
O(owner berbeda + kedaluwarsa tertunda + jendela masa tenggang
berjalan + jurnal undo) — semuanya berbatas independen dari
jumlah domain, semuanya dipulihkan bit-eksak oleh jurnal.

## Kursi dan komite finalitas (mekanisme PoS eksak)

Bagian ini menggambarkan mekanisme kelayakan dan pemilihan komite
yang EKSAK sebagaimana diimplementasikan (`finality.rs`) —
pelengkap operasionalnya (loop anchor, wire) ada di
`/docs/en/technical/relay.md`, analisis keamanan di
`/docs/en/technical/security-model.md`.

### Kursi = kepemilikan (tanpa registrasi)

**Kursi** PoS BUKAN registrasi: ia kepemilikan domain atau TLD
hidup. `eligible_validators(now)` menghitung kumpulannya:

- kunci owner hidup via indeks `owner_refcounts` berbatas (satu
  owner BERBEDA per domain/TLD hidup — seratus domain di bawah
  satu kunci = SATU kursi: anti-Sybil ekonomisnya per
  identitas, linier);
- TLD dihitung (klaimnya berbiaya PoW — owner TLD hidup adalah
  pemangku kepentingan), domain hidup juga
  (`valid_until > now` — dijamin oleh GC deterministik, sudah
  teruras sebelum panggilan);
- **dikurangi kunci ter-ban** (M9: ekuivokator yang di-slash keluar
  dari kumpulan seumur hidup);
- TLD **terbuka** (`owner: None`) TIDAK memegang kursi: owner
  lamanya dilepaskan saat pembukaan (`owner_release` dalam apply
  `SetTldOpen`), dan ia mati di GC tanpa pernah memilih lagi.

Keluaran terurut (urutan deterministik, tak bergantung urutan
iterasi indeks).

Konsekuensi struktural: **kematian kursi = kematian
domain/TLD**. Kedaluwarsa + GC, `RevokeTld` (kursi dilepaskan),
`TransferTld`/`TransferDomain` (kursi mengikuti domain ke owner
baru), pembukaan `SetTldOpen` (kursi dilepaskan) — setiap
transisi kepemilikan memindahkan atau melepaskan kursi dalam
kumpulan, simetris dengan jurnal `owner_refcounts`
(akuisisi/lepas).

### Kumpulan DIBEKUKAN pada checkpoint terakhir terfinalisasi (`FinalizedBase.pool`)

Saat menerima checkpoint (`accept_checkpoint`), basis finalitas
menangkap kumpulan dari state PADA BLOK TERFINALISASI:

```text
FinalizedBase { checkpoint, pool, block_ts, seed }
```

- `pool` = `eligible_validators(timestamp blok terfinalisasi)`
  — finalitas TIDAK PERNAH membaca ulang state hidup
  setelahnya: registrasi apa pun (klaim, transfer, ban)
  setelah checkpoint hanya berlaku untuk epoch berikutnya;
- `seed` = `next_seed(signing_hash(checkpoint))` — seed epoch
  BERIKUTNYA diturunkan dari hash **konten** checkpoint
  (berkunci, himpunan tanda tangan dikecualikan — dua view yang
  memegang subkumpulan kuorum berbeda dari checkpoint yang sama
  memilih komite yang sama).

### Undian komite (bootstrap, kesegaran, degradasi)

`committee(recovery)`:

1. **Bootstrap** (tanpa checkpoint terfinalisasi): SELURUH
   kumpulan menandatangani — komite yang diundi dari view lokal
   akan menyimpang antar node yang tertinggal (akar masalah
   pembekuan finalitas); seluruh kumpulan deterministik per
   chain. Seleksi ketat resume pada finalitas pertama.
2. **Rezim terfinalisasi**: undian dari subkumpulan **segar**
   dari kumpulan beku (RFC proof-of-service §5.2) — tiap kursi
   (kunci owner) diselesaikan ke operator EFEKTIFNYA oleh state
   di tip (delegasi `SetTldOperator`/`SetDomainOperator` = fakta
   on-chain baru, atau kunci owner saat mandiri); kursi ikut
   BERKUNCI EFEKTIFNYA (itulah yang menandatangani), asal punya
   **heartbeat segar** (jendela geser). Kunci ter-ban tetap
   dikecualikan.
3. **Degradasi aman**: kurang dari
   `MIN_FINALITY_COMMITTEE_SIZE` (4) operator segar → undian
   dari SELURUH kumpulan (bootstrap dingin, pemadaman operator
   umum — finalitas tidak pernah hilang menurut konstruksi).

`allowed_producers(block_ts)` (produksi blok) mengikuti skema
yang sama: seluruh kumpulan saat bootstrap; komite terpilih +
undian pemulihan (undian k terbuka setelah k × `epoch_secs`
waktu konsensus) pada rezim terfinalisasi.

Kuorum checkpoint adalah `quorum_for(ukuran komite)`
(floor(2n/3)+1); komite di bawah `MIN_FINALITY_COMMITTEE_SIZE`
tidak pernah finalisasi (penjaga BFT).

## Ditunda ke konsensus

Sengaja tidak didefinisikan dalam crate ini:

- **mempool**: admisi/pengurutan ulang terperinci (biaya,
  penggantian) tetap di relay (M4 membawa batas per-domain +
  anti-replay TXID berjendela — `REPLAY_WINDOW_BLOCKS = 256`);
- **pengurutan global**: sort mempool adalah sort deterministik
  lokal (relay), bukan otoritas global;
- **otoritas timestamp** melampaui GC: kedaluwarsa (M8b)
  memakai timestamp blok induk; aturan waktu lain (anti-replay
  terperinci) tetap di konsensus;
- penyimpanan terdistribusi: state di memori; backend
  (`scone-storage`, redb) menerapkan aturan ini tanpa
  menduplikasinya.

Terkirim sejak M8b (dulu ditunda): PoW registrasi per-jaringan
(`RegisterTld` selalu, `RegisterDomain` pada TLD terbuka),
kedaluwarsa/perpanjangan 1 tahun + masa tenggang 30 hari +
horizon 3 tahun, GC deterministik pada timestamp blok induk,
replay lintas-jaringan mustahil (genesis + field bertanda +
digest PoW).

## Simulasi terdistribusi deterministik (P1.6)

`crates/scone-blockchain/src/sim/` memuat harness simulasi
jaringan **deterministik** (`SimNet`): N `Blockchain` nyata
(tanpa mock) yang digerakkan **jam virtual** dan **antrean
peristiwa** berprioritas `(tick, urutan penyisipan)`. Tanpa
socket, tanpa async, tanpa jam dinding — setiap undian (kehilangan,
latensi, duplikasi) adalah fungsi murni dari `(seed, nomor
kirim)`, maka skenario dengan seed tetap dapat direproduksi byte
demi byte.

### Model

- **Transport**: setiap emisi mengalami kehilangan probabilistik
  (dapat dikonfigurasi dalam ‰), latensi artifisial berbatas
  (`latency_ticks`), kemungkinan duplikasi (satu salinan
  tertunda), dan peta partisi saat ini (pesan antar-grup
  hilang). Duplikasi tidak pernah memajukan jam logis dan tidak
  memicu kirim baru: aliran peristiwa nyata identik dengan atau
  tanpa duplikasi (kontrak determinisme, dibuktikan test
  `duplicate_and_delay`).
- **Gossip epidemi**: node mengirim ulang ke peer yang
  terhubung (partisi sama) sebuah blok/transaksi/checkpoint
  hanya saat pertama kali terlihat — aturan relay H2
  (memotong loop). Blok yang diparkir (orphan) TIDAK PERNAH
  disiarkan ulang; celah penerimaan diisi oleh jalur
  sinkronisasi (`GetBlocks`), seperti relay.
- **Produksi**: satu langkah produksi global per interval; tiap
  grup partisi memproduksi paling banyak SATU blok per putaran,
  oleh node terotorisasinya yang memegang tinggi terbaik. Saat
  bootstrap (sebelum finalitas pertama) produsen di-pin ke node
  terendah: himpunan produsen terotorisasi diturunkan dari state
  (kumpulan domain hidup), dua view yang sedikit berbeda memilih
  komite berbeda dan blok yang sah secara lokal akan menjadi
  `InvalidProducer` di tempat lain — divergensi permanen.
  Setelah finalitas pertama komite dibekukan dalam basis
  finalitas (identik di node konvergen mana pun) dan produksi
  bebas.
- **Finalitas**: pada setiap kelipatan tinggi `sign_interval`,
  anchor yang ditunjuk komite menandatangani konten deterministik
  tipnya via `SignerGuard` tahan-crash nyata (satu kunci = satu
  suara per epoch, state di-fsync sebelum difusi); anggota komite
  lain menandatangani saat diterima (setelah memverifikasi bahwa
  blok yang dirujuk memang blok kanonik mereka pada tinggi itu —
  penjaga level relay) dan menyiarkan ulang agregat yang
  diperkaya. Pada kuorum, `accept_checkpoint` memfinalisasi.
  Tiap node berkomitmen pada proposal pertama yang terlihat per
  epoch (penjaga anti-finalitas-ganda bootstrap: dua proposal
  yang bersaing pada epoch yang sama dengan penandatangan
  terpisah tidak bisa keduanya terakumulasi).
- **Crash/restart**: node yang crash kehilangan seluruh state
  RAM-nya (chain, mempool, parking, view) tetapi mempertahankan
  direktori signer-guard-nya di disk, persis seperti proses
  nyata; ia resinkronkan dengan me-replay blok kanonik yang
  dilayani peer-nya (`GetBlocks`/`GetCheckpoints`).

### Skenario tercakup (`sim::tests`)

| Test | Apa yang dibuktikan |
|---|---|
| `convergence_simple` | 8 node, kehilangan 5 ‰, latensi 1–5 tick, 500 blok: tip sama, `state_root_smt` sama di mana-mana; finalitas aktif; tidak pernah dua checkpoint tak kompatibel per epoch. |
| `partition_then_heal` | Partisi 4a/4b selama ~300 blok: kedua sisi memproduksi, penyembuhan → konvergensi dengan reorg cabang yang kalah; jendela checkpoint yang bersilang tidak pernah bertentangan dengan epoch bersama (sebelum dan sesudah penyembuhan). |
| `crash_restart` | Crash pada h≈200, restart pada h≈400: resinkronisasi dengan me-replay blok jaringan — tinggi, tip, state sama; transaksi pasca-crash terlihat oleh node yang bangkit. |
| `duplicate_and_delay` | Duplikasi 10 ‰: tinggi akhir, akar state, dan jumlah checkpoint terfinalisasi identik dengan eksekusi tanpa duplikasi (idempotensi: TxReplay/seen-set menyerap duplikat). |
| `concurrent_tx` | Dua owner bersaing pada domain yang sama: tepat satu update menang (ownernya — `NotOwner` bagi penyusup, penolakan bertipe bersih), tidak pernah penerapan state ganda, tx yang kalah tidak masuk chain mana pun. |

### Parameter dan batas durasi

Nilai default: `produce_interval` 7 tick, `sign_interval` 16
tinggi, `sync_interval` 29–60 tick (41 default, diturunkan di
beberapa test), `horizon` 200k–500k tick virtual. Setiap test
menanam penjaga jam-dinding < 60 dtk dalam mode debug
(`assert_under_60s`). Biaya dominan adalah kernel validasi
(~9 ms per `push_block_with_gc` kosong dalam debug, Ed25519 tak
teroptimasi): skenario tetap di bawah ~8 node × 500 blok.

### Batasan

- **Tanpa lapisan transport nyata**: harness ini menguji logika
  chain di bawah pengiriman buruk, bukan swarm (libp2p,
  request-response, backpressure, discovery). Test e2e nyata
  (`crates/scone-network/tests/`) tetap menjadi bukti transport.
- **Sinkronisasi batch di RAM**: blok yang dilayani berasal dari
  jendela RAM (skenario tetap di bawah `RAM_WINDOW_BLOCKS`);
  jalur `BlockPruned` + muat-ulang store tidak disimulasikan
  (dicakup unit test chain).
- **Model anchor jujur**: anchor hanya menandatangani konten yang
  dapat diverifikasi terhadap chain kanonik mereka sendiri;
  ekuivokasi byzantine (penandatanganan-ganda yang disengaja)
  dicegah oleh `SignerGuard` tetapi tidak diuji sebagai serangan
  aktif.
- **Satu partisi pada satu waktu**: peta grup global dan diubah
  sekuensial oleh driver; partisi yang terus tumpang tindih/evolusi
  tidak dimodelkan.

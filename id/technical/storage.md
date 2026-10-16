# Penyimpanan — persistensi node lokal (redb)

> Dokumen normatif. Setiap perbedaan antara dokumen ini dan
> `crates/scone-storage/src/` adalah bug (identifikasi sumber
> kebenarannya, lalu perbaiki). Referensi implementasi: crate
> `scone-storage` (format v1), backend `redb` 4.2.

## Tujuan

Persistensi lokal tertanam node: blok kanonik, state domain,
cache DHT, metadata. Store adalah **persistensi**, bukan
otoritas: state RAM ([`ChainState`], sumber konsensus)
direkonstruksi saat startup dari store, dan setiap tulisan adalah
**delta** (domain yang diubah oleh sebuah blok), bukan salinan
penuh state.

Kendala lintas bagian (menghadap pengguna, tiap milestone):
**jejak memori minimal**. Volume besar (blok, state domain,
cache DHT) hidup di redb dan diakses via kursor/rentang — tidak
pernah dimuat penuh ke RAM, tidak pernah diduplikasi sia-sia.

## Batas: trait `NodeStore`

`crates/scone-storage/src/lib.rs` mendefinisikan satu-satunya
wajah penyimpanan yang terlihat oleh sisa kode (blockchain,
relay M4 mendatang):

- `append_block(height, hash, bytes)` /
  `append_block_with_state(..., deltas)` — append atomik (lihat
  bawah);
- `block_at_height(h)` / `block_by_hash(hash)` — baca titik;
- `tip() -> (height, hash)` — O(1), dibaca dari `meta`;
- `put_domain_state` / `domain_state` / `domain_count` —
  penghitung **terpelihara** (`meta["domain_count"]`), bukan
  `COUNT(*)`;
- `iterate_domains(after, max) -> DomainPage` — paginasi
  berbasis kursor (batch 100 secara default), urutan byte
  menaik `DomainId`;
- `put_tld_state` / `tld_state` / `tld_count` / `iterate_tlds` —
  kontrak sama untuk registri TLD (`tld_count` terpelihara,
  kursor berbatas) (M7d);
- `put_dht_cache` / `dht_cache` — byte `SignedDnsRecord` opaque;
- `resolve_name(canonical)` / `resolve_tld(tld)` — indeks
  persisten Nama→Id (P0.2), pencarian titik O(1), tanpa iterasi
  atas domain (default `None` = backend tanpa indeks, pemanggil
  jatuh ke `iterate_domains` atau state chain);
- `tld_floors_all` / `snapshot_tld_floors_all` — membaca lantai
  umur TLD terbuka (opsi owner: tabel khusus, hidup `tld_floors_v1`
  dan beku `snapshot_v1_floors`);
- `rewind_to_height(h)` — reorg: menghapus blok di atas `h` dan
  mengembalikan `tip`/`tip_height` ke `h` dalam SATU transaksi
  (prasyarat `persist_state_snapshot`, lihat bawah);
- `meta_get` / `meta_set`;
- `snapshot_meta` / `snapshot_domains` / `snapshot_tlds` —
  pembacaan snapshot state boot (M6b; default kosong = backend
  tanpa snapshot, `load_chain` lalu jatuh ke pemuatan penuh).

Aturan:

- **Tanpa tipe redb dalam API**: hanya byte (`&[u8]`, `[u8; 32]`)
  dan tipe core. Backend dapat diganti.
- **Tanpa async, tanpa interior mutability**: tulisan memakai
  `&mut self` (redb menserialisasi penulis), bacaan memakai
  `&self` (satu transaksi baca pendek per panggilan).
- API tidak pernah memvalidasi ulang blok: hash yang diteruskan
  ke `append_block*` adalah hash **yang dihitung ulang** oleh
  pemanggil (`Blockchain::push_block`).

## Skema tabel redb

| Tabel | Kunci (sort) | Nilai |
|---|---|---|
| `blocks_by_height` | tinggi `u64` (b-tree terurut) | `hash(32) ‖ height(8 BE) ‖ blok kanonik` |
| `blocks_by_hash` | hash blok (32 B) | enkode kanonik blok |
| `domains` | `DomainId` (32 B, urutan byte) | `DomainStateBytes` (bawah) |
| `tlds` | `TldId` (32 B, urutan byte) | `TldStateBytes` (bawah, M7d) |
| `dht_cache` | `DomainId` (32 B) | byte `SignedDnsRecord` (opaque) |
| `names_v3` (P0.2) | kunci indeks (32 B, urutan byte) | id target (32 B) |
| `name_reverse_v3` (P0.2) | id target (32 B, urutan byte) | `tag(1) ‖ nama kanonik` |
| `meta` | `&[u8]` | `&[u8]` |
| `snapshot_v1_meta` (M6b) | `u64` = 0 (slot tunggal) | `height(8 BE) ‖ tip_hash(32)` = tepat 40 B |
| `snapshot_v1_domains` (M6b) | `DomainId` (32 B, urutan byte) | `DomainStateBytes` (format sama dengan `domains`) |
| `snapshot_v1_tlds` (M6b) | `TldId` (32 B, urutan byte) | `TldStateBytes` (format sama dengan `tlds`) |
| `tld_floors_v1` (opsi owner) | `TldId` (32 B, urutan byte) | `u64` (lantai kedaluwarsa, BE implisit redb) |
| `snapshot_v1_floors` (M6b) | `TldId` (32 B, urutan byte) | `u64` — salinan beku lantai (state PADA snapshot, bukan tip) |

Kunci `meta` tambahan (P0.3/P0.4):

| Kunci | Nilai |
|---|---|
| `verified_snapshot_v1` (P0.3) | `height(8 BE) ‖ tip_hash(32) ‖ manifest_hash(32)` = 72 B — penanda snapshot terverifikasi yang diimpor (dicadangkan, seperti `tip`) |

Kunci `meta`:

| Kunci | Nilai |
|---|---|
| `format_version` | `u64 BE` = `STORAGE_FORMAT_VERSION` (1) |
| `tip` | hash (32 B) blok tip |
| `tip_height` | `u64 BE` (membuat `tip()` O(1)) |
| `domain_count` | `u64 BE` (penghitung terpelihara) |
| `tld_count` | `u64 BE` (penghitung terpelihara, M7d) |

Header `hash ‖ height` dari `blocks_by_height` membuat tiap
indeks mandiri (dapat diverifikasi tanpa membaca blok); kedua
indeks diperbarui dalam transaksi yang sama.

### Konvensi genesis

Blok genesis deterministik (konstanta protokol) dan **tidak
pernah disimpan** sebagai byte. Store segar memiliki
`tip == (0, genesis_hash)`; blok pertama yang di-append
bertinggi 1. Hash genesis selalu diturunkan ulang
(`scone_blockchain::genesis_hash()`), tidak pernah dibaca.

## Format enkode `DomainStateBytes` (M8b)

57 atau 89 byte menurut tag versi di depan, integer lebar-tetap
big-endian (kanonik, dapat dibandingkan byte-demi-byte):

```text
DomainStateBytes = tag ‖ owner(32) ‖ sequence(8 BE) ‖ registered_at(8 BE)
                   ‖ valid_until(8 BE) [‖ record_hash(32)]

tag = 0x01 : record_hash ada   → tepat 89 byte
tag = 0x02 : record_hash absen → tepat 57 byte
```

- `registered_at` / `valid_until` (M8b): kedaluwarsa registrasi
  (1 tahun saat klaim, dapat diperpanjang, lihat
  `/docs/en/technical/blockchain.md`).

Enkode M7 (41/73 byte, tanpa kedaluwarsa) **ilegitim** sejak
M8b: dekode ketat → `Corrupted`. (Store M7 beregenerasi via
replay: `load_chain_replay`.)

- `owner`: `OwnerId` (32 B);
- `sequence`: sequence `UpdateDomain` terakhir yang diterapkan
  (0 tepat setelah `RegisterDomain`);
- `record_hash`: komitmen on-chain saat ini dari himpunan record
  DNS (`None` selama belum ada `UpdateDomain` yang diterapkan).

Dekode **ketat** (`DomainStateBytes::decode`): tag tak dikenal,
panjang salah, atau byte berlebih → `StorageError::Corrupted`,
tidak pernah panic.

## Format enkode `TldStateBytes` (opsi owner)

Panjang variabel menurut varian enum — 34 byte untuk TLD
**privat**, 2 untuk TLD **terbuka** (tanpa sequence maupun hash
record):

```text
TldStateBytes = tag(0x04) ‖ flag(0x00|0x01) ‖ owner(32)? ‖ until(8)
```

- `flag` (opsi owner, enum `TldState`): `0x00` =
  `TldState::Open` (milik tak seorang pun — mandiri + PoW,
  tanpa byte owner), `0x01` = `TldState::Private` (32 byte owner
  menyusul). Secara ketat kanonik — byte lain → `Corrupted`.
  Invarian "terbuka ⟺ tanpa owner" STRUKTURAL: dua state itu
  tidak bisa tertukar.
- **`until` (8 byte, BE64)**: jam milik TLD sendiri — akhir masa
  privat (klaim/perpanjangan + 3 tahun) atau jadwal kematian
  terbuka (lantai 1 tahun saat lahir, didorong oleh tiap domain
  hidup). Ia atribut entri lagi (perubahan siklus hidup TLD).
  Tabel turunan `tld_floors_v1` (+ salinan bekunya
  `snapshot_v1_floors`) tetap melacak lantai, dibangun ulang
  saat replay dan dipulihkan saat boot — node yang direstart
  membunuh TLD yang sama dengan node kontinu.

Enkode M7d (33 byte, tag `0x01`), M8b (34 byte, tag `0x02`,
boolean `open`), dan tag warisan `0x03` (`until` tertanam)
**ilegitim**: dekode ketat → `Corrupted` (pra-mainnet,
regenerasi via replay adalah jalur migrasi).

Tag hanya ada untuk evolusi format: tata letak mendatang
menaikkan tag dan pembaca lama gagal dengan `Corrupted`
alih-alih menebak. Dekode **ketat** (`TldStateBytes::decode`):
tag tak dikenal, panjang salah, byte berlebih, atau flag
non-kanonik → `StorageError::Corrupted`, tidak pernah panic.

## Jaminan atomicitas

`append_block_with_state` menulis dalam **satu** redb
`WriteTransaction`:

1. blok dalam `blocks_by_height` (cek awal: tinggi tidak boleh
   ada dengan konten berbeda, jika tidak
   `StorageError::Conflict`);
2. blok dalam `blocks_by_hash`;
3. setiap delta state domain (`domains`);
4. penghitung `domain_count` (naik hanya per jumlah domain
   **baru**; `UpdateDomain` atas domain yang ada tidak
   menaikkannya);
5. setiap delta registri TLD (`tlds`, M8b: klaim, transfer,
   buka/tutup) — **dalam transaksi yang sama**: blok yang
   ditulis tanpa perubahan TLD-nya mustahil secara struktural;
6. penghitung `tld_count` (hanya TLD baru);
7. setiap **penghapusan** (M8b): domain yang kedaluwarsa oleh GC
   deterministik (`removed_domains`) dan TLD yang dicabut
   (`removed_tlds`) keluar dari store dalam transaksi yang sama,
   penghitung diturunkan — restart tidak pernah bisa membangkitkan
   registrasi kedaluwarsa;
8. delta **indeks nama** P0.2 (upsert dan penghapusan, lihat
   bawah) — jaminan sama: crash tidak pernah meninggalkan entri
   indeks menunjuk domain yang hilang;
9. `tip` dan `tip_height`.

redb mengkomit gaya-WAL: crash di tengah tulisan meninggalkan
state koheren **sebelumnya**. **Blok yang ditulis tanpa
state-nya karena itu mustahil secara struktural** — inilah
invarian kunci milestone ini.

Jaminan tambahan:

- **Monotonisitas**: append pada tinggi ≠ `tip_height + 1` →
  `StorageError::NonMonotonicHeight`, tidak ada yang ditulis
  (diperiksa *sebelum* membuka transaksi tulis);
- **Idempotensi**: meng-append ulang blok tip saat ini dengan
  pasangan `(tinggi, hash)` sama adalah no-op (jalur cepat
  membandingkan tinggi dan hash; pada tinggi sama, byte blok
  identik juga no-op diam, byte berbeda `Conflict`);
- **Basis data rusak**: berkas terpotong/tercemar → error bertipe
  (`StorageError::Database`/`Corrupted`) saat buka atau baca,
  tidak pernah panic (teruji);
- **Versi format**: `format_version` ≠ 1 →
  `StorageError::UnsupportedFormat` (jalur migrasi eksplisit
  disyaratkan, tanpa menebak).

## Integrasi blockchain (`integration.rs`)

Pemisahan peran: [`ChainState`] di RAM = otoritas konsensus;
store = persistensi. Strategi yang dipilih (yang paling
sederhana, sebagaimana didokumentasikan): **persist state pada
setiap blok** — `store_block` menulis blok + state domain
**yang tersentuh blok itu** (delta). Replay penuh tetap mungkin
dan berfungsi sebagai jalur kendali/perbaikan.

- `store_block(store, chain, block, hash)`: mengenkode blok
  secara kanonik, membaca state akhir domain yang tersentuh DAN
  TLD yang diklaim dari state RAM (sudah diperbarui oleh
  `push_block`), append atomik.
- `load_chain(store)`: **tanpa replay** — membaca tip + blok tip
  (baca blok O(1)), memulihkan state domain dalam halaman 100
  (`DOMAIN_PAGE`) **dan registri TLD dalam halaman 100**
  (`TLD_PAGE`, M7d — menutup jendela restart: tanpanya, node
  yang direstart melupakan setiap TLD yang diklaim dan menolak
  setiap `RegisterDomain` di bawah TLD itu dengan `UnknownTld`,
  padahal disknya memegang state lengkap). Chain RAM hanya
  menyimpan genesis + tip; blok historis tetap di store dan
  dilayani darinya. Sejak M8, `load_chain` lalu membangun ulang
  **indeks anti-replay TXID** (lihat bawah).
- `load_chain_replay(store)`: me-replay semua blok tersimpan
  satu per satu (tidak pernah semua di RAM) dengan validasi
  penuh; tip hasil harus sama dengan tip tersimpan, jika tidak
  `Corrupted`.
- `rebuild_replay_index_from_store(store, chain, tip_height)`
  (M8): rescan `min(REPLAY_WINDOW_BLOCKS, tip_height)` blok
  ter-persist terakhir (satu blok di RAM pada satu waktu) dan
  menyisipkan ulang tiap TxId ke indeks chain **pada tinggi
  inklusi aslinya**, via API publik
  `Blockchain::rebuild_replay_index` (tanpa akses field
  privat). Blok hilang di bawah tip atau tak terdekode dalam
  jendela → `Corrupted` (byte itu sudah dilayani membangun
  chain: kegagalan di sini menandakan store rusak).
- `touched_domains(block)`: id domain yang diubah oleh transaksi
  blok, ter-dedupe, urutan blok.
- `touched_tlds(block)`: id TLD yang dimutasi oleh blok
  (`RegisterTld`, `TransferTld`, `RevokeTld`, `SetTldOpen`),
  ter-dedupe, urutan blok (M8b).
- `store_block_with_removals(...)`: varian `store_block` yang
  juga menerima domain yang dihapus oleh GC blok (M8b) untuk
  tulisan atomik yang dijelaskan di atas. Relay mengumpankan
  daftar ini dari `Blockchain::push_block_with_gc` (hasil
  penerapan mengekspos `gc_removed_domains`). **Pengusiran GC
  cache DHT**: entri `dht_cache` domain yang dihapus dihapus
  dalam transaksi yang SAMA dengan blok (delta append membawa
  tabel `dht_cache`) — record DHT tidak pernah
  melampaui umur domain yang dilayaninya, entri kedaluwarsa
  tidak pernah menumpuk selamanya (test:
  `gc_evicts_the_dht_cache_entry`).

Relay (M4) akan memakai: `load_chain` saat startup, lalu untuk
tiap blok yang diterima `push_block` (RAM) lalu `store_block`
(disk).

## Persistensi reorg (`rewind_to_height` + `persist_state_snapshot`)

Reorg (perpindahan ke cabang saudari yang lebih kuat) tidak bisa
lewat append monotonik — menulis akhiran pemenang akan gagal
dengan `NonMonotonicHeight`. Jalur khusus (perbaikan
minoritas-lengket, relay + store):

1. `rewind_to_height(fork_height)` — SATU transaksi redb
   menghapus blok di atas titik fork dan mengembalikan
   `tip`/`tip_height` (indeks nama dan state hidup diperbaiki
   pada langkah berikutnya);
2. `persist_state_snapshot(store, chain)` — menulis ulang tabel
   hidup (`domains`, `tlds`, indeks nama, lantai TLD) dari
   ekspor state RAM pasca-reorg: tabel hidup TIDAK PERNAH
   menggambarkan cabang yang KALAH saat boot;
3. re-append monotonik akhiran yang diadopsi (delta biasa).

Kegagalan persistensi pada jalur ini dicatat dan didegradasi
(store sembuh saat boot via replay) — tidak pernah fatal.

## Snapshot state boot (M6b)

`load_chain` sebagaimana dijelaskan di atas masih O(ukuran
state): tidak ada blok yang di-replay, tetapi seluruh tabel
`domains` dibaca ulang. Jalur state-terpersist-per-blok-lah yang
membuat replay tak perlu. Snapshot M6b menangani biaya lain:
kasus di mana state RAM harus direkonstruksi **dengan replay**
(perbaikan, kendali, atau backend tanpa state terpersist
per-blok). Ia menawarkan boot dalam **O(tip − H)** — me-replay
hanya akhiran di atas snapshot.

### Tulis

- **Interval**: `SNAPSHOT_INTERVAL = 64` (konstanta publik
  `scone-storage`). Pemicunya adalah append blok
  `k·SNAPSHOT_INTERVAL + 1`: transaksi itu membekukan state
  tabel hidup **sebagaimana adanya sebelum menerapkan blok itu**,
  yaitu state kanonik setelah blok `k·SNAPSHOT_INTERVAL`.
- **Tidak pernah di tip**: snapshot karenanya berada paling
  dekat di `tip − 1`, selalu di bawah ketat tip pada saat ia
  ditulis. Crash tepat setelahnya tidak pernah bisa
  meninggalkan snapshot "di atas" chain yang terpersist.
- **Atomik**: pembekuan (hapus + salin tabel `domains`/`tlds` ke
  `snapshot_v1_*`, tulis `snapshot_v1_meta`) terjadi dalam
  transaksi redb yang SAMA dengan append blok pemicu — crash
  meninggalkan entah snapshot lama utuh, entah yang baru lengkap
  dengan bloknya (WAL redb).
- **Slot tunggal**: setiap snapshot MENGGANTIKAN yang sebelumnya
  (tabel dikosongkan lalu disalin ulang) — tanpa akumulasi,
  tanpa pemangkasan yang harus dikelola, jejak disk dibatasi
  ~2× state.
- Format state beku secara ketat adalah
  `DomainStateBytes`/`TldStateBytes` (pakai ulang, tanpa enkode
  baru); hanya baris `snapshot_v1_meta` yang format baru
  (`height(8 BE) ‖ tip_hash(32)`, dekode ketat: panjang lain →
  `Corrupted`).

### Baca saat boot (`load_chain`)

1. Tip divalidasi seperti selalu (hash dihitung ulang dari
   header blok tip vs `meta["tip"]`).
2. Jika `snapshot_meta()` ada, tingginya `H` memenuhi
   `0 < H < tip_height`, dan hash TERHITUNG ULANG dari blok yang
   tersimpan di `H` sama dengan `tip_hash` snapshot (**jangkar**
   — snapshot asing terhadap chain terpersist, yang diubah, atau
   dari berkas lain terdeteksi di sini), state beku dipulihkan
   dalam halaman 100 lalu hanya blok `H+1..=tip` yang di-replay
   (validasi penuh, identik dengan blok hidup).
3. Tip yang dicapai replay harus sama dengan tip tersimpan.
4. **Setiap kegagalan pada jalur ini (snapshot hilang, tinggi
   tak terpakai, jangkar tidak valid, byte tak terdekode, replay
   gagal) didegradasi diam-diam ke pemuatan penuh pra-M6b**
   (membaca tabel hidup, tanpa replay) — snapshot buruk tidak
   pernah menggagalkan boot. Kompatibilitas mundur adalah kasus
   khusus "tanpa snapshot": store yang ditulis sebelum M6b
   (tanpa tabel `snapshot_v1_*`) termuat identik,
   `snapshot_meta()` bernilai `None`.

Akhiran yang di-replay dibatasi `SNAPSHOT_INTERVAL` (paling
banyak 64 blok saat pemicuan + blok yang ditambang sejak itu):
boot adalah O(tip − H) dengan H dijamin baru.

## Indeks anti-replay TXID dibangun ulang saat boot (M8)

Indeks anti-replay (`included`, TxId → tinggi inklusi, jendela
`REPLAY_WINDOW_BLOCKS = 256` — lihat
`/docs/en/technical/blockchain.md`) adalah **fungsi murni dari
chain kanonik**: ia tidak pernah dipersist. Karena `load_chain`
tidak me-replay riwayat, chain yang dipulihkan sebelum M8 mulai
dengan indeks **kosong** — node yang direstart menerima
re-inklusi transaksi apa pun yang masih dalam jendela replaynya
(keterbatasan terdokumentasi; aturan state tetap satu-satunya
penjaga).

M8 menutup jendela ini: setelah pemulihan (jalur penuh ATAU
snapshot + replay akhiran), `load_chain` memanggil
`rebuild_replay_index_from_store`, yang membaca ulang
`min(REPLAY_WINDOW_BLOCKS, tip_height)` blok ter-persist
terakhir — satu blok di RAM pada satu waktu, memori berbatas
seperti di mana-mana — dan menyisipkan ulang tiap TxId pada
**tinggi aslinya**. Store dengan kurang dari 256 blok karena itu
membangun ulang seluruh indeksnya.

Properti:

- **Keputusan identik dengan node hidup**: entri sama, aturan
  pemangkasan sama (`h > tip − 256`) — node yang restart dan
  node yang tak pernah berhenti menolak replay yang sama (teruji:
  jumlah indeks eksak dan kesetaraan dengan indeks hidup yang
  dipangkas).
- **Semantik pemangkasan tak berubah**: transaksi lebih tua dari
  jendela tetap diakui indeks (lalu gagal aturan state jika
  berlaku) — persis seperti sebelumnya.
- **Snapshot TIDAK membekukan indeks**: replay akhiran snapshot
  merekonstruksi entri di atas H, rescan M8 menambah bagian
  jendela DI BAWAH H (kontrak sama dengan owner_keys/grace:
  dibangun ulang, tidak pernah dipersist).
- **Biaya**: ≤ 256 baca blok titik saat boot, O(jendela) —
  independen dari umur chain.
- Blok hilang atau tak terdekode **dalam jendela** menggagalkan
  boot dengan `Corrupted` (byte itu sudah dilayani membangun
  chain); di luar jendela, blok historis tak tersentuh.

`load_chain_replay` (nol kepercayaan) membangun ulang indeks
dengan cara yang sama melalui replay penuhnya — kedua jalur
konvergen.

## Indeks persisten Nama → Id (P0.2)

`DomainId = BLAKE3-256(DOMAIN_ID_VERSION ‖ nama kanonik)` tidak
invertibel: me-resolve nama (relay/DNS) memerlukan indeks
persisten. P0.2 menambahkannya ke store, dengan disiplin
transaksional yang sama seperti sisanya.

### Skema

Dua tabel pendamping, ditulis dan dihapus bersama:

```text
names_v3        key  = BLAKE3-256("SCONE-NAME-IDX-V1" ‖ nama kanonik)  (32 B)
                val  = id target (32 B : DomainId atau TldId)

name_reverse_v3 key  = id target (32 B)
                val  = tag(0x01 domain | 0x02 TLD) ‖ nama kanonik
```

- Kunci `names_v3` adalah **hash nama, bukan nama teks polos**
  (tabel indeks ringkas, tanpa gossip nama via dump). Kunci
  adalah **fungsi murni dari nama**: indeks dapat dibangun ulang
  kapan pun dari chain (`rebuild_name_index`), tanpa metadata
  tambahan yang dipersist untuknya.
- `name_reverse_v3` menyimpan namanya (tepat sekali) karena
  jalur penghapusan (GC domain kedaluwarsa, `RevokeTld`) hanya
  mengenal **id**: kunci ter-hash yang harus dihapus harus
  ditemukan entah bagaimana. Tag membedakan dua namespace.
- Pemisahan ketat: `resolve_name` TIDAK PERNAH menjawab entri
  TLD dan `resolve_tld` tidak pernah entri domain (tag tabel
  balik yang menentukan; `DomainName` punya ≥ 2 label, TLD
  punya 1 — kedua derivasi kunci berbagi ruang `names_v3`
  tetapi bukan hasilnya).
- `DomainId` tetap 32 byte, tidak pernah direduksi (aturan keras
  proyek: identitas konsensus, resistensi tabrakan).

### Konsistensi transaksional

Upsert (`RegisterDomain`, `AssignDomain`, `RegisterTld` — nama
dibawa teks polos oleh transaksi) dan penghapusan (GC,
pencabutan — id polos) dibawa oleh `StateDelta` blok
(`name_upserts` / `name_removals`) dan diterapkan dalam
**transaksi redb yang sama** dengan tulisan blok dan state.
Crash karena itu tidak bisa meninggalkan entri indeks menunjuk
domain yang hilang (tanpa hantu), maupun domain terdaftar tanpa
entrinya. Append yang gagal tidak menulis apa pun (teruji).
Varian bernama `put_domain_state_with_name` /
`put_tld_state_with_name` / `remove_domain_state` menawarkan
atomicitas yang sama bagi perkakas perbaikan.

`store_block` / `store_block_with_removals` menurunkan delta
nama dari transaksi blok yang SAMA — tanpa parameter ekstra di
antarmuka relay.

### Pembangunan ulang (`RedbStore::rebuild_name_index`)

Indeks adalah fungsi murni dari chain kanonik, seperti indeks
anti-replay M8. `rebuild_name_index` membangun ulang dari **blok
ter-persist** (`DomainStateBytes` TIDAK memuat nama — hanya
owner/sequence/kedaluwarsa — maka sumber kebenarannya adalah
aliran transaksi, tempat `RegisterDomain` dan `AssignDomain`
membawa nama kanonik dalam teks polos):

1. putaran 1: pindai `blocks_by_height` genesis→tip, satu blok
   di RAM pada satu waktu; kumpulkan ikatan `nama → id` yang
   terlihat dalam `RegisterDomain`/`AssignDomain`/`RegisterTld`;
2. putaran 2: filter hanya target **hidup** (state ter-persist
   otoritatif) — domain yang terdaftar lalu GC'd/kedaluwarsa
   tidak diindeks ulang;
3. putaran 3: hapus + tulis ulang penuh kedua tabel dalam SATU
   transaksi (crash → indeks lama atau baru lengkap).

Blok hilang atau tak terdekode → `Corrupted` (byte yang sama
yang membangun chain). Biayanya O(nama berbeda yang terlihat) di
RAM (≤ 253 B per nama) dan O(blok) dalam bacaan — perkakas
perbaikan, bukan jalur boot.

### Apa yang BUKAN indeks ini

- Bukan otoritas: state chain RAM tetap satu-satunya otoritas;
  indeks adalah cache yang dapat dibangun ulang (penyimpangan
  adalah bug transaksionalitas, terdeteksi oleh
  `rebuild_name_index` yang dibandingkan dengannya).
- Tidak dibekukan dalam snapshot M6b: tabel
  `names_v3`/`name_reverse_v3` hidup dan mengikuti delta tiap
  blok; snapshot state tidak menyematkannya (rekonstruksi via
  delta saat resume, atau `rebuild_name_index` sebagai
  perbaikan).

### Apa yang TIDAK dikomitmenkan snapshot

State RAM `ChainState` membawa struktur internal yang tidak
dikomitmenkan konsensus (kunci owner, jendela masa tenggang).
Mereka TIDAK dibekukan: replay akhiran membangun ulang dengan
menerapkan transaksi (deterministik, konten sama dengan replay
penuh untuk jendela yang relevan). Indeks anti-replay TXID
mengikuti kontrak yang sama (M8, lihat bagian sebelumnya):
dibangun ulang saat boot, tidak pernah dibekukan. Kesetaraan
bit-eksak dengan `load_chain_replay` diverifikasi oleh test
pada state kanonik (state_root + SMT).

## Snapshot state TERVERIFIKASI — potongan + manifest + verifikasi kriptografis (P0.3)

Snapshot M6b di atas mempercepat boot node yang mempercayai
disknya. P0.3 menangani kebutuhan berbeda: mengangkut state
antarnode (bootstrap node baru, perbaikan) tanpa memercayai
transportnya. Format dan verifikasinya hidup di
`crates/scone-storage/src/verified_snapshot.rs` + rekonstruksi
akar di `crates/scone-blockchain/src/snapshot_verify.rs`.

### Model kepercayaan

Snapshot membawa state blok **terfinalisasi** — checkpoint yang
ditandatangani kuorum komite (`CheckpointData.state_root`, akar
`state_root_smt` O(1)). Keaslian checkpoint (kuorum tanda tangan
Ed25519) divalidasi oleh lapisan chain SEBELUM impor. Impornya
sendiri memverifikasi kesesuaian snapshot ↔ checkpoint dengan
PENGHITUNGAN ULANG:

1. **hash tiap halaman**: `blake3("SCONE-SNAP-PAGE-HASH" ‖
   enkode halaman penuh)` harus sama dengan hash yang
   diumumkan manifest (satu byte palsu → penolakan);
2. **jumlah entri == penghitung manifest** (halaman hilang atau
   penghitung salah → penolakan);
3. **akar SMT dihitung ulang dari entri yang diimpor ==
   `checkpoint.data.state_root`** — PEMERIKSAAN kriptografis:
   seluruh state (domain + TLD) dilipat ulang ke SMT segar
   (`snapshot_verify::recompute_state_root`, leaf
   `SCONE-LEAF-DOM-V1`/`SCONE-LEAF-TLD-V1` persis seperti jalur
   hidup) dan akarnya harus sama dengan yang ditandatangani
   komite. Snapshot palsu, terpotong, atau tidak lengkap ditolak
   SEBELUM TULISAN APA PUN;
4. **konsistensi struktural**: `height`/`tip_hash`/`state_root`
   manifest == milik checkpoint; id meningkat ketat (tanpa
   duplikat); indeks halaman sekuensial; batas DoS
   (≤ 10.000 entri/halaman, ≤ 2^20 halaman, anggaran total).

Kegagalan verifikasi → `StorageError::SnapshotRejected`, store
tetap ketat tak berubah (semuanya diverifikasi sebelum
transaksi tulis).

### Format terserial (ketat, big-endian, dekode tanpa ampun)

```text
manifest = "SCONE-SNAP-MAN-V1"
        ‖ height(8) ‖ tip_hash(32) ‖ state_root(32)
        ‖ page_count(4) ‖ domain_count(8) ‖ tld_count(8)
        ‖ page_hashes(page_count × 32)

page    = "SCONE-SNAP-PAGE-V1"
        ‖ index(4) ‖ height(8) ‖ state_root(32)      ← tiap halaman
        ‖ entry_count(4) ‖ entries                   mengidentifikasi diri

entry   = kind(0x01 domain | 0x02 TLD | 0x03 lantai TLD) ‖ id(32)
          ‖ [len(2) ‖ state | floor(8 BE)]
          (state = DomainStateBytes 57/89 B atau TldStateBytes 2/34 B —
           format penyimpanan kanonik yang ada, dipakai ulang; kind
           0x03 (lantai umur TLD terbuka, opsi owner) punya
           format khusus TANPA prefiks panjang:
           id(32) ‖ floor(8 BE), ukuran tetap)
```

Urutan halaman: semua entri domain (id menaik), lalu semua entri
TLD (id menaik), lalu lantai umur TLD DIBEKUKAN pada tinggi
snapshot (`snapshot_tld_floors_all` — bukan tabel hidup) —
urutan paginasi kursor store. Tiap halaman membawa `(indeks,
tinggi, state_root checkpoint, entri)`: satu halaman sendirian
tahu snapshot milik siapa ia. Halaman berisi 256 entri saat
ekspor (`VERIFIED_SNAPSHOT_PAGE_ENTRIES`); batas DoS saat
impor. Lantai TLD (`tld_floors_v1`) bagian dari format
portabel: kind 0x03 membawanya, dan impor mensyaratkan mereka
meningkat ketat dan menargetkan TLD yang HADIR dalam snapshot —
jika tidak, importer tidak akan pernah membunuh apa yang akan
dibunuh exporter (fork diam, ditolak).

### Ekspor (`export_verified_snapshot(store, checkpoint)`)

Menyerialisasi state CHECKPOINT TERAKHIR TERFINALISASI — bukan
tip. Memancarkan domain, TLD, lalu lantai umur TLD yang dibekukan
pada tinggi snapshot (simetri ekspor/impor: ekspor secara
historis menghilangkan lantai — diperbaiki, test regresi
`export_carries_tld_lifetime_floors`). Prasyarat: store memegang
snapshot ter-persist M6b (`snapshot_v1_*`) tepat pada tinggi
checkpoint (jendela `CHECKPOINT_KEEP`/`SNAPSHOT_INTERVAL`
menjamin state seperti itu tersedia berkala); jangkar blok
DIHITUNG ULANG dari byte tersimpan dan harus sama dengan
`checkpoint.data.block_hash`. Jika tidak →
`StorageError::SnapshotUnavailable` (ekspor adalah tindakan
eksplisit: tunggu interval/checkpoint berikutnya).

### Impor (`import_verified_snapshot(store, checkpoint, manifest, pages)`)

Memverifikasi segalanya (lihat atas), lalu menulis dalam SATU
transaksi redb: tabel hidup `domains`/`tlds` + penghitung +
`tip = (H, hash checkpoint)`, tabel beku `snapshot_v1_*`
berjangkar pada blok yang sama, dan penanda cadangan
`meta["verified_snapshot_v1"]` (`height ‖ tip_hash ‖
manifest_hash`). Target: hanya store kosong (jika tidak
`SnapshotUnavailable`).

### Biaya verifikasi (terdokumentasi)

Rekonstruksi SMT via penyisipan terurut: N entri × O(40) hash ≈
O(N·40). ~2,1 M hash blake3 untuk 100.000 domain (puluhan ms).
SEKALI SAJA — impor bootstrap/perbaikan, bukan jalur per-blok:
dapat diterima. Penyisipan terurut (id menaik) menciptakan tiap
titik cabang hanya sekali; bagaimanapun akar hanya bergantung
pada konten (jaminan SMT, fuzz diferensial di `smt.rs`).

## Bootstrap state — tanpa replay genesis (P0.4)

Node baru bergabung dengan jaringan tanpa me-replay riwayat dari
genesis:

```text
checkpoint terfinalisasi (relay, divalidasi oleh lapisan chain)
        │
        ▼
snapshot terverifikasi (manifest + halaman, verifikasi kriptografis di atas)
        │  impor → state pada H + tip (H, hash) + penanda
        ▼
blok jangkar H (diminta dari jaringan BERDASARKAN HASH — append_bootstrap_anchor)
        │
        ▼
blok terbaru H+1..H+k (relay, tinggi meningkat, append atomik
        │               biasa — API penyimpanan hanya mensyaratkan ini)
        ▼
tersinkron — boot me-replay HANYA akhiran di atas H
```

- **API penyimpanan terekspos ke relay**:
  `append_bootstrap_anchor(H, hash, bytes)` (meletakkan blok
  jangkar: penanda wajib, konsistensi tinggi/hash dengan
  penanda, idempoten) lalu append atomik biasa per tinggi
  menaik. Pengambilan jaringan (per hash dari manifest) urusan
  relay, bukan store.
- **`load_chain`** mengenali penanda: riwayat di bawah H absen
  secara sah (node non-arsip). Konkret: rescan jendela
  anti-replay M8 dilantai pada H (blok di bawah H tidak ada dan
  tidak dikatakan hilang); tip tanpa blok tersimpan SETELAH
  impor (jangkar belum diletakkan) → diagnosis bertipe
  `SnapshotUnavailable` ("jangkar harus datang dari jaringan"),
  bukan korupsi.
- **Node arsip vs full-state**: node arsip (flag operator)
  menyimpan semuanya — ia melayani blok historis ke jaringan dan
  boot via jalur yang ada. Node full-state (default P0.4 untuk
  node baru yang DIKONFIGURASI demikian) boot pada H: ia
  memvalidasi blok mendatang (replay akhiran) tetapi tidak bisa
  melayani riwayat di bawah H atau me-replay genesis. **Perilaku
  default TIDAK berubah**: jalur ini hanya aktif jika operator
  secara eksplisit memicu impor (`import_verified_snapshot`
  adalah tindakan khusus, tidak pernah otomatis); setiap node
  yang ada boot identik.
- **Kesetaraan dengan node penuh**: setelah bergabung dengan
  akhiran H+1..H+k, state, tip, dan `state_root_smt` bit-eksak
  dengan node penuh pada tip yang sama (teruji); indeks
  anti-replay mencakup tepat tx pada/di atas H — node non-arsip
  tidak bisa mengenal jendela di bawah H, aturan state tetap
  penjaga (tx terduplikasi di bawah H gagal pada state, tidak
  pernah diterima dua kali).

## Kebijakan memori

- **Kursor di mana-mana**: `iterate_domains` membatasi halaman
  ke `maks` entri (100 secara default) via `range` malas redb;
  seluruh tabel domain tidak pernah termaterialisasi. Daftar
  besar (TLD, domain) wajib lewat kursor ini (`iterate_tlds`,
  kontrak sama, M7d).
- **Hanya delta**: `append_block_with_state` hanya menulis
  domain dan TLD yang diubah oleh blok — tidak pernah state
  penuh;
- **Tanpa duplikasi sinkron**: state RAM tidak di-persist ulang
  per blok; state disk tidak dimuat ulang per blok;
- **Baca titik**: setiap baca membuka transaksi baca pendek,
  mengembalikan nilai, menutup; `AccessGuard` redb hidup hanya
  selama salinan berbatas;
- **Blok historis tidak pernah di RAM**: `load_chain` hanya
  memuat tip; `load_chain_replay` mengalir tinggi demi tinggi;
- **Penghitung terpelihara**: `domain_count` O(1), bukan
  pemindaian.

## Batas DoS

Store membatasi apa yang bisa dibuat peer permusuhan dipersist
atau dialokasikan:

- **`MAX_DHT_CACHE_ENTRY` (64 KiB)**: record DHT lebih besar
  ditolak sebelum transaksi tulis apa pun oleh `put_dht_cache`
  → `StorageError::TooLarge` (tidak ada yang ditulis);
- **`MAX_DOMAIN_PAGE` (10.000)**: `iterate_domains` *secara
  internal* membatasi halaman yang diminta ke
  `min(maks, 10_000)` — `maks` berlebihan tidak pernah
  termaterialisasi lebih dari 10.000 state di RAM;
  `iterate_tlds` punya batas sama (`MAX_TLD_PAGE`, M7d);
- **Kunci `meta` cadangan**: `meta_set` menolak `tip`,
  `tip_height`, `format_version`, `domain_count`, `tld_count`,
  dan `verified_snapshot_v1` (P0.3) dengan
  `StorageError::ReservedKey` — menimpa akuntansi internal store
  akan diam-diam merusaknya.
- **Cache DHT berbatas ganda (LRU)**:
  `DHT_CACHE_MAX_ENTRIES` (100.000 entri) DAN
  `DHT_CACHE_MAX_BYTES` (2 GiB — penjaga kasus permusuhan;
  tiap entri sudah dibatasi 64 KiB oleh `MAX_DHT_CACHE_ENTRY`),
  batas mana yang lebih dulu tercapai mengusir per urutan kunci.
  Penghitung (entri + byte) dipelihara saat tulis — tanpa
  pemindaian untuk menghitung; pembangunan ulang eksak hanya
  saat buka (satu-satunya pemindaian penuh, saat boot). Aksesor
  `dht_cache_entries`/`dht_cache_bytes`, batas dapat disetel via
  `set_dht_cache_bounds` (test:
  `dht_cache_respects_the_double_bound`).

## Model kepercayaan

Dua jalur pemuatan, dua level jaminan:

- **`load_chain` — kepercayaan disk lokal**: hash tip
  **dihitung ulang** dari header blok tip dan dibandingkan
  dengan `meta["tip"]` tersimpan (penyimpangan →
  `StorageError::Corrupted`, tidak pernah chain yang dipulihkan
  atas hash palsu), dan state domain didekode ketat. Namun,
  blok historis **tidak di-replay**: state terpersist diterima
  apa adanya. Inilah startup cepat node yang mempercayai
  disknya sendiri (hanya ditulis oleh node yang sama, atomik).
  Sejak M6b, jalur ini lebih dulu memakai **snapshot state**
  bila terpakai (lihat bawah): jangkar snapshot dihitung ulang
  dan akhiran di atas H divalidasi ulang seperti blok hidup —
  bagian yang di-replay karenanya nol-kepercayaan, basis beku
  kepercayaan-disk.
- **`load_chain_replay` — nol kepercayaan**: semua blok dibaca
  ulang dan divalidasi ulang satu per satu (validasi penuh,
  identik dengan blok hidup); tip hasil harus sama dengan tip
  tersimpan. Inilah perkakas kendali/perbaikan.

**Rekomendasi**: node ketat (atau node yang berkas
penyimpanannya mungkin telah dimanipulasi — disk bersama,
pemulihan backup meragukan) harus mulai via `load_chain_replay`,
bukan `load_chain`.

## Konkurensi

redb 4.x mengizinkan **satu objek `Database` per berkas dan per
proses** (double `open` simultan gagal dengan
`DatabaseAlreadyOpen`). Model yang didukung: `RedbStore` yang
diklon (`Arc<Database>`) dibagi lintas utas — penulis
diserialisasi oleh `&mut self` + kunci internal redb, pembaca
bersamaan (transaksi baca snapshot). Setelah semua handle
sepenuhnya dijatuhkan, berkas bisa dibuka ulang.

## Test (`tests/redb_store.rs` + modul `#[cfg(test)]`)

Setiap test memakai berkas tmp redb-nya sendiri (`tempfile`).
Cakupan:

- roundtrip blok genesis→N, restart, baca ulang, replay;
- M7d: registri TLD dipersist per delta atomik, dipulihkan saat
  restart (`RegisterDomain` di bawah TLD yang diklaim sebelum
  restart tetap dapat diterima — jendela tertutup), penghitung
  eksak saat buka ulang, kursor `tlds` menaik, append gagal
  tidak meninggalkan TLD, kunci cadangan `tld_count` ditolak;
- atomicitas: append gagal tidak meninggalkan apa pun (tip,
  domain, indeks utuh); delta duplikat dihitung sekali;
- kursor domain: paginasi 100/7, urutan `DomainId` menaik,
  mulai ketat setelah id yang ada atau hilang, `maks = 0`;
- penghitung `domain_count` eksak (sisip/perbarui/buka ulang);
- roundtrip cache DHT + timpa;
- konkurensi: baca dari dua utas pada handle klon;
- basis data rusak: berkas terpotong / sampah → error bertipe,
  tanpa panic (`catch_unwind`);
- M6b: snapshot ditulis pada interval, ketat di bawah tip; boot
  dari snapshot = replay penuh bit-eksak (state_root + SMT,
  state logis lengkap, tip); snapshot yang diubah (tip_hash
  rusak, state tak terdekode, tinggi > tip, meta salah panjang)
  → fallback diam ke pemuatan penuh, chain akhir sama; akhiran
  yang di-replay < interval; penggantian snapshot (tanpa
  akumulasi) saat melintasi interval; store pra-M6b tanpa tabel
  snapshot termuat identik;
- M8: indeks anti-replay dibangun ulang saat boot — tx yang
  di-re-inklusi dalam jendela ditolak (`TxReplay`) setelah
  pemulihan; tx lebih tua dari jendela diterima (pemangkasan tak
  berubah, aturan state yang memutuskan); jumlah eksak ==
  indeks hidup (== `load_chain_replay`) untuk store lebih pendek
  dan lebih panjang dari jendela; boot snapshot: bagian jendela
  di bawah H dibangun ulang oleh rescan, bagian atas oleh replay
  akhiran;
- P0.2: indeks nama — `resolve_name == DomainId::from_name`
  untuk 120 domain (hidup, setelah restart, setelah hapus +
  bangun ulang), `resolve_tld` untuk TLD yang diklaim; GC domain
  kedaluwarsa menghapus entri (tanpa hantu), sebelum dan
  sesudah pembangunan ulang; pemisahan ketat namespace
  domain/TLD; append gagal tidak meninggalkan entri indeks;
  kunci indeks adalah derivasi BLAKE3 yang didokumentasikan;
- P0.3/P0.4 (`tests/verified_snapshot.rs`): ekspor → impor →
  boot == replay penuh (state, tip, `state_root_smt` bit-eksak;
  state yang diimpor adalah milik blok terfinalisasi, bukan
  tip); halaman dipalsukan satu byte → penolakan sebelum
  tulisan apa pun (store ketat tak berubah); halaman hilang →
  penolakan; `state_root` salah (manifest ↔ checkpoint, lalu
  akar terhitung ulang ≠ akar bertanda) → penolakan;
  penghitung/tinggi salah / tip tak konsisten / id duplikat →
  penolakan; impor mensyaratkan store kosong; ekspor
  mensyaratkan snapshot ter-persist pada tinggi checkpoint;
  kontrak jangkar (`append_bootstrap_anchor`: penanda wajib,
  konsistensi, idempotensi); format terserial ketat (byte
  berlebih, tag, batas DoS); snapshot pada H lalu blok
  H+1..H+k bergabung → chain identik dengan node penuh
  (state/tip/akar), boot tanpa riwayat di bawah H, indeks
  anti-replay = tepat tx pada/di atas H.

[`ChainState`]: ../../../crates/scone-blockchain/src/state.rs

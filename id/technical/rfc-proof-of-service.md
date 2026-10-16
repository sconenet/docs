# RFC — Bukti operasi dan komite finalitas (proof-of-service)

> Status: NORMATIF — diimplementasikan ujung ke ujung (langkah 1
> sampai 5: core/protocol/blockchain/network/CLI; dokumen ini
> adalah bentuk konsolidasinya). Setiap aturan di bawah
> diverifikasi terhadap kode; konstanta yang dikutip adalah nilai
> yang terkode.
> Tanggal: Oktober 2026 (konsolidasi normatif). Menyentuh:
> `scone-core`, `scone-protocol`, `scone-blockchain`,
> `scone-network`, `scone` (CLI).

## 1. Motivasi

Kumpulan finalitas adalah kumpulan owner (owner domain/TLD hidup).
Memiliki memberi kursi — **mengoperasikan** tidak memberi apa-apa.
RFC ini mengikat hak menandatangani checkpoint pada bukti operasi
relay daring, tanpa pernah mensyaratkan kepemilikan:

- **Kebebasan terpelihara**: TLD privat tetap dapat diberikan tanpa
  syarat aktivitas selama masanya diperpanjang (`RenewTld`, masa
  3 tahun). Hanya *penandatanganan checkpoint* yang diperoleh
  melalui kehadiran.
- **Insentif beroperasi**: mengoperasikan TLD privat (atau kursi
  apa pun) = duduk di komite. Operator ikut menjaga jaringan
  tetap hidup dan protokol mengakuinya.
- **Desentralisasi terukur**: komite menjadi himpunan operator
  yang *benar-benar daring*, bukan kunci dorman.

## 2. Ringkasan

Transaksi `Heartbeat` ringan (diskriminan `0x71`), ditandatangani
oleh kunci **operator**, dipancarkan sekali per epoch. State
menampung jendela geser heartbeat terbaru. Komite sebuah epoch
dipilih **dari subkumpulan pool yang operator efektifnya memiliki
heartbeat segar**. Tanpa heartbeat: tidak menandatangani, tetapi
tidak kehilangan hak kepemilikan.

## 3. Transaksi Heartbeat

```
Heartbeat {
    operator: PublicKey,     // kunci Ed25519 operator (penandatangan)
    owner: OwnerId,          // identitas DITURUNKAN ULANG dari operator (tidak pernah dipercaya)
    tip_seen: [u8; 32],      // tip terakhir yang dilihat node
    height_seen: u64,        // tinggi yang bersesuaian
    timestamp: u64,          // detik Unix
} + tanda tangan Ed25519 atas payload kanonik (jaringan disuntikkan)
```

Body bertanda tangan (wire, `scone-protocol`):
`operator[32] ‖ owner[32] ‖ tip_seen[32] ‖ height_seen.v ‖
timestamp.v ‖ operator[32]` (kunci penandatangan muncul di ekor
body, seperti setiap tx) `‖ signature[64]`, yaitu ~230 byte
terenkode — **tanpa PoW**: biayanya adalah kehadiran kontinu
(satu heartbeat per epoch, berbulan-bulan).

Penandatangan ADALAH operatornya: `owner` selalu diturunkan
ulang dari `operator` (tidak pernah diambil dari input), dan
dekode menolak dengan error bertipe (`InvalidOwner`) kunci tanda
tangan yang berbeda dari field `operator` di kepala.

- `tip_seen`/`height_seen` melayani observabilitas (deteksi
  operator buggy atau tertinggal); ia TIDAK memengaruhi validitas
  (node yang sedikit tertinggal tetap node hidup).

### 3.1 Validasi murni (scone-core)

1. Tanda tangan Ed25519 valid atas payload kanonik jaringan.
2. `owner` == identitas yang diturunkan dari `operator`
   (`InvalidOwner` jika tidak) — diperiksa saat enkode, saat
   dekode DAN oleh `validate()`.

### 3.2 Aturan state (penerapan, scone-blockchain)

Saat penerapan heartbeat dalam blok pembawa dengan tinggi
`block_height` (setiap penolakan mendahului mutasi apa pun):

1. **Anti-replay lintas-epoch**: `timestamp` meningkat ketat per
   operator — penolakan bertipe `StaleHeartbeat` jika
   `timestamp <= operator_last_hb[operator]`.
2. **Keunikan per-epoch**: hanya satu heartbeat diterima per
   (`operator`, epoch blok pembawa) — duplikat ditolak dengan
   error bertipe (`HeartbeatDuplicate`). Epoch adalah
   `block_height / epoch_min_blocks` (definisi sama dengan
   finalitas, bukan konstanta baru).
3. Penyisipan terjurnal ke jendela + pembaruan
   `operator_last_hb` (dapat dibatalkan dalam reorg, undo
   bit-eksak).

### 3.3 Anti-spam

- 1 tx diterima per operator per epoch — aliran maksimum adalah
  `pool_size / epoch_secs`, dapat diabaikan.
- Tanpa biaya (pra-ekonomi): lajunya terbatas secara struktural.

## 4. Delegasi operasi

Tautan kursi → operator melewati dua transaksi khusus:

- `SetTldOperator { tld_id, operator }` (diskriminan `0xA9`),
  ditandatangani oleh owner **saat ini** dari TLD **privat** (TLD
  terbuka tidak punya siapa pun untuk menandatangani secara sah —
  penolakan bertipe `TldIsOpen`). `operator: None` = pencabutan,
  kembali ke operasi mandiri oleh kunci owner; dapat dicabut
  seketika.
- `SetDomainOperator { domain_id, operator }` (diskriminan
  `0x5E`), ditandatangani oleh owner domain saat ini. Semantik
  sama.
- `RegisterTld` / `RegisterDomain` / `AssignDomain` TIDAK membawa
  field `operator`: tautan kursi → operator selalu merupakan
  fakta on-chain baru, diselesaikan di tip.

Opsi operator di wire bersifat kanonik: `0x00` = `None`
(operasi mandiri), `0x01` = kunci mentah 32-byte; nilai lain
ditolak saat dekode.

State memelihara:

```
seat_operators: HashMap<OwnerId, OperatorKey>   // per kursi
operator_seats: HashMap<OperatorKey, u64>       // penghitung berbatas
operator_last_hb: HashMap<OperatorKey, u64>     // ts terakhir yang diterima
```

Aturan penerapan (`apply_seat_operator`, terjurnal):

- **Idempotensi**: mendelegasikan ulang ke operator yang sama
  (atau mencabut delegasi yang tidak ada) adalah no-op tanpa
  jurnal (undo bit-eksak mensyaratkannya).
- **Batas anti-sewa**: sebuah operator hanya bisa didelegasikan
  oleh **satu kursi saja** pada satu waktu
  (`MAX_SEATS_PER_OPERATOR = 1`). Kursi kedua yang mendelegasikan
  ke operator yang sudah diambil → penolakan bertipe
  `OperatorAlreadyDelegated`. Alasan: tanpa batas, operator besar
  menjual penandatanganan secara grosir dan menciptakan kembali
  sentralisasi — persis yang ingin dihindari oleh komite terpilih.

## 5. Jendela kesegaran

### 5.1 Jendela geser

```
recent_heartbeats: BTreeMap<(epoch, OperatorKey), ()>
```

- Entri disisipkan saat penerapan heartbeat yang valid (§3.2).
- Pembersihan deterministik pada SETIAP penerapan blok
  (`gc_heartbeats_journaled`): setiap entri epoch `e` dengan
  `e + HEARTBEAT_WINDOW_EPOCHS <= epoch` dihapus (terjurnal —
  dapat dibatalkan dalam reorg, seperti antrean kedaluwarsa).
- `HEARTBEAT_WINDOW_EPOCHS = 2`: epoch saat ini + epoch
  sebelumnya — seorang operator boleh melewatkan SATU epoch
  tanpa kehilangan kesegaran, yang kedua mengakhirinya.
- **Tutup memori struktural**: jendela menampung paling banyak
  `HEARTBEAT_WINDOW_EPOCHS × operator_count` entri (satu per
  operator per epoch, kunci tuple); `operator_last_hb` dibatasi
  jumlah operator dan tidak pernah dibersihkan (aturan
  anti-replay bersifat monoton).

### 5.2 Kesegaran

"Segar" = keberadaan entri `(e, operator)` untuk `e` dalam
`[epoch − HEARTBEAT_WINDOW_EPOCHS + 1, epoch]`. Karena state
membersihkan di setiap blok, keberadaan dalam `recent_heartbeats`
ADALAH kesegaran (`operator_is_fresh`).

### 5.3 Operator efektif

**Operator efektif** sebuah kursi adalah `seat_operators[seat]`
jika delegasi hidup ada, jika tidak kunci owner
(`effective_operator`). Kursi menandatangani di bawah kunci
efektif itu — delegasi memindahkan hak menandatangani, bukan
kepemilikan.

## 6. Pemilihan komite

```
eligible = { operator efektif dari kursi pool beku
             (kunci ter-ban dikecualikan) yang memiliki heartbeat segar }

if |eligible| >= MIN_FINALITY_COMMITTEE_SIZE (= 4) :
    committee = select_committee(seed, eligible)   // undian yang sudah ada
else :
    committee = select_committee(seed, pool)       // degradasi aman
```

Detail normatif (diverifikasi di `finality.rs`, dispesifikasikan
di `/docs/en/technical/blockchain.md` § "Kursi dan komite
finalitas"):

- `pool` adalah pool **yang dibekukan pada checkpoint terakhir
  terfinalisasi** (`FinalizedBase.pool` — bukan state hidup):
  view lokal akan menyimpang antar node dan kuorum tidak akan
  pernah terbentuk;
- `seed` diturunkan dari hash konten checkpoint sebelumnya
  (`signing_hash`);
- setiap kursi pool beku diselesaikan ke operator EFEKTIFNYA oleh
  state di tip (delegasi dan heartbeat adalah fakta on-chain
  baru: dua node pada tip yang sama menghitung himpunan yang
  sama); kunci ter-ban tetap dikecualikan;
- undian berjalan atas himpunan kunci efektif yang
  terurut/ter-dedupe (`select_committee` tak berubah).

**Degradasi eksplisit dan aman**: jaringan yang sedang bootstrap
(tanpa heartbeat), atau pemadaman umum operator, jatuh kembali ke
seluruh pool — finalitas tidak pernah hilang menurut konstruksi.
Ini aturan "tidak pernah deadlock" proyek yang diterapkan di
sini.

### 6.1 Bootstrap dingin

Epoch 0 (dan selama kurang dari
`MIN_FINALITY_COMMITTEE_SIZE` operator segar): seluruh pool.
Batasnya mengencang secara alami seiring operator memancarkan.
Tanpa masa transisi khusus yang harus dikonfigurasi.

## 7. Emisi otomatis (scone-network)

Pada setiap siklus produksi (`Wake::Produce` — tanpa timer baru),
relay memeriksa apakah BATAS EPOCH dilewati
(`height / epoch_min_blocks`, definisi sama dengan finalitas).
Pada siklus pertama epoch baru, ia memancarkan heartbeat jika:

1. kunci anchor terpasang (`--anchor-key` atau otomatis), DAN
2. kunci itu sendiri memegang kursi hidup (cek LOKAL: kunci ada
   di kumpulan kelayakan on-chain — owner domain/TLD hidup,
   kunci ter-ban dikecualikan).

Anti-spam lokal: paling banyak SATU emisi per epoch per node
(memo `last_emitted_epoch`, memori lokal, bukan state chain);
memo hanya diambil setelah injeksi mempool yang berhasil —
kegagalan mencoba lagi di siklus berikutnya. Tidak ada yang fatal
di sini: paling buruk heartbeat menunggu siklus berikutnya.

**Operator terdelegasi** (yang bukan owner sendiri) tidak
dicakup emisi otomatis: mereka memancarkan via pembangun CLI §8
(operasi manual atau air-gapped). Heartbeat TIDAK harus
diproduksi ke dalam blok oleh pemancarnya sendiri: ia tx seperti
lainnya (mempool, produksi oleh siapa pun).

## 8. CLI (`scone`)

Pembangun heartbeat offline (dihapus bersama permukaan `tx`)
dihapus bersama permukaan tx offline (cli-simplify-v1). Relay
memancarkan heartbeat mereka sendiri secara otomatis (§7); emisi
manual/air-gapped, jika suatu saat diperlukan lagi, akan kembali
sebagai perintah khusus. Defaultnya dulu: `tip_seen` = 32 byte
nol (hanya observabilitas), `height_seen` = 0, `timestamp` =
sekarang.
- Seperti setiap tx, `tx sign` **mengikat ulang** operator dan
  owner ke kunci penandatangan NYATA (nilai input tidak pernah
  dipercaya); `tx verify` memvalidasi secara lokal.

## 9. Observabilitas

- `NodeStatus` (pesan p2p wire `0x0d`) membawa
  `fresh_operators: u32` (varint): jumlah operator dengan
  heartbeat segar — subset hidup dari pool yang layak untuk
  komite (operator efektif dari kursi pool, kunci ter-ban
  dikecualikan, terurut/ter-dedupe). Diisi relay dari
  `state().fresh_operators(height)`; ditampilkan `scone status`
  sebagai `fresh operators: N`.
- Argumen kampanye: "mengoperasikan TLD privat = duduk di komite
  finalitas DNS terdesentralisasi" — dapat diverifikasi
  on-chain.
- Permukaan mendatang (belum dispesifikasikan):
  `operator_status` — per kursi, operator yang didelegasikan,
  kesegaran (segar / kedaluwarsa / tak pernah terlihat), `tip_seen`
  terakhir. RPC kontrol lokal dihapus (node p2p-saja);
  permukaannya masih akan didefinisikan.

## 10. Yang TIDAK dilakukan RFC ini

- **Tanpa kehilangan kepemilikan** karena inaktivitas (opsi A —
  peluruhan TLD yang tidak dioperasikan — ditolak secara
  eksplisit).
- **Tanpa reward** (opsi C menunggu lapisan ekonomi).
- Heartbeat tidak membuktikan *kualitas* layanan (blok
  diproduksi, record DHT dilayani) — hanya kehadiran.
  Menghitung kontribusi nyata adalah ekstensi mendatang yang
  mungkin, lebih mudah dimanipulasi.

## 11. Invarian yang dilindungi

- Determinisme bit-eksak: jendela dan pemilihan adalah fungsi
  murni dari state yang diterapkan (terjurnal, undo bit-eksak).
- Tidak pernah panic atas data jaringan: heartbeat atau delegasi
  salah-bentuk → penolakan bertipe (`InvalidOwner`,
  `StaleHeartbeat`, `HeartbeatDuplicate`,
  `OperatorAlreadyDelegated`, `TldIsOpen`, `NotTldOwner`,
  `NotOwner`).
- Kepemilikan tidak bersyarat; hanya penandatanganan checkpoint
  yang bersyarat.

## 12. Konstanta dan parameter (nilai terkode)

| Nama | Nilai | Lokasi |
|---|---|---|
| `HEARTBEAT_WINDOW_EPOCHS` | 2 | `scone-blockchain/src/state.rs` |
| `MAX_SEATS_PER_OPERATOR` | 1 | `scone-blockchain/src/state.rs` |
| `MIN_FINALITY_COMMITTEE_SIZE` | 4 | `scone-core/src/consensus_params.rs` |
| Diskriminan `Heartbeat` | `0x71` | `scone-protocol/src/transaction.rs` |
| Diskriminan `SetTldOperator` | `0xA9` | sama |
| Diskriminan `SetDomainOperator` | `0x5E` | sama |

Nilai-nilai ini bagian dari konsensus: mengubahnya merusak
replayability chain (jendela/pembersihan) atau pemilihan
(komite).

## 13. Rencana implementasi — keadaan akhir

| Langkah | Crate | Konten | Status |
|---|---|---|---|
| 1 | core + protocol | `Heartbeat`, `SetTldOperator`, `SetDomainOperator`, wire, validasi | selesai |
| 2 | blockchain | jendela geser (terjurnal), pool terfilter, pemilihan, ~15 test | selesai |
| 3 | network | emisi otomatis, relay tx | selesai |
| 4 | CLI | `tx build heartbeat`, tampilan `fresh operators` di `scone status` | selesai |
| 5 | docs | RFC ini dikonsolidasikan ke bentuk normatif, cermin fr/en | selesai |

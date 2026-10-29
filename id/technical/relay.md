# Relay jaringan (M4) — arsitektur, alur, dan kendali CLI

> Spesifikasi normatif relay jaringan Scone (`scone-network`).
> Setiap perbedaan dokumen/kode = bug (perbaiki sumber
> kebenarannya).

## Ringkasan

**Relay** adalah node P2P penuh Scone: ia memelihara blockchain
kanonik (RAM + `NodeStore` redb), me-relay transaksi dan blok,
memproduksi blok (mode devnet), melayani protokol sinkronisasi,
mempublikasikan dan me-resolve `SignedDnsRecord` dalam DHT
Kademlia, dan mengekspos **protokol p2p** (`/scone/reqres/1`)
yang ditanyai CLI secara langsung (unifikasi RPC→p2p: kanal TCP
JSON hilang).

```text
                    ┌────────────────────────────────────────┐
                    │              scone relay               │
  P2P (libp2p)      │                                        │
  QUIC (udp,        │  Swarm libp2p                          │
  port ephemeral    │  ├─ identify  (penemuan alamat)        │
  secara default)◄──┤  ├─ ping      (kehidupan)              │
                    │  ├─ kad       (DHT record)             │
                    │  └─ reqres    (pesan scone-prot.)      │
                    │                                        │
                    │  Inti relay (task tunggal)             │
                    │  ├─ Blockchain (validasi penuh)        │
                    │  ├─ Mempool (berbatas, dedup per TxId) │
                    │  ├─ BlockBuilder (produksi devnet)     │
                    │  ├─ Parking fork (berbatas, 2048)      │
                    │  └─ Driver sinkron (pengejaran batch)  │
                    │                                        │
  CLI ◄─────────────┤  reqres p2p (QUIC, pesan yang sama)    │
                    └────────────────┬───────────────────────┘
                                     │
                              NodeStore (redb)
```

`crates/scone-network` adalah **satu-satunya crate async** dalam
workspace: semua di bawahnya (crypto → core → protocol →
blockchain → storage) tetap sinkron dan murni.

## Admisi koneksi (P1.5)

Batas keras di level swarm (perilaku `connection_limits` libp2p,
entri pertama `SconeBehaviour`):

- `MAX_TOTAL_CONNECTIONS = 128` — tutup koneksi terjalin
  (masuk + keluar) per relay; kelebihan masuk DITOLAK,
  tidak pernah mengantre;
- `MAX_PER_PEER_CONNECTIONS = 4` — satu peer tidak bisa
  memonopoli kuota total dengan membuka banyak koneksi;
- `MAX_PENDING_INCOMING = 32` — penjaga slowloris:
  setengah-koneksi tak terautentikasi tidak pernah menunggu tanpa
  batas.

Batas ini melengkapi batas per-pesan yang ada
(`MAX_MESSAGE_LEN` 1 MiB) dan per-anggaran (sinkron `max ≤ 128`
blok, DNS TCP 64 koneksi/512 B/5 dtk). Penyerang banjir-koneksi
mendapat penolakan, bukan pertumbuhan memori.

## Penemuan peer

Tiga mekanisme pelengkap, dalam urutan prioritas ini:

1. **Seed eksplisit** (`--bootstrap`, multiaddr penuh) — selalu
   berprioritas; memberikan daftar tak-kosong MENGGANTIKAN
   segala hal lain. `--no-builtin-seeds` mengosongkan seed
   tertanam untuk deployment 100% privat.
2. **Seed tertanam** (`builtin_seeds(network)` di
   `scone-network/src/config.rs`) — dibekukan pada tiap rilis
   biner resmi, per jaringan. Relay tak terkonfigurasi bergabung
   dengan jaringan sendirian. (Kosong di testnet selama testnet
   publik belum dideploy.)
3. **Direktori terpelajari** (`<data-dir>/peers.json`) — setiap
   alamat yang dipelajari via `identify` disimpan (terakhir
   dilihat; tutup **2000 entri** — alamat, bukan koneksi, ~400
   KiB RAM pada maksimum — dan **pemotongan umur 90 hari**: entri
   yang terakhir dilihat lebih dari 90 hari lalu dijatuhkan saat
   tulis dan diabaikan saat muat, menjaga 2000 slot terisi peer
   yang masuk akal alih-alih fosil pra-churn yang
   satu-satunya kontribusinya adalah timeout dial saat boot) dan
   disimpan ~1×/menit pada tick produksi. Saat boot tanpa seed
   dan tanpa bootstrap, direktori direkomposisi: relay menemukan
   peer biasanya tanpa konfigurasi. Ini cache: berkas
   hilang/rusak = direktori kosong, bukan error.
4. **UPnP** (`--upnp`, opt-in) — pemetaan port listen QUIC oleh
   router rumah: node menjadi dapat dijangkau dari luar TANPA
   circuit relay. Pada `NewExternalAddr`, alamat eksternal
   didaftarkan sebagai alamat eksternal nyata swarm
   (`add_external_address` — dilihat identify/autonat, kandidat
   transport-relay). Tanpa gateway IGD (`GatewayNotFound`) atau
   gateway tak-dirutekan (`NonRoutableGateway`): log info
   polos, pemetaan dilewati — kasus normal di LAN rumah.
   Pemetaan kedaluwarsa (pembaruan gagal) adalah peringatan.

UPnP tetap **nonaktif secara default** (`enable_upnp` = `false`
di `config.rs`): deployment publik tidak berbicara dengan
router tanpa persetujuan. **Penemuan mDNS telah dihapus**
(remove-mdns-discovery, 2026-10-22): peering datang secara
eksklusif dari bootstrap eksplisit, seed bawaan dan Kademlia —
flag lama `--mdns`/`SCONE_MDNS` masih diterima tanpa efek
(kompatibilitas skrip, perlakuan sama seperti `--rpc`).

### Optimasi topologi lokal (selalu aktif)

Relay aktif mengoptimalkan lingkungannya memakai engine murni
`scone-topology` (lihat docs crate). **Selalu aktif**; satu-
satunya penalaan terekspos adalah **profil**
(`--topology-profile`, env `SCONE_TOPOLOGY_PROFILE`, default
`default`):

| Profil | min / target / maks | eval. | untuk siapa |
|---|---|---|---|
| `tiny` | 2 / 4 / 6 | 120 dtk | Raspberry Pi, kotak rumah, VPS 1-vCPU |
| `small` | 4 / 8 / 12 | 90 dtk | server kecil, NAS rumah |
| `default` | 8 / 16 / 32 | 60 dtk | server standar (default) |
| `high` | 16 / 32 / 64 | 45 dtk | node berdaulat, backbone |
| `very-high` | 32 / 64 / 128 | 30 dtk | hub pendiri |
| `devtest` | 2 / 4 / 6 | 5 dtk | **HANYA LAB/TEST** (tidak stabil menurut desain) |

```
scone relay --topology-profile tiny
```

Parameter pelindung (histeresis 0,15, cooldown 600 dtk, grace
120 dtk, pengukuran 15 dtk) identik di mana-mana: mereka
menjaga stabilitas dan tak ada alasan menskalakan dengan ukuran
node. Nama tak dikenal adalah error startup bertipe.

Konfigurasi inkonsisten (`min > target > maks`, ambang di luar
`[0,1]` — `NaN` termasuk —, interval nol) adalah **error startup
bertipe**, bukan perilaku diam.

**View (`TopologyView`)** — diberi makan oleh event swarm:

- `ping` berhasil → observasi RTT (median jendela 8 sampel);
- `ping` gagal → penalti keandalan;
- `identify` diterima → alamat (≤ 4/peer) + protokol
  yang diumumkan (bitmask kad/reqres/identify/relay);
- koneksi terjalin → state terhubung, arah
  (masuk/keluar), umur, circuit-relay terdeteksi;
- koneksi tertutup → penalti + penghitung churn HANYA jika tanpa
  permintaan (pengusiran sukarela tidak dihitung);
- kegagalan dial keluar → penalti keandalan.

View berbatas (500 peer; melampaui itu, hanya peer terhubung
yang diingat).

**`kad_required` dari kbuckets** — sebelum tiap evaluasi, relay
menyusuri tabel routing Kademlia: peer dilindungi jika `Connected`
dalam sebuah bucket, atau `Disconnected` dalam bucket
JENUH (k = 20). Peer terlindungi TIDAK PERNAH dipilih untuk
pengusiran.

**Loop optimasi** — pada interval evaluasi profil (60 dtk di
bawah `default`; lihat tabel atas):

1. hitung ulang `kad_required` (kbuckets);
2. evaluasi engine (histeresis: masa grace 120 dtk, perbaikan
   skor minimum +15%, cooldown pengusiran 600 dtk);
3. keputusan diterapkan: `AddCandidate` → dial kandidat;
   `Replace` → tutup peer yang diusir (SEKALI — progresif, tanpa
   perulangan) lalu dial penggantinya;
4. snapshot **metrik** dicatat (level INFO): jumlah
   dikenal/terhubung/masuk/keluar/circuit/terlindung, RTT
   rata-rata/median, bucket IP berbeda, konsentrasi maks,
   churn (penggantian/pemutusan).

Skor lingkungan TIDAK PERNAH ukuran kepercayaan (lihat docs
`scone-topology`): tidak ada keputusan konsensus yang
diturunkan darinya.

### Jangkauan (AutoNAT) dan relay transport (circuit-relay v2)

Setiap relay menjalankan AutoNAT (klien + server): node meminta
peer-nya mencoba dial MUNDUR kepadanya; vonis (`reachable`: ya =
dapat dijangkau dari luar, tidak = di belakang NAT, tak
diketahui = belum teruji) disegarkan tiap 5 menit dan terekspos
dalam `NodeStatus` (field `nat_reachable`, varint wire 0/1/2) —
CLI menampilkannya (`scone status`). Alamat privat dihitung
(`only_global_ips = false`): mesh LAN menguji jangkauan LAN-nya.

Relay yang dapat dijangkau mengaktifkan circuit-relay v2 **hop**
(ia bisa me-relay lalu lintas bagi peer ter-NAT) dan **DCUtR**
(hole-punching: dua node terhubung via circuit mencoba koneksi
langsung dan melepaskan relay). Relay yang divonis ter-NAT
otomatis meminta reservasi dari peer terjangkau pertama yang
dikenal dan mendengarkan melaluinya (alamat
`/p2p/<relay>/p2p-circuit`).

Aturan produksi: node ter-NAT yang di-relay sebaiknya tidak
menjadi produsen blok (latensi/ketersediaan terdegradasi) —
aturan PoS (kursi = domain hidup) belum melarangnya; akan
didokumentasikan sebagai persyaratan operasional sebelum testnet
publik.

## Transport dan protokol libp2p

| Protokol | Nama | Peran |
|---|---|---|
| QUIC (quic-v1) | — | Transport utama (udp) |
| identify | `/scone/id/1` | Penemuan alamat peer (mengisi tabel Kademlia) |
| ping | standar libp2p | Kehidupan |
| Kademlia | `/scone/kad/1` | DHT: kunci = `DomainId` (32 B), nilai = enkode kanonik `SignedDnsRecord` |
| request-response | `/scone/reqres/1` | pesan aplikasi scone-protocol |

### Frame aplikasi (`/scone/reqres/1`)

Setiap frame adalah `len: u32 BE || byte` di mana byte adalah
enkode kanonik sebuah `Message` dari `scone-protocol` (kodek
yang ada — tanpa format kedua). Panjang dibatasi
`limits::MAX_MESSAGE_LEN` **sebelum** baca atau alokasi apa pun:
peer yang mengumumkan frame lebih besar ditolak tanpa biaya
memori.

Pesan yang dipertukarkan (satu per putaran
permintaan/respons):

| Pesan | Arah | Balasan |
|---|---|---|
| `Hello { version }` | handshake | `Hello` (versi lokal) |
| `Ping` / `Pong` | kehidupan | `Pong` |
| `GetBlock { hash }` | kueri | `Block` (jendela RAM) atau `Pong` jika tak dikenal |
| `GetBlocks { start, max }` | sinkron | **satu** `Block` (tinggi `start`), atau `Pong` jika tidak ada |
| `Block (b)` | relay blok | `Pong` (ack) |
| `Transaction (tx)` | relay tx | `Pong` (ack) |
| `GetRecord { domain_id }` | DHT lokal | `Record` (cache lokal) atau `Pong` |
| `Record (r)` | publikasi | `Pong` jika diterima, error jika tidak |

Aturan: **tepat satu balasan per permintaan**; pesan
pengakuan adalah `Pong`. Balasan sinkron membawa satu blok pada
satu waktu — pengejaran beriterasi di sisi peminta (lihat
"Sinkron"). Ini sengaja berbatas: tidak pernah lebih dari satu
blok in flight per peer.

## Perilaku relay

### Startup

0. resolusi jaringan (`--network testnet|mainnet|devnet`,
   default testnet selama proyek dalam dev — M8b). Data-dir
   **per jaringan** (`~/.scone/<network>/`): store tak-kosong
   yang memegang chain dari jaringan lain ditolak
   (`Corrupted`);
1. buka/buat store redb (`<data-dir>/chain.redb`);
2. `load_chain(store, network)`: tip O(1) + state domain
   berhalaman (jendela RAM: genesis + tip);
3. konstruksi swarm (identitas Ed25519 ephemeral per-sesi —
   mempersist identitas jaringan adalah keputusan terbuka);
4. susun daftar boot (lihat [Penemuan peer](#penemuan-peer)):
   `--bootstrap` eksplisit (harus berakhir `/p2p/<peer-id>`)
   ATAU seed tertanam jaringan ATAU direktori terpelajari
   (`peers.json`);
5. dial tiap multiaddr dan daftarkan dalam tabel Kademlia
   (dial mati diabaikan — yang lain membentuk jaringan).

### Menerima transaksi (pengiriman P2P atau CLI `submit tx`)

1. dekode ketat berbatas (`MAX_MESSAGE_LEN`);
2. validasi kriptografis penuh (`validate_transaction`: ikatan
   owner/kunci, `verify_strict` atas payload kanonik);
3. **anti-replay DULU** (murah, tak bergantung state, definitif),
   lalu pre-check state (cermin murah dari aturan `push_block` —
   PoW, yang mahal, hanya diverifikasi saat push): `RenewDomain`
   yang di-replay menjawab `TxReplay`, bukan error PoW yang
   berbohong (buktinya mengikat kedaluwarsa yang dilewati
   penerapan pertama). Pemisahan jaringan ditegakkan pada
   langkah 2: payload tanda tangan menyuntikkan jaringan RELAY,
   tx yang ditandatangani di tempat lain gagal dengan
   `InvalidSignature` (wire 2026-09: tidak ada lagi field
   `network`); `RegisterDomain` → TLD dari nama yang
   didaftarkan (`UnknownTld` jika tidak, D1/M7c) **dan terbuka**
   (`TldClosed` jika tidak, M8b) lalu nama bebas; `UpdateDomain`
   → domain dikenal, sequence eksak `current + 1`, owner;
   `RegisterTld` → TLD bebas; `TransferTld`/`RevokeTld`/
   `SetTldOpen` → TLD dikenal + penandatangan owner;
   `AssignDomain` → owner TLD + nama bebas; `RenewTld` → TLD
   privat dikenal + owner, masa tetap diturunkan ulang,
   pre-check jendela masa tenggang; `RenewDomain` → domain
   dikenal + owner, pre-check PoW perpanjangan (terikat pada
   kedaluwarsa saat ini) pada tinggi berikutnya;
4. penyisipan mempool (kapasitas tetap, dedup per `TxId` —
   mengirim ulang tx yang sudah di pool adalah no-op);
5. broadcast ke peer (kecuali pengirim).

### Menerima blok (P2P, sinkron, atau produksi sendiri)

1. `push_block`: validasi penuh yang ada (induk = tip kanonik,
   tinggi, akar Merkle dihitung ulang, setiap transaksi
   divalidasi lalu diterapkan — atomik per blok);
2. jika diterima: relay membaca hasil penerapan
   (`push_block_with_gc` → hash + domain yang dihapus oleh GC
   deterministik) dan mempersist via `store_block_with_removals`
   (append delta atomik: blok + tip + state yang berubah —
   upsert DAN penghapusan: domain kedaluwarsa, TLD dicabut).
   Restart karena itu tidak pernah bisa membangkitkan
   registrasi kedaluwarsa;
3. rekonsiliasi mempool (tx yang terinklusi dihapus);
4. broadcast ke peer (kecuali pengirim).

Blok tidak valid ditolak dengan error bertipe; relay tetap
berjalan. Tanpa panic atas data jaringan. Penerapan berat
(Argon2 + komit redb) lewat `tokio::task::block_in_place`
(driver sinkron, accept p2p, produksi): runtime memindahkan task
siap ke pekerja lain — menerapkan blok tidak lagi menghentikan
layanan DNS atau metrik.

### Snapshot DNS (publikasi)

Setelah SETIAP blok yang diterapkan (accept p2p — reorg
termasuk —, driver sinkron, produksi) DAN saat boot (restart
tidak boleh NXDOMAIN nama hidup sampai blok berikutnya), relay
mempublikasikan **snapshot DNS otoritatif** ke umpan `watch`
yang dibaca permukaan DNS (spesifikasi lengkap di `dns.md`):

- perpanjangan tip (jalur yang menyaingi segalanya): **delta
  O(yang tersentuh)**, copy-on-write — transaksi blok
  menyebut domain yang tersentuh, ditambah penghapusan GC
  deterministik (`AppliedBlock.gc_removed_domains`,
  dievaluasi saat push);
- reorg / boot: **pembangunan ulang penuh** (satu walk
  berhalaman `ChainState::dns_facts`).

### Produksi blok (M5: produsen bertanda tangan)

Setiap `produce_interval` detik (2 dtk secara default): jika
mempool tak kosong, blok dirakit oleh `BlockBuilder`
(`timestamp = now`) **ditandatangani oleh produsen
terotorisasi** (payload `SCONE-BLOCK-V1` dalam
`header.consensus`, lihat `/docs/en/technical/blockchain.md`)
dan mengikuti tepat jalur penerimaan blok yang diterima. Relay
tanpa tx tidak memproduksi apa pun.

Kunci produksi: kunci anchor (`--anchor-key`) saat relay
memilikinya — anchor adalah produsen terotorisasi — jika tidak
kunci devnet tetap, valid selama kumpulan produsen kosong
(bootstrap terbuka: genesis). Begitu domain/TLD hidup ada, blok
yang ditandatangani kunci devnet ditolak (`InvalidProducer`).

### Loop anchor (finalitas, M5)

Dengan `--anchor-key` (keyfile `.sconekey`, passphrase via
`--anchor-passphrase-env`, default `SCONE_ANCHOR_PASS` — default
pustaka `scone-network` `SCONE_ANCHOR_PASSPHRASE` tidak pernah
tercapai melalui biner, clap selalu menimpanya):

- **rotasi proposer**: tepat SATU anggota komite yang ditunjuk
  mengusulkan dirinya sendiri per epoch
  (`committee[epoch % size]`, deterministik); anggota lain tidak
  pernah menandatangani tip milik mereka sendiri — konten sebuah
  checkpoint bergantung pada tip proposer, dan usulan-diri tanpa
  gerbang pernah memecah epoch menjadi konten-konten saingan yang
  masing-masing ditolak dengan benar oleh guard sebagai
  ekuivokasi (testnet, 2026-10-21);
- setelah tiap blok diterima/diproduksi, jika node ini anggota
  komite saat ini **dan** komite mencapai
  `MIN_FINALITY_COMMITTEE_SIZE`, relay menandatangani checkpoint
  yang diusulkan via `SignerGuard`-nya (`signer-state.dat` dalam
  data-dir — satu kunci = satu suara/epoch, tahan-crash
  fail-closed) dan menggossipkannya;
- **vote-on-receipt**: usulan yang kontennya valid di chain KITA
  (blok yang dirujuk ada, pada hash yang sama) dan membawa tanda
  tangan anggota komite mendapat tanda tangan KITA juga — tanpa
  itu, setiap anchor hanya menandatangani konten tip-nya sendiri
  dan kuorum tidak pernah terkumpul;
- tanda tangan yang diterima (`Message::Checkpoint`)
  terakumulasi per hash `CheckpointData` (berbatas 256 entri);
  pada tanda tangan `quorum_for(ukuran komite)` anggota atas
  data yang SAMA → `accept_checkpoint` + broadcast checkpoint
  terfinalisasi;
- **persistensi finalitas**: jendela checkpoint dan basis beku
  ditulis ke store pada setiap finalisasi dan dimuat ulang saat
  boot. Checkpoint adalah state GOSSIP (tidak pernah dibawa blok)
  — restart tanpa persistensi ini mulai dari basis kosong dan tak
  pernah bisa lagi merangkai jendela yang dilayani peer (anchor
  epoch di bawah jendela jaringan sudah hilang);
- **catch-up finalitas**: relay yang basisnya kosong (boot node
  baru) meminta `GetCheckpoints` ke sebuah peer pada setiap tick
  sampai basisnya pulih;
- relay tanpa kunci anchor tetap **pasif**: ia mengakumulasi,
  memfinalisasi, dan me-relay, tetapi tidak pernah
  menandatangani;
- `GetCheckpoints` → jendela `checkpoint_window()` dilayani;
- error loop apa pun dicatat dan ditelan — tidak pernah fatal;
- status (`GetStatus`) mengekspos `checkpoints` dan
  `finalized_epoch`.

### Parking fork dan reorg (perbaikan minoritas-lengket)

Blok saudari pada tinggi yang sama, atau blok yang induknya
bukan tip saat ini (`ParentNotTip`/`UnknownParent`), TIDAK
PERNAH hilang: ia **diparkir** (map berbatas, `PARKED_CAP =
2048` blok, ~KiB masing-masing — RAM kasus-terburuk berbatas;
parking penuh diam-diam mengabaikan pendatang). Hash yang sudah
ditawarkan ke `accept_block` di-dedupe (`seen_blocks`): blok
yang kalah dan di-gossip ulang tidak dievaluasi ulang.

Pada SETIAP tick `Produce`, `reconsider_parked`:

1. pengurasan linear blok parkir yang induknya menjadi tip;
2. jika cabang parkir lebih kuat dari tip saat ini (aturan
   fork-choice TAK BERUBAH: lantai finalitas, rantai terpanjang,
   hash terendah), adopsi — akhiran yang tergeser diparkir
   sendiri (KEDUA sisi fork tetap dapat dievaluasi).

Ini melepas node yang mengadopsi sisi yang kalah dari fork
tinggi-sama: blok pemenang, ditawarkan sekali lalu ditolak,
dievaluasi ulang berkala alih-alih dilupakan selamanya (temuan
chaos-30: dua blok saudari 15 ms berselang).

**Persistensi reorg**: mengadopsi cabang lain tidak bisa lewat
append monotonik (`NonMonotonicHeight`). Relay menerapkan
`rewind_to_height(titik fork)` lalu `persist_state_snapshot`
(penulisan ulang state hidup dari ekspor chain) lalu re-append
monotonik akhiran pemenang (lihat `storage.md`). Kegagalan
persistensi dicatat dan didegradasi (sembuh saat boot via
replay) — tidak pernah fatal.

## Sinkron (pengejaran)

Pada setiap koneksi ke peer, relay meminta
`GetBlocks(start = tinggi_lokal + 1)`. Setiap `Block` yang
diterima yang tepat melengkapi tinggi yang diharapkan memicu
permintaan tinggi berikutnya, sampai habis (`Pong` = tidak ada
lagi) atau `MAX_SYNC_ROUNDS` (10.000) batch — batas absolut
bahkan melawan peer yang berbohong. Blok yang diterima melewati
validasi penuh: peer berbahaya tidak bisa membuat apa pun yang
tidak valid diterima.

### Record DHT

- **put** (alur API/web — kanal heks CLI dihapus): dekode ketat
  berbatas (`MAX_DHT_CACHE_ENTRY`), verifikasi terhadap state
  on-chain (owner, sequence, `record_hash` dihitung ulang —
  lihat "Verifikasi record"), lalu `kad.put_record`
  (penyimpanan lokal + replikasi ke peer terdekat,
  `Quorum::Majority` dari faktor replikasi) dan cache redb
  persisten.
- **get** (`domain resolve`): `kad.get_record`; penunggu
  terikat pada `QueryId` dan `DomainId` yang diminta — balasan
  hanya me-resolve kueri yang cocok, dan kunci record harus
  tepat domain yang diminta (diperiksa **sebelum** verifikasi
  on-chain). Hasilnya lalu diverifikasi terhadap chain sebelum
  dikembalikan. Tutup permintaan simultan: `MAX_DHT_WAITERS`
  (256); melampaui itu, error segera. Resolusi yang berhasil
  diverifikasi ditulis ke cache persisten (M5): baca lokal
  (`domain_info`) lalu melayaninya tanpa kueri DHT baru.
- Kademlia berjalan dalam mode server, tanpa TTL record (chain
  yang mengatur validitas, bukan waktu).

### Verifikasi record (aturan "chain adalah otoritas")

`SignedDnsRecord` yang di-resolve atau diterima hanya
diterima/dikembalikan jika:

1. domain terdaftar on-chain;
2. `record.owner` = owner on-chain saat ini;
3. `record.sequence` = sequence on-chain saat ini;
4. `record_hash(record.record)` (BLAKE3 dihitung ulang) =
   `record_hash` yang terkomit on-chain.

Jika tidak: penolakan diam di sisi P2P, error eksplisit di sisi
CLI. Tanda tangan Ed25519 record **diverifikasi saat diterima**
oleh relay (`ResolveLocal`): tanda tangan harus terbuka terhadap
kunci publik owner yang dipelajari ATAU salah satu kunci delegasi
record domain (KSK/ZSK-lite — delegasi menerbitkan atas nama
owner; record yang ditandatangani kunci tak terkait ditolak,
`record rejected`); ia juga diangkut bagi
verifier mendatang — komitmen on-chain tetap kendali integritas
utama.

## Metrik Prometheus

Endpoint HTTP (format teks Prometheus 0.0.4, eksposisi buatan
sendiri — tanpa dependensi baru) yang dilayani task khusus:

- **Nonaktif secara default**: tidak ada server HTTP tanpa `--metrics-addr` (metrik opt-in).
- `--metrics-addr ADDR:PORT`: mengikat endpoint ke alamat tersebut (mis. `127.0.0.1:9101` lokal, `0.0.0.0:9101` untuk diekspos).
- `--metrics-token TOKEN`: beralih ke mode **privat** —
  `Authorization: Bearer *** wajib (perbandingan waktu-konstan,
  `401` jika tidak; token tidak pernah muncul di log maupun
  eksposisi).
- Kegagalan bind **menonaktifkan endpoint** (peringatan non
  fatal) — bukan nodenya.

Bind efektif diumumkan di stdout (kontrak
dapat-dibaca-mesin, seperti `p2p:`): baris `metrics: <addr>`
begitu mendengarkan.

**19 penghitung** (`counter`, monotonik — hot path:
`fetch_add(Relaxed)` pada `Arc<Metrics>` bersama, tidak pernah
masukan konsensus):

| Grup | Seri |
|---|---|
| Chain | `scone_blocks_applied_total`, `scone_blocks_rejected_total`, `scone_checkpoints_signed_total`, `scone_checkpoint_sigs_received_total` |
| Mempool | `scone_txs_admitted_total`, `scone_txs_rejected_total`, `scone_txs_evicted_grace_total`, `scone_txs_included_total` |
| DNS | `scone_dns_queries_total`, `scone_dns_cache_hits_total`, `scone_dns_cache_misses_total`, `scone_dns_local_chain_total`, `scone_dns_dht_answers_total`, `scone_dns_rejected_unverified_total` |
| DHT | `scone_dht_pulls_issued_total`, `scone_dht_negative_hits_total`, `scone_dht_negative_expired_total`, `scone_dht_negative_capped_total`, `scone_dht_cache_evictions_total` |
| Jaringan peer | `scone_peer_connections_established_total`, `scone_peer_connections_closed_total`, `scone_peer_connections_incoming_total`, `scone_peer_dial_failures_total`, `scone_peer_blocks_received_total`, `scone_peer_transactions_received_total` |

**6 gauge** (`scone_height`, `scone_mempool`, `scone_peers`,
`scone_domains`, `scone_finalized_epoch`,
`scone_addr_book_entries`) — lima pertama disegarkan tiap 15 dtk
via kanal `RelayCommand` (jalur yang sama dengan `status` CLI —
satu sumber kebenaran); `scone_addr_book_entries` dipelihara
langsung oleh registry (diperbarui pada tiap penyisipan buku
alamat, sumber kebenaran).

Test E2E: `metrics_endpoint_public_and_private` (200 +
eksposisi, 401 tanpa/dengan token salah, 200 dengan Bearer).

## Kendali CLI (unifikasi RPC→p2p)

Server RPC TCP JSON telah **dihapus**: CLI (dan jembatan axum
registrar) berbicara protokol p2p **secara langsung** — wire
yang sama (`/scone/reqres/1`), handler yang sama di sisi relay,
nol kanal paralel. Setiap perintah membuka koneksi QUIC ke
multiaddr node (`--peer`), bertukar SATU `Message`, menutup.

Pesan kendali yang dilayani relay:

- `GetStatus` → `NodeStatus { network, nat_reachable, tip,
  height, domain_count, mempool, peers, finalized_epoch,
  checkpoints, fresh_operators }` (ditampilkan `scone status` —
  `fresh_operators` = operator dengan heartbeat segar, RFC
  proof-of-service);
- `LookupName { name }` → `NameState` (state on-chain, seperti
  `lookup`);
- `GetOwnedNames { owner }` → `OwnedNames` (domain hidup + TLD
  dari sebuah `OwnerId`);
- `Transaction (tx)` → pengiriman dengan gossip gratis (relay
  menyiarkan ulang ke peer);
- `GetRecord { domain_id }` / `Record (r)` — resolusi dan
  publikasi DHT (seperti `scone domain resolve` di sisi CLI);
- penolakan aplikasi → `Message::Error(msg)` (mencapai klien
  sebagai pesan terbaca, bukan EOF).

Klien membangkitkan swarm libp2p minimal (request-response +
identify), timeout panggilan 30 dtk (20 dtk putaran). Default
`--peer`:
`/ip4/127.0.0.1/udp/4001/quic-v1/p2p/12D3KooW…` (node dev
lokal; multiaddr penuh dengan peer id nyata diumumkan relay di
stdout melalui baris `p2p:`).

Kanal internal `RelayCommand` bertahan untuk permukaan DNS
tertanam (`ResolveLocal`, satu putaran dalam-proses, bukan
penunggu DHT — lihat `dns.md`).

## CLI

```
scone relay [--network testnet|mainnet|devnet] [--data-dir DIR] [--listen MULTIADDR]
            [--bootstrap MULTIADDR]... [--upnp]
            [--anchor-key PATH] [--no-anchor-auto] [--anchor-passphrase-env VAR] [--genesis FILE]
            [--topology-profile tiny|small|default|high|very-high|devtest]
            [--dns ADDR] [--dns-upstream ADDR]...
            [--metrics-addr ADDR:PORT] [--metrics-token TOKEN]
scone status [--peer MULTIADDR]
scone domains [--identity ID | --pubkey HEX] [--peer MULTIADDR]
scone domain register NAME [--identity ID] [--dir DIR] [--passphrase-env VAR] [--peer MULTIADDR]
scone domain set NAME TYPE VALUE [TTL] [--identity ID] [--peer MULTIADDR]   # record tunggal ekspres
scone domain update NAME --file FILE [--identity ID] [--peer MULTIADDR]
scone domain get NAME [--peer MULTIADDR]
scone domain renew NAME [--identity ID] [--peer MULTIADDR]
scone domain resolve NAME [--peer MULTIADDR]

# --peer default ke relay lokal (env SCONE_PEER menimpa);
# --identity default ke identitas tunggal keystore saat tepat
# satu. Dihapus (stub menyarankan pengganti): show,
# submit, lookup, owned.
```

Setiap perintah kendali berbicara QUIC langsung ke multiaddr
relay (`--peer`, default node lokal — unifikasi RPC→p2p, tanpa
kanal TCP JSON lagi). Relay absen → pesan jelas "p2p node
unreachable — is a relay running?", keluar non-nol. Flag `--rpc`
dipertahankan tetapi **usang dan diabaikan** (server RPC
dihapus; kompatibilitas skrip saja).

### Konfigurasi lingkungan (`.env`)

Saat startup, `scone` memuat (via dotenvy, sebelum parsing clap)
berkas `.env` dari direktori saat ini: berkas hilang = no-op
diam, bukan error. Nilai yang dimuat menjadi **baseline
lingkungan** opsi yang mendeklarasikan variabel
(`env = "…"` di `cli.rs`); **flag CLI eksplisit selalu menang**
atas `.env`, dan `.env` menang atas default bawaan:

```text
flag CLI eksplisit  >  variabel .env  >  default bawaan
```

Berkas `.env.example` di akar repositori mendokumentasikan
setiap variabel (salin ke `.env` dan sesuaikan; `.env`
**di-gitignore** — jangan pernah commit `.env` nyata). Variabel:

| Variabel | Flag cermin | Semantik |
|---|---|---|
| `SCONE_NETWORK` | `--network` | `testnet` (default), `mainnet`, `devnet` |
| `SCONE_DATA_DIR` | `--data-dir` | direktori data (default `~/.scone/<network>/`) |
| `SCONE_LISTEN` | `--listen` | multiaddr listen P2P (default: semua antarmuka, port ephemeral) |
| `SCONE_BOOTSTRAP` | `--bootstrap` | seed bootstrap, **dapat diulang** (satu baris per nilai — dotenvy mengakumulasi); menggantikan seed bawaan |
| `SCONE_GENESIS` | `--genesis` | berkas genesis (node pendiri) |
| `SCONE_MDNS` | `--mdns` | (usang — penemuan mDNS dihapus; diterima, diabaikan) |
| `SCONE_UPNP` | `--upnp` | pemetaan port UPnP (`true` = aktif) |
| `SCONE_DNS_ADDR` | `--dns` | alamat bind server DNS UDP |
| `SCONE_DNS_UPSTREAM` | `--dns-upstream` | fallback rekursif, **dapat diulang** |
| `SCONE_DNS_CACHE_LIMIT` | `--dns-cache-limit` | tutup entri cache DNS |
| `SCONE_DNS_CACHE_MAX_MEM` | `--dns-cache-max-mem` | tutup memori cache DNS (mis. `64MB`) |
| `SCONE_DNS_MAX_INFLIGHT` | `--dns-max-inflight` | tutup kueri DNS bersamaan |
| `SCONE_METRICS_ADDR` | `--metrics-addr` | metrics endpoint address (`ADDR:PORT`)|
| `SCONE_METRICS_TOKEN` | `--metrics-token` | token bearer endpoint metrik |
| `SCONE_TOPOLOGY_PROFILE` | `--topology-profile` | profil topologi (`tiny`/`small`/`default`/`high`/`very-high`/`devtest`; default `default`) |
| `SCONE_ANCHOR_KEY` | `--anchor-key` | keyfile anchor (checkpoint + produksi) |

`SCONE_ANCHOR_PASS` BUKAN opsi sebuah variabel: ia NAMA DEFAULT
**variabel yang memegang passphrase anchor**
(`--anchor-passphrase-env`, default `SCONE_ANCHOR_PASS` di level
CLI) — relay membaca passphrase dari variabel yang dinamai flag
ini.

**Anchor otomatis (default AKTIF)**: tanpa `--anchor-key`,
keyfile anchor dibuat/dipersist otomatis dalam direktori data
(`anchor.sconekey`, passphrase `SCONE_ANCHOR_PASS` jika
didefinisikan, jika tidak dibuat acak dan ditulis ke
`anchor.pass`) — relay menjadi produsen begitu kuncinya memiliki
domain/TLD hidup (kelayakan PoS tidak pernah gratis).
`--no-anchor-auto` menjaga relay ketat pasif. Variabel env
default `--anchor-passphrase-env` adalah `SCONE_ANCHOR_PASS`.

`--network`: default mengikuti BUILD (mainnet pada biner
mainnet-aktif, testnet jika tidak); `devnet` adalah jaringan
lokal yang dapat di-reset (genesis `scone-devnet` sendiri, PoW
0..=2 bit mendekati nol — menghapus `~/.scone/devnet/` dan
menjalankan ulang me-reset chain). Menyetel genesis, kesulitan
PoW, dan data-dir default (`~/.scone/<network>/`); tx dari
jaringan lain ditolak secara bertipe (`WrongNetwork`).

Permukaan heks offline (subperintah `tx` dan `submit`) hilang
(remove-tx-surface-v2). CLI mencakup jalur pengguna
(`domain register`, `domain renew`, `domains`) DAN keluarga M8b
sebagai perintah kelas satu: `tld open`, `tld transfer`,
`tld revoke`, `domain assign`, `domain transfer`. Klaim
`register` menambang PoW-nya pada kesulitan jaringan
(milidetik di devnet).

**Tidak ada opsi geo maupun kesehatan**: geozone
**diwajibkan oleh build** (tabel DB-IP Lite tertanam — semantik
lengkap di
[`dns.md`](dns.md#geodns-geographic-routing)); routing
geografis **selalu aktif** tanpa opsi apa pun, dan resolver
tidak melakukan pemeriksaan kesehatan — ia melayani himpunan
rekam bertanda tangan apa adanya (ketersediaan endpoint adalah
urusan operator endpoint). Flag `--geo-*`/`--health-*` apa pun
adalah penolakan clap tegas.

### `domain register|update` (M5) — transaksi bertanda tangan dalam satu perintah

Perintah ini merantai bangun → tanda tangan (keystore) →
kirim → (konfirmasi) dalam satu invokasi.

- **register**: membangun `RegisterDomain` bertanda tangan
  (timestamp = sekarang, proof kosong di devnet), mengirimnya,
  menunggu konfirmasi on-chain (sequence ≥ 0, timeout 30 dtk).
  Nama yang sudah terdaftar ditolak di sisi klien DAN sisi
  relay.
- **update**: berkas record adalah satu-satunya sumber
  kebenaran. Sequence (on-chain + 1) dan `record_hash` (BLAKE3
  kanonik dari record berkas) **diturunkan**, tidak pernah
  dimasukkan: penandatangan dan verifier tidak bisa menyimpang.
  Setelah konfirmasi UpdateDomain, `SignedDnsRecord` (record
  dari berkas + owner + tanda tangan Ed25519 atas enkode
  kanonik) dipublikasikan ke DHT via `put_record` (yang
  memverifikasi ulang terhadap chain). Domain tak terdaftar,
  identitas bukan-owner, atau error berkas gagal bersih, tanpa
  menyentuh chain.

#### Berkas record

Teks, satu record per baris, `#` = komentar, baris kosong
diabaikan; enkode kanonik terurut membuat urutan berkas tanpa
efek pada hash yang dikomitkan. Berkas adalah himpunan LENGKAP
(update menggantikan, tidak menggabungkan). Tipe:

```text
A 192.0.2.1
AAAA 2001:db8::1
CNAME www.example.uip
NS ns1.example.uip
MX 10 mail.example.uip
TXT free text (spasi dinormalisasi)
```

Error bertipe: tipe tak dikenal, IP/nama tidak valid (dengan
nomor baris), himpunan kosong, duplikat, >
`MAX_RECORDS_PER_SET`.

## `--genesis` (pendiri)

`scone relay --genesis genesis.bin` — **hanya node pendiri**
(kampanye kickstarter; berkas diproduksi biner
`scone-genesis`). Aturan normatif:

1. Berkas ditolak **sebelum dibaca** jika melebihi 1 MiB, lalu
   didekode ketat (`decode_complete`: tanpa byte tersisa). Hash
   blok 0 **dihitung ulang**, tidak pernah dibaca dari berkas.
2. Validasi semantik: tinggi 0, `consensus` diawali id jaringan
   dari jaringan yang diminta (`--network`), tanpa transaksi,
   manifest alokasi kanonik. Setiap pelanggaran = error bertipe
   saat startup (tanpa panic, daemon tidak berjalan).
3. Data-dir **perawan**: chain lahir pada genesis teralokasi —
   alokasi diterapkan di RAM **sebelum tulisan disk apa pun**,
   lalu dipersist (state ter-seed + blok genesis di bawah kunci
   meta). Blok pertama yang diproduksi karena itu hanya bisa
   memperpanjang genesis teralokasi.
4. Data-dir **sudah produktif**: genesis lahir store adalah
   `prev_hash` blok 1 (serifikasi-diri). Hash berkas sama = OK
   diam (opsi **redundan setelah startup pertama}); berbeda =
   penolakan bertipe — node bergabung ke jaringan via
   sinkronisasi, tidak pernah via `--genesis`.
5. Restart **tanpa** opsi: data-dir kelahiran-pendiri
   memulihkan genesis teralokasinya via kunci meta (rebind RAM
   di setiap boot).

Pin hash genesis konstan (anti divergensi jaringan) adalah alur
kerja terpisah.

### Manifest biner `scone-genesis` (kickstarter)

Biner `scone-genesis` (crate `scone-genesis`) menyiapkan
manifest `genesis-manifest.json` (format `ALLOC-V1`) lalu
memproduksi `genesis.bin`. Aturan normatif:

- `scone-genesis new --network <network>` membuat manifest
  kosong; `add`/`remove`/`list` mengelola alokasi, `build`
  memproduksi `genesis.bin` (+ hash), `verify` memvalidasi
  ulang.
- `add domain <name>` mensyaratkan **TLD domain sudah
  teralokasi dalam manifest** (penolakan segera jika tidak —
  chain akan menolaknya saat penerapan genesis);
  `valid_until`-nya **otomatis**: `born_at + 1 tahun`
  (default_validity manifest), tidak pernah dimasukkan manual.
- `add tld <name>`: **terbuka secara default** (milik tak
  seorang pun, mandiri, lantai 1 tahun, owner = nol di wire —
  kind `0x03`); `--private` = privat (owner, masa genesis
  `born_at + TLD_TERM_SECS` — 3 tahun, siklus hidup standar:
  `RenewTld`, masa tenggang 60 hari).
- Sort kanonik per nama (konten sama → blok sama → hash sama,
  apa pun urutan penyisipan); duplikat ditolak.

## Batasan dan jaminan memori

| Sumber daya | Batas |
|---|---|
| Frame P2P | `MAX_MESSAGE_LEN` (1 MiB) sebelum parsing |
| Mempool | 4096 tx (penolakan bertipe melampaui); `MAX_PENDING_PER_DOMAIN = 4` tx tertunda per domain |
| Grace mempool | `MAX_GRACE_ATTEMPTS = 10` penyisipan-ulang tx yang di-flush karena alasan non-definitif (≈ 7 menit pada 40 dtk/siklus), lalu pengusiran |
| Parking fork | `PARKED_CAP = 2048` blok bersaing |
| Checkpoint terakumulasi | `MAX_PENDING_CHECKPOINTS = 256` (pengusiran: hash data terkecil) |
| Penunggu DHT simultan | `MAX_DHT_WAITERS` (256); pencarian DHT berbatas 30 dtk (`DHT_LOOKUP_TIMEOUT`) |
| Blok per balasan sinkron | 1 per putaran; ≤ 10.000 batch per peer (`MAX_SYNC_ROUNDS`); batas protokol `MAX_BLOCKS_PER_REQUEST = 128` pada permintaan |
| Record DHT | `MAX_DHT_CACHE_ENTRY` (64 KiB) |
| Cache DHT persisten (disk) | batas ganda LRU: `DHT_CACHE_MAX_ENTRIES` (100.000 entri) DAN `DHT_CACHE_MAX_BYTES` (2 GiB) — mana yang lebih dulu tercapai mengusir per urutan kunci; pengusiran GC dalam transaksi yang SAMA dengan blok (domain yang dibunuh GC chain tidak pernah meninggalkan entri yatim) |
| Cache negatif DHT | `DHT_NEGATIVE_TTL_SECS` (30 dtk), tutup FIFO `DHT_NEGATIVE_CAP` (4.096) |
| Chain di RAM | genesis + jendela tip (riwayat dilayani dari store) |

Aturan anti-loop (H2): transaksi hanya disiarkan ulang jika
benar-benar **disisipkan** ke mempool (pertama kali terlihat);
duplikat (sudah di pool atau sudah di chain) tidak pernah
di-relay ulang. Blok yang sudah kanonik terdeteksi dan
diabaikan sebelum `push_block` (tanpa re-store, tanpa
re-broadcast).

Keputusan terbuka: identitas jaringan relay yang persisten,
gossip broadcast (saat ini: unicast request-response ke tiap
peer yang dikenal). Terselesaikan sejak penulisan awal: fork
choice nyata (parking + reconsider_parked, lihat atas), konsensus
PoW (produsen bertanda tangan + komite anchor), resolver DNS
(permukaan `--dns`, lihat dns.md), kanal kendali (unifikasi
RPC→p2p: tanpa TCP JSON lagi).

## Shutdown anggun (SIGTERM / SIGINT)

Daemon `scone relay` (foreground) menangkap SIGTERM dan SIGINT
via `tokio::signal`: saat sinyal, future `Relay::run`
dibatalkan (`tokio::select!` dalam biner), yang menjatuhkan
Swarm libp2p dan menutup socket QUIC. Runtime tokio lalu
dihentikan dengan shutdown berbatas (≤ 2 dtk) yang menjatuhkan
task DNS lepas (UDP/TCP) dan menutup socket-nya — tujuannya
agar `kill` tidak meninggalkan socket UDP yatim (kasus diamati
pada testnet 60). Shutdown karena sinyal mencatat `relay
shutting down (signal)` (INFO, stderr) dan keluar dengan kode
keluar **0**: shutdown yang diminta bukan kegagalan. Jika
instalasi handler gagal, perilaku default sinyal berlaku
(degradasi, tanpa panic).


## Transport Tor (isu terbuka)

QUIC/UDP tidak melintasi proxy SOCKS polos, maka relay tidak
bisa secara trivial berjalan di atas Tor hari ini. Transport
Tor libp2p asli (relay beralamat onion) adalah jalur yang
dilacak ke depan — lihat panduan resistensi-sensor situs web
untuk keadaan saat ini dan jembatan CNAME `.tor` (nama terbaca
untuk hidden service, didukung hari ini).

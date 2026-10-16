# Metodologi Audit Keamanan — Scone (pra-mainnet)

Dokumen kerja. Target: keyakinan untuk deployment produksi.
Rujukan lintas-crate: aturan keras AGENTS.md (tanpa kripto buatan
sendiri, dekode ketat, hash dihitung ulang, determinisme) adalah
spesifikasi kebutuhan audit ini.

## 0. Ruang lingkup dan ancaman (sebelum baris kode apa pun dibaca ulang)

Model lawan: node p2p yang permusuhan sewenang-wenang (bisa
mengirim byte apa pun via QUIC/UDP), pengirim tx permusuhan,
operator relay yang ingin tahu, data-dir yang dicuri. DIKELUARKAN
menurut desain: klien akhir (DNS bersifat publik), ketersediaan
global (chaos sudah diuji).

Tabel aset, menurut urutan kritisitas:
1. **Ownership** (state chain: Register/Update/Transfer/Assign) —
   false positive = pencurian domain.
2. **Penguasaan tanda tangan/kunci** (keystore, Ed25519, derivasi
   OwnerId).
3. **Validasi input tidak dipercaya** (dekode wire, batasan,
   panic).
4. **DoS sumber daya** (cache DNS, mempool, DHT, sinkronisasi).
5. **Determinisme/konsensus** (reorg, GC, kebangkitan state).

## 1. Audit statis otomatis (hari 1 — mekanis)

- `cargo audit` (CVE dependensi: libp2p/quinn/rustls/redb/ed25519).
- `cargo geiger` + tinjauan `unsafe` (diharapkan ≈ 0).
- `cargo clippy --workspace --all-targets -- -D warnings`
  (sudah hijau, jadikan penghalang CI).
- Fuzzing batas dekode (hari 2-3):
  `cargo-fuzz` pada `scone_protocol::decode_complete` (Message,
  Transaction, Block, SignedDnsRecord, Checkpoint) — inilah
  permukaan di mana "tidak pernah panic atas data tidak dipercaya"
  harus berlaku. Anggaran: 24 jam fuzzing kontinu, korpus di-seed
  dari test yang ada. Kriteria keluar: nol crash, nol timeout,
  cakupan cabang dekode > 95%.
- Fuzzing parser wire DNS (dns.rs `parse_query`,
  `extract_answers`) dan parser berkas record (domain.rs).

## 2. Tinjauan manual tertarget per komponen (hari 4-8)

Diurutkan menurut kritisitas aset, 1 komponen = 1 daftar periksa:

### 2.1 scone-crypto + derivasi identitas (hari 4)
- OwnerId = BLAKE3(pubkey): periksa kanonisasi byte (tidak ada
  representasi ambigu → 2 kunci, 1 owner).
- Ed25519: DESERIALIZATION ketat (kunci non-kanonik ditolak?),
  verifikasi dengan pemisahan konteks (tiap pemakaian punya tag
  domainnya: SCONE-TX-SIG-V1, SCONE-BLOCK-V1, record DHT — replay
  lintas protokol mustahil).
- Keystore: passphrase → kunci (KDF? iterasi?), zeroization,
  izin berkas, padding oracle pada ciphertext.

### 2.2 Penerapan transaksi (hari 5) — intinya
Untuk SETIAP varian Transaction (13 hari ini):
- Apakah tx idempoten di bawah replay (txid unik, nonce/sequence)?
- Apakah pemeriksaan ownership mendahului efek apa pun? (penandatangan ==
  owner yang dipelajari on-chain, tidak pernah dipercaya dari tx itu
  sendiri.)
- Sequence UpdateDomain: tepat current+1 (bukan ≥) — periksa
  penolakan celah dan penolakan replay.
- Renew: batas waktu (≤ 3 tahun, > sekarang) — waktu berasal dari
  BLOK, bukan jam lokal (aturan sudah ditetapkan, periksa setiap
  call site).
- AssignDomain/TransferTld: apakah pubkey penerima DPELAJARI
  (jebakan pool tanpa pubkey — lihat skill) di semua jalur.
- GC/kedaluwarsa: tidak mungkin kebangkitan (paradoks
  precheck-OK/lookup-OK/push-gagal) — periksa
  gc_expired_journaled.

### 2.3 Lapisan jaringan (hari 6)
- Setiap byte remote melewati dekoder ketat berbatas SEBELUM
  alokasi (MAX_* dari limits.rs) — cari `Vec::with_capacity(input_len)`
  tanpa batas.
- requery/sync GetBlocks: batas per peer dan per siklus
  (sudah: 128 blok/req) — periksa ketiadaan loop unduh-ulang.
- Gossip Record: verifikasi ulang chain SEBELUM cache/gossip
  (diverifikasi oleh test record_dht saya) — periksa juga batas
  fanout.
- kad DHT: bisakah peer permusuhan memaksa penyimpanan N record
  (kuota per-peer?) atau pengusiran terarah atas record jujur?

### 2.4 Permukaan DNS (hari 7)
- Cache: injeksi mustahil (kunci (qname,qtype) kanonik — tanpa
  tabrakan kasus/enkode), anggaran memori keras (teruji), tanpa
  cache poisoning via jalur fallback (TXID + gema pertanyaan —
  pastikan upstream tidak bisa dipalsukan dari jauh).
- Jawaban: AA jujur, TC jujur, tanpa amplifikasi (rasio
  respons/permintaan berbatas — MAX_INFLIGHT_UDP sudah ada,
  periksa replikasi TCP).

### 2.5 Penyimpanan & genesis (hari 8)
- redb: kerusakan disk → error bertipe, tanpa panic (test diskfail
  sudah ada di sisi penandatangan — padanan relay-nya?).
- Genesis: hash mencakup seluruh manifest; dua genesis berbeda =
  penolakan data-dir (teruji); verifikasi kemustahilan menyuntik
  genesis SETELAH fakta pada data-dir yang tersinkron.

## 3. Test lawan (hari 9-12) — tulis yang kurang

Untuk tiap temuan tinjauan manual, test yang membuktikannya:
- node permusuhan dalam sim (SimNet dengan node yang mengirim
  pesan salah-bentuk/tidak setuju) — perluas SimNet dengan peran
  "byzantine pasif";
- mesh nyata 30 node dengan 3 node permusuhan (byte acak, ledakan
  tx tidak valid, record DHT palsu) — baterai chaos sudah ada,
  tambahkan peran penyerang;
- cek silang: dua implementasi dekode (dokumen protocol.md
  NORMATIF — auditor mengimplementasikan wire dari dokumen SAJA
  dan membandingkan byte; setiap perbedaan = bug dokumen ATAU
  kode).

## 4. Audit eksternal (jika ada anggaran) — setelah 1-3

Ruang lingkup minimal yang menguntungkan untuk vendor: scone-crypto,
scone-protocol (wire), penerapan tx (state.rs), ~8-10 k LOC.
Format: 2 pekan, 1 auditor kripto senior + 1 peninjau. Hasil:
laporan dengan severity, PoC, perbaikan yang ditinjau. Andalkan
dokumentasi normatif yang sudah ada (docs/technical) sebagai dasar
spesifikasi kebutuhan — ini keuntungan biaya nyata.

## 5. Kriteria keluar (go/no-go mainnet)

- [ ] cargo-fuzz 24 jam tanpa crash pada 5 dekoder
- [ ] Tinjauan 2.2 lengkap: tiap varian tx punya lembarnya
      (ownership/replay/waktu/efek) yang disetujui peninjau
- [ ] Mesh 30 node termasuk yang permusuhan: 48 jam tanpa
      perbedaan tip atau korupsi cache
- [ ] Perbedaan dokumen/kode: nol (diverifikasi oleh implementasi
      wire ganda)
- [ ] CVE pemblokir: nol (cargo audit)
- [ ] Audit eksternal lingkup kripto/wire/tx selesai (jika
      dianggarkan)

## Urutan eksekusi yang disarankan

1→2.1→2.2 (inti, 60% dari risiko)→2.3→2.4→2.5→3 paralel
dengan tiap tinjauan (test lawan ditulis pada hari penemuan,
bukan setelahnya).

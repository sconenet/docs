# Model keamanan — pemilihan komite dan anti-grinding

> Dokumen normatif. Setiap perbedaan dokumen/kode = bug.

## Ringkasan

Komite epoch N+1 diundi dari seed
`next_seed(signing_hash(checkpoint N)) = blake3("SCONE-SEED-V1" || signing_hash(checkpoint N))`,
atas **kumpulan kelayakan yang dibekukan pada blok yang
difinalisasi oleh checkpoint N**. `signing_hash` adalah hash
**konten** checkpoint (epoch, tinggi, blok, prev, akar state —
himpunan tanda tangan dikecualikan): seed adalah fungsi murni
dari APA yang difinalisasi, bukan dari SUBKUMPULAN kuorum mana
yang menandatanganinya — dua view yang memegang subkumpulan
berbeda (keduanya valid) dari checkpoint yang sama memilih komite
yang sama.

## Ancaman: grinding seed komite

Penyerang ingin memengaruhi komposisi komite. Tiga jendela
serangan ada; ketiganya tertutup secara struktural:

### 1. Sebelum seed — mustahil meng-grinding posisi

Seed epoch N+1 diturunkan dari **hash checkpoint N**,
bertanda kuorum (floor(2n/3)+1 dari komite N — `quorum_for`).
Checkpoint N mengikat `state_root` dari state yang
**membekukan kumpulan** (basis PoS = state tepat pada blok
terfinalisasi, `FinalizedBase.pool`). Menghasilkan seed berbeda
memerlukan checkpoint bertanda kuorum yang berbeda — artinya
mengkorupsi ≥ 1/3 komite N, yang merupakan hipotesis kegagalan
yang dikecualikan model BFT.

### 2. Setelah seed — terlambat untuk bergabung

Registrasi apa pun (klaim domain/TLD, kemunculan kunci owner baru)
hanya berlaku untuk epoch berikutnya, yang seednya belum
diketahui. Kumpulan dibekukan dalam `FinalizedBase` SEBELUM siapa
pun (termasuk penandatangan checkpoint N) bisa menghitung seed
N+1.

### 3. Seleksi "terbaik-dari-m" tanpa keuntungan

Posisi kunci dalam undian =
`blake3("SCONE-COMMITTEE-V1" || seed || pk)` (seragam), yang
satu-satunya derajat kebebasannya adalah `pk`. Penggrinder yang
menghasilkan m kunci dan hanya mendaftarkan yang posisinya
terbaik harus lebih dulu **mengenal seed** — tetapi pada titik itu
kumpulan sudah dibekukan. Mendaftarkan m kandidat "untuk
jaga-jaga" berbiaya m proof of work (satu per domain/TLD) dan
menghasilkan tepat m/|pool| kehadiran yang diharapkan:
**tidak ada keuntungan atas m registrasi langsung**. Strategi
adaptif (lihat seed, lalu pilih yang akan didaftarkan) mustahil
secara temporal.

### 4. Bias produsen-terakhir (serangan menahan)

Produsen blok terakhir sebelum checkpoint bisa menahan bloknya
untuk memengaruhi `signing_hash(checkpoint N)` dan karenanya
seed. Dua pengaman:

- checkpoint menandatangani state pada tinggi H, dan **kumpulan
  dievaluasi pada timestamp blok H** — menahan blok menunda
  checkpoint tetapi tidak mengubah kumpulan epoch yang sudah
  dibekukan oleh checkpoint N−1 (kumpulan sedikit tumpang tindih
  antar epoch);
- menahan merugikan penyerang reward produksinya sendiri dan
  hanya memengaruhi LIVENESS (checkpoint tertunda), bukan SAFETY:
  dua checkpoint final yang tak kompatibel memerlukan ≥ 1/3 tanda
  tangan ganda, dapat dibuktikan dengan `SlashTx` dan dihukum
  dengan banning.

Kesimpulan: VRF atau commit-reveal tidak menambah apa pun di sini
— seed sudah merupakan fungsi dari peristiwa bertanda kuorum yang
tak-terduga-tapi-tak-ter palsukan-delay, dan pembekuan kumpulan
menghapus satu-satunya tuas adaptif. Jangan rumitkan protokol
(batasan P1.7: "jika perlu" — analisis: tidak perlu).

## Ekuivokasi dan slashing

Anchor yang menandatangani dua checkpoint bertentangan pada epoch
yang sama bisa dibuktikan oleh siapa pun yang memegang kedua
tanda tangan (`SlashTx`, diskriminan `0x15`): pengusiran dari
kumpulan PoS (terjurnal, dapat dibatalkan saat reorg).
Domain/TLD terdakwa tidak disita (sanksi minimal — keputusan M9).

## Properti yang diverifikasi oleh test

- `checkpoint_hash_engages_signatures`: hash checkpoint mencakup
  tanda tangan (tanpa manipulasi belakangan);
- `seed_chain_and_recovery`: rantai seed diturunkan secara eksak,
  pemulihan k disertakan;
- `small_committee_never_finalizes`: penjaga BFT — komite di bawah
  `MIN_FINALITY_COMMITTEE_SIZE` tidak pernah finalisasi;
- e2e 4 relay: checkpoint terfinalisasi dan menyebar, epoch sama
  di semua node.

## Batas yang diketahui (terdokumentasi, belum diperbaiki)

- Kumpulan yang layak mencakup owner TLD hidup (klaim = PoW yang
  dibayar) — permukaan Sybil dibatasi oleh biaya PoW per kunci;
- `state_root` checkpoint diverifikasi di RAM hanya saat
  checkpoint menargetkan tip; replay retrospektif adalah jalur
  storage/relay (lihat `docs/technical/blockchain.md`).

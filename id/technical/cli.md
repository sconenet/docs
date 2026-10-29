# CLI (`scone`) — logging dan verbositas

> Perilaku normatif logging biner `scone`. Setiap perbedaan
> dokumen/kode = bug (perbaiki sumber kebenarannya).

## Prinsip

- **stdout dicadangkan untuk keluaran perintah** (`scone status`,
  `identity list`, `status`, …). Skrip bisa mem-redirect stdout
  tanpa pernah menemukan log di dalamnya — piping tetap andal.
  Pengecualian yang diterima: `scone relay` memancarkan baris data
  `p2p: <multiaddr>`, `dns: <addr>`, dan `metrics: <addr>` di
  stdout begitu listener terikat (data yang bisa dipakai untuk
  `--bootstrap` / konfigurasi, bukan log; dipancarkan tepat sekali
  masing-masing, di-flush segera).
- **Semua log pergi ke stderr** (format `tracing` kompak),
  termasuk milik relay — relay memang sudah log ke stderr sebelum
  migrasi.
- Pesan error CLI yang menghadap pengguna (`scone: <error>`)
  tetap di stderr: ini UX, bukan log.

## Level dan `--verbose` / `-v`

| Pemanggilan | Level efektif |
|---|---|
| `scone <cmd>` (tanpa `-v`) | `warn` |
| `scone relay` (tanpa `-v`) | `info` |
| `scone -v <cmd>` / `scone relay -v` | `info` |
| `scone -vv …` | `debug` |
| `scone -vvv …` (dan seterusnya) | `trace` |

`-v`/`--verbose` adalah flag **global dan dapat diulang**: ia
diterima sebelum atau sesudah subperintah (`scone -vv identity list`
dan `scone identity list -vv` setara).

**Mengapa `scone relay` tetap di INFO secara default**: relay
adalah daemon — ia harus mencatat aktivitasnya (listening RPC/p2p,
blok yang diterima dan diproduksi, peer yang terhubung, penolakan)
agar bisa dioperasikan, tanpa memerlukan flag. Perintah sekali
jalan sebaliknya tetap diam (WARN) agar tidak mengotori terminal
atau skrip.

## `RUST_LOG` (override)

Jika variabel lingkungan `RUST_LOG` diset, ia **sepenuhnya
menggantikan** level yang diturunkan dari `-v` (sintaks
`env-filter`):

```bash
RUST_LOG=debug scone identity list        # debug tanpa -v
RUST_LOG=error scone -vv relay            # error menang atas -vv
RUST_LOG=scone_network=trace,info scone relay   # per-target
```

## Konfigurasi lingkungan (`.env`)

Sebelum mem-parsing argumen, `scone` memuat berkas `.env` dari
direktori saat ini (`dotenvy::dotenv()` di `main.rs`). Berkas
hilang = no-op diam, tidak pernah error. Nilai yang dimuat menjadi
baseline opsi relay yang mendeklarasikan variabel lingkungan
(`env = "…"` di `cli.rs`).

Prioritas (satu opsi, tiga sumber):

```text
flag CLI eksplisit  >  variabel .env  >  default bawaan
```

- flag yang diberikan di baris perintah SELALU menang;
- jika tidak, variabelnya (baik dimuat dari `.env` maupun diset di
  lingkungan proses nyata — variabel sama, efek sama);
- jika tidak, default bawaan clap.

Variabel yang dapat diulang (`SCONE_BOOTSTRAP`, `SCONE_DNS_UPSTREAM`)
berakumulasi: satu baris per nilai di `.env` (dotenvy menambahkan).

Tabel lengkap variabel `SCONE_*` (jaringan, listen P2P, bootstrap,
genesis, UPnP, permukaan DNS, metrik, profil topologi,
anchor) dan semantik masing-masing ada di
[`relay.md`](relay.md#environment-configuration-env); templat
berkomentarnya adalah `.env.example` di akar repositori (salin ke
`.env`, jangan pernah commit — `.env` di-gitignore).

## Semantik level (konvensi kode)

- `error!`: kegagalan (tidak ada call site hari ini — kegagalan
  jaringan ditelan sebagai `warn!` dan hanya kegagalan lokal yang
  fatal muncul sebagai error bertipe dari `run()`);
- `warn!`: masukan jaringan permusuhan atau berbahaya yang
  ditolak, tx basi diusir dari mempool, tx bertentangan
  dijatuhkan saat produksi, kegagalan outbound/inbound,
  ketidakcocokan kunci DHT;
- `info!`: blok diterima/diproduksi, listening RPC/p2p, peer
  terhubung, startup relay (checkpoint state);
- `debug!`: detail internal (tx diterima, peer terputus, putaran
  RPC, daftar keystore, event kad di bawah `SCONE_KAD_DEBUG`);
- `trace!`: dicadangkan untuk payload (tidak ada call site hari
  ini).

Crate library (`scone-network`, …) memancarkan makro `tracing`
tetapi **tidak menginisialisasi subscriber**: inisialisasi hanya
ada di biner `scone` (sebelum dispatch subperintah — relay
membutuhkan lognya sejak boot).


## Penggunaan sederhana (cli-simplify-v1)

- **Tanpa `--peer`**: perintah berbicara dengan relay lokal secara
  default; env `SCONE_PEER` atau `--peer` menggantikan.
- **Tanpa `--identity`** saat keystore memegang satu identitas
  (dipakai diam-diam; beberapa → error mendaftarkannya).
- Dihapus (stub menyarankan penggantinya): `show`, `submit`,
  `lookup`, `owned`.
- Baru: `scone domain renew <name>`, `scone domains`, `scone domain
  get|set` (record tunggal, tanpa berkas).

```bash
scone identity generate me
scone domain register example.scone
scone domain set example.scone A 192.0.2.1
scone domain renew example.scone
```

## Siklus hidup TLD (cli-unified-lifecycle)

`scone tld register <tld>` adalah titik masuk TLD satu-perintah,
simetris dengan `scone domain register` — state on-chain dibaca
DULU, lalu tindakan yang tepat berjalan:

- **TLD bebas** → didaftarkan, lahir **TERBUKA secara default**
  (tanpa flag: namespace mandiri, 1/3 dari PoW). `--private`
  memilih namespace milik-penuh dengan harga PoW penuh, dengan
  konfirmasi eksplisit (keputusan ekonomi). `--yes` melewati
  konfirmasi (skrip).
- **milik Anda, kedaluwarsa dalam 90 hari** → PERPANJANGAN
  diusulkan (PoW terikat pada kedaluwarsa saat ini — anti-reuse)
  dan berjalan setelah konfirmasi.
- **milik Anda, jauh dari kedaluwarsa** → tidak ada yang perlu
  dilakukan (`tld renew --force` untuk memperpanjang tetap).
- **milik orang lain** → error bertipe yang bersih dengan
  ownernya.

```bash
scone tld register mytld            # kelahiran terbuka (default)
scone tld register mytld --private  # namespace milik (konfirmasi)
scone tld renew mytld               # perpanjangan eksplisit
```

Demikian pula, `scone domain register <name>` atas nama yang Anda
miliki yang kedaluwarsa dalam 90 hari mengusulkan perpanjangan
alih-alih gagal.

## Delegasi record (KSK/ZSK-lite)

Owner sebuah domain dapat memberikan KUNCI lain hak untuk
menandatangani `domain update` atas namanya — himpunan delegasi
record (paling banyak 3 kunci, penggantian penuh pada setiap
`set`):

```bash
# pemberian (ditandatangani owner; CLI mengambil generasi CAS
# namespace dan mengikat transaksi padanya):
scone domain delegates set example.uip \
  --keys <pk-hex-1>,<pk-hex-2> --identity alice

# inspeksi (tanpa identitas):
scone domain delegates list example.uip

# cabut semuanya:
scone domain delegates clear example.uip --identity alice
```

Seorang delegasi kemudian menandatangani `domain update` persis
seperti yang dilakukan owner (`--identity deleg`) — on-chain,
field `owner` tx adalah owner DOMAIN sementara tanda tangannya
berasal dari delegasi. Pemberian itu eksklusif owner (seorang
delegasi tidak bisa mengelola himpunannya), dihapus oleh sebuah
transfer, dan setiap `set`/`clear` menaikkan penghitung mutasi
namespace (anti-replay CAS — lihat `transactions.md`): `set` yang
di-replay tidak bisa menghidupkan kembali kunci yang dicabut.
Pembaruan oleh delegasi yang dicabut ditolak secara lokal
("bukan owner maupun delegasi record").

## Perintah lanjutan

Default di atas mencakup pemakaian sehari-hari. Untuk skrip dan
interop web/API permukaan lengkap tetap tersedia:

setiap aksi memiliki tepat SATU jalur kanonik; alias root lama
(`show`, `lookup`, `owned`, `renew`) telah dihapus
(remove-cli-aliases), begitu juga `domain info` yang tidak
pernah menjadi perintah nyata.

Permukaan heks offline (`tx build|sign|verify`, subperintah
`submit tx`) DIHAPUS (remove-tx-surface-v2): operasi M8b yang
dilayaninya kini perintah kelas satu —
`scone tld open <tld>` (pembukaan mandiri satu arah, PoW
pembukaan ditambangkan untuk Anda), `scone tld transfer <tld>
<owner>`, `scone tld revoke <tld>`, `scone domain assign <name>
<owner>` (owner TLD, TLD privat) dan `scone domain transfer
<name> <owner>`. Semuanya: baca state, tanda tangan, pengiriman,
txid. Resolusi record DNS pindah ke `scone domain resolve <name>`
(dulu `record get`); kanal publikasi heks (`record put`) hilang —
penyuntingan lewat `domain set` / `domain update`.
- `scone domain mine <name>` — menambang HANYA bukti
  registrasi (tanpa identitas, tanpa keystore, tanpa
  pengiriman): mencetak JSON `scone-domain-pow-v1` `{type,
  name, network_id, epoch, proof}` ke stdout. Penambangan
  terdelegasi: mesin lain menambang untuk Anda, kunci Anda
  tidak pernah keluar dari milik Anda. Pipe atau redirect:
  `scone domain mine n.tld | scone domain register n.tld
  --proof -` (atau `--proof FILE`). Bukti terikat pada nama
  dan jaringan; kesegaran ditegakkan node saat penerapan
  (jendela validitas epoch). Dengan `--mine-for <pubkey>`, ia
  malah menambang bukti POW-TO-OWNER untuk penerima manfaat
  itu dan mencetak JSON `scone-domain-pow-to-v1` `{type,
  name, network_id, epoch, pow_to, proof}` — hanya dapat
  dikonsumsi oleh `register --to` / `renew --to` untuk kunci
  tepat itu (tantangan menanamkannya; pemeriksaan tak-
  peka-huruf saat pengiriman).
- Pengiriman pow-to-owner (tanpa keystore — `--to` eksklusif
  dengan `--identity`): `scone domain register <name> --to
  <pubkey>` / `scone domain renew <name> --to <pubkey>` dan
  `scone tld register <tld> --to <pubkey>` / `scone tld renew
  <tld> --to <pubkey>` mengirim bukti yang tantangannya
  mengikat kunci PENERIMA MANFAAT; `owner` on-chain dihitung
  ulang sebagai `hash(pubkey_dest)` (sebuah renew TIDAK
  PERNAH berganti tangan). Tanpa `--proof`, perintah
  menambang buktinya sendiri; dengan `--proof -`/FILE ia
  mengonsumsi payload `scone-domain-pow-to-v1` dan menolak
  bukti yang ditambang untuk penerima manfaat LAIN. Gaya
  pipe: `scone domain mine n.tld --mine-for <pubkey> |
  scone domain register n.tld --to <pubkey> --proof -`.
- `scone domain export <name> [--out FILE]` — menulis himpunan
  record terverifikasi-chain dalam format `update --file` yang
  persis (round-trip terjamin: export → edit → update mengirim
  ulang himpunan yang sama). Tanpa `--out`: stdout. `domain
  resolve` memakai format terbaca yang sama, jadi `resolve >
  zone.txt` langsung bisa diimpor. Kind tanpa sintaks tekstual
  (GEOROUTE, tidak dikenal) diekspor sebagai peringatan
  berkomentar — tidak pernah dijatuhkan diam-diam.

`domain get` juga mencetak `domain_id` dan `valid_until`.

## Benchmark PoW (`scone bench-pow`)

Mengukur laju hash Argon2id mesin (hash yang sama yang dipakai
verifikasi konsensus) dan menerjemahkan kesulitan jaringan
menjadi waktu tunggu konkret:

```
scone bench-pow --network mainnet [--seconds 5] [--cores N]
```

Keluaran: H/s total dan per-worker, lalu perkiraan waktu
rata-rata per operasi — transfer/assign (tarif datar),
pembukaan TLD privat, dan lantai registrasi TLD/domain (kurva
panjang). Jaringan yang dipilih menentukan parameter Argon2id
(mainnet: 64 MiB, t=2 — biaya sebenarnya; testnet/devnet:
profil ringan, bukti milidetik). Setiap worker mengalokasikan
jalur Argon2id 64 MiB-nya sendiri — sesuaikan `--cores` dengan
RAM yang tersedia.

## Build: dev vs mainnet-aktif (mainnet-compile-gate)

Build default **tanpa mainnet**: `scone relay --network
mainnet` (dan layanan registrar dengan `SCONE_NETWORK=mainnet`)
gagal segera dengan petunjuk build ulang. Biner resmi yang
didistribusikan dibangun dengan:

```bash
cargo build --release --features mainnet
```

`scone --version` menunjukkan `+mainnet` pada build demikian, dan
log startup relay mengatakan `build: mainnet-enabled`.

Default `--network` mengikuti build juga: `mainnet` pada build
mainnet-aktif (biner resmi mendaratkan pengguna di jaringan
produksi), `testnet` sebaliknya. Flag `--network` eksplisit atau
env `SCONE_NETWORK` selalu menang (urutan clap: flag > env >
default). Ini pagar operasional (eksekusi tak sengaja, biner
dev/CI) — bukan batas kriptografis; aturan konsensus tetap
pertahanan melawan node permusuhan.

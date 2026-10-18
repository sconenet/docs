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
genesis, mDNS, UPnP, permukaan DNS, metrik, profil topologi,
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
- `scone domain export <name> [--out FILE]` — menulis himpunan
  record terverifikasi-chain dalam format `update --file` yang
  persis (round-trip terjamin: export → edit → update mengirim
  ulang himpunan yang sama). Tanpa `--out`: stdout. `domain
  resolve` memakai format terbaca yang sama, jadi `resolve >
  zone.txt` langsung bisa diimpor. Kind tanpa sintaks tekstual
  (GEOROUTE, tidak dikenal) diekspor sebagai peringatan
  berkomentar — tidak pernah dijatuhkan diam-diam.

`domain get` juga mencetak `domain_id` dan `valid_until`.

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

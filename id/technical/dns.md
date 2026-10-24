# Server DNS UDP + TCP (M6 + M7b) — `scone dns`

> Spesifikasi normatif permukaan DNS Scone.
> Setiap perbedaan dokumen/kode = bug (perbaiki sumber
> kebenarannya).

## Ringkasan

**Server DNS** adalah permukaan opsional relay: ia menjawab
kueri DNS **UDP dan TCP** (RFC 7766, port sama) untuk nama Scone
**hanya dari data yang diverifikasi terhadap chain**, dan
menawarkan **fallback rekursif opsional** (`addr:port`) untuk
segala hal lain. Ia tidak memiliki chain, store, maupun DHT:
keputusan eksistensi/kedaluwarsa melewati **snapshot DNS
otoritatif** (view RAM yang dipublikasikan setelah tiap blok
yang diterapkan — lihat bawah), dan pengambilan record melewati
relay (proses), via kanal internal `RelayCommand`/`ResolveLocal`
(dalam-proses, satu putaran) — jalur terverifikasi yang sama
dengan CLI.

```text
              UDP+TCP :5353 (default 127.0.0.1)
                    │
             scone relay --dns 127.0.0.1:5353
                    │  ┌────────────────────────────────┐
                    ├──┤ Cache ter-shard, 16 arah (qname)│
                    │  └────────────────────────────────┘
                    │  Berbentuk Scone?
                    │  ├─ YA → snapshot DNS otoritatif:
                    │  │        tidak dikenal/kedaluwarsa → NXDOMAIN (RAM,
                    │  │        nol putaran masuk ke core)
                    │  │        → pencarian apex per sufiks
                    │  │        → `resolve_local` internal (relay)
                    │  │           chain + record terverifikasi (hash,
                    │  │           owner, sequence) → jawaban
                    │  │        apex tanpa record terpublikasi → NODATA
                    │  └─ TIDAK → fallback upstream (jika dikonfigurasi)
                    │           jika tidak REFUSED
                    │
                    │  anggaran respons:
                    │    UDP → 512 byte (tanpa EDNS); luber =
                    │          dipotong pada record terakhir yang muat + TC=1 →
                    │          klien mencoba ulang via TCP
                    │    TCP → 64 KiB (prefiks panjang 2-byte,
                    │          RFC 1035 §4.2.2): himpunan penuh
```

## Startup

```bash
scone relay --dns 127.0.0.1:5353            # hanya permukaan DNS
scone relay --dns 0.0.0.0:53 \
            --dns-upstream 1.1.1.1:53 \
            --dns-upstream 8.8.8.8:53       # + fallback rekursif
```

Relay mengumumkan alamat efektif di stdout (kontrak
dapat-dibaca-mesin, seperti `p2p:`): baris `dns: <addr>` begitu
socket terikat.

Klien validasi: `dig <name> --dns 127.0.0.1:5353
--qtype A` (A, AAAA, TXT, MX, NS, CNAME, ANY).

## Jalur resolusi (OTORITAS = chain, selalu)

0. **Snapshot DNS otoritatif**: view tak dapat diubah
   (`Arc<HashMap<DomainId, {valid_until, has_record}>>`,
   umpan `tokio::sync::watch`) yang dipublikasikan oleh core relay
   **setelah setiap blok yang diterapkan** (push, bukan poll) dan
   saat boot. Keputusan eksistensi/kedaluwarsa — NXDOMAIN, domain
   kedaluwarsa — dilayani dari RAM melalui snapshot ini: satu
   klon `Arc` + satu pencarian map, **tidak pernah masuk ke run
   loop pemilik-tunggal relay**. Jalur penuh
   (`RelayCommand`/`ResolveLocal`) tetap dicadangkan untuk
   pengambilan record yang sebenarnya (pull DHT), langka berkat
   cache. Snapshot TIDAK PERNAH membawa data record: hanya
   eksistensi dan kedaluwarsa.
   - **Delta copy-on-write O(yang-tersentuh)**: map adalah
     `Arc<HashMap>`; perpanjangan tip mempublikasikan delta blok
     (transaksi blok itu sendiri menyebut setiap domain yang bisa
     mereka ubah — Register/Assign/Update/Renew/TransferDomain —
     ditambah penghapusan GC kedaluwarsa deterministik yang
     dievaluasi saat push) melalui `Arc::make_mut`: map diklon
     HANYA jika pembaca masih memegang view sebelumnya. Boot dan
     reorg membangun ulang snapshot penuh (satu walk backend
     berhalaman).
   - Kesegaran: pembaca bisa melihat snapshot berumur beberapa
     ratus ms selama penerapan blok — keusangan yang sama seperti
     yang dimiliki RPC status; TTL cache DNS tidak terpengaruh
     (jawaban tetap otoritatif menurut asalnya).
1. **Cache ter-shard**: [`ShardedCache`] — 16 shard `Cache`
   independen berkunci `hash(qname)` (qtype berbagi shard:
   lokalitas per-nama). Nama panas bersaing dengan 1/16 lalu
   lintas alih-alih mengantre di belakang satu mutex pada ribuan
   qps. Batas GLOBAL (100.000 entri / 64 MiB default,
   `--dns-cache-limit` / `--dns-cache-max-mem`) dipecah per shard
   (kasus terburuk terdokumentasi: faktor 1 + N/shard). Ukuran
   rdata NYATA tiap entri diperhitungkan (~128 byte overhead per
   entri) dan pengusiran FIFO memicu pada batas pertama yang
   tercapai — tutup memori bertahan bahkan terhadap entri
   ukuran penuh (`MAX_DHT_RECORD_LEN`). TTL 60 dtk
   (otoritatif) / di-clamp ke 600 dtk (fallback). Cache negatif
   (NXDOMAIN/NODATA) mengikuti aturan yang sama; tiap entri
   membawa asalnya (otoritatif atau fallback) agar bit AA tetap
   jujur saat hit. **Jangan pernah cache SERVFAIL** (kegagalan
   lokal sementara) **maupun REFUSED** (state konfigurasi):
   keduanya dihitung ulang pada setiap kueri. Pengusiran FIFO
   saat kapasitas. **Hot path (terinspirasi Cloudflare/1.1.1.1)**:
   - entri `Arc<CacheEntry>` bersama: hit adalah kenaikan
     refcount, TIDAK PERNAH klon dalam rdata (`put` MENGGANTIKAN
     Arc — tanpa mutasi bersama; TTL tetap diperiksa di bawah
     kunci shard);
   - **pencarian tanpa alokasi**: map dua level
     (`Box<str>` qname → qtype → entri), `&str` pinjaman kueri
     memprobe langsung (`Borrow<str>`) — kunci tuple
     `(String, u16)` lama mengalokasikan `String` baru pada
     SETIAP pencarian;
   - **record berpadat kontigu**: satu buffer `Box<[u8]>` per
     entri (`u16 rdlen ‖ u16 rtype ‖ u32 ttl ‖ rdata` per
     record) alih-alih satu `Vec` per record — anggaran memori
     melacak tepat apa yang dialokasikan;
   - **kompresi nama RFC 1035 saat keluaran**: owner jawaban
     otoritatif adalah penunjuk kompresi 2-byte
     (`0xC00C`) ke pertanyaan yang digemakan — pertanyaan tidak
     pernah dienkode ulang penuh per record.
2. **Berbentuk Scone?** Pre-filter ketat: label `[a-z0-9-_]`
   (1..63), ≥ 2 label, TLD 1..63 byte `[a-z0-9-]` (lihat
   `TldName::MAX_LEN`), panjang total ≤ 253 — **dan TLD di luar
   root ICANN** (`is_icann_tld`, daftar IANA beku + RFC 2606:
   `com`, `fr`, `org`… tidak pernah milik Scone).
   Nama yang TIDAK BISA jadi nama Scone (mis. `www.foo_bar`,
   garis bawah di TLD) pergi ke fallback; nama Scone yang
   mungkin (`absent.uip`) urusan chain: tidak terdaftar →
   NXDOMAIN otoritatif, bahkan tanpa upstream. TLD ICANN
   (`example.com`) diteruskan ke upstream jika dikonfigurasi,
   REFUSED jika tidak — tidak pernah diselesaikan otoritatif
   oleh Scone.
3. **Pencarian apex**: sufiks dari terpanjang ke terpendek
   (min. 2 label). Sufiks pertama yang state on-chain-nya dikenal
   relay = apex; himpunan record melayani seluruh qname
   (subnama termasuk — perilaku wildcard implisit, keputusan
   terbuka).
4. **`resolve_local` (kanal internal)**: satu putaran, tidak
   pernah penunggu DHT. State on-chain (`registered`, owner,
   sequence) + record cache DHT lokal **diverifikasi ulang**
   (owner, sequence, record_hash == komitmen on-chain). Record
   yang tidak sesuai tidak pernah menjawab — ketersediaan ≠
   otoritas.
   - tidak terdaftar → NXDOMAIN;
   - terdaftar tanpa record valid → NODATA (NOERROR, 0
     jawaban);
   - record terverifikasi → jawaban difilter per qtype.

## Kodek (RFC 1035, subset ketat)

- Kueri: header 12-byte, **satu** pertanyaan, QCLASS = IN,
  **tanpa kompresi** (penunjuk = tolak). Label diturunkan
  hurufnya dan divalidasi. QR=1, QDCOUNT≠1, label tidak valid →
  **diam** (sampah tidak berbiaya apa-apa).
- Respons: QR|RD(gema)|RA, **AA hanya pada jawaban otoritatif**
  (chain, segar atau ter-cache — tidak pernah pada fallback
  maupun RCODE error), pertanyaan disalin verbatim, jawaban
  otoritatif dengan **owner terkompresi RFC 1035** (penunjuk
  2-byte `0xC00C` ke pertanyaan yang digemakan); balasan upstream
  fallback dilayankan apa adanya (tidak pernah dienkode ulang).
- Tipe yang dilayani: A, AAAA, CNAME, NS, MX, TXT, **PTR, SRV,
  CAA** (+ Unknown dienkode ulang verbatim; 9 tipe asli
  totalnya, rdata dienkode per RFC — 1035/2782/8659; CAA ketat:
  bit kritis 0x80 saja, tag huruf kecil). Kueri A/AAAA
  menyertakan CNAME himpunan (pengejaran alias). Kueri ANY
  (255): seluruh himpunan difilter oleh kodek.
- Batasan (M7b): **anggaran respons bergantung pada transport** —
  512 byte via UDP (payload klasik RFC 1035 §2.3.4, tanpa
  EDNS), 65.535 via TCP. Himpunan yang melampaui anggaran
  dipotong pada record terakhir yang muat, dengan **TC=1**
  (pemotongan jujur, RFC 1035 §4.2.1): klien mencoba ulang via
  TCP (RFC 7766), tempat himpunan penuh dilayankan. Balasan
  upstream yang lebih besar dari anggaran UDP dienkode ulang
  terpotong (TC=1) alih-alih diserahkan apa adanya; via TCP ia
  lewat byte demi byte. Pemotongan juga berlaku pada hit cache
  (anggaran diterapkan saat respons, bukan saat pengisian
  cache).
- RCODE: 0 NOERROR (dengan atau tanpa jawaban), 2 SERVFAIL
  (kegagalan lokal), 3 NXDOMAIN (otoritatif), 5 REFUSED
  (non-Scone tanpa upstream).

## Fallback rekursif

- Upstream `addr:port` (UDP), dicoba berurutan, timeout 2 dtk per
  upstream, socket **ter-connect** ke upstream (filtering
  kernel: paket dari sumber lain tidak pernah diterima).
  Respons upstream hanya diterima jika **cocok dengan kueri**:
  TXID sama, QR=1, pertanyaan (QNAME+QTYPE+QCLASS) disalin byte
  demi byte. Respons tidak sesuai → SERVFAIL, dan tidak ada yang
  di-cache.
- Respons upstream tervalidasi diteruskan apa adanya;
  jawabannya di-cache dengan asal `fallback` (AA tidak pernah
  diset, bahkan saat hit cache) dan TTL di-clamp ke 600 dtk.
  RCODE error upstream (selain NOERROR/NXDOMAIN) tidak di-cache.
- **Privat secara default**: tanpa upstream = REFUSED untuk nama
  di luar bentuk Scone.

## Cache negatif DHT (anti-amplifikasi)

**Pull DHT yang tak terjawab** (record diharapkan on-chain —
`record_hash` terkomit — tetapi absen dari cache lokal dan
peer yang ditanya) memarkir nama selama `DHT_NEGATIVE_TTL_SECS`
(30 dtk) dalam cache negatif berbatas (FIFO, tutup
`DHT_NEGATIVE_CAP` = 4.096 nama): resolusi berulang nama yang
sama dalam TTL **ditekan** (dihitung, tidak pernah di-pull
ulang). Pemindai — atau nama panas yang ownernya tak pernah
mempublikasikan — tidak bisa lagi mengubah satu kueri DNS
menjadi satu `GetRecord` per peer mesh per kueri. Entri negatif
mati saat TTL, saat kedatangan record (pull yang berhasil
menulis ke cache nyata), atau terdorong keluar tutup FIFO.

Penghitung terekspos (`scone_dht_negative_hits_total`,
`_expired_total`, `_capped_total` — lihat "Metrik" di
`relay.md`).

## TCP (M7b, RFC 7766)

Port sama dengan UDP (bind TCP kedua pada alamat `--dns`).
Transport mengikuti RFC 1035 §4.2.2: setiap pesan berprefiks
panjangnya pada **2 byte** big-endian.

- **Kueri**: panjang prefiks ≥ 1 dan ≤ 512 byte (kueri DNS muat
  payload klasik — RFC 7766 §6; tanpa EDNS). Di luar batas →
  penutupan koneksi segera.
- **Respons**: anggaran 65.535 byte (maksimum prefiks 2-byte) —
  himpunan yang terlalu besar untuk UDP dilayankan **penuh** di
  sini, TC=0.
- **Pipelining**: beberapa kueri per koneksi dilayani serial,
  berurutan (satu tulis = satu respons berprefiks).
- **Timeout baca**: 5 dtk per baca (prefiks atau body) — klien
  diam diputus, slotnya dibebaskan.
- **Koneksi bersamaan**: maks 64 (semafor); yang ke-65 ditolak
  segera (koneksi ditutup), tidak pernah mengantre — tanpa
  pertumbuhan task atau memori tanpa batas.
- Error baca/tulis/timeout: log debug + penutupan bersih.
  **Tidak pernah panic atas masukan jaringan.**
- Sampah (kueri tak terpaham) mendapat keheningan tetapi tidak
  menutup koneksi: kueri (valid) berikutnya dilayani.

```text
klien                     server
  │ [len u16][kueri]        │
  │ ─────────────────────►   │ cache → otoritatif → fallback
  │ ◄─────────────────────   │ [len u16][respons ≤ 64 KiB]
  │ (UDP telah menjawab TC=1 → klien mencoba ulang di sini)
```

## Batasan dan pagar

| Pagar | Nilai |
|---|---|
| Datagram UDP (kueri/respons) | 4096 byte (ditolak melampaui) |
| Anggaran respons UDP (TC=1 melampaui) | 512 byte (tanpa EDNS) |
| Kueri TCP maks | 512 byte (ditutup melampaui) |
| Respons TCP maks | 65.535 byte |
| Koneksi TCP bersamaan | 64 (semafor, tolak segera) |
| Timeout baca TCP | 5 dtk |
| Cache | batas GLOBAL 100.000 entri / 64 MiB default, dipecah 16 shard (`--dns-cache-limit` / `--dns-cache-max-mem`) |
| TTL otoritatif | **per record** (`record.ttl`, batas 1..=2.592.000 dtk; konstanta historis 60 dtk tetap hanya untuk respons layanan) |
| TTL fallback (clamp) | 600 dtk |
| Kueri UDP paralel | 4.096 (`MAX_INFLIGHT_UDP`, semafor; melampaui: jatuhkan, klien mencoba ulang) — dapat disetel `--dns-max-inflight` (min 1) |
| Cache negatif DHT | 30 dtk (`DHT_NEGATIVE_TTL_SECS`), tutup FIFO 4.096 nama |
| Timeout per upstream | 2 dtk |

Aturan proyek yang dihormati: tidak pernah panic atas data
jaringan; dekode ketat di mana-mana; server tidak menyentuh
chain, store, maupun DHT (relay tetap satu-satunya otoritas
verifikasi).

## Test

- Unit (`scone-network::dns`): kodek ketat (QR, kompresi,
  penolakan label), jawaban otoritatif, NODATA, NXDOMAIN,
  REFUSED tanpa upstream, fallback ke mock UDP, cache berbatas,
  pemotongan TXT, parsing upstream; perbaikan M6: kejujuran
  bit-AA (otoritatif vs fallback vs error, termasuk hit cache),
  pemotongan TC=1 himpunan terlalu besar, penolakan respons
  upstream (TXID atau pertanyaan tak cocok), tanpa cache
  SERVFAIL/REFUSED; cache ter-shard: sebaran shard, pembagian
  batas global, 4 pekerja bersamaan pada nama panas; hot path:
  pencarian tanpa alokasi, entri berpadat, kompresi owner.
- M7b: anggaran per-transport (UDP 512 byte → TC=1, TCP →
  himpunan penuh), pemotongan hit cache, `run_tcp` nyata
  (himpunan > 512 dilayani penuh, pipelining 2 kueri, kueri
  oversize → tutup, klien diam → timeout 5 dtk, sampah lalu
  kueri valid pada koneksi yang sama), balasan upstream > 512
  dienkode ulang TC=1 via UDP / byte demi byte via TCP.
- Integrasi nyata (`tests/dns_server.rs`): relay penuh
  (termasuk produksi devnet) + registrasi + UpdateDomain +
  PutRecord, lalu kueri UDP nyata: A terverifikasi (bit AA),
  TXT, NODATA AAAA, NXDOMAIN, fallback via mock, kueri ulang,
  subnama via apex.
- E2E biner (`crates/scone/tests/dns_cli.rs`): biner `scone`
  nyata — `relay --dns`, `identity generate`, `domain register`,
  `domain update --file`, lalu `dig` (dnsutils) (A terverifikasi,
  TXT, NXDOMAIN, REFUSED) pada port UDP nyata.

## Keputusan terbuka

- Tanpa EDNS (anggaran UDP tetap 512 byte; pemotongan TC=1 +
  TCP mencukupi kebutuhan devnet).
- Himpunan record tidak membawa nama per-record: seluruh
  himpunan melayani setiap subnama apex (wildcard implisit).
  Pengalamatan per-record memerlukan perluasan `DnsRecord`
  (protokol).
- Fallback meneruskan kueri mentah: upstream berbahaya bisa
  menjawab apa pun (respons tak tervalidasi — ini model
  rekursif standar; nama Scone TIDAK PERNAH melewatinya).

## GeoDNS (routing geografis)

> Spesifikasi normatif. Setiap perbedaan dokumen/kode = bug.

### Tabel GeoIP tertanam (otomatis)

Setiap biner menanam **tabel pemetaan IP → negara** yang
dikompilasi ke dalam relay: **tanpa konfigurasi**, routing
geografis aktif di mana-mana, otomatis.

- **Sumber data**: **DB-IP Lite** ("IP to Country Lite"),
  berlisensi **CC BY 4.0** — dapat didistribusikan ulang dengan
  atribusi: `https://db-ip.com` (dikurasi bulanan; sebelumnya:
  delegasi RIR via `scripts/gen-geo-table.py`, dihapus).
  Sel terkuantisasi **/20 v4 dan /24 v6** (granularitas natif
  DB-IP, ~1,3 juta rentang, melampaui batas memori parser) —
  ~96.500 aturan. Tanggal generasi ada di header tabel
  (`assets/embedded-geo.table`).
- **Checksum**: SHA-256 tabel dihitung saat kompilasi
  (`build.rs`), **diverifikasi ulang saat boot** (ketidakcocokan
  = penolakan bertipe — biner yang diubah tidak pernah melayani
  jawaban geografis) dan dicatat saat boot beserta atribusinya.
  Hash menjamin "setiap node dari sebuah build berbagi tabel
  yang sama" adalah sesuatu yang BISA DIPERIKSA, bukan
  diasumsikan.
- **Regenerasi**: `scripts/geo-convert.py` (mengunduh CSV
  DB-IP Lite bulan itu, mengonversi — negara numerik, benua
  turunan ISO→M49, ASN/region absen dari Lite).
- **Granularitas**: negara (ISO 3166-1 numerik) + benua (UN
  M49) turunan. Tanpa ASN maupun region (absen dari Lite —
  kebutuhan ASN/region menunggu evolusi pipeline tertanam,
  tidak pernah tabel lokal).
- **Kinerja**: pencarian **binary search** atas rentang terurut
  (indeks dibangun sekali pada pemakaian pertama, per keluarga
  v4/v6) — terukur ~95 ns/pencarian hot, ~50 µs akses-acak pada
  tabel penuh (bench: `cargo test -p scone-network --release
  --test geo_bench -- --ignored`).
- **Lokasi-dir relay**: routing mengikuti **IP sumber tiap
  kueri** — relay LAN merutekan klien lokalnya ke zona yang
  tepat secara otomatis.
- **Tabel DIWAJIBKAN** (remove-geo-operator-table, 2026-10-22):
  tabel tertanam adalah **satu-satunya sumber** — tidak ada
  impor maupun penimpaan operator. **Tidak ada opsi geo
  runtime**: setiap flag `--geo-*` adalah penolakan clap tegas
  (geozone diwajibkan oleh build). Operator dengan kebutuhan
  khusus (ASN, region) menunggu evolusi pipeline tertanam,
  bukan tabel lokal.

### Ringkasan

Domain bisa mempublikasikan aturan **routing geografis** dalam
himpunan record bertanda tangannya: record `GEOROUTE` (tipe
privat Scone `32.768`, di luar rentang IANA). Resolver menjawab
pertanyaan A/AAAA untuk domain ini dengan **alamat endpoint yang
dipilih aturan** (scope terdekat dengan posisi KLIEN, diturunkan
dari IP sumbernya via tabel), alih-alih record A/AAAA statis.
Segala hal lain (domain tanpa GEOROUTE, qtype lain) mengikuti
jalur DNS klasik, **byte demi byte**.

### Metrik GeoDNS

Endpoint Prometheus mengekspos pemecahan jawaban geo:

- `scone_dns_geo_answers_total` — total jawaban GEOROUTE yang
  dilayani;
- `scone_dns_geo_by_continent_total{continent="<M49>"}` — per
  benua;
- `scone_dns_geo_by_country_total{country="<ISO>"}` — per negara
  (dibatasi oleh standarnya sendiri: ≤ 9 benua, ≤ 250 negara).

### Record GEOROUTE (wire)

GEOROUTE = sebuah **scope** + ≤ 32 **endpoint** (sort kanonik
saat dekode: priority menaik, weight menurun, target menaik;
duplikat = tolak):

```text
GeoScope   = 0x00 (dunia)
           | 0x01 continent.v (kode UN M49: AF=2, AS=142, EU=150,
                                NA=3, SA=5, OC=9, AN=999)
           | 0x02 country.v   (kode numerik ISO 3166-1: US=840,
                                TW=158, FR=250…)
           | 0x03 region.v    (subdivisi ISO 3166-2 OPAQUE, u32:
                                nomor ditentukan penerbit, dibandingkan
                                apa adanya — tanpa tabel lokal)
           | 0x04 asn.v       (nomor AS, u32)
           | 0x05 prefix.addr[16] || prefix.len u8
             (prefiks IP 16-byte ternormalisasi: IPv4 = alamat dalam
              4 byte pertama + EKOR NOL, len ≤ 32;
              IPv6 = 16 byte bebas, len ≤ 128; bit host SELALU
              nol — prefiks tak ternormalisasi ditolak)
GeoTarget  = 0x00 octets[4]  (alamat IPv4 biner)
           | 0x01 octets[16] (alamat IPv6 biner)
GeoEndpoint = target || priority.v || weight.v
GEOROUTE    = scope || count.v(≤32) || endpoints…
```

Batas validasi (dekode): `priority > 0`, `weight ≤ 100`
(`weight = 0` = endpoint tidak aktif, tidak pernah terpilih),
TTL dalam 1..=2.592.000 dtk seperti record apa pun. Negara/benua
yang tak dikenal tabel lokal **diizinkan** di wire: routing
cukup jatuh kembali ke scope yang lebih tinggi.

**Kompatibilitas mundur wire (v1 → v2)**: byte v1 TIDAK
BERUBAH — scope v1 (dunia/benua/negara) mempertahankan
diskriminan `0x00..=0x02` dan enkode eksaknya; tiga scope baru
mengambil diskriminan berikutnya (`0x03..=0x05`) dalam urutan
total yang konsisten dengan sort kanonik
(`World < Continent < Country < Region < Asn < IpPrefix`).
Dekoder v1 yang bertemu diskriminan ≥ `0x03` sudah gagal ketat
(record ditolak); byte buatan v1 tetap terdekode identik (lihat
test `geo_known_bytes`, dibekukan).

### Pemilihan (resolusi)

1. **Scope** — spesifisitas menurun, **6 level**:
   **prefiks IP terpanjang yang cocok → AS eksak → negara eksak
   → region eksak → benua → dunia**.
   - *IpPrefix*: di antara rute yang prefiksnya cocok dengan IP
     klien (sebagaimana dikenal locator: `client_ip`, jika tidak
     prefiks asal yang dikenal), yang **terpanjang** (`len`
     maks) menang; pada `len` sama, yang pertama dalam urutan
     kanonik (urutan slice `routes`, terurut saat enkode).
     Subdivisi kurang spesifik daripada prefiks/AS (yang
     mengidentifikasi jaringan eksak) dan kurang dari negara
     (penomoran opaque, andal hanya jika penerbit dan locator
     berbagi yang sama), tetapi lebih dari benua (ia terkandung
     dalam negaranya, maka a fortiori dalam benuanya).
   - *Asn*: nomor AS eksak klien.
   - *Country*: negara eksak.
   - *Region*: subdivisi eksak (identifier opaque).
   - *Continent*: diberikan locator **atau diturunkan dari
     negara** via tabel lokal (`continent_of_country`); negara
     yang tak dikenal tabel jatuh ke benua eksplisit atau
     dunia.
   - *World*.
   [`GeoLocation`] v1 (region/asn/prefiks/client_ip tak diset)
   menghasilkan tepat perilaku v1: negara → benua → dunia.
   Tanpa rute yang cocok (termasuk tanpa scope World) →
   perilaku DNS klasik, seolah GEOROUTE tidak ada.
2. **Priority** — semantik mirip SRV: hanya grup **priority
   minimum** yang layak; priority lebih tinggi adalah level
   **failover**.
3. **Weight** — dalam grup terpilih, undian **terbobot
   deterministik**: hash stabil (SplitMix64) atas nonce
   (penghitung kueri resolver) terhadap bobot kumulatif. Tanpa
   RNG: dua kueri dengan nonce sama memberi hasil sama, nonce
   yang meningkat mendistribusikan proporsional terhadap bobot.
   `weight = 0` = tidak aktif, tidak pernah terpilih (bahkan
   sendirian — pindah ke level berikutnya).
4. **Urutan bertanda tangan, tanpa probing** — resolver TIDAK
   PERNAH menilai ketersediaan sebuah endpoint: setiap endpoint
   aktif (weight > 0) dari scope terpilih adalah kandidat,
   kepala level minimal terpilih lewat undian terbobot, dan
   yang berikutnya bertindak sebagai failover **dalam urutan
   himpunan record bertanda tangan** (urutan itulah mekanisme
   failover-nya). Jangkauan endpoint adalah urusan operator
   endpoint, bukan urusan resolver.

### Respons

- Satu alamat: **A jika IPv4, AAAA jika IPv6** — filter ketat
  per keluarga menurut qtype (pertanyaan A TIDAK PERNAH menjawab
  IPv6, dan sebaliknya).
- TTL = TTL record GEOROUTE sumbernya (dibatasi saat dekode
  oleh aturan TTL umum).
- AA diset (respons otoritatif — GEOROUTE adalah record bertanda
  tangan terverifikasi seperti lainnya).
- Respons kosong setelah filter (tanpa kandidat keluarga yang
  tepat, tanpa rute untuk zona tersebut) → resolver
  melanjutkan jalur klasiknya (biasanya NODATA: NOERROR, 0
  jawaban); rcode error mengikuti aturan umum (tidak pernah
  di-cache).
- Record A/AAAA klasik yang koexist dalam himpunan
  DIDAHULUKAN atas geo (geo berlaku hanya jika tak ada jawaban
  wire untuk qtype).

### Cache: keputusan eksplisit

Keputusan routing geo **TIDAK PERNAH di-cache** di bawah
`(qname, qtype)`: ia bergantung pada scope klien resolver, dan
kunci seperti itu akan melayani rute AS ke klien TW. Jawaban geo
dihitung ulang pada setiap kueri (biaya: satu select_scope +
undian terbobot, µs) dan melewati cache. Cache klasik
(qname, qtype) tetap tak berubah untuk segalanya lain —
termasuk NODATA yang dihasilkan saat geo tidak memutuskan
apa pun. Rcode error tetap tidak pernah di-cache.

### Delegasi NS (record `ns`) — owner memilih siapa yang melayani

Owner sebuah domain memilih mode pelayanan per domain:

- **DHT (default)** — tanpa record `ns`, jaringan Scone melayani
  A/AAAA on-chain (relay `--dns`, atau jembatan NSD via export);
- **NS independen** — owner menambahkan record
  `ns <ip-atau-nama>` (nameserver MILIKNYA, IP langsung atau
  hostname): domain DIDELIGASIKAN — jaringan berhenti melayani
  kontennya (A/AAAA → NODATA + otoritas NS), dan siapa pun yang
  me-resolve mengikuti NS milik owner. Batasnya: **12 NS
  berbeda per domain** (semua bentuk, terdeduplikasi per nilai
  — cermin dari batas registry dunia nyata). Menghapus sebuah
  NS membebaskan slotnya; menghapus semuanya mengembalikan
  mode DHT.

**Tidak ada pemeriksaan kesehatan yang ada**: ketersediaan
endpoint adalah urusan operator endpoint — resolver melayani
himpunan record bertanda tangan apa adanya, dalam urutan record
(urutan record itulah mekanisme failover-nya).

### Keamanan

- Record GEOROUTE adalah record bertanda tangan seperti
  lainnya: diverifikasi terhadap chain (owner, sequence,
  record_hash) sebelum penggunaan apa pun. **Tanpa kepercayaan
  jaringan tanpa validasi** — GEOROUTE DHT tak terverifikasi
  tidak pernah menjawab.
- Posisi klien berasal dari **sirkuit yang ditemukan otomatis**:
  relay menemukan lokasinya sendiri via alamat eksternalnya
  (STUN/AutoNAT/UPnP) yang diselesaikan oleh tabel tertanam
  yang diwajibkan build — klien LAN (IP sumber privat) mewarisi
  posisi itu, klien Internet diselesaikan oleh IP mereka
  sendiri, yang tak dikenal jatuh ke World. Tidak pernah dari
  API GeoIP jaringan (tanpa kebocoran kueri, tanpa dependensi)
  dan tidak pernah dari deklarasi operator (zona-zona
  diwajibkan oleh build).
- Undian deterministik dan lokal: tanpa rahasia, tanpa
  ketakterdugaan yang disyaratkan (keamanannya ada pada tanda
  tangan record, bukan pada undian).

### Contoh lengkap (spesifikasi referensi)

`www.example.scone` mempublikasikan (apex `example.scone`,
seluruh himpunan melayani setiap subnama):

| Scope | Endpoint | Priority | Weight |
|---|---|---|---|
| US (840) | 10.1.0.1 | 10 | 70 |
| US (840) | 10.1.0.2 | 10 | 30 |
| US (840) | 10.2.0.1 | 20 | 100 |
| TW (158) | 10.3.0.1 | 10 | 80 |
| TW (158) | 10.4.0.1 | 10 | 20 |
| TW (158) | 10.2.0.1 | 20 | 100 |
| WORLD | 10.0.0.1 | 100 | 100 |

Perilaku yang diharapkan:

- klien **US** → 10.1.0.1 (≈70%) atau 10.1.0.2 (≈30%); level
  p20 hanya tercapai jika tak ada endpoint p10 yang aktif
  (weight = 0 di mana-mana) → barulah **10.2.0.1**;
- klien **TW** → 10.3.0.1 (≈80%) atau 10.4.0.1 (≈20%);
- klien **FR** (250) → tanpa rute FR atau EU dalam himpunan →
  WORLD → **10.0.0.1**;
- pertanyaan **A** pada rute khusus-IPv6 → jawaban kosong
  (tidak pernah IPv6 dalam A);
- domain tanpa aturan yang berlaku → pertanyaan sama, perilaku
  klasik (NODATA: GEOROUTE tidak punya rdata A wire).

### Opsi CLI (`scone relay`)

| Opsi | Default | Peran |
|---|---|---|
| (tanpa — GeoDNS selalu aktif, tabel tertanam) | on | Routing otomatis per IP sumber |
| `--geo-ip-table`, `--geo-config`, `--geo-country/…`, `--health-*` | — | (DIHAPUS — kerusakan keras: geozone diwajibkan oleh build dan resolver tidak melakukan pemeriksaan kesehatan; tidak ada flag runtime yang ada) |

#### Tabel tertanam adalah satu-satunya sumber

Tabel GeoIP **diwajibkan oleh build** (DB-IP Lite, terkuantisasi,
lihat bagian tabel tertanam di atas): tidak ada tabel operator
yang bisa diimpor. Resolusi: **IP sumber kueri DNS**
(per-klien: UDP `recv_from` / TCP `peer_addr`) dibandingkan
dalam 16 byte terhadap tabel tertanam; aturan dengan **prefiks
cocok terpanjang** menang (seri: yang pertama dalam urutan
berkas).

- **sumber privat/tak-terroute** (RFC1918, ULA fc00::/7,
  link-local, loopback): klien berada DI DALAM jaringan relay —
  ia mewarisi posisi milik relay yang ditemukan sendiri;
- **sumber publik yang absen dari tabel**: World (tanpa
  pemisahan geo — tidak pernah jawaban yang salah).

**Keketatan** dipertahankan untuk permukaan tabel masa depan
apa pun: baris tidak valid adalah error startup bertipe yang
membawa nomor baris pelanggaran; tidak pernah baris yang
diabaikan diam-diam, tidak pernah tabel parsial.

#### Prioritas opsi

- **Locator**: tabel tertanam MENGWAJIBKAN geolokasi — tidak
  ada penimpaan operator. Relay menemukan posisinya dari alamat
  eksternalnya (STUN/AutoNAT/UPnP yang diselesaikan oleh
  tabel); klien LAN mewarisi posisi itu; klien Internet
  diselesaikan oleh IP sumber mereka; yang tak dikenal jatuh
  ke World.

Contoh (routing geo aktif TANPA flag geo):

```bash
scone relay --dns 127.0.0.1:5353
```


## DoH / DoT (transport terenkripsi) — via frontend eksternal

Server DNS Scone berbicara **UDP dan TCP polos** (RFC 1035 /
7766) — itulah lingkup aslinya, lengkap dan teruji. Transport
terenkripsi ke *klien akhir* (DoH — DNS-over-HTTPS, RFC 8484;
DoT — DNS-over-TLS, RFC 7858) **didelegasikan ke frontend
standar**: Scone melayani datanya, frontend melayani
transportnya.

Mengapa desain ini: server DoH asli memerlukan stack HTTPS penuh
di dalam node (sertifikat, pembaruan, HTTP/2) — permukaan
operasional besar tanpa manfaat sebelum ada pengguna akhir di
sisi peramban. Frontend yang ada melakukannya lebih baik dan
lebih lama; node tetap minimal.

### Contoh: unbound melakukan DoH di depan relay Scone

`/etc/unbound/unbound.conf`:

```
server:
    interface: 127.0.0.1
    port: 5354
    do-ip4: yes
    do-udp: no
    do-tcp: yes

    # DoH (RFC 8484) pada 443
    https-port: 443
    tls-service-key: "/etc/unbound/scone.key"
    tls-service-pem: "/etc/unbound/scone.pem"

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353   # relay Scone (--dns 127.0.0.1:5353)
```

Sertifikat (`scone.pem`/`scone.key`) diperoleh seperti layanan
TLS apa pun — mis. certbot/acme.sh pada domain resolver. Klien
DoH lalu mengarah ke `https://dns.example.tld/dns-query`.

### Contoh: knot-resolver melakukan DoT

`/etc/knot-resolver/kresd.conf`:

```
net.listen('127.0.0.1', 5353, { kind = 'dns' })
net.listen('0.0.0.0', 853, { kind = 'tls' })
tls.cert_file = '/etc/knot-resolver/scone.pem'
tls.key_file  = '/etc/knot-resolver/scone.key'
```

### Apa yang ditambahkan frontend

- **Enkripsi kanal klien→frontend** (segmen permusuhan:
  jaringan pengguna); segmen frontend→relay tetap lokal (mesin
  sama atau jaringan terpercaya).
- **Cache ekstra di sisi frontend** (cache unbound/knot juga —
  tak berbahaya, TTL Scone tetap batas atas).
- **Endpoint stabil** (IPv4/IPv6, domain) sementara relay
  berpindah.

### Apa yang TIDAK dilakukan frontend

- Ia tidak *memverifikasi* apa pun: setiap jawaban yang
  dilayani relay sudah diverifikasi terhadap chain sebelum
  di-cache (lihat "Jalur resolusi"). Frontend meneruskan
  jawaban yang sudah otoritatif.
- Ia tidak menggantikan desentralisasi: frontend DoH adalah
  layanan *yang dioperasikan* — pilihan operator resolver, bukan
  milik protokol. Setiap relay Scone tetap dapat dipakai
  independen via UDP/TCP asli.

### Implementasi asli mendatang (terdokumentasi, belum
dikomitkan)

Transport DoH asli (`--doh ADDR`) tetap mungkin: format wirenya
identik (RFC 8484 memakai ulang pesan DNS biner), itu akan
berupa dekode POST/GET + HTTP/2 di belakang proxy TLS
(caddy/nginx menangani sertifikat), node itu sendiri tetap tanpa
TLS. Tanpa pekerjaan berjalan — permintaan akan datang bersama
pengguna akhir.

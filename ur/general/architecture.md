# فنِ تعمیر

## پہلے سادہ الفاظ میں

باقی سب کچھ سمجھنے کے لیے پانچ تصورات کافی ہیں:

1. **ایک نام** — انٹرنیٹ پر ایک پتہ، جیسے `mysite.scone`۔
2. **آپ کا 24 لفظی جملہ** — آپ کی شناخت۔ یہ وہ دستاویز ہے جو
   ثابت کرتی ہے کہ کوئی نام آپ کا ہے، اور آپ کی ہر تبدیلی پر دستخط
   کرتی ہے۔
3. **عوامی ریکارڈ بک** (بلاکچین) — یہ مشترکہ، عوامی
   کھاتا ہے کہ کس کے پاس کیا ہے۔ کوئی اسے کنٹرول نہیں کرتا، کوئی
   اسے دوبارہ لکھ نہیں سکتا۔
4. **مائننگ** — نام محفوظ کرنے کا طریقہ: آپ کا کمپیوٹر کچھ
   کام کرتا ہے، سب کے لیے منصفانہ اور برابر۔ کوئی قطار نہیں، کوئی
   درمیانی شخص نہیں۔
5. **ایک ریلے** — ایک کمپیوٹر جو بک کی ایک کاپی رکھ کر اور
   سوالات کے جواب دے کر نیٹ ورک چلاتا ہے۔ کوئی بھی اسے
   چلا سکتا ہے؛ جتنے زیادہ ہوں گے، نیٹ ورک اتنا ہی مضبوط ہوگا۔

اس صفحے کا باقی حصہ گہرائی میں جاتا ہے — آپ یہیں رک سکتے ہیں اور
پروجیکٹ سمجھ چکے ہوں گے۔

## رہنما اصول

Scone اختیار کو دستیابی سے الگ کرتا ہے:

- **بلاکچین** ملکیت اور تبدیلیوں کی ترتیب کے لیے سچائی کا ذریعہ ہے؛
- **DHT** ایک تقسیم/نقل نظام ہے، یہ کوئی **اختیار** نہیں؛
- **مقامی اسٹوریج** نوڈ کی اسٹیٹ، انڈیکسز اور کیشیز رکھتا ہے۔

ایک نوڈ کو DHT سے آنے والے ہر ڈیٹا کو ممکنہ طور پر
موقع اندیش سمجھنا چاہیے: یہ صرف خفیہ نگاری کی تصدیق کے بعد
قبول کیا جاتا ہے (مالک کا دستخط، چین کی ملکیت / sequence /
record_hash سے مطابقت)۔

## ذمہ داریوں کی علیحدگی

```text
BLOCKCHAIN
──────────
Ownership
Registration
Authorization
Ordering
Sequence
Record hash
Consensus information

DHT
───
DNS records
Signed records
Replication
Availability

LOCAL STORAGE
─────────────
Local state
Indexes
Cached DHT records
Blockchain state
```

## Crates اور انحصارات

```text
scone-crypto     ← lowest layer: pure primitives over bytes
    ▲
scone-core       ← protocol types (depends on scone-crypto)
    ▲
scone-protocol  ← wire/blockchain binary format
    ▲
scone-blockchain← blockchain logic, canonical state (in memory)
    ▲               └ scone-topology (pure engine, zero network/fs/
                     async; fed by scone-network) — scone-genesis
                     (one-shot mainnet genesis tool binary)
scone-storage   ← local persistence: redb backend + verified snapshots
    ▲
scone-network   ← p2p relay: libp2p/QUIC, mempool, sync, DHT, DNS
    ▲
scone-keystore  ← encrypted `.sconekey` keyfiles (depends on scone-crypto)
    ▲
scone           ← binary / application
```

### فیصلہ: `scone-core` کا `scone-crypto` پر انحصار

`DomainId` کو بنیاد سے قابلِ حساب ہونا چاہیے
(`BLAKE3-256(prefix || canonical name)`)، لیکن ٹھوس خفیہ نگاری
کو core میں داخل نہیں ہونا چاہیے۔ حل:

- `scone-crypto` سب سے نچلی پرت ہے: بائٹس پر خالص متعین فنکشنز
  (بمطابق کلید کی تخلیق کے لیے OS RNG)، صفر core اقسام،
  کوئی نیٹ ورک/فائل سسٹم/async انحصار نہیں → کوئی سرکلٹی ممکن نہیں؛
- `scone-core` اسے صرف ہیشنگ کے لیے استعمال کرتا ہے؛
- Ed25519 دستخط اور کلیدیں `scone-crypto` میں رہتی ہیں اور
  بائٹس پر کام کرتی ہیں (کیننیکل پے لوڈز جو `scone-protocol` بنتا ہے)۔

### ہر crate کا کردار

**scone-core** — نام (`TldName`، `DomainName`)، شناخت کنندگان
(`DomainId`، `OwnerId`، `PublicKeyRef`)، DNS ڈیٹا (`DnsRecord`،
`SignedDnsRecord`)، ٹرانزیکشنز (`RegisterDomain`، `UpdateDomain`)۔
متعین اور پورٹیبل۔ اس crate میں ممنوع: libp2p، redb،
tokio/async، نیٹ ورک، فائلیں، OS، DNS سرور، بلاکچین لاجک۔

**scone-crypto** — خفیہ نگاری کے پریمیٹو۔ آج: `hash256`
(BLAKE3)، Ed25519 دستخط اور کلیدیں (`SigningKey`،
`PublicKey`، `Signature`، سخت بیچ تصدیق)، ریکارڈ
گروپس کا AEAD لفافہ (XChaCha20-Poly1305)۔ منصوبے میں: Merkle درخت۔
قاعدہ: کوئی گھریلو الگورتھم نہیں، صرف آڈٹ شدہ انحصارات۔

**scone-protocol** — پروٹوکول کا بائنری فارمیٹ: ٹرانزیکشنز، DNS
ریکارڈز، بلاکس (`Block`/`BlockHeader` بمع ایک اپاہج کنسنسس
فیلڈ) اور P2P پیغامات کی کیننیکل انکوڈنگ/ڈی کوڈنگ (کم سے کم LEB128
varint، ننگے 32 بائٹ شناخت کنندگان)، مختص کرنے سے پہلے سائز
کی حدوں کی جانچ، غیر کیننیکل شکلوں کا انکار، اور ورژننگ
(`PROTOCOL_VERSION`)۔ کوئی کنسنسس، نیٹ ورک، یا اسٹوریج لاجک نہیں۔
فارمیٹ حوالہ: `/docs/en/technical/protocol.md`۔

**scone-blockchain** — بلاکچین لاجک اور کیننیکل اسٹیٹ:
ٹرانزیکشن کی شناخت (`TxId`)، ترتیب شدہ Merkle درخت،
`BlockHash`، متعین جنسیس، بلاک اور ٹرانزیکشن کی تصدیق،
اسٹیٹ اطلاق کے قواعد (`DomainState`/`ChainState`، میموری میں،
`DomainId` سے براہِ راست رسائی)، فورک کی دریافت اور برانچ کا انتخاب
(`fork_choice`: فائنلٹی فرش + طویل ترین چین + سب سے کم ہیش
ٹائی بریک)، چیک پوائنٹ فائنلٹی، اور کنسنسس تجرید
(`trait Consensus`، بطور ڈیفالٹ روایادار نفاذ)۔
رجسٹریشن PoW (مائن/تصدیق، فی نیٹ ورک مشکلیں) `scone-core`
میں رہتا ہے۔ اس crate میں ممنوع: نیٹ ورک، DHT، اسٹوریج (redb)،
DNS سرور۔ حوالہ: `/docs/en/technical/blockchain.md`۔

**scone-storage** — مقامی استقامت (شپڈ)۔ ٹھوس بیک اینڈ
`redb` ہے (`NodeStore`)، جو ایک کم سے کم trait کے پیچھے چھپا ہوا ہے
تاکہ بلاکچین/DHT/DNS کبھی براہِ راست redb سے جڑے نہ رہیں؛ کوئی
دوسرا بیک اینڈ (تقسیم شدہ KV) پسوندھا جا سکتا ہے۔ اس میں صفحہ بندی
شدہ KV اسٹیٹ بیک اینڈ، بلاک لاگ، تصدیق شدہ اسنیپ شاٹس (ہیشڈ مینی فیسٹ
+ صفحات) اور ری سٹارٹ ریسٹور/ری پلے پاتھ شامل ہیں۔ حوالہ:
`/docs/en/technical/storage.md`۔

**scone-network** — نوڈ کا p2p ریلے: libp2p/QUIC سوارم،
ٹرانزیکشن اور بلاک ریلے (مکمل مقامی تصدیق)، محدود
mempool، محدود بلاک سنک (128 بلاکس/درخواست)، `SignedDnsRecord` کے
لیے Kademlia DHT، UDP+TCP DNS سرور (GeoDNS ہمیشہ آن —
ایمبیڈڈ GeoIP ٹیبل، ریکرسو فال بیک)، اینکرز کا
چیک پوائنٹ/ہارٹ بیٹ لوپ (CLI ریلے سے براہِ راست p2p QUIC بولتی
ہے، اب کوئی مقامی RPC نہیں)۔ پہنچ اور دریافت: AutoNAT
v2 (قابلِ رسائی/NATed فیصلہ)، سرکٹ ریلے v2 + DCUtR برائے NATed
نوڈز اور UPnP (روٹر کی جانب سے QUIC پورٹ میپنگ، آپٹ اِن)۔ `.env` فائل کے
ذریعے کنفیگریشن (دیکھیں `.env.example`)۔
حوالہ: `/docs/en/technical/relay.md`۔

**scone-keystore** — Ed25519 دستخطی کلیدوں کا خفیہ شدہ اسٹوریج
(`.sconekey` فائلیں: Argon2id + XChaCha20-Poly1305)۔ یہ crate
فائل سسٹم کا مالک ہے تاکہ `scone-crypto` خالص رہ سکے۔ حوالہ:
`/docs/en/technical/keystore.md`۔

**scone-topology** — مقامی p2p ٹوپولوجی کی بہتری کے لیے خالص
انجن: زیرو نیٹ ورک، زیرو فائل سسٹم، زیرو async۔ یہ نہ libp2p
جانتا ہے نہ سوارم: `scone-network` اسے مشاہدات (ping RTT،
کنکشنز) کھلاتا ہے اور اس سے محلے کے انتظام کے فیصلے واپس لیتا ہے۔

**scone-genesis** — مین نیٹ جنسیس کی تیاری کے لیے ایک بار
استعمال ہونے والا ٹول بائنری (kickstarter): ورژن شدہ JSON مینی فیسٹ،
الاٹمنٹس، متعین بلاک۔ ٹول آرٹیفیکٹس پیدا کرتا ہے، ریپوزٹوری انہیں
کبھی ورژن نہیں کرتا (Rust ورک اسپیس کے اندر لکھنے سے انکار؛
تقسیم لانچ کے لیے مخصوص، دیکھیں `blockchain.md`)۔

**scone** — بائنری/CLI۔ کسی crate پر صرف اُس وقت انحصار کرتا ہے جب
واقعی اسے استعمال کرے (آج: ریلے کے لیے scone-network،
scone-blockchain، scone-storage، scone-keystore، scone-protocol،
scone-core، scone-crypto، scone-topology)۔

## نوڈ

ایک Scone نوڈ ملانے والا ہے:

1. ایک بلاکچین (کنسنسس تصدیق، ملکیت کی اسٹیٹ)؛
2. DHT روٹنگ (`SignedDnsRecord` کی اشاعت/حصول)؛
3. مقامی اسٹوریج (اسٹیٹ، انڈیکسز، کیشیز)؛
4. سامنے ایک ریزالور / DNS سرور۔

### نیا نوڈ شروع کرنا (اسٹیٹ بوٹ اسٹریپ، P0.3/P0.4)

نیا نوڈ جنسیس دوبارہ ری پلے نہیں کرتا: یہ ایک تصدیق شدہ
فائنل شدہ اسٹیٹ سے شروع ہوتا ہے۔

```text
finalized checkpoint (committee, signature quorum — validated by
        │              the chain layer before any use)
        ▼
verified snapshot (manifest + paginated pages; the importer
        │              RECOMPUTES the SMT root from the entries
        │              and compares it to the signed state_root —
        │              a falsified/incomplete snapshot is rejected
        │              before any write)
        ▼
anchor block H (requested from the network BY HASH)
        │
        ▼
recent blocks H+1..H+k (relay, increasing height)
        │
        ▼
synced — at boot, replay of ONLY the suffix above H
```

دو نوڈ پروفائلز:

- **آرکائیو نوڈ** (آپریٹر فلیگ): تمام تاریخی بلاکس رکھتا ہے،
  انہیں نیٹ ورک کو پیش کرتا ہے، موجودہ پاتھز سے شروع ہوتا ہے؛
- **فُل اسٹیٹ نوڈ**: H پر مکمل اسٹیٹ اور سفکس رکھتا ہے،
  لیکن H سے نیچے تاریخ نہیں — یہ مستقبل کے بلاکس کی تصدیق کرتا ہے اور
  اسٹیٹ پیش کرتا ہے، تاریخ پیش نہیں کرتا۔

یہ پاتھ آپٹ اِن (OPT-IN) ہے: تصدیق شدہ اسنیپ شاٹ درآمد کرنا ایک
واضح آپریٹر عمل ہے (`import_verified_snapshot`)، کبھی خودکار نہیں —
کسی موجودہ نوڈ کا ڈیفالٹ رویہ تبدیل نہیں ہوتا۔
نافذہ ذرائع کی تفصیل: `/docs/en/technical/storage.md`، سیکشنز
"VERIFIED state snapshot" اور "State bootstrap"۔

## DNS ریزولوشن فلو

```text
DNS query (example.uip)
    │
    ▼
Node: DomainId = BLAKE3-256("SCONE-DOMAIN-V1" || "example.uip")
    │
    ├── authoritative DNS snapshot (RAM) ─► unknown/expired → NXDOMAIN
    │                                          (without touching the core)
    ├── sharded DNS cache (qname,qtype) ───► hit: shared entry,
    │                                          lazy verification
    ├── local cache (SignedDnsRecord) ─────► verification (signature, owner,
    │                                          sequence, record_hash vs chain)
    ├── DHT get(DomainId) ─────────────────► same verification
    │                                          (miss → 30 s negative cache)
    ▼
DNS answer (if and only if valid)
```

DNS اسنیپ شاٹ (ہر زندہ ڈومین کا وجود + میعاد، ہر لاگو شدہ بلاک کے
بعد شائع شدہ) گارنٹی دیتا ہے کہ وجود کا فیصلہ کبھی بھی ریلے کے رن
لوپ سے نہیں گزرتا: core کو صرف اصل ریکارڈ کی بازیافت کے لیے چھوا
جاتا ہے۔

## ارتقاء پذیری

- ورژن شدہ پری امیجز (`SCONE-DOMAIN-V1`، `SCONE-PUBKEY-V1`،
  `SCONE-OWNER-V1`، …): کسی حصول کے قاعدے میں تبدیلی ایک نیا
  نیم اسپیس بناتی ہے بجائے موجودہ کو توڑنے کے؛
- `RecordData` variants + `Unknown { type_code, data }` کے ذریعے
  پھیلنے پذیر ہے، پڑھنے کے وقت نامعلوم اقسام کے لیے؛
- وائر فارمیٹ کے وقفے کے لیے `PROTOCOL_VERSION` (`scone-protocol`)؛
- چین صرف ہیش رکھتی ہے: بڑا ڈیٹا DHT میں رہتا ہے۔

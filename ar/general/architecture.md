# الهندسة المعمارية

## أولًا بكلمات بسيطة

خمسة مفاهيم تكفي لفهم كل ما تبقّى:

1. **اسم** — عنوان على الإنترنت، مثل `mysite.scone`.
2. **عبارتك المكوّنة من 24 كلمة** — هي هويتك. إنها سند الملكية
   الذي يثبت أن الاسم لك، وتوقّع على كل تغيير تجريه.
3. **دفتر السجلات العام** (البلوكشين) — السجل المشترك
   والعام لمن يملك ماذا. لا أحد يسيطر عليه، ولا أحد
   يستطيع إعادة كتابته.
4. **التعدين** — كيف تحجز اسمًا: يقوم حاسوبك ببعض
   العمل، عادل ومتساوٍ للجميع. لا طوابير ولا وسيط.
5. **مُرحّل (relay)** — حاسوب يشغّل الشبكة بالإبقاء على
   نسخة من الدفتر والإجابة عن الأسئلة. يمكن لأي أحد تشغيل
   واحد؛ وكلما زاد عددها، قويت الشبكة.

بقية هذه الصفحة تتعمق أكثر — يمكنك التوقف هنا وستكون
قد فهمت المشروع بالفعل.

## المبدأ الموجّه

يفصل Scone السلطة عن التوفر:

- **البلوكشين** هو مصدر الحقيقة للملكية وترتيب التغييرات؛
- **الـ DHT** نظام توزيع/نسخ، وهو **ليس** سلطة؛
- **التخزين المحلي** يحتفظ بحالة العقدة والفهارس وذاكرات التخزين المؤقت.

يجب على العقدة أن تعامل أي بيانات قادمة من الـ DHT على أنها
ربما خبيثة: لا تُقبل إلا بعد تحقق تشفيري (توقيع المالك، مطابقة
مع الملكية / التسلسل / record_hash الخاصة بالسلسلة).

## فصل المسؤوليات

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

## الـ crates والتبعيات

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

### قرار: `scone-core` يعتمد على `scone-crypto`

يجب أن يكون حساب `DomainId` ممكنًا من الأساس صعودًا
(`BLAKE3-256(prefix || canonical name)`)، لكن التشفير الملموس
يجب ألا يلوّث النواة. الحل:

- `scone-crypto` هو الطبقة الأدنى: دوال حتمية بحتة فوق
  البايتات (بالإضافة إلى مولّد العشوائية OS لتوليد المفاتيح)،
  صفر أنواع نواة، لا تبعيات شبكة/نظام ملفات/async ← لا دوران
  ممكن؛
- يستخدمه `scone-core` للت hashing فقط؛
- توقيعات Ed25519 والمفاتيح تعيش في `scone-crypto` وتعمل على
  البايتات (الحمولات القانونية ينتجها `scone-protocol`).

### دور كل crate

**scone-core** — الأسماء (`TldName`، `DomainName`)، المعرّفات
(`DomainId`، `OwnerId`، `PublicKeyRef`)، بيانات DNS
(`DnsRecord`، `SignedDnsRecord`)، المعاملات (`RegisterDomain`،
`UpdateDomain`). حتمي وقابل للنقل. ممنوع في هذا الـ crate:
libp2p، redb، tokio/async، الشبكة، الملفات، OS، خادم DNS، منطق
البلوكشين.

**scone-crypto** — البدائيات التشفيرية. اليوم: `hash256`
(BLAKE3)، توقيعات ومفاتيح Ed25519 (`SigningKey`،
`PublicKey`، `Signature`، تحقق دفعي صارم)، غلاف AEAD
لمجموعات السجلات (XChaCha20-Poly1305). مخطط له: شجرة Merkle.
قاعدة: لا خوارزميات منزلية، تبعيات مدققة فقط.

**scone-protocol** — الصيغة الثنائية للبروتوكول: ترميز/فك
ترميز قانوني (varint أدنى LEB128، معرّفات عارية من 32 بايت)
للمعاملات، سجلات DNS، الكتل (`Block`/`BlockHeader` بحقل
consensus معتم) ورسائل P2P، مع فحص حدود الحجم قبل التخصيص،
ورفض الأشكال غير القانونية، والإصدارات (`PROTOCOL_VERSION`).
لا منطق توافق ولا شبكة ولا تخزين. مرجع الصيغة:
`/docs/en/technical/protocol.md`.

**scone-blockchain** — منطق البلوكشين والحالة القانونية:
تعريف المعاملات (`TxId`)، شجرة Merkle المرتّبة،
`BlockHash`، تكوين حتمي (genesis)، التحقق من الكتل
والمعاملات، قواعد تطبيق الحالة (`DomainState`/`ChainState`،
في الذاكرة، وصول مباشر بـ `DomainId`)، اكتشاف التشعّبات
واختيار الفرع (`fork_choice`: أرضية النهائية + أطول سلسلة
+ كسر التعادل بأدنى hash)، نهائية نقاط التفتيش، وتجريد
التوافق (`trait Consensus`، تنفيذ متسامح افتراضيًا). تعدين
PoW التسجيل (mine/verify، صعوبات لكل شبكة) يعيش في
`scone-core`. ممنوع في هذا الـ crate: الشبكة، DHT، التخزين
(redb)، خادم DNS. المرجع: `/docs/en/technical/blockchain.md`.

**scone-storage** — التخزين المحلي الدائم (المُسلَّم). الخلفية
الملموسة هي `redb` (`NodeStore`)، مخفية خلف trait أدنى بحيث
لا يقترن البلوكشين/DHT/DNS مباشرة بـ redb أبدًا؛ يمكن توصيل
خلفية أخرى (KV موزّع). يشمل خلفية الحالة KV المرقّمة الصفحات،
وسجلّ الكتل، واللقطات المتحقق منها (manifest مُhash + صفحات)
ومسار الاستعادة/إعادة التشغيل عند الإقلاع. المرجع:
`/docs/en/technical/storage.md`.

**scone-network** — مُرحّل p2p للعقدة: سرب libp2p/QUIC، ترحيل
المعاملات والكتل (تحقق محلي منهجي)، mempool محدود، مزامنة
كتل محدودة (128 كتلة/طلب)، DHT بـ Kademlia لـ
`SignedDnsRecord`، خادم DNS عبر UDP+TCP (GeoDNS مفعّل دائمًا —
جدول GeoIP مدمج، تراجع تكراري)، حلقة checkpoint/heartbeat
للـ anchors (يتحدث CLI مباشرة عبر p2p QUIC إلى المُرحّل، لم يعد
هناك RPC محلي). قابلية الوصول والاكتشاف: AutoNAT v2 (حكم
قابل للوصول/خلف NAT)، circuit-relay v2 + DCUtR للعقد خلف NAT
وUPnP (تخطيط منفذ QUIC من قبل الموجّه، opt-in). الإعداد عبر
ملف `.env` (انظر `.env.example`). المرجع: `/docs/en/technical/relay.md`.

**scone-keystore** — تخزين مشفّر لمفاتيح توقيع Ed25519
(ملفات `.sconekey`: Argon2id + XChaCha20-Poly1305). هذا الـ
crate يملك نظام الملفات ليبقى `scone-crypto` نقيًا. المرجع:
`/docs/en/technical/keystore.md`.

**scone-topology** — محرك نقي لتحسين الطوبولوجيا المحلية p2p:
صفر شبكة، صفر نظام ملفات، صفر async. لا يعرف libp2p ولا
السرب: يغذّيه `scone-network` بالملاحظات (RTT الـ ping،
الاتصالات) ويتلقى قرارات إدارة الجوار.

**scone-genesis** — أداة تنفيذية أحادية الاستخدام (kickstarter)
لتحضير تكوين mainnet: manifest بإصدارات JSON، التخصيصات،
كتلة حتمية. الأداة تُنتج المُخرجات (artifacts)، والمستودع لا
يضعها أبدًا تحت الإصدارات (يرفض الكتابة داخل workspace لـ
Rust؛ التوزيع محفوظ للإطلاق، انظر `blockchain.md`).

**scone** — الملف التنفيذي/CLI. يعتمد على الـ crate فقط عندما
يستخدمه فعلًا (اليوم: scone-network للمُرحّل، scone-blockchain،
scone-storage، scone-keystore، scone-protocol، scone-core،
scone-crypto، scone-topology).

## العقدة

تجمع عقدة Scone بين:

1. بلوكشين (تحقق التوافق، حالة الملكية)؛
2. توجيه DHT (نشر/جلب `SignedDnsRecord`)؛
3. تخزين محلي (الحالة، الفهارس، ذاكرات التخزين المؤقت)؛
4. محلّل / خادم DNS في المقدمة.

### إقلاع عقدة جديدة (تهيئة الحالة، P0.3/P0.4)

العقدة الجديدة لا تعيد تشغيل تكوين genesis: تبدأ من حالة
نهائية متحقق منها.

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

ملفان تعريفان للعقدة:

- **عقدة أرشيف** (علامة المشغّل): تحتفظ بكل الكتل التاريخية،
  تقدّمها للشبكة، تبدأ عبر المسارات القائمة؛
- **عقدة حالة كاملة**: تحمل الحالة الكاملة عند H واللاحقة،
  لكنها لا تحمل التاريخ تحت H — تتحقق من الكتل المستقبلية
  وتقدّم الحالة، ولا تقدّم التاريخ.

هذا المسار اختياري (OPT-IN): استيراد لقطة متحقق منها فعلٌ
صريح من المشغّل (`import_verified_snapshot`)، ولا تلقائي أبدًا —
سلوك عقدة قائمة افتراضيًا لا يتغير. التفاصيل المعيارية:
`/docs/en/technical/storage.md`، قسمَي
"VERIFIED state snapshot" و"State bootstrap".

## تدفق حلّ DNS

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

تضمن لقطة DNS (الوجود + انتهاء الصلاحية لكل نطاق حي،
تُنشر بعد كل كتلة مطبَّقة) أن قرار الوجود لا يمرّ أبدًا عبر حلقة
تشغيل المُرحّل: لا تُلمس النواة إلا لجلب السجلات الفعلي.

## قابلية التطور

- preimages مُصدَرة (`SCONE-DOMAIN-V1`، `SCONE-PUBKEY-V1`،
  `SCONE-OWNER-V1`، …): تغيير قاعدة اشتقاق ينشئ فضاء أسماء
  جديدًا بدل أن يكسر القائم؛
- `RecordData` قابلة للتوسعة عبر variants + `Unknown { type_code, data }`
  للأنواع غير المعروفة وقت القراءة؛
- `PROTOCOL_VERSION` (في `scone-protocol`) لكسر صيغة الـ wire؛
- السلسلة تخزّن الـ hashes فقط: البيانات الكبيرة تبقى في الـ DHT.
